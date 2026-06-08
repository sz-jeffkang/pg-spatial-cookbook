# PostGIS 实战速查表

> **20+ 生产验证 Pattern 的一页纸速查。看到触发信号 → 跳转对应解决方案。**

---

## 坐标系与几何

| # | 触发信号 | Pattern | 一句话解决 |
|---|---------|---------|-----------|
| 1 | `ST_SRID`=4326 但坐标是 39,451,234 | [SRID 伪装](./patterns/01-srid-camouflage.md) | `ST_SetSRID(geom, real_srid)` → `ST_Transform` |
| 2 | `ST_Intersects`/`ST_DWithin` 返回 0 行 | [Extent 预检](./patterns/01-srid-camouflage.md) | `SELECT ST_Extent(geom) FROM` 两个表对比 |
| 3 | 两个表坐标系不同却要 JOIN | [坐标系对齐](./patterns/01-srid-camouflage.md) | 统一 `ST_Transform` 到一个 SRID，在其中一张表上建函数索引 |
| 4 | `ST_MakePoint(lat, lon)` 点飘到南极 | 经纬度顺序 | `ST_MakePoint(lon, lat)` — **经度在前，纬度在后** |

---

## 查询性能

| # | 触发信号 | Pattern | 一句话解决 |
|---|---------|---------|-----------|
| 5 | `ST_DWithin` 在 8M 行上超时 300s | [BBOX 预过滤](./patterns/02-bbox-pre-filter.md) | 数值范围过滤 → 临时表 → GIST 索引 → `ST_DWithin` |
| 6 | 多表 JOIN 子查询 → 1 小时 timeout | 子查询→JOIN 重构 | 把 `WHERE IN (SELECT ...)` 改成 `JOIN` + `GROUP BY` |
| 7 | `ST_DWithin(geom, geom, 5000)` 极慢 | Geography vs Geometry | 改用 `ST_DWithin(geom::geography, geom::geography, 5000)` + GIST(geography) |
| 8 | `EXPLAIN` 显示 Seq Scan 而非 Index Scan | 索引未被使用 | 检查 `ST_Transform` 是否包裹了索引列；在查询中用和索引相同的 SRID |
| 9 | 375 次 sequential scan 拖死查询 | 子查询膨胀 | 改写为 CTE + JOIN，确保只扫描一次 |
| 10 | 167M 行两阶段过滤超时 | 两阶段过滤 | Phase 1: 过滤表 A → Phase 2: 只对存活 ID 过滤表 B |
| 11 | `ST_Buffer` + `ST_Intersects` 极慢 | Buffer 反模式 | 改用 `ST_DWithin` — 它是为邻近查询设计的，不用先生成 buffer polygon |

---

## OD 与几何构造

| # | 触发信号 | Pattern | 一句话解决 |
|---|---------|---------|-----------|
| 12 | OD 线 12.5% 为零长度 | [OD 线生成](./patterns/03-od-lines.md) | 检查 `origin_lon != dest_lon OR origin_lat != dest_lat` |
| 13 | OD 线全指向同一个方向 | 坐标复制错误 | 验证 `ROUND(origin::numeric,6) = ROUND(dest::numeric,6)` |
| 14 | 走廊带 OD 匹配（46 对从 167M 行） | Corridor-Band | 沿路线生成 corridor polygon → `ST_DWithin` 过滤 |

---

## 聚类与分析

| # | 触发信号 | Pattern | 一句话解决 |
|---|---------|---------|-----------|
| 15 | 百万 GPS 点需要分组 | [DBSCAN 聚类](./patterns/04-dbscan-clustering.md) | `ST_ClusterDBSCAN(geom, eps, minpoints) OVER()` — 1% 采样调参，全量 2 分钟 |
| 16 | DBSCAN 所有点在同一个簇 | eps 太大 | 用最近邻距离的 p75-p90 分位数做 eps |
| 17 | 稀疏站点数据需要插值到面 | Voronoi 插值 | `ST_VoronoiPolygons(ST_Collect(geom))` → JOIN 属性 |
| 18 | 区间值 "0.03-0.05" 无法计算 | 区间解析 | `SPLIT_PART(val, '-', 1)` + `SPLIT_PART(val, '-', 2)` → 取中值 |

---

## Python 集成

| # | 触发信号 | Pattern | 一句话解决 |
|---|---------|---------|-----------|
| 19 | `IndexError: tuple index out of range` | [psycopg2 % 转义](./patterns/05-psycopg2-pitfalls.md) | LIKE 中的 `%` 双写为 `%%` |
| 20 | `ST_Distance` 返回 `str` 无法比较 | [psycopg2 类型陷阱](./patterns/05-psycopg2-pitfalls.md) | `float(cur.fetchone()[0])` |
| 21 | `Table "name" does not exist` (pgsql2shp) | [SHP 导出](./patterns/05-psycopg2-pitfalls.md) | geometry 列放 SELECT **第一位**，不要用 `-g` 参数 |
| 22 | `\\x00` null bytes 导致 INSERT 失败 | 脏数据清洗 | `execute_values()` + Python 侧 `strip('\\x00')` |
| 23 | CSV 中嵌入了 tab 字符导致列错位 | 脏数据清洗 | `csv.reader(dialect='excel-tab')` 或预处理替换 `\t` |

---

## 数据导入/导出

| # | 触发信号 | Pattern | 一句话解决 |
|---|---------|---------|-----------|
| 24 | 16+ CSV 文件批量导入 | 批量 CSV→PostGIS | `\\copy` + `ST_GeomFromText(wkt_col, srid)` — 单文件 < 30s |
| 25 | `pgsql2shp` 报 "does not exist" | SHP 导出 | geometry 列必须是 SELECT 第一列；省略 `-g` |
| 26 | CSV WKT 列无法直接转 geometry | WKT 转换 | `ST_GeomFromText(wkt_column, 4326)` |

---

## 调试与诊断

| # | 触发信号 | Pattern | 一句话解决 |
|---|---------|---------|-----------|
| 27 | 空间 JOIN 返回 0 行，两表都有数据 | 通用诊断三步法 | `ST_Extent` → `ST_SRID` → `ST_X/ST_Y` 采样 |
| 28 | 查询计划用 Hash Join 而非空间索引 | 索引策略 | `SET enable_seqscan = off;` 测试；检查 SRID 变换是否阻止索引使用 |

---

## 性能速查

| 操作 | 错误做法 | 正确做法 |
|------|---------|---------|
| 邻近查询 | `ST_Buffer(geom, 5000)` + `ST_Intersects` | `ST_DWithin(geom::geography, ref::geography, 5000)` |
| 距离计算 | `ST_Distance(geom, geom)` (geometry) | `ST_Distance(geom::geography, geom::geography)` (球面距离) |
| 大表 JOIN | 子查询 `WHERE IN (SELECT ...)` | `JOIN` + `GROUP BY` |
| 百万行过滤 | `ST_DWithin` 直接扫全表 | BBOX 数值预过滤 → 临时表 |
| 聚类调参 | 全量数据试参 | 1% `TABLESAMPLE` 采样 |

---

## 通用调试模板

```sql
-- ═══════════════════════════════════
-- 空间查询不返回结果的调试三步
-- ═══════════════════════════════════

-- 1️⃣ 两个表的 extent 重叠吗？
SELECT 'table_a', ST_Extent(geom) FROM table_a
UNION ALL
SELECT 'table_b', ST_Extent(geom) FROM table_b;

-- 2️⃣ SRID 一致吗？
SELECT DISTINCT 'table_a', ST_SRID(geom) FROM table_a
UNION ALL
SELECT DISTINCT 'table_b', ST_SRID(geom) FROM table_b;

-- 3️⃣ 坐标值在合法范围内吗？（SRID 伪装检测）
SELECT
    MIN(ST_X(ST_Centroid(geom))) AS min_x,
    MAX(ST_X(ST_Centroid(geom))) AS max_x,
    MIN(ST_Y(ST_Centroid(geom))) AS min_y,
    MAX(ST_Y(ST_Centroid(geom))) AS max_y
FROM table_a;
```

```python
# ═══════════════════════════════════
# psycopg2 安全模板
# ═══════════════════════════════════
from psycopg2.extras import RealDictCursor

conn = psycopg2.connect("dbname=mydb")
cur = conn.cursor(cursor_factory=RealDictCursor)

# LIKE 查询：双写 %%
cur.execute("SELECT * FROM cities WHERE name ILIKE '%%' || %s || '%%'", ("Beijing",))

# 距离查询：显式 float()
cur.execute("SELECT ST_Distance(a.geom::geography, b.geom::geography) FROM ...")
dist = float(cur.fetchone()[0])
```

```bash
# ═══════════════════════════════════
# SHP 导出安全模板
# ═══════════════════════════════════
# geometry 列必须在 SELECT 第一位！不需要 -g 参数
pgsql2shp -f output.shp -h localhost -u user db \
  "SELECT geom_col, attr1, attr2 FROM my_table"
```

---

## 记忆口诀

| 口诀 | 含义 |
|------|------|
| **SRID 是标签，不是坐标** | 不要信任 `ST_SRID()`，自己检查坐标范围 |
| **% 要双写，距离要 float** | psycopg2 的两条铁律 |
| **geometry 第一列，不要 -g** | pgsql2shp 的唯一规则 |
| **先缩圈，再精确** | BBOX 预过滤的核心思想 |
| **1% 采样调参，全量一次跑** | DBSCAN 参数选择方法论 |
| **JOIN 代替子查询** | 167M 行的查询从 1 小时到 3 秒 |
| **ST_DWithin 不用 ST_Buffer** | 邻近查询的正确姿势 |
