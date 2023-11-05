---
title: MySQL幻读和可重复读
auther: gzj
tags:
  - MySQL
categories: 数据库
description: >-
  当同一查询在不同时间产生不同的行集时，在事务内就会发生所谓的幻象问题。
  例如，如果SELECT执行两次，但是第二次返回的行却不是第一次返回，则该行是“幻像”行，还有select没有的数据，insert时失
abbrlink: 9617db49
---

### 幻读和可重复读

1. 由于很多人容易搞混 `不可重复读` 和 `幻读`，这两者确实非常相似。
   - 但 `不可重复读` 主要是说多次读取一条记录, 发现该记录中某些列值被修改过。
   - 而 `幻读` 主要是说多次读取一个范围内的记录(包括直接查询所有记录结果或者做聚合统计), 发现结果不一致(标准档案一般指记录增多, 记录的减少应该也算是幻读)。(可以参考[MySQL官方文档对 Phantom Rows 的介绍](https://dev.mysql.com/doc/refman/5.7/en/innodb-next-key-locking.html))
2. 其实对于 `幻读`, MySQL的InnoDB引擎默认的`RR`级别已经通过`MVCC自动帮我们解决了`, 所以该级别下, 你也模拟不出幻读的场景; 退回到 `RC` 隔离级别的话, 你又容易把`幻读`和`不可重复读`搞混淆, 所以这可能就是比较头痛的点吧!
   具体可以参考《高性能MySQL》对 `RR` 隔离级别的描述, 理论上RR级别是无法解决幻读的问题, 但是由于InnoDB引擎的RR级别还使用了MVCC, 所以也就避免了幻读的出现!

MVCC虽然解决了`幻读`问题, 但严格来说只是解决了**部分**幻读问题。

比如可能会发生这种情况：

事务A：

![image-20210412220546067](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/image-20210412220546067.png)

另外一个直接执行：

![image-20210412220604432](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/image-20210412220604432.png)

就会发生图片上的情况。

关于第5步可以读出最新数据而且第6步无法插入，是因为select分为当前读和快照读，普通select不加for update是快照读，insert、delete、update都是当前读。

### 关于快照读和当前读

- 快照读, 读取专门的快照
```
简单的select操作即可(不需要加锁,如: select ... lock in share mode, select ... for update)
```
针对的也是select操作

- 当前读, 读取最新版本的记录
```
select ... lock in share mode
select ... for update
```
针对如下操作, 会让如下操作阻塞:    
```
insert
update
delete
```
- 在RR级别下, 快照读是通过MVVC(多版本控制)和undo log来实现的, 当前读是通过手动加record lock(记录锁)和gap lock(间隙锁)来实现的。所以从上面的显示来看，如果需要实时显示数据，还是需要通过加锁来实现。这个时候会使用next-key技术来实现。

### SERIALIZABLE隔离级别如何解决幻读

RR隔离级别可以使用对记录手动加 X锁 的方法消除幻读，比如select ... for update等可以加锁，存在则会被加行（X）锁，如果不存在，则会加 next-lock key / gap 锁（范围行锁），即记录存在与否，mysql 都会对记录应该对应的索引加锁，其他事务是无法再获得做操作的。

另外，使用SERIALIZABLE 隔离级别也可以完全解决幻读，它是对所有事务都加 X锁，不过我们大部分事务都没必要，所以造成了性能浪费

在此级别下，我们便不需要对 SELECT 操作显式加锁，InnoDB会自动加锁，事务安全，但性能很低

### 关于MySQL隔离级别为什么是RR

众所周知，常见的关系型数据库的默认事务隔离级别采用的是READ_COMMITED，例如PostgreSQL、ORACLE、SQL Server和DB2。但是使用InnoDB引擎的MySQL数据库默认事务隔离级别是REPEATABLE_READ。

那为什么MySQL要独树一帜的选用RR的隔离级别呢？

其实这是一个历史遗留问题。

我们都知道，MySQL的主从复制是基于binlog复制的，在MySQL 5.7.7之前，默认的格式是 `STATEMENT`，在 MySQL 5.7.7 及更高版本中，默认值是 `ROW`。日志格式通过 `binlog-format` 指定。

binlog目前有三种格式：statement、row、mixed：

- statement:记录的是修改SQL语句

- row：记录的是每行实际数据的变更

- mixed：statement和row模式的混合

MySQL5.0以前只有statement格式，而这种格式在读已提交(Read Commited)这个隔离级别下主从复制是有bug的，因此Mysql将可重复读(Repeatable Read)作为默认的隔离级别。
接下来，就要说说当binlog为STATEMENT格式，且隔离级别为读已提交(Read Commited)时，有什么bug呢？如下图所示，在主(master)上执行如下事务:
  ![在这里插入图片描述](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/c0243ecb466b6f3c7c30c26bb26c757d.png)
 此时在主库中查询：

```
select * from t;
1
```

输出结果：

```
+---+---+
| c1 |c2
+---+---+
| 2 | 2
+---+---+
1 row in set
```

从库中查询：

```
select * from t;
1
```

输出结果：

```
Empty set
1
```

这里出现了主从不一致性的问题！原因其实很简单，就是在master上执行的顺序为先删后插！而此时binlog为STATEMENT格式，它记录的顺序为先插后删！从(slave)同步的是binglog，因此从机执行的顺序和主机不一致！就会出现主从不一致！
如何解决？
解决方案有两种！
(1)隔离级别设为可重复读(Repeatable Read),在该隔离级别下引入间隙锁。当Session 1执行delete语句时，会锁住间隙。那么，Ssession 2执行插入语句就会阻塞住！
(2)将binglog的格式修改为row格式，此时是基于行的复制，自然就不会出现sql执行顺序不一样的问题！奈何这个格式在mysql5.1版本开始才引入。
**因此由于历史原因，mysql将默认的隔离级别设为可重复读(Repeatable Read)，保证主从复制不出问题！**

当前这个历史遗漏问题以及解决，大家可以将其设置为RC+ROW组合的方式（例如ORACLE等数据库隔离级别就是RC），而不是必须使用RR（会带来更多的锁等待），具体可以视情况选择。