# Breakroom — Build Plan

## Team

- **Aarohi** — config API endpoints, config storage, Pydantic models, AI inference
- **Mihir** — project setup, feed processor, `runReconciliation()` endpoint, multi-source orchestration
- **Ashutosh** — computation engine: normaliser, filter engine, duplicate checker, matcher, tolerance evaluator, classifier

---

## Who builds what, who uses what

| Module | Built by | Used by |
|---|---|---|
| `models.py` | Aarohi | All endpoints (all phases), `config_store/store.py` (Aarohi), `reconciliation_engine/pipeline.py` (Mihir), all engine modules (Ashutosh), `ai_inference/` (Aarohi) |
| `config_store/store.py` | Aarohi | Config API endpoints (Aarohi), `run_reconciliation.py` (Mihir, Phase 3) |
| `config_api/` | Aarohi | API consumers (product UI, LLM agent) |
| `ai_inference/` | Aarohi | `create_recon_config.py` AI mode (Aarohi, Phase 4) |
| `feed_processor/parser.py` | Mihir | `run_reconciliation.py` (Mihir, Phase 3), `ai_inference/analyzer.py` (Aarohi, Phase 4) |
| `feed_processor/merger.py` | Mihir | `run_reconciliation.py` (Mihir, Phase 3), `ai_inference/analyzer.py` (Aarohi, Phase 4) |
| `feed_processor/validator.py` | Mihir | `run_reconciliation.py` (Mihir, Phase 3) |
| `reconciliation_engine/normaliser.py` | Ashutosh | `reconciliation_engine/pipeline.py` (Mihir, Phase 2+) |
| `reconciliation_engine/filter_engine.py` | Ashutosh | `reconciliation_engine/pipeline.py` (Mihir, Phase 2+) |
| `reconciliation_engine/duplicate_checker.py` | Ashutosh | `reconciliation_engine/pipeline.py` (Mihir, Phase 2+) |
| `reconciliation_engine/tolerance.py` | Ashutosh | `reconciliation_engine/matcher.py` (Ashutosh, Phase 3) |
| `reconciliation_engine/matcher.py` | Ashutosh | `reconciliation_engine/pipeline.py` (Mihir, Phase 3) |
| `reconciliation_engine/classifier.py` | Ashutosh | `reconciliation_engine/pipeline.py` (Mihir, Phase 3) |
| `reconciliation_engine/pipeline.py` | Mihir | `run_reconciliation.py` (Mihir) |

---

## Scope: 5 methods

| Method | Route | Owned by | Solves |
|---|---|---|---|
| `createReconConfig()` | `POST /breakroom/createReconConfig` | Aarohi | Problem 1 |
| `listReconConfigs()` | `GET /breakroom/listReconConfigs` | Aarohi | Problem 1 |
| `getReconConfig()` | `GET /breakroom/getReconConfig/{config_id}` | Aarohi | Problem 1 |
| `updateReconConfig()` | `POST /breakroom/updateReconConfig` | Aarohi | Problem 1 |
| `runReconciliation()` | `POST /breakroom/runReconciliation` | Mihir | Problem 2 |

**Out of scope:** Auth, position/cash reconciliation, tenant overlays, write-back to Osyte, per-centre weekend rules (assume Sat/Sun everywhere), reconciliation history storage.

---

## Milestone Mapping

Milestones gate pipeline capabilities in `runReconciliation()`. The config always stores the full ruleset — what changes per milestone is what the engine actually runs.

| Milestone | Gate | Build phase achieved | Day |
|---|---|---|---|
| **M1** | Checks 1–5 pass per file | End of Phase 1 | Day 4 |
| **M2** | Check 6 passes — filter, dedup, normalise run | End of Phase 2 | Day 8 |
| **M3** | Full record classification with field-level evidence | End of Phase 3 | Day 11 |

### M1 — Basic Validation

**Unlocked by:**
- Mihir: `feed_processor/` (parser, merger, validator) — checks 1–5
- Mihir: `run_reconciliation.py` M1 path
- Aarohi: `createReconConfig()` manual mode — config must exist to run against

**Response sections active:** `summary` (external_records total per source, no breakdown), `basic_validation` per file with checks 1–5 results.

**Check names (exact from contract):**
1. Feed Received Check → `feed_not_received`
2. Feed Format Check → `feed_format_error`
3. Feed Failed Check → `feed_file_error`
4. Feed Field Check → `missing_feed_fields`
5. Feed Field Data Type Check → `data_type_mismatch`

### M2 — Data Transformation

**Unlocked by:**
- Ashutosh: `normaliser.py`, `filter_engine.py`, `duplicate_checker.py`
- Mihir: `pipeline.py` check 6 wiring

**Check 6 execution order (contract-exact):**
Filter rules (external feed) → filter rules (internal feed) → duplicate check (source feed) → duplicate check (Osyte feed) → normalization (source feed only)

**Response sections added:** `summary.filtered` per source, check 6 result in `basic_validation`.

**Check name:** Feed Formatting Service Check → `normalization_error`

### M3 — Record Validation + Full Reconciliation

**Unlocked by:**
- Ashutosh: `tolerance.py`, `matcher.py`, `classifier.py`
- Mihir: `pipeline.py` M3 extension, `run_reconciliation.py` M3 path

**Response sections added:** full `records` array with `record_id`, `source_id`, `reconciliation_status`, `external_record`, `matched_internal_record_ref`, `field_comparison`.

**`matched_internal_record_ref` rules:**
- `auto_match`, `partial_match`, `duplicate` → populated with Osyte record ref
- `unmatch`, `filtered` → null

**`field_comparison` rules:**
- `auto_match`, `partial_match`, `duplicate` → populated
- `unmatch`, `filtered` → null

**Summary count invariant per source (must hold):**
`external_records = auto_match + partial_match + unmatch + duplicate + filtered`

---

## System Defaults — `trade_reconciliation`

`composite_key` and `tolerance_rules` are **never inputs** — applied as system defaults, returned in response for review, overridable via `updateReconConfig()`.

| Field | Default |
|---|---|
| `composite_key` | `["fund_id", "account_id", "trade_dt", "trade_transaction_type_cd"]` |

| Field | Tolerance type | Value | Note |
|---|---|---|---|
| `trade_entered_dt` | `business_days` | `1` | ±1 day window |
| `avg_price_per_share` | `percentage` | `0.0001` | Means 0.0001%, NOT 0.01% |
| `net_cash_amt` | `absolute` | `1.00` | $1.00 — NOT $0.01 |
| `final_quantity` | `absolute` | `0` | Exact match required |
| `settlement_dt` | `directional` | `direction: "lte"` | Custodian date ≤ Osyte date |

---

## Error Code Ownership

Every error returns `{"error": "code", "message": "...", "details": {}}`.

| Code | HTTP | Raised by | Module |
|---|---|---|---|
| `invalid_field_mapping` | 422 | Aarohi | `create_recon_config.py`, `update_recon_config.py` |
| `invalid_composite_key` | 422 | Aarohi | `create_recon_config.py`, `update_recon_config.py` |
| `invalid_tolerance_rule` | 422 | Aarohi | `create_recon_config.py`, `update_recon_config.py` |
| `invalid_filter_operator` | 422 | Aarohi | `create_recon_config.py`, `update_recon_config.py` |
| `invalid_input_mode` | 422 | Aarohi | `create_recon_config.py` (AI mode guard) |
| `ai_analysis_failed` | 422 | Aarohi | `ai_inference/analyzer.py` |
| `duplicate_source_id` | 422 | Aarohi | `create_recon_config.py`, `update_recon_config.py` |
| `config_not_found` | 404 | Aarohi | `config_store/store.py` → surfaced by `get_recon_config.py`, `update_recon_config.py`, `run_reconciliation.py` |
| `tenant_not_found` | 404 | Aarohi | `list_recon_configs.py` |
| `source_not_found` | 404 | Mihir | `run_reconciliation.py` |
| `feed_not_received` | 422 | Mihir | `feed_processor/validator.py` check 1 |
| `feed_format_error` | 422 | Mihir | `feed_processor/parser.py` |
| `feed_file_error` | 422 | Mihir | `feed_processor/validator.py` check 3 |
| `missing_feed_fields` | 422 | Mihir | `feed_processor/validator.py` check 4 |
| `data_type_mismatch` | 422 | Mihir | `feed_processor/validator.py` check 5 |
| `normalization_error` | 422 | Ashutosh | `reconciliation_engine/normaliser.py` (hard failure only) |

**Hard vs soft failure for normalisation:** A hard failure (malformed mapping entry, unparseable date) raises `normalization_error` and stops that source's run. A soft failure (TXT-04 `from` value not found in mappings) does NOT raise an error — the original value is preserved and the record will classify as `unmatch` or `partial_match`. If unmatch rate is unexpectedly high, the caller should review TXT-04 mappings via `getReconConfig()`.

---

## Contract-Critical Notes

These are precise behaviours from the contract that are easy to implement incorrectly:

- **`creation_mode`** — returned in `createReconConfig()`, `listReconConfigs()`, and `getReconConfig()` responses. Values: `"ai_assisted"` | `"manual"`.
- **`review_notes`** — present in `createReconConfig()` response for both modes. Empty array `[]` in manual mode. Never returned by `getReconConfig()`.
- **filter operators are case-insensitive for text fields** — `equals`, `not_equals`, `in`, `not_in`, `contains`, `starts_with` all ignore case.
- **percentage tolerance**: `tolerance_value: 0.0001` = 0.0001%, not the decimal fraction 0.0001. `abs(source - osyte) / abs(osyte) * 100 <= tolerance_value`.
- **TXT-04 soft failure**: no match in mappings → return original value, no exception. Record will likely unmatch at matching step.
- **NUM-04**: Price=0 → tag record `_exclude=True`. This becomes `reconciliation_status: "filtered"`.
- **NUM-05**: positive Net Amount on Buy → flip sign. Determines direction from transaction type field after TXT-04 normalisation.
- **DT-04 does not exist in the catalog** — normalization rules go DT-01, DT-02, DT-03, DT-05 (DT-04 was removed).
- **Missing source in `runReconciliation()`** — source in config but not in `external_feeds_data` → skip silently, no error, no `by_source` entry in summary.
- **Source failure isolation** — one source failing basic validation does not stop other sources. Overall `status: "failed"` only if all sources fail OR internal feed fails.
- **`delivery_type`** — always `"upload"` in v1 (system-defaulted, not caller-provided).
- **`file_number`** — `"#1"`, `"#2"` etc., incrementing per file within a source.
- **`updateReconConfig()` immutable fields** — `tenant_id` cannot be changed.

---

## Phase 1 — Foundation (Days 1–4)

**Achieves: M1**

Everyone works independently after agreeing interfaces on Day 1.

### Aarohi: Models + Config API + Config Storage

1. **`models.py`** — lock by end of Day 1
   - Shared dataclasses: `ReconConfig`, `ExternalFeed`, `InternalFeed`, `FieldSchema`, `FieldMapping`, `FilterRule`, `NormalizationRule`, `MappingEntry`, `ToleranceRule`, `ReconciliationRecord`, `FieldComparison`, `ToleranceEvaluation`, `ReconConfigSummary`
   - Pydantic request/response models for all 5 methods
   - Custom exceptions: `ConfigNotFoundError`, `DuplicateConfigError`
   - Error response model

2. **`config_store/store.py`**
   - SQLite schema: `recon_configs` table, full config stored as JSON blob
   - `save_config()`, `get_config()`, `list_configs()`, `update_config()`
   - Merge semantics: `external_feeds` matched by `source_id` — only included sources updated

3. **Config API endpoints**
   - `createReconConfig()` manual mode: validate → save → return full config with `creation_mode: "manual"`, `review_notes: []`
   - `listReconConfigs()`: query by `tenant_id`, return summaries including `creation_mode`
   - `getReconConfig()`: return full config including `creation_mode` (no `review_notes`)
   - `updateReconConfig()`: partial update, source_id merge for `external_feeds`, `composite_key` and `tolerance_rules` can be updated to override system defaults

4. Tests — config CRUD, partial update, source_id merge, system defaults applied, config_not_found, tenant_not_found, duplicate_source_id

### Mihir: Project Setup + Feed Processor

1. **Project skeleton** — repo, `requirements.txt`, FastAPI app shell, two routers, SQLite init on startup, pytest config

2. **`feed_processor/parser.py`** — decode base64, parse CSV, return `list[dict]`, raw strings, raises `FeedFormatError`

3. **`feed_processor/merger.py`** — merge files, tag rows with `_file_index` (1-based)

4. **`feed_processor/validator.py`** — checks 1–5, stop at first failure, return `BasicValidationResult` with `source_id`, `feed_type`, `received_date`, `processed_date`

5. **`run_reconciliation.py` M1 path** — decode feeds, call `parse_csv()` + `run_basic_validation()` per file, assemble `basic_validation` list + `summary` totals, return response. Multi-file merge via `merge_files()`. No pipeline or matching — M1 stops after validation.

6. Tests — parse edge cases, multi-file merge, each check 1–5 pass/fail, M1 end-to-end (valid feeds return completed; invalid feed returns failed with correct check)

### Ashutosh: Start Engine (no Phase 1 blockers)

Normaliser and filter engine operate on plain dicts — no dependency on `models.py`. Agree interfaces Day 1 and start immediately.

1. **`reconciliation_engine/normaliser.py`** (Days 1–3)
   - Writes the code that enforces all 15 normalisation rules in declared order against a source record
   - Each rule is actual transformation logic: TXT-01 does `str.upper()`, TXT-02 does `str.strip()` + whitespace collapse, TXT-03 removes separators unless the value matches a structural identifier pattern, TXT-04 does lookup in a MappingEntry list and substitutes the to_value (returns original on miss — soft failure, no exception), TXT-05 replaces empty/whitespace with `"N/A"`, TXT-06 strips leading zeros
   - NUM rules: NUM-01/NUM-02 handle precision and rounding, NUM-03 takes `abs(value)`, NUM-04 replaces null/empty with 0.0 and flags Price=0 records with `_exclude=True`, NUM-05 flips Net Amount sign for Buy records (scans record for `"BTO"` after TXT-04 has run)
   - DT rules: DT-01 converts to UTC, DT-02 strips time component, DT-03 converts to ISO 8601, DT-05 replaces null/unparseable dates with `date(1900, 1, 1)`
   - Returns a new dict — original not mutated. Hard failure (malformed mapping, unparseable date) raises `NormalizationError`

2. **`reconciliation_engine/filter_engine.py`** (Days 3–4)
   - Writes the code that evaluates each filter rule's condition against every record and includes/excludes accordingly
   - Text operators: case-insensitive string comparison for all 6 operators (`equals`, `not_equals`, `in`, `not_in`, `contains`, `starts_with`) — `str.lower()` comparison throughout
   - Numeric operators: `greater_than`, `less_than`, `between` (inclusive) in addition to equality operators — values parsed as float
   - Date operators: `before`, `after`, `between` — values parsed as date
   - `include` action keeps only matching records; `exclude` drops matching records. Rules applied in sequence. Excluded records tagged with `_filter_rule = rule_name`

---

## Phase 2 — Engine Completion + Check 6 (Days 5–8)

**Achieves: M2**

### Ashutosh: Duplicate Checker + Tolerance

1. **`reconciliation_engine/duplicate_checker.py`** (Days 5–6)
   - Writes two functions: one for the source feed, one for the Osyte feed — different rules for each
   - Source: groups records by composite key (using external field labels — translated by `run_check6()`). Any group with 2+ records: every record in that group gets `_is_duplicate=True`, none pass through. This applies regardless of whether the non-key field values differ between the records
   - Osyte: groups by composite key. Exact duplicates (same key AND all field values identical) → keep the last record in the group, drop the first. Non-exact same-key records (same key, different values) → all kept, handled as an engine implementation detail that doesn't surface in the response

2. **`reconciliation_engine/tolerance.py`** (Days 6–7)
   - Writes the code that evaluates a single field comparison against a tolerance rule and returns a structured result
   - `absolute`: computes `abs(source - osyte)`, checks `<= tolerance_value`, returns actual difference
   - `percentage`: computes `abs(source - osyte) / abs(osyte) * 100`, checks `<= tolerance_value`. Note: `tolerance_value: 0.0001` means 0.0001% — no unit conversion. Edge case: if osyte_value is 0, match only if source is also 0
   - `business_days`: computes `abs((source_date - osyte_date).days)` (calendar days in v1), checks `<= tolerance_value`
   - `directional lte`: checks `source_date <= osyte_date`. Any earlier = pass regardless of how far. Any later = fail. Returns `result: "pass"|"fail"` string instead of numeric difference since there's nothing meaningful to compare

3. Engine tests (Days 7–8) — rule-by-rule normalisation, filter per operator/type, both duplicate scenarios (exact key match, and key + value variants), all 4 tolerance types including directional boundary at exact same day

### Mihir: Pipeline check 6 wiring

1. **`reconciliation_engine/pipeline.py`** — check 6 (Days 5–7)
   - Writes the orchestration code that runs all Data Transformation steps in the exact order required by the contract: filter (external) → filter (internal) → dedup (source) → dedup (Osyte) → normalise (source)
   - Before calling `check_duplicates_source()`, translates the composite key from internal field labels to external field labels using `ExternalFeed.field_mapping` — source records pre-normalisation still have external field names
   - Catches `NormalizationError` from the normaliser and propagates it — `run_reconciliation.py` handles stopping that source's run
   - Returns `Check6Result` separating: normalised records ready for matching, deduplicated Osyte records, filtered records (from filter rules + NUM-04 exclusion), and duplicate source records — these last two are pre-classified and go directly into the response without going through matching

2. Tests — check 6 end-to-end, composite key translation to external labels, NUM-04 exclusion path ends up in filtered (not an error), TXT-04 miss ends up as eventual unmatch (not an error at check 6 level)

### Aarohi: AI Inference begins

1. **`ai_inference/schema_detector.py`** — infer `label`, `data_type`, `format` from up to 100 sample rows

---

## Phase 3 — Matching + runReconciliation (Days 9–11)

**Achieves: M3**

### Ashutosh: Matcher + Classifier

1. **`reconciliation_engine/matcher.py`** (Days 9–10)
   - Writes two functions: `find_match()` and `compare_fields()`
   - `find_match()`: extracts the composite key values from a normalised source record (field values are already in internal format after normalization), scans the Osyte records list to find one with matching values across all composite key fields. Returns the matched Osyte record or `None` (unmatch)
   - `compare_fields()`: for each mapped non-key field, retrieves the source's normalised value and the Osyte value, looks up whether a tolerance rule exists for that field. If yes: calls `evaluate_tolerance()`. If no: exact string/numeric equality check. Returns `list[FieldComparison]` with one entry per field showing both values, match result, and tolerance details

2. **`reconciliation_engine/classifier.py`** (Day 10)
   - Writes the code that assigns a final `reconciliation_status` to each source record based on what happened to it in the pipeline
   - Priority order matters — a record could theoretically satisfy multiple conditions: filtered > duplicate > unmatch > partial_match > auto_match
   - `filtered`: record has `_exclude=True` (NUM-04 Price=0) or was dropped by a filter rule
   - `duplicate`: record was in a composite key group of 2+ (from `check_duplicates_source()`)
   - `unmatch`: `find_match()` returned `None`
   - `partial_match`: `find_match()` returned a match AND at least one `FieldComparison.match == False`
   - `auto_match`: `find_match()` returned a match AND all `FieldComparison.match == True`

3. Engine integration tests (Days 10–11) — all 5 statuses produced correctly, classifier priority order tested, summary invariant holds (`external_records = auto_match + partial_match + unmatch + duplicate + filtered`)

### Mihir: runReconciliation endpoint

1. **`reconciliation_engine/pipeline.py`** — M3 extension (Days 9–10)
   - Extends the pipeline to run matching and classification after check 6 completes
   - For each record in `normalised_source_records` from `Check6Result`: calls `find_match()` against the `osyte_records` lookup table, then `compare_fields()` if a match is found, then `classify_record()` to assign status
   - Assigns a `record_id` (UUID or sequential, unique within this reconciliation run) to every source record across all categories: filtered, duplicate, and those going through matching
   - Attaches `source_id` to every record
   - Returns `PipelineResult` with the complete list of `ReconciliationRecord` objects

2. **`reconciliation_api/run_reconciliation.py`** (Days 10–11)
   - The main endpoint: decodes `internal_feed_data` and all `external_feeds_data[i].files`, runs `parse_csv()` on each file, runs `run_basic_validation()` per file (all 6 checks)
   - After validation: loads all feeds into in-memory staging, runs the full pipeline independently for each source against the same Osyte feed. Sources don't share state — one source's normalised records and Osyte staging are independent of another source's run
   - Missing source (in config but not in request) → skip silently, no `by_source` entry
   - Unknown source_id (in request but not in config) → `source_not_found` 404
   - Assembles the response: `summary.total` (sum across all sources), `summary.by_source` (one entry per source that ran), `basic_validation` list (one entry per file per feed including internal), `records` array (all records across all sources with `source_id` on each)
   - Overall status: `"completed"` if ≥1 source completes; `"failed"` only if all sources fail OR the internal feed itself fails validation

3. Tests — summary invariant per source, single source, multi-source, partial delivery, one source fails (other continues), basic_validation entries correct per file number

### Aarohi: AI field mapping

1. **`ai_inference/field_mapper.py`** (Days 9–10)

---

## Phase 4 — AI Integration (Days 12–14)

### Aarohi: AI Inference completion + dual-mode createReconConfig

1. **`ai_inference/mapping_inferrer.py`** (Day 12)
   - Writes the code that finds co-occurring identifier values across matched record pairs (identified by `field_mapper.py`) and builds TXT-04 inline mapping arrays from them
   - For each mapped identifier field (e.g. Account # ↔ fund_id): collects all (source_value, osyte_value) pairs where the same trade appeared in both files. If `NYK-003640` always co-occurs with `11001`, that becomes `{"from": "NYK-003640", "to": "11001"}`. Scans for `org_id` in the Osyte record alongside the identifier and includes it in the mapping entry if found
   - Returns one `NormalizationRule(rule_id="TXT-04")` per field that needs translation

2. **`ai_inference/rule_selector.py`** (Day 12)
   - Writes the code that scans source records and selects appropriate normalization rules from the existing 15-rule catalog — it does NOT invent new rules
   - Detects: mixed casing → TXT-01; leading/trailing spaces → TXT-02; inconsistent date formats → DT-03; timestamps in date fields → DT-01 + DT-02; negative quantities in any row → NUM-03; inconsistent decimal places → NUM-01 + NUM-02; null/empty in text fields → TXT-05; null/empty in numeric fields → NUM-04
   - Never auto-selects TXT-03, TXT-06, NUM-05, DT-05 — these require business knowledge that can't be inferred from data. Never produces TXT-04 entries (that's `mapping_inferrer`)

3. **`ai_inference/analyzer.py`** (Day 13)
   - Writes the orchestrator that calls all inference modules in sequence and assembles the final `InferenceResult`
   - Order: `detect_schema()` on both feeds → `infer_field_mapping()` → `infer_txt04_mappings()` → `select_normalization_rules()` → apply system defaults for `composite_key` and `tolerance_rules` → generate `review_notes`
   - `review_notes` content: flags field mappings inferred with fewer than N co-occurrence patterns (lower confidence), notes that composite_key and tolerance_rules are system defaults, notes that filter rules could not be inferred (business-specific — add manually via `updateReconConfig()`)
   - Raises `AIAnalysisFailedError` if zero field mappings could be inferred across all sources (files too dissimilar to cross-reference)

4. **`createReconConfig()` AI mode** (Days 13–14)
   - Adds the AI mode path to the existing `createReconConfig()` endpoint
   - Guard at the start: if request contains both `internal_feed_data` (CSV files) AND `internal_feed` (JSON schema) → return `invalid_input_mode` 422 immediately
   - AI mode path: decode base64 files → call `analyzer.analyze_feeds()` → save resulting config via `config_store.save_config()` → return full inferred config with `creation_mode: "ai_assisted"` and populated `review_notes`
   - Manual mode path (existing, unchanged): validate JSON config → save → return full config with `creation_mode: "manual"` and `review_notes: []`
   - Both paths return the same response shape — full config, not a summary

5. Tests (Day 14) — AI mode end-to-end (upload files → get full inferred config back), `invalid_input_mode` guard fires correctly, `ai_analysis_failed` when files are empty or completely dissimilar, `review_notes` populated for AI mode, `review_notes: []` for manual mode, system defaults present in response

---

## Phase 5 — Testing + Hardening (Days 15–18)

All three together.

1. **End-to-end** — `createReconConfig()` AI → `getReconConfig()` → `updateReconConfig()` corrections → `runReconciliation()` complete results
2. **Reconciliation edge cases** — DUP_EXACT, DUP_AMT_VAR, DUP_PRICE_VAR, DUP_QTY_VAR, DUP_KEY_ONLY, 100% unmatch, 100% filtered, all records duplicate, mixed statuses
3. **Normalisation edge cases** — NUM-04 Price=0, NUM-05 sign flip, TXT-04 soft fail → unmatch, DT-05 null date, TXT-03 structural identifier preservation
4. **Tolerance edge cases** — directional lte pass at exact boundary, fail at 1-day late, percentage with small denominator, absolute with $0.00 tolerance, business_days at exact count
5. **Multi-source** — one fails (other continues), partial delivery, between operator, filter rule on non-existent field
6. **AI edge cases** — <10 rows, conflicting co-occurrence patterns, ambiguous types, completely dissimilar files
7. **Summary invariant** — assert `external_records = auto_match + partial_match + unmatch + duplicate + filtered` in every test

---

## Day-by-day

| Day | Aarohi | Mihir | Ashutosh |
|---|---|---|---|
| 1 | `models.py` + `config_store` schema (lock interfaces end of day) | Project skeleton + `parser.py` | Normaliser TXT rules (agree interfaces Day 1) |
| 2 | `config_store/store.py` CRUD | `merger.py` | Normaliser NUM rules |
| 3 | `createReconConfig()` manual + `listReconConfigs()` | `validator.py` checks 1–5 | Normaliser DT rules + `filter_engine.py` |
| 4 | `getReconConfig()` + `updateReconConfig()` + config tests | `run_reconciliation.py` M1 path + feed processor tests | Filter engine tests |
| **M1 — Day 4** | | | |
| 5 | `schema_detector.py` | `pipeline.py` check 6 wiring | `duplicate_checker.py` |
| 6 | `field_mapper.py` start | `pipeline.py` check 6 tests | `tolerance.py` |
| 7 | `field_mapper.py` complete | Multi-source check 6 integration | Engine tests (normaliser + filter + dedup) |
| 8 | AI inference tests (schema + field mapper) | `run_reconciliation.py` check 6 path | Tolerance edge case tests |
| **M2 — Day 8** | | | |
| 9 | `mapping_inferrer.py` start | `pipeline.py` M3: key lookup + compare | `matcher.py` |
| 10 | `mapping_inferrer.py` complete | `pipeline.py` M3: classify + record assembly | `classifier.py` |
| 11 | `rule_selector.py` | `run_reconciliation.py` M3 + multi-source + tests | Engine integration tests (all 5 statuses) |
| **M3 — Day 11** | | | |
| 12 | `analyzer.py` orchestrator | Multi-source edge cases | Tolerance edge cases (directional, percentage boundary) |
| 13 | `createReconConfig()` AI mode wiring | `run_reconciliation.py` response polish + tests | Normaliser edge cases (NUM-04, NUM-05, TXT-04 soft fail) |
| 14 | AI mode tests (all paths, review_notes, system defaults) | Performance: large feeds | Classifier: verify all 5 statuses + priority order |
| 15–18 | End-to-end + all edge cases + performance (all three) | | |

---

## Dependencies

**Day 1 — must agree and lock:**
- `models.py` shared dataclasses (Aarohi leads, all agree)
- `FeedFormatError`, `ConfigNotFoundError`, `DuplicateConfigError` custom exceptions (Aarohi defines)
- `parse_csv()` + `BasicValidationResult` signatures — Aarohi's AI mode and Mihir's pipeline both use these
- `save_config()` / `get_config()` signatures — Mihir's `run_reconciliation.py` calls these

**Day 6:** Mihir's `pipeline.py` check 6 needs Ashutosh's normaliser + filter ready. Ashutosh delivers by end of Day 5.

**Day 9:** Mihir's `pipeline.py` M3 needs Ashutosh's matcher + classifier. Ashutosh: matcher Day 9, classifier Day 10.

**Day 13:** Aarohi's AI mode calls `parser.py` + `merger.py` (Mihir, stable from Phase 1) and `config_store` (Aarohi, stable from Phase 1).

**Days 15–18:** All three test together.

---

## Normalisation Rule Catalog Reference

15 rules total. DT-04 does not exist in this catalog.

| Rule | Category | Build notes |
|---|---|---|
| `TXT-01` | Text | Uppercase all values |
| `TXT-02` | Text | Trim + collapse internal spaces |
| `TXT-03` | Text | Remove separators (`-`, `/`, `.`, `_`) unless structural identifier |
| `TXT-04` | Text | Inline mapping lookup. Soft failure: no match → return original, no exception |
| `TXT-05` | Text | Replace empty/whitespace/placeholder with `N/A` |
| `TXT-06` | Text | Remove leading zeros + trailing fillers |
| `NUM-01` | Numeric | 4-decimal precision |
| `NUM-02` | Numeric | HALF_UP rounding |
| `NUM-03` | Numeric | Absolute value (sign removal) |
| `NUM-04` | Numeric | Null/empty/non-numeric → 0. Price=0 → tag `_exclude=True` |
| `NUM-05` | Numeric | Positive Net Amount on Buy → flip sign. Also handles split integer/decimal |
| `DT-01` | Date | Convert to UTC |
| `DT-02` | Date | Remove time component |
| `DT-03` | Date | ISO 8601 format (YYYY-MM-DD) |
| `DT-05` | Date | Null/invalid → `1900-01-01` |

---

## Codebase Structure

```
breakroom/
  app.py                          # FastAPI startup, SQLite init, router registration
  models.py                       # All Pydantic models + shared dataclasses (Aarohi)

  config_api/                     # Aarohi
    router.py
    create_recon_config.py
    list_recon_configs.py
    get_recon_config.py
    update_recon_config.py

  reconciliation_api/             # Mihir
    router.py
    run_reconciliation.py

  config_store/                   # Aarohi
    store.py
    schema.sql

  feed_processor/                 # Mihir
    parser.py
    merger.py
    validator.py

  reconciliation_engine/
    normaliser.py                 # Ashutosh
    filter_engine.py              # Ashutosh
    duplicate_checker.py          # Ashutosh
    matcher.py                    # Ashutosh
    tolerance.py                  # Ashutosh
    classifier.py                 # Ashutosh
    pipeline.py                   # Mihir

  ai_inference/                   # Aarohi
    analyzer.py
    schema_detector.py
    field_mapper.py
    mapping_inferrer.py
    rule_selector.py

  tests/
    test_normaliser.py
    test_filter_engine.py
    test_duplicate_checker.py
    test_matcher.py
    test_tolerance.py
    test_pipeline.py
    test_config_api.py
    test_run_reconciliation.py
    test_ai_inference.py
```