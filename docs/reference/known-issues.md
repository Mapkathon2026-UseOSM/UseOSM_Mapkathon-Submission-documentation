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

Fixing this bug substantially changed Akure North's key underserved
statistics and strengthened the submission's core multi-modal findings —
the corrected numbers are what the project's actual dashboard and static
maps now reflect.

## Other Notable Design Decisions Worth Understanding as a Group

These aren't bugs, but are easy to misread as inconsistencies if
encountered in isolation across different files — documented together
here for a single reference point.

### CRS strategy differs between the two repositories

`lga-osm-extractor` auto-selects the correct UTM zone for wherever an LGA
actually is (`clean.resolve_target_crs()` / `clean.utm_epsg_for_longitude()`
— see [`clean.md`](../lga-osm-extractor/modules/clean.md)), making it
correct for any LGA in Nigeria. `akure-accessibility-dashboard` hardcodes
`EPSG:32631` throughout `scoring.py` — correct for the project's actual
scope (Akure North/South, both within Zone 31N), but would need updating
if ever pointed at an LGA outside that zone. This is a genuine, current
scope difference between the two repos, not an oversight in the dashboard
— see [Cross-Repo Integration](../cross-repo/integration.md).

### Two different "settled cell" thresholds coexist

`akure_access.accessibility.scoring` (and `dashboard/app.py`) treat any
cell with `building_count > 0` as settled, for both routing and display
purposes. `akure_access.completeness.grid_check` uses a stricter default
threshold of `building_count >= 3` for its own settlement definition. A
cell with exactly 1 or 2 buildings is therefore scored for accessibility
but excluded from completeness flagging entirely. This is very likely
intentional — a completeness flag is a stronger claim ("this looks like a
genuine gap in OSM mapping") that plausibly warrants more confidence in
the settlement signal than a routing decision does — but nothing in the
code ties the two thresholds together, and no comment states the
difference is deliberate rather than accidental. See
[`grid_check.md`](../akure-accessibility-dashboard/modules/completeness/grid_check.md).

### Color palette constants are duplicated, not shared

`dashboard/app.py`'s `DEFICIT_COLORS`/`CONTINUOUS_CMAP` and
`visualization/static_maps.py`'s `DEFICIT_PALETTES` are independently
defined in each file, both documented as intentionally using the same
actual color values so the live dashboard and static exports never
visually disagree — but the values are duplicated in source, not imported
from one shared location. A future palette change applied to only one file
would silently break that visual consistency, with no test currently
guarding against it (unlike the closely analogous
`DEFAULT_THRESHOLDS_MIN`-vs-notebook-config drift, which *is* directly
tested — see [`tests.md`](../akure-accessibility-dashboard/tests.md)).

### Two genuinely different isochrone accuracy models coexist

`isochrones.compute_isochrone_polygon()` produces a fast convex-hull
*approximation*, used only for illustrative dashboard catchment overlays.
Everything the project's actual deficit scores are built from —
`nearest_facility_distance_and_time()`,
`batch_nearest_facility_distances()`, `lookup_nearest_distance_time()` —
computes exact network shortest-path results instead. A map showing a
facility's "15-minute walking catchment" and a grid cell's individually
scored "18 minutes to nearest facility" are produced by genuinely
different computational methods with different accuracy characteristics,
despite both describing the same underlying concept. See
[`isochrones.md`](../akure-accessibility-dashboard/modules/accessibility/isochrones.md).

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
  available for this study area at the required resolution.
- **A 500m fixed grid cell size**, not derived from any formal
  sizing/precision analysis — a reasonable round number, not a
  statistically justified resolution choice.
- **A spherical-Earth approximation** in `static_maps.add_scale_bar()`'s
  km-to-degree conversion — acceptable at map-reading scale, explicitly
  not survey-grade.
- **The fallback graph-construction path** in `network_graph._graph_from_geometries()`
  has no OSM-topology cleanup, meaning two road segments that visually
  cross without sharing an exact coordinate aren't connected in the
  resulting graph — a known, accepted limitation of that specific
  (non-default) construction path.
