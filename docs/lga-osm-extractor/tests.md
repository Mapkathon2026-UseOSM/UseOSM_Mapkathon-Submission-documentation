# Tests — lga-osm-extractor

!!! info "Source"
    `tests/test_extraction.py` (1,110 lines, 45 test functions — grew from
    465 lines / 21 functions with the addition of tests for every new
    module and every changed function in this revision, see below)

All tests in this file are pure `pytest` functions (no test classes). One
test is marked `@pytest.mark.integration` and requires live network access
to OSM/Nominatim; every other test is fully offline, using synthetic
`GeoDataFrame`s, temp files, and `unittest.mock.patch` to substitute for
live OSM calls.

## Coverage by module

### `clean.py`

| Test | What it verifies |
|---|---|
| `test_clean_layers_reprojects_and_dedupes` | `clean_layers()` reprojects to the expected CRS and preserves feature counts on synthetic input. |
| `test_clean_layers_standard_schema` | Every non-empty cleaned layer has at least the `geometry`/`osmid`/`name` core schema `CORE_COLUMNS` promises. |
| `test_utm_epsg_for_longitude_known_nigerian_locations` | `utm_epsg_for_longitude()` against four real reference points spanning Nigeria's actual UTM range (Akure/31N, Abuja/32N, Maiduguri/33N, Lagos/31N) — not just the original Ondo State study area, confirming the auto-selection fix generalizes correctly. |
| `test_utm_epsg_for_longitude_zone_boundaries` | Checks right at a zone boundary (5.999°E vs 6.001°E) to rule out an off-by-one error in the zone formula. |
| `test_resolve_target_crs_falls_back_with_no_boundary` | Confirms the `FALLBACK_CRS` (`EPSG:32631`) behavior when no boundary is given — the backward-compatibility path for existing callers. |
| `test_resolve_target_crs_auto_selects_zone_from_boundary` | Confirms a boundary near Abuja resolves to zone 32N (not the Ondo-based default), and a boundary near Akure still resolves to 31N. |
| `test_clean_layers_uses_boundary_to_select_crs` | Deliberately passes Akure-area *data* alongside an Abuja-area *boundary*, confirming the CRS choice follows the boundary, not the data's own coordinates. |
| `test_clean_layers_preserves_semantic_columns_when_present` **(new)** | A layer's `SEMANTIC_COLUMNS`-listed tags (e.g. a road's `surface`, `maxspeed`) that are actually present in the raw input survive cleaning and appear in the output schema. |
| `test_clean_layers_semantic_columns_are_layer_specific` **(new)** | A tag relevant to one layer (e.g. `beds`, a health-facility tag) does **not** leak into an unrelated layer's schema (e.g. `roads`) — confirms `SEMANTIC_COLUMNS.get(layer_name, [])`'s per-layer lookup is genuinely scoped correctly, not accidentally applying one layer's curated list globally. |
| `test_clean_layers_raw_tags_preserves_everything_as_json` **(new)** | `RAW_TAGS_COLUMN` contains a JSON-parseable string, and parsing it back recovers the original tag values — including at least one tag *not* in `SEMANTIC_COLUMNS`, confirming the escape-hatch column genuinely captures more than the curated subset alone. |

Notably **still not** directly tested here: `_collapse_areas_to_points()` —
the polygon-to-centroid collapse function central to the Akure North bug
fix — has no test in this file that isolates it directly with mixed
Point/Polygon input, unchanged from the previous revision. See
[Gaps](#gaps-worth-closing) below.

### `layers.py`

| Test | What it verifies |
|---|---|
| `test_extract_layers_permissive_returns_empty_on_failure` | Default (`strict=False`) behavior: a simulated Overpass failure on one layer is caught, recorded as a warning, and that layer returns empty — while other layers still extract successfully. |
| `test_extract_layers_strict_raises_on_genuine_failure` | The same simulated failure, but with `strict=True`, must now raise `LayerExtractionError` immediately. |
| `test_extract_layers_strict_does_not_raise_on_genuine_empty_result` | A layer that queries *successfully* but genuinely finds zero features must **not** raise, even in strict mode. |
| `test_extract_layers_status_distinguishes_failed_from_empty` **(new)** | The core distinction the new structured `"_status"` dict exists for: a layer that genuinely failed to query is marked `status="failed"`; a layer that queried successfully but found nothing is marked `status="success_empty"` — both look identical as an empty `GeoDataFrame` on their own, and a downstream consumer must be able to tell them apart from `"_status"` alone, without re-deriving it from warning text. |
| `test_extract_layers_emits_started_and_completed_events` **(new)** | A real `on_event` callback (a plain list-appending function, not a mock) receives `stage_started` and `stage_completed` events for each layer, in a form matching [`events.py`](modules/events.md)'s documented schema. |
| `test_extract_layers_emits_retry_events_on_transient_failure` **(new)** | Simulating a transient `ConnectionError` that succeeds on a later attempt produces at least one `retry` event with the correct `attempt`/`max_attempts` fields, before the eventual `stage_completed`. |

Both original failure-simulating tests use `unittest.mock.patch` on
`lga_extractor.layers.ox.features_from_polygon` — patched at the point of
use inside the `layers` module's namespace, not at `osmnx`'s own
namespace, the correct pattern for how `layers.py` imports `osmnx`.

### `boundary.py`

| Test | What it verifies |
|---|---|
| `test_resolve_boundary_accepts_plausible_akure_sized_boundary` | The happy path: a realistically-sized boundary near Akure passes validation cleanly with no warnings. |
| `test_resolve_boundary_rejects_centroid_outside_nigeria` | A boundary centered near Paris raises `BoundaryResolutionError` matching `"outside Nigeria"`. |
| `test_resolve_boundary_rejects_implausibly_tiny_area` | A boundary only a few meters across raises, matching `"implausibly small"`. |
| `test_resolve_boundary_rejects_implausibly_huge_area` | A boundary roughly the size of the whole country raises, matching `"implausibly large"`. |
| `test_validate_and_standardize_display_name_mismatch_warns_not_raises` | The `display_name` soft check produces a warning but never raises. |
| `test_validate_and_standardize_accepts_correct_class_and_type` **(new)** | The happy path for the new hard check: `class="boundary"`, `type="administrative"` passes cleanly. |
| `test_validate_and_standardize_rejects_non_boundary_class` **(new)** | A `class` value other than `"boundary"` (e.g. a road or POI mistakenly matched) raises — the new hard check catching a wrong-feature-type resolution the old geometry/area checks alone couldn't catch. |
| `test_validate_and_standardize_rejects_non_administrative_type` **(new)** | Same hard check, the `type` half — a `type` other than `"administrative"` raises. |
| `test_validate_and_standardize_missing_class_type_columns_is_a_noop` **(new)** | Critical for backward compatibility: when `class`/`type` columns are entirely absent (the manual-boundary-path case, which never carries Nominatim tags), the new hard check is a complete no-op — doesn't raise, doesn't warn, doesn't block the manual path in any way. |
| `test_validate_and_standardize_warns_on_non_relation_osm_type` **(new)** | The new soft check: `osm_type != "relation"` produces a warning, not a raise. |
| `test_validate_and_standardize_accepts_relation_osm_type_silently` **(new)** | The corresponding happy path — `osm_type="relation"` produces zero warnings for this check specifically. |
| `test_validate_and_standardize_warns_on_implausible_admin_level` **(new)** | `admin_level` outside `PLAUSIBLE_ADMIN_LEVEL_RANGE` (3–10) produces a warning. |
| `test_validate_and_standardize_accepts_plausible_admin_level_silently` **(new)** | `admin_level="6"` (Nigeria's conventional LGA level) produces zero warnings for this check. |
| `test_validate_and_standardize_missing_admin_level_produces_no_warning` **(new)** | Directly confirms the deliberate asymmetry documented in [`boundary.md`](modules/boundary.md): an *absent* `admin_level` column produces silence, not a warning — since OSMnx's default geocode result doesn't reliably include this field, warning about its absence would be noise, not signal. |
| `test_validate_and_standardize_treats_nan_metadata_as_absent` **(new)** | Confirms `_first_value_or_none()`'s dual handling of "column missing" and "column present but `NaN`" — both must be treated identically as "we don't have this information" across all three new metadata checks. |

All boundary tests use `_write_manual_boundary()` (a local helper) to
write a synthetic geometry to a temp GeoJSON file and exercise
`resolve_boundary()`'s `manual_boundary_path` code path, or call
`_validate_and_standardize()` directly for the checks that need
Nominatim-style metadata columns a manual file wouldn't carry.

### `export.py`

| Test | What it verifies |
|---|---|
| `test_export_layers_writes_geojson_and_shapefile` | Basic export: GeoJSON and Shapefile paths both exist on disk for a non-empty layer; an empty layer is correctly recorded under `_skipped`. |
| `test_export_layers_splits_mixed_geometry_types` | **Regression test** for the `highway=*` mixed Point/LineString scenario — verifies it splits into separate `roads_line.shp`/`roads_point.shp` files. |
| `test_export_layers_shapefile_stays_core_columns_only` **(new)** | Directly verifies `_shapefile_safe_columns()`'s whole purpose: reads a written `.shp` file back and confirms its columns are exactly `CORE_COLUMNS` — no semantic tag columns, no `raw_tags` — even when the corresponding GeoJSON for the same layer carries the full richer schema. This is the test that would catch a regression silently reintroducing Shapefile field-name truncation/corruption risk. |

### `pipeline.py` and `manifest.py` (new coverage)

| Test | What it verifies |
|---|---|
| `test_extract_lga_end_to_end_live_osm` | Marked `@pytest.mark.integration`, requires network access. Runs the full real pipeline for Akure North against live OSM/Nominatim; asserts `boundary_source`, `run_log` path, and `target_crs` are all correct. |
| `test_extract_lga_emits_full_stage_sequence` **(new, uses `monkeypatch`)** | Confirms a full `extract_lga()` run (with the underlying network/query calls mocked out via `monkeypatch`, so this test runs offline unlike the live-OSM test above) emits events for every expected stage, in a sensible order, ending with `pipeline_completed` carrying the summary dict as its payload. |
| `test_extract_lga_writes_boundary_geojson_and_records_it_in_manifest` **(new)** | End-to-end check that `boundary.geojson` genuinely exists on disk after a run, and that `manifest.json`'s `boundary_path` field points at that exact, existing file — not just that both artifacts exist independently. |
| `test_build_manifest_reconciles_query_and_export_status` **(new)** | Directly tests `manifest.build_manifest()`'s core reconciliation logic: given a `layer_status` dict and an `exported` dict with different information for the same layer, confirms both `feature_count` (post-cleaning) and `feature_count_raw` (query-time) end up correctly, distinctly recorded. |
| `test_build_manifest_carries_boundary_path` **(new)** | `boundary_path`, when passed, appears verbatim in the built manifest dict. |
| `test_build_manifest_boundary_path_defaults_to_none` **(new)** | When `boundary_path` isn't passed, the manifest's field is `None`, not a missing key or an empty string — a specific, checkable default. |
| `test_write_manifest_writes_valid_json` **(new)** | `write_manifest()`'s output is genuinely valid, parseable JSON at the expected path — a basic but necessary sanity check for the function actually consumed by `akure_access.data_contract` on the other repo. |

### `events.py` (new module, new test coverage)

| Test | What it verifies |
|---|---|
| `test_build_stage_order_includes_layers_in_config` **(new)** | `build_stage_order()` produces the correct, ordered stage list for a given `tag_config` — including for a `tag_config` with a *different* set of layers than `DEFAULT_TAG_CONFIG`, confirming the stage list is genuinely derived from the config passed in, not hardcoded. |
| `test_thread_safe_event_queue_drains_events_from_multiple_threads` **(new)** | Spawns 10 real `threading.Thread`s, each calling the same `ThreadSafeEventQueue` instance concurrently, then confirms `drain()` recovers all 10 events with none lost or corrupted — the actual concurrency guarantee this class exists to provide, tested with real threads rather than only reasoned about. |

### `visualize.py`

| Test | What it verifies |
|---|---|
| `test_strip_mapbox_token_removes_real_token_from_export` | Builds an actual `KeplerGl` preview map end-to-end and confirms the Mapbox token pattern is genuinely absent from the resulting HTML afterward. Uses `pytest.importorskip("keplergl", ...)`. |
| `test_strip_mapbox_token_returns_false_when_no_token_present` | `_strip_mapbox_token()` returns `False` (not an error) on a file with no token. |

Unchanged in this revision — no new tests, consistent with `visualize.py`
itself being unchanged.

## Test Helpers

- **`_dummy_raw_layers()`** — builds a small synthetic `layers_dict`
  resembling `extract_layers()`'s real output shape: two point features
  (hospital, school), one line feature (a residential road), one genuinely
  empty layer (`schools`), plus `"_warnings"` and (**new**) a synthetic
  `"_status"` entry — used across most of the `clean.py`/`export.py`/
  `manifest.py` tests as shared fixture data.
- **`_synthetic_boundary()`** — a fixed small `box()` boundary near Akure
  (`5.15, 7.2, 5.25, 7.3`), used across the `layers.py` mock-based tests
  and (**new**) the pipeline-level event/manifest tests.
- **`_write_manual_boundary(geom, crs="EPSG:4326")`** — writes an arbitrary
  Shapely geometry to a temp GeoJSON file and returns its path, used across
  every `boundary.py` validation test to exercise `resolve_boundary()`'s
  `manual_boundary_path` path without a live geocode call. **Still the
  correct helper for the new class/type/admin_level tests too** — those
  tests call `_validate_and_standardize()` directly with synthetic metadata
  columns attached, rather than relying on this helper for that specific
  part, since a manual-file boundary never carries `class`/`type`/
  `admin_level` at all.

## Gaps Worth Closing

- **`_collapse_areas_to_points()` still has no dedicated unit test in this
  file, unchanged from the previous revision.** Given this is the function
  behind the project's most significant documented bug fix (see
  [Known Issues](../reference/known-issues.md)), a direct test with mixed
  Point/Polygon input — asserting only the Polygon rows change, and that
  outputs are true `Point` geometries — would be valuable, explicit
  regression coverage for exactly the failure mode that motivated the fix.
  Current coverage only reaches it indirectly, through `clean_layers()` on
  `POINT_LAYERS`-designated layers, without isolating the collapse behavior
  itself. **This gap persisted through this revision** despite a
  substantial round of new test coverage elsewhere in the same file —
  worth flagging specifically since it means the single most consequential
  bug fix in the whole project still lacks the most direct possible
  regression test for it.
- **`app.py` and `cli.py` still have no test coverage at all, despite both
  growing substantially in this revision.** `app.py` in particular gained
  an entirely new threading/polling architecture
  (`_run_extraction_with_live_progress()`, see [`app.md`](app.md)) with
  real concurrency-correctness properties (the `outcome` dict pattern for
  cross-thread exception propagation, the `thread.is_alive() or not
  events.empty()` race-avoidance condition) that would benefit
  meaningfully from direct testing — these are exactly the kind of subtle,
  hard-to-eyeball-correct patterns unit tests are best at catching if they
  regress. `cli.py`'s argument-parsing and exit-code behavior remains
  similarly untested.
- **`logging_utils.py` still has no direct unit test**, though it's now
  exercised somewhat more thoroughly via `test_extract_lga_emits_full_stage_sequence`
  (which runs the full pipeline, including the `log_run()` call, under
  `monkeypatch` rather than requiring live network access) — an
  improvement over the previous revision's coverage, which only reached
  `log_run()` via the live-network integration test. Still, no test
  directly inspects `run_log.json`'s contents in detail, including its new
  `"layers"` key.
- **New: the cross-repo integration test was not updated for any of this
  revision's new contract surface.** `tests/test_cross_repo_integration.py`
  in the `akure-accessibility-dashboard` repository is byte-identical to
  its pre-revision version — meaning `manifest.json`, `boundary.geojson`,
  and the richer `clean_layers()` schema are all currently unverified by
  the one test whose entire purpose is confirming the two repositories
  actually agree with each other. See
  [Cross-Repo Integration](../cross-repo/integration.md) and
  [Known Issues](../reference/known-issues.md) for the full detail on this
  gap — arguably the single most significant testing gap introduced by
  this revision, precisely because so much of what changed is contract
  surface this specific test exists to protect.
