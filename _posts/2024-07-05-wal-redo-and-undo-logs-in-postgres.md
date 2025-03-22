---
title: "Part 5: Database Engineering Fundamentals: WAL, Redo and undo logs in postgres"
author: pravin_tripathi
date: 2024-07-05 09:00:00 +0530
readtime: true
img_path: /assets/img/database-engineering-fundamental-part-5/
categories: [Blogging, Article]
mermaid: true
permalink: /database-engineering-fundamental-part-5/wal-redo-and-undo-logs-in-postgres/
parent: /database-engineering-fundamental-part-5/
tags: [softwareengineering, backenddevelopment,database-engineering]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Copilot
---
# WAL, Redo and undo logs in postgres

In PostgreSQL, **Write-Ahead Logging (WAL)**, **Redo logs**, and **Undo logs** are crucial components for maintaining data integrity, supporting crash recovery, and providing consistency in database operations. Let’s break these concepts down in detail.

### 1. **Write-Ahead Logging (WAL)**

**Write-Ahead Logging (WAL)** is a fundamental mechanism in PostgreSQL to ensure data durability, consistency, and crash recovery. The core idea behind WAL is that before any changes are written to the actual data files (e.g., tables, indexes), those changes must first be written to a special log file (the WAL log). This guarantees that even if the system crashes before the changes are fully written to disk, the changes can still be recovered.

### Key Aspects of WAL:

- **Durability**: Once a change is logged in the WAL, PostgreSQL guarantees that it will be applied to the database, even in the event of a crash. This is part of the ACID properties of a transaction (Atomicity, Consistency, Isolation, Durability).
- **Log Structure**: The WAL log records all changes made to the database at the level of individual operations. Each WAL entry contains enough information to reconstruct the changes made to a database record (e.g., inserting a row, updating a value, etc.).
- **Sequential Write Optimization**: WAL logs are written sequentially to a log file, which makes writes faster because random disk I/O (which is common when modifying data pages) is minimized. PostgreSQL writes data pages to disk asynchronously and in the background, while the WAL log ensures that any change can be recovered.
- **WAL Files**: WAL logs are stored in segments, and these segments are named with an identifier based on the timeline of the transaction. Once the WAL log is written to disk, it’s referred to as a WAL "segment." These segments can be archived for long-term recovery purposes.
- **Log Shipping/Replication**: WAL logs are the key to implementing replication in PostgreSQL. Changes recorded in the WAL are streamed to replicas in real-time, allowing replicas to keep up with the primary server and ensure they have the latest state of the database.

### Basic Workflow:

1. A transaction starts.
2. Before any changes to the database are made (e.g., modifying a table row), the changes are first written to the WAL log.
3. The actual database file is updated in the background, but the WAL log entry ensures that the change can be recovered if necessary.
4. After the changes are written to both the WAL and the data files, PostgreSQL acknowledges that the transaction is complete (commit).

### 2. **Redo Logs in PostgreSQL**

The **redo log** is part of the WAL mechanism. It is the sequence of log entries that records changes that need to be re-applied (or “redone”) to the database during recovery in case of a crash.

When PostgreSQL starts after a crash, it will **replay** the redo log to bring the database back to the last consistent state. The redo log entries store information about every committed transaction, which means that if the system crashes after a transaction is committed but before the changes are fully written to the data files, the redo log will ensure the changes are applied to the database files once PostgreSQL restarts.

### Key Points about Redo Logs:

- The redo log contains the necessary information to re-apply changes that have been made to the database, i.e., all committed changes.
- The redo log is sequential, meaning that PostgreSQL can replay the logs in the order they were written to ensure a consistent state.
- Redo logs are a key part of **crash recovery**. After a crash, the recovery process replays the logs from the last consistent state to the most recent transaction to bring the system back up.

The **redo process** involves reading the WAL logs that were written since the last checkpoint and applying them to the database files.

### Example:

- A transaction inserts a row into a table.
- The WAL log records that the insert operation has occurred.
- If the system crashes before the data is written to disk, PostgreSQL can use the redo log to replay that insert operation when it restarts.

### 3. **Undo Logs in PostgreSQL**

In PostgreSQL, **Undo logs** (which are also often associated with rollback operations) are **not explicitly used** in the same way they are in some other database systems. Unlike databases that use undo logs to roll back transactions (like Oracle or MySQL's InnoDB), PostgreSQL typically uses a **multi-version concurrency control (MVCC)** system to handle transaction isolation and rollbacks.

However, PostgreSQL does handle transaction rollbacks and guarantees that all changes made during a transaction can be undone, but this is achieved through its MVCC mechanism, rather than by explicitly writing undo logs.

### MVCC Overview:

- PostgreSQL maintains multiple versions of a row. When a transaction modifies a row, the old version of the row is preserved, allowing other transactions to see the previous state (depending on isolation level).
- If a transaction is rolled back, the database doesn't need an undo log. Instead, it simply removes any versions of rows created by the transaction and allows the original versions to be accessed.

This means that when a transaction is aborted or rolled back, PostgreSQL **undoes** the changes by discarding the changes made in the current transaction. Since each version of a row is tracked by the transaction ID and the system handles this with visibility rules, there is no need for traditional undo logs as seen in other databases.

### Example of MVCC:

- A transaction updates a row.
- The update is written as a new version of the row, and the old version is kept.
- If the transaction commits, the new version of the row becomes visible to other transactions.
- If the transaction is rolled back, the new version is discarded, and the old version becomes visible again.

### Recovery Process

In PostgreSQL, the recovery process after a crash involves the following steps:

1. **Start the Database**: PostgreSQL begins by checking the last checkpoint in the WAL logs. A checkpoint is a point at which all data changes are guaranteed to have been written to disk.
2. **Replay WAL (Redo Logs)**: After identifying the last checkpoint, PostgreSQL replays the WAL logs since that checkpoint to ensure that all committed transactions are applied to the database, even if they were not yet written to disk at the time of the crash.
3. **Undo Uncommitted Transactions**: Any uncommitted transactions at the time of the crash will be discarded (this is handled by the MVCC system, not undo logs).
4. **Recovery Complete**: Once the redo and undo processes are completed, the database is back in a consistent state.

### Conclusion

- **WAL** ensures durability and crash recovery by logging every change made to the database before it is written to disk.
- **Redo logs** (part of WAL) ensure that committed changes can be reapplied during recovery.
- PostgreSQL does not explicitly use **undo logs** but uses its MVCC system to handle transaction rollbacks and concurrency control, maintaining old versions of rows instead of needing undo logs.

The combination of these mechanisms allows PostgreSQL to guarantee **ACID properties**, ensuring that data remains consistent and durable even in the event of crashes.

[Back to Parent Page]({{ page.parent }})