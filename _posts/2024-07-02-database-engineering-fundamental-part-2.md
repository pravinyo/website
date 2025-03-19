---
title: "Part 2: Database Engineering Fundamentals"
author: pravin_tripathi
date: 2024-07-02 01:00:00 +0530
readtime: true
img_path: /assets/img/database-engineering-fundamental-part-2/
attachment_path: /assets/document/attachment/database-engineering-fundamental-part-2/
categories: [Blogging, Article]
mermaid: true
permalink: /database-engineering-fundamental-part-2/
tags: [softwareengineering, backenddevelopment]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Copilot
---

## Table of Contents

### **B-Tree vs B+ Tree in Production Database Systems**
- [Full Table Scans](#full-table-scan)
- [B-Tree](#b-tree)
- [B-Tree limitations](#limitation-of-b-tree)
- [B+Tree](#b-tree-1)
- [B+Tree Considerations](#b-tree-dbms-considerations)
- [B+Tree storage cost in MySQL vs Postgres](#storage-cost-in-postgres-vs-mysql)
- [Summary](#summary)

### **Database Partitioning**
- [What is Partitioning?](#database-partitioning)
- [Horizontal Partitioning vs Vertical Partitioning](#vertical-vs-horizontal-partitioning)
- [Partitioning Types](#partitioning-types)
- [Partitioning vs Sharding](#horizontal-partitioning-vs-sharding)
- [Demo](#implementing-partitioning-in-postgresql)
- [Pros & Cons](#pros-of-partitioning)

### **Database Sharding**
- [What is sharding?](#what-is-sharding)
- [Consistent Hashing](#consistent-hashing)
- [Horizontal Partitioning vs Sharding](#horizontal-partitioning-vs-sharding-1)
- [Pros & Cons](#pros-of-sharding)
- [When Should You Consider Sharding Your Database?](#when-should-you-consider-sharding-your-database)

---

## B-Tree vs B+ Tree in Production Database Systems

_And their impact on production database systems_

### Topics Overview
- Full Table Scans
- B-Tree
- B-Tree limitations
- B+Tree
- B+Tree Considerations
- B+Tree storage cost in MySQL vs Postgres
- Summary

### Full Table Scan

- To find a row in a large table we perform full table scan
- Reading the entire table is slow (many IO to fetch pages)
- We need a way to reduce the search space

### B-Tree

- Balanced data structure for fast traversal
- B-Tree has nodes
- In B-Tree of "m" degree each node can have (m) child nodes
- Node has up to (m-1) elements
- Each element has key and value
- The value is usually data pointer to the row
- Data pointer can point to primary key or tuple
- Root Node, internal node and leaf nodes
- A node = disk page

![B-Tree Structure](image.png)

### Limitation of B-Tree

- Elements in all nodes store both key and value
- Internal nodes take more space thus require more IO and can slow down traversal
- Range queries are slow because of random access (give me all values 1-5)
- B+Tree solves both these problems
- Hard to fit internal nodes in memory

### B+Tree

- Exactly like B-Tree but only stores keys in internal nodes
- Values are only stored in leaf nodes
- Internal nodes are smaller since they only store keys and they can fit more elements
- Leaf nodes are "linked" so once you find a key you can find all values before and after that key
- Great for range queries

### B+Tree & DBMS Considerations

- Cost of leaf pointer (cheap)
- 1 Node fits a DBMS page (most DBMS)
- Can fit internal nodes easily in memory for fast traversal
- Leaf nodes can live in data files in the heap
- Most DBMS systems use B+Tree
- MongoDB (WiredTiger) uses B-Tree

### Storage Cost in Postgres vs MySQL

- B+Trees secondary index values can either point directly to the tuple (Postgres) or to the primary key (MySQL)
- If the Primary key data type is expensive this can cause bloat in all secondary indexes for databases such as MySQL (InnoDB)
- Leaf nodes in MySQL (InnoDB) contains the full row since it's an IOT / clustered index

Reference: [b-tree-original-paper.pdf]({{page.attachment_path}}/b-tree-original-paper.pdf)

---

## Database Partitioning

### Topics Overview
- What is Partitioning?
- Horizontal Partitioning vs Vertical Partitioning
- Partitioning Types
- Partitioning vs Sharding
- Demo
- Pros & Cons

![Partitioning Diagram](image%201.png)

Using partitioning, we can split large tables into smaller manageable tables that make lookups quite efficient.

### Vertical vs Horizontal Partitioning

- **Horizontal Partitioning** splits rows into partitions
  - Range or list
- **Vertical partitioning** splits columns into partitions
  - Large column (blob) that you can store in a slow access drive in its own tablespace

### Partitioning Types

- **By Range**
  - Dates, ids (e.g., by logdate or customerid from-to)
- **By List**
  - Discrete values (e.g., states CA, AL, etc.) or zip codes
- **By Hash**
  - Hash functions (consistent hashing)

### Horizontal Partitioning vs Sharding

- HP splits big table into multiple tables in the same database, client is agnostic
- Sharding splits big table into multiple tables across multiple database servers
- HP table name changes (or schema)
- Sharding everything is the same but server changes

### Implementing Partitioning in PostgreSQL

To create a partition, follow these steps:

```sql
-- Create the parent table with partitioning scheme
CREATE TABLE grades_parts(id serial NOT NULL, g int NOT NULL) PARTITION BY RANGE(g);

-- Create individual partition tables
CREATE TABLE grade0035 (LIKE grades_parts INCLUDING INDEXES);
CREATE TABLE grade3560 (LIKE grades_parts INCLUDING INDEXES);
CREATE TABLE grade6080 (LIKE grades_parts INCLUDING INDEXES);
CREATE TABLE grade80100 (LIKE grades_parts INCLUDING INDEXES);
```

Once we have created the main table and partition tables, we attach the partition tables to the main table:

```sql
-- Attach partitions with their respective ranges
ALTER TABLE grades_parts ATTACH PARTITION grade0035 FOR VALUES FROM (0) TO (35);
ALTER TABLE grades_parts ATTACH PARTITION grade3560 FOR VALUES FROM (35) TO (60);
ALTER TABLE grades_parts ATTACH PARTITION grade6080 FOR VALUES FROM (60) TO (80);
ALTER TABLE grades_parts ATTACH PARTITION grade80100 FOR VALUES FROM (80) TO (100);
```

Now our partition is attached and ready to populate data.

When we insert data into the main table (grades_parts), the database will decide which partition that record should go to based on the g values.

If you want to have an index on each partition, which is a common use case, you can do so by creating an index on the parent table:

```sql
CREATE INDEX grades_parts_idx ON grades_parts(g);
```

This is powerful and simplified - separate indexes will be created by the database for each partition table.

Some queries may check other partitions even though data isn't present there:

```sql
SELECT COUNT(*) FROM grades_parts WHERE g = 30;
```

Even though records should only be in the first partition (`grade0035`), the database might check other partitions too.

To optimize this behavior, you can enable partition pruning:

```sql
SHOW enable_partition_pruning;
SET enable_partition_pruning = on;
```

This will make lookup happen in the first partition only and avoid looking at other partitions. It's enabled by default in PostgreSQL.

### Pros of Partitioning

- Improves query performance when accessing a single partition
- Sequential scan vs scattered index scan
- Easy bulk loading (attach partition)
- Archive old data that are barely accessed into cheap storage

### Cons of Partitioning

- Updates that move rows from a partition to another (slow or fail sometimes)
- Inefficient queries could accidentally scan all partitions resulting in slower performance
- Schema changes can be challenging (DBMS could manage it though)

---

## Database Sharding

### Topics Overview
- What is sharding?
- Consistent Hashing
- Horizontal Partitioning vs Sharding
- Pros & Cons

### What is Sharding?

Sharding is a database architecture pattern related to horizontal partitioning, which is the practice of separating one table's rows into multiple different tables, known as partitions or shards. Each partition has the same schema and columns, but entirely different rows.

In sharding, these smaller partitions are distributed across separate database nodes, often on different physical servers. This approach allows for better scalability and performance in large-scale applications.

- Sharding distributes data across multiple machines
- Each shard is an independent database, and collectively, the shards make up a single logical database
- Sharding is typically used to improve the performance and scalability of very large databases
- It allows for horizontal scaling, which can be more cost-effective than vertical scaling (upgrading to more powerful hardware)

Sharding is particularly useful for applications that deal with big data or high traffic, as it helps distribute the load and improve query response times.

![Sharding Diagram 1](image%202.png)

![Sharding Diagram 2](image%203.png)

### Consistent Hashing

Consistent hashing is a technique used in database sharding to distribute data across multiple nodes efficiently. Here's how it works:

- The hash space is represented as a fixed circular ring (usually 0 to 2^32 - 1)
- Both data items and nodes (shards) are mapped to this ring using a hash function
- To find which shard a data item belongs to, move clockwise on the ring from the item's hash position until a node is found
- When adding or removing nodes, only a fraction of the data needs to be redistributed

This approach provides better distribution and minimizes data movement when the number of shards changes, making it ideal for dynamic environments.

Benefits of consistent hashing in database sharding:

- **Scalability**: Easily add or remove shards without major data redistribution
- **Load balancing**: Evenly distributes data across shards
- **Fault tolerance**: If a shard fails, its data is redistributed among remaining shards
- **Reduced hotspots**: Helps prevent overloading of specific shards

Many distributed databases and caching systems, like Cassandra and Redis, use consistent hashing for efficient data distribution across shards.

![Consistent Hashing Diagram](image%204.png)

### Horizontal Partitioning vs Sharding

- HP splits big table into multiple tables in the same database
- Sharding splits big table into multiple tables across multiple database servers
- HP table name changes (or schema)
- Sharding everything is the same but server changes

### Sharding with Postgres

[Sharding with Postgres](sharding-with-postgres)

### Pros of Sharding

- **Scalability**
  - Data
  - Memory
- **Security** (users can access certain shards)
- **Optimal and smaller index size**

### Cons of Sharding

- Complex client (aware of the shard)
- Transactions across shards problem
- Rollbacks
- Schema changes are hard
- Joins
- Has to be something you know in the query

### When Should You Consider Sharding Your Database?

**It should be the last option in your scaling strategy.**

You should consider sharding only if these options aren't working for you:

1. Apply indexing to tables
2. If indexing is not working, split tables into multiple partitions, each partition maintains small tables and has their own indexes
3. Have master-slave setup
4. Replicate the setup to region-specific instances like Asia, America, or EU users

Sharding is one of the most complicated database scaling strategies.

In sharding, ACID transactions are difficult as they are distributed across different servers, becoming a distributed transaction use case.

You typically sacrifice full rollback and commit capabilities.

The application must be aware of the shards.

There are additional server maintenance costs.