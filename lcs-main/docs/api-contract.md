# LCS API Contract

> **Version:** 1.0-draft  
> **Schema dependency:** Fund Liquidity Terms v15.5  
> **Base path:** `/api/v1`

---

## Table of Contents

1. [Shared Types](#1-shared-types)
2. [Compute API](#2-compute-api) (core)
3. [Calendar API](#3-calendar-api) (core)
4. [Liquidation Planning API](#4-liquidation-planning-api)
5. [Holiday API](#5-holiday-api) (supporting)
6. [Error Handling](#6-error-handling)

---

## 1. Shared Types

These types are referenced across all API surfaces.

### LifecycleDate

A single computed date with its derivation metadata.

```jsonc
{
  "date": "2026-09-01",              // ISO 8601 date
  "unadjusted_date": "2026-08-30",   // date before roll convention applied (null if no adjustment needed)
  "roll_convention_applied": "following", // null | "following" | "modified_following" | "preceding" | "modified_preceding"
  "day_type": "business",            // "calendar" | "business" — what was counted
  "holiday_centers_used": ["New York", "Cayman Islands"],
  "cutoff_time": "17:00",            // null if no cutoff specified
  "cutoff_timezone": "New York"       // null if no cutoff specified
}
```

### SubscriptionDateSet

All dates for a single subscription dealing date. A fund with multiple dealing days per period (e.g. 1st and 15th) produces one `SubscriptionDateSet` per dealing day — they are returned chronologically, not grouped by type.

```jsonc
{
  "dealing_date": { /* LifecycleDate */ },
  "dealing_day_label": "1st",           // which dealing day spec produced this date
                                         // e.g. "1st", "15th", "last business day"
                                         // helps distinguish when multiple dealing days exist
  "document_deadline": { /* LifecycleDate | null — if not specified in terms */ },
  "cash_funding_deadline": { /* LifecycleDate | null */ },
  "nav_pricing_cutoff": { /* LifecycleDate | null */ }
}
```

### RedemptionDateSet

All dates for a single redemption dealing date. Same as subscription — one per dealing day, sorted chronologically.

```jsonc
{
  "dealing_date": { /* LifecycleDate */ },
  "dealing_day_label": "1st",           // which dealing day spec produced this date
  "notice_deadline": { /* LifecycleDate */ },
  "settlement_date": { /* LifecycleDate | null — if settlement not specified */ },
  "nav_pricing_cutoff": { /* LifecycleDate | null */ },
  "tiered_notice_deadlines": [
    // only present if fund has tiered notice periods
    {
      "condition": "redemption_gt_pct_nav",
      "threshold_pct": 25,
      "notice_deadline": { /* LifecycleDate */ }
    }
  ]
}
```

### LockupStatus

```jsonc
{
  "has_lockup": true,
  "lockup_type": "hard",                    // "hard" | "soft" | null
  "lockup_start": "2025-01-15",             // null if no lockup
  "lockup_expiry": "2026-01-15",            // computed from start + duration
  "is_currently_locked": true,               // as of the anchor_date
  "early_exit_fee_applies": false,           // true only for soft lockup
  "early_exit_fee_pct": null,                // from lockup_exceptions if available
  "next_eligible_redemption_date": { /* LifecycleDate | null */ }
}
```

### InstrumentSummary

Minimal instrument context returned with every response.

```jsonc
{
  "instrument_id": "C.444",
  "fund_name": "HBK Multi-Strategy Offshore Fund Ltd.",
  "class_name": "A - Subclass A",
  "operational_currency": "USD",
  "dealing_basis": "periodic",
  "dealing_interval": { "count": 1, "unit": "month" },
  "dealing_days_count": 2,              // how many dealing days per period (1 = single, 2+ = multiple)
  "fund_terms_version": "C.444@2026-02-27",
  "schema_version": "15.5.0"
}
```

### Consideration

Human-judgment items surfaced from the fund terms.

```jsonc
{
  "tag": "INFO",         // "INFO" | "WARN" | "ACTION"
  "text": "Voting rights not stated in provided sources; defaulted to false.",
  "section": "instrument" // which section of the terms this came from
}
```

---

## 2. Compute API

Stateless, real-time date computation. No side effects. Every call with the same inputs produces the same outputs.

### Anchor modes

The user doesn't always start from "today." Often they know **one** date and need the rest derived. The Compute API supports anchoring from any lifecycle date:

| `anchor_type` | User is saying... | Engine works... |
|---|---|---|
| `as_of` (default) | "Starting from this date, give me the next dealing dates" | Forward from the given date to find upcoming dealing dates, then derives all other dates from each |
| `target_settlement_date` | "I need cash by this date — what do I need to do?" | Backward from the target to find the latest dealing date whose settlement falls on or before it, then derives notice deadline etc. |
| `target_dealing_date` | "I know the dealing date — give me the full date set" | Uses the given dealing date directly, derives notice deadline (backward) and settlement (forward) |
| `target_notice_deadline` | "I can submit notice by this date — which dealing date does that catch?" | Forward from the notice date to find the earliest dealing date this notice qualifies for, then derives settlement etc. |

When anchoring backward (e.g. from a settlement date), the engine finds the **nearest valid dealing date** that satisfies the constraint. The response always shows the full lifecycle date set regardless of which date the user started from.

---

### `POST /api/v1/compute`

Compute lifecycle dates for an instrument, anchored from any lifecycle date.

#### Request

```jsonc
{
  // --- Required ---
  "instrument_id": "C.444",
  "tenant_id": "client-acme",            // identifies the tenant — used to fetch
                                          // the correct holiday overlays from the DB

  // --- Anchor (exactly one of these groups is required) ---

  // Option A: "starting from this date, give me the next N dealing dates" (default)
  "anchor_type": "as_of",
  "anchor_date": "2026-07-01",

  // Option B: "I need cash by this date"
  // "anchor_type": "target_settlement_date",
  // "anchor_date": "2026-10-31",

  // Option C: "I know the dealing date"
  // "anchor_type": "target_dealing_date",
  // "anchor_date": "2026-10-01",

  // Option D: "I can submit notice by this date"
  // "anchor_type": "target_notice_deadline",
  // "anchor_date": "2026-07-03",

  // --- Optional ---
  "scope": ["subscription", "redemption", "lockup"],
  // Which date sets to compute. Defaults to all.
  // Options: "subscription", "redemption", "lockup"

  "count": 3,
  // Number of dealing dates to return. Default: 1. Max: 12.
  // For as_of: returns next N dealing dates forward.
  // For target_settlement_date: returns the N dealing dates whose settlement
  //   falls on or before the target (i.e. the latest opportunities to get cash by that date).
  // For target_dealing_date: returns N dealing dates starting from the given one.
  // For target_notice_deadline: returns the N dealing dates whose notice deadline
  //   falls on or after the anchor (i.e. the ones you can still catch).

  "roll_convention": "modified_following",
  // Override roll convention. Default: "modified_following".
  // Options: "following", "modified_following", "preceding", "modified_preceding"

  // tenant_id (above) determines which holiday overlays are applied.
  // No separate overlay_id needed — the DB resolves overlays per tenant.

  "lockup_start_date": "2025-01-15",
  // Required if scope includes "lockup" and the fund has lockup provisions.
  // The date the investor subscribed (lockup is measured from subscription).

  "include_considerations": true
  // Whether to surface INFO/WARN/ACTION items from terms. Default: true.
}
```

#### Example: "I need cash by October 31st"

```jsonc
{
  "instrument_id": "C.444",
  "tenant_id": "client-acme",
  "anchor_type": "target_settlement_date",
  "anchor_date": "2026-10-31",
  "scope": ["redemption"],
  "count": 1
}
```

The engine works backward:
1. Settlement is 30 calendar days after the dealing date → dealing date must be on or before Oct 1
2. Nearest valid dealing date on or before Oct 1 → Oct 1 (first business day of the month)
3. Notice period is 90 calendar days before dealing date → notice deadline is Jul 3
4. Returns the full set: notice by Jul 3 → deal on Oct 1 → cash by Oct 31

#### Response `200 OK`

```jsonc
{
  "instrument": { /* InstrumentSummary */ },

  "anchor": {
    "anchor_type": "target_settlement_date",  // echoes back what was requested
    "anchor_date": "2026-10-31",
    "resolved_direction": "backward"           // "forward" | "backward" — how the engine searched
  },

  "computed_at": "2026-07-01T14:23:01Z",

  // Dataset version — enables reproducibility. Given the same inputs + dataset_version,
  // the Compute Engine will always produce the same output.
  "dataset_version": {
    "holiday_file_id": 5560,             // which Copp Clark file version was used
    "terms_version": "C.444@2026-02-27", // fund_terms_version from the record
    "overlay_hash": "a3f8c1"             // hash of tenant overlay rows applied (null if none)
  },

  "subscription": {
    "dealing_basis": "periodic",
    "dealing_interval": { "count": 1, "unit": "month" },
    "dealing_days": [
      // funds can have multiple dealing days per period (e.g. 1st and 15th)
      // stored as %+% delimited in OSYTE
      { "anchor": "first", "day_type": "business" },
      { "anchor": "nth", "ordinal": 15, "day_type": "calendar" }
    ],
    "dates": [
      // sorted chronologically across all dealing day types
      { /* SubscriptionDateSet for 2026-08-01 (label: "1st business day") */ },
      { /* SubscriptionDateSet for 2026-08-15 (label: "15th") */ },
      { /* SubscriptionDateSet for 2026-09-01 (label: "1st business day") */ },
      { /* SubscriptionDateSet for 2026-09-15 (label: "15th") */ },
      { /* SubscriptionDateSet for 2026-10-01 (label: "1st business day") */ }
    ]
  },

  "redemption": {
    "dealing_basis": "periodic",
    "dealing_interval": { "count": 1, "unit": "month" },
    "dealing_days": [
      { "anchor": "first", "day_type": "business" }
    ],
    "notice_period_summary": "90 calendar days before redemption_day",
    "settlement_summary": "30 calendar days after redemption_day",
    "dates": [
      { /* RedemptionDateSet for 2026-10-01 */ }
      // When anchoring from settlement, the dates returned are the ones
      // whose settlement falls on or before the target date
      // Each dealing day type is evaluated independently
    ]
  },

  "lockup": { /* LockupStatus | null */ },

  "considerations": [
    { /* Consideration */ }
  ],

  "warnings": [
    // Engine-generated warnings (not from fund terms)
    // e.g. "notice_period value_type is 'minimum'; actual notice may be longer"
    // e.g. "settlement value_type is 'estimated'; actual settlement may differ"
    // e.g. "dealing_basis is 'discretionary'; dealing dates cannot be generated — contact fund administrator"
  ]
}
```

#### Response `422 Unprocessable Entity` (no valid dealing date)

Returned when no dealing date satisfies the constraint — e.g. the target settlement date is too soon for any reachable dealing date.

```jsonc
{
  "error": "no_valid_dealing_date",
  "message": "No dealing date found whose settlement falls on or before 2026-07-15. The earliest reachable settlement date is 2026-08-31 (dealing date 2026-08-01, notice deadline 2026-05-03).",
  "earliest_reachable": {
    "dealing_date": "2026-08-01",
    "notice_deadline": "2026-05-03",
    "settlement_date": "2026-08-31"
  }
}
```

#### Response `404 Not Found`

```jsonc
{
  "error": "instrument_not_found",
  "message": "No instrument found with id 'C.999'",
  "instrument_id": "C.999"
}
```

---

### `POST /api/v1/compute/batch`

Compute lifecycle dates for multiple instruments in a single request.

#### Request

```jsonc
{
  "requests": [
    {
      "instrument_id": "C.444",
      "anchor_type": "target_settlement_date",
      "anchor_date": "2026-10-31",
      "scope": ["redemption"],
      "count": 1
    },
    {
      "instrument_id": "C.503",
      "anchor_type": "as_of",
      "anchor_date": "2026-07-01",
      "scope": ["subscription"],
      "count": 2
    }
  ],
  // Max 50 instruments per batch.
  // Each request can use a different anchor_type.

  "tenant_id": "client-acme",
  // Applied to all requests — determines which holiday overlays are used.

  "roll_convention": "modified_following"
  // Applied to all requests unless overridden per-request.
}
```

#### Response `200 OK`

```jsonc
{
  "results": [
    {
      "instrument_id": "C.444",
      "status": "ok",
      "data": { /* same shape as single compute response */ }
    },
    {
      "instrument_id": "C.503",
      "status": "ok",
      "data": { /* ... */ }
    }
  ],
  "computed_at": "2026-07-01T14:23:01Z"
}
```

Individual instruments can fail without failing the entire batch:

```jsonc
{
  "instrument_id": "C.999",
  "status": "error",
  "error": {
    "error": "instrument_not_found",
    "message": "No instrument found with id 'C.999'"
  }
}
```

---

### `GET /api/v1/compute/next-actionable`

Convenience endpoint: "What's the next thing I need to do for this instrument?"

Returns the single nearest action date (notice deadline, document deadline, etc.) that is **still in the future** relative to `as_of`.

#### Request (query params)

| Param | Type | Required | Description |
|---|---|---|---|
| `instrument_id` | string | yes | |
| `as_of` | date | no | Defaults to today |
| `lockup_start_date` | date | no | For lockup-aware results |

#### Response `200 OK`

```jsonc
{
  "instrument": { /* InstrumentSummary */ },
  "as_of": "2026-07-01",
  "next_action": {
    "type": "redemption_notice_deadline",
    // "subscription_document_deadline" | "subscription_cash_deadline" |
    // "redemption_notice_deadline" | "redemption_dealing_date" | "lockup_expiry"
    "date": { /* LifecycleDate */ },
    "for_dealing_date": "2026-10-01",
    "days_remaining": 2,
    "description": "Submit redemption notice by 2026-07-03 for the 2026-10-01 dealing date"
  }
}
```

---

## 3. Calendar API

Persisted, forward-looking calendars. Materialized by the system; consumed by downstream services.

---

### `GET /api/v1/calendars/{instrument_id}`

Retrieve the materialized calendar for an instrument.

#### Path params

| Param | Type | Description |
|---|---|---|
| `instrument_id` | string | Instrument class ID |

#### Query params

| Param | Type | Required | Default | Description |
|---|---|---|---|---|
| `tenant_id` | string | yes | — | Tenant ID — calendars are materialized per tenant (different overlays produce different dates) |
| `from` | date | no | today | Start of date range |
| `to` | date | no | from + 12 months | End of date range |
| `scope` | string | no | `subscription,redemption` | Comma-separated: `subscription`, `redemption` |
| `version` | string | no | latest | Effective version ID for historical queries ("what did the calendar say on date X?") |

#### Response `200 OK`

```jsonc
{
  "instrument": { /* InstrumentSummary */ },
  "calendar_version": "v-2026-07-01T10:00:00Z",
  "effective_from": "2026-07-01T10:00:00Z",
  "supersedes_version": "v-2026-06-15T08:30:00Z",
  "range": { "from": "2026-07-01", "to": "2027-07-01" },

  "subscription_calendar": [
    // one entry per dealing date — funds with multiple dealing days per period
    // (e.g. 1st and 15th) produce multiple entries per period, sorted chronologically
    {
      "period_label": "August 2026",
      "dealing_day_label": "1st business day",
      "dealing_date": { /* LifecycleDate */ },
      "document_deadline": { /* LifecycleDate | null */ },
      "cash_funding_deadline": { /* LifecycleDate | null */ },
      "nav_pricing_cutoff": { /* LifecycleDate | null */ }
    },
    {
      "period_label": "August 2026",
      "dealing_day_label": "15th",
      "dealing_date": { /* LifecycleDate */ },
      "document_deadline": { /* LifecycleDate | null */ },
      "cash_funding_deadline": { /* LifecycleDate | null */ },
      "nav_pricing_cutoff": { /* LifecycleDate | null */ }
    }
    // ...
  ],

  "redemption_calendar": [
    {
      "period_label": "August 2026",
      "dealing_day_label": "1st business day",
      "dealing_date": { /* LifecycleDate */ },
      "notice_deadline": { /* LifecycleDate */ },
      "settlement_date": { /* LifecycleDate | null */ },
      "nav_pricing_cutoff": { /* LifecycleDate | null */ },
      "tiered_notice_deadlines": [ /* ... if applicable */ ]
    }
    // ...
  ],

  "considerations": [ /* Consideration[] */ ]
}
```

---

### `GET /api/v1/calendars/{instrument_id}/changelog`

Retrieve the changelog showing what moved between calendar versions.

#### Query params

| Param | Type | Required | Default | Description |
|---|---|---|---|---|
| `since_version` | string | no | previous version | Show changes since this version |
| `from` | date | no | none | Filter changes affecting dates after this |
| `to` | date | no | none | Filter changes affecting dates before this |

#### Response `200 OK`

```jsonc
{
  "instrument_id": "C.444",
  "current_version": "v-2026-07-01T10:00:00Z",
  "compared_to_version": "v-2026-06-15T08:30:00Z",
  "trigger": "holiday_data_update",
  // "holiday_data_update" | "terms_update" | "overlay_change" | "scheduled_refresh"

  "changes": [
    {
      "dealing_date": "2026-12-02",
      "scope": "redemption",
      "field": "notice_deadline",
      "previous_date": "2026-09-03",
      "new_date": "2026-09-02",
      "reason": "New York holiday on 2026-09-03 added by Copp Clark mid-year update; deadline rolled to preceding business day"
    }
  ],

  "summary": {
    "total_changes": 1,
    "subscription_dates_affected": 0,
    "redemption_dates_affected": 1
  }
}
```

---

### `GET /api/v1/calendars`

List all instruments with materialized calendars.

#### Query params

| Param | Type | Required | Description |
|---|---|---|---|
| `fund_id` | string | no | Filter by fund |
| `currency` | string | no | Filter by operational_currency |
| `stale` | boolean | no | If `true`, return only instruments whose calendar needs recomputation |

#### Response `200 OK`

```jsonc
{
  "instruments": [
    {
      "instrument_id": "C.444",
      "fund_name": "HBK Multi-Strategy Offshore Fund Ltd.",
      "class_name": "A - Subclass A",
      "calendar_version": "v-2026-07-01T10:00:00Z",
      "horizon_to": "2027-07-01",
      "is_stale": false,
      "next_dealing_date": "2026-08-01"
    }
  ]
}
```

---

### `POST /api/v1/calendars/materialize`

Trigger calendar materialization for one or more instruments. Typically called by an internal scheduler or after data updates.

#### Request

```jsonc
{
  "tenant_id": "client-acme",
  // Materialization is tenant-scoped — uses that tenant's holiday overlays.

  "instrument_ids": ["C.444", "C.503"],
  // Omit or pass [] to rematerialize all instruments for this tenant.

  "horizon_months": 24,
  // How far forward to generate. Default: 24.

  "reason": "holiday_data_update"
  // "holiday_data_update" | "terms_update" | "overlay_change" | "scheduled_refresh"
}
```

#### Response `202 Accepted`

```jsonc
{
  "job_id": "mat-20260701-001",
  "status": "accepted",
  "instruments_queued": 2,
  "estimated_completion": "2026-07-01T10:01:00Z"
}
```

Materialization is async — the response returns immediately. Poll the job status endpoint to check progress.

---

### `GET /api/v1/jobs/{job_id}`

Check the status of an async materialization job.

#### Response `200 OK`

```jsonc
{
  "job_id": "mat-20260701-001",
  "status": "completed",          // "accepted" | "in_progress" | "completed" | "failed"
  "tenant_id": "client-acme",
  "reason": "holiday_data_update",
  "instruments_queued": 2,
  "instruments_completed": 2,
  "instruments_failed": 0,
  "started_at": "2026-07-01T10:00:01Z",
  "completed_at": "2026-07-01T10:00:47Z",
  "results": [
    {
      "instrument_id": "C.444",
      "status": "ok",
      "calendar_version": "v-2026-07-01T10:00:47Z",
      "dates_changed": 3
    },
    {
      "instrument_id": "C.503",
      "status": "ok",
      "calendar_version": "v-2026-07-01T10:00:47Z",
      "dates_changed": 0
    }
  ]
}
```

---

### `POST /api/v1/calendars/export`

Export a calendar in a downloadable format for LP reporting or OMS integration.

#### Request

```jsonc
{
  "instrument_ids": ["C.444"],
  "from": "2026-07-01",
  "to": "2027-07-01",
  "format": "csv",       // "csv" | "xlsx" | "ics" (iCalendar)
  "scope": ["subscription", "redemption"]
}
```

#### Response `200 OK`

Content-Type based on format. Body is the file contents.

---

## 4. Liquidation Planning API

The Planning API is a separate layer that consumes the Compute API for date math and applies amount-level constraints (gates, holdbacks, lockups) on top.

---

### `POST /api/v1/plan/redeem`

Given a desired redemption amount, simulate the redemption schedule accounting for gates, holdbacks, lockups, and tiered notice periods.

#### Request

```jsonc
{
  // --- Required ---
  "instrument_id": "C.444",
  "tenant_id": "client-acme",
  "desired_amount": 5000000,          // USD amount to redeem
  "position_nav": 8000000,            // current position NAV in fund currency
  "as_of_date": "2026-07-01",

  // --- Optional ---
  "lockup_start_date": "2025-01-15",  // subscription date for lockup calculation
  "fund_nav": 200000000,              // required if fund-level gates exist
  "dealing_dates_forward": 8,         // how many dealing dates to simulate. Default: 8
  "include_holdback_projection": true  // whether to break out holdback amounts. Default: true
}
```

#### Response `200 OK`

```jsonc
{
  "instrument": { /* InstrumentSummary */ },
  "as_of_date": "2026-07-01",
  "desired_amount": 5000000,
  "position_nav": 8000000,

  // --- Constraint summary ---
  "constraints_applied": {
    "lockup": {
      "applies": false,
      "lockup_type": null,
      "lockup_expiry": null,
      "early_exit_fee_pct": null
    },
    "investor_gate": {
      "applies": true,
      "gate_level": "investor_level",
      "threshold_pct": 25,
      "measurement_period": "quarterly",
      "max_per_period": 2000000   // 25% of 8M position
    },
    "fund_gate": {
      "applies": false
    },
    "audit_holdback": {
      "applies": true,
      "condition": "redemption_gte_pct_nav",
      "threshold_pct": 95,
      "holdback_pct": 5,
      "triggered": false    // 5M / 8M = 62.5% < 95% threshold
    }
  },

  // --- Tranche schedule ---
  "tranches": [
    {
      "tranche_number": 1,
      "dealing_date": "2026-10-01",
      "notice_deadline": "2026-07-03",
      "notice_still_submittable": true,
      "redemption_amount": 2000000,
      "gate_limited": true,
      "settlement_date": "2026-10-31",
      "projected_cash_date": "2026-10-31",
      "holdback_amount": 0,
      "net_cash": 2000000
    },
    {
      "tranche_number": 2,
      "dealing_date": "2027-01-02",
      "notice_deadline": "2026-10-04",
      "notice_still_submittable": false,
      "redemption_amount": 2000000,
      "gate_limited": true,
      "settlement_date": "2027-02-01",
      "projected_cash_date": "2027-02-01",
      "holdback_amount": 0,
      "net_cash": 2000000
    },
    {
      "tranche_number": 3,
      "dealing_date": "2027-04-01",
      "notice_deadline": "2027-01-01",
      "notice_still_submittable": false,
      "redemption_amount": 1000000,
      "gate_limited": false,
      "settlement_date": "2027-05-01",
      "projected_cash_date": "2027-05-01",
      "holdback_amount": 0,
      "net_cash": 1000000
    }
  ],

  // --- Summary ---
  "summary": {
    "total_redeemable": 5000000,
    "total_tranches": 3,
    "first_cash_date": "2026-10-31",
    "last_cash_date": "2027-05-01",
    "total_holdback": 0,
    "holdback_release_trigger": null,
    "shortfall": 0,            // desired - total_redeemable
    "shortfall_reason": null,  // "gate_limited" | "lockup" | "fund_level_gate" | null
    "early_exit_fee_total": 0
  },

  "considerations": [
    {
      "tag": "INFO",
      "text": "Investor-level gate limits redemptions to 25% of holding per quarter. Full redemption requires 3 quarterly dealing dates.",
      "section": "gates"
    }
  ],

  "warnings": [
    "Gate threshold_pct value_type is 'exact'; actual application may vary at fund administrator's discretion.",
    "Projected amounts assume stable NAV. Actual amounts will be based on NAV at each dealing date."
  ]
}
```

---

### `POST /api/v1/plan/redeem/batch`

Batch planning across a portfolio — "how do I liquidate these 10 positions?"

#### Request

```jsonc
{
  "tenant_id": "client-acme",
  "positions": [
    {
      "instrument_id": "C.444",
      "desired_amount": 5000000,
      "position_nav": 8000000,
      "lockup_start_date": "2025-01-15"
    },
    {
      "instrument_id": "C.503",
      "desired_amount": 2000000,
      "position_nav": 3000000
    }
  ],
  "as_of_date": "2026-07-01",
  "dealing_dates_forward": 8
}
```

#### Response `200 OK`

```jsonc
{
  "plans": [
    {
      "instrument_id": "C.444",
      "status": "ok",
      "data": { /* same shape as single plan response */ }
    },
    {
      "instrument_id": "C.503",
      "status": "ok",
      "data": { /* ... */ }
    }
  ],

  "portfolio_summary": {
    "total_desired": 7000000,
    "total_redeemable": 7000000,
    "total_shortfall": 0,
    "first_cash_date": "2026-10-15",
    "last_cash_date": "2027-05-01",
    "total_holdback": 0,
    "total_early_exit_fees": 0,
    "cash_flow_timeline": [
      { "month": "2026-10", "projected_cash": 2500000 },
      { "month": "2027-01", "projected_cash": 2500000 },
      { "month": "2027-04", "projected_cash": 2000000 }
    ]
  }
}
```

---

## 5. Holiday API

Supporting API for querying holiday data. All holiday data (Copp Clark base, tenant overlays, centre aliases, weekend rules) lives in OSYTE's existing DB. LCS reads from it — there are no ingestion or write endpoints for base data. Overlay management endpoints are provided for tenant-specific overrides.

---

### `GET /api/v1/holidays/{center}`

List holidays for a financial centre or exchange. Shows the base Copp Clark holidays. Optionally shows a tenant's resolved view (base + overlays merged).

#### Path params

| Param | Type | Description |
|---|---|---|
| `center` | string | Centre name (e.g. "New York"), alias (e.g. "Cayman Islands"), or ISO MIC code (e.g. "XNYS") |

#### Query params

| Param | Type | Required | Default | Description |
|---|---|---|---|---|
| `from` | date | no | today | Start of date range |
| `to` | date | no | from + 12 months | End of date range |
| `tenant_id` | string | no | none | If provided, returns the merged view (base + tenant overlays). If omitted, returns base Copp Clark holidays only. |
| `include_weekends` | boolean | no | false | Include weekend days in the listing |

#### Response `200 OK`

```jsonc
{
  "center": "New York",
  "center_id": 17,
  "resolved_from_alias": null,          // e.g. "Cayman Islands" if an alias was used
  "source": "copp_clark_financial_centres",
  "weekend_days": ["saturday", "sunday"],
  "tenant_id": "client-acme",           // null if no tenant_id was provided
  "holidays": [
    {
      "date": "2026-07-03",
      "day_of_week": "Friday",
      "name": "Independence Day (Observed)",
      "source": "copp_clark",            // "copp_clark" = base data
      "is_overlay": false
    },
    {
      "date": "2026-11-27",
      "day_of_week": "Friday",
      "name": "Day After Thanksgiving (client closure)",
      "source": "tenant_overlay",        // "tenant_overlay" = added by this tenant
      "is_overlay": true
    }
    // Note: if a Copp Clark holiday was *removed* by the tenant overlay,
    // it simply won't appear in the list when tenant_id is provided
  ]
}
```

---

### `GET /api/v1/holidays/check`

Check whether a specific date is a business day in given centres. Tenant-aware — applies overlays if `tenant_id` is provided.

#### Query params

| Param | Type | Required | Description |
|---|---|---|---|
| `date` | date | yes | |
| `centers` | string (comma-separated) | yes | Centre names or aliases |
| `tenant_id` | string | no | If provided, includes tenant overlays in the check |

#### Response `200 OK`

```jsonc
{
  "date": "2026-07-03",
  "tenant_id": "client-acme",
  "is_business_day": false,             // false if ANY listed centre is on holiday
  "centers": {
    "New York": { "is_business_day": false, "reason": "Independence Day (Observed)", "source": "copp_clark" },
    "Cayman Islands": { "is_business_day": true, "reason": null, "resolved_center": "George Town" }
  },
  "next_joint_business_day": "2026-07-06"
}
```

---

### `GET /api/v1/holidays/overlays/{tenant_id}`

List all overlay entries for a tenant.

#### Query params

| Param | Type | Required | Default | Description |
|---|---|---|---|---|
| `center` | string | no | all centres | Filter by centre |
| `from` | date | no | today | Start of date range |
| `to` | date | no | from + 12 months | End of date range |

#### Response `200 OK`

```jsonc
{
  "tenant_id": "client-acme",
  "overlays": [
    {
      "center": "New York",
      "center_id": 17,
      "date": "2026-11-27",
      "action": "add",
      "event_name": "Day After Thanksgiving (client closure)",
      "created_by": "jane.doe@acme.com",
      "created_at": "2026-06-01T10:00:00Z"
    },
    {
      "center": "New York",
      "center_id": 17,
      "date": "2026-12-24",
      "action": "remove",
      "event_name": "Client considers Christmas Eve a working day",
      "created_by": "jane.doe@acme.com",
      "created_at": "2026-06-01T10:00:00Z"
    }
  ]
}
```

---

### `POST /api/v1/holidays/overlays/{tenant_id}`

Add or update overlay entries for a tenant. Entries are upserted — if an overlay already exists for the same `(tenant_id, center, date)`, it is replaced.

#### Request

```jsonc
{
  "center": "New York",
  "entries": [
    { "date": "2026-11-27", "action": "add", "name": "Day After Thanksgiving (client closure)" },
    { "date": "2026-12-24", "action": "remove", "name": "Client considers Christmas Eve a working day" }
  ]
}
```

#### Response `201 Created`

```jsonc
{
  "tenant_id": "client-acme",
  "center": "New York",
  "entries_upserted": 2,
  "affected_instruments": ["C.444", "C.503"],
  // instruments belonging to this tenant whose business_day_centers include New York
  "recomputation_triggered": true
  // Calendar API auto-rematerializes affected calendars
}
```

---

### `DELETE /api/v1/holidays/overlays/{tenant_id}`

Remove overlay entries for a tenant.

#### Request

```jsonc
{
  "center": "New York",
  "dates": ["2026-11-27", "2026-12-24"]
  // omit "dates" to remove ALL overlays for this tenant + centre
}
```

#### Response `200 OK`

```jsonc
{
  "tenant_id": "client-acme",
  "entries_removed": 2,
  "affected_instruments": ["C.444", "C.503"],
  "recomputation_triggered": true
}
```

---

## 6. Error Handling

All errors follow a consistent shape:

```jsonc
{
  "error": "error_code",
  "message": "Human-readable description",
  "details": { /* optional structured context */ }
}
```

### Error codes

| Code | HTTP Status | Description |
|---|---|---|
| `instrument_not_found` | 404 | No instrument with the given ID |
| `calendar_not_materialized` | 404 | Calendar hasn't been materialized for this instrument yet |
| `version_not_found` | 404 | Requested calendar version doesn't exist |
| `invalid_anchor_date` | 400 | Anchor date is invalid or outside supported range |
| `invalid_date_range` | 400 | `from` > `to` or range exceeds maximum (5 years) |
| `undetermined_dealing_dates` | 422 | Fund has `discretionary` or `complex` dealing basis; dates cannot be generated |
| `lockup_start_required` | 400 | Fund has lockup provisions but `lockup_start_date` was not provided |
| `fund_nav_required` | 400 | Fund-level gates exist but `fund_nav` was not provided (Planning API only) |
| `holiday_center_unknown` | 400 | A `business_day_center` in the fund terms has no matching holiday data |
| `batch_limit_exceeded` | 400 | Batch request exceeds maximum size |
| `overlay_conflict` | 409 | Overlay entry conflicts with an existing overlay |
| `internal_error` | 500 | Unexpected server error |

### Value-type warnings

The Compute and Calendar APIs surface **warnings** (not errors) when fund terms contain non-exact values:

| Fund term `value_type` | Warning text |
|---|---|
| `minimum` | "Notice period is stated as a minimum (≥N days); actual requirement may be longer" |
| `maximum` | "Settlement period is stated as a maximum (≤N days); actual may be shorter" |
| `estimated` | "This value is estimated from a derived source; confirm with offering memorandum" |
| `discretionary` | "This timing is at manager/director discretion; computed date is indicative only" |

### Availability handling

When a fund term has `availability != "populated"`:

| `availability` | API behaviour |
|---|---|
| `populated` | Normal computation |
| `not_applicable` | Field omitted from response (null) with no warning |
| `unknown` | Field set to null; warning: "Term is unknown — value not stated in source documents" |
| `not_assessed` | Field set to null; warning: "Term has not been assessed — review offering memorandum" |
