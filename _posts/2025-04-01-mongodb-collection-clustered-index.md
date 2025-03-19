---
title: "Part 3: Database Engineering Fundamentals: MongoDB collection clustered index"
author: pravin_tripathi
date: 2025-03-04 04:00:00 +0530
readtime: true
img_path: /assets/img/database-engineering-fundamental-part-3/
categories: [Blogging, Article]
mermaid: true
permalink: /database-engineering-fundamental-part-3/mongodb-collection-clustered-index/
parent: /database-engineering-fundamental-part-3/
tags: [softwareengineering, backenddevelopment]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Copilot
---
# MongoDB collection clustered index

[https://www.mongodb.com/docs/v6.2/core/clustered-collections/](https://www.mongodb.com/docs/v6.2/core/clustered-collections/)

# Clustered Collections

*New in version 5.3*.

# **Overview**

Starting in MongoDB 5.3, you can create a collection with a [clustered index](https://www.mongodb.com/docs/v6.2/reference/method/db.createCollection/#std-label-db.createCollection.clusteredIndex). Collections created with a clustered index are called clustered collections.

# **Benefits**

Because clustered collections store documents ordered by the [clustered index](https://www.mongodb.com/docs/v6.2/reference/method/db.createCollection/#std-label-db.createCollection.clusteredIndex) key value, clustered collections have the following benefits compared to non-clustered collections:

- Faster queries on clustered collections without needing a secondary index, such as queries with range scans and equality comparisons on the clustered index key.
- Clustered collections have a lower storage size, which improves performance for queries and bulk inserts.
- Clustered collections can eliminate the need for a secondary [TTL (Time To Live) index.](https://www.mongodb.com/docs/v6.2/indexes/#std-label-ttl-index)
    - A clustered index is also a TTL index if you specify the [expireAfterSeconds](https://www.mongodb.com/docs/v6.2/reference/method/db.createCollection/#std-label-db.createCollection.expireAfterSeconds) field.
    - To be used as a TTL index, the `_id` field must be a supported date type. See [TTL Indexes.](https://www.mongodb.com/docs/v6.2/core/index-ttl/#std-label-index-feature-ttl)
    - If you use a clustered index as a TTL index, it improves document delete performance and reduces the clustered collection storage size.
- Clustered collections have additional performance improvements for inserts, updates, deletes, and queries.
    - All collections have an [_id index.](https://www.mongodb.com/docs/v6.2/indexes/#std-label-index-type-id)
    - A non-clustered collection stores the `_id` index separately from the documents. This requires two writes for inserts, updates, and deletes, and two reads for queries.
    - A clustered collection stores the index and the documents together in `_id` value order. This requires one write for inserts, updates, and deletes, and one read for queries.

# **Behavior**

Clustered collections store documents ordered by the [clustered index](https://www.mongodb.com/docs/v6.2/reference/method/db.createCollection/#std-label-db.createCollection.clusteredIndex) key value.

You can only have one clustered index in a collection because the documents can be stored in only one order. Only collections with a clustered index store the data in sorted order.

You can have a clustered index and add [secondary indexes](https://www.mongodb.com/docs/v6.2/reference/glossary/#std-term-secondary-index) to a clustered collection. Clustered indexes differ from secondary indexes:

- A clustered index can only be created when you create the collection.
- The clustered index keys are stored with the collection. The collection size returned by the [`collStats`](https://www.mongodb.com/docs/v6.2/reference/command/collStats/#mongodb-dbcommand-dbcmd.collStats) command includes the clustered index size.

## **Important**

### **Backward-Incompatible Feature**

You must drop clustered collections before you can downgrade to a version of MongoDB earlier than 5.3.

# **Limitations**

Clustered collection limitations:

- You cannot transform a non-clustered collection to a clustered collection, or the reverse. Instead, you can:
    - Read documents from one collection and write them to another collection using an [aggregation pipeline](https://www.mongodb.com/docs/v6.2/aggregation/#std-label-aggregation-pipeline-intro) with an [`$out`](https://www.mongodb.com/docs/v6.2/reference/operator/aggregation/out/#mongodb-pipeline-pipe.-out) stage or a [`$merge`](https://www.mongodb.com/docs/v6.2/reference/operator/aggregation/merge/#mongodb-pipeline-pipe.-merge) stage.
    - Export collection data with [`mongodump`](https://www.mongodb.com/docs/database-tools/mongodump/#mongodb-binary-bin.mongodump) and import the data into another collection with [`mongorestore`.](https://www.mongodb.com/docs/database-tools/mongorestore/#mongodb-binary-bin.mongorestore)
- By default, if a [secondary index](https://www.mongodb.com/docs/v6.2/reference/glossary/#std-term-secondary-index) exists on a clustered collection and the secondary index is usable by your query, the secondary index is selected instead of the clustered index.
    - You must provide a hint to use the clustered index because it is not automatically selected by the [query optimizer.](https://www.mongodb.com/docs/v6.2/core/query-plans/)
    - The [clustered index](https://www.mongodb.com/docs/v6.2/reference/method/db.createCollection/#std-label-db.createCollection.clusteredIndex) is not automatically used by the query optimizer if a usable secondary index exists.
    - When a query uses a clustered index, it will perform a [bounded collection scan.](https://www.mongodb.com/docs/v6.2/reference/glossary/#std-term-bounded-collection-scan)
- The clustered index key must be on the `_id` field.
- You cannot hide a clustered index. See [Hidden indexes.](https://www.mongodb.com/docs/v6.2/core/index-hidden/)
- If there are secondary indexes for the clustered collection, the collection has a larger storage size. This is because secondary indexes on a clustered collection with large clustered index keys may have a larger storage size than secondary indexes on a non-clustered collection.
- Clustered collections may not be [capped collections.](https://www.mongodb.com/docs/v6.2/core/capped-collections/#std-label-manual-capped-collection)

# **Set Your Own Clustered Index Key Values**

By default, the [clustered index](https://www.mongodb.com/docs/v6.2/reference/method/db.createCollection/#std-label-db.createCollection.clusteredIndex) key values are the unique document [object identifiers.](https://www.mongodb.com/docs/v6.2/reference/bson-types/#std-label-objectid)

You can set your own clustered index key values. Your key:

- Must contain unique values.
- Must be immutable.
- Should contain sequentially increasing values. This is not a requirement but improves insert performance.
- Should be as small in size as possible.
    - A clustered index supports keys up to 8 MB in size, but a much smaller clustered index key is best.
    - A large clustered index key causes the clustered collection to increase in size and secondary indexes are also larger. This reduces the performance and storage benefits of the clustered collection.
    - Secondary indexes on clustered collections with large clustered index keys may use more space compared to secondary indexes on non-clustered collections.

# **Examples**

This section shows clustered collection examples.

### **`Create` Example**

The following [`create`](https://www.mongodb.com/docs/v6.2/reference/command/create/#mongodb-dbcommand-dbcmd.create) example adds a [clustered collection](https://www.mongodb.com/docs/v6.2/core/clustered-collections/#std-label-clustered-collections) named `products`:

```jsx
db.runCommand( {create:"products",clusteredIndex: {"key": {_id:1 },"unique":true,"name":"products clustered key" }} )
```

In the example, [clusteredIndex](https://www.mongodb.com/docs/v6.2/reference/command/create/#std-label-create.clusteredIndex) specifies:

- `"key": { _id: 1 }`, which sets the clustered index key to the `_id` field.
- `"unique": true`, which indicates the clustered index key value must be unique.
- `"name": "products clustered key"`, which sets the clustered index name.

### **`db.createCollection` Example**

The following [`db.createCollection()`](https://www.mongodb.com/docs/v6.2/reference/method/db.createCollection/#mongodb-method-db.createCollection) example adds a [clustered collection](https://www.mongodb.com/docs/v6.2/core/clustered-collections/#std-label-clustered-collections) named `stocks`:

```jsx
db.createCollection("stocks",   {clusteredIndex: {"key": {_id:1 },"unique":true,"name":"stocks clustered key" } })
```

In the example, [clusteredIndex](https://www.mongodb.com/docs/v6.2/reference/method/db.createCollection/#std-label-db.createCollection.clusteredIndex) specifies:

- `"key": { _id: 1 }`, which sets the clustered index key to the `_id` field.
- `"unique": true`, which indicates the clustered index key value must be unique.
- `"name": "stocks clustered key"`, which sets the clustered index name.

### **Date Clustered Index Key Example**

The following [`create`](https://www.mongodb.com/docs/v6.2/reference/command/create/#mongodb-dbcommand-dbcmd.create) example adds a clustered collection named `orders`:

```jsx
db.createCollection("orders",   {clusteredIndex: {"key": {_id:1 },"unique":true,"name":"orders clustered key" } })
```

In the example, [clusteredIndex](https://www.mongodb.com/docs/v6.2/reference/method/db.createCollection/#std-label-db.createCollection.clusteredIndex) specifies:

- `"key": { _id: 1 }`, which sets the clustered index key to the `_id` field.
- `"unique": true`, which indicates the clustered index key value must be unique.
- `"name": "orders clustered key"`, which sets the clustered index name.

The following example adds documents to the `orders` collection:

```jsx
db.orders.insertMany( [   {_id:ISODate("2022-03-18T12:45:20Z" ),"quantity":50,"totalOrderPrice":500 },   {_id:ISODate("2022-03-18T12:47:00Z" ),"quantity":5,"totalOrderPrice":50 },   {_id:ISODate("2022-03-18T12:50:00Z" ),"quantity":1,"totalOrderPrice":10 }] )
```

The `_id` [clusteredIndex](https://www.mongodb.com/docs/v6.2/reference/command/create/#std-label-create.clusteredIndex) key stores the order date.

If you use the `_id` field in a range query, performance is improved. For example, the following query uses `_id` and [`$gt`](https://www.mongodb.com/docs/v6.2/reference/operator/aggregation/gt/#mongodb-expression-exp.-gt) to return the orders where the order date is greater than the supplied date:

```jsx
db.orders.find( {_id: {$gt:ISODate("2022-03-18T12:47:00.000Z" ) } } )
```

Example output:

`[   {      _id: ISODate( "2022-03-18T12:50:00.000Z" ),      quantity: 1,      totalOrderPrice: 10   }]`

### **Determine if a Collection is Clustered**

To determine if a collection is clustered, use the [`listCollections`](https://www.mongodb.com/docs/v6.2/reference/command/listCollections/#mongodb-dbcommand-dbcmd.listCollections) command:

```jsx
db.runCommand( {listCollections:1 } )
```

For clustered collections, you will see the [clusteredIndex](https://www.mongodb.com/docs/v6.2/reference/command/create/#std-label-create.clusteredIndex) details in the output. For example, the following output shows the details for the `orders` clustered collection:

`...name: 'orders',type: 'collection',options: {   clusteredIndex: {      v: 2,      key: { _id: 1 },      name: 'orders clustered key',      unique: true   }},...`

`v` is the index version.

[Back to Parent Page]({{ page.parent }})