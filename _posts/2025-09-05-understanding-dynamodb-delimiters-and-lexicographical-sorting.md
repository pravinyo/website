---
title: "Understanding DynamoDB Delimiters and Lexicographical Sorting"
author: pravin_tripathi
date: 2025-09-05 00:00:00 +0530
readtime: true
media_subpath: /assets/img/understanding-dynamodb-delimiters-and-lexicographical-sorting/
categories: [Blogging, Article]
mermaid: true
tags: [softwareengineering, backenddevelopment, dynamodb]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Copilot
---

As backend developers working with DynamoDB, we often encounter the recommendation to use delimiters like "\#" in our sort keys to create structured, hierarchical data patterns. But have you ever wondered how DynamoDB actually processes these delimiters, and why certain practices like zero-padding are considered essential? Let's dive deep into the mechanics behind DynamoDB's sorting behavior and uncover some critical insights that can save you from unexpected query results.

## **The Delimiter Illusion: What DynamoDB Really Sees**

When you create sort keys like `USER#12345` or `ORG#Berkshire`, you might assume DynamoDB understands the hierarchical structure you're creating. However, here's the first crucial insight: **DynamoDB treats these keys as completely opaque strings**[^2].

DynamoDB doesn't parse, interpret, or understand any delimiter structure. Whether you use:

- `USER#12345`
- `USER|12345`
- `USER_12345`
- `USERXYZ12345`

All of these are just plain strings to DynamoDB. The database has no built-in concept of hierarchical structure or special delimiter meaning.

## **How the "Structure" Actually Works**

The power of delimiters comes from leveraging DynamoDB's **lexicographical sorting behavior**. When DynamoDB sorts string sort keys, it compares them character by character using UTF-8 byte values[^1][^3]. This creates predictable grouping patterns:

```
USER#001
USER#002
USER#003
ORDER#001
ORDER#002
```

The keys naturally group together because strings starting with "USER\#" sort consecutively. Your application code then:

- **Constructs** keys using delimiter patterns
- **Parses** returned keys to extract meaningful components
- **Interprets** the hierarchical structure you designed


## **The Critical Sorting Problem: Why Zero-Padding Matters**

Here's where many developers encounter their first major surprise. Consider this common scenario: you're storing user records with sort keys like `user#1`, `user#2`, `user#10`, `user#11`, etc.

Without zero padding, DynamoDB's lexicographical sort order will be:

```
user#1
user#10    ← Comes BEFORE user#2!
user#11
user#12
...
user#19
user#2     ← Comes AFTER user#19!
user#20
...
user#9
```


### **Real-World Impact**

Let's say your e-commerce application needs to query orders 6-15 for processing. With non-padded keys, a query like `between user#6 and user#15` returns **nothing** because `user#6` comes lexicographically after `user#15`[^3].

This breaks fundamental business operations:

- Order processing systems skip legitimate records
- Range queries return unexpected results
- Business logic based on numerical ranges fails silently


### **The Solution: Zero-Padding**

With zero-padded sort keys (`user#001`, `user#002`, etc.), lexicographical sorting matches numerical sorting:

```
user#001
user#002
user#003
...
user#010
user#011
user#012
```

Now `between user#006 and user#015` works exactly as business logic expects.

## **Understanding DynamoDB's Internal Architecture**

You might wonder: "If DynamoDB uses B-trees for sort keys, why doesn't it sort numerically?" This touches on a fundamental aspect of how database systems work.

**B-trees are storage structures, not comparison logic**. A B-tree efficiently stores and retrieves sorted data, but it needs a comparison function to determine order[^10][^11]. DynamoDB uses standard comparison functions:

- **Numbers**: Numerical comparison (1 < 2 < 10 < 20)
- **Strings**: Lexicographical comparison (UTF-8 byte order)
- **Binary**: Binary comparison

The B-tree provides O(log n) performance for whatever order the comparison function defines, but it doesn't change what that order is[^3].

## **Delimiter Symbol Choice and UTF-8 Ordering**

Since any valid string character can serve as a delimiter, you might wonder which to choose. Different symbols sort at different positions in the UTF-8 order:

**Early in sort order:**

- `#` (hash) - UTF-8 value 35
- `$` (dollar) - UTF-8 value 36
- `-` (hyphen) - UTF-8 value 45
- `:` (colon) - UTF-8 value 58

**Later in sort order:**

- `_` (underscore) - UTF-8 value 95
- `|` (pipe) - UTF-8 value 124

The "\#" symbol is popular because it sorts early, rarely conflicts with actual data, and has established community convention[^12][^2].

## **Range Query Performance: Efficiency vs. Expectations**

Here's some good news: DynamoDB's range queries are highly efficient. When you query `between user#010 and user#015`, DynamoDB:

1. **Directly navigates** to the starting point using its B-tree index
2. **Scans only the specified range**
3. **Stops at the endpoint**

No unnecessary scanning occurs before or after your range[^13]. The performance issue isn't about efficiency—it's about getting the results you actually intended.

## **Best Practices for Production Systems**

Based on these insights, here are key recommendations:

### **1. Choose Consistent Delimiters**

- Use "\#" for broad compatibility and convention
- Avoid characters that appear in your actual data
- Document your delimiter strategy for team consistency


### **2. Always Zero-Pad Numeric Components**

```
// Good
user#001, user#002, user#010

// Problematic  
user#1, user#2, user#10
```


### **3. Design for Lexicographical Order**

Structure your keys so lexicographical sorting matches your business logic requirements[^2]:

```
[country]#[region]#[state]#[county]#[city]#[neighborhood]
```


### **4. Test Your Range Queries**

Always test range queries with realistic data sets that include the edge cases around lexicographical vs. numerical ordering.

## **Key Takeaways**

Understanding DynamoDB's delimiter and sorting behavior is crucial for building reliable applications:

1. **DynamoDB sees delimiters as plain string characters** - the structure exists only in your application logic
2. **Lexicographical sorting can break intuitive numerical ranges** - always use zero-padding for numeric components
3. **B-trees provide efficiency, not different comparison logic** - string comparison remains lexicographical regardless of internal storage structure
4. **Range queries are performant** - the issue is getting the right results, not query efficiency

By mastering these concepts, you can design DynamoDB schemas that behave predictably and avoid the common pitfalls that lead to missing data in production systems. Remember: in DynamoDB, your key design directly impacts your query capabilities, so invest the time upfront to get the structure right.

The delimiter patterns that seem like DynamoDB magic are actually elegant applications of fundamental computer science principles - lexicographical sorting and B-tree indexing working together to create powerful NoSQL access patterns.
<span style="display:none">[^4][^5][^6][^7][^8][^9]</span>

<div style="text-align: center">⁂</div>

[^1]: https://aws.amazon.com/blogs/database/effective-data-sorting-with-amazon-dynamodb/

[^2]: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-sort-keys.html

[^3]: https://www.craftsmensoftware.com/mastering-sorting-techniques-in-dynamodb-from-chaos-to-clarity/

[^4]: https://jonathanbradbury.dev/blog/2022-08-15-dynamodb-lexicographical-sorting/

[^5]: https://stackoverflow.com/questions/75017825/dynamodb-sort-key-not-returning-data-in-sorted-order

[^6]: https://benoitboure.com/understanding-the-dynamodb-sort-key-order

[^7]: https://www.youtube.com/watch?v=XpZeppLmABk

[^8]: https://aws.plainenglish.io/all-the-dynamodb-query-methods-and-how-they-work-6b537080372d

[^9]: https://www.clouddefense.ai/glossary/aws/sort-key

[^10]: https://engineering.cred.club/dynamodb-internals-90c87184ab88

[^11]: https://www.hellointerview.com/learn/system-design/deep-dives/dynamodb

[^12]: https://aws.amazon.com/blogs/database/choosing-the-right-dynamodb-partition-key/

[^13]: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-query-scan.html

