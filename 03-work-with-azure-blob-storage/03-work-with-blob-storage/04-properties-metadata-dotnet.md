# Manage Container Properties and Metadata Using .NET

## Overview

Azure Blob Storage containers expose **system properties** and support **user-defined metadata**. This unit covers managing both using the .NET client library.

---

## System Properties vs User-Defined Metadata

| Aspect | System Properties | User-Defined Metadata |
|--------|-------------------|------------------------|
| **Definition** | Read-only properties managed by Azure | Custom key-value pairs you define |
| **Examples** | ETag, Last-Modified, Lease Status | Department, Project, Cost Center |
| **Modification** | Azure-controlled (read-only for most) | Fully modifiable by you |
| **Size Limit** | N/A | 8 KB total per resource |
| **Name Format** | Fixed property names | Must be valid C# identifiers |
| **Purpose** | Resource metadata and state | Custom organization/categorization |

---

## Container System Properties

### Read-Only Properties

```csharp
BlobContainerProperties properties = await containerClient.GetPropertiesAsync();

// System properties
DateTimeOffset lastModified = properties.LastModified;
string eTag = properties.ETag.ToString();
LeaseStatus leaseStatus = properties.LeaseStatus;
LeaseState leaseState = properties.LeaseState;
PublicAccessType publicAccess = properties.PublicAccess;
bool hasImmutabilityPolicy = properties.HasImmutabilityPolicy;
bool hasLegalHold = properties.HasLegalHold;
```

### System Properties Reference

| Property | Type | Description |
|----------|------|-------------|
| **LastModified** | DateTimeOffset | Last modification time (UTC) |
| **ETag** | ETag | Entity tag for concurrency control |
| **LeaseStatus** | LeaseStatus | Locked, Unlocked |
| **LeaseState** | LeaseState | Available, Leased, Expired, Breaking, Broken |
| **LeaseDuration** | LeaseDuration | Infinite, Fixed |
| **PublicAccess** | PublicAccessType | None, Blob, Container |
| **HasImmutabilityPolicy** | bool | Immutability policy present |
| **HasLegalHold** | bool | Legal hold applied |

---

## User-Defined Metadata

### Metadata Naming Rules

✅ **Valid metadata names:**
- Must be valid C# identifiers
- Case-insensitive (Azure stores as lowercase)
- Only alphanumeric characters and underscores
- Cannot start with a number

❌ **Invalid metadata names:**
```
metadata["Cost-Center"]    // Hyphen not allowed
metadata["2ndProject"]     // Starts with number
metadata["Project Name"]   // Space not allowed
```

✅ **Valid examples:**
```
metadata["CostCenter"]
metadata["Project_Name"]
metadata["Department"]
```

### Metadata Size Limits

- **Maximum size**: 8 KB total (name + value pairs combined)
- **Includes**: HTTP header overhead
- **Best practice**: Keep names short and values concise

---

## Retrieve Container Properties and Metadata

### Get All Properties and Metadata

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;

BlobContainerClient containerClient = serviceClient.GetBlobContainerClient("mycontainer");

// Retrieve properties and metadata in one call
BlobContainerProperties properties = await containerClient.GetPropertiesAsync();

// Access system properties
Console.WriteLine($"Last Modified: {properties.LastModified}");
Console.WriteLine($"ETag: {properties.ETag}");
Console.WriteLine($"Lease Status: {properties.LeaseStatus}");

// Access metadata
if (properties.Metadata.Count > 0)
{
    Console.WriteLine("\nMetadata:");
    foreach (var metadataItem in properties.Metadata)
    {
        Console.WriteLine($"  {metadataItem.Key}: {metadataItem.Value}");
    }
}
else
{
    Console.WriteLine("\nNo metadata found");
}
```

### Metadata Structure

```csharp
// Metadata is IDictionary<string, string>
IDictionary<string, string> metadata = properties.Metadata;

// Access specific metadata
if (metadata.ContainsKey("department"))
{
    string department = metadata["department"];
    Console.WriteLine($"Department: {department}");
}

// Iterate all metadata
foreach (KeyValuePair<string, string> item in metadata)
{
    Console.WriteLine($"{item.Key} = {item.Value}");
}
```

---

## Set Container Metadata

### Add or Update Metadata

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;

BlobContainerClient containerClient = serviceClient.GetBlobContainerClient("mycontainer");

// Create metadata dictionary
IDictionary<string, string> metadata = new Dictionary<string, string>
{
    { "department", "engineering" },
    { "project", "az204-training" },
    { "costcenter", "12345" },
    { "environment", "production" }
};

// Set metadata on container
await containerClient.SetMetadataAsync(metadata);

Console.WriteLine("Metadata set successfully");
```

### Update Existing Metadata

```csharp
// Retrieve current properties (includes metadata)
BlobContainerProperties properties = await containerClient.GetPropertiesAsync();

// Modify metadata
IDictionary<string, string> metadata = properties.Metadata;
metadata["lastreviewed"] = DateTime.UtcNow.ToString("yyyy-MM-dd");
metadata["reviewedby"] = "admin@contoso.com";

// Update metadata (replaces all metadata)
await containerClient.SetMetadataAsync(metadata);
```

⚠️ **Important**: `SetMetadataAsync` **replaces all** metadata. Always retrieve current metadata first if you want to preserve existing values.

---

## Complete Container Metadata Example

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;

namespace ContainerMetadataExample
{
    class Program
    {
        static async Task Main(string[] args)
        {
            string storageAccountName = "mystorageaccount";
            string containerName = "mycontainer";

            // Create BlobServiceClient
            var serviceClient = new BlobServiceClient(
                new Uri($"https://{storageAccountName}.blob.core.windows.net"),
                new DefaultAzureCredential());

            // Get container client
            BlobContainerClient containerClient = serviceClient.GetBlobContainerClient(containerName);

            // Ensure container exists
            await containerClient.CreateIfNotExistsAsync();

            // === SET METADATA ===
            Console.WriteLine("Setting container metadata...");
            
            IDictionary<string, string> metadata = new Dictionary<string, string>
            {
                { "department", "engineering" },
                { "project", "blobstorage" },
                { "costcenter", "12345" },
                { "owner", "alice@contoso.com" },
                { "createdate", DateTime.UtcNow.ToString("yyyy-MM-dd") }
            };

            await containerClient.SetMetadataAsync(metadata);
            Console.WriteLine("✓ Metadata set\n");

            // === RETRIEVE PROPERTIES AND METADATA ===
            Console.WriteLine("Retrieving container properties and metadata...");
            
            BlobContainerProperties properties = await containerClient.GetPropertiesAsync();

            // Display system properties
            Console.WriteLine("System Properties:");
            Console.WriteLine($"  Last Modified: {properties.LastModified}");
            Console.WriteLine($"  ETag: {properties.ETag}");
            Console.WriteLine($"  Lease Status: {properties.LeaseStatus}");
            Console.WriteLine($"  Lease State: {properties.LeaseState}");
            Console.WriteLine($"  Public Access: {properties.PublicAccess}");
            Console.WriteLine();

            // Display metadata
            Console.WriteLine("User-Defined Metadata:");
            foreach (var item in properties.Metadata)
            {
                Console.WriteLine($"  {item.Key}: {item.Value}");
            }
            Console.WriteLine();

            // === UPDATE METADATA ===
            Console.WriteLine("Updating metadata...");
            
            IDictionary<string, string> updatedMetadata = properties.Metadata;
            updatedMetadata["lastmodified"] = DateTime.UtcNow.ToString("yyyy-MM-dd HH:mm:ss");
            updatedMetadata["modifiedby"] = "bob@contoso.com";

            await containerClient.SetMetadataAsync(updatedMetadata);
            Console.WriteLine("✓ Metadata updated\n");

            // === VERIFY UPDATE ===
            Console.WriteLine("Verifying metadata update...");
            
            properties = await containerClient.GetPropertiesAsync();
            
            Console.WriteLine("Updated Metadata:");
            foreach (var item in properties.Metadata)
            {
                Console.WriteLine($"  {item.Key}: {item.Value}");
            }
        }
    }
}
```

### Expected Output

```
Setting container metadata...
✓ Metadata set

Retrieving container properties and metadata...
System Properties:
  Last Modified: 1/3/2026 10:30:00 AM +00:00
  ETag: "0x8DCB123456789AB"
  Lease Status: Unlocked
  Lease State: Available
  Public Access: None

User-Defined Metadata:
  department: engineering
  project: blobstorage
  costcenter: 12345
  owner: alice@contoso.com
  createdate: 2026-01-03

Updating metadata...
✓ Metadata updated

Verifying metadata update...
Updated Metadata:
  department: engineering
  project: blobstorage
  costcenter: 12345
  owner: alice@contoso.com
  createdate: 2026-01-03
  lastmodified: 2026-01-03 10:30:15
  modifiedby: bob@contoso.com
```

---

## Manage Blob Properties and Metadata

### Blob System Properties

```csharp
BlobClient blobClient = containerClient.GetBlobClient("myblob.txt");

// Get properties
BlobProperties blobProperties = await blobClient.GetPropertiesAsync();

// System properties
string contentType = blobProperties.ContentType;
long contentLength = blobProperties.ContentLength;
DateTimeOffset lastModified = blobProperties.LastModified;
string eTag = blobProperties.ETag.ToString();
BlobType blobType = blobProperties.BlobType;
string contentEncoding = blobProperties.ContentEncoding;
string cacheControl = blobProperties.CacheControl;
```

### Set Blob HTTP Headers

```csharp
// Create HTTP headers
var headers = new BlobHttpHeaders
{
    ContentType = "text/plain",
    ContentLanguage = "en-US",
    ContentEncoding = "utf-8",
    CacheControl = "max-age=3600",
    ContentDisposition = "attachment; filename=myfile.txt"
};

// Set headers on blob
await blobClient.SetHttpHeadersAsync(headers);

Console.WriteLine("Blob HTTP headers updated");
```

### Set Blob Metadata

```csharp
BlobClient blobClient = containerClient.GetBlobClient("myblob.txt");

// Create metadata
var blobMetadata = new Dictionary<string, string>
{
    { "author", "Alice Smith" },
    { "version", "1.2" },
    { "classification", "public" },
    { "uploadedon", DateTime.UtcNow.ToString("o") }
};

// Set metadata
await blobClient.SetMetadataAsync(blobMetadata);

Console.WriteLine("Blob metadata set");
```

### Retrieve Blob Metadata

```csharp
// Get properties (includes metadata)
BlobProperties blobProperties = await blobClient.GetPropertiesAsync();

// Access metadata
Console.WriteLine("Blob Metadata:");
foreach (var item in blobProperties.Metadata)
{
    Console.WriteLine($"  {item.Key}: {item.Value}");
}
```

---

## Complete Blob Metadata Example

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;

namespace BlobMetadataExample
{
    class Program
    {
        static async Task Main(string[] args)
        {
            string storageAccountName = "mystorageaccount";
            string containerName = "mycontainer";
            string blobName = "sample.txt";

            // Create clients
            var serviceClient = new BlobServiceClient(
                new Uri($"https://{storageAccountName}.blob.core.windows.net"),
                new DefaultAzureCredential());

            BlobContainerClient containerClient = serviceClient.GetBlobContainerClient(containerName);
            await containerClient.CreateIfNotExistsAsync();

            BlobClient blobClient = containerClient.GetBlobClient(blobName);

            // Upload sample blob
            using (var stream = new MemoryStream(System.Text.Encoding.UTF8.GetBytes("Hello, Blob!")))
            {
                await blobClient.UploadAsync(stream, overwrite: true);
            }

            // === SET BLOB HTTP HEADERS ===
            Console.WriteLine("Setting blob HTTP headers...");
            
            var headers = new BlobHttpHeaders
            {
                ContentType = "text/plain",
                ContentLanguage = "en-US",
                ContentEncoding = "utf-8",
                CacheControl = "max-age=3600"
            };

            await blobClient.SetHttpHeadersAsync(headers);
            Console.WriteLine("✓ HTTP headers set\n");

            // === SET BLOB METADATA ===
            Console.WriteLine("Setting blob metadata...");
            
            var metadata = new Dictionary<string, string>
            {
                { "author", "Alice" },
                { "version", "1.0" },
                { "category", "documentation" },
                { "createdate", DateTime.UtcNow.ToString("yyyy-MM-dd") }
            };

            await blobClient.SetMetadataAsync(metadata);
            Console.WriteLine("✓ Metadata set\n");

            // === RETRIEVE PROPERTIES ===
            Console.WriteLine("Retrieving blob properties...");
            
            BlobProperties properties = await blobClient.GetPropertiesAsync();

            Console.WriteLine("HTTP Headers:");
            Console.WriteLine($"  Content-Type: {properties.ContentType}");
            Console.WriteLine($"  Content-Length: {properties.ContentLength}");
            Console.WriteLine($"  Content-Encoding: {properties.ContentEncoding}");
            Console.WriteLine($"  Cache-Control: {properties.CacheControl}");
            Console.WriteLine();

            Console.WriteLine("System Properties:");
            Console.WriteLine($"  ETag: {properties.ETag}");
            Console.WriteLine($"  Last Modified: {properties.LastModified}");
            Console.WriteLine($"  Blob Type: {properties.BlobType}");
            Console.WriteLine();

            Console.WriteLine("Metadata:");
            foreach (var item in properties.Metadata)
            {
                Console.WriteLine($"  {item.Key}: {item.Value}");
            }
        }
    }
}
```

---

## Key Methods Summary

### Container Methods

| Method | Purpose | Returns |
|--------|---------|---------|
| **GetPropertiesAsync()** | Retrieve properties and metadata | BlobContainerProperties |
| **SetMetadataAsync()** | Set/replace all metadata | Response |
| **CreateIfNotExistsAsync()** | Create container if missing | BlobContainerClient |

### Blob Methods

| Method | Purpose | Returns |
|--------|---------|---------|
| **GetPropertiesAsync()** | Retrieve properties, headers, metadata | BlobProperties |
| **SetMetadataAsync()** | Set/replace all metadata | Response |
| **SetHttpHeadersAsync()** | Update HTTP headers | Response |
| **UploadAsync()** | Upload blob with optional metadata | Response |

---

## Best Practices

### 1. Efficient Metadata Retrieval

✅ **Do**: Get properties and metadata in one call
```csharp
BlobContainerProperties props = await containerClient.GetPropertiesAsync();
// Access both props and props.Metadata
```

❌ **Don't**: Make separate calls unnecessarily
```csharp
// Less efficient
var props = await containerClient.GetPropertiesAsync();
var metadata = await containerClient.GetPropertiesAsync().Metadata; // Redundant
```

### 2. Metadata Naming

✅ **Do**: Use clear, descriptive names
```csharp
metadata["department"] = "engineering";
metadata["cost_center"] = "CC-12345";
metadata["project_code"] = "AZ204";
```

❌ **Don't**: Use cryptic abbreviations
```csharp
metadata["dept"] = "eng";  // Less clear
metadata["cc"] = "12345";  // Ambiguous
```

### 3. Preserve Existing Metadata

✅ **Do**: Retrieve before updating
```csharp
var props = await containerClient.GetPropertiesAsync();
var metadata = props.Metadata;
metadata["newkey"] = "newvalue";  // Add to existing
await containerClient.SetMetadataAsync(metadata);
```

❌ **Don't**: Overwrite without retrieving
```csharp
var newMetadata = new Dictionary<string, string> { { "newkey", "value" } };
await containerClient.SetMetadataAsync(newMetadata); // Loses existing metadata!
```

### 4. Handle Case-Insensitivity

```csharp
// Azure stores metadata keys as lowercase
metadata["Department"] = "Engineering";

// Later retrieval (case doesn't matter)
var dept = props.Metadata["department"];  // Works
var dept2 = props.Metadata["DEPARTMENT"]; // Also works
```

### 5. Size Management

```csharp
// Calculate approximate metadata size
int totalSize = 0;
foreach (var item in metadata)
{
    totalSize += item.Key.Length + item.Value.Length;
}

if (totalSize > 7000) // Leave buffer for headers
{
    Console.WriteLine("Warning: Metadata approaching 8 KB limit");
}
```

---

## Error Handling

```csharp
try
{
    await containerClient.SetMetadataAsync(metadata);
}
catch (Azure.RequestFailedException ex) when (ex.Status == 400)
{
    Console.WriteLine("Invalid metadata: " + ex.Message);
    // Likely invalid metadata name
}
catch (Azure.RequestFailedException ex) when (ex.Status == 404)
{
    Console.WriteLine("Container not found");
}
catch (Azure.RequestFailedException ex) when (ex.Status == 403)
{
    Console.WriteLine("Access denied - check RBAC roles");
}
```

---

## Exam Tips

🎯 **GetPropertiesAsync**: Retrieves system properties AND metadata in one call

🎯 **SetMetadataAsync**: REPLACES all metadata (not a merge operation)

🎯 **Metadata naming**: Must be valid C# identifiers (no hyphens, spaces, or special characters)

🎯 **Case-insensitive**: Azure stores metadata keys as lowercase

🎯 **8 KB limit**: Maximum size for all metadata (names + values combined)

🎯 **IDictionary<string, string>**: Metadata collection type in .NET SDK

🎯 **System properties**: Read-only (ETag, LastModified, LeaseStatus, etc.)

🎯 **BlobHttpHeaders**: Use SetHttpHeadersAsync to update Content-Type, Cache-Control, etc.

🎯 **Preserve metadata**: Always retrieve current metadata before updating to avoid losing data

🎯 **Authentication**: Requires Storage Blob Data Contributor role for write operations

---

## Additional Resources

- [Container properties and metadata (.NET)](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-container-properties-metadata)
- [Blob properties and metadata (.NET)](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-properties-metadata)
- [BlobContainerClient.GetPropertiesAsync](https://learn.microsoft.com/en-us/dotnet/api/azure.storage.blobs.blobcontainerclient.getpropertiesasync)
- [BlobContainerClient.SetMetadataAsync](https://learn.microsoft.com/en-us/dotnet/api/azure.storage.blobs.blobcontainerclient.setmetadataasync)

[Microsoft Learn - Manage container properties and metadata by using .NET](https://learn.microsoft.com/en-us/training/modules/work-azure-blob-storage/5-manage-container-properties-metadata-dotnet)
