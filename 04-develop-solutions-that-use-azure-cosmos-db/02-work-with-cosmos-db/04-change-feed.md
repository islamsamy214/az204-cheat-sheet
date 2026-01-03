# Azure Cosmos DB Change Feed

## Key Concepts
- **Persistent log** - Record of all changes to container
- **Push model** - Azure Functions or Change Feed Processor
- **Pull model** - Manual processing with SDK
- **Time-ordered** - Changes in order within partition

## What is Change Feed?

**Persistent, ordered record of changes** to Azure Cosmos DB container:

- **Captures**: Creates and updates (not deletes by default)
- **Order**: Time-ordered within each partition key
- **Persistent**: Retained based on container TTL
- **Processing**: Push (automated) or Pull (manual) models
- **Use cases**: Real-time processing, data synchronization, event sourcing

### Key Characteristics

| Feature | Description |
|---------|-------------|
| **Scope** | Container-level (monitors one container) |
| **Operations** | Inserts and updates (deletes optional with change feed mode) |
| **Order** | Guaranteed within partition key |
| **Cross-partition** | No ordering guarantee across partitions |
| **Retention** | Based on container TTL or manual checkpoint |
| **Starting point** | Beginning, now, or specific point in time |
| **Processing** | Real-time or batch |

### Change Feed Modes

```csharp
// Latest version mode (default) - captures creates and updates
ChangeFeedMode.LatestVersion

// All versions and deletes mode - captures all changes including deletes
ChangeFeedMode.AllVersionsAndDeletes
```

## Push Model

### Option 1: Azure Functions Trigger

**Easiest way** - Automatic processing with Azure Functions:

```csharp
using Microsoft.Azure.Documents;
using Microsoft.Azure.WebJobs;
using Microsoft.Extensions.Logging;
using System.Collections.Generic;

public static class ChangeFeedFunction
{
    [FunctionName("ProcessChangeFeed")]
    public static void Run(
        [CosmosDBTrigger(
            databaseName: "myDatabase",
            collectionName: "myContainer",
            ConnectionStringSetting = "CosmosDBConnection",
            LeaseCollectionName = "leases",
            CreateLeaseCollectionIfNotExists = true)]
        IReadOnlyList<Document> documents,
        ILogger log)
    {
        if (documents != null && documents.Count > 0)
        {
            log.LogInformation($"Processing {documents.Count} documents");
            
            foreach (var doc in documents)
            {
                log.LogInformation($"Document ID: {doc.Id}");
                
                // Process each changed document
                ProcessDocument(doc);
            }
        }
    }
    
    private static void ProcessDocument(Document doc)
    {
        // Your processing logic
        // - Update search index
        // - Send notification
        // - Trigger workflow
        // - Synchronize data
    }
}
```

**Function configuration** (local.settings.json):

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet",
    "CosmosDBConnection": "AccountEndpoint=https://...;AccountKey=..."
  }
}
```

**Key benefits**:
- ✅ Zero infrastructure management
- ✅ Automatic scaling
- ✅ Built-in retry logic
- ✅ Checkpoint management handled
- ✅ Easy deployment

### Option 2: Change Feed Processor

**Programmatic processing** with full control:

#### Four Components

1. **Monitored container** - Source of changes
2. **Lease container** - Stores processing state (checkpoints)
3. **Compute instance** - Host that runs processor
4. **Delegate** - Your code that processes changes

#### Basic Implementation

```csharp
using Microsoft.Azure.Cosmos;
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

public class ChangeFeedProcessorExample
{
    private CosmosClient cosmosClient;
    private Container monitoredContainer;
    private Container leaseContainer;
    private ChangeFeedProcessor changeFeedProcessor;
    
    public async Task StartAsync()
    {
        // Initialize Cosmos Client
        cosmosClient = new CosmosClient(
            "AccountEndpoint=https://...;AccountKey=..."
        );
        
        // Get containers
        monitoredContainer = cosmosClient
            .GetContainer("myDatabase", "myContainer");
        
        leaseContainer = cosmosClient
            .GetContainer("myDatabase", "leases");
        
        // Build change feed processor
        changeFeedProcessor = monitoredContainer
            .GetChangeFeedProcessorBuilder<Product>(
                processorName: "productProcessor",
                onChangesDelegate: HandleChangesAsync)
            .WithInstanceName("consoleApp-instance1")
            .WithLeaseContainer(leaseContainer)
            .WithStartTime(DateTime.UtcNow.AddHours(-1))  // Start from 1 hour ago
            .Build();
        
        // Start processing
        await changeFeedProcessor.StartAsync();
        
        Console.WriteLine("Change feed processor started. Press any key to stop...");
        Console.ReadKey();
        
        // Stop processing
        await changeFeedProcessor.StopAsync();
    }
    
    // Delegate that processes changes
    static async Task HandleChangesAsync(
        ChangeFeedProcessorContext context,
        IReadOnlyCollection<Product> changes,
        CancellationToken cancellationToken)
    {
        Console.WriteLine($"Processing {changes.Count} changes...");
        
        foreach (var product in changes)
        {
            Console.WriteLine($"Product changed: {product.id}");
            Console.WriteLine($"  Name: {product.name}");
            Console.WriteLine($"  Price: {product.price}");
            
            // Process the change
            await ProcessProductChangeAsync(product);
        }
    }
    
    static async Task ProcessProductChangeAsync(Product product)
    {
        // Your business logic
        // - Update search index
        // - Send to event hub
        // - Update cache
        // - Trigger notification
        
        await Task.CompletedTask;
    }
}

public class Product
{
    public string id { get; set; }
    public string name { get; set; }
    public decimal price { get; set; }
    public string category { get; set; }
}
```

#### Configuration Options

```csharp
changeFeedProcessor = container
    .GetChangeFeedProcessorBuilder<MyDocument>(
        processorName: "myProcessor",
        onChangesDelegate: HandleChangesAsync)
    
    // Lease configuration
    .WithInstanceName("instance-1")                    // Unique instance identifier
    .WithLeaseContainer(leaseContainer)                // Container for checkpoints
    
    // Starting position
    .WithStartTime(DateTime.UtcNow.AddDays(-1))       // Start from 1 day ago
    // OR
    .WithStartTime(DateTime.MinValue)                  // Start from beginning
    
    // Polling configuration
    .WithPollInterval(TimeSpan.FromSeconds(5))         // Check for changes every 5s
    
    // Batch size
    .WithMaxItems(100)                                 // Max items per batch
    
    // Error handling
    .WithErrorNotification((leaseToken, exception) =>
    {
        Console.WriteLine($"Error on lease {leaseToken}: {exception.Message}");
        return Task.CompletedTask;
    })
    
    .Build();
```

#### Multiple Processors (Scale Out)

```csharp
// Instance 1
var processor1 = container
    .GetChangeFeedProcessorBuilder<Product>("productProcessor", HandleChangesAsync)
    .WithInstanceName("instance-1")  // Unique name
    .WithLeaseContainer(leaseContainer)
    .Build();

// Instance 2
var processor2 = container
    .GetChangeFeedProcessorBuilder<Product>("productProcessor", HandleChangesAsync)
    .WithInstanceName("instance-2")  // Unique name
    .WithLeaseContainer(leaseContainer)
    .Build();

// Both process different partitions automatically
await processor1.StartAsync();
await processor2.StartAsync();
```

### Lease Container

**Stores processing state** for each partition:

```csharp
// Create lease container
Database database = cosmosClient.GetDatabase("myDatabase");

ContainerProperties leaseContainerProperties = new ContainerProperties
{
    Id = "leases",
    PartitionKeyPath = "/id"
};

Container leaseContainer = await database.CreateContainerIfNotExistsAsync(
    leaseContainerProperties,
    throughput: 400  // Manual throughput (can be low)
);
```

**Lease document structure**:

```json
{
  "id": "myContainer.lease.0",
  "LeaseToken": "0",
  "Owner": "instance-1",
  "ContinuationToken": "\"8400022\"",
  "Timestamp": "2024-01-15T10:30:00Z"
}
```

**Key points**:
- One lease document per partition
- Tracks which instance owns which partition
- Stores continuation token (checkpoint)
- Enables automatic partition rebalancing

## Pull Model

### Manual Processing with SDK

**Full control** over when and how to process changes:

```csharp
using Microsoft.Azure.Cosmos;
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

public class PullModelExample
{
    public async Task ProcessChangesManuallyAsync()
    {
        CosmosClient client = new CosmosClient(
            "AccountEndpoint=https://...;AccountKey=..."
        );
        
        Container container = client.GetContainer("myDatabase", "myContainer");
        
        // Option 1: Process all partitions
        await ProcessAllPartitionsAsync(container);
        
        // Option 2: Process specific partition
        await ProcessSinglePartitionAsync(container, "electronics");
    }
    
    async Task ProcessAllPartitionsAsync(Container container)
    {
        // Get change feed iterator for all partitions
        FeedIterator<Product> iterator = container
            .GetChangeFeedIterator<Product>(
                ChangeFeedStartFrom.Beginning(),
                ChangeFeedMode.LatestVersion
            );
        
        while (iterator.HasMoreResults)
        {
            FeedResponse<Product> response = await iterator.ReadNextAsync();
            
            // Check if there are changes
            if (response.StatusCode == System.Net.HttpStatusCode.NotModified)
            {
                Console.WriteLine("No new changes. Waiting...");
                await Task.Delay(TimeSpan.FromSeconds(5));
                continue;
            }
            
            // Process changes
            Console.WriteLine($"Processing {response.Count} changes");
            
            foreach (var product in response)
            {
                Console.WriteLine($"Changed: {product.id} - {product.name}");
                await ProcessChangeAsync(product);
            }
            
            // Save continuation token for resuming later
            string continuationToken = response.ContinuationToken;
            await SaveCheckpointAsync(continuationToken);
        }
    }
    
    async Task ProcessSinglePartitionAsync(Container container, string partitionKey)
    {
        // Get change feed iterator for specific partition
        FeedIterator<Product> iterator = container
            .GetChangeFeedIterator<Product>(
                ChangeFeedStartFrom.Now(
                    FeedRange.FromPartitionKey(new PartitionKey(partitionKey))
                ),
                ChangeFeedMode.LatestVersion
            );
        
        while (iterator.HasMoreResults)
        {
            FeedResponse<Product> response = await iterator.ReadNextAsync();
            
            if (response.StatusCode != System.Net.HttpStatusCode.NotModified)
            {
                foreach (var product in response)
                {
                    Console.WriteLine($"Product in {partitionKey}: {product.id}");
                }
            }
            else
            {
                await Task.Delay(TimeSpan.FromSeconds(5));
            }
        }
    }
    
    async Task ProcessChangeAsync(Product product)
    {
        // Your processing logic
        await Task.CompletedTask;
    }
    
    async Task SaveCheckpointAsync(string continuationToken)
    {
        // Save to persistent storage (database, file, etc.)
        // For resuming processing later
        await Task.CompletedTask;
    }
}
```

### Starting Points

```csharp
// 1. From beginning (all historical changes)
ChangeFeedStartFrom.Beginning()

// 2. From now (only new changes)
ChangeFeedStartFrom.Now()

// 3. From specific time
ChangeFeedStartFrom.Time(DateTime.UtcNow.AddDays(-7))

// 4. Resume from checkpoint (continuation token)
ChangeFeedStartFrom.ContinuationToken(savedToken)

// 5. Specific partition from beginning
ChangeFeedStartFrom.Beginning(
    FeedRange.FromPartitionKey(new PartitionKey("electronics"))
)
```

### Pull Model with Checkpointing

```csharp
public class ManualCheckpointingExample
{
    private string checkpointFile = "checkpoint.txt";
    
    public async Task ProcessWithCheckpointAsync()
    {
        CosmosClient client = new CosmosClient("...");
        Container container = client.GetContainer("myDatabase", "myContainer");
        
        // Load last checkpoint
        string continuationToken = await LoadCheckpointAsync();
        
        FeedIterator<Product> iterator;
        
        if (string.IsNullOrEmpty(continuationToken))
        {
            // Start from beginning
            iterator = container.GetChangeFeedIterator<Product>(
                ChangeFeedStartFrom.Beginning(),
                ChangeFeedMode.LatestVersion
            );
        }
        else
        {
            // Resume from checkpoint
            iterator = container.GetChangeFeedIterator<Product>(
                ChangeFeedStartFrom.ContinuationToken(continuationToken),
                ChangeFeedMode.LatestVersion
            );
        }
        
        while (iterator.HasMoreResults)
        {
            try
            {
                FeedResponse<Product> response = await iterator.ReadNextAsync();
                
                if (response.StatusCode == System.Net.HttpStatusCode.NotModified)
                {
                    await Task.Delay(TimeSpan.FromSeconds(5));
                    continue;
                }
                
                // Process changes
                foreach (var product in response)
                {
                    await ProcessChangeAsync(product);
                }
                
                // Save checkpoint after successful processing
                await SaveCheckpointAsync(response.ContinuationToken);
                
                Console.WriteLine($"Processed {response.Count} items. RU: {response.RequestCharge}");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error processing changes: {ex.Message}");
                // Retry or handle error
            }
        }
    }
    
    async Task<string> LoadCheckpointAsync()
    {
        if (File.Exists(checkpointFile))
        {
            return await File.ReadAllTextAsync(checkpointFile);
        }
        return null;
    }
    
    async Task SaveCheckpointAsync(string token)
    {
        await File.WriteAllTextAsync(checkpointFile, token);
    }
    
    async Task ProcessChangeAsync(Product product)
    {
        // Your processing logic
        await Task.CompletedTask;
    }
}
```

## Change Feed Modes

### Latest Version Mode (Default)

**Captures creates and updates**:

```csharp
FeedIterator<Product> iterator = container
    .GetChangeFeedIterator<Product>(
        ChangeFeedStartFrom.Beginning(),
        ChangeFeedMode.LatestVersion  // Default mode
    );

// Returns latest version of each changed document
// Does NOT capture deletes
```

### All Versions and Deletes Mode

**Captures all operations including deletes**:

```csharp
FeedIterator<ChangeFeedItem<Product>> iterator = container
    .GetChangeFeedIterator<ChangeFeedItem<Product>>(
        ChangeFeedStartFrom.Beginning(),
        ChangeFeedMode.AllVersionsAndDeletes  // Includes deletes
    );

while (iterator.HasMoreResults)
{
    FeedResponse<ChangeFeedItem<Product>> response = await iterator.ReadNextAsync();
    
    foreach (var item in response)
    {
        if (item.Metadata.OperationType == ChangeFeedOperationType.Create)
        {
            Console.WriteLine($"Created: {item.Current.id}");
        }
        else if (item.Metadata.OperationType == ChangeFeedOperationType.Replace)
        {
            Console.WriteLine($"Updated: {item.Current.id}");
            // item.Previous contains previous version
        }
        else if (item.Metadata.OperationType == ChangeFeedOperationType.Delete)
        {
            Console.WriteLine($"Deleted: {item.Previous.id}");
            // item.Current is null for deletes
        }
    }
}
```

## Common Use Cases

### 1. Real-Time Search Index Updates

```csharp
static async Task HandleChangesAsync(
    ChangeFeedProcessorContext context,
    IReadOnlyCollection<Product> changes,
    CancellationToken cancellationToken)
{
    var searchClient = new SearchClient("...");
    
    foreach (var product in changes)
    {
        // Update search index in real-time
        await searchClient.IndexDocumentAsync(new
        {
            id = product.id,
            name = product.name,
            description = product.description,
            category = product.category,
            price = product.price
        });
    }
}
```

### 2. Data Synchronization

```csharp
static async Task SyncToSqlDatabase(
    ChangeFeedProcessorContext context,
    IReadOnlyCollection<Order> changes,
    CancellationToken cancellationToken)
{
    using var connection = new SqlConnection("...");
    await connection.OpenAsync();
    
    foreach (var order in changes)
    {
        // Sync to SQL database
        var command = new SqlCommand(
            "MERGE INTO Orders ...",
            connection
        );
        
        await command.ExecuteNonQueryAsync();
    }
}
```

### 3. Event-Driven Workflows

```csharp
static async Task TriggerWorkflow(
    ChangeFeedProcessorContext context,
    IReadOnlyCollection<Order> changes,
    CancellationToken cancellationToken)
{
    var eventHubClient = new EventHubProducerClient("...");
    
    foreach (var order in changes)
    {
        if (order.status == "paid")
        {
            // Trigger fulfillment workflow
            await eventHubClient.SendAsync(new EventData(
                JsonSerializer.Serialize(new
                {
                    orderId = order.id,
                    action = "fulfill"
                })
            ));
        }
    }
}
```

### 4. Cache Invalidation

```csharp
static async Task InvalidateCache(
    ChangeFeedProcessorContext context,
    IReadOnlyCollection<Product> changes,
    CancellationToken cancellationToken)
{
    var cache = ConnectionMultiplexer.Connect("redis-connection");
    var db = cache.GetDatabase();
    
    foreach (var product in changes)
    {
        // Invalidate cache entry
        await db.KeyDeleteAsync($"product:{product.id}");
        await db.KeyDeleteAsync($"category:{product.category}");
    }
}
```

### 5. Analytics and Reporting

```csharp
static async Task UpdateAnalytics(
    ChangeFeedProcessorContext context,
    IReadOnlyCollection<Sale> changes,
    CancellationToken cancellationToken)
{
    var analyticsClient = new AnalyticsClient("...");
    
    foreach (var sale in changes)
    {
        // Update real-time analytics
        await analyticsClient.TrackEventAsync("sale", new Dictionary<string, string>
        {
            ["productId"] = sale.productId,
            ["amount"] = sale.amount.ToString(),
            ["category"] = sale.category
        });
    }
}
```

## Push vs Pull Model Comparison

| Aspect | Push Model (Processor/Functions) | Pull Model (Manual) |
|--------|----------------------------------|---------------------|
| **Ease of use** | Easy, automated | More complex |
| **Control** | Less control | Full control |
| **Checkpointing** | Automatic | Manual |
| **Scaling** | Automatic | Manual |
| **Best for** | Real-time processing | Batch processing, custom logic |
| **Infrastructure** | Managed | Self-managed |
| **Error handling** | Built-in retry | Custom |
| **Starting point** | Configurable | Full flexibility |

## Error Handling

### In Change Feed Processor

```csharp
changeFeedProcessor = container
    .GetChangeFeedProcessorBuilder<Product>("processor", HandleChangesAsync)
    .WithInstanceName("instance-1")
    .WithLeaseContainer(leaseContainer)
    .WithErrorNotification(async (leaseToken, exception) =>
    {
        // Log error
        Console.WriteLine($"Error on partition {leaseToken}");
        Console.WriteLine($"Exception: {exception.Message}");
        
        // Custom error handling
        // - Send alert
        // - Log to monitoring system
        // - Implement retry logic
        
        await Task.CompletedTask;
    })
    .Build();
```

### In Delegate

```csharp
static async Task HandleChangesAsync(
    ChangeFeedProcessorContext context,
    IReadOnlyCollection<Product> changes,
    CancellationToken cancellationToken)
{
    foreach (var product in changes)
    {
        try
        {
            await ProcessProductAsync(product);
        }
        catch (Exception ex)
        {
            // Handle error for individual item
            Console.WriteLine($"Error processing {product.id}: {ex.Message}");
            
            // Options:
            // 1. Continue to next item
            // 2. Save to dead-letter queue
            // 3. Retry with backoff
            // 4. Throw to mark entire batch as failed
        }
    }
}
```

### In Pull Model

```csharp
while (iterator.HasMoreResults)
{
    try
    {
        FeedResponse<Product> response = await iterator.ReadNextAsync();
        
        // Process changes
        foreach (var product in response)
        {
            await ProcessAsync(product);
        }
        
        // Save checkpoint
        await SaveCheckpointAsync(response.ContinuationToken);
    }
    catch (CosmosException ex) when (ex.StatusCode == System.Net.HttpStatusCode.TooManyRequests)
    {
        // Rate limited - wait and retry
        Console.WriteLine("Rate limited. Waiting...");
        await Task.Delay(TimeSpan.FromSeconds(ex.RetryAfter?.TotalSeconds ?? 5));
    }
    catch (Exception ex)
    {
        // Other errors
        Console.WriteLine($"Error: {ex.Message}");
        // Implement retry logic or dead-letter handling
    }
}
```

## Performance Considerations

### 1. Batch Size

```csharp
// Larger batches = fewer round trips but more memory
changeFeedProcessor = container
    .GetChangeFeedProcessorBuilder<Product>("processor", HandleChangesAsync)
    .WithMaxItems(1000)  // Process up to 1000 items per batch
    .Build();
```

### 2. Polling Interval

```csharp
// Balance between latency and RU consumption
changeFeedProcessor = container
    .GetChangeFeedProcessorBuilder<Product>("processor", HandleChangesAsync)
    .WithPollInterval(TimeSpan.FromSeconds(5))  // Check every 5 seconds
    .Build();
```

### 3. Parallel Processing

```csharp
// Scale out with multiple instances
// Each instance processes different partitions automatically
var tasks = new List<Task>();

for (int i = 0; i < 5; i++)
{
    var processor = container
        .GetChangeFeedProcessorBuilder<Product>("processor", HandleChangesAsync)
        .WithInstanceName($"instance-{i}")
        .WithLeaseContainer(leaseContainer)
        .Build();
    
    tasks.Add(processor.StartAsync());
}

await Task.WhenAll(tasks);
```

## Best Practices

### 1. Use Change Feed Processor for Most Cases

```csharp
// ✅ Recommended: Change Feed Processor for real-time scenarios
// - Automatic checkpoint management
// - Built-in error handling
// - Automatic scaling
```

### 2. Idempotent Processing

```csharp
static async Task HandleChangesAsync(
    ChangeFeedProcessorContext context,
    IReadOnlyCollection<Product> changes,
    CancellationToken cancellationToken)
{
    // Make processing idempotent
    // Same change processed multiple times = same result
    
    foreach (var product in changes)
    {
        // Check if already processed
        if (await IsAlreadyProcessedAsync(product.id, product._etag))
        {
            continue;
        }
        
        await ProcessAsync(product);
        await MarkAsProcessedAsync(product.id, product._etag);
    }
}
```

### 3. Separate Lease Container

```csharp
// ✅ Good: Dedicated lease container
Container leaseContainer = database.GetContainer("leases");

// ❌ Bad: Using monitored container for leases
// Can cause performance issues
```

### 4. Handle NotModified Status

```csharp
FeedResponse<Product> response = await iterator.ReadNextAsync();

if (response.StatusCode == System.Net.HttpStatusCode.NotModified)
{
    // No new changes - wait before checking again
    await Task.Delay(TimeSpan.FromSeconds(5));
    continue;
}
```

### 5. Monitor RU Consumption

```csharp
FeedResponse<Product> response = await iterator.ReadNextAsync();

Console.WriteLine($"RU consumed: {response.RequestCharge}");

// Adjust polling frequency based on RU consumption
```

## Critical Notes
- 💡 **Persistent log** - Ordered record of all changes to container
- 🎯 **Operations** - Captures creates and updates (deletes optional)
- ✅ **Time-ordered** - Changes ordered within each partition key
- ⚠️ **No cross-partition order** - No ordering guarantee across partitions
- 🔄 **Push model** - Azure Functions or Change Feed Processor (automated)
- 📊 **Pull model** - Manual processing with SDK (full control)
- 💡 **Lease container** - Stores checkpoints and partition ownership
- ✅ **Scale out** - Multiple instances process different partitions
- ⚠️ **Starting point** - Beginning, Now, Time, or ContinuationToken
- 🔒 **Checkpointing** - Save continuation token to resume processing
- 🎯 **Change feed modes** - LatestVersion (default) or AllVersionsAndDeletes
- 💡 **Azure Functions** - Easiest with CosmosDBTrigger
- ⚠️ **Polling interval** - Balance between latency and RU consumption
- ✅ **Idempotent** - Design processing to handle duplicates
- 🔄 **Error handling** - WithErrorNotification for processor-level errors

## Exam Tips
- Change feed: Persistent, time-ordered log of changes to container
- Captures: Creates and updates (deletes with AllVersionsAndDeletes mode)
- Ordering: Guaranteed within partition key, not across partitions
- Push model: Azure Functions (CosmosDBTrigger) or Change Feed Processor
- Pull model: Manual processing with GetChangeFeedIterator
- Components: Monitored container, lease container, compute instance, delegate
- Lease container: Stores checkpoints and partition ownership
- Scaling: Multiple processor instances automatically partition workload
- Starting points: Beginning(), Now(), Time(), ContinuationToken()
- Azure Functions: Easiest option with automatic scaling
- Change Feed Processor: GetChangeFeedProcessorBuilder()
- Delegate: HandleChangesAsync receives batch of changes
- Configuration: WithInstanceName, WithLeaseContainer, WithPollInterval, WithMaxItems
- Pull model iterator: GetChangeFeedIterator<T>()
- Status codes: NotModified = no new changes
- Checkpointing: ContinuationToken for resuming processing
- Change feed modes: LatestVersion (default), AllVersionsAndDeletes
- Error handling: WithErrorNotification() for processor errors
- Best practice: Use Change Feed Processor for most scenarios
- Idempotent: Design processing to handle duplicate messages
- RU consumption: Monitor RequestCharge, adjust polling frequency
- Use cases: Real-time indexing, data sync, event-driven workflows, cache invalidation

[Learn More](https://learn.microsoft.com/en-us/training/modules/work-with-cosmos-db/6-cosmos-db-change-feed)
