# Breakroom API Contract

## What Breakroom does

Enterprise clients reconcile custodian transaction feeds against Osyte's internal book of record daily. Today, ops analysts manually download two CSVs, paste them into Excel, write VLOOKUP formulas, and spend hours hunting down breaks that may turn out to be a date format mismatch or a sign convention difference. Breakroom replaces that with a deterministic matching engine.

**Breakroom solves two problems:**

1. **"How do I set up reconciliation against a new custodian?"** — In a single call, provide both feed schemas, the field mapping between them, filter and normalization rules to handle format differences, and the composite key and tolerance rules for matching. Breakroom stores all of this as a named environment. Every subsequent daily run references the environment by ID — no ruleset resent, no config drift.

2. **"Do these two feeds agree?"** — Given a configured reconciliation environment and two CSV feeds, run the full validation, normalization, and matching pipeline. Classify every custodian record as Auto-matched, Partial Match, Unmatched, or Duplicate, flag Osyte records the custodian never reported as Internal-only, and report records dropped by filter rules.

---

## Methods

Two methods on a single Breakroom API:

| # | Method name | Route | Solves | What it does |
|---|---|---|---|---|
| 1 | `createEnvironment()` | `POST /breakroom/createEnvironment` | Problem 1 | Creates a reconciliation environment in a single call with everything needed to run reconciliations against a custodian: both feed schemas, field mapping, filter rules, normalization rules, composite key, and tolerances |
| 2 | `runReconciliation()` | `POST /breakroom/runReconciliation` | Problem 2 | Uploads both feeds, runs the full pipeline (basic validation → filtering → normalization → record validation → classification), and returns the complete JSON result the client uses to render the output file (CSV in v1; Excel later) |

---

## Assumptions

### 1. Configuration is set once, reconciliations run daily

`createEnvironment()` is called once per custodian relationship. Every `runReconciliation()` call references the environment by ID — feed schemas, field mapping, filter rules, normalization rules, composite key, and tolerances are all stored server-side. The caller sends only the two CSV files per run.

**Why:** Resending the full config with every daily run is wasteful and breaks auditability. Storing config separately means every reconciliation can be traced to the exact ruleset it ran against. If a custodian changes their feed format, the caller creates a new environment — old reconciliations stay linked to the old config.

### 2. Reconciliation runs synchronously and returns the complete result

`runReconciliation()` is a single round-trip. The caller uploads both feeds, the server runs the full pipeline (basic validation, filtering, normalization, record validation, classification), and returns the summary, validation details, and per-record results in one response.

**Why:** The caller is waiting for the reconciliation result regardless — there's no benefit to splitting the trigger and the read into two calls. Combining them is one round-trip instead of two, no state to track between calls, and no scenario where you'd want the results without also triggering the run.

### 3. Normalization applies to the external (custodian) feed only

The internal Osyte feed is the source of truth. All format standardization runs against the external feed to align it with internal field formats before matching.

**Why:** Osyte controls the internal schema; it does not change per custodian. The external feed varies by custodian and needs to be brought into alignment.

### 4. Matching runs in both directions

The engine runs two passes. The first pass matches each external (custodian) record against the internal feed using the composite key — classifying records as `auto_matched`, `partial_match`, `unmatched`, or `duplicate`. The second pass checks each internal record to see if any external record matched to it — records with no external match are classified as `internal_only`.

**Why:** A one-way pass only catches what the custodian sent that Osyte doesn't recognise. The reverse pass catches what Osyte booked that the custodian never reported — a different and equally important class of break. In both passes, the composite key of each record is looked up against every record in the other feed.

### 5. Reference tables for cross-system value mapping are caller-managed

The composite key maps Osyte integer fields (e.g. `fund_id`, `account_id`) to custodian text identifiers (e.g. `Account #`, `Security ID`). The resolution of that mapping is provided as a `TXT-04` normalization rule referencing a named reference table. Breakroom does not maintain reference tables natively.

**Why:** Reference tables are tenant- and custodian-specific. The caller populates them during environment setup and updates them when account relationships change.

### 6. The API returns JSON; the client renders the output file

`runReconciliation()` returns a structured JSON response. The client uses this response to render the user-facing output file — CSV in v1, Excel in later phases. Breakroom never produces the output file itself.

**Why:** Keeping the API response format-agnostic means the same response can drive the UI, downstream automation, and any future export format without requiring a contract change. The server has no business owning file storage or template lifecycle for what is fundamentally a presentation concern.

---

## Milestone alignment

This contract supports the 3-milestone delivery plan. Both methods exist from M1 onward; functionality fills in by milestone:

| Milestone | `createEnvironment()` fields used | `runReconciliation()` pipeline stages active | Response sections populated |
|---|---|---|---|
| **M1** | `internal_feed`, `external_feed`, `field_mapping` | Basic validation (checks 1–5) | `summary` (totals only), `basic_validation` |
| **M2** | + `filter_rules`, `normalization_rules` | + Normalization service (check 6) | + `summary.filtered`, extended `basic_validation` |
| **M3** | + `composite_key`, `tolerance_rules` | + Record validation (duplicate check, key matching, field comparison, classification, reverse pass) | + `records` |

In M1 and M2, `composite_key` and `tolerance_rules` may be omitted in `createEnvironment()` and the `records` array in the response will be empty. The contract structure is stable across all three milestones — only the populated sections grow.

---

## `createEnvironment()`

**Route:** `POST /breakroom/createEnvironment`

**Purpose:** "Set up everything Breakroom needs to reconcile Osyte against this custodian."

Creates a named reconciliation environment in a single call. The request contains the complete configuration: both feed schemas, the field mapping between them, filter rules (applicable to both feeds) and normalization rules (external feed only), the composite key definition, and tolerance rules for comparison fields. On success the environment is immediately active and ready to receive reconciliation runs.

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `environment_name` | string | yes | A human-readable name for this environment. Convention: `Osyte-{CustodianShortCode}-Reconciliation`. E.g. `Osyte-CNB-Reconciliation`. |
| `tenant_id` | string | yes | The tenant this environment belongs to. |
| `custodian_id` | string | yes | The custodian this environment reconciles against. Breakroom uses this to look up the custodian's `org_id` from its internal registry and injects it automatically into every composite key lookup at run time. The caller never handles `org_id` directly. |
| `reconciliation_type` | string | no | Default: `trade_reconciliation`. v1 supports only `trade_reconciliation`. |
| `internal_feed` | object | yes | Feed definition for the Osyte internal data. Contains `feed_name` and `fields`. |
| `external_feed` | object | yes | Feed definition for the custodian data. Contains `feed_name` and `fields`. |
| `field_mapping` | object[] | yes | Pairs linking each internal mandatory field to its counterpart in the external feed. Only mandatory fields appear here. |
| `filter_rules` | object[] | no | Records to include or exclude before matching. Applied before normalization. |
| `normalization_rules` | object[] | no | Format standardization rules applied to the external feed before matching. |
| `composite_key` | string[] | conditional | Labels of internal feed fields that together form the unique record identifier. Required for M3. |
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
| `rule_name` | string | A caller-supplied identifier for this rule. Surfaced in the response so filtered records can be attributed to a named rule. E.g. `exclude_cancelled`. |
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
| `parameters` | object | Rule-specific parameters. Only `TXT-04` requires a parameter (`reference_table`). |

To apply an additional normalization rule to a field, add a new entry to this array referencing an existing `rule_id`. New `rule_id` values beyond the catalog below require a Breakroom engineering change and are not configurable by the caller.

**Normalization rule catalog:**

| Rule ID | Category | What it does |
|---|---|---|
| `TXT-01` | Text | Convert all values to upper case |
| `TXT-02` | Text | Trim leading/trailing spaces; collapse internal multiples |
| `TXT-03` | Text | Remove non-meaningful separators (`-`, `/`, `.`, `_`) unless part of a structural identifier |
| `TXT-04` | Text | Map different labels to a canonical value via a named reference table (e.g. `"buy"`, `"Purchase"`, `"B"` → `"BTO"`) |
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
POST /breakroom/createEnvironment

{
  "environment_name": "Osyte-CNB-Reconciliation",
  "tenant_id": "common-fund",
  "custodian_id": "cnb",
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
      "rule_name": "include_buy_sell_only",
      "feed": "external",
      "field_label": "Transaction Type",
      "action": "include",
      "operator": "in",
      "values": ["Buy", "Sell", "BUY", "SELL", "Purchase", "BTO", "STC"]
    },
    {
      "rule_name": "exclude_cancelled",
      "feed": "external",
      "field_label": "Cancel",
      "action": "exclude",
      "operator": "equals",
      "values": ["Yes"]
    },
    {
      "rule_name": "exclude_completed_internal",
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
    {"rule_id": "TXT-04", "target_fields": ["Transaction Type"], "parameters": {"reference_table": "REF_TxType"}},
    {"rule_id": "TXT-04", "target_fields": ["Account #"],        "parameters": {"reference_table": "REF_Account"}},
    {"rule_id": "TXT-04", "target_fields": ["Security ID"],      "parameters": {"reference_table": "REF_Security"}},
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

**Response:**
```jsonc
{
  "environment_id": "env-cf-cnb-001",
  "environment_name": "Osyte-CNB-Reconciliation",
  "tenant_id": "common-fund",
  "custodian_id": "cnb",
  "reconciliation_type": "trade_reconciliation",
  "status": "active",
  "mapped_fields": 9,
  "composite_key": ["fund_id", "account_id", "trade_dt", "trade_transaction_type_cd"],
  "tolerance_rules_set": 5
}
```

The environment is immediately `active` — `runReconciliation()` can be called as soon as this response is received.

### `createEnvironment()` output signature

```jsonc
{
  "environment_id": "string",
  "environment_name": "string",
  "tenant_id": "string",
  "custodian_id": "string",
  "reconciliation_type": "string",
  "status": "active",
  "mapped_fields": "int",
  "composite_key": ["string"] | null,
  "tolerance_rules_set": "int"
}
```

---


## `runReconciliation()`

**Route:** `POST /breakroom/runReconciliation`

**Purpose:** "Do these two feeds agree?"

Uploads the internal and external CSV feeds against a configured environment and runs the full processing pipeline in a single synchronous call. Returns the complete result — reconciliation ID, summary counts, per-feed basic validation details, and per-record classification — as JSON, which the client renders into the user-facing output file.

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `environment_id` | string | yes | The active environment to run against. |
| `internal_feed_data` | string | yes | Base64-encoded CSV content for the internal (Osyte) feed. Must conform to the feed schema defined in `createEnvironment()`. |
| `external_feed_data` | string | yes | Base64-encoded CSV content for the external (custodian) feed. Must conform to the external feed schema. |

For large files, multipart/form-data upload is also accepted using the same field names.

### Processing pipeline

The engine runs the basic validation checks on each feed. M1 runs checks 1–5; check 6 (Feed Formatting Service Check) is added in M2 once normalization rules are introduced. Any failure stops the run for that feed and the overall status is `failed`:

| Step | Check | Available from | Failure code |
|---|---|---|---|
| 1 | Feed received check — confirms the feed was received and is non-empty | M1 | `feed_not_received` |
| 2 | Feed format check — confirms CSV structure is well-formed | M1 | `feed_format_error` |
| 3 | Feed file check — confirms the file is readable and not corrupted | M1 | `feed_file_error` |
| 4 | Feed field check — confirms all mandatory fields are present | M1 | `missing_feed_fields` |
| 5 | Feed field data type check — confirms field values match configured data types | M1 | `data_type_mismatch` |
| 6 | Feed formatting service check — applies filter rules (both feeds), then normalization rules (external feed) | M2 | `normalization_error` |

Within check 6, filter rules run first and apply to whichever feed each rule specifies (`internal` or `external`) — a filter is never restricted to one side. Normalization rules run second and apply to the external feed only, transforming the records that survived filtering.

If all configured checks pass on both feeds, record validation begins:
1. Each remaining external record is assigned a system-generated `record_id`. Internal records that surface as `internal_only` in the reverse pass are also assigned a `record_id` at that stage. `record_id` values are globally unique within a single reconciliation across both feeds.
2. **Duplicate check:** records sharing the same composite key are compared field-by-field. Exact duplicates → `duplicate`. Unique records among those sharing a key are prioritized and passed through.
3. **External → internal key lookup:** the composite key of each external record is looked up against the internal feed. No match → `unmatched`. Match found → proceed to field comparison.
4. **Field comparison:** compare all mapped non-key fields within tolerance. Any field exceeds tolerance → `partial_match`. All fields within tolerance → `auto_matched`.
5. **Internal → external reverse pass:** each internal record is checked for whether any external record matched to it. Internal records with no external match → `internal_only`. These represent Osyte trades the custodian did not report.

Auto-matched records trigger a write-back to the internal record's `reconciliation_status_code` field (set to `AUTO_MATCH`). No other status (`partial_match`, `unmatched`, `duplicate`, `internal_only`) triggers a write-back in v1 — those records require analyst review before any internal state changes.

### Example — End-of-day reconciliation, Common Fund / CNB

Four Osyte trades vs. four CNB records. Two match exactly. One has a price difference beyond tolerance. One CNB record resolves to no known Osyte account. One Osyte record has no custodian counterpart (internal-only).

**Request:**
```jsonc
POST /breakroom/runReconciliation

{
  "environment_id": "env-cf-cnb-001",
  "internal_feed_data": "ZnVuZF9pZCxhY2NvdW50X2lkLHRyYWRlX3RyYW5zYWN0aW9uX3R5cGVfY2QsdHJhZGVfYW10...",
  "external_feed_data": "QWNjb3VudCAjLFByaW1hcnkgQWNjb3VudCBIb2xkZXIsTG9nIERhdGUsd..."
}
```

Internal feed (decoded, 4 records):
```
fund_id, account_id, trade_transaction_type_cd, trade_dt,   trade_entered_dt,  settlement_dt, final_quantity, avg_price_per_share, net_cash_amt
12425,   234644,     BTO,                        5/22/2026,  5/15/2026 4:51,    5/26/2026,     214803.848,     257.6500,            55344211.4500
12425,   234644,     STC,                        5/22/2026,  5/15/2026 9:30,    5/26/2026,     100000.000,     150.2500,            -15025000.0000
12425,   235001,     BTO,                        5/22/2026,  5/16/2026 10:00,   5/26/2026,     50000.000,      100.0000,            5000000.0000
12425,   234888,     STC,                        5/22/2026,  5/17/2026 11:00,   5/26/2026,     75000.000,      200.0000,            -15000000.0000
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
Filter rules: 
  External — all 4 records pass include_buy_sell_only; none are cancelled → 0 excluded.
  Internal — no records have trade_item_status_id 15 or 16 → 0 excluded.

Normalization:
  TXT-01/TXT-04: "buy" → "BUY" → "BTO", "Sell"/"Buy" → "STC"/"BTO"
  TXT-04: NYK-003640 → 12425, AVGO → 234644, MSFT → 235001 via reference tables
  NUM-03: -100000 → 100000 (Quantity sign removed for sells)
  DT-03:  "5/22/2026" → "2026-05-22"

Duplicate check: no duplicate composite keys.

Classification:
  Record 1: key (12425, 234644, 2026-05-22, BTO) → match. All fields within tolerance → AUTO_MATCHED.
  Record 2: key (12425, 234644, 2026-05-22, STC) → match. All fields within tolerance → AUTO_MATCHED.
  Record 3: key (12425, 235001, 2026-05-22, BTO) → match. Price 100.0500 vs 100.0000 →
    diff 0.05%, tolerance 0.0001% → exceeds → PARTIAL_MATCH.
  Record 4: NYK-003999 resolves to no fund_id in REF_Account → no key match → UNMATCHED.

Reverse pass (internal → external):
  Internal record 610992 (fund_id=12425, account_id=234888, STC, 5/22/2026) → no external record
    matched to it → INTERNAL_ONLY. Osyte has this trade; CNB did not report it.
```

**Response:**
```jsonc
{
  "reconciliation_id": "REC-20260617-001",
  "environment_id": "env-cf-cnb-001",
  "environment_name": "Osyte-CNB-Reconciliation",
  "reconciliation_type": "trade_reconciliation",
  "status": "completed",
  "start_date": "2026-06-17T09:30:00Z",
  "end_date": "2026-06-17T09:30:42Z",

  "summary": {
    "external_records": 4,
    "auto_matched": 2,
    "partial_match": 1,
    "unmatched": 1,
    "duplicate": 0,
    "filtered": 0,
    "internal_only": 1
  },

  "basic_validation": [
    {
      "feed_type": "custodian",
      "feed_name": "CNB Records",
      "received_date": "2026-06-17T09:30:00Z",
      "processed_date": "2026-06-17T09:30:05Z",
      "feed_status": "completed",
      "checks": [
        {"sno": 1, "validation": "Feed Received Check",          "status": "passed"},
        {"sno": 2, "validation": "Feed Format Check",            "status": "passed"},
        {"sno": 3, "validation": "Feed File Check",              "status": "passed"},
        {"sno": 4, "validation": "Feed Field Check",             "status": "passed"},
        {"sno": 5, "validation": "Feed Field Data Type Check",   "status": "passed"},
        {"sno": 6, "validation": "Feed Formatting Service Check","status": "passed"}
      ]
    },
    {
      "feed_type": "internal",
      "feed_name": "Osyte Records",
      "received_date": "2026-06-17T09:30:00Z",
      "processed_date": "2026-06-17T09:30:05Z",
      "feed_status": "completed",
      "checks": [
        {"sno": 1, "validation": "Feed Received Check",          "status": "passed"},
        {"sno": 2, "validation": "Feed Format Check",            "status": "passed"},
        {"sno": 3, "validation": "Feed File Check",              "status": "passed"},
        {"sno": 4, "validation": "Feed Field Check",             "status": "passed"},
        {"sno": 5, "validation": "Feed Field Data Type Check",   "status": "passed"},
        {"sno": 6, "validation": "Feed Formatting Service Check","status": "passed"}
      ]
    }
  ],

  "records": [
    {
      "record_id": "REC-001",
      "reconciliation_status": "auto_matched",
      "external_record": {
        "Account #": "NYK-003640", "Date (Trade)": "5/22/2026", "Transaction Type": "buy",
        "Quantity": 214803.848, "Price": 257.6500, "Net Amount": 55344211.45,
        "Security ID": "AVGO", "Cancel": "No"
      },
      "matched_internal_record_ref": "610989",
      "field_comparison": [
        // all fields match within tolerance
      ]
    },
    {
      "record_id": "REC-002",
      "reconciliation_status": "auto_matched",
      "external_record": { /* ... */ },
      "matched_internal_record_ref": "610990",
      "field_comparison": [ /* ... */ ]
    },
    {
      "record_id": "REC-003",
      "reconciliation_status": "partial_match",
      "external_record": {
        "Account #": "NYK-003641", "Date (Trade)": "5/22/2026", "Transaction Type": "Buy",
        "Quantity": 50000, "Price": 100.05, "Net Amount": 5002500.00, "Security ID": "MSFT"
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
        {"field": "trade_entered_dt / Date (Entry)",  "internal": "2026-05-16",    "external_normalized": "2026-05-16",   "match": true,  "tolerance_applied": {"type": "business_days", "limit": 1,      "actual": 0}}
      ]
    },
    {
      "record_id": "REC-004",
      "reconciliation_status": "unmatched",
      "external_record": {
        "Account #": "NYK-003999", "Date (Trade)": "5/22/2026", "Transaction Type": "Sell",
        "Quantity": 30000, "Price": 180.0000, "Net Amount": -5400000.00, "Security ID": "TSLA"
      },
      "matched_internal_record_ref": null,
      "field_comparison": null
    },
    {
      "record_id": "REC-005",
      "reconciliation_status": "internal_only",
      "internal_record": {
        "fund_id": 12425, "account_id": 234888, "trade_transaction_type_cd": "STC",
        "trade_dt": "5/22/2026", "final_quantity": 75000, "avg_price_per_share": 200.0000,
        "net_cash_amt": -15000000.0000
      },
      "external_record": null,
      "field_comparison": null
    }
  ]
}
```

Note on duplicate records: when a record is classified as `duplicate`, `matched_internal_record_ref` still carries the internal record reference if a key match was found — duplicate classification means another external record shares this composite key, not that no internal counterpart exists. This lets analysts see what the duplicate was duplicating against.

### `runReconciliation()` output signature

```jsonc
{
  "reconciliation_id": "string",
  "environment_id": "string",
  "environment_name": "string",
  "reconciliation_type": "string",
  "status": "completed | failed",
  "start_date": "ISO 8601 timestamp",
  "end_date": "ISO 8601 timestamp",

  "summary": {
    "external_records": "int",
    "auto_matched": "int",
    "partial_match": "int",
    "unmatched": "int",
    "duplicate": "int",
    "filtered": "int",
    "internal_only": "int"
  },

  "basic_validation": [
    {
      "feed_type": "custodian | internal",
      "feed_name": "string",
      "received_date": "ISO 8601 timestamp",
      "processed_date": "ISO 8601 timestamp",
      "feed_status": "completed | failed",
      "checks": [
        {
          "sno": "int",
          "validation": "string",
          "status": "passed | failed",
          "break_type": "string | null",
          "break_description": "string | null"
        }
      ]
    }
  ],

  "records": [
    {
      "record_id": "string",
      "reconciliation_status": "auto_matched | partial_match | unmatched | duplicate | internal_only",
      "external_record": "object (custodian field values as received) | null",
      "internal_record": "object (Osyte field values) | null",
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

If status is `failed`, the `basic_validation` block is populated and `records` is empty. The first failed check is the termination point — subsequent checks did not run. `matched_internal_record_ref` is populated for `auto_matched`, `partial_match`, and `duplicate` records, and is `null` for `unmatched` and `internal_only`. `field_comparison` is populated for `auto_matched`, `partial_match`, and `duplicate` records, and is `null` for `unmatched` and `internal_only`. `internal_only` records carry `internal_record` and `null` for `external_record` and `field_comparison`. Filtered records are counted in `summary.filtered` but do not appear in the `records` array.

**Summary count invariants:**
- `external_records` is the total count of records received in the custodian feed (after basic validation, before filtering). It equals the sum of `auto_matched + partial_match + unmatched + duplicate + filtered`.
- `internal_only` is a separate count from the internal feed and is not part of `external_records`. The two summaries cover different feeds and should not be added together.

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
| `environment_not_found` | 404 | No environment found for the given `environment_id` | `runReconciliation` |
| `feed_not_received` | 422 | A feed was not received or is empty | `runReconciliation` |
| `feed_format_error` | 422 | Uploaded feed is not a valid CSV | `runReconciliation` |
| `feed_file_error` | 422 | The file is unreadable or corrupted | `runReconciliation` |
| `missing_feed_fields` | 422 | One or more mandatory fields are absent from the uploaded feed | `runReconciliation` |
| `data_type_mismatch` | 422 | A field value does not match its configured `data_type` | `runReconciliation` |
| `normalization_error` | 422 | A normalization rule failed to execute (e.g. reference table not found, unparseable date) | `runReconciliation` |

Note: when a basic validation check fails, the response body still returns the full structure with `status: failed` and the specific check marked `failed` in `basic_validation`. The HTTP-level error response above is for request-level rejections (malformed JSON, missing required parameters, unknown environment).

---

## Open Questions

### 1. How are reference tables managed?

`TXT-04` normalization rules reference named reference tables (`REF_TxType`, `REF_Account`, `REF_Security`). Who creates and maintains them? Does Breakroom expose a reference table management API, or are they managed via a separate admin interface? If a new custodian account is added mid-cycle and its mapping isn't yet in the reference table, does the reconciliation fail with `normalization_error`, or does the record fall through as `unmatched`?

### 2. Environment lookup and management

The contract has `createEnvironment()` but no way to list existing environments for a tenant, retrieve the details of an environment by ID, or update an environment's configuration. In practice the caller must remember `environment_id` from the creation response indefinitely, with no recovery path if it's lost. An operational system typically needs at least `getEnvironment(environment_id)` and `listEnvironments(tenant_id)`. Confirm whether environment lookup is in scope for v1 or deferred.

### 3. Filter rule operator expressiveness

The current filter rule operators are `equals`, `not_equals`, `in`, `not_in`. These cannot express date-range or numeric-range conditions — e.g. "filter trades dated after 2026-01-01" or "filter amounts above $1M." Should the operator set be extended with `greater_than`, `less_than`, `between`, `before`, `after` based on field data type? If yes, this affects the filter rule schema and validation logic.