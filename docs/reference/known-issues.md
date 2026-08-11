# Known Issues & Design Decisions

## Case Study: Polygon-vs-Point Health Facilities (Akure North)

_TODO: full writeup of the bug — all 14 Akure North health facilities stored as
Polygons rather than Points in OSM, causing false "unreachable" scores in
isochrone/nearest-facility logic; fixed via centroid-collapse logic
(`_collapse_areas_to_points()` in `clean.py`). Cover: how it was discovered,
why it happened, the fix, and the before/after impact on underserved
statistics._

## Other Resolved Issues

- _TODO: Earth Engine-style 50MB caps / analogous limits, if any apply here._
- _TODO: dependency and repo-naming fixes noted in project history._
- _TODO: CI/CD (GitHub Actions) issues encountered and resolved._

## Open Limitations / Assumptions

_TODO: anything not yet fixed, or accepted as a known simplification (e.g. fixed travel speeds per mode, grid cell size trade-offs)._
