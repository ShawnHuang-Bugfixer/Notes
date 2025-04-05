---
aliases:
  - 索引失效
date: 2025-04-05
tags:
  - MySQL进阶
---
## 1. 索引列运算导致索引失效

​**​场景​**​：对索引列进行运算或函数操作

sql

复制

```sql
-- 假设user表有age索引
-- 失效示例：
SELECT * FROM user WHERE age + 1 > 20;
SELECT * FROM user WHERE YEAR(create_time) = 2023;

-- 优化方案：
SELECT * FROM user WHERE age > 19;
SELECT * FROM user WHERE create_time BETWEEN '2023-01-01' AND '2023-12-31';
```

## 2. 字符串不加引号索引失效

​**​场景​**​：字符串类型字段比较时不加引号


```sql
-- 假设user表有phone字段索引，类型为varchar
-- 失效示例：
SELECT * FROM user WHERE phone = 13800138000;  -- 数字形式

-- 优化方案：
SELECT * FROM user WHERE phone = '13800138000';  -- 字符串形式
```

## 3. 头部模糊匹配失效

​**​场景​**​：使用前导通配符的LIKE查询

sql

复制

```sql
-- 假设user表有name字段索引
-- 失效示例：
SELECT * FROM user WHERE name LIKE '%张%';  -- 头部模糊匹配
SELECT * FROM user WHERE name LIKE '%张';   -- 尾部模糊匹配

-- 部分有效示例：
SELECT * FROM user WHERE name LIKE '张%';   -- 只有尾部%可以使用索引
```

## 4. OR连接索引失效

​**​场景​**​：OR连接的条件中有非索引列

sql

复制

```sql
-- 假设user表有age索引，但address无索引
-- 失效示例：
SELECT * FROM user WHERE age = 20 OR address = '北京';

-- 优化方案1（使用UNION）：
SELECT * FROM user WHERE age = 20
UNION
SELECT * FROM user WHERE address = '北京';

-- 优化方案2（确保OR两边都有索引）：
-- 如果address也有索引，OR可以正常使用
```

## 5. 数据分布导致索引失效

​**​场景​**​：当查询返回大量数据时，优化器可能选择全表扫描

sql

复制

```sql
-- 假设user表有gender字段索引，但90%数据gender=1
-- 可能失效：
SELECT * FROM user WHERE gender = 1;  -- 优化器可能选择全表扫描

-- 强制使用索引（不总是更好）：
SELECT * FROM user FORCE INDEX(idx_gender) WHERE gender = 1;
```

## 6. ORDER BY导致不走索引

​**​场景​**​：ORDER BY与WHERE条件不匹配或排序方向不一致

sql

复制

```sql
-- 假设user表有联合索引(age, name)
-- 失效示例1：
SELECT * FROM user ORDER BY name;  -- 不满足最左前缀

-- 失效示例2：
SELECT * FROM user WHERE age > 20 ORDER BY name DESC, age ASC;  -- 排序方向不一致

-- 有效示例：
SELECT * FROM user WHERE age > 20 ORDER BY age, name;  -- 使用联合索引
```

## 补充其他常见失效场景

### 7. 使用NOT、!=、<>操作符

sql

复制

```sql
-- 假设user表有status索引
-- 失效示例：
SELECT * FROM user WHERE status != 1;

-- 优化方案（视情况而定）：
SELECT * FROM user WHERE status IN (0, 2, 3);
```

### 8. 联合索引不满足最左前缀原则

sql

复制

```sql
-- 假设有联合索引(col1, col2, col3)
-- 失效示例：
SELECT * FROM table WHERE col2 = 'value';  -- 跳过col1
SELECT * FROM table WHERE col1 = 'value' AND col3 = 'value';  -- 跳过col2

-- 有效示例：
SELECT * FROM table WHERE col1 = 'value' AND col2 = 'value';
```

### 9. 使用IS NULL/IS NOT NULL

sql

复制

```sql
-- 假设user表有name索引
-- 可能失效（取决于数据库和版本）：
SELECT * FROM user WHERE name IS NULL;
```

### 10. 类型隐式转换

sql

复制

```sql
-- 假设user表有id索引，类型为varchar
-- 失效示例：
SELECT * FROM user WHERE id = 123;  -- 数字与字符串比较

-- 优化方案：
SELECT * FROM user WHERE id = '123';
```
