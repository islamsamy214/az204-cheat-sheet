# Receive Events by Using Webhooks in Azure Event Grid

## Overview

**Webhooks** are HTTP/HTTPS endpoints that receive events from Azure Event Grid via POST requests. Event Grid pushes events to your webhook endpoint, making it ideal for custom applications and serverless scenarios.

**Key Features:**
- **Push delivery**: Event Grid sends events to your endpoint
- **Endpoint validation**: Prove webhook ownership before receiving events
- **Flexible hosting**: Any HTTP endpoint (cloud, on-premises, containers)
- **Automatic retry**: Built-in retry with exponential backoff
- **Custom authentication**: Add headers for API keys or OAuth tokens

---

## Webhook Endpoint Requirements

### Basic Requirements

| Requirement | Description |
|-------------|-------------|
| **Protocol** | HTTPS (HTTP supported but not recommended) |
| **Response Time** | Must respond within **30 seconds** |
| **Response Code** | Return **HTTP 200 OK** for successful processing |
| **Certificate** | Valid TLS/SSL certificate (no self-signed in production) |
| **Accessibility** | Publicly accessible or via Azure Relay/Private Endpoint |
| **Validation** | Must handle endpoint validation handshake |

### Certificate Requirements

**Production:**
- ✅ Certificate from commercial certificate authority (CA)
- ✅ Domain validation or extended validation
- ❌ No self-signed certificates

**Development/Testing:**
- ⚠️ Self-signed certificates allowed with Azure CLI flag
- Not recommended for production

```bash
# Allow self-signed certificates (development only)
az eventgrid event-subscription create \
  --name mySubscription \
  --source-resource-id $TOPIC_ID \
  --endpoint https://localhost:5001/api/events \
  --azure-active-directory-tenant-id $TENANT_ID \
  --endpoint-type webhook \
  --deadletter-endpoint $DEADLETTER_ENDPOINT
```

---

## Webhook Endpoint Validation

Event Grid requires **endpoint validation** to prove you own the webhook URL before delivering events.

### Validation Purpose

- **Prevent abuse**: Stop malicious actors from sending events to your endpoints
- **Verify ownership**: Ensure you control the webhook URL
- **Avoid accidental subscriptions**: Prevent sending events to wrong endpoints

### Auto-Handled Validation

These Azure services **automatically handle validation**:

| Service | Notes |
|---------|-------|
| **Azure Functions** | With Event Grid trigger binding |
| **Logic Apps** | With Event Grid connector |
| **Azure Automation** | Webhook-triggered runbooks |

**No manual validation code needed for these services!**

---

## Validation Methods

### Method 1: Synchronous Handshake (Recommended)

**How it works:**
1. Event Grid sends `SubscriptionValidationEvent` to your endpoint
2. Your endpoint extracts `validationCode` from event data
3. Your endpoint responds **synchronously** with the validation code
4. Subscription becomes active immediately

**Validation Event Format:**
```json
[{
  "id": "2d1781af-3a4c-4d7c-bd0c-e34b19da4e66",
  "topic": "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/myTopic",
  "subject": "",
  "data": {
    "validationCode": "512d38b6-c7b8-40c8-89fe-f46f9e9622b6",
    "validationUrl": "https://rp-eastus2.eventgrid.azure.net:553/eventsubscriptions/mySubscription/validate?id=512d38b6-c7b8-40c8-89fe-f46f9e9622b6&t=2024-01-15T14:30:00.0000000Z&apiVersion=2018-05-01-preview&token=..."
  },
  "eventType": "Microsoft.EventGrid.SubscriptionValidationEvent",
  "eventTime": "2024-01-15T14:30:00.0000000Z",
  "metadataVersion": "1",
  "dataVersion": "2"
}]
```

**Response Format:**
```json
{
  "validationResponse": "512d38b6-c7b8-40c8-89fe-f46f9e9622b6"
}
```

**C# Implementation:**
```csharp
using Microsoft.AspNetCore.Mvc;
using Azure.Messaging.EventGrid;
using Azure.Messaging.EventGrid.SystemEvents;

[ApiController]
[Route("api/[controller]")]
public class EventsController : ControllerBase
{
    [HttpPost]
    [HttpOptions] // Support OPTIONS requests for validation
    public async Task<IActionResult> Post()
    {
        // Read request body
        using var reader = new StreamReader(Request.Body);
        string requestBody = await reader.ReadToEndAsync();
        
        // Parse events
        EventGridEvent[] events = EventGridEvent.ParseMany(BinaryData.FromString(requestBody));
        
        foreach (EventGridEvent eventGridEvent in events)
        {
            // Handle validation event
            if (eventGridEvent.EventType == "Microsoft.EventGrid.SubscriptionValidationEvent")
            {
                var validationData = eventGridEvent.Data.ToObjectFromJson<SubscriptionValidationEventData>();
                var validationCode = validationData.ValidationCode;
                
                // Return validation response
                return Ok(new { validationResponse = validationCode });
            }
            
            // Handle business events
            await ProcessEvent(eventGridEvent);
        }
        
        return Ok();
    }
    
    private async Task ProcessEvent(EventGridEvent eventGridEvent)
    {
        // Your event processing logic
        Console.WriteLine($"Event Type: {eventGridEvent.EventType}");
        Console.WriteLine($"Subject: {eventGridEvent.Subject}");
        Console.WriteLine($"Data: {eventGridEvent.Data}");
    }
}
```

**Python Flask Implementation:**
```python
from flask import Flask, request, jsonify
import json

app = Flask(__name__)

@app.route('/api/events', methods=['POST', 'OPTIONS'])
def handle_events():
    # Handle OPTIONS request
    if request.method == 'OPTIONS':
        return '', 200
    
    events = request.json
    
    for event in events:
        event_type = event.get('eventType')
        
        # Handle validation event
        if event_type == 'Microsoft.EventGrid.SubscriptionValidationEvent':
            validation_code = event['data']['validationCode']
            return jsonify({'validationResponse': validation_code}), 200
        
        # Handle business events
        elif event_type == 'Microsoft.Storage.BlobCreated':
            blob_url = event['data']['url']
            print(f'Blob created: {blob_url}')
            # Process the event
        
        # Handle other event types
        else:
            print(f'Received event: {event_type}')
    
    return jsonify({'status': 'success'}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, ssl_context='adhoc')
```

**Node.js Express Implementation:**
```javascript
const express = require('express');
const app = express();

app.use(express.json());

app.post('/api/events', (req, res) => {
    const events = req.body;
    
    for (const event of events) {
        // Handle validation event
        if (event.eventType === 'Microsoft.EventGrid.SubscriptionValidationEvent') {
            const validationCode = event.data.validationCode;
            return res.status(200).json({ validationResponse: validationCode });
        }
        
        // Handle business events
        console.log(`Event Type: ${event.eventType}`);
        console.log(`Subject: ${event.subject}`);
        console.log(`Data: ${JSON.stringify(event.data)}`);
    }
    
    res.status(200).send('OK');
});

app.listen(5000, () => {
    console.log('Webhook listening on port 5000');
});
```

### Method 2: Asynchronous Handshake (Manual)

**How it works:**
1. Event Grid sends validation event with `validationUrl`
2. You (or automated process) make GET request to `validationUrl` within **5 minutes**
3. Subscription provisioning completes
4. Events start flowing

**When to use:**
- Can't modify webhook code to handle validation event
- Third-party endpoints
- Legacy systems

**Validation Process:**

```bash
# 1. Create subscription (stays in "AwaitingManualAction" state)
az eventgrid event-subscription create \
  --name mySubscription \
  --source-resource-id $TOPIC_ID \
  --endpoint https://mylegacyapp.example.com/events

# 2. Check subscription status
az eventgrid event-subscription show \
  --name mySubscription \
  --source-resource-id $TOPIC_ID \
  --query "provisioningState"
# Output: "AwaitingManualAction"

# 3. Get validation URL from the validation event
# Event Grid sends validation event to your endpoint
# Extract validationUrl from the event data

# 4. Make GET request to validation URL
curl -X GET "https://rp-eastus2.eventgrid.azure.net:553/eventsubscriptions/mySubscription/validate?id=...&apiVersion=2018-05-01-preview&token=..."

# 5. Check subscription status again
az eventgrid event-subscription show \
  --name mySubscription \
  --source-resource-id $TOPIC_ID \
  --query "provisioningState"
# Output: "Succeeded"
```

**Validation URL Expiration:**
- ⏱️ **5 minutes** to complete validation
- ❌ After 5 minutes, validation URL expires
- 🔄 Delete and recreate subscription if expired

**Provisioning States:**

| State | Description |
|-------|-------------|
| `Creating` | Subscription being created |
| `AwaitingManualAction` | Waiting for manual validation (asynchronous handshake) |
| `Succeeded` | Subscription active, receiving events |
| `Failed` | Validation failed or error occurred |

---

## Webhook Event Delivery

### Request Format

**HTTP Headers:**
```
POST /api/events HTTP/1.1
Host: mywebhook.example.com
Content-Type: application/json; charset=utf-8
aeg-subscription-name: mySubscription
aeg-event-type: Notification
aeg-data-version: 1.0
aeg-metadata-version: 1
aeg-delivery-count: 1
```

**Request Body (CloudEvents):**
```json
[
  {
    "specversion": "1.0",
    "type": "Microsoft.Storage.BlobCreated",
    "source": "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage",
    "subject": "/blobServices/default/containers/images/blobs/photo.jpg",
    "id": "9aeb0fdf-c01e-0131-0922-9eb54906e209",
    "time": "2024-01-15T14:30:00Z",
    "datacontenttype": "application/json",
    "data": {
      "api": "PutBlob",
      "contentType": "image/jpeg",
      "contentLength": 524288,
      "blobType": "BlockBlob",
      "url": "https://mystorage.blob.core.windows.net/images/photo.jpg"
    }
  }
]
```

### Event Grid HTTP Headers

| Header | Description | Example |
|--------|-------------|---------|
| `aeg-subscription-name` | Name of the event subscription | `mySubscription` |
| `aeg-event-type` | Type of delivery | `Notification`, `SubscriptionValidation` |
| `aeg-data-version` | Data schema version | `1.0` |
| `aeg-metadata-version` | Event metadata version | `1` |
| `aeg-delivery-count` | Delivery attempt number | `1` (first attempt) |

### Response Requirements

**Success Response:**
```
HTTP/1.1 200 OK
Content-Length: 0
```

**Processing Time:**
- Must respond within **30 seconds**
- Longer = timeout and retry
- Process events asynchronously if needed

**Asynchronous Processing Pattern:**
```csharp
[HttpPost]
public async Task<IActionResult> Post()
{
    var events = await ParseEventsAsync(Request.Body);
    
    // Queue events for background processing
    foreach (var evt in events)
    {
        await _queue.EnqueueAsync(evt);
    }
    
    // Return 200 immediately
    return Ok();
}
```

---

## Authentication Options

### Option 1: Custom HTTP Headers (API Keys)

```bash
# Add API key as custom header
az eventgrid event-subscription create \
  --name mySubscription \
  --source-resource-id $TOPIC_ID \
  --endpoint https://myapi.example.com/webhooks/events \
  --delivery-attribute-mapping \
    X-API-Key static "your-api-key-here" \
    X-Client-ID static "client-123"
```

**Webhook Validation:**
```csharp
[HttpPost]
public async Task<IActionResult> Post()
{
    // Validate API key
    if (!Request.Headers.TryGetValue("X-API-Key", out var apiKey) ||
        apiKey != "your-api-key-here")
    {
        return Unauthorized();
    }
    
    // Process events
    await ProcessEventsAsync(Request.Body);
    return Ok();
}
```

### Option 2: Azure AD OAuth Token

**Configure OAuth authentication:**
```json
{
  "deliveryWithResourceIdentity": {
    "identity": {
      "type": "SystemAssigned"
    },
    "destination": {
      "endpointType": "WebHook",
      "properties": {
        "endpointUrl": "https://myapi.azurewebsites.net/api/events",
        "azureActiveDirectoryTenantId": "tenant-id",
        "azureActiveDirectoryApplicationIdOrUri": "api://myapi"
      }
    }
  }
}
```

**Validate JWT Token in Webhook:**
```csharp
using Microsoft.Identity.Web;

[Authorize]
[HttpPost]
public async Task<IActionResult> Post()
{
    // Token automatically validated by [Authorize] attribute
    var userId = User.Identity?.Name;
    
    await ProcessEventsAsync(Request.Body);
    return Ok();
}
```

### Option 3: Query String Parameters

```bash
# Add authentication token in URL
az eventgrid event-subscription create \
  --name mySubscription \
  --source-resource-id $TOPIC_ID \
  --endpoint "https://myapi.example.com/webhooks/events?code=abc123xyz"
```

---

## Webhook Best Practices

### Design Patterns

**Pattern 1: Quick Acknowledgment + Background Processing**
```csharp
[HttpPost]
public async Task<IActionResult> Post()
{
    var events = await ParseEventsAsync(Request.Body);
    
    // Queue for background processing
    foreach (var evt in events)
    {
        await _backgroundQueue.QueueBackgroundWorkItemAsync(async token =>
        {
            await ProcessEventAsync(evt, token);
        });
    }
    
    // Return immediately
    return Ok();
}
```

**Pattern 2: Idempotent Processing**
```csharp
private static readonly ConcurrentDictionary<string, bool> _processedEvents 
    = new ConcurrentDictionary<string, bool>();

public async Task<IActionResult> Post()
{
    var events = await ParseEventsAsync(Request.Body);
    
    foreach (var evt in events)
    {
        // Skip if already processed
        if (_processedEvents.ContainsKey(evt.Id))
        {
            continue;
        }
        
        await ProcessEventAsync(evt);
        _processedEvents.TryAdd(evt.Id, true);
    }
    
    return Ok();
}
```

**Pattern 3: Circuit Breaker for Downstream Services**
```csharp
using Polly;
using Polly.CircuitBreaker;

private static readonly AsyncCircuitBreakerPolicy _circuitBreaker =
    Policy
        .Handle<Exception>()
        .CircuitBreakerAsync(
            exceptionsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30)
        );

[HttpPost]
public async Task<IActionResult> Post()
{
    if (_circuitBreaker.CircuitState == CircuitState.Open)
    {
        // Return 503 to trigger retry
        return StatusCode(503, "Circuit breaker open");
    }
    
    try
    {
        await _circuitBreaker.ExecuteAsync(async () =>
        {
            await ProcessEventsAsync(Request.Body);
        });
        
        return Ok();
    }
    catch
    {
        return StatusCode(500);
    }
}
```

### Error Handling

```csharp
[HttpPost]
public async Task<IActionResult> Post()
{
    try
    {
        var events = await ParseEventsAsync(Request.Body);
        
        foreach (var evt in events)
        {
            try
            {
                // Validate event
                if (string.IsNullOrEmpty(evt.Subject))
                {
                    _logger.LogWarning($"Invalid event: {evt.Id}");
                    // Return 400 - don't retry invalid events
                    return BadRequest("Invalid event format");
                }
                
                // Process event
                await ProcessEventAsync(evt);
            }
            catch (InvalidOperationException ex)
            {
                _logger.LogError($"Validation error: {ex.Message}");
                // Return 400 - don't retry validation errors
                return BadRequest(ex.Message);
            }
            catch (Exception ex)
            {
                _logger.LogError($"Processing error: {ex.Message}");
                // Return 500 - retry transient errors
                return StatusCode(500);
            }
        }
        
        return Ok();
    }
    catch (Exception ex)
    {
        _logger.LogError($"Webhook error: {ex.Message}");
        return StatusCode(500);
    }
}
```

### Security Best Practices

1. **HTTPS Only**: Always use HTTPS endpoints
2. **Validate Certificates**: Use valid TLS/SSL certificates
3. **Authenticate Requests**: Use API keys, OAuth, or custom headers
4. **Validate Event Origin**: Check `aeg-subscription-name` header
5. **Rate Limiting**: Implement rate limiting to prevent abuse
6. **Input Validation**: Validate all event data
7. **Logging**: Log all events for audit and troubleshooting

---

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Validation failing | Endpoint not responding correctly | Check validation code logic |
| Timeout errors | Processing takes > 30 seconds | Implement async processing |
| Certificate errors | Self-signed or expired certificate | Use valid CA-signed certificate |
| 401 Unauthorized | Missing/invalid authentication | Check API keys or OAuth config |
| Events not received | Subscription filters too restrictive | Review filter configuration |
| Duplicate events | Retries after timeout | Implement idempotent processing |

### Validation Troubleshooting

**Check validation response:**
```bash
# Test validation manually
curl -X POST https://mywebhook.example.com/api/events \
  -H "Content-Type: application/json" \
  -d '[{
    "id": "test-id",
    "eventType": "Microsoft.EventGrid.SubscriptionValidationEvent",
    "data": {
      "validationCode": "test-code-123"
    }
  }]'

# Expected response:
# {"validationResponse":"test-code-123"}
```

### Delivery Troubleshooting

**Check subscription status:**
```bash
az eventgrid event-subscription show \
  --name mySubscription \
  --source-resource-id $TOPIC_ID \
  --query "{State:provisioningState, Endpoint:destination.endpointUrl}"
```

**Check delivery metrics:**
```bash
az monitor metrics list \
  --resource $TOPIC_ID \
  --metric "DeliveryFailedCount" \
  --start-time 2024-01-15T00:00:00Z
```

---

## Exam Tips for AZ-204

### Key Concepts to Remember

1. **Validation required**: All webhooks must validate endpoint ownership
2. **Two validation methods**: Synchronous (recommended) and asynchronous (manual)
3. **30-second timeout**: Webhook must respond within 30 seconds
4. **HTTP 200 response**: Required for successful event processing
5. **Auto-validation**: Azure Functions, Logic Apps, Automation handle automatically
6. **5-minute window**: Asynchronous validation must complete within 5 minutes
7. **HTTPS required**: Valid TLS/SSL certificate (no self-signed in production)

### Common Exam Scenarios

**Scenario 1**: Webhook validation failing
- ✅ Implement synchronous handshake (return validationResponse)
- ✅ Check if response format is correct
- ❌ Don't ignore validation event

**Scenario 2**: Events timing out
- ✅ Implement asynchronous processing (queue events)
- ✅ Return HTTP 200 immediately
- ❌ Don't process events synchronously if > 30 seconds

**Scenario 3**: Secure webhook endpoint
- ✅ Add custom headers with API keys
- ✅ Use Azure AD OAuth tokens
- ❌ Don't rely on URL obscurity alone

**Scenario 4**: Legacy system can't handle validation
- ✅ Use asynchronous handshake (manual GET request)
- ⏱️ Complete within 5 minutes

### Remember for Exam

- **Validation event type**: `Microsoft.EventGrid.SubscriptionValidationEvent`
- **Response property**: `validationResponse`
- **Timeout**: 30 seconds
- **Async validation window**: 5 minutes
- **Auto-handled**: Functions, Logic Apps, Automation
- **Required response**: HTTP 200 OK
- **Certificate**: Valid CA-signed (no self-signed)
- **Retries**: Automatic with exponential backoff
- **Idempotency**: Handle duplicate events

### Quick Command Reference

```bash
# Create webhook subscription
az eventgrid event-subscription create \
  --name <name> \
  --source-resource-id <topic-id> \
  --endpoint https://<webhook-url>

# Add API key header
--delivery-attribute-mapping \
  X-API-Key static "<api-key>"

# Check subscription status
az eventgrid event-subscription show \
  --name <name> \
  --source-resource-id <topic-id> \
  --query "provisioningState"
```

---

## Summary

**Webhook Endpoint Validation:**
- **Synchronous handshake**: Return `validationResponse` immediately (recommended)
- **Asynchronous handshake**: Manual GET request within 5 minutes
- **Auto-handled**: Azure Functions, Logic Apps, Automation

**Webhook Requirements:**
- HTTPS endpoint with valid certificate
- Respond within 30 seconds
- Return HTTP 200 OK
- Handle validation event

**Best Practices:**
- ✅ Implement asynchronous processing for long-running tasks
- ✅ Design idempotent event handlers
- ✅ Use authentication (API keys, OAuth)
- ✅ Log all events for troubleshooting
- ✅ Return appropriate status codes (400 for validation, 500 for transient errors)
- ✅ Implement circuit breakers for downstream dependencies

**Security:**
- Use HTTPS only
- Valid TLS/SSL certificate
- Authenticate requests (custom headers, OAuth)
- Validate event origin
- Implement rate limiting