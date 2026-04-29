# TIGER/Line Audit - Northgate / Mt. Healthy

Read-only audit of OpenStreetMap ways with `tiger:reviewed=no` inside
the Northgate / Mt. Healthy MetroNow zone (bbox (39.22, -84.58, 39.3, -84.48)).

Audit timestamp (UTC): 2026-04-29T02:09:54+00:00

## Re-run

```
python3 tiger_audit.py --zone northgate_mt_healthy
```

## Files

- `data/northgate_mt_healthy_raw_<UTC>.json` - raw Overpass response (preserved)
- `reports/TIGER-Audit-*.xlsx` - styled multi-sheet workbook
- `reports/TIGER-Audit-*-Dashboard.html` - interactive dashboard (open in browser)
- `csv/all_ways.csv` - master inventory
- `csv/class_a_false_oneway.csv` - residential + oneway=yes
- `csv/class_b_multi_segment.csv` - streets with 2+ unreviewed segments
- `csv/class_ab_compound.csv` - intersection of A and B
