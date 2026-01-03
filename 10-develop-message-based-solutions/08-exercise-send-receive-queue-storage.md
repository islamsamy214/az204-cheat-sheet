# Exercise: Send and Receive Messages from Queue Storage

## Overview

In this hands-on exercise, you'll create an Azure Storage account with Queue Storage, send messages to a queue, and receive messages using the Azure SDK for .NET. You'll also implement a poison message queue and monitor queue metrics.

**What you'll learn:**
- Create storage account and queue
- Send messages to queue
- Receive and process messages
- Handle failed messages (poison queue)
- Update message visibility timeout
- Monitor queue depth
- Clean up resources

**Estimated time:** 20-30 minutes

---

## Prerequisites

- Azure subscription
- Azure CLI installed
- .NET 6.0 or later installed
- Code editor (VS Code, Visual Studio, or similar)

---

## Architecture

```
Producer App              Azure Queue Storage           Consumer App
┌──────────┐             ┌────────────────┐            ┌──────────┐
│          │──Send────────│  Main Queue    │─Receive───│          │
│ Sender   │             │  ├── Msg 1     │            │ Receiver │
│          │             │  ├── Msg 2     │            │          │
│          │             │  └── Msg N     │            │          │
└──────────┘             └────────────────┘            └──────────┘
                                │                            │
                                │                            │
                         Failed messages              After 5 retries
                                │                            │
                                v                            v
                         ┌────────────────┐           ┌──────────┐
                         │ Poison Queue   │<──────────│ Failed   │
                         │ ├── Failed 1   │           │ Messages │
                         │ └── Failed N   │           └──────────┘
                         └────────────────┘
```

---

## Part 1: Create Azure Resources

### Step 1: Create Resource Group

```bash
# Set variables
RESOURCE_GROUP="rg-queuestorage-demo"
LOCATION="eastus"
STORAGE_ACCOUNT="stqueue$RANDOM"  # Must be globally unique, lowercase only
QUEUE_NAME="orders"
POISON_QUEUE_NAME="orders-poison"

# Create resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION

echo "Resource group created: $RESOURCE_GROUP"
```

### Step 2: Create Storage Account

```bash
# Create storage account
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --min-tls-version TLS1_2

echo "Storage account created: $STORAGE_ACCOUNT"
echo "URL: https://$STORAGE_ACCOUNT.queue.core.windows.net"
```

**SKU options:**
- `Standard_LRS`: Locally redundant storage
- `Standard_GRS`: Geo-redundant storage
- `Standard_ZRS`: Zone-redundant storage

### Step 3: Get Connection String

```bash
# Get connection string
CONNECTION_STRING=$(az storage account show-connection-string \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query connectionString \
  --output tsv)

echo "Connection string: $CONNECTION_STRING"
```

**Save this connection string** - you'll need it in your code.

### Step 4: Create Queues

```bash
# Create main queue
az storage queue create \
  --name $QUEUE_NAME \
  --connection-string $CONNECTION_STRING

echo "Queue created: $QUEUE_NAME"

# Create poison message queue
az storage queue create \
  --name $POISON_QUEUE_NAME \
  --connection-string $CONNECTION_STRING

echo "Poison queue created: $POISON_QUEUE_NAME"
```

---

## Part 2: Send Messages to Queue

### Step 1: Create .NET Console Application

```bash
# Create new console app
dotnet new console -n QueueStorageSender
cd QueueStorageSender

# Add Azure Storage Queues package
dotnet add package Azure.Storage.Queues
```

### Step 2: Sender Code

Create `Program.cs`:

```csharp
using Azure.Storage.Queues;
using System;
using System.Threading.Tasks;

class Program
{
    // Replace with your connection string
    static string connectionString = "<YOUR_CONNECTION_STRING>";
    static string queueName = "orders";

    static async Task Main(string[] args)
    {
        // Create queue client
        QueueClient queueClient = new QueueClient(connectionString, queueName);

        // Ensure queue exists
        await queueClient.CreateIfNotExistsAsync();

        Console.WriteLine("=== Queue Storage Sender ===\n");

        // Send single message
        await SendSingleMessageAsync(queueClient);

        // Send multiple messages
        await SendMultipleMessagesAsync(queueClient);

        // Send message with custom TTL
        await SendMessageWithTTLAsync(queueClient);

        Console.WriteLine("\nAll messages sent successfully!");
        Console.WriteLine($"Queue URL: {queueClient.Uri}");
    }

    static async Task SendSingleMessageAsync(QueueClient queueClient)
    {
        string message = "Order #1001 - Customer: John Doe - Amount: $299.99";
        
        await queueClient.SendMessageAsync(message);
        Console.WriteLine($"✅ Sent: {message}");
    }

    static async Task SendMultipleMessagesAsync(QueueClient queueClient)
    {
        Console.WriteLine("\nSending batch of messages...");
        
        for (int i = 1; i <= 5; i++)
        {
            string message = $"Order #{1000 + i} - Amount: ${i * 100}.00";
            await queueClient.SendMessageAsync(message);
            Console.WriteLine($"✅ Sent: {message}");
            
            await Task.Delay(500);  // Wait 0.5 seconds
        }
    }

    static async Task SendMessageWithTTLAsync(QueueClient queueClient)
    {
        Console.WriteLine("\nSending message with 1-hour TTL...");
        
        string message = "Time-sensitive order #1099";
        
        await queueClient.SendMessageAsync(
            messageText: message,
            visibilityTimeout: TimeSpan.Zero,  // Visible immediately
            timeToLive: TimeSpan.FromHours(1));  // Expires in 1 hour
        
        Console.WriteLine($"✅ Sent: {message} (expires in 1 hour)");
    }
}
```

### Step 3: Run Sender

```bash
# Update connection string in Program.cs
# Then run
dotnet run
```

**Expected output:**
```
=== Queue Storage Sender ===

✅ Sent: Order #1001 - Customer: John Doe - Amount: $299.99

Sending batch of messages...
✅ Sent: Order #1002 - Amount: $100.00
✅ Sent: Order #1003 - Amount: $200.00
✅ Sent: Order #1004 - Amount: $300.00
✅ Sent: Order #1005 - Amount: $400.00
✅ Sent: Order #1006 - Amount: $500.00

Sending message with 1-hour TTL...
✅ Sent: Time-sensitive order #1099 (expires in 1 hour)

All messages sent successfully!
Queue URL: https://stqueue12345.queue.core.windows.net/orders
```

---

## Part 3: Receive and Process Messages

### Step 1: Create Receiver Application

```bash
# Create new console app
cd ..
dotnet new console -n QueueStorageReceiver
cd QueueStorageReceiver

# Add Azure Storage Queues package
dotnet add package Azure.Storage.Queues
```

### Step 2: Receiver Code

Create `Program.cs`:

```csharp
using Azure.Storage.Queues;
using Azure.Storage.Queues.Models;
using System;
using System.Threading.Tasks;

class Program
{
    static string connectionString = "<YOUR_CONNECTION_STRING>";
    static string queueName = "orders";
    static string poisonQueueName = "orders-poison";
    
    static async Task Main(string[] args)
    {
        // Create queue clients
        QueueClient queueClient = new QueueClient(connectionString, queueName);
        QueueClient poisonQueueClient = new QueueClient(connectionString, poisonQueueName);
        
        await queueClient.CreateIfNotExistsAsync();
        await poisonQueueClient.CreateIfNotExistsAsync();

        Console.WriteLine("=== Queue Storage Receiver ===");
        Console.WriteLine("Press Ctrl+C to stop\n");

        int processedCount = 0;
        int failedCount = 0;
        int poisonCount = 0;

        while (true)
        {
            // Receive up to 10 messages
            QueueMessage[] messages = await queueClient.ReceiveMessagesAsync(
                maxMessages: 10,
                visibilityTimeout: TimeSpan.FromSeconds(30));

            if (messages.Length == 0)
            {
                Console.WriteLine($"[{DateTime.Now:HH:mm:ss}] No messages. Waiting...");
                Console.WriteLine($"Stats: Processed={processedCount}, Failed={failedCount}, Poison={poisonCount}\n");
                await Task.Delay(5000);  // Wait 5 seconds
                continue;
            }

            foreach (QueueMessage message in messages)
            {
                Console.WriteLine($"\n📨 Received message:");
                Console.WriteLine($"   MessageId: {message.MessageId}");
                Console.WriteLine($"   Body: {message.Body}");
                Console.WriteLine($"   DequeueCount: {message.DequeueCount}");
                Console.WriteLine($"   InsertedOn: {message.InsertedOn:yyyy-MM-dd HH:mm:ss}");
                Console.WriteLine($"   ExpiresOn: {message.ExpiresOn:yyyy-MM-dd HH:mm:ss}");
                Console.WriteLine($"   NextVisibleOn: {message.NextVisibleOn:yyyy-MM-dd HH:mm:ss}");

                try
                {
                    // Process message
                    await ProcessMessageAsync(message);
                    
                    // Delete message after successful processing
                    await queueClient.DeleteMessageAsync(
                        message.MessageId,
                        message.PopReceipt);
                    
                    processedCount++;
                    Console.WriteLine("   ✅ Processed and deleted");
                }
                catch (Exception ex)
                {
                    failedCount++;
                    Console.WriteLine($"   ❌ Processing failed: {ex.Message}");
                    
                    // Check if exceeded max retries (5 attempts)
                    if (message.DequeueCount >= 5)
                    {
                        // Move to poison message queue
                        await poisonQueueClient.SendMessageAsync(
                            $"[Failed after {message.DequeueCount} attempts] {message.Body}");
                        
                        await queueClient.DeleteMessageAsync(
                            message.MessageId,
                            message.PopReceipt);
                        
                        poisonCount++;
                        Console.WriteLine($"   ☠️  Moved to poison queue (attempt {message.DequeueCount})");
                    }
                    else
                    {
                        Console.WriteLine($"   ⚠️  Will retry (attempt {message.DequeueCount}/5)");
                        // Don't delete - message becomes visible again after timeout
                    }
                }
            }
        }
    }

    static async Task ProcessMessageAsync(QueueMessage message)
    {
        // Simulate processing time
        await Task.Delay(100);
        
        // Simulate random failures (20% failure rate for testing)
        Random random = new Random();
        if (random.Next(10) < 2)
        {
            throw new Exception("Simulated processing failure");
        }
        
        Console.WriteLine("   Processing order...");
        // Your business logic here
    }
}
```

### Step 3: Run Receiver

```bash
# Update connection string in Program.cs
# Then run
dotnet run
```

**Expected output:**
```
=== Queue Storage Receiver ===
Press Ctrl+C to stop

📨 Received message:
   MessageId: abc123...
   Body: Order #1001 - Customer: John Doe - Amount: $299.99
   DequeueCount: 1
   InsertedOn: 2024-01-15 10:30:00
   ExpiresOn: 2024-01-22 10:30:00
   NextVisibleOn: 2024-01-15 10:30:30
   Processing order...
   ✅ Processed and deleted

📨 Received message:
   MessageId: def456...
   Body: Order #1002 - Amount: $100.00
   DequeueCount: 1
   InsertedOn: 2024-01-15 10:30:01
   ExpiresOn: 2024-01-22 10:30:01
   NextVisibleOn: 2024-01-15 10:30:31
   ❌ Processing failed: Simulated processing failure
   ⚠️  Will retry (attempt 1/5)

[10:30:35] No messages. Waiting...
Stats: Processed=5, Failed=2, Poison=0
```

---

## Part 4: Extend Visibility Timeout

### Long-Running Processing Example

```csharp
using Azure.Storage.Queues;
using Azure.Storage.Queues.Models;
using System;
using System.Threading.Tasks;

class LongRunningProcessor
{
    static string connectionString = "<YOUR_CONNECTION_STRING>";
    static string queueName = "orders";

    static async Task Main(string[] args)
    {
        QueueClient queueClient = new QueueClient(connectionString, queueName);

        // Receive message with 30-second visibility
        var messages = await queueClient.ReceiveMessagesAsync(
            maxMessages: 1,
            visibilityTimeout: TimeSpan.FromSeconds(30));

        if (messages.Value.Length > 0)
        {
            QueueMessage message = messages.Value[0];
            Console.WriteLine($"Received: {message.Body}");
            Console.WriteLine($"Invisible until: {message.NextVisibleOn}");

            // Long processing starts
            Console.WriteLine("\nStarting long-running process...");
            await Task.Delay(TimeSpan.FromSeconds(25));  // 25 of 30 seconds used

            // Extend visibility timeout before it expires
            Console.WriteLine("\nExtending visibility timeout...");
            var updateResponse = await queueClient.UpdateMessageAsync(
                message.MessageId,
                message.PopReceipt,
                visibilityTimeout: TimeSpan.FromMinutes(5));  // Extend by 5 minutes

            Console.WriteLine($"New visibility timeout: {updateResponse.Value.NextVisibleOn}");
            string newPopReceipt = updateResponse.Value.PopReceipt;

            // Continue processing with extended time
            await Task.Delay(TimeSpan.FromMinutes(4));  // 4 more minutes
            Console.WriteLine("\nProcessing complete");

            // Delete with new pop receipt
            await queueClient.DeleteMessageAsync(message.MessageId, newPopReceipt);
            Console.WriteLine("Message deleted");
        }
    }
}
```

---

## Part 5: Process Poison Messages

### Poison Message Processor

```csharp
using Azure.Storage.Queues;
using Azure.Storage.Queues.Models;
using System;
using System.Threading.Tasks;

class PoisonMessageProcessor
{
    static string connectionString = "<YOUR_CONNECTION_STRING>";
    static string poisonQueueName = "orders-poison";

    static async Task Main(string[] args)
    {
        QueueClient poisonQueueClient = new QueueClient(connectionString, poisonQueueName);
        await poisonQueueClient.CreateIfNotExistsAsync();

        Console.WriteLine("=== Poison Message Processor ===\n");

        // Get properties to see message count
        var properties = await poisonQueueClient.GetPropertiesAsync();
        int messageCount = properties.Value.ApproximateMessagesCount;
        
        Console.WriteLine($"Poison queue has approximately {messageCount} messages\n");

        // Process all poison messages
        while (true)
        {
            var messages = await poisonQueueClient.ReceiveMessagesAsync(
                maxMessages: 10,
                visibilityTimeout: TimeSpan.FromMinutes(5));

            if (messages.Value.Length == 0)
            {
                Console.WriteLine("No more poison messages");
                break;
            }

            foreach (QueueMessage message in messages.Value)
            {
                Console.WriteLine($"☠️  Poison message:");
                Console.WriteLine($"   MessageId: {message.MessageId}");
                Console.WriteLine($"   Body: {message.Body}");
                Console.WriteLine($"   DequeueCount: {message.DequeueCount}");
                Console.WriteLine($"   InsertedOn: {message.InsertedOn}");

                // Options for handling poison messages:
                // 1. Log to monitoring system
                // 2. Send alert to operations team
                // 3. Store in database for manual review
                // 4. Attempt to fix and resubmit to main queue
                // 5. Delete after logging

                // For this demo, just log and delete
                await LogPoisonMessageAsync(message);
                
                await poisonQueueClient.DeleteMessageAsync(
                    message.MessageId,
                    message.PopReceipt);
                
                Console.WriteLine("   ✅ Logged and deleted\n");
            }
        }

        Console.WriteLine("Poison message processing complete");
    }

    static async Task LogPoisonMessageAsync(QueueMessage message)
    {
        // Log to file, database, or monitoring system
        Console.WriteLine("   📝 Logging to monitoring system...");
        await Task.Delay(100);  // Simulate logging
    }
}
```

---

## Part 6: Monitor Queue Metrics

### Check Queue Depth

```bash
# Get message count for main queue
az storage queue metadata show \
  --name $QUEUE_NAME \
  --connection-string $CONNECTION_STRING \
  --query approximateMessageCount \
  --output tsv

# Get message count for poison queue
az storage queue metadata show \
  --name $POISON_QUEUE_NAME \
  --connection-string $CONNECTION_STRING \
  --query approximateMessageCount \
  --output tsv
```

### Monitor with Code

```csharp
using Azure.Storage.Queues;
using System;
using System.Threading.Tasks;

class QueueMonitor
{
    static async Task Main(string[] args)
    {
        string connectionString = "<YOUR_CONNECTION_STRING>";
        
        QueueClient queueClient = new QueueClient(connectionString, "orders");
        QueueClient poisonQueueClient = new QueueClient(connectionString, "orders-poison");

        Console.WriteLine("=== Queue Monitor ===\n");

        while (true)
        {
            // Get main queue properties
            var properties = await queueClient.GetPropertiesAsync();
            int mainQueueCount = properties.Value.ApproximateMessagesCount;

            // Get poison queue properties
            var poisonProperties = await poisonQueueClient.GetPropertiesAsync();
            int poisonQueueCount = poisonProperties.Value.ApproximateMessagesCount;

            // Display metrics
            Console.Clear();
            Console.WriteLine("=== Queue Monitor ===");
            Console.WriteLine($"Time: {DateTime.Now:yyyy-MM-dd HH:mm:ss}\n");
            Console.WriteLine($"Main Queue ({queueClient.Name}):");
            Console.WriteLine($"  Messages: {mainQueueCount}");
            Console.WriteLine($"  URL: {queueClient.Uri}\n");
            Console.WriteLine($"Poison Queue ({poisonQueueClient.Name}):");
            Console.WriteLine($"  Messages: {poisonQueueCount}");
            Console.WriteLine($"  URL: {poisonQueueClient.Uri}\n");

            // Alert if backlog too large
            if (mainQueueCount > 100)
            {
                Console.WriteLine("⚠️  WARNING: Main queue backlog > 100 messages");
                Console.WriteLine("   Consider scaling out consumers\n");
            }

            if (poisonQueueCount > 10)
            {
                Console.WriteLine("⚠️  WARNING: Poison queue has > 10 messages");
                Console.WriteLine("   Review failed messages\n");
            }

            Console.WriteLine("Press Ctrl+C to stop");

            await Task.Delay(5000);  // Update every 5 seconds
        }
    }
}
```

---

## Part 7: Cleanup

### Delete Queues

```bash
# Delete main queue
az storage queue delete \
  --name $QUEUE_NAME \
  --connection-string $CONNECTION_STRING

# Delete poison queue
az storage queue delete \
  --name $POISON_QUEUE_NAME \
  --connection-string $CONNECTION_STRING

echo "Queues deleted"
```

### Delete Resource Group

```bash
# Delete resource group (removes storage account and all queues)
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait

echo "Resource group deletion initiated"
```

---

## Troubleshooting

### Issue: "Storage account name not available"

**Solution:** Storage account names must be globally unique
```bash
# Use random suffix
STORAGE_ACCOUNT="stqueue$RANDOM"
```

### Issue: Messages not appearing

**Check:**
1. Connection string is correct
2. Queue name matches exactly (case-sensitive)
3. Messages haven't expired (check TTL)

```bash
# Verify queue exists
az storage queue exists \
  --name $QUEUE_NAME \
  --connection-string $CONNECTION_STRING
```

### Issue: "Pop receipt mismatch" error

**Cause:** Message was updated, changing pop receipt

**Solution:** Use the new pop receipt from UpdateMessage response
```csharp
var updateResponse = await queueClient.UpdateMessageAsync(...);
string newPopReceipt = updateResponse.Value.PopReceipt;
await queueClient.DeleteMessageAsync(messageId, newPopReceipt);
```

### Issue: Messages being processed multiple times

**Cause:** Not deleting messages after processing

**Solution:** Always delete after successful processing
```csharp
await ProcessMessageAsync(message);
await queueClient.DeleteMessageAsync(message.MessageId, message.PopReceipt);
```

---

## Key Takeaways

✅ **Created Azure Queue Storage**
- Storage account with queues
- Main queue and poison queue

✅ **Sent messages to queue**
- Single and batch messages
- Messages with custom TTL
- Got message receipts

✅ **Received and processed messages**
- Two-step dequeue pattern (receive, delete)
- Visibility timeout prevents double processing
- Automatic retry on failure

✅ **Handled poison messages**
- Tracked delivery count
- Moved to poison queue after 5 retries
- Processed and logged poison messages

✅ **Extended visibility timeout**
- Updated message during long processing
- Prevented timeout expiration

✅ **Monitored queue metrics**
- Checked message count
- Alerted on backlog

---

## Exam Tips

1. **Receive makes message invisible** for 30 seconds (default)
2. **Delete requires MessageId and PopReceipt**
3. **DequeueCount tracks delivery attempts** - use for poison queue logic
4. **Update can extend visibility timeout** during processing
5. **Max 32 messages per receive** operation
6. **TTL default is 7 days** but can be unlimited
7. **Always delete after successful processing** to prevent redelivery

**Remember:** The two-step pattern (receive → process → delete) ensures at-least-once delivery with automatic retry on failure!