---
aliases:
  - 覆盖查询
  - 回表查询
date: 2025-04-06
tags:
  - 索引设计
  - MySQL进阶
---
[[Index-索引分类]]中明确提及 InnoDB 引擎中索引的数据结构为 B+ 树，索引分为**聚集索引、二级索引**。聚集索引叶子节点存储了行信息，二级索引叶子节点存储了主键和索引值。当 DQL 返回列仅仅包含了索引值和主键信息时，仅仅查询二级索引已然足够，否则则会根据二级索引叶子节点存储的主键信息再次查询聚集索引，即触发徽标查询。
```sql
CREATE TABLE `orders` (
  `id` int NOT NULL AUTO_INCREMENT,
  `order_no` varchar(32) NOT NULL,
  `user_id` int NOT NULL,
  `amount` decimal(10,2) NOT NULL,
  `create_time` datetime NOT NULL,
  PRIMARY KEY (`id`),
  -- 创建联合索引
  KEY `idx_user_order` (`user_id`,`order_no`)
) ENGINE=InnoDB;

-- 查询3: 需要回表
EXPLAIN SELECT * FROM orders WHERE user_id = 100;

-- 查询1: 完全覆盖
EXPLAIN SELECT user_id, order_no FROM orders WHERE user_id = 100;
```
