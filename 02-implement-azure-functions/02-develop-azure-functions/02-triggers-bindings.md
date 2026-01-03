# Triggers and Bindings

## Key Concepts
- **Trigger** - Event that starts function execution (exactly one per function)
- **Binding** - Declarative connection to data source/destination
- **Input binding** - Read data into function
- **Output binding** - Write data from function
- **Direction** - `in`, `out`, or `inout`

## Triggers

### Definition
**Event that invokes a function** - Each function must have exactly ONE trigger

### Trigger Properties
- **Type** - What kind of event (http, timer, queue, etc.)
- **Data** - Often provided as function parameter payload
- **Direction** - Always `in`

### Common Triggers
| Trigger | Event | Use Case |
|---------|-------|----------|
| **HTTP** | HTTP request | REST APIs, webhooks |
| **Timer** | Schedule (cron) | Scheduled jobs, cleanup |
| **Queue** | Message in queue | Async processing |
| **Blob** | File added/modified | File processing |
| **Event Hub** | Stream events | IoT, telemetry |
| **Event Grid** | Event notification | Resource changes |
| **Service Bus** | Enterprise message | Reliable messaging |
| **Cosmos DB** | Document change | Data synchronization |

## Bindings

### Definition
**Declarative way** to connect to external data sources/destinations

### Binding Types
1. **Input binding** - Bring data INTO function
2. **Output binding** - Send data OUT of function
3. **Bidirectional (inout)** - Read and write

### Benefits
✅ **No hardcoded connections** - Configuration, not code
✅ **Simplified code** - Framework handles connection logic
✅ **Testability** - Easy to mock bindings
✅ **Reusability** - Same binding in multiple functions

### Binding Properties
- **Type** - Data source (blob, table, queue, etc.)
- **Direction** - `in`, `out`, or `inout`
- **Name** - Parameter name in function
- **Connection** - App setting with connection string

## Configuration Methods

### Method 1: function.json (JavaScript/Python/PowerShell)
```json
{
  "disabled": false,
  "bindings": [
    {
      "type": "queueTrigger",
      "direction": "in",
      "name": "myQueueItem",
      "queueName": "myqueue-items",
      "connection": "MyStorageConnectionAppSetting"
    },
    {
      "type": "table",
      "direction": "out",
      "name": "tableBinding",
      "tableName": "Person",
      "connection": "MyStorageConnectionAppSetting"
    }
  ]
}
```

**Properties**:
- `type` - Trigger/binding type
- `direction` - Data flow direction
- `name` - Function parameter name
- `queueName` / `tableName` - Resource name
- `connection` - App setting name (NOT connection string itself)

### Method 2: Attributes (C#)
```csharp
[FunctionName("QueueTriggerTableOutput")]
[return: Table("outTable", Connection = "MY_TABLE_STORAGE_ACCT_APP_SETTING")]
public static Person Run(
    [QueueTrigger("myqueue-items", Connection = "MY_STORAGE_ACCT_APP_SETTING")] JObject order,
    ILogger log)
{
    return new Person() {
        PartitionKey = "Orders",
        RowKey = Guid.NewGuid().ToString(),
        Name = order["Name"].ToString(),
        MobileNumber = order["MobileNumber"].ToString()
    };
}
```

**Attributes provide**:
- Trigger definition (`QueueTrigger`)
- Input/output bindings (`Table`)
- Connection settings
- Resource names

### Method 3: Annotations (Java)
```java
@FunctionName("QueueTriggerTableOutput")
@TableOutput(name = "tableBinding", tableName = "Person", connection = "MyStorageConnection")
public Person run(
    @QueueTrigger(name = "myQueueItem", queueName = "myqueue-items", connection = "MyStorageConnection") String order,
    final ExecutionContext context) {
    
    Person person = new Person();
    person.setPartitionKey("Orders");
    person.setRowKey(UUID.randomUUID().toString());
    return person;
}
```

## Binding Direction

### Trigger
- **Direction**: Always `in`
- **Count**: Exactly one per function
- **Purpose**: Start function execution

### Input Binding
- **Direction**: `in`
- **Count**: Zero or more
- **Purpose**: Read data into function

### Output Binding
- **Direction**: `out`
- **Count**: Zero or more
- **Purpose**: Write data from function

### Bidirectional Binding
- **Direction**: `inout`
- **Count**: Zero or more
- **Purpose**: Read and write same resource
- **Portal**: Requires Advanced editor

### Direction Summary
| Binding | Direction | Count | Example |
|---------|-----------|-------|---------|
| **Trigger** | `in` | 1 (required) | Queue message arrives |
| **Input** | `in` | 0+ | Read blob, query table |
| **Output** | `out` | 0+ | Write blob, insert table row |
| **Bidirectional** | `inout` | 0+ | Update document |

## Complete Example: Queue → Table

### Scenario
New message in queue → Write row to table

### function.json (JavaScript)
```json
{
  "disabled": false,
  "bindings": [
    {
      "type": "queueTrigger",
      "direction": "in",
      "name": "myQueueItem",
      "queueName": "myqueue-items",
      "connection": "MyStorageConnectionAppSetting"
    },
    {
      "type": "table",
      "direction": "out",
      "name": "tableBinding",
      "tableName": "Person",
      "connection": "MyStorageConnectionAppSetting"
    }
  ]
}
```

### index.js
```javascript
module.exports = async function (context, myQueueItem) {
    context.log('Processing queue message:', myQueueItem);
    
    // Output to table via binding
    context.bindings.tableBinding = {
        PartitionKey: "Orders",
        RowKey: context.bindingData.id,
        Name: myQueueItem.name,
        MobileNumber: myQueueItem.mobile
    };
};
```

### C# Equivalent
```csharp
[FunctionName("QueueTriggerTableOutput")]
[return: Table("Person", Connection = "MyStorageConnectionAppSetting")]
public static Person Run(
    [QueueTrigger("myqueue-items", Connection = "MyStorageConnectionAppSetting")] Order order,
    ILogger log)
{
    log.LogInformation($"Processing order: {order.Name}");
    
    return new Person
    {
        PartitionKey = "Orders",
        RowKey = Guid.NewGuid().ToString(),
        Name = order.Name,
        MobileNumber = order.MobileNumber
    };
}
```

## Data Types

### JavaScript/Python (Dynamically Typed)
Use `dataType` property in function.json:

```json
{
    "type": "httpTrigger",
    "name": "req",
    "direction": "in",
    "dataType": "binary"
}
```

**Options**:
- `binary` - Byte array
- `stream` - Stream data
- `string` - String (default)

### C#/.NET (Statically Typed)
Type inferred from parameter:

```csharp
// String
[HttpTrigger] string req

// Binary
[HttpTrigger] byte[] req

// Stream
[HttpTrigger] Stream req

// Object (JSON deserialization)
[HttpTrigger] MyCustomType req
```

## Multiple Bindings Example

### Scenario
HTTP trigger → Read from Blob → Write to Queue and Table

### function.json
```json
{
  "bindings": [
    {
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": ["post"]
    },
    {
      "type": "blob",
      "direction": "in",
      "name": "inputBlob",
      "path": "input/{filename}",
      "connection": "AzureWebJobsStorage"
    },
    {
      "type": "queue",
      "direction": "out",
      "name": "outputQueue",
      "queueName": "processed-items",
      "connection": "AzureWebJobsStorage"
    },
    {
      "type": "table",
      "direction": "out",
      "name": "outputTable",
      "tableName": "ProcessedFiles",
      "connection": "AzureWebJobsStorage"
    },
    {
      "type": "http",
      "direction": "out",
      "name": "res"
    }
  ]
}
```

### index.js
```javascript
module.exports = async function (context, req, inputBlob) {
    context.log('Processing file:', req.query.filename);
    
    // Read from blob (input binding)
    const fileContent = inputBlob.toString();
    
    // Write to queue (output binding)
    context.bindings.outputQueue = {
        filename: req.query.filename,
        processed: true
    };
    
    // Write to table (output binding)
    context.bindings.outputTable = {
        PartitionKey: "Files",
        RowKey: new Date().toISOString(),
        Filename: req.query.filename,
        Size: inputBlob.length
    };
    
    // HTTP response (output binding)
    context.res = {
        status: 200,
        body: "File processed successfully"
    };
};
```

## Common Binding Patterns

### Pattern 1: Trigger → Process → Output
```
Queue Trigger → Function Logic → Blob Output
Timer Trigger → Function Logic → Table Output
HTTP Trigger → Function Logic → Queue Output
```

### Pattern 2: Trigger → Read → Process → Write
```
HTTP Trigger → Blob Input → Process → Table Output
Queue Trigger → Cosmos Input → Process → Queue Output
```

### Pattern 3: Fan-Out
```
Single Trigger → Multiple Outputs
Queue message → Write to Blob + Table + Queue
```

### Pattern 4: Chain
```
HTTP → Queue (output)
Queue (trigger) → Process → Blob (output)
Blob (trigger) → Process → Table (output)
```

## Connection Configuration

### App Settings (Recommended)
```json
// local.settings.json
{
  "Values": {
    "MyStorageConnectionAppSetting": "DefaultEndpointsProtocol=https;AccountName=mystorage;..."
  }
}
```

### Reference in Binding
```json
{
  "type": "queueTrigger",
  "connection": "MyStorageConnectionAppSetting"
}
```

⚠️ **Security**: Never hardcode connection strings in code

### Azure App Settings
```bash
# Set connection string
az functionapp config appsettings set \
  --name <function-app-name> \
  --resource-group <rg-name> \
  --settings "MyStorageConnectionAppSetting=DefaultEndpointsProtocol=https;..."
```

## Portal Binding Configuration

### For JavaScript/Python/PowerShell:
1. Navigate to function in portal
2. Click "Integration" tab
3. Click "+ Add input" or "+ Add output"
4. Select binding type
5. Configure properties
6. Save

### For C#:
❌ Cannot add bindings in portal - Use attributes in code

## Testing Bindings Locally

### Option 1: Live Azure Services
```json
// local.settings.json - Point to live Azure
{
  "Values": {
    "AzureWebJobsStorage": "DefaultEndpointsProtocol=https;AccountName=..."
  }
}
```

⚠️ **Caution**: Uses real data, can incur costs

### Option 2: Azurite Emulator
```bash
# Install and start Azurite
npm install -g azurite
azurite --silent --location c:\azurite

# In local.settings.json
"AzureWebJobsStorage": "UseDevelopmentStorage=true"
```

✅ **Recommended**: Safe, no costs, offline capable

### Option 3: Manual Admin Endpoint
```bash
# Trigger non-HTTP functions manually
curl -X POST http://localhost:7071/admin/functions/MyQueueFunction \
  -H "Content-Type: application/json" \
  -d '{"input": "test message"}'
```

## Binding Expressions

### Automatic Values
Use binding expressions in `path` or other properties:

```json
{
  "type": "blob",
  "direction": "out",
  "name": "outputBlob",
  "path": "output/{rand-guid}.txt",
  "connection": "AzureWebJobsStorage"
}
```

**Available expressions**:
- `{rand-guid}` - Random GUID
- `{datetime}` - Current timestamp
- `{name}` - From trigger data
- `{queueTrigger}` - Queue message content

### Example
```json
{
  "bindings": [
    {
      "type": "queueTrigger",
      "name": "order",
      "queueName": "orders"
    },
    {
      "type": "blob",
      "direction": "out",
      "name": "receipt",
      "path": "receipts/{id}-{datetime}.json"
    }
  ]
}
```

## Critical Notes
- 💡 **One trigger** - Exactly one per function (required)
- ⚠️ **Multiple bindings** - Zero or more input/output bindings
- 🎯 **Direction** - Trigger always `in`, bindings `in`/`out`/`inout`
- 📊 **Declarative** - Configuration, not code
- ✅ **Connection names** - Reference app settings, not connection strings
- 🔄 **C# uses attributes** - Other languages use function.json
- ⏱️ **Azurite for local** - Test storage bindings without Azure
- 🔒 **Never hardcode** - Use app settings for secrets

## Exam Tips
- Trigger = event that starts function (exactly ONE per function)
- Binding = connection to data source/destination (zero or MORE)
- Direction: Trigger always `in`, bindings `in`/`out`/`inout`
- C# uses attributes, JavaScript/Python/PowerShell use function.json
- Connection property references app setting NAME, not connection string
- Java uses annotations (@QueueTrigger, @TableOutput, etc.)
- dataType for dynamically typed languages (binary, stream, string)
- C# type inferred from parameter type
- Portal editing: Integration tab for function.json languages only
- Cannot configure C# bindings in portal (use attributes)
- Azurite emulator for local storage binding testing
- Admin endpoint for manual trigger: http://localhost:7071/admin/functions/{name}
- Binding expressions: {rand-guid}, {datetime}, {propertyName}

[Learn More](https://learn.microsoft.com/en-us/training/modules/develop-azure-functions/3-create-triggers-bindings)
