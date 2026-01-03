# Manage Container Properties and Metadata with .NET

## Key Concepts
- **System properties** - Read-only or settable properties maintained by Azure
- **User-defined metadata** - Custom name-value pairs for your purposes
- **Metadata naming** - Must follow C# identifier rules, case-insensitive
- **Metadata size** - Total maximum 8 KB for all pairs
- **HTTP headers** - Properties and metadata represented as standard HTTP headers

## Properties vs Metadata

### System Properties

**Characteristics**:
- Exist on every Blob Storage resource
- Some are read-only, others can be set
- Correspond to standard HTTP headers
- Maintained automatically by Azure Storage
- No custom naming

**Common system properties**:

| Property | Access | Description |
|----------|--------|-------------|
| **ETag** | Read-only | Entity tag for optimistic concurrency |
| **Last-Modified** | Read-only | Last modification timestamp |
| **Content-Length** | Read-only | Size of blob in bytes |
| **Content-Type** | Read/Write | MIME type of blob content |
| **Content-Encoding** | Read/Write | Content encoding (gzip, deflate) |
| **Content-Language** | Read/Write | Language of content |
| **Cache-Control** | Read/Write | Caching directives |
| **PublicAccess** | Read/Write | Container access level |

### User-Defined Metadata

**Characteristics**:
- Custom name-value pairs
- Stored with the resource
- For your application's purposes only
- Does not affect resource behavior
- Must follow HTTP header restrictions
- Case-insensitive

**Restrictions**:
- ✅ Must be valid C# identifiers
- ✅ Valid HTTP header names
- ✅ ASCII characters only
- ✅ Maximum 8 KB total size
- ⚠️ Non-ASCII values must be Base64 or URL-encoded
- ⚠️ Names are case-insensitive but preserve case

## Retrieve Container Properties

### Using GetPropertiesAsync

**Method signatures**:
- `GetProperties()` - Synchronous
- `GetPropertiesAsync()` - Asynchronous (recommended)

**Complete example**:

```csharp
using Azure.Storage.Blobs;
using Azure;

private static async Task ReadContainerPropertiesAsync(BlobContainerClient container)
{
    try
    {
        // Fetch some container properties and write out their values
        var properties = await container.GetPropertiesAsync();
        
        Console.WriteLine($"Properties for container {container.Uri}");
        Console.WriteLine($"Public access level: {properties.Value.PublicAccess}");
        Console.WriteLine($"Last modified time in UTC: {properties.Value.LastModified}");
        Console.WriteLine($"ETag: {properties.Value.ETag}");
        Console.WriteLine($"Has immutability policy: {properties.Value.HasImmutabilityPolicy}");
        Console.WriteLine($"Has legal hold: {properties.Value.HasLegalHold}");
        Console.WriteLine($"Lease status: {properties.Value.LeaseStatus}");
        Console.WriteLine($"Lease state: {properties.Value.LeaseState}");
    }
    catch (RequestFailedException e)
    {
        Console.WriteLine($"HTTP error code {e.Status}: {e.ErrorCode}");
        Console.WriteLine(e.Message);
        Console.ReadLine();
    }
}
```

**Usage**:

```csharp
var containerClient = blobServiceClient.GetBlobContainerClient("mycontainer");
await ReadContainerPropertiesAsync(containerClient);
```

### Available Properties

**BlobContainerProperties object**:

```csharp
var properties = await container.GetPropertiesAsync();
var props = properties.Value;

// Access various properties
DateTimeOffset? lastModified = props.LastModified;
ETag? etag = props.ETag;
PublicAccessType? accessLevel = props.PublicAccess;
LeaseDurationType? leaseDuration = props.LeaseDuration;
LeaseState? leaseState = props.LeaseState;
LeaseStatus? leaseStatus = props.LeaseStatus;
bool? hasImmutabilityPolicy = props.HasImmutabilityPolicy;
bool? hasLegalHold = props.HasLegalHold;
string defaultEncryptionScope = props.DefaultEncryptionScope;
IDictionary<string, string> metadata = props.Metadata;
```

## Set and Retrieve Metadata

### Metadata Structure

**Name-value pairs**:
- **Name** - Must be valid C# identifier
- **Value** - String value
- **Storage** - Stored as HTTP headers with `x-ms-meta-` prefix

**Example metadata**:
```
x-ms-meta-docType: textDocuments
x-ms-meta-category: guidance
x-ms-meta-author: john-doe
x-ms-meta-version: 1.0
```

### Set Metadata

**Method signatures**:
- `SetMetadata(IDictionary<string, string>)` - Synchronous
- `SetMetadataAsync(IDictionary<string, string>)` - Asynchronous

**Complete example**:

```csharp
using System.Collections.Generic;
using Azure;

public static async Task AddContainerMetadataAsync(BlobContainerClient container)
{
    try
    {
        // Create metadata dictionary
        IDictionary<string, string> metadata = new Dictionary<string, string>();

        // Add some metadata to the container
        metadata.Add("docType", "textDocuments");
        metadata.Add("category", "guidance");
        metadata.Add("author", "john-doe");
        metadata.Add("version", "1.0");
        metadata.Add("environment", "production");

        // Set the container's metadata
        await container.SetMetadataAsync(metadata);
        
        Console.WriteLine("Metadata set successfully");
    }
    catch (RequestFailedException e)
    {
        Console.WriteLine($"HTTP error code {e.Status}: {e.ErrorCode}");
        Console.WriteLine(e.Message);
        Console.ReadLine();
    }
}
```

**Usage**:

```csharp
var containerClient = blobServiceClient.GetBlobContainerClient("mycontainer");
await AddContainerMetadataAsync(containerClient);
```

### Update Metadata

**Overwrite existing metadata**:

```csharp
public static async Task UpdateContainerMetadataAsync(BlobContainerClient container)
{
    try
    {
        // Get existing metadata first (optional)
        var properties = await container.GetPropertiesAsync();
        var existingMetadata = properties.Value.Metadata;

        // Create new metadata dictionary
        IDictionary<string, string> metadata = new Dictionary<string, string>(existingMetadata);

        // Update or add values
        metadata["version"] = "2.0";  // Update existing
        metadata["lastUpdated"] = DateTime.UtcNow.ToString("o");  // Add new

        // Set metadata (overwrites all)
        await container.SetMetadataAsync(metadata);
        
        Console.WriteLine("Metadata updated successfully");
    }
    catch (RequestFailedException e)
    {
        Console.WriteLine($"Error updating metadata: {e.Message}");
    }
}
```

⚠️ **Important**: `SetMetadataAsync()` **overwrites** all existing metadata. To preserve existing values, retrieve them first.

### Retrieve Metadata

**GetPropertiesAsync returns properties AND metadata**:

```csharp
using System.Collections.Generic;

public static async Task ReadContainerMetadataAsync(BlobContainerClient container)
{
    try
    {
        // Get properties which includes metadata
        var properties = await container.GetPropertiesAsync();

        // Enumerate the container's metadata
        Console.WriteLine("Container metadata:");
        foreach (var metadataItem in properties.Value.Metadata)
        {
            Console.WriteLine($"\tKey: {metadataItem.Key}");
            Console.WriteLine($"\tValue: {metadataItem.Value}");
        }
        
        // Check if specific metadata exists
        if (properties.Value.Metadata.TryGetValue("docType", out string docType))
        {
            Console.WriteLine($"\nDocument type: {docType}");
        }
    }
    catch (RequestFailedException e)
    {
        Console.WriteLine($"HTTP error code {e.Status}: {e.ErrorCode}");
        Console.WriteLine(e.Message);
        Console.ReadLine();
    }
}
```

**Output example**:
```
Container metadata:
    Key: docType
    Value: textDocuments
    Key: category
    Value: guidance
    Key: author
    Value: john-doe
    
Document type: textDocuments
```

### Clear Metadata

**Set empty dictionary**:

```csharp
public static async Task ClearContainerMetadataAsync(BlobContainerClient container)
{
    try
    {
        // Set empty metadata dictionary to clear all
        await container.SetMetadataAsync(new Dictionary<string, string>());
        
        Console.WriteLine("All metadata cleared");
    }
    catch (RequestFailedException e)
    {
        Console.WriteLine($"Error clearing metadata: {e.Message}");
    }
}
```

## Blob Properties and Metadata

### Retrieve Blob Properties

**Same pattern as containers**:

```csharp
public static async Task ReadBlobPropertiesAsync(BlobClient blob)
{
    try
    {
        var properties = await blob.GetPropertiesAsync();
        var props = properties.Value;

        Console.WriteLine($"Blob properties for {blob.Name}:");
        Console.WriteLine($"Content type: {props.ContentType}");
        Console.WriteLine($"Content length: {props.ContentLength} bytes");
        Console.WriteLine($"Content encoding: {props.ContentEncoding}");
        Console.WriteLine($"Last modified: {props.LastModified}");
        Console.WriteLine($"ETag: {props.ETag}");
        Console.WriteLine($"Blob type: {props.BlobType}");
        Console.WriteLine($"Access tier: {props.AccessTier}");
        Console.WriteLine($"Created on: {props.CreatedOn}");
    }
    catch (RequestFailedException e)
    {
        Console.WriteLine($"Error: {e.Status} - {e.ErrorCode}");
    }
}
```

### Set Blob Metadata

**Same methods as containers**:

```csharp
public static async Task SetBlobMetadataAsync(BlobClient blob)
{
    try
    {
        var metadata = new Dictionary<string, string>
        {
            { "documentType", "invoice" },
            { "fiscalYear", "2024" },
            { "department", "finance" },
            { "processed", "true" }
        };

        await blob.SetMetadataAsync(metadata);
        Console.WriteLine($"Metadata set for blob: {blob.Name}");
    }
    catch (RequestFailedException e)
    {
        Console.WriteLine($"Error: {e.Message}");
    }
}
```

### Set Blob HTTP Headers

**Configure Content-Type, Content-Encoding, etc.**:

```csharp
using Azure.Storage.Blobs.Models;

public static async Task SetBlobHttpHeadersAsync(BlobClient blob)
{
    try
    {
        var headers = new BlobHttpHeaders
        {
            ContentType = "application/pdf",
            ContentEncoding = "gzip",
            ContentLanguage = "en-US",
            CacheControl = "public, max-age=3600",
            ContentDisposition = "attachment; filename=document.pdf"
        };

        await blob.SetHttpHeadersAsync(headers);
        Console.WriteLine("HTTP headers set successfully");
    }
    catch (RequestFailedException e)
    {
        Console.WriteLine($"Error: {e.Message}");
    }
}
```

## Complete Working Example

### Container Metadata Manager

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;
using Azure;
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

public class ContainerMetadataManager
{
    private readonly BlobContainerClient _containerClient;

    public ContainerMetadataManager(BlobServiceClient serviceClient, string containerName)
    {
        _containerClient = serviceClient.GetBlobContainerClient(containerName);
    }

    // Create container with initial metadata
    public async Task CreateContainerWithMetadataAsync()
    {
        try
        {
            var metadata = new Dictionary<string, string>
            {
                { "environment", "production" },
                { "createdBy", "app-service" },
                { "purpose", "document-storage" }
            };

            var options = new BlobContainerCreateOptions
            {
                Metadata = metadata,
                PublicAccessType = PublicAccessType.None
            };

            await _containerClient.CreateIfNotExistsAsync(
                publicAccessType: PublicAccessType.None,
                metadata: metadata
            );

            Console.WriteLine($"Container created: {_containerClient.Name}");
        }
        catch (RequestFailedException e)
        {
            Console.WriteLine($"Error creating container: {e.Message}");
        }
    }

    // Display all properties and metadata
    public async Task DisplayContainerInfoAsync()
    {
        try
        {
            var properties = await _containerClient.GetPropertiesAsync();
            var props = properties.Value;

            Console.WriteLine($"\n=== Container: {_containerClient.Name} ===");
            Console.WriteLine($"URI: {_containerClient.Uri}");
            Console.WriteLine($"\nSystem Properties:");
            Console.WriteLine($"  Last Modified: {props.LastModified}");
            Console.WriteLine($"  ETag: {props.ETag}");
            Console.WriteLine($"  Public Access: {props.PublicAccess}");
            Console.WriteLine($"  Lease Status: {props.LeaseStatus}");

            Console.WriteLine($"\nMetadata ({props.Metadata.Count} items):");
            foreach (var item in props.Metadata)
            {
                Console.WriteLine($"  {item.Key} = {item.Value}");
            }
        }
        catch (RequestFailedException e)
        {
            Console.WriteLine($"Error: {e.Status} - {e.ErrorCode}");
        }
    }

    // Update specific metadata value
    public async Task UpdateMetadataValueAsync(string key, string value)
    {
        try
        {
            // Get existing metadata
            var properties = await _containerClient.GetPropertiesAsync();
            var metadata = new Dictionary<string, string>(properties.Value.Metadata);

            // Update or add value
            metadata[key] = value;

            // Set updated metadata
            await _containerClient.SetMetadataAsync(metadata);
            Console.WriteLine($"Metadata updated: {key} = {value}");
        }
        catch (RequestFailedException e)
        {
            Console.WriteLine($"Error: {e.Message}");
        }
    }

    // Remove specific metadata key
    public async Task RemoveMetadataKeyAsync(string key)
    {
        try
        {
            var properties = await _containerClient.GetPropertiesAsync();
            var metadata = new Dictionary<string, string>(properties.Value.Metadata);

            if (metadata.Remove(key))
            {
                await _containerClient.SetMetadataAsync(metadata);
                Console.WriteLine($"Metadata removed: {key}");
            }
            else
            {
                Console.WriteLine($"Key not found: {key}");
            }
        }
        catch (RequestFailedException e)
        {
            Console.WriteLine($"Error: {e.Message}");
        }
    }

    // Check if metadata key exists
    public async Task<bool> MetadataExistsAsync(string key)
    {
        try
        {
            var properties = await _containerClient.GetPropertiesAsync();
            return properties.Value.Metadata.ContainsKey(key);
        }
        catch (RequestFailedException)
        {
            return false;
        }
    }
}

// Usage example
public class Program
{
    public static async Task Main(string[] args)
    {
        var serviceClient = new BlobServiceClient(
            new Uri("https://myaccount.blob.core.windows.net"),
            new DefaultAzureCredential()
        );

        var manager = new ContainerMetadataManager(serviceClient, "documents");

        // Create container with metadata
        await manager.CreateContainerWithMetadataAsync();

        // Display info
        await manager.DisplayContainerInfoAsync();

        // Update metadata
        await manager.UpdateMetadataValueAsync("version", "2.0");
        await manager.UpdateMetadataValueAsync("lastModified", DateTime.UtcNow.ToString("o"));

        // Display updated info
        await manager.DisplayContainerInfoAsync();

        // Check if key exists
        bool hasVersion = await manager.MetadataExistsAsync("version");
        Console.WriteLine($"\nHas version metadata: {hasVersion}");

        // Remove metadata key
        await manager.RemoveMetadataKeyAsync("purpose");

        // Final display
        await manager.DisplayContainerInfoAsync();
    }
}
```

## Metadata Naming Best Practices

### Valid Names

```csharp
// ✅ Good: Valid C# identifiers
metadata.Add("docType", "invoice");
metadata.Add("fiscalYear", "2024");
metadata.Add("isProcessed", "true");
metadata.Add("createdBy", "john-doe");
metadata.Add("version_number", "1.0");

// ❌ Bad: Invalid names
metadata.Add("doc-type", "invoice");  // Hyphens not allowed
metadata.Add("fiscal year", "2024");   // Spaces not allowed
metadata.Add("is.processed", "true");  // Dots not allowed
metadata.Add("123version", "1.0");     // Cannot start with number
```

### Case Sensitivity

```csharp
// Metadata names are case-insensitive but preserve case
metadata.Add("DocumentType", "invoice");

// Later retrieval (any case works)
metadata.TryGetValue("documenttype", out string value);  // Works
metadata.TryGetValue("DocumentType", out string value);  // Works
metadata.TryGetValue("DOCUMENTTYPE", out string value);  // Works

// But the stored name preserves original case
// Stored as: "DocumentType"
```

### Value Encoding

**ASCII values** (no encoding needed):

```csharp
metadata.Add("author", "John Smith");
metadata.Add("department", "Engineering");
```

**Non-ASCII values** (must encode):

```csharp
// ❌ Bad: Non-ASCII characters
metadata.Add("author", "José García");  // May cause issues

// ✅ Good: Base64 encode
byte[] bytes = Encoding.UTF8.GetBytes("José García");
string encoded = Convert.ToBase64String(bytes);
metadata.Add("author", encoded);

// Later decode
byte[] decodedBytes = Convert.FromBase64String(metadata["author"]);
string author = Encoding.UTF8.GetString(decodedBytes);
```

## Critical Notes
- 💡 **Two types** - System properties (Azure-maintained) vs user-defined metadata (custom)
- 🎯 **GetPropertiesAsync** - Retrieves both properties and metadata
- ✅ **SetMetadataAsync** - Overwrites all metadata, not just updates
- ⚠️ **Metadata naming** - Must be valid C# identifiers, case-insensitive
- 🔒 **Size limit** - Maximum 8 KB for all metadata pairs combined
- 📊 **HTTP headers** - Metadata stored as x-ms-meta-name headers
- 💡 **Non-ASCII** - Must Base64 or URL-encode non-ASCII values
- ✅ **Case preservation** - Names preserve case but comparisons are case-insensitive
- ⚠️ **Duplicate names** - Multiple headers with same name return 400 Bad Request
- 🔄 **Update pattern** - Retrieve existing, modify dictionary, set all

## Exam Tips
- System properties: Read-only (ETag, Last-Modified, Content-Length) or settable (Content-Type, Cache-Control)
- User metadata: Custom name-value pairs, max 8 KB total
- GetPropertiesAsync: Retrieves both system properties and user-defined metadata
- SetMetadataAsync: Overwrites ALL metadata (not partial update)
- Metadata naming: Must be valid C# identifiers (letters, numbers, underscores)
- Case sensitivity: Names are case-insensitive but preserve original case
- HTTP representation: Metadata stored as x-ms-meta-name headers
- Metadata dictionary: IDictionary<string, string>
- Clear metadata: Call SetMetadataAsync with empty dictionary
- Blob metadata: Same methods as containers (GetPropertiesAsync, SetMetadataAsync)
- BlobHttpHeaders: Set Content-Type, Content-Encoding, Cache-Control, etc.
- Error handling: Catch RequestFailedException, check Status and ErrorCode
- Update pattern: Get existing metadata first, modify, then set to preserve other values
- Non-ASCII values: Must Base64-encode or URL-encode
- Maximum metadata: 8 KB total for all name-value pairs

[Learn More](https://learn.microsoft.com/en-us/training/modules/work-azure-blob-storage/5-manage-container-properties-metadata-dotnet)
