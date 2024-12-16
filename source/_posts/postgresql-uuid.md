---
title: PostgreSQL UUID 类型
date: 2024-12-16 21:59:26 +0800
categories: 数据库
tags:
  - PostgreSQL
---

UUID 代表通用唯一标识符，在 [RFC 4122][] 中定义。它是一个 128 位数字，通常以十六进制书写，并用破折号分成五组。典型的 UUID 值如下所示：

	0193cfc2-5f8c-7f75-87ce-d26c47161eb5

在 [78c5e141][] 之前，PostgreSQL 只提供了 v4 版本，提交 [78c5e141][] 引入了 v7 版本的支持。不出意外的话，该功能将在 PostgreSQL 18 中发布。

<!-- more -->

## 示例

我们创建连个表来测试 v4 与 v7 之间的差异。

``` sql
CREATE TABLE uuid_v4_test(id uuid primary key, info text);
CREATE TABLE uuid_v7_test(id uuid primary key, info text);
```

接着，灌入数据。

``` sql
postgres=# INSERT INTO uuid_v4_test SELECT uuidv4(), 'data' || id FROM generate_series(1, 1000000) id;
INSERT 0 1000000
Time: 9383.798 ms (00:09.384)
postgres=# INSERT INTO uuid_v7_test SELECT uuidv7(), 'data' || id FROM generate_series(1, 1000000) id;
INSERT 0 1000000
Time: 6050.475 ms (00:06.050)
```

从上面可以看到，UUID v7 明显比 v4 快。此外，v7 相较于 v4 的另外一个特点就是有序性，如下所示：

```
postgres=# SELECT * FROM uuid_v4_test ORDER BY id LIMIT 10;
                  id                  |    info
--------------------------------------+------------
 00003562-1bb6-42da-a5be-070cfb42857d | data69864
 0000387e-d978-49a1-b474-f10f64044fc2 | data899555
 00003e2f-ea91-4bbd-b2c1-5d35116adfe8 | data907466
 00004402-c462-4fe8-95b7-9ceedfea30bd | data232215
 00005749-bb2e-4f50-bc5c-333baae135ba | data999604
 000057c9-cfd6-40b2-bb2b-550396fe9749 | data504685
 00006809-9e41-4efc-a778-5623f3ab055f | data13585
 00006ff6-16c1-4310-be3e-c8f6f5503382 | data764974
 000077af-84b3-4030-9b43-f04a10a1c142 | data1722
 00008614-ee29-4acd-84bd-6a41a7954fa5 | data705404
(10 rows)

Time: 1.428 ms
postgres=# SELECT * FROM uuid_v7_test ORDER BY id LIMIT 10;
                  id                  |  info
--------------------------------------+--------
 0193cfef-0d4b-7c36-84a4-7a4fc90defa7 | data1
 0193cfef-0d4c-7ac8-88ae-406920469dc6 | data2
 0193cfef-0d4c-7b48-87a0-cc302b835e35 | data3
 0193cfef-0d4c-7b6c-94f5-6ac9f10653d6 | data4
 0193cfef-0d4c-7b84-82d1-7c67e874240d | data5
 0193cfef-0d4c-7b9d-ba52-b6ce21d1b2b5 | data6
 0193cfef-0d4c-7bb2-84b4-f66a4bbeef0b | data7
 0193cfef-0d4c-7bc9-acd2-8eb1bbaaac89 | data8
 0193cfef-0d4c-7be5-bd9f-e2a34e5f61ae | data9
 0193cfef-0d4c-7c03-bf62-c7cdffafb8e5 | data10
(10 rows)

Time: 0.617 ms
```
从上面可以看出，UUID v4 生成的 UUID 是无序的，而 v7 版本则是有序的。不过 UUID v7 在连续生成时，存在一定的重复域，从上面的 UUID v7 可以看出，连续生成的 UUID v7 前面的 ` 0193cfef-0d4` 都是相同的。

```
postgres=# SELECT oid::regclass, pg_size_pretty(pg_relation_size(oid)) FROM pg_class WHERE relname ~ 'uuid_v.*_test_pkey';
        oid        | pg_size_pretty
-------------------+----------------
 uuid_v4_test_pkey | 38 MB
 uuid_v7_test_pkey | 30 MB
(2 rows)
```

UUID 作为主键，v7 相比与 v4 节省了约 20% 的空间。

## 参考

[1] https://tools.ietf.org/html/rfc4122
[2] https://postgr.es/m/CAAhFRxitJv%3DyoGnXUgeLB_O%2BM7J2BJAmb5jqAT9gZ3bij3uLDA%40mail.gmail.com

[RFC 4122]: https://tools.ietf.org/html/rfc4122
[78c5e141]: https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=78c5e141e9c139fc2ff36a220334e4aa25e1b0eb

<div class="just-for-fun">
笑林广记 - 搁浅

矮人乘舟出游，因搁浅，自起撑之。
失手坠水，水没过顶，矮人起而怒曰：“偏我搁浅搁在深处。”
</div>
