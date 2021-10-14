---

title: 一文详解mysql InnoDB存储引擎中的锁
date: 2021-10-13 13:28:19
tags:
  - mysql
  - InnoDB
  - 锁
categories: mysql
---

### 一文详解mysql InnoDB存储引擎中的锁

#### 1 InnoDB 存储引擎锁分类

​		mysql官网上介绍了InnoDB中7种所 **`Shared and Exclusive Locks`**（共享与排他锁）、**`Intention Locks`**（意向锁）、**`Record Locks`**（记录锁）、**`Gap Locks`**（间隙锁）、**`Next-Key Locks`**（临键锁） 、**`Insert Intention Locks`**（插入意向锁）、**`AUTO-INC Locks` **（自增锁），这7种配合MVCC机制，使得mysql在高并发的读写业务场景上有较好的表现。

​		mysql InnoDB 支持不同粒度的锁，如表锁与行锁，同时支持表锁与行锁的共存。为更高效地支持不同粒度的锁，而引入**Intention Locks**（意向锁）。**Shared and Exclusive Locks**（共享与排他锁）主要是为了提高加行锁时读写的并发性。**Record Locks**（记录锁）用于控制带索引记录的操作性能。**Gap Locks**（间隙锁）+ **Next-Key Locks**（临键锁）避免**Repeatable Read**级别下面出现幻读。**Intention Locks**（插入意向锁）保证多个事务，在同一个索引，同一个范围区间插入记录时，如果插入的位置不冲突，不会阻塞彼此。**AUTO-INC Locks** （自增锁）保证同一事务中利用数据库自增ID插入数据时ID的连续性。

#### 2 Shared and Exclusive Locks（共享与排他锁）

​		InnoDB内实现了标准的行锁，其中包括两类锁**Shared Locks (共享锁 S)** 与 **Exclusive Locks （排他锁 X）**。

- **Shared Locks (共享锁 S) 允许获取锁的事务读取数据。**
- **Exclusive Locks （排他锁 X）允许获取锁的事务修改、删除数据。**

##### 2.1 共享锁 S 与 排他锁 X兼容互斥性

 		如果事务A获取数据行Data上的共享锁 S，其他事务也可以获取数据行Data上的共享锁 S。如果事务A获取数据行Data上的排他锁 X，则其他事务不能获取数据行Data上的排他锁 X，同样也不能获取数据行Data上的共享锁 S，必须待到事务A**释放**数据行Data上的排他锁 X。共享锁 S与排他锁 X之间的兼容互斥性如下图。

<img src="https://gitee.com/0909/blog/raw/master/img/SX锁兼容性.png" style="zoom:80%;" />

##### 2.2 共享锁 S 与 排他锁 X兼容互斥性示例

​		下面看一下在user表上，共享锁 S 与 排他锁 X兼容互斥性示例。user表中user_id 为**primary key**。

**一、Session1加共享锁S**

​		在Session1中 执行 select * from user where user_id = 1 lock in share mode 语句，此时user_id = 1 的行上加了**共享锁 S**。由于user_id的行上没有其他锁，查询结果如下。

<img src="https://gitee.com/0909/blog/raw/master/img/20211014100110.png" style="zoom:80%;" />

**二、Session2再加共享锁s（可再次读取数据）**

​		在Session2中 执行 select * from user where user_id = 1 lock in share mode 语句，此时上面的Session1已加共享锁，由于**共享锁S之间兼容**可再次在user_id = 1 的行上加**共享锁 S**。Session2查询结果如下。

<img src="https://gitee.com/0909/blog/raw/master/img/20211014100017.png" style="zoom:80%;" />

**三、Session3再加排他锁X（出现锁等待超时）**

​		在Session3中 执行 select * from user where user_id = 1 for upate 语句，此时user_id = 1 的行已被上次两个Session加**共享锁 S**。由于**共享锁S与之排他锁X之间互斥**，Session3要等待user_id = 1 的行上**共享锁 S**  释放才能获取结果，否则出现**锁等待超时**，查询结果如下。

<img src="https://gitee.com/0909/blog/raw/master/img/20211014100211.png" style="zoom:80%;" />

​		共享锁 S 与 排他锁 X兼容互斥性其他情况不一一分析了。

#### 3 Intention Locks（意向锁）

​		意向锁，表示未来的某个时刻，事务可能要加共享/排它锁了，先提前声明一个意向。InnoDB 支持**`多粒度锁（multiple granularity locking）`**，他允许**`行级锁`**与**`表级锁`**共存，为更高效地支持不同粒度的锁，而引入**Intention Locks（意向锁）**，而实际是一种**`表锁`**。Intention Locks（意向锁）分为**意向共享锁（intention shared lock, IS）** 与**意向排他锁（intention exclusive lock, IX）**两种。

- **意向共享锁**（intention shared lock, IS）：事务有意向对表中的某些行加**共享锁**（S锁）

  ```sql
  -- 事务要获取某些行的 S 锁，必须先获得表的 IS 锁。
  SELECT column FROM table ... LOCK IN SHARE MODE;
  ```

- **意向排他锁**（intention exclusive lock, IX）：事务有意向对表中的某些行加**排他锁**（X锁）

  ```sql
  -- 事务要获取某些行的 X 锁，必须先获得表的 IX 锁。
  SELECT column FROM table ... FOR UPDATE;
  ```

意向锁是一种比较弱的锁，他们之间相互兼容

<img src="https://gitee.com/0909/blog/raw/master/img/IS与IX兼容性.png" style="zoom:80%;" />

**注意：** 意向锁是一种与*行锁不冲突* 的表锁，意向锁是用来提高表级加锁判断效率的，看完下面**意向锁解决的问题**便会明白。重要的事重复三遍

**意向锁不会与行级的共享 / 排他锁互斥**

**意向锁不会与行级的共享 / 排他锁互斥**

**意向锁不会与行级的共享 / 排他锁互斥**

##### 3.1 意向锁解决的问题

​		来看看如果没有意向锁时，一个事务想向表加表锁的整体处理流程。假设事务想向表加共享锁比如执行如下操作，

```sql
LOCK TABLES user READ;
```

由于共享锁S与排他锁X互斥，所以当前事务试图为表加共享锁时，必须保证：

- 当前没有其他事务持有user表的排他锁。
- 当前没有其他事务持有user表中任意一行的排他锁。

所以当前事务在试图对表加共享锁前，会去检测上面的两个条件是否满足，即要检测user表每一行是否加了排他锁，没有意向锁时只能一行一行的检测，这种作法必然效率低下。**意向锁是如何解决这种检测效率低下的问题的呢？**

​		上面已说到过事务要获取某些行的 S 锁，必须先获得表的 IS 锁，事务要获取某些行的 X 锁，必须先获得表的 IX 锁。同样事务想对表加共享锁，必须先获得表的 IS 锁，事务想对表加排他锁，必须先获得表的 IX 锁。这样事务试图对表加共享锁时，只要检测表上有没有已加 IX 锁就行，而不用一行行去检测表中所有记录存没存在加行排他锁（只要表中某一行要加行排他锁，必然要先加IX 锁），如果表上已加IX锁，事务试图对表加共享锁的事务将被阻塞。

##### 3.2 S锁 X锁 IS锁 IX锁之间的兼容互斥性

​		前面已了解S锁与X锁，IS锁与 IX锁之间的兼容互斥性，现在看一下S锁 X锁 IS锁 IX锁之间的兼容互斥性。意向锁之间互相兼容。

<img src="https://gitee.com/0909/blog/raw/master/img/锁兼容性.png" style="zoom:80%;" />

注意：上面所说的S锁与X锁是针对于表锁，而不是行锁，意向锁只针对表，不针对行进行锁定，意向锁是用来提高表级加锁判断效率的，其不会影响多个事务并发对不同的行数据加行排他锁。重要的事再重复三遍

**意向锁不会与行级的共享 / 排他锁互斥**

**意向锁不会与行级的共享 / 排他锁互斥**

**意向锁不会与行级的共享 / 排他锁互斥**

##### 3.3 示例分析

**一、事务 A 先获取了`user`表 (user_id为主键)某一行的排他锁（并未提交）**

```sql
select * from user where user_id = 1 for update;
```

1、事务A会先获取了 `user` 表上的**意向排他锁（IX）**。

2、事务A会先获取了 `user` 表中，user_id=1对应行上的**排他锁(X)**。

**二、事务B尝试获取user表上的表共享锁（S）**

```sql
lock tables use read;
```

1、事务B检测事务A持有use表上的**意向排他锁（IX）**。

2、事务B对user表加锁请求处理阻塞等待

**三、事务C想获取user表中另一行的排他锁**

```sql
select * from user where user_id = 7 for update;
```

1、事务C申请获取user表上的**意向排他锁（IX）**。

2、事务C检测到事务A已持有user表的**意向排他锁（IX）**。

3、因为意向排他锁（IX）之间不互斥，事务C成功获取user表上的**意向排他锁（IX）**。

4、因为user_id = 7对应行记录上没有排他锁，事务C成功user表上user_id=7的行上对应的**行排它锁**。

#### 4 Record Locks（记录锁)

​		Record Locks（记录锁）他锁定索引对应的数据行记录。

```sql
select * from user where user_id = 7 for update;
```

这个词句有保证user表上user_id = 7 的记录被锁定，在当前事务没提交前，其他事务对该记录的当前读操作会被阻塞（当前读具体介绍可见我之的文章[一文了解mysql InnDB中MVCC的今世前生](https://mp.weixin.qq.com/s/026g7KHA4ZbjsoIUf9fXNg)）。

#### 5 Gap Locks（间隙锁）

​		Gap Locks（间隙锁）用于锁定某一区间范围内的记录。它封锁索引记录中的间隔，或者第一条索引记录之前的范围，又或者最后一条索引记录之后的范围。其主要用于**防止其他事务在索引记录中的间隔内部插入数据，而引起的幻读**。对于**读已提交**(Read Committed, RC) 事务隔离级别，Gap Locks会自动失效。

##### 5.1 数据锁定范围分析

假设user表中**user_id**为**主键(Primary Key)**且为**unsigned int** 类型。

```sql
select * from user where user_id < 7 for update;
```

​		上面的操作锁定的记录范围为 user_id >= 0 到 user_id = 6 的记录，只有事务提交后，其他事务才能操作该范围的记录。注意即使 user_id=3对应的记录不存在，事务没提交前，插入user_id=3记录也是不允许的，不然就没办法解决幻读问题。

```sql
select * from user where user_id > 7 and user_id < 1000 for update; 
```

​		上面的操作锁定的记录范围为 user_id > 7 到 user_id < 10000 的记录，只有事务提交后，其他事务才能操作该范围的记录。

```sql
select * from user where user_id > 1000 for update; 
```

​		上面的操作锁定的记录范围为 user_id > 1000 到 int 最大值之间的记录，只有事务提交后，其他事务才能操作该范围的记录。

#### 6 Next-Key Locks（临键锁）

​		**Next-Key Locks（临键锁）**是 **Recod Locks（记录锁）**与**Gap Locks（间隙锁）**的组合，它的封锁范围，既包含索引记录，又包含索引区间。同样其主要用于**防止其他事务在索引记录中的间隔内部插入数据，而引起的幻读**。

​		假设user表中**user_id**为**主键(Primary Key)**且为**unsigned int** 类型，表中已有use_id 为 3，5，6，9，1000的记录。则对user_id而言可能的Next-Key Locks（临键锁）范围有

- [0，3]
- （3，5]
- （5，9]
- （9，1000]
- （1000，int最大值]

#### 7 Insert Intention Locks（插入意向锁）

​		**Insert Intention Locks（插入意向锁）**是间隙锁(Gap Locks)的一种，其专门针对插入操作，其能提高多个事务并发向锁区间的范围内插入不同数据的并发效率，**插入意向锁提高了并发插入的效率**。

​		假设user表中**user_id**为**主键(Primary Key)**且为**unsigned int** 类型且非自增，表中已有use_id 为 7，99，1000的记录。

​		事务A先执行，在user_id=7与user_id=99两条记录中插入了一行包含user_id = 9的记录，还未提交

```sql
insert into user values(9, xxx,...); 
```

​		事务B后执行，也在user_id=7与user_id=99两条记录中插入了一行包含user_id = 97的记录，提交。

```sql
insert into user values(97, xxx,...); 
```

此事务A与事务B分别会加user_id=9与user_id=97的	**Insert Intention Locks（插入意向锁）**， 事务B不会被事务B给阻塞。

#### 8 AUTO-INC Locks（自增锁）

​		**AUTO-INC Locks（自增锁）**是一种特殊的表级别锁（table-level lock），专门针对事务插入AUTO_INCREMENT类型的列。最简单的情况，如果一个事务正在往表中插入记录，所有其他事务的插入必须等待，以便第一个事务插入的行，是连续的主键值。InnoDB 提供了 [`innodb_autoinc_lock_mode`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_autoinc_lock_mode) 参数用于控制AUTO-INC Locks（自增锁）。其允许选择如何在可预测的自动增量值序列和插入操作的最大并发之间进行权衡。

#### 总结

​		本文介绍了mysql InnoDB存储引擎中常见的锁，并通过部分示例对锁进行简要地分析。mysql InnoDB 支持不同粒度的锁，如表锁与行锁，同时支持表锁与行锁的共存。为更高效地支持不同粒度的锁，而引入**Intention Locks**（意向锁）。**Shared and Exclusive Locks**（共享与排他锁）主要是为了提高加行锁时读写的并发性。**Record Locks**（记录锁）用于控制带索引记录的操作性能。**Gap Locks**（间隙锁）+ **Next-Key Locks**（临键锁）避免**Repeatable Read**级别下面出现幻读。**Intention Locks**（插入意向锁）保证多个事务，在同一个索引，同一个范围区间插入记录时，如果插入的位置不冲突，不会阻塞彼此。**AUTO-INC Locks** （自增锁）保证同一事务中利用数据库自增ID插入数据时ID的连续性。

#### 参考文档

[InnoDB Locking](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-shared-exclusive-locks)

[这次终于懂了，InnoDB的七种锁（收藏）](https://mp.weixin.qq.com/s/f4_o-6-JEbIzPCH09mxHJg)

#### 题外话

​		原创不易，点赞、在看、转发是对我莫大的鼓励，关注公众号洞悉源码是对我最大支持。同时相信我会分享更多干货，我同你一起成长，我同你一起进步。

