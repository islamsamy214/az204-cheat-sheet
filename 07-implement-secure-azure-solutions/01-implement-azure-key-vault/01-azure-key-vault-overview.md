# Explore Azure Key Vault

## Overview

**Azure Key Vault** is a cloud service for securely storing and accessing secrets, keys, and certificates. It eliminates the need to store sensitive information in application code and provides centralized secret management with strict access control.

---

## What is Azure Key Vault?

Azure Key Vault helps solve three key problems:

| Problem | Solution |
|---------|----------|
| **Secrets Management** | Securely store and control access to tokens, passwords, certificates, API keys, and other secrets |
| **Key Management** | Create and control encryption keys used to encrypt your data |
| **Certificate Management** | Provision, manage, and deploy public and private SSL/TLS certificates |

---

## Key Vault Types

Azure Key Vault supports two types of containers:

### 1. Vaults (Standard)

- **Storage**: Software and HSM-backed keys, secrets, and certificates
- **Protection**: Software-based encryption
- **Use case**: Most applications
- **Cost**: Lower cost option

### 2. Managed HSM Pools (Premium)

- **Storage**: HSM-backed keys only
- **Protection**: Hardware Security Module (HSM) protected
- **Use case**: Regulatory compliance, high-security requirements
- **Cost**: Higher cost, FIPS 140-2 Level 3 validated

---

## Service Tiers Comparison

| Feature | Standard | Premium |
|---------|----------|---------|
| **Key Protection** | Software-protected | HSM-protected |
| **Encryption** | AES 256-bit | AES 256-bit |
| **FIPS Compliance** | FIPS 140-2 Level 1 | FIPS 140-2 Level 2 (Vaults)<br>FIPS 140-2 Level 3 (Managed HSM) |
| **Key Types** | RSA, EC | RSA, EC, OCT (Managed HSM only) |
| **Pricing** | Per-operation | Per-operation + HSM fee |
| **Best For** | Most applications | Regulatory compliance, high-security |

---

## Key Benefits of Using Azure Key Vault

### 1. Centralized Application Secrets

✅ **Single source of truth** for all application secrets
- Store connection strings, API keys, passwords centrally
- Applications access secrets via URIs instead of hardcoding
- Retrieve specific versions of secrets when needed
- Easy rotation and updates without code changes

**Example URI format:**
```
https://{vault-name}.vault.azure.net/secrets/{secret-name}/{version}
```

### 2. Securely Store Secrets and Keys

✅ **Multi-layered security**:
- **Authentication**: Microsoft Entra ID (formerly Azure AD)
- **Authorization**: 
  - Azure RBAC (Role-Based Access Control) - recommended
  - Key Vault access policies (legacy)
- **Encryption**:
  - Standard tier: Software-protected keys
  - Premium tier: FIPS 140-2 Level 2 HSM-protected keys

| Authorization Method | Management Plane | Data Plane |
|---------------------|------------------|------------|
| **Azure RBAC** | ✅ Supported | ✅ Supported |
| **Access Policies** | ❌ Not supported | ✅ Supported |

**Best practice**: Use Azure RBAC for consistent authorization across Azure resources.

### 3. Monitor Access and Use

✅ **Comprehensive logging**:
- Enable logging for all vaults
- Track who accessed what and when
- Audit secret retrieval and modifications

**Monitoring options:**
| Destination | Purpose |
|-------------|---------|
| **Storage Account** | Archive logs long-term |
| **Event Hub** | Stream logs to SIEM systems |
| **Azure Monitor Logs** | Query and analyze with KQL |

**Example metrics:**
- Total API requests
- Failed authentication attempts
- Secret retrieval operations
- Average latency

### 4. Simplified Administration

✅ **Eliminates complexity**:

| Traditional Approach | With Key Vault |
|---------------------|----------------|
| Purchase and maintain HSMs | Azure manages HSMs |
| Manual scaling | Auto-scales for usage spikes |
| Manual replication | Automatic region replication |
| Manual failover | Automatic failover |
| Complex certificate lifecycle | Automated enrollment and renewal |

**Key simplifications:**
- **No HSM knowledge required**: Azure handles HSM complexities
- **Auto-scaling**: Handles traffic spikes automatically
- **High availability**: Automatic replication within region and to secondary region
- **Certificate automation**: Auto-renew certificates from Public CAs
- **Standard management**: Use Azure Portal, Azure CLI, or PowerShell

---

## How Key Vault Works

### Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Application                       │
│  ┌─────────────────────────────────────────────┐   │
│  │   Azure Identity (DefaultAzureCredential)    │   │
│  └─────────────────┬───────────────────────────┘   │
└────────────────────┼─────────────────────────────────┘
                     │
                     │ Authenticate (Microsoft Entra ID)
                     ▼
┌─────────────────────────────────────────────────────┐
│              Microsoft Entra ID                      │
│         (Authentication & Authorization)             │
└─────────────────┬───────────────────────────────────┘
                  │
                  │ Access Token
                  ▼
┌─────────────────────────────────────────────────────┐
│              Azure Key Vault                         │
│  ┌───────────────────────────────────────────────┐ │
│  │  Secrets  │  Keys  │  Certificates             │ │
│  └───────────────────────────────────────────────┘ │
│  ┌───────────────────────────────────────────────┐ │
│  │  Audit Logs → Azure Monitor / Storage / Event │ │
│  └───────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

### Access Flow

1. **Application requests access** → Uses managed identity or service principal
2. **Microsoft Entra ID authenticates** → Verifies identity
3. **Azure RBAC authorizes** → Checks permissions
4. **Key Vault returns secret** → Application receives secret value
5. **Operation logged** → Audit trail created

---

## What Can You Store in Key Vault?

### 1. Secrets

**What**: Any sensitive string data

**Examples:**
- Database connection strings
- API keys
- Passwords
- SAS tokens
- Storage account keys

**Size limit**: 25 KB per secret

**Example:**
```bash
az keyvault secret set \
  --vault-name mykeyvault \
  --name "DatabaseConnection" \
  --value "Server=myserver;Database=mydb;User=admin;Password=secret123"
```

### 2. Keys

**What**: Cryptographic keys for encryption/decryption

**Key types:**
- **RSA**: 2048-bit, 3072-bit, 4096-bit
- **EC (Elliptic Curve)**: P-256, P-384, P-521, P-256K
- **OCT (Symmetric)**: 128-bit, 192-bit, 256-bit (Managed HSM only)

**Operations:**
- Encrypt/Decrypt
- Sign/Verify
- Wrap/Unwrap (key wrapping)

**Example:**
```bash
az keyvault key create \
  --vault-name mykeyvault \
  --name "MyEncryptionKey" \
  --protection software \
  --kty RSA \
  --size 2048
```

### 3. Certificates

**What**: X.509 certificates (SSL/TLS)

**Features:**
- Automated certificate lifecycle management
- Integration with Certificate Authorities (CAs)
- Auto-renewal for supported CAs
- PFX or PEM format

**Supported CAs:**
- DigiCert
- GlobalSign
- Self-signed certificates

**Example:**
```bash
az keyvault certificate create \
  --vault-name mykeyvault \
  --name "MyCertificate" \
  --policy @policy.json
```

---

## Regional Availability and Replication

### Data Replication

| Type | Behavior |
|------|----------|
| **Primary region** | All data replicated within region |
| **Secondary region** | Paired Azure region (automatic) |
| **Failover** | Automatic (no admin action needed) |
| **Read access** | Primary only (unless failover occurs) |

**Example paired regions:**
- East US ↔ West US
- North Europe ↔ West Europe
- Southeast Asia ↔ East Asia

### High Availability

- **RPO (Recovery Point Objective)**: Minutes
- **RTO (Recovery Time Objective)**: Automatic failover
- **Data durability**: 99.999999999% (11 nines)
- **Service SLA**: 99.99% availability

---

## Common Use Cases

### 1. Secure Application Configuration

**Before (Insecure):**
```csharp
// ❌ Hardcoded in appsettings.json
{
  "ConnectionStrings": {
    "Database": "Server=sql.database.windows.net;Password=MySecret123"
  }
}
```

**After (Secure with Key Vault):**
```csharp
// ✅ Reference from Key Vault
{
  "ConnectionStrings": {
    "Database": "@Microsoft.KeyVault(SecretUri=https://mykv.vault.azure.net/secrets/DbConnection)"
  }
}
```

### 2. Encryption Key Management

```csharp
// Encrypt data with Key Vault key
var keyClient = new KeyClient(new Uri("https://mykv.vault.azure.net"), new DefaultAzureCredential());
var key = await keyClient.GetKeyAsync("MyEncryptionKey");

var cryptoClient = new CryptographyClient(key.Value.Id, new DefaultAzureCredential());
byte[] plaintext = Encoding.UTF8.GetBytes("Sensitive data");
var encryptResult = await cryptoClient.EncryptAsync(EncryptionAlgorithm.RsaOaep, plaintext);
```

### 3. Certificate Lifecycle Management

- Deploy SSL/TLS certificates to Azure services
- Auto-renew certificates before expiration
- Centralized certificate inventory
- Track certificate expiration dates

---

## Security Features

### Authentication Options

1. **Managed Identity** (Recommended)
   - No credentials in code
   - Automatic credential rotation
   - Works with Azure services

2. **Service Principal**
   - Client ID + Certificate (recommended)
   - Client ID + Secret (less secure)

3. **User Identity**
   - For interactive scenarios
   - Azure CLI/PowerShell authentication

### Authorization Models

#### Azure RBAC (Recommended)

| Role | Permissions |
|------|-------------|
| **Key Vault Administrator** | Full access to Key Vault and all objects |
| **Key Vault Secrets Officer** | Full access to secrets |
| **Key Vault Secrets User** | Read secrets only |
| **Key Vault Crypto Officer** | Full access to keys |
| **Key Vault Crypto User** | Use keys for crypto operations |
| **Key Vault Certificates Officer** | Manage certificates |
| **Key Vault Reader** | Read metadata (not secret values) |

#### Access Policies (Legacy)

- **Permissions**: Grant/deny per object type (secrets, keys, certificates)
- **Granularity**: Per user/service principal
- **Limitation**: No management plane control

---

## Networking and Security

### Network Access Options

| Option | Description | Use Case |
|--------|-------------|----------|
| **Public endpoint** | Accessible from internet | Default, most applications |
| **Service endpoints** | Restrict to Azure VNet | Internal Azure resources |
| **Private endpoints** | Private IP in VNet | Zero internet exposure |
| **Firewall rules** | IP allowlist/blocklist | Specific IP ranges only |

**Best practice**: Use private endpoints for production workloads.

### Soft Delete and Purge Protection

| Feature | Purpose | Default |
|---------|---------|---------|
| **Soft Delete** | Retain deleted objects for 7-90 days | Enabled (90 days) |
| **Purge Protection** | Prevent permanent deletion during retention | Optional (recommended) |

**Recovery process:**
```bash
# List deleted vaults
az keyvault list-deleted

# Recover deleted vault
az keyvault recover --name mykeyvault

# Recover deleted secret
az keyvault secret recover --vault-name mykeyvault --name MySecret
```

---

## Pricing

### Standard Tier

| Operation | Cost (approximate) |
|-----------|-------------------|
| Secret operations | $0.03 per 10,000 transactions |
| Key operations (software) | $0.03 per 10,000 transactions |
| Certificate operations | $3.00 per renewal |
| Managed storage account keys | $3.00 per account per month |

### Premium Tier

- **All Standard costs** +
- **HSM-protected keys**: $1.00 per key per month
- **HSM operations**: Higher per-transaction cost

### Managed HSM

- **Pool fee**: ~$4.00 per hour (per HSM)
- **High availability**: 3 HSM replicas minimum
- **Monthly cost**: ~$3,000+ per month

💡 **Cost tip**: Use Standard tier for most workloads. Premium/Managed HSM only for compliance requirements.

---

## Exam Tips

🎯 **Two container types**: Vaults (most common) and Managed HSM Pools

🎯 **Three object types**: Secrets, Keys, Certificates

🎯 **Service tiers**: Standard (software-protected) vs Premium (HSM-protected)

🎯 **Authentication**: Microsoft Entra ID required

🎯 **Authorization**: Azure RBAC (recommended) or Access Policies (legacy)

🎯 **Managed Identity**: Best practice for authentication (no credential management)

🎯 **Soft delete**: Enabled by default (90-day retention)

🎯 **Regional replication**: Automatic within region and to paired region

🎯 **Logging**: Support for Storage Account, Event Hub, and Azure Monitor

🎯 **URI format**: `https://{vault-name}.vault.azure.net/{object-type}/{object-name}`

🎯 **Size limits**: 25 KB for secrets, no limit for keys/certificates

🎯 **High availability**: 99.99% SLA, automatic failover

---

## Quick Reference Commands

### Create Key Vault
```bash
az keyvault create \
  --name mykeyvault \
  --resource-group myresourcegroup \
  --location eastus \
  --sku standard
```

### Set Secret
```bash
az keyvault secret set \
  --vault-name mykeyvault \
  --name "MySecret" \
  --value "MySecretValue"
```

### Get Secret
```bash
az keyvault secret show \
  --vault-name mykeyvault \
  --name "MySecret" \
  --query value -o tsv
```

### Create Key
```bash
az keyvault key create \
  --vault-name mykeyvault \
  --name "MyKey" \
  --protection software
```

### Enable Logging
```bash
az monitor diagnostic-settings create \
  --name KeyVaultLogs \
  --resource $(az keyvault show --name mykeyvault --query id -o tsv) \
  --logs '[{"category":"AuditEvent","enabled":true}]' \
  --workspace myworkspace
```

---

## Additional Resources

- [Azure Key Vault Documentation](https://learn.microsoft.com/en-us/azure/key-vault/)
- [Azure Key Vault Pricing](https://azure.microsoft.com/pricing/details/key-vault/)
- [Key Vault Developer's Guide](https://learn.microsoft.com/en-us/azure/key-vault/general/developers-guide)
- [Best Practices for Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/general/best-practices)

[Microsoft Learn - Explore Azure Key Vault](https://learn.microsoft.com/en-us/training/modules/implement-azure-key-vault/2-key-vault-overview)
