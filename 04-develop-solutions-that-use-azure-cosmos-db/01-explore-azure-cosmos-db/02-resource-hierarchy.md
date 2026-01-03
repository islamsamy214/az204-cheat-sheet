# Azure Cosmos DB Resource Hierarchy

## Key Concepts
- **Account** - Top-level unit with unique DNS name
- **Database** - Namespace for containers
- **Container** - Scalability unit with unlimited throughput/storage
- **Items** - Individual data entities (documents, rows, nodes)
- **Partition key** - Critical for distribution and scale

## Resource Hierarchy

### Visual Structure
```
Azure Subscription
     ↓
Azure Cosmos DB Account (myaccount.documents.azure.com)
     ↓
Database (logical namespace)
     ├── Container 1 (unlimited RU/s & storage)
     │   ├── Partition Key: /category
     │   ├── Item 1 (document/row/node)
     │   ├── Item 2
     │   └── Item n
     ├── Container 2
     └── Container n
```

### Hierarchy Levels

| Level | Description | Purpose | Limit |
|-------|-------------|---------|-------|
| **Subscription** | Azure billing unit | Organization | Many accounts |
| **Account** | Global distribution unit | DNS, regions | 50 per subscription* |
| **Database** | Namespace | Logical grouping | Unlimited |
| **Container** | Scalability unit | Data storage | Unlimited |
| **Item** | Data entity | Actual data | Unlimited |

*Can be increased via support request

## Azure Cosmos DB Account

### What is an Account?

**Top-level entity** for global distribution:

- **Unique DNS name** - `https://<account-name>.documents.azure.com`
- **Global distribution** - Manage multiple regions
- **High availability** - Configure failover
- **Consistency settings** - Account-level default
- **API selection** - Chosen at account creation

### Account Properties

```json
{
  "name": "mycosmosaccount",
  "location": "East US",
  "kind": "GlobalDocumentDB",
  "databaseAccountOfferType": "Standard",
  "locations": [
    {
      "locationName": "East US",
      "failoverPriority": 0
    },
    {
      "locationName": "West US",
      "failoverPriority": 1
    }
  ],
  "consistencyPolicy": {
    "defaultConsistencyLevel": "Session"
  }
}
```

### Create Account

```bash
# Create Cosmos DB account
az cosmosdb create \
  --name mycosmosaccount \
  --resource-group myResourceGroup \
  --locations regionName=eastus failoverPriority=0 \
  --locations regionName=westus failoverPriority=1 \
  --default-consistency-level Session \
  --enable-multiple-write-locations true

# Account limits per subscription
Default: 50 accounts
Can increase: Submit support request
```

### Account Management Tools

| Tool | Use Case |
|------|----------|
| **Azure Portal** | GUI management |
| **Azure CLI** | Command-line automation |
| **PowerShell** | Windows scripting |
| **.NET SDK** | Programmatic management |
| **Java SDK** | Programmatic management |
| **Python SDK** | Programmatic management |

## Azure Cosmos DB Database

### What is a Database?

**Logical namespace** for containers:

- **Container grouping** - Organize related containers
- **Throughput sharing** - Optional shared RU/s across containers
- **Management unit** - Logical boundary for operations
- **No schema enforcement** - Schema-less

### Database Characteristics

✅ **Unlimited databases** per account
✅ **Unlimited containers** per database
✅ **No data stored** at database level (only in containers)
✅ **Optional shared throughput** across containers

### Database vs Container Throughput

#### Shared Throughput (Database-level)
```
Database: 1,000 RU/s (shared)
├── Container 1 (shares 1,000 RU/s)
├── Container 2 (shares 1,000 RU/s)
└── Container 3 (shares 1,000 RU/s)

Total cost: 1,000 RU/s
Use case: Many small containers, variable traffic
```

#### Dedicated Throughput (Container-level)
```
Database (no throughput)
├── Container 1: 400 RU/s (dedicated)
├── Container 2: 1,000 RU/s (dedicated)
└── Container 3: 10,000 RU/s (dedicated)

Total cost: 11,400 RU/s
Use case: Predictable per-container needs
```

### Create Database

```bash
# Create database
az cosmosdb sql database create \
  --account-name mycosmosaccount \
  --resource-group myResourceGroup \
  --name mydb

# Create database with shared throughput
az cosmosdb sql database create \
  --account-name mycosmosaccount \
  --resource-group myResourceGroup \
  --name mydb \
  --throughput 1000

# Create database with autoscale
az cosmosdb sql database create \
  --account-name mycosmosaccount \
  --resource-group myResourceGroup \
  --name mydb \
  --max-throughput 4000
```

## Azure Cosmos DB Container

### What is a Container?

**Fundamental unit of scalability**:

- **Unlimited throughput** - Provision RU/s
- **Unlimited storage** - Automatic partitioning
- **Partition key** - Required for distribution
- **Indexing policy** - Customize indexing
- **TTL (Time-to-Live)** - Auto-expire items

### Container Characteristics

✅ **Schema-agnostic** - No predefined schema
✅ **Automatic partitioning** - Based on partition key
✅ **Independent scaling** - Per-container RU/s and storage
✅ **API-specific naming** - Collection (MongoDB), Table (Table API), Graph (Gremlin)

### Container Naming by API

| API | Container Name |
|-----|----------------|
| **NoSQL** | Container |
| **MongoDB** | Collection |
| **Cassandra** | Table |
| **Gremlin** | Graph |
| **Table** | Table |

### Partition Key

**Critical decision** - Cannot be changed after creation:

```json
// Container with partition key
{
  "id": "products",
  "partitionKey": {
    "paths": ["/category"],
    "kind": "Hash"
  }
}

// Items distributed by category
{
  "id": "product-1",
  "category": "electronics",  // ← Partition key value
  "name": "Laptop"
}
```

**Partition key best practices**:
- ✅ High cardinality (many unique values)
- ✅ Even distribution (balanced storage)
- ✅ Query pattern friendly (filters use partition key)
- ❌ Avoid hot partitions (uneven load)

### Physical vs Logical Partitions

#### Physical Partition
**Azure Cosmos DB managed**:

- **Fixed size** - Up to 50 GB storage
- **Fixed throughput** - Up to 10,000 RU/s
- **Automatic creation** - As data grows
- **Transparent** - Not exposed to users

#### Logical Partition
**User-defined by partition key**:

- **Max size** - 20 GB per unique partition key value
- **Items grouped** - Same partition key value together
- **Query efficiency** - Queries within partition are faster
- **Transaction scope** - Transactions limited to single partition

```
Partition Key: /category
     ↓
Logical Partitions:
├── category=electronics (15 GB) → Physical Partition 1
├── category=books (8 GB) → Physical Partition 2
├── category=clothing (18 GB) → Physical Partition 3
└── category=toys (5 GB) → Physical Partition 4
```

### Throughput Provisioning Modes

#### 1. Dedicated Throughput (Container-level)
```bash
# Create container with dedicated RU/s
az cosmosdb sql container create \
  --account-name mycosmosaccount \
  --database-name mydb \
  --name mycontainer \
  --partition-key-path "/category" \
  --throughput 1000

# Exclusively for this container
# More expensive but guaranteed
```

**Types**:
- **Standard (manual)** - Fixed RU/s
- **Autoscale** - Auto-scale between min (10% of max) and max

#### 2. Shared Throughput (Database-level)
```bash
# Create container sharing database throughput
az cosmosdb sql container create \
  --account-name mycosmosaccount \
  --database-name mydb \
  --name mycontainer \
  --partition-key-path "/category"

# Shares RU/s with up to 25 containers
# More cost-effective
# Excludes containers with dedicated throughput
```

**Limitations**:
- ⚠️ Max 25 containers can share
- ⚠️ Containers with dedicated throughput don't share
- ⚠️ Database must be created with shared throughput

### Create Container

```bash
# Basic container
az cosmosdb sql container create \
  --account-name mycosmosaccount \
  --database-name mydb \
  --name mycontainer \
  --partition-key-path "/id" \
  --throughput 400

# Container with autoscale
az cosmosdb sql container create \
  --account-name mycosmosaccount \
  --database-name mydb \
  --name mycontainer \
  --partition-key-path "/category" \
  --max-throughput 4000

# Container with TTL
az cosmosdb sql container create \
  --account-name mycosmosaccount \
  --database-name mydb \
  --name mycontainer \
  --partition-key-path "/deviceId" \
  --throughput 1000 \
  --ttl 3600  # Expire after 1 hour
```

### Container Settings

| Setting | Purpose | Changeable |
|---------|---------|------------|
| **Partition key** | Data distribution | ❌ No |
| **Throughput** | RU/s provisioning | ✅ Yes |
| **Indexing policy** | Index configuration | ✅ Yes |
| **TTL** | Auto-expiration | ✅ Yes |
| **Unique keys** | Enforce uniqueness | ❌ No |
| **Conflict resolution** | Multi-write conflicts | ❌ No |

## Azure Cosmos DB Items

### What are Items?

**Individual data entities** in containers:

- **JSON documents** (NoSQL API)
- **BSON documents** (MongoDB API)
- **Rows** (Cassandra API)
- **Vertices/Edges** (Gremlin API)
- **Entities** (Table API)

### Item Representation by API

| API | Item Type | Example |
|-----|-----------|---------|
| **NoSQL** | JSON document | `{"id": "1", "name": "John"}` |
| **MongoDB** | BSON document | `{_id: ObjectId(), name: "John"}` |
| **Cassandra** | Row | `id=1, name='John'` |
| **Gremlin** | Vertex/Edge | `g.V('1').property('name','John')` |
| **Table** | Entity | `PartitionKey='pk1', RowKey='1', Name='John'` |

### Item Properties

**Required properties** (NoSQL API):

```json
{
  "id": "unique-id",           // Required: Unique within partition
  "category": "electronics",   // Partition key value
  "name": "Laptop",            // Custom properties
  "price": 999.99,
  "_rid": "...",               // System: Resource ID
  "_self": "...",              // System: Self-link
  "_etag": "...",              // System: Concurrency
  "_ts": 1640000000            // System: Timestamp (epoch)
}
```

**System properties** (prefixed with `_`):
- `_rid` - Resource ID (unique)
- `_self` - Self-link URI
- `_etag` - Concurrency control (optimistic locking)
- `_ts` - Last modified timestamp
- `_attachments` - Attachments link (deprecated)

### Item Size Limits

| Item Property | Limit |
|---------------|-------|
| **Max item size** | 2 MB |
| **Max property name length** | No official limit (reasonable) |
| **Max property nesting** | 128 levels |
| **Max array length** | No limit (within 2 MB) |

### CRUD Operations

```csharp
// Create item
var product = new Product
{
    Id = "product-1",
    Category = "electronics",
    Name = "Laptop",
    Price = 999.99m
};
await container.CreateItemAsync(product, new PartitionKey("electronics"));

// Read item (point read - 1 RU for 1KB)
var response = await container.ReadItemAsync<Product>(
    "product-1",
    new PartitionKey("electronics")
);

// Update item
product.Price = 899.99m;
await container.UpsertItemAsync(product, new PartitionKey("electronics"));

// Delete item
await container.DeleteItemAsync<Product>(
    "product-1",
    new PartitionKey("electronics")
);
```

## Hierarchy Best Practices

### 1. Account Planning
```
Production: prod-cosmos-account
Development: dev-cosmos-account
Testing: test-cosmos-account

Reason: Isolated environments, separate billing
```

### 2. Database Organization
```
Account: myapp-cosmos
├── Database: UserData (user profiles, preferences)
├── Database: Orders (orders, payments, invoices)
└── Database: Catalog (products, categories)

Reason: Logical separation, optional shared throughput
```

### 3. Container Design
```
Database: UserData
├── Container: users (partition key: /userId)
├── Container: sessions (partition key: /userId, TTL: 3600)
└── Container: preferences (partition key: /userId)

Reason: Different scaling needs, TTL requirements
```

### 4. Partition Key Selection
```
❌ Bad: /id (every item in different partition)
❌ Bad: /type (few unique values, hot partitions)
✅ Good: /userId (high cardinality, even distribution)
✅ Good: /tenantId (multi-tenant apps)
✅ Good: /category + synthetic (e.g., /category-region)
```

## Cost Implications

### Throughput vs Storage

```
Container A: 10,000 RU/s, 10 GB storage
Cost: ~$58/month (RU/s) + $2.50 (storage) = $60.50/month

Container B: 400 RU/s, 1 TB storage
Cost: ~$23/month (RU/s) + $250 (storage) = $273/month

Insight: Throughput often costs more than storage
```

### Shared vs Dedicated

```
Scenario: 10 small containers, 100 RU/s each

Dedicated: 10 × 400 RU/s (min) = 4,000 RU/s = $230/month
Shared: 1,000 RU/s database = $58/month

Savings: $172/month (75% reduction)
```

## Critical Notes
- 💡 **Account** - Top-level with DNS name, max 50 per subscription
- 🎯 **Database** - Namespace, optional shared throughput
- ✅ **Container** - Scalability unit, unlimited RU/s and storage
- ⚠️ **Partition key** - Cannot change after creation
- 🔄 **Physical partition** - 50 GB, 10,000 RU/s (Azure-managed)
- 📊 **Logical partition** - 20 GB max, defined by partition key
- 💡 **Items** - Max 2 MB, JSON/BSON/Row format
- ✅ **Throughput modes** - Dedicated (container) or Shared (database)
- ⚠️ **Shared throughput** - Max 25 containers, excludes dedicated containers
- 🔒 **Autoscale** - Auto-scale between 10% and 100% of max RU/s

## Exam Tips
- Resource hierarchy: Account → Database → Container → Items
- Account: Unique DNS name, max 50 per subscription (increasable)
- Database: Namespace, logical grouping, optional shared throughput
- Container: Fundamental scalability unit, unlimited throughput/storage
- Partition key: Required, cannot be changed, critical for performance
- Physical partition: 50 GB storage, 10,000 RU/s max (Azure-managed)
- Logical partition: 20 GB max, same partition key value
- Item max size: 2 MB
- Throughput modes: Dedicated (container-level) or Shared (database-level)
- Dedicated throughput: Guaranteed, more expensive, container-exclusive
- Shared throughput: Cost-effective, up to 25 containers, database-level
- Autoscale: Automatically scale between 10% and max RU/s
- API naming: NoSQL=Container, MongoDB=Collection, Cassandra=Table, Gremlin=Graph
- Point read cost: 1 RU for 1KB item (ID + partition key)
- System properties: _rid, _self, _etag, _ts (read-only)
- Partition key best practices: High cardinality, even distribution, query-friendly
- Cannot change: Partition key, unique keys, conflict resolution policy
- Can change: Throughput, indexing policy, TTL

[Learn More](https://learn.microsoft.com/en-us/training/modules/explore-azure-cosmos-db/3-cosmos-db-resource-hierarchy)
