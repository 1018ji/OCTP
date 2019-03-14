# 12.19.3 MySQL Handling of GROUP BY (MySQL 如何处理 GROUP BY)

SQL-92 and earlier does not permit queries for which the select list, HAVING condition, or ORDER BY list refer to nonaggregated columns that are not named in the GROUP BY clause. For example, this query is illegal in standard SQL-92 because the nonaggregated name column in the select list does not appear in the GROUP BY:

SQL-92 标准以及更早版本不允许在 `select` 列表， `HAVING` 条件，以及 `ORDER BY` 列表中使用未在 `GROUP BY` 从句中列出的非聚合列。例如，以下查询在 SQL-92 标准中是非法的，因为 `select` 列表中使用非聚合列 `name` 没有出现在 `GROUP BY` 中：

``` sql
SELECT o.custid, c.name, MAX(o.payment)
FROM orders AS o, customers AS c
WHERE o.custid = c.custid
GROUP BY o.custid;
```
For the query to be legal in SQL-92, the name column must be omitted from the select list or named in the GROUP BY clause.

要使以上查询在 SQL-92 中合法，`name` 列必须在 `select` 列表中删除或者在 `GROUP BY` 使用该字段。

SQL:1999 and later permits such nonaggregates per optional feature T301 if they are functionally dependent on GROUP BY columns: If such a relationship exists between name and custid, the query is legal. This would be the case, for example, were custid a primary key of customers.

SQL:1999 以及更高版本允许非聚合列基于可选功能 T301（注：T301 可在 SQL:1999 标准中查询到），前提是它们与 `GROUP BY` 的列存在函数依赖：如果 `name` 和 `custid` 之间存在此类关系，那么查询将会是合法的。例如，`customers` 表的主键是 `custid`，便是其中的一种情况。

MySQL implements detection of functional dependence. If the ONLY\_FULL\_GROUP\_BY SQL mode is enabled (which it is by default), MySQL rejects queries for which the select list, HAVING condition, or ORDER BY list refer to nonaggregated columns that are neither named in the GROUP BY clause nor are functionally dependent on them.

MySQL 实现了对函数依赖的检测。如果启用了 `ONLY_FULL_GROUP_BY` SQL 模式（默认开启），MySQL会拒绝查询在 `select` 列表，`HAVING` 条件，以及 `ORDER BY` 列表使用未在 `GROUP BY` 从句中列出并且与 `GROUP BY` 从句列无依赖非聚合列。

If ONLY\_FULL\_GROUP\_BY is disabled, a MySQL extension to the standard SQL use of GROUP BY permits the select list, HAVING condition, or ORDER BY list to refer to nonaggregated columns even if the columns are not functionally dependent on GROUP BY columns. This causes MySQL to accept the preceding query. In this case, the server is free to choose any value from each group, so unless they are the same, the values chosen are nondeterministic, which is probably not what you want. Furthermore, the selection of values from each group cannot be influenced by adding an ORDER BY clause. Result set sorting occurs after values have been chosen, and ORDER BY does not affect which value within each group the server chooses. Disabling ONLY\_FULL\_GROUP\_BY is useful primarily when you know that, due to some property of the data, all values in each nonaggregated column not named in the GROUP BY are the same for each group.

如果关闭 `ONLY_FULL_GROUP_BY`，MySQL 会将标准 SQL 扩展为支持使用在 `select` 列表，`HAVING` 条件，以及 `ORDER BY` 列表使用未在 `GROUP BY` 从句中列出甚至与 `GROUP BY` 从句列无依赖非聚合列。这会导致 MySQL 接受前面的非法查询。在这种情况下，可以自由地从每个分组中选择任何值，除非它们相同，但字段返回值是不确定的，可能不是你的期望值。此外，添加`ORDER BY` 从句不会影响分组中值的选择。结果集排序发生在值选择之后，排序方式不影响服务器分组中列值选择。对于数据的一些属性，`GROUP BY` 从句列中非聚合列在每个分组中都是一致的，那么关闭 `ONLY_FULL_GROUP_BY` 将会是有益的。

You can achieve the same effect without disabling ONLY\_FULL\_GROUP\_BY by using ANY_VALUE() to refer to the nonaggregated column.

通过使用 `ANY_VALUE()` 来引用非聚合列，可以在不禁用 `ONLY_FULL_GROUP_BY` 的情况下实现相同的效果。

The following discussion demonstrates functional dependence, the error message MySQL produces when functional dependence is absent, and ways of causing MySQL to accept a query in the absence of functional dependence.

以下操作演示函数依赖性，MySQL 在缺少函数依赖时产生的错误消息，以及在没有函数依赖性的情况下使 MySQL 接受查询的方法。

This query might be invalid with ONLY\_FULL\_GROUP\_BY enabled because the nonaggregated address column in the select list is not named in the GROUP BY clause:

此查询可能是无效的在 `ONLY_FULL_GROUP_BY` 开启的情况下，因为 `select` 列表中的非聚合列未在 `GROUP BY` 从句中列出：

``` sql
SELECT name, address, MAX(age) FROM t GROUP BY name;
```

The query is valid if name is a primary key of t or is a unique NOT NULL column. In such cases, MySQL recognizes that the selected column is functionally dependent on a grouping column. For example, if name is a primary key, its value determines the value of address because each group has only one value of the primary key and thus only one row. As a result, there is no randomness in the choice of address value in a group and no need to reject the query.

如果 `name` 列为表 `t` 的主键或者非 NULL 的唯一列，那个查询将会是正确的。在这种情况下，MySQL 会意识到所选择的列函数依赖分组列。假设 `name` 列为主键，其值将会确定 `address` 的值，因为每个分组只包含一个主键值以及一行数据。结果就是，分组中 `address` 的值讲会没有随机性，那么就没有拒绝查询的必要了。

The query is invalid if name is not a primary key of t or a unique NOT NULL column. In this case, no functional dependency can be inferred and an error occurs:

假设 `name` 列不是表 `t` 的主键或者非 NULL 的唯一列，那么以下查询会是错误的。这种情况下不能推断出函数依赖并会出现错误

```
mysql> SELECT name, address, MAX(age) FROM t GROUP BY name;

ERROR 1055 (42000): Expression #2 of SELECT list is not in GROUP
BY clause and contains nonaggregated column 'mydb.t.address' which
is not functionally dependent on columns in GROUP BY clause; this
is incompatible with sql_mode=only_full_group_by
```

If you know that, for a given data set, each name value in fact uniquely determines the address value, address is effectively functionally dependent on name. To tell MySQL to accept the query, you can use the ANY_VALUE() function:

对于给定的数据，如果能确定每个 `name` 值确定一个 `address` 值，`address` 列函数依赖 `name` 列。可以使用 `ANY_VALUE()` 函数让 MySQL 接收查询：

``` sql
SELECT name, ANY_VALUE(address), MAX(age) FROM t GROUP BY name;
```

Alternatively, disable ONLY\_FULL\_GROUP\_BY.
或者禁用 `ONLY_FULL_GROUP_BY`。

The preceding example is quite simple, however. In particular, it is unlikely you would group on a single primary key column because every group would contain only one row. For addtional examples demonstrating functional dependence in more complex queries, see Section 12.20.4, “Detection of Functional Dependence”.

前面的例子非常简单。实际上你不太可能在主键上进行分组，因为每个分组只有一行数据。有关函数依赖更复杂的查询，参见 12.20.4 部分，[Detection of Functional Dependence](https://dev.mysql.com/doc/refman/8.0/en/group-by-functional-dependence.html)

If a query has aggregate functions and no GROUP BY clause, it cannot have nonaggregated columns in the select list, HAVING condition, or ORDER BY list with ONLY\_FULL\_GROUP\_BY enabled:

如果一个查询有聚合函数列但没使用 `GROUP BY` 从句，那么在 `ONLY_FULL_GROUP_BY` 开启的条件下，在 `select` 列表， `HAVING` 条件，以及 `ORDER BY` 列表不能出现非聚合列：

```
mysql> SELECT name, MAX(age) FROM t;

ERROR 1140 (42000): In aggregated query without GROUP BY, expression
#1 of SELECT list contains nonaggregated column 'mydb.t.name'; this
is incompatible with sql_mode=only_full_group_by
```

Without GROUP BY, there is a single group and it is nondeterministic which name value to choose for the group. Here, too, ANY_VALUE() can be used, if it is immaterial which name value MySQL chooses:

未使用 `GROUP BY`，这里将会只有一个分组，那么 `name` 的值是不确定的。如果 `name` 值选择无关紧要，在这里使用 `ANY_VALUE()` 也会是有效的。

``` sql
SELECT ANY_VALUE(name), MAX(age) FROM t;
```

ONLY\_FULL\_GROUP\_BY also affects handling of queries that use DISTINCT and ORDER BY. Consider the case of a table t with three columns c1, c2, and c3 that contains these rows:

`ONLY_FULL_GROUP_BY` 也会影响到 `DISTINCT` 和 `ORDER BY` 的查询处理。假设有表 `t`，其有 `c1`，`c2` 以及 `c3` 三列，并有如下数据

| c1 | c2 | c3 |
| ------| ------ | ------ |
| 1 | 2 | A |
| 3 | 4 | B |
| 1 | 2 | C |

Suppose that we execute the following query, expecting the results to be ordered by c3:

假设我们执行一下查询，期望结果根据 `c3` 列排序：

``` sql
SELECT DISTINCT c1, c2 FROM t ORDER BY c3;
```

To order the result, duplicates must be eliminated first. But to do so, should we keep the first row or the third? This arbitrary choice influences the retained value of c3, which in turn influences ordering and makes it arbitrary as well. To prevent this problem, a query that has DISTINCT and ORDER BY is rejected as invalid if any ORDER BY expression does not satisfy at least one of these conditions:

为了排序结果，必须先消除重复项。但是要这样做，我们应该保留第一排还是第三排？这种任意选择会影响 `c3` 的保留值，而 `c3` 的保留值反过来又会影响排序并使其变为随机的。为避免出现此问题，如果任何 `ORDER BY` 表达式不满足以下条件之一，则拒绝带有 `DISTINCT` 和 `ORDER BY` 从句的的查询：

* The expression is equal to one in the select list
* All columns referenced by the expression and belonging to the query's selected tables are elements of the select list

* 表达式将使 `select` 列只包含一个值
* 所有被查询表中被表达式使用的列须包含在 `select` 列

Another MySQL extension to standard SQL permits references in the HAVING clause to aliased expressions in the select list. For example, the following query returns name values that occur only once in table orders:

另外一个 MySQL 对于标准 SQL 的扩展为允许在 `HAVING` 从句中使用在 `select` 列定义的别名。如下查询将会返回在表 `orders ` 中只出现一次的 `name` 值：

``` sql
SELECT name, COUNT(name) FROM orders
GROUP BY name
HAVING COUNT(name) = 1;
```

The MySQL extension permits the use of an alias in the HAVING clause for the aggregated column:

MySQL扩展允许在聚合列的 `HAVING` 从句中使用别名：

``` sql
SELECT name, COUNT(name) AS c FROM orders
GROUP BY name
HAVING c = 1;
```

Standard SQL permits only column expressions in GROUP BY clauses, so a statement such as this is invalid because FLOOR(value/100) is a noncolumn expression:

标准 SQL 仅允许在 `GROUP BY` 从句中使用列表达式，因此诸如此类的语句将是无效的，因为 `FLOOR(value/100))` 为非列表达式：

``` sql
SELECT id, FLOOR(value/100)
FROM tbl_name
GROUP BY id, FLOOR(value/100);
```

MySQL extends standard SQL to permit noncolumn expressions in GROUP BY clauses and considers the preceding statement valid.

MySQL 扩展标准 SQL 以允许在 `GROUP BY` 从句中使用非列表达式，并认为前面的语句是有效的。

Standard SQL also does not permit aliases in GROUP BY clauses. MySQL extends standard SQL to permit aliases, so another way to write the query is as follows:

标准 SQL 也不允许 `GROUP BY` 从句中使用别名。MySQL 扩展标准 SQL 以允许别名，因此另一种查询方法如下：

``` sql
SELECT id, FLOOR(value/100) AS val
FROM tbl_name
GROUP BY id, val;
```

The alias val is considered a column expression in the GROUP BY clause.

别名 `val` 被视为 `GROUP BY` 从句中的列表达式。

In the presence of a noncolumn expression in the GROUP BY clause, MySQL recognizes equality between that expression and expressions in the select list. This means that with ONLY\_FULL\_GROUP\_BY SQL mode enabled, the query containing GROUP BY id, FLOOR(value/100) is valid because that same FLOOR() expression occurs in the select list. However, MySQL does not try to recognize functional dependence on GROUP BY noncolumn expressions, so the following query is invalid with ONLY\_FULL\_GROUP\_BY enabled, even though the third selected expression is a simple formula of the id column and the FLOOR() expression in the GROUP BY clause:

在 `GROUP BY` 从句出现非列表达式时，MySQL 会意识到该表达式与 `select` 列中的表达式之间的相等性。这意味着启用了 `ONLY_FULL_GROUP_BY` SQL 模式后，查询中包含的 `GROUP BY id, FLOOR(value/100)` 会是有效的，因为同样的 `FLOOR()` 表达式出现在 `select` 列。MySQL 不会尝试分析 `GROUP BY` 中非列表达式的函数依赖，在 `ONLY_FULL_GROUP_BY` 开启的情况下一下查询将会是无效的，及时第三个表达式就是讲 `GROUP BY` 的 `id` 列以及 `FLOOR()`表达式使用简单公式进行计算。

``` sql
SELECT id, FLOOR(value/100), id+FLOOR(value/100)
FROM tbl_name
GROUP BY id, FLOOR(value/100);
```

A workaround is to use a derived table:

一种解决的办法就是使用派生表：

``` sql
SELECT id, F, id+F
FROM
(
  SELECT id, FLOOR(value/100) AS F
  FROM tbl_name
  GROUP BY id, FLOOR(value/100)
) AS dt;
```

## 原文

[12.19.3 MySQL Handling of GROUP BY](https://dev.mysql.com/doc/refman/8.0/en/group-by-handling.html)

****
**THE END [ 2019-01-22 ]**