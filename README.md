# Tiger — TIGER/Line Defect Audit for SORTA MetroNow

A read-only OpenStreetMap audit pipeline that finds routing-breaking defects in
unreviewed 2007–2008 TIGER/Line Census road segments inside Hamilton County, OH
microtransit zones operated by SORTA's [MetroNow](https://www.go-metro.com/metronow)
service (powered by [Via Transportation](https://ridewithvia.com/)).

It produces a publication-ready Excel workbook, an interactive Leaflet dashboard
(road geometry, node-disconnect markers, JOSM/iD edit links), and supporting
CSVs that human OSM editors can act on.

> **The pipeline never modifies OSM.** It is strictly an audit. Every output
> traces back to a single live Overpass API query.

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
any pair within **30 m** that aren't already coincident. Each gap shows up as a
purple marker on the dashboard with one-click links to JOSM (via Remote
Control) or the iD editor at the gap location.

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
  data/<zone_key>_raw_<UTC>.json          # raw Overpass response (timestamped, preserved)
  reports/
    TIGER_Audit_<Zone>.xlsx               # 8-sheet styled workbook
    TIGER_Audit_<Zone>_Dashboard.html     # interactive Leaflet dashboard
  csv/
    all_ways.csv                          # master inventory
    class_a_false_oneway.csv              # residential + oneway=yes
    class_b_multi_segment.csv             # streets with 2+ unreviewed segments
    class_ab_compound.csv                 # intersection of A and B
  README.md                               # re-run instructions + file manifest
```

### XLSX workbook (8 sheets)

1. **Executive Summary** — totals, residential share, defect counts, context.
2. **Class A – Highest Risk** — Class AB streets (compound), grouped by name.
3. **Class A – Moderate Risk** — single-segment Class A only; if "O'Leary
   Avenue" is present in the data it's highlighted as the index case.
4. **Class B – Multi-Segment** — streets with ≥5 segments, sorted by count;
   one-way segments per street called out in red.
5. **All Ways** — the complete inventory, sorted AB → A → B → C then by name.
6. **MetroNow Zones** — all four zones; the audited zone is marked
   `AUDIT COMPLETE` in green.
7. **Work Plan** — phased volunteer-hour estimates with formula-driven totals
   (12 / 4 / 7 / 2 minutes per item for P1 / P2 / P3 / P4).
8. **Overpass Query** — exact query, endpoint, bbox, timestamp.

### Interactive dashboard (single-file HTML)

Open by double-clicking — no server required. Loads Leaflet + leaflet.heat +
leaflet.markercluster from CDN; all defect data is embedded.

> **Windows note:** opening the HTML directly via `file://` works on Chrome,
> Edge, and Firefox; CARTO and Esri tiles load fine. If a corporate proxy
> intercepts CDN requests, or if the browser blocks mixed-content on
> `file://`, run `python -m http.server` from the `reports/` directory and
> open `http://localhost:8000/` instead. JOSM Remote Control is HTTP-only;
> `https://` won't work as the dashboard origin.

- Six-card sidebar (Total / Residential / Class AB / Class A / Class B / Node Gaps).
- Layer toggles per defect class + node-disconnect markers + density heatmap.
- **Multi-source basemap selector** with token support — see below.
- Street-name search.
- Live viewport stats (segments/streets/gaps in the current view).
- "Export visible" buttons for GeoJSON or CSV — only what's in the current
  viewport, ready to hand to a volunteer for that neighborhood.
- Every way and every gap has a popup with deep links to OSM, the iD editor at
  that location, and JOSM Remote Control.

#### Basemap options

The dashboard's "Basemap" dropdown switches between several free imagery / road
tile sources. The current selection (and any custom URL or token) is persisted
in browser `localStorage` — never embedded in the committed HTML and never sent
anywhere except to the tile server you select.

| Option | Provider | Notes |
|---|---|---|
| CARTO Voyager *(default)* | CARTO | Vector road basemap derived from OSM |
| CARTO Positron | CARTO | Light, low-contrast — good for screenshots |
| OpenStreetMap Standard | OSMF | Standard OSM tiles |
| Esri World Imagery | Esri | Satellite, public-tier |
| Esri World Imagery (Clarity) | Esri | Sharper / Firefly-style imagery |
| USGS National Map — Imagery | USGS | NAIP-derived; very crisp at high zoom in OH |
| USGS National Map — Topo | USGS | Standard USGS topographic tiles |
| Esri World Topographic | Esri | Topo with road labels |
| **Ohio OSIP** *(best-effort)* | OGRIP / Ohio | 6-inch / 15 cm imagery; URL is best-effort and may need adjustment per service availability |
| **Custom (URL + token)** | any | See below |

#### Premium / authenticated content (your USGS / ArcGIS Online token)

The "Premium / Custom" panel accepts a tile URL template and an optional token.
Use it for ArcGIS Online services that require authentication — e.g.
[Wayback Imagery](https://livingatlas.arcgis.com/wayback/), federal Living Atlas
content, or your organization's hosted layers.

1. Paste a URL template with `{z}/{x}/{y}` (or `{z}/{y}/{x}` — Esri uses the
   latter) into the Custom URL box. Examples:
   - Wayback for a specific release date:
     `https://wayback.maptiles.arcgis.com/arcgis/rest/services/World_Imagery/WMTS/1.0.0/default028mm/MapServer/tile/<release>/{z}/{y}/{x}`
   - Any ArcGIS Online hosted MapServer: paste the Service URL +
     `/tile/{z}/{y}/{x}`
2. Paste your ArcGIS token in the second box.
3. Pick "Custom (uses URL + token below)" from the Basemap dropdown.

The dashboard appends `?token=...` (URL-encoded) when fetching tiles. Token and
URL live in `localStorage` only; they are not written to the dashboard HTML, the
git repo, or any committed file. Closing the browser tab keeps them; clearing
site data wipes them.

> **Don't paste org-issued tokens into a dashboard you intend to share.** A
> shared HTML file plus a screen-shared browser session would expose the token
> via `localStorage`. The token field is for your own local browsing.

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

Wayback is free / unauthenticated for normal viewing — no token needed. The
selected release persists in `localStorage`.

The practical use for this audit: when you spot a Class AB or node-disconnect
candidate, flip through 2-3 Wayback dates to confirm the road geometry on the
ground actually disagrees with what's tagged. If imagery from 2014 already
shows the road connected and OSM still has a gap, that's a high-confidence fix.

---

## Generating an ArcGIS Online token

Most options in the basemap dropdown work without authentication. You only need
a token for **paid Living Atlas content, premium subscriber layers, or hosted
services in your AGOL organization**. Two paths depending on how you sign in:

### A) Built-in AGOL account (username + password)

```bash
curl -X POST 'https://www.arcgis.com/sharing/rest/generateToken' \
  --data-urlencode 'username=YOUR_AGOL_USERNAME' \
  --data-urlencode 'password=YOUR_AGOL_PASSWORD' \
  --data-urlencode 'referer=https://localhost' \
  --data-urlencode 'expiration=120' \
  --data 'f=json'
```

PowerShell equivalent:

```powershell
$body = @{
  username   = 'YOUR_AGOL_USERNAME'
  password   = 'YOUR_AGOL_PASSWORD'
  referer    = 'https://localhost'
  expiration = 120
  f          = 'json'
}
Invoke-RestMethod -Uri 'https://www.arcgis.com/sharing/rest/generateToken' `
                  -Method POST -Body $body
```

Returns `{"token": "...", "expires": <unix-ms>, "ssl": true}`. Paste the
`token` value into the dashboard's "ArcGIS token" field. `expiration` is in
minutes; max is `20160` (14 days). Use `referer=https://localhost` so the token
will validate when the dashboard opens from `file://` or localhost — but note
the token is bound to that referer, so don't paste it into a dashboard hosted
elsewhere.

### B) Federated SSO account (e.g. USGS / GeoPlatform via Login.gov)

The `generateToken` endpoint won't accept your federated password, and pasting
`https://*.maps.arcgis.com/sharing/rest/community/self?f=pjson` directly into a
tab often returns `{"error": {"code": 400, "message": "Not logged in."}}` —
that's because federated SSO sessions don't always propagate cookies across
all `*.arcgis.com` subdomains. Use OAuth instead.

**1. One-click button in the dashboard *(easiest)*.**
The dashboard sidebar has a **"Get token via OAuth (signs in with SSO)"**
button. Steps:

  - Type your portal URL into the "Portal URL" field — e.g.
    `https://geoplatform.maps.arcgis.com` (defaults to `https://www.arcgis.com`).
  - Click the button. A popup opens, redirects through your IdP (Login.gov or
    your org's SSO), then displays the token as plain text on the OAuth
    success page.
  - Copy the token, close the popup, paste it into the **ArcGIS token** field.

Under the hood the button opens:

```
https://<your-portal>/sharing/rest/oauth2/authorize
  ?client_id=arcgisonline
  &response_type=token
  &expiration=20160
  &redirect_uri=urn:ietf:wg:oauth:2.0:oob
```

`client_id=arcgisonline` is a **pre-registered built-in OAuth client on every
AGOL portal** — it works regardless of your role (Viewer, Creator, Publisher,
or Admin). You don't need to register a developer app. `expiration=20160` is
14 days (the max for refresh-less tokens).

**2. ArcGIS Pro / ArcMap "Get Tokens".**
If you have ArcGIS Pro installed and signed in, the Portal panel has a "Get
Tokens" action that issues a long-lived token bound to your portal account.

**3. Network-tab harvest.**
While signed in to your org subdomain (e.g. `geoplatform.maps.arcgis.com`),
DevTools → Network → filter `token=` → trigger any API call (avatar menu,
content browse, etc.) → copy `&token=<long>` from any matching request URL.

For all three: the token never has to be saved to disk. Paste it into the
dashboard once; `localStorage` keeps it across reloads of that file in that
browser only.

#### "Not logged in" troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `community/self` returns `{"error": {"code": 400, "message": "Not logged in."}}` even though you signed in via the org's web app | Federated SSO session is scoped to one app/subdomain; cookies aren't sent to other `*.arcgis.com` paths | Use the OAuth button (option 1) — it forces a fresh auth round-trip and bypasses cookie scope entirely |
| OAuth popup shows `Invalid client_id` | Your portal restricts the public `arcgisonline` client | Register a personal OAuth app at `https://developers.arcgis.com/applications/` (Creator role can do this) and substitute its `client_id` |
| Token validates against `community/self` but `ibasemaps-api.arcgis.com` returns 401 | Token's referer policy doesn't match | Use a tile URL on your **org subdomain** (e.g. `https://<org>.maps.arcgis.com/server/...`) instead of the cross-subscription `ibasemaps-api` host |
| Token works in browser but expires after a few hours | Default expiration | Pass `expiration=20160` (14 days) when generating |

### What the strings on the "License Strings" page are NOT for

The Esri Developers dashboard has a page titled **"ArcGIS Maps SDKs for Native
Apps License Strings"** showing `runtimelite,...` and `nativelite,...` values.
**Those are for compiled native apps** (iOS / Android / .NET / Java / Qt) —
they're passed to `ArcGISRuntimeEnvironment.setLicense()` at app startup. They
don't authenticate browser HTTP tile requests and won't work in the dashboard's
token field. Use the `generateToken` / OAuth flows above instead.

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
| Probable node disconnects | 308 |
| Estimated volunteer hours (Class AB only) | 11.0 |
| Estimated volunteer hours (everything) | 69.2 |

A real run is included under
[`tiger_audit_blue_ash_montgomery/`](tiger_audit_blue_ash_montgomery/).
Open the dashboard locally:

```
tiger_audit_blue_ash_montgomery/reports/TIGER_Audit_Blue_Ash_Montgomery_Dashboard.html
```

---

## How it works

1. **Phase 1 — Fetch.** A single Overpass query per zone, with retry against
   the kumi.systems mirror if the primary endpoint fails. The query uses
   `out tags geom;` so polylines come back inline. Falls back to the most
   recent cached snapshot if all endpoints are unreachable.
2. **Phase 2 — Classify.** Group ways by case-insensitive normalized street
   name; apply the Class A / B / AB / C rules above; run the
   30 m endpoint-distance gap detector across all Class B streets.
3. **Phase 3 — XLSX.** `openpyxl` produces the 8-sheet workbook; totals,
   percentages, and volunteer-hour columns are real Excel formulas, not
   pre-computed values.
4. **Phase 4 — HTML dashboard.** Single self-contained file; defect data
   embedded as compact JSON; Leaflet + leaflet.heat loaded from CDN.
5. **Phase 5 — CSVs + README + summary.** Four sorted CSVs, a per-zone
   re-run README, and a console summary box.

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
├── requirements.txt
├── tiger_audit.py                          # the entire pipeline
└── tiger_audit_blue_ash_montgomery/        # example output (committed for reference)
    ├── README.md
    ├── csv/
    │   ├── all_ways.csv
    │   ├── class_a_false_oneway.csv
    │   ├── class_ab_compound.csv
    │   └── class_b_multi_segment.csv
    └── reports/
        ├── TIGER_Audit_Blue_Ash_Montgomery.xlsx
        └── TIGER_Audit_Blue_Ash_Montgomery_Dashboard.html
```

The raw Overpass JSON snapshots in `tiger_audit_*/data/` are gitignored — they
are ~1 MB each and reproducible from any current run.

---

## Acknowledgements

- Road data: © [OpenStreetMap contributors](https://www.openstreetmap.org/copyright), ODbL.
- Tile imagery: CARTO Voyager (OSM-derived), Esri World Imagery.
- Original 2007–2008 import: U.S. Census Bureau TIGER/Line.
- MetroNow operations: SORTA / Via Transportation.

This is a community audit tool. It is not affiliated with SORTA, Via, or the
OpenStreetMap Foundation.
