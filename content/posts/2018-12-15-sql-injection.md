---
title: SQL 注入
author: fangfeng
date: 2018-12-15
categories:
  - 技术
tags:
  - Security
  - SQL
---

说实话此前对 SQL 注入的理解仅仅只是皮毛。当然，目前也是，只是有了一定程度的理解。

最近好像工具用得有些过头了，需要停下来整理下工具的实现原理。

更好地理解了工具实现，才能更加心安理得地使用工具。毕竟等别人怼的时候，还能够比较安心地回道: "我用不用现成的工具只是取决于我想不想自己再写一套"

当然，毕竟成熟的工具有更多的优化，这就不是短时间内我想不想自己写的问题了，哈哈。

<!--more-->

## 概要

SQL 注入漏洞究竟会产生怎么样的危害性，仅仅只是绕过某些登录账号密码的验证? 只是绕过某些访问限制实现特权内容的访问?

此前我的认识也仅仅只是停留在这里。可谁曾想，SQL SELECT 语句真的是功能强大啊。通过各种各样的字符串拼接，最终能达到的目的，可以说把整个数据库的内容拖下来也不为过。当然，这也是限定在没有实现分库分表，数据库查询账号的权限过大的前提下。

不过，这也就够了。只有真正认识到问题的严重性，最终才会想着去做出一些改善。

## SQL 注入技术

### 基于布尔的注入

最早接触到 SQL 注入问题，还是两个月前。操作的内容也相当简单，猜解某账号的密码。

某接口提供了查询账号的功能，后台的拼接 SQL 可以简单理解成 `SELECT * FROM users WHERE username = '${}'` 。其中 `${}` 就是直接使用的接口请求参数。

而接口的请求结果会根据 SQL 查询的结果，有或没有记录呈现两个不同的页面。

这算是最简单容易的 SQL 注入了，也是凭借个人能力直接能够想到注入点的问题了。

最先做的就是猜解存储密码字段的字段名究竟是什么? 很好，符合常理的设计，直接就是 `passwd` 。

下面就是苦力工作，利用 SQL LIKE 操作符，逐一去猜每一位。`${}` 的注入内容就类似 `admin' AND passwd LIKE '?%` 这里的问号就是逐一猜解的内容(1-9a-zA-Z + 特殊符号)

至于为什么利用 LIKE 操作符，很明确，减少猜解的次数。否则，在密码未知的情况下，即使六位密码也有猜解百万次。而 LIKE 操作符能将猜解次数降为线性，`密码长度 * 字符集数` 次

这里仅仅用了 `AND`，但熟悉了一个，其它就基本类似了。

如果登录也能够注入，认证 SQL 类似 `SELECT * FROM users WHERE username = '${}' AND passwd = '${}'`，那么直接在第一个 `${}` 处注入 `admin' OR 1=1; -- `

### 基于时间的注入

绝大多数时候，注入绝对没有这么简单。也许注入点背后的拼接 SQL 并不对返回值产生影响。例如仅仅只是为了从 DB 查询用户信息，并打一条日志，返回的永远是静态日志（当然，我知道这个例子不恰当，无奈目前只能想到这个）。

既然拼接 SQL 总是存在，但没法给我们一个直观的注入成功 OR 失败的提示。那么，时间就成了一个最好的判断。毕竟是阻塞式的执行。

还是以 `SELECT * FROM users WHERE username = '${}'` 为例，使用类似 `admin' AND IF(passwd LIKE '5%', SLEEP(5), 1);-- ` 的 PAYLOAD ，当满足 `passwd` 以 5 开始时，则 IF 判断进入 `SLEEP(5)` ，根据网页的响应时长就可以进行相应的判断。

### 基于报错的注入

这应该也算一种比较容易自动化的注入方式了。其实在猜解存储密码字段的字段名时，前面也是这样用的。

```sql
> SELECT * FROM users WHERE password ='1';
ERROR 1054 (42S22): Unknown column 'password' in 'where clause'
```

如果程序不做任何处理，对 SQL 错误直接抛出，那么，通过这个就能获得相当多的信息。

由于这部分接触不深，仅仅给出 sqlmap 提的 PAYLOAD

```
AND (SELECT [RANDNUM] FROM(SELECT COUNT(*),CONCAT('[DELIMITER_START]',([QUERY]),'[DELIMITER_STOP]',FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)
```

很标准的 PAYLOAD，而且完全可以。`INFORMATION_SCHEMA` 库的表对所有用户都可查，因此不存在授权的问题。而且表中数据至少大于等于 3 条。

这个 PAYLOAD 一定导致报错的主因，就是对 `RAND()` 与 `GROUP BY` 的配合应用。

> Use of a column with RAND() values in an ORDER BY or GROUP BY clause may yield unexpected results because for either clause a RAND() expression can be evaluated multiple times for the same row, each time returning a different result. If the goal is to retrieve rows in random order, you can use a statement like this:

而真正想要得到的内容，通过 `CONCAT('[DELIMITER_START]',([QUERY]),'[DELIMITER_STOP]',FLOOR(RAND(0)*2))x` 得到，`[QUERY]` 就是真正想要注入的完整SQL串。

而这里的 `DELIMITER_START` `DELIMITER_STOP` 作为界定符，帮助程序提取 `[QUERY]` 得到的结果。否则，直接对请求返回的结果进行过滤可真是太困难了。

### 联合查询注入

联合查询，应该算是最顾名思义的注入方式。使用 `UNION` 在原 SQL 已经限定了查询表的前提下，获得数据库其它库表的信息。

LIKE: `1' UNION SELECT * FROM users;-- ` 这样的 PAYLOAD。

### 堆查询注入

我想这应该是最让人摸不着头脑的命名方式了。

形象化的，我们利用 PAYLOAD 来进行说明。`1'; INSERT INTO users (user, passwd) VALUES ('aaa', 'aaa');-- `

看起来和 UNION 挺像的哈，都是补充上一个 OR 多个 SQL 。当然，也是有区别的，否则为什么会出现这样一种技术呢。

最大的区别，就是堆查询注入能够完成 `UPDATE`, `INSERT`, `DELETE` 等操作，毕竟，UNION 联合的只能是作为查询了（可能说法不太恰当，但姑且这样吧）

### 另类注入

之前的几种，我们都是利用了 `SELECT` 完成的注入，那么对于 `INSERT`, `UPDATE` 之类的语句是否有注入的可能呢。当然也是存在可能的。

不过，精力有限，而且目前看来也不需要做到这一步。暂时挖个坑吧。不填

## SQLMAP 

简单的了解了几种注入原理之后，还是要看看 SQL 注入神器的——`SQLMAP`

也算是尝试了好久才真正了解它的强大之处。此处强烈推荐 DVWA，`Damn Vulnerable Web Application`，一个用来合法攻击的工具。

部署方式也是开箱可用，只要有 docker，直接 `docker run --rm -it -p 80:80 vulnerables/web-dvwa` 即可完成部署。

对于将 DVWA 安全级别设置为 low 时，仅仅只需要执行下列命令就可以获取到 DB 里几乎完整的信息。

```sh
$ sqlmap -u https://127.0.0.1/vulnerabilities/sqli/\?id\=1\&Submit\=Submit\# --cookie="PHPSESSID=jhton35qahj78l3unjea9k2lf7;security=low" -v 3 --banner
```

当然，换一下相关获取的内容，例如把 `--banner` 换成 `--dump` ，我们借此来简单看看 SQL 注入漏洞的可怕之处

```sh
[10:41:39] [INFO] using default dictionary
do you want to use common password suffixes? (slow!) [y/N]
[10:41:40] [INFO] starting dictionary-based cracking (md5_generic_passwd)
[10:41:40] [INFO] starting 8 processes
[10:41:42] [INFO] cracked password 'abc123' for hash 'e99a18c428cb38d5f260853678922e03'
[10:41:44] [INFO] cracked password 'charley' for hash '8d3533d75ae2c3966d7e0d4fcc69216b'
[10:41:47] [INFO] cracked password 'letmein' for hash '0d107d09f5bbe40cade3de5c71e9e9b7'
[10:41:49] [INFO] cracked password 'password' for hash '5f4dcc3b5aa765d61d8327deb882cf99'
[10:41:53] [DEBUG] post-processing table dump
Database: dvwa
Table: users
[5 entries]
+---------+-----------------------------+---------+---------------------------------------------+-----------+------------+---------------------+--------------+
| user_id | avatar                      | user    | password                                    | last_name | first_name | last_login          | failed_login |
+---------+-----------------------------+---------+---------------------------------------------+-----------+------------+---------------------+--------------+
| 1       | /hackable/users/admin.jpg   | admin   | 5f4dcc3b5aa765d61d8327deb882cf99 (password) | admin     | admin      | 2018-12-15 00:42:31 | 0            |
| 2       | /hackable/users/gordonb.jpg | gordonb | e99a18c428cb38d5f260853678922e03 (abc123)   | Brown     | Gordon     | 2018-12-15 00:42:31 | 0            |
| 3       | /hackable/users/1337.jpg    | 1337    | 8d3533d75ae2c3966d7e0d4fcc69216b (charley)  | Me        | Hack       | 2018-12-15 00:42:31 | 0            |
| 4       | /hackable/users/pablo.jpg   | pablo   | 0d107d09f5bbe40cade3de5c71e9e9b7 (letmein)  | Picasso   | Pablo      | 2018-12-15 00:42:31 | 0            |
| 5       | /hackable/users/smithy.jpg  | smithy  | 5f4dcc3b5aa765d61d8327deb882cf99 (password) | Smith     | Bob        | 2018-12-15 00:42:31 | 0            |
+---------+-----------------------------+---------+---------------------------------------------+-----------+------------+---------------------+--------------+
```

这里就可以看到 `dvwa.users` 表的全部内容，甚至连简单密码都帮你完成了爆破。

更多内容可以自行尝试，说真的，这个工具相当强大。如果仅仅说我针对某型数据库(例如 MySQL)完成诸如基于报错的注入，在不考虑鲁棒性的情况下，完全可以做到。但是支持超10余种数据库，且自动完成注入的全过程...

## 预编译 SQL

提了这么多，究竟 SQL 注入就不能够预防吗? 当然可以，不然整个网络环境下的WEB应用不得都完蛋了。有釜底抽薪的处理方式吗，也有。

预编译 SQL 就是一个超级 OK 的解决方案。同时在使用上也能够明显提升 DB 查询效率

我相信预编译 SQL 很多人写过，毕竟借助例如 MyBatis 框架，能够快速实现 SQL 预编译语句的编写。同时在求学过程中估计也都用过 Java 原生 JDBC 的 `PreparedStatement` 。

那么，与 MySQL 交互时的预编译 SQL 是怎么样地填上了占位符呢? 先来看看再说

```sql
mysql> prepare {name} from 'SELECT * FROM users WHERE user=? AND passwd=?'; # 这里的 {name} 可以自定义命名，无需 {}

mysql> set @a='admin', @b='password';     # 声明变量，并赋值

mysql> execute {name} using @a, @b;     # 提供变量并执行预编译 SQL
```

我想看到这里应该就应该能够明白了吧，MySQL 对于预编译 SQL 的每一个占位符，都已经认定了是单一的元素，也就不可能存在允许打破已有预编译结果的内容了。

即使真的注入了 `admin OR 1=1` 之类的内容，也是会被认为这是一个完整的字符串，用来替代 `user` 字段或 `passwd` 字段，根本不可能重新拆解。

 ```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```
