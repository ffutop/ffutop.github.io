---
title: MySQL DeadLocks with INSERT 
author: fangfeng
date: 2020-11-06
tags:
  - MySQL
  - DeadLock
categories: 
  - 案例分析
---

首次碰到 INSERT + UPDATE statement 造成的死锁问题，触及了知识盲区，费时做了分析。

## 死锁场景描述

*下述所有事务隔离级别为 Read Committed*

**1. 简化后的数据库表**

```sql
CREATE TABLE `user_address` (
    `id` bigint(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
    `user_id` varchar(40) DEFAULT '' COMMENT '用户主键',
    `comm_address` varchar(1) DEFAULT NULL COMMENT '常用地址标识',
    `address` varchar(200) DEFAULT NULL COMMENT '地址',
    `longitude` bigint(20) DEFAULT NULL COMMENT '经度',
    `latitude` bigint(20) DEFAULT NULL COMMENT '纬度',
    `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '建立时间',
    `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (`id`),
    KEY `idx_user_id` (`user_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='用户常用地址';
```

**2. 死锁事务时序**

| 时序 | 事务 1 | 事务 2 |
| --- | --- | --- |
| 1 | `START TRANSACTION` | |
| 2 | | `START TRANSACTION` |
| 3 | `INSERT INTO user_address(user_id, address, longitude, latitude, create_time, update_time) VALUES ( 'c86d0f931d164fc1b53162355cf39fb4', '浙江省 杭州市 余杭区 五常街道 XXX', 121000000, 31000000, NOW(), NOW());` | |
| 4 | | `INSERT INTO user_address(user_id, address, longitude, latitude, create_time, update_time) VALUES ( 'c86d0f931d164fc1b53162355cf39fb4', '浙江省 杭州市 余杭区 五常街道 XXX', 121000000, 31000000, NOW(), NOW());` |
| 5 | `UPDATE user_address SET comm_address = '0' WHERE user_id = 'c86d0f931d164fc1b53162355cf39fb4';` | |
| 6 | | `UPDATE user_address SET comm_address = '0' WHERE user_id = 'c86d0f931d164fc1b53162355cf39fb4';` |

**3. INNODB Monitor Output**

```plain
------------------------

LATEST DETECTED DEADLOCK

------------------------

2020-11-03 08:25:25 0x7f2801e1d700

*** (1) TRANSACTION:

TRANSACTION 13335216894, ACTIVE 0 sec fetching rows

mysql tables in use 1, locked 1

LOCK WAIT 4 lock struct(s), heap size 1136, 3 row lock(s), undo log entries 1

MySQL thread id 680655, OS thread handle 139810239182592, query id 120901872537 10.1.X.YYY mask_user updating

UPDATE `user_address` SET

		`comm_address` = '0'

		WHERE `user_id` = 'c760c70e42924b869beee418d3b30165'

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 2383 page no 125759 n bits 328 index idx_user_id of table `default`.`user_address` trx id 13335216894 lock_mode X locks rec but not gap waiting

Record lock, heap no 262 PHYSICAL RECORD: n_fields 2; compact format; info bits 0

 0: len 20; hex 6337363063373065343239323462383639626565; asc c760c70e42924b869bee;;

 1: len 4; hex 803c4394; asc  <C ;;



*** (2) TRANSACTION:

TRANSACTION 13335216895, ACTIVE 0 sec starting index read, thread declared inside InnoDB 5000

mysql tables in use 1, locked 1

3 lock struct(s), heap size 1136, 2 row lock(s), undo log entries 1

MySQL thread id 672593, OS thread handle 139809806997248, query id 120901872539 10.1.X.YYY mask_user updating

UPDATE `user_address` SET

		`comm_address` = '0'

		WHERE `user_id` = 'c760c70e42924b869beee418d3b30165'

*** (2) HOLDS THE LOCK(S):

RECORD LOCKS space id 2383 page no 125759 n bits 328 index idx_user_id of table `default`.`user_address` trx id 13335216895 lock_mode X locks rec but not gap

Record lock, heap no 262 PHYSICAL RECORD: n_fields 2; compact format; info bits 0

 0: len 20; hex 6337363063373065343239323462383639626565; asc c760c70e42924b869bee;;

 1: len 4; hex 803c4394; asc  <C ;;



*** (2) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 2383 page no 125759 n bits 328 index idx_user_id of table `default`.`user_address` trx id 13335216895 lock_mode X locks rec but not gap waiting

Record lock, heap no 261 PHYSICAL RECORD: n_fields 2; compact format; info bits 0

 0: len 20; hex 6337363063373065343239323462383639626565; asc c760c70e42924b869bee;;

 1: len 4; hex 803c4393; asc  <C ;;



*** WE ROLL BACK TRANSACTION (2)
```
<!--more-->

## 分析

前置知识：

1. INSERT 会对插入行设置 X 锁 （针对不同的隔离级别，会有所不同）

>  INSERT sets an exclusive lock on the inserted row. This lock is an index-record lock, not a next-key lock (that is, there is no gap lock) and does not prevent other sessions from inserting into the gap before the inserted row.

2. 记录锁锁的是索引记录。DML 使用主键(聚簇)索引时，InnoDB会锁住主键索引；使用非主键(非聚簇/辅助)索引时，InnoDB会先锁住非主键索引，再锁定主键索引。

| 操作 | 事务 1 | 事务 2 |
| --- | --- | --- |
| 1 | `START TRANSACTION` | |
| 2 | | `START TRANSACTION` |
| 3 | `INSERT INTO user_address(user_id, address, longitude, latitude, create_time, update_time) VALUES ( 'c86d0f931d164fc1b53162355cf39fb4', '浙江省 杭州市 余杭区 五常街道 XXX', 121000000, 31000000, NOW(), NOW());` | |
|  | 新记录 A 针对聚簇索引被加了 记录锁 (X lock) | |
| 4 | | `INSERT INTO user_address(user_id, address, longitude, latitude, create_time, update_time) VALUES ( 'c86d0f931d164fc1b53162355cf39fb4', '浙江省 杭州市 余杭区 五常街道 XXX', 121000000, 31000000, NOW(), NOW());` |
|  | | 新记录 B 针对聚簇索引被加了 记录锁 (X lock) |
| 5 | `UPDATE user_address SET comm_address = '0' WHERE user_id = 'c86d0f931d164fc1b53162355cf39fb4';` | |
|  | 执行计划命中 `idx_user_id` ，`UPDATE` 操作需要获取非聚簇索引 `idx_user_id` 的记录锁，但由于记录对应的主键索引已被获取，进入锁等待队列 | |
| 6 | | `UPDATE user_address SET comm_address = '0' WHERE user_id = 'c86d0f931d164fc1b53162355cf39fb4';` |
|  | | 执行计划命中 `idx_user_id` ，`UPDATE` 操作需要获取非聚簇索引 `idx_user_id` 的记录锁，但由于记录对应的主键索引已被获取，进入锁等待队列，且排在事务 1 后面 |

操作 3、4、5、6 造成了：事务 1 的继续执行必须等待事务 2 释放操作 4 加的锁；事务 2 的继续执行必须等待事务 1 操作 5 先获得锁，并释放。死锁条件达成，MySQL 基于检测结果在两个事务间选择了影响较小的事务 2 强制执行了 Rollback 操作。


## 参考资料

\[1\]. [14.7.1 InnoDB Locking](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html)
\[2\]. [14.7.3 Locks Set by Different SQL Statements in InnoDB](https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html)

