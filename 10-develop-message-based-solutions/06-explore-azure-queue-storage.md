# Explore Azure Queue Storage

## What is Azure Queue Storage?

**Azure Queue Storage** is a service for storing large numbers of messages that can be accessed from anywhere via authenticated HTTP or HTTPS calls. It provides a simple, cost-effective solution for asynchronous message queuing.

### Key Characteristics

```
Azure Queue Storage Architecture
═══════════════════════════════════════════════════════════

Storage Account: mystorageaccount
├── Blob Storage
├── File Storage
├── Table Storage
└── Queue Storage
    ├── Queue 1: orders
    │   ├── Message 1 (up to 64 KB)
    │   ├── Message 2
    │   └── Message N
    ├── Queue 2: notifications
    │   ├── Message 1
    │   └── Message N
    └── Queue N: tasks
        ├── Message 1
        └── Message N

URL Format:
https://<storage-account>.queue.core.windows.net/<queue-name>

Example:
https://mystorageaccount.queue.core.windows.net/orders
```

### Core Capabilities

| Capability | Description |
|------------|-------------|
| **Simple Queue Model** | Basic FIFO queue (no strict ordering guarantee) |
| **HTTP/HTTPS Access** | REST API accessible from anywhere |
| **Massive Storage** | Store millions of messages (up to 500 TB per account) |
| **Message Size** | Up to 64 KB per message |
| **Cost-Effective** | Low cost for high-volume scenarios |
| **Visibility Timeout** | Hide messages temporarily during processing |
| **Peek Without Lock** | View messages without dequeuing |

---

## Queue Storage Components

### 1. Storage Account

A **storage account** provides a unique namespace for your Azure Storage data.

**URL format:**
```
https://<storage-account-name>.queue.core.windows.net
```

**Account types:**
| Account Type | Performance | Redundancy Options | Use Case |
|--------------|-------------|-------------------|----------|
| **Standard (General-purpose v2)** | Standard | LRS, ZRS, GRS, RA-GRS, GZRS, RA-GZRS | Most scenarios |
| **Premium (Block blobs)** | Premium | LRS, ZRS | Low-latency scenarios |

```bash
# Create storage account
az storage account create \
  --name mystorageaccount \
  --resource-group myResourceGroup \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2
```

### 2. Queue

A **queue** contains a set of messages. All messages must be in a queue.

**Queue naming rules:**
- ✅ Lowercase letters, numbers, hyphens
- ✅ 3-63 characters
- ✅ Must start with letter or number
- ❌ No uppercase, underscores, or special characters

**URL format:**
```
https://<storage-account>.queue.core.windows.net/<queue-name>
```

**Examples:**
```
https://mystorageaccount.queue.core.windows.net/orders
https://mystorageaccount.queue.core.windows.net/notifications
https://mystorageaccount.queue.core.windows.net/image-processing-tasks
```

```bash
# Create queue
az storage queue create \
  --name orders \
  --account-name mystorageaccount \
  --account-key <storage-account-key>
```

### 3. Message

A **message** is a piece of data in any format, up to 64 KB in size.

**Message characteristics:**
| Property | Value |
|----------|-------|
| **Max size** | 64 KB |
| **Format** | String (UTF-8) or byte array |
| **Default TTL** | 7 days |
| **Max TTL** | 7 days (messages created before 2017-07-29)<br>Unlimited (messages created after 2017-07-29) |
| **Visibility timeout** | 30 seconds (default) |

**Message lifecycle:**

```
1. Enqueue         2. Dequeue           3. Process       4. Delete
┌────────┐        ┌────────┐           ┌──────┐        ┌────────┐
│Message │───────>│Visible │──────────>│Invisi│───────>│Deleted │
│Created │        │30s timer│          │ble   │        │        │
└────────┘        └────────┘           └──────┘        └────────┘
                       │                    │
                       │                    └─> If not deleted:
                       │                        Message becomes
                       └─────────────────────> visible again
                                               (auto retry)

Steps:
1. Producer sends message → Queue (visible)
2. Consumer receives message → Queue (invisible for 30s)
3. Consumer processes message
4. Consumer deletes message → Removed from queue

If consumer crashes before step 4:
→ Message becomes visible again after 30s timeout
→ Another consumer can process it (automatic retry)
```

---

## Queue Storage vs Service Bus

### Quick Comparison

| Feature | Queue Storage | Service Bus Queue |
|---------|---------------|-------------------|
| **Max Message Size** | 64 KB | 256 KB (Standard)<br>100 MB (Premium) |
| **Max Queue Size** | 500 TB | Unlimited |
| **Ordering Guarantee** | ❌ No (best-effort) | ✅ Yes (with sessions) |
| **Delivery Guarantee** | At-least-once | At-least-once or at-most-once |
| **Protocol** | HTTP/HTTPS | AMQP, HTTP, SBMP |
| **Transactions** | ❌ No | ✅ Yes |
| **Duplicate Detection** | ❌ No | ✅ Yes |
| **Dead-Letter Queue** | ❌ No (manual) | ✅ Yes (automatic) |
| **Publish-Subscribe** | ❌ No | ✅ Yes (topics) |
| **Message Sessions** | ❌ No | ✅ Yes |
| **TTL** | 7 days (default), unlimited | Unlimited |
| **Pricing** | ~$0.05 per GB/month | Pay per operation or fixed (Premium) |
| **Best For** | Simple queues, cost-effective | Enterprise messaging, advanced features |

### Decision Matrix

| Requirement | Choose Queue Storage | Choose Service Bus |
|-------------|---------------------|-------------------|
| **Simple queue needed** | ✅ Yes | Overkill |
| **Cost-effective** | ✅ Yes | More expensive |
| **> 80 GB storage** | ✅ Yes | Expensive at scale |
| **FIFO ordering** | ❌ No | ✅ Yes (sessions) |
| **Pub/sub pattern** | ❌ No | ✅ Yes (topics) |
| **Transactions** | ❌ No | ✅ Yes |
| **Message > 64 KB** | ❌ No | ✅ Yes |
| **Duplicate detection** | ❌ No | ✅ Yes |

---

## Queue Storage Features

### 1. Message TTL (Time-To-Live)

**Default:** 7 days
**Configurable:** 1 second to 7 days (old API), unlimited (new API)

```csharp
// Send message with custom TTL
await queueClient.SendMessageAsync(
    "Message content",
    timeToLive: TimeSpan.FromHours(1));  // Expires in 1 hour

// Send message with unlimited TTL (-1)
await queueClient.SendMessageAsync(
    "Important message",
    timeToLive: TimeSpan.FromSeconds(-1));  // Never expires
```

### 2. Visibility Timeout

**Concept:** When a message is received, it becomes **invisible** to other consumers for a specified time (default 30 seconds).

**Why?** Prevents multiple consumers from processing the same message simultaneously.

```
Consumer 1               Queue                    Consumer 2
┌────────┐              ┌──────┐                 ┌────────┐
│        │──Receive────>│ Msg  │                 │        │
│        │              │(invisible              │        │
│        │              │30 sec)│                 │        │
│        │              │      │──Receive────────│Can't   │
│        │              │      │   (no messages) │see msg │
└────────┘              └──────┘                 └────────┘
     │                       │
     │                       │ After 30 seconds:
     └──Delete───────────────│ If not deleted, message
                             │ becomes visible again
```

**Default:** 30 seconds
**Max:** 7 days

```csharp
// Receive message with custom visibility timeout
var messages = await queueClient.ReceiveMessagesAsync(
    maxMessages: 1,
    visibilityTimeout: TimeSpan.FromMinutes(5));  // Invisible for 5 minutes
```

### 3. Peek Messages

**View messages** without removing them or making them invisible.

```csharp
// Peek next message (doesn't affect visibility)
var peekedMessages = await queueClient.PeekMessagesAsync(maxMessages: 10);

foreach (var message in peekedMessages)
{
    Console.WriteLine($"Peeked: {message.Body}");
    // Message still visible to other consumers
}
```

### 4. Update Message

**Modify message content** and extend visibility timeout during processing.

```csharp
// Receive message
var messages = await queueClient.ReceiveMessagesAsync(1);
var message = messages.Value[0];

// Update message content and extend visibility
await queueClient.UpdateMessageAsync(
    message.MessageId,
    message.PopReceipt,
    "Updated content",
    visibilityTimeout: TimeSpan.FromMinutes(5));  // Extend by 5 minutes
```

### 5. Dequeue Count

**Track delivery attempts** using `DequeueCount` property.

```csharp
var messages = await queueClient.ReceiveMessagesAsync(1);
var message = messages.Value[0];

Console.WriteLine($"Delivery attempt: {message.DequeueCount}");

if (message.DequeueCount > 5)
{
    // Give up after 5 retries
    // Move to poison message queue
    await poisonQueueClient.SendMessageAsync(message.Body.ToString());
    await queueClient.DeleteMessageAsync(message.MessageId, message.PopReceipt);
}
```

### 6. Approximate Message Count

**Get queue length** (approximate count of messages).

```csharp
var properties = await queueClient.GetPropertiesAsync();
int approximateMessageCount = properties.Value.ApproximateMessagesCount;

Console.WriteLine($"Approx messages in queue: {approximateMessageCount}");
```

---

## Access Control and Security

### Authentication Methods

#### 1. Storage Account Key (Shared Key)

**Full access** to storage account.

```csharp
using Azure.Storage.Queues;

string connectionString = "DefaultEndpointsProtocol=https;" +
    "AccountName=mystorageaccount;" +
    "AccountKey=<account-key>;" +
    "EndpointSuffix=core.windows.net";

var queueClient = new QueueClient(connectionString, "orders");
```

**Get account key:**
```bash
az storage account keys list \
  --resource-group myResourceGroup \
  --account-name mystorageaccount \
  --query "[0].value" \
  --output tsv
```

#### 2. Shared Access Signature (SAS)

**Limited access** with specific permissions and expiration.

```bash
# Generate SAS token for queue
az storage queue generate-sas \
  --name orders \
  --account-name mystorageaccount \
  --account-key <account-key> \
  --permissions raup \
  --expiry 2024-12-31T23:59:59Z \
  --output tsv
```

**SAS permissions:**
- `r` - Read (peek, receive)
- `a` - Add (send messages)
- `u` - Update (update messages)
- `p` - Process (receive and delete messages)

```csharp
// Use SAS token
string sasUrl = "https://mystorageaccount.queue.core.windows.net/orders?<sas-token>";
var queueClient = new QueueClient(new Uri(sasUrl));
```

#### 3. Azure AD (RBAC)

**Identity-based access** using Azure Active Directory.

```csharp
using Azure.Identity;
using Azure.Storage.Queues;

// Use default Azure credential (managed identity, Azure CLI, etc.)
var credential = new DefaultAzureCredential();
var queueClient = new QueueClient(
    new Uri("https://mystorageaccount.queue.core.windows.net/orders"),
    credential);
```

**Azure RBAC roles:**
- `Storage Queue Data Contributor`: Read, write, delete messages
- `Storage Queue Data Reader`: Read and peek messages
- `Storage Queue Data Message Processor`: Peek, receive, delete messages
- `Storage Queue Data Message Sender`: Send messages only

```bash
# Assign role
az role assignment create \
  --role "Storage Queue Data Contributor" \
  --assignee <user-or-service-principal> \
  --scope /subscriptions/<subscription-id>/resourceGroups/myResourceGroup/providers/Microsoft.Storage/storageAccounts/mystorageaccount
```

---

## Monitoring and Logging

### Storage Analytics

**Enable logging and metrics:**

```bash
# Enable Storage Analytics logging
az storage logging update \
  --account-name mystorageaccount \
  --account-key <account-key> \
  --services q \
  --log rwd \
  --retention 7

# Enable metrics
az storage metrics update \
  --account-name mystorageaccount \
  --account-key <account-key> \
  --services q \
  --hour true \
  --minute false \
  --retention 7
```

**Logs include:**
- Authenticated requests
- Anonymous requests
- Success/failure status
- Error codes
- Request/response details

### Azure Monitor Integration

```bash
# Create diagnostic settings
az monitor diagnostic-settings create \
  --resource /subscriptions/<subscription-id>/resourceGroups/myResourceGroup/providers/Microsoft.Storage/storageAccounts/mystorageaccount/queueServices/default \
  --name "queue-diagnostics" \
  --logs '[{"category": "StorageRead", "enabled": true}, {"category": "StorageWrite", "enabled": true}]' \
  --metrics '[{"category": "Transaction", "enabled": true}]' \
  --workspace /subscriptions/<subscription-id>/resourceGroups/myResourceGroup/providers/Microsoft.OperationalInsights/workspaces/myWorkspace
```

---

## Best Practices

### 1. Use Poison Message Queue

**Handle messages that fail repeatedly:**

```csharp
var message = messages.Value[0];

if (message.DequeueCount > 5)
{
    // Move to poison message queue
    await poisonQueueClient.SendMessageAsync(message.Body.ToString());
    await queueClient.DeleteMessageAsync(message.MessageId, message.PopReceipt);
    Console.WriteLine("Moved to poison queue");
}
```

### 2. Batch Operations

**Improve performance with batching:**

```csharp
// ✅ Good: Batch receive
var messages = await queueClient.ReceiveMessagesAsync(maxMessages: 32);

// ❌ Bad: Single message receive in loop
for (int i = 0; i < 32; i++)
{
    var message = await queueClient.ReceiveMessageAsync();  // 32 round trips!
}
```

### 3. Reuse QueueClient

```csharp
// ✅ Good: Reuse client
private static QueueClient _queueClient = new QueueClient(connectionString, queueName);

// ❌ Bad: Create new client per operation
var client = new QueueClient(connectionString, queueName);  // Don't repeat
```

### 4. Set Appropriate Visibility Timeout

```csharp
// ✅ Match timeout to processing time
var messages = await queueClient.ReceiveMessagesAsync(
    maxMessages: 1,
    visibilityTimeout: TimeSpan.FromMinutes(5));  // 5 minutes to process
```

### 5. Implement Exponential Backoff

```csharp
int retryCount = 0;
while (retryCount < 5)
{
    var messages = await queueClient.ReceiveMessagesAsync(1);
    
    if (messages.Value.Length == 0)
    {
        // Exponential backoff: 1s, 2s, 4s, 8s, 16s
        await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, retryCount)));
        retryCount++;
        continue;
    }
    
    // Process message
    break;
}
```

### 6. Monitor Queue Depth

```csharp
// Alert if queue grows too large
var properties = await queueClient.GetPropertiesAsync();
int count = properties.Value.ApproximateMessagesCount;

if (count > 10000)
{
    // Alert: Queue backlog too large, scale out consumers
    await SendAlertAsync($"Queue depth: {count}");
}
```

---

## Exam Tips for AZ-204

### Key Concepts

1. **Queue Storage** = Simple, cost-effective queue
2. **Message size** = 64 KB max
3. **Queue size** = 500 TB per storage account
4. **Visibility timeout** = 30 seconds default
5. **TTL** = 7 days default, unlimited possible
6. **No FIFO guarantee** = Best-effort ordering

### Remember

| Feature | Queue Storage |
|---------|---------------|
| **Max message size** | 64 KB |
| **Max queue size** | 500 TB per account |
| **Ordering** | No guarantee (best-effort) |
| **Delivery** | At-least-once |
| **TTL** | 7 days (default), unlimited |
| **Visibility timeout** | 30 seconds (default) |
| **Protocol** | HTTP/HTTPS |
| **Best for** | Simple queues, high volume |

### Common Scenarios

- **Cost-effective queue** → Queue Storage
- **> 80 GB messages** → Queue Storage
- **Simple producer-consumer** → Queue Storage
- **Track retries** → DequeueCount
- **Prevent double processing** → Visibility timeout
- **Handle failures** → Poison message queue

---

## Summary

**Azure Queue Storage provides:**
- ✅ Simple HTTP/HTTPS message queue
- ✅ Massive storage capacity (500 TB)
- ✅ Cost-effective solution
- ✅ Messages up to 64 KB
- ✅ Visibility timeout for safe processing
- ✅ Peek without dequeuing
- ✅ Three authentication methods (key, SAS, Azure AD)

**Use Queue Storage for simple, cost-effective queuing scenarios. Use Service Bus for advanced features like FIFO, pub/sub, and transactions!