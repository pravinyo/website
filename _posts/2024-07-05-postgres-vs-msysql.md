---
title: "Part 5: Database Engineering Fundamentals: Postgres vs MySQL"
author: pravin_tripathi
date: 2024-07-05 06:00:00 +0530
readtime: true
media_subpath: /assets/img/database-engineering-fundamental-part-5/
categories: [Blogging, Article]
mermaid: true
permalink: /database-engineering-fundamental-part-5/postgres-vs-msysql/
parent: /database-engineering-fundamental-part-5/
tags: [softwareengineering, backenddevelopment,database-engineering]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Copilot
---
# Postgres vs MySQL (The fundamental differences)

### Postgres vs MySQL

### The main differences with examples

One of you guys asked a question on the Q&A section about the difference between Postgres and MySQL. The answer turned out too long I thought I’ll make it into a post.

In a nutshell, the main difference between the two databases really boils down to the implementation of primary and secondary indexes and how data is stored and updated.

Let us explore this further.

### But First.. Fundamentals

An index is a data structure (B+Tree mostly) that allows searching for keys through layers of nodes which databases implement as pages. The tree traversal allows eliminating pages that don’t have the result and norrowing down pages that has it. This continues until a leaf page where key live is found.

Leaf nodes or pages contain a list of ordered keys and their values. When a key is found you get its value and the page is cached in the database shared buffers with the hope that future queries may request keys in the same page.

> This last sentence is the fundamental understanding of all what database engineering, administration, programming and modeling is about. Knowing that your queries are hitting keys next to each other in a page will minimize I/Os and increase performance.
> 

Keys in B+Tree indexes are the column(s) on the table the index is created on, and the value is what databases implement differently. Let us explore what value is in Postgres vs MySQL.

### MySQL

In a primary index, the value is the full row object with all the attributes*. This is why primary indexes are often referred to as clustered indexes or another term that I prefer index-organized table. This means the primary index is the table.

> *Note that this is true for row-store, databases might use different storage model such as column-store, graphs or documents, fundamentally those could be potential values.
> 

If you do a lookup for a key in the primary index you find the page where the key lives and its value which is the full row of that key, no more I/Os are necessary to get additional columns.

In a secondary index the key is whatever column(s) you indexed and the value is a pointer to where the full row really live. The value of secondary index leaf pages are usually primary keys.

This is the case in MySQL. In MySQL all tables [must](https://dev.mysql.com/doc/refman/5.7/en/innodb-index-types.html) have a primary index and all additional secondary index point to the primary keys. If you don’t create a primary key in a MySQL table, one is created for you.

Example of where MySQL innoDB all tables must have a clustered primary index

### Postgres

In Postgres technically there is no primary index, all indexes are secondary and all point to system managed tuple ids in data pages loaded in the heap. The table data in the heap are unordered, unlike primary index leaf pages which are ordered. So if you insert rows 1–100 and all of them are in the same page, then later updated rows 1–20, those 20 rows may jump into a different page and become out of order. While in a clustered primary index, insert must go to page that satisfies they key’s order. That is why Postgres tables are often referred to as “heap organized tables” instead of “index organized tables”.

It is important to note that updates and deletes in Postgres are actually inserts. Every update or delete creates a new tuple id and the old tuple id is kept for MVCC reasons. I’ll explore that later in the post.

> The truth is the tid by it itself is not enough. Really we need both the tuple id and also the page number, this is referred to as c_tid. Think about it, it is not enough to just know the tuple id we need to know which page the tuple live. Something we didn’t have to do for MySQL because we are actually doing a lookup to find the page of the primary key. Where as in Postgres we are simply doing an I/O to fetch the full row.

> Example of where Postgres tables are heap organized and all indexes point to the tuple ids

### Queries Cost

Take the following table for the examples below.

- TABLE T;  
- PRIMARY INDEX ON PK AND SECONDARY INDEX ON C2, NO INDEX ON C1  
- C1 and C2 are text
- PK is integer


| PK | C1 | C2 |
|----|----|----|
| 1  | x1 | x2 |
| 2  | y1 | y2 |
| 3  | z1 | z1 |

Let us compare what happens in MySQL vs Postgres

`SELECT * FROM T WHERE C2 = 'x2';`

That query in MySQL will cost us two B+Tree lookups*. We need first to lookup *x2* using the secondary index to find x2's primary key which is 1, then do another lookup for 1 on the primary index to find the full row so we return all the attributes (hence the *).

> *One might think this is simply two I/Os which isn’t true, a B+Tree lookup is a O(logN) and depending on the size of the tree it might result in many I/Os. While most of the I/Os might be logical (hitting cached pages in shared buffers) it is important to understand the difference.  


In Postgres looking up any secondary index will only require one index lookup followed by a constant single I/O to the heap to fetch the page where the full row live. One B+Tree lookup is better than two lookups of course.

To make this example even more interesting, say if C2 is not-unique and there were multiple entries of x2, then we will find tons of tids (or PKs in MySQL) matching the x2. The problem is those row ids will be in different pages causing random reads. In MySQL it will cause Index lookups ( perhaps the planner may opt for an [index scan](https://medium.com/@hnasr/index-seek-vs-index-scan-in-database-systems-641c2ac811fc) vs a seek based on the volume of these keys) but both databases will result in many random I/Os.

Postgres attempts to minimize random reads by using bitmap index scans, grouping the results into pages instead of tuples and fetching the pages from the heap in fewer I/Os possible. Later additional filtering is applied to present the candidate rows.

Let us take a different query.

`SELECT * FROM T WHERE PK BETWEEN 1 AND 3;`

I think MySQL is the winner here for range queries on primary key index, with a single lookup we find the first key and we walk the B+Tree linked leaf pages to find the nearby keys as we walk we find the full rows.

Postgres struggles here I think, sure the secondary index lookup will do the same B+Tree walk on leaf pages and it will find the keys however it will only collect tids and pages. Its work is not finished. Postgres still need to do random reads on the heap to fetch the full rows, and those rows might be all over the heap and not nicely tucked together, especially if the rows were updated. Update heavy workload is Postgres’s enemy, pick a good [FillFactor](https://youtu.be/qXDhMJCuDEc) for your table.

Ok let us do an update.

`UPDATE T SET C1 = ‘XX1’ WHERE PK = 1;`

In MySQL updating a column that is not indexed will result in only updating the leaf page where the row is with the new value. No other secondary indexes need to be updated because all of them point to the primary key which didn’t change.

In Postgres updating a column that is not indexed will generate a new tuple and *might** require ALL secondary indexes to be updated with the new tuple id because they only know about the old tuple id. This causes many write I/Os. [`Uber didn’t like this one in particular back in 2016`](https://youtu.be/_E43l5EbNI4), one of their main reasons to switch to MySQL off of Postgres.

> I said might here because in Postgres there is an optimization called HOT (heap only tuple) not to be confused with (Heap organized table) that keeps old tuple id in the secondary indexes but put a link on the heap page header that points old tuple to the new one.
> 

### Data types Matter

In MySQL choosing the primary key data type is critical, as that key will live in all secondary indexes. For example a UUID primary key will bloat all secondary indexes size causing more storage and read I/Os.

In Postgres the tuple id is fixed 4 bytes so the secondary indexes won’t have the UUID values but just the tids pointing the heap.

### Undo logs

All modern databases support multi version concurrency control (MVCC). In simple read committed isolation level if a transaction *tx1* updates a row and didn’t commit yet while another concurrent transaction *tx2* wants to read that row it MUST read the old row not the updated one. Most databases (MySQL included) implement this feature using undo logs.

When a transaction makes a change to a row, the change is written to the page in the shared buffer pool so the page where the row live always has the latest data. The transaction then logs information of how to undo the latest changes to row (enough info to construct the old state) in an undo log, this way concurrent transactions that still need the old state based on their isolation level must crack the undo log and construct the old row.

You might wonder if writing uncommitted changes to the page is a good idea. What happens if a background process flushes the page to disk and then the database crashed before the transaction can commit? That is where the undo log is critical. Right after a crash, the uncommitted changes are undone using undo logs on database startup*.

One can’t deny the cost of undo logs for long running transactions on other running transactions. More I/Os will be required to construct the old states and there is a chance the undo logs can get full and the transaction might fail.

> In one case I have seen one database system takes over an hour to recover from a crash after running a 3 hour uncommitted long transaction. Yeah avoid long transactions at all costs.

Postgres does this very differently, each update, insert and delete gets a new copy of the row with a new tuple id with hints about what transaction id created the tuple and what transaction id deleted the tuple. So Postgres can safely write the changes to the data pages and concurrent transactions can read the old or new tuples based on their transaction id. Clever design.

Of course no solution is without its problems. We actually talked about the cost of creating new tuple ids on secondary indexes. Plus Postgres need to purge old tuples that are no longer required if all running transactions ids are greater than the transaction that deleted the tuples. [`Vacuum`](https://medium.com/@hnasr/postgresql-process-architecture-f21e16459907) takes care of that.

### Processes vs Threads

MySQL uses threads, Postgres uses processes, there are pros and cons for both I covered that in details in its own post [`here`](https://medium.com/@hnasr/postgresql-process-architecture-f21e16459907).

Postgres Process Architecture

I like threads better than processes in database systems. Just because they are lighter weight and share their parent process virtual memory address. Processes come with the overhead of dedicated virtual memory and larger control block (PCB) compared to the smaller thread control block (TCB).

If we are eventually going to share memory and deal with mutexes and semaphores anyway why not use threads. Just my two cents.

### Summary

With that in mind you get to pick which database system is right for you. What really matters is breaking down your use cases and queries and understand what each database does and see what works and what doesn’t for you.

No wrong or right here.

[Back to Parent Page]({{ page.parent }})