# Breakroom Phase 1 — Interfaces & Signatures

## Who builds what, who uses what

| Module | Built by | Used by |
|---|---|---|
| `models.py` | Aarohi | All endpoints (all phases), `config_store/store.py` (Aarohi), `reconciliation_engine/pipeline.py` (Mihir), all engine modules (Ashutosh), `ai_inference/` (Aarohi) |
| `config_store/store.py` | Aarohi | Config API endpoints (Aarohi), `run_reconciliation.py` (Mihir, Phase 3) |
| `feed_processor/parser.py` | Mihir | `run_reconciliation.py` (Mihir, Phase 3), `ai_inference/analyzer.py` (Aarohi, Phase 4) |
| `feed_processor/merger.py` | Mihir | `run_reconciliation.py` (Mihir, Phase 3), `ai_inference/analyzer.py` (Aarohi, Phase 4) |
| `feed_processor/validator.py` | Mihir | `run_reconciliation.py` (Mihir, Phase 3) |

**Lock by end of Day 1:**
- `models.py` shared dataclasses and custom exceptions — used by all modules from Phase 2 onward
- `parse_csv()` signature — Aarohi's AI inference calls this in Phase 4
- `BasicValidationResult` dataclass — Mihir builds it, Mihir assembles the response in `run_reconciliation.py`
- `save_config()` / `get_config()` signatures — Mihir's `run_reconciliation.py` calls these in Phase 3

---

## Interfaces

```python
# Aarohi: models.py
# Shared dataclasses for all 5 methods and all modules.
# Pydantic request/response models for all 5 API methods.
# Custom exceptions: ConfigNotFoundError, DuplicateConfigError.
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
# Updates last_updated. Raises ConfigNotFoundError if not found.

# Mihir: feed_processor/parser.py
parse_csv(file_data: str) -> list[dict]
# Decode base64, parse CSV, return list of row dicts keyed by column headers.
# Values are raw strings — no type conversion.
# Raises FeedFormatError if base64 decode fails, CSV is malformed, or file is empty.

# Mihir: feed_processor/merger.py
merge_files(files: list[str]) -> list[dict]
# Parse each base64 CSV via parse_csv(), tag each row with _file_index (int, 1-based).
# Concatenate all rows into a single flat list.
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
# file_index: 1-based integer passed by caller — NOT extracted from _file_index row tags.
#             Validator receives rows from parse_csv() (single file); _file_index is merger.py's tag.
# Check 1 — Feed Received Check: rows not empty
# Check 2 — Feed Format Check: well-formed (guaranteed by parser — mark passed)
# Check 3 — Feed Failed Check: readable (guaranteed by parser — mark passed)
# Check 4 — Feed Field Check: all mandatory fields present as keys in rows[0]
# Check 5 — Feed Field Data Type Check: mandatory field values match declared data_type
```

---

## Signatures

### Custom exceptions — `models.py` (Aarohi, lock Day 1)

```python
class FeedFormatError(Exception):
    """Raised by feed_processor/parser.py and merger.py.
    Maps to HTTP 422 feed_format_error / feed_file_error at the endpoint level."""
    pass

class ConfigNotFoundError(Exception):
    """Raised by config_store/store.py.
    Maps to HTTP 404 config_not_found at the endpoint level."""
    pass

class DuplicateConfigError(Exception):
    """Raised by config_store/store.py save_config().
    config_name + tenant_id combination already exists."""
    pass

class AIAnalysisFailedError(Exception):
    """Raised by ai_inference/analyzer.py.
    Maps to HTTP 422 ai_analysis_failed. Used in Phase 4."""
    pass
```

---

### Shared dataclasses — `models.py` (Aarohi, lock Day 1)

```python
from dataclasses import dataclass, field
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
    from_value: str                     # JSON key is "from" — use field_alias="from" in Pydantic model
    to_value: str                       # JSON key is "to"
    org_id: str | None = None

@dataclass
class NormalizationRule:
    rule_id: str                        # "TXT-01" through "DT-05" — note: DT-04 does NOT exist
    target_fields: list[str]
    mappings: list[MappingEntry] | None = None  # TXT-04 only

@dataclass
class ToleranceRule:
    field_label: str
    tolerance_type: str                 # "absolute" | "percentage" | "business_days" | "directional"
    tolerance_value: float | None = None  # required for absolute, percentage, business_days
                                          # percentage: 0.0001 means 0.0001%, NOT the fraction 0.0001
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
    record_id: str                      # unique within the reconciliation run
    source_id: str
    reconciliation_status: str          # "auto_match" | "partial_match" | "unmatch" | "duplicate" | "filtered"
    external_record: dict               # original source field values as received (not normalised)
    matched_internal_record_ref: str | None  # populated for auto_match, partial_match, duplicate
                                             # null for unmatch, filtered
    field_comparison: list[FieldComparison] | None  # populated for auto_match, partial_match, duplicate
                                                     # null for unmatch, filtered
```

**Import note:** `CheckResult` and `BasicValidationResult` are defined in `feed_processor/validator.py` (Mihir). `run_reconciliation.py` (Phase 3, also Mihir) imports them from there: `from feed_processor.validator import BasicValidationResult, CheckResult`. If this creates a circular import, move both to `models.py` — decide and lock on Day 1.

**Phase 3 forward reference:** `PipelineResult` (returned by `pipeline.py` M3 extension) is not defined here — it will be locked in Phase 3 interfaces. It contains `list[ReconciliationRecord]` and summary counts per source.

---

```python
def save_config(
    config: ReconConfig,
) -> str:
    # Generates UUID config_id if not already set
    # Sets created_date + last_updated to datetime.utcnow()
    # Serialises full ReconConfig to JSON blob, inserts into recon_configs SQLite table
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
    # Returns empty list if no configs exist for tenant_id (no error)

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
    # Note: all files for the same source are expected to share column schema
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
    delivery_type: str                  # always "upload" in v1 (system default, not caller-provided)
    file_number: str                    # "#1", "#2" etc — set from file_index parameter in run_basic_validation()
    received_date: datetime             # set to datetime.utcnow() at start of validation
    processed_date: datetime            # set to datetime.utcnow() at end of validation
    feed_status: str                    # "completed" | "failed"
    checks: list[CheckResult]           # only includes checks that were run (stops at first failure)

def run_basic_validation(
    rows: list[dict],                   # parsed rows from parse_csv() — single file only
    field_schema: list[FieldSchema],    # from ReconConfig — defines mandatory fields + data types
    file_index: int,                    # 1-based integer passed by caller (1 for first file, 2 for second, etc.)
                                        # sets file_number in response as "#1", "#2" etc.
                                        # NOT derived from _file_index row tags — those are added by merger.py
                                        # validator receives rows from parse_csv() directly (single file, no tags)
    feed_name: str,
    source_id: str | None,              # None for internal Osyte feed
    feed_type: str | None,              # "internal" for Osyte; None for source feeds
) -> BasicValidationResult:
    # Check 1 — Feed Received Check: len(rows) > 0
    #   Fail: break_type="empty_feed", break_description="No records found in file"
    # Check 2 — Feed Format Check: well-formed CSV
    #   Mark passed if rows arrived (parser already validated structure)
    # Check 3 — Feed Failed Check: file readable
    #   Mark passed if rows arrived (parser already validated readability)
    # Check 4 — Feed Field Check: all FieldSchema where mandatory=True have label in rows[0].keys()
    #   Fail: break_type="missing_field", break_description="Missing mandatory fields: [list]"
    # Check 5 — Feed Field Data Type Check: for each mandatory field, all non-empty values
    #   match declared data_type:
    #     "numeric" → parseable as float
    #     "date" → parseable as a date (common formats: M/D/YYYY, MM/DD/YYYY, YYYY-MM-DD, MM-DD-YYYY)
    #     "text" → always passes
    #   Fail: break_type="type_mismatch", break_description="Field X: expected numeric, got 'abc'"
    # Stop at first failure — do not run subsequent checks
    # feed_status: "failed" if any check failed; "completed" if all ran and passed
    # file_number: f"#{file_index}"
```