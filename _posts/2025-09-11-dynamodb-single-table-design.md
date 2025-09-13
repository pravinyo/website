---
title: "DynamoDB Single Table Design: Strategic Key Design and GSI Optimization"
author: pravin_tripathi
date: 2025-09-11 00:00:00 +0530
readtime: true
media_subpath: /assets/img/dynamodb-single-table-design/
attachment_path: /assets/document/attachment/dynamodb-single-table-design
categories: [Blogging, Article]
mermaid: true
tags: [softwareengineering, backenddevelopment, dynamodb]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Copilot
---

DynamoDB's **single table design** remains one of the most powerful yet misunderstood patterns in NoSQL data modeling. This comprehensive guide demonstrates how to properly implement single table design using a real-world investment fund management system, focusing on strategic use of partition keys, sort keys, and GSIs while avoiding critical pitfalls like hot partitions and broken access patterns.

## The Foundation: Understanding Single Table Design

Single table design stores **multiple related entity types within one DynamoDB table**, using composite keys to maintain relationships and optimize access patterns. The core principle is **data co-location**: entities accessed together should be stored together, eliminating the need for joins and reducing query complexity.

The fundamental rule: **items that are accessed together should be stored together**. This creates what's effectively a "pre-joined" data structure that can be retrieved in a single query operation.

## Domain Model: Investment Fund Management

Let's consider a simplified investment fund management system to illustrate single table design principles.
Our system manages investment funds with multiple investors, where each **Fund + Investor combination** represents a unique **Position**. Here's how the entities relate:

- **Document**: Contains position summaries and can be latest or historical (document-level data)
- **Capital Activity**: Tracks capital-related transactions, versioned as latest or historical (document-level data)
- **Capital Call**: Specific capital call requests tied to positions (position-specific, always latest)
- **Distribution**: Profit distributions to position holders (position-specific, always latest)
- **Unfunded Commitment**: Outstanding investment commitments (position-specific, always latest)


**Entity Relationships:**

- `FUND_A + INVESTOR_1 = POSITION_1`
- `FUND_A + INVESTOR_2 = POSITION_2`

- **Document DOC001 contains:**
    - Document (latest/historical) — document-level
    - Capital Activity (latest/historical) — document-level
    - Capital Call (latest only) → POSITION_1, POSITION_2
    - Distribution (latest only) → POSITION_1, POSITION_2
    - Unfunded Commitment (latest only) → POSITION_1, POSITION_2

## The Single Table Structure

Table Name: `investment_fund`  

| PK | SK | EntityType | Amount | Status | Version | PositionId |
|----|----|------------|--------|--------|---------|------------|
| DOC001 | DOCUMENT#LATEST | Document | - | Active | Latest | - |
| DOC001 | DOCUMENT#2025-09-01 | Document | - | Historical | Historical | - |
| DOC001 | CAPITAL_ACTIVITY#LATEST | CapitalActivity | 500000 | Pending | Latest | - |
| DOC001 | CAPITAL_ACTIVITY#2025-08-15 | CapitalActivity | 450000 | Completed | Historical | - |
| DOC001 | CAPITAL_CALL#POSITION_1 | CapitalCall | 500000 | Pending | Latest | POSITION_1 |
| DOC001 | DISTRIBUTION#POSITION_1 | Distribution | 75000 | Completed | Latest | POSITION_1 |
| DOC001 | UNFUNDED_COMMITMENT#POSITION_1 | UnfundedCommitment | 1500000 | Active | Latest | POSITION_1 |
| DOC001 | CAPITAL_CALL#POSITION_2 | CapitalCall | 300000 | Pending | Latest | POSITION_2 |
| DOC001 | DISTRIBUTION#POSITION_2 | Distribution | 45000 | Completed | Latest | POSITION_2 |
| DOC001 | UNFUNDED_COMMITMENT#POSITION_2 | UnfundedCommitment | 850000 | Active | Latest | POSITION_2 |
| DOC002 | DOCUMENT#LATEST | Document | - | Active | Latest | - |
| DOC002 | CAPITAL_CALL#POSITION_1 | CapitalCall | 750000 | Completed | Latest | POSITION_1 |
| DOC002 | DISTRIBUTION#POSITION_1 | Distribution | 120000 | Pending | Latest | POSITION_1 |
| DOC002 | CAPITAL_CALL#POSITION_2 | CapitalCall | 400000 | Pending | Latest | POSITION_2 |

**Key Design Notes:**
- **Document ID as Partition Key (PK):** Ensures all entities related to a document are co-located, enabling fast, single-query document overviews.
- **Selective PositionId Usage:** The `PositionId` attribute is present only for position-specific entities (`CAPITAL_CALL`, `DISTRIBUTION`, `UNFUNDED_COMMITMENT`), keeping document-level items clean and unambiguous.
- **Document-Level Aggregates:** Entities like `Document` and `Capital Activity` represent document-wide data and do not include a `PositionId`.
- **Sort Key (SK) Prefix Patterns:** Consistent SK prefixes (e.g., `CAPITAL_CALL#`, `DISTRIBUTION#`) allow efficient entity-type filtering with DynamoDB's `begins_with()` operator, supporting targeted queries without scans.


## Core Query Patterns: The Power of Strategic Keys

Let's explore key query patterns enabled by this design. 

### Query 1: Complete Document Overview (Primary Access Pattern)

Let's say we want to retrieve all entities for a specific document, including all positions. Because all related entities share the same partition key (document ID), we can retrieve everything with a single query:

```python
import boto3
from boto3.dynamodb.conditions import Key

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('investment_fund')

def get_document_overview(document_id):
    """
    Single query retrieves all entities for a document.
    This is the power of single table design - one query replaces multiple joins.
    """
    response = table.query(
        KeyConditionExpression=Key('PK').eq(document_id)
    )
    
    # Organize results by entity type and position
    entities = {
        'documents': [],
        'capital_activities': [],
        'positions': {}  # Group position-specific items
    }
    
    for item in response['Items']:
        sk = item['SK']
        if sk.startswith('DOCUMENT'):
            entities['documents'].append(item)
        elif sk.startswith('CAPITAL_ACTIVITY'):
            entities['capital_activities'].append(item)
        elif sk.startswith(('CAPITAL_CALL', 'DISTRIBUTION', 'UNFUNDED_COMMITMENT')):
            position_id = item.get('PositionId')
            if position_id not in entities['positions']:
                entities['positions'][position_id] = {
                    'capital_calls': [],
                    'distributions': [],
                    'unfunded_commitments': []
                }
            
            if sk.startswith('CAPITAL_CALL'):
                entities['positions'][position_id]['capital_calls'].append(item)
            elif sk.startswith('DISTRIBUTION'):
                entities['positions'][position_id]['distributions'].append(item)
            elif sk.startswith('UNFUNDED_COMMITMENT'):
                entities['positions'][position_id]['unfunded_commitments'].append(item)
    
    return entities

# Single query gets all related entities
document_data = get_document_overview('DOC001')
print(f"Found {len(document_data['positions'])} positions")
print(f"Position 1 has {len(document_data['positions']['POSITION_1']['capital_calls'])} capital calls")
```

### Query 2: Position-Specific Items within Document

Let's say we want to get all items for a specific position within a document. We can still do this efficiently with a single query, filtering by `PositionId`:

```python
def get_position_items(document_id, position_id):
    """
    Get all items for a specific position using attribute filtering.
    """
    response = table.query(
        KeyConditionExpression=Key('PK').eq(document_id),
        FilterExpression=Key('PositionId').eq(position_id)
    )
    return response['Items']

def get_position_capital_calls(document_id, position_id):
    """
    Get capital calls for a specific position using exact SK match.
    """
    response = table.query(
        KeyConditionExpression=Key('PK').eq(document_id) & 
                              Key('SK').eq(f'CAPITAL_CALL#{position_id}')
    )
    return response['Items']

# Efficient targeted queries
position1_items = get_position_items('DOC001', 'POSITION_1')
position1_calls = get_position_capital_calls('DOC001', 'POSITION_1')
```

### Query 3: Entity Type Filtering with SK Prefixes

Let's say we want to get all capital-related items (capital calls, distributions, commitments) for a document. We can leverage the SK prefix patterns:

```python
def get_all_capital_entities(document_id):
    """
    Query using SK prefix to get all capital-related items.
    """
    response = table.query(
        KeyConditionExpression=Key('PK').eq(document_id) & Key('SK').begins_with('CAPITAL_')
    )
    return response['Items']

def get_latest_document(document_id):
    """
    Get just the latest document version.
    """
    response = table.query(
        KeyConditionExpression=Key('PK').eq(document_id) & Key('SK').eq('DOCUMENT#LATEST')
    )
    return response['Items'][0] if response['Items'] else None

# SK prefix filtering enables efficient entity type queries
capital_items = get_all_capital_entities('DOC001')
latest_doc = get_latest_document('DOC001')
```

## GSI Design: Avoiding Hot Partitions

One of the most critical mistakes in DynamoDB design is creating GSIs that concentrate similar entities in the same partition, leading to hot partition problems. Let's see how to avoid this.

### Problematic GSI Design (Hot Partition Anti-Pattern)

Incorrect GSI designs often use low cardinality partition keys, causing many items to land in the same partition:

```python
# DANGEROUS: All latest documents hit same partition
hot_partition_gsi = [
    {"GSI_PK": "DOCUMENT#LATEST", "GSI_SK": "DOC001"},  # Same partition
    {"GSI_PK": "DOCUMENT#LATEST", "GSI_SK": "DOC002"},  # Same partition  
    {"GSI_PK": "DOCUMENT#LATEST", "GSI_SK": "DOC003"},  # Same partition
    # ... thousands more documents = hot partition problem!
]

# This creates throttling and poor performance under load
```

Instead, we design GSIs with **high cardinality partition keys** that uniquely identify each item or a small group of items.


### Corrected GSI Design: High Cardinality Keys

We create two GSIs to support secondary access patterns without hot partitions:

#### GSI 1: Entity-Document Index (Document-Specific Queries)

GSI 1 allows efficient queries for specific entities within a document, using a composite key that includes the entity type and document ID.

| GSI_PK                        | GSI_SK   | PK     | SK                      | EntityType   | PositionId   |
|-------------------------------|----------|--------|-------------------------|--------------|--------------|
| DOCUMENT#LATEST#DOC001        | METADATA | DOC001 | DOCUMENT#LATEST         | Document     | -            |
| DOCUMENT#LATEST#DOC002        | METADATA | DOC002 | DOCUMENT#LATEST         | Document     | -            |
| CAPITAL_CALL#POSITION_1#DOC001| LATEST   | DOC001 | CAPITAL_CALL#POSITION_1 | CapitalCall  | POSITION_1   |
| CAPITAL_CALL#POSITION_2#DOC001| LATEST   | DOC001 | CAPITAL_CALL#POSITION_2 | CapitalCall  | POSITION_2   |
| DISTRIBUTION#POSITION_1#DOC001| LATEST   | DOC001 | DISTRIBUTION#POSITION_1 | Distribution | POSITION_1   |

#### GSI 2: Position-Document Index (Position-Centric Queries)

GSI 2 allows efficient queries for all data related to a position across documents, using the position ID as the partition key.

| GSI2_PK    | GSI2_SK             | PK     | SK                      | EntityType   | PositionId   |
|------------|---------------------|--------|-------------------------|--------------|--------------|
| POSITION_1 | CAPITAL_CALL#DOC001 | DOC001 | CAPITAL_CALL#POSITION_1 | CapitalCall  | POSITION_1   |
| POSITION_1 | DISTRIBUTION#DOC001 | DOC001 | DISTRIBUTION#POSITION_1 | Distribution | POSITION_1   |
| POSITION_1 | CAPITAL_CALL#DOC002 | DOC002 | CAPITAL_CALL#POSITION_1 | CapitalCall  | POSITION_1   |
| POSITION_2 | CAPITAL_CALL#DOC001 | DOC001 | CAPITAL_CALL#POSITION_2 | CapitalCall  | POSITION_2   |

### GSI Query Patterns

Let's see how to use these GSIs effectively:

```python
def find_specific_document_latest(document_id):
    """
    Direct access to specific document's latest version.
    High cardinality GSI_PK avoids hot partitions.
    """
    response = table.query(
        IndexName='GSI_Entity_Document',
        KeyConditionExpression=Key('GSI_PK').eq(f'DOCUMENT#LATEST#{document_id}')
    )
    return response['Items']

def find_position_capital_calls_in_document(position_id, document_id):
    """
    Find capital calls for specific position in specific document.
    """
    response = table.query(
        IndexName='GSI_Entity_Document', 
        KeyConditionExpression=Key('GSI_PK').eq(f'CAPITAL_CALL#{position_id}#{document_id}')
    )
    return response['Items']

def find_all_position_data_across_documents(position_id):
    """
    Get all data for a position across all documents using GSI2.
    """
    response = table.query(
        IndexName='GSI_Position_Document',
        KeyConditionExpression=Key('GSI2_PK').eq(position_id)
    )
    return response['Items']

# Efficient, scalable queries without hot partitions
doc001_latest = find_specific_document_latest('DOC001')
position1_calls = find_position_capital_calls_in_document('POSITION_1', 'DOC001')
position1_all_data = find_all_position_data_across_documents('POSITION_1')
```

## Advanced Query Examples

In addition to the core patterns, here are some advanced queries enabled by this design.

### Multi-Position Analysis within Document

Here's how to analyze all positions within a document in one query:

```python
def analyze_document_positions(document_id):
    """
    Comprehensive analysis of all positions within a document.
    Single query retrieves all data, then processes by position.
    """
    response = table.query(
        KeyConditionExpression=Key('PK').eq(document_id)
    )
    
    position_analysis = {}
    
    for item in response['Items']:
        position_id = item.get('PositionId')
        if position_id:  # Only position-specific items
            if position_id not in position_analysis:
                position_analysis[position_id] = {
                    'capital_calls': 0,
                    'total_called': 0,
                    'distributions': 0,
                    'total_distributed': 0,
                    'unfunded_commitments': 0,
                    'total_unfunded': 0
                }
            
            sk = item['SK']
            amount = item.get('Amount', 0)
            
            if sk.startswith('CAPITAL_CALL'):
                position_analysis[position_id]['capital_calls'] += 1
                position_analysis[position_id]['total_called'] += amount
            elif sk.startswith('DISTRIBUTION'):
                position_analysis[position_id]['distributions'] += 1
                position_analysis[position_id]['total_distributed'] += amount
            elif sk.startswith('UNFUNDED_COMMITMENT'):
                position_analysis[position_id]['unfunded_commitments'] += 1
                position_analysis[position_id]['total_unfunded'] += amount
    
    return position_analysis

# One query analyzes all positions in document
doc001_analysis = analyze_document_positions('DOC001')
print(f"POSITION_1 total called: ${doc001_analysis['POSITION_1']['total_called']:,}")
print(f"POSITION_2 total distributed: ${doc001_analysis['POSITION_2']['total_distributed']:,}")
```

### Historical Data Queries

It is often necessary to analyze historical versions of documents. Here's how to do that efficiently:

```python
def get_historical_documents(document_id, start_date, end_date):
    """
    Query historical documents within a date range using SK filtering.
    """
    response = table.query(
        KeyConditionExpression=Key('PK').eq(document_id) & 
                              Key('SK').between(f'DOCUMENT#{start_date}', f'DOCUMENT#{end_date}')
    )
    return response['Items']

def compare_document_versions(document_id):
    """
    Compare latest vs historical document versions.
    """
    response = table.query(
        KeyConditionExpression=Key('PK').eq(document_id) & 
                              Key('SK').begins_with('DOCUMENT#')
    )
    
    latest_doc = None
    historical_docs = []
    
    for item in response['Items']:
        if item['Version'] == 'Latest':
            latest_doc = item
        else:
            historical_docs.append(item)
    
    return {
        'latest': latest_doc,
        'historical': sorted(historical_docs, key=lambda x: x['SK'])
    }

# Historical analysis with efficient SK filtering
historical_docs = get_historical_documents('DOC001', '2025-01-01', '2025-06-30')
doc_versions = compare_document_versions('DOC001')
```

## Anti-Patterns: What Breaks Single Table Design

In contrast, here are some common anti-patterns that break the principles of single table design.

### Anti-Pattern 1: Different Partition Keys for Related Entities

Instead of using the document ID as the partition key for all related entities, some designs use different PKs for each entity type. This breaks co-location and forces multiple queries or expensive scans:

```python
# WRONG: Different PK for each entity type breaks co-location
broken_structure = [
    {"PK": "DOCUMENT#DOC001", "SK": "METADATA", "EntityType": "Document"},
    {"PK": "CAPITAL_CALL#CC001", "SK": "DOC001", "EntityType": "CapitalCall"},  # Different PK!
    {"PK": "DISTRIBUTION#DIST001", "SK": "DOC001", "EntityType": "Distribution"}  # Different PK!
]

def get_document_overview_broken(document_id):
    """
    BROKEN: Requires multiple queries and expensive scans.
    """
    # Query 1: Get document
    doc_response = table.query(
        KeyConditionExpression=Key('PK').eq(f'DOCUMENT#{document_id}')
    )
    
    # Query 2: Expensive scan for capital calls
    capital_calls = table.scan(
        FilterExpression=Attr('SK').eq(document_id) & Attr('EntityType').eq('CapitalCall')
    )['Items']
    
    # Query 3: Expensive scan for distributions  
    distributions = table.scan(
        FilterExpression=Attr('SK').eq(document_id) & Attr('EntityType').eq('Distribution')
    )['Items']
    
    return doc_response['Items'], capital_calls, distributions

# Result: Multiple queries + expensive scans = poor performance
```

### Anti-Pattern 2: Hot Partition GSIs

Hot partitions occur when a GSI uses a low cardinality partition key, causing many items to be concentrated in the same partition. This leads to throttling and poor performance under load:

```python
# WRONG: Low cardinality GSI partition keys
bad_gsi_patterns = [
    "DOCUMENT#LATEST",     # All latest docs → same partition
    "STATUS#ACTIVE",       # All active items → same partition  
    "TYPE#CAPITAL_CALL"    # All capital calls → same partition
]

# These create hot partitions under load, causing throttling
```

### The Alternative Approach: Position-Centric Design

Some developers suggest using position-centric partition keys:

```python
# Alternative approach - position-centric partitioning
position_centric_structure = [
    {"PK": "DOCUMENT#DOC001", "SK": "METADATA", "EntityType": "Document"},
    {"PK": "CAPITAL_CALL#POSITION_1", "SK": "DOC001", "EntityType": "CapitalCall"},
    {"PK": "CAPITAL_CALL#POSITION_1", "SK": "DOC002", "EntityType": "CapitalCall"},
    {"PK": "DISTRIBUTION#POSITION_1", "SK": "DOC001", "EntityType": "Distribution"}
]
```

**Why This Still Breaks Single Table Design for Our Use Case:**

While this approach avoids expensive scans, it violates our **primary access pattern** (complete document overview):

```python
def get_document_overview_position_centric(document_id):
    """
    Position-centric approach still requires multiple queries for document overview.
    """
    # Query 1: Get document metadata
    doc_response = table.query(
        KeyConditionExpression=Key('PK').eq(f'DOCUMENT#{document_id}')
    )
    
    # Query 2-N: One query per position per entity type
    capital_calls_pos1 = table.query(
        KeyConditionExpression=Key('PK').eq('CAPITAL_CALL#POSITION_1') & 
                              Key('SK').eq(document_id)
    )
    capital_calls_pos2 = table.query(
        KeyConditionExpression=Key('PK').eq('CAPITAL_CALL#POSITION_2') & 
                              Key('SK').eq(document_id)
    )
    # ... more queries for each position and entity type
    
    # Result: 5+ queries vs 1 query in proper single table design
    return "Multiple efficient queries, but breaks single table principle"
```

**Key Insight:** Access patterns determine the correct design. Position-centric partitioning would be optimal if the primary access pattern were "get all data for a position across documents," but it fails for "get complete document overview."

## Performance Comparison

| Approach | Document Overview | Position Across Docs | Scan Operations | Query Count | Scalability |
|----------|-------------------|---------------------|-----------------|-------------|-------------|
| **Proper Single Table** | 1 query (~5ms) | GSI query | None | 1 | High |
| **Position-Centric** | 5+ queries (~25ms) | 1 query (~5ms) | None | 5+ | Medium |
| **Broken Pattern** | Multiple scans (~100ms+) | Scan operations | Yes | 2+ scans | Poor |
| **Hot Partition GSI** | 1 query (variable) | 1 query (variable) | None | 1 | Poor |


## Design Decision Framework

When designing your single table, follow these structured steps:

### Step 1: Analyze Access Patterns

Start by listing and prioritizing your application's access patterns. This ensures your key design supports the most common and performance-critical queries.

**Primary Access Patterns (optimize your main table for these):**
- Get complete document overview (all entities for all positions)
- Get a specific entity within a document
- Get historical versions of documents

**Secondary Access Patterns (optimize with GSIs):**
- Get all data for a position across documents
- Find a specific entity type across documents
- Perform cross-document analytics

---

### Step 2: Choose Partition Key Strategy

Select a partition key (PK) that aligns with your primary access pattern:

- **Document-centric (recommended for this use case):**
    - Use the document ID as the PK to group all related entities for efficient document-centric queries.
- **Position-centric (alternative):**
    - Use the position ID as the PK to group all data for a position across documents (best if your main queries are position-based).
- **Entity-centric (not recommended):**
    - Using entity type as PK often leads to hot partitions and should be avoided.

---

### Step 3: Design Sort Key Patterns

Define sort key (SK) patterns that enable efficient filtering and retrieval:

- `DOCUMENT#VERSION` &mdash; for document-level entities and versioning
- `ENTITY_TYPE#POSITION_ID` &mdash; for position-specific entities
- `ENTITY_TYPE#TIMESTAMP` &mdash; for time-based versioning or ordering
- `PARENT#CHILD#IDENTIFIER` &mdash; for hierarchical relationships

---

### Step 4: GSI Strategy

Design GSIs with high-cardinality partition keys to avoid hot partitions and support secondary access patterns:

- `ENTITY_TYPE#POSITION_ID#DOCUMENT_ID` &mdash; unique combination for targeted queries
- `POSITION_ID` &mdash; enables efficient queries across all documents for a position
- `DOCUMENT_ID#ENTITY_TYPE` &mdash; unique per document and entity type
- `TIME_SHARD#ENTITY_TYPE` &mdash; distributes load by time for analytics or reporting

---

## Real-World Implementation Considerations

### Scaling and Performance
- **Single query retrieval** for primary access patterns provides consistent ~5ms response times
- **High cardinality GSIs** ensure even load distribution across partitions
- **Selective attribute usage** (PositionId only where needed) maintains data clarity
- **Efficient filtering** through SK prefix patterns reduces network overhead

### Operational Benefits
- **Reduced complexity** - one table vs multiple tables with complex joins
- **Lower latency** - single network call for related data retrieval
- **Cost efficiency** - fewer read operations and no cross-table joins
- **Easier monitoring** - single table to monitor for performance metrics

### Future Challenges
While single table design offers powerful benefits, be aware of these challenges:

- **Evolving Access Patterns:** If your application's queries change significantly, your key design may become suboptimal, requiring costly migrations.
- **Complexity:** Modeling all entities in one table increases cognitive load and can make onboarding new developers harder.
- **Item Size Limits:** DynamoDB has a 400KB item size limit; denormalizing too much data can hit this ceiling.
- **GSI Management:** As your application grows, you may need to add or redesign GSIs, which can be operationally complex.
- **Monitoring Hot Partitions:** Even with careful design, unexpected data skew can cause hot partitions, requiring ongoing monitoring and adjustment.
- **Testing and Validation:** More complex key patterns require thorough testing to ensure all access patterns are efficient and correct.

## Conclusion

Single table design succeeds when you **strategically group related entities under shared partition keys** and **design sort keys that match your query patterns**, while carefully avoiding hot partitions in GSIs. The investment fund management example demonstrates several critical principles:

### Core Success Factors
1. **Access pattern-driven design** - Document ID as PK optimizes for document-centric queries
2. **Strategic entity grouping** - Related entities share partition keys for co-location
3. **Meaningful sort key patterns** - Prefixed SKs enable efficient filtering and retrieval
4. **High-cardinality GSIs** - Unique partition keys prevent hot partition problems
5. **Selective attribute usage** - PositionId only where semantically appropriate

### Performance Benefits
- **Single-digit millisecond response times** for complex queries
- **Elimination of expensive scan operations** through proper key design
- **Scalable GSI performance** through high cardinality partition keys
- **Consistent performance** regardless of data volume growth

### Critical Pitfalls to Avoid
- **Different partition keys for related data** - breaks co-location benefits
- **Hot partition GSIs** - creates throttling and poor performance under load
- **Ignoring primary access patterns** - leads to suboptimal key design
- **Overcomplicating sort key patterns** - should match actual query needs

The key insight is that **there's no universally correct single table design** - the optimal structure depends entirely on your specific access patterns. Document-centric partitioning works for our investment fund example because we primarily access complete document overviews, but position-centric partitioning might be better for different access patterns.

Remember: **Single table design is about optimizing for your primary access patterns while using GSIs strategically for secondary patterns**. When you follow this principle and avoid common pitfalls like hot partitions, single table design becomes a powerful tool for building high-performance, cost-effective DynamoDB applications that scale efficiently under load.

The investment fund example shows that even complex financial relationships—with multiple entity types, historical versioning, and position-specific data—can be elegantly modeled using single table design when you understand and apply these fundamental principles correctly.

Happy modeling! 🚀

### Download the Report

For a deeper dive into the rationale behind DynamoDB's single table design pattern, download the comprehensive PDF report: [Download DynamoDB_Single_Table_Design.pdf]({{page.attachment_path}}/DynamoDB_Single_Table_Design.pdf)
