# Explore Azure Blob Storage Client Library

## Key Concepts
- **Azure.Storage.Blobs** - Main NuGet package for Blob Storage operations
- **BlobServiceClient** - Entry point for storage account-level operations
- **BlobContainerClient** - Manages containers and their blobs
- **BlobClient** - Manipulates individual blobs
- **Version 12.x** - Latest recommended client library version

## Azure Storage Client Libraries

### Package Information

**Primary package**:
- **Azure.Storage.Blobs** - Contains all main client classes

**Additional packages**:
- **Azure.Storage.Blobs.Specialized** - Blob type-specific operations (block blobs, append blobs, page blobs)
- **Azure.Storage.Blobs.Models** - Utility classes, structures, and enumerations

### Installation

```bash
# Install via .NET CLI
dotnet add package Azure.Storage.Blobs

# Install via Package Manager Console
Install-Package Azure.Storage.Blobs

# Install specific version
dotnet add package Azure.Storage.Blobs --version 12.21.0
```

## Core Client Classes

### Client Class Hierarchy

| Class | Purpose | Scope |
|-------|---------|-------|
| **BlobServiceClient** | Storage account operations | Account-level |
| **BlobContainerClient** | Container operations | Container-level |
| **BlobClient** | Blob operations | Blob-level |
| **BlobUriBuilder** | URI manipulation | Helper utility |

### BlobServiceClient

**Purpose**: Entry point for storage account-level operations

**Capabilities**:
- Retrieve and configure account properties
- List containers in the storage account
- Create and delete containers
- Get service statistics
- Set service properties

**Common methods**:
- `GetBlobContainerClient()` - Get container client
- `CreateBlobContainerAsync()` - Create container
- `DeleteBlobContainerAsync()` - Delete container
- `GetBlobContainersAsync()` - List containers
- `GetPropertiesAsync()` - Get account properties
- `SetPropertiesAsync()` - Set account properties

### BlobContainerClient

**Purpose**: Container-level operations

**Capabilities**:
- Create, delete, or configure containers
- List blobs within the container
- Upload and delete blobs
- Manage container properties and metadata
- Set container access policy

**Common methods**:
- `CreateAsync()` - Create container
- `DeleteAsync()` - Delete container
- `GetBlobClient()` - Get blob client
- `UploadBlobAsync()` - Upload blob
- `DeleteBlobAsync()` - Delete blob
- `GetBlobsAsync()` - List blobs
- `GetPropertiesAsync()` - Get container properties
- `SetMetadataAsync()` - Set container metadata

### BlobClient

**Purpose**: Individual blob operations

**Capabilities**:
- Upload, download, and delete blobs
- Copy blobs
- Create snapshots
- Manage blob properties and metadata
- Set blob tier

**Common methods**:
- `UploadAsync()` - Upload blob content
- `DownloadAsync()` - Download blob
- `DeleteAsync()` - Delete blob
- `ExistsAsync()` - Check if blob exists
- `GetPropertiesAsync()` - Get blob properties
- `SetMetadataAsync()` - Set blob metadata
- `SetHttpHeadersAsync()` - Set HTTP headers
- `CreateSnapshotAsync()` - Create snapshot

### BlobUriBuilder

**Purpose**: Construct and modify blob URIs

**Capabilities**:
- Parse existing blob URIs
- Modify URI components (account, container, blob)
- Add SAS tokens to URIs
- Change blob snapshots or versions

**Usage**:
```csharp
var builder = new BlobUriBuilder(new Uri("https://account.blob.core.windows.net/container/blob"));
builder.BlobName = "newblob.txt";
Uri newUri = builder.ToUri();
```

## Client Configuration

### BlobClientOptions

**Configure client behavior**:

```csharp
var options = new BlobClientOptions
{
    Retry = 
    {
        MaxRetries = 5,
        Delay = TimeSpan.FromSeconds(2),
        Mode = RetryMode.Exponential
    },
    Diagnostics =
    {
        IsLoggingEnabled = true,
        ApplicationId = "MyApp/1.0"
    }
};

var client = new BlobServiceClient(connectionString, options);
```

**Common options**:
- **Retry** - Configure retry policy
- **Diagnostics** - Enable logging and tracing
- **Transport** - Configure HTTP transport
- **ApiVersion** - Specify service version

## Client Lifetime Management

### Singleton Pattern (Recommended)

**Single instance for application lifetime**:

```csharp
public class BlobStorageService
{
    private static readonly BlobServiceClient _serviceClient;

    static BlobStorageService()
    {
        _serviceClient = new BlobServiceClient(connectionString);
    }

    public static BlobServiceClient GetServiceClient() => _serviceClient;
}
```

### Dependency Injection

**Register in ASP.NET Core**:

```csharp
// Startup.cs or Program.cs
services.AddSingleton(sp =>
{
    var connectionString = configuration.GetConnectionString("AzureStorage");
    return new BlobServiceClient(connectionString);
});

// Usage in controller/service
public class MyService
{
    private readonly BlobServiceClient _blobServiceClient;

    public MyService(BlobServiceClient blobServiceClient)
    {
        _blobServiceClient = blobServiceClient;
    }
}
```

## Authentication Methods

### Connection String

**Using storage account connection string**:

```csharp
string connectionString = "DefaultEndpointsProtocol=https;AccountName=myaccount;AccountKey=key;EndpointSuffix=core.windows.net";

BlobServiceClient serviceClient = new BlobServiceClient(connectionString);
```

### Account Key

**Using account name and key**:

```csharp
string accountName = "myaccount";
string accountKey = "key";

var credential = new StorageSharedKeyCredential(accountName, accountKey);
var serviceClient = new BlobServiceClient(
    new Uri($"https://{accountName}.blob.core.windows.net"),
    credential
);
```

### Azure AD (Microsoft Entra ID)

**Using DefaultAzureCredential**:

```csharp
using Azure.Identity;

var credential = new DefaultAzureCredential();
var serviceClient = new BlobServiceClient(
    new Uri("https://myaccount.blob.core.windows.net"),
    credential
);
```

### SAS Token

**Using Shared Access Signature**:

```csharp
string sasToken = "?sv=2021-06-08&ss=b&srt=sco&sp=rwdlac&se=...";
var serviceClient = new BlobServiceClient(
    new Uri($"https://myaccount.blob.core.windows.net{sasToken}")
);
```

### Anonymous Access

**Public containers only**:

```csharp
var serviceClient = new BlobServiceClient(
    new Uri("https://myaccount.blob.core.windows.net")
);
```

## Package Namespaces

### Azure.Storage.Blobs

**Main classes**:
- `BlobServiceClient` - Service-level operations
- `BlobContainerClient` - Container operations
- `BlobClient` - Blob operations
- `BlobClientOptions` - Configuration options
- `BlobUriBuilder` - URI builder

### Azure.Storage.Blobs.Specialized

**Specialized blob types**:
- `BlockBlobClient` - Block blob operations (default)
- `AppendBlobClient` - Append blob operations (logs)
- `PageBlobClient` - Page blob operations (VHDs)
- `BlobLeaseClient` - Lease management
- `BlobBatchClient` - Batch operations

**Example**:
```csharp
// Work with block blob specifically
var blockBlobClient = containerClient.GetBlockBlobClient("file.txt");

// Upload with specific block blob features
await blockBlobClient.UploadAsync(stream, new BlockBlobUploadOptions
{
    HttpHeaders = new BlobHttpHeaders { ContentType = "text/plain" },
    Metadata = new Dictionary<string, string> { { "key", "value" } }
});
```

### Azure.Storage.Blobs.Models

**Data structures and enumerations**:
- `BlobProperties` - Blob property information
- `BlobContainerProperties` - Container properties
- `BlobHttpHeaders` - HTTP header settings
- `PublicAccessType` - Container access levels
- `BlobType` - Blob type enum (Block, Append, Page)
- `BlobDownloadInfo` - Download result
- `BlobItem` - Blob list item
- `BlobContainerItem` - Container list item

## Common Patterns

### Pattern 1: Account → Container → Blob

**Hierarchical navigation**:

```csharp
// Start at account level
var serviceClient = new BlobServiceClient(connectionString);

// Navigate to container
var containerClient = serviceClient.GetBlobContainerClient("mycontainer");

// Navigate to blob
var blobClient = containerClient.GetBlobClient("myblob.txt");
```

### Pattern 2: Direct to Container

**Skip account level if not needed**:

```csharp
var containerClient = new BlobContainerClient(
    connectionString,
    "mycontainer"
);

var blobClient = containerClient.GetBlobClient("myblob.txt");
```

### Pattern 3: Direct to Blob

**Go straight to blob**:

```csharp
var blobClient = new BlobClient(
    connectionString,
    "mycontainer",
    "myblob.txt"
);
```

## Error Handling

### RequestFailedException

**All Azure Storage exceptions**:

```csharp
using Azure;

try
{
    await blobClient.UploadAsync(stream);
}
catch (RequestFailedException ex) when (ex.Status == 404)
{
    Console.WriteLine("Container not found");
}
catch (RequestFailedException ex) when (ex.Status == 409)
{
    Console.WriteLine("Blob already exists");
}
catch (RequestFailedException ex)
{
    Console.WriteLine($"Error: {ex.Status} - {ex.ErrorCode}");
    Console.WriteLine(ex.Message);
}
```

### Common Status Codes

| Status | ErrorCode | Meaning |
|--------|-----------|---------|
| 404 | ContainerNotFound | Container doesn't exist |
| 404 | BlobNotFound | Blob doesn't exist |
| 409 | ContainerAlreadyExists | Container exists |
| 409 | BlobAlreadyExists | Blob exists |
| 403 | AuthorizationFailure | Insufficient permissions |
| 401 | InvalidAuthenticationInfo | Invalid credentials |

## Critical Notes
- 💡 **Version 12.x** - Latest recommended client library version
- 🎯 **Three main clients** - BlobServiceClient (account), BlobContainerClient (container), BlobClient (blob)
- ✅ **NuGet package** - Azure.Storage.Blobs contains core functionality
- ⚠️ **Singleton pattern** - Create clients once, reuse throughout application
- 🔒 **Authentication** - Support for connection string, account key, Azure AD, SAS token
- 📊 **Specialized package** - Azure.Storage.Blobs.Specialized for block/append/page blobs
- 💡 **BlobClientOptions** - Configure retry, logging, transport behavior
- ✅ **Error handling** - Catch RequestFailedException, check Status property
- ⚠️ **URI construction** - Use BlobUriBuilder for safe URI manipulation
- 🔄 **Dependency injection** - Register as singleton in ASP.NET Core

## Exam Tips
- Client library: Azure.Storage.Blobs version 12.x recommended
- Three client classes: BlobServiceClient (account-level), BlobContainerClient (container), BlobClient (blob)
- BlobServiceClient: Entry point, list/create/delete containers, account properties
- BlobContainerClient: Container operations, list/upload/delete blobs, metadata
- BlobClient: Individual blob operations, upload/download/delete/properties
- Packages: Azure.Storage.Blobs (main), Azure.Storage.Blobs.Specialized (blob types), Azure.Storage.Blobs.Models (utilities)
- Authentication: Connection string, StorageSharedKeyCredential, DefaultAzureCredential, SAS token
- DefaultAzureCredential: Recommended for Azure AD authentication
- BlobClientOptions: Configure retry policy, logging, diagnostics
- Singleton pattern: Create client once, reuse for application lifetime
- Dependency injection: Register BlobServiceClient as singleton in ASP.NET Core
- Error handling: Catch RequestFailedException, check Status (404/409/403/401)
- URI builder: Use BlobUriBuilder to construct and modify URIs
- Navigation: serviceClient → GetBlobContainerClient → GetBlobClient
- Direct access: Can create BlobContainerClient or BlobClient directly

[Learn More](https://learn.microsoft.com/en-us/training/modules/work-azure-blob-storage/2-blob-storage-client-library-overview)
