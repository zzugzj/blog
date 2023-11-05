---
title: InnoDB的锁和事务模型
auther: gzj
tags:
  - MySQL
categories: 数据库
description: >-
  InnoDB与MyISAM的最大不同有两点:一是支持事务(TRANSACTION);二是采用了行级锁。关于事务我们之前有专题介绍,这里就着重介绍下它的锁机制。
  总的来说,InnoDB按照不同的分类...
abbrlink: 5c27e2b9
---

## InnoDB的锁

### 共享锁和排他锁

InnoDB通过共享锁和排他锁实现了行级锁。

- 共享(s)锁允许持有锁的事务读取一行。
- 排他(x)锁允许持有锁的事务更新或删除一行。

事务T1在row行上拥有s锁，那事务T2申请row行上的s锁可立即获得，T1，2会同时拥有s锁。申请x锁需要等待T1释放锁。

事务T1在row行上拥有x锁，那事务T2申请row行的s或x锁必须等待T1释放锁。

### 意向锁

InnoDB存储引擎支持多粒度的锁：即可以同时支持行锁和表锁。为了支持这个机制，有了意图锁。

意向锁是表级锁，目的是表明事务稍后需要对表中的行使用那种类型的锁(s或x)。

存在两种类型的意向锁：

- 意向共享(IS)锁：事务打算在一张表的个别行申请共享锁(select ... lock in share mode)。
- 意向排他(IX)锁：事务打算再一张表的个别行申请排他锁(select ... for update)。

意向锁需要遵循一下协定：

- 事务打算在一张表的个别行申请一个s锁前，必须先申请表的IS或IX锁。
- 事务打算在一张表的个别行申请一个x锁前，必须先申请表的IX锁。

表的锁类型兼容性：

|      | X    | IX   | S    | IS   |
| ---- | ---- | ---- | ---- | ---- |
| X    | 互斥 | 互斥 | 互斥 | 互斥 |
| IX   | 互斥 | 兼容 | 互斥 | 兼容 |
| S    | 互斥 | 互斥 | 兼容 | 兼容 |
| IS   | 互斥 | 兼容 | 兼容 | 兼容 |

（意图锁之间互相兼容，其余遵循X,S互斥原则）

如果事务请求的锁与现有锁兼容，则将其授予请求的事务。 事务会等待直到现有的锁被释放。 如果锁请求与现有锁发生冲突，并且由于可能导致死锁而无法被授予许可，则会发生错误。

意向锁不会阻塞除表级锁(比如：lock tables ... write)外的任何锁，意图锁的主要目的是表明有人正在锁表中的行，或者将要锁表中的行。

### 记录锁(Record Locks)

记录锁是索引记录上的锁。比如：SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE;就是给c1=10的行上加排他锁，防止别的事务的insert、update、delete这个c1=10的行。即使这个表没有定义索引，InnoDB也会隐式的创建一个聚集索引来加记录锁。

记录锁的事务在SHOW ENGINE INNODB STATUS和InnoDB监视器输出中看起来类似于以下内容：

```mysql
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t`
trx id 10078 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
```

### 间隙锁(Gap Locks)

间隙锁是对索引记录间的间隙的锁定。比如：SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE;给10-20上排他间隙锁，防止其他事务插入10-20间的值。

间隙锁在部分隔离级别下使用。

对于使用唯一索引来查询行的语句，不需要间隙锁(如果是多列的索引，需要where条件全部命中，才不会加间隙锁，只会加记录锁)。例如，`SELECT * FROM child WHERE id = 100;`只有记录锁，没有间隙锁。如果id没有索引或者不是唯一索引，则该语句会锁住前面的间隙。

注意：不同的事务可以在一个间隙上拥有不同的间隙锁。例如：事务A在一个间隙上拥有共享间隙锁，同时事务B在同一间隙上可以拥有排他间隙锁（无论是间隙S锁还是间隙X锁）。这个机制被允许存在是因为，一个记录被删除时，会合并不同事务在这个记录上的间隙锁。

InnoDB中的间隙锁“纯粹是抑制性的”，这意味着它们的唯一目的是防止其他事务更改间隙间的内容。 间隙锁可以共存。一个事务进行的间隙锁不会阻止另一事务对相同的间隙获取间隙锁。 共享间隙锁和排他间隙锁之间没有区别。 它们彼此不冲突，并且执行相同的功能。

间隙锁可以在READ COMMITTED隔离级别下显式禁用。在这种情况下，将禁用间隙锁进行搜索和索引扫描，并且仅将其用于外键约束检查和重复键检查。

在READ COMMITTED隔离级别上还有其他效果。 MySQL评估WHERE条件后，将释放不匹配行的记录锁。 对于UPDATE语句，InnoDB进行“半一致”读取，将最新的提交版本返回给MySQL，以便MySQL可以确定该行是否与UPDATE的WHERE条件匹配。

### Next-Key Locks

Next-Key Locks是索引记录上的记录锁和索引记录之前的间隙上的间隙锁的组合。

InnoDB执行行级锁的方式是：当它搜索或扫描表索引时，会在遇到的索引记录上设置共享或排他锁。 因此，行级锁实际上是索引记录锁。 索引记录上的Next-Key Locks也会影响该索引记录之前的“间隙”。 所以，Next-Key Locks是索引记录锁加上索引记录之前的间隙上的间隙锁。

如果一个会话在索引中的记录R上具有共享或排他锁，则另一会话不能按照索引顺序在R之前的间隙中插入新的索引记录。

假设一个索引包含10，11，13和20，该索引可能的Next-Key Locks覆盖以下间隔：

```mysql
(负无穷, 10]
(10, 11]
(11, 13]
(13, 20]
(20, 正无穷)
```

最后一个间隔，next-key lock锁定最大索引值之后的间隙。

默认情况下，InnoDB在REPEATABLE READ隔离级别中运行。 在这种情况下，InnoDB使用next-key锁进行搜索和索引扫描，这可以防止幻读。

### 插入意向锁

插入意向锁是一种在行插入之前通过INSERT操作设置的间隙锁。

如果多个事务插入共同的索引间隙的不同位置，那无需等待。假设有索引记录，其值分别为4和7。单独的事务分别尝试插入值5和6，在获得插入行的排他锁之前，每个事务都使用插入意图锁来锁定4和7之间的间隙， 但不互相阻塞，因为行是无冲突的。

举例：客户端A创建一张表有2个索引数据（90和102），开始一个事务，放置一个排他索引记录锁在ID>100的数据，排他锁也包括一个102数据之前的间隙锁。

```mysql
mysql> CREATE TABLE child (id int(11) NOT NULL, PRIMARY KEY(id)) ENGINE=InnoDB;
mysql> INSERT INTO child (id) values (90),(102);

mysql> START TRANSACTION;
mysql> SELECT * FROM child WHERE id > 100 FOR UPDATE;
+-----+
| id  |
+-----+
| 102 |
+-----+
```

客户端B开始一个事务插入一条记录到间隙中，这个事务在等待排他锁的时候持有一个插入意图锁

```mysql
mysql> START TRANSACTION;
mysql> INSERT INTO child (id) VALUES (101);
```

### AUTO-INC锁（自增锁）

AUTO-INC锁是一种特殊的表级锁，当事务插入表并有自增列的时候持有。 在最简单的情况下，如果一个事务正在向表中插入值，其他事务都必须等待自己的插入，以便第一个事务插入的行接收连续的主键值。

## autocommit, Commit, and Rollback

在InnoDB中，所有用户活动都在事务内部进行。 如果启用了自动提交模式，则每个SQL语句将自己形成一个事务。 默认情况下，MySQL在启用了自动提交的情况下为每个新连接启动会话，因此如果该SQL语句未返回错误，则MySQL在每个SQL语句之后执行一次提交。 如果一条语句返回错误，则提交或回退行为取决于该错误。 

启用了自动提交的会话可以通过以显式START TRANSACTION或BEGIN语句开始并以COMMIT或ROLLBACK语句结束的方式执行多语句事务。

如果在SET autocommit = 0的会话中禁用了自动提交模式，则该会话将始终打开一个事务。 COMMIT或ROLLBACK语句结束当前事务，并开始新的事务。如果没有在显式提交最终事务的情况下结束，则MySQL将回滚事务。

COMMIT表示在当前事务中所做的更改将永久化，并在其他会话中可见。 另一方面，ROLLBACK语句取消当前事务所做的所有修改。 COMMIT和ROLLBACK都释放在当前事务期间设置的所有InnoDB锁。

## InnoDB中由不同的SQL语句设置的锁

锁读取、更新或删除通常会对SQL语句扫描的每个索引记录设置记录锁。语句中是否存在排除该行的WHERE条件并不重要。InnoDB不知道确切的WHERE条件，只知道扫描了哪些索引范围。这些锁通常是next-key锁，也会阻止插入到紧靠记录之前的“间隙”中。

如果没有适合语句的索引，MySQL必须扫描整个表来处理语句，那么表的每一行都会被锁定，从而阻止其他用户对表的所有插入。创建好的索引非常重要，这样查询就不会扫描许多行。

下面是一些InnoDB设置的特殊类型的锁：

- SELECT ... FROM是一致的读取，读取数据库的快照并且没有锁，除非将事务隔离级别设置为SERIALIZABLE。对于SERIALIZABLE级别，搜索会在遇到的索引记录上设置共享的下一键锁定。但是，对于使用唯一索引来搜索唯一行的行锁定的语句，仅需要索引记录锁。 

- SELECT ... FOR UPDATE或SELECT ... LOCK IN SHARE MODE，将为扫描的行获取锁，并释放在结果集中不符合查询条件的行（例如，如果它们不符合WHERE子句中给出的条件）。但是，在某些情况下，行可能不会立即被解锁，因为结果行与其原始行之间的关系在查询执行过程中会丢失。例如，在UNION中，在评估它们是否符合结果集之前，可以将表中的扫描（和锁定）行插入到临时表中。在这种情况下，临时表中的行与原始表中的行之间的关系将丢失，并且直到查询执行结束后，行才被解锁。 

- SELECT ... LOCK IN SHARE MODE在所有遇到的索引记录上设置共享的next-key锁。但是，对于使用唯一索引来搜索的语句，仅需要索引记录锁定。 

- SELECT ... FOR UPDATE在所有遇到的记录上设置排他的next-key锁。但是，对于使用唯一索引来搜索的语句，仅需要索引记录锁定。 

  对于查询索引记录遇到的问题，SELECT ... FOR UPDATE阻止其他会话执行SELECT ... LOCK IN SHARE MODE。

- UPDATE ... WHERE ...在搜索遇到的每条记录上设置排他的next-key锁。但是，对于使用唯一索引的语句，仅需要索引记录锁。 

- 当UPDATE修改聚簇索引记录时，将对受影响的辅助索引记录进行隐式锁定。在插入新的二级索引记录之前执行重复检查扫描时，以及在插入新的二级索引记录时，UPDATE操作还会在受影响的二级索引记录上获得共享锁。 

- DELETE FROM ... WHERE ...在搜索遇到的每条记录上设置独占的next-key锁。但是，对于使用唯一索引的语句，仅需要索引记录锁定。 

- INSERT在插入的行上设置排他锁。该锁是索引记录锁，不是next-key锁（即没有间隙锁），并且不会阻止其他事务插入到插入行之前的间隙中。 

  在插入行之前，设置了一种称为插入意图间隙锁的间隙锁。此锁表示：如果插入到同一索引间隙中的多个事务不在间隙中的同一位置插入，则它们无需等待对方。假设有索引记录，其值为4和7，在插入行上获得排他锁之前，插入值5和6的事务都使用插入意图锁来锁定4和7之间的间隙，不会彼此阻塞。

## 死锁

### 一个死锁例子

下面的示例说明了锁请求可能会导致死锁。 该示例涉及两个客户端A和B。

首先，客户端A创建一个包含一行的表，然后开始事务。 在事务中，A通过在共享模式下选择该行来获得该行的S锁：

```mysql
mysql> CREATE TABLE t (i INT) ENGINE = InnoDB;
Query OK, 0 rows affected (1.07 sec)

mysql> INSERT INTO t (i) VALUES(1);
Query OK, 1 row affected (0.09 sec)

mysql> START TRANSACTION;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT * FROM t WHERE i = 1 LOCK IN SHARE MODE;
+------+
| i    |
+------+
|    1 |
+------+
```

客户端B开启一个事务并尝试删除一行：

```mysql
mysql> START TRANSACTION;
Query OK, 0 rows affected (0.00 sec)

mysql> DELETE FROM t WHERE i = 1;
```

删除操作需要x锁，但由于客户端A持有s锁，不与x锁兼容，客户端请求不到x锁，发生阻塞，进入请求队列中。

最后，客户端A尝试删除该行：

```mysql
mysql> DELETE FROM t WHERE i = 1;
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

此处发生了死锁，客户端A需要x锁才能删除该行，但不能授予x锁。因为B已经在申请x锁，并等待A释放s锁。由于B事先要求X锁，因此A持有的S锁也不能升级为X锁。结果就是InnoDB为其中 一个客户端生成错误并释放其锁，客户端返回错误。

### 死锁检测

启用死锁检测（默认）后，InnoDB自动检测事务死锁并回滚一个或多个事务以打破死锁。 InnoDB会尝试选择一些小的事务以进行回滚，其中事务的大小取决于插入，更新或删除的行数。

如果innodb_table_locks = 1（默认值）且autocommit = 0，则InnoDB可以检测表锁，并且它上面的MySQL层可以检测行级锁。 否则，InnoDB无法检测到涉及MySQL LOCK TABLES语句设置的表锁或InnoDB以外的存储引擎设置的锁的死锁。 通过设置innodb_lock_wait_timeout系统变量的值来解决这些情况。

如果InnoDB Monitor输出的LATEST DETECTED DEADLOCK部分包含一条消息，指出`TOO DEEP OR LONG SEARCH IN THE LOCK TABLE WAITS-FOR GRAPH, WE WILL ROLL BACK FOLLOWING TRANSACTION,`，这表明等待的事务数 list已达到200的限制。超过200个事务的等待列表将被视为死锁，并且尝试检查等待列表的事务将回退。 如果锁定线程必须查看等待列表上的事务拥有的1,000,000个以上的锁，也可能发生相同的错误。

#### 禁用死锁检测

在高并发系统上，当多个线程等待相同的锁时，死锁检测会导致速度变慢。 有时，当发生死锁时，禁用死锁检测并依靠innodb_lock_wait_timeout设置进行事务回滚可能会更有效。 可以使用innodb_deadlock_detect配置选项禁用死锁检测。

### 如何减少和处理死锁

死锁是含有事务的数据库的经典问题，但死锁并不危险，除非死锁发生很频繁以至于无法运行某些事务。通常需要编写应用程序，以便在由于死锁而使事务回滚时，可以准备重新启动事务。

InnoDB使用行级锁。 即使在仅插入或删除单行的事务中，可能会陷入死锁。 这是因为这些操作并不是真正的“原子”操作。 它们会自动对插入或删除的行的（可能是多个）索引记录设置锁。

您可以使用以下技术来处理死锁并减少发生死锁的可能性：

- 在任何时候，发出SHOW ENGINE INNODB STATUS命令来确定最新死锁的原因。 这可以帮助您调整应用程序以避免死锁。

- 如果频繁出现死锁警告引起关注，请通过启用innodb_print_all_deadlocks配置选项来收集更广泛的调试信息。有关每个死锁的信息，而不仅仅是最新的死锁，都记录在MySQL错误日志中。 完成调试后，请禁用此选项。

- 如果由于死锁而失败，请始终准备重新发出事务。 死锁并不危险。 请再试一次。

- 保持事务小巧且持续时间短，以使事务不易发生冲突。

- 如果使用锁定读取（SELECT ... FOR UPDATE或SELECT ... LOCK IN SHARE MODE），请尝试使用较低的隔离级别，例如READ COMMITTED。

- 修改事务中的多个表或同一表中的不同行时，每次都要以一致的顺序执行这些操作。 然后，事务形成定义良好的队列，并且不会死锁。 例如，将数据库操作组织到应用程序内的函数中，或调用存储的例程，而不是在不同位置编码多个类似的INSERT，UPDATE和DELETE语句序列。

- 将索引添加到表中。然后，查询需要扫描更少的索引记录，并因此设置更少的锁。 使用EXPLAIN SELECT确定MySQL服务器认为哪个索引最适合需要的查询。

- 使用更少的锁定。 如果您有能力允许SELECT从旧快照返回数据，请不要在其中添加FOR UPDATE或LOCK IN SHARE MODE子句。在这里使用READ COMMITTED隔离级别是好的，因为在同一事务中的每个一致读取都从其自己的新快照读取。

- 如果没有其他帮助，请使用表级锁序列化事务。将LOCK TABLES与事务表（例如InnoDB表）一起使用的正确方法是，先以SET autocommit = 0（不是START TRANSACTION）加上LOCK TABLES开始事务，然后在明确提交事务之前不调用UNLOCK TABLES。 例如，如果您需要写入表t1并从表t2中读取，则可以执行以下操作：

  ```mysql
  SET autocommit=0;
  LOCK TABLES t1 WRITE, t2 READ, ...;
  ... do something with tables t1 and t2 here ...
  COMMIT;
  UNLOCK TABLES;
  ```

  表级锁可防止对表的并发更新，从而避免死锁，但代价是对繁忙系统的响应速度较慢。

- 序列化事务的另一种方法是创建一个仅包含一行的辅助“信号量”表。 在访问其他表之前，让每个事务更新该行。 这样，所有事务都以串行方式进行。 请注意，在这种情况下，InnoDB即时死锁检测算法也适用，因为序列化锁是行级锁。 对于MySQL表级锁，必须使用超时方法来解决死锁。