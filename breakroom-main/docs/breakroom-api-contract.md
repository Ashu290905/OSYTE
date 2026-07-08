# Breakroom API Contract

## What Breakroom does

Osyte reconciles the shared Custodian Feed against the Internal Feed (initiated transaction). Currently, a complete end-to-end reconciliation process doesn't exists that does data transformation, basic validation and record validation and share the reconciliation status both at process level and record level (custodian).

**Breakroom solves two problems:**

1. **"How do I set up reconciliation against a new custodian?"** — In a single call, provide both feed schemas, the field mapping between them, filter and normalization rules to handle format differences, and the composite key and tolerance rules for matching. Breakroom stores all of this as a named environment. Every subsequent daily run references the environment by ID — no ruleset resent, no config drift.

2. **"Do these two feeds agree?"** — Given a configured reconciliation environment and two CSV feeds, run the full validation, normalization, and matching pipeline. Classify every custodian record as Auto-Match, Partial Match, Unmatch, Duplicate, or Filtered, and report the result as structured JSON.

---

## Methods

Five methods on a single Breakroom API:

| # | Method name | Route | Solves | What it does |
|---|---|---|---|---|
| 1 | `createEnvironment()` | `POST /breakroom/createEnvironment` | Problem 1 | Creates a reconciliation environment in a single call: feed schemas, field mapping, filter rules, normalization rules, composite key, and tolerances |
| 2 | `listEnvironments()` | `GET /breakroom/listEnvironments` | Problem 1 | Returns a summary list of all environments for a tenant |
| 3 | `getEnvironment()` | `GET /breakroom/getEnvironment/{environment_id}` | Problem 1 | Returns the full configuration of a single environment |
| 4 | `updateEnvironment()` | `POST /breakroom/updateEnvironment` | Problem 1 | Updates an existing environment's configuration. Only fields included in the request are changed |
| 5 | `runReconciliation()` | `POST /breakroom/runReconciliation` | Problem 2 | Uploads both feeds, runs the full pipeline, and returns the complete JSON result |

---

## Assumptions

### 1. Configuration is set once, reconciliations run daily

`createEnvironment()` is called once per custodian relationship. Every `runReconciliation()` call references the environment by ID — feed schemas, field mapping, filter rules, normalization rules, composite key, and tolerances are all stored server-side. The caller sends only the two CSV files per run.

**Why:** Resending the full config with every daily run is wasteful and breaks auditability. Storing config separately means every reconciliation can be traced to the ruleset that was active when it ran.

### 2. Reconciliation runs synchronously and returns the complete result

`runReconciliation()` is a single round-trip. The caller uploads both feeds, the server runs the full pipeline, and returns the complete result in one response.

**Why:** The caller is waiting for the result regardless. A single call returning everything is simpler than a separate trigger-and-fetch pattern.

### 3. The custodian feed is the source of truth — normalization bridges it to Osyte's format

The custodian feed is the source of truth for the reconciliation. It represents what the custodian has recorded, and the goal is to understand how well Osyte's internal records align with it. The custodian feed is the output header — its field labels define how records appear in the result.

Because the custodian uses different naming conventions, formats, and codes than Osyte, normalization runs on the custodian feed to bridge its values into Osyte's internal format before matching. This is a bridge, not a correction — the custodian's data is not wrong; it simply needs to be expressed in terms Osyte's matching engine can compare against.

**Why:** Custodian data is external and varies per vendor. Normalization aligns the custodian's representation to Osyte's without altering the custodian's underlying values — the original custodian data is always preserved and shown as-received in the output.

### 4. Matching runs in a single pass — custodian drives

Each custodian record's composite key is looked up against every record in the Osyte feed. This is a single pass. The result tells the analyst which custodian records reconciled and which did not.

The `custodian_id` parameter in `createEnvironment()` scopes this environment to a specific custodian relationship. When TXT-04 normalizes custodian identifiers such as `Account #`, it uses the inline mappings provided at environment setup. These mappings can carry additional values beyond the core from/to pair — for example, the `Account #` mapping includes `org_id` per entry, which Breakroom automatically injects into the composite key at run time. The caller never handles `org_id` directly.

**Why:** The custodian feed drives the reconciliation. The goal is to understand which of the custodian's reported records match Osyte's book.

### 5. Custodian-to-Osyte value mappings are provided inline at environment setup

Where custodian values differ from Osyte values (e.g. `"Buy"` vs `"BTO"`, `"NYK-003640"` vs `"11001"`), a `TXT-04` normalization rule carries the mapping data inline as part of `createEnvironment()`. Breakroom stores these mappings in its own storage alongside the rest of the environment config.

Each mapping entry has a fixed structure:

| Field | What it holds |
|---|---|
| `from` | The value as it appears in the custodian feed (e.g. `"Buy"`, `"NYK-003640"`) |
| `to` | The corresponding Osyte value (e.g. `"BTO"`, `"11001"`) |

Mapping entries may carry additional fields (e.g. `org_id`) that Breakroom uses internally at match time. At run time, `TXT-04` looks up the custodian value in `from` and substitutes the Osyte value from `to`. The caller who sets up the environment already holds this mapping knowledge — they operate on both sides, knowing both their custodian's identifiers and their own Osyte internal values. When mappings change, the caller updates via `updateEnvironment()`.

**Why:** The caller inherently has mapping knowledge as a prerequisite for reconciliation. Embedding mappings inline keeps Breakroom fully self-contained — no external mapping service, no external database.

### 6. The API returns JSON; the client renders the output file

`runReconciliation()` returns a structured JSON response. The client uses this to render the user-facing output file — CSV in v1, Excel in later phases. Breakroom never produces the output file itself.

**Why:** Format-agnostic JSON means the same response drives the UI, downstream automation, and any future export format without a contract change.

---

## Milestone alignment

| Milestone | `createEnvironment()` fields used | `runReconciliation()` pipeline stages active | Response sections populated |
|---|---|---|---|
| **M1** | `internal_feed`, `external_feed`, `field_mapping` | Basic validation (checks 1–5) | `summary` (totals only), `basic_validation` |
| **M2** | + `filter_rules`, `normalization_rules` | + Normalization service (check 6) | + `summary.filtered`, extended `basic_validation` |
| **M3** | + `composite_key`, `tolerance_rules` | + Record validation (duplicate check, key matching, field comparison, classification) | + `records` |

In M1 and M2, `composite_key` and `tolerance_rules` may be omitted; `records` will be empty. The contract structure is stable across all three milestones — only the populated sections grow.

---

## `createEnvironment()`

**Route:** `POST /breakroom/createEnvironment`

**Purpose:** "Set up everything Breakroom needs to reconcile Osyte against this custodian."

Creates a named reconciliation environment in a single call. Contains the complete configuration: both feed schemas, the field mapping between them, filter rules (applicable to both feeds) and normalization rules (custodian feed only), the composite key definition, and tolerance rules for comparison fields. On success the environment is immediately active.

### What the caller sends

**Inputs:**

| Param | Type | Required | What it means |
|---|---|---|---|
| `environment_name` | string | yes | A human-readable name. Convention: `Osyte-{CustodianShortCode}-Reconciliation`. |
| `tenant_id` | string | yes | The tenant this environment belongs to. |
| `custodian_id` | string | yes | The custodian this environment reconciles against. Scopes the environment to a specific custodian relationship. `org_id` is derived automatically at run time from the `org_id` field in the `Account #` TXT-04 inline mappings. The caller never handles `org_id` directly. |
| `reconciliation_type` | string | no | Default: `trade_reconciliation`. v1 supports only `trade_reconciliation`. |
| `internal_feed` | object | yes | Feed definition for the Osyte data. Contains `feed_name` and `fields`. |
| `external_feed` | object | yes | Feed definition for the custodian data. Contains `feed_name` and `fields`. The custodian feed is the output header — its field labels define how results are displayed. |
| `field_mapping` | object[] | yes | Pairs linking internal fields to their corresponding external fields. The `mandatory` flag on a field definition governs CSV validation only — any field can be mapped regardless of its `mandatory` setting. |
| `filter_rules` | object[] | no | Records to include or exclude before matching. Applied before normalization. Can target either feed. |
| `normalization_rules` | object[] | no | Format standardization rules applied to the custodian feed before matching. To add a rule for a new field, add a new entry referencing an existing `rule_id`. New `rule_id` values require a Breakroom engineering change. |
| `composite_key` | string[] | conditional | Labels of internal feed fields that form the unique record identifier. Required for M3. |
| `tolerance_rules` | object[] | no | Acceptable variance on non-key comparison fields. Fields without a tolerance rule require exact match after normalization. |

**`internal_feed.fields` and `external_feed.fields` — each field object:**

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
| `external_field` | string | Label of the external (custodian) feed field it maps to. |

**`filter_rules` — each rule object:**

| Field | Type | What it means |
|---|---|---|
| `rule_name` | string | A caller-supplied identifier. E.g. `exclude_cancelled`. |
| `feed` | string | Which feed to filter: `internal` or `external`. |
| `field_label` | string | The field to evaluate. |
| `action` | string | `include` (keep only matching records) or `exclude` (drop matching records). |
| `operator` | string | The comparison operator. Valid operators depend on the field's `data_type` as defined in the feed schema. **Text:** `equals`, `not_equals`, `in`, `not_in`, `contains`, `starts_with`. **Numeric:** `equals`, `not_equals`, `in`, `not_in`, `greater_than`, `less_than`, `between`. **Date:** `equals`, `not_equals`, `before`, `after`, `between`. Using an operator incompatible with the field's data type returns `invalid_filter_operator`. |
| `values` | array | The values to test against. For `between`, provide exactly two values (lower bound, upper bound). For all other operators, provide one or more values as appropriate. |

**`normalization_rules` — each rule object:**

| Field | Type | What it means |
|---|---|---|
| `rule_id` | string | The rule to apply. See rule catalog below. |
| `target_fields` | string[] | The custodian feed fields this rule applies to. |
| `parameters` | object | Rule-specific parameters. Only `TXT-04` requires a parameter (`mappings`). |

**Normalization rule catalog:**

| Rule ID | Category | What it does |
|---|---|---|
| `TXT-01` | Text | Convert all values to upper case |
| `TXT-02` | Text | Trim leading/trailing spaces; collapse internal multiples |
| `TXT-03` | Text | Remove non-meaningful separators (`-`, `/`, `.`, `_`) unless part of a structural identifier |
| `TXT-04` | Text | Map different labels to a canonical value via inline mappings provided at environment setup (e.g. `"buy"`, `"Purchase"`, `"B"` → `"BTO"`) |
| `TXT-05` | Text | Replace empty, whitespace-only, or placeholder values with `N/A` |
| `TXT-06` | Text | Remove leading zeros and trailing fillers from identifier fields |
| `NUM-01` | Numeric | Convert to fixed 4-decimal precision |
| `NUM-02` | Numeric | Apply `HALF_UP` rounding to target precision |
| `NUM-03` | Numeric | Convert to absolute value where sign is non-meaningful (e.g. custodian sends negative quantity for sells; Osyte stores always-positive) |
| `NUM-04` | Numeric | Replace null, empty, or non-numeric values with `0`. Records with Price=0 (e.g. money-market rows) are flagged for exclusion before matching |
| `NUM-05` | Numeric | Correct Net Amount sign and format. Buy transactions should have negative net amounts (cash out); converts positive net amounts on Buy records to negative. Also handles custodian formats that split integer and decimal into separate columns |
| `DT-01` | Date | Convert all timestamps to UTC |
| `DT-02` | Date | Remove time component when date-only comparison is required |
| `DT-03` | Date | Convert all dates to ISO 8601 (`YYYY-MM-DD`) |
| `DT-05` | Date | Replace null or invalid dates with placeholder `1900-01-01` |

**`tolerance_rules` — each rule object:**

| Field | Type | Required | What it means |
|---|---|---|---|
| `field_label` | string | yes | Internal feed label of the comparison field. Must be a mapped field. |
| `tolerance_type` | string | yes | `absolute` (fixed numeric difference), `percentage` (relative to the internal value), `business_days` (symmetric calendar window), or `directional` (one-sided comparison). |
| `tolerance_value` | float | conditional | The permitted deviation. Required for `absolute`, `percentage`, and `business_days`. For `percentage`: `tolerance_value: 0.0001` means 0.0001%, `tolerance_value: 1` means 1%. For `absolute`: value is in the field's native unit (e.g. `1.00` = $1.00). |
| `direction` | string | conditional | Required when `tolerance_type` is `directional`. `lte` = custodian value must be ≤ Osyte value (e.g. settlement date can arrive early but not late). |

### Example — Setting up Osyte ↔ City National Bank reconciliation for Common Fund

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
      {"from": "MSFT", "to": "235001"},
      {"from": "TSLA", "to": "235002"},
      {"from": "PYPL", "to": "235003"}
    ]}},
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
    {"field_label": "net_cash_amt",        "tolerance_type": "absolute",      "tolerance_value": 1.00},
    {"field_label": "final_quantity",      "tolerance_type": "absolute",      "tolerance_value": 0},
    {"field_label": "settlement_dt",       "tolerance_type": "directional",   "direction": "lte"}
  ]
}
```

```
tolerance_value: 0.0001 on avg_price_per_share means ±0.0001%, not 0.0001 as a decimal fraction.
tolerance_value: 1.00 on net_cash_amt means ±$1.00 absolute difference.
settlement_dt directional lte: custodian settlement date must be ≤ Osyte settlement date.
  Custodian arriving early = pass. Custodian arriving late = Partial Match.
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

## `listEnvironments()`

**Route:** `GET /breakroom/listEnvironments`

**Purpose:** "What environments do I have set up?"

Returns a summary of every environment for a tenant. Does not return full feed schemas, field mappings, or rules — use `getEnvironment()` for full detail.

### What the caller sends

| Param | Type | Required | What it means |
|---|---|---|---|
| `tenant_id` | string | yes | The tenant to list environments for. |
| `custodian_id` | string | no | Filter to environments for a specific custodian. |

### Example

**Request:** `GET /breakroom/listEnvironments?tenant_id=common-fund`

**Response:**
```jsonc
{
  "tenant_id": "common-fund",
  "environments": [
    {
      "environment_id": "env-cf-cnb-001",
      "environment_name": "Osyte-CNB-Reconciliation",
      "custodian_id": "cnb",
      "reconciliation_type": "trade_reconciliation",
      "status": "active",
      "created_date": "2026-06-10T14:00:00Z",
      "last_updated": "2026-06-17T09:00:00Z"
    }
  ]
}
```

### `listEnvironments()` output signature

```jsonc
{
  "tenant_id": "string",
  "environments": [
    {
      "environment_id": "string",
      "environment_name": "string",
      "custodian_id": "string",
      "reconciliation_type": "string",
      "status": "active",
      "created_date": "ISO 8601 timestamp",
      "last_updated": "ISO 8601 timestamp"
    }
  ]
}
```

---

## `getEnvironment()`

**Route:** `GET /breakroom/getEnvironment/{environment_id}`

**Purpose:** "Show me exactly how this environment is configured."

Returns the full configuration of a single environment — everything set in `createEnvironment()` or changed since via `updateEnvironment()`.

### What the caller sends

| Param | Type | Required | What it means |
|---|---|---|---|
| `environment_id` | string | yes | The environment to retrieve (in the URL path). |

### `getEnvironment()` output signature

```jsonc
{
  "environment_id": "string",
  "environment_name": "string",
  "tenant_id": "string",
  "custodian_id": "string",
  "reconciliation_type": "string",
  "status": "active",
  "created_date": "ISO 8601 timestamp",
  "last_updated": "ISO 8601 timestamp",
  "internal_feed": "object (same shape as createEnvironment() input)",
  "external_feed": "object (same shape as createEnvironment() input)",
  "field_mapping": "object[] (same shape as createEnvironment() input)",
  "filter_rules": "object[] (same shape as createEnvironment() input)",
  "normalization_rules": "object[] (same shape as createEnvironment() input)",
  "composite_key": ["string"] | null,
  "tolerance_rules": "object[] (same shape as createEnvironment() input)"
}
```

---

## `updateEnvironment()`

**Route:** `POST /breakroom/updateEnvironment`

**Purpose:** "Change this environment's config without rebuilding it from scratch."

Accepts the same fields as `createEnvironment()`. Only fields included in the request are changed — omitted fields keep their current value. Past reconciliations stay linked to the config that was active when they ran. `tenant_id` and `custodian_id` are immutable.

### What the caller sends

| Param | Type | Required | What it means |
|---|---|---|---|
| `environment_id` | string | yes | The environment to update. |
| *(any field from `createEnvironment()` except `tenant_id` and `custodian_id`)* | varies | no | Only the fields to change. |

### Example — Adding a new filter rule

**Request:**
```jsonc
POST /breakroom/updateEnvironment

{
  "environment_id": "env-cf-cnb-001",
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
      "values": [15, 16, 18]
    }
  ]
}
```

**Response:**
```jsonc
{
  "environment_id": "env-cf-cnb-001",
  "environment_name": "Osyte-CNB-Reconciliation",
  "status": "active",
  "fields_updated": ["filter_rules"],
  "last_updated": "2026-06-18T10:15:00Z"
}
```

### `updateEnvironment()` output signature

```jsonc
{
  "environment_id": "string",
  "environment_name": "string",
  "status": "active",
  "fields_updated": ["string"],
  "last_updated": "ISO 8601 timestamp"
}
```

---

## `runReconciliation()`

**Route:** `POST /breakroom/runReconciliation`

**Purpose:** "Do these two feeds agree?"

Uploads both feeds against a configured environment and runs the full processing pipeline in a single synchronous call. Returns the complete result as JSON, which the client renders into the output file.

### What the caller sends

| Param | Type | Required | What it means |
|---|---|---|---|
| `environment_id` | string | yes | The active environment to run against. |
| `internal_feed_data` | string | yes | Base64-encoded CSV content for the Osyte feed. Must conform to the feed schema defined in `createEnvironment()`. |
| `external_feed_data` | string | yes | Base64-encoded CSV content for the custodian feed. Must conform to the external feed schema. |

For large files, multipart/form-data upload is also accepted using the same field names.

### Processing pipeline

M1 runs checks 1–5; check 6 is added in M2 once normalization rules are introduced. Any failure stops the run and returns `status: failed`:

| Step | Check | Available from | Failure code |
|---|---|---|---|
| 1 | Feed Received Check — confirms the feed was received and is non-empty | M1 | `feed_not_received` |
| 2 | Feed Format Check — confirms CSV structure is well-formed | M1 | `feed_format_error` |
| 3 | Feed Failed Check — confirms the file is readable and not corrupted | M1 | `feed_file_error` |
| 4 | Feed Field Check — confirms all mandatory fields are present | M1 | `missing_feed_fields` |
| 5 | Feed Field Data Type Check — confirms field values match configured data types | M1 | `data_type_mismatch` |
| 6 | Feed Formatting Service Check — applies filter rules (both feeds), duplicate check (both feeds), then normalization rules (custodian feed) | M2 | `normalization_error` |

If all configured checks pass, record validation begins:
1. Each custodian record is assigned a system-generated `record_id`. `record_id` values are globally unique within a reconciliation across both feeds.
2. **Filter rules** drop records flagged for exclusion by the configured `filter_rules`. These records are assigned a `record_id` and appear in the `records` array with `reconciliation_status: "filtered"`.
3. **Duplicate check (custodian feed):** if 2 or more custodian records share the same composite key — regardless of whether their non-key values match or differ — all of them are classified as `duplicate`. No record is prioritized or passed through.
4. **Duplicate check (Osyte feed):** the duplicate check also runs on the Osyte feed. If 2 or more Osyte records share the same composite key with identical field values (exact duplicates), the first record is excluded and the latest record participates in matching. Handling of non-exact Osyte duplicates (same composite key, different field values) is an engine implementation detail and does not surface in the response.
5. **Key lookup:** the composite key of each remaining custodian record is looked up against the Osyte feed. No match → `unmatch`. Match found → proceed to field comparison.
6. **Field comparison:** compare all mapped non-key fields within tolerance. Any field exceeds tolerance → `partial_match`. All fields within tolerance → `auto_match`.

**Note on normalization failures:** A *hard failure* — the normalization service cannot execute (e.g. a mapping entry is malformed, a date value cannot be parsed) — returns `normalization_error` at check 6 and stops the reconciliation. A *soft failure* — the service runs but values are semantically wrong (e.g. a custodian value has no matching `from` entry in the inline mappings) — does not stop the reconciliation. Affected records will classify as `unmatch` or `partial_match`. If the unmatch rate is unexpectedly high, review the inline mapping configuration in the environment.

### Example — End-of-day reconciliation, Common Fund / CNB

Five CNB records vs. four Osyte trades. Two match exactly. One has a price difference beyond tolerance. One CNB record resolves to no known Osyte account. One CNB record is cancelled and filtered out.

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
11001,   234644,     BTO,                        5/22/2026,  5/15/2026 4:51,    5/26/2026,     214803.848,     257.6500,            55344211.4500
11001,   234645,     STC,                        5/22/2026,  5/15/2026 9:30,    5/26/2026,     100000.000,     150.2500,            -15025000.0000
11002,   235001,     BTO,                        5/22/2026,  5/16/2026 10:00,   5/26/2026,     50000.000,      100.0000,            5000000.0000
11003,   235002,     STC,                        5/22/2026,  5/17/2026 11:00,   5/26/2026,     75000.000,      200.0000,            -15000000.0000
```

External feed (decoded, 5 records):
```
Account #,   Date (Trade), Transaction Type, Quantity,    Price,    Net Amount,   Security ID, Cancel
NYK-003640,  5/22/2026,    buy,              214803.848,  257.6500, 55344211.45,  AVGO,        No
NYK-003640,  5/22/2026,    Sell,             -100000,     150.2500, -15025000.00, AAPL,        No
NYK-003641,  5/22/2026,    Buy,              50000,       100.0500, 5002500.00,   MSFT,        No
NYK-003999,  5/22/2026,    Sell,             30000,       180.0000, -5400000.00,  TSLA,        No
NYK-003640,  5/21/2026,    Sell,             100,         50.0000,  5000.00,      PYPL,        Yes
```

```
Filter rules:
  External — records 1-4 pass include_buy_sell_only; none are cancelled.
  External — record 5: Cancel=Yes → excluded by exclude_cancelled → reconciliation_status: filtered.
  Internal — no records have trade_item_status_id 15 or 16 → 0 excluded.

Normalization (custodian feed, records 1-4):
  TXT-01/TXT-04: "buy"→"BUY"→"BTO", "Sell"/"Buy"→"STC"/"BTO"
  TXT-04: NYK-003640→11001, AVGO→234644, MSFT→235001 via inline mappings
  NUM-03: -100000→100000 (Quantity sign removed for sells)
  DT-03:  "5/22/2026"→"2026-05-22"

Duplicate check: no duplicate composite keys in records 1-4.

Classification:
  Record 1: key (11001, 234644, 2026-05-22, BTO) → match. All fields within tolerance → AUTO_MATCH.
  Record 2: key (11001, 234645, 2026-05-22, STC) → match. All fields within tolerance → AUTO_MATCH.
  Record 3: key (11002, 235001, 2026-05-22, BTO) → match.
    Price: 100.0500 vs 100.0000 → diff 0.05%, tolerance 0.0001% → exceeds → PARTIAL_MATCH.
  Record 4: NYK-003999 → no matching "from" entry in Account # inline mappings → no key match → UNMATCH.
  Record 5: Cancel=Yes → FILTERED (before matching).
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
    "external_records": 5,
    "auto_match": 2,
    "partial_match": 1,
    "unmatch": 1,
    "duplicate": 0,
    "filtered": 1
  },

  "basic_validation": [
    {
      "feed_type": "custodian",
      "feed_name": "CNB Records",
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
    }
  ],

  "records": [
    {
      "record_id": "REC-001",
      "reconciliation_status": "auto_match",
      "external_record": {
        "Account #": "NYK-003640", "Date (Trade)": "5/22/2026", "Transaction Type": "buy",
        "Quantity": 214803.848, "Price": 257.6500, "Net Amount": 55344211.45,
        "Security ID": "AVGO", "Cancel": "No"
      },
      "matched_internal_record_ref": "610989",
      "field_comparison": [ /* all fields match within tolerance */ ]
    },
    {
      "record_id": "REC-002",
      "reconciliation_status": "auto_match",
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
        {"field": "fund_id / Account #",              "internal": "11002",         "external_normalized": "11002",        "match": true,  "tolerance_applied": null},
        {"field": "account_id / Security ID",         "internal": "235001",        "external_normalized": "235001",       "match": true,  "tolerance_applied": null},
        {"field": "trade_dt / Date (Trade)",          "internal": "2026-05-22",    "external_normalized": "2026-05-22",   "match": true,  "tolerance_applied": null},
        {"field": "trade_transaction_type_cd / Type", "internal": "BTO",           "external_normalized": "BTO",          "match": true,  "tolerance_applied": null},
        {"field": "avg_price_per_share / Price",      "internal": "100.0000",      "external_normalized": "100.0500",     "match": false, "tolerance_applied": {"type": "percentage",    "limit": 0.0001, "actual": 0.0005}},
        {"field": "net_cash_amt / Net Amount",        "internal": "5000000.0000",  "external_normalized": "5002500.0000", "match": false, "tolerance_applied": {"type": "absolute",      "limit": 1.00,   "actual": 2500.00}},
        {"field": "final_quantity / Quantity",        "internal": "50000.000000",  "external_normalized": "50000.000000", "match": true,  "tolerance_applied": null},
        {"field": "trade_entered_dt / Date (Entry)",  "internal": "2026-05-16",    "external_normalized": "2026-05-16",   "match": true,  "tolerance_applied": {"type": "business_days", "limit": 1, "actual": 0}},
        {"field": "settlement_dt / Date (Settle)",    "internal": "2026-05-26",    "external_normalized": "2026-05-26",   "match": true,  "tolerance_applied": {"type": "directional",   "direction": "lte", "result": "pass"}}
      ]
    },
    {
      "record_id": "REC-004",
      "reconciliation_status": "unmatch",
      "external_record": {
        "Account #": "NYK-003999", "Date (Trade)": "5/22/2026", "Transaction Type": "Sell",
        "Quantity": 30000, "Price": 180.0000, "Net Amount": -5400000.00, "Security ID": "TSLA"
      },
      "matched_internal_record_ref": null,
      "field_comparison": null
    },
    {
      "record_id": "REC-005",
      "reconciliation_status": "filtered",
      "external_record": {
        "Account #": "NYK-003640", "Date (Trade)": "5/21/2026", "Transaction Type": "Sell",
        "Quantity": 100, "Price": 50.0000, "Net Amount": 5000.00, "Security ID": "PYPL",
        "Cancel": "Yes"
      },
      "matched_internal_record_ref": null,
      "field_comparison": null
    }
  ]
}
```

Note on duplicate records: when a record is classified as `duplicate`, `matched_internal_record_ref` still carries the internal record reference if a key match was found — `duplicate` means another custodian record shares this composite key, not that no Osyte counterpart exists.

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
    "auto_match": "int",
    "partial_match": "int",
    "unmatch": "int",
    "duplicate": "int",
    "filtered": "int"
  },

  "basic_validation": [
    {
      "feed_type": "custodian | internal",
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
      "reconciliation_status": "auto_match | partial_match | unmatch | duplicate | filtered",
      "external_record": "object (custodian field values as received)",
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

If status is `failed`, `basic_validation` is populated and `records` is empty. The first failed check is the termination point. `matched_internal_record_ref` is populated for `auto_match`, `partial_match`, and `duplicate` records; `null` for `unmatch` and `filtered`. `field_comparison` is populated for `auto_match`, `partial_match`, and `duplicate` records; `null` for `unmatch` and `filtered`. `delivery_type` and `file_number` in `basic_validation` are system-defaulted to `"upload"` and `"#1"` in v1.

**Summary count invariant:** `external_records = auto_match + partial_match + unmatch + duplicate + filtered`.

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
| `invalid_field_mapping` | 422 | A `field_mapping` entry references a label not defined in either feed | `createEnvironment`, `updateEnvironment` |
| `invalid_composite_key` | 422 | A field in `composite_key` is not present in `field_mapping` | `createEnvironment`, `updateEnvironment` |
| `invalid_tolerance_rule` | 422 | A field in `tolerance_rules` is not a mapped field, or `tolerance_type` / `direction` is invalid | `createEnvironment`, `updateEnvironment` |
| `invalid_filter_operator` | 422 | A filter rule uses an operator incompatible with the field's `data_type` (e.g. `greater_than` on a text field) | `createEnvironment`, `updateEnvironment` |
| `environment_not_found` | 404 | No environment found for the given `environment_id` | `getEnvironment`, `updateEnvironment`, `runReconciliation` |
| `tenant_not_found` | 404 | No environments found for the given `tenant_id` | `listEnvironments` |
| `feed_not_received` | 422 | A feed was not received or is empty | `runReconciliation` |
| `feed_format_error` | 422 | Uploaded feed is not a valid CSV | `runReconciliation` |
| `feed_file_error` | 422 | The file is unreadable or corrupted | `runReconciliation` |
| `missing_feed_fields` | 422 | One or more mandatory fields are absent from the uploaded feed | `runReconciliation` |
| `data_type_mismatch` | 422 | A field value does not match its configured `data_type` | `runReconciliation` |
| `normalization_error` | 422 | A normalization rule failed to execute (e.g. malformed mapping entry, unparseable date) | `runReconciliation` |

Note: when a basic validation check fails, the response body still returns with `status: failed` and the specific check marked `failed` in `basic_validation`. The HTTP-level errors above are for request-level rejections (malformed JSON, missing required parameters, unknown environment).