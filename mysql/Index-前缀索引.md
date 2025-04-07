---
aliases:
  - 前缀索引
date: 2025-04-06
tags:
  - MySQL进阶
  - 索引设计
---
使用指定列的前 n 个字符构成索引，目的是平衡索引大小和索引效率。使用前缀索引时应当先计算出**选择性**。

**选择性**本质就是某长度的去重前缀占全数据的比例。计算方式:

```sql
SELECT 
    COUNT(DISTINCT LEFT(col_name, 3))/COUNT(*) AS sel3,
    COUNT(DISTINCT LEFT(col_name, 5))/COUNT(*) AS sel5,
    COUNT(DISTINCT LEFT(col_name, 7))/COUNT(*) AS sel7,
    COUNT(DISTINCT LEFT(col_name, 10))/COUNT(*) AS sel10,
    COUNT(DISTINCT col_name)/COUNT(*) AS total_sel
FROM table_name;

-- 创建唯一前缀索引
CREATE UNIQUE INDEX idx_name ON table_name (col_name(N));
```

由于前缀索引存储的索引信息并不完整，有以下缺点：
1. ​**​无法用于排序​**​：前缀索引不能用于ORDER BY或GROUP BY操作
2. ​**​无法覆盖索引​**​：SELECT查询必须回表获取完整值
3. ​**​可能降低区分度​**​：导致查询需要检查更多行