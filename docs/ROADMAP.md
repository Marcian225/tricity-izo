# ROADMAP — Isometric Tricity

Development in three stages, each ending with a working release:

| Stage | Scope | Goal |
|---|---|---|
| **1** | Gdańsk + Gdynia | Full pipeline, look and feel, transit, landmarks. First public release |
| **2** | + Sopot | Joining the cities into one Tricity, consistent SKM and tile borders |
| **3** | + Rumia, Reda, Wejherowo | The Small Tricity, mostly adding regions and transit |

Principle: **stages 2 and 3 should mostly be a matter of adding files in `regions/` and data**, not rewriting code. If the engine has to be touched, that is a sign stage 1 lacked generalisation. Record such cases in `DECISIONS.md`.

Time estimates are rough and assume several agents working in parallel under human supervision. According to its author, the first Warsaw version took two days, but it then kept growing for several more days, mainly through fixes and additions. Plan for a few days for a basic version and a few weeks for a full one.

---

## Stage 1 — Gdańsk + Gdynia

### M0. Data reconnaissance (0.5–1 day) — **blocks everything else**

Goal: know where we stand before any code is written.

- Check **LoD2** building coverage from GUGiK for Gdańsk and Gdynia (Geoportal 3D, download a sample). If missing: evaluate LoD1 + LiDAR as a substitute and describe what we lose (roof shapes).
- Check the CRS and format of every source (CityGML, terrain model, orthophoto, trees).
- Download a sample of each source for one tile and confirm that they line up.
- Check the availability and quality of GTFS: ZTM Gdańsk, ZKM Gdynia, SKM, PKM, Intercity (route shapes, identifier consistency).
- Estimate data sizes and required disk space.
- Output: `docs/data-audit.md` with a table of source / coverage / format / licence / risk, plus a stack decision recorded in `DECISIONS.md`.

**Done when:** for every layer in CLAUDE.md it is known whether it is available, in what format and with what gaps.

### M1. Spike: two tiles end to end (1–2 days)

Goal: test the whole approach on a small sample before scaling it.

- Tile A: **the Main Town in Gdańsk** (dense development, water, bridges, landmarks).
- Tile B: **a steep area in Gdynia** (e.g. around Kamienna Góra or Witomino), to test terrain from the start.
- Pipeline: data → binary tile → browser, with simple rendering (plain volumes, no pixel-art yet).
- Decide and record the tile format (`docs/tile-format.md`).

**Done when:** both tiles show in the browser with correct terrain, buildings, streets and water, and the build time per tile is known.

### M2. Pipeline for whole regions (2–4 days)

Runs in parallel with M3 once the tile format is frozen.

- Configs `regions/gdansk.yaml` and `regions/gdynia.yaml`, a `build --region` command.
- Buildings: LoD2 where available, filled in from footprints and heights where it is not. Roof colour from the orthophoto.
- Streets from OSM tags (width from lane count, dual carriageways, cycle paths), parks, water, street lamps, car parks.
- Terrain from the terrain model, trees from GUGiK models (streamed).
- Tile manifest and check scripts.
- Data buffer at region borders (important for Stage 2).

**Done when:** all tiles of both cities build without errors, and the build is reproducible.

### M3. Rendering engine and pixel-art style (2–3 days)

Runs in parallel with M2.

- Half-resolution rendering, palette, depth-based outlines, pixel snapping.
- Tile streaming (priority from the centre of the view), isometric camera, rotation, zoom, touch.
- Sun depending on date and time, shadows, night (windows, street lamps), seasons (snow, leaves).
- Instancing for trees and street lamps.

**Done when:** the centre shows within the time budget on desktop and phone, and the style is consistent across all tiles.

### M4. Bridges, viaducts, tunnels and steep terrain (2–3 days)

Depends on M2. Usually the hardest visual element.

- Bridge solver: a deck spanning between its ends, minimum clearance above road, rail and water.
- Flyovers, railway viaducts, tunnels (portals, ramps), stairs and embankments on steep streets.
- `tests/scan_bridges.py` script as a quality gate.

**Done when:** the control-spot list in `docs/qa-spots.md` passes, and the number of suspicious bridges is low and known.

### M5. Water, sea and beaches (1–2 days)

- Gulf of Gdańsk, beaches, piers, the Motława, the Martwa Wisła, ports as water.
- Water animation, frozen water bodies in winter.
- Decide whether level variation makes sense (see the note in CLAUDE.md).

### M6. Transit (3–5 days)

Can run in parallel with M5.

- Merge the GTFS feeds (ZTM Gdańsk, ZKM Gdynia, SKM, PKM, Intercity) into one model, with stop and route mapping onto the geometry.
- Trams in Gdańsk, trolleybuses in Gdynia, SKM/PKM/Intercity trains, tunnel entrances.
- Clicking a vehicle opens a card with the line, direction, next stop, operator and a note that the position is simulated.
- Parked cars following a daily pattern.

**Done when:** 3 randomly chosen lines match the timetable at the check times.

### M7. Ports, airport, shipyards (2–3 days)

- Port of Gdańsk and port of Gdynia: quays, containers, cranes, ferries.
- Gdańsk airport: runways, taxiways, terminal, simulated air traffic, runway lighting from the AIP.
- Shipyard area and Westerplatte.

### M8. Landmarks and info cards (ongoing, parallel)

- First a priority list in `landmarks/PRIORITY.md`, sorted by visibility and significance. Examples: Długi Targ and the Crane, St Mary's Church, the European Solidarity Centre, Stadion Energa, Westerplatte, Dar Pomorza and Błyskawica, Skwer Kościuszki and Gdynia's modernist centre.
- Landmark format: a file with the model, date, author and source of reference photos, and the card text. Cards without verified facts do not go into the release.
- Churches: a separate style for all churches in the data (brick or stone walls, windows, towers) so they do not look like office blocks.

### M9. Release 1.0 (1–2 days)

- Sources and attribution panel, an "about" page, view screenshots, SEO and social media previews.
- Static hosting, correct MIME types for JS files (a typical problem on shared hosting).
- A channel for bug reports from users. In Warsaw many fixes came from exactly this.

---

## Stage 2 — Sopot (the full Tricity)

The scope is small but important, because Sopot **joins** the two cities.

- `regions/sopot.yaml` and tile build. Verify that tiles on the borders of Gdańsk, Sopot and Gdynia stitch together without seams.
- Continuity of SKM and shared transit lines along the whole axis, including Sopot being served by lines from both operators.
- Landmarks: the Pier, the Grand Hotel, the Crooked House, the Forest Opera, the Hippodrome.
- Scaling review: loading time, manifest size, memory, after adding the whole axis of cities.

**Done when:** the camera can fly from Gdynia to Gdańsk with no holes or data jumps.

## Stage 3 — Small Tricity (Rumia, Reda, Wejherowo)

- `regions/*.yaml` files and tile build. Check the availability of 3D trees and LoD2 in these counties **[TO CONFIRM]**.
- Transit: SKM extension, local lines (confirm available GTFS, e.g. Wejherowo).
- Local landmarks and seasonal events.
- If stages 1 and 2 were done correctly, this is mostly data work.

---

## Parallel tracks and dependencies

```
M0 ──► M1 ──┬─► M2 (pipeline) ──┬─► M4 (bridges) ──┐
            │                    ├─► M5 (water) ────┤
            └─► M3 (render) ─────┤                  ├─► M9 (release 1.0)
                                 ├─► M6 (transit) ──┤
                                 ├─► M7 (ports) ────┤
                                 └─► M8 (landmarks, ongoing)
```

Example split between agents after M1:

| Agent | Area | Directories |
|---|---|---|
| A | Building data, terrain, trees | `pipeline/parse/`, `pipeline/build/` |
| B | Rendering, pixel-art, sun, seasons | `web/src/render/`, `web/src/sim/sun*` |
| C | Bridges and steep terrain | `pipeline/build/bridges*`, `tests/` |
| D | Transit | `pipeline/parse/gtfs*`, `web/src/sim/transit*` |
| E | Landmarks | `landmarks/` |

Each agent in a separate worktree. Merging follows the rules in `CLAUDE.md`.

## Risks and how we limit them

| Risk | Effect | Response |
|---|---|---|
| No LoD2 for Pomerania | Flat or simplified roofs | M0 detects it. Fallback: LoD1 + LiDAR, roof shapes from OSM where available |
| Outdated data | Errors on new development | List of known gaps in `data-audit.md`, manual fixes from the `qa-spots.md` list |
| Steep terrain and bridges | Floating or sunken objects | M4 with a quality gate, screenshot-based tests |
| GTFS quality and consistency (3 operators) | Wrong routes, duplicate stops | Audit in M0, manual verification of three lines per stage |
| Performance on phones | Slow start, freezing | Budgets from M1, measurements after every major addition |
| Scope creep (landmarks, events) | Delayed release | Priority list, release 1.0 without the "nice little details" |
| Claude Code usage limits with many agents | Interruptions in work | Start with 2–3 agents, increase after M1 |
| Landmark photos and models | Licence problems | Only sources with a clear licence, author and licence recorded with each object |
