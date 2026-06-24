# LCS Phase 2 — Interfaces & Signatures

## Who builds what, who uses what

### Ashutosh

| Builds | Uses |
|---|---|
| `engine.py` — the find-check-skip loop, produces CalendarRows | `dealing_dates.py` (Ashutosh) — generates candidate trade dates |
| | `offsets.py` (Mihir) — computes notice, settlement, document deadlines |
| | `instrument.py` (Mihir) — reads side terms and holidays from the Instrument |

### Aarohi

| Builds | Uses |
|---|---|
| `getHolidayCalendars` endpoint | Holiday store registry loaded at startup |
| `getNextTransactionDates` endpoint | `engine.get_next_transaction_dates()` (Ashutosh) |
| `isValidTransactionDate` endpoint | `engine.get_next_transaction_dates()` (Ashutosh) |

### Mihir

| Builds | Uses |
|---|---|
| `updateInstrumentCalendar` endpoint | `engine.get_next_transaction_dates()` (Ashutosh), `store.save_calendar()` (Mihir) |

`getInstrumentCalendar` is already implemented — no changes needed.

---

## The engine

The engine is one function. It runs the find-check-skip loop and returns fully-chained CalendarRows. Everything else in Phase 2 calls this function — either directly (Aarohi's endpoints) or in a loop (Mihir's updateInstrumentCalendar).

**The loop:**

```
for trade_date in generate_dealing_dates(instrument, start_date=as_of, side):
    row = _compute_chain(trade_date, side_terms, holidays)
    if row.notice_deadline_date is None or row.notice_deadline_date >= as_of:
        keep    # notice window still open
    else:
        skip    # notice already passed — advance to next dealing date
```

The loop always terminates: for any recurring dealing schedule with a finite notice period,
there is always a future trade date far enough ahead that its notice deadline >= as_of.

---

## Interfaces

```python
# Ashutosh: engine.py

get_next_transaction_dates(
    instrument: Instrument,
    side: str,          # "subscription" | "redemption"
    as_of: date,
    count: int = 1,
) -> list[CalendarRow]
# Runs the find-check-skip loop. Returns `count` CalendarRows where
# notice_deadline_date is None or >= as_of.
# Raises UnschedulableDealingError if terms have no deterministic cadence.

# Aarohi: getHolidayCalendars

GET /liquidity-dates/getHolidayCalendars?tenant_id=<optional>
# Returns base calendars from the in-memory registry.
# If tenant_id is provided, appends that tenant's overlay calendars.
# Response: GetHolidayCalendarsResponse

# Aarohi: getNextTransactionDates

POST /liquidity-dates/getNextTransactionDates
# body: GetNextTransactionDatesRequest
# 1. term_loader.load(body terms) → Instrument
# 2. engine.get_next_transaction_dates(instrument, side, as_of, count) → list[CalendarRow]
# 3. Map CalendarRow → TransactionDateResult (add citation from side_terms)
# body.date defaults to date.today() if None.
# body.date_type must be "as_of" — return 400 for any other value.
# Response: GetNextTransactionDatesResponse

# Aarohi: isValidTransactionDate

POST /liquidity-dates/isValidTransactionDate
# body: IsValidTransactionDateRequest
# Converts query_date to a dealing_date, runs engine, cross-checks.
# date_type="as_of" → 400 immediately.
#
# date_type="trade_date":
#   rows = engine.get_next_transaction_dates(instrument, side, as_of=body.date, count=1)
#   valid = rows[0].date == body.date
#
# date_type="settlement_date":
#   as_of = body.date - timedelta(days=90)
#   scan engine rows until row.settlement_date == body.date → valid = True
#                    or row.date > body.date               → valid = False
#
# date_type="notice_deadline":
#   rows = engine.get_next_transaction_dates(instrument, side, as_of=body.date, count=1)
#   valid = rows[0].notice_deadline_date == body.date
#
# Response: IsValidTransactionDateResponse

# Mihir: updateInstrumentCalendar

POST /forward-calendar/updateInstrumentCalendar
# body: UpdateInstrumentCalendarRequest
# Calls engine in a loop from today to 2056-01-01, saves to store.
#
# sides = ["subscription", "redemption"] if body.transactionType is None
#         else [body.transactionType]
#
# for side in sides:
#     rows = []
#     as_of = date.today()
#     while as_of < date(2056, 1, 1):
#         batch = engine.get_next_transaction_dates(instrument, side, as_of, count=1)
#         rows.append(batch[0])
#         as_of = batch[0].date + timedelta(days=1)
#     save_calendar(body.instrument_id, body.tenant_id, rows)
#
# Response: UpdateInstrumentCalendarResponse
```

---

## Signatures

### Ashutosh: `engine.py`

```python
from datetime import date
from lcs.terms_loader.instrument import Instrument, ManagerTradedSideTerms
from lcs.models import CalendarRow

def get_next_transaction_dates(
    instrument: Instrument,
    side: str,
    as_of: date,
    count: int = 1,
) -> list[CalendarRow]:
    ...

# Internal — not exported, but shared understanding across the team:
def _compute_chain(
    trade_date: date,
    side_terms: ManagerTradedSideTerms,
    holidays: set[date],
) -> CalendarRow:
    # Calls count_days() from offsets.py for each offset field on side_terms.
    # Redemption:    notice_deadline, settlement_date
    # Subscription:  document_deadline, cash_funding_deadline
    ...
```

### Ashutosh: exceptions

```python
# Already exists in dealing_dates.py — imported and re-raised by engine
class UnschedulableDealingError(Exception):
    """Raised when dealing terms do not define a deterministic cadence."""
```

### Aarohi: `getHolidayCalendars`

```python
# liquidity_dates_api/get_holiday_calendars.py

@router.get("/getHolidayCalendars", response_model=GetHolidayCalendarsResponse)
def get_holiday_calendars(
    tenant_id: Optional[str] = Query(None),
) -> GetHolidayCalendarsResponse:
    ...
```

### Aarohi: `getNextTransactionDates`

```python
# liquidity_dates_api/get_next_transaction_dates.py

@router.post("/getNextTransactionDates", response_model=GetNextTransactionDatesResponse)
def get_next_transaction_dates(
    body: GetNextTransactionDatesRequest,
) -> GetNextTransactionDatesResponse:
    ...
```

### Aarohi: `isValidTransactionDate`

```python
# liquidity_dates_api/is_valid_transaction_date.py

@router.post("/isValidTransactionDate", response_model=IsValidTransactionDateResponse)
def is_valid_transaction_date(
    body: IsValidTransactionDateRequest,
) -> IsValidTransactionDateResponse:
    ...
```

### Mihir: `updateInstrumentCalendar`

```python
# forward_calendar_api/update_instrument_calendar.py

@router.post("/updateInstrumentCalendar", response_model=UpdateInstrumentCalendarResponse)
def update_instrument_calendar(
    body: UpdateInstrumentCalendarRequest,
) -> UpdateInstrumentCalendarResponse:
    ...
```

---

## Error → HTTP mapping (all endpoints)

| Cause | `error` code | HTTP |
|---|---|---|
| `UnschedulableDealingError` from engine | `unschedulable_trade` | 422 |
| `terms is None` (side not present on instrument) | `missing_required_terms` | 422 |
| `KeyError` from resolver (unknown calendar ID) | `unknown_calendar_id` | 422 |
| `date_type="as_of"` on isValidTransactionDate | `invalid_date_type` | 400 |

---

## Dependency order days 5–7

```
Day 5:  Ashutosh: engine.py
             ↓
        unblocks Aarohi (getNextTransactionDates, isValidTransactionDate)
        unblocks Mihir  (updateInstrumentCalendar)

Day 6:  Aarohi: getNextTransactionDates + getHolidayCalendars
        Mihir:  updateInstrumentCalendar

Day 7:  Aarohi: isValidTransactionDate
        Mihir:  full integration — update → get round-trip, full flow tests
```
