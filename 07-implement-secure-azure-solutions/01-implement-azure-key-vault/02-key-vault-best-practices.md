# Discover Azure Key Vault Best Practices

## Overview

Azure Key Vault is a tool for securely storing and accessing secrets. A **secret** is anything you want to tightly control access to: API keys, passwords, certificates, tokens, or connection strings. A **vault** is a logical group of secrets with centralized access control.

---

## Authentication Methods

To perform operations with Key Vault, you must **authenticate first**. There are three ways to authenticate:

### 1. Managed Identities for Azure Resources ✅ **RECOMMENDED**

**How it works:**
- Assign an identity to your Azure resource (VM, App Service, Function, etc.)
- Azure automatically manages credential rotation
- No secrets in code or configuration
- Service principal client secret rotated automatically by Azure

**Benefits:**
- ✅ **No credential management**: Azure handles everything
- ✅ **Automatic rotation**: No expiration concerns
- ✅ **Best security**: No secrets to leak
- ✅ **Seamless integration**: Works with all Azure services

**Example:**
```bash
# Enable system-assigned managed identity on App Service
az webapp identity assign \
  --name myappservice \
  --resource-group myresourcegroup

# Grant Key Vault access to the managed identity
az keyvault set-policy \
  --name mykeyvault \
  --object-id <managed-identity-object-id> \
  --secret-permissions get list
```

**Code example (.NET):**
```csharp
// Managed identity automatically authenticates
var client = new SecretClient(
    new Uri("https://mykeyvault.vault.azure.net"),
    new DefaultAzureCredential()  // Uses managed identity automatically
);

var secret = await client.GetSecretAsync("MySecret");
Console.WriteLine($"Secret value: {secret.Value.Value}");
```

### 2. Service Principal and Certificate ⚠️ **ACCEPTABLE**

**How it works:**
- Create service principal in Microsoft Entra ID
- Associate X.509 certificate with service principal
- Application uses certificate to authenticate

**Considerations:**
- ⚠️ **Manual rotation**: You must rotate certificates before expiration
- ⚠️ **Certificate storage**: Certificate must be stored securely
- ✅ **Better than secrets**: Certificates more secure than passwords

**Example:**
```bash
# Create service principal with certificate
az ad sp create-for-rbac \
  --name myapp \
  --create-cert \
  --cert MyAppCert \
  --keyvault mykeyvault

# Application authenticates with certificate
```

**Code example (.NET):**
```csharp
var credential = new ClientCertificateCredential(
    tenantId: "your-tenant-id",
    clientId: "your-client-id",
    certificatePath: "/path/to/cert.pfx"
);

var client = new SecretClient(
    new Uri("https://mykeyvault.vault.azure.net"),
    credential
);
```

### 3. Service Principal and Secret ❌ **NOT RECOMMENDED**

**How it works:**
- Service principal with client secret (password)
- Application uses client ID + secret to authenticate

**Why avoid:**
- ❌ **Hard to rotate**: Difficult to automate rotation
- ❌ **Bootstrap problem**: Where do you store the secret used to access Key Vault?
- ❌ **Secret sprawl**: Defeats purpose of Key Vault
- ❌ **Security risk**: Secrets can be leaked, stolen, or compromised

**Only use when:**
- Managed identity not available (on-premises, third-party cloud)
- Testing/development scenarios

---

## Authentication Method Comparison

| Method | Security | Rotation | Complexity | Use Case |
|--------|----------|----------|------------|----------|
| **Managed Identity** | ⭐⭐⭐⭐⭐ | Automatic | Low | Azure resources (VMs, App Service, Functions) |
| **Service Principal + Cert** | ⭐⭐⭐⭐ | Manual | Medium | Non-Azure resources, cross-tenant |
| **Service Principal + Secret** | ⭐⭐ | Manual | Medium | Last resort, dev/test only |

**Decision tree:**
```
Is your application running in Azure?
├─ YES → Use Managed Identity ✅
└─ NO → Is certificate management feasible?
    ├─ YES → Use Service Principal + Certificate
    └─ NO → Use Service Principal + Secret (with caution)
```

---

## Encryption of Data in Transit

Azure Key Vault enforces **Transport Layer Security (TLS)** protocol to protect data traveling between clients and Key Vault.

### TLS Security Features

| Feature | Description |
|---------|-------------|
| **Protocol** | TLS 1.2+ (TLS 1.0/1.1 deprecated) |
| **Authentication** | Strong mutual authentication |
| **Privacy** | AES encryption for all traffic |
| **Integrity** | Detects tampering, interception, forgery |
| **Key lengths** | RSA 2048-bit minimum |

### Perfect Forward Secrecy (PFS)

**How it works:**
- Each connection uses unique session keys
- Compromising one session key doesn't affect others
- Even if long-term private key is compromised, past sessions remain secure

**Key features:**
- **Unique keys per connection**: Different key for each TLS session
- **Ephemeral keys**: Keys discarded after session ends
- **RSA 2048-bit**: Strong encryption for key exchange

**Client negotiation:**
```
Client → Server: ClientHello (supported ciphers, TLS versions)
Server → Client: ServerHello (chosen cipher, certificate)
Client → Server: Key exchange (encrypted with server's public key)
Server → Client: Server finished
[Secure encrypted communication begins]
```

**Supported cipher suites:**
- TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
- TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
- TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384
- TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256

---

## Azure Key Vault Best Practices

### 1. Use Separate Key Vaults ✅

**Pattern: One vault per application per environment**

```
Organization
├── App1-Dev-KeyVault
├── App1-PreProd-KeyVault
├── App1-Prod-KeyVault
├── App2-Dev-KeyVault
├── App2-PreProd-KeyVault
└── App2-Prod-KeyVault
```

**Benefits:**
- ✅ **Isolation**: Secrets not shared across environments
- ✅ **Security**: Breach in one vault doesn't affect others
- ✅ **Access control**: Different permissions per environment
- ✅ **Compliance**: Easier to audit and meet regulatory requirements

**Example naming convention:**
```
{AppName}-{Environment}-kv-{Region}

Examples:
- contoso-dev-kv-eastus
- contoso-prod-kv-westus
- billing-uat-kv-northeurope
```

**Implementation:**
```bash
# Create vaults for each environment
for env in dev uat prod; do
  az keyvault create \
    --name "myapp-${env}-kv" \
    --resource-group "myapp-${env}-rg" \
    --location eastus \
    --tags Environment=$env Application=MyApp
done
```

### 2. Control Access to Your Vault 🔒

**Principle of Least Privilege:** Grant only the minimum permissions required.

**Azure RBAC roles (recommended):**

| Role | Scope | Permissions |
|------|-------|-------------|
| **Key Vault Administrator** | Full control | Create/delete vaults, manage all objects |
| **Key Vault Secrets Officer** | Secrets management | Create, read, update, delete secrets |
| **Key Vault Secrets User** | Read-only | Read secret values only |
| **Key Vault Crypto Officer** | Keys management | Create, read, update, delete keys |
| **Key Vault Crypto User** | Use keys | Encrypt, decrypt, sign, verify |
| **Key Vault Reader** | Metadata only | View vault/object metadata (not values) |

**Best practices:**
```bash
# ✅ GOOD: Grant specific role to specific principal
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee <app-service-managed-identity-id> \
  --scope /subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/mykeyvault

# ❌ BAD: Grant broad permissions
az keyvault set-policy \
  --name mykeyvault \
  --object-id <id> \
  --secret-permissions all  # Too permissive!
```

**Access control checklist:**
- ✅ Use Azure RBAC instead of access policies
- ✅ Assign permissions to managed identities, not individual users
- ✅ Use groups for user access management
- ✅ Regularly review and audit access
- ✅ Remove unused permissions
- ❌ Never grant "all" permissions
- ❌ Avoid using access keys/connection strings

### 3. Backup Your Vault 💾

**Why backup:**
- Accidental deletion of secrets/keys/certificates
- Corruption or misconfiguration
- Compliance and audit requirements

**Backup strategy:**
```bash
# Backup individual secret
az keyvault secret backup \
  --vault-name mykeyvault \
  --name MySecret \
  --file MySecret.backup

# Backup all secrets (script)
for secret in $(az keyvault secret list --vault-name mykeyvault --query "[].name" -o tsv); do
  az keyvault secret backup \
    --vault-name mykeyvault \
    --name "$secret" \
    --file "${secret}.backup"
done
```

**Restore process:**
```bash
# Restore secret from backup
az keyvault secret restore \
  --vault-name mykeyvault \
  --file MySecret.backup
```

**Backup best practices:**
- ✅ **Regular schedule**: Daily or weekly backups
- ✅ **Secure storage**: Store backups in separate Azure Storage account
- ✅ **Encryption**: Backups are encrypted automatically
- ✅ **Test restores**: Verify backups work before you need them
- ✅ **Version control**: Keep multiple backup versions
- ✅ **Automation**: Use Azure Automation or Logic Apps

**Automated backup example:**
```json
// Logic App trigger: Daily at 2 AM
{
  "schedule": {
    "frequency": "Day",
    "interval": 1,
    "timeZone": "UTC",
    "startTime": "2026-01-01T02:00:00Z"
  },
  "actions": {
    "BackupSecrets": {
      "type": "AzureKeyVault.BackupSecret",
      "inputs": {
        "vaultName": "mykeyvault",
        "storageAccount": "backupstorage"
      }
    }
  }
}
```

### 4. Enable Logging 📊

**Why logging matters:**
- Security auditing (who accessed what, when)
- Troubleshooting access issues
- Compliance requirements (SOC 2, ISO 27001, HIPAA)
- Anomaly detection

**Enable diagnostic logs:**
```bash
# Create Log Analytics workspace (if needed)
az monitor log-analytics workspace create \
  --resource-group myresourcegroup \
  --workspace-name mylogworkspace

# Enable Key Vault logging
az monitor diagnostic-settings create \
  --name KeyVaultAuditLogs \
  --resource $(az keyvault show --name mykeyvault --query id -o tsv) \
  --logs '[
    {
      "category": "AuditEvent",
      "enabled": true,
      "retentionPolicy": {
        "enabled": true,
        "days": 90
      }
    },
    {
      "category": "AzurePolicyEvaluationDetails",
      "enabled": true
    }
  ]' \
  --metrics '[
    {
      "category": "AllMetrics",
      "enabled": true
    }
  ]' \
  --workspace $(az monitor log-analytics workspace show --resource-group myresourcegroup --workspace-name mylogworkspace --query id -o tsv)
```

**Log destinations:**

| Destination | Use Case | Cost |
|-------------|----------|------|
| **Log Analytics** | Query, analyze, create alerts | Moderate |
| **Storage Account** | Long-term archive | Low |
| **Event Hub** | Stream to SIEM (Splunk, QRadar) | Moderate |

**Key metrics to monitor:**

```kql
// Failed authentication attempts
AzureDiagnostics
| where ResourceType == "VAULTS"
| where ResultType == "Unauthorized"
| summarize FailedAttempts = count() by CallerIPAddress, TimeGenerated
| where FailedAttempts > 10
```

```kql
// Successful secret access
AzureDiagnostics
| where ResourceType == "VAULTS"
| where OperationName == "SecretGet"
| where ResultType == "Success"
| summarize AccessCount = count() by identity_claim_upn_s, Resource, bin(TimeGenerated, 1h)
```

**Set up alerts:**
```bash
# Alert on multiple failed attempts
az monitor metrics alert create \
  --name "KeyVault-FailedAuth" \
  --resource-group myresourcegroup \
  --scopes $(az keyvault show --name mykeyvault --query id -o tsv) \
  --condition "avg Availability < 99" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action email notify-team@contoso.com
```

### 5. Enable Recovery Options 🔄

#### Soft Delete (Enabled by Default)

**What it does:**
- Deleted secrets/keys/certificates retained for 7-90 days
- Vault itself retained if deleted
- Can be recovered during retention period

**Configuration:**
```bash
# Soft delete is enabled by default with 90-day retention
# Update retention period (7-90 days)
az keyvault update \
  --name mykeyvault \
  --retention-days 90
```

**Recovery process:**
```bash
# List deleted secrets
az keyvault secret list-deleted --vault-name mykeyvault

# Recover deleted secret
az keyvault secret recover \
  --vault-name mykeyvault \
  --name MyDeletedSecret

# Permanently delete (purge) - only if purge protection is off
az keyvault secret purge \
  --vault-name mykeyvault \
  --name MyDeletedSecret
```

#### Purge Protection (Recommended for Production)

**What it does:**
- Prevents **permanent deletion** during soft-delete retention period
- Even admins cannot purge deleted objects
- Mandatory wait period before permanent deletion

**Enable purge protection:**
```bash
# Enable during vault creation (cannot be disabled later!)
az keyvault create \
  --name mykeyvault \
  --resource-group myresourcegroup \
  --enable-purge-protection true \
  --retention-days 90

# Enable on existing vault (IRREVERSIBLE!)
az keyvault update \
  --name mykeyvault \
  --enable-purge-protection true
```

⚠️ **Warning**: Once purge protection is enabled, it **cannot be disabled**. Plan carefully before enabling.

**Recovery scenario with purge protection:**

```
Day 0: Secret deleted
       ↓
Days 1-90: Secret in soft-deleted state
           ↓ Can be recovered
           └─ Cannot be purged (protected)
       ↓
Day 91: Soft-delete retention expires
       ↓
       Automatic permanent deletion
```

**Best practices:**
- ✅ Enable for **production** environments
- ✅ Enable for **compliance** requirements (GDPR, HIPAA)
- ⚠️ Test impact in dev/test first (cannot undo)
- ❌ Not needed for short-lived dev/test vaults

---

## Security Checklist

### Setup Phase
- ✅ Create separate vaults per app per environment
- ✅ Enable soft delete (enabled by default)
- ✅ Enable purge protection for production vaults
- ✅ Configure diagnostic logging
- ✅ Set up Log Analytics workspace

### Access Control
- ✅ Use managed identities for Azure resources
- ✅ Use Azure RBAC (not access policies)
- ✅ Follow principle of least privilege
- ✅ Grant permissions to groups, not individual users
- ✅ Regularly review access assignments

### Network Security
- ✅ Use private endpoints for production
- ✅ Configure firewall rules if using public endpoint
- ✅ Disable public access if not needed
- ✅ Use service endpoints for Azure VNet resources

### Operations
- ✅ Implement backup strategy (daily recommended)
- ✅ Test backup restores regularly
- ✅ Monitor logs for suspicious activity
- ✅ Set up alerts for failed authentication
- ✅ Rotate secrets regularly
- ✅ Document secret ownership and purpose

### Compliance
- ✅ Enable audit logging for compliance
- ✅ Retain logs per regulatory requirements (90+ days)
- ✅ Use HSM-backed keys for compliance (if required)
- ✅ Implement key rotation policies
- ✅ Document security controls

---

## Common Pitfalls to Avoid

| Pitfall | Risk | Solution |
|---------|------|----------|
| Sharing vaults across apps | Secret sprawl, unclear ownership | One vault per app |
| Granting "all" permissions | Over-privileged access | Specific roles only |
| Ignoring logs | Missed security incidents | Enable logging + alerts |
| No backup strategy | Data loss | Regular automated backups |
| Hardcoded credentials | Secret leakage | Use managed identities |
| Public endpoint without firewall | Unauthorized access | Private endpoints or firewall |
| No soft delete/purge protection | Accidental permanent deletion | Enable both for production |
| Using access policies | Inconsistent permissions | Use Azure RBAC |

---

## Exam Tips

🎯 **Managed identity**: Always the recommended authentication method for Azure resources

🎯 **Three authentication methods**: Managed identity (best), Service principal + certificate (acceptable), Service principal + secret (avoid)

🎯 **TLS encryption**: All data in transit encrypted with TLS 1.2+

🎯 **Perfect Forward Secrecy (PFS)**: Unique keys per session, RSA 2048-bit minimum

🎯 **Separate vaults**: One vault per application per environment (best practice)

🎯 **Azure RBAC**: Recommended over legacy access policies

🎯 **Soft delete**: Enabled by default, 7-90 day retention (90 days default)

🎯 **Purge protection**: Prevents permanent deletion, cannot be disabled once enabled

🎯 **Logging destinations**: Storage Account (archive), Event Hub (stream), Log Analytics (query)

🎯 **Backup scope**: Individual objects (secrets, keys, certificates), not entire vault

🎯 **Security principle**: Least privilege access, grant minimum required permissions

🎯 **Bootstrap problem**: Don't use secrets to access Key Vault (use managed identity)

---

## Additional Resources

- [Key Vault Best Practices](https://learn.microsoft.com/en-us/azure/key-vault/general/best-practices)
- [Soft Delete Overview](https://learn.microsoft.com/en-us/azure/key-vault/general/soft-delete-overview)
- [Logging and Monitoring](https://learn.microsoft.com/en-us/azure/key-vault/general/logging)
- [Azure RBAC for Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide)

[Microsoft Learn - Discover Azure Key Vault best practices](https://learn.microsoft.com/en-us/training/modules/implement-azure-key-vault/3-key-vault-concepts)
