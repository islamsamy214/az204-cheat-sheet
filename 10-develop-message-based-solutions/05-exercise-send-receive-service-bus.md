# Exercise: Send and Receive Messages from Service Bus Queue

## Overview

In this exercise, you'll create a Service Bus namespace, send messages to a queue, and receive messages using the Azure SDK for .NET. You'll also explore advanced features like message sessions and dead-letter queues.

**What you'll learn:**
- Create Service Bus namespace and queue
- Send messages to a queue
- Receive messages using Peek Lock mode
- Use message sessions for FIFO ordering
- Handle dead-letter queue
- Monitor queue metrics

**Estimated time:** 30-40 minutes

---

## Prerequisites

- Azure subscription
- Azure CLI installed
- .NET 6.0 or later installed
- Code editor (VS Code, Visual Studio, or similar)

---

## Architecture

```
Your Application                Service Bus                    Your Application
┌───────────────┐              ┌───────────┐                 ┌───────────────┐
│               │              │ Namespace │                 │               │
│   Producer    │ ──Send──────>│           │                 │   Consumer    │
│  (Sender)     │              │   Queue   │──────Receive───>│  (Receiver)   │
│               │              │           │                 │               │
└───────────────┘              └───────────┘                 └───────────────┘
                                     │
                                     v
                              ┌──────────────┐
                              │ Dead-Letter  │
                              │    Queue     │
                              └──────────────┘
```

---

## Part 1: Create Service Bus Resources

### Step 1: Create Resource Group

```bash
# Set variables
RESOURCE_GROUP="rg-servicebus-demo"
LOCATION="eastus"
NAMESPACE_NAME="sb-namespace-$RANDOM"
QUEUE_NAME="demoqueue"

# Create resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

### Step 2: Create Service Bus Namespace

```bash
# Create Service Bus namespace (Standard tier)
az servicebus namespace create \
  --resource-group $RESOURCE_GROUP \
  --name $NAMESPACE_NAME \
  --location $LOCATION \
  --sku Standard

# Wait for namespace to be created
echo "Namespace created: $NAMESPACE_NAME"
```

**Tier options:**
- `Basic`: Queues only, 256 KB messages
- `Standard`: Queues + topics, 256 KB messages, transactions
- `Premium`: Dedicated resources, 100 MB messages, geo-DR

### Step 3: Create Queue

```bash
# Create queue
az servicebus queue create \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $NAMESPACE_NAME \
  --name $QUEUE_NAME \
  --max-size 1024 \
  --default-message-time-to-live P14D \
  --lock-duration PT30S \
  --max-delivery-count 10 \
  --enable-duplicate-detection true \
  --duplicate-detection-history-time-window PT10M

echo "Queue created: $QUEUE_NAME"
```

**Queue properties explained:**
- `max-size`: Maximum queue size (1024 MB)
- `default-message-time-to-live`: Message TTL (14 days)
- `lock-duration`: Lock timeout for Peek Lock (30 seconds)
- `max-delivery-count`: Max retries before dead-lettering (10)
- `enable-duplicate-detection`: Prevent duplicate messages
- `duplicate-detection-history-time-window`: Deduplication window (10 minutes)

### Step 4: Get Connection String

```bash
# Get connection string
CONNECTION_STRING=$(az servicebus namespace authorization-rule keys list \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $NAMESPACE_NAME \
  --name RootManageSharedAccessKey \
  --query primaryConnectionString \
  --output tsv)

echo "Connection string: $CONNECTION_STRING"
```

**Save this connection string** - you'll need it in your code.

---

## Part 2: Send Messages to Queue

### Step 1: Create .NET Console Application

```bash
# Create new console app
dotnet new console -n ServiceBusSender
cd ServiceBusSender

# Add Service Bus package
dotnet add package Azure.Messaging.ServiceBus
```

### Step 2: Sender Code

Create `Program.cs`:

```csharp
using Azure.Messaging.ServiceBus;
using System;
using System.Threading.Tasks;

class Program
{
    // Replace with your connection string and queue name
    static string connectionString = "<YOUR_CONNECTION_STRING>";
    static string queueName = "demoqueue";

    static async Task Main(string[] args)
    {
        // Create Service Bus client
        await using ServiceBusClient client = new ServiceBusClient(connectionString);
        ServiceBusSender sender = client.CreateSender(queueName);

        try
        {
            // Send single message
            await SendSingleMessageAsync(sender);

            // Send batch of messages
            await SendBatchMessagesAsync(sender);

            // Send message with properties
            await SendMessageWithPropertiesAsync(sender);

            Console.WriteLine("All messages sent successfully!");
        }
        finally
        {
            await sender.DisposeAsync();
            await client.DisposeAsync();
        }
    }

    static async Task SendSingleMessageAsync(ServiceBusSender sender)
    {
        var message = new ServiceBusMessage("Hello from Service Bus!");
        message.MessageId = Guid.NewGuid().ToString();
        
        await sender.SendMessageAsync(message);
        Console.WriteLine($"Sent single message: {message.MessageId}");
    }

    static async Task SendBatchMessagesAsync(ServiceBusSender sender)
    {
        // Create batch
        using ServiceBusMessageBatch messageBatch = 
            await sender.CreateMessageBatchAsync();

        // Add messages to batch
        for (int i = 1; i <= 5; i++)
        {
            var message = new ServiceBusMessage($"Message {i} in batch");
            message.MessageId = $"batch-msg-{i}";

            if (!messageBatch.TryAddMessage(message))
            {
                throw new Exception($"Message {i} is too large for the batch");
            }
        }

        // Send batch
        await sender.SendMessagesAsync(messageBatch);
        Console.WriteLine($"Sent batch of {messageBatch.Count} messages");
    }

    static async Task SendMessageWithPropertiesAsync(ServiceBusSender sender)
    {
        var message = new ServiceBusMessage("Order #12345");
        message.MessageId = "order-12345";
        message.CorrelationId = "correlation-001";
        message.ContentType = "application/json";
        message.Subject = "OrderCreated";
        message.TimeToLive = TimeSpan.FromMinutes(10);

        // Add custom properties
        message.ApplicationProperties["Priority"] = "High";
        message.ApplicationProperties["Region"] = "US-West";
        message.ApplicationProperties["OrderAmount"] = 599.99;

        await sender.SendMessageAsync(message);
        Console.WriteLine($"Sent message with properties: {message.MessageId}");
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
Sent single message: <guid>
Sent batch of 5 messages
Sent message with properties: order-12345
All messages sent successfully!
```

---

## Part 3: Receive Messages from Queue

### Step 1: Create Receiver Application

```bash
# Create new console app
cd ..
dotnet new console -n ServiceBusReceiver
cd ServiceBusReceiver

# Add Service Bus package
dotnet add package Azure.Messaging.ServiceBus
```

### Step 2: Receiver Code

Create `Program.cs`:

```csharp
using Azure.Messaging.ServiceBus;
using System;
using System.Threading.Tasks;

class Program
{
    static string connectionString = "<YOUR_CONNECTION_STRING>";
    static string queueName = "demoqueue";

    static async Task Main(string[] args)
    {
        await using ServiceBusClient client = new ServiceBusClient(connectionString);
        
        // Peek Lock mode (default, recommended)
        ServiceBusReceiver receiver = client.CreateReceiver(queueName);

        try
        {
            Console.WriteLine("Receiving messages...");
            Console.WriteLine("Press any key to stop receiving");

            // Receive messages until key pressed
            await ReceiveMessagesAsync(receiver);
        }
        finally
        {
            await receiver.DisposeAsync();
            await client.DisposeAsync();
        }
    }

    static async Task ReceiveMessagesAsync(ServiceBusReceiver receiver)
    {
        while (!Console.KeyAvailable)
        {
            // Receive up to 10 messages
            var messages = await receiver.ReceiveMessagesAsync(
                maxMessages: 10,
                maxWaitTime: TimeSpan.FromSeconds(5));

            foreach (var message in messages)
            {
                try
                {
                    Console.WriteLine($"\nReceived message:");
                    Console.WriteLine($"  MessageId: {message.MessageId}");
                    Console.WriteLine($"  Body: {message.Body}");
                    Console.WriteLine($"  DeliveryCount: {message.DeliveryCount}");
                    Console.WriteLine($"  EnqueuedTime: {message.EnqueuedTime}");

                    // Display custom properties
                    if (message.ApplicationProperties.Count > 0)
                    {
                        Console.WriteLine("  Application Properties:");
                        foreach (var prop in message.ApplicationProperties)
                        {
                            Console.WriteLine($"    {prop.Key}: {prop.Value}");
                        }
                    }

                    // Simulate processing
                    await ProcessMessageAsync(message);

                    // Complete the message (remove from queue)
                    await receiver.CompleteMessageAsync(message);
                    Console.WriteLine("  ✅ Message completed");
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"  ❌ Error processing message: {ex.Message}");
                    
                    // Abandon the message (requeue for retry)
                    await receiver.AbandonMessageAsync(message);
                    Console.WriteLine("  ⚠️  Message abandoned (will be redelivered)");
                }
            }

            if (messages.Count == 0)
            {
                Console.WriteLine("No messages available. Waiting...");
            }
        }
    }

    static async Task ProcessMessageAsync(ServiceBusReceivedMessage message)
    {
        // Simulate processing time
        await Task.Delay(100);
        
        // Your business logic here
        Console.WriteLine("  Processing message...");
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
Receiving messages...
Press any key to stop receiving

Received message:
  MessageId: <guid>
  Body: Hello from Service Bus!
  DeliveryCount: 1
  EnqueuedTime: 2024-01-15T10:30:00Z
  Processing message...
  ✅ Message completed

Received message:
  MessageId: order-12345
  Body: Order #12345
  DeliveryCount: 1
  EnqueuedTime: 2024-01-15T10:30:01Z
  Application Properties:
    Priority: High
    Region: US-West
    OrderAmount: 599.99
  Processing message...
  ✅ Message completed

No messages available. Waiting...
```

---

## Part 4: Message Sessions (FIFO Ordering)

### Step 1: Create Session-Enabled Queue

```bash
# Create queue with sessions enabled
az servicebus queue create \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $NAMESPACE_NAME \
  --name sessionqueue \
  --enable-session true

echo "Session queue created: sessionqueue"
```

### Step 2: Send Messages with SessionId

```csharp
using Azure.Messaging.ServiceBus;
using System;
using System.Threading.Tasks;

class SessionSender
{
    static string connectionString = "<YOUR_CONNECTION_STRING>";
    static string queueName = "sessionqueue";

    static async Task Main(string[] args)
    {
        await using var client = new ServiceBusClient(connectionString);
        var sender = client.CreateSender(queueName);

        try
        {
            // Send messages for Customer A (session)
            await SendSessionMessagesAsync(sender, "customer-A", 5);

            // Send messages for Customer B (session)
            await SendSessionMessagesAsync(sender, "customer-B", 3);

            // Send messages for Customer C (session)
            await SendSessionMessagesAsync(sender, "customer-C", 4);

            Console.WriteLine("Session messages sent!");
        }
        finally
        {
            await sender.DisposeAsync();
        }
    }

    static async Task SendSessionMessagesAsync(
        ServiceBusSender sender, 
        string sessionId, 
        int count)
    {
        for (int i = 1; i <= count; i++)
        {
            var message = new ServiceBusMessage($"Order {i} for {sessionId}");
            message.SessionId = sessionId;  // FIFO guarantee per session
            message.MessageId = $"{sessionId}-order-{i}";

            await sender.SendMessageAsync(message);
            Console.WriteLine($"Sent: {message.Body} (Session: {sessionId})");
        }
    }
}
```

### Step 3: Receive Messages from Session

```csharp
using Azure.Messaging.ServiceBus;
using System;
using System.Threading.Tasks;

class SessionReceiver
{
    static string connectionString = "<YOUR_CONNECTION_STRING>";
    static string queueName = "sessionqueue";

    static async Task Main(string[] args)
    {
        await using var client = new ServiceBusClient(connectionString);

        // Accept next available session
        var sessionReceiver = await client.AcceptNextSessionAsync(queueName);

        try
        {
            Console.WriteLine($"Processing session: {sessionReceiver.SessionId}");

            // Receive messages from session (FIFO order)
            await foreach (var message in sessionReceiver.ReceiveMessagesAsync())
            {
                Console.WriteLine($"  Received: {message.Body}");
                Console.WriteLine($"  SequenceNumber: {message.SequenceNumber}");
                
                await sessionReceiver.CompleteMessageAsync(message);
            }

            Console.WriteLine($"Session {sessionReceiver.SessionId} complete!");
        }
        finally
        {
            await sessionReceiver.DisposeAsync();
        }
    }
}
```

**Process multiple sessions in parallel:**

```csharp
// Process 3 sessions concurrently
var tasks = Enumerable.Range(0, 3).Select(async i =>
{
    var sessionReceiver = await client.AcceptNextSessionAsync(queueName);
    
    Console.WriteLine($"Worker {i} processing session: {sessionReceiver.SessionId}");
    
    await foreach (var message in sessionReceiver.ReceiveMessagesAsync())
    {
        Console.WriteLine($"  [{sessionReceiver.SessionId}] {message.Body}");
        await sessionReceiver.CompleteMessageAsync(message);
    }
    
    await sessionReceiver.DisposeAsync();
});

await Task.WhenAll(tasks);
```

---

## Part 5: Dead-Letter Queue (DLQ)

### Step 1: Simulate Failed Processing

```csharp
using Azure.Messaging.ServiceBus;
using System;
using System.Threading.Tasks;

class DLQDemo
{
    static string connectionString = "<YOUR_CONNECTION_STRING>";
    static string queueName = "demoqueue";

    static async Task Main(string[] args)
    {
        await using var client = new ServiceBusClient(connectionString);
        var receiver = client.CreateReceiver(queueName);

        try
        {
            var message = await receiver.ReceiveMessageAsync();

            if (message != null)
            {
                Console.WriteLine($"Received: {message.Body}");
                Console.WriteLine($"DeliveryCount: {message.DeliveryCount}");

                try
                {
                    // Simulate processing failure
                    throw new Exception("Simulated processing error");
                }
                catch (Exception ex)
                {
                    if (message.DeliveryCount >= 3)
                    {
                        // Give up after 3 retries, move to DLQ
                        await receiver.DeadLetterMessageAsync(
                            message,
                            deadLetterReason: "MaxRetriesExceeded",
                            deadLetterErrorDescription: ex.Message);
                        
                        Console.WriteLine("❌ Message moved to dead-letter queue");
                    }
                    else
                    {
                        // Retry by abandoning
                        await receiver.AbandonMessageAsync(message);
                        Console.WriteLine($"⚠️  Retry {message.DeliveryCount}/3");
                    }
                }
            }
        }
        finally
        {
            await receiver.DisposeAsync();
        }
    }
}
```

### Step 2: Process Dead-Letter Queue

```csharp
using Azure.Messaging.ServiceBus;
using System;
using System.Threading.Tasks;

class DLQProcessor
{
    static string connectionString = "<YOUR_CONNECTION_STRING>";
    static string queueName = "demoqueue";

    static async Task Main(string[] args)
    {
        await using var client = new ServiceBusClient(connectionString);
        
        // Create receiver for dead-letter queue
        var dlqReceiver = client.CreateReceiver(
            queueName,
            new ServiceBusReceiverOptions 
            { 
                SubQueue = SubQueue.DeadLetter 
            });

        try
        {
            Console.WriteLine("Processing dead-letter queue...\n");

            var messages = await dlqReceiver.ReceiveMessagesAsync(
                maxMessages: 10,
                maxWaitTime: TimeSpan.FromSeconds(5));

            foreach (var message in messages)
            {
                Console.WriteLine($"Dead-letter message:");
                Console.WriteLine($"  MessageId: {message.MessageId}");
                Console.WriteLine($"  Body: {message.Body}");
                Console.WriteLine($"  DeadLetterReason: {message.DeadLetterReason}");
                Console.WriteLine($"  DeadLetterErrorDescription: {message.DeadLetterErrorDescription}");
                Console.WriteLine($"  DeliveryCount: {message.DeliveryCount}");
                Console.WriteLine($"  EnqueuedTime: {message.EnqueuedTime}");

                // Option 1: Fix and resubmit to main queue
                // Option 2: Log and delete
                // Option 3: Move to error storage for later analysis

                await dlqReceiver.CompleteMessageAsync(message);
                Console.WriteLine("  ✅ DLQ message processed\n");
            }

            if (messages.Count == 0)
            {
                Console.WriteLine("No messages in dead-letter queue");
            }
        }
        finally
        {
            await dlqReceiver.DisposeAsync();
        }
    }
}
```

---

## Part 6: Monitor Queue Metrics

### View Queue Metrics in Portal

1. Navigate to Azure Portal
2. Go to your Service Bus namespace
3. Select "Queues" → Select your queue
4. View "Overview" tab for metrics:
   - Active message count
   - Dead-letter message count
   - Size (MB)
   - Incoming/outgoing messages

### Query Queue Metrics with Azure CLI

```bash
# Get active message count
az servicebus queue show \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $NAMESPACE_NAME \
  --name $QUEUE_NAME \
  --query "countDetails.activeMessageCount" \
  --output tsv

# Get dead-letter message count
az servicebus queue show \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $NAMESPACE_NAME \
  --name $QUEUE_NAME \
  --query "countDetails.deadLetterMessageCount" \
  --output tsv

# Get all queue properties
az servicebus queue show \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $NAMESPACE_NAME \
  --name $QUEUE_NAME
```

### Monitor with Azure Monitor

```bash
# Enable diagnostic settings
az monitor diagnostic-settings create \
  --resource /subscriptions/<subscription-id>/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.ServiceBus/namespaces/$NAMESPACE_NAME \
  --name "servicebus-diagnostics" \
  --logs '[{"category": "OperationalLogs", "enabled": true}]' \
  --metrics '[{"category": "AllMetrics", "enabled": true}]' \
  --workspace /subscriptions/<subscription-id>/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.OperationalInsights/workspaces/myWorkspace
```

---

## Part 7: Cleanup

```bash
# Delete resource group (removes all resources)
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait

echo "Cleanup initiated"
```

---

## Troubleshooting

### Issue: "Unauthorized access" error

**Solution:**
```bash
# Regenerate connection string
az servicebus namespace authorization-rule keys renew \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $NAMESPACE_NAME \
  --name RootManageSharedAccessKey \
  --key PrimaryKey
```

### Issue: Messages not received

**Check:**
1. Connection string is correct
2. Queue name matches exactly
3. Messages haven't expired (TTL)
4. Queue is not full (check max size)

```bash
# Verify queue exists and has messages
az servicebus queue show \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $NAMESPACE_NAME \
  --name $QUEUE_NAME \
  --query "countDetails"
```

### Issue: Lock timeout errors

**Solution:** Increase lock duration or process faster
```bash
az servicebus queue update \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $NAMESPACE_NAME \
  --name $QUEUE_NAME \
  --lock-duration PT5M  # 5 minutes
```

### Issue: Too many dead-letter messages

**Solution:** Check DLQ and increase max-delivery-count
```bash
az servicebus queue update \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $NAMESPACE_NAME \
  --name $QUEUE_NAME \
  --max-delivery-count 20
```

---

## Key Takeaways

✅ **Created Service Bus namespace and queue**
- Used Azure CLI to provision resources
- Configured queue properties (lock duration, TTL, max delivery count)

✅ **Sent messages to queue**
- Single messages
- Batch messages
- Messages with custom properties

✅ **Received messages with Peek Lock**
- Complete messages after successful processing
- Abandon messages for retry
- Dead-letter messages after max retries

✅ **Used message sessions for FIFO ordering**
- Grouped related messages by SessionId
- Processed sessions in parallel

✅ **Handled dead-letter queue**
- Moved failed messages to DLQ
- Processed and inspected DLQ messages

✅ **Monitored queue metrics**
- Viewed metrics in Azure Portal
- Queried metrics with Azure CLI

---

## Exam Tips

1. **Peek Lock is default and recommended** for reliable processing
2. **Sessions enable FIFO** ordering within session
3. **Dead-letter queue** holds undeliverable messages
4. **Max delivery count** determines when to dead-letter
5. **Lock duration** should match processing time
6. **Use MessageId** for duplicate detection and idempotency
7. **Batch sending** improves performance

**Remember:** Always complete, abandon, or dead-letter messages in Peek Lock mode to prevent lock timeouts!