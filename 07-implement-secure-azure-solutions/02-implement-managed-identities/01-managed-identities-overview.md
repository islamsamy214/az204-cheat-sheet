# Explore Managed Identities

## Overview

**Managed identities** eliminate the need for developers to manage credentials (secrets, certificates, keys) when accessing Azure resources. Applications use managed identities to obtain Microsoft Entra tokens without storing or managing any credentials.

**Problem solved:** The credential management challenge
- ❌ Before: Store credentials in config, secrets in Key Vault, rotation complexity
- ✅ After: Azure manages everything automatically, no credentials to store

---

## Why Use Managed Identities?

### The Credential Management Challenge

**Traditional approach problems:**
```csharp
// ❌ BAD: Credentials in code/config
string connectionString = "AccountName=storage;AccountKey=ABC123...";
string clientSecret = "super-secret-password-12345";
```

**Issues:**
- Credentials must be stored somewhere (code, config, Key Vault)
- Manual rotation required
- Risk of credential leak
- Complex secret management

**Managed identity approach:**
```csharp
// ✅ GOOD: No credentials needed!
var credential = new DefaultAzureCredential();  // Uses managed identity automatically
var client = new BlobServiceClient(new Uri("https://myaccount.blob.core.windows.net"), credential);
```

**Benefits:**
- ✅ **Zero credential management**: Azure handles everything
- ✅ **Automatic rotation**: No expiration concerns
- ✅ **No secrets to leak**: Nothing to steal or expose
- ✅ **Simplified code**: Same code locally (Azure CLI) and in production (managed identity)

---

## What is a Managed Identity?

A managed identity is an **automatically managed service principal** in Microsoft Entra ID that applications use to authenticate to Azure resources.

**Key characteristics:**
- Created and managed by Azure
- Locked to only be used with Azure resources
- Automatically deleted when associated resource is deleted (system-assigned)
- No credential management required

**Architecture:**
```
Application (VM, App Service, Function)
    ↓ Uses
Managed Identity (Service Principal)
    ↓ Authenticates to
Microsoft Entra ID
    ↓ Issues token for
Azure Resource (Key Vault, Storage, SQL, etc.)
```

---

## Types of Managed Identities

### 1. System-Assigned Managed Identity

**Definition:** Enabled directly on an Azure service instance. Lifecycle tied to the resource.

**How it works:**
1. Enable managed identity on resource (VM, App Service, etc.)
2. Azure creates service principal in Microsoft Entra ID
3. Credentials provisioned to the resource automatically
4. When resource is deleted, identity is automatically removed

**Characteristics:**

| Aspect | System-Assigned |
|--------|-----------------|
| **Creation** | Created as part of Azure resource |
| **Lifecycle** | Tied to parent resource (shared lifecycle) |
| **Sharing** | Cannot be shared (1:1 relationship) |
| **Deletion** | Automatic when parent resource deleted |
| **Use case** | Single resource needing its own identity |

**Example:**
```bash
# Enable on Virtual Machine
az vm identity assign \
  --name myvm \
  --resource-group myresourcegroup

# Enable on App Service
az webapp identity assign \
  --name myappservice \
  --resource-group myresourcegroup

# Enable on Azure Function
az functionapp identity assign \
  --name myfunctionapp \
  --resource-group myresourcegroup
```

**Visual representation:**
```
VM-1 ←→ System-Assigned Identity A (dedicated, cannot share)
VM-2 ←→ System-Assigned Identity B (dedicated, cannot share)
AppService-1 ←→ System-Assigned Identity C (dedicated, cannot share)
```

**When to use:**
- ✅ Single resource workloads
- ✅ Resources needing independent identities
- ✅ Simple scenarios with 1:1 mapping
- ✅ Resources that won't be frequently recreated

**Example scenario:**
```
Single web application on App Service
    ↓ Uses system-assigned identity
Accesses Key Vault to retrieve secrets
```

### 2. User-Assigned Managed Identity

**Definition:** Created as a standalone Azure resource. Independent lifecycle from resources using it.

**How it works:**
1. Create user-assigned identity as separate resource
2. Azure creates service principal in Microsoft Entra ID
3. Assign identity to one or more Azure resources
4. Identity persists independently of resource lifecycle

**Characteristics:**

| Aspect | User-Assigned |
|--------|---------------|
| **Creation** | Standalone Azure resource |
| **Lifecycle** | Independent (must be explicitly deleted) |
| **Sharing** | Can be shared across multiple resources |
| **Deletion** | Manual (persists after resource deletion) |
| **Use case** | Multiple resources sharing same identity |

**Example:**
```bash
# 1. Create user-assigned identity
az identity create \
  --name myIdentity \
  --resource-group myresourcegroup

# 2. Assign to VM
az vm identity assign \
  --name myvm \
  --resource-group myresourcegroup \
  --identities myIdentity

# 3. Assign to App Service (same identity)
az webapp identity assign \
  --name myappservice \
  --resource-group myresourcegroup \
  --identities myIdentity
```

**Visual representation:**
```
User-Assigned Identity X
    ↓ Shared by
    ├── VM-1
    ├── VM-2
    ├── AppService-1
    └── Function-1
```

**When to use:**
- ✅ Multiple resources needing same permissions
- ✅ Resources recycled frequently (permissions stay consistent)
- ✅ Pre-authorization needed during provisioning
- ✅ Resources requiring identical access to resources

**Example scenario:**
```
3 VMs running same application
    ↓ All use same user-assigned identity
    ↓ Grant permissions once to identity
Access same Key Vault, Storage, and SQL Database
```

---

## Comparison: System-Assigned vs User-Assigned

| Feature | System-Assigned | User-Assigned |
|---------|-----------------|---------------|
| **Creation** | Part of resource creation | Standalone resource |
| **Lifecycle** | Tied to resource | Independent |
| **Sharing** | No (1:1 only) | Yes (many:1) |
| **Deletion** | Automatic | Manual |
| **Permission Management** | Per resource | Centralized |
| **Resource Recreation** | Identity lost | Identity persists |
| **Complexity** | Simple | Slightly more complex |
| **Best For** | Single resources | Multiple resources |

---

## Common Use Cases

### System-Assigned Use Cases

1. **Single VM Application**
```
VM running web app
    ↓ System-assigned identity
Access Key Vault for app secrets
```

2. **Isolated Microservice**
```
App Service hosting API
    ↓ System-assigned identity
Access SQL Database and Storage Account
```

3. **Function App**
```
Azure Function processing data
    ↓ System-assigned identity
Read/write to Cosmos DB
```

### User-Assigned Use Cases

1. **Auto-Scaling Web Farm**
```
User-Assigned Identity "WebAppIdentity"
    ↓ Shared by
    ├── VM Instance 1
    ├── VM Instance 2
    ├── VM Instance 3 (auto-scaled)
    └── VM Instance 4 (auto-scaled)
    ↓ All access
Key Vault, Storage, Application Insights
```

**Benefits:**
- Add/remove VMs without updating permissions
- Pre-configure identity before VM creation
- Consistent access across all instances

2. **Blue-Green Deployment**
```
User-Assigned Identity "AppIdentity"
    ↓ Shared by
    ├── Blue Environment (Production)
    └── Green Environment (Staging)
    ↓ Both access
Same Azure resources with identical permissions
```

**Benefits:**
- Switch environments without permission changes
- Test with production permissions
- Zero-downtime deployments

3. **Dev/Test/Prod Consistency**
```
User-Assigned Identity per environment
    ├── Dev Identity → Dev resources
    ├── Test Identity → Test resources
    └── Prod Identity → Prod resources
```

**Benefits:**
- Consistent identity model across environments
- Permissions stay with identity, not individual resources
- Easy to replicate environments

---

## Supported Azure Services

### Services Supporting Managed Identities

| Category | Services |
|----------|----------|
| **Compute** | Virtual Machines, VM Scale Sets, App Service, Functions, Container Instances, AKS |
| **Data** | SQL Database, Synapse Analytics, Data Factory, Data Lake Storage |
| **Analytics** | Azure Databricks, Stream Analytics, HDInsight |
| **Integration** | Logic Apps, API Management, Event Grid |
| **Management** | Automation, Azure DevOps |
| **Security** | Key Vault, App Configuration |

### Services Supporting Microsoft Entra Authentication

Managed identities can authenticate to any service that supports **Microsoft Entra authentication**:

- ✅ Key Vault
- ✅ Azure Storage (Blob, Queue, Table, Files)
- ✅ Azure SQL Database
- ✅ Azure Cosmos DB
- ✅ Azure Service Bus
- ✅ Azure Event Hubs
- ✅ Azure Container Registry
- ✅ Azure Resource Manager
- ✅ Azure Data Lake Storage
- ✅ Azure App Configuration

---

## How Managed Identities Work

### Authentication Flow

```
1. Application requests access to Azure resource
        ↓
2. Azure Instance Metadata Service (IMDS) provides token
   Endpoint: http://169.254.169.254/metadata/identity/oauth2/token
        ↓
3. Microsoft Entra ID validates managed identity
        ↓
4. Microsoft Entra ID issues JWT access token
        ↓
5. Application includes token in Authorization header
   Authorization: Bearer <access_token>
        ↓
6. Azure resource validates token and grants access
```

### Behind the Scenes

**System-Assigned Identity Creation:**
1. Azure Resource Manager receives request to enable identity
2. Service principal created in Microsoft Entra ID
3. Credentials provisioned to Azure Instance Metadata Service
4. Resource can now request tokens from IMDS

**Token Acquisition:**
```bash
# Inside Azure VM/App Service/Function
curl 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://vault.azure.net' \
  -H Metadata:true

# Response:
# {
#   "access_token": "eyJ0eXAiOi...",
#   "expires_in": "3599",
#   "expires_on": "1577836800",
#   "resource": "https://vault.azure.net",
#   "token_type": "Bearer"
# }
```

---

## Role Assignments

After enabling managed identity, you must grant it permissions:

### Grant Access with Azure RBAC

```bash
# Get managed identity principal ID
PRINCIPAL_ID=$(az vm identity show \
  --name myvm \
  --resource-group myresourcegroup \
  --query principalId -o tsv)

# Grant Key Vault Secrets User role
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $PRINCIPAL_ID \
  --scope /subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/myvault

# Grant Storage Blob Data Contributor role
az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee $PRINCIPAL_ID \
  --scope /subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/mystorageaccount
```

### Common RBAC Roles for Managed Identities

| Resource | Role | Permissions |
|----------|------|-------------|
| **Key Vault** | Key Vault Secrets User | Read secrets |
| **Key Vault** | Key Vault Secrets Officer | Manage secrets |
| **Storage** | Storage Blob Data Reader | Read blobs |
| **Storage** | Storage Blob Data Contributor | Read/write blobs |
| **SQL Database** | SQL DB Contributor | Manage databases |
| **Cosmos DB** | Cosmos DB Account Reader | Read Cosmos DB account |

---

## Exam Tips

🎯 **Two types**: System-assigned (tied to resource) vs User-assigned (standalone)

🎯 **System-assigned lifecycle**: Automatic deletion when resource deleted

🎯 **User-assigned lifecycle**: Independent, must be explicitly deleted

🎯 **System-assigned sharing**: Cannot be shared (1:1 relationship)

🎯 **User-assigned sharing**: Can be assigned to multiple resources

🎯 **Service principal**: Managed identities are special service principals

🎯 **Authentication**: Microsoft Entra ID required

🎯 **Token endpoint**: `http://169.254.169.254/metadata/identity/oauth2/token`

🎯 **Best practice**: Use managed identities instead of service principals with secrets

🎯 **Role assignments**: Required after enabling managed identity

🎯 **DefaultAzureCredential**: Automatically uses managed identity when available

🎯 **Supported services**: VM, App Service, Functions, Container Instances, and more

---

## Quick Reference

### Enable System-Assigned Identity
```bash
# VM
az vm identity assign --name myvm --resource-group myrg

# App Service
az webapp identity assign --name myapp --resource-group myrg

# Function
az functionapp identity assign --name myfunc --resource-group myrg
```

### Create and Assign User-Assigned Identity
```bash
# Create
az identity create --name myidentity --resource-group myrg

# Assign to VM
az vm identity assign --name myvm --resource-group myrg --identities myidentity

# Assign to App Service
az webapp identity assign --name myapp --resource-group myrg --identities myidentity
```

### Grant Permissions
```bash
# Get principal ID
PRINCIPAL_ID=$(az vm identity show --name myvm --resource-group myrg --query principalId -o tsv)

# Grant role
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $PRINCIPAL_ID \
  --scope /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/myvault
```

---

## Additional Resources

- [Managed Identities for Azure Resources](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/)
- [Services Supporting Managed Identities](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/services-support-managed-identities)
- [Azure Identity SDK](https://learn.microsoft.com/en-us/dotnet/api/overview/azure/identity-readme)

[Microsoft Learn - Explore managed identities](https://learn.microsoft.com/en-us/training/modules/implement-managed-identities/2-managed-identities-overview)
