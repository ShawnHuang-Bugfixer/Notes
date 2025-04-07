---
aliases:
  - 视图
date: 2025-04-07
tags:
  - MySQL基础
---
# 视图定义

以一张或多张表为基础，选择指定列构成视图（看作虚拟表，不存储数据）。视图的作用：
- **封装复杂 SQL** ：多表联查。
- 数据安全：暴露指定列，针对视图进行授权。
- 虚拟表结构独立：底层表结构变化时，视图可以保持不变。

# 视图检查条件

创建视图可以基于表也可以基于视图，基于视图时有两个检查选项：`with cascaded check option` 和 `with local check option`，对应级联检查和本地检查。级联检查会检查所有底层视图条件，本地检查仅仅检查当前视图条件。例如：

## 1. 级联检查
```sql
-- 基础视图
CREATE VIEW v1 AS 
SELECT * FROM t WHERE salary > 5000;

-- 基于v1创建另一个视图
CREATE VIEW v2 AS 
SELECT * FROM v1 WHERE salary < 10000
WITH CASCADED CHECK OPTION;

-- 尝试插入数据
INSERT INTO v2 VALUES (..., 6000);  -- 成功，满足v1和v2条件
INSERT INTO v2 VALUES (..., 4000);  -- 失败，不满足v1条件
INSERT INTO v2 VALUES (..., 11000); -- 失败，不满足v2条件
```
## 2. 本地检查
```sql
CREATE VIEW v2 AS 
SELECT * FROM v1 WHERE salary < 10000
WITH LOCAL CHECK OPTION;

INSERT INTO v2 VALUES (..., 4000);  -- 成功，只检查v2条件(不检查v1)
```

# MySQL 中视图不可更新的情况

在 MySQL 中，并非所有视图都可以进行更新操作（INSERT、UPDATE、DELETE）。以下是视图不可更新的常见情况：

## 1. 聚合函数或分组

如果视图定义包含以下内容，则不可更新：
- 使用聚合函数（SUM(), COUNT(), AVG(), MAX(), MIN()等）
- 使用 GROUP BY 子句
- 使用 HAVING 子句
```sql
CREATE VIEW sales_summary AS
SELECT product_id, SUM(quantity) AS total_quantity
FROM sales
GROUP BY product_id;
-- 此视图不可更新
```

## 2. 使用 DISTINCT

视图定义中包含 DISTINCT 关键字：
```sql
CREATE VIEW unique_customers AS
SELECT DISTINCT customer_id FROM orders;
-- 此视图不可更新
```

## 3. 使用 UNION 或 UNION ALL

视图由 UNION 或 UNION ALL 操作组合而成：
```sql
CREATE VIEW combined_orders AS
SELECT * FROM current_orders
UNION
SELECT * FROM archived_orders;
-- 此视图不可更新
```

## 4. 子查询引用不可更新视图

如果视图基于另一个不可更新的视图：
```sql
CREATE VIEW v1 AS SELECT * FROM table1 WHERE condition;
CREATE VIEW v2 AS SELECT * FROM v1 WHERE other_condition;
-- 如果v1不可更新，v2也不可更新
```