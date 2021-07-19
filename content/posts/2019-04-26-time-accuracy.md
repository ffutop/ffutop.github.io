---
title: MySQL TIMESTAMP 时间精度问题
author: fangfeng
date: 2019-04-26
categories:
  - 技术
tags:
  - MySQL
  - DateTime
  - TimeStamp
  - time accuracy
---

最近一段单元测试代码总是不定时地爆炸。test pass 与 failed 的比例大约 10:1 。伪代码如下:

```java
/**
  * 表结构
  * CREATE TABLE `time_0` (
  *     `timeout` timestamp NOT NULL
  * )
  */

// part 1
jdbcTemplate.execute("UPDATE `time_0` SET `timeout`=now() WHERE `id` = xxx;");

// part 2
Date timeout = jdbcTemplate.queryForObject("SELECT `timeout` FROM `time_0` WHERE `id` = xxx;", Date.class);
Assert.assertTrue(timeout.getTime() < System.currentTimeMillis());
```

在绝大多数模拟中，先执行 `part 1`，紧跟着执行 `part 2` 都能通过测试。但偶尔还是挂掉了。

<!--more-->

## MySQL 时间表示

| **Type**  | **Storage before MySQL 5.6.4** | **Storage as of MySQL 5.6.4**                    |
| --------- | ------------------------------ | ------------------------------------------------ |
| YEAR      | 1 byte, little endian          | Unchanged                                        |
| DATE      | 3 bytes, little endian         | Unchanged                                        |
| TIME      | 3 bytes, little endian         | 3 bytes + fractional-seconds storage, big endian |
| TIMESTAMP | 4 bytes, little endian         | 4 bytes + fractional-seconds storage, big endian |
| DATETIME  | 8 bytes, little endian         | 5 bytes + fractional-seconds storage, big endian |

<center><small>本表来源于 [Date and Time Data Type Representation](https://dev.mysql.com/doc/internals/en/date-and-time-data-type-representation.html)</small></center>

`TimeStamp` 由四字节描述，可以存储 `1970-01-01 00:00:00` 到 `2038-01-19 03:14:07`。4 字节存储正好是精确到秒为止。根据表中所描述的毫秒存储，是依赖于额外的存储空间。

```sh
mysql> show variables like 'version';
+---------------+--------+
| Variable_name | Value  |
+---------------+--------+
| version       | 5.7.20 |
+---------------+--------+
```

OK，版本大于 5.6.4，能够支持毫秒级精度。

```sh
# mysql-cli 直接插入新的数据
mysql> INSERT INTO `time_0` (timeout) VALUES ('2019-04-26 08:00:00.500');

# Check 结果
mysql> SELECT * FROM `time_0`;
+---------------------+
| timeout             |
+---------------------+
| 2019-04-26 08:00:01 |
+---------------------+
1 row in set (0.00 sec)
```

被向上取整了？没有存储毫秒级数据？还是说 `SELECT` 展示结果的时候被加工了？直接去检查数据文件 `/path/to/time_0.ibd`

```sh
# 2019-04-26 08:00:00 -> 1556236800 second -> 0x5cc24a00 
# 2019-04-26 08:00:00 -> 1556236801 second -> 0x5cc24a01
$ xxd time_0.ibd | grep 5cc2
0000c090: 5cc2 4a01 0000 0000 0000 0000 0000 0000  \.J.............
```
存储时已经被四舍五入了，存了 `5cc24a01`。

:< 如果需要存储毫秒级精度，需要在声明类型 `timestamp` 时添加毫秒精度的声明。Like `timestamp(1)`：最小精度 0.1 秒。不同的毫秒精度还将决定所需的存储空间大小。

| 毫秒精度 | 存储空间 |
| ------- | ----------- |
| 0       | 0 bytes     |
| 1,2     | 1 byte      |
| 3,4     | 2 bytes     |
| 4,5     | 3 bytes     |

新建表 `time_1` (timestamp(3))

```sh
mysql> insert into time_1 (timeout) values ('2019-04-26 08:00:00.500');
mysql> select * from time_1;
+----+-------------------------+
| id | timeout                 |
+----+-------------------------+
|  1 | 2019-04-26 08:00:00.500 |
+----+-------------------------+
```

从 `/path/to/time_1.ibd` 检查数据

```sh
# 2019-04-26 08:00:00.500 -> 2019-04-26 08:00:00.5000  3,4 位毫秒级精度存储方式相同
# -> 1556236800.5000 second -> 0x5cc24a00 0x1388
$ xxd time_1.ibd | grep -A 1 5cc2
0000c080: 0100 0000 008c 87ce 0000 01e4 0110 5cc2  ..............\.
0000c090: 4a00 1388 0000 0000 0000 0000 0000 0000  J...............
```

OK，看来确实如此。

## NOW() 

确认了 MySQL 对 `TIMESTAMP` 的存储方式。还有 `now()` 函数的表现亟待确认。

```mysql
mysql> SELECT now();
+---------------------+
| now()               |
+---------------------+
| 2019-04-26 09:03:27 |
+---------------------+
1 row in set (0.00 sec)

mysql> SELECT now(3);
+-------------------------+
| now(3)                  |
+-------------------------+
| 2019-04-26 09:03:31.491 |
+-------------------------+
1 row in set (0.00 sec)

# 加入 `sleep(1)` 是为了体现两个 now() 是同时产生效果的。
# 但并没有出现四舍五入的现象。从网上的源码看到 now(3) 对于毫秒级数据是额外附加的。
mysql> SELECT now(), sleep(1), now(3);
+---------------------+----------+-------------------------+
| now()               | sleep(1) | now(3)                  |
+---------------------+----------+-------------------------+
| 2019-04-26 09:04:58 |        0 | 2019-04-26 09:04:58.946 |
+---------------------+----------+-------------------------+
1 row in set (1.00 sec)
```

## 结论

综合 `now()` 和 `timestamp` 的表现，结论就是默认是按向下取整的方式进行的。因为 `UPDATE` 语句用了 `now()`。

既然如此，还能够出现已经设置 `timeout = now()` 而 SELECT 得到的 `timeout > System.currentTimeMillis()`。只能推断为数据库服务器和本机的系统时间不一致，而且数据库服务器的时间更快，但快的有限，不超过 1 秒。

至于如何比较两台机器的时间差，`clockdiff` 是个好工具，但没找到 MacOS 下的替代品。具体计算时间差，只能暂时放弃了。
```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```
