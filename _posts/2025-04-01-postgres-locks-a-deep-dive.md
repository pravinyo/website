---
title: "Part 5: Database Engineering Fundamentals: Postgres Locks — A Deep Dive"
author: pravin_tripathi
date: 2025-03-01 04:00:00 +0530
readtime: true
img_path: /assets/img/database-engineering-fundamental-part-5/
attachment_path: /assets/document/attachment/database-engineering-fundamental-part-5/
categories: [Blogging, Article]
mermaid: true
permalink: /database-engineering-fundamental-part-5/postgres-locks-a-deep-dive/
parent: /database-engineering-fundamental-part-5/
tags: [softwareengineering, backenddevelopment]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Copilot
---
# Postgres Locks — A Deep Dive

![](https://miro.medium.com/v2/resize:fill:88:88/1*j-h09TiaKTgYsIvVAHPa4Q@2x.jpeg)

[Hussein Nasser](https://medium.com/@hnasr?source=post_page---byline--9fc158a5641c--------------------------------)

![](1_LxTYAGHFfSLRTyI4qO6TfA.png)

A VACUUM full can block a select, we will learn how and why in this blog and much more

I used to think database locks are two types, shared and exclusive. Readers acquire many shared locks on a resource (row, object or table) but only one writer can acquire an exclusive lock. If a writer has an exclusive lock no one can acquire shared locks and as a result no one can read (but the writer). When I started to dig into Postgres this binary view of locking changed completely and I understand why.

You see, in Postgres there are five lock categories and over 12 individual lock types. To be honest knowing which command obtains which lock is less relevant than knowing which command can conflict which command. By conflict here I mean commands block each other and can’t run concurrently.

In this blog I explore all types and categories of locks and at the end I show you the [Postgres Lock Conflicts tool](https://postgres-locks.husseinnasser.com/) I wrote that shows what commands conflict with each other because the list can get huge looking at a matrix.

All the information here are official Postgres [doc](https://www.postgresql.org/docs/current/explicit-locking.html) , [source code](https://github.com/postgres/postgres) and ad-hoc testing on the app.

# **Table Locks**

If you ask me what a table lock three years ago I would say its a lock you obtain on a table so no one can do anything on that table while you hold that lock. No inserts, updates or deletes or any DDLs. But that is further from the truth in Postgres. There are eight types of table locks in Postgres and transactions can have multiple table locks on the same table. Some of those locks conflict, some don’t. Let us explore them one by one. You can view the table locks in mode column in pg_locks table.

## **ACCESS EXCLUSIVE**

[ACCESS EXCLUSIVE](https://postgres-locks.husseinnasser.com/?pglock=AccessExclusiveLock) (or [*AccessExclusiveLock*](https://postgres-locks.husseinnasser.com/?pglock=AccessExclusiveLock) in the code) is the most aggressive table lock. If a command obtains this lock type on a table nothing can be done to this table as this lock type conflicts with all other table locks. You can’t do DMLs so select, update or delete rows are blocked. You can’t do DDLs, alter a column, create an index and even system operation such as [VACUUM](https://postgres-locks.husseinnasser.com/?pgcommand=VACUUM) cannot execute on the table. It is a complete block.

What operations and commands in Postgres obtain this lock? I went through the doc pages and found all commands that obtain this kind of lock. Here they are, when anything in this list run, you can’t do anything to this table.

```
--LIST OF POSTGRES COMMANDS THAT OBTAIN ACCESS EXCLUSIVE TABLE LOCK
DROP TABLE
TRUNCATE
REINDEX
CLUSTER
VACUUM FULL
REFRESH MATERIALIZED VIEW
ALTER INDEX SET TABLESPACE
ALTER INDEX ATTACH PARTITION
ALTER INDEX SET FILLFACTOR
ALTER TABLE ADD COLUMN
ALTER TABLE DROP COLUMN
ALTER TABLE SET DATA TYPE
ALTER TABLE SET/DROP DEFAULT
ALTER TABLE DROP EXPRESSION
ALTER TABLE SET SEQUENCE
ALTER TABLE SET STORAGE
ALTER TABLE SET COMPRESSION
ALTER TABLE ALTER CONSTRAINT
ALTER TABLE DROP CONSTRAINT
ALTER TABLE ENABLE/DISABLE RULE
ALTER TABLE ENABLE/DISABLE ROW LEVEL SECURITY
ALTER TABLE SET TABLESPACE
ALTER TABLE RESET STORAGE
ALTER TABLE INHERIT PARENT
ALTER TABLE RENAME
```

This means for example, if you run [VACUUM FULL](https://postgres-locks.husseinnasser.com/?pgcommand=VACUUM+FULL) for instance on a table, you can’t select or update to this table.

> You might say why did you have to spell out different methods on Alter table, the reason because different Alter tables obtain different types of locks which makes some alters block certain operations while others might not.
> 

## **ACCESS SHARE**

[ACCESS SHARE](https://postgres-locks.husseinnasser.com/?pglock=AccessShareLock) (or [*AccessShareLock*](https://postgres-locks.husseinnasser.com/?pglock=AccessShareLock)) is the lightest weight lock type. Only two commands that I’m aware of acquire this lock and those are [SELECT](https://postgres-locks.husseinnasser.com/?pgcommand=SELECT) and [COPY TO](https://postgres-locks.husseinnasser.com/?pgcommand=COPY+TO). It is an indication that someone is reading the table, whether it is a single row, all the rows and yes even no rows if you do a query on a table that returned nothing, that lock is also acquired (I tested it).

```
--LIST OF POSTGRES COMMANDS THAT OBTAIN ACCESS SHARE TABLE LOCK
SELECT
COPY TO
```

This lock type only conflicts with the [ACCESS EXCLUSIVE](https://postgres-locks.husseinnasser.com/?pglock=AccessExclusiveLock) which makes it easy to understand, if run a transaction that does a select you can’t do a VACUUM FULL (normal VACUUM is fine). The reason is [VACUUM FULL](https://postgres-locks.husseinnasser.com/?pgcommand=VACUUM+FULL) or any of the commands that acquire ACCESS EXCLUSIVE does sergical changes to the layout of the table which will break consistency when selects are running. For example VACUUM FULL actually changes tuple ids, purging tables and reshuffling data, we can’t be having people reading the table while this is happening. This is as opposed to normal VACUUM which really (almost) act like both update and deletes.

## **EXCLUSIVE**

The [EXCLUSIVE](https://postgres-locks.husseinnasser.com/?pglock=ExclusiveLock) (or [ExclusiveLock](https://postgres-locks.husseinnasser.com/?pglock=ExclusiveLock)) is very similar to the ACCESS EXCLUSIVE except it doesn’t conflict with reads acquired by ACCESS SHARE. This means you can do selects while an EXCLUSIVE table lock is on the table.

The odd thing I only found one command ([REFRESH MATERIALIZED VIEW CONCURRENTLY](https://postgres-locks.husseinnasser.com/?pgcommand=REFRESH+MATERIALIZED+VIEW+CONCURRENTLY)) that acquires this lock. If I had to guess, this lock type was added because people wanted a way to refresh their materialized views and select from the table at the same time. The Refresh materialized view acquires an [ACCESS EXCLUSIVE](https://postgres-locks.husseinnasser.com/?pglock=AccessExclusiveLock) blocking selects, so Postgres added both a new lock type Exclusive which conflicts with everything except [ACCESS SHARE](https://postgres-locks.husseinnasser.com/?pglock=AccessShareLock) and then made a new command to allow refreshing the view concurrently. I’m sure more methods will fit into this slot lock type.

```
--LIST OF POSTGRES COMMANDS THAT OBTAIN EXCLUSIVE lock
REFRESH MATERIALIZED VIEW CONCURRENTLY
```

So you if you refresh your materialized view concurrently your table can’t be edited but can be read.

## **ROW SHARE**

Buckle up the names only get more confusing from here on. [ROW SHARE](https://postgres-locks.husseinnasser.com/?pglock=RowShareLock) (or [RowShareLock](https://postgres-locks.husseinnasser.com/?pglock=RowShareLock)) is similar to ACCESS SHARE but was designed for the SELECT FORs command family. That is probably why it has the name ROW in it. While [SELECT FOR UPDATE](https://postgres-locks.husseinnasser.com/?pgcommand=SELECT+FOR+UPDATE), [SELECT FOR SHARE](https://postgres-locks.husseinnasser.com/?pgcommand=SELECT+FOR+SHARE) and others work on rows remember this is still a table lock. So these kind of commands actually acquire two types of locks row locks (will see later) and the ROW SHARE row locks.

This lock type conflicts with ACCESS EXCLUSIVE and the [EXCLUSIVE](https://postgres-locks.husseinnasser.com/?pglock=ExclusiveLock) lock. Which means anything that is acquired by the ACCESS EXCLUSIVE + our refresh materialized view concurrently.

Here is a list of commands that acquire ROW SHARE.

```
--LIST OF POSTGRES COMMANDS THAT OBTAIN ROW SHARE table lock
SELECT FOR UPDATE
SELECT FOR NO KEY SHARE
SELECT FOR SHARE
SELECT FOR KEY SHARE
```

So while true you can do normal SELECTs while refreshing your materialized view concurrently, you can’t really do a [SELECT FOR SHARE](https://postgres-locks.husseinnasser.com/?pgcommand=SELECT+FOR+SHARE) for instance.

## **ROW EXCLUSIVE**

The [ROW EXCLUSIVE](https://postgres-locks.husseinnasser.com/?pglock=RowExclusiveLock) (or [RowExclusiveLock](https://postgres-locks.husseinnasser.com/?pglock=RowExclusiveLock)) is obtained by DMLs ([Insert](https://postgres-locks.husseinnasser.com/?pgcommand=INSERT), [Update](https://postgres-locks.husseinnasser.com/?pgcommand=UPDATE+%28KEYS%29), [Delete](https://postgres-locks.husseinnasser.com/?pgcommand=DELETE), [Merge](https://postgres-locks.husseinnasser.com/?pgcommand=MERGE) and [Copy From](https://postgres-locks.husseinnasser.com/?pgcommand=COPY+FROM). If you care about write latency to your table watch out for operations that conflict with this lock. Thus the name ROW in the lock name because methods often operates on rows.

Gotta watch out again, you might you a update or a delete that end touching no rows, the ROW EXCLUSIVE lock is still acquired. So if you have long running transactions watch out for blocks.

The methods acquiring ROW EXCLUSIVE are as follows

```
--LIST OF POSTGRES COMMANDS THAT OBTAIN ROW EXCLUSIVE table lock
UPDATE
DELETE
INSERT
MERGE
COPY FROM
```

## **SHARE ROW EXCLUSIVE**

[SHARE ROW EXCLUSIVE](https://postgres-locks.husseinnasser.com/?pglock=ShareRowExclusiveLock) (or [ShareRowExclusiveLock](https://postgres-locks.husseinnasser.com/?pglock=ShareRowExclusiveLock)) is similar to [EXCLUSIVE](https://postgres-locks.husseinnasser.com/?pglock=ExclusiveLock) but is relaxed so that ROW SHAREs don’t conflict, so you may do the SELECT FORs with this type of lock but you still can’t do any modifications through DMLs. So methods that obtain this type of locks want to allow reads even the SELECT FORs but block writes. In case you had the question why not just use EXCLUSIVE that is why. It is all about relaxing and minimizing blocks.

What is interesting about SHARE ROW EXCLUSIVE is it does conflict with it self which means only one operation of this lock type can run so for instance you can’t run two create triggers (which is a method that acquire SHARE ROW EXCLUSIVE) at the same time on the same table, my guess is one create trigger might modify rows in the table and the other create trigger might also change the table and we don’t want that.

> Again remember if a transaction obtains a SHARE ROW EXCLUSIVE it can still make modifications to rows, other transactions can’t.
> 

Here are the methods that obtain this type.

```
--LIST OF POSTGRES COMMANDS THAT OBTAIN SHARE ROW EXCLUSIVE table lock
CREATE TRIGGER
ALTER TABLE ADD FOREIGN KEY
ALTER TABLE ENABLE/DISABLE TRIGGER
```

## **SHARE**

The [SHARE](https://postgres-locks.husseinnasser.com/?pglock=ShareLock) (ShareLock) is similar to the [SHARE ROW EXCLUSIVE](https://postgres-locks.husseinnasser.com/?pglock=ShareRowExclusiveLock) in a sense it blocks concurrent modifications but it doesn’t conflict with itself. Only [CREATE INDEX](https://postgres-locks.husseinnasser.com/?pgcommand=CREATE+INDEX) obtain this type of lock, which means while creating an index you can’t change the data (because the index is reading the table and building the b+tree) but technically nothing stopping 7 different transactions from running 7 CREATE INDEX on the same table, so if you are blocking writes to create indexes, you can technically create them all at the same time. Of course you can also use the CREATE INDEX Concurrently which allows concurrent modifications but that doesn’t run in a transaction.

![](1_8rtdhmbP9oNLonbM1npiCw.png)

```
--LIST OF POSTGRES COMMANDS THAT OBTAIN SHARE table lock
CREATE INDEX
```

## **SHARE UPDATE EXCLUSIVE**

The [SHARE UPDATE EXCLUSIVE](https://postgres-locks.husseinnasser.com/?pglock=ShareUpdateExclusiveLock) (or [ShareUpdateExclusiveLock](https://postgres-locks.husseinnasser.com/?pglock=ShareUpdateExclusiveLock)) is designed for methods who want to allow concurrent writes and reads but prevent schema changes and VACUUM runs. Normal VACUUM for instance acquire this lock which is why you can run VACUUM and still do edit to your table otherwise it will be a disaster.

[CREATE INDEX CONCURRENTLY](https://postgres-locks.husseinnasser.com/?pgcommand=CREATE+INDEX+CONCURRENTLY) is another interesting one where you can create an index and allow writes. This lock conflict with itself so no two VACUUMs can run concurrently and no two CREATE INDEX CONCURRENTLY as well. This also explains why many forms of ALTER TABLE commands acquire this type of lock, you want to allow edits but no two alters at the same time.

Following are the commands that acquire SHARE UPDATE EXCLUSIVE.

```
--LIST OF POSTGRES COMMANDS THAT OBTAIN SHARE UPDATE EXCLUSIVE table lock
VACUUM
REINDEX CONCURRENTLY
CREATE STATISTICS
CREATE INDEX CONCURRENTLY
COMMENT ON
ANALYZE
ALTER TABLE VALIDATE CONSTRAINT
ALTER TABLE SET WITHOUT CLUSTER
ALTER TABLE SET TOAST
ALTER TABLE SET STATISTICS
ALTER TABLE SET N_DISTINCT
ALTER TABLE SET FILLFACTOR
ALTER TABLE SET AUTOVACUUUM
ALTER TABLE DETACH PARTITION
ALTER TABLE CLUSTER ON
ALTER TABLE ATTACH PARTITION (PARENT)
ALTER INDEX (RENAME)
```

## **Table Lock Matrix**

This is the matrix from the [doc](https://www.postgresql.org/docs/15/explicit-locking.html), it helps us understand what locks conflicts with what.

|                    | ACCESS SHARE | ROW SHARE | ROW EXCL. | SHARE UPDATE EXCL. | SHARE | SHARE ROW EXCL. | EXCL. | ACCESS EXCL. |
|--------------------|--------------|-----------|-----------|--------------------|-------|-----------------|-------|--------------|
| ACCESS SHARE       |              |           |           |                    |       |                 |       |       X      |
| ROW SHARE          |              |           |           |                    |       |                 |   X   |       X      |
| ROW EXCL.          |              |           |           |                    |   X   |        X        |   X   |       X      |
| SHARE UPDATE EXCL. |              |           |           |          X         |   X   |        X        |   X   |       X      |
| SHARE              |              |           |     X     |          X         |       |        X        |   X   |       X      |
| SHARE ROW EXCL.    |              |           |     X     |          X         |   X   |        X        |   X   |       X      |
| EXCL.              |              |     X     |     X     |          X         |   X   |        X        |   X   |       X      |
| ACCESS EXCL.       |       X      |     X     |     X     |          X         |   X   |        X        |   X   |       X      |

To me however the methods and commands that acquire the locks are more important than the locks themselves. Which is why I wrote this [tool](https://postgres-locks.husseinnasser.com/?pglock=ShareRowExclusiveLock) to dynamically show which method’s conflicts with what commands and what commands are allowed concurrently. You will see it referenced all over this blog.

> Table locks are in memory and can be retrieved from the pg_locks view. The memory requirements for table locks are low because they are coarser compared to row locks. Which we will discuss later.
> 

Here is an example from the [Postgres Lock Conflicts](https://postgres-locks.husseinnasser.com/) tool on VACUUM.

![](1_ihD9J_6RMjXbDGs5nfxJ_Q.png)

![](1_yLTFavp48E3WPgEDCjcduQ.png)

# **Row Locks**

Now that we talked about table locks time to go one level deeper to row locks. Row locks are critical to detect changes to row objects so we prevent two transactions changing the same row which results in lost updates.

Worth noting that INSERTed tuples don’t require row locks in postgres because they are only visible to the transaction that creates them. One reason probably why Postgres doesn’t support read uncommitted isolation level.

The methods that lock rows are limited to [DELETE](https://postgres-locks.husseinnasser.com/?pgcommand=DELETE), [UPDATE (NO KEY)](https://postgres-locks.husseinnasser.com/?pgcommand=UPDATE+%28NO+KEYS%29), [UPDATE (KEY)](https://postgres-locks.husseinnasser.com/?pgcommand=UPDATE+%28KEYS%29), and all the SELECT FORs.

UPDATE (NO KEY) is an update to a column that doesn’t have a unique index while UPDATE (KEY) is an update to a column that does have a unique index. Those two acquire different locks that is why they are spelled out.

Here are four row locks in Postgres we discuss them here.

## **FOR UPDATE**

[FOR UPDATE](https://postgres-locks.husseinnasser.com/?pglock=FORUPDATE) is the highest row lock, when a row is locked FOR UPDATE you cannot delete or update it or do a SELECT FOR UPDATE on it. However you can still read it through a normal SELECT, if you want your selects to be blocked if someone is touching a row you may use [SELECT FOR KEY SHARE](https://postgres-locks.husseinnasser.com/?pgcommand=SELECT+FOR+KEY+SHARE) instead which conflicts.

The following commands acquire a FOR UPDATE row lock.

```
--LIST OF POSTGRES COMMANDS THAT OBTAIN FOR UPDATE row lock
DELETE
UPDATE (KEY) -- UPDATE TO A COLUMN WITH A UNIQUE INDEX
SELECT
```

## **FOR NO KEY UPDATE**

This [lock](https://postgres-locks.husseinnasser.com/?pglock=FORNOKEYUPDATE) is acquired by UPDATES to columns without unique index, so it is weaker than FOR UPDATE as it allows SELECT FOR KEY SHARE.

```
--LIST OF POSTGRES COMMANDS THAT OBTAIN FOR NO KEY UPDATE ROW LOCK
UPDATE (NO KEY) -- UPDATE TO A COLUMN WITH NO INDEX OR REGULAR INDEX (NON-UNIQUE)
```

## **FOR SHARE**

This is the true [shared lock](https://postgres-locks.husseinnasser.com/?pglock=FORSHARE), transactions can acquire multiple FOR SHARE locks on a row. When a row is FOR SHAREd no DML can modify it.

```
--LIST OF POSTGRES COMMANDS THAT OBTAIN FOR SHARE
SELECT FOR SHARE
```

## **FOR KEY SHARE**

The [weakest row lock](https://postgres-locks.husseinnasser.com/?pglock=FORKEYSHARE), behaves like FOR SHARE but allows updates to columns without unique indexes.

```
--LIST OF POSTGRES COMMANDS THAT OBTAIN FOR SHARE
SELECT FOR KEY SHARE
```

## **Row Lock Matrix**

This matrix shows the 4 row locks and how they conflict with each other. The tool I wrote gives more visibility to the commands that block each other.

|                   | FOR KEY SHARE | FOR SHARE | FOR NO KEY UPDATE | FOR UPDATE |
|-------------------|---------------|-----------|-------------------|------------|
| FOR KEY SHARE     |               |           |                   |      X     |
| FOR SHARE         |               |           |         X         |      X     |
| FOR NO KEY UPDATE |               |     X     |         X         |      X     |
| FOR UPDATE        |       X       |     X     |         X         |      X     |

Postgres table locks are in memory, row locks are stored in the tuple (xmax system field), which saves memory at a cost of potential disk writes. This isn’t so bad for deletes or updates because we are technically touching the row but select for updates for instance, those read operations can now cause pages to get dirty which will trigger the background writer to flush them to disk.

> Row locks are memory intensive in other databases, I work with SQL Server almost on daily basis and if row locks are enabled SQL Server can easily run out of memory and fail the transaction when row locks can’t be acquired. That is why I appreciate the brilliance of Postgres row-lock designs. Nothing is free though, disk writes.
> 

# **Page locks**

A [postgres page](https://medium.com/@hnasr/database-pages-a-deep-dive-38cdb2c79eb5) is 8KB and stores tuples for table and indexes. Because the page is an in memory data structure, it needs to be protected from concurrent process accesses. You see, [Postgres design is process](https://medium.com/@hnasr/postgresql-process-architecture-f21e16459907) based which means when you connect to the database you get your own backend process, and those backend processes compete to access pages stored in shared buffer pool.

It is not so bad if multiple processes reading the same page, but it is a problem if we have two processes attempting to write to the same page. You want to serialize [those accesses](https://medium.com/@hnasr/threads-and-connections-in-backend-applications-a225eed3eddb) so you don’t corrupt data. This is classic operating system concepts with mutex and semaphores.

# **Dead Locks**

Dead locks happen when two transactions each holding different locks attempting to access each other’s resources and end up in an indefinite wait. Postgres detects dead locks and kills one of the transaction to move on.

Here is an example (while unlikely it can happen)

```
Tx1
BEGIN;
-- ACQUIRES AccessSharelock (OK
SELECT * FROM TEST
                                    Tx2
                                    BEGIN;
                                    -- ACQUIRES AccessSharelock (OK)
                                    SELECT * FROM TEST;
                                    -- Attempts to acquire
                                    -- AccessExlusiveLock get blocked by tx1
                                    ALTER TABLE ADD COLUMN A TEXT;
--Attempts to acquire Access
--Exclusive blocks by tx2
--dead lock
TRUNCATE TABLE TEST;

---DEAD LOCK X_X
```

Another example with row locks

```
Tx1
BEGIN;
-- ACQUIRES ShareRowLock (OK)
SELECT * FROM TEST
WHERE ID = 1
FOR SHARE
                                    Tx2
                                     BEGIN;
                                    -- ACQUIRES ShareRowLock (OK)
                                    SELECT * FROM TEST
                                    WHERE ID = 1
                                    FOR SHARE

                                    --Attempts to update the row (X)
                                    --blocked by Tx1 share lock
                                    UPDATE TEST SET V = 1
                                    WHERE ID = 1;

--Attempts to delete the row
--Blocked by Tx2 share lock
DELETE FROM TEST
WHERE ID = 1;

---DEAD LOCK X_X
```

Another dead lock example from the doc.

```
Tx1
BEGIN;
-- ACQUIRES FOR UPDATE lock
-- ON row 11111 (OK)
UPDATE accounts
SET balance = balance + 100.00
WHERE acctnum = 11111;                  Tx2
                                        BEGIN;
                                        -- ACQUIRES FOR UPDATE lock (OK)
                                        -- ON row 22222 (OK)
                                        UPDATE accounts
                                        SET balance = balance + 100.00
                                        WHERE acctnum = 22222;

                                        -- Attempts to acquire FOR UPDATE
                                        -- ON row 11111, blocked by tx1 (X)
                                        UPDATE accounts
                                        SET balance = balance - 100.00
                                        WHERE acctnum = 11111;

--Attempts to acquire FOR UPDATE
--On row 22222, blocked by tx2 (X)
UPDATE accounts
SET balance = balance - 100.00
WHERE acctnum = 22222;

---DEAD LOCK X_X
```

# **Advisory Locks**

Sometimes the application requirement makes the native MVCC locks insufficent. That is why most databases provide application-level locks that are controlled by the application to be held and released. While those locks still live in the database they are acquired and released by the application.

You might say why not just use FOR SHARE and FOR UPDATE to simulate that. The problem is you have to have a row to lock in those cases and you might be unnecessarily blocking other transactions from doing legitment modifications to that row. Advisory locks are obtained on integer values not rows or tables. Those numbers can come from columns of rows they don’t have to be.

Another reason why row locks don’t work is they are always tied to a transacation, if the transcation commits or rollsback the locks are gone and this is something your application might not want. Take for example a long — running operation in the app that does multiple database transactions, you want to prevent multiple users from running this long running operation concurrently. It is almost very hard to control that with just normal locks so what you can do is obtain a session level adversary lock when the operation starts so other users will attempt to also to obtain the same Advisory lock and get blocked.

There are two types of advisory locks, session and transcation. Session locks obtained with pg_advisory_lock () are kept for the length of the session (connection), while transaction advisory locks obtained with pg_advisory_xact_lock () are kept for the length of the current running transaction.

Here is an example.

```

-- Start Applicaiton Operation
-- Acquires a session lock
SELECT pg_advisory_lock(100);
--TxA1
BEGIN;
--DO WORK
COMMIT;

                                    -- Start Applicaiton Operation
                                    -- Attemps Acquires a session lock
                                    -- BLOCKS (X)
                                    SELECT pg_advisory_lock(100);

--TxA2
BEGIN;
--DO MORE WORK
COMMIT;
--Release session lock 100
SELECT pg_advisory_unlock(100);
-- End Applicaiton Operation
                                    -- TxB1, lock unblocks
                                    BEGIN;
                                    --DO MORE WORK
                                    COMMIT;
                                    -- TxB2

                                     BEGIN;
                                    --DO MORE WORK
                                    COMMIT;

                                    --Release session lock 100
                                    SELECT pg_advisory_unlock(100);

```

# **Weak Locks**

6/21/2023 — I added this additional paragraph to mention weak locks, something I recently learned in Postgres. Postgres has weak locks, those are table locks that rarely conflicts , acquired by DMLs, they are mainly AccessShareLock, RowShareLock, RowExclusiveLock.

Because they are common, and weak, Postgres manages them through a fast path a data structure in the process as oppose through the normal lock manager. But it can’t just use that data structure with no limit, it has a limit and that limit is 16 weak locks per backend process according to the constant [FP_LOCK_SLOTS_PER_BACKEND](https://github.com/postgres/postgres/blob/master/src/include/storage/proc.h#L85) which can’t be changed unless you recompile postgres alas. If you don’t know backend process == connection in postgres. So if your data model is heavily normalized OR you are using partitioning and your queries are scanning multiple partitioning in a long transaction watch out not to exceed that. otherwise you hit the lock manager and contention is created.

![](1_i7LdtmJ2-HlRYSo16yxYKg.png)

# **Summary**

Understanding Postgres locking will give you an edge to making better application design choices and better luck at trouble-shooting lock waiting, dead-locks and general latency when executing queries. Advisory locks can also be used to build interesting use cases that don’t work with normal MVCC locks. Hope you enjoyed this post.

If you like this content, consider checking out my [database course](https://database.husseinnasser.com/).

[Postgres+Locks+—+A+Deep+Dive-New.pdf]({{page.attachment_path}}/PostgresLocksADeepDive-New.pdf)

[Back to Parent Page]({{ page.parent }})