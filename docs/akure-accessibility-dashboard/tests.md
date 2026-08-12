# Tests — akure-accessibility-dashboard

!!! info "Source"
    `tests/` (1,243 lines across 6 files, plus a dedicated cross-repo
    integration test — see [Cross-Repo Integration](../cross-repo/integration.md)
    for that file's own dedicated coverage)

All tests are pure `pytest` functions operating on small synthetic
`GeoDataFrame`s and graphs — no live OSM/Overpass calls in the standard
suite (one test in `test_network_graph.py` is the sole exception, marked
appropriately).

## `test_network_graph.py` (287 lines)

| Test | What it verifies |
|---|---|
| `test_mode_config_has_expected_modes` | `MODE_CONFIG` has exactly the three expected mode keys with sane speed values. |
| `test_graph_from_roads_geometry_fallback_assigns_travel_times` | Path 2 (`_graph_from_geometries()`, no boundary given) produces a graph with correctly computed `travel_time_min` edge attributes. |
| `test_graph_from_roads_speed_override` | An explicit `speed_kph` argument overrides `MODE_CONFIG`'s default for that mode. |
| `test_graph_from_roads_invalid_mode_raises` | An unrecognized `mode` string raises `ValueError` with the valid options listed. |
| `test_nearest_facility_distance_and_time_finds_closer_facility` | Given two facilities at different distances from an origin, the nearer one (by network time) is correctly selected. |
| `test_nearest_facility_distance_and_time_handles_empty_facilities` | Empty `facilities_gdf` returns `(inf, inf)` rather than raising. |
| `test_batch_nearest_facility_distances_matches_naive_per_pair_approach` | **The most important correctness test for the module's central performance optimization**: verifies the fast multi-source Dijkstra result (`batch_nearest_facility_distances()` + `lookup_nearest_distance_time()`) produces the *same* answer as the slow, naive per-pair approach (`nearest_facility_distance_and_time()`, called once per origin) on the same synthetic graph — direct proof the optimization documented at length in [`isochrones.md`](modules/accessibility/isochrones.md) doesn't trade away correctness for speed. |
| `test_batch_nearest_facility_distances_handles_empty_facilities` | Empty facilities returns `{}` rather than raising. |
| `test_batch_nearest_facility_distances_much_faster_than_naive_at_scale` | Directly measures and asserts a real wall-clock speedup of the batch approach over the naive approach on a larger synthetic graph — turning the module's own prose claims ("over an hour" vs. "roughly one Dijkstra run") into an enforced, automatically-checked regression guard, not just documentation. |
| `test_compute_isochrone_polygon_returns_larger_area_for_longer_trip_time` | A sanity check on the convex-hull isochrone approximation: a longer trip-time budget produces a larger (or equal) reachable-area polygon, as expected. |
| `test_compute_isochrone_polygon_returns_none_for_unmatchable_origin` | An origin that can't be snapped to the graph returns `None` rather than raising. |
| `test_build_isochrones_for_facilities_one_row_per_facility_per_trip_time` | Confirms the batch wrapper's output shape (one row per successful `(facility, trip_time)` pair). |
| `test_build_isochrones_for_facilities_handles_empty_facilities` | Empty input produces the explicitly-typed empty `GeoDataFrame` (with correct columns/CRS) documented in [`isochrones.md`](modules/accessibility/isochrones.md), not a bare `None` or untyped empty structure. |
| `test_graph_from_roads_live_osm_polygon` | The one test in this file requiring live network access — builds a real graph via `ox.graph_from_polygon()` against a real small area. Presumably marked/skippable similarly to `lga_extractor`'s own `@pytest.mark.integration` convention, though worth confirming this file uses the same marker name for consistency across both repos' test suites. |

**Notably not directly, isolatedly tested here**: `nearest_graph_node()`'s
KD-tree caching behavior itself (exercised only indirectly, through every
other function that calls it) and `_has_lengths()` (a defensive fallback
not normally triggered — see [`network_graph.md`](modules/accessibility/network_graph.md)).

## `test_scoring.py` (189 lines)

| Test | What it verifies |
|---|---|
| `test_build_grid_covers_boundary` | The generated fishnet grid actually covers the given boundary. |
| `test_build_grid_respects_custom_cell_size` | A non-default `cell_size_m` produces the expected cell dimensions. |
| `test_add_building_density_counts_correctly` | `building_count` matches a hand-verifiable count for synthetic building points against known grid cells. |
| `test_add_building_density_handles_empty_buildings` | Empty `buildings_gdf` produces `building_count = 0` everywhere, not an error. |
| `test_access_deficit_score_composite_logic` | The core 0/1/2 scoring logic: cells underserved for zero, one, or both services get the correct composite score. |
| `test_access_deficit_score_mode_suffix_columns` | Mode-suffixed output columns are correctly named and populated. |
| `test_access_deficit_score_treats_inf_as_underserved` | **Direct test of the `inf`-means-unreachable convention**: confirms a cell with `inf` travel time is correctly scored as underserved (`1` or `2`), not silently treated as served — the exact failure mode the module's own inline comments warn about repeatedly (see [`scoring.md`](modules/accessibility/scoring.md)). |
| `test_sanitize_for_export_does_not_break_deficit_scoring` | Confirms calling `sanitize_for_export()` **after** scoring (the correct order) doesn't corrupt the already-computed deficit scores. |
| `test_sanitize_for_export_called_before_scoring_gives_wrong_result` | **This is the single most valuable test in the entire test suite for this repository.** It doesn't merely test the correct call order — it directly demonstrates and asserts the *actual wrong result* produced by calling `sanitize_for_export()` too early (before `add_access_deficit_score()`), turning the module's extensively-documented ordering-constraint gotcha (see [`scoring.md`](modules/accessibility/scoring.md)) into an executable, permanent regression guard rather than a comment a future editor could miss. Any future refactor that accidentally breaks this ordering, or any code path that calls these functions in the wrong sequence, would be caught here. |

## `test_completeness.py` (182 lines)

| Test | What it verifies |
|---|---|
| `test_flag_completeness_flags_settled_unmapped_cell` | A settled cell with no nearby facility is correctly flagged. |
| `test_flag_completeness_does_not_flag_when_facility_nearby` | A settled cell *with* a nearby facility is correctly not flagged — the negative case, equally important to test as the positive one. |
| `test_flag_completeness_ignores_unsettled_cells` | An unsettled cell (below the building threshold) is never flagged, regardless of facility proximity. |
| `test_summarize_completeness_reports_correct_percentages` | The summary dict's percentages match hand-computed expected values. |
| `test_summarize_completeness_handles_no_settled_cells` | The documented minimal-shape return (no percentage keys) when there are zero settled cells — see [`grid_check.md`](modules/completeness/grid_check.md)'s own gotcha about this inconsistent return shape. |
| `test_spatial_index_flagging_matches_naive_linear_scan` | The same correctness-vs-performance pairing pattern seen in `test_network_graph.py`: confirms `_flag_via_spatial_index()`'s `sjoin_nearest()`-based approach produces the same result as a naive linear distance scan would, directly validating the module's own documented performance reasoning without sacrificing correctness. |

## `test_insights.py` (175 lines)

| Test | What it verifies |
|---|---|
| `test_describe_deficit_map_returns_real_numbers` | The generated caption text actually contains the real computed percentages, not placeholder text. |
| `test_describe_deficit_map_missing_mode_column_does_not_crash` | The "not yet scored" early-return path produces a graceful message, not an exception. |
| `test_describe_deficit_map_reports_directional_skew` | `_compass_skew()`'s integration into the full caption — a deliberately-skewed synthetic dataset produces a directional phrase in the output. |
| `test_describe_continuous_map_returns_real_numbers` | Same real-numbers check for the continuous-map caption. |
| `test_describe_continuous_map_all_nan_column_does_not_crash` | The specific "column exists but every value is null" edge case (distinct from "column missing entirely") is handled gracefully — directly testing the `.notna().sum() == 0` check documented in [`insights.md`](modules/insights.md). |
| `test_describe_continuous_map_missing_column_does_not_crash` | The simpler "column doesn't exist at all" case. |
| `test_describe_completeness_map_returns_real_numbers` | Real-numbers check for the completeness caption. |
| `test_describe_completeness_map_missing_column_does_not_crash` | Graceful handling when completeness hasn't been run yet. |
| `test_describe_mode_comparison_chart_normal_case` | The standard ranking-sentence path. |
| `test_describe_mode_comparison_chart_handles_ties_gracefully` | **Direct test of the near-tie branch** (`abs(worst_pct - best_pct) < 0.5`) — confirms the "comparably restrictive across every mode" phrasing actually triggers at the documented threshold, rather than producing the nonsensical "highest at X%, lowest at X%" sentence the branch exists to avoid. |
| `test_describe_mode_comparison_chart_empty_input_does_not_crash` | Empty `mode_stats` produces the graceful "no data" message. |
| `test_describe_interactive_view_dispatches_correctly` | All three `view_choice` values route to the correct underlying caption function. |
| `test_no_em_dash_in_any_generated_caption` | **A style-consistency test, not a correctness test** — presumably enforcing a house style rule against em-dashes in generated prose, run across every caption-generating function to catch any accidentally-introduced em-dash in future edits to the caption templates. |
| `test_describe_completeness_map_uses_correct_article` | Direct test of `_article()`'s "a health facility" vs. "an education facility" behavior in the actual generated caption text, not just the helper function in isolation. |
| `test_default_thresholds_match_notebook_config` | **Directly guards against the manual-sync drift risk** documented in [`insights.md`](modules/insights.md): confirms `DEFAULT_THRESHOLDS_MIN` here actually matches whatever `ACCESS_THRESHOLDS_MIN` is configured as in the analysis notebook, catching the exact "two independently-maintained copies of the same values" drift scenario before it can silently produce mismatched caption phrasing. |

## `test_static_maps.py` (202 lines)

| Test | What it verifies |
|---|---|
| `test_ensure_lonlat_reprojects_projected_input` | `_ensure_lonlat()` actually reprojects UTM input to EPSG:4326. |
| `test_ensure_lonlat_leaves_already_lonlat_input_unchanged` | No-op on already-correct input (no unnecessary reprojection). |
| `test_scale_bar_and_gridlines_do_not_hang_on_projected_bounds` (uses `monkeypatch`) | **Direct regression test for the exact failure mode `_ensure_lonlat()`'s own docstring describes** — confirms passing projected (UTM-meter-scale) bounds to the gridline/scale-bar functions doesn't hang attempting to draw tens of thousands of gridlines, the specific bug caught during testing against real Akure North data (see [`static_maps.md`](modules/visualization/static_maps.md)). |
| `test_plot_deficit_map_saves_file` (uses `tmp_path`) | Basic save-to-disk smoke test. |
| `test_plot_deficit_map_missing_column_raises_clear_error` | The explicit `KeyError` with an actionable message, not a generic pandas error. |
| `test_plot_continuous_map_excludes_nan_and_unsettled` | Confirms `NaN`/unsettled cells are actually excluded from the plotted layer, not rendered with a placeholder value. |
| `test_plot_continuous_map_all_nan_raises_clear_error` | The all-null-column edge case raises `ValueError` with a clear message. |
| `test_plot_completeness_map_saves_file` | Basic smoke test. |
| `test_plot_mode_comparison_chart_saves_file` | Basic smoke test. |
| `test_plot_mode_comparison_chart_empty_input_raises` | Empty `mode_stats` raises `ValueError`. |
| `test_generate_all_static_outputs_produces_expected_file_count` | The orchestrator's conditional-generation logic produces exactly the expected number of files given a specific synthetic input's mode/service/completeness column combination. |
| `test_generate_all_static_outputs_skips_web_tier_when_disabled` | `web_dpi=None` correctly skips web-tier generation entirely. |
| `test_generate_all_static_outputs_uses_lga_first_title_convention` (uses `monkeypatch`) | Confirms every generated figure's title follows the `_map_title()` `"{LGA}: {metric}"` convention consistently. |
| `test_generate_all_static_outputs_skips_all_nan_service_layer` | The conditional-generation logic correctly skips a service/mode combination where the column exists but is entirely null. |
| `test_deficit_palette_and_labels_length_match` | A simple consistency check: `DEFICIT_PALETTES` and `DEFICIT_LABELS` have the same length (three colors, three labels) — cheap insurance against an off-by-one mismatch between colors and their labels. |

**Notably not covered here**: `add_osm_basemap()`'s actual tile-fetch-failure
recovery path is marked `# pragma: no cover` in the source itself (see
[`static_maps.md`](modules/visualization/static_maps.md)), meaning that
specific network-dependent fallback branch is deliberately excluded from
this file's coverage accounting, not silently missed.

## Gaps Worth Closing

- **`dashboard/app.py` has zero direct test coverage** — reasonable given
  it's almost entirely Streamlit UI/layout code over already-tested data,
  but the file itself documents two real regression bugs that were fixed
  manually after being noticed (the Findings Summary LGA-scope bug, and
  the single-image full-width-stretch layout bug — see
  [`dashboard-app.md`](dashboard-app.md)'s Gotchas) — both are exactly the
  class of state/layout bug that even lightweight `streamlit.testing`
  framework smoke tests could plausibly have caught earlier.
- **The palette-duplication risk between `dashboard/app.py` and
  `static_maps.py`** (see [`dashboard-app.md`](dashboard-app.md)'s
  gotchas) has no test guarding against the two independently-defined
  color constant sets drifting apart — unlike the very similar
  `DEFAULT_THRESHOLDS_MIN`-vs-notebook-config drift risk in `insights.py`,
  which *is* directly tested (`test_default_thresholds_match_notebook_config`).
  Adding an equivalent direct-equality test between
  `dashboard.app.DEFICIT_COLORS` and
  `visualization.static_maps.DEFICIT_PALETTES` would close a real, currently
  unguarded gap using the exact same pattern already proven useful
  elsewhere in this suite.
- **`grid_check.py`'s two different settlement thresholds** (`building_count > 0`
  used in `scoring.py`/`dashboard/app.py` vs. `building_count >= 3` in
  `grid_check.py` itself — see [`grid_check.md`](modules/completeness/grid_check.md))
  has no test explicitly documenting or asserting this intentional
  difference — a test confirming a cell with 1-2 buildings is scored by
  `accessibility` but excluded from `completeness` flagging would make this
  documented-but-implicit design decision an explicit, checked contract.
