# Azure Cosmos DB Consistency Levels

## Key Concepts
- **Consistency spectrum** - 5 levels from Strong to Eventual
- **Tradeoffs** - Consistency vs Availability vs Latency vs Throughput
- **Global scope** - Applied across all regions
- **Per-request override** - Relax consistency for specific reads

## Consistency Spectrum

### The Five Levels

```
Strongest ←────────────────────────────→ Weakest
Highest Cost                        Lowest Cost
Lowest Availability                 Highest Availability
Highest Latency                     Lowest Latency

Strong → Bounded Staleness → Session → Consistent Prefix → Eventual
```

### Visual Representation

```
Strong
├── Always reads latest committed write
├── Linearizability guarantee
├── Highest latency
└── May sacrifice availability

Bounded Staleness
├── Lag limited by K versions or T time
├── Consistent within staleness window
├── Good for single-write, multi-read
└── Predictable staleness

Session (DEFAULT)
├── Consistent within client session
├── Read-your-writes guarantee
├── Most common choice
└── Balance of consistency and performance

Consistent Prefix
├── Reads never see out-of-order writes
├── Eventually consistent
├── Maintains write order
└── Good for social media feeds

Eventual
├── No ordering guarantee
├── Highest availability
├── Lowest latency
├── Replicas converge eventually
└── Good for analytics, counters
```

## Consistency Level Comparison

### Quick Reference Table

| Level | Reads | Latency | Throughput | Availability | Use Case |
|-------|-------|---------|------------|--------------|----------|
| **Strong** | Most recent | Highest | Lowest | Lowest | Banking, inventory |
| **Bounded Staleness** | Within lag | High | Low | Medium | Stock quotes |
| **Session** | Your writes | Medium | Medium | High | Shopping carts, profiles |
| **Consistent Prefix** | In order | Low | High | High | Social feeds |
| **Eventual** | May be stale | Lowest | Highest | Highest | Likes, views, analytics |

### CAP Theorem Tradeoffs

```
CAP Theorem: Choose 2 of 3
- Consistency (C)
- Availability (A)  
- Partition Tolerance (P)

Azure Cosmos DB provides:
├── Strong: C + P (may sacrifice A)
├── Bounded Staleness: C + P (within bounds)
├── Session: Balanced (C + A + P within session)
├── Consistent Prefix: A + P (ordered)
└── Eventual: A + P (maximum)
```

## Strong Consistency

### Characteristics

**Linearizability guarantee**:

- ✅ Always reads most recent committed write
- ✅ Writes visible to all readers immediately
- ✅ Never see uncommitted or partial writes
- ❌ Highest latency (cross-region coordination)
- ❌ May block during regional failures

### How It Works

```
Write Request
     ↓
Primary Region Commits
     ↓
Wait for quorum in ALL regions
     ↓
Acknowledge write to client
     ↓
Read from ANY region sees latest data

Result: Highest consistency, highest cost
```

### Multi-Region Behavior

```
App writes "value=100" to Region A
     ↓
Cosmos DB replicates to ALL regions
     ↓
Waits for majority quorum in EACH region
     ↓
Write acknowledged
     ↓
Read from Region B immediately sees "value=100"

Cost: Cross-region round-trip latency
```

### Use Cases

✅ **Financial transactions** - Money transfers, account balances
✅ **Inventory management** - Stock levels, reservations
✅ **Voting systems** - Election results
✅ **Compliance** - Regulatory requirements

### Configuration

```bash
# Set strong consistency at account level
az cosmosdb update \
  --name mycosmosaccount \
  --resource-group myResourceGroup \
  --default-consistency-level Strong
```

```csharp
// Override per request (can only relax, not strengthen)
var response = await container.ReadItemAsync<Product>(
    "product-1",
    new PartitionKey("electronics"),
    new ItemRequestOptions
    {
        ConsistencyLevel = ConsistencyLevel.Strong
    }
);
```

## Bounded Staleness Consistency

### Characteristics

**Configurable lag guarantee**:

- ✅ Staleness bounded by K versions OR T time
- ✅ Predictable maximum staleness
- ✅ Good for single-write, multi-read scenarios
- ⚠️ Writes throttled if lag exceeds bounds

### Configuration Options

**Staleness window**:

| Parameter | Description | Min | Max |
|-----------|-------------|-----|-----|
| **K versions** | Number of item versions lag | 1 | 1,000,000 |
| **T time** | Time interval lag (seconds) | 5 | 86,400 (24 hours) |

### How It Works

```
Multi-Region Configuration:
K = 100 versions
T = 300 seconds (5 minutes)

Write to Region A: version 1000
     ↓
Region B might read: version 900-1000 (within K)
     ↓
If lag > K versions or T time:
     ↓
Writes throttled until caught up
```

### Single-Region Behavior

⚠️ **Important**: For single-region accounts:
- Bounded Staleness = Session + Eventual consistency guarantees
- No staleness bounds enforced (only one region)

### Multi-Region Behavior

```
Primary Region (Write)
     ↓ Async replication
Secondary Regions (Read)
     ↓
Lag monitored per physical partition
     ↓
If lag > configured bounds:
     ↓
Writes throttled to maintain guarantee
```

### Use Cases

✅ **Stock quotes** - Real-time within seconds acceptable
✅ **Monitoring dashboards** - Recent data acceptable
✅ **Leaderboards** - Near real-time rankings
✅ **News feeds** - Slight delay acceptable

### Configuration

```bash
# Set bounded staleness with K versions
az cosmosdb update \
  --name mycosmosaccount \
  --resource-group myResourceGroup \
  --default-consistency-level BoundedStaleness \
  --max-staleness-prefix 100

# Set bounded staleness with T time
az cosmosdb update \
  --name mycosmosaccount \
  --resource-group myResourceGroup \
  --default-consistency-level BoundedStaleness \
  --max-interval 300  # 5 minutes
```

## Session Consistency (DEFAULT)

### Characteristics

**Read-your-writes within session**:

- ✅ Client sees own writes immediately
- ✅ Monotonic reads (don't go backwards)
- ✅ Monotonic writes (writes ordered)
- ✅ Read-your-writes guarantee
- ✅ Write-follows-reads guarantee
- ⚠️ Only within single client session

### Session Scope

```
Client A Session:
Write "value=100"
     ↓
Read sees "value=100" (guaranteed)
     ↓
Write "value=200"
     ↓
Read sees "value=200" (monotonic)

Client B Session (different):
Might see "value=100" (eventual with A's writes)
```

### Session Token

**How sessions work**:

```csharp
// Write returns session token
ItemResponse<Product> writeResponse = await container.CreateItemAsync(product);
string sessionToken = writeResponse.Headers.Session;

// Use session token in subsequent reads
ItemResponse<Product> readResponse = await container.ReadItemAsync<Product>(
    "product-1",
    new PartitionKey("electronics"),
    new ItemRequestOptions { SessionToken = sessionToken }
);

// SDK manages session tokens automatically for same client
```

### Guarantees

| Guarantee | Description | Example |
|-----------|-------------|---------|
| **Read-your-writes** | See your own writes | Write then immediately read |
| **Monotonic reads** | Never go backwards | Read v2, never see v1 later |
| **Monotonic writes** | Writes ordered | Write A before B, all see same order |
| **Write-follows-reads** | Writes see prior reads | Read v1, write based on v1 |

### Multi-Region Behavior

```
Write to Region A
     ↓
Session token includes write LSN (Logical Sequence Number)
     ↓
Read from Region B with session token
     ↓
Region B ensures read is >= LSN from token
     ↓
If not caught up, waits or reads from another region
```

### Use Cases

✅ **Shopping carts** - See items you just added
✅ **User profiles** - See your own edits
✅ **Document editing** - See your changes
✅ **Most applications** - Best balance of consistency and performance

### Configuration

```bash
# Set session consistency (default)
az cosmosdb update \
  --name mycosmosaccount \
  --resource-group myResourceGroup \
  --default-consistency-level Session
```

**💡 Session is the default** - No need to explicitly set unless changing from another level

## Consistent Prefix Consistency

### Characteristics

**Ordered eventual consistency**:

- ✅ Reads never see out-of-order writes
- ✅ Maintains causal relationships
- ✅ Eventually consistent
- ❌ May see stale data
- ❌ No staleness bounds

### How It Works

```
Writes: A → B → C (in order)
     ↓
Possible reads:
✅ A
✅ A, B
✅ A, B, C
❌ B (without A)
❌ A, C (without B)
❌ C, A (out of order)

Guarantee: If you see B, you've seen A
```

### Causal Consistency Example

```
Social Media Scenario:
User posts: "Check out this photo!" (Write 1)
User uploads: photo.jpg (Write 2)

Consistent Prefix ensures:
- Never see photo without post
- Never see post without context
- May take time to see both
- But always in correct order
```

### Single vs Batch Writes

#### Single Document Writes
```csharp
// Single writes: eventual consistency
await container.CreateItemAsync(doc1);
await container.CreateItemAsync(doc2);

// Readers might see:
// - Neither
// - doc1 only
// - Both
// - doc2 only (NOT guaranteed ordered for separate writes)
```

#### Transactional Batch
```csharp
// Batch writes: consistent prefix guaranteed
TransactionalBatch batch = container.CreateTransactionalBatch(
    new PartitionKey("category")
);
batch.CreateItem(doc1);
batch.CreateItem(doc2);
await batch.ExecuteAsync();

// Readers see:
// - Neither
// - Both (in order)
// - Never doc2 without doc1
```

### Use Cases

✅ **Social media feeds** - Posts in chronological order
✅ **Comment threads** - Replies after parent comments
✅ **Event streams** - Events in sequence
✅ **Audit logs** - Ordered history

### Configuration

```bash
# Set consistent prefix
az cosmosdb update \
  --name mycosmosaccount \
  --resource-group myResourceGroup \
  --default-consistency-level ConsistentPrefix
```

## Eventual Consistency

### Characteristics

**Weakest guarantee, highest performance**:

- ❌ No ordering guarantee
- ❌ May read stale or old data
- ❌ May see values older than previous reads
- ✅ Lowest latency
- ✅ Highest throughput
- ✅ Highest availability
- ✅ Replicas eventually converge

### How It Works

```
Write "value=100"
     ↓
Asynchronous replication to all regions
     ↓
Read from Region A: might see "value=100"
Read from Region B: might see old value
Read again from Region A: might see older value (time warp)
     ↓
Eventually all regions converge to "value=100"

No guarantees on timing or ordering
```

### Non-Monotonic Reads Example

```
Timeline:
T0: value=0 (in all regions)
T1: Write value=100 (Region A)
T2: Read from Region A → value=100 ✓
T3: Read from Region B → value=0 ⚠️ (went backwards!)
T4: All regions caught up → value=100
```

### Use Cases

✅ **Analytics** - Aggregations, counts (eventual accuracy OK)
✅ **Social counters** - Likes, views, shares
✅ **Non-critical metrics** - Download counts, page views
✅ **Caching** - Read-heavy, infrequently updated
✅ **IoT telemetry** - Sensor data (individual values less critical)

### Anti-Patterns

❌ **Don't use for**:
- User-facing data requiring consistency
- Financial data
- Inventory management
- Any scenario requiring read-your-writes

### Configuration

```bash
# Set eventual consistency
az cosmosdb update \
  --name mycosmosaccount \
  --resource-group myResourceGroup \
  --default-consistency-level Eventual
```

## Consistency Level Scope

### Region-Agnostic

**All consistency levels work globally**:

```
Account with 5 regions:
├── Region 1 (East US)
├── Region 2 (West US)
├── Region 3 (Europe)
├── Region 4 (Asia)
└── Region 5 (Australia)

Consistency level applies uniformly across ALL regions
```

### Guaranteed Regardless Of:

✅ **Region serving reads/writes** - Works in any region
✅ **Number of regions** - 1 region or 50 regions
✅ **Read/write region configuration** - Single or multi-write

### Partition-Scoped Operations

**Read consistency applies to**:

- Single read operation
- Scoped within partition-key range
- Or logical partition

```csharp
// Consistency applies to this specific read
var item = await container.ReadItemAsync<Product>(
    "product-1",
    new PartitionKey("electronics")  // Scoped to this partition
);

// Different partition = independent consistency view
var item2 = await container.ReadItemAsync<Product>(
    "product-2",
    new PartitionKey("books")  // Different consistency view
);
```

## Configuring Consistency

### Account-Level Default

```bash
# Set default consistency for entire account
az cosmosdb update \
  --name mycosmosaccount \
  --resource-group myResourceGroup \
  --default-consistency-level Session

# Applies to:
# - All databases
# - All containers
# - All operations (unless overridden)
```

### Per-Request Override

**Can only relax consistency, not strengthen**:

```csharp
// Account default: Session

// ✅ Can relax to Eventual
var response = await container.ReadItemAsync<Product>(
    "product-1",
    new PartitionKey("electronics"),
    new ItemRequestOptions
    {
        ConsistencyLevel = ConsistencyLevel.Eventual  // Weaker OK
    }
);

// ❌ Cannot strengthen to Strong (if account default is Session)
var response2 = await container.ReadItemAsync<Product>(
    "product-1",
    new PartitionKey("electronics"),
    new ItemRequestOptions
    {
        ConsistencyLevel = ConsistencyLevel.Strong  // Error!
    }
);
```

### Override Rules

| Account Level | Can Override To |
|---------------|-----------------|
| **Strong** | None (already strongest) |
| **Bounded Staleness** | Session, Consistent Prefix, Eventual |
| **Session** | Consistent Prefix, Eventual |
| **Consistent Prefix** | Eventual |
| **Eventual** | None (already weakest) |

## Performance & Cost Impact

### Read Latency

```
Strong:             ~20-50ms (cross-region quorum)
Bounded Staleness:  ~10-20ms (within bounds)
Session:            ~5-10ms (local + session token)
Consistent Prefix:  ~5-10ms (local read)
Eventual:           ~5ms (local read)
```

### RU Consumption

**Relative RU cost** for same operation:

```
Strong:             2x RU (cross-region coordination)
Bounded Staleness:  1.5x RU (monitoring lag)
Session:            1x RU (baseline)
Consistent Prefix:  1x RU
Eventual:           1x RU
```

### Throughput Impact

```
Account: 10,000 RU/s provisioned

Strong consistency:
Effective: ~5,000 ops/sec

Eventual consistency:
Effective: ~10,000 ops/sec

Reason: Strong requires cross-region coordination
```

## Choosing the Right Level

### Decision Tree

```
Do you need linearizability (banking, inventory)?
├── Yes → Strong
└── No ↓

Do you need bounded staleness (stock quotes, monitoring)?
├── Yes → Bounded Staleness (configure K and T)
└── No ↓

Do you need read-your-writes (user-facing apps)?
├── Yes → Session (DEFAULT - best for most apps)
└── No ↓

Do you need ordered reads (feeds, logs)?
├── Yes → Consistent Prefix
└── No ↓

Analytics, counters, non-critical?
└── Yes → Eventual
```

### Common Scenarios

| Scenario | Recommended Level | Reason |
|----------|------------------|---------|
| **E-commerce cart** | Session | User sees their actions |
| **Banking** | Strong | Money correctness critical |
| **Social feed** | Consistent Prefix | Chronological order |
| **Page views counter** | Eventual | Accuracy not time-critical |
| **Stock quotes** | Bounded Staleness | Near real-time acceptable |
| **User profile** | Session | User sees their edits |
| **Analytics dashboard** | Eventual | Aggregate data, eventual accuracy OK |

## Critical Notes
- 💡 **Five levels** - Strong, Bounded Staleness, Session, Consistent Prefix, Eventual
- 🎯 **Session default** - Best balance for most applications
- ✅ **Region-agnostic** - Applied globally across all regions
- ⚠️ **Override rule** - Can only relax consistency, not strengthen
- 🔄 **Tradeoff** - Consistency ↔ Availability + Latency + Throughput
- 📊 **Strong** - Highest consistency, highest cost, may sacrifice availability
- 💡 **Session** - Read-your-writes within client session
- ✅ **Bounded Staleness** - Configurable lag (K versions or T time)
- ⚠️ **Eventual** - No ordering, highest performance, lowest cost
- 🔒 **100% guarantee** - All consistency levels backed by SLA

## Exam Tips
- Default consistency: Session (read-your-writes within session)
- Strongest: Strong (linearizability, most recent write always)
- Weakest: Eventual (no ordering, highest availability)
- Bounded Staleness: Lag limited by K versions OR T time (whichever first)
- Bounded Staleness limits: K (1 to 1M), T (5 sec to 24 hours)
- Session guarantees: Read-your-writes, monotonic reads, monotonic writes
- Consistent Prefix: Ordered eventual consistency (never out-of-order)
- Override rule: Can only relax consistency, not strengthen
- Strong RU cost: ~2x compared to Session/Eventual
- Region-agnostic: Consistency applies across all regions
- Single-region Bounded Staleness: Behaves like Session + Eventual
- Session token: Managed by SDK automatically
- 100% guarantee: All reads meet consistency level SLA
- Partition-scoped: Consistency applies within partition-key range
- Account-level: Default set at account, override per request
- Strong use case: Banking, inventory, voting
- Session use case: Shopping carts, user profiles (most apps)
- Eventual use case: Analytics, counters, non-critical metrics
- Consistency spectrum: Strong → BS → Session → CP → Eventual
- CAP tradeoffs: Strong (C+P), Session (balanced), Eventual (A+P)

[Learn More](https://learn.microsoft.com/en-us/training/modules/explore-azure-cosmos-db/4-cosmos-db-consistency-levels-overview)
