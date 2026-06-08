# Pattern 01: SRID 伪装陷阱

> **SRID Camouflage: `ST_SRID` says 4326, but coordinates are projected meters.**

---

## 症状

你有一张空间表，想和另一张 WGS84（SRID=4326）的表做空间 JOIN：

```sql
SELECT a.id, b.id
FROM land_parcels a
JOIN poi_points b
  ON ST_Intersects(a.geom, b.geom)
WHERE ST_SRID(a.geom) = 4326
  AND ST_SRID(b.geom) = 4326;
```

结果：**0 行**。

但你明明知道两个表的数据在地理上应该重叠——你用 QGIS 打开看过，地块覆盖了整个城区，POI 也在城区内。

更让人困惑的是，`ST_SRID()` 返回的都是 4326：

```sql
SELECT DISTINCT ST_SRID(geom) FROM land_parcels;
-- 返回: 4326

SELECT DISTINCT ST_SRID(geom) FROM poi_points;
-- 返回: 4326
```

SRID 一致，几何类型一致，为什么 JOIN 返回 0 行？

---

## 诊断

取几条样本数据的实际坐标值：

```sql
SELECT
    ST_AsText(ST_Centroid(geom)) AS centroid_wkt,
    ST_X(ST_Centroid(geom)) AS x,
    ST_Y(ST_Centroid(geom)) AS y,
    ST_SRID(geom) AS claimed_srid
FROM land_parcels
LIMIT 3;
```

你可能看到类似这样的输出：

| centroid_wkt | x | y | claimed_srid |
|---|---|---|---|
| POINT(39451234.12 3541234.56) | 39451234.12 | 3541234.56 | 4326 |

**这是一个红色警报。**

WGS84（EPSG:4326）的坐标范围：
- 经度：-180 ~ +180
- 纬度：-90 ~ +90

而这里 `x = 39,451,234` — 这显然不是经纬度，这是**投影坐标系**的米制坐标（很可能是 CGCS2000 / EPSG:4525 或类似的地方投影）。

**根因**：这条几何数据的 SRID 标签被人为设置成了 4326，但实际存储的坐标值来自一个投影坐标系。`ST_SRID()` 只读取元数据标签，不验证坐标值是否与该 SRID 匹配。

这就像一个箱子上贴着 "苹果" 的标签，但里面装的是橘子。PostGIS 信任这个标签。

---

## 真实影响：从 0 条到 1,699 条

在一个真实案例中：

- `land_parcels` 表有 **1,699 条地块数据**，坐标实际是 CGCS2000 投影坐标，但 SRID 标签是 4326
- `target_poi` 表有几千条 POI，SRID 正确为 4326
- 直接 JOIN：**0 条匹配**（因为 PostGIS 按 SRID 标签认为坐标系一致，不去转换，导致两个几何体在空间上根本不在同一位置）
- 修复 SRID 后 JOIN：**1,699 条匹配**（全部地块都能找到 POI）

**这不是性能问题，这是正确性问题。** 你的查询静默地返回了错误结果（0 条），没有任何报错。

---

## 检测脚本

在你的数据库中运行此通用检测脚本：

```sql
-- 检测 SRID 伪装：SRID=4326 但坐标值在经纬度范围之外
WITH suspicious AS (
    SELECT
        'your_table'::text AS table_name,
        COUNT(*) AS total_rows,
        COUNT(*) FILTER (
            WHERE ST_SRID(geom) = 4326
              AND (
                  ST_X(ST_Centroid(geom)) < -180
                  OR ST_X(ST_Centroid(geom)) > 180
                  OR ST_Y(ST_Centroid(geom)) < -90
                  OR ST_Y(ST_Centroid(geom)) > 90
              )
        ) AS camouflaged_rows
    FROM your_table
    WHERE geom IS NOT NULL
)
SELECT
    table_name,
    total_rows,
    camouflaged_rows,
    ROUND(100.0 * camouflaged_rows / NULLIF(total_rows, 0), 1) AS pct
FROM suspicious
WHERE camouflaged_rows > 0;
```

如果 `camouflaged_rows > 0`，你中招了。

---

## 修复

### 第一步：确定真实坐标系

需要业务知识来判断。常见可能性：

| 坐标范围（x 值） | 可能坐标系 | EPSG |
|---|---|---|
| 39,xxx,xxx ~ 39,xxx,xxx | CGCS2000 3-degree GK Zone 40 | 4525 |
| 39,xxx,xxx ~ 40,xxx,xxx | CGCS2000 / 3-degree GK CM 117E | 4547 |
| ~12,xxx,xxx | UTM Zone 50N (WGS84) | 32650 |
| ~500,000 | UTM Zone (典型) | 326xx |

### 第二步：修复（就地更新）

```sql
-- 假设真实坐标系是 CGCS2000 / EPSG:4525
-- 步骤1：先改 SRID 标签为真实值
UPDATE your_table
SET geom = ST_SetSRID(geom, 4525)
WHERE ST_SRID(geom) = 4326
  AND ST_X(ST_Centroid(geom)) > 180;  -- 安全阀，只改坐标异常的

-- 步骤2：转为 WGS84
UPDATE your_table
SET geom = ST_Transform(geom, 4326)
WHERE ST_SRID(geom) = 4525;
```

### 第二步（替代方案）：在查询中修复

如果不想修改原表：

```sql
-- 在查询中动态修复
SELECT *
FROM your_table
WHERE ST_Intersects(
    ST_Transform(ST_SetSRID(geom, 4525), 4326),
    ST_SetSRID(ST_MakePoint(lon, lat), 4326)
);
```

**注意**：这种方式无法使用空间索引，仅适合小数据量。

---

## 预防

1. **数据入库时严格校验**：入库脚本应该检查坐标值是否在 SRID 的合法范围内
2. **使用 `AddGeometryColumn`** 而非手动设置 SRID，它会在写入时做基本校验
3. **定期运行上述检测脚本**，作为数据质量监控的一部分

```sql
-- 入库校验模板
CREATE OR REPLACE FUNCTION validate_4326_coords(geom geometry)
RETURNS boolean AS $$
BEGIN
    IF ST_SRID(geom) != 4326 THEN
        RETURN false;
    END IF;
    IF ST_X(ST_Centroid(geom)) < -180 OR ST_X(ST_Centroid(geom)) > 180 THEN
        RAISE WARNING 'X coordinate % out of range for SRID 4326', ST_X(ST_Centroid(geom));
        RETURN false;
    END IF;
    IF ST_Y(ST_Centroid(geom)) < -90 OR ST_Y(ST_Centroid(geom)) > 90 THEN
        RAISE WARNING 'Y coordinate % out of range for SRID 4326', ST_Y(ST_Centroid(geom));
        RETURN false;
    END IF;
    RETURN true;
END;
$$ LANGUAGE plpgsql;
```

---

## 关键要点

> **`ST_SRID()` 是标签读取器，不是坐标验证器。永远不要信任它。**

- 你的空间 JOIN 静默返回 0 行 → 第一时间检查实际坐标值范围
- 坐标值是 7-8 位数，但 SRID=4326 → SRID 伪装
- 修复方法：`ST_SetSRID` 重置标签 → `ST_Transform` 转换
- 1,699 条数据从 "不存在" 到 "全部匹配"，这就是 SRID 伪装的威力
