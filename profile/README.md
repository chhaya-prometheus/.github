# Chhaya: Where Should Delhi Plant Trees First?

*A student project that ranks every part of a city by how urgently it needs shade*

Project write-up by the Chhaya team, October 2026

---

## 1. The question a planner actually asks

Heatwaves are not felt evenly. A crowded block with no trees bakes, while a leafy street a few kilometres away stays bearable. A city that can plant only so many trees has to decide where they go first.

Chhaya (छाया, Hindi for "shade") answers that decision with a map. It divides a city into equal hexagons, scores each one by how badly it needs trees, and colours the map from most to least urgent. A planner can click any hexagon, see the numbers behind its rank, and ask what would happen if canopy were added.

We built it for the Heat and Water track, which asks for tools that change something for people who live through heatwaves. A planner needs a ranked map, not a live feed. That single observation shaped the whole design.

We held to one rule throughout: never claim more than the data shows. The priority score is a proxy, not a measurement. Trees cool surfaces; they do not clean the air. Every what-if figure is a model estimate.

Three people each owned one layer: the data pipeline, the backend API, and the frontend. This write-up covers all three and the seams between them.

## 2. What Chhaya does

A visitor opens one web page and sees a city covered in coloured hexagons. From there they can:

- pick a city from a dropdown, with one city on the map at a time
- recolour the map by priority, heat, greenery, people or dust
- search a Delhi ward, see its outline, and read a summary panel
- click a hexagon to see temperature, greenery, built-up share, dust, population and schools
- run a what-if by adding hectares of canopy and reading the estimated cooling and people helped
- read a Top-10 list of the most urgent hexagons or wards
- enter a number of hectares and see where a planting plan would go
- copy a link that reopens the same city, ward and hexagon

Only Delhi is a real-data city. It has real ward boundaries and real vegetation data. Jaipur, Lucknow, Hyderabad and Chandigarh are demo cities. They use a boundary-only grid filled with synthetic values, and the interface labels them "Demo data (synthetic)" wherever the numbers could be mistaken for findings. They exist to show that adding a city is a configuration change, not a rewrite.

## 3. Architecture at a glance

Chhaya data changes monthly, not by the second. So we split the system in two. A slow batch pipeline produces finished files. A fast API and a map only ever read those files.

```
satellite + open data  -->  pipeline (batch)  -->  S3: features/
                                                      |
                                         FastAPI (in-memory cache)
                                                      |
                      Caddy: /api/* proxied, everything else static
                                                      |
                                          React + MapLibre in the browser
```

The three layers live in three repositories.

| Repository | Layer | What it owns |
| --- | --- | --- |
| `chhaya` | Data pipeline | H3 grids, data sources, priority score, schema validation, S3 upload |
| `chhaya-backend` | Backend API | FastAPI service, in-memory cache of S3 files, tests, CI and deploy workflow, Docker Compose and Caddyfile |
| `chhaya-frontend-aws` | Frontend | React, Vite, MapLibre and Tailwind app in `web/`, its Dockerfile and deploy workflow |

This split has a useful side effect. The map never talks to a satellite, so a failed data source can never take the site down. At worst, some numbers get older.

Production runs on one AWS EC2 instance with Docker Compose at `https://chhaya.duckdns.org`. Caddy serves the built frontend at `/` and proxies `/api/*` to the `api` container with the `/api` prefix stripped. The frontend is baked into the Caddy image.

## 4. The data pipeline

The pipeline is a slow, boring batch job on purpose. It builds a table with one row per hexagon, scores it, validates it, and writes it to S3.

### 4.1 Hexagons and ward names

Every cell is an H3 resolution-8 hexagon, about 0.7 km² each. Delhi gives 1,945 cells, and the live API reports the same count. We chose hexagons because every cell has the same area and the same six neighbours. Scores therefore compare fairly across the city, and the cell ID `h3_index` is a stable key that every source can join on.

The first version had a usability hole: a hexagon has no name. Nobody looking at the map could tell which part of Delhi a cell was. We fixed this with DataMeet's Delhi Municipal Spatial Data ward file.

- Each hexagon is joined to the ward it overlaps most, measured in a metric projection.
- If the best overlap is under 20% of the hexagon, the nearest ward within 500 m is used.
- Hexagons beyond 500 m from any ward stay null, and the run logs which ones.

The ward name and a stable `ward_id` become columns of the cell file. The ward file does not state which ward delimitation it uses, so it may be older than the current one. The pipeline README says so.

### 4.2 Sources and the priority score

Each hexagon gets one number, `priority`, a weighted blend of four signals. Each signal is scaled to 0..1 across the city.

| Signal | Column | Weight |
| --- | --- | --- |
| Heat (land surface temperature) | `lst_c` | 0.40 |
| People exposed | `population` | 0.25 |
| Missing greenery (1 minus NDVI) | `ndvi` | 0.20 |
| Dust | `dust_idx` | 0.15 |

Missing heat, greenery and dust values take the city median. Missing population counts as zero. `priority_rank` orders the cells, with rank 1 the most urgent. The weights are a judgement call, not the result of fitting. They sit together in `src/bottom_pipeline.py`, so a planner can change them.

Here is the honest status of each data source in the code as committed.

| Data | Columns | Status |
| --- | --- | --- |
| Vegetation | `ndvi` | Real. Sentinel-2 L2A scenes from the Earth Search catalogue, red and near-infrared bands at 10 m, composited from four scenes, averaged per hexagon |
| Wards | `ward`, `ward_id` | Real. DataMeet boundaries joined by largest overlap |
| Heat | `lst_c` | Placeholder (`LandsatStub`), deterministic values between 32 and 48 °C |
| Population | `population` | Placeholder (`WorldPopStub`) |
| Built-up share, schools, water proximity | `built_frac`, `school_count`, `water_prox` | Placeholder (`OSMStub`) |
| Dust | `dust_idx` | Placeholder (`DustStub`) |

The placeholders are enabled by `use_stubs: true` in `config/base.yaml`. Each is seeded by city slug and hexagon ID, so it returns the same values on every run and every machine. The names point to the real sources we intend to wire in: Landsat, WorldPop and OpenStreetMap.

The consequence matters. Delhi's ranking today pairs real greenery and real ward names with placeholder heat, population and dust. It demonstrates the method end to end. It is not planning advice.

Demo cities use a different source, `MockSource`. It places a few Gaussian "blobs" over the city, such as a dense core, green pockets and a dusty zone, and derives every value from distance to them. The result is spatially coherent, deterministic, and never presented as real.

### 4.3 Never overwrite good data

The first export simply overwrote the cell file. A failed run could then replace good values with nulls. We replaced it with a two-tier merge.

- Each source runs inside its own `try/except`. A failure produces an empty column and a status entry, not a crash.
- `merge_preserving_old()` in `src/export.py` keeps the previous file from S3. Columns from sources that succeeded take new values. Columns from failed sources keep their old ones.
- The run writes a status file per city. For each source it records whether it succeeded, how many nulls it left, and any error.

That status file lets the map warn users when a source is stale instead of silently showing old numbers. A production upload also needs an explicit `confirm_prod_upload: true`, because a bad run must never overwrite good data by accident.

### 4.4 Layered configuration and one schema

Configuration is layered. `base.yaml` holds shared defaults. `dev.yaml` and `prod.yaml` override them per environment. One small YAML file per city sets its name, slug, boundary, H3 resolution and data mode. The `CHHAYA_ENV` variable switches environments. In dev nothing uploads to S3. In prod everything does.

Every source returns exactly `[h3_index, <its columns>]` and never geometry. The orchestrator merges the sources onto the grid, computes the score, and validates each row against `contract/cell.schema.json` before anything is uploaded. If validation fails, nothing is uploaded.

## 5. The backend API

The backend turns the files in S3 into a small HTTP API. It does almost no computation. It loads whole files into memory and serves them from there.

### 5.1 Design decisions

**Cache whole files in memory.** At startup `load_data()` in `src/services/s3_service.py` fetches every file. A daemon thread then wakes every `REFRESH_SECONDS`, which defaults to 300. It compares each file's S3 `ETag` and downloads only what changed. We rejected reading S3 on every request. That adds latency and makes every request depend on AWS being reachable.

**Keep the last good copy.** `_install()` builds the complete entry first, then stores it with one assignment. If building fails, the old entry stays in place. Serialisation uses `json.dumps(..., allow_nan=False)`, so a `NaN` in the data fails the refresh instead of producing JSON that browsers cannot parse.

**One broken city must not break the others.** `cities.json` is required at startup. A broken city file is logged and skipped. The app refuses to start only if no cells file loads at all.

**Per-city routes, plus legacy aliases.** New routes live under `/cities/{slug}/...`. The older Delhi-only routes remain so the first deployed site kept working during the transition.

### 5.2 Endpoints

| Endpoint | Behaviour |
| --- | --- |
| `GET /health` | `status`, plus per city `cell_count`, `loaded_at` and `etag` |
| `GET /cities` | Contents of `cities.json` |
| `GET /cities/{slug}/cells` | The city's GeoJSON, with `Cache-Control: public, max-age=300` |
| `GET /cities/{slug}/status` | The city's pipeline status file |
| `GET /cities/{slug}/wards` | Ward summaries sorted by `ward_rank`; empty if the city has no wards |
| `GET /cities/{slug}/wards/{ward_id}` | One ward with `cell_ids` and `bbox` |
| `GET /cities/{slug}/top?level=cell\|ward&n=10` | Top cells by `priority`, or top wards |
| `POST /cities/{slug}/whatif` | Body `{h3_index, canopy_ha}`; returns `delta_c` and `people_benefited` |
| `/cells`, `/cells/{h3}`, `/wards`, `/status`, `/whatif` | Legacy, Delhi only |

Ward summaries are built in `_build_wards()`. Cells with a null `ward_id` are skipped, which is how demo cities end up with no wards. Wards rank by average `priority`.

### 5.3 Testing

The 20 tests in `tests/test_s3_data.py` run against a fake S3 built with `moto` in `tests/conftest.py`. The fake bucket holds a three-ward Delhi, a demo city with null ward columns, a cell with no ward, and a cell with a null `ndvi`. We ran the suite locally against the committed code, and all 20 passed.

They cover contract fields, the cache header, 404 handling, ward totals matching the cells, sort order of `/top`, the cooling formula, legacy routes, CORS preflight for `POST`, startup failures for a missing bucket, and one broken city leaving the others working.

They do not cover the background refresh loop, the gzip and logging middleware, `GET /cells/{h3}`, or the running Docker image. The pipeline has no tests, and the frontend has no tests in its repository.

## 6. The frontend

The frontend lives in `web/` and is deliberately thin. It holds no secrets and does no scoring of its own. It shows what the pipeline and API provide.

### 6.1 One city at a time

The map holds one city. When someone switches from Delhi to Chandigarh, Delhi's hexagons, selection, ward and plan are discarded before the new city loads. Keeping several cities in memory invites stale selections, mixed colour scales and ward outlines from the wrong city.

The map is keyed by the city slug, so no layer or filter survives a switch. The URL carries `city`, `ward` and `cell`, so a copied link reopens the same view. The default city is the one in the URL if valid, otherwise the first city marked real.

### 6.2 Saying what is fake

We treat synthetic numbers as a trust problem. For demo cities the interface shows a "Demo data (synthetic)" badge in the header, the legend and the detail panel, driven by the city's `data_mode`. The whole ward interface is hidden when `has_wards` is false. A footer is always visible and credits the Delhi ward boundaries, OpenStreetMap contributors and the CARTO basemap.

The offline mock files used in development contain invented "Mock Ward" names. A Vite plugin strips them from production builds, and the app reads them only when `VITE_USE_MOCK=true`.

### 6.3 Wards, labels and the Top-10

Wards are built in the browser by grouping hexagons on `ward_id`. The ward view therefore cannot disagree with the hexagons on the map. A ward's reason compares its mean heat, greenery and population with the city average and names the factor that stands out.

Labels follow one rule everywhere: use `area_name`, else the ward name, else "Cell" plus a short ID. The short ID needed a trick. Every real H3 index ends in `ffff`, so the last four characters would give every cell the same label. The code strips the trailing `f` characters first.

### 6.4 Resilience

Real data is messy, so the app keeps working when a value or a route is missing.

- **Null-safe rendering.** Every number passes through `fmt()`, which prints "n/a" for anything not finite. Missing values draw grey on the map.
- **Robust requests.** Calls time out after 15 seconds, or 30 for the large cells file. Failures show a readable message and a Retry button.
- **Optional endpoints fall back.** Only `/cities` and `/cities/{slug}/cells` are required. Status, what-if, Top-10 and the plan each degrade to a browser-side computation.
- **Robust colour scales.** Each layer scales between the 2nd and 98th percentile, so a few extreme hexagons do not wash out the city.

This choice let the frontend be built and demonstrated while the backend and pipeline were still catching up.

### 6.5 The what-if estimate

The backend and the browser use the same hand-written formula. With `h` hectares of new canopy, greenery `n` and population `p`:

```
cooling (°C) = 0.5 * h * (0.4 + (1 - n)) / (1 + 0.2 * h)
people       = round(p * min(1, h / 4) * 0.6)
```

A missing `n` counts as 0.2 and a missing `p` as 0. Two worked examples show the shape. With `n` = 0.25, 2 ha and 1,200 people, the result is 0.82 °C cooler and 360 people helped. With `n` missing, 4 ha and no people, it is 1.33 °C and 0 people.

The formula has diminishing returns: each extra hectare cools less than the last. It is a heuristic, not a calibrated physical model. The code comment in `run_whatif()` reads "Keep identical to the frontend fallback" because two copies of one formula can quietly drift apart. The backend test checks its copy against an independently computed value. The frontend copy is not under test yet.

### 6.6 Map engineering and the cut-off bug

The map is one MapLibre instance with a single GeoJSON source and five layers: `cells-fill`, `cells-line`, `plan-fill`, `ward-outline` and `cells-selected`. All sit beneath the basemap labels so place names stay readable. Changing the colour layer updates one paint property, so nothing is rebuilt.

One bug taught us something. We added a pan limit so visitors could not drift away from the selected city. Part of the city was then cut off. The limit was the city's bounding box plus 25%. A browser window is rarely the same shape as a city, so the fitted view sometimes exceeded the limit, and MapLibre resolved the conflict by zooming in.

The fix frames the city from the real extent of its hexagons and sets the pan limit to that extent plus a full city-width on every side. We believe that was the cause, but we have not proved it line by line. It deserves a re-check on a very wide screen and on a phone.

## 7. The contract that held three repositories together

Three people building in parallel only works if everyone builds against the same shapes. We agreed two rules. Existing field names are never renamed, only added. And whoever needs a shape to change tells the owner of the data first.

`features/cities.json` lists the cities. Each entry has `slug`, `name`, `bbox`, `center`, `zoom`, `data_mode` (`real` or `demo`) and `has_wards`. The slug is the filename prefix in the bucket, so no city name is hardcoded in the frontend.

Per city, `features/<slug>_cells.geojson` holds one feature per hexagon with these properties:

| Property | Meaning |
| --- | --- |
| `h3_index` | Stable cell ID |
| `ndvi`, `lst_c` | Greenery and surface temperature; either may be null |
| `population`, `built_frac`, `dust_idx`, `school_count` | Context and score inputs |
| `priority`, `priority_rank` | Score from 0 to 1, and rank with 1 most urgent |
| `ward`, `ward_id`, `area_name` | Place names; null for demo cities |

The contract was tested by real contact. The live file named the cell ID `h3` while the app expected `h3_index`. Rather than change the contract, `getCells` copies `h3` into `h3_index` when it is missing. The fix lives in one function.

## 8. Deployment: three repositories, no AWS keys

The AWS account sits inside an AWS Organization whose policy denies IAM OIDC providers. That rules out GitHub OIDC, IAM roles for GitHub, and SSM Run Command. We also chose not to store AWS keys in GitHub secrets. So CI and deploy use no AWS credentials at all.

### 8.1 The forced-command SSH key

Deploys use an SSH key whose entry in the server's `authorized_keys` begins `restrict,command="/home/ubuntu/<script>.sh"`. The key can run that one script and cannot open a shell. The workflow connects and passes no command. What runs is decided by the server-side script, not by the workflow.

The backend and frontend each have their own script. Both take the same server-side lock with `flock` before touching Docker Compose. GitHub's `concurrency` setting only works inside one repository, so this lock is what stops a backend and a frontend deploy from running Compose at the same time. The frontend script rebuilds only the `caddy` service, using `--no-deps` so it leaves the `api` service alone.

### 8.2 The workflows

| Repository | On push to a non-`main` branch | On push to `main` |
| --- | --- | --- |
| `chhaya-backend` | `ci.yml`: `pytest` on Python 3.11, plus a `docker build` of the image | `deploy.yml`: runs CI as a gate, deploys over SSH, polls `/api/health` up to 12 times five seconds apart, then checks `/api/cells` returns a non-empty `FeatureCollection` |
| `chhaya-frontend-aws` | No CI workflow in the repository yet | `deploy.yml`: `npm ci` and `npm run build` on Node 22, deploys over SSH, then polls until the site returns 200 and `/api/health` reports `ok` |

Both deploy workflows share a `concurrency` group named `deploy-ec2` that never cancels a running deploy.

The branching rule is the same for both. Work happens on `dev`, is tested locally, and is merged to `main` only when it works. A push to `main` deploys. The rule matters more than usual because the backend's Compose file builds Caddy from the sibling frontend folder. A frontend change can therefore reach production through either pipeline.

The backend runs from `python:3.11-slim` under `uvicorn` with `--no-access-log`, because `src/main.py` logs requests itself and keeps `/health` at debug level. Compose caps container logs at 10 MB and three files.

## 9. Bugs we hit and what they taught us

Most of our problems were at the seams between repositories. The table gives each one, its cause and its fix. Rows marked Open are problems we found while writing this and have not fixed.

| Symptom | Cause | Fix |
| --- | --- | --- |
| Every H3 tutorial failed | H3 4.x renamed `polyfill` to `geo_to_cells` and changed boundary output to (lat, lng) | Check `h3.__version__` first; swap coordinates by hand for Shapely |
| "Cannot pick a common CRS" in `stackstac` | Earth Search stores the projection as `proj:code`; the library reads only `proj:epsg` | Pass `epsg=32643` explicitly |
| NDVI ranged from minus to plus infinity | Sentinel-2's empty border is 0, so the division gave 0/0 | Compute on raw arrays, then mask after dividing where both bands are positive |
| 533 of 1,945 hexagons had no NDVI | Satellite geometry, not clouds: no single tile covers Delhi, and our scenes overlapped | Pick one scene per tile across four tiles; valid pixels rose from 44% to 79%, nulls fell to 0 |
| "Multiple geometry columns" on write | Sources returned geometry and the merge duplicated it | Sources return only `[h3_index, columns]` |
| `rasterstats` shape error on cached rasters | Array and transform passed separately | Pass the file path so both come from one place |
| One failed source wiped a good column | Export overwrote the previous file | Two-tier merge keeping old values for failed sources |
| `src/lib` missing on GitHub | A Python `.gitignore` template contained `lib/` | A clean Node `.gitignore`, checked with `git check-ignore -v` |
| Cells would not load | Live API named the ID `h3`, app expected `h3_index` | `getCells` copies `h3` into `h3_index` |
| No ranks in the Top-10 | Early response lacked `priority_rank` | Rank by `priority`, descending |
| Every cell shared one label | All H3 IDs end in `ffff` | Strip trailing `f` characters first |
| Part of the map cut off | Pan limit tighter than the fitted view | Frame and limit from hexagon extent with a wide margin |
| One frame of the wrong city on switch | Data cleared one render late | Clear data with the city change; key the map by slug |
| "Data updated" date never appears (Open) | The pipeline writes `run_at`; the frontend reads `updated_at` or `updated` | Read `run_at` in `getStatus` |
| Stale contract copy (Open) | `chhaya-backend/contract/cell.schema.json` still says `h3` and omits `ward_id` and `priority_rank` | Sync it from the pipeline repository |

Three lessons came out of this.

- **Put every mismatch in one place.** Because every API call goes through `api.js`, a backend difference became a three-line fix.
- **Optional should really be optional.** Local fallbacks let the frontend run while the backend was still catching up.
- **Say what is fake.** Labelling demo data on every surface costs almost nothing. It removes the one mistake that would damage trust in the whole project.

## 10. Honest status and limitations

We would rather you know what is unproven than discover it.

**What we verified by running it**

- All 20 backend tests pass locally against the committed code.
- The live API reported 1,945 cells when we checked.

**What we only wrote or read**

- The deploy scripts and the server-side lock live on the server, not in the repositories. We describe them from the workflows and our own setup.
- The frontend has no automated tests. Its what-if figures above are worked by hand, not test results.
- The pipeline has no tests and no CI.

**Known limitations**

- **Placeholder data.** Heat, population, built-up share, schools and dust are placeholders in Delhi. Only greenery and ward names are real.
- **Missing endpoints.** The frontend calls `POST /cities/{slug}/plan` and `GET /cities/{slug}/wards/{id}/brief`. The backend has neither. The planner falls back to a greedy browser-side plan. The "Generate ward brief" button shows its error state.
- **Two ward rankings.** The backend ranks wards by average `priority`. The frontend ranks them by mean `priority_rank`. The two could disagree, and the frontend uses its own.
- **Missing label column.** The contract lists `area_name`, and labels prefer it. The pipeline code we reviewed does not write it, so labels fall back to the ward name.
- **Live deployment.** When we checked, the live `/api/health` still answered in the older single-city shape. We verified the committed code, not that the latest backend is what is running.
- **Planner.** It assumes one hectare per hexagon and picks the most urgent first. It is a greedy plan, not an optimisation.
- **Operations.** There is no automatic rollback. Port 22 is open, limited only by the forced command. Rebuilding Caddy causes a few seconds of downtime.
- **Stale ward file.** The ward boundaries may predate the current delimitation.
- **Unused settings.** The backend config includes Postgres settings, but no code uses a database.
- **Double compression.** Both the app and Caddy compress responses.

## 11. What we would do next

1. Wire in the real sources for heat, population, built-up share, schools and dust, then switch `use_stubs` off.
2. Add the `/plan` and ward-brief endpoints, or remove the calls.
3. Fix the `run_at` mismatch and sync the stale schema copy.
4. Write `area_name` in the pipeline.
5. Add a frontend CI workflow with lint, tests and a component test that fails if a demo city loses its badge.
6. Add tests for the pipeline merge logic and the backend refresh loop.
7. Put the deploy scripts in the repositories.
8. Remove the legacy routes and the single-city fallback once nothing depends on them.
9. Add screenshots of the map, ward panel and demo badge to the READMEs.

## 12. Data credits and repositories

- Ward boundaries: DataMeet, Delhi Municipal Spatial Data (CC BY-SA 2.5 IN).
- Vegetation: Sentinel-2 L2A, searched through the Earth Search STAC catalogue.
- Basemap and map data: CARTO and OpenStreetMap contributors.
- Repositories: `chhaya-prometheus/chhaya`, `chhaya-prometheus/chhaya-backend` and `chhaya-prometheus/chhaya-frontend-aws`, all under the MIT license.

Chhaya is a prototype of a method. It shows how a planner could rank a city by need, and it says plainly which parts of that ranking are real today.
