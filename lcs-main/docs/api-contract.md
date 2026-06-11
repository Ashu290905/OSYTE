# LCS API Contract

All data (fund terms, holiday calendars) must be provided by the caller. LCS is a stateless computation service — it does not read from OSYTE's database. The only data LCS persists is the Calendar Store (materialized calendars).

---

## Assumptions

1. The caller provides fund liquidity terms in full (v15.5 schema).
2. The caller provides the merged holiday list (base + overlays already resolved).
3. Roll convention defaults to Modified Following if not specified.
4. All dates are ISO 8601 (`YYYY-MM-DD`). All timestamps are UTC.
5. Monetary amounts are in the fund's operational currency.
6. The Calendar Store is the only persistent state LCS owns.

---

## Data Structures

### FundTerms

The liquidity terms for one side (subscription or redemption) of an instrument. Follows the v15.5 schema.

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

### RestrictionTerms

Constraints that affect planning (lockups, gates, holdbacks). Provided alongside FundTerms for `/compute/plan`.

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
    "no_lockup": true                                   // or:
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

A flat array of ISO dates representing all non-business days for the relevant centres. The caller is responsible for merging base holidays + tenant overlays before sending.

```jsonc
["2026-01-01", "2026-01-19", "2026-01-26", "2026-02-16", "2026-05-25", "2026-07-03", "2026-09-07", "2026-11-26", "2026-11-27", "2026-12-25"]
```

### LifecycleDateSet

One complete set of dates for a single dealing date.

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

### Tranche

One tranche in a liquidation plan.

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

### ConstraintsSummary

What the planning engine evaluated before computing tranches.

```jsonc
{
  "lockup": {
    "active": false,                                    // boolean — is the position currently locked?
    "lockup_type": "hard",                              // "hard" | "soft" | null
    "expiry_date": "2026-01-15",                        // date | null
    "anchor_shifted": false,                            // boolean — was anchor moved to lockup expiry?
    "early_exit_fee_pct": null                           // number | null — for soft lockups
  },
  "gate": {
    "active": true,                                     // boolean — does a gate apply?
    "gate_level": "investor_level",                     // string | null
    "threshold_pct": 25,                                // number | null
    "max_per_period": 2000000,                          // number | null — computed from threshold_pct × position_nav
    "measurement_period": "quarterly"                   // string | null
  },
  "holdback": {
    "active": false,                                    // boolean — does a holdback apply to this fund?
    "threshold_pct": 95,                                // number | null
    "holdback_pct": 5,                                  // number | null
    "triggered": false                                  // boolean — did cumulative redemption cross the threshold?
  }
}
```

### PlanSummary

Aggregate numbers across all tranches.

```jsonc
{
  "total_redeemable": 5000000,                          // number
  "total_tranches": 3,                                  // int
  "first_cash_date": "2026-10-30",                      // date
  "last_cash_date": "2027-05-04",                       // date
  "total_holdback": 0,                                  // number
  "total_early_exit_fee": 0,                            // number
  "shortfall": 0                                        // number — desired_amount - total_redeemable
}
```

### CalendarEntry

One row in a materialized calendar.

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

### ChangelogEntry

One date that moved between calendar versions.

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

### DatasetVersion

Stamps on every response for reproducibility and audit.

```jsonc
{
  "terms_hash": "a3f8c1",                               // hash of the terms provided
  "holiday_hash": "b7d2e4"                              // hash of the holiday list provided
}
```

For Calendar API responses, the dataset version includes source info:

```jsonc
{
  "holiday_source": "copp_clark",                       // "copp_clark" | "client_supplied"
  "holiday_file_id": 5560,                              // int — which file version
  "terms_version": "C.444@2026-02-27",                  // fund_terms_version from the record
  "overlay_hash": "a3f8c1"                              // hash of tenant overlays applied
}
```

---

## 1. Compute API

### `GET /compute/dates` — Get lifecycle dates

Returns the next actionable lifecycle date sets for an instrument. No planning, no tranches — just dates.

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
  "terms": FundTerms,
  "holidays": HolidayList
}
```

**Response `200 OK`:**

```jsonc
{
  "date_sets": [LifecycleDateSet],                      // array, length = count
  "dataset_version": DatasetVersion,
  "warnings": ["string"]
}
```

---

### `GET /compute/plan` — Get liquidation plan

Returns a tranche schedule accounting for lockups, gates, and holdbacks.

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
  "terms": FundTerms,
  "restrictions": RestrictionTerms,
  "holidays": HolidayList
}
```

**Response `200 OK`:**

```jsonc
{
  "constraints": ConstraintsSummary,
  "tranches": [Tranche],
  "summary": PlanSummary,
  "dataset_version": DatasetVersion,
  "warnings": ["string"]
}
```

---

## 2. Calendar API

### `GET /calendars/{instrument_id}` — Get materialized calendar

Returns the stored forward-looking calendar for an instrument.

**Query params:**

| Param | Type | Required | Description |
|---|---|---|---|
| `tenant_id` | string | yes | Tenant identifier |
| `from` | date | no | Start of range. Default: today |
| `to` | date | no | End of range. Default: from + 12 months |
| `side` | string | no | `subscription` `redemption` or both (default) |
| `version` | string | no | Specific version ID for historical queries. Default: latest |

**Response `200 OK`:**

```jsonc
{
  "instrument_id": "string",
  "calendar_version": "string",
  "range": {"from": "date", "to": "date"},
  "entries": [CalendarEntry],
  "dataset_version": DatasetVersion                     // calendar-specific version with source info
}
```

---

### `GET /calendars/{instrument_id}/changelog` — Get calendar changes

Returns what moved between calendar versions and why.

**Query params:**

| Param | Type | Required | Description |
|---|---|---|---|
| `tenant_id` | string | yes | Tenant identifier |
| `since_version` | string | no | Show changes since this version. Default: previous version |
| `from` | date | no | Filter changes affecting dates after this |
| `to` | date | no | Filter changes affecting dates before this |

**Response `200 OK`:**

```jsonc
{
  "instrument_id": "string",
  "current_version": "string",
  "compared_to_version": "string",
  "trigger": "string",                                  // "holiday_data_update" | "terms_update" | "overlay_change" | "scheduled_refresh"
  "changes": [ChangelogEntry],
  "summary": {
    "total_changes": 0,
    "subscription_affected": 0,
    "redemption_affected": 0
  }
}
```

---

### `POST /calendars/recompute` — Trigger recomputation

Creates new calendar versions. The only write endpoint.

**Request body:**

```jsonc
{
  "tenant_id": "string",
  "instrument_ids": ["string"],                         // omit or [] for all instruments
  "horizon_months": 24,                                 // default: 24
  "trigger_reason": "string",                           // "holiday_data_update" | "terms_update" | "overlay_change" | "scheduled_refresh"
  "instruments": {                                      // keyed by instrument_id
    "C.444": {
      "subscription_terms": FundTerms,
      "redemption_terms": FundTerms,
      "restrictions": RestrictionTerms
    }
  },
  "holidays": HolidayList                               // single merged list covering all centres
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

### `GET /calendars/jobs/{job_id}` — Check recomputation status

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

All errors:

```jsonc
{
  "error": "error_code",
  "message": "Human-readable description",
  "details": {}                                         // optional structured context
}
```

| Code | HTTP | When |
|---|---|---|
| `incomplete_terms` | 422 | Required fields missing from terms |
| `unschedulable_dealing_basis` | 422 | `dealing_basis` is `discretionary` or `complex` |
| `no_valid_dealing_date` | 422 | No dealing date satisfies the constraint |
| `lockup_start_required` | 400 | Fund has lockup but `lockup_start_date` not provided |
| `fund_nav_required` | 400 | Fund-level gates exist but `fund_nav` not provided |
| `invalid_date_range` | 400 | `from` > `to` or range exceeds 5 years |
| `calendar_not_found` | 404 | No materialized calendar for this instrument + tenant |
| `version_not_found` | 404 | Requested calendar version doesn't exist |
| `job_not_found` | 404 | Job ID not found |

### Warnings

Returned in the `warnings` array when terms have non-exact values:

| `value_type` | Warning |
|---|---|
| `minimum` | "Notice period is a minimum (≥N days); actual may be longer" |
| `maximum` | "Settlement is a maximum (≤N days); actual may be shorter" |
| `estimated` | "Value is estimated; confirm with offering memorandum" |
| `discretionary` | "Timing is at manager discretion; date is indicative only" |
