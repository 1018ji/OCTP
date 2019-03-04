# Index Merge Optimization (索引合并优化)

The Index Merge access method retrieves rows with multiple range scans and merges their results into one. This access method merges index scans from a single table only, not scans across multiple tables. The merge can produce unions, intersections, or unions-of-intersections of its underlying scans.

索引合并访问方法检索具有多个范围扫描的行，并将其结果合并为一个。此访问方法仅合并来自单个表的索引扫描行，而不是跨多个表扫描行。 合并可以生成并集、交集或者差集。

Example queries for which Index Merge may be used:

可用于索引合并的查询示例：

```
SELECT * FROM tbl_name WHERE key1 = 10 OR key2 = 20;

SELECT * FROM tbl_name
WHERE (key1 = 10 OR key2 = 20) AND non_key = 30;

SELECT * FROM t1, t2
WHERE (t1.key1 IN (1,2) OR t1.key2 LIKE 'value%')
    AND t2.key1 = t1.some_col;

SELECT * FROM t1, t2
WHERE t1.key1 = 1
    AND (t2.key1 = t1.some_col OR t2.key2 = t1.some_col2);
```

**Note 注意**

The Index Merge optimization algorithm has the following known limitations:

索引合并优化算法具有以下已知限制：

* If your query has a complex WHERE clause with deep AND/OR nesting and MySQL does not choose the optimal plan, try distributing terms using the following identity transformations:

如果您的查询有一个复杂的 `WHERE` 子句，其中包含深层次 `AND/OR`嵌套，而 MySQL 没有选择最佳计划，请尝试使用以下转换方法：

```
(x AND y) OR z => (x OR z) AND (y OR z)
(x OR y) AND z => (x AND z) OR (y AND z)
```

* Index Merge is not applicable to full-text indexes.

索引合并不适用于全文索引

In EXPLAIN output, the Index Merge method appears as index_merge in the type column. In this case, the key column contains a list of indexes used, and key_len contains a list of the longest key parts for those indexes.

在 `EXPLAIN` 输出中，索引合并方法在 `type` 列中显示为index_merge。在这种情况下，`key` 列包含使用的索引列表，`key_len` 包含这些索引的最长使用的 `key` 部分。

The Index Merge access method has several algorithms, which are displayed in the Extra field of EXPLAIN output:

索引合并访问方法有几种算法，它们显示在 `EXPLAIN` 输出的 `Extra` 字段中：

* Using intersect(...)
* Using union(...)
* Using sort_union(...)

The following sections describe these algorithms in greater detail. The optimizer chooses between different possible Index Merge algorithms and other access methods based on cost estimates of the various available options.

以下部分更详细地描述了这些算法。优化器根据各种可用选项的成本估算在不同的索引合并算法和其他访问方法之间进行选择。

* Index Merge Intersection Access Algorithm
* Index Merge Union Access Algorithm
* Index Merge Sort-Union Access Algorithm
* Influencing Index Merge Optimization

## Index Merge Intersection Access Algorithm 索引合并交集访问算法

This access algorithm is applicable when a WHERE clause is converted to several range conditions on different keys combined with AND, and each condition is one of the following:

当一个 `WHERE` 子句可被转换为多个范围条件时并使用 `AND` 组合，并且每个条件都是下列条件之一：

* An N-part expression of this form, where the index has exactly N parts (that is, all index parts are covered):

此形式的 N 部分表达式，其中索引具有正好 N 个部分（即所有索引部分都被覆盖）

```
key_part1 = const1 AND key_part2 = const2 ... AND key_partN = constN
```

* Any range condition over the primary key of an InnoDB table.

InnoDB 表的主键上的任何范围条件。

Examples:

例如：

```
SELECT * FROM innodb_table
WHERE primary_key < 10 AND key_col1 = 20;

SELECT * FROM tbl_name
WHERE key1_part1 = 1 AND key1_part2 = 2 AND key2 = 2;
```

The Index Merge intersection algorithm performs simultaneous scans on all used indexes and produces the intersection of row sequences that it receives from the merged index scans.

索引合并交集算法对所有使用的索引执行同时扫描，并根据从合并索引扫描接收的行生成交集。

If all columns used in the query are covered by the used indexes, full table rows are not retrieved (EXPLAIN output contains Using index in Extra field in this case). Here is an example of such a query:

如果查询中使用的所有列都被使用的索引覆盖，则不会检索完整的表行（在这种情况下，`EXPLAIN` 输出在 `Extra` 字段中包含 `Using index` 字样）。 以下是此类查询的示例：

```
SELECT COUNT(*) FROM t1 WHERE key1 = 1 AND key2 = 1;
```

If the used indexes do not cover all columns used in the query, full rows are retrieved only when the range conditions for all used keys are satisfied.

如果使用的索引未涵盖查询中使用的所有列，则仅在满足所使用的键的范围条件时才检索完整行。

If one of the merged conditions is a condition over the primary key of an InnoDB table, it is not used for row retrieval, but is used to filter out rows retrieved using other conditions.

如果其中一个合并条件是 InnoDB 表的主键上的条件，则它不用于行检索，而是用于过滤掉使用其他条件检索的行。

## Index Merge Union Access Algorithm 索引合并并集访问算法

The criteria for this algorithm are similar to those for the Index Merge intersection algorithm. The algorithm is applicable when the table's WHERE clause is converted to several range conditions on different keys combined with OR, and each condition is one of the following:

该算法类似于索引合并交集算法。当表的 `WHERE` 子句可转换为多个范围条件并使用 `OR` 组合时，该算法适用，并且每个条件是以下之一：

* An N-part expression of this form, where the index has exactly N parts (that is, all index parts are covered):

此形式的 N 部分表达式，其中索引具有正好 N 个部分（即所有索引部分都被覆盖）

```
key_part1 = const1 AND key_part2 = const2 ... AND key_partN = constN
```
* Any range condition over a primary key of an InnoDB table.

InnoDB表的主键上的任何范围条件

* A condition for which the Index Merge intersection algorithm is applicable.

索引合并交集算法适用的条件

Examples:

例如：

```
SELECT * FROM t1
WHERE key1 = 1 OR key2 = 2 OR key3 = 3;

SELECT * FROM innodb_table
WHERE (key1 = 1 AND key2 = 2)
    OR (key3 = 'foo' AND key4 = 'bar') AND key5 = 5;
```

## Index Merge Sort-Union Access Algorithm 索引合并排序并集访问算法

This access algorithm is applicable when the WHERE clause is converted to several range conditions combined by OR, but the Index Merge union algorithm is not applicable.

当 `WHERE` 子句转换为由 `OR` 组合的多个范围条件时，此访问算法适用，但索引合并并集算法不适用。

Examples:

例如：

```
SELECT * FROM tbl_name
WHERE key_col1 < 10 OR key_col2 < 20;

SELECT * FROM tbl_name
WHERE (key_col1 > 10 OR key_col2 = 20) AND nonkey_col = 30;
```

The difference between the sort-union algorithm and the union algorithm is that the sort-union algorithm must first fetch row IDs for all rows and sort them before returning any rows.

`sort-union` 算法和 `union` 算法之间的区别在于 `sort-union` 算法必须首先获取所有行的行 ID，然后在返回任何行之前对它们进行排序。

## Influencing Index Merge Optimization 影响索引合并优化的参数

Use of Index Merge is subject to the value of the index_merge, index_merge_intersection, index_merge_union, and index_merge_sort_union flags of the optimizer_switch system variable. See Section 8.9.3, “Switchable Optimizations”. By default, all those flags are on. To enable only certain algorithms, set index_merge to off, and enable only such of the others as should be permitted.

索引合并的使用取决于 `optimizer_switch` 系统变量的 `index_merge`，`index_merge_intersection`，`index_merge_union` 和 `index_merge_sort_union` 参数的值。 请参见[第8.9.3节 “可切换的优化”](https://dev.mysql.com/doc/refman/8.0/en/switchable-optimizations.html)。默认情况下，所有这些标志都已打开。如仅启用部分算法，请将 `index_merge` 设置为 `off` ，并启用允许的算法。

In addition to using the optimizer_switch system variable to control optimizer use of the Index Merge algorithms session-wide, MySQL supports optimizer hints to influence the optimizer on a per-statement basis. See Section 8.9.2, “Optimizer Hints”.

除了使用 `optimizer_switch` 系统变量控制优化程序在会话范围内使用索引合并算法之外，mysql还支持 `optimizer` 提示，按语句影响 `optimizer`。请参见[第8.9.2节“优化程序提示”](https://dev.mysql.com/doc/refman/8.0/en/optimizer-hints.html)。

## 原文

[Index Merge Optimization](https://dev.mysql.com/doc/refman/8.0/en/index-merge-optimization.html)

****
**THE END [ 2019-03-04 ]**