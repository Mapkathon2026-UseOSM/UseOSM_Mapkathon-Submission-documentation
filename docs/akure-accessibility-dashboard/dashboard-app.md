# Dashboard App — dashboard/app.py

!!! info "Source"
    `dashboard/app.py` (742 lines — the largest single file across both
    repositories)

## Purpose

The interactive Streamlit dashboard: the project's primary public-facing
deliverable, presenting the health/education accessibility analysis for
Akure North and Akure South LGAs. Unlike `lga-osm-extractor`'s `app.py`
(which triggers a live extraction pipeline), this dashboard is entirely a
**consumer of pre-computed outputs** — it never runs any of
`accessibility.scoring`, `accessibility.isochrones`, or
`completeness.grid_check` itself. It reads already-scored GeoJSON files
(produced by the project's analysis notebooks 01–04) and already-generated
static image exports (from `visualization.static_maps`'s notebook-driven
run), and presents both interactively.

Run with `streamlit run dashboard/app.py` from the repo root.

## Dependencies

- **Imports:** `json`, `os`, `sys`, `geopandas`, `pandas`, `streamlit`,
  `leafmap.foliumap`; `describe_interactive_view` from
  `akure_access.insights` (see the `sys.path` handling below for why this
  import needs special care).
- **Reads from disk (not imported as code):** `data/processed/{lga}/
  grid_access_scored.geojson` (the scored grid, output of notebooks 01–04);
  `visuals/{lga}/` and `visuals/{lga}/web/` (static images and
  `captions.json`, output of `visualization.static_maps.generate_all_static_outputs()`).

## Visual Design System

The file opens with a substantial, deliberately-documented CSS block —
worth summarizing since it's a real, considered design decision, not
boilerplate Streamlit theming:

| Element | Choice | Stated reasoning |
|---|---|---|
| Palette | `#141625` background (adire indigo-black), `#C4622D` primary accent (laterite road/soil red-orange), `#4C9A8C` secondary accent (vegetation teal), `#E8B84B` highlight gold | Grounded in the subject matter rather than a generic dashboard theme — accent colors reference Akure's actual laterite-red roads/soil and Southwest Nigeria's adire indigo-dyeing tradition. |
| Type | Space Grotesk (headings), IBM Plex Sans (body), IBM Plex Mono (data figures) | IBM Plex Mono specifically for data/metrics "ties to the coordinate/data nature of a GIS tool." |
| Signature motif | Concentric rings (◎) | Tied to the accessibility/catchment concept at the conceptual core of the study — used as the page icon and every section divider. |

Two specific CSS decisions are worth noting for anyone maintaining this
file, since both are explained via inline comments addressing a subtlety
that isn't obvious from the CSS alone:

- **The global `html { font-size: 118%; }` bump is deliberately not
  applied to the hero title/subtitle.** Since Streamlit's own built-in
  widgets, labels, radio buttons, and dataframes all size themselves in
  `rem` units (relative to the root font size), bumping the root value is
  what makes "everything else" larger app-wide. The hero title/subtitle are
  set in fixed `px` specifically so they *don't* also get multiplied by
  this root increase — keeping their size bump independent and
  intentionally smaller.
- **`[data-testid="stWidgetLabel"] p` is targeted app-wide, but
  `st.caption()` is handled differently.** The former is safe to target
  globally because every widget label in the app happens to sit above the
  interactive map (verified, per the inline comment, that none appear
  below it) — so a global rule can't accidentally affect anything in the
  lower half of the page. `st.caption()`, by contrast, is used both above
  *and* below the map (the static-maps section reuses it for image
  captions), so it can't be safely resized globally without also affecting
  those — the two explanatory captions above the map are instead rendered
  via a custom `.pre-map-caption` CSS class specifically so they can be
  sized independently of `st.caption()`'s other, unrelated uses further
  down the page.

## The `sys.path` Fix

Before any `akure_access` import, the file does this:

```python
_REPO_ROOT = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
if _REPO_ROOT not in sys.path:
    sys.path.insert(0, _REPO_ROOT)
```

This is a documented, real deployment fix, not defensive boilerplate. The
inline comment explains: Streamlit Cloud installs from
`dashboard/requirements.txt`, not `pyproject.toml` — and that
`requirements.txt` has never actually `pip install`ed the local
`akure_access` package (confirmed, per the comment, by checking the full
list of installed packages in actual deploy logs). `akure_access` was only
ever importable on Streamlit Cloud "by accident of working directory," not
because it was properly installed as a package. Rather than continuing to
depend on that accident of working directory continuing to hold,
`_REPO_ROOT` is computed explicitly (this file's grandparent directory:
`dashboard/app.py` → `dashboard/` → repo root) and inserted into `sys.path`
directly, making the import work correctly regardless of how or where the
app is actually launched from.

## Page Structure and Functions

### Section: Page config + CSS (lines ~47–244)

`st.set_page_config()`, then the CSS block described above.

### `section_divider(title)`

A small helper rendering the concentric-ring section divider (the ◎ motif)
in place of Streamlit's default `st.subheader()` styling — used
consistently at every major section break throughout the rest of the file,
keeping the signature visual motif consistent app-wide rather than each
section header being styled ad hoc.

### `load_data()`

| | |
|---|---|
| **What it does** | Loads both LGAs' scored GeoJSON files, tagging each with an `"lga"` column; any LGA whose file can't be loaded (missing, corrupt) gets `None` in the returned dict rather than raising and crashing the whole app. |
| **Why written this way** | Decorated with `@st.cache_data` (no arguments — this function takes none, so the cache key is trivial/constant, meaning this only actually runs once per app process lifetime, not once per user session or per interaction). The broad `except Exception: frames[lga] = None` per-LGA (not a single try/except around the whole loop) means one LGA's data being unavailable (e.g. only Akure North's notebooks have been run so far) doesn't prevent the other LGA from loading successfully — the app can run in a genuinely partial state, which the code immediately downstream explicitly checks for and surfaces to the user (see below) rather than silently proceeding as if both LGAs were present. |
| **Inputs** | None (reads from the module-level `DATA_PATHS` dict). |
| **Outputs** | `dict[str, Optional[GeoDataFrame]]` — `{"Akure North": gdf_or_None, "Akure South": gdf_or_None}`. |
| **Complexity** | O(1) relative to app runtime — file I/O for two GeoJSON files, cached after the first call. |
| **Concurrency / race conditions** | `@st.cache_data`'s shared-across-sessions caching (the same consideration documented in `lga_extractor.app.py`'s `_cached_extract()`) applies here too, though with lower practical stakes — this function has no arguments and no side effects beyond reading static files that don't change during the app's runtime, so concurrent cache population by multiple users would, at worst, mean the same read happening more than once before the cache settles, not any inconsistent-state risk. |
| **Covered by test(s)** | See [tests.md](tests.md) — like `lga_extractor.app.py`'s Streamlit-specific functions, direct unit testing of a `@st.cache_data`-decorated function is limited; core scoring/data-shape logic is tested via `accessibility`/`completeness` module tests instead. |

### Section: Hero header + missing-data handling (lines ~294–320)

Renders the hero banner, calls `load_data()`, then explicitly checks for
missing LGAs: `st.warning()` lists any LGA with no data found, naming which
notebooks need to be run. If **no** LGA loaded successfully at all,
`st.stop()` halts the entire script here — the rest of the app (which
assumes at least one LGA's data exists) never executes, rather than
proceeding into a cascade of downstream errors against empty/missing data.

### Section: Control widgets (lines ~322–362)

Three columns of controls: **Study area** (a selectbox — `"Both (compare)"`
is only offered as an option if *both* LGAs actually loaded successfully;
otherwise the available LGA list is just whichever single LGA is present),
**Access view** (`"Combined"` / `"Health only"` / `"Education only"` —
directly feeds `insights.describe_interactive_view()`'s `view_choice`
parameter), **Transport mode**. A second row adds **Basemap** choice and a
**colorblind-safe palette** checkbox.

The default-selection logic for Study area is worth noting: `default_lga_index`
picks `"Both (compare)"` as the default *only if it's actually present* in
the options list (i.e. both LGAs loaded), falling back to index `0`
(whichever single LGA is available) otherwise — avoiding an `IndexError`
or a nonsensical default that references an option that doesn't exist for
a partial-data run.

### Section: Score-column / palette configuration (lines ~366–393)

`MODE_LABELS`, `DEFICIT_COLORS` (mirroring `visualization.static_maps`'s
own palette constants — standard traffic-light vs. Okabe-Ito
colorblind-safe), `DEFICIT_LABELS`, `CONTINUOUS_CMAP` (`"YlOrRd"` standard
vs. `"viridis"` colorblind-safe) are all defined here, independently of
`static_maps.py`'s equivalent constants — see Gotchas below on this
duplication.

### `score_column(view, mode)`

A small pure function: given the current `view` and `mode` selector state,
returns the correct grid column name to visualize — `health_time_min_{mode}`,
`education_time_min_{mode}`, or `{mode}_access_deficit_score` for
`"Combined"`. This is the dashboard's equivalent of `insights.py`'s own
internal column-name resolution logic, kept here as its own tiny function
specifically because `render_map()` needs the resolved column name for
plotting, not just for caption text.

### `render_map(gdf, view, mode, basemap="CartoDB.Positron", colorblind_safe=False)`

| | |
|---|---|
| **What it does** | Builds and displays one Leafmap interactive map for one LGA, styled according to the current view/mode/basemap/colorblind-safe selections. |
| **Why written this way** | The `"Combined"` view and the two continuous (health-only/education-only) views use genuinely different Leafmap rendering schemes, matching the same categorical-vs-continuous distinction documented in `visualization.static_maps.py`: `"Combined"` uses `scheme="UserDefined"` with explicit `bins=[0, 1, 2]` and explicit colors/labels — deliberately explicit rather than an automatic numeric quantile legend, because a raw numeric legend would show score values (0, 1, 2) with no explanation of what each number actually means; the continuous views use `scheme="Quantiles"` with `k=6` bins and a continuous colormap instead, appropriate for a genuinely continuous minutes-to-facility value. |
| **Inputs** | `gdf: GeoDataFrame` (one LGA's scored grid); `view`, `mode: str`; `basemap: str`, default `"CartoDB.Positron"`; `colorblind_safe: bool`, default `False`. |
| **Outputs** | `None` — renders directly into the Streamlit page via `m.to_streamlit(height=600)`. |
| **Internal workflow** | 1. Create the Leafmap map, add the chosen basemap.<br>2. Resolve the column via `score_column()`; filter to settled cells (`building_count > 0`).<br>3. If settled data exists and the column is present: branch on `view == "Combined"` to choose the categorical vs. continuous rendering scheme, calling `m.add_data()` with the appropriate `scheme`/`colors`/`cmap`/`legend_title` for that branch.<br>4. If the column genuinely doesn't exist (e.g. this mode wasn't included in the notebook run that produced the current data), show `st.info()` with actionable guidance (which notebook, which mode) rather than a raw exception or a blank map.<br>5. Render via `m.to_streamlit()`. |
| **Assumptions** | Assumes `building_count > 0` is the correct settlement filter for map display — consistent with `scoring.py`'s own convention, not `completeness.grid_check.py`'s stricter 3-building threshold (see that module's own gotcha about the two different thresholds coexisting in the codebase). |
| **Complexity** | O(N) where N = settled cell count, for the filter and rendering — Leafmap/Folium's own rendering cost dominates for large cell counts. |
| **Covered by test(s)** | See [tests.md](tests.md). |

### Section: Access map (lines ~436–485)

Calls `render_map()` for either one LGA, or — if `"Both (compare)"` is
selected — **side-by-side columns**, not tabs. The inline comment is
explicit about why: "'compare' implies seeing both LGAs at once; tabs would
only let you switch between them one at a time, which isn't actually a
comparison view." Each map is immediately followed by its
`describe_interactive_view()` caption, recomputed fresh on every script
rerun (Streamlit reruns the whole script on every widget interaction) —
the inline comment notes this guarantees the caption shown always matches
exactly what the map is currently displaying, "rather than a fixed caption
written once that could silently drift out of sync with the actual
selection," directly echoing `insights.py`'s own foundational design
principle in the specific context of a live, interactive UI.

### Section: Most underserved settlements (lines ~487–520)

A ranked table of the top-15 most-underserved individual grid cells for
the current mode (across both LGAs if `"Both (compare)"` is selected),
built via `pd.concat()` of the relevant frame(s), filtered to
`deficit_col > 0`, sorted descending. Column names are mapped to
human-readable headers (`friendly_names` dict) and values rounded to 1
decimal before display — a small but real usability detail (raw column
names like `health_time_min_walk` and five-decimal floats would be
technically correct but harder to scan at a glance).

### Section: Findings summary (lines ~522–602)

**This section carries a documented regression fix worth reproducing in
full**, since it's a genuine example of a subtle, easy-to-reintroduce UI
bug:

> "Match the same LGA-selection behavior as 'Most underserved settlements'
> above: respect `lga_choice` rather than always combining every LGA.
> Before this fix, this section silently ignored `lga_choice` and always
> showed a combined North+South statistic, so switching the LGA tab
> visibly updated the map and table above but left these percentage cards
> unchanged — a confusing inconsistency for anyone comparing the two
> sections side by side."

The current code builds `settled_frames` conditionally on `lga_choice`
(exactly matching the pattern used for `frames_to_rank` in the section
above), rather than unconditionally concatenating both LGAs — this is the
actual fix. The section then computes:

- **Per-mode metric cards**: `pct_any`/`pct_both` computed live for
  whichever modes have score columns present in the currently-scoped data,
  rendered as styled metric cards (one per available mode).
- **A walking-vs-fastest-mode gap callout**: if more than one mode is
  available, computes the percentage-point gap between walking's
  underserved rate and the *best* (lowest) non-walking mode's rate, and
  surfaces it as a headline finding — "Walking-only analysis would
  overstate underserved communities by roughly N percentage points..." —
  directly connecting back to the project's own stated headline finding
  about why modeling multiple modes matters (see [repository
  overview](overview.md)).
- **A completeness cross-check callout**: computed live, conditional on
  both `health_completeness_flag` and `walk_access_deficit_score` being
  present — among *walking-underserved* cells specifically (not all
  settled cells), what percentage also carry a possible OSM data gap for
  health, and separately for education. This is the dashboard's most
  direct, quantified expression of the project's core problem statement:
  explicitly warning the reader, with a live-computed number, that "some
  portion of the underserved findings above may reflect incomplete OSM
  tagging rather than a confirmed absence of nearby facilities."

### Section: Static / publication maps (lines ~604–718)

Displays the pre-generated static images from `visuals/{lga}/`, produced
separately (offline, via the analysis notebook calling
`generate_all_static_outputs()`) rather than generated live by the
dashboard itself.

- **Web-tier preference with fallback**: for each LGA, checks whether a
  `visuals/{lga}/web/` subfolder exists (the lower-resolution tier from
  `generate_all_static_outputs()`'s dual-save mechanism) and prefers it if
  present — "loads faster," per the inline comment — falling back to the
  top-level (print-resolution) folder for compatibility with older runs
  generated before the `web/` subfolder convention existed.
- **Caption lookup**: reads `captions.json` from the top-level LGA folder
  (always written there by `generate_all_static_outputs()`, regardless of
  which tier is being displayed) — the inline comment clarifies this
  lookup works correctly either way since `captions.json`'s keys are plain
  filenames shared by both tiers (same figure, same filename, different
  folder/resolution only).
- **`_categorize(filename)`**: a small local closure classifying each
  filename into one of six display categories (`"Access deficit"`,
  `"Health access time"`, `"Education access time"`, `"Data completeness"`,
  `"Mode comparison"`, `"Other"`) via substring matching on the filename —
  grouping related figures together (across LGAs and modes) into
  expandable sections, "rather than one long alphabetical list."
- **Fixed 3-column image grid**: the inline comment documents a second
  concrete regression fix here — using `min(3, len(cat_files))` for the
  column count previously meant a category with only *one* image (e.g.
  "Mode comparison," which only ever produces a single chart) got a single
  full-width column, stretching that one image to fill the entire page
  width while every other category's images stayed a normal, consistent
  size. The current code always allocates a fixed `st.columns(3)`
  regardless of image count, producing visually consistent image sizing
  across every category.
- **Caption fallback for pre-`captions.json` runs**: if a specific filename
  has no entry in the loaded `captions.json` (or no `captions.json` exists
  at all for that LGA), falls back to a readable label derived from the
  filename itself (underscore-to-space conversion) — explicit backward
  compatibility with static exports generated before the caption-writing
  feature existed.
- **Bulk download**: if `visuals/akure_access_static_maps.zip` exists on
  disk, offers it via `st.download_button()`.

### Section: Footer (lines ~720–741)

A GitHub link back to the source repository, with an inline SVG GitHub
mark icon (avoiding an external image dependency for a small UI element).

## Gotchas

- **Palette/colormap constants are defined independently in two places**:
  `dashboard/app.py`'s own `DEFICIT_COLORS`/`CONTINUOUS_CMAP` and
  `visualization/static_maps.py`'s `DEFICIT_PALETTES`. Both are documented
  (in their respective files) as intentionally using the *same actual
  color values*, specifically so the live dashboard and static exports
  never visually disagree — but the values themselves are duplicated in
  source, not imported from one shared location. A future palette change
  applied to only one of the two files would silently break that
  consistency, with no test or code-level check that would catch the
  drift.
- **`load_data()`'s file paths are hardcoded module-level constants**
  (`DATA_PATHS`), not derived from any shared configuration with the
  analysis notebooks that actually produce those files — the path
  convention (`data/processed/{lga_slug}/grid_access_scored.geojson`) is
  an implicit contract between the notebooks and this dashboard, similar
  in kind to the implicit filename contract documented between
  `lga_extractor.export.py` and `lga_extractor.visualize.py`.
- **The dashboard has zero test coverage** — being almost entirely
  Streamlit UI/layout code operating on already-tested data (the
  `accessibility`/`completeness` modules' own test suites cover the
  underlying scoring logic this dashboard only *displays*), this is a
  reasonable trade-off, but the two documented regression fixes in this
  file (the Findings Summary LGA-scope bug, and the single-image-stretch
  layout bug) are both exactly the class of UI-state bug that direct test
  coverage — even lightweight `streamlit.testing` framework smoke tests —
  could have caught before manual discovery.
