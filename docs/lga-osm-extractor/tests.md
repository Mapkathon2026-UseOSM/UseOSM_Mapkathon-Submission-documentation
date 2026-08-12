# Tests — lga-osm-extractor

!!! info "Source"
    `tests/test_extraction.py` (465 lines, 21 test functions)

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
| `test_clean_layers_standard_schema` | Every non-empty cleaned layer has exactly the `geometry`/`osmid`/`name` schema `KEEP_COLUMNS` promises. |
| `test_utm_epsg_for_longitude_known_nigerian_locations` | `utm_epsg_for_longitude()` against four real reference points spanning Nigeria's actual UTM range (Akure/31N, Abuja/32N, Maiduguri/33N, Lagos/31N) — not just the original Ondo State study area, confirming the auto-selection fix generalizes correctly. |
| `test_utm_epsg_for_longitude_zone_boundaries` | Checks right at a zone boundary (5.999°E vs 6.001°E) to rule out an off-by-one error in the zone formula. |
| `test_resolve_target_crs_falls_back_with_no_boundary` | Confirms the `FALLBACK_CRS` (`EPSG:32631`) behavior when no boundary is given — the backward-compatibility path for existing callers. |
| `test_resolve_target_crs_auto_selects_zone_from_boundary` | Confirms a boundary near Abuja resolves to zone 32N (not the Ondo-based default), and a boundary near Akure still resolves to 31N — the core behavior this feature adds, tested both ways. |
| `test_clean_layers_uses_boundary_to_select_crs` | End-to-end: deliberately passes Akure-area *data* alongside an Abuja-area *boundary*, confirming the CRS choice follows the boundary, not the data's own coordinates — an important distinction, since it proves the boundary (not a guess from the data itself) is the authoritative signal. |

Notably **not** directly tested here: `_collapse_areas_to_points()` — the
polygon-to-centroid collapse function central to the Akure North bug fix —
has no test in this file that isolates it directly with mixed Point/Polygon
input. See [Gaps](#gaps-worth-closing) below.

### `layers.py`

| Test | What it verifies |
|---|---|
| `test_extract_layers_permissive_returns_empty_on_failure` | Default (`strict=False`) behavior: a simulated Overpass failure on one layer (`health_facilities`) is caught, recorded as a warning, and that layer returns empty — while other layers (`roads`) still extract successfully. Explicitly asserts this is the original, must-not-regress behavior. |
| `test_extract_layers_strict_raises_on_genuine_failure` | The same simulated failure, but with `strict=True`, must now raise `LayerExtractionError` immediately rather than being silently swallowed. |
| `test_extract_layers_strict_does_not_raise_on_genuine_empty_result` | The critical distinction the whole `strict` feature depends on: a layer that queries *successfully* but genuinely finds zero features must **not** raise, even in strict mode — only an actual exception from the underlying query should raise. Directly guards against `strict` mode being accidentally too aggressive. |

Both failure-simulating tests use `unittest.mock.patch` on
`lga_extractor.layers.ox.features_from_polygon` — patched at the point of
use inside the `layers` module's namespace, not at `osmnx`'s own namespace,
which is the correct pattern for how `layers.py` imports `osmnx` (`import
osmnx as ox`, then calls `ox.features_from_polygon`).

### `boundary.py`

| Test | What it verifies |
|---|---|
| `test_resolve_boundary_accepts_plausible_akure_sized_boundary` | The happy path: a realistically-sized boundary near Akure passes validation cleanly with no warnings — confirms nothing regressed for the actual study area. |
| `test_resolve_boundary_rejects_centroid_outside_nigeria` | A boundary centered near Paris raises `BoundaryResolutionError` matching `"outside Nigeria"` — the geographic bounding-box hard check. |
| `test_resolve_boundary_rejects_implausibly_tiny_area` | A boundary only a few meters across (a buffered point) raises, matching `"implausibly small"` — the minimum-area hard check. |
| `test_resolve_boundary_rejects_implausibly_huge_area` | A boundary roughly the size of the whole country raises, matching `"implausibly large"` — the maximum-area hard check, the opposite failure mode from the tiny-area test. |
| `test_validate_and_standardize_display_name_mismatch_warns_not_raises` | The `display_name` soft check produces a warning (containing the requested LGA name) but never raises — tested directly against `_validate_and_standardize()` rather than `resolve_boundary()`, since manual-file boundaries never carry a `display_name` column at all, so this behavior can only be exercised by calling the internal function directly with a synthetic `display_name`. |

All four boundary tests use `_write_manual_boundary()` (a local helper) to
write a synthetic geometry to a temp GeoJSON file and exercise
`resolve_boundary()`'s `manual_boundary_path` code path — this avoids
needing a live Nominatim call to test validation logic that is, by design,
identical regardless of which resolution path produced the candidate
boundary.

### `export.py`

| Test | What it verifies |
|---|---|
| `test_export_layers_writes_geojson_and_shapefile` | Basic export: GeoJSON and Shapefile paths both exist on disk for a non-empty layer; an empty layer (`schools`) is correctly recorded under `_skipped`, not exported. |
| `test_export_layers_splits_mixed_geometry_types` | **Regression test**, explicitly documented as such in its own docstring: verifies the exact `highway=*` mixed Point/LineString scenario splits into separate `roads_line.shp` and `roads_point.shp` files, both existing on disk, both recorded under the category-keyed `shapefile` dict, and the layer name appears in `_split_layers`. This is the test that would fail if the Shapefile-splitting logic in `export.py` were ever removed or broken — protecting against the exact `pyogrio` `FeatureError` the module's own docstring describes as the original problem. |

### `visualize.py`

| Test | What it verifies |
|---|---|
| `test_strip_mapbox_token_removes_real_token_from_export` | **The most operationally important test in this file.** Builds an actual `KeplerGl` preview map end-to-end (real `keplergl` import, not mocked) and confirms the Mapbox token pattern is genuinely absent from the resulting HTML afterward — not just theoretically stripped by inspecting the function in isolation. Its docstring explicitly notes this caused a real GitHub push-protection failure during development, which is exactly why this test exists as a genuine regression guard rather than a nice-to-have. Uses `pytest.importorskip("keplergl", ...)` so it — and only it — skips cleanly in an environment where `keplergl` can't be imported, without affecting collection of any other test in the file (consistent with `visualize.py`'s own lazy-import design, see [`visualize.md`](modules/visualize.md)). Also asserts `</html>` is still present, confirming the regex-based stripping doesn't corrupt the file. |
| `test_strip_mapbox_token_returns_false_when_no_token_present` | `_strip_mapbox_token()` returns `False` (not an error) on a file with no token — e.g. an already-stripped file, or a future `keplergl` version that stops bundling one. |

### `pipeline.py` (integration)

| Test | What it verifies |
|---|---|
| `test_extract_lga_end_to_end_live_osm` | Marked `@pytest.mark.integration`, excluded from default test runs, requires network access. Runs the full real pipeline for Akure North against live OSM/Nominatim, and asserts: `boundary_source` starts with `"osm_geocode"` (not manual); `run_log` path exists on disk; `target_crs` is exactly `"EPSG:32631"` — confirming the UTM auto-selection gives the *correct real-world* answer for the project's actual original study area, not just a synthetic/mocked one. Run explicitly with `pytest -m integration`. |

## Test Helpers

- **`_dummy_raw_layers()`** — builds a small synthetic `layers_dict`
  resembling `extract_layers()`'s real output shape: two point features
  (hospital, school), one line feature (a residential road), one genuinely
  empty layer (`schools`), plus a `"_warnings"` entry — used across most of
  the `clean.py`/`export.py` tests as shared fixture data.
- **`_synthetic_boundary()`** — a fixed small `box()` boundary near Akure
  (`5.15, 7.2, 5.25, 7.3`), used across the `layers.py` mock-based tests.
- **`_write_manual_boundary(geom, crs="EPSG:4326")`** — writes an arbitrary
  Shapely geometry to a temp GeoJSON file and returns its path, used across
  all four `boundary.py` validation tests to exercise `resolve_boundary()`'s
  `manual_boundary_path` path without a live geocode call.

## Gaps Worth Closing

- **`_collapse_areas_to_points()` has no dedicated unit test in this file.**
  Given this is the function behind the project's most significant
  documented bug fix (see [Known Issues](../reference/known-issues.md)), a
  direct test with mixed Point/Polygon input — asserting only the Polygon
  rows change, and that outputs are true `Point` geometries — would be
  valuable, explicit regression coverage for exactly the failure mode that
  motivated the fix. Current coverage only reaches it indirectly, through
  `clean_layers()` on `POINT_LAYERS`-designated layers, without isolating
  the collapse behavior itself.
- **`app.py` and `cli.py` have no test coverage at all.** Both are thin
  wrappers around `extract_lga()`, which reduces risk, but `cli.py`'s
  argument-parsing and exit-code behavior, and `app.py`'s layer-zoom
  fallback logic (see [`app.md`](app.md)'s detailed writeup of the
  zoom-to-layer bug it works around), are both non-trivial enough to be
  worth at least minimal direct coverage.
- **`logging_utils.py` has no direct test.** `log_run()` and
  `_capture_environment()` are exercised only implicitly via the live
  integration test (which asserts the log file exists, but not its
  contents in detail).
