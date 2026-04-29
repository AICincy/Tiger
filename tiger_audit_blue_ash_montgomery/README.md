# TIGER/Line Audit - Blue Ash / Montgomery

Read-only audit of OpenStreetMap ways with `tiger:reviewed=no` inside
the Blue Ash / Montgomery MetroNow zone (bbox (39.16, -84.44, 39.24, -84.33)).

Audit timestamp (UTC): 2026-04-29T13:07:54+00:00

## Re-run

```
python3 tiger_audit.py --zone blue_ash_montgomery
```

## Files

- `data/blue_ash_montgomery_raw_<UTC>.json` - raw Overpass response (preserved)
- `reports/TIGER-Audit-*.xlsx` - styled multi-sheet workbook
- `reports/TIGER-Audit-*-Dashboard.html` - interactive dashboard (open in browser)
- `csv/all_ways.csv` - master inventory
- `csv/class_a_false_oneway.csv` - residential + oneway=yes
- `csv/class_b_multi_segment.csv` - streets with 2+ unreviewed segments
- `csv/class_ab_compound.csv` - intersection of A and B
