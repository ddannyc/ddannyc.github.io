---
title: PostgreSQL事务隔离层级
date: 2022-05-10 18:25:27
tags:
categories: translation
---

SQL标准定义了四种事务隔离级别。其中最严格的是串行级，标准中有一段是这样描述，以某种顺序一次执行一个事务，可以保证一组串行事务的任意并发执行产生相同的效果。其他三种级别根据现象定义，如并发交互的事务里，每个级别哪些现象必须不能出现。SQL标准中可以知道由于串行级的定义，所有这些现象都不会在这个级别出现。  
  
各个层级中禁止的现象：  
  
* dirty read(脏读)  
  
  一个事务里读取了另一个并发事务里修改后还未提交的数据。

* nonrepeatable read(不可复读)  
  
  一个事务里重读上次获取的数据时发现数据已经被修改。

* phantom read(幻读)  
  
  一个事务里使用上次有效的搜索条件再次执行查询时发现由于其他事务的提交导致搜索条件已经变更。
  
* serialization anomaly(串行异常)  
  
  同一组事务提交的结果不一致，是事务里所有可能的顺序执行后的结果。
  
SQL标准和PostgreSQL-实现事务隔离层级如 表13.1  
  
## 表 13.1. 事务隔离层级

| 隔离层级     | 脏读         | 不可复读     | 幻读         | 串行异常     |
| ----------- | ----------- | ----------- | ----------- | ----------- |
| Read uncommitted | 允许，PG不允许 | 可能 | 可能 | 可能 |
| Read committed | 不可能 | 可能 | 可能 | 可能 |
| Repeatable read |不可能 | 不可能 | 允许，PG不允许 | 可能 |
| Serializable | 不可能 | 不可能 | 不可能 | 不可能 |
  
在PostgreSQL中，你可以使用标准事务隔离层级中的任意一种，但内部只实现其中唯一的三种隔离层级。PostgreSQL的Read Uncommitted模式的表现和Read Committed差不多。因为这是PostgreSQL的多版本并发架构实现标准事务隔离层级唯一合理的方式。  
  
表格也显示PostgreSQL的Repeatable Read实现不允许幻读。SQL标准中定义更严格的行为控制是允许的：四种隔离层级只定义了必定不能发生的现象，而不是必定会发生的现象。可用的隔离层级行为在下面的小节里会详细介绍到。  
  
要为一个事务设置事务隔离层级可以使用`SET TRANSACTION`命令。
  
## 强调
一些PostgreSQL数据类型和函数对于事务性行为会有特殊的规则。通常，对于sequence的变更在其他事务里是立即可见并且即使事务变更终止也不会回滚。见[Section 9.17](https://www.postgresql.org/docs/current/functions-sequence.html)和[Section 8.1.4](https://www.postgresql.org/docs/current/datatype-numeric.html#DATATYPE-SERIAL)。
  
## 13.2.1. Read Committed Isolation Level
Read Committed是PostgreSQL默认的隔离层级。当一个事务使用这种隔离层级，一个SELECT查询(不带FOR UPDATE/SHARE clause)只能看到查询开始之前提交的数据；决不会看到未提交的数据或者查询执行期间其它并发事务提交的修改。实际上，一个SELECT查询看到的是查询开始执行时候数据库的快照。尽管如此，SELECT还是可以看到自己事务中还未提交的更新。还要注意的是即使在一个事务里两个连续SELECT命令也可能看到不一样的数据，如果其他的事务在第一个SELECT之后第二SELECT之前提交了数据变更。  
  
UPDATE，DELETE，SELECT FOR UPDATE和SELECT FOR SHARE命令搜索目标行时的表现是一样的：他们仅查找命令执行前已提交的数据。尽管如此，这些查找到的目标行有可能被其他并发事务修改（或删除或加锁）。在这种情况里，would-be更新器会等待第一个更新事务提交或者回滚。如果第一个更新回滚，那么记录没有影响第二个更新可以继续更新初始记录。如果第一个更新成功提交，当该记录被删除第二个更新则忽略该记录，否则就尝试对更新之后的记录执行操作。搜索条件会重新执行来检查更新以后的记录是否仍然满足。如果满足，第二个更新将继续在更新后的记录上执行操作。在SELECT FOR UPDATE和SELECT FOR SHARE的例子里，意味着更新后的记录被锁住并返回给客户端。  
  
INSERT包含ON CONFLICT DO UPDATE子句表现类似。在Read Committed模式里，被操作的每行记录要么插入要么更新。除非其他不相关的错误出现，否则可以保证只有这两种结果产生。如果另一个事务产生了还未可见的冲突，则UPDATE子句会在该行上执行操作，虽然可能通常都看不到这样的记录。  
  
INSERT包含ON CONFLICT DO NOTHING子句可能会因为别的事务的结果在当前操作尚不可见而导致插入操作无法进行。再次说明，这只是在Read Committed模式里是如此。  
  
由于以上规则，可能会出现更新的时候看到了不一致的快照：当一个并发操作尝试更新同一行时，该行影响是可见的，但是其它行的影响是不可见的。这样的行为使得Read Committed模式对于涉及到复杂搜索条件的命令不太合适；尽管如此，对于简单的情况是没有问题的。举例，考虑以下更新银行账户的事务：
```sql
BEGIN;
UPDATE accounts SET balance = balance + 100.00 WHERE acctnum = 12345;
UPDATE accounts SET balance = balance - 100.00 WHERE acctnum = 7534;
COMMIT;
```
如果两个并发事务尝试修改账户12345的余额，我们当然希望第二个事务基于更新后的版本去执行。因为每个命令只影响预设的行，允许更新后的版本可见不会产生任何不一致的问题。  
  
在Read Committed模式里更复杂的用法会产生非期望的结果。举例，一条删除命令的条件数据限制于另外一条命令对于该数据的增减操作。假设website是一张包含website.hits等于9和10的两行表格。：
```sql
BEGIN;
UPDATE website SET hits = hits + 1;
-- run from another session:  DELETE FROM website WHERE hits = 10;
COMMIT;
```
即使在UPDATE之前和之后存在website.hits = 10的行，DELETE操作没有生效。因为预更新时value为9的行直接跳过，而value为10的行在更新操作完成删除操作获取到该行的锁时，该行的值已经是11了，所以没有匹配的条件。  
  
因为Read Committed模式在每条命令执行前会建立一个新的包含了所有已提交事务的快照，所以任何情况下相同事务里的子命令可以看到并发事务里提交完成的变更。这是上面问题的关键，一条命令能否在数据库里看到绝对一致的视图。  
  
Read Committed事务隔离模式适合大多数应用，因为它简单易用且速度快。尽管如此，它也不能满足所有情况。那些需要做复杂查询和更新的应用可能要求比Read Committed模式更严格的数据库视图一致性。

## 13.2.2. Reapeatable Read Isolation Level
Reapeatable Read隔离层级只能看见事务开始前提交的数据；决不会看到未提交的数据或者查询执行期间其它并发事务提交的修改。(尽管如此，SELECT还是可以看到自己事务中还未提交的更新。)这项保证比SQL标准要求的隔离层级更强，它防止了[表13.1](#表-13-1-事务隔离层级)除串行异常以外的所有现象。如前所述，这是标准里明确允许的，SQL标准里只描述了每个事务隔离层级必须提供的最小保护。  
  
这层不同于Read Committed，repeatable read事务里一个查询的快照起始于第一条非事务控制语句，而不是起始于事务里当前的语句。因此单个事务里连续的SELECT命令可以看见相同的数据，不会看见自己事务开始以后其他事务提交的变更。  
  
应用使用这层模式必须为串行失败事务重试做准备。  
  
UPDATE，DELETE，SELECT FOR UPDATE和SELECT FOR SHARE命令搜索目标行时的表现是一样的：他们仅查找命令执行前已提交的数据。尽管如此，这些查找到的目标行有可能被其他并发事务修改（或删除或加锁）。在这种情况里，repeatable transaction会等待第一个更新事务提交或者回滚。如果第一个更新回滚，那么记录没有影响第二个更新可以继续更新初始记录。如果第一个更新成功提交(更新、删除或锁定)那么repeatable read事务回滚，回滚信息是：  
```
ERROR:  could not serialize access due to concurrent update
```
因为repeatable read事务开始之后其他事务就不能修改或者锁定这些行了。  
  
当应用接收到这条错误信息，应该中断当前事务并从起始的位置重试整个事务。通过第二次，事务可以看到上一次提交的变更并作为本次事务初始的数据库视图，所以使用新版本行的事务更新没有逻辑上的冲突。  
  
值得注意的是只有写事务才可能需要重试；只读事务不存在串行冲突。  
  
Repeatable Read模式严格保证每个事务看到的都是数据库稳定视图。尽管如此，这种一致性在相同层级以某种顺序执行的并发事务里不是那么必要。举例，这层模式里即使一个只读事务可能会出现控制记录告诉我们一个批处理已完成，但是却看不到批处理逻辑部分的详细记录，因为它读的是控制记录之前的版本。如果不仔细得显式使用锁来防止冲突事务尝试强制将业务运行在这层隔离模式上大概率会出问题。  
  
Repeatable Read隔离级别是使用学术数据库文献中已知的技术以及在一些其他数据库产品中称为快照隔离的技术实现的。与使用降低并发性的传统锁定技术的系统相比，可能会观察到行为和性能的差异。其他一些系统可能提供Repeatable Read和Snapshot Isolation作为具有不同行为的隔离层级。数据库研究人员直到 SQL 标准制定之后才正式确定区分这两种技术的允许现象，并且不在本手册的范围内。
  
## 13.2.3. Serializable Isolation Level
  
串行隔离层级提供最严格的事务隔离。此层所有提交事务模拟串行执行；就像是事务一个接一个得执行，串行得，而非并发得。尽管如此，像Repeatable Read层级那样，应用使用此层隔离必须要准备好重试串行失败的事务。实际上，这层隔离层级工作方式和Repeatable Read完全一样，除了它需要监控那些可能引起串行事务因为执行顺序导致结果不一致的事务。监控操作除了一些监控开销和发现会引起串行异常的条件时将触发串行失败，不会引入任何Repeatable Read以外的阻塞。  
  
举例，考虑有一张表mytab，初始时包含：
```sql
 class | value
-------+-------
     1 |    10
     1 |    20
     2 |   100
     2 |   200
```
假设串行事务A计算：
```sql
SELECT SUM(value) FROM mytab WHERE class = 1;
```
然后插入一条新的记录以结果(30)作为value和class = 2。并发得，串行事务B计算：
```sql
SELECT SUM(value) FROM mytab WHERE class = 2;
```
得到结果300，将其和class = 1作为新的记录插入。然后两个事务尝试提交。如果事务都运行在Repeatable Read隔离层级，两者都将允许提交；但由于没有与结果一致的串行执行顺序，使用串行事务将允许一个事务提交其他事务则得到一下信息并回滚：
```
ERROR:  could not serialize access due to read/write dependencies among transactions
```
这是因为如果A在B之前执行，B计算sum得到330，而非300，同样得另一种顺序则导致A得到不同的sum值。  
  
当依赖串行事务来防止异常时，重要的是，从持久用户表读取的任何数据在读取它的事务成功提交之前都不会被视为有效。即使对于只读事务亦是如此，除了在可延迟只读事务中读取的数据一旦被读取就可     视为有效的，因为这样的事务在开始读取任何数据之前一直等到它可以获取保证没有此类问题的快照。在所有其他情况下，应用程序不得依赖于后来事务执行期间读取的结果；相反，应该重试事务直到成功。  
  
为保证真实的串行化PostgreSQL使用predicate locking，这意味着它可以决定当一个写操作影响了之前并发事务里读取时让保持锁定，让它先执行完成。在 PostgreSQL 中，这些锁不会导致任何阻塞，因此不会导致死锁。它们用于识别和标记那些可能导致串行异常事务之间的依赖关系。相反，一个想要确保数据一致性的Read Committed或Repeatable Read事务可能需要对整表加锁，这可能会阻止其他用户尝试使用该表，或者它可能使用 SELECT FOR UPDATE 或 SELECT FOR SHARE 这不仅会阻止其他事务，还会导致磁盘访问。  
  
Predicate locks在PostgreSQL，像其他大多数数据库系统，基于事务实际访问的数据。这些将以 SIReadLock 模式显示在 `pg_locks` 系统视图中。