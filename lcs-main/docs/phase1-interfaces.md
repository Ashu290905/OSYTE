# LCS Phase 1 — Interfaces & Signatures

## Who builds what, who uses what

| Module | Built by | Used by |
|---|---|---|
| `dealing_dates.py` | Ashutosh | Engine (Phase 2, Ashutosh), `updateInstrumentCalendar` (Phase 2, Mihir) |
| `business_days.py` | Ashutosh | `dealing_dates.py` (Ashutosh), `offsets.py` (Mihir), Engine (Phase 2, Ashutosh) |
| `loader.py` | Aarohi | `app.py` startup, passed to `resolver.py` |
| `resolver.py` | Aarohi | API endpoints (Phase 2, Aarohi + Mihir), Engine (Phase 2, Ashutosh) |
| `models.py` | Aarohi | All API endpoints, `store.py` (Mihir), Engine (Ashutosh) — includes TermRecord parsing |
| `offsets.py` | Mihir | Engine (Phase 2, Ashutosh) |
| `store.py` | Mihir | `getInstrumentCalendar` (Phase 2, Mihir), `updateInstrumentCalendar` (Phase 2, Mihir) |

---

## Interfaces

```python
# Ashutosh: dealing_dates.py

generate_dealing_dates(dealing_basis, dealing_interval, dealing_day, start_date, holiday_set) -> Iterator[date]
# For MANAGER_TRADED instruments.
# Generates dealing dates forward from start_date.
# For PERIODIC: uses interval + dealing_day to place dates within each period.
# For ANNIVERSARY: uses interval from start_date, ignores dealing_day.
# Raises UnschedulableDealingError for COMPLEX, AT_CLOSING, AT_MATURITY.

generate_exchange_dealing_dates(exchange_holidays, start_date) -> Iterator[date]
# For EXCHANGE_TRADED instruments.
# Yields every business day (not weekend, not in exchange_holidays) forward from start_date.
# No dealing basis, interval, or dealing day — every open day is a dealing date.

# Ashutosh: business_days.py

is_business_day(d, holiday_set) -> bool       # True if not Saturday, not Sunday, not in holiday_set
adjust(d, holiday_set) -> date                # Modified Following: roll forward, if crosses month boundary roll backward
next_business_day(d, holiday_set) -> date     # Next business day strictly after d
prev_business_day(d, holiday_set) -> date     # Previous business day strictly before d

# Aarohi: loader.py

load_calendars(financial_centres_csv, exchange_trading_csv) -> (fc_holidays, et_holidays, registry)
# fc_holidays: Financial Centre holidays keyed by UN_LOCODE (e.g. {"USNYC": {date, ...}, "KYGEC": {date, ...}})
# et_holidays: Exchange Trading holidays keyed by MIC code (e.g. {"XNYS": {date, ...}, "XLON": {date, ...}})
# registry: list of calendar info dicts for the getHolidayCalendars() API response
# Two separate dicts because calendarType determines which one to look in.
# Parses DD-MM-YYYY dates from CSVs.

# Aarohi: resolver.py

resolve(calendar_refs, fc_holidays, et_holidays) -> set[date]
# Takes CalendarRef objects directly from the instrument terms.
# Each ref has: {"calendarType": "FINANCIAL_CENTRE" | "EXCHANGE_TRADING", "calendarId": "USNYC", "source": "COPP_CLARK"}
# Routes to fc_holidays or et_holidays based on calendarType.
# Returns the union of all holiday dates across the given refs.
# Raises KeyError if a calendarId is not found in the corresponding dict.

# Aarohi: models.py

# Pydantic models for:
# - TermRecord: parses the full instrument terms object, discriminates on dealingModel
#   (MANAGER_TRADED, EXCHANGE_TRADED, DRAWDOWN), extracts the right sub-terms.
#   This is where the raw API request gets parsed into the shapes that dealing_dates.py
#   and engine.py expect.
# - Request/response models for all 5 API methods
# - CalendarRow dataclass (shared with store.py)
# - Error response model
# Built in Phase 1 so everyone can use the parsed types from day 1.

# Mihir: offsets.py

count_days(start, days, direction, day_type, holiday_set) -> date
# Counts N days forward (AFTER) or backward (BEFORE) from start.
# CALENDAR: counts all days including weekends and holidays.
# BUSINESS: skips weekends and holidays (uses is_business_day()).
# Does NOT apply roll convention — caller does that via adjust() if needed.

# Mihir: store.py

save_calendar(instrument_id, tenant_id, rows) -> None
# Deletes all existing rows for this instrument_id + tenant_id, then inserts new rows.

get_calendar(instrument_id, tenant_id, transaction_type, count, from_date, to_date) -> list[CalendarRow]
# transaction_type: "redemption", "subscription", or None (both).
# count: return next N rows from today. Mutually exclusive with from/to.
# from_date/to_date: return rows within range. Ignored if count is set.
# Raises CalendarNotFoundError if no rows exist for this instrument + tenant.
```

---

## Signatures

### Ashutosh: `dealing_dates.py`

```python
generate_dealing_dates(
    dealing_basis: str,          # "PERIODIC" | "ANNIVERSARY"
    dealing_interval: dict,      # {"count": 3, "unit": "MONTH"} — count is int, unit is "DAY" | "WEEK" | "MONTH"
    dealing_day: dict | None,    # {"anchor": "FIRST", "dayType": "BUSINESS"} or None
                                 #   anchor: "FIRST" | "LAST" | "NTH" | "SPECIFIC_DATE"
                                 #   dayType: "BUSINESS" | "CALENDAR"
                                 #   ordinal: int — only when anchor="NTH" (e.g. 3 = 3rd day of period)
                                 #   specificDates: [{"month": 6, "day": 30}, ...] — only when anchor="SPECIFIC_DATE"
                                 # None for daily/weekly dealing and ANNIVERSARY
    start_date: date,            # generate from this date forward
    holiday_set: set[date],      # merged holidays from resolve() — used for BUSINESS dayType
) -> Iterator[date]
# Raises UnschedulableDealingError for COMPLEX, AT_CLOSING, AT_MATURITY

generate_exchange_dealing_dates(
    exchange_holidays: set[date], # holidays for this exchange (e.g. XNYS holidays from et_holidays)
    start_date: date,             # generate from this date forward
) -> Iterator[date]
# Yields every day that is not a weekend and not in exchange_holidays
```

### Ashutosh: `business_days.py`

```python
is_business_day(d: date, holiday_set: set[date]) -> bool
adjust(d: date, holiday_set: set[date]) -> date
next_business_day(d: date, holiday_set: set[date]) -> date
prev_business_day(d: date, holiday_set: set[date]) -> date
```

### Aarohi: `loader.py`

```python
load_calendars(
    financial_centres_csv: str,  # file path to copp_clark_FinancialCentres CSV
    exchange_trading_csv: str,   # file path to copp_clark_ExchangeTrading CSV
) -> tuple[
    dict[str, set[date]],        # fc_holidays: Financial Centre holidays
                                 #   keyed by UN_LOCODE: "USNYC", "KYGEC", "GBLNB", etc.
    dict[str, set[date]],        # et_holidays: Exchange Trading holidays
                                 #   keyed by MIC code: "XNYS", "XLON", etc.
    list[dict],                  # registry: one dict per calendar, each has:
                                 #   {"calendar_id": str, "centre": str,
                                 #    "calendar_type": "FINANCIAL_CENTRE" | "EXCHANGE_TRADING",
                                 #    "source": str, "coverage_from": date, "coverage_to": date}
]
```

### Aarohi: `resolver.py`

```python
resolve(
    calendar_refs: list[dict],           # CalendarRef objects from instrument terms, e.g.:
                                         #   [{"calendarType": "FINANCIAL_CENTRE", "calendarId": "USNYC", "source": "COPP_CLARK"},
                                         #    {"calendarType": "FINANCIAL_CENTRE", "calendarId": "KYGEC", "source": "COPP_CLARK"}]
                                         # or for exchange traded:
                                         #   [{"calendarType": "EXCHANGE_TRADING", "calendarId": "XNYS", "source": "COPP_CLARK"}]
    fc_holidays: dict[str, set[date]],   # Financial Centre holidays from load_calendars()
    et_holidays: dict[str, set[date]],   # Exchange Trading holidays from load_calendars()
) -> set[date]                           # union of all holidays across the given refs
# Routes each ref to fc_holidays or et_holidays based on calendarType
# Raises KeyError if a calendarId is not found in the corresponding dict
```

### Aarohi: `models.py`

```python
# TermRecord: parses the full instrument terms JSON
# Input shape from API request:
#   {"termId": "...", "dealingModel": "MANAGER_TRADED", "managerTraded": {...}}
#   {"termId": "...", "dealingModel": "EXCHANGE_TRADED", "exchangeTraded": {...}}
#   {"termId": "...", "dealingModel": "DRAWDOWN", "drawdown": {...}}
#
# TermRecord discriminates on dealingModel and extracts:
#   - For MANAGER_TRADED: subscriptionTerms, redemptionTerms, gates, restrictions, dealingCalendar
#   - For EXCHANGE_TRADED: tradingCalendar, settlementCalendar, settlement
#   - For DRAWDOWN: raises UnsupportedDealingModelError (out of scope)
#
# This parsing happens in Phase 1 so the engine and API endpoints can work with clean typed objects.

# Request/response Pydantic models for all 5 API methods
# CalendarRow dataclass (shared with store.py)
# Error response model
```

### Mihir: `offsets.py`

```python
count_days(
    start: date,
    days: int,                   # number of days to count (positive integer)
    direction: str,              # "BEFORE" | "AFTER"
    day_type: str,               # "CALENDAR" | "BUSINESS"
    holiday_set: set[date],      # merged holidays from resolve() — only used when day_type="BUSINESS"
) -> date
```

### Mihir: `store.py`

```python
save_calendar(
    instrument_id: str,
    tenant_id: str,
    rows: list[CalendarRow],
) -> None

get_calendar(
    instrument_id: str,
    tenant_id: str,
    transaction_type: str | None,  # "redemption" | "subscription" | None (both)
    count: int | None,             # next N rows from today — mutually exclusive with from/to
    from_date: date | None,        # start of range — ignored if count is set
    to_date: date | None,          # end of range — ignored if count is set
) -> list[CalendarRow]
```

### Shared: `CalendarRow` dataclass

```python
@dataclass
class CalendarRow:
    transaction_type: str                          # "redemption" | "subscription"
    date: date                                     # the trade/dealing date
    notice_deadline_date: date | None              # redemption only
    notice_deadline_cutoff_hour: int | None         # 0-23
    notice_deadline_cutoff_timezone: str | None     # e.g. "New York"
    settlement_date: date | None                   # redemption only
    document_deadline_date: date | None            # subscription only
    document_deadline_cutoff_hour: int | None       # 0-23
    document_deadline_cutoff_timezone: str | None
    cash_funding_deadline: date | None             # subscription only
```

Defined by Aarohi in `models.py`. Produced by the Engine (Phase 2, Ashutosh). Stored by `save_calendar()` (Mihir). Returned by `get_calendar()` (Mihir).
