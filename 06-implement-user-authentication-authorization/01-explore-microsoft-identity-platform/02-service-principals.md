# Service Principals and Application Objects

## Key Concepts
- **Application Object** - Global template/blueprint
- **Service Principal** - Local instance per tenant
- **Three types** - Application, Managed Identity, Legacy
- **Registration** - Creates both objects automatically

## Overview

**Identity configuration** for applications in Microsoft Entra ID:

- **Application Object** - One per application (home tenant)
- **Service Principal** - One or more per tenant where app is used
- **Relationship** - Application object is template for service principals

## Application Registration

### Registration Process

**Register app in Azure Portal**:

```bash
# Navigate to
Azure Portal → Microsoft Entra ID → App registrations → New registration
```

**Configuration**:

| Setting | Options | Description |
|---------|---------|-------------|
| **Name** | Your app name | Display name for the application |
| **Supported account types** | Single-tenant, Multi-tenant | Who can use the application |
| **Redirect URI** | Web, SPA, Mobile | Where auth responses are sent |

### Tenant Types

**1. Single-Tenant**:
- Accessible only in your tenant
- Most common for line-of-business apps
- More restrictive, more secure

```json
{
  "signInAudience": "AzureADMyOrg"
}
```

**2. Multi-Tenant**:
- Accessible in other tenants
- Common for SaaS applications
- Requires consent in each tenant

```json
{
  "signInAudience": "AzureADMultipleOrgs"
}
```

### Automatic Creation

**When you register an app, Azure creates**:

1. ✅ **Application Object** - In your home tenant
2. ✅ **Service Principal** - In your home tenant
3. ✅ **Application (Client) ID** - Globally unique identifier

## Application Object

### What is an Application Object?

**Global template** for the application:

- **Location** - Home tenant only (where registered)
- **Scope** - One per application (globally unique)
- **Purpose** - Template/blueprint for service principals
- **Properties** - Static configuration applied to all instances

### Three Aspects Defined

| Aspect | Description |
|--------|-------------|
| **Token Issuance** | How the service issues tokens to access the application |
| **Resource Access** | Resources the application needs to access |
| **Actions** | Operations the application can perform |

### Application Object Properties

**Defined via Microsoft Graph [Application entity](https://learn.microsoft.com/en-us/graph/api/resources/application)**:

```json
{
  "id": "00000000-0000-0000-0000-000000000000",
  "appId": "00001111-aaaa-2222-bbbb-3333cccc4444",
  "displayName": "My Application",
  "signInAudience": "AzureADMyOrg",
  "web": {
    "redirectUris": [
      "https://localhost:5001/signin-oidc"
    ],
    "implicitGrantSettings": {
      "enableIdTokenIssuance": true,
      "enableAccessTokenIssuance": false
    }
  },
  "requiredResourceAccess": [
    {
      "resourceAppId": "00000003-0000-0000-c000-000000000000",
      "resourceAccess": [
        {
          "id": "e1fe6dd8-ba31-4d61-89e7-88639da4683d",
          "type": "Scope"
        }
      ]
    }
  ],
  "keyCredentials": [],
  "passwordCredentials": []
}
```

### Key Properties

```csharp
// Reading application object
using Microsoft.Graph;

var graphClient = new GraphServiceClient(...);

var application = await graphClient.Applications["{application-object-id}"]
    .Request()
    .GetAsync();

Console.WriteLine($"App ID: {application.AppId}");
Console.WriteLine($"Display Name: {application.DisplayName}");
Console.WriteLine($"Sign-in Audience: {application.SignInAudience}");
```

## Service Principal Object

### What is a Service Principal?

**Local representation** of application in a tenant:

- **Purpose** - Represents app instance in specific tenant
- **Security Principal** - Defines access policy and permissions
- **Created** - In each tenant where app is used
- **References** - Points to global application object

### Why Service Principals?

**Security principal** for applications (like user principal for users):

```
User → User Principal → Access permissions for user
App  → Service Principal → Access permissions for app
```

**Enables**:
- ✅ Authentication during sign-in
- ✅ Authorization during resource access
- ✅ Access policy definition
- ✅ Permission management

## Three Types of Service Principals

### 1. Application Service Principal

**Standard application instance**:

- **Most common type** - Regular app registrations
- **One per tenant** - Created when app is used in tenant
- **References** - Global application object
- **Defines** - What app can do in specific tenant

**Creation**:

```csharp
// Automatically created when:
// 1. You register the app (in home tenant)
// 2. Admin/user consents to app in their tenant (multi-tenant apps)
```

**Example scenario**:

```
1. Company A registers a multi-tenant SaaS app
   → Application Object created in Company A's tenant
   → Service Principal created in Company A's tenant

2. Company B consents to use the app
   → Service Principal created in Company B's tenant
   → References same Application Object from Company A

3. Company C consents to use the app
   → Service Principal created in Company C's tenant
   → References same Application Object from Company A
```

### 2. Managed Identity Service Principal

**Represents a [Managed Identity](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)**:

- **Purpose** - Identity for Azure resources
- **No credentials** - No passwords or certificates to manage
- **Automatic** - Created when managed identity enabled
- **Cannot modify** - System-managed properties
- **Azure services** - VM, App Service, Functions, etc.

**Types of Managed Identities**:

| Type | Description | Use Case |
|------|-------------|----------|
| **System-Assigned** | Tied to single Azure resource | VM accessing Key Vault |
| **User-Assigned** | Standalone resource, reusable | Multiple VMs sharing identity |

**Example - System-Assigned Managed Identity**:

```bash
# Enable managed identity on Azure VM
az vm identity assign \
    --resource-group MyResourceGroup \
    --name MyVM

# Output includes principalId (service principal object ID)
{
  "principalId": "11111111-1111-1111-1111-111111111111",
  "tenantId": "22222222-2222-2222-2222-222222222222",
  "type": "SystemAssigned"
}
```

**Using in code**:

```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

// Use DefaultAzureCredential - automatically uses managed identity
var client = new SecretClient(
    new Uri("https://myvault.vault.azure.net/"),
    new DefaultAzureCredential()
);

// No credentials needed - managed identity handles authentication
var secret = await client.GetSecretAsync("MySecret");
```

**Benefits**:
- ✅ No credentials in code
- ✅ Automatic rotation
- ✅ Azure-managed lifecycle
- ✅ No permission to modify

### 3. Legacy Service Principal

**Old application** created before modern app registrations:

- **Legacy apps** - Created through old experiences
- **No app registration** - Doesn't have associated app object
- **Can edit** - But deprecated approach
- **Should migrate** - To modern app registrations

**Properties legacy service principals can have**:
- Credentials
- Service principal names
- Reply URLs
- Other properties

**Example (deprecated pattern)**:

```powershell
# Old way (deprecated)
New-AzADServicePrincipal -DisplayName "LegacyApp"

# Modern way (recommended)
New-AzADApplication -DisplayName "ModernApp"
```

## Relationship Between Objects

### One-to-Many Relationship

```
┌─────────────────────────────────────┐
│   Application Object (Global)       │
│   Home Tenant: Contoso              │
│   - App ID: 00001111-aaaa...        │
│   - Display Name: My SaaS App       │
│   - Configuration/Template          │
└──────────────┬──────────────────────┘
               │
               │ Creates/References
               │
    ┌──────────┴─────────────┬─────────────────┐
    │                        │                 │
    ▼                        ▼                 ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ Service         │  │ Service         │  │ Service         │
│ Principal       │  │ Principal       │  │ Principal       │
│ (Contoso)       │  │ (Fabrikam)      │  │ (Woodgrove)     │
│ - Local config  │  │ - Local config  │  │ - Local config  │
│ - Permissions   │  │ - Permissions   │  │ - Permissions   │
└─────────────────┘  └─────────────────┘  └─────────────────┘
    Home Tenant         Tenant 2             Tenant 3
```

### Application Object Relationships

**An application object has**:

- ✅ **One-to-one** - With the software application
- ✅ **One-to-many** - With service principal objects

**Properties**:
- Globally unique
- Lives in home tenant
- Template for all service principals

### Service Principal Creation

**Single-Tenant Application**:

```
1. Register app
   → Application Object created
   → Service Principal created (home tenant only)
   → Consent given during registration

2. Users sign in
   → Use service principal in home tenant
```

**Multi-Tenant Application**:

```
1. Register app (Tenant A)
   → Application Object created in Tenant A
   → Service Principal created in Tenant A

2. User from Tenant B consents
   → Service Principal created in Tenant B
   → References Application Object in Tenant A

3. User from Tenant C consents
   → Service Principal created in Tenant C
   → References Application Object in Tenant A
```

## Creating Service Principals

### Automatic Creation

**When registering in portal**:

```
Azure Portal → Microsoft Entra ID → App registrations → New
→ Both Application Object and Service Principal created automatically
```

### Programmatic Creation

**Using Azure CLI**:

```bash
# Create app registration and service principal
az ad sp create-for-rbac --name "MyApp" \
    --role "Contributor" \
    --scopes /subscriptions/{subscription-id}

# Output
{
  "appId": "00001111-aaaa-2222-bbbb-3333cccc4444",
  "displayName": "MyApp",
  "password": "...",
  "tenant": "22222222-2222-2222-2222-222222222222"
}
```

**Using PowerShell**:

```powershell
# Create application
$app = New-AzADApplication -DisplayName "MyApp"

# Create service principal from application
$sp = New-AzADServicePrincipal -ApplicationId $app.AppId

Write-Host "Application ID: $($app.AppId)"
Write-Host "Service Principal ID: $($sp.Id)"
```

**Using Microsoft Graph API**:

```http
POST https://graph.microsoft.com/v1.0/applications
Content-Type: application/json

{
  "displayName": "My Application",
  "signInAudience": "AzureADMyOrg"
}

# Response includes application object ID

# Then create service principal
POST https://graph.microsoft.com/v1.0/servicePrincipals
Content-Type: application/json

{
  "appId": "00001111-aaaa-2222-bbbb-3333cccc4444"
}
```

## Managing Application Objects

### Read Application

```csharp
using Microsoft.Graph;
using Azure.Identity;

var credential = new ClientSecretCredential(tenantId, clientId, clientSecret);
var graphClient = new GraphServiceClient(credential);

// Get application by object ID
var app = await graphClient.Applications["{object-id}"]
    .Request()
    .GetAsync();

Console.WriteLine($"App ID: {app.AppId}");
Console.WriteLine($"Display Name: {app.DisplayName}");
```

### Update Application

```csharp
var app = await graphClient.Applications["{object-id}"]
    .Request()
    .GetAsync();

// Update properties
app.DisplayName = "Updated Name";
app.Web.RedirectUris.Add("https://new-redirect-uri.com");

await graphClient.Applications["{object-id}"]
    .Request()
    .UpdateAsync(app);
```

### Delete Application

```csharp
// Deleting application also deletes associated service principals
await graphClient.Applications["{object-id}"]
    .Request()
    .DeleteAsync();
```

## Managing Service Principals

### List Service Principals

```bash
# Azure CLI
az ad sp list --display-name "MyApp"

# Get by App ID
az ad sp show --id "00001111-aaaa-2222-bbbb-3333cccc4444"
```

```powershell
# PowerShell
Get-AzADServicePrincipal -DisplayName "MyApp"

# Get by Application ID
Get-AzADServicePrincipal -ApplicationId "00001111-aaaa-2222-bbbb-3333cccc4444"
```

### Assign Roles to Service Principal

```bash
# Azure CLI - Assign Contributor role
az role assignment create \
    --assignee "{service-principal-id}" \
    --role "Contributor" \
    --scope "/subscriptions/{subscription-id}"
```

```powershell
# PowerShell
New-AzRoleAssignment `
    -ObjectId "{service-principal-object-id}" `
    -RoleDefinitionName "Contributor" `
    -Scope "/subscriptions/{subscription-id}"
```

## Best Practices

### 1. Use Managed Identities When Possible

```csharp
// ✅ Good: Use managed identity (no credentials)
var credential = new DefaultAzureCredential();

// ❌ Bad: Use client secret in code
var credential = new ClientSecretCredential(tenantId, clientId, secret);
```

### 2. Single-Tenant for Internal Apps

```
✅ Good: Single-tenant for line-of-business apps
   - More restrictive
   - Better security
   - Simpler management

❌ Bad: Multi-tenant for internal apps
   - Unnecessary complexity
   - Broader attack surface
```

### 3. Descriptive Names

```csharp
// ✅ Good: Clear, descriptive names
DisplayName = "Contoso-HR-System-Production"

// ❌ Bad: Generic names
DisplayName = "App1"
```

### 4. Minimize Permissions

```
✅ Good: Grant only required permissions
   - Read-only when possible
   - Specific scopes

❌ Bad: Grant broad permissions
   - Full control unnecessarily
   - Administrative access by default
```

## Common Patterns

### Pattern 1: Multi-Tenant SaaS

```
1. Register app as multi-tenant
2. Application Object created in your tenant
3. Service Principal created in your tenant
4. Customers consent to app
5. Service Principal created in each customer tenant
6. Each Service Principal has customer-specific permissions
```

### Pattern 2: Managed Identity for Azure Resource

```
1. Enable managed identity on Azure resource (VM, App Service, etc.)
2. Service Principal created automatically
3. No application object needed
4. Assign Azure RBAC roles to service principal
5. Resource uses identity to access other Azure services
```

### Pattern 3: Daemon Application

```
1. Register app as single-tenant
2. Create client secret or certificate
3. Application Object and Service Principal created
4. Grant application permissions (not delegated)
5. Admin consent required
6. App acquires tokens using client credentials
```

## Critical Notes
- 💡 **Application Object** - Global template in home tenant
- 🎯 **Service Principal** - Local instance per tenant
- ✅ **One-to-many** - One app object, many service principals
- ⚠️ **Three types** - Application, Managed Identity, Legacy
- 🔄 **Registration** - Creates both objects automatically
- 📊 **Single-tenant** - One tenant only, more secure
- 💡 **Multi-tenant** - Multiple tenants, requires consent
- ✅ **Managed Identity** - Best for Azure resources (no credentials)
- ⚠️ **Service Principal** - Represents app in tenant
- 🔒 **Security principal** - Defines access policy and permissions
- 🎯 **Automatic creation** - Portal, CLI, PowerShell, Graph API
- 💡 **Azure RBAC** - Assign roles to service principals
- ⚠️ **Best practice** - Use managed identities when possible

## Exam Tips
- Application object: Global template/blueprint in home tenant only
- Service principal: Local representation in each tenant where app is used
- Relationship: One application object → many service principals (one-to-many)
- Three types of service principals: Application, Managed Identity, Legacy
- Application service principal: Most common, represents app instance
- Managed identity service principal: For Azure resources, no credentials to manage
- Legacy service principal: Old apps, should migrate to modern registrations
- Registration creates: Both application object and service principal automatically
- Single-tenant: Accessible only in your tenant (AzureADMyOrg)
- Multi-tenant: Accessible in other tenants (AzureADMultipleOrgs)
- Service principal creation: Automatically when admin/user consents in their tenant
- Managed identities: System-assigned (tied to resource) or User-assigned (standalone)
- DefaultAzureCredential: Automatically uses managed identity in Azure
- Application object defines: Token issuance, resource access, actions
- Security principal: Defines access policy and permissions for user/app
- Microsoft Graph: Use Application entity for programmatic management
- Azure CLI: az ad sp create-for-rbac, az ad sp list
- PowerShell: New-AzADServicePrincipal, Get-AzADServicePrincipal
- Best practice: Use managed identities for Azure resources (no credentials in code)
- RBAC: Assign roles to service principals for Azure resource access

[Learn More](https://learn.microsoft.com/en-us/training/modules/explore-microsoft-identity-platform/3-app-service-principals)
