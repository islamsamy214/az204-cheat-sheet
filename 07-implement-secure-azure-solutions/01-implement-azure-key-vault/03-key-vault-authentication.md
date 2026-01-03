# Authenticate to Azure Key Vault

## Overview

Authentication with Key Vault works through **Microsoft Entra ID** (formerly Azure Active Directory), which authenticates the identity of any **security principal** requesting access to Azure resources.

---

## What is a Security Principal?

A **security principal** is an entity that can request access to Azure resources.

| Type | Description | Examples |
|------|-------------|----------|
| **User** | Real people with accounts in Microsoft Entra ID | alice@contoso.com, bob@contoso.com |
| **Group** | Collections of users | Developers, Admins, Data Scientists |
| **Service Principal** | Represents apps or services (not people) | Web app, API, background job |
| **Managed Identity** | Automatically managed service principal | VM, App Service, Function App |

**Think of service principals as:** User accounts for applications instead of people.

---

## Service Principal Creation Methods

For applications, there are **two main ways** to obtain a service principal:

### 1. Managed Identity (Recommended) ✅

**How it works:**
- Azure creates and manages the service principal automatically
- No credentials to store or rotate
- Integrated with Azure Identity libraries
- Works with Azure services: App Service, Functions, VMs, Container Instances, etc.

**Types:**

| Type | Lifecycle | Use Case |
|------|-----------|----------|
| **System-assigned** | Tied to resource (deleted with resource) | Single resource, simple scenarios |
| **User-assigned** | Independent lifecycle | Multiple resources, cross-resource scenarios |

**Benefits:**
- ✅ Zero credential management
- ✅ Automatic rotation
- ✅ No secrets in code or config
- ✅ Azure handles everything

**Example - Enable system-assigned managed identity:**
```bash
# For App Service
az webapp identity assign \
  --name myappservice \
  --resource-group myresourcegroup

# For Virtual Machine
az vm identity assign \
  --name myvm \
  --resource-group myresourcegroup

# For Azure Function
az functionapp identity assign \
  --name myfunctionapp \
  --resource-group myresourcegroup
```

**Grant Key Vault access:**
```bash
# Get the managed identity principal ID
PRINCIPAL_ID=$(az webapp identity show \
  --name myappservice \
  --resource-group myresourcegroup \
  --query principalId -o tsv)

# Grant Key Vault Secrets User role
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $PRINCIPAL_ID \
  --scope /subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/mykeyvault
```

### 2. Manually Register Application

**How it works:**
- Register app in Microsoft Entra ID manually
- Creates service principal and app object
- App object identifies the app across tenants
- You manage credentials (certificate or secret)

**When to use:**
- Non-Azure environments (on-premises, other clouds)
- Multi-tenant applications
- Managed identity not available

**Steps:**
```bash
# 1. Create app registration
az ad app create --display-name "MyApplication"

# 2. Create service principal
az ad sp create-for-rbac \
  --name "MyApplication" \
  --role "Key Vault Secrets User" \
  --scopes /subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/mykeyvault

# Output (SAVE THIS - shown only once):
# {
#   "appId": "12345678-1234-1234-1234-123456789abc",
#   "displayName": "MyApplication",
#   "password": "super-secret-password",
#   "tenant": "87654321-4321-4321-4321-876543219876"
# }
```

**Credential options:**

| Credential Type | Security | Rotation | Recommendation |
|----------------|----------|----------|----------------|
| **Certificate** | High | Manual | ✅ Recommended |
| **Client Secret** | Medium | Manual | ⚠️ Use with caution |

---

## Authentication Methods Comparison

| Method | Security | Management | Use Case |
|--------|----------|------------|----------|
| **System-Assigned Managed Identity** | ⭐⭐⭐⭐⭐ | Automatic | Single Azure resource |
| **User-Assigned Managed Identity** | ⭐⭐⭐⭐⭐ | Automatic | Multiple Azure resources |
| **Service Principal + Certificate** | ⭐⭐⭐⭐ | Manual | Non-Azure, multi-tenant |
| **Service Principal + Secret** | ⭐⭐⭐ | Manual | Last resort |

**Decision tree:**
```
Running in Azure?
├─ YES → Use Managed Identity ✅
│   └─ Single resource? → System-assigned
│       Multiple resources? → User-assigned
└─ NO → Running on-premises/other cloud?
    └─ Use Service Principal
        └─ Certificate > Secret
```

---

## Authentication in Application Code

### Azure Identity Client Libraries

Key Vault SDK uses **Azure Identity client library** for seamless authentication across environments with the same code.

**Available SDKs:**

| Language | Package | Latest Version |
|----------|---------|----------------|
| **.NET** | Azure.Identity | 1.10+ |
| **Python** | azure-identity | 1.14+ |
| **Java** | azure-identity | 1.10+ |
| **JavaScript** | @azure/identity | 4.0+ |

### DefaultAzureCredential

**The recommended authentication method** - tries multiple credential sources automatically:

**Credential chain (in order):**

1. **EnvironmentCredential** - Environment variables
2. **WorkloadIdentityCredential** - Kubernetes workload identity
3. **ManagedIdentityCredential** - Managed identity (VM, App Service, etc.)
4. **SharedTokenCacheCredential** - Shared token cache
5. **VisualStudioCredential** - Visual Studio authentication
6. **VisualStudioCodeCredential** - VS Code authentication
7. **AzureCliCredential** - Azure CLI authentication
8. **AzurePowerShellCredential** - Azure PowerShell authentication
9. **AzureDeveloperCliCredential** - Azure Developer CLI

**Benefits:**
- ✅ Works in development (Azure CLI) and production (managed identity)
- ✅ No code changes between environments
- ✅ Falls back automatically if one method fails

### .NET Example

```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

// DefaultAzureCredential tries multiple auth methods
var client = new SecretClient(
    new Uri("https://mykeyvault.vault.azure.net"),
    new DefaultAzureCredential()
);

// Get secret
KeyVaultSecret secret = await client.GetSecretAsync("MySecret");
Console.WriteLine($"Secret value: {secret.Value}");
```

**Install packages:**
```bash
dotnet add package Azure.Identity
dotnet add package Azure.Security.KeyVault.Secrets
```

**How it works:**
```
DefaultAzureCredential attempts:
1. Environment variables? ❌ Not set
2. Managed Identity? ✅ Found (App Service)
   → Uses managed identity to authenticate
   → Gets access token from Azure Instance Metadata Service (IMDS)
   → Access token used to call Key Vault API
```

### Python Example

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

# DefaultAzureCredential handles authentication
credential = DefaultAzureCredential()
client = SecretClient(
    vault_url="https://mykeyvault.vault.azure.net",
    credential=credential
)

# Get secret
secret = client.get_secret("MySecret")
print(f"Secret value: {secret.value}")
```

**Install packages:**
```bash
pip install azure-identity azure-keyvault-secrets
```

### Java Example

```java
import com.azure.identity.DefaultAzureCredentialBuilder;
import com.azure.security.keyvault.secrets.SecretClient;
import com.azure.security.keyvault.secrets.SecretClientBuilder;
import com.azure.security.keyvault.secrets.models.KeyVaultSecret;

// DefaultAzureCredential for authentication
SecretClient client = new SecretClientBuilder()
    .vaultUrl("https://mykeyvault.vault.azure.net")
    .credential(new DefaultAzureCredentialBuilder().build())
    .buildClient();

// Get secret
KeyVaultSecret secret = client.getSecret("MySecret");
System.out.println("Secret value: " + secret.getValue());
```

**Maven dependency:**
```xml
<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-identity</artifactId>
    <version>1.10.0</version>
</dependency>
<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-security-keyvault-secrets</artifactId>
    <version>4.6.0</version>
</dependency>
```

### JavaScript/TypeScript Example

```javascript
const { DefaultAzureCredential } = require("@azure/identity");
const { SecretClient } = require("@azure/keyvault-secrets");

// DefaultAzureCredential for authentication
const credential = new DefaultAzureCredential();
const client = new SecretClient(
    "https://mykeyvault.vault.azure.net",
    credential
);

// Get secret
async function getSecret() {
    const secret = await client.getSecret("MySecret");
    console.log(`Secret value: ${secret.value}`);
}

getSecret();
```

**Install packages:**
```bash
npm install @azure/identity @azure/keyvault-secrets
```

---

## Authentication with REST API

Access tokens must be sent to Key Vault using the **HTTP Authorization header**.

### Request Format

```http
PUT /keys/MYKEY?api-version=7.4 HTTP/1.1
Host: mykeyvault.vault.azure.net
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "kty": "RSA",
  "key_size": 2048
}
```

### Get Access Token

**Using Azure CLI:**
```bash
# Get access token for Key Vault
ACCESS_TOKEN=$(az account get-access-token \
  --resource https://vault.azure.net \
  --query accessToken -o tsv)

echo $ACCESS_TOKEN
```

**Using PowerShell:**
```powershell
# Get access token
$token = (Get-AzAccessToken -ResourceUrl "https://vault.azure.net").Token
```

**Using cURL:**
```bash
# Get secret using access token
curl -X GET "https://mykeyvault.vault.azure.net/secrets/MySecret?api-version=7.4" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

### Unauthorized Response (401)

When access token is missing or invalid, Key Vault returns **HTTP 401** with `WWW-Authenticate` header:

```http
HTTP/1.1 401 Not Authorized
WWW-Authenticate: Bearer authorization="https://login.microsoftonline.com/tenant-id", resource="https://vault.azure.net"
```

**Header parameters:**

| Parameter | Description |
|-----------|-------------|
| **authorization** | OAuth2 authorization service URL to obtain access token |
| **resource** | Resource identifier (`https://vault.azure.net`) for authorization request |

**Example - Get token manually:**
```bash
# Extract tenant ID from WWW-Authenticate header
TENANT_ID="your-tenant-id"

# Get access token using service principal
curl -X POST "https://login.microsoftonline.com/$TENANT_ID/oauth2/v2.0/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=your-client-id" \
  -d "client_secret=your-client-secret" \
  -d "scope=https://vault.azure.net/.default" \
  -d "grant_type=client_credentials"

# Response:
# {
#   "token_type": "Bearer",
#   "expires_in": 3599,
#   "access_token": "eyJ0eXAiOiJKV1QiLCJhbGc..."
# }
```

---

## Authentication Scenarios

### Scenario 1: Azure App Service

**Setup:**
```bash
# 1. Enable managed identity
az webapp identity assign \
  --name myappservice \
  --resource-group myresourcegroup

# 2. Get principal ID
PRINCIPAL_ID=$(az webapp identity show \
  --name myappservice \
  --resource-group myresourcegroup \
  --query principalId -o tsv)

# 3. Grant Key Vault access
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $PRINCIPAL_ID \
  --scope /subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/mykeyvault
```

**Code (.NET):**
```csharp
// Automatically uses App Service managed identity
var client = new SecretClient(
    new Uri("https://mykeyvault.vault.azure.net"),
    new DefaultAzureCredential()
);

var secret = await client.GetSecretAsync("ConnectionString");
```

### Scenario 2: Azure Functions

**Setup:**
```bash
# 1. Enable managed identity
az functionapp identity assign \
  --name myfunctionapp \
  --resource-group myresourcegroup

# 2. Grant access (same as App Service)
```

**Code (.NET - Function):**
```csharp
[FunctionName("GetSecret")]
public static async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Function, "get")] HttpRequest req,
    ILogger log)
{
    var client = new SecretClient(
        new Uri("https://mykeyvault.vault.azure.net"),
        new DefaultAzureCredential()
    );

    var secret = await client.GetSecretAsync("ApiKey");
    return new OkObjectResult($"Secret retrieved: {secret.Value.Value}");
}
```

### Scenario 3: Virtual Machine

**Setup:**
```bash
# 1. Enable managed identity on VM
az vm identity assign \
  --name myvm \
  --resource-group myresourcegroup

# 2. Grant Key Vault access (same as above)
```

**Code (Python on VM):**
```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

# Automatically uses VM managed identity
credential = DefaultAzureCredential()
client = SecretClient(
    vault_url="https://mykeyvault.vault.azure.net",
    credential=credential
)

secret = client.get_secret("DatabasePassword")
print(f"Retrieved secret: {secret.value}")
```

### Scenario 4: Local Development

**Setup Azure CLI authentication:**
```bash
# Login to Azure CLI
az login

# Set subscription
az account set --subscription "my-subscription"
```

**Code (same code works locally and in Azure):**
```csharp
// In Azure: Uses managed identity
// Locally: Uses Azure CLI credentials
var client = new SecretClient(
    new Uri("https://mykeyvault.vault.azure.net"),
    new DefaultAzureCredential()
);

var secret = await client.GetSecretAsync("MySecret");
```

**How DefaultAzureCredential works locally:**
```
DefaultAzureCredential attempts:
1. Environment variables? ❌ Not set
2. Managed Identity? ❌ Not in Azure
3. Shared Token Cache? ❌ Not available
4. Visual Studio? ❌ Not signed in
5. VS Code? ❌ Not signed in
6. Azure CLI? ✅ Authenticated!
   → Uses Azure CLI token
   → Same code works in Azure and locally!
```

### Scenario 5: Service Principal (Non-Azure)

**Setup:**
```bash
# Create service principal with certificate
az ad sp create-for-rbac \
  --name "MyOnPremApp" \
  --create-cert \
  --cert MyAppCert \
  --keyvault mykeyvault \
  --role "Key Vault Secrets User" \
  --scopes /subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/mykeyvault
```

**Code (on-premises application):**
```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

// Use service principal with certificate
var credential = new ClientCertificateCredential(
    tenantId: "your-tenant-id",
    clientId: "your-client-id",
    certificatePath: "/path/to/cert.pfx",
    certificatePassword: "cert-password"  // if encrypted
);

var client = new SecretClient(
    new Uri("https://mykeyvault.vault.azure.net"),
    credential
);

var secret = await client.GetSecretAsync("ApiKey");
```

---

## Exam Tips

🎯 **Microsoft Entra ID**: Required for all Key Vault authentication

🎯 **Security principal types**: User, Group, Service Principal, Managed Identity

🎯 **Two ways to get service principal**: Managed identity (recommended) or manual registration

🎯 **System-assigned managed identity**: Recommended for Azure resources

🎯 **DefaultAzureCredential**: Best practice - tries multiple auth methods automatically

🎯 **Credential chain**: Environment → Managed Identity → Azure CLI → others

🎯 **Azure Identity libraries**: Available for .NET, Python, Java, JavaScript

🎯 **REST API**: Requires Bearer token in Authorization header

🎯 **HTTP 401**: Returned when token missing/invalid, includes WWW-Authenticate header

🎯 **Resource URL**: `https://vault.azure.net` for token requests

🎯 **Local development**: Use Azure CLI authentication with DefaultAzureCredential

🎯 **No code changes**: Same code works locally (Azure CLI) and in Azure (managed identity)

---

## Additional Resources

- [Azure Key Vault Developer's Guide](https://learn.microsoft.com/en-us/azure/key-vault/general/developers-guide)
- [Azure Identity Documentation](https://learn.microsoft.com/en-us/dotnet/api/overview/azure/identity-readme)
- [DefaultAzureCredential Class](https://learn.microsoft.com/en-us/dotnet/api/azure.identity.defaultazurecredential)
- [Managed Identities for Azure Resources](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/)

[Microsoft Learn - Authenticate to Azure Key Vault](https://learn.microsoft.com/en-us/training/modules/implement-azure-key-vault/4-key-vault-authentication)
