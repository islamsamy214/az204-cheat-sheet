# Azure Cosmos DB Supported APIs

## Key Concepts
- **Multi-model database** - One service, multiple APIs
- **Wire protocol compatibility** - Use existing tools and drivers
- **API selection** - Choose at account creation (cannot change)
- **Native vs Compatible** - NoSQL native, others wire-compatible

## What are Cosmos DB APIs?

**Multiple interfaces to same data**:

- **One service** - Azure Cosmos DB (globally distributed, low latency)
- **Multiple APIs** - Choose based on app needs and skills
- **Same benefits** - All APIs get SLAs, global distribution, auto-scaling

### Available APIs

| API | Data Model | Use Case |
|-----|------------|----------|
| **NoSQL** | Document (JSON) | New modern apps |
| **MongoDB** | Document (BSON) | Existing MongoDB apps |
| **PostgreSQL** | Relational (Citus) | Distributed PostgreSQL |
| **Cassandra** | Column-family | Wide-column, high scale |
| **Gremlin** | Graph | Complex relationships |
| **Table** | Key-value | Azure Table Storage migration |

### API Selection

**Choose at account creation**:

```bash
# API for NoSQL (default)
az cosmosdb create \
  --name mycosmosaccount \
  --resource-group myResourceGroup \
  --kind GlobalDocumentDB  # NoSQL API

# API for MongoDB
az cosmosdb create \
  --name mycosmosaccount \
  --resource-group myResourceGroup \
  --kind MongoDB

# API for Cassandra
az cosmosdb create \
  --name mycosmosaccount \
  --resource-group myResourceGroup \
  --capabilities EnableCassandra

# API for Gremlin
az cosmosdb create \
  --name mycosmosaccount \
  --resource-group myResourceGroup \
  --capabilities EnableGremlin

# API for Table
az cosmosdb create \
  --name mycosmosaccount \
  --resource-group myResourceGroup \
  --capabilities EnableTable
```

⚠️ **Cannot change API after creation** - Choose carefully!

## API Comparison Matrix

### Feature Comparison

| Feature | NoSQL | MongoDB | PostgreSQL | Cassandra | Gremlin | Table |
|---------|-------|---------|-----------|-----------|---------|-------|
| **Native to Cosmos DB** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Wire protocol compatible** | N/A | ✅ | ✅ | ✅ | ✅ | ✅ |
| **First-class features** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Global distribution** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Multi-region writes** | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ |
| **Serverless** | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ |
| **Autoscale** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

### When to Use Each API

```
Decision Tree:

New application?
├── Modern JSON-based → NoSQL (best choice)
└── Existing app migration ↓

What database are you using?
├── MongoDB → API for MongoDB
├── Cassandra → API for Cassandra
├── PostgreSQL → API for PostgreSQL
├── Azure Table Storage → API for Table
└── Need graph queries → API for Gremlin
```

## API for NoSQL

### Characteristics

**Native to Azure Cosmos DB**:

- ✅ Best end-to-end experience
- ✅ First to get new features
- ✅ Full control over interface and SDKs
- ✅ SQL-like query syntax
- ✅ JSON document storage

### Document Format

```json
{
  "id": "product-1",
  "category": "electronics",
  "name": "Laptop",
  "price": 999.99,
  "specs": {
    "cpu": "Intel i7",
    "ram": "16GB",
    "storage": "512GB SSD"
  },
  "tags": ["laptop", "computer", "portable"]
}
```

### Query Language

**SQL-like syntax**:

```sql
-- Select all
SELECT * FROM products p

-- Filter
SELECT * FROM products p 
WHERE p.category = 'electronics' AND p.price < 1000

-- Project specific fields
SELECT p.name, p.price FROM products p

-- Join arrays
SELECT p.name, tag
FROM products p
JOIN tag IN p.tags

-- Aggregations
SELECT p.category, COUNT(1) AS count, AVG(p.price) AS avgPrice
FROM products p
GROUP BY p.category
```

### SDK Support

```csharp
// .NET SDK
using Microsoft.Azure.Cosmos;

var client = new CosmosClient(endpoint, key);
var container = client.GetContainer("mydb", "products");

// Create
await container.CreateItemAsync(product);

// Read
var response = await container.ReadItemAsync<Product>(
    "product-1",
    new PartitionKey("electronics")
);

// Query
var query = container.GetItemQueryIterator<Product>(
    "SELECT * FROM c WHERE c.category = 'electronics'"
);
```

### When to Use

✅ **New applications** - Modern, cloud-native apps
✅ **JSON data** - Natural document model
✅ **Flexible schema** - Schema-less design
✅ **Rich queries** - SQL-like capabilities
✅ **Latest features** - Always first to get updates

❌ **Don't use if** - Existing MongoDB/Cassandra app (use compatible API)

## API for MongoDB

### Characteristics

**MongoDB wire protocol compatible**:

- ✅ Use existing MongoDB tools, drivers, and skills
- ✅ No code changes to connection logic
- ✅ BSON format (Binary JSON)
- ✅ MongoDB query language
- ⚠️ Compatible, not identical (some features differ)

### Version Support

| Version | Features |
|---------|----------|
| **6.0** | Latest features, best performance |
| **5.0** | Time-series collections, native aggregation |
| **4.2** | Distributed transactions |
| **4.0** | Multi-document ACID transactions |
| **3.6** | Change streams, aggregation improvements |

### Connection String

```javascript
// MongoDB connection (minimal code change)
const { MongoClient } = require('mongodb');

const connectionString = "mongodb://mycosmosaccount:key@mycosmosaccount.mongo.cosmos.azure.com:10255/?ssl=true&replicaSet=globaldb";

const client = new MongoClient(connectionString);
await client.connect();

const db = client.db('mydb');
const collection = db.collection('products');

// Standard MongoDB operations
await collection.insertOne({ name: "Laptop", price: 999.99 });
const product = await collection.findOne({ name: "Laptop" });
```

### Compatibility

**Supported operations**:
- ✅ CRUD operations (insert, find, update, delete)
- ✅ Aggregation pipeline
- ✅ Indexes (single-field, compound, geospatial)
- ✅ Change streams
- ✅ Transactions (4.0+)

**Differences from native MongoDB**:
- ⚠️ Different performance characteristics
- ⚠️ RU-based billing (not CPU/memory)
- ⚠️ Some features may have limitations

### When to Use

✅ **Existing MongoDB app** - Migrate without rewrite
✅ **MongoDB expertise** - Use existing skills
✅ **MongoDB ecosystem** - Tools and drivers
✅ **BSON format** - Binary JSON benefits

❌ **Don't use if** - New app with no MongoDB experience (use NoSQL API)

### Migration Benefit

```
Before (Hosted MongoDB):
├── Manual scaling
├── Manual replication
├── Manual failover
├── Regional only
└── Manual backups

After (Cosmos DB with MongoDB API):
├── Automatic scaling ✅
├── Turnkey global distribution ✅
├── Automatic failover ✅
├── Multi-region ✅
└── Automatic backups ✅

Code changes: Minimal (just connection string)
```

## API for PostgreSQL

### Characteristics

**Distributed PostgreSQL with Citus**:

- ✅ PostgreSQL wire protocol compatible
- ✅ Citus extension for horizontal scaling
- ✅ Relational model with scale-out
- ✅ Standard PostgreSQL tools and drivers
- ✅ Single-node or multi-node configurations

### Architecture Options

#### Single Node
```
Use case: Small databases (<100 GB)
Benefits: Simple, PostgreSQL-compatible
Limitations: No distribution
```

#### Multi-Node (Citus)
```
Use case: Large databases (>100 GB)
Architecture:
  Coordinator Node
       ↓
  Worker Node 1 | Worker Node 2 | Worker Node n
       ↓
  Distributed tables across workers
  Automatic query routing
  Parallel query execution
```

### Connection

```python
import psycopg2

# Standard PostgreSQL connection
conn = psycopg2.connect(
    host="c-mycosmospostgres.12345.postgres.cosmos.azure.com",
    database="mydb",
    user="citus",
    password="password",
    port=5432
)

cursor = conn.cursor()
cursor.execute("SELECT * FROM products WHERE category = 'electronics'")
rows = cursor.fetchall()
```

### When to Use

✅ **Relational workloads** - Need SQL, joins, constraints
✅ **PostgreSQL apps** - Existing PostgreSQL applications
✅ **Large datasets** - Multi-terabyte databases
✅ **Multi-tenant** - Distribute by tenant_id

❌ **Don't use if** - Pure document/NoSQL model (use NoSQL API)

## API for Apache Cassandra

### Characteristics

**Cassandra wire protocol compatible**:

- ✅ CQL (Cassandra Query Language)
- ✅ Column-family data model
- ✅ Wide-column storage
- ✅ High write throughput
- ✅ Use existing Cassandra drivers

### Data Model

```cql
-- Keyspace (like database)
CREATE KEYSPACE store WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'datacenter1': 3
};

-- Table (column-family)
CREATE TABLE products (
  category text,
  product_id uuid,
  name text,
  price decimal,
  PRIMARY KEY (category, product_id)
);

-- Insert
INSERT INTO products (category, product_id, name, price)
VALUES ('electronics', uuid(), 'Laptop', 999.99);

-- Query
SELECT * FROM products WHERE category = 'electronics';
```

### Key Features

**Wide-column model**:
```
Row key: category=electronics
├── product_id=uuid1 → name=Laptop, price=999.99
├── product_id=uuid2 → name=Phone, price=699.99
└── product_id=uuid3 → name=Tablet, price=499.99

Efficient for time-series, IoT, high write throughput
```

### When to Use

✅ **High write throughput** - IoT, logging, time-series
✅ **Existing Cassandra apps** - Migrate without rewrite
✅ **Wide-column model** - Flexible column schema
✅ **Cassandra expertise** - Use existing skills

❌ **Don't use if** - Document model better fit (use NoSQL/MongoDB API)

## API for Apache Gremlin

### Characteristics

**Graph database API**:

- ✅ Apache TinkerPop Gremlin
- ✅ Vertices and edges
- ✅ Complex relationship queries
- ✅ Graph traversals
- ✅ Property graph model

### Graph Model

```
Vertices (Nodes):
├── Person: {id: "john", name: "John", age: 30}
├── Person: {id: "jane", name: "Jane", age: 28}
└── Product: {id: "laptop", name: "Laptop", price: 999.99}

Edges (Relationships):
├── john -[knows]-> jane
├── john -[purchased]-> laptop
└── jane -[likes]-> laptop
```

### Gremlin Queries

```groovy
// Add vertex
g.addV('person')
 .property('id', 'john')
 .property('name', 'John')
 .property('age', 30)

// Add edge
g.V('john').addE('knows').to(g.V('jane'))

// Traversal queries
// Find John's friends
g.V('john').out('knows').values('name')

// Find products liked by John's friends
g.V('john').out('knows').out('likes').hasLabel('product')

// Find common interests
g.V('john').out('knows').where(out('likes').hasId('laptop'))
```

### When to Use

✅ **Social networks** - Friends, connections, recommendations
✅ **Fraud detection** - Transaction patterns, relationships
✅ **Recommendation engines** - Product suggestions based on graphs
✅ **Knowledge graphs** - Entity relationships
✅ **Network topology** - Infrastructure, routing

❌ **Don't use if** - Simple key-value or document model (use NoSQL API)

## API for Table

### Characteristics

**Azure Table Storage compatible**:

- ✅ Key-value store
- ✅ PartitionKey + RowKey model
- ✅ Drop-in replacement for Azure Table Storage
- ✅ Better performance and global distribution
- ✅ Backward compatible

### Data Model

```csharp
public class ProductEntity : ITableEntity
{
    public string PartitionKey { get; set; }  // category
    public string RowKey { get; set; }        // product-id
    public string Name { get; set; }
    public double Price { get; set; }
    public DateTimeOffset? Timestamp { get; set; }
    public ETag ETag { get; set; }
}

// Insert
var entity = new ProductEntity
{
    PartitionKey = "electronics",
    RowKey = "product-1",
    Name = "Laptop",
    Price = 999.99
};
await tableClient.AddEntityAsync(entity);

// Query
var entities = tableClient.Query<ProductEntity>(
    e => e.PartitionKey == "electronics"
);
```

### Migration from Azure Table Storage

```
Before (Azure Table Storage):
├── Single region
├── Limited throughput
├── No global distribution
├── Basic SLAs

After (Cosmos DB Table API):
├── Multi-region ✅
├── Unlimited throughput ✅
├── Global distribution ✅
├── 99.999% SLA ✅

Migration: Change connection string only!
```

### When to Use

✅ **Azure Table Storage migration** - Upgrade with minimal changes
✅ **Simple key-value** - PartitionKey + RowKey access patterns
✅ **OLTP workloads** - Transactional processing
✅ **Existing Table Storage apps** - Drop-in replacement

❌ **Don't use if** - Need complex queries (use NoSQL API)

## Choosing the Right API

### Decision Matrix

| Current State | Recommended API | Reason |
|---------------|-----------------|--------|
| **New app, JSON data** | NoSQL | Best features, native experience |
| **Existing MongoDB** | MongoDB | Minimal migration, wire compatible |
| **Existing Cassandra** | Cassandra | CQL compatible, wide-column model |
| **Existing PostgreSQL** | PostgreSQL | Relational + scale-out with Citus |
| **Graph relationships** | Gremlin | Purpose-built for graph traversals |
| **Azure Table Storage** | Table | Drop-in replacement, better SLAs |

### Considerations

**Technical factors**:
- ✅ Existing codebase (wire protocol compatibility)
- ✅ Team expertise (use familiar tools)
- ✅ Data model (document, graph, wide-column, relational, key-value)
- ✅ Query patterns (simple reads vs complex joins/traversals)

**Business factors**:
- ✅ Migration cost (rewrite vs minimal changes)
- ✅ Time to market (faster with compatible API)
- ✅ Feature requirements (check API-specific limitations)

## API Limitations

### Common Across All APIs

✅ **Available in all APIs**:
- Global distribution
- Automatic scaling
- Low latency (<10ms)
- High availability (99.999%)
- Encryption at rest and in transit

⚠️ **NoSQL API exclusive features**:
- Serverless (all except PostgreSQL)
- All new features first
- Richest SDK support

### API-Specific Limitations

| API | Notable Limitations |
|-----|---------------------|
| **MongoDB** | Some operations differ from native MongoDB |
| **PostgreSQL** | No serverless mode, multi-node required for scale |
| **Cassandra** | No serverless mode |
| **Gremlin** | Limited to certain Gremlin steps |
| **Table** | Limited query capabilities vs NoSQL API |

## Critical Notes
- 💡 **NoSQL API** - Native, best features, first to get updates
- 🎯 **Wire protocol** - MongoDB, Cassandra, Gremlin, Table compatible
- ✅ **Choose at creation** - Cannot change API after account creation
- ⚠️ **Migration** - Compatible APIs minimize code changes
- 🔄 **All APIs** - Same global distribution, SLAs, scaling benefits
- 📊 **Data models** - Document, column-family, graph, key-value, relational
- 💡 **NoSQL for new apps** - Best choice for greenfield projects
- ✅ **Compatible APIs** - Use existing tools, drivers, and skills
- 🔒 **Serverless** - NoSQL, MongoDB, Gremlin, Table (not PostgreSQL, Cassandra)

## Exam Tips
- APIs available: NoSQL, MongoDB, PostgreSQL, Cassandra, Gremlin, Table
- NoSQL API: Native to Cosmos DB, best features, SQL-like queries, JSON documents
- MongoDB API: Wire protocol compatible, BSON format, existing MongoDB apps
- PostgreSQL API: Relational with Citus, distributed PostgreSQL, multi-node scale
- Cassandra API: CQL, column-family model, high write throughput
- Gremlin API: Graph database, vertices and edges, relationship queries
- Table API: Azure Table Storage compatible, PartitionKey + RowKey
- Cannot change API: Choose at account creation, cannot switch later
- NoSQL use case: New modern applications, JSON data, flexible schema
- MongoDB use case: Existing MongoDB apps, BSON format, minimal migration
- Cassandra use case: IoT, time-series, high write throughput
- Gremlin use case: Social networks, fraud detection, recommendations
- Table use case: Azure Table Storage migration, simple key-value
- PostgreSQL use case: Relational workloads, multi-tenant, large datasets
- Wire protocol: MongoDB, Cassandra, Gremlin, Table (compatible with existing tools)
- All APIs: Global distribution, auto-scaling, high availability, encryption
- Serverless support: NoSQL, MongoDB, Gremlin, Table (not PostgreSQL, Cassandra)
- NoSQL first: New features released to NoSQL API first

[Learn More](https://learn.microsoft.com/en-us/training/modules/explore-azure-cosmos-db/6-cosmos-db-supported-apis)
