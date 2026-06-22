# Breakroom API Contract

## What Breakroom does

Enterprise clients reconcile custodian transaction feeds against Osyte's internal book of record daily. Today, ops analysts manually download two CSVs, paste them into Excel, write VLOOKUP formulas, and spend hours hunting down breaks that may turn out to be a date format mismatch or a sign convention difference. Breakroom replaces that with a deterministic matching engine.

**Breakroom solves two problems:**

1. **"Do these two feeds agree?"** — Given a configured reconciliation environment and two CSV feeds, run the full validation, normalization, and matching pipeline. Classify every custodian record as Auto-matched, Partial Match, Unmatched, or Duplicate.

2. **"How do I set up reconciliation against a new custodian?"** — Define both feed schemas, map fields between them, configure normalization and filter rules to handle format differences, and declare which fields form the composite matching key.

---

## Methods

Five methods across two APIs:

### Reconciliation Config API

Stateful. Manages the one-time configuration for a tenant–custodian reconciliation pair. Nothing is computed here — it is pure setup.

| # | Method name | Route | Solves | What it does |
|---|---|---|---|---|
| 1 | `createEnvironment()` | `POST /recon-config/createEnvironment` | Problem 2 | Creates a named reconciliation environment for a tenant, defines both input feed schemas, and establishes the field-to-field mapping between them |
| 2 | `setTransformationRules()` | `POST /recon-config/setTransformationRules` | Problem 2 | Configures filter rules (records to include or exclude before matching) and normalization rules (format standardization applied to the external feed) |
| 3 | `setMatchingConditions()` | `POST /recon-config/setMatchingConditions` | Problem 2 | Declares which mapped fields form the composite key and sets field-level tolerance rules for non-key comparison fields |

### Reconciliation Sessions API

Stateful. Runs reconciliation sessions against configured environments and returns results.

| # | Method name | Route | Solves | What it does |
|---|---|---|---|---|
| 4 | `runSession()` | `POST /recon-sessions/runSession` | Problem 1 | Uploads both feeds, runs the full validation and classification pipeline, and returns a session ID with a result summary |
| 5 | `getSessionResults()` | `GET /recon-sessions/getSessionResults/{session_id}` | Problem 1 | Returns classified records for a session, filterable by status |

---

## Assumptions

### 1. Configuration is complete before a session is run

An environment must have `createEnvironment()`, `setTransformationRules()`, and `setMatchingConditions()` all called before `runSession()` can be invoked against it. The environment holds the full ruleset — feed schemas, field mappings, filter rules, normalization rules, composite key, and tolerances.

**Why:** Separating configuration from processing keeps session runs lightweight (no schema or rule data in the request) and ensures all sessions against the same custodian use a consistent, auditable ruleset.

### 2. Normalization applies to the external (custodian) feed only

The internal Osyte feed is the source of truth. All format standardization — date parsing, case normalization, canonical value mapping, sign convention adjustment — runs against the external feed to align it with internal field formats before matching.

**Why:** Transforming the custodian data to match Osyte's format is the natural direction. Osyte controls the internal schema and it does not change per custodian. The external feed varies by custodian and needs to be brought into alignment.

### 3. Matching runs external → internal

Each external (custodian) record is looked up against the internal feed using the composite key. Internal records with no custodian counterpart are not classified in v1. See Open Questions.

**Why:** The custodian feed drives the reconciliation. Every custodian record must have an Osyte counterpart. The v1 scope is to validate that the custodian's view of trades matches Osyte's — not to audit whether every Osyte trade appears in the custodian feed.

### 4. Bridge tables for cross-system value mapping are caller-managed

The composite key maps Osyte integer fields (e.g. `fund_id`, `account_id`) to custodian text identifiers (e.g. `Account #`, `Security ID`). The resolution of that mapping — the bridge table that translates `fund_id = 12425` to `Account # = NYK-003640` — is managed by the caller and provided as a `TXT-04` normalization rule (canonical mapping). Breakroom does not maintain bridge tables natively.

**Why:** Bridge tables are tenant- and custodian-specific. The caller populates the mapping during environment configuration and updates it when account relationships change.

---

## Reconciliation Config API: `createEnvironment()`

**Route:** `POST /recon-config/createEnvironment`

**Purpose:** "How do I set up reconciliation between Osyte and a new custodian?"

Creates a named reconciliation environment for a tenant. Defines both input feed schemas (internal Osyte feed and external custodian feed) and establishes the field-to-field mapping between them. The internal feed's field labels become the output headers for session results.

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `environment_name` | string | yes | A human-readable name for this environment. Convention: `Osyte-{CustodianShortCode}`. E.g. `Osyte-CNB`. |
| `tenant_id` | string | yes | The tenant this environment belongs to. |
| `reconciliation_type` | string | no | Default: `trade_reconciliation`. v1 supports only `trade_reconciliation`. |
| `internal_feed` | object | yes | Feed definition for the Osyte internal data. Contains `feed_name` and `fields`. |
| `external_feed` | object | yes | Feed definition for the custodian data. Contains `feed_name` and `fields`. |
| `field_mapping` | object[] | yes | An array of pairs linking each internal mandatory field to its counterpart in the external feed. Only mandatory fields appear here. |

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

The mapping defines an exact-match expectation between the two fields, subject to normalization rules (configured via `setTransformationRules()`) and tolerance rules (configured via `setMatchingConditions()`).

### Example — Setting up Osyte ↔ City National Bank reconciliation for Common Fund

Common Fund holds positions custodied at City National Bank (CNB). CNB delivers a 15-column CSV daily. Osyte's internal feed has 81 columns, 6 mandatory. Nine fields are mapped — 4 will form the composite key, 5 are comparison fields.

**Request:**
```jsonc
POST /recon-config/createEnvironment

{
  "environment_name": "Osyte-CNB",
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
      {"label": "Account #",                "data_type": "text",    "format": "VARCHAR(20)",             "mandatory": true},
      {"label": "Primary Account Holder",   "data_type": "text",    "format": "VARCHAR(200)",            "mandatory": true},
      {"label": "Date (Entry)",             "data_type": "date",    "format": "MM/DD/YYYY or MM-DD-YYYY","mandatory": true},
      {"label": "Date (Trade)",             "data_type": "date",    "format": "MM/DD/YYYY or MM-DD-YYYY","mandatory": true},
      {"label": "Date (Settle)",            "data_type": "date",    "format": "MM/DD/YYYY",              "mandatory": true},
      {"label": "Transaction Type",         "data_type": "text",    "format": "VARCHAR(50)",             "mandatory": true},
      {"label": "Transaction Key/Mnemonic", "data_type": "text",    "format": "VARCHAR(20)",             "mandatory": true},
      {"label": "Quantity",                 "data_type": "numeric", "format": "DECIMAL(18,6)",           "mandatory": true},
      {"label": "Price",                    "data_type": "numeric", "format": "DECIMAL(18,4)",           "mandatory": true},
      {"label": "Net Amount",               "data_type": "numeric", "format": "DECIMAL(18,2)",           "mandatory": true},
      {"label": "Security ID",              "data_type": "text",    "format": "VARCHAR(20)",             "mandatory": true},
      {"label": "Description",              "data_type": "text",    "format": "VARCHAR(200)",            "mandatory": true},
      {"label": "Security Type Code",       "data_type": "text",    "format": "VARCHAR(30)",             "mandatory": true},
      {"label": "Cancel",                   "data_type": "text",    "format": "VARCHAR(3)",              "mandatory": true},
      {"label": "Reference #",              "data_type": "text",    "format": "VARCHAR(30)",             "mandatory": false}
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
  ]
}
```

Note: `fund_id` (Osyte integer) maps to `Account #` (CNB varchar) and `account_id` maps to `Security ID`. These are not direct type matches — `setTransformationRules()` must include `TXT-04` canonical mapping rules that resolve CNB identifiers to Osyte integer IDs via bridge tables.

**Response:**
```jsonc
{
  "environment_id": "env-cf-cnb-001",
  "environment_name": "Osyte-CNB",
  "tenant_id": "common-fund",
  "reconciliation_type": "trade_reconciliation",
  "status": "draft",
  "mapped_fields": 9,
  "internal_feed_id": "feed-osyte-cf-001",
  "external_feed_id": "feed-cnb-cf-001"
}
```

Status `draft` means configuration is not yet complete — `setTransformationRules()` and `setMatchingConditions()` must follow before `runSession()` can be called.

### `createEnvironment()` output signature

```jsonc
{
  "environment_id": "string",
  "environment_name": "string",
  "tenant_id": "string",
  "reconciliation_type": "string",
  "status": "draft | active",
  "mapped_fields": "int",
  "internal_feed_id": "string",
  "external_feed_id": "string"
}
```

---

## Reconciliation Config API: `setTransformationRules()`

**Route:** `POST /recon-config/setTransformationRules`

**Purpose:** "How do I handle the format differences between Osyte and this custodian?"

Configures two rule sets on an environment: filter rules that include or exclude records before matching, and normalization rules that standardize external feed values to align with the internal feed's formats. Both rule sets apply to the external feed only. Filter rules execute first.

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `environment_id` | string | yes | The environment to configure. |
| `filter_rules` | object[] | no | Records to include or exclude from processing. See fields below. |
| `normalization_rules` | object[] | no | Format standardization rules applied to the external feed before matching. See fields below. |

**`filter_rules` — each rule object:**

| Field | Type | What it means |
|---|---|---|
| `feed` | string | Which feed to filter: `internal` or `external`. |
| `field_label` | string | The field to evaluate. |
| `action` | string | `include` (keep only records matching the condition) or `exclude` (drop records matching the condition). |
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

### Example — Configuring CNB normalization: mixed date formats, free-text transaction types, and sign conventions

CNB sends `Transaction Type` as free text ("Buy", "Sell"). Osyte stores `trade_transaction_type_cd` as coded values ("BTO", "STC"). CNB sends `Quantity` as negative for sells; Osyte stores `final_quantity` as always positive. CNB uses `MM/DD/YYYY` and `MM-DD-YYYY` interchangeably.

**Request:**
```jsonc
POST /recon-config/setTransformationRules

{
  "environment_id": "env-cf-cnb-001",
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
  ]
}
```

```
Filter rules run first:
  External: keep only Buy/Sell-type transactions; drop cancelled records (Cancel = Yes).
  Internal: drop status_id 15 (Cancelled) and 16 (Completed) — only pending trades reconcile.

Normalization chain for a CNB record with Transaction Type = "buy":
  TXT-01: "buy" → "BUY"
  TXT-02: no-op (no whitespace)
  TXT-04: "BUY" → "BTO" via REF_TxType_Bridge
  Result: "BTO" — now matches Osyte's trade_transaction_type_cd

Normalization chain for CNB Quantity = -214803.848 (a sell):
  NUM-01: already 6dp — no change
  NUM-03: -214803.848 → 214803.848 (absolute value)
  Result: 214803.848 — matches Osyte's final_quantity (always positive)

Normalization chain for CNB Date (Trade) = "06-12-2025":
  DT-02: no time component — no-op
  DT-03: "06-12-2025" → "2025-06-12"
  Result: "2025-06-12" — matches Osyte's trade_dt after internal ISO conversion
```

**Response:**
```jsonc
{
  "environment_id": "env-cf-cnb-001",
  "filter_rules_set": 3,
  "normalization_rules_set": 12,
  "status": "draft"
}
```

### `setTransformationRules()` output signature

```jsonc
{
  "environment_id": "string",
  "filter_rules_set": "int",
  "normalization_rules_set": "int",
  "status": "draft | active"
}
```

---

## Reconciliation Config API: `setMatchingConditions()`

**Route:** `POST /recon-config/setMatchingConditions`

**Purpose:** "Which fields identify a record, and how much variance is acceptable?"

Declares which mapped fields form the composite key (used for record lookup) and sets field-level tolerance rules for non-key comparison fields. Calling this method marks the environment `active` — `runSession()` can be called once status is `active`.

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `environment_id` | string | yes | The environment to finalize. |
| `composite_key` | string[] | yes | Labels of internal feed fields (from `field_mapping`) that together form the unique record identifier. A single field is a simple key; multiple fields form a composite key. |
| `tolerance_rules` | object[] | no | Acceptable variance on non-key comparison fields. Fields with no tolerance rule require an exact match after normalization. |

**`tolerance_rules` — each rule object:**

| Field | Type | What it means |
|---|---|---|
| `field_label` | string | Internal feed label of the comparison field. Must be a mapped field. |
| `tolerance_type` | string | `absolute` (fixed numeric difference), `percentage` (relative to the internal value), or `business_days` (calendar tolerance for date fields). |
| `tolerance_value` | float | The permitted deviation. E.g. `1` for `business_days`, `0.0001` for `percentage`, `0.01` for `absolute`. |

### Example — Declaring the Osyte-CNB composite key and tolerances

**Request:**
```jsonc
POST /recon-config/setMatchingConditions

{
  "environment_id": "env-cf-cnb-001",
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
trade_entered_dt: ±1 business day. Custodian posts entry date the next morning — a 1bd window
  prevents a false Partial Match on what is functionally the same instruction.

avg_price_per_share: 0.0001% tolerance. Covers floating-point precision differences after
  4-decimal standardization via NUM-01/NUM-02. E.g. 257.6116 vs 257.6117 → within tolerance.

net_cash_amt: ±$0.01 absolute. Covers penny-rounding differences in commission or fee treatment.

final_quantity: exact match (0 tolerance). Quantity differences are material breaks.

settlement_dt: exact match (0 business day tolerance). Settlement date differences are material.
```

**Response:**
```jsonc
{
  "environment_id": "env-cf-cnb-001",
  "composite_key": ["fund_id", "account_id", "trade_dt", "trade_transaction_type_cd"],
  "tolerance_rules_set": 5,
  "status": "active"
}
```

Status transitions to `active` — the environment is ready to receive sessions.

### `setMatchingConditions()` output signature

```jsonc
{
  "environment_id": "string",
  "composite_key": ["string"],
  "tolerance_rules_set": "int",
  "status": "active"
}
```

---

## Reconciliation Sessions API: `runSession()`

**Route:** `POST /recon-sessions/runSession`

**Purpose:** "Do these two feeds agree?"

Uploads the internal and external CSV feeds for a given environment and runs the full processing pipeline: four-step basic validation, normalization service, duplicate check, composite key matching, and field-level comparison. Returns a session ID and summary. Individual record results are retrieved via `getSessionResults()`.

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `environment_id` | string | yes | The active environment to run against. |
| `internal_feed_data` | string | yes | Base64-encoded CSV content for the internal (Osyte) feed. Must conform to the feed schema defined in `createEnvironment()`. |
| `external_feed_data` | string | yes | Base64-encoded CSV content for the external (custodian) feed. Must conform to the external feed schema. |

Both feeds are passed as base64-encoded strings. For large files, multipart/form-data upload is also accepted — use field names `internal_feed_data` and `external_feed_data`.

### Processing pipeline

The engine runs four sequential validation checks on each feed before proceeding to matching. Any failure stops the session:

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

Four Osyte trades vs. four CNB records. Two match exactly. One has a price difference beyond tolerance (partial match). One CNB record resolves to no known Osyte account (unmatched).

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
fund_id, account_id, trade_transaction_type_cd, trade_dt,   trade_entered_dt,    settlement_dt, final_quantity, avg_price_per_share, net_cash_amt
12425,   234644,     BTO,                        5/22/2026,  5/15/2026 4:51,      5/26/2026,     214803.848,     257.6500,            55344211.4500
12425,   234644,     STC,                        5/22/2026,  5/15/2026 9:30,      5/26/2026,     100000.000,     150.2500,            -15025000.0000
12425,   235001,     BTO,                        5/22/2026,  5/16/2026 10:00,     5/26/2026,     50000.000,      100.0000,            5000000.0000
12425,   234888,     STC,                        5/22/2026,  5/17/2026 11:00,     5/26/2026,     75000.000,      200.0000,            -15000000.0000
```

External feed (decoded, 4 records):
```
Account #,   Date (Trade), Transaction Type, Quantity,    Price,    Net Amount,    Security ID, Cancel
NYK-003640,  5/22/2026,    buy,              214803.848,  257.6500, 55344211.45,   AVGO,        No
NYK-003640,  5/22/2026,    Sell,             -100000,     150.2500, -15025000.00,  AAPL,        No
NYK-003641,  5/22/2026,    Buy,              50000,       100.0500, 5002500.00,    MSFT,        No
NYK-003999,  5/22/2026,    Sell,             30000,       180.0000, -5400000.00,   TSLA,        No
```

```
Normalization (external feed):
  TXT-01: "buy" → "BUY", "Sell" → "SELL", "Buy" → "BUY"
  TXT-04: "BUY" → "BTO", "SELL" → "STC" via REF_TxType_Bridge
  TXT-04: "NYK-003640" → 12425 / "AVGO" → 234644 via REF_Account and REF_Security bridges
  NUM-01: 100.05 → 100.0500
  NUM-03: -100000 → 100000 (Quantity sign removed for sells)
  DT-03:  "5/22/2026" → "2026-05-22"

Duplicate check: no duplicate composite keys in external feed.

Key matching and classification (external → internal):
  Record 1: key (12425, 234644, 2026-05-22, BTO) → match found.
    price 257.6500 = 257.6500 ✓, quantity 214803.848 = 214803.848 ✓,
    net_cash_amt 55344211.45 ≈ 55344211.4500 (within $0.01) ✓ → AUTO_MATCHED.
  Record 2: key (12425, 234644, 2026-05-22, STC) → match found. All fields within tolerance → AUTO_MATCHED.
  Record 3: key (12425, 235001, 2026-05-22, BTO) → match found.
    Price: 100.0500 (CNB) vs 100.0000 (Osyte) → diff 0.05%. Tolerance: 0.0001%. Exceeds → PARTIAL_MATCH.
  Record 4: "NYK-003999" resolves to no known fund_id in REF_Account_Bridge → no key match → UNMATCHED.
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

Returns record-level results for a session. Each record includes its assigned status and field-level comparison details. For matched records, includes a reference to the matched internal record. Filterable by status. Paginated.

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
        {"field": "fund_id / Account #",              "internal": "12425",          "external_normalized": "12425",        "match": true,  "tolerance_applied": null},
        {"field": "account_id / Security ID",         "internal": "235001",         "external_normalized": "235001",       "match": true,  "tolerance_applied": null},
        {"field": "trade_dt / Date (Trade)",          "internal": "2026-05-22",     "external_normalized": "2026-05-22",   "match": true,  "tolerance_applied": null},
        {"field": "trade_transaction_type_cd / Type", "internal": "BTO",            "external_normalized": "BTO",          "match": true,  "tolerance_applied": null},
        {"field": "avg_price_per_share / Price",      "internal": "100.0000",       "external_normalized": "100.0500",     "match": false, "tolerance_applied": {"type": "percentage",    "limit": 0.0001, "actual": 0.0005}},
        {"field": "net_cash_amt / Net Amount",        "internal": "5000000.0000",   "external_normalized": "5002500.0000", "match": false, "tolerance_applied": {"type": "absolute",      "limit": 0.01,   "actual": 2500.00}},
        {"field": "final_quantity / Quantity",        "internal": "50000.000000",   "external_normalized": "50000.000000", "match": true,  "tolerance_applied": null},
        {"field": "trade_entered_dt / Date (Entry)",  "internal": "2026-05-16",     "external_normalized": "2026-05-16",   "match": true,  "tolerance_applied": {"type": "business_days", "limit": 1, "actual": 0}},
        {"field": "settlement_dt / Date (Settle)",    "internal": "2026-05-26",     "external_normalized": "2026-05-26",   "match": true,  "tolerance_applied": null}
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
| `environment_not_found` | 404 | No environment found for the given `environment_id` | `setTransformationRules`, `setMatchingConditions`, `runSession` |
| `environment_not_active` | 400 | `runSession()` called on a `draft` environment — configuration not complete | `runSession` |
| `feed_format_error` | 422 | Uploaded feed is not a valid CSV | `runSession` |
| `missing_feed_fields` | 422 | One or more mandatory fields are absent from the uploaded feed | `runSession` |
| `data_type_mismatch` | 422 | A field value does not match its configured `data_type` | `runSession` |
| `normalization_error` | 422 | A normalization rule failed to execute (e.g. bridge table not found, unparseable date) | `runSession` |
| `session_not_found` | 404 | No session found for the given `session_id` | `getSessionResults` |
| `session_failed` | 400 | `getSessionResults()` called on a session that did not complete successfully | `getSessionResults` |

---

## Open Questions

### 1. Should matching run in both directions?

The engine currently matches external → internal: each custodian record is looked up against the Osyte feed. This means an Osyte trade with no custodian counterpart never appears in the reconciliation output — it is not classified, not flagged, and not visible to the analyst.

This is a meaningful gap: Osyte may have booked a trade the custodian hasn't settled yet, or a custodian feed delivery failure may have dropped records. Should `runSession()` include a flag that also runs the reverse pass, flagging internal records with no external match? If so, what status should they carry — a new `internal_only` status, or reuse `unmatched`?

### 2. How are bridge tables managed?

`TXT-04` normalization rules reference named bridge tables (`REF_TxType_Bridge`, `REF_Account_Bridge`, `REF_Security_Bridge`). The contract assumes these exist and are populated before `runSession()` is called. Who creates and maintains them? Does Breakroom expose a bridge table management API, or are they managed via a separate admin interface? What happens when a new custodian account is added mid-cycle and its mapping isn't yet in the bridge table — does the session fail with `normalization_error`, or does the record fall through as `unmatched`?

### 3. Which headers appear in the output CSV?

The Confluence Data Modelling page states that the internal feed's headers are used as output column names. The flow diagram PDF shows the custodian feed's headers used instead. `getSessionResults()` currently returns both (the `external_record` block uses custodian headers; `field_comparison` uses the `internal / external` pair). What should the downloadable flat-file export format use?

### 4. How is `org_id` handled in the composite key?

The flow diagram includes `org_id` (an Osyte system field, not present in the custodian feed) as part of the composite key mapping for the CNB implementation. `org_id` has no corresponding CNB field and is not in the 15-field CNB schema. Is it injected by the system at session time (derived from `tenant_id`), or must the caller pass it explicitly? The current contract omits it from the composite key definition — this needs to be resolved before the CNB configuration is considered complete.
