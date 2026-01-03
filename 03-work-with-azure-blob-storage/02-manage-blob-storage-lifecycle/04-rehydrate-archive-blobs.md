# Rehydrate Blob Data from Archive Tier

## What is Rehydration?

**Rehydration** is the process of moving a blob from the **offline Archive tier** to an **online tier** (Hot, Cool, or Cold) so it can be read or modified.

### Archive Tier Characteristics

| Characteristic | Detail |
|----------------|--------|
| **Status** | Offline (cannot be read or modified) |
| **Access** | Requires rehydration first |
| **Storage Cost** | Lowest |
| **Access Cost** | Highest |
| **Rehydration Time** | Several hours |

⚠️ **Critical**: Blobs in Archive tier are **offline** and must be rehydrated before accessing data.

---

## Rehydration Methods

There are **two options** for rehydrating archived blobs:

### Method Comparison

| Method | Operation | Source Blob | Recommended For |
|--------|-----------|-------------|-----------------|
| **Copy to Online Tier** | Copy Blob / Copy Blob From URL | Remains in Archive | Most scenarios (Microsoft recommended) |
| **Change Blob Tier** | Set Blob Tier | Moved to online tier | Simple tier change |

---

## Method 1: Copy Archived Blob to Online Tier (Recommended)

### Overview

**Microsoft Recommends This Method** ✅

**Process:**
1. Copy the archived blob to a **new destination blob** in Hot, Cool, or Cold tier
2. Source blob **remains unmodified** in Archive tier
3. Destination blob is immediately accessible

### Key Rules

⚠️ **Naming Requirements:**
- Must copy to a **different blob name** OR **different container**
- **Cannot overwrite** the source blob by copying to the same name

### Service Version Support

| Service Version | Scope | Support |
|-----------------|-------|---------|
| **< 2021-02-12** | Within same storage account only | Rehydrate within account |
| **≥ 2021-02-12** | Different storage account (same region) | Rehydrate across accounts |

💡 **Version 2021-02-12+**: Can rehydrate by copying to a **different storage account** as long as both accounts are in the **same region**.

### REST API: Copy Blob

```http
PUT https://<dest-account>.blob.core.windows.net/<dest-container>/<dest-blob>
x-ms-copy-source: https://<source-account>.blob.core.windows.net/<source-container>/<source-blob>
x-ms-access-tier: Hot
x-ms-rehydrate-priority: High
Authorization: <auth-header>
```

### Azure CLI Example

```bash
# Copy archived blob to Hot tier in same account
az storage blob copy start \
    --source-account-name <source-account> \
    --source-container <source-container> \
    --source-blob <source-blob> \
    --account-name <dest-account> \
    --destination-container <dest-container> \
    --destination-blob <dest-blob> \
    --tier Hot \
    --rehydrate-priority High \
    --auth-mode login

# Copy archived blob to different account (same region, v2021-02-12+)
az storage blob copy start \
    --source-account-name sourceaccount \
    --source-container archive \
    --source-blob data.bin \
    --account-name destaccount \
    --destination-container hot \
    --destination-blob data-restored.bin \
    --tier Hot \
    --rehydrate-priority Standard \
    --auth-mode login
```

### PowerShell Example

```powershell
# Get source blob URI
$sourceContext = New-AzStorageContext -StorageAccountName <source-account>
$sourceBlob = Get-AzStorageBlob `
    -Container <source-container> `
    -Blob <source-blob> `
    -Context $sourceContext

# Copy to Hot tier
$destContext = New-AzStorageContext -StorageAccountName <dest-account>
Start-AzStorageBlobCopy `
    -SrcBlob $sourceBlob.Name `
    -SrcContainer <source-container> `
    -Context $sourceContext `
    -DestContainer <dest-container> `
    -DestBlob <dest-blob> `
    -DestContext $destContext `
    -StandardBlobTier Hot `
    -RehydratePriority High
```

### .NET SDK Example

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;

// Source archived blob
var sourceClient = new BlobClient(connectionString, "archive-container", "archived-blob");

// Destination blob in Hot tier
var destClient = new BlobClient(connectionString, "hot-container", "restored-blob");

// Copy operation with rehydration
var options = new BlobCopyFromUriOptions
{
    AccessTier = AccessTier.Hot,
    RehydratePriority = RehydratePriority.High
};

await destClient.StartCopyFromUriAsync(sourceClient.Uri, options);
```

### Advantages of Copy Method

✅ **Source preserved**: Original archived blob remains intact
✅ **Immediate use**: New blob available in online tier immediately after copy
✅ **Safe**: No risk of data loss if something goes wrong
✅ **Cross-account**: Can copy to different storage account (v2021-02-12+)
✅ **Recommended**: Microsoft's recommended approach

### Disadvantages

❌ **Storage duplication**: Both source and destination blobs exist (2x storage cost temporarily)
❌ **Manual cleanup**: Need to delete source blob after verification

---

## Method 2: Change Blob's Access Tier

### Overview

**Process:**
1. Use **Set Blob Tier** operation to change tier from Archive to Hot/Cool/Cold
2. Blob tier changes **in place**
3. Once initiated, **cannot be canceled**
4. During rehydration, blob still shows as "archived"

### REST API: Set Blob Tier

```http
PUT https://<account>.blob.core.windows.net/<container>/<blob>?comp=tier
x-ms-access-tier: Hot
x-ms-rehydrate-priority: High
Authorization: <auth-header>
```

### Azure CLI Example

```bash
# Change blob tier from Archive to Hot
az storage blob set-tier \
    --account-name <account-name> \
    --container-name <container-name> \
    --name <blob-name> \
    --tier Hot \
    --rehydrate-priority High \
    --auth-mode login

# Change to Cool tier
az storage blob set-tier \
    --account-name <account-name> \
    --container-name <container-name> \
    --name <blob-name> \
    --tier Cool \
    --rehydrate-priority Standard \
    --auth-mode login
```

### PowerShell Example

```powershell
# Get blob
$blob = Get-AzStorageBlob `
    -Container <container-name> `
    -Blob <blob-name> `
    -Context $ctx

# Change tier to Hot with high priority
$blob.ICloudBlob.SetStandardBlobTier(
    [Microsoft.Azure.Storage.Blob.StandardBlobTier]::Hot,
    [Microsoft.Azure.Storage.Blob.RehydratePriority]::High
)
```

### .NET SDK Example

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;

var blobClient = new BlobClient(connectionString, containerName, blobName);

// Change tier to Hot with high priority
await blobClient.SetAccessTierAsync(
    AccessTier.Hot,
    rehydratePriority: RehydratePriority.High
);

// Check tier status
var properties = await blobClient.GetPropertiesAsync();
Console.WriteLine($"Current Tier: {properties.Value.AccessTier}");
Console.WriteLine($"Archive Status: {properties.Value.ArchiveStatus}");
Console.WriteLine($"Rehydrate Priority: {properties.Value.RehydratePriority}");
```

### Important Considerations

⚠️ **Cannot Cancel**: Once **Set Blob Tier** is initiated, it **cannot be canceled**.

⚠️ **Last Modified Time**: Changing tier does **NOT** update the blob's last modified time.

⚠️ **Lifecycle Policy Risk**: If a lifecycle management policy exists, the blob may be moved back to Archive after rehydration if the last modified time exceeds the policy threshold.

### Lifecycle Policy Scenario

```
Problem:
1. Blob last modified: 120 days ago
2. Lifecycle policy: Move to Archive after 90 days
3. Rehydrate blob to Hot tier
4. Last modified time still shows 120 days ago
5. Lifecycle policy runs → Moves blob BACK to Archive

Solution:
- Update last modified time after rehydration
- OR adjust lifecycle policy with daysAfterLastTierChangeGreaterThan
- OR exclude specific blobs from lifecycle policy
```

### Advantages of Set Tier Method

✅ **No duplication**: Blob remains in same location, no extra storage cost
✅ **Simpler**: Single operation to change tier
✅ **Direct**: Changes blob in place

### Disadvantages

❌ **Cannot cancel**: Once started, cannot be stopped
❌ **Lifecycle risk**: May be re-archived by lifecycle policies
❌ **Last modified unchanged**: Doesn't update last modified time
❌ **No fallback**: Original archived version is lost

---

## Rehydration Priority

When rehydrating a blob, you can set the **rehydration priority** via the `x-ms-rehydrate-priority` header.

### Priority Options

| Priority | Processing | Completion Time | Use Case |
|----------|-----------|-----------------|----------|
| **Standard** | Order received | Up to **15 hours** | Non-urgent, cost-sensitive |
| **High** | Prioritized over Standard | Under **1 hour** for objects < 10 GB | Urgent access needed |

### How to Set Priority

```bash
# Azure CLI - High priority
az storage blob copy start \
    --source-blob <source> \
    --destination-blob <dest> \
    --tier Hot \
    --rehydrate-priority High

# Azure CLI - Standard priority (default)
az storage blob set-tier \
    --name <blob-name> \
    --tier Cool \
    --rehydrate-priority Standard
```

### Check Rehydration Status

#### Azure CLI

```bash
# Check rehydration priority
az storage blob show \
    --account-name <account-name> \
    --container-name <container-name> \
    --name <blob-name> \
    --query "properties.rehydrationStatus" \
    --auth-mode login
```

#### PowerShell

```powershell
# Get blob properties
$blob = Get-AzStorageBlob `
    -Container <container> `
    -Blob <blob-name> `
    -Context $ctx

# Check archive status and rehydrate priority
$blob.ICloudBlob.Properties.RehydrationStatus
$blob.ICloudBlob.Properties.StandardBlobTier
```

#### .NET SDK

```csharp
// Get blob properties
var properties = await blobClient.GetPropertiesAsync();

// Check rehydration status
Console.WriteLine($"Archive Status: {properties.Value.ArchiveStatus}");
Console.WriteLine($"Rehydrate Priority: {properties.Value.RehydratePriority}");
Console.WriteLine($"Access Tier: {properties.Value.AccessTier}");

// Archive status values:
// - rehydrate-pending-to-hot
// - rehydrate-pending-to-cool
// - rehydrate-pending-to-cold
```

### Rehydration Status Values

| Status | Meaning |
|--------|---------|
| `rehydrate-pending-to-hot` | Rehydration to Hot tier in progress |
| `rehydrate-pending-to-cool` | Rehydration to Cool tier in progress |
| `rehydrate-pending-to-cold` | Rehydration to Cold tier in progress |
| `null` | Blob is not archived or rehydration complete |

---

## Rehydration Performance

### Timeline Comparison

| Priority | Size | Expected Time |
|----------|------|---------------|
| **High** | < 10 GB | < 1 hour |
| **High** | > 10 GB | Variable, prioritized |
| **Standard** | Any size | Up to 15 hours |

💡 **Recommendation**: Rehydrate **larger blobs** for optimal performance. Rehydrating many small blobs concurrently may require extra time.

### Cost Considerations

| Aspect | Standard Priority | High Priority |
|--------|-------------------|---------------|
| **Rehydration Cost** | Lower | Higher |
| **Processing Time** | Up to 15 hours | < 1 hour (< 10 GB) |
| **Best For** | Non-urgent, cost-sensitive | Time-critical scenarios |

---

## Complete Rehydration Workflow

### Workflow 1: Copy Method (Recommended)

```
1. Identify archived blob
   ↓
2. Start copy operation to online tier (Hot/Cool/Cold)
   - Set rehydration priority (Standard/High)
   ↓
3. Monitor copy status
   ↓
4. New blob available in online tier (source remains in Archive)
   ↓
5. Verify data integrity
   ↓
6. (Optional) Delete archived source blob
```

### Workflow 2: Set Tier Method

```
1. Identify archived blob
   ↓
2. Execute Set Blob Tier operation to online tier
   - Set rehydration priority (Standard/High)
   ↓
3. Monitor rehydration status (cannot cancel)
   ↓
4. Blob becomes available in online tier (in place)
   ↓
5. (Optional) Update last modified time to prevent lifecycle re-archival
```

---

## Best Practices

### Rehydration Strategy

✅ **DO:**
- Use **Copy method** for most scenarios (Microsoft recommended)
- Use **High priority** for time-critical rehydration (< 10 GB)
- Use **Standard priority** for cost optimization
- Monitor rehydration status
- Verify data after rehydration
- Plan rehydration in advance (allow time for Standard priority)

❌ **DON'T:**
- Use Set Tier method if lifecycle policies might re-archive
- Expect immediate access to archived blobs
- Rehydrate many small blobs concurrently (use fewer, larger blobs)
- Forget to account for rehydration costs

### Lifecycle Policy Protection

When using **Set Tier method**, protect against re-archival:

```json
{
  "rules": [
    {
      "name": "archive-with-cooldown",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 90,
              "daysAfterLastTierChangeGreaterThan": 7
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"]
        }
      }
    }
  ]
}
```

**Key**: `daysAfterLastTierChangeGreaterThan: 7` ensures blob stays in online tier for at least 7 days after rehydration.

---

## Cost Optimization

### Rehydration Costs

| Component | Cost Factor |
|-----------|-------------|
| **Rehydration operation** | Per-GB charge (higher for High priority) |
| **Data retrieval** | Per-GB egress charge |
| **Storage during rehydration** | Archive tier storage cost continues |
| **Destination storage** | Online tier storage cost (if copying) |

### Cost Optimization Tips

💰 **Optimize Costs:**
1. Use **Standard priority** when possible
2. Rehydrate only necessary blobs
3. Batch rehydration requests
4. Use **Copy method** to verify before deleting source
5. Delete archived source after verification
6. Plan rehydration to minimize urgent (High priority) requests

---

## Exam Tips

🎯 **Two rehydration methods**: Copy Blob (recommended), Set Blob Tier

🎯 **Copy method recommended**: Microsoft recommends Copy Blob for most scenarios

🎯 **Copy naming rule**: Must copy to different name OR different container (cannot overwrite source)

🎯 **Set Tier characteristics**: Cannot cancel, doesn't update last modified time

🎯 **Lifecycle risk**: Set Tier may cause re-archival if lifecycle policies based on last modified time

🎯 **Two priorities**: Standard (up to 15 hours), High (< 1 hour for < 10 GB)

🎯 **Rehydration time**: Several hours (not immediate)

🎯 **Archive status values**: `rehydrate-pending-to-hot/cool/cold`

🎯 **Cross-account support**: v2021-02-12+ allows copy to different account (same region)

🎯 **Check rehydration**: Use Get Blob Properties with `x-ms-rehydrate-priority` header

🎯 **Performance tip**: Rehydrate fewer, larger blobs (not many small blobs)

🎯 **REST API operations**: Copy Blob, Copy Blob From URL, Set Blob Tier

---

## Quick Reference Commands

### Check if Blob is Archived

```bash
# Azure CLI
az storage blob show \
    --account-name <account> \
    --container-name <container> \
    --name <blob> \
    --query "properties.blobTier" \
    --auth-mode login
```

### Rehydrate via Copy (Recommended)

```bash
# Azure CLI
az storage blob copy start \
    --source-container <source-container> \
    --source-blob <source-blob> \
    --destination-container <dest-container> \
    --destination-blob <dest-blob> \
    --account-name <account> \
    --tier Hot \
    --rehydrate-priority High \
    --auth-mode login
```

### Rehydrate via Set Tier

```bash
# Azure CLI
az storage blob set-tier \
    --account-name <account> \
    --container-name <container> \
    --name <blob> \
    --tier Hot \
    --rehydrate-priority High \
    --auth-mode login
```

### Monitor Rehydration Status

```bash
# Azure CLI
az storage blob show \
    --account-name <account> \
    --container-name <container> \
    --name <blob> \
    --query "properties.[blobTier,rehydrationStatus]" \
    --auth-mode login
```

---

## Decision Tree

```
Need to access archived blob?
│
├─ Need immediate preservation of source?
│  └─ YES → Use Copy Blob method ✅
│             - Source preserved in Archive
│             - New blob in online tier
│             - Verify before deleting source
│
└─ Simple in-place tier change acceptable?
   └─ YES → Check: Do you have lifecycle policies?
              │
              ├─ NO → Use Set Blob Tier ✅
              │        - Simpler, no duplication
              │
              └─ YES → Risk of re-archival!
                       → Add daysAfterLastTierChangeGreaterThan
                       → OR use Copy Blob method instead
```

---

## Additional Resources

- [Rehydrate an archived blob to an online tier](https://learn.microsoft.com/en-us/azure/storage/blobs/archive-rehydrate-to-online-tier)
- [Archive tier best practices](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-best-practices)

[Microsoft Learn - Rehydrate blob data from the archive tier](https://learn.microsoft.com/en-us/training/modules/manage-azure-blob-storage-lifecycle/5-rehydrate-blob-data)
