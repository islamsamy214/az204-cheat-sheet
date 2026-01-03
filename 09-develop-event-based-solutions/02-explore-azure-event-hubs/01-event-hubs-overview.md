# Azure Event Hubs Overview

## What is Azure Event Hubs?

**Azure Event Hubs** is a fully managed, real-time data streaming platform and event ingestion service that can receive and process **millions of events per second** with low latency.

### Key Characteristics

- **Big Data Streaming**: Ingest millions of events per second
- **Low Latency**: Sub-second latency for real-time processing
- **Apache Kafka Compatible**: Run existing Kafka workloads without code changes
- **Durable Storage**: Store streaming data for batch and real-time processing
- **Partitioned Consumer Model**: Scale with multiple parallel consumers
- **Multi-Protocol Support**: AMQP, Kafka, HTTPS protocols
- **Fully Managed**: No infrastructure to manage

### Primary Use Cases

| Use Case | Description | Example |
|----------|-------------|---------|
| **Telemetry & IoT** | Ingest IoT device telemetry at scale | Smart home sensors, connected vehicles |
| **Application Logging** | Collect application logs from distributed systems | Microservices log aggregation |
| **Clickstream Analytics** | Track user behavior on websites/apps | E-commerce user journey tracking |
| **Live Dashboarding** | Real-time metrics and monitoring | Operations dashboard, KPI tracking |
| **Anomaly Detection** | Detect fraud or security threats | Credit card fraud detection |
| **Archiving** | Store streaming data for compliance | Financial transaction logs |
| **Transaction Processing** | Process financial transactions | Payment processing, order management |

---

## Event Hubs Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        EVENT PRODUCERS                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ IoT      │  │ Web/     │  │ Mobile   │  │ Kafka    │       │
│  │ Devices  │  │ Mobile   │  │ Apps     │  │ Apps     │       │
│  │          │  │ Apps     │  │          │  │          │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
└───────┼─────────────┼─────────────┼─────────────┼──────────────┘
        │             │             │             │
        │ AMQP/HTTPS  │ HTTPS       │ HTTPS       │ Kafka Protocol
        └─────────────┴─────────────┴─────────────┘
                      │
                      ▼
        ┌─────────────────────────────────┐
        │   EVENT HUBS NAMESPACE          │
        │  ┌───────────────────────────┐  │
        │  │   Event Hub (Topic)       │  │
        │  │  ┌────┬────┬────┬────┐   │  │
        │  │  │ P0 │ P1 │ P2 │ P3 │   │  │  ← Partitions
        │  │  └────┴────┴────┴────┘   │  │
        │  └───────────────────────────┘  │
        │                                 │
        │  ┌───────────────────────────┐  │
        │  │   Consumer Groups         │  │
        │  │  • $Default               │  │
        │  │  • StreamAnalytics        │  │
        │  │  • ApplicationProcessing  │  │
        │  └───────────────────────────┘  │
        └─────────────┬───────────────────┘
                      │
        ┌─────────────┴──────────────────────┐
        │                                    │
        ▼                                    ▼
┌───────────────────┐            ┌───────────────────┐
│  CONSUMER GROUP 1 │            │  CONSUMER GROUP 2 │
│                   │            │                   │
│ ┌───────────────┐ │            │ ┌───────────────┐ │
│ │ Consumer 1    │ │            │ │ Stream        │ │
│ │ (reads P0-P1) │ │            │ │ Analytics     │ │
│ ├───────────────┤ │            │ │ (reads all)   │ │
│ │ Consumer 2    │ │            │ └───────────────┘ │
│ │ (reads P2-P3) │ │            │                   │
│ └───────────────┘ │            └───────────────────┘
└───────────────────┘
        │
        ▼
┌───────────────────┐
│ Event Processor   │
│ (Checkpointing)   │
│                   │
│ Azure Blob        │
│ Storage           │
└───────────────────┘
```

---

## Core Event Hubs Concepts

### 1. Event Hubs Namespace

A **namespace** is a management container for one or more Event Hubs (or Kafka topics).

**Namespace Characteristics:**
- **Scope**: Regional resource in a specific Azure location
- **Capacity**: Defines throughput units or processing units
- **Network**: Configure VNet, firewall, private endpoints
- **Security**: Manage access keys and authorization policies
- **Unique Endpoint**: `<namespace>.servicebus.windows.net`

**Create Namespace (Azure CLI):**
```bash
az eventhubs namespace create \
  --name myeventhubns \
  --resource-group myResourceGroup \
  --location eastus \
  --sku Standard \
  --capacity 1
```

**Namespace Tiers:**

| Tier | Throughput | Features | Use Case |
|------|-----------|----------|----------|
| **Basic** | Up to 20 TUs | Basic features, 1-day retention | Development, testing |
| **Standard** | Up to 40 TUs | Consumer groups, Capture, auto-inflate | Production workloads |
| **Premium** | Processing Units | Dedicated resources, isolation, longer retention | Mission-critical, high throughput |
| **Dedicated** | Capacity Units | Single-tenant deployment | Enterprise, compliance |

### 2. Event Hub (Kafka Topic)

An **Event Hub** is a log-structured append-only stream, similar to a Kafka topic.

**Event Hub Properties:**
- **Partitions**: 1-32 partitions (Standard), up to 100 (Premium)
- **Retention**: 1-90 days (configurable)
- **Message Size**: Up to 1 MB per event
- **Throughput**: Controlled by throughput units
- **Ordering**: Guaranteed within a partition

**Create Event Hub (Azure CLI):**
```bash
az eventhubs eventhub create \
  --name myeventhub \
  --namespace-name myeventhubns \
  --resource-group myResourceGroup \
  --partition-count 4 \
  --message-retention 7
```

### 3. Partitions

**Partitions** are ordered sequences of events within an Event Hub, similar to lanes on a freeway.

**Why Partitions?**
- **Scalability**: Distribute load across multiple consumers
- **Parallelism**: Process events in parallel
- **Ordering**: Events in same partition maintain order
- **Performance**: Each partition has independent throughput

**Partition Architecture:**

```
Event Hub: "telemetry-data"
├── Partition 0: [E1] [E2] [E3] [E4] [E5] → oldest...newest
├── Partition 1: [E6] [E7] [E8] [E9] [E10]
├── Partition 2: [E11] [E12] [E13] [E14] [E15]
└── Partition 3: [E16] [E17] [E18] [E19] [E20]
```

**Partition Key:**
- **Purpose**: Determines which partition receives an event
- **Hashing**: Event Hubs hashes the key to select partition
- **Use**: Group related events (e.g., all events from device-001)

**Example - Partition Key:**
```csharp
// Events with same partition key go to same partition
var eventData = new EventData(Encoding.UTF8.GetBytes(jsonData))
{
    PartitionKey = "device-001"  // All events from this device go to same partition
};

await producer.SendAsync(eventData);
```

**Choosing Partition Count:**

| Partition Count | Throughput | Cost | Use Case |
|----------------|-----------|------|----------|
| **1-2** | Low (1-2 MB/s) | Lower | Development, low-volume |
| **4-8** | Medium (4-8 MB/s) | Moderate | Standard production |
| **16-32** | High (16-32 MB/s) | Higher | High-throughput applications |

**Important Notes:**
- ⚠️ **Cannot change partition count** after Event Hub creation
- ✅ Plan for future growth
- ✅ More partitions = more parallelism but higher cost

### 4. Consumer Groups

A **consumer group** is a view of the entire Event Hub, enabling multiple applications to read the same stream independently.

**Consumer Group Characteristics:**
- **Independent Offsets**: Each group tracks its own position
- **Parallel Processing**: Multiple groups process events simultaneously
- **Default Group**: `$Default` group created automatically
- **Maximum Groups**: 20 consumer groups per Event Hub (Standard tier)

**Consumer Group Use Cases:**

```
Event Hub: "orders"
├── Consumer Group: "$Default"
│   └── Real-time order processing application
│
├── Consumer Group: "analytics"
│   └── Azure Stream Analytics (near real-time analytics)
│
├── Consumer Group: "archiving"
│   └── Event Hubs Capture (long-term storage)
│
└── Consumer Group: "monitoring"
    └── Application monitoring dashboard
```

**Create Consumer Group (Azure CLI):**
```bash
az eventhubs eventhub consumer-group create \
  --name analytics \
  --eventhub-name myeventhub \
  --namespace-name myeventhubns \
  --resource-group myResourceGroup
```

**Example Scenario:**
```
Single Event Hub: "telemetry-data"

Consumer Group 1: Real-time alerting
  └── Processes events immediately for anomaly detection

Consumer Group 2: Batch analytics
  └── Reads events for hourly aggregation

Consumer Group 3: Machine learning
  └── Reads events for model training

All groups read the SAME events independently!
```

### 5. Event Producers

**Event producers** are applications that send data to Event Hubs.

**Producer Protocols:**
- **AMQP 1.0**: Recommended for high-throughput scenarios
- **Kafka**: Use existing Kafka producer clients
- **HTTPS**: Simple REST API for occasional publishers

**Producer Patterns:**

**Pattern 1: Round-Robin (No Partition Key)**
```csharp
// Events distributed evenly across partitions
var eventData = new EventData(Encoding.UTF8.GetBytes(jsonData));
await producer.SendAsync(eventData);
```

**Pattern 2: Partition Key (Related Events)**
```csharp
// Events with same key go to same partition (ordered)
var eventData = new EventData(Encoding.UTF8.GetBytes(jsonData))
{
    PartitionKey = customerId  // Group by customer
};
await producer.SendAsync(eventData);
```

**Pattern 3: Specific Partition**
```csharp
// Send directly to partition 2
var sendOptions = new SendEventOptions
{
    PartitionId = "2"
};
await producer.SendAsync(eventData, sendOptions);
```

**Batching for Performance:**
```csharp
// Create batch (more efficient)
EventDataBatch batch = await producer.CreateBatchAsync();

foreach (var data in dataList)
{
    var eventData = new EventData(Encoding.UTF8.GetBytes(data));
    
    if (!batch.TryAdd(eventData))
    {
        // Batch full, send it
        await producer.SendAsync(batch);
        
        // Create new batch
        batch = await producer.CreateBatchAsync();
        batch.TryAdd(eventData);
    }
}

// Send remaining events
if (batch.Count > 0)
{
    await producer.SendAsync(batch);
}
```

### 6. Event Consumers

**Event consumers** read and process data from Event Hubs.

**Consumer Types:**

| Consumer Type | Use Case | Example |
|--------------|----------|---------|
| **EventHubConsumerClient** | Simple prototyping, testing | Read events manually |
| **EventProcessorClient** | Production applications | Scalable, fault-tolerant processing |
| **Azure Stream Analytics** | Real-time analytics | SQL-like queries on streams |
| **Azure Functions** | Serverless event processing | Triggered by new events |
| **Apache Spark** | Big data processing | Large-scale batch processing |

**Consumer Pattern - EventHubConsumerClient:**
```csharp
var consumerGroup = EventHubConsumerClient.DefaultConsumerGroupName;
var consumer = new EventHubConsumerClient(consumerGroup, connectionString, eventHubName);

await foreach (PartitionEvent evt in consumer.ReadEventsAsync())
{
    Console.WriteLine($"Event: {evt.Data.EventBody}");
    Console.WriteLine($"Partition: {evt.Partition.PartitionId}");
    Console.WriteLine($"Offset: {evt.Data.Offset}");
}
```

**Consumer Pattern - EventProcessorClient (Recommended):**
```csharp
var storageClient = new BlobContainerClient(storageConnectionString, containerName);
var processor = new EventProcessorClient(storageClient, consumerGroup, connectionString, eventHubName);

processor.ProcessEventAsync += async (args) =>
{
    // Process event
    Console.WriteLine($"Event: {args.Data.EventBody}");
    
    // Update checkpoint
    await args.UpdateCheckpointAsync();
};

processor.ProcessErrorAsync += async (args) =>
{
    Console.WriteLine($"Error: {args.Exception.Message}");
};

await processor.StartProcessingAsync();
```

### 7. Checkpoints and Offsets

**Offset**: Position of an event within a partition

**Checkpoint**: Marking a specific offset as processed

```
Partition 0: [E0] [E1] [E2] [E3] [E4] [E5] [E6] [E7]
              ↑              ↑              ↑
           Offset=0     Offset=3      Offset=6
                           ↑
                      Checkpoint
              (Consumer has processed up to E3)
```

**Why Checkpointing?**
- **Fault Tolerance**: Resume from last checkpoint after failure
- **Exactly-Once**: Avoid reprocessing events
- **Progress Tracking**: Monitor consumer lag

**Checkpointing Storage:**
- **Azure Blob Storage**: Most common (EventProcessorClient)
- **Other Storage**: Custom implementation possible

---

## Throughput Units (TUs) and Processing Units (PUs)

### Throughput Units (Standard Tier)

**Throughput Unit** = capacity unit for Event Hubs Standard tier

**Per TU Capacity:**
- **Ingress**: Up to 1 MB/s or 1,000 events/second
- **Egress**: Up to 2 MB/s or 4,096 events/second

**Example Calculations:**

```
Scenario 1: 5,000 events/second, 500 bytes per event
- Ingress: 5,000 × 0.5 KB = 2.5 MB/s → Need 3 TUs
- Cost: Based on TU hours

Scenario 2: 500 KB events, 100 events/second
- Ingress: 100 × 0.5 MB = 50 MB/s → Need 50 TUs
- Consider Premium tier for better value
```

**Auto-Inflate:**
- Automatically scale TUs based on load
- Set maximum TU limit
- Useful for variable workloads

```bash
# Enable auto-inflate
az eventhubs namespace update \
  --name myeventhubns \
  --resource-group myResourceGroup \
  --enable-auto-inflate true \
  --maximum-throughput-units 20
```

### Processing Units (Premium/Dedicated Tiers)

**Processing Unit (PU)** = capacity for Premium tier

**Per PU:**
- More powerful than TUs
- Dedicated CPU and memory
- Better isolation and performance

---

## Event Retention

**Retention Period**: How long events are stored in Event Hubs

| Tier | Min Retention | Max Retention | Default |
|------|--------------|---------------|---------|
| **Basic** | 1 day | 1 day | 1 day |
| **Standard** | 1 day | 7 days | 1 day |
| **Premium** | 1 day | 90 days | 1 day |
| **Dedicated** | 1 day | 90 days | 1 day |

**Configure Retention:**
```bash
az eventhubs eventhub update \
  --name myeventhub \
  --namespace-name myeventhubns \
  --resource-group myResourceGroup \
  --message-retention 7
```

**Use Cases by Retention:**
- **1 day**: Real-time processing only, no replay needed
- **3-7 days**: Buffer for consumer recovery, debugging
- **30-90 days**: Compliance, reprocessing, batch analytics

---

## Apache Kafka Compatibility

Event Hubs supports **Apache Kafka protocol** natively.

**Benefits:**
- Use existing Kafka applications
- No code changes required
- Fully managed (no Kafka cluster management)
- Azure integration (security, monitoring, networking)

**Connection String Format:**
```
Bootstrap servers: <namespace>.servicebus.windows.net:9093
SASL mechanism: PLAIN
Security protocol: SASL_SSL
Username: $ConnectionString
Password: <connection-string>
```

**Kafka Producer Example (Java):**
```java
Properties props = new Properties();
props.put("bootstrap.servers", "myeventhubns.servicebus.windows.net:9093");
props.put("security.protocol", "SASL_SSL");
props.put("sasl.mechanism", "PLAIN");
props.put("sasl.jaas.config", "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"$ConnectionString\" password=\"Endpoint=sb://myeventhubns.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=<key>\";");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);

ProducerRecord<String, String> record = new ProducerRecord<>("myeventhub", "key", "value");
producer.send(record);
```

**Kafka to Event Hubs Mapping:**

| Kafka Concept | Event Hubs Equivalent |
|--------------|----------------------|
| Kafka Cluster | Event Hubs Namespace |
| Kafka Topic | Event Hub |
| Partition | Partition |
| Consumer Group | Consumer Group |
| Offset | Offset |
| Broker | Not applicable (managed service) |

---

## Schema Registry

**Azure Schema Registry** provides centralized schema management for event streaming applications.

**Features:**
- Store and version schemas
- Avro, JSON Schema support
- Integration with Kafka and Event Hubs SDKs
- Schema evolution and compatibility

**Benefits:**
- **Type Safety**: Ensure data contracts
- **Compatibility**: Manage schema evolution
- **Efficiency**: Reduce payload size with schema references
- **Governance**: Centralized schema management

---

## Event Hubs vs. Other Azure Services

| Feature | **Event Hubs** | **Event Grid** | **Service Bus** |
|---------|---------------|---------------|----------------|
| **Pattern** | Big data streaming | Event distribution | Message queue |
| **Throughput** | Millions/sec | Millions/sec | Thousands/sec |
| **Latency** | Sub-second | Sub-second | Low |
| **Retention** | 1-90 days | None (immediate) | Up to 14 days |
| **Ordering** | Per partition | No guarantee | FIFO (sessions) |
| **Message Size** | 1 MB | 1 MB | 256 KB (1 MB Premium) |
| **Protocols** | AMQP, Kafka, HTTPS | HTTP/HTTPS | AMQP, HTTP |
| **Use Case** | Telemetry ingestion | Reactive events | Transactional messages |
| **Pull/Push** | Pull | Push (+ Pull) | Pull |

**When to Use Event Hubs:**
- ✅ High-volume telemetry ingestion (IoT, logs, metrics)
- ✅ Real-time analytics and dashboards
- ✅ Event replay and reprocessing
- ✅ Kafka workload migration
- ✅ Big data pipelines

**When NOT to Use Event Hubs:**
- ❌ Need complex message workflows (use Service Bus)
- ❌ Need push-based event distribution (use Event Grid)
- ❌ Low-volume transactional messages (use Service Bus)
- ❌ Need guaranteed message ordering across all events (use Service Bus sessions)

---

## Quick Start Example

### Step 1: Create Event Hubs Resources

```bash
# Variables
RG="rg-eventhubs"
LOCATION="eastus"
NAMESPACE="myeventhubns"
EVENTHUB="myeventhub"

# Create resource group
az group create --name $RG --location $LOCATION

# Create namespace
az eventhubs namespace create \
  --name $NAMESPACE \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard

# Create event hub
az eventhubs eventhub create \
  --name $EVENTHUB \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --partition-count 4 \
  --message-retention 7

# Get connection string
az eventhubs namespace authorization-rule keys list \
  --name RootManageSharedAccessKey \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query primaryConnectionString \
  --output tsv
```

### Step 2: Send Events (C#)

```csharp
using Azure.Messaging.EventHubs;
using Azure.Messaging.EventHubs.Producer;

string connectionString = "<connection-string>";
string eventHubName = "myeventhub";

await using var producer = new EventHubProducerClient(connectionString, eventHubName);

// Create batch
using EventDataBatch eventBatch = await producer.CreateBatchAsync();

for (int i = 0; i < 10; i++)
{
    eventBatch.TryAdd(new EventData($"Event {i}"));
}

// Send batch
await producer.SendAsync(eventBatch);
Console.WriteLine("Events sent successfully!");
```

### Step 3: Receive Events (C#)

```csharp
using Azure.Messaging.EventHubs.Consumer;

string connectionString = "<connection-string>";
string eventHubName = "myeventhub";
string consumerGroup = EventHubConsumerClient.DefaultConsumerGroupName;

await using var consumer = new EventHubConsumerClient(consumerGroup, connectionString, eventHubName);

await foreach (PartitionEvent partitionEvent in consumer.ReadEventsAsync())
{
    Console.WriteLine($"Partition: {partitionEvent.Partition.PartitionId}");
    Console.WriteLine($"Event: {partitionEvent.Data.EventBody}");
}
```

---

## Best Practices

### Design Patterns

1. **Partition Key Strategy**
   - Use partition keys to group related events
   - Balance partition distribution
   - Example: User ID, Device ID, Session ID

2. **Batch Events**
   - Send events in batches for better throughput
   - Reduce network overhead
   - Use `CreateBatchAsync()` method

3. **Consumer Groups**
   - Create separate consumer groups for each application
   - Don't share consumer groups between apps
   - Name groups descriptively

4. **Error Handling**
   - Implement retry logic with exponential backoff
   - Handle transient failures gracefully
   - Monitor and alert on persistent errors

5. **Checkpointing**
   - Checkpoint frequently (but not after every event)
   - Balance between fault tolerance and performance
   - Checkpoint after processing batches

### Performance Optimization

- **Compression**: Compress event data before sending
- **Batching**: Group multiple events in single send operation
- **Partition Strategy**: Distribute load evenly across partitions
- **Connection Pooling**: Reuse producer/consumer clients
- **Async Operations**: Use async/await patterns

### Security Best Practices

- ✅ Use managed identities instead of connection strings
- ✅ Enable VNet integration and private endpoints
- ✅ Configure IP firewall rules
- ✅ Use Azure RBAC for fine-grained access control
- ✅ Rotate access keys regularly
- ✅ Enable diagnostic logging
- ✅ Use TLS 1.2 or higher

---

## Exam Tips for AZ-204

### Key Concepts to Remember

1. **Event Hubs** = big data streaming platform (millions events/second)
2. **Partitions** = ordered sequences, scale unit (cannot change after creation)
3. **Consumer Groups** = independent views of Event Hub
4. **Throughput Units** = capacity units (1 TU = 1 MB/s ingress, 2 MB/s egress)
5. **Checkpointing** = tracking processing progress (requires blob storage)
6. **Kafka Compatible** = no code changes for Kafka applications
7. **Retention** = 1-90 days (tier-dependent)

### Common Exam Scenarios

**Scenario 1**: Ingest IoT telemetry at scale
- ✅ Use Event Hubs (designed for high-volume ingestion)
- ❌ Don't use Event Grid (not for streaming)

**Scenario 2**: Multiple applications processing same events
- ✅ Create separate consumer groups for each application
- ❌ Don't share consumer group (causes conflicts)

**Scenario 3**: Need event ordering
- ✅ Use partition key (ordering within partition)
- ❌ Don't expect ordering across partitions

**Scenario 4**: Migrate Kafka workloads to Azure
- ✅ Use Event Hubs Kafka endpoint
- ✅ No code changes required

**Scenario 5**: Scale consumer processing
- ✅ Use EventProcessorClient with multiple instances
- ✅ Automatic load balancing across partitions

### Remember for Exam

- **Cannot change partition count** after Event Hub creation
- **1 MB maximum event size**
- **Throughput Unit**: 1 MB/s ingress, 2 MB/s egress
- **Consumer groups**: Up to 20 per Event Hub (Standard)
- **Partition key**: Determines partition assignment
- **EventProcessorClient**: Recommended for production (auto load balancing)
- **Checkpointing**: Requires Azure Blob Storage
- **Kafka port**: 9093 (SASL_SSL)

### Quick Command Reference

```bash
# Create namespace
az eventhubs namespace create --name <ns> --resource-group <rg> --sku Standard

# Create event hub
az eventhubs eventhub create --name <eh> --namespace-name <ns> --resource-group <rg> --partition-count 4

# Create consumer group
az eventhubs eventhub consumer-group create --name <cg> --eventhub-name <eh> --namespace-name <ns> --resource-group <rg>

# Get connection string
az eventhubs namespace authorization-rule keys list --name RootManageSharedAccessKey --namespace-name <ns> --resource-group <rg>
```

---

## Summary

**Azure Event Hubs** is a fully managed, real-time data streaming platform for big data scenarios.

**Core Components:**
- **Namespace**: Management container
- **Event Hub**: Append-only distributed log
- **Partitions**: Ordered sequences (scale unit)
- **Consumer Groups**: Independent views
- **Producers**: Send events
- **Consumers**: Process events

**Key Features:**
- Millions of events per second
- 1-90 days retention
- Apache Kafka compatible
- Partitioned consumer model
- Auto-scaling with throughput units

**Use Event Hubs for:**
- IoT telemetry ingestion
- Application logging and metrics
- Clickstream analytics
- Real-time dashboards
- Big data pipelines
- Kafka workload migration