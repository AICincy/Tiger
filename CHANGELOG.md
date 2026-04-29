# Changelog

All notable changes to the TIGER/Line audit pipeline.

## [Unreleased]

### Pipeline
- Move `import math` to module-level imports (was previously placed mid-file).
- Add explicit lat/lon bounds validation to `_haversine_m`; raises on out-of-range input.
- Cache fallback now sorts by file mtime (previously relied on filename string sort).
- Print cache age and warn when fallback cache is older than 14 days.
- Auto-prune raw Overpass JSON snapshots older than 14 days, keeping the 3 newest.
- Skip ways with missing or invalid geometry during classification; surface count in summary stats as `ways_missing_geom`.
- Narrow the Overpass retry-loop `except` from `Exception` to a fixed allowlist (`requests.RequestException`, `ValueError`, `json.JSONDecodeError`, `RuntimeError`).
- Make the `O'Leary Avenue` "INDEX CASE" highlight zone-configurable via a new `index_case_street` field on each `ZONES` entry.

### CSV outputs
- `class_b_multi_segment.csv` now includes a `_street_gap_count` column and is sorted by gap count first, then segment count, so volunteers can prioritize the streets with the most probable node disconnects.

### Dashboard template (in `tiger_audit.py`)
- Hide the Leaflet zoom control on screens ≤ 768 px (overlapped the hamburger).
- Add a visible 3 px focus outline on interactive map elements and search options.
- Backdrop element now carries `role="presentation"`.
- Search combobox gets `aria-activedescendant` updates on ↑/↓ navigation; result options toggle `aria-selected`.
- Dark-mode toggle's `aria-label` and `title` update to reflect the next state ("Switch to light mode" / "Switch to dark mode").
- Print View now temporarily strips the `dark` class so paper output stays light, then restores the prior theme on `afterprint`.
- JOSM batch URL chunk size raised from 7000 → 7500 chars (still well under the typical 8000-char URL limit).
- Escape-key handler clears the search `aria-activedescendant` and resets the active-result index.

### Documentation
- README sample-run table now breaks out Class A (single-segment) vs Class AB (compound) instead of conflating them as "Class A: 57".
- README adds a Windows `file://` note covering corporate-proxy and JOSM-on-HTTPS edge cases.
- `TigerAuditPrompt.md` updated for the current pipeline: Overpass query uses `out geom tags` with a 180s timeout, Phase 2 documents node-disconnect detection, Phase 4 specifies Leaflet (not Chart.js) with CARTO Voyager tiles, Phase 5 console summary includes a gap count line, and an Expected Results section cites ~308 gaps / ~182 streets / ~55 Class AB.

### Dependencies
- `requirements.txt` now bounds versions: `openpyxl>=3.1,<4`, `requests>=2.31,<3`.
