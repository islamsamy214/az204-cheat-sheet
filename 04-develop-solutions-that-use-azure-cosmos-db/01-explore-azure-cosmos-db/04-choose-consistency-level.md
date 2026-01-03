# Choose the Right Consistency Level

## Key Concepts
- **Default consistency** - Account-level configuration
- **Per-request override** - Relax for specific operations
- **Use case alignment** - Match level to business requirements
- **Tradeoff understanding** - Balance consistency, availability, performance, cost

## Configuring Default Consistency

### Account-Level Setting

**Applies to all operations** unless overridden:

```bash
# Set default consistency at account creation
az cosmosdb create \
  --name mycosmosaccount \
  --resource-group myResourceGroup \
  --locations regionName=eastus \
  --default-consistency-level Session

# Update existing account
az cosmosdb update \
  --name mycosmosaccount \
  --resource-group myResourceGroup \
  --default-consistency-level BoundedStaleness \
  --max-staleness-prefix 100 \
  --max-interval 300
```

### What Gets the Default?

**Inherits account default**:
- ✅ All databases
- ✅ All containers
- ✅ All read operations
- ✅ All query operations
- ⚠️ Unless explicitly overridden per request

```
Account: Session consistency
     ↓
Database 1: Session (inherited)
├── Container A: Session
└── Container B: Session
     ↓
Database 2: Session (inherited)
└── Container C: Session

All operations use Session unless overridden
```

## Consistency Level Guarantees

### Strong Consistency

**Linearizability guarantee**:

```
Timeline:
T1: Client A writes value=100
T2: Write committed to quorum in all regions
T3: Client B reads from any region → value=100 (guaranteed)

Guarantee: Always reads most recent committed write
Cost: Highest latency, lowest throughput
```

**Characteristics**:
- ✅ Never see uncommitted writes
- ✅ Never see partial writes
- ✅ Always see latest committed version
- ❌ Blocks on cross-region coordination
- ❌ May impact availability during failures

**When to use**:
```
✅ Banking: Account balances, transfers
✅ Inventory: Stock levels, reservations  
✅ Voting: Election results
✅ Regulatory: Compliance requirements
❌ Social media: Not worth the cost
❌ Analytics: Overkill for aggregates
```

### Bounded Staleness Consistency

**Staleness within configured bounds**:

```
Configuration:
K = 100 versions
T = 5 minutes

Scenario:
Primary region at version 1000
Secondary might read: version 900-1000 (within K)

If lag exceeds bounds:
Writes throttled until replicas catch up
```

**Staleness Window Options**:

| Bound Type | Min | Max | Use Case |
|------------|-----|-----|----------|
| **K versions** | 1 | 1,000,000 | Predictable version lag |
| **T time** | 5 sec | 86,400 sec (24h) | Time-based guarantees |

**Multi-Region Behavior**:
```
Write Region: East US (version 1000)
     ↓
Read Region: West US
     ↓
If lag > K or T:
     ├── Reads see data within bounds
     └── Writes throttled to maintain guarantee
```

**Single-Region Behavior**:
```
⚠️ Important: For single-region accounts:
Bounded Staleness = Session + Eventual consistency
No staleness bounds enforced (only one region to sync)
```

**When to use**:
```
✅ Stock quotes: Near real-time (5-10 sec lag OK)
✅ Monitoring: Dashboard metrics (recent data acceptable)
✅ Leaderboards: Gaming scores (slight delay OK)
❌ Banking: Need exact values
❌ User profiles: Session better for read-your-writes
```

### Session Consistency

**Read-your-writes within session**:

```
Client Session:
├── Write: value=100
├── Read: value=100 (guaranteed - read-your-write)
├── Write: value=200
└── Read: value=200 (guaranteed - monotonic reads)

Different Client:
└── Might see value=100 or 200 (eventual with other clients)
```

**Guarantees**:

| Guarantee | Description | Example |
|-----------|-------------|---------|
| **Read-your-writes** | See own writes immediately | Add to cart, see item |
| **Monotonic reads** | Never go backwards | See v2, never see v1 later |
| **Monotonic writes** | Writes ordered | Write A before B preserved |
| **Write-follows-reads** | Writes observe prior reads | Read v1, update based on v1 |

**Session Token Management**:

```csharp
// SDK manages automatically for same client instance
CosmosClient client = new CosmosClient(endpoint, key);
var container = client.GetContainer("db", "container");

// Write
await container.CreateItemAsync(new Product { Id = "1", Name = "Laptop" });

// Read - automatically sees own write (session token managed by SDK)
var item = await container.ReadItemAsync<Product>("1", new PartitionKey("1"));

// Manual session token (advanced scenarios)
ItemResponse<Product> writeResponse = await container.CreateItemAsync(product);
string sessionToken = writeResponse.Headers.Session;

// Use in subsequent read
await container.ReadItemAsync<Product>(
    "1",
    new PartitionKey("1"),
    new ItemRequestOptions { SessionToken = sessionToken }
);
```

**When to use**:
```
✅ Shopping carts: User sees their actions
✅ User profiles: See own edits
✅ Document editing: See your changes
✅ Most web apps: Default choice for user-facing apps
❌ Multi-user collaboration: Users don't see each other's changes immediately
```

### Consistent Prefix Consistency

**Ordered eventual consistency**:

```
Writes: A → B → C → D (in order)
     ↓
Possible reads:
✅ Empty
✅ A
✅ A, B
✅ A, B, C
✅ A, B, C, D
❌ B (missing A)
❌ A, C (missing B)
❌ D, A (out of order)

Guarantee: Never out-of-order, but may be stale
```

**Transaction Behavior**:

```csharp
// Single writes: eventual consistency
await container.CreateItemAsync(doc1);  // Write 1
await container.CreateItemAsync(doc2);  // Write 2
// Readers might see doc2 without doc1 ⚠️

// Transactional batch: consistent prefix guaranteed
TransactionalBatch batch = container.CreateTransactionalBatch(
    new PartitionKey("category")
);
batch.CreateItem(doc1);
batch.CreateItem(doc2);
await batch.ExecuteAsync();
// Readers see neither or both (in order) ✅
```

**When to use**:
```
✅ Social media feeds: Posts in chronological order
✅ Comment threads: Replies after parent
✅ Event streams: Sequence matters
✅ Audit logs: Ordered history
❌ User-facing writes: Need read-your-writes (use Session)
❌ Analytics: Order doesn't matter (use Eventual)
```

### Eventual Consistency

**No ordering guarantee**:

```
Timeline:
T0: value=0 (all regions)
T1: Write value=100 (Region A)
T2: Read Region A → 100
T3: Read Region B → 0 (hasn't replicated yet)
T4: Read Region A → 0 (time warp! ⚠️)
T5: All regions → 100 (eventually consistent)

Guarantee: Replicas eventually converge
No guarantees on timing or ordering
```

**Characteristics**:
- ❌ No ordering
- ❌ May read stale data
- ❌ May read older than previous read
- ✅ Lowest latency
- ✅ Highest throughput  
- ✅ Highest availability
- ✅ Lowest cost

**When to use**:
```
✅ Counters: Likes, views, shares (eventual accuracy OK)
✅ Analytics: Aggregations, dashboards
✅ Telemetry: IoT sensor data
✅ Caching: Read-heavy, infrequently updated
❌ User-facing critical data: Use Session
❌ Financial data: Use Strong
❌ Inventory: Use Strong or Bounded Staleness
```

## Relaxing Consistency Per Request

### Override Rules

**Can only relax, not strengthen**:

```csharp
// Account default: Session

// ✅ Relax to Eventual (weaker)
var response = await container.ReadItemAsync<Product>(
    "product-1",
    new PartitionKey("electronics"),
    new ItemRequestOptions
    {
        ConsistencyLevel = ConsistencyLevel.Eventual
    }
);

// ❌ Strengthen to Strong (ERROR!)
var response2 = await container.ReadItemAsync<Product>(
    "product-1",
    new PartitionKey("electronics"),
    new ItemRequestOptions
    {
        ConsistencyLevel = ConsistencyLevel.Strong  // Throws exception!
    }
);
```

### Override Matrix

| Account Default | Can Override To |
|-----------------|-----------------|
| **Strong** | None (already strongest) |
| **Bounded Staleness** | Session, Consistent Prefix, Eventual |
| **Session** | Consistent Prefix, Eventual |
| **Consistent Prefix** | Eventual |
| **Eventual** | None (already weakest) |

### Why Relax Consistency?

**Scenarios for per-request override**:

1. **Analytics queries**:
```csharp
// Account default: Session (for user-facing operations)
// Analytics query can use Eventual (performance + cost savings)
var query = container.GetItemQueryIterator<Product>(
    new QueryDefinition("SELECT COUNT(1) FROM c"),
    requestOptions: new QueryRequestOptions
    {
        ConsistencyLevel = ConsistencyLevel.Eventual
    }
);
```

2. **Background processing**:
```csharp
// User operations: Session (read-your-writes)
await container.CreateItemAsync(product);

// Background sync: Eventual (performance)
var items = container.GetItemQueryIterator<Product>(
    "SELECT * FROM c WHERE c.status = 'pending'",
    requestOptions: new QueryRequestOptions
    {
        ConsistencyLevel = ConsistencyLevel.Eventual
    }
);
```

3. **Non-critical reads**:
```csharp
// Critical: User's own cart (Session)
var cart = await container.ReadItemAsync<Cart>(
    userId,
    new PartitionKey(userId)
);

// Non-critical: Product recommendations (Eventual - performance boost)
var recommendations = container.GetItemQueryIterator<Product>(
    "SELECT TOP 10 * FROM c WHERE c.category = @category",
    requestOptions: new QueryRequestOptions
    {
        ConsistencyLevel = ConsistencyLevel.Eventual
    }
);
```

## Practical Decision Framework

### Step 1: Identify Requirements

**Ask these questions**:

| Question | Answer → Level |
|----------|----------------|
| Need linearizability? | Yes → Strong |
| Multi-user real-time collaboration? | Yes → Strong or Bounded Staleness |
| User needs to see own writes? | Yes → Session |
| Order of operations matters? | Yes → Consistent Prefix |
| Analytics/counters only? | Yes → Eventual |

### Step 2: Consider Multi-Region

**Single write region**:
```
Scenario: Application in one region, read replicas globally

Strong: High latency for writes (cross-region quorum)
Bounded Staleness: Good choice (predictable lag)
Session: Best for user-facing (read-your-writes)
```

**Multi-write regions**:
```
Scenario: Write from any region

Strong: Very high latency (all-region quorum)
Bounded Staleness: Conflicts possible, good for certain scenarios
Session: Best default (per-user consistency)
Eventual: Conflicts common, use conflict resolution
```

### Step 3: Evaluate Tradeoffs

**Performance vs Consistency**:

```
                Consistency
                    ↑
                  Strong
                    |
              Bounded Staleness
                    |
                 Session (DEFAULT - sweet spot)
                    |
            Consistent Prefix
                    |
                 Eventual
                    ↓
              Performance →
```

### Step 4: Cost Consideration

**RU consumption comparison** (same operation):

```
Strong:             2x RU
Bounded Staleness:  1.5x RU
Session:            1x RU (baseline)
Consistent Prefix:  1x RU
Eventual:           1x RU

Example: 10,000 RU/s provisioned
Strong: Effective ~5,000 ops/sec
Eventual: Effective ~10,000 ops/sec
```

## Common Patterns

### Pattern 1: Hybrid Approach

```csharp
// Account default: Session

public class OrderService
{
    // Critical operations: Use default (Session)
    public async Task CreateOrder(Order order)
    {
        await container.CreateItemAsync(order);
        // Session ensures user sees their order immediately
    }

    // Analytics: Relax to Eventual
    public async Task<int> GetTotalOrders()
    {
        var query = container.GetItemQueryIterator<Order>(
            "SELECT VALUE COUNT(1) FROM c",
            requestOptions: new QueryRequestOptions
            {
                ConsistencyLevel = ConsistencyLevel.Eventual
            }
        );
        // Eventual: Better performance, eventual accuracy OK
    }
}
```

### Pattern 2: Progressive Consistency

```csharp
// Start with Session, upgrade to Strong if needed
public async Task<Product> GetProduct(string id, bool critical = false)
{
    var options = new ItemRequestOptions();
    
    if (critical)
    {
        // Critical operation: Cannot override to Strong if account is Session
        // Must set account to Strong and relax others
        // For now, use Session (best available)
    }
    
    return await container.ReadItemAsync<Product>(
        id,
        new PartitionKey(id),
        options
    );
}
```

### Pattern 3: Read-Heavy Optimization

```csharp
// Account: Session (for writes and user-facing reads)
// Background reads: Eventual (performance)

public async Task SyncToCache()
{
    // Background job - use Eventual for better performance
    var query = container.GetItemQueryIterator<Product>(
        "SELECT * FROM c",
        requestOptions: new QueryRequestOptions
        {
            ConsistencyLevel = ConsistencyLevel.Eventual
        }
    );
    
    await foreach (var product in query)
    {
        await cache.SetAsync(product.Id, product);
    }
}
```

## Anti-Patterns

### ❌ Anti-Pattern 1: Wrong Default

```csharp
// BAD: Account default = Eventual
// User writes then immediately reads → might not see their write ⚠️

await container.CreateItemAsync(newOrder);
var order = await container.ReadItemAsync<Order>(orderId, pk);
// Might not see the order just created!

// GOOD: Account default = Session
// Guarantees read-your-writes
```

### ❌ Anti-Pattern 2: Overusing Strong

```csharp
// BAD: Everything uses Strong consistency
// Account default: Strong
// Every operation pays 2x RU cost and high latency

// GOOD: Use Strong only where needed
// Account default: Session
// Override to Strong only for critical operations
```

### ❌ Anti-Pattern 3: Trying to Strengthen

```csharp
// BAD: Account default = Session
await container.ReadItemAsync<Product>(
    id,
    pk,
    new ItemRequestOptions
    {
        ConsistencyLevel = ConsistencyLevel.Strong  // ERROR! ❌
    }
);

// GOOD: Change account default to Strong, relax others to Session
```

## Critical Notes
- 💡 **Default** - Session consistency is default and best for most apps
- 🎯 **Account-level** - Set at account, applies to all operations
- ✅ **Override rule** - Can only relax consistency, not strengthen
- ⚠️ **Per-request** - Override on specific reads/queries for optimization
- 🔄 **Strong** - Linearizability, highest cost, use sparingly
- 📊 **Session** - Read-your-writes within client session, best balance
- 💡 **Eventual** - Analytics and counters, highest performance
- ✅ **100% SLA** - All levels guaranteed by Azure Cosmos DB
- 🔒 **Multi-region** - Consistency applies uniformly across all regions
- ⚠️ **Bounded Staleness single-region** - Behaves like Session + Eventual

## Exam Tips
- Default consistency: Session (read-your-writes within session)
- Configure at: Account level (applies to all databases/containers)
- Override: Can only relax (weaken), not strengthen
- Strong guarantee: Linearizability, always most recent write
- Session guarantee: Read-your-writes, monotonic reads, monotonic writes
- Bounded Staleness: Lag limited by K versions OR T time
- Consistent Prefix: Never out-of-order, but may be stale
- Eventual: No ordering, highest performance, lowest cost
- Strong use case: Banking, inventory, voting systems
- Session use case: Shopping carts, user profiles, most web apps
- Eventual use case: Analytics, counters, telemetry
- RU cost: Strong (~2x), Session (1x), Eventual (1x)
- Single-region Bounded Staleness: Equivalent to Session + Eventual
- Session token: Managed automatically by SDK
- Override scenarios: Analytics (Eventual), background jobs (Eventual)
- Multi-write + Strong: Very high latency (all-region quorum)
- Multi-write + Session: Best balance for multi-region writes
- 100% guarantee: All reads meet consistency level SLA
- Change default: Can change anytime (no downtime)

[Learn More](https://learn.microsoft.com/en-us/training/modules/explore-azure-cosmos-db/5-choose-cosmos-db-consistency-level)
