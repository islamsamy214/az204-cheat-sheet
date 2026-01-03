# Azure Event Grid Overview

## What is Azure Event Grid?

**Azure Event Grid** is a fully managed event routing service that enables reactive, event-driven programming using a **publish-subscribe model**. It provides reliable message delivery at massive scale, allowing you to build event-driven architectures.

### Key Characteristics

- **Serverless and fully managed**: No infrastructure to provision or manage
- **HTTP and MQTT support**: Multiple protocols for different scenarios
  - **HTTP**: Request-response messaging, cloud events delivery
  - **MQTT**: IoT messaging, bidirectional communication
- **CloudEvents v1.0 compliant**: Industry-standard event schema
- **Massive scale**: Millions of events per second
- **Low cost**: Pay only for what you use
- **High availability**: 99.99% SLA
- **Advanced filtering**: Route events based on content

### Event Grid Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        EVENT PUBLISHERS                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ Azure    │  │ Custom   │  │ Azure    │  │ Partner  │       │
│  │ Services │  │ Apps     │  │ IoT      │  │ Services │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
└───────┼─────────────┼─────────────┼─────────────┼──────────────┘
        │             │             │             │
        └─────────────┴─────────────┴─────────────┘
                      │
                      ▼
        ┌─────────────────────────────┐
        │     AZURE EVENT GRID        │
        │  ┌───────────────────────┐  │
        │  │   Event Routing &     │  │
        │  │   Filtering Engine    │  │
        │  └───────────────────────┘  │
        │  ┌───────────────────────┐  │
        │  │   Topics (Endpoints)  │  │
        │  │   - Custom Topics     │  │
        │  │   - System Topics     │  │
        │  │   - Partner Topics    │  │
        │  └───────────────────────┘  │
        └─────────────┬───────────────┘
                      │
        ┌─────────────┴─────────────┐
        │   EVENT SUBSCRIPTIONS     │
        │   (Filters & Routes)      │
        └─────────────┬───────────────┘
                      │
        ┌─────────────┴─────────────────────────┐
        │                                       │
        ▼                                       ▼
┌───────────────────┐               ┌───────────────────┐
│  PUSH DELIVERY    │               │  PULL DELIVERY    │
│                   │               │                   │
│ ┌───────────────┐ │               │ ┌───────────────┐ │
│ │ Azure         │ │               │ │ HTTP Client   │ │
│ │ Functions     │ │               │ │ Applications  │ │
│ ├───────────────┤ │               │ └───────────────┘ │
│ │ Logic Apps    │ │               └───────────────────┘
│ ├───────────────┤ │
│ │ Event Hubs    │ │
│ ├───────────────┤ │
│ │ Service Bus   │ │
│ ├───────────────┤ │
│ │ Webhooks      │ │
│ └───────────────┘ │
└───────────────────┘
```

---

## Core Event Grid Concepts

### 1. Events

An **event** is the smallest unit of information that fully describes something that happened in a system.

**Event Characteristics:**
- **Immutable**: Events describe facts about what happened
- **Lightweight**: Maximum size of 1 MB per event
- **Structured**: JSON format following CloudEvents or Event Grid schema
- **Timestamped**: Contains event generation time

**Common Event Properties:**
```json
{
  "specversion": "1.0",
  "type": "com.example.someevent",
  "source": "/mycontext",
  "subject": "resource/operation",
  "id": "A234-1234-1234",
  "time": "2024-01-15T10:30:00Z",
  "datacontenttype": "application/json",
  "data": {
    "appinfoA": "value",
    "appinfoB": "another value"
  }
}
```

**Event Size and Billing:**
- Maximum event size: **1 MB**
- Billing increments: **64 KB**
- Example: A 130 KB event is billed as 3 operations (192 KB rounded up)

### 2. Publishers

A **publisher** is the application or service that sends events to Event Grid.

**Types of Publishers:**

| Publisher Type | Description | Examples |
|---------------|-------------|----------|
| **Azure Services** | Built-in Azure resources that emit events | Storage accounts, Resource Manager, IoT Hub |
| **Custom Applications** | Your applications that publish custom events | Web apps, background services, microservices |
| **Partner Services** | SaaS providers integrated with Event Grid | Auth0, SAP, Microsoft Graph API |
| **IoT Devices** | IoT devices publishing over MQTT | Sensors, gateways, edge devices |

**Publishing Methods:**
```bash
# Publish events using Azure CLI
az eventgrid event publish \
  --topic-name myTopic \
  --resource-group myResourceGroup \
  --events '[{
    "id": "event-001",
    "eventType": "recordInserted",
    "subject": "myapp/vehicles/motorcycles",
    "eventTime": "2024-01-15T10:30:00Z",
    "data": {
      "make": "Ducati",
      "model": "Monster"
    },
    "dataVersion": "1.0"
  }]'
```

### 3. Event Sources

An **event source** is where the event happens. Each event source is related to one or more event types.

**Built-in Event Sources:**

| Event Source | Common Event Types | Use Case |
|-------------|-------------------|----------|
| **Azure Blob Storage** | `Microsoft.Storage.BlobCreated`, `Microsoft.Storage.BlobDeleted` | Trigger workflows when files are uploaded |
| **Azure Resource Manager** | `Microsoft.Resources.ResourceWriteSuccess` | Track resource deployments |
| **Azure Event Hubs** | `Microsoft.EventHub.CaptureFileCreated` | Process captured event data |
| **Azure IoT Hub** | `Microsoft.Devices.DeviceCreated` | Device lifecycle management |
| **Azure Media Services** | `Microsoft.Media.JobStateChange` | Monitor encoding jobs |
| **Azure Container Registry** | `Microsoft.ContainerRegistry.ImagePushed` | Trigger CI/CD pipelines |
| **Azure Service Bus** | `Microsoft.ServiceBus.ActiveMessagesAvailableWithNoListeners` | Monitor queue health |
| **Azure App Configuration** | `Microsoft.AppConfiguration.KeyValueModified` | Dynamic configuration updates |

### 4. Topics

A **topic** is an endpoint where publishers send events. Topics are containers that group related events.

**Topic Types:**

#### System Topics
- **Definition**: Topics for events from Azure services
- **Creation**: Automatically created (no manual creation needed)
- **Scope**: Tied to specific Azure resource
- **Naming**: Based on resource ID
- **Lifecycle**: Deleted when source resource is deleted

```bash
# Subscribe to system topic events
az eventgrid event-subscription create \
  --name myStorageSubscription \
  --source-resource-id "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount" \
  --endpoint https://myfunction.azurewebsites.net/api/handler \
  --included-event-types Microsoft.Storage.BlobCreated Microsoft.Storage.BlobDeleted
```

#### Custom Topics
- **Definition**: Topics for custom application events
- **Creation**: Explicitly created by users
- **Scope**: Independent resources
- **Naming**: User-defined
- **Flexibility**: Full control over event schema

```bash
# Create custom topic
az eventgrid topic create \
  --name myCustomTopic \
  --resource-group myResourceGroup \
  --location eastus

# Get topic endpoint
az eventgrid topic show \
  --name myCustomTopic \
  --resource-group myResourceGroup \
  --query "endpoint" \
  --output tsv
```

#### Partner Topics
- **Definition**: Topics for events from SaaS providers
- **Creation**: Partner creates, you activate
- **Examples**: Auth0, Microsoft Graph, SAP
- **Use Case**: Integrate third-party events

**Topic Comparison:**

| Feature | System Topics | Custom Topics | Partner Topics |
|---------|--------------|---------------|----------------|
| **Creation** | Automatic | Manual | Partner creates |
| **Event Schema** | Azure-defined | User-defined | Partner-defined |
| **Lifecycle** | Tied to resource | Independent | Partner-managed |
| **Use Case** | Azure service events | Custom app events | Third-party events |
| **Cost** | Included with resource | Standard pricing | Varies by partner |

### 5. Event Subscriptions

An **event subscription** tells Event Grid which events on a topic you want to receive and where to send them.

**Subscription Configuration:**

```json
{
  "name": "mySubscription",
  "properties": {
    "destination": {
      "endpointType": "WebHook",
      "properties": {
        "endpointUrl": "https://myapp.com/api/events"
      }
    },
    "filter": {
      "includedEventTypes": [
        "Microsoft.Storage.BlobCreated"
      ],
      "subjectBeginsWith": "/blobServices/default/containers/images/",
      "subjectEndsWith": ".jpg",
      "advancedFilters": [
        {
          "operatorType": "NumberGreaterThan",
          "key": "data.contentLength",
          "value": 1024
        }
      ]
    },
    "retryPolicy": {
      "maxDeliveryAttempts": 30,
      "eventTimeToLiveInMinutes": 1440
    }
  }
}
```

**Key Subscription Properties:**

| Property | Description | Options |
|----------|-------------|---------|
| **Destination** | Where to send events | Webhook, Azure Function, Event Hubs, Service Bus, Storage Queue, Hybrid Connection |
| **Filter** | Which events to receive | Event types, subject patterns, advanced filters |
| **Retry Policy** | Delivery retry behavior | Max attempts (1-30), TTL (1-1440 min) |
| **Dead Letter** | Failed event storage | Storage account blob container |
| **Expiration** | Subscription end time | DateTime or duration |

**Create Event Subscription:**

```bash
# Subscribe with Azure Function endpoint
az eventgrid event-subscription create \
  --name imageProcessingSubscription \
  --source-resource-id "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage" \
  --endpoint-type azurefunction \
  --endpoint "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.Web/sites/myFunctionApp/functions/processImage" \
  --included-event-types Microsoft.Storage.BlobCreated \
  --subject-begins-with "/blobServices/default/containers/images/" \
  --subject-ends-with ".jpg"
```

### 6. Event Handlers

An **event handler** is the destination where events are sent. Handlers process or react to events.

**Supported Event Handlers:**

| Handler Type | Use Case | Delivery Method | Limitations |
|-------------|----------|-----------------|-------------|
| **Azure Functions** | Serverless event processing | Direct invocation | 230 seconds timeout |
| **Webhooks** | Custom HTTP endpoints | HTTP POST | Must validate endpoint |
| **Logic Apps** | Workflow automation | Direct trigger | 120 seconds timeout |
| **Azure Event Hubs** | Event streaming | Push to stream | N/A |
| **Azure Service Bus** | Message queuing | Push to queue/topic | 256 KB message size |
| **Storage Queues** | Simple queuing | Push to queue | 64 KB message size |
| **Hybrid Connections** | On-premises endpoints | Azure Relay | Requires relay setup |

**Azure Function Handler Example (C#):**

```csharp
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.EventGrid;
using Microsoft.Extensions.Logging;
using Azure.Messaging.EventGrid;

public static class BlobCreatedHandler
{
    [FunctionName("BlobCreatedHandler")]
    public static void Run(
        [EventGridTrigger] EventGridEvent eventGridEvent,
        ILogger log)
    {
        log.LogInformation($"Event Type: {eventGridEvent.EventType}");
        log.LogInformation($"Event Subject: {eventGridEvent.Subject}");
        log.LogInformation($"Event Data: {eventGridEvent.Data}");
        
        // Process the event
        if (eventGridEvent.EventType == "Microsoft.Storage.BlobCreated")
        {
            var blobData = eventGridEvent.Data.ToObjectFromJson<BlobCreatedEventData>();
            log.LogInformation($"Blob URL: {blobData.Url}");
            
            // Your processing logic here
        }
    }
}
```

**Webhook Handler Requirements:**

1. **Endpoint Validation**: Respond to validation handshake
2. **HTTP 200 Response**: Return success within 30 seconds
3. **TLS/SSL**: HTTPS endpoint with valid certificate
4. **Idempotency**: Handle duplicate events gracefully

```python
# Python Flask webhook example
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/api/events', methods=['POST'])
def handle_event():
    events = request.json
    
    for event in events:
        # Handle validation event
        if event['eventType'] == 'Microsoft.EventGrid.SubscriptionValidationEvent':
            validation_code = event['data']['validationCode']
            return jsonify({'validationResponse': validation_code})
        
        # Handle business events
        elif event['eventType'] == 'Microsoft.Storage.BlobCreated':
            blob_url = event['data']['url']
            print(f"New blob created: {blob_url}")
            # Process the event
    
    return jsonify({'status': 'success'}), 200
```

---

## Event Delivery Models

### Push Delivery (Default)

Event Grid pushes events to configured endpoints.

**Characteristics:**
- **Automatic**: Events delivered immediately when published
- **At-least-once**: Guaranteed delivery with retries
- **Retry Policy**: Exponential backoff (30 sec to 1 day)
- **Dead Lettering**: Failed events stored for later processing

**Best For:**
- Azure Functions, Logic Apps
- Webhooks with high availability
- Event-driven architectures

**Push Delivery Flow:**
```
Publisher → Event Grid → [Filter] → [Retry if needed] → Handler
                ↓ (if failed after retries)
           Dead Letter Location
```

### Pull Delivery

Consumers pull events from Event Grid on demand.

**Characteristics:**
- **On-Demand**: Consumer controls when to receive events
- **Batch Processing**: Retrieve multiple events at once
- **Client Acknowledgment**: Consumer acknowledges processed events
- **No Endpoint**: No webhook validation required

**Best For:**
- Batch processing scenarios
- Applications behind firewalls
- Client-controlled event processing

**Pull Delivery Code Example (C#):**

```csharp
using Azure;
using Azure.Messaging.EventGrid.Namespaces;

var endpoint = new Uri("https://mynamespace.eastus-1.eventgrid.azure.net");
var credential = new AzureKeyCredential(topicKey);
var client = new EventGridReceiverClient(endpoint, "mytopic", "mysubscription", credential);

// Receive events
ReceiveResult result = await client.ReceiveAsync(maxEvents: 10, maxWaitTime: TimeSpan.FromSeconds(30));

foreach (ReceiveDetails details in result.Value)
{
    CloudEvent cloudEvent = details.Event;
    BrokerProperties brokerProperties = details.BrokerProperties;
    
    // Process the event
    Console.WriteLine($"Event Type: {cloudEvent.Type}");
    Console.WriteLine($"Event Data: {cloudEvent.Data}");
    
    // Acknowledge the event
    await client.AcknowledgeAsync(new[] { brokerProperties.LockToken });
}
```

---

## Security in Event Grid

### Authentication

**Publisher Authentication:**
- **Access Keys**: Shared access keys for custom topics
- **SAS Tokens**: Time-limited access tokens
- **Azure AD**: OAuth 2.0 with managed identities

**Handler Authentication:**
- **Endpoint Validation**: Prove ownership during subscription
- **Event Delivery**: Optional authentication headers

### Authorization

**Azure RBAC Roles:**

| Role | Permissions | Use Case |
|------|-------------|----------|
| **Event Grid Contributor** | Full control over Event Grid resources | Admins managing topics |
| **Event Grid Data Sender** | Publish events to topics | Applications publishing events |
| **Event Grid Subscription Reader** | Read event subscriptions | Monitoring and auditing |
| **Event Grid Subscription Contributor** | Manage event subscriptions | Operations team |

```bash
# Grant Data Sender role to managed identity
az role assignment create \
  --role "EventGrid Data Sender" \
  --assignee <managed-identity-object-id> \
  --scope "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/myTopic"
```

### Network Security

- **Private Endpoints**: Access topics via VNet
- **IP Filtering**: Restrict publisher IP addresses
- **Managed Identity**: No credentials in code

---

## Common Use Cases

### 1. Serverless Application Integration

**Scenario**: Process uploaded images automatically

```
Azure Blob Storage → Event Grid → Azure Functions → Thumbnail Creation
                                 → Azure Functions → Image Recognition
                                 → Azure Functions → Database Update
```

### 2. Ops Automation

**Scenario**: Auto-tag resources when created

```
Azure Resource Manager → Event Grid → Azure Function → Apply Tags
                                    → Logic App → Send Notification
```

### 3. Application Integration

**Scenario**: Order processing pipeline

```
Order API → Event Grid (Custom Topic) → Azure Function (Validate Order)
                                      → Service Bus Queue (Inventory)
                                      → Event Hubs (Analytics)
```

### 4. IoT Telemetry

**Scenario**: Process device telemetry

```
IoT Devices (MQTT) → Event Grid → Azure Functions → Time Series Insights
                                 → Event Hubs → Stream Analytics
```

---

## Event Grid vs. Other Azure Messaging Services

| Feature | **Event Grid** | **Event Hubs** | **Service Bus** |
|---------|---------------|---------------|----------------|
| **Pattern** | Pub/Sub (Reactive) | Streaming (Big data) | Message Queue |
| **Message Size** | 1 MB | 1 MB | 256 KB (Premium: 100 MB) |
| **Ordering** | No guarantee | Per partition | FIFO (sessions) |
| **Delivery** | Push + Pull | Pull | Pull |
| **Retention** | No retention (immediate) | 1-90 days | 14 days max |
| **Throughput** | Millions/sec | Millions/sec | Thousands/sec |
| **Latency** | Sub-second | Real-time | Low latency |
| **Use Case** | Event notifications | Telemetry ingestion | Transactional messaging |
| **Filtering** | Advanced filtering | Consumer group | Message filters |

**When to Use Event Grid:**
- ✅ React to state changes in Azure resources
- ✅ Integrate multiple services with event-driven patterns
- ✅ Build serverless applications
- ✅ Need advanced filtering and routing
- ✅ Want push-based delivery

**When NOT to Use Event Grid:**
- ❌ Need message ordering guarantees
- ❌ Need long-term event retention
- ❌ Need complex message workflows (use Service Bus)
- ❌ Need high-throughput streaming (use Event Hubs)

---

## Quick Start Example

### Step 1: Create Custom Topic

```bash
# Create resource group
az group create --name rg-eventgrid --location eastus

# Create custom topic
az eventgrid topic create \
  --name mytopic \
  --resource-group rg-eventgrid \
  --location eastus

# Get topic endpoint and key
TOPIC_ENDPOINT=$(az eventgrid topic show \
  --name mytopic \
  --resource-group rg-eventgrid \
  --query "endpoint" --output tsv)

TOPIC_KEY=$(az eventgrid topic key list \
  --name mytopic \
  --resource-group rg-eventgrid \
  --query "key1" --output tsv)
```

### Step 2: Create Event Subscription

```bash
# Subscribe with webhook endpoint
az eventgrid event-subscription create \
  --name mysubscription \
  --source-resource-id "/subscriptions/{sub-id}/resourceGroups/rg-eventgrid/providers/Microsoft.EventGrid/topics/mytopic" \
  --endpoint https://mywebhook.azurewebsites.net/api/events \
  --included-event-types MyApp.Orders.OrderCreated
```

### Step 3: Publish Event

```bash
# Publish event
az eventgrid event publish \
  --topic-name mytopic \
  --resource-group rg-eventgrid \
  --events '[
    {
      "id": "order-001",
      "eventType": "MyApp.Orders.OrderCreated",
      "subject": "orders/motorcycles",
      "eventTime": "2024-01-15T10:30:00Z",
      "data": {
        "orderId": "12345",
        "customerId": "CUST-001",
        "amount": 15000.00
      },
      "dataVersion": "1.0"
    }
  ]'
```

---

## Best Practices

### Design Patterns

1. **Event Naming**: Use clear, hierarchical event types
   ```
   ✅ MyApp.Orders.OrderCreated
   ❌ orderCreated
   ```

2. **Subject Hierarchy**: Structure subjects for easy filtering
   ```
   ✅ /orders/region/west/store/101
   ❌ order-west-101
   ```

3. **Idempotency**: Design handlers to process duplicate events safely
   ```csharp
   // Store processed event IDs
   if (await processedEvents.Contains(eventId))
   {
       return; // Already processed
   }
   ```

4. **Error Handling**: Implement proper retry and dead-letter handling

5. **Event Versioning**: Include data version for schema evolution
   ```json
   {
     "dataVersion": "2.0",
     "data": { /* new schema */ }
   }
   ```

### Performance Optimization

- **Batch Publishing**: Send multiple events in one request
- **Async Handlers**: Process events asynchronously
- **Parallel Processing**: Use multiple handler instances
- **Filter Early**: Apply filters at subscription level

### Security Best Practices

- ✅ Use managed identities instead of access keys
- ✅ Enable private endpoints for sensitive workloads
- ✅ Validate webhook endpoints properly
- ✅ Use HTTPS for all endpoints
- ✅ Implement least privilege access with RBAC

---

## Exam Tips for AZ-204

### Key Concepts to Remember

1. **Event Grid is for reactive programming** - push-based event distribution
2. **System topics** are automatic; **custom topics** require creation
3. **Event subscriptions** filter and route events to handlers
4. **CloudEvents 1.0** is the preferred schema standard
5. **Maximum event size**: 1 MB (billed in 64 KB increments)

### Common Exam Scenarios

**Scenario 1**: Trigger Azure Function when blob is uploaded
- ✅ Use Event Grid with BlobCreated event
- ❌ Don't use polling or timers

**Scenario 2**: Process events from third-party SaaS
- ✅ Use Partner Topics
- ❌ Don't build custom integration

**Scenario 3**: Need guaranteed message ordering
- ❌ Event Grid doesn't guarantee order
- ✅ Use Service Bus with sessions instead

**Scenario 4**: High-volume telemetry ingestion
- ❌ Event Grid isn't for streaming
- ✅ Use Event Hubs instead

### Important Commands

```bash
# Create custom topic
az eventgrid topic create --name <name> --resource-group <rg> --location <location>

# Create event subscription
az eventgrid event-subscription create --name <name> --source-resource-id <id> --endpoint <url>

# Publish event
az eventgrid event publish --topic-name <name> --resource-group <rg> --events <json>

# List event types
az eventgrid topic-type list

# Grant access
az role assignment create --role "EventGrid Data Sender" --assignee <identity>
```

### Troubleshooting Checklist

- ❓ Events not delivered? Check endpoint validation and retry policy
- ❓ Handler timing out? Ensure response within 30 seconds
- ❓ Events missing? Check event filters and subscription configuration
- ❓ Authentication failing? Verify access keys or managed identity setup
- ❓ Dead letter storage? Configure blob container for failed events

### Remember for Exam

- **At-least-once delivery**: Events may be delivered multiple times
- **30-second webhook timeout**: Handler must respond quickly
- **Retry policy defaults**: 30 attempts, 24-hour TTL
- **System topics** tied to resource lifecycle
- **Advanced filtering** supports 25 conditions per subscription
- **RBAC roles**: Know Data Sender, Contributor, Subscription Reader
- **No event retention**: Events delivered immediately (use Event Hubs for retention)

---

## Summary

Azure Event Grid enables **event-driven architectures** with:
- **Publishers** that send events
- **Topics** that organize events
- **Subscriptions** that filter and route events
- **Handlers** that process events

**Key Takeaways:**
- Use **system topics** for Azure service events
- Use **custom topics** for application events
- Use **partner topics** for third-party events
- Apply **filters** to route specific events
- Configure **retry policies** for reliable delivery
- Use **dead-letter** storage for failed events
- Implement **idempotent handlers** for duplicate events

Event Grid is ideal for **reactive programming**, **serverless applications**, and **service integration** where you need to respond to events as they happen.