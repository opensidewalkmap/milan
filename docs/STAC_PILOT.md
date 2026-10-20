# Milan processed-data STAC pilot

## Scope and license decision (2026-10-20)

Publish only the `processed` family under ODbL-1.0. This covers sidewalks,
crossings, kerbs, aggregate other footways and its five subcategories. The
software repository's MIT license does not license these datasets.

Source review at Milan commit `d04bfa8d65063dbedd414acd5c9b83090c2fc2cc`:

- `data/updates/acquisition.json` records Overpass acquisition on
  `2026-10-13T20:55:20Z`. Overpass supplies OpenStreetMap data.
- `metadata/processed/*.metadata.json` identifies OpenStreetMap and `data/raw`
  as the source of the processed products.
- Core `getting_data.py` obtains OSM features via incremental OHSOME,
  OSMnx/Overpass, or the regional OSM PBF fallback. `filtering_adapting_data.py`
  reads raw OSM layers and versioning records, filters and normalizes them,
  calculates derived geometry attributes, and writes the processed products.
  It does not read DEM data or merge Data Hub project geometries.
- Milan enables Copernicus GLO-30/GLO-90 enrichment for routing. That separate
  pipeline does not supply the processed STAC family. Routing, hazard products,
  basemaps, raw extracts and completeness roads are outside this pilot.

OpenStreetMap's official licensing page identifies ODbL and requires attribution
and share-alike for adapted databases:
https://www.openstreetmap.org/copyright

License text: https://opendatacommons.org/licenses/odbl/1-0/

Attribution: **© OpenStreetMap contributors**. The exporter retains this in each
Item and links the license from each Collection. The reviewed processed family
is distributed as an OSM-derived database under ODbL-1.0. Adding another source
requires reviewing this decision. Routing remains unselected until its DEM and
other input terms are separately established.

## Reproducible pre-deployment validation

Validated the above Milan revision with core
`74cbda522dbbc833c7fceabe1c16d83299e72c43` and the proposed configuration:

```bash
python /path/to/oswm_codebase/metadata/stac_generation.py --node-root .
python /path/to/oswm_codebase/metadata/stac_generation.py --node-root . --validate-only
```

- 9 Collections, 9 Items, 19 STAC 1.1.0 documents; zero validation errors.
- Offline schema validation, local relationships, SHA-256 and byte sizes pass.
- PySTAC traverses all 9 Items. Repeated generation is byte-identical.
- Item generation time is `2026-10-18T13:35:37Z`, matching Data Pre-Processing;
  acquisition time remains `2026-10-13T20:55:20Z`.
- Existing tile reports do not establish matching source checksums, so all
  visual PMTiles assets are correctly omitted.
- All 18 linked public assets (9 GeoParquet and 9 metadata JSON files) returned
  HTTP 200 and `Access-Control-Allow-Origin: *` when requested with an external
  Origin. Their downloaded SHA-256 and byte sizes match the generated Items.

These checks establish pre-deployment validity; they do not establish that the
catalog itself has been published.

## Production deployment and acceptance

Merge the configuration, then run the existing `data_daily_updating` workflow
on `main` with `force_regeneration=false`. The reusable thin-node builder
checks out core ephemerally, regenerates metadata and STAC even on a no-change
daily cycle, stages `stac/`, and dispatches `pages.yml` after committing outputs.
Do not copy the local pilot files over potentially newer production outputs.

After the build and Pages deployment complete, fetch:
https://opensidewalkmap.github.io/milan/stac/catalog.json

Require HTTP 200, STAC 1.1.0, valid child/Item relationships, processed products
only, and successful traversal of all Items. Download every linked asset and
compare SHA-256 and byte size with its Item; check cross-origin GET responses
on the catalog, Collections, Items and assets for `Access-Control-Allow-Origin`.
Repeat against the newly deployed bytes, since pre-deployment results do not
validate later regenerated products. Record the build and Pages run URLs.

The Pages push path filter currently omits `stac/**`; the reusable builder's
explicit Pages dispatch is therefore part of the publication procedure.
Leave global catalog integration for a separate change after acceptance.

To withdraw the catalog, set `STAC_ENABLED = False` and run the daily builder;
it prunes this exporter's records and deploys the resulting publication.
