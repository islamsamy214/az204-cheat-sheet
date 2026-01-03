# Azure Blob Storage Client Library Overview

## Introduction

The **Azure Storage client libraries for .NET** provide a convenient interface for making calls to Azure Storage. Microsoft recommends using **version 12.x** for new applications.

---

## Core Client Classes

The following table lists the basic classes in the Azure Blob Storage client library:

| Class | Purpose |
|-------|---------|
| **BlobServiceClient** | Manipulate Azure Storage service resources and blob containers. Provides the top-level namespace for the Blob service. |
| **BlobContainerClient** | Manipulate Azure Storage containers and their blobs. |
| **BlobClient** | Manipulate Azure Storage blobs (upload, download, delete, properties). |
| **BlobClientOptions** | Provides client configuration options for connecting to Azure Blob Storage. |
| **BlobUriBuilder** | Convenient way to modify URI contents to point to different Azure Storage resources (account, container, blob). |

---

## Client Hierarchy

The client objects follow a hierarchical structure:

```
BlobServiceClient (Storage Account Level)
    ↓
BlobContainerClient (Container Level)
    ↓
BlobClient (Blob Level)
```

### Navigation Pattern

```csharp
// Start at service level
BlobServiceClient serviceClient = new BlobServiceClient(connectionString);

// Navigate to container level
BlobContainerClient containerClient = serviceClient.GetBlobContainerClient("mycontainer");

// Navigate to blob level
BlobClient blobClient = containerClient.GetBlobClient("myblob.txt");
```

---

## NuGet Packages

The Azure Blob Storage client library is distributed across multiple packages:

### 1. Azure.Storage.Blobs (Primary Package)

**Purpose**: Contains the primary classes (client objects) to operate on the service, containers, and blobs.

**Key Classes:**
- `BlobServiceClient`
- `BlobContainerClient`
- `BlobClient`
- `BlobClientOptions`
- `BlobUriBuilder`

**Installation:**
```bash
dotnet add package Azure.Storage.Blobs
```

### 2. Azure.Storage.Blobs.Specialized

**Purpose**: Contains classes for operations specific to blob types (block blobs, append blobs, page blobs).

**Key Classes:**
- `BlockBlobClient` - Operations specific to block blobs
- `AppendBlobClient` - Operations specific to append blobs
- `PageBlobClient` - Operations specific to page blobs
- `BlobLeaseClient` - Lease management operations

**Example Usage:**
```csharp
using Azure.Storage.Blobs.Specialized;

// Work with block blobs specifically
BlockBlobClient blockBlobClient = containerClient.GetBlockBlobClient("data.bin");
await blockBlobClient.UploadAsync(stream);

// Work with append blobs (logging scenarios)
AppendBlobClient appendBlobClient = containerClient.GetAppendBlobClient("log.txt");
await appendBlobClient.CreateAsync();
await appendBlobClient.AppendBlockAsync(logStream);
```

### 3. Azure.Storage.Blobs.Models

**Purpose**: All other utility classes, structures, and enumeration types.

**Contains:**
- Enumerations (e.g., `AccessTier`, `BlobType`, `PublicAccessType`)
- Response models
- Request options
- Error types

**Example:**
```csharp
using Azure.Storage.Blobs.Models;

// Use enumerations and models
BlobUploadOptions options = new BlobUploadOptions
{
    AccessTier = AccessTier.Cool,
    HttpHeaders = new BlobHttpHeaders
    {
        ContentType = "application/json"
    }
};
```

---

## Package Namespace Structure

```
Azure.Storage.Blobs
├── BlobServiceClient
├── BlobContainerClient
├── BlobClient
├── BlobClientOptions
└── BlobUriBuilder

Azure.Storage.Blobs.Specialized
├── BlockBlobClient
├── AppendBlobClient
├── PageBlobClient
└── BlobLeaseClient

Azure.Storage.Blobs.Models
├── AccessTier (enum)
├── BlobType (enum)
├── BlobProperties
├── BlobHttpHeaders
└── ... (other models and types)
```

---

## Installation

### Using .NET CLI

```bash
# Install primary package
dotnet add package Azure.Storage.Blobs

# Install specialized package (if needed)
dotnet add package Azure.Storage.Blobs.Specialized

# Install identity package for authentication
dotnet add package Azure.Identity
```

### Using Package Manager Console

```powershell
Install-Package Azure.Storage.Blobs
Install-Package Azure.Storage.Blobs.Specialized
Install-Package Azure.Identity
```

### Using Visual Studio

1. Right-click on project → Manage NuGet Packages
2. Search for "Azure.Storage.Blobs"
3. Click Install

---

## Basic Usage Example

### Complete Example with All Packages

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;
using Azure.Storage.Blobs.Specialized;

namespace BlobStorageExample
{
    public class BlobOperations
    {
        private readonly string accountName = "mystorageaccount";
        private readonly string containerName = "mycontainer";
        
        public async Task BasicOperationsAsync()
        {
            // 1. Create service client (Azure.Storage.Blobs)
            var serviceClient = new BlobServiceClient(
                new Uri($"https://{accountName}.blob.core.windows.net"),
                new DefaultAzureCredential());
            
            // 2. Get container client (Azure.Storage.Blobs)
            var containerClient = serviceClient.GetBlobContainerClient(containerName);
            await containerClient.CreateIfNotExistsAsync();
            
            // 3. Upload blob (Azure.Storage.Blobs)
            var blobClient = containerClient.GetBlobClient("data.txt");
            await blobClient.UploadAsync("./data.txt");
            
            // 4. Work with specialized blob client (Azure.Storage.Blobs.Specialized)
            var blockBlobClient = containerClient.GetBlockBlobClient("block.bin");
            await blockBlobClient.UploadAsync(stream);
            
            // 5. Use models and enums (Azure.Storage.Blobs.Models)
            var options = new BlobUploadOptions
            {
                AccessTier = AccessTier.Cool,
                HttpHeaders = new BlobHttpHeaders
                {
                    ContentType = "text/plain"
                }
            };
        }
    }
}
```

---

## BlobClientOptions Configuration

The `BlobClientOptions` class provides configuration for client behavior:

```csharp
var options = new BlobClientOptions
{
    // Retry configuration
    Retry =
    {
        MaxRetries = 5,
        Delay = TimeSpan.FromSeconds(2),
        MaxDelay = TimeSpan.FromSeconds(10),
        Mode = RetryMode.Exponential
    },
    
    // Diagnostics
    Diagnostics =
    {
        IsLoggingEnabled = true,
        ApplicationId = "MyApp"
    }
};

var serviceClient = new BlobServiceClient(connectionString, options);
```

---

## Version Recommendations

| Version | Status | Recommendation |
|---------|--------|----------------|
| **12.x** | Current | ✅ **Recommended for new applications** |
| 11.x | Legacy | ⚠️ Migrate to 12.x |
| < 11.x | Deprecated | ❌ No longer supported |

### Version 12.x Benefits

✅ Modern async/await patterns
✅ Improved performance
✅ Better error handling
✅ Azure Identity integration
✅ Consistent API across Azure SDKs
✅ Active support and updates

---

## Authentication Integration

Version 12.x integrates seamlessly with **Azure.Identity** package:

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;

// DefaultAzureCredential (recommended)
var serviceClient = new BlobServiceClient(
    new Uri("https://account.blob.core.windows.net"),
    new DefaultAzureCredential());

// Managed Identity
var serviceClient = new BlobServiceClient(
    new Uri("https://account.blob.core.windows.net"),
    new ManagedIdentityCredential());

// Connection String
var serviceClient = new BlobServiceClient(connectionString);
```

---

## Key Features by Package

### Azure.Storage.Blobs Features

- ✅ Create/delete containers
- ✅ Upload/download blobs
- ✅ List blobs
- ✅ Copy blobs
- ✅ Get/set properties and metadata
- ✅ Set access tiers
- ✅ Generate SAS tokens

### Azure.Storage.Blobs.Specialized Features

- ✅ Block blob operations (stage blocks, commit block list)
- ✅ Append blob operations (append blocks)
- ✅ Page blob operations (upload pages, clear pages)
- ✅ Blob lease operations (acquire, release, renew)
- ✅ Blob versioning operations

### Azure.Storage.Blobs.Models Features

- ✅ Access tier enumerations
- ✅ Blob type enumerations
- ✅ HTTP headers configuration
- ✅ Response models
- ✅ Conditions and filters

---

## Best Practices

### Package Usage

✅ **DO:**
- Use `Azure.Storage.Blobs` for most scenarios
- Add `Azure.Storage.Blobs.Specialized` only when needed
- Use version 12.x for new applications
- Install `Azure.Identity` for authentication
- Use `BlobClientOptions` for retry and diagnostics

❌ **DON'T:**
- Mix v11 and v12 packages
- Install packages you don't need
- Use deprecated versions

### Client Lifecycle

✅ **DO:**
- Create clients once and reuse
- Use singleton pattern for service clients
- Dispose clients when done (implements IDisposable)

❌ **DON'T:**
- Create new clients for every operation
- Create clients in tight loops

---

## Exam Tips

🎯 **Version 12.x**: Microsoft recommends version 12.x for new applications

🎯 **Three main classes**: BlobServiceClient (service), BlobContainerClient (container), BlobClient (blob)

🎯 **Three packages**: Azure.Storage.Blobs (primary), Specialized (blob-type specific), Models (utilities)

🎯 **Hierarchy**: Service → Container → Blob (navigational pattern)

🎯 **BlobClientOptions**: Configure retry, diagnostics, transport options

🎯 **Specialized package**: BlockBlobClient, AppendBlobClient, PageBlobClient for type-specific operations

🎯 **Authentication**: Integrates with Azure.Identity package (DefaultAzureCredential)

🎯 **BlobUriBuilder**: Convenient way to construct and modify blob URIs

---

## Quick Reference

### Required Packages

```xml
<ItemGroup>
  <PackageReference Include="Azure.Storage.Blobs" Version="12.x" />
  <PackageReference Include="Azure.Identity" Version="1.x" />
</ItemGroup>
```

### Basic Client Creation

```csharp
// Service client
var serviceClient = new BlobServiceClient(uri, credential);

// Container client
var containerClient = new BlobContainerClient(uri, credential);

// Blob client
var blobClient = new BlobClient(uri, credential);
```

### Specialized Clients

```csharp
// Block blob
var blockBlob = new BlockBlobClient(uri, credential);

// Append blob
var appendBlob = new AppendBlobClient(uri, credential);

// Page blob
var pageBlob = new PageBlobClient(uri, credential);
```

---

## Additional Resources

- [Azure.Storage.Blobs API Reference](https://learn.microsoft.com/en-us/dotnet/api/azure.storage.blobs)
- [Azure.Storage.Blobs.Specialized API Reference](https://learn.microsoft.com/en-us/dotnet/api/azure.storage.blobs.specialized)
- [Azure.Storage.Blobs.Models API Reference](https://learn.microsoft.com/en-us/dotnet/api/azure.storage.blobs.models)
- [Azure Blob Storage client library for .NET](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-dotnet)

[Microsoft Learn - Explore Azure Blob storage client library](https://learn.microsoft.com/en-us/training/modules/work-azure-blob-storage/2-blob-storage-client-library-overview)
