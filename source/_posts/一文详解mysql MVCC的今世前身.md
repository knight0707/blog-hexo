---
title: 一文详解mysql MVCC的今世前身
date: 2021-10-12 13:28:19
tags:
  - mysql
  - MVCC
categories: mysql
---

### 一文详解mysql MVCC的今世前身

#### 1 为什么需求MVCC

​		MVCC全称为[Multi-Version Concurrency Control](https://en.wikipedia.org/wiki/Multiversion_concurrency_control)（多版本并发控制），是一种数据库并发访问无锁的优化方案。其无锁优化是针对于读取时不需要加锁，其基本思路是数据库的任何修改都不会直接覆盖原来的数据，而是生成新的副本，并通过链表将新老版本副本串连，每个副本中都会包含修改数据记录的版本号，读取数据时根据版本号的对比确定链表上哪个版本的数据对当前事务可见。在该技术出现前，只有读-读之间可以并发，引入多版本控制之后，读-读、读-写、写-读都可以并发，实现读无需加锁、读写不冲突，极大提高了数据库系统的并发性。

​		在mysql InnoDB中，只有**READ COMMITE** 和 **REPEATABLE READ** 二个隔离级别下面，MVCC才会工作。READ UNCOMMITED 总读取最新版本的数据不需求MVCC进行多版本并发的控制，而SERIALIZABLE会对所有读取操作加锁，同样不需求MVCC。mysql InnoDB 存储引擎中 MVCC 机制简单的来说是通过**隐式列 + undo log + read view** 实现的。

#### 2 隐式列、undo log 、 read view、快照读和当前读

##### 2.1 隐式列

​		InnoDB 会为第一行数据加三列，分别是**DB_TRX_ID、DB_ROLL_PTR、DB_ROW_ID**（ [InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/5.7/en/innodb-multi-versioning.html)）。

- **DB_TRX_ID 代表最近更新或新增该行数据事务的事务标识，也可叫事务ID(删除操作也被看作一种特殊的更新)**

- **DB_ROLL_PTR 表示回滚指针，其指向当前记录上一次更新或新增操作对应的undo log，undo log 能还原数据到之前的状态**

- **DB_ROW_ID 在无聚簇索引时 innodb内部自动创建的自增id  **

  

  <img src="https://gitee.com/0909/blog/raw/master/img/隐式列.png" style="zoom:80%;" />

##### 2.2 undo log

​		undo log 主要有两大用途一是实现InnoDB中事务的原子性，保障已部分落盘的数据可以被回滚，另外其用于实现MVCC机制中多版本数据的变向存储。undo log 是逻辑日志，其记录了上一次操作的逆向操作。

<img src="https://gitee.com/0909/blog/raw/master/img/20211012145919.png" style="zoom:80%;" />

##### 2.3 read view

​		[read veiw](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_read_view) 是InnoDB在进行快照读时产生时产生的**读视图**（后面介绍快照读，也叫一致性读 **consistent read** ）。当执行快照读时，系统会为其创建一个read view,  并通过最新数据与undo log链上数据中的DB_TRX_ID 与read view 进行比较，来决定**当前事务**（**创建read view对应的事务**）能够看到undo log上哪个版本的数据。

##### 2.4 快照读和当前读

​		**快照读**

​		简单的select 语句（不包含 select ... lock in share model, select ... for update）都快照读。mysql官网的叫[consistent read](https://dev.mysql.com/doc/refman/5.7/en/innodb-consistent-read.html)，因为一致性读是基于快照的，通常会叫快照读，这类读不用加读锁，而是通过**隐式列 +  undo log + read view **来获取所要的数据。快照读实际就是InnoDB中MVCC机制的实现。

​		**当前读**

​		所有的需要加锁的涉及读取的操作（可以参考mysql官网中的[innodb locking reads](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking-reads.html) ）如

- select ... lock in share model
- select ... for update
- update ...
- insert ...
- delect ...

#### 3 Innodb MVCC 快照读处理流程

​		通过前面简单的介绍已了解到快照读实际就是InnoDB中MVCC机制的实现。现在来看一下MVCC快照读的整体处理流程。快照读时数据可见的判断依赖于ReadView 这个数据结构，其包含关键属性如下。

- **`m_ids`** 当前快照被创建时，活跃（已启动但未提交）的事务集合。
- **`m_low_limit_id`** 高水位，当前系统中的事务的最大ID值加一，即下一个开启的事务的ID。
- **`m_up_limit_id`**  低水位，当前活跃事务的最小ID，小于该ID的事务都是已经提交的事务。
- **`m_creator_trx_id`** 创建该read view的事务ID。



<img src="https://gitee.com/0909/blog/raw/master/img/事务ReadView1.png" style="zoom:80%;" />

​		InnoDB在处理快照读（consistent read 一致性读时），只要拿出数据中的DB_TRX_ID（隐式列中最近更新或新增该行数据事务的事务标识，事务ID）然后与ReadView中的m_ids、m_low_limit_id、m_up_limit_id、m_creator_trx_id进行比较，就可确定当前版本的数据对ReadView读视图是否可见。处理算法大体流程如下：

**1、DB_TRX_ID ≥ m_low_limit_id （高水位）， DB_TRX_ID 对应数据对当前事务不可见，表示这个数据是由将来的事务更新的。**

**2、DB_TRX_ID < m_up_limit_id （低水位） ，DB_TRX_ID 对应数据对当前事务可见，表示这个数据是由已经提交的事务更新的。**

**3、DB_TRX_ID 介于高水位与低水位之间，如果DB_TRX_ID在m_ids（活跃事务集合）中，表示尚未提交的事务， DB_TRX_ID 对应数据对当前事务不可见;  如果DB_TRX_ID不在m_ids（活跃事务集合）中，表示事务已经提交了， DB_TRX_ID 对应数据对当前事务可见。**

#### 参考文档

[The InnoDB Storage Engine](https://dev.mysql.com/doc/refman/5.7/en/innodb-storage-engine.html)

[mysql5.7 ReadView 源码]( https://github.com/mysql/mysql-server/blob/5.7/storage/innobase/include/read0types.h)

