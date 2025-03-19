---
title: "Part 5: Database Engineering Fundamentals"
author: pravin_tripathi
date: 2025-03-04 01:00:00 +0530
readtime: true
img_path: /assets/img/database-engineering-fundamental-part-5/
attachment_path: /assets/document/attachment/database-engineering-fundamental-part-5/
categories: [Blogging, Article]
mermaid: true
permalink: /database-engineering-fundamental-part-5/
tags: [softwareengineering, backenddevelopment]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Copilot
---

## Database Security
- Enabling TLS and SSL in postgres.conf file
- don’t allow larger query as if around 14MB, it will crash server

## Best Practices Working with REST & Databases
- give different access to different user/client/application
- table own by one user should not be modified by another table.
- better to use database management tools

## Homomorphic Encryption
Why we can’t always encrypt?
- Database Queries can only be performed on plain text
- Analysis, Indexing, tuning
- Applications must read data to process it
- TLS Termination Layer 7 Reverse Proxies and Load Balancing

## Meet Homomorphic Encryption!
- Ability to perform arithmetic operations on encrypted data
- No need to decrypt!
- You can query a database that is encrypted!
- Layer 7 Reverse Proxies don’t have to terminate TLS, can route traffic based on rules without decrypting traffic
- Databases can index and optimize without decrypting data

Example,
https://github.com/IBM/fhe-toolkit-linux
it is not Production ready yet. It takes 2 mins to fetch record in smaller DB.

## Questions and Answers

### Heap index scan instead of index only scan?

It is possible that statistics is not updated which causes the database to think based on incorrect information. This statistics is used by database to plan the execution and decide the index that it will use to look for the required data.

In order to update statistics, we have to execute vacuum command in the database to trigger clean up of the unused page and also updation of the statistics.

### What is cost in the execution plan?

It is effort/cost required by Database to fetch the data. It is not in milliseconds but number. higher means more cost. There is a possibility that cost in execution plan can be higher than the execution plan. It is probably due to statistics not been update. Well we have to execute vacuum command.

### All Isolation levels

PostgreSQL supports several isolation levels that control the visibility of data changes to other transactions. The isolation levels define how transactions interact with each other when reading and writing data, ensuring consistency and correctness of operations in multi-user environments.

Here's a breakdown of the **four main isolation levels** in PostgreSQL:

1. **Read Uncommitted** (Not supported in PostgreSQL, but conceptually available in other DBMS like MySQL or SQL Server).
2. **Read Committed** (default in PostgreSQL).
3. **Repeatable Read**.
4. **Serializable**.

---

### 1. **Read Committed** (Default in PostgreSQL)

In the **Read Committed** isolation level, each query within a transaction sees a consistent view of the database, but data changes made by other transactions can be visible during the execution of the transaction. This means:

- If two transactions run concurrently, one transaction might see the changes made by the other **before** it commits.

**Example Scenario:**

- Transaction 1 starts and updates a record.
- Transaction 2 starts and reads the same record, seeing the changes made by Transaction 1, even though Transaction 1 hasn't committed yet.

### 2. **Repeatable Read**

In **Repeatable Read**, the database guarantees that any data read by a transaction will remain the same throughout the entire transaction. This prevents "non-repeatable reads," where a transaction could read different values for the same data if another transaction modifies it.

- **Phantom Reads** are still possible in this isolation level, which means that if new rows are added to the database, they might not be visible to the current transaction.

**Example Scenario:**

- Transaction 1 starts and reads a record.
- Transaction 2 starts and updates the same record.
- Transaction 1 reads the record again and sees the same value as initially read.

### 3. **Serializable**

The **Serializable** isolation level ensures the strictest consistency, where transactions are executed in a way that they could be serially ordered (i.e., one after the other) without conflict. This level prevents **dirty reads**, **non-repeatable reads**, and **phantom reads**.

- This isolation level effectively serializes access to the data, as if the transactions were run sequentially, ensuring that the final database state is the same as if the transactions were processed one by one, without overlap.

**Example Scenario:**

- Transaction 1 starts and reads a record.
- Transaction 2 starts and attempts to update the same record but is blocked until Transaction 1 is completed.

---

### Key Differences Between Isolation Levels

| **Isolation Level** | **Dirty Reads** | **Non-Repeatable Reads** | **Phantom Reads** | **Description** |
| --- | --- | --- | --- | --- |
| **Read Uncommitted** | Yes | Yes | Yes | Transactions can see uncommitted changes from other transactions. (Not supported in PostgreSQL) |
| **Read Committed** | No | Yes | Yes | Transactions see only committed data, but results might change between queries. |
| **Repeatable Read** | No | No | Yes | Transactions see consistent data, but new rows can be inserted, leading to phantom reads. |
| **Serializable** | No | No | No | Transactions are executed as if they were serially ordered, avoiding all anomalies. |

---

### Repeatable read vs Snapshot Isolation

In repeatable read, if max value in the table is queried to be found that it will behave different in below scenario,

If after query, some other transaction insert new record that changes the max score value, then next time execution of same query will return different value. so it is like a phantom read.

This problem is solved by snapshot isolation level, here it says read rows older than the time it stared the transaction so newer record will not be read. so execution of same query like above will return same value both times.

`In postgres, repeatable read is same as snapshot isolation. basically internal implementation is different.` due to versioning it by default solves the phantom read problem

### Postgres is using sequence scan instead of index scan on smaller table.

I am using postgresql. I have created a table with 4 columns (id, first_name, last_name, dob), and created an index on id which is a primary key as well. When i do select query like

`select id where id = 2`

It is using seq scan rather than index-only scan. What's the issue here?

This table has only 7 rows.

**Answer:**
Your last statement is why.

The 7 rows fits nicely in one page in the heap, so in postgres optimizer says hey. The entire table is one page I can just read that page directly instead of cracking up the id index and read the btree and go through the btree complex structure. My guess is the index is never loaded to memory at this stage too because the table is too small and not worth it.

Add another 10k rows and it the plan will change

### **Fetching page vs row in index**

where you explain how indexes and table are stored, you mentioned that when we execute sql query, behind the scenes the whole page is fetched and only the row needed is filtered and sent to user. I have 2 questions:

a) why does it get the whole page, why not just the row since the db now already knows the row location.

b) How does this work during index? I mean an index will point you to the exact row, so does it still fetches the whole page the row is in?

**Answer:**
A) that is how disks work , you can’t fetch one row you have to fetch a “block” this is unlike RAM where you can do byte level addressing. There are new SSDs called ultraram that allows for byte addressability which will allow us to do what you are saying but for now we have to fetch a page and we get multiple rows with it.

B) the index itself is stored on disk correct? And anything stored in disk has to be stored in pages :) so yes even the index read in pages and we get a bunch of row entries. Of course once a page is in memory you can pull any byte off it.

**Follow-up:**
it doesn't make sense to me that the index is stored on disk !! won't be better if the index is stored in memory for faster search & access?  idk, if the index is relatively small, won't it be stored in memory instead?

**Answer:**
the indexes are stored in memory but also stored on disk. otherwise if you shutdown the database you will have to recreate all indexes or all tables of all databases which is not feasible

furthermore you will run out of memory what do you do in that case? do you lose your index.

you need to store the index on disk and load it on memory for performance

### **Index on column with duplicate values**

If we are creating index on a column which can have duplicate values then how is it stored in index data structure?

Does it maintain a key with name as duplicate value and values would be list of rows and corresponding page information?

**Answer:**
Good question, databases implements this differently. Postgres for the longest time stored the duplicated values in the index for each key. Then later improved thati in postgres 13 by deduplicating it (k1-(row1,row2,row7)

Instead of k1-row1 / k1-row2 / k1-row7
---

Is indexing a boolean column useful, let's suppose we have a table which contains a processed flag, as rows are processed, this flag is marked as true/1. However, this table can contain huge number of rows and new rows are inserted with processed=false/0 for processing, so we need to query these columns periodically to process. Is it useful to add index on this column?

**Answer:**
this depends on the selectively of the boolean column.

say you have your processed field,

if we know that unprocessed rows (processed = 0 ) is FAR less than processed =1 say 1% of the rows are unprocessed then indexing is useful especially if you only query for unprocessed rows

but if you query for processed rows = 1 you know you will get massive rows and the index will not he as useful.

remember index is only useful if the rows that you predict to come back are small

## Deduplication of B-tree Indexes in PostgreSQL 13

In PostgreSQL 13, one of the significant changes related to indexing was the improvement in **B-tree index deduplication**. B-trees are a common data structure used for indexing in databases, and they are used by default in PostgreSQL for most index types. B-tree indexes organize data in a balanced tree structure that allows for fast search, insert, and delete operations.

Previously, when multiple identical values were inserted into a B-tree index (for example, inserting rows with the same key value), each of these duplicate values would take up space in the index, even though they essentially represent the same data in the context of the index. This could lead to inefficient use of disk space and performance issues, especially with large datasets.

With the introduction of deduplication in PostgreSQL 13, when multiple rows with identical indexed values are inserted, PostgreSQL now eliminates the need for storing duplicate entries in the index. Instead, it only stores the value once and maintains a list of the associated tuple (row) locations. This can significantly improve the efficiency of indexing, reduce index size, and improve performance for certain types of queries that involve duplicate indexed values.

### Benefits of B-tree Index Deduplication

1. **Reduced Disk Space Usage**: Deduplication reduces the number of entries stored in the index, thereby saving disk space.
2. **Improved Performance**: Since the index becomes smaller and more compact, searches, insertions, and deletions involving the indexed column can become faster.
3. **Faster Index Maintenance**: When indexes are smaller, PostgreSQL can rebuild and maintain them more efficiently.

### Use Cases

This feature is particularly beneficial for columns where there are many duplicate values, such as:

- Status fields (e.g., active/inactive flags)
- Gender fields (e.g., "Male", "Female")
- Categorical data (e.g., product categories)

### Technical Details

- The deduplication happens automatically when creating or maintaining B-tree indexes.
- The database checks for duplicate values when inserting into the index and keeps only one entry for each distinct value.
- It also stores pointers to the corresponding tuples in the index, so the database can still retrieve the correct rows when queried.

### Scenario: Table of Sales Records

Let's consider a table of **sales records**, where we want to create an index on the `product_id` column. Many sales records might involve the same product, so there will be lots of duplicate `product_id` values.

### Table: `sales`

```sql
CREATE TABLE sales (
    sale_id SERIAL PRIMARY KEY,
    product_id INT,
    quantity INT,
    sale_date DATE
);

```

### Step 1: Insert Some Data

Let's say we insert a few rows into the `sales` table:

```sql
INSERT INTO sales (product_id, quantity, sale_date) VALUES
    (1, 100, '2024-01-01'),
    (2, 50, '2024-01-02'),
    (1, 200, '2024-01-03'),
    (1, 150, '2024-01-04'),
    (3, 300, '2024-01-05');

```

Now, the `sales` table looks like this:

| sale_id | product_id | quantity | sale_date |
| --- | --- | --- | --- |
| 1 | 1 | 100 | 2024-01-01 |
| 2 | 2 | 50 | 2024-01-02 |
| 3 | 1 | 200 | 2024-01-03 |
| 4 | 1 | 150 | 2024-01-04 |
| 5 | 3 | 300 | 2024-01-05 |

We have a `product_id` of **1** appearing multiple times (for product 1).

### Step 2: Create an Index on `product_id`

Now, you decide to create an index on the `product_id` column to speed up queries that look for specific products:

```sql
CREATE INDEX idx_product_id ON sales(product_id);

```

### Step 3: How the Index Looks Before PostgreSQL 13

Before PostgreSQL 13, a B-tree index for `product_id` would look something like this (with no deduplication):

| product_id | tuple IDs (row pointers) |
| --- | --- |
| 1 | (1, 3, 4) |
| 2 | (2) |
| 3 | (5) |

Here:

- For `product_id = 1`, we store three pointers (to rows 1, 3, and 4) because the same product appears in multiple rows.
- For `product_id = 2`, there is one pointer (to row 2).
- For `product_id = 3`, there is one pointer (to row 5).

### Step 4: Deduplication in PostgreSQL 13

In **PostgreSQL 13** (and later), the **deduplication** feature in B-tree indexes changes this process.

Instead of storing multiple copies of `product_id = 1` in the index, **PostgreSQL 13 will store only one entry for each distinct value** in the indexed column, with a list of **tuple pointers** (row IDs) for each occurrence.

So, after deduplication, the index looks like this:

| product_id | tuple IDs (row pointers) |
| --- | --- |
| 1 | (1, 3, 4) |
| 2 | (2) |
| 3 | (5) |

### Explanation of Deduplication:

- **Before deduplication (Pre-PostgreSQL 13)**:
    - The index stored **multiple entries** for the same `product_id` if it appeared multiple times. For `product_id = 1`, it stored 3 entries, each pointing to a different row.
- **After deduplication (PostgreSQL 13)**:
    - The index stores **only one entry** for `product_id = 1`. Instead of multiple entries for each instance of `product_id = 1`, it only stores one `product_id = 1` with a list of **tuple IDs** (row pointers) that refer to the actual rows where that product appears (rows 1, 3, and 4).

### Step 5: Benefits of Deduplication

1. **Reduced Index Size**:
    - In the deduplicated index, `product_id = 1` only needs one entry (with a list of row pointers), whereas in the pre-PostgreSQL 13 index, `product_id = 1` would have had three separate entries. This reduces the overall size of the index, which is especially important when dealing with large tables with many duplicates.
2. **Improved Performance**:
    - Query performance for searches on the indexed column (`product_id`) will be faster because the index is smaller and more efficient. For example, a search for `product_id = 1` can quickly retrieve the row pointers without needing to traverse multiple entries.
3. **More Efficient Insertions/Updates**:
    - When new rows are inserted or existing rows are updated, PostgreSQL doesn't need to insert multiple copies of the same `product_id` in the index. This saves time and resources, improving overall database performance.

### Example Query:

Let's say you run a query to find all sales for `product_id = 1`:

```sql
SELECT * FROM sales WHERE product_id = 1;

```

With the deduplicated index:

- PostgreSQL will find the index entry for `product_id = 1`, and since it points to rows 1, 3, and 4, it will directly fetch those rows.

### Conclusion:

Deduplication in PostgreSQL 13 B-tree indexes significantly reduces index size and improves query performance by ensuring that each distinct value in the indexed column is stored only once in the index, even if it appears multiple times in the table. This is particularly useful in cases where there are many duplicate values in the indexed column.

## **Lock with serializable isolation**

You mentioned that serializable isolation level transactions are run one after another. So what's a point of lock with pessimistically implementation with SELECT FOR UPDATE?

**Answer:**
Good question, the serializable isolation level will give you the result of having each transaction run one after the other without actually having the transactions being blocked. The transactions can still run concurrently in serializable isolation level and when the DBMS detects that a change will be out of order it issues an error (serialization failure). This is using optimistic concurrency control.

With SELECT FOR UPDATE you achieve the serialization by actually blocking the transactions from running concurrently so you will get the same result but its at the cost of concurrency. Plus you won’t fail on other word pessimistic concurrency control

Summary serialization isolation level uses optimistic concurrency control which can be faster than pessimistic. But can fail and transactions will need to be retried

Suppose i have products table, i have two outlets of my shop accessing the same database (serializable isolation level)

Consider two scenarios,

scenario 1:

(both transactions start simultaneously)

T1:

update QTY of product A

T2:

update QTY of product B

So here i assume that since both the transactions are accessing different rows, they will be ran concurrently

scenario 2:

(both transactions start simultaneously)

T1:

update QTY of product A

T2:

update QTY of product A

So here i assume that since both the transactions are accessing same row it will be ran concurrently and then DBMS will throw serialization failure error and hence it will fail both the transaction (or will it retry automatically but in a serializable way?)

because both accessing the same the row the first transaction to update the row holds a FOR UPDATE lock, even in serializable mode.

so the second transaction will just wait and be blocked. once the first transaction commits the second transaction attempts to update the same row and it will see the the value has changed from its snapshot and it will fail with a serilization error.

remember that serializable doesn’t force the trans to run one after the other physically. they can still run concurrently as long as this rule is satisfied

if tx1 read x then tx2 changed it, tx1 will fail to commit

if tx1 writes something that tx2 then reads that also fails because tx2 is now reading something that has changed. and as a result can’t guarantee the order

## HOT updates in Postgres (by design some space in the page is left empty to accommodate future update in same page/location)

In PostgreSQL, **HOT (Heap-Only Tuple) updates** refer to an optimization technique that aims to reduce the overhead of index updates when modifying a row in a table. The primary goal of HOT updates is to improve **write performance** by avoiding unnecessary index maintenance during certain kinds of row updates.

### What is a HOT Update?

A **HOT update** occurs when a row in a table is updated in such a way that it doesn't require any changes to the **indexes** that reference the row. This is possible when the update:

- **Does not affect indexed columns** (i.e., columns that are part of any index).
- The **row remains physically in the same position** in the table.

In this case, PostgreSQL can simply update the tuple in the heap (the main storage of table data) without having to modify the associated index entries. This reduces the need for additional writes to the index, which is often the more expensive part of an update operation.

### How Does HOT Update Work Internally?

Internally, PostgreSQL stores table data as **tuples** in a **heap** file. When an update is performed on a row, PostgreSQL typically writes a new version of the row to the heap and marks the old version as "dead" (in a special state called **MVCC**—Multi-Version Concurrency Control). The **dead tuples** are later cleaned up by a **VACUUM** process.

In the case of a HOT update, the new tuple is written in place in the same physical location in the heap, **without modifying any indexes**. This allows the update to be more efficient than a traditional update, which would require index entries to be updated, potentially leading to more disk I/O.

Here’s how HOT update works step by step:

1. **Update Request**: An update is issued to a row, but the update does not modify any indexed columns (or does not change the values in such a way that it would require an index update).
2. **Heap Update**: PostgreSQL checks whether the update can be classified as a HOT update. If the update can be performed without needing to update the index, the row is updated directly in the table’s heap.
3. **Tuple Versioning**: The old version of the tuple is marked as obsolete but isn't physically deleted until the next vacuum cycle (due to MVCC). The new version is written in the same location.
4. **No Index Update**: Since indexed columns are not modified, no changes are made to the indexes, reducing the overhead of maintaining the index structures.
5. **Tuple Visibility**: When querying the table, PostgreSQL ensures that the most recent tuple version (i.e., the new version after the HOT update) is visible to transactions, using its MVCC mechanism.

### Example of HOT Update

Let’s take an example with a simple table:

```sql
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name TEXT,
    salary INT
);

```

And let's say we have an index on the `salary` column:

```sql
CREATE INDEX idx_salary ON employees(salary);

```

Now, suppose we have a row with `id=1`, `name='Alice'`, and `salary=50000`:

```sql
INSERT INTO employees (name, salary) VALUES ('Alice', 50000);

```

Now, let’s update this row to change the `name`:

```sql
UPDATE employees SET name = 'Alicia' WHERE id = 1;

```

In this case, the update only modifies the `name` column, which is **not indexed**, so PostgreSQL will perform a **HOT update**:

1. **Heap update**: The new `name` ('Alicia') is written in the same heap tuple location.
2. **No index update**: The `salary` column, which is part of the index, is not modified, so the index on `salary` is **not updated**.
3. The old tuple (with `name='Alice'`) is marked as **dead** but not removed immediately.
4. The index on `salary` remains unchanged.

### When HOT Updates Are Not Possible

HOT updates are not possible when:

1. **Indexed columns are modified**: If any column that is part of an index is updated, the index will need to be updated, and the update cannot be a HOT update. In such cases, PostgreSQL has to update the index, which incurs more overhead.
    
    For example:
    
    ```sql
    UPDATE employees SET salary = 55000 WHERE id = 1;
    
    ```
    
    Here, the `salary` column is indexed, so the index on `salary` must be updated, and a HOT update cannot be used.
    
2. **Row is moved**: If the update changes the physical location of the row (e.g., due to the row becoming too large), a new version of the tuple will be written in a different location in the heap. This will require index updates to reflect the new location of the tuple.
3. **The table is not well-suited for HOT updates**: If a table is heavily indexed, HOT updates will be less common because even minor changes may require updating the indexes.

### Performance Impact of HOT Updates

HOT updates can significantly improve **write performance** because they reduce the need for index maintenance during updates. This means that:

- **Fewer I/O operations**: There is less disk I/O because the index does not need to be updated.
- **Faster update times**: Since the index does not need to be updated, the overall time to perform an update is faster.
- **Reduced contention**: Since indexes are not modified, there is less contention between different transactions trying to modify the same index.

However, there are some trade-offs:

- **Vacuum overhead**: Dead tuples accumulate faster because updates are done in place without cleaning up old rows immediately. This means the vacuum process has to work harder to clean up old tuples that are no longer visible.
- **Hotspotting**: If rows are frequently updated, the same tuple may be rewritten in place multiple times, leading to potential **tuple chaining** or **index bloat** (especially if there are more complex updates later on).

### Example of Performance Impact

Consider the following two scenarios:

1. **Without HOT Update (Traditional Update)**:
    - You have an index on `salary`.
    - Every time the `salary` column is updated, PostgreSQL needs to:
        - Write the new version of the row.
        - Update the index on `salary` to point to the new location of the row.
2. **With HOT Update**:
    - You update the `name` column (which is not indexed).
    - PostgreSQL can update the row in place without modifying the `salary` index, reducing I/O.

If you update `name` frequently, PostgreSQL will only have to update the heap and not the index, which results in **significant performance improvements** in cases of high update traffic.

### Conclusion

HOT updates are a performance optimization in PostgreSQL that allows for faster row updates by avoiding unnecessary index modifications. When certain conditions are met (such as not modifying indexed columns), PostgreSQL will update the row directly in the heap, reducing the overhead of maintaining indexes. This results in better performance, particularly in write-heavy applications, but comes with trade-offs like increased need for vacuuming to clean up dead tuples.

## **How to choose the order of columns to create a composite index?**

1. If I have a query that does a filter on say 10 columns (joined by AND), is it advisable to have a composite index on all 10 columns? Let's assume for simplicity that this is the only query for this table

2. Is the Query Planner smart enough to arrange the filter in an order that will be a subset of the composite index? For example if I have a composite index on a,b, c and d columns. I hit a query saying `select a,b,c,d where c=10 and a=20 and b=30`; Will the Query Planner use the composite index?

3. Is there good practice for the order in which the column order for composite indexes be chosen?

**Answer:**
1) you see think about how a composite index works, It includes all values of indexed columns in the b-tree structure. This increases the size of the index which leads to more IOs to read. This of course depends on the data types of what you are indexing. Another side effect is updates to any of those columns Would require updating the index, this is not necessary slow but just increases IO.

I would be pragmatic and find out the frequency of values in each column and only index the column that would give me maximum benefit, this requires you understanding the data model, the nature of the data stored and the correlation between them.

2) yes, the planner will do what’s necessary the order of the where clause doesn’t matter as long as everything is an AND.

3) depends on your where clause and the minimum set of the query.

## **Redis "Durability" vs "Persistence"**

You said that Redis doesn't offer Durability because of course it is memory database "lives in RAM" if electricity goes down, data is lost. And also yo said but it has Persistence. I need to know what do you mean by that and i want to understand from your point of view. What is Durability vs Persistence?

Thanks in advance and thank you so much for this amazing lecture, i loved it so much and i installed Postgres on my RaspberryPi and went with all examples and applied them all and took 160 lines of notes and steps :D

**Answer:**
Persistence is the ability and feature that a database provide to persist and store data on disk.

Durability is when you successfully write a value to the DB it should always persist to disk so it is available when the DB crashes/restarts.

Redis is an in memory database that supports persistence but they do offer true durability.

Redis persist data in an asynchronous snapshot every x seconds. So you can write a value in memory but if the power goes off before it gets persisted to snapshot, you lost it..

I believe this might have changed in the recent versions of redis. Subtle difference but important to point out.

Redis uses AOF (Append Only File) and hence supports high durability at high throughput. It also allows controlling the knob (Snapshot vs AOF)


## Reference to related articles
- [**Postgres vs MySQL (The fundamental differences)**](postgres-vs-msysql)
- [**PostgreSQL Process Architecture**](postgresql-process-architecture)
- [**WAL, Redo and undo logs in postgres**](wal-redo-and-undo-logs-in-postgres)
- [**How Shopify’s engineering improved database writes by 50% with ULID**](how-shopify-engineering-improved-database-writes)
- [**Postgres Locks — A Deep Dive**](postgres-locks-a-deep-dive)
- [**How Slow is select * in row store?**](how-slow-is-select-in-row-store)
- [**Why Uber Engineering Switched from Postgres to MySQL**](why-uber-engineering-switched-from-postgres-to-mysql)
- [**NULL.pdf**]({{page.attachment_path}}/NULL.pdf)
- [**Write Amplification Problem in PostgreSQL**](write-amplification-problem-in-postgresql)
- [**TOAST table in Postgres**](toast-table-in-postgres)
- [**InnoDB B-tree Latch Optimization History**](innodb-b-tree-latch-optimization-history)
