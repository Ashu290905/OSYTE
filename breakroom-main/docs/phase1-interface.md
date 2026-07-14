# Breakroom Phase 1 — Interfaces & Signatures

## Who builds what, who uses what

| Module | Built by | Used by |
|---|---|---|
| `models.py` | Aarohi | All endpoints (all phases), `config_store/store.py` (Aarohi), `reconciliation_engine/pipeline.py` (Ashutosh), all engine modules (Ashutosh), `ai_inference/` (all three) |
| `config_store/store.py` | Aarohi | Config API endpoints (all three), `run_reconciliation.py` (Mihir, Phase 3) |
| `config_api/create_recon_config.py` | Aarohi | API consumers |
| `config_api/update_recon_config.py` | Aarohi | API consumers |
| `config_api/list_recon_configs.py` | Mihir | API consumers |
| `config_api/get_recon_config.py` | Ashutosh | API consumers |
| `feed_processor/parser.py` | Mihir | `run_reconciliation.py`, `ai_inference/analyzer.py` (Phase 4) |
| `feed_processor/merger.py` | Mihir | `run_reconciliation.py`, `ai_inference/analyzer.py` (Phase 4) |
| `feed_processor/validator.py` | Mihir | `run_reconciliation.py` |
| `reconciliation_engine/normaliser.py` | Ashutosh | `reconciliation_engine/pipeline.py` (Phase 2+) |
| `reconciliation_engine/filter_engine.py` | Ashutosh | `reconciliation_engine/pipeline.py` (Phase 2+) |
| `reconciliation_api/run_reconciliation.py` (M1 path) | Mihir | API consumers |

**Lock by end of Day 1:**
- `models.py` shared dataclasses and custom exceptions — used by all modules from Phase 2 onward
- `parse_csv()` signature — used by `run_reconciliation.py` (Phase 1) and `ai_inference/analyzer.py` (Phase 4)
- `BasicValidationResult` dataclass — assembled into response by `run_reconciliation.py`
- `save_config()` / `get_config()` signatures — called by Mihir's `run_reconciliation.py` and Ashutosh's `getReconConfig()`

---

## Interfaces

```python
# Aarohi: models.py
# Shared dataclasses for all 5 methods and all modules.
# Pydantic request/response models for all 5 API methods.
# Custom exceptions: FeedFormatError, ConfigNotFoundError, DuplicateConfigError,
#                    AIAnalysisFailedError, NormalizationError
# Lock by end of Day 1.

# Aarohi: config_store/store.py
save_config(config: ReconConfig) -> str
# Serialises full ReconConfig to SQLite JSON blob. Generates UUID config_id.
# Sets created_date + last_updated to now. Returns config_id.
# Raises DuplicateConfigError if (config_name, tenant_id) already exists.

get_config(config_id: str) -> ReconConfig
# Deserialises from SQLite JSON blob. Raises ConfigNotFoundError if not found.

list_configs(tenant_id: str) -> list[ReconConfigSummary]
# Returns lightweight summaries ordered by created_date descending.

update_config(config_id: str, updates: dict) -> ReconConfig
# Partial update. For external_feeds: match by source_id — only included sources change.
# For composite_key, tolerance_rules, internal_feed: replace entirely when provided.
# tenant_id immutable — raise ValueError if update attempted.
# Updates last_updated. Raises ConfigNotFoundError if not found.

# Aarohi: config_api/create_recon_config.py (manual mode)
create_recon_config_manual(request: CreateReconConfigManualRequest) -> CreateReconConfigResponse
# Validates input JSON. Applies system defaults for composite_key and tolerance_rules
# if not provided. Saves via config_store.save_config().
# Returns CreateReconConfigResponse (full config + review_notes=[]), creation_mode="manual".
# Validates: field_mapping references, composite_key fields exist, tolerance targets
# are mapped fields, filter operators match field data_type, unique source_ids,
# and TXT-01/TXT-04 casing consistency (any field with both rules must have every
# TXT-04 from_value equal to its own uppercase form) → invalid_field_mapping if violated.

# Aarohi: config_api/update_recon_config.py
update_recon_config(request: UpdateReconConfigRequest) -> UpdateReconConfigResponse
# Extracts config_id + provided fields from the request, calls config_store.update_config()
# with only the fields the caller set. Returns UpdateReconConfigResponse
# {config_id, config_name, status, fields_updated, last_updated}.
# Rejects tenant_id changes (ValueError → 422). Applies the same TXT-01/TXT-04 casing
# validation as create when normalization_rules are updated.

# Mihir: config_api/list_recon_configs.py
list_recon_configs(tenant_id: str) -> ListReconConfigsResponse
# Calls config_store.list_configs(). Empty list → 404 tenant_not_found.
# Returns {tenant_id, configs: list[ReconConfigSummary]}.

# Ashutosh: config_api/get_recon_config.py
get_recon_config(config_id: str) -> ReconConfig
# Calls config_store.get_config(). Propagates ConfigNotFoundError → 404.

# Mihir: feed_processor/parser.py
parse_csv(file_data: str) -> list[dict]
# Decode base64, parse CSV, return list of row dicts keyed by column headers.
# Values are raw strings — no type conversion.
# Raises FeedFormatError if base64 decode fails, CSV is malformed, or file is empty.

# Mihir: feed_processor/merger.py
merge_files(files: list[str]) -> list[dict]
# Parse each base64 CSV via parse_csv(), tag each row with _file_index (1-based).
# Concatenate all rows into single flat list.
# Raises FeedFormatError if any file fails parse_csv().

# Mihir: feed_processor/validator.py
run_basic_validation(
    rows: list[dict],
    field_schema: list[FieldSchema],
    file_index: int,
    feed_name: str,
    source_id: str | None,   # None for internal feed; "cnb", "boa" etc for external
    feed_type: str | None,   # "internal" for Osyte feed; None for external sources
) -> BasicValidationResult
# Runs checks 1–5 in order. Stops at first failure.
# file_index: 1-based integer passed by caller — NOT extracted from _file_index tags.

# Mihir: reconciliation_api/run_reconciliation.py (M1 path only in Phase 1)
run_reconciliation_m1(request: RunReconciliationRequest) -> RunReconciliationResponse
# M1 scope: basic validation only. No pipeline or matching.
# 1. Fetch config via config_store.get_config()
# 2. Decode all feeds; parse each file via parse_csv()
# 3. Multi-file merge for sources with multiple files via merge_files()
# 4. Run run_basic_validation() per file per feed
# 5. Assemble: summary (external_records totals per source), basic_validation list
# 6. Return response with status="completed" if all validations pass
# Raises config_not_found, source_not_found, feed_not_received, feed_format_error,
# feed_file_error, missing_feed_fields, data_type_mismatch as appropriate.

# Ashutosh: reconciliation_engine/normaliser.py (foundational for Phase 2 wiring)
normalise_record(record: dict, rules: list[NormalizationRule]) -> dict
# See Phase 2 interfaces for full behaviour spec.

# Ashutosh: reconciliation_engine/filter_engine.py (foundational for Phase 2 wiring)
apply_filter_rules(records: list[dict], rules: list[FilterRule], feed: str) -> FilterResult
# See Phase 2 interfaces for full behaviour spec.
```

**Import note:** `CheckResult` and `BasicValidationResult` are defined in `feed_processor/validator.py` (Mihir). `run_reconciliation.py` imports them from there. If circular imports arise, move both to `models.py` — decide on Day 1.

**Phase 3 forward reference:** `PipelineResult` (returned by `pipeline.py` M3 extension) will be locked in Phase 3 interfaces.

---

## Signatures

### Custom exceptions — `models.py` (Aarohi, lock Day 1)

```python
class FeedFormatError(Exception):
    """Raised by feed_processor/parser.py and merger.py.
    Maps to HTTP 422 feed_format_error / feed_file_error at endpoint."""
    pass

class ConfigNotFoundError(Exception):
    """Raised by config_store/store.py.
    Maps to HTTP 404 config_not_found at endpoint."""
    pass

class DuplicateConfigError(Exception):
    """Raised by config_store/store.py save_config().
    (config_name, tenant_id) combination already exists."""
    pass

class AIAnalysisFailedError(Exception):
    """Raised by ai_inference/analyzer.py (Phase 4).
    Maps to HTTP 422 ai_analysis_failed."""
    pass

class NormalizationError(Exception):
    """Raised by reconciliation_engine/normaliser.py (Phase 2) on hard failure only.
    Soft failures (TXT-04 miss, NUM-04 flag) do NOT raise this exception.
    Maps to HTTP 422 normalization_error at endpoint.
    Stops the run for that source; other sources continue independently."""
    pass
```

---

### Internal row-annotation keys (convention — lock Day 1)

Row dicts flow through parser → merger → validator → engine. Modules annotate rows with reserved keys prefixed `_`. These are pipeline metadata, never feed columns: validation ignores them (checks 4–5), matching ignores them when building composite keys, and they are stripped before a row's values are placed in a response `external_record`. All four are defined here so no two modules invent competing names.

| Key | Added by | Phase | Meaning |
|---|---|---|---|
| `_file_index` | `merger.merge_files()` | 1 | 1-based index of the source file a row came from; drives `file_number` in `BasicValidationResult` |
| `_filter_rule` | `filter_engine.apply_filter_rules()` | 2 | name of the rule that excluded this row; present only on excluded rows |
| `_is_duplicate` | `duplicate_checker.check_duplicates_source()` | 2 | `True` on every row in a source composite-key group of ≥2 |
| `_exclude` | `normaliser.normalise_record()` (NUM-04) | 2 | `True` when a Price field normalised to 0 → row becomes `filtered` |

`external_record` in the response always reflects the row **as received** (original values, no `_`-keys). The classifier reads `_is_duplicate`, `_exclude`, and `_filter_rule` to assign status, then those keys are dropped from the emitted record.

---

```python
from dataclasses import dataclass
from datetime import date, datetime
from typing import Any

@dataclass
class FieldSchema:
    label: str
    data_type: str                      # "numeric" | "text" | "date"
    format: str                         # e.g. "DECIMAL(18,4)", "VARCHAR(30)", "DATE (M/D/YYYY)"
    mandatory: bool

@dataclass
class FieldMapping:
    internal_field: str
    external_field: str

@dataclass
class FilterRule:
    rule_name: str
    feed: str                           # "internal" | "external"
    field_label: str
    action: str                         # "include" | "exclude"
    operator: str                       # text: equals/not_equals/in/not_in/contains/starts_with (case-insensitive)
                                        # numeric: equals/not_equals/in/not_in/greater_than/less_than/between
                                        # date: equals/not_equals/before/after/between
    values: list                        # for "between": exactly 2 elements [lower, upper]

@dataclass
class MappingEntry:
    from_value: str                     # JSON key is "from" — field_alias="from" in Pydantic model
    to_value: str                       # JSON key is "to"
    org_id: str | None = None

@dataclass
class NormalizationRule:
    rule_id: str                        # "TXT-01" through "DT-05" — DT-04 does NOT exist
    target_fields: list[str]
    mappings: list[MappingEntry] | None = None  # TXT-04 only

@dataclass
class ToleranceRule:
    field_label: str
    tolerance_type: str                 # "absolute" | "percentage" | "business_days" | "directional"
    tolerance_value: float | None = None  # required for absolute, percentage, business_days
                                          # percentage: 0.0001 means 0.0001%, NOT the fraction
                                          # absolute: value in native unit (1.00 = $1.00, NOT $0.01)
    direction: str | None = None        # "lte" — required for directional only

@dataclass
class InternalFeed:
    feed_name: str
    fields: list[FieldSchema]

@dataclass
class ExternalFeed:
    source_id: str
    feed_name: str
    fields: list[FieldSchema]
    field_mapping: list[FieldMapping]
    filter_rules: list[FilterRule]
    normalization_rules: list[NormalizationRule]

@dataclass
class ReconConfig:
    config_id: str
    config_name: str
    tenant_id: str
    reconciliation_type: str            # "trade_reconciliation" in v1
    status: str                         # "active"
    creation_mode: str                  # "ai_assisted" | "manual"
    internal_feed: InternalFeed
    external_feeds: list[ExternalFeed]
    composite_key: list[str]            # system default: ["fund_id","account_id","trade_dt","trade_transaction_type_cd"]
    tolerance_rules: list[ToleranceRule]  # system defaults applied if not set
    created_date: datetime
    last_updated: datetime

@dataclass
class ReconConfigSummary:
    config_id: str
    config_name: str
    reconciliation_type: str
    sources: list[str]                  # list of source_ids
    status: str
    creation_mode: str                  # "ai_assisted" | "manual"
    created_date: datetime
    last_updated: datetime

@dataclass
class ToleranceEvaluation:
    tolerance_type: str
    match: bool
    limit: float | None                 # None for directional
    actual: float | None                # None for directional
    direction: str | None               # "lte" for directional
    result: str | None                  # "pass" | "fail" for directional

@dataclass
class FieldComparison:
    field: str                          # "internal_label / external_label"
    internal: str
    external_normalized: str
    match: bool
    tolerance_applied: ToleranceEvaluation | None

@dataclass
class ReconciliationRecord:
    # Defined at Day 1 lock so the full config/response surface is stable,
    # but only populated from Phase 3 (M3) onward — M1/M2 responses omit the records array.
    record_id: str                      # unique within the reconciliation run
    source_id: str
    reconciliation_status: str          # "auto_match" | "partial_match" | "unmatch" | "duplicate" | "filtered"
    external_record: dict               # original source field values as received (not normalised)
    matched_internal_record_ref: str | None  # populated for auto_match, partial_match, duplicate
                                             # null for unmatch, filtered
    field_comparison: list[FieldComparison] | None  # populated for auto_match, partial_match, duplicate
                                                     # null for unmatch, filtered
```

---

### Request / response envelope models — `models.py` (Aarohi, lock Day 1)

These are the Pydantic models at the API boundary. Every endpoint references them, so they must be locked Day 1 alongside the dataclasses. Field names here are the exact JSON keys the contract specifies.

```python
from pydantic import BaseModel, Field

# ---------- createReconConfig ----------

class SourceFeedManualInput(BaseModel):
    source_id: str
    feed_name: str
    fields: list[FieldSchema]
    field_mapping: list[FieldMapping]
    filter_rules: list[FilterRule] = []
    normalization_rules: list[NormalizationRule] = []

class CreateReconConfigManualRequest(BaseModel):
    config_name: str
    tenant_id: str
    reconciliation_type: str = "trade_reconciliation"
    internal_feed: InternalFeed
    external_feeds: list[SourceFeedManualInput]
    composite_key: list[str] | None = None       # omitted → system default applied
    tolerance_rules: list[ToleranceRule] | None = None  # omitted → system default applied

class SourceFeedDataInput(BaseModel):
    # Used by createReconConfig AI mode, where feed_name is being established.
    source_id: str
    feed_name: str
    files: list[str]                              # base64-encoded CSVs

class SourceRunInput(BaseModel):
    # Used by runReconciliation, where feed_name already lives in the stored config.
    source_id: str
    files: list[str]                              # base64-encoded CSVs

class CreateReconConfigAIRequest(BaseModel):
    config_name: str
    tenant_id: str
    reconciliation_type: str = "trade_reconciliation"
    internal_feed_data: str                       # base64-encoded Osyte CSV
    external_feeds_data: list[SourceFeedDataInput]

# createReconConfig response = full ReconConfig + review_notes.
class CreateReconConfigResponse(BaseModel):
    config: ReconConfig
    review_notes: list[str]                       # [] in manual mode; populated in AI mode

# The endpoint accepts EITHER shape. Presence of internal_feed_data/external_feeds_data
# → AI mode; presence of internal_feed/external_feeds → manual mode. Both present →
# invalid_input_mode (422). Neither present → invalid_input_mode (422).

# ---------- updateReconConfig ----------

class UpdateReconConfigRequest(BaseModel):
    config_id: str
    # Any subset of the mutable ReconConfig fields. tenant_id is rejected if present.
    config_name: str | None = None
    reconciliation_type: str | None = None
    internal_feed: InternalFeed | None = None
    external_feeds: list[SourceFeedManualInput] | None = None
    composite_key: list[str] | None = None
    tolerance_rules: list[ToleranceRule] | None = None

class UpdateReconConfigResponse(BaseModel):
    config_id: str
    config_name: str
    status: str                                   # "active"
    fields_updated: list[str]                     # dotted paths, e.g. "external_feeds.cnb.field_mapping"
    last_updated: datetime

# ---------- listReconConfigs ----------

class ListReconConfigsResponse(BaseModel):
    tenant_id: str
    configs: list[ReconConfigSummary]

# getReconConfig returns a bare ReconConfig (no envelope, no review_notes).

# ---------- runReconciliation ----------

class RunReconciliationRequest(BaseModel):
    config_id: str
    internal_feed_data: str                       # base64-encoded Osyte CSV
    external_feeds_data: list[SourceRunInput]

class SourceSummary(BaseModel):
    source_id: str
    external_records: int
    auto_match: int
    partial_match: int
    unmatch: int
    duplicate: int
    filtered: int

class ReconciliationSummary(BaseModel):
    total: SourceSummary                          # source_id field set to "total"
    by_source: list[SourceSummary]

class RunReconciliationResponse(BaseModel):
    reconciliation_id: str
    config_id: str
    config_name: str
    reconciliation_type: str
    status: str                                   # "completed" | "failed"
    start_date: datetime
    end_date: datetime
    summary: ReconciliationSummary
    basic_validation: list[BasicValidationResult]
    records: list[ReconciliationRecord]           # empty until M3
```

**Envelope note:** `BasicValidationResult`, `CheckResult`, `FilterResult`, `DuplicateCheckResult`, `Check6Result`, and `ToleranceEvaluation` are internal (not Pydantic request/response models) — see the module that owns each. Only the models above cross the API boundary.

---

```python
def save_config(
    config: ReconConfig,
) -> str:
    # Generates UUID config_id if not already set
    # Sets created_date + last_updated to datetime.utcnow()
    # Serialises full ReconConfig to JSON blob, inserts into recon_configs SQLite table
    # Schema enforces a UNIQUE constraint on (config_name, tenant_id); the insert
    #   catches the integrity error and re-raises as DuplicateConfigError
    # Returns config_id
    # Raises DuplicateConfigError if (config_name, tenant_id) row already exists

def get_config(
    config_id: str,
) -> ReconConfig:
    # SELECT from recon_configs WHERE config_id = ?
    # Deserialise JSON blob → ReconConfig
    # Raises ConfigNotFoundError if no row found

def list_configs(
    tenant_id: str,
) -> list[ReconConfigSummary]:
    # SELECT all rows WHERE tenant_id = ?
    # Return as ReconConfigSummary (no full feed schemas or rules)
    # Ordered by created_date DESC
    # Returns empty list if no configs exist for tenant_id (caller decides on 404)

def update_config(
    config_id: str,
    updates: dict,                      # top-level keys map to ReconConfig fields
) -> ReconConfig:
    # Fetch existing config via get_config() (raises ConfigNotFoundError if missing)
    # Apply updates:
    #   Scalar fields (config_name, reconciliation_type, status): overwrite directly
    #   internal_feed (object): replace entirely when provided
    #   composite_key (list): replace entirely when provided
    #   tolerance_rules (list): replace entirely when provided
    #   external_feeds (list): match each provided entry by source_id
    #     — found source: merge provided sub-fields only, leave others unchanged
    #     — not found source: leave unchanged (do NOT delete)
    #   tenant_id: immutable, raise ValueError if caller attempts to update
    # Set last_updated = datetime.utcnow()
    # Save updated config back to SQLite
    # Return full updated ReconConfig
```

---

### Mihir: `feed_processor/parser.py`

```python
def parse_csv(
    file_data: str,                     # base64-encoded CSV string
) -> list[dict]:
    # 1. Decode base64 → bytes → UTF-8 string
    # 2. Parse with csv.DictReader — first row is headers
    # 3. Return list of row dicts; all values remain as raw strings
    # Raises FeedFormatError if:
    #   — base64 decode fails
    #   — result is empty after stripping whitespace
    #   — csv.DictReader produces no rows (empty file)
    #   — CSV has no headers
    #   — malformed quoting or inconsistent column count
```

---

### Mihir: `feed_processor/merger.py`

```python
def merge_files(
    files: list[str],                   # list of base64-encoded CSV strings, same source
) -> list[dict]:
    # For each file at index i (0-based):
    #   rows = parse_csv(files[i])
    #   for each row: row["_file_index"] = i + 1   # 1-based for file_number in response
    # Concatenate all tagged rows into single flat list
    # Raises FeedFormatError if any individual file fails parse_csv()
```

---

### Mihir: `feed_processor/validator.py`

```python
@dataclass
class CheckResult:
    sno: int                            # 1-based check number (1–6)
    validation: str                     # exact name from contract:
                                        # 1: "Feed Received Check"
                                        # 2: "Feed Format Check"
                                        # 3: "Feed Failed Check"     ← NOT "Feed File Check"
                                        # 4: "Feed Field Check"
                                        # 5: "Feed Field Data Type Check"
                                        # 6: "Feed Formatting Service Check" (Phase 2+)
    status: str                         # "passed" | "failed"
    break_type: str | None
    break_description: str | None

@dataclass
class BasicValidationResult:
    feed_type: str | None               # "internal" for Osyte feed; None for external sources
    source_id: str | None               # None for internal; "cnb", "boa" etc for external
    feed_name: str
    delivery_type: str                  # always "upload" in v1 (system default)
    file_number: str                    # "#1", "#2" etc — set from file_index parameter
    received_date: datetime             # set to datetime.utcnow() at start of validation
    processed_date: datetime            # set to datetime.utcnow() at end of validation
    feed_status: str                    # "completed" | "failed"
    checks: list[CheckResult]           # only includes checks that were run (stops at first failure)

def run_basic_validation(
    rows: list[dict],
    field_schema: list[FieldSchema],
    file_index: int,                    # 1-based integer passed by caller
    feed_name: str,
    source_id: str | None,
    feed_type: str | None,
) -> BasicValidationResult:
    # Check 1 — Feed Received Check: len(rows) > 0
    # Check 2 — Feed Format Check: well-formed (guaranteed by parser — mark passed)
    # Check 3 — Feed Failed Check: readable (guaranteed by parser — mark passed)
    # Check 4 — Feed Field Check: all mandatory fields present as keys in rows[0]
    # Check 5 — Feed Field Data Type Check: mandatory values match declared data_type
    #     numeric → parseable as float
    #     date → parseable in common formats (M/D/YYYY, YYYY-MM-DD, MM-DD-YYYY, etc.)
    #     text → always passes
    # Internal row-annotation keys (any key starting with "_", e.g. _file_index) are
    #   ignored by checks 4 and 5 — they are pipeline metadata, not feed columns.
    # Stop at first failure — do not run subsequent checks
    # feed_status: "failed" if any failed; "completed" if all ran and passed
    # file_number: f"#{file_index}"
```

---

### Mihir: `reconciliation_api/run_reconciliation.py` (M1 path)

```python
def run_reconciliation_m1(
    request: RunReconciliationRequest,
) -> RunReconciliationResponse:
    # M1 scope: basic validation only. No pipeline or matching.
    # 1. Fetch config via config_store.get_config(request.config_id)
    #    → raises ConfigNotFoundError → 404 config_not_found
    # 2. Validate each source_id in request.external_feeds_data exists in config.external_feeds
    #    → 404 source_not_found if unknown
    # 3. Decode internal_feed_data (base64) → parse_csv() → list[dict]
    # 4. For each source in request.external_feeds_data:
    #    — files_data = source.files (list of base64 CSVs)
    #    — merged_rows = merge_files(files_data)
    #    — For each file_index (1-based per source):
    #        rows_for_this_file = [r for r in merged_rows if r["_file_index"] == file_index]
    #        result = run_basic_validation(
    #            rows=rows_for_this_file,
    #            field_schema=source_config.fields,
    #            file_index=file_index,
    #            feed_name=source_config.feed_name,
    #            source_id=source.source_id,
    #            feed_type=None,
    #        )
    #        Append result to basic_validation list
    # 5. Run same validation for internal feed (file_index=1, source_id=None, feed_type="internal")
    # 6. Assemble summary: external_records total per source
    # 7. status="completed" if all validations passed; "failed" if internal feed failed
    #    OR all sources failed. Individual source failures do NOT fail overall status.
    # 8. Return RunReconciliationResponse
```

---

### Ashutosh: `config_api/get_recon_config.py`

```python
def get_recon_config(
    config_id: str,
) -> ReconConfig:
    # Thin wrapper over config_store.get_config()
    # Propagates ConfigNotFoundError → 404 config_not_found
    # Returns full ReconConfig — same shape as createReconConfig() response
    # No review_notes field (that's only in create response)
```