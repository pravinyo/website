---
title: "Part 4: Database Engineering Fundamentals"
author: pravin_tripathi
date: 2025-03-04 01:00:00 +0530
readtime: true
img_path: /assets/img/database-engineering-fundamental-part-4/
attachment_path: /assets/document/attachment/database-engineering-fundamental-part-4/
categories: [Blogging, Article]
mermaid: true
permalink: /database-engineering-fundamental-part-4/
tags: [softwareengineering, backenddevelopment]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Copilot
---

## Database Engine Fundamentals

### What is a Database Engine?
- Library that takes care of the on-disk storage and CRUD operations
  - Can be as simple as a key-value store
  - Or as rich and complex as full ACID support with transactions and foreign keys
- DBMS can use the database engine and build features on top (server, replication, isolation, stored procedures, etc.)
- Want to write a new database? Don't start from scratch - use an engine
- Sometimes referred to as Storage Engine or embedded database
- Some DBMS gives you the flexibility to switch engines like MySQL & MariaDB
- Some DBMS comes with a built-in engine that you can't change (PostgreSQL)

---

## Popular Database Engines

### MyISAM
- Stands for Indexed Sequential Access Method
- B-tree (Balanced tree) indexes point to the rows directly
- No transaction support
- Open Source & Owned by Oracle
- Inserts are fast, updates and deletes are problematic (fragments)
- Database crashes corrupt tables (have to manually repair)
- Table level locking
- MySQL, MariaDB, Percona (MySQL forks) supports MyISAM
- Used to be default engine for MySQL

### Aria
- Created by Michael Widenius
- Very similar to MyISAM
- Crash-safe unlike MyISAM
- Not owned by Oracle
- Designed specifically for MariaDB (MySQL Fork)
- In MariaDB 10.4 all system tables are Aria

### InnoDB
- B+tree - with indexes point to the primary key and the PK points to the row
- Replaces MyISAM
- Default for MySQL & MariaDB
- ACID compliant transactions support
- Foreign keys
- Tablespaces
- Row level locking
- Spatial operations
- Owned by Oracle

### XtraDB
- Fork of InnoDB
- Was the default for MariaDB until 10.1
- In MariaDB 10.2 for InnoDB [switched the default](https://mariadb.com/kb/en/library/changes-improvements-in-mariadb-102/)
- "XtraDB couldn't be kept up to date with the latest features of InnoDB and cannot be used." [link](https://mariadb.com/kb/en/library/why-does-mariadb-102-use-innodb-instead-of-xtradb/)
- [System tables](https://mariadb.com/kb/en/library/aria-storage-engine/) in MariaDB starting with 10.4 are all Aria

### SQLite
- Designed by D. Richard Hipp in 2000
- Very popular embedded database for local data
- B-Tree (LSM as extension)
- PostgreSQL-like syntax
- Full ACID & table locking
- Concurrent read & writes
- Web SQL in browsers uses it
- Included in many operating systems by default

### Berkeley DB
- Developed by Sleepycat Software in 1994 (owned by Oracle)
- Key-value embedded database
- Supports ACID transactions, locks, replications etc.
- Used to be used in Bitcoin Core (switched to LevelDB)
- Used in MemcacheDB

### LevelDB
- Written by Jeff and Sanjay from Google in 2011
- Log structured merge tree (LSM) (great for high insert and SSD)
- No transactions
- Inspired by Google BigTable
- Levels of files:
  - Memtable
  - Level 0 (young level)
  - Level 1 - 6
- As files grow, large levels are merged
- Used in Bitcoin Core blockchain, AutoCAD, Minecraft

### RocksDB
- Facebook forked LevelDB in 2012 to become RocksDB
- Transactional
- High Performance, Multi-threaded compaction
- [Many features not in LevelDB](https://github.com/facebook/rocksdb/wiki/Features-Not-in-LevelDB)
- MyRocks for MySQL, MariaDB and Percona
- MongoRocks for MongoDB
- Many more projects use it!

---

## Database Cursors

### What are Database Cursors?

Database cursors are powerful tools that allow for row-by-row processing of result sets. In PostgreSQL, a **cursor** is a database object used to retrieve rows from a result set one at a time. This is particularly useful when working with large datasets or when you need to fetch data incrementally.

#### Cursor Capabilities:
- **Efficient row-by-row processing**: Fetch and process rows one at a time or in small chunks
- **Multiple result sets**: Keep a cursor open to fetch additional rows later, helpful for procedural processing

#### Key Steps for Using a Cursor in PostgreSQL:
1. **Declare a cursor** – Define a cursor to point to a specific query or result set
2. **Open the cursor** – Execute the query and generate the result set
3. **Fetch rows from the cursor** – Retrieve individual or multiple rows
4. **Close the cursor** – Release resources after processing

### Example of Using Cursors in PostgreSQL:

#### Step 1: Declare the cursor
To declare a cursor, you need to use the `DECLARE` statement, followed by the cursor name and the SQL query.

```sql
-- Declare a cursor
DECLARE my_cursor CURSOR FOR
SELECT id, name FROM employees WHERE department = 'Sales';
```
Here, `my_cursor` is the name of the cursor, and the `SELECT` statement fetches `id` and `name` columns from the `employees` table where the `department` is 'Sales'.

#### Step 2: Fetch rows from the cursor
After declaring the cursor, you can use the `FETCH` command to retrieve rows from the result set.

```sql
-- Fetch the first row
FETCH NEXT FROM my_cursor;
```
You can also specify how many rows to fetch at once, such as `FETCH 5 FROM my_cursor` to fetch 5 rows at a time.

#### Step 3: Loop to fetch multiple rows
You can use a loop to fetch all rows one by one. Here is an example using a `LOOP` in PL/pgSQL, PostgreSQL’s procedural language.

```sql
DO $$
DECLARE
    rec RECORD;
BEGIN
    -- Declare the cursor
    DECLARE my_cursor CURSOR FOR
    SELECT id, name FROM employees WHERE department = 'Sales';

    -- Open the cursor
    OPEN my_cursor;

    -- Fetch and process rows
    LOOP
        FETCH NEXT FROM my_cursor INTO rec;
        EXIT WHEN NOT FOUND;  -- Exit when no more rows
        -- Process the row (For example, just outputting the data)
        RAISE NOTICE 'Employee ID: %, Name: %', rec.id, rec.name;
    END LOOP;

    -- Close the cursor
    CLOSE my_cursor;
END $$;
```

In this example:
- `rec` is a `RECORD` type variable that holds the fetched row.
- The `LOOP` continues fetching rows using the `FETCH NEXT` statement until there are no more rows (`EXIT WHEN NOT FOUND`).
- After processing, the cursor is closed with `CLOSE my_cursor`.

#### Step 4: Close the cursor
```sql
CLOSE my_cursor;
```
Once you’ve finished working with the cursor, it is good practice to close it. This is done with the `CLOSE` statement, as shown above.

### Important Notes:
1. **Implicit Cursors**: PostgreSQL automatically creates implicit cursors for SELECT queries outside of procedural code. For complex operations, explicit cursors are necessary.

2. **Cursor Types**:
   - **Simple cursor**: Basic cursor that fetches rows in order
   - **Scroll cursor**: Allows fetching rows both forward and backward
   - **No scroll cursor**: Can only fetch rows in one direction (default)

3. **Memory Considerations**: Cursors are more memory-efficient than loading entire result sets, but should be closed when no longer needed.

### Example in a Transaction Block:
```sql
BEGIN;

-- Declare and open the cursor
DECLARE my_cursor CURSOR FOR
SELECT id, name FROM employees WHERE department = 'HR';

-- Fetch and process rows
FETCH NEXT FROM my_cursor;

-- Close the cursor
CLOSE my_cursor;

COMMIT;
```

---

## Pros and Cons of Database Cursors

### Pros of Using Cursors

1. **Memory Efficiency**:
   - Process rows one at a time or in small batches
   - Improved resource management for large result sets
   - Avoids loading entire datasets into memory at once

2. **Better Performance for Large Datasets**:
   - Sequential processing can be more efficient for certain procedural operations
   - Ability to pause and resume fetching process
   - Advantageous for complex workflows or long-running tasks

3. **Control over Row Fetching**:
   - Control over order and frequency of row fetching
   - Explicit iteration through result sets
   - Scroll cursors provide flexibility for both forward and backward fetching

4. **Complex Query Handling**:
   - Useful for processing complex queries row-by-row
   - Commonly used in stored procedures or functions
   - Ideal for operations that need to be executed in steps

5. **Transactional Processing**:
   - Process data incrementally within transactions
   - Useful for batch updates or long-running transactions

### Cons of Using Cursors

1. **Performance Overhead**:
   - Multiple context switches between database and application code
   - Opening, fetching, and closing a cursor can be more costly than executing a single query
   - Performance issues can be exacerbated for unoptimized queries

2. **Complexity and Maintenance**:
   - Adds complexity to code
   - Requires explicit handling of cursor lifecycle
   - Forgetting to close cursors can lead to resource leaks

3. **Concurrency Issues**:
   - Can lock resources depending on usage
   - May hold locks on rows during fetching
   - Potential for deadlocks if not handled properly

4. **Limited Use Case**:
   - Not always necessary; simple SQL queries often suffice
   - Set-based operations are usually more efficient
   - Often overkill for read-heavy use cases

5. **Potential for Unintended Side Effects**:
   - Stateful nature can lead to unexpected results if data changes between fetches
   - May cause long-running transactions
   - Potential for transaction contention or deadlocks

6. **Resource Management**:
   - Requires explicit resource management
   - Open cursors consume memory and resources
   - Can cause performance issues in high-concurrency environments

### When to Use Cursors:
- When row-by-row processing is necessary
- When memory constraints are a concern
- When implementing procedural logic in stored procedures or functions
- When performing complex updates or deletions with multiple operations per row

### When to Avoid Cursors:
- For simple queries that don't require row-by-row processing
- When performance is critical
- For read-heavy operations better handled by set-based SQL
- For simple CRUD operations where set-based operations are more efficient

---

## Implementing Cursor-like Functionality in Spring Boot

In a Spring Boot application using **Spring Data JPA**, we typically work with repositories to perform CRUD (Create, Read, Update, Delete) operations on entities. However, if you need to implement **cursor-like behavior** (such as fetching results incrementally or processing rows one by one), you can use **native SQL queries** along with **`@Query` annotation** or **JPA Criteria API** in combination with pagination.
While Spring Data JPA doesn’t directly provide a `cursor` concept like in PostgreSQL, you can simulate cursor-like behavior using **pagination**, **streaming**, or custom **native queries**.

### 1. Using `@Query` Annotation with Pagination

```java
// Entity Class
@Entity
public class Employee {
    @Id
    private Long id;
    private String name;
    private String department;
    // Getters and Setters
}

// Repository Interface
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    // Custom query with pagination
    @Query("SELECT e FROM Employee e WHERE e.department = :department")
    Page<Employee> findEmployeesByDepartment(String department, Pageable pageable);
}

// Service Layer
@Service
public class EmployeeService {
    @Autowired
    private EmployeeRepository employeeRepository;

    public void processEmployeesInBatches(String department) {
        int page = 0;
        int pageSize = 10; // Batch size
        Pageable pageable = PageRequest.of(page, pageSize);

        Page<Employee> employeePage;

        // Fetch in batches (mimicking cursor behavior)
        do {
            employeePage = employeeRepository.findEmployeesByDepartment(department, pageable);
            employeePage.getContent().forEach(employee -> {
                // Process each employee
                System.out.println("Processing Employee ID: " + employee.getId());
            });

            // Move to next page (mimicking cursor movement)
            pageable = pageable.next();
            page++;

        } while (employeePage.hasContent());
    }
}
```

### 2. Using `Stream` for Processing Large Results

```java
// Repository with Streaming Query
@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    @Transactional
    @Query("SELECT e FROM Employee e WHERE e.department = :department")
    Stream<Employee> findEmployeesByDepartmentStream(String department);
}

// Service Layer (Streaming)
@Service
public class EmployeeService {
    @Autowired
    private EmployeeRepository employeeRepository;

    public void processEmployeesInStream(String department) {
        // Open a stream to process the data lazily
        try (Stream<Employee> employeeStream = employeeRepository.findEmployeesByDepartmentStream(department)) {
            employeeStream.forEach(employee -> {
                // Process each employee (row by row)
                System.out.println("Processing Employee ID: " + employee.getId());
            });
        }
    }
}
```

### 3. Using Native SQL Queries with `@Query` and Streaming

```java
// Repository with Native Query
@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    @Query(value = "SELECT * FROM employees WHERE department = :department", nativeQuery = true)
    Stream<Employee> findEmployeesByDepartmentNativeStream(String department);
}

// Service Layer (Using Native Stream)
@Service
public class EmployeeService {
    @Autowired
    private EmployeeRepository employeeRepository;

    public void processEmployeesWithNativeQueryStream(String department) {
        try (Stream<Employee> employeeStream = employeeRepository.findEmployeesByDepartmentNativeStream(department)) {
            employeeStream.forEach(employee -> {
                // Process each employee from the native query
                System.out.println("Processing Employee ID: " + employee.getId());
            });
        }
    }
}
```

---

## Server-Side vs. Client-Side Cursors

### Server-Side Cursor

A **server-side cursor** is managed by the database server, which handles the cursor state and row retrieval.

#### Characteristics:
- Cursor management is done by the database server
- Client only issues fetch commands
- Memory-efficient as rows are sent as needed
- Ideal for large datasets
- Stateful: server maintains cursor position between fetches

#### Advantages:
- Memory efficiency
- Better handling of large result sets
- Server-controlled optimizations

#### Disadvantages:
- Latency due to client-server communication
- Resource consumption on the server

#### Example in PostgreSQL:
```sql
DECLARE my_cursor CURSOR FOR
SELECT id, name FROM employees WHERE department = 'Sales';
```

### Client-Side Cursor

A **client-side cursor** is managed by the client application, which retrieves the entire result set at once.

#### Characteristics:
- Cursor management is done by the client application
- Client fetches all rows at once
- Client controls cursor position
- Used for smaller result sets that fit in memory

#### Advantages:
- Simpler implementation
- Faster for small result sets
- Lower server load

#### Disadvantages:
- High memory consumption
- Inefficient for large result sets
- No incremental fetching

#### Example in Java (JDBC):
```java
Connection conn = DriverManager.getConnection("jdbc:postgresql://localhost:5432/mydb", "user", "password");
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT id, name FROM employees");

while (rs.next()) {
    int id = rs.getInt("id");
    String name = rs.getString("name");
    // Process the row
}
```

### Comparison: Server-Side vs. Client-Side Cursors

| Aspect | Server-Side Cursor | Client-Side Cursor |
|--------|-------------------|-------------------|
| **Cursor Management** | Database server | Client application |
| **Memory Usage** | Efficient: only portions in memory | Inefficient: entire result set in memory |
| **Fetch Behavior** | Retrieving rows in batches | All rows fetched at once |
| **Resource Consumption** | Server resources | Client resources |
| **Use Case** | Large result sets | Smaller result sets |
| **Performance** | Overhead from multiple round trips | Faster for small datasets |
| **Latency** | Some latency due to batch fetching | No latency once loaded |

### Which One Should You Use?

- **Server-Side Cursors**: Best for large result sets where memory efficiency is crucial
- **Client-Side Cursors**: Appropriate for smaller result sets that fit comfortably in memory
    
[SQLServer-ServerSide-Cursor-Types.pdf]({{page.attachment_path}}/SQLServer-ServerSide-Cursor-Types.pdf)