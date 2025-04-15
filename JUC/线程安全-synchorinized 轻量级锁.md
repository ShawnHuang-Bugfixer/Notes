---
date: 2025-04-15
---
# 设计目的

同步代码块被多个**线程交替执行**，此时**无并发冲突**。依靠**重量级锁​**​（通过操作系统互斥量）会导致操作系统频繁地执行​**​用户态-内核态切换​**​、​**​线程阻塞/唤醒​**​等操作，导致显著开销。

轻量级锁实现依赖**线程虚拟机栈栈帧中的锁记录**和对象头中的`markword`。锁记录保存了**锁记录地址轻量级锁标记 `00`** 和**锁对象引用**。

# 加锁流程

![[attachments/Pasted image 20250415160515.png]]

![[attachments/Pasted image 20250415161125.png]]

![[attachments/Pasted image 20250415161136.png]]

![[attachments/Pasted image 20250415161320.png]]

**轻量级锁加锁流程**:

```mermaid
flowchart TD
    A[Mark Word 为 无锁状态 或 00] --> B[创建 Lock Record 在线程栈]
    B --> C[CAS 尝试将 Mark Word 指向 Lock Record]
    C --> D{CAS 是否成功？}
    D -- 成功 --> E[获得轻量级锁]
    D -- 失败 --> F{Mark Word 是否指向当前线程的 Lock Record？}
    F -- 是 --> G[锁重入，创建新 Lock Record]
    F -- 否 --> H[存在竞争，自旋等待]
    H --> I{自旋成功？}
    I -- 成功 --> E
    I -- 失败 --> J[锁膨胀]
```

# 锁膨胀

`锁记录`和对象头 `markword` 执行 `CAS` 操作失败。并且对象头中的 `record lock` 不属于当前栈。当前线程创建 Monitor 管程，更新对象头 mark word，进入管程EntryList，最后变为阻塞状态。

```mermaid
flowchart TD
    A[自旋失败或冲突激烈] --> B[创建 Monitor，更新 Mark Word 为 10]
    B --> C[线程加入 EntryList，变为 Blocked]
    D[持锁线程解锁时] --> E[CAS 替换 Mark Word 失败]
    E --> F[进入重量级解锁流程]
    F --> G[清空 Owner]
    G --> H[唤醒 EntryList 中线程竞争锁]
```
