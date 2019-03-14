# 13.3.1 START TRANSACTION, COMMIT, and ROLLBACK Syntax (START TRANSACTION, COMMIT, and ROLLBACK 语法)

```
START TRANSACTION
    [transaction_characteristic [, transaction_characteristic] ...]

transaction_characteristic: {
    WITH CONSISTENT SNAPSHOT
  | READ WRITE
  | READ ONLY
}

BEGIN [WORK]
COMMIT [WORK] [AND [NO] CHAIN] [[NO] RELEASE]
ROLLBACK [WORK] [AND [NO] CHAIN] [[NO] RELEASE]
SET autocommit = {0 | 1}
```

These statements provide control over use of transactions:

这些语句提供在事务中使用：

* START TRANSACTION or BEGIN start a new transaction.

`START TRANSACTION` 或者 `BEGIN` 启动一个新事物。

* COMMIT commits the current transaction, making its changes permanent.

`COMMIT` 提交当前事务，并将其持久化。

* ROLLBACK rolls back the current transaction, canceling its changes.

`ROLLBACK` 回滚当前事务操作，取消其更改。

* SET autocommit disables or enables the default autocommit mode for the current session.

`SET autocommit` 对当前会话开启或关闭自动提交模式。

By default, MySQL runs with autocommit mode enabled. This means that as soon as you execute a statement that updates (modifies) a table, MySQL stores the update on disk to make it permanent. The change cannot be rolled back.

默认情况下，MySQL 默认开启自动提交模式。这意味着执行或修改表语句，MySQL 会将更新持久化到硬盘。这些操作无法被回滚。

To disable autocommit mode implicitly for a single series of statements, use the START TRANSACTION statement:

要为一系列简单语句关闭自动提交模式，需使用 `START TRANSACTION` 语句：

```
START TRANSACTION;
SELECT @A:=SUM(salary) FROM table1 WHERE type=1;
UPDATE table2 SET summary=@A WHERE type=1;
COMMIT;
```

With START TRANSACTION, autocommit remains disabled until you end the transaction with COMMIT or ROLLBACK. The autocommit mode then reverts to its previous state.

使用 `START TRANSACTION` 将会导致自动提交关闭直至使用 `COMMIT` 或 `ROLLBACK` 终止事务。自动提交模式将恢复到先前的状态。

START TRANSACTION permits several modifiers that control transaction characteristics. To specify multiple modifiers, separate them by commas.

START TRANSACTION 允许使用多个控制使用特征的修饰符。如需指定多个修饰符，请使用逗号分隔。

* The WITH CONSISTENT SNAPSHOT modifier starts a consistent read for storage engines that are capable of it. This applies only to InnoDB. The effect is the same as issuing a START TRANSACTION followed by a SELECT from any InnoDB table. See Section 14.7.2.3, “Consistent Nonlocking Reads”. The WITH CONSISTENT SNAPSHOT modifier does not change the current transaction isolation level, so it provides a consistent snapshot only if the current isolation level is one that permits a consistent read. The only isolation level that permits a consistent read is REPEATABLE READ. For all other isolation levels, the WITH CONSISTENT SNAPSHOT clause is ignored.

`WITH CONSISTENT SNAPSHOT` 修饰符会为存储引擎开启一致性读（快照读）的能力。此修订符只试用于 `InnoDB` 引擎。此修饰符与对 `InnoDB` 表使用 `START TRANSACTION` 后紧跟 `SELECT` 的效果是一样的。参照[章节 14.7.2.3, “Consistent Nonlocking Reads”](https://dev.mysql.com/doc/refman/5.6/en/innodb-consistent-read.html)。 `WITH CONSISTENT SNAPSHOT` 修饰符不会更改当前事务隔离级别，因此仅当当前隔离级别允许一致读取时才提供一致性读。允许一致性读的隔离级别只有 `REPEATABLE READ` 可重复读 (RR) 一个。对于其他隔离级别，`WITH CONSISTENT SNAPSHOT` 将被忽略。

* The READ WRITE and READ ONLY modifiers set the transaction access mode. They permit or prohibit changes to tables used in the transaction. The READ ONLY restriction prevents the transaction from modifying or locking both transactional and nontransactional tables that are visible to other transactions; the transaction can still modify or lock temporary tables. These modifiers are available as of MySQL 5.6.5.

`READ WRITE` 与 `READ ONLY` 修饰符用户设定事务的访问模式。他们允许或禁止在事务中修改表数据。`READ ONLY` 限制并阻止事务修改或锁订对其它事务可见的事务表或非事务表；事务仍可修改或锁定临时表。这些修饰符从 MySQL 5.6.5 开始提供。

MySQL enables extra optimizations for queries on InnoDB tables when the transaction is known to be read-only. Specifying READ ONLY ensures these optimizations are applied in cases where the read-only status cannot be determined automatically. See Section 8.5.3, “Optimizing InnoDB Read-Only Transactions” for more information.

当事务开启只读模式时，`MySQL` 将对 `InnoDB` 表上的查询进行额外优化。指定 `READ ONLY` 可确保在无法自动确定只读状态的情况下应用这些优化。参照[章节 8.5.3, “Optimizing InnoDB Read-Only Transactions](https://dev.mysql.com/doc/refman/5.6/en/innodb-performance-ro-txn.html)获取更多信息。

If no access mode is specified, the default mode applies. Unless the default has been changed, it is read/write. It is not permitted to specify both READ WRITE and READ ONLY in the same statement.

如果未指定访问模式将会使用默认的访问模式。除非默认访问模式被更改，否则为读写模式。不允许在同一语句中同时指定 `READ WRITE` 和 `READ ONLY`。

In read-only mode, it remains possible to change tables created with the TEMPORARY keyword using DML statements. Changes made with DDL statements are not permitted, just as with permanent tables.

在只读模式下，仍然可以使用 `DML` 语句更改使用 `TEMPORARY` 关键字创建的表。与永久表一样，不允许使用DDL语句进行更改。

For additional information about transaction access mode, including ways to change the default mode, see Section 13.3.6, “SET TRANSACTION Syntax”.

有关事务访问模式（包括更改默认模式的方法）的更多信息，参照[章节 13.3.6, “SET TRANSACTION Syntax”](https://dev.mysql.com/doc/refman/5.6/en/set-transaction.html)。

If the read_only system variable is enabled, explicitly starting a transaction with START TRANSACTION READ WRITE requires the SUPER privilege.

如果启用了 `read_only` 系统变量，使用 `START TRANSACTION READ WRITE` 显式启动事务需要 `SUPER` 权限。

**Important 重要**

Many APIs used for writing MySQL client applications (such as JDBC) provide their own methods for starting transactions that can (and sometimes should) be used instead of sending a START TRANSACTION statement from the client. See Chapter 23, Connectors and APIs, or the documentation for your API, for more information.

许多用于链接 `MySQL` 客户端应用程序（例如JDBC）的API提供了自己的方法来启动事务，而不是从客户端发送 `START TRANSACTION` 语句。 有关详细信息，参照[章节 23, Connectors and APIs](https://dev.mysql.com/doc/refman/5.6/en/connectors-apis.html) 或者你使用客户端的 API 文档。

To disable autocommit mode explicitly, use the following statement:

要显式禁用自动提交模式，请使用以下语句：

```
SET autocommit=0;
```

After disabling autocommit mode by setting the autocommit variable to zero, changes to transaction-safe tables (such as those for InnoDB or NDB) are not made permanent immediately. You must use COMMIT to store your changes to disk or ROLLBACK to ignore the changes.

通过将 `autocommit` 变量设置为0来禁用自动提交模式后，对事务安全表（例如 `InnoDB` 或 `NDB` 的表）的更改不会立即持久化。必须使用 `COMMIT` 将更改持久化到磁盘或 `ROLLBACK` 以忽略更改。

autocommit is a session variable and must be set for each session. To disable autocommit mode for each new connection, see the description of the autocommit system variable at Section 5.1.7, “Server System Variables”.

`autocommit` 是一个会话变量，必须为每个会话设置。 要为每个新连接禁用自动提交模式，参照[章节 5.1.7, “Server System Variables”](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html)的自动提交系统变量的说明。

BEGIN and BEGIN WORK are supported as aliases of START TRANSACTION for initiating a transaction. START TRANSACTION is standard SQL syntax, is the recommended way to start an ad-hoc transaction, and permits modifiers that BEGIN does not.

`BEGIN` 和 `BEGIN WORK` 作为 `START TRANSACTION` 的别名来启动事务。`START TRANSACTION` 是标准的 SQL 语法，是启动临时事务的推荐方法，注意 `BEGIN` 不允许使用的修饰符。

The BEGIN statement differs from the use of the BEGIN keyword that starts a BEGIN ... END compound statement. The latter does not begin a transaction. See Section 13.6.1, “BEGIN ... END Compound-Statement Syntax”.

`BEGIN` 语句不同于使用 `BEGIN` 关键字启动 `BEGIN ... END` 复合语句。后者不会开启一个事务。 参照[章节 13.6.1, “BEGIN ... END Compound-Statement Syntax”](https://dev.mysql.com/doc/refman/5.6/en/begin-end.html)。

**Note 注意**

Within all stored programs (stored procedures and functions, triggers, and events), the parser treats BEGIN [WORK] as the beginning of a BEGIN ... END block. Begin a transaction in this context with START TRANSACTION instead.

对于例如 `stored procedures`（存储过程）、`functions`（自定义函数）、`triggers`（触发器）以及 `events`（事件）来说，解析器将 `BEGIN [WORK]` 视为 `BEGIN ... END` 块的开头。在此上下文中需使用 `START TRANSACTION` 开始事务。

The optional WORK keyword is supported for COMMIT and ROLLBACK, as are the CHAIN and RELEASE clauses. CHAIN and RELEASE can be used for additional control over transaction completion. The value of the completion_type system variable determines the default completion behavior. See Section 5.1.7, “Server System Variables”.

`COMMIT` 和 `ROLLBACK` 支持可选的 `WORK` 关键字，`CHAIN` 和 `RELEASE` 子句也支持。`CHAIN` 和 `RELEASE` 可用于对事务完成的附加控制。`completion_type` 系统变量的值确定默认的完成行为。 参照[章节 5.1.7, “Server System Variables”](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html)。

The AND CHAIN clause causes a new transaction to begin as soon as the current one ends, and the new transaction has the same isolation level as the just-terminated transaction. The new transaction also uses the same access mode (READ WRITE or READ ONLY) as the just-terminated transaction. The RELEASE clause causes the server to disconnect the current client session after terminating the current transaction. Including the NO keyword suppresses CHAIN or RELEASE completion, which can be useful if the completion_type system variable is set to cause chaining or release completion by default.

`AND CHAIN` 子句在当前事务结束后开启一个新事务，并且新事务与刚刚结束的事务具有相同的隔离级别。 新事务还使用与刚刚结束的事务相同的访问模式（`READ WRITE` 或 `READ ONLY`）。`RELEASE` 子句使服务器在结束当前事务后断开当前客户端会话。包含 `NO` 关键字会抑制 `CHAIN` 或 `RELEASE` 完成，如果将 `completion_type` 系统变量被默认设置为连续链接或释放完成，则此选项将会很有用。

Beginning a transaction causes any pending transaction to be committed. See Section 13.3.3, “Statements That Cause an Implicit Commit”, for more information.

开始事务会导致提交的事务被挂起。 有关更多信息，参照[章节 13.3.3, “Statements That Cause an Implicit Commit”](https://dev.mysql.com/doc/refman/5.6/en/implicit-commit.html)。

Beginning a transaction also causes table locks acquired with LOCK TABLES to be released, as though you had executed UNLOCK TABLES. Beginning a transaction does not release a global read lock acquired with FLUSH TABLES WITH READ LOCK.

开始事务还会导致使用 `LOCK TABLES` 获取的表锁被释放，就像已经执行 `UNLOCK TABLES` 一样。 开始事务不会释放使用 `FLUSH TABLES WITH READ LOCK` 获取的全局读锁定。

For best results, transactions should be performed using only tables managed by a single transaction-safe storage engine. Otherwise, the following problems can occur:

为获得最佳结果，应该使用由单个事务安全存储引擎管理的表来执行事务。 否则，可能会出现以下问题：

If you use tables from more than one transaction-safe storage engine (such as InnoDB), and the transaction isolation level is not SERIALIZABLE, it is possible that when one transaction commits, another ongoing transaction that uses the same tables will see only some of the changes made by the first transaction. That is, the atomicity of transactions is not guaranteed with mixed engines and inconsistencies can result. (If mixed-engine transactions are infrequent, you can use SET TRANSACTION ISOLATION LEVEL to set the isolation level to SERIALIZABLE on a per-transaction basis as necessary.)

如果使用来自多个事务安全存储引擎（如 InnoDB）的表，并且事务隔离级别不为 `SERIALIZABLE`，则在提交一个事务时，使用相同表的另一个正在进行的事务可能看到第一个事务所做的一些更改。也就是说，混合引擎无法保证事务的原子性，并可能导致不一致。（如果混合引擎事务不常使用，则可以根据需要使用 `SET TRANSACTION ISOLATION LEVEL` 将隔离级别设置为按事务可序列化。）

If you use tables that are not transaction-safe within a transaction, changes to those tables are stored at once, regardless of the status of autocommit mode.

如果在事务中使用非事务安全的表，则无论自动提交模式的状态如何，都会立即存储对这些表的更改。

If you issue a ROLLBACK statement after updating a nontransactional table within a transaction, an ER_WARNING_NOT_COMPLETE_ROLLBACK warning occurs. Changes to transaction-safe tables are rolled back, but not changes to nontransaction-safe tables.

如果在更新事务中的非事务表后发出 `ROLLBACK` 语句，则会发生 `ER_WARNING_NOT_COMPLETE_ROLLBACK` 警告。将回滚对事务安全表的更改，但不会回滚对非事务安全表的更改。

Each transaction is stored in the binary log in one chunk, upon COMMIT. Transactions that are rolled back are not logged. (Exception: Modifications to nontransactional tables cannot be rolled back. If a transaction that is rolled back includes modifications to nontransactional tables, the entire transaction is logged with a ROLLBACK statement at the end to ensure that modifications to the nontransactional tables are replicated.) See Section 5.4.4, “The Binary Log”.

在 `COMMIT` 时，每个事务都存储在一个二进制日志块中。回滚的事务不会被记录。（例外：无法回滚对非事务表的修改。如果回滚的事务包括对非事务表的修改，则在末尾使用ROLLBACK语句记录整个事务，以确保复制对非事务性表的修改。）参照[章节 5.4.4, “The Binary Log”](https://dev.mysql.com/doc/refman/5.6/en/binary-log.html)。

You can change the isolation level or access mode for transactions with the SET TRANSACTION statement. See Section 13.3.6, “SET TRANSACTION Syntax”.

可以使用 `SET TRANSACTION` 语句更改事务的隔离级别或访问模式。 参照[章节 13.3.6, “SET TRANSACTION Syntax”](https://dev.mysql.com/doc/refman/5.6/en/set-transaction.html)。

Rolling back can be a slow operation that may occur implicitly without the user having explicitly asked for it (for example, when an error occurs). Because of this, SHOW PROCESSLIST displays Rolling back in the State column for the session, not only for explicit rollbacks performed with the ROLLBACK statement but also for implicit rollbacks.

回滚可能是一种缓慢的操作，可能会在没有用户明确要求的情况下隐式发生（例如发生错误时）。因此 `SHOW PROCESSLIST` 在会话的 `State` 列中显示回滚，不仅适用于 `ROLLBACK` 语句执行的显式回滚，还用适用于隐式回滚。

**Note 注意**

In MySQL 5.6, BEGIN, COMMIT, and ROLLBACK are not affected by --replicate-do-db or --replicate-ignore-db rules.

在 `MySQL 5.6` 中，`BEGIN`，`COMMIT` 和 `ROLLBACK` 不受 `--replicate-do-db` 或 `--replicate-ignore-db` 规则的影响。

## 原文

[13.3.1 START TRANSACTION, COMMIT, and ROLLBACK Syntax](https://dev.mysql.com/doc/refman/5.6/en/commit.html)

****
**THE END [ 2019-03-12 ]**