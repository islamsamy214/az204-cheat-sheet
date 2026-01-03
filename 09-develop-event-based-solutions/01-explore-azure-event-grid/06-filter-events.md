# Filter Events in Azure Event Grid

## Overview

**Event filtering** allows you to control which events are delivered to each subscription endpoint, reducing unnecessary traffic and processing costs.

**Three Filtering Types:**
1. **Event Type Filtering** - Filter by event type
2. **Subject Filtering** - Filter by subject patterns
3. **Advanced Filtering** - Filter by event data fields

---

## Event Type Filtering

Filter events based on their `eventType` (Event Grid schema) or `type` (CloudEvents schema).

### Filter by Specific Event Types

**Azure CLI:**
```bash
# Subscribe to specific event types only
az eventgrid event-subscription create \
  --name mySubscription \
  --source-resource-id "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage" \
  --endpoint https://myfunction.azurewebsites.net/api/handler \
  --included-event-types Microsoft.Storage.BlobCreated Microsoft.Storage.BlobDeleted
```

**ARM Template:**
```json
{
  "type": "Microsoft.EventGrid/eventSubscriptions",
  "properties": {
    "destination": {
      "endpointType": "WebHook",
      "properties": {
        "endpointUrl": "https://myfunction.azurewebsites.net/api/handler"
      }
    },
    "filter": {
      "includedEventTypes": [
        "Microsoft.Storage.BlobCreated",
        "Microsoft.Storage.BlobDeleted"
      ]
    }
  }
}
```

### Subscribe to All Event Types

**Azure CLI:**
```bash
# Receive all event types (default behavior)
az eventgrid event-subscription create \
  --name mySubscription \
  --source-resource-id $STORAGE_ID \
  --endpoint $WEBHOOK_URL \
  --included-event-types All
```

**ARM Template:**
```json
{
  "filter": {
    "includedEventTypes": []
  }
}
```
*Empty array = all event types*

### Common Azure Event Types

| Azure Service | Event Types |
|--------------|-------------|
| **Blob Storage** | `Microsoft.Storage.BlobCreated`, `Microsoft.Storage.BlobDeleted`, `Microsoft.Storage.BlobTierChanged` |
| **Resource Manager** | `Microsoft.Resources.ResourceWriteSuccess`, `Microsoft.Resources.ResourceDeleteSuccess`, `Microsoft.Resources.ResourceActionSuccess` |
| **Event Hubs** | `Microsoft.EventHub.CaptureFileCreated` |
| **IoT Hub** | `Microsoft.Devices.DeviceCreated`, `Microsoft.Devices.DeviceDeleted`, `Microsoft.Devices.DeviceConnected`, `Microsoft.Devices.DeviceDisconnected` |
| **Container Registry** | `Microsoft.ContainerRegistry.ImagePushed`, `Microsoft.ContainerRegistry.ImageDeleted`, `Microsoft.ContainerRegistry.ChartPushed` |
| **Media Services** | `Microsoft.Media.JobStateChange`, `Microsoft.Media.JobOutputStateChange`, `Microsoft.Media.LiveEventEncoderConnected` |
| **Service Bus** | `Microsoft.ServiceBus.ActiveMessagesAvailableWithNoListeners`, `Microsoft.ServiceBus.DeadletterMessagesAvailableWithNoListener` |
| **App Configuration** | `Microsoft.AppConfiguration.KeyValueModified`, `Microsoft.AppConfiguration.KeyValueDeleted` |

### Custom Event Types

For custom topics, define your own event types:

```bash
az eventgrid event-subscription create \
  --name orderSubscription \
  --source-resource-id $CUSTOM_TOPIC_ID \
  --endpoint $WEBHOOK_URL \
  --included-event-types \
    MyApp.Orders.OrderCreated \
    MyApp.Orders.OrderShipped \
    MyApp.Orders.OrderCancelled
```

---

## Subject Filtering

Filter events based on the beginning or ending of the `subject` field.

### Subject Structure Best Practices

Design hierarchical subjects for flexible filtering:

```
Format: /category/subcategory/resource

Examples:
/blobServices/default/containers/images/blobs/photo.jpg
/blobServices/default/containers/documents/blobs/report.pdf
/orders/region/west/store/101/order/12345
/users/domain/example.com/user/john.doe
```

### Filter by Subject Prefix (subjectBeginsWith)

**Use Case:** Filter events from specific container or folder

```bash
# Filter: Only blobs from "images" container
az eventgrid event-subscription create \
  --name imageSubscription \
  --source-resource-id $STORAGE_ID \
  --endpoint $WEBHOOK_URL \
  --subject-begins-with "/blobServices/default/containers/images/"
```

**ARM Template:**
```json
{
  "filter": {
    "subjectBeginsWith": "/blobServices/default/containers/images/"
  }
}
```

**What matches:**
- ✅ `/blobServices/default/containers/images/blobs/photo1.jpg`
- ✅ `/blobServices/default/containers/images/blobs/subfolder/photo2.jpg`
- ❌ `/blobServices/default/containers/documents/blobs/doc.pdf`

### Filter by Subject Suffix (subjectEndsWith)

**Use Case:** Filter events for specific file types

```bash
# Filter: Only .jpg files
az eventgrid event-subscription create \
  --name jpgSubscription \
  --source-resource-id $STORAGE_ID \
  --endpoint $WEBHOOK_URL \
  --subject-ends-with ".jpg"
```

**ARM Template:**
```json
{
  "filter": {
    "subjectEndsWith": ".jpg"
  }
}
```

**What matches:**
- ✅ `/blobServices/default/containers/images/blobs/photo.jpg`
- ✅ `/blobServices/default/containers/archive/blobs/old-photo.jpg`
- ❌ `/blobServices/default/containers/images/blobs/photo.png`

### Combine Prefix and Suffix

**Use Case:** Filter .jpg files from specific container

```bash
# Filter: .jpg files from "images" container only
az eventgrid event-subscription create \
  --name imageJpgSubscription \
  --source-resource-id $STORAGE_ID \
  --endpoint $WEBHOOK_URL \
  --subject-begins-with "/blobServices/default/containers/images/" \
  --subject-ends-with ".jpg"
```

**ARM Template:**
```json
{
  "filter": {
    "subjectBeginsWith": "/blobServices/default/containers/images/",
    "subjectEndsWith": ".jpg"
  }
}
```

**What matches:**
- ✅ `/blobServices/default/containers/images/blobs/photo.jpg`
- ✅ `/blobServices/default/containers/images/blobs/vacation/beach.jpg`
- ❌ `/blobServices/default/containers/images/blobs/photo.png` (wrong extension)
- ❌ `/blobServices/default/containers/documents/blobs/scan.jpg` (wrong container)

### Subject Filtering Examples

**Example 1: Filter by Region**
```bash
# Orders from West region only
--subject-begins-with "/orders/region/west/"
```

**Example 2: Filter by Customer Domain**
```bash
# Users from example.com domain
--subject-begins-with "/users/domain/example.com/"
```

**Example 3: Filter by File Extension**
```bash
# PDF documents only
--subject-ends-with ".pdf"

# Images only (.jpg, .png, .gif)
# Note: Need separate subscriptions for each extension
--subject-ends-with ".jpg"  # Subscription 1
--subject-ends-with ".png"  # Subscription 2
--subject-ends-with ".gif"  # Subscription 3
```

**Example 4: Filter by Blob Container**
```bash
# Production container only
--subject-begins-with "/blobServices/default/containers/production/"

# Exclude system containers
--subject-begins-with "/blobServices/default/containers/" \
--subject-does-not-begin-with "/blobServices/default/containers/$"
```

---

## Advanced Filtering

Filter events based on **data field values** using comparison operators.

### Advanced Filter Operators

| Operator | Data Types | Description |
|----------|-----------|-------------|
| `NumberIn` | Number | Value in list |
| `NumberNotIn` | Number | Value not in list |
| `NumberLessThan` | Number | Value < specified |
| `NumberLessThanOrEquals` | Number | Value ≤ specified |
| `NumberGreaterThan` | Number | Value > specified |
| `NumberGreaterThanOrEquals` | Number | Value ≥ specified |
| `BoolEquals` | Boolean | Value equals true/false |
| `StringIn` | String | Value in list |
| `StringNotIn` | String | Value not in list |
| `StringBeginsWith` | String | Value starts with |
| `StringEndsWith` | String | Value ends with |
| `StringContains` | String | Value contains substring |
| `StringNotContains` | String | Value doesn't contain |
| `StringNotBeginsWith` | String | Value doesn't start with |
| `StringNotEndsWith` | String | Value doesn't end with |
| `IsNullOrUndefined` | Any | Field is null or undefined |
| `IsNotNull` | Any | Field is not null |

### Advanced Filter Limits

- **Maximum filters per subscription**: 25
- **Maximum values per array operator (In/NotIn)**: 25
- **Maximum characters per string value**: 512
- **Maximum key length**: 64 characters

### Number Filtering Examples

**Example 1: Filter by Blob Size**

```bash
# Only blobs larger than 1 MB (1,048,576 bytes)
az eventgrid event-subscription create \
  --name largeBlobSubscription \
  --source-resource-id $STORAGE_ID \
  --endpoint $WEBHOOK_URL \
  --included-event-types Microsoft.Storage.BlobCreated \
  --advanced-filter data.contentLength NumberGreaterThan 1048576
```

**ARM Template:**
```json
{
  "filter": {
    "advancedFilters": [
      {
        "operatorType": "NumberGreaterThan",
        "key": "data.contentLength",
        "value": 1048576
      }
    ]
  }
}
```

**Example 2: Filter by Range**

```bash
# Blobs between 100 KB and 10 MB
az eventgrid event-subscription create \
  --name mediumBlobSubscription \
  --source-resource-id $STORAGE_ID \
  --endpoint $WEBHOOK_URL \
  --advanced-filter data.contentLength NumberGreaterThanOrEquals 102400 \
  --advanced-filter data.contentLength NumberLessThanOrEquals 10485760
```

**ARM Template:**
```json
{
  "filter": {
    "advancedFilters": [
      {
        "operatorType": "NumberGreaterThanOrEquals",
        "key": "data.contentLength",
        "value": 102400
      },
      {
        "operatorType": "NumberLessThanOrEquals",
        "key": "data.contentLength",
        "value": 10485760
      }
    ]
  }
}
```

### String Filtering Examples

**Example 1: Filter by Blob Type**

```bash
# Only Block Blobs
az eventgrid event-subscription create \
  --name blockBlobSubscription \
  --source-resource-id $STORAGE_ID \
  --endpoint $WEBHOOK_URL \
  --advanced-filter data.blobType StringIn BlockBlob
```

**ARM Template:**
```json
{
  "filter": {
    "advancedFilters": [
      {
        "operatorType": "StringIn",
        "key": "data.blobType",
        "values": ["BlockBlob"]
      }
    ]
  }
}
```

**Example 2: Filter by Content Type**

```bash
# Only image files (MIME type starts with "image/")
az eventgrid event-subscription create \
  --name imageTypeSubscription \
  --source-resource-id $STORAGE_ID \
  --endpoint $WEBHOOK_URL \
  --advanced-filter data.contentType StringBeginsWith image/
```

**ARM Template:**
```json
{
  "filter": {
    "advancedFilters": [
      {
        "operatorType": "StringBeginsWith",
        "key": "data.contentType",
        "values": ["image/"]
      }
    ]
  }
}
```

**Example 3: Filter by Multiple Content Types**

```bash
# Images and videos only
az eventgrid event-subscription create \
  --name mediaSubscription \
  --source-resource-id $STORAGE_ID \
  --endpoint $WEBHOOK_URL \
  --advanced-filter data.contentType StringIn image/jpeg image/png video/mp4 video/mpeg
```

**ARM Template:**
```json
{
  "filter": {
    "advancedFilters": [
      {
        "operatorType": "StringIn",
        "key": "data.contentType",
        "values": ["image/jpeg", "image/png", "video/mp4", "video/mpeg"]
      }
    ]
  }
}
```

**Example 4: Exclude System Blobs**

```bash
# Exclude blobs starting with "."
az eventgrid event-subscription create \
  --name userBlobSubscription \
  --source-resource-id $STORAGE_ID \
  --endpoint $WEBHOOK_URL \
  --advanced-filter data.url StringNotContains "/.well-known/" \
  --advanced-filter data.url StringNotContains "/$logs/"
```

### Boolean Filtering Examples

**Example: Filter by Blob Tier**

```bash
# Only hot tier blobs
az eventgrid event-subscription create \
  --name hotTierSubscription \
  --source-resource-id $STORAGE_ID \
  --endpoint $WEBHOOK_URL \
  --advanced-filter data.accessTier StringIn Hot
```

### Null/Undefined Filtering

**Example: Filter Events with Missing Fields**

```bash
# Events where metadata is not set
az eventgrid event-subscription create \
  --name noMetadataSubscription \
  --source-resource-id $STORAGE_ID \
  --endpoint $WEBHOOK_URL \
  --advanced-filter data.metadata IsNullOrUndefined
```

### Custom Event Advanced Filtering

**Example: Order Processing**

```bash
# Orders over $1000 from premium customers
az eventgrid event-subscription create \
  --name premiumOrderSubscription \
  --source-resource-id $CUSTOM_TOPIC_ID \
  --endpoint $WEBHOOK_URL \
  --included-event-types MyApp.Orders.OrderCreated \
  --advanced-filter data.amount NumberGreaterThan 1000 \
  --advanced-filter data.customerTier StringIn Premium Gold
```

**ARM Template:**
```json
{
  "filter": {
    "includedEventTypes": ["MyApp.Orders.OrderCreated"],
    "advancedFilters": [
      {
        "operatorType": "NumberGreaterThan",
        "key": "data.amount",
        "value": 1000
      },
      {
        "operatorType": "StringIn",
        "key": "data.customerTier",
        "values": ["Premium", "Gold"]
      }
    ]
  }
}
```

---

## Combining Filter Types

Combine event type, subject, and advanced filters for precise event routing.

### Example 1: Image Processing Pipeline

**Requirement:** Process .jpg images over 500 KB from "uploads" container

```bash
az eventgrid event-subscription create \
  --name imageProcessingSubscription \
  --source-resource-id $STORAGE_ID \
  --endpoint $WEBHOOK_URL \
  --included-event-types Microsoft.Storage.BlobCreated \
  --subject-begins-with "/blobServices/default/containers/uploads/" \
  --subject-ends-with ".jpg" \
  --advanced-filter data.contentLength NumberGreaterThan 512000
```

**ARM Template:**
```json
{
  "filter": {
    "includedEventTypes": ["Microsoft.Storage.BlobCreated"],
    "subjectBeginsWith": "/blobServices/default/containers/uploads/",
    "subjectEndsWith": ".jpg",
    "advancedFilters": [
      {
        "operatorType": "NumberGreaterThan",
        "key": "data.contentLength",
        "value": 512000
      }
    ]
  }
}
```

### Example 2: Multi-Region Order Processing

**Requirement:** High-value orders from West or East regions

```bash
# West region orders over $5000
az eventgrid event-subscription create \
  --name westHighValueOrders \
  --source-resource-id $CUSTOM_TOPIC_ID \
  --endpoint $WEBHOOK_URL_WEST \
  --included-event-types MyApp.Orders.OrderCreated \
  --subject-begins-with "/orders/region/west/" \
  --advanced-filter data.amount NumberGreaterThan 5000

# East region orders over $5000
az eventgrid event-subscription create \
  --name eastHighValueOrders \
  --source-resource-id $CUSTOM_TOPIC_ID \
  --endpoint $WEBHOOK_URL_EAST \
  --included-event-types MyApp.Orders.OrderCreated \
  --subject-begins-with "/orders/region/east/" \
  --advanced-filter data.amount NumberGreaterThan 5000
```

### Example 3: Document Processing Workflow

**Requirement:** PDF and Word documents from specific folders

```bash
# Subscription 1: PDF documents
az eventgrid event-subscription create \
  --name pdfProcessing \
  --source-resource-id $STORAGE_ID \
  --endpoint $PDF_PROCESSOR_URL \
  --included-event-types Microsoft.Storage.BlobCreated \
  --subject-begins-with "/blobServices/default/containers/documents/" \
  --subject-ends-with ".pdf"

# Subscription 2: Word documents
az eventgrid event-subscription create \
  --name wordProcessing \
  --source-resource-id $STORAGE_ID \
  --endpoint $WORD_PROCESSOR_URL \
  --included-event-types Microsoft.Storage.BlobCreated \
  --subject-begins-with "/blobServices/default/containers/documents/" \
  --advanced-filter data.contentType StringIn \
    application/msword \
    application/vnd.openxmlformats-officedocument.wordprocessingml.document
```

---

## Filter Evaluation Order

Filters are evaluated in this order:

1. **Event Type Filter** - Fastest, checked first
2. **Subject Filter** - String prefix/suffix matching
3. **Advanced Filters** - Most expensive, evaluated last

**Optimization Tip:** Use event type and subject filters when possible for better performance.

---

## Testing Filters

### Test with Azure CLI

```bash
# Publish test event
az eventgrid event publish \
  --topic-name myTopic \
  --resource-group myRG \
  --events '[{
    "id": "test-001",
    "eventType": "Microsoft.Storage.BlobCreated",
    "subject": "/blobServices/default/containers/images/blobs/test.jpg",
    "eventTime": "2024-01-15T14:30:00Z",
    "data": {
      "api": "PutBlob",
      "contentType": "image/jpeg",
      "contentLength": 1048576,
      "blobType": "BlockBlob",
      "url": "https://mystorage.blob.core.windows.net/images/test.jpg"
    },
    "dataVersion": "1.0"
  }]'
```

### Test with PowerShell

```powershell
# Test event that should match
$event = @{
    id = "test-001"
    eventType = "Microsoft.Storage.BlobCreated"
    subject = "/blobServices/default/containers/images/blobs/test.jpg"
    eventTime = (Get-Date -Format o)
    data = @{
        contentType = "image/jpeg"
        contentLength = 2000000
        blobType = "BlockBlob"
    }
    dataVersion = "1.0"
}

Invoke-RestMethod -Method Post -Uri $topicEndpoint -Headers @{"aeg-sas-key"=$topicKey} -Body ($event | ConvertTo-Json)
```

### Check Subscription Metrics

```bash
# Check matched events
az monitor metrics list \
  --resource $TOPIC_ID \
  --metric "MatchedEventCount" \
  --dimension "EventSubscriptionName=mySubscription" \
  --start-time 2024-01-15T00:00:00Z

# Check delivered events
az monitor metrics list \
  --resource $TOPIC_ID \
  --metric "DeliverySuccessCount" \
  --dimension "EventSubscriptionName=mySubscription" \
  --start-time 2024-01-15T00:00:00Z
```

---

## Best Practices

### Filter Design

1. **Use event type filters first** - Most efficient
2. **Design hierarchical subjects** - Enable flexible prefix filtering
3. **Minimize advanced filters** - Most expensive to evaluate
4. **Avoid overlapping subscriptions** - Can cause duplicate processing
5. **Test filters thoroughly** - Verify with sample events

### Subject Patterns

```
✅ Good patterns (hierarchical):
/resource-type/region/group/instance
/orders/region/west/store/101/order/12345
/blobServices/default/containers/images/blobs/photo.jpg

❌ Bad patterns (flat):
order-12345
photo.jpg
west-store-101
```

### Performance Optimization

1. **Event Type**: Fastest filter
2. **Subject Prefix/Suffix**: Fast string matching
3. **Advanced Filters**: Slower, limit to ≤ 25 per subscription

### Multiple Subscriptions vs. Complex Filters

**Scenario:** Route events to different endpoints based on region

**Option 1: Multiple Subscriptions (Recommended)**
```bash
# Subscription 1: West region
az eventgrid event-subscription create \
  --name westSubscription \
  --endpoint $WEST_ENDPOINT \
  --subject-begins-with "/orders/region/west/"

# Subscription 2: East region
az eventgrid event-subscription create \
  --name eastSubscription \
  --endpoint $EAST_ENDPOINT \
  --subject-begins-with "/orders/region/east/"
```

**Option 2: Single Subscription with Complex Logic (Not Recommended)**
```bash
# Single endpoint processes all regions
az eventgrid event-subscription create \
  --name allRegionsSubscription \
  --endpoint $ROUTING_ENDPOINT
# Routing logic in handler code ❌
```

---

## Exam Tips for AZ-204

### Key Concepts to Remember

1. **Three filter types**: Event type, subject, advanced
2. **Subject filters**: `subjectBeginsWith`, `subjectEndsWith`
3. **Advanced filters**: 25 max per subscription
4. **Filter evaluation order**: Event type → subject → advanced
5. **Case sensitivity**: Subject filters are case-sensitive
6. **Advanced filter operators**: NumberGreaterThan, StringContains, BoolEquals, etc.

### Common Exam Scenarios

**Scenario 1**: Filter .jpg images from specific container
- ✅ Use `subjectBeginsWith` for container + `subjectEndsWith` for `.jpg`
- ❌ Don't use advanced filters (less efficient)

**Scenario 2**: Filter events by data field value
- ✅ Use advanced filters with appropriate operator
- ❌ Don't try to use subject filters for data fields

**Scenario 3**: Optimize performance
- ✅ Use event type filter first (fastest)
- ✅ Use subject filters over advanced filters
- ❌ Don't use 25 advanced filters if simpler options exist

**Scenario 4**: Multiple conditions
- ✅ Combine event type + subject + advanced filters
- ✅ All conditions must match (AND logic)

### Remember for Exam

- **Event type**: Most efficient filter
- **Subject**: Case-sensitive prefix/suffix matching
- **Advanced**: 25 filters max, most expensive
- **Operators**: NumberGreaterThan, StringContains, StringIn, BoolEquals, IsNullOrUndefined
- **Evaluation order**: Type → Subject → Advanced
- **All filters**: AND logic (all must match)
- **Empty event types**: Matches all events

### Quick Command Reference

```bash
# Event type filter
--included-event-types Microsoft.Storage.BlobCreated

# Subject filters
--subject-begins-with "/blobServices/default/containers/images/"
--subject-ends-with ".jpg"

# Advanced filter
--advanced-filter data.contentLength NumberGreaterThan 1048576
--advanced-filter data.blobType StringIn BlockBlob
--advanced-filter data.contentType StringBeginsWith image/
```

---

## Summary

**Filter Types:**
- **Event Type**: Filter by event type (fastest)
- **Subject**: Filter by subject prefix or suffix
- **Advanced**: Filter by data field values (most flexible)

**Subject Filtering:**
- `subjectBeginsWith`: Prefix matching (e.g., container path)
- `subjectEndsWith`: Suffix matching (e.g., file extension)
- Case-sensitive string matching

**Advanced Filtering:**
- **Operators**: Number (>, <, =), String (contains, begins/ends with), Boolean, Null
- **Limits**: 25 filters max per subscription
- **Data Fields**: Filter by any field in event data

**Best Practices:**
- ✅ Design hierarchical subjects for flexible filtering
- ✅ Use event type filters when possible (most efficient)
- ✅ Combine filters for precise routing
- ✅ Test filters with sample events
- ✅ Use multiple subscriptions for different endpoints
- ❌ Avoid complex filtering in handler code
- ❌ Don't exceed 25 advanced filters per subscription

**Common Patterns:**
- Images only: `--subject-ends-with .jpg`
- Specific container: `--subject-begins-with /blobServices/default/containers/images/`
- Large files: `--advanced-filter data.contentLength NumberGreaterThan 1048576`
- Specific content type: `--advanced-filter data.contentType StringIn image/jpeg`