# network_graph.py

!!! info "Source"
    `akure_access/accessibility/network_graph.py` (156 lines)

## Purpose

Builds a routable `NetworkX` graph for a given travel mode (walk, okada,
drive) — the foundation everything else in the `accessibility` package sits
on top of. Every isochrone computation and nearest-facility distance/time
lookup in [`isochrones.py`](isochrones.md) operates on a graph produced
here.

This is the first module in `akure-accessibility-dashboard` to consume
output from the upstream `lga-osm-extractor` repo — its `roads_gdf`
parameter expects exactly the cleaned roads layer `lga_extractor.clean`
produces (see [Cross-Repo Integration](../../../cross-repo/integration.md)).

## Dependencies

- **Imports:** `geopandas`, `networkx`, `osmnx`.
- **Imported by:** [`isochrones.py`](isochrones.md) (which builds on graphs
  from this module), `dashboard/app.py`.

## Functions & Classes

### `WALKING_SPEED_KPH` (module-level constant, `5.0`)

Kept explicitly for backward compatibility — the module comment notes this
exists only so any code still importing `WALKING_SPEED_KPH` directly (rather
than reading `MODE_CONFIG["walk"]["speed_kph"]`) continues to work. The
*actual* speed used by `graph_from_roads()` is read from `MODE_CONFIG`, not
this constant.

### `MODE_CONFIG` (module-level dict)

```python
{
    "walk":  {"network_type": "walk",  "speed_kph": 5.0},
    "okada": {"network_type": "drive", "speed_kph": 25.0},
    "drive": {"network_type": "drive", "speed_kph": 35.0},
}
```

The one entry worth explaining is `"okada"` (commercial motorcycle taxis, a
common transport mode in Nigeria): it's modeled on OSM's `"drive"` network
type, **not** a distinct motorcycle network — OSM has no separate motorcycle
network classification, and in practice okadas use the same road network
cars do. What differentiates it is a lower assumed average speed (25 km/h
vs. 35 km/h for cars), reflecting local traffic conditions and okada riding
behavior (weaving through traffic, using narrower roads a car might avoid).
The module is explicit that these speed figures are approximations, not
measured data — see [Known Issues](../../../reference/known-issues.md) for
the broader methodology-limitations context, and any caller can override
the assumption via `graph_from_roads()`'s `speed_kph` parameter.

### `graph_from_roads(roads_gdf, boundary_polygon=None, mode="walk", speed_kph=None)`

| | |
|---|---|
| **What it does** | Builds a routable `networkx.MultiDiGraph` for the requested mode, with `length` (meters) and `travel_time_min` (minutes) attributes on every edge, via one of two construction paths depending on whether a boundary polygon is supplied. |
| **Why written this way** | The two-path design directly trades off robustness against consistency-with-extracted-data. **Path 1** (boundary polygon given, the recommended default): delegates entirely to `osmnx.graph_from_polygon()`, which OSMnx handles correctly — proper topology cleanup, connected-component handling, standard OSM network typing. **Path 2** (`roads_gdf` only, no boundary): builds a graph directly from the exact roads geometries `lga_extractor` already extracted and versioned on disk, useful specifically when guaranteeing the routing graph matches precisely what was extracted matters more than routing correctness — e.g. reproducibility, or auditing what a specific archived extraction actually contains. The documented cost of path 2: it can't distinguish walk-only paths from vehicle roads (that distinction lives in OSM's tag-based network typing, which the extractor's plain roads layer doesn't preserve), so `mode="walk"` and `mode="drive"` produce the *same underlying graph* when built this way, differing only in the assumed speed used to compute `travel_time_min` — not in which edges exist. |
| **Inputs** | `roads_gdf: GeoDataFrame` (cleaned roads layer in EPSG:32631, as produced by `lga_extractor.clean.clean_layers()` — required for path 2, optional/unused-for-topology in path 1); `boundary_polygon` (Shapely polygon in EPSG:4326, optional — presence of this argument is what selects path 1 vs. path 2); `mode: str`, default `"walk"` (one of `"walk"`/`"okada"`/`"drive"`); `speed_kph: float`, optional (overrides `MODE_CONFIG`'s default speed for the chosen mode). |
| **Outputs** | `networkx.MultiDiGraph` with `length`/`travel_time_min` edge attributes, plus graph-level attributes `mode` and `speed_kph` recording how it was built. |
| **Internal workflow** | 1. Validate `mode` is a known key in `MODE_CONFIG`, raise `ValueError` listing valid options if not.<br>2. Resolve `effective_speed`: the explicit `speed_kph` argument if given, otherwise `MODE_CONFIG[mode]["speed_kph"]`.<br>3. **Path 1** (`boundary_polygon is not None`): call `ox.graph_from_polygon(boundary_polygon, network_type=config["network_type"])`; then call `_has_lengths(G)` to check whether edge lengths are already present — if not, call `ox.distance.add_edge_lengths(G)` to add them. (See Gotchas below on why this check exists at all.)<br>4. **Path 2** (no boundary): call `_graph_from_geometries(roads_gdf)`.<br>5. Regardless of path: call `_assign_travel_times(G, effective_speed)` to populate `travel_time_min` on every edge.<br>6. Set `G.graph["mode"]` and `G.graph["speed_kph"]`.<br>7. Return `G`. |
| **Assumptions** | Assumes a single fixed average speed per mode is an acceptable simplification — no accounting for road-type-specific speed variation (e.g. an unpaved residential street vs. a paved arterial road, which likely have genuinely different real-world okada speeds), congestion, time-of-day, or road surface quality. This is a deliberate, documented simplification, not an oversight — see the module's own comment pointing to the methodology limitations. |
| **Complexity** | Path 1: dominated by `ox.graph_from_polygon()`'s own complexity (an Overpass query plus OSMnx's internal graph construction — not controlled by this function). Path 2: O(N·V̄) where N = number of road features, V̄ = average vertices per line — one pass building edges from consecutive coordinate pairs per geometry. `_assign_travel_times()` (called in both paths) is O(E) in edge count. |
| **Concurrency / race conditions** | None — no threading or shared mutable state; this function is not called concurrently anywhere in the codebase. |
| **Covered by test(s)** | See [tests.md](../../tests.md) — `test_network_graph.py`. |

### `_has_lengths(G)`

| | |
|---|---|
| **What it does** | Returns `True` if every edge in `G` already has a `"length"` attribute. |
| **Why written this way** | This exists as a defensive fallback, not a normally-triggered code path. The inline comment is explicit about this: `osmnx.graph_from_polygon()` already computes edge lengths internally in both OSMnx 1.x and 2.x, so `_has_lengths(G)` normally short-circuits the fallback (`ox.distance.add_edge_lengths(G)` is skipped). It's kept in case that internal OSMnx behavior ever changes in a future version — cheap insurance rather than a load-bearing code path today. |
| **Inputs** | `G: MultiDiGraph`. |
| **Outputs** | `bool`. |
| **Complexity** | O(E) — one pass over all edges. |
| **Concurrency / race conditions** | None. |
| **Covered by test(s)** | No dedicated test — this is a defensive fallback check not normally triggered in practice (OSMnx already computes edge lengths internally), so there's no realistic test scenario that exercises the `False` branch without deliberately constructing a malformed graph. |

### `_assign_travel_times(G, speed_kph)`

| | |
|---|---|
| **What it does** | Mutates `G` in place, adding a `travel_time_min` attribute to every edge, computed from that edge's `length` and the given speed. |
| **Why written this way** | A simple unit conversion (meters and km/h into minutes) applied uniformly across every edge — the function itself carries no per-road-type logic, keeping the mode/speed assumption entirely in `MODE_CONFIG` (or the caller's override) as the single place that assumption lives. |
| **Inputs** | `G: MultiDiGraph` (mutated in place, not returned — the `-> None` return type in the signature reflects this); `speed_kph: float`. |
| **Outputs** | `None` — side effect only. |
| **Internal workflow** | 1. Convert `speed_kph` to meters-per-minute: `(speed_kph * 1000) / 60`.<br>2. For every edge, read `length` (defaulting to `0` if somehow missing — a defensive default, since by the time this is called in `graph_from_roads()`, lengths are guaranteed present via `_has_lengths()`/`_graph_from_geometries()`); compute `travel_time_min = length_m / speed_m_per_min`.<br>3. Guard against division by zero: if `speed_m_per_min` is falsy (i.e. `speed_kph == 0`), assign `float("inf")` instead of raising `ZeroDivisionError` — a zero-speed mode is nonsensical but shouldn't crash the pipeline; an infinite travel time correctly signals "unreachable at this speed" to any downstream consumer. |
| **Assumptions** | Assumes `length` is already in meters (true for both construction paths — OSMnx's `add_edge_lengths()` and `_graph_from_geometries()`'s own Euclidean distance calculation both produce meters, since the graph is built/queried in a projected CRS). |
| **Complexity** | O(E) — one pass over all edges, each a constant-time computation. |
| **Concurrency / race conditions** | Mutates the graph object in place. If this were ever called concurrently on the *same* graph object from multiple threads, edge attribute writes could race — not a concern under current usage (always called synchronously, once, immediately after graph construction), but worth flagging for anyone considering parallelizing graph-building across LGAs sharing state. |
| **Covered by test(s)** | See [tests.md](../../tests.md) — exercised indirectly by `test_graph_from_roads_geometry_fallback_assigns_travel_times` and `test_graph_from_roads_speed_override`, both of which check this function's output (`travel_time_min` edge attributes) rather than calling it directly. |

### `_graph_from_geometries(roads_gdf)`

| | |
|---|---|
| **What it does** | Builds a simple `networkx.MultiGraph` directly from a cleaned roads `GeoDataFrame`'s line geometries, using raw coordinate tuples as node identifiers — the fallback construction path used when no boundary polygon is available. |
| **Why written this way — and why the `x`/`y` node attributes matter more than they look.** | This is explicitly documented as *less robust* than OSMnx's own graph construction (no topology cleanup — e.g. two roads that visually cross but don't share an OSM-tagged intersection node won't be connected in this graph, whereas OSMnx's Overpass-based construction handles this correctly), intended only as a consistency fallback, not a routing-quality-first choice. The critical detail: **every node is given explicit `x`/`y` attributes matching its coordinate tuple**, not just relying on the node key itself (which is *also* the coordinate tuple). This duplication exists for [`isochrones.nearest_graph_node()`](isochrones.md)'s benefit — that function reads `node['x']`/`node['y']` attributes specifically, not the node key. **A precise note on which nearest-node lookup is actually involved here**: `isochrones.nearest_graph_node()` does *not* call `osmnx.distance.nearest_nodes()` — it deliberately replaces it with a custom `scipy.spatial.cKDTree` built directly over these `x`/`y` attributes, specifically *because* OSMnx's own nearest-node function assumes OSMnx's internal graph conventions (integer node IDs) and doesn't work reliably against this fallback graph's coordinate-tuple node IDs. This module's own docstring mentions `osmnx.distance.nearest_nodes()` in explaining *why* the `x`/`y` attributes matter, which can read as if that function is what's actually called — it isn't; see [`isochrones.md`](isochrones.md#nearest_graph_nodeg-point) for the real mechanism. Without explicitly setting these `x`/`y` attributes, the KD-tree-based lookup against a fallback-path graph would silently fail — not raise an error, just silently return nothing useful — and every downstream distance/time computation would come back as `inf`, with no obvious signal pointing at why. This is the kind of failure mode that's easy to reintroduce accidentally in a future edit and hard to notice without dedicated test coverage. |
| **Inputs** | `roads_gdf: GeoDataFrame` (cleaned roads layer). |
| **Outputs** | `networkx.MultiGraph` (undirected — note this differs from path 1's `MultiDiGraph`, see Gotchas), with a graph-level `crs` attribute (taken from `roads_gdf.crs` if set, else defaulting to `"EPSG:32631"`), and `x`/`y` node attributes as described above. |
| **Internal workflow** | 1. Create an empty `MultiGraph`, tag its `crs` attribute.<br>2. For each row in `roads_gdf`: skip if geometry is `None` or empty; extract the geometry's coordinate list; for each consecutive coordinate pair `(u, v)` along the line, compute Euclidean distance as `length`, add both endpoints as nodes (with `x`/`y` attributes), add an edge between them carrying `length` and the row's `osmid` (via `.get()`, tolerant of a missing column). |
| **Assumptions** | Assumes coordinate tuples are a sufficient and stable node identity — two road segments sharing an exact coordinate (a shared vertex) are correctly treated as connected at that point; two segments that are geometrically close but don't share an *exact* coordinate (a common real-world digitizing imprecision in crowd-sourced OSM data) are **not** connected, silently producing a more fragmented, less-routable graph than the true road network. This is the direct, unstated cost of skipping OSMnx's own topology handling. |
| **Complexity** | O(N·V̄) where N = number of road features, V̄ = average vertices per line geometry — one pass building consecutive-pair edges per line. |
| **Concurrency / race conditions** | None — sequential row iteration, no shared mutable state beyond the graph being built. |
| **Covered by test(s)** | See [tests.md](../../tests.md) — this is one of the more important functions here to have direct test coverage on, given the documented `x`/`y` attribute fragility described above. |

## Internal Workflow

```mermaid
flowchart TD
    A["graph_from_roads(roads_gdf, boundary_polygon, mode, speed_kph)"] --> B{mode in MODE_CONFIG?}
    B -- no --> C["raise ValueError"]
    B -- yes --> D["effective_speed = speed_kph or MODE_CONFIG[mode].speed_kph"]
    D --> E{boundary_polygon given?}
    E -- yes --> F["ox.graph_from_polygon(boundary_polygon, network_type)"]
    F --> G{_has_lengths(G)?}
    G -- no --> H["ox.distance.add_edge_lengths(G)"]
    G -- yes --> I
    H --> I
    E -- no --> J["_graph_from_geometries(roads_gdf):<br/>coordinate-tuple nodes with x/y attrs,<br/>pairwise edges with Euclidean length"]
    J --> I["_assign_travel_times(G, effective_speed): mutate every edge in place"]
    I --> K["G.graph['mode'] = mode, G.graph['speed_kph'] = effective_speed"]
    K --> L["return G"]
```

## Gotchas

- **Path 1 and Path 2 return different graph types.** `ox.graph_from_polygon()`
  returns a directed `MultiDiGraph` (edges have direction, reflecting
  one-way streets and OSM's directional tagging); `_graph_from_geometries()`
  returns an undirected `MultiGraph`. Both are documented as the function's
  return type (`nx.MultiDiGraph` in the signature), but path 2's actual
  runtime type doesn't match that annotation — worth being careful about if
  writing code that relies on directed-graph-specific behavior (e.g.
  one-way street handling) downstream, since that behavior silently
  disappears when the fallback path is used.
- **`mode="walk"` and `mode="drive"` are identical graphs in Path 2.** As
  documented in the function's own docstring, this fallback path has no
  awareness of OSM's walk-vs-drive network-type distinction — it's the same
  set of edges regardless of `mode`, differing only in the speed used to
  compute `travel_time_min`. This is very different behavior from Path 1,
  where `mode` actually changes which roads are included in the graph at
  all (e.g. a highway with no pedestrian access is excluded from a `"walk"`
  network-type query but included in a `"drive"` one). Any analysis relying
  on Path 2 should be aware mode-based road *inclusion* differences are not
  modeled, only speed differences are.
- **The `x`/`y` node-attribute duplication in `_graph_from_geometries()` is
  load-bearing, not defensive.** Removing it wouldn't just be a minor
  inefficiency — it would silently break every nearest-node lookup and
  isochrone computation that runs against a fallback-path graph, with the
  failure surfacing only as unexplained `inf` distances downstream, not as
  an error at the point of the actual mistake.
