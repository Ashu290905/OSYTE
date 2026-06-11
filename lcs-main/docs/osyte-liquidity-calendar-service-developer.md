# Liquidity Calendar Service — Developer Spec

> **Audience:** Engineering, QA, Design | **Author:** PM (Osyte) | **Last updated:** May 2026
> **Format:** New feature
> **Linked Client PRFAQ:** [Liquidity Calendar Service — Confluence](https://osyte.atlassian.net/wiki/spaces/ZOTBPB/pages/1309573122/Liquidity+Calendar+Service)
> **Linked Jira:** 
> **Build complexity (PM estimate):** (1–2 months)
> **Data dependencies:** External data provider — Copp Clark (default) + client-supplied holiday files; existing Instrument records extended with liquidity terms (Security Master)

---

## Context (one-line)

Enterprise multi-asset clients need consistent, auditable per-instrument key dates (notification, trade, valuation, settlement) computed against their chosen holiday data — today this is manual Excel work that produces inconsistent dates across Investment Planning, Rebalancing, and Trade Ops and risks missed redemption windows.

## Scope

The service ships in two release phases. **Phase 1** is the initial production release: the deterministic computation engine, both APIs, holiday ingestion, overlays, recompute pipeline, and a persistent forward calendar. **Phase 2** layers on effective-dated versioning (so the service can answer "what did it say on date X") and exposes the service through a natural-language query surface and an MCP server, both scoped to the requester's entitlements.

**In scope — Phase 1 (initial release):**
- **Compute API** — real-time, stateless, per-(instrument, anchor date) lookup returning the full set of derived key dates
- **Calendar API** — persisted, forward-looking per-instrument calendar of key dates, readable, subscribable, exportable
- **Persistent forward calendar** — materialized horizon of upcoming key dates per instrument, queryable without re-invoking the compute engine on each read
- **Shared computation engine** backing both APIs (must return identical dates for the same inputs)
- **Copp Clark integration** as the default holiday source
- **Client-supplied holiday file ingestion** in standard formats
- **Client-specific overlay** on a base vendor source (conflicts resolved in client's favor; surfaced in changelog)
- **Auto-recompute on holiday-data updates** + published changelog of affected instruments and moved dates

**In scope — Phase 2 (follow-on release):**
- **Effective-dated versioning** — supports both "what does the calendar say now" and "what did it say on date X"; immutable `CalendarVersion` snapshots stamped with effective-from date
- **NL query layer** over the service, scoped to the requester's entitled book
- **MCP server surface** exposing the same capabilities to AI assistants, scoped to the requester's entitled book

**Out of scope (v1, both phases):**
- AI-assisted extraction of liquidity terms from PPMs, side letters, term sheets
- Richer instrument-extraction tooling beyond standard Osyte instrument onboarding
- AI in the date-computation path (fully deterministic rules based)

---

## 1. Mockups


**#1 — Instrument Forward Calendar (viewer)**

```
+--------------------------------------------------------------------+
| Forward Calendar — ABC Global Macro Fund (Class I)                 |
| Holiday source: Copp Clark v2026.05 + Acme overlay (eff. 2026-05)  |
+--------------------------------------------------------------------+
|  Window: [ Next 12 months ▾ ]   [ Export CSV ] [ Subscribe ] [⟳]   |
+--------------------------------------------------------------------+
|  Dealing date  | Notif. deadline | Trade date | Val. date | Settle |
| --------------+------------------+------------+-----------+--------|
|  2026-05-29    | 2026-05-07 16:00 | 2026-05-29 | 2026-05-30| 06-04 |
|  2026-06-30    | 2026-06-09 16:00 | 2026-06-30 | 2026-07-01| 07-07 |
|  2026-07-31    | 2026-07-10 16:00 | 2026-07-31 | 2026-08-03| 08-06 |
|  ...                                                               |
+--------------------------------------------------------------------+
|  Last recomputed: 2026-05-16 04:12  |  Version: v17 (effective)    |
+--------------------------------------------------------------------+
```

**#2 — Changelog (post-recompute review)**

```
+--------------------------------------------------------------------+
| Holiday Data Changelog                                             |
+--------------------------------------------------------------------+
| Trigger: Copp Clark publish 2026-05-16 (added: Juneteenth obs.)    |
| Affected instruments: 142 / 3,418 in scope                         |
| Dates moved: 487                                                   |
|                                                                    |
|  [ View by instrument ]  [ View by date ]  [ Acknowledge ]         |
+--------------------------------------------------------------------+
|  Instrument             | Date type   | Was         | Now          |
| ------------------------+-------------+-------------+-------------|
|  ABC Global Macro       | Settle      | 2026-06-19  | 2026-06-22  |
|  XYZ Credit Opps        | Notif dl.   | 2026-06-05  | 2026-06-08  |
|  ...                                                               |
+--------------------------------------------------------------------+
```

**#3 — Holiday Overlay Manager (client admin)**

```
+--------------------------------------------------------------------+
| Holiday Overlays — Acme Capital                                    |
+--------------------------------------------------------------------+
| Base source: Copp Clark v2026.05                                   |
|                                                                    |
| Active overlay: "Acme house holidays"  (eff. 2026-01-01 → open)    |
|   2026-07-03  Half-day close (firm holiday)                        |
|   2026-12-24  Closed (firm holiday)                                |
|   ...                                                              |
|                                                                    |
|  [ + Add holiday ]  [ Upload file ]  [ Diff vs base ]  [ Publish ] |
+--------------------------------------------------------------------+
|  Conflicts with base: 2 entries (resolved in client favor)         |
+--------------------------------------------------------------------+
```

**#4 — NL Query surface (read-only across entitled book)**

```
+--------------------------------------------------------------------+
| Ask the calendar                                                   |
+--------------------------------------------------------------------+
|  > Which instruments have notification deadlines in the next 2     |
|    weeks?                                                          |
|                                                                    |
|  Found 23 instruments with deadlines between 2026-05-27 and        |
|  2026-06-10:                                                       |
|                                                                    |
|    2026-05-29 16:00 ET  ABC Global Macro (Class I)                 |
|    2026-06-02 12:00 LN  EMEA Credit Opps                           |
|    ...                                                             |
|                                                                    |
|  [ Open in Investment Planning ]   [ Export ]                      |
+--------------------------------------------------------------------+
```

---

## 2. Business Rules


1. **Single-engine guarantee:** The Compute API and Calendar API call the same computation engine over the same holiday data. For identical (instrument, anchor date, holiday source) inputs they MUST return identical dates. If the persisted calendar ever drifts from a fresh compute, that is a defect.
2. **Pluggable holiday source:** Copp Clark is the default. Clients can also supply their own holiday files in standard formats, or have Osyte integrate another vendor.
3. **Composite overlay support:** A client-specific overlay can be applied on top of a base vendor source. Where the overlay and base disagree on a given date, the overlay wins. The conflict is recorded in the changelog.
4. **Deterministic computation:** Date computation is purely rule-based — AI does not sit in the computation path.
5. **Effective-dated versioning:** Calendar versions are stored with effective dates. The service answers both "what does the calendar say now" and "what did it say on date X."
6. **Auto-recompute on holiday update:** When holiday data updates arrive (e.g., a Copp Clark publish), affected instrument calendars recompute automatically. A changelog records which instruments are affected and which dates moved.
7. **Standard roll conventions per rule:** Following, Modified Following, Preceding, and Modified Preceding are supported and configurable per key-date rule (notification / trade / valuation / settlement).
8. **Multi-calendar alignment:** Cross-currency and cross-market instruments that require multiple calendars to align as business days are supported from day one.
9. **Instrument-level term capture:** Liquidity terms are configured on the instrument record (notification offset, dealing frequency, valuation lag, settlement cycle, cut-off time, roll convention). Onboarding new instruments uses the standard Osyte instrument workflow in v1.
10. **Consistent entitlements across surfaces:** The NL query layer and MCP server expose the service's data; they MUST honor the same per-user / per-role entitlements as direct API access. (Draft-from-PRFAQ — PM to confirm exact entitlement model.)

---

## 3. Boundaries & Guardrails

**System constraints:**

1. **Reject compute against incomplete terms** : An instrument missing any required liquidity term (notification offset, dealing frequency, valuation lag, settlement cycle, cut-off time, roll convention) MUST NOT silently produce dates.
2. **Holiday update idempotency** : Re-applying the same Copp Clark version MUST NOT produce a different recompute outcome or duplicate changelog entries. → **On violation:** TBD — block re-apply vs. log-only.


---

## 4. Test Scenarios

### Acceptance criteria

**Phase 1 — Acceptance criteria:**

- **AC1:** Given an instrument with complete liquidity terms and an anchor date, the Compute API returns notification, trade, valuation, and settlement dates derived from the configured holiday source and roll convention.
- **AC2:** For the same instrument and anchor date, the Calendar API's persisted forward calendar returns dates identical to the Compute API.
- **AC3:** When a holiday data publish arrives (Copp Clark or client-supplied), affected instrument calendars recompute automatically and a changelog is published listing affected instruments and moved dates.
- **AC4:** A client-specific overlay on top of Copp Clark resolves conflicts in the client's favor, and the conflict is surfaced as an entry in the changelog.
- **AC7:** The persistent forward calendar exposes a configurable horizon of upcoming key dates for each in-scope instrument. The calendar is materialized by the engine, queryable without re-invoking the compute engine on each read, and survives service restarts. Reads return the most recent effective `CalendarVersion` for the instrument.

**Phase 2 — Acceptance criteria:**

- **AC5:** A historical query — "what did the calendar say for instrument X on date Y" — returns the version of the calendar effective on date Y, not the current version.
- **AC6:** NL query and MCP server responses for key dates are scoped to the requester's entitled instruments.

### Scenarios

> Each scenario includes Given/When/Then, the **Calculation steps** the engine (or system) executes, and the expected **Solution** — the concrete output that confirms the AC is met.

**Scenario 1 — Happy path: compute key dates for a monthly-dealing fund** *(maps to AC1, AC2 — Phase 1)*
- **Given** instrument "ABC Global Macro (Class I)" has complete liquidity terms (15 BD notification offset, monthly dealing on last BD, T+1 valuation, T+3 settle from valuation, Modified Following), and Copp Clark v2026.05 is the active holiday source
- **When** an Investment Planning user opens the instrument and views its forward calendar for the next 12 months
- **Then** the displayed dealing dates, notification deadlines, valuation dates, and settle dates match a fresh Compute API call against the same anchor date, and all dates correctly skip Copp Clark holidays per the Modified Following roll convention

**Calculation steps** (for anchor date 2026-07-31, assuming no US holidays in the surrounding window):
1. Resolve dealing date: last business day of July 2026 = Friday, 2026-07-31 (Modified Following — already a BD, no roll applied).
2. Compute notification deadline: count 15 business days backward from 2026-07-31, skipping weekends and Copp Clark US holidays → Friday, 2026-07-10 (cut-off 16:00 America/New_York).
3. Trade date = dealing date = 2026-07-31.
4. Compute valuation date: count 1 business day forward from trade date → Monday, 2026-08-03.
5. Compute settlement date: count 3 business days forward from valuation date → Thursday, 2026-08-06.

**Solution:**
```
notification_deadline: 2026-07-10T16:00:00-04:00
trade_date:           2026-07-31
valuation_date:       2026-08-03
settlement_date:      2026-08-06
```
A fresh Compute API call against the same (instrument, anchor date) returns the same four values — single-engine guarantee holds.

---

**Scenario 2 — Persistent forward calendar materializes and survives restart** *(maps to AC7 — Phase 1)*
- **Given** instrument "ABC Global Macro" has complete liquidity terms, a forward window depth of 12 months is configured, and the service has just materialized the instrument's calendar
- **When** the service is restarted and an Investment Planning user immediately requests `GET /api/v1/instruments/ABC-GM-001/calendar`
- **Then** the response returns the 12 months of key dates from the persisted `InstrumentCalendar` without invoking the engine on read, and the returned dates are identical to a fresh Compute API call against each dealing date in the window

**Calculation steps:**
1. On materialization, the engine ran once per dealing date in the 12-month window, producing N rows (one per dealing date) in `InstrumentCalendar`. Each row references the active `CalendarVersion`.
2. On read after restart, the Calendar API queries `InstrumentCalendar` by `instrument_id`, sorted by `dealing_date`. No engine invocation. No holiday-data resolution at read time.
3. The single-engine guarantee is verified independently: for each dealing date returned, a parallel Compute API call is issued and its output is compared row-by-row to the persisted value.

**Solution:**
- Calendar API returns N rows of (dealing_date, notification_deadline, trade_date, valuation_date, settlement_date) for ABC Global Macro across the next 12 months.

---

**Scenario 3 — Guardrail violation: compute against incomplete terms** *(maps to Section 3 system constraint #1 — Phase 1, failure mode TBD)*
- **Given** instrument "DEF Private Credit" has been onboarded but its valuation lag and roll convention are not yet configured
- **When** any consumer (Compute API, Calendar API, NL query, or MCP server) requests key dates for DEF
- **Then** the system rejects the request with a typed error identifying the missing fields, and no partial date set is returned or persisted

---

**Scenario 4 — Holiday update triggers recompute and changelog** *(maps to AC3, AC4 — Phase 1)*
- **Given** Copp Clark publishes an update adding a new observed holiday on 2026-07-03, and Acme Capital has 142 active instruments referencing that calendar
- **When** the update is ingested
- **Then** all 142 affected instrument calendars are recomputed within the recompute SLA (TBD), a changelog is published listing each instrument and each moved date, and downstream Investment Planning / Rebalancing / Trade Ops reads return the new dates on next refresh

**Calculation steps** (for one affected instrument, ABC Global Macro, dealing date 2026-07-31, 15 BD notification offset):
1. Ingest new Copp Clark version; persist holiday entry for 2026-07-03.
2. Recompute pipeline identifies affected instruments: forward calendar window intersects 2026-07-03 → ABC Global Macro is in the affected set.
3. Re-run the engine for ABC Global Macro at its July 2026 anchor. Before the publish, notification deadline = 2026-07-10. After the publish, the engine skips 2026-07-03 (now a holiday) when counting 15 BD backward → notification deadline moves to **Thursday, 2026-07-09**.
4. Diff old vs new `InstrumentCalendar`. Write a `ChangelogEntry` row for the moved date. Write a new `CalendarVersion` stamped with the effective-from timestamp.
5. Emit `RecomputeEvent`. Subscribers (Investment Planning, Rebalancing, Trade Ops) refresh on next read.

**Solution:**
- Changelog row:
  ```
  Instrument: ABC Global Macro
  Date type:  Notification deadline (for 2026-07-31 dealing)
  Was:        2026-07-10
  Now:        2026-07-09
  Reason:     Copp Clark added 2026-07-03 as observed holiday
  ```
- All 142 affected instruments are visible in the changelog UI.
- A fresh Compute API call against ABC Global Macro returns the new deadline (2026-07-09), matching the persisted calendar.


---

**Scenario 5 — Entitlement boundary: NL query scopes to entitled book** *(maps to AC6, Section 3 #4 — **Phase 2**)*
- **Given** user U is entitled to read instruments in "Acme Alternatives" book but not "Acme Public Equities"
- **When** U submits the NL query "show me all instruments with notification deadlines in the next 2 weeks"
- **Then** the response includes only Alternatives instruments and never references the Public Equities book, even when matching instruments exist there

**Solution:**
- Response lists only Acme Alternatives instruments with deadlines in the window.
- Response never mentions Public Equities by name, ID, or count (no leakage of existence).
- Audit log records: NL query issued by U, entitlement filter applied, N matching instruments returned out of M scanned across U's entitled book.

---
---

**Scenario 6 — Historical version integrity** *(maps to AC5 — **Phase 2**)*
- **Given** instrument "GHI Hedge Fund" had its calendar recomputed three times between 2026-01-01 and 2026-05-01 due to overlay changes — `v1` effective 2026-01-01, `v2` effective 2026-02-12, `v3` effective 2026-04-05
- **When** an auditor queries "what was the notification deadline for GHI's April dealing date, as of 2026-03-15"
- **Then** the response returns the deadline from the calendar version that was effective on 2026-03-15 — not the current version — and the response identifies the version and effective-date range used


**Solution:**
```
{
  "instrument_id": "GHI-HF-001",
  "calendar_version": "v2",
  "version_effective_from": "2026-02-12T00:00:00Z",
  "version_effective_to":   "2026-04-04T23:59:59Z",
  "as_of_requested":        "2026-03-15",
  "note": "Returning the calendar version effective on 2026-03-15. Current version is v3.",
  "key_dates": [
    {
      "dealing_date":           "2026-04-30",
      "notification_deadline":  "<as recorded in v2>",
      "trade_date":             "2026-04-30",
      "valuation_date":         "<as recorded in v2>",
      "settlement_date":        "<as recorded in v2>"
    }
  ]
}
```
The deadline returned is the one the firm relied on at the time. An auditor in 2027 can reproduce the same response with the same `as_of` parameter.

---

**Scenario 6 — Persistent forward calendar materializes and survives restart** *(maps to AC7 — Phase 1)*
- **Given** instrument "ABC Global Macro" has complete liquidity terms, a forward window depth of 12 months is configured, and the service has just materialized the instrument's calendar
- **When** the service is restarted and an Investment Planning user immediately requests `GET /api/v1/instruments/ABC-GM-001/calendar`
- **Then** the response returns the 12 months of key dates from the persisted `InstrumentCalendar` without invoking the engine on read, and the returned dates are identical to a fresh Compute API call against each dealing date in the window

**Calculation steps:**
1. On materialization, the engine ran once per dealing date in the 12-month window, producing N rows (one per dealing date) in `InstrumentCalendar`. Each row references the active `CalendarVersion`.
2. On read after restart, the Calendar API queries `InstrumentCalendar` by `instrument_id`, sorted by `dealing_date`. No engine invocation. No holiday-data resolution at read time.
3. The single-engine guarantee is verified independently: for each dealing date returned, a parallel Compute API call is issued and its output is compared row-by-row to the persisted value.

**Solution:**
- Calendar API returns N rows of (dealing_date, notification_deadline, trade_date, valuation_date, settlement_date) for ABC Global Macro across the next 12 months.

---

## 5. Logical / Data Model Schema

> All Phase 1 entities below must be built so Phase 2 can add `CalendarVersion` history without schema rewrites. In particular, `InstrumentCalendar` should be designed with version-snapshot semantics from day one, even if only the active version is queryable in Phase 1.

| Entity | Purpose | Status |
|---|---|---|
| `Instrument` | Existing Osyte entity in `Security Master: Schema dealcloud_fundproduct_master`; extended to carry `LiquidityTerms` reference. | Existing |
| `LiquidityTerms` | Per-instrument capture of notification offset, dealing frequency, valuation lag, settlement cycle, cut-off time, time zone, default roll convention. | Existing |
| `KeyDateRule` | One rule per derived date type (notification / trade / valuation / settlement) with its own offset, calendar reference, and roll convention. | |
| `HolidaySource` | Source descriptor: vendor (e.g., Copp Clark), version, market coverage. |  |
| `HolidayOverlay` | Client-specific overlay layered on a `HolidaySource`; effective-dated; lists holiday adds/removes. |  |
| `HolidayCalendar` | Materialized holiday set = `HolidaySource` + applied overlays, resolved at an effective date. |  |
| `InstrumentCalendar` | Persisted forward-looking key dates for one instrument under one resolved `HolidayCalendar`. | (Phase 1) |
| `CalendarVersion` | Effective-dated immutable snapshot of an `InstrumentCalendar`; powers historical queries. |  (Phase 2) |
| `RecomputeEvent` | Triggered by a holiday data update or overlay change; spawns `ChangelogEntry` records. |  |
| `ChangelogEntry` | One row per (instrument, date type) that moved as a result of a `RecomputeEvent`; carries old → new. | Phase 1 |

**Key relationships:**
- `Instrument` 1—1 `LiquidityTerms` (or 1—* across effective-dated versions — TBD)
- `LiquidityTerms` 1—* `KeyDateRule`
- `HolidaySource` 1—* `HolidayOverlay` (per client)
- `HolidaySource` + `HolidayOverlay[]` → resolved `HolidayCalendar` (effective-dated)
- `Instrument` 1—* `InstrumentCalendar` (one active, many historical versions via `CalendarVersion`)
- `RecomputeEvent` 1—* `ChangelogEntry`

**Confirmed inputs:**
- Copp Clark holiday data — external integration
- Client-supplied holiday files — new ingestion path
- Existing Osyte `Instrument` records in `Security Master` — extended with liquidity-terms data

---

## Open Questions

1. **Cut-off time-zone handling** — Which TZ governs when an instrument's cut-off time intersects multiple market calendars? Currently, we are using UTC.
2. **Behavior on incomplete liquidity terms** — Block, partial response, or typed error? (Scenario 3 currently assumes block.)
3. **Multi-overlay conflict resolution** — If a client can stack overlays, how do conflicts between overlays (not just overlay vs. base) resolve?
4. **Forward window depth for persisted calendar** — How many months / dealing cycles forward?
5. **Composite overlay depth** — "Copp Clark + own exception list" is supported; clarify how many overlays may stack and in what order.