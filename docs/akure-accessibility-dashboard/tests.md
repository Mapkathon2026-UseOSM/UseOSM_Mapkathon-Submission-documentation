# Tests — akure-accessibility-dashboard

!!! info "Source"
    `tests/` (1,939 lines across 10 files — grew from 1,243 lines / 6
    files with the addition of test files for every new module, plus new
    tests within every changed file — see below), plus a dedicated
    cross-repo integration test — see [Cross-Repo Integration](../cross-repo/integration.md)
    for that file's own dedicated coverage (unchanged by this revision,
    a real gap — see below)

All tests are pure `pytest` functions operating on small synthetic
`GeoDataFrame`s and graphs — no live OSM/Overpass calls in the standard
suite (one test in `test_network_graph.py` is the sole exception, marked
`@pytest.mark.integration` — the identical convention used in
`lga_extractor`'s own test suite).

## `test_network_graph.py` (350 lines — grew from 287)

| Test | What it verifies |
|---|---|
| `test_mode_config_has_expected_modes` | `MODE_CONFIG` has exactly the three expected mode keys with sane speed values. |
| `test_graph_from_roads_geometry_fallback_assigns_travel_times` | The `roads_gdf` path produces a graph with correctly computed `travel_time_min` edge attributes. |
| `test_graph_from_roads_speed_override` | An explicit `speed_kph` argument overrides `MODE_CONFIG`'s default for that mode. |
| `test_graph_from_roads_source_auto_prefers_roads_gdf` **(new)** | With `source="auto"` (the default) and both `roads_gdf` and a boundary supplied, confirms the `roads_gdf` path is chosen — the core assertion behind this revision's default-path reversal. |
| `test_graph_from_roads_source_roads_gdf_requires_data` **(new)** | `source="roads_gdf"` with empty/missing `roads_gdf` raises `ValueError`, rather than silently falling back to a live query. |
| `test_graph_from_roads_source_live_osm_requires_boundary` **(new)** | `source="live_osm"` with no boundary supplied raises `ValueError`. |
| `test_graph_from_roads_invalid_source_raises` **(new)** | An unrecognized `source` string raises `ValueError` listing valid options. |
| `test_graph_from_roads_snaps_nearly_coincident_endpoints` **(new)** | Two road segments whose endpoints differ only by floating-point noise (well within `snap_tolerance_m`) end up sharing a single node — the core assertion behind the endpoint-snapping fix. |
| `test_graph_from_roads_snap_does_not_merge_distinct_junctions` **(new)** | Two genuinely distinct junctions, further apart than `snap_tolerance_m`, remain separate nodes — the necessary companion test proving the snap tolerance doesn't over-merge. |
| `test_graph_from_roads_invalid_mode_raises` | An unrecognized `mode` string raises `ValueError` with the valid options listed. |
| `test_nearest_facility_distance_and_time_finds_closer_facility` | Given two facilities at different distances from an origin, the nearer one (by network time) is correctly selected. |
| `test_nearest_facility_distance_and_time_handles_empty_facilities` | Empty `facilities_gdf` returns `(inf, inf)` rather than raising. |
| `test_batch_nearest_facility_distances_matches_naive_per_pair_approach` | Verifies the fast multi-source Dijkstra result produces the *same* answer as the slow, naive per-pair approach. |
| `test_batch_nearest_facility_distances_handles_empty_facilities` | Empty facilities returns `{}` rather than raising. |
| `test_batch_nearest_facility_distances_much_faster_than_naive_at_scale` | Directly measures and asserts a real wall-clock speedup of the batch approach at scale. |
| `test_compute_isochrone_polygon_returns_larger_area_for_longer_trip_time` | A longer trip-time budget produces a larger (or equal) reachable-area polygon. |
| `test_compute_isochrone_polygon_returns_none_for_unmatchable_origin` | An origin that can't be snapped to the graph returns `None` rather than raising. |
| `test_build_isochrones_for_facilities_one_row_per_facility_per_trip_time` | Confirms the batch wrapper's output shape. |
| `test_build_isochrones_for_facilities_handles_empty_facilities` | Empty input produces the explicitly-typed empty `GeoDataFrame`. |
| `test_graph_from_roads_live_osm_polygon` | Marked `@pytest.mark.integration` — the one test in this file requiring live network access, building a real graph via `ox.graph_from_polygon()` against a real small area. |

**Notably not directly, isolatedly tested here**: `nearest_graph_node()`'s
KD-tree caching behavior itself and `_has_lengths()` (both unchanged from
the previous revision's coverage gaps).

## `test_scoring.py` (218 lines — grew from 189)

| Test | What it verifies |
|---|---|
| `test_build_grid_covers_boundary` | The generated fishnet grid actually covers the given boundary. |
| `test_build_grid_uses_explicit_target_crs_not_hardcoded_default` **(new)** | Passing an explicit `target_crs` produces a grid in exactly that CRS, not the fallback. |
| `test_build_grid_falls_back_to_epsg_32631_when_crs_not_supplied` **(new)** | Confirms the backward-compatible fallback behavior when `target_crs` is omitted — unchanged output for any existing caller not yet aware of the new parameter. |
| `test_build_grid_respects_custom_cell_size` | A non-default `cell_size_m` produces the expected cell dimensions. |
| `test_add_building_density_counts_correctly` | `building_count` matches a hand-verifiable count. |
| `test_add_building_density_handles_empty_buildings` | Empty `buildings_gdf` produces `building_count = 0` everywhere. |
| `test_access_deficit_score_composite_logic` | The core 0/1/2 scoring logic. |
| `test_access_deficit_score_mode_suffix_columns` | Mode-suffixed output columns are correctly named and populated. |
| `test_access_deficit_score_treats_inf_as_underserved` | Confirms `inf` travel time is correctly scored as underserved. |
| `test_sanitize_for_export_does_not_break_deficit_scoring` | Correct call order doesn't corrupt already-computed scores. |
| `test_sanitize_for_export_called_before_scoring_gives_wrong_result` | Directly demonstrates the actual wrong result from the incorrect call order. |

**A real, worth-flagging gap this revision introduces:** neither
`add_access_times()`'s new `source` parameter nor its CRS-branch fix
(`G.graph.get("source") == "live_osm"`, see [`scoring.md`](modules/accessibility/scoring.md))
has a dedicated test at the `scoring.py` level in this file. The `source`
parameter's branching logic *is* tested, but one layer down, in
`network_graph.graph_from_roads()` itself (see above) — `add_access_times()`
just threads the argument through without adding logic of its own, so the
underlying branching is covered, but the specific fact that
`add_access_times()` passes it through correctly, and that its CRS-branch
condition genuinely reads the graph's `source` attribute rather than the
old `boundary_polygon_wgs84 is not None` check, is currently verified only
indirectly, via the cross-repo integration test's end-to-end run.

## `test_completeness.py` (182 lines — unchanged)

**This file is byte-identical to its pre-revision version.** No new tests
were added here, consistent with `grid_check.py`'s own source diff being
limited to a config-derivation refactor with no behavior change (see
[`grid_check.md`](modules/completeness/grid_check.md)). The six existing
tests (flagging logic, summary percentages, the spatial-index-vs-naive-scan
correctness check) remain accurate and unchanged.

## `test_insights.py` (175 lines — line count unchanged, content changed)

| Test | What it verifies |
|---|---|
| `test_describe_deficit_map_returns_real_numbers` | The generated caption text actually contains the real computed percentages. |
| `test_describe_deficit_map_missing_mode_column_does_not_crash` | The "not yet scored" early-return path produces a graceful message. |
| `test_describe_deficit_map_reports_directional_skew` | `_compass_skew()`'s integration into the full caption. |
| `test_describe_continuous_map_returns_real_numbers` | Same real-numbers check for the continuous-map caption. |
| `test_describe_continuous_map_all_nan_column_does_not_crash` | The "column exists but every value is null" edge case. |
| `test_describe_continuous_map_missing_column_does_not_crash` | The "column doesn't exist at all" case. |
| `test_describe_completeness_map_returns_real_numbers` | Real-numbers check for the completeness caption. |
| `test_describe_completeness_map_missing_column_does_not_crash` | Graceful handling when completeness hasn't been run yet. |
| `test_describe_mode_comparison_chart_normal_case` | The standard ranking-sentence path. |
| `test_describe_mode_comparison_chart_handles_ties_gracefully` | The near-tie branch. |
| `test_describe_mode_comparison_chart_empty_input_does_not_crash` | Empty `mode_stats` produces the graceful "no data" message. |
| `test_describe_interactive_view_dispatches_correctly` | All three `view_choice` values route correctly. |
| `test_no_em_dash_in_any_generated_caption` | A style-consistency test enforced across every caption-generating function. |
| `test_describe_completeness_map_uses_correct_article` | Direct test of `_article()`'s behavior in generated caption text. |
| `test_default_thresholds_match_notebook_config` | **This test's purpose has shifted, worth noting explicitly.** Previously, this test guarded against a real, live drift risk between two independently-maintained copies of the same values. **Now that `DEFAULT_THRESHOLDS_MIN` is config-derived** (see [`insights.md`](modules/insights.md) and [`config.md`](modules/config.md)), the two values it compares can no longer actually diverge by construction — this test still passes and still has value as an explicit, checkable assertion of that fact, but it's no longer protecting against an active risk, more confirming a structural guarantee. |

**New coverage gap**: `describe_settlement_proxy_limitation()` — the one
new function added to this module in this revision — has no dedicated
test in this file.

## `test_static_maps.py` (202 lines — unchanged)

**This file is byte-identical to its pre-revision version**, consistent
with `static_maps.py` itself being unchanged (confirmed by direct diff).
All fourteen existing tests remain accurate, including the
`_ensure_lonlat()` hang-bug regression test and the correctness-vs-naive
spatial join comparison.

## `test_config.py` — does not exist (new gap)

**`config.py`, despite being a new, foundational module five other
modules now depend on, has no dedicated test file at all.** Its
`_deep_merge()` recursive-merge logic, its three-tier resolution order
(parameter > environment variable > file), its `reload=True` cache-bypass
behavior, and its malformed-YAML-falls-back-to-hardcoded-defaults path are
all currently unverified by any direct test — coverage of `config.py`'s
correctness is entirely incidental, via whatever tests happen to exercise
the five modules that import constants derived from it. Given how much
this module's correct behavior is now load-bearing across the package
(five modules' default values trace back to it), this is arguably the
single most significant testing gap this revision introduces.

## `test_data_contract.py` (188 lines, new file, 10 tests)

| Test | What it verifies |
|---|---|
| `test_read_manifest_returns_none_when_missing` | A missing `manifest.json` produces `None`, not an exception. |
| `test_read_manifest_reads_written_file` | A genuine manifest file round-trips correctly through `read_manifest()`. |
| `test_resolve_crs_from_manifest_uses_manifest_value` | The happy path: `target_crs` is read correctly from a present manifest. |
| `test_resolve_crs_from_manifest_falls_back_and_warns_when_missing` | No manifest at all → warning + fallback CRS returned. |
| `test_resolve_crs_from_manifest_falls_back_and_warns_when_field_missing` | Manifest present but no `target_crs` field (an older extractor version) → a distinct warning + fallback. |
| `test_resolve_crs_from_manifest_custom_fallback` | A caller-supplied `fallback` argument is honored instead of the module's own default. |
| `test_resolve_boundary_path_from_manifest_uses_manifest_value` | The happy path for boundary path resolution. |
| `test_resolve_boundary_path_from_manifest_returns_none_and_warns_when_manifest_missing` | No manifest → `None` + warning. |
| `test_resolve_boundary_path_from_manifest_returns_none_and_warns_when_field_missing` | Manifest present, no `boundary_path` field → `None` + warning. |
| `test_resolve_boundary_path_from_manifest_returns_none_and_warns_when_file_missing` | `boundary_path` recorded in the manifest, but nothing exists at that path on disk → `None` + warning — the third, most subtle failure mode this function guards against. |

Notably, this test file does **not** test that
`resolve_boundary_path_from_manifest()` refuses to fall back to a live
`resolve_boundary()` call — a reasonable omission, since that's precisely
the *absence* of behavior the function is designed around (see
[`data_contract.md`](modules/data_contract.md)'s dedicated explanation);
there's no live-fallback code path to exercise.

## `test_facility_classification.py` (148 lines, new file, 23 tests across 4 classes)

| Test class | What it covers |
|---|---|
| `TestClassifyHealthFacility` (8 tests) | Hospital-via-`amenity`, hospital-via-`healthcare` tag, pharmacy, clinic/doctors-as-primary-care, unrecognized tags → `HEALTH_OTHER` (not an error), both inputs `None` → `HEALTH_OTHER`, case-insensitivity, and — directly — the hospital-over-pharmacy priority-ordering rule (`test_hospital_takes_priority_over_pharmacy_tag`). |
| `TestClassifySchool` (6 tests) | ISCED levels 0/1 → Primary, 2/3 → Secondary, semicolon-separated multi-value lists use only the first value, the `school` tag fallback when `isced:level` is absent, `isced:level`'s priority over the `school` tag when both are present, and no signal at all → `EDU_OTHER`. |
| `TestAddFacilityClass` (6 tests) | Real column classification for both health and education, missing semantic columns classify as `*_OTHER` rather than raising, empty input produces an empty output with the `facility_class` column still present (correct dtype), an invalid `kind` raises `ValueError`, and the function does not mutate its input. |
| `TestSummarizeFacilityClasses` (3 tests) | Per-class counts, the explicit `KeyError` when `facility_class` isn't present, and empty input returns `{}`. |

This is, notably, the **most thoroughly tested** of the five new modules —
23 tests for a 192-line module is a high test-to-code ratio, reflecting
(consistent with the module's own emphasis) how much of its logic is
priority-ordering and edge-case handling in two small, high-stakes
classification functions.

## `test_sensitivity.py` (108 lines, new file, 8 tests)

| Test | What it verifies |
|---|---|
| `test_run_threshold_sensitivity_reuses_already_computed_times` | Confirms the cheap-sweep claim directly: no new routing/graph computation occurs, only re-derivation from already-present time columns. |
| `test_run_threshold_sensitivity_missing_time_column_raises_clear_error` | The `KeyError` with an actionable message when `add_access_times()` hasn't been run for the requested mode. |
| `test_run_threshold_sensitivity_first_row_jaccard_is_always_one` | The first-tested threshold's Jaccard value is `1.0` by construction (compared against itself). |
| `test_run_threshold_sensitivity_stable_underserved_set_gives_high_jaccard` | A synthetic scenario where the underserved cell set genuinely doesn't change across thresholds produces Jaccard values near `1.0`. |
| `test_run_threshold_sensitivity_unsettled_cells_excluded_from_denominator` | Confirms unsettled cells don't dilute the reported percentages. |
| `test_summarize_robustness_reports_robust_for_stable_set` | The "robust" sentence triggers correctly for a stable sweep. |
| `test_summarize_robustness_reports_sensitive_for_unstable_set` | The "sensitive" sentence triggers correctly for an unstable sweep. |
| `test_summarize_robustness_handles_empty_dataframe` | The empty-input fallback string, avoiding an error on `.min()`/`.max()` against nothing. |

**Notably absent**: no test exercises `run_speed_sensitivity()` directly —
the more expensive of the two sweep functions, requiring a full graph
rebuild and Dijkstra pass per candidate speed, is untested at the unit
level, likely because doing so meaningfully within a fast test suite would
require either mocking the routing stack or accepting real per-test
runtime cost the file's other seven tests deliberately avoid.

## `test_status.py` (160 lines, new file, 16 tests across 3 classes + 1 module-level function)

| Test class | What it covers |
|---|---|
| `TestClassifyCellStatus` (6 tests) | Unsettled cell → `UNKNOWN` regardless of other flags (tested with three different "unsettled" input shapes: `0`, `None`, `NaN`); a served cell; underserved without a completeness flag → confirmed `UNDERSERVED`; underserved *with* a positively-`True` completeness flag → `POTENTIAL_DATA_GAP` (the docstring calls this out as "the core distinction this whole module exists to draw"); a `NaN`/missing completeness flag does **not** escalate to `POTENTIAL_DATA_GAP` (directly guards the exact behavior [`status.md`](modules/status.md) flags as the easiest thing to get backwards); the underserved flag itself being `NaN` → `UNKNOWN`. |
| `TestAddAccessStatus` (5 tests) | All four categories produced correctly on a small synthetic grid; the mode-suffixed-vs-unsuffixed column fallback (mirroring `scoring.py`'s own convention) for `mode="walk"`; the explicit `KeyError` naming `add_access_deficit_score()` when the underserved column is missing entirely; a missing completeness column triggers a warning **and** defaults every underserved cell to `UNDERSERVED` rather than `POTENTIAL_DATA_GAP`; the function does not mutate its input. |
| `TestSummarizeStatus` (3 tests) | Percentages scoped to settled cells only (with a real, hand-verifiable 33.3%/33.3%/33.3% split on synthetic data); the explicit `KeyError` naming `add_access_status()`; the minimal-shape return for zero settled cells. |
| `test_status_display_has_an_entry_for_every_category` (module-level) | A "cheap insurance" test, in its own docstring's words: every one of the four status constants has a corresponding `STATUS_DISPLAY` entry with both `color` and `label` keys — guards against a future dashboard/legend integration silently missing a category. |

## Gaps Worth Closing

- **`config.py` has no dedicated test file at all** — see the dedicated
  section above. Given five other modules now derive default values from
  it, this is the most significant net-new testing gap in this revision.
- **`dashboard/app.py` still has zero direct test coverage**, unchanged
  from before this revision — and, unlike the module-level code, is now
  further behind the rest of the package: five new modules exist as
  tested library capabilities that `dashboard/app.py` doesn't yet call at
  all, so even indirect coverage of "does the dashboard's own logic
  interact correctly with the new capabilities" doesn't exist, because
  that interaction doesn't exist yet either.
- **The cross-repo integration test was not updated for any of this
  revision's new contract surface.** `tests/test_cross_repo_integration.py`
  is byte-identical to its pre-revision version — `manifest.json`,
  `boundary.geojson`, and the richer `clean_layers()` schema (semantic
  columns, `raw_tags`) are all currently unverified by the one test whose
  entire purpose is confirming the two repositories actually agree with
  each other. See [Cross-Repo Integration](../cross-repo/integration.md)
  and [Known Issues](../reference/known-issues.md) — this is very likely
  the single most significant testing gap introduced across *both*
  repositories by this revision, since it's specifically the test designed
  to catch exactly the kind of cross-repo contract drift this revision's
  changes are most at risk of introducing.
- **`add_access_times()`'s new `source` parameter and CRS-branch fix have
  no dedicated test at the `scoring.py` level** — see the note under
  `test_scoring.py` above. The underlying branching logic is tested one
  layer down in `network_graph.py`, but `add_access_times()`'s own correct
  pass-through and its corrected CRS-branch condition are currently
  verified only indirectly, via the cross-repo integration test.
- **`insights.describe_settlement_proxy_limitation()` has no dedicated
  test.** A minor gap for a small, fixed-string-returning function, but
  worth a one-line smoke test confirming it returns a non-empty string, if
  only to catch an accidental typo breaking the function outright.
- **`run_speed_sensitivity()` has no direct unit test**, unlike its
  cheaper sibling `run_threshold_sensitivity()` — see the note under
  `test_sensitivity.py` above.
- **The palette-duplication risk between `dashboard/app.py` and
  `static_maps.py`** (see [`dashboard-app.md`](dashboard-app.md)'s
  gotchas) still has no test guarding against the two independently-defined
  color constant sets drifting apart — unchanged from before this
  revision, and now a slightly sharper contrast given `DEFAULT_THRESHOLDS_MIN`'s
  equivalent drift risk *was* resolved via `config.py` in this same
  revision. The same centralization pattern could close this gap too, but
  hasn't yet.
