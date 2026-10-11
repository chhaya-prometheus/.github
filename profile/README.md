# Chhaya blog, Part 1: Data pipeline and AWS (Pratham)

Oct 11, 2026 · @Pratham

## Why a slow pipeline

Chhaya answers one planner's question for Delhi, "where should the next tree go?", from data that changes monthly, not by the second, so we built the data side as a slow, boring batch job that writes files to S3 and let everything else read those files.

The Heat and Water track asks for tools that change something for the people who live through heatwaves. A planner needs a ranked map, not a live feed. So the system splits in two. A monthly pipeline (my part) fills 1,945 equal-sized hexagons over Delhi, about 0.7 km² each, with vegetation, heat, population and dust. A separate API (Hitesh) and map (Aditya) only ever read the finished files from Amazon S3. The frontend never talks to a satellite, and a failed data source can never take the map down.

This part covers the pipeline and the AWS side: how the grid and ward names work, how the score is built, the seven bugs that cost us the most time, and how it all runs on AWS.

## Hex grid and ward names

Every cell is an H3 resolution-8 hexagon, and every Delhi hexagon carries the name of the ward it sits in, so a planner can read the map by place name and not by cell ID.

Why hexagons: every cell has the same area and the same six neighbours, so scores compare fairly across the city, and the cell ID (`h3_index`) is a stable key that every source can join on. Delhi at resolution 8 gives 1,945 cells.

The first version had a real usability hole: a hex has no name. Looking at the map, nobody could tell which part of Delhi a cell was. We fixed it with DataMeet's Delhi Municipal Spatial Data ward file (CC BY-SA 2.5 IN). Each hex is joined to the ward it overlaps most, in a metric projection (UTM 43N). Hexes with no overlap fall back to the nearest ward within 500 m, and the run logs how many hexes stayed unassigned and which wards received none. The ward name and a stable ward ID become columns of the cell file. The data source does not state which ward delimitation it uses, so I treat it as possibly older than the current one and say so in the README.

**One pipeline, many cities.** A city is a small YAML file: a name, a slug, a boundary file, an H3 resolution and a data mode. `build_city` turns any boundary into a hex grid. Delhi is `real` and has the ward file. Jaipur, Lucknow, Hyderabad and Chandigarh use boundary-only grids with deterministic synthetic values, and the app labels them "Demo data (synthetic)". The synthetic cities exist to show that adding a city is a config change, not a rewrite.

## Data sources and the priority score

Each hex gets one number, `priority`, a weighted blend of four signals. It is a proxy for where shade would help most, not a measurement of anything.

| Signal | Column | Source | Weight |
| --- | --- | --- | --- |
| Heat | `lst_c` | Land surface temperature, satellite-derived | 0.40 |
| People exposed | `population` | Gridded population | 0.25 |
| Missing greenery | `ndvi` (inverted) | Sentinel-2 NDVI | 0.20 |
| Dust | `dust_idx` | Dust index | 0.15 |

Each signal is min-max scaled to 0..1 across the city, nulls take the median, and the blend gives `priority`. `priority_rank` orders the cells, 1 = most urgent, so the map can colour by rank. Built-up fraction and school counts from OpenStreetMap ride along as context columns in the panel.

**\[Pratham: confirm before publishing\]** NDVI from Sentinel-2 is the source I built and verified end to end. State here exactly which of the other sources (temperature, population, OSM, dust) ran on real data for Delhi at submission, and which are still synthetic stubs. Judges read this honestly or not at all.

The weights are a judgement call, not a result of fitting. They are in one place in the config so a planner can change them.

## Seven problems and how we fixed them

The bug that taught me most was not a bug in code: 533 of 1,945 hexes had no vegetation value, and the cause was satellite geometry, not clouds.

1. **H3 4.x broke every tutorial.** `polyfill` became `geo_to_cells`, and `h3_to_geo_boundary` became `cell_to_boundary` with no `geo_json` option. It returns (lat, lng) but Shapely wants (lng, lat), so I swap by hand. Lesson: on any H3 error, check `h3.__version__` first.
2. **stackstac: "Cannot pick a common CRS".** Earth Search v1 stores the projection as `proj:code` ("EPSG:32643"); stackstac 0.5.1 only reads the old `proj:epsg`. Fix: pass `epsg=32643` explicitly. Delhi is UTM zone 43N, which is the "43" in tile names like `S2B_43RFM`.
3. **NDVI ran from minus infinity to infinity.** Sentinel-2's empty border is 0, so the division gave 0/0. My first fix masked before dividing with `|`, which still let a positive red band over a zero near-infrared band through. The right fix computes NDVI on the raw arrays, then masks after dividing with `&` (both bands positive), and the zonal mean skips NaNs.
4. **533 hexes with null NDVI, a footprint problem.** My three cleanest scenes shared one latitude band, so their footprints overlapped and recovered only 15 hexes. Four tiles cover Delhi (43RFM, 43RGM, 43RFN, 43RGN) and none covers it alone; the city sits where four tiles meet. The fix picks one scene per tile and skips tiles already used. Valid pixels went from 44% to 79% and nulls from 533 to 0. When a "clouds are the problem" fix buys 3%, the problem is geometry.
5. **Sources that returned geometry broke the merge.** I got "multiple geometry columns" on write and `geometry_x` / `geometry_y` after merging. Rule now: a source returns only `[h3_index, <its columns>]`; geometry lives in the grid fixture alone.
6. **`rasterstats` shape error on cached rasters.** Passing an array plus an affine transform failed with "Invalid out\_shape, must be 2D" on a cached GeoTIFF. Passing the file path lets the array and its transform come from the same place.
7. **One failed source wiped a good column.** The first export overwrote `cells.parquet`, so a failed run replaced good values with nulls. Now a two-tier merge keeps the previous file from S3: new values where a source succeeded, old values where it failed. `pipeline_status.json` records fresh or stale per source, so the map can say "Temperature: last updated 28 days ago, refresh failed".

## Config layering and the data contract

Two small decisions let three people work in parallel without breaking each other: layered config, and one JSON schema that every side reads.

**Config.** `base.yaml` holds shared defaults, `dev.yaml` and `prod.yaml` override per environment, per-city YAML files describe each city, and `.env` holds secrets. `config.py` deep-merges them, resolves paths against the repo root and caches the result. No other module reads `os.environ` or opens a YAML file. The same code runs on a laptop and on a server, and only `CHHAYA_ENV` changes: date ranges, cloud thresholds and whether anything uploads to S3. Production uploads need an explicit `confirm_prod_upload` flag, because a bad run must never overwrite good data by accident.

**Contract.** Each source returns exactly `[h3_index, <its columns>]`. The orchestrator merges them onto the grid, computes `priority` and `priority_rank`, and validates every row against `contract/cell.schema.json` before anything is uploaded. `pipeline_status.schema.json` describes the freshness file: when it ran, which environment, and per source whether it succeeded, how many nulls it left and any error. Hitesh's API and Aditya's map both build against the same schema, so adding a column means adding a field, never renaming one.
