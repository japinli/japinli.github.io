---
title: PostgreSQL UUID 类型
date: 2024-12-16 21:59:26 +0800
categories: 数据库
tags:
  - PostgreSQL
---

UUID 代表通用唯一标识符，在 [RFC 4122][] 中定义。它是一个 128 位数字，通常以十六进制书写，并用破折号分成五组。典型的 UUID 值如下所示：

	0193cfc2-5f8c-7f75-87ce-d26c47161eb5

在 [RFC 9562][] 中给出了 UUID v7 的定义，目前它还只是一个建议标准。在 [78c5e141][] 之前，PostgreSQL 只提供了 v4 版本，提交 [78c5e141][] 引入了 v7 版本的支持。不出意外的话，该功能将在 PostgreSQL 18 中发布。

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

## UUID 格式

UUID 的大小为 16 个字节（128 位）；其中变体位（variant）和版本位（version）一起确定了更精细的结构。

变体字段决定了 UUID 的布局，即 UUID 中所有其他位的解释取决于变体字段中位的设置。变体字段由 UUID 的第 8 个字节的最高有效位（数量可变）组成。

下表列出了变体字段的内容，其中字母 `x` 表示不关心的值。

| MSB0 | MSB1 | MSB2 | MSB3 | Variant  | Description                                            |
|------|------|------|------|----------|--------------------------------------------------------|
| 0    | x    | x    | x    | 1-7      | 保留。网络计算系统 (NCS) 向后兼容，并且包含 Nil UUID。 |
| 1    | 0    | x    | x    | 8-9, A-B | [RFC 9562][] 中定义。                                  |
| 1    | 1    | 0    | x    | C-D      | 保留。微软公司向后兼容。                               |
| 1    | 1    | 1    | x    | E-F      | 为将来的定义保留，并包括最大 UUID。                    |

版本号位于 UUID 的第 6 个字节的最高有效 4 位（UUID 的第 48 位至 51 位）。

下表列出了 [RFC 9562][] 中指定的 UUID 变体 `10xx` 的所有版本。

| MSB0 | MSB1 | MSB2 | MSB3 | Version | Description                                            |
|------|------|------|------|---------|--------------------------------------------------------|
| 0    | 0    | 0    | 0    | 0       | 未使用。                                               |
| 0    | 0    | 0    | 1    | 1       | [RFC 9562][] 中指定的基于公历时间的版本。              |
| 0    | 0    | 1    | 0    | 2       | 为 DCE 安全版本保留，带有嵌入式 POSIX UUID。           |
| 0    | 0    | 1    | 1    | 3       | [RFC 9562][] 中指定的使用 MD5 算法的基于名称的版本。   |
| 0    | 1    | 0    | 0    | 4       | [RFC 9562][] 中指定的随机或伪随机生成的版本。          |
| 0    | 1    | 0    | 1    | 5       | [RFC 9562][] 中指定的使用 SHA-1 算法的基于名称的版本。 |
| 0    | 1    | 1    | 0    | 6       | [RFC 9562][] 中指定的重新排序的基于公历时间的版本。    |
| 0    | 1    | 1    | 1    | 7       | [RFC 9562][] 中指定的基于 Unix Epoch 时间的版本。      |
| 1    | 0    | 0    | 0    | 8       | [RFC 9562][] 中指定的自定义 UUID 格式保留。            |
| 1    | 0    | 0    | 1    | 9       | 保留以供将来定义。                                     |
| 1    | 0    | 1    | 0    | 10      | 保留以供将来定义。                                     |
| 1    | 0    | 1    | 1    | 11      | 保留以供将来定义。                                     |
| 1    | 1    | 0    | 0    | 12      | 保留以供将来定义。                                     |
| 1    | 1    | 0    | 1    | 13      | 保留以供将来定义。                                     |
| 1    | 1    | 1    | 0    | 14      | 保留以供将来定义。                                     |
| 1    | 1    | 1    | 1    | 15      | 保留以供将来定义。                                     |


从上面可以看出，目前只有 v1，v6 和 v7 是基于时间的 UUID。本文中我们主要来看看 v4 和 v7 的布局。

{% asset_img uuidv4.png %}

上图是 UUID v4 的字段及其布局。

- **random_a:** 48 位随机数据，占用 0 到 47 位。
- **ver:** 4 位版本字段，占用 48 到 51 位，在 UUID v4 中被设置位 0b0100 (4)。
- **random_b:** 12 位随机数据，占用 52 到 63 位。
- **var:** 2 位变体字段，占用 64 到 65 位，设置位 0b10。
- **random_c:** 最后 62 位随机数据，占用 66 到 127 位。

**注意：**随机数据应具有不可猜测性，即其实现应使用加密安全的伪随机数生成器 (CSPRNG) 来提供难以预测（“不可猜测”）且发生冲突的可能性低（“唯一”）的值。例外情况是执行环境中没有合适的 CSPRNG。注意确保在状态更改（例如进程分叉）时正确重新播种 CSPRNG 状态，以确保正确的 CSPRNG 操作。

{% asset_img uuidv7.png %}

上图是 UUID v7 的字段及其布局。

- **unix_ts_ms:** 48 位以毫秒位单位的 Unix 纪元时间戳，采用大端无符号数，占用 0 到 47 位。
- **ver:** 4 位版本字段，占用 48 到 51 位，在 UUID v4 中被设置位 0b0111 (7)。
- **rand_a:** 12 位随机数据，占用 52 到 63 位。
- **var:** 2 位变体字段，占用 64 到 65 位，设置位 0b10。
- **rand_b:** 最后 62 位随机数据，占用 66 到 127 位。

**注意：** UUID v7 版本中的随机数需要提供唯一性，并且可选的提供额外的单调性。

## 总结

[RFC 9562][] 针对 UUID 提出了 8 个不同的版本，其中 UUID v2 是适用于 DEC 安全，没有给出定义，其他 7 中都给出了定义，如下图所示：

{% asset_img uuid-summary.png %}

## 参考

[1] https://tools.ietf.org/html/rfc4122
[2] https://datatracker.ietf.org/doc/rfc9562/
[2] https://postgr.es/m/CAAhFRxitJv%3DyoGnXUgeLB_O%2BM7J2BJAmb5jqAT9gZ3bij3uLDA%40mail.gmail.com

[RFC 4122]: https://tools.ietf.org/html/rfc4122
[RFC 9562]: https://datatracker.ietf.org/doc/rfc9562/
[78c5e141]: https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=78c5e141e9c139fc2ff36a220334e4aa25e1b0eb

<div class="just-for-fun">
笑林广记 - 搁浅

矮人乘舟出游，因搁浅，自起撑之。
失手坠水，水没过顶，矮人起而怒曰：“偏我搁浅搁在深处。”
</div>
