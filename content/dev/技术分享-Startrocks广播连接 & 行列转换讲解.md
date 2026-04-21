---
title: StarRocks 技术分析：广播连接 & 行列转换
date: 2026-04-21
description: 本文章由刘行通投稿
toc: true
slug: starrocks-broadcast-join-column-row-conversion
categories:
    - Dev
    - 技术分享
tags:
    - StarRocks
---

# StarRocks 技术分析：广播连接 & 行列转换

---

## 一、广播连接（Broadcast Join）

### 1.1 原理

广播连接是 StarRocks 分布式查询中的一种 Join 策略。当两张表进行 JOIN 时，StarRocks 会将**较小的表（右表）完整地广播复制到所有参与计算的节点**，使每个节点都能独立完成与大表（左表）对应分片的 Join，无需数据 Shuffle。

```
左表（大表）                右表（小表）
分片1  分片2  分片3         整张表
  ↓      ↓      ↓           ↓  ↓  ↓
Node1  Node2  Node3  ← 广播到每个节点
  └────JOIN   JOIN   JOIN────┘
```

### 1.2 适用场景

| 场景 | 说明 |
|------|------|
| 大表 JOIN 小表 | 右表数据量小，广播成本低 |
| 维度表关联 | 事实表 JOIN 维度表（字典表） |
| 避免数据倾斜 | 小表广播比 Shuffle 更稳定 |

### 1.3 SQL 使用方式

StarRocks 支持通过 Hint 强制指定广播连接：

```sql
-- 方式一：使用 [broadcast] hint（推荐）
SELECT /*+ SET_VAR(broadcast_row_limit=15000000) */
    o.order_id,
    c.customer_name
FROM orders o
JOIN [broadcast] customer c
    ON o.customer_id = c.id;

-- 方式二：使用 broadcast join hint（旧版）
SELECT
    o.order_id,
    c.customer_name
FROM orders o
JOIN customer c [broadcast]
    ON o.customer_id = c.id;
```

### 1.4 关键参数

```sql
-- 控制广播阈值（右表行数超过此值则不允许广播）
SET broadcast_row_limit = 15000000;  -- 默认 1500万行

-- 查看执行计划，确认是否走广播
EXPLAIN SELECT ...;
-- 在 PLAN 中看到 BROADCAST 字样即为广播连接
```

### 1.5 执行计划示例

```
PLAN FRAGMENT 1
  OUTPUT EXPRS:
  PARTITION: RANDOM

  STREAM DATA SINK
    EXCHANGE ID: 02
    BROADCAST          ← 右表被广播

  1:OlapScanNode
     TABLE: customer
```

### 1.6 注意事项

- 右表过大时广播会消耗大量内存，导致 OOM，需严格控制 `broadcast_row_limit`
- StarRocks CBO（基于代价优化器）会自动选择最优 Join 策略，多数情况无需手动指定
- 若需强制关闭广播，可用 `[shuffle]` hint

---

## 二、复杂视图 + 广播导致数据膨胀问题

> **场景**：建立了复杂视图，在使用视图时又嵌套子查询，导致数值异常（偏大/翻倍），像是数据被重复合并。

### 2.1 问题本质

当视图内部已经做了 JOIN，外层查询再对这个视图做子查询或再次 JOIN 时，StarRocks 会**展开视图重新组合执行计划**，极易产生**笛卡尔积式的数据膨胀**，导致数值被重复累加。

```
视图内部：A JOIN B  →  已经是 1:N 关联
外层查询：再 JOIN C 或子查询  →  变成 1:N:M，数值被重复累加
```

### 2.2 典型问题模式

#### 模式一：视图内已聚合，外层又 JOIN 了原始明细表

```sql
-- 视图内部已聚合
CREATE VIEW v_order_summary AS
SELECT customer_id, SUM(amount) AS total_amount
FROM orders
GROUP BY customer_id;

-- ❌ 外层使用视图时又 JOIN 了原始明细 → total_amount 被重复展开
SELECT v.customer_id, v.total_amount, o.order_date
FROM v_order_summary v
JOIN orders o ON v.customer_id = o.customer_id;
```

#### 模式二：视图内是 1:N 关系，外层再嵌套一层视图聚合

```sql
-- 视图内部 orders JOIN order_items（1:N 关系）
CREATE VIEW v_order_detail AS
SELECT o.order_id, o.customer_id, oi.product_id, oi.qty
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id;

-- ❌ 再套一层视图引用，且又关联其他表 → 乘数效应叠加
CREATE VIEW v_customer_report AS
SELECT d.customer_id, c.name, SUM(d.qty) AS total_qty
FROM v_order_detail d
JOIN customer c ON d.customer_id = c.id
GROUP BY d.customer_id, c.name;
```

#### 模式三：广播右表关联键不唯一，导致每行匹配多条

```sql
-- ❌ dim_table.type_id 存在重复值，广播后每条大表记录匹配多条
SELECT a.id, b.value
FROM big_table a
JOIN [broadcast] dim_table b ON a.type_id = b.type_id;
-- 结果：数据行数 = 大表行数 × 右表重复数，数值严重虚高
```

### 2.3 排查步骤

**第一步：用 EXPLAIN 查看视图展开后的完整执行计划**

```sql
EXPLAIN SELECT * FROM your_complex_view WHERE ...;
-- 重点关注：JOIN 顺序、是否存在多个 AGGREGATE 节点、数据流向
```

**第二步：逐层拆开视图，用 COUNT 验证每层是否有重复行**

```sql
-- 检查视图中间层是否已产生重复
SELECT key_col, COUNT(*) AS cnt
FROM (/* 视图内部SQL，逐段拆出来 */) t
GROUP BY key_col
HAVING cnt > 1;
-- 有结果 → 该层已产生重复行，问题在此层或更内层
```

**第三步：确认广播右表的 JOIN key 是否唯一**

```sql
-- 广播表关联键唯一性检查
SELECT type_id, COUNT(*) AS cnt
FROM dim_table
GROUP BY type_id
HAVING COUNT(*) > 1;
-- 有结果 → 右表不唯一 → 广播后必然膨胀
```

**第四步：对比总行数与去重行数**

```sql
-- 快速判断结果集是否有重复
SELECT
    COUNT(*)                  AS total_rows,
    COUNT(DISTINCT order_id)  AS distinct_orders
FROM v_your_view;
-- 两值不一致 → 有重复行，数值必然偏大
```

### 2.4 解决方案

#### 方案一：广播前先对右表去重（最直接）

```sql
-- ✅ 广播前用子查询去重，确保关联键唯一
SELECT a.id, b.value
FROM big_table a
JOIN [broadcast] (
    SELECT DISTINCT type_id, value FROM dim_table
) b ON a.type_id = b.type_id;
```

#### 方案二：将聚合逻辑下沉到视图内部，不暴露明细行

```sql
-- ✅ 视图内完成聚合，外层直接使用，不再关联原始明细
CREATE VIEW v_customer_stats AS
SELECT
    o.customer_id,
    SUM(oi.amount)     AS total_amount,
    COUNT(o.order_id)  AS order_cnt
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.customer_id;

-- 外层直接用聚合结果，不再 JOIN 原始表
SELECT cs.customer_id, c.name, cs.total_amount
FROM v_customer_stats cs
JOIN [broadcast] customer c ON cs.customer_id = c.id;
```

#### 方案三：用 CTE（WITH）替代复杂视图嵌套，结构透明可控

```sql
-- ✅ 用 CTE 拍平逻辑，每一层清晰可见，避免视图套视图
WITH
order_agg AS (
    -- 第一层：先在订单粒度聚合
    SELECT order_id, customer_id, SUM(amount) AS order_total
    FROM orders
    GROUP BY order_id, customer_id
),
customer_agg AS (
    -- 第二层：再在客户粒度聚合
    SELECT customer_id, SUM(order_total) AS total
    FROM order_agg
    GROUP BY customer_id
),
enriched AS (
    -- 第三层：最后广播关联维度表
    SELECT ca.customer_id, c.name, ca.total
    FROM customer_agg ca
    JOIN [broadcast] customer c ON ca.customer_id = c.id
)
SELECT * FROM enriched;
```

#### 方案四：使用 ROW_NUMBER 去重后再广播

```sql
-- ✅ 右表有重复时，先取每个 key 的最新/最优一条
WITH dedup_dim AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY type_id ORDER BY updated_at DESC) AS rn
    FROM dim_table
)
SELECT a.id, b.value
FROM big_table a
JOIN [broadcast] (SELECT * FROM dedup_dim WHERE rn = 1) b
    ON a.type_id = b.type_id;
```

### 2.5 根本原则总结

| 问题根因 | 解决原则 |
|----------|----------|
| 视图嵌套视图，逻辑不透明 | 改用 CTE，每层粒度清晰 |
| 广播右表 JOIN key 不唯一 | 广播前 `DISTINCT` 或 `ROW_NUMBER` 去重 |
| 视图内 1:N JOIN 后外层再聚合 | 聚合逻辑下沉到视图内部 |
| 不确定哪层产生膨胀 | 逐层 `COUNT(*) vs COUNT(DISTINCT key)` 定位 |
| 视图内已聚合，外层又关联明细 | 严禁将聚合视图与其源明细表再次 JOIN |

---

## 三、行转列（PIVOT / 列聚合展开）

### 3.1 原理

行转列（Pivot）是将多行数据中某一列的不同**值**变成多个独立的**列**，通常配合聚合函数使用。

**原始数据：**

| student | subject | score |
|---------|---------|-------|
| 张三    | 语文    | 90    |
| 张三    | 数学    | 85    |
| 李四    | 语文    | 78    |
| 李四    | 数学    | 92    |

**转换后：**

| student | 语文 | 数学 |
|---------|------|------|
| 张三    | 90   | 85   |
| 李四    | 78   | 92   |

### 3.2 实现方式

#### 方式一：CASE WHEN + GROUP BY（推荐，兼容性强）

```sql
SELECT
    student,
    MAX(CASE WHEN subject = '语文' THEN score END) AS `语文`,
    MAX(CASE WHEN subject = '数学' THEN score END) AS `数学`,
    MAX(CASE WHEN subject = '英语' THEN score END) AS `英语`
FROM student_scores
GROUP BY student;
```

#### 方式二：IF + GROUP BY（StarRocks 支持）

```sql
SELECT
    student,
    MAX(IF(subject = '语文', score, NULL)) AS `语文`,
    MAX(IF(subject = '数学', score, NULL)) AS `数学`,
    MAX(IF(subject = '英语', score, NULL)) AS `英语`
FROM student_scores
GROUP BY student;
```

#### 方式三：SUM + CASE WHEN（数值类型求和场景）

```sql
SELECT
    department,
    SUM(CASE WHEN month = '2024-01' THEN amount ELSE 0 END) AS `1月`,
    SUM(CASE WHEN month = '2024-02' THEN amount ELSE 0 END) AS `2月`,
    SUM(CASE WHEN month = '2024-03' THEN amount ELSE 0 END) AS `3月`
FROM sales
GROUP BY department;
```

### 3.3 动态行转列（借助外部程序）

StarRocks 原生不支持动态 PIVOT（列名在查询时才确定），需在应用层先查出所有可能的列值，再拼接 SQL：

```python
# Python 示例：动态生成行转列 SQL
import starrocks_connector as conn

# Step1: 查出所有科目
subjects = conn.query("SELECT DISTINCT subject FROM student_scores")

# Step2: 动态拼接 CASE WHEN
case_clauses = ",\n".join([
    f"MAX(CASE WHEN subject = '{s}' THEN score END) AS `{s}`"
    for s in subjects
])

sql = f"""
SELECT student, {case_clauses}
FROM student_scores
GROUP BY student
"""
result = conn.query(sql)
```

---

## 四、列转行（UNPIVOT / 行展开）

### 4.1 原理

列转行（Unpivot）与行转列相反，将多个**列**的数据展开成多**行**，通常用于宽表变窄表、数据归一化。

**原始数据（宽表）：**

| student | 语文 | 数学 | 英语 |
|---------|------|------|------|
| 张三    | 90   | 85   | 88   |
| 李四    | 78   | 92   | 80   |

**转换后（窄表）：**

| student | subject | score |
|---------|---------|-------|
| 张三    | 语文    | 90    |
| 张三    | 数学    | 85    |
| 张三    | 英语    | 88    |
| 李四    | 语文    | 78    |
| 李四    | 数学    | 92    |
| 李四    | 英语    | 80    |

### 3.2 实现方式

#### 方式一：UNION ALL（通用方式，StarRocks 完全支持）

```sql
SELECT student, '语文' AS subject, `语文` AS score FROM student_wide
UNION ALL
SELECT student, '数学' AS subject, `数学` AS score FROM student_wide
UNION ALL
SELECT student, '英语' AS subject, `英语` AS score FROM student_wide
ORDER BY student, subject;
```

#### 方式二：LATERAL + VALUES（StarRocks 2.5+ 支持）

StarRocks 支持 `LATERAL` 子查询配合 `VALUES` 做列转行，性能优于多次 UNION ALL：

```sql
SELECT
    t.student,
    v.subject,
    v.score
FROM student_wide t
CROSS JOIN LATERAL (
    VALUES
        ('语文', t.`语文`),
        ('数学', t.`数学`),
        ('英语', t.`英语`)
) AS v(subject, score);
```

#### 方式三：使用 unnest（数组场景）

当宽表数据以数组存储时，可使用 `unnest` 展开：

```sql
-- 假设 scores 列为 ARRAY 类型
SELECT
    student,
    subject,
    score
FROM student_array_table,
LATERAL unnest(subjects, scores) AS t(subject, score);
```

### 4.3 多列同时展开（复杂场景）

```sql
-- 展开多个维度列（月份 + 指标类型）
SELECT
    department,
    month,
    metric_type,
    value
FROM sales_wide t
CROSS JOIN LATERAL (
    VALUES
        ('2024-01', '销售额', t.sales_jan),
        ('2024-01', '成本',   t.cost_jan),
        ('2024-02', '销售额', t.sales_feb),
        ('2024-02', '成本',   t.cost_feb)
) AS v(month, metric_type, value)
WHERE value IS NOT NULL;
```

---

## 五、性能优化建议

### 广播连接优化

```sql
-- 1. 确认右表满足广播阈值
SELECT COUNT(*) FROM dim_table;  -- 确保行数在 broadcast_row_limit 以内

-- 2. 适当调大阈值（谨慎，需评估内存）
SET broadcast_row_limit = 20000000;

-- 3. 查看实际执行计划
EXPLAIN COSTS SELECT ...;
```

### 行列转换优化

| 优化点 | 建议 |
|--------|------|
| 过滤前置 | 在 CASE WHEN / UNION ALL 前先用 WHERE 减少数据量 |
| 分区裁剪 | 确保分区列出现在 WHERE 条件中 |
| 物化视图 | 对频繁行转列的固定维度，可创建同步物化视图预计算 |
| LATERAL 替代 UNION ALL | 减少多次全表扫描，提升性能 |

### 物化视图示例（行转列固化）

```sql
-- 创建同步物化视图（自动维护）
CREATE MATERIALIZED VIEW mv_score_pivot
AS
SELECT
    student,
    MAX(IF(subject = '语文', score, NULL)) AS score_chinese,
    MAX(IF(subject = '数学', score, NULL)) AS score_math,
    MAX(IF(subject = '英语', score, NULL)) AS score_english
FROM student_scores
GROUP BY student;

-- 查询时自动命中物化视图
SELECT student, score_chinese, score_math FROM student_scores;
```

---

## 六、功能对比总结

| 功能 | 实现方式 | StarRocks 支持版本 | 适用场景 |
|------|----------|-------------------|----------|
| 广播连接 | `JOIN [broadcast]` Hint | 全版本 | 大表 JOIN 小表 |
| 行转列（静态） | CASE WHEN + GROUP BY | 全版本 | 列值固定 |
| 行转列（动态） | 应用层拼接 SQL | 全版本 | 列值不固定 |
| 列转行 | UNION ALL | 全版本 | 宽表转窄表 |
| 列转行（高效） | CROSS JOIN LATERAL VALUES | 2.5+ | 宽表转窄表 |
| 数组展开 | unnest + LATERAL | 2.5+ | 数组类型列展开 |
| 预计算固化 | 同步物化视图 | 2.4+ | 高频固定维度转换 |

---

> **参考文档**：[StarRocks 官方文档](https://docs.starrocks.io/zh/docs/introduction/StarRocks_intro/)
