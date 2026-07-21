# Breakroom Phase 3 — Interfaces & Signatures (M3: Record Validation)

> Achieves **M3** — full record classification with field-level evidence. **Forward spec — M3 is not yet built** (the repo currently has the M1/M2 engine: `pipeline.py` holds `run_check6` only; there is no `find_match`/`classify` yet). It follows the established engine conventions: each step is its own module (`matcher.py`, `classifier.py`) orchestrated by `pipeline.py`, and engine-internal result containers live in `reconciliation_engine/results.py`. Builds on the check-6 output from [phase2-interface.md](phase2-interface.md). AI modules remain deferred (see the AI-capability section of [build-plan.md](build-plan.md)). Multi-layout / multi-file source support is out of scope for this plan (owned separately).

## Who builds what, who uses what

| Module | Built by | Used by |
|---|---|---|
| `reconciliation_engine/tolerance.py` | Aarohi | `reconciliation_engine/matcher.py` (delivered end of Phase 2) |
| `reconciliation_engine/matcher.py` | Ashutosh | `reconciliation_engine/pipeline.py` |
| `reconciliation_engine/classifier.py` | Aarohi | `reconciliation_engine/pipeline.py` |
| `reconciliation_engine/pipeline.py` (M3 extension) | Ashutosh | `reconciliation_api/run_reconciliation.py` (Mihir) |
| `reconciliation_api/run_reconciliation.py` (M3 path) | Mihir | API consumers |

**Agree before Phase 3 build:**
- `PipelineResult` dataclass — Mihir's `run_reconciliation.py` unpacks it per source
- `find_match()` / `compare_fields()` / `classify_record()` signatures
- The per-source iteration + roll-up contract (below), so Mihir's aggregation and Ashutosh's pipeline agree on granularity
- `classify_record()` inputs (row tags + matched Osyte record + comparisons) — Aarohi builds the classifier; Ashutosh's `pipeline.py` calls it

---

## Iteration & roll-up contract

`runReconciliation` runs the pipeline **once per source** against the same Osyte lookup pool, then aggregates:

- **Per source** — one `PipelineResult`; the summary invariant `external_records = auto_match + partial_match + unmatch + duplicate + filtered` holds over its `records`, and its counts become that source's `SourceSummary` in `by_source`.
- **Total** — `summary.total` is the sum across all sources.
- Every `ReconciliationRecord` carries `source_id`. Sources share no state; one source's check-6 `NormalizationError` fails only that source.

---

## Interfaces

```python
# Ashutosh: reconciliation_engine/matcher.py
find_match(source_record: dict, osyte_records: list[dict], composite_key: list[str]) -> dict | None
# source_record is normalised AND already re-keyed to internal labels (the pipeline applies
#   field_mapping to rename external->internal after check 6 — see pipeline note below).
# Build the composite-key tuple from source_record using the internal composite_key labels;
#   scan osyte_records for the record whose same-key tuple matches exactly.
# Returns the matched Osyte record, or None if no key match (-> unmatch downstream).
# Osyte records are already deduplicated (check_duplicates_osyte) and in internal labels.

compare_fields(
    source_record: dict,                # normalised, internal-labelled
    osyte_record: dict,                 # internal-labelled
    field_mapping: list[FieldMapping],  # the source's mapping (for the field pair labels)
    tolerance_rules: list[ToleranceRule],
) -> list[FieldComparison]
# For each mapped NON-key field:
#   - if a ToleranceRule targets that internal field -> evaluate_tolerance(source, osyte, rule)
#   - else -> exact equality
#   Produce a FieldComparison(field="internal / external", internal, external_normalized,
#                             match, tolerance_applied=ToleranceEvaluation|None).
# Returns the list (order follows field_mapping). Key fields are not re-compared (the match
#   already established key equality).

# Aarohi: reconciliation_engine/classifier.py
classify_record(
    record: dict,                       # carries _filter_rule / _is_duplicate / _exclude if set
    matched_osyte: dict | None,
    comparisons: list[FieldComparison] | None,
) -> str
# Strict priority (first wins):
#   filtered   : _exclude is True OR _filter_rule present
#   duplicate  : _is_duplicate is True
#   unmatch    : matched_osyte is None
#   partial_match : any comparison.match is False
#   auto_match : all comparisons pass
# Returns the reconciliation_status string.

# Ashutosh: reconciliation_engine/pipeline.py (M3 extension) — per source
run_pipeline(
    source_rows: list[dict],            # merged rows for the source
    osyte_rows: list[dict],
    external_feed: ExternalFeed,        # the source
    internal_filter_rules: list[FilterRule],
    composite_key: list[str],
    tolerance_rules: list[ToleranceRule],
) -> PipelineResult
# 1. Check 6 for this source (run_check6 from Phase 2) -> Check6Result.
# 2. Re-key each normalised_source_record external->internal using the source's field_mapping
#    (so matcher/classifier see internal labels; composite_key + tolerance targets are internal).
# 3. For each re-keyed record: find_match() -> compare_fields() (on match) -> classify_record().
#    For the pre-classified sets from check 6 (filtered_source_records, duplicate_source_records)
#    classify directly (status already implied by their tag) — do NOT re-run matching.
# 4. Assign a run-unique record_id to EVERY record (matched, unmatched, filtered, duplicate);
#    attach source_id; build external_record from ORIGINAL (pre-normalisation) values with all
#    _-prefixed keys stripped; set matched_internal_record_ref + field_comparison
#    for auto_match/partial_match/duplicate, null for unmatch/filtered.
# Raises NormalizationError (from check 6) -> caller marks this source failed, continues others.
# Returns PipelineResult.

# Mihir: reconciliation_api/run_reconciliation.py (M3 path) — per source
run_reconciliation(request: RunReconciliationRequest) -> RunReconciliationResponse
# 1. get_config(); validate each requested source_id exists (else source_not_found);
#    skip configured sources absent from the request.
# 2. Decode + parse internal feed once -> osyte_rows (shared lookup pool).
# 3. For each requested source: merge_files() the source's files.
# 4. pipeline.run_pipeline(...) -> PipelineResult per source.
#    Catch NormalizationError -> mark that source failed (check 6 failed), continue.
# 5. Aggregate: records = concat of all PipelineResult.records;
#    by_source[source] = that source's status counts; total = sum across sources;
#    basic_validation per file.
# 6. status: "completed" if >=1 source completes; "failed" only if all sources fail
#    OR the internal feed fails.
```

---

## New Dataclass — add to `reconciliation_engine/results.py`

`PipelineResult` is **engine-internal** (never serialised over the API), so it joins the other engine containers in `results.py` alongside the built `FilterResult`, `DuplicateCheckResult`, and `Check6Result` — not `models.py`.

```python
@dataclass
class PipelineResult:
    source_id: str
    records: list[ReconciliationRecord]   # every record, record_id assigned, status classified
    # Status counts are derived from records by the caller when building SourceSummary;
    # PipelineResult stores no separate counts (single source of truth).
```

`ReconciliationRecord` (the API-boundary record model, locked first in `models.py`) already carries `source_id`, `reconciliation_status`, `external_record`, `matched_internal_record_ref`, `field_comparison`. `ToleranceEvaluation` and `FieldComparison` are the only Phase-1 API-boundary models this consumes. `PipelineResult` and `Check6Result` are engine-internal (`results.py`), not boundary models.

---

## Signatures (detail)

### Ashutosh: `reconciliation_engine/matcher.py`
```python
def find_match(source_record, osyte_records, composite_key):
    # key = tuple(source_record[f] for f in composite_key)   # internal labels, normalised values
    # return first osyte_record whose tuple(osyte[f] for f in composite_key) == key, else None
    # (osyte_records already deduped; at most one legitimate key match expected)

def compare_fields(source_record, osyte_record, field_mapping, tolerance_rules):
    # tol_by_field = {r.field_label: r for r in tolerance_rules}
    # for m in field_mapping where m.internal_field not in composite_key:
    #     s = source_record.get(m.internal_field); o = osyte_record.get(m.internal_field)
    #     if m.internal_field in tol_by_field:
    #         te = evaluate_tolerance(s, o, tol_by_field[m.internal_field]); match = te.match
    #     else:
    #         te = None; match = (s == o)
    #     append FieldComparison(field=f"{m.internal_field} / {m.external_field}",
    #                            internal=o, external_normalized=s, match=match, tolerance_applied=te)
```

### Aarohi: `reconciliation_engine/classifier.py`
```python
def classify_record(record, matched_osyte, comparisons):
    if record.get("_exclude") or record.get("_filter_rule"): return "filtered"
    if record.get("_is_duplicate"):                          return "duplicate"
    if matched_osyte is None:                                return "unmatch"
    if any(not c.match for c in comparisons):                return "partial_match"
    return "auto_match"
```

### Ashutosh: `reconciliation_engine/pipeline.py` (M3)
```python
def run_pipeline(source_rows, osyte_rows, external_feed,
                 internal_filter_rules, composite_key, tolerance_rules) -> PipelineResult:
    c6 = run_check6(source_rows, osyte_rows, external_feed, internal_filter_rules, composite_key)
    records = []
    # matched candidates
    for norm in c6.normalised_source_records:
        internal_row = _rekey_external_to_internal(norm, external_feed.field_mapping)
        osyte = find_match(internal_row, c6.osyte_records, composite_key)
        comps = compare_fields(internal_row, osyte, external_feed.field_mapping, tolerance_rules) if osyte else None
        status = classify_record(internal_row, osyte, comps)
        records.append(_build_record(norm, osyte, comps, status, external_feed.source_id))
    # pre-classified sets (no matching)
    for f in c6.filtered_source_records:
        records.append(_build_record(f, None, None, "filtered", external_feed.source_id))
    for d in c6.duplicate_source_records:
        osyte = find_match(_rekey_external_to_internal(d, external_feed.field_mapping), c6.osyte_records, composite_key)
        records.append(_build_record(d, osyte, None, "duplicate", external_feed.source_id))
    _assign_record_ids(records)                 # run-unique
    return PipelineResult(external_feed.source_id, records)
# _build_record: external_record = original pre-normalisation values, _-keys stripped;
#   matched_internal_record_ref + field_comparison populated for auto/partial/duplicate, null otherwise.
```

### Mihir: `reconciliation_api/run_reconciliation.py` (M3)
```python
def run_reconciliation(request) -> RunReconciliationResponse:
    config = get_config(request.config_id)                       # config_not_found -> 404
    _validate_sources_exist(request, config)                     # source_not_found -> 404
    osyte_rows = parse_csv(decode(request.internal_feed_data))
    results, basic_validation = [], []
    internal_ok = _validate_internal(osyte_rows, config, basic_validation)
    for src in request.external_feeds_data:                      # skip configured-but-absent silently
        ext = _find_source(config, src.source_id)
        rows = merge_files(src.files)
        _basic_validate(rows, ext, basic_validation)
        try:
            results.append(run_pipeline(rows, osyte_rows, ext,
                                        config.internal_filter_rules(), config.composite_key,
                                        config.tolerance_rules))
        except NormalizationError:
            _mark_source_failed(src.source_id, basic_validation)  # isolate
    records = [r for pr in results for r in pr.records]
    by_source = _sum_by_source(results)
    total = _sum_total(by_source)
    status = _overall_status(results, internal_ok, request)      # failed only if all fail or internal fails
    return RunReconciliationResponse(..., summary=ReconciliationSummary(total, by_source),
                                     basic_validation=basic_validation, records=records)
```
