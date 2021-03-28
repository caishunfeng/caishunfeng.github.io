---
layout: post
title: "事务隔离级别"
date: 2021-03-28 15:55:00 +0800
categories: transaction-isolation-level
---

### 事务特性：

事务有 ACID 四大特性：

- Atomicity（原子性）：事务中的全部操作不可分割，要么全部成功，要么全部失败；
- Consistency（一致性）：事务必须使数据库从一个一致性状态变换到另一个一致性状态；
- Isolation（隔离性）：事务在操作的过程中不会被其他事务所干扰，并发事务之间相互隔离；
- Durability（持久性）：事务一旦被提交了，那么对数据库中的数据的改变就是永久性的；

### innodb 的事务隔离级别：

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
| Read uncommitted (读未提交) | 会 | 会 | 会 |
| Read committed (读已提交) | 不会 | 会 | 会 |
| Repeatable read (可重复读) | 不会 | 不会 | 不会 |
| Serializable (串行化) | 不会 | 不会 | 不会 |

{% highlight sql %}

-- 查看当前会话隔离级别
select @@tx_isolation;
-- 查看系统当前隔离级别
select @@global.tx_isolation;
-- 查看是否自动提交
select @@autocommit;

-- 设置当前会话的隔离级别
set session transaction isolation level read committed;
-- 设置全局的隔离级别
set global transaction isolation level read committed;

CREATE TABLE `user` (
`id` int(11) unsigned NOT NULL AUTO_INCREMENT,
`name` varchar(64) DEFAULT NULL,
`age` int(4) DEFAULT NULL,
`sex` tinyint(1) DEFAULT NULL,
PRIMARY KEY (`id`),
UNIQUE KEY `uniq_name` (`name`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4;

{% endhighlight %}

- 脏读：在一个事务的处理过程中读取到了其他事务未提交的数据；

| Time | A | B |
| 0 | insert into user(name,age,sex) values('张三',20,1); | |
| 1 | start transaction; | |
| 2 | update user set age = age + 1 where name = '张三'; | |
| 3 | | start transaction; |
| 4 | | select age from user where name = '张三'; (result: 21) |
| 5 | rollback; | |

Time4：虽然 A 未提交，但此时 A 事务操作的数据对 B 已可见，即产生了脏读

<br>
- 不可重复读：在一个事务的处理过程中查询同一记录两次的结果可能不一致（该记录被其他事务操作并提交了）；

| Time | A | B |
| 0 | insert into user(name,age,sex) values('张三',20,1); | |
| 1 | start transaction; |
| 2 | update user set age = age + 1 where name = '张三'; |
| 3 | | start transaction; |
| 4 | | select age from user where name = '张三';(result: 20) |
| 5 | commit; | |
| 6 | | select age from user where name = '张三';(result: 21) |

Time6：A 在事务 B 开启后提交，此时 B 已经可以读取到 A 事务操作的最新数据，即对 B 来说，两次读取同一记录结果不一致)

<br>
- 幻读：在一个事务的处理过程中相同查询执行两次，可能产生不一样的结果集，即产生幻行；

| Time | A | B |
| 0 | insert into user(name,age,sex) values('张三',20,1); | |
| 1 | start transaction; |
| 2 | insert into user(name,age,sex) values('李四',30,1); |
| 3 | | start transaction; |
| 4 | | select count(1) from user; (result: 1) |
| 5 | commit; | |
| 6 | | select count(1) from user; (result: 2) |

Time6：A 在事务 B 开启后提交，此时 B 已经读取到 A 事务新增的李四这条数据，即对事务 B 来说，两次查询的结果集不一致，产生了幻行；
innodb 通过 MVCC(多版本并发控制)解决幻读现象；

### 拓展：

- [MVCC][mvcc]
- [phantom rows][phantom rows]

[mvcc]: https://dev.mysql.com/doc/refman/5.6/en/innodb-multi-versioning.html
[phantom rows]: https://dev.mysql.com/doc/refman/5.6/en/innodb-next-key-locking.html
