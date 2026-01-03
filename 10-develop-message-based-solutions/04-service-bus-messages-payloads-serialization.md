# Service Bus Messages, Payloads, and Serialization

## Message Structure

A Service Bus message consists of a **binary payload** (data section) and **metadata** (properties).

### Message Anatomy

```
Service Bus Message
══════════════════════════════════════════════════════════════

┌──────────────────────────────────────────────────────────┐
│                     MESSAGE METADATA                      │
├──────────────────────────────────────────────────────────┤
│  Broker Properties (System-Defined)                       │
│  ├── MessageId: "order-12345"                            │
│  ├── CorrelationId: "correlation-abc"                    │
│  ├── SessionId: "session-001"                            │
│  ├── To: "processQueue"                                  │
│  ├── ReplyTo: "replyQueue"                               │
│  ├── ReplyToSessionId: "reply-session-001"               │
│  ├── ContentType: "application/json;charset=utf-8"       │
│  ├── Label/Subject: "OrderCreated"                       │
│  ├── TimeToLive: PT10M                                   │
│  └── ScheduledEnqueueTime: 2024-01-15T10:00:00Z          │
│                                                           │
│  User Properties (Application-Defined)                    │
│  ├── Priority: "High"                                    │
│  ├── Region: "US-West"                                   │
│  ├── OrderType: "Express"                                │
│  ├── CustomerId: "12345"                                 │
│  └── ...                                                 │
└──────────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────────┐
│                     BINARY PAYLOAD                        │
│  (Opaque to Service Bus - Any Format)                    │
├──────────────────────────────────────────────────────────┤
│  {                                                        │
│    "orderId": "12345",                                   │
│    "customerId": "67890",                                │
│    "items": [...],                                       │
│    "total": 599.99                                       │
│  }                                                        │
└──────────────────────────────────────────────────────────┘
```

### Key Characteristics

| Component | Description | Visibility |
|-----------|-------------|------------|
| **Binary Payload** | Message data (any format) | Opaque to Service Bus |
| **Broker Properties** | System-defined metadata | Used for routing and processing |
| **User Properties** | Application-defined key-value pairs | Used for filtering and routing |

---

## Broker Properties (System-Defined)

Broker properties are **system-defined** fields that control message behavior and routing.

### Essential Broker Properties

| Property | Type | Description | Use Case |
|----------|------|-------------|----------|
| **MessageId** | string | Unique message identifier | Duplicate detection, idempotency |
| **CorrelationId** | string | Links related messages | Request-reply pattern |
| **SessionId** | string | Groups related messages | FIFO ordering |
| **ContentType** | string | MIME type of payload | Serialization hint |
| **Subject** (Label) | string | Application-specific label | Message classification |
| **To** | string | Destination address | Routing |
| **ReplyTo** | string | Reply queue/topic address | Request-reply pattern |
| **ReplyToSessionId** | string | Reply session identifier | Session-based request-reply |
| **TimeToLive** | TimeSpan | Message expiration | Auto-cleanup |
| **ScheduledEnqueueTimeUtc** | DateTime | Delayed delivery time | Scheduled messages |
| **PartitionKey** | string | Partitioning key | Message grouping |

### MessageId

**Unique identifier** for the message, used for duplicate detection and idempotency.

```csharp
var message = new ServiceBusMessage("Order data");
message.MessageId = $"order-{order.OrderId}";  // Unique identifier

await sender.SendMessageAsync(message);

// Service Bus uses MessageId for duplicate detection
// If sent twice, second message is discarded (if duplicate detection enabled)
```

**Best practices:**
- ✅ Use business identifiers (order ID, transaction ID)
- ✅ Enable duplicate detection on queue/topic
- ✅ Use for idempotent processing

```csharp
// Idempotent processing using MessageId
var orderId = message.MessageId;
if (!await _orderRepository.ExistsAsync(orderId))
{
    await ProcessOrderAsync(orderId);
}
```

### CorrelationId

**Links related messages** together, commonly used in request-reply patterns.

```csharp
// REQUEST: Send message with CorrelationId
var requestMessage = new ServiceBusMessage("Get order details");
requestMessage.CorrelationId = Guid.NewGuid().ToString();
requestMessage.ReplyTo = "replyQueue";  // Where to send reply

await requestSender.SendMessageAsync(requestMessage);

// REPLY: Send response with same CorrelationId
var replyMessage = new ServiceBusMessage("Order details: ...");
replyMessage.CorrelationId = requestMessage.CorrelationId;  // Link to request

await replySender.SendMessageAsync(replyMessage);

// REQUEST SENDER: Match response to request
var replyReceiver = client.CreateReceiver("replyQueue");
var reply = await replyReceiver.ReceiveMessageAsync();

if (reply.CorrelationId == requestMessage.CorrelationId)
{
    // Process matched response
}
```

### SessionId

**Groups related messages** for FIFO processing within a session.

```csharp
// Send messages with same SessionId for FIFO guarantee
var message1 = new ServiceBusMessage("Step 1");
message1.SessionId = "session-customer-123";

var message2 = new ServiceBusMessage("Step 2");
message2.SessionId = "session-customer-123";

var message3 = new ServiceBusMessage("Step 3");
message3.SessionId = "session-customer-123";

// All messages with same SessionId processed in FIFO order
await sender.SendMessageAsync(message1);
await sender.SendMessageAsync(message2);
await sender.SendMessageAsync(message3);

// Receive with session
var sessionReceiver = await client.AcceptSessionAsync(
    queueName,
    "session-customer-123");

// Messages received in order: Step 1, Step 2, Step 3
```

### ContentType

**MIME type** describing payload format for deserialization.

```csharp
// JSON payload
var message = new ServiceBusMessage(JsonSerializer.Serialize(order));
message.ContentType = "application/json;charset=utf-8";

// XML payload
var xmlMessage = new ServiceBusMessage(xmlData);
xmlMessage.ContentType = "application/xml";

// Binary payload
var binaryMessage = new ServiceBusMessage(binaryData);
binaryMessage.ContentType = "application/octet-stream";

// Receiver uses ContentType for deserialization
if (message.ContentType == "application/json;charset=utf-8")
{
    var order = JsonSerializer.Deserialize<Order>(message.Body.ToString());
}
```

### Subject (Label)

**Application-specific label** for message classification.

```csharp
var message = new ServiceBusMessage(JsonSerializer.Serialize(order));
message.Subject = "OrderCreated";  // or use Label property

// Filter by Subject in subscription
// SQL Filter: Subject = 'OrderCreated'
```

### To and ReplyTo

**Routing properties** for request-reply pattern.

```csharp
// REQUEST
var request = new ServiceBusMessage("Query data");
request.To = "dataQueryQueue";        // Target queue
request.ReplyTo = "responseQueue";    // Where to send reply
request.ReplyToSessionId = "reply-session-001";  // Reply session

// REPLY
var reply = new ServiceBusMessage("Query results");
reply.To = request.ReplyTo;  // Send to reply address
```

### TimeToLive

**Message expiration** time after which message is automatically dead-lettered.

```csharp
var message = new ServiceBusMessage("Time-sensitive data");
message.TimeToLive = TimeSpan.FromMinutes(10);  // Expire in 10 minutes

await sender.SendMessageAsync(message);

// If not processed within 10 minutes:
// - Message moves to dead-letter queue (if enabled)
// - Or deleted automatically
```

### ScheduledEnqueueTimeUtc

**Schedule message** for future delivery.

```csharp
// Schedule message for delivery in 1 hour
var message = new ServiceBusMessage("Reminder notification");
var scheduledTime = DateTimeOffset.UtcNow.AddHours(1);

long sequenceNumber = await sender.ScheduleMessageAsync(message, scheduledTime);

// Cancel scheduled message
await sender.CancelScheduledMessageAsync(sequenceNumber);
```

---

## User Properties (Application-Defined)

User properties are **application-defined key-value pairs** used for custom metadata, filtering, and routing.

### Defining User Properties

```csharp
var message = new ServiceBusMessage(JsonSerializer.Serialize(order));

// Add user properties
message.ApplicationProperties["Priority"] = "High";
message.ApplicationProperties["Region"] = "US-West";
message.ApplicationProperties["OrderType"] = "Express";
message.ApplicationProperties["CustomerId"] = 12345;
message.ApplicationProperties["IsUrgent"] = true;
message.ApplicationProperties["Amount"] = 599.99;

await sender.SendMessageAsync(message);
```

### Reading User Properties

```csharp
var message = await receiver.ReceiveMessageAsync();

// Read user properties
if (message.ApplicationProperties.TryGetValue("Priority", out object priorityObj))
{
    string priority = priorityObj.ToString();
    Console.WriteLine($"Priority: {priority}");
}

// Direct access (throws if key doesn't exist)
string region = message.ApplicationProperties["Region"].ToString();
int customerId = (int)message.ApplicationProperties["CustomerId"];
bool isUrgent = (bool)message.ApplicationProperties["IsUrgent"];
```

### Using User Properties for Filtering

```csharp
// Create subscription with filter on user properties
var filter = new SqlRuleFilter(
    "Priority = 'High' AND Region = 'US-West' AND Amount > 500");

await administrationClient.CreateSubscriptionAsync(
    topicName,
    "highPriorityWestCoast",
    new CreateRuleOptions 
    { 
        Name = "HighPriorityFilter", 
        Filter = filter 
    });

// Only messages with Priority='High', Region='US-West', and Amount>500
// will be delivered to this subscription
```

### User Properties Best Practices

✅ **Use for:**
- Filtering and routing
- Message classification
- Business metadata
- Application-specific flags

❌ **Don't use for:**
- Large data (use payload instead)
- Sensitive data without encryption
- Data that changes frequently

```csharp
// ✅ Good: Small metadata
message.ApplicationProperties["Priority"] = "High";
message.ApplicationProperties["Region"] = "US-West";

// ❌ Bad: Large data
message.ApplicationProperties["OrderDetails"] = largeJsonString;  // Use payload instead

// ✅ Good: Payload for large data
var message = new ServiceBusMessage(largeJsonString);
message.ApplicationProperties["Priority"] = "High";
```

---

## Message Routing Patterns

Service Bus supports several messaging patterns using broker properties and user properties.

### 1. Simple Request-Reply

**One-to-one request-response** pattern.

```
Client               Request Queue          Server              Reply Queue
┌──────┐            ┌────────────┐         ┌──────┐            ┌────────────┐
│      │──Request──>│            │────────>│      │            │            │
│      │            │ReplyTo:    │         │      │──Response─>│            │
│      │            │"replyQueue"│         │      │            │            │
│      │<─Response──│            │<────────│      │            │            │
│      │            │CorrelationId         │      │            │            │
└──────┘            └────────────┘         └──────┘            └────────────┘
```

**Implementation:**

```csharp
// CLIENT: Send request
var requestMessage = new ServiceBusMessage("Get customer data");
requestMessage.MessageId = Guid.NewGuid().ToString();
requestMessage.CorrelationId = requestMessage.MessageId;
requestMessage.ReplyTo = "clientReplyQueue";

await requestSender.SendMessageAsync(requestMessage);

// Wait for response
var replyReceiver = client.CreateReceiver("clientReplyQueue");
var response = await replyReceiver.ReceiveMessageAsync();

if (response.CorrelationId == requestMessage.CorrelationId)
{
    var customerData = JsonSerializer.Deserialize<Customer>(response.Body.ToString());
    Console.WriteLine($"Received: {customerData}");
}

// SERVER: Process request and send reply
var requestReceiver = client.CreateReceiver("requestQueue");
var request = await requestReceiver.ReceiveMessageAsync();

// Process request
var customerData = await GetCustomerDataAsync(request.Body.ToString());

// Send reply
var replyMessage = new ServiceBusMessage(JsonSerializer.Serialize(customerData));
replyMessage.CorrelationId = request.CorrelationId;

var replySender = client.CreateSender(request.ReplyTo);
await replySender.SendMessageAsync(replyMessage);
await requestReceiver.CompleteMessageAsync(request);
```

### 2. Multicast Request-Reply

**One-to-many request-response** pattern using topics.

```
Client          Request Topic          Servers (Multiple)      Reply Queue
┌──────┐        ┌────────────┐         ┌──────┐               ┌────────────┐
│      │──Req──>│            │────────>│Srv 1 │──Response1───>│            │
│      │        │ReplyTo:    │         └──────┘               │            │
│      │        │"replyQueue"│         ┌──────┐               │            │
│      │        │            │────────>│Srv 2 │──Response2───>│            │
│      │        │            │         └──────┘               │            │
│      │<──────────────────────────────────────Multiple replies│            │
└──────┘        └────────────┘                                └────────────┘
```

**Use case:** Broadcast query to multiple services, collect responses.

```csharp
// CLIENT: Send request to topic
var requestMessage = new ServiceBusMessage("Query all servers");
requestMessage.CorrelationId = Guid.NewGuid().ToString();
requestMessage.ReplyTo = "clientReplyQueue";

await topicSender.SendMessageAsync(requestMessage);

// Receive multiple responses
var replyReceiver = client.CreateReceiver("clientReplyQueue");
var responses = new List<ServiceBusReceivedMessage>();

for (int i = 0; i < 3; i++)  // Expect 3 responses
{
    var response = await replyReceiver.ReceiveMessageAsync(TimeSpan.FromSeconds(5));
    if (response != null && response.CorrelationId == requestMessage.CorrelationId)
    {
        responses.Add(response);
    }
}

Console.WriteLine($"Received {responses.Count} responses");
```

### 3. Multiplexing

**Multiple logical message streams** through a single queue using sessions.

```
Multiple Senders            Session-Enabled Queue              Receivers
┌──────────┐               ┌─────────────────────┐            ┌──────────┐
│ Client 1 │──Session A──>│ Session A: Msg1-Msg5│──────────>│Receiver A│
└──────────┘               ├─────────────────────┤            └──────────┘
┌──────────┐               │ Session B: Msg1-Msg3│            ┌──────────┐
│ Client 2 │──Session B──>├─────────────────────┤──────────>│Receiver B│
└──────────┘               │ Session C: Msg1-Msg7│            └──────────┘
┌──────────┐               └─────────────────────┘            ┌──────────┐
│ Client 3 │──Session C──>                                   │Receiver C│
└──────────┘                                                  └──────────┘

Each session is an independent FIFO stream
```

**Implementation:**

```csharp
// SENDER: Send to different sessions
var message1 = new ServiceBusMessage("Task for customer A");
message1.SessionId = "session-customer-A";
await sender.SendMessageAsync(message1);

var message2 = new ServiceBusMessage("Task for customer B");
message2.SessionId = "session-customer-B";
await sender.SendMessageAsync(message2);

// RECEIVER: Process sessions in parallel
var tasks = Enumerable.Range(0, 3).Select(async i =>
{
    var sessionReceiver = await client.AcceptNextSessionAsync(queueName);
    Console.WriteLine($"Processing session: {sessionReceiver.SessionId}");
    
    await foreach (var message in sessionReceiver.ReceiveMessagesAsync())
    {
        await ProcessMessageAsync(message);
        await sessionReceiver.CompleteMessageAsync(message);
    }
});

await Task.WhenAll(tasks);
```

### 4. Multiplexed Request-Reply

**Multiple request-reply conversations** sharing a single reply queue using sessions.

```
Multiple Clients        Request Queue         Server          Reply Queue (Sessioned)
┌──────────┐           ┌────────────┐        ┌──────┐        ┌─────────────────────┐
│ Client 1 │──Req1────>│            │───────>│      │        │ReplySession A: Resp1│
└──────────┘           │ReplyToSess:│        │      │────────>├─────────────────────┤
┌──────────┐           │"session-A" │        │      │        │ReplySession B: Resp2│
│ Client 2 │──Req2────>│            │        │      │────────>├─────────────────────┤
└──────────┘           │ReplyToSess:│        │      │        │ReplySession C: Resp3│
┌──────────┐           │"session-B" │        │      │────────>└─────────────────────┘
│ Client 3 │──Req3────>│            │        │      │
└──────────┘           └────────────┘        └──────┘

Each client receives only its own replies (by session)
```

**Implementation:**

```csharp
// CLIENT: Send request with reply session
var requestMessage = new ServiceBusMessage("Process data");
requestMessage.ReplyTo = "sharedReplyQueue";
requestMessage.ReplyToSessionId = $"reply-session-{Guid.NewGuid()}";
requestMessage.CorrelationId = requestMessage.ReplyToSessionId;

await requestSender.SendMessageAsync(requestMessage);

// Receive reply from specific session
var sessionReceiver = await client.AcceptSessionAsync(
    "sharedReplyQueue",
    requestMessage.ReplyToSessionId);

var reply = await sessionReceiver.ReceiveMessageAsync();
Console.WriteLine($"Received reply: {reply.Body}");

// SERVER: Send reply to specific session
var request = await requestReceiver.ReceiveMessageAsync();

var replyMessage = new ServiceBusMessage("Processing complete");
replyMessage.SessionId = request.ReplyToSessionId;  // Target specific session
replyMessage.CorrelationId = request.CorrelationId;

var replySender = client.CreateSender(request.ReplyTo);
await replySender.SendMessageAsync(replyMessage);
```

---

## Payload Serialization

The binary payload can contain data in any format. Use `ContentType` to indicate serialization format.

### JSON Serialization (Recommended)

```csharp
// SEND: JSON payload
var order = new Order 
{ 
    OrderId = 12345, 
    CustomerId = 67890, 
    Total = 599.99 
};

var message = new ServiceBusMessage(JsonSerializer.Serialize(order));
message.ContentType = "application/json;charset=utf-8";

await sender.SendMessageAsync(message);

// RECEIVE: Deserialize JSON
var receivedMessage = await receiver.ReceiveMessageAsync();

if (receivedMessage.ContentType == "application/json;charset=utf-8")
{
    var receivedOrder = JsonSerializer.Deserialize<Order>(
        receivedMessage.Body.ToString());
    
    Console.WriteLine($"Order: {receivedOrder.OrderId}");
}
```

### XML Serialization

```csharp
// SEND: XML payload
var xmlSerializer = new XmlSerializer(typeof(Order));
using var stringWriter = new StringWriter();
xmlSerializer.Serialize(stringWriter, order);

var message = new ServiceBusMessage(stringWriter.ToString());
message.ContentType = "application/xml;charset=utf-8";

await sender.SendMessageAsync(message);

// RECEIVE: Deserialize XML
if (receivedMessage.ContentType == "application/xml;charset=utf-8")
{
    using var stringReader = new StringReader(receivedMessage.Body.ToString());
    var receivedOrder = (Order)xmlSerializer.Deserialize(stringReader);
}
```

### Binary Serialization

```csharp
// SEND: Binary payload
byte[] binaryData = GetBinaryData();

var message = new ServiceBusMessage(binaryData);
message.ContentType = "application/octet-stream";

await sender.SendMessageAsync(message);

// RECEIVE: Binary data
byte[] receivedBinary = receivedMessage.Body.ToArray();
```

### AMQP Serialization

Service Bus natively supports **AMQP type system** for interoperability.

```csharp
// AMQP types automatically serialized
var message = new ServiceBusMessage(new AmqpSerializedData
{
    IntValue = 123,
    StringValue = "Hello",
    BoolValue = true
});

message.ContentType = "application/vnd.microsoft.amqp";
```

### Best Practices

✅ **Always set ContentType**
```csharp
message.ContentType = "application/json;charset=utf-8";
```

✅ **Keep messages small** (< 256 KB for Standard, < 100 MB for Premium)
```csharp
// ✅ Good: Store large data externally
var message = new ServiceBusMessage(JsonSerializer.Serialize(new
{
    OrderId = order.Id,
    DataUrl = "https://storage.blob.core.windows.net/orders/12345.json"
}));

// ❌ Bad: Embed large data (unless Premium tier)
var message = new ServiceBusMessage(largeByteArray);  // > 256 KB
```

✅ **Version your messages**
```csharp
message.ApplicationProperties["MessageVersion"] = "2.0";
message.ApplicationProperties["SchemaVersion"] = "v1";
```

✅ **Compress large payloads**
```csharp
using var memoryStream = new MemoryStream();
using (var gzipStream = new GZipStream(memoryStream, CompressionMode.Compress))
{
    var jsonBytes = Encoding.UTF8.GetBytes(JsonSerializer.Serialize(order));
    gzipStream.Write(jsonBytes, 0, jsonBytes.Length);
}

var message = new ServiceBusMessage(memoryStream.ToArray());
message.ContentType = "application/json+gzip";
```

---

## Best Practices

### 1. Use MessageId for Idempotency

```csharp
// ✅ Set MessageId to business identifier
message.MessageId = $"order-{order.OrderId}";

// Enable duplicate detection
az servicebus queue create --enable-duplicate-detection true
```

### 2. Use CorrelationId for Request-Reply

```csharp
// ✅ Link request and response
var request = new ServiceBusMessage("Query");
request.CorrelationId = Guid.NewGuid().ToString();
request.ReplyTo = "replyQueue";

// Server uses same CorrelationId in reply
reply.CorrelationId = request.CorrelationId;
```

### 3. Use SessionId for Ordering

```csharp
// ✅ Group related messages
message.SessionId = $"customer-{customerId}";
```

### 4. Set ContentType

```csharp
// ✅ Always set ContentType
message.ContentType = "application/json;charset=utf-8";
```

### 5. Use User Properties for Filtering

```csharp
// ✅ Enable efficient routing
message.ApplicationProperties["Priority"] = "High";
message.ApplicationProperties["Region"] = "US-West";

// Create subscription filter
var filter = new SqlRuleFilter("Priority = 'High' AND Region = 'US-West'");
```

---

## Exam Tips for AZ-204

### Key Concepts

1. **MessageId** = Unique identifier, duplicate detection, idempotency
2. **CorrelationId** = Link related messages, request-reply
3. **SessionId** = FIFO ordering, message grouping
4. **ContentType** = Serialization format
5. **User Properties** = Custom metadata, filtering
6. **ReplyTo** = Reply queue address

### Remember

| Property | Used For |
|----------|----------|
| **MessageId** | Duplicate detection, idempotency |
| **CorrelationId** | Request-reply linking |
| **SessionId** | FIFO ordering |
| **ContentType** | Serialization format |
| **ApplicationProperties** | Filtering, routing |
| **ReplyTo** | Reply address |
| **TimeToLive** | Message expiration |

### Common Scenarios

- **Prevent duplicates** → MessageId + duplicate detection
- **Request-reply** → CorrelationId + ReplyTo
- **Ordered processing** → SessionId
- **Route by properties** → ApplicationProperties + SQL filter
- **Schedule delivery** → ScheduledEnqueueTimeUtc

---

## Summary

**Service Bus messages consist of:**
- ✅ Binary payload (any format)
- ✅ Broker properties (system-defined)
- ✅ User properties (application-defined)

**Key broker properties:**
- **MessageId**: Unique identifier
- **CorrelationId**: Link related messages
- **SessionId**: FIFO grouping
- **ContentType**: Serialization format
- **ReplyTo**: Reply address

**Use properties for routing, filtering, and implementing messaging patterns like request-reply!