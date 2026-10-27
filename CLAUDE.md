# CLAUDE.md — Isometric Tricity

An isometric pixel-art map of the Tricity (Gdańsk, Gdynia, Sopot), drawn live in the browser from open data.
Developed in stages: **Stage 1** Gdańsk + Gdynia → **Stage 2** Sopot (the full Tricity) → **Stage 3** Rumia, Reda, Wejherowo (the "Small Tricity").
Work plan: `docs/ROADMAP.md`. Read it before starting any larger task.

> Items marked **[TO CONFIRM]** are assumptions that must be verified against real data. Do not treat them as facts. Once they are confirmed or disproved, update this file.

## Overriding principles

1. **No image-generating models.** The picture is 3D geometry rendered in three.js, followed by pixel-art post-processing. Everything must run locally and for free.
2. **We do not invent details that are not in the data.** No made-up shop signs, brand logos, lettering on buildings, or people. When we do not know something, we draw it neutrally or not at all. Estimated positions (e.g. wrecks, simulated vehicles) are marked as approximate in the info cards.
3. **Everything is driven by data and region configuration.** No city names, bounding boxes, or TERYT codes hard-coded in the source. Adding a new region means adding a file in `regions/`, not changing code (see below).
4. **The pipeline is reproducible.** Raw data is not in the repo. Every processing step can be run from scratch with a single command and gives identical output for identical inputs.
5. **We judge the result, not the report.** A visual task is finished only when you have seen a browser screenshot and decided it looks good. "It compiles" is not enough.
6. **Attribution and licences are part of the product**, not an afterthought (see Data and licences).

## Stack **[TO CONFIRM during M0]**

- **ETL:** Python 3.12+, `geopandas`, `shapely`, `pyproj`, `rasterio`, `numpy`, `lxml` (CityGML), `laspy` if LiDAR is needed. Dependencies only inside a `venv`/`uv` environment, never global.
- **Tile format:** custom binary format + Brotli. Specification in `docs/tile-format.md` (keep it in sync with the code).
- **Frontend:** TypeScript, Vite, three.js. No UI frameworks unless the side panel genuinely needs one.
- **Visual testing:** Playwright (headless Chromium), screenshots saved to `artifacts/screens/`.
- **Hosting:** static files. No backend in version 1.

## Coordinate system and grid

- Internally a single metric coordinate system: **EPSG:2180 (PL-1992)**. Verify the CRS of every source before combining it (GUGiK, OSM and GTFS use different systems). Convert only at the pipeline boundaries.
- Tile grid: **1 km × 1 km, aligned to 1000 m in EPSG:2180**, tile identifier = `{x_km}_{y_km}`. The grid is **global for the project**, not per region, so new regions add tiles without changing existing ones.
- Units throughout the code: **metres**, with Y as the height axis in the scene. Heights are relative to the terrain model (NMT), not sea level, unless explicitly stated otherwise.

## Regions (staged development)

```
regions/
  gdansk.yaml        # Stage 1
  gdynia.yaml        # Stage 1
  sopot.yaml         # Stage 2
  rumia.yaml reda.yaml wejherowo.yaml   # Stage 3
```

A region file contains: the boundary (polygon, or bbox plus buffer), the list of sources and their versions, the list of transit feeds (GTFS), the list of landmarks and events, and airport/port settings where applicable. The pipeline accepts `--region gdynia` and produces only that region's tiles. Tiles on the border between two regions must be built identically regardless of which region the build was started from (use a data buffer around the boundary).

Data versioning: `manifest.json` lists the tiles, their hashes and the pipeline version. A new region must never invalidate tiles of existing regions.

## Repository layout

```
regions/        region configurations
pipeline/       ETL (Python): fetch/ parse/ build/ export/
web/            application (TypeScript + three.js)
  src/render/   scene, materials, pixel-art post-processing
  src/world/    tile loading, streaming, camera
  src/sim/      sun, seasons, transit, animals
  src/ui/       panels, landmark cards
landmarks/      hand-made models and card data (one file per object)
docs/           ROADMAP.md, tile-format.md, data-audit.md, DECISIONS.md
tests/          pipeline tests and check scripts (e.g. bridge scan)
data/           (gitignored) raw data and cache
artifacts/      (gitignored) screenshots, reports
```

## Data and licences

| Layer | Source | Notes |
|---|---|---|
| Streets, 2D buildings, water, parks, street lamps, names | OpenStreetMap | ODbL, attribution mandatory |
| 3D buildings (LoD2, LoD1 as a supplement) | GUGiK, geoportal.gov.pl | **LoD2 coverage for Pomerania [TO CONFIRM in M0]**. Missing buildings: footprints from BDOT10k/OSM + height from LiDAR |
| Terrain | GUGiK NMT 1 m | Under bridges it contains bare ground, so a bridge solver is needed |
| Roof colour | GUGiK orthophoto | Sample the colour; do not paste the imagery |
| Trees | GUGiK 3D tree models | Available for Gdańsk, Gdynia and Sopot. Rumia, Reda, Wejherowo **[TO CONFIRM]** |
| Transit | GTFS: ZTM Gdańsk, ZKM Gdynia, SKM, PKM, Intercity | Separate feeds; identifiers must be merged. Vehicle positions are **simulated from the timetable, not GPS**, and are labelled as such |
| Weather | Open-Meteo | CC BY 4.0; other dates are simulated |

Rules:
- Process large files (trees, LiDAR, orthophoto) **as streams and delete them after use**. Do not keep multi-gigabyte dumps in `data/` without a reason.
- Record every source in `data/SOURCES.md` with the download date and version. Add each new source to the attribution panel in the UI within the same task.
- GUGiK data can be outdated (in Warsaw part of the scans date from 2012). New buildings may be wrong. Record known gaps in `docs/data-audit.md` (e.g. new development in Gdańsk and Gdynia) and do not pretend they are not there.
- Reference photos for landmarks: only those with a licence that allows use (e.g. Wikimedia Commons). Record the author and licence in the landmark file. Do not copy photos into the repository or onto the site.
- We do not copy other people's code, models or assets from other isometric maps. Conceptual inspiration is fine, but we write our own code.

## Pixel-art style specification (change only deliberately)

- Render the scene at **half resolution** and upscale without smoothing (nearest neighbour).
- Limited palette, with fixed colours per material (wall, roof, water, grass, asphalt, cycle path, etc.) defined in a single file, `web/src/render/palette.ts`.
- Dark outlines where depth jumps (edge detection on depth/normals).
- Pixel positions snapped to the grid so the image does not "swim" when the camera moves.
- Camera: isometric view, rotation, zoom. The `G` key returns to the classic angle.

## Visual rules specific to the Tricity

- **The sea and beaches** are a separate water layer (not a river). Sea level has little seasonal variation, so we do not build a flood slider modelled on the Vistula unless the data justifies it.
- **The terrain is steep** (the moraine plateau above Gdynia, Sopot and Oliwa) and **low-lying in the Żuławy delta**. Roads, buildings and stairs must agree with the terrain model. Check screenshots for "floating" and "sunken" objects.
- **Bridges, viaducts, flyovers and tunnels** (e.g. the tunnel under the Martwa Wisła) need a solver. The rule: a deck spanning between its ends, minimum clearance above road, rail and water, no sagging. The script `tests/scan_bridges.py` counts suspicious bridges, and that count must not grow between commits.
- **Trolleybuses in Gdynia** and **trams in Gdańsk** are distinguishing features. We do not draw moving buses and cars (they would obscure the trams). Cars appear only as parked, following a daily pattern.

## Rules for agents

1. **One agent, one worktree, one area** (e.g. `pipeline/parse/buildings`, `web/src/sim/transit`). Do not edit files outside your area without agreement.
2. **Never revert or overwrite anyone else's changes**, not even briefly "to compare a screenshot". Use a separate checkout for comparisons.
3. Merge to `main` only after: tests, a browser screenshot, and a short description (what changed, on which tiles it was checked).
4. **For every visual change**, take screenshots from at least 2 places of different character (e.g. dense development + steep terrain), day and night, and save them in `artifacts/screens/`.
5. Finish a task with a report in this form: what was done, what was checked visually, what does **not** work or is uncertain. Do not write "done" if you have not looked at it.
6. Record architectural decisions in `docs/DECISIONS.md` (date, decision, reason, alternatives).
7. When the data does not match the assumptions in this file, **stop and report it** rather than quietly working around the problem.

## Performance (mobile matters)

- Load only the tiles visible on screen; the rest lazily and with priority from the centre of the view.
- Budgets to watch (starting targets **[TO BE SET after first measurements]**): time to first image of the centre on fast 4G, size of a single tile, number of draw calls.
- Use instancing for trees, street lamps and parked cars. Do not create a separate mesh per object.

## Commands **[fill in after M1]**

```
make setup                          # venv, dependencies, npm install
make fetch REGION=gdynia            # download sources
make build REGION=gdynia            # tiles into web/public/data/
make dev                            # development server
make test                           # pipeline tests and bridge scan
make shots                          # Playwright screenshots to artifacts/screens/
```

## Definition of done for a region

- All tiles of the region built and present in the manifest, with no failed tiles.
- No visible holes in the terrain, buildings floating in the air, or bridges sunk into the ground on the check screenshots (list of spots in `docs/qa-spots.md`).
- The region's transit works according to the timetable (sample: 3 lines checked manually against the timetable).
- The attribution panel contains every source used.
- Loading time of the region's centre is within budget.
