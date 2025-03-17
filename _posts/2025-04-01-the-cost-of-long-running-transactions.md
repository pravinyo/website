---
title: "Part 1: Database Engineering Fundamentals: The Cost of Long running Transactions"
author: pravin_tripathi
date: 2025-04-01 04:00:00 +0530
readtime: true
img_path: /assets/img/database-engineering-fundamental-part-1/
categories: [Blogging, Article]
mermaid: true
permalink: /database-engineering-fundamental-part-1/the-cost-of-long-running-transactions/
parent: /database-engineering-fundamental-part-1/
tags: [softwareengineering, backenddevelopment]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Copilot
---
# Article - The Cost of Long running Transactions

**The cost of a long-running update transaction that eventually failed in Postgres (or any other database for that matter.**

In Postgres, any DML transaction touching a row creates a new version of that row. if the row is referenced in indexes, those need to be updated with the new tuple id as well. There are exceptions with optimization such as heap only tuples (HOT) where the index doesn’t need to be updated but that only happens if the page where the row lives have enough space (fill factor < 100%)

If a long transaction that has updated millions of rows rolls back, then the new row versions created by this transaction (millions in my case) are now invalid and should NOT be read by any new transaction. You have many ways to address this, do you clean all dead rows eagerly on transaction rollback? Or do you do it lazily as a post-process? Or do you lock the table and clean those up until the database fully restarts?

Postgres does the lazy approach, a command called vacuum which is called periodically. Postgres attempts to remove dead rows and free up space on the page.

What's the harm of leaving those dead rows in? It's not really correctness issues at all, in fact, transactions know not to read those dead rows by checking the state of the transaction that created them. This is however an expensive check, the check to see if the transaction that created this row is committed or rolled back. Also, the fact that those dead rows live in disk pages with alive rows makes an IO not efficient as the database has to filter out dead rows. For example, a page may have contained 1000 rows, but only 1 live row and 999 dead rows, the database will make that IO but only will get a single row of it. Repeat that and you end up making more IOs. More IOs = slower performance.

Other databases do the eager approach and won’t let you even start the database before rolling back is successfully complete, using undo logs. Which one is right and which one is wrong? Here is the fun part! Nothing is wrong or right, it's all decisions that we engineers make. It's all fundamentals. It's up to you to understand and pick. Anything can work. You can make anything work if you know what you are dealing with.

[Back to Parent Page]({{ page.parent }})