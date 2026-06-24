# Breakroom API Contract

## What Breakroom does

Enterprise clients reconcile custodian transaction feeds against Osyte's internal book of record daily. Today, ops analysts manually download two CSVs, paste them into Excel, write VLOOKUP formulas, and spend hours hunting down breaks that may turn out to be a date format mismatch or a sign convention difference. Breakroom replaces that with a deterministic matching engine.

**Breakroom solves two problems:**

1. **"Do these two feeds agree?"** — Given a configured reconciliation environment and two CSV feeds, run the full validation, normalization, and matching pipeline. Classify every custodian record as Auto-matched, Partial Match, Unmatched, or Duplicate.

2. **"How do I set up reconciliation against a new custodian?"** — In a single call, provide both feed schemas, the field mapping between them, normalization and filter rules to handle format differences, and the composite key and tolerance rules for matching. Breakroom stores all of this as a named environment. Every subsequent daily run references the environment by ID — no ruleset resent, no config drift.

---

## Methods

Three methods across two APIs:

### Reconciliation Config API

Stateful. One call per custodian relationship. Stores the complete configuration — feed schemas, field mappings, transformation rules, and matching conditions — as a named, versioned environment.

| # | Method name | Route | Solves | What it does |
|---|---|---|---|---|
| 1 | `createEnvironment()` | `POST /recon-config/createEnvironment` | Problem 2 | Creates a reconciliation environment in a single call with everything needed to run sessions against a custodian: both feed schemas, field mapping, filter rules, normalization rules, composite key, and tolerances |

### Reconciliation Sessions API

Stateful. Runs reconciliation sessions against a configured environment. The caller sends only the environment ID and the two CSV files — no config resent.

| # | Method name | Route | Solves | What it does |
|---|---|---|---|---|
| 2 | `runSession()` | `POST /recon-sessions/runSession` | Problem 1 | Uploads both feeds and runs the full validation and classification pipeline against the stored environment config. Returns a session ID and summary. |
| 3 | `getSessionResults()` | `GET /recon-sessions/getSessionResults/{session_id}` | Problem 1 | Returns classified records for a session, filterable by status |

---

## Assumptions

### 1. Configuration is set once, sessions run daily

`createEnvironment()` is called once per custodian relationship. Every `runSession()` call references the environment by ID — feed schemas, normalization rules, composite key, and tolerances are all stored server-side. The caller sends only the two CSV files per run.

**Why:** Resending the full config with every daily session is wasteful, error-prone, and breaks auditability. Storing config separately means every session can be traced to the exact ruleset it ran against. If CNB changes their feed format, the caller creates a new environment version — old sessions stay linked to the old config.

### 2. Normalization applies to the external (custodian) feed only

The internal Osyte feed is the source of truth. All format standardization — date parsing, case normalization, canonical value mapping, sign convention adjustment — runs against the external feed to align it with internal field formats before matching.

**Why:** Transforming the custodian data to match Osyte's format is the natural direction. Osyte controls the internal schema; it does not change per custodian.

### 3. Matching runs external → internal

Each external (custodian) record is looked up against the internal feed using the composite key. Internal records with no custodian counterpart are not classified in v1. See Open Questions.

**Why:** The custodian feed drives the reconciliation. The v1 scope is to validate that the custodian's view of trades matches Osyte's — not to audit whether every Osyte trade appears in the custodian feed.

### 4. Bridge tables for cross-system value mapping are caller-managed

The composite key maps Osyte integer fields (e.g. `fund_id`, `account_id`) to custodian text identifiers (e.g. `Account #`, `Security ID`). The resolution of that mapping is provided as a `TXT-04` normalization rule referencing a named bridge table. Breakroom does not maintain bridge tables natively.

**Why:** Bridge tables are tenant- and custodian-specific. The caller populates them during environment setup and updates them when account relationships change.

---

## Reconciliation Config API: `createEnvironment()`

**Route:** `POST /recon-config/createEnvironment`

**Purpose:** "Set up everything Breakroom needs to reconcile Osyte against this custodian."

Creates a named reconciliation environment in a single call. The request contains the complete configuration: both feed schemas, the field mapping between them, filter and normalization rules for the external feed, the composite key definition, and tolerance rules for comparison fields. On success the environment is immediately active and ready to receive sessions.

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `environment_name` | string | yes | A human-readable name for this environment. Convention: `Osyte-{CustodianShortCode}-Reconciliation`. E.g. `Osyte-CNB-Reconciliation`. |
| `tenant_id` | string | yes | The tenant this environment belongs to. |
| `reconciliation_type` | string | no | Default: `trade_reconciliation`. v1 supports only `trade_reconciliation`. |
| `internal_feed` | object | yes | Feed definition for the Osyte internal data. Contains `feed_name` and `fields`. |
| `external_feed` | object | yes | Feed definition for the custodian data. Contains `feed_name` and `fields`. |
| `field_mapping` | object[] | yes | Pairs linking each internal mandatory field to its counterpart in the external feed. Only mandatory fields appear here. |
| `filter_rules` | object[] | no | Records to include or exclude before matching. Applied before normalization. |
| `normalization_rules` | object[] | no | Format standardization rules applied to the external feed before matching. |
| `composite_key` | string[] | yes | Labels of internal feed fields that together form the unique record identifier. |
| `tolerance_rules` | object[] | no | Acceptable variance on non-key comparison fields. Fields with no tolerance rule require an exact match after normalization. |

**`internal_feed.fields` and `external_feed.fields` — each field object:**

| Field | Type | What it means |
|---|---|---|
| `label` | string | The column header as it appears in the CSV. |
| `data_type` | string | `numeric`, `text`, or `date`. |
| `format` | string | The specific format or pattern. E.g. `DECIMAL(18,4)`, `VARCHAR(30)`, `DATE (M/D/YYYY)`. |
| `mandatory` | boolean | Whether this field participates in mapping and matching. Non-mandatory fields are stored but ignored in comparisons. |

**`field_mapping` — each mapping object:**

| Field | Type | What it means |
|---|---|---|
| `internal_field` | string | Label of the internal feed field. |
| `external_field` | string | Label of the external feed field it maps to. |

The mapping defines an exact-match expectation between the two fields, subject to normalization and tolerance rules.

**`filter_rules` — each rule object:**

| Field | Type | What it means |
|---|---|---|
| `feed` | string | Which feed to filter: `internal` or `external`. |
| `field_label` | string | The field to evaluate. |
| `action` | string | `include` (keep only matching records) or `exclude` (drop matching records). |
| `operator` | string | `equals`, `not_equals`, `in`, `not_in`. |
| `values` | array | The values to test against. |

**`normalization_rules` — each rule object:**

| Field | Type | What it means |
|---|---|---|
| `rule_id` | string | The rule to apply. See rule catalog below. |
| `target_fields` | string[] | The external feed fields this rule applies to. |
| `parameters` | object | Rule-specific parameters. Only `TXT-04` requires a parameter (`bridge_table`). |

**Normalization rule catalog:**

| Rule ID | Category | What it does |
|---|---|---|
| `TXT-01` | Text | Convert all values to upper case |
| `TXT-02` | Text | Trim leading/trailing spaces; collapse internal multiples |
| `TXT-03` | Text | Remove non-meaningful separators (`-`, `/`, `.`, `_`) unless part of a structural identifier |
| `TXT-04` | Text | Map different labels to a canonical value via a named bridge table (e.g. `"buy"`, `"Purchase"`, `"B"` → `"BTO"`) |
| `TXT-05` | Text | Replace empty, whitespace-only, or placeholder values with `N/A` |
| `TXT-06` | Text | Remove leading zeros and trailing fillers from identifier fields |
| `NUM-01` | Numeric | Convert to fixed 4-decimal precision |
| `NUM-02` | Numeric | Apply `HALF_UP` rounding to target precision |
| `NUM-03` | Numeric | Convert to absolute value where sign is non-meaningful (e.g. custodian sends negative quantity for sells; Osyte stores always-positive) |
| `NUM-04` | Numeric | Replace null, empty, or non-numeric values with `0` |
| `NUM-05` | Numeric | Combine split integer and decimal components into a single value |
| `DT-01` | Date | Convert all timestamps to UTC |
| `DT-02` | Date | Remove time component when date-only comparison is required |
| `DT-03` | Date | Convert all dates to ISO 8601 (`YYYY-MM-DD`) |
| `DT-04` | Date | Treat dates within ±1 business day as equal (applied at match time, not as a transform) |
| `DT-05` | Date | Replace null or invalid dates with placeholder `1900-01-01` |

**`tolerance_rules` — each rule object:**

| Field | Type | What it means |
|---|---|---|
| `field_label` | string | Internal feed label of the comparison field. Must be a mapped field. |
| `tolerance_type` | string | `absolute` (fixed numeric difference), `percentage` (relative to the internal value), or `business_days` (calendar tolerance for date fields). |
| `tolerance_value` | float | The permitted deviation. E.g. `1` for `business_days`, `0.0001` for `percentage`, `0.01` for `absolute`. |

### Example — Setting up Osyte ↔ City National Bank reconciliation for Common Fund

Common Fund holds positions custodied at City National Bank (CNB). CNB delivers a 15-column CSV daily. Osyte's internal feed has 81 columns, 6 mandatory. Nine fields are mapped — 4 form the composite key, 5 are comparison fields. CNB sends transaction types as free text and quantities as negative for sells; normalization aligns both to Osyte's format before matching.

**Request:**
```jsonc
POST /recon-config/createEnvironment

{
  "environment_name": "Osyte-CNB-Reconciliation",
  "tenant_id": "common-fund",
  "reconciliation_type": "trade_reconciliation",

  "internal_feed": {
    "feed_name": "Osyte Records",
    "fields": [
      {"label": "fund_id",                   "data_type": "numeric", "format": "integer",               "mandatory": true},
      {"label": "account_id",                "data_type": "numeric", "format": "integer",               "mandatory": true},
      {"label": "trade_transaction_type_cd", "data_type": "text",    "format": "VARCHAR(30)",           "mandatory": true},
      {"label": "trade_amt",                 "data_type": "numeric", "format": "DECIMAL(18,4)",         "mandatory": true},
      {"label": "trade_dt",                  "data_type": "date",    "format": "DATE (M/D/YYYY)",       "mandatory": true},
      {"label": "cash_transaction_id",       "data_type": "numeric", "format": "integer",               "mandatory": true},
      {"label": "trade_entered_dt",          "data_type": "date",    "format": "DATETIME (M/D/YYYY H:MM)", "mandatory": false},
      {"label": "settlement_dt",             "data_type": "date",    "format": "DATE (M/D/YYYY)",       "mandatory": false},
      {"label": "final_quantity",            "data_type": "numeric", "format": "DECIMAL(18,6)",         "mandatory": false},
      {"label": "avg_price_per_share",       "data_type": "numeric", "format": "DECIMAL(18,4)",         "mandatory": false},
      {"label": "net_cash_amt",              "data_type": "numeric", "format": "DECIMAL(18,4)",         "mandatory": false}
      // full 81-field schema defined in Osyte internal feed reference
    ]
  },

  "external_feed": {
    "feed_name": "CNB Records",
    "fields": [
      {"label": "Account #",                "data_type": "text",    "format": "VARCHAR(20)",              "mandatory": true},
      {"label": "Primary Account Holder",   "data_type": "text",    "format": "VARCHAR(200)",             "mandatory": true},
      {"label": "Date (Entry)",             "data_type": "date",    "format": "MM/DD/YYYY or MM-DD-YYYY", "mandatory": true},
      {"label": "Date (Trade)",             "data_type": "date",    "format": "MM/DD/YYYY or MM-DD-YYYY", "mandatory": true},
      {"label": "Date (Settle)",            "data_type": "date",    "format": "MM/DD/YYYY",               "mandatory": true},
      {"label": "Transaction Type",         "data_type": "text",    "format": "VARCHAR(50)",              "mandatory": true},
      {"label": "Transaction Key/Mnemonic", "data_type": "text",    "format": "VARCHAR(20)",              "mandatory": true},
      {"label": "Quantity",                 "data_type": "numeric", "format": "DECIMAL(18,6)",            "mandatory": true},
      {"label": "Price",                    "data_type": "numeric", "format": "DECIMAL(18,4)",            "mandatory": true},
      {"label": "Net Amount",               "data_type": "numeric", "format": "DECIMAL(18,2)",            "mandatory": true},
      {"label": "Security ID",              "data_type": "text",    "format": "VARCHAR(20)",              "mandatory": true},
      {"label": "Description",              "data_type": "text",    "format": "VARCHAR(200)",             "mandatory": true},
      {"label": "Security Type Code",       "data_type": "text",    "format": "VARCHAR(30)",              "mandatory": true},
      {"label": "Cancel",                   "data_type": "text",    "format": "VARCHAR(3)",               "mandatory": true},
      {"label": "Reference #",              "data_type": "text",    "format": "VARCHAR(30)",              "mandatory": false}
    ]
  },

  "field_mapping": [
    {"internal_field": "fund_id",                   "external_field": "Account #"},
    {"internal_field": "account_id",                "external_field": "Security ID"},
    {"internal_field": "trade_dt",                  "external_field": "Date (Trade)"},
    {"internal_field": "trade_transaction_type_cd", "external_field": "Transaction Type"},
    {"internal_field": "trade_entered_dt",          "external_field": "Date (Entry)"},
    {"internal_field": "settlement_dt",             "external_field": "Date (Settle)"},
    {"internal_field": "final_quantity",            "external_field": "Quantity"},
    {"internal_field": "avg_price_per_share",       "external_field": "Price"},
    {"internal_field": "net_cash_amt",              "external_field": "Net Amount"}
  ],

  "filter_rules": [
    {
      "feed": "external",
      "field_label": "Transaction Type",
      "action": "include",
      "operator": "in",
      "values": ["Buy", "Sell", "BUY", "SELL", "Purchase", "BTO", "STC"]
    },
    {
      "feed": "external",
      "field_label": "Cancel",
      "action": "exclude",
      "operator": "equals",
      "values": ["Yes"]
    },
    {
      "feed": "internal",
      "field_label": "trade_item_status_id",
      "action": "exclude",
      "operator": "in",
      "values": [15, 16]
    }
  ],

  "normalization_rules": [
    {"rule_id": "TXT-01", "target_fields": ["Transaction Type", "Security ID", "Account #"]},
    {"rule_id": "TXT-02", "target_fields": ["Account #", "Security ID", "Transaction Type"]},
    {"rule_id": "TXT-04", "target_fields": ["Transaction Type"], "parameters": {"bridge_table": "REF_TxType_Bridge"}},
    {"rule_id": "TXT-04", "target_fields": ["Account #"],        "parameters": {"bridge_table": "REF_Account_Bridge"}},
    {"rule_id": "TXT-04", "target_fields": ["Security ID"],      "parameters": {"bridge_table": "REF_Security_Bridge"}},
    {"rule_id": "TXT-06", "target_fields": ["Account #", "Security ID"]},
    {"rule_id": "NUM-01", "target_fields": ["Price", "Net Amount", "Quantity"]},
    {"rule_id": "NUM-02", "target_fields": ["Price", "Net Amount"]},
    {"rule_id": "NUM-03", "target_fields": ["Quantity"]},
    {"rule_id": "DT-01",  "target_fields": ["Date (Entry)"]},
    {"rule_id": "DT-02",  "target_fields": ["Date (Trade)", "Date (Settle)"]},
    {"rule_id": "DT-03",  "target_fields": ["Date (Entry)", "Date (Trade)", "Date (Settle)"]}
  ],

  "composite_key": ["fund_id", "account_id", "trade_dt", "trade_transaction_type_cd"],

  "tolerance_rules": [
    {"field_label": "trade_entered_dt",    "tolerance_type": "business_days", "tolerance_value": 1},
    {"field_label": "avg_price_per_share", "tolerance_type": "percentage",    "tolerance_value": 0.0001},
    {"field_label": "net_cash_amt",        "tolerance_type": "absolute",      "tolerance_value": 0.01},
    {"field_label": "final_quantity",      "tolerance_type": "absolute",      "tolerance_value": 0},
    {"field_label": "settlement_dt",       "tolerance_type": "business_days", "tolerance_value": 0}
  ]
}
```

```
fund_id (Osyte integer) maps to Account # (CNB varchar) — not a direct type match.
account_id (Osyte integer) maps to Security ID (CNB ticker string) — same.
TXT-04 rules on both fields resolve CNB identifiers to Osyte integers via bridge tables
before the composite key lookup runs.

TXT-04 on Transaction Type: "buy" → "BUY" (TXT-01) → "BTO" (TXT-04 via REF_TxType_Bridge).
NUM-03 on Quantity: -214803.848 (CNB sell) → 214803.848. Matches Osyte's always-positive final_quantity.
DT-03 on Date (Trade): "06-12-2025" / "6/12/2025" → "2025-06-12". Handles CNB's mixed date formats.

trade_entered_dt: ±1 business day — custodian posts entry date the next morning.
avg_price_per_share: 0.0001% — covers floating-point differences after 4-decimal standardization.
net_cash_amt: ±$0.01 — covers penny-rounding differences in fee treatment.
final_quantity: exact match (0 tolerance) — quantity differences are material breaks.
settlement_dt: exact match — settlement date differences are material.
```

**Response:**
```jsonc
{
  "environment_id": "env-cf-cnb-001",
  "environment_name": "Osyte-CNB-Reconciliation",
  "tenant_id": "common-fund",
  "reconciliation_type": "trade_reconciliation",
  "status": "active",
  "mapped_fields": 9,
  "composite_key": ["fund_id", "account_id", "trade_dt", "trade_transaction_type_cd"],
  "tolerance_rules_set": 5
}
```

The environment is immediately `active` — `runSession()` can be called as soon as this response is received.

### `createEnvironment()` output signature

```jsonc
{
  "environment_id": "string",
  "environment_name": "string",
  "tenant_id": "string",
  "reconciliation_type": "string",
  "status": "active",
  "mapped_fields": "int",
  "composite_key": ["string"],
  "tolerance_rules_set": "int"
}
```

---

## Reconciliation Sessions API: `runSession()`

**Route:** `POST /recon-sessions/runSession`

**Purpose:** "Do these two feeds agree?"

Uploads the internal and external CSV feeds against a configured environment and runs the full processing pipeline: four-step basic validation, normalization service, duplicate check, composite key matching, and field-level comparison. Returns a session ID and summary. Record-level results are retrieved via `getSessionResults()`.

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `environment_id` | string | yes | The active environment to run against. |
| `internal_feed_data` | string | yes | Base64-encoded CSV content for the internal (Osyte) feed. Must conform to the feed schema defined in `createEnvironment()`. |
| `external_feed_data` | string | yes | Base64-encoded CSV content for the external (custodian) feed. Must conform to the external feed schema. |

Both feeds are passed as base64-encoded strings. For large files, multipart/form-data upload is also accepted — use field names `internal_feed_data` and `external_feed_data`.

### Processing pipeline

The engine runs four sequential validation checks on each feed. Any failure stops the session:

| Step | Check | Failure code |
|---|---|---|
| 1 | Feed format check — confirms CSV structure is well-formed | `feed_format_error` |
| 2 | Feed field check — confirms all mandatory fields are present | `missing_feed_fields` |
| 3 | Data type check — confirms field values match configured data types | `data_type_mismatch` |
| 4 | Normalization service — applies filter and normalization rules | `normalization_error` |

If all four pass, record validation begins:
1. Each external record is assigned a system-generated `record_id`.
2. **Duplicate check:** records sharing the same composite key are compared field-by-field. Exact duplicates → `duplicate`. Unique records among those sharing a key are prioritized and passed through.
3. **Key lookup:** the composite key is looked up against the internal feed.
4. **Classification:** No internal match → `unmatched`. Key matches but a non-key field differs beyond tolerance → `partial_match`. All fields within tolerance → `auto_matched`.

Auto-matched records trigger a write-back to the internal record's `reconciliation_status_code` field (set to `AUTO_MATCH`).

### Example — End-of-day reconciliation, Common Fund / CNB

Four Osyte trades vs. four CNB records. Two match exactly. One has a price difference beyond tolerance. One CNB record resolves to no known Osyte account.

**Request:**
```jsonc
POST /recon-sessions/runSession

{
  "environment_id": "env-cf-cnb-001",
  "internal_feed_data": "ZnVuZF9pZCxhY2NvdW50X2lkLHRyYWRlX3RyYW5zYWN0aW9uX3R5cGVfY2QsdHJhZGVfYW10...",
  "external_feed_data": "QWNjb3VudCAjLFByaW1hcnkgQWNjb3VudCBIb2xkZXIsTG9nIERhdGUsd..."
}
```

Internal feed (decoded, 4 records):
```
fund_id, account_id, trade_transaction_type_cd, trade_dt,   trade_entered_dt,   settlement_dt, final_quantity, avg_price_per_share, net_cash_amt
12425,   234644,     BTO,                        5/22/2026,  5/15/2026 4:51,     5/26/2026,     214803.848,     257.6500,            55344211.4500
12425,   234644,     STC,                        5/22/2026,  5/15/2026 9:30,     5/26/2026,     100000.000,     150.2500,            -15025000.0000
12425,   235001,     BTO,                        5/22/2026,  5/16/2026 10:00,    5/26/2026,     50000.000,      100.0000,            5000000.0000
12425,   234888,     STC,                        5/22/2026,  5/17/2026 11:00,    5/26/2026,     75000.000,      200.0000,            -15000000.0000
```

External feed (decoded, 4 records):
```
Account #,   Date (Trade), Transaction Type, Quantity,    Price,    Net Amount,   Security ID, Cancel
NYK-003640,  5/22/2026,    buy,              214803.848,  257.6500, 55344211.45,  AVGO,        No
NYK-003640,  5/22/2026,    Sell,             -100000,     150.2500, -15025000.00, AAPL,        No
NYK-003641,  5/22/2026,    Buy,              50000,       100.0500, 5002500.00,   MSFT,        No
NYK-003999,  5/22/2026,    Sell,             30000,       180.0000, -5400000.00,  TSLA,        No
```

```
Normalization:
  TXT-01/TXT-04: "buy" → "BUY" → "BTO", "Sell"/"Buy" → "SELL"/"BUY" → "STC"/"BTO"
  TXT-04: NYK-003640 → 12425 / AVGO → 234644, AAPL → 234644 (STC), MSFT → 235001 via bridges
  NUM-03: -100000 → 100000 (Quantity sign removed for sells)
  DT-03:  "5/22/2026" → "2026-05-22"

Duplicate check: no duplicate composite keys.

Classification:
  Record 1: key (12425, 234644, 2026-05-22, BTO) → match. All fields within tolerance → AUTO_MATCHED.
  Record 2: key (12425, 234644, 2026-05-22, STC) → match. All fields within tolerance → AUTO_MATCHED.
  Record 3: key (12425, 235001, 2026-05-22, BTO) → match. Price: 100.0500 vs 100.0000.
    Diff: 0.05%. Tolerance: 0.0001%. Exceeds → PARTIAL_MATCH.
  Record 4: NYK-003999 resolves to no fund_id in REF_Account_Bridge → no key match → UNMATCHED.
```

**Response:**
```jsonc
{
  "session_id": "session-20260617-001",
  "environment_id": "env-cf-cnb-001",
  "status": "completed",
  "validation": {
    "feed_format_check": "passed",
    "feed_field_check": "passed",
    "data_type_check": "passed",
    "normalization_check": "passed"
  },
  "summary": {
    "external_records_processed": 4,
    "auto_matched": 2,
    "partial_match": 1,
    "unmatched": 1,
    "duplicate": 0
  }
}
```

### `runSession()` output signature

```jsonc
{
  "session_id": "string",
  "environment_id": "string",
  "status": "completed | failed | in_progress",
  "validation": {
    "feed_format_check": "passed | failed",
    "feed_field_check": "passed | failed",
    "data_type_check": "passed | failed",
    "normalization_check": "passed | failed"
  },
  "summary": {
    "external_records_processed": "int",
    "auto_matched": "int",
    "partial_match": "int",
    "unmatched": "int",
    "duplicate": "int"
  }
}
```

If status is `failed`, only the validation block is populated. The first failed check is the termination point — subsequent checks did not run.

---

## Reconciliation Sessions API: `getSessionResults()`

**Route:** `GET /recon-sessions/getSessionResults/{session_id}`

**Purpose:** "Show me the classified records from this session."

Returns record-level results for a session. Each record includes its assigned status, field-level comparison details, and (for matched records) a reference to the matched internal record. Filterable by status. Paginated.

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `session_id` | string | yes | The session to retrieve results for (in the URL path). |
| `status` | string | no | Filter by classification: `auto_matched`, `partial_match`, `unmatched`, `duplicate`. Omit to return all records. |
| `page` | int | no | Page number. Default: 1. |
| `page_size` | int | no | Records per page. Default: 50. Max: 500. |

### Example — Reviewing partial match records from session-20260617-001

**Request:** `GET /recon-sessions/getSessionResults/session-20260617-001?status=partial_match`

**Response:**
```jsonc
{
  "session_id": "session-20260617-001",
  "environment_id": "env-cf-cnb-001",
  "status_filter": "partial_match",
  "total_records": 1,
  "page": 1,
  "page_size": 50,
  "records": [
    {
      "record_id": "REC-003",
      "status": "partial_match",
      "external_record": {
        "Account #": "NYK-003641",
        "Date (Trade)": "5/22/2026",
        "Transaction Type": "Buy",
        "Quantity": 50000,
        "Price": 100.05,
        "Net Amount": 5002500.00,
        "Security ID": "MSFT"
      },
      "matched_internal_record_ref": "610991",
      "field_comparison": [
        {"field": "fund_id / Account #",              "internal": "12425",         "external_normalized": "12425",        "match": true,  "tolerance_applied": null},
        {"field": "account_id / Security ID",         "internal": "235001",        "external_normalized": "235001",       "match": true,  "tolerance_applied": null},
        {"field": "trade_dt / Date (Trade)",          "internal": "2026-05-22",    "external_normalized": "2026-05-22",   "match": true,  "tolerance_applied": null},
        {"field": "trade_transaction_type_cd / Type", "internal": "BTO",           "external_normalized": "BTO",          "match": true,  "tolerance_applied": null},
        {"field": "avg_price_per_share / Price",      "internal": "100.0000",      "external_normalized": "100.0500",     "match": false, "tolerance_applied": {"type": "percentage",    "limit": 0.0001, "actual": 0.0005}},
        {"field": "net_cash_amt / Net Amount",        "internal": "5000000.0000",  "external_normalized": "5002500.0000", "match": false, "tolerance_applied": {"type": "absolute",      "limit": 0.01,   "actual": 2500.00}},
        {"field": "final_quantity / Quantity",        "internal": "50000.000000",  "external_normalized": "50000.000000", "match": true,  "tolerance_applied": null},
        {"field": "trade_entered_dt / Date (Entry)",  "internal": "2026-05-16",    "external_normalized": "2026-05-16",   "match": true,  "tolerance_applied": {"type": "business_days", "limit": 1, "actual": 0}},
        {"field": "settlement_dt / Date (Settle)",    "internal": "2026-05-26",    "external_normalized": "2026-05-26",   "match": true,  "tolerance_applied": null}
      ]
    }
  ]
}
```

### `getSessionResults()` output signature

```jsonc
{
  "session_id": "string",
  "environment_id": "string",
  "status_filter": "string | null",
  "total_records": "int",
  "page": "int",
  "page_size": "int",
  "records": [
    {
      "record_id": "string",
      "status": "auto_matched | partial_match | unmatched | duplicate",
      "external_record": "object (custodian field values as received)",
      "matched_internal_record_ref": "string | null",
      "field_comparison": [
        {
          "field": "string (internal_label / external_label)",
          "internal": "string",
          "external_normalized": "string",
          "match": "boolean",
          "tolerance_applied": "{ type: string, limit: float, actual: float } | null"
        }
      ]
    }
  ]
}
```

`matched_internal_record_ref` and `field_comparison` are `null` for `unmatched` and `duplicate` records.

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
| `invalid_field_mapping` | 422 | A `field_mapping` entry references a field label not defined in either feed, or references a non-mandatory field | `createEnvironment` |
| `invalid_composite_key` | 422 | A field in `composite_key` is not present in `field_mapping` | `createEnvironment` |
| `invalid_tolerance_rule` | 422 | A field in `tolerance_rules` is not a mapped field, or `tolerance_type` is not valid for the field's `data_type` | `createEnvironment` |
| `environment_not_found` | 404 | No environment found for the given `environment_id` | `runSession` |
| `feed_format_error` | 422 | Uploaded feed is not a valid CSV | `runSession` |
| `missing_feed_fields` | 422 | One or more mandatory fields are absent from the uploaded feed | `runSession` |
| `data_type_mismatch` | 422 | A field value does not match its configured `data_type` | `runSession` |
| `normalization_error` | 422 | A normalization rule failed to execute (e.g. bridge table not found, unparseable date) | `runSession` |
| `session_not_found` | 404 | No session found for the given `session_id` | `getSessionResults` |
| `session_failed` | 400 | `getSessionResults()` called on a session that did not complete successfully | `getSessionResults` |

---

## Open Questions

### 1. Should matching run in both directions?

The engine currently matches external → internal: each custodian record is looked up against the Osyte feed. An Osyte trade with no custodian counterpart never appears in output — it is not classified, not flagged, and not visible to the analyst. Should `runSession()` also run the reverse pass, flagging internal records with no external match? If so, what status should they carry?

### 2. How are bridge tables managed?

`TXT-04` normalization rules reference named bridge tables (`REF_TxType_Bridge`, `REF_Account_Bridge`, `REF_Security_Bridge`). Who creates and maintains them? Does Breakroom expose a bridge table management API, or are they managed via a separate admin interface? If a new custodian account is added mid-cycle and its mapping isn't yet in the bridge table, does the session fail with `normalization_error`, or does the record fall through as `unmatched`?

### 3. Which headers appear in the output?

The Confluence Data Modelling page states that the internal feed's headers are used as output column names. The flow diagram PDF shows custodian headers instead. `getSessionResults()` currently uses custodian headers in `external_record` and paired labels in `field_comparison`. What should the downloadable flat-file export use?

### 4. How is `org_id` handled in the composite key?

The flow diagram includes `org_id` (an Osyte system field not present in the CNB feed) as a composite key component. It has no corresponding CNB field and is not in the 15-field CNB schema. Is it injected by the system at session time (derived from `tenant_id`), or must the caller include it explicitly in the internal feed and composite key definition?