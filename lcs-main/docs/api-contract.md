# LCS API Contract

## What LCS does

Enterprise clients hold portfolios of instruments — stocks, ETFs, hedge funds, private equity — each with different dealing frequencies, notice periods, settlement timelines, lockups, gates, and holdbacks. Today, ops analysts manually count business days against holiday calendars to figure out when they can trade, when they need to submit notice, and when cash lands. LCS replaces that with deterministic computation.

**LCS solves three problems:**

1. **"When are the next key dates for this instrument?"** — Given an instrument's liquidity terms and the relevant holiday calendar, compute the next dealing date, notice deadline, and settlement date.

2. **"How do I redeem $X from this position?"** — Given the same terms plus the investor's position size and constraints (lockups, gates, holdbacks), produce a tranche-by-tranche schedule showing how much can be redeemed on which dates.

3. **"Give me a pre-built calendar for this instrument."** — For downstream systems that need all dates pre-computed, maintain a stored forward-looking calendar that can be rebuilt when holidays or terms change.

---

## Methods

Five methods across two APIs:

### Date Calculator API

Stateless computation. Nothing is stored. The caller sends all the data, gets dates back.

| # | Method | Route | Solves | What it does |
|---|---|---|---|---|
| 1 | `GET` | `/date-calculator/lifecycle-dates` | Problem 1 | Returns the next actionable dealing dates with their notice deadlines and settlement dates |
| 2 | `GET` | `/date-calculator/redemption-plan` | Problem 2 | Returns a tranche-by-tranche redemption schedule accounting for lockups, gates, and holdbacks |

### Instrument Calendar API

Persistent storage. Maintains forward-looking calendars that downstream systems can query without re-computing.

| # | Method | Route | Solves | What it does |
|---|---|---|---|---|
| 3 | `GET` | `/instrument-calendars/{instrument_id}` | Problem 3 | Returns the stored forward-looking calendar for an instrument |
| 4 | `POST` | `/instrument-calendars/rebuild` | Problem 3 | Triggers a rebuild of stored calendars when holidays or terms change (async) |
| 5 | `GET` | `/instrument-calendars/jobs/{job_id}` | Problem 3 | Checks the status of a rebuild job |

---

## Assumptions

### 1. The caller provides all data

LCS does not read from OSYTE's database. The caller sends the fund's liquidity terms and the merged holiday list with every request.

**Why:** LCS is a computation service, not a data service. Keeping it stateless means it has no dependency on OSYTE's database being available, no cache staleness issues for terms, and no authorization complexity for who can read which fund's data. The caller already has the data — they just need the dates computed.

### 2. The holiday list is pre-merged

The caller sends one flat list of non-business days. Base holidays (Copp Clark) and tenant overlays (firm-level, fund-level) are already merged before the request reaches LCS.

**Why:** Holiday resolution involves tenant-specific overlays, alias mapping (e.g. "Cayman Islands" → "George Town"), and caching strategy — that's OSYTE platform logic, not date computation logic. LCS doesn't need to know about tenants or overlays. It just needs to know which days are holidays.

### 3. Roll convention defaults to Modified Following

If the caller doesn't specify a roll convention, Modified Following is used. This means: if a computed date lands on a holiday or weekend, roll forward to the next business day — unless that crosses a month boundary, in which case roll backward instead.

**Why:** Modified Following is the most common convention in fund dealing terms. The v15.5 schema doesn't have a per-instrument roll convention field yet, so it's an API parameter with a sensible default.

### 4. Dates are ISO 8601, amounts are in fund currency

All dates are `YYYY-MM-DD`. All timestamps are UTC. Monetary amounts (`redemption_amount`, `position_nav`) are in the fund's operational currency.

**Why:** Unambiguous. No timezone confusion. No currency conversion inside LCS.

### 5. The only data LCS persists is the Instrument Calendar Store

The Date Calculator is fully stateless. The Instrument Calendar API stores materialized calendars — that's the only write path in LCS.

**Why:** Downstream systems (Investment Planning, Rebalancing, reporting) need pre-built calendars they can query without sending full terms and holidays every time. But the computation itself remains stateless.

---

## Method 1: `GET /date-calculator/lifecycle-dates`

**Purpose:** "When are the next key dates for this instrument?"

Given an instrument's liquidity terms and holiday list, returns the next actionable dealing dates with notice deadlines and settlement dates.

### What the caller sends

**Inputs:**

| Param | Type | Required | Source | What it means |
|---|---|---|---|---|
| `instrument_id` | string | yes | caller | The instrument to compute dates for |
| `subscriptionTerms` or `redemptionTerms` | object | yes | from liquidity terms JSON | The full terms block for the requested side. Pass `subscriptionTerms` if `side=subscription`, `redemptionTerms` if `side=redemption`. Contains `dealingBasis`, `dealingInterval`, `dealingDay`, `noticePeriod`, `settlement`, etc. |
| `holidays` | date[] | yes | caller (merged) | Merged list of non-business days for the instrument's business day centres |
| `anchor_date` | date | yes | caller | The date to search from (usually today) |
| `anchor_type` | string | no | caller | How to search. Default: `as_of`. Options: `as_of`, `target_settlement_date`, `target_dealing_date`, `target_notice_deadline` |
| `side` | string | no | caller | Default: `redemption`. Options: `redemption`, `subscription` |
| `roll_convention` | string | no | caller | Default: `modified_following`. Options: `modified_following`, `following`, `preceding`, `modified_preceding` |
| `count` | int | no | caller | How many date sets to return. Default: 1. Max: 12 |

`anchor_type` explained:
- `as_of` — "Starting from this date, what's the next dealing date I can still act on?" (most common)
- `target_settlement_date` — "I need cash by this date — what's the latest dealing date that settles in time?"
- `target_dealing_date` — "I know the dealing date — give me the notice deadline and settlement date"
- `target_notice_deadline` — "I can submit notice by this date — which dealing date does that catch?"

### What LCS returns

```jsonc
{
  "results": [
    {
      "dealing_date": "2026-10-01",
      "notice_deadline": "2026-09-01",
      "settlement_date": "2026-10-30",
      "notice_window_open": true,
      "roll_applied": null,
      "unadjusted_dealing_date": null,
      "cutoff_time": null,
      "cutoff_timezone": null
    }
  ],
  "fingerprint": {
    "terms_hash": "a3f8c1",
    "holiday_hash": "b7d2e4"
  },
  "warnings": []
}
```

### Example — Listed ETF, "what's next?"

A portfolio manager wants to sell a London-listed ETF. Daily dealing, no notice period, T+1 settlement.

**Request:**
```jsonc
GET /date-calculator/lifecycle-dates?instrument_id=ETF.IWRD&anchor_date=2026-06-12&anchor_type=as_of&side=redemption

{
  "redemptionTerms": {
    "dealingBasis": "periodic",
    "dealingInterval": {"count": 1, "unit": "day"},
    "settlement": {"days": 1, "dayType": "business", "direction": "after", "availability": "populated", "valueType": "exact"}
  },
  "holidays": ["2026-01-01", "2026-04-03", "2026-04-06", "2026-05-04", "2026-05-25", "2026-08-31", "2026-12-25", "2026-12-28"]
}
```

The `redemptionTerms` block is passed as-is from the liquidity terms JSON. This ETF has no `noticePeriod` or `dealingDay` — they're simply absent.

```
Engine: Jun 12 is a business day in London. No notice period. Settlement = Jun 13.
```

**Response:**
```jsonc
{
  "results": [
    {
      "dealing_date": "2026-06-12",
      "notice_deadline": null,
      "settlement_date": "2026-06-13",
      "notice_window_open": true,
      "roll_applied": null,
      "unadjusted_dealing_date": null,
      "cutoff_time": null,
      "cutoff_timezone": null
    }
  ],
  "fingerprint": {"terms_hash": "c1a2b3", "holiday_hash": "d4e5f6"},
  "warnings": []
}
```

### Example — Hedge fund, "I need cash by October 31st"

Fund B: quarterly dealing (1st business day), 30-day notice, 30-day settlement. Centres: New York + Cayman Islands.

**Request:**
```jsonc
GET /date-calculator/lifecycle-dates?instrument_id=C.444&anchor_date=2026-10-31&anchor_type=target_settlement_date&side=redemption

{
  "redemptionTerms": {
    "dealingBasis": "periodic",
    "dealingInterval": {"count": 3, "unit": "month"},
    "dealingDay": {"anchor": "first", "dayType": "business"},
    "noticePeriod": {"days": 30, "dayType": "calendar", "direction": "before", "availability": "populated", "valueType": "exact"},
    "settlement": {"days": 30, "dayType": "calendar", "direction": "after", "availability": "populated", "valueType": "exact"}
  },
  "holidays": ["2026-01-01", "2026-01-19", "2026-01-26", "2026-02-16", "2026-05-18", "2026-05-25", "2026-07-03", "2026-07-06", "2026-09-07", "2026-11-09", "2026-11-26", "2026-12-25"]
}
```

Just the `redemptionTerms` block from the JSON — no `gates`, `restrictions`, `redemptionFees`, `governance`, or `context` needed for date computation.

```
Engine works backward: Q4 dealing Oct 1 → settlement Oct 31 (Sat) → rolls to Oct 30 ≤ target Oct 31 → yes.
Notice: Oct 1 − 30 days = Sep 1.
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
      "roll_applied": "modified_following",
      "unadjusted_dealing_date": null,
      "cutoff_time": null,
      "cutoff_timezone": null
    }
  ],
  "fingerprint": {"terms_hash": "a3f8c1", "holiday_hash": "b7d2e4"},
  "warnings": []
}
```

---

## Method 2: `GET /date-calculator/redemption-plan`

**Purpose:** "How do I redeem $X from this position?"

Same as Method 1, but the caller also provides the amount, position size, and redemption constraints (lockups, gates, holdbacks). LCS evaluates the constraints, splits into tranches if needed, and returns a full schedule.

### What the caller sends

**Inputs:**

| Param | Type | Required | Source | What it means |
|---|---|---|---|---|
| `instrument_id` | string | yes | caller | The instrument to plan redemption for |
| `redemptionTerms` | object | yes | from liquidity terms JSON | The full `redemptionTerms` block. Contains `dealingBasis`, `dealingInterval`, `dealingDay`, `noticePeriod`, `settlement`, `redemptionSchedule`, `considerations`. |
| `gates` | object[] | no | from liquidity terms JSON | The full `gates` array. Contains `gateLevel`, `thresholdPct`, `measurementPeriod`, etc. Omit if no gates. |
| `restrictions` | object | no | from liquidity terms JSON | The full `restrictions` block. Contains `lockupProvisions`, `lockupExceptions`, `auditHoldbacks`, `transferRestrictions`. Omit if no restrictions. |
| `holidays` | date[] | yes | caller (merged) | Merged list of non-business days for the instrument's business day centres |
| `anchor_date` | date | yes | caller | As-of date (usually today) |
| `redemption_amount` | number | yes | caller | How much the investor wants to redeem (in fund currency) |
| `position_nav` | number | yes | caller | Current position value (in fund currency) |
| `lockup_start_date` | date | conditional | caller | When the investor subscribed. Required if `restrictions.lockupProvisions` has a lockup. |
| `side` | string | no | caller | Default: `redemption` |
| `roll_convention` | string | no | caller | Default: `modified_following` |

**Not needed from the liquidity terms JSON:** `metadata`, `instrument`, `redemptionFees`, `governance`, `context`. The engine only reads dealing terms + constraints.

### What LCS returns

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
      "amount": 2000000,
      "dealing_date": "2026-10-01",
      "notice_deadline": "2026-09-01",
      "settlement_date": "2026-10-30",
      "notice_window_open": true,
      "gate_limited": true,
      "holdback_amount": 0,
      "early_exit_fee": 0
    }
    // ... more tranches
  ],
  "summary": {
    "total_redeemable": 5000000,
    "total_tranches": 3,
    "first_cash_date": "2026-10-30",
    "last_cash_date": "2027-05-04",
    "total_holdback": 0,
    "total_early_exit_fee": 0,
    "shortfall": 0
  },
  "fingerprint": {
    "terms_hash": "a3f8c1",
    "holiday_hash": "b7d2e4"
  },
  "warnings": []
}
```

### Example — Listed ETF, redeem $1M

No constraints. One tranche, cash tomorrow.

**Request:**
```jsonc
GET /date-calculator/redemption-plan?instrument_id=ETF.IWRD&anchor_date=2026-06-12&redemption_amount=1000000&position_nav=5000000

{
  "redemptionTerms": {
    "dealingBasis": "periodic",
    "dealingInterval": {"count": 1, "unit": "day"},
    "settlement": {"days": 1, "dayType": "business", "direction": "after", "availability": "populated", "valueType": "exact"}
  },
  "holidays": ["2026-01-01", "2026-04-03", "2026-04-06", "2026-05-04", "2026-05-25", "2026-08-31", "2026-12-25", "2026-12-28"]
}
```

No `gates`, `restrictions`, or `lockup_start_date` — the ETF has no constraints. Only `redemptionTerms` and `holidays`.

```
Constraints: no lockup, no gates, no holdback. Nothing to adjust.
Engine: Jun 12 → dealing today, settlement Jun 13.
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
      "dealing_date": "2026-06-12",
      "notice_deadline": null,
      "settlement_date": "2026-06-13",
      "notice_window_open": true,
      "gate_limited": false,
      "holdback_amount": 0,
      "early_exit_fee": 0
    }
  ],
  "summary": {
    "total_redeemable": 1000000,
    "total_tranches": 1,
    "first_cash_date": "2026-06-13",
    "last_cash_date": "2026-06-13",
    "total_holdback": 0,
    "total_early_exit_fee": 0,
    "shortfall": 0
  },
  "fingerprint": {"terms_hash": "c1a2b3", "holiday_hash": "d4e5f6"},
  "warnings": []
}
```

### Example — Hedge fund, redeem $5M from $8M position

Fund B. Subscribed Jan 15, 2025. 12-month hard lockup, 25% quarterly gate, 5% holdback on ≥95%.

**Request:**
```jsonc
GET /date-calculator/redemption-plan?instrument_id=C.444&anchor_date=2026-06-12&redemption_amount=5000000&position_nav=8000000&lockup_start_date=2025-01-15

{
  "redemptionTerms": {
    "dealingBasis": "periodic",
    "dealingInterval": {"count": 3, "unit": "month"},
    "dealingDay": {"anchor": "first", "dayType": "business"},
    "noticePeriod": {"days": 30, "dayType": "calendar", "direction": "before", "availability": "populated", "valueType": "exact"},
    "settlement": {"days": 30, "dayType": "calendar", "direction": "after", "availability": "populated", "valueType": "exact"}
  },
  "gates": [
    {"gateLevel": "investor_level", "gateBasis": "nav_percentage", "thresholdPct": 25, "thresholdBasis": "investor_holding", "measurementPeriod": "quarterly"}
  ],
  "restrictions": {
    "lockupProvisions": {"hardLockup": {"lockupType": "hard", "duration": {"count": 12, "unit": "month"}, "startBasis": "subscription_day"}},
    "auditHoldbacks": {"holdbackApplies": {"value": true, "availability": "populated", "valueType": "exact"}, "holdbackTiers": [{"condition": "redemption_gte_pct_account", "thresholdPct": 95, "holdbackPct": 5, "holdbackReleaseTrigger": "audit_completion"}]}
  },
  "holidays": ["2026-01-01", "2026-01-19", "2026-01-26", "2026-02-16", "2026-05-18", "2026-05-25", "2026-07-03", "2026-07-06", "2026-09-07", "2026-11-09", "2026-11-26", "2026-12-25"]
}
```

Three blocks from the liquidity terms JSON — `redemptionTerms`, `gates`, `restrictions` — passed as-is. No `redemptionFees`, `governance`, `context`, or `metadata`.

```
Lockup: expired Jan 15, 2026. Today Jun 12 → unlocked.
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
    {"tranche_number": 1, "amount": 2000000, "dealing_date": "2026-10-01", "notice_deadline": "2026-09-01", "settlement_date": "2026-10-30", "notice_window_open": true, "gate_limited": true, "holdback_amount": 0, "early_exit_fee": 0},
    {"tranche_number": 2, "amount": 2000000, "dealing_date": "2027-01-04", "notice_deadline": "2026-12-04", "settlement_date": "2027-02-03", "notice_window_open": false, "gate_limited": true, "holdback_amount": 0, "early_exit_fee": 0},
    {"tranche_number": 3, "amount": 1000000, "dealing_date": "2027-04-01", "notice_deadline": "2027-03-02", "settlement_date": "2027-05-04", "notice_window_open": false, "gate_limited": false, "holdback_amount": 0, "early_exit_fee": 0}
  ],
  "summary": {
    "total_redeemable": 5000000,
    "total_tranches": 3,
    "first_cash_date": "2026-10-30",
    "last_cash_date": "2027-05-04",
    "total_holdback": 0,
    "total_early_exit_fee": 0,
    "shortfall": 0
  },
  "fingerprint": {"terms_hash": "a3f8c1", "holiday_hash": "b7d2e4"},
  "warnings": []
}
```

---

## Method 3: `GET /instrument-calendars/{instrument_id}`

**Purpose:** "Give me the full pre-built calendar for this instrument."

Unlike Methods 1 and 2 which compute on the fly, this reads from the stored Instrument Calendar. The calendar is pre-computed and updated when holidays or terms change (see Method 5).

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `instrument_id` | string | yes | The instrument (in the URL path) |
| `tenant_id` | string | yes | Which tenant's calendar (different tenants may have different holiday overlays) |
| `from` | date | no | Start of range. Default: today |
| `to` | date | no | End of range. Default: from + 12 months |
| `side` | string | no | `subscription`, `redemption`, or both (default) |

No liquidity terms or holidays needed — the data is already stored in the calendar.

### What LCS returns

```jsonc
{
  "instrument_id": "C.444",
  "range": {"from": "2026-07-01", "to": "2027-07-01"},
  "rows": [
    {
      "side": "redemption",
      "dealing_day_label": "1st biz day",
      "dealing_date": "2026-10-01",
      "notice_deadline": "2026-09-01",
      "settlement_date": "2026-10-30",
      "cutoff_time": "17:00",
      "cutoff_timezone": "New York"
    },
    {
      "side": "subscription",
      "dealing_day_label": "1st biz day",
      "dealing_date": "2026-10-01",
      "document_deadline": "2026-09-25",
      "cash_funding_deadline": "2026-09-28"
    }
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

**Inputs:**

| Param | Type | Required | Source | What it means |
|---|---|---|---|---|
| `tenant_id` | string | yes | caller | Which tenant's calendars to rebuild |
| `instrument_ids` | string[] | no | caller | Which instruments to rebuild. Omit or `[]` for all. In practice, only rebuild instruments whose centres were affected. |
| `instruments` | map | yes | from liquidity terms JSON | Keyed by instrument_id. Each entry contains `subscriptionTerms` and `redemptionTerms` blocks from the liquidity terms JSON. |
| `holidays` | date[] | yes | caller (merged) | Merged holiday list covering all centres for the instruments being rebuilt |
| `horizon_months` | int | no | caller | How far forward to generate. Default: 24 |
| `reason` | string | yes | caller | Why dates might move: `holiday_update`, `terms_update`, `overlay_change`, `scheduled_rebuild` |

`reason` values:
- `holiday_update` — Copp Clark published new holidays
- `terms_update` — fund liquidity terms changed
- `overlay_change` — tenant's holiday overlay was modified
- `scheduled_rebuild` — weekly cron extending the horizon

In practice, the caller should check which centres were affected by the holiday update and only rebuild instruments that use those centres. For example, a new Hong Kong holiday only needs to rebuild funds whose business day centres include Hong Kong — not every fund.

### What LCS returns

```jsonc
{
  "job_id": "rebuild-20260715-001",
  "status": "accepted",
  "instruments_queued": 2
}
```

The rebuild runs asynchronously. Check status with Method 5.

---

## Method 5: `GET /instrument-calendars/jobs/{job_id}`

**Purpose:** "Is the calendar rebuild done yet?"

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `job_id` | string | yes | The job ID returned by Method 5 (in the URL path) |

### What LCS returns

```jsonc
{
  "job_id": "rebuild-20260715-001",
  "status": "completed",
  "instruments_queued": 2,
  "instruments_completed": 2,
  "instruments_failed": 0,
  "results": [
    {"instrument_id": "C.444", "status": "ok"},
    {"instrument_id": "C.503", "status": "ok"}
  ]
}
```

`status` values: `queued` → `running` → `completed` | `partial` (some instruments failed) | `failed`

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
| `rebuild_job_not_found` | 404 | Job ID not found | 5 |

### Warnings

Not errors — the computation succeeds but the caller should know the result has caveats. Returned in the `warnings` array.

| Term `value_type` | Warning | Why it matters |
|---|---|---|
| `minimum` | "Notice period is a minimum (≥N days); actual may be longer" | The fund admin might require more notice than stated |
| `maximum` | "Settlement is a maximum (≤N days); actual may be shorter" | Cash might arrive earlier |
| `estimated` | "Value is estimated; confirm with offering memorandum" | The number came from a derived source, not the PPM |
| `discretionary` | "Timing is at manager discretion; date is indicative only" | The manager can override this date |
