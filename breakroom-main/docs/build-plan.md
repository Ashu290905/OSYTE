# Breakroom — Build Plan (deterministic-first)

> **Revision note.** This plan builds the recon platform as a **fully deterministic system first** — manual configuration plus the complete reconciliation pipeline through record classification — and hardens it to production before any AI/config-inference work begins. The AI capability is a deferred layer, described in the final section of this document (no separate file), added **only once the deterministic recon is live**. Interface signatures: [phase1-interface.md](phase1-interface.md) (M1), [phase2-interface.md](phase2-interface.md) (M2), [phase3-interface.md](phase3-interface.md) (M3).
>
> Multi-layout / multi-file support (one custodian delivering several files with different column schemas) is **out of scope for this plan** — it is owned separately by a teammate and will be documented on its own when picked up.

## Guiding principle — why the split is clean

AI in Breakroom only changes **how a configuration is created** — inferred from sample CSVs versus typed by hand. It never sits in the reconciliation hot path: `runReconciliation()` consumes a stored `ReconConfig` and does not care whether that config was authored manually or by inference. That single seam is what makes deterministic-first safe:

- The deterministic core is a complete product on its own: a user (or a programmatic caller / LLM-generated JSON) supplies a full config via `createReconConfig()` manual mode, and every reconciliation runs end-to-end.
- The AI layer plugs in at exactly **one** point — `createReconConfig()` AI mode calling `analyzer.analyze_feeds()` — and produces a `ReconConfig` in the identical shape manual mode produces. Nothing downstream changes.

Removing AI from the early phases is therefore a re-sequencing, not a redesign. The M1/M2/M3 gates were already deterministic; the AI tasks were interleaved alongside them and are now consolidated into the deferred layer at the end.

---

## Team — core (deterministic) responsibilities

- **Aarohi** — models, config storage, config write endpoints (create/update, **manual mode**), tolerance evaluator, classifier
- **Mihir** — project setup, feed processor, `runReconciliation`, `listReconConfigs`
- **Ashutosh** — computation engine (normaliser, filter, dedup, matcher), pipeline orchestration, `getReconConfig`

AI-layer responsibilities are assigned in the deferred-layer section at the end of this document.

---

## Who builds what, who uses what (deterministic core)

| Module | Built by | Used by |
|---|---|---|
| `models.py` | Aarohi | All endpoints, `config_store/store.py`, `pipeline.py`, all engine modules |
| `config_store/store.py` | Aarohi | Config API endpoints (all three), `run_reconciliation.py` |
| `config_api/create_recon_config.py` (manual mode) | Aarohi | API consumers |
| `config_api/update_recon_config.py` | Aarohi | API consumers |
| `config_api/list_recon_configs.py` | Mihir | API consumers |
| `config_api/get_recon_config.py` | Ashutosh | API consumers |
| `feed_processor/parser.py` | Mihir | `merger.py`, `run_reconciliation.py` |
| `feed_processor/merger.py` | Mihir | `run_reconciliation.py` |
| `feed_processor/validator.py` | Mihir | `run_reconciliation.py` |
| `reconciliation_engine/normaliser.py` | Ashutosh | `reconciliation_engine/pipeline.py` |
| `reconciliation_engine/filter_engine.py` | Ashutosh | `reconciliation_engine/pipeline.py` |
| `reconciliation_engine/duplicate_checker.py` | Ashutosh | `reconciliation_engine/pipeline.py` |
| `reconciliation_engine/tolerance.py` | Aarohi | `reconciliation_engine/matcher.py` |
| `reconciliation_engine/matcher.py` | Ashutosh | `reconciliation_engine/pipeline.py` |
| `reconciliation_engine/classifier.py` | Aarohi | `reconciliation_engine/pipeline.py` |
| `reconciliation_engine/pipeline.py` | Ashutosh | `run_reconciliation.py` |
| `reconciliation_api/run_reconciliation.py` | Mihir | API consumers |

The `ai_inference/` package is **out of the core** — see the deferred-layer section. Any AI code already in the repo (e.g. `field_mapper.py`) is isolated (no core module imports it) and is not part of any core milestone's definition of done.

---

## Scope: 5 methods

| Method | Route | Owned by | Solves | Core behaviour |
|---|---|---|---|---|
| `createReconConfig()` | `POST /breakroom/createReconConfig` | Aarohi | Problem 1 | **Manual mode only**; AI mode → 501 `not_implemented` |
| `listReconConfigs()` | `GET /breakroom/listReconConfigs` | Mihir | Problem 1 | full |
| `getReconConfig()` | `GET /breakroom/getReconConfig/{config_id}` | Ashutosh | Problem 1 | full |
| `updateReconConfig()` | `POST /breakroom/updateReconConfig` | Aarohi | Problem 1 | full |
| `runReconciliation()` | `POST /breakroom/runReconciliation` | Mihir | Problem 2 | full through M3 |

**`createReconConfig()` in the core:** the dual-mode input guard is retained so the endpoint contract is stable — an AI-mode request (CSV files) is recognised and rejected with **501 `not_implemented`** until the AI layer lands. Every core config is `creation_mode: "manual"` with `review_notes: []`. The `review_notes` field stays in the response model (contract stability); it is always empty until the AI layer populates it.

**Out of scope (core):** Auth, position/cash reconciliation, tenant overlays, write-back to Osyte, per-centre weekend rules (assume Sat/Sun everywhere), reconciliation history storage, multi-layout / multi-file source support (owned separately), and AI-assisted configuration (deferred — see the final section).

---

## Milestone Mapping

| Milestone | Gate | Phase |
|---|---|---|
| **M1** | Checks 1–5 pass per file | End of Phase 1 |
| **M2** | Check 6 passes — filter, dedup, normalise | End of Phase 2 |
| **M3** | Full record classification with field-level evidence | End of Phase 3 |
| **M4** | **Production-ready deterministic core** — hardening, edge cases, multi-source, performance, full error-path coverage | End of Phase 4 |

M4 is the ship gate; the AI layer (M5–M7) begins only after M4, once the deterministic recon is live and stable.

### M1 — Basic Validation *(built)*
**Unlocked by:**
- Mihir: `feed_processor/` (parser, merger, validator) + `run_reconciliation.py` M1 path
- Aarohi: `createReconConfig()` manual mode + `config_store/store.py`

**Response sections active:** `summary` (external_records total per source), `basic_validation` per file with checks 1–5.

**Check names (exact from contract):**
1. Feed Received Check — `feed_not_received`
2. Feed Format Check — `feed_format_error`
3. Feed Failed Check — `feed_file_error`
4. Feed Field Check — `missing_feed_fields`
5. Feed Field Data Type Check — `data_type_mismatch`

### M2 — Data Transformation *(built)*
**Unlocked by:**
- Ashutosh: `normaliser.py`, `filter_engine.py`, `duplicate_checker.py`, `pipeline.py` check 6
- Mihir: `run_reconciliation.py` check 6 integration

**Check 6 execution order (contract-exact):**
filter (external feed) → filter (internal feed) → dedup (source feed) → dedup (Osyte feed) → normalise (source feed only)

**Response sections added:** `summary.filtered` per source, check 6 in `basic_validation`.

**Check name:** Feed Formatting Service Check — `normalization_error`

### M3 — Record Validation + Full Reconciliation
**Unlocked by:**
- Aarohi: `tolerance.py`, `classifier.py`
- Ashutosh: `matcher.py`, `pipeline.py` M3 extension
- Mihir: `run_reconciliation.py` M3 path

**Response sections added:** full `records` array.

**`matched_internal_record_ref` and `field_comparison`:** populated for `auto_match`, `partial_match`, `duplicate`; null for `unmatch`, `filtered`.

**Summary count invariant per source:** `external_records = auto_match + partial_match + unmatch + duplicate + filtered`.

### M4 — Production-Ready Deterministic Core
All three, together. The platform must be complete, stable, and production-ready before any AI work begins.
- **End-to-end (manual)** — create → get → update corrections → run.
- **Reconciliation edge cases** — DUP_EXACT, DUP_AMT_VAR, DUP_PRICE_VAR, DUP_QTY_VAR, DUP_KEY_ONLY, 100% unmatch, 100% filtered, all-duplicate.
- **Multi-source** — partial delivery, source failure isolation, `by_source` correctness, summary invariant at every level.
- **Error-path coverage** — every core error code; hard vs soft normalisation; `between` bounds; filter targeting a non-existent field.
- **Performance** — 10k-record feeds through M3; multi-file merge (5 files per source); `listReconConfigs()` with 50+ configs.
- **Exit criteria (ship gate):** all green; no known correctness gaps; endpoint contract frozen. Only then does the AI layer begin.

---

## System Defaults — `trade_reconciliation`

`composite_key` and `tolerance_rules` are never inputs. Applied at config creation, returned in the response, overridable via `updateReconConfig()`.

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

Each source's `field_mapping` must cover every composite-key field (else `invalid_field_mapping`); tolerance targets must be mapped.

---

## Error Code Ownership (core)

| Code | HTTP | Raised by | Module |
|---|---|---|---|
| `invalid_field_mapping` | 422 | Aarohi | `create_recon_config.py`, `update_recon_config.py` |
| `invalid_composite_key` | 422 | Aarohi | `create_recon_config.py`, `update_recon_config.py` |
| `invalid_tolerance_rule` | 422 | Aarohi | `create_recon_config.py`, `update_recon_config.py` |
| `invalid_filter_operator` | 422 | Aarohi | `create_recon_config.py`, `update_recon_config.py` |
| `invalid_input_mode` | 422 | Aarohi | `create_recon_config.py` — AI-vs-manual mode guard (CSV files + JSON config both/neither) |
| `not_implemented` | 501 | Aarohi | `create_recon_config.py` (AI mode before AI layer) |
| `duplicate_source_id` | 422 | Aarohi | `create_recon_config.py`, `update_recon_config.py` — repeated `source_id` within one request |
| `duplicate_config_name` | 422 | Aarohi | `create_recon_config.py`, `update_recon_config.py` — `(config_name, tenant_id)` collision (`DuplicateConfigError`) |
| `config_not_found` | 404 | Aarohi | `config_store/store.py` — surfaced by get/update/run |
| `tenant_not_found` | 404 | Mihir | `list_recon_configs.py` |
| `source_not_found` | 404 | Mihir | `run_reconciliation.py` |
| `feed_not_received` | 422 | Mihir | `feed_processor/validator.py` check 1 |
| `feed_format_error` | 422 | Mihir | `feed_processor/parser.py` |
| `feed_file_error` | 422 | Mihir | `feed_processor/validator.py` check 3 |
| `missing_feed_fields` | 422 | Mihir | `feed_processor/validator.py` check 4 |
| `data_type_mismatch` | 422 | Mihir | `feed_processor/validator.py` check 5 |
| `normalization_error` | 422 | Ashutosh | `reconciliation_engine/normaliser.py` (hard failure only) |

`ai_analysis_failed` (422) is **deferred** to the AI layer (`analyzer.py`) — see the final section.

**Hard vs soft normalisation failure:** Hard (malformed mapping entry, unparseable date) raises `normalization_error`, stops that source's run. Soft (TXT-04 miss, NUM-04 flag) → no exception; the record becomes `unmatch` or `filtered`.

---

## Contract-Critical Notes (core)

- **`creation_mode`** — always `"manual"` in the core (`"ai_assisted"` arrives with the AI layer).
- **`review_notes`** — `createReconConfig()` response only; always `[]` in the core; never in `getReconConfig()`.
- **Filter operators case-insensitive for text fields** — `equals`, `not_equals`, `in`, `not_in`, `contains`, `starts_with`.
- **Percentage tolerance**: `0.0001` = 0.0001%, not the fraction.
- **TXT-04 soft failure**: no match → return original, no exception. Lookup case-insensitive (casefold).
- **TXT-01/TXT-04 casing (manual mode)**: for a field with both rules, every `from_value` must equal `from_value.upper()` (validated at config time → `invalid_field_mapping`).
- **NUM-04**: Price=0 → `_exclude=True` → `filtered`.
- **NUM-05**: positive Net Amount on Buy → flip sign; scans record for `"BTO"`; TXT-04 on the tx-type field must precede NUM-05 in the rules list.
- **DT-04 does not exist** — catalog: DT-01, DT-02, DT-03, DT-05.
- **Missing source** — configured but absent from `external_feeds_data` → skip silently, no `by_source` entry.
- **Source failure isolation** — one source failing does not stop others. Overall `status: "failed"` only if all sources fail OR the internal feed fails.
- **`delivery_type`** — always `"upload"` in v1.
- **`file_number`** — `"#1"`, `"#2"` incrementing per file within a source.
- **`updateReconConfig()` immutable fields** — `tenant_id`.

---

## Phase 1 — Foundation · Achieves M1

### Aarohi: Models + Config Storage + Write Endpoints (manual)
1. **`models.py`** — lock first. All shared dataclasses (`ExternalFeed`) + Pydantic request/response models for all 5 methods. Exceptions: `FeedFormatError`, `ConfigNotFoundError`, `DuplicateConfigError`, `NormalizationError` (`AIAnalysisFailedError` reserved for the AI layer).
2. **`config_store/store.py`** — SQLite (`recon_configs`, JSON blob). `save_config()` (UUID, timestamps, `DuplicateConfigError`), `get_config()` (`ConfigNotFoundError`), `list_configs()`, `update_config()` (scalar overwrite; `composite_key`/`tolerance_rules`/`internal_feed` replace; `external_feeds` merged per `source_id`; `tenant_id` immutable).
3. **`createReconConfig()` manual mode** — validate config incl. per-source composite-key coverage; apply system defaults; save; return full config, `creation_mode: "manual"`, `review_notes: []`.
4. **`updateReconConfig()`** — partial update with `source_id` merge; override defaults; returns `fields_updated`.
5. Tests — config CRUD, partial update, source merge, `config_not_found`, `duplicate_source_id`.

### Mihir: Project Setup + Feed Processor + List + M1 Endpoint
1. Project skeleton — repo, `requirements.txt`, FastAPI shell, two routers, SQLite init, pytest config.
2. **`parser.py`** — base64 → `csv.DictReader` → `list[dict]` raw strings; `FeedFormatError` on decode/empty/missing-headers/malformed.
3. **`merger.py`** — `parse_csv()` per file, tag `_file_index` (1-based), concatenate per source.
4. **`validator.py`** — checks 1–5 in exact order, stop at first failure; per file; returns `BasicValidationResult`.
5. **`listReconConfigs()`** — `list_configs(tenant_id)`; empty → 404 `tenant_not_found`.
6. **`run_reconciliation.py` M1 path** — fetch config; parse + merge per source; `run_basic_validation()` per file per feed; assemble `basic_validation` + `summary`. `source_not_found` for unknown sources; skip absent configured sources.
7. Tests — parse edge cases, multi-file merge, checks 1–5, M1 end-to-end.

### Ashutosh: Normaliser + Filter Engine + Get Endpoint
1. **`normaliser.py`** — all 15 rules in declared order per record; new dict; hard failures → `NormalizationError`.
2. **`filter_engine.py`** — per-rule include/exclude, feed-scoped, text ops case-insensitive, numeric `between` inclusive (`ValueError` on inverted), sequential, excluded rows tagged `_filter_rule`; returns `FilterResult`.
3. **`getReconConfig()`** — `get_config()`, full config (no `review_notes`), `ConfigNotFoundError` → 404.
4. Tests — rule-by-rule normalisation (all 15), filter per operator per type, `getReconConfig` found + not-found.

---

## Phase 2 — Engine Completion + Check 6 · Achieves M2

### Aarohi: Tolerance Evaluator + Config Hardening
1. **`tolerance.py`** — `absolute`, `percentage` (0.0001 = 0.0001%; osyte 0 → match iff source 0), `business_days` (calendar days, v1), `directional lte`. Returns `ToleranceEvaluation`. Consumed by `matcher.py` from Phase 3.
2. **Config hardening** (pulled earlier now AI is deferred) — `updateReconConfig()` partial semantics across sources, overriding defaults, adding a source via update, cross-checks against `getReconConfig()`.
3. Tests — all 4 tolerance types incl. directional boundary + zero-denominator percentage; multi-source update cases.

### Mihir: Check 6 Endpoint Integration
1. **`run_reconciliation.py` check 6 integration** — call `run_check6()` per source, record check 6 in `basic_validation`, populate `summary.filtered` per source. One source's `NormalizationError` marks that source failed; others continue.
2. Tests — check 6 path, source failure isolation, `NormalizationError` propagation.

### Ashutosh: Pipeline Check 6 + Duplicate Checker
1. **`duplicate_checker.py`** — source: group by external-labelled composite key, any group ≥2 → all `_is_duplicate=True`, none pass; Osyte: exact duplicates → keep last; non-exact same-key → all kept.
2. **`pipeline.py` — check 6** — orchestration in contract-exact order; pre-step translates `composite_key` internal→external via the source's `field_mapping` before `check_duplicates_custodian()`; catches/propagates `NormalizationError`; returns `Check6Result`.
3. Tests — check 6 end-to-end, composite-key translation, NUM-04 → filtered, TXT-04 miss survives, both duplicate scenarios.

---

## Phase 3 — Matching + runReconciliation M3 · Achieves M3

Interfaces: [phase3-interface.md](phase3-interface.md).

### Aarohi: Classifier + Tolerance Edge Cases
1. **`classifier.py`** — `classify_record()`, strict priority `filtered > duplicate > unmatch > partial_match > auto_match`. Pure function over the row tags (`_exclude`/`_filter_rule`/`_is_duplicate`), the matched Osyte record, and the field comparisons; returns the `reconciliation_status` string.
2. Tolerance edge-case tests — directional lte pass same-day / fail one day late; percentage tiny denominator; absolute $0.00 exact.
3. Tests — classification priority order (filtered-before-duplicate, partial vs unmatch), all 5 statuses, against `data/recon_samples/` expected outputs.

### Mihir: runReconciliation M3
1. **`run_reconciliation.py` M3** — consume `PipelineResult` per source; run the pipeline independently per source against the same Osyte feed; sources share no state. Missing configured source → skip; unknown `source_id` → `source_not_found`. Aggregate: `summary.by_source`, `summary.total`, `basic_validation` per file, `records`. Overall status: `completed` if ≥1 source completes; `failed` only if all sources fail or the internal feed fails.
2. Tests — invariant per source, multi-source, partial delivery, source failure isolation, `by_source` omits skipped sources.

### Ashutosh: Matcher + Pipeline M3
1. **`matcher.py`** — `find_match()` (composite-key lookup against Osyte), `compare_fields()` (per mapped non-key field, tolerance rule via `evaluate_tolerance()` else exact equality → `list[FieldComparison]`).
2. **`pipeline.py` M3 extension** — after check 6: rename normalised source keys external→internal via `field_mapping` → `find_match()` → `compare_fields()` → `classify_record()`; assign run-unique `record_id`; attach `source_id`. Returns `PipelineResult`.
3. Engine integration tests — all 5 statuses, priority order, invariant at pipeline level.

---

## Phase 4 — Core Hardening & Production-Readiness · Achieves M4

All three together (AI edge cases removed). Freed capacity from deferring AI is reinvested here.
1. End-to-end (manual): create → get → update → run.
2. Reconciliation edge cases (all DUP_* variants, 100% unmatch/filtered, all-duplicate).
3. Multi-source: partial delivery, source failure isolation, `by_source` correctness, invariant at every level.
4. Error-path coverage (every core code; hard vs soft normalisation; `between` bounds; filter on non-existent field).
5. Performance (10k records through M3; 5-file merge; 50+ configs in list).
6. Invariant asserted in every reconciliation test.

**Exit criteria (M4 / ship gate):** all green; no known correctness gaps; endpoint contract frozen.

---

## Dependencies (core)

**Locked first (all three agree):** `models.py` dataclasses (`ExternalFeed`) + exceptions; `parse_csv()` + `BasicValidationResult`; `save_config()` / `get_config()` signatures.

**Phase 2:** Ashutosh's `pipeline.py` check 6 needs his normaliser + filter engine (end of Phase 1). Mihir's endpoint integration needs `run_check6()` — mid-Phase 2 handoff via the locked `Check6Result` shape.

**Phase 3:** Ashutosh's `matcher.py` needs Aarohi's `tolerance.py` (end of Phase 2). Aarohi's `classifier.py` consumes `FieldComparison` (locked in `models.py`) and the row tags — no runtime dependency on the matcher, so it can be built and unit-tested in parallel; Ashutosh's `pipeline.py` wires `matcher` + `classifier` together. Mihir's M3 endpoint needs `PipelineResult` — delivered by Ashutosh mid-Phase 3.

**Phase 4:** all core modules stable; the three harden and test together. No AI dependencies.

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
    create_recon_config.py        # Aarohi (manual; AI mode → 501 until AI layer)
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
    classifier.py                 # Aarohi
    pipeline.py                   # Ashutosh

  ai_inference/                   # DEFERRED — AI layer (see final section)
    (analyzer.py, schema_detector.py, field_mapper.py,
     mapping_inferrer.py, rule_selector.py)

  tests/
    test_models.py  test_config_api.py  test_config_store.py
    test_parser.py  test_merger.py  test_validator.py
    test_normaliser.py  test_filter_engine.py  test_duplicate_checker.py
    test_tolerance.py  test_matcher.py  test_classifier.py  test_pipeline.py
    test_run_reconciliation.py
    # test_ai_inference.py — deferred to the AI layer
```

---

# AI capability — deferred layer (post-M4)

> Built **only after M4** — the deterministic core is complete, stable, and production-ready (the recon is live). Nothing here gates a core milestone. This is the forward roadmap: what the layer is, where it plugs in, milestones, owners, and interfaces.

## What it is (and isn't)

The AI layer solves one problem: **"set up a reconciliation from sample files without hand-authoring the config."** It reads sample CSVs (Osyte + one or more source feeds) and infers the full `ReconConfig` — field schemas, field mappings, and normalization rules including TXT-04 inline mappings — that a user would otherwise type by hand. It does **not** touch reconciliation; its entire output is a `ReconConfig` (identical shape to manual mode) plus `review_notes`.

**On "AI":** the inference primitives are currently **deterministic pattern-matching** (schema detection = measurement over sample rows; field mapping = record-pair value alignment). "AI" here means "auto-inferred configuration." An LLM-assisted field mapper — model proposes the mapping from column names/samples, the deterministic aligner **verifies** it — is a candidate enhancement, not committed scope; if pursued it lives only in the config-setup path (a once-per-onboarding call), never in the per-run hot path, with the deterministic matcher as the guardrail.

## Single integration point
```
createReconConfig() AI mode
  → decode internal_feed_data + external_feeds_data (base64 CSVs)
  → analyzer.analyze_feeds(...)            # orchestrates the inference sequence
  → save_config(ReconConfig)               # existing core store, unchanged
  → return full config, creation_mode="ai_assisted", review_notes=[...]
```
The core already reserves this seam (AI-mode requests return 501 `not_implemented`). Turning the layer on replaces that 501 with the `analyzer` call. Manual mode is untouched; both modes return the identical full-config response.

## Modules & owners
| Module | Built by | Consumes | Produces |
|---|---|---|---|
| `schema_detector.py` | Ashutosh | sample rows | `list[FieldSchema]` per feed |
| `field_mapper.py` | Aarohi | rows + detected schemas | `(field_mapping, matched_pairs, confidence_warnings)` |
| `mapping_inferrer.py` | Mihir | `matched_pairs` | `NormalizationRule(TXT-04)` per field needing translation |
| `rule_selector.py` | Ashutosh | source rows | `list[NormalizationRule]` (catalog rules, never TXT-04) |
| `analyzer.py` | Mihir | all of the above + feed processor | `InferenceResult` → `ReconConfig` + `review_notes` |
| `createReconConfig()` AI wiring | Aarohi | `analyzer` | full config response |

Some of these may already be partially built (e.g. `field_mapper.py`) — audit against the interfaces below before writing new code.

## Milestones
| Milestone | Gate | Contents |
|---|---|---|
| **M5** | Inference primitives pass in isolation | `schema_detector.py` + `field_mapper.py` on known Osyte↔CNB fixtures |
| **M6** | Config assembly produces a valid `ReconConfig` offline | `mapping_inferrer.py` + `rule_selector.py` + `analyzer.py` + `InferenceResult`/`review_notes`; system defaults applied; `ai_analysis_failed` on unusable inputs |
| **M7** | AI mode live end-to-end | `createReconConfig()` AI wiring; files-in → full inferred config out; `review_notes` populated; cross-mode consistency; AI edge cases |

Sequencing after M4: M5 (primitives, parallel) → M6 (assembly; lock `InferenceResult` before wiring) → M7 (wiring + hardening).

## Deferred error code & behaviour
- `ai_analysis_failed` (422) — `analyzer.py` — zero field mappings inferable / unusable samples.
- `creation_mode: "ai_assisted"` becomes reachable; `review_notes` becomes non-empty (low-confidence mappings, system-defaulted fields, filter rules not inferred).
- TXT-01/TXT-04 casing (AI mode): `mapping_inferrer` emits `from_value` post-TXT-01 (uppercase where TXT-01 targets the field), so the runtime casing contract holds without a config-time check.

## Deferred interfaces
```python
# Ashutosh: schema_detector.py
detect_schema(rows: list[dict]) -> list[FieldSchema]
# Up to 100 rows. data_type via 80% threshold (date > numeric > text).
# format: dominant date pattern (annotate if mixed); DECIMAL(18,N)/integer; VARCHAR(N).
# Ambiguous date order (all components <=12) -> default to declared custodian convention,
# flag low-confidence. mandatory=True (caller adjusts).

# Aarohi: field_mapper.py
infer_field_mapping(internal_rows, source_rows, internal_schema, source_schema)
  -> tuple[list[FieldMapping], list[tuple[dict, dict]], list[str]]
# Pair same-trade records (shared date + numeric agreement within 5% over >=2 fields);
# vote column alignment (equality / within-tolerance / consistent relabel Buy->BTO);
# emit mappings with support; matched_pairs feed mapping_inferrer.

# Mihir: mapping_inferrer.py
infer_txt04_mappings(matched_pairs, field_mapping) -> list[NormalizationRule]
# Consistent (source_value, osyte_value) co-occurrence -> MappingEntry; extract org_id;
# one TXT-04 rule per field needing value translation; from_value in post-TXT-01 form.

# Ashutosh: rule_selector.py
select_normalization_rules(source_rows) -> list[NormalizationRule]
# Catalog rules only: mixed casing->TXT-01; spaces->TXT-02; mixed dates->DT-03;
# timestamps->DT-01+DT-02; negatives->NUM-03; inconsistent decimals->NUM-01+NUM-02;
# null text->TXT-05; null numeric->NUM-04. Never TXT-03/06, NUM-05, DT-05, or TXT-04.

# Mihir: analyzer.py
analyze_feeds(internal_rows, source_feeds) -> InferenceResult
# detect_schema (both) -> infer_field_mapping -> infer_txt04_mappings ->
# select_normalization_rules -> apply system defaults -> generate review_notes.
# AIAnalysisFailedError if zero mappings.
```
`InferenceResult` and the final `review_notes` shape are locked at the start of M6 (added to `models.py` then, with `AIAnalysisFailedError` if not reserved up front).
