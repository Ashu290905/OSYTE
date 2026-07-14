# Breakroom Phase 2 — Interfaces & Signatures

## Who builds what, who uses what

| Module | Built by | Used by |
|---|---|---|
| `reconciliation_engine/normaliser.py` | Ashutosh | `reconciliation_engine/pipeline.py` (Ashutosh, Phase 2+) |
| `reconciliation_engine/filter_engine.py` | Ashutosh | `reconciliation_engine/pipeline.py` (Ashutosh, Phase 2+) |
| `reconciliation_engine/duplicate_checker.py` | Ashutosh | `reconciliation_engine/pipeline.py` (Ashutosh, Phase 2+) |
| `reconciliation_engine/tolerance.py` | Aarohi | `reconciliation_engine/matcher.py` (Ashutosh, Phase 3) |
| `reconciliation_engine/pipeline.py` (check 6) | Ashutosh | `reconciliation_api/run_reconciliation.py` (Mihir, Phase 3) |
| `ai_inference/schema_detector.py` | Ashutosh | `ai_inference/analyzer.py` (Mihir, Phase 4) |
| `ai_inference/field_mapper.py` | Aarohi | `ai_inference/mapping_inferrer.py` (Mihir, Phase 4), `ai_inference/analyzer.py` (Mihir, Phase 4) |

**Agree by start of Day 5:**
- `FilterResult` and `DuplicateCheckResult` dataclasses — `pipeline.py` (Ashutosh) builds with these; agree before integration
- `Check6Result` dataclass — Mihir's `run_reconciliation.py` unpacks this; agree before endpoint integration
- `evaluate_tolerance()` signature — Aarohi builds it, Ashutosh's `matcher.py` consumes it from Day 9
- `normalise_record()` + `apply_filter_rules()` signatures — called by `pipeline.py` (Ashutosh)

---

## Interfaces

```python
# Ashutosh: reconciliation_engine/normaliser.py
normalise_record(record: dict, rules: list[NormalizationRule]) -> dict
# Apply rules in declared order. Returns normalised copy — original not mutated.
# TXT-04: from_value not found → return original value, no exception (soft failure).
# NUM-04: Price field == 0.0 → set _exclude=True on returned dict.
# NUM-05: positive Net Amount on Buy transaction → flip sign to negative.
# Hard failure (malformed mapping, unparseable date) → raise NormalizationError.

apply_txt04(value: str, mappings: list[MappingEntry]) -> str
# Case-sensitive lookup of value against mappings[i].from_value.
# Returns matching to_value, or original value if no match (soft failure).
# CASING CONTRACT: if a TXT-01 (uppercase) rule targets the same field, it runs first
# and the record value arrives here already uppercased. The mapping from_value is
# NOT auto-uppercased — so config must store from_value in the post-TXT-01 form.
# In practice: when TXT-01 precedes TXT-04 on a field, author from_value in UPPERCASE
# ("BUY" not "Buy"). AI mode enforces this automatically (see mapping_inferrer, Phase 4);
# manual mode is validated at config time (see casing-consistency check below).

# Ashutosh: reconciliation_engine/filter_engine.py
apply_filter_rules(records: list[dict], rules: list[FilterRule], feed: str) -> FilterResult
# Apply only rules where rule.feed == feed ("internal" or "external").
# Text operators: all case-insensitive (equals, not_equals, in, not_in, contains, starts_with).
# Multiple rules applied in sequence — kept from rule N pass into rule N+1.
# Excluded records tagged with _filter_rule = rule_name.

# Ashutosh: reconciliation_engine/duplicate_checker.py
check_duplicates_source(records: list[dict], composite_key: list[str]) -> DuplicateCheckResult
# Group by composite key. Any group ≥2 → ALL records are duplicate. None pass through.
# Duplicate records tagged with _is_duplicate=True.

check_duplicates_osyte(records: list[dict], composite_key: list[str]) -> list[dict]
# Exact duplicates (same composite key + all field values identical) → keep latest, drop first.
# Non-exact same-key records → all kept (engine implementation detail).
# Returns deduplicated list ready for key lookup.

# Aarohi: reconciliation_engine/tolerance.py
evaluate_tolerance(source_value: Any, osyte_value: Any, rule: ToleranceRule) -> ToleranceEvaluation
# absolute: abs(source - osyte) <= tolerance_value
# percentage: abs(source - osyte) / abs(osyte) * 100 <= tolerance_value
#             (0.0001 means 0.0001%, NOT the fraction 0.0001)
# business_days: abs((source_date - osyte_date).days) <= tolerance_value (calendar days, v1)
# directional lte: source_date <= osyte_date — any earlier is pass; any later is fail

# Ashutosh: reconciliation_engine/pipeline.py (check 6)
run_check6(
    source_rows: list[dict],
    osyte_rows: list[dict],
    external_feed: ExternalFeed,
    internal_filter_rules: list[FilterRule],
    composite_key: list[str],
) -> Check6Result
# Exact execution order from contract:
#   1. filter_engine(source_rows, external_feed.filter_rules, feed="external")
#   2. filter_engine(osyte_rows, internal_filter_rules, feed="internal")
#   3. duplicate_checker.check_duplicates_source(kept_source, composite_key)
#   4. duplicate_checker.check_duplicates_osyte(kept_osyte, composite_key)
#   5. normalise each unique source record via normaliser.normalise_record()
# Raises NormalizationError on hard failure in step 5 → stops this source's run.

# Ashutosh: ai_inference/schema_detector.py
detect_schema(rows: list[dict]) -> list[FieldSchema]
# Infer label, data_type, format from column headers and up to 100 sample rows.

# Aarohi: ai_inference/field_mapper.py
infer_field_mapping(
    internal_rows: list[dict],
    source_rows: list[dict],
    internal_schema: list[FieldSchema],
    source_schema: list[FieldSchema],
) -> tuple[list[FieldMapping], list[tuple[dict, dict]], list[str]]
# Finds record pairs that appear to be the same trade across both files
# Observes which source field aligns with which internal field
# Returns: (field_mapping, matched_pairs, confidence_warnings)
# matched_pairs feeds into mapping_inferrer.py in Phase 4
```

---

## New Dataclasses — add to `models.py`

These must be added to `models.py` at the start of Phase 2 (Day 5). All modules in Phase 2 depend on them.

```python
@dataclass
class FilterResult:
    kept: list[dict]
    excluded: list[dict]                # each row has _filter_rule: str tag added

@dataclass
class DuplicateCheckResult:             # source feed only
    unique_records: list[dict]
    duplicate_records: list[dict]       # each row has _is_duplicate: True tag added

@dataclass
class Check6Result:
    normalised_source_records: list[dict]   # passed filter + dedup + normalised → go to matching
    osyte_records: list[dict]               # passed filter + dedup → lookup table for key matching
    filtered_source_records: list[dict]     # excluded by filter rules or NUM-04 → status: "filtered"
    duplicate_source_records: list[dict]    # failed source duplicate check → status: "duplicate"
```

## New Exception — add to `models.py`

```python
class NormalizationError(Exception):
    """Raised by reconciliation_engine/normaliser.py on hard failure only.
    Hard failure: malformed mapping entry, date value that cannot be parsed at all.
    Soft failure (TXT-04 miss, NUM-04 flag) does NOT raise this exception.
    Maps to HTTP 422 normalization_error at the endpoint level.
    Stops the run for that source; other sources continue independently."""
    pass
```

---

## Signatures

### Ashutosh: `reconciliation_engine/normaliser.py`

```python
def normalise_record(
    record: dict,                       # raw or partially-normalised row dict
    rules: list[NormalizationRule],     # from ExternalFeed.normalization_rules
) -> dict:
    # Apply each rule in declared list order
    # Each rule applied only to its target_fields
    # Returns a NEW dict — original is not mutated
    #
    # Rule-specific behaviour:
    # TXT-01: str.upper() on target field values
    # TXT-02: str.strip() + collapse internal whitespace sequences to single space
    # TXT-03: remove "-", "/", ".", "_" unless value matches a structural identifier pattern
    #         (e.g. preserve "NYK-003640", remove separators from "12-34-56")
    # TXT-04: call apply_txt04() — soft failure if no match, see below
    # TXT-05: if value is empty, whitespace-only, or a known placeholder → replace with "N/A"
    # TXT-06: lstrip("0") and rstrip known filler chars on identifier fields
    # NUM-01: round to 4 decimal places
    # NUM-02: apply HALF_UP rounding to 4 decimal places
    # NUM-03: abs(value) — removes sign regardless of transaction type
    # NUM-04: if value is null/empty/non-numeric → replace with 0.0
    #         if value == 0.0 for a Price field → set returned_dict["_exclude"] = True
    # NUM-05: Net Amount sign correction — cross-field dependency
    #         After TXT-04 runs, check the value of the Transaction Type field in the record
    #         If the transaction type value is "BTO" (normalised Buy) AND Net Amount > 0:
    #             → multiply Net Amount by -1 (Buy = cash out = negative)
    #         Implementation note: scan ALL fields in the current record state for value "BTO"
    #         rather than hardcoding the field name — the Transaction Type field name varies by
    #         custodian ("Transaction Type" for CNB, "Tran Type" for BOA etc.).
    #         This means TXT-04 on the transaction type field MUST be declared before NUM-05
    #         in the normalization_rules list for this to work correctly.
    # DT-01: convert datetime with timezone info to UTC
    # DT-02: strip time component, keep date only
    # DT-03: normalise to ISO 8601 string "YYYY-MM-DD"
    # DT-05: if value is null or cannot be parsed as any date → replace with date(1900, 1, 1)
    #
    # Hard failure → raise NormalizationError (malformed MappingEntry, date entirely unparseable)
    # Soft failure (TXT-04 no match, NUM-04 zero flag) → no exception, handled silently

def apply_txt04(
    value: str,
    mappings: list[MappingEntry],
) -> str:
    # Case-sensitive exact match of value against MappingEntry.from_value
    # Returns matching MappingEntry.to_value on match
    # Returns original value unchanged if no match found (soft failure, no exception)
    #
    # CASING CONTRACT (resolves the TXT-01/TXT-04 ordering trap):
    #   apply_txt04 does NOT uppercase anything itself. If a TXT-01 rule targets the
    #   same field, normalise_record applies it before TXT-04, so `value` arrives
    #   already uppercased. The mapping's from_value must therefore already be in the
    #   post-TXT-01 form or the lookup silently misses.
    #   Enforcement:
    #     - AI mode: mapping_inferrer emits from_value in the normalised (post-TXT-01)
    #       form, so keys always align.
    #     - Manual mode: createReconConfig validates that, for any field with both a
    #       TXT-01 and a TXT-04 rule, every from_value equals from_value.upper();
    #       otherwise raises invalid_field_mapping at config time (never a silent
    #       runtime unmatch).
```

---

### Ashutosh: `reconciliation_engine/filter_engine.py`

```python
def apply_filter_rules(
    records: list[dict],
    rules: list[FilterRule],
    feed: str,                          # "internal" | "external" — apply only matching rules
) -> FilterResult:
    # Filter only rules where rule.feed == feed; skip others silently
    # Apply rules in declared list order
    # After each rule, the kept set flows into the next rule
    # Text operators are ALL case-insensitive:
    #   equals / not_equals: str.lower() comparison
    #   in / not_in: check against [v.lower() for v in values]
    #   contains: value.lower() in target.lower()
    #   starts_with: target.lower().startswith(value.lower())
    # Numeric operators:
    #   between: values[0] <= field_value <= values[1] (inclusive both ends)
    #   between requires values[0] <= values[1]; raise ValueError if inverted
    # Date operators:
    #   before: field_date < values[0]
    #   after: field_date > values[0]
    #   between: values[0] <= field_date <= values[1] (inclusive both ends)
    # "include" action: keep only records that match the rule condition
    # "exclude" action: drop records that match the rule condition
    # Excluded records: add _filter_rule = rule.rule_name to each excluded row
    # Returns FilterResult with final kept list and all excluded rows across all rules
```

---

### Ashutosh: `reconciliation_engine/duplicate_checker.py`

```python
def check_duplicates_source(
    records: list[dict],                # filtered source records — pre-normalisation
                                        # still have external field names (e.g. "Account #" not "fund_id")
                                        # composite_key passed here must use EXTERNAL field labels
                                        # (translated by the caller in run_check6 via field_mapping)
    composite_key: list[str],           # EXTERNAL field labels for the composite key
                                        # run_check6() translates from internal labels using field_mapping
) -> DuplicateCheckResult:
    # Build the composite key for each record by reading the fields named in
    #   composite_key (EXTERNAL labels) from the record's raw pre-normalisation values.
    #   Dedup runs BEFORE normalisation in check 6, so these are as-received strings —
    #   compare them as-is (no casing/precision normalisation applied here).
    # Group records by their composite key tuple
    # Any group with count >= 2: ALL records in that group are duplicate
    #   — regardless of whether non-key values differ
    # Add _is_duplicate=True tag to all duplicate records
    # Return:
    #   unique_records: all groups of exactly 1 → pass to matching
    #   duplicate_records: all records from groups of 2+ → status: "duplicate"

def check_duplicates_osyte(
    records: list[dict],
    composite_key: list[str],
) -> list[dict]:
    # Group Osyte records by composite key
    # For groups of 1: record passes through unchanged
    # For exact duplicates (same composite key AND all non-key field values identical):
    #   keep the LAST record in the group (latest), drop all earlier ones
    # For non-exact same-key groups (same composite key, different field values):
    #   keep all records — engine implementation detail, does not surface in response
    # Returns flat list of deduplicated Osyte records ready for key lookup
```

---

### Aarohi: `reconciliation_engine/tolerance.py`

```python
def evaluate_tolerance(
    source_value: Any,                  # normalised source value
    osyte_value: Any,                   # Osyte value
    rule: ToleranceRule,
) -> ToleranceEvaluation:
    # absolute:
    #   diff = abs(float(source_value) - float(osyte_value))
    #   match = diff <= rule.tolerance_value
    #   return ToleranceEvaluation(type="absolute", match=match, limit=rule.tolerance_value, actual=diff)
    #
    # percentage:
    #   diff_pct = abs(float(source_value) - float(osyte_value)) / abs(float(osyte_value)) * 100
    #   match = diff_pct <= rule.tolerance_value
    #   Note: tolerance_value=0.0001 means 0.0001%, NOT the fraction; no conversion needed
    #   return ToleranceEvaluation(type="percentage", match=match, limit=rule.tolerance_value, actual=diff_pct)
    #   Edge case: if osyte_value == 0, match = (source_value == 0)
    #
    # business_days (calendar days in v1):
    #   diff_days = abs((source_value - osyte_value).days)
    #   match = diff_days <= int(rule.tolerance_value)
    #   return ToleranceEvaluation(type="business_days", match=match, limit=rule.tolerance_value, actual=float(diff_days))
    #
    # directional lte:
    #   match = source_value <= osyte_value
    #   return ToleranceEvaluation(type="directional", match=match, limit=None, actual=None,
    #                              direction="lte", result="pass" if match else "fail")
    #
    # Raises ValueError if tolerance_type is unknown
    # Raises TypeError if value types are incompatible with the operation
```

---

### Ashutosh: `reconciliation_engine/pipeline.py` — check 6

```python
def run_check6(
    source_rows: list[dict],            # merged rows from merger.merge_files() for this source
    osyte_rows: list[dict],             # rows from parser.parse_csv() for the internal feed
    external_feed: ExternalFeed,        # source config — filter_rules, normalization_rules, composite_key
    internal_filter_rules: list[FilterRule],  # from ReconConfig — rules targeting feed="internal"
    composite_key: list[str],           # from ReconConfig.composite_key
) -> Check6Result:
    # Exact execution order from contract:
    # Step 0 (pre-step): translate composite_key from internal to external field labels
    #   source_composite_key = []
    #   for each internal_field in composite_key:
    #     find the FieldMapping where mapping.internal_field == internal_field
    #     append mapping.external_field to source_composite_key
    #   This is needed because source records pre-normalisation have external field names.
    # Step 1: filter source rows by external filter rules
    #   source_filter = apply_filter_rules(source_rows, external_feed.filter_rules, feed="external")
    # Step 2: filter osyte rows by internal filter rules
    #   osyte_filter = apply_filter_rules(osyte_rows, internal_filter_rules, feed="internal")
    # Step 3: duplicate check on source (kept from step 1) using translated external composite key
    #   source_dedup = check_duplicates_source(source_filter.kept, source_composite_key)
    # Step 4: duplicate check on Osyte (kept from step 2) using internal composite key directly
    #   osyte_dedup = check_duplicates_osyte(osyte_filter.kept, composite_key)
    #   (No label translation needed here: Osyte rows are already in internal field labels,
    #    which is exactly what composite_key uses. Only the SOURCE side needs the Step 0
    #    internal→external translation.)
    # Step 5: normalise each record in source_dedup.unique_records
    #   for each record: normalised = normalise_record(record, external_feed.normalization_rules)
    #   if normalised.get("_exclude"): move to filtered_source (NUM-04 exclusion)
    #   else: add to normalised_source_records
    #
    # Raises NormalizationError on hard failure in step 5 — caller (run_reconciliation.py) catches
    # and stops this source's pipeline, marks check 6 as failed for this source.
    #
    # Returns Check6Result:
    #   normalised_source_records: records that passed all steps + normalised → proceed to matching
    #   osyte_records: osyte_dedup result → lookup table for matching
    #                  Note: Osyte records excluded by filter rules are simply absent from this list.
    #                  They do not surface in the response — filtered Osyte records only reduce the
    #                  available lookup pool. No separate tracking of filtered Osyte records needed.
    #   filtered_source_records: source_filter.excluded + NUM-04 excluded → status: "filtered"
    #   duplicate_source_records: source_dedup.duplicate_records → status: "duplicate"
```

---

### Ashutosh: `ai_inference/schema_detector.py`

```python
def detect_schema(
    rows: list[dict],                   # parsed rows from parse_csv() — use up to first 100 rows
) -> list[FieldSchema]:
    # For each column header found in rows[0].keys():
    #   Collect sample values from all rows (excluding empty/null values)
    #   Infer data_type:
    #     "date" — if ≥80% of samples parse as any common date format
    #     "numeric" — if ≥80% of samples parse as float and none parse as dates
    #     "text" — otherwise
    #   Infer format:
    #     date → detect dominant format among: "MM/DD/YYYY", "MM-DD-YYYY", "YYYY-MM-DD",
    #             "DATETIME (M/D/YYYY H:MM)", "MM/DD/YYYY or MM-DD-YYYY" (if mixed)
    #     numeric → "DECIMAL(18,N)" where N = max decimal places seen; "integer" if all integers
    #     text → "VARCHAR(N)" where N = max string length seen (round up to 10, 20, 30, 50, 100, 200)
    #   mandatory = True for all detected fields (caller adjusts if needed)
    # Returns list[FieldSchema] with one entry per column, in column order
```

---

### Aarohi: `ai_inference/field_mapper.py`

```python
def infer_field_mapping(
    internal_rows: list[dict],          # sample Osyte records (up to 100)
    source_rows: list[dict],            # sample source records (up to 100)
    internal_schema: list[FieldSchema], # from schema_detector on internal feed
    source_schema: list[FieldSchema],   # from schema_detector on source feed
) -> tuple[list[FieldMapping], list[tuple[dict, dict]], list[str]]:
    # 1. Find record pairs across both files that appear to be the same trade:
    #    — Same trade date (allowing DT-03 normalisation for format differences)
    #    — Numeric values within 5% tolerance across at least 2 numeric fields
    # 2. For each candidate pair, observe which source field consistently aligns
    #    with which internal field (values are similar or in a consistent transformation)
    # 3. Build FieldMapping list from patterns with sufficient support (≥N pairs)
    # 4. Populate confidence_warnings for fields with fewer than N/2 pairs
    # 5. Return matched_pairs as a byproduct — consumed by mapping_inferrer.py in Phase 4
    #    to derive TXT-04 inline value mappings. mapping_inferrer is responsible for
    #    emitting from_value in the post-TXT-01 (uppercase, if TXT-01 targets the field)
    #    form so the runtime casing contract in apply_txt04 holds.
    # Returns: (field_mapping, matched_pairs, confidence_warnings)
```