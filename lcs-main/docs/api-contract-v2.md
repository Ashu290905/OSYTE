# LCS API Contract (v2)

## What LCS does

Enterprise clients hold portfolios of instruments — stocks, ETFs, hedge funds, private equity — each with different dealing frequencies, notice periods, settlement timelines, lockups, gates, and holdbacks. Today, ops analysts manually count business days against holiday calendars to figure out when they can trade, when they need to submit notice, and when cash lands. LCS replaces that with deterministic computation.

**LCS solves three problems:**

1. **"When are the next key dates for this instrument?"** — Given an instrument's liquidity terms and the relevant holiday calendars, compute the next dealing date, notice deadline, and settlement date.

2. **"How do I redeem $X from this position?"** — Given the same terms plus the investor's position size and constraints (lockups, gates, holdbacks), produce a tranche-by-tranche schedule showing how much can be redeemed on which dates.

3. **"Give me a pre-built calendar for this instrument."** — For downstream systems that need all dates pre-computed, maintain a stored forward-looking calendar that can be rebuilt when holidays or terms change.

---

## Methods

Four methods across two APIs:

### Date Calculator API

Stateless computation. Nothing is stored. The caller sends all inputs, gets dates back.

| # | Method | Route | Solves | What it does |
|---|---|---|---|---|
| 1 | `POST` | `/date-calculator/lifecycle-dates` | Problem 1 | Returns the next actionable dealing dates with their notice deadlines and settlement dates |
| 2 | `POST` | `/date-calculator/redemption-plan` | Problem 2 | Returns a tranche-by-tranche redemption schedule accounting for lockups, gates, and holdbacks |

### Instrument Calendar API

Persistent storage. Maintains forward-looking calendars that downstream systems can query without re-computing.

| # | Method | Route | Solves | What it does |
|---|---|---|---|---|
| 3 | `GET` | `/instrument-calendars/{instrument_id}` | Problem 3 | Returns the stored forward-looking calendar for an instrument |
| 4 | `POST` | `/instrument-calendars/rebuild` | Problem 3 | Triggers a rebuild of stored calendars when holidays or terms change |

---

## Assumptions

### 1. The caller provides all required inputs

LCS does not read from OSYTE's database. The caller sends the instrument's liquidity terms and holiday calendars with every request.

**Why:** LCS is a computation service, not a data service. Keeping it stateless means it has no dependency on OSYTE's database being available, no cache staleness issues for terms, and no authorization complexity for who can read which instrument's data. The caller already has the inputs — they just need the dates computed.

### 2. The caller provides all relevant holiday calendars individually

The caller sends the holiday calendars for each business day centre the instrument uses. LCS receives them as separate calendars (e.g. one for New York, one for Cayman Islands) — the merging and intersection logic happens inside the engine.

**Why:** Different instruments reference different centres. The caller fetches the calendars it needs from OSYTE (Copp Clark base + any tenant overlays already applied per centre) and passes them to LCS. LCS doesn't need to know about tenants, overlays, or alias resolution — it just receives calendars keyed by centre name.

### 3. Dates are ISO 8601, amounts are in fund currency

All dates are `YYYY-MM-DD`. All timestamps are UTC. Monetary amounts (`redemption_amount`, `position_nav`) are in the fund's operational currency.

**Why:** Unambiguous. No timezone confusion. No currency conversion inside LCS.

> **Note:** Ping Yash to confirm currency handling for multi-currency instruments.

### 4. The only persistent state LCS owns is the Instrument Calendar Store

The Date Calculator is fully stateless. The Instrument Calendar API stores materialized calendars — that's the only write path in LCS.

**Why:** Downstream systems (Investment Planning, Rebalancing, reporting) need pre-built calendars they can query without sending full terms and holidays every time. But the computation itself remains stateless.

> **Note:** Confirm with Yash whether `tenant_id` is derived from the auth token or passed explicitly as a parameter.

---

## Method 1: `POST /date-calculator/lifecycle-dates`

**Purpose:** "When are the next key dates for this instrument?"

Given an instrument's liquidity terms and holiday calendars, returns the next actionable dealing dates with notice deadlines and settlement dates.

### What the caller sends

**Inputs (ordered by relevance):**

| Param | Type | Required | Source | What it means |
|---|---|---|---|---|
| `instrument_id` | string | yes | caller | The instrument to compute dates for |
| `side` | string | yes | caller | Which side of the instrument: `redemption` or `subscription` |
| `subscriptionTerms` or `redemptionTerms` | object | yes | from liquidity terms JSON | The full terms block for the requested side. Pass `subscriptionTerms` if `side=subscription`, `redemptionTerms` if `side=redemption`. See Fields below. |
| `anchor_date` | date | yes | caller | The date to search from (usually today) |
| `anchor_type` | string | no | caller | How to search. Default: `as_of`. Options: `as_of`, `target_settlement_date`, `target_dealing_date`, `target_notice_deadline` |
| `count` | int | no | caller | How many date sets to return. Default: 1. Max: 12 |
| `holidays` | object | yes | caller | Holiday calendars keyed by centre name. E.g. `{"New York": ["2026-01-01", ...], "Cayman Islands": ["2026-01-26", ...]}` |

**Fields inside `redemptionTerms`:**

| Field | Type | What it means |
|---|---|---|
| `dealingBasis` | string | How often dealing occurs: `periodic`, `anniversary`, `at_closing`, `at_maturity`, `discretionary`, `complex` |
| `dealingInterval` | object | Recurrence period. E.g. `{"count": 3, "unit": "month"}`. Required when `dealingBasis` is `periodic` or `anniversary`. |
| `dealingDay` | object | Which day within the period. E.g. `{"anchor": "first", "dayType": "business"}` |
| `noticePeriod` | object | Notice requirement. Contains `days`, `dayType`, `direction`, `availability`, `valueType`, optionally `businessDayCenters`, `cutoffHour` (0-23, UTC). Absent for listed assets. |
| `settlement` | object | Settlement timing. Contains `days`, `dayType`, `direction`, `availability`, `valueType`. |
| `redemptionSchedule` | object | Complex redemption scheduling (tiered tranches, anniversary-based). Only present for funds with non-standard structures. |


**Fields inside `subscriptionTerms`:**

| Field | Type | What it means |
|---|---|---|
| `dealingBasis` | string | Same as redemption |
| `dealingInterval` | object | Same as redemption |
| `dealingDay` | object | Same as redemption |
| `documentDeadline` | object | Deadline for subscription application forms. Contains `days`, `dayType`, `direction`. |
| `cashFundingDeadline` | object | Deadline for cleared subscription funds. |

> **Note:** Current sample data only covers hedge funds. Field names and structures may vary for other instrument types (listed assets, PE, real estate). Flag this as a known gap.

`anchor_type` explained:
- `as_of` — "Starting from this date, what's the next dealing date I can still act on?" (most common)
- `target_settlement_date` — "I need cash by this date — what's the latest dealing date that settles in time?"
- `target_dealing_date` — "I know the dealing date — give me the notice deadline and settlement date"
- `target_notice_deadline` — "I can submit notice by this date — which dealing date does that catch?"

### What Method 1 returns

```jsonc
{
  "results": [
    {
      "dealing_date": "YYYY-MM-DD",
      "notice_deadline": "YYYY-MM-DD | null",
      "settlement_date": "YYYY-MM-DD | null",
      "notice_window_open": true,
      "cutoff_hour": 17                       // 0-23 UTC, null if not specified
    }
  ]
}
```

### Example — Listed ETF, "what's next?"

A portfolio manager wants to sell a London-listed ETF. Daily dealing, no notice period, T+1 settlement.

**Request:**
```jsonc
POST /date-calculator/lifecycle-dates

{
  "instrument_id": "ETF.IWRD",
  "side": "redemption",
  "redemptionTerms": {
    "dealingBasis": "periodic",
    "dealingInterval": {"count": 1, "unit": "day"},
    "settlement": {"days": 1, "dayType": "business", "direction": "after", "availability": "populated", "valueType": "exact"}
  },
  "anchor_date": "2026-06-15",
  "holidays": {
    "London": ["2026-01-01", "2026-04-03", "2026-04-06", "2026-05-04", "2026-05-25", "2026-08-31", "2026-12-25", "2026-12-28"]
  }
}
```

The `redemptionTerms` block is passed as-is from the liquidity terms JSON. This ETF has no `noticePeriod` or `dealingDay` — they're simply absent. Only one centre (London), so `holidays` has one key.

```
Engine: Jun 15 is a Monday, business day in London. No notice period. Settlement = Jun 16.
```

**Response:**
```jsonc
{
  "results": [
    {
      "dealing_date": "2026-06-15",
      "notice_deadline": null,
      "settlement_date": "2026-06-16",
      "notice_window_open": true,
      "cutoff_hour": null
    }
  ]
}
```

### Example — Hedge fund, "I need cash by October 31st"

Fund B: quarterly dealing (1st business day), 30-day notice, 30-day settlement. Centres: New York + Cayman Islands.

**Request:**
```jsonc
POST /date-calculator/lifecycle-dates

{
  "instrument_id": "C.444",
  "side": "redemption",
  "redemptionTerms": {
    "dealingBasis": "periodic",
    "dealingInterval": {"count": 3, "unit": "month"},
    "dealingDay": {"anchor": "first", "dayType": "business"},
    "noticePeriod": {"days": 30, "dayType": "calendar", "direction": "before", "availability": "populated", "valueType": "exact"},
    "settlement": {"days": 30, "dayType": "calendar", "direction": "after", "availability": "populated", "valueType": "exact"}
  },
  "anchor_date": "2026-10-31",
  "anchor_type": "target_settlement_date",
  "holidays": {
    "New York": ["2026-01-01", "2026-01-19", "2026-02-16", "2026-05-25", "2026-07-03", "2026-09-07", "2026-11-26", "2026-12-25"],
    "Cayman Islands": ["2026-01-01", "2026-01-26", "2026-05-18", "2026-07-06", "2026-11-09", "2026-12-25"]
  }
}
```

Just the `redemptionTerms` block from the JSON — no `gates`, `restrictions`, `redemptionFees`, `governance`, or `context` needed for date computation. Two centres, so `holidays` has two keys.

```
Engine works backward: Q4 dealing Oct 1 → settlement Oct 1 + 30 days = Oct 31 (Sat) → rolls to Oct 30 (Fri).
Oct 30 ≤ target Oct 31 → yes. Notice: Oct 1 − 30 days = Sep 1.
```

**Response:**
```jsonc
{
  "results": [
    {
      "dealing_date": "2026-10-01",
      "notice_deadline": "2026-09-01",
      "settlement_date": "2026-10-30",
      "notice_window_open": true,
      "cutoff_hour": 17
    }
  ]
}
```

---

## Method 2: `POST /date-calculator/redemption-plan`

**Purpose:** "How do I redeem $X from this position?"

Same as Method 1, but the caller also provides the amount, position size, and redemption constraints (lockups, gates, holdbacks). LCS evaluates the constraints, splits into tranches if needed, and returns a full schedule.

### What the caller sends

**Inputs (ordered by workflow: constraints first, then dates):**

| Param | Type | Required | Source | What it means |
|---|---|---|---|---|
| `instrument_id` | string | yes | caller | The instrument to plan redemption for |
| `side` | string | yes | caller | `redemption` (or `subscription` for buy-side planning) |
| `redemption_amount` | number | yes | caller | How much the investor wants to redeem (in fund currency) |
| `position_nav` | number | yes | caller | Current position value (in fund currency) |
| `lockup_start_date` | date | conditional | caller | When the investor subscribed. Required if `restrictions.lockupProvisions` has a lockup. |
| `restrictions` | object | no | from liquidity terms JSON | The full `restrictions` block. Contains `lockupProvisions`, `lockupExceptions`, `auditHoldbacks`, `transferRestrictions`. Omit if no restrictions. |
| `gates` | object[] | no | from liquidity terms JSON | The full `gates` array. Contains `gateLevel`, `thresholdPct`, `measurementPeriod`, etc. Omit if no gates. |
| `redemptionTerms` | object | yes | from liquidity terms JSON | The full `redemptionTerms` block (same fields as Method 1). |
| `anchor_date` | date | yes | caller | As-of date (usually today) |
| `holidays` | object | yes | caller | Holiday calendars keyed by centre name. Same format as Method 1. |

**Not needed from the liquidity terms JSON:** `metadata`, `instrument`, `subscriptionTerms`, `redemptionFees`, `governance`, `context`. The engine only reads dealing terms + constraints.

### What Method 2 returns

```jsonc
{
  "applied_constraints": {
    "lockup": {
      "active": false,
      "lockup_type": "hard",
      "expiry_date": "2026-01-15",
      "anchor_shifted": false,
      "early_exit_fee_pct": null
    },
    "gate": {
      "active": true,
      "gate_level": "investor_level",
      "threshold_pct": 25,
      "max_per_period": 2000000,
      "measurement_period": "quarterly"
    },
    "holdback": {
      "active": false,
      "threshold_pct": 95,
      "holdback_pct": 5,
      "triggered": false
    }
  },
  "tranches": [
    {
      "tranche_number": 1,
      "amount": 0,
      "dealing_date": "YYYY-MM-DD",
      "notice_deadline": "YYYY-MM-DD | null",
      "settlement_date": "YYYY-MM-DD | null",
      "cutoff_hour": 17,
      "notice_window_open": true,
      "gate_limited": true,
      "holdback_amount": 0,
      "early_exit_fee": 0
    }
  ],
  "summary": {
    "redeemable": 5000000,
    "tranches": 3,
    "holdback": 0,
    "early_exit_fee": 0
  }
}
```

> **Note:** If position NAV changes between tranches (because earlier redemptions reduce it), how do we calculate gate and holdback amounts for later tranches? Ping Yash — does the gate threshold apply to original NAV or remaining NAV at each tranche?

> **Note:** Confirm whether holdback creates a separate payment (paid later after audit) or reduces the amount within the same tranche's settlement.

### Example — Listed ETF, redeem $1M

No constraints. One tranche, cash tomorrow.

**Request:**
```jsonc
POST /date-calculator/redemption-plan

{
  "instrument_id": "ETF.IWRD",
  "side": "redemption",
  "redemption_amount": 1000000,
  "position_nav": 5000000,
  "redemptionTerms": {
    "dealingBasis": "periodic",
    "dealingInterval": {"count": 1, "unit": "day"},
    "settlement": {"days": 1, "dayType": "business", "direction": "after", "availability": "populated", "valueType": "exact"}
  },
  "anchor_date": "2026-06-15",
  "holidays": {
    "London": ["2026-01-01", "2026-04-03", "2026-04-06", "2026-05-04", "2026-05-25", "2026-08-31", "2026-12-25", "2026-12-28"]
  }
}
```

No `gates`, `restrictions`, or `lockup_start_date` — the ETF has no constraints.

```
Constraints: no lockup, no gates, no holdback. Nothing to adjust.
Engine: Jun 15 → dealing today, settlement Jun 16.
```

**Response:**
```jsonc
{
  "applied_constraints": {
    "lockup": {"active": false, "lockup_type": null, "expiry_date": null, "anchor_shifted": false, "early_exit_fee_pct": null},
    "gate": {"active": false, "gate_level": null, "threshold_pct": null, "max_per_period": null, "measurement_period": null},
    "holdback": {"active": false, "threshold_pct": null, "holdback_pct": null, "triggered": false}
  },
  "tranches": [
    {
      "tranche_number": 1,
      "amount": 1000000,
      "dealing_date": "2026-06-15",
      "notice_deadline": null,
      "settlement_date": "2026-06-16",
      "cutoff_hour": null,
      "notice_window_open": true,
      "gate_limited": false,
      "holdback_amount": 0,
      "early_exit_fee": 0
    }
  ],
  "summary": {
    "redeemable": 1000000,
    "tranches": 1,
    "holdback": 0,
    "early_exit_fee": 0
  }
}
```

### Example — Hedge fund, redeem $5M from $8M position

Fund B. Subscribed Jan 15, 2025. 12-month hard lockup, 25% quarterly gate, 5% holdback on ≥95%.

**Request:**
```jsonc
POST /date-calculator/redemption-plan

{
  "instrument_id": "C.444",
  "side": "redemption",
  "redemption_amount": 5000000,
  "position_nav": 8000000,
  "lockup_start_date": "2025-01-15",
  "restrictions": {
    "lockupProvisions": {"hardLockup": {"lockupType": "hard", "duration": {"count": 12, "unit": "month"}, "startBasis": "subscription_day"}},
    "auditHoldbacks": {"holdbackApplies": {"value": true, "availability": "populated", "valueType": "exact"}, "holdbackTiers": [{"condition": "redemption_gte_pct_account", "thresholdPct": 95, "holdbackPct": 5, "holdbackReleaseTrigger": "audit_completion"}]}
  },
  "gates": [
    {"gateLevel": "investor_level", "gateBasis": "nav_percentage", "thresholdPct": 25, "thresholdBasis": "investor_holding", "measurementPeriod": "quarterly"}
  ],
  "redemptionTerms": {
    "dealingBasis": "periodic",
    "dealingInterval": {"count": 3, "unit": "month"},
    "dealingDay": {"anchor": "first", "dayType": "business"},
    "noticePeriod": {"days": 30, "dayType": "calendar", "direction": "before", "availability": "populated", "valueType": "exact"},
    "settlement": {"days": 30, "dayType": "calendar", "direction": "after", "availability": "populated", "valueType": "exact"}
  },
  "anchor_date": "2026-06-15",
  "holidays": {
    "New York": ["2026-01-01", "2026-01-19", "2026-02-16", "2026-05-25", "2026-07-03", "2026-09-07", "2026-11-26", "2026-12-25"],
    "Cayman Islands": ["2026-01-01", "2026-01-26", "2026-05-18", "2026-07-06", "2026-11-09", "2026-12-25"]
  }
}
```

Three blocks from the liquidity terms JSON — `redemptionTerms`, `gates`, `restrictions` — passed as-is. Inputs ordered: constraints first (lockup, gates), then terms, then dates.

```
Lockup: expired Jan 15, 2026. Today Jun 15 → unlocked.
Gate: 25% of $8M = $2M/quarter. $5M needs 3 tranches.
Holdback: $5M / $8M = 62.5% < 95% → not triggered.
T1: Q3 Jul 1 notice Jun 1 → PASSED. Skip to Q4 Oct 1 → notice Sep 1 → open.
T2: Q1 Jan 4 → notice Dec 4 → open.
T3: Q2 Apr 1 → notice Mar 2 → open.
```

**Response:**
```jsonc
{
  "applied_constraints": {
    "lockup": {"active": false, "lockup_type": "hard", "expiry_date": "2026-01-15", "anchor_shifted": false, "early_exit_fee_pct": null},
    "gate": {"active": true, "gate_level": "investor_level", "threshold_pct": 25, "max_per_period": 2000000, "measurement_period": "quarterly"},
    "holdback": {"active": false, "threshold_pct": 95, "holdback_pct": 5, "triggered": false}
  },
  "tranches": [
    {"tranche_number": 1, "amount": 2000000, "dealing_date": "2026-10-01", "notice_deadline": "2026-09-01", "settlement_date": "2026-10-30", "cutoff_hour": 17, "notice_window_open": true, "gate_limited": true, "holdback_amount": 0, "early_exit_fee": 0},
    {"tranche_number": 2, "amount": 2000000, "dealing_date": "2027-01-04", "notice_deadline": "2026-12-04", "settlement_date": "2027-02-03", "cutoff_hour": 17, "notice_window_open": true, "gate_limited": true, "holdback_amount": 0, "early_exit_fee": 0},
    {"tranche_number": 3, "amount": 1000000, "dealing_date": "2027-04-01", "notice_deadline": "2027-03-02", "settlement_date": "2027-05-04", "cutoff_hour": 17, "notice_window_open": true, "gate_limited": false, "holdback_amount": 0, "early_exit_fee": 0}
  ],
  "summary": {
    "redeemable": 5000000,
    "tranches": 3,
    "holdback": 0,
    "early_exit_fee": 0
  }
}
```

> **Note:** `notice_window_open` is `true` for all tranches because tranches 2 and 3 are future dealing dates whose notice windows haven't opened yet — they will be actionable when the time comes. The field means "can notice still be submitted as of today", not "is the notice window currently open for submission."

---

## Method 3: `GET /instrument-calendars/{instrument_id}`

**Purpose:** "Give me the full pre-built calendar for this instrument."

Unlike Methods 1 and 2 which compute on the fly, this reads from the stored Instrument Calendar. The calendar is pre-computed and updated when holidays or terms change (see Method 4).

### What the caller sends

**Inputs (ordered by relevance):**

| Param | Type | Required | What it means |
|---|---|---|---|
| `instrument_id` | string | yes | The instrument (in the URL path) |
| `side` | string | no | `subscription`, `redemption`, or both (default) |
| `tenant_id` | string | yes | Which tenant's calendar (different tenants may have different holiday overlays) |
| `date` | date | no | A specific date to look up. Returns only the row for that dealing date. Mutually exclusive with `from`/`to`. |
| `from` | date | no | Start of range. Default: today. Ignored if `date` is provided. |
| `to` | date | no | End of range. Default: from + 12 months. Ignored if `date` is provided. |

No liquidity terms or holidays needed — the data is already stored in the calendar.

### What Method 3 returns

```jsonc
{
  "instrument_id": "string",
  "tenant_id": "string",
  "range": {"from": "YYYY-MM-DD", "to": "YYYY-MM-DD"},
  "rows": [
    {
      "side": "redemption | subscription",
      "dealing_date": "YYYY-MM-DD",
      "notice_deadline": "YYYY-MM-DD | null",
      "settlement_date": "YYYY-MM-DD | null",
      "document_deadline": "YYYY-MM-DD | null",
      "cash_funding_deadline": "YYYY-MM-DD | null",
      "cutoff_hour": 17
    }
  ]
}
```

### Example — Hedge fund (Fund B), quarterly redemption calendar

**Request:** `GET /instrument-calendars/C.444?tenant_id=client-acme&side=redemption&from=2026-07-01&to=2027-07-01`

**Response:**
```jsonc
{
  "instrument_id": "C.444",
  "tenant_id": "client-acme",
  "range": {"from": "2026-07-01", "to": "2027-07-01"},
  "rows": [
    {"side": "redemption", "dealing_date": "2026-10-01", "notice_deadline": "2026-09-01", "settlement_date": "2026-10-30", "cutoff_hour": 17},
    {"side": "redemption", "dealing_date": "2027-01-04", "notice_deadline": "2026-12-04", "settlement_date": "2027-02-03", "cutoff_hour": 17},
    {"side": "redemption", "dealing_date": "2027-04-01", "notice_deadline": "2027-03-02", "settlement_date": "2027-05-04", "cutoff_hour": 17},
    {"side": "redemption", "dealing_date": "2027-07-01", "notice_deadline": "2027-06-01", "settlement_date": "2027-07-31", "cutoff_hour": 17}
  ]
}
```

### Example — Hedge fund (Fund B), both sides

**Request:** `GET /instrument-calendars/C.444?tenant_id=client-acme&from=2026-10-01&to=2027-01-31`

**Response:**
```jsonc
{
  "instrument_id": "C.444",
  "tenant_id": "client-acme",
  "range": {"from": "2026-10-01", "to": "2027-01-31"},
  "rows": [
    {"side": "subscription", "dealing_date": "2026-10-01", "document_deadline": "2026-09-25", "cash_funding_deadline": "2026-09-28", "cutoff_hour": 17},
    {"side": "redemption",   "dealing_date": "2026-10-01", "notice_deadline": "2026-09-01", "settlement_date": "2026-10-30", "cutoff_hour": 17},
    {"side": "subscription", "dealing_date": "2026-11-02", "document_deadline": "2026-10-27", "cash_funding_deadline": "2026-10-29", "cutoff_hour": 17},
    {"side": "subscription", "dealing_date": "2026-12-01", "document_deadline": "2026-11-25", "cash_funding_deadline": "2026-11-27", "cutoff_hour": 17},
    {"side": "subscription", "dealing_date": "2027-01-04", "document_deadline": "2026-12-29", "cash_funding_deadline": "2026-12-31", "cutoff_hour": 17},
    {"side": "redemption",   "dealing_date": "2027-01-04", "notice_deadline": "2026-12-04", "settlement_date": "2027-02-03", "cutoff_hour": 17}
  ]
}
```

Note: Fund B subscribes monthly but redeems quarterly — so there are more subscription rows than redemption rows in the same range.

### Example — Single date lookup

**Request:** `GET /instrument-calendars/C.444?tenant_id=client-acme&date=2026-10-01`

**Response:**
```jsonc
{
  "instrument_id": "C.444",
  "tenant_id": "client-acme",
  "range": {"from": "2026-10-01", "to": "2026-10-01"},
  "rows": [
    {"side": "subscription", "dealing_date": "2026-10-01", "document_deadline": "2026-09-25", "cash_funding_deadline": "2026-09-28", "cutoff_hour": 17},
    {"side": "redemption",   "dealing_date": "2026-10-01", "notice_deadline": "2026-09-01", "settlement_date": "2026-10-30", "cutoff_hour": 17}
  ]
}
```

### Why this exists separately from Method 1

Method 1 computes on every call — the caller sends terms and holidays each time. That's fine for one-off queries ("when's my next dealing date?") but bad for systems that need the full calendar for reporting, LP exports, or dashboards. Method 3 serves pre-built data — fast reads, no computation, no need to send terms and holidays.

---

## Method 4: `POST /instrument-calendars/rebuild`

**Purpose:** "Holidays or terms changed — rebuild the affected calendars."

This is the only write operation in LCS. It rebuilds stored calendars. It's async — returns immediately with a job ID, and the rebuild runs in the background.

### What the caller sends

**Inputs (ordered by relevance):**

| Param | Type | Required | Source | What it means |
|---|---|---|---|---|
| `tenant_id` | string | yes | caller | Which tenant's calendars to rebuild |
| `instrument_ids` | string[] | no | caller | Which instruments to rebuild. Omit or `[]` for all. In practice, only rebuild instruments whose centres were affected by the change. |
| `reason` | string | yes | caller | Why the rebuild was triggered: `holiday_update`, `terms_update`, `overlay_change`, `scheduled_rebuild` |
| `instruments` | map | yes | from liquidity terms JSON | Keyed by instrument_id. Each entry contains `subscriptionTerms` and `redemptionTerms` blocks from the liquidity terms JSON. |
| `holidays` | object | yes | caller | Holiday calendars keyed by centre name, covering all centres for the instruments being rebuilt |
| `horizon_months` | int | no | caller | How far forward to generate. Default: 24 |

`reason` values:
- `holiday_update` — Copp Clark published new holidays
- `terms_update` — fund liquidity terms changed
- `overlay_change` — tenant's holiday overlay was modified
- `scheduled_rebuild` — weekly cron extending the horizon

In practice, the caller should check which centres were affected by the holiday update and only rebuild instruments that use those centres. For example, a new Hong Kong holiday only needs to rebuild funds whose business day centres include Hong Kong — not every fund.

### What Method 4 returns

```jsonc
{
  "job_id": "string",
  "status": "accepted",
  "instruments_queued": 0
}
```

The rebuild runs asynchronously. How job tracking is handled (polling, webhooks, etc.) is an implementation detail.

### Example — Rebuild after a Hong Kong holiday update

Copp Clark added a new holiday on 2026-10-30 for Hong Kong. Only instruments using Hong Kong as a business day centre need rebuilding.

**Request:**
```jsonc
POST /instrument-calendars/rebuild

{
  "tenant_id": "client-acme",
  "instrument_ids": ["C.503", "C.612"],
  "reason": "holiday_update",
  "instruments": {
    "C.503": {
      "subscriptionTerms": {
        "dealingBasis": "periodic",
        "dealingInterval": {"count": 1, "unit": "month"},
        "dealingDay": {"anchor": "first", "dayType": "business"}
      },
      "redemptionTerms": {
        "dealingBasis": "periodic",
        "dealingInterval": {"count": 1, "unit": "month"},
        "dealingDay": {"anchor": "first", "dayType": "business"},
        "noticePeriod": {"days": 30, "dayType": "calendar", "direction": "before", "availability": "populated", "valueType": "exact"},
        "settlement": {"days": 20, "dayType": "calendar", "direction": "after", "availability": "populated", "valueType": "exact"}
      }
    },
    "C.612": {
      "subscriptionTerms": { "dealingBasis": "periodic", "dealingInterval": {"count": 1, "unit": "month"}, "dealingDay": {"anchor": "first", "dayType": "business"} },
      "redemptionTerms": { "dealingBasis": "periodic", "dealingInterval": {"count": 3, "unit": "month"}, "dealingDay": {"anchor": "first", "dayType": "business"}, "noticePeriod": {"days": 45, "dayType": "calendar", "direction": "before"}, "settlement": {"days": 30, "dayType": "calendar", "direction": "after"} }
    }
  },
  "holidays": {
    "Hong Kong": ["2026-01-01", "2026-01-26", "2026-05-18", "2026-07-06", "2026-10-01", "2026-10-30", "2026-11-09", "2026-12-25"]
  },
  "horizon_months": 24
}
```

Only 2 instruments queued — not the entire portfolio. The caller checked which centres were affected (Hong Kong) and only included instruments that use it.

**Response:**
```jsonc
{
  "job_id": "rebuild-20260715-001",
  "status": "accepted",
  "instruments_queued": 2
}
```

---

## Errors

Every error follows the same shape:

```jsonc
{
  "error": "error_code",
  "message": "Human-readable description",
  "details": {}
}
```

| Code | HTTP | When | Which methods |
|---|---|---|---|
| `missing_required_terms` | 422 | Required fields missing from terms (e.g. no `dealingBasis`) | 1, 2 |
| `unschedulable_dealing` | 422 | `dealingBasis` is `discretionary` or `complex` — engine can't generate dates | 1, 2 |
| `no_reachable_dealing_date` | 422 | No dealing date satisfies the anchor constraint (e.g. target settlement too soon) | 1, 2 |
| `lockup_start_date_required` | 400 | Fund has lockup but `lockup_start_date` not provided | 2 |
| `invalid_date_range` | 400 | `from` > `to` or range exceeds 5 years | 3 |
| `calendar_not_found` | 404 | No stored calendar for this instrument + tenant | 3 |

### Warnings

Not errors — the computation succeeds but the caller should know the result has caveats. Returned in the `warnings` array on Methods 1 and 2.

| Term `valueType` | Warning | Why it matters |
|---|---|---|
| `minimum` | "Notice period is a minimum (≥N days); actual may be longer" | The fund admin might require more notice than stated |
| `maximum` | "Settlement is a maximum (≤N days); actual may be shorter" | Cash might arrive earlier |
| `estimated` | "Value is estimated; confirm with offering memorandum" | The number came from a derived source, not the PPM |
| `discretionary` | "Timing is at manager discretion; date is indicative only" | The manager can override this date |
