# LCS Phase 1 — Interfaces & Signatures

## Who builds what, who uses what

| Module | Built by | Used by |
|---|---|---|
| `dealing_dates.py` | Ashutosh | Engine (Phase 2, Ashutosh), `updateInstrumentCalendar` (Phase 2, Mihir) |
| `business_days.py` | Ashutosh | `dealing_dates.py` (Ashutosh), `offsets.py` (Mihir), Engine (Phase 2, Ashutosh) |
| `loader.py` | Aarohi | `app.py` startup, passed to `resolver.py` |
| `resolver.py` | Aarohi | API endpoints (Phase 2, Aarohi + Mihir), Engine (Phase 2, Ashutosh) |
| `offsets.py` | Mihir | Engine (Phase 2, Ashutosh) |
| `store.py` | Mihir | `getInstrumentCalendar` (Phase 2, Mihir), `updateInstrumentCalendar` (Phase 2, Mihir) |
| `models.py` | Aarohi | All API endpoints (Phase 2), `store.py` (Mihir) |

---

## Interfaces

```python
# Ashutosh: dealing_dates.py
generate_dealing_dates(dealing_basis, dealing_interval, dealing_day, start_date, holiday_set) -> Iterator[date]
# Generates dealing dates forward from start_date. Always goes forward.
# For PERIODIC: uses interval + dealing_day to place dates within each period.
# For ANNIVERSARY: uses interval from start_date, ignores dealing_day.
# EXCHANGE_TRADED instruments don't have dealing terms — they trade every day the exchange is open.
# The API layer normalizes them before calling this:
#   dealing_basis = "PERIODIC"
#   dealing_interval = {"count": 1, "unit": "DAY"}
#   dealing_day = None
#   holiday_set = resolve(["XNYS"])  (exchange trading calendar)
# So this function never sees "EXCHANGE_TRADED" — it just sees PERIODIC daily.
# Raises UnschedulableDealingError for COMPLEX, AT_CLOSING, AT_MATURITY.

# Ashutosh: business_days.py
is_business_day(d, holiday_set) -> bool       # True if not Saturday, not Sunday, not in holiday_set
adjust(d, holiday_set) -> date                # Modified Following: roll forward, if crosses month boundary roll backward
next_business_day(d, holiday_set) -> date     # Next business day strictly after d
prev_business_day(d, holiday_set) -> date     # Previous business day strictly before d

# Aarohi: loader.py
load_calendars(financial_centres_csv, exchange_trading_csv) -> (holidays, registry)
# holidays: single dict, calendar_id -> set of holiday dates
#   Financial Centres keyed by UN_LOCODE (e.g. "USNYC", "KYGEC") — 5-letter codes
#   Exchange Trading keyed by MIC code (e.g. "XNYS", "XLON") — 4-letter codes starting with X
#   Both go in the same dict — the codes don't overlap (verify on load).
#   These keys must match calendarId in the instrument terms.
# registry: list of calendar info dicts for the getHolidayCalendars() API response
#   each dict has: calendar_id, centre, calendar_type ("FINANCIAL_CENTRE" | "EXCHANGE_TRADING"),
#   source, coverage_from, coverage_to
# Parses DD-MM-YYYY dates from CSVs.

# Aarohi: resolver.py
resolve(calendar_ids, holidays) -> set[date]
# Takes a list of calendar IDs (e.g. ["USNYC", "KYGEC"] or ["XNYS"]) and the holidays dict from load_calendars().
# Returns the union of all holiday dates across those calendars.
# The API layer extracts calendarId strings from the instrument terms CalendarRef objects
# ({"calendarType": "FINANCIAL_CENTRE", "calendarId": "USNYC", ...}) before calling this.
# The resolver doesn't need to know about calendarType — it just looks up by ID.
# Raises KeyError if a calendar_id is not found.

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
    dict[str, set[date]],        # holidays: single dict with both Financial Centre and Exchange Trading calendars
                                 #   Financial Centres keyed by UN_LOCODE: "USNYC", "KYGEC", "GBLNB", etc.
                                 #   Exchange Trading keyed by MIC code: "XNYS", "XLON", etc.
                                 #   Both in the same dict — codes don't overlap (5-letter LOCODE vs 4-letter MIC)
                                 #   Must match calendarId in instrument terms
                                 #   Verify no key collisions on load
    list[dict],                  # registry: one dict per calendar, each has:
                                 #   {"calendar_id": str, "centre": str,
                                 #    "calendar_type": "FINANCIAL_CENTRE" | "EXCHANGE_TRADING",
                                 #    "source": str, "coverage_from": date, "coverage_to": date}
]
```

### Aarohi: `resolver.py`

```python
resolve(
    calendar_ids: list[str],             # e.g. ["USNYC", "KYGEC"] or ["XNYS"]
                                         # API layer extracts these from CalendarRef objects in the terms
                                         # resolver doesn't need calendarType — just looks up by ID
    holidays: dict[str, set[date]],      # the single dict returned by load_calendars() (both LOCODE + MIC keys)
) -> set[date]                           # union of all holidays across the given calendars
```

### Aarohi: `models.py`

```python
# Pydantic request/response models for all 5 API methods
# CalendarRow dataclass (shared with Mihir's store.py)
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

Defined by Mihir in `store.py` or `models.py`. Produced by the Engine (Phase 2, Ashutosh). Stored by `save_calendar()` (Mihir). Returned by `get_calendar()` (Mihir).
