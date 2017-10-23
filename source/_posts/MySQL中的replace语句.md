---
title: MySQL中的replace语句
date: 2017-10-06 23:00:55
tags:
---
## 一、背景

当使用replace语句更新数据库中的信息时，原有的字段信息丢失。
错误的原因：
> 认为replace语句相当于当主键冲突时，已修改的列将使用新值，未修改的列将使用旧值。
> 但replace语句对于缺失的列将使用默认值，而不是当前行中的值，导致原有的列信息丢失。
 
## 二、replace语句详解
MySQL中replace语句具体算法如下：
> 1. 尝试把新行插入到表中
> 2. 当因为主键(PRIMARY KEY)冲突错误或(UNIQUE INDEX)唯一索引重复错误而造成插入失败时：
>   * 从表中删除含有重复关键字值的冲突行
>   * 再次尝试把新行插入到表中

可以看出replace语句相当于insert操作或者delete+insert操作，因此，为了能够使用replace语句，必须同时具备insert和delete的权限。
 
replace语句执行时，分以下两种情况：
#### 情况1：insert
当不存在主键冲突或唯一索引冲突，相当于insert操作
#### 情况2：delete and insert
当存在主键冲突或唯一索引冲突，相当于delete操作，加insert操作
 
所有列的值均取自在replace语句中被指定的值，所有缺失的列的值被设置为列的默认值，这和INSERT一样。
不能引用当前行的值，然后用于更新新行。(因为当当前行与新行发生冲突时，当前行将被删除，也就无法被引用了！)
例如，如果使用一个形如“SET col_name = col_name + 1”的赋值，则对位于右侧的列名称的引用会被作为DEFAULT(col_name)处理。
因此，该赋值相当于SET col_name = DEFAULT(col_name) + 1。
 
#### replace语句的返回值
replace语句会返回一个数，来指示受影响的行的数目。该数是被删除和被插入的行数的和。
1.如果对于一个replace语句，返回值为1，则一行被插入，同时没有行被删除。
2.如果该数大于1，则在新行被插入前，有一个或多个旧行被删除。如果表包含多个唯一索引，并且新行与不同行的不同的唯一索引发生了重复。

#### 示例1：test表id作为主键
```
CREATE TABLE test (
  id INT UNSIGNED NOT NULL AUTO_INCREMENT,
  data VARCHAR(64) DEFAULT NULL,
  ts TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id)
);
mysql> REPLACE INTO test VALUES (1, 'Old', '2014-08-20 18:47:00');
Query OK, 1 row affected (0.04 sec)
 
mysql> REPLACE INTO test VALUES (1, 'New', '2014-08-20 18:47:42');
Query OK, 2 rows affected (0.04 sec)
 
mysql> SELECT * FROM test;
+----+------+---------------------+
| id | data | ts                  |
+----+------+---------------------+
|  1 | New  | 2014-08-20 18:47:42 |
+----+------+---------------------+
1 row in set (0.00 sec)
```
第一个replace语句执行时，test表中没有数据，没有发生冲突，所以相当于执行了insert操作，返回值为1（1 row affected）。
第二个replace语句执行时，id=1的数据已存在，发生了主键冲突，所以相当于先执行了delete操作，然后执行了insert操作，返回值为2（2 rows affected）。
#### 示例2：test表id，ts作为主键
```
CREATE TABLE test2 (
  id INT UNSIGNED NOT NULL AUTO_INCREMENT,
  data VARCHAR(64) DEFAULT NULL,
  ts TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id, ts)
);
mysql> REPLACE INTO test2 VALUES (1, 'Old', '2014-08-20 18:47:00');
Query OK, 1 row affected (0.05 sec)
 
mysql> REPLACE INTO test2 VALUES (1, 'New', '2014-08-20 18:47:42');
Query OK, 1 row affected (0.06 sec)
 
mysql> SELECT * FROM test2;
+----+------+---------------------+
| id | data | ts                  |
+----+------+---------------------+
|  1 | Old  | 2014-08-20 18:47:00 |
|  1 | New  | 2014-08-20 18:47:42 |
+----+------+---------------------+
2 rows in set (0.00 sec)
```
因id，ts作为主键，replace时未发生主键冲突，所以均相当于insert操作。

## 三、参考资料
https://dev.mysql.com/doc/refman/5.7/en/replace.html MySQL官网replace语法
http://www.cnblogs.com/c-961900940/p/6197878.html  MySQL中replace into的用法
http://www.cnblogs.com/martin1009/archive/2012/10/08/2714858.html mysql replace into用法详细说明