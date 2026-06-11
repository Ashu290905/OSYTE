# LCS API Contract

All data (liquidity terms, holiday calendars) must be provided by the caller. LCS is a stateless date calculation service — it does not read from OSYTE's database. The only data LCS persists is materialized calendars.

---

## Assumptions

1. The caller provides fund liquidity terms in full (v15.5 schema).
2. The caller provides the merged holiday list (base + overlays already resolved).
3. Roll convention defaults to Modified Following if not specified.
4. All dates are ISO 8601 (`YYYY-MM-DD`). All timestamps are UTC.
5. Monetary amounts are in the fund's operational currency.
6. Materialized calendars are the only persistent state LCS owns.

---

## Data Structures

### LiquidityTerms

The dealing and settlement terms for one side (subscription or redemption) of an instrument.

```jsonc
{
  "dealing_basis": "periodic",                          // "periodic" | "anniversary" | "at_closing" | "at_maturity" | "discretionary" | "complex"
  "dealing_interval": {                                 // required when dealing_basis is "periodic" or "anniversary"
    "count": 3,                                         // int, ≥ 1
    "unit": "month"                                     // "day" | "week" | "month"
  },
  "dealing_day": {                                      // which day within the period
    "anchor": "first",                                  // "first" | "last" | "nth" | "specific_date"
    "day_type": "business",                             // "business" | "calendar"
    "ordinal": null                                     // int, for anchor="nth" only (e.g. 3 = 3rd business day)
  },
  "notice_period": {                                    // null or not_applicable for listed assets
    "days": 30,                                         // int, ≥ 0
    "day_type": "calendar",                             // "calendar" | "business"
    "direction": "before",                              // "before" | "after" | "same_day"
    "availability": "populated",                        // "populated" | "not_applicable" | "unknown" | "not_assessed"
    "value_type": "exact",                              // "exact" | "minimum" | "maximum" | "estimated" | "discretionary"
    "cutoff_hour": 17,                                  // optional, 0-23
    "cutoff_timezone": "New York",                      // optional
    "business_day_centers": ["New York"]                // optional, overrides instrument-level centres for this offset
  },
  "settlement": {                                       // same shape as notice_period
    "days": 30,
    "day_type": "calendar",
    "direction": "after",
    "availability": "populated",
    "value_type": "exact"
  }
}
```

### RedemptionConstraints

Lockups, gates, and holdbacks that restrict how much can be redeemed and when.

```jsonc
{
  "gates": [                                            // array, can be empty
    {
      "gate_level": "investor_level",                   // "investor_level" | "class_level" | "fund_level" | "master_fund_level"
      "gate_basis": "nav_percentage",                   // "nav_percentage" | "fixed_amount" | "nav_drawdown"
      "threshold_pct": 25,                              // number, 0-100
      "threshold_basis": "investor_holding",            // "investor_holding" | "class_nav" | "fund_nav"
      "measurement_period": "quarterly"                 // "per_redemption_day" | "monthly" | "quarterly" | "annually"
    }
  ],
  "lockup_provisions": {                                // one of: no_lockup, hard_lockup, soft_lockup
    "no_lockup": true
    // "hard_lockup": {
    //   "lockup_type": "hard",
    //   "duration": {"count": 12, "unit": "month"},
    //   "start_basis": "subscription_day"              // "subscription_day" | "first_capital_call" | "fund_inception"
    // }
    // "soft_lockup": {
    //   "lockup_type": "soft",
    //   "duration": {"count": 12, "unit": "month"},
    //   "start_basis": "subscription_day",
    //   "early_exit_fee_applies": true
    // }
  },
  "audit_holdbacks": {
    "holdback_applies": false                           // boolean
    // when true:
    // "period_end_basis": "fiscal_year_end",
    // "holdback_tiers": [
    //   {
    //     "condition": "redemption_gte_pct_account",   // "redemption_gte_pct_nav" | "redemption_gte_pct_account"
    //     "threshold_pct": 95,
    //     "holdback_pct": 5,
    //     "holdback_release_trigger": "audit_completion"
    //   }
    // ]
  }
}
```

### HolidayList

A flat array of ISO dates — all non-business days for the relevant centres. The caller merges base holidays + tenant overlays before sending.

```jsonc
["2026-01-01", "2026-01-19", "2026-01-26", "2026-02-16", "2026-05-25", "2026-07-03", "2026-09-07", "2026-11-26", "2026-11-27", "2026-12-25"]
```

### KeyDates

The computed dates for a single dealing date — notice deadline, settlement, cutoffs.

```jsonc
{
  "dealing_date": "2026-10-01",                         // date
  "notice_deadline": "2026-09-01",                      // date | null (null for listed assets)
  "settlement_date": "2026-10-30",                      // date | null
  "document_deadline": null,                            // date | null (subscription only)
  "cash_funding_deadline": null,                        // date | null (subscription only)
  "nav_pricing_cutoff": null,                           // date | null
  "cutoff_time": "17:00",                               // string | null
  "cutoff_timezone": "New York",                        // string | null
  "roll_applied": "modified_following",                 // string | null (null if no adjustment)
  "unadjusted_dealing_date": null,                      // date | null (non-null if roll changed the dealing date)
  "notice_window_open": true                            // boolean — can notice still be submitted as of anchor_date?
}
```

### RedemptionTranche

One chunk of a liquidation plan — amount, dates, and whether it was constrained.

```jsonc
{
  "tranche_number": 1,                                  // int, 1-indexed
  "amount": 2000000,                                    // number, in fund currency
  "dealing_date": "2026-10-01",                         // date
  "notice_deadline": "2026-09-01",                      // date | null
  "settlement_date": "2026-10-30",                      // date
  "notice_window_open": true,                           // boolean
  "gate_limited": true,                                 // boolean — was this tranche capped by a gate?
  "holdback_amount": 0,                                 // number — amount withheld pending audit
  "early_exit_fee": 0                                   // number — fee for soft lockup early exit
}
```

### AppliedConstraints

What the planner checked and how it affected the inputs.

```jsonc
{
  "lockup": {
    "active": false,                                    // boolean — is the position currently locked?
    "lockup_type": "hard",                              // "hard" | "soft" | null
    "expiry_date": "2026-01-15",                        // date | null
    "anchor_shifted": false,                            // boolean — was anchor moved to lockup expiry?
    "early_exit_fee_pct": null                           // number | null
  },
  "gate": {
    "active": true,                                     // boolean — does a gate apply?
    "gate_level": "investor_level",                     // string | null
    "threshold_pct": 25,                                // number | null
    "max_per_period": 2000000,                          // number | null — threshold_pct × position_nav
    "measurement_period": "quarterly"                   // string | null
  },
  "holdback": {
    "active": false,                                    // boolean — does a holdback apply?
    "threshold_pct": 95,                                // number | null
    "holdback_pct": 5,                                  // number | null
    "triggered": false                                  // boolean — did cumulative redemption cross the threshold?
  }
}
```

### RedemptionSummary

Totals across all tranches in a liquidation plan.

```jsonc
{
  "total_redeemable": 5000000,                          // number
  "total_tranches": 3,                                  // int
  "first_cash_date": "2026-10-30",                      // date
  "last_cash_date": "2027-05-04",                       // date
  "total_holdback": 0,                                  // number
  "total_early_exit_fee": 0,                            // number
  "shortfall": 0                                        // number — desired_amount minus total_redeemable
}
```

### CalendarRow

One row in a materialized calendar — a single dealing date with all its deadlines.

```jsonc
{
  "side": "redemption",                                 // "subscription" | "redemption"
  "dealing_day_label": "1st biz day",                   // string — which dealing day spec produced this
  "dealing_date": "2026-10-01",                         // date
  "notice_deadline": "2026-09-01",                      // date | null
  "settlement_date": "2026-10-30",                      // date | null
  "document_deadline": null,                            // date | null (subscription only)
  "cash_funding_deadline": null,                        // date | null (subscription only)
  "nav_pricing_cutoff": null,                           // date | null
  "cutoff_time": "17:00",                               // string | null
  "cutoff_timezone": "New York"                         // string | null
}
```

### DateChange

A single date that moved between calendar versions.

```jsonc
{
  "side": "redemption",                                 // "subscription" | "redemption"
  "dealing_date": "2026-12-01",                         // date
  "field": "notice_deadline",                           // which field changed
  "previous_value": "2026-10-30",                       // date
  "new_value": "2026-10-29",                            // date
  "reason": "Hong Kong holiday added on 2026-10-30 by Copp Clark mid-year update"
}
```

### InputFingerprint

Hashes of the inputs used to produce a result. Enables reproducibility — same inputs + same fingerprint = same output.

```jsonc
{
  "terms_hash": "a3f8c1",                               // hash of the LiquidityTerms provided
  "holiday_hash": "b7d2e4"                              // hash of the HolidayList provided
}
```

For calendar responses, the fingerprint includes source info:

```jsonc
{
  "holiday_source": "copp_clark",                       // "copp_clark" | "client_supplied"
  "holiday_file_id": 5560,                              // int — which file version
  "terms_version": "C.444@2026-02-27",                  // fund_terms_version from the record
  "overlay_hash": "a3f8c1"                              // hash of tenant overlays applied
}
```

---

## 1. Date Calculator

Stateless date calculation. Two endpoints: one for raw dates, one for redemption planning.

### `GET /calculate/lifecycle-dates` — Next key dates for an instrument

Returns the next actionable dealing dates with their notice deadlines and settlement dates. No planning, no tranches.

**Query params:**

| Param | Type | Required | Description |
|---|---|---|---|
| `anchor_date` | date | yes | Starting point for the search |
| `anchor_type` | string | no | `as_of` (default) `target_settlement_date` `target_dealing_date` `target_notice_deadline` |
| `side` | string | no | `redemption` (default) `subscription` |
| `roll_convention` | string | no | `modified_following` (default) `following` `preceding` `modified_preceding` |
| `count` | int | no | Number of date sets to return. Default: 1. Max: 12 |

**Request body:**

```jsonc
{
  "terms": LiquidityTerms,
  "holidays": HolidayList
}
```

**Response `200 OK`:**

```jsonc
{
  "results": [KeyDates],                                // array, length = count
  "fingerprint": InputFingerprint,
  "warnings": ["string"]
}
```

---

### `GET /calculate/redemption-plan` — Liquidation schedule with constraints

Returns a tranche-by-tranche redemption schedule, accounting for lockups, gates, and holdbacks.

**Query params:**

| Param | Type | Required | Description |
|---|---|---|---|
| `anchor_date` | date | yes | As-of date for the plan |
| `desired_amount` | number | yes | Amount to redeem |
| `position_nav` | number | yes | Current position NAV |
| `side` | string | no | `redemption` (default) |
| `roll_convention` | string | no | `modified_following` (default) |
| `lockup_start_date` | date | conditional | Required if fund has lockup provisions |
| `fund_nav` | number | conditional | Required if fund-level gates exist |

**Request body:**

```jsonc
{
  "terms": LiquidityTerms,
  "constraints": RedemptionConstraints,
  "holidays": HolidayList
}
```

**Response `200 OK`:**

```jsonc
{
  "applied_constraints": AppliedConstraints,
  "tranches": [RedemptionTranche],
  "summary": RedemptionSummary,
  "fingerprint": InputFingerprint,
  "warnings": ["string"]
}
```

---

## 2. Calendar Store API

Manages persisted forward-looking calendars. The only part of LCS that writes data.

### `GET /calendars/{instrument_id}` — Stored calendar for an instrument

**Query params:**

| Param | Type | Required | Description |
|---|---|---|---|
| `tenant_id` | string | yes | Tenant identifier |
| `from` | date | no | Start of range. Default: today |
| `to` | date | no | End of range. Default: from + 12 months |
| `side` | string | no | `subscription` `redemption` or both (default) |
| `version` | string | no | Specific version ID for historical audit queries. Default: latest |

**Response `200 OK`:**

```jsonc
{
  "instrument_id": "string",
  "calendar_version": "string",
  "range": {"from": "date", "to": "date"},
  "rows": [CalendarRow],
  "fingerprint": InputFingerprint                       // calendar-specific with source info
}
```

---

### `GET /calendars/{instrument_id}/changes` — What moved and why

**Query params:**

| Param | Type | Required | Description |
|---|---|---|---|
| `tenant_id` | string | yes | Tenant identifier |
| `since_version` | string | no | Compare against this version. Default: previous version |
| `from` | date | no | Filter changes affecting dates after this |
| `to` | date | no | Filter changes affecting dates before this |

**Response `200 OK`:**

```jsonc
{
  "instrument_id": "string",
  "current_version": "string",
  "compared_to_version": "string",
  "trigger": "string",                                  // "holiday_update" | "terms_update" | "overlay_change" | "scheduled_refresh"
  "changes": [DateChange],
  "totals": {
    "total_changes": 0,
    "subscription_affected": 0,
    "redemption_affected": 0
  }
}
```

---

### `POST /calendars/refresh` — Rebuild calendars

Triggers recomputation of stored calendars. Creates new versions.

**Request body:**

```jsonc
{
  "tenant_id": "string",
  "instrument_ids": ["string"],                         // omit or [] for all
  "horizon_months": 24,                                 // default: 24
  "reason": "string",                                   // "holiday_update" | "terms_update" | "overlay_change" | "scheduled_refresh"
  "instruments": {                                      // keyed by instrument_id
    "C.444": {
      "subscription_terms": LiquidityTerms,
      "redemption_terms": LiquidityTerms,
      "constraints": RedemptionConstraints
    }
  },
  "holidays": HolidayList
}
```

**Response `202 Accepted`:**

```jsonc
{
  "job_id": "string",
  "status": "accepted",
  "instruments_queued": 0
}
```

---

### `GET /calendars/jobs/{job_id}` — Refresh job status

**Response `200 OK`:**

```jsonc
{
  "job_id": "string",
  "status": "string",                                   // "queued" | "running" | "completed" | "partial" | "failed"
  "instruments_queued": 0,
  "instruments_completed": 0,
  "instruments_failed": 0,
  "results": [
    {
      "instrument_id": "string",
      "status": "ok",                                   // "ok" | "error"
      "calendar_version": "string",
      "dates_changed": 0
    }
  ]
}
```

---

## 3. Errors

```jsonc
{
  "error": "error_code",
  "message": "Human-readable description",
  "details": {}
}
```

| Code | HTTP | When |
|---|---|---|
| `missing_required_terms` | 422 | Required fields missing from LiquidityTerms |
| `unschedulable_dealing` | 422 | `dealing_basis` is `discretionary` or `complex` — can't generate dates |
| `no_reachable_dealing_date` | 422 | No dealing date satisfies the anchor constraint |
| `lockup_start_date_required` | 400 | Fund has lockup but `lockup_start_date` not provided |
| `fund_nav_required` | 400 | Fund-level gates exist but `fund_nav` not provided |
| `invalid_date_range` | 400 | `from` > `to` or range exceeds 5 years |
| `calendar_not_found` | 404 | No stored calendar for this instrument + tenant |
| `calendar_version_not_found` | 404 | Requested version doesn't exist |
| `refresh_job_not_found` | 404 | Job ID not found |

### Warnings

Returned in `warnings` when terms have non-exact values:

| `value_type` | Warning |
|---|---|
| `minimum` | "Notice period is a minimum (≥N days); actual may be longer" |
| `maximum` | "Settlement is a maximum (≤N days); actual may be shorter" |
| `estimated` | "Value is estimated; confirm with offering memorandum" |
| `discretionary` | "Timing is at manager discretion; date is indicative only" |
