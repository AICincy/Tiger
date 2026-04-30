# Changelog

All notable changes to the TIGER/Line audit pipeline.

## [Unreleased]

_Empty — next batch of changes lands here._

## [2026-04-30] — Reproducibility CLI flags

### `--from-cache`
- New CLI flag that skips the live Overpass API fetch and reads directly
  from the newest cached JSON in `data/`. `fetch_overpass()` gains a
  `force_cache` keyword parameter; when `True`, the entire retry loop is
  bypassed and the function goes straight to the mtime-sorted cache
  lookup. Prints `--from-cache: using {path} (no API call)`.
- Threaded from `main()` → `run_zone()` → `fetch_overpass()`.

### `--from-csv`
- New CLI flag (mutually exclusive with `--from-cache`) that rebuilds
  the pipeline's internal data structures from previously written CSVs
  (`all_ways.csv`, `class_b_multi_segment.csv`), skipping both the
  Overpass fetch and the classify phase entirely.
- New function `classify_from_csvs(csv_dir)` reads the CSVs via
  `csv.DictReader`, reconstructs way records and `class_b_streets`
  grouping, recomputes `summary_stats`, and returns a dict matching
  the exact shape of `classify()`.
- Geometry is set to empty lists (CSVs don't carry coordinates);
  dashboard maps will be blank but all tabular outputs are fully
  functional. Gap detection is likewise empty since it requires
  geometry.

### `--self-test`
- New CLI flag (can combine with `--from-cache` or `--from-csv`).
- After all zone runs complete, reads
  `tiger_audit_all_zones/all_zones_summary.csv` as a reference and
  compares each zone's run-output counts against the CSV values.
- Prints a comparison table and exits 0 on full match, 1 on any
  mismatch.
- Canonical invocation:
  `python tiger_audit.py --zone all --from-cache --self-test`

## [2026-04-29] — Author suite

### Graduate-register rewrite of `author.html` and `README.md`; design handoff (PR #15)
- `author.html`: prose register raised across §1–§6 (passive-leaning,
  denser, less rhetorical); abstract recast as a position-and-contribution
  paragraph; §7 ("About the Author") replaced by a minimal title-block
  byline; footnotes tightened. Visual: serif body
  (Charter / Iowan Old Style), sans-serif headings (Inter / system
  stack), Tufte-leaning table grammar (top + bottom rules only,
  tabular numerals on numeric columns), lighter section breaks.
  Mobile rules preserved.
- New Appendix A: design-handoff reference with a 2D-prototype
  screenshot inline and links to the live prototype URLs.
- `README.md`: matching register; "Why this exists" replaced by a
  "Position" section; Scope, defect-classification, and
  node-disconnect-detection sections written formally; outputs and
  layout sections preserved (file-tree content unchanged); Sample-run
  section retained verbatim (numbers unchanged); new "Methodological
  constraints" cross-link to `author.html` §5.1. Layout tree updated
  to include `design_handoff_tiger_audit_map/`.
- New `design_handoff_tiger_audit_map/` directory: high-fidelity
  design reference for an alternative cartographic presentation —
  2D Leaflet and 3D MapLibre prototypes, screenshots, design-token
  tables. Reference-only; the production port (recommended target
  React + Vite per the handoff README) is out of scope for this
  repo's current pipeline.

### Tabbed UX for `author.html` (PR #16)
- Long-scroll layout replaced with a six-tab interface: Overview /
  Methods / Results / Discussion / Map / Appendix & Notes. §1–§6
  numbering preserved inside the body for citation; tabs group
  sections into reading-paced units rather than mirroring every
  numbered section one-for-one.
- Sticky title block + sticky tab strip; the strip scrolls
  horizontally on narrow viewports without a visible scrollbar.
- Map tab embeds the live combined four-zone dashboard
  (`tiger_audit_all_zones/...Dashboard.html`) via a full-width iframe;
  a meta strip below links to the standalone view.
- ARIA: `role="tablist" / "tab" / "tabpanel"`, `aria-selected` and
  `tabindex` flip on activation, arrow-key / Home / End navigation
  across the strip, visible focus ring.
- Footnote sup-links auto-switch to the Appendix & Notes tab and
  scroll-and-highlight the targeted footnote.
- URL hash carries the active tab (`#overview` … `#appendix`); deep
  links to footnotes (`#fn1` …) open the Appendix tab and scroll.
- `@media print` expands every panel into a continuous document and
  hides the tab strip, so paper output is identical to the prior
  long-scroll version.
- New `design_handoff_tiger_audit_map/index.html` landing page so
  `/design_handoff_tiger_audit_map/` resolves to a 2D-vs-3D chooser
  rather than a directory listing. `map_3d.html`:
  `preserveDrawingBuffer: true` added to the MapLibre constructor so
  the `CAPTURE-MEDIA.md` capture script works without a manual edit.

### Author follow-ups (PR #17)
- Tab switches use `history.pushState` (was `replaceState`) so the
  browser back/forward buttons traverse tab history; `activate()`
  guards against re-pushing the active hash, and the existing
  `hashchange` listener routes back/forward through `activate()` with
  `{updateHash:false}` to avoid double-pushing.
- Print stylesheet no longer hides `.map-panel` wholesale (the rule
  conflicted with `.panel{display:block !important}` at equal
  specificity and was getting overridden anyway). The Map panel now
  renders in print, but its iframe and live-meta strip are hidden and
  a new `.map-print-fallback` paragraph carries the dashboard URL on
  paper so the section is preserved.
- `preserveDrawingBuffer` in `map_3d.html` now toggles from a URL
  query flag (`?capture=1`) rather than always-on. The flag is
  required for `canvas.toBlob()` to return real pixels for the
  `CAPTURE-MEDIA.md` screenshot scripts, but it disables a WebGL
  back-buffer optimisation and costs interactive performance for
  ordinary browsing. `CAPTURE-MEDIA.md` updated to describe the new
  entry point.
- Footnote `<li>` targets carry `tabindex="-1"` so the click
  handler's `target.focus({preventScroll:true})` reliably moves
  keyboard / screen-reader focus and the `:target` highlight
  resolves accessibly.
- `activate()` cleanup: dropped a no-op
  `'instant' in window ? 'auto' : 'auto'` ternary in `window.scrollTo`
  and folded the trailing focus-loop into the main `TABS.forEach`
  pass where the active tab is already in scope.
- CSS shell comment updated from `sticky header + sticky tab strip`
  to `naturally-scrolling title block above a sticky tab strip` so
  it matches the implemented behaviour (the title block scrolls
  naturally; only the tab strip is `position: sticky`).

## [2026-04-28] — Pipeline perfection pass + cross-zone audit

### Output filenames — hyphens, not underscores
- Generated XLSX, dashboard HTML, and combined-zone files now use hyphens
  (`TIGER-Audit-Blue-Ash-Montgomery-Dashboard.html`) instead of underscores.
  Slack, Discord, and most Markdown renderers italicize text between
  underscores, so URLs to the previous filenames were getting mangled in
  shared messages. `_zone_name_to_filename` produces hyphenated `safe_name`,
  the prefix and suffix template strings are hyphenated, and the combined
  dashboard filename is `TIGER-Audit-All-Zones-Dashboard.html`.
- `index.html`, `README.md`, and `LetterToSORTA.docx` (downstream) all
  updated to match.

### Combined dashboard correctness — bbox-overlap deduplication
- Zone bboxes overlap (Northgate ↔ Forest Park ~32 km², Springdale ↔
  Forest Park ~14 km²), so the same OSM way appeared in multiple per-zone
  audits and the combined dashboard's totals double-counted. Fixed:
  - `merged_ways` is now deduped by OSM way ID before stats are computed.
  - `merged_gaps` is deduped by the unordered `(way1_id, way2_id)` pair.
  - Combined `total`, `residential`, `oneway`, and `by_class` are
    recomputed from the deduped lists rather than summed across zones.
  - `class_b_streets_total` uses connected-components analysis over
    `(name, way_id_set)` groups: an arterial that crosses a zone boundary
    collapses to one street, while two unrelated streets that happen to
    share a name in different zones (disjoint way IDs) are counted
    separately.
  - One additional cross-zone spatial pass on gaps (within `GAP_CLUSTER_M`
    on the same street) catches the boundary edge case where two zones
    pick different "tightest pair" representatives for the same physical
    junction.
- Console prints a one-line dedup summary every `--zone all` run so the
  overlap stays visible during regeneration.
- `all_zones_summary.csv` now ends with two trailing rows: the per-zone
  sum (with overlap duplicates, kept for transparency) and the deduped
  unique count.

### Per-zone gap detection — junction-level clustering
- `detect_gaps()` previously emitted one record per (way_i, way_j) pair,
  so a 3+ way junction with no shared node produced N gap records for
  what's really one missing node. Pairwise candidate gaps are now
  spatially clustered within 5 m on the same street and the
  smallest-distance pair is kept as the cluster's representative. Per-zone
  gap counts shifted accordingly: Blue Ash 308 → 248, Springdale 239 →
  182, Northgate 222 → 148, Forest Park 394 → 264. System-wide unique
  gaps: 949 → 710.

### Per-zone XLSX, Sheet 6 "MetroNow Zones" — cross-zone awareness
- The MetroNow Zones sheet in each per-zone workbook now marks the OTHER
  three zones as `AUDIT COMPLETE` if their `csv/all_ways.csv` exists on
  disk (with the actual unreviewed-segment count from that file), instead
  of always reporting them as `NOT STARTED`. A workbook for one zone now
  reflects the global audit state at the time of generation.

### Pipeline (initial remediation)
- Move `import math` to module-level imports.
- Add explicit lat/lon bounds validation to `_haversine_m`; raises on
  out-of-range input.
- Cache fallback sorts by file mtime (previously relied on filename
  string sort), prints age, warns past 14 days, and auto-prunes
  snapshots older than 14 days while keeping the 3 newest.
- Skip ways with missing/invalid geometry during classification; surface
  the count as `ways_missing_geom` in summary stats and on the console.
- Narrow the Overpass retry-loop `except` from `Exception` to
  `(requests.RequestException, ValueError, json.JSONDecodeError, RuntimeError)`.
- Make the "INDEX CASE" highlight zone-configurable via a new
  `index_case_street` field on each `ZONES` entry.

### CSV outputs
- `class_b_multi_segment.csv` includes a `_street_gap_count` column and
  is sorted by gap count first, then segment count, so volunteers can
  prioritize streets with the most probable node disconnects.

### Dashboard template (in `tiger_audit.py`)
- Hide `.leaflet-control-zoom` on screens ≤ 768 px (overlapped the
  hamburger).
- Visible 3 px focus outline on interactive map elements and search
  options.
- Backdrop element carries `role="presentation"`.
- Search combobox updates `aria-activedescendant` on ↑/↓ navigation;
  result options toggle `aria-selected`.
- Dark-mode toggle's `aria-label` and `title` reflect the next state
  ("Switch to light mode" / "Switch to dark mode").
- Print View temporarily strips the `dark` class so paper output stays
  light, then restores the prior theme on `afterprint`.
- JOSM batch URL chunk size raised from 7000 → 7500 chars.
- Escape-key handler clears the search `aria-activedescendant` and
  resets the active-result index.

### Documentation
- README sample-run table breaks out Class A (single-segment) vs Class AB
  (compound) instead of conflating them as "Class A: 57".
- README adds a Windows `file://` note covering corporate-proxy and
  JOSM-on-HTTPS edge cases.
- `TigerAuditPrompt.md` (in `Krass\`, separate repo) is synced to the
  current pipeline: Overpass query uses `out tags geom` with a 180 s
  timeout, Phase 2 documents node-disconnect detection, Phase 4
  specifies Leaflet (not Chart.js) with CARTO Voyager tiles, Phase 5
  console summary includes a gap count line, and an Expected Results
  section cites the post-clustering numbers.

### Dependencies
- `requirements.txt` bounds majors: `openpyxl>=3.1,<4`,
  `requests>=2.31,<3`.
