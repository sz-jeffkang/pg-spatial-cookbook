# Pattern 03: OD 线生成

> **OD Line Generation: Using start + end coordinates correctly to avoid zero-length lines.**

---

## 症状

你有一张 OD（Origin-Destination）表，包含起点和终点的坐标：

```sql
SELECT * FROM od_pairs LIMIT 5;

-- origin_lon | origin_lat | dest_lon | dest_lat
-- -----------+------------+----------+----------
--    116.397 |    39.908  |  121.473 |  31.230
--    113.264 |    23.129  |  114.057 |  22.543
```

你需要生成从起点到终点的线段（LineString），于是写了：

```sql
SELECT
    id,
    ST_MakeLine(
        ST_SetSRID(ST_MakePoint(origin_lon, origin_lat), 4326),
        ST_SetSRID(ST_MakePoint(dest_lon, dest_lat), 4326)
    ) AS geom
FROM od_pairs;
```

在 QGIS 中打开结果，几百条线看起来正常——从北京到上海、广州到深圳都有。

但当你运行验证脚本时，发现：

```
Total OD lines:   10,000
Zero-length:      1,247  (12.5%)
Same start/end:   1,247
```

**12.5% 的线长度为零！** 这些线的起点和终点完全相同。

---

## 根因

OD 数据常见两类错误：

### 错误 1：起点和终点用了同一组坐标

最典型的是复制粘贴错误——Excel 中从 origin 列复制到 destination 列时忘了改值：

```csv
origin_lon,origin_lat,dest_lon,dest_lat
116.397,39.908,116.397,39.908    ← 起终点完全相同
```

### 错误 2：经纬度值刚好在浮点精度内相等

某些数据源（如基站聚合、小区级定位）会把多个记录的起点或终点近似到同一个坐标。如果两个点的坐标在小数点后 6 位内完全相同，`ST_MakeLine` 生成的就是零长度线。

### 错误 3：NULL 值导致 ST_MakePoint 返回 NULL

如果 origin 或 destination 的经纬度中有 NULL，`ST_MakePoint` 返回 NULL，`ST_MakeLine(NULL, geom)` 返回 NULL（整条线消失）。

---

## 验证脚本

在生成 OD 线之后，务必运行此验证脚本：

```sql
-- ============================================
-- OD 线质量验证
-- ============================================
WITH od_lines AS (
    SELECT
        id,
        origin_lon, origin_lat,
        dest_lon, dest_lat,
        ST_MakeLine(
            ST_SetSRID(ST_MakePoint(origin_lon, origin_lat), 4326),
            ST_SetSRID(ST_MakePoint(dest_lon, dest_lat), 4326)
        ) AS geom
    FROM od_pairs
    WHERE origin_lon IS NOT NULL
      AND origin_lat IS NOT NULL
      AND dest_lon IS NOT NULL
      AND dest_lat IS NOT NULL
)
SELECT
    -- 总数
    COUNT(*) AS total_lines,

    -- 零长度线（起终点坐标完全相同，容差 6 位小数）
    COUNT(*) FILTER (
        WHERE ROUND(origin_lon::numeric, 6) = ROUND(dest_lon::numeric, 6)
          AND ROUND(origin_lat::numeric, 6) = ROUND(dest_lat::numeric, 6)
    ) AS zero_length_lines,

    -- 异常短线（< 10 米，可能不是真正的 OD）
    COUNT(*) FILTER (
        WHERE ST_Length(geom::geography) < 10
          AND ST_Length(geom::geography) > 0
    ) AS very_short_lines,

    -- NULL 几何（坐标中有 NULL）
    (SELECT COUNT(*) FROM od_pairs
     WHERE origin_lon IS NULL OR origin_lat IS NULL
        OR dest_lon IS NULL OR dest_lat IS NULL
    ) AS null_geom_lines,

    -- 各占比例
    ROUND(100.0 * COUNT(*) FILTER (
        WHERE ROUND(origin_lon::numeric, 6) = ROUND(dest_lon::numeric, 6)
          AND ROUND(origin_lat::numeric, 6) = ROUND(dest_lat::numeric, 6)
    ) / NULLIF(COUNT(*), 0), 1) AS zero_pct,

    ROUND(100.0 * COUNT(*) FILTER (
        WHERE ST_Length(geom::geography) < 10
          AND ST_Length(geom::geography) > 0
    ) / NULLIF(COUNT(*), 0), 1) AS short_pct

FROM od_lines;
```

输出示例：

```
 total_lines | zero_length_lines | very_short_lines | null_geom_lines | zero_pct | short_pct
-------------+-------------------+------------------+-----------------+----------+-----------
       10000 |              1247 |               53 |              28 |     12.5 |      0.5
```

---

## 修复

### 修复 1：在查询中排除

```sql
SELECT
    id,
    ST_MakeLine(
        ST_SetSRID(ST_MakePoint(origin_lon, origin_lat), 4326),
        ST_SetSRID(ST_MakePoint(dest_lon, dest_lat), 4326)
    ) AS geom
FROM od_pairs
WHERE origin_lon IS NOT NULL
  AND origin_lat IS NOT NULL
  AND dest_lon IS NOT NULL
  AND dest_lat IS NOT NULL
  -- 排除起终点相同（容差 6 位小数，约 0.1 米）
  AND NOT (
      ROUND(origin_lon::numeric, 6) = ROUND(dest_lon::numeric, 6)
      AND ROUND(origin_lat::numeric, 6) = ROUND(dest_lat::numeric, 6)
  );
```

### 修复 2：标记而非删除

如果零长度线可能是合法的（如同一建筑物内的不同楼层），标记而非删除：

```sql
SELECT
    id,
    ST_MakeLine(
        ST_SetSRID(ST_MakePoint(origin_lon, origin_lat), 4326),
        ST_SetSRID(ST_MakePoint(dest_lon, dest_lat), 4326)
    ) AS geom,
    CASE
        WHEN origin_lon IS NULL OR dest_lon IS NULL THEN 'null'
        WHEN ROUND(origin_lon::numeric, 6) = ROUND(dest_lon::numeric, 6)
         AND ROUND(origin_lat::numeric, 6) = ROUND(dest_lat::numeric, 6)
        THEN 'zero_length'
        WHEN ST_Length(
            ST_MakeLine(
                ST_SetSRID(ST_MakePoint(origin_lon, origin_lat), 4326),
                ST_SetSRID(ST_MakePoint(dest_lon, dest_lat), 4326)
            )::geography
        ) < 10 THEN 'very_short'
        ELSE 'ok'
    END AS quality_flag
FROM od_pairs;
```

---

## 最佳实践

1. **永远验证起终点是否不同** — 在生成几何之后立即运行验证脚本
2. **设置合理的最短距离阈值** — 根据业务含义（如同一建筑物内 < 50m 可能合法）
3. **保留原始坐标列** — 不要只保留生成的 geometry，保留 `origin_lon/origin_lat/dest_lon/dest_lat` 便于调试
4. **注意 ST_MakePoint 的参数顺序** — `ST_MakePoint(lon, lat)`，不是 `(lat, lon)`

---

## 关键要点

> **OD 线零长度最常见的根因：数据源中起点和终点列被填入了相同的值。12.5% 的错误率并不罕见。**

- 生成 OD 线后立即验证
- 用 `ROUND(x::numeric, 6)` 做容差比较，避免浮点问题
- 区分零长度（数据错误）和极短线（业务含义）
- 保留原始列，方便回溯
