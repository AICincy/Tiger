# Changelog

All notable changes to the TIGER/Line audit pipeline.

## [Unreleased]

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
  current pipeline: Overpass query uses `out geom tags` with a 180 s
  timeout, Phase 2 documents node-disconnect detection, Phase 4
  specifies Leaflet (not Chart.js) with CARTO Voyager tiles, Phase 5
  console summary includes a gap count line, and an Expected Results
  section cites the post-clustering numbers.

### Dependencies
- `requirements.txt` bounds majors: `openpyxl>=3.1,<4`,
  `requests>=2.31,<3`.
