# Create Client Objects for Azure Blob Storage

## Key Concepts
- **Client hierarchy** - ServiceClient → ContainerClient → BlobClient
- **Authentication** - DefaultAzureCredential recommended for Azure AD
- **URI construction** - https://{account}.blob.core.windows.net/{container}/{blob}
- **RBAC roles** - Storage Blob Data Owner/Contributor/Reader for Azure AD auth
- **Client reuse** - Create once, use throughout application lifetime

## Client Object Overview

### Three-Level Hierarchy

```
BlobServiceClient (Account Level)
    ↓
BlobContainerClient (Container Level)
    ↓
BlobClient (Blob Level)
```

**When to use each**:
- **BlobServiceClient** - List containers, create containers, account properties
- **BlobContainerClient** - Container operations, list blobs, upload files
- **BlobClient** - Single blob operations, download, delete, properties

## Creating BlobServiceClient

### Using DefaultAzureCredential (Recommended)

**Azure AD authentication**:

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

**Explanation**:
- `DefaultAzureCredential` - Tries multiple authentication methods in order
- **Local development** - Uses Visual Studio, Azure CLI, or environment variables
- **Azure deployment** - Uses Managed Identity automatically
- **No secrets in code** - Credentials managed externally

### DefaultAzureCredential Chain

**Authentication methods tried (in order)**:

1. **EnvironmentCredential** - Environment variables
2. **WorkloadIdentityCredential** - Azure Kubernetes workload identity
3. **ManagedIdentityCredential** - Managed Identity (Azure resources)
4. **SharedTokenCacheCredential** - Cached tokens
5. **VisualStudioCredential** - Visual Studio account
6. **VisualStudioCodeCredential** - VS Code account
7. **AzureCliCredential** - Azure CLI login
8. **AzurePowerShellCredential** - Azure PowerShell login
9. **AzureDeveloperCliCredential** - Azure Developer CLI
10. **InteractiveBrowserCredential** - Browser popup (last resort)

### Configure DefaultAzureCredential

**Exclude specific authentication methods**:

```csharp
var options = new DefaultAzureCredentialOptions
{
    ExcludeEnvironmentCredential = true,
    ExcludeManagedIdentityCredential = true,
    ExcludeSharedTokenCacheCredential = true,
    ExcludeVisualStudioCredential = false,  // Keep for local dev
    ExcludeAzureCliCredential = false  // Keep for local dev
};

var credential = new DefaultAzureCredential(options);
var serviceClient = new BlobServiceClient(
    new Uri($"https://{accountName}.blob.core.windows.net"),
    credential
);
```

### Using Connection String

**Development/testing scenarios**:

```csharp
string connectionString = Environment.GetEnvironmentVariable("AZURE_STORAGE_CONNECTION_STRING");

BlobServiceClient serviceClient = new BlobServiceClient(connectionString);
```

**Connection string format**:
```
DefaultEndpointsProtocol=https;
AccountName=mystorageaccount;
AccountKey=abc123...==;
EndpointSuffix=core.windows.net
```

⚠️ **Security warning**: Never hardcode connection strings. Use environment variables, Azure Key Vault, or App Configuration.

### Using Storage Shared Key

**Account name and key**:

```csharp
using Azure.Storage;

string accountName = "mystorageaccount";
string accountKey = Environment.GetEnvironmentVariable("AZURE_STORAGE_KEY");

var credential = new StorageSharedKeyCredential(accountName, accountKey);

var serviceClient = new BlobServiceClient(
    new Uri($"https://{accountName}.blob.core.windows.net"),
    credential
);
```

### Using SAS Token

**Shared Access Signature**:

```csharp
string accountName = "mystorageaccount";
string sasToken = "?sv=2021-06-08&ss=bfqt&srt=sco&sp=rwdlacupx&se=2024-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=...";

var serviceClient = new BlobServiceClient(
    new Uri($"https://{accountName}.blob.core.windows.net{sasToken}")
);
```

## Creating BlobContainerClient

### From BlobServiceClient (Recommended)

**Navigate from service to container**:

```csharp
public BlobContainerClient GetBlobContainerClient(
    BlobServiceClient blobServiceClient,
    string containerName)
{
    // Create container client using the service client object
    BlobContainerClient client = blobServiceClient.GetBlobContainerClient(containerName);
    return client;
}
```

**Advantages**:
- ✅ Reuses authentication from service client
- ✅ Maintains consistent configuration
- ✅ Supports account-level operations first

### Direct Container Client Creation

**Skip service client if not needed**:

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

**When to use**:
- Work is scoped to single container
- Don't need account-level operations
- Want to minimize object creation

### Using Connection String

**Direct container access**:

```csharp
string connectionString = Environment.GetEnvironmentVariable("AZURE_STORAGE_CONNECTION_STRING");
string containerName = "mycontainer";

BlobContainerClient containerClient = new BlobContainerClient(
    connectionString,
    containerName
);
```

## Creating BlobClient

### From Container Client (Recommended)

**Navigate from container to blob**:

```csharp
public BlobClient GetBlobClient(
    BlobContainerClient containerClient,
    string blobName)
{
    BlobClient client = containerClient.GetBlobClient(blobName);
    return client;
}
```

### From Service Client

**Navigate through hierarchy**:

```csharp
public BlobClient GetBlobClient(
    BlobServiceClient blobServiceClient,
    string containerName,
    string blobName)
{
    BlobClient client =
        blobServiceClient
            .GetBlobContainerClient(containerName)
            .GetBlobClient(blobName);
    
    return client;
}
```

### Direct Blob Client Creation

**Skip hierarchy for single blob operations**:

```csharp
string accountName = "mystorageaccount";
string containerName = "mycontainer";
string blobName = "myfile.txt";

var blobClient = new BlobClient(
    new Uri($"https://{accountName}.blob.core.windows.net/{containerName}/{blobName}"),
    new DefaultAzureCredential()
);
```

### Using Connection String

**Direct blob access**:

```csharp
string connectionString = Environment.GetEnvironmentVariable("AZURE_STORAGE_CONNECTION_STRING");
string containerName = "mycontainer";
string blobName = "myfile.txt";

BlobClient blobClient = new BlobClient(
    connectionString,
    containerName,
    blobName
);
```

## Azure RBAC Roles for Azure AD Authentication

### Required Roles

**Data plane access**:

| Role | Permissions | Use Case |
|------|-------------|----------|
| **Storage Blob Data Owner** | Full access including ACLs | Development, admin tasks |
| **Storage Blob Data Contributor** | Read, write, delete blobs | Application read/write |
| **Storage Blob Data Reader** | Read and list blobs | Read-only applications |

### Assign Role with Azure CLI

**Assign to user**:

```bash
# Get user principal name
userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
    --headers 'Content-Type=application/json' \
    --query userPrincipalName --output tsv)

# Get storage account resource ID
resourceID=$(az storage account show --name $accountName \
    --resource-group $resourceGroup \
    --query id --output tsv)

# Assign role
az role assignment create \
    --assignee $userPrincipal \
    --role "Storage Blob Data Owner" \
    --scope $resourceID
```

**Assign to managed identity**:

```bash
# Get managed identity principal ID
principalId=$(az identity show \
    --name myManagedIdentity \
    --resource-group $resourceGroup \
    --query principalId --output tsv)

# Assign role
az role assignment create \
    --assignee $principalId \
    --role "Storage Blob Data Contributor" \
    --scope $resourceID
```

### Assign Role with Azure Portal

1. Navigate to **Storage Account** in Azure Portal
2. Select **Access Control (IAM)** from left menu
3. Click **+ Add** → **Add role assignment**
4. Select role: **Storage Blob Data Contributor**
5. Select **Members** tab
6. Click **+ Select members**
7. Search for and select user or managed identity
8. Click **Review + assign**

## Complete Examples

### Example 1: Full Hierarchy with Azure AD

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;

public class BlobStorageService
{
    private readonly BlobServiceClient _serviceClient;

    public BlobStorageService(string accountName)
    {
        // Create service client with Azure AD
        _serviceClient = new BlobServiceClient(
            new Uri($"https://{accountName}.blob.core.windows.net"),
            new DefaultAzureCredential()
        );
    }

    public async Task<BlobClient> GetBlobClientAsync(
        string containerName,
        string blobName)
    {
        // Get container client
        var containerClient = _serviceClient.GetBlobContainerClient(containerName);

        // Ensure container exists
        await containerClient.CreateIfNotExistsAsync();

        // Get blob client
        return containerClient.GetBlobClient(blobName);
    }
}
```

### Example 2: Direct Container Access

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;

public class ContainerService
{
    private readonly BlobContainerClient _containerClient;

    public ContainerService(string accountName, string containerName)
    {
        // Direct container client creation
        _containerClient = new BlobContainerClient(
            new Uri($"https://{accountName}.blob.core.windows.net/{containerName}"),
            new DefaultAzureCredential()
        );
    }

    public async Task UploadFileAsync(string fileName, Stream content)
    {
        var blobClient = _containerClient.GetBlobClient(fileName);
        await blobClient.UploadAsync(content, overwrite: true);
    }

    public async Task<Stream> DownloadFileAsync(string fileName)
    {
        var blobClient = _containerClient.GetBlobClient(fileName);
        var response = await blobClient.DownloadAsync();
        return response.Value.Content;
    }
}
```

### Example 3: Configuration-Based Client

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;
using Microsoft.Extensions.Configuration;

public class ConfigurableBlobService
{
    private readonly BlobServiceClient _serviceClient;

    public ConfigurableBlobService(IConfiguration configuration)
    {
        var accountName = configuration["AzureStorage:AccountName"];
        var useConnectionString = configuration.GetValue<bool>("AzureStorage:UseConnectionString");

        if (useConnectionString)
        {
            var connectionString = configuration.GetConnectionString("AzureStorage");
            _serviceClient = new BlobServiceClient(connectionString);
        }
        else
        {
            // Use Azure AD
            _serviceClient = new BlobServiceClient(
                new Uri($"https://{accountName}.blob.core.windows.net"),
                new DefaultAzureCredential()
            );
        }
    }

    public BlobContainerClient GetContainerClient(string containerName)
    {
        return _serviceClient.GetBlobContainerClient(containerName);
    }
}
```

## Best Practices

### 1. Use DefaultAzureCredential

**Advantages**:
- ✅ Works locally and in Azure
- ✅ No secrets in code
- ✅ Automatic managed identity support
- ✅ Simplifies deployment

```csharp
// ✅ Good: DefaultAzureCredential
var credential = new DefaultAzureCredential();

// ❌ Bad: Hardcoded key
var credential = new StorageSharedKeyCredential("account", "hardcodedkey");
```

### 2. Create Clients Once

**Singleton pattern**:

```csharp
// ✅ Good: Create once, reuse
public class BlobService
{
    private static readonly BlobServiceClient _client = 
        new BlobServiceClient(endpoint, new DefaultAzureCredential());
}

// ❌ Bad: Create every time
public async Task UploadAsync()
{
    var client = new BlobServiceClient(...);  // Don't do this repeatedly
    await client.GetBlobContainerClient("test").GetBlobClient("file").UploadAsync(...);
}
```

### 3. Use Appropriate Hierarchy Level

```csharp
// ✅ Good: Direct to container for scoped work
if (onlyWorkingWithOneContainer)
{
    var containerClient = new BlobContainerClient(uri, credential);
}

// ✅ Good: Service client for multiple containers
if (workingWithMultipleContainers)
{
    var serviceClient = new BlobServiceClient(uri, credential);
    var container1 = serviceClient.GetBlobContainerClient("container1");
    var container2 = serviceClient.GetBlobContainerClient("container2");
}
```

### 4. Handle Authentication Errors

```csharp
using Azure;

try
{
    var client = new BlobServiceClient(uri, new DefaultAzureCredential());
    await client.GetPropertiesAsync();  // Test authentication
}
catch (RequestFailedException ex) when (ex.Status == 401)
{
    Console.WriteLine("Authentication failed. Check credentials.");
}
catch (RequestFailedException ex) when (ex.Status == 403)
{
    Console.WriteLine("Authorization failed. Check RBAC role assignments.");
}
```

## Critical Notes
- 💡 **Hierarchy** - ServiceClient → ContainerClient → BlobClient
- 🔒 **DefaultAzureCredential** - Recommended for Azure AD authentication
- ✅ **URI format** - https://{account}.blob.core.windows.net/{container}/{blob}
- ⚠️ **RBAC roles** - Storage Blob Data Owner/Contributor/Reader required
- 🎯 **Create once** - Clients are thread-safe, reuse throughout application
- 💡 **Authentication chain** - DefaultAzureCredential tries multiple methods automatically
- ✅ **Local development** - Uses Visual Studio, Azure CLI, or VS Code credentials
- 🔄 **Azure deployment** - Automatically uses Managed Identity
- ⚠️ **Connection strings** - Only for development, use environment variables
- 📊 **Direct creation** - Can skip hierarchy if working with single container/blob

## Exam Tips
- Client creation: BlobServiceClient for account-level, BlobContainerClient for container, BlobClient for blob
- DefaultAzureCredential: Recommended authentication, tries multiple methods in order
- URI construction: https://{accountName}.blob.core.windows.net for service, add /{container} for container, add /{blob} for blob
- Authentication methods: DefaultAzureCredential (Azure AD), ConnectionString (dev/test), StorageSharedKeyCredential (account key), SAS token
- RBAC roles: Storage Blob Data Owner (full access), Contributor (read/write), Reader (read-only)
- Role assignment: Use az role assignment create with --assignee and --scope
- From service client: Use GetBlobContainerClient(containerName) then GetBlobClient(blobName)
- Direct creation: Can create BlobContainerClient or BlobClient directly without service client
- Client reuse: Create once as singleton, reuse throughout application lifetime
- Authentication error handling: Catch RequestFailedException, check Status 401 (auth failed) or 403 (no permission)
- DefaultAzureCredentialOptions: Configure which authentication methods to try
- Local development: DefaultAzureCredential uses Visual Studio, Azure CLI, VS Code credentials
- Azure deployment: DefaultAzureCredential automatically uses Managed Identity

[Learn More](https://learn.microsoft.com/en-us/training/modules/work-azure-blob-storage/3-create-client-object)
