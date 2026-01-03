# Managed Identities Authentication Flow & Configuration

## Part 1: Authentication Flows

### System-Assigned Managed Identity Flow

**7-step authentication process for Azure Virtual Machines:**

```
Step 1: Enable Identity
   ↓
   Azure Resource Manager receives request to enable system-assigned identity on VM
   
Step 2: Create Service Principal
   ↓
   ARM creates service principal in Microsoft Entra ID
   Service Principal = Identity of the VM in Azure AD
   
Step 3: Configure Identity on VM
   ↓
   ARM configures identity on VM by updating Azure Instance Metadata Service (IMDS)
   IMDS endpoint receives service principal client ID and certificate
   
Step 4: Grant Access to Azure Resources
   ↓
   Use RBAC to assign appropriate role to VM's service principal
   Example: Key Vault Secrets User, Storage Blob Data Contributor
   
Step 5: Request Token from IMDS
   ↓
   Code running on VM requests token from IMDS endpoint
   Endpoint: http://169.254.169.254/metadata/identity/oauth2/token
   Only accessible from within the VM
   
Step 6: Microsoft Entra ID Issues Token
   ↓
   IMDS calls Microsoft Entra ID with client ID and certificate (from Step 3)
   Azure AD validates credentials and returns JWT access token
   
Step 7: Access Azure Resource
   ↓
   Code sends access token in Authorization header
   Azure resource validates token and grants access
```

**Visual Flow:**
```
┌─────────────────────────────────────────────────────────────┐
│ Step 1-3: Setup Phase (One-time)                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Admin → ARM → Enable Identity → Create Service Principal   │
│                 ↓                                            │
│           Configure IMDS with credentials                    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Step 4: Authorization (One-time or as needed)               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Admin → RBAC → Grant roles to service principal            │
│  Example: Key Vault Secrets User, Storage Contributor       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Step 5-7: Runtime Flow (Every access request)               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  App on VM                                                   │
│     ↓ (1) Request token                                     │
│  IMDS (169.254.169.254)                                     │
│     ↓ (2) Forward request with credentials                  │
│  Microsoft Entra ID                                          │
│     ↓ (3) Validate & issue JWT token                        │
│  App on VM                                                   │
│     ↓ (4) Call Azure resource with token                    │
│  Azure Resource (Key Vault, Storage, etc.)                  │
│     ↓ (5) Validate token & return data                      │
│  App on VM ✅                                               │
└─────────────────────────────────────────────────────────────┘
```

**Code Example - Token Request:**
```bash
# From within Azure VM/App Service/Function
curl 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://vault.azure.net' \
  -H Metadata:true

# Response:
{
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
  "expires_in": "3599",
  "expires_on": "1609459200",
  "not_before": "1609455300",
  "resource": "https://vault.azure.net",
  "token_type": "Bearer"
}
```

### User-Assigned Managed Identity Flow

**7-step authentication process:**

```
Step 1: Create User-Assigned Identity
   ↓
   ARM receives request to create standalone user-assigned identity
   
Step 2: Create Service Principal
   ↓
   ARM creates service principal in Microsoft Entra ID
   Identity exists independently of any VM
   
Step 3: Assign Identity to VM
   ↓
   ARM receives request to assign identity to VM
   IMDS updated with user-assigned identity's client ID and certificate
   
Step 4: Grant Access (can be done BEFORE Step 3!)
   ↓
   Assign RBAC roles to user-assigned identity's service principal
   Pre-authorization possible before VM even exists
   
Step 5: Request Token from IMDS
   ↓
   Code specifies which user-assigned identity to use (if multiple)
   Endpoint: http://169.254.169.254/metadata/identity/oauth2/token?client_id=<id>
   
Step 6: Microsoft Entra ID Issues Token
   ↓
   Same process as system-assigned
   
Step 7: Access Azure Resource
   ↓
   Same process as system-assigned
```

**Key Difference: Multiple Identities**
```
VM with Multiple User-Assigned Identities:
   ├─ User-Assigned Identity A (Dev environment permissions)
   ├─ User-Assigned Identity B (Prod environment permissions)
   └─ User-Assigned Identity C (Monitoring permissions)

Request token for specific identity:
curl 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://vault.azure.net&client_id=<Identity-A-Client-ID>' \
  -H Metadata:true
```

---

## Part 2: Configure Managed Identities

### System-Assigned Identity Configuration

#### Enable During VM Creation

```bash
# Create VM with system-assigned identity
az vm create \
    --resource-group myResourceGroup \
    --name myVM \
    --image Ubuntu2204 \
    --assign-identity \
    --role contributor \
    --scope /subscriptions/{subscription-id} \
    --admin-username azureuser \
    --generate-ssh-keys

# What happens:
# 1. VM created
# 2. System-assigned identity enabled automatically
# 3. Service principal created in Azure AD
# 4. Contributor role assigned at subscription scope
```

**Parameters explained:**

| Parameter | Purpose |
|-----------|---------|
| `--assign-identity` | Enable system-assigned managed identity |
| `--role contributor` | RBAC role to assign |
| `--scope /subscriptions/{id}` | Scope of role assignment |

#### Enable on Existing VM

```bash
# Enable system-assigned identity on existing VM
az vm identity assign \
    --resource-group myResourceGroup \
    --name myVM

# Output:
{
  "systemAssignedIdentity": "12345678-1234-1234-1234-123456789abc",
  "userAssignedIdentities": null
}
```

**Grant specific permissions:**
```bash
# Get the principal ID
PRINCIPAL_ID=$(az vm identity show \
    --resource-group myResourceGroup \
    --name myVM \
    --query principalId -o tsv)

# Grant Key Vault Secrets User role
az role assignment create \
    --role "Key Vault Secrets User" \
    --assignee $PRINCIPAL_ID \
    --scope /subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/mykeyvault

# Grant Storage Blob Data Contributor role
az role assignment create \
    --role "Storage Blob Data Contributor" \
    --assignee $PRINCIPAL_ID \
    --scope /subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/mystorageaccount
```

#### Other Azure Resources

**App Service:**
```bash
# Enable system-assigned identity
az webapp identity assign \
    --name myappservice \
    --resource-group myResourceGroup

# Get principal ID
PRINCIPAL_ID=$(az webapp identity show \
    --name myappservice \
    --resource-group myResourceGroup \
    --query principalId -o tsv)
```

**Azure Functions:**
```bash
# Enable system-assigned identity
az functionapp identity assign \
    --name myfunctionapp \
    --resource-group myResourceGroup

# Get principal ID
PRINCIPAL_ID=$(az functionapp identity show \
    --name myfunctionapp \
    --resource-group myResourceGroup \
    --query principalId -o tsv)
```

**Container Instances:**
```bash
# Create container with system-assigned identity
az container create \
    --resource-group myResourceGroup \
    --name mycontainer \
    --image mcr.microsoft.com/azuredocs/aci-helloworld \
    --assign-identity
```

### User-Assigned Identity Configuration

#### Two-Step Process

**Step 1: Create user-assigned identity**
```bash
# Create standalone identity
az identity create \
    --resource-group myResourceGroup \
    --name myUserAssignedIdentity

# Output includes:
# - id: Full resource ID
# - clientId: Used in code
# - principalId: Used for RBAC assignments
```

**Step 2: Assign to resources**
```bash
# Assign to VM during creation
az vm create \
    --resource-group myResourceGroup \
    --name myVM \
    --image Ubuntu2204 \
    --assign-identity myUserAssignedIdentity \
    --role contributor \
    --scope /subscriptions/{subscription-id} \
    --admin-username azureuser \
    --generate-ssh-keys

# OR assign to existing VM
az vm identity assign \
    --resource-group myResourceGroup \
    --name myVM \
    --identities myUserAssignedIdentity
```

#### Assign to Multiple Resources

```bash
# Create one user-assigned identity
az identity create \
    --resource-group myResourceGroup \
    --name SharedAppIdentity

# Get identity details
IDENTITY_ID=$(az identity show \
    --resource-group myResourceGroup \
    --name SharedAppIdentity \
    --query id -o tsv)

PRINCIPAL_ID=$(az identity show \
    --resource-group myResourceGroup \
    --name SharedAppIdentity \
    --query principalId -o tsv)

# Grant permissions ONCE to the identity
az role assignment create \
    --role "Key Vault Secrets User" \
    --assignee $PRINCIPAL_ID \
    --scope /subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/mykeyvault

# Assign same identity to multiple VMs
az vm identity assign --name myVM1 --resource-group myResourceGroup --identities $IDENTITY_ID
az vm identity assign --name myVM2 --resource-group myResourceGroup --identities $IDENTITY_ID
az vm identity assign --name myVM3 --resource-group myResourceGroup --identities $IDENTITY_ID

# All 3 VMs now have same permissions!
```

#### App Service with User-Assigned Identity

```bash
# Create identity
az identity create --resource-group myResourceGroup --name WebAppIdentity

# Assign to App Service
az webapp identity assign \
    --resource-group myResourceGroup \
    --name myappservice \
    --identities WebAppIdentity

# Assign to another App Service
az webapp identity assign \
    --resource-group myResourceGroup \
    --name myappservice2 \
    --identities WebAppIdentity
```

---

## Part 3: Using Managed Identities in Code

### DefaultAzureCredential (.NET)

**Install package:**
```bash
dotnet add package Azure.Identity
dotnet add package Azure.Security.KeyVault.Secrets
```

**Code example:**
```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

// Automatically uses managed identity in Azure
var client = new SecretClient(
    new Uri("https://mykeyvault.vault.azure.net"),
    new DefaultAzureCredential()
);

var secret = await client.GetSecretAsync("MySecret");
Console.WriteLine($"Secret value: {secret.Value.Value}");
```

**How it works:**
- **In Azure**: Uses managed identity automatically
- **Locally**: Uses Azure CLI credentials
- **No code changes** between environments!

### Specify User-Assigned Identity

```csharp
// When VM has multiple user-assigned identities, specify which one
string clientId = "12345678-1234-1234-1234-123456789abc";  // User-assigned identity client ID

var credential = new DefaultAzureCredential(
    new DefaultAzureCredentialOptions 
    { 
        ManagedIdentityClientId = clientId 
    }
);

var client = new SecretClient(
    new Uri("https://mykeyvault.vault.azure.net"),
    credential
);
```

### ChainedTokenCredential (Custom Flow)

```csharp
using Azure.Identity;
using Azure.Messaging.EventHubs;

// Try managed identity first, fall back to Azure CLI
var credential = new ChainedTokenCredential(
    new ManagedIdentityCredential(),  // Try this first
    new AzureCliCredential()           // Fall back to this
);

var client = new EventHubProducerClient(
    "myeventhub.eventhubs.windows.net",
    "myhubpath",
    credential
);
```

### Direct ManagedIdentityCredential

```csharp
// System-assigned identity
var credential = new ManagedIdentityCredential();

// User-assigned identity (specify client ID)
var credential = new ManagedIdentityCredential(clientId: "12345678-...");

// Use with any Azure SDK client
var blobClient = new BlobClient(
    new Uri("https://myaccount.blob.core.windows.net/mycontainer/myblob"),
    credential
);
```

### Python Example

```python
from azure.identity import DefaultAzureCredential, ManagedIdentityCredential
from azure.keyvault.secrets import SecretClient

# Option 1: DefaultAzureCredential (recommended)
credential = DefaultAzureCredential()

# Option 2: Explicit managed identity
credential = ManagedIdentityCredential()

# Option 3: User-assigned identity
credential = ManagedIdentityCredential(client_id="12345678-...")

# Use credential
client = SecretClient(
    vault_url="https://mykeyvault.vault.azure.net",
    credential=credential
)

secret = client.get_secret("MySecret")
print(f"Secret: {secret.value}")
```

---

## Required RBAC Roles

### To Configure Managed Identities

| Operation | Required Role |
|-----------|--------------|
| Enable system-assigned identity | Virtual Machine Contributor |
| Create user-assigned identity | Managed Identity Contributor |
| Assign user-assigned identity to VM | Virtual Machine Contributor + Managed Identity Operator |

### Grant Permissions to Managed Identity

```bash
# General pattern:
az role assignment create \
    --role "<ROLE_NAME>" \
    --assignee <PRINCIPAL_ID> \
    --scope <RESOURCE_SCOPE>
```

**Common roles for managed identities:**

| Resource | Role | Purpose |
|----------|------|---------|
| Key Vault | Key Vault Secrets User | Read secrets |
| Key Vault | Key Vault Secrets Officer | Manage secrets |
| Storage | Storage Blob Data Reader | Read blobs |
| Storage | Storage Blob Data Contributor | Read/write blobs |
| SQL Database | SQL DB Contributor | Manage databases |
| Cosmos DB | Cosmos DB Account Reader Role | Read Cosmos DB data |
| Event Hubs | Azure Event Hubs Data Receiver | Receive events |
| Service Bus | Azure Service Bus Data Receiver | Receive messages |

---

## Azure SDKs Supporting Managed Identities

| Language | Package | Example |
|----------|---------|---------|
| **.NET** | Azure.Identity | [.NET Example](https://github.com/Azure/azure-sdk-for-net/tree/main/sdk/identity/Azure.Identity) |
| **Java** | azure-identity | [Java Example](https://github.com/Azure/azure-sdk-for-java/tree/main/sdk/identity/azure-identity) |
| **Python** | azure-identity | [Python Example](https://github.com/Azure/azure-sdk-for-python/tree/main/sdk/identity/azure-identity) |
| **JavaScript** | @azure/identity | [JS Example](https://github.com/Azure/azure-sdk-for-js/tree/main/sdk/identity/identity) |
| **Go** | azidentity | [Go Example](https://github.com/Azure/azure-sdk-for-go/tree/main/sdk/azidentity) |

---

## Complete Example: VM with Managed Identity

### Setup Script

```bash
#!/bin/bash

# Variables
RG="myResourceGroup"
LOCATION="eastus"
VM_NAME="myVM"
KEYVAULT_NAME="mykv$(openssl rand -hex 4)"

# 1. Create resource group
az group create --name $RG --location $LOCATION

# 2. Create Key Vault
az keyvault create \
    --name $KEYVAULT_NAME \
    --resource-group $RG \
    --location $LOCATION

# 3. Add secret to Key Vault
az keyvault secret set \
    --vault-name $KEYVAULT_NAME \
    --name "DatabasePassword" \
    --value "P@ssw0rd123!"

# 4. Create VM with system-assigned identity
az vm create \
    --resource-group $RG \
    --name $VM_NAME \
    --image Ubuntu2204 \
    --assign-identity \
    --admin-username azureuser \
    --generate-ssh-keys

# 5. Get VM's principal ID
PRINCIPAL_ID=$(az vm identity show \
    --resource-group $RG \
    --name $VM_NAME \
    --query principalId -o tsv)

# 6. Grant Key Vault access
az role assignment create \
    --role "Key Vault Secrets User" \
    --assignee $PRINCIPAL_ID \
    --scope $(az keyvault show --name $KEYVAULT_NAME --resource-group $RG --query id -o tsv)

echo "✓ Setup complete!"
echo "VM: $VM_NAME"
echo "Key Vault: $KEYVAULT_NAME"
echo "Principal ID: $PRINCIPAL_ID"
```

### Application Code (Python on VM)

```python
# app.py
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient
import os

# Automatically uses VM's managed identity
credential = DefaultAzureCredential()

# Connect to Key Vault
vault_url = "https://mykv1234.vault.azure.net"
client = SecretClient(vault_url=vault_url, credential=credential)

# Retrieve secret
secret = client.get_secret("DatabasePassword")
print(f"Retrieved password: {secret.value}")

# Use secret to connect to database
# connection_string = f"Server=myserver;Password={secret.value}"
```

---

## Exam Tips

🎯 **System-assigned flow**: 7 steps from enable to access

🎯 **User-assigned flow**: Identity created separately, then assigned to resources

🎯 **IMDS endpoint**: `http://169.254.169.254/metadata/identity/oauth2/token`

🎯 **IMDS accessible**: Only from within the Azure resource (VM, App Service, etc.)

🎯 **Token format**: JWT (JSON Web Token)

🎯 **Token expiration**: Typically 3599 seconds (1 hour)

🎯 **Pre-authorization**: User-assigned identities can be granted permissions before assignment

🎯 **DefaultAzureCredential**: Recommended - tries managed identity first

🎯 **Client ID**: Required when multiple user-assigned identities on same resource

🎯 **Required roles**: Virtual Machine Contributor, Managed Identity Operator

🎯 **Code changes**: None needed between local dev and Azure production

🎯 **Multiple identities**: User-assigned can be shared; system-assigned cannot

---

## Additional Resources

- [Managed Identities Documentation](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/)
- [Azure Identity SDK](https://learn.microsoft.com/en-us/dotnet/api/overview/azure/identity-readme)
- [IMDS Documentation](https://learn.microsoft.com/en-us/azure/virtual-machines/instance-metadata-service)

[Microsoft Learn - Managed Identities Authentication Flow & Configuration](https://learn.microsoft.com/en-us/training/modules/implement-managed-identities/)
