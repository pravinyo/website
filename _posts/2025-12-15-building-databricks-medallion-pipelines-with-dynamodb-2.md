---
title: "Building Databricks Medallion Pipelines with DynamoDB – Part 2: Silver Layer, Data Quality, and Integration"
author: pravin_tripathi
date: 2025-12-15 00:00:00 +0530
readtime: true
media_subpath: /assets/img/building-databricks-medallion-pipelines-with-dynamodb/
categories: [Blogging, Article]
mermaid: true
tags: [softwareengineering, backenddevelopment,databricks, dynamodb, medallionarchitecture, dataengineering]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Gemini AI
---

In [Part 1](/posts/building-databricks-medallion-pipelines-with-dynamodb-1/), you learned how the **Lakehouse** and **Medallion architecture** organize data into Bronze, Silver, and Gold layers, and how to ingest raw data from Amazon DynamoDB into a **Bronze Delta table** on Databricks.  

Part 2 focuses on the **Silver layer**—the most critical transformation stage in a Medallion pipeline. This is where raw Bronze data is cleaned, validated, deduplicated, and enriched into a trusted **"enterprise view"** that analysts, data scientists, and downstream systems can rely on. 

In this article, you will learn:

- What the Silver layer is for and why it matters.
- How to apply data quality rules and track rejections.
- How to clean, standardize, and enrich data with dimension joins.
- Beginner‑friendly explanations of the core PySpark methods used in Silver transformations.
- How to build streaming Silver pipelines with Databricks declarative patterns.  

## The Silver layer: Engine of integration and quality

The transition from Bronze to Silver is where the heaviest **data engineering work** happens. 

### What Silver does?

The Silver layer takes raw, unvalidated Bronze data and applies: 

- **Schema enforcement** – converting dynamic DynamoDB types into strict analytical types (strings → dates, strings → doubles, etc.).
- **Data validation** – filtering out records with null IDs, negative amounts, or invalid timestamps.
- **Deduplication** – removing duplicate events by business keys (e.g., transaction ID).
- **Normalization** – flattening nested JSON structures, trimming strings, lowercasing categorical fields.
- **Enrichment** – joining transactional data with dimension tables (customers, products, regions) to add business context.
- **Late/out‑of‑order handling** – resolving events that arrive after their logical timestamp.

The result is a set of **conformed tables** that represent business entities like customers, transactions, or products in a consistent, reliable structure. 

### Data quality as code: expectations

Modern data platforms like Databricks support the idea of **"expectations"**: declarative rules about what constitutes valid data. 

For example:

- "Transaction ID must not be null."
- "Amount must be positive."
- "Email must contain an `@` symbol."

When a record fails an expectation, you can:

- **Drop** it (exclude from Silver).
- **Quarantine** it (write to a separate "bad records" table for manual review).
- **Allow** it with a warning (for soft validations).

Expectations are tracked over time, giving you **data quality metrics** (e.g., "98.5% of records passed validation this week"). 

### Silver vs Bronze vs Gold

To recap from Part 1:  


| Aspect | Bronze | Silver | Gold |
| :-- | :-- | :-- | :-- |
| **Data quality** | Unvalidated | Cleaned & validated | Highly refined |
| **Schema** | Flexible/source‑like | Strictly enforced | Optimized for BI |
| **Transformation** | Minimal (metadata only) | Dedup, normalization, joins | Aggregation, denormalization |
| **Primary goal** | Historical archive | Integration & quality | Business insights |
| **Audience** | Engineers (debugging) | Analysts, data scientists | Executives, dashboards |

Silver is the **"single source of truth"** for clean, conformed data.

## Silver transformation implementation

The following sections walk through a complete Silver transformation with step‑by‑step explanations of what the code does and how each PySpark method works. 

### Read Bronze data

```python
from pyspark.sql.functions import (
    col, when, trim, lower, to_date, isnan, current_timestamp
)

print("Reading bronze layer data...")
bronze_df = spark.read.format("delta").load(
    "s3a://my-data-lake/bronze/customer_transactions"
)

original_count = bronze_df.count()
print(f"Bronze records: {original_count:,}")
```

**What this does:**

- Loads the Bronze Delta table into a DataFrame.
- Counts the rows to establish a baseline for quality metrics (how many records you started with). 

**PySpark methods:**

- `spark.read.format("delta").load(path)` – reads a Delta table from a storage path and returns a DataFrame. 
- `.count()` – returns the number of rows; triggers a distributed read across partitions (can be expensive on very large tables). 

### Define validation rules

```python
validation_rules = {
    "transaction_id": {
        "rule": lambda df: df.filter(col("transaction_id").isNotNull()),
        "description": "transaction_id must not be null"
    },
    "customer_id": {
        "rule": lambda df: df.filter(col("customer_id").isNotNull() & (col("customer_id") > 0)),
        "description": "customer_id must be positive and not null"
    },
    "amount": {
        "rule": lambda df: df.filter((col("amount") > 0) & (~isnan(col("amount")))),
        "description": "amount must be positive and not NaN"
    },
    "transaction_date": {
        "rule": lambda df: df.filter(col("transaction_date").isNotNull()),
        "description": "transaction_date must not be null"
    }
}
```

**What this does:**

- Defines a dictionary of **validation rules**, where each rule is:
    - A **function** (lambda) that filters the DataFrame.
    - A **description** explaining the rule (useful for logs and quality reports). 

**PySpark methods:**

- `col("column_name")` – creates a column reference for use in expressions. 
- `.isNotNull()` – returns a boolean column: `True` where the column is not null. 
- `.filter(condition)` – keeps only rows where the condition is `True`. 
- `&` – logical AND operator for column expressions.
- `~` – logical NOT operator.
- `isnan(col)` – checks if a numeric column is NaN (Not a Number). 

### Apply validation and track rejections

```python
validated_df = bronze_df
rejected_records = {}

for rule_name, rule_config in validation_rules.items():
    before_count = validated_df.count()
    validated_df = rule_config["rule"](validated_df)
    after_count = validated_df.count()
    rejected = before_count - after_count
    
    rejected_records[rule_name] = rejected
    print(f"  {rule_name}: {rejected:,} rejected ({100*rejected/original_count:.2f}%)")
    print(f"    Reason: {rule_config['description']}")
```

**What this does:**

- Starts with the full Bronze DataFrame.
- Loops through each validation rule:
    - Counts rows **before** applying the rule.
    - Applies the filter (drops invalid rows).
    - Counts rows **after** to see how many were rejected.
    - Logs the rejection count and percentage. 

**Why this matters:**

- You now have a **quality audit trail** showing how many records failed each rule.
- If rejection rates spike, it signals a problem upstream (e.g., a bug in the source app, or a schema change in DynamoDB). 

**Example output:**

```
  transaction_id: 1,245 rejected (0.12%)
    Reason: transaction_id must not be null
  customer_id: 8,934 rejected (0.89%)
    Reason: customer_id must be positive and not null
  amount: 3,398 rejected (0.34%)
    Reason: amount must be positive and not NaN
  transaction_date: 0 rejected (0.00%)
    Reason: transaction_date must not be null
```

### Clean and standardize columns

```python
print("\nApplying cleansing transformations...")

cleansed_df = validated_df.select(
    # Rename and cast columns
    col("transaction_id").alias("txn_id"),
    col("customer_id").alias("cust_id"),
    col("amount").cast("double").alias("txn_amount"),
    
    # Parse dates
    to_date(col("transaction_date"), "yyyy-MM-dd").alias("txn_date"),
    
    # Standardize categories (trim, lowercase, fill nulls)
    lower(trim(col("product_category"))).alias("product_category"),
    when(col("product_category").isNull(), "uncategorized")
        .otherwise(lower(trim(col("product_category"))))
        .alias("product_category_clean"),
    
    # Keep metadata from bronze
    col("_ingestion_timestamp"),
    col("_source_table"),
    
    # Add silver processing timestamp
    current_timestamp().alias("_processed_timestamp")
).dropDuplicates(["txn_id"])

final_count = cleansed_df.count()
duplicates_removed = original_count - final_count - sum(rejected_records.values())

print(f"  Duplicates removed: {duplicates_removed:,}")
print(f"  Final record count: {final_count:,}")
```

**What this does:**

- **Renames** columns to shorter, standardized names (`transaction_id` → `txn_id`).
- **Casts** the `amount` column to `double` (guarantees numeric type).
- **Parses** the `transaction_date` string into a proper `date` type.
- **Normalizes** the `product_category`:
    - Trims whitespace and converts to lowercase for consistency.
    - Fills null categories with "uncategorized".
- Keeps Bronze metadata (`_ingestion_timestamp`, `_source_table`).
- Adds a `_processed_timestamp` to track when Silver processing happened.
- Removes **duplicate transactions** by `txn_id` (keeps the first occurrence). 

**PySpark methods:**

- `.select(...)` – projects (selects) specific columns and transformations; think of it as "build a new table with exactly these columns." 
- `.alias("new_name")` – renames a column. 
- `.cast("type")` – converts a column to a different data type (e.g., string → double). 
- `to_date(col, format)` – parses a string column into a date using the specified format. 
- `trim(col)` – removes leading and trailing spaces. 
- `lower(col)` – converts strings to lowercase. 
- `when(condition, value).otherwise(value)` – SQL‑style `CASE WHEN`; used here to replace nulls with a default value. 
- `current_timestamp()` – returns the current cluster timestamp. 
- `.dropDuplicates(["col1", "col2", ...])` – removes duplicate rows based on the given key columns, keeping the first occurrence. 

**Why this matters:**

- Consistent naming and types make Silver data much easier to query and join.
- Date parsing enables date‑based filtering and partitioning (critical for performance).
- Deduplication prevents double‑counting in downstream aggregations. 

### Enrich with dimension tables

```python
print("\nEnriching with customer dimension data...")

# Load customer dimension (from Delta or another Bronze table)
customer_dim = spark.read.format("delta").load(
    "s3a://my-data-lake/dimensions/customers"
).select(
    col("customer_id").alias("cust_id"),
    col("segment").alias("customer_segment"),
    col("region"),
    col("country")
)

# Join transactional data with customer attributes
enriched_df = cleansed_df.join(
    customer_dim,
    on="cust_id",
    how="left"
)

print(f"  Enrichment complete: {enriched_df.count():,} records")
```

**What this does:**

- Loads a **customer dimension table** (another Delta table storing customer attributes).
- Selects and renames columns to match the transactional data's key (`cust_id`).
- Performs a **left join**: keeps all transactions, adding customer segment and region where available (nulls if customer not found in the dimension).  

**PySpark methods:**

- `.join(other_df, on="key", how="left")` – joins two DataFrames.
    - `on` specifies the join key(s).
    - `how="left"` keeps all rows from the left DataFrame (transactions), even if there's no match on the right (dimension). 

**Why this matters:**

- Enrichment turns transactional records into **business‑meaningful records** by adding context (e.g., "This transaction was by a premium customer in the North region").
- Silver becomes the **single source of truth** for integrated data, decoupling analysts from having to manually join transactions with dimensions every time. 

### Write to Silver layer

```python
SILVER_PATH = "s3a://my-data-lake/silver/customer_transactions"

enriched_df.write \
    .format("delta") \
    .mode("append") \
    .option("mergeSchema", "true") \
    .partitionBy("txn_date") \
    .save(SILVER_PATH)

print(f"\n✓ Silver transformation complete!")
print(f"  Location: {SILVER_PATH}")
```

**What this does:**

- Writes the cleaned, validated, enriched DataFrame as a **Silver Delta table**.
- Uses `append` mode to add new data without deleting previous Silver batches.
- Partitions the table by `txn_date` for query performance (date‑based filters can skip irrelevant partitions). 

**PySpark methods:**

- `.write.format("delta").mode("append")` – same as Bronze, but now targeting the Silver path. 
- `.partitionBy("txn_date")` – organizes data into subfolders by date; speeds up queries that filter on date. 
- `.option("mergeSchema","true")` – allows schema evolution (new columns can be added over time). 

**Why partition by date?**

- Analytical queries often filter by date ranges (e.g., "last 30 days").
- Partitioning lets Spark skip reading entire historical data and only scan relevant date folders. 

### Data quality report

```python
print("\n=== DATA QUALITY REPORT ===")
print(f"Input records (bronze):  {original_count:,}")
print(f"Output records (silver): {final_count:,}")
print(f"Quality score: {100*final_count/original_count:.2f}%")
print(f"\nRejection breakdown:")
for rule, count in rejected_records.items():
    print(f"  {rule}: {count:,}")
```

**Example output:**

```
=== DATA QUALITY REPORT ===
Input records (bronze):  1,000,000
Output records (silver): 985,423
Quality score: 98.54%

Rejection breakdown:
  transaction_id: 1,245
  customer_id: 8,934
  amount: 3,398
```

**What this does:**

- Summarizes the transformation:
    - How many records started (Bronze).
    - How many passed validation and made it to Silver.
    - Overall quality score (% passed).
    - Breakdown by validation rule. 

**Why this matters:**

- Quality reports let you **monitor pipeline health** over time.
- If the quality score drops suddenly, you know to investigate upstream issues (e.g., a bug in the source app or a schema change in DynamoDB). 


## Streaming Silver with Lakeflow Spark Declarative Pipelines

The code above is **batch‑oriented**: it reads the entire Bronze table, processes it, and writes Silver. For **real‑time or near‑real‑time** pipelines, Databricks offers **Lakeflow Spark Declarative Pipelines** (formerly Delta Live Tables), which let you define Silver transformations as **streaming tables**. 

### Declarative Silver example

```python
from pyspark import pipelines as dp
from pyspark.sql.functions import col, lower, trim

@dp.table(
   name="customers_silver",
   comment="Cleansed customer profiles with conformed emails and names."
)
def customers_silver():
   return (
       dp.read_stream("catalog.bronze.customers_raw")
      .select(
           col("customer_id").cast("int"),
           trim(col("first_name")).alias("first_name"),
           trim(col("last_name")).alias("last_name"),
           lower(trim(col("email"))).alias("email"),
           col("account_status")
       )
      .filter(col("email").contains("@") & col("customer_id").isNotNull())
   )
```

**What this does:**

- Defines a **streaming table** called `customers_silver`.
- Uses `dp.read_stream(...)` to read from the Bronze table in **streaming mode** (processes new data as it arrives).
- Applies the same transformations (trim, lowercase, cast) as the batch code.
- Filters out invalid records (email must contain "@", customer_id not null). 

**Key concepts:**

- `@dp.table(...)` – decorator that tells Databricks this is a managed table in a declarative pipeline. 
- `dp.read_stream(...)` – continuously reads new data from the Bronze table as it's written. 
- Databricks manages the **streaming checkpoints, state, and retries** for you. 

**Benefits of streaming Silver:**

- **Lower latency**: new Bronze data flows to Silver within seconds or minutes instead of waiting for batch jobs.
- **Automatic orchestration**: Databricks schedules and monitors the streaming query; no need to manage cron jobs or triggers manually.
- **Built‑in quality metrics**: declarative pipelines track expectation pass/fail rates over time. 


## Silver as the "enterprise view"

By the end of Silver processing, you have: 

- **High‑quality, conformed data** where schemas are enforced, nulls are handled, and invalid records are removed.
- **Integrated entities** where transactions are enriched with customer/product/region context.
- **Deduplicated records** so downstream analytics don't double‑count events.
- **Partitioned tables** optimized for common query patterns (e.g., date‑based filtering).

This makes Silver the **single source of truth** for analysts, data scientists, and ML workflows. 

Analysts no longer need to:

- Worry about raw JSON structures or DynamoDB quirks.
- Write complex joins every time they query transactional data.
- Handle data quality issues in ad‑hoc queries. 

Instead, they query clean Silver tables and trust the data.


## Summary

In this article, you learned:

- The **role of the Silver layer**: integration, quality, normalization, and enrichment. 
- How to implement **data quality as code** with validation rules and rejection tracking.  
- How to **clean and standardize columns** using `select`, `cast`, `to_date`, `trim`, `lower`, `when/otherwise`. 
- How to **enrich transactional data** with dimension joins using `.join(..., how="left")`. 
- How to **partition Silver tables by date** for query performance. 
- How to build **streaming Silver pipelines** with Lakeflow Spark Declarative Pipelines for near‑real‑time processing. 
- How to generate **data quality reports** that track rejection rates and overall pipeline health. 

In **Part 3**, the focus shifts to the **Gold layer**, where Silver data is aggregated into business‑ready metrics (daily sales, customer lifetime value, churn cohorts), and how to orchestrate the entire Bronze → Silver → Gold flow with **event‑driven workflows** on Databricks, including writing Gold results back to DynamoDB for serving to APIs and applications.  [Read Part 3 →](/posts/building-databricks-medallion-pipelines-with-dynamodb-3/)
