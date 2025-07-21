---
date: 2025-05-05 14:47:46 +0800
title: "PostgreSQL rewind 分析"
slug: pg_rewind
authors:
  - japinli
tags:
  - PostgreSQL
---

最近在使用 PostgreSQL 的物理复制时，遇到了关于 PostgreSQL rewind 操作相关的问题，本文梳理了 `pg_rewind` 的工作流程，并针对 rewind 期间可能遇到的问题进行整理。

<!-- more -->

我将 `pg_rewind` 整个流程大致分为三个阶段：

- **初始阶段：**根据 `source` 和 `target` 的 `ControlFileData` 来判断是否需要进行 `rewind` 操作。
- **准备阶段：**遍历 `source` 和 `target` 数据目录并建立文件的映射表、确定各文件的操作。
- **执行阶段：**根据**准备阶段**生成的文件映射表执行相应的操作，如拷贝数据块、删除文件等。

# 初始阶段

在初始阶段 `pg_rewind` 首先会根据 `--source-server` 和 `--source-pgdata` 来创建 [`rewind_source`](https://github.com/postgres/postgres/blob/REL_17_STABLE/src/bin/pg_rewind/rewind_source.h) 对象，它主要用于封装远程在线实例和本地实例的访问。

- `--source-server` - 通过 libpq 连接数据库，随后利用 `init_libpq_source()` 创建 `rewind_source` 对象。
- `--source-pgdata` - 通过 `init_local_source()` 创建 `rewind_source` 对象。

接下来，`pg_rewind` 解析 `source` 和 `target` 的 `ControlFileData` 并它们进行验证（CRC32 校验）。

{% note info no-icon %}

`pg_rewind` 执行时需要 `target` 数据目录是 `shut down` 或 `shut down in recovery` 状态。若是其他状态，并且没有指定 `--no-ensure-shutdown` 选项，那么它将以当用户模式启动强制恢复完成。

{% endnote %}

`pg_rewind` 能执行的前提是它们来自同一个实例，`sanityChecks()` 函数用于完成这类检查，包括数据库系统标识符、版本号是否匹配，`target` 是否开启数据校验（`data_checksums`）或 WAL 日志提示（`wal_log_hints`）。

{% note info no-icon %}

如果通过 `--source-pgdata` 指定 `source`，那么它还需要验证 `source` 数据目录也处于 `shut down` 或 `shut down in recovery` 状态。
{% endnote %}	

接着确定 `source` 和 `target` 的时间线，然后通过 `getTimelineHistory()` 获取 `source` 和 `target` 的时间线历史，并通过 `findCommonAncestorTimeline()` 查找其共同的祖先，从而找到分叉点（`divergerec`）。

{% note info no-icon %}

在 PostgreSQL 16 之前，`source` 和 `target` 的时间线通过 `ControlFileData` 中的 `checkPointCopy.ThisTimeLineID` 来确定，但是这可能会导致类似下面的错误：

```
requested timeline 11 does not contain minimum recovery point 5/D9000108 on timeline 9
```

这是由于在执行 `promote` 之后，立马进行 `rewind` 操作，这个时候可能 `promote` 的节点还没来得及执行 `CHECKPOINT`，从而导致错误。该问题在 [009eeee7468](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=009eeee746825) 中已经被修复（获取 `minRecoveryTLI` 和 `ThisTimeLineID` 中较大的作为其时间线）。

{% endnote %}

最后通过读取 `ControlFileData.checkPoint` 并结合 `ControlFileData.minRecoveryPoint` 来获取 `target` 结束的 WAL 日志位置（`target_wal_endrec`）。如果 `target_wal_endrec` 大于 `divergerec`，那么需要进行 `rewind` 操作。

# 准备阶段

该阶段主要负责收集 `source` 和 `target` 之间的文件差异、建立文件映射表并决定每个文件应该执行的操作类型，为后续的**执行阶段**提供依据。

该阶段从 `keepwal_init()` 开始，它负责创建一个哈希表用于跟踪 WAL 日志文件，确保在 `rewind` 期间，这些 WAL 日志不会被删除掉。

随后通过 `findLastCheckpoint()` 函数查找 `target` 上分叉点（`divergerec`）之前的检查点信息。`findLastCheckpoint()` 从 `divergerec` 开始从后向前查找检查点，并将这个过程中访问的 WAL 日志保存在 `keepwal_init()` 创建的哈希表中。

接着通过 `filehash_init()` 函数初始化文件映射表，遍历 `source` 和 `target` 数据目录下的所有文件填充文件映射表。`target` 的遍历通过 `traverse_datadir()` 来完成。

- `source` 如果是 `--source-pgdata`，则通过 `local_traverse_files()` 函数遍历数据目录。本质上是包裹 `traverse_datadir()` 函数。
- `source` 如果是 `--source-server`，则通过 `libpq_traverse_file()` 函数遍历数据目录。本质上是使用 SQL 语句获取文件列表，对应的 SQL 语句如下所示：

	``` sql
	WITH RECURSIVE files (path, filename, size, isdir) AS (
	  SELECT '' AS path, filename, size, isdir FROM
	  (SELECT pg_ls_dir('.', true, false) AS filename) AS fn,
	        pg_stat_file(fn.filename, true) AS this
	  UNION ALL
	  SELECT parent.path || parent.filename || '/' AS path,
	         fn, this.size, this.isdir
	  FROM files AS parent,
	       pg_ls_dir(parent.path || parent.filename, true, false) AS fn,
	       pg_stat_file(parent.path || parent.filename || '/' || fn, true) AS this
	       WHERE parent.isdir = 't'
	)
	SELECT path || filename, size, isdir,
	       pg_tablespace_location(pg_tablespace.oid) AS link_target
	FROM files
	LEFT OUTER JOIN pg_tablespace ON files.path = 'pg_tblspc/'
	                              AND oid::text = files.filename
	;
	```

在遍历数据目录的过程中，`source` 通过 `process_source_file()` 函数处理文件并将其加入到文件映射表中，而 `target` 则通过 `process_target_file()` 函数进行类似的操作。

当初始化 `source` 和 `target` 数据目录中的文件映射表之后，接着通过 `extractPageMap()` 函数从分叉点之前的最后一个检查点读取目标 WAL，以提取分叉后在 `target` 数据目录上修改的所有页面。其核心在 `extractPageInfo()` 函数中，它负责提取 WAL 修改的数据块信息。

最后，通过 `decide_file_actions()` 函数决定每个文件对应的操作类型，包含如下操作：

- `FILE_ACTION_UNDECIDED` - 尚未确定
- `FILE_ACTION_CREATE` - 创建本地目录或符号链接
- `FILE_ACTION_COPY` - 拷贝整个文件，如果存在则覆盖原文件
- `FILE_ACTION_COPY_TAIL` - 负责文件末位部分
- `FILE_ACTION_NONE` - 不采取任何操作（可能仍会根据解析的 WAL 复制修改过的块）
- `FILE_ACTION_TRUNCATE` - 截断本地文件到指定大小
- `FILE_ACTION_REMOVE` - 删除本地文件/目录/链接

# 执行阶段

现在我们已经从 `source` 和 `target` 收集了我们需要的所有信息，并且我们准备开始修改 `target` 数据目录。

{% note warning no-icon %}

这是一条不归路，一旦开始复制，就无法回头了！
{% endnote %}

最后的执行阶段是通过 `perform_rewind()` 函数完成的。

1. 遍历 `filemap->nentries` 数组，并从 `source` 数据目录中获取数据。
2. 如果该文件在 `target` 数据目录中的 WAL 日志中有修改数据块，通过 `libpq_queue_fetch_range()` 或 `local_queue_fetch_range()` 获取相应的数据块。
3. 根据 `entry->action` 执行相应的动作，如截断文件 `truncate_target_file()`，删除文件 `remove_target()` 等。
4. 结束遍历，调用 `local_finish_fetch()` 或 `libpq_finish_fetch()` 获取队列中的所有数据获取。
5. 从 `source` 中获取 `pg_control` 文件，并更新。

# 问题

从上面的实现可以得知，PostgreSQL 在 rewind 期间除了数据文件和忽略的文件外，都会完全的拷贝到从节点，因此，如果采用默认的 `log_directory` （即 `$PGDATA/log`），那么在 rewind 期间也会将这些日志同步到从节点。为了避免这个问题，我们可以将其从 `$PGDATA` 目录中移除。

此外，在 rewind 期间，PostgreSQL 同样会将 `$PGDATA/pg_wal` 下面的内容拷贝到从节点，但实际情况下，我们只需要[拷贝发生分叉之后的日志即可](https://www.postgresql.org/message-id/181b4c6fa9c.b8b725681941212.7547232617810891479%40viggy28.dev)。

# 参考

[1] https://thebuild.com/blog/2018/07/18/pg_rewind-and-checkpoints-caution/
[2] https://www.postgresql.org/message-id/9f568c97-87fe-a716-bd39-65299b8a60f4%40iki.fi
[3] https://www.postgresql.org/message-id/181b4c6fa9c.b8b725681941212.7547232617810891479%40viggy28.dev

<!--
``` c
void (*traverse_files) (struct rewind_source *,
                        process_file_callback_t callback);
```

- 遍历 `source` 数据目录中的所有文件，并在每个文件上调用 `callback` 回调函数。

``` c
char *(*fetch_file) (struct rewind_source *,
                     const char *path,
                     size_t *filesize);
```

- 将单个文件提取到已分配内存的缓冲区中。文件大小由 `*filesize` 返回。返回的缓冲区始终以零结尾，这对于文本文件来说非常方便。

``` c
void (*queue_fetch_range) (struct rewind_source *,
                           const char *path,
                           off_t offset,
                           size_t len);
```

- 请求从 `source` 数据目录获取由偏移量和长度指定的文件（部分内容），并将其写入到 `target` 数据目录中相应的文件中。它可能会将请求放入队列中，并在后续执行。调用 `finish_fetch()` 刷新队列并执行所有请求。

``` c
void (*queue_fetch_file) (struct rewind_source *,
                          const char *path,
                          size_t len);
```

- 与 `queue_fetch_range()` 类似，但请求从 `source` 数据目录替换 `target` 数据目录整个文件。

``` c
void (*finish_fetch) (struct rewind_source *);
```

- 执行 `queue_fetch_range()` 排队的所有请求。

``` c
XLogRecPtr (*get_current_wal_insert_lsn) (struct rewind_source *);
```

- 获取 `source` 数据目录中当前的 WAL 插入位置。

``` c
void (*destroy) (struct rewind_source *);
```

- 释放 `rewind_source` 对象。

-->
