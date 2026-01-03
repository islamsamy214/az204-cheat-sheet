# Create Blob Client Objects

## Overview

Working with Azure Blob Storage using the SDK begins with **creating client objects**. This unit covers how to create and use the three main client types:

1. **BlobServiceClient** - Storage account level
2. **BlobContainerClient** - Container level
3. **BlobClient** - Blob level

---

## Authentication Prerequisites

### DefaultAzureCredential

The examples in this unit use **DefaultAzureCredential** from the `Azure.Identity` package for authentication.

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;
```

**Authentication Process:**
1. Obtains an access token for authorization
2. Token passed as credential when client is instantiated
3. Credential persists throughout the client lifetime

### Required RBAC Roles

The Microsoft Entra security principal requesting the token must be assigned an appropriate **Azure RBAC role** that grants access to blob data:

| Role | Permissions | Use Case |
|------|-------------|----------|
| **Storage Blob Data Owner** | Full access (read, write, delete, manage ACLs) | Administrative access |
| **Storage Blob Data Contributor** | Read, write, delete blobs | Application write access |
| **Storage Blob Data Reader** | Read and list blobs | Application read-only access |

### Assign RBAC Role

```bash
# Azure CLI - Assign role to current user
az role assignment create \
    --role "Storage Blob Data Contributor" \
    --assignee <user-email-or-object-id> \
    --scope /subscriptions/<subscription-id>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<account>

# Assign to service principal
az role assignment create \
    --role "Storage Blob Data Contributor" \
    --assignee <service-principal-id> \
    --scope /subscriptions/<subscription-id>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<account>
```

---

## 1. Create BlobServiceClient Object

### Purpose

A `BlobServiceClient` allows your app to interact with resources at the **storage account level**.

**Capabilities:**
- ✅ Retrieve and configure account properties
- ✅ List containers in the account
- ✅ Create containers
- ✅ Delete containers
- ✅ Get account statistics

### Basic Creation

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;

public BlobServiceClient GetBlobServiceClient(string accountName)
{
    BlobServiceClient client = new(
        new Uri($"https://{accountName}.blob.core.windows.net"),
        new DefaultAzureCredential());

    return client;
}
```

### URI Format

```
https://<account-name>.blob.core.windows.net
```

**Example:**
```
https://mystorageaccount.blob.core.windows.net
```

### With Client Options

```csharp
public BlobServiceClient GetBlobServiceClientWithOptions(string accountName)
{
    var options = new BlobClientOptions
    {
        Retry =
        {
            MaxRetries = 5,
            Delay = TimeSpan.FromSeconds(2)
        }
    };

    var client = new BlobServiceClient(
        new Uri($"https://{accountName}.blob.core.windows.net"),
        new DefaultAzureCredential(),
        options);

    return client;
}
```

### Common Operations

```csharp
// List all containers
await foreach (var container in serviceClient.GetBlobContainersAsync())
{
    Console.WriteLine($"Container: {container.Name}");
}

// Create a container
BlobContainerClient newContainer = await serviceClient.CreateBlobContainerAsync("mycontainer");

// Delete a container
await serviceClient.DeleteBlobContainerAsync("mycontainer");

// Get account info
var accountInfo = await serviceClient.GetAccountInfoAsync();
Console.WriteLine($"Account Kind: {accountInfo.Value.AccountKind}");
```

---

## 2. Create BlobContainerClient Object

### Purpose

A `BlobContainerClient` allows you to interact with a **specific container resource**.

**Capabilities:**
- ✅ Create container
- ✅ Delete container
- ✅ Configure container properties
- ✅ List blobs in the container
- ✅ Upload blobs
- ✅ Delete blobs

### Method 1: Create from BlobServiceClient (Recommended)

```csharp
public BlobContainerClient GetBlobContainerClient(
    BlobServiceClient blobServiceClient,
    string containerName)
{
    // Create the container client using the service client object
    BlobContainerClient client = blobServiceClient.GetBlobContainerClient(containerName);
    return client;
}
```

**Advantages:**
- ✅ Leverages existing service client
- ✅ Shares configuration and credentials
- ✅ Cleaner hierarchy

### Method 2: Create Directly

If your work is narrowly scoped to a single container, you can create a `BlobContainerClient` directly:

```csharp
public BlobContainerClient GetBlobContainerClient(
    string accountName,
    string containerName,
    BlobClientOptions clientOptions)
{
    // Append the container name to the end of the URI
    BlobContainerClient client = new(
        new Uri($"https://{accountName}.blob.core.windows.net/{containerName}"),
        new DefaultAzureCredential(),
        clientOptions);

    return client;
}
```

### URI Format

```
https://<account-name>.blob.core.windows.net/<container-name>
```

**Example:**
```
https://mystorageaccount.blob.core.windows.net/mycontainer
```

### Common Operations

```csharp
// Create container if it doesn't exist
await containerClient.CreateIfNotExistsAsync();

// Upload a blob
await containerClient.UploadBlobAsync("file.txt", File.OpenRead("./file.txt"));

// List blobs
await foreach (var blobItem in containerClient.GetBlobsAsync())
{
    Console.WriteLine($"Blob: {blobItem.Name}");
}

// Delete blob
await containerClient.DeleteBlobAsync("file.txt");

// Get container properties
var properties = await containerClient.GetPropertiesAsync();
Console.WriteLine($"Last Modified: {properties.Value.LastModified}");
```

---

## 3. Create BlobClient Object

### Purpose

A `BlobClient` allows you to interact with a **specific blob resource**.

**Capabilities:**
- ✅ Upload blob
- ✅ Download blob
- ✅ Delete blob
- ✅ Copy blob
- ✅ Get/set blob properties
- ✅ Get/set blob metadata
- ✅ Set access tier

### Method 1: Create from Service/Container Client (Recommended)

```csharp
public BlobClient GetBlobClient(
    BlobServiceClient blobServiceClient,
    string containerName,
    string blobName)
{
    BlobClient client =
        blobServiceClient.GetBlobContainerClient(containerName).GetBlobClient(blobName);
    return client;
}
```

### Method 2: Create from Container Client

```csharp
public BlobClient GetBlobClientFromContainer(
    BlobContainerClient containerClient,
    string blobName)
{
    BlobClient client = containerClient.GetBlobClient(blobName);
    return client;
}
```

### Method 3: Create Directly

```csharp
public BlobClient GetBlobClientDirect(
    string accountName,
    string containerName,
    string blobName)
{
    var client = new BlobClient(
        new Uri($"https://{accountName}.blob.core.windows.net/{containerName}/{blobName}"),
        new DefaultAzureCredential());
    
    return client;
}
```

### URI Format

```
https://<account-name>.blob.core.windows.net/<container-name>/<blob-name>
```

**Example:**
```
https://mystorageaccount.blob.core.windows.net/mycontainer/myfile.txt
```

### Common Operations

```csharp
// Upload file
await blobClient.UploadAsync("./localfile.txt", overwrite: true);

// Download to file
await blobClient.DownloadToAsync("./downloadedfile.txt");

// Download to stream
using var stream = new MemoryStream();
await blobClient.DownloadToAsync(stream);

// Check if blob exists
bool exists = await blobClient.ExistsAsync();

// Get blob properties
var properties = await blobClient.GetPropertiesAsync();
Console.WriteLine($"Content Type: {properties.Value.ContentType}");
Console.WriteLine($"Size: {properties.Value.ContentLength} bytes");

// Set access tier
await blobClient.SetAccessTierAsync(AccessTier.Cool);

// Delete blob
await blobClient.DeleteAsync();
```

---

## Client Creation Patterns

### Pattern 1: Top-Down Navigation (Recommended)

```csharp
// Start at service level
var serviceClient = new BlobServiceClient(
    new Uri("https://account.blob.core.windows.net"),
    new DefaultAzureCredential());

// Navigate to container
var containerClient = serviceClient.GetBlobContainerClient("mycontainer");

// Navigate to blob
var blobClient = containerClient.GetBlobClient("myfile.txt");
```

**Advantages:**
- ✅ Clearest hierarchy
- ✅ Shares configuration across levels
- ✅ Easy to navigate between levels

### Pattern 2: Direct Client Creation

```csharp
// Create container client directly
var containerClient = new BlobContainerClient(
    new Uri("https://account.blob.core.windows.net/mycontainer"),
    new DefaultAzureCredential());

// Create blob client directly
var blobClient = new BlobClient(
    new Uri("https://account.blob.core.windows.net/mycontainer/myfile.txt"),
    new DefaultAzureCredential());
```

**Use When:**
- ✅ Work is scoped to specific container/blob
- ✅ Don't need account-level operations
- ✅ Working with specific URIs

### Pattern 3: Using Connection String

```csharp
string connectionString = "DefaultEndpointsProtocol=https;AccountName=...";

// Service client
var serviceClient = new BlobServiceClient(connectionString);

// Container client
var containerClient = new BlobContainerClient(connectionString, "mycontainer");

// Blob client
var blobClient = new BlobClient(connectionString, "mycontainer", "myfile.txt");
```

⚠️ **Note:** Connection strings contain sensitive credentials. Use DefaultAzureCredential in production when possible.

---

## Complete Example

### Full Workflow with All Three Clients

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;

public class BlobStorageExample
{
    private readonly string accountName = "mystorageaccount";
    
    public async Task CompleteWorkflowAsync()
    {
        // 1. Create service client (account level)
        var serviceClient = new BlobServiceClient(
            new Uri($"https://{accountName}.blob.core.windows.net"),
            new DefaultAzureCredential());
        
        Console.WriteLine("Created BlobServiceClient");
        
        // 2. List existing containers
        Console.WriteLine("\nExisting containers:");
        await foreach (var container in serviceClient.GetBlobContainersAsync())
        {
            Console.WriteLine($"  - {container.Name}");
        }
        
        // 3. Create container client
        var containerName = $"demo-{Guid.NewGuid()}";
        var containerClient = serviceClient.GetBlobContainerClient(containerName);
        
        Console.WriteLine($"\nCreating container: {containerName}");
        await containerClient.CreateAsync();
        
        // 4. Create blob client
        var blobName = "sample.txt";
        var blobClient = containerClient.GetBlobClient(blobName);
        
        // 5. Upload data
        Console.WriteLine($"\nUploading blob: {blobName}");
        await blobClient.UploadAsync(
            BinaryData.FromString("Hello, Azure Blob Storage!"),
            overwrite: true);
        
        // 6. Verify blob exists
        bool exists = await blobClient.ExistsAsync();
        Console.WriteLine($"Blob exists: {exists}");
        
        // 7. Download blob
        Console.WriteLine("\nDownloading blob...");
        var download = await blobClient.DownloadContentAsync();
        Console.WriteLine($"Content: {download.Value.Content}");
        
        // 8. Cleanup
        Console.WriteLine("\nCleaning up...");
        await containerClient.DeleteAsync();
        Console.WriteLine("Container deleted");
    }
}
```

---

## URI Construction with BlobUriBuilder

### Manual URI Construction

```csharp
// Manually construct URI
var uri = new Uri($"https://{accountName}.blob.core.windows.net/{containerName}/{blobName}");
```

### Using BlobUriBuilder

```csharp
var builder = new BlobUriBuilder(new Uri($"https://{accountName}.blob.core.windows.net"))
{
    BlobContainerName = containerName,
    BlobName = blobName
};

Uri blobUri = builder.ToUri();
var blobClient = new BlobClient(blobUri, new DefaultAzureCredential());
```

**Advantages:**
- ✅ Type-safe URI construction
- ✅ Easy to modify URI components
- ✅ Handles URL encoding

---

## Best Practices

### Client Lifecycle

✅ **DO:**
- Create clients once and reuse them (singleton pattern)
- Use `BlobServiceClient` for account-level operations
- Navigate from service → container → blob when possible
- Use appropriate client for the scope of work
- Share credentials and options across clients

❌ **DON'T:**
- Create new clients for every operation
- Create clients in loops
- Mix connection strings and credential objects unnecessarily

### Authentication

✅ **DO:**
- Use `DefaultAzureCredential` for production
- Assign appropriate RBAC roles
- Use Managed Identity in Azure-hosted applications
- Store connection strings securely (Key Vault)

❌ **DON'T:**
- Hardcode connection strings in code
- Use connection strings when Managed Identity is available
- Grant excessive RBAC permissions

### URI Construction

✅ **DO:**
- Use `BlobUriBuilder` for complex URIs
- Validate URI format
- Include proper error handling

❌ **DON'T:**
- Manually concatenate URI strings without encoding
- Forget to encode special characters in blob names

---

## Exam Tips

🎯 **Three client types**: BlobServiceClient (account), BlobContainerClient (container), BlobClient (blob)

🎯 **DefaultAzureCredential**: Recommended authentication method, tries multiple credential types

🎯 **RBAC roles**: Storage Blob Data Owner/Contributor/Reader required for Azure AD auth

🎯 **Client hierarchy**: Service → Container → Blob (navigational pattern)

🎯 **URI format**: `https://{account}.blob.core.windows.net/{container}/{blob}`

🎯 **Client reuse**: Create once, reuse multiple times (singleton pattern)

🎯 **GetBlobContainerClient**: Method to navigate from service to container client

🎯 **GetBlobClient**: Method to navigate from container to blob client

🎯 **BlobUriBuilder**: Convenient class for constructing and modifying blob URIs

🎯 **Direct creation**: Can create container/blob clients directly without service client

---

## Quick Reference Commands

### Create Service Client

```csharp
var serviceClient = new BlobServiceClient(
    new Uri($"https://{accountName}.blob.core.windows.net"),
    new DefaultAzureCredential());
```

### Create Container Client (from service)

```csharp
var containerClient = serviceClient.GetBlobContainerClient(containerName);
```

### Create Blob Client (from container)

```csharp
var blobClient = containerClient.GetBlobClient(blobName);
```

### Create Blob Client (from service, one line)

```csharp
var blobClient = serviceClient
    .GetBlobContainerClient(containerName)
    .GetBlobClient(blobName);
```

---

## Additional Resources

- [BlobServiceClient Class](https://learn.microsoft.com/en-us/dotnet/api/azure.storage.blobs.blobserviceclient)
- [BlobContainerClient Class](https://learn.microsoft.com/en-us/dotnet/api/azure.storage.blobs.blobcontainerclient)
- [BlobClient Class](https://learn.microsoft.com/en-us/dotnet/api/azure.storage.blobs.blobclient)
- [DefaultAzureCredential Class](https://learn.microsoft.com/en-us/dotnet/api/azure.identity.defaultazurecredential)

[Microsoft Learn - Create a client object](https://learn.microsoft.com/en-us/training/modules/work-azure-blob-storage/3-create-client-object)
