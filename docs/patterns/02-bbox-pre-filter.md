# Pattern 02: BBOX 预过滤

> **BBOX Pre-filter: 8M rows → 300s timeout → 15 seconds via 5-step pipeline.**

---

## 症状

你有一张 **800 万行** 的点表，需要找出所有在某个多边形缓冲区内的点：

```sql
SELECT p.id, p.geom
FROM points_table p
JOIN region_table r ON ST_DWithin(p.geom, r.geom, 1000)
WHERE r.region_name = 'target_region';
```

查询跑了几分钟，最终：
```
ERROR:  canceling statement due to statement timeout
```

即使加了 GIST 空间索引也一样超时。

```sql
-- 你确认过索引存在
SELECT indexname FROM pg_indexes WHERE tablename = 'points_table';
-- points_table_geom_idx | CREATE INDEX ... USING GIST (geom)
```

---

## 根因分析

`ST_DWithin` 配合 GIST 索引通常很快，但前提是 PostGIS 能利用索引做边界框（bounding box）过滤。问题在于：

1. **800 万行的 bounding box 太大**，索引选择性差
2. 即使 GIST 索引过滤掉了 90% 的行，仍然有 80 万行需要精确计算 `ST_DWithin`
3. `ST_DWithin` 的精确计算（球面距离/笛卡尔距离）比 bounding box 比较慢几个数量级
4. 如果有 JOIN，优化器可能选择全表扫描

**核心矛盾**：GIST 索引做 bounding box 过滤时，如果数据分布太广，索引几乎无法削减候选集。

---

## 解决方案：5 步流水线

思路：用**应用层逻辑**先做一个粗糙但极快的前置过滤，把候选集从 800 万缩减到几千，然后再用 GIST + ST_DWithin 精确过滤。

```
Step 1: 计算目标多边形 + buffer 的 bounding box
Step 2: 用数值范围过滤（NOT 空间索引），砍掉 99%+ 的数据
Step 3: 候选集写入临时表
Step 4: 在临时表上建 GIST 索引（小表建索引极快）
Step 5: 在临时表上跑 ST_DWithin（精确过滤）
```

### 完整 SQL

```sql
-- ============================================
-- Step 1 & 2: BBOX 数值预过滤
-- 获取目标多边形的 buffer envelope 的坐标范围
-- ============================================
WITH target_envelope AS (
    SELECT
        ST_XMin(bbox) AS xmin,
        ST_YMin(bbox) AS ymin,
        ST_XMax(bbox) AS xmax,
        ST_YMax(bbox) AS ymax
    FROM (
        SELECT ST_Envelope(
            ST_Buffer(geom::geography, 1000)::geometry
        ) AS bbox
        FROM region_table
        WHERE region_name = 'target_region'
    ) sub
)
-- ============================================
-- Step 3: 候选集写入临时表
-- 仅用 X/Y 坐标的数值范围过滤 —— 这是纯 B-tree 操作，极快
-- ============================================
SELECT p.id, p.geom
INTO TEMP TABLE candidate_points
FROM points_table p, target_envelope e
WHERE ST_X(p.geom) BETWEEN e.xmin AND e.xmax
  AND ST_Y(p.geom) BETWEEN e.ymin AND e.ymax;

-- ============================================
-- Step 4: 在候选临时表上建 GIST 索引
-- 候选集通常只有几千到几万行，建索引 < 1 秒
-- ============================================
CREATE INDEX ON candidate_points USING GIST (geom);
ANALYZE candidate_points;

-- ============================================
-- Step 5: 精确过滤
-- 在小候选集上跑 ST_DWithin，毫秒级完成
-- ============================================
SELECT cp.id, cp.geom
FROM candidate_points cp
JOIN region_table r ON ST_DWithin(cp.geom, r.geom, 1000)
WHERE r.region_name = 'target_region';

-- 清理
DROP TABLE IF EXISTS candidate_points;
```

---

## 性能对比

| 方法 | 数据量 | 耗时 | 说明 |
|------|--------|------|------|
| 直接 `ST_DWithin` + GIST | 8M 行 | **300s (timeout)** | GIST 索引无法有效削减候选集 |
| 直接 `ST_DWithin` + geography | 8M 行 | **~180s** | 仍太慢，geography 计算开销更大 |
| **BBOX 预过滤 + 临时表 + GIST + ST_DWithin** | 8M 行 | **~15 秒** | ✅ 生产方案 |

**分步耗时分析（典型情况）**：

| 步骤 | 耗时 |
|------|------|
| Step 1-2: 计算 envelope | < 0.1s |
| Step 3: 数值范围过滤 + 写入临时表 | ~10s |
| Step 4: 建 GIST 索引 | ~1s |
| Step 5: 精确 ST_DWithin | ~3s |
| **合计** | **~15s** |

---

## 为什么这比纯 GIST 索引更快？

1. **数值范围过滤是 B-tree/O(n) 扫描**，不需要索引遍历，对宽表反而是最快路径
2. **临时表把候选集从 800 万砍到几千**，后续 GIST 索引在一个极小数据集上工作
3. **临时表的 GIST 索引建索引成本几乎为零**（几千行）
4. **`ST_DWithin` 只在候选集上执行**，精确计算次数从百万级降到千级

---

## 变体：分块并行版本

如果 800 万行的数值过滤本身也需要时间（~10s），可以分块并行：

```sql
-- 按空间分块（每个 tile 一个并发连接）
-- Tile 1: 经度 [xmin, xmid], 纬度 [ymin, ymid]
-- Tile 2: 经度 [xmid, xmax], 纬度 [ymin, ymid]
-- Tile 3: 经度 [xmin, xmid], 纬度 [ymid, ymax]
-- Tile 4: 经度 [xmid, xmax], 纬度 [ymid, ymax]
```

---

## 何时使用此模式？

**适用**：
- 一张大表（>1M 行）需要空间邻近查询
- 数据分布广，GIST 索引选择性差
- 目标区域的 bounding box 远小于全表 extent

**不适用**：
- 小表（< 10 万行），直接 `ST_DWithin` + GIST 就够
- 查询条件本身就是按 bounding box 批量查询
- 数据已经在分区表中按空间分区

---

## 关键要点

> **数值范围的 B-tree 过滤比空间 GIST 索引的 bounding box 检查更快，因为它不需要遍历索引树。用应用的智慧弥补优化器的盲区。**

- 先缩候选集（数值范围），再做精确过滤（`ST_DWithin`）
- 候选集写入临时表 + 小表建 GIST 索引
- 8M → 15 秒，而非 300 秒 timeout
