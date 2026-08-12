# static_maps.py

!!! info "Source"
    `akure_access/visualization/static_maps.py` (670 lines — the largest
    single module in this repository)

## Purpose

Generates publication-styled static maps and charts (JPEG, at both print
and web resolution) from the project's scored grid: categorical
access-deficit maps, continuous travel-time choropleths, OSM-completeness
maps, and a mode-comparison bar chart — each carrying standard reference-map
cartographic furniture (OSM basemap, lat/lon gridlines, north arrow, scale
bar, legend) and a data-driven caption from
[`insights.py`](../insights.md).

## Dependencies

- **Imports:** `json`, `os`, `warnings`, `geopandas`,
  `matplotlib.patches`/`matplotlib.pyplot`, `numpy`,
  `matplotlib.lines.Line2D`; `describe_deficit_map`,
  `describe_continuous_map`, `describe_completeness_map`,
  `describe_mode_comparison_chart` from `akure_access.insights` (imported
  under underscore-prefixed aliases, e.g. `_describe_deficit_map`, to
  distinguish the imported caption function from this module's own
  `plot_*` functions of similar purpose without a naming collision).
  Optionally imports `contextily` for the OSM basemap — see below for why
  this import is wrapped in a `try/except`.
- **Imported by:** the project's analysis notebook (for producing the
  full static export set); not imported by `dashboard/app.py` directly —
  the live dashboard uses its own lighter-weight rendering path, this
  module is specifically for the downloadable/publication artifact set.

## Why This Module Exists, and Why It's Vector-Based, Not Raster

This reasoning is stated directly in the module's own top-of-file
docstring and is worth preserving here in full, since it explains a design
decision that shapes every function in the file: standard reference-map
styling (OSM basemap, gridlines, north arrow, scale bar, legend) is
normally built for single/multi-band *rasters* — satellite imagery,
`rasterio` plus a GeoTIFF per layer. This project's accessibility outputs
are *vector* data instead — a polygon grid with per-cell scores, plus
point/line facility and road layers — so this module ports the
cartographic *styling conventions* (gridlines, north arrow, scale bar, OSM
basemap, legend/colorbar placement, dpi/format choices) to vector plotting
via `gdf.plot()` (GeoPandas' matplotlib-based vector plotting), rather than
`ax.imshow()`.

Every plotting function accepts a `GeoDataFrame` in EPSG:4326 (lat/lon) and
reprojects internally to Web Mercator only for the OSM basemap layer, via
`contextily.add_basemap(..., crs='EPSG:4326')` — reprojecting the *tiles*
on the fly rather than reprojecting the data itself. This keeps every
function's public interface in the same CRS the rest of `akure_access`
already standardizes on (well, mostly — see the `EPSG:4326`-vs-`EPSG:32631`
tension addressed by `_ensure_lonlat()` below).

## Module-Level Constants

| Constant | Purpose |
|---|---|
| `DEFICIT_PALETTES` | `{"standard": [green, amber, red], "colorblind_safe": [blue, orange, vermillion]}` — explicitly documented as reusing **the exact same palette** as the Streamlit dashboard's discrete legend, so a downloaded static JPEG and the live interactive map never disagree on what a given color means. |
| `DEFICIT_LABELS` | `["Well served", "Underserved (1 service)", "Underserved (both services)"]` — the three category labels matching `DEFICIT_PALETTES`' three colors. |
| `ACCENT_PRIMARY`/`ACCENT_SECONDARY`/`ACCENT_HIGHLIGHT` | Brand accent colors, documented as being reused from the dashboard's own theme, "so charts/maps produced by the notebook visually match the Streamlit app they'll sit alongside, rather than looking like a separate, disconnected artifact." |

## Functions & Classes — Cartographic Building Blocks

### `_ensure_lonlat(gdf)`

| | |
|---|---|
| **What it does** | Reprojects `gdf` to EPSG:4326 if it isn't already there (or has no CRS set, in which case it's assumed to already be EPSG:4326 and just tagged as such, with a warning). |
| **Why written this way — a real, previously-hit failure this function exists to prevent.** | The module's docstring is explicit: gridline spacing, the scale bar's km-per-degree math, and the OSM basemap's `crs=` argument all assume geographic coordinates. But `akure_access.accessibility` deliberately reprojects everything to a metric UTM CRS (EPSG:32631) for correct distance/area math — meaning the real `grid_access_scored.geojson` this module actually receives in practice is in UTM meters, not lon/lat degrees. Without this conversion, gridline/scale-bar spacing code that assumes degree-scale coordinates would instead be handed meter-scale UTM values — attempting to draw gridlines at fractional-degree intervals across a value range spanning tens of thousands (UTM meters, not degrees) would produce tens of thousands of gridlines and hang the plotting call. The docstring states this is exactly the failure caught while testing this module against real Akure North data — not a hypothetical concern. |
| **Inputs / Outputs** | `gdf: GeoDataFrame` in, reprojected (or CRS-tagged) `GeoDataFrame` out. |
| **Internal workflow** | 1. If `gdf.crs` is `None`: warn, then `set_crs("EPSG:4326")` (tags without reprojecting — assumes the untagged data is already lon/lat).<br>2. If `gdf.crs` is set but isn't `EPSG:4326`/`OGC:CRS84`: `to_crs("EPSG:4326")` (actually reprojects).<br>3. Otherwise, return unchanged. |
| **Assumptions** | The no-CRS branch assumes untagged input is already geographic — a reasonable default, but if an untagged UTM-coordinate `GeoDataFrame` were ever passed in, this function would tag it as EPSG:4326 without reprojecting, silently producing wildly wrong coordinates rather than the hang this function is designed to prevent for the *tagged*-but-wrong-CRS case. |
| **Complexity** | O(N) — a reprojection pass over N features, when reprojection is actually needed; O(1) otherwise. |
| **Covered by test(s)** | See [tests.md](../../tests.md) — `test_static_maps.py`. |

### `add_north_arrow(ax, xy=(0.94, 0.94))`

A simple "N" arrow annotation in the upper-right of the axes (position
configurable, defaulting to a consistent placement used throughout the
module). Pure matplotlib annotation logic — no geospatial computation.

### `add_scale_bar(ax, bounds_lonlat, length_km=None, xy=(0.05, 0.05))`

| | |
|---|---|
| **What it does** | Draws a ground-truth-length scale bar in the lower-left of the map, auto-sizing to a "nice" round number if `length_km` isn't specified. |
| **Why written this way** | Two things worth explaining. **(1) Why length can't just be "N pixels":** since the axes are in geographic degrees, a fixed pixel/axis-fraction length would represent a *different real-world distance* depending on latitude — the physical length of one degree of longitude shrinks toward the poles (`cos(latitude)` scaling), so the function explicitly converts a target kilometer length into the correct number of degrees at the map's actual mid-latitude, using the standard 111.32 km/degree-of-latitude constant. The docstring is upfront this is a spherical approximation (not ellipsoid-accurate) — acceptable at map-reading scale, not survey-grade. **(2) The "nice round number" auto-sizing:** mirrors how QGIS/ArcGIS auto-scale bars behave — targets roughly a fifth of the map's width, then snaps to the nearest of `{1, 2, 5, 10, 20, 50, 100...} × 10^n`, so an LGA-scale map and a wider comparison view each get a sensibly-sized bar rather than sharing one hardcoded length that would look wrong at one scale or the other. |
| **Inputs** | `ax` (matplotlib axes); `bounds_lonlat: tuple` (west, south, east, north); `length_km`, optional; `xy: tuple`, default `(0.05, 0.05)` (axes-fraction position). |
| **Outputs** | `None` — draws directly on `ax`. |
| **Internal workflow** | 1. Compute `lat_mid`, then `km_per_deg_lon = 111.32 * cos(radians(lat_mid))`.<br>2. Compute the map's actual width in km from its lon/lat extent.<br>3. If `length_km` not given: target = width/5; find the magnitude via `10 ** floor(log10(target))`; loop through candidate multipliers `(1, 2, 5, 10)` of that magnitude, picking the first that's `>= target` (a `for...else` construction — the `else` branch, reached only if no candidate satisfied the condition, falls back to `10 * magnitude`, guaranteeing `length_km` is always set by the end of this block).<br>4. Convert `length_km` back to degrees (`bar_deg`) using the same `km_per_deg_lon` conversion.<br>5. Compute bar start position from `xy` (fraction of the map's bounds), draw a horizontal line of that length, and label it with the km value. |
| **Assumptions** | Assumes the spherical-Earth approximation is acceptable at the LGA scale this project operates at — explicitly not survey-grade, as the docstring states. |
| **Complexity** | O(1) — fixed number of arithmetic operations and one drawing call, independent of data size. |
| **Covered by test(s)** | See [tests.md](../../tests.md). |

### `add_gridlines(ax, bounds_lonlat, interval=None, color="gray", alpha=0.6, fontsize=9, label_sides=("left", "bottom"))`

| | |
|---|---|
| **What it does** | Draws a lat/long graticule (grid of reference lines) with edge coordinate labels, styled after standard printed reference maps. |
| **Why written this way** | Two deliberate cartographic conventions, both explained in the docstring. **(1) Single-side labeling**: coordinate labels appear on the left (latitude) and bottom (longitude) only, not mirrored on all four sides — the docstring is explicit that mirrored labeling (showing the same values on opposite sides) is "a plotting-library default, not a cartographic convention"; standard topographic/reference maps label each axis once. The implementation still allows *tick marks* (short lines, not text) on all sides for a finished "boxed" frame look, only the text labels themselves are restricted to `label_sides`. **(2) Auto-picked interval scaled to map extent**: an LGA is only a few km wide, so a continental-map-scale interval like 1.0° would draw zero or one gridline across the whole map and be useless — the function instead picks from a candidate list `(0.005, 0.01, 0.02, 0.05, 0.1, 0.2, 0.5, 1.0)`, choosing the finest interval that still produces no more than 8 gridlines across the map's larger dimension (`span / candidate <= 8`), so grid density scales sensibly whether plotting a single ward, a full LGA, or a wider multi-LGA comparison view. |
| **Inputs** | `ax`; `bounds_lonlat: tuple`; `interval`, optional (degrees); styling kwargs; `label_sides: tuple`, default `("left", "bottom")`. |
| **Outputs** | `None` — draws directly on `ax`. |
| **Internal workflow** | 1. Auto-pick `interval` if not given, per the candidate-list logic above.<br>2. Build `lons`/`lats` arrays via `np.arange()`, snapped to interval boundaries starting below `west`/`south` (via `np.floor(west/interval) * interval`) so gridlines align to round coordinate values rather than starting at the map's arbitrary edge.<br>3. Draw a vertical line (`axvline`) per longitude, horizontal line (`axhline`) per latitude.<br>4. Format tick labels with degree symbols and hemisphere letters (`_fmt_lon`/`_fmt_lat` closures) — e.g. `"5.234°E"`, `"7.123°N"`.<br>5. Set tick positions/labels, then call `ax.tick_params()` with per-side boolean flags derived from `label_sides`, controlling label visibility independently per side while ticks themselves remain visible on top/bottom and left/right regardless. |
| **Assumptions** | Assumes an 8-gridline cap across the larger dimension is a reasonable density — a readability judgment, not derived from any formal cartographic standard. |
| **Complexity** | O(G) where G = number of gridlines drawn (bounded by the 8-per-axis cap, so effectively a small constant regardless of map size). |
| **Covered by test(s)** | See [tests.md](../../tests.md). |

### `add_osm_basemap(ax, crs="EPSG:4326", timeout=8)`

| | |
|---|---|
| **What it does** | Adds OpenStreetMap tiles behind the plotted data (`zorder=0`), reprojecting tiles on the fly via `contextily` to match `crs`, rather than reprojecting the data into Web Mercator first. |
| **Why written this way — graceful degradation is the central design goal here, not basemap-fetching itself.** | Two independent failure modes are handled, both falling back to the same plain light-gray background with an explanatory note rather than raising: **(1) `contextily` not installed at all** — checked once at module import time (`_HAS_CONTEXTILY`), avoiding a hard dependency on what the module docstring calls an "optional/heavy" package; **(2) tiles can't be fetched** (no internet access, timeout, tile-server error) — caught via a broad `except Exception`, with the warning message explicitly noting this is expected in offline/sandboxed environments, and that Colab/Streamlit Cloud (both having internet access) would normally succeed. The `timeout=8` parameter exists specifically so a blocked/unreachable tile server fails fast rather than hanging the whole map-generation run indefinitely, since `contextily`'s own default behavior is to retry without a bounded timeout. This graceful-degradation design is what makes every other function in this module safe to call from automated/CI/testing contexts without needing live network access or a basemap-mocking setup — the rest of the map (data, gridlines, legend, scale bar) still renders correctly either way. |
| **Inputs** | `ax`; `crs: str`, default `"EPSG:4326"`; `timeout: int`, default `8` seconds. |
| **Outputs** | `None` — side effect on `ax` (either real tiles, or a gray background + note). |
| **Internal workflow** | 1. If `contextily` isn't available: set a light-gray facecolor, add an explanatory centered text annotation, return.<br>2. Otherwise, try `ctx.add_basemap(ax, crs=crs, source=ctx.providers.OpenStreetMap.Mapnik, ..., timeout=timeout)`.<br>3. On any exception: warn with a message including the actual exception and the offline-context explanation, fall back to the same gray facecolor (though notably, without the explanatory text annotation the no-`contextily` branch adds — see Gotchas). |
| **Assumptions** | Assumes a broad `except Exception` is the right granularity for "basemap fetch failed for any reason" — deliberately not distinguishing network timeout from a `contextily`-internal error from a malformed CRS, since the user-facing remedy (proceed without a basemap) is the same regardless of the specific cause. |
| **Complexity** | Dominated by the network I/O of tile fetching when it succeeds; O(1) in this function's own logic for either fallback path. |
| **Concurrency / race conditions** | None introduced by this function; tile-fetch network calls are synchronous, not parallelized. |
| **Covered by test(s)** | See [tests.md](../../tests.md) — the two distinct fallback paths (no `contextily` vs. `contextily` present but fetch fails) are each worth direct coverage given they're marked `# pragma: no cover` in the source for the network-dependent branch, meaning that specific path likely isn't exercised in the existing automated test suite — see [tests.md](../../tests.md) for what's actually covered. |

### `_figure_bounds(gdf, pad_frac=0.04)`

A small helper: returns `gdf.total_bounds` (lon/lat) padded outward by
`pad_frac` (default 4%) on each side, "so the data doesn't touch the frame
edge, matching standard reference-map use of the raster's own bounds as
the map extent."

### `_finalize_and_save(fig, ax, bounds, title, out_path, scale_bar_km, dpi, web_path=None, web_dpi=None)`

| | |
|---|---|
| **What it does** | The shared finishing sequence for every map function: adds gridlines/north arrow/scale bar, sets title/axis labels/extent, then saves the figure once at `dpi` and optionally a second time at `web_dpi` to a different path. |
| **Why written this way — the dual-save mechanism is a deliberate, documented performance optimization, not incidental convenience.** | The docstring is explicit: the second save is "nearly free" — by the time this function runs, the expensive parts (fetching OSM basemap tiles, plotting the data layer, computing gridlines) have already happened exactly once; a second `fig.savefig()` call at a different dpi simply re-rasterizes the same already-rendered, already-in-memory figure — it does **not** re-fetch tiles or re-run the plot. This is what makes generating a "print-quality download" + "fast web display" pair practical to produce together in one pass, rather than needing to build the whole figure twice, which would also double the load placed on OSM's tile servers for no visual benefit — directly connecting this function's design back to `add_osm_basemap()`'s network-fetch concern above. |
| **Inputs** | `fig`, `ax` (matplotlib objects); `bounds`; `title: str`; `out_path: str`; `scale_bar_km`, optional; `dpi: int`; `web_path`, `web_dpi`, optional. |
| **Outputs** | `str` — returns `out_path` (the print-tier path) regardless of whether a web-tier copy was also saved. |
| **Internal workflow** | 1. Call `add_gridlines()`, `add_north_arrow()`, `add_scale_bar()` in sequence against the given `bounds`.<br>2. Set title, axis labels, and explicit x/y limits from `bounds` (locking the final displayed extent).<br>3. Ensure `out_path`'s parent directory exists; save at `dpi`.<br>4. If `web_path` given: ensure its parent directory exists too; save again at `web_dpi` (falling back to `dpi` if `web_dpi` wasn't specified) — the "nearly free" second save described above.<br>5. `plt.close(fig)` — releases the figure's memory; important given this function may be called many times in a row inside `generate_all_static_outputs()`'s loop, where leaving figures open would accumulate memory across dozens of maps in one run.<br>6. Return `out_path`. |
| **Complexity** | O(1) in this function's own logic (fixed number of drawing/saving calls); the actual cost is dominated by matplotlib's rasterization, proportional to figure size/dpi, done once (or twice, cheaply, per the dual-save reasoning) per call. |
| **Covered by test(s)** | See [tests.md](../../tests.md). |

### `_map_title(lga_name, metric_label)`

One line: `f"{lga_name}: {metric_label}"` — the single shared title
convention every map/chart type uses (e.g. `"Akure South: Access Deficit
(Walk)"`), kept as one function specifically so every figure type uses the
exact same formatting rather than each caller phrasing titles slightly
differently.

## Functions & Classes — Map Builders

### `plot_deficit_map(grid_gdf, mode, title, out_path, settled_only=True, palette="standard", scale_bar_km=None, dpi=300, web_path=None, web_dpi=None, figsize=(11, 12))`

| | |
|---|---|
| **What it does** | The categorical (0/1/2) access-deficit map for one mode, styled to match the reference cartographic conventions (basemap, gridlines, north arrow, scale bar, legend), using the same palette as the interactive dashboard. |
| **Why written this way** | `settled_only=True` (the default) excludes unsettled cells from the plotted layer **entirely**, rather than coloring them a fourth "unscored" color — the docstring's reasoning: "no people, no score" is a conceptually different situation from "well served," and coloring unsettled land as if it were a positive finding (or even a neutral fourth category sitting visually alongside the other three) would misrepresent empty land as meaningful analysis output. The map's overall extent (`bounds`) is still computed from the *full* grid (not just settled cells) "for context" — so the map frame shows the whole LGA even though only settled portions are actually colored, giving geographic context without implying unsettled areas were scored. |
| **Inputs** | `grid_gdf: GeoDataFrame`; `mode: str`; `title`, `out_path: str`; `settled_only: bool`, default `True`; `palette: str`, default `"standard"`; `scale_bar_km`, `dpi`, `web_path`, `web_dpi`, `figsize` — the last five all passed straight through to `_finalize_and_save()`. |
| **Outputs** | `str` (the saved print-tier path, from `_finalize_and_save()`). |
| **Internal workflow** | 1. Resolve the mode-specific deficit-score column name; raise `KeyError` with an explicit, actionable message (naming the expected column and its source function) if it doesn't exist — a clearer failure than a generic pandas `KeyError` would produce.<br>2. `_ensure_lonlat()` the input.<br>3. Filter to settled cells if `settled_only`; raise `ValueError` if the result is empty (nothing to plot).<br>4. Compute the full-grid `bounds` (before the settled filter, per the "for context" reasoning above).<br>5. Create the figure, add the OSM basemap.<br>6. For each of the three score values (0, 1, 2), filter to matching rows and plot them in that category's color — plotted as three separate `.plot()` calls rather than one call with a categorical colormap, giving explicit control over each category's exact color and allowing empty categories (a mode/LGA combination with, say, zero cells scoring `2`) to be skipped cleanly without an empty/broken legend entry.<br>7. Build legend patches matching the three colors/labels, add as a titled legend.<br>8. Delegate to `_finalize_and_save()` for the shared finishing sequence. |
| **Assumptions** | Assumes the three-category palette adequately represents the deficit score's full 0–2 range — true given the scoring model in `scoring.add_access_deficit_score()` only ever produces exactly these three integer values (plus `NaN` for unsettled cells, which are excluded here rather than plotted). |
| **Complexity** | O(N) for the settled filter and per-category subsetting (N = grid cell count); plotting cost itself scales with the number of settled cells drawn. |
| **Covered by test(s)** | See [tests.md](../../tests.md). |

### `plot_continuous_map(grid_gdf, value_col, title, out_path, colorbar_label, settled_only=True, cmap="turbo", vmin=None, vmax=None, alpha=0.9, scale_bar_km=None, dpi=300, web_path=None, web_dpi=None, figsize=(11, 12))`

| | |
|---|---|
| **What it does** | The continuous choropleth map (e.g. `health_time_min_walk`) with a colorbar, for any single numeric column. |
| **Why written this way** | `NaN` cells (unsettled, or genuinely unreachable-then-sanitized via `scoring.sanitize_for_export()`) are filtered out and left uncolored/transparent, rather than plotted with some placeholder value — explicitly described as the same "don't visually claim a score exists where it doesn't" principle used throughout the scoring pipeline itself (directly echoing `scoring.py`'s own design philosophy around `NaN`/`inf` handling, applied here at the visualization layer). `vmin`/`vmax` default to the actual data's min/max if not explicitly given, so the color scale is always meaningful for whatever's actually being plotted, rather than assuming a fixed range that might not match a given LGA/mode/service combination's real value distribution. |
| **Inputs** | `grid_gdf`; `value_col: str` (any numeric column, generic — this function isn't hardcoded to a specific metric); `title`, `out_path`, `colorbar_label: str`; `settled_only`, `cmap`, `vmin`, `vmax`, `alpha`, plus the shared `scale_bar_km`/`dpi`/`web_path`/`web_dpi`/`figsize` set. |
| **Outputs** | `str`. |
| **Internal workflow** | 1. Raise `KeyError` (with the full column list for debugging) if `value_col` doesn't exist.<br>2. `_ensure_lonlat()`.<br>3. Filter to settled cells if requested, then filter again to non-null values of `value_col` — raise `ValueError` if nothing remains after both filters.<br>4. Compute full-grid `bounds` for context (same pattern as `plot_deficit_map()`).<br>5. Create figure, add basemap.<br>6. Resolve `vmin`/`vmax` from the plotted data's actual range if not explicitly given.<br>7. Single `.plot()` call using GeoPandas' built-in `column=`/`cmap=`/`legend=True` continuous-choropleth support, with `legend_kwds` controlling the colorbar's label, size (`shrink=0.6`), and padding.<br>8. Delegate to `_finalize_and_save()`. |
| **Assumptions** | Assumes `"turbo"` is a reasonable default colormap for travel-time data — a perceptually broad, high-contrast colormap; no explicit justification given in the docstring for this specific choice over alternatives like `"viridis"`. |
| **Complexity** | O(N) for filtering; plotting cost scales with the number of valid cells drawn. |
| **Covered by test(s)** | See [tests.md](../../tests.md). |

### `plot_completeness_map(grid_gdf, service, title, out_path, scale_bar_km=None, dpi=300, web_path=None, web_dpi=None, figsize=(11, 12))`

| | |
|---|---|
| **What it does** | A two-category map (confirmed nearby vs. possible OSM data gap) for one service's completeness flag. |
| **Why written this way — deliberately NOT reusing `plot_deficit_map()`.** | The docstring states this explicitly: this is kept as its own function rather than generalizing `plot_deficit_map()` to handle it, because the underlying *concept* is genuinely different — this map is about OSM coverage confidence, not about access itself — and the docstring warns that conflating the two palettes "would blur exactly the distinction the completeness module exists to preserve." This mirrors, at the visualization layer, the same architectural separation documented in `completeness/grid_check.py` and the [repository overview](../../overview.md): completeness and accessibility are kept as visibly, deliberately distinct concerns throughout the codebase, not just in the underlying data model but in how each is presented. |
| **Inputs** | `grid_gdf`; `service: str`; `title`, `out_path: str`; plus the shared styling/output parameter set. |
| **Outputs** | `str`. |
| **Internal workflow** | 1. Raise `KeyError` if the expected `{service}_completeness_flag` column is missing, with a message pointing at `grid_check.flag_completeness()` as the expected source.<br>2. `_ensure_lonlat()`, filter to settled cells, raise `ValueError` if empty.<br>3. Compute full-grid `bounds`, create figure, add basemap.<br>4. Split settled cells into `confirmed` (`flag_col == False`) and `possible_gap` (`flag_col == True`) — both comparisons use explicit `== True`/`== False` rather than boolean truthiness directly, with an inline `# noqa: E712` comment explaining this is deliberate: explicit comparison behaves correctly against a nullable boolean dtype (where a bare truthiness check could behave unexpectedly on a `pd.NA` value, whereas an explicit `== True`/`== False` comparison evaluates cleanly to `False` for either branch on a null, silently excluding it from both categories rather than raising or misclassifying it).<br>5. Plot each non-empty subset in its accent color (`ACCENT_SECONDARY` for confirmed, `ACCENT_HIGHLIGHT` for gap).<br>6. Build a two-entry legend.<br>7. Delegate to `_finalize_and_save()`. |
| **Assumptions** | Assumes the flag column, while typically a plain Python `bool` dtype in practice, might sometimes carry a nullable/`pd.NA`-capable dtype — the explicit `==` comparisons are defensive against that possibility, per the inline comment's reasoning. |
| **Complexity** | O(N) for the settled filter and category split. |
| **Covered by test(s)** | See [tests.md](../../tests.md). |

## Functions & Classes — Charts

### `plot_mode_comparison_chart(mode_stats, title, out_path, dpi=300, web_path=None, web_dpi=None, figsize=(8, 5.5))`

| | |
|---|---|
| **What it does** | A grouped bar chart (matplotlib, not GeoPandas — this is the one function in the module that isn't a map) comparing "underserved (≥1 service)" and "underserved (both services)" percentages across travel modes. |
| **Why written this way** | Explicitly documented as the static-export equivalent of the Streamlit dashboard's "Findings Summary" metric cards — "so the same headline numbers a judge sees on the live dashboard are also available as a standalone, downloadable figure for a report/slide deck." `mode_stats` (a plain sequence of `(mode_name, pct_any, pct_both)` tuples) is deliberately kept as a plain tuple sequence rather than a bespoke data class, specifically "so both the notebook and the Streamlit app can build this input the same simple way" — minimizing the shape of data either caller needs to construct. |
| **Inputs** | `mode_stats: Sequence[tuple]`; `title`, `out_path: str`; `dpi`, `web_path`, `web_dpi`, `figsize`. |
| **Outputs** | `str`. |
| **Internal workflow** | 1. Raise `ValueError` immediately if `mode_stats` is empty — nothing to chart.<br>2. Unpack labels (capitalized mode names) and the two percentage series.<br>3. Standard matplotlib grouped-bar-chart construction: side-by-side bars per mode (`width=0.35`, offset `± width/2`) for the two percentage series, each in its own accent color.<br>4. Annotate each bar with its exact percentage value as text above the bar.<br>5. Style: x-tick labels, y-axis label, y-limit padded to `1.2×` the max value (headroom for the text annotations above the tallest bars), title, legend, and hiding the top/right plot spines (a common "cleaner chart" styling convention).<br>6. Save at `dpi`, optionally also at `web_path`/`web_dpi` — the same dual-save pattern as `_finalize_and_save()`, but implemented inline here rather than delegating to that function, since this is a chart (no basemap, gridlines, scale bar, or north arrow apply) rather than a map. |
| **Assumptions** | Assumes at most a handful of modes will ever be compared (currently 3: walk/okada/drive) — the `width=0.35` grouped-bar spacing is tuned for a small number of categories and would look cramped with significantly more. |
| **Complexity** | O(M) where M = number of modes in `mode_stats` (at most 3 in practice). |
| **Covered by test(s)** | See [tests.md](../../tests.md). |

## The Orchestrator

### `generate_all_static_outputs(lga_name, grid_gdf, out_dir, modes=("walk", "okada", "drive"), palette="standard", dpi=300, web_dpi=150)`

| | |
|---|---|
| **What it does** | Produces the **full standard set** of publication-styled outputs for one LGA in a single call: a deficit map + health-time map + education-time map per mode, one completeness map per service, and one mode-comparison chart — every image saved at both print and web resolution, plus a `captions.json` mapping every filename to its data-driven caption. |
| **Why written this way** | Three design decisions worth calling out. **(1) Conditional generation, not blind generation**: a continuous time map for a given `(mode, service)` combination is only produced if the corresponding column both exists in `grid_gdf` *and* has at least one non-null value (`grid_gdf[col].notna().any()`) — meaning a partially-scored grid (e.g. scored for `walk` only) produces exactly the outputs that data supports, not empty/broken figures for missing modes. Same conditional pattern applies to completeness maps (only generated if the flag column exists) and the mode-comparison chart (only generated if at least one mode's deficit-score column exists in the settled data). **(2) Captions are written to disk, not just returned** — every caption is included in the function's return dict *and* written to `out_dir/captions.json`, specifically so a consumer that only has file paths later (not this function's original return value) — the docstring's example is `dashboard/app.py` reading exported files back off disk in a later session — can still look up the correct caption for a given image without needing to regenerate or hand-write it separately. **(3) The `_web_kwargs()` inline closure** centralizes the repeated "build a web-tier path, register it in `produced['web']`, return the kwargs to pass through" logic used identically across all four plot-function call types (deficit, continuous, completeness, chart), rather than repeating that bookkeeping at each of the roughly half-dozen call sites within this function. |
| **Inputs** | `lga_name: str`; `grid_gdf: GeoDataFrame` (fully-scored, expected to already have gone through the complete `scoring.py` + `grid_check.py` pipeline for whichever modes/services are being exported); `out_dir: str`; `modes: Iterable[str]`, default all three; `palette: str`, default `"standard"`; `dpi: int`, default `300`; `web_dpi: Optional[int]`, default `150` (pass `None` to skip web-tier generation entirely). |
| **Outputs** | `dict` with keys `"print"` (list of print-tier file paths), `"web"` (list of web-tier file paths, empty if `web_dpi is None`), `"captions"` (dict mapping each print-tier filename to its caption string). Also writes `out_dir/captions.json` as a side effect. |
| **Internal workflow** | 1. Create `out_dir` and, if `web_dpi is not None`, `out_dir/web/`.<br>2. Define the `_web_kwargs(fname)` closure described above.<br>3. **Per-mode loop**: for each mode, always generate the deficit map (via `plot_deficit_map()`) and its caption; then, for each of health/education, generate the continuous time map *only if* the column exists with at least one non-null value.<br>4. **Per-service loop**: for each of health/education, generate the completeness map only if the flag column exists.<br>5. **Mode-comparison chart**: filter to settled cells; for each mode with a deficit-score column present, compute `pct_any`/`pct_both` and collect into `mode_stats`; if `mode_stats` ended up non-empty, generate the chart and its caption.<br>6. Write `produced["captions"]` to `out_dir/captions.json` as indented JSON.<br>7. Return `produced`. |
| **Assumptions** | Assumes filenames are unique and stable across runs for the same LGA (`f"{safe_lga}_{...}.jpg"` naming, where `safe_lga = lga_name.replace(" ", "_")`) — re-running this function for the same LGA will silently overwrite previous outputs at the same paths, which is presumably the intended behavior for a re-run with updated data, but worth being aware of (no versioning or backup of previous outputs happens here). |
| **Complexity** | O(M × (map generation cost)) where M = number of modes, dominated by the sum of all individual `plot_*()` calls' costs — the dominant real-world cost is almost certainly the repeated OSM basemap tile fetches (one per generated map/chart, each independently calling `add_osm_basemap()`), not the underlying data plotting itself. |
| **Concurrency / race conditions** | None — fully sequential generation, no threading; this means total wall-clock time for a full LGA export (up to ~11 figures: 3 deficit + up to 6 continuous + 2 completeness + 1 chart) is the sum of every individual figure's generation time, including basemap fetch latency for each one independently — a plausible target for future parallelization, though not currently implemented. |
| **Covered by test(s)** | See [tests.md](../../tests.md). |

## Gotchas

- **`add_osm_basemap()`'s two fallback paths produce visually different
  results.** The no-`contextily`-installed path adds an explanatory text
  annotation ("OSM basemap unavailable...") directly on the figure; the
  tile-fetch-failed path (network/timeout error) only sets the gray
  background and emits a Python `warnings.warn()` — which won't be visible
  on the figure itself, only in whatever captured stderr/logs a caller
  happens to be watching. A generated map with a plain gray background and
  no on-figure explanation could be mistaken for a rendering bug rather
  than a network-access limitation, unless the accompanying warning is
  actually seen.
- **The network-dependent branch of `add_osm_basemap()` is marked
  `# pragma: no cover`** in the source, signaling it's excluded from
  code-coverage accounting — meaning the actual tile-fetch-failure recovery
  path likely isn't exercised by the automated test suite in the normal
  course of running tests (see [tests.md](../../tests.md) for what
  `test_static_maps.py` actually covers).
- **`generate_all_static_outputs()` silently overwrites previous output
  files** on a re-run for the same LGA — no versioning, timestamping, or
  backup of prior figures happens automatically.
- **`plot_completeness_map()`'s explicit `== True`/`== False` comparisons
  are a deliberate defensive choice**, not an oversight the linter is
  merely tolerating — see that function's own documentation above for the
  nullable-dtype reasoning.
