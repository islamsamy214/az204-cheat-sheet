# Exercise: Send and Receive Events with Event Hubs

## Exercise Overview

In this hands-on exercise, you'll create an Event Hubs namespace and event hub, then build producer and consumer applications to send and receive events.

**Estimated Time:** 30-40 minutes

**What You'll Learn:**
- Create Event Hubs namespace and event hub
- Send events with EventHubProducerClient
- Receive events with EventHubConsumerClient
- Process events with EventProcessorClient
- Monitor event flow with Azure Portal

### Architecture

```
┌────────────────────┐
│  Producer App      │
│  (Console App)     │
│  • Send 100 events │
│  • Batching        │
│  • Partition key   │
└─────────┬──────────┘
          │
          ▼
┌─────────────────────────────┐
│  EVENT HUBS NAMESPACE       │
│  ┌───────────────────────┐  │
│  │  Event Hub: telemetry │  │
│  │  ┌────┬────┬────┬───┐ │  │
│  │  │ P0 │ P1 │ P2 │P3 │ │  │
│  │  └────┴────┴────┴───┘ │  │
│  └───────────────────────┘  │
└─────────────┬───────────────┘
              │
              ▼
┌──────────────────────────────┐
│  Consumer App                │
│  (Console App)               │
│  • Read all partitions       │
│  • Display events            │
└──────────────────────────────┘
              │
              ▼
┌──────────────────────────────┐
│  Event Processor App         │
│  (Console App)               │
│  • Automatic load balancing  │
│  • Checkpointing             │
│  • Fault tolerance           │
└──────────────────────────────┘
```

---

## Prerequisites

### Required Tools

- **Azure Subscription**: Free or paid subscription
- **.NET SDK**: 6.0 or later ([Download](https://dotnet.microsoft.com/download))
- **Azure CLI**: Latest version ([Install](https://docs.microsoft.com/cli/azure/install-azure-cli))
- **Code Editor**: Visual Studio Code or Visual Studio

### Verify Prerequisites

```bash
# Check .NET SDK
dotnet --version
# Output: 6.0.x or later

# Check Azure CLI
az --version
# Output: azure-cli 2.x.x

# Login to Azure
az login
```

---

## Part 1: Create Azure Resources

### Step 1: Define Variables

```bash
# Resource group
RESOURCE_GROUP="rg-eventhubs-lab"
LOCATION="eastus"

# Event Hubs
NAMESPACE="ehns-lab-$RANDOM"  # Must be globally unique
EVENTHUB="telemetry"

echo "Namespace: $NAMESPACE"
```

### Step 2: Create Resource Group

```bash
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION

# Output:
# {
#   "id": "/subscriptions/.../resourceGroups/rg-eventhubs-lab",
#   "location": "eastus",
#   "name": "rg-eventhubs-lab",
#   "properties": {
#     "provisioningState": "Succeeded"
#   }
# }
```

### Step 3: Create Event Hubs Namespace

```bash
az eventhubs namespace create \
  --name $NAMESPACE \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard \
  --capacity 1

# Wait for provisioning (1-2 minutes)
echo "Namespace created: $NAMESPACE.servicebus.windows.net"
```

**Namespace SKUs:**

| SKU | Max TUs | Features | Cost |
|-----|---------|----------|------|
| Basic | 20 | Basic features | Lowest |
| Standard | 40 | Consumer groups, Capture | Moderate |
| Premium | Processing Units | Dedicated, isolation | Higher |

### Step 4: Create Event Hub

```bash
az eventhubs eventhub create \
  --name $EVENTHUB \
  --namespace-name $NAMESPACE \
  --resource-group $RESOURCE_GROUP \
  --partition-count 4 \
  --message-retention 1

# Output:
# {
#   "name": "telemetry",
#   "partitionCount": 4,
#   "status": "Active",
#   "messageRetentionInDays": 1
# }
```

### Step 5: Create Consumer Group

```bash
az eventhubs eventhub consumer-group create \
  --name processor \
  --eventhub-name $EVENTHUB \
  --namespace-name $NAMESPACE \
  --resource-group $RESOURCE_GROUP

echo "Consumer group 'processor' created"
```

### Step 6: Get Connection String

```bash
# Get connection string
CONNECTION_STRING=$(az eventhubs namespace authorization-rule keys list \
  --name RootManageSharedAccessKey \
  --namespace-name $NAMESPACE \
  --resource-group $RESOURCE_GROUP \
  --query primaryConnectionString \
  --output tsv)

echo "Connection String: $CONNECTION_STRING"

# Save to file for later use
echo $CONNECTION_STRING > connection-string.txt
```

**⚠️ Security Note**: Connection string contains sensitive information. Never commit to source control!

---

## Part 2: Create Producer Application

### Step 1: Create Console Application

```bash
# Create directory
mkdir EventHubsProducer
cd EventHubsProducer

# Create console app
dotnet new console

# Add Event Hubs package
dotnet add package Azure.Messaging.EventHubs

# Restore packages
dotnet restore
```

### Step 2: Producer Code

Edit `Program.cs`:

```csharp
using Azure.Messaging.EventHubs;
using Azure.Messaging.EventHubs.Producer;
using System.Text;
using System.Text.Json;

// Configuration
string connectionString = "<YOUR-CONNECTION-STRING>";
string eventHubName = "telemetry";

Console.WriteLine("Event Hubs Producer");
Console.WriteLine("===================");
Console.WriteLine();

// Create producer client
await using var producer = new EventHubProducerClient(connectionString, eventHubName);

Console.WriteLine($"Connected to Event Hub: {eventHubName}");
Console.WriteLine();

// Get Event Hub properties
EventHubProperties properties = await producer.GetEventHubPropertiesAsync();
Console.WriteLine($"Event Hub: {properties.Name}");
Console.WriteLine($"Created: {properties.CreatedOn}");
Console.WriteLine($"Partitions: {properties.PartitionIds.Length}");
Console.WriteLine($"Partition IDs: {string.Join(", ", properties.PartitionIds)}");
Console.WriteLine();

// Send events
int eventCount = 100;
int eventsSent = 0;

Console.WriteLine($"Sending {eventCount} events...");
Console.WriteLine();

// Create batch
using EventDataBatch eventBatch = await producer.CreateBatchAsync();

for (int i = 1; i <= eventCount; i++)
{
    // Create telemetry data
    var telemetry = new
    {
        DeviceId = $"device-{i % 10:D3}",  // 10 devices (device-000 to device-009)
        Temperature = 20 + (i % 30),        // Temperature: 20-50°C
        Humidity = 30 + (i % 40),           // Humidity: 30-70%
        Timestamp = DateTime.UtcNow
    };
    
    string jsonData = JsonSerializer.Serialize(telemetry);
    
    // Create event data
    var eventData = new EventData(Encoding.UTF8.GetBytes(jsonData));
    
    // Add application properties
    eventData.Properties["DeviceId"] = telemetry.DeviceId;
    eventData.Properties["MessageType"] = "Telemetry";
    
    // Set partition key (all events from same device go to same partition)
    var batchOptions = new CreateBatchOptions
    {
        PartitionKey = telemetry.DeviceId
    };
    
    // Try to add to batch
    if (!eventBatch.TryAdd(eventData))
    {
        // Batch full, send it
        await producer.SendAsync(eventBatch);
        eventsSent += eventBatch.Count;
        Console.WriteLine($"Sent batch: {eventBatch.Count} events (Total: {eventsSent}/{eventCount})");
        
        // Create new batch and add current event
        eventBatch = await producer.CreateBatchAsync(batchOptions);
        
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
    eventsSent += eventBatch.Count;
    Console.WriteLine($"Sent final batch: {eventBatch.Count} events (Total: {eventsSent}/{eventCount})");
}

Console.WriteLine();
Console.WriteLine($"✓ Successfully sent {eventsSent} events!");
Console.WriteLine();
Console.WriteLine("Press any key to exit...");
Console.ReadKey();
```

### Step 3: Run Producer

```bash
# Update connection string in Program.cs
# Replace <YOUR-CONNECTION-STRING> with actual value

# Run application
dotnet run

# Expected output:
# Event Hubs Producer
# ===================
#
# Connected to Event Hub: telemetry
#
# Event Hub: telemetry
# Created: 2024-01-15 10:00:00Z
# Partitions: 4
# Partition IDs: 0, 1, 2, 3
#
# Sending 100 events...
#
# Sent batch: 50 events (Total: 50/100)
# Sent final batch: 50 events (Total: 100/100)
#
# ✓ Successfully sent 100 events!
```

---

## Part 3: Create Consumer Application

### Step 1: Create Console Application

```bash
# Go back to parent directory
cd ..

# Create directory
mkdir EventHubsConsumer
cd EventHubsConsumer

# Create console app
dotnet new console

# Add Event Hubs package
dotnet add package Azure.Messaging.EventHubs

# Restore packages
dotnet restore
```

### Step 2: Consumer Code

Edit `Program.cs`:

```csharp
using Azure.Messaging.EventHubs.Consumer;
using System.Text;
using System.Text.Json;

// Configuration
string connectionString = "<YOUR-CONNECTION-STRING>";
string eventHubName = "telemetry";
string consumerGroup = EventHubConsumerClient.DefaultConsumerGroupName;

Console.WriteLine("Event Hubs Consumer");
Console.WriteLine("===================");
Console.WriteLine();

// Create consumer client
await using var consumer = new EventHubConsumerClient(
    consumerGroup,
    connectionString,
    eventHubName
);

Console.WriteLine($"Connected to Event Hub: {eventHubName}");
Console.WriteLine($"Consumer Group: {consumerGroup}");
Console.WriteLine();

// Configure read options
var readOptions = new ReadEventOptions
{
    MaximumWaitTime = TimeSpan.FromSeconds(5)
};

Console.WriteLine("Reading events (Ctrl+C to stop)...");
Console.WriteLine();

int eventCount = 0;

// Create cancellation token (stop after 30 seconds for demo)
using var cancellationSource = new CancellationTokenSource();
cancellationSource.CancelAfter(TimeSpan.FromSeconds(30));

try
{
    // Read events
    await foreach (PartitionEvent partitionEvent in consumer.ReadEventsAsync(
        readOptions,
        cancellationSource.Token))
    {
        // Skip null events (timeout)
        if (partitionEvent.Data == null)
        {
            continue;
        }
        
        eventCount++;
        
        // Parse event data
        string body = Encoding.UTF8.GetString(partitionEvent.Data.EventBody.ToArray());
        
        // Display event information
        Console.WriteLine($"Event #{eventCount}");
        Console.WriteLine($"  Partition: {partitionEvent.Partition.PartitionId}");
        Console.WriteLine($"  Offset: {partitionEvent.Data.Offset}");
        Console.WriteLine($"  Sequence: {partitionEvent.Data.SequenceNumber}");
        Console.WriteLine($"  Enqueued: {partitionEvent.Data.EnqueuedTime:yyyy-MM-dd HH:mm:ss}");
        
        // Display application properties
        if (partitionEvent.Data.Properties.Count > 0)
        {
            Console.WriteLine($"  Properties:");
            foreach (var prop in partitionEvent.Data.Properties)
            {
                Console.WriteLine($"    {prop.Key}: {prop.Value}");
            }
        }
        
        // Display body
        Console.WriteLine($"  Body: {body}");
        Console.WriteLine();
        
        // Optional: Parse JSON
        try
        {
            var telemetry = JsonSerializer.Deserialize<JsonElement>(body);
            string deviceId = telemetry.GetProperty("DeviceId").GetString();
            double temperature = telemetry.GetProperty("Temperature").GetDouble();
            double humidity = telemetry.GetProperty("Humidity").GetDouble();
            
            // Check for alerts
            if (temperature > 40)
            {
                Console.WriteLine($"  ⚠️  ALERT: High temperature detected! Device: {deviceId}, Temp: {temperature}°C");
                Console.WriteLine();
            }
        }
        catch
        {
            // Ignore JSON parsing errors
        }
    }
}
catch (TaskCanceledException)
{
    Console.WriteLine("Reading stopped (timeout reached)");
}

Console.WriteLine();
Console.WriteLine($"✓ Read {eventCount} events");
Console.WriteLine();
Console.WriteLine("Press any key to exit...");
Console.ReadKey();
```

### Step 3: Run Consumer

**Terminal 1 (Producer):**
```bash
cd EventHubsProducer
dotnet run
```

**Terminal 2 (Consumer):**
```bash
cd EventHubsConsumer
dotnet run

# Expected output:
# Event Hubs Consumer
# ===================
#
# Connected to Event Hub: telemetry
# Consumer Group: $Default
#
# Reading events (Ctrl+C to stop)...
#
# Event #1
#   Partition: 2
#   Offset: 12345
#   Sequence: 67890
#   Enqueued: 2024-01-15 10:30:00
#   Properties:
#     DeviceId: device-001
#     MessageType: Telemetry
#   Body: {"DeviceId":"device-001","Temperature":21,"Humidity":31,"Timestamp":"..."}
#
# Event #2
#   Partition: 1
#   ...
```

---

## Part 4: Create Event Processor Application

### Step 1: Create Console Application

```bash
# Go back to parent directory
cd ..

# Create directory
mkdir EventHubsProcessor
cd EventHubsProcessor

# Create console app
dotnet new console

# Add Event Hubs packages
dotnet add package Azure.Messaging.EventHubs
dotnet add package Azure.Messaging.EventHubs.Processor
dotnet add package Azure.Storage.Blobs

# Restore packages
dotnet restore
```

### Step 2: Create Storage Account

```bash
# Storage account name (must be globally unique)
STORAGE_ACCOUNT="stehlab$RANDOM"

# Create storage account
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS

# Get connection string
STORAGE_CONNECTION_STRING=$(az storage account show-connection-string \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query connectionString \
  --output tsv)

echo "Storage Connection String: $STORAGE_CONNECTION_STRING"

# Create container for checkpoints
az storage container create \
  --name checkpoints \
  --account-name $STORAGE_ACCOUNT \
  --connection-string "$STORAGE_CONNECTION_STRING"
```

### Step 3: Event Processor Code

Edit `Program.cs`:

```csharp
using Azure.Messaging.EventHubs;
using Azure.Messaging.EventHubs.Consumer;
using Azure.Messaging.EventHubs.Processor;
using Azure.Storage.Blobs;
using System.Text;
using System.Text.Json;

// Configuration
string eventHubsConnectionString = "<YOUR-EVENTHUBS-CONNECTION-STRING>";
string eventHubName = "telemetry";
string consumerGroup = "processor";  // Use custom consumer group

string blobStorageConnectionString = "<YOUR-STORAGE-CONNECTION-STRING>";
string blobContainerName = "checkpoints";

Console.WriteLine("Event Hubs Processor");
Console.WriteLine("====================");
Console.WriteLine();

// Create blob container client for checkpoint store
BlobContainerClient storageClient = new BlobContainerClient(
    blobStorageConnectionString,
    blobContainerName
);

// Create container if it doesn't exist
await storageClient.CreateIfNotExistsAsync();

Console.WriteLine($"Checkpoint store: {blobContainerName}");
Console.WriteLine();

// Create event processor client
EventProcessorClient processor = new EventProcessorClient(
    storageClient,
    consumerGroup,
    eventHubsConnectionString,
    eventHubName
);

// Track statistics
int eventCount = 0;
int checkpointCount = 0;
DateTime startTime = DateTime.UtcNow;

// Event handler
processor.ProcessEventAsync += async (ProcessEventArgs args) =>
{
    try
    {
        // Skip null events
        if (args.Data == null)
        {
            return;
        }
        
        // Parse event data
        string body = Encoding.UTF8.GetString(args.Data.EventBody.ToArray());
        
        // Increment counter (thread-safe)
        int currentCount = Interlocked.Increment(ref eventCount);
        
        // Display event
        Console.WriteLine($"[{args.Partition.PartitionId}] Event #{currentCount}: {body}");
        
        // Process event (your business logic)
        await ProcessEventAsync(body);
        
        // Checkpoint every 10 events (balance fault tolerance vs performance)
        if (currentCount % 10 == 0)
        {
            await args.UpdateCheckpointAsync();
            Interlocked.Increment(ref checkpointCount);
            Console.WriteLine($"  ✓ Checkpointed at event {currentCount}");
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error processing event: {ex.Message}");
        // Don't checkpoint on error - event will be reprocessed
    }
};

// Error handler
processor.ProcessErrorAsync += (ProcessErrorEventArgs args) =>
{
    Console.WriteLine($"Error in partition {args.PartitionId}: {args.Exception.Message}");
    return Task.CompletedTask;
};

// Start processing
Console.WriteLine("Starting event processor...");
await processor.StartProcessingAsync();
Console.WriteLine("✓ Event processor started");
Console.WriteLine();

Console.WriteLine("Processing events (press Enter to stop)...");
Console.WriteLine();

// Wait for user input
Console.ReadLine();

// Display statistics
TimeSpan duration = DateTime.UtcNow - startTime;
Console.WriteLine();
Console.WriteLine("Statistics:");
Console.WriteLine($"  Events processed: {eventCount}");
Console.WriteLine($"  Checkpoints: {checkpointCount}");
Console.WriteLine($"  Duration: {duration.TotalSeconds:F1} seconds");
Console.WriteLine($"  Throughput: {eventCount / duration.TotalSeconds:F1} events/sec");
Console.WriteLine();

// Stop processing
Console.WriteLine("Stopping event processor...");
await processor.StopProcessingAsync();
Console.WriteLine("✓ Event processor stopped");

async Task ProcessEventAsync(string eventData)
{
    // Your business logic here
    // Example: Parse JSON, save to database, send alert, etc.
    
    try
    {
        var telemetry = JsonSerializer.Deserialize<JsonElement>(eventData);
        string deviceId = telemetry.GetProperty("DeviceId").GetString();
        double temperature = telemetry.GetProperty("Temperature").GetDouble();
        
        // Check for alerts
        if (temperature > 40)
        {
            Console.WriteLine($"  ⚠️  ALERT: High temperature! Device: {deviceId}, Temp: {temperature}°C");
        }
    }
    catch
    {
        // Ignore parsing errors
    }
    
    // Simulate processing time
    await Task.Delay(10);
}
```

### Step 4: Run Event Processor

**Terminal 1 (Producer - continuous):**
```bash
cd EventHubsProducer

# Modify Program.cs to loop continuously
# Add: while (true) { ... await Task.Delay(1000); }

dotnet run
```

**Terminal 2 (Event Processor):**
```bash
cd EventHubsProcessor
dotnet run

# Expected output:
# Event Hubs Processor
# ====================
#
# Checkpoint store: checkpoints
#
# Starting event processor...
# ✓ Event processor started
#
# Processing events (press Enter to stop)...
#
# [2] Event #1: {"DeviceId":"device-001","Temperature":21,...}
# [1] Event #2: {"DeviceId":"device-002","Temperature":22,...}
# [0] Event #3: {"DeviceId":"device-003","Temperature":43,...}
#   ⚠️  ALERT: High temperature! Device: device-003, Temp: 43°C
# ...
# [2] Event #10: {"DeviceId":"device-000","Temperature":30,...}
#   ✓ Checkpointed at event 10
```

**Terminal 3 (Second Event Processor Instance - Load Balancing Test):**
```bash
cd EventHubsProcessor
dotnet run

# Observe partition rebalancing
# Instance 1 will release some partitions
# Instance 2 will claim those partitions
# Events distributed between both instances
```

---

## Part 5: Monitor with Azure Portal

### Step 1: View Metrics

1. Navigate to [Azure Portal](https://portal.azure.com)
2. Go to your Event Hubs namespace
3. Select **Metrics** under Monitoring
4. Add metrics:
   - **Incoming Messages**: Events sent
   - **Outgoing Messages**: Events read
   - **Throttled Requests**: Throttling events
   - **User Errors**: Client errors

### Step 2: View Event Hub Details

1. Select your Event Hub (`telemetry`)
2. Select **Overview**
3. View:
   - **Message Count**: Total messages
   - **Throughput Units**: Current utilization
   - **Partitions**: Distribution

### Step 3: View Partition Information

1. Select **Partitions** under Entities
2. View per-partition metrics:
   - Incoming messages
   - Outgoing messages
   - Active connections

### Step 4: View Consumer Groups

1. Select **Consumer groups** under Entities
2. View consumer groups:
   - `$Default`: Default group
   - `processor`: Custom group

---

## Part 6: Test Scenarios

### Scenario 1: Partition Distribution

**Test:** Verify events distributed across partitions

```bash
# Send 100 events with partition key
cd EventHubsProducer
dotnet run

# Check Azure Portal
# Navigate to: Event Hub → Partitions
# Verify events distributed based on partition key hash
```

### Scenario 2: Load Balancing

**Test:** Multiple processor instances share load

```bash
# Terminal 1: Start first processor
cd EventHubsProcessor
dotnet run

# Terminal 2: Start second processor (in separate directory)
cd EventHubsProcessor
dotnet run

# Observe:
# - Partitions rebalanced between instances
# - Each instance processes subset of partitions
# - Total throughput increases
```

### Scenario 3: Fault Tolerance

**Test:** Processor recovery after crash

```bash
# Terminal 1: Start processor
dotnet run

# Process some events (observe checkpointing)
# Press Ctrl+C to stop (simulate crash)

# Restart processor
dotnet run

# Observe:
# - Processor resumes from last checkpoint
# - Events after checkpoint reprocessed
# - No events lost
```

### Scenario 4: Consumer Lag

**Test:** Monitor consumer lag

```bash
# Send events faster than consumption
# Terminal 1: Send 1000 events
cd EventHubsProducer
# Modify: Send 1000 events
dotnet run

# Terminal 2: Slow consumer
cd EventHubsConsumer
# Modify: Add Task.Delay(100) per event
dotnet run

# Azure Portal:
# Navigate to: Event Hub → Metrics
# Add metric: Consumer Lag
# Observe lag increasing
```

---

## Part 7: Cleanup Resources

### Delete Resource Group

```bash
# Delete all resources
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait

echo "Resource group deletion initiated"
```

**⚠️ Warning**: This deletes ALL resources in the resource group, including:
- Event Hubs namespace
- Event Hub
- Storage account
- All data

---

## Key Takeaways

### Concepts Learned

1. **Event Hubs Architecture**
   - Namespace → Event Hub → Partitions
   - Consumer groups for independent consumption
   - Partition-based parallelism

2. **EventHubProducerClient**
   - Batching for performance
   - Partition key for related events
   - Application properties for metadata

3. **EventHubConsumerClient**
   - Read events from all partitions
   - Iterator pattern for consumption
   - Good for prototyping

4. **EventProcessorClient**
   - Automatic load balancing
   - Checkpoint-based fault tolerance
   - Production-ready scalability

5. **Checkpointing**
   - Tracks processing progress
   - Enables failure recovery
   - Requires Azure Blob Storage

### Best Practices Applied

✅ **Batching**: Used `CreateBatchAsync()` for efficient sending
✅ **Partition Key**: Grouped related events (same device)
✅ **Checkpointing**: Balanced frequency (every 10 events)
✅ **Error Handling**: Try-catch with no checkpoint on error
✅ **Resource Management**: Used `await using` for disposal
✅ **Load Balancing**: Multiple processor instances automatically balanced
✅ **Monitoring**: Used Azure Portal metrics for observability

---

## Troubleshooting Guide

### Issue 1: Connection Failed

**Error:**
```
Azure.Messaging.EventHubs.EventHubsException: The Azure Active Directory access token has expired.
```

**Solution:**
```bash
# Refresh Azure login
az login

# Or check connection string
az eventhubs namespace authorization-rule keys list \
  --name RootManageSharedAccessKey \
  --namespace-name $NAMESPACE \
  --resource-group $RESOURCE_GROUP
```

### Issue 2: Partition Not Found

**Error:**
```
The specified partition '5' does not exist.
```

**Solution:**
```bash
# Check partition count
az eventhubs eventhub show \
  --name $EVENTHUB \
  --namespace-name $NAMESPACE \
  --resource-group $RESOURCE_GROUP \
  --query partitionCount

# Update code to use valid partition IDs (0 to partitionCount-1)
```

### Issue 3: Checkpoint Store Access Denied

**Error:**
```
Azure.RequestFailedException: This request is not authorized to perform this operation.
```

**Solution:**
```bash
# Verify storage connection string
az storage account show-connection-string \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP

# Ensure container exists
az storage container create \
  --name checkpoints \
  --account-name $STORAGE_ACCOUNT
```

### Issue 4: No Events Received

**Possible Causes:**
- Events sent to different Event Hub
- Using wrong consumer group
- Events expired (retention period)

**Solution:**
```bash
# Verify Event Hub has events
az monitor metrics list \
  --resource /subscriptions/.../providers/Microsoft.EventHub/namespaces/$NAMESPACE/eventhubs/$EVENTHUB \
  --metric "IncomingMessages" \
  --start-time "2024-01-15T00:00:00Z"

# Check retention period
az eventhubs eventhub show \
  --name $EVENTHUB \
  --namespace-name $NAMESPACE \
  --resource-group $RESOURCE_GROUP \
  --query messageRetentionInDays
```

---

## Exam Tips for AZ-204

### Key Concepts from Exercise

1. **Event Hubs Setup**
   - Create namespace (Standard SKU for consumer groups)
   - Create event hub with partitions
   - Get connection string

2. **Producer Pattern**
   - Use `EventHubProducerClient`
   - Always use batching (`CreateBatchAsync()`)
   - Set partition key for related events

3. **Consumer Pattern**
   - `EventHubConsumerClient`: Prototyping only
   - `EventProcessorClient`: Production (requires Blob Storage)

4. **Checkpointing**
   - Call `UpdateCheckpointAsync()` periodically
   - Don't checkpoint on error
   - Balance frequency (fault tolerance vs performance)

5. **Load Balancing**
   - Multiple EventProcessorClient instances
   - Automatic partition distribution
   - Scales horizontally

### Remember for Exam

- **Namespace**: Container for Event Hubs (like SQL Server)
- **Event Hub**: Append-only log (like SQL table)
- **Partitions**: Ordered sequences (cannot change after creation)
- **Consumer Group**: Independent view of Event Hub
- **Checkpoint Store**: Azure Blob Storage (required for EventProcessorClient)
- **Batching**: `CreateBatchAsync()` → `TryAdd()` → `SendAsync()`
- **Partition Key**: Hash-based distribution (maintains order)
- **EventProcessorClient**: Production choice (automatic load balancing)

### Common Exam Scenarios

**Scenario 1**: Build telemetry ingestion system
- ✅ Use Event Hubs (designed for high-volume ingestion)
- ✅ EventHubProducerClient with batching
- ✅ Partition key for device grouping

**Scenario 2**: Scale event processing
- ✅ Use EventProcessorClient
- ✅ Multiple instances for horizontal scaling
- ✅ Checkpoint store for fault tolerance

**Scenario 3**: Track processing progress
- ✅ Use checkpointing (`UpdateCheckpointAsync`)
- ✅ Azure Blob Storage as checkpoint store
- ✅ Resume from checkpoint after failure

---

## Summary

In this exercise, you:

✅ Created Event Hubs namespace and event hub
✅ Built producer application with batching
✅ Built consumer application with iterator pattern
✅ Built event processor with load balancing and checkpointing
✅ Tested fault tolerance and scaling
✅ Monitored metrics in Azure Portal

**Production Checklist:**
- ✅ Use EventProcessorClient (not EventHubConsumerClient)
- ✅ Configure checkpoint store (Azure Blob Storage)
- ✅ Implement proper error handling
- ✅ Use batching for sending events
- ✅ Set appropriate checkpoint frequency
- ✅ Monitor metrics (consumer lag, throughput)
- ✅ Use managed identity (not connection strings)
- ✅ Configure auto-scaling for consumer apps