# Breakroom API Contract

## What Breakroom does

Osyte reconciles the shared Custodian Feed against the Internal Feed (initiated transaction). Currently, a complete end-to-end reconciliation process doesn't exist that does data transformation, basic validation and record validation and share the reconciliation status both at process level and record level (custodian).

**Breakroom solves two problems:**

1. **"How do I set up reconciliation against one or more sources?"** — In a single call, define the internal Osyte feed as the Output Maker, configure each external source with its own feed schema, field mapping, filter rules, and normalization rules, then set the composite key and tolerance rules that apply across all sources. Breakroom stores all of this as a named reconciliation configuration. Every subsequent daily run references the config by ID.

2. **"Do these feeds agree?"** — Upload the Osyte feed and one or more source files. Breakroom runs the full pipeline for each source against the Output Maker and returns a single combined result — summary totals, per-source breakdown, and per-record classification.

---

## Methods

Five methods on a single Breakroom API:

| # | Method name | Route | Solves | What it does |
|---|---|---|---|---|
| 1 | `createReconConfig()` | `POST /breakroom/createReconConfig` | Problem 1 | Creates a reconciliation configuration: internal feed (Output Maker), one or more external source feeds each with their own mapping and transformation rules, composite key, and tolerances |
| 2 | `listReconConfigs()` | `GET /breakroom/listReconConfigs` | Problem 1 | Returns a summary list of all configurations for a tenant |
| 3 | `getReconConfig()` | `GET /breakroom/getReconConfig/{config_id}` | Problem 1 | Returns the full configuration for a single reconciliation setup |
| 4 | `updateReconConfig()` | `POST /breakroom/updateReconConfig` | Problem 1 | Updates an existing configuration. Only fields included in the request are changed |
| 5 | `runReconciliation()` | `POST /breakroom/runReconciliation` | Problem 2 | Uploads the Osyte feed and one or more source files, runs the full pipeline per source, returns a combined JSON result |

---

## Assumptions

### 1. Configuration is set once, reconciliations run daily

`createReconConfig()` is called once per reconciliation setup. Every `runReconciliation()` call references the config by ID — all feed schemas, field mappings, filter rules, normalization rules, composite key, and tolerances are stored server-side. The caller sends only the CSV files per run.

**Why:** Resending the full config with every daily run is wasteful. Storing it separately means the caller sends only files, and every reconciliation is traceable to the ruleset active at the time.

### 2. Reconciliation runs synchronously and returns the complete result

`runReconciliation()` is a single round-trip. The caller uploads all feeds, the server runs the full pipeline, and returns the complete result in one response.

**Why:** The caller is waiting for the result regardless. A single call returning everything is simpler than a separate trigger-and-fetch pattern.

### 3. The internal feed is the Output Maker — external sources bridge to its format

The internal Osyte feed is the Output Maker. Its field labels define how records appear in the result. Each external source feed is normalized to align with the Output Maker's formats before matching. This is a bridge — the source data is not wrong, it simply uses different naming conventions and codes that need to be translated to Osyte's format.

**Why:** Osyte's schema is fixed and serves as the matching reference. Source data varies per vendor and requires format standardization. Original source values are always preserved and shown as-received in the output.

### 4. Matching runs in a single pass per source — Output Maker drives

For each external source, every source record's composite key is looked up against the Output Maker (Osyte feed). This is a single pass per source. The `source_id` parameter in `createReconConfig()` scopes each external feed to its own field mapping, transformation rules, and inline value mappings. `org_id` is derived automatically at run time from the `org_id` field in that source's account identifier TXT-04 inline mappings. The caller never handles `org_id` directly.

**Why:** The goal is to understand which records from each source agree with Osyte's book. A separate pass per source keeps results clean and independently attributable.

### 5. Value mappings are provided inline at configuration setup

Where source values differ from Osyte values (e.g. `"Buy"` vs `"BTO"`, `"NYK-003640"` vs `"11001"`), a `TXT-04` normalization rule carries the mapping data inline per source feed. Each mapping entry has a `from` field (source value) and a `to` field (Osyte value), and may carry additional fields like `org_id`. Breakroom stores these alongside all other config. When mappings change, the caller updates via `updateReconConfig()`.

**Why:** The caller setting up the configuration already holds this mapping knowledge — they operate on both sides. Embedding mappings inline keeps Breakroom fully self-contained with no external mapping service needed.

### 6. Multiple files from the same source merge into one virtual feed

When a source delivers its daily data across multiple files (e.g. split by fund), all files for that source are merged into one virtual feed before the pipeline runs. Basic validation checks run per file individually. Normalization, duplicate check, and matching run on the combined dataset.

**Why:** Source delivery format (one file vs. many) is an operational detail that should not change how matching works. The combined dataset is the true unit of reconciliation.

### 7. The API returns JSON; the client renders the output file

`runReconciliation()` returns a structured JSON response. The client uses this to render the user-facing output file — CSV in v1, Excel in later phases. Breakroom never produces the output file itself.

**Why:** Format-agnostic JSON means the same response drives the UI, downstream automation, and any future export format without a contract change.

---

## Milestone alignment

| Milestone | `createReconConfig()` fields used | `runReconciliation()` pipeline stages active | Response sections populated |
|---|---|---|---|
| **M1** | `internal_feed`, `external_feeds` (fields only), `field_mapping` per source | Basic validation (checks 1–5) | `summary` (totals only), `basic_validation` |
| **M2** | + `filter_rules`, `normalization_rules` per source | + Normalization service (check 6) | + `summary.filtered`, extended `basic_validation` |
| **M3** | + `composite_key`, `tolerance_rules` | + Record validation (duplicate check, key matching, field comparison, classification) | + `records` |

---

## `createReconConfig()`

**Route:** `POST /breakroom/createReconConfig`

**Purpose:** "Set up everything Breakroom needs to reconcile Osyte against one or more sources."

Creates a named reconciliation configuration in a single call. Defines the internal Osyte feed as the Output Maker, one or more external source feeds each with their own field schema, field mapping, filter rules, and normalization rules, plus the composite key and tolerance rules that apply across all sources. On success the configuration is immediately active.

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `config_name` | string | yes | A human-readable name. Convention: `{TenantShortCode}-Trade-Reconciliation`. |
| `tenant_id` | string | yes | The tenant this configuration belongs to. |
| `reconciliation_type` | string | no | The type of reconciliation. Default: `trade_reconciliation`. v1 supports only `trade_reconciliation`. Future values: `position_reconciliation`, `cash_reconciliation`. |
| `internal_feed` | object | yes | Feed definition for the Osyte data (Output Maker). Contains `feed_name` and `fields`. Its field labels define how results are displayed. |
| `external_feeds` | object[] | yes | Array of external source feed definitions. Each source has its own schema, field mapping, filter rules, and normalization rules. At least one source required. |
| `composite_key` | string[] | conditional | Labels of internal feed fields that form the unique record identifier across all sources. Required for M3. |
| `tolerance_rules` | object[] | no | Acceptable variance on non-key comparison fields. Applies uniformly across all sources. |

**`internal_feed` — object:**

| Field | Type | What it means |
|---|---|---|
| `feed_name` | string | A human-readable name for the Osyte feed. |
| `fields` | object[] | Field definitions. Each field has `label`, `data_type`, `format`, and `mandatory`. |

**`external_feeds` — each source object:**

| Field | Type | Required | What it means |
|---|---|---|---|
| `source_id` | string | yes | A unique identifier for this source within the configuration. E.g. `"cnb"`, `"boa"`. |
| `feed_name` | string | yes | A human-readable name for this source feed. |
| `fields` | object[] | yes | Field definitions for this source's CSV. Each field has `label`, `data_type`, `format`, and `mandatory`. |
| `field_mapping` | object[] | yes | Pairs linking internal (Osyte) fields to this source's fields. |
| `filter_rules` | object[] | no | Records to include or exclude for this source before matching. Applied before normalization. Can target either feed. |
| `normalization_rules` | object[] | no | Format standardization rules applied to this source's feed before matching. |

**`internal_feed.fields` and source `fields` — each field object:**

| Field | Type | What it means |
|---|---|---|
| `label` | string | The column header as it appears in the CSV. |
| `data_type` | string | `numeric`, `text`, or `date`. |
| `format` | string | The specific format or pattern. E.g. `DECIMAL(18,4)`, `VARCHAR(30)`, `DATE (M/D/YYYY)`. |
| `mandatory` | boolean | Whether this field must be present in the uploaded CSV for basic validation to pass. Does not restrict mapping. |

**`field_mapping` — each mapping object:**

| Field | Type | What it means |
|---|---|---|
| `internal_field` | string | Label of the internal (Osyte) feed field. |
| `external_field` | string | Label of this source feed's field it maps to. |

**`filter_rules` — each rule object:**

| Field | Type | What it means |
|---|---|---|
| `rule_name` | string | A caller-supplied identifier. E.g. `exclude_cancelled`. |
| `feed` | string | Which feed to filter: `internal` or `external`. |
| `field_label` | string | The field to evaluate. |
| `action` | string | `include` (keep only matching records) or `exclude` (drop matching records). |
| `operator` | string | Valid operators depend on the field's `data_type`. **Text:** `equals`, `not_equals`, `in`, `not_in`, `contains`, `starts_with`. **Numeric:** `equals`, `not_equals`, `in`, `not_in`, `greater_than`, `less_than`, `between`. **Date:** `equals`, `not_equals`, `before`, `after`, `between`. Text operators are case-insensitive. Using an operator incompatible with the field's data type returns `invalid_filter_operator`. |
| `values` | array | The values to test against. For `between`, provide exactly two values (lower bound, upper bound). |

**`normalization_rules` — each rule object:**

| Field | Type | What it means |
|---|---|---|
| `rule_id` | string | The rule to apply. See rule catalog below. |
| `target_fields` | string[] | The source feed fields this rule applies to. |
| `parameters` | object | Rule-specific parameters. Only `TXT-04` requires a parameter (`mappings`). |

**Normalization rule catalog:**

| Rule ID | Category | What it does |
|---|---|---|
| `TXT-01` | Text | Convert all values to upper case |
| `TXT-02` | Text | Trim leading/trailing spaces; collapse internal multiples |
| `TXT-03` | Text | Remove non-meaningful separators (`-`, `/`, `.`, `_`) unless part of a structural identifier |
| `TXT-04` | Text | Map different labels to a canonical value via inline mappings provided at configuration setup (e.g. `"buy"`, `"Purchase"`, `"B"` → `"BTO"`) |
| `TXT-05` | Text | Replace empty, whitespace-only, or placeholder values with `N/A` |
| `TXT-06` | Text | Remove leading zeros and trailing fillers from identifier fields |
| `NUM-01` | Numeric | Convert to fixed 4-decimal precision |
| `NUM-02` | Numeric | Apply `HALF_UP` rounding to target precision |
| `NUM-03` | Numeric | Convert to absolute value where sign is non-meaningful (e.g. source sends negative quantity for sells; Osyte stores always-positive) |
| `NUM-04` | Numeric | Replace null, empty, or non-numeric values with `0`. Records with Price=0 (e.g. money-market rows) are flagged for exclusion before matching |
| `NUM-05` | Numeric | Correct Net Amount sign and format. Buy transactions should have negative net amounts (cash out); converts positive net amounts on Buy records to negative. Also handles source formats that split integer and decimal into separate columns |
| `DT-01` | Date | Convert all timestamps to UTC |
| `DT-02` | Date | Remove time component when date-only comparison is required |
| `DT-03` | Date | Convert all dates to ISO 8601 (`YYYY-MM-DD`) |
| `DT-05` | Date | Replace null or invalid dates with placeholder `1900-01-01` |

**`tolerance_rules` — each rule object:**

| Field | Type | Required | What it means |
|---|---|---|---|
| `field_label` | string | yes | Internal feed label of the comparison field. Must be a mapped field. |
| `tolerance_type` | string | yes | `absolute` (fixed numeric difference), `percentage` (relative to the internal value), `business_days` (symmetric calendar window), or `directional` (one-sided comparison). |
| `tolerance_value` | float | conditional | Required for `absolute`, `percentage`, and `business_days`. For `percentage`: `0.0001` means 0.0001%. For `absolute`: value is in the field's native unit (e.g. `1.00` = $1.00). |
| `direction` | string | conditional | Required when `tolerance_type` is `directional`. `lte` = source value must be ≤ Osyte value. |

### Example — Setting up Common Fund reconciliation with two sources (CNB and BOA)

**Request:**
```jsonc
POST /breakroom/createReconConfig

{
  "config_name": "Common-Fund-Reconciliation",
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

  "external_feeds": [
    {
      "source_id": "cnb",
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
      ],
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
          "values": ["Buy", "Sell", "Purchase", "BTO", "STC"]
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
        {"rule_id": "TXT-04", "target_fields": ["Transaction Type"], "parameters": {"mappings": [
          {"from": "Buy",      "to": "BTO"},
          {"from": "BUY",      "to": "BTO"},
          {"from": "Purchase", "to": "BTO"},
          {"from": "Sell",     "to": "STC"},
          {"from": "SELL",     "to": "STC"}
        ]}},
        {"rule_id": "TXT-04", "target_fields": ["Account #"], "parameters": {"mappings": [
          {"from": "NYK-003640", "to": "11001", "org_id": "33"},
          {"from": "NYK-003641", "to": "11002", "org_id": "33"},
          {"from": "NYK-003818", "to": "11003", "org_id": "33"}
        ]}},
        {"rule_id": "TXT-04", "target_fields": ["Security ID"], "parameters": {"mappings": [
          {"from": "AVGO", "to": "234644"},
          {"from": "AAPL", "to": "234645"},
          {"from": "MSFT", "to": "235001"}
        ]}},
        {"rule_id": "TXT-06", "target_fields": ["Account #", "Security ID"]},
        {"rule_id": "NUM-01", "target_fields": ["Price", "Net Amount", "Quantity"]},
        {"rule_id": "NUM-02", "target_fields": ["Price", "Net Amount"]},
        {"rule_id": "NUM-03", "target_fields": ["Quantity"]},
        {"rule_id": "DT-01",  "target_fields": ["Date (Entry)"]},
        {"rule_id": "DT-02",  "target_fields": ["Date (Trade)", "Date (Settle)"]},
        {"rule_id": "DT-03",  "target_fields": ["Date (Entry)", "Date (Trade)", "Date (Settle)"]}
      ]
    },
    {
      "source_id": "boa",
      "feed_name": "BOA Records",
      "fields": [
        {"label": "Account Number", "data_type": "text",    "format": "VARCHAR(20)",  "mandatory": true},
        {"label": "Trade Date",     "data_type": "date",    "format": "YYYY-MM-DD",   "mandatory": true},
        {"label": "Settle Date",    "data_type": "date",    "format": "YYYY-MM-DD",   "mandatory": true},
        {"label": "Ticker",         "data_type": "text",    "format": "VARCHAR(20)",  "mandatory": true},
        {"label": "Tran Type",      "data_type": "text",    "format": "VARCHAR(30)",  "mandatory": true},
        {"label": "Units",          "data_type": "numeric", "format": "DECIMAL(18,6)","mandatory": true},
        {"label": "Unit Price",     "data_type": "numeric", "format": "DECIMAL(18,4)","mandatory": true},
        {"label": "Net Amt",        "data_type": "numeric", "format": "DECIMAL(18,2)","mandatory": true}
      ],
      "field_mapping": [
        {"internal_field": "fund_id",                   "external_field": "Account Number"},
        {"internal_field": "account_id",                "external_field": "Ticker"},
        {"internal_field": "trade_dt",                  "external_field": "Trade Date"},
        {"internal_field": "trade_transaction_type_cd", "external_field": "Tran Type"},
        {"internal_field": "settlement_dt",             "external_field": "Settle Date"},
        {"internal_field": "final_quantity",            "external_field": "Units"},
        {"internal_field": "avg_price_per_share",       "external_field": "Unit Price"},
        {"internal_field": "net_cash_amt",              "external_field": "Net Amt"}
      ],
      "filter_rules": [
        {
          "rule_name": "include_buy_sell_only",
          "feed": "external",
          "field_label": "Tran Type",
          "action": "include",
          "operator": "in",
          "values": ["BUY", "SELL", "B", "S"]
        }
      ],
      "normalization_rules": [
        {"rule_id": "TXT-01", "target_fields": ["Tran Type", "Ticker", "Account Number"]},
        {"rule_id": "TXT-04", "target_fields": ["Tran Type"], "parameters": {"mappings": [
          {"from": "B",   "to": "BTO"},
          {"from": "BUY", "to": "BTO"},
          {"from": "S",   "to": "STC"},
          {"from": "SELL","to": "STC"}
        ]}},
        {"rule_id": "TXT-04", "target_fields": ["Account Number"], "parameters": {"mappings": [
          {"from": "CF-001", "to": "11001", "org_id": "33"},
          {"from": "CF-002", "to": "11002", "org_id": "33"}
        ]}},
        {"rule_id": "TXT-04", "target_fields": ["Ticker"], "parameters": {"mappings": [
          {"from": "AVGO", "to": "234644"},
          {"from": "AAPL", "to": "234645"},
          {"from": "MSFT", "to": "235001"}
        ]}},
        {"rule_id": "NUM-01", "target_fields": ["Unit Price", "Net Amt", "Units"]},
        {"rule_id": "NUM-02", "target_fields": ["Unit Price", "Net Amt"]},
        {"rule_id": "DT-03",  "target_fields": ["Trade Date", "Settle Date"]}
      ]
    }
  ],

  "composite_key": ["fund_id", "account_id", "trade_dt", "trade_transaction_type_cd"],

  "tolerance_rules": [
    {"field_label": "trade_entered_dt",    "tolerance_type": "business_days", "tolerance_value": 1},
    {"field_label": "avg_price_per_share", "tolerance_type": "percentage",    "tolerance_value": 0.0001},
    {"field_label": "net_cash_amt",        "tolerance_type": "absolute",      "tolerance_value": 1.00},
    {"field_label": "final_quantity",      "tolerance_type": "absolute",      "tolerance_value": 0},
    {"field_label": "settlement_dt",       "tolerance_type": "directional",   "direction": "lte"}
  ]
}
```

```
Each source has its own field naming conventions and transaction type codes.
CNB uses "Buy"/"Sell", BOA uses "B"/"S" — each has its own TXT-04 mappings.
CNB uses "Account #" column, BOA uses "Account Number" — separate field_mapping per source.
composite_key and tolerance_rules are defined once and apply to all sources uniformly.
```

**Response:**
```jsonc
{
  "config_id": "cfg-cf-001",
  "config_name": "Common-Fund-Reconciliation",
  "tenant_id": "common-fund",
  "reconciliation_type": "trade_reconciliation",
  "status": "active",
  "sources": ["cnb", "boa"],
  "composite_key": ["fund_id", "account_id", "trade_dt", "trade_transaction_type_cd"],
  "tolerance_rules_set": 5
}
```

### `createReconConfig()` output signature

```jsonc
{
  "config_id": "string",
  "config_name": "string",
  "tenant_id": "string",
  "reconciliation_type": "string",
  "status": "active",
  "sources": ["string"],
  "composite_key": ["string"] | null,
  "tolerance_rules_set": "int"
}
```

---

## `listReconConfigs()`

**Route:** `GET /breakroom/listReconConfigs`

**Purpose:** "What reconciliation configurations do I have set up?"

Returns a summary of every configuration for a tenant. Does not return full feed schemas or rules — use `getReconConfig()` for full detail.

### What the caller sends

| Param | Type | Required | What it means |
|---|---|---|---|
| `tenant_id` | string | yes | The tenant to list configurations for. |

### Example

**Request:** `GET /breakroom/listReconConfigs?tenant_id=common-fund`

**Response:**
```jsonc
{
  "tenant_id": "common-fund",
  "configs": [
    {
      "config_id": "cfg-cf-001",
      "config_name": "Common-Fund-Reconciliation",
      "reconciliation_type": "trade_reconciliation",
      "sources": ["cnb", "boa"],
      "status": "active",
      "created_date": "2026-06-10T14:00:00Z",
      "last_updated": "2026-06-17T09:00:00Z"
    }
  ]
}
```

### `listReconConfigs()` output signature

```jsonc
{
  "tenant_id": "string",
  "configs": [
    {
      "config_id": "string",
      "config_name": "string",
      "reconciliation_type": "string",
      "sources": ["string"],
      "status": "active",
      "created_date": "ISO 8601 timestamp",
      "last_updated": "ISO 8601 timestamp"
    }
  ]
}
```

---

## `getReconConfig()`

**Route:** `GET /breakroom/getReconConfig/{config_id}`

**Purpose:** "Show me exactly how this reconciliation is configured."

Returns the full configuration — everything set in `createReconConfig()` or changed since via `updateReconConfig()`.

### What the caller sends

| Param | Type | Required | What it means |
|---|---|---|---|
| `config_id` | string | yes | The configuration to retrieve (in the URL path). |

### `getReconConfig()` output signature

```jsonc
{
  "config_id": "string",
  "config_name": "string",
  "tenant_id": "string",
  "reconciliation_type": "string",
  "status": "active",
  "created_date": "ISO 8601 timestamp",
  "last_updated": "ISO 8601 timestamp",
  "internal_feed": "object (same shape as createReconConfig() input)",
  "external_feeds": "object[] (same shape as createReconConfig() input)",
  "composite_key": ["string"] | null,
  "tolerance_rules": "object[] (same shape as createReconConfig() input)"
}
```

---

## `updateReconConfig()`

**Route:** `POST /breakroom/updateReconConfig`

**Purpose:** "Change this configuration without rebuilding it from scratch."

Accepts the same fields as `createReconConfig()`. Only fields included in the request are changed — omitted fields keep their current value. `tenant_id` is immutable.

For `external_feeds`: when included in the request, each entry is matched by `source_id`. Only the sources included are updated — sources not included in the request are unchanged. Within a matched source, only the fields provided are updated; omitted source fields keep their current value. To add a new source, include its full definition. To remove a source, this requires recreating the configuration.

### What the caller sends

| Param | Type | Required | What it means |
|---|---|---|---|
| `config_id` | string | yes | The configuration to update. |
| *(any field from `createReconConfig()` except `tenant_id`)* | varies | no | Only the fields to change. |

### Example — Adding a new mapping entry to CNB's Account # mappings

**Request:**
```jsonc
POST /breakroom/updateReconConfig

{
  "config_id": "cfg-cf-001",
  "external_feeds": [
    {
      "source_id": "cnb",
      "normalization_rules": [
        // ...existing rules...
        {"rule_id": "TXT-04", "target_fields": ["Account #"], "parameters": {"mappings": [
          {"from": "NYK-003640", "to": "11001", "org_id": "33"},
          {"from": "NYK-003641", "to": "11002", "org_id": "33"},
          {"from": "NYK-003818", "to": "11003", "org_id": "33"},
          {"from": "NYK-003922", "to": "11004", "org_id": "33"}
        ]}}
      ]
    }
  ]
}
```

**Response:**
```jsonc
{
  "config_id": "cfg-cf-001",
  "config_name": "Common-Fund-Reconciliation",
  "status": "active",
  "fields_updated": ["external_feeds.cnb.normalization_rules"],
  "last_updated": "2026-06-18T10:15:00Z"
}
```

### `updateReconConfig()` output signature

```jsonc
{
  "config_id": "string",
  "config_name": "string",
  "status": "active",
  "fields_updated": ["string"],
  "last_updated": "ISO 8601 timestamp"
}
```

---

## `runReconciliation()`

**Route:** `POST /breakroom/runReconciliation`

**Purpose:** "Do these feeds agree?"

Uploads the Osyte internal feed and one or more source feeds. Runs the full pipeline per source against the Output Maker and returns a combined result — aggregate summary, per-source breakdown, and per-record classification.

### What the caller sends

| Param | Type | Required | What it means |
|---|---|---|---|
| `config_id` | string | yes | The active configuration to run against. |
| `internal_feed_data` | string | yes | Base64-encoded CSV for the Osyte feed (Output Maker). |
| `external_feeds_data` | object[] | yes | Array of source feed payloads. One entry per source. |

**`external_feeds_data` — each source payload:**

| Field | Type | Required | What it means |
|---|---|---|---|
| `source_id` | string | yes | Must match a `source_id` defined in the configuration. |
| `files` | string[] | yes | Array of base64-encoded CSV files for this source. Multiple files are merged into one virtual feed before the pipeline runs. |

For large files, multipart/form-data upload is also accepted using the same field names.

If a source defined in the configuration is not included in `external_feeds_data`, that source is skipped for this run — no validation, no matching, no records in the output for that source. Its `by_source` entry will not appear in the summary. This supports scenarios where a custodian does not deliver their file on a given day. All other sources process normally.

### Processing pipeline

M1 runs checks 1–5; check 6 is added in M2. Any failure stops the run for that source. The pipeline runs independently per source:

| Step | Check | Available from | Failure code |
|---|---|---|---|
| 1 | Feed Received Check — confirms the feed was received and is non-empty | M1 | `feed_not_received` |
| 2 | Feed Format Check — confirms CSV structure is well-formed | M1 | `feed_format_error` |
| 3 | Feed Failed Check — confirms the file is readable and not corrupted | M1 | `feed_file_error` |
| 4 | Feed Field Check — confirms all mandatory fields are present | M1 | `missing_feed_fields` |
| 5 | Feed Field Data Type Check — confirms field values match configured data types | M1 | `data_type_mismatch` |
| 6 | Feed Formatting Service Check — applies filter rules (both feeds), duplicate check (both feeds), then normalization rules (source feed) | M2 | `normalization_error` |

If all configured checks pass for a source, record validation begins for that source:
1. Each source record is assigned a system-generated `record_id`, unique within the reconciliation run.
2. **Filter rules** drop excluded records. These appear in `records` with `reconciliation_status: "filtered"`.
3. **Duplicate check (source feed):** 2 or more source records sharing the same composite key → all classified as `duplicate`.
4. **Duplicate check (Osyte feed):** exact duplicates (same composite key and same field values) → latest Osyte record participates, first is excluded.
5. **Key lookup:** composite key of each remaining source record looked up against the Osyte feed. No match → `unmatch`. Match found → field comparison.
6. **Field comparison:** compare all mapped non-key fields within tolerance. Any field exceeds tolerance → `partial_match`. All within tolerance → `auto_match`.

**Note on normalization failures:** A hard failure (malformed mapping entry, unparseable date) returns `normalization_error` and stops that source's run. A soft failure (source value has no matching `from` entry in inline mappings) does not stop the run — affected records classify as `unmatch` or `partial_match`. If unmatch rate is unexpectedly high, review the inline mapping configuration.

### Example — Running Common Fund reconciliation with CNB and BOA

**Request:**
```jsonc
POST /breakroom/runReconciliation

{
  "config_id": "cfg-cf-001",
  "internal_feed_data": "ZnVuZF9pZCxhY2NvdW50X2lkLHRyYWRlX3RyYW5zYWN0aW9uX3R5cGVfY2QsdHJhZGVfYW10...",
  "external_feeds_data": [
    {
      "source_id": "cnb",
      "files": [
        "QWNjb3VudCAjLFByaW1hcnkgQWNjb3VudCBIb2xkZXIsTG9nIERhdGUsd..."
      ]
    },
    {
      "source_id": "boa",
      "files": [
        "QWNjb3VudCBOdW1iZXIsTHJhZGUgRGF0ZSxTZXR0bGUgRGF0ZSxUaWNrZXIs...",
        "QWNjb3VudCBOdW1iZXIsTHJhZGUgRGF0ZSxTZXR0bGUgRGF0ZSxUaWNrZXIs..."
      ]
    }
  ]
}
```

```
CNB: single file. Pipeline runs standard flow.
BOA: two files (split by fund). Merged into one virtual feed before pipeline runs.
  Basic validation checks 1-5 run per file individually.
  Normalization, duplicate check, and matching run on the combined BOA dataset.

Each source runs the full pipeline independently against the same Osyte feed.
The Osyte feed is the shared reference — each source runs over it separately.
```

**Response:**
```jsonc
{
  "reconciliation_id": "REC-20260617-001",
  "config_id": "cfg-cf-001",
  "config_name": "Common-Fund-Reconciliation",
  "reconciliation_type": "trade_reconciliation",
  "status": "completed",
  "start_date": "2026-06-17T09:30:00Z",
  "end_date": "2026-06-17T09:30:42Z",

  "summary": {
    "total": {
      "external_records": 800,
      "auto_match": 650,
      "partial_match": 100,
      "unmatch": 40,
      "duplicate": 5,
      "filtered": 5
    },
    "by_source": [
      {
        "source_id": "cnb",
        "external_records": 500,
        "auto_match": 420,
        "partial_match": 60,
        "unmatch": 15,
        "duplicate": 3,
        "filtered": 2
      },
      {
        "source_id": "boa",
        "external_records": 300,
        "auto_match": 230,
        "partial_match": 40,
        "unmatch": 25,
        "duplicate": 2,
        "filtered": 3
      }
    ]
  },

  "basic_validation": [
    {
      "feed_type": "internal",
      "feed_name": "Osyte Records",
      "delivery_type": "upload",
      "file_number": "#1",
      "received_date": "2026-06-17T09:30:00Z",
      "processed_date": "2026-06-17T09:30:05Z",
      "feed_status": "completed",
      "checks": [
        {"sno": 1, "validation": "Feed Received Check",           "status": "passed", "break_type": null, "break_description": null},
        {"sno": 2, "validation": "Feed Format Check",             "status": "passed", "break_type": null, "break_description": null},
        {"sno": 3, "validation": "Feed Failed Check",             "status": "passed", "break_type": null, "break_description": null},
        {"sno": 4, "validation": "Feed Field Check",              "status": "passed", "break_type": null, "break_description": null},
        {"sno": 5, "validation": "Feed Field Data Type Check",    "status": "passed", "break_type": null, "break_description": null},
        {"sno": 6, "validation": "Feed Formatting Service Check", "status": "passed", "break_type": null, "break_description": null}
      ]
    },
    {
      "source_id": "cnb",
      "feed_name": "CNB Records",
      "delivery_type": "upload",
      "file_number": "#1",
      "received_date": "2026-06-17T09:30:00Z",
      "processed_date": "2026-06-17T09:30:08Z",
      "feed_status": "completed",
      "checks": [ /* same 6 checks */ ]
    },
    {
      "source_id": "boa",
      "feed_name": "BOA Records",
      "delivery_type": "upload",
      "file_number": "#1",
      "received_date": "2026-06-17T09:30:00Z",
      "processed_date": "2026-06-17T09:30:10Z",
      "feed_status": "completed",
      "checks": [ /* same 6 checks, run per file */ ]
    },
    {
      "source_id": "boa",
      "feed_name": "BOA Records",
      "delivery_type": "upload",
      "file_number": "#2",
      "received_date": "2026-06-17T09:30:00Z",
      "processed_date": "2026-06-17T09:30:12Z",
      "feed_status": "completed",
      "checks": [ /* same 6 checks */ ]
    }
  ],

  "records": [
    {
      "record_id": "REC-001",
      "source_id": "cnb",
      "reconciliation_status": "auto_match",
      "external_record": {
        "Account #": "NYK-003640", "Date (Trade)": "5/22/2026",
        "Transaction Type": "buy", "Quantity": 214803.848,
        "Price": 257.6500, "Net Amount": 55344211.45, "Security ID": "AVGO"
      },
      "matched_internal_record_ref": "610989",
      "field_comparison": [ /* all fields match within tolerance */ ]
    },
    {
      "record_id": "REC-501",
      "source_id": "boa",
      "reconciliation_status": "partial_match",
      "external_record": {
        "Account Number": "CF-001", "Trade Date": "2026-05-22",
        "Tran Type": "B", "Units": 50000,
        "Unit Price": 100.0500, "Net Amt": 5002500.00, "Ticker": "MSFT"
      },
      "matched_internal_record_ref": "610991",
      "field_comparison": [
        {"field": "fund_id / Account Number",         "internal": "11001",        "external_normalized": "11001",       "match": true,  "tolerance_applied": null},
        {"field": "avg_price_per_share / Unit Price",  "internal": "100.0000",     "external_normalized": "100.0500",    "match": false, "tolerance_applied": {"type": "percentage", "limit": 0.0001, "actual": 0.0005}},
        {"field": "net_cash_amt / Net Amt",           "internal": "5000000.0000", "external_normalized": "5002500.0000","match": false, "tolerance_applied": {"type": "absolute",   "limit": 1.00,   "actual": 2500.00}}
      ]
    }
  ]
}
```

**Summary count invariant per source:** `external_records = auto_match + partial_match + unmatch + duplicate + filtered`. Total summary is the sum across all sources.

### `runReconciliation()` output signature

```jsonc
{
  "reconciliation_id": "string",
  "config_id": "string",
  "config_name": "string",
  "reconciliation_type": "string",
  "status": "completed | failed",
  "start_date": "ISO 8601 timestamp",
  "end_date": "ISO 8601 timestamp",

  "summary": {
    "total": {
      "external_records": "int",
      "auto_match": "int",
      "partial_match": "int",
      "unmatch": "int",
      "duplicate": "int",
      "filtered": "int"
    },
    "by_source": [
      {
        "source_id": "string",
        "external_records": "int",
        "auto_match": "int",
        "partial_match": "int",
        "unmatch": "int",
        "duplicate": "int",
        "filtered": "int"
      }
    ]
  },

  "basic_validation": [
    {
      "feed_type": "internal | null",
      "source_id": "string | null",
      "feed_name": "string",
      "delivery_type": "string",
      "file_number": "string",
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
      "source_id": "string",
      "reconciliation_status": "auto_match | partial_match | unmatch | duplicate | filtered",
      "external_record": "object (source field values as received)",
      "matched_internal_record_ref": "string | null",
      "field_comparison": [
        {
          "field": "string (internal_label / external_label)",
          "internal": "string",
          "external_normalized": "string",
          "match": "boolean",
          "tolerance_applied": "{ type: string, limit: float, actual: float } | { type: string, direction: string, result: string } | null"
        }
      ]
    }
  ]
}
```

If `status` is `failed` for a source, that source's `basic_validation` entries are populated with the failing check. Other sources continue processing independently. `matched_internal_record_ref` is populated for `auto_match`, `partial_match`, and `duplicate`; `null` for `unmatch` and `filtered`. `delivery_type` and `file_number` in `basic_validation` are system-defaulted to `"upload"` and `"#1"` (incrementing for multiple files per source).

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
| `invalid_field_mapping` | 422 | A `field_mapping` entry references a label not defined in either feed | `createReconConfig`, `updateReconConfig` |
| `invalid_composite_key` | 422 | A field in `composite_key` is not present in any source's `field_mapping` | `createReconConfig`, `updateReconConfig` |
| `invalid_tolerance_rule` | 422 | A field in `tolerance_rules` is not a mapped field, or `tolerance_type` / `direction` is invalid | `createReconConfig`, `updateReconConfig` |
| `invalid_filter_operator` | 422 | A filter rule uses an operator incompatible with the field's `data_type` | `createReconConfig`, `updateReconConfig` |
| `duplicate_source_id` | 422 | Two entries in `external_feeds` have the same `source_id` | `createReconConfig`, `updateReconConfig` |
| `config_not_found` | 404 | No configuration found for the given `config_id` | `getReconConfig`, `updateReconConfig`, `runReconciliation` |
| `tenant_not_found` | 404 | No configurations found for the given `tenant_id` | `listReconConfigs` |
| `source_not_found` | 404 | A `source_id` in `external_feeds_data` does not match any source defined in the configuration | `runReconciliation` |
| `feed_not_received` | 422 | A feed was not received or is empty | `runReconciliation` |
| `feed_format_error` | 422 | Uploaded feed is not a valid CSV | `runReconciliation` |
| `feed_file_error` | 422 | The file is unreadable or corrupted | `runReconciliation` |
| `missing_feed_fields` | 422 | One or more mandatory fields are absent from the uploaded feed | `runReconciliation` |
| `data_type_mismatch` | 422 | A field value does not match its configured `data_type` | `runReconciliation` |
| `normalization_error` | 422 | A normalization rule failed to execute (e.g. malformed mapping entry, unparseable date) | `runReconciliation` |

Note: a source-level failure (one source fails basic validation) does not stop other sources from processing. The overall `status` is `completed` if at least one source completes successfully; `failed` only if all sources fail or the internal feed fails validation.

## Changelog 

**Rename: All five environment methods**
`createEnvironment()` → `createReconConfig()`
`listEnvironments()` → `listReconConfigs()`
`getEnvironment()` → `getReconConfig()`
`updateEnvironment()` → `updateReconConfig()`
`runReconciliation()` — unchanged
All associated IDs and field names updated: `environment_id` → `config_id`, `environment_name` → `config_name` throughout.

**Rename: `custodian_id` → `source_id`**
Renamed everywhere — top-level config, each external feed entry, `runReconciliation()` payload, response records, summary breakdown, errors table, assumptions.

**AI implementation — architecture decision documented**
The Breakroom API contract does not change for AI. The five existing methods are the tools an LLM agent can call directly with the two CSV files as input. AI generates valid `createReconConfig()` / `updateReconConfig()` calls — the API receives them identically whether produced by a human or an LLM. No new Breakroom endpoint needed. AI assistance is a product layer on top of the existing contract.

**Multiple sources — single reconciliation run**
`external_feed` (single object) → `external_feeds` (array). Each source carries its own `source_id`, `fields`, `field_mapping`, `filter_rules`, and `normalization_rules`. `composite_key` and `tolerance_rules` remain at the top level and apply uniformly across all sources. Each source runs the full pipeline independently against the same Osyte Output Maker feed.

`runReconciliation()` input updated: `external_feed_data` (single string) → `external_feeds_data` (array of `{ source_id, files[] }`).

Response summary updated: single flat summary → `total` block plus `by_source` array breakdown. Each record in `records` array now carries `source_id`.

`basic_validation` now has one entry per file per source, plus the internal feed entry.

**Single file from multiple sources**
Each source in `external_feeds_data` accepts a `files` array. Multiple files from the same source merge into one virtual feed before the pipeline runs. Basic validation runs per file individually. Normalization, duplicate check, and matching run on the combined dataset. Assumption 6 added to document this.

**New error codes**
`duplicate_source_id` — two entries in `external_feeds` share the same `source_id`.
`source_not_found` — a `source_id` in `external_feeds_data` does not match any source in the configuration.

