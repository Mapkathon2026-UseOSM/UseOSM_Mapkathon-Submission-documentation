# boundary.py

!!! info "Source"
    `lga_extractor/boundary.py` (220 lines)

## Purpose

Resolves a Nigerian LGA name (e.g. `"Akure North"`) into a validated,
standardized administrative boundary polygon, either by geocoding it against
OSM/Nominatim via OSMnx, or by loading a manually-supplied boundary file.
This boundary is the first thing every other module in the pipeline depends
on — `layers.py` queries features *within* it, and `clean.py` uses its
centroid to pick a correct UTM zone for the whole run.

The module's central concern isn't really "how do I get a boundary" (that's
one line: `ox.geocode_to_gdf(query)`) — it's **how do I know whether the
boundary I got back is actually right**, since a geocoder can return a
confident-looking but wrong result with no indication anything went wrong.

## Dependencies

- **Imports:** `geopandas`, `shapely.geometry.base.BaseGeometry`, `osmnx`.
  Also imports `resolve_target_crs` from [`clean.py`](clean.md) — locally,
  inside `_validate_and_standardize()` — to measure candidate boundary area
  in a correct metric projection rather than raw WGS84 degrees.
- **Imported by:** `pipeline.py` (as the first stage of `extract_lga()`).

## Functions & Classes

### `BoundaryResolutionError`

An exception class (subclass of `Exception`, no custom behavior) raised
whenever a boundary cannot be confidently resolved — either because OSM
returned nothing / raised an error, or because a resolved boundary fails one
of the hard validation checks in `_validate_and_standardize()`. Callers
(`pipeline.py`, `app.py`, `cli.py`) catch this to surface a clear,
actionable message rather than letting a downstream module fail confusingly
on a bad boundary later in the pipeline.

### `resolve_boundary(lga_name, state_name=None, manual_boundary_path=None)`

| | |
|---|---|
| **What it does** | Returns a single-row `GeoDataFrame` containing the validated LGA boundary, in EPSG:4326, tagged with `boundary_source` and `validation_warnings` columns. |
| **Why written this way** | Two resolution paths are supported — manual file and OSM geocode — and both are funneled through the *same* validation function (`_validate_and_standardize()`) at the end, so a manually-supplied boundary gets exactly the same sanity checks as an OSM-geocoded one. This matters: a manual file is a human-supplied escape hatch, but a typo'd path to the wrong file, or an outdated boundary, is just as capable of silently corrupting the run as a bad geocode. |
| **Inputs** | `lga_name: str` (required, e.g. `"Akure North"`); `state_name: str` (optional, e.g. `"Ondo"` — recommended to disambiguate LGAs that share a name across states); `manual_boundary_path: str` (optional path to a GeoJSON/Shapefile, used instead of querying OSM). |
| **Outputs** | A single-row `geopandas.GeoDataFrame`, CRS EPSG:4326, with the boundary geometry plus `boundary_source` and `validation_warnings` columns. |
| **Internal workflow** | 1. If `manual_boundary_path` is given: read it with `gpd.read_file()`, check it's non-empty with valid geometry, pass to `_validate_and_standardize()` with `source="manual:<path>"`, return.<br>2. Otherwise: build a Nominatim-style query string (`"{lga_name}, {state_name}, Nigeria"` or `"{lga_name}, Nigeria"` if no state given), call `ox.geocode_to_gdf(query)`.<br>3. If that raises, wrap it in `BoundaryResolutionError` with the original exception chained (`from exc`), so the underlying cause isn't lost.<br>4. If it returns an empty result, raise `BoundaryResolutionError` — OSM found nothing for that query.<br>5. Otherwise pass the result to `_validate_and_standardize()` with `source="osm_geocode:<query>"`, return its result. |
| **Assumptions** | Assumes `state_name`, when given, is spelled/formatted the way Nominatim expects (no fuzzy matching is attempted). Assumes network access to OSM's Nominatim/geocoding service when not using a manual file. |
| **Complexity** | O(1) beyond the network call itself — no loops or scaling with input size. The actual latency is dominated by the geocoding API round-trip, not by anything in this function. |
| **Concurrency / race conditions** | None — this function is not called concurrently anywhere in the codebase (`layers.py` is the module that introduces threading, not this one). |
| **Covered by test(s)** | See [tests.md](../tests.md) — `test_extraction.py`. |

### `_validate_and_standardize(gdf, source, lga_name=None, state_name=None)`

This is the real substance of the module. It takes whatever geometry was
resolved (from either path above) and decides whether to trust it.

| | |
|---|---|
| **What it does** | Standardizes CRS to EPSG:4326, then runs two tiers of validation — **hard checks** that raise `BoundaryResolutionError`, and **soft checks** that are recorded as warnings but don't block execution — before returning a cleaned, single-row, metadata-tagged `GeoDataFrame`. |
| **Why written this way** | The hard/soft split reflects a deliberate confidence judgment. Geometry validity, being inside Nigeria's bounding box, and having a plausible LGA-scale area are all strong, mechanical signals that something went *badly* wrong (wrong place entirely) — these block. Whether Nominatim's free-text `display_name` field happens to mention the requested LGA/state name is a much weaker signal — Nominatim's naming and abbreviation conventions vary a lot, so a mismatch is often nothing, not necessarily a wrong resolution — so this only warns, never blocks. Treating a weak signal as a hard failure would create false rejections; treating a strong signal as just a warning would let genuinely wrong boundaries slip through silently. |
| **Inputs** | `gdf` (raw resolved `GeoDataFrame`, from either source); `source: str` (human-readable provenance string, e.g. `"osm_geocode:Akure North, Ondo, Nigeria"`, recorded verbatim in the output); `lga_name`, `state_name` (optional, used only for the soft `display_name` check). |
| **Outputs** | Single-row `GeoDataFrame`, CRS EPSG:4326, with two added columns: `boundary_source` (the `source` string) and `validation_warnings` (`None`, or a `"; "`-joined string of soft-check concerns). |
| **Internal workflow** | 1. Grab the first geometry; raise if it's `None`, not a valid Shapely geometry, or empty.<br>2. Standardize CRS: `set_crs("EPSG:4326")` if the input has no CRS at all, otherwise `to_crs("EPSG:4326")` to reproject.<br>3. **Hard check — geography:** compute the geometry's centroid, raise if it falls outside `NIGERIA_BBOX` (a generous approximate bounding box for the whole country). This catches name collisions with similarly-named places elsewhere in the world.<br>4. **Hard check — area:** call `resolve_target_crs(gdf)` (from `clean.py`) to get the correct local UTM zone, reproject a copy into it, compute area in km². Raise if area is below `MIN_PLAUSIBLE_LGA_AREA_KM2` (2 km² — catches "a single point/building was resolved") or above `MAX_PLAUSIBLE_LGA_AREA_KM2` (10,000 km² — catches "a whole state/country was resolved").<br>5. **Soft check — display name:** if the input has a `display_name` column and an `lga_name`/`state_name` was supplied, lowercase-substring-check whether the first word of `lga_name` and/or `state_name` appears in it; append a warning string (not a raise) for each miss.<br>6. Take just the first row (`.iloc[[0]]`), attach `boundary_source` and `validation_warnings`, return. |
| **Assumptions** | Assumes Nigeria's true extent is fully contained within the hardcoded `NIGERIA_BBOX` margin. Assumes `MIN_PLAUSIBLE_LGA_AREA_KM2`/`MAX_PLAUSIBLE_LGA_AREA_KM2` (2–10,000 km²) is wide enough to never falsely reject a genuinely valid but unusually small/large LGA — the module's own comments note these bounds are deliberately generous for exactly this reason. Assumes the display-name substring check on just the *first word* of `lga_name` is a reasonable enough heuristic (it will not catch a boundary whose display name uses a different word order or abbreviation entirely — but that's the intended looseness, since this is a warning, not a hard check). |
| **Complexity** | O(1) — a fixed, small number of geometric operations (centroid, one reprojection, one area calculation) regardless of the boundary's vertex count; no loops over the geometry itself. |
| **Concurrency / race conditions** | None. Pure function over its inputs, no shared mutable state, no I/O beyond the CRS transform (in-memory). |
| **Covered by test(s)** | See [tests.md](../tests.md) — `test_extraction.py`. |

## Gotchas

- **The area check depends on `clean.py`, creating an import that only
  resolves at call time.** `_validate_and_standardize()` does
  `from .clean import resolve_target_crs` *inside* the function body, not at
  module top level. This avoids a circular import at module load time (since
  `clean.py` doesn't import from `boundary.py`, there's no actual cycle here
  — but the local import keeps the dependency scoped to exactly where it's
  needed and makes the coupling visible at the call site rather than buried
  in the top-of-file import block).
- **A "wrong but plausible" boundary can still pass.** The validation is
  deliberately loose — it catches *categorically* wrong resolutions (totally
  different location, single point, entire state), not *subtly* wrong ones
  (e.g. a neighboring LGA with a similar name and similar size, inside
  Nigeria, at a plausible area). For genuinely ambiguous LGA names, a manual
  boundary file is the safer path, not blind trust in geocoding.
- **`display_name` isn't guaranteed to exist.** The soft check is skipped
  entirely (no warning generated) if the resolved `GeoDataFrame` has no
  `display_name` column at all — this happens for manually-supplied boundary
  files, which typically won't have that field.
