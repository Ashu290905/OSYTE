# LCS API Contract

All data (fund terms, holiday calendars, overlays) must be provided by the caller in the request. LCS does not read from OSYTE's database directly — it is a stateless computation service. The only data LCS persists is the Calendar Store (materialized calendars).

---

## Assumptions

1. The caller has access to the fund's liquidity terms (v15.5 schema) and provides them in full.
2. The caller provides the merged holiday list for the relevant business day centres (base + overlays already resolved).
3. Roll convention defaults to Modified Following if not specified.
4. All dates are ISO 8601 (`YYYY-MM-DD`). All timestamps are UTC.
5. `desired_amount` and `position_nav` are in the fund's operational currency.
6. The Calendar Store is the only persistent state LCS owns.

---

## 1. Compute API

### `GET /compute/dates` — Get next lifecycle dates

Returns the next actionable lifecycle date set for an instrument. No planning, no tranches — just raw dates.

**Inputs (query params):**

| Param | Type | Required | Description |
|---|---|---|---|
| `anchor_date` | date | yes | Starting point for the search |
| `anchor_type` | string | no | `as_of` (default), `target_settlement_date`, `target_dealing_date`, `target_notice_deadline` |
| `side` | string | no | `redemption` (default), `subscription` |
| `roll_convention` | string | no | `modified_following` (default), `following`, `preceding`, `modified_preceding` |
| `count` | int | no | Number of date sets to return. Default: 1. Max: 12. |

**Inputs (request body):**

```jsonc
{
  "terms": {
    // fund liquidity terms for the requested side — full v15.5 schema
    "dealing_basis": "periodic",
    "dealing_interval": {"count": 3, "unit": "month"},
    "dealing_day": {"anchor": "first", "day_type": "business"},
    "notice_period": {"days": 30, "day_type": "calendar", "direction": "before"},
    "settlement": {"days": 30, "day_type": "calendar", "direction": "after"}
  },
  "holidays": ["2026-01-01", "2026-01-19", "2026-05-25", "2026-07-03", "..."]
  // merged holiday list — caller resolves base + overlays before calling
}
```

**Output:**

```jsonc
{
  "date_sets": [
    {
      "dealing_date": "2026-10-01",
      "notice_deadline": "2026-09-01",        // null for listed assets
      "settlement_date": "2026-10-30",
      "roll_applied": "modified_following",    // null if no adjustment needed
      "unadjusted_dealing_date": null,         // non-null if roll changed the dealing date
      "notice_window_open": true               // can notice still be submitted as of anchor_date?
    }
  ],
  "dataset_version": {
    "terms_hash": "a3f8c1",
    "holiday_hash": "b7d2e4"
  },
  "warnings": []                               // e.g. "notice value_type is 'minimum'; actual may be longer"
}
```

---

### `GET /compute/plan` — Get liquidation plan

Returns a tranche schedule accounting for lockups, gates, and holdbacks. Requires amount and position data.

**Inputs (query params):**

| Param | Type | Required | Description |
|---|---|---|---|
| `anchor_date` | date | yes | As-of date for the plan |
| `desired_amount` | number | yes | Amount to redeem in fund currency |
| `position_nav` | number | yes | Current position NAV in fund currency |
| `side` | string | no | `redemption` (default) |
| `roll_convention` | string | no | `modified_following` (default) |
| `lockup_start_date` | date | conditional | Required if the fund has lockup provisions |
| `fund_nav` | number | conditional | Required if fund-level gates exist |

**Inputs (request body):**

```jsonc
{
  "terms": {
    // full fund liquidity terms — including gates, lockup, holdbacks
    "redemption_terms": {
      "dealing_basis": "periodic",
      "dealing_interval": {"count": 3, "unit": "month"},
      "dealing_day": {"anchor": "first", "day_type": "business"},
      "notice_period": {"days": 30, "day_type": "calendar", "direction": "before"},
      "settlement": {"days": 30, "day_type": "calendar", "direction": "after"}
    },
    "gates": [{"gate_level": "investor_level", "threshold_pct": 25, "measurement_period": "quarterly"}],
    "restrictions": {
      "lockup_provisions": {"hard_lockup": {"duration": {"count": 12, "unit": "month"}, "start_basis": "subscription_day"}},
      "audit_holdbacks": {"holdback_applies": true, "holdback_tiers": [{"condition": "redemption_gte_pct_account", "threshold_pct": 95, "holdback_pct": 5}]}
    }
  },
  "holidays": ["2026-01-01", "2026-01-19", "2026-05-25", "2026-07-03", "..."]
}
```

**Output:**

```jsonc
{
  "constraints_applied": {
    "lockup": {
      "active": false,
      "lockup_type": "hard",
      "expiry_date": "2026-01-15",
      "anchor_shifted": false,              // true if anchor was moved to lockup expiry
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
      "triggered": false                    // true if cumulative redemption ≥ threshold
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
      "holdback_amount": 0
    },
    {
      "tranche_number": 2,
      "amount": 2000000,
      "dealing_date": "2027-01-04",
      "notice_deadline": "2026-12-04",
      "settlement_date": "2027-02-03",
      "notice_window_open": false,
      "gate_limited": true,
      "holdback_amount": 0
    },
    {
      "tranche_number": 3,
      "amount": 1000000,
      "dealing_date": "2027-04-01",
      "notice_deadline": "2027-03-02",
      "settlement_date": "2027-05-04",
      "notice_window_open": false,
      "gate_limited": false,
      "holdback_amount": 0
    }
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
  "dataset_version": {
    "terms_hash": "a3f8c1",
    "holiday_hash": "b7d2e4"
  },
  "warnings": []
}
```

---

## 2. Calendar API

### `GET /calendars/{instrument_id}` — Get materialized calendar

Returns the stored forward-looking calendar for an instrument.

**Inputs (query params):**

| Param | Type | Required | Description |
|---|---|---|---|
| `tenant_id` | string | yes | Tenant identifier |
| `from` | date | no | Start of range. Default: today |
| `to` | date | no | End of range. Default: from + 12 months |
| `side` | string | no | `subscription`, `redemption`, or both (default) |
| `version` | string | no | Specific version ID for historical queries. Default: latest |

**Output:**

```jsonc
{
  "instrument_id": "C.444",
  "calendar_version": "v-2026-07-01T10:00:00Z",
  "range": {"from": "2026-07-01", "to": "2027-07-01"},

  "entries": [
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
      "cash_funding_deadline": "2026-09-28",
      "nav_pricing_cutoff": "2026-09-30"
    }
    // ... more entries
  ],

  "dataset_version": {
    "holiday_source": "copp_clark",
    "holiday_file_id": 5560,
    "terms_version": "C.444@2026-02-27",
    "overlay_hash": "a3f8c1"
  }
}
```

---

### `GET /calendars/{instrument_id}/changelog` — Get calendar changes

Returns what moved between calendar versions and why.

**Inputs (query params):**

| Param | Type | Required | Description |
|---|---|---|---|
| `tenant_id` | string | yes | Tenant identifier |
| `since_version` | string | no | Show changes since this version. Default: previous version |
| `from` | date | no | Filter changes affecting dates after this |
| `to` | date | no | Filter changes affecting dates before this |

**Output:**

```jsonc
{
  "instrument_id": "C.444",
  "current_version": "v-2026-07-15T08:00:00Z",
  "compared_to_version": "v-2026-07-01T10:00:00Z",
  "trigger": "holiday_data_update",

  "changes": [
    {
      "side": "redemption",
      "dealing_date": "2026-12-01",
      "field": "notice_deadline",
      "previous_value": "2026-10-30",
      "new_value": "2026-10-29",
      "reason": "Hong Kong holiday added on 2026-10-30 by Copp Clark mid-year update"
    }
  ],

  "summary": {
    "total_changes": 1,
    "subscription_affected": 0,
    "redemption_affected": 1
  }
}
```

---

### `POST /calendars/recompute` — Trigger calendar recomputation

Triggers recomputation of stored calendars. This is the only POST — it creates new calendar versions.

**Inputs (request body):**

```jsonc
{
  "tenant_id": "client-acme",
  "instrument_ids": ["C.444", "C.503"],   // omit or [] for all instruments
  "horizon_months": 24,                    // default: 24
  "trigger_reason": "holiday_data_update", // holiday_data_update | terms_update | overlay_change | scheduled_refresh

  // caller provides the updated data that triggered the recompute
  "terms": {
    "C.444": { /* full terms */ },
    "C.503": { /* full terms */ }
  },
  "holidays": {
    // per-centre merged holiday lists
    "New York": ["2026-01-01", "2026-01-19", "..."],
    "Hong Kong": ["2026-01-01", "2026-01-26", "..."]
  }
}
```

**Output:**

```jsonc
{
  "job_id": "recomp-20260701-001",
  "status": "accepted",
  "instruments_queued": 2
}
```

---

### `GET /calendars/recompute/{job_id}` — Check recomputation status

**Output:**

```jsonc
{
  "job_id": "recomp-20260701-001",
  "status": "completed",                  // queued | running | completed | partial | failed
  "instruments_queued": 2,
  "instruments_completed": 2,
  "instruments_failed": 0,
  "results": [
    {"instrument_id": "C.444", "status": "ok", "calendar_version": "v-2026-07-15T08:00:00Z", "dates_changed": 3},
    {"instrument_id": "C.503", "status": "ok", "calendar_version": "v-2026-07-15T08:00:00Z", "dates_changed": 0}
  ]
}
```

---

## 3. Error Handling

All errors follow:

```jsonc
{
  "error": "error_code",
  "message": "Human-readable description",
  "details": {}
}
```

| Code | HTTP | When |
|---|---|---|
| `incomplete_terms` | 422 | Required fields missing from terms (e.g. `dealing_basis` not provided) |
| `unschedulable_dealing_basis` | 422 | `dealing_basis` is `discretionary` or `complex` — engine can't generate dates |
| `no_valid_dealing_date` | 422 | No dealing date satisfies the anchor constraint (e.g. target settlement date too soon) |
| `lockup_start_required` | 400 | Fund has lockup but `lockup_start_date` not provided |
| `fund_nav_required` | 400 | Fund-level gates exist but `fund_nav` not provided |
| `invalid_date_range` | 400 | `from` > `to` or range exceeds 5 years |
| `calendar_not_found` | 404 | No materialized calendar for this instrument + tenant |
| `version_not_found` | 404 | Requested calendar version doesn't exist |
| `job_not_found` | 404 | Recomputation job ID not found |

### Warnings (not errors)

Returned in the `warnings` array when fund terms have non-exact values:

| `value_type` | Warning |
|---|---|
| `minimum` | "Notice period is a minimum (≥N days); actual may be longer" |
| `maximum` | "Settlement is a maximum (≤N days); actual may be shorter" |
| `estimated` | "Value is estimated; confirm with offering memorandum" |
| `discretionary` | "Timing is at manager discretion; date is indicative only" |
