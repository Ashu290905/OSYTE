# LCS Phase 1 — Interfaces & Signatures

## Who builds what, who uses what

### Ashutosh

| Builds | Uses |
|---|---|
| `dealing_dates.py` — generates dealing dates for MANAGER_TRADED and EXCHANGE_TRADED | `instrument.py` (Mihir) — reads dealing terms and holidays from the Instrument object |
| `business_days.py` — is_business_day, adjust, next/prev business day | Nothing external — standalone utility |

### Aarohi

| Builds | Uses |
|---|---|
| `loader.py` — parses Copp Clark CSVs into holiday dicts + registry | Nothing external — standalone, runs on startup |
| `resolver.py` — merges holidays from CalendarRef objects into one set | `loader.py` (Aarohi) — reads fc_holidays and et_holidays |
| `models.py` — Pydantic request/response models, error models | `instrument.py` (Mihir) — references Instrument and CalendarRow types |

### Mihir

| Builds | Uses |
|---|---|
| `instrument.py` — the shared Instrument class | Nothing external — class definition |
| `term_loader.py` — parses raw JSON into an Instrument object | `resolver.py` (Aarohi) — calls resolve() to populate instrument.holidays |
| `offsets.py` — counts N days forward/backward | `business_days.py` (Ashutosh) — calls is_business_day() for BUSINESS day counting |
| `store.py` — SQLite calendar store (save + query) | `instrument.py` (Mihir) — reads CalendarRow type |

---

## The Instrument object

Instead of passing 5-6 individual fields to every function, we pass one `Instrument` object. Each function reads what it needs from it.

The raw instrument terms JSON comes in from the API request. Mihir's `term_loader.py` parses it into an `Instrument` object and resolves the holidays onto it. From that point on, everything just takes `instrument`.

**Flow:**

```
Raw JSON from API request
    → term_loader.load(raw_json, fc_holidays, et_holidays)
        → parses dealingModel, extracts sub-terms
        → calls resolve() to get the holiday set
        → returns Instrument object with holidays attached
    → passed to generate_dealing_dates(instrument, ...) / engine / etc.
```

**Why:** Functions don't need to know what fields exist on other sides. `generate_dealing_dates` reads `instrument.redemption.dealing_basis` — it doesn't care about gates or restrictions. When we add `getProposedTransaction` later, the instrument already carries gates and restrictions — no signature changes needed.

**About the holiday set on the instrument:** `instrument.holidays` is a Python reference to a `set[date]`, not a copy. The set is created once by `resolve()` during loading. Every function that reads `instrument.holidays` accesses the same object in memory. No data is copied when passing the instrument around.

---

## Interfaces

```python
# Ashutosh: dealing_dates.py

generate_dealing_dates(instrument, start_date, side) -> Iterator[date]
# For MANAGER_TRADED instruments.
# Reads dealing_basis, dealing_interval, dealing_day from instrument.subscription or instrument.redemption
# based on the side param.
# For PERIODIC: uses interval + dealing_day to place dates within each period.
# For ANNIVERSARY: uses interval from start_date, ignores dealing_day.
# Uses instrument.holidays for BUSINESS dayType.
# Raises UnschedulableDealingError for COMPLEX, AT_CLOSING, AT_MATURITY.

generate_exchange_dealing_dates(instrument, start_date) -> Iterator[date]
# For EXCHANGE_TRADED instruments.
# Yields every business day (not weekend, not in instrument.holidays) forward from start_date.
# No dealing basis, interval, or dealing day — every open day is a dealing date.

# Ashutosh: business_days.py

is_business_day(d, holiday_set) -> bool       # True if not Saturday, not Sunday, not in holiday_set
adjust(d, holiday_set) -> date                # Modified Following: roll forward, if crosses month boundary roll backward
next_business_day(d, holiday_set) -> date     # Next business day strictly after d
prev_business_day(d, holiday_set) -> date     # Previous business day strictly before d
# These still take holiday_set directly (not instrument) because they're low-level utilities
# used by offsets.py and internally. The caller passes instrument.holidays.

# Aarohi: loader.py

load_calendars(financial_centres_csv, exchange_trading_csv) -> (fc_holidays, et_holidays, registry)
# fc_holidays: Financial Centre holidays keyed by UN_LOCODE (e.g. {"USNYC": {date, ...}, "KYGEC": {date, ...}})
# et_holidays: Exchange Trading holidays keyed by MIC code (e.g. {"XNYS": {date, ...}, "XLON": {date, ...}})
# registry: list of calendar info dicts for getHolidayCalendars() API response
# Two separate dicts because calendarType determines which one to look in.
# Parses DD-MM-YYYY dates from CSVs. Called once on app startup.

# Aarohi: resolver.py

resolve(calendar_refs, fc_holidays, et_holidays) -> set[date]
# Takes CalendarRef objects from the instrument terms.
# Each ref: {"calendarType": "FINANCIAL_CENTRE" | "EXCHANGE_TRADING", "calendarId": "USNYC", "source": "COPP_CLARK"}
# Routes to fc_holidays or et_holidays based on calendarType.
# Returns the union of all holiday dates.
# Called by term_loader.load() — the result is stored on instrument.holidays.
# Call site: Mihir merges fc_holidays and et_holidays into a single dict in app.py startup
# before passing to resolve(). Ashutosh receives the already-merged dict.
# resolve() itself expects a single merged dict.
# Raises KeyError if a calendarId is not found.

# Mihir: term_loader.py

load(raw_json, fc_holidays, et_holidays) -> Instrument
# Parses the raw instrument terms JSON into an Instrument object.
# Discriminates on dealingModel:
#   MANAGER_TRADED → extracts subscriptionTerms, redemptionTerms, gates, restrictions, dealingCalendar
#   EXCHANGE_TRADED → extracts tradingCalendar, settlementCalendar, settlement
#   DRAWDOWN → raises UnsupportedDealingModelError
# Calls resolve() with the calendar refs + fc/et holidays to get the holiday set.
# Attaches the result as instrument.holidays.
# Returns a fully populated Instrument object ready to use.

# Mihir: offsets.py

count_days(start, days, direction, day_type, holiday_set) -> date
# Counts N days forward (AFTER) or backward (BEFORE) from start.
# CALENDAR: counts all days including weekends and holidays.
# BUSINESS: skips weekends and holidays (uses is_business_day()).
# Does NOT apply roll convention — caller does that via adjust() if needed.
# The caller passes instrument.holidays as the holiday_set.

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

### Mihir: `instrument.py`

```python
@dataclass
class ManagerTradedSideTerms:
    dealing_basis: str                     # "PERIODIC" | "ANNIVERSARY" | "COMPLEX" | "AT_CLOSING" | "AT_MATURITY"
    dealing_interval: dict | None          # {"count": 3, "unit": "MONTH"} or None
    dealing_day: dict | None               # {"anchor": "FIRST", "dayType": "BUSINESS", ...} or None
    notice_period: dict | None             # redemption only — {"days": 30, "dayType": "CALENDAR", ...}
    settlement: dict | None                # redemption only
    document_deadline: dict | None         # subscription only
    cash_funding_deadline: dict | None     # subscription only
    redemption_schedule: dict | None       # complex redemption only

@dataclass
class ExchangeTradedTerms:
    trading_calendar: dict                 # {"calendarType": "EXCHANGE_TRADING", "calendarId": "XNYS", ...}
    settlement_calendar: dict              # same shape
    settlement: dict                       # {"days": 1, "direction": "AFTER", ...}

@dataclass
class Instrument:
    term_id: str                           # e.g. "TERM.C.10264"
    dealing_model: str                     # "MANAGER_TRADED" | "EXCHANGE_TRADED"
    label: str                             # human-readable name
    calendar_refs: list[dict]              # CalendarRef objects from the terms
    holidays: set[date]                    # resolved holiday set — populated by term_loader.load()

    # MANAGER_TRADED fields (None if EXCHANGE_TRADED)
    subscription: ManagerTradedSideTerms | None
    redemption: ManagerTradedSideTerms | None
    gates: list[dict] | None
    restrictions: dict | None

    # EXCHANGE_TRADED fields (None if MANAGER_TRADED)
    exchange_traded: ExchangeTradedTerms | None
```

### Mihir: `term_loader.py`

```python
load(
    raw_json: dict,                          # the full instrument terms object from the API request
    fc_holidays: dict[str, set[date]],       # Financial Centre holidays from load_calendars()
    et_holidays: dict[str, set[date]],       # Exchange Trading holidays from load_calendars()
) -> Instrument
# Raises UnsupportedDealingModelError for DRAWDOWN
```

### Ashutosh: `dealing_dates.py`

```python
generate_dealing_dates(
    instrument: Instrument,      # reads instrument.subscription or instrument.redemption based on side
    start_date: date,            # generate from this date forward
    side: str,                   # "subscription" | "redemption"
) -> Iterator[date]
# Uses instrument.holidays for BUSINESS dayType
# Raises UnschedulableDealingError for COMPLEX, AT_CLOSING, AT_MATURITY

generate_exchange_dealing_dates(
    instrument: Instrument,      # reads instrument.holidays (exchange trading holidays)
    start_date: date,            # generate from this date forward
) -> Iterator[date]
# Yields every day that is not a weekend and not in instrument.holidays
```

### Ashutosh: `business_days.py`

```python
is_business_day(d: date, holiday_set: set[date]) -> bool
adjust(d: date, holiday_set: set[date]) -> date
next_business_day(d: date, holiday_set: set[date]) -> date
prev_business_day(d: date, holiday_set: set[date]) -> date
# Low-level utilities. Caller passes instrument.holidays as holiday_set.
```

### Aarohi: `loader.py`

```python
load_calendars(
    financial_centres_csv: str,
    exchange_trading_csv: str,
) -> tuple[
    dict[str, set[date]],        # fc_holidays: Financial Centre holidays keyed by UN_LOCODE
    dict[str, set[date]],        # et_holidays: Exchange Trading holidays keyed by MIC code
    list[dict],                  # registry: one dict per calendar
                                 #   {"calendar_id": str, "centre": str,
                                 #    "calendar_type": "FINANCIAL_CENTRE" | "EXCHANGE_TRADING",
                                 #    "source": str, "coverage_from": date, "coverage_to": date}
]
```

### Aarohi: `resolver.py`

```python
resolve(
    calendar_refs: list[dict],           # CalendarRef objects from instrument terms
                                         #   {"calendarType": "FINANCIAL_CENTRE", "calendarId": "USNYC", "source": "COPP_CLARK"}
    fc_holidays: dict[str, set[date]],   # Financial Centre holidays from load_calendars()
    et_holidays: dict[str, set[date]],   # Exchange Trading holidays from load_calendars()
) -> set[date]                           # union of all holidays across the given refs
# Raises KeyError with the bad calendar ID in the message.
# All callers (endpoint layer and engine) must catch KeyError and return HTTP 422 — do not let it propagate as a 500.
```

### Mihir: `offsets.py`

```python
count_days(
    start: date,
    days: int,                   # positive integer
    direction: str,              # "BEFORE" | "AFTER"
    day_type: str,               # "CALENDAR" | "BUSINESS"
    holiday_set: set[date],      # caller passes instrument.holidays
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
    transaction_type: str | None,
    count: int | None,
    from_date: date | None,
    to_date: date | None,
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

Defined by Mihir in `models.py`. `store.py` must import it from there — do not redefine it. Produced by the Engine (Phase 2, Ashutosh). Stored by `save_calendar()` (Mihir). Returned by `get_calendar()` (Mihir).
