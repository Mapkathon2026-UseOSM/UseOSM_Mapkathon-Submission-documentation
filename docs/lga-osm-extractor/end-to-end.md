# End-to-End Walkthrough — lga-osm-extractor

This page traces exactly what happens, function call by function call, for
one real extraction — using the Streamlit app (`app.py`) as the entry
point, since it's the richest interface (the CLI path is identical from
`extract_lga()` onward, just without the UI layer, and without wiring up
`on_event` to anything — see [`cli.md`](cli.md)). For how the *data*
changes shape at each stage rather than the call sequence, see
[Data Flow](data-flow.md).

**This walkthrough reflects the revision that added `events.py`,
`manifest.py`, `boundary.geojson` export, and `app.py`'s live-progress UI
rework.** The stage sequence itself (boundary → layers → clean → export →
log) is unchanged; what's new is the observability layer running alongside
it, plus two additional artifacts written to disk.

## Sequence Overview

```mermaid
sequenceDiagram
    participant User
    participant Streamlit as app.py (Streamlit, main thread)
    participant Thread as background thread
    participant Pipeline as pipeline.extract_lga()
    participant Boundary as boundary.py
    participant Layers as layers.py
    participant Clean as clean.py
    participant Export as export.py
    participant Manifest as manifest.py
    participant Log as logging_utils.py
    participant Overpass as OSM Overpass API

    Note over Streamlit: On app load: st.set_page_config,\nCSS injection, hero copy, form render.\nNo pipeline code runs yet.
    User->>Streamlit: Types LGA name (+ optional state), clicks "Extract OSM Data"
    Streamlit->>Streamlit: Validate lga_name non-empty
    Streamlit->>Streamlit: check manual dict cache for (lga, state)
    alt cache hit
        Streamlit-->>User: instant result, NO progress UI shown
    else cache miss
        Streamlit->>Streamlit: build stage checklist via events.build_stage_order()
        Streamlit->>Thread: threading.Thread(target=_worker), events = ThreadSafeEventQueue()
        activate Thread
        Thread->>Pipeline: extract_lga(..., on_event=events)

        Pipeline->>Pipeline: emit stage_started("boundary")
        Pipeline->>Boundary: resolve_boundary(lga_name, state_name)
        Boundary->>Overpass: geocode_to_gdf() [Nominatim]
        Overpass-->>Boundary: candidate boundary geometry
        Boundary->>Boundary: hard checks (geometry, bbox, area, NEW: class/type)<br/>soft checks (display_name, NEW: osm_type, admin_level)
        Boundary-->>Pipeline: validated GeoDataFrame
        Pipeline->>Pipeline: emit stage_completed("boundary")

        Pipeline->>Layers: extract_layers(boundary, tag_config, strict, on_event=events)
        loop 6 layers, 2 concurrent, staggered — events from WORKER THREADS
            Layers->>Layers: emit stage_started("layer:X")
            Layers->>Overpass: features_from_polygon() per layer
            Overpass-->>Layers: raw features (or transient failure -> retry event -> retry)
            Layers->>Layers: emit stage_completed / stage_failed("layer:X")
        end
        Layers-->>Pipeline: dict of raw GeoDataFrames + warnings + STRUCTURED layer_status (new)

        Pipeline->>Pipeline: emit stage_started("cleaning")
        Pipeline->>Clean: clean_layers(raw_layers, boundary)
        Clean->>Clean: reproject, repair, dedupe,<br/>CORE+SEMANTIC+RAW_TAGS schema (new),<br/>collapse Polygon->Point (health_facilities, schools)
        Clean-->>Pipeline: cleaned GeoDataFrames
        Pipeline->>Pipeline: emit stage_completed("cleaning")

        Pipeline->>Pipeline: emit stage_started("export")
        Pipeline->>Export: export_layers(cleaned, output_dir)
        Export->>Export: write GeoJSON (full schema);<br/>Shapefile (CORE ONLY, new) split if mixed categories
        Export-->>Pipeline: exported paths dict + feature_count (new)
        Pipeline->>Pipeline: write boundary.geojson (new)
        Pipeline->>Pipeline: emit stage_completed("export")

        Pipeline->>Manifest: build_manifest(...) + write_manifest() (new)
        Manifest-->>Pipeline: manifest.json written

        Pipeline->>Log: log_run(..., layer_status)
        Log-->>Pipeline: run_log.json path

        Pipeline->>Pipeline: emit pipeline_completed(summary) (new)
        Pipeline-->>Thread: summary dict (or raises)
        Thread->>Thread: outcome["result"] = summary (or outcome["error"] = exc)
        deactivate Thread
    end

    par main thread polls while background thread runs
        Streamlit->>Streamlit: while thread.is_alive() or not events.empty():<br/>drain events, update stage_state per event type,<br/>re-render checklist rows + progress bar
    end
    Streamlit->>Streamlit: thread.join()
    alt outcome has error
        Streamlit->>User: st.error (BoundaryResolutionError-specific or generic)
    else success
        Streamlit->>Streamlit: cache[cache_key] = result
        Streamlit->>User: Extraction summary expander (new) + warnings + preview map
        Streamlit->>User: Download button (zip of GeoJSON + Shapefiles +<br/>boundary.geojson + manifest.json + run_log.json)
    end
```

## 1. Process start

`streamlit run app.py` starts a Streamlit server process. `app.py`
executes top-to-bottom **once** at startup (page config, CSS injection,
hero text, form definition, `_extraction_cache()`'s definition,
`LAYER_STYLES`'s definition) — none of this touches `lga_extractor` yet.
The form renders with `submitted = False`. Streamlit then waits for user
interaction, re-running the whole script from top to bottom on every
interaction, as before.

## 2. User submits the form

User types `"Akure North"` / `"Ondo"`, clicks **Extract OSM Data**.
Streamlit reruns the script; `submitted = True`, the `lga_name` non-blank
check passes.

## 3. Cache check

`app.py` checks its manual `_extraction_cache()` dict (an `st.cache_resource`
dict the app controls directly, replacing the previous revision's
`@st.cache_data`-decorated function — see [`app.md`](app.md) for why the
mechanism changed) for the key `("akure north", "ondo")`. **First run for
this LGA: cache miss.** No progress UI would be shown on a cache hit — this
walkthrough continues down the miss path, where the new live-progress
architecture actually runs.

## 4. `_run_extraction_with_live_progress()` starts

- `events.build_stage_order(DEFAULT_TAG_CONFIG)` builds the fixed stage
  checklist: `["boundary", "layer:roads", "layer:buildings",
  "layer:waterways", "layer:landuse", "layer:health_facilities",
  "layer:schools", "cleaning", "export"]`.
- `st.status(...)` renders the checklist, every row starting `"pending"`.
- A `ThreadSafeEventQueue()` instance (`events`) is created.
- A `threading.Thread` is started, targeting a worker function that calls
  `extract_lga(lga_name="Akure North", state_name="Ondo",
  on_event=events)` and stashes the result (or any caught exception) in a
  shared `outcome` dict.
- The main thread enters its polling loop: `while thread.is_alive() or not
  events.empty(): drain events, update UI`.

## 5. Inside `extract_lga()` (background thread) — stage 1: boundary

- `output_dir` defaults to `"output/akure_north"`; `tag_config` defaults
  to `DEFAULT_TAG_CONFIG`.
- **New:** emits `{"type": "stage_started", "stage": "boundary"}` via
  `events` — the main thread's polling loop picks this up on its next
  iteration and flips the "Resolving boundary" row to running.
- Calls `boundary.resolve_boundary(lga_name="Akure North",
  state_name="Ondo", manual_boundary_path=None)`.
  - OSM geocode branch: builds `"Akure North, Ondo, Nigeria"`, calls
    `ox.geocode_to_gdf(query)` — the pipeline's first network call.
  - `_validate_and_standardize()` runs: CRS standardized; hard checks
    (geometry validity, Nigeria bbox, plausible area via
    `clean.resolve_target_crs()`, and **new:** OSM `class`/`type` must be
    `"boundary"`/`"administrative"`); soft checks (`display_name`
    mention, and **new:** `osm_type` should be `"relation"`,
    `admin_level` should fall in a plausible range). Assuming Akure North
    resolves cleanly, no exception is raised.
  - Result now carries three **new** metadata columns —
    `osm_class="boundary"`, `osm_type_tag="administrative"`,
    `admin_level="6"` (Nigeria's conventional LGA admin level) — alongside
    the existing `boundary_source` and `validation_warnings=None`.
- **New:** emits `stage_completed` for `"boundary"`.

## 6. Stage 2: layer extraction — events now genuinely interleave across threads

- Calls `layers.extract_layers(boundary_gdf, tag_config=DEFAULT_TAG_CONFIG,
  strict=False, on_event=events)`.
  - A `ThreadPoolExecutor(max_workers=2)` opens. All six layers are
    submitted, staggered as before (`[0, 0, 3, 3, 6, 6]` seconds).
  - **New:** each `_extract_single_layer()` call, running inside its own
    worker thread, calls `_emit(on_event, {...})` directly — meaning
    `events` (the same `ThreadSafeEventQueue` instance) is now genuinely
    written to from up to two worker threads simultaneously, in addition
    to the main pipeline thread's own boundary/cleaning/export events.
    `ThreadSafeEventQueue`'s use of `queue.Queue` internally is exactly
    what makes this safe (see [`events.md`](modules/events.md)).
  - Each layer emits `stage_started`, zero or more `retry` events, and a
    terminal `stage_completed` (carrying `status: "success"` or
    `"success_empty"`) or `stage_failed` event.
  - As each future completes (`as_completed`, completion order — not
    submission order, so a UI's checklist rows for later-listed layers can
    genuinely flip to "done" before earlier-listed ones), the main
    *pipeline* thread (still on the background thread overall, distinct
    from `app.py`'s UI-polling main thread) collects results into
    `layer_status` (**new** — the structured dict) and `warnings`.
  - Assuming no genuine failures, all six layers succeed; some may be
    `"success_empty"`.
  - Returns `{"roads": gdf, ..., "_warnings": [...], "_status": {...}}`
    (**new key**).
- Meanwhile, on `app.py`'s **separate** UI-polling loop (main thread): each
  drained event updates exactly one row's `stage_state`/`stage_detail` and
  triggers a re-render of that row plus the overall progress bar — a user
  watching the page sees layer rows flip from `○` (pending) through `⟳`
  (running, possibly `⟳` again with a retry counter) to `✓` (done, with a
  feature-count detail) or `✗` (failed, with an error message), live,
  during this stage.

## 7. Stage 3: cleaning — richer schema, new event

- **New:** emits `stage_started` for `"cleaning"`.
- Calls `clean.clean_layers(raw_layers, boundary_gdf=boundary_gdf)`.
  - `resolve_target_crs(boundary_gdf)` resolves to `"EPSG:32631"`.
  - Each layer passes through `_clean_single_layer()`: null/empty geometry
    dropped, invalid geometry repaired, index flattened, `osmid`/`name`
    standardized, **new:** the complete original tag set captured as JSON
    into `RAW_TAGS_COLUMN` (`_row_tags_to_json()`), duplicates dropped,
    reprojected.
  - For `health_facilities`/`schools`, `_collapse_areas_to_points()` runs
    after reprojection — the same fix for the Akure North bug as before,
    unchanged in this revision.
  - **New:** columns are reduced to `CORE_COLUMNS` + this layer's present
    `SEMANTIC_COLUMNS` + `RAW_TAGS_COLUMN` — not just `osmid`/`name`/
    `geometry` as before. A road feature might now also carry `surface`,
    `maxspeed`, `oneway`; a health facility might carry `amenity`,
    `healthcare`, `beds`.
  - Returns six cleaned `GeoDataFrame`s, richer schema, all in
    `EPSG:32631`.
- `"_warnings"` **and** (new) `"_status"` are popped off the cleaned dict.
- **New:** emits `stage_completed` for `"cleaning"`.

## 8. Stage 4: export — two new artifacts written

- **New:** emits `stage_started` for `"export"` (this single event covers
  both the layer export and the new boundary-file write below).
- Calls `export.export_layers(cleaned, "output/akure_north")`.
  - Creates `output/akure_north/` and `.../shapefiles/`.
  - For each non-empty layer: writes `{layer_name}.geojson` with the
    **full** schema (core + semantic + raw tags); checks
    `_geom_categories_present()`; **new:** calls
    `_shapefile_safe_columns(gdf)` — reducing to `CORE_COLUMNS` only —
    before writing `.shp` file(s), so `roads.shp` (or its split
    `roads_point.shp`/`roads_line.shp` if mixed) never contains the new
    semantic columns or the JSON tag blob, only `osmid`/`name`/`geometry`.
  - Returns paths, plus **new**: `feature_count` per layer (post-cleaning
    count).
- **New:** immediately after, `boundary_gdf.to_file("output/akure_north/boundary.geojson",
  driver="GeoJSON")` — the boundary polygon itself is now cached to disk,
  closing the gap where it previously required a live re-query to obtain.
- **New:** emits `stage_completed` for `"export"`.

## 9. CRS resolved again, for logging and the manifest — unchanged mechanism, new consumer

- `resolve_target_crs(boundary_gdf)` is called a **second** time, as
  before — now the resolved value feeds both `log_run()` (as before) and
  (**new**) `build_manifest()`.

## 10. New: manifest construction

- `manifest.build_manifest(lga_name, state_name, resolved_crs,
  boundary_source, layer_status, exported, boundary_path)` reconciles
  `layer_status` (query-time outcome, from step 6) with `exported`
  (post-cleaning export outcome, from step 8) into one unified `layers`
  dict, keyed by layer name, with both `feature_count` (post-cleaning) and
  `feature_count_raw` (query-time) recorded separately per layer.
- `manifest.write_manifest(manifest, "output/akure_north")` writes
  `output/akure_north/manifest.json` — the formal, versioned contract
  `akure-accessibility-dashboard`'s
  [`data_contract.py`](../akure-accessibility-dashboard/modules/data_contract.md)
  will later read.
- Not treated as its own progress-UI stage (no dedicated `stage_started`/
  `stage_completed` pair) — near-instant bookkeeping, per the module's own
  framing.

## 11. Stage 5: logging

- Calls `logging_utils.log_run(..., layer_status)` — **new argument**: the
  same structured per-layer status now also lands in `run_log.json`'s
  `"layers"` key, not just in `manifest.json`.
- `_capture_environment()` runs as before.
- Writes `output/akure_north/run_log.json`.

## 12. `extract_lga()` returns — and emits its final event

- **New:** emits `{"type": "pipeline_completed", "summary": {...}}` —
  the complete summary dict, as the event payload itself, not just a
  "done" signal.
- The summary dict now includes, beyond the previous fields: `boundary_path`,
  `layer_status`, `manifest`, `manifest_path`.
- Returns to the worker function, which stores it in `outcome["result"]`.
  The background thread ends.

## 13. Back on the main thread: loop exit and result handling

- `app.py`'s polling loop's condition (`thread.is_alive() or not
  events.empty()`) becomes `False` once the thread has ended **and** the
  final event(s) have been drained — the `or not events.empty()` half of
  this condition is what prevents the `pipeline_completed` event from
  being missed in the narrow race where the thread finishes right after
  emitting it.
- `thread.join()`.
- If `outcome` contains `"error"`: the status box updates to an error
  state, and the exception is re-raised on the main thread — caught by
  `app.py`'s existing `try/except BoundaryResolutionError` / generic
  `Exception` block, exactly as if `extract_lga()` had been called
  directly and synchronously.
- Otherwise: the status box updates to complete, the result is cached in
  `_extraction_cache()`, and rendering proceeds.

## 14. Rendering the result

- **New:** an "Extraction summary" expander shows a per-layer
  feature-count table (from `exported[layer]["feature_count"]`) plus a
  caption of CRS/boundary source/warning count — this is new; previously
  a user had no dedicated UI element showing per-layer counts without
  opening the downloaded zip.
- Warnings expander, if any (unchanged).
- Preview map: `leafmap.foliumap.Map()`, layers added roads-first, the
  `zoomed_yet`-flag zoom logic and the malformed-file `try/except`
  backstop both completely unchanged from before this revision.
- A zip of `output/akure_north/` is built in memory — **now also
  including `boundary.geojson` and `manifest.json`**, since the zip walks
  the whole output directory and both new files live there — and offered
  via `st.download_button()`.

## Where State Lives, Summarized

| State | Where it lives | How long it persists |
|---|---|---|
| The resolved boundary polygon | In-memory during the run; **new:** also `output/{lga}/boundary.geojson` | Disk copy persists indefinitely |
| Raw/cleaned layer GeoDataFrames | In-memory only | Duration of one `extract_lga()` call |
| Structured per-layer status (`layer_status`) | In-memory during the run; persisted into both `run_log.json` and `manifest.json` (**new**, previously nowhere structured) | Disk copies persist indefinitely |
| Exported GeoJSON/Shapefile | Disk, under `output/{lga_name}/` | Persists indefinitely |
| `manifest.json` (**new**) | Disk, same directory | Persists indefinitely; the formal cross-repo contract file |
| `run_log.json` | Disk, same directory | Persists indefinitely |
| Progress events | `events.ThreadSafeEventQueue`, in-process (**new**) | Exist only for the duration of one extraction call; fully drained and discarded once the polling loop exits |
| The `extract_lga()` result summary dict | `app.py`'s manually-managed `st.cache_resource` dict (**changed mechanism**, same sharing semantics as before) | Persists for the life of the Streamlit server process, shared across all sessions/users requesting the same LGA/state |
| Form input values | Streamlit's per-session widget state | Current browser session only |
