# Azure Blob Storage Overview

## What is Azure Blob Storage?

Azure Blob storage is Microsoft's **object storage solution for the cloud**, optimized for storing massive amounts of **unstructured data** (data that doesn't adhere to a particular data model or definition, such as text or binary data).

## Use Cases

| Use Case | Description |
|----------|-------------|
| **Browser Access** | Serving images or documents directly to a browser |
| **Distributed Access** | Storing files for distributed access across applications |
| **Streaming** | Streaming video and audio content |
| **Logging** | Writing to log files for diagnostics and analytics |
| **Backup/DR** | Storing data for backup, restore, disaster recovery, and archiving |
| **Analysis** | Storing data for analysis by on-premises or Azure-hosted services |

## Access Methods

- **HTTP/HTTPS**: Accessible from anywhere in the world
- **Azure Storage REST API**: Direct REST API calls
- **Azure PowerShell**: PowerShell cmdlets
- **Azure CLI**: Command-line interface
- **Client Libraries**: SDKs for .NET, Java, Python, JavaScript, etc.

## Storage Account

The **Azure Storage account** is the top-level container for all Azure Blob storage. It provides:
- **Unique namespace** for your Azure Storage data
- **Global accessibility** over HTTP or HTTPS
- **Base address** for all objects: `http://<account-name>.blob.core.windows.net`

---

## Types of Storage Accounts

### Performance Levels

| Performance Level | Description | Use Case |
|-------------------|-------------|----------|
| **Standard** | General-purpose v2 account (recommended) | Most scenarios using Azure Storage |
| **Premium** | Higher performance using solid-state drives | High transaction rates, low latency needs |

### Account Types

| Account Type | Supported Services | Redundancy Options | Description |
|--------------|-------------------|-------------------|-------------|
| **Standard general-purpose v2** | Blob Storage (including Data Lake Storage), Queue Storage, Table Storage, Azure Files | LRS, GRS, RA-GRS, ZRS, GZRS, RA-GZRS | Standard storage account type for blobs, file shares, queues, and tables. **Recommended for most scenarios**. |
| **Premium block blobs** | Blob Storage (including Data Lake Storage) | LRS, ZRS | Premium storage account for block blobs and append blobs. Recommended for **high transaction rates** or scenarios using **smaller objects** or requiring **consistently low storage latency**. |
| **Premium file shares** | Azure Files | LRS, ZRS | Premium storage account for file shares only. Recommended for **enterprise or high-performance scale applications**. |
| **Premium page blobs** | Page blobs only | LRS, ZRS | Premium storage account for page blobs only (used for VM disks). |

### Redundancy Options

| Option | Full Name | Description |
|--------|-----------|-------------|
| **LRS** | Locally Redundant Storage | Replicates data 3 times within a single data center |
| **ZRS** | Zone-Redundant Storage | Replicates data across 3 availability zones |
| **GRS** | Geo-Redundant Storage | Replicates data to a secondary region |
| **RA-GRS** | Read-Access Geo-Redundant Storage | GRS with read access to secondary region |
| **GZRS** | Geo-Zone-Redundant Storage | Combines ZRS and GRS |
| **RA-GZRS** | Read-Access Geo-Zone-Redundant Storage | GZRS with read access to secondary region |

---

## Access Tiers for Block Blob Data

Azure Storage provides different access tiers optimized for particular usage patterns. Each tier balances storage costs vs. access costs.

### Access Tier Comparison

| Tier | Optimization | Minimum Storage Duration | Storage Cost | Access Cost | Use Case |
|------|--------------|-------------------------|--------------|-------------|----------|
| **Hot** | Frequent access of objects | None | Highest | Lowest | Frequently accessed data, **default tier** for new accounts |
| **Cool** | Infrequently accessed data | 30 days | Lower | Higher | Storing large amounts of data accessed infrequently |
| **Cold** | Rarely accessed data | 90 days | Lower than Cool | Higher than Cool | Data accessed rarely but needs quick retrieval |
| **Archive** | Long-term archival | 180 days | Lowest | Highest | Data that can tolerate several hours of retrieval latency |

### Tier Characteristics

#### Hot Tier
- **Highest storage costs**, but **lowest access costs**
- Optimized for **frequent access**
- **Default tier** for new storage accounts
- Data accessed regularly

#### Cool Tier
- **Lower storage costs**, **higher access costs** compared to Hot
- Optimized for data stored for **minimum 30 days**
- Ideal for short-term backup and disaster recovery

#### Cold Tier
- **Lower storage costs** than Cool, **higher access costs** than Cool
- Optimized for data stored for **minimum 90 days**
- Balance between Cool and Archive tiers

#### Archive Tier
- **Available only for individual block blobs** (not at account/container level)
- **Lowest storage cost**, **highest access cost**
- Data must remain for **minimum 180 days**
- Can tolerate **several hours of retrieval latency** (rehydration required)
- Most cost-effective for **long-term archival**

### Tier Switching

💡 **Flexibility**: You can switch between access tiers at any time if usage patterns change.

⚠️ **Early Deletion Fees**: Deleting or moving data before minimum storage duration incurs early deletion charges.

---

## Key Concepts

### Unstructured Data
Data that doesn't adhere to a particular data model or definition, such as:
- Text files
- Binary data
- Images, videos, audio
- Log files
- Backup files

### Storage Account Namespace
Each storage account has a unique namespace:
```
http://<account-name>.blob.core.windows.net
```

All objects in the account are addressable via this base URL.

---

## Best Practices

✅ **Use Standard general-purpose v2** for most scenarios
✅ **Choose Premium block blobs** for high-performance requirements
✅ **Select appropriate access tier** based on data access patterns
✅ **Use Hot tier** for frequently accessed data
✅ **Use Cool/Cold tier** for infrequently accessed data (30/90 days minimum)
✅ **Use Archive tier** for long-term archival (180 days minimum)
✅ **Consider tier switching** as usage patterns change

---

## Exam Tips

🎯 **Know the account types**: Standard general-purpose v2 (most common), Premium block blobs (high performance)

🎯 **Understand access tiers**: Hot (frequent), Cool (30 days), Cold (90 days), Archive (180 days)

🎯 **Remember minimum durations**: Cool = 30 days, Cold = 90 days, Archive = 180 days

🎯 **Archive tier limitations**: Only for individual block blobs, requires rehydration (hours)

🎯 **Cost tradeoff**: Storage cost ↓ as tier gets colder, Access cost ↑ as tier gets colder

🎯 **Redundancy options**: LRS (local), ZRS (zonal), GRS (geo), RA-GRS (geo + read)

🎯 **Premium vs Standard**: Premium uses SSDs, supports LRS/ZRS only, higher performance

---

## Quick Reference Commands

### Create Storage Account (Azure CLI)
```bash
# Standard general-purpose v2
az storage account create \
  --name <account-name> \
  --resource-group <resource-group> \
  --location <location> \
  --sku Standard_LRS \
  --kind StorageV2

# Premium block blobs
az storage account create \
  --name <account-name> \
  --resource-group <resource-group> \
  --location <location> \
  --sku Premium_LRS \
  --kind BlockBlobStorage
```

### Create Storage Account (PowerShell)
```powershell
# Standard general-purpose v2
New-AzStorageAccount `
  -ResourceGroupName <resource-group> `
  -Name <account-name> `
  -Location <location> `
  -SkuName Standard_LRS `
  -Kind StorageV2

# Premium block blobs
New-AzStorageAccount `
  -ResourceGroupName <resource-group> `
  -Name <account-name> `
  -Location <location> `
  -SkuName Premium_LRS `
  -Kind BlockBlobStorage
```

### SKU Names

| SKU Name | Description |
|----------|-------------|
| `Standard_LRS` | Standard locally redundant storage |
| `Standard_GRS` | Standard geo-redundant storage |
| `Standard_RAGRS` | Standard read-access geo-redundant storage |
| `Standard_ZRS` | Standard zone-redundant storage |
| `Standard_GZRS` | Standard geo-zone-redundant storage |
| `Standard_RAGZRS` | Standard read-access geo-zone-redundant storage |
| `Premium_LRS` | Premium locally redundant storage |
| `Premium_ZRS` | Premium zone-redundant storage |

---

## Additional Resources

- [Azure Blob Storage documentation](https://learn.microsoft.com/en-us/azure/storage/blobs/)
- [Storage account overview](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-overview)
- [Access tiers for blob data](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview)

[Microsoft Learn - Explore Azure Blob storage](https://learn.microsoft.com/en-us/training/modules/explore-azure-blob-storage/2-blob-storage-overview)
