---
aliases:
  - 数据定义语言
date: 2025-04-02
tags:
  - MySQL基础
---
定义数据库对象（表格、视图、索引等）。

# 定义表格

定义表格 **[[SQL-DML-字段数据类型]]、存储引擎、字符集、数据类型、索引、默认值、[[SQL-DML-约束]]、自增等**。
```sql
CREATE TABLE users (
    user_id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY COMMENT '用户ID',
    username     VARCHAR(50) NOT NULL UNIQUE COMMENT '用户名，唯一',
    email        VARCHAR(100) NOT NULL UNIQUE COMMENT '邮箱，唯一',
    password     VARCHAR(255) NOT NULL COMMENT '用户密码，经过加密存储',
    age          TINYINT UNSIGNED CHECK (age >= 18) COMMENT '用户年龄，必须18岁及以上',
    gender       ENUM('male', 'female', 'other') DEFAULT 'other' COMMENT '性别，可选值: male, female, other',
    status       TINYINT(1) NOT NULL DEFAULT 1 COMMENT '用户状态，1-正常，0-禁用',
    created_at   DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '账户创建时间',
    updated_at   DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '账户更新时间',
    INDEX idx_email (email),  -- 创建邮箱索引，提高查询效率
    CONSTRAINT chk_status CHECK (status IN (0, 1))  -- 状态只能是 0 或 1
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户信息表';
```
