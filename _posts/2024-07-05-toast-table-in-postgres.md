---
title: "Part 5: Database Engineering Fundamentals: TOAST table in Postgres"
author: pravin_tripathi
date: 2024-07-05 08:00:00 +0530
readtime: true
img_path: /assets/img/database-engineering-fundamental-part-5/
categories: [Blogging, Article]
mermaid: true
permalink: /database-engineering-fundamental-part-5/toast-table-in-postgres/
parent: /database-engineering-fundamental-part-5/
tags: [softwareengineering, backenddevelopment]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Copilot
---
# TOAST table in Postgres

### **TOAST Table in PostgreSQL**

**TOAST** (The Oversized-Attribute Storage Technique) is a mechanism used by PostgreSQL to efficiently store **large data** types (such as large text fields, byte arrays, or large objects) that do not fit within the regular 8 KB page size for a table. PostgreSQL, by default, uses 8 KB blocks (or pages) for storing table data. If a column's value exceeds this size, PostgreSQL uses **TOAST tables** to store the large data outside of the regular table, ensuring that performance isn't compromised.

### **Why Do We Need TOAST?**

- **Limitations of the 8 KB Page Size**: PostgreSQL organizes data in fixed-size pages (typically 8 KB). If a single column's data exceeds this limit, storing it directly in the main table would either waste space or require excessive storage and memory.
- **Efficient Handling of Large Data**: Instead of storing large values directly in the main table, PostgreSQL stores them in a **TOAST table**. This method helps manage large data types, like:
    - **TEXT** or **VARCHAR** columns with very long strings.
    - **BYTEA** columns that store binary data (e.g., images, files).
    - **Large Object (LOB)** data types.

### **How Does TOAST Work?**

1. **TOAST Storage**:
    - Each table that contains large data types will have an associated **TOAST table**. These tables are automatically created by PostgreSQL when needed. The TOAST table stores the large values for the main table in a separate storage area.
    - PostgreSQL uses **out-of-line storage** for values that exceed a certain threshold (about 2 KB by default). Smaller values are stored directly in the regular table page, while larger ones are stored in TOAST tables.
2. **Compression**:
    - When PostgreSQL detects that a column contains large data, it may use **compression** (if configured) to reduce the size of the data before storing it in the TOAST table. This helps save storage space.
    - Common compression algorithms used include **PGLZ** and **LZ4**. Compression is especially useful for text or data that can be highly compressed (e.g., repetitive strings).
3. **Out-of-Line Storage**:
    - If the data exceeds a certain threshold (around 2 KB by default), PostgreSQL stores the large column's data **out-of-line**, i.e., outside the main table, in the TOAST table. The main table stores a **pointer** to the location of the data in the TOAST table.
    - This allows PostgreSQL to keep the main table's pages small, ensuring that regular queries can still operate efficiently, even with large data types present.
4. **Chunking**:
    - Large values are **split into chunks** (usually 2 KB each) and stored across multiple pages in the TOAST table. This ensures that PostgreSQL doesn’t need to store excessively large values on a single page, which could be inefficient.
    - When a value is requested, PostgreSQL reconstructs it by reading these chunks from the TOAST table.

### **TOAST Table Structure**

A TOAST table is not something a user normally interacts with directly. However, you can observe its structure and use it indirectly in PostgreSQL:

- **Automatic Creation**: PostgreSQL automatically creates TOAST tables for any regular table that has large data columns. For example, if a table has a `TEXT` or `BYTEA` column with large values, PostgreSQL creates a TOAST table behind the scenes.
- **TOAST Tables Naming Convention**: The TOAST table is automatically named based on the original table name. For example, if you have a table named `my_table`, the associated TOAST table might be named `pg_toast.pg_toast_<oid_of_my_table>`. Here, `<oid_of_my_table>` refers to the internal object ID of the original table.
- **Columns in TOAST Table**:
A typical TOAST table has three columns:
    1. **chunk_id**: A unique identifier for each chunk of the large object.
    2. **chunk_data**: The actual data for each chunk (compressed and stored in a binary format).
    3. **main_table_row_id**: A reference (foreign key) to the corresponding row in the main table.

### **TOAST Table Example**

If you have a table that stores large text data:

```sql
CREATE TABLE my_table (
    id serial primary key,
    large_text TEXT
);

```

If the `large_text` column contains data larger than 2 KB, PostgreSQL will automatically move the `large_text` data into a TOAST table, and the `my_table` will store a reference to the data's location in the TOAST table. You won’t have to manually interact with the TOAST table—PostgreSQL handles the process for you.

### **Viewing the TOAST Table**

While you usually don't interact with TOAST tables directly, you can inspect the TOAST table if necessary using PostgreSQL system catalogs. Here’s an example of how you can find a TOAST table associated with a regular table:

```sql
SELECT
    t.relname AS toast_table
FROM
    pg_class t
    JOIN pg_attribute a ON a.attrelid = t.oid
WHERE
    t.relkind = 'r' -- regular table
    AND a.attname = 'large_text';  -- The column name that uses TOAST

```

This query will return the name of the TOAST table for the column `large_text`.

### **Managing TOAST**

- **Disabling TOAST for Specific Columns**: PostgreSQL allows you to configure whether certain columns use TOAST or not, by using the `storage` attribute:
    
    ```sql
    CREATE TABLE my_table (
        id serial primary key,
        large_text TEXT STORAGE EXTERNAL
    );
    
    ```
    
    The `STORAGE` parameter can be:
    
    - **PLAIN**: No TOAST, store directly in the main table.
    - **EXTERNAL**: Store large data externally (use TOAST).
    - **MAIN**: Store small values directly on the main table page (default).
    
    By default, PostgreSQL uses **EXTERNAL** for large data types, but you can tweak it if necessary.
    
- **TOAST Compression**: You can adjust compression settings for TOAST using `pg_catalog.pglz_compress` or other methods for fine-tuning performance.

### **Summary**

- **TOAST** is PostgreSQL's internal mechanism to handle large values in **TEXT**, **BYTEA**, or other large object data types.
- Large values are **stored outside the main table**, typically in a separate TOAST table.
- PostgreSQL uses **compression** and **chunking** to minimize the storage overhead of large values.
- TOAST improves the performance of database queries by keeping regular table pages smaller and efficient.

Thus, TOAST helps PostgreSQL efficiently manage storage and access for large data types without sacrificing performance.

[Back to Parent Page]({{ page.parent }})