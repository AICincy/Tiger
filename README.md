# Tiger — TIGER/Line Defect Audit for SORTA MetroNow

**Live dashboards:** https://aicincy.github.io/Tiger/

A read-only OpenStreetMap audit pipeline that finds routing-breaking defects in
unreviewed 2007–2008 TIGER/Line Census road segments inside Hamilton County, OH
microtransit zones operated by SORTA's [MetroNow](https://www.go-metro.com/metronow)
service (powered by [Via Transportation](https://ridewithvia.com/)).

It produces a publication-ready Excel workbook, an interactive Leaflet dashboard
(road geometry, node-disconnect markers, JOSM/iD edit links), and supporting
CSVs that human OSM editors can act on.

> **The pipeline never modifies OSM.** It is strictly an audit. Every output
> traces back to a single live Overpass API query.

| Per-zone dashboards (live) | Combined view |
|---|---|
| [Blue Ash / Montgomery](https://aicincy.github.io/Tiger/tiger_audit_blue_ash_montgomery/reports/TIGER-Audit-Blue-Ash-Montgomery-Dashboard.html) | [All four zones (deduped)](https://aicincy.github.io/Tiger/tiger_audit_all_zones/TIGER-Audit-All-Zones-Dashboard.html) |
| [Springdale / Sharonville](https://aicincy.github.io/Tiger/tiger_audit_springdale_sharonville/reports/TIGER-Audit-Springdale-Sharonville-Dashboard.html) | [All-zones summary CSV](https://aicincy.github.io/Tiger/tiger_audit_all_zones/all_zones_summary.csv) |
| [Northgate / Mt. Healthy](https://aicincy.github.io/Tiger/tiger_audit_northgate_mt_healthy/reports/TIGER-Audit-Northgate-Mt-Healthy-Dashboard.html) | |
| [Forest Park / Pleasant Run](https://aicincy.github.io/Tiger/tiger_audit_forest_park_pleasant_run/reports/TIGER-Audit-Forest-Park-Pleasant-Run-Dashboard.html) | |

---

## Why this exists

SORTA's MetroNow microtransit routes on top of OpenStreetMap. The 2007–2008
TIGER/Line Census import seeded thousands of road segments with
`tiger:reviewed=no` — meaning no human has verified the geometry or tags since
import. Two recurring defects in that data translate directly into routing
failures for transit-dependent riders:

- **False `oneway=yes` on residential streets** — Via's routing engine sees
  cul-de-sacs and dead-end residential streets as legal one-ways, which makes
  them entry-only with no exit, producing failed routes.
- **Disconnected nodes between same-street segments** — endpoints of two ways
  on the same named street that sit ~2–30 m apart but aren't joined by a shared
  OSM node. The router treats them as different roads.

For a transit-dependent rider, a routing failure is a service denial. This tool
surfaces those defects so volunteer OSM editors can fix them.

---

## Scope

**Geographic:** Hamilton County, OH. Four MetroNow zones (one zone per audit
run, or `--zone all` for the union):

| Zone key | Name | Description |
|---|---|---|
| `blue_ash_montgomery` *(default)* | Blue Ash / Montgomery | Blue Ash, Montgomery, Deer Park, Silverton, Kenwood, Madeira |
| `springdale_sharonville` | Springdale / Sharonville | Springdale, Sharonville, Glendale, Evendale, Lincoln Heights |
| `northgate_mt_healthy` | Northgate / Mt. Healthy | Mt. Healthy, North College Hill, Finneytown, Northgate |
| `forest_park_pleasant_run` | Forest Park / Pleasant Run | Forest Park, Pleasant Run, Greenhills |

**Data source:** A single Overpass API query per zone, filtered to ways with
`highway=*` and `tiger:reviewed=no` inside the zone bounding box. Raw JSON is
preserved (timestamped) on every run.

**Defect classification (from each unreviewed way):**

| Class | Rule | Severity |
|---|---|---|
| **A** — false one-way | `highway=residential` AND `oneway=yes` | CRITICAL |
| **B** — multi-segment | named street with ≥2 unreviewed segments in the dataset | HIGH |
| **AB** — compound | A AND B (false one-way on a multi-segment street — worst case) | CRITICAL |
| **C** — unreviewed | everything else with `tiger:reviewed=no` | LOW |

**Node-disconnect detection:** for every Class B street, the pipeline scans
endpoint pairs across that street's ways with the haversine distance and flags
any pair within **30 m** that aren't already coincident. Pairwise gap records
within 5 m of each other on the same street are then collapsed to a single
representative (since a 3+ way junction with no shared node would otherwise
emit one record per pair for what's really one missing node). Each gap shows
up as a purple marker on the dashboard with one-click links to JOSM (via
Remote Control) or the iD editor at the gap location.

**Out of scope:**
- Modifying OSM (this is read-only).
- Verifying that flagged defects are *actually* defects on the ground —
  flagging is heuristic; a human editor confirms each fix.
- Routing engine internals (we don't run Via; we just produce the data
  defects that affect it).
- Zones outside Hamilton County, OH.

---

## What the pipeline produces

For each zone (default `blue_ash_montgomery`):

```
tiger_audit_<zone_key>/
  data/<zone_key>_raw_<UTC>.json          # raw Overpass response (timestamped, gitignored)
  reports/
    TIGER-Audit-<Zone>.xlsx               # 8-sheet styled workbook
    TIGER-Audit-<Zone>-Dashboard.html     # interactive Leaflet dashboard
  csv/
    all_ways.csv                          # master inventory
    class_a_false_oneway.csv              # residential + oneway=yes
    class_b_multi_segment.csv             # streets with 2+ unreviewed segments
                                          #   (sorted by gap count + segment count;
                                          #    `_street_gap_count` column added)
    class_ab_compound.csv                 # intersection of A and B
  README.md                               # re-run instructions + file manifest
```

When `--zone all` runs, an additional combined directory is produced:

```
tiger_audit_all_zones/
  TIGER-Audit-All-Zones-Dashboard.html    # combined Leaflet dashboard, deduped
  all_zones_summary.csv                   # one row per zone + Hamilton County
                                          #   per-zone-sum and unique-deduped totals
```

The four MetroNow zone bboxes overlap (most substantially Northgate ↔ Forest
Park, ~32 km²). The combined view dedupes ways by OSM ID, dedupes gaps by
unordered way-pair plus a 5 m spatial sweep, and counts unique multi-segment
streets via a connected-components analysis on shared way IDs — so a way that
sits in two zones counts once, while two unrelated streets that happen to share
a name in different zones count separately.

### XLSX workbook (8 sheets)

1. **Executive Summary** — totals, residential share, defect counts, context.
2. **Class A – Highest Risk** — Class AB streets (compound), grouped by name.
3. **Class A – Moderate Risk** — single-segment Class A only; if "O'Leary
   Avenue" is present in the data it's highlighted as the index case.
4. **Class B – Multi-Segment** — streets with ≥5 segments, sorted by count;
   one-way segments per street called out in red.
5. **All Ways** — the complete inventory, sorted AB → A → B → C then by name.
6. **MetroNow Zones** — all four zones; the audited zone is marked
   `AUDIT COMPLETE` in green, and any of the other three zones whose
   `csv/all_ways.csv` already exists on disk are also marked complete with
   their actual segment count (so a workbook reflects the global audit
   state at the moment it was generated, not just the one-zone view).
7. **Work Plan** — phased volunteer-hour estimates with formula-driven totals
   (12 / 4 / 7 / 2 minutes per item for P1 / P2 / P3 / P4).
8. **Overpass Query** — exact query, endpoint, bbox, timestamp.

### Interactive dashboard (single-file HTML)

Open by double-clicking — no server required. Loads Leaflet 1.9.4,
Leaflet.heat, and Leaflet.markercluster from CDN; all defect data is embedded
inline. Public copy: https://aicincy.github.io/Tiger/.

> **Windows note:** opening the HTML directly via `file://` works on Chrome,
> Edge, and Firefox; CARTO and Esri tiles load fine. If a corporate proxy
> intercepts CDN requests, or if the browser blocks mixed-content on
> `file://`, run `python -m http.server` from the `reports/` directory and
> open `http://localhost:8000/` instead. JOSM Remote Control is HTTP-only;
> `https://` won't work as the dashboard origin.

**Sidebar:**

- Six KPI cards (Total / Residential / Class AB / Class A / Class B / Node Gaps).
- Layer toggles for each defect class + node-disconnect markers + density
  heatmap. Class C is off by default.
- Basemap selector (see below). Place-label overlay applies automatically over
  any imagery basemap.
- Street-name search with combobox semantics — fly to a street, highlight all
  segments sharing that name.
- Live viewport stats: segments / streets / per-class counts / gaps in the
  current view, updated on pan/zoom.
- Export buttons: visible-viewport GeoJSON, visible-viewport CSV, and
  **Load in JOSM** which batches all visible way IDs and POSTs to JOSM
  Remote Control at `localhost:8111` (chunked under ~7,500 chars/URL).
- **Print View** button — strips chrome and dark theme, fills the viewport
  for clean export to PDF or paper.
- Dark-mode toggle (🌙/☀️) in the sidebar header; choice persists in
  `localStorage` and respects `prefers-color-scheme` on first load.

**Map:**

- Each unreviewed way is a colored polyline (color/weight by defect class).
- Clicking any polyline highlights all segments of that street and opens a
  popup with the defect breakdown and OSM/iD/JOSM links per segment.
- Clustered purple circle markers for probable node disconnects (auto-cluster
  below zoom 16). Clicking a marker pulses the two involved segments and
  draws a magenta dashed line at the near-miss endpoints.
- URL hash state: zoom/center/active-layers/basemap/dark/highlighted-street
  encode into `location.hash` (debounced) so a copied URL restores the same
  view.

**Keyboard shortcuts** (also in the `?` overlay):

| Key | Action |
|---|---|
| `/` or `Ctrl+K` | Focus the search box |
| `Esc` | Clear search / highlight / popup |
| `1`–`6` | Toggle layers (AB, A, B, C, Gaps, Heatmap) |
| `S` | Toggle Esri satellite imagery |
| `?` | Show / hide shortcut help overlay |

**Mobile:** at ≤ 768 px the sidebar collapses behind a hamburger; the
Leaflet zoom control hides (pinch-zoom replaces it).

#### Basemap options

The dashboard's "Basemap" dropdown switches between several free imagery / road
tile sources. The current selection is persisted in browser `localStorage`.

| Option | Provider | Notes |
|---|---|---|
| CARTO Voyager *(default)* | CARTO | Vector road basemap derived from OSM |
| CARTO Positron | CARTO | Light, low-contrast — good for screenshots |
| CARTO Dark Matter | CARTO | Dark theme variant |
| OpenStreetMap Standard | OSMF | Standard OSM tiles |
| Esri World Imagery | Esri | Satellite, public-tier *(+ auto labels)* |
| Esri World Imagery (Clarity) | Esri | Sharper / Firefly imagery *(+ auto labels)* |
| USGS National Map — Imagery | USGS | NAIP-derived *(+ auto labels)* |
| USGS National Map — Topo | USGS | Topographic tiles |
| Esri World Topographic | Esri | Topo with road labels |
| **Ohio OSIP** *(best-effort)* | OGRIP / Ohio | 6-inch / 15 cm imagery *(+ auto labels)* |
| Esri Wayback | Esri | Date-stamped historical satellite *(see below; + auto labels)* |

For any imagery basemap (rows marked *+ auto labels*), the dashboard
auto-overlays a transparent CARTO `voyager_only_labels` tile layer on top so
neighborhood/place names and street labels remain visible. CARTO's own
basemaps and the topographic options keep their built-in labels and don't
get the overlay.

#### Wayback Imagery (date-stamped historical satellite)

Picking **Esri Wayback (date selector)** from the basemap dropdown reveals a
"Wayback Release" panel. The dashboard fetches Esri's release manifest at
`https://s3-us-west-2.amazonaws.com/config.maptiles.arcgis.com/waybackconfig.json`
and populates the dropdown with every dated capture (typically 60–80 going back
to 2014-02-20). Picking a release switches the tile URL to the corresponding
WMTS layer:

```
https://wayback.maptiles.arcgis.com/arcgis/rest/services/World_Imagery/WMTS/1.0.0/default028mm/MapServer/tile/<release>/{z}/{y}/{x}
```

Wayback is free / unauthenticated — no token needed. The selected release
persists in `localStorage`.

The practical use for this audit: when you spot a Class AB or node-disconnect
candidate, flip through 2-3 Wayback dates to confirm the road geometry on the
ground actually disagrees with what's tagged. If imagery from 2014 already
shows the road connected and OSM still has a gap, that's a high-confidence fix.

---

## Quick start

Requires Python 3.11+.

```bash
pip install -r requirements.txt
python3 tiger_audit.py                              # default: Blue Ash / Montgomery
python3 tiger_audit.py --zone springdale_sharonville
python3 tiger_audit.py --zone all                   # all four zones + combined dashboard
```

Outputs land in `tiger_audit_<zone_key>/` next to the script. The raw Overpass
JSON is timestamped, so re-runs accumulate snapshots without overwriting
historical data — useful for tracking progress as volunteers fix defects.

---

## Sample run (Blue Ash / Montgomery, 2026-04-28)

| Metric | Count |
|---|---|
| Total unreviewed segments | 1,934 |
| Residential | 1,163 |
| False-one-way residential ways (Class A + Class AB combined) | 57 |
| &nbsp;&nbsp;— of which Class A only (single-segment) | 2 |
| &nbsp;&nbsp;— of which Class AB (compound, also multi-segment) | 55 |
| Class B (multi-segment ways) | 1,064 |
| Multi-segment streets (Class B groups) | 182 |
| Probable node disconnects | 248 |
| Estimated volunteer hours (Class AB only) | 11.0 |
| Estimated volunteer hours (everything) | 69.2 |

A real run is included under
[`tiger_audit_blue_ash_montgomery/`](tiger_audit_blue_ash_montgomery/).
Open the dashboard locally:

```
tiger_audit_blue_ash_montgomery/reports/TIGER-Audit-Blue-Ash-Montgomery-Dashboard.html
```

---

## How it works

1. **Phase 1 — Fetch.** A single Overpass query per zone, with retry against
   the kumi.systems mirror if the primary endpoint fails. The query uses
   `out geom tags;` so polylines come back inline. Falls back to the most
   recent cached snapshot (sorted by mtime, age-warned past 14 days) if all
   endpoints are unreachable. JSON-`null` responses are treated as retryable.
2. **Phase 2 — Classify.** Group ways by case-insensitive normalized street
   name; apply the Class A / B / AB / C rules above; run the
   30 m endpoint-distance gap detector across all Class B streets, then
   collapse pairwise gap records within 5 m on the same street into a
   single representative.
3. **Phase 3 — XLSX.** `openpyxl` produces the 8-sheet workbook; totals,
   percentages, and volunteer-hour columns are real Excel formulas, not
   pre-computed values.
4. **Phase 4 — HTML dashboard.** Single self-contained file; defect data
   embedded as compact JSON; Leaflet + Leaflet.heat + Leaflet.markercluster
   loaded from CDN.
5. **Phase 5 — CSVs + README + summary.** Four sorted CSVs (with
   `_street_gap_count` on the multi-segment one), a per-zone re-run README,
   and a console summary box including the gap count.

When run with `--zone all`, a sixth phase generates the deduped combined
dashboard and `all_zones_summary.csv`.

The pipeline is one Python file ([`tiger_audit.py`](tiger_audit.py)).

---

## Conventions and invariants

- **No data hallucination.** Every count, list, and chart series traces back
  to the live Overpass response. If the query returns 1,800 ways instead of
  1,934, the dashboard reports 1,800.
- **Original tag values preserved.** Street names, highway types, and oneway
  values are passed through verbatim from OSM. Normalization is
  grouping-only.
- **Missing tags handled gracefully.** Ways without a `name` are still
  classified (a residential way with `oneway=yes` is Class A regardless of
  name), shown as `[Unnamed]` in displays, and never grouped into Class B.
- **Idempotent re-runs.** XLSX / dashboard / CSVs / per-zone README are
  overwritten; only the raw JSON snapshot accumulates with timestamps.

---

## Editing OSM

This repo doesn't edit OSM. Each defect has deep links you can use:

- **iD editor** (browser): every way and gap popup has a link to
  `openstreetmap.org/edit?editor=id#map=18/<lat>/<lon>` zoomed in on the
  defect.
- **JOSM Remote Control**: every popup also has a link to
  `localhost:8111/load_and_zoom?...`. This works only if you have JOSM
  running locally with Remote Control enabled (Preferences → Remote Control).

If you fix something, please remove `tiger:reviewed=no` (or upgrade it to
`tiger:reviewed=yes` if your team uses that tag) so the next audit run will
no longer flag the segment.

---

## Layout

```
.
├── README.md
├── CHANGELOG.md
├── LICENSE
├── requirements.txt
├── tiger_audit.py                          # the entire pipeline
├── index.html                              # GitHub Pages landing → redirects to BAM dashboard
├── .nojekyll                               # tells GitHub Pages to skip Jekyll
├── tiger_audit_blue_ash_montgomery/        # per-zone outputs (one dir per audited zone)
│   ├── README.md
│   ├── csv/
│   │   ├── all_ways.csv
│   │   ├── class_a_false_oneway.csv
│   │   ├── class_ab_compound.csv
│   │   └── class_b_multi_segment.csv      # + _street_gap_count column
│   └── reports/
│       ├── TIGER-Audit-Blue-Ash-Montgomery.xlsx
│       └── TIGER-Audit-Blue-Ash-Montgomery-Dashboard.html
├── tiger_audit_springdale_sharonville/     # same shape; produced by --zone springdale_sharonville
├── tiger_audit_northgate_mt_healthy/       # same shape; produced by --zone northgate_mt_healthy
├── tiger_audit_forest_park_pleasant_run/   # same shape; produced by --zone forest_park_pleasant_run
└── tiger_audit_all_zones/                  # only when --zone all has been run
    ├── TIGER-Audit-All-Zones-Dashboard.html
    └── all_zones_summary.csv
```

The raw Overpass JSON snapshots in `tiger_audit_*/data/` are gitignored — they
are ~1 MB each and reproducible from any current run. The pipeline auto-prunes
snapshots older than 14 days, keeping the 3 most recent per zone.

---

## Acknowledgements

- Road data: © [OpenStreetMap contributors](https://www.openstreetmap.org/copyright), ODbL.
- Tile imagery: CARTO Voyager (OSM-derived), Esri World Imagery.
- Original 2007–2008 import: U.S. Census Bureau TIGER/Line.
- MetroNow operations: SORTA / Via Transportation.

This is a community audit tool. It is not affiliated with SORTA, Via, or the
OpenStreetMap Foundation.
