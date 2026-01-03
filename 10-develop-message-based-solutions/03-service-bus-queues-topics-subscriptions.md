# Service Bus Queues, Topics, and Subscriptions

## Service Bus Queues

**Queues** provide **point-to-point** message delivery using a **competing consumers** pattern. Messages are stored in the queue until a receiver retrieves and processes them.

### Queue Architecture

```
Senders (Multiple)              Queue                Receivers (Multiple)
┌──────────┐                  ┌───────┐            ┌──────────┐
│ Sender 1 │ ───────────────> │ Msg 1 │ ────┐      │Receiver 1│
└──────────┘                  │ Msg 2 │     │  ───>└──────────┘
┌──────────┐                  │ Msg 3 │     └─┐    ┌──────────┐
│ Sender 2 │ ───────────────> │ Msg 4 │       ───> │Receiver 2│
└──────────┘                  │ Msg 5 │       ┌──> └──────────┘
┌──────────┐                  │  ...  │       │    ┌──────────┐
│ Sender 3 │ ───────────────> │       │ ──────┘    │Receiver 3│
└──────────┘                  └───────┘            └──────────┘

Key Characteristics:
• One message delivered to ONE receiver only
• Multiple senders can send to same queue
• Multiple receivers compete for messages (load balancing)
• Messages persist until successfully processed
```

### Key Features

| Feature | Description |
|---------|-------------|
| **FIFO Ordering** | Guaranteed when sessions are enabled (per session) |
| **Competing Consumers** | Multiple receivers process messages in parallel |
| **Load Balancing** | Distribute work evenly across receivers |
| **Load Leveling** | Smooth out traffic bursts |
| **Temporal Decoupling** | Senders and receivers don't need to be online simultaneously |
| **Message Persistence** | Messages stored durably until processed |

### Creating a Queue

```bash
# Create queue with Azure CLI
az servicebus queue create \
  --resource-group myResourceGroup \
  --namespace-name myServiceBusNamespace \
  --name myQueue \
  --max-size 1024 \
  --default-message-time-to-live P14D \
  --lock-duration PT30S \
  --max-delivery-count 10 \
  --enable-duplicate-detection true \
  --duplicate-detection-history-time-window PT10M
```

### Queue Properties

| Property | Description | Default | Range |
|----------|-------------|---------|-------|
| **MaxSizeInMegabytes** | Maximum queue size | 1024 MB | 1024-5120 MB (Standard)<br>1024-81920 MB (Premium) |
| **DefaultMessageTimeToLive** | Message TTL | 14 days | 1 sec to unlimited |
| **LockDuration** | Lock timeout for Peek Lock | 30 sec | 5 sec to 5 min |
| **MaxDeliveryCount** | Max delivery attempts before dead-lettering | 10 | 1 to 2000 |
| **RequiresDuplicateDetection** | Enable duplicate detection | false | true/false |
| **DuplicateDetectionHistoryTimeWindow** | Deduplication window | N/A | 20 sec to 7 days |
| **EnableBatchedOperations** | Enable server-side batching | true | true/false |
| **RequiresSession** | Enable sessions for FIFO | false | true/false |
| **DeadLetteringOnMessageExpiration** | Dead-letter expired messages | false | true/false |

---

## Receive Modes

Service Bus offers two modes for receiving messages, each with different trade-offs between simplicity and reliability.

### Comparison Table

| Feature | Receive and Delete | Peek Lock |
|---------|-------------------|-----------|
| **Delivery Guarantee** | At-most-once | At-least-once |
| **Processing Steps** | 1 step | 2 steps |
| **Simplicity** | ✅ Simple | More complex |
| **Fault Tolerance** | ❌ Not fault-tolerant | ✅ Fault-tolerant |
| **Message Lost on Crash?** | ✅ Yes | ❌ No (auto-redelivery) |
| **Lock/Timeout** | No lock | ✅ Lock with timeout |
| **Best For** | Non-critical data, logging | Critical data, transactions |
| **Performance** | Slightly faster | Slightly slower |

### 1. Receive and Delete Mode

**Simplest model**: Service Bus marks message as consumed when it sends to receiver.

```
Client              Service Bus              Outcome
┌──────┐            ┌──────────┐            
│      │ ─Request─> │  Queue   │            
│      │ <─Message──│ (marks   │  ✅ Success: Message delivered
│      │            │ consumed)│            
│      │ [CRASH]    │          │  ❌ Failure: Message LOST
└──────┘            └──────────┘            

Risk: If application crashes before processing, message is lost forever
```

**Use when:**
- ✅ Data loss is acceptable (e.g., telemetry, logs)
- ✅ Performance is critical
- ✅ Processing is very reliable
- ❌ NOT for critical data

**Code example:**
```csharp
// Receive and Delete mode
var receiver = client.CreateReceiver(
    queueName,
    new ServiceBusReceiverOptions 
    { 
        ReceiveMode = ServiceBusReceiveMode.ReceiveAndDelete 
    });

// Message automatically deleted after receive
var message = await receiver.ReceiveMessageAsync();

// No need to call CompleteMessage
// If crash occurs here, message is LOST
await ProcessMessageAsync(message);
```

```python
# Python: Receive and Delete
from azure.servicebus import ServiceBusClient, ServiceBusReceiveMode

client = ServiceBusClient.from_connection_string(connection_string)
receiver = client.get_queue_receiver(
    queue_name,
    receive_mode=ServiceBusReceiveMode.RECEIVE_AND_DELETE)

with receiver:
    for message in receiver:
        # Message already deleted from queue
        process_message(message)
```

### 2. Peek Lock Mode (Recommended)

**Two-stage receive**: Lock message, process it, then explicitly complete or abandon.

```
Client              Service Bus                   Outcome
┌──────┐            ┌──────────┐            
│      │ ─Request─> │  Queue   │            
│      │ <─Message──│ (LOCKED  │  Lock acquired
│      │            │ 30 sec)  │            
│      │            │          │            
│      │  Process   │          │            
│      │            │          │            
│      │ ─Complete─>│ (delete) │  ✅ Success: Message deleted
│      │            │          │            
│      │ [CRASH]    │          │  ✅ Auto-redelivery after timeout
│      │            │ (unlock  │     Message NOT lost
│      │            │ & retry) │            
└──────┘            └──────────┘            
```

**Workflow:**

1. **Receive**: Message locked (invisible to other receivers) for `LockDuration` (default 30 seconds)
2. **Process**: Application processes the message
3. **Complete**: Explicitly delete message from queue
   - OR **Abandon**: Release lock (message becomes visible again)
   - OR **Dead-letter**: Move to dead-letter queue
   - OR **Defer**: Defer processing to later time

**Lock timeout:**
- Default: 30 seconds
- Configurable: 5 seconds to 5 minutes
- If timeout expires before Complete/Abandon, message automatically unlocked and redelivered
- Lock can be renewed during processing

**Use when:**
- ✅ Critical data (no loss acceptable)
- ✅ Long processing time
- ✅ Transaction support needed
- ✅ Error handling required

**Code example:**
```csharp
// Peek Lock mode (default)
var receiver = client.CreateReceiver(
    queueName,
    new ServiceBusReceiverOptions 
    { 
        ReceiveMode = ServiceBusReceiveMode.PeekLock  // Default
    });

var message = await receiver.ReceiveMessageAsync();

try
{
    // Process message (lock held for 30 seconds by default)
    await ProcessMessageAsync(message);
    
    // Explicitly complete (delete from queue)
    await receiver.CompleteMessageAsync(message);
}
catch (Exception ex)
{
    // Option 1: Abandon (requeue for retry)
    await receiver.AbandonMessageAsync(message);
    
    // Option 2: Dead-letter (move to DLQ)
    await receiver.DeadLetterMessageAsync(
        message,
        deadLetterReason: "ProcessingFailed",
        deadLetterErrorDescription: ex.Message);
    
    // Option 3: Defer (process later)
    await receiver.DeferMessageAsync(message);
}
```

**Renew lock for long processing:**
```csharp
var message = await receiver.ReceiveMessageAsync();

// Start background task to renew lock every 20 seconds
using var cts = new CancellationTokenSource();
var renewTask = Task.Run(async () =>
{
    while (!cts.Token.IsCancellationRequested)
    {
        await Task.Delay(TimeSpan.FromSeconds(20), cts.Token);
        await receiver.RenewMessageLockAsync(message);
    }
}, cts.Token);

try
{
    // Long-running processing (> 30 seconds)
    await LongRunningProcessAsync(message);
    await receiver.CompleteMessageAsync(message);
}
finally
{
    cts.Cancel(); // Stop renewing
}
```

```javascript
// JavaScript: Peek Lock with error handling
const receiver = client.createReceiver(queueName);

async function processMessages() {
    const messages = await receiver.receiveMessages(1);
    const message = messages[0];
    
    try {
        await processMessage(message);
        await receiver.completeMessage(message);
    } catch (error) {
        // Abandon: message will be redelivered
        await receiver.abandonMessage(message);
        
        // Or dead-letter if max retries exceeded
        if (message.deliveryCount >= 3) {
            await receiver.deadLetterMessage(message, {
                deadLetterReason: "MaxRetriesExceeded",
                deadLetterErrorDescription: error.message
            });
        }
    }
}
```

### Delivery Count and Retries

**Delivery count** tracks how many times a message has been delivered.

```csharp
var message = await receiver.ReceiveMessageAsync();

Console.WriteLine($"Delivery count: {message.DeliveryCount}");

if (message.DeliveryCount > 5)
{
    // Give up after 5 retries
    await receiver.DeadLetterMessageAsync(
        message,
        deadLetterReason: "MaxRetriesExceeded");
}
else
{
    try
    {
        await ProcessMessageAsync(message);
        await receiver.CompleteMessageAsync(message);
    }
    catch
    {
        await receiver.AbandonMessageAsync(message); // Retry
    }
}
```

---

## Topics and Subscriptions

**Topics** enable **publish-subscribe (pub/sub)** messaging pattern where messages are broadcast to multiple independent subscribers.

### Topic and Subscription Architecture

```
Publishers                Topic                 Subscriptions                Receivers
┌─────────┐             ┌────────┐           ┌──────────────┐            ┌──────────┐
│ App 1   │ ─────────> │        │ ────────> │Subscription 1│ ────────> │Receiver 1│
└─────────┘             │  Topic │           │(All messages)│            └──────────┘
┌─────────┐             │        │           └──────────────┘            
│ App 2   │ ─────────> │        │           ┌──────────────┐            ┌──────────┐
└─────────┘             │        │ ────────> │Subscription 2│ ────────> │Receiver 2│
┌─────────┐             │        │           │(Filtered)    │            └──────────┘
│ App 3   │ ─────────> │        │           └──────────────┘            
└─────────┘             └────────┘           ┌──────────────┐            ┌──────────┐
                                             │Subscription 3│ ────────> │Receiver 3│
                                             │(Filtered)    │            └──────────┘
                                             └──────────────┘            

Key Characteristics:
• Publishers send to topic (not subscriptions)
• Each subscription receives COPY of messages
• Subscriptions can filter messages
• Multiple receivers per subscription (competing consumers)
```

### Key Differences: Queue vs Topic

| Aspect | Queue | Topic |
|--------|-------|-------|
| **Pattern** | Point-to-point | Publish-subscribe |
| **Message Delivery** | One consumer | Multiple consumers (one per subscription) |
| **Use Case** | Task distribution | Event broadcasting |
| **Message Copy** | Single copy | Copy per subscription |
| **Filtering** | Not supported | Supported (per subscription) |

### Creating Topics and Subscriptions

```bash
# Create topic
az servicebus topic create \
  --resource-group myResourceGroup \
  --namespace-name myServiceBusNamespace \
  --name orderEvents \
  --max-size 1024

# Create subscription (all messages)
az servicebus topic subscription create \
  --resource-group myResourceGroup \
  --namespace-name myServiceBusNamespace \
  --topic-name orderEvents \
  --name allOrders

# Create subscription with SQL filter
az servicebus topic subscription rule create \
  --resource-group myResourceGroup \
  --namespace-name myServiceBusNamespace \
  --topic-name orderEvents \
  --subscription-name highPriorityOrders \
  --name HighPriorityFilter \
  --filter-sql-expression "Priority = 'High'"
```

### Publishing to Topic

```csharp
// Send to topic
await using var client = new ServiceBusClient(connectionString);
var sender = client.CreateSender("orderEvents");

var message = new ServiceBusMessage(JsonSerializer.Serialize(order));
message.ApplicationProperties["Priority"] = "High";
message.ApplicationProperties["OrderType"] = "Express";
message.ApplicationProperties["Region"] = "US-West";

await sender.SendMessageAsync(message);
```

### Receiving from Subscription

```csharp
// Receive from subscription (same as queue)
var receiver = client.CreateReceiver("orderEvents", "highPriorityOrders");

await foreach (var message in receiver.ReceiveMessagesAsync())
{
    var order = JsonSerializer.Deserialize<Order>(message.Body.ToString());
    await ProcessOrderAsync(order);
    await receiver.CompleteMessageAsync(message);
}
```

---

## Message Filtering

Subscriptions can **filter messages** using SQL-like filter expressions. Only messages matching the filter are delivered to that subscription.

### Filter Types

| Filter Type | Description | Example |
|------------|-------------|---------|
| **SQL Filter** | SQL-92 expression on message properties | `Priority = 'High' AND Region = 'US'` |
| **Correlation Filter** | Match specific property values (optimized) | `CorrelationId = '123'` |
| **Boolean Filter** | True filter (all messages) or False filter (no messages) | `TrueFilter`, `FalseFilter` |

### SQL Filter Examples

```csharp
// Filter 1: High priority orders
var filter1 = new SqlRuleFilter("Priority = 'High'");

// Filter 2: Orders from specific region
var filter2 = new SqlRuleFilter("Region = 'US-West'");

// Filter 3: Complex filter (multiple conditions)
var filter3 = new SqlRuleFilter(
    "Priority = 'High' AND (Region = 'US-West' OR Region = 'US-East')");

// Filter 4: Numeric comparison
var filter4 = new SqlRuleFilter("Amount > 1000");

// Filter 5: String operations
var filter5 = new SqlRuleFilter("OrderType LIKE 'Express%'");

// Filter 6: NULL checks
var filter6 = new SqlRuleFilter("CustomProperty IS NOT NULL");

// Create subscription with filter
var ruleOptions = new CreateRuleOptions
{
    Name = "HighPriorityFilter",
    Filter = filter1
};

await administrationClient.CreateSubscriptionAsync(
    topicName,
    subscriptionName,
    ruleOptions);
```

### SQL Filter Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `=` | Equal | `Priority = 'High'` |
| `!=` or `<>` | Not equal | `Status != 'Completed'` |
| `>`, `>=` | Greater than | `Amount > 1000` |
| `<`, `<=` | Less than | `Quantity <= 100` |
| `AND` | Logical AND | `Priority = 'High' AND Region = 'US'` |
| `OR` | Logical OR | `Priority = 'High' OR Priority = 'Critical'` |
| `NOT` | Logical NOT | `NOT (Status = 'Cancelled')` |
| `IN` | In list | `Region IN ('US-West', 'US-East')` |
| `LIKE` | Pattern match | `OrderType LIKE 'Express%'` |
| `IS NULL` | Null check | `CustomField IS NULL` |
| `IS NOT NULL` | Not null check | `CustomField IS NOT NULL` |

### Correlation Filter (Optimized)

**Correlation filters** are optimized for matching specific property values without expression evaluation.

```csharp
// Correlation filter (faster than SQL filter)
var correlationFilter = new CorrelationRuleFilter
{
    ContentType = "application/json",
    CorrelationId = "order-123",
    Subject = "OrderCreated"
};

correlationFilter.ApplicationProperties["Priority"] = "High";
correlationFilter.ApplicationProperties["Region"] = "US-West";

await administrationClient.CreateSubscriptionAsync(
    topicName,
    subscriptionName,
    new CreateRuleOptions 
    { 
        Name = "CorrelationFilter", 
        Filter = correlationFilter 
    });
```

**Performance:**
- ✅ Correlation filters are faster than SQL filters
- ✅ Use correlation filters when matching exact property values
- ✅ Use SQL filters for complex expressions

### Filter Actions

**Actions** modify message properties as they are copied to the subscription.

```csharp
// Create filter with action
var ruleOptions = new CreateRuleOptions
{
    Name = "USWestFilter",
    Filter = new SqlRuleFilter("Region = 'US-West'"),
    Action = new SqlRuleAction("SET ProcessedBy = 'WestRegionProcessor'")
};

await administrationClient.CreateSubscriptionAsync(
    topicName,
    subscriptionName,
    ruleOptions);
```

**Action operations:**
- `SET property = value`: Set or update property
- `REMOVE property`: Remove property

```csharp
// Multiple actions
var action = new SqlRuleAction(
    "SET ProcessedBy = 'Processor1'; SET ProcessedDate = GetDate(); REMOVE TempProperty");
```

---

## Real-World Scenarios

### Scenario 1: Order Processing (Queue)

**Competing consumers** pattern for distributing orders across multiple workers.

```csharp
// Producer (Web API)
var message = new ServiceBusMessage(JsonSerializer.Serialize(order));
message.MessageId = order.OrderId.ToString();
await queueSender.SendMessageAsync(message);

// Consumer 1, 2, 3 (Worker Services) - Compete for messages
var receiver = client.CreateReceiver("orderQueue");
await foreach (var message in receiver.ReceiveMessagesAsync())
{
    var order = JsonSerializer.Deserialize<Order>(message.Body.ToString());
    await ProcessOrderAsync(order);
    await receiver.CompleteMessageAsync(message);
}
```

### Scenario 2: Event Broadcasting (Topic)

**Publish-subscribe** pattern for notifying multiple systems about events.

```
Event Source          Topic: OrderEvents        Subscriptions
┌──────────┐         ┌────────────────┐       ┌──────────────────┐
│          │         │                │──────>│Analytics         │
│ E-commerce│────────>│ OrderCreated   │       │(all events)      │
│ Website  │         │ OrderUpdated   │       └──────────────────┘
│          │         │ OrderCancelled │       ┌──────────────────┐
└──────────┘         │                │──────>│Notifications     │
                     │                │       │(Priority=High)   │
                     │                │       └──────────────────┘
                     │                │       ┌──────────────────┐
                     │                │──────>│Inventory         │
                     └────────────────┘       │(OrderCreated only)│
                                              └──────────────────┘
```

```csharp
// Publisher
var message = new ServiceBusMessage(JsonSerializer.Serialize(orderEvent));
message.ApplicationProperties["EventType"] = "OrderCreated";
message.ApplicationProperties["Priority"] = "High";
await topicSender.SendMessageAsync(message);

// Subscription 1: Analytics (all events)
// No filter - receives all messages

// Subscription 2: Notifications (high priority only)
// Filter: Priority = 'High'

// Subscription 3: Inventory (OrderCreated only)
// Filter: EventType = 'OrderCreated'
```

### Scenario 3: Regional Routing (Topic with Filters)

Route messages to different processors based on region.

```csharp
// Publisher
var message = new ServiceBusMessage(JsonSerializer.Serialize(data));
message.ApplicationProperties["Region"] = "US-West";
await topicSender.SendMessageAsync(message);

// Subscription filters
// US-West subscription: Region = 'US-West'
// US-East subscription: Region = 'US-East'
// EU subscription: Region LIKE 'EU-%'
// Global subscription: TrueFilter (all messages)
```

### Scenario 4: FIFO Processing with Sessions

Guarantee order processing per customer using sessions.

```csharp
// Send with session (FIFO per customer)
var message = new ServiceBusMessage(JsonSerializer.Serialize(order));
message.SessionId = $"customer-{order.CustomerId}";
message.MessageId = $"order-{order.OrderId}";
await sender.SendMessageAsync(message);

// Receive with session
await using var sessionReceiver = await client.AcceptSessionAsync(
    queueName,
    new ServiceBusSessionReceiverOptions());

// Messages for this session processed in FIFO order
await foreach (var message in sessionReceiver.ReceiveMessagesAsync())
{
    var order = JsonSerializer.Deserialize<Order>(message.Body.ToString());
    await ProcessOrderAsync(order); // Guaranteed FIFO per customer
    await sessionReceiver.CompleteMessageAsync(message);
}
```

---

## Best Practices

### 1. Choose Right Entity

✅ **Use Queue when:**
- Point-to-point communication
- Task distribution (competing consumers)
- Load balancing

✅ **Use Topic when:**
- Broadcast to multiple subscribers
- Event notification
- Independent processing by multiple systems

### 2. Implement Idempotency

```csharp
// ✅ Good: Idempotent processing
var orderId = message.MessageId;
if (!await _orderRepository.ExistsAsync(orderId))
{
    await ProcessOrderAsync(orderId);
}
await receiver.CompleteMessageAsync(message);
```

### 3. Handle Errors Gracefully

```csharp
try
{
    await ProcessMessageAsync(message);
    await receiver.CompleteMessageAsync(message);
}
catch (TransientException ex)
{
    // Retry by abandoning
    if (message.DeliveryCount < 5)
    {
        await receiver.AbandonMessageAsync(message);
    }
    else
    {
        await receiver.DeadLetterMessageAsync(message);
    }
}
catch (PermanentException ex)
{
    // Don't retry - dead-letter immediately
    await receiver.DeadLetterMessageAsync(message);
}
```

### 4. Use Sessions for Ordering

```csharp
// ✅ Enable sessions for FIFO
az servicebus queue create \
  --name sessionQueue \
  --enable-session true

// Send with SessionId
message.SessionId = groupIdentifier;
```

### 5. Monitor Dead-Letter Queue

```csharp
// Periodically check DLQ
var dlqReceiver = client.CreateReceiver(
    queueName,
    new ServiceBusReceiverOptions 
    { 
        SubQueue = SubQueue.DeadLetter 
    });

await foreach (var dlqMessage in dlqReceiver.ReceiveMessagesAsync())
{
    Console.WriteLine($"Dead-letter reason: {dlqMessage.DeadLetterReason}");
    // Inspect, fix, and resubmit
}
```

### 6. Optimize Filters

```csharp
// ✅ Good: Correlation filter (faster)
var filter = new CorrelationRuleFilter
{
    ApplicationProperties = { ["Region"] = "US-West" }
};

// ❌ Avoid: Complex SQL expressions (slower)
var filter = new SqlRuleFilter(
    "SQRT(Amount) > 100 AND SUBSTRING(Name, 1, 5) = 'Order'");
```

---

## Exam Tips for AZ-204

### Key Concepts

1. **Queue** = Point-to-point (one consumer per message)
2. **Topic** = Publish-subscribe (multiple consumers)
3. **Peek Lock** = Fault-tolerant (two-stage receive)
4. **Receive and Delete** = Simple but lossy
5. **Sessions** = FIFO guarantee (within session)
6. **Filters** = Route messages to subscriptions

### Remember

| Scenario | Solution |
|----------|----------|
| **Ordered processing** | Queue with sessions |
| **Broadcast events** | Topic with subscriptions |
| **Fault-tolerant processing** | Peek Lock mode |
| **Performance critical, data loss OK** | Receive and Delete mode |
| **Route by properties** | Topic with SQL filters |
| **Distribute tasks** | Queue with competing consumers |

### Common Exam Questions

**Q: How to guarantee FIFO ordering?**
✅ Enable sessions on queue/topic and use SessionId

**Q: How to send events to multiple systems?**
✅ Use topics with multiple subscriptions

**Q: How to prevent message loss on crash?**
✅ Use Peek Lock mode with explicit Complete

**Q: How to route messages based on properties?**
✅ Use topic subscriptions with SQL filters

**Q: How to handle poison messages?**
✅ Check DeliveryCount and dead-letter after max retries

---

## Summary

**Service Bus Queues:**
- ✅ Point-to-point messaging
- ✅ Competing consumers pattern
- ✅ One message to one receiver
- ✅ FIFO with sessions

**Service Bus Topics:**
- ✅ Publish-subscribe pattern
- ✅ One message to multiple subscribers
- ✅ Message filtering per subscription
- ✅ Independent processing

**Receive Modes:**
- **Peek Lock**: Two-stage, fault-tolerant (recommended)
- **Receive and Delete**: One-stage, simple, lossy

**Use sessions for FIFO ordering, filters for routing, and Peek Lock for reliability!