# export.py

!!! info "Source"
    `lga_extractor/export.py` (130 lines)

## Purpose

Writes cleaned layers (from [`clean.py`](clean.md)) to disk as GeoJSON and
Shapefile, organized under a per-LGA output directory. This is the module
that turns in-memory `GeoDataFrame`s into the actual deliverable files
consumed by ArcGIS Pro, Google Earth Pro, and — most importantly — by
`akure-accessibility-dashboard` downstream (see
[Cross-Repo Integration](../../cross-repo/integration.md)).

The module's one real technical problem is a format mismatch: GeoJSON
tolerates mixed geometry types within a single file; Shapefile does not.
Most of this file's logic exists to handle that mismatch correctly rather
than simply failing on it.

## Dependencies

- **Imports:** `os`, `geopandas`.
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
| **Covered by test(s)** | See [tests.md](../tests.md). |

### `export_layers(layers_dict, output_dir)`

| | |
|---|---|
| **What it does** | Writes every non-empty layer in `layers_dict` to both a `.geojson` file (always one file per layer, mixed types are fine) and one or more `.shp` files (split by geometry category only when the layer actually mixes categories). Returns a summary dict of what was written, skipped, and split. |
| **Why written this way** | The design deliberately keeps GeoJSON export simple and uniform (one file per layer, no splitting logic needed, since GeoJSON has no restriction here) while only paying the complexity cost of splitting where Shapefile's format constraint actually forces it — layers with a single geometry category (the common case: `buildings`, `health_facilities`, `landuse`) still export as one plain `{layer_name}.shp`, completely unchanged from a version of this function that never had to think about mixed types at all. This backward-compatible shape (single path string for the simple case, dict of `{category: path}` only when needed) means existing callers that only handle single-category layers don't break. |
| **Inputs** | `layers_dict: dict` (layer_name → cleaned `GeoDataFrame`, as returned by `clean.clean_layers()`; a `"_warnings"` key, if present, is explicitly skipped during export — it isn't a layer); `output_dir: str` (created if it doesn't exist; a `shapefiles/` subfolder is created within it to keep Shapefile's multi-file sidecar outputs — `.shp`/`.shx`/`.dbf`/`.prj` — organized separately from the flat GeoJSON files). |
| **Outputs** | `dict` mapping each exported `layer_name` to `{"geojson": path, "shapefile": shapefile_value}`, where `shapefile_value` is a single path string for single-category layers, or a `{category: path}` dict for split layers. Also includes `"_skipped"` (list of layer names skipped because they were empty) and, only if any layer needed splitting, `"_split_layers"` (dict of `{layer_name: categories_list}`, useful for logging/visibility into which layers triggered the split path). |
| **Internal workflow** | 1. Create `output_dir` and `output_dir/shapefiles/` (both `exist_ok=True`, safe to call repeatedly).<br>2. For each `(layer_name, gdf)` in `layers_dict`:<br>&nbsp;&nbsp;a. Skip `"_warnings"` entirely (not a real layer).<br>&nbsp;&nbsp;b. If `gdf` is `None` or empty, record it in `skipped` and continue — no files written for empty layers.<br>&nbsp;&nbsp;c. Write the GeoJSON unconditionally: `gdf.to_file(path, driver="GeoJSON")`.<br>&nbsp;&nbsp;d. Compute `categories = _geom_categories_present(gdf)`.<br>&nbsp;&nbsp;e. If `len(categories) <= 1`: write one Shapefile as before, record a single path.<br>&nbsp;&nbsp;f. Otherwise: for each category present, filter `gdf` down to just the geometry types belonging to that category (via `.isin()` against the subset of `_GEOM_CATEGORY` keys mapping to that category), skip if the filtered subset happens to be empty, write `{layer_name}_{category}.shp`, and record the path under that category key. Also record this layer in `split_layers` for the summary.<br>3. Attach `"_skipped"` and (if non-empty) `"_split_layers"` to the result dict, return it. |
| **Assumptions** | Assumes `gdf.to_file(..., driver="GeoJSON")` and `driver="ESRI Shapefile"` (both via GeoPandas → Fiona/pyogrio) handle CRS metadata correctly on their own — this function does no explicit CRS handling itself, relying entirely on whatever CRS `clean_layers()` already set. Assumes Shapefile's 10-character field name truncation and other format limitations (a well-known Shapefile constraint, unrelated to the geometry-type issue this module solves) aren't a practical problem given `clean.py`'s minimal `KEEP_COLUMNS` schema (`osmid`, `name`, `geometry` — none close to the 10-character limit). |
| **Complexity** | O(L × N) where L = number of layers, N = features per layer, for the write operations themselves; the geometry-category filtering step is an additional O(N) pass per category for split layers. |
| **Concurrency / race conditions** | None — sequential loop, no threading. Writing to `output_dir` is not guarded against concurrent calls to `export_layers()` targeting the *same* `output_dir` from multiple processes — this isn't a concern in the current pipeline (each LGA extraction run uses its own output directory sequentially), but would be worth locking if this function were ever called concurrently for the same target directory. |
| **Covered by test(s)** | See [tests.md](../tests.md). |

## Gotchas

- **A layer's Shapefile output shape depends on its data, not on any
  configuration.** Whether `export_layers()`'s returned `shapefile_value`
  for a given layer is a plain string or a `{category: path}` dict is
  determined entirely by what geometry types happen to be present in that
  particular LGA's extracted data — the same layer name (e.g. `"roads"`)
  could return a single path for one LGA (no point-type road features
  present) and a split dict for another. Any code consuming this return
  value needs to check the type, not assume a fixed shape per layer name.
- **`_geom_categories_present()`'s `"other"` fallback is silent.** A
  geometry type outside the six mapped ones would be grouped as `"other"`
  with no warning — worth being aware of if OSM ever returns an unexpected
  geometry type (e.g. a `GeometryCollection`) for some tag combination.
- **Empty-subset skip inside the split loop is a real (if rare) edge case.**
  Step 2f's `if subset.empty: continue` guards against a category appearing
  in `categories` (from the unique-values pass) but then producing zero rows
  when filtered — this shouldn't normally happen given how `categories` is
  derived, but the check is there defensively.
