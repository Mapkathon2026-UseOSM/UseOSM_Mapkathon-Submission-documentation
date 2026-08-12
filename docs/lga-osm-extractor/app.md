# Streamlit App — app.py

!!! info "Source"
    `app.py` (289 lines)

## Purpose

A Streamlit web UI wrapping `pipeline.extract_lga()`: a user types an LGA
name (and optionally a state), clicks "Extract OSM Data," previews every
extracted layer on an interactive Leafmap map with toggleable layers, and
downloads everything as a zip. This is the no-code, no-GIS-software path
into the extractor — everything it does is a thin presentation layer over
the same `extract_lga()` function `cli.py` calls.

Run with `streamlit run app.py`.

## Dependencies

- **Imports:** `streamlit`, `leafmap.foliumap` (for the interactive
  preview map), `os`, `zipfile`, `io` (for the download-as-zip feature);
  `extract_lga` and `BoundaryResolutionError` from the `lga_extractor`
  package's top-level `__init__.py`.
- **Imports from:** `pipeline.py` (via the package's `__init__.py`
  re-export), indirectly depends on everything `extract_lga()` itself
  depends on.

## Page Structure

1. **Page config + custom CSS** — sets page title/icon/wide layout, injects
   a small CSS block (global font-size bump, hero title/subtitle styling, a
   colored callout box style, and orange-accented form/download buttons).
2. **Hero section + "About this tool" explanation** — static Markdown
   explaining the three-step flow, why extraction can take a few minutes
   (live Overpass queries, no cached OSM data), and what each of the six
   layers contains.
3. **Input form** (`st.form("extract_form")`) — two text inputs (`lga_name`,
   `state_name`) side by side, one primary submit button.
4. **`_cached_extract()` definition** — see below.
5. **`LAYER_STYLES`** — a module-level dict of Folium styling
   (color/weight/fillOpacity) per layer name, deliberately kept visually
   consistent with `akure-accessibility-dashboard`'s own palette
   conventions, so a given color means roughly the same thing to anyone
   who's seen both tools.
6. **On submit** — validates the LGA name isn't blank, runs the (cached)
   extraction inside a spinner, and on success: shows warnings (if any) in
   an expander, builds and displays the preview map, offers a zip download
   of the output directory.
7. **Footer** — a GitHub link back to the source repository.

## Functions

### `_cached_extract(lga_name, state_name)`

| | |
|---|---|
| **What it does** | A thin wrapper around `extract_lga()`, decorated with `@st.cache_data(show_spinner=False)`. |
| **Why written this way** | Extraction is genuinely slow — it hits OpenStreetMap's live Overpass API for six separate layers, and larger/denser LGAs (especially `buildings`) can take one to five minutes. `st.cache_data` means re-running the *exact same* `(lga_name, state_name)` combination again in the same session — or from a **different user entirely**, since Streamlit's `@st.cache_data` cache is shared across all sessions of a running app by default, not scoped per-user — returns instantly instead of re-querying Overpass from scratch. This works correctly with `extract_lga()`'s own behavior: its `output_dir` defaults to a path deterministically derived from `lga_name`, so the files already written to disk on the first run remain valid and present for a cached return — the cache doesn't need to re-materialize any files, only skip re-computation of the returned summary dict. `show_spinner=False` is set because the app already shows its own custom spinner message around the call site (step 6 above), avoiding a redundant/generic second spinner from Streamlit's cache decorator itself. |
| **Inputs** | `lga_name: str`, `state_name` (str or `None`). |
| **Outputs** | Whatever `extract_lga()` returns (see [`pipeline.md`](modules/pipeline.md)) — cached by Streamlit keyed on the exact input arguments. |
| **Internal workflow** | Single line: `return extract_lga(lga_name=lga_name, state_name=state_name)`. All the actual work is Streamlit's caching machinery plus `extract_lga()` itself. |
| **Assumptions** | Assumes `extract_lga()`'s default `manual_boundary_path=None` and `strict=False` are the right choices for this UI — the app never exposes these as user-facing options, unlike `cli.py`, which does expose both. A user needing a manual boundary or strict-mode behavior currently has no path to that through the Streamlit UI. |
| **Complexity** | O(1) in this function's own logic; wall-clock cost is entirely `extract_lga()`'s (dominated by Overpass query latency across six layers, see [`layers.md`](modules/layers.md)). |
| **Concurrency / race conditions** | **This is the one place in the whole repository where Streamlit's shared-cache model introduces a genuine cross-user consideration.** Because `@st.cache_data`'s cache is shared across all sessions by default, two different users requesting the same LGA/state at nearly the same time could both trigger the underlying `extract_lga()` call if neither result is cached yet (Streamlit doesn't lock/dedupe concurrent misses for the same key across sessions) — both would independently query Overpass and independently write to the same deterministic `output_dir`, a benign but wasteful race (see `export.py`'s and `logging_utils.py`'s own gotchas about concurrent writes to the same output directory not being guarded against). Once either completes, subsequent requests for that same key are served from cache. This isn't a correctness bug (final state is fine, both would produce equivalent output), but it is a real, if narrow, concurrency consideration unique to the Streamlit deployment. |
| **Covered by test(s)** | See [tests.md](tests.md) — note `_cached_extract()` itself, being a decorated Streamlit function, isn't directly unit-testable in the same way as the core pipeline functions; `test_extraction.py` tests `extract_lga()` and its dependencies directly instead. |

## Layer Rendering Logic (the `if submitted:` block)

The most detailed piece of logic in this file isn't `_cached_extract()` —
it's how the preview map decides which layer to zoom the map to, since a
naive approach has a real bug the code explicitly works around:

- Layers are sorted so `"roads"` is attempted first (roads typically span
  the full LGA boundary, making it a good layer to frame the initial map
  view around), with every other layer following in whatever order they
  appear in `result["exported"]`.
- **The zoom-to-layer bug this avoids:** an earlier, simpler approach would
  tie `zoom_to_layer=True` to a fixed list position (e.g. "the first item").
  But if that first item in sorted order (`"roads"`) happens to have no
  file on disk — a real, valid case: an LGA where roads legitimately
  extracted as empty, or was skipped — indexing by fixed position would
  mean **no layer ever gets zoomed to**, since the loop simply moves to the
  next index without the map having zoomed to anything yet, leaving the map
  at Leaflet's global default view even though other layers loaded
  successfully underneath, just invisible at that zoom level. The actual
  code instead tracks a running `zoomed_yet: bool` flag across the loop,
  and passes `zoom_to_layer=(not zoomed_yet)` to `m.add_geojson()` for each
  layer — guaranteeing whichever layer is genuinely added *first* (in
  practice, regardless of which earlier layers in sorted order turned out
  to be empty) gets the zoom.
- **A second defensive layer:** each `m.add_geojson()` call is wrapped in a
  `try/except (IndexError, KeyError, ValueError)`. The comment explains this
  shouldn't normally trigger — `export_layers()` already filters out
  genuinely empty layers before writing any file at all (they go into
  `exported["_skipped"]` instead of being written) — but it exists as a
  backstop against a file that *exists* on disk but is malformed or
  unexpectedly empty, e.g. from a partial/interrupted write during a
  long-running extraction (a real risk given extractions can take several
  minutes). Without this, one bad file would crash the whole preview map
  and silently drop every layer added after it in the loop, rather than
  just that one layer being reported as un-previewable while the rest (and
  the download) continue to work.

## Gotchas

- **The Streamlit cache is shared across users, not per-session.** As noted
  above, this is a deliberate and reasonable choice for this app's use case
  (avoid re-querying Overpass for a popular LGA), but it does mean one
  user's extraction can "warm the cache" for another user requesting the
  same LGA/state, which is worth knowing if this is ever deployed somewhere
  privacy/isolation between users matters more than it does for a public
  demo tool.
- **`manual_boundary_path` and `strict` are not exposed in the UI**, even
  though `extract_lga()` supports both — only `cli.py` currently exposes
  the full parameter surface.
- **The download zip is built entirely in memory** (`io.BytesIO()`), by
  walking `output_dir` on disk — this means the zip always reflects
  whatever is currently on disk at download time, not a snapshot taken at
  extraction time; for a cached result, this is the same files from the
  original run.
