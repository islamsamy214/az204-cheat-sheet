# Scale Event Processing Application

## Scaling Challenges with Event Streams

### The Problem

Consider a scenario with **100,000 homes** sending sensor data to Event Hubs:

**Requirements:**
- ✅ **Scale**: Handle varying numbers of consumers dynamically
- ✅ **Load Balance**: Distribute partitions among consumers evenly
- ✅ **Fault Tolerance**: Resume processing after consumer failures
- ✅ **Efficiency**: Process events without duplication or data loss
- ✅ **Checkpointing**: Track processing progress per partition

**Challenge:**
How do you coordinate multiple consumer instances to efficiently process events from multiple partitions without conflicts?

**Solution:**
Use **EventProcessorClient** (or **EventHubConsumerClient** with manual coordination)

---

## Partitioned Consumer Pattern

### Traditional Pattern: Competing Consumers

```
Queue (Single Stream)
├── Event 1
├── Event 2
├── Event 3        ┌──────────────┐
├── Event 4  ───→  │  Consumer 1  │ (reads any available event)
├── Event 5        └──────────────┘
├── Event 6        ┌──────────────┐
├── Event 7  ───→  │  Consumer 2  │ (reads any available event)
└── Event 8        └──────────────┘

Problem: Bottleneck at queue, limited scalability
```

### Event Hubs Pattern: Partitioned Consumers

```
Event Hub (Multiple Partitions)
├── Partition 0  ───→  ┌──────────────┐
├── Partition 1  ───→  │  Consumer 1  │ (owns P0, P1)
├── Partition 2  ───→  └──────────────┘
├── Partition 3        ┌──────────────┐
├── Partition 4  ───→  │  Consumer 2  │ (owns P2, P3, P4)
├── Partition 5  ───→  └──────────────┘
└── Partition 6        ┌──────────────┐
    Partition 7  ───→  │  Consumer 3  │ (owns P6, P7)
                       └──────────────┘

Benefit: Parallel processing, high scalability
```

**Key Principles:**
1. Each **partition** is assigned to **one consumer** at a time
2. A **consumer** can own **multiple partitions**
3. Partition ownership is **dynamically distributed**
4. When consumers join/leave, partitions are **rebalanced**

---

## EventProcessorClient Architecture

### Component Overview

```
┌──────────────────────────────────────────────────────────────┐
│                    CONSUMER GROUP                             │
│                                                               │
│  ┌──────────────────────┐         ┌──────────────────────┐  │
│  │ Consumer Instance 1  │         │ Consumer Instance 2  │  │
│  │                      │         │                      │  │
│  │ EventProcessorClient │         │ EventProcessorClient │  │
│  │  • Partition 0       │         │  • Partition 2       │  │
│  │  • Partition 1       │         │  • Partition 3       │  │
│  └──────────┬───────────┘         └──────────┬───────────┘  │
│             │                                 │              │
│             │     ┌──────────────────────┐   │              │
│             │     │ Consumer Instance 3  │   │              │
│             │     │                      │   │              │
│             │     │ EventProcessorClient │   │              │
│             │     │  • Partition 4       │   │              │
│             │     │  • Partition 5       │   │              │
│             │     └──────────┬───────────┘   │              │
│             │                │                │              │
└─────────────┼────────────────┼────────────────┼──────────────┘
              │                │                │
              ▼                ▼                ▼
    ┌─────────────────────────────────────────────────────┐
    │         CHECKPOINT STORE (Blob Storage)             │
    │                                                     │
    │  Partition Ownership:                               │
    │  ├── Partition 0 → Instance 1 (lease expires: ...)│
    │  ├── Partition 1 → Instance 1 (lease expires: ...)│
    │  ├── Partition 2 → Instance 2 (lease expires: ...)│
    │  ├── Partition 3 → Instance 2 (lease expires: ...)│
    │  ├── Partition 4 → Instance 3 (lease expires: ...)│
    │  └── Partition 5 → Instance 3 (lease expires: ...)│
    │                                                     │
    │  Checkpoints (per partition):                       │
    │  ├── Partition 0: Offset 12345, Sequence 67890    │
    │  ├── Partition 1: Offset 23456, Sequence 78901    │
    │  ├── Partition 2: Offset 34567, Sequence 89012    │
    │  └── ...                                           │
    └─────────────────────────────────────────────────────┘
              ▲                ▲                ▲
              │                │                │
              │ Read/Write     │ Read/Write     │ Read/Write
              │ Ownership      │ Ownership      │ Ownership
              │ Checkpoints    │ Checkpoints    │ Checkpoints
              │                │                │
              └────────────────┴────────────────┘
```

### How EventProcessorClient Works

**1. Partition Ownership Claim:**
- Each EventProcessorClient instance claims ownership of partitions
- Ownership tracked via **blob leases** in Azure Blob Storage
- Lease duration: 15-60 seconds (configurable)
- Lease renewal: Every few seconds to maintain ownership

**2. Load Balancing:**
- Continuously monitors partition distribution
- Rebalances when:
  - New consumer instance starts
  - Existing consumer instance stops
  - Lease renewal fails (instance crashed)

**3. Checkpointing:**
- Consumer periodically saves processing progress
- Checkpoint includes:
  - Partition ID
  - Offset (position in partition)
  - Sequence number
- Stored in checkpoint store (Blob Storage)

**4. Failure Recovery:**
- When consumer crashes, lease expires
- Another consumer claims ownership
- Resumes from last checkpoint
- Prevents duplicate processing (if checkpointed correctly)

---

## EventProcessorClient Implementation

### .NET Implementation

**Install NuGet Package:**
```bash
dotnet add package Azure.Messaging.EventHubs
dotnet add package Azure.Messaging.EventHubs.Processor
dotnet add package Azure.Storage.Blobs
```

**Basic Implementation:**

```csharp
using Azure.Messaging.EventHubs;
using Azure.Messaging.EventHubs.Consumer;
using Azure.Messaging.EventHubs.Processor;
using Azure.Storage.Blobs;
using System.Text;

// Connection strings
string eventHubsConnectionString = "<event-hubs-connection-string>";
string eventHubName = "telemetry";
string consumerGroup = EventHubConsumerClient.DefaultConsumerGroupName;

// Checkpoint store
string blobStorageConnectionString = "<storage-connection-string>";
string blobContainerName = "checkpoints";

// Create blob container client for checkpoint store
BlobContainerClient storageClient = new BlobContainerClient(
    blobStorageConnectionString,
    blobContainerName
);

// Create the container if it doesn't exist
await storageClient.CreateIfNotExistsAsync();

// Create event processor client
EventProcessorClient processor = new EventProcessorClient(
    storageClient,
    consumerGroup,
    eventHubsConnectionString,
    eventHubName
);

// Register event handler
processor.ProcessEventAsync += async (ProcessEventArgs args) =>
{
    try
    {
        // Access event data
        string partition = args.Partition.PartitionId;
        byte[] eventBody = args.Data.EventBody.ToArray();
        string bodyText = Encoding.UTF8.GetString(eventBody);
        
        Console.WriteLine($"Partition: {partition}");
        Console.WriteLine($"Offset: {args.Data.Offset}");
        Console.WriteLine($"Sequence: {args.Data.SequenceNumber}");
        Console.WriteLine($"Event: {bodyText}");
        Console.WriteLine("---");
        
        // Process event (your business logic)
        await ProcessEventAsync(bodyText);
        
        // Update checkpoint
        await args.UpdateCheckpointAsync();
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error processing event: {ex.Message}");
        // Don't checkpoint on error
    }
};

// Register error handler
processor.ProcessErrorAsync += (ProcessErrorEventArgs args) =>
{
    Console.WriteLine($"Error in partition {args.PartitionId}: {args.Exception.Message}");
    return Task.CompletedTask;
};

// Start processing
await processor.StartProcessingAsync();
Console.WriteLine("Event processor started. Press Enter to stop.");
Console.ReadLine();

// Stop processing
await processor.StopProcessingAsync();
Console.WriteLine("Event processor stopped.");

async Task ProcessEventAsync(string eventData)
{
    // Your business logic here
    // Example: Parse JSON, save to database, send alert, etc.
    await Task.Delay(10); // Simulate processing
}
```

**Advanced Configuration:**

```csharp
var options = new EventProcessorClientOptions
{
    // Maximum wait time for events
    MaximumWaitTime = TimeSpan.FromSeconds(30),
    
    // Track last enqueued event properties
    TrackLastEnqueuedEventProperties = true,
    
    // Retry options
    RetryOptions = new EventHubsRetryOptions
    {
        MaximumRetries = 5,
        Delay = TimeSpan.FromSeconds(1),
        MaximumDelay = TimeSpan.FromSeconds(30),
        Mode = EventHubsRetryMode.Exponential
    },
    
    // Connection options
    ConnectionOptions = new EventHubConnectionOptions
    {
        TransportType = EventHubsTransportType.AmqpTcp
    }
};

EventProcessorClient processor = new EventProcessorClient(
    storageClient,
    consumerGroup,
    eventHubsConnectionString,
    eventHubName,
    options
);
```

### Python Implementation

**Install Package:**
```bash
pip install azure-eventhub
pip install azure-eventhub-checkpointstoreblob
pip install azure-storage-blob
```

**Implementation:**

```python
from azure.eventhub import EventHubConsumerClient
from azure.eventhub.extensions.checkpointstoreblobaio import BlobCheckpointStore
import asyncio

# Connection strings
connection_string = "<event-hubs-connection-string>"
eventhub_name = "telemetry"
consumer_group = "$Default"

# Checkpoint store
storage_connection_string = "<storage-connection-string>"
container_name = "checkpoints"

async def on_event(partition_context, event):
    """Process event"""
    try:
        # Access event data
        partition_id = partition_context.partition_id
        offset = event.offset
        sequence_number = event.sequence_number
        body = event.body_as_str()
        
        print(f"Partition: {partition_id}")
        print(f"Offset: {offset}")
        print(f"Sequence: {sequence_number}")
        print(f"Event: {body}")
        print("---")
        
        # Process event (your business logic)
        await process_event(body)
        
        # Update checkpoint
        await partition_context.update_checkpoint(event)
        
    except Exception as e:
        print(f"Error processing event: {e}")
        # Don't checkpoint on error

async def on_error(partition_context, error):
    """Handle errors"""
    if partition_context:
        print(f"Error in partition {partition_context.partition_id}: {error}")
    else:
        print(f"Error: {error}")

async def process_event(event_data):
    """Your business logic"""
    await asyncio.sleep(0.01)  # Simulate processing

async def main():
    # Create checkpoint store
    checkpoint_store = BlobCheckpointStore.from_connection_string(
        storage_connection_string,
        container_name
    )
    
    # Create consumer client
    client = EventHubConsumerClient.from_connection_string(
        connection_string,
        consumer_group,
        eventhub_name=eventhub_name,
        checkpoint_store=checkpoint_store
    )
    
    async with client:
        # Start processing
        await client.receive(
            on_event=on_event,
            on_error=on_error
        )

if __name__ == "__main__":
    asyncio.run(main())
```

### JavaScript/TypeScript Implementation

**Install Packages:**
```bash
npm install @azure/event-hubs
npm install @azure/storage-blob
```

**Implementation:**

```javascript
const { EventHubConsumerClient } = require("@azure/event-hubs");
const { ContainerClient } = require("@azure/storage-blob");
const { BlobCheckpointStore } = require("@azure/eventhubs-checkpointstore-blob");

// Connection strings
const connectionString = "<event-hubs-connection-string>";
const eventHubName = "telemetry";
const consumerGroup = "$Default";

// Checkpoint store
const storageConnectionString = "<storage-connection-string>";
const containerName = "checkpoints";

async function main() {
    // Create checkpoint store
    const containerClient = new ContainerClient(
        storageConnectionString,
        containerName
    );
    
    await containerClient.createIfNotExists();
    
    const checkpointStore = new BlobCheckpointStore(containerClient);
    
    // Create consumer client
    const consumerClient = new EventHubConsumerClient(
        consumerGroup,
        connectionString,
        eventHubName,
        checkpointStore
    );
    
    // Subscribe to events
    const subscription = consumerClient.subscribe({
        processEvents: async (events, context) => {
            for (const event of events) {
                try {
                    console.log(`Partition: ${context.partitionId}`);
                    console.log(`Offset: ${event.offset}`);
                    console.log(`Sequence: ${event.sequenceNumber}`);
                    console.log(`Event: ${event.body}`);
                    console.log("---");
                    
                    // Process event
                    await processEvent(event.body);
                    
                    // Update checkpoint
                    await context.updateCheckpoint(event);
                } catch (err) {
                    console.error(`Error processing event: ${err}`);
                }
            }
        },
        processError: async (err, context) => {
            console.error(`Error in partition ${context.partitionId}: ${err}`);
        }
    });
    
    // Wait for termination signal
    console.log("Event processor started. Press Ctrl+C to stop.");
    await new Promise(() => {});
}

async function processEvent(eventData) {
    // Your business logic
    await new Promise(resolve => setTimeout(resolve, 10));
}

main().catch((err) => {
    console.error("Error:", err);
});
```

---

## Partition Ownership and Load Balancing

### Partition Ownership Tracking

**Storage Structure (Blob Storage):**

```
Container: "checkpoints"
├── <eventhub-name>/
│   ├── <consumer-group>/
│   │   ├── ownership/
│   │   │   ├── 0  ← Partition 0 ownership (blob lease)
│   │   │   ├── 1  ← Partition 1 ownership
│   │   │   ├── 2  ← Partition 2 ownership
│   │   │   └── 3  ← Partition 3 ownership
│   │   └── checkpoint/
│   │       ├── 0  ← Partition 0 checkpoint
│   │       ├── 1  ← Partition 1 checkpoint
│   │       ├── 2  ← Partition 2 checkpoint
│   │       └── 3  ← Partition 3 checkpoint
```

**Ownership Blob Content (JSON):**

```json
{
  "ownerIdentifier": "consumer-instance-1",
  "partitionId": "0",
  "eventHubName": "telemetry",
  "consumerGroup": "$Default",
  "fullyQualifiedNamespace": "myeventhubns.servicebus.windows.net",
  "lastModifiedTime": "2024-01-15T10:30:00Z",
  "eTag": "0x8D9E5F7A8B3C1D2"
}
```

**Checkpoint Blob Content (JSON):**

```json
{
  "partitionId": "0",
  "offset": "12345",
  "sequenceNumber": 67890,
  "eventHubName": "telemetry",
  "consumerGroup": "$Default",
  "fullyQualifiedNamespace": "myeventhubns.servicebus.windows.net"
}
```

### Load Balancing Example

**Scenario: 8 Partitions, Consumer Instances Scale**

```
Time T0: 1 Consumer Instance
┌────────────────────────┐
│   Consumer Instance 1  │
│   Owns: P0-P7 (all 8)  │
└────────────────────────┘

Time T1: 2 Consumer Instances (scale out)
┌────────────────────────┐  ┌────────────────────────┐
│   Consumer Instance 1  │  │   Consumer Instance 2  │
│   Owns: P0-P3 (4)      │  │   Owns: P4-P7 (4)      │
└────────────────────────┘  └────────────────────────┘
         ▲                           ▲
         └───── Load Balanced ───────┘

Time T2: 4 Consumer Instances (scale out further)
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Instance 1  │  │  Instance 2  │  │  Instance 3  │  │  Instance 4  │
│  Owns: P0-P1 │  │  Owns: P2-P3 │  │  Owns: P4-P5 │  │  Owns: P6-P7 │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘

Time T3: 1 Consumer Instance (scale in - Instance 2, 3, 4 stopped)
┌────────────────────────┐
│   Consumer Instance 1  │
│   Owns: P0-P7 (all 8)  │
└────────────────────────┘
         ▲
         └─── Takes over all partitions
```

**Load Balancing Algorithm:**

1. **Ownership Claim Cycle** (every 10-30 seconds):
   - Count total partitions
   - Count active consumers
   - Calculate fair share: `partitions / consumers`
   
2. **Partition Distribution:**
   - If consumer owns < fair share: claim more partitions
   - If consumer owns > fair share: release partitions
   - Balance achieved over multiple cycles

3. **Lease Management:**
   - Lease duration: 15-60 seconds
   - Renewal interval: Every few seconds
   - Expired lease: Partition available for claiming

---

## Checkpointing Strategies

### When to Checkpoint?

| Strategy | Frequency | Pros | Cons | Use Case |
|----------|-----------|------|------|----------|
| **Every Event** | After each event | Maximum fault tolerance | High overhead, slow | Critical financial transactions |
| **Every Batch** | After processing batch | Good balance | Some reprocessing risk | Standard applications |
| **Time-Based** | Every N seconds | Predictable | Potential data loss | High-throughput scenarios |
| **Count-Based** | Every N events | Consistent | Variable time | Moderate throughput |
| **Hybrid** | Batch + time | Best balance | More complex | Production recommended |

### Checkpointing Examples

**Example 1: Checkpoint Every Batch (Recommended)**

```csharp
processor.ProcessEventAsync += async (ProcessEventArgs args) =>
{
    try
    {
        await ProcessEventAsync(args.Data);
        
        // Checkpoint after each event (simple but overhead)
        await args.UpdateCheckpointAsync();
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error: {ex.Message}");
        // Don't checkpoint on error
    }
};
```

**Example 2: Checkpoint Every N Events**

```csharp
private static int eventCount = 0;
private static readonly int checkpointFrequency = 100;

processor.ProcessEventAsync += async (ProcessEventArgs args) =>
{
    try
    {
        await ProcessEventAsync(args.Data);
        
        // Increment counter
        Interlocked.Increment(ref eventCount);
        
        // Checkpoint every 100 events
        if (eventCount % checkpointFrequency == 0)
        {
            await args.UpdateCheckpointAsync();
            Console.WriteLine($"Checkpointed at event {eventCount}");
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error: {ex.Message}");
    }
};
```

**Example 3: Checkpoint Every N Seconds**

```csharp
private static DateTime lastCheckpoint = DateTime.UtcNow;
private static readonly TimeSpan checkpointInterval = TimeSpan.FromSeconds(30);

processor.ProcessEventAsync += async (ProcessEventArgs args) =>
{
    try
    {
        await ProcessEventAsync(args.Data);
        
        // Checkpoint every 30 seconds
        if (DateTime.UtcNow - lastCheckpoint >= checkpointInterval)
        {
            await args.UpdateCheckpointAsync();
            lastCheckpoint = DateTime.UtcNow;
            Console.WriteLine($"Checkpointed at {lastCheckpoint}");
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error: {ex.Message}");
    }
};
```

**Example 4: Hybrid Strategy (Best Practice)**

```csharp
private static int eventCount = 0;
private static DateTime lastCheckpoint = DateTime.UtcNow;
private static readonly int checkpointEventCount = 100;
private static readonly TimeSpan checkpointInterval = TimeSpan.FromSeconds(30);

processor.ProcessEventAsync += async (ProcessEventArgs args) =>
{
    try
    {
        await ProcessEventAsync(args.Data);
        
        Interlocked.Increment(ref eventCount);
        
        // Checkpoint if either condition met
        bool shouldCheckpoint = 
            eventCount >= checkpointEventCount ||
            DateTime.UtcNow - lastCheckpoint >= checkpointInterval;
        
        if (shouldCheckpoint)
        {
            await args.UpdateCheckpointAsync();
            eventCount = 0;
            lastCheckpoint = DateTime.UtcNow;
            Console.WriteLine($"Checkpointed: Events={eventCount}, Time={lastCheckpoint}");
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error: {ex.Message}");
    }
};
```

---

## Thread Safety and Concurrency

### EventProcessorClient Thread Safety

**Key Guarantees:**

1. **Sequential per Partition**: Events from same partition processed sequentially
2. **Concurrent across Partitions**: Different partitions processed in parallel
3. **Thread-Safe**: EventProcessorClient is thread-safe

**Processing Model:**

```
EventProcessorClient
├── Partition 0 → Thread 1 (sequential: E1 → E2 → E3)
├── Partition 1 → Thread 2 (sequential: E4 → E5 → E6)
├── Partition 2 → Thread 3 (sequential: E7 → E8 → E9)
└── Partition 3 → Thread 4 (sequential: E10 → E11 → E12)

Thread 1, 2, 3, 4 run concurrently (parallel processing)
Within each thread, events processed sequentially
```

**Implications:**

✅ **Safe**: Share EventProcessorClient across threads
✅ **Ordering**: Events in same partition maintain order
✅ **Parallelism**: Maximum parallelism = number of partitions
❌ **Blocking**: Slow processing in one partition doesn't block others

### Handling Concurrent Processing

**Bad Practice: Blocking Processing**

```csharp
// DON'T DO THIS - Blocks processing
processor.ProcessEventAsync += async (ProcessEventArgs args) =>
{
    Thread.Sleep(5000);  // Blocks thread for 5 seconds!
    await ProcessEventAsync(args.Data);
    await args.UpdateCheckpointAsync();
};
```

**Good Practice: Async Processing**

```csharp
// DO THIS - Non-blocking async processing
processor.ProcessEventAsync += async (ProcessEventArgs args) =>
{
    await Task.Delay(5000);  // Async delay, doesn't block
    await ProcessEventAsync(args.Data);
    await args.UpdateCheckpointAsync();
};
```

**Advanced: Parallel Processing within Partition (Use Carefully)**

```csharp
processor.ProcessEventAsync += async (ProcessEventArgs args) =>
{
    // Process event in background task
    _ = Task.Run(async () =>
    {
        try
        {
            await ProcessEventAsync(args.Data);
            // Note: Cannot checkpoint here (lost context)
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error: {ex.Message}");
        }
    });
    
    // Immediately return (fast acknowledgment)
    // Warning: Events may be processed out of order!
};
```

⚠️ **Warning**: Parallel processing within partition breaks ordering guarantees!

---

## Best Practices

### Performance Optimization

1. **Batch Checkpointing**
   - Checkpoint every 50-100 events or 30 seconds
   - Reduces storage operations
   - Balance fault tolerance vs performance

2. **Async Processing**
   - Use async/await throughout
   - Don't block threads
   - Leverage I/O concurrency

3. **Resource Management**
   - Reuse connections and clients
   - Dispose resources properly
   - Use connection pooling

4. **Partition Count**
   - More partitions = more parallelism
   - Match partition count to expected consumer instances
   - Plan for future scale

### Fault Tolerance

1. **Error Handling**
   - Catch exceptions in event handler
   - Don't checkpoint on error (allows retry)
   - Log errors for debugging
   - Implement dead-letter queue for poison messages

2. **Graceful Shutdown**
   ```csharp
   // Graceful shutdown
   await processor.StopProcessingAsync();
   ```

3. **Idempotency**
   - Design processing to be idempotent
   - Handle duplicate events gracefully
   - Use unique identifiers to detect duplicates

4. **Monitoring**
   - Track checkpoint lag
   - Monitor partition distribution
   - Alert on processing errors

### Scalability

1. **Horizontal Scaling**
   - Add more consumer instances
   - Automatic load balancing
   - Maximum instances = number of partitions

2. **Vertical Scaling**
   - Increase CPU/memory per instance
   - Optimize processing logic
   - Use faster storage for checkpoints

3. **Autoscaling**
   - Scale based on consumer lag
   - Use Azure Container Instances or AKS
   - KEDA for event-driven autoscaling

---

## Exam Tips for AZ-204

### Key Concepts to Remember

1. **EventProcessorClient** = production-recommended client (automatic load balancing)
2. **Checkpoint Store** = Azure Blob Storage (tracks progress and ownership)
3. **Partition Ownership** = One partition owned by one consumer at a time
4. **Load Balancing** = Automatic distribution of partitions among consumers
5. **Checkpointing** = Marking processing progress (offset + sequence number)
6. **Fault Tolerance** = Resume from checkpoint after failure
7. **Thread Safety** = Sequential per partition, concurrent across partitions

### Common Exam Scenarios

**Scenario 1**: Scale event processing dynamically
- ✅ Use EventProcessorClient
- ✅ Deploy multiple instances
- ✅ Automatic load balancing

**Scenario 2**: Track processing progress
- ✅ Use checkpointing (UpdateCheckpointAsync)
- ✅ Requires Blob Storage
- ✅ Resume from checkpoint on failure

**Scenario 3**: Maximize parallelism
- ✅ Match consumer instances to partition count
- ✅ Example: 16 partitions = up to 16 consumers

**Scenario 4**: Ensure event ordering
- ✅ Use partition key (events in same partition maintain order)
- ❌ No ordering guarantee across partitions

### Remember for Exam

- **EventProcessorClient**: Recommended for production
- **EventHubConsumerClient**: For prototyping only (no automatic load balancing)
- **Checkpoint store**: Requires Azure Blob Storage
- **Checkpointing**: Call UpdateCheckpointAsync() periodically
- **Partition ownership**: Tracked via blob leases (15-60 second duration)
- **Load balancing**: Automatic, rebalances every 10-30 seconds
- **Maximum consumers**: Limited by partition count
- **Thread safety**: Sequential per partition, parallel across partitions

### Quick Reference

```csharp
// EventProcessorClient setup
var storageClient = new BlobContainerClient(storageConnString, containerName);
var processor = new EventProcessorClient(storageClient, consumerGroup, ehConnString, ehName);

// Event handler
processor.ProcessEventAsync += async (args) => {
    await ProcessEvent(args.Data);
    await args.UpdateCheckpointAsync();  // Checkpoint
};

// Error handler
processor.ProcessErrorAsync += (args) => {
    Console.WriteLine($"Error: {args.Exception.Message}");
    return Task.CompletedTask;
};

// Start/stop
await processor.StartProcessingAsync();
await processor.StopProcessingAsync();
```

---

## Summary

**Scaling event processing** requires coordination among multiple consumer instances to efficiently process events from multiple partitions.

**Key Components:**
- **EventProcessorClient**: Automatic load balancing and fault tolerance
- **Checkpoint Store**: Azure Blob Storage for progress tracking
- **Partition Ownership**: One partition per consumer (dynamic distribution)
- **Checkpointing**: Track processing progress (offset + sequence)

**Benefits:**
- Horizontal scalability (add/remove consumers dynamically)
- Fault tolerance (resume from checkpoint)
- Load balancing (automatic partition distribution)
- Concurrency (parallel processing across partitions)

**Best Practices:**
- Checkpoint periodically (not every event)
- Handle errors gracefully (don't checkpoint on error)
- Design for idempotency (handle duplicates)
- Monitor partition distribution and lag