# Breakroom — Build Plan

## Team

- **Aarohi** — models, config storage, config write endpoints (create/update), tolerance evaluator, field mapper, AI mode wiring
- **Mihir** — project setup, feed processor, runReconciliation, listReconConfigs, mapping inferrer, analyzer
- **Ashutosh** — computation engine (normaliser, filter, dedup, matcher, classifier), pipeline orchestration, getReconConfig, schema detector, rule selector

---

## Who builds what, who uses what

| Module | Built by | Used by |
|---|---|---|
| `models.py` | Aarohi | All endpoints, `config_store/store.py`, `pipeline.py`, all engine modules, all AI modules |
| `config_store/store.py` | Aarohi | Config API endpoints (all three), `run_reconciliation.py` |
| `config_api/create_recon_config.py` | Aarohi | API consumers |
| `config_api/update_recon_config.py` | Aarohi | API consumers |
| `config_api/list_recon_configs.py` | Mihir | API consumers |
| `config_api/get_recon_config.py` | Ashutosh | API consumers |
| `feed_processor/parser.py` | Mihir | `run_reconciliation.py`, `ai_inference/analyzer.py` |
| `feed_processor/merger.py` | Mihir | `run_reconciliation.py`, `ai_inference/analyzer.py` |
| `feed_processor/validator.py` | Mihir | `run_reconciliation.py` |
| `reconciliation_engine/normaliser.py` | Ashutosh | `reconciliation_engine/pipeline.py` |
| `reconciliation_engine/filter_engine.py` | Ashutosh | `reconciliation_engine/pipeline.py` |
| `reconciliation_engine/duplicate_checker.py` | Ashutosh | `reconciliation_engine/pipeline.py` |
| `reconciliation_engine/tolerance.py` | Aarohi | `reconciliation_engine/matcher.py` |
| `reconciliation_engine/matcher.py` | Ashutosh | `reconciliation_engine/pipeline.py` |
| `reconciliation_engine/classifier.py` | Ashutosh | `reconciliation_engine/pipeline.py` |
| `reconciliation_engine/pipeline.py` | Ashutosh | `run_reconciliation.py` |
| `reconciliation_api/run_reconciliation.py` | Mihir | API consumers |
| `ai_inference/schema_detector.py` | Ashutosh | `ai_inference/analyzer.py` |
| `ai_inference/field_mapper.py` | Aarohi | `ai_inference/mapping_inferrer.py`, `ai_inference/analyzer.py` |
| `ai_inference/mapping_inferrer.py` | Mihir | `ai_inference/analyzer.py` |
| `ai_inference/rule_selector.py` | Ashutosh | `ai_inference/analyzer.py` |
| `ai_inference/analyzer.py` | Mihir | `create_recon_config.py` AI mode |

---

## Scope: 5 methods

| Method | Route | Owned by | Solves |
|---|---|---|---|
| `createReconConfig()` | `POST /breakroom/createReconConfig` | Aarohi | Problem 1 |
| `listReconConfigs()` | `GET /breakroom/listReconConfigs` | Mihir | Problem 1 |
| `getReconConfig()` | `GET /breakroom/getReconConfig/{config_id}` | Ashutosh | Problem 1 |
| `updateReconConfig()` | `POST /breakroom/updateReconConfig` | Aarohi | Problem 1 |
| `runReconciliation()` | `POST /breakroom/runReconciliation` | Mihir | Problem 2 |

**Out of scope:** Auth, position/cash reconciliation, tenant overlays, write-back to Osyte, per-centre weekend rules (assume Sat/Sun everywhere), reconciliation history storage.

---

## Milestone Mapping

| Milestone | Gate | Phase | Day |
|---|---|---|---|
| **M1** | Checks 1–5 pass per file | End of Phase 1 | Day 4 |
| **M2** | Check 6 passes — filter, dedup, normalise | End of Phase 2 | Day 8 |
| **M3** | Full record classification with field-level evidence | End of Phase 3 | Day 11 |

### M1 — Basic Validation

**Unlocked by:**
- Mihir: `feed_processor/` (parser, merger, validator) + `run_reconciliation.py` M1 path
- Aarohi: `createReconConfig()` manual mode + `config_store/store.py`

**Response sections active:** `summary` (external_records total per source), `basic_validation` per file with checks 1–5.

**Check names (exact from contract):**
1. Feed Received Check → `feed_not_received`
2. Feed Format Check → `feed_format_error`
3. Feed Failed Check → `feed_file_error`
4. Feed Field Check → `missing_feed_fields`
5. Feed Field Data Type Check → `data_type_mismatch`

### M2 — Data Transformation

**Unlocked by:**
- Ashutosh: `normaliser.py`, `filter_engine.py`, `duplicate_checker.py`, `pipeline.py` check 6
- Mihir: `run_reconciliation.py` check 6 integration

**Check 6 execution order (contract-exact):**
filter (external feed) → filter (internal feed) → dedup (source feed) → dedup (Osyte feed) → normalise (source feed only)

**Response sections added:** `summary.filtered` per source, check 6 in `basic_validation`.

**Check name:** Feed Formatting Service Check → `normalization_error`

### M3 — Record Validation + Full Reconciliation

**Unlocked by:**
- Aarohi: `tolerance.py`
- Ashutosh: `matcher.py`, `classifier.py`, `pipeline.py` M3 extension
- Mihir: `run_reconciliation.py` M3 path

**Response sections added:** full `records` array.

**`matched_internal_record_ref` and `field_comparison`:** populated for `auto_match`, `partial_match`, `duplicate`; null for `unmatch`, `filtered`.

**Summary count invariant per source:** `external_records = auto_match + partial_match + unmatch + duplicate + filtered`

---

## System Defaults — `trade_reconciliation`

`composite_key` and `tolerance_rules` are never inputs. Applied at config creation in both modes, returned in the response, overridable via `updateReconConfig()`.

| Field | Default |
|---|---|
| `composite_key` | `["fund_id", "account_id", "trade_dt", "trade_transaction_type_cd"]` |

| Field | Tolerance type | Value | Note |
|---|---|---|---|
| `trade_entered_dt` | `business_days` | `1` | ±1 day window |
| `avg_price_per_share` | `percentage` | `0.0001` | 0.0001% — NOT 0.01% |
| `net_cash_amt` | `absolute` | `1.00` | $1.00 — NOT $0.01 |
| `final_quantity` | `absolute` | `0` | Exact match |
| `settlement_dt` | `directional` | `direction: "lte"` | Custodian ≤ Osyte |

---

## Error Code Ownership

| Code | HTTP | Raised by | Module |
|---|---|---|---|
| `invalid_field_mapping` | 422 | Aarohi | `create_recon_config.py`, `update_recon_config.py` |
| `invalid_composite_key` | 422 | Aarohi | `create_recon_config.py`, `update_recon_config.py` |
| `invalid_tolerance_rule` | 422 | Aarohi | `create_recon_config.py`, `update_recon_config.py` |
| `invalid_filter_operator` | 422 | Aarohi | `create_recon_config.py`, `update_recon_config.py` |
| `invalid_input_mode` | 422 | Aarohi | `create_recon_config.py` (AI mode guard) |
| `ai_analysis_failed` | 422 | Mihir | `ai_inference/analyzer.py` |
| `duplicate_source_id` | 422 | Aarohi | `create_recon_config.py`, `update_recon_config.py` |
| `config_not_found` | 404 | Aarohi | `config_store/store.py` → surfaced by `get_recon_config.py`, `update_recon_config.py`, `run_reconciliation.py` |
| `tenant_not_found` | 404 | Mihir | `list_recon_configs.py` |
| `source_not_found` | 404 | Mihir | `run_reconciliation.py` |
| `feed_not_received` | 422 | Mihir | `feed_processor/validator.py` check 1 |
| `feed_format_error` | 422 | Mihir | `feed_processor/parser.py` |
| `feed_file_error` | 422 | Mihir | `feed_processor/validator.py` check 3 |
| `missing_feed_fields` | 422 | Mihir | `feed_processor/validator.py` check 4 |
| `data_type_mismatch` | 422 | Mihir | `feed_processor/validator.py` check 5 |
| `normalization_error` | 422 | Ashutosh | `reconciliation_engine/normaliser.py` (hard failure only) |

**Hard vs soft normalisation failure:** Hard failure (malformed mapping entry, unparseable date) raises `normalization_error`, stops that source's run. Soft failure (TXT-04 miss, NUM-04 flag) produces no exception — the record becomes `unmatch` or `filtered`.

---

## Contract-Critical Notes

- **`creation_mode`** — in `createReconConfig()`, `listReconConfigs()`, `getReconConfig()` responses. Values: `"ai_assisted"` | `"manual"`.
- **`review_notes`** — in `createReconConfig()` response only. `[]` in manual mode. Never in `getReconConfig()`.
- **Filter operators case-insensitive for text fields** — `equals`, `not_equals`, `in`, `not_in`, `contains`, `starts_with`.
- **Percentage tolerance**: `0.0001` = 0.0001%, not the fraction.
- **TXT-04 soft failure**: no match → return original value, no exception.
- **NUM-04**: Price=0 → tag `_exclude=True`. Becomes `reconciliation_status: "filtered"`.
- **NUM-05**: positive Net Amount on Buy → flip sign. Scans record for `"BTO"`; requires TXT-04 on the transaction type field to be declared before NUM-05 in the rules list.
- **DT-04 does not exist** — catalog goes DT-01, DT-02, DT-03, DT-05.
- **Missing source** — in config but not in `external_feeds_data` → skip silently, no error, no `by_source` entry.
- **Source failure isolation** — one source failing does not stop others. Overall `status: "failed"` only if all fail OR internal feed fails.
- **`delivery_type`** — always `"upload"` in v1.
- **`file_number`** — `"#1"`, `"#2"` incrementing per file within a source.
- **`updateReconConfig()` immutable fields** — `tenant_id` cannot be changed.

---

## Phase 1 — Foundation (Days 1–4)

**Achieves: M1**

### Aarohi: Models + Config Storage + Write Endpoints

1. **`models.py`** — lock by end of Day 1. All shared dataclasses (`ReconConfig`, `ExternalFeed`, `InternalFeed`, `FieldSchema`, `FieldMapping`, `FilterRule`, `NormalizationRule`, `MappingEntry`, `ToleranceRule`, `ReconciliationRecord`, `FieldComparison`, `ToleranceEvaluation`, `ReconConfigSummary`). Pydantic request/response models for all 5 methods. Custom exceptions: `FeedFormatError`, `ConfigNotFoundError`, `DuplicateConfigError`, `AIAnalysisFailedError`, `NormalizationError`.

2. **`config_store/store.py`** — SQLite schema (`recon_configs` table, config as JSON blob). `save_config()` generates UUID config_id, sets timestamps, raises `DuplicateConfigError` on (config_name, tenant_id) collision. `get_config()` deserialises, raises `ConfigNotFoundError`. `list_configs()` returns lightweight summaries. `update_config()`: scalar fields overwrite; `composite_key`, `tolerance_rules`, `internal_feed` replace entirely; `external_feeds` merged per source_id (sources not included are unchanged, never deleted); `tenant_id` immutable.

3. **`createReconConfig()` manual mode** — validates JSON config (field_mapping references, composite_key fields, tolerance targets, filter operators, unique source_ids), applies system defaults for `composite_key` and `tolerance_rules` when omitted, saves, returns full config with `creation_mode: "manual"`, `review_notes: []`.

4. **`updateReconConfig()`** — partial update with source_id merge for `external_feeds`. Can override system-defaulted `composite_key` and `tolerance_rules`. Returns `fields_updated` list.

5. Tests — config CRUD, partial update all field types, source_id merge, `config_not_found`, `duplicate_source_id`.

### Mihir: Project Setup + Feed Processor + List Endpoint + M1 Endpoint

1. **Project skeleton** — repo, `requirements.txt`, FastAPI app shell, two routers, SQLite init on startup, pytest config.

2. **`feed_processor/parser.py`** — decode base64, parse with `csv.DictReader`, return `list[dict]` (raw strings only), raises `FeedFormatError` on decode failure, empty file, missing headers, or malformed structure.

3. **`feed_processor/merger.py`** — call `parse_csv()` per file, tag rows with `_file_index` (1-based), concatenate into one flat list per source.

4. **`feed_processor/validator.py`** — checks 1–5 in exact contract order, stop at first failure. Returns `BasicValidationResult` carrying `source_id`, `feed_type`, `delivery_type`, `file_number`, `received_date`, `processed_date`, `feed_status`, and per-check results.

5. **`listReconConfigs()`** — calls `config_store.list_configs(tenant_id)`, returns summaries including `creation_mode`; empty → 404 `tenant_not_found`.

6. **`run_reconciliation.py` M1 path** — fetches config, decodes all feeds, parses each file, runs `run_basic_validation()` per file per feed (including internal), assembles `basic_validation` list + `summary` totals, returns response. No pipeline or matching yet. Raises `source_not_found` for unknown source_ids; skips configured sources absent from the request.

7. Tests — parse edge cases, multi-file merge, each check 1–5 pass/fail, M1 end-to-end.

### Ashutosh: Normaliser + Filter Engine + Get Endpoint

1. **`reconciliation_engine/normaliser.py`** — enforcement code for all 15 rules, applied in declared order per record. TXT-01: `str.upper()`. TXT-02: `str.strip()` + collapse internal whitespace. TXT-03: remove separators unless the value matches a structural identifier pattern. TXT-04: case-sensitive lookup in the rule's MappingEntry list → substitute `to_value`, return original on miss (soft failure, no exception). TXT-05: empty/whitespace/placeholder → `"N/A"`. TXT-06: strip leading zeros and trailing fillers. NUM-01/NUM-02: 4-decimal precision + HALF_UP rounding. NUM-03: `abs(value)`. NUM-04: null/empty/non-numeric → 0.0; Price=0 → set `_exclude=True` on the returned dict. NUM-05: after TXT-04 has run, scan the record for `"BTO"`; if present and Net Amount > 0 → multiply by −1. DT-01: convert to UTC. DT-02: strip time component. DT-03: ISO 8601. DT-05: null/unparseable → `date(1900,1,1)`. Returns a new dict — the original is never mutated. Hard failures raise `NormalizationError`.

2. **`reconciliation_engine/filter_engine.py`** — evaluates each rule's condition against every record and includes/excludes accordingly. Applies only rules whose `feed` matches the target feed. All six text operators are case-insensitive. Numeric `between` is inclusive on both ends and raises `ValueError` on inverted bounds. Rules apply in sequence — the kept set from rule N flows into rule N+1. Excluded records are tagged `_filter_rule = rule_name`. Returns `FilterResult`.

3. **`getReconConfig()`** — calls `config_store.get_config(config_id)`, returns full config (same shape as create response, without `review_notes`). Propagates `ConfigNotFoundError` → 404.

4. Tests — rule-by-rule normalisation (all 15), filter per operator per data type, `getReconConfig` found + not-found.

---

## Phase 2 — Engine Completion + Check 6 (Days 5–8)

**Achieves: M2**

### Aarohi: Tolerance Evaluator + Field Mapper + AI Mode Skeleton

1. **`reconciliation_engine/tolerance.py`** — evaluation of a single field comparison against a rule. `absolute`: `abs(source − osyte) ≤ tolerance_value`. `percentage`: `abs(source − osyte) / abs(osyte) × 100 ≤ tolerance_value` (0.0001 = 0.0001%); if osyte is 0, match only if source is 0. `business_days`: `abs(day_diff) ≤ tolerance_value` (calendar days, v1). `directional lte`: `source ≤ osyte` — returns `result: "pass"|"fail"` with `limit`/`actual` as None. Returns `ToleranceEvaluation`. Delivered by end of Phase 2 — consumed by `matcher.py` from Phase 3.

2. **`ai_inference/field_mapper.py`** — cross-file record matching. Finds record pairs that appear to be the same trade: same trade date (tolerating format differences) + numeric values within 5% across at least two numeric fields. From matched pairs, observes which source field consistently aligns with which internal field. Builds `FieldMapping` entries from patterns with sufficient support; flags low-support fields in `confidence_warnings`. Returns `(field_mapping, matched_pairs, confidence_warnings)` — `matched_pairs` is consumed by `mapping_inferrer.py` in Phase 4.

3. **`createReconConfig()` dual-mode skeleton** — request models for AI mode (`internal_feed_data`, `external_feeds_data`), the `invalid_input_mode` guard (CSV files AND JSON config in the same request → 422), and mode routing with the AI path stubbed until `analyzer.py` lands in Phase 4.

4. Tests — all 4 tolerance types including the directional boundary and zero-denominator percentage; input-mode guard; field mapper on clean and ambiguous two-file samples.

### Mihir: Check 6 Endpoint Integration

1. **`run_reconciliation.py` check 6 integration** — calls Ashutosh's `run_check6()` per source, records check 6 result in `basic_validation`, populates `summary.filtered` per source. One source's `NormalizationError` marks that source failed; other sources continue.

2. Tests — check 6 endpoint path, source failure isolation, NormalizationError propagation from one source while others complete.

### Ashutosh: Pipeline Check 6 + Duplicate Checker + Schema Detector

1. **`reconciliation_engine/duplicate_checker.py`** — two functions with different rules. Source: group by (externally-labelled) composite key → any group ≥2: every record tagged `_is_duplicate=True`, none pass through, regardless of non-key value differences. Osyte: exact duplicates (same key + all values identical) → keep last, drop first; non-exact same-key records all kept.

2. **`reconciliation_engine/pipeline.py` — check 6** — orchestration in contract-exact order: filter (external) → filter (internal) → dedup (source) → dedup (Osyte) → normalise (source). Pre-step: translate `composite_key` from internal to external field labels via `ExternalFeed.field_mapping` before calling `check_duplicates_source()` — source records pre-normalisation carry external field names. Catches and propagates `NormalizationError`. Returns `Check6Result`: normalised source records, Osyte lookup records, filtered source records (filter-excluded + NUM-04), duplicate source records. Osyte records excluded by filter are simply absent from the lookup pool and never surface in the response.

3. **`ai_inference/schema_detector.py`** — scans up to 100 sample rows per feed. Infers `data_type` with an 80% threshold: date-like → `"date"`, numeric → `"numeric"`, otherwise → `"text"`. Infers `format`: dominant date pattern (including mixed-format annotation), `DECIMAL(18,N)` from max decimal places or `integer`, `VARCHAR(N)` rounded up from max string length. Returns `list[FieldSchema]` in column order.

4. Tests — check 6 end-to-end at pipeline level, composite key translation, NUM-04 → filtered, TXT-04 miss survives check 6, both duplicate scenarios, schema detection on mixed-format samples.

---

## Phase 3 — Matching + runReconciliation M3 (Days 9–11)

**Achieves: M3**

### Aarohi: Field Mapper Integration + Config Hardening

1. Field mapper integration tests — real sample CSV pairs end-to-end, verify `FieldMapping` and `matched_pairs` output, exercise confidence warnings, low-record inputs.

2. Config API hardening — `updateReconConfig()` partial semantics across multiple sources, overriding system defaults, adding a new source via update, cross-checks against `getReconConfig()` output shape.

3. `InferenceResult` + `review_notes` model finalisation in `models.py` — the shapes `analyzer.py` will emit in Phase 4, agreed with Mihir (analyzer/mapping_inferrer outputs) and Ashutosh (schema/rule outputs) so Phase 4 integration is assembly, not negotiation.

4. Tolerance edge-case tests — directional lte pass at same-day and fail at one day late, percentage with very small denominator, absolute $0.00 exact match.

### Mihir: runReconciliation M3

1. **`run_reconciliation.py` M3** — consumes `PipelineResult` from Ashutosh's pipeline per source; runs the full pipeline independently per source against the same Osyte feed; sources share no state. Missing configured source → skip silently. Unknown source_id → `source_not_found`. Assembles `summary.total` + `summary.by_source`, `basic_validation` per file (all 6 checks), `records` with `source_id`. Overall status: `completed` if ≥1 source completes; `failed` only if all sources fail or the internal feed fails.

2. Tests — summary invariant per source, multi-source, partial delivery, source failure isolation, `by_source` omits skipped sources.

### Ashutosh: Matcher + Classifier + Pipeline M3

1. **`reconciliation_engine/matcher.py`** — `find_match()`: extract composite key values from the normalised source record (already in internal format), scan Osyte records for a full-key match → matched record or None. `compare_fields()`: for each mapped non-key field, apply the tolerance rule if one exists (via Aarohi's `evaluate_tolerance()`), else exact equality → `list[FieldComparison]`.

2. **`reconciliation_engine/classifier.py`** — assigns final status with strict priority: filtered > duplicate > unmatch > partial_match > auto_match. `filtered`: `_exclude=True` or filter-dropped. `duplicate`: `_is_duplicate=True`. `unmatch`: no key match. `partial_match`: any comparison failed. `auto_match`: all passed.

3. **`reconciliation_engine/pipeline.py` M3 extension** — after check 6: `find_match()` per normalised record → `compare_fields()` on match → `classify_record()`. Assigns `record_id` (unique within the run) to every record including the pre-classified filtered and duplicate sets from `Check6Result`; attaches `source_id`. Returns `PipelineResult`.

4. Engine integration tests — all 5 statuses, priority order, summary invariant at pipeline level.

---

## Phase 4 — AI Integration (Days 12–14)

### Aarohi: AI Mode Wiring + Cross-Mode Hardening

1. **`createReconConfig()` AI mode completion** — the Phase 2 skeleton gains its real AI path: decode files → `analyzer.analyze_feeds()` (Mihir) → `save_config()` → return full config with `creation_mode: "ai_assisted"` and populated `review_notes`. Manual path unchanged (`creation_mode: "manual"`, `review_notes: []`). Both paths return the identical full-config response shape.

2. AI mode tests — end-to-end (files in → full inferred config out), `invalid_input_mode` guard, `ai_analysis_failed` propagation, `review_notes` content, system defaults present in response.

3. Cross-mode consistency tests — a config created in AI mode is updatable via `updateReconConfig()` and readable via `getReconConfig()` identically to a manual-mode config.

4. Multi-source edge cases — partial delivery, one source fails while others continue, `between` operator bounds, filter rule targeting a non-existent field.

### Mihir: Mapping Inferrer + Analyzer

1. **`ai_inference/mapping_inferrer.py`** — consumes `matched_pairs` from Aarohi's `field_mapper.py`. For each mapped identifier field, collects (source_value, osyte_value) co-occurrences across pairs; consistent co-occurrence becomes a `MappingEntry`. Extracts `org_id` from the Osyte record when present alongside the identifier. Emits one `NormalizationRule(rule_id="TXT-04")` per field needing value translation.

2. **`ai_inference/analyzer.py`** — orchestrates the full inference sequence: `detect_schema()` (both feeds) → `infer_field_mapping()` → `infer_txt04_mappings()` → `select_normalization_rules()` → apply system defaults for `composite_key` and `tolerance_rules` → generate `review_notes` (low-confidence mappings, system-defaulted fields, filter rules not inferred). Raises `AIAnalysisFailedError` if zero field mappings can be inferred across all sources. Returns `InferenceResult`.

3. Integration tests — mapping_inferrer against field_mapper output, analyzer end-to-end with sample feeds, `ai_analysis_failed` on empty/dissimilar files.

4. Performance — 10k-record feeds through the full M3 endpoint, multi-file merge (5 files per source).

### Ashutosh: Rule Selector + Engine Edge Cases

1. **`ai_inference/rule_selector.py`** — scans source rows and selects from the existing 15-rule catalog only. Mixed casing → TXT-01. Leading/trailing spaces → TXT-02. Mixed date formats → DT-03. Timestamps in date fields → DT-01 + DT-02. Negative quantities → NUM-03. Inconsistent decimal places → NUM-01 + NUM-02. Null/empty text → TXT-05. Null/empty numeric → NUM-04. Never auto-selects TXT-03, TXT-06, NUM-05, DT-05 (business-knowledge rules); never emits TXT-04 (owned by `mapping_inferrer`). Returns `list[NormalizationRule]`.

2. Engine edge cases — NUM-04 Price=0 → filtered; NUM-05 sign flip after TXT-04; TXT-04 soft fail → unmatch; DT-05 null date; all duplicate variants at pipeline level.

---

## Phase 5 — Testing + Hardening (Days 15–18)

All three together.

1. **End-to-end** — `createReconConfig()` AI mode → `getReconConfig()` → `updateReconConfig()` corrections → `runReconciliation()` full results; same flow with manual mode.
2. **Reconciliation edge cases** — DUP_EXACT, DUP_AMT_VAR, DUP_PRICE_VAR, DUP_QTY_VAR, DUP_KEY_ONLY, 100% unmatch, 100% filtered, all duplicate.
3. **Multi-source** — partial delivery, source failure isolation, `by_source` correctness, summary invariant per source.
4. **AI edge cases** — <10 rows, conflicting co-occurrence patterns, ambiguous schema types, dissimilar files.
5. **Invariant assertion in every test** — `external_records = auto_match + partial_match + unmatch + duplicate + filtered`.
6. **Performance** — AI inference on 100-row samples, `listReconConfigs()` with 50+ configs.

---

## Dependencies

**Day 1 lock (all three agree):**
- `models.py` shared dataclasses and exceptions
- `parse_csv()` + `BasicValidationResult` signatures
- `save_config()` / `get_config()` signatures — called by `run_reconciliation.py`, `getReconConfig()`, and both write endpoints

**Phase 2 (Day 5):** Ashutosh's `pipeline.py` check 6 needs his own normaliser + filter engine — delivered end of Phase 1. Mihir's endpoint integration needs `run_check6()` — mid-Phase 2 handoff via the locked `Check6Result` shape.

**Phase 3 (Day 9):** Ashutosh's `matcher.py` needs Aarohi's `tolerance.py` — delivered end of Phase 2. Mihir's M3 endpoint needs `PipelineResult` — delivered by Ashutosh mid-Phase 3.

**Phase 4 (Days 12–14):** Mihir's `analyzer.py` depends on `schema_detector.py` (Ashutosh, Phase 2 complete), `field_mapper.py` (Aarohi, Phase 2–3 complete), `mapping_inferrer.py` (Mihir) and `rule_selector.py` (Ashutosh) — both built early in this phase; the `InferenceResult` shape was locked in Phase 3 so integration is assembly. Aarohi's AI mode wiring depends on `analyzer.py` — coordinate handoff with Mihir.

**Phase 5 (Days 15–18):** all modules stable; all three test together.

---

## Normalisation Rule Catalog Reference

15 rules. DT-04 does not exist.

| Rule | Category | Build notes |
|---|---|---|
| `TXT-01` | Text | Uppercase |
| `TXT-02` | Text | Trim + collapse spaces |
| `TXT-03` | Text | Remove separators unless structural identifier |
| `TXT-04` | Text | Inline mapping lookup. Soft failure: no match → return original |
| `TXT-05` | Text | Empty/whitespace/placeholder → `N/A` |
| `TXT-06` | Text | Remove leading zeros + trailing fillers |
| `NUM-01` | Numeric | 4-decimal precision |
| `NUM-02` | Numeric | HALF_UP rounding |
| `NUM-03` | Numeric | Absolute value |
| `NUM-04` | Numeric | Null/empty/non-numeric → 0. Price=0 → `_exclude=True` |
| `NUM-05` | Numeric | Positive Net Amount on Buy → flip sign. Scans record for `"BTO"` |
| `DT-01` | Date | Convert to UTC |
| `DT-02` | Date | Remove time component |
| `DT-03` | Date | ISO 8601 |
| `DT-05` | Date | Null/invalid → `1900-01-01` |

---

## Codebase Structure

```
breakroom/
  app.py                          # FastAPI startup, SQLite init, router registration
  models.py                       # Aarohi

  config_api/
    router.py
    create_recon_config.py        # Aarohi
    list_recon_configs.py         # Mihir
    get_recon_config.py           # Ashutosh
    update_recon_config.py        # Aarohi

  reconciliation_api/
    router.py
    run_reconciliation.py         # Mihir

  config_store/
    store.py                      # Aarohi
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
    tolerance.py                  # Aarohi
    classifier.py                 # Ashutosh
    pipeline.py                   # Ashutosh

  ai_inference/
    analyzer.py                   # Mihir
    schema_detector.py            # Ashutosh
    field_mapper.py               # Aarohi
    mapping_inferrer.py           # Mihir
    rule_selector.py              # Ashutosh

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