---
date: 2025-04-14
---
# 对象头 Object Header

Java 对象头（Object Header）是 JVM 用于存储对象元数据的结构，对象头通常由 `Mark Word​`  和`​​Klass Pointer​`。它是实现 `synchronized` 锁机制、GC 分代年龄、哈希码等功能的基础。

**对象头格式（32 位虚拟机）**：

- **普通对象**
	![[attachments/Pasted image 20250414161654.png]]
- **数组对象**
	![[attachments/Pasted image 20250414161809.png]]

**Mark Word** 格式（32 位虚拟机）

![[attachments/Pasted image 20250414162231.png]]

​**​🔹 关键字段说明​**​：

- ​**​hashcode​**​：对象的哈希码（调用 `System.identityHashCode()` 时生成）。
- ​**​age​**​：GC 分代年龄（用于分代垃圾回收）。
- ​**​biased_lock​**​：是否启用偏向锁（`1` 启用，`0` 禁用）。
- ​**​threadID​**​：偏向锁记录的线程 ID。
- ​**​epoch​**​：偏向锁的版本号（用于撤销偏向锁）。
- ​**​Lock Record​**​：轻量级锁时，存储指向线程栈帧的指针。
- ​**​Monitor​**​：重量级锁时，指向 `Monitor`（管程）的指针。

# Monitor 管程

`sychornized` **重量级锁**的底层原理，JVM 进行管理。管程的主要功能是 ​**​实现线程互斥和协作**。管程的关键组件包含 `Owner` `EntryList` `WaitList`。

**管程创建时机**：
当发生以下情况时，JVM 会为对象 ​**​隐式创建并关联一个管程​**​：

- ​**​升级为重量级锁​**​：当 `synchronized` 竞争激烈（轻量级锁自旋失败），锁升级为重量级锁，Mark Word 指向一个 `ObjectMonitor` 实例。
- ​**​调用 `wait()/notify()`​**​：方法依赖管程数据结构，隐含锁升级条件。

**管程的主要结构**如下：

![[attachments/Pasted image 20250414164121.png]]

- Owner：**​JVM 层的线程指针（C++ 结构）​**，表示该线程成功获取锁。
- EntryList：线程竞争锁失败后进入该集合，此时该线程进入 Blocked 状态。
- WatiSet：线程拥有者调用 wait() 等方法**先**后进入 Waiting 状态，**后**释放锁。被唤醒后，进入 EntryList 重新竞争锁。