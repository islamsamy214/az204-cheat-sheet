# Azure Cosmos DB Stored Procedures

## Key Concepts
- **JavaScript execution** - Server-side code in JavaScript
- **Transactional** - ACID within single partition
- **Collection-scoped** - Registered per container
- **Bounded execution** - Time-limited operations

## What are Stored Procedures?

**Server-side JavaScript code** in Azure Cosmos DB:

- **Language**: JavaScript (not T-SQL)
- **Execution**: Server-side in database engine
- **Scope**: Container-level (operates on items in one container)
- **Transactions**: ACID guarantees within partition
- **Performance**: Reduced network roundtrips

### Benefits

✅ **Transactional** - Multiple operations in single transaction
✅ **Performance** - Server-side execution (no network latency)
✅ **Atomic** - All succeed or all fail
✅ **Consistent** - ACID guarantees
❌ **Partition-bound** - Cannot span partitions

## Writing Stored Procedures

### Basic Structure

```javascript
function storedProcedureName(param1, param2) {
    var context = getContext();
    var collection = context.getCollection();
    var response = context.getResponse();
    
    // Your logic here
    
    response.setBody("result");
}
```

### Hello World Example

```javascript
var helloWorldStoredProc = {
    id: "helloWorld",
    serverScript: function () {
        var context = getContext();
        var response = context.getResponse();
        
        response.setBody("Hello, World!");
    }
}
```

**Register and execute**:

```csharp
// Register
StoredProcedureProperties sproc = new StoredProcedureProperties
{
    Id = "helloWorld",
    Body = @"
        function() {
            var context = getContext();
            var response = context.getResponse();
            response.setBody('Hello, World!');
        }"
};

await container.Scripts.CreateStoredProcedureAsync(sproc);

// Execute
var result = await container.Scripts.ExecuteStoredProcedureAsync<string>(
    "helloWorld",
    new PartitionKey("partitionKeyValue"),
    Array.Empty<object>()  // No parameters
);

Console.WriteLine(result.Resource);  // "Hello, World!"
```

## Context Objects

### getContext()

**Main entry point** for stored procedure:

```javascript
var context = getContext();
var collection = context.getCollection();  // Container operations
var request = context.getRequest();        // Request data
var response = context.getResponse();      // Response to client
```

### Collection Object

**CRUD operations on items**:

| Method | Purpose |
|--------|---------|
| `createDocument()` | Insert item |
| `readDocument()` | Read single item |
| `replaceDocument()` | Update item |
| `deleteDocument()` | Delete item |
| `queryDocuments()` | Query items |
| `getSelfLink()` | Container self-link |

### Request Object

**Access request data**:

```javascript
var request = context.getRequest();
var body = request.getBody();  // Get request payload
request.setBody(newBody);      // Modify request
```

### Response Object

**Set response to client**:

```javascript
var response = context.getResponse();
response.setBody({ result: "success" });  // Return data
```

## Create Item Example

### Simple Create

```javascript
var createDocumentStoredProc = {
    id: "createMyDocument",
    body: function createMyDocument(documentToCreate) {
        var context = getContext();
        var collection = context.getCollection();
        
        // Async operation with callback
        var accepted = collection.createDocument(
            collection.getSelfLink(),
            documentToCreate,
            function (err, documentCreated) {
                if (err) throw new Error('Error: ' + err.message);
                
                // Return created document's ID
                context.getResponse().setBody(documentCreated.id);
            }
        );
        
        if (!accepted) return;  // Request not accepted
    }
};
```

**Execute from C#**:

```csharp
// Register stored procedure
var sproc = new StoredProcedureProperties
{
    Id = "createMyDocument",
    Body = File.ReadAllText("createDocument.js")
};
await container.Scripts.CreateStoredProcedureAsync(sproc);

// Execute with document payload
var newProduct = new
{
    id = "product-1",
    category = "electronics",
    name = "Laptop",
    price = 999.99
};

var result = await container.Scripts.ExecuteStoredProcedureAsync<string>(
    "createMyDocument",
    new PartitionKey("electronics"),
    new[] { newProduct }
);

Console.WriteLine($"Created document ID: {result.Resource}");
```

### Create with Validation

```javascript
function createDocumentWithValidation(document) {
    var context = getContext();
    var collection = context.getCollection();
    var response = context.getResponse();
    
    // Validate required fields
    if (!document.id) {
        throw new Error("Document must have an id");
    }
    if (!document.category) {
        throw new Error("Document must have a category");
    }
    
    // Add timestamp
    document.createdAt = new Date().toISOString();
    
    // Create document
    var accepted = collection.createDocument(
        collection.getSelfLink(),
        document,
        function (err, created) {
            if (err) throw new Error('Error: ' + err.message);
            response.setBody(created);
        }
    );
    
    if (!accepted) throw new Error("Request not accepted");
}
```

## Update Item Example

```javascript
function updateDocument(documentId, updates) {
    var context = getContext();
    var collection = context.getCollection();
    var response = context.getResponse();
    
    // Build query to find document
    var query = 'SELECT * FROM c WHERE c.id = "' + documentId + '"';
    
    var accepted = collection.queryDocuments(
        collection.getSelfLink(),
        query,
        function (err, documents) {
            if (err) throw new Error('Error: ' + err.message);
            
            if (documents.length === 0) {
                throw new Error('Document not found');
            }
            
            var doc = documents[0];
            
            // Apply updates
            for (var key in updates) {
                doc[key] = updates[key];
            }
            
            // Update timestamp
            doc.updatedAt = new Date().toISOString();
            
            // Replace document
            var replaceAccepted = collection.replaceDocument(
                doc._self,
                doc,
                function (err, replaced) {
                    if (err) throw new Error('Error: ' + err.message);
                    response.setBody(replaced);
                }
            );
            
            if (!replaceAccepted) throw new Error("Replace not accepted");
        }
    );
    
    if (!accepted) throw new Error("Query not accepted");
}
```

## Bulk Operations

### Bulk Insert Example

```javascript
function bulkInsert(documents) {
    var context = getContext();
    var collection = context.getCollection();
    var response = context.getResponse();
    
    var count = 0;
    var docsLength = documents.length;
    
    if (docsLength === 0) {
        response.setBody(0);
        return;
    }
    
    // Recursive function to insert documents
    tryCreate(documents[count], callback);
    
    function tryCreate(doc, callback) {
        var accepted = collection.createDocument(
            collection.getSelfLink(),
            doc,
            callback
        );
        
        // If request not accepted, we've hit capacity
        if (!accepted) {
            response.setBody(count);
        }
    }
    
    function callback(err, doc) {
        if (err) throw new Error('Error: ' + err.message);
        
        count++;
        
        if (count >= docsLength) {
            // All documents inserted
            response.setBody(count);
        } else {
            // Insert next document
            tryCreate(documents[count], callback);
        }
    }
}
```

**Execute from C#**:

```csharp
var documents = new[]
{
    new { id = "1", category = "electronics", name = "Laptop" },
    new { id = "2", category = "electronics", name = "Mouse" },
    new { id = "3", category = "electronics", name = "Keyboard" }
};

var result = await container.Scripts.ExecuteStoredProcedureAsync<int>(
    "bulkInsert",
    new PartitionKey("electronics"),
    new[] { documents }
);

Console.WriteLine($"Inserted {result.Resource} documents");
```

## Arrays as Input Parameters

### Parsing String Arrays

```javascript
function processArray(arr) {
    // Parameters always come as strings from portal
    if (typeof arr === "string") {
        arr = JSON.parse(arr);
    }
    
    arr.forEach(function(item) {
        // Process each item
        console.log(item);
    });
}
```

**Why this matters**:

```javascript
// From Azure Portal: Input sent as string
// Input: ["item1", "item2", "item3"]
// Actual parameter: "[\\"item1\\", \\"item2\\", \\"item3\\"]"

// From SDK: Input sent correctly
// Input: ["item1", "item2", "item3"]
// Actual parameter: ["item1", "item2", "item3"]

// Solution: Check and parse if string
function sample(arr) {
    if (typeof arr === "string") {
        arr = JSON.parse(arr);
    }
    // Now arr is always an array
}
```

## Bounded Execution

### Time Limits

**All operations must complete quickly**:

```javascript
function boundedOperation() {
    var context = getContext();
    var collection = context.getCollection();
    var response = context.getResponse();
    
    var query = 'SELECT * FROM c';
    var continuationToken = null;
    
    var accepted = collection.queryDocuments(
        collection.getSelfLink(),
        query,
        { continuation: continuationToken },
        function (err, documents, options) {
            if (err) throw new Error('Error: ' + err.message);
            
            // Process documents
            var count = documents.length;
            
            // Check if there's more data
            if (options.continuation) {
                // More data available, but we're out of time
                response.setBody({
                    count: count,
                    continuation: options.continuation
                });
            } else {
                // All done
                response.setBody({
                    count: count,
                    continuation: null
                });
            }
        }
    );
    
    // Boolean indicates if request was accepted
    if (!accepted) {
        // Request queue is full, try again later
        throw new Error("Request not accepted, try again");
    }
}
```

### Checking Acceptance

**All collection methods return boolean**:

```javascript
var accepted = collection.createDocument(...);
if (!accepted) {
    // Operation not queued (time/capacity limit)
    // Return partial results or continuation token
}
```

## Transactions

### Transaction Model

**ACID guarantees within partition**:

```javascript
function transferBalance(fromId, toId, amount) {
    var context = getContext();
    var collection = context.getCollection();
    var response = context.getResponse();
    
    // Read source account
    readDocument(fromId, function(fromAccount) {
        if (fromAccount.balance < amount) {
            throw new Error("Insufficient funds");
        }
        
        // Read destination account
        readDocument(toId, function(toAccount) {
            // Update balances
            fromAccount.balance -= amount;
            toAccount.balance += amount;
            
            // Write both updates
            updateDocument(fromAccount, function() {
                updateDocument(toAccount, function() {
                    response.setBody({
                        success: true,
                        from: fromAccount.id,
                        to: toAccount.id,
                        amount: amount
                    });
                });
            });
        });
    });
    
    function readDocument(id, callback) {
        var query = 'SELECT * FROM c WHERE c.id = "' + id + '"';
        collection.queryDocuments(
            collection.getSelfLink(),
            query,
            function (err, docs) {
                if (err || docs.length === 0) throw new Error("Document not found");
                callback(docs[0]);
            }
        );
    }
    
    function updateDocument(doc, callback) {
        collection.replaceDocument(
            doc._self,
            doc,
            function (err, replaced) {
                if (err) throw new Error("Update failed");
                callback();
            }
        );
    }
}
```

**Important transaction properties**:

✅ **Atomic** - All operations succeed or all fail
✅ **Isolated** - No partial visibility
✅ **Consistent** - Database remains consistent
✅ **Single partition** - Limited to one partition key
❌ **No cross-partition** - Cannot span partitions
❌ **Time-bound** - Must complete within timeout

### Continuation-Based Transactions

```javascript
function processLargeDataset(continuationToken) {
    var context = getContext();
    var collection = context.getCollection();
    var response = context.getResponse();
    
    var maxCount = 1000;
    var count = 0;
    
    var query = 'SELECT * FROM c';
    var requestOptions = { 
        continuation: continuationToken,
        pageSize: 100
    };
    
    var accepted = collection.queryDocuments(
        collection.getSelfLink(),
        query,
        requestOptions,
        function (err, documents, options) {
            if (err) throw new Error('Error: ' + err.message);
            
            // Process documents
            for (var i = 0; i < documents.length; i++) {
                // Do something with each document
                processDocument(documents[i]);
                count++;
                
                if (count >= maxCount) break;
            }
            
            // Return continuation for next batch
            response.setBody({
                processed: count,
                continuation: options.continuation
            });
        }
    );
    
    if (!accepted) {
        response.setBody({
            processed: count,
            continuation: continuationToken
        });
    }
    
    function processDocument(doc) {
        // Processing logic
        doc.processed = true;
        doc.processedAt = new Date().toISOString();
    }
}
```

**Use continuation token for large datasets**:

```csharp
string continuationToken = null;
int totalProcessed = 0;

do
{
    var result = await container.Scripts.ExecuteStoredProcedureAsync<dynamic>(
        "processLargeDataset",
        new PartitionKey("partitionKey"),
        new[] { continuationToken }
    );
    
    totalProcessed += (int)result.Resource.processed;
    continuationToken = result.Resource.continuation;
    
} while (continuationToken != null);

Console.WriteLine($"Processed {totalProcessed} documents");
```

## Registering Stored Procedures

### Register from C#

```csharp
var storedProcedure = new StoredProcedureProperties
{
    Id = "myStoredProc",
    Body = @"
        function(param1, param2) {
            var context = getContext();
            var response = context.getResponse();
            response.setBody('Result: ' + param1 + ', ' + param2);
        }"
};

StoredProcedureResponse response = await container.Scripts.CreateStoredProcedureAsync(
    storedProcedure
);

Console.WriteLine($"Created stored procedure: {response.Resource.Id}");
```

### Update Stored Procedure

```csharp
// Read existing
var existingSproc = await container.Scripts.ReadStoredProcedureAsync("myStoredProc");

// Update body
existingSproc.Resource.Body = @"
    function(param) {
        // Updated logic
    }";

// Replace
await container.Scripts.ReplaceStoredProcedureAsync(existingSproc.Resource);
```

### Delete Stored Procedure

```csharp
await container.Scripts.DeleteStoredProcedureAsync("myStoredProc");
```

### List Stored Procedures

```csharp
FeedIterator<StoredProcedureProperties> iterator = 
    container.Scripts.GetStoredProcedureQueryIterator<StoredProcedureProperties>();

while (iterator.HasMoreResults)
{
    FeedResponse<StoredProcedureProperties> sprocs = await iterator.ReadNextAsync();
    
    foreach (var sproc in sprocs)
    {
        Console.WriteLine($"Stored Procedure: {sproc.Id}");
    }
}
```

## Executing Stored Procedures

### Basic Execution

```csharp
// No parameters
var result = await container.Scripts.ExecuteStoredProcedureAsync<string>(
    "helloWorld",
    new PartitionKey("partitionKeyValue"),
    Array.Empty<object>()
);

// With parameters
var result2 = await container.Scripts.ExecuteStoredProcedureAsync<dynamic>(
    "createDocument",
    new PartitionKey("electronics"),
    new[] { new { id = "1", name = "Laptop" } }
);

Console.WriteLine($"Result: {result2.Resource}");
Console.WriteLine($"RU charge: {result2.RequestCharge}");
```

### With Multiple Parameters

```csharp
var result = await container.Scripts.ExecuteStoredProcedureAsync<int>(
    "bulkInsert",
    new PartitionKey("electronics"),
    new object[] 
    { 
        new[] 
        {
            new { id = "1", name = "Laptop" },
            new { id = "2", name = "Mouse" }
        }
    }
);

Console.WriteLine($"Inserted {result.Resource} documents");
```

## Best Practices

### 1. Partition Key Awareness

```javascript
// ✅ Good: All operations within same partition
function updateRelatedDocuments(orderId, status) {
    // All documents have same partition key (customerId)
    // Can update order and related items atomically
}

// ❌ Bad: Trying to access multiple partitions
function updateAcrossPartitions() {
    // Stored procedure can only access one partition
    // This will fail or only process one partition
}
```

### 2. Error Handling

```javascript
function robustOperation(data) {
    var context = getContext();
    var collection = context.getCollection();
    var response = context.getResponse();
    
    try {
        // Validate input
        if (!data || !data.id) {
            throw new Error("Invalid input");
        }
        
        var accepted = collection.createDocument(
            collection.getSelfLink(),
            data,
            function (err, created) {
                if (err) {
                    // Log error details
                    response.setBody({
                        success: false,
                        error: err.message
                    });
                    return;
                }
                
                response.setBody({
                    success: true,
                    id: created.id
                });
            }
        );
        
        if (!accepted) {
            response.setBody({
                success: false,
                error: "Request not accepted"
            });
        }
    } catch (error) {
        response.setBody({
            success: false,
            error: error.message
        });
    }
}
```

### 3. Use Continuation for Large Operations

```javascript
// Process in batches with continuation
function processInBatches(continuationToken) {
    // Return continuation token for client to call again
    // Prevents timeout on large datasets
}
```

### 4. Minimize Logic Complexity

```javascript
// ✅ Good: Simple, focused logic
function updatePrice(productId, newPrice) {
    // Single purpose, quick execution
}

// ❌ Bad: Complex business logic
function complexBusinessOperation() {
    // Multiple queries, complex calculations
    // Risk of timeout
}
```

## Critical Notes
- 💡 **JavaScript** - Server-side code written in JavaScript
- 🎯 **Transactional** - ACID guarantees within single partition
- ✅ **Container-scoped** - Registered per container
- ⚠️ **Partition-bound** - Cannot span multiple partitions
- 🔄 **Bounded execution** - Time-limited, return boolean acceptance
- 📊 **Context objects** - getContext() provides collection, request, response
- 💡 **Async callbacks** - All operations use callbacks
- ✅ **Arrays as strings** - Parse JSON if input from Azure Portal
- ⚠️ **Continuation** - Use for large datasets to avoid timeout
- 🔒 **Atomic** - All operations succeed or all fail within partition

## Exam Tips
- Stored procedures: Server-side JavaScript in Azure Cosmos DB
- Language: JavaScript (not T-SQL or other SQL variants)
- Scope: Container-level, registered per container
- Partition: All operations within single partition key value
- Transactions: ACID guarantees within partition, no cross-partition
- Context: getContext() returns context object
- Collection methods: createDocument, readDocument, replaceDocument, deleteDocument, queryDocuments
- Return value: response.setBody() to return data to client
- Boolean return: All collection methods return true (accepted) or false (not accepted)
- Bounded execution: Operations must complete within time limit
- Continuation: Use continuation token for large datasets
- Arrays: Parse JSON.parse() if string (from Azure Portal)
- Registration: Scripts.CreateStoredProcedureAsync()
- Execution: Scripts.ExecuteStoredProcedureAsync() with partition key
- Error handling: Throw errors, catch in client code
- Async: All operations use callbacks, not promises/async-await
- Parameters: Pass as array in ExecuteStoredProcedureAsync
- Benefits: Reduced network roundtrips, server-side execution, transactional
- Limitations: Cannot span partitions, time-bound execution
- Use cases: Bulk operations, atomic updates, server-side validation

[Learn More](https://learn.microsoft.com/en-us/training/modules/work-with-cosmos-db/4-cosmos-db-stored-procedures)
