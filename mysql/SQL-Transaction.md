---
aliases:
  - 事务
tags:
  - MySQL基础
date: 2025-04-05
---
# 事务特性（ACID）

事务定义一组 SQL 完成特定任务。事务满足4个核心特性。满足 **原子性、隔离性、持久性**以达到**一致性**。
# 并发事务问题

| ​**​问题​**​                         | ​**​描述​**​                                 | ​**​示例​**​                                   |
| ---------------------------------- | ------------------------------------------ | -------------------------------------------- |
| ​**​脏读（Dirty Read）​**​             | 事务A读取了事务B未提交的数据，若B回滚，A读到的是无效数据。            | 事务B修改某行未提交，事务A读取该行，B回滚后，A得到错误数据。             |
| ​**​不可重复读（Non-Repeatable Read）​**​ | 事务A多次读取同一数据，期间事务B修改并提交，导致A两次结果不一致。         | 事务A第一次读取某行值为100，事务B将其改为200并提交，事务A第二次读取得到200。 |
| ​**​幻读（Phantom Read）​**​           | 事务A按条件查询数据，期间事务B插入或删除符合条件的数据，导致A两次结果的行数不同。 |                                              |
# 事务隔离级别

| ​**​隔离级别​**​                   | ​**​脏读​**​ | ​**​不可重复读​**​ | ​**​幻读​**​ | ​**​描述​**​                                   |
| ------------------------------ | ---------- | ------------- | ---------- | -------------------------------------------- |
| ​**​读未提交（Read Uncommitted）​**​ | 可能         | 可能            | 可能         | 最低隔离级别，事务可读取其他未提交事务的数据。性能高，但存在所有并发问题。        |
| ​**​读已提交（Read Committed）​**​   | 不可能        | 可能            | 可能         | 事务只能读取已提交的数据。多数数据库的默认级别（如Oracle、PostgreSQL）。 |
| ​**​可重复读（Repeatable Read）​**​  | 不可能        | 不可能           | 可能         | 事务内多次读取同一数据结果一致。MySQL的默认隔离级别。                |
| ​**​串行化（Serializable）​**​      | 不可能        | 不可能           | 不可能        | 最高隔离级别，事务串行执行，完全避免并发问题，但性能最低。                |
