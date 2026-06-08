# PostgreSQL Spatial Cookbook

<p align="center">
  <img src="https://img.shields.io/badge/Patterns-20+-blue" alt="20+ Patterns">
  <img src="https://img.shields.io/badge/PostGIS-3.x-green" alt="PostGIS 3.x">
  <img src="https://img.shields.io/badge/License-MIT-yellow" alt="MIT">
</p>

> **Production-tested PostgreSQL/PostGIS patterns from real-world GIS data pipelines. Every pattern here has failed at least once before it worked.**

---

## Why This Exists

PostGIS documentation tells you what functions exist. It doesn't tell you:
- Why `ST_DWithin` on 8M rows times out (and how to fix it in 15 seconds)
- Why `%` in psycopg2 LIKE queries crashes with `IndexError` (double every `%`)
- Why `ST_SRID` says 4326 but your coordinates are clearly projected (the SRID camouflage trap)
- Why your OD lines are all zero-length (you used the same coordinate twice)

This cookbook is the compiled scar tissue from processing billions of spatial records.

---

## Pattern Index

| Pattern | Problem | Solution |
|---------|---------|----------|
| **SRID Camouflage** | ST_SRID=4326 but coords are CGCS2000 projected | `ST_SetSRID→ST_Transform` |
| **psycopg2 % Escape** | LIKE crashes on parameterized queries | Double every `%` to `%%` |
| **OD Line Generation** | All lines zero-length | Use start + end coords, not end×2 |
| **Corridor-Band** | 46 OD pairs from 167M rows | `ST_DWithin` on route corridor |
| **Subquery vs JOIN** | 375 sequential scans → 1hr timeout | JOIN + GROUP BY |
| **ST_DWithin vs ST_Buffer** | 300s timeout on proximity | Geography + GIST index |
| **BBOX Pre-filter** | 8M rows → ST_DWithin timeout | Numeric range → temp table → GIST → precise |
| **DBSCAN Clustering** | Millions of GPS points need grouping | `ST_ClusterDBSCAN(geom, eps, minpoints)` |
| **Voronoi Interpolation** | Sparse station data → regional coverage | `ST_VoronoiPolygons` + attribute join |
| **Batch CSV→PostGIS** | 16+ CSV files with WKT geometry | `\\copy` + `ST_GeomFromText` |
| **Interval Cleaning** | "0.03-0.05" → numeric crash | `SPLIT_PART` + midpoint calc |
| **Dirty Data** | `\x00` null bytes, embedded tabs in CSV | `execute_values` + strip `\x00` |
| **SHP Export** | pgsql2shp "Table -g does not exist" | Geometry column FIRST, omit `-g` flag |
| **Two-Step Filter** | 3-table JOIN on aggregates → timeout | Filter table A → filter table B on surviving IDs |

---

## Real-World Performance Gains

| Pattern | Before | After |
|---------|--------|-------|
| BBOX Pre-filter (8M rows) | 300s timeout | **15 seconds** |
| Subquery→JOIN (167M rows) | 1hr timeout | **3 seconds** |
| SRID Fix (1,699 M-land parcels) | **0 results** | 1,699 results |
| DBSCAN (1.89M points) | N/A | **5,995 clusters in 2 min** |

---

## Quick Start

```sql
-- The single most important check before ANY spatial join:
SELECT 'table_a', ST_Extent(geom) FROM table_a
UNION ALL
SELECT 'table_b', ST_Extent(geom) FROM table_b;
-- If extents don't overlap, your join returns 0 rows. Period.
```

```bash
# SHP export done right: geometry FIRST, no -g flag
pgsql2shp -f out.shp -h localhost -u user db \
  "SELECT geom_col, attr1, attr2 FROM tbl ORDER BY id"
```

```python
# psycopg2: double every % in LIKE patterns
cur.execute("SELECT * FROM t WHERE name ILIKE '%%Beijing%%'")
```

---

## License

MIT. Share the scar tissue.

<p align="center">
  <sub>20+ patterns · billions of rows · zero secrets</sub>
</p>
