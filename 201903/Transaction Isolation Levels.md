# 15.7.2.1 Transaction Isolation Levels (事务隔离级别)

Transaction isolation is one of the foundations of database processing. Isolation is the I in the acronym ACID; the isolation level is the setting that fine-tunes the balance between performance and reliability, consistency, and reproducibility of results when multiple transactions are making changes and performing queries at the same time.

事务隔离是数据库处理的基础之一。`Isolation` 是 `ACID`的缩写中的 `I`；事务隔离级别设置用于在多个事务同时进行更改和执行查询时使性能与可靠性、一致性和可再现性之间达到平衡。

InnoDB offers all four transaction isolation levels described by the SQL:1992 standard: READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, and SERIALIZABLE. The default isolation level for InnoDB is REPEATABLE READ.

`InnoDB` 提供了 `SQL:1992` 标准描述的所有四个事务隔离级别：`READ UNCOMMITTED`，`READ COMMITTED`，`REPEATABLE READ` 和 `SERIALIZABLE`。`InnoDB` 的默认隔离级别是 `REPEATABLE READ`。

A user can change the isolation level for a single session or for all subsequent connections with the SET TRANSACTION statement. To set the server's default isolation level for all connections, use the --transaction-isolation option on the command line or in an option file. For detailed information about isolation levels and level-setting syntax, see Section 13.3.7, “SET TRANSACTION Syntax”.

用户可以使用 `SET TRANSACTION` 语句更改单个会话或所有连续连接的隔离级别。要为所有连接设置默认的隔离级别，请在命令行或配置文件中使用 `--transaction-isolation` 选项。有关隔离级别和级别设置语法的详细信息，参照[章节 13.3.7, “SET TRANSACTION Syntax”](https://dev.mysql.com/doc/refman/8.0/en/set-transaction.html)。

InnoDB supports each of the transaction isolation levels described here using different locking strategies. You can enforce a high degree of consistency with the default REPEATABLE READ level, for operations on crucial data where ACID compliance is important. Or you can relax the consistency rules with READ COMMITTED or even READ UNCOMMITTED, in situations such as bulk reporting where precise consistency and repeatable results are less important than minimizing the amount of overhead for locking. SERIALIZABLE enforces even stricter rules than REPEATABLE READ, and is used mainly in specialized situations, such as with XA transactions and for troubleshooting issues with concurrency and deadlocks.

`InnoDB` 使用不同的锁定策略支持每个事务隔离级别。您可以使用默认的 `REPEATABLE READ` 级别强制达到高度一致性，以便对需符合 `ACID` 规范的重要关键数据进行操作。或者您可以选择使用较弱的 `READ COMMITTED` 甚至 `READ UNCOMMITTED` 一致性规则，例如在批量报告情形下，其准确一致性和可重复结果不如加锁带来的开销重要。`SERIALIZABLE` 强制执行比 `REPEATABLE READ` 更严格的规则，这主要用于某些特殊情况，例如 `XA` 事务和并发和死锁的带来的故障排除问题。

The following list describes how MySQL supports the different transaction levels. The list goes from the most commonly used level to the least used.

以下列表描述 `MySQL` 如何支持不同的事务级别。该列表从最常用的级别至最少使用的级别进行描述。

* REPEATABLE READ 可重复读

This is the default isolation level for InnoDB. Consistent reads within the same transaction read the snapshot established by the first read. This means that if you issue several plain (nonlocking) SELECT statements within the same transaction, these SELECT statements are consistent also with respect to each other. See Section 15.7.2.3, “Consistent Nonlocking Reads”.

这是 `InnoDB` 的默认隔离级别。同一事务中的一致读读取第一次读取所建立的快照 （开启事务不一定会马上简历快照的版本，可能需要第一次读取后才建立）。这意味着在同一事务中发出几个普通（非锁定）`SELECT` 语句，那么这些 `SELECT` 语句读取结果已知。参照[章节 15.7.2.3, “Consistent Nonlocking Reads”](https://dev.mysql.com/doc/refman/8.0/en/innodb-consistent-read.html)。

For locking reads (SELECT with FOR UPDATE or FOR SHARE), UPDATE, and DELETE statements, locking depends on whether the statement uses a unique index with a unique search condition, or a range-type search condition.

对于锁定读取（`SELECT` 使用 `FOR UPDATE` 或 `FOR SHARE`），`UPDATE` 和 `DELETE` 语句，锁定范围取决于语句是使用唯一索引还是范围索引搜索。

- For a unique index with a unique search condition, InnoDB locks only the index record found, not the gap before it.

对于使用唯一索引进行搜索的条件 `InnoDB` 仅锁定找到的索引记录，而不锁定间隙。

- For other search conditions, InnoDB locks the index range scanned, using gap locks or next-key locks to block insertions by other sessions into the gaps covered by the range. For information about gap locks and next-key locks, see Section 15.7.1, “InnoDB Locking”.

对于其他搜索条件，`InnoDB` 锁定扫描的索引范围，使用 `gap` 锁或 `next-key` 锁来阻止其他会话插入范围所覆盖的间隙。 有关`gap` 锁或 `next-key` 锁的信息，参照[章节 15.7.1, “InnoDB Locking”](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)。

* READ COMMITTED 读已提交

Each consistent read, even within the same transaction, sets and reads its own fresh snapshot. For information about consistent reads, see Section 15.7.2.3, “Consistent Nonlocking Reads”.

即使在同一事务中每个一致性读取也会设置并读取自己的新快照。 有关一致性读取的信息，参照[章节 15.7.2.3, “Consistent Nonlocking Reads”](https://dev.mysql.com/doc/refman/8.0/en/innodb-consistent-read.html)。

For locking reads (SELECT with FOR UPDATE or FOR SHARE), UPDATE statements, and DELETE statements, InnoDB locks only index records, not the gaps before them, and thus permits the free insertion of new records next to locked records. Gap locking is only used for foreign-key constraint checking and duplicate-key checking.

对于锁定读取（`SELECT` 使用 `FOR UPDATE` 或 `FOR SHARE`），`UPDATE` 和 `DELETE` 语句，`InnoDB` 仅锁定索引记录，而不锁定它们之前的间隙，因此允许在锁定记录旁边自由插入新记录。`gap` 锁定仅用于外键约束检查和重复键检查。

Because gap locking is disabled, phantom problems may occur, as other sessions can insert new rows into the gaps. For information about phantoms, see Section 15.7.4, “Phantom Rows”.

由于禁用了 `gap` 锁，因此可能会出现幻读问题，因为其他会话可以在间隙中插入新行。 有关幻读的信息，参照[章节 15.7.4, “Phantom Rows”](https://dev.mysql.com/doc/refman/8.0/en/innodb-next-key-locking.html)。

Only row-based binary logging is supported with the READ COMMITTED isolation level. If you use READ COMMITTED with binlog_format=MIXED, the server automatically uses row-based logging.

`READ COMMITTED` 仅支持基于行的二进制日志记录。如果将 `READ COMMITTED` 与 `binlog_format=MIXED` 一起使用，服务器将自动使用基于行的日志记录。

Using READ COMMITTED has additional effects:

使用 `READ COMMITTED` 其他影响：

- For UPDATE or DELETE statements, InnoDB holds locks only for rows that it updates or deletes. Record locks for nonmatching rows are released after MySQL has evaluated the WHERE condition. This greatly reduces the probability of deadlocks, but they can still happen.

对于 `UPDATE` 或 `DELETE` 语句，`InnoDB` 仅锁定其更新或删除的行。`MySQL` 评估 `WHERE` 条件后，将释放不匹配行的记录锁。 这大大降低了死锁的可能性，但它们仍然可以发生。

- For UPDATE statements, if a row is already locked, InnoDB performs a “semi-consistent” read, returning the latest committed version to MySQL so that MySQL can determine whether the row matches the WHERE condition of the UPDATE. If the row matches (must be updated), MySQL reads the row again and this time InnoDB either locks it or waits for a lock on it.

对于 `UPDATE` 语句，如果一行已被锁定，`InnoDB` 将执行半一致性读取，将最近最新提交的版本返回给 `MySQL`，以便 `MySQL` 可以确定该行是否与 `UPDATE` 的 `WHERE` 条件匹配。 如果行匹配（必须更新），MySQL再次读取该行，这次InnoDB将其锁定或等待锁定。

Consider the following example, beginning with this table:

请考虑以下情况，从此表开始：

```
CREATE TABLE t (a INT NOT NULL, b INT) ENGINE = InnoDB;
INSERT INTO t VALUES (1,2),(2,3),(3,2),(4,3),(5,2);
COMMIT;
```

In this case, the table has no indexes, so searches and index scans use the hidden clustered index for record locking (see Section 15.6.2.1, “Clustered and Secondary Indexes”) rather than indexed columns.

在这种情况下，表没有索引，因此搜索和索引扫描使用隐藏的聚簇索引进行记录锁定（参照[章节 15.6.2.1, “Clustered and Secondary Indexes”](https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html)）而不是索引列。

Suppose that one session performs an UPDATE using these statements:

假设一个会话使用以下语句执行 `UPDATE`：

```
# Session A
START TRANSACTION;
UPDATE t SET b = 5 WHERE b = 3;
```

Suppose also that a second session performs an UPDATE by executing these statements following those of the first session:

另外，假设第二个会话在第一个会话的语句之后执行 `UPDATE`：

```
# Session B
UPDATE t SET b = 4 WHERE b = 2;
```

As InnoDB executes each UPDATE, it first acquires an exclusive lock for each row, and then determines whether to modify it. If InnoDB does not modify the row, it releases the lock. Otherwise, InnoDB retains the lock until the end of the transaction. This affects transaction processing as follows.

当 `InnoDB` 执行每个更新时，它首先为每一行获取一个排它锁，然后决定是否修改它。如果 `InnoDB` 不修改行，行排它锁将被释放。否则，`InnoDB` 将保留排它锁，直到事务结束。这会影响事务处理，如下所示。

When using the default REPEATABLE READ isolation level, the first UPDATE acquires an x-lock on each row that it reads and does not release any of them:

当使用默认的 `REPEATABLE READ` 隔离级别时，第一次更新会在其读取的每一行上获取一个排它锁，并且不会释放其中的任何一行：

```
x-lock(1,2); retain x-lock
x-lock(2,3); update(2,3) to (2,5); retain x-lock
x-lock(3,2); retain x-lock
x-lock(4,3); update(4,3) to (4,5); retain x-lock
x-lock(5,2); retain x-lock
```

The second UPDATE blocks as soon as it tries to acquire any locks (because first update has retained locks on all rows), and does not proceed until the first UPDATE commits or rolls back:

第二个更新在尝试获取任何锁时立即阻塞（因为第一个更新保留所有行的锁），并且在第一个更新提交或回滚之前不会继续：

```
x-lock(1,2); block and wait for first UPDATE to commit or roll back
```

If READ COMMITTED is used instead, the first UPDATE acquires an x-lock on each row that it reads and releases those for rows that it does not modify:

如果改为使用 `READ COMMITTED`，则第一次更新将在其读取的每一行上加排它锁，并释放其未修改的行排它锁：

```
x-lock(1,2); unlock(1,2)
x-lock(2,3); update(2,3) to (2,5); retain x-lock
x-lock(3,2); unlock(3,2)
x-lock(4,3); update(4,3) to (4,5); retain x-lock
x-lock(5,2); unlock(5,2)
```

For the second UPDATE, InnoDB does a “semi-consistent” read, returning the latest committed version of each row that it reads to MySQL so that MySQL can determine whether the row matches the WHERE condition of the UPDATE:

对于第二次更新，`InnoDB `执行半一致性读取，将其读取的每一行的最新提交版本返回到 `MySQL`，以便 `MySQL` 可以确定 `WHERE` 条件是否可执行 `UPDATE`：

```
x-lock(1,2); update(1,2) to (1,4); retain x-lock
x-lock(2,3); unlock(2,3)
x-lock(3,2); update(3,2) to (3,4); retain x-lock
x-lock(4,3); unlock(4,3)
x-lock(5,2); update(5,2) to (5,4); retain x-lock
```

However, if the WHERE condition includes an indexed column, and InnoDB uses the index, only the indexed column is considered when taking and retaining record locks. In the following example, the first UPDATE takes and retains an x-lock on each row where b = 2. The second UPDATE blocks when it tries to acquire x-locks on the same records, as it also uses the index defined on column b.

但是，如果 `WHERE `条件包含索引列，并且 `InnoDB` 使用了该索引，则只有索引列能被获取和保留排它锁。 在下面的示例中，第一个 `UPDATE` 在 `where b = 2` 每个行上获取并保留排它锁而第二个 `UPDATE` 在尝试获取相同排它锁时阻塞，因为它同样使用在列 `b` 上定义的索引。

```
CREATE TABLE t (a INT NOT NULL, b INT, c INT, INDEX (b)) ENGINE = InnoDB;
INSERT INTO t VALUES (1,2,3),(2,2,4);
COMMIT;

# Session A
START TRANSACTION;
UPDATE t SET b = 3 WHERE b = 2 AND c = 3;

# Session B
UPDATE t SET b = 4 WHERE b = 2 AND c = 4;
```

The effects of using the READ COMMITTED isolation level are the same as enabling the deprecated innodb_locks_unsafe_for_binlog configuration option, with these exceptions:

使用 `READ COMMITTED` 隔离级别的效果与启用不推荐使用的 `innodb_locks_unsafe_for_binlog` 配置选项相同，但有以下例外：

- Enabling innodb_locks_unsafe_for_binlog is a global setting and affects all sessions, whereas the isolation level can be set globally for all sessions, or individually per session.

启用 `innodb_locks_unsafe_for_binlog` 是一个全局设置并影响所有会话，而隔离级别可以为所有会话全局设置，也可以为每个会话单独设置。

- innodb_locks_unsafe_for_binlog can be set only at server startup, whereas the isolation level can be set at startup or changed at runtime.

`innodb_locks_unsafe_for_binlog` 只能在服务器启动时设置，而隔离级别可以在启动时设置或在运行时更改。

READ COMMITTED therefore offers finer and more flexible control than innodb_locks_unsafe_for_binlog.

因此，`READ COMMITTED` 提供比 `innodb_locks_unsafe_for_binlog` 更精细更灵活的控制。

* READ UNCOMMITTED 读未提交

SELECT statements are performed in a nonlocking fashion, but a possible earlier version of a row might be used. Thus, using this isolation level, such reads are not consistent. This is also called a dirty read. Otherwise, this isolation level works like READ COMMITTED.

`SELECT` 语句以非锁定方式执行，但可能读取到行早期使用的版本。因此使用此隔离级别可能导致数据不一致。这也称为脏读。此外此隔离级别与 `READ COMMITTED` 类似。

* SERIALIZABLE 序列化

This level is like REPEATABLE READ, but InnoDB implicitly converts all plain SELECT statements to SELECT ... FOR SHARE if autocommit is disabled. If autocommit is enabled, the SELECT is its own transaction. It therefore is known to be read only and can be serialized if performed as a consistent (nonlocking) read and need not block for other transactions. (To force a plain SELECT to block if other transactions have modified the selected rows, disable autocommit.)

此级别类似于 `REPEATABLE READ`，但 `InnoDB` 在禁用自动提交情况下隐式地将所有 `SELECT` 语句转换为 `SELECT ... FOR SHARE`。如果启用自动提交，`SELECT` 本身作为一个事务。这样就被认为只读并可被序列化，表现为非锁定的一致性读以及不能被其他事物锁定。（强制 `SELECT` 阻止其他事物修改所选的行以及自动提交。）

## 原文

[15.7.2.1 Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)

****
**THE END [ 2019-03-14 ]**