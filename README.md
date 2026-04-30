# Tiger

A read-only audit pipeline that enumerates and classifies the unreviewed-TIGER residual in OpenStreetMap inside the four MetroNow microtransit service zones operated by SORTA in Hamilton County, OH. The pipeline emits a styled XLSX workbook per zone, an interactive Leaflet dashboard with full road geometry and node-disconnect markers, four sorted CSV slices per zone, and a deduplicated combined-zone view. Every output traces to a single live [Overpass API](https://overpass-api.de/) query of record. The pipeline performs no OSM mutation.

**Live:** <https://aicincy.github.io/Tiger/> &middot; **Methods writeup:** [author.html](https://aicincy.github.io/Tiger/author.html)

| Per-zone dashboards (live) | Combined |
|---|---|
| [Blue Ash / Montgomery](https://aicincy.github.io/Tiger/tiger_audit_blue_ash_montgomery/reports/TIGER-Audit-Blue-Ash-Montgomery-Dashboard.html) | [All four zones (deduplicated)](https://aicincy.github.io/Tiger/tiger_audit_all_zones/TIGER-Audit-All-Zones-Dashboard.html) |
| [Springdale / Sharonville](https://aicincy.github.io/Tiger/tiger_audit_springdale_sharonville/reports/TIGER-Audit-Springdale-Sharonville-Dashboard.html) | [All-zones summary CSV](https://aicincy.github.io/Tiger/tiger_audit_all_zones/all_zones_summary.csv) |
| [Northgate / Mt. Healthy](https://aicincy.github.io/Tiger/tiger_audit_northgate_mt_healthy/reports/TIGER-Audit-Northgate-Mt-Healthy-Dashboard.html) | |
| [Forest Park / Pleasant Run](https://aicincy.github.io/Tiger/tiger_audit_forest_park_pleasant_run/reports/TIGER-Audit-Forest-Park-Pleasant-Run-Dashboard.html) | |

---

## Position

SORTA's MetroNow microtransit service consumes OpenStreetMap (OSM) as its routing base layer via Via Transportation. A non-trivial residual of unreviewed ways persists in OSM from the 2007–2008 TIGER/Line bulk import; each such way carries the tag `tiger:reviewed=no`, indicating that no human editor has verified its geometry, topology, or attribute set against ground truth. Two failure modes recur in this residual: erroneous directional restrictions (`oneway=yes` on residential dead-ends and cul-de-sacs), and disconnected nodes between two ways representing segments of the same physical street that terminate within a few metres of each other without sharing an OSM node. Either is sufficient, in the worst case, to render a residential address unreachable to a routing engine that treats the OSM graph as authoritative. The audit is designed to enumerate the empirical predicate; the legal and contractual questions that may follow from it are out of scope for the pipeline and are deferred to counsel review by the relevant agency.

---

## Scope

Hamilton County, OH. Four MetroNow service zones, one zone per audit run, or `--zone all` for the deduplicated combined view:

| Zone key | Name | Coverage |
|---|---|---|
| `blue_ash_montgomery` *(default)* | Blue Ash / Montgomery | Blue Ash, Montgomery, Deer Park, Silverton, Kenwood, Madeira |
| `springdale_sharonville` | Springdale / Sharonville | Springdale, Sharonville, Glendale, Evendale, Lincoln Heights |
| `northgate_mt_healthy` | Northgate / Mt. Healthy | Mt. Healthy, North College Hill, Finneytown, Northgate |
| `forest_park_pleasant_run` | Forest Park / Pleasant Run | Forest Park, Pleasant Run, Greenhills |

**Acquisition.** A single Overpass query per zone, filtered to ways carrying `highway=*` and `tiger:reviewed=no` inside the zone bounding box. Raw JSON is timestamped and persisted on every successful fetch; a JSON-`null` response is treated as retryable rather than as a successful empty fetch.

**Defect classification.** Each unreviewed way is assigned to exactly one of four disjoint classes:

| Class | Predicate | Severity |
|---|---|---|
| **A** — false one-way | `highway=residential` ∧ `oneway=yes` ∧ ¬*B(w)* | Critical |
| **B** — multi-segment | ∃ *w′* ≠ *w*: `norm(name(w′)) = norm(name(w))` ∧ ¬*A(w)* | High |
| **AB** — compound | *A(w)* ∧ *B(w)* | Critical |
| **C** — residual | ¬*A(w)* ∧ ¬*B(w)* | Low |

`norm` denotes case-insensitive trim; anonymous ways (no `name`) cannot satisfy *B* but may satisfy *A*.

**Probable-node-disconnect detection.** For each Class B street (the maximal set of ways sharing a normalised `name`), all way-pairs are enumerated and the minimum great-circle endpoint distance computed via haversine. Pairs in the half-open interval (0.01 m, 30 m] are emitted as candidates. A per-street spatial-clustering pass collapses the *k*(*k* − 1)/2 records emitted at a junction of *k* ≥ 3 unjoined ways into a single representative (the tightest pair, retained at 5 m clustering radius).

**Out of scope.** OSM mutation; ground-truth confirmation of flagged candidates; routing-engine internals; geographies outside Hamilton County, OH.

---

## Outputs

For each zone (default `blue_ash_montgomery`):

```
tiger_audit_<zone_key>/
  data/<zone_key>_raw_<UTC>.json          # raw Overpass response (timestamped, gitignored)
  reports/
    TIGER-Audit-<Zone>.xlsx               # 8-sheet styled workbook
    TIGER-Audit-<Zone>-Dashboard.html     # interactive Leaflet dashboard (self-contained)
  csv/
    all_ways.csv                          # master inventory
    class_a_false_oneway.csv              # residential + oneway=yes
    class_b_multi_segment.csv             # streets with ≥2 unreviewed segments;
                                          #   sorted by gap count then segment count;
                                          #   carries the _street_gap_count column
    class_ab_compound.csv                 # A ∩ B
  README.md                               # re-run instructions + file manifest
```

A `--zone all` run produces an additional combined directory:

```
tiger_audit_all_zones/
  TIGER-Audit-All-Zones-Dashboard.html    # combined Leaflet dashboard, deduplicated
  all_zones_summary.csv                   # one row per zone, plus the per-zone-sum
                                          #   row and the deduplicated unique row
                                          #   for Hamilton County totals
```

The four service-zone bounding boxes overlap pairwise; the most substantial overlaps are Northgate ↔ Forest Park (~32 km²) and Springdale ↔ Forest Park (~14.5 km²). Combined-zone aggregation deduplicates along three axes: OSM way ID for ways, transitive connected-components analysis over (`name`, way-ID set) groups for multi-segment-street counting, and a two-pass reduction for gaps (unordered way-pair, then 5 m spatial clustering per street). A way present in two zones counts once; two unrelated streets sharing a name in zones with disjoint way-ID populations count separately.

### XLSX workbook (eight sheets)

1. **Executive Summary.** Totals, residential share, defect counts, context, and a banner row when the Overpass response fell below the < 100-element sanity threshold.
2. **Class A — Highest Risk.** Class AB streets (compound), grouped by name.
3. **Class A — Moderate Risk.** Single-segment Class A only; the per-zone `index_case_street` (configurable per zone) is highlighted as the index case.
4. **Class B — Multi-Segment.** Streets with ≥ 5 segments, sorted by segment count; one-way segments per street called out in red.
5. **All Ways.** The complete inventory, sorted AB → A → B → C then alphabetically by name.
6. **MetroNow Zones.** All four zones; the audited zone is marked `AUDIT COMPLETE`. The other three zones are also marked `AUDIT COMPLETE` if their `csv/all_ways.csv` exists on disk, with the actual segment count read from that file. A workbook therefore reflects the global audit state at the moment it was generated.
7. **Work Plan.** Phased volunteer-hour estimates with formula-driven totals (12 / 4 / 7 / 2 minutes per item for P1 / P2 / P3 / P4).
8. **Overpass Query.** The exact query, endpoint, bbox, and timestamp.

### Interactive dashboard (single-file HTML)

Self-contained; opens by double-click. Loads Leaflet 1.9.4, Leaflet.heat, and Leaflet.markercluster from CDN; all defect data is embedded inline.

> **Windows note.** `file://` opening works on Chrome, Edge, and Firefox; CARTO and Esri tiles load. If a corporate proxy intercepts CDN requests or the browser blocks mixed content over `file://`, run `python -m http.server` from the `reports/` directory and open `http://localhost:8000/`. JOSM Remote Control is HTTP-only; an `https://` dashboard origin will not reach it.

**Sidebar.**

- Six KPI cards: Total · Residential · Class AB · Class A · Class B · Node Gaps. The Class A and Class AB cards are disjoint counts; their union recovers the per-zone CSV `class_a_count` column.
- Layer toggles per defect class plus node-disconnect markers and a density heatmap. Class C is off by default.
- Basemap selector. A transparent CARTO `voyager_only_labels` layer is auto-stacked over any imagery basemap so place-name and street labels remain visible on satellite tiles.
- Street-name search with combobox semantics; selecting a result flies to the street and highlights every segment of it.
- Live viewport stats: segments / streets / per-class counts / gaps inside the current view, recomputed on `moveend`.
- Visible-viewport export to GeoJSON or CSV, and `Load in JOSM` which batches all visible way IDs and POSTs to `localhost:8111` (chunked under ~7,500 chars per URL).
- Print View — strips chrome and dark theme, fills the viewport, calls `window.print()`, restores the prior theme on `afterprint`.
- Dark-mode toggle (🌙/☀️) in the sidebar header. Choice persists in `localStorage` and respects `prefers-color-scheme` on first load.

**Map.**

- Each unreviewed way renders as a polyline coloured/weighted by defect class. Clicking any polyline highlights every segment sharing the same normalised `name`; the popup carries the street-level defect-class breakdown and OSM/iD/JOSM links per segment.
- Clustered purple circle markers for probable node disconnects (auto-cluster below zoom 16). Clicking a marker pulses the two involved segments and draws a magenta dashed line at the near-miss endpoints.
- URL hash state — zoom, centre, active layers, basemap, dark flag, and highlighted street are encoded into `location.hash` (debounced) so a copied URL restores the same view.

**Keyboard shortcuts** (also surfaced via the `?` overlay):

| Key | Action |
|---|---|
| `/` or `Ctrl+K` | Focus the search box |
| `Esc` | Clear search / highlight / popup |
| `1`–`6` | Toggle layers (AB, A, B, C, Gaps, Heatmap) |
| `S` | Toggle Esri satellite imagery |
| `?` | Show / hide shortcut help overlay |

**Mobile.** At ≤ 768 px the sidebar collapses behind a hamburger; the Leaflet zoom control hides (pinch-zoom replaces it).

#### Basemap options

The current selection is persisted in `localStorage`.

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
| Esri Wayback | Esri | Date-stamped historical satellite *(+ auto labels)* |

For any imagery basemap (rows marked *+ auto labels*), the dashboard auto-overlays a transparent CARTO `voyager_only_labels` tile layer so neighbourhood and street labels remain legible. CARTO's own basemaps and the topographic options carry their own labels and receive no overlay.

#### Wayback imagery

Selecting **Esri Wayback (date selector)** reveals a "Wayback Release" panel. The dashboard fetches Esri's release manifest at `https://s3-us-west-2.amazonaws.com/config.maptiles.arcgis.com/waybackconfig.json` and populates the dropdown with every dated capture (≈60–80 releases dating to 2014-02-20). Selecting a release switches the tile URL to the corresponding WMTS layer:

```
https://wayback.maptiles.arcgis.com/arcgis/rest/services/World_Imagery/WMTS/1.0.0/default028mm/MapServer/tile/<release>/{z}/{y}/{x}
```

Wayback is unauthenticated. The selected release persists in `localStorage`. The audit-relevant use is corroborating a Class AB or node-disconnect candidate against several historical captures: if 2014 imagery already shows the road connected and OSM still has the gap, the candidate is high-confidence.

---

## Quick start

Requires Python 3.11+.

```bash
pip install -r requirements.txt
python3 tiger_audit.py                              # default: Blue Ash / Montgomery
python3 tiger_audit.py --zone springdale_sharonville
python3 tiger_audit.py --zone all                   # all four zones + combined view
```

Outputs land in `tiger_audit_<zone_key>/` next to the script. The pipeline anchors all writes and cross-zone reads to the script's directory, so the working directory at invocation time does not affect output location. Raw Overpass JSON is timestamped; re-runs accumulate snapshots without overwriting historical data, with snapshots older than 14 days auto-pruned to keep the three most recent per zone.

### Offline reproducibility

Three flags support offline reruns and self-verification:

```bash
python3 tiger_audit.py --zone all --from-cache              # skip API; use newest cached JSON per zone
python3 tiger_audit.py --zone blue_ash_montgomery --from-csv # rebuild from CSVs (skip fetch + classify)
python3 tiger_audit.py --zone all --from-cache --self-test   # offline rerun + count check vs reference CSV
```

`--from-cache` and `--from-csv` are mutually exclusive sources. `--self-test` compares per-zone counts against `tiger_audit_all_zones/all_zones_summary.csv` and exits 0 on full match, 1 on any mismatch. With `--from-csv`, dashboard maps render blank (CSVs do not carry coordinates) but tabular outputs and counts are fully functional.

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

System-wide totals after deduplication across all four zones: 6,096 unique unreviewed ways, 221 Class AB defects, 522 multi-segment streets, 710 unique probable node disconnects. See [`tiger_audit_all_zones/all_zones_summary.csv`](tiger_audit_all_zones/all_zones_summary.csv) for the per-zone breakdown plus the per-zone-sum row (preserving overlap duplicates) and the deduplicated unique row.

A reference run is committed under [`tiger_audit_blue_ash_montgomery/`](tiger_audit_blue_ash_montgomery/). To open the dashboard locally:

```
tiger_audit_blue_ash_montgomery/reports/TIGER-Audit-Blue-Ash-Montgomery-Dashboard.html
```

---

## Pipeline

1. **Fetch.** A single Overpass query per zone, with retry against the kumi.systems mirror on transient failure. The query emits `out tags geom;` so polylines arrive inline. Falls back to the most recent locally cached snapshot (sorted by mtime; warned past 14 days) only on full upstream failure. JSON-`null` responses are retryable. A non-dict payload from either fetch path raises a source-labelled `RuntimeError` with a bounded snippet of the offending payload.
2. **Classify.** Group ways by case-insensitive normalised name, apply the four disjoint class predicates, then run the 30 m endpoint-distance heuristic across all Class B streets and collapse pairwise candidates within 5 m on the same street to a single representative.
3. **XLSX.** `openpyxl` produces the eight-sheet workbook. Totals, percentages, and volunteer-hour columns are real Excel formulas, not pre-computed values.
4. **Dashboard HTML.** Single self-contained file; defect data is embedded as compact JSON; Leaflet, Leaflet.heat, and Leaflet.markercluster are loaded from CDN.
5. **CSVs and per-zone README.** Four sorted CSVs (the multi-segment slice carries `_street_gap_count`), a per-zone re-run README, and a console summary with the gap count.
6. **Combined dashboard (`--zone all` only).** Cross-zone deduplication along the three axes described under [Outputs](#outputs); a combined Leaflet dashboard and `all_zones_summary.csv` are written to `tiger_audit_all_zones/`.

The pipeline is one Python file: [`tiger_audit.py`](tiger_audit.py).

---

## Conventions and invariants

- **No data hallucination.** Every count, list, and chart series traces to the live Overpass response. A query returning 1,800 elements is reported as 1,800.
- **Original tag values preserved.** Street names, highway types, and `oneway` values are passed through verbatim from OSM. Normalisation is grouping-only.
- **Missing tags handled.** Anonymous ways are still classified (a residential way with `oneway=yes` is Class A regardless of name), shown as `[Unnamed]` in displays, and never grouped into Class B.
- **Idempotent re-runs.** XLSX, dashboard, CSVs, and per-zone README are overwritten in place; only raw JSON snapshots accumulate, gated by the 14-day auto-prune.
- **Filename convention.** Generated artefacts use hyphenated names (`TIGER-Audit-Blue-Ash-Montgomery-Dashboard.html`). Underscored filenames italicise inside Markdown / Slack / Discord renderers and corrupt shared URLs; hyphens survive every channel.

---

## Editing OSM

The pipeline produces no mutation. Each defect carries deep links a human editor can use:

- **iD editor** (browser): every way and gap popup links to `openstreetmap.org/edit?editor=id#map=18/<lat>/<lon>` zoomed to the defect.
- **JOSM Remote Control**: every popup also links to `localhost:8111/load_and_zoom?…`. Requires JOSM running locally with Remote Control enabled (Preferences → Remote Control). The dashboard's `Load in JOSM` button batches all visible way IDs into chunked `load_object` requests under the ~8 KB URL limit.

If you fix a defect, removing `tiger:reviewed=no` (or upgrading to `tiger:reviewed=yes` if your team uses the latter) is the conventional way to drop the segment from subsequent audit runs.

---

## Layout

```
.
├── README.md
├── CHANGELOG.md
├── LICENSE
├── requirements.txt
├── tiger_audit.py                          # the entire pipeline
├── author.html                              # methods writeup
├── index.html                              # GitHub Pages landing → redirects to BAM dashboard
├── .nojekyll                               # tells GitHub Pages to skip Jekyll processing
├── tiger_audit_blue_ash_montgomery/        # per-zone outputs, one directory per audited zone
│   ├── README.md
│   ├── csv/
│   │   ├── all_ways.csv
│   │   ├── class_a_false_oneway.csv
│   │   ├── class_ab_compound.csv
│   │   └── class_b_multi_segment.csv      # carries _street_gap_count
│   └── reports/
│       ├── TIGER-Audit-Blue-Ash-Montgomery.xlsx
│       └── TIGER-Audit-Blue-Ash-Montgomery-Dashboard.html
├── tiger_audit_springdale_sharonville/     # same shape; produced by --zone springdale_sharonville
├── tiger_audit_northgate_mt_healthy/       # same shape; produced by --zone northgate_mt_healthy
├── tiger_audit_forest_park_pleasant_run/   # same shape; produced by --zone forest_park_pleasant_run
├── tiger_audit_all_zones/                  # only after a --zone all run
│   ├── TIGER-Audit-All-Zones-Dashboard.html
│   └── all_zones_summary.csv
└── design_handoff_tiger_audit_map/         # design reference (2D + 3D prototypes,
    ├── README.md                           #   high-fidelity tokens, screenshots);
    ├── CAPTURE-MEDIA.md                    #   not production code — intended for
    ├── prototypes/                         #   a future port into a modern stack
    │   ├── map_2d.html
    │   ├── map_3d.html
    │   ├── data.json
    │   └── ways.geojson
    └── screenshots/
```

The raw Overpass JSON snapshots in `tiger_audit_*/data/` are gitignored; they are ~1 MB each and reproducible from any current run. The pipeline auto-prunes snapshots older than 14 days, retaining the three most recent per zone.

---

## Methodological constraints

The flagged population is a lower bound on the true defect set, not a complete enumeration. Five constraints structure the gap between the two: heuristic-vs-ground-truth (presence of `tiger:reviewed=no` records absent verification, not present error); name-string equivalence for Class B grouping (which can conflate two unrelated same-named streets within a single zone bbox); endpoint-only gap detection (mid-way disconnects are outside discrimination); the cross-zone-boundary blind spot (a disconnect whose two ways lie strictly in different zones is detected by no per-zone audit); and cache-fallback freshness (cache-age relative to recent OSM activity is not enforced). The full discussion is in [`author.html`](author.html) §5.1.

---

## Acknowledgements

Road data © [OpenStreetMap contributors](https://www.openstreetmap.org/copyright), ODbL. TIGER/Line: U.S. Census Bureau, public domain. Tile imagery on the live dashboards: CARTO Voyager (OSM-derived), Esri World Imagery and Wayback (public tiers). Independent work; no affiliation is asserted with SORTA, Via Transportation, or the OpenStreetMap Foundation.
