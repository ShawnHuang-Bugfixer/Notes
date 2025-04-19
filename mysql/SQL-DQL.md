---
aliases:
  - 数据查询语言
date: 2025-04-02
tags:
  - MySQL基础
---
数据查询语言分为**基础查询、条件查询、分组查询结合聚合函数、排序查询、分页查询**。重点理解**查询执行流程**。

# 查询执行流程

![[Pasted image 20250402221537.png]]
# 多表查询

本质是 ER 模型的实现，表之间的关系主要包括 **1:1、1:n、n:n**。多表查询需要消除无效的 **笛卡尔积**。

多表查询主要分为 **内连接、外连接、自连接、联合查询、子查询**。从集合角度来看

![[attachments/Pasted image 20250402224848.png]]
## 自连接

从**集合角度**来看，自连接相当于：A X A。即，同一个集合的**笛卡尔积**，然后再通过 `ON` 条件筛选出符合需求的组合。典型自连接为父子关系层级结构。
```sql
CREATE TABLE employees (
    id INT PRIMARY KEY,       -- 员工ID
    name VARCHAR(50),         -- 员工姓名
    manager_id INT            -- 上级经理ID（外键）
);
```
自连接让 `employees` 表自己与自己关联，实现了员工-经理的层级查询。
## 联合查询

从 **集合的角度** 来看，`联合` 操作相当于：A U B 即，**取两个查询结果的并集**。
```sql
SELECT column1, column2 FROM table1
UNION # 去重
SELECT column1, column2 FROM table2;

SELECT name FROM employees
UNION ALL # 保留重复项
SELECT name FROM customers;
```
## 子查询

按照位置划分，子查询可以插入三个位置 **`select` `from` `where`**。由于需要查询多个表，性能相比 `join` 可能较差

子查询可以分为以下几类：

1. **标量子查询（Scalar Subquery）**：返回单个值
    
2. **多行子查询（Multi-Row Subquery）**：返回多行数据 1 列`IN`、`ANY`、`ALL`
    
3. **多列子查询（Multi-Column Subquery）**：返回多列数据 1 行
    
4. **相关子查询（Correlated Subquery）**：子查询依赖外部查询
    
5. **独立子查询（Independent Subquery）**：子查询可独立执行
### 相关子查询

```sql
# 查询工资高于本部门平均工资的员工, 子查询依赖外部定义的 e1 表, 每行执行一
# 次查询
SELECT name, department, salary 
FROM employees e1
WHERE salary > (SELECT AVG(salary) FROM employees e2 WHERE e1.department = e2.department);
```
