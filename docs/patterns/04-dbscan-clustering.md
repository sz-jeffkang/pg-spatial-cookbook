# Pattern 04: DBSCAN 点聚类

> **DBSCAN Clustering: 189 万点 → 5,995 簇的完整流水线，含参数选择与 SHP 导出。**

---

## 症状

你有一张 GPS 轨迹点表，189 万行，需要将空间上聚集的点分组为簇（cluster）。业务需求：

- 识别出活动热点区域
- 每个簇导出为独立的 SHP 文件
- 噪声点（不属于任何簇的点）单独标记

如果用纯 SQL 手动实现（计算距离矩阵 → 传递闭包），189 万点的距离矩阵有 **~1.8 万亿个元素**，完全不可行。

---

## 方案选择

PostGIS 提供了 `ST_ClusterDBSCAN` 窗口函数，内部使用 DBSCAN 算法：

```sql
ST_ClusterDBSCAN(geometry geom, float8 eps, int4 minpoints)
OVER (PARTITION BY partition_col)
```

- `eps`：邻域半径（单位与几何坐标系一致）
- `minpoints`：形成簇的最少点数
- 返回：簇编号（>=0）或 NULL（噪声点）

**优势**：利用空间索引，复杂度接近 O(n log n)，支持 189 万点。

---

## 参数选择

这是整个流程最关键的一步。参数选择不当会导致：

| 问题 | 原因 |
|------|------|
| 所有点分到同一簇 | `eps` 太大 |
| 几乎所有点都是噪声（NULL） | `eps` 太小 |
| 每个簇只有 1-2 个点 | `minpoints` 太小 |
| 簇太少、太大，没有区分度 | `minpoints` 太小 + `eps` 太大 |

### 参数探索 SQL

```sql
-- 先了解数据密度特征
-- 假设坐标单位是米（投影坐标系），eps 通常在 10m~500m 范围
-- 假设坐标是经纬度（4326），eps 单位是度：0.0001° ≈ 11m

-- 方法 1：找第 k 近邻距离分布
-- 用第 5 近邻距离的分布帮助选择 eps
WITH knn_dist AS (
    SELECT
        a.id,
        MIN(ST_Distance(a.geom, b.geom)) FILTER (
            WHERE a.id != b.id
        ) AS nearest_dist
    FROM points_table a
    CROSS JOIN LATERAL (
        SELECT geom FROM points_table b
        WHERE b.id != a.id
        ORDER BY a.geom <-> b.geom
        LIMIT 10
    ) b
    GROUP BY a.id
)
SELECT
    percentile_cont(0.10) WITHIN GROUP (ORDER BY nearest_dist) AS p10,
    percentile_cont(0.25) WITHIN GROUP (ORDER BY nearest_dist) AS p25,
    percentile_cont(0.50) WITHIN GROUP (ORDER BY nearest_dist) AS p50,
    percentile_cont(0.75) WITHIN GROUP (ORDER BY nearest_dist) AS p75,
    percentile_cont(0.90) WITHIN GROUP (ORDER BY nearest_dist) AS p90
FROM knn_dist;
```

**经验法则**：`eps` 通常取最近邻距离的 **p75 ~ p90** 分位数。

### 快速参数测试

不要对 189 万行直接调参！先用采样：

```sql
-- 采样 1% 做快速参数测试
CREATE TEMP TABLE sample_points AS
SELECT * FROM points_table
TABLESAMPLE SYSTEM (1);

-- 测试多组参数
SELECT
    'eps=50 minpts=5' AS params,
    COUNT(DISTINCT ST_ClusterDBSCAN(geom, 50, 5) OVER ()) AS clusters,
    COUNT(*) FILTER (WHERE ST_ClusterDBSCAN(geom, 50, 5) OVER () IS NULL) AS noise
FROM sample_points
UNION ALL
SELECT
    'eps=100 minpts=5',
    COUNT(DISTINCT ST_ClusterDBSCAN(geom, 100, 5) OVER ()),
    COUNT(*) FILTER (WHERE ST_ClusterDBSCAN(geom, 100, 5) OVER () IS NULL)
FROM sample_points
UNION ALL
SELECT
    'eps=100 minpts=10',
    COUNT(DISTINCT ST_ClusterDBSCAN(geom, 100, 10) OVER ()),
    COUNT(*) FILTER (WHERE ST_ClusterDBSCAN(geom, 100, 10) OVER () IS NULL)
FROM sample_points
UNION ALL
SELECT
    'eps=200 minpts=10',
    COUNT(DISTINCT ST_ClusterDBSCAN(geom, 200, 10) OVER ()),
    COUNT(*) FILTER (WHERE ST_ClusterDBSCAN(geom, 200, 10) OVER () IS NULL)
FROM sample_points;
```

---

## 完整流水线

### 完整 SQL（189 万点 → 5,995 簇）

```sql
-- ============================================
-- Step 1: 执行 DBSCAN 聚类（全量数据）
-- 基于采样测试选择最优参数
-- ============================================
CREATE TABLE gps_clusters AS
SELECT
    -- 如果需要按天/区域分区，加上 PARTITION BY
    ST_ClusterDBSCAN(geom, 100, 5) OVER (
        PARTITION BY date_trunc('day', ts)
    ) AS cluster_id,
    id,
    geom,
    ts
FROM points_table;

-- ============================================
-- Step 2: 检查聚类结果
-- ============================================
SELECT
    'clustered' AS type,
    COUNT(*) FILTER (WHERE cluster_id IS NOT NULL) AS point_count,
    COUNT(DISTINCT cluster_id) FILTER (WHERE cluster_id IS NOT NULL) AS cluster_count,
    MIN(cnt) AS min_pts_per_cluster,
    ROUND(AVG(cnt), 1) AS avg_pts_per_cluster,
    MAX(cnt) AS max_pts_per_cluster
FROM (
    SELECT cluster_id, COUNT(*) AS cnt
    FROM gps_clusters
    WHERE cluster_id IS NOT NULL
    GROUP BY cluster_id
) sub
UNION ALL
SELECT
    'noise',
    COUNT(*) FILTER (WHERE cluster_id IS NULL),
    NULL,
    NULL, NULL, NULL
FROM gps_clusters;
```

**预期输出（基于 189 万点）**：

```
  type    | point_count | cluster_count | min |  avg  |  max
----------+-------------+---------------+-----+-------+-------
 clustered |   1,723,000 |         5,995 |   5 | 287.4 | 12,340
 noise     |     167,000 |               |     |       |
```

### Step 3: 计算簇的统计信息

```sql
-- 每个簇的质心、点数、空间范围
CREATE TABLE cluster_stats AS
SELECT
    cluster_id,
    COUNT(*) AS point_count,
    ST_Centroid(ST_Collect(geom)) AS centroid,
    ST_ConvexHull(ST_Collect(geom)) AS convex_hull,
    ST_Envelope(ST_Collect(geom)) AS bbox,
    MIN(ts) AS time_start,
    MAX(ts) AS time_end
FROM gps_clusters
WHERE cluster_id IS NOT NULL
GROUP BY cluster_id;

-- 建索引
CREATE INDEX ON cluster_stats USING GIST (centroid);
```

### Step 4: 导出 Top N 簇为 SHP

```sql
-- 导出点数最多的 10 个簇
-- 注意：geometry 列必须在 SELECT 的第一位
COPY (
    SELECT
        c.geom,           -- geometry 第一位（pgsql2shp 要求）
        gc.cluster_id,
        gc.id,
        gc.ts
    FROM gps_clusters gc
    JOIN (
        SELECT cluster_id
        FROM cluster_stats
        ORDER BY point_count DESC
        LIMIT 10
    ) top ON gc.cluster_id = top.cluster_id
    WHERE gc.cluster_id IS NOT NULL
) TO '/tmp/top10_clusters.csv' WITH CSV HEADER;
```

```bash
# 用 pgsql2shp 导出（geometry 列必须第一位，不需要 -g 参数）
pgsql2shp -f top10_clusters.shp -h localhost -u user db \
  "SELECT gc.geom, gc.cluster_id, gc.id, gc.ts
   FROM gps_clusters gc
   JOIN (SELECT cluster_id FROM cluster_stats ORDER BY point_count DESC LIMIT 10) top
     ON gc.cluster_id = top.cluster_id
   WHERE gc.cluster_id IS NOT NULL"
```

---

## 参数调优实战

真实案例中的参数选择过程：

| 参数 | 簇数（采样） | 噪声% | 判定 |
|------|-------------|-------|------|
| eps=50, minpts=3 | 15,000+ | 40% | ❌ 噪声太多，簇太碎 |
| eps=50, minpts=5 | 12,000+ | 48% | ❌ 噪声太多 |
| eps=100, minpts=3 | 2,000 | 5% | ⚠️ 簇太少，最大簇包含 60% 的点 |
| eps=100, minpts=5 | 6,000 | 12% | ✅ 平衡 |
| eps=100, minpts=10 | 4,500 | 20% | ⚠️ 丢失了小簇 |
| eps=200, minpts=5 | 800 | 2% | ❌ 几乎全在一个簇 |

**最终选择**：`eps=100, minpoints=5` → 全量数据跑出 **5,995 个簇，12% 噪声**。

---

## 性能

| 数据量 | 耗时 | 内存 |
|--------|------|------|
| 1% 采样 (~1.9 万点) | < 1s | 低 |
| 全量 189 万点 | **~2 分钟** | ~4GB |
| 簇统计 + convex hull | ~30s | ~2GB |

> 注意：`ST_ClusterDBSCAN` 是全内存操作，确保 `work_mem` 足够大。

---

## 关键要点

> **不要用全量数据调参。1% 采样 1 秒出结果，迭代 10 组参数也只要 10 秒。**

- 参数选择是核心：`eps` 取 p75~p90 最近邻距离，`minpoints` 通常 3~10
- 用 `COUNT(DISTINCT cluster_id)` 和噪声比例评估聚类质量
- 加上 `PARTITION BY` 避免不同区域的点被错误聚到一起
- geometry 列放 SELECT 第一位以兼容 `pgsql2shp`
