# Breakroom API Contract

## What Breakroom does

Osyte reconciles the shared Custodian Feed against the Internal Feed (initiated transaction). Currently, a complete end-to-end reconciliation process doesn't exist that does data transformation, basic validation and record validation and share the reconciliation status both at process level and record level (custodian).

**Breakroom solves two problems:**

1. **"How do I set up reconciliation against one or more sources?"** — Upload the Osyte file and one or more custodian files. AI analyses them, infers the full configuration, and stores it. Review what AI determined, correct anything it got wrong, and the setup is done.

2. **"Do these feeds agree?"** — Upload the Osyte feed and one or more source files. Breakroom runs the full pipeline for each source against the Output Maker and returns a single combined result — summary totals, per-source breakdown, and per-record classification.

---

## Methods

Five methods on a single Breakroom API:

| # | Method name | Route | Solves | What it does |
|---|---|---|---|---|
| 1 | `createReconConfig()` | `POST /breakroom/createReconConfig` | Problem 1 | Creates a reconciliation configuration. AI mode: accepts CSV files and infers the full config. Manual mode: accepts a fully-formed JSON config |
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

For each external source, every source record's composite key is looked up against the Output Maker (Osyte feed). This is a single pass per source. Each source is scoped to its own field mapping, transformation rules, and inline value mappings. `org_id` is derived automatically at run time from the `org_id` field in that source's account identifier TXT-04 inline mappings. The caller never handles `org_id` directly.

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

### 8. AI-assisted configuration infers the full setup from uploaded files

`createReconConfig()` supports an AI mode where the caller uploads the Osyte CSV and one or more source CSVs instead of providing a manually constructed JSON config. AI analyses both files, infers field schemas, field mappings, and normalization rules including TXT-04 inline mappings. The composite key and tolerance rules are never inputs — Breakroom applies system defaults for `trade_reconciliation` and returns them in the response for review.

The full inferred config is returned immediately. A `review_notes` field highlights what AI inferred with lower confidence and what was system-defaulted. The user reviews and corrects anything via `updateReconConfig()`. No separate confirmation or activation step is required — the config is stored and ready to use.

**Why:** Manually constructing a complete config JSON is the primary friction in onboarding a new custodian. AI removes this by turning a form-filling exercise into a file upload and review. The five existing methods are unchanged — AI is an alternative input mode to `createReconConfig()`, not a new concept in the API.

---

## Milestone alignment

Milestones describe what Breakroom's processing pipeline is capable of, not what the user configures. A config created via AI mode always contains the full ruleset — milestones govern which pipeline stages are active when `runReconciliation()` runs.

| Milestone | Pipeline stages active in `runReconciliation()` | Response sections populated |
|---|---|---|
| **M1** | Basic validation (checks 1–5) | `summary` (totals only), `basic_validation` |
| **M2** | + Check 6: filter rules, duplicate check, normalization | + `summary.filtered`, extended `basic_validation` |
| **M3** | + Record validation: key lookup, field comparison, classification | + `records` |

---

## `createReconConfig()`

**Route:** `POST /breakroom/createReconConfig`

**Purpose:** "Set up everything Breakroom needs to reconcile Osyte against one or more sources."

Supports two input modes. Exactly one mode must be used per call — mixing both returns `invalid_input_mode`.

**AI mode:** Caller provides CSV files. AI infers the full configuration. Composite key and tolerance rules are system-defaulted. Full config returned for review.

**Manual mode:** Caller provides a fully-formed JSON config. Composite key and tolerance rules are optional — system defaults applied if omitted. Full config returned.

---

### AI mode inputs

| Param | Type | Required | What it means |
|---|---|---|---|
| `config_name` | string | yes | A human-readable name. Convention: `{TenantShortCode}-Trade-Reconciliation`. |
| `tenant_id` | string | yes | The tenant this configuration belongs to. |
| `reconciliation_type` | string | no | Default: `trade_reconciliation`. v1 supports only `trade_reconciliation`. Future values: `position_reconciliation`, `cash_reconciliation`. |
| `internal_feed_data` | string | yes | Base64-encoded CSV for the Osyte feed (Output Maker). |
| `external_feeds_data` | object[] | yes | Array of source feed payloads. At least one source required. |

**`external_feeds_data` — each source entry (AI mode):**

| Field | Type | Required | What it means |
|---|---|---|---|
| `source_id` | string | yes | A unique identifier for this source. E.g. `"cnb"`, `"boa"`. |
| `feed_name` | string | yes | A human-readable name for this source feed. |
| `files` | string[] | yes | Base64-encoded CSV files for this source. Multiple files are merged before analysis. |

### AI mode example

**Request:**
```jsonc
POST /breakroom/createReconConfig

{
  "config_name": "CommonFund-Trade-Reconciliation",
  "tenant_id": "common-fund",
  "reconciliation_type": "trade_reconciliation",
  "internal_feed_data": "ZnVuZF9pZCxhY2NvdW50X2lkLHRyYWRlX3RyYW5zYWN0aW9uX3R5cGVfY2Qsd...",
  "external_feeds_data": [
    {
      "source_id": "cnb",
      "feed_name": "CNB Records",
      "files": ["QWNjb3VudCAjLFByaW1hcnkgQWNjb3VudCBIb2xkZXIsTG9nIERhdGUsd..."]
    }
  ]
}
```

**Response:**
```jsonc
{
  "config_id": "cfg-cf-001",
  "config_name": "CommonFund-Trade-Reconciliation",
  "tenant_id": "common-fund",
  "reconciliation_type": "trade_reconciliation",
  "status": "active",
  "creation_mode": "ai_assisted",

  "internal_feed": {
    "feed_name": "Osyte Records",
    "fields": [
      {"label": "fund_id",                   "data_type": "numeric", "format": "integer",               "mandatory": true},
      {"label": "account_id",                "data_type": "numeric", "format": "integer",               "mandatory": true},
      {"label": "trade_transaction_type_cd", "data_type": "text",    "format": "VARCHAR(30)",           "mandatory": true},
      {"label": "trade_dt",                  "data_type": "date",    "format": "DATE (M/D/YYYY)",       "mandatory": true},
      {"label": "trade_entered_dt",          "data_type": "date",    "format": "DATETIME (M/D/YYYY H:MM)", "mandatory": false},
      {"label": "settlement_dt",             "data_type": "date",    "format": "DATE (M/D/YYYY)",       "mandatory": false},
      {"label": "final_quantity",            "data_type": "numeric", "format": "DECIMAL(18,6)",         "mandatory": false},
      {"label": "avg_price_per_share",       "data_type": "numeric", "format": "DECIMAL(18,4)",         "mandatory": false},
      {"label": "net_cash_amt",              "data_type": "numeric", "format": "DECIMAL(18,4)",         "mandatory": false}
      // full inferred schema
    ]
  },

  "external_feeds": [
    {
      "source_id": "cnb",
      "feed_name": "CNB Records",
      "fields": [
        {"label": "Account #",        "data_type": "text",    "format": "VARCHAR(20)",              "mandatory": true},
        {"label": "Date (Entry)",     "data_type": "date",    "format": "MM/DD/YYYY or MM-DD-YYYY", "mandatory": true},
        {"label": "Date (Trade)",     "data_type": "date",    "format": "MM/DD/YYYY or MM-DD-YYYY", "mandatory": true},
        {"label": "Date (Settle)",    "data_type": "date",    "format": "MM/DD/YYYY",               "mandatory": true},
        {"label": "Transaction Type", "data_type": "text",    "format": "VARCHAR(50)",              "mandatory": true},
        {"label": "Quantity",         "data_type": "numeric", "format": "DECIMAL(18,6)",            "mandatory": true},
        {"label": "Price",            "data_type": "numeric", "format": "DECIMAL(18,4)",            "mandatory": true},
        {"label": "Net Amount",       "data_type": "numeric", "format": "DECIMAL(18,2)",            "mandatory": true},
        {"label": "Security ID",      "data_type": "text",    "format": "VARCHAR(20)",              "mandatory": true},
        {"label": "Cancel",           "data_type": "text",    "format": "VARCHAR(3)",               "mandatory": true}
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
          "rule_name": "exclude_cancelled",
          "feed": "external",
          "field_label": "Cancel",
          "action": "exclude",
          "operator": "equals",
          "values": ["Yes"]
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
    }
  ],

  "composite_key": ["fund_id", "account_id", "trade_dt", "trade_transaction_type_cd"],

  "tolerance_rules": [
    {"field_label": "trade_entered_dt",    "tolerance_type": "business_days", "tolerance_value": 1},
    {"field_label": "avg_price_per_share", "tolerance_type": "percentage",    "tolerance_value": 0.0001},
    {"field_label": "net_cash_amt",        "tolerance_type": "absolute",      "tolerance_value": 1.00},
    {"field_label": "final_quantity",      "tolerance_type": "absolute",      "tolerance_value": 0},
    {"field_label": "settlement_dt",       "tolerance_type": "directional",   "direction": "lte"}
  ],

  "review_notes": [
    "field_mapping for cnb: inferred from 43 co-occurrence patterns across both files — review recommended",
    "normalization_rules for cnb: TXT-04 Account # mappings inferred from co-occurrence patterns — verify completeness",
    "filter_rules for cnb: exclude_cancelled detected automatically — add any business-specific exclusion rules via updateReconConfig()",
    "composite_key: system default for trade_reconciliation — override via updateReconConfig() if needed",
    "tolerance_rules: system defaults applied — override via updateReconConfig() if your business requires different thresholds"
  ]
}
```

---

### Manual mode inputs

For programmatic config creation or LLM-generated configs. Caller provides the fully-formed config JSON. `composite_key` and `tolerance_rules` are optional — system defaults apply if omitted.

| Param | Type | Required | What it means |
|---|---|---|---|
| `config_name` | string | yes | A human-readable name. Convention: `{TenantShortCode}-Trade-Reconciliation`. |
| `tenant_id` | string | yes | The tenant this configuration belongs to. |
| `reconciliation_type` | string | no | Default: `trade_reconciliation`. Future values: `position_reconciliation`, `cash_reconciliation`. |
| `internal_feed` | object | yes | Feed definition for the Osyte data (Output Maker). Contains `feed_name` and `fields`. |
| `external_feeds` | object[] | yes | Array of external source feed definitions. At least one source required. |
| `composite_key` | string[] | no | Overrides the system default composite key. If omitted, defaults to `["fund_id", "account_id", "trade_dt", "trade_transaction_type_cd"]` for `trade_reconciliation`. |
| `tolerance_rules` | object[] | no | Overrides system default tolerance rules. If omitted, the five standard defaults apply. |

**`internal_feed` — object:**

| Field | Type | What it means |
|---|---|---|
| `feed_name` | string | A human-readable name for the Osyte feed. |
| `fields` | object[] | Field definitions. Each field has `label`, `data_type`, `format`, and `mandatory`. |

**`external_feeds` — each source object:**

| Field | Type | Required | What it means |
|---|---|---|---|
| `source_id` | string | yes | A unique identifier for this source. E.g. `"cnb"`, `"boa"`. |
| `feed_name` | string | yes | A human-readable name for this source feed. |
| `fields` | object[] | yes | Field definitions for this source's CSV. |
| `field_mapping` | object[] | yes | Pairs linking internal (Osyte) fields to this source's fields. |
| `filter_rules` | object[] | no | Records to include or exclude for this source before matching. Can target either feed. Text operators are case-insensitive. |
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
| `operator` | string | Valid operators depend on the field's `data_type`. **Text:** `equals`, `not_equals`, `in`, `not_in`, `contains`, `starts_with`. **Numeric:** `equals`, `not_equals`, `in`, `not_in`, `greater_than`, `less_than`, `between`. **Date:** `equals`, `not_equals`, `before`, `after`, `between`. Text operators are case-insensitive. |
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
| `TXT-04` | Text | Map different labels to a canonical value via inline mappings (e.g. `"buy"`, `"Purchase"`, `"B"` → `"BTO"`) |
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

**System defaults for `trade_reconciliation`:**

`composite_key` default: `["fund_id", "account_id", "trade_dt", "trade_transaction_type_cd"]`

`tolerance_rules` defaults:

| Field | Type | Value |
|---|---|---|
| `trade_entered_dt` | `business_days` | 1 |
| `avg_price_per_share` | `percentage` | 0.0001 (= 0.0001%) |
| `net_cash_amt` | `absolute` | 1.00 (= $1.00) |
| `final_quantity` | `absolute` | 0 (exact match) |
| `settlement_dt` | `directional lte` | custodian date ≤ Osyte date |

**`tolerance_rules` — each rule object:**

| Field | Type | Required | What it means |
|---|---|---|---|
| `field_label` | string | yes | Internal feed label of the comparison field. Must be a mapped field. |
| `tolerance_type` | string | yes | `absolute`, `percentage`, `business_days`, or `directional`. |
| `tolerance_value` | float | conditional | Required for `absolute`, `percentage`, `business_days`. For `percentage`: `0.0001` = 0.0001%. For `absolute`: value in field's native unit. |
| `direction` | string | conditional | Required for `directional`. `lte` = source value must be ≤ Osyte value. |

### `createReconConfig()` output signature

The same full config is returned in both AI and manual modes.

```jsonc
{
  "config_id": "string",
  "config_name": "string",
  "tenant_id": "string",
  "reconciliation_type": "string",
  "status": "active",
  "creation_mode": "ai_assisted | manual",
  "internal_feed": "object (full schema)",
  "external_feeds": "object[] (full per-source config)",
  "composite_key": ["string"],
  "tolerance_rules": "object[]",
  "review_notes": ["string"]
}
```

`review_notes` is present in AI mode only. It lists what AI inferred with lower confidence, what was system-defaulted, and what may need business-specific additions (e.g. filter rules AI could not infer). Empty in manual mode.

---

## `listReconConfigs()`

**Route:** `GET /breakroom/listReconConfigs`

**Purpose:** "What reconciliation configurations do I have set up?"

Returns a summary of every configuration for a tenant. Does not return full schemas or rules — use `getReconConfig()` for full detail.

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
      "config_name": "CommonFund-Trade-Reconciliation",
      "reconciliation_type": "trade_reconciliation",
      "sources": ["cnb", "boa"],
      "status": "active",
      "creation_mode": "ai_assisted",
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
      "creation_mode": "ai_assisted | manual",
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

Returns the full configuration — everything set in `createReconConfig()` or updated since via `updateReconConfig()`.

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
  "creation_mode": "ai_assisted | manual",
  "created_date": "ISO 8601 timestamp",
  "last_updated": "ISO 8601 timestamp",
  "internal_feed": "object (same shape as manual mode input)",
  "external_feeds": "object[] (same shape as manual mode input)",
  "composite_key": ["string"],
  "tolerance_rules": "object[]"
}
```

---

## `updateReconConfig()`

**Route:** `POST /breakroom/updateReconConfig`

**Purpose:** "Change this configuration without rebuilding it from scratch."

Accepts the same fields as `createReconConfig()` manual mode. Only fields included in the request are changed — omitted fields keep their current value. `tenant_id` is immutable. Can also be used to override system-suggested `composite_key` and `tolerance_rules` from AI-assisted creation.

For `external_feeds`: entries are matched by `source_id`. Only sources included in the request are updated — sources not included are unchanged. Within a matched source, only fields provided are updated. To remove a source, recreate the configuration.

### What the caller sends

| Param | Type | Required | What it means |
|---|---|---|---|
| `config_id` | string | yes | The configuration to update. |
| `composite_key` | string[] | no | Override the system-suggested or existing composite key. |
| `tolerance_rules` | object[] | no | Override the system-suggested or existing tolerance rules. |
| *(any other field from manual mode `createReconConfig()` except `tenant_id`)* | varies | no | Only the fields to change. |

### Example — Correcting an AI-inferred field mapping and adding a business filter rule

**Request:**
```jsonc
POST /breakroom/updateReconConfig

{
  "config_id": "cfg-cf-001",
  "external_feeds": [
    {
      "source_id": "cnb",
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
      ]
    }
  ]
}
```

**Response:**
```jsonc
{
  "config_id": "cfg-cf-001",
  "config_name": "CommonFund-Trade-Reconciliation",
  "status": "active",
  "fields_updated": ["external_feeds.cnb.field_mapping", "external_feeds.cnb.filter_rules"],
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

If a source defined in the configuration is not included in `external_feeds_data`, that source is skipped for this run — no validation, no matching, no records in the output for that source. Its `by_source` entry will not appear in the summary. This supports scenarios where a custodian does not deliver their file on a given day.

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

**Note on normalization failures:** A hard failure (malformed mapping entry, unparseable date) returns `normalization_error` and stops that source's run. A soft failure (source value has no matching `from` entry in inline mappings) does not stop the run — affected records classify as `unmatch` or `partial_match`. If unmatch rate is unexpectedly high, review the inline mapping configuration via `getReconConfig()` and correct via `updateReconConfig()`.

### Example

**Request:**
```jsonc
POST /breakroom/runReconciliation

{
  "config_id": "cfg-cf-001",
  "internal_feed_data": "ZnVuZF9pZCxhY2NvdW50X2lkLHRyYWRlX3RyYW5zYWN0aW9uX3R5cGVfY2QsdHJhZGVfYW10...",
  "external_feeds_data": [
    {
      "source_id": "cnb",
      "files": ["QWNjb3VudCAjLFByaW1hcnkgQWNjb3VudCBIb2xkZXIsTG9nIERhdGUsd..."]
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
Note: the createReconConfig() AI mode example created this config with CNB only.
BOA was added as a second source via updateReconConfig() before this run.

CNB: single file. Pipeline runs standard flow.
BOA: two files (split by fund). Merged into one virtual feed before pipeline runs.
  Basic validation runs per file individually.
  Normalization, duplicate check, and matching run on the combined BOA dataset.
Each source runs the full pipeline independently against the same Osyte feed.
```

**Response:**
```jsonc
{
  "reconciliation_id": "REC-20260617-001",
  "config_id": "cfg-cf-001",
  "config_name": "CommonFund-Trade-Reconciliation",
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
      "source_id": null,
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
      "feed_type": null,
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
      "feed_type": null,
      "source_id": "boa",
      "feed_name": "BOA Records",
      "delivery_type": "upload",
      "file_number": "#1",
      "received_date": "2026-06-17T09:30:00Z",
      "processed_date": "2026-06-17T09:30:10Z",
      "feed_status": "completed",
      "checks": [ /* same 6 checks */ ]
    },
    {
      "feed_type": null,
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
        {"field": "fund_id / Account Number",        "internal": "11001",        "external_normalized": "11001",       "match": true,  "tolerance_applied": null},
        {"field": "avg_price_per_share / Unit Price", "internal": "100.0000",     "external_normalized": "100.0500",    "match": false, "tolerance_applied": {"type": "percentage", "limit": 0.0001, "actual": 0.0005}},
        {"field": "net_cash_amt / Net Amt",          "internal": "5000000.0000", "external_normalized": "5002500.0000","match": false, "tolerance_applied": {"type": "absolute",   "limit": 1.00,   "actual": 2500.00}}
      ]
    }
  ]
}
```

**Summary count invariant per source:** `external_records = auto_match + partial_match + unmatch + duplicate + filtered`. Total is the sum across all sources.

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

If `status` is `failed` for a source, that source's `basic_validation` entries show the failing check. Other sources continue independently. Overall `status` is `completed` if at least one source completes; `failed` only if all sources fail or the internal feed fails. `matched_internal_record_ref` is populated for `auto_match`, `partial_match`, and `duplicate`; `null` for `unmatch` and `filtered`. `delivery_type` and `file_number` are system-defaulted to `"upload"` and incrementing `"#1"`, `"#2"` for multiple files per source.

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
| `invalid_field_mapping` | 422 | A `field_mapping` entry references a label not defined in either feed | `createReconConfig` (manual), `updateReconConfig` |
| `invalid_composite_key` | 422 | A field in `composite_key` is not present in any source's `field_mapping` | `createReconConfig` (manual), `updateReconConfig` |
| `invalid_tolerance_rule` | 422 | A field in `tolerance_rules` is not a mapped field, or `tolerance_type` / `direction` is invalid | `createReconConfig` (manual), `updateReconConfig` |
| `invalid_filter_operator` | 422 | A filter rule uses an operator incompatible with the field's `data_type` | `createReconConfig` (manual), `updateReconConfig` |
| `invalid_input_mode` | 422 | Both CSV files and full JSON config provided in the same request | `createReconConfig` |
| `ai_analysis_failed` | 422 | AI could not determine a valid configuration from the provided files (e.g. files are too dissimilar to cross-reference, or contain insufficient records for pattern inference) | `createReconConfig` (AI mode) |
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

Note: a source-level failure does not stop other sources from processing. The overall `status` is `completed` if at least one source completes successfully; `failed` only if all sources fail or the internal feed fails validation.

## Changelog — 2026-07-13

**AI-assisted configuration — `createReconConfig()` dual input mode**
`createReconConfig()` now supports two input modes. AI mode accepts `internal_feed_data` and `external_feeds_data` (base64 CSV files) — AI analyses both files and infers the full configuration including field schemas, field mapping, normalization rules, and TXT-04 inline mappings. Manual mode accepts the full JSON config as before.

**`composite_key` and `tolerance_rules` removed as inputs**
Neither field is ever provided as input. Breakroom applies system defaults for `trade_reconciliation` in both modes. Defaults are always returned in the response and can be overridden via `updateReconConfig()`. In manual mode they remain optional — system defaults apply if omitted.

**`createReconConfig()` response expanded to full config**
Was: lightweight summary (`config_id`, `sources`, `tolerance_rules_set`). Now: full config — all field schemas, all per-source mappings and rules, composite_key, tolerance_rules, plus `review_notes` in AI mode flagging what AI inferred with lower confidence and what was system-defaulted.

**New `review_notes` field**
Present in AI mode response only. Lists what AI inferred vs. what was system-defaulted, and what business-specific rules may still need to be added via `updateReconConfig()`. Empty array in manual mode.

**`creation_mode` field added**
`"ai_assisted" | "manual"` — returned in `createReconConfig()`, `listReconConfigs()`, and `getReconConfig()` responses.

**`updateReconConfig()` — `composite_key` and `tolerance_rules` explicitly added as inputs**
Makes clear that system-suggested defaults from AI-assisted creation can be overridden via `updateReconConfig()`.

**New Assumption 8**
Documents the AI-assisted configuration flow: files uploaded, full config inferred, system defaults applied, user reviews and corrects via `updateReconConfig()`. No draft state — config stored and ready immediately.

**Milestone table reframed**
Milestones now describe Breakroom's processing capabilities, not what the user configures. AI mode always infers a full config upfront — milestones govern which pipeline stages are active in `runReconciliation()`.

**New error codes**
`invalid_input_mode` — both CSV files and full JSON config provided in same request.
`ai_analysis_failed` — AI could not determine a valid configuration from the provided files.

**`runReconciliation()` example — documentation note added**
Added note clarifying that BOA was added as a second source via `updateReconConfig()` before the multi-source run example, making the example consistent with the single-source `createReconConfig()` AI mode example.