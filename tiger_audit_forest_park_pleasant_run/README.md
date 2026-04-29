# TIGER/Line Audit - Forest Park / Pleasant Run

Read-only audit of OpenStreetMap ways with `tiger:reviewed=no` inside
the Forest Park / Pleasant Run MetroNow zone (bbox (39.26, -84.56, 39.34, -84.46)).

Audit timestamp (UTC): 2026-04-29T02:09:57+00:00

## Re-run

```
python3 tiger_audit.py --zone forest_park_pleasant_run
```

## Files

- `data/forest_park_pleasant_run_raw_<UTC>.json` - raw Overpass response (preserved)
- `reports/TIGER-Audit-*.xlsx` - styled multi-sheet workbook
- `reports/TIGER-Audit-*-Dashboard.html` - interactive dashboard (open in browser)
- `csv/all_ways.csv` - master inventory
- `csv/class_a_false_oneway.csv` - residential + oneway=yes
- `csv/class_b_multi_segment.csv` - streets with 2+ unreviewed segments
- `csv/class_ab_compound.csv` - intersection of A and B
