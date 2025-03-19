---
title: "Part 5: Database Engineering Fundamentals: PostgreSQL Process Architecture"
author: pravin_tripathi
date: 2024-07-05 07:00:00 +0530
readtime: true
img_path: /assets/img/database-engineering-fundamental-part-5/
categories: [Blogging, Article]
mermaid: true
permalink: /database-engineering-fundamental-part-5/postgresql-process-architecture/
parent: /database-engineering-fundamental-part-5/
tags: [softwareengineering, backenddevelopment]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Copilot
---
# PostgreSQL Process Architecture

![](https://miro.medium.com/v2/resize:fit:700/1*ysek1HEB8TpoWVxSVXmOHw@2x.jpeg)

# **PostgreSQL Process Architecture**

## **Creating a listener on the backend application that accepts connections is simple. You listen on an address-port pair, connection attempts…**

![](https://miro.medium.com/v2/resize:fill:88:88/1*j-h09TiaKTgYsIvVAHPa4Q@2x.jpeg)

[**Hussein Nasser**Follow](https://medium.com/@hnasr)

Creating a listener on the backend application that accepts connections is simple. You listen on an address-port pair, connection attempts to that address and port will get added to an accept queue; The application accepts connections from the queue and start reading the data stream sent on the connection.

However, what part of your application does the accepting and what part does the reading and what part does the execution? You can architect your application in many ways based on your use cases. I have a medium post just exploring the different options, you may read the story [here](https://medium.com/@hnasr/threads-and-connections-in-backend-applications-a225eed3eddb).

In this post I explore the PostgreSQL process architecture in details. Please note that the information here is derived from both the Postgres [doc](https://www.postgresql.org/docs/current/index.html) and [code](https://github.com/postgres/postgres/tree/master/src/backend). Discussions about scalability and performance are solely based on my opinions.

### **Postmaster Process**

This is the main process that manages everything in Postgres. It creates a listener on the configured interfaces and port (default 5432). It is also responsible for forking other processes to do various tasks.

### **Backend Process**

The postmaster process creates a new "backend" process for every connection it [accepts](https://www.postgresql.org/docs/current/connect-estab.html). The connection is then handed over to the new backend process to perform the reading of the TCP stream, request parsing, SQL query parsing (yes those are different), planning, execution and returning the results. The process uses its local virtual memory for sorting and parsing logic, this memory is controlled by the *work_mem* parameter.

The more connections the postmaster accepts the more backend processes are created. The number of user connections is directly proportional to the number of processes which means more resources, memory, CPU usage and context switching. The benefits of course, each process enjoys a dedicated virtual memory space isolated from other processes, so it is great for security especially that each connection is made to a single database.

The problem with this architecture is scalability. With limited CPU cores, how can Postgres scale to thousands or tens of thousands of client connections on a single CPU? The context switching alone and the competition between all the dedicated backend processes for CPU time will cause processes to starve each other. Worth mentioning while some part of query execution will require the CPU, most of the asynchronous I/O to read and write to buffers/memory and won't involve the CPU.

Postgres knows this limitation and that is why the number of backend processes is capped by the number of connections, the [max_connections](https://www.postgresql.org/docs/15/runtime-config-connection.html#GUC-MAX-CONNECTIONS) parameter defaults to 100 which may look low but we will find out in the next few paragraphs is it actually enough for most cases. Perhaps Postgres set it this low to discourage large number of connections by default (for a good reason).

You see, backend applications such as web servers and reverse proxies are directly exposed to end-user clients resulting in potentially millions of connections. While the "clients" of Postgres as a database tend to be other web servers and backend applications that are not as verbose and can safely share a pool of connections.

> The number of clients of a Web server coming from mobile phones and browsers is much higher than those of a database which is handful of applications that can share connections.
> 

In the next section, we will learn that Postgres has a feature to offload processing to a pool of worker threads known as background workers for parallel processing.

### **Background Workers**

Most proxies, web servers (and even databases e.g. [memcached](https://medium.com/@hnasr/memcached-architecture-af3369845c09)) create a handful of processes or threads (often one for every CPU core) and distribute connections among these processes. This keeps context switching to a minimum and allow sharing of resources.

Spawning a backend process in Postgres for every connection and having that process do the work doesn't scale in my opinion. Imagine having 1000 connections, the corresponding 1000 backend processes executing client queries and competing for CPU and resources, we are left at the mercy of the operating system scheduler deciding which process gets the CPU, as a result the overall performance of the system will degrade.

### **Parallel Queries**

In [version 9.6](https://www.postgresql.org/docs/9.6/runtime-config-resource.html#GUC-MAX-PARALLEL-WORKERS-PER-GATHER), Postgres introduced the parallel queries feature which allowed multiple workers to execute a single query. This allowed for two things:

1. Break a single query and execute it on multiple cores in parallel so it completes faster.
2. Delegate the work to a fixed size pool of workers instead of connection-bound backend processes.

With parallel queries, the backend process builds a parallel plan and pulls x number of background workers from the pool to execute the plan. While a worker is executing a query it is marked as busy, no other queries can use that process. Even with large number of clients, the limited pool of background workers will serve as a configurable and predictable performance metric which didn't exist prior to Postgres 9.6.

### **Pros and Cons of Parallel Queries**

Based on the [doc](https://www.postgresql.org/docs/current/runtime-config-resource.html#GUC-MAX-PARALLEL-WORKERS-PER-GATHER), parallel queries can result in higher CPU utilization compared to non-parallel queries with the same number of clients and queries. A single query with joins and nested queries will run on a single process and single CPU core but when broken into parts it will run on multiple cores consuming more CPU. While this is true in normal/low load environment, it is slightly different in high load.

When all background workers are busy in a high load environment, new parallel queries will have to wait causing only that client to experience the wait delay while the database remain stable. Compare this to when parallel queries are disabled, nothing (as far as I know) stops the backend processes from executing all or any incoming queries, leading to an overall system performance degradation with all the processes competing for CPU and OS context switching. Of course you can limit the number of backend processes with the *max_connections* parameter to avoid this problem and Postgres does set that to a low value of 100, but then you prevent clients from connecting.

> You will notice all my focus here is on a single machine, of course having many read replicas to distribute the load is a good idea when a single machine can't handle it. But I also believe that we should not rush to distribute until we squeezed every bit of performance from a single machine.
> 

It is a trade-off, few clients will suffer "waits" when all workers are busy, but the waits can be measured, logged, understood and even in some cases tolerated. When we always let the backend process do the work everyone is competing for CPU time and we have little control.

> It is imporant to note that parallel queries will only get triggered if the backend process comes up with parallel plan. If the query is simple and the cost seems low the backend process will likely do the work. Parallel queries can be disabled by setting max_parallel_workers_per_gather to zero.
> 

### **Notes on Background workers**

Besides parallel queries, [background workers](https://www.postgresql.org/docs/15/glossary.html#GLOSSARY-BACKGROUND-WORKER) are also responsible for logical replication and custom user-code extensions. Just like backend processes, background worker processes have their own virtual memory space mainly used to store the *work_mem* area which is used for holding data for sorting. It is important to understand that *work_mem* is per process, so really multiply that for each background worker and backend process.

The background worker pool is limited by the *max_worker_processes* parameter which defaults to 8. To me, I would match it to the number of cores on the machine, but we also need to think about the nature of the workload here. Postgres is a database, while it uses the CPU for parsing, planning and sorting, the rest of the work is mostly [I/O bound](https://medium.com/@hnasr/when-nodejs-i-o-blocks-327f8a36fbd4) whether that is hitting the memory or disk. Again, each background worker will allocate a *work_mem* worth of memory in its private virtual memory space.

### **Auxiliary Processes**

Aside of query execution, Postgres does routine maintenance and management use auxiliary processes unrelated to background workers for these tasks. Below I illustrate the auxiliary processes in Postgres:

### **Background Writer (bw)**

Noted in my diagram as *bw,* the Background writer is responsible for flushing dirty pages on the shared buffers to file system. HUGE emphasis on file system and NOT necessary disk. You see, the operating system has a file system in memory cache that holds writes until it has enough for them and flushes all them at once to disk to reduce disk I/O. In case of a crash we *may* lose those changes that haven't been flushed to disk, that is why we have fsync O_DIRECT which bypasses all that stuff. Background writer simply writes the dirty pages to the file system cache to free some room in memory buffer for more pages to come in.

All the processes we talked about so far has their own private virtual memory space not accessable to each other, but they also has access to a shared memory space so that pages can be readable by multiple processes. This way all processes have access to the latest and can detect locks and other changes.

Whether it is from the backend process via non-parallel or one of the background workers through parallels, queries read pages from disk and put them in the shared buffers, and when they write they write to the pages in memory marking them dirty. The shared buffers size can be configured with the *shared_buffer* parameter and can get full, so dirty pages have to be written to the file system (and eventually to disk for durability) to free up some space for more pages to get to the shared buffers. The background writer job is to write the dirty pages to file system just to free up space in the shared buffers. There is another process that make sure dirty pages get flushed to disk and that is our next auxiliary process.

> You might say isn't it bad to write changes to memory, what if the database crashed? wouldn't we lose the changes? Actually no, we also write the changes to WAL and persist that to disk more frequently especially on commit. So even if we crashed we can pull whatever we have on disk and "redo" the changes from WAL to the data pages to get to the final state. Of course we might have also flushed pages with changes from transactions that have since rolled back, in that case Postgres "should" "undo" those changes (but doesn't). The magic or redo and undo logs. To be frank if the data pages have uncommitted changes from transcations that rolledback it is fine, future queries know to ignore those tuples. This makes postgres start up even faster as UNDO technically is not implemented yet as of writing this post. It doesn't change the fact that this bloat pages and eventually slows down performance, future vacuums should clean those
> 

### **Checkpointer (cp)**

The background writer writes the dirty pages from shared buffers to the file system cache to free up shared buffers. While changes to the file system eventually goes to disk, they do stay in the operating system file cache for a while in the hopes pages (OS pages that is) might receive more writes and then OS can flush all them in one I/O to disk. There is a possibility that we might lose pages in the file system cache in case of a crash so databases never relay on file system cache for durabilty.

There is another auxiliary process called checkpointer, which bypasses the file system cache and enforces that the pages are written to disk. The checkpointer (cp) also creates a checkpoint record that guarantees that at this point the WAL and data files pages are 100% in sync and if we crash we will use that checkpoint as our starting point to redo the changes from the WAL, which also has been flushed to disk.

### **Startup Process (st)**

While discussing the background writer and checkpointer we mentioned in case of crash, Postgres applies the WAL changes to data pages to come back to a consistent state. Well , it is the startup process auxiliary process that redo the changes. I suppose that nothing can be done until the startup process completes. This makes me think that the startup process might run even before the postmaster, I would sure implement it this way so that no one can connect unless I can recover my database.

### **Logger (lg)**

Someone needs to write database events, warnings, errors, and (if you enabled tracing) logging the SQL statements, this is the job of the auxiliary process Logger also called syslogger.

### **Autovacuum Launcher (avl)**

Another auxiliary process that wakes up and launches autovacuum workers which is a completely different pool to do the vacuum process. Not much info on this process so just adding it for completion. I suppose when autovacuum is disabled the launcher is not spawn nor its workers.

### **WAL writer (ww)**

The WAL (Write-ahead log) lives as WAL records in the shared memory, many processes write to the WAL as transcations takes place. Eventually the WAL has to go into the WAL files on disk not just file system cache, but actually physically on disk for them to be useful. The auxiliary process WAL writer (ww) is responsible to flush the WAL.

### **WAL archiver (wa)**

Once a checkpoint is created by the checkpointer process, older WAL records can be safely purged. However, for backup, recovery and replication purposes, WAL entries can be archived, the WAL archiver auxiliary process takes car of this.

### **WAL receiver (wr)**

It is enough to stream WAL records from primary to standby databases to achieve replications. Data files can be updates accordingly as they are much larger. The auxiliary process WAL receiver runs on the replica to receive these changes and apply them to the pages in memory. It is worth mentioning that any process that understands the replication protocol can also receive WAL records.

### **Other Processes**

I couldn't find a category where I put these so I created one. Those are processes that are not backend nor auxiliary (don't ask me why). Let us explore them.

### **Autovaccume workers**

Vacuum is the process that cleans up entries in pages that are no longer required, whether dead tuples or those left over from rollbacked transactions. Vacuum also cleans entries in the pages from transcations that didn't get to commit because of a crash but their changes have made it to the disk by background or checkpointer, a process refered to by undo which the startup process should do but not yet implemented in postgres.

The Autovacuum workers which are spawned by the autovacuum launcher take care of the vacuuming. The autovacuum workers also perform analyze to update the statistics on all tables and indexes.

### **WAL senders**

According to the postgres doc, these are referred to as special backend process that streams WAL changes to the WAL receiver auxiliary process which we discussed before. You can configure how many WAL senders postgres spins up with max_wal_senders parameter.

### **Process vs Thread**

To the million dollar question. Why processes and not threads? To be honest I couldn't find a convincing answer. I wanted to explore the differences between processes and threads here but that will make this post even longer. I rather do that in a new post. Will link it here once authored. Actually do me a favor, highlight this sentence to see how many of you actually reached this section and found this post interesting.

Until then I'll summarize what I know, processes are definitely heaver than threads, each process have its own virtual memory, it maintains metadata called PCB (process control block) which includes page table for mapping virtual to physical addresses and any other metadata about the process. The PCB has to be stored in memory and brought into the CPU cache registers do translate virtual memory addresses to physical addresses. Threads on the other hand share the virtual memory space with their parent process and their TCB (thread control block) is much smaller with a pointer to parent process PCB. So your cache hits are much higher with threads than processes.

The only reason I can find as to why Postgres use processes instead of threads are because threads used to be unstable. This discussion is dated on [2004](https://www.postgresql.org/message-id/1098894087.31930.62.camel@localhost.localdomain) but since then threading subsystem in operating system are much more stable of course. The question remain, is it really worth it for postgres to switch to threads instead of processes? To me I don't think, that will far destabilize postgres and it will take years to implement and even then how much is the benefit really is?

If I would change something in Postgres its not really the Processes, but the concept of one backend process per connection. We can create a pool of backend processes per database, effectively move the connection pooling from the application down to the database.

Edit: 6/9/2023 A [Postgres community thread](https://www.postgresql.org/message-id/31cc6df9-53fe-3cd9-af5b-ac0d801163f4%40iki.fi) has started to discuss the possiblity of making Postgres multi-threaded.

### **Summary**

Postgres uses processes for all its operations. In this post I illustrated the process architecture of Postgres and explored all the processes (that I'm aware of). Processes have mainly two categories of process groups, one called backend processes which is directly client facing and get one per connection and do the actual work (unless in parallel), the others are system auxiliary processes which do maintaince and routine tasks. Postgres also have other types of special processes such as autovacuum workers.

If you enjoyed this post consider checking out my [database](https://database.husseinnasser.com/) and [backend engineering](https://backend.husseinnasser.com/) courses.

If you prefer to watch a video of this article

[#postgres](https://medium.com/tag/postgres)[#database](https://medium.com/tag/database)[#postgresql](https://medium.com/tag/postgresql)[#software-architecture](https://medium.com/tag/software-architecture)[#process-vs-thread](https://medium.com/tag/process-vs-thread)

[Back to Parent Page]({{ page.parent }})