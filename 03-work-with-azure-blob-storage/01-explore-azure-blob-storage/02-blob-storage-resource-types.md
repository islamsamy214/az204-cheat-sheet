# Azure Blob Storage Resource Types

## Resource Hierarchy

Blob storage offers **three types of resources** organized in a hierarchical structure:

```
Storage Account (mystorageaccount)
└── Container (mycontainer)
    └── Blob (myblob)
        ├── Block Blob
        ├── Append Blob
        └── Page Blob
```

---

## 1. Storage Account

### Overview
A storage account provides a **unique namespace in Azure** for your data. Every object stored in Azure Storage has an address that includes your unique account name.

### Naming and Addressing

**Base endpoint format:**
```
http://<account-name>.blob.core.windows.net
```

**Example:**
```
http://mystorageaccount.blob.core.windows.net
```

### Key Characteristics

| Characteristic | Details |
|----------------|---------|
| **Uniqueness** | Account name must be globally unique across Azure |
| **Namespace** | Provides unique namespace for all your data |
| **Endpoint** | Forms the base address for all objects in the account |
| **Capacity** | Can contain unlimited containers |

---

## 2. Containers

### Overview
A container organizes a set of blobs, **similar to a directory in a file system**. It provides a way to group related blobs together.

### Container Characteristics

| Characteristic | Details |
|----------------|---------|
| **Number per account** | Unlimited |
| **Blobs per container** | Unlimited |
| **Naming** | Must be a valid DNS name |
| **URL part** | Forms part of the unique URI for blobs |

### Container Naming Rules

✅ **Must follow these rules:**

1. **Length**: Between 3 and 63 characters long
2. **Start character**: Must start with a letter or number
3. **Allowed characters**: Only lowercase letters, numbers, and dash (-) character
4. **Consecutive dashes**: Two or more consecutive dash characters are NOT permitted

### Container Naming Examples

| Name | Valid? | Reason |
|------|--------|--------|
| `mycontainer` | ✅ Yes | All lowercase, valid characters |
| `my-container` | ✅ Yes | Dash separator allowed |
| `my-container-123` | ✅ Yes | Numbers and dashes allowed |
| `MyContainer` | ❌ No | Uppercase letters not allowed |
| `my--container` | ❌ No | Consecutive dashes not allowed |
| `-mycontainer` | ❌ No | Cannot start with dash |
| `my` | ❌ No | Too short (less than 3 characters) |

### Container URI Format

```
https://<account-name>.blob.core.windows.net/<container-name>
```

**Example:**
```
https://myaccount.blob.core.windows.net/mycontainer
```

---

## 3. Blobs

### Overview
Blobs are the actual data objects stored in containers. Azure Storage supports **three types of blobs**, each optimized for different scenarios.

### Blob Types Comparison

| Blob Type | Composition | Max Size | Primary Use Case |
|-----------|-------------|----------|------------------|
| **Block Blobs** | Individual blocks of data | ~190.7 TiB | Text and binary data, general-purpose storage |
| **Append Blobs** | Blocks optimized for append operations | ~190.7 TiB | Logging data, continuous data streams |
| **Page Blobs** | Random access files | 8 TB | Virtual hard drive (VHD) files, VM disks |

### Block Blobs

**Best for:** General-purpose storage of text and binary data

#### Key Features
- Composed of individual blocks that can be managed independently
- Each block can be a different size
- Blocks can be uploaded in parallel
- Maximum size: approximately **190.7 TiB**

#### Use Cases
- Documents and files
- Images, videos, audio
- Backup and archive data
- Application data
- General binary/text data

#### CLI Example
```bash
# Upload file as block blob
az storage blob upload \
  --account-name <account-name> \
  --container-name <container-name> \
  --name <blob-name> \
  --file <local-file-path> \
  --auth-mode login
```

#### PowerShell Example
```powershell
# Upload file as block blob
Set-AzStorageBlobContent `
  -Container <container-name> `
  -File <local-file-path> `
  -Blob <blob-name> `
  -Context $ctx
```

### Append Blobs

**Best for:** Append operations (logging scenarios)

#### Key Features
- Made up of blocks like block blobs
- **Optimized for append operations**
- Blocks can only be added to the end
- Cannot modify or delete existing blocks
- Maximum size: approximately **190.7 TiB**

#### Use Cases
- **Logging data** from virtual machines
- Application logs
- Audit logs
- Continuous data streams
- Time-series data

#### Key Characteristic
⚠️ **Append-only**: Once a block is written, it cannot be updated or deleted—only new blocks can be appended.

#### CLI Example
```bash
# Create append blob
az storage blob append create \
  --account-name <account-name> \
  --container-name <container-name> \
  --name <blob-name> \
  --auth-mode login

# Append data
az storage blob append upload \
  --account-name <account-name> \
  --container-name <container-name> \
  --name <blob-name> \
  --file <data-file> \
  --auth-mode login
```

### Page Blobs

**Best for:** Random access scenarios (VM disks)

#### Key Features
- Store **random access files**
- Maximum size: **8 TB**
- Optimized for random read/write operations
- Organized as a collection of 512-byte pages
- Primary use: Virtual hard drive (VHD) files

#### Use Cases
- **Virtual machine disks** (OS and data disks)
- Azure virtual machines storage
- Random access data
- Database files

#### Key Characteristic
💡 **Random Access**: Designed for scenarios requiring random read/write operations, not sequential access.

#### CLI Example
```bash
# Create page blob (for VHD)
az storage blob upload \
  --account-name <account-name> \
  --container-name <container-name> \
  --name <blob-name> \
  --file <vhd-file-path> \
  --type page \
  --auth-mode login
```

---

## Blob URI Formats

### Simple Blob URI
```
https://<account-name>.blob.core.windows.net/<container-name>/<blob-name>
```

**Example:**
```
https://myaccount.blob.core.windows.net/mycontainer/myblob
```

### Blob with Virtual Directory
```
https://<account-name>.blob.core.windows.net/<container-name>/<virtual-directory>/<blob-name>
```

**Example:**
```
https://myaccount.blob.core.windows.net/mycontainer/myvirtualdirectory/myblob
```

💡 **Virtual Directories**: Blob storage is flat, but you can simulate hierarchical structure using forward slashes (/) in blob names.

---

## Resource Hierarchy Visualization

```
mystorageaccount.blob.core.windows.net (Storage Account)
│
├── images (Container)
│   ├── photo1.jpg (Block Blob)
│   ├── photo2.jpg (Block Blob)
│   └── vacation/photo3.jpg (Block Blob - virtual directory)
│
├── logs (Container)
│   ├── app.log (Append Blob)
│   └── system.log (Append Blob)
│
└── vhds (Container)
    ├── os-disk.vhd (Page Blob)
    └── data-disk.vhd (Page Blob)
```

---

## Key Concepts

### Storage Account Namespace
- **Globally unique** across all Azure
- Forms the base URL for all resources
- Cannot be changed after creation

### Container as Logical Grouping
- Similar to folders/directories
- Organize related blobs
- Apply access policies at container level
- Unlimited number per account

### Blob Type Selection
- **Block Blobs**: Default choice for most scenarios
- **Append Blobs**: When you need append-only operations
- **Page Blobs**: When you need random access (VMs)

---

## Naming Best Practices

### Storage Account Names
✅ **DO:**
- Use 3-24 characters
- Use only lowercase letters and numbers
- Make it globally unique
- Make it descriptive

❌ **DON'T:**
- Use uppercase letters
- Use special characters (except numbers)
- Use names already taken in Azure

### Container Names
✅ **DO:**
- Use 3-63 characters
- Start with letter or number
- Use lowercase letters, numbers, and dashes
- Make it descriptive of contents

❌ **DON'T:**
- Use uppercase letters
- Use consecutive dashes
- Start or end with dash

### Blob Names
✅ **DO:**
- Use descriptive names
- Use forward slashes (/) to simulate directories
- Include file extensions for clarity

❌ **DON'T:**
- Use special characters that need URL encoding unnecessarily

---

## Exam Tips

🎯 **Know the hierarchy**: Storage Account → Container → Blob (3 levels)

🎯 **Container naming**: 3-63 chars, lowercase only, no consecutive dashes

🎯 **Three blob types**: Block (general), Append (logging), Page (VM disks)

🎯 **Block blob size**: ~190.7 TiB (most scenarios)

🎯 **Page blob size**: 8 TB (VM disks)

🎯 **Append blob characteristic**: Append-only, cannot modify existing blocks

🎯 **URI format**: `https://<account>.blob.core.windows.net/<container>/<blob>`

🎯 **Unlimited**: Unlimited containers per account, unlimited blobs per container

🎯 **Virtual directories**: Simulated using forward slashes in blob names

---

## Quick Reference Commands

### Create Container
```bash
# Azure CLI
az storage container create \
  --name <container-name> \
  --account-name <account-name> \
  --auth-mode login

# PowerShell
New-AzStorageContainer `
  -Name <container-name> `
  -Context $ctx
```

### List Containers
```bash
# Azure CLI
az storage container list \
  --account-name <account-name> \
  --auth-mode login

# PowerShell
Get-AzStorageContainer -Context $ctx
```

### Upload Blob
```bash
# Azure CLI (Block Blob - default)
az storage blob upload \
  --account-name <account-name> \
  --container-name <container-name> \
  --name <blob-name> \
  --file <local-file> \
  --auth-mode login

# PowerShell
Set-AzStorageBlobContent `
  -Container <container-name> `
  -File <local-file> `
  -Blob <blob-name> `
  -Context $ctx
```

### List Blobs
```bash
# Azure CLI
az storage blob list \
  --account-name <account-name> \
  --container-name <container-name> \
  --auth-mode login

# PowerShell
Get-AzStorageBlob `
  -Container <container-name> `
  -Context $ctx
```

---

## Comparison Summary

| Aspect | Storage Account | Container | Blob |
|--------|----------------|-----------|------|
| **Purpose** | Top-level namespace | Logical grouping | Actual data object |
| **Capacity** | Unlimited containers | Unlimited blobs | Up to 190.7 TiB (block/append) or 8 TB (page) |
| **Naming Rules** | 3-24 chars, lowercase, numbers | 3-63 chars, lowercase, numbers, dash | No strict rules, use URL-safe characters |
| **Uniqueness** | Globally unique | Unique within account | Unique within container |
| **URL Format** | `<account>.blob.core.windows.net` | `<account>.blob.core.windows.net/<container>` | `<account>.blob.core.windows.net/<container>/<blob>` |

---

## Additional Resources

- [Storage account overview](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-overview)
- [Container and blob naming](https://learn.microsoft.com/en-us/rest/api/storageservices/naming-and-referencing-containers--blobs--and-metadata)
- [Understanding block blobs, append blobs, and page blobs](https://learn.microsoft.com/en-us/rest/api/storageservices/understanding-block-blobs--append-blobs--and-page-blobs)

[Microsoft Learn - Discover Azure Blob storage resource types](https://learn.microsoft.com/en-us/training/modules/explore-azure-blob-storage/3-blob-storage-resources)
