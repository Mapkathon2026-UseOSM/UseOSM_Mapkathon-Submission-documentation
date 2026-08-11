# Cross-Repo Integration

_TODO: the data contract between lga-osm-extractor and akure-accessibility-dashboard._

Anchor this page to `tests/test_cross_repo_integration.py` in the dashboard repo —
that test is the executable specification of this contract.

## What the Extractor Produces

_TODO: exact file formats, folder structure, CRS, expected layer names/schemas from `export_layers()`._

## What the Dashboard Expects

_TODO: what `akure_access` / `dashboard/app.py` assumes about input file structure and schema (see `load_data()`)._

## Schema Table

| Field | Type | Produced by | Consumed by |
|---|---|---|---|
| _TODO_ | _TODO_ | _TODO_ | _TODO_ |

## Known Failure Modes

_TODO: what happens if the extractor's output doesn't match what the dashboard expects (e.g. the Polygon-vs-Point issue — link to known-issues.md)._
