---
aliases:
  - SQL优化
date: 2025-04-06
tags:
  - MySQL进阶
---
# 插入数据优化

## insert 语句优化

插入多条数据时可以采用**批量插入，手动开启事务，主键顺序插入**的方式优化。InnoDB 引擎使用聚集索引存储行信息。聚集索引数据结构为 b+ 树，叶子节点存储了行信息，且为有序的双向链表。如果主键乱序插入可能导致**页分裂**。

## load 加载数据

![[attachments/Pasted image 20250406195215.png]]

# 主键优化

InnoDB 使用**页**存储 b+ 树的节点和叶子节点，并且聚集索引的叶子节点**顺序存储**了完整的 row 信息。如果主键乱序插入表中，可能导致**页分裂**。

基于聚集索引和二级索引的结构，主键设计时因该满足以下要求：
- 满足业务的最小主键长度（二级索引存储了主键）
- 顺序主键插入，尽量避免随机主键。
- 避免修改主键

## 页分裂

1. 主键顺序插入
	![[attachments/Pasted image 20250406200413.png]]
2. 主键乱序插入
	![[attachments/Pasted image 20250406200619.png]]
	![[attachments/Pasted image 20250406200645.png]]
	![[attachments/Pasted image 20250406200702.png]]

## 页合并

删除一行记录时，将页中对应的 row 标记为删除，此时该记录可以被覆盖。当页中删除的记录达到设定的阈值时，出发页合并，检查前后页并判断是否可以合并。
1. ![[attachments/Pasted image 20250406200834.png]]
2. ![[attachments/Pasted image 20250406200900.png]]
3. ![[attachments/Pasted image 20250406201017.png]]
# order by 优化

order by 有两种实现方式**Using filesort, Using index**。如果排序的字段在表中有对应索引（字段相同，顺序相同）会走索引并直接返回顺序，效率较高。否则会通过 sort buffer 缓冲区排序后返回，效率较低。order by 索引优化需要满足的要求：

## **​1. 排序字段相同（完全匹配或最左前缀）​**​

- ​**​完全匹配​**​：ORDER BY 子句中的所有列必须​**​完全匹配​**​索引中的列（包括顺序）
    
    ```sql
    -- 索引: (a, b, c)
    -- 能使用索引:
    ORDER BY a
    ORDER BY a, b
    ORDER BY a, b, c
    
    -- 不能使用索引:
    ORDER BY b       -- 不满足最左前缀
    ORDER BY a, c    -- 跳过了 b
    ```
    
- ​**​最左前缀原则​**​：可以只使用索引的​**​最左连续部分​**​，但不能跳过中间列    

## ​**​2. 排序方向一致​**​

- ​**​所有列的排序方向（ASC/DESC）必须与索引定义一致​**​  
    （MySQL 8.0+ 支持混合方向索引，但早期版本要求完全一致）
    ```sql
    -- 索引: (a ASC, b DESC)
    -- 能使用索引:
    ORDER BY a ASC, b DESC
    
    -- 不能使用索引:
    ORDER BY a DESC, b ASC  -- 方向不匹配
    ```

## ​**​3. 索引顺序和 ORDER BY 顺序一致​**​

- ​**​列顺序必须严格匹配​**​，不能颠倒：
   ```sql
    -- 索引: (a, b)
    -- 能使用索引:
    ORDER BY a, b
    
    -- 不能使用索引:
    ORDER BY b, a    -- 顺序颠倒
    ```

# group by 优化

分组操作时也可以使用索引进行优化，要求满足最左前缀法则。

# limit 优化

大数据量下，通过**覆盖索引**拿到分页数据作为临时表，然后使用**子查询**内连接返回结果性能更高。
```sql
-- 直接检索
EXPLAIN SELECT * FROM large_table ORDER BY id LIMIT 100000, 20;

-- 覆盖索引 内连接
SELECT t.* FROM large_table t
JOIN (
    SELECT id FROM large_table
    ORDER BY create_time DESC
    LIMIT 100000, 20
) AS tmp ON t.id = tmp.id;
```

# count 优化

![[attachments/Pasted image 20250406211227.png]]
