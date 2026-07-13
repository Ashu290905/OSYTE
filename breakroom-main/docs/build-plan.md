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

1. **`reconciliation_engine/normaliser.py`** (Days 1–3) — all 15 rules. See rule catalog below.
2. **`reconciliation_engine/filter_engine.py`** (Days 3–4) — all operators per data type, case-insensitive text.

---

## Phase 2 — Engine Completion + Check 6 (Days 5–8)

**Achieves: M2**

### Ashutosh: Duplicate Checker + Tolerance

1. **`reconciliation_engine/duplicate_checker.py`** (Days 5–6)
   - Source: group by composite key → any group ≥2 → ALL records are `duplicate`, none pass
   - Osyte: exact duplicates → keep latest (last in list), drop first. Non-exact same-key records all kept.

2. **`reconciliation_engine/tolerance.py`** (Days 6–7)
   - `absolute`: `abs(source - osyte) <= tolerance_value`
   - `percentage`: `abs(source - osyte) / abs(osyte) * 100 <= tolerance_value` (value already a %, e.g. 0.0001 = 0.0001%)
   - `business_days`: `abs((source_date - osyte_date).days) <= tolerance_value` (calendar days, v1)
   - `directional lte`: `source_date <= osyte_date`

3. Engine tests (Days 7–8) — all 15 normalisation rules, filter per operator/type, both duplicate scenarios, all 4 tolerance types

### Mihir: Pipeline check 6 wiring

1. **`reconciliation_engine/pipeline.py`** — check 6 in exact contract order:
   - filter rules (external feed) → filter rules (internal feed) → dedup (source) → dedup (Osyte) → normalise (source only)
   - Per-source: uses that source's `filter_rules` and `normalization_rules`

2. Tests — check 6 end-to-end, NUM-04 exclusion path, TXT-04 soft fail produces unmatch (not error)

### Aarohi: AI Inference begins

1. **`ai_inference/schema_detector.py`** — infer `label`, `data_type`, `format` from up to 100 sample rows

---

## Phase 3 — Matching + runReconciliation (Days 9–11)

**Achieves: M3**

### Ashutosh: Matcher + Classifier

1. **`reconciliation_engine/matcher.py`** (Days 9–10)
   - `find_match()`: extract composite key from normalised source, scan Osyte records → match or None
   - `compare_fields()`: each mapped non-key field → apply tolerance rule if exists, else exact match → `list[FieldComparison]`

2. **`reconciliation_engine/classifier.py`** (Day 10)
   - Priority order: filtered > duplicate > unmatch > partial_match > auto_match
   - `filtered` if `_exclude=True` (NUM-04) or dropped by filter rule
   - `duplicate` if in duplicate group
   - `unmatch` if no composite key match
   - `partial_match` if any `FieldComparison.match == False`
   - `auto_match` if all `FieldComparison.match == True`

3. Engine integration tests (Days 10–11) — all 5 statuses, tolerance boundaries, verify summary invariant holds

### Mihir: runReconciliation endpoint

1. **`reconciliation_engine/pipeline.py`** — M3 extension
   - Key lookup → field comparison → classify per source record
   - Assign `record_id` (unique within run), attach `source_id`

2. **`reconciliation_api/run_reconciliation.py`** (Days 10–11)
   - Decode all feeds; run basic validation per file (all 6 checks — M2 is complete by Phase 3)
   - Load all feeds into in-memory staging
   - For each source: run full pipeline independently against same Osyte feed
   - Missing source in request → skip silently (no entry in `by_source`)
   - Unknown source_id in request → `source_not_found` 404
   - Assemble: `summary.total` + `summary.by_source`, `basic_validation` per file, `records` with `source_id`
   - Overall status: `completed` if ≥1 source completes; `failed` only if all fail OR internal feed fails

3. Tests — verify summary invariant, single source, multi-source, partial delivery, one source failure isolation

### Aarohi: AI field mapping

1. **`ai_inference/field_mapper.py`** (Days 9–10)

---

## Phase 4 — AI Integration (Days 12–14)

### Aarohi: AI Inference completion + dual-mode createReconConfig

1. **`ai_inference/mapping_inferrer.py`** (Day 12) — TXT-04 mappings from co-occurrence patterns, `org_id` extraction
2. **`ai_inference/rule_selector.py`** (Day 12) — selects from 15-rule catalog only. Never auto-selects: TXT-03, TXT-06, NUM-05, DT-05. Never produces TXT-04 (that's `mapping_inferrer`).
3. **`ai_inference/analyzer.py`** (Day 13) — orchestrates all inference steps, applies system defaults, generates `review_notes`, raises `AIAnalysisFailedError` if zero field mappings inferred
4. **`createReconConfig()` AI mode** (Days 13–14)
   - Guard: both CSV files AND full JSON in same call → `invalid_input_mode` 422
   - AI path: `analyzer.analyze_feeds()` → save → return full config + `review_notes`, `creation_mode: "ai_assisted"`
   - Manual path: `creation_mode: "manual"`, `review_notes: []`

5. Tests (Day 14) — AI mode, invalid_input_mode guard, ai_analysis_failed, review_notes populated, manual mode `review_notes: []`

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