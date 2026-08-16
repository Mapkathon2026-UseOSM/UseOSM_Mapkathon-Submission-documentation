# Known Issues & Design Decisions

## Case Study: Polygon-vs-Point Health Facilities (Akure North)

**The single most significant technical finding across both repositories.**

### What happened

All 14 health facilities extracted from OpenStreetMap for Akure North LGA
were mapped in OSM as building-outline `Polygon`/`MultiPolygon` geometry —
someone had traced each facility's footprint, rather than dropping a single
point node at its location. Both mapping styles are entirely valid OSM
practice; OSM has no rule requiring one over the other for a given feature
type.

The problem is what a `Polygon` geometry *lacks*: a `.x`/`.y` coordinate
pair. Every piece of routing/distance logic in
`akure-accessibility-dashboard` — snapping a facility to its nearest graph
node, computing shortest-path distance/time — needs a single coordinate to
work from. Before the fix described below existed, a facility with no
usable coordinate simply failed to participate in routing at all — not
with a raised error, but silently. Downstream, every grid cell that would
have depended on that facility for its accessibility score instead scored
as if the facility didn't exist. Because *all 14* of Akure North's health
facilities were mapped this way, the practical effect was that health
accessibility for the entire LGA scored as catastrophically worse than
reality — 0% effective health access, indistinguishable in the output from
a genuine, severe service crisis.

### The fix

`lga_extractor.clean._collapse_areas_to_points()` (see
[`clean.md`](../lga-osm-extractor/modules/clean.md)) runs as part of the
standard cleaning pipeline for the `health_facilities` and `schools`
layers specifically (the two layers in `clean.POINT_LAYERS`): any
`Polygon`/`MultiPolygon` geometry is replaced with its centroid `Point`,
computed *after* reprojection into a metric CRS (never in raw lat/lon
degrees, where a centroid calculation would be measurably distorted). This
guarantees every facility exported by `lga-osm-extractor` for these two
layers is a genuine `Point`, before `akure-accessibility-dashboard` ever
sees the data — see [Cross-Repo Integration](../cross-repo/integration.md)
for the exact schema guarantee this provides downstream.

### Defense in depth

The fix isn't confined to one function. Two further, independent
safeguards exist specifically because of this incident:

1. **`isochrones.nearest_graph_node()`** (see
   [`isochrones.md`](../akure-accessibility-dashboard/modules/accessibility/isochrones.md))
   has its own defensive `.centroid` fallback for any non-`Point` geometry
   that reaches it — a genuine second line of defense, for any geometry
   that somehow arrives at this function without having gone through the
   upstream cleaning step.
2. **`isochrones.batch_nearest_facility_distances()`** distinguishes a
   *partial* facility-snapping failure (some facilities genuinely
   unreachable — expected, unremarkable) from a *total* failure (every
   facility in a non-empty layer fails to snap — almost certainly a
   real bug), raising a loud `UserWarning` with actionable guidance
   (check geometry type and CRS) specifically for the total-failure case.
   Before this distinction existed, a bare `except: continue` at this
   point in the code is exactly what allowed the original bug to produce
   0% health access silently, with no visible signal anything had gone
   wrong.

### Impact on the project's findings

This fix changes what the underserved statistics for Akure North actually
measure: before it, the 14 health facilities affected by the tagging
mismatch were silently excluded from every routing and scoring
calculation, so any "underserved" figure computed prior to the fix
understates real access by omitting those facilities entirely rather than
correctly counting them as reachable or unreachable. The dashboard and
static maps in their current form reflect the corrected data — with all
mapped facilities actually participating in scoring — not the pre-fix
output. No specific before/after percentage change is quoted here, since
that comparison was not preserved as a tracked artifact; the point worth
understanding is the *direction and mechanism* of the error (facilities
dropped out silently, not mis-scored), not a specific magnitude.

## Other Notable Design Decisions Worth Understanding as a Group

These aren't bugs, but are easy to misread as inconsistencies if
encountered in isolation across different files — documented together
here for a single reference point.

### CRS strategy differs between the two repositories — now partially, not fully, resolved

`lga-osm-extractor` auto-selects the correct UTM zone for wherever an LGA
actually is (`clean.resolve_target_crs()` / `clean.utm_epsg_for_longitude()`
— see [`clean.md`](../lga-osm-extractor/modules/clean.md)), making it
correct for any LGA in Nigeria. `akure-accessibility-dashboard` previously
hardcoded `EPSG:32631` throughout `scoring.py` unconditionally. **This
revision adds a genuine fix**: `scoring.build_grid()` now accepts an
explicit `target_crs` parameter, intended to be sourced from the
extractor's own manifest via
[`data_contract.resolve_crs_from_manifest()`](../akure-accessibility-dashboard/modules/data_contract.md)
— see [`scoring.md`](../akure-accessibility-dashboard/modules/accessibility/scoring.md).
**The gap is narrowed, not eliminated**: a caller who doesn't know to pass
`target_crs` explicitly still silently gets the Akure-only-correct
`FALLBACK_CRS`. Unlike `lga_extractor`, `akure_access` still does not
*automatically* derive the correct zone from the boundary's own
longitude — it depends on the caller opting in to read the extractor's
determination. See [Cross-Repo Integration](../cross-repo/integration.md).

### Two different "settled cell" thresholds coexist — now explicitly justified in source, not just inferred

`akure_access.accessibility.scoring` (and `dashboard/app.py`) treat any
cell with `building_count > 0` as settled, for both routing and display
purposes. `akure_access.completeness.grid_check` uses a stricter default
threshold of `building_count >= 3` for its own settlement definition. A
cell with exactly 1 or 2 buildings is therefore scored for accessibility
but excluded from completeness flagging entirely. **This documentation
site previously noted this was "very likely intentional" based on
reasoning about what would make sense, without direct confirmation in the
source.** This revision's [`config.py`](../akure-accessibility-dashboard/modules/config.md)
resolves that uncertainty directly: `config/default.yaml`'s own inline
comment now states the justification explicitly — a completeness flag is
a stronger claim than a routing-inclusion decision, and warrants more
confidence in the settlement signal. No longer an unexplained
discrepancy, just a documented design choice. See
[`grid_check.md`](../akure-accessibility-dashboard/modules/completeness/grid_check.md).

### The `DEFAULT_THRESHOLDS_MIN` drift risk — resolved in this revision

**This documentation site previously flagged a specific, live risk**:
`insights.py`'s `DEFAULT_THRESHOLDS_MIN` was an independently-hardcoded
dict, manually kept in sync with the project's analysis notebook's own
separate threshold configuration by convention, not by any code that
would catch the two drifting apart. **This revision's `config.py`
resolves this directly** — both `insights.DEFAULT_THRESHOLDS_MIN` and
`scoring.DEFAULT_ACCESS_THRESHOLD_MIN` now derive from the same
`config.get_config()` call, so there is no longer a second copy of these
values that could silently diverge. `insights.py`'s own updated source
comment references this exact risk by name, pointing at this
documentation site's known-issues page directly. See
[`insights.md`](../akure-accessibility-dashboard/modules/insights.md) and
[`config.md`](../akure-accessibility-dashboard/modules/config.md) for the
full before/after.

### Color palette constants are duplicated, not shared — still true, now a sharper contrast

`dashboard/app.py`'s `DEFICIT_COLORS`/`CONTINUOUS_CMAP` and
`visualization/static_maps.py`'s `DEFICIT_PALETTES` remain independently
defined in each file, unchanged by this revision, both documented as
intentionally using the same actual color values so the live dashboard
and static exports never visually disagree — but the values are still
duplicated in source, not imported from one shared location, and still
have no test guarding against the two drifting apart. **This is now a
sharper, more visible inconsistency than before**: the closely analogous
`DEFAULT_THRESHOLDS_MIN` drift risk (previous entry above) *was* resolved
in this same revision via the same `config.py` centralization pattern
that could, in principle, resolve this one too — but hasn't yet. See
[`tests.md`](../akure-accessibility-dashboard/tests.md).

### Two genuinely different isochrone accuracy models coexist

`isochrones.compute_isochrone_polygon()` produces a fast convex-hull
*approximation*, used only for illustrative dashboard catchment overlays.
Everything the project's actual deficit scores are built from —
`nearest_facility_distance_and_time()`,
`batch_nearest_facility_distances()`, `lookup_nearest_distance_time()` —
computes exact network shortest-path results instead. Unchanged by this
revision. See
[`isochrones.md`](../akure-accessibility-dashboard/modules/accessibility/isochrones.md).

### New: the extractor's routing-graph construction paths reversed roles

Before this revision, supplying a boundary polygon to
`network_graph.graph_from_roads()` selected a live OSM query as the
"recommended default," with the extractor's own `roads_gdf` as a
documented fallback with known topology limitations (no
walk/drive network differentiation, no endpoint-junction merging). **This
revision inverts that**: `roads_gdf` is now the default path whenever it's
available — even alongside a supplied boundary — chosen specifically for
reproducibility (no live-network dependency at analysis time, no risk of
OSM's live state differing from the extractor's cached snapshot). The
`roads_gdf` path's previous topology limitation (no shared-junction
merging across floating-point-mismatched coordinates) is also fixed in
this same revision, via endpoint snapping — see
[`network_graph.md`](../akure-accessibility-dashboard/modules/accessibility/network_graph.md).
The walk/drive network-differentiation limitation is **not** fixed, and
now applies by default rather than only in a fallback scenario — worth
knowing if genuine mode-specific network topology matters for a specific
analysis.

### New: five modules exist as tested library capabilities, not yet integrated into the dashboard

`config.py`, `data_contract.py`, `facility_classification.py`,
`sensitivity.py`, and `status.py` are all new, genuinely tested additions
in this revision — but `dashboard/app.py` itself is **unchanged**,
confirmed by direct diff. None of these five modules' outputs are
currently read anywhere in the deployed interactive dashboard. This is
not a bug — each module is independently useful from a notebook — but
worth knowing if you're looking for where, say, facility classification or
the fused accessibility/completeness status shows up in the live app: it
doesn't, yet. See each module's own page (linked from the
[akure-accessibility-dashboard overview](../akure-accessibility-dashboard/overview.md#module-map))
for this same note, restated per-module.

### New: the cross-repo integration test was not updated for any of this revision's new contract surface

The single test whose entire purpose is verifying the two repositories
actually agree with each other —
`tests/test_cross_repo_integration.py` — is byte-identical to its
pre-revision version. `manifest.json`, `boundary.geojson`, the richer
`clean_layers()` GeoJSON schema, the new `target_crs`/`source` parameters,
and the default routing-graph path reversal are all currently unverified
by this test. See [Cross-Repo Integration](../cross-repo/integration.md)
for the full detail — this is likely the single most significant gap
introduced across both repositories by this revision, precisely because
it's the test specifically designed to catch this class of drift.

### CI/CD and dependency fixes (from project history)

The Map<>kathon 2026 project history (per accumulated context across both
repos) also involved: repository-naming fixes across the `Mapkathon2026-UseOSM`
GitHub org, multiple dependency-resolution fixes for CI/CD via GitHub
Actions, and a documented Streamlit Cloud deployment fix — the `sys.path`
insertion at the top of `dashboard/app.py` (see
[`dashboard-app.md`](../akure-accessibility-dashboard/dashboard-app.md)),
addressing a confirmed case where Streamlit Cloud's `requirements.txt`
based install never actually installed the local `akure_access` package as
a proper package, and the app had only ever worked by accident of working
directory.

## Open Limitations / Accepted Simplifications

These are stated directly in the source as deliberate, documented
trade-offs — not oversights:

- **A single fixed average speed per travel mode**, with no accounting for
  road-type-specific speed variation, congestion, time-of-day, or road
  surface quality. See [`network_graph.md`](../akure-accessibility-dashboard/modules/accessibility/network_graph.md).
- **Building count as a population proxy**, not actual census/population
  data — used throughout because fine-grained population figures aren't
  available for this study area at the required resolution. **New this
  revision**: this limitation is now stated explicitly and prominently in
  source, via `scoring.SETTLEMENT_PROXY_DISCLAIMER` and
  `insights.describe_settlement_proxy_limitation()` — both new additions
  specifically making this caveat visible to any consumer, rather than
  something only documented in a docstring a reader might not see. Both
  also note the codebase is structured so a real population dataset could
  be substituted in the future without changing any downstream function's
  interface, since every consumer only ever checks `building_count`
  against a threshold, never its specific value in a population-specific
  way.
- **A 500m fixed grid cell size**, not derived from any formal
  sizing/precision analysis — a reasonable round number, now
  config-derived (see [`config.md`](../akure-accessibility-dashboard/modules/config.md))
  but numerically unchanged.
- **A spherical-Earth approximation** in `static_maps.add_scale_bar()`'s
  km-to-degree conversion — acceptable at map-reading scale, explicitly
  not survey-grade. Unchanged by this revision.
- **The `roads_gdf` graph-construction path still cannot distinguish
  walk-only paths from vehicle roads** — unchanged limitation, but now
  the cost of the *default* path rather than a documented fallback's
  limitation, since this revision reversed which path is preferred by
  default. See the dedicated entry above and
  [`network_graph.md`](../akure-accessibility-dashboard/modules/accessibility/network_graph.md).
- **New: `run_speed_sensitivity()` is expensive and should only be swept
  across a small handful of values**, not a fine grid — each additional
  speed tested is a full graph rebuild plus a full routing pass, unlike
  the much cheaper threshold-sweep sibling function. See
  [`sensitivity.md`](../akure-accessibility-dashboard/modules/sensitivity.md).
- **New: `sensitivity.summarize_robustness()`'s `jaccard_threshold=0.8`
  is a stated convention, not a statistically derived cutoff** — a
  different tolerance for what counts as "robust enough" could flip a
  borderline finding's reported conclusion. See
  [`sensitivity.md`](../akure-accessibility-dashboard/modules/sensitivity.md).
