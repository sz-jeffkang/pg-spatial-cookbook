# Pattern Index & Quick Lookup

> 所有 Pattern 的索引和快速查找表。按症状关键词匹配你的问题，跳转到对应文档。

---

## 按症状查找

| 你看到的现象 | 可能原因 | 对应 Pattern |
|-------------|---------|-------------|
| `ST_DWithin` 超时 300s | 全表扫描，缺少空间索引 | [02 BBOX Pre-filter](./02-bbox-pre-filter.md) |
| `ST_DWithin` 超时 300s（另一类） | 使用 geometry 而非 geography | [02 BBOX Pre-filter](./02-bbox-pre-filter.md) |
| `ST_SRID` 返回 4326，但坐标值是 8 位数 | SRID 标签和实际坐标系不一致 | [01 SRID Camouflage](./01-srid-camouflage.md) |
| OD 线全为零长度 | 起终点用了同一组坐标 | [03 OD Lines](./03-od-lines.md) |
| 空间 JOIN 返回 0 行，明明有数据 | 两个表坐标系不一致 | [01 SRID Camouflage](./01-srid-camouflage.md) |
| `LIKE '%xxx%'` 在 psycopg2 报 `IndexError` | `%` 被 psycopg2 解释为参数占位符 | [05 psycopg2 Pitfalls](./05-psycopg2-pitfalls.md) |
| `ST_Distance` 返回值是字符串，无法比较 | psycopg2 的 `ST_Distance` 返回 `str` 而非 `float` | [05 psycopg2 Pitfalls](./05-psycopg2-pitfalls.md) |
| `pgsql2shp` 报 `"Table -g does not exist"` | geometry 列不在 SELECT 第一位 | [05 psycopg2 Pitfalls](./05-psycopg2-pitfalls.md) |
| 百万级 GPS 点需要分组聚类 | 需要高效聚类算法 | [04 DBSCAN Clustering](./04-dbscan-clustering.md) |
| DBSCAN 聚类全部分到同一簇 | `eps` 太大或 `minpoints` 太小 | [04 DBSCAN Clustering](./04-dbscan-clustering.md) |
| DBSCAN 所有点都是噪声（簇号 NULL） | `eps` 太小 | [04 DBSCAN Clustering](./04-dbscan-clustering.md) |
| 大规模空间 JOIN 超时 | 子查询套子查询导致嵌套循环 | 子查询→JOIN 重构 |
| 空间缓冲区查询极慢 | 不该用 `ST_Buffer` 做邻近查询 | 用 `ST_DWithin` + geography |

---

## 按性能改进查找

| 操作 | 优化前 | 优化后 | Pattern |
|------|--------|--------|---------|
| 8M 行空间距离查询 | 300s timeout | **15 秒** | [BBOX Pre-filter](./02-bbox-pre-filter.md) |
| 167M 行多表 JOIN | 1 小时 timeout | **3 秒** | 子查询→JOIN |
| 1,699 条空间查询 | **0 条结果** | 1,699 条 | [SRID Camouflage](./01-srid-camouflage.md) |
| 189 万点聚类 | 无方案 | **5,995 簇 / 2 分钟** | [DBSCAN Clustering](./04-dbscan-clustering.md) |

---

## 所有 Pattern 一览

| # | Pattern | 领域 | 难度 |
|---|---------|------|------|
| 01 | [SRID Camouflage](./01-srid-camouflage.md) | 坐标系 | ⭐ |
| 02 | [BBOX Pre-filter](./02-bbox-pre-filter.md) | 查询性能 | ⭐⭐⭐ |
| 03 | [OD Line Generation](./03-od-lines.md) | 几何构造 | ⭐ |
| 04 | [DBSCAN Clustering](./04-dbscan-clustering.md) | 空间聚类 | ⭐⭐ |
| 05 | [psycopg2 Pitfalls](./05-psycopg2-pitfalls.md) | Python 集成 | ⭐ |
| — | 子查询→JOIN 重构 | 查询性能 | ⭐⭐ |
| — | Corridor-Band 过滤 | 空间过滤 | ⭐⭐⭐ |
| — | Geography vs Geometry | 索引策略 | ⭐⭐ |
| — | Voronoi 插值 | 空间分析 | ⭐⭐ |
| — | 两阶段过滤 | 查询性能 | ⭐⭐ |
| — | 批量 CSV→PostGIS | 数据导入 | ⭐ |
| — | SHP 导出 | 数据导出 | ⭐ |
| — | 脏数据清洗 | 数据质量 | ⭐ |
| — | 区间值解析 | 数据清洗 | ⭐ |
| — | ST_Extent 预检 | 调试 | ⭐ |
| — | 分片并行 | 大规模处理 | ⭐⭐⭐ |

> **完整速查表**（含所有 20+ Pattern 的触发信号和一句话解决方案）：见 [`../cheatsheet.md`](../cheatsheet.md)

---

## 阅读建议

- **新手**：从 SRID Camouflage（01）和 OD Lines（03）开始 → 理解坐标系和几何构造基础
- **性能优化**：直接看 BBOX Pre-filter（02）和子查询→JOIN 重构
- **Python 用户**：先读 psycopg2 Pitfalls（05），省 3 小时 debug
- **大规模数据**：DBSCAN Clustering（04）+ 分片并行

---

## 通用调试口诀

```sql
-- 1. 检查两个表的 extent 是否重叠
SELECT 'table_a', ST_Extent(geom) FROM table_a
UNION ALL
SELECT 'table_b', ST_Extent(geom) FROM table_b;

-- 2. 检查两个表的 SRID 是否一致
SELECT DISTINCT ST_SRID(geom) FROM table_a
UNION
SELECT DISTINCT ST_SRID(geom) FROM table_b;

-- 3. 采样看实际的坐标范围
SELECT ST_X(geom), ST_Y(geom) FROM table_a LIMIT 5;
```
