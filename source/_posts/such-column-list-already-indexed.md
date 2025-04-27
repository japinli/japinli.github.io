---
title: "Oracle 属性列已被索引"
date: 2025-04-24 12:00:00 +0800
categories: 数据库
tags:
  - Oracle
  - PostgreSQL
---

在数据库管理中，索引是提升查询性能的关键，但不当的索引设计可能导致存储浪费和性能下降。在 Oracle 数据库中，当尝试对已索引的列列表（以相同顺序）再次创建索引时，会触发 ORA-01408: such column list already indexed 错误，这一特性有效避免冗余索引。根据 [Oracle 官方文档](https://docs.oracle.com/en/error-help/db/ora-01408/?r=19c)，此功能至少在 Oracle 19c 中已明确实现。相比之下，PostgreSQL 没有类似的内置机制，用户需手动检查和管理索引以避免冗余。本文将深入探讨 Oracle 和 PostgreSQL 在处理“属性列已被索引”场景中的实现、优缺点及实践建议。

<!-- more -->

## Oracle 中的索引管理

在 Oracle 数据库中，当用户尝试为表的主键、唯一约束或其他场景创建索引时，如果目标列列表（以相同顺序）已存在于某个索引中，Oracle 会抛出 `ORA-01408` 错误，提示无需创建新索引。这一特性通常出现在以下场景：

- 创建主键或唯一约束时，Oracle 自动尝试为约束列创建索引，但发现已有索引覆盖。
- 手动创建索引时，目标列列表与现有索引完全匹配。

### 工作原理

Oracle 的索引管理依赖 B+树索引（默认）、位图索引或分区索引等结构。当创建约束或索引时，数据库会检查 `USER_IND_COLUMNS` 和 `USER_CONS_COLUMNS` 视图，确认是否存在匹配的索引（包括列顺序和索引类型）。如果匹配，Oracle 会复用现有索引，避免冗余。

如下示例所示：

``` sql
CREATE TABLE employee (
  id int,
  name varchar(50),
  dept_id int
);

-- 创建唯一索引
CREATE UNIQUE INDEX employee_id_idx ON employee(id);

-- 尝试添加索引（将触发 ORA-01408 错误）
CREATE INDEX employee_id_idx1 ON employee(id);
```

执行上述 `CREATE INDEX` 语句时，Oracle 检测到 `employee_id_idx` 已覆盖 `id` 列，因此复用该索引，而不是创建新索引，如下所示：

```
CREATE INDEX employee_id_idx1 ON employee(id)
                                          *
ERROR at line 1:
ORA-01408: such column list already indexed
```


Oracle 中的索引管理具有如下优点：

- **避免冗余索引：**
	- Oracle 检测到现有索引（如 B+树索引）已覆盖约束列（如主键或唯一约束的列），无需创建新索引，节省存储空间。
	- 减少索引维护开销，提升写操作性能（如 `INSERT`、`UPDATE`、`DELETE`）。

- **查询性能优化：**
  - 复用索引可加速 `SELECT` 查询（如 `WHERE`、`JOIN`、`ORDER BY`），特别是高选择性列。
  - 如果索引是唯一索引或覆盖索引，可进一步提升查询效率（如跳过表访问）。

- **约束验证高效：**
  - 主键或唯一约束可直接利用现有索引验证数据的唯一性，无需额外索引结构，降低维护成本。

- **灵活性：**
  - 允许使用非唯一索引支持唯一约束（Oracle 特性），适合需要延迟唯一性检查的场景（如批量插入）。

当然，任何事务都具有两面性，它具有以下缺点：

- **索引复用风险：**
  - 如果现有索引并非为约束优化设计（如非唯一索引支持唯一约束），可能导致性能问题。例如，非唯一索引可能比唯一索引稍慢。
  - 索引列顺序或包含的额外列可能不完全匹配查询需求，导致部分查询无法充分利用索引。

- **潜在的维护复杂性：**
  - 现有索引可能被多个约束或查询依赖，修改或删除索引（如 `DROP INDEX`）可能影响其他功能，需谨慎操作。
  - 如果索引碎片化或未优化（如未定期重建），可能降低查询和约束验证性能。

- **空间与性能权衡：**
  - 虽然避免了冗余索引，但现有索引可能包含额外列（如复合索引），占用更多空间或增加维护成本。
  - 如果索引设计不佳（如低选择性列），可能降低查询效率。

- **隐式依赖问题：**
  - 开发人员可能未意识到约束依赖现有索引，误删除索引可能导致约束失效或性能下降。
  - Oracle 不强制要求索引与约束绑定，管理不当可能引发问题。

### 实践建议

- **检查索引使用情况：**使用以下 SQL 查询确认索引与约束的映射：
	``` sql
	SELECT
	  i.index_name,
	  i.table_name,
	  c.column_name
	FROM
	  user_indexes i JOIN user_ind_columns c ON i.index_name = c.index_name
	WHERE
	  i.table_name = 'EMPLOYEE'
	;
	```
- **监控索引性能：**定期分析索引碎片（`ANALYZE INDEX ... VALIDATE STRUCTURE`）并重建。
- **文档化依赖：**记录约束与索引的关系，避免误操作。
- **优化索引设计：**确保复用索引的列顺序和选择性适合查询需求。

## PostgreSQL 中的索引管理

与 Oracle 不同，PostgreSQL 没有内置机制检测和阻止冗余索引的创建。如果用户为相同的列列表（以相同顺序）创建多个索引，PostgreSQL 会允许操作，导致潜在的存储浪费和性能问题。因此，DBA 需手动检查和管理索引。

### 工作原理

PostgreSQL 的索引基于 B+树（默认）、GiST、GIN、BRIN 等结构。创建索引时，数据库不会检查是否已有相同列列表的索引。用户需通过系统视图（如 `pg_index` 和 `pg_attribute`）分析索引配置。

``` sql
CREATE TABLE employee (
  id int,
  name varchar(50),
  dept_id int
);

-- 创建唯一索引
CREATE UNIQUE INDEX employee_id_idx ON employee(id);

-- 创建重复索引（PostgreSQL 允许此操作）
CREATE INDEX employee_id_idx1 ON employee(id);
```

上述操作在 PostgreSQL 中不会报错，但会创建冗余索引，浪费空间并增加维护成本。

### 检查冗余索引

我们可以通过下面的 SQL 语句来检查冗余索引：

``` sql
WITH dup_idx AS (
  SELECT
    indrelid AS duprelid,
    indkey AS dupkey,
    array_agg(indexrelid::regclass) AS dupnames,
    count(1) AS dupnum
  FROM
    pg_index
  GROUP BY
    indrelid, indkey
)
SELECT
  a.attrelid::regclass AS table_name,
  d.dupnum AS index_number,
  d.dupnames AS index_names,
  array_agg(a.attname ORDER BY idx.ord) AS index_columns
FROM
  dup_idx d JOIN pg_attribute a ON a.attrelid = d.duprelid
  CROSS JOIN LATERAL unnest(d.dupkey::int[]) WITH ordinality AS idx(attnum, ord)
WHERE
  a.attnum = idx.attnum AND d.dupnum > 1
GROUP BY
  a.attrelid, d.dupnum, d.dupnames
;
```

此查询会列出表名、冗余索引数量、索引名称及涉及的列，帮助 DBA 识别并清理冗余索引。

下面的查询是根据 [PostgreSQL 社区提供的版本](https://wiki.postgresql.org/wiki/Index_Maintenance#Duplicate_indexes)进行修改的：

``` sql
WITH dup_idx AS (
  SELECT
    indrelid AS duprelid,
    array_agg(indexrelid::regclass) AS dupindnames,
    count(1) AS dupnum,
    array[indrelid::text,
          indclass::text,
          indkey::text,
          coalesce(indexprs::text, ''),
          coalesce(indpred::text, '')] AS dupkey
  FROM
    pg_index
  GROUP BY
    duprelid, dupkey
  HAVING
    count(1) > 1
)
SELECT
  a.attrelid::regclass AS table_name,
  d.dupnum AS index_number,
  d.dupindnames AS index_names,
  array_agg(a.attname ORDER BY idx.ord) AS index_columns
FROM
  dup_idx d JOIN pg_attribute a ON a.attrelid = d.duprelid
  CROSS JOIN LATERAL
    unnest(string_to_array(d.dupkey[3], ' ')::int[]) WITH ordinality AS idx(attnum, ord)
WHERE
  a.attnum = idx.attnum
GROUP BY
  a.attrelid, d.dupnum, d.dupindnames
;
```

第一个查询只是单纯的从索引所包含的属性列来考虑的，并没有考虑到诸如 `INCLUDE`，表达式索引等情况。例如：

``` sql
CREATE INDEX ON employee(id) INCLUDE (dept_id);
CREATE INDEX ON employee(id, dept_id);
```

上述两个索引在第一个查询中会被视为重复索引，单实际上他们并不是完全重复的；第二个查询则可以区分此类索引。

如果您只想知道有哪些索引是重复的，您可以使用下面的语句：

``` sql
SELECT
  relname AS table_name,
  pg_size_pretty(sum(pg_relation_size(indname))::bigint) as size,
  array_agg(indname) AS index_names,
  count(1) AS dupnum
FROM
  (
    SELECT
      indrelid::regclass AS relname,
      indexrelid::regclass AS indname,
      (indrelid::text || E'\n' ||
       indclass::text || E'\n' ||
       indkey::text || E'\n' ||
       coalesce(indexprs::text, '') || E'\n' ||
       coalesce(indpred::text, '') || E'\n') AS key
     FROM
       pg_index
  ) sub
GROUP BY
  relname, key
HAVING
  count(1) > 1
;
```

PostgreSQL 中的索引管理具有以下优点：

- **灵活性：**
  - PostgreSQL 允许用户完全控制索引创建，适合复杂场景（如部分索引或表达式索引）。
  - 无强制检查，简化某些临时索引的创建。

- **透明性：**
  - 索引管理完全由用户控制，避免隐式依赖问题。

当然，PostgreSQL 中的索引管理同样具有缺点：

- **冗余索引风险：**
  - 缺乏自动检测机制，易导致重复索引，增加存储和维护开销。
  - 写操作性能可能因冗余索引而下降。

- **管理负担：**
  - DBA 需手动运行查询（如上述 SQL）检查冗余索引，增加了维护工作量。
  - 清理冗余索引需谨慎分析，避免影响查询或约束。

- **性能隐患：**
  - 冗余索引可能导致优化器选择次优索引，降低查询性能。

### 实践建议

- **定期检查冗余索引：**使用上述 SQL 查询监控索引状态，删除不必要的索引（`DROP INDEX`）。

- **规范化索引设计：**在创建索引前，检查现有索引（`SELECT * FROM pg_indexes WHERE tablename = 'employee'`）。

- **自动化脚本：**编写脚本定期检测冗余索引，集成到运维流程。

## 总结

Oracle 的“such column list already indexed”特性通过自动检测和复用索引，有效避免冗余，降低存储和维护成本，但需注意复用索引的性能和隐式依赖问题。PostgreSQL 则将索引管理完全交给用户，提供了更大灵活性，但增加了冗余索引的风险和维护负担。无论使用哪种数据库，DBA 都应通过查询分析、定期优化和文档化管理，确保索引设计高效且符合业务需求。

## 参考

[1] https://docs.oracle.com/en/error-help/db/ora-01408/?r=19c
[2] https://www.postgresql.org/docs/current/indexes.html
[3] https://www.postgresql.org/docs/current/catalogs.html
[4] https://wiki.postgresql.org/wiki/Index_Maintenance#Duplicate_indexes

<div class="just-for-fun">
笑林广记 - 被打

二瞽者同行。曰：“世上惟瞽者最好，有眼人终日奔忙，农家更甚。怎如得我们心上清闲。”
众农夫窃听之，乃伪为官过，谓其失于回避，以锄把各打一顿而呵之去。随复窃听之。
一瞽者曰：“毕竟是瞽者好，若是有眼人，打了还要问罪哩！”
</div>
