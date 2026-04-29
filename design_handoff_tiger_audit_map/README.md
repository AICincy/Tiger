# Handoff: TIGER Audit Interactive Map (2D + 3D)

## Overview

An interactive web map for the **AICincy/Tiger** project — a read-only OpenStreetMap audit pipeline that surfaces routing-breaking defects in unreviewed 2007–2008 TIGER/Line road segments inside SORTA MetroNow service zones in Hamilton County, OH.

The map presents 1,934 unreviewed road ways in the Blue Ash / Montgomery zone, color-coded by defect class (AB compound / A false-oneway / B multi-segment / C unreviewed). Volunteers click any segment to see its tags, defect narrative, and one-click jump-to-edit links into OSM, the iD editor, and Overpass Turbo.

Two views ship together:
- **`map_2d.html`** — flat Leaflet map, fast, low-bandwidth, prints cleanly.
- **`map_3d.html`** — MapLibre GL JS 3D view with full pan / tilt / zoom, orbit animation, retina imagery basemaps.

## About the Design Files

The files in `prototypes/` are **design references created in HTML** — working prototypes that show the intended look, behavior, and data flow. **They are not production code to ship as-is.** The task is to recreate these designs inside the target codebase using its established framework, component patterns, and conventions. If no framework exists yet, the recommended target is React + Vite (with MapLibre GL for the 3D view and Leaflet for the 2D view), but any modern stack works.

The prototypes are fully functional — open `prototypes/map_2d.html` or `prototypes/map_3d.html` in a browser (the relative `data.json` / `ways.geojson` must be served, so use `python3 -m http.server` from the `prototypes/` folder rather than `file://`).

## Fidelity

**High-fidelity.** All colors, typography, spacing, layout grids, hover/selection states, and copy are final. Pixel-perfect recreation expected.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ HEADER (56px)  brand · live KPI strip · view-switch buttons │
├──────────────┬──────────────────────────────────────────────┤
│              │                                              │
│   SIDEBAR    │              MAP CANVAS                      │
│   (320px)    │                                              │
│              │   - HUD overlays (camera state, coords)      │
│   sections:  │   - Compass                                  │
│   - classes  │   - Native zoom/pitch controls               │
│   - basemap  │   - Hover halo + click selection             │
│   - PTZ      │                                              │
│   - search   │                                              │
│   - selected │                                              │
│              │                                              │
├──────────────┴──────────────────────────────────────────────┤
│ FOOTER (32px)  zone label · legend · visible-count · stamp  │
└─────────────────────────────────────────────────────────────┘
```

Layout is a **CSS grid** with three rows (`56px / 1fr / 32px`) and two columns (`320px / 1fr`):

```css
#app {
  display: grid;
  grid-template-columns: 320px 1fr;
  grid-template-rows: 56px 1fr 32px;
  grid-template-areas: "head head" "side map" "foot foot";
  height: 100vh;
}
```

On viewports `≤ 760px`, layout collapses to a single column with the map above the sidebar.

---

## Design Tokens

### 2D map (`map_2d.html`) — warm cartographer palette

| Token | Value | Use |
|---|---|---|
| `--bg` | `#f4f0e8` | App background |
| `--paper` | `#ebe4d4` | Header / footer |
| `--panel` | `#fbf8f1` | Sidebar |
| `--ink` | `#1d1c1a` | Primary text |
| `--ink-soft` | `#514c44` | Secondary text |
| `--ink-mute` | `#8a8377` | Tertiary / labels |
| `--rule` | `#c9bfa9` | Borders |
| `--rule-soft` | `#d9d0bc` | Soft dividers |
| `--accent` | `#6b3a2e` | Brand accent (used for selection borders) |
| `--crit` | `#b42a1d` | Class AB / A line color |
| `--high` | `#c8761a` | Class B line color |
| `--low` | `#6e6759` | Class C line color |
| `--gap` | `#4f3aa8` | Node-disconnect markers |

### 3D map (`map_3d.html`) — dark control-room palette

| Token | Value | Use |
|---|---|---|
| `--bg` | `#0f1216` | App background |
| `--panel` | `#161b22` | Sidebar / header / footer |
| `--panel-soft` | `#1d242c` | Inset panels (chips, PTZ buttons) |
| `--rule` | `#2a323d` | Borders |
| `--ink` | `#e8e3d6` | Primary text |
| `--ink-soft` | `#a8a294` | Secondary text |
| `--ink-mute` | `#6b6557` | Tertiary / labels |
| `--accent` | `#d4af37` | Active chip / focus / HUD highlight |
| `--crit` | `#ff5a47` | Class AB / A line |
| `--high` | `#ffa84a` | Class B line |
| `--low` | `#7c8089` | Class C line |
| `--gap` | `#9d7bff` | Reserved for node-gap markers |

### Typography

Both views use the same pairing, loaded from Google Fonts:

```html
<link href="https://fonts.googleapis.com/css2?family=IBM+Plex+Mono:wght@400;500;600&family=IBM+Plex+Sans:wght@400;500;600;700&display=swap" rel="stylesheet">
```

| Role | Family | Weight | Size | Tracking |
|---|---|---|---|---|
| App body | IBM Plex Sans | 400 | 13px / 1.5 | normal |
| Header H1 | IBM Plex Sans | 600 | 13px | 0.04em uppercase |
| Sidebar section H2 | IBM Plex Sans | 600 | 10px | 0.12em uppercase |
| Selection name | IBM Plex Sans | 600 | 16px | normal |
| KPI value | IBM Plex Mono | 500 | 17px | tabular-nums |
| Coordinate / data values | IBM Plex Mono | 400 | 11px | tabular-nums |
| HUD readouts | IBM Plex Mono | 400 | 10px | uppercase |
| Class label name | IBM Plex Sans | 500 | 12px | normal |
| Class label desc | IBM Plex Sans | 400 | 11px | normal |

`font-feature-settings: "tnum"` is applied to all `.mono` text so numbers don't jitter as values change.

### Spacing scale

Used throughout: `4 · 6 · 8 · 10 · 12 · 14 · 18 · 22 · 32 px`. No 16px multiples — the look is intentionally a notch denser than typical web UIs.

### Borders & shadows

- All borders are `1px solid var(--rule)` or `1px dotted var(--rule-soft)` for inset dividers.
- **No border-radius** anywhere except the compass (full circle) and PTZ orb pad icons. Square edges everywhere; this is a deliberate cartographic / instrument-panel feel.
- Only one subtle box-shadow exists, on the compass: `0 1px 0 rgba(0,0,0,0.04)` in 2D.
- 3D HUD panels use `backdrop-filter: blur(6px)` over `rgba(22,27,34,0.92)`.

---

## Data Model

Two data files live next to the HTML and are fetched at load:

### `data.json` (used by 2D map)

```ts
{
  summary: {
    total: 1934,
    byClass: { AB: 55, A: 2, B: 1009, C: 868 },
    byHighway: { residential: 1163, secondary: 320, tertiary: 314, ... },
    bounds: { minLat, maxLat, minLon, maxLon },
    generated: "2026-04-29T13:06:30Z"
  },
  features: Array<{
    id: number,            // OSM way ID
    name: string,          // street name (may be "")
    hw: string,            // highway= tag value
    ow: string,            // oneway= tag value (yes / no / "")
    sf: string,            // surface
    ln: string,            // lanes
    sp: string,            // maxspeed
    nb: string,            // tiger:name_base
    nt: string,            // tiger:name_type
    zl: string,            // tiger:zip_left
    cls: 'AB' | 'A' | 'B' | 'C',
    sev: 'CRITICAL' | 'HIGH' | '',
    segs: number,          // (B/AB only) total OSM ways for this named street
    gaps: number,          // (B/AB only) detected node-disconnect count
    g: Array<[lat, lon]>,  // polyline points, 5-decimal precision
  }>
}
```

### `ways.geojson` (used by 3D map)

Standard GeoJSON `FeatureCollection` with `LineString` geometries. Every feature carries the same properties as above plus `rank` (4=AB, 3=A, 2=B, 1=C) for line-sort-key ordering. The top-level object also carries `summary` (same shape as above).

### Defect classification rules

These come from the audit pipeline (`tiger_audit.py` in the source repo) and must be preserved verbatim — the map is just a viewer:

| Class | Rule | Severity | Color |
|---|---|---|---|
| **A** | `highway=residential` AND `oneway=yes` | CRITICAL | `--crit` |
| **B** | named street with ≥2 unreviewed segments | HIGH | `--high` |
| **AB** | A AND B (compound) | CRITICAL | `--crit` |
| **C** | everything else with `tiger:reviewed=no` | LOW | `--low` |

Default visible classes: `AB, A, B`. Class C is **off by default** to keep critical defects readable.

---

## Components — 2D map

### `<Header>`
- 56px tall, full width, `--paper` background, 1px bottom rule.
- Left: 320px column (border-right) with H1 "Blue Ash · Montgomery" and a 11px IBM Plex Mono subtitle "TIGER audit · OSM unreviewed segments".
- Center: KPI strip with `Total ways / Class AB / Class A / Class B / Class C`. Each stat is two stacked lines: 10px uppercase label (mute) over 17px mono value. AB and A values use `--crit`, B uses `--high`.
- Right: action buttons `Fit bounds`, `Critical only`, `?` — 11px caps, 1px ruled, hover toggles to inverted (ink bg, paper text).

### `<DefectClassFilter>`

A list of four rows. Each row is a 3-column grid `16px 1fr auto`:

```
┌────┬────────────────────────────────┬──────┐
│ ▬▬ │ AB · Compound                  │   55 │
│    │ False oneway + multi-segment   │      │
└────┴────────────────────────────────┴──────┘
```

- Swatch is a 16px-wide stripe whose color and thickness encodes the line style on the map (AB/A: 5px thick crit, B: 5px thick high, C: 3px thick low).
- Name row: 12px Plex Sans 500.
- Desc row: 11px Plex Sans 400 mute.
- Count: 11px mono in a 1px-ruled chip with `--bg` background.
- Rows are toggleable; off rows fade to `opacity: 0.4` and the swatch desaturates to `--ink-mute`.
- Inputs are visually hidden — the entire row is the click target via `<label>`.

### `<BasemapPicker>`
- Horizontal segmented row of equal-flex buttons separated by 1px rules.
- Active button: ink background, paper text, no border-radius.
- 2D map ships three: `None / Positron / OSM` (extend to a second row matching the 3D version if desired).

### `<Search>`
- Single 12px text input, 1px ruled, transparent until focus where border darkens to `--ink`.
- Below: vertical list of grouped results (one entry per unique street name). Each result shows the street name on the left and `Nx CLS` (e.g. `12× AB`) on the right in 10px mono mute.
- Hover state adds a 2px left border in `--accent`.
- Min query length: 2.

### `<SelectionPanel>`
- Hidden when nothing is selected. An empty-state placeholder is shown instead with a `←` arrow in `--accent` and instructional copy.
- When populated:
  - **Head** (12px padding, `--bg` background, 1px bottom rule): street name (16px Plex Sans 600) + `Class XX` chip (10px mono caps with letter-spacing 0.1em, white text on the class color).
  - **Body** (12/14px padding):
    - **Definition list** (`<dl class="kv">`) — 90px label / 1fr value grid. Labels 10px mute uppercase, values 11px mono.
    - **Narrative** — 1.55 line-height paragraph in `--panel-soft`, with a 2px left accent rule. Copy varies by class:
      - AB: "Both defects present: TIGER source records this as a two-way residential street, but OSM tags it **oneway=yes**, AND the street is fragmented into N separate ways with N node disconnects. Verify on imagery and merge."
      - A: "Residential street tagged **oneway=yes** against TIGER baseline. Likely a stale import artifact — check Wayback imagery for residential cul-de-sac patterns."
      - B: "Continuous TIGER-named street fragmented across **N** OSM ways with **N** node-connectivity gaps. Candidate for ways-merge audit."
      - C: "No specific defect class assigned. Marked tiger:reviewed=no — included for completeness."
    - **External links** — three small ruled buttons:
      - `OSM ↗` → `https://www.openstreetmap.org/way/{id}`
      - `iD edit ↗` → `https://www.openstreetmap.org/edit?way={id}`
      - `Overpass ↗` → `https://overpass-turbo.eu/?Q=way({id});out%20geom;&R`

### `<MapCanvas>` (Leaflet)
- Init with `L.map('map', { minZoom: 10, maxZoom: 21, zoomSnap: 0.5, zoomDelta: 0.5, wheelPxPerZoomLevel: 80 })`.
- Default fit-bounds to the data envelope with 30px padding.
- Custom-skinned controls — zoom buttons get `--panel` bg with 1px `--rule` border, no border-radius, 28px square.
- Attribution restyled to translucent paper with 9px mono mute text.
- Polylines drawn with `L.polyline` using class-based style sets:
  ```js
  STYLES = {
    crit: { color: '#b42a1d', weight: 3,   opacity: 0.95 },
    high: { color: '#c8761a', weight: 2.5, opacity: 0.9  },
    low:  { color: '#6e6759', weight: 1,   opacity: 0.55 }
  };
  ```
  Hover state: black stroke, weight + 2, opacity 1. Selected: black stroke, weight + 3, opacity 1, brought to front.
- **Z-order matters** — sort features so C draws first, then B, A, AB on top.

### Map-canvas overlays (corner widgets)
- **Compass** (top-right): 56px round paper disc with 1px rule. Hand-drawn SVG: outer ring, 4 cardinal ticks, red north arrow, ink south arrow, "N" label. Static (no rotation in 2D).
- **Scale block** (bottom-left): 150px-wide paper card with `Scale` 9px mute caps label, then a 4-segment alternating black/white scale bar with 1px ink border, then `0 / half / full` ticks in 9px mono. Recomputed on every `moveend` from the meridian width of the visible bounds.
- **Coord readout** (bottom-right): paper card with `lat / lon / z` labels in mute caps and live mono values updating on `mousemove`.

### `<Footer>`
- 32px tall, paper background, 1px top rule.
- Left: zone label (`BLUE ASH / MONTGOMERY · HAMILTON CO., OH`).
- Inline legend with 4 stripes/dots colored to match the line classes.
- Filler.
- Visible-count (`1,066 of 1,934 visible`).
- OSM extract timestamp.
- All 10px mono mute.

---

## Components — 3D map (additional + variant)

The 3D map keeps the same sidebar structure but on a dark palette and adds:

### `<BasemapPicker>` — chip grid
- 3 chips per row across multiple rows. Chip is `var(--panel-soft)` bg, 1px `--rule`, 10.5px font, `flex: 1 1 calc(33% - 4px)`.
- Active chip: `--accent` (gold) bg with `#1a1410` text and 600 weight.
- Sources (URL templates and tile-size are critical — copy verbatim):
  - **dark** — `https://{a|b}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}@2x.png` · tileSize 512 · maxzoom 20
  - **positron** — `https://{a|b}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}@2x.png` · tileSize 512 · maxzoom 20
  - **osm** — `https://tile.openstreetmap.org/{z}/{x}/{y}.png` · tileSize 256 · maxzoom 19
  - **esri-imagery** — `https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}` · tileSize 256 · maxzoom 19
  - **esri-clarity** — `https://clarity.maptiles.arcgis.com/arcgis/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}` · tileSize 256 · maxzoom 18 *(maxzoom must be 18 — going higher returns blank tiles)*
  - **usgs** — `https://basemap.nationalmap.gov/arcgis/rest/services/USGSImageryOnly/MapServer/tile/{z}/{y}/{x}` · tileSize 256 · maxzoom 18
  - **osip** — `https://gis5.oit.ohio.gov/arcgis/rest/services/OSIP_3/OSIP_2021_Pictometry_18in/MapServer/tile/{z}/{y}/{x}` · tileSize 256 · maxzoom 21
  - **esri-topo** — `https://server.arcgisonline.com/ArcGIS/rest/services/World_Topo_Map/MapServer/tile/{z}/{y}/{x}` · tileSize 256 · maxzoom 19
- All raster layers must use `paint: { 'raster-resampling': 'linear' }` for smoother overzoom.

### `<PTZPad>` (Camera section)
- 3×3 button grid for directional pan + recenter + a "full-row" zoom-in button + a row with zoom-out and "STREET (Z19)".
- Each button: `var(--panel-soft)` bg, 1px rule, mono 12px symbols.
- Buttons map to:
  - `↑/↓/←/→` → `map.panBy([±120, 0 or ±120])`
  - `⊙` recenter → `map.fitBounds(envelope, { padding: 60, pitch: 55, bearing: -18, duration: 800 })`
  - `+ ZOOM IN` → `map.zoomTo(zoom + 1, { duration: 350 })`
  - `− ZOOM OUT` → `map.zoomTo(zoom - 1, { duration: 350 })`
  - `STREET (Z19)` → `map.easeTo({ zoom: 19, duration: 700 })`
- Below the pad, four labeled sliders (Tilt 0–85°, Bearing −180–180°, Zoom 11–20, Lines opacity 20–100%). Each is a row with: 10px mute caps label · `<input type="range">` · 11px mono live readout. `accent-color: var(--accent)` colors the slider.
- Sliders are **bidirectional** with the camera — moving one updates the camera; the camera's `move` event also writes back into the sliders so they always reflect current state.

### `<HUD>`
- **`#hud-cam`** (top-left): blurred translucent dark panel with 4 rows showing `CAM / ZOOM / PITCH / BEARING`. Labels in 9px mute caps, values in `--accent` mono.
- **`#hud-coord`** (bottom-right): `LAT / LON` live readout under cursor.
- **`#compass-3d`** (top-right): 64px round panel with an SVG needle that rotates with `transform: rotate(-bearing)deg` on every camera move. **Click resets bearing and pitch to 0** (north-up flat).

### Map init (MapLibre GL JS 4.7.1)
```js
map = new maplibregl.Map({
  container: 'map',
  style: makeStyle('esri-imagery'),
  center,
  zoom: 12,
  pitch: 55,
  bearing: -18,
  maxZoom: 22,
  minZoom: 10,
  antialias: true,
  pixelRatio: Math.max(window.devicePixelRatio || 1, 2),  // force HiDPI
});
map.addControl(new maplibregl.NavigationControl({ visualizePitch: true }), 'bottom-right');
map.addControl(new maplibregl.ScaleControl({ unit: 'imperial' }), 'bottom-left');
```

**Important boot sequencing**: bind both `load` and `style.load` to a `boot()` function gated by a `booted` flag, plus a 1.5s + 4s setTimeout fallback. Some glyph/tile racing in MapLibre 4.7.1 can prevent `load` from firing on its own with a minimal style. **Do not include a `glyphs` URL** in the style spec — there are no text labels and a missing/slow glyph endpoint will block style.load indefinitely.

### Layers (rendered in this order)
1. `bg` — background (always first), `#0f1216`.
2. `bm` — raster basemap (only if not "none"). Inserted before `ways-casing`.
3. `ways-casing` — black 0.55-opacity casing under critical lines, width interpolated 12→18 zoom, AB/A: 3→9, B: 2→6, C: 1→3.
4. `ways-line` — colored lines, `line-sort-key: ['get','rank']` so AB renders on top:
   ```js
   'line-color': ['match', ['get','cls'],
     'AB', '#ff5a47', 'A', '#ff5a47', 'B', '#ffa84a', '#7c8089'],
   'line-width': ['interpolate', ['linear'], ['zoom'],
     12, ['match', ['get','cls'], 'AB', 2, 'A', 2, 'B', 1.4, 0.6],
     16, ['match', ['get','cls'], 'AB', 4, 'A', 4, 'B', 3, 1.2],
     20, ['match', ['get','cls'], 'AB', 9, 'A', 9, 'B', 7, 3]],
   ```
5. `ways-hl` — highlight layer above everything, filtered by `['==', ['id'], -1]` until a feature is hovered or selected. White stroke, weight 8, line-blur 0.5.

The GeoJSON source uses `promoteId: 'id'` so MapLibre's feature-state and id-based filters work against the OSM way ID directly.

---

## Interactions & Behavior

### Click selection (both views)
1. User clicks a polyline.
2. Previous selection (if any) reverts to its base style.
3. New polyline gets selected style (black thick stroke / white halo) and is brought to front.
4. Selection panel populates with that feature's data.
5. If current zoom < 16 (2D) or < 16.5 (3D), the map fits the segment's bounding box with a 60–100px padding and `maxZoom: 17–18.5`. In 3D it also applies `pitch: 60` and `duration: 700–900ms`.

### Class toggling
- Toggling a class updates `visibleClasses` Set and either:
  - 2D: hides each polyline via `display: none` on the SVG element.
  - 3D: applies `setFilter` on `ways-line` and `ways-casing` with `['in', ['get','cls'], ['literal', [...visibleClasses]]]`.
- Footer's `visible-count` is recomputed.

### Search
- Debounced not required (cheap operation). On `input`:
  1. Trim, lowercase, return early if length < 2.
  2. Group features by name; for each group compute the "worst" class.
  3. Render up to 30 results, sorted alphabetically.
  4. Click on a result: fit bounds to all matching features (`maxZoom: 17–18`, padding 60–100, 3D adds `pitch: 60`), then select the first feature.

### Header buttons
- **Fit bounds**: `map.fitBounds(envelope, { padding: 60, pitch: 55, bearing: -18 })` (3D) or `map.fitBounds(envelope, { padding: [30,30] })` (2D).
- **Critical only**: filter to `cls === 'AB' || cls === 'A'`, fit to that subset.
- **Orbit** (3D only): toggles a `setInterval(30ms, () => bearing += 0.6)` animation. Button label flips between `Orbit` and `Stop`. HUD CAM mode flips between `PTZ` and `ORBIT`.
- **`?`** (2D only): native `alert()` with class definitions and interaction hints. Replace with a proper modal in production.
- **`⊟ 2D View`** link in 3D header: navigates to `map_2d.html` (relative). Recreate as a route or button to the 2D view component.

### Hover state (3D)
- Cursor switches to `pointer` over `ways-line`.
- Hovered feature ID is set into the `ways-hl` filter while no permanent selection is held.

### Compass click (3D)
- `map.easeTo({ bearing: 0, pitch: 0, duration: 700 })` — flattens to north-up.

---

## State Management

Minimal global state — no need for Redux/Zustand at this size. Recommended React shape:

```ts
type Cls = 'AB' | 'A' | 'B' | 'C';

interface AppState {
  data: FeatureCollection | null;        // loaded once at boot
  visibleClasses: Set<Cls>;              // default: {AB, A, B}
  activeBasemap: string;                 // 'esri-imagery' default for 3D, 'none' for 2D
  selectedId: number | null;
  searchQuery: string;
  // 3D-only:
  camera: { zoom: number, pitch: number, bearing: number };
  orbiting: boolean;
  lineOpacity: number;                   // 0.2–1.0
}
```

The MapLibre / Leaflet instance is held in a ref (not state) — its imperative API drives camera changes, and a `move` event listener writes back into `camera` state for the slider readouts. **Don't put the map instance in React state — re-render storms.**

### Data fetching
- Both files fetch their data at startup with a single `fetch()` to `data.json` or `ways.geojson`. ~720 KB minified. Could be code-split or moved to a tile server later, but at this size in-memory is fine.
- The audit pipeline regenerates these files; treat them as build-time inputs, not user uploads.

---

## Responsive

- **`> 760px`** — full grid (sidebar + map).
- **`≤ 760px`** — sidebar drops below the map (`grid-template-areas: "head" "map" "side" "foot"` with 240px sidebar height). KPI strip and 3D HUD/compass hidden. Touch gestures on the map handle pinch zoom; 3D tilt via two-finger drag.

No mobile-specific design has been polished — verify before public launch on real devices.

---

## Assets

- **Fonts**: IBM Plex Sans + IBM Plex Mono (Google Fonts CDN).
- **Map libraries**:
  - 2D: Leaflet 1.9.4 (`unpkg.com/leaflet@1.9.4/dist/leaflet.{js,css}`).
  - 3D: MapLibre GL JS 4.7.1 (`unpkg.com/maplibre-gl@4.7.1/dist/maplibre-gl.{js,css}`).
- **Icons / illustrations**: none. The compass is hand-rolled inline SVG (paste verbatim from `prototypes/map_2d.html` / `prototypes/map_3d.html`). No icon font, no SVG icon library.
- **Tile imagery**: see Basemap section. All sources are public; **OSIP HD requires a US-based egress IP** (Ohio state service). When deploying behind a CDN, allowlist or proxy as needed.
- **Source data**: from the [AICincy/Tiger](https://github.com/AICincy/Tiger) repo's Overpass extract (timestamped raw JSON in `tiger_audit_blue_ash_montgomery/data/`). Re-run the Python pipeline to refresh.

---

## Implementation Notes for Claude Code

1. **Stack recommendation**: React 18 + Vite + TypeScript. Components: `<App>`, `<Header>`, `<Sidebar>` (with sub-components `<DefectClassFilter>`, `<BasemapPicker>`, `<PTZPad>`, `<Search>`, `<SelectionPanel>`), `<MapCanvas2D>`, `<MapCanvas3D>`, `<HUD>`, `<Compass>`, `<Footer>`. Use `react-leaflet` for the 2D map; use raw `maplibre-gl` (not a wrapper) for the 3D map — wrappers lag the underlying lib.
2. **Routing**: two routes (`/` → 2D, `/3d` → 3D) sharing the same data fetch and sidebar state. The header's view-switch button should preserve selection (deep-linkable: `?way={id}` query param).
3. **Style system**: define the two palettes as CSS custom properties under `:root` and `:root[data-theme="dark"]`, gated by which view is active. Don't try to share components between views with conditional theming — they're cousins, not twins, and the 3D layout has extra controls.
4. **Performance**: at 1,934 features, both maps render fast with no virtualization. If the dataset grows past ~10k features, switch the 2D map to `L.canvas()` renderer and the 3D map to vector tiles (build with `tippecanoe`).
5. **Accessibility**: keep the implicit `<label>` wrapping on class toggles. The PTZ pad needs `aria-label`s on directional buttons. The selection panel should have a focusable close button (currently selection persists until another segment is clicked — add an `Escape` keyboard shortcut to clear).
6. **URL state**: persist `?way={id}&bm={key}&classes=AB,A,B&pitch=55&bearing=-18&zoom=14` so users can share specific defect views — this is the user's stated near-term goal.
7. **Self-contained build for static hosting**: both prototypes fetch their data files via relative URL. For a single-file deploy, inline the JSON via `<script type="application/json" id="data">`.
8. **Don't change the audit semantics**: `cls`, `sev`, `segs`, `gaps`, and the narrative copy are part of the contract with the audit pipeline. The map is read-only.

---

## Files

```
prototypes/
  map_2d.html        # 2D Leaflet view (full prototype, ~31 KB)
  map_3d.html        # 3D MapLibre PTZ view (full prototype, ~35 KB)
  data.json          # 720 KB — feature inventory used by 2D
  ways.geojson       # GeoJSON FeatureCollection used by 3D
screenshots/
  2d-01-default.png            # default fit-bounds view, no basemap
  2d-02-critical-only.png      # zoom on the AB+A critical envelope
  2d-03-positron-basemap.png   # CARTO Positron basemap underneath
CAPTURE-MEDIA.md     # paste-in console scripts to capture 3D screenshots + an orbit GIF from the live prototype
README.md            # this document
```

The 3D screenshots and the orbit-sweep GIF/video have to be captured against the live prototype (browser DevTools console, not headless) — see `CAPTURE-MEDIA.md` for one-paste scripts that do it. Both prototypes are fully working when served (e.g. `python3 -m http.server` from `prototypes/`).
