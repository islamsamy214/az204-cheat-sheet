# Event Delivery Durability in Azure Event Grid

## Overview

Azure Event Grid provides **durable event delivery** with built-in retry mechanisms, dead-lettering, and configurable delivery options to ensure events reach their destinations reliably.

**Key Features:**
- **At-least-once delivery**: Events delivered at least once (may be duplicated)
- **Retry with exponential backoff**: Automatic retries for failed deliveries
- **Dead-lettering**: Store undeliverable events for later analysis
- **Output batching**: Deliver multiple events in single request
- **Delayed delivery**: Automatically pause delivery to unhealthy endpoints
- **Custom delivery properties**: Add custom headers to event deliveries

---

## Retry Mechanism

Event Grid automatically retries event delivery when the endpoint doesn't respond successfully or returns specific error codes.

### Retry Schedule

Event Grid uses **exponential backoff** for retries:

```
Attempt    Wait Time       Cumulative Time
1          0 seconds       0 seconds
2          30 seconds      30 seconds
3          1 minute        1.5 minutes
4          2 minutes       3.5 minutes
5          4 minutes       7.5 minutes
6          8 minutes       15.5 minutes
7          16 minutes      31.5 minutes
8          32 minutes      63.5 minutes
...        ...             ...
30         ~13 hours       ~24 hours (default TTL)
```

**Retry Behavior:**
1. **First attempt**: Immediate delivery
2. **Second attempt**: After 30 seconds
3. **Subsequent attempts**: Exponential backoff (doubles each time)
4. **Randomization**: Small random delay added to prevent thundering herd
5. **Maximum attempts**: Configurable (1-30, default 30)
6. **Time-to-live (TTL)**: Configurable (1-1440 minutes, default 1440 = 24 hours)

### Retry Flow Diagram

```
┌─────────────┐
│   Publish   │
│    Event    │
└──────┬──────┘
       │
       ▼
┌─────────────────────┐
│   Event Grid        │
│   (Queue Event)     │
└──────┬──────────────┘
       │
       ▼
┌─────────────────────┐
│ Deliver to Endpoint │
└──────┬──────────────┘
       │
       ├─────────────────────────┐
       │                         │
       ▼                         ▼
  ┌─────────┐            ┌────────────┐
  │ Success │            │   Failure  │
  │ (2xx)   │            │  (see list)│
  └────┬────┘            └─────┬──────┘
       │                       │
       ▼                       ▼
  ┌─────────┐          ┌──────────────┐
  │  Done   │          │ Retry Policy │
  └─────────┘          │  Evaluation  │
                       └─────┬────────┘
                             │
                ┌────────────┴────────────┐
                │                         │
                ▼                         ▼
    ┌───────────────────┐      ┌──────────────────┐
    │ Within TTL &      │      │ Exceeded TTL or  │
    │ Max Attempts?     │      │ Max Attempts?    │
    └────┬──────────────┘      └────────┬─────────┘
         │                              │
         ▼                              ▼
┌─────────────────┐           ┌──────────────────┐
│ Wait (backoff)  │           │  Dead Letter     │
│ Then Retry      │           │  (if configured) │
└─────────────────┘           │  or Drop Event   │
                              └──────────────────┘
```

---

## Retry Policy Configuration

### Default Retry Policy

```json
{
  "retryPolicy": {
    "maxDeliveryAttempts": 30,
    "eventTimeToLiveInMinutes": 1440
  }
}
```

### Retry Policy Properties

| Property | Description | Min | Max | Default |
|----------|-------------|-----|-----|---------|
| `maxDeliveryAttempts` | Maximum number of delivery attempts | 1 | 30 | 30 |
| `eventTimeToLiveInMinutes` | Time before event expires | 1 | 1440 (24 hours) | 1440 |

### Configure Retry Policy (Azure CLI)

```bash
# Create subscription with custom retry policy
az eventgrid event-subscription create \
  --name mySubscription \
  --source-resource-id "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage" \
  --endpoint https://myfunction.azurewebsites.net/api/handler \
  --max-delivery-attempts 10 \
  --event-ttl 60

# Update existing subscription retry policy
az eventgrid event-subscription update \
  --name mySubscription \
  --source-resource-id "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage" \
  --max-delivery-attempts 5 \
  --event-ttl 30
```

### Configure Retry Policy (Azure Portal)

1. Navigate to **Event Grid Topic** or **System Topic**
2. Select **Event Subscriptions**
3. Create or edit subscription
4. In **Additional Features** tab:
   - Set **Max delivery attempts**: 1-30
   - Set **Event time to live**: 1-1440 minutes
5. Save configuration

### Configure Retry Policy (ARM Template)

```json
{
  "type": "Microsoft.EventGrid/eventSubscriptions",
  "apiVersion": "2022-06-15",
  "name": "mySubscription",
  "properties": {
    "destination": {
      "endpointType": "WebHook",
      "properties": {
        "endpointUrl": "https://myfunction.azurewebsites.net/api/handler"
      }
    },
    "retryPolicy": {
      "maxDeliveryAttempts": 10,
      "eventTimeToLiveInMinutes": 60
    }
  }
}
```

---

## Retryable vs. Non-Retryable Errors

### Retryable Errors (Event Grid will retry)

| Error Type | HTTP Code | Description | Retry? |
|------------|-----------|-------------|--------|
| Server Error | 500 | Internal server error | ✅ Yes |
| Service Unavailable | 503 | Service temporarily unavailable | ✅ Yes |
| Gateway Timeout | 504 | Gateway timeout | ✅ Yes |
| Request Timeout | 408 | Request timeout | ✅ Yes |
| Too Many Requests | 429 | Rate limiting | ✅ Yes |
| Network Errors | N/A | DNS resolution, connection failures | ✅ Yes |

### Non-Retryable Errors (Event Grid won't retry)

| Error Type | HTTP Code | Description | Retry? |
|------------|-----------|-------------|--------|
| Bad Request | 400 | Malformed request or validation error | ❌ No |
| Unauthorized | 401 | Authentication failed (webhook only) | ❌ No |
| Not Found | 404 | Endpoint not found | ❌ No |
| Payload Too Large | 413 | Request entity too large | ❌ No |
| URI Too Long | 414 | Request URI too long | ❌ No |
| Unsupported Media Type | 415 | Unsupported content type | ❌ No |

**Important Notes:**
- **401 Unauthorized**: Non-retryable for webhooks (assume configuration error)
- **401 for Azure services**: Retryable (temporary Azure AD issues)
- **Non-2xx responses**: Generally trigger retries unless explicitly non-retryable
- **Silent failures**: No response from endpoint triggers retry

### Handling Non-Retryable Errors

```csharp
[FunctionName("EventHandler")]
public static async Task<IActionResult> Run(
    [EventGridTrigger] EventGridEvent eventGridEvent,
    ILogger log)
{
    try
    {
        // Validate event
        if (string.IsNullOrEmpty(eventGridEvent.Subject))
        {
            log.LogError("Invalid event: missing subject");
            // Return 400 - don't retry invalid events
            return new BadRequestObjectResult("Event validation failed");
        }
        
        // Process event
        await ProcessEvent(eventGridEvent);
        
        // Return 200 - success
        return new OkResult();
    }
    catch (InvalidOperationException ex)
    {
        log.LogError($"Validation error: {ex.Message}");
        // Return 400 - don't retry validation errors
        return new BadRequestResult();
    }
    catch (Exception ex)
    {
        log.LogError($"Processing error: {ex.Message}");
        // Return 500 - retry transient errors
        return new StatusCodeResult(500);
    }
}
```

---

## Dead-Lettering

**Dead-lettering** stores events that can't be delivered after all retry attempts are exhausted.

### Dead-Letter Configuration

```bash
# Create subscription with dead-letter storage
az eventgrid event-subscription create \
  --name mySubscription \
  --source-resource-id "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage" \
  --endpoint https://myfunction.azurewebsites.net/api/handler \
  --deadletter-endpoint "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/deadletterstorage/blobServices/default/containers/eventgrid-deadletter"
```

### Dead-Letter Blob Container Structure

```
eventgrid-deadletter/
├── 2024/
│   ├── 01/
│   │   ├── 15/
│   │   │   ├── 14/
│   │   │   │   ├── 30/
│   │   │   │   │   └── deadletter-event-1.json
│   │   │   │   │   └── deadletter-event-2.json
```

**Naming Pattern:**
```
{container}/{year}/{month}/{day}/{hour}/{minute}/{event-id}.json
```

### Dead-Letter Event Format

```json
{
  "id": "9aeb0fdf-c01e-0131-0922-9eb54906e209",
  "eventTime": "2024-01-15T14:30:00Z",
  "eventType": "Microsoft.Storage.BlobCreated",
  "dataVersion": "1.0",
  "metadataVersion": "1",
  "topic": "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage",
  "subject": "/blobServices/default/containers/images/blobs/photo.jpg",
  "data": {
    "api": "PutBlob",
    "contentType": "image/jpeg",
    "url": "https://mystorage.blob.core.windows.net/images/photo.jpg"
  },
  "deadLetterReason": "MaximumDeliveryAttemptsExceeded",
  "deliveryAttempts": 30,
  "lastDeliveryAttemptTime": "2024-01-16T14:30:00Z",
  "lastHttpStatusCode": 503
}
```

### Dead-Letter Reasons

| Reason | Description |
|--------|-------------|
| `MaxDeliveryAttemptsExceeded` | Exceeded maximum retry attempts |
| `EventTimeToLiveExceeded` | Event TTL expired |
| `DestinationEndpointNotFound` | Endpoint deleted or not found |
| `EndpointDisabled` | Endpoint disabled by Event Grid |

### Processing Dead-Letter Events

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;
using System.Text.Json;

public class DeadLetterProcessor
{
    private readonly BlobServiceClient _blobServiceClient;
    
    public DeadLetterProcessor(string connectionString)
    {
        _blobServiceClient = new BlobServiceClient(connectionString);
    }
    
    public async Task ProcessDeadLetterEvents(string containerName)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        
        await foreach (BlobItem blobItem in containerClient.GetBlobsAsync())
        {
            var blobClient = containerClient.GetBlobClient(blobItem.Name);
            BlobDownloadResult download = await blobClient.DownloadContentAsync();
            string content = download.Content.ToString();
            
            var deadLetterEvent = JsonSerializer.Deserialize<DeadLetterEvent>(content);
            
            Console.WriteLine($"Dead Letter Event ID: {deadLetterEvent.Id}");
            Console.WriteLine($"Reason: {deadLetterEvent.DeadLetterReason}");
            Console.WriteLine($"Attempts: {deadLetterEvent.DeliveryAttempts}");
            Console.WriteLine($"Last Status: {deadLetterEvent.LastHttpStatusCode}");
            
            // Reprocess or investigate
            await ReprocessEvent(deadLetterEvent);
            
            // Optionally delete after processing
            await blobClient.DeleteAsync();
        }
    }
    
    private async Task ReprocessEvent(DeadLetterEvent deadLetterEvent)
    {
        // Implement reprocessing logic
        // Option 1: Manual investigation
        // Option 2: Republish to Event Grid
        // Option 3: Send to alternate processing pipeline
    }
}
```

### Dead-Letter Best Practices

1. **Always configure dead-letter storage** for production subscriptions
2. **Monitor dead-letter container** with alerts
3. **Set up automated processing** for common failures
4. **Investigate patterns** in dead-lettered events
5. **Clean up old dead-letter events** periodically

---

## Delayed Delivery

Event Grid automatically **delays delivery** to endpoints that are consistently unhealthy.

### Delayed Delivery Behavior

**Trigger Conditions:**
- Multiple consecutive delivery failures
- Consistent error responses (500, 503, 504)
- Network connectivity issues

**Delay Duration:**
- Starts with **5 minutes**
- Gradually increases to **maximum of 1 hour**
- Automatically resumes when endpoint recovers

**Benefits:**
- Reduces load on unhealthy endpoints
- Allows time for recovery
- Prevents overwhelming failing services

### Monitoring Delayed Delivery

```bash
# Check delivery metrics
az monitor metrics list \
  --resource "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/myTopic" \
  --metric "DeliveryFailedCount" \
  --start-time 2024-01-15T00:00:00Z \
  --end-time 2024-01-15T23:59:59Z

# Check matched events
az monitor metrics list \
  --resource "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/myTopic" \
  --metric "MatchedEventCount" \
  --start-time 2024-01-15T00:00:00Z
```

---

## Output Batching

**Output batching** delivers multiple events in a single HTTP request to improve throughput and reduce costs.

### Batch Configuration

```bash
# Create subscription with batching
az eventgrid event-subscription create \
  --name mySubscription \
  --source-resource-id "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage" \
  --endpoint https://myfunction.azurewebsites.net/api/handler \
  --max-events-per-batch 100 \
  --preferred-batch-size-in-kilobytes 128
```

### Batch Properties

| Property | Description | Min | Max | Default |
|----------|-------------|-----|-----|---------|
| `maxEventsPerBatch` | Maximum events in single request | 1 | 5000 | 1 |
| `preferredBatchSizeInKilobytes` | Preferred batch size in KB | 1 | 1024 | 64 |

**Batching Behavior:**
- Event Grid waits for either **max events** or **preferred size**, whichever comes first
- Small wait time (~1 second) to collect events
- Does NOT wait indefinitely (optimizes for latency)
- Useful for high-volume scenarios

### Handling Batched Events

```csharp
[FunctionName("BatchEventHandler")]
public static async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req,
    ILogger log)
{
    string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
    var events = JsonSerializer.Deserialize<EventGridEvent[]>(requestBody);
    
    log.LogInformation($"Received batch of {events.Length} events");
    
    var tasks = events.Select(async eventGridEvent =>
    {
        try
        {
            await ProcessEvent(eventGridEvent);
            log.LogInformation($"Processed event: {eventGridEvent.Id}");
        }
        catch (Exception ex)
        {
            log.LogError($"Failed to process event {eventGridEvent.Id}: {ex.Message}");
            throw; // Fail entire batch
        }
    });
    
    await Task.WhenAll(tasks);
    
    return new OkResult();
}
```

### Batch Processing Strategies

**Strategy 1: Parallel Processing**
```csharp
// Process all events in parallel
await Task.WhenAll(events.Select(ProcessEvent));
```

**Strategy 2: Sequential Processing**
```csharp
// Process events one at a time
foreach (var evt in events)
{
    await ProcessEvent(evt);
}
```

**Strategy 3: Batched Database Operations**
```csharp
// Batch insert to database
var records = events.Select(evt => new Record
{
    Id = evt.Id,
    Data = evt.Data.ToString()
});

await dbContext.Records.AddRangeAsync(records);
await dbContext.SaveChangesAsync();
```

---

## Custom Delivery Properties

Add **custom HTTP headers** to event deliveries for authentication, routing, or metadata.

### Configure Custom Headers

```bash
# Create subscription with custom headers
az eventgrid event-subscription create \
  --name mySubscription \
  --source-resource-id "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage" \
  --endpoint https://myapi.example.com/webhooks/events \
  --delivery-attribute-mapping \
    X-Custom-Header static myValue \
    X-Event-Source dynamic subject \
    X-Event-Id dynamic id \
    X-Event-Time dynamic eventTime \
    X-API-Key static "api-key-12345"
```

### Custom Header Types

| Type | Description | Example |
|------|-------------|---------|
| **Static** | Fixed value for all events | API key, tenant ID |
| **Dynamic** | Value from event property | Subject, ID, timestamp |

### Delivery Attribute Mapping (ARM Template)

```json
{
  "type": "Microsoft.EventGrid/eventSubscriptions",
  "properties": {
    "destination": {
      "endpointType": "WebHook",
      "properties": {
        "endpointUrl": "https://myapi.example.com/webhooks/events"
      }
    },
    "deliveryWithResourceIdentity": {
      "identity": {
        "type": "SystemAssigned"
      },
      "destination": {
        "endpointType": "WebHook",
        "properties": {
          "endpointUrl": "https://myapi.example.com/webhooks/events",
          "deliveryAttributeMappings": [
            {
              "name": "X-API-Key",
              "type": "Static",
              "properties": {
                "value": "api-key-12345",
                "isSecret": true
              }
            },
            {
              "name": "X-Event-Subject",
              "type": "Dynamic",
              "properties": {
                "sourceField": "subject"
              }
            },
            {
              "name": "X-Event-Id",
              "type": "Dynamic",
              "properties": {
                "sourceField": "id"
              }
            }
          ]
        }
      }
    }
  }
}
```

### Custom Header Limits

- **Maximum headers**: 10
- **Maximum header size**: 4096 bytes each
- **Secret headers**: Can be marked as secret (not displayed in portal)

### Use Cases for Custom Headers

1. **Authentication**: API keys, tokens
2. **Routing**: Tenant IDs, region identifiers
3. **Correlation**: Trace IDs, request IDs
4. **Metadata**: Environment, version, source system

---

## Monitoring Event Delivery

### Key Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| **PublishSuccessCount** | Events successfully published | N/A |
| **PublishFailCount** | Events failed to publish | > 0 |
| **MatchedEventCount** | Events matched to subscriptions | Baseline |
| **DeliverySuccessCount** | Events successfully delivered | Baseline |
| **DeliveryFailCount** | Events failed delivery | > 10% |
| **DeadLetterCount** | Events dead-lettered | > 0 |
| **DroppedEventCount** | Events dropped (no dead-letter) | > 0 |
| **DestinationProcessingDurationInMs** | Endpoint processing time | > 1000ms |

### Create Delivery Alert (Azure CLI)

```bash
# Alert on delivery failures
az monitor metrics alert create \
  --name "Event Delivery Failures" \
  --resource-group myRG \
  --scopes "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/myTopic" \
  --condition "avg DeliveryFailCount > 10" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action-group "/subscriptions/{sub-id}/resourceGroups/myRG/providers/microsoft.insights/actionGroups/myActionGroup"
```

### Azure Monitor Query (KQL)

```kusto
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.EVENTGRID"
| where Category == "DeliveryFailures"
| project TimeGenerated, SubscriptionName, EventSubscriptionName, Message, DeliveryAttempts
| order by TimeGenerated desc
| take 100
```

---

## Best Practices

### Retry Policy Design

1. **Production workloads**: Use default retry policy (30 attempts, 24-hour TTL)
2. **Time-sensitive events**: Reduce TTL (e.g., 60 minutes)
3. **Idempotency**: Design handlers to handle duplicate deliveries
4. **Fast failure**: Return 400 for validation errors (don't retry)
5. **Transient errors**: Return 500 to trigger retry

### Dead-Letter Management

1. **Always configure**: Set up dead-letter storage for all subscriptions
2. **Monitor regularly**: Check dead-letter container daily
3. **Automated alerts**: Alert on dead-letter events
4. **Reprocessing pipeline**: Build automated reprocessing for common issues
5. **Retention policy**: Clean up old dead-letter events (e.g., 90 days)

### Performance Optimization

1. **Enable batching**: For high-volume scenarios (100-1000 events/batch)
2. **Async processing**: Process events asynchronously
3. **Quick response**: Return HTTP 200 within 30 seconds
4. **Parallel handlers**: Scale out handler instances
5. **Monitor latency**: Track endpoint processing time

### Error Handling

```csharp
public static class EventHandlerPatterns
{
    // Pattern 1: Return appropriate status codes
    public static IActionResult HandleValidationError()
    {
        return new BadRequestResult(); // Don't retry
    }
    
    public static IActionResult HandleTransientError()
    {
        return new StatusCodeResult(500); // Retry
    }
    
    // Pattern 2: Idempotent processing
    private static HashSet<string> processedEventIds = new();
    
    public static async Task<IActionResult> HandleIdempotent(EventGridEvent evt)
    {
        if (processedEventIds.Contains(evt.Id))
        {
            return new OkResult(); // Already processed
        }
        
        await ProcessEvent(evt);
        processedEventIds.Add(evt.Id);
        
        return new OkResult();
    }
    
    // Pattern 3: Circuit breaker for downstream services
    public static async Task<IActionResult> HandleWithCircuitBreaker(EventGridEvent evt)
    {
        if (await IsDownstreamHealthy())
        {
            await ProcessEvent(evt);
            return new OkResult();
        }
        else
        {
            return new StatusCodeResult(503); // Service unavailable, retry
        }
    }
}
```

---

## Exam Tips for AZ-204

### Key Concepts to Remember

1. **At-least-once delivery**: Events may be delivered multiple times
2. **Retry policy defaults**: 30 attempts, 24-hour (1440 minutes) TTL
3. **Exponential backoff**: Starts at 30 seconds, doubles each retry
4. **Non-retryable errors**: 400, 401 (webhooks), 413
5. **Dead-lettering**: Requires Azure Storage blob container
6. **Output batching**: Max 5000 events or 1024 KB
7. **Custom headers**: Max 10 headers, 4096 bytes each

### Common Exam Scenarios

**Scenario 1**: Events not being retried
- ✅ Check if endpoint returns non-retryable status (400, 413)
- ✅ Verify retry policy configuration
- ❌ Don't assume all errors trigger retry

**Scenario 2**: Need to minimize costs for high-volume events
- ✅ Enable output batching
- ✅ Increase max events per batch
- ❌ Don't process events individually

**Scenario 3**: Events disappearing without trace
- ❌ Dead-letter storage not configured
- ✅ Configure dead-letter blob container
- ✅ Monitor dropped event count

**Scenario 4**: Endpoint overwhelmed by retries
- ✅ Event Grid uses exponential backoff (automatic)
- ✅ Consider delayed delivery feature
- ✅ Scale out endpoint instances

### Remember for Exam

- **Default max attempts**: 30
- **Default TTL**: 1440 minutes (24 hours)
- **First retry wait**: 30 seconds
- **Subsequent retries**: Exponential backoff
- **Dead-letter format**: JSON in blob storage
- **Batching max**: 5000 events or 1024 KB
- **Custom headers**: Up to 10, 4096 bytes each
- **Non-retryable**: 400, 401 (webhooks), 413, 414, 415
- **Retryable**: 500, 503, 504, 408, 429

### Quick Command Reference

```bash
# Configure retry policy
az eventgrid event-subscription create \
  --max-delivery-attempts 10 \
  --event-ttl 60

# Configure dead-letter
--deadletter-endpoint "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{sa}/blobServices/default/containers/{container}"

# Configure batching
--max-events-per-batch 100 \
--preferred-batch-size-in-kilobytes 128

# Add custom headers
--delivery-attribute-mapping \
  X-Custom-Header static myValue \
  X-Event-Id dynamic id
```

---

## Summary

**Event Delivery Durability Features:**
- **Retry mechanism**: Automatic with exponential backoff (30 sec to 13 hours)
- **Retry policy**: Configurable max attempts (1-30) and TTL (1-1440 min)
- **Dead-lettering**: Store undeliverable events in blob storage
- **Output batching**: Deliver up to 5000 events in single request
- **Delayed delivery**: Automatic pause for unhealthy endpoints
- **Custom headers**: Add up to 10 custom HTTP headers

**Best Practices:**
- ✅ Use default retry policy for most scenarios (30 attempts, 24 hours)
- ✅ Always configure dead-letter storage
- ✅ Enable batching for high-volume scenarios
- ✅ Design idempotent event handlers
- ✅ Return appropriate HTTP status codes (400 for validation, 500 for transient errors)
- ✅ Monitor delivery metrics and set up alerts
- ✅ Process dead-lettered events regularly

**Key Takeaways:**
- Event Grid guarantees **at-least-once delivery**
- Events may be **delivered multiple times** (design for idempotency)
- **Non-retryable errors** (400, 413) skip retry logic
- **Dead-letter storage** prevents event loss
- **Batching** improves throughput and reduces costs