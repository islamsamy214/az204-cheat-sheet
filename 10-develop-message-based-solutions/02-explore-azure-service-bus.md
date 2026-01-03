# Explore Azure Service Bus

## What is Azure Service Bus?

**Azure Service Bus** is a fully managed enterprise message broker with message queues and publish-subscribe topics. It enables decoupling of applications and services, providing reliable message delivery and advanced features for enterprise integration patterns.

### Key Characteristics

```
Azure Service Bus Architecture
═══════════════════════════════════════════════════════════

                    Service Bus Namespace
         ┌──────────────────────────────────────────┐
         │                                          │
         │   Queues (Point-to-Point)                │
         │   ├── Queue 1                            │
         │   ├── Queue 2                            │
         │   └── Queue N                            │
         │                                          │
         │   Topics (Publish-Subscribe)             │
         │   ├── Topic 1                            │
         │   │   ├── Subscription A                 │
         │   │   ├── Subscription B                 │
         │   │   └── Subscription C                 │
         │   └── Topic 2                            │
         │       ├── Subscription X                 │
         │       └── Subscription Y                 │
         │                                          │
         └──────────────────────────────────────────┘

Producers ──────> Service Bus ──────> Consumers
(Senders)         (Broker)            (Receivers)
```

### Core Capabilities

| Capability | Description |
|------------|-------------|
| **Message Queuing** | Store messages until receiving application retrieves them |
| **Load Balancing** | Distribute work across multiple competing consumers |
| **Temporal Decoupling** | Producers and consumers don't need to be online simultaneously |
| **Load Leveling** | Smooth out bursts of traffic |
| **Reliable Delivery** | At-least-once or at-most-once delivery guarantees |
| **Publish-Subscribe** | Broadcast messages to multiple independent subscribers |
| **Advanced Routing** | Filter and route messages based on properties |

---

## Service Bus Concepts

### Namespace

A **namespace** is a container for all messaging components (queues and topics).

**Properties:**
- Unique name (DNS name): `{namespace-name}.servicebus.windows.net`
- Region/location
- Pricing tier (Basic, Standard, Premium)
- Capacity units (Premium only)

```bash
# Create namespace
az servicebus namespace create \
  --resource-group myResourceGroup \
  --name myServiceBusNamespace \
  --location eastus \
  --sku Standard
```

### Queues

**Point-to-point** messaging entities that store messages in **FIFO order** (when sessions are enabled).

**Key features:**
- Competing consumers pattern
- One message delivered to one consumer
- Messages persist until retrieved and deleted
- Load balancing across multiple consumers

```bash
# Create queue
az servicebus queue create \
  --resource-group myResourceGroup \
  --namespace-name myServiceBusNamespace \
  --name myQueue \
  --max-size 1024
```

### Topics and Subscriptions

**Publish-subscribe** pattern where multiple subscribers can receive copies of each message.

**Key features:**
- One-to-many communication
- Each subscription acts like a queue
- Independent message retrieval per subscription
- Filter rules per subscription

```bash
# Create topic
az servicebus topic create \
  --resource-group myResourceGroup \
  --namespace-name myServiceBusNamespace \
  --name myTopic

# Create subscription
az servicebus topic subscription create \
  --resource-group myResourceGroup \
  --namespace-name myServiceBusNamespace \
  --topic-name myTopic \
  --name mySubscription
```

---

## Service Bus Tiers

Azure Service Bus offers three pricing tiers with different capabilities and performance characteristics.

### Tier Comparison

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| **Queues** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Topics/Subscriptions** | ❌ No | ✅ Yes | ✅ Yes |
| **Max Message Size** | 256 KB | 256 KB | 100 MB |
| **Max Queue/Topic Size** | 1 GB | 1-5 GB | 1-80 GB |
| **Throughput** | Low | Variable (shared) | High (dedicated) |
| **Latency** | Standard | Standard | Low (<10 ms) |
| **Resource Isolation** | ❌ Shared | ❌ Shared | ✅ Dedicated CPU/Memory |
| **Transactions** | ❌ No | ✅ Yes | ✅ Yes |
| **Duplicate Detection** | ❌ No | ✅ Yes | ✅ Yes |
| **Sessions** | ❌ No | ✅ Yes | ✅ Yes |
| **Batching** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Auto-Forwarding** | ❌ No | ✅ Yes | ✅ Yes |
| **Scheduled Messages** | ❌ No | ✅ Yes | ✅ Yes |
| **Dead-Letter Queue** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Geo-Disaster Recovery** | ❌ No | ❌ No | ✅ Yes |
| **Availability Zones** | ❌ No | ❌ No | ✅ Yes |
| **Private Endpoints** | ❌ No | ❌ No | ✅ Yes |
| **JMS 2.0 Support** | ❌ No | ❌ No | ✅ Yes |
| **Pricing Model** | Pay-per-operation | Pay-per-operation | Fixed monthly |
| **Typical Use Case** | Dev/test, simple queues | Production, standard workloads | Mission-critical, high-performance |

### Basic Tier

**Best for:**
- Development and testing
- Simple queue scenarios
- Budget-conscious projects

**Limitations:**
- No topics/subscriptions
- No advanced features (sessions, transactions, etc.)
- Lower throughput

```csharp
// Basic tier usage
await using var client = new ServiceBusClient(connectionString);
var sender = client.CreateSender("myqueue");
await sender.SendMessageAsync(new ServiceBusMessage("Hello"));
```

### Standard Tier

**Best for:**
- Production workloads
- Need topics and subscriptions
- Variable workloads

**Features:**
- Topics and subscriptions (pub/sub)
- Transactions
- Duplicate detection
- Sessions (FIFO)
- Advanced routing

```csharp
// Standard tier with topic
await using var client = new ServiceBusClient(connectionString);
var sender = client.CreateSender("mytopic");

var message = new ServiceBusMessage("Event occurred");
message.ApplicationProperties["EventType"] = "OrderCreated";
await sender.SendMessageAsync(message);
```

### Premium Tier

**Best for:**
- Mission-critical workloads
- Predictable performance required
- High throughput (80,000+ msg/sec)
- Large messages (up to 100 MB)

**Features:**
- Dedicated resources (CPU and memory)
- Predictable latency (<10 ms)
- Resource isolation
- Availability zones
- Geo-disaster recovery
- Private endpoints
- JMS 2.0 support

```csharp
// Premium tier with large message
await using var client = new ServiceBusClient(connectionString);
var sender = client.CreateSender("myqueue");

// Send up to 100 MB message
byte[] largePayload = new byte[50 * 1024 * 1024]; // 50 MB
var message = new ServiceBusMessage(largePayload);
await sender.SendMessageAsync(message);
```

### Capacity Units (Premium Only)

Premium tier uses **Messaging Units (MU)** for capacity:

| Messaging Units | Throughput | Approximate Cost |
|----------------|------------|------------------|
| **1 MU** | ~1,000 msg/sec | ~$667/month |
| **2 MU** | ~2,000 msg/sec | ~$1,334/month |
| **4 MU** | ~4,000 msg/sec | ~$2,668/month |
| **8 MU** | ~8,000 msg/sec | ~$5,336/month |

---

## Common Messaging Scenarios

### 1. Messaging (Asynchronous Communication)

Decouple applications using queues for reliable message transfer.

```
Producer                Queue               Consumer
┌────────┐            ┌──────┐            ┌────────┐
│ Web    │ ─────────> │ Msg1 │ ─────────> │Worker  │
│ API    │            │ Msg2 │            │Service │
└────────┘            │ Msg3 │            └────────┘
                      └──────┘
```

**Use case:** Web API queues orders for background processing.

```csharp
// Producer (Web API)
var message = new ServiceBusMessage(JsonSerializer.Serialize(order));
message.MessageId = order.OrderId.ToString();
await sender.SendMessageAsync(message);

// Consumer (Worker Service)
await foreach (ServiceBusReceivedMessage message in receiver.ReceiveMessagesAsync())
{
    var order = JsonSerializer.Deserialize<Order>(message.Body.ToString());
    await ProcessOrderAsync(order);
    await receiver.CompleteMessageAsync(message);
}
```

### 2. Decoupling Applications

Separate application tiers to enable independent scaling and deployment.

```
Front-End              Service Bus          Back-End Services
┌────────┐            ┌──────────┐         ┌─────────────┐
│        │            │  Queue   │         │  Inventory  │
│  Web   │ ────────> │  Topic   │ ─────> │  Payment    │
│  App   │            │          │         │  Shipping   │
└────────┘            └──────────┘         └─────────────┘
```

**Benefits:**
- Front-end doesn't wait for back-end processing
- Back-end services can be updated independently
- Failures don't cascade across tiers

### 3. Topics and Subscriptions (Fan-Out)

Broadcast events to multiple independent subscribers.

```
Publisher              Topic                Subscriptions
┌────────┐            ┌──────┐            ┌──────────────┐
│        │            │      │───────────>│ Analytics    │
│ Event  │ ────────> │ Topic│            └──────────────┘
│ Source │            │      │───────────>┌──────────────┐
└────────┘            │      │            │ Notifications│
                      │      │            └──────────────┘
                      └──────┘───────────>┌──────────────┐
                                          │ Archival     │
                                          └──────────────┘
```

**Use case:** Order events sent to analytics, notifications, and archival systems.

```csharp
// Publisher
var message = new ServiceBusMessage(JsonSerializer.Serialize(orderEvent));
message.ApplicationProperties["EventType"] = "OrderCreated";
message.ApplicationProperties["Priority"] = "High";
await topicSender.SendMessageAsync(message);

// Subscription 1: Analytics (all events)
// Subscription 2: Notifications (only High priority)
// Subscription 3: Archival (all events)
```

### 4. Message Sessions (FIFO Processing)

Process related messages in order using sessions.

```
Producer               Session-Enabled Queue        Consumer
┌────────┐            ┌─────────────────────┐      ┌────────┐
│        │ ──Order1─> │ Session: Customer1  │ ───> │        │
│ Order  │ ──Order2─> │ - Order1            │      │Process │
│ System │ ──Order3─> │ - Order2            │      │in FIFO │
│        │            │                     │      │order   │
│        │ ──Order4─> │ Session: Customer2  │      │        │
│        │ ──Order5─> │ - Order4            │      │        │
└────────┘            │ - Order5            │      └────────┘
                      └─────────────────────┘
```

**Use case:** Process orders for each customer in sequence.

```csharp
// Send with session
var message = new ServiceBusMessage(JsonSerializer.Serialize(order));
message.SessionId = $"customer-{order.CustomerId}";
await sender.SendMessageAsync(message);

// Receive with session
await using var sessionReceiver = await client.AcceptSessionAsync(
    queueName, 
    sessionId: $"customer-{customerId}");

await foreach (var message in sessionReceiver.ReceiveMessagesAsync())
{
    // Messages processed in FIFO order per session
    await ProcessOrderAsync(message);
    await sessionReceiver.CompleteMessageAsync(message);
}
```

---

## Advanced Features

### 1. Message Sessions

**Enable FIFO guarantee** by grouping related messages using `SessionId`.

**Features:**
- Strict ordering within a session
- Session state storage
- Single receiver per session
- Parallel processing across sessions

```csharp
// Enable sessions on queue
az servicebus queue create \
  --resource-group myResourceGroup \
  --namespace-name myServiceBusNamespace \
  --name sessionQueue \
  --enable-session true

// Send with session
var message = new ServiceBusMessage("Order data");
message.SessionId = "session-123";
await sender.SendMessageAsync(message);

// Receive with session
var sessionReceiver = await client.AcceptSessionAsync("sessionQueue", "session-123");
var message = await sessionReceiver.ReceiveMessageAsync();
```

### 2. Auto-Forwarding

**Automatically forward** messages from one queue/subscription to another queue/topic.

**Use cases:**
- Chain processing steps
- Aggregate messages from multiple sources
- Route messages to different regions

```bash
# Create auto-forwarding rule
az servicebus queue create \
  --resource-group myResourceGroup \
  --namespace-name myServiceBusNamespace \
  --name sourceQueue \
  --forward-to destinationQueue
```

```
Source Queue    Auto-Forward     Destination Queue
┌──────────┐   ──────────────>  ┌────────────┐
│  Step 1  │                    │   Step 2   │
└──────────┘                    └────────────┘
```

### 3. Dead-Letter Queue (DLQ)

**Automatically move** unprocessable messages to a separate queue for inspection.

**Messages are dead-lettered when:**
- Max delivery count exceeded
- Message expired (TTL)
- Session lock lost
- Explicitly dead-lettered by receiver
- Filter evaluation fails

```csharp
// Process with dead-lettering
try
{
    await ProcessMessageAsync(message);
    await receiver.CompleteMessageAsync(message);
}
catch (Exception ex)
{
    // Move to dead-letter queue
    await receiver.DeadLetterMessageAsync(
        message, 
        deadLetterReason: "ProcessingFailed",
        deadLetterErrorDescription: ex.Message);
}

// Process dead-letter queue
var dlqReceiver = client.CreateReceiver(
    queueName, 
    new ServiceBusReceiverOptions 
    { 
        SubQueue = SubQueue.DeadLetter 
    });

await foreach (var dlqMessage in dlqReceiver.ReceiveMessagesAsync())
{
    // Inspect, fix, and resubmit
    Console.WriteLine($"Dead-letter reason: {dlqMessage.DeadLetterReason}");
    Console.WriteLine($"Description: {dlqMessage.DeadLetterErrorDescription}");
}
```

### 4. Scheduled Delivery

**Schedule messages** for future delivery.

```csharp
// Schedule message for delivery in 1 hour
var message = new ServiceBusMessage("Reminder: Meeting in 10 minutes");
var scheduleTime = DateTimeOffset.UtcNow.AddHours(1);

long sequenceNumber = await sender.ScheduleMessageAsync(message, scheduleTime);

// Cancel scheduled message
await sender.CancelScheduledMessageAsync(sequenceNumber);
```

### 5. Message Deferral

**Defer message processing** to retrieve later by sequence number.

```csharp
// Defer message
long sequenceNumber = message.SequenceNumber;
await receiver.DeferMessageAsync(message);

// Retrieve deferred message later
var deferredMessage = await receiver.ReceiveDeferredMessageAsync(sequenceNumber);
```

### 6. Transactions

**Group operations** into atomic transaction scopes.

```csharp
using var ts = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);

// All operations succeed or fail together
await receiver.CompleteMessageAsync(message1);
await sender.SendMessageAsync(message2);
await receiver.CompleteMessageAsync(message3);

ts.Complete(); // Commit transaction
```

### 7. Duplicate Detection

**Automatically detect** and discard duplicate messages.

**Enable duplicate detection:**
```bash
az servicebus queue create \
  --resource-group myResourceGroup \
  --namespace-name myServiceBusNamespace \
  --name dedupeQueue \
  --enable-duplicate-detection true \
  --duplicate-detection-history-time-window 10
```

**Send with MessageId:**
```csharp
var message = new ServiceBusMessage("Order data");
message.MessageId = $"order-{order.OrderId}"; // Used for deduplication

// If sent twice within detection window, second message is discarded
await sender.SendMessageAsync(message);
```

### 8. Filtering and Actions

**Filter messages** in topic subscriptions using SQL-like expressions.

```csharp
// Create subscription with filter
var options = new CreateSubscriptionOptions("myTopic", "highPrioritySubscription");
var rule = new CreateRuleOptions("HighPriorityFilter")
{
    Filter = new SqlRuleFilter("Priority = 'High'")
};

await administrationClient.CreateSubscriptionAsync(options, rule);

// Send message with property
var message = new ServiceBusMessage("Important event");
message.ApplicationProperties["Priority"] = "High";
await sender.SendMessageAsync(message);
```

### 9. Security

**Authentication options:**
- **Shared Access Signatures (SAS)**: Token-based access
- **Azure AD (RBAC)**: Role-based access control
- **Managed Identity**: Identity-based authentication

```csharp
// Using Azure AD (Managed Identity)
var credential = new DefaultAzureCredential();
await using var client = new ServiceBusClient(
    "myNamespace.servicebus.windows.net",
    credential);

// Using SAS connection string
await using var client = new ServiceBusClient(connectionString);
```

**Azure RBAC roles:**
- `Azure Service Bus Data Owner`: Full access
- `Azure Service Bus Data Sender`: Send messages only
- `Azure Service Bus Data Receiver`: Receive messages only

### 10. Geo-Disaster Recovery (Premium Only)

**Pair namespaces** across regions for disaster recovery.

```bash
# Create geo-pairing
az servicebus georecovery-alias create \
  --resource-group myResourceGroup \
  --namespace-name primaryNamespace \
  --alias myAlias \
  --partner-namespace secondaryNamespaceResourceId
```

**Features:**
- Metadata replication (queues, topics, subscriptions)
- Automatic failover
- Single connection string (alias)
- No message replication (use separately for data redundancy)

---

## Protocols

### AMQP 1.0 (Recommended)

**Advanced Message Queuing Protocol** - Open ISO/IEC standard protocol.

**Benefits:**
- Binary protocol (efficient)
- Multiplexing (multiple sessions over single TCP connection)
- Cross-platform
- Long-lived connections
- Supports transactions

```csharp
// Uses AMQP by default
await using var client = new ServiceBusClient(connectionString);
```

### HTTP/REST

**HTTP-based protocol** for RESTful operations.

**Benefits:**
- Firewall-friendly (port 443)
- Simpler for debugging
- Works with any HTTP client

**Limitations:**
- Higher overhead than AMQP
- No long-polling
- New connection per request

```bash
# REST API example
curl -X POST "https://myNamespace.servicebus.windows.net/myQueue/messages" \
  -H "Authorization: SharedAccessSignature sr=..." \
  -H "Content-Type: application/json" \
  -d '{"body": "Hello World"}'
```

### JMS 2.0 (Premium Only)

**Java Message Service** - Java/Jakarta EE standard API.

**Benefits:**
- Java/Jakarta EE compliance
- Portable across JMS providers
- Enterprise Java integration

```java
// JMS 2.0 with Service Bus
ConnectionFactory factory = new ServiceBusJmsConnectionFactory(connectionString);
Connection connection = factory.createConnection();
Session session = connection.createSession();

Queue queue = session.createQueue("myQueue");
MessageProducer producer = session.createProducer(queue);

TextMessage message = session.createTextMessage("Hello World");
producer.send(message);
```

---

## Client Libraries

### .NET (Azure.Messaging.ServiceBus)

```bash
dotnet add package Azure.Messaging.ServiceBus
```

```csharp
await using var client = new ServiceBusClient(connectionString);

// Send
var sender = client.CreateSender(queueName);
await sender.SendMessageAsync(new ServiceBusMessage("Hello"));

// Receive
var receiver = client.CreateReceiver(queueName);
var message = await receiver.ReceiveMessageAsync();
await receiver.CompleteMessageAsync(message);
```

### Java (azure-messaging-servicebus)

```xml
<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-messaging-servicebus</artifactId>
    <version>7.13.0</version>
</dependency>
```

```java
ServiceBusClient client = new ServiceBusClientBuilder()
    .connectionString(connectionString)
    .buildClient();

// Send
ServiceBusSender sender = client.createSender(queueName);
sender.sendMessage(new ServiceBusMessage("Hello"));

// Receive
ServiceBusReceiver receiver = client.createReceiver(queueName);
ServiceBusReceivedMessage message = receiver.receiveMessages(1).iterator().next();
receiver.complete(message);
```

### Python (azure-servicebus)

```bash
pip install azure-servicebus
```

```python
from azure.servicebus import ServiceBusClient, ServiceBusMessage

client = ServiceBusClient.from_connection_string(connection_string)

# Send
with client.get_queue_sender(queue_name) as sender:
    sender.send_messages(ServiceBusMessage("Hello"))

# Receive
with client.get_queue_receiver(queue_name) as receiver:
    for message in receiver:
        print(message)
        receiver.complete_message(message)
```

### JavaScript/TypeScript (@azure/service-bus)

```bash
npm install @azure/service-bus
```

```typescript
import { ServiceBusClient } from "@azure/service-bus";

const client = new ServiceBusClient(connectionString);

// Send
const sender = client.createSender(queueName);
await sender.sendMessages({ body: "Hello" });

// Receive
const receiver = client.createReceiver(queueName);
const messages = await receiver.receiveMessages(1);
await receiver.completeMessage(messages[0]);
```

---

## Best Practices

### 1. Connection Management

✅ **Reuse ServiceBusClient**
```csharp
// ✅ Good: Singleton client
private static ServiceBusClient _client = new ServiceBusClient(connectionString);

// ❌ Bad: Create new client per operation
var client = new ServiceBusClient(connectionString); // Don't do this repeatedly
```

### 2. Error Handling

✅ **Implement retry logic**
```csharp
var options = new ServiceBusClientOptions
{
    RetryOptions = new ServiceBusRetryOptions
    {
        MaxRetries = 3,
        Delay = TimeSpan.FromSeconds(1),
        MaxDelay = TimeSpan.FromSeconds(30),
        Mode = ServiceBusRetryMode.Exponential
    }
};

var client = new ServiceBusClient(connectionString, options);
```

### 3. Message Size

✅ **Keep messages small** (< 256 KB)
```csharp
// ✅ Good: Store large data externally
var message = new ServiceBusMessage(JsonSerializer.Serialize(new
{
    OrderId = order.Id,
    BlobUrl = "https://storage.blob.core.windows.net/orders/123"
}));

// ❌ Bad: Embed large data in message (unless Premium tier)
var message = new ServiceBusMessage(largeByteArray); // > 256 KB
```

### 4. Idempotency

✅ **Design for duplicate processing**
```csharp
// ✅ Good: Idempotent processing
var orderId = message.MessageId;
if (!await _orderRepository.ExistsAsync(orderId))
{
    await ProcessOrderAsync(orderId);
}
```

### 5. Monitoring

✅ **Enable diagnostics and metrics**
```bash
az monitor diagnostic-settings create \
  --resource /subscriptions/.../namespaces/myNamespace \
  --name myDiagnostics \
  --logs '[{"category": "OperationalLogs", "enabled": true}]' \
  --metrics '[{"category": "AllMetrics", "enabled": true}]' \
  --workspace /subscriptions/.../workspaces/myWorkspace
```

### 6. Scaling

✅ **Use multiple receivers for parallel processing**
```csharp
// Scale out with multiple receivers
var tasks = Enumerable.Range(0, 10).Select(async i =>
{
    var receiver = client.CreateReceiver(queueName);
    await ProcessMessagesAsync(receiver);
});

await Task.WhenAll(tasks);
```

---

## Exam Tips for AZ-204

### Key Concepts

1. **Service Bus = Enterprise messaging** (FIFO, transactions, pub/sub)
2. **Queues = Point-to-point** (one consumer per message)
3. **Topics = Publish-subscribe** (multiple consumers)
4. **Sessions = FIFO ordering** (within session)
5. **Premium = Dedicated resources** (predictable performance)

### Remember

| Feature | Requires |
|---------|----------|
| **Topics/Subscriptions** | Standard or Premium tier |
| **FIFO Ordering** | Sessions enabled |
| **Transactions** | Standard or Premium tier |
| **Duplicate Detection** | Standard or Premium tier |
| **100 MB Messages** | Premium tier only |
| **Geo-DR** | Premium tier only |
| **JMS 2.0** | Premium tier only |

### Common Scenarios

- **Ordered processing** → Sessions with SessionId
- **Broadcast events** → Topics with subscriptions
- **Mission-critical** → Premium tier
- **Large messages** → Premium tier (100 MB)
- **Decouple apps** → Queues or topics
- **Filter messages** → Topic subscriptions with SQL filters

### Quick Reference

```csharp
// Send
var client = new ServiceBusClient(connectionString);
var sender = client.CreateSender(queueOrTopic);
await sender.SendMessageAsync(new ServiceBusMessage("data"));

// Receive
var receiver = client.CreateReceiver(queueOrSubscription);
var message = await receiver.ReceiveMessageAsync();
await receiver.CompleteMessageAsync(message);

// Session
var sessionReceiver = await client.AcceptSessionAsync(queue, sessionId);
var message = await sessionReceiver.ReceiveMessageAsync();
```

---

## Summary

**Azure Service Bus** is a fully managed enterprise message broker providing:
- ✅ Reliable message delivery
- ✅ Queues (point-to-point) and topics (publish-subscribe)
- ✅ Three tiers: Basic, Standard, Premium
- ✅ Advanced features: sessions, transactions, duplicate detection
- ✅ Multiple protocols: AMQP 1.0, HTTP/REST, JMS 2.0
- ✅ Enterprise-grade: geo-DR, private endpoints, RBAC

Use Service Bus for decoupling applications, load leveling, and building reliable distributed systems!