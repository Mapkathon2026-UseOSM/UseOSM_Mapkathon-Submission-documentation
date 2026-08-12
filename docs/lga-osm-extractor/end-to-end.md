# End-to-End Walkthrough — lga-osm-extractor

This page traces exactly what happens, function call by function call, for
one real extraction — using the Streamlit app (`app.py`) as the entry point,
since it's the richest interface (the CLI path is identical from
`extract_lga()` onward, just without the UI layer). For how the *data*
changes shape at each stage rather than the call sequence, see
[Data Flow](data-flow.md).

## 1. Process start

`streamlit run app.py` starts a Streamlit server process. `app.py` executes
top-to-bottom **once** at startup (page config, CSS injection, hero text,
form definition, `_cached_extract()`'s definition, `LAYER_STYLES`'s
definition) — none of this touches `lga_extractor` yet. The form renders
with `submitted = False`, so the entire `if submitted:` block is skipped on
this first pass. Streamlit then waits for user interaction, re-running the
whole script from top to bottom on every interaction (Streamlit's standard
rerun-on-interaction model) — this matters for understanding why
`_cached_extract()`'s caching is what prevents redundant work across
reruns, not just across separate user sessions.

## 2. User submits the form

User types `"Akure North"` / `"Ondo"`, clicks **Extract OSM Data**.
Streamlit reruns the script; this time `submitted = True`. The `lga_name`
non-blank check passes, so execution enters the `with st.spinner(...)`
block.

## 3. `_cached_extract("Akure North", "Ondo")` is called

- Streamlit checks its cache for this exact `(lga_name, state_name)` key.
  **First run for this LGA: cache miss.**
- Calls `pipeline.extract_lga(lga_name="Akure North", state_name="Ondo")`
  — this is where control passes into the actual `lga_extractor` package.

## 4. Inside `extract_lga()` — stage 1: boundary

- `output_dir` defaults to `"output/akure_north"` (slugified from
  `lga_name`).
- `tag_config` defaults to `layers.DEFAULT_TAG_CONFIG` (the six standard
  layers).
- Calls `boundary.resolve_boundary(lga_name="Akure North",
  state_name="Ondo", manual_boundary_path=None)`.
  - No manual path given, so this takes the OSM geocode branch: builds the
    query string `"Akure North, Ondo, Nigeria"`, calls
    `ox.geocode_to_gdf(query)` — **this is the pipeline's first network
    call**, hitting OSM's Nominatim geocoding service.
  - Result is passed to `_validate_and_standardize()`: CRS standardized to
    EPSG:4326; hard checks run (centroid-inside-Nigeria, plausible area via
    a call into `clean.resolve_target_crs()` — **the one place `boundary.py`
    calls into `clean.py`**); soft check runs (`display_name` mention).
    Assuming Akure North resolves cleanly (the expected happy path), no
    exception is raised, and a single-row `GeoDataFrame` with
    `boundary_source="osm_geocode:Akure North, Ondo, Nigeria"` and
    `validation_warnings=None` is returned.
- Back in `extract_lga()`: `boundary_source` and
  `boundary_validation_warning` (here, `None`) are extracted from the
  result's columns.

## 5. Stage 2: layer extraction

- Calls `layers.extract_layers(boundary_gdf, tag_config=DEFAULT_TAG_CONFIG,
  strict=False)` (permissive mode, since `app.py` never passes `strict`).
  - A `ThreadPoolExecutor(max_workers=2)` is opened.
  - All six `(layer_name, tags)` pairs are submitted as futures, staggered:
    `roads` and `buildings` (say) start immediately; `waterways` and
    `landuse` start after a 3-second delay; `health_facilities` and
    `schools` after a 6-second delay — this staggering is the mechanism
    that keeps the shared Overpass server from refusing connections under
    burst load (see [`layers.md`](modules/layers.md) for the full
    reasoning).
  - Each `_extract_single_layer()` call independently sleeps its
    `start_delay`, then calls `ox.features_from_polygon(polygon, tags)` —
    **the pipeline's remaining network calls**, one per layer, each
    potentially retried up to 6 times with linear backoff on transient
    connection failures.
  - As each future completes (in *completion* order, not submission
    order), the main thread collects its `(layer_name, gdf, warning,
    error)` tuple into the `layers` dict and `warnings` list.
  - Assuming no genuine failures (the common case), all six layers return
    successfully — some may be empty (e.g. if Akure North genuinely has no
    OSM-tagged waterways), each empty result adds a `"...returned no
    features"` warning but does not raise.
  - Returns `{"roads": gdf, "buildings": gdf, ..., "_warnings": [...]}`.
- Back in `extract_lga()`: warnings are pulled out; since
  `boundary_validation_warning` was `None` in this run, nothing is
  prepended to the warnings list.

## 6. Stage 3: cleaning

- Calls `clean.clean_layers(raw_layers, boundary_gdf=boundary_gdf)`.
  - `resolve_target_crs(boundary_gdf)` is called **once**, resolving to
    `"EPSG:32631"` (Akure North's centroid falls in UTM zone 31N).
  - Each of the six layers is passed through `_clean_single_layer()`
    sequentially (no threading at this stage): null/empty geometry
    dropped, invalid geometry repaired, index flattened, `osmid`/`name`
    standardized, duplicates dropped by geometry, reprojected to
    `EPSG:32631`.
  - For `health_facilities` and `schools` specifically (members of
    `POINT_LAYERS`), `_collapse_areas_to_points()` runs after
    reprojection — any facility mapped as a building-outline `Polygon`
    rather than a point node is replaced with its centroid `Point` here.
    **This is the exact step that fixes the Akure North bug** — before
    this logic existed, all 14 of that LGA's health facilities (mapped as
    polygons) would have failed to collapse and caused downstream
    accessibility analysis to treat them as unreachable.
  - Returns the six cleaned `GeoDataFrame`s, all now in `EPSG:32631`, all
    with the uniform `osmid`/`name`/`geometry` schema.
- Back in `extract_lga()`: `"_warnings"` is popped off the cleaned dict
  (redundant but harmless, since `export_layers()` would skip it anyway).

## 7. Stage 4: export

- Calls `export.export_layers(cleaned, "output/akure_north")`.
  - Creates `output/akure_north/` and `output/akure_north/shapefiles/`.
  - For each non-empty layer: writes `{layer_name}.geojson`; checks
    `_geom_categories_present()`; if `roads` (say) contains both `Point`
    and `LineString` features (traffic signals + road ways — a realistic
    outcome for a real LGA), writes `roads_point.shp` and `roads_line.shp`
    separately rather than one `roads.shp`; every single-category layer
    (e.g. `buildings`, `health_facilities`) writes one plain `.shp` as
    before.
  - **This is the second point data is written to disk** (the first being
    nothing — this is actually the *only* point in the whole pipeline data
    is persisted; everything before this stage exists only in memory for
    the duration of this one Python process).
  - Empty layers recorded under `_skipped`; split layers recorded under
    `_split_layers`.

## 8. CRS resolved again, for logging

- `resolve_target_crs(boundary_gdf)` is called a **second** time, purely to
  have the string `"EPSG:32631"` available to pass into `log_run()` — see
  [`pipeline.md`](modules/pipeline.md)'s gotcha explaining why this isn't
  simply read off one of the cleaned layers instead.

## 9. Stage 5: logging

- Calls `logging_utils.log_run(...)` with everything gathered: LGA/state
  name, tag config, output dir, boundary source, combined warnings,
  the export summary, resolved CRS.
  - `_capture_environment()` runs, recording Python version, platform, and
    the installed versions of `osmnx`/`geopandas`/`shapely`/`fiona`/
    `pandas`.
  - Writes `output/akure_north/run_log.json`.

## 10. `extract_lga()` returns

A summary dict (`lga_name`, `state_name`, `output_dir`, `boundary_source`,
`target_crs`, `exported`, `warnings`, `run_log`) propagates back up through
`_cached_extract()` (which Streamlit now caches under the
`("Akure North", "Ondo")` key for any future rerun/user) to `app.py`.

## 11. Rendering the result

- `st.success(...)` shows a completion message.
- If `result["warnings"]` is non-empty, an expander lists them.
- The preview map is built: `leafmap.foliumap.Map()`, then each exported
  layer's GeoJSON file is added via `m.add_geojson()`, in an order that
  puts `roads` first (for good initial map framing), tracking a
  `zoomed_yet` flag so the *first successfully-added* layer gets
  `zoom_to_layer=True` regardless of which earlier layers turned out to be
  empty (see [`app.md`](app.md) for the full reasoning behind this).
- A zip of the entire `output/akure_north/` directory is built in memory
  and offered via `st.download_button()`.

## Where state lives, summarized

| State | Where it lives | How long it persists |
|---|---|---|
| The resolved boundary polygon | In-memory, inside `extract_lga()`'s local scope | Duration of one `extract_lga()` call only |
| Raw/cleaned layer GeoDataFrames | In-memory | Duration of one `extract_lga()` call only |
| Exported GeoJSON/Shapefile | Disk, under `output/{lga_name}/` | Persists indefinitely (until manually deleted) |
| `run_log.json` | Disk, same directory | Persists indefinitely |
| The `extract_lga()` result summary dict | Streamlit's `@st.cache_data` cache (server memory) | Persists for the life of the Streamlit server process, shared across all sessions/users requesting the same LGA/state |
| Form input values (`lga_name`, `state_name`) | Streamlit's per-session widget state | Persists only for the current browser session |
