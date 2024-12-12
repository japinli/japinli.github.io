---
title: PostgreSQL 只读模式
date: 2024-12-11 21:58:31 +0800
tags:
  - PostgreSQL
category: 数据库
---

通常我们谈到关于 PostgreSQL 的只读连接时都是在从节点的环境中。但是，在某些情况下您可能需要在主节点建立只读连接。PostgreSQL 提供了 [`default_transaction_read_only`][] 参数来控制事务读写属性。

<!--more-->

当全局设置 [`default_transaction_read_only`][] 为 `on` 时，此时所有的连接都不能修改数据库，该参数是可重新加载的，因此您不需要重新启动数据库实例，使用 `SELECT pg_reload_conf();` 即可使修改生效。

## 示例

下面是一个简单的示例，首先需要确保你没有修改 [`default_transaction_read_only`][]。

```
postgres=# SHOW default_transaction_read_only;
 default_transaction_read_only
-------------------------------
 off
(1 row)

postgres=# CREATE TABLE IF NOT EXISTS test (id int, info text);
CREATE TABLE
postgres=# INSERT INTO test VALUES (1, 'hello world');
INSERT 0 1
postgres=# SELECT * FROM test;
 id |    info
----+-------------
  1 | hello world
(1 row)
```

目前都是按照预期进行的：我们创建了一张 `test` 表，并插入了一条数据。现在我们尝试将 [`default_transaction_read_only`][] 设置为 `on`。

```
postgres=# ALTER SYSTEM SET default_transaction_read_only TO on;
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# SHOW default_transaction_read_only;
 default_transaction_read_only
-------------------------------
 on
(1 row)
```

接着验证一下修改是否被拒绝：

```
postgres=# INSERT INTO test VALUES (2, 'Hi, PostgreSQL!') RETURNING *;
ERROR:  cannot execute INSERT in a read-only transaction
```

## 注意

虽然可以通过 [`default_transaction_read_only`][] 控制连接的读写属性，但是该参数是会话级别的，这就意味着用户可以在会话级别对其进行修改，例如：

```
postgres=# SET default_transaction_read_only TO off;
SET
postgres=# INSERT INTO test VALUES (2, 'Hi, PostgreSQL!') RETURNING *;
 id |      info
----+-----------------
  2 | Hi, PostgreSQL!
(1 row)

INSERT 0 1
```

此外，当开启 [`default_transaction_read_only`][] 时，并不意味着它不能写入，对于临时表而言，它是可以写入的，如下所示：

```
postgres=# CREATE TEMPORARY TABLE tmp_table (id int, info text);
CREATE TABLE
postgres=# SET default_transaction_read_only TO on;
SET
postgres=# INSERT INTO test VALUES (3, 'Hi, everybody') RETURNING *;
ERROR:  cannot execute INSERT in a read-only transaction
postgres=# INSERT INTO tmp_table VALUES (3, 'Hi, everybody') RETURNING *;
 id |     info
----+---------------
  3 | Hi, everybody
(1 row)

INSERT 0 1
```

需要注意的是，虽然您可以对临时表进行写入操作，但是您没法在只读连接中创建临时表。

```
postgres=# CREATE TEMPORARY TABLE tmp (id int);
ERROR:  cannot execute CREATE TABLE in a read-only transaction
```

## 参考

[1] https://jkatz05.com/post/postgres/postgres-read-only/

[`default_transaction_read_only`]: https://www.postgresql.org/docs/current/runtime-config-client.html#GUC-DEFAULT-TRANSACTION-READ-ONLY

<div class="just-for-fun">
笑林广记 - 长卵叹气

一官到任，出票要唤兄弟三人。一胖子、一长子、一矮子备用。异姓者不许进见。
一家有兄弟四人，仅有一胖三矮。
私相计议曰：“四人之中，胖矮俱有，单少一长人。只得将二矮缝一长裤。两个接起充作长人，便觉全备。”
如计行之。官见大喜，簪花赏酒。三人一时荣宠。下矮压得受苦，在内哓哓，大有怨词。
官听见，问：“下面甚响？”
众慌禀曰：“这是长卵叹气。”
</div>
