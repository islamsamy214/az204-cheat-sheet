# Azure Storage Security Features

## Security Overview

Azure Storage provides comprehensive security features to protect your data:
1. **Encryption at rest** (server-side)
2. **Encryption in transit** (HTTPS)
3. **Client-side encryption**
4. **Authentication and authorization**
5. **Network security**

---

## Azure Storage Encryption for Data at Rest

### Automatic Encryption

Azure Storage uses **service-side encryption (SSE)** to automatically encrypt your data when persisting it to the cloud.

#### Key Features

| Feature | Details |
|---------|---------|
| **Algorithm** | 256-bit Advanced Encryption Standard (AES) encryption |
| **Compliance** | FIPS 140-2 compliant |
| **Status** | **Always enabled** (cannot be disabled) |
| **Scope** | All storage accounts (new and existing) |
| **Performance tiers** | All tiers (Standard and Premium) |
| **Access tiers** | All access tiers (Hot, Cool, Cold, Archive) |
| **Blob types** | All blob types (block, append, page) |
| **Redundancy** | All redundancy options supported |
| **Metadata** | Object metadata also encrypted |
| **Cost** | **No extra cost** for encryption |

#### Transparent Encryption

💡 **Automatic Process**: Data is encrypted and decrypted **transparently** using AES encryption.

✅ **No Code Changes**: You don't need to modify your code or applications to take advantage of Azure Storage encryption.

⚠️ **Cannot Be Disabled**: Because data is secured by default, encryption cannot be turned off.

#### Encryption Scope

All Azure Storage resources are encrypted:
- ✅ Blobs (block, append, page)
- ✅ Disks
- ✅ Files
- ✅ Queues
- ✅ Tables

#### Geo-Replication Encryption

When geo-replication is enabled:
- Primary region data is encrypted
- Secondary region data is encrypted
- Encryption remains active across all replicas

---

## Encryption Key Management

Azure Storage provides **three options** for managing encryption keys:

### 1. Microsoft-Managed Keys (Default)

**Overview**: Microsoft manages all aspects of key management.

#### Characteristics

| Aspect | Details |
|--------|---------|
| **Key Storage** | Microsoft key store |
| **Key Rotation** | Managed by Microsoft |
| **Key Control** | Microsoft |
| **Scope** | Account (default), container, or blob |
| **Supported Services** | All Azure Storage services |
| **Complexity** | Lowest (fully automated) |
| **Cost** | Included in storage cost |

✅ **Best for**: Most scenarios where you don't need direct key control

### 2. Customer-Managed Keys (CMK)

**Overview**: You manage encryption keys using Azure Key Vault or Azure Key Vault Managed HSM.

#### Characteristics

| Aspect | Details |
|--------|---------|
| **Key Storage** | Azure Key Vault or Key Vault HSM |
| **Key Rotation** | Customer responsibility |
| **Key Control** | Customer |
| **Scope** | Account (default), container, or blob |
| **Supported Services** | Blob Storage, Azure Files |
| **Complexity** | Medium (requires Key Vault setup) |

#### Key Vault Integration

```bash
# Create Key Vault
az keyvault create \
  --name <keyvault-name> \
  --resource-group <resource-group> \
  --location <location>

# Create encryption key
az keyvault key create \
  --vault-name <keyvault-name> \
  --name <key-name> \
  --protection software

# Configure storage account to use customer-managed key
az storage account update \
  --name <account-name> \
  --resource-group <resource-group> \
  --encryption-key-name <key-name> \
  --encryption-key-vault <keyvault-uri> \
  --encryption-key-source Microsoft.Keyvault
```

#### PowerShell Example

```powershell
# Create Key Vault key
$key = Add-AzKeyVaultKey `
  -VaultName <keyvault-name> `
  -Name <key-name> `
  -Destination Software

# Configure storage account
Set-AzStorageAccount `
  -ResourceGroupName <resource-group> `
  -Name <account-name> `
  -KeyvaultEncryption `
  -KeyName <key-name> `
  -KeyVaultUri <keyvault-uri>
```

✅ **Best for**: Scenarios requiring direct control over encryption keys

⚠️ **Requirements**:
- Azure Key Vault or Key Vault Managed HSM
- Appropriate permissions to Key Vault
- Key rotation management

### 3. Customer-Provided Keys (CPK)

**Overview**: You provide an encryption key on each read/write request for granular control.

#### Characteristics

| Aspect | Details |
|--------|---------|
| **Key Storage** | Customer's own key store |
| **Key Rotation** | Customer responsibility |
| **Key Control** | Customer (full control) |
| **Scope** | Per-request (most granular) |
| **Supported Services** | Blob Storage only |
| **Complexity** | Highest (must provide key on every request) |

#### Usage Pattern

```csharp
// Create customer-provided key
var cpk = new CustomerProvidedKey(keyBytes);

// Upload with customer-provided key
BlobUploadOptions options = new BlobUploadOptions
{
    CustomerProvidedKey = cpk
};

await blobClient.UploadAsync(stream, options);

// Download with customer-provided key
BlobDownloadOptions downloadOptions = new BlobDownloadOptions
{
    CustomerProvidedKey = cpk
};

await blobClient.DownloadToAsync(stream, downloadOptions);
```

✅ **Best for**: Maximum granular control, different keys for different blobs

⚠️ **Requirement**: Must provide key on **every read/write operation**

---

## Key Management Comparison

| Feature | Microsoft-Managed | Customer-Managed (CMK) | Customer-Provided (CPK) |
|---------|-------------------|------------------------|-------------------------|
| **Key Storage** | Microsoft store | Azure Key Vault/HSM | Your own store |
| **Rotation Responsibility** | Microsoft | Customer | Customer |
| **Control Level** | Microsoft | Customer | Customer |
| **Granularity** | Account/container/blob | Account/container/blob | Per-request |
| **Services** | All | Blob Storage, Azure Files | Blob Storage only |
| **Complexity** | Low | Medium | High |
| **Setup Required** | None | Key Vault | Custom key management |
| **Request Overhead** | None | None | Every request |

---

## Client-Side Encryption

### Overview

Client-side encryption allows you to **encrypt data in your client application** before uploading to Azure Storage and **decrypt data after downloading**.

### Supported SDKs

Azure Blob Storage and Queue Storage client libraries support client-side encryption:
- ✅ .NET
- ✅ Java
- ✅ Python

### Encryption Versions

| Version | Algorithm | Mode | Supported Services |
|---------|-----------|------|-------------------|
| **Version 2** (Recommended) | AES | **Galois/Counter Mode (GCM)** | Blob Storage, Queue Storage |
| **Version 1** (Legacy) | AES | **Cipher Block Chaining (CBC)** | Blob Storage, Queue Storage, Table Storage |

💡 **Recommendation**: Use **Version 2** (GCM mode) for new implementations—it's more secure and efficient.

### How It Works

```
Client Application
    ↓ (Encrypt data with AES-GCM)
Azure Blob Storage
    ↓ (Also encrypted at rest by Azure)
Storage Account
```

### Client-Side Encryption (.NET Example)

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Specialized;

// Create blob client with client-side encryption
var clientOptions = new SpecializedBlobClientOptions
{
    ClientSideEncryption = new ClientSideEncryptionOptions(
        ClientSideEncryptionVersion.V2_0)
    {
        KeyEncryptionKey = keyEncryptionKey,
        KeyResolver = keyResolver,
        KeyWrapAlgorithm = "RSA-OAEP-256"
    }
};

var blobClient = new BlobClient(connectionString, containerName, blobName, clientOptions);

// Upload - data encrypted on client before upload
await blobClient.UploadAsync(stream);

// Download - data decrypted on client after download
await blobClient.DownloadToAsync(stream);
```

### Layered Encryption

When using client-side encryption:
1. **Client encrypts** data before upload (AES-GCM)
2. **Azure encrypts** data at rest (AES-256)
3. Result: **Double encryption** (defense in depth)

---

## Encryption at Rest vs. In Transit

| Type | When | How | Configuration |
|------|------|-----|---------------|
| **At Rest** | Data stored on disk | AES-256 encryption | Always enabled, automatic |
| **In Transit** | Data moving over network | HTTPS/TLS | Use HTTPS endpoints |

### Secure Transfer Required

```bash
# Enable secure transfer (HTTPS only)
az storage account update \
  --name <account-name> \
  --resource-group <resource-group> \
  --https-only true
```

```powershell
# PowerShell
Set-AzStorageAccount `
  -ResourceGroupName <resource-group> `
  -Name <account-name> `
  -EnableHttpsTrafficOnly $true
```

⚠️ **Best Practice**: Always enable "Secure transfer required" to enforce HTTPS.

---

## Security Best Practices

### Encryption

✅ **DO:**
- Trust default encryption (always enabled)
- Use HTTPS for all connections
- Enable "Secure transfer required" setting
- Use customer-managed keys if you need key control
- Implement client-side encryption for sensitive data
- Use AES-GCM (Version 2) for client-side encryption

❌ **DON'T:**
- Assume encryption is optional (it's always on)
- Use HTTP endpoints for sensitive data
- Use deprecated CBC mode (Version 1) for new applications

### Key Management

✅ **DO:**
- Use Microsoft-managed keys for most scenarios
- Use customer-managed keys when compliance requires it
- Store customer-managed keys in Azure Key Vault
- Rotate customer-managed keys regularly
- Document key rotation procedures

❌ **DON'T:**
- Manage keys manually without proper procedures
- Store keys in code or configuration files
- Forget to rotate customer-managed keys

### Access Control

✅ **DO:**
- Use Azure AD authentication when possible
- Apply principle of least privilege
- Use SAS tokens with minimal permissions and short expiration
- Monitor access logs

---

## Exam Tips

🎯 **Encryption always enabled**: Azure Storage encryption is **always on** and **cannot be disabled**

🎯 **AES-256**: Azure uses 256-bit AES encryption, FIPS 140-2 compliant

🎯 **No extra cost**: Encryption at rest is included, no additional charge

🎯 **Three key options**: Microsoft-managed (default), Customer-managed (Key Vault), Customer-provided (per-request)

🎯 **Client-side encryption**: Available in .NET, Java, Python SDKs

🎯 **Two client-side versions**: V2 (GCM - recommended), V1 (CBC - legacy)

🎯 **CMK requires Key Vault**: Customer-managed keys must be stored in Azure Key Vault or Key Vault HSM

🎯 **CPK is per-request**: Customer-provided keys must be supplied on every read/write operation

🎯 **Supported services for CMK**: Blob Storage and Azure Files (not Queue/Table)

🎯 **Supported services for CPK**: Blob Storage only

🎯 **Transparent encryption**: No code changes needed for at-rest encryption

🎯 **Secure transfer**: Enable HTTPS-only to enforce encryption in transit

---

## Quick Reference Commands

### Check Encryption Status
```bash
# Azure CLI
az storage account show \
  --name <account-name> \
  --resource-group <resource-group> \
  --query encryption

# PowerShell
(Get-AzStorageAccount `
  -ResourceGroupName <resource-group> `
  -Name <account-name>).Encryption
```

### Enable Secure Transfer (HTTPS Only)
```bash
# Azure CLI
az storage account update \
  --name <account-name> \
  --resource-group <resource-group> \
  --https-only true

# PowerShell
Set-AzStorageAccount `
  -ResourceGroupName <resource-group> `
  -Name <account-name> `
  -EnableHttpsTrafficOnly $true
```

### Configure Customer-Managed Key
```bash
# Azure CLI
az storage account update \
  --name <account-name> \
  --resource-group <resource-group> \
  --encryption-key-name <key-name> \
  --encryption-key-vault <keyvault-uri> \
  --encryption-key-source Microsoft.Keyvault

# PowerShell
Set-AzStorageAccount `
  -ResourceGroupName <resource-group> `
  -Name <account-name> `
  -KeyvaultEncryption `
  -KeyName <key-name> `
  -KeyVaultUri <keyvault-uri>
```

---

## Encryption Decision Tree

```
Do you need control over encryption keys?
├── No → Use Microsoft-managed keys (default)
│         ✅ Simplest option
│         ✅ No additional setup
│         ✅ Microsoft handles rotation
│
└── Yes → Do you need per-request granularity?
          ├── No → Use Customer-managed keys (CMK)
          │         ✅ Store in Azure Key Vault
          │         ✅ Control rotation
          │         ✅ Works with Blob Storage & Files
          │
          └── Yes → Use Customer-provided keys (CPK)
                    ✅ Maximum control
                    ✅ Different key per blob
                    ⚠️  Blob Storage only
                    ⚠️  Must provide on every request
```

---

## Additional Resources

- [Azure Storage encryption for data at rest](https://learn.microsoft.com/en-us/azure/storage/common/storage-service-encryption)
- [Customer-managed keys for Azure Storage encryption](https://learn.microsoft.com/en-us/azure/storage/common/customer-managed-keys-overview)
- [Client-side encryption for blobs](https://learn.microsoft.com/en-us/azure/storage/blobs/client-side-encryption)

[Microsoft Learn - Explore Azure Storage security features](https://learn.microsoft.com/en-us/training/modules/explore-azure-blob-storage/4-blob-storage-security)
