# network_graph.py

!!! info "Source"
    `akure_access/accessibility/network_graph.py` (185 lines — grew from
    156 with the addition of the `source` parameter, endpoint snapping,
    and config-driven constants, see below)

## Purpose

Builds a routable `NetworkX` graph for a given travel mode (walk, okada,
drive) — the foundation everything else in the `accessibility` package sits
on top of. Every isochrone computation and nearest-facility distance/time
lookup in [`isochrones.py`](isochrones.md) operates on a graph produced
here.

This is the module most consequentially reworked in this revision, and the
change is a genuine **reversal of which construction path is the default**
— worth understanding in detail, since it changes which path a caller gets
without any argument change on their part.

## The reversal: `roads_gdf` is now the default, live OSM is now explicit opt-in

**Before this revision:** supplying `boundary_polygon` selected the
"recommended default" path (a live `osmnx.graph_from_polygon()` query);
`roads_gdf`-only construction was the fallback, used only when no boundary
was available, and was documented as *less robust* than the OSMnx path.

**In this revision, that recommendation is inverted.** The module's own
docstring states the new default plainly: *"roads_gdf... is the DEFAULT
path whenever `roads_gdf` is non-empty, because it guarantees the graph
matches exactly the same roads that were extracted, validated and
versioned upstream — reproducibility of the whole pipeline depends on the
dashboard not silently re-querying OSM with different results at analysis
time."* Live OSM querying is now named `"live_osm"` and treated as *"an
explicit opt-in 'refresh' mode, not the default, so that a dashboard run
is reproducible from the extractor's exported files without requiring live
internet access to OSM."*

**Why this matters beyond just "which default changed":** it's a
philosophical shift in what this module optimizes for. The previous
revision prioritized routing *fidelity* (OSMnx's proper topology handling,
correct walk/drive network differentiation) as the default, accepting a
live-network dependency as the cost. This revision prioritizes
*reproducibility* — the same `roads_gdf` input, read from the extractor's
versioned, cached export, now always produces the same graph, with no
dependency on OSM's live state at analysis time, or on network access being
available at all. The previous revision's fidelity advantage for the
`roads_gdf` path (no walk/drive differentiation) hasn't gone away — see
Gotchas — but it's now the accepted cost of the *default* path, not the
fallback path.

## New: the `source` parameter

```python
VALID_SOURCES = ("auto", "roads_gdf", "live_osm")
```

`graph_from_roads(..., source="auto")` — three explicit modes, replacing
the previous implicit "boundary given → path 1, else → path 2" branching:

- **`"auto"` (default):** use `roads_gdf` if it's non-empty; otherwise fall
  back to `boundary_polygon` (a live OSM query) if one was supplied;
  otherwise raise `ValueError` — there's nothing to build a graph from.
- **`"roads_gdf"`:** force the versioned-dataset path explicitly. Raises
  `ValueError` if `roads_gdf` is empty or `None` — a caller asking for this
  path specifically wants an error if it can't be honored, not a silent
  fallback to something else.
- **`"live_osm"`:** force a live OSM query explicitly. Raises `ValueError`
  if `boundary_polygon` wasn't supplied. This is the path a caller
  deliberately reaches for when they specifically want a fresh, current
  view of OSM data rather than the extractor's cached snapshot — e.g.
  checking whether OSM coverage has improved for an LGA since the last
  extraction run.

**Both explicit modes fail loudly rather than silently falling back** —
only `"auto"` has fallback behavior at all. This is a deliberate design
choice: a caller who explicitly asked for `source="roads_gdf"` and gets
silently routed to a live OSM query instead (because `roads_gdf` happened
to be empty) would have no way to know their reproducibility guarantee
was quietly violated; raising instead makes that impossible.

## Constants (now config-driven)

```python
WALKING_SPEED_KPH = _config["accessibility"]["modes"]["walk"]["speed_kph"]
MODE_CONFIG = _config["accessibility"]["modes"]
```

**Both are now derived from [`config.get_config()`](../config.md)** rather
than independent hardcoded literals — read once, at import time, into
plain module-level constants (not functions), specifically so every
existing caller/import (`from network_graph import MODE_CONFIG`) continues
to work completely unchanged. See [`config.md`](../config.md) for the full
migration story and its import-time-snapshot caveat, which applies
identically here. The actual numeric values are unchanged from before —
`"okada"` still models on OSM's `"drive"` network type at a lower assumed
speed (25 km/h vs. 35 km/h for cars), for the same reasons as before (see
[Known Issues](../../../reference/known-issues.md)).

## `graph_from_roads(roads_gdf, boundary_polygon=None, mode="walk", speed_kph=None, source="auto")`

| | |
|---|---|
| **What it does** | Builds a routable `networkx.MultiDiGraph` (or, on the `roads_gdf` path, a `MultiGraph` — see Gotchas, unchanged nuance) for the requested mode, resolving which construction path to use per the `source` parameter described above. |
| **Why written this way** | Validation happens in two stages: `mode` and `source` are checked against their respective valid-value tuples up front (`ValueError` listing valid options on either mismatch); then `effective_source` is resolved from `source` + data availability, with the three-branch logic (`auto` / `roads_gdf` / `live_osm`) described above. Resolving `effective_source` as an explicit local variable, rather than branching on `source` directly further down, means the graph-construction step (path selection) and the later `G.graph["source"] = effective_source` assignment both reference one single, already-validated value. |
| **New: `G.graph["source"]` records which path actually ran.** | Every returned graph now carries a `"source"` graph-level attribute — `"roads_gdf"` or `"live_osm"` — recording which path was **actually used**, not just what the caller requested (relevant specifically for `source="auto"`, where the effective path depends on data availability, not just the argument value). A caller inspecting an already-built graph can determine its provenance without needing to have tracked the original call arguments themselves. |
| **Inputs** | `roads_gdf: GeoDataFrame`, now effectively optional (can be `None` or empty when `source="live_osm"` or `source="auto"` with a boundary available) — a real signature/behavior change from the previous revision, where it was a required positional-style argument in practice. `boundary_polygon`, optional. `mode: str`, default `"walk"`. `speed_kph: float`, optional. `source: str`, **new**, default `"auto"`. |
| **Outputs** | The graph, plus graph-level `mode`, `speed_kph`, and (**new**) `source` attributes. |
| **Internal workflow** | 1. Validate `mode` against `MODE_CONFIG`.<br>2. Validate `source` against `VALID_SOURCES` (**new**).<br>3. Resolve `effective_speed`.<br>4. Compute `roads_available = roads_gdf is not None and len(roads_gdf) > 0`.<br>5. Resolve `effective_source`: for `"auto"`, prefer `roads_gdf` if available, else `boundary_polygon` if given, else raise; for `"roads_gdf"`, raise if not available; for `"live_osm"`, raise if no boundary.<br>6. If `effective_source == "live_osm"`: call `ox.graph_from_polygon()`, add lengths via `_has_lengths()`/`ox.distance.add_edge_lengths()` as before (unchanged logic, just renamed from "path 1").<br>7. Otherwise (`effective_source == "roads_gdf"`): call `_graph_from_geometries(roads_gdf)` — **now with endpoint snapping, see below**.<br>8. `_assign_travel_times()` (unchanged).<br>9. Set `G.graph["mode"]`, `G.graph["speed_kph"]`, and (**new**) `G.graph["source"]`.<br>10. Return. |
| **Assumptions** | Unchanged: a single fixed average speed per mode is an acceptable simplification. **New:** assumes a caller who omits `source` (relying on `"auto"`'s default) actually wants the reproducibility-favoring `roads_gdf` path whenever it's viable — this is now baked into the default behavior itself, not just documented as a recommendation. |
| **Complexity** | Unchanged for the `live_osm` path. The `roads_gdf` path's complexity is now O(N·V̄) plus a small constant-factor overhead per coordinate for the new snapping calculation (see below) — still linear, not a complexity-class change. |
| **Concurrency / race conditions** | None — unchanged. |
| **Covered by test(s)** | See [tests.md](../../tests.md) — `test_network_graph.py`, plus new: `test_graph_from_roads_source_auto_prefers_roads_gdf`, `test_graph_from_roads_source_roads_gdf_requires_data`, `test_graph_from_roads_source_live_osm_requires_boundary`, `test_graph_from_roads_invalid_source_raises`, `test_graph_from_roads_snaps_nearly_coincident_endpoints`, `test_graph_from_roads_snap_does_not_merge_distinct_junctions` (the last two directly test the endpoint-snapping fix described below, not just the `source` parameter). |

## `_has_lengths(G)` and `_assign_travel_times(G, speed_kph)`

Both **unchanged** from the previous revision — see their existing
documentation: `_has_lengths()` remains a defensive fallback check rarely
triggered in practice (OSMnx computes lengths internally already);
`_assign_travel_times()` remains a straightforward in-place unit
conversion from `length`/`speed_kph` into `travel_time_min`, with the same
`inf`-on-zero-speed guard as before.

## New in this revision: endpoint snapping in `_graph_from_geometries(roads_gdf, snap_tolerance_m=0.5)`

This is the second major addition in this module, solving a real,
previously-undocumented correctness gap in the `roads_gdf` construction
path.

**The problem, stated directly from the function's own docstring:** two
road segments that meet at "the same" real-world junction rarely share
bit-identical coordinates once they've each been through separate OSM
ways, clipping, and reprojection — a few millimeters of floating-point
drift is enough to make coordinate-tuple-keyed node identity treat them as
two *different* nodes, silently splitting the network into disconnected
components at every such junction. Since the previous revision's
`_graph_from_geometries()` used raw, unmodified coordinate tuples as node
keys, this fragmentation was a real, latent risk in every graph built via
this path — potentially degrading routing quality significantly, in a way
that wouldn't necessarily be obvious just from inspecting the graph's node
or edge counts.

**The fix:** every endpoint coordinate is snapped to a `snap_tolerance_m`
grid (default `0.5` meters — the graph is built in EPSG:32631, so this is
metres, not degrees) *before* being used as a node key:
```python
def _snap(coord):
    return (
        round(coord[0] / snap_tolerance_m) * snap_tolerance_m,
        round(coord[1] / snap_tolerance_m) * snap_tolerance_m,
    )
```
Nearly-coincident endpoints — the same real-world junction, differing only
by floating-point noise — now collapse onto the identical snapped
coordinate, and therefore the identical node, recovering the shared-node
topology a raw coordinate-tuple graph would otherwise lose.

**Why `0.5` metres specifically, and why this makes the `roads_gdf` path a
genuine substitute for `live_osm`, not just a degraded fallback:** the
docstring is explicit that 0.5m is chosen to be comfortably below realistic
OSM/extraction precision loss, while remaining well below the width of an
actual road — so it recovers genuine shared junctions without ever
accidentally merging two *distinct* junctions that happen to be close
together. This tolerance choice is precisely what upgrades this path's
status from "a fallback with known topology limitations" (the previous
framing) to "the default, reproducibility-favoring path" (this revision's
framing) — without it, defaulting to this path would mean defaulting to a
graph with unknown, silent connectivity gaps.

**Self-loop avoidance:** `if u == v: continue` — after snapping, two
originally-distinct-but-very-close coordinates within the same line
segment could, in principle, snap to the identical point; this guard skips
adding a zero-length self-loop edge for that case, rather than polluting
the graph with degenerate edges.

**`G.graph["snap_tolerance_m"]` is recorded on the graph itself** — the
same self-describing-graph pattern already used for `mode`/`speed_kph`/
`source`, so a consumer inspecting an already-built graph can tell exactly
what tolerance was used to construct it, without needing that information
passed alongside the graph object separately.

## Internal Workflow

```mermaid
flowchart TD
    A["graph_from_roads(roads_gdf, boundary_polygon, mode, speed_kph, source)"] --> B{mode in MODE_CONFIG?}
    B -- no --> C["raise ValueError"]
    B -- yes --> D{source in VALID_SOURCES?}
    D -- no --> C2["raise ValueError"]
    D -- yes --> E["effective_speed = speed_kph or MODE_CONFIG[mode].speed_kph"]
    E --> F["roads_available = roads_gdf not None and len > 0"]
    F --> G{source value?}
    G -- "auto" --> H{roads_available?}
    H -- yes --> I["effective_source = 'roads_gdf'"]
    H -- no --> J{boundary_polygon given?}
    J -- yes --> K["effective_source = 'live_osm'"]
    J -- no --> L["raise ValueError: need roads_gdf or boundary_polygon"]
    G -- "roads_gdf" --> M{roads_available?}
    M -- no --> N["raise ValueError"]
    M -- yes --> I
    G -- "live_osm" --> O{boundary_polygon given?}
    O -- no --> P["raise ValueError"]
    O -- yes --> K

    I --> Q["_graph_from_geometries(roads_gdf, snap_tolerance_m=0.5)<br/>NEW: endpoints snapped to 0.5m grid before use as node keys"]
    K --> R["ox.graph_from_polygon(boundary_polygon, network_type)"]
    R --> S{_has_lengths(G)?}
    S -- no --> T["ox.distance.add_edge_lengths(G)"]
    S -- yes --> U
    T --> U
    Q --> U["_assign_travel_times(G, effective_speed): mutate every edge in place"]
    U --> V["G.graph['mode'], ['speed_kph'], ['source'] = ... (source is NEW)"]
    V --> W["return G"]
```

## Gotchas

- **The `roads_gdf` path still cannot distinguish walk-only paths from
  vehicle roads — unchanged from before, but now the cost of the
  *default* path, not the fallback.** This is the previous revision's
  documented limitation, carried forward unchanged: `mode="walk"` and
  `mode="drive"` produce the *same underlying graph* when built via
  `roads_gdf`, differing only in the speed used for `travel_time_min`, not
  in which edges exist. Because `roads_gdf` is now the default whenever
  it's available, **this limitation now applies by default**, not only in
  a fallback scenario — a caller who wants genuine walk/drive network-type
  differentiation must explicitly pass `source="live_osm"`.
- **Path 1/Path 2 return different graph types — still true, worth
  re-stating given the renaming.** `"live_osm"` returns a directed
  `MultiDiGraph`; `"roads_gdf"` (via `_graph_from_geometries()`) returns an
  undirected `MultiGraph`. Both are still documented as `nx.MultiDiGraph`
  in the type-hinted return signature, so this mismatch remains present
  and unresolved by this revision — code relying on directed-graph-specific
  behavior (one-way street handling) downstream should be aware this
  silently disappears whenever the now-default `roads_gdf` path is used.
- **The endpoint-snapping fix genuinely changes what a `roads_gdf`-built
  graph looked like before this revision — for the better, but worth
  knowing if comparing against archived pre-revision output.** A graph
  built via this path before this fix could have more disconnected
  components (fragmented at floating-point-mismatched junctions) than the
  same input produces now. Any pre-revision cached routing results or
  isochrones built from this path should not be assumed directly
  comparable to post-revision results — the underlying graph topology
  itself changed, not just the code around it.
- **`snap_tolerance_m` is a function parameter with a sensible default
  (`0.5`), not currently threaded through `graph_from_roads()`'s own
  signature or `config.py`.** A caller needing a different snap tolerance
  for unusual data (e.g. a much coarser or much more precise source
  dataset than typical OSM extraction) would need to call
  `_graph_from_geometries()` directly rather than through the public
  `graph_from_roads()` entry point, since the public function doesn't
  currently expose this parameter.
- **The `x`/`y` node-attribute duplication in `_graph_from_geometries()`
  remains load-bearing, unchanged** — every node still carries explicit
  `x`/`y` attributes matching its (now snapped) coordinate, required for
  [`isochrones.nearest_graph_node()`](isochrones.md)'s KD-tree lookup, for
  the same reasons documented previously.

## Related

- [`config.py`](../config.md) — the source of `MODE_CONFIG` and
  `WALKING_SPEED_KPH`'s values in this revision.
- [`isochrones.py`](isochrones.md) — the primary consumer of graphs this
  module builds, including the `x`/`y` node attributes this module is
  careful to set.
- [`scoring.py`](scoring.md) — calls `graph_from_roads()` once per mode
  inside `add_access_times()`, and is the primary place `source`'s default
  behavior actually matters in practice.
- [Cross-Repo Integration](../../../cross-repo/integration.md) — the
  `roads_gdf` path's entire reproducibility argument depends on
  `lga_extractor`'s versioned, cached export being the trustworthy source
  of truth this module defaults to preferring.
