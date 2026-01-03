# Create and Manage Azure Queue Storage with .NET

## Overview

This guide covers creating and managing Azure Queue Storage using the **Azure.Storage.Queues** client library for .NET. You'll learn all queue operations with complete code examples.

---

## Setup

### NuGet Packages

```bash
# Create new .NET console app
dotnet new console -n QueueStorageDemo
cd QueueStorageDemo

# Add required packages
dotnet add package Azure.Storage.Queues
dotnet add package Azure.Identity  # For Azure AD authentication
```

**Package versions:**
```xml
<ItemGroup>
  <PackageReference Include="Azure.Storage.Queues" Version="12.17.0" />
  <PackageReference Include="Azure.Identity" Version="1.10.0" />
</ItemGroup>
```

### Using Statements

```csharp
using Azure.Storage.Queues;
using Azure.Storage.Queues.Models;
using System;
using System.Threading.Tasks;
```

---

## Create QueueClient

### Option 1: Connection String

```csharp
// Connection string format
string connectionString = "DefaultEndpointsProtocol=https;" +
    "AccountName=<storage-account-name>;" +
    "AccountKey=<storage-account-key>;" +
    "EndpointSuffix=core.windows.net";

string queueName = "orders";

// Create QueueClient
QueueClient queueClient = new QueueClient(connectionString, queueName);
```

**Get connection string:**
```bash
# Get connection string from Azure
az storage account show-connection-string \
  --name mystorageaccount \
  --resource-group myResourceGroup \
  --query connectionString \
  --output tsv
```

### Option 2: Azure AD (Recommended)

```csharp
using Azure.Identity;

// Use default Azure credential (managed identity, Azure CLI, etc.)
var credential = new DefaultAzureCredential();

string storageAccountName = "mystorageaccount";
string queueName = "orders";

// Create QueueClient with Azure AD
QueueClient queueClient = new QueueClient(
    new Uri($"https://{storageAccountName}.queue.core.windows.net/{queueName}"),
    credential);
```

### Option 3: Shared Access Signature (SAS)

```csharp
// Queue URL with SAS token
string queueUrl = "https://mystorageaccount.queue.core.windows.net/orders?sv=2021-06-08&...";

// Create QueueClient
QueueClient queueClient = new QueueClient(new Uri(queueUrl));
```

---

## Create Queue

### CreateIfNotExists (Recommended)

```csharp
// Create queue if it doesn't exist
await queueClient.CreateIfNotExistsAsync();

// Check if created
if (await queueClient.ExistsAsync())
{
    Console.WriteLine($"Queue '{queueClient.Name}' created successfully");
}
else
{
    Console.WriteLine($"Queue '{queueClient.Name}' already exists");
}
```

### Create with Error Handling

```csharp
using Azure;

try
{
    // Attempt to create queue
    Response response = await queueClient.CreateAsync();
    Console.WriteLine($"Queue created. Status: {response.Status}");
}
catch (RequestFailedException ex) when (ex.ErrorCode == "QueueAlreadyExists")
{
    Console.WriteLine("Queue already exists");
}
catch (RequestFailedException ex)
{
    Console.WriteLine($"Error creating queue: {ex.Message}");
    Console.WriteLine($"Error code: {ex.ErrorCode}");
}
```

### QueueServiceClient (Create Multiple Queues)

```csharp
// Create service client
QueueServiceClient serviceClient = new QueueServiceClient(connectionString);

// Create multiple queues
string[] queueNames = { "orders", "notifications", "logs" };

foreach (var queueName in queueNames)
{
    await serviceClient.CreateQueueAsync(queueName);
    Console.WriteLine($"Created queue: {queueName}");
}

// List all queues
await foreach (QueueItem queue in serviceClient.GetQueuesAsync())
{
    Console.WriteLine($"Queue: {queue.Name}");
}
```

---

## Send Messages

### Send Single Message

```csharp
// Send simple text message
await queueClient.SendMessageAsync("Hello, Queue!");

Console.WriteLine("Message sent successfully");
```

### Send Message with Options

```csharp
// Send message with TTL and visibility delay
await queueClient.SendMessageAsync(
    messageText: "Order #12345",
    visibilityTimeout: TimeSpan.FromSeconds(30),  // Invisible for 30 seconds
    timeToLive: TimeSpan.FromHours(1));           // Expires in 1 hour

Console.WriteLine("Message sent with options");
```

### Send Message with Unlimited TTL

```csharp
// Message never expires
await queueClient.SendMessageAsync(
    "Important message",
    timeToLive: TimeSpan.FromSeconds(-1));  // Never expires
```

### Send Binary Message

```csharp
// Convert object to byte array
var order = new Order { OrderId = 12345, Total = 599.99 };
byte[] messageBytes = System.Text.Json.JsonSerializer.SerializeToUtf8Bytes(order);

// Encode as Base64
string messageText = Convert.ToBase64String(messageBytes);

await queueClient.SendMessageAsync(messageText);
```

### Send Multiple Messages

```csharp
// Send batch of messages
for (int i = 1; i <= 10; i++)
{
    await queueClient.SendMessageAsync($"Message {i}");
    Console.WriteLine($"Sent message {i}");
}
```

### Send Message and Get Receipt

```csharp
// SendMessage returns response with message details
Response<SendReceipt> response = await queueClient.SendMessageAsync("Hello");

SendReceipt receipt = response.Value;
Console.WriteLine($"Message ID: {receipt.MessageId}");
Console.WriteLine($"Pop Receipt: {receipt.PopReceipt}");
Console.WriteLine($"Expiration Time: {receipt.ExpirationTime}");
Console.WriteLine($"Next Visible Time: {receipt.NextVisibleOn}");
```

---

## Peek Messages

**Peek messages** without removing them or affecting their visibility.

### Peek Single Message

```csharp
// Peek next message (doesn't dequeue)
Response<PeekedMessage[]> peekedMessages = await queueClient.PeekMessagesAsync(maxMessages: 1);

if (peekedMessages.Value.Length > 0)
{
    PeekedMessage message = peekedMessages.Value[0];
    Console.WriteLine($"Peeked Message ID: {message.MessageId}");
    Console.WriteLine($"Message Text: {message.Body.ToString()}");
    Console.WriteLine($"Dequeue Count: {message.DequeueCount}");
    Console.WriteLine($"Inserted On: {message.InsertedOn}");
}
else
{
    Console.WriteLine("No messages in queue");
}
```

### Peek Multiple Messages

```csharp
// Peek up to 32 messages
Response<PeekedMessage[]> peekedMessages = await queueClient.PeekMessagesAsync(maxMessages: 32);

Console.WriteLine($"Peeked {peekedMessages.Value.Length} messages:");

foreach (PeekedMessage message in peekedMessages.Value)
{
    Console.WriteLine($"  ID: {message.MessageId}");
    Console.WriteLine($"  Text: {message.Body}");
    Console.WriteLine($"  Dequeue Count: {message.DequeueCount}");
    Console.WriteLine();
}
```

---

## Receive Messages

**Receive messages** makes them invisible for processing (default 30 seconds).

### Receive Single Message

```csharp
// Receive next message
Response<QueueMessage[]> messages = await queueClient.ReceiveMessagesAsync(maxMessages: 1);

if (messages.Value.Length > 0)
{
    QueueMessage message = messages.Value[0];
    
    Console.WriteLine($"Message ID: {message.MessageId}");
    Console.WriteLine($"Message Text: {message.Body.ToString()}");
    Console.WriteLine($"Dequeue Count: {message.DequeueCount}");
    Console.WriteLine($"Inserted On: {message.InsertedOn}");
    Console.WriteLine($"Expires On: {message.ExpiresOn}");
    Console.WriteLine($"Next Visible On: {message.NextVisibleOn}");
    Console.WriteLine($"Pop Receipt: {message.PopReceipt}");
    
    // Message is now invisible to other consumers for 30 seconds
}
else
{
    Console.WriteLine("No messages available");
}
```

### Receive with Custom Visibility Timeout

```csharp
// Receive message with 5-minute visibility timeout
Response<QueueMessage[]> messages = await queueClient.ReceiveMessagesAsync(
    maxMessages: 1,
    visibilityTimeout: TimeSpan.FromMinutes(5));

if (messages.Value.Length > 0)
{
    QueueMessage message = messages.Value[0];
    Console.WriteLine($"Message invisible until: {message.NextVisibleOn}");
}
```

### Receive Multiple Messages

```csharp
// Receive up to 32 messages (maximum allowed)
Response<QueueMessage[]> messages = await queueClient.ReceiveMessagesAsync(maxMessages: 32);

Console.WriteLine($"Received {messages.Value.Length} messages");

foreach (QueueMessage message in messages.Value)
{
    Console.WriteLine($"Processing: {message.Body}");
    
    // Process message
    await ProcessMessageAsync(message.Body.ToString());
    
    // Delete after processing
    await queueClient.DeleteMessageAsync(message.MessageId, message.PopReceipt);
}
```

---

## Update Message

**Update message** content and extend visibility timeout during processing.

### Update Message Content

```csharp
// Receive message
var messages = await queueClient.ReceiveMessagesAsync(1);
QueueMessage message = messages.Value[0];

Console.WriteLine($"Original: {message.Body}");

// Update message content
Response<UpdateReceipt> updateResponse = await queueClient.UpdateMessageAsync(
    message.MessageId,
    message.PopReceipt,
    messageText: "Updated message content");

UpdateReceipt receipt = updateResponse.Value;
Console.WriteLine($"Message updated. New pop receipt: {receipt.PopReceipt}");
```

### Extend Visibility Timeout

```csharp
// Receive message
var messages = await queueClient.ReceiveMessagesAsync(1);
QueueMessage message = messages.Value[0];

// Long-running processing...
Console.WriteLine("Processing started...");
await Task.Delay(TimeSpan.FromSeconds(25));  // 25 seconds of 30 second timeout used

// Extend visibility timeout by another 5 minutes
Response<UpdateReceipt> updateResponse = await queueClient.UpdateMessageAsync(
    message.MessageId,
    message.PopReceipt,
    visibilityTimeout: TimeSpan.FromMinutes(5));

Console.WriteLine($"Visibility extended. New timeout: {updateResponse.Value.NextVisibleOn}");

// Continue processing...
await Task.Delay(TimeSpan.FromMinutes(4));
Console.WriteLine("Processing completed");

// Delete message
await queueClient.DeleteMessageAsync(
    message.MessageId,
    updateResponse.Value.PopReceipt);  // Use new pop receipt!
```

### Update with Content and Timeout

```csharp
// Receive message
var messages = await queueClient.ReceiveMessagesAsync(1);
QueueMessage message = messages.Value[0];

// Update both content and visibility
await queueClient.UpdateMessageAsync(
    message.MessageId,
    message.PopReceipt,
    messageText: "Updated content",
    visibilityTimeout: TimeSpan.FromMinutes(10));
```

---

## Delete Messages

### Delete After Processing

```csharp
// Receive message
var messages = await queueClient.ReceiveMessagesAsync(1);
QueueMessage message = messages.Value[0];

try
{
    // Process message
    await ProcessMessageAsync(message.Body.ToString());
    
    // Delete after successful processing
    await queueClient.DeleteMessageAsync(message.MessageId, message.PopReceipt);
    Console.WriteLine("Message deleted successfully");
}
catch (Exception ex)
{
    Console.WriteLine($"Error processing message: {ex.Message}");
    // Don't delete - message will become visible again
}
```

### Delete with Error Handling

```csharp
using Azure;

try
{
    await queueClient.DeleteMessageAsync(messageId, popReceipt);
    Console.WriteLine("Message deleted");
}
catch (RequestFailedException ex) when (ex.ErrorCode == "MessageNotFound")
{
    Console.WriteLine("Message not found (already deleted?)");
}
catch (RequestFailedException ex) when (ex.ErrorCode == "PopReceiptMismatch")
{
    Console.WriteLine("Pop receipt mismatch (message was updated)");
}
catch (RequestFailedException ex)
{
    Console.WriteLine($"Error deleting message: {ex.Message}");
}
```

---

## Get Queue Properties

### Get Message Count

```csharp
// Get queue properties
Response<QueueProperties> properties = await queueClient.GetPropertiesAsync();

// Approximate message count
int approximateMessageCount = properties.Value.ApproximateMessagesCount;

Console.WriteLine($"Approximate messages in queue: {approximateMessageCount}");
Console.WriteLine($"Queue metadata:");
foreach (var metadata in properties.Value.Metadata)
{
    Console.WriteLine($"  {metadata.Key}: {metadata.Value}");
}
```

### Set Queue Metadata

```csharp
// Set metadata
var metadata = new Dictionary<string, string>
{
    { "Department", "Sales" },
    { "Environment", "Production" },
    { "CreatedBy", "Admin" }
};

await queueClient.SetMetadataAsync(metadata);
Console.WriteLine("Metadata set successfully");

// Get metadata
var properties = await queueClient.GetPropertiesAsync();
foreach (var meta in properties.Value.Metadata)
{
    Console.WriteLine($"{meta.Key}: {meta.Value}");
}
```

---

## Delete Queue

### Delete Queue

```csharp
// Delete queue
await queueClient.DeleteAsync();
Console.WriteLine($"Queue '{queueClient.Name}' deleted");
```

### Delete If Exists

```csharp
// Delete queue if it exists
bool deleted = await queueClient.DeleteIfExistsAsync();

if (deleted)
{
    Console.WriteLine("Queue deleted");
}
else
{
    Console.WriteLine("Queue didn't exist");
}
```

---

## Complete Example: Producer-Consumer

### Producer

```csharp
using Azure.Storage.Queues;
using System;
using System.Threading.Tasks;

class Producer
{
    static async Task Main(string[] args)
    {
        string connectionString = "<your-connection-string>";
        string queueName = "orders";

        QueueClient queueClient = new QueueClient(connectionString, queueName);
        await queueClient.CreateIfNotExistsAsync();

        Console.WriteLine("Sending messages...");

        for (int i = 1; i <= 10; i++)
        {
            string message = $"Order #{i} - Amount: ${i * 100}";
            await queueClient.SendMessageAsync(message);
            Console.WriteLine($"Sent: {message}");
            
            await Task.Delay(1000);  // Wait 1 second between messages
        }

        Console.WriteLine("All messages sent!");
    }
}
```

### Consumer

```csharp
using Azure.Storage.Queues;
using Azure.Storage.Queues.Models;
using System;
using System.Threading.Tasks;

class Consumer
{
    static async Task Main(string[] args)
    {
        string connectionString = "<your-connection-string>";
        string queueName = "orders";
        string poisonQueueName = "orders-poison";

        QueueClient queueClient = new QueueClient(connectionString, queueName);
        QueueClient poisonQueueClient = new QueueClient(connectionString, poisonQueueName);
        
        await queueClient.CreateIfNotExistsAsync();
        await poisonQueueClient.CreateIfNotExistsAsync();

        Console.WriteLine("Receiving messages... (Press Ctrl+C to stop)");

        while (true)
        {
            // Receive up to 10 messages
            Response<QueueMessage[]> messages = await queueClient.ReceiveMessagesAsync(
                maxMessages: 10,
                visibilityTimeout: TimeSpan.FromSeconds(30));

            if (messages.Value.Length == 0)
            {
                Console.WriteLine("No messages available. Waiting...");
                await Task.Delay(5000);  // Wait 5 seconds
                continue;
            }

            foreach (QueueMessage message in messages.Value)
            {
                Console.WriteLine($"\nReceived: {message.Body}");
                Console.WriteLine($"  Dequeue count: {message.DequeueCount}");

                try
                {
                    // Process message
                    await ProcessMessageAsync(message.Body.ToString());
                    
                    // Delete after successful processing
                    await queueClient.DeleteMessageAsync(
                        message.MessageId,
                        message.PopReceipt);
                    
                    Console.WriteLine("  ✅ Processed and deleted");
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"  ❌ Error: {ex.Message}");
                    
                    // Check if exceeded max retries
                    if (message.DequeueCount >= 5)
                    {
                        // Move to poison message queue
                        await poisonQueueClient.SendMessageAsync(message.Body.ToString());
                        await queueClient.DeleteMessageAsync(
                            message.MessageId,
                            message.PopReceipt);
                        
                        Console.WriteLine("  ☠️  Moved to poison queue");
                    }
                    else
                    {
                        Console.WriteLine($"  ⚠️  Will retry (attempt {message.DequeueCount}/5)");
                        // Don't delete - message will become visible again
                    }
                }
            }
        }
    }

    static async Task ProcessMessageAsync(string message)
    {
        // Simulate processing
        await Task.Delay(100);
        
        // Simulate random failures for testing
        if (new Random().Next(10) < 2)  // 20% failure rate
        {
            throw new Exception("Simulated processing failure");
        }
        
        Console.WriteLine("  Processing complete");
    }
}
```

---

## Best Practices

### 1. Reuse QueueClient

```csharp
// ✅ Good: Singleton QueueClient
private static readonly QueueClient _queueClient = 
    new QueueClient(connectionString, queueName);

// ❌ Bad: Create new client per operation
var client = new QueueClient(connectionString, queueName);  // Don't repeat
```

### 2. Handle Poison Messages

```csharp
// ✅ Move to poison queue after max retries
if (message.DequeueCount >= 5)
{
    await poisonQueueClient.SendMessageAsync(message.Body.ToString());
    await queueClient.DeleteMessageAsync(message.MessageId, message.PopReceipt);
}
```

### 3. Use Batch Receive

```csharp
// ✅ Good: Receive multiple messages
var messages = await queueClient.ReceiveMessagesAsync(maxMessages: 32);

// ❌ Bad: Receive one at a time in loop
for (int i = 0; i < 32; i++)
{
    var message = await queueClient.ReceiveMessagesAsync(1);  // 32 round trips!
}
```

### 4. Set Appropriate Visibility Timeout

```csharp
// ✅ Match timeout to expected processing time
var messages = await queueClient.ReceiveMessagesAsync(
    maxMessages: 1,
    visibilityTimeout: TimeSpan.FromMinutes(5));  // 5 minutes
```

### 5. Implement Exponential Backoff

```csharp
// ✅ Wait longer between polls when queue is empty
int retryCount = 0;
while (true)
{
    var messages = await queueClient.ReceiveMessagesAsync(32);
    
    if (messages.Value.Length == 0)
    {
        int delay = (int)Math.Min(Math.Pow(2, retryCount) * 1000, 30000);
        await Task.Delay(delay);  // 1s, 2s, 4s, 8s, 16s, 30s (max)
        retryCount++;
        continue;
    }
    
    retryCount = 0;  // Reset on successful receive
    // Process messages...
}
```

### 6. Monitor Queue Depth

```csharp
// ✅ Alert when queue grows too large
var properties = await queueClient.GetPropertiesAsync();
int count = properties.Value.ApproximateMessagesCount;

if (count > 10000)
{
    await SendAlertAsync($"Queue backlog: {count} messages");
}
```

---

## Exam Tips for AZ-204

### Key APIs

| Operation | Method |
|-----------|--------|
| **Create queue** | `CreateIfNotExistsAsync()` |
| **Send message** | `SendMessageAsync()` |
| **Peek message** | `PeekMessagesAsync()` |
| **Receive message** | `ReceiveMessagesAsync()` |
| **Update message** | `UpdateMessageAsync()` |
| **Delete message** | `DeleteMessageAsync()` |
| **Get properties** | `GetPropertiesAsync()` |
| **Delete queue** | `DeleteAsync()` |

### Remember

1. **ReceiveMessages** makes messages invisible (default 30 seconds)
2. **PeekMessages** doesn't affect visibility
3. **DeleteMessage** requires MessageId and PopReceipt
4. **UpdateMessage** can change content and extend visibility
5. **Max messages** per receive is 32
6. **DequeueCount** tracks delivery attempts

### Common Patterns

```csharp
// Send
await queueClient.SendMessageAsync("data");

// Receive, process, delete
var messages = await queueClient.ReceiveMessagesAsync(1);
var message = messages.Value[0];
await ProcessAsync(message.Body.ToString());
await queueClient.DeleteMessageAsync(message.MessageId, message.PopReceipt);

// Extend timeout
await queueClient.UpdateMessageAsync(
    message.MessageId,
    message.PopReceipt,
    visibilityTimeout: TimeSpan.FromMinutes(5));

// Poison queue
if (message.DequeueCount >= 5)
{
    await poisonQueueClient.SendMessageAsync(message.Body.ToString());
    await queueClient.DeleteMessageAsync(message.MessageId, message.PopReceipt);
}
```

---

## Summary

**Azure.Storage.Queues library provides:**
- ✅ QueueClient for queue operations
- ✅ Send messages with optional TTL and visibility delay
- ✅ Receive messages with configurable visibility timeout
- ✅ Peek messages without affecting visibility
- ✅ Update message content and extend timeout
- ✅ Delete messages after processing
- ✅ Get queue properties and message count

**Key pattern:** Receive → Process → Delete (or move to poison queue after retries)!