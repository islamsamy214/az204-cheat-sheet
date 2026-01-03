# Azure Cosmos DB .NET SDK v3

## Key Concepts
- **Microsoft.Azure.Cosmos** - NuGet package for .NET SDK v3
- **CosmosClient** - Thread-safe client (singleton pattern)
- **Async operations** - All operations are asynchronous
- **Generic terms** - Container (not collection), Item (not document)

## SDK Overview

### What is .NET SDK v3?

**Modern .NET SDK** for Azure Cosmos DB:

- **Package**: `Microsoft.Azure.Cosmos`
- **Target**: .NET Standard 2.0 (compatible with .NET Core, .NET Framework)
- **Generic terminology**: Container and Item (works across all APIs)
- **Performance**: Optimized for throughput and latency
- **Async-first**: All operations return `Task`

### SDK Evolution

```
SDK v2 (Legacy)
├── Collection (MongoDB specific)
├── Document (NoSQL specific)
└── Microsoft.Azure.DocumentDB

SDK v3 (Current)
├── Container (generic)
├── Item (generic)
└── Microsoft.Azure.Cosmos ✅
```

### Installation

```bash
# Install NuGet package
dotnet add package Microsoft.Azure.Cosmos

# Or via Package Manager
Install-Package Microsoft.Azure.Cosmos
```

```xml
<!-- In .csproj -->
<ItemGroup>
  <PackageReference Include="Microsoft.Azure.Cosmos" Version="3.*" />
</ItemGroup>
```

## CosmosClient

### Creating the Client

**Thread-safe singleton**:

```csharp
using Microsoft.Azure.Cosmos;

// Recommended: Single instance per application lifetime
public class CosmosDbService
{
    private static CosmosClient _client;

    public static CosmosClient GetClient()
    {
        if (_client == null)
        {
            string endpoint = "https://myaccount.documents.azure.com:443/";
            string key = "your-primary-key";
            
            _client = new CosmosClient(endpoint, key);
        }
        
        return _client;
    }
}

// Usage
CosmosClient client = CosmosDbService.GetClient();
```

### Client Options

```csharp
var clientOptions = new CosmosClientOptions
{
    ApplicationName = "MyApp",
    ConnectionMode = ConnectionMode.Direct,  // Default, better performance
    ConsistencyLevel = ConsistencyLevel.Session,
    MaxRetryAttemptsOnRateLimitedRequests = 9,
    MaxRetryWaitTimeOnRateLimitedRequests = TimeSpan.FromSeconds(30),
    RequestTimeout = TimeSpan.FromSeconds(30),
    AllowBulkExecution = true  // Enable bulk operations
};

CosmosClient client = new CosmosClient(endpoint, key, clientOptions);
```

### Connection String

```csharp
// Alternative: Use connection string
string connectionString = "AccountEndpoint=https://myaccount.documents.azure.com:443/;AccountKey=your-key;";
CosmosClient client = new CosmosClient(connectionString);
```

### Client Lifetime Best Practices

✅ **Do**:
- Create single `CosmosClient` instance per application
- Reuse across entire application lifetime
- Register as singleton in DI container

❌ **Don't**:
- Create new client per request
- Dispose client after each operation
- Create multiple clients for same account

```csharp
// ❌ Bad: New client per request
public async Task<Product> GetProduct(string id)
{
    using var client = new CosmosClient(endpoint, key);  // Inefficient!
    // ... operation
}

// ✅ Good: Reuse singleton client
private readonly CosmosClient _client;

public ProductService(CosmosClient client)
{
    _client = client;  // Inject singleton
}

public async Task<Product> GetProduct(string id)
{
    // Use injected client
}
```

## Database Operations

### Create Database

```csharp
// Throws exception if database exists
Database database1 = await client.CreateDatabaseAsync(
    id: "mydb"
);

// Creates only if doesn't exist (recommended)
Database database2 = await client.CreateDatabaseIfNotExistsAsync(
    id: "mydb"
);

// With throughput
Database database3 = await client.CreateDatabaseIfNotExistsAsync(
    id: "mydb",
    throughput: 1000  // Manual RU/s
);

// With autoscale
Database database4 = await client.CreateDatabaseIfNotExistsAsync(
    id: "mydb",
    ThroughputProperties.CreateAutoscaleThroughput(4000)  // Max RU/s
);
```

### Get Database Reference

```csharp
// Get reference (doesn't make network call)
Database database = client.GetDatabase("mydb");

// Read database properties (network call)
DatabaseResponse response = await database.ReadAsync();
Console.WriteLine($"Database ID: {response.Resource.Id}");
Console.WriteLine($"RU charge: {response.RequestCharge}");
```

### Read Database

```csharp
// Read database details
Database database = client.GetDatabase("mydb");
DatabaseResponse response = await database.ReadAsync();

// Access properties
string id = response.Resource.Id;
string selfLink = response.Resource.SelfLink;
double ruCharge = response.RequestCharge;
```

### Delete Database

```csharp
Database database = client.GetDatabase("mydb");
await database.DeleteAsync();

// With response
DatabaseResponse response = await database.DeleteAsync();
Console.WriteLine($"Deleted database. RU charge: {response.RequestCharge}");
```

### List Databases

```csharp
// Get all databases
FeedIterator<DatabaseProperties> iterator = client.GetDatabaseQueryIterator<DatabaseProperties>();

while (iterator.HasMoreResults)
{
    FeedResponse<DatabaseProperties> databases = await iterator.ReadNextAsync();
    
    foreach (DatabaseProperties db in databases)
    {
        Console.WriteLine($"Database: {db.Id}");
    }
}
```

## Container Operations

### Create Container

```csharp
// Minimum configuration
ContainerResponse container1 = await database.CreateContainerIfNotExistsAsync(
    id: "products",
    partitionKeyPath: "/category",
    throughput: 400
);

// Full configuration
ContainerProperties containerProperties = new ContainerProperties
{
    Id = "products",
    PartitionKeyPath = "/category",
    IndexingPolicy = new IndexingPolicy
    {
        Automatic = true,
        IndexingMode = IndexingMode.Consistent
    },
    DefaultTimeToLive = -1  // Enable TTL, no default expiration
};

ContainerResponse container2 = await database.CreateContainerIfNotExistsAsync(
    containerProperties,
    throughput: 400
);

// With autoscale
ContainerResponse container3 = await database.CreateContainerIfNotExistsAsync(
    id: "orders",
    partitionKeyPath: "/customerId",
    ThroughputProperties.CreateAutoscaleThroughput(4000)
);
```

### Get Container Reference

```csharp
// Get reference (no network call)
Container container = database.GetContainer("products");

// Read container properties (network call)
ContainerResponse response = await container.ReadContainerAsync();
Console.WriteLine($"Container ID: {response.Resource.Id}");
Console.WriteLine($"Partition Key: {response.Resource.PartitionKeyPath}");
```

### Read Container

```csharp
Container container = database.GetContainer("products");
ContainerProperties properties = await container.ReadContainerAsync();

// Access properties
string id = properties.Id;
string partitionKeyPath = properties.PartitionKeyPath;
IndexingPolicy indexing = properties.IndexingPolicy;
```

### Delete Container

```csharp
Container container = database.GetContainer("products");
await container.DeleteContainerAsync();

// With response
ContainerResponse response = await container.DeleteContainerAsync();
Console.WriteLine($"Deleted container. RU charge: {response.RequestCharge}");
```

### Read Container Throughput

```csharp
Container container = database.GetContainer("products");

// Read throughput
ThroughputResponse throughput = await container.ReadThroughputAsync();

if (throughput != null)
{
    Console.WriteLine($"Current throughput: {throughput.Resource?.Throughput} RU/s");
    Console.WriteLine($"Min throughput: {throughput.MinThroughput}");
}
```

### Update Container Throughput

```csharp
// Scale up/down
await container.ReplaceThroughputAsync(10000);

// Switch to autoscale
await container.ReplaceThroughputAsync(
    ThroughputProperties.CreateAutoscaleThroughput(4000)
);
```

## Item Operations (CRUD)

### Create Item

```csharp
public class Product
{
    [JsonProperty("id")]
    public string Id { get; set; }
    
    [JsonProperty("category")]
    public string Category { get; set; }
    
    [JsonProperty("name")]
    public string Name { get; set; }
    
    [JsonProperty("price")]
    public decimal Price { get; set; }
}

// Create item
var product = new Product
{
    Id = "product-1",
    Category = "electronics",
    Name = "Laptop",
    Price = 999.99m
};

ItemResponse<Product> response = await container.CreateItemAsync(
    product,
    new PartitionKey(product.Category)
);

Console.WriteLine($"Created item. RU charge: {response.RequestCharge}");
Console.WriteLine($"Status code: {response.StatusCode}");
```

### Read Item (Point Read)

```csharp
// Point read - Most efficient (1 RU for 1KB)
ItemResponse<Product> response = await container.ReadItemAsync<Product>(
    id: "product-1",
    partitionKey: new PartitionKey("electronics")
);

Product product = response.Resource;
Console.WriteLine($"Product: {product.Name}, Price: ${product.Price}");
Console.WriteLine($"RU charge: {response.RequestCharge}");  // ~1 RU for 1KB
```

### Update Item (Replace)

```csharp
// Read item
ItemResponse<Product> readResponse = await container.ReadItemAsync<Product>(
    "product-1",
    new PartitionKey("electronics")
);

Product product = readResponse.Resource;

// Modify
product.Price = 899.99m;

// Replace (full document update)
ItemResponse<Product> updateResponse = await container.ReplaceItemAsync(
    product,
    product.Id,
    new PartitionKey(product.Category)
);

Console.WriteLine($"Updated item. RU charge: {updateResponse.RequestCharge}");
```

### Upsert Item

```csharp
// Create if doesn't exist, replace if exists
var product = new Product
{
    Id = "product-1",
    Category = "electronics",
    Name = "Laptop",
    Price = 899.99m
};

ItemResponse<Product> response = await container.UpsertItemAsync(
    product,
    new PartitionKey(product.Category)
);

Console.WriteLine($"Upserted item. RU charge: {response.RequestCharge}");
```

### Delete Item

```csharp
ItemResponse<Product> response = await container.DeleteItemAsync<Product>(
    id: "product-1",
    partitionKey: new PartitionKey("electronics")
);

Console.WriteLine($"Deleted item. RU charge: {response.RequestCharge}");
Console.WriteLine($"Status: {response.StatusCode}");  // 204 No Content
```

### Patch Item (Partial Update)

```csharp
// Patch operations - More efficient than replace
List<PatchOperation> patchOperations = new List<PatchOperation>
{
    PatchOperation.Replace("/price", 799.99m),
    PatchOperation.Add("/discounted", true),
    PatchOperation.Remove("/oldField")
};

ItemResponse<Product> response = await container.PatchItemAsync<Product>(
    id: "product-1",
    partitionKey: new PartitionKey("electronics"),
    patchOperations: patchOperations
);

Console.WriteLine($"Patched item. RU charge: {response.RequestCharge}");
```

## Query Operations

### Simple Query

```csharp
// Query with LINQ
var query = container.GetItemLinqQueryable<Product>()
    .Where(p => p.Category == "electronics" && p.Price < 1000);

// Execute query
FeedIterator<Product> iterator = query.ToFeedIterator();

while (iterator.HasMoreResults)
{
    FeedResponse<Product> response = await iterator.ReadNextAsync();
    
    foreach (Product product in response)
    {
        Console.WriteLine($"{product.Name}: ${product.Price}");
    }
    
    Console.WriteLine($"RU charge: {response.RequestCharge}");
}
```

### SQL Query

```csharp
// SQL query
var query = new QueryDefinition(
    "SELECT * FROM products p WHERE p.category = @category AND p.price < @maxPrice"
)
.WithParameter("@category", "electronics")
.WithParameter("@maxPrice", 1000);

FeedIterator<Product> iterator = container.GetItemQueryIterator<Product>(
    query,
    requestOptions: new QueryRequestOptions
    {
        PartitionKey = new PartitionKey("electronics"),
        MaxItemCount = 10
    }
);

List<Product> products = new List<Product>();

while (iterator.HasMoreResults)
{
    FeedResponse<Product> response = await iterator.ReadNextAsync();
    products.AddRange(response);
    
    Console.WriteLine($"Page RU charge: {response.RequestCharge}");
}
```

### Cross-Partition Query

```csharp
// Query across all partitions (more expensive)
var query = new QueryDefinition(
    "SELECT * FROM products p WHERE p.price < @maxPrice"
)
.WithParameter("@maxPrice", 100);

// No partition key specified - scans all partitions
FeedIterator<Product> iterator = container.GetItemQueryIterator<Product>(
    query,
    requestOptions: new QueryRequestOptions
    {
        MaxItemCount = 100,
        MaxConcurrency = -1  // Unlimited parallelism
    }
);

while (iterator.HasMoreResults)
{
    FeedResponse<Product> response = await iterator.ReadNextAsync();
    Console.WriteLine($"RU charge: {response.RequestCharge}");  // Higher cost
}
```

### Aggregation Query

```csharp
// COUNT
var countQuery = new QueryDefinition(
    "SELECT VALUE COUNT(1) FROM products p WHERE p.category = @category"
)
.WithParameter("@category", "electronics");

FeedIterator<int> countIterator = container.GetItemQueryIterator<int>(countQuery);
FeedResponse<int> countResponse = await countIterator.ReadNextAsync();
int count = countResponse.First();

// GROUP BY
var groupQuery = new QueryDefinition(
    "SELECT p.category, COUNT(1) AS count, AVG(p.price) AS avgPrice " +
    "FROM products p " +
    "GROUP BY p.category"
);

FeedIterator<dynamic> groupIterator = container.GetItemQueryIterator<dynamic>(groupQuery);
while (groupIterator.HasMoreResults)
{
    FeedResponse<dynamic> response = await groupIterator.ReadNextAsync();
    foreach (var item in response)
    {
        Console.WriteLine($"{item.category}: {item.count} items, avg ${item.avgPrice}");
    }
}
```

## Batch Operations

### Transactional Batch

```csharp
// All operations within same partition key
var partitionKey = new PartitionKey("electronics");

TransactionalBatch batch = container.CreateTransactionalBatch(partitionKey);

batch.CreateItem(new Product { Id = "p1", Category = "electronics", Name = "Laptop" });
batch.CreateItem(new Product { Id = "p2", Category = "electronics", Name = "Mouse" });
batch.ReplaceItem("p3", new Product { Id = "p3", Category = "electronics", Name = "Keyboard" });
batch.DeleteItem("p4");

// Execute - all succeed or all fail (transactional)
TransactionalBatchResponse response = await batch.ExecuteAsync();

if (response.IsSuccessStatusCode)
{
    Console.WriteLine($"Batch succeeded. RU charge: {response.RequestCharge}");
}
else
{
    Console.WriteLine($"Batch failed: {response.StatusCode}");
}
```

### Bulk Operations

```csharp
// Enable bulk execution in client options
var clientOptions = new CosmosClientOptions
{
    AllowBulkExecution = true
};

CosmosClient client = new CosmosClient(endpoint, key, clientOptions);
Container container = client.GetContainer("mydb", "products");

// Create many items concurrently
List<Task> tasks = new List<Task>();

for (int i = 0; i < 1000; i++)
{
    var product = new Product
    {
        Id = $"product-{i}",
        Category = $"category-{i % 10}",
        Name = $"Product {i}",
        Price = i * 10
    };
    
    tasks.Add(container.CreateItemAsync(product, new PartitionKey(product.Category)));
}

await Task.WhenAll(tasks);
Console.WriteLine("Bulk insert completed");
```

## Error Handling

### Common Status Codes

```csharp
try
{
    ItemResponse<Product> response = await container.ReadItemAsync<Product>(
        "product-1",
        new PartitionKey("electronics")
    );
}
catch (CosmosException ex)
{
    Console.WriteLine($"Status code: {ex.StatusCode}");
    Console.WriteLine($"Message: {ex.Message}");
    Console.WriteLine($"RU charge: {ex.RequestCharge}");
    
    switch (ex.StatusCode)
    {
        case System.Net.HttpStatusCode.NotFound:  // 404
            Console.WriteLine("Item not found");
            break;
        case System.Net.HttpStatusCode.Conflict:  // 409
            Console.WriteLine("Item already exists");
            break;
        case (System.Net.HttpStatusCode)429:  // 429 Too Many Requests
            Console.WriteLine("Throttled - too many requests");
            Console.WriteLine($"Retry after: {ex.RetryAfter}");
            break;
        default:
            throw;
    }
}
```

## Best Practices

### 1. Singleton Client

```csharp
// Register in DI container
services.AddSingleton<CosmosClient>(serviceProvider =>
{
    return new CosmosClient(endpoint, key, new CosmosClientOptions
    {
        ApplicationName = "MyApp",
        ConnectionMode = ConnectionMode.Direct,
        MaxRetryAttemptsOnRateLimitedRequests = 9
    });
});
```

### 2. Use Async/Await Properly

```csharp
// ✅ Good: Await asynchronously
public async Task<Product> GetProductAsync(string id)
{
    return await container.ReadItemAsync<Product>(id, partitionKey);
}

// ❌ Bad: Blocking on async operation
public Product GetProduct(string id)
{
    return container.ReadItemAsync<Product>(id, partitionKey).Result;  // Deadlock risk!
}
```

### 3. Partition Key in Queries

```csharp
// ✅ Good: Include partition key
var query = new QueryDefinition("SELECT * FROM c WHERE c.category = @cat")
    .WithParameter("@cat", "electronics");

var iterator = container.GetItemQueryIterator<Product>(
    query,
    requestOptions: new QueryRequestOptions
    {
        PartitionKey = new PartitionKey("electronics")  // Efficient!
    }
);

// ❌ Bad: Cross-partition query
var badQuery = new QueryDefinition("SELECT * FROM c WHERE c.name = @name")
    .WithParameter("@name", "Laptop");
// Scans all partitions - expensive!
```

### 4. Use Point Reads When Possible

```csharp
// ✅ Best: Point read (1 RU for 1KB)
var response = await container.ReadItemAsync<Product>(id, partitionKey);

// ❌ Inefficient: Query for single item
var query = new QueryDefinition("SELECT * FROM c WHERE c.id = @id")
    .WithParameter("@id", id);
// More RUs than point read
```

## Critical Notes
- 💡 **CosmosClient** - Thread-safe singleton, reuse across application
- 🎯 **Async operations** - All methods return Task, use async/await
- ✅ **CreateIfNotExists** - Prefer over Create (idempotent)
- ⚠️ **Point read** - Most efficient (id + partition key = 1 RU for 1KB)
- 🔄 **Queries** - Use partition key in filter for efficiency
- 📊 **Batch operations** - Transactional within same partition key
- 💡 **Bulk execution** - Enable for high-throughput scenarios
- ✅ **Error handling** - Catch CosmosException for status codes
- ⚠️ **Connection mode** - Direct is default and faster than Gateway
- 🔒 **Resource hierarchy** - Client → Database → Container → Item

## Exam Tips
- CosmosClient: Thread-safe, singleton pattern, reuse per application lifetime
- NuGet package: Microsoft.Azure.Cosmos (v3)
- Generic terms: Container (not collection), Item (not document)
- CreateIfNotExistsAsync: Idempotent, preferred over CreateAsync
- Point read: ReadItemAsync with id + partition key (most efficient)
- Query: GetItemQueryIterator, returns FeedIterator
- Partition key: Include in QueryRequestOptions for efficiency
- Transactional batch: All operations same partition key, all succeed or fail
- Bulk execution: Enable with AllowBulkExecution = true
- CosmosException: Catch for error handling, check StatusCode
- Common status codes: 404 (Not Found), 409 (Conflict), 429 (Throttled)
- Retry logic: SDK handles 429 automatically with MaxRetryAttempts
- Connection modes: Direct (default, better performance), Gateway
- Request charge: Available in response.RequestCharge (RU consumed)
- Async pattern: All operations async, use await, never block with .Result
- Patch operations: PatchItemAsync for partial updates (more efficient than replace)
- LINQ queries: GetItemLinqQueryable for type-safe queries
- SQL queries: QueryDefinition with parameterized values
- Client options: Configure consistency, timeouts, retry policy at client level

[Learn More](https://learn.microsoft.com/en-us/training/modules/work-with-cosmos-db/2-cosmos-db-dotnet-overview)
