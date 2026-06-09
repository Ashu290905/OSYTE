# Liquidity Calendar Service — Architecture & Implementation Plan

> **Source spec:** [Liquidity Calendar Service — Confluence](https://osyte.atlassian.net/wiki/spaces/ZOTBPB/pages/1309573122/Liquidity+Calendar+Service)
> **Last updated:** June 2026

---

## Table of Contents

1. [Requirements Summary](#1-requirements-summary)
2. [System Context & Boundaries](#2-system-context--boundaries)
3. [Architecture Overview](#3-architecture-overview)
4. [Core Domain Model](#4-core-domain-model)
5. [Component Design](#5-component-design)
6. [API Design](#6-api-design)
7. [Holiday Data Subsystem](#7-holiday-data-subsystem)
8. [Event & Change Propagation](#8-event--change-propagation)
9. [NL Query & MCP Layer](#9-nl-query--mcp-layer)
10. [Data Storage Strategy](#10-data-storage-strategy)
11. [Non-Functional Requirements](#11-non-functional-requirements)
12. [Tech Stack](#12-tech-stack)
13. [Phased Delivery Plan](#13-phased-delivery-plan)
14. [Risks & Mitigations](#14-risks--mitigations)
15. [Open Questions](#15-open-questions)

---

## 1. Requirements Summary

### 1.1 Functional Requirements

| # | Requirement |
|---|-------------|
| FR-01 | Given an instrument and an anchor date, compute the full set of key dates: **notification**, **trade**, **valuation**, and **settlement**. |
| FR-02 | Persist a forward-looking calendar per instrument (materialized view), refreshed automatically when holiday data changes. |
| FR-03 | Support **pluggable holiday data sources**: Copp Clark (default), client-supplied files, and third-party vendor integrations. |
| FR-04 | Support **composite holiday calendars**: a base vendor calendar overlaid with client-specific house holidays and exceptions. Conflicts resolve in the client's favour and are surfaced in a changelog. |
| FR-05 | When holiday data updates, **automatically recompute** all affected instrument calendars and publish a changelog (what moved, which instruments, which dates). |
| FR-06 | Capture liquidity terms per instrument: notification offset, dealing frequency, valuation lag, settlement cycle, cut-off time, roll convention per rule. |
| FR-07 | Support standard **roll conventions**: Following, Modified Following, Preceding, Modified Preceding. |
| FR-08 | Support **cross-currency/cross-market** instruments requiring multiple holiday calendars to align. |
| FR-09 | Store calendars with **effective-dated versions**: answer "what did the calendar say on date X" (audit) as well as "what does it say today" (current). |
| FR-10 | Expose a **Compute API** (real-time, stateless, per-trade) consumed inline by Investment Planning, Rebalancing, and Trade Ops. |
| FR-11 | Expose a **Calendar API** (persisted, forward-looking) for reading, subscribing to, and exporting calendars. |
| FR-12 | Expose a **Natural-Language Query layer** for non-API users (ops, PMs, IR, client-service). |
| FR-13 | Expose the same query surface as an **MCP server** consumable by AI assistants inside Osyte or in the client's own tools. |
| FR-14 | Integrate with the standard Osyte **instrument onboarding workflow** for capturing liquidity terms on new instruments. |
| FR-15 | Roadmap: AI-assisted extraction of liquidity terms from PPMs, side letters, term sheets — always with human review before persistence. |

### 1.2 Non-Functional Requirements

| # | Requirement |
|---|-------------|
| NFR-01 | Compute API p95 latency < 100 ms. |
| NFR-02 | Calendar reads served from the persisted store; no blocking on live computation during reads. |
| NFR-03 | Date computation is **fully deterministic** — no AI in the calculation path. |
| NFR-04 | Dates returned by the Compute API must always match the persisted Calendar for the same inputs and holiday version. |
| NFR-05 | Full audit trail: every calendar version, every holiday-data ingestion, every date that moved. |
| NFR-06 | Multi-tenancy: each client sees only their own instruments and is isolated from other clients' holiday overlays. |
| NFR-07 | Changelog events are durable and consumable downstream (Investment Planning, Rebalancing, Trade Ops can subscribe). |
| NFR-08 | System must handle retroactive holiday changes — historical calendar versions must remain queryable. |

---

## 2. System Context & Boundaries

```
┌─────────────────────────────────────────────────────────────────┐
│                         OSYTE PLATFORM                          │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  Investment  │  │ Rebalancing  │  │     Trade Ops        │  │
│  │  Planning    │  │   Service    │  │     Workflow         │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
│         │                 │                      │              │
│         └─────────────────┼──────────────────────┘              │
│                           │  Compute API / Calendar API         │
│                    ┌──────▼────────────────────┐                │
│                    │  LIQUIDITY CALENDAR SVC   │◄───── MCP ─────┼── AI Agents
│                    └──────┬─────────┬──────────┘                │
│                           │         │                           │
│                    ┌──────▼──┐  ┌───▼──────────────┐           │
│                    │Compute  │  │Calendar Store +  │           │
│                    │Engine   │  │Changelog         │           │
│                    └──────┬──┘  └──────────────────┘           │
│                           │                                     │
│              ┌────────────▼────────────┐                        │
│              │  Holiday Data Subsystem │                        │
│              │  ┌──────────────────┐   │                        │
│              │  │ Copp Clark (dflt)│   │                        │
│              │  │ Client Files     │   │                        │
│              │  │ 3rd-party vendor │   │                        │
│              │  │ Client Overlay   │   │                        │
│              │  └──────────────────┘   │                        │
│              └─────────────────────────┘                        │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Instrument Registry (liquidity terms, onboarding WF)    │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

**In-scope:** Computation engine, Calendar store, Holiday data subsystem, Compute & Calendar APIs, NL/MCP query layer, change-event streaming.

**Out-of-scope (for now):** Instrument onboarding UI (uses existing Osyte workflow), AI liquidity-term extraction from PDFs (roadmap).

---

## 3. Architecture Overview

The service follows a **CQRS + Event-Sourced** pattern:

- **Write path:** Holiday data ingestion → Computation Engine → Calendar materialization → Changelog event publication.
- **Read path:** Compute API (pure function, no DB hit) and Calendar API (read from persisted store) both call the same engine kernel, guaranteeing date consistency.
- **Query path:** NL interface and MCP server sit on top of the Calendar read path, never on the computation path.

```
                     ┌──────────────────────────────────────────┐
                     │              WRITE PATH                   │
                     │                                          │
  Holiday Source ───►│ Ingestion Worker                         │
  (Copp Clark,       │   │                                      │
   Client file,      │   ▼                                      │
   Vendor)           │ Holiday Store (versioned)                │
                     │   │                                      │
  Instrument        │   ├──────────────────────────────────┐   │
  Terms change ─────┤   ▼                                  ▼   │
                     │ Recompute Trigger (event-driven)         │
                     │   │                                      │
                     │   ▼                                      │
                     │ Computation Engine (pure function)       │
                     │   │                                      │
                     │   ├──► Calendar Store (versioned rows)   │
                     │   └──► Changelog Event → Event Bus       │
                     └──────────────────────────────────────────┘

                     ┌──────────────────────────────────────────┐
                     │              READ PATHS                   │
                     │                                          │
  Compute API ──────►│ Engine kernel (stateless, in-process)   │
  (real-time)        │   └──► Response (<100ms)                │
                     │                                          │
  Calendar API ─────►│ Calendar Store read                      │
  (persisted)        │   └──► Paginated key-date rows          │
                     │                                          │
  NL / MCP ─────────►│ Query translator → Calendar Store read  │
                     └──────────────────────────────────────────┘
```

---

## 4. Core Domain Model

### 4.1 Entities

```
InstrumentLiquidityTerms
├── instrumentId            UUID (FK to Instrument Registry)
├── tenantId                UUID
├── notificationOffset      BusinessDayOffset
├── dealingFrequency        DealingFrequency  (Daily|Weekly|Monthly|Quarterly|etc.)
├── dealingDayRule          DayRule           (e.g. "last BD of month", "15th BD")
├── cutOffTime              TimeOfDay + Timezone
├── valuationLag            CalendarDayOffset
├── settlementCycle         BusinessDayOffset (T+n)
├── rollConvention          RollConvention    (Following|ModFollowing|Preceding|ModPreceding)
├── calendarRefs            [CalendarRef]     (1..n market calendars to AND together)
├── effectiveFrom           Date
├── effectiveTo             Date | null       (null = current)
└── version                 Integer

HolidayCalendar
├── calendarId              UUID
├── tenantId                UUID | null       (null = global/vendor)
├── source                  Enum (CoppClark|ClientFile|Vendor|ClientOverlay)
├── marketCode              String            (ISO 10383 MIC or custom)
├── effectiveYear           Integer
├── holidays                [Date]
├── ingestedAt              Timestamp
├── dataVersion             String            (vendor's version/file hash)
└── baseCalendarId          UUID | null       (for overlays)

ComputedCalendar
├── instrumentId            UUID
├── tenantId                UUID
├── termsVersion            Integer
├── holidayDataVersion      String
├── generatedAt             Timestamp
├── effectiveDateRange      DateRange
└── entries                 [CalendarEntry]

CalendarEntry
├── dealingDate             Date
├── notificationDeadline    Date
├── tradeDate               Date
├── valuationDate           Date
├── settlementDate          Date
└── flags                   [String]          (e.g. "rolled:following", "holiday:NYSE")

CalendarChangelog
├── changeId                UUID
├── tenantId                UUID
├── triggeredBy             Enum (HolidayUpdate|TermsUpdate|Manual)
├── triggerRef              String
├── affectedInstruments     [UUID]
├── changedEntries          [EntryDiff]
└── publishedAt             Timestamp

EntryDiff
├── instrumentId            UUID
├── dealingDate             Date
├── field                   Enum (NotificationDeadline|TradeDate|ValuationDate|SettlementDate)
├── previousValue           Date | null
└── newValue                Date | null
```

### 4.2 Value Objects

```
BusinessDayOffset    { days: Int, calendarRef: CalendarRef }
CalendarDayOffset    { days: Int }
DealingFrequency     { rule: Enum, interval: Int? }
DayRule              { type: Enum, nth?: Int, dayOfWeek?: Enum }
RollConvention       Enum
```

---

## 5. Component Design

### 5.1 Computation Engine

The engine is a **pure function** — zero side effects, zero DB calls, fully testable in isolation:

```
compute(
  terms: InstrumentLiquidityTerms,
  holidayCalendars: Map<MarketCode, Set<Date>>,
  anchorDate: Date,
  horizon: DateRange
) → List<CalendarEntry>
```

**Algorithm per dealing window:**

1. Generate raw dealing dates from `DealingFrequency` + `DayRule` within `horizon`.
2. For each raw date, apply `RollConvention` using the relevant market calendars.
3. For each rolled dealing date:
   - `tradeDate = dealingDate` (or adjusted per terms)
   - `notificationDeadline = rollBack(tradeDate, notificationOffset, calendars)`
   - `valuationDate = addCalendarDays(dealingDate, valuationLag)`
   - `settlementDate = addBusinessDays(dealingDate, settlementCycle, calendars)`
4. Attach audit flags (which dates rolled, which holiday triggered the roll).

**Multi-calendar AND logic:** a date is only a business day if it is a business day in ALL referenced market calendars.

### 5.2 Holiday Data Subsystem

```
┌──────────────────────────────────────────────────────┐
│               Holiday Data Subsystem                  │
│                                                      │
│  ┌────────────────┐   ┌─────────────────────────┐   │
│  │ Copp Clark     │   │ Client File Uploader    │   │
│  │ Connector      │   │ (CSV / iCal / Excel)    │   │
│  └───────┬────────┘   └────────────┬────────────┘   │
│          │                         │                 │
│          ▼                         ▼                 │
│  ┌──────────────────────────────────────────────┐   │
│  │           Normalisation Layer                 │   │
│  │  (any format → canonical HolidayCalendar row) │   │
│  └───────────────────┬──────────────────────────┘   │
│                      │                              │
│                      ▼                              │
│  ┌──────────────────────────────────────────────┐   │
│  │           Overlay Merge Engine               │   │
│  │  base calendar + client overlay → effective  │   │
│  │  Conflicts: client wins, logged in changelog  │   │
│  └───────────────────┬──────────────────────────┘   │
│                      │                              │
│                      ▼                              │
│  ┌──────────────────────────────────────────────┐   │
│  │         Holiday Store (versioned)             │   │
│  └───────────────────┬──────────────────────────┘   │
│                      │                              │
│          Publishes: HolidayDataUpdated event         │
└──────────────────────────────────────────────────────┘
```

**Supported input formats at launch:** Copp Clark native API/file, CSV (date column), iCal (.ics), Excel (.xlsx).

### 5.3 Recompute Orchestrator

Triggered by two event types:

| Trigger | Scope of Recompute |
|---|---|
| `HolidayDataUpdated(calendarId, marketCode)` | All instruments for tenant whose `calendarRefs` include `marketCode` |
| `InstrumentTermsUpdated(instrumentId)` | That instrument only |

The orchestrator:
1. Fans out recompute jobs per instrument (async, queue-backed).
2. Each job: load current terms + holiday data, call Computation Engine over the standard forward horizon (e.g. 24 months).
3. Diff new vs. previous calendar entries → produce `EntryDiff[]`.
4. Upsert new `ComputedCalendar` version into Calendar Store.
5. If any diffs: publish `CalendarChanged` event with changelog payload.

**Idempotency:** jobs are keyed by `(instrumentId, termsVersion, holidayDataVersion)` — duplicate triggers are deduplicated.

### 5.4 API Gateway (internal)

Two API surfaces on one backend service, enforcing the same auth/tenancy:

| Surface | Transport | Cacheable? |
|---|---|---|
| Compute API | REST / gRPC | No (always live) |
| Calendar API | REST | Yes (ETags on calendar versions) |
| NL Query | REST (POST) | No |
| MCP Server | SSE (MCP protocol) | No |

---

## 6. API Design

### 6.1 Compute API

**`POST /v1/compute`**

Request:
```json
{
  "instrument_id": "uuid",
  "anchor_date": "2026-07-01",
  "horizon_days": 365,
  "holiday_version": "latest"  // or specific version hash
}
```

Response:
```json
{
  "instrument_id": "uuid",
  "computed_at": "2026-06-05T10:00:00Z",
  "holiday_version": "cc-2026-05-31",
  "entries": [
    {
      "dealing_date": "2026-07-31",
      "notification_deadline": "2026-07-10",
      "trade_date": "2026-07-31",
      "valuation_date": "2026-08-02",
      "settlement_date": "2026-08-05",
      "flags": ["rolled:modified_following", "holiday:NYSE:2026-07-04"]
    }
  ]
}
```

### 6.2 Calendar API

**`GET /v1/calendars/{instrumentId}`**

Query params: `from`, `to`, `version` (defaults to `current`), `page`, `page_size`

**`GET /v1/calendars/{instrumentId}/versions`** — list all historical versions

**`GET /v1/calendars/{instrumentId}/versions/{versionId}`** — retrieve a historical snapshot (audit)

**`GET /v1/changelog`**

Query params: `tenant_id`, `from`, `to`, `instrument_id` (optional filter)

**`POST /v1/calendars/export`**

Body: `{ instrument_ids: [...], format: "csv|xlsx|ical", from, to }`

### 6.3 Subscription / Webhook

**`POST /v1/subscriptions`**

```json
{
  "scope": { "instrument_ids": ["uuid-1", "uuid-2"] },
  "events": ["calendar.changed", "calendar.recomputed"],
  "delivery": { "type": "webhook", "url": "https://..." }
}
```

Internal Osyte services (Investment Planning, Trade Ops) subscribe via the event bus directly.

---

## 7. Holiday Data Subsystem

### 7.1 Copp Clark Integration

- **Pull mode:** scheduled connector polls Copp Clark API daily (configurable per tenant). Detects new data version by comparing version identifiers.
- **Push mode (future):** webhook from Copp Clark on file publish.
- On new data: normalise → merge with any client overlay → store versioned → publish `HolidayDataUpdated`.

### 7.2 Client File Upload

- Accepts: CSV, Excel, iCal.
- Validation: date format, year range, duplicate detection, market code mapping.
- Files stored in object storage with SHA-256 content hash as version key.
- Normaliser maps to canonical `HolidayCalendar` rows.

### 7.3 Composite Calendar / Overlay Merge

```
EffectiveCalendar(marketCode, tenant) =
  base_calendar(marketCode)                    // e.g. Copp Clark NYSE
  UNION client_overlay(marketCode, tenant)     // client's house exceptions
```

Conflict rule: if a date exists in the overlay, the overlay value wins regardless of base. Every conflict is logged as an `OverlayConflict` record and surfaced in the changelog.

### 7.4 Versioning

Every ingestion of holiday data (base or overlay) creates a new immutable version record:

```
HolidayVersion
├── versionId   String  (SHA-256 of content or vendor version string)
├── source      Enum
├── marketCode  String
├── ingestedAt  Timestamp
└── dataFile    S3 URL
```

Computation Engine always takes an explicit `HolidayVersion` — no implicit "current." The Calendar Store records which `holidayDataVersion` produced each calendar snapshot.

---

## 8. Event & Change Propagation

### 8.1 Events

| Event | Producer | Consumers |
|---|---|---|
| `HolidayDataUpdated` | Holiday Subsystem | Recompute Orchestrator |
| `InstrumentTermsUpdated` | Instrument Registry | Recompute Orchestrator |
| `CalendarRecomputed` | Recompute Orchestrator | Internal consumers |
| `CalendarChanged` | Recompute Orchestrator (when diffs exist) | Investment Planning, Rebalancing, Trade Ops, Webhook delivery |

### 8.2 Changelog Schema (published with `CalendarChanged`)

```json
{
  "change_id": "uuid",
  "tenant_id": "uuid",
  "triggered_by": "holiday_update",
  "trigger_ref": "cc-2026-05-31",
  "published_at": "2026-06-01T06:00:00Z",
  "summary": {
    "instruments_affected": 47,
    "entries_changed": 134
  },
  "diffs": [
    {
      "instrument_id": "uuid",
      "dealing_date": "2026-11-27",
      "field": "notification_deadline",
      "previous_value": "2026-11-06",
      "new_value":      "2026-11-05",
      "reason": "holiday:NYSE:2026-11-11 removed from calendar"
    }
  ]
}
```

### 8.3 Message Bus

Use the existing Osyte event bus (Kafka / equivalent). Topics:

- `liquidity-calendar.holiday-updates`
- `liquidity-calendar.recompute-jobs` (partitioned by `tenantId`)
- `liquidity-calendar.calendar-changed`

---

## 9. NL Query & MCP Layer

### 9.1 Natural-Language Query

- Sits on top of Calendar Store read path — never touches the Computation Engine.
- LLM receives the user's question + a tool schema for structured Calendar API calls.
- LLM translates question → Calendar API query → fetches data → synthesises answer.
- **Guardrail:** LLM output is never trusted for date arithmetic. All numbers come from the Calendar Store.

Example translations:

| User question | Structured query |
|---|---|
| "Next redemption notification deadlines across the alternatives book" | `GET /v1/calendars?portfolio=alternatives&field=notification_deadline&from=today&to=today+30d` |
| "Instruments with key dates in Thanksgiving week" | `GET /v1/calendars?from=2026-11-23&to=2026-11-27&fields=all` |
| "Did any dates move after the last Copp Clark update?" | `GET /v1/changelog?trigger_ref=cc-latest&from=-7d` |

### 9.2 MCP Server

Expose the following MCP tools (per MCP spec):

```
tool: get_instrument_calendar
  params: instrument_id, from, to, version?
  returns: CalendarEntry[]

tool: search_calendar
  params: portfolio?, fund?, from, to, fields?, limit
  returns: CalendarEntry[]

tool: get_changelog
  params: from, to, instrument_id?
  returns: CalendarChangelog[]

tool: get_next_key_dates
  params: instrument_id, n_results
  returns: CalendarEntry[]  // next N upcoming entries
```

Auth: same RBAC as Calendar API — MCP client must present a valid Osyte token with `calendar:read` scope.

---

## 10. Data Storage Strategy

### 10.1 Technology Choices

| Store | Technology | Rationale |
|---|---|---|
| Instrument terms | Existing Instrument DB (PostgreSQL) | Already owns instrument data; add `liquidity_terms` table |
| Holiday data | PostgreSQL (holiday rows) + S3 (raw files) | Structured queries on dates; raw files preserved for audit |
| Computed calendars | PostgreSQL (time-series optimised, partitioned by tenant + year) | Rich query, ETag versioning, audit history |
| Changelog | PostgreSQL + published to event bus | Queryable history; streamed downstream |
| Recompute queue | Kafka (or existing Osyte queue) | Fan-out, deduplication, backpressure |
| NL query cache | Redis (short TTL) | Avoid re-running identical NL→API translations |

### 10.2 Calendar Store Schema (simplified)

```sql
-- One row per instrument per dealing date per version
CREATE TABLE calendar_entries (
  tenant_id            UUID NOT NULL,
  instrument_id        UUID NOT NULL,
  terms_version        INT  NOT NULL,
  holiday_data_version TEXT NOT NULL,
  generated_at         TIMESTAMPTZ NOT NULL,
  is_current           BOOLEAN NOT NULL DEFAULT TRUE,

  dealing_date          DATE NOT NULL,
  notification_deadline DATE,
  trade_date            DATE,
  valuation_date        DATE,
  settlement_date       DATE,
  flags                 JSONB,

  PRIMARY KEY (tenant_id, instrument_id, terms_version, holiday_data_version, dealing_date)
) PARTITION BY RANGE (dealing_date);

CREATE INDEX idx_calendar_current
  ON calendar_entries (tenant_id, instrument_id, dealing_date)
  WHERE is_current = TRUE;
```

`is_current = FALSE` rows are historical snapshots retained for audit (`GET /versions/{versionId}`).

---

## 11. Non-Functional Requirements

### 11.1 Performance

| Scenario | Target |
|---|---|
| Compute API single instrument (1-year horizon) | p95 < 100 ms |
| Calendar API read (paginated, 1 year, current) | p95 < 50 ms |
| Full recompute on holiday update (1,000 instruments) | < 5 min end-to-end |
| Changelog publish lag after recompute | < 30 s |

### 11.2 Correctness

- Computation Engine has a dedicated golden-dataset test suite: known liquidity terms × known holiday calendars → expected output dates. Maintained as checked-in fixtures; run on every PR.
- Parity tests assert `Compute API response == Calendar Store current row` for a random sample of (instrument, date) pairs daily.

### 11.3 Multi-Tenancy

- All DB queries include `tenant_id` predicate. Row-level security (RLS) enforced at DB level.
- Holiday calendars: global/vendor calendars visible to all tenants; client overlays scoped to `tenant_id`.
- MCP tokens carry `tenant_id` claim; server enforces it on every tool call.

### 11.4 Observability

- **Metrics:** recompute job lag, compute API latency histogram, calendar freshness (time since last recompute per instrument), changelog event publish lag.
- **Alerts:** recompute job DLQ depth > 0 (failed jobs), compute API error rate > 0.1%, calendar freshness > 1 hour after a holiday update.
- **Audit log:** every holiday ingestion, every recompute, every date that changed — immutable, retained per compliance policy.

---

## 12. Tech Stack

```
Language:        TypeScript / Node.js  (or existing Osyte backend stack)
Framework:       NestJS (or equivalent)
Database:        PostgreSQL 15+ (partitioned calendar_entries, RLS)
Object Store:    S3-compatible (raw holiday files)
Message Bus:     Kafka (or existing Osyte bus)
Cache:           Redis 7
MCP Protocol:    @anthropic/mcp-sdk (SSE transport)
Holiday APIs:    Copp Clark REST API + file connectors
Testing:         Jest + golden-dataset fixture suite
Observability:   OpenTelemetry → existing Osyte observability stack
```

The Computation Engine is implemented as a standalone, framework-free module — pure functions, no DB dependency — so it can be independently unit-tested and potentially extracted to a shared library.

---

## 13. Phased Delivery Plan

### Phase 1 — Core Engine & Compute API (Weeks 1–6)

**Goal:** Computation Engine is live; Investment Planning can call it for real-time dates.

| Deliverable | Notes |
|---|---|
| Liquidity Terms schema + migration | Extend Instrument Registry DB |
| Computation Engine (pure module) | Notification, trade, valuation, settlement offsets; all 4 roll conventions |
| Multi-calendar AND logic | Cross-currency/cross-market instruments |
| Golden-dataset test suite | Minimum 50 instrument × calendar combinations |
| Copp Clark connector (poll mode) | Normalisation + versioned Holiday Store |
| CSV/Excel/iCal client file upload | Validation, normalisation, S3 storage |
| `POST /v1/compute` API | Real-time endpoint, <100ms |
| Basic auth/tenancy enforcement | tenant_id scoping |

**Exit criteria:** Investment Planning team can call `/v1/compute` and get correct dates against a live Copp Clark calendar for a sample of real instruments.

---

### Phase 2 — Persisted Calendar & Changelog (Weeks 7–12)

**Goal:** Calendar Store is live; Rebalancing and Trade Ops can read current calendars; changelog is published on updates.

| Deliverable | Notes |
|---|---|
| Calendar Store schema + partitioning | `calendar_entries` table, `is_current` index |
| Recompute Orchestrator | Event-driven fan-out, idempotent jobs |
| `HolidayDataUpdated` trigger + pipeline | Full recompute on holiday change |
| `InstrumentTermsUpdated` trigger | Recompute on terms change |
| `GET /v1/calendars/{id}` API | Current view, paginated |
| `GET /v1/changelog` API | Full changelog query |
| `CalendarChanged` event publication | Kafka; Investment Planning + Trade Ops subscribe |
| Webhook delivery for external subscribers | `POST /v1/subscriptions` |
| Parity test (Compute == Calendar Store) | Daily scheduled assertion |

**Exit criteria:** Holiday update from Copp Clark triggers automatic recompute of all affected instruments within 5 minutes; changelog is accurate and consumable by downstream services.

---

### Phase 3 — Overlay, Audit & Export (Weeks 13–16)

**Goal:** Client-specific holiday overlays are supported; audit/historical queries work; calendar export is available.

| Deliverable | Notes |
|---|---|
| Overlay Merge Engine | Base + client overlay; conflict logging |
| Client overlay upload & management | CRUD for overlay calendars |
| Historical calendar versions | `GET /v1/calendars/{id}/versions/{versionId}` |
| Effective-dated terms versioning | `effectiveFrom/effectiveTo` on liquidity terms |
| `POST /v1/calendars/export` | CSV, Excel, iCal |
| Retroactive holiday change handling | Correct version reconstruction for audit |
| Observability dashboards | Recompute lag, calendar freshness, error rates |

**Exit criteria:** A client can upload their own holiday exception list; the overlay is applied; audit team can retrieve "what did the calendar say on [past date]."

---

### Phase 4 — NL Query & MCP Server (Weeks 17–20)

**Goal:** Ops and PM users can query calendars in natural language; AI agents can consume key dates via MCP.

| Deliverable | Notes |
|---|---|
| NL Query endpoint | LLM → structured Calendar API call → synthesised response |
| MCP server (4 tools: get, search, changelog, next_dates) | SSE transport, Osyte token auth |
| RBAC: `calendar:read` scope | Enforced on API + MCP |
| Redis NL query cache | Short TTL to avoid redundant LLM calls |
| MCP integration with Osyte AI assistant | Internal dogfooding |

**Exit criteria:** Ops team can ask "which instruments have notification deadlines this week?" in plain English and get a correct, sourced answer. An AI agent with an Osyte token can read key dates via MCP.

---

### Phase 5 — AI Term Extraction (Roadmap, Weeks 21+)

**Goal:** AI-assisted extraction of liquidity terms from PDFs (PPMs, side letters) with mandatory human review before persistence.

| Deliverable | Notes |
|---|---|
| PDF ingestion pipeline | Upload PPM/side letter → extract structured terms |
| Human review UI | Extracted terms shown for approval before saving |
| Audit trail for AI-extracted terms | Who approved, what was changed |

---

## 14. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Copp Clark API changes format or authentication | Medium | High | Connector abstraction layer; integration tests run nightly against Copp Clark sandbox |
| Retroactive holiday changes break existing settled trades | Low | High | Effective-dated versioning; clear UI messaging that changes apply prospectively unless explicitly back-filled |
| Recompute fan-out overwhelms DB on large tenant (10k+ instruments) | Medium | Medium | Partition recompute queue by tenant; rate-limit jobs per tenant; pre-scale partitioned table |
| Compute API and Calendar Store drift (date mismatch) | Low | High | Daily parity assertion test + alert on any mismatch |
| Multi-calendar AND logic produces unexpectedly few business days for cross-currency instruments | Medium | Medium | Include cross-market scenarios in golden-dataset suite; expose `flags` in API response to make calendar intersections visible |
| Client uploads garbage holiday files | High | Low | Strict validation on upload (date range, format, market code whitelist); validation errors returned before storage |
| NL layer returns confident but wrong date context | Medium | High | NL layer never performs arithmetic — all data from Calendar Store; LLM only formats/narrates |

---

## 15. Open Questions

| # | Question | Owner | Priority |
|---|---|---|---|
| OQ-01 | What is the standard forward horizon for materialized calendars? (24 months assumed) | Product / Ops | High |
| OQ-02 | What is the exact Copp Clark API/delivery mechanism available to Osyte? (REST API vs. file delivery) | Engineering | High |
| OQ-03 | Which downstream systems (Investment Planning, Rebalancing, Trade Ops) are already event-bus consumers vs. need new webhook integration? | Architecture | High |
| OQ-04 | What is the expected instrument count per tenant at P50 and P99? (drives recompute queue sizing) | Product | Medium |
| OQ-05 | Is there an existing Instrument Registry service with a terms schema, or does this service own the full instrument data model? | Architecture | High |
| OQ-06 | Compliance: what is the data retention policy for historical calendar versions and changelogs? | Legal / Compliance | Medium |
| OQ-07 | Which Osyte auth system issues the `calendar:read` scope for MCP tokens? | Platform / Security | Medium |
| OQ-08 | For the NL layer, which LLM endpoint does Osyte use internally? Is there an approved model/provider? | Platform | Low |
| OQ-09 | Are there any instruments where settlement cycles or valuation lags are dynamic (e.g., vary by NAV gate)? | Product | Medium |

---

*Document version 1.0 — prepared June 2026*
*Based on: [Liquidity Calendar Service spec — Confluence](https://osyte.atlassian.net/wiki/spaces/ZOTBPB/pages/1309573122/Liquidity+Calendar+Service)*
