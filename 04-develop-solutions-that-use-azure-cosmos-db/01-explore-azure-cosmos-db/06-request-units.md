# Azure Cosmos DB Request Units

## Key Concepts
- **Request Units (RU)** - Normalized cost of database operations
- **Throughput provisioning** - Reserve RU/s capacity
- **Provisioning modes** - Manual, Autoscale, Serverless
- **Cost model** - Pay for throughput (RU/s) + storage (GB)

## What are Request Units?

### Unified Cost Metric

**Request Unit (RU)** - Abstraction of system resources:

```
1 RU = Resources to read 1 KB item by ID + partition key

System Resources:
├── CPU
├── IOPS (disk I/O)
├── Memory
└── Network bandwidth

All normalized into single metric: RU
```

### Why RUs Matter

**Predictable performance**:

```
Traditional Databases:
├── CPU % → unpredictable
├── Memory GB → complex
├── IOPS → variable
└── Cost → hard to estimate

Azure Cosmos DB:
├── RU → predictable
├── 1 RU = 1 KB point read
├── All operations measured in RUs
└── Cost → simple calculation
```

## RU Consumption Examples

### Point Read

**Most efficient operation**:

```csharp
// Read 1 KB item = 1 RU
await container.ReadItemAsync<Product>(
    "product-1",
    new PartitionKey("electronics")
);
// Cost: 1 RU

// Read 5 KB item = 5 RU
await container.ReadItemAsync<LargeProduct>(
    "product-2",
    new PartitionKey("electronics")
);
// Cost: 5 RU (scales linearly with size)
```

**Point read formula**:
```
RU cost = Item size (KB) × 1 RU
```

### Write Operations

**More expensive than reads**:

```csharp
// Create 1 KB item ≈ 5 RU
await container.CreateItemAsync(product);
// Cost: ~5 RU

// Update 1 KB item ≈ 5 RU
await container.ReplaceItemAsync(product, product.Id);
// Cost: ~5 RU

// Delete 1 KB item ≈ 5 RU
await container.DeleteItemAsync<Product>(
    "product-1",
    new PartitionKey("electronics")
);
// Cost: ~5 RU
```

**Write formula** (approximate):
```
RU cost = Item size (KB) × 5 RU
```

### Query Operations

**Most expensive, variable cost**:

```csharp
// Simple query with partition key
var query = container.GetItemQueryIterator<Product>(
    "SELECT * FROM c WHERE c.category = 'electronics' AND c.price < 1000"
);
// Cost: 2-10 RU (efficient with partition key)

// Cross-partition query
var query = container.GetItemQueryIterator<Product>(
    "SELECT * FROM c WHERE c.price < 1000"
);
// Cost: 10-1000+ RU (scans multiple partitions)

// Complex query with aggregations
var query = container.GetItemQueryIterator<Product>(
    "SELECT c.category, COUNT(1), AVG(c.price) FROM c GROUP BY c.category"
);
// Cost: 100-10000+ RU (depends on data volume)
```

**Query factors affecting RU cost**:
- ✅ Partition key in filter (efficient)
- ⚠️ Cross-partition query (expensive)
- ⚠️ Aggregations (COUNT, AVG, etc.)
- ⚠️ ORDER BY (sorting cost)
- ⚠️ Result set size

## Operation Cost Comparison

### Cost Table

| Operation | Item Size | RU Cost | Notes |
|-----------|-----------|---------|-------|
| **Point read** | 1 KB | 1 RU | Most efficient |
| **Point read** | 10 KB | 10 RU | Linear scaling |
| **Create** | 1 KB | ~5 RU | Indexing cost |
| **Update** | 1 KB | ~5 RU | Re-indexing cost |
| **Delete** | 1 KB | ~5 RU | Index cleanup |
| **Query (in-partition)** | - | 2-10 RU | With partition key |
| **Query (cross-partition)** | - | 10-1000+ RU | Scans partitions |

### Cost Factors

**What affects RU consumption**:

1. **Item size** - Larger items = more RUs (linear)
2. **Operation type** - Writes > Reads
3. **Indexing** - More indexes = higher write cost
4. **Consistency** - Strong ≈ 2× cost of Session/Eventual
5. **Query complexity** - Aggregations, joins, sorting
6. **Partition key** - In-partition vs cross-partition queries

### Optimize RU Usage

```csharp
// ❌ Expensive: Cross-partition query
var query = "SELECT * FROM c WHERE c.price < 100";
// Scans all partitions

// ✅ Efficient: In-partition query
var query = "SELECT * FROM c WHERE c.category = 'electronics' AND c.price < 100";
// Only scans electronics partition

// ❌ Expensive: Read entire item to check one field
var item = await container.ReadItemAsync<Product>(id, pk);
if (item.Resource.IsActive) { /* ... */ }
// Cost: Full item read

// ✅ Efficient: Query only needed fields
var query = "SELECT c.id, c.isActive FROM c WHERE c.id = @id";
// Cost: Minimal (only selected fields)
```

## Throughput Provisioning Modes

### 1. Manual (Provisioned Throughput)

**Fixed RU/s capacity**:

```bash
# Provision 1,000 RU/s
az cosmosdb sql container create \
  --account-name mycosmosaccount \
  --database-name mydb \
  --name mycontainer \
  --partition-key-path "/category" \
  --throughput 1000

# Result:
# - 1,000 RU/s capacity
# - Steady, predictable cost
# - Can exceed briefly (burst), then throttled
```

**Characteristics**:
- ✅ Predictable cost ($0.008/hour per 100 RU/s)
- ✅ Lowest cost for steady workloads
- ⚠️ Manual scaling required
- ⚠️ Throttled if exceeded (429 errors)

**Pricing example**:
```
1,000 RU/s × 730 hours/month × $0.008/100 RU/s
= $58.40/month
```

### 2. Autoscale (Provisioned Throughput)

**Automatic scaling**:

```bash
# Configure autoscale (max 4,000 RU/s)
az cosmosdb sql container create \
  --account-name mycosmosaccount \
  --database-name mydb \
  --name mycontainer \
  --partition-key-path "/category" \
  --max-throughput 4000

# Result:
# - Scales between 400 (10%) and 4,000 RU/s
# - Automatic based on usage
# - Pay for highest RU/s used per hour
```

**Scaling behavior**:
```
Idle: 400 RU/s (10% of max)
Low traffic: 800 RU/s (20% of max)
Medium: 2,000 RU/s (50% of max)
Peak: 4,000 RU/s (100% of max)

Scale up: Instant
Scale down: Gradual (no abrupt drops)
```

**Characteristics**:
- ✅ Automatic scaling
- ✅ No throttling (scales up instantly)
- ✅ Pay for usage (per hour)
- ⚠️ 1.5× cost of manual (convenience premium)

**Pricing example**:
```
Max: 4,000 RU/s
Actual usage:
- 20 hours at 4,000 RU/s = 20 hours × $3.20/hour = $64
- 10 hours at 2,000 RU/s = 10 hours × $1.60/hour = $16
Total: $80

vs Manual 4,000 RU/s always:
730 hours × $3.20/hour = $2,336/month
```

### 3. Serverless

**Pay-per-request** (no provisioning):

```bash
# Create serverless account
az cosmosdb create \
  --name mycosmosaccount \
  --resource-group myResourceGroup \
  --capabilities EnableServerless

# Create container (no throughput specified)
az cosmosdb sql container create \
  --account-name mycosmosaccount \
  --database-name mydb \
  --name mycontainer \
  --partition-key-path "/category"

# No throughput parameter needed
```

**Characteristics**:
- ✅ No provisioning required
- ✅ Pay only for RUs consumed
- ✅ No throttling (elastic capacity)
- ⚠️ Higher per-RU cost ($0.25 per million RUs)
- ⚠️ Max 5,000 RU/s per container
- ⚠️ Max 50 GB per container
- ❌ No SLA on latency/throughput

**Pricing example**:
```
Monthly consumption: 100 million RUs
Cost: 100 × $0.25 = $25/month

vs Manual 400 RU/s (lowest):
$23.36/month (close for low usage)

vs Manual 10,000 RU/s:
$584/month (serverless much cheaper for sporadic use)
```

### Mode Comparison

| Feature | Manual | Autoscale | Serverless |
|---------|--------|-----------|------------|
| **Provisioning** | Fixed RU/s | Max RU/s | None |
| **Scaling** | Manual | Automatic | Automatic |
| **Min RU/s** | 400 | 400 (10% max) | 0 |
| **Max RU/s** | Unlimited | Unlimited | 5,000/container |
| **Cost** | Lowest (steady) | 1.5× manual | Pay-per-use |
| **Throttling** | Yes (429) | No | No |
| **Best for** | Steady traffic | Variable traffic | Sporadic/dev |
| **SLA** | Yes | Yes | No (best-effort) |
| **Max storage** | Unlimited | Unlimited | 50 GB/container |

## Provisioning Levels

### Container-Level Throughput

**Dedicated RU/s per container**:

```bash
# Container with dedicated 1,000 RU/s
az cosmosdb sql container create \
  --account-name mycosmosaccount \
  --database-name mydb \
  --name mycontainer \
  --partition-key-path "/category" \
  --throughput 1000

# Guaranteed capacity:
# - Exclusively for this container
# - Not shared with other containers
# - More expensive but predictable
```

### Database-Level Throughput (Shared)

**Shared RU/s across containers**:

```bash
# Database with 1,000 RU/s shared
az cosmosdb sql database create \
  --account-name mycosmosaccount \
  --name mydb \
  --throughput 1000

# Containers share capacity:
az cosmosdb sql container create \
  --account-name mycosmosaccount \
  --database-name mydb \
  --name container1 \
  --partition-key-path "/category"
  # No --throughput (shares database RU/s)

az cosmosdb sql container create \
  --account-name mycosmosaccount \
  --database-name mydb \
  --name container2 \
  --partition-key-path "/id"
  # Also shares same 1,000 RU/s
```

**Shared throughput characteristics**:
- ✅ Cost-effective for many small containers
- ✅ Up to 25 containers can share
- ⚠️ Minimum 400 RU/s per database
- ⚠️ No guarantees per container (dynamic allocation)
- ⚠️ Cannot mix: Containers either all shared or have dedicated

### Cost Comparison

```
Scenario: 5 containers, each needs ~200 RU/s

Option 1: Dedicated throughput
5 × 400 RU/s (min per container) = 2,000 RU/s
Cost: $116.80/month

Option 2: Shared throughput
1,000 RU/s database (shared)
Cost: $58.40/month

Savings: $58.40/month (50%)
```

## Scaling Throughput

### Manual Scaling

```bash
# Scale up
az cosmosdb sql container throughput update \
  --account-name mycosmosaccount \
  --database-name mydb \
  --name mycontainer \
  --throughput 10000
# Instant scale-up

# Scale down
az cosmosdb sql container throughput update \
  --account-name mycosmosaccount \
  --database-name mydb \
  --name mycontainer \
  --throughput 1000
# May take time if data stored > scale-down allows
```

### Programmatic Scaling

```csharp
// Get current throughput
ThroughputResponse throughput = await container.ReadThroughputAsync();
int? currentRUs = throughput.Resource?.Throughput;

// Scale up
await container.ReplaceThroughputAsync(10000);

// Scale down
await container.ReplaceThroughputAsync(1000);

// Autoscale
await container.ReplaceThroughputAsync(
    ThroughputProperties.CreateAutoscaleThroughput(4000)  // max
);
```

### Scaling Limitations

**Scale-down limitations**:

```
Current: 10,000 RU/s with 50 GB data

Can scale down to:
Minimum = Max(400 RU/s, Storage GB / 100 × 10 RU/s)
        = Max(400, 50 / 100 × 10)
        = Max(400, 5)
        = 400 RU/s ✓

With 100 GB data:
Minimum = Max(400, 100 / 100 × 10)
        = Max(400, 10)
        = 400 RU/s ✓

With 10 TB (10,000 GB) data:
Minimum = Max(400, 10000 / 100 × 10)
        = Max(400, 1000)
        = 1,000 RU/s (can't go below)
```

**Scale-down formula**:
```
Minimum RU/s = Max(400, Storage (GB) / 100 × 10)
```

## Monitoring RU Consumption

### View RU Charge

```csharp
// Check RU cost of operation
var response = await container.CreateItemAsync(product);
double ruCharge = response.RequestCharge;
Console.WriteLine($"Operation consumed {ruCharge} RUs");

// Query RU cost
var query = container.GetItemQueryIterator<Product>(
    "SELECT * FROM c WHERE c.category = 'electronics'"
);

double totalRUs = 0;
while (query.HasMoreResults)
{
    var page = await query.ReadNextAsync();
    totalRUs += page.RequestCharge;
}
Console.WriteLine($"Query consumed {totalRUs} RUs");
```

### Response Headers

```
HTTP/1.1 200 OK
x-ms-request-charge: 5.43
x-ms-retry-after-ms: 0

If throttled (429):
HTTP/1.1 429 Request Rate Too Large
x-ms-request-charge: 10.0
x-ms-retry-after-ms: 50
```

### Azure Portal Metrics

**Monitor in Azure Portal**:
- **Metrics blade** → Select Cosmos DB account
- **Request Units** → View RU/s consumption over time
- **Throttled Requests** → See 429 errors
- **Normalized RU Consumption** → % of provisioned capacity used

## Throttling (429 Errors)

### What is Throttling?

**Request Rate Too Large**:

```
Provisioned: 1,000 RU/s
Consumed: 1,200 RU/s (burst)
     ↓
Result: 429 Too Many Requests
Retry-After: 50 ms
```

### Handling Throttling

```csharp
// SDK handles retries automatically
var clientOptions = new CosmosClientOptions
{
    MaxRetryAttemptsOnRateLimitedRequests = 9,
    MaxRetryWaitTimeOnRateLimitedRequests = TimeSpan.FromSeconds(30)
};

var client = new CosmosClient(endpoint, key, clientOptions);

// SDK will:
// 1. Catch 429 errors
// 2. Wait retry-after duration
// 3. Retry up to 9 times
// 4. Total wait up to 30 seconds
```

### Avoiding Throttling

**Solutions**:

1. **Increase provisioned RU/s**
```bash
az cosmosdb sql container throughput update \
  --name mycontainer \
  --throughput 2000
```

2. **Use autoscale**
```bash
az cosmosdb sql container update \
  --name mycontainer \
  --max-throughput 4000
```

3. **Optimize queries**
```csharp
// ❌ Expensive
"SELECT * FROM c"

// ✅ Efficient
"SELECT c.id, c.name FROM c WHERE c.category = @category"
```

4. **Batch operations**
```csharp
// ❌ Individual creates (5 RU × 100 = 500 RU)
for (int i = 0; i < 100; i++)
{
    await container.CreateItemAsync(items[i]);
}

// ✅ Batch create (more efficient)
var batch = container.CreateTransactionalBatch(partitionKey);
for (int i = 0; i < 100; i++)
{
    batch.CreateItem(items[i]);
}
await batch.ExecuteAsync();
```

## Cost Optimization

### 1. Right-Size Throughput

```
Monitor actual usage:
Peak: 800 RU/s
Average: 400 RU/s

Provisioned: 1,000 RU/s → Wasted capacity

Optimize:
Use autoscale: 400-1,000 RU/s
Save: ~40% on off-peak hours
```

### 2. Use Serverless for Dev/Test

```
Development environment:
Provisioned: 400 RU/s = $23.36/month × 24/7

Serverless: ~10 million RUs/month = $2.50/month
Savings: $20.86/month per environment
```

### 3. Shared Throughput

```
10 small containers:
Dedicated: 10 × 400 RU/s = $233.60/month
Shared: 1,000 RU/s = $58.40/month
Savings: $175.20/month (75%)
```

### 4. Optimize Indexing

```csharp
// Reduce indexing overhead
var indexingPolicy = new IndexingPolicy
{
    IndexingMode = IndexingMode.Consistent,
    IncludedPaths =
    {
        new IncludedPath { Path = "/category/?" },
        new IncludedPath { Path = "/price/?" }
    },
    ExcludedPaths =
    {
        new ExcludedPath { Path = "/*" }  // Exclude all others
    }
};

// Result: Lower write costs (less indexing)
```

## Critical Notes
- 💡 **1 RU** - Cost to read 1 KB item by ID + partition key
- 🎯 **Modes** - Manual, Autoscale, Serverless
- ✅ **Manual** - Fixed RU/s, lowest cost for steady workloads
- ⚠️ **Autoscale** - 10%-100% of max, scales automatically, 1.5× cost
- 🔄 **Serverless** - Pay-per-request, no provisioning, $0.25/million RUs
- 📊 **Point read** - 1 RU per KB (most efficient)
- 💡 **Writes** - ~5 RU per KB (indexing cost)
- ✅ **Queries** - Variable (2-1000+ RU depending on complexity)
- ⚠️ **Throttling** - 429 errors when exceeding capacity
- 🔒 **Minimum** - 400 RU/s for manual, 400 RU/s min for autoscale (10% of max)

## Exam Tips
- Request Unit (RU): Normalized cost of database operations
- 1 RU cost: Read 1 KB item by ID + partition key
- Point read formula: Item size (KB) × 1 RU
- Write formula: Item size (KB) × ~5 RU (approximate)
- Provisioning modes: Manual, Autoscale, Serverless
- Manual: Fixed RU/s, predictable cost, can be throttled
- Autoscale: Scales 10%-100% of max, automatic, 1.5× cost
- Serverless: Pay-per-request, no provisioning, $0.25/million RUs
- Minimum RU/s: 400 for container/database (manual mode)
- Autoscale min: 10% of max RU/s (e.g., max 4000 → min 400)
- Shared throughput: Up to 25 containers share database RU/s
- Dedicated throughput: Guaranteed per container, more expensive
- Throttling: 429 error when exceeding provisioned capacity
- Retry-after header: SDK uses to wait before retry
- Scale-down limit: Max(400, Storage GB / 100 × 10) RU/s
- Strong consistency: ~2× RU cost vs Session/Eventual
- Query optimization: Use partition key in WHERE clause
- Cross-partition query: More expensive (scans multiple partitions)
- Serverless limits: 5,000 RU/s per container, 50 GB max storage
- Autoscale benefit: No throttling, scales instantly on demand
- Cost calculation: RU/s × hours × $0.008/100 RU/s
- Free tier: 1000 RU/s and 25 GB free (first account)

[Learn More](https://learn.microsoft.com/en-us/training/modules/explore-azure-cosmos-db/7-cosmos-db-request-units)
