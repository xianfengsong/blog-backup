title: MySQL事务隔离性测试
date: 2018-02-02 13:25:46
categories: mysql进阶
tags: [mysql,数据库]
---
### 环境准备
MySQL版本 5.5.35-rel33.0-log Percona Server with XtraDB (GPL), Release rel33.0, Revision 611
引擎 InnoDB
*(才发现公司用的MySQL是Percona这个公司开发的一个版本，Percona Server [产品介绍][1])*
```
mysql> select version();
+--------------------+
| version()          |
+--------------------+
| 5.5.35-rel33.0-log |
+--------------------+

mysql> show engines \G;
*************************** 6. row ***************************
      Engine: InnoDB
     Support: DEFAULT
     Comment: Percona-XtraDB, Supports transactions, row-level locking, and foreign keys
Transactions: YES
          XA: YES
  Savepoints: YES

```
<!--more-->

### 基本概念
隔离性：事务的操作何时对其他事务可见
脏读/不可重复读/幻读


### 建表
```
CREATE TABLE `test` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(45) DEFAULT NULL,
  `age` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8
```
初始数据如下：
```
mysql> select * from test;
+----+------+------+
| id | name | age  |
+----+------+------+
|  1 | Tom  |   20 |
|  2 | Jack |   22 |
|  3 | Rose |   24 |
|  4 | Bob  |   26 |
+----+------+------+
```
### Repeatable Read 隔离级别测试

**可重复读测试**

启动两个客户端连接，连接一执行事务A，连接二执行事务B。

1.事务A 执行查询操作：
```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from test;
+----+------+------+
| id | name | age  |
+----+------+------+
|  1 | Tom  |   20 |
|  2 | Jack |   22 |
|  3 | Rose |   24 |
|  4 | Bob  |   26 |
+----+------+------+
4 rows in set (0.01 sec)
```
2.事务B 更新数据：
```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update test set age=10 where id=1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

```
3.事务A 再执行查询：
```
mysql> select * from test;
+----+------+------+
| id | name | age  |
+----+------+------+
|  1 | Tom  |   20 |
|  2 | Jack |   22 |
|  3 | Rose |   24 |
|  4 | Bob  |   26 |
+----+------+------+
4 rows in set (0.00 sec)
```
4.事务B 提交：
```
mysql> commit;
Query OK, 0 rows affected (0.00 sec)
```
5.事务A 再次执行查询并提交，然后执行一次查询：
```
mysql> select * from test;
+----+------+------+
| id | name | age  |
+----+------+------+
|  1 | Tom  |   20 |
|  2 | Jack |   22 |
|  3 | Rose |   24 |
|  4 | Bob  |   26 |
+----+------+------+

mysql> commit;

mysql> select * from test;
+----+------+------+
| id | name | age  |
+----+------+------+
|  1 | Tom  |   10 |
|  2 | Jack |   22 |
|  3 | Rose |   24 |
|  4 | Bob  |   26 |
+----+------+------+
```
测试结果结果说明在RR级别下，尽管事务A执行结束前，事务B修改了数据，但是事务A中的几次查询（1,3,5）返回的结果是一致的，这样就保证了可重复读，AB事务的操作时序如下：
|----A----|----B----|
|--begin--|-------------|
|--select--| begin |
|-------------| update|
|-------------| commit|
|--select--|-------------|
|commit|-------------|

**幻读测试**
1.事务A 开始并查询数据：
```
mysql> begin;
mysql> select * from test where age<25;
+----+------+------+
| id | name | age  |
+----+------+------+
|  1 | Tom  |   20 |
|  2 | Jack |   22 |
|  3 | Rose |   24 |
+----+------+------+
```
2.事务B 插入一条新纪录并提交:
```
mysql> begin;
mysql> select * from test;
+----+------+------+
| id | name | age  |
+----+------+------+
|  1 | Tom  |   20 |
|  2 | Jack |   22 |
|  3 | Rose |   24 |
|  4 | Bob  |   26 |
+----+------+------+
mysql> insert test(name,age) values("Tony",18);
mysql> commit;
```
此时新纪录“Tony”也满足A的查询条件
3.事务A 再次查询：
返回结果和上次一致，没有新纪录。
```
mysql> select * from test where age<25;
+----+------+------+
| id | name | age  |
+----+------+------+
|  1 | Tom  |   20 |
|  2 | Jack |   22 |
|  3 | Rose |   24 |
+----+------+------+
```
4.事务A提交 然后执行新查询：
事务提交后，查询返回了新纪录"Tony"。
```
mysql> commit;
mysql> select * from test where age<25;
+----+------+------+
| id | name | age  |
+----+------+------+
|  1 | Tom  |   20 |
|  2 | Jack |   22 |
|  3 | Rose |   24 |
|  5 | Tony |   18 |
+----+------+------+
```

### Read Commit 隔离级别测试
原本的默认级别是RR需要修改一下，只修改当前会话的事务隔离级别：
```
mysql> set session transaction isolation level read committed;
Query OK, 0 rows affected (0.00 sec)

mysql> select @@global.tx_isolation,@@session.tx_isolation;
+-----------------------+------------------------+
| @@global.tx_isolation | @@session.tx_isolation |
+-----------------------+------------------------+
| REPEATABLE-READ       | READ-COMMITTED         |
+-----------------------+------------------------+
```
**可重复读测试：**
和之前类似，事务A执行查询，事务B在A尚未完成时修改数据。
1.事务A 查询:
```
mysql> begin;
mysql> select * from test;
+----+------+------+
| id | name | age  |
+----+------+------+
|  1 | Tom  |   20 |
|  2 | Jack |   22 |
|  3 | Rose |   24 |
|  4 | Bob  |   26 |
+----+------+------+
```
2.事务B修改数据：
```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update test set age=10 where id=1;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```
3.事务A 再次查询：
此时结果还是和上次一致的，这说明并未发生"脏读"
```
mysql> select * from test;
+----+------+------+
| id | name | age  |
+----+------+------+
|  1 | Tom  |   20 |
|  2 | Jack |   22 |
|  3 | Rose |   24 |
|  4 | Bob  |   26 |
+----+------+------+
```
4.事务B 提交：
```
mysql> commit;
```
5.事务A 查询：
在B提交之后A读到了B修改的数据，和之前结果不一致
```
mysql> select * from test;
+----+------+------+
| id | name | age  |
+----+------+------+
|  1 | Tom  |   10 |
|  2 | Jack |   22 |
|  3 | Rose |   24 |
|  4 | Bob  |   26 |
+----+------+------+
mysql> commit;
```
**幻读测试**
既然发生了不可重复读，那么幻读也会发生。
1.事务A 查询：
```
mysql> begin;
mysql> select * from test;
+----+------+------+
| id | name | age  |
+----+------+------+
|  1 | Tom  |   10 |
|  2 | Jack |   22 |
|  3 | Rose |   24 |
|  4 | Bob  |   26 |
+----+------+------+
```
2.事务B 新增记录&提：
```
mysql> begin;
mysql> insert test(name,age) values("Tony",18);
mysql> commit;
```
3.事务A 再查询：
```
mysql> select * from test;
+----+------+------+
| id | name | age  |
+----+------+------+
|  1 | Tom  |   10 |
|  2 | Jack |   22 |
|  3 | Rose |   24 |
|  4 | Bob  |   26 |
|  6 | Tony |   18 |
+----+------+------+
```



  [1]: https://www.percona.com/software/mysql-database/percona-server