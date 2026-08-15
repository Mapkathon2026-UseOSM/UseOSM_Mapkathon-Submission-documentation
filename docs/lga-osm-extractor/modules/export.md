# export.py

!!! info "Source"
    `lga_extractor/export.py` (176 lines — grew from 130 with the addition
    of Shapefile-safe column filtering, see below)

## Purpose

Writes cleaned layers (from [`clean.py`](clean.md)) to disk as GeoJSON and
Shapefile, organized under a per-LGA output directory. This is the module
that turns in-memory `GeoDataFrame`s into the actual deliverable files
consumed by ArcGIS Pro, QGIS, and — most importantly — by
`akure-accessibility-dashboard` downstream (see
[Cross-Repo Integration](../../cross-repo/integration.md)).

The module now has **two** real technical problems to solve, not one.
The original problem is a geometry-type mismatch: GeoJSON tolerates mixed
geometry types within a single file; Shapefile does not — most of the
splitting logic below exists to handle that. The new problem, introduced
by [`clean.py`](clean.md)'s richer per-layer attribute schema (semantic OSM
tags plus a full JSON `raw_tags` column), is an *attribute* mismatch:
Shapefile's DBF format truncates field names to 10 characters and has no
sensible representation for a JSON blob at all. See
`_shapefile_safe_columns()` below for how this module now handles that.

## Dependencies

- **Imports:** `os`, `geopandas`, and (new) `CORE_COLUMNS`, `RAW_TAGS_COLUMN`
  from [`clean.py`](clean.md).
- **Imported by:** `pipeline.py` (fourth stage, after `clean.clean_layers()`).

## Functions & Classes

### `_GEOM_CATEGORY` (module-level dict)

Maps each of the six Shapely geometry type names GeoPandas reports
(`Point`, `MultiPoint`, `LineString`, `MultiLineString`, `Polygon`,
`MultiPolygon`) down to one of three coarse categories: `"point"`,
`"line"`, `"polygon"`. This is the categorization used to decide whether a
layer needs to be split across multiple Shapefiles.

### `_geom_categories_present(gdf)`

| | |
|---|---|
| **What it does** | Returns the sorted list of distinct geometry *categories* (not raw geometry types) present in `gdf`. |
| **Why written this way** | Some OSM tag filters legitimately return more than one geometry type in a single query — the clearest example is the `"roads"` layer's `highway=*` filter, which matches both road ways (`LineString`) and point features like traffic signals and pedestrian crossings (`highway=traffic_signals`, `highway=crossing`). Similarly, the `"waterways"` layer's `waterway=*` + `natural=water` filter mixes `LineString` (rivers/streams) with `Polygon` (lakes/reservoirs). `_geom_categories_present()` exists purely to answer "does this layer have this problem," so `export_layers()` can decide whether it needs to split. |
| **Inputs** | `gdf: GeoDataFrame`. |
| **Outputs** | `list[str]`, sorted, e.g. `["line", "point"]` for a mixed roads layer, or `["polygon"]` for a single-category layer. |
| **Internal workflow** | `gdf.geometry.geom_type` gives the raw type string per row; `.map()` looks each up in `_GEOM_CATEGORY` (defaulting to `"other"` for any unmapped type — a defensive fallback, not expected to trigger for OSM data); `.unique().tolist()` collapses to distinct values; `sorted()` gives a deterministic order. |
| **Assumptions** | Assumes every geometry type actually encountered in practice is one of the six keys in `_GEOM_CATEGORY` — a `GeometryCollection` or other exotic type would fall into `"other"` and could produce unexpected Shapefile-splitting behavior, though this hasn't been observed with OSM-sourced data. |
| **Complexity** | O(N) in feature count — one pass over the geometry type column. |
| **Concurrency / race conditions** | None — pure function. |
| **Covered by test(s)** | See [tests.md](../tests.md) — exercised indirectly through `test_export_layers_splits_mixed_geometry_types`, which depends on this function correctly identifying the point/line mix that triggers the Shapefile-splitting behavior. |

## New in this revision: `_shapefile_safe_columns(gdf)` (private)

**What it does:** reduces a cleaned layer to `CORE_COLUMNS` only (`osmid`, `name`, `geometry`) — discarding whatever `SEMANTIC_COLUMNS` and `RAW_TAGS_COLUMN` [`clean.py`](clean.md) added — for Shapefile export specifically.

**Why this is necessary, not just a stylistic choice:** since `clean.clean_layers()` started preserving richer per-layer semantic OSM attributes (e.g. `building:levels`, `building:use`) plus a full `raw_tags` JSON blob, writing that full schema straight to Shapefile would be **actively harmful, not just imprecise**. Shapefile field names are silently truncated to 10 characters — `"building:levels"` and `"building:use"` could both collide/mangle down to something like `"building:l"` — and a JSON blob has no meaningful place in a fixed-width DBF field at all (it would either be truncated mid-string, corrupting the JSON, or rejected by the writer depending on the driver). GeoJSON has no such limit and is where the full schema belongs; Shapefile output stays deliberately at the original minimal schema and remains **fully backward compatible** with every pre-existing Shapefile consumer of this extractor's output — a consumer that only ever read `osmid`/`name`/`geometry` from Shapefile sees no change in behavior at all.

**Implementation:** `keep = [c for c in CORE_COLUMNS if c in gdf.columns]; return gdf[keep]` — a straightforward column-list filter, applied at both Shapefile write sites (the single-file path and the per-category split path) via `_shapefile_safe_columns(gdf).to_file(...)` / `_shapefile_safe_columns(subset).to_file(...)`, rather than passed as a parameter — GeoJSON's `gdf.to_file(geojson_path, driver="GeoJSON")` call is unaffected, still writing the complete schema.

### `export_layers(layers_dict, output_dir)`

| | |
|---|---|
| **What it does** | Writes every non-empty layer in `layers_dict` to both a `.geojson` file (full schema, always one file per layer, mixed types are fine) and one or more `.shp` files (**core schema only, new** — split by geometry category only when the layer actually mixes categories). Returns a summary dict of what was written, skipped, and split. |
| **Why written this way** | The design deliberately keeps GeoJSON export simple and uniform (one file per layer, no splitting logic needed) while only paying the complexity cost of splitting where Shapefile's format constraint actually forces it. This backward-compatible shape (single path string for the simple case, dict of `{category: path}` only when needed) means existing callers that only handle single-category layers don't break. |
| **Inputs** | `layers_dict: dict` (layer_name → cleaned `GeoDataFrame`, as returned by `clean.clean_layers()`; `"_warnings"` **and, new, `"_status"`** keys, if present, are explicitly skipped during export — neither is a layer); `output_dir: str` (created if it doesn't exist; a `shapefiles/` subfolder is created within it). |
| **Outputs** | `dict` mapping each exported `layer_name` to `{"geojson": path, "shapefile": shapefile_value, "feature_count": int}` — **`feature_count` is new**: the number of features actually written (post-cleaning), distinct from the pre-cleaning `feature_count` recorded in `layers.extract_layers()`'s `"_status"` (see [`layers.md`](layers.md)) — `pipeline.extract_lga()` reconciles both into the single [extraction manifest](manifest.md). `shapefile_value` is a single path string for single-category layers, or a `{category: path}` dict for split layers. Also includes `"_skipped"` and, only if any layer needed splitting, `"_split_layers"`. |
| **Internal workflow** | 1. Create `output_dir` and `output_dir/shapefiles/`.<br>2. For each `(layer_name, gdf)` in `layers_dict`:<br>&nbsp;&nbsp;a. Skip `"_warnings"` and `"_status"` (new — previously only `"_warnings"`).<br>&nbsp;&nbsp;b. If `gdf` is `None` or empty, record it in `skipped` and continue.<br>&nbsp;&nbsp;c. Write the full-schema GeoJSON unconditionally.<br>&nbsp;&nbsp;d. Compute `categories = _geom_categories_present(gdf)`.<br>&nbsp;&nbsp;e. If `len(categories) <= 1`: write one Shapefile via `_shapefile_safe_columns(gdf).to_file(...)` (**new** — core-columns-only), record a single path plus `feature_count: len(gdf)` (**new**).<br>&nbsp;&nbsp;f. Otherwise: for each category present, filter `gdf`, write `_shapefile_safe_columns(subset).to_file(...)` (**new**) per category, record paths plus `feature_count: len(gdf)` (**new** — the *layer's* total count, not per-category).<br>3. Attach `"_skipped"` and (if non-empty) `"_split_layers"`, return. |
| **Assumptions** | Assumes `gdf.to_file(..., driver="GeoJSON")` and `driver="ESRI Shapefile"` handle CRS metadata correctly on their own. **Updated:** previously assumed Shapefile's 10-character field truncation "wasn't a practical problem" given the old minimal `KEEP_COLUMNS` schema — this assumption is now handled explicitly via `_shapefile_safe_columns()` rather than relying on the schema staying small by construction, since it no longer does. |
| **Complexity** | O(L × N) where L = number of layers, N = features per layer, for the write operations; the geometry-category filtering step is an additional O(N) pass per category for split layers. The new `_shapefile_safe_columns()` filter is O(1) in column count, negligible. |
| **Concurrency / race conditions** | None — sequential loop, no threading. |
| **Covered by test(s)** | See [tests.md](../tests.md) — `test_export_layers_writes_geojson_and_shapefile`, `test_export_layers_splits_mixed_geometry_types`, plus new: `test_export_layers_shapefile_stays_core_columns_only`. |

## Internal Workflow

```mermaid
flowchart TD
    A["export_layers(layers_dict, output_dir)"] --> B["makedirs(output_dir), makedirs(output_dir/shapefiles)"]
    B --> C["for layer_name, gdf in layers_dict:"]
    C --> D{"layer_name in ('_warnings', '_status')?"}
    D -- yes --> C
    D -- no --> E{gdf empty or None?}
    E -- yes --> F["append to skipped list"] --> C
    E -- no --> G["write {layer_name}.geojson — FULL schema<br/>(core + semantic + raw_tags)"]
    G --> H["categories = _geom_categories_present(gdf)"]
    H --> I{len(categories) <= 1?}
    I -- yes --> J["_shapefile_safe_columns(gdf) — CORE ONLY<br/>write single {layer_name}.shp"]
    J --> K["exported[layer_name] = {geojson, shapefile: path_string, feature_count}"]
    I -- no --> L["for each category: subset by geom_type,<br/>_shapefile_safe_columns(subset) — CORE ONLY<br/>write {layer_name}_{category}.shp"]
    L --> M["exported[layer_name] = {geojson, shapefile: {category: path, ...}, feature_count}"]
    M --> N["record in split_layers dict"]
    K --> C
    N --> C
    C --> O["exported['_skipped'] = skipped"]
    O --> P{any split_layers?}
    P -- yes --> Q["exported['_split_layers'] = split_layers"]
    P -- no --> R["return exported"]
    Q --> R
```

## Gotchas

- **GeoJSON and Shapefile now carry genuinely different attribute schemas
  for the same layer, not just different file formats.** A consumer that
  reads a layer's GeoJSON and expects the same columns to be present in the
  corresponding Shapefile (or vice versa) will be surprised — GeoJSON has
  `CORE_COLUMNS` + `SEMANTIC_COLUMNS` + `RAW_TAGS_COLUMN`; Shapefile has
  `CORE_COLUMNS` only. This is new, deliberate, and documented, but a real
  behavior change for any tooling built assuming schema parity between the
  two formats.
- **A layer's Shapefile output shape depends on its data, not on any
  configuration.** Whether `export_layers()`'s returned `shapefile_value`
  for a given layer is a plain string or a `{category: path}` dict is
  determined entirely by what geometry types happen to be present in that
  particular LGA's extracted data. Any code consuming this return value
  needs to check the type, not assume a fixed shape per layer name.
- **`_geom_categories_present()`'s `"other"` fallback is silent.** A
  geometry type outside the six mapped ones would be grouped as `"other"`
  with no warning.
- **Empty-subset skip inside the split loop is a real (if rare) edge case.**
  The `if subset.empty: continue` guard inside the per-category split path
  handles a category appearing in `categories` but then producing zero rows
  when filtered — shouldn't normally happen, but defensive.
