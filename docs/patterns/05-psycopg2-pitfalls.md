# Pattern 05: psycopg2 三大坑

> **psycopg2 Pitfalls: % escaping, ST_Distance returning str, and column index chaos.**

---

## 概述

`psycopg2` 是 Python 连接 PostgreSQL 的事实标准，但它与 PostGIS 配合时有三个高频陷阱，每个都能浪费你数小时。以下按"先症状、再根因、最后修复"的实战风格逐一拆解。

---

## 坑 1：`%` 转义 —— LIKE 查询的无声杀手

### 症状

你写了一个简单的模糊搜索：

```python
import psycopg2

conn = psycopg2.connect("dbname=mydb user=me")
cur = conn.cursor()

keyword = "Beijing"
cur.execute(
    "SELECT id, name FROM cities WHERE name ILIKE '%Beijing%'"
)
```

运行结果：

```
IndexError: tuple index out of range
```

你检查 SQL，语法完全正确。在 pgAdmin 里跑同一句 SQL，完美返回结果。但在 psycopg2 里就报 `IndexError`。

换成参数化查询也一样：

```python
cur.execute(
    "SELECT * FROM cities WHERE name ILIKE '%s%'",
    ("Beijing",)
)
```

还是报 `IndexError`。

### 根因

psycopg2 使用 `%` 作为参数占位符（类似 Python 的 `%s` 格式化）。当你写 `'%Beijing%'` 时，psycopg2 会把第一个 `%` 解释为参数占位符的开始，但找不到对应的参数，于是抛出 `IndexError`。

**不是 PostgreSQL 的问题，是 psycopg2 在客户端解析 SQL 时出错。**

### 修复

**规则**：在 psycopg2 的 SQL 字符串中，所有 `%` 必须双写为 `%%`。

```python
# ✅ 方法 1：固定字符串中的 % 双写
cur.execute(
    "SELECT id, name FROM cities WHERE name ILIKE '%%Beijing%%'"
)

# ✅ 方法 2：参数化查询 —— 推荐
# 用 %% 写 LIKE 通配符，用 %s 传参数
cur.execute(
    "SELECT id, name FROM cities WHERE name ILIKE '%%' || %s || '%%'",
    ("Beijing",)
)

# ✅ 方法 3：构造 LIKE 模式
pattern = f"%{keyword}%"
cur.execute(
    "SELECT id, name FROM cities WHERE name ILIKE %s",
    (pattern,)
)

# ❌ 错误：单 % 在 LIKE 中
cur.execute("SELECT * FROM t WHERE name ILIKE '%Beijing%'")
# → IndexError: tuple index out of range
```

### 记忆口诀

> **"在 psycopg2 中，SQL 里的每个 `%` 都要双写 `%%`，除了 `%s` 作为参数占位符。"**

---

## 坑 2：`ST_Distance` 返回字符串，不是浮点数

### 症状

你获取了两点之间的距离，想做数值比较：

```python
cur.execute("""
    SELECT ST_Distance(a.geom, b.geom)
    FROM table_a a, table_b b
    WHERE a.id = 1 AND b.id = 2
""")
result = cur.fetchone()[0]
print(type(result))  # <class 'str'>
print(result)        # "1234.567890123"

if result < 5000:   # TypeError: '<' not supported between 'str' and 'int'
    print("close")
```

```python
TypeError: '<' not supported between instances of 'str' and 'int'
```

你打印 `type(result)`，发现是 **`str`**，不是 `float`。

### 根因

PostGIS 的 `ST_Distance` 返回的是 `double precision`（PostgreSQL 内部是 `float8`）。但 psycopg2 的默认类型转换可能不涵盖所有 PostGIS 返回类型，导致某些数值类型以字符串形式返回。

特别容易发生在：
- `ST_Distance` 返回的结果
- `ST_Area` 返回的结果
- 任何 `double precision` 类型的 PostGIS 函数

### 修复

**方法 1：显式转换（最简单）**

```python
result = float(cur.fetchone()[0])
```

**方法 2：注册类型转换器**

```python
import psycopg2
from psycopg2.extensions import register_adapter, AsIs

# 注册 float8 → Python float 的自动转换
import psycopg2.extensions

def cast_float8(value, cur):
    if value is None:
        return None
    return float(value)

# 701 是 float8 的 OID
psycopg2.extensions.register_type(
    psycopg2.extensions.new_type((701,), 'FLOAT8', cast_float8)
)
```

**方法 3：在 SQL 中做类型转换**

```sql
SELECT ST_Distance(a.geom, b.geom)::numeric AS dist
```

然后在 Python 中用 `float()` 或 `Decimal()` 接收。

### 最佳实践

```python
def fetch_distance(cur, sql, params=None):
    """安全获取 PostGIS 距离值"""
    cur.execute(sql, params)
    val = cur.fetchone()[0]
    return float(val) if val is not None else None

# 使用
dist = fetch_distance(cur, """
    SELECT ST_Distance(a.geom, b.geom)
    FROM table_a a, table_b b
    WHERE a.id = %s AND b.id = %s
""", (1, 2))
```

---

## 坑 3：列索引混乱（SHP 导出 + psycopg2 通用）

### 症状

你用 `pgsql2shp` 导出数据：

```bash
pgsql2shp -f output.shp -h localhost -u user -g geom_col db \
  "SELECT attr1, geom_col, attr2 FROM my_table"
```

报错：

```
Table "attr1" does not exist
```

或：

```
Error: column "geom_col" specified with -g does not appear in the query
```

### 根因

`pgsql2shp` 要求 **geometry 列必须是 SELECT 的第一列**。如果你指定了 `-g geom_col` 但它不是第一列，或者根本没有 geometry 列在第一位，就会报错。

另外，在 psycopg2 中按索引访问列时也很容易出错：

```python
# 危险：依赖列顺序
row = cur.fetchone()
geom = row[0]   # 假设 geometry 在第一列 — 容易搞混
name = row[1]
```

### 修复

**SHP 导出**：

```bash
# ✅ geometry 列放在 SELECT 第一位
pgsql2shp -f output.shp -h localhost -u user db \
  "SELECT geom_col, attr1, attr2 FROM my_table"
# geom_col 在第一位，不需要 -g 参数
```

**psycopg2 中按名称访问**：

```python
from psycopg2.extras import RealDictCursor

# ✅ 方法 1：使用 DictCursor
cur = conn.cursor(cursor_factory=RealDictCursor)
cur.execute("SELECT geom_col, attr1, attr2 FROM my_table")
row = cur.fetchone()
geom = row['geom_col']  # 按名称访问，不依赖列顺序

# ✅ 方法 2：手动构建列名映射
cur.execute("SELECT geom_col, attr1, attr2 FROM my_table")
col_names = [desc[0] for desc in cur.description]
row = cur.fetchone()
col_map = dict(zip(col_names, row))
geom = col_map['geom_col']
```

---

## 总结：三大坑速查

| 陷阱 | 症状 | 修复 |
|------|------|------|
| `%` 转义 | `IndexError: tuple index out of range` | 双写 `%%`：`'%%Beijing%%'` |
| `ST_Distance` 返回 str | `TypeError: '<' not supported` | `float(cur.fetchone()[0])` |
| 列索引混乱 | `Table "attr1" does not exist` | geometry 放 SELECT 第一位；Python 用 DictCursor |

---

## 一键自检脚本

将此脚本放在每个 psycopg2 + PostGIS 项目开头：

```python
import psycopg2
from psycopg2.extras import RealDictCursor

def check_psycopg2_postgis(conn):
    """三大坑自检"""
    cur = conn.cursor()

    # 坑 1：% 转义
    try:
        cur.execute("SELECT '%%test%%' AS result")
        assert cur.fetchone()[0] == '%test%', "坑1: % 转义不符合预期"
        print("✅ 坑1: % 转义正常")
    except Exception as e:
        print(f"❌ 坑1: % 转义异常 — {e}")

    # 坑 2：ST_Distance 类型
    try:
        cur.execute("""
            SELECT ST_Distance(
                ST_SetSRID(ST_MakePoint(0, 0), 4326)::geography,
                ST_SetSRID(ST_MakePoint(1, 1), 4326)::geography
            )
        """)
        val = cur.fetchone()[0]
        dist = float(val)
        assert dist > 0, "距离应为正数"
        print(f"✅ 坑2: ST_Distance 返回 float ({dist:.1f}m)")
    except Exception as e:
        print(f"❌ 坑2: ST_Distance 异常 — {e}")

    # 坑 3：列索引
    try:
        cur = conn.cursor(cursor_factory=RealDictCursor)
        cur.execute("SELECT 1 AS a, 2 AS b")
        row = cur.fetchone()
        assert row['a'] == 1 and row['b'] == 2
        print("✅ 坑3: DictCursor 正常")
    except Exception as e:
        print(f"❌ 坑3: DictCursor 异常 — {e}")

    print("\n自检完成。")
```

---

## 关键要点

> **在 psycopg2 中，每个 `%` 都要双写，每个 `ST_Distance` 都要 `float()`，每个行都用 RealDictCursor。**

- LIKE 中的 `%` → `%%`
- `ST_Distance/ST_Area/ST_Length` 返回值 → `float()`
- SHP 导出：geometry 列放 SELECT 第一位
- Python：用 `RealDictCursor` 按名称访问列
