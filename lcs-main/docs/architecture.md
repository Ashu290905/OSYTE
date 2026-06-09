# LCS Architecture & Workflow Design

## 1. System Overview

LCS is a **deterministic date-computation service** that reads fund liquidity terms and market holiday calendars from OSYTE's existing platform and computes canonical lifecycle dates for every instrument. LCS does not own or store the source data — it reads from OSYTE and only persists its own computed output (materialized calendars).

The system is split into a **core layer** (date computation — not debated) and an **optional planning layer** (liquidation simulation through gates/holdbacks — under discussion). The architecture treats these as cleanly separable: the planning layer consumes the core layer's output but never contaminates it.

```mermaid
graph TD
    subgraph OSYTE["OSYTE Platform (existing)"]
        HD[("Holiday Data<br/>Copp Clark + overlays<br/>+ aliases + weekend rules")]
        FT[("Fund Liquidity Terms<br/>v15.5 records")]
    end

    subgraph LCS["LCS Service"]
        subgraph CORE["CORE (deterministic)"]
            HR["Holiday Resolver<br/>(fetches & resolves<br/>per tenant, cached)"]
            TR["Terms Reader<br/>(fetches by instrument)"]
            CE["Compute Engine<br/>Anchor resolver | Roll conventions<br/>Biz-day math | Dealing-date generation"]
            CA["Compute API<br/>(stateless)"]
            CLA["Calendar API<br/>(persisted)"]

            HR --> CE
            TR --> CE
            CE --> CA
            CE --> CLA
        end

        subgraph OPT["OPTIONAL — Liquidation Planning (under discussion)"]
            PE["Planning Engine<br/>gates, holdbacks, lockups"]
            PA["Planning API"]
            PE --> PA
        end

        CA -.-> PE
    end

    HD -->|read| HR
    FT -->|read| TR

    style OPT stroke-dasharray: 5 5
```

---

## 2. Component Architecture

### 2.1 Holiday Resolver

Reads holiday data from OSYTE's existing DB (Copp Clark base holidays, tenant overlays, centre aliases, weekend rules) and merges it into a resolved holiday set for each computation request. LCS does not ingest, store, or manage any holiday data — that's all handled by the OSYTE platform.

#### Holiday Resolution

When the Compute or Calendar API processes a request, the Holiday Resolver fetches the relevant data from OSYTE's DB and resolves it for the specific tenant + fund combination:

```mermaid
sequenceDiagram
    participant DE as Compute Engine
    participant HR as Holiday Resolver
    participant DB as OSYTE DB<br/>(existing)

    Note over DE: Fund C.444 needs business_day_centers:<br/>["New York", "Cayman Islands"]

    HR->>HR: Check cache for (tenant, centers, year)

    alt Cache hit
        HR-->>DE: ResolvedHolidaySet (from cache)
    else Cache miss
        HR->>DB: 1. Resolve aliases<br/>"Cayman Islands" → center_id 42 (George Town/KY)<br/>"New York" → center_id 17
        DB-->>HR: center_ids: [17, 42]

        HR->>DB: 2. Fetch base holidays<br/>for center_ids IN (17, 42)<br/>within date range
        DB-->>HR: base holiday rows

        HR->>DB: 3. Fetch tenant overlays<br/>for tenant + center_ids<br/>within date range
        DB-->>HR: overlay rows (adds + removes)

        HR->>DB: 4. Fetch weekend rules<br/>for center_ids IN (17, 42)
        DB-->>HR: weekend patterns

        Note over HR: Merge:<br/>base holidays<br/>+ overlay adds<br/>− overlay removes<br/>+ weekend rules<br/>= resolved holiday set

        HR->>HR: Cache resolved set

        HR-->>DE: ResolvedHolidaySet<br/>.isHoliday(date, center)<br/>.isBusinessDay(date, center)<br/>.nextBusinessDay(date, center, convention)
    end
```

The returned `ResolvedHolidaySet` is an in-memory object that the Compute Engine uses for all business-day calculations during that computation. This means:
- Different tenants get different holiday sets (same Copp Clark base, different overlays)
- Only the centres relevant to the fund are fetched, not the entire 400K-row dataset

#### Caching

A small number of centres dominate traffic — US (New York) and GB (London) appear in the vast majority of fund terms. Hitting the DB for these on every request is wasteful.

The Holiday Resolver maintains a **two-tier cache**:

| Tier | What's cached | Key | TTL | Invalidation |
|---|---|---|---|---|
| **Base calendar cache** | Copp Clark holidays for a centre + date range | `(center_id, year)` | Long-lived (until OSYTE signals a Copp Clark update) | Evict entries for affected centres when notified of a data update |
| **Resolved set cache** | Fully merged holiday set (base + tenant overlay) for a tenant + centre + date range | `(tenant_id, center_id, year)` | Short-lived (minutes) or event-driven | Evict when notified of an overlay change for that tenant + centre |

**How it works:**
1. Request comes in for tenant `acme`, centres `["New York", "London"]`, date range 2026
2. Check resolved set cache for `(acme, new_york, 2026)` and `(acme, london, 2026)` — **cache hit** on most requests since these are the popular centres
3. On cache miss: check base calendar cache for `(new_york, 2026)` — almost always warm since US/GB base calendars are requested constantly
4. Fetch tenant overlays from OSYTE DB (these are small — typically 0–5 rows per tenant per centre)
5. Merge, cache the resolved set, return

Weekend rules and centre aliases are cached indefinitely (they change approximately never).

Since most tenants have few or zero overlays for the popular centres, the resolved set cache has a very high hit rate — the base calendar is shared, and the overlay diff is tiny.

**Key design decisions:**

| Concern | Decision |
|---|---|
| **Read-only — LCS doesn't own holiday data** | All holiday data (Copp Clark, overlays, aliases, weekend rules) lives in OSYTE's existing DB. The Holiday Resolver is a read-only client with caching. |
| **Two-tier cache for popular centres** | Base calendars (US, GB, etc.) are cached long-lived. Resolved sets (base + tenant overlay) are cached with short TTL or event-driven invalidation. Most requests hit cache — DB is only touched for cold centres or after data changes. |
| **Only relevant centres are fetched** | A fund with `business_day_centers: ["New York", "Cayman Islands"]` triggers a query for 2 centres, not 417. Keeps queries fast. |
| **Centre alias resolution** | Fund terms say "Cayman Islands"; OSYTE's alias table maps it to "George Town" (CenterID 42). The Holiday Resolver handles this transparently — no other component needs to know about the mismatch. |
| **Overlay semantics** | An overlay with `action: "add"` creates a new holiday. `action: "remove"` suppresses a Copp Clark holiday for that tenant. A remove for a date that isn't in Copp Clark is a no-op. |
| **Weekend rules** | Copp Clark doesn't list weekends. OSYTE stores per-centre weekend patterns (Sat–Sun for most; Fri–Sat for UAE/Saudi; Sun-only for Israel, etc.). The resolved set uses these when answering `isBusinessDay`. |

### 2.2 Terms Reader

LCS does not store fund liquidity terms — they already live in OSYTE's platform (e.g. `o2.instrument_fund`). The Terms Reader is a thin client that fetches terms on demand.

```mermaid
graph LR
    OSYTE[("OSYTE Platform<br/>o2.instrument_fund<br/>+ liquidity terms")] --> TR["Terms Reader"]
    TR --> Q["getTerms(instrument_id)"]
    TR --> QF["getTermsByFund(fund_id)"]
```

**Lookup:** By `instrument_id` (single class) or `fund_id` (all classes for a fund). The reader expects v15.5 schema records.

**Versioning:** Each record carries `metadata.fund_terms_version`. When OSYTE notifies LCS that terms have changed, the Calendar API triggers recomputation for affected instruments. The Terms Reader always fetches the current version — it has no local cache or copy.

### 2.3 Compute Engine

The pure-function heart of LCS. Given an instrument's terms (from the Terms Reader), a resolved holiday set (from the Holiday Resolver), an anchor, and a roll convention, it produces deterministic lifecycle dates.

The engine supports **multi-directional anchoring** — the caller can pin any one lifecycle date and the engine derives all others:

| Anchor mode | Direction | Example |
|---|---|---|
| `as_of` | Forward | "Starting from today, find next dealing dates" → derive deadlines forward |
| `target_settlement_date` | Backward | "I need cash by Oct 31" → find latest dealing date whose settlement is ≤ Oct 31 → derive notice deadline backward |
| `target_dealing_date` | Both | "I know the dealing date is Oct 1" → derive notice backward, settlement forward |
| `target_notice_deadline` | Forward | "I can submit notice by Jul 3" → find earliest dealing date this catches → derive settlement forward |

Internally, every mode resolves to a **dealing date** first, then derives all other dates from it. The Dealing Date Generator either searches forward or backward depending on the anchor mode.

```mermaid
graph TD
    subgraph "Upstream (feeds into Compute Engine)"
        HS["Holiday Resolver<br/>(ResolvedHolidaySet)"]
        TR["Terms Reader<br/>(instrument terms)"]
    end

    subgraph Inputs
        A["Anchor Date + Mode"]
        RC["Roll Convention"]
    end

    subgraph "Compute Engine"
        AR["Anchor Resolver<br/>resolve any anchor mode<br/>to a dealing date"]
        DDG["Dealing Date Generator"]
        BDC["Business Day Calculator<br/>+ Roll Convention Engine"]
        OC["Offset Calculator"]

        A --> AR
        AR -->|"resolved dealing date(s)"| DDG
        DDG -->|"validated dealing dates"| OC
        BDC -->|"adjusted dates"| OC
    end

    HS --> BDC
    HS --> AR
    TR --> AR
    TR --> DDG
    TR --> OC
    RC --> BDC

    OC --> SUB["Subscription Dates<br/>• dealing date<br/>• document deadline<br/>• cash funding deadline<br/>• NAV pricing cutoff"]
    OC --> RED["Redemption Dates<br/>• dealing date<br/>• notice deadline<br/>• settlement date<br/>• NAV pricing cutoff"]
    OC --> META["Ancillary Dates<br/>• lockup expiry<br/>• next eligible redemption<br/>• valuation day"]
```

#### 2.3.0 Anchor Resolver

Translates any anchor mode into one or more dealing dates. Uses a **search-and-verify** approach rather than naive arithmetic, because day offsets combined with roll conventions and multi-centre holidays are non-invertible — you can't reliably subtract "30 business days" backward and get the right answer without knowing which holidays were skipped and which roll adjustments were made.

**Algorithm (search-and-verify):**

1. **Generate a window of candidate dealing dates** around the anchor using the Dealing Date Generator
2. **Forward-compute** the full lifecycle chain (notice deadline, settlement date, etc.) for each candidate
3. **Select** the candidate(s) that satisfy the anchor constraint

**Per anchor mode:**

- **`as_of`** → Generate dealing dates forward from the anchor. Return the first N whose dealing date is ≥ anchor. (No reverse computation needed.)

- **`target_settlement_date`** → Generate dealing dates backward from the anchor. For each candidate, forward-compute its settlement date. Select the latest dealing date whose settlement falls on or before the target. This is correct because the settlement offset involves business-day counting and roll conventions that change the result non-linearly.

- **`target_dealing_date`** → Check if the given date is a valid dealing date. If yes, use it directly. If not, snap to the nearest valid dealing date and include a warning showing both the requested and resolved dates.

- **`target_notice_deadline`** → Generate dealing dates forward from the anchor. For each candidate, forward-compute (backward from the dealing date) its notice deadline. Select the earliest dealing date whose notice deadline is on or after the anchor. This ensures the caller's available notice window is respected.

When no valid dealing date satisfies the constraint (e.g. the target settlement date is too soon), the engine returns the **earliest reachable** dealing date set with an explanation, so the caller knows what *is* possible.

#### 2.3.1 Dealing Date Generator

Produces the sequence of dealing dates from terms. A fund can have **multiple dealing days** within a single period — e.g. a monthly fund might deal on both the 1st and the 15th. In OSYTE's existing data these are stored as `%+%`-delimited values (e.g. `1%+%15`).

**Key principle:** Each dealing day specification within a period is independent. It generates its own complete lifecycle chain (notice deadline, settlement date, NAV pricing cutoff, etc.). The Compute Engine iterates over all dealing days for each period rather than making separate calls per dealing day type (unlike the old platform).

**Dealing basis classification:**

| `dealing_basis` | Schedulable? | Behaviour |
|---|---|---|
| `periodic` | Yes | Recurring on `dealing_interval` (e.g. `{3, month}` = quarterly). The generator produces dates. |
| `anniversary` | Yes | Recurring on anniversary of subscription + `dealing_interval`. Requires `lockup_start_date` as the base. |
| `at_closing` | No (subscription only) | Deals only at fund closing. Engine returns a warning: "Subscription dealing occurs at closing only — no periodic dates to generate." |
| `at_maturity` | No (redemption only) | Deals only at fund maturity. Engine returns a warning: "Redemption dealing occurs at maturity only — no periodic dates to generate." |
| `discretionary` | No | Manager/board determines timing. Engine cannot generate dates. Warning: "Dealing is discretionary — contact fund administrator." |
| `complex` | No | Irregular or multi-rule structure. Engine cannot generate dates. Warning: "Dealing structure is complex — refer to `redemption_schedule` in fund terms." |

For unschedulable bases, the Compute API returns `dates: []` with a warning explaining why no dates can be generated. The Calendar API skips materialization for these instruments.

**Algorithm (for schedulable bases):**

1. Read `dealing_basis` and `dealing_interval`
2. For each period in the range, iterate over **every dealing day** in the fund's dealing day list:
   - `first/business` → first business day of the period
   - `last/calendar` → last calendar day of the period
   - `nth/business` + `ordinal: 3` → 3rd business day of the period
   - `specific_date/calendar` + `"15th"` → 15th of the month
   - When multiple dealing days exist (e.g. `[{anchor: "first", day_type: "business"}, {anchor: "nth", ordinal: 15, day_type: "calendar"}]`), both are generated for each period
3. Each generated dealing date is independently passed to the Offset Calculator, which derives the full lifecycle set from it
4. Results are returned **chronologically sorted** across all dealing day types — not grouped by type

**Example:** Fund with monthly dealing on 1st and 15th, 90-day notice, 30-day settlement:

```
Period: August 2026
  Dealing day 1: Aug 1   → notice deadline: May 3   → settlement: Aug 31
  Dealing day 2: Aug 15  → notice deadline: May 17  → settlement: Sep 14

Period: September 2026
  Dealing day 1: Sep 1   → notice deadline: Jun 3   → settlement: Oct 1
  Dealing day 2: Sep 15  → notice deadline: Jun 17  → settlement: Oct 15
```

The API returns all 4 date sets sorted chronologically: Aug 1, Aug 15, Sep 1, Sep 15 — each with its own notice and settlement chain. The caller never needs to know how many dealing day types exist or make separate calls for each.

#### 2.3.2 Completeness Gate

Before computation begins, the engine inspects the `availability` and `value_type` of every field it needs. Not all terms are fully populated — the v15.5 schema distinguishes between a value being present vs absent, and between exact vs approximate.

**Availability handling:**

| `availability` | Engine behaviour |
|---|---|
| `populated` | Use the value. Proceed. |
| `not_applicable` | Skip this field — it doesn't apply to this instrument. No warning. |
| `unknown` | Field is null. Engine cannot compute this date. Return null + warning: "Term unknown — not stated in source documents." |
| `not_assessed` | Field is null. Engine cannot compute this date. Return null + warning: "Term not assessed — review offering memorandum." |

**Value-type handling (when availability = populated):**

| `value_type` | Engine behaviour |
|---|---|
| `exact` | Use the value as-is. No caveats. |
| `minimum` | Use the value but flag: "Notice period is stated as a minimum (≥N days); actual requirement may be longer." |
| `maximum` | Use the value but flag: "Settlement is stated as a maximum (≤N days); actual may be shorter." |
| `estimated` | Use the value but flag: "This value is estimated from a derived source; confirm with offering memorandum." |
| `discretionary` | Use the value but flag: "This timing is at manager/director discretion; computed date is indicative only." |

**Required drivers:** The engine treats the following as required for computation — if any has `availability != populated`, the corresponding date set is returned as null with a warning:
- `dealing_basis` + `dealing_interval` (for periodic/anniversary) → dealing dates
- `notice_period.days` → notice deadline
- `settlement.days` → settlement date

**Considerations passthrough:** The engine collects all `considerations` (INFO/WARN/ACTION) from the terms sections it touches and surfaces them in the response. ACTION-tagged considerations trigger a top-level warning.

#### 2.3.3 Business Day Calculator & Roll Convention Engine

Every computed date must land on a valid business day. When a raw date falls on a holiday or weekend, the roll convention determines how it's adjusted. This logic is central to the Compute Engine — it's used by the Anchor Resolver, the Dealing Date Generator, and the Offset Calculator.

**Inputs:**
- A candidate date
- The `ResolvedHolidaySet` from the Holiday Resolver (includes holidays + weekend rules per centre)
- The applicable `business_day_centers`
- The roll convention (from the API request, default: Modified Following)

> **Known schema gap:** The v15.5 schema has no `roll_convention` field on `dealing_day` or `day_offset`. The Compute Engine currently accepts roll convention as an API-level parameter (defaulting to Modified Following). Recommend adding a `roll_convention` field to the schema's `dealing_day` and `day_offset` definitions so it can be specified per-instrument and per-offset. See Open Question #1.

**Roll conventions supported:**

| Convention | Rule | Example (Sat 2026-08-29) |
|---|---|---|
| **Following** | Roll forward to next business day | → Mon 2026-08-31 |
| **Modified Following** | Roll forward, but if that crosses month-end, roll backward instead | → Fri 2026-08-28 (forward would be Sep 1, crosses month boundary) |
| **Preceding** | Roll backward to previous business day | → Fri 2026-08-28 |
| **Modified Preceding** | Roll backward, but if that crosses month-start, roll forward instead | → Mon (only triggers at month boundaries) |

**Business day centre precedence:** Some terms carry their own `business_day_centers` at the rule level (e.g. `notice_period.business_day_centers: ["London", "Dublin"]`). When present, these override the instrument-level centres for that specific calculation. Precedence:

1. **Rule-level centres** (e.g. `notice_period.business_day_centers`) — if present, use these
2. **Instrument-level centres** (e.g. `instrument.business_day_centers`) — fallback

This matters because a fund domiciled in Cayman with USD currency might have instrument-level centres `["New York", "Cayman Islands"]` but a notice period specifically governed by `["London", "Dublin"]`.

**Multi-centre rule:** When multiple centres apply, a date is a business day only if it's a business day in **all** listed centres. If it's a holiday in any one centre, the roll convention is applied.

#### 2.3.4 Offset Calculator

Applies `day_offset` primitives (the universal timing building block) to compute deadlines:

```
Input:  anchor_date, days, direction, day_type, business_day_centers, roll_convention
Output: adjusted_date, cutoff_time (if applicable)

Algorithm:
  1. Resolve business_day_centers (rule-level if present, else instrument-level)
  2. Start from anchor_date
  3. Move `days` in `direction` (before/after/same_day)
     - If day_type = "calendar": count all days
     - If day_type = "business": count only business days (per resolved centers)
  4. Apply roll convention if landing on a non-business day
  5. If cutoff_hour + cutoff_timezone present in the day_offset:
     - Resolve cutoff_timezone to an IANA timezone via the centre's timezone
       (e.g. "New York" → "America/New_York")
     - Attach the absolute cutoff instant: adjusted_date @ cutoff_hour:cutoff_minute
       in the resolved timezone
```

> **Cutoff timezone note:** The v15.5 schema uses named centres for `cutoff_timezone` (e.g. "New York", "Dublin"), not IANA timezone strings. The Offset Calculator resolves these via the same centre alias/metadata used by the Holiday Resolver. See Open Question #2.

---

## 3. Data Flow — Compute API (Stateless)

```mermaid
sequenceDiagram
    participant Client
    participant ComputeAPI as Compute API
    participant TS as Terms Reader
    participant HS as Holiday Resolver
    participant DB as Holiday DB
    participant DE as Compute Engine

    Client->>ComputeAPI: POST /compute<br/>{instrument_id, anchor_type,<br/> anchor_date, tenant_id, ...}

    ComputeAPI->>TS: getTerms(instrument_id)
    TS-->>ComputeAPI: instrument terms (v15.5)<br/>includes business_day_centers

    ComputeAPI->>HS: resolveHolidays(tenant_id,<br/> business_day_centers, date_range)
    HS->>DB: fetch base holidays + tenant overlays<br/>+ aliases + weekend rules<br/>for relevant centres only
    DB-->>HS: rows
    Note over HS: Merge: base + overlay adds<br/>− overlay removes + weekends<br/>= resolved holiday set
    HS-->>ComputeAPI: ResolvedHolidaySet

    ComputeAPI->>DE: computeDates(terms, anchor_type,<br/> anchor_date, resolved_holidays,<br/> roll_convention)

    Note over DE: Anchor Resolver:<br/>translate anchor_type + date<br/>into target dealing date(s)<br/>(forward search for as_of,<br/> backward search for target_settlement)

    Note over DE: From resolved dealing date(s):<br/>Apply roll convention to each raw date<br/>Apply notice offsets (backward)<br/>Apply settlement offsets (forward)<br/>Compute all lifecycle dates<br/>(using ResolvedHolidaySet for<br/> all business-day checks)

    DE-->>ComputeAPI: lifecycle dates
    ComputeAPI-->>Client: 200 OK {anchor, subscription,<br/> redemption, lockup, ...}
```

---

## 4. Data Flow — Calendar API (Persisted)

```mermaid
sequenceDiagram
    participant Trigger as Trigger<br/>(terms change / holiday update / schedule)
    participant CalAPI as Calendar API
    participant DE as Compute Engine
    participant Store as Calendar Store (DB)
    participant Sub as Subscribers

    Note over Trigger: Triggers:<br/>1. New/updated fund terms<br/>2. Copp Clark holiday update<br/>3. Scheduled forward-fill

    Trigger->>CalAPI: POST /calendars/materialize<br/>{instrument_id, horizon}

    CalAPI->>DE: computeDates(terms, each dealing_date in horizon)
    DE-->>CalAPI: lifecycle dates for each period

    CalAPI->>Store: upsert calendar rows<br/>(effective-versioned)

    CalAPI->>Store: generate changelog<br/>(diff vs. previous version)

    CalAPI-->>Trigger: 200 OK {version, dates_changed}

    Note over Sub: Downstream consumers

    Sub->>CalAPI: GET /calendars/{instrument_id}
    CalAPI->>Store: query
    Store-->>CalAPI: calendar rows
    CalAPI-->>Sub: 200 OK {calendar}

    Sub->>CalAPI: GET /calendars/{instrument_id}/changelog
    CalAPI->>Store: query diffs
    Store-->>CalAPI: changelog entries
    CalAPI-->>Sub: 200 OK {changelog}
```

### Calendar Materialization

When triggered, the Calendar API:
1. Reads the instrument's terms from the Terms Reader
2. Generates all dealing dates within the specified horizon (e.g. 24 months forward)
3. For each dealing date, calls the Compute Engine to compute the full lifecycle date set
4. Writes the results to the Calendar Store with an `effective_version` timestamp
5. Diffs against the previous version to produce a changelog
6. Notifies subscribers of any date movements

### Recomputation Triggers

| Trigger | Scope | Behaviour |
|---|---|---|
| Fund terms updated | Single instrument | Recompute that instrument's calendar; changelog shows which dates moved |
| Copp Clark holiday file update | All instruments using affected centres | Identify affected instruments via `business_day_centers`; batch recompute; changelog per instrument |
| Client overlay change | Instruments using that overlay | Same as holiday update but scoped to client's instruments |
| Scheduled forward-fill | All instruments | Extend horizon as time passes (e.g. weekly cron to maintain 24-month forward window) |

Materialization is **async** — the API returns `202 Accepted` with a `job_id`. The job runs in the background; callers poll `GET /jobs/{job_id}` for status. Jobs are idempotent on `(tenant_id, instrument_id, reason)`.

### Calendar Store Schema

The Calendar Store is the **one thing LCS owns** — it persists the materialized output. It is effective-versioned: every materialization creates a new version; old versions are never deleted, enabling "what did the calendar say on date X?" queries.

```sql
-- One row per materialization run per instrument
CREATE TABLE calendar_versions (
    version_id          BIGSERIAL PRIMARY KEY,
    tenant_id           VARCHAR(50) NOT NULL,
    instrument_id       VARCHAR(50) NOT NULL,
    effective_from      TIMESTAMPTZ NOT NULL,  -- when this version became current
    superseded_at       TIMESTAMPTZ,           -- when the next version replaced it (null = current)
    trigger_reason      VARCHAR(30) NOT NULL,   -- holiday_data_update | terms_update | overlay_change | scheduled_refresh
    dataset_version     JSONB NOT NULL,         -- {holiday_file_id, terms_version, overlay_hash}
    horizon_from        DATE NOT NULL,
    horizon_to          DATE NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL,
    UNIQUE (tenant_id, instrument_id, effective_from)
);

-- One row per lifecycle date per dealing date per version
CREATE TABLE calendar_dates (
    id                  BIGSERIAL PRIMARY KEY,
    version_id          BIGINT NOT NULL REFERENCES calendar_versions(version_id),
    scope               VARCHAR(12) NOT NULL,   -- 'subscription' | 'redemption'
    dealing_day_label   VARCHAR(50),            -- e.g. "1st business day", "15th"
    dealing_date        DATE NOT NULL,
    notice_deadline     DATE,                   -- redemption only
    settlement_date     DATE,                   -- redemption only
    document_deadline   DATE,                   -- subscription only
    cash_funding_deadline DATE,                 -- subscription only
    nav_pricing_cutoff  DATE,
    cutoff_time         TIME,
    cutoff_timezone     VARCHAR(50),
    roll_applied        VARCHAR(20),            -- null | following | modified_following | ...
    unadjusted_date     DATE,                   -- dealing date before roll (null if no adjustment)
    UNIQUE (version_id, scope, dealing_date, dealing_day_label)
);

-- Changelog: what moved between versions
CREATE TABLE calendar_changelog (
    id                  BIGSERIAL PRIMARY KEY,
    version_id          BIGINT NOT NULL REFERENCES calendar_versions(version_id),
    previous_version_id BIGINT REFERENCES calendar_versions(version_id),
    scope               VARCHAR(12) NOT NULL,
    dealing_date        DATE NOT NULL,
    field_name          VARCHAR(30) NOT NULL,   -- e.g. 'notice_deadline', 'settlement_date'
    previous_value      DATE,
    new_value           DATE,
    reason              TEXT                     -- e.g. "New York holiday added; deadline rolled"
);
```

---

## 5. Data Flow — Liquidation Planning API (OPTIONAL — under discussion)

> **Status:** This capability is under active debate. The architecture isolates it completely from the core date engine. It is additive — removing it has zero impact on the Compute and Calendar APIs.

```mermaid
sequenceDiagram
    participant Client
    participant PlanAPI as Planning API
    participant ComputeAPI as Compute API
    participant TS as Terms Reader

    Client->>PlanAPI: POST /plan/redeem<br/>{instrument_id, desired_amount,<br/> position_nav, as_of_date}

    PlanAPI->>TS: getTerms(instrument_id)
    TS-->>PlanAPI: terms (gates, holdbacks, lockup, redemption)

    PlanAPI->>ComputeAPI: POST /compute<br/>{instrument_id, anchor: as_of_date,<br/> count: N dealing dates}
    ComputeAPI-->>PlanAPI: next N lifecycle date sets

    Note over PlanAPI: Planning Engine applies:<br/>1. Lockup check (is position locked?)<br/>2. Gate constraints (max % per period)<br/>3. Audit holdback (% withheld)<br/>4. Tiered notice periods (amount-dependent)

    PlanAPI-->>Client: 200 OK {tranches[], summary}
```

### What the Planning Engine does (if built)

Given a desired redemption amount and current position, it simulates the redemption schedule:

1. **Lockup check** — Is the position still within a hard/soft lockup? If hard, no redemption is possible until expiry. If soft, flag the early-exit fee.
2. **Gate application** — Apply investor-level and fund-level gate thresholds. E.g. a 25% investor gate on a quarterly fund means at most 25% of the investor's holding can be redeemed per quarter.
3. **Tranche scheduling** — If the desired amount exceeds the gate limit, split into multiple tranches across successive dealing dates.
4. **Audit holdback** — For redemptions exceeding the holdback threshold (e.g. >=95% of account), withhold the holdback percentage (e.g. 5%) and schedule its release after audit completion.
5. **Settlement projection** — For each tranche, compute expected cash-in-hand date based on settlement terms.

**The Planning Engine never computes dates itself** — it calls the Compute API for all date math and only applies amount-level constraints on top.

---

## 6. Component Dependency Map

```mermaid
graph TD
    subgraph "OSYTE Platform (existing — LCS reads only)"
        HDB[("Holiday DB<br/>base_holidays + tenant_overlays<br/>+ center_aliases + weekend_rules")]
        TDB[("Fund Terms<br/>o2.instrument_fund<br/>+ liquidity terms")]
    end

    HDB --> HS["Holiday Resolver<br/>(cached)"]
    TDB --> TR["Terms Reader"]

    HS --> CE["Compute Engine"]
    TR --> CE

    CE --> CA["Compute API"]
    CE --> CLA["Calendar API"]
    CLA --> CS["Calendar Store"]
    CA -.-> PA["Planning API<br/>(optional)"]
    TR -.-> PA

    style PA stroke-dasharray: 5 5
```

Data flows top-down: OSYTE platform data → Holiday Resolver / Terms Reader → Compute Engine → APIs. Dashed lines = optional dependency. The Planning API depends on the Compute API but is never depended upon. LCS owns no data stores except the Calendar Store (materialized calendars); everything else is read from OSYTE. The Holiday Resolver caches popular base calendars (US, GB) and resolved sets to avoid repeated DB hits.

---

## 7. Key Architectural Decisions

| # | Decision | Rationale |
|---|---|---|
| 1 | **LCS owns no source data — it reads from OSYTE** | Holiday data, fund terms, overlays, and aliases all live in OSYTE's existing platform. LCS only owns the Calendar Store (materialized output). This avoids data duplication and keeps OSYTE as the single source of truth. |
| 2 | **Compute Engine is a pure function** — no side effects, no state | Given the same terms + holidays + anchor + roll convention, it always produces the same output. Testability, auditability, reproducibility. |
| 3 | **Planning layer calls Compute API, never the Compute Engine directly** | Clean separation. Planning is a consumer of dates, not a producer. Can be removed without touching core code. |
| 4 | **Holidays resolved per-tenant with two-tier cache** | The Holiday Resolver caches popular base calendars (US, GB) long-lived and resolved sets (base + overlay) short-lived. On cache miss, it fetches only the relevant centres from OSYTE's DB and merges. Cache is invalidated when OSYTE signals a data update. Different tenants get different resolved sets from the same base data. |
| 5 | **Centre-alias resolution is in the DB (`center_aliases` table)** | Fund terms say "Cayman Islands"; Copp Clark says "George Town". The mapping is a DB lookup in OSYTE, not hardcoded. New aliases are added with a row insert, not a code change. |
| 6 | **Calendar Store is effective-versioned, not mutable** | Supports "what did the calendar say on date X?" queries for audit and compliance. Old versions are never deleted, only superseded. |
| 7 | **Roll conventions are applied at the Business Day Calculator level, not the Offset Calculator** | Keeps the offset logic simple (count days) and the adjustment logic in one place. All four standard conventions supported: Following, Modified Following, Preceding, Modified Preceding. |
| 8 | **Multi-centre business day = intersection** | A date is a business day only if it's a business day in ALL listed centres. This is the standard market convention for multi-currency instruments. |
| 9 | **Weekend rules are per-centre, not global** | UAE (Fri–Sat), Israel (Fri–Sat or Sun-only depending on context), most others (Sat–Sun). Stored in OSYTE's `weekend_rules` table. |
| 10 | **Completeness gate before computation** | The engine inspects `availability` and `value_type` on every required driver before computing. Unpopulated required fields → null result + warning. Estimated/minimum/discretionary values → computed result + caveat. |
| 11 | **Dataset version stamp on every response** | The compute response includes `{holiday_file_id, terms_version, overlay_hash}`. Given the same inputs + dataset version, the engine produces the same output. Enables audit trail and replay. |
| 12 | **Search-and-verify for backward anchoring** | Reverse anchor modes (target_settlement_date, target_notice_deadline) enumerate candidate dealing dates and forward-compute each, rather than naively inverting offsets. Offsets + roll conventions are non-invertible. |

---

## 8. Security & Entitlements

LCS is a multi-tenant service. Tenant isolation is enforced at the API boundary.

| Concern | Approach |
|---|---|
| **Authentication** | Every API request carries a bearer token. The token is validated against OSYTE's auth service. |
| **Tenant binding** | The `tenant_id` in each request is verified against the token's claims. A token for tenant A cannot query tenant B's overlays or calendars. |
| **Holiday data isolation** | Base Copp Clark data is shared across all tenants (read-only). Tenant overlays are scoped by `tenant_id` — a tenant can only read/write their own. |
| **Calendar Store isolation** | Materialized calendars are keyed by `(tenant_id, instrument_id)`. A tenant can only read their own calendars. |
| **Fund terms access** | The Terms Reader inherits OSYTE's existing entitlement model — a tenant can only fetch terms for instruments they are entitled to. |
| **Planning API** (if built) | Position data (NAV, desired_amount) is never persisted by LCS. It is accepted in the request, used for computation, and discarded. |

---

## 9. Open Questions

| # | Question | Context | Impact |
|---|---|---|---|
| 1 | **Roll convention per instrument/offset** | The v15.5 schema has no `roll_convention` field. Currently accepted as an API parameter (default: Modified Following). Should the schema add it to `dealing_day` and `day_offset`? | Different instruments may require different conventions. API-level default works for now but doesn't scale. |
| 2 | **Cutoff timezone resolution** | The schema uses named centres ("New York", "Dublin") for `cutoff_timezone`, not IANA timezone strings. How does the Compute Engine resolve these? | Requires centre metadata to include timezone. Currently assumes the Holiday Resolver's alias/centre data carries this. Needs confirmation from OSYTE's data model. |
| 3 | **Planning API scope** | Should LCS compute only *when* (dates) or also *how much* (amounts through gates/holdbacks)? | Fundamental scope question. Architecture isolates it — no decision needed until Phase 2. |
| 4 | **Planning API position data** | If built, where does position NAV come from? Does the caller supply it, or does LCS read it from OSYTE? | Affects API contract and data coupling. Current design: caller supplies it. |
| 5 | **Calendar horizon and SLA** | How far forward should calendars be materialized? What's the acceptable staleness? | Affects storage size and recomputation frequency. Current default: 24 months, weekly refresh. |
| 6 | **Half-day closes** | Some exchanges close early on certain days (e.g. Christmas Eve). Copp Clark may flag these. How should the engine treat them? | Currently treated as full business days. May need a `partial_close` flag if cutoff times matter. |
| 7 | **Overlay stacking** | Can a tenant have multiple overlays that interact? E.g. a fund-level overlay and a firm-level overlay? | Current design: one overlay set per `(tenant_id, center_id)`. Multi-layer overlays would need a precedence model. |
| 8 | **NL/MCP assistive layer** | The README mentions natural-language querying and an MCP server. When/how does this integrate with the Compute and Calendar APIs? | Phase 2 feature. Architecture should leave room for a query layer on top of the Calendar API. |
| 9 | **Terms version reconciliation** | Fund terms carry `metadata.schema_version` (currently 15.5.0). If the schema evolves, how does the Compute Engine handle mixed versions? | Needs a version gate or adapter. Not urgent while all records are v15.5. |
