# Exercise: Route Custom Events to Webhook Endpoint

## Exercise Overview

In this hands-on exercise, you will:
1. Create a custom Event Grid topic
2. Deploy a webhook endpoint (Azure Function)
3. Create an event subscription with filters
4. Publish custom events
5. Verify event delivery
6. Test filtering and retry mechanisms

**Estimated Time:** 30-40 minutes

**Prerequisites:**
- Azure subscription
- Azure CLI installed
- Basic knowledge of Azure Functions

---

## Architecture Diagram

```
┌─────────────────────┐
│  Your Application   │
│  (Event Publisher)  │
└──────────┬──────────┘
           │ POST events
           ▼
┌─────────────────────────┐
│  Azure Event Grid Topic │
│   (Custom Topic)        │
└──────────┬──────────────┘
           │ Filter & Route
           ▼
┌─────────────────────────┐
│  Event Subscription     │
│  (Filter Configuration) │
└──────────┬──────────────┘
           │ Push events
           ▼
┌─────────────────────────┐
│  Azure Function         │
│  (Webhook Endpoint)     │
└─────────────────────────┘
```

---

## Part 1: Set Up Environment

### Step 1.1: Define Variables

```bash
# Set variables
RESOURCE_GROUP="rg-eventgrid-lab"
LOCATION="eastus"
TOPIC_NAME="topic-orders-$(openssl rand -hex 4)"
FUNCTION_APP_NAME="func-eventhandler-$(openssl rand -hex 4)"
STORAGE_ACCOUNT="steventgrid$(openssl rand -hex 4)"
SUBSCRIPTION_NAME="order-subscription"

echo "Resource Group: $RESOURCE_GROUP"
echo "Topic Name: $TOPIC_NAME"
echo "Function App: $FUNCTION_APP_NAME"
```

### Step 1.2: Create Resource Group

```bash
# Create resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION

echo "✓ Resource group created"
```

---

## Part 2: Create Event Grid Custom Topic

### Step 2.1: Create Topic

```bash
# Create custom topic
az eventgrid topic create \
  --name $TOPIC_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

echo "✓ Event Grid topic created"
```

### Step 2.2: Get Topic Endpoint and Keys

```bash
# Get topic endpoint
TOPIC_ENDPOINT=$(az eventgrid topic show \
  --name $TOPIC_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "endpoint" \
  --output tsv)

# Get topic key
TOPIC_KEY=$(az eventgrid topic key list \
  --name $TOPIC_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "key1" \
  --output tsv)

# Get topic resource ID
TOPIC_ID=$(az eventgrid topic show \
  --name $TOPIC_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "id" \
  --output tsv)

echo "Topic Endpoint: $TOPIC_ENDPOINT"
echo "Topic Key: $TOPIC_KEY"
echo "Topic ID: $TOPIC_ID"
```

---

## Part 3: Create Webhook Endpoint (Azure Function)

### Step 3.1: Create Storage Account

```bash
# Create storage account for Function App
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS

echo "✓ Storage account created"
```

### Step 3.2: Create Function App

```bash
# Create Function App (Linux, .NET 8)
az functionapp create \
  --name $FUNCTION_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --storage-account $STORAGE_ACCOUNT \
  --consumption-plan-location $LOCATION \
  --runtime dotnet-isolated \
  --runtime-version 8 \
  --functions-version 4 \
  --os-type Linux

echo "✓ Function App created"
```

### Step 3.3: Get Function App URL

```bash
# Get Function App default hostname
FUNCTION_APP_URL="https://${FUNCTION_APP_NAME}.azurewebsites.net"

echo "Function App URL: $FUNCTION_APP_URL"
```

### Step 3.4: Create Event Handler Function

Create a file named `EventGridTriggerFunction.cs`:

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Logging;
using Azure.Messaging.EventGrid;
using Azure.Messaging.EventGrid.SystemEvents;

namespace EventGridFunctions
{
    public class EventGridTriggerFunction
    {
        private readonly ILogger<EventGridTriggerFunction> _logger;

        public EventGridTriggerFunction(ILogger<EventGridTriggerFunction> logger)
        {
            _logger = logger;
        }

        [Function("EventGridTrigger")]
        public void Run([EventGridTrigger] EventGridEvent eventGridEvent)
        {
            _logger.LogInformation($"=== Event Received ===");
            _logger.LogInformation($"Event ID: {eventGridEvent.Id}");
            _logger.LogInformation($"Event Type: {eventGridEvent.EventType}");
            _logger.LogInformation($"Event Subject: {eventGridEvent.Subject}");
            _logger.LogInformation($"Event Time: {eventGridEvent.EventTime}");
            _logger.LogInformation($"Event Data: {eventGridEvent.Data}");
            
            // Handle different event types
            switch (eventGridEvent.EventType)
            {
                case "MyApp.Orders.OrderCreated":
                    HandleOrderCreated(eventGridEvent);
                    break;
                case "MyApp.Orders.OrderShipped":
                    HandleOrderShipped(eventGridEvent);
                    break;
                case "MyApp.Orders.OrderCancelled":
                    HandleOrderCancelled(eventGridEvent);
                    break;
                default:
                    _logger.LogInformation($"Unknown event type: {eventGridEvent.EventType}");
                    break;
            }
        }

        private void HandleOrderCreated(EventGridEvent evt)
        {
            _logger.LogInformation("Processing OrderCreated event...");
            // Your business logic here
            var data = evt.Data.ToObjectFromJson<OrderCreatedData>();
            _logger.LogInformation($"Order ID: {data.OrderId}");
            _logger.LogInformation($"Customer: {data.CustomerEmail}");
            _logger.LogInformation($"Amount: ${data.Amount}");
        }

        private void HandleOrderShipped(EventGridEvent evt)
        {
            _logger.LogInformation("Processing OrderShipped event...");
            // Your business logic here
        }

        private void HandleOrderCancelled(EventGridEvent evt)
        {
            _logger.LogInformation("Processing OrderCancelled event...");
            // Your business logic here
        }
    }

    // Data models
    public class OrderCreatedData
    {
        public string OrderId { get; set; }
        public string CustomerEmail { get; set; }
        public decimal Amount { get; set; }
        public string Status { get; set; }
    }
}
```

**Alternative: HTTP Webhook Function (if not using Event Grid trigger)**

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.Extensions.Logging;
using System.Net;
using System.Text.Json;
using Azure.Messaging.EventGrid;
using Azure.Messaging.EventGrid.SystemEvents;

namespace EventGridFunctions
{
    public class WebhookFunction
    {
        private readonly ILogger<WebhookFunction> _logger;

        public WebhookFunction(ILogger<WebhookFunction> logger)
        {
            _logger = logger;
        }

        [Function("Webhook")]
        public async Task<HttpResponseData> Run(
            [HttpTrigger(AuthorizationLevel.Function, "post", "options")] HttpRequestData req)
        {
            // Handle OPTIONS request
            if (req.Method == "OPTIONS")
            {
                var optionsResponse = req.CreateResponse(HttpStatusCode.OK);
                return optionsResponse;
            }

            // Read request body
            string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
            
            // Parse Event Grid events
            EventGridEvent[] events = EventGridEvent.ParseMany(BinaryData.FromString(requestBody));

            foreach (EventGridEvent eventGridEvent in events)
            {
                // Handle validation event
                if (eventGridEvent.EventType == "Microsoft.EventGrid.SubscriptionValidationEvent")
                {
                    var validationData = eventGridEvent.Data.ToObjectFromJson<SubscriptionValidationEventData>();
                    var validationResponse = new { validationResponse = validationData.ValidationCode };
                    
                    var response = req.CreateResponse(HttpStatusCode.OK);
                    await response.WriteAsJsonAsync(validationResponse);
                    return response;
                }

                // Log event details
                _logger.LogInformation($"Event ID: {eventGridEvent.Id}");
                _logger.LogInformation($"Event Type: {eventGridEvent.EventType}");
                _logger.LogInformation($"Subject: {eventGridEvent.Subject}");
                _logger.LogInformation($"Data: {eventGridEvent.Data}");
                
                // Process event
                await ProcessEvent(eventGridEvent);
            }

            var successResponse = req.CreateResponse(HttpStatusCode.OK);
            return successResponse;
        }

        private async Task ProcessEvent(EventGridEvent evt)
        {
            // Your event processing logic
            _logger.LogInformation($"Processing event: {evt.EventType}");
            await Task.CompletedTask;
        }
    }
}
```

### Step 3.5: Deploy Function

```bash
# Deploy function (if using local development)
# Navigate to function project directory
cd EventGridFunctions

# Publish to Azure
func azure functionapp publish $FUNCTION_APP_NAME

echo "✓ Function deployed"
```

---

## Part 4: Create Event Subscription

### Step 4.1: Get Function Endpoint

**For Event Grid Trigger:**
```bash
# Get system key for Event Grid trigger
FUNCTION_KEY=$(az functionapp keys list \
  --name $FUNCTION_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "systemKeys.eventgrid_extension" \
  --output tsv)

FUNCTION_ENDPOINT="${FUNCTION_APP_URL}/runtime/webhooks/EventGrid?functionName=EventGridTrigger&code=${FUNCTION_KEY}"
```

**For HTTP Trigger Webhook:**
```bash
# Get function key
FUNCTION_KEY=$(az functionapp function keys list \
  --name $FUNCTION_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --function-name Webhook \
  --query "default" \
  --output tsv)

FUNCTION_ENDPOINT="${FUNCTION_APP_URL}/api/Webhook?code=${FUNCTION_KEY}"
```

### Step 4.2: Create Subscription with Filters

```bash
# Create event subscription with filters
az eventgrid event-subscription create \
  --name $SUBSCRIPTION_NAME \
  --source-resource-id $TOPIC_ID \
  --endpoint $FUNCTION_ENDPOINT \
  --included-event-types \
    MyApp.Orders.OrderCreated \
    MyApp.Orders.OrderShipped \
    MyApp.Orders.OrderCancelled \
  --subject-begins-with "/orders/" \
  --max-delivery-attempts 5 \
  --event-ttl 60

echo "✓ Event subscription created"
```

### Step 4.3: Verify Subscription

```bash
# Check subscription status
az eventgrid event-subscription show \
  --name $SUBSCRIPTION_NAME \
  --source-resource-id $TOPIC_ID \
  --query "{State:provisioningState, Endpoint:destination.endpointBaseUrl}"
```

---

## Part 5: Publish Custom Events

### Step 5.1: Publish Single Event

```bash
# Publish OrderCreated event
az eventgrid event publish \
  --topic-name $TOPIC_NAME \
  --resource-group $RESOURCE_GROUP \
  --events '[
    {
      "id": "order-001",
      "eventType": "MyApp.Orders.OrderCreated",
      "subject": "/orders/region/west/order/001",
      "eventTime": "'$(date -u +"%Y-%m-%dT%H:%M:%SZ")'",
      "data": {
        "orderId": "ORD-2024-001",
        "customerEmail": "customer@example.com",
        "amount": 149.99,
        "status": "Pending",
        "items": [
          {
            "productId": "PROD-001",
            "quantity": 2,
            "price": 74.99
          }
        ]
      },
      "dataVersion": "1.0"
    }
  ]'

echo "✓ Event published"
```

### Step 5.2: Publish Multiple Events (Batch)

```bash
# Publish multiple events
az eventgrid event publish \
  --topic-name $TOPIC_NAME \
  --resource-group $RESOURCE_GROUP \
  --events '[
    {
      "id": "order-002",
      "eventType": "MyApp.Orders.OrderCreated",
      "subject": "/orders/region/east/order/002",
      "eventTime": "'$(date -u +"%Y-%m-%dT%H:%M:%SZ")'",
      "data": {
        "orderId": "ORD-2024-002",
        "customerEmail": "alice@example.com",
        "amount": 299.99,
        "status": "Pending"
      },
      "dataVersion": "1.0"
    },
    {
      "id": "order-003",
      "eventType": "MyApp.Orders.OrderShipped",
      "subject": "/orders/region/west/order/003",
      "eventTime": "'$(date -u +"%Y-%m-%dT%H:%M:%SZ")'",
      "data": {
        "orderId": "ORD-2024-003",
        "trackingNumber": "TRACK-12345",
        "carrier": "UPS"
      },
      "dataVersion": "1.0"
    },
    {
      "id": "order-004",
      "eventType": "MyApp.Orders.OrderCancelled",
      "subject": "/orders/region/east/order/004",
      "eventTime": "'$(date -u +"%Y-%m-%dT%H:%M:%SZ")'",
      "data": {
        "orderId": "ORD-2024-004",
        "reason": "Customer request",
        "refundAmount": 199.99
      },
      "dataVersion": "1.0"
    }
  ]'

echo "✓ Batch events published"
```

### Step 5.3: Publish with Python Script

Create `publish_event.py`:

```python
from azure.eventgrid import EventGridPublisherClient
from azure.core.credentials import AzureKeyCredential
from datetime import datetime
import os

# Configuration
endpoint = os.environ['TOPIC_ENDPOINT']
key = os.environ['TOPIC_KEY']

# Create client
credential = AzureKeyCredential(key)
client = EventGridPublisherClient(endpoint, credential)

# Create event
event = {
    "id": "order-005",
    "eventType": "MyApp.Orders.OrderCreated",
    "subject": "/orders/region/west/order/005",
    "eventTime": datetime.utcnow().isoformat() + "Z",
    "data": {
        "orderId": "ORD-2024-005",
        "customerEmail": "bob@example.com",
        "amount": 499.99,
        "status": "Pending"
    },
    "dataVersion": "1.0"
}

# Publish event
client.send(event)
print("✓ Event published from Python")
```

Run the script:
```bash
export TOPIC_ENDPOINT=$TOPIC_ENDPOINT
export TOPIC_KEY=$TOPIC_KEY
python publish_event.py
```

---

## Part 6: Verify Event Delivery

### Step 6.1: Check Function Logs

```bash
# Stream function logs
az functionapp log tail \
  --name $FUNCTION_APP_NAME \
  --resource-group $RESOURCE_GROUP
```

**Expected Output:**
```
2024-01-15T14:30:00.123 [Information] === Event Received ===
2024-01-15T14:30:00.124 [Information] Event ID: order-001
2024-01-15T14:30:00.125 [Information] Event Type: MyApp.Orders.OrderCreated
2024-01-15T14:30:00.126 [Information] Event Subject: /orders/region/west/order/001
2024-01-15T14:30:00.127 [Information] Processing OrderCreated event...
2024-01-15T14:30:00.128 [Information] Order ID: ORD-2024-001
2024-01-15T14:30:00.129 [Information] Customer: customer@example.com
2024-01-15T14:30:00.130 [Information] Amount: $149.99
```

### Step 6.2: Check Event Grid Metrics

```bash
# Check published events
az monitor metrics list \
  --resource $TOPIC_ID \
  --metric "PublishSuccessCount" \
  --start-time $(date -u -d '1 hour ago' +"%Y-%m-%dT%H:%M:%SZ") \
  --interval PT1M

# Check matched events
az monitor metrics list \
  --resource $TOPIC_ID \
  --metric "MatchedEventCount" \
  --start-time $(date -u -d '1 hour ago' +"%Y-%m-%dT%H:%M:%SZ") \
  --interval PT1M

# Check delivered events
az monitor metrics list \
  --resource $TOPIC_ID \
  --metric "DeliverySuccessCount" \
  --start-time $(date -u -d '1 hour ago' +"%Y-%m-%dT%H:%M:%SZ") \
  --interval PT1M

# Check failed deliveries
az monitor metrics list \
  --resource $TOPIC_ID \
  --metric "DeliveryFailedCount" \
  --start-time $(date -u -d '1 hour ago' +"%Y-%m-%dT%H:%M:%SZ") \
  --interval PT1M
```

---

## Part 7: Test Event Filtering

### Step 7.1: Test Subject Filter (Should Match)

```bash
# Event with matching subject
az eventgrid event publish \
  --topic-name $TOPIC_NAME \
  --resource-group $RESOURCE_GROUP \
  --events '[{
    "id": "test-match",
    "eventType": "MyApp.Orders.OrderCreated",
    "subject": "/orders/region/south/order/999",
    "eventTime": "'$(date -u +"%Y-%m-%dT%H:%M:%SZ")'",
    "data": { "test": "This should be delivered" },
    "dataVersion": "1.0"
  }]'

# Check function logs - should see event
```

### Step 7.2: Test Subject Filter (Should NOT Match)

```bash
# Event with non-matching subject
az eventgrid event publish \
  --topic-name $TOPIC_NAME \
  --resource-group $RESOURCE_GROUP \
  --events '[{
    "id": "test-no-match",
    "eventType": "MyApp.Orders.OrderCreated",
    "subject": "/products/product/123",
    "eventTime": "'$(date -u +"%Y-%m-%dT%H:%M:%SZ")'",
    "data": { "test": "This should NOT be delivered" },
    "dataVersion": "1.0"
  }]'

# Check function logs - should NOT see event
# Event filtered out by subject filter
```

### Step 7.3: Test Event Type Filter (Should NOT Match)

```bash
# Event with non-matching type
az eventgrid event publish \
  --topic-name $TOPIC_NAME \
  --resource-group $RESOURCE_GROUP \
  --events '[{
    "id": "test-wrong-type",
    "eventType": "MyApp.Products.ProductCreated",
    "subject": "/orders/region/west/order/888",
    "eventTime": "'$(date -u +"%Y-%m-%dT%H:%M:%SZ")'",
    "data": { "test": "Wrong event type" },
    "dataVersion": "1.0"
  }]'

# Check function logs - should NOT see event
# Event filtered out by event type filter
```

---

## Part 8: Configure Advanced Features

### Step 8.1: Add Dead-Letter Storage

```bash
# Create storage container for dead-letter
DEADLETTER_CONTAINER="eventgrid-deadletter"

az storage container create \
  --name $DEADLETTER_CONTAINER \
  --account-name $STORAGE_ACCOUNT

# Get storage account resource ID
STORAGE_ID=$(az storage account show \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query "id" \
  --output tsv)

# Update subscription with dead-letter
az eventgrid event-subscription update \
  --name $SUBSCRIPTION_NAME \
  --source-resource-id $TOPIC_ID \
  --deadletter-endpoint "${STORAGE_ID}/blobServices/default/containers/${DEADLETTER_CONTAINER}"

echo "✓ Dead-letter configured"
```

### Step 8.2: Add Advanced Filtering

```bash
# Create new subscription with advanced filters
az eventgrid event-subscription create \
  --name "high-value-orders" \
  --source-resource-id $TOPIC_ID \
  --endpoint $FUNCTION_ENDPOINT \
  --included-event-types MyApp.Orders.OrderCreated \
  --advanced-filter data.amount NumberGreaterThan 1000 \
  --advanced-filter data.status StringIn Pending Confirmed

echo "✓ Advanced filtering configured"
```

### Step 8.3: Enable Output Batching

```bash
# Update subscription with batching
az eventgrid event-subscription update \
  --name $SUBSCRIPTION_NAME \
  --source-resource-id $TOPIC_ID \
  --max-events-per-batch 10 \
  --preferred-batch-size-in-kilobytes 64

echo "✓ Output batching enabled"
```

---

## Part 9: Monitor and Troubleshoot

### Step 9.1: View Subscription Details

```bash
# Get full subscription details
az eventgrid event-subscription show \
  --name $SUBSCRIPTION_NAME \
  --source-resource-id $TOPIC_ID
```

### Step 9.2: Test Retry Mechanism

**Simulate failed delivery:**

1. Stop the Function App:
```bash
az functionapp stop --name $FUNCTION_APP_NAME --resource-group $RESOURCE_GROUP
```

2. Publish event:
```bash
az eventgrid event publish \
  --topic-name $TOPIC_NAME \
  --resource-group $RESOURCE_GROUP \
  --events '[{
    "id": "retry-test",
    "eventType": "MyApp.Orders.OrderCreated",
    "subject": "/orders/test",
    "eventTime": "'$(date -u +"%Y-%m-%dT%H:%M:%SZ")'",
    "data": { "test": "retry" },
    "dataVersion": "1.0"
  }]'
```

3. Check delivery attempts:
```bash
# Wait a few minutes, check metrics
az monitor metrics list \
  --resource $TOPIC_ID \
  --metric "DeliveryFailedCount" \
  --start-time $(date -u -d '10 minutes ago' +"%Y-%m-%dT%H:%M:%SZ")
```

4. Restart Function App:
```bash
az functionapp start --name $FUNCTION_APP_NAME --resource-group $RESOURCE_GROUP
```

5. Event Grid will retry delivery automatically

---

## Part 10: Clean Up Resources

```bash
# Delete entire resource group
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait

echo "✓ Resources deleted"
```

---

## Key Takeaways

1. **Custom Topics**: Created Event Grid topic for custom events
2. **Webhook Endpoint**: Deployed Azure Function as event handler
3. **Event Subscription**: Configured filters for event routing
4. **Event Publishing**: Published events via Azure CLI and SDK
5. **Filtering**: Tested event type and subject filtering
6. **Advanced Features**: Configured dead-letter and batching
7. **Monitoring**: Verified delivery with metrics and logs

---

## Troubleshooting Guide

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| Validation failing | Function not responding | Check function logs, ensure HTTPS |
| Events not delivered | Filter too restrictive | Review filter configuration |
| Timeout errors | Function processing > 30s | Implement async processing |
| 401 errors | Invalid function key | Regenerate and update endpoint |
| Dead-letter events | Persistent failures | Check dead-letter container, investigate |

---

## Additional Challenges

**Challenge 1:** Add authentication to webhook using custom headers

**Challenge 2:** Implement idempotent event processing (track event IDs)

**Challenge 3:** Create multiple subscriptions routing to different functions based on event type

**Challenge 4:** Implement circuit breaker pattern in event handler

**Challenge 5:** Set up automated reprocessing of dead-letter events

---

## Exam Tips

**Key Points to Remember:**
- Custom topics must be explicitly created
- Webhooks require endpoint validation
- Filters reduce unnecessary event delivery
- Dead-letter storage requires Azure Storage blob container
- Retry policy defaults: 30 attempts, 24-hour TTL
- Maximum event size: 1 MB
- Billing: 64 KB increments

**Common Exam Scenarios:**
- Creating custom topics for application events
- Configuring event subscriptions with filters
- Implementing webhook validation
- Setting up dead-letter storage
- Troubleshooting event delivery issues

---

## Summary

You have successfully:
✅ Created an Event Grid custom topic
✅ Deployed a webhook endpoint (Azure Function)
✅ Created event subscriptions with filtering
✅ Published custom events
✅ Verified event delivery
✅ Configured advanced features (dead-letter, batching)
✅ Monitored metrics and logs
✅ Tested retry mechanisms

**Next Steps:**
- Explore Event Grid with Azure services (Storage, IoT Hub)
- Implement advanced filtering scenarios
- Build production event-driven applications
- Learn about Event Grid domains for multi-tenant scenarios