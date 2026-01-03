# Stored Access Policies

## Key Concepts
- **Stored access policy** - Server-side policy for service-level SAS
- **Revocable** - Change or revoke SAS without regenerating storage keys
- **Centralized control** - Manage SAS expiry and permissions from one place
- **Maximum 5 policies** - Per container, queue, table, or file share

## What is a Stored Access Policy?

A **stored access policy** provides an additional level of control over **service-level shared access signatures (SAS)** on the server side.

### Purpose

**Group and manage SAS tokens** with:
- Centralized expiry time management
- Permission changes for all associated SAS
- Revocation without regenerating storage account keys
- Start time control for scheduled access

### Supported Resources

| Resource Type | Stored Policy Support |
|--------------|----------------------|
| **Blob containers** | ✅ Yes |
| **File shares** | ✅ Yes |
| **Queues** | ✅ Yes |
| **Tables** | ✅ Yes |
| **Individual blobs/files** | ❌ No (use container/share policy) |
| **Account SAS** | ❌ No (only service SAS) |

## SAS Parameters: Policy vs Token

### Parameter Distribution

You can specify SAS parameters in **three ways**:

| Option | Start Time | Expiry Time | Permissions | Example Use Case |
|--------|-----------|-------------|-------------|------------------|
| **All on SAS token** | Token | Token | Token | One-time, ad-hoc access |
| **All in policy** | Policy | Policy | Policy | Managed, revocable access |
| **Mixed** | Token | Policy | Policy | Scheduled start with managed expiry |

⚠️ **Rule**: Cannot specify the same parameter in both SAS token and stored access policy.

### Example: All Parameters in Policy

```csharp
// Create stored access policy
BlobSignedIdentifier identifier = new BlobSignedIdentifier
{
    Id = "read-policy",
    AccessPolicy = new BlobAccessPolicy
    {
        StartsOn = DateTimeOffset.UtcNow,
        ExpiresOn = DateTimeOffset.UtcNow.AddDays(7),
        Permissions = "r"  // Read-only
    }
};

await blobContainer.SetAccessPolicyAsync(permissions: new[] { identifier });

// Generate SAS referencing policy (no expiry/permissions in token)
BlobSasBuilder sasBuilder = new BlobSasBuilder()
{
    BlobContainerName = "documents",
    BlobName = "report.pdf",
    Resource = "b",
    Identifier = "read-policy"  // Reference policy by ID
};

BlobClient blobClient = containerClient.GetBlobClient("report.pdf");
Uri sasUri = blobClient.GenerateSasUri(sasBuilder);
```

**SAS token** will look like:

```
https://storageaccount.blob.core.windows.net/documents/report.pdf?si=read-policy&sig=...
```

Notice: No `sp`, `st`, `se` parameters (they're in the policy).

### Example: Mixed Parameters

```csharp
// Policy has expiry and permissions
BlobSignedIdentifier identifier = new BlobSignedIdentifier
{
    Id = "standard-access",
    AccessPolicy = new BlobAccessPolicy
    {
        ExpiresOn = DateTimeOffset.UtcNow.AddDays(30),
        Permissions = "rl"  // Read + List
    }
};

await blobContainer.SetAccessPolicyAsync(permissions: new[] { identifier });

// SAS token specifies start time (not in policy)
BlobSasBuilder sasBuilder = new BlobSasBuilder()
{
    BlobContainerName = "documents",
    BlobName = "report.pdf",
    Resource = "b",
    Identifier = "standard-access",
    StartsOn = DateTimeOffset.UtcNow.AddHours(1)  // Delayed start
};

Uri sasUri = blobClient.GenerateSasUri(sasBuilder);
```

## Creating Stored Access Policies

### Using C# (.NET)

**Blob container policy**:

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;

// Get container client
BlobContainerClient containerClient = new BlobContainerClient(
    connectionString,
    "my-container"
);

// Create policy
BlobSignedIdentifier identifier = new BlobSignedIdentifier
{
    Id = "my-policy-id",  // Unique identifier (up to 64 chars)
    AccessPolicy = new BlobAccessPolicy
    {
        StartsOn = DateTimeOffset.UtcNow,
        ExpiresOn = DateTimeOffset.UtcNow.AddHours(1),
        Permissions = "rw"  // Read + Write
    }
};

// Set access policy on container
await containerClient.SetAccessPolicyAsync(
    permissions: new BlobSignedIdentifier[] { identifier }
);

Console.WriteLine("Stored access policy created");
```

**Queue policy**:

```csharp
using Azure.Storage.Queues;
using Azure.Storage.Queues.Models;

QueueClient queueClient = new QueueClient(connectionString, "my-queue");

QueueSignedIdentifier identifier = new QueueSignedIdentifier
{
    Id = "queue-policy",
    AccessPolicy = new QueueAccessPolicy
    {
        StartsOn = DateTimeOffset.UtcNow,
        ExpiresOn = DateTimeOffset.UtcNow.AddDays(1),
        Permissions = "raup"  // Read, Add, Update, Process
    }
};

await queueClient.SetAccessPolicyAsync(new[] { identifier });
```

**Table policy**:

```csharp
using Azure.Data.Tables;
using Azure.Data.Tables.Models;

TableClient tableClient = new TableClient(connectionString, "MyTable");

TableSignedIdentifier identifier = new TableSignedIdentifier("table-policy")
{
    AccessPolicy = new TableAccessPolicy
    {
        StartsOn = DateTimeOffset.UtcNow,
        ExpiresOn = DateTimeOffset.UtcNow.AddDays(7),
        Permissions = "raud"  // Read, Add, Update, Delete
    }
};

await tableClient.SetAccessPolicyAsync(new[] { identifier });
```

**File share policy**:

```csharp
using Azure.Storage.Files.Shares;
using Azure.Storage.Files.Shares.Models;

ShareClient shareClient = new ShareClient(connectionString, "my-share");

ShareSignedIdentifier identifier = new ShareSignedIdentifier
{
    Id = "share-policy",
    AccessPolicy = new ShareAccessPolicy
    {
        StartsOn = DateTimeOffset.UtcNow,
        ExpiresOn = DateTimeOffset.UtcNow.AddMonths(1),
        Permissions = "rcwdl"  // Read, Create, Write, Delete, List
    }
};

await shareClient.SetAccessPolicyAsync(new[] { identifier });
```

### Using Azure CLI

**Create blob container policy**:

```bash
az storage container policy create \
    --name my-policy-id \
    --container-name my-container \
    --account-name mystorageaccount \
    --account-key $ACCOUNT_KEY \
    --start 2024-01-01T00:00:00Z \
    --expiry 2024-12-31T23:59:59Z \
    --permissions rw
```

**Create queue policy**:

```bash
az storage queue policy create \
    --name queue-policy \
    --queue-name my-queue \
    --account-name mystorageaccount \
    --account-key $ACCOUNT_KEY \
    --start 2024-01-01T00:00:00Z \
    --expiry 2024-12-31T23:59:59Z \
    --permissions raup
```

**Create table policy**:

```bash
az storage table policy create \
    --name table-policy \
    --table-name MyTable \
    --account-name mystorageaccount \
    --account-key $ACCOUNT_KEY \
    --start 2024-01-01T00:00:00Z \
    --expiry 2024-12-31T23:59:59Z \
    --permissions raud
```

**Create file share policy**:

```bash
az storage share policy create \
    --name share-policy \
    --share-name my-share \
    --account-name mystorageaccount \
    --account-key $ACCOUNT_KEY \
    --start 2024-01-01T00:00:00Z \
    --expiry 2024-12-31T23:59:59Z \
    --permissions rcwdl
```

### Using PowerShell

```powershell
# Get storage context
$context = New-AzStorageContext -StorageAccountName "mystorageaccount" -StorageAccountKey $accountKey

# Create policy
$policy = New-AzStorageContainerStoredAccessPolicy `
    -Context $context `
    -Container "my-container" `
    -Policy "my-policy" `
    -Permission rw `
    -StartTime (Get-Date) `
    -ExpiryTime (Get-Date).AddDays(7)

Write-Host "Policy created: $($policy.Policy)"
```

### Using Azure Portal

1. Navigate to storage account
2. Select **Containers** (or Queues, Tables, File shares)
3. Select container
4. Click **Access policy** (or **Stored access policies**)
5. Click **+ Add policy**
6. Fill in:
   - **Identifier**: Policy name (e.g., `read-policy`)
   - **Permissions**: Select permissions (Read, Write, Delete, List, etc.)
   - **Start time**: When access begins (optional)
   - **Expiry time**: When access ends
7. Click **OK**
8. Click **Save**

## Using Stored Access Policies with SAS

### Generate SAS with Policy Reference

```csharp
// After creating stored access policy, generate SAS
BlobSasBuilder sasBuilder = new BlobSasBuilder()
{
    BlobContainerName = "my-container",
    BlobName = "document.pdf",
    Resource = "b",
    Identifier = "my-policy-id"  // Reference stored policy
};

// No need to specify permissions, start time, or expiry
// They come from the stored access policy

BlobClient blobClient = containerClient.GetBlobClient("document.pdf");
Uri sasUri = blobClient.GenerateSasUri(sasBuilder);

Console.WriteLine($"SAS URI: {sasUri}");
```

### Azure CLI

```bash
# Generate SAS using stored policy
az storage blob generate-sas \
    --account-name mystorageaccount \
    --container-name my-container \
    --name document.pdf \
    --policy-name my-policy-id \
    --output tsv
```

## Modifying Stored Access Policies

### Update Policy Parameters

```csharp
// Get existing policies
var existingPolicies = await containerClient.GetAccessPolicyAsync();
var policies = existingPolicies.Value.SignedIdentifiers.ToList();

// Find and modify policy
var policyToUpdate = policies.FirstOrDefault(p => p.Id == "my-policy-id");
if (policyToUpdate != null)
{
    // Change permissions from read-write to read-only
    policyToUpdate.AccessPolicy.Permissions = "r";
    
    // Extend expiry by 7 days
    policyToUpdate.AccessPolicy.ExpiresOn = DateTimeOffset.UtcNow.AddDays(7);
}

// Save updated policies
await containerClient.SetAccessPolicyAsync(permissions: policies);

Console.WriteLine("Policy updated - all associated SAS now affected");
```

### Azure CLI

```bash
# Update existing policy
az storage container policy update \
    --name my-policy-id \
    --container-name my-container \
    --account-name mystorageaccount \
    --account-key $ACCOUNT_KEY \
    --expiry 2025-12-31T23:59:59Z \
    --permissions r
```

**Effect**: All SAS tokens associated with this policy are immediately affected.

## Revoking Access

### Method 1: Delete Policy

```csharp
// Get existing policies
var existingPolicies = await containerClient.GetAccessPolicyAsync();
var policies = existingPolicies.Value.SignedIdentifiers.ToList();

// Remove policy
policies.RemoveAll(p => p.Id == "my-policy-id");

// Save updated list (without deleted policy)
await containerClient.SetAccessPolicyAsync(permissions: policies);

Console.WriteLine("Policy deleted - all associated SAS immediately revoked");
```

**Azure CLI**:

```bash
az storage container policy delete \
    --name my-policy-id \
    --container-name my-container \
    --account-name mystorageaccount \
    --account-key $ACCOUNT_KEY
```

### Method 2: Change Policy Identifier

```csharp
// Rename policy (breaks association with existing SAS)
var existingPolicies = await containerClient.GetAccessPolicyAsync();
var policies = existingPolicies.Value.SignedIdentifiers.ToList();

var policy = policies.FirstOrDefault(p => p.Id == "my-policy-id");
if (policy != null)
{
    // Remove old policy
    policies.Remove(policy);
    
    // Add with new identifier
    policy.Id = "new-policy-id";
    policies.Add(policy);
}

await containerClient.SetAccessPolicyAsync(permissions: policies);

Console.WriteLine("Policy ID changed - existing SAS tokens no longer work");
```

### Method 3: Change Expiry to Past

```csharp
// Set expiry time in the past
var existingPolicies = await containerClient.GetAccessPolicyAsync();
var policies = existingPolicies.Value.SignedIdentifiers.ToList();

var policy = policies.FirstOrDefault(p => p.Id == "my-policy-id");
if (policy != null)
{
    policy.AccessPolicy.ExpiresOn = DateTimeOffset.UtcNow.AddMinutes(-1);
}

await containerClient.SetAccessPolicyAsync(permissions: policies);

Console.WriteLine("Policy expired - all associated SAS immediately invalid");
```

### Comparison of Revocation Methods

| Method | Effect | Recovery | Use When |
|--------|--------|----------|----------|
| **Delete policy** | Immediate revocation | Cannot recover | Permanent revocation |
| **Change identifier** | Breaks association | Can create new with same params | Want to rotate access |
| **Set past expiry** | Immediate expiration | Can extend later | Temporary revocation |

## Managing Multiple Policies

### List All Policies

```csharp
var accessPolicy = await containerClient.GetAccessPolicyAsync();

foreach (var policy in accessPolicy.Value.SignedIdentifiers)
{
    Console.WriteLine($"Policy ID: {policy.Id}");
    Console.WriteLine($"  Permissions: {policy.AccessPolicy.Permissions}");
    Console.WriteLine($"  Starts: {policy.AccessPolicy.StartsOn}");
    Console.WriteLine($"  Expires: {policy.AccessPolicy.ExpiresOn}");
}
```

**Azure CLI**:

```bash
# List container policies
az storage container policy list \
    --container-name my-container \
    --account-name mystorageaccount \
    --account-key $ACCOUNT_KEY
```

### Create Multiple Policies

```csharp
var policies = new List<BlobSignedIdentifier>
{
    new BlobSignedIdentifier
    {
        Id = "read-only",
        AccessPolicy = new BlobAccessPolicy
        {
            ExpiresOn = DateTimeOffset.UtcNow.AddDays(7),
            Permissions = "r"
        }
    },
    new BlobSignedIdentifier
    {
        Id = "read-write",
        AccessPolicy = new BlobAccessPolicy
        {
            ExpiresOn = DateTimeOffset.UtcNow.AddDays(7),
            Permissions = "rw"
        }
    },
    new BlobSignedIdentifier
    {
        Id = "full-access",
        AccessPolicy = new BlobAccessPolicy
        {
            ExpiresOn = DateTimeOffset.UtcNow.AddDays(1),
            Permissions = "rwdl"
        }
    }
};

await containerClient.SetAccessPolicyAsync(permissions: policies);
```

⚠️ **Limit**: Maximum **5 policies** per container, queue, table, or file share.

### Remove All Policies

```csharp
// Pass empty array to remove all policies
await containerClient.SetAccessPolicyAsync(permissions: Array.Empty<BlobSignedIdentifier>());

Console.WriteLine("All policies removed");
```

**Azure CLI**:

```bash
# Set empty policy (removes all)
az storage container policy list \
    --container-name my-container \
    --account-name mystorageaccount \
    --account-key $ACCOUNT_KEY \
    --query "[]" \
    | az storage container set-metadata \
        --container-name my-container \
        --account-name mystorageaccount
```

## Best Practices

### 1. Use Stored Policies for Long-Lived SAS

```csharp
// ✅ Good: Stored policy (revocable)
BlobSasBuilder sasBuilder = new BlobSasBuilder()
{
    BlobContainerName = "data",
    BlobName = "file.csv",
    Resource = "b",
    Identifier = "long-term-access"  // References policy
};

// ❌ Bad: Direct SAS for long-lived access (cannot revoke)
BlobSasBuilder sasBuilder = new BlobSasBuilder()
{
    BlobContainerName = "data",
    BlobName = "file.csv",
    Resource = "b",
    ExpiresOn = DateTimeOffset.UtcNow.AddYears(1),  // 1 year, not revocable!
    Permissions = "rw"
};
```

### 2. Create Policy Before Distributing SAS

```csharp
// ✅ Good: Policy exists first
await CreateStoredAccessPolicy("my-policy");
var sasUri = GenerateSasWithPolicy("my-policy");
await DistributeToClients(sasUri);

// ❌ Bad: SAS before policy
var sasUri = GenerateSasWithPolicy("non-existent-policy");  // Will fail
await CreateStoredAccessPolicy("my-policy");
```

⏱️ **Note**: Policy may take up to **30 seconds** to take effect after creation.

### 3. Use Descriptive Policy Names

```csharp
// ✅ Good: Descriptive
"read-only-reports-2024"
"partner-upload-access"
"temp-download-q1"

// ❌ Bad: Generic
"policy1"
"temp"
"access"
```

### 4. Monitor Policy Expiration

```csharp
// Check expiring policies
var policies = await containerClient.GetAccessPolicyAsync();

foreach (var policy in policies.Value.SignedIdentifiers)
{
    var daysUntilExpiry = (policy.AccessPolicy.ExpiresOn - DateTimeOffset.UtcNow).TotalDays;
    
    if (daysUntilExpiry < 7)
    {
        Console.WriteLine($"⚠️ Policy '{policy.Id}' expires in {daysUntilExpiry:F1} days");
    }
}
```

### 5. Rotate Policies Regularly

```csharp
// Monthly policy rotation
public async Task RotateMonthlyPolicy()
{
    var currentMonth = DateTime.UtcNow.ToString("yyyy-MM");
    var newPolicyId = $"access-{currentMonth}";
    
    // Create new policy
    var newPolicy = new BlobSignedIdentifier
    {
        Id = newPolicyId,
        AccessPolicy = new BlobAccessPolicy
        {
            ExpiresOn = DateTimeOffset.UtcNow.AddMonths(1),
            Permissions = "r"
        }
    };
    
    var policies = new[] { newPolicy };
    await containerClient.SetAccessPolicyAsync(permissions: policies);
    
    // Issue new SAS tokens referencing new policy
    Console.WriteLine($"New policy created: {newPolicyId}");
}
```

## Limitations and Considerations

### Activation Delay

⏱️ **30-second delay**: Policy may take up to 30 seconds to activate.

```csharp
// Create policy
await containerClient.SetAccessPolicyAsync(permissions: new[] { identifier });

// Wait for policy to activate
await Task.Delay(TimeSpan.FromSeconds(35));

// Now safe to generate SAS
var sasUri = blobClient.GenerateSasUri(sasBuilder);
```

**During activation period**: SAS requests may fail with **403 Forbidden**.

### Table Entity Range Restrictions

⚠️ **Cannot specify in stored policy**:
- `startpk` (start partition key)
- `startrk` (start row key)
- `endpk` (end partition key)
- `endrk` (end row key)

These must be specified on the SAS token itself.

### Policy Limit

Maximum **5 stored access policies** per:
- Blob container
- File share
- Queue
- Table

```csharp
// ❌ Will fail: 6th policy
var policies = new List<BlobSignedIdentifier>();
for (int i = 1; i <= 6; i++)
{
    policies.Add(new BlobSignedIdentifier { Id = $"policy{i}", AccessPolicy = new BlobAccessPolicy() });
}

await containerClient.SetAccessPolicyAsync(permissions: policies);  // ERROR
```

### User Delegation SAS Not Supported

❌ Stored access policies only work with **service-level SAS**, not **user delegation SAS**.

```csharp
// ✅ Works: Service SAS
var sasBuilder = new BlobSasBuilder { Identifier = "policy-id" };

// ❌ Does not work: User delegation SAS cannot use policies
var userDelegationKey = await blobServiceClient.GetUserDelegationKeyAsync(...);
// Cannot reference stored policy with user delegation key
```

## Critical Notes
- 💡 **Stored access policy** - Server-side policy for service-level SAS
- 🎯 **Benefits** - Revoke SAS without regenerating keys, centralized control
- ✅ **Supports** - Blob containers, file shares, queues, tables
- ⚠️ **Limitations** - Max 5 policies per resource, only service SAS (not user delegation)
- 🔄 **Parameters** - Can be in policy, token, or split between both
- 📊 **Revocation methods** - Delete policy, change identifier, set past expiry
- 💡 **Effect** - Immediate for all SAS using that policy
- ✅ **Activation delay** - Up to 30 seconds for new policies
- ⚠️ **Best practice** - Use policies for long-lived SAS (revocable)
- 🔒 **Cannot specify** - Same parameter in both policy and token

## Exam Tips
- Stored access policy: Server-side policy providing extra control over service SAS
- Supports: Blob containers, file shares, queues, tables
- Does not support: Individual blobs/files, account SAS, user delegation SAS
- Maximum: 5 stored access policies per container/share/queue/table
- Parameters: Cannot specify same parameter in both policy and SAS token
- Revocation: Delete policy, change identifier, or set expiry to past
- Immediate effect: Modifying/deleting policy affects all associated SAS instantly
- Activation delay: May take up to 30 seconds after creation/modification
- SetAccessPolicy: Method to create/modify stored access policies
- Identifier: Unique ID for policy (up to 64 characters)
- Permissions: Specify in policy (r, w, d, l, a, c) or SAS token, not both
- Table restrictions: Cannot specify startpk, startrk, endpk, endrk in policy
- Best practice: Use stored policies for long-lived SAS (enables revocation)
- Revoke without keys: Change/delete policy instead of regenerating storage keys
- CLI: az storage container policy create/update/delete/list
- C#: SetAccessPolicyAsync, GetAccessPolicyAsync, BlobSignedIdentifier

[Learn More](https://learn.microsoft.com/en-us/training/modules/implement-shared-access-signatures/4-stored-access-policies)
