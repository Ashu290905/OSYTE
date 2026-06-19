# Liquidity Calendar Service (LCS) API Contract

## What LCS does

Enterprise clients hold portfolios of instruments — each with different dealing frequencies, notice periods, settlement timelines, lockups, gates, and holdbacks. Today, ops analysts manually count business days against holiday calendars to figure out when they can trade, when they need to submit notice, and when cash lands. LCS replaces that with deterministic computation.

**LCS solves three problems:**

1. **"When are the next key dates for this instrument?"** — Given an instrument's liquidity terms and business day centres, compute the next trade date, notice deadline, and settlement date.

2. **"How do I redeem $X from this position?"** — Given the same terms plus the investor's position size, constraints (lockups, gates, holdbacks), and previous transaction history, produce a tranche-by-tranche schedule showing how much can be redeemed on which dates.

3. **"Give me a pre-built calendar for this instrument."** — For downstream systems that need all dates pre-computed, maintain a stored forward-looking calendar that can be rebuilt when holidays or terms change.

---

## Methods

Six methods across two APIs:

### Liquidity Dates API

Stateless computation. Nothing is stored. The caller sends the instrument's terms and calendar IDs, LCS resolves holidays internally and computes dates.

| # | Method name | Route | Solves | What it does |
|---|---|---|---|---|
| 1 | `getHolidayCalendars()` | `GET /liquidity-dates/getHolidayCalendars` | — | Returns the list of holiday calendars LCS has access to (base + overlays), with their IDs |
| 2 | `getNextTransactionDates()` | `POST /liquidity-dates/getNextTransactionDates` | Problem 1 | Returns the next actionable trade dates with their notice deadlines and settlement dates |
| — | `isValidTransactionDate()` | `POST /liquidity-dates/isValidTransactionDate` | Problem 1 | Given an instrument's terms, a date, and a date type, returns whether the date is valid |
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

LCS has direct access to holiday calendar data (base Copp Clark calendars and tenant overlays). The caller does not pass holiday lists — they pass calendar IDs (obtained via `getHolidayCalendars()`), and LCS resolves the holidays internally.

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

Returns all holiday calendars LCS has access to — both base Copp Clark calendars and tenant overlays. Each calendar has an ID that the caller passes to other methods.

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `tenant_id` | string | no | If provided, includes that tenant's overlay calendars alongside the base calendars. If omitted, only base calendars are returned. |

### Example

**Request:** `GET /liquidity-dates/getHolidayCalendars?tenant_id=client-acme`

**Response:**
```jsonc
{
  "calendars": [
    {"calendar_id": "nyse-2026", "centre": "New York", "type": "base", "source": "copp_clark", "coverage": {"from": "2026-01-01", "to": "2056-12-31"}},
    {"calendar_id": "lse-2026", "centre": "London", "type": "base", "source": "copp_clark", "coverage": {"from": "2026-01-01", "to": "2056-12-31"}},
    {"calendar_id": "hkex-2026", "centre": "Hong Kong", "type": "base", "source": "copp_clark", "coverage": {"from": "2026-01-01", "to": "2056-12-31"}},
    {"calendar_id": "cayman-2026", "centre": "Cayman Islands", "type": "base", "source": "copp_clark", "coverage": {"from": "2026-01-01", "to": "2056-12-31"}},
    {"calendar_id": "acme-overlay-2026", "centre": null, "type": "overlay", "source": "client-acme", "coverage": {"from": "2026-01-01", "to": "2026-12-31"}}
  ]
}
```

### `getHolidayCalendars()` output signature

```jsonc
{
  "calendars": [
    {
      "calendar_id": "string",
      "centre": "string | null",
      "type": "base | overlay",
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
| `transactionType` | string | yes | Which side of the instrument: `redemption` or `subscription` |
| `subscriptionTerms` or `redemptionTerms` | object | yes | The full terms block for the requested side, passed as-is from the liquidity terms JSON. |
| `date` | date | no | The reference date. Default: today. |
| `date_type` | string | no | How to interpret `date`. Default: `as_of`. Options: `as_of`, `settlement_date`, `trade_date`, `notice_deadline` |
| `calendar_ids` | string[] | yes | IDs of the holiday calendars to use (from `getHolidayCalendars()`). E.g. `["nyse-2026", "cayman-2026"]`. |
| `count` | int | no | How many date sets to return. Default: 1 |

`date_type` explained:
- `as_of` (default) — "Starting from this date (or today if `date` is omitted), what's the next trade date I can still act on?"
- `settlement_date` — "I need cash by this date — what's the latest trade date that settles in time?"
- `trade_date` — "I know the trade date — give me the notice deadline and settlement date"
- `notice_deadline` — "I can submit notice by this date — which trade date does that catch?"

### Example — Redemption: "I need cash by October 31st"

Fund B: quarterly trading (1st business day), 30-day notice, 30-day settlement. Centres: New York + Cayman Islands.

**Request:**
```jsonc
POST /liquidity-dates/getNextTransactionDates

{
  "instrument_id": "C.444",
  "transactionType": "redemption",
  "redemptionTerms": {
    "dealingBasis": "periodic",
    "dealingInterval": {"count": 3, "unit": "month"},
    "dealingDay": {"anchor": "first", "dayType": "business"},
    "noticePeriod": {"days": 30, "dayType": "calendar", "direction": "before", "relativeTo": "redemption_day", "valueType": "exact", "cutoffHour": 17, "cutoffTimezone": "New York"},
    "settlement": {"days": 30, "dayType": "calendar", "direction": "after", "relativeTo": "redemption_day", "valueType": "exact"}
  },
  "date": "2026-10-31",
  "date_type": "settlement_date",
  "calendar_ids": ["nyse-2026", "cayman-2026"]
}
```

```
Engine works backward: Q4 trade date Oct 1 → settlement Oct 1 + 30 days = Oct 31 (Sat) → rolls to Oct 30 (Fri).
Oct 30 ≤ target Oct 31 → yes. Notice: Oct 1 − 30 days = Sep 1.
```

**Response:**
```jsonc
{
  "results": [
    {
      "date": "2026-10-01",
      "notice_deadline": {"date": "2026-09-01", "cutoff_hour": 17, "cutoff_timezone": "New York"},
      "settlement_date": "2026-10-30",
      "citation": {"dealingBasis": "periodic", "dealingInterval": {"count": 3, "unit": "month"}, "dealingDay": {"anchor": "first", "dayType": "business"}}
    }
  ]
}
```

### Example — Subscription: "When is the next subscription date?"

Fund B: monthly subscription (1st business day), documents due 5 business days before, cash funding due 2 business days before.

**Request:**
```jsonc
POST /liquidity-dates/getNextTransactionDates

{
  "instrument_id": "C.444",
  "transactionType": "subscription",
  "subscriptionTerms": {
    "dealingBasis": "periodic",
    "dealingInterval": {"count": 1, "unit": "month"},
    "dealingDay": {"anchor": "first", "dayType": "business"},
    "documentDeadline": {"days": 5, "dayType": "business", "direction": "before", "relativeTo": "dealing_day", "valueType": "exact", "cutoffHour": 17, "cutoffTimezone": "New York"},
    "cashFundingDeadline": {"days": 2, "dayType": "business", "direction": "before", "relativeTo": "dealing_day", "valueType": "exact"}
  },
  "date": "2026-06-16",
  "calendar_ids": ["nyse-2026", "cayman-2026"]
}
```

**Response:**
```jsonc
{
  "results": [
    {
      "date": "2026-07-01",
      "document_deadline": {"date": "2026-06-24", "cutoff_hour": 17, "cutoff_timezone": "New York"},
      "cash_funding_deadline": "2026-06-29",
      "citation": {"dealingBasis": "periodic", "dealingInterval": {"count": 1, "unit": "month"}, "dealingDay": {"anchor": "first", "dayType": "business"}}
    }
  ]
}
```

### `getNextTransactionDates()` output signature

```jsonc
{
  "results": [
    {
      "date": "date",
      "notice_deadline": "{ date: date, cutoff_hour: int, cutoff_timezone: string } | null",
      "settlement_date": "date | null",
      "document_deadline": "{ date: date, cutoff_hour: int, cutoff_timezone: string } | null",
      "cash_funding_deadline": "date | null",
      "citation": "object"
    }
  ]
}
```

Fields present depend on `transactionType`: redemption returns `notice_deadline` + `settlement_date`, subscription returns `document_deadline` + `cash_funding_deadline`. `date` is the trade/dealing date. `citation` contains the terms fields used to compute this result.

---

## Liquidity Dates API: `isValidTransactionDate()`

**Route:** `POST /liquidity-dates/isValidTransactionDate`

**Purpose:** "Is this a valid dealing date for this instrument?"

Given an instrument's terms, a date, and a date type, returns whether the date is a scheduled dealing date and a business day in all relevant centres.

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `instrument_id` | string | yes | The instrument to validate against |
| `transactionType` | string | yes | `redemption` or `subscription` |
| `subscriptionTerms` or `redemptionTerms` | object | yes | The full terms block for the requested side |
| `date` | date | yes | The date to validate |
| `date_type` | string | yes | What kind of date this is: `trade_date`, `settlement_date`, or `notice_deadline`. `as_of` is not valid here. |
| `calendar_ids` | string[] | yes | IDs of the holiday calendars to use |

### What LCS returns

```jsonc
{
  "instrument_id": "C.444",
  "date": "2026-10-01",
  "date_type": "trade_date",
  "transactionType": "redemption",
  "valid": true
}
```

### `isValidTransactionDate()` output signature

```jsonc
{
  "instrument_id": "string",
  "date": "date",
  "date_type": "trade_date | settlement_date | notice_deadline",
  "transactionType": "redemption | subscription",
  "valid": "boolean"
}
```

---

## Liquidity Dates API: `getProposedTransaction()`

**Route:** `POST /liquidity-dates/getProposedTransaction`

**Purpose:** "How do I redeem $X from this position?"

Same as `getNextTransactionDates()`, but the caller also provides the amount, position size, redemption constraints (lockups, gates, holdbacks), and previous transaction history. LCS evaluates the constraints, splits into tranches if needed, and returns a full schedule.

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `instrument_id` | string | yes | The instrument to plan redemption for |
| `transactionType` | string | yes | `redemption` (or `subscription` for buy-side planning) |
| `redemption_amount` | float | yes | How much the investor wants to redeem |
| `position_nav` | float | yes | Current position value |
| `lockup_start_date` | date | conditional | When the investor subscribed. Required if the fund has a lockup. |
| `previous_transactions` | array | no | Previous redemption history for this investor + instrument. Used to determine which gate number the investor is on. See format below. |
| `restrictions` | object | no | The full `restrictions` block from the liquidity terms JSON. Omit if no restrictions. |
| `gates` | object[] | no | The full `gates` array from the liquidity terms JSON. Omit if no gates. |
| `redemptionTerms` | object | yes | The full `redemptionTerms` block. |
| `date` | date | no | The reference date. Default: today. |
| `calendar_ids` | string[] | yes | IDs of the holiday calendars to use. |

**`previous_transactions` format:**

```jsonc
[
  {"trade_date": "2026-04-01", "amount": 2000000}
]
```

Each entry is a past redemption for this investor on this instrument. LCS uses this to determine which gate number the investor is on. Only one redemption is allowed per gate period (e.g. per quarter). The number of previous transactions tells LCS which gate applies next — for example, if the fund has a tiered gate schedule (25% → 33.3% → 50% → 100%) and the investor has one previous transaction, they're on gate 2 and can redeem up to 33.3% of their current NAV.

### Example — Redeem $5M from a position (with prior transaction)

Fund B. Subscribed Jan 15, 2025. 12-month hard lockup. Tiered gate: 25% → 33.3% → 50% → 100% of holding per quarter. 5% holdback on ≥95%. The investor already redeemed 25% ($2M) in the previous quarter (gate 1 done). Current NAV is $6M.

**Request:**
```jsonc
POST /liquidity-dates/getProposedTransaction

{
  "instrument_id": "C.444",
  "transactionType": "redemption",
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
  "date": "2026-06-16",
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
    {"tranche_number": 1, "amount": 2000000, "date": "2026-10-01", "notice_deadline": {"date": "2026-09-01", "cutoff_hour": 17, "cutoff_timezone": "New York"}, "settlement_date": "2026-10-30", "gate_number": 2, "gate_pct": 33.3, "gate_full": true, "holdback_amount": 0, "early_exit_fee": 0, "citation": {}},
    {"tranche_number": 2, "amount": 2000000, "date": "2027-01-04", "notice_deadline": {"date": "2026-12-04", "cutoff_hour": 17, "cutoff_timezone": "New York"}, "settlement_date": "2027-02-03", "gate_number": 3, "gate_pct": 50, "gate_full": true, "holdback_amount": 0, "early_exit_fee": 0, "citation": {}},
    {"tranche_number": 3, "amount": 1000000, "date": "2027-04-01", "notice_deadline": {"date": "2027-03-02", "cutoff_hour": 17, "cutoff_timezone": "New York"}, "settlement_date": "2027-05-04", "gate_number": 4, "gate_pct": 100, "gate_full": false, "holdback_amount": 0, "early_exit_fee": 0, "citation": {}}
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
    "lockup": {"active": "boolean", "lockup_type": "string | null", "expiry_date": "date | null", "anchor_shifted": "boolean", "early_exit_fee_pct": "float | null"},
    "gate": {"active": "boolean", "gate_level": "string | null", "current_gate_number": "int | null", "current_gate_pct": "float | null", "measurement_period": "string | null"},
    "holdback": {"active": "boolean", "threshold_pct": "float | null", "holdback_pct": "float | null", "triggered": "boolean"}
  },
  "tranches": [
    {
      "tranche_number": "int",
      "amount": "float",
      "date": "date",
      "notice_deadline": "{ date: date, cutoff_hour: int, cutoff_timezone: string } | null",
      "settlement_date": "date | null",
      "gate_number": "int | null",
      "gate_pct": "float | null",
      "gate_full": "boolean",
      "holdback_amount": "float",
      "early_exit_fee": "float",
      "citation": "object"
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

Unlike the Liquidity Dates API which computes on the fly, this reads from the stored Forward Calendar. The calendar is pre-computed and updated when holidays or terms change (see `updateInstrumentCalendar()`).

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `instrument_id` | string | yes | The instrument (in the URL path) |
| `transactionType` | string | no | `subscription`, `redemption`, or both (default) |
| `tenant_id` | string | yes | Which tenant's calendar (different tenants may have different holiday overlays) |
| `count` | int | no | Number of trade dates to return (e.g. "next 10 trade dates"). Mutually exclusive with `from`/`to`. |
| `from` | date | no | Start of range. Default: today. Ignored if `count` is provided. |
| `to` | date | no | End of range. Default: end of holiday calendar data. Ignored if `count` is provided. |

No liquidity terms or holidays needed — the data is already stored in the calendar.

### Example — "Give me the next 4 quarterly redemption dates"

**Request:** `GET /forward-calendar/getInstrumentCalendar/C.444?tenant_id=client-acme&transactionType=redemption&count=4`

**Response:**
```jsonc
{
  "instrument_id": "C.444",
  "tenant_id": "client-acme",
  "rows": [
    {"transactionType": "redemption", "date": "2026-10-01", "notice_deadline": {"date": "2026-09-01", "cutoff_hour": 17, "cutoff_timezone": "New York"}, "settlement_date": "2026-10-30"},
    {"transactionType": "redemption", "date": "2027-01-04", "notice_deadline": {"date": "2026-12-04", "cutoff_hour": 17, "cutoff_timezone": "New York"}, "settlement_date": "2027-02-03"},
    {"transactionType": "redemption", "date": "2027-04-01", "notice_deadline": {"date": "2027-03-02", "cutoff_hour": 17, "cutoff_timezone": "New York"}, "settlement_date": "2027-05-04"},
    {"transactionType": "redemption", "date": "2027-07-01", "notice_deadline": {"date": "2027-06-01", "cutoff_hour": 17, "cutoff_timezone": "New York"}, "settlement_date": "2027-07-31"}
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
    {"transactionType": "subscription", "date": "2026-10-01", "document_deadline": "2026-09-25", "cash_funding_deadline": "2026-09-28"},
    {"transactionType": "redemption",   "date": "2026-10-01", "notice_deadline": {"date": "2026-09-01", "cutoff_hour": 17, "cutoff_timezone": "New York"}, "settlement_date": "2026-10-30"},
    {"transactionType": "subscription", "date": "2026-11-02", "document_deadline": "2026-10-27", "cash_funding_deadline": "2026-10-29"},
    {"transactionType": "subscription", "date": "2026-12-01", "document_deadline": "2026-11-25", "cash_funding_deadline": "2026-11-27"},
    {"transactionType": "subscription", "date": "2027-01-04", "document_deadline": "2026-12-29", "cash_funding_deadline": "2026-12-31"},
    {"transactionType": "redemption",   "date": "2027-01-04", "notice_deadline": {"date": "2026-12-04", "cutoff_hour": 17, "cutoff_timezone": "New York"}, "settlement_date": "2027-02-03"}
  ]
}
```

### `getInstrumentCalendar()` output signature

```jsonc
{
  "instrument_id": "string",
  "tenant_id": "string",
  "range": {"from": "date", "to": "date"},
  "rows": [
    {
      "transactionType": "redemption | subscription",
      "date": "date",
      "notice_deadline": "{ date: date, cutoff_hour: int, cutoff_timezone: string } | null",
      "settlement_date": "date | null",
      "document_deadline": "{ date: date, cutoff_hour: int, cutoff_timezone: string } | null",
      "cash_funding_deadline": "date | null"
    }
  ]
}
```

### Why this exists separately from getNextTransactionDates

`getNextTransactionDates` computes on every call — the caller sends terms each time. That's fine for one-off queries but bad for systems that need the full calendar for reporting, LP exports, or dashboards. `getInstrumentCalendar` serves pre-built data — fast reads, no computation, no need to send terms.

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
| `transactionType` | string | no | `subscription`, `redemption`, or both (default) |
| `subscriptionTerms` | object | conditional | The full `subscriptionTerms` block. Required if `transactionType` includes subscription. |
| `redemptionTerms` | object | conditional | The full `redemptionTerms` block. Required if `transactionType` includes redemption. |
| `calendar_ids` | string[] | yes | IDs of the holiday calendars to use (from `getHolidayCalendars()`). |

Same inputs as `getNextTransactionDates()` — no `date`, no `date_type`, no `count`. The engine starts from today and runs until the holiday data ends.

### Example — Rebuild after a Hong Kong holiday update

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
  "dates_generated_to": "date"
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
| `missing_required_terms` | 422 | Required fields missing from terms (e.g. no `dealingBasis`) | `getNextTransactionDates`, `getProposedTransaction`, `isValidTransactionDate` |
| `unschedulable_trade` | 422 | `dealingBasis` is `discretionary` or `complex` — engine can't generate dates | `getNextTransactionDates`, `getProposedTransaction`, `isValidTransactionDate` |
| `no_reachable_trade_date` | 422 | No trade date satisfies the constraint (e.g. target settlement too soon) | `getNextTransactionDates`, `getProposedTransaction` |
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
