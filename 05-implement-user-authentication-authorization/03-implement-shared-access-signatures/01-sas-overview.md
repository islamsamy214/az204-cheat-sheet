# Shared Access Signatures (SAS) Overview

## Key Concepts
- **SAS** - Signed URI granting delegated access to Azure Storage resources
- **Token** - Query parameters including signature for authorization
- **Three types** - User delegation, Service, Account
- **Secure** - Time-limited, permission-scoped access without sharing account keys

## What is a Shared Access Signature?

A **shared access signature (SAS)** is a signed URI that points to one or more storage resources and includes a token containing query parameters. The token indicates how resources can be accessed by the client.

### Purpose

**Delegate access** to storage resources without sharing account keys:
- Grant specific permissions (read, write, delete, list)
- Time-limited access (start and expiry times)
- Secure authorization with cryptographic signature
- Revocable access through stored access policies

## Types of Shared Access Signatures

### Comparison Table

| Type | Secured With | Scope | Services | Best For |
|------|--------------|-------|----------|----------|
| **User Delegation SAS** | Microsoft Entra ID credentials | Service-level | Blob Storage, Data Lake Storage | **Most secure** - Recommended |
| **Service SAS** | Storage account key | Service-level | Blob, Queue, Table, Files | Service-specific access |
| **Account SAS** | Storage account key | Account-level | All storage services | Cross-service operations |

### 1. User Delegation SAS (Recommended)

**Most secure option** - Uses Microsoft Entra ID credentials:

```csharp
// Create user delegation SAS
BlobServiceClient blobServiceClient = new BlobServiceClient(
    new Uri("https://storageaccount.blob.core.windows.net"),
    new DefaultAzureCredential()
);

// Get user delegation key (requires Microsoft Entra authentication)
UserDelegationKey userDelegationKey = await blobServiceClient
    .GetUserDelegationKeyAsync(DateTimeOffset.UtcNow, DateTimeOffset.UtcNow.AddHours(1));

// Create SAS token
BlobSasBuilder sasBuilder = new BlobSasBuilder()
{
    BlobContainerName = "container-name",
    BlobName = "blob-name",
    Resource = "b",  // b = blob
    StartsOn = DateTimeOffset.UtcNow,
    ExpiresOn = DateTimeOffset.UtcNow.AddHours(1)
};

sasBuilder.SetPermissions(BlobSasPermissions.Read);

BlobUriBuilder uriBuilder = new BlobUriBuilder(blobClient.Uri)
{
    Sas = sasBuilder.ToSasQueryParameters(userDelegationKey, blobServiceClient.AccountName)
};

Uri sasUri = uriBuilder.ToUri();
```

**Benefits**:
- ✅ **No account key exposure** - Uses Microsoft Entra ID
- ✅ **Auditable** - User actions tracked
- ✅ **Revocable** - Disable user account to revoke all SAS
- ✅ **Best practice** - Recommended by Microsoft

**Applies to**:
- Blob Storage
- Data Lake Storage Gen2

### 2. Service SAS

**Service-level access** secured with storage account key:

```csharp
// Create service SAS for blob
BlobSasBuilder sasBuilder = new BlobSasBuilder()
{
    BlobContainerName = "documents",
    BlobName = "report.pdf",
    Resource = "b",
    StartsOn = DateTimeOffset.UtcNow,
    ExpiresOn = DateTimeOffset.UtcNow.AddHours(2)
};

sasBuilder.SetPermissions(BlobSasPermissions.Read);

// Sign with storage account key
BlobClient blobClient = new BlobClient(
    new Uri("https://storageaccount.blob.core.windows.net/documents/report.pdf"),
    new StorageSharedKeyCredential(accountName, accountKey)
);

Uri sasUri = blobClient.GenerateSasUri(sasBuilder);
```

**Use cases**:
- Single service access (Blob, Queue, Table, or Files)
- When Microsoft Entra ID not available
- Legacy applications

**Applies to**:
- Blob Storage
- Queue Storage
- Table Storage
- Azure Files

### 3. Account SAS

**Account-level access** to multiple services:

```csharp
// Create account SAS
AccountSasBuilder sasBuilder = new AccountSasBuilder()
{
    Services = AccountSasServices.Blobs | AccountSasServices.Queues,
    ResourceTypes = AccountSasResourceTypes.Service | AccountSasResourceTypes.Container | AccountSasResourceTypes.Object,
    ExpiresOn = DateTimeOffset.UtcNow.AddHours(1),
    Protocol = SasProtocol.Https
};

sasBuilder.SetPermissions(AccountSasPermissions.Read | AccountSasPermissions.List);

StorageSharedKeyCredential credential = new StorageSharedKeyCredential(accountName, accountKey);
string sasToken = sasBuilder.ToSasQueryParameters(credential).ToString();

string sasUrl = $"https://{accountName}.blob.core.windows.net?{sasToken}";
```

**Use cases**:
- Access to multiple services
- Operations across services
- Copy operations between storage accounts

**Applies to**:
- All storage services (Blob, Queue, Table, Files)

## How Shared Access Signatures Work

### SAS URI Structure

**Complete URI with SAS token**:

```
https://storageaccount.blob.core.windows.net/container/blob.jpg?sp=r&st=2020-01-20T11:42:32Z&se=2020-01-20T19:42:32Z&spr=https&sv=2019-02-02&sr=b&sig=SrW1HZ5Nb6MbRzTbXCaPm%2BJiSEn15tC91Y4umMPwVZs%3D
```

**Breaking it down**:

| Component | Value |
|-----------|-------|
| **Resource URI** | `https://storageaccount.blob.core.windows.net/container/blob.jpg` |
| **SAS Token** | `sp=r&st=...&sig=...` |

### SAS Token Components

| Parameter | Description | Example | Values |
|-----------|-------------|---------|--------|
| **sp** | **Permissions** | `sp=r` | `r` (read), `w` (write), `d` (delete), `l` (list), `a` (add), `c` (create) |
| **st** | **Start time** (UTC) | `st=2020-01-20T11:42:32Z` | ISO 8601 datetime |
| **se** | **Expiry time** (UTC) | `se=2020-01-20T19:42:32Z` | ISO 8601 datetime |
| **spr** | **Protocol** | `spr=https` | `https`, `http,https` |
| **sv** | **Storage API version** | `sv=2019-02-02` | Version string |
| **sr** | **Resource type** | `sr=b` | `b` (blob), `c` (container), `bs` (blob service) |
| **sig** | **Signature** | `sig=SrW1HZ5...` | Base64-encoded HMAC-SHA256 signature |

### Permissions Values

```
a = Add (Queue, Table)
c = Create (Blob, File)
d = Delete (Blob, Queue, Table, File)
l = List (Blob container, Queue, Table, File share)
r = Read (Blob, Queue, Table, File)
w = Write (Blob, Queue, Table, File)
```

**Combined permissions**:

```
sp=rl     # Read + List
sp=rw     # Read + Write
sp=acdlrw # All permissions
```

### Example: Decode a SAS Token

```
https://medicalrecords.blob.core.windows.net/patient-images/patient-116139-nq8z7f.jpg?sp=r&st=2020-01-20T11:42:32Z&se=2020-01-20T19:42:32Z&spr=https&sv=2019-02-02&sr=b&sig=SrW1HZ5Nb6MbRzTbXCaPm%2BJiSEn15tC91Y4umMPwVZs%3D
```

**Decoded parameters**:

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `sp` | `r` | **Read-only** access |
| `st` | `2020-01-20T11:42:32Z` | Valid **from** 11:42:32 UTC |
| `se` | `2020-01-20T19:42:32Z` | Valid **until** 19:42:32 UTC (8 hours) |
| `spr` | `https` | **HTTPS only** |
| `sv` | `2019-02-02` | Storage API version 2019-02-02 |
| `sr` | `b` | Access to **blob** |
| `sig` | `SrW1HZ5...` | Cryptographic signature |

## Creating SAS Tokens

### Using Azure Storage SDK (.NET)

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Sas;

// Create blob client
BlobClient blobClient = new BlobClient(
    new Uri("https://storageaccount.blob.core.windows.net/container/file.txt"),
    new StorageSharedKeyCredential(accountName, accountKey)
);

// Configure SAS
BlobSasBuilder sasBuilder = new BlobSasBuilder()
{
    BlobContainerName = "container",
    BlobName = "file.txt",
    Resource = "b",
    StartsOn = DateTimeOffset.UtcNow,
    ExpiresOn = DateTimeOffset.UtcNow.AddHours(1)
};

// Set permissions
sasBuilder.SetPermissions(BlobSasPermissions.Read);

// Generate SAS URI
Uri sasUri = blobClient.GenerateSasUri(sasBuilder);
Console.WriteLine($"SAS URI: {sasUri}");
```

### Using Azure CLI

```bash
# Generate blob SAS token
az storage blob generate-sas \
    --account-name mystorageaccount \
    --container-name mycontainer \
    --name myblob.txt \
    --permissions r \
    --expiry 2024-12-31T23:59:00Z \
    --https-only \
    --output tsv

# Generate container SAS token
az storage container generate-sas \
    --account-name mystorageaccount \
    --name mycontainer \
    --permissions rl \
    --expiry 2024-12-31T23:59:00Z \
    --https-only \
    --output tsv

# Generate account SAS token
az storage account generate-sas \
    --account-name mystorageaccount \
    --account-key <key> \
    --services b \
    --resource-types sco \
    --permissions rl \
    --expiry 2024-12-31T23:59:00Z \
    --https-only \
    --output tsv
```

### Using PowerShell

```powershell
# Connect to storage account
$context = New-AzStorageContext -StorageAccountName "mystorageaccount" -StorageAccountKey $accountKey

# Generate blob SAS
$sasToken = New-AzStorageBlobSASToken `
    -Context $context `
    -Container "mycontainer" `
    -Blob "myblob.txt" `
    -Permission r `
    -ExpiryTime (Get-Date).AddHours(2) `
    -Protocol HttpsOnly

# Full URI
$blobUri = "https://mystorageaccount.blob.core.windows.net/mycontainer/myblob.txt$sasToken"
Write-Host $blobUri
```

## Best Practices

### 1. Always Use HTTPS

```csharp
// ✅ Good: HTTPS only
sasBuilder.Protocol = SasProtocol.Https;

// ❌ Bad: Allows HTTP
sasBuilder.Protocol = SasProtocol.HttpsAndHttp;
```

**Why**: Prevents man-in-the-middle attacks and token interception.

### 2. Use User Delegation SAS When Possible

```csharp
// ✅ Best: User delegation SAS with Microsoft Entra ID
var userDelegationKey = await blobServiceClient.GetUserDelegationKeyAsync(
    DateTimeOffset.UtcNow,
    DateTimeOffset.UtcNow.AddHours(1)
);

var sas = sasBuilder.ToSasQueryParameters(userDelegationKey, accountName);

// ⚠️ OK but less secure: Service SAS with storage key
var credential = new StorageSharedKeyCredential(accountName, accountKey);
var sas = sasBuilder.ToSasQueryParameters(credential);
```

**Why**: Eliminates need to store account keys in code, provides better audit trail.

### 3. Minimum Required Permissions

```csharp
// ✅ Good: Only what's needed
sasBuilder.SetPermissions(BlobSasPermissions.Read);

// ❌ Bad: Excessive permissions
sasBuilder.SetPermissions(
    BlobSasPermissions.Read | 
    BlobSasPermissions.Write | 
    BlobSasPermissions.Delete
);
```

**Why**: Reduces impact if SAS is compromised.

### 4. Shortest Useful Expiration Time

```csharp
// ✅ Good: Short-lived (1 hour)
sasBuilder.ExpiresOn = DateTimeOffset.UtcNow.AddHours(1);

// ⚠️ Risky: Long-lived (1 year)
sasBuilder.ExpiresOn = DateTimeOffset.UtcNow.AddYears(1);
```

**Why**: Limits exposure window if token is compromised.

### 5. Use Stored Access Policies for Service SAS

```csharp
// ✅ Good: Use stored access policy (revocable)
BlobSasBuilder sasBuilder = new BlobSasBuilder()
{
    BlobContainerName = "container",
    Identifier = "policy-id"  // References stored policy
};

// ⚠️ Less flexible: Direct SAS (not revocable without key rotation)
BlobSasBuilder sasBuilder = new BlobSasBuilder()
{
    BlobContainerName = "container",
    ExpiresOn = DateTimeOffset.UtcNow.AddHours(1)
};
```

**Why**: Allows revoking SAS without regenerating storage account keys.

### 6. Consider Using Middle-Tier Service

**When NOT to use SAS directly**:
- High-security requirements
- Need for business logic validation
- Complex permission rules
- Sensitive data access

**Alternative**: Create middle-tier service that:
- Authenticates users
- Validates business rules
- Generates short-lived SAS tokens
- Logs access for audit

### 7. IP Address Restrictions (When Possible)

```csharp
// Restrict to specific IP
sasBuilder.IPRange = new SasIPRange(IPAddress.Parse("203.0.113.5"));

// Restrict to IP range
sasBuilder.IPRange = new SasIPRange(
    IPAddress.Parse("203.0.113.0"),
    IPAddress.Parse("203.0.113.255")
);
```

### 8. Monitor SAS Usage

**Use Azure Monitor** to track:
- SAS authentication failures
- Unusual access patterns
- Geographic anomalies
- Excessive read/write operations

```kusto
// Azure Monitor query for SAS usage
StorageBlobLogs
| where AuthenticationType == "SAS"
| where TimeGenerated > ago(1d)
| summarize count() by bin(TimeGenerated, 1h), StatusText
| render timechart
```

## Security Considerations

### Risks of Using SAS

| Risk | Impact | Mitigation |
|------|--------|-----------|
| **Token interception** | Unauthorized access | Use HTTPS only |
| **Token sharing** | Uncontrolled distribution | Short expiration times |
| **Excessive permissions** | Data modification/deletion | Minimum required permissions |
| **Long-lived tokens** | Extended exposure | Regular rotation |
| **Compromised account key** | All SAS invalid | Use user delegation SAS |

### When NOT to Use SAS

**Consider alternatives when**:
- Unacceptable risk of token exposure
- Need for real-time access revocation
- Complex authorization logic required
- High-security data (PHI, PCI, etc.)

**Alternatives**:
- **Managed Identity** - For Azure services
- **Microsoft Entra ID** - For user authentication
- **Middle-tier service** - For complex logic
- **Azure AD B2C** - For customer identity

## SAS vs Other Authentication Methods

| Method | Use Case | Pros | Cons |
|--------|----------|------|------|
| **SAS** | Time-limited delegated access | No credentials needed, fine-grained | Can be intercepted |
| **Storage Account Key** | Full administrative access | Complete control | High security risk |
| **Microsoft Entra ID** | User/app authentication | Most secure, revocable | Requires Azure AD setup |
| **Managed Identity** | Azure service-to-service | No credentials in code | Only for Azure services |
| **Anonymous Access** | Public data | Simple, no auth | No security |

## Critical Notes
- 💡 **Three types** - User delegation (best), Service, Account
- 🎯 **User delegation SAS** - Most secure, uses Microsoft Entra ID
- ✅ **Service SAS** - Single service access with storage key
- ⚠️ **Account SAS** - Cross-service access with storage key
- 🔄 **Components** - Permissions (sp), start time (st), expiry (se), signature (sig)
- 📊 **HTTPS required** - Always use HTTPS to prevent interception
- 💡 **Short-lived** - Set shortest useful expiration time
- ✅ **Minimum permissions** - Grant only what's required (r, w, d, l, a, c)
- ⚠️ **Revocation** - Use stored access policies for service SAS
- 🔒 **Best practice** - User delegation SAS > Service SAS > Account SAS

## Exam Tips
- SAS: Shared Access Signature - signed URI for delegated storage access
- Three types: User delegation (Microsoft Entra ID), Service (storage key), Account (storage key)
- User delegation SAS: Most secure, recommended, Blob/Data Lake only
- Service SAS: Single service access (Blob, Queue, Table, Files)
- Account SAS: Cross-service access, all storage services
- SAS token parameters: sp (permissions), st (start), se (expiry), sr (resource), sig (signature)
- Permissions: r (read), w (write), d (delete), l (list), a (add), c (create)
- Best practices: HTTPS only, user delegation when possible, minimum permissions, short expiration
- Security: Use HTTPS, shortest expiration, minimum permissions
- Stored access policies: Enable revocation without key rotation
- Protocol: Always use HTTPS (spr=https) to prevent interception
- Resource types: b (blob), c (container), bs (blob service)
- Signature: HMAC-SHA256 cryptographic signature
- Middle-tier service: Consider when SAS risk is unacceptable
- Copy operations: SAS required for cross-account blob/file copy
- When to use: Delegate access without sharing account keys
- Microsoft recommendation: User delegation SAS preferred over key-based SAS

[Learn More](https://learn.microsoft.com/en-us/training/modules/implement-shared-access-signatures/2-shared-access-signatures-overview)
