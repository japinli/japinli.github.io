---
title: "PostgreSQL 查询冲突"
date: 2024-05-18 15:48:33 +0800
category: 数据库
tags:
  - PostgreSQL
---

在 PostgreSQL 主从架构下，从节点可以提供一个只读副本用于缓解主节点的压力，但是这也可能带来其他问题，例如，您可能会在应用或者数据库日志中看到如下错误信息。

{% codeblock line_number:false %}
ERROR:  canceling statement due to conflict with recovery
DETAIL:  User query might have needed to see row versions that must be removed.
{% endcodeblock %}

<!--more-->

## 模拟查询冲突

首先，搭建 PostgreSQL 主从集群，您可以参考 [PostgreSQL 主从集群](https://www.postgresql.org/docs/current/replication.html)文档。此外，您也可以参考我写的 {% post_link postgresql-stream-replication "PostgreSQL 12 流复制配置" %}。

接着，我们可以使用图所示的方式来模拟查询冲突的情况。

{% asset_img canceling-statement.jpg %}

上图中左边为主节点，右边为从节点。在从节点上面可以看到当前 `max_standby_streaming_delay` 的值为 `30s`。我们模拟一个需要执行 `40s` 的查询。随后在主节点上对表进行更新并执行 `VACUUM` 操作。最后，可以在从节点上观察到该查询由于冲突被取消了。

关于 SQL 需要注意的一个重要方面是其数据视图的一致性。在整个执行过程中，SQL 语句始终会看到一致的数据快照，即使其他地方同时发生更改也是如此。

虽然主节点的 `UPDATE` 本身并不一定会破坏从节点副本的行为，但后续操作会破坏从节点副本的行为（即后续的 `VACUUM` 操作，需要注意的时，这也可能时由于 autovacuum 自动触发的）。

`VACUUM` 的作用是通过删除旧版本的行来回收空间，它也将记录在 WAL 日志中，并发送给从节点。由于主节点的 `VACUUM` 操作并不知道从节点副本上正在进行的 `SELECT` 语句，从而删除了从节点副本仍然需要的行。此时便导致所谓的查询冲突（复制冲突）。

## 分析

通过上面的示例，我们了解了查询冲突的基本情况，现在，我们来详细看看数据库日志的处理。为了方便测试观察，我们将 `max_standby_streaming_delay` 调整为 `5min`。

{% codeblock Replica line_number:false %}
ALTER SYSTEM SET max_standby_streaming_delay TO '5min';
SELECT pg_reload_conf();
{% endcodeblock %}

在主节点上创建表，并插入数据，如下所示。

{% codeblock Primary line_number:false %}
[local]:3682393 postgres=# CREATE TABLE t01 (id int, info text);
CREATE TABLE
[local]:3682393 postgres=# INSERT INTO t01 VALUES (1, 'hello'), (2, 'world');
INSERT 0 2
{% endcodeblock %}

紧接着，我们可以在从节点上查看数据已经由主节点同步到从节点。

{% codeblock Replica line_number:false %}
[local]:3823209 postgres=# SELECT * FROM t01;
 id | info
----+-------
  1 | hello
  2 | world
(2 rows)
{% endcodeblock %}

我们也可以在主节点的观察流复制状态来验证这一点。

{% codeblock Primary line_number:false %}
[local]:3682393 postgres=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 3823096
usesysid         | 16410
usename          | replicator
application_name | walreceiver
client_addr      |
client_hostname  |
client_port      | -1
backend_start    | 2024-05-17 14:36:14.532439+08
backend_xmin     |
state            | streaming
sent_lsn         | 0/3026368
write_lsn        | 0/3026368
flush_lsn        | 0/3026368
replay_lsn       | 0/3026368
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2024-05-17 14:49:32.96194+08
{% endcodeblock %}

到目前为止，一切都还正常。我们在从节点上模拟一个长查询，之后在主节点上对表进行更新、`VACUUM`，并观察流复制状态。

{% codeblock Replica line_number:false %}
[local]:3823209 postgres=# SELECT pg_sleep(120), * FROM t01;
{% endcodeblock %}

此时，从节点上查询将长时间运行，我们先观察主节点上更新操作的日志信息。

{% codeblock Primary line_number:false %}
[local]:3682393 postgres=# UPDATE t01 SET info = 'Hello' WHERE id = 1;
UPDATE 1

...

[local]:3682393 postgres=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 3823096
usesysid         | 16410
usename          | replicator
application_name | walreceiver
client_addr      |
client_hostname  |
client_port      | -1
backend_start    | 2024-05-17 14:36:14.532439+08
backend_xmin     |
state            | streaming
sent_lsn         | 0/30265B0
write_lsn        | 0/30265B0
flush_lsn        | 0/30265B0
replay_lsn       | 0/30265B0
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2024-05-17 14:53:10.881918+08
{% endcodeblock %}

您可能需要反复执行上面的查询来观察复制状态，您会发现即便从节点的查询正在执行，更新操作也可以正常的在从节点回放 WAL 日志。

当我们在主节点上执行 `VACUUM` 后，在观察流复制状态。

{% codeblock Primary line_number:false %}
[local]:3682393 postgres=# VACUUM t01;
VACUUM

...

[local]:3682393 postgres=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 3823096
usesysid         | 16410
usename          | replicator
application_name | walreceiver
client_addr      |
client_hostname  |
client_port      | -1
backend_start    | 2024-05-17 14:36:14.532439+08
backend_xmin     |
state            | streaming
sent_lsn         | 0/3029FE0
write_lsn        | 0/3029FE0
flush_lsn        | 0/3029FE0
replay_lsn       | 0/30265B0
write_lag        | 00:00:00.000286
flush_lag        | 00:00:00.001455
replay_lag       | 00:02:00.239551
sync_priority    | 0
sync_state       | async
reply_time       | 2024-05-17 14:55:48.70995+08
{% endcodeblock %}

注意到 `flush_lsn` 的位置为 `0/3029FE0`，而 `replay_lsn` 的位置为 `0/30265B0`，略微延后（通过 `replay_lay` 也可以看到），当从节点的查询执行完成之后，主节点便会继续回放 WAL 日志。

{% codeblock Replica line_number:false %}
 pg_sleep | id | info
----------+----+-------
          |  1 | hello
          |  2 | world
(2 rows)
{% endcodeblock %}

{% codeblock Primary line_number:false %}
[local]:3682393 postgres=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 3823096
usesysid         | 16410
usename          | replicator
application_name | walreceiver
client_addr      |
client_hostname  |
client_port      | -1
backend_start    | 2024-05-17 14:36:14.532439+08
backend_xmin     |
state            | streaming
sent_lsn         | 0/302A100
write_lsn        | 0/302A100
flush_lsn        | 0/302A100
replay_lsn       | 0/302A100
write_lag        | 00:00:00.000211
flush_lag        | 00:00:00.001392
replay_lag       | 00:00:22.49792
sync_priority    | 0
sync_state       | async
reply_time       | 2024-05-17 14:56:34.52255+08
{% endcodeblock %}

为什么更新操作的日志可以回放，而 `VACUUM` 操作的日志不能立即回放呢？这是由于 PostgreSQL 的 MVCC 机制导致的，PostgreSQL 的更新和删除是非原地操作（即删除是标记删除，更新则是删除 + 插入），读写不冲突，因此，更新删除操作不会影响数据的读取。然而 `VACUUM` 则不同，它将对数据进行整理并回收，在上述情况下，由于主节点没有任何查询访问到 `VACUUM` 需要回收的空间，因此它可以正常回收；但是由于从节点当前正在读取就版本数据，而此时接收到了来自主节点的空间回收日志，由于从节点的查询还未终止，因此，从节点的日志回放被阻塞了。

为了避免从节点上的长查询导致 `VACUMM` 不能及时得到处理，PostgreSQL 提供了 `max_standby_streaming_delay` 参数来控制诸如 `VACUUM` 这类日志的回放时机。当日志回放超过 `max_standby_streaming_delay` 指定的时间时，PostgreSQL 将强制断开从节点的查询，从而回放日志，这时候您就会看到文章开始提到的错误信息。

## 解决办法

### 调整 `max_standby_streaming_delay` 参数

增加从节点 `max_standby_streaming_delay` 参数的值，增加该参数的值可以让从节点在决定取消查询之前有更多时间来解决冲突。但是，这也可能会增加复制延迟，因此这是一个权衡。该参数可动态调整，这意味着更改无需重新启动服务器即可生效。

### 调整 `max_standby_archive_delay` 参数

`max_standby_archive_delay` 服务器参数指定服务器在读取存档的 WAL 数据时允许的最大延迟。如果 PostgreSQL 服务器实例从节点由流模式切换到基于文件的日志传送（尽管很少见），则调整此值可以帮助解决查询取消问题。

### 监控和优化查询

查看针对只读节点上运行的查询的类型和频率。长时间运行或复杂的查询可能更容易发生冲突。以不同的方式优化或安排它们会有所帮助。

### 低峰期执行查询

考虑在低峰时段运行繁重或长时间运行的查询，以减少发生冲突的可能性。

### 启用 `hot_standby_feedback` 参数

考虑在从节点上将 `hot_standby_feedback` 设置为 `on`。启用后，它会通知主节点有关当前从节点正在执行的查询的信息。这可以防止主节点数据库删除从节点数据库仍需要的行，从而降低复制冲突的可能性。该参数可动态调整，这意味着更改无需重新启动服务器即可生效。

{% note warning no-icon %}
启用 `hot_standby_feedback` 可能会导致以下潜在问题：

* 此设置将阻止主数据库上进行一些必要的清理操作，从而可能导致表膨胀（由于未清理旧行版本而增加磁盘空间使用量）。
* 定期监控主数据库的磁盘空间和表大小至关重要。
* 如果出现问题，请准备好手动管理潜在的表膨胀。
{% endnote %}


## 参考

[1] [Canceling statement due to conflict with recovery](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/troubleshoot-canceling-statement-due-to-conflict-with-recovery)
[2] https://www.postgresql.org/docs/current/runtime-config-replication.html
