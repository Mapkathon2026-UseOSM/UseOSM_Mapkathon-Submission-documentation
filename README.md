# UseOSM Mapkathon Submission Documentation

This repository is the source for a [MkDocs Material](https://squidfunk.github.io/mkdocs-material/)
documentation site covering the two code repositories submitted for
Map<>kathon 2026 (UseOSM):

- **[`lga-osm-extractor`](https://github.com/Mapkathon2026-UseOSM/lga-osm-extractor)**
  — extracts and cleans Nigerian LGA OSM data (roads, buildings, waterways,
  land use, health facilities, schools).
- **[`akure-accessibility-dashboard`](https://github.com/Mapkathon2026-UseOSM/akure-accessibility-dashboard)**
  — the health/education accessibility analysis and Streamlit dashboard
  built on top of the extractor's output.

This repository does not contain the two projects' source code — it
contains **only documentation about that code**, built as a static site
and published via GitHub Pages at:

**https://mapkathon2026-useosm.github.io/UseOSM_Mapkathon-Submission-documentation/**

---

## Why a separate repo for documentation

The documentation covers *both* code repositories together — most
importantly the [Cross-Repo Integration](docs/cross-repo/integration.md)
page, which explains the data contract between them and doesn't
naturally belong inside either individual repo. Keeping documentation
here also means the two code repositories' own READMEs can stay focused
on "how to run this" rather than growing into the much longer
function-by-function reference this site provides.

## Repository structure

```
UseOSM_Mapkathon-Submission-documentation/
├── mkdocs.yml                  # Site configuration: nav, theme, plugins
├── requirements.txt            # Python deps needed to build the site
├── docs/                       # All actual documentation content (Markdown)
│   ├── index.md                # Site landing page
│   ├── stylesheets/
│   │   └── extra.css           # Custom navy/cyan theme (light + dark mode)
│   ├── lga-osm-extractor/
│   │   ├── overview.md         # Purpose, architecture, design philosophy
│   │   ├── data-flow.md        # How data changes shape at each stage
│   │   ├── end-to-end.md       # Full startup-to-output walkthrough
│   │   ├── tests.md            # What each test actually verifies
│   │   ├── app.md               # Streamlit UI (app.py)
│   │   ├── cli.md               # Command-line interface (cli.py)
│   │   └── modules/            # One page per source file in lga_extractor/
│   │       ├── boundary.md
│   │       ├── layers.md
│   │       ├── clean.md
│   │       ├── export.md
│   │       ├── visualize.md
│   │       ├── logging_utils.md
│   │       └── pipeline.md
│   ├── akure-accessibility-dashboard/
│   │   ├── overview.md
│   │   ├── data-flow.md
│   │   ├── end-to-end.md
│   │   ├── tests.md
│   │   ├── dashboard-app.md    # Streamlit runtime (dashboard/app.py)
│   │   └── modules/            # One page per source file in akure_access/
│   │       ├── accessibility/
│   │       │   ├── network_graph.md
│   │       │   ├── isochrones.md
│   │       │   └── scoring.md
│   │       ├── completeness/
│   │       │   └── grid_check.md
│   │       ├── visualization/
│   │       │   └── static_maps.md
│   │       └── insights.md
│   ├── cross-repo/
│   │   └── integration.md      # The data contract between the two repos,
│   │                            # anchored on test_cross_repo_integration.py
│   └── reference/
│       ├── glossary.md         # Recurring terms across both repos
│       └── known-issues.md     # Every documented bug fix / design trade-off,
│                                # indexed and linked back to its source page
└── .github/
    └── workflows/
        └── deploy-docs.yml     # Builds and publishes to GitHub Pages on push to main
```

## How each page is structured

Every module page (one per source file) follows the same shape, so you
always know where to look for a given kind of information:

| Section | What it covers |
|---|---|
| **Purpose** | Why this file exists and what stage of the pipeline it belongs to |
| **Dependencies** | What this module imports from, and what depends on it |
| **Functions & Classes** | One entry per function/class: what it does, why it was written that way, inputs/outputs, internal logic, assumptions, complexity, and — where relevant — race conditions |
| **Covered by test(s)** | (inside each function's entry) The specific test function(s) that exercise it, or an honest note if none exist |
| **Internal Workflow** | A Mermaid diagram of the module's control flow |
| **Gotchas** | Things that look fine individually but are easy to misuse or misread in combination |

The **Overview**, **Data Flow**, and **End-to-End Walkthrough** pages at
each repo's root work at a higher level than individual modules — they're
the right starting point if you want the big picture before diving into
a specific file.

## Building the site locally

```bash
pip install -r requirements.txt
mkdocs serve
```

This starts a local server (usually `http://127.0.0.1:8000`) with live
reload — edits to any file under `docs/` refresh the browser
automatically.

To produce a static build without serving it (useful for checking the
build succeeds before pushing):

```bash
mkdocs build --strict
```

`--strict` turns broken internal links, missing files referenced in
`mkdocs.yml`'s nav, and other structural problems into build failures
rather than silent warnings — this is exactly what the deploy workflow
runs, so if `mkdocs build --strict` passes locally, the deploy will too.

## Deployment

Pushing to `main` with changes under `docs/` or `mkdocs.yml` triggers
`.github/workflows/deploy-docs.yml`, which builds the site with
`mkdocs build --strict` and publishes the result to GitHub Pages via the
`actions/deploy-pages` action. No manual deploy step is needed — the
repo's **Settings → Pages → Source** should be set to **GitHub Actions**
(not "Deploy from a branch") for this to work.

## Theme

The site uses a custom color scheme (`docs/stylesheets/extra.css`)
matching the submission author's portfolio branding — a deep navy
background with a cyan/teal accent in dark mode, and the same navy
header with a white reading surface in light mode. The light/dark toggle
is in the top-right corner of every page.

## Source of truth

Every technical claim on this site is grounded in a direct reading of
the two repositories' actual source code and test files at the time each
page was written — not from general knowledge about what such a project
"probably" does. Where a page states a design rationale, an incident, or
a bug fix, it's because that reasoning or history is present in the
source (a docstring, a comment, a test name, or an observable code
pattern), not inferred or invented. If either upstream repository
changes in a way that makes a page here inaccurate, that page needs a
manual update — this site does not regenerate itself from source
automatically.
