# Event Schemas in Azure Event Grid

## Overview

Azure Event Grid supports two event schema formats for describing events:
1. **Event Grid Schema** - Azure's native event format
2. **CloudEvents Schema** - Industry-standard format (v1.0 specification)

Both schemas provide a structured way to describe **what happened**, **when it happened**, and **where it happened**, along with custom data about the event.

---

## CloudEvents v1.0 Schema (Recommended)

**CloudEvents** is an open specification for describing event data in a common format. It provides interoperability across services, platforms, and systems.

### Why Use CloudEvents?

- ✅ **Industry Standard**: CNCF (Cloud Native Computing Foundation) specification
- ✅ **Interoperability**: Works across cloud providers and platforms
- ✅ **Future-Proof**: Actively maintained and evolving
- ✅ **Tool Support**: Broad SDK and tooling ecosystem
- ✅ **Recommended by Microsoft**: Preferred format for new applications

### CloudEvents Schema Structure

```json
{
  "specversion": "1.0",
  "type": "com.example.someevent",
  "source": "/mycontext/subcontext",
  "subject": "larger-context/specific-resource",
  "id": "A234-1234-1234",
  "time": "2024-01-15T13:45:30.0000000Z",
  "datacontenttype": "application/json",
  "dataschema": "https://mycompany.com/schemas/v1",
  "data": {
    "appinfoA": "abc",
    "appinfoB": 123,
    "appinfoC": true
  }
}
```

### CloudEvents Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `specversion` | string | ✅ Yes | CloudEvents version (always "1.0") |
| `type` | string | ✅ Yes | Event type (e.g., "com.example.object.created") |
| `source` | URI | ✅ Yes | Context in which event occurred |
| `id` | string | ✅ Yes | Unique identifier for the event |
| `time` | timestamp | ❌ No | When event occurred (RFC 3339 format) |
| `subject` | string | ❌ No | Subject of the event in context of source |
| `datacontenttype` | string | ❌ No | Content type of data (e.g., "application/json") |
| `dataschema` | URI | ❌ No | Schema that data adheres to |
| `data` | object | ❌ No | Event-specific data payload |

### CloudEvents Property Details

#### specversion
- **Purpose**: Indicates CloudEvents specification version
- **Value**: Always `"1.0"`
- **Example**: `"specversion": "1.0"`

#### type
- **Purpose**: Describes the type of event
- **Format**: Reverse domain notation recommended
- **Examples**:
  ```
  "com.microsoft.storage.blobcreated"
  "com.example.orders.ordercreated"
  "io.github.pull_request.opened"
  ```

#### source
- **Purpose**: Identifies the context where the event occurred
- **Format**: URI reference
- **Examples**:
  ```
  "/subscriptions/{sub-id}/resourceGroups/rg/providers/Microsoft.Storage/storageAccounts/myaccount"
  "/tenants/my-tenant/applications/my-app"
  "https://api.example.com/v1/orders"
  ```

#### id
- **Purpose**: Uniquely identifies the event
- **Requirements**: 
  - Must be unique within the scope of the source
  - Should be immutable
- **Examples**:
  ```
  "A234-1234-1234"
  "550e8400-e29b-41d4-a716-446655440000"
  "event-2024-01-15-001"
  ```

#### time
- **Purpose**: Timestamp when event occurred
- **Format**: RFC 3339 (ISO 8601)
- **Example**: `"2024-01-15T13:45:30.0000000Z"`

#### subject
- **Purpose**: Describes the subject of the event in the context of the source
- **Use**: Provides filtering capability
- **Examples**:
  ```
  "/blobServices/default/containers/images/blobs/photo.jpg"
  "/orders/12345"
  "/users/user@example.com/profile"
  ```

#### datacontenttype
- **Purpose**: Describes the content type of the data value
- **Common Values**: `application/json`, `application/xml`, `text/plain`
- **Default**: `application/json` if omitted

#### data
- **Purpose**: Contains event-specific information
- **Type**: Any JSON-serializable value
- **Size Limit**: Maximum 1 MB for entire event

### Complete CloudEvents Examples

#### Example 1: Azure Storage Blob Created

```json
{
  "specversion": "1.0",
  "type": "Microsoft.Storage.BlobCreated",
  "source": "/subscriptions/abc123/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage",
  "subject": "/blobServices/default/containers/images/blobs/vacation-photo.jpg",
  "id": "9aeb0fdf-c01e-0131-0922-9eb54906e209",
  "time": "2024-01-15T13:45:30.0000000Z",
  "datacontenttype": "application/json",
  "data": {
    "api": "PutBlob",
    "clientRequestId": "6d79dbfb-0e37-4fc4-981f-442c9ca65760",
    "requestId": "831e1650-001e-001b-66ab-eeb76e000000",
    "eTag": "0x8D4BCC2E4835CD0",
    "contentType": "image/jpeg",
    "contentLength": 524288,
    "blobType": "BlockBlob",
    "url": "https://mystorage.blob.core.windows.net/images/vacation-photo.jpg",
    "sequencer": "00000000000004420000000000028963",
    "storageDiagnostics": {
      "batchId": "b68529f3-68cd-4744-baa4-3c0498ec19f0"
    }
  }
}
```

#### Example 2: Custom Application Event

```json
{
  "specversion": "1.0",
  "type": "com.example.ecommerce.orders.OrderCreated",
  "source": "https://api.example.com/v1/orders",
  "subject": "/orders/ORD-2024-001",
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "time": "2024-01-15T14:30:00Z",
  "datacontenttype": "application/json",
  "dataschema": "https://api.example.com/schemas/order/v2",
  "data": {
    "orderId": "ORD-2024-001",
    "customerId": "CUST-12345",
    "orderDate": "2024-01-15T14:30:00Z",
    "totalAmount": 1599.99,
    "currency": "USD",
    "items": [
      {
        "productId": "PROD-001",
        "quantity": 2,
        "price": 799.99
      }
    ],
    "shippingAddress": {
      "street": "123 Main St",
      "city": "Seattle",
      "state": "WA",
      "zipCode": "98101",
      "country": "USA"
    }
  }
}
```

#### Example 3: IoT Device Telemetry

```json
{
  "specversion": "1.0",
  "type": "io.example.iot.sensors.TemperatureReading",
  "source": "/devices/sensor-001",
  "subject": "/buildings/building-a/floor-3/room-301",
  "id": "temp-reading-20240115-143500",
  "time": "2024-01-15T14:35:00Z",
  "datacontenttype": "application/json",
  "data": {
    "deviceId": "sensor-001",
    "temperature": 22.5,
    "humidity": 45.2,
    "unit": "celsius",
    "batteryLevel": 87,
    "location": {
      "building": "building-a",
      "floor": 3,
      "room": "301"
    }
  }
}
```

---

## Event Grid Schema (Legacy)

The **Event Grid Schema** is Azure's native format, supported for backward compatibility. New applications should use CloudEvents.

### Event Grid Schema Structure

```json
{
  "topic": "/subscriptions/{subscription-id}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage",
  "subject": "/blobServices/default/containers/images/blobs/photo.jpg",
  "eventType": "Microsoft.Storage.BlobCreated",
  "id": "9aeb0fdf-c01e-0131-0922-9eb54906e209",
  "eventTime": "2024-01-15T13:45:30.0000000Z",
  "data": {
    "api": "PutBlob",
    "contentType": "image/jpeg",
    "contentLength": 524288,
    "blobType": "BlockBlob",
    "url": "https://mystorage.blob.core.windows.net/images/photo.jpg"
  },
  "dataVersion": "1.0",
  "metadataVersion": "1"
}
```

### Event Grid Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `topic` | string | ✅ Yes | Full resource path to event source |
| `subject` | string | ✅ Yes | Publisher-defined path to event subject |
| `eventType` | string | ✅ Yes | Registered event type for this source |
| `id` | string | ✅ Yes | Unique identifier for the event |
| `eventTime` | datetime | ✅ Yes | Event generation time (UTC) |
| `data` | object | ✅ Yes | Event-specific data |
| `dataVersion` | string | ✅ Yes | Schema version of data object |
| `metadataVersion` | string | ❌ No | Schema version of event metadata (read-only) |

### Event Grid Property Details

#### topic
- **Purpose**: Full resource path to the event source
- **Set By**: Event Grid (automatically for system topics)
- **Format**: Azure Resource Manager ID
- **Example**: 
  ```
  "/subscriptions/abc123/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage"
  ```

#### subject
- **Purpose**: Publisher-defined path to the event subject
- **Use**: Enables filtering by subject patterns
- **Best Practice**: Use hierarchical structure
- **Examples**:
  ```
  "/blobServices/default/containers/images/blobs/photo.jpg"
  "/orders/region/west/store/101/order/12345"
  ```

#### eventType
- **Purpose**: Identifies the type of event
- **Format**: Dot-notation recommended
- **Azure Examples**:
  ```
  "Microsoft.Storage.BlobCreated"
  "Microsoft.Resources.ResourceWriteSuccess"
  "Microsoft.EventHub.CaptureFileCreated"
  ```
- **Custom Examples**:
  ```
  "MyApp.Orders.OrderCreated"
  "MyApp.Inventory.StockLevelLow"
  ```

#### id
- **Purpose**: Unique identifier for the event
- **Uniqueness**: Must be unique per event
- **Use**: Implement idempotency in handlers

#### eventTime
- **Purpose**: Time the event was generated
- **Format**: ISO 8601 datetime string
- **Timezone**: Always UTC
- **Example**: `"2024-01-15T13:45:30.0000000Z"`

#### data
- **Purpose**: Contains event-specific payload
- **Type**: Object (JSON)
- **Schema**: Defined by event source
- **Versioning**: Tracked via dataVersion

#### dataVersion
- **Purpose**: Schema version of the data object
- **Set By**: Publisher
- **Use**: Handle schema evolution
- **Example**: `"1.0"`, `"2.1"`

### Complete Event Grid Examples

#### Example 1: Storage Blob Deleted

```json
{
  "topic": "/subscriptions/abc123/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage",
  "subject": "/blobServices/default/containers/archive/blobs/old-file.txt",
  "eventType": "Microsoft.Storage.BlobDeleted",
  "id": "7c5d6de5-eb70-4de2-b788-c8e3c9862eaa",
  "eventTime": "2024-01-15T15:20:00.0000000Z",
  "data": {
    "api": "DeleteBlob",
    "requestId": "831e1650-001e-001b-66ab-eeb76e000000",
    "contentType": "text/plain",
    "blobType": "BlockBlob",
    "url": "https://mystorage.blob.core.windows.net/archive/old-file.txt",
    "sequencer": "00000000000004420000000000028963"
  },
  "dataVersion": "1.0",
  "metadataVersion": "1"
}
```

#### Example 2: Resource Manager Deployment

```json
{
  "topic": "/subscriptions/abc123/resourceGroups/myRG",
  "subject": "/subscriptions/abc123/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/newstorage",
  "eventType": "Microsoft.Resources.ResourceWriteSuccess",
  "id": "72f988bf-86f1-41af-91ab-2d7cd011db47",
  "eventTime": "2024-01-15T16:00:00.0000000Z",
  "data": {
    "authorization": {
      "scope": "/subscriptions/abc123/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/newstorage",
      "action": "Microsoft.Storage/storageAccounts/write",
      "evidence": {
        "role": "Owner"
      }
    },
    "claims": {
      "aud": "https://management.core.windows.net/",
      "iss": "https://sts.windows.net/{tenant-id}/",
      "iat": "1705330800",
      "nbf": "1705330800",
      "exp": "1705334400"
    },
    "correlationId": "72f988bf-86f1-41af-91ab-2d7cd011db47",
    "httpRequest": {
      "clientRequestId": "72f988bf-86f1-41af-91ab-2d7cd011db47",
      "clientIpAddress": "203.0.113.42",
      "method": "PUT"
    },
    "resourceProvider": "Microsoft.Storage",
    "resourceUri": "/subscriptions/abc123/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/newstorage",
    "operationName": "Microsoft.Storage/storageAccounts/write",
    "status": "Succeeded",
    "subscriptionId": "abc123",
    "tenantId": "tenant-id"
  },
  "dataVersion": "2.0",
  "metadataVersion": "1"
}
```

---

## Schema Comparison

### CloudEvents vs. Event Grid Schema

| Feature | CloudEvents | Event Grid Schema |
|---------|-------------|-------------------|
| **Standard** | CNCF open standard | Azure proprietary |
| **Interoperability** | Cross-platform | Azure-specific |
| **Required Fields** | 4 (specversion, type, source, id) | 6 (topic, subject, eventType, id, eventTime, data) |
| **Timestamp Field** | `time` (optional) | `eventTime` (required) |
| **Type Field** | `type` | `eventType` |
| **Source Field** | `source` | `topic` |
| **Data Schema** | `dataschema` | `dataVersion` |
| **Content Type** | `datacontenttype` | Inferred |
| **Recommendation** | ✅ Use for new apps | ⚠️ Legacy/compatibility |

### Field Mapping

| CloudEvents | Event Grid Schema | Notes |
|-------------|-------------------|-------|
| `specversion` | `metadataVersion` | Schema version |
| `type` | `eventType` | Event type identifier |
| `source` | `topic` | Event source |
| `id` | `id` | Unique identifier |
| `time` | `eventTime` | Timestamp |
| `subject` | `subject` | Event subject |
| `data` | `data` | Event payload |
| `datacontenttype` | N/A | Content type |
| `dataschema` | `dataVersion` | Data schema version |

---

## Event Size and Billing

### Size Limits

| Limit Type | Value | Notes |
|------------|-------|-------|
| **Maximum Event Size** | 1 MB | Entire event including headers |
| **Minimum Billing Unit** | 64 KB | Rounded up |
| **Batch Size** | 1 MB | Total for all events in batch |

### Billing Examples

```
Event Size    → Billing Units (64 KB) → Operations Billed
32 KB         → 1 unit (64 KB)        → 1 operation
64 KB         → 1 unit (64 KB)        → 1 operation
65 KB         → 2 units (128 KB)      → 2 operations
130 KB        → 3 units (192 KB)      → 3 operations
256 KB        → 4 units (256 KB)      → 4 operations
1 MB (max)    → 16 units (1024 KB)    → 16 operations
```

### Size Optimization Tips

1. **Minimize Data Payload**: Include only necessary information
2. **Use References**: Store large data elsewhere, pass references
3. **Compress Data**: Use compression for large payloads
4. **Batch Events**: Send multiple small events together

**Example - Inefficient (large payload):**
```json
{
  "type": "ImageUploaded",
  "data": {
    "imageBase64": "iVBORw0KGgoAAAANSUhEUgAA... (500 KB of base64 data)"
  }
}
```

**Example - Efficient (reference-based):**
```json
{
  "type": "ImageUploaded",
  "data": {
    "imageUrl": "https://mystorage.blob.core.windows.net/images/photo.jpg",
    "imageSizeBytes": 524288,
    "contentType": "image/jpeg"
  }
}
```

---

## Subject Pattern Best Practices

The `subject` field enables powerful filtering. Design subjects hierarchically for maximum flexibility.

### Hierarchical Subject Structure

```
Format: /category/subcategory/resource/action

Examples:
✅ /blobServices/default/containers/images/blobs/photo.jpg
✅ /orders/region/west/store/101/order/12345
✅ /users/domain/example.com/user/john.doe@example.com
✅ /buildings/building-a/floor-3/room-301/sensor/temp-001

❌ photo.jpg
❌ order-12345
❌ john-doe-user
```

### Filtering with Subjects

```bash
# Filter by subject prefix (all blobs in "images" container)
--subject-begins-with "/blobServices/default/containers/images/"

# Filter by subject suffix (only .jpg files)
--subject-ends-with ".jpg"

# Combine both
--subject-begins-with "/blobServices/default/containers/images/" \
--subject-ends-with ".jpg"
```

### Subject Design Patterns

**Pattern 1: Resource Hierarchy**
```
/resource-type/region/group/instance
Example: /databases/eastus/production/db-001
```

**Pattern 2: Namespace Hierarchy**
```
/namespace/entity/id/operation
Example: /myapp/orders/12345/created
```

**Pattern 3: Domain-Driven Design**
```
/bounded-context/aggregate/id
Example: /inventory/products/SKU-001
```

---

## Publishing Events with Different Schemas

### Publish CloudEvents (Azure CLI)

```bash
# Publish CloudEvents to custom topic
az eventgrid topic publish \
  --name myTopic \
  --resource-group myRG \
  --events '[
    {
      "specversion": "1.0",
      "type": "com.example.orders.OrderCreated",
      "source": "https://api.example.com/orders",
      "subject": "/orders/12345",
      "id": "event-001",
      "time": "2024-01-15T14:30:00Z",
      "datacontenttype": "application/json",
      "data": {
        "orderId": "12345",
        "amount": 99.99
      }
    }
  ]' \
  --schema cloudevents
```

### Publish Event Grid Schema (Azure CLI)

```bash
# Publish Event Grid schema events
az eventgrid topic publish \
  --name myTopic \
  --resource-group myRG \
  --events '[
    {
      "id": "event-001",
      "eventType": "MyApp.Orders.OrderCreated",
      "subject": "/orders/12345",
      "eventTime": "2024-01-15T14:30:00Z",
      "data": {
        "orderId": "12345",
        "amount": 99.99
      },
      "dataVersion": "1.0"
    }
  ]'
```

### Publish CloudEvents (C# SDK)

```csharp
using Azure;
using Azure.Messaging;
using Azure.Messaging.EventGrid;

var endpoint = new Uri("https://mytopic.eastus-1.eventgrid.azure.net/api/events");
var credential = new AzureKeyCredential(topicKey);
var client = new EventGridPublisherClient(endpoint, credential);

// Create CloudEvent
var cloudEvent = new CloudEvent(
    source: "https://api.example.com/orders",
    type: "com.example.orders.OrderCreated",
    jsonSerializableData: new
    {
        orderId = "12345",
        amount = 99.99
    })
{
    Id = "event-001",
    Subject = "/orders/12345",
    Time = DateTimeOffset.UtcNow
};

// Publish event
await client.SendEventAsync(cloudEvent);

// Publish multiple events
var events = new[]
{
    cloudEvent,
    new CloudEvent("https://api.example.com/orders", "com.example.orders.OrderCreated", 
        new { orderId = "12346", amount = 149.99 })
};
await client.SendEventsAsync(events);
```

### Publish CloudEvents (Python SDK)

```python
from azure.eventgrid import EventGridPublisherClient
from azure.core.credentials import AzureKeyCredential
from azure.core.messaging import CloudEvent
from datetime import datetime

endpoint = "https://mytopic.eastus-1.eventgrid.azure.net/api/events"
credential = AzureKeyCredential(topic_key)
client = EventGridPublisherClient(endpoint, credential)

# Create CloudEvent
cloud_event = CloudEvent(
    source="https://api.example.com/orders",
    type="com.example.orders.OrderCreated",
    data={
        "orderId": "12345",
        "amount": 99.99
    },
    subject="/orders/12345",
    time=datetime.utcnow()
)

# Publish event
client.send(cloud_event)

# Publish multiple events
events = [
    cloud_event,
    CloudEvent(
        source="https://api.example.com/orders",
        type="com.example.orders.OrderCreated",
        data={"orderId": "12346", "amount": 149.99}
    )
]
client.send(events)
```

---

## Schema Validation

### Validating CloudEvents

```python
from jsonschema import validate
import json

# CloudEvents schema
cloudevents_schema = {
    "type": "object",
    "required": ["specversion", "type", "source", "id"],
    "properties": {
        "specversion": {"type": "string", "const": "1.0"},
        "type": {"type": "string"},
        "source": {"type": "string"},
        "id": {"type": "string"},
        "time": {"type": "string", "format": "date-time"},
        "data": {"type": "object"}
    }
}

# Validate event
event = {
    "specversion": "1.0",
    "type": "com.example.someevent",
    "source": "/mycontext",
    "id": "A234-1234-1234",
    "data": {"key": "value"}
}

try:
    validate(instance=event, schema=cloudevents_schema)
    print("Event is valid")
except Exception as e:
    print(f"Event validation failed: {e}")
```

---

## Exam Tips for AZ-204

### Key Concepts to Remember

1. **CloudEvents is recommended** for new applications (v1.0 specification)
2. **Event Grid schema** supported for backward compatibility
3. **Maximum event size**: 1 MB
4. **Billing**: 64 KB increments
5. **Subject** field enables powerful filtering
6. **CloudEvents** has 4 required fields; **Event Grid** has 6
7. **datacontenttype** in CloudEvents; inferred in Event Grid schema

### Schema Selection Decision Tree

```
New Application?
├─ Yes → Use CloudEvents ✅
│
└─ No (Existing)
   ├─ Need cross-platform compatibility? → CloudEvents ✅
   ├─ Backward compatibility required? → Event Grid Schema
   └─ Migrating? → CloudEvents (recommended)
```

### Common Exam Scenarios

**Scenario 1**: Design event schema for multi-cloud application
- ✅ Use CloudEvents for interoperability
- ❌ Don't use Event Grid schema (Azure-specific)

**Scenario 2**: Minimize event costs
- ✅ Keep events under 64 KB when possible
- ✅ Use reference-based data (URLs, IDs)
- ❌ Don't embed large payloads

**Scenario 3**: Enable complex filtering
- ✅ Design hierarchical subject structure
- ✅ Use consistent naming conventions
- ❌ Don't use flat subject names

### Remember for Exam

- **CloudEvents required fields**: specversion, type, source, id
- **Event Grid required fields**: topic, subject, eventType, id, eventTime, data
- **Content-Type header**: `application/cloudevents+json` for CloudEvents
- **Maximum size**: 1 MB per event
- **Billing increment**: 64 KB
- **Subject filtering**: Use `subjectBeginsWith` and `subjectEndsWith`
- **dataVersion**: Track schema evolution in Event Grid
- **dataschema**: Define schema URI in CloudEvents

### Quick Comparison

| Question | CloudEvents | Event Grid |
|----------|-------------|------------|
| Industry standard? | ✅ Yes (CNCF) | ❌ Azure-only |
| Recommended? | ✅ Yes | ⚠️ Legacy |
| Required fields | 4 | 6 |
| Schema versioning | dataschema (URI) | dataVersion (string) |
| Content type | Explicit | Inferred |

---

## Summary

**Event Schemas:**
- **CloudEvents v1.0**: Industry-standard, recommended for new apps
- **Event Grid Schema**: Azure-native, backward compatibility

**Key Differences:**
- CloudEvents has **fewer required fields** (4 vs 6)
- CloudEvents is **cross-platform compatible**
- CloudEvents specifies **content type explicitly**

**Best Practices:**
- ✅ Use **CloudEvents** for new applications
- ✅ Design **hierarchical subjects** for filtering
- ✅ Keep events **under 64 KB** to minimize cost
- ✅ Use **references** instead of embedding large data
- ✅ Include **dataVersion** or **dataschema** for versioning

**Size Limits:**
- Maximum event size: **1 MB**
- Billing increments: **64 KB**
- Batch size: **1 MB total**