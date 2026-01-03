# Event Hubs Client Library

## Azure Event Hubs Client Library Overview

The **Azure Event Hubs client library** (`Azure.Messaging.EventHubs`) provides APIs for sending and receiving events.

### Key Client Types

| Client Type | Purpose | Use Case |
|------------|---------|----------|
| **EventHubProducerClient** | Send events | Publish telemetry, logs, events |
| **EventHubConsumerClient** | Read events | Prototyping, simple scenarios |
| **EventProcessorClient** | Process events at scale | Production applications |

### NuGet Packages

```bash
# Core Event Hubs client library
dotnet add package Azure.Messaging.EventHubs

# Event Processor (includes checkpoint store)
dotnet add package Azure.Messaging.EventHubs.Processor

# Azure Storage Blobs (for checkpoint store)
dotnet add package Azure.Storage.Blobs
```

### Python Packages

```bash
pip install azure-eventhub
pip install azure-eventhub-checkpointstoreblob
pip install azure-storage-blob
```

### JavaScript/TypeScript Packages

```bash
npm install @azure/event-hubs
npm install @azure/eventhubs-checkpointstore-blob
npm install @azure/storage-blob
```

---

## Inspect Event Hub

### Get Partition Information

**C# - Get Partition IDs:**

```csharp
using Azure.Messaging.EventHubs;
using Azure.Messaging.EventHubs.Producer;

string connectionString = "<connection-string>";
string eventHubName = "myeventhub";

await using var producer = new EventHubProducerClient(connectionString, eventHubName);

// Get Event Hub properties
EventHubProperties properties = await producer.GetEventHubPropertiesAsync();

Console.WriteLine($"Event Hub Name: {properties.Name}");
Console.WriteLine($"Created At: {properties.CreatedOn}");
Console.WriteLine($"Partition Count: {properties.PartitionIds.Length}");
Console.WriteLine($"Partitions: {string.Join(", ", properties.PartitionIds)}");
```

**Output:**
```
Event Hub Name: myeventhub
Created At: 2024-01-15 10:00:00Z
Partition Count: 4
Partitions: 0, 1, 2, 3
```

**C# - Get Partition Properties:**

```csharp
// Get properties for each partition
foreach (string partitionId in properties.PartitionIds)
{
    PartitionProperties partitionProperties = await producer.GetPartitionPropertiesAsync(partitionId);
    
    Console.WriteLine($"\nPartition {partitionId}:");
    Console.WriteLine($"  Beginning Sequence Number: {partitionProperties.BeginningSequenceNumber}");
    Console.WriteLine($"  Last Sequence Number: {partitionProperties.LastEnqueuedSequenceNumber}");
    Console.WriteLine($"  Last Offset: {partitionProperties.LastEnqueuedOffset}");
    Console.WriteLine($"  Last Enqueued Time: {partitionProperties.LastEnqueuedTime}");
    Console.WriteLine($"  Is Empty: {partitionProperties.IsEmpty}");
}
```

**Python:**

```python
from azure.eventhub import EventHubProducerClient

connection_string = "<connection-string>"
eventhub_name = "myeventhub"

producer = EventHubProducerClient.from_connection_string(
    connection_string,
    eventhub_name=eventhub_name
)

# Get Event Hub properties
with producer:
    properties = producer.get_eventhub_properties()
    
    print(f"Event Hub Name: {properties['name']}")
    print(f"Partition Count: {len(properties['partition_ids'])}")
    print(f"Partitions: {properties['partition_ids']}")
    
    # Get partition properties
    for partition_id in properties['partition_ids']:
        partition_props = producer.get_partition_properties(partition_id)
        print(f"\nPartition {partition_id}:")
        print(f"  Last Sequence: {partition_props['last_enqueued_sequence_number']}")
        print(f"  Last Offset: {partition_props['last_enqueued_offset']}")
```

---

## EventHubProducerClient: Publish Events

### Basic Event Publishing

**C# - Send Single Event:**

```csharp
using Azure.Messaging.EventHubs;
using Azure.Messaging.EventHubs.Producer;
using System.Text;

string connectionString = "<connection-string>";
string eventHubName = "myeventhub";

await using var producer = new EventHubProducerClient(connectionString, eventHubName);

// Create event data
var eventData = new EventData(Encoding.UTF8.GetBytes("Hello Event Hubs!"));

// Send single event
await producer.SendAsync(new[] { eventData });
Console.WriteLine("Event sent successfully!");
```

### Batch Publishing (Recommended)

**C# - Send Batch of Events:**

```csharp
await using var producer = new EventHubProducerClient(connectionString, eventHubName);

// Create event batch
using EventDataBatch eventBatch = await producer.CreateBatchAsync();

// Add events to batch
for (int i = 0; i < 10; i++)
{
    string eventBody = $"Event {i}";
    var eventData = new EventData(Encoding.UTF8.GetBytes(eventBody));
    
    // Try to add event to batch
    if (!eventBatch.TryAdd(eventData))
    {
        // Batch is full, send it
        await producer.SendAsync(eventBatch);
        Console.WriteLine($"Batch sent with {eventBatch.Count} events");
        
        // Create new batch and add current event
        eventBatch = await producer.CreateBatchAsync();
        
        if (!eventBatch.TryAdd(eventData))
        {
            throw new Exception($"Event {i} is too large for an empty batch");
        }
    }
}

// Send remaining events
if (eventBatch.Count > 0)
{
    await producer.SendAsync(eventBatch);
    Console.WriteLine($"Final batch sent with {eventBatch.Count} events");
}
```

**Why Use Batching?**
- ✅ **Performance**: Reduces network overhead
- ✅ **Efficiency**: Maximizes throughput
- ✅ **Cost**: Fewer network calls
- ✅ **Size Limit**: Automatic handling of batch size limits (1 MB)

### Event Properties

**C# - Add Custom Properties:**

```csharp
var eventData = new EventData(Encoding.UTF8.GetBytes(jsonData));

// Add application properties
eventData.Properties["DeviceId"] = "device-001";
eventData.Properties["Temperature"] = 72.5;
eventData.Properties["Timestamp"] = DateTime.UtcNow;
eventData.Properties["Location"] = "Building-A";

// Add content type (for schema identification)
eventData.ContentType = "application/json";

// Add message ID (for deduplication)
eventData.MessageId = Guid.NewGuid().ToString();

// Add correlation ID (for request tracking)
eventData.CorrelationId = correlationId;

await producer.SendAsync(new[] { eventData });
```

### Partition Key (Related Events)

**C# - Send with Partition Key:**

```csharp
// Events with same partition key go to same partition
string partitionKey = "device-001";

var eventData = new EventData(Encoding.UTF8.GetBytes(jsonData));

var batchOptions = new CreateBatchOptions
{
    PartitionKey = partitionKey  // All events in batch use this key
};

using EventDataBatch eventBatch = await producer.CreateBatchAsync(batchOptions);
eventBatch.TryAdd(eventData);

await producer.SendAsync(eventBatch);
```

**Why Use Partition Key?**
- ✅ **Ordering**: Events with same key maintain order
- ✅ **Grouping**: Related events processed together
- ✅ **Distribution**: Hash-based distribution across partitions

**Example Use Cases:**
- Device ID: All telemetry from device-001 in same partition
- User ID: All user actions in same partition
- Session ID: All session events in same partition

### Send to Specific Partition

**C# - Target Specific Partition:**

```csharp
// Send directly to partition 2
var batchOptions = new CreateBatchOptions
{
    PartitionId = "2"  // Explicit partition assignment
};

using EventDataBatch eventBatch = await producer.CreateBatchAsync(batchOptions);

for (int i = 0; i < 10; i++)
{
    var eventData = new EventData(Encoding.UTF8.GetBytes($"Event {i} for partition 2"));
    eventBatch.TryAdd(eventData);
}

await producer.SendAsync(eventBatch);
Console.WriteLine("Events sent to partition 2");
```

⚠️ **Caution**: Explicit partition assignment can lead to unbalanced load. Use partition key instead.

### Python Examples

**Python - Send Events:**

```python
from azure.eventhub import EventHubProducerClient, EventData

connection_string = "<connection-string>"
eventhub_name = "myeventhub"

producer = EventHubProducerClient.from_connection_string(
    connection_string,
    eventhub_name=eventhub_name
)

# Create batch
with producer:
    event_data_batch = producer.create_batch()
    
    for i in range(10):
        event_data = EventData(f"Event {i}")
        
        # Add custom properties
        event_data.properties = {
            "DeviceId": "device-001",
            "Index": i
        }
        
        try:
            event_data_batch.add(event_data)
        except ValueError:
            # Batch full, send it
            producer.send_batch(event_data_batch)
            event_data_batch = producer.create_batch()
            event_data_batch.add(event_data)
    
    # Send remaining events
    if len(event_data_batch) > 0:
        producer.send_batch(event_data_batch)
```

**Python - Partition Key:**

```python
# Create batch with partition key
event_data_batch = producer.create_batch(partition_key="device-001")

for i in range(10):
    event_data = EventData(f"Event {i}")
    event_data_batch.add(event_data)

producer.send_batch(event_data_batch)
```

### JavaScript Examples

**JavaScript - Send Events:**

```javascript
const { EventHubProducerClient } = require("@azure/event-hubs");

const connectionString = "<connection-string>";
const eventHubName = "myeventhub";

async function main() {
    const producer = new EventHubProducerClient(connectionString, eventHubName);
    
    // Create batch
    const batch = await producer.createBatch();
    
    for (let i = 0; i < 10; i++) {
        const eventData = {
            body: `Event ${i}`,
            properties: {
                deviceId: "device-001",
                index: i
            }
        };
        
        const added = batch.tryAdd(eventData);
        
        if (!added) {
            // Batch full, send it
            await producer.sendBatch(batch);
            batch = await producer.createBatch();
            batch.tryAdd(eventData);
        }
    }
    
    // Send remaining events
    if (batch.count > 0) {
        await producer.sendBatch(batch);
    }
    
    await producer.close();
}

main().catch(console.error);
```

---

## EventHubConsumerClient: Read Events

### Read Events from All Partitions

**C# - Read Events (Iterator Pattern):**

```csharp
using Azure.Messaging.EventHubs.Consumer;

string connectionString = "<connection-string>";
string eventHubName = "myeventhub";
string consumerGroup = EventHubConsumerClient.DefaultConsumerGroupName;

await using var consumer = new EventHubConsumerClient(
    consumerGroup,
    connectionString,
    eventHubName
);

// Configure read options
var readOptions = new ReadEventOptions
{
    MaximumWaitTime = TimeSpan.FromSeconds(30)
};

// Read events (infinite loop)
await foreach (PartitionEvent partitionEvent in consumer.ReadEventsAsync(readOptions))
{
    Console.WriteLine($"Partition: {partitionEvent.Partition.PartitionId}");
    Console.WriteLine($"Offset: {partitionEvent.Data.Offset}");
    Console.WriteLine($"Sequence: {partitionEvent.Data.SequenceNumber}");
    Console.WriteLine($"Enqueued Time: {partitionEvent.Data.EnqueuedTime}");
    
    // Read body
    string body = Encoding.UTF8.GetString(partitionEvent.Data.EventBody.ToArray());
    Console.WriteLine($"Body: {body}");
    
    // Read custom properties
    foreach (var property in partitionEvent.Data.Properties)
    {
        Console.WriteLine($"  {property.Key}: {property.Value}");
    }
    
    Console.WriteLine("---");
}
```

**C# - Read with Cancellation:**

```csharp
using var cancellationSource = new CancellationTokenSource();
cancellationSource.CancelAfter(TimeSpan.FromMinutes(5));

try
{
    await foreach (PartitionEvent evt in consumer.ReadEventsAsync(cancellationSource.Token))
    {
        Console.WriteLine($"Event: {evt.Data.EventBody}");
    }
}
catch (TaskCanceledException)
{
    Console.WriteLine("Reading stopped after 5 minutes");
}
```

### Read Events from Specific Partition

**C# - Read from Partition:**

```csharp
string partitionId = "2";

// Start reading from beginning
EventPosition startingPosition = EventPosition.Earliest;

var readOptions = new ReadEventOptions
{
    MaximumWaitTime = TimeSpan.FromSeconds(10)
};

await foreach (PartitionEvent evt in consumer.ReadEventsFromPartitionAsync(
    partitionId,
    startingPosition,
    readOptions))
{
    Console.WriteLine($"Partition {partitionId}: {evt.Data.EventBody}");
}
```

### Event Position Strategies

**EventPosition Options:**

| Position | Description | Use Case |
|----------|-------------|----------|
| **Earliest** | Start from beginning | Read all historical events |
| **Latest** | Start from newest events | Real-time monitoring |
| **FromOffset** | Start from specific offset | Resume from known position |
| **FromSequenceNumber** | Start from sequence number | Precise resume point |
| **FromEnqueuedTime** | Start from timestamp | Time-based replay |

**C# - EventPosition Examples:**

```csharp
// Start from beginning
EventPosition position = EventPosition.Earliest;

// Start from latest (only new events)
EventPosition position = EventPosition.Latest;

// Start from specific offset
long offset = 12345;
EventPosition position = EventPosition.FromOffset(offset, isInclusive: false);

// Start from sequence number
long sequenceNumber = 67890;
EventPosition position = EventPosition.FromSequenceNumber(sequenceNumber, isInclusive: true);

// Start from timestamp (e.g., 1 hour ago)
DateTimeOffset enqueuedTime = DateTimeOffset.UtcNow.AddHours(-1);
EventPosition position = EventPosition.FromEnqueuedTime(enqueuedTime);
```

### Python Examples

**Python - Read Events:**

```python
from azure.eventhub import EventHubConsumerClient

connection_string = "<connection-string>"
eventhub_name = "myeventhub"
consumer_group = "$Default"

consumer = EventHubConsumerClient.from_connection_string(
    connection_string,
    consumer_group,
    eventhub_name=eventhub_name
)

def on_event(partition_context, event):
    print(f"Partition: {partition_context.partition_id}")
    print(f"Offset: {event.offset}")
    print(f"Sequence: {event.sequence_number}")
    print(f"Body: {event.body_as_str()}")
    print("---")

def on_error(partition_context, error):
    print(f"Error: {error}")

with consumer:
    consumer.receive(
        on_event=on_event,
        on_error=on_error,
        starting_position="-1"  # From beginning
    )
```

**Python - Read from Specific Partition:**

```python
def on_event_batch(partition_context, events):
    for event in events:
        print(f"Event: {event.body_as_str()}")

with consumer:
    consumer.receive_batch(
        on_event_batch=on_event_batch,
        partition_id="2",
        starting_position="-1"  # From beginning
    )
```

### JavaScript Examples

**JavaScript - Read Events:**

```javascript
const { EventHubConsumerClient } = require("@azure/event-hubs");

const connectionString = "<connection-string>";
const eventHubName = "myeventhub";
const consumerGroup = "$Default";

async function main() {
    const consumer = new EventHubConsumerClient(consumerGroup, connectionString, eventHubName);
    
    const subscription = consumer.subscribe({
        processEvents: async (events, context) => {
            for (const event of events) {
                console.log(`Partition: ${context.partitionId}`);
                console.log(`Offset: ${event.offset}`);
                console.log(`Body: ${event.body}`);
            }
        },
        processError: async (err, context) => {
            console.error(`Error: ${err.message}`);
        }
    }, {
        startPosition: EventPosition.earliest()
    });
    
    // Stop after 1 minute
    setTimeout(() => subscription.close(), 60000);
}

main().catch(console.error);
```

⚠️ **Important**: `EventHubConsumerClient` is for **prototyping only**. For production, use `EventProcessorClient`.

---

## EventProcessorClient: Production Processing

### Why EventProcessorClient?

| Feature | EventHubConsumerClient | EventProcessorClient |
|---------|----------------------|---------------------|
| **Load Balancing** | Manual | ✅ Automatic |
| **Checkpointing** | Manual | ✅ Built-in |
| **Fault Tolerance** | Manual | ✅ Automatic |
| **Scalability** | Limited | ✅ Horizontal |
| **Production Ready** | ❌ No | ✅ Yes |

### EventProcessorClient Setup

**C# - Complete Example:**

```csharp
using Azure.Messaging.EventHubs;
using Azure.Messaging.EventHubs.Consumer;
using Azure.Messaging.EventHubs.Processor;
using Azure.Storage.Blobs;
using System.Text;

// Event Hubs connection
string eventHubsConnectionString = "<event-hubs-connection-string>";
string eventHubName = "myeventhub";
string consumerGroup = EventHubConsumerClient.DefaultConsumerGroupName;

// Checkpoint store (Azure Blob Storage)
string blobStorageConnectionString = "<storage-connection-string>";
string blobContainerName = "checkpoints";

// Create blob container client
BlobContainerClient storageClient = new BlobContainerClient(
    blobStorageConnectionString,
    blobContainerName
);

// Create container if it doesn't exist
await storageClient.CreateIfNotExistsAsync();

// Create event processor client
EventProcessorClient processor = new EventProcessorClient(
    storageClient,
    consumerGroup,
    eventHubsConnectionString,
    eventHubName
);

// Configure processor options (optional)
var options = new EventProcessorClientOptions
{
    MaximumWaitTime = TimeSpan.FromSeconds(30),
    TrackLastEnqueuedEventProperties = true,
    LoadBalancingUpdateInterval = TimeSpan.FromSeconds(10),
    PartitionOwnershipExpirationInterval = TimeSpan.FromSeconds(30)
};

// Event handler (required)
processor.ProcessEventAsync += async (ProcessEventArgs args) =>
{
    try
    {
        // Process event
        string partition = args.Partition.PartitionId;
        long offset = args.Data.Offset;
        long sequence = args.Data.SequenceNumber;
        string body = Encoding.UTF8.GetString(args.Data.EventBody.ToArray());
        
        Console.WriteLine($"[{partition}] Offset={offset}, Seq={sequence}, Body={body}");
        
        // Your business logic here
        await ProcessEventAsync(body);
        
        // Update checkpoint (recommended frequency: every 50-100 events)
        await args.UpdateCheckpointAsync();
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error processing event: {ex.Message}");
        // Don't checkpoint on error
    }
};

// Error handler (required)
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
    // Your business logic
    await Task.Delay(10); // Simulate processing
}
```

### Event Handler Details

**ProcessEventArgs Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `Data` | EventData | The event being processed |
| `Partition` | PartitionContext | Partition information |
| `CancellationToken` | CancellationToken | Cancellation token |
| `UpdateCheckpointAsync()` | Method | Save checkpoint |

**PartitionContext Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `PartitionId` | string | Current partition ID |
| `FullyQualifiedNamespace` | string | Event Hubs namespace |
| `EventHubName` | string | Event Hub name |
| `ConsumerGroup` | string | Consumer group name |
| `ReadLastEnqueuedEventProperties()` | Method | Get partition stats |

### Checkpointing Strategies

**Strategy 1: Checkpoint Every Event (Simple)**

```csharp
processor.ProcessEventAsync += async (ProcessEventArgs args) =>
{
    await ProcessEventAsync(args.Data);
    await args.UpdateCheckpointAsync();  // Checkpoint immediately
};
```

**Strategy 2: Checkpoint Every N Events (Recommended)**

```csharp
private static int eventCount = 0;
private static readonly int checkpointInterval = 100;

processor.ProcessEventAsync += async (ProcessEventArgs args) =>
{
    await ProcessEventAsync(args.Data);
    
    if (Interlocked.Increment(ref eventCount) % checkpointInterval == 0)
    {
        await args.UpdateCheckpointAsync();
        Console.WriteLine($"Checkpointed at event {eventCount}");
    }
};
```

**Strategy 3: Checkpoint Every N Seconds**

```csharp
private static DateTime lastCheckpoint = DateTime.UtcNow;
private static readonly TimeSpan checkpointInterval = TimeSpan.FromSeconds(30);

processor.ProcessEventAsync += async (ProcessEventArgs args) =>
{
    await ProcessEventAsync(args.Data);
    
    if (DateTime.UtcNow - lastCheckpoint >= checkpointInterval)
    {
        await args.UpdateCheckpointAsync();
        lastCheckpoint = DateTime.UtcNow;
    }
};
```

### Python EventProcessorClient

**Python - Complete Example:**

```python
import asyncio
from azure.eventhub.aio import EventHubConsumerClient
from azure.eventhub.extensions.checkpointstoreblobaio import BlobCheckpointStore

connection_string = "<event-hubs-connection-string>"
eventhub_name = "myeventhub"
consumer_group = "$Default"

storage_connection_string = "<storage-connection-string>"
container_name = "checkpoints"

async def on_event(partition_context, event):
    try:
        print(f"Partition: {partition_context.partition_id}")
        print(f"Event: {event.body_as_str()}")
        
        # Process event
        await process_event(event.body_as_str())
        
        # Update checkpoint
        await partition_context.update_checkpoint(event)
    except Exception as e:
        print(f"Error: {e}")

async def on_error(partition_context, error):
    print(f"Error: {error}")

async def process_event(event_data):
    await asyncio.sleep(0.01)  # Simulate processing

async def main():
    checkpoint_store = BlobCheckpointStore.from_connection_string(
        storage_connection_string,
        container_name
    )
    
    client = EventHubConsumerClient.from_connection_string(
        connection_string,
        consumer_group,
        eventhub_name=eventhub_name,
        checkpoint_store=checkpoint_store
    )
    
    async with client:
        await client.receive(
            on_event=on_event,
            on_error=on_error
        )

if __name__ == "__main__":
    asyncio.run(main())
```

### JavaScript EventProcessorClient

**JavaScript - Complete Example:**

```javascript
const { EventHubConsumerClient } = require("@azure/event-hubs");
const { ContainerClient } = require("@azure/storage-blob");
const { BlobCheckpointStore } = require("@azure/eventhubs-checkpointstore-blob");

const connectionString = "<event-hubs-connection-string>";
const eventHubName = "myeventhub";
const consumerGroup = "$Default";

const storageConnectionString = "<storage-connection-string>";
const containerName = "checkpoints";

async function main() {
    const containerClient = new ContainerClient(storageConnectionString, containerName);
    await containerClient.createIfNotExists();
    
    const checkpointStore = new BlobCheckpointStore(containerClient);
    
    const consumerClient = new EventHubConsumerClient(
        consumerGroup,
        connectionString,
        eventHubName,
        checkpointStore
    );
    
    const subscription = consumerClient.subscribe({
        processEvents: async (events, context) => {
            for (const event of events) {
                console.log(`Partition: ${context.partitionId}`);
                console.log(`Event: ${event.body}`);
                
                // Process event
                await processEvent(event.body);
            }
            
            // Update checkpoint (after batch)
            if (events.length > 0) {
                await context.updateCheckpoint(events[events.length - 1]);
            }
        },
        processError: async (err, context) => {
            console.error(`Error: ${err.message}`);
        }
    });
    
    // Wait for termination
    await new Promise(() => {});
}

async function processEvent(eventData) {
    await new Promise(resolve => setTimeout(resolve, 10));
}

main().catch(console.error);
```

---

## Best Practices

### Performance Optimization

1. **Use Batching**: Always use `CreateBatchAsync()` for sending events
2. **Checkpoint Frequency**: Balance between fault tolerance and performance (every 50-100 events)
3. **Async Operations**: Use async/await throughout to avoid blocking
4. **Connection Reuse**: Reuse producer/consumer clients across multiple operations
5. **Partition Strategy**: Use partition key for related events, avoid explicit partition assignment

### Error Handling

1. **Transient Failures**: Implement retry logic with exponential backoff
2. **Permanent Failures**: Log and move to dead-letter queue
3. **Don't Checkpoint on Error**: Allows retry from last successful checkpoint

```csharp
processor.ProcessEventAsync += async (ProcessEventArgs args) =>
{
    try
    {
        await ProcessEventAsync(args.Data);
        await args.UpdateCheckpointAsync();
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error: {ex.Message}");
        // Don't checkpoint - event will be reprocessed
    }
};
```

### Resource Management

```csharp
// Always dispose resources
await using var producer = new EventHubProducerClient(connectionString, eventHubName);
// Use producer
// Automatically disposed at end of scope

// Or explicitly
var producer = new EventHubProducerClient(connectionString, eventHubName);
try
{
    // Use producer
}
finally
{
    await producer.DisposeAsync();
}
```

---

## Exam Tips for AZ-204

### Key Concepts to Remember

1. **EventHubProducerClient** = Send events (batching recommended)
2. **EventHubConsumerClient** = Read events (prototyping only)
3. **EventProcessorClient** = Production processing (automatic load balancing, checkpointing)
4. **Checkpoint Store** = Azure Blob Storage (required for EventProcessorClient)
5. **Batching** = Use `CreateBatchAsync()` for better performance
6. **Partition Key** = Group related events (maintain ordering)
7. **EventPosition** = Earliest, Latest, FromOffset, FromSequenceNumber, FromEnqueuedTime

### Common Exam Scenarios

**Scenario 1**: Send events to Event Hubs
- ✅ Use EventHubProducerClient
- ✅ Use batching (`CreateBatchAsync()`)
- ✅ Use partition key for related events

**Scenario 2**: Production event processing
- ✅ Use EventProcessorClient (not EventHubConsumerClient)
- ✅ Requires checkpoint store (Blob Storage)
- ✅ Automatic load balancing and fault tolerance

**Scenario 3**: Checkpoint processing progress
- ✅ Call `UpdateCheckpointAsync()` periodically
- ✅ Checkpoint every 50-100 events (not every event)
- ✅ Don't checkpoint on error

**Scenario 4**: Read events from beginning
- ✅ Use `EventPosition.Earliest`
- ✅ Or `EventPosition.FromEnqueuedTime(timestamp)`

### Remember for Exam

- **EventHubProducerClient**: Send events (use batching)
- **EventHubConsumerClient**: Prototyping only (no load balancing)
- **EventProcessorClient**: Production (requires Blob Storage)
- **Batching**: `CreateBatchAsync()` and `TryAdd()`
- **Partition Key**: Groups related events (ordering maintained)
- **Checkpointing**: `UpdateCheckpointAsync()` (every 50-100 events)
- **EventPosition**: Earliest, Latest, FromOffset, etc.

### Quick Reference

```csharp
// Send events (batching)
var producer = new EventHubProducerClient(connString, ehName);
using EventDataBatch batch = await producer.CreateBatchAsync();
batch.TryAdd(new EventData("event"));
await producer.SendAsync(batch);

// Read events (prototyping)
var consumer = new EventHubConsumerClient(group, connString, ehName);
await foreach (var evt in consumer.ReadEventsAsync())
{
    Console.WriteLine(evt.Data.EventBody);
}

// Process events (production)
var processor = new EventProcessorClient(storageClient, group, connString, ehName);
processor.ProcessEventAsync += async (args) => {
    await Process(args.Data);
    await args.UpdateCheckpointAsync();
};
await processor.StartProcessingAsync();
```

---

## Summary

**Event Hubs client library** provides three main client types for different scenarios.

**EventHubProducerClient:**
- Send events to Event Hubs
- Use batching for performance
- Partition key for related events

**EventHubConsumerClient:**
- Read events from Event Hubs
- Prototyping and simple scenarios only
- Not recommended for production

**EventProcessorClient:**
- Production-ready event processing
- Automatic load balancing
- Built-in checkpointing
- Requires Azure Blob Storage

**Best Practices:**
- Always use batching when sending
- Checkpoint periodically (not every event)
- Use EventProcessorClient for production
- Handle errors gracefully (don't checkpoint on error)
- Reuse clients across operations