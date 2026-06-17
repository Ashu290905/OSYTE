# Liquidity Calendar Service (LCS) API Contract

## What LCS does

Enterprise clients hold portfolios of instruments — each with different dealing frequencies, notice periods, settlement timelines, lockups, gates, and holdbacks. Today, ops analysts manually count business days against holiday calendars to figure out when they can trade, when they need to submit notice, and when cash lands. LCS replaces that with deterministic computation.

**LCS solves three problems:**

1. **"When are the next key dates for this instrument?"** — Given an instrument's liquidity terms and business day centres, compute the next trade date, notice deadline, and settlement date.

2. **"How do I redeem $X from this position?"** — Given the same terms plus the investor's position size, constraints (lockups, gates, holdbacks), and previous transaction history, produce a tranche-by-tranche schedule showing how much can be redeemed on which dates.

3. **"Give me a pre-built calendar for this instrument."** — For downstream systems that need all dates pre-computed, maintain a stored forward-looking calendar that can be rebuilt when holidays or terms change.

---

## Methods

Five methods across two APIs:

### Liquidity Dates API

Stateless computation. Nothing is stored. The caller sends the instrument's terms and calendar IDs, LCS resolves holidays internally and computes dates.

| # | Method name | Route | Solves | What it does |
|---|---|---|---|---|
| 1 | `getHolidayCalendars()` | `GET /liquidity-dates/getHolidayCalendars` | — | Returns the list of holiday calendars LCS has access to, with their IDs |
| 2 | `getNextTransactionDates()` | `POST /liquidity-dates/getNextTransactionDates` | Problem 1 | Returns the next actionable trade dates with their notice deadlines and settlement dates |
| 3 | `getProposedTransaction()` | `POST /liquidity-dates/getProposedTransaction` | Problem 2 | Returns a tranche-by-tranche redemption schedule accounting for lockups, gates, holdbacks, and previous transactions |

### Forward Calendar API

Persistent storage. Maintains forward-looking calendars that downstream systems can query without re-computing.

| # | Method name | Route | Solves | What it does |
|---|---|---|---|---|
| 4 | `getInstrumentCalendar()` | `GET /forward-calendar/getInstrumentCalendar/{instrument_id}` | Problem 3 | Returns the stored forward-looking calendar for an instrument |
| 5 | `updateInstrumentCalendar()` | `POST /forward-calendar/updateInstrumentCalendar` | Problem 3 | Builds the stored calendar for a single instrument — same engine as `getNextTransactionDates()` but generates all dates until the end of the holiday data and stores them |

---

## Assumptions

### 1. The caller provides the instrument's liquidity terms

LCS does not read liquidity terms from OSYTE's database. The caller sends the instrument's `subscriptionTerms` or `redemptionTerms` (and constraints for `getProposedTransaction()`) with each request.

**Why:** Liquidity terms are instrument-specific and the caller already has them. Keeping LCS out of the terms database avoids authorization complexity and cache staleness.

### 2. LCS has access to holiday calendars

LCS has direct access to holiday calendar data. The caller does not pass holiday lists — they pass calendar IDs (obtained via `getHolidayCalendars()`), and LCS resolves the holidays internally.

**Why:** Holiday data is large and shared across instruments. Having the caller pass it with every request is wasteful. LCS can fetch and cache it efficiently.

### 3. Dates are ISO 8601

All dates are `YYYY-MM-DD`. Cutoff times include a timezone (e.g. `cutoffHour: 17`, `cutoffTimezone: "New York"`).

**Why:** Unambiguous date format. Cutoff times are timezone-specific because notice deadlines are governed by the fund's local business hours, not UTC.

### 4. The only persistent data LCS owns is the Forward Calendar Store

The Liquidity Dates API is fully stateless. The Forward Calendar API stores materialized calendars — that's the only write path in LCS.

**Why:** Downstream systems (Investment Planning, Rebalancing, reporting) need pre-built calendars they can query without sending full terms every time. But the computation itself remains stateless.

### 5. Calendar builds are one instrument at a time

`updateInstrumentCalendar()` is called once per instrument. Forward fills, generating calendars for new instruments, and updating calendars after holiday changes all work the same way — the caller calls `updateInstrumentCalendar()` with the instrument's terms and calendar IDs. The only difference is which instruments are affected:

- **Forward fill / new instrument:** Call `updateInstrumentCalendar()` for each instrument that needs a calendar.
- **Holiday change:** The caller identifies which instruments use the affected calendar and calls `updateInstrumentCalendar()` for each one.
- **Terms change:** The caller calls `updateInstrumentCalendar()` for the instrument whose terms changed.

**Why:** LCS doesn't have access to the full instrument roster or know which instruments use which calendars. The caller has that information and can filter efficiently before calling LCS.

---

## Liquidity Dates API: `getHolidayCalendars()`

**Route:** `GET /liquidity-dates/getHolidayCalendars`

**Purpose:** "What holiday calendars does LCS have?"

Returns the list of holiday calendars LCS has access to. Each calendar has an ID that the caller passes to other methods instead of centre names.

### What the caller sends

No inputs required — this is a simple lookup.

### Example

**Request:** `GET /liquidity-dates/getHolidayCalendars`

**Response:**
```jsonc
{
  "calendars": [
    {"calendar_id": "nyse-2026", "centre": "New York", "source": "copp_clark", "coverage": {"from": "2026-01-01", "to": "2056-12-31"}},
    {"calendar_id": "lse-2026", "centre": "London", "source": "copp_clark", "coverage": {"from": "2026-01-01", "to": "2056-12-31"}},
    {"calendar_id": "hkex-2026", "centre": "Hong Kong", "source": "copp_clark", "coverage": {"from": "2026-01-01", "to": "2056-12-31"}},
    {"calendar_id": "cayman-2026", "centre": "Cayman Islands", "source": "copp_clark", "coverage": {"from": "2026-01-01", "to": "2056-12-31"}}
  ]
}
```

### `getHolidayCalendars()` output signature

```jsonc
{
  "calendars": [
    {
      "calendar_id": "string",
      "centre": "string",
      "source": "string",
      "coverage": {"from": "YYYY-MM-DD", "to": "YYYY-MM-DD"}
    }
  ]
}
```

---

## Liquidity Dates API: `getNextTransactionDates()`

**Route:** `POST /liquidity-dates/getNextTransactionDates`

**Purpose:** "When are the next key dates for this instrument?"

Given an instrument's liquidity terms and calendar IDs, returns the next actionable trade dates with notice deadlines and settlement dates. LCS resolves holidays internally from the calendars provided.

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `instrument_id` | string | yes | The instrument to compute dates for |
| `side` | string | yes | Which side of the instrument: `redemption` or `subscription` |
| `subscriptionTerms` or `redemptionTerms` | object | yes | The full terms block for the requested side. See fields below. |
| `anchor_date` | date | no | The reference date. Default: today. |
| `anchor_type` | string | no | How to interpret `anchor_date`. Default: `today`. Options: `today`, `target_settlement_date`, `target_trade_date`, `target_notice_deadline` |
| `calendar_ids` | string[] | yes | IDs of the holiday calendars to use (from `getHolidayCalendars()`). E.g. `["nyse-2026", "cayman-2026"]`. |
| `count` | int | no | How many date sets to return. Default: 1 |

**Fields needed from `redemptionTerms`:**

| Field | Type | What it means |
|---|---|---|
| `dealingBasis` | string | How often dealing occurs: `periodic`, `anniversary`, `at_closing`, `at_maturity`, `discretionary`, `complex` |
| `dealingInterval` | object | Recurrence period. E.g. `{"count": 3, "unit": "month"}`. Required when `dealingBasis` is `periodic` or `anniversary`. |
| `dealingDay` | object | Which day within the period. E.g. `{"anchor": "first", "dayType": "business"}` |
| `noticePeriod` | object | Notice requirement. Contains `days`, `dayType`, `direction`, `relativeTo`, `valueType`, optionally `businessDayCenters`, `cutoffHour` (0-23), `cutoffTimezone`. |
| `settlement` | object | Settlement timing. Contains `days`, `dayType`, `direction`, `relativeTo`, `valueType`, optionally `inSpeciePermitted`, `inSpecieConditions`. |
| `redemptionSchedule` | object | Complex redemption scheduling (tiered tranches, anniversary-based). Only present for funds with non-standard structures. |

**Fields needed from `subscriptionTerms`:**

| Field | Type | What it means |
|---|---|---|
| `dealingBasis` | string | Same as redemption |
| `dealingInterval` | object | Same as redemption |
| `dealingDay` | object | Same as redemption |
| `documentDeadline` | object | Deadline for subscription application forms. Contains `days`, `dayType`, `direction`, `relativeTo`, `valueType`, optionally `cutoffHour`, `cutoffMinute`, `cutoffTimezone`. |
| `cashFundingDeadline` | object | Deadline for cleared subscription funds. Contains `days`, `dayType`, `direction`, `relativeTo`, `valueType`, optionally `cutoffHour`, `cutoffMinute`, `cutoffTimezone`. |

`anchor_type` explained:
- `today` (default) — "Starting from this date (or today if `anchor_date` is omitted), what's the next trade date I can still act on?"
- `target_settlement_date` — "I need cash by this date — what's the latest trade date that settles in time?"
- `target_trade_date` — "I know the trade date — give me the notice deadline and settlement date"
- `target_notice_deadline` — "I can submit notice by this date — which trade date does that catch?"

### Example — "I need cash by October 31st"

Fund B: quarterly trading (1st business day), 30-day notice, 30-day settlement. Centres: New York + Cayman Islands.

**Request:**
```jsonc
POST /liquidity-dates/getNextTransactionDates

{
  "instrument_id": "C.444",
  "side": "redemption",
  "redemptionTerms": {
    "dealingBasis": "periodic",
    "dealingInterval": {"count": 3, "unit": "month"},
    "dealingDay": {"anchor": "first", "dayType": "business"},
    "noticePeriod": {"days": 30, "dayType": "calendar", "direction": "before", "relativeTo": "redemption_day", "valueType": "exact", "cutoffHour": 17, "cutoffTimezone": "New York"},
    "settlement": {"days": 30, "dayType": "calendar", "direction": "after", "relativeTo": "redemption_day", "valueType": "exact"}
  },
  "anchor_date": "2026-10-31",
  "anchor_type": "target_settlement_date",
  // anchor_type tells LCS: "I need cash BY this date, work backward"
  "calendar_ids": ["nyse-2026", "cayman-2026"]
}
```

Just `redemptionTerms` and `calendar_ids` — no holidays to pass. LCS resolves holidays from the calendar IDs internally.

```
Engine works backward: Q4 trade date Oct 1 → settlement Oct 1 + 30 days = Oct 31 (Sat) → rolls to Oct 30 (Fri).
Oct 30 ≤ target Oct 31 → yes. Notice: Oct 1 − 30 days = Sep 1.
```

**Response:**
```jsonc
{
  "results": [
    {
      "trade_date": "2026-10-01",
      "notice_deadline": {"date": "2026-09-01", "cutoff_hour": 17, "cutoff_timezone": "New York"},
      "settlement_date": "2026-10-30"
    }
  ]
}
```

### `getNextTransactionDates()` output signature

```jsonc
{
  "results": [
    {
      "trade_date": "YYYY-MM-DD",
      "notice_deadline": "{ date: YYYY-MM-DD, cutoff_hour: int (0-23), cutoff_timezone: string } | null",
      "settlement_date": "YYYY-MM-DD | null"
    }
  ]
}
```

---

## Liquidity Dates API: `getProposedTransaction()`

**Route:** `POST /liquidity-dates/getProposedTransaction`

**Purpose:** "How do I redeem $X from this position?"

Same as Method 1, but the caller also provides the amount, position size, redemption constraints (lockups, gates, holdbacks), and previous transaction history. LCS evaluates the constraints — including how much gate capacity has already been used — splits into tranches if needed, and returns a full schedule.

### What the caller sends

**Inputs (ordered by workflow: constraints first, then dates):**

| Param | Type | Required | What it means |
|---|---|---|---|
| `instrument_id` | string | yes | The instrument to plan redemption for |
| `side` | string | yes | `redemption` (or `subscription` for buy-side planning) |
| `redemption_amount` | float | yes | How much the investor wants to redeem |
| `position_nav` | float | yes | Current position value |
| `lockup_start_date` | date | conditional | When the investor subscribed. Required if the fund has a lockup. |
| `previous_transactions` | array | no | Previous redemption history for this investor + instrument. Used to determine which gate number the investor is on. See format below. |
| `restrictions` | object | no | The full `restrictions` block. See fields below. Omit if no restrictions. |
| `gates` | object[] | no | The full `gates` array. See fields below. Omit if no gates. |
| `redemptionTerms` | object | yes | The full `redemptionTerms` block (same fields as Method 1). |
| `anchor_date` | date | no | The reference date. Default: today. |
| `calendar_ids` | string[] | yes | IDs of the holiday calendars to use (from `getHolidayCalendars()`). |

**`previous_transactions` format:**

```jsonc
[
  {"trade_date": "2026-04-01", "amount": 2000000}
]
```

Each entry is a past redemption for this investor on this instrument. LCS uses this to determine which gate number the investor is on. Only one redemption is allowed per gate period (e.g. per quarter). The number of previous transactions tells LCS which gate applies next — for example, if the fund has a tiered gate schedule (25% → 33.3% → 50% → 100%) and the investor has one previous transaction, they're on gate 2 and can redeem up to 33.3% of their current NAV.

**Fields needed from `restrictions`:**

| Field | Type | What it means |
|---|---|---|
| `lockupProvisions` | object | Contains `noLockup`, `hardLockup`, or `softLockup`. Hard lockup has `lockupType`, `duration` (`count`, `unit`), `startBasis`. |
| `lockupExceptions` | object | Whether early exit from lockup is available and under what conditions. |
| `auditHoldbacks` | object | Contains `holdbackApplies` (boolean with availability/valueType), `holdbackTiers` (array with `condition`, `thresholdPct`, `holdbackPct`, `holdbackReleaseTrigger`), `periodEndBasis`. |
| `transferRestrictions` | object | Whether transfers are permitted, consent required. |

**Fields needed from `gates`:**

Each gate object contains:

| Field | Type | What it means |
|---|---|---|
| `gateLevel` | string | `investor_level`, `class_level`, `fund_level`, `master_fund_level` |
| `gateBasis` | string | `nav_percentage`, `fixed_amount`, `nav_drawdown` |
| `thresholdPct` | float | Percentage that triggers the gate (0-100) |
| `thresholdBasis` | string | What the threshold is measured against: `investor_holding`, `class_nav`, `fund_nav` |
| `measurementPeriod` | string | `per_redemption_day`, `monthly`, `quarterly`, `annually` |

**Not needed from the liquidity terms JSON:** `metadata`, `instrument`, `subscriptionTerms`, `redemptionFees`, `governance`, `context`.

### Example — Redeem $5M from a position (with prior transaction)

Fund B. Subscribed Jan 15, 2025. 12-month hard lockup. Tiered gate: 25% → 33.3% → 50% → 100% of holding per quarter. 5% holdback on ≥95%. The investor already redeemed 25% ($2M) in the previous quarter (gate 1 done). Current NAV is $6M.

**Request:**
```jsonc
POST /liquidity-dates/getProposedTransaction

{
  "instrument_id": "C.444",
  "side": "redemption",
  "redemption_amount": 5000000,
  "position_nav": 6000000,
  "lockup_start_date": "2025-01-15",
  "previous_transactions": [
    {"trade_date": "2026-04-01", "amount": 2000000}
  ],
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
    "noticePeriod": {"days": 30, "dayType": "calendar", "direction": "before", "relativeTo": "redemption_day", "valueType": "exact", "cutoffHour": 17, "cutoffTimezone": "New York"},
    "settlement": {"days": 30, "dayType": "calendar", "direction": "after", "relativeTo": "redemption_day", "valueType": "exact"}
  },
  "anchor_date": "2026-06-16",
  "calendar_ids": ["nyse-2026", "cayman-2026"]
}
```

```
Lockup: expired Jan 15, 2026. Today Jun 16 → unlocked.
Gate: 1 previous transaction → investor is on gate 2 (33.3% of current NAV).
  Gate 2: 33.3% of $6M = $2,000,000
Holdback: $5M / $6M = 83.3% < 95% → not triggered.
T1: Q4 Oct 1 → gate 2 → 33.3% of $6M = $2,000,000
T2: Q1 Jan 4 → gate 3 → 50% of $4M (remaining) = $2,000,000
T3: Q2 Apr 1 → gate 4 → 100% of $2M (remaining) = $1,000,000 (only $1M left to reach $5M target)
```

**Response:**
```jsonc
{
  "applied_constraints": {
    "lockup": {"active": false, "lockup_type": "hard", "expiry_date": "2026-01-15", "anchor_shifted": false, "early_exit_fee_pct": null},
    "gate": {"active": true, "gate_level": "investor_level", "current_gate_number": 2, "current_gate_pct": 33.3, "measurement_period": "quarterly"},
    "holdback": {"active": false, "threshold_pct": 95, "holdback_pct": 5, "triggered": false}
  },
  "tranches": [
    {"tranche_number": 1, "amount": 2000000, "trade_date": "2026-10-01", "notice_deadline": {"date": "2026-09-01", "cutoff_hour": 17, "cutoff_timezone": "New York"}, "settlement_date": "2026-10-30", "gate_number": 2, "gate_pct": 33.3, "gate_full": true, "holdback_amount": 0, "early_exit_fee": 0},
    {"tranche_number": 2, "amount": 2000000, "trade_date": "2027-01-04", "notice_deadline": {"date": "2026-12-04", "cutoff_hour": 17, "cutoff_timezone": "New York"}, "settlement_date": "2027-02-03", "gate_number": 3, "gate_pct": 50, "gate_full": true, "holdback_amount": 0, "early_exit_fee": 0},
    {"tranche_number": 3, "amount": 1000000, "trade_date": "2027-04-01", "notice_deadline": {"date": "2027-03-02", "cutoff_hour": 17, "cutoff_timezone": "New York"}, "settlement_date": "2027-05-04", "gate_number": 4, "gate_pct": 100, "gate_full": false, "holdback_amount": 0, "early_exit_fee": 0}
  ],
  "summary": {
    "redeemable": 5000000,
    "tranches": 3,
    "holdback": 0,
    "early_exit_fee": 0
  }
}
```

### `getProposedTransaction()` output signature

```jsonc
{
  "applied_constraints": {
    "lockup": {"active": "boolean", "lockup_type": "string | null", "expiry_date": "YYYY-MM-DD | null", "anchor_shifted": "boolean", "early_exit_fee_pct": "float | null"},
    "gate": {"active": "boolean", "gate_level": "string | null", "current_gate_number": "int | null", "current_gate_pct": "float | null", "measurement_period": "string | null"},
    "holdback": {"active": "boolean", "threshold_pct": "float | null", "holdback_pct": "float | null", "triggered": "boolean"}
  },
  "tranches": [
    {
      "tranche_number": "int",
      "amount": "float",
      "trade_date": "YYYY-MM-DD",
      "notice_deadline": "{ date: YYYY-MM-DD, cutoff_hour: int (0-23), cutoff_timezone: string } | null",
      "settlement_date": "YYYY-MM-DD | null",
      "gate_number": "int | null",
      "gate_pct": "float | null",
      "gate_full": "boolean",
      "holdback_amount": "float",
      "early_exit_fee": "float"
    }
  ],
  "summary": {
    "redeemable": "float",
    "tranches": "int",
    "holdback": "float",
    "early_exit_fee": "float"
  }
}
```

---

## Forward Calendar API: `getInstrumentCalendar()`

**Route:** `GET /forward-calendar/getInstrumentCalendar/{instrument_id}`

**Purpose:** "Give me the full pre-built calendar for this instrument."

Unlike Methods 1 and 2 which compute on the fly, this reads from the stored Forward Calendar. The calendar is pre-computed and updated when holidays or terms change (see Method 4).

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `instrument_id` | string | yes | The instrument (in the URL path) |
| `side` | string | no | `subscription`, `redemption`, or both (default) |
| `tenant_id` | string | yes | Which tenant's calendar (different tenants may have different holiday overlays) |
| `count` | int | no | Number of trade dates to return (e.g. "next 10 trade dates"). Mutually exclusive with `from`/`to`. |
| `from` | date | no | Start of range. Default: today. Ignored if `count` is provided. |
| `to` | date | no | End of range. Default: end of holiday calendar data. Ignored if `count` is provided. |

No liquidity terms or holidays needed — the data is already stored in the calendar.

### Example — "Give me the next 4 quarterly redemption dates"

**Request:** `GET /forward-calendar/getInstrumentCalendar/C.444?tenant_id=client-acme&side=redemption&count=4`

**Response:**
```jsonc
{
  "instrument_id": "C.444",
  "tenant_id": "client-acme",
  "rows": [
    {"side": "redemption", "trade_date": "2026-10-01", "notice_deadline": {"date": "2026-09-01", "cutoff_hour": 17, "cutoff_timezone": "New York"}, "settlement_date": "2026-10-30"},
    {"side": "redemption", "trade_date": "2027-01-04", "notice_deadline": {"date": "2026-12-04", "cutoff_hour": 17, "cutoff_timezone": "New York"}, "settlement_date": "2027-02-03"},
    {"side": "redemption", "trade_date": "2027-04-01", "notice_deadline": {"date": "2027-03-02", "cutoff_hour": 17, "cutoff_timezone": "New York"}, "settlement_date": "2027-05-04"},
    {"side": "redemption", "trade_date": "2027-07-01", "notice_deadline": {"date": "2027-06-01", "cutoff_hour": 17, "cutoff_timezone": "New York"}, "settlement_date": "2027-07-31"}
  ]
}
```

### Example — Both sides, date range

**Request:** `GET /forward-calendar/getInstrumentCalendar/C.444?tenant_id=client-acme&from=2026-10-01&to=2027-01-31`

**Response:**
```jsonc
{
  "instrument_id": "C.444",
  "tenant_id": "client-acme",
  "range": {"from": "2026-10-01", "to": "2027-01-31"},
  "rows": [
    {"side": "subscription", "trade_date": "2026-10-01", "document_deadline": "2026-09-25", "cash_funding_deadline": "2026-09-28"},
    {"side": "redemption",   "trade_date": "2026-10-01", "notice_deadline": {"date": "2026-09-01", "cutoff_hour": 17, "cutoff_timezone": "New York"}, "settlement_date": "2026-10-30"},
    {"side": "subscription", "trade_date": "2026-11-02", "document_deadline": "2026-10-27", "cash_funding_deadline": "2026-10-29"},
    {"side": "subscription", "trade_date": "2026-12-01", "document_deadline": "2026-11-25", "cash_funding_deadline": "2026-11-27"},
    {"side": "subscription", "trade_date": "2027-01-04", "document_deadline": "2026-12-29", "cash_funding_deadline": "2026-12-31"},
    {"side": "redemption",   "trade_date": "2027-01-04", "notice_deadline": {"date": "2026-12-04", "cutoff_hour": 17, "cutoff_timezone": "New York"}, "settlement_date": "2027-02-03"}
  ]
}
```

### `getInstrumentCalendar()` output signature

```jsonc
{
  "instrument_id": "string",
  "tenant_id": "string",
  "range": {"from": "YYYY-MM-DD", "to": "YYYY-MM-DD"},     // present when from/to used
  "rows": [
    {
      "side": "redemption | subscription",
      "trade_date": "YYYY-MM-DD",
      "notice_deadline": "{ date: YYYY-MM-DD, cutoff_hour: int (0-23), cutoff_timezone: string } | null",
      "settlement_date": "YYYY-MM-DD | null",
      "document_deadline": "YYYY-MM-DD | null",
      "cash_funding_deadline": "YYYY-MM-DD | null"
    }
  ]
}
```

### Why this exists separately from Method 1

Method 1 computes on every call — the caller sends terms each time. That's fine for one-off queries ("when's my next trade date?") but bad for systems that need the full calendar for reporting, LP exports, or dashboards. Method 3 serves pre-built data — fast reads, no computation, no need to send terms.

---

## Forward Calendar API: `updateInstrumentCalendar()`

**Route:** `POST /forward-calendar/updateInstrumentCalendar`

**Purpose:** "Rebuild the stored calendar for this instrument."

Works exactly like `getNextTransactionDates()` — same engine, same inputs — but instead of returning the next N dates, it generates every trade date from today until the end of the holiday calendar data (currently Copp Clark covers up to 2056) and stores the results in the Forward Calendar Store.

Called once per instrument:
- **Scheduled forward fill:** A cron job loops through all instruments and calls `updateInstrumentCalendar()` for each.
- **Holiday data change:** The caller identifies which instruments use the affected centre and calls `updateInstrumentCalendar()` once per affected instrument.
- **Terms change:** The caller calls `updateInstrumentCalendar()` for the instrument whose terms changed.

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `instrument_id` | string | yes | The instrument to rebuild the calendar for |
| `tenant_id` | string | yes | Which tenant's calendar to rebuild |
| `side` | string | no | `subscription`, `redemption`, or both (default) |
| `subscriptionTerms` | object | conditional | The full `subscriptionTerms` block. Required if `side` includes subscription. |
| `redemptionTerms` | object | conditional | The full `redemptionTerms` block. Required if `side` includes redemption. |
| `calendar_ids` | string[] | yes | IDs of the holiday calendars to use (from `getHolidayCalendars()`). |

Same fields as `getNextTransactionDates()` — no `anchor_date`, no `anchor_type`, no `count`. Uses `calendar_ids` instead of centre names. The engine starts from today and runs until the holiday data ends.

### Example — Rebuild after a Hong Kong holiday update

Copp Clark added a new holiday on 2026-10-30 for Hong Kong. The caller identified C.503 as an affected instrument (it uses the Hong Kong calendar) and calls `updateInstrumentCalendar()` for it.

**Request:**
```jsonc
POST /forward-calendar/updateInstrumentCalendar

{
  "instrument_id": "C.503",
  "tenant_id": "client-acme",
  "subscriptionTerms": {
    "dealingBasis": "periodic",
    "dealingInterval": {"count": 1, "unit": "month"},
    "dealingDay": {"anchor": "first", "dayType": "business"}
  },
  "redemptionTerms": {
    "dealingBasis": "periodic",
    "dealingInterval": {"count": 1, "unit": "month"},
    "dealingDay": {"anchor": "first", "dayType": "business"},
    "noticePeriod": {"days": 30, "dayType": "calendar", "direction": "before", "relativeTo": "redemption_day", "valueType": "exact", "cutoffHour": 17, "cutoffTimezone": "Hong Kong"},
    "settlement": {"days": 20, "dayType": "calendar", "direction": "after", "relativeTo": "redemption_day", "valueType": "exact"}
  },
  "calendar_ids": ["hkex-2026"]
}
```

One instrument, one call. If C.612 is also affected, the caller makes a separate call for it.

**Response:**
```jsonc
{
  "instrument_id": "C.503",
  "tenant_id": "client-acme",
  "status": "accepted",
  "dates_generated_to": "2056-12-31"
}
```

### `updateInstrumentCalendar()` output signature

```jsonc
{
  "instrument_id": "string",
  "tenant_id": "string",
  "status": "accepted",
  "dates_generated_to": "YYYY-MM-DD"
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
| `missing_required_terms` | 422 | Required fields missing from terms (e.g. no `dealingBasis`) | `getNextTransactionDates`, `getProposedTransaction` |
| `unschedulable_trade` | 422 | `dealingBasis` is `discretionary` or `complex` — engine can't generate dates | `getNextTransactionDates`, `getProposedTransaction` |
| `no_reachable_trade_date` | 422 | No trade date satisfies the anchor constraint (e.g. target settlement too soon) | `getNextTransactionDates`, `getProposedTransaction` |
| `lockup_start_date_required` | 400 | Fund has lockup but `lockup_start_date` not provided | `getProposedTransaction` |
| `invalid_date_range` | 400 | `from` > `to` or range exceeds 5 years | `getInstrumentCalendar` |
| `calendar_not_found` | 404 | No stored calendar for this instrument + tenant | `getInstrumentCalendar` |

---

## Open Questions

### 1. How should we handle roll conventions?

When a computed trade date falls on a non-business day, the engine needs to decide whether to move it forward or backward. For example: an instrument deals on the 15th of every month. July 15th, 2026 is a Saturday. Should the trade date move back to Friday July 14th, or forward to Monday July 17th? This is what a roll convention determines — Modified Following would roll to Monday unless that crosses a month boundary, Preceding would roll to Friday, and so on.

The problem is that the v15.5 liquidity terms schema has no roll convention field. There's nothing in the data that tells us which convention to use for a given instrument. We need to decide: should LCS always use a default (Modified Following is the most common), should the caller pass it as a parameter, or should it be added to the liquidity terms schema?

### 2. How do tenant overlays affect the dates and their calculation?

Tenant overlays add extra holidays for a specific tenant — for example, Morgan Stanley might have additional firm-wide closure days that aren't in the Copp Clark base calendar. How does this affect date calculation?

### 3. How do we access tenant overlays?

How do we access tenant overlays — the same way as base holiday calendars through `getHolidayCalendars()`? If so, do we show all tenant overlays to everyone, or do we take a `tenant_id` as input to filter them?

### 4. Do we need instrument_id as an input?

Currently `getNextTransactionDates()` and `getProposedTransaction()` take `instrument_id` as a required input, but LCS doesn't use it for any computation — the caller already passes the full liquidity terms and calendar IDs. The only thing LCS does with it is echo it back in the response. Should we keep it for traceability and logging, make it optional, or remove it entirely?
