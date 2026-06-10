# LCS Architecture

## Running Examples

Two funds are used throughout this document to explain every component:

**Fund A — Simple (any listed asset like a stock or ETF):** Daily dealing. No lockup, no gates, no holdback. No notice period. T+1 settlement. Business day centres: `["London"]`.

**Fund B — Complex:** Quarterly dealing (1st business day of each quarter). 12-month hard lockup from subscription. 25% investor gate per quarter. 5% audit holdback on redemptions ≥95% of account. 30-day notice, 30-day settlement. Business day centres: `["New York", "Cayman Islands"]`.

---

## 1. System Overview

LCS reads fund liquidity terms and holiday calendars from OSYTE's existing platform, computes lifecycle dates, and simulates redemption schedules. It does not store source data — it reads from OSYTE and only persists materialized calendars.

Two APIs, one shared engine:

| API | Purpose |
|---|---|
| **Compute API** | Date math + liquidation planning. `/compute` returns lifecycle dates, `/plan` adds gates/holdbacks/lockups on top. |
| **Calendar API** | Persisted forward-looking calendars per instrument, with changelogs when dates move. |

Both APIs use the same **Compute Engine** module for all date math. The Compute API exposes it directly. The Calendar API calls it internally to materialize calendars. The `/plan` routes read constraints first, then call the same engine per tranche.

```
OSYTE Platform (existing)
  │
  ├── Holiday Data ──→ Holiday Resolver (cached)──┐
  │                                                ├──→ Compute Engine (shared module)
  └── Fund Terms ────→ Terms Reader ──────────────┘          │
                            │                                 │
                            │                      ┌──────────┴──────────┐
                            │                      │                     │
                            │               Compute API            Calendar API
                            │                │                     │
                            │          ┌─────┴─────┐         Calendar Store
                            │          │           │
                            └───→ /compute     /plan
                          (for constraints)
```

---

## 2. Holiday Resolver

Reads holiday data from OSYTE's DB and merges it into a resolved holiday set per request. LCS does not store or manage holiday data.

**Steps:**
1. Fetch all holidays for the fund's business centres (Copp Clark base + weekends)
2. Fetch all applicable tenant overlays (multiple supported — firm-level, fund-level, etc.)
3. Merge: base → overlays in precedence order → resolved set

**Caching:** Base calendars for popular centres (US, GB) are cached long-lived. Resolved sets (base + tenant overlays) are cached with short TTL. Cache is invalidated when OSYTE signals a data update.

**Centre aliases:** Fund terms say "Cayman Islands"; Copp Clark says "George Town". The Holiday Resolver resolves this transparently via OSYTE's alias table.

### Example — Fund A (London)

```
Fetch holidays for London (2026)
  → London: Jan 1, Apr 10, Apr 13, May 8, May 25, Aug 31, Dec 25, Dec 26 ...
Fetch tenant overlays → none
Merge → ResolvedHolidaySet
  isBusinessDay("2026-07-01", ["London"]) → true
```

### Example — Fund B (New York, Cayman Islands)

```
Resolve alias: "Cayman Islands" → "George Town" (center_id 42)
Fetch holidays for New York + George Town (2026)
  → New York: Jan 1, Jan 19, Feb 16, May 25, Jul 3, Sep 7, Nov 26, Dec 25 ...
  → George Town: Jan 1, Jan 26, May 18, Jul 6, Nov 9, Dec 25 ...
Fetch tenant overlays → firm-level: adds Nov 27 (day after Thanksgiving) to New York
Merge → base + overlay add
  isBusinessDay("2026-11-27", ["New York", "Cayman Islands"]) → false (overlay holiday in NY)
```

---

## 3. Terms Reader

Reads fund liquidity terms from OSYTE on demand. No local storage. Fetches by `instrument_id` or `fund_id`.

### Example — Fund A

```json
{
  "dealing_basis": "periodic",
  "dealing_interval": {"count": 1, "unit": "day"},
  "notice_period": {"days": 0, "availability": "not_applicable"},
  "settlement": {"days": 1, "day_type": "business", "direction": "after"},
  "gates": [],
  "restrictions": {"lockup_provisions": {"no_lockup": true}, "audit_holdbacks": {"holdback_applies": false}}
}
```

### Example — Fund B

```json
{
  "dealing_basis": "periodic",
  "dealing_interval": {"count": 3, "unit": "month"},
  "dealing_day": {"anchor": "first", "day_type": "business"},
  "notice_period": {"days": 30, "day_type": "calendar", "direction": "before"},
  "settlement": {"days": 30, "day_type": "calendar", "direction": "after"},
  "gates": [{"gate_level": "investor_level", "threshold_pct": 25, "measurement_period": "quarterly"}],
  "restrictions": {
    "lockup_provisions": {"hard_lockup": {"duration": {"count": 12, "unit": "month"}, "start_basis": "subscription_day"}},
    "audit_holdbacks": {"holdback_applies": true, "holdback_tiers": [{"condition": "redemption_gte_pct_nav", "threshold_pct": 95, "holdback_pct": 5}]}
  }
}
```

---

## 4. Compute Engine (shared module)

Used by both the Compute API and the Calendar API. Takes terms + holidays + anchor + roll convention → produces lifecycle dates. Pure function — same inputs always produce same outputs.

### Algorithm

1. **Find the nearest dealing date** from the anchor
2. **Compute the full lifecycle chain** for that date (notice deadline, settlement, etc.)
3. **Check if it satisfies the anchor constraint** — yes → keep; no → skip to next dealing date
4. **Repeat** until N results collected

| `anchor_type` | "Nearest" means | Check |
|---|---|---|
| `as_of` | Next dealing date ≥ anchor | Always passes |
| `target_settlement_date` | Nearest dealing date before anchor | `settlement_date ≤ target`? |
| `target_dealing_date` | Nearest valid dealing date to given date | Snap if not valid, warn |
| `target_notice_deadline` | Next dealing date after anchor | `notice_deadline ≥ anchor`? |

**Multiple dealing days:** A fund can have multiple dealing days per period (e.g. 1st and 15th, stored as `%+%` in OSYTE). The engine iterates all per period, returns results chronologically sorted.

**Business day adjustment:** If a date lands on a holiday/weekend, the roll convention adjusts it (Following, Modified Following, Preceding, Modified Preceding). Default: Modified Following. When multiple centres apply, a date must be a business day in all of them.

> **Known schema gap:** v15.5 has no `roll_convention` field. Currently an API parameter.

**Centre precedence:** Rule-level `business_day_centers` (e.g. on notice_period) override instrument-level for that calculation.

**Completeness gate:** Before computing, the engine checks `availability` on every field. `unknown` / `not_assessed` → null + warning. When `value_type` is `minimum` / `estimated` / `discretionary`, the result is flagged as approximate.

### Example — Fund A: "When's my next redemption?"

Request: `anchor_type=as_of, anchor_date=2026-07-01`

```
Find nearest dealing date ≥ Jul 1 → Jul 1 (daily dealing, it's a business day in London)
Compute chain:
  Notice: n/a (no notice period for listed assets)
  Settlement: Jul 1 + 1 business day = Jul 2

Result:
  Dealing: 2026-07-01 | Notice: n/a | Cash: 2026-07-02
```

### Example — Fund B: "I need cash by October 31st"

Request: `anchor_type=target_settlement_date, anchor_date=2026-10-31`

```
Find nearest dealing date before Oct 31:
  Q4 dealing date = Oct 1 (1st business day of Q4)
  Compute chain:
    Notice deadline: Oct 1 − 30 calendar days = Sep 1
    Settlement: Oct 1 + 30 calendar days = Oct 31
  Check: settlement (Oct 31) ≤ target (Oct 31)? → Yes ✓

Result:
  Dealing: 2026-10-01 | Notice by: 2026-09-01 | Cash: 2026-10-31

(If target were Oct 15, settlement would be Oct 31 > Oct 15 → fail.
 Engine would try the previous quarter: Jul 1 → settlement Jul 31 ≤ Oct 15 → pass.)
```

---

## 5. Workflow — Compute API

The Compute API exposes the Compute Engine directly. Stateless — no persistence, no side effects.

The `/compute` routes handle pure date queries. The `/plan` routes read constraints first, adjust inputs, then call the same Compute Engine per tranche.

### Workflow — `/compute` (date math)

```mermaid
sequenceDiagram
    participant Client
    participant API as Compute API
    participant TR as Terms Reader
    participant HR as Holiday Resolver
    participant DB as OSYTE DB
    participant CE as Compute Engine

    Client->>API: POST /compute<br/>{instrument_id, anchor_type,<br/> anchor_date, tenant_id}

    API->>TR: getTerms(instrument_id)
    TR-->>API: instrument terms (v15.5)<br/>includes business_day_centers

    API->>HR: resolveHolidays(tenant_id,<br/> business_day_centers, date_range)
    HR->>DB: fetch holidays + overlays<br/>for relevant centres
    DB-->>HR: rows
    Note over HR: Merge: base + overlays<br/>= resolved holiday set
    HR-->>API: ResolvedHolidaySet

    API->>CE: computeDates(terms, holidays,<br/> anchor, roll_convention)

    Note over CE: Find nearest dealing date<br/>→ compute lifecycle chain<br/>→ check constraint<br/>→ skip or keep

    CE-->>API: lifecycle dates
    API-->>Client: 200 OK {dates, dataset_version}
```

Every response includes `dataset_version: {holiday_file_id, terms_version, overlay_hash}` for reproducibility.

### Workflow — `/plan` (liquidation planning)

The `/plan` routes read the fund's constraints **first**, adjust inputs, then call the Compute Engine per tranche.

```mermaid
sequenceDiagram
    participant Client
    participant API as Compute API (/plan)
    participant TR as Terms Reader
    participant CE as Compute Engine

    Client->>API: POST /plan/redeem<br/>{instrument_id, desired_amount,<br/> position_nav, as_of_date}

    API->>TR: getTerms(instrument_id)
    TR-->>API: terms (gates, holdbacks,<br/>lockup, redemption)

    Note over API: Read all constraints and adjust inputs:<br/>— Lockup: shift anchor if locked<br/>— Gates: split amount into chunks<br/>— Holdback: flag if threshold crossed<br/>— Adjust notice period for amount

    loop For each tranche (with adjusted inputs)
        API->>CE: computeDates(adjusted_anchor,<br/>adjusted_notice)
        CE-->>API: dealing date + notice + settlement
        Note over API: Record tranche,<br/>advance anchor to next period,<br/>reduce remaining amount
    end

    API-->>Client: 200 OK {tranches[], summary}
```

### Example — Fund A: Redeem $1M, as of 2026-07-01

```
Read constraints:
  Lockup?    No   → no adjustment
  Gates?     None → no split
  Holdback?  No   → skip

Nothing to adjust — pass straight through to Compute Engine.

Call Compute Engine: anchor=2026-07-01, no notice
  → Dealing: Jul 1 | Notice: n/a | Settlement: Jul 2

Result: 1 tranche
┌─────────┬────────────┬────────┬─────────┬────────┐
│ Tranche │ Amount     │ Notice │ Dealing │ Cash   │
├─────────┼────────────┼────────┼─────────┼────────┤
│ 1       │ $1,000,000 │ n/a    │ Jul 01  │ Jul 02 │
└─────────┴────────────┴────────┴─────────┴────────┘
```

### Example — Fund B: Redeem $5M from $8M position, subscribed 2025-01-15, as of 2026-07-01

```
Read constraints:
  Lockup?    Hard, 12 months from 2025-01-15 → expires 2026-01-15
             as_of (Jul 1) > expiry (Jan 15) → unlocked. Proceed.
             (If as_of were 2025-06-01, anchor would shift to 2026-01-15.)
  Gates?     25% of holding per quarter → $2M max/quarter
             $5M ÷ $2M = 3 tranches needed
  Holdback?  5% if ≥ 95% of account
             $5M / $8M = 62.5% → NOT triggered

Compute Engine calls (one per tranche):
  T1: anchor=2026-07-01, notice=30 days
      → Dealing: Oct 1 | Notice: Sep 1 | Settlement: Oct 31
  T2: anchor=2026-10-02, notice=30 days
      → Dealing: Jan 2 | Notice: Dec 3 | Settlement: Feb 1
  T3: anchor=2027-01-03, notice=30 days
      → Dealing: Apr 1 | Notice: Mar 2 | Settlement: May 1

Result: 3 tranches
┌─────────┬────────────┬────────┬─────────┬────────┬───────────┐
│ Tranche │ Amount     │ Notice │ Dealing │ Cash   │ Gate-ltd? │
├─────────┼────────────┼────────┼─────────┼────────┼───────────┤
│ 1       │ $2,000,000 │ Sep 01 │ Oct 01  │ Oct 31 │ Yes       │
│ 2       │ $2,000,000 │ Dec 03 │ Jan 02  │ Feb 01 │ Yes       │
│ 3       │ $1,000,000 │ Mar 02 │ Apr 01  │ May 01 │ No        │
└─────────┴────────────┴────────┴─────────┴────────┴───────────┘

Total: $5M | First cash: Oct 31 | Last cash: May 1 | Holdback: $0 | Exit fee: $0
```

---

## 6. Workflow — Calendar API

Separate API from Compute. Uses the same Compute Engine module to materialize forward-looking calendars, then serves them from the Calendar Store.

### Materialization

When triggered (terms change, holiday update, scheduled refresh), the Calendar API:
1. Calls the Compute Engine for every dealing date in the horizon (default: 24 months)
2. Writes results to the Calendar Store (effective-versioned — old versions never deleted)
3. Diffs against the previous version → changelog

Materialization is async — returns `202 Accepted` with a `job_id`. Poll `GET /jobs/{job_id}` for status.

### Example — Fund A calendar (excerpt)

```
2026-07-01  Settlement: Jul 02
2026-07-02  Settlement: Jul 03
2026-07-03  Settlement: Jul 06  (Fri → settles Mon)
...
(one row per business day — daily dealing, no notice period)
```

### Example — Fund B calendar (excerpt)

```
2026-10-01 (Q4)  Notice: Sep 01  |  Settlement: Oct 31
2027-01-02 (Q1)  Notice: Dec 03  |  Settlement: Feb 01
2027-04-01 (Q2)  Notice: Mar 02  |  Settlement: May 01
2027-07-01 (Q3)  Notice: Jun 01  |  Settlement: Jul 31
```

When Copp Clark adds a holiday on Sep 1 (New York), the calendar auto-recomputes. Changelog: "Fund B Q4 notice deadline moved from Sep 1 → Aug 29 (Sep 1 is now a holiday)."

### Calendar Store

The one thing LCS owns. Effective-versioned: every materialization creates a new version, enabling "what did the calendar say on date X?" queries for audit.

```sql
CREATE TABLE calendar_versions (
    version_id      BIGSERIAL PRIMARY KEY,
    tenant_id       VARCHAR(50) NOT NULL,
    instrument_id   VARCHAR(50) NOT NULL,
    effective_from  TIMESTAMPTZ NOT NULL,
    superseded_at   TIMESTAMPTZ,
    trigger_reason  VARCHAR(30) NOT NULL,
    dataset_version JSONB NOT NULL,
    horizon_from    DATE NOT NULL,
    horizon_to      DATE NOT NULL,
    UNIQUE (tenant_id, instrument_id, effective_from)
);

CREATE TABLE calendar_dates (
    id                  BIGSERIAL PRIMARY KEY,
    version_id          BIGINT NOT NULL REFERENCES calendar_versions(version_id),
    scope               VARCHAR(12) NOT NULL,
    dealing_day_label   VARCHAR(50),
    dealing_date        DATE NOT NULL,
    notice_deadline     DATE,
    settlement_date     DATE,
    document_deadline   DATE,
    cash_funding_deadline DATE,
    nav_pricing_cutoff  DATE,
    cutoff_time         TIME,
    cutoff_timezone     VARCHAR(50),
    roll_applied        VARCHAR(20),
    unadjusted_date     DATE
);

CREATE TABLE calendar_changelog (
    id                  BIGSERIAL PRIMARY KEY,
    version_id          BIGINT NOT NULL REFERENCES calendar_versions(version_id),
    previous_version_id BIGINT REFERENCES calendar_versions(version_id),
    scope               VARCHAR(12) NOT NULL,
    dealing_date        DATE NOT NULL,
    field_name          VARCHAR(30) NOT NULL,
    previous_value      DATE,
    new_value           DATE,
    reason              TEXT
);
```

---

## 7. Key Decisions

| # | Decision | Why |
|---|---|---|
| 1 | **LCS reads from OSYTE, owns nothing except Calendar Store** | Single source of truth. No data duplication. |
| 2 | **Compute Engine is a shared module** | Both the Compute API (`/compute` + `/plan`) and the Calendar API use the same engine. No duplicated date logic. |
| 3 | **Compute API and Calendar API are separate** | Compute is stateless, real-time. Calendar is persisted, async materialization. Different consumers, different scaling, different deployment cadence. |
| 4 | **`/compute` and `/plan` are in the same API** | They share the same engine, same terms, same holidays. Planning just reads constraints first and adjusts inputs before calling the engine. No reason to separate them. |
| 5 | **Planning reads constraints first, then calls Compute Engine** | Constraints change the inputs (anchor shift for lockup, amount split for gates, notice selection for tiered amounts). The Compute Engine never needs to know about constraints. |
| 6 | **Holiday Resolver caches popular centres** | US/GB base calendars cached long-lived. Resolved sets cached short-lived. Only cold centres hit the DB. |
| 7 | **Calendar Store is effective-versioned** | Supports "what did the calendar say on date X?" for audit. Old versions never deleted. |
| 8 | **Dataset version stamp on every response** | `{holiday_file_id, terms_version, overlay_hash}` enables reproducibility and audit. |
| 9 | **Find-check-skip algorithm** | All anchor modes use the same loop. Offsets + rolls are non-invertible, so we always compute forward and verify. |

---

## 8. Security

| Concern | Approach |
|---|---|
| **Authentication** | Bearer token validated against OSYTE's auth service |
| **Tenant isolation** | `tenant_id` verified against token claims. Tenant A cannot access tenant B's overlays or calendars |
| **Holiday isolation** | Base Copp Clark is shared (read-only). Overlays scoped by tenant. |
| **Calendar isolation** | Keyed by `(tenant_id, instrument_id)` |
| **Planning data** | Position data (NAV, amounts) is never persisted — accepted, used, discarded |
