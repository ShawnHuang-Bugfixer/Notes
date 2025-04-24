---
aliases:
  - MVCC
tags:
  - UndoLog
---
`MVCC` 主要在 **READ COMMITTED** 和 **REPEATABLE READ** 隔离级别下生效，提供了更高的并发性能和一致性保障。使用 `MVCC`（Multi-Version Concurrency Control，多版本并发控制）是为了提高数据库的并发性能。`MVCC` 能够实现即使有**读写冲突**时，也能做到**不加锁，非阻塞并发读**。

# `MVCC` 实现原理

`MVCC` 主要依赖 `Undo Log` 和 `Read View` 实现。

## Undo Log 版本日志

`InnoDB` 引擎中，每条记录都包含了 `DB_TRX_ID,DB_ROLL_PTR,DB_ROW_ID` 等隐藏字段。

- **DB_ROW_ID** 6byte, 隐含的自增ID（隐藏主键），如果数据表没有主键，InnoDB会自动以DB_ROW_ID产生一个聚簇索引
- **DB_TRX_ID** 6byte, 最近修改(修改/插入)事务ID：记录创建这条记录/最后一次修改该记录的事务ID
- **DB_ROLL_PTR** 7byte, 回滚指针，指向这条记录的上一个版本（存储于rollback segment里）

Undo Log 利用 `事务ID` 和 `回滚指针` 记录 `DML` （INSERT、DELETE，UPDATE）操作并生成 Undo Log 版本链。

> 事务 ID 严格自增。同一版本链中，事务 ID 相同，说明同一事务多次对该记录执行了 `DML` 操作。

![[attachments/Pasted image 20250424095445.png]]

## Read View 一致性视图

>Read View 仅在使用 `快照读（不加锁）` 时才会产生，`当前读（加锁的 DQL 或者 DML）` 读取版本最新数据（索引项），不使用 Read View。

> 只读事务的事务ID默认为0，只有当事务中执行了 `DML` 操作时，才会为该事务分配事务ID。Read View 是静态的，被创建后不再改变。某事务中先执行快照读（此时产生了 Read View，Read View 中的事务ID 为 0）再执行 `DML`，Read View 事务ID 不再改变，仍然为 0。

**Read View 是静态的，一但产生，无法修改**。**范围查询和精确查询产生的 Read View 结构一致**。Read View 利用事务 ID 划定了三个范围：已提交事务，活跃事务，将来事务。

| 字段               | 含义                                          |
| ---------------- | ------------------------------------------- |
| `trx_ids`        | 活跃事务的 ID 集合（创建 Read View 时尚未提交的事务）          |
| `min_trx_id`     | 活跃事务中最小的事务 ID，用于优化判断                        |
| `max_trx_id`     | 创建 Read View 时，系统分配给下一个事务的 ID（不是当前事务 ID）    |
| `creator_trx_id` | 当前创建该 Read View 的事务 ID（如果当前事务还未分配 ID，可能为 0） |
- 活跃事务：`事务ID ∈ trx_ids` 开启事务但尚未提交事务。
- 将来事务：`事务ID` >= `max_trx_id`  `MySQL` 将来开启的事务。
- 已提交事务：`事务ID` < `min_trx_id` ， `事务ID` > `min_trx_id` 且 `事务ID` < `max_trx_id` 且 `事务ID` 不属于 `trx_ids`。

例如：

![[attachments/Pasted image 20250424103440.png]]
事务 id = 84，不在将来事务中，不在活跃事务中，是已经提交的事务。

### 版本是否可见的判断标准

已提交事务可见，版本事务ID == 当前 Read View视图中记录的事务ID可见，活跃事务不可见，将来事务不可见。

```Text
1. 如果 version_trx_id < min_trx_id，则该版本在所有活跃事务之前生成，✅ 可见。

2. 如果 version_trx_id >= max_trx_id，则该版本是在当前事务之后创建的，❌ 不可见。

3. 如果 version_trx_id == creator_trx_id（创建 ReadView 的当前事务 ID），✅ 可见。

4. 如果 version_trx_id 在 m_ids 中（仍是活跃事务之一），❌ 不可见。

5. 否则，✅ 可见。
```

### 版本链遍历

快照读时，InnoDB 从记录本身出发，结合 Read View 判断当前版本是否可见；  
如果不可见，就**沿着记录中的回滚指针（`ROLL_PTR`）向后回溯版本链**，返回可见版本。

### Read View 在 RR 和 RC 下生成时机

|隔离级别|Read View 创建时机|同一个事务多次快照读是否共享同一个 Read View？|
|---|---|---|
|**READ COMMITTED**|每次快照读时都重新创建|❌ 否，视图实时刷新|
|**REPEATABLE READ**|第一次快照读时创建一次|✅ 是，后续查询都使用同一个视图|

# RC 隔离级别下利用锁机制解决幻读和不可重复读。

事务每次快照读都会产生新的 Read View，这导致该事务读取到数据可能不同（新的 Read View 可能包含新提交的事务），必须利用锁机制解决。
# RR 隔离级别下仅仅利用 `MVCC` 解决幻读

事务每次快照读都基于第一次快照读所产生的的 Read View（静态），只要该事务始终使用快照读，便不会出现不可重复读和幻读。


