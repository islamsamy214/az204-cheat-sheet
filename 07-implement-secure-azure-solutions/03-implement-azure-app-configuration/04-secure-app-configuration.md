# Secure App Configuration Data

## Overview

Securing Azure App Configuration data involves multiple layers of protection: authentication, authorization, encryption, and network isolation. This unit covers security features and best practices to protect your application configuration.

---

## Security Layers

```
┌───────────────────────────────────────────────────────────┐
│  Network Security                                          │
│  • Private Endpoints                                       │
│  • Firewall Rules                                          │
│  • Virtual Network Integration                             │
└───────────────┬───────────────────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────────────────┐
│  Authentication                                            │
│  • Managed Identities (Recommended)                        │
│  • Azure AD Service Principals                             │
│  • Connection Strings (Development Only)                   │
└───────────────┬───────────────────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────────────────┐
│  Authorization                                             │
│  • Azure RBAC Roles                                        │
│  • App Configuration Data Owner                            │
│  • App Configuration Data Reader                           │
└───────────────┬───────────────────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────────────────┐
│  Encryption                                                │
│  • Microsoft-Managed Keys (Default)                        │
│  • Customer-Managed Keys (CMK)                             │
│  • Data encrypted at rest and in transit                   │
└───────────────────────────────────────────────────────────┘
```

---

## Managed Identities

### What Are Managed Identities?

A **managed identity** from Microsoft Entra ID (formerly Azure AD) allows Azure App Configuration to access other protected resources (like Azure Key Vault) without managing credentials.

**Benefits**:
- No credentials in code or configuration
- Automatic credential rotation
- Seamless integration with Azure services
- Azure platform manages identity lifecycle

### Types of Managed Identities

| Type | Lifecycle | Sharing | Use Case |
|------|-----------|---------|----------|
| **System-Assigned** | Tied to App Configuration store | Cannot share | Simple scenarios, 1:1 relationship |
| **User-Assigned** | Independent resource | Can share across multiple stores | Multiple stores, cross-region scenarios |

---

## Add System-Assigned Identity

### When to Use

- Single App Configuration store
- Simple authentication scenario
- Identity lifecycle tied to store
- No need to share identity

### Setup with Azure CLI

```bash
# Variables
RESOURCE_GROUP="rg-appconfig"
APP_CONFIG_NAME="myappconfig"

# Assign system-assigned identity
az appconfig identity assign \
  --name $APP_CONFIG_NAME \
  --resource-group $RESOURCE_GROUP
```

**Output**:
```json
{
  "principalId": "12345678-1234-1234-1234-123456789012",
  "tenantId": "87654321-4321-4321-4321-210987654321",
  "type": "SystemAssigned"
}
```

### Grant Access to Key Vault

After enabling identity, grant permissions to access Key Vault:

```bash
# Get App Configuration identity Principal ID
PRINCIPAL_ID=$(az appconfig identity show \
  --name $APP_CONFIG_NAME \
  --resource-group $RESOURCE_GROUP \
  --query principalId -o tsv)

# Grant access to Key Vault
KEY_VAULT_NAME="myvault"

az keyvault set-policy \
  --name $KEY_VAULT_NAME \
  --object-id $PRINCIPAL_ID \
  --secret-permissions get list
```

**Alternative: Using Azure RBAC** (Recommended):
```bash
# Get Key Vault resource ID
VAULT_ID=$(az keyvault show --name $KEY_VAULT_NAME --query id -o tsv)

# Assign role
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Key Vault Secrets User" \
  --scope $VAULT_ID
```

### Verify Identity

```bash
# Show identity details
az appconfig identity show \
  --name $APP_CONFIG_NAME \
  --resource-group $RESOURCE_GROUP
```

---

## Add User-Assigned Identity

### When to Use

- Multiple App Configuration stores
- Share identity across stores
- Cross-region scenarios
- Independent lifecycle management

### Setup with Azure CLI

#### Step 1: Create User-Assigned Identity

```bash
# Variables
IDENTITY_NAME="id-appconfig-reader"
RESOURCE_GROUP="rg-identities"
LOCATION="eastus"

# Create identity
az identity create \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME \
  --location $LOCATION
```

**Output**:
```json
{
  "clientId": "abcdef12-ab12-ab12-ab12-abcdef123456",
  "id": "/subscriptions/{sub-id}/resourcegroups/rg-identities/providers/Microsoft.ManagedIdentity/userAssignedIdentities/id-appconfig-reader",
  "location": "eastus",
  "name": "id-appconfig-reader",
  "principalId": "98765432-9876-9876-9876-987654321098",
  "resourceGroup": "rg-identities",
  "type": "Microsoft.ManagedIdentity/userAssignedIdentities"
}
```

#### Step 2: Get Identity Resource ID

```bash
IDENTITY_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME \
  --query id -o tsv)

echo $IDENTITY_ID
# Output: /subscriptions/{sub-id}/resourcegroups/rg-identities/providers/Microsoft.ManagedIdentity/userAssignedIdentities/id-appconfig-reader
```

#### Step 3: Assign Identity to App Configuration

```bash
# Assign to App Configuration store
az appconfig identity assign \
  --name myappconfig \
  --resource-group rg-appconfig \
  --identities $IDENTITY_ID
```

#### Step 4: Grant Permissions

```bash
# Get principal ID
PRINCIPAL_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME \
  --query principalId -o tsv)

# Grant Key Vault access
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Key Vault Secrets User" \
  --scope "/subscriptions/{sub-id}/resourceGroups/rg-keyvault/providers/Microsoft.KeyVault/vaults/myvault"
```

### Assign to Multiple Stores

```bash
# Assign same identity to multiple App Configuration stores
az appconfig identity assign --name appconfig-dev --identities $IDENTITY_ID
az appconfig identity assign --name appconfig-staging --identities $IDENTITY_ID
az appconfig identity assign --name appconfig-prod --identities $IDENTITY_ID
```

---

## Customer-Managed Encryption Keys (CMK)

### Default Encryption

By default, Azure App Configuration encrypts all data at rest using **Microsoft-managed keys**:
- 256-bit AES encryption
- Each store has its own encryption key
- Managed by Microsoft
- No additional configuration needed

### Customer-Managed Keys (CMK)

For additional control, use **customer-managed keys** stored in Azure Key Vault.

**Benefits**:
- Full control over encryption keys
- Key rotation control
- Compliance requirements
- Ability to revoke access

**Requirements**:
- ✅ **Standard tier** App Configuration instance
- ✅ Azure Key Vault with **soft-delete** enabled
- ✅ Azure Key Vault with **purge-protection** enabled
- ✅ RSA or RSA-HSM key in Key Vault
- ✅ Key must be enabled
- ✅ Key must have **wrap** and **unwrap** capabilities

### How CMK Works

1. **App Configuration** gets a managed identity
2. **Managed identity** is granted `GET`, `WRAP`, and `UNWRAP` permissions on Key Vault
3. **App Configuration** calls Key Vault to wrap its encryption key
4. **Wrapped encryption key** is stored
5. **Unwrapped key** is cached for 1 hour
6. **App Configuration refreshes** unwrapped key hourly

```
┌──────────────────────────────────────────────────────┐
│  Azure App Configuration                              │
│  ┌──────────────────────────────────────────┐        │
│  │  Configuration Data                      │        │
│  │  (encrypted with Data Encryption Key)    │        │
│  └──────────────────────────────────────────┘        │
│  ┌──────────────────────────────────────────┐        │
│  │  Data Encryption Key (DEK)               │        │
│  │  (wrapped by Key Encryption Key)         │◄───────┼──┐
│  └──────────────────────────────────────────┘        │  │
│  ┌──────────────────────────────────────────┐        │  │
│  │  Managed Identity                        │────────┼──┤
│  │  (authenticates to Key Vault)            │        │  │
│  └──────────────────────────────────────────┘        │  │
└──────────────────────────────────────────────────────┘  │
                                                           │
                     ┌─────────────────────────────────────┘
                     │
                     ▼
         ┌───────────────────────────────┐
         │  Azure Key Vault              │
         │  ┌─────────────────────────┐  │
         │  │  Key Encryption Key     │  │
         │  │  (RSA or RSA-HSM)       │  │
         │  │  • Wrap capability      │  │
         │  │  • Unwrap capability    │  │
         │  └─────────────────────────┘  │
         └───────────────────────────────┘
```

### Enable CMK

#### Step 1: Create Key Vault with Required Features

```bash
# Variables
KEY_VAULT_NAME="kv-appconfig-cmk"
RESOURCE_GROUP="rg-appconfig"
LOCATION="eastus"

# Create Key Vault with soft-delete and purge-protection
az keyvault create \
  --name $KEY_VAULT_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --enable-soft-delete true \
  --enable-purge-protection true
```

#### Step 2: Create RSA Key

```bash
# Create RSA key with wrap and unwrap capabilities
az keyvault key create \
  --vault-name $KEY_VAULT_NAME \
  --name appconfig-encryption-key \
  --kty RSA \
  --size 2048 \
  --ops wrapKey unwrapKey
```

#### Step 3: Enable Managed Identity on App Configuration

```bash
APP_CONFIG_NAME="myappconfig"

# Assign system-assigned identity
az appconfig identity assign \
  --name $APP_CONFIG_NAME \
  --resource-group $RESOURCE_GROUP

# Get principal ID
PRINCIPAL_ID=$(az appconfig identity show \
  --name $APP_CONFIG_NAME \
  --resource-group $RESOURCE_GROUP \
  --query principalId -o tsv)
```

#### Step 4: Grant Key Vault Permissions

```bash
# Grant GET, WRAP, and UNWRAP permissions
az keyvault set-policy \
  --name $KEY_VAULT_NAME \
  --object-id $PRINCIPAL_ID \
  --key-permissions get wrapKey unwrapKey
```

#### Step 5: Configure App Configuration to Use CMK

```bash
# Get Key Vault key identifier
KEY_IDENTIFIER=$(az keyvault key show \
  --vault-name $KEY_VAULT_NAME \
  --name appconfig-encryption-key \
  --query key.kid -o tsv)

# Enable CMK on App Configuration
az appconfig update \
  --name $APP_CONFIG_NAME \
  --resource-group $RESOURCE_GROUP \
  --encryption-key-name appconfig-encryption-key \
  --encryption-key-vault-uri https://${KEY_VAULT_NAME}.vault.azure.net \
  --identity-client-id $(az appconfig identity show \
    --name $APP_CONFIG_NAME \
    --resource-group $RESOURCE_GROUP \
    --query principalId -o tsv)
```

### Key Rotation

**Automatic rotation** when key version changes:
```bash
# Create new key version
az keyvault key create \
  --vault-name $KEY_VAULT_NAME \
  --name appconfig-encryption-key \
  --kty RSA \
  --size 2048

# App Configuration automatically detects and uses new version within 1 hour
```

---

## Private Endpoints

### What Are Private Endpoints?

**Private endpoints** allow clients on a virtual network to securely access App Configuration over a **private link** using a private IP address from the VNet.

**Benefits**:
- Eliminate exposure to public internet
- Secure data from VNet or on-premises networks
- No data exfiltration risk
- Comply with network security policies

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Virtual Network (10.0.0.0/16)                          │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Subnet: app-subnet (10.0.1.0/24)                 │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐        │  │
│  │  │ Web App  │  │ Function │  │   VM     │        │  │
│  │  │          │  │   App    │  │          │        │  │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘        │  │
│  │       │             │              │              │  │
│  │       └─────────────┴──────────────┘              │  │
│  │                     │                             │  │
│  │                     ▼                             │  │
│  └───────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Subnet: pe-subnet (10.0.2.0/24)                  │  │
│  │  ┌──────────────────────────────────────┐         │  │
│  │  │  Private Endpoint                    │         │  │
│  │  │  Private IP: 10.0.2.4                │         │  │
│  │  │  DNS: myappconfig.privatelink...     │         │  │
│  │  └────────────────┬─────────────────────┘         │  │
│  │                   │                               │  │
│  └───────────────────┼───────────────────────────────┘  │
└────────────────────┼─────────────────────────────────────┘
                     │ Private Link
                     ▼
   ┌──────────────────────────────────────┐
   │  Azure App Configuration             │
   │  • Public endpoint: DISABLED         │
   │  • Private endpoint: ENABLED         │
   └──────────────────────────────────────┘
```

### Create Private Endpoint

#### Prerequisites

- Virtual Network with dedicated subnet
- App Configuration Standard tier

#### Step 1: Create Subnet for Private Endpoint

```bash
# Variables
VNET_NAME="vnet-prod"
SUBNET_NAME="subnet-pe-appconfig"
RESOURCE_GROUP="rg-network"

# Create subnet with private endpoint network policies disabled
az network vnet subnet create \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $SUBNET_NAME \
  --address-prefixes 10.0.2.0/24 \
  --disable-private-endpoint-network-policies true
```

#### Step 2: Create Private Endpoint

```bash
# Variables
PE_NAME="pe-appconfig"
APP_CONFIG_NAME="myappconfig"
APP_CONFIG_RG="rg-appconfig"

# Get App Configuration resource ID
APP_CONFIG_ID=$(az appconfig show \
  --name $APP_CONFIG_NAME \
  --resource-group $APP_CONFIG_RG \
  --query id -o tsv)

# Create private endpoint
az network private-endpoint create \
  --name $PE_NAME \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --subnet $SUBNET_NAME \
  --private-connection-resource-id $APP_CONFIG_ID \
  --group-id configurationStores \
  --connection-name appconfig-connection
```

#### Step 3: Configure Private DNS Zone

```bash
# Create private DNS zone
az network private-dns zone create \
  --resource-group $RESOURCE_GROUP \
  --name privatelink.azconfig.io

# Link DNS zone to VNet
az network private-dns link vnet create \
  --resource-group $RESOURCE_GROUP \
  --zone-name privatelink.azconfig.io \
  --name appconfig-dns-link \
  --virtual-network $VNET_NAME \
  --registration-enabled false

# Create DNS zone group (automatic DNS records)
az network private-endpoint dns-zone-group create \
  --resource-group $RESOURCE_GROUP \
  --endpoint-name $PE_NAME \
  --name appconfig-zone-group \
  --private-dns-zone privatelink.azconfig.io \
  --zone-name privatelink.azconfig.io
```

#### Step 4: Disable Public Access (Optional)

```bash
# Disable public network access
az appconfig update \
  --name $APP_CONFIG_NAME \
  --resource-group $APP_CONFIG_RG \
  --enable-public-network false
```

### Verify Private Endpoint

```bash
# From a VM in the VNet
nslookup myappconfig.azconfig.io

# Should resolve to private IP (10.0.2.4)
# Instead of public IP
```

---

## Public Access Firewall Rules

If private endpoints are not used, configure firewall rules to restrict public access.

### Disable Public Access Completely

```bash
az appconfig update \
  --name myappconfig \
  --resource-group rg-appconfig \
  --enable-public-network false
```

### Allow Specific IP Ranges

```bash
# Enable public access (required before adding rules)
az appconfig update \
  --name myappconfig \
  --resource-group rg-appconfig \
  --enable-public-network true

# Add IP rule (single IP)
az appconfig network-rule add \
  --name myappconfig \
  --resource-group rg-appconfig \
  --ip-address 203.0.113.25

# Add IP rule (CIDR range)
az appconfig network-rule add \
  --name myappconfig \
  --resource-group rg-appconfig \
  --ip-address 203.0.113.0/24

# Allow Azure services
az appconfig update \
  --name myappconfig \
  --resource-group rg-appconfig \
  --bypass AzureServices
```

### List Network Rules

```bash
az appconfig network-rule list \
  --name myappconfig \
  --resource-group rg-appconfig
```

### Remove Network Rule

```bash
az appconfig network-rule remove \
  --name myappconfig \
  --resource-group rg-appconfig \
  --ip-address 203.0.113.25
```

---

## Azure RBAC Roles

Azure App Configuration uses Azure RBAC for authorization.

### Built-in Roles

| Role | Permissions | Use Case |
|------|-------------|----------|
| **App Configuration Data Owner** | Read, write, delete configuration data | Administrators, CI/CD pipelines |
| **App Configuration Data Reader** | Read configuration data | Applications, services (recommended) |
| **App Configuration** (Contributor) | Manage App Configuration resource | Infrastructure management |

### Grant Access to Application

```bash
# Variables
APP_CONFIG_NAME="myappconfig"
RESOURCE_GROUP="rg-appconfig"
APP_NAME="mywebapp"

# Get application's managed identity principal ID
PRINCIPAL_ID=$(az webapp identity show \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query principalId -o tsv)

# Get App Configuration resource ID
APP_CONFIG_ID=$(az appconfig show \
  --name $APP_CONFIG_NAME \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

# Grant Data Reader role
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "App Configuration Data Reader" \
  --scope $APP_CONFIG_ID
```

### Grant Access to User

```bash
# Grant to specific user
az role assignment create \
  --assignee user@contoso.com \
  --role "App Configuration Data Owner" \
  --scope $APP_CONFIG_ID
```

### Grant Access to Service Principal

```bash
# Get service principal app ID
SP_APP_ID="12345678-1234-1234-1234-123456789012"

# Grant access
az role assignment create \
  --assignee $SP_APP_ID \
  --role "App Configuration Data Reader" \
  --scope $APP_CONFIG_ID
```

---

## Authenticate Applications

### Option 1: Managed Identity (Recommended)

**.NET Example**:
```csharp
using Azure.Identity;
using Microsoft.Extensions.Configuration.AzureAppConfiguration;

var builder = WebApplication.CreateBuilder(args);

// Use managed identity
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(
        new Uri("https://myappconfig.azconfig.io"),
        new DefaultAzureCredential());
});

var app = builder.Build();
app.Run();
```

**Requirements**:
1. Enable managed identity on application (Web App, VM, Function)
2. Grant "App Configuration Data Reader" role to identity

### Option 2: Connection String (Development Only)

```bash
# Get connection string
CONNECTION_STRING=$(az appconfig credential list \
  --name myappconfig \
  --resource-group rg-appconfig \
  --query "[?name=='Primary'].connectionString" -o tsv)

# Set as environment variable
export APP_CONFIG_CONNECTION_STRING="$CONNECTION_STRING"
```

**.NET Example**:
```csharp
builder.Configuration.AddAzureAppConfiguration(
    Environment.GetEnvironmentVariable("APP_CONFIG_CONNECTION_STRING"));
```

⚠️ **Warning**: Connection strings contain secrets. Use only for local development.

### Option 3: Service Principal (CI/CD)

```bash
# Create service principal
SP=$(az ad sp create-for-rbac --name "AppConfigReader" --json)
SP_APP_ID=$(echo $SP | jq -r .appId)
SP_PASSWORD=$(echo $SP | jq -r .password)
SP_TENANT=$(echo $SP | jq -r .tenant)

# Grant access
az role assignment create \
  --assignee $SP_APP_ID \
  --role "App Configuration Data Reader" \
  --scope $APP_CONFIG_ID

# Use in CI/CD
export AZURE_CLIENT_ID=$SP_APP_ID
export AZURE_CLIENT_SECRET=$SP_PASSWORD
export AZURE_TENANT_ID=$SP_TENANT
```

**.NET Example**:
```csharp
var credential = new ClientSecretCredential(
    Environment.GetEnvironmentVariable("AZURE_TENANT_ID"),
    Environment.GetEnvironmentVariable("AZURE_CLIENT_ID"),
    Environment.GetEnvironmentVariable("AZURE_CLIENT_SECRET"));

builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(new Uri("https://myappconfig.azconfig.io"), credential);
});
```

---

## Best Practices

### 1. **Use Managed Identities**

✅ **Recommended**:
```csharp
options.Connect(new Uri("https://myappconfig.azconfig.io"), 
               new DefaultAzureCredential());
```

❌ **Avoid**:
```csharp
options.Connect(connectionString); // Contains secrets
```

### 2. **Apply Least Privilege**

- Applications: Grant **Data Reader** (read-only)
- CI/CD pipelines: Grant **Data Owner** (read-write)
- Administrators: Use Azure AD groups

### 3. **Use Private Endpoints for Production**

```bash
# Disable public access
az appconfig update --name myappconfig --enable-public-network false

# Use private endpoint from VNet
```

### 4. **Enable Customer-Managed Keys for Compliance**

Required for:
- Healthcare (HIPAA)
- Financial services (PCI-DSS)
- Government (FedRAMP)

### 5. **Separate Stores per Environment**

```
appconfig-dev    (Standard tier, public access)
appconfig-staging (Standard tier, private endpoint)
appconfig-prod   (Standard tier, private endpoint, CMK)
```

### 6. **Audit Access**

```bash
# Enable diagnostic logs
az monitor diagnostic-settings create \
  --name appconfig-logs \
  --resource $APP_CONFIG_ID \
  --workspace $LOG_ANALYTICS_WORKSPACE_ID \
  --logs '[{"category":"HttpRequest","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'
```

---

## Exam Tips

### Key Concepts for AZ-204

1. **Managed identities** eliminate need for credentials in code

2. **System-assigned**: Tied to App Configuration store, deleted with store

3. **User-assigned**: Independent resource, can be shared across stores

4. **CMK requirements**: Standard tier, soft-delete, purge-protection, RSA key

5. **Private endpoints**: Secure access over private link from VNet

6. **RBAC roles**: Data Owner (read/write), Data Reader (read-only)

7. **Connection strings**: Only for development, contain secrets

8. **DefaultAzureCredential**: Tries managed identity, then other methods

9. **Public access firewall**: Allow specific IP ranges

10. **CMK refresh**: Unwrapped key cached for 1 hour, refreshed hourly

11. **Private DNS zone**: `privatelink.azconfig.io`

12. **Key permissions for CMK**: GET, WRAP, UNWRAP

### Common Exam Scenarios

**Scenario 1**: "Web App needs to read App Configuration"
→ **Answer**: Enable managed identity on Web App, grant "App Configuration Data Reader" role

**Scenario 2**: "Secure App Configuration from public internet"
→ **Answer**: Create private endpoint, disable public access

**Scenario 3**: "Comply with regulation requiring customer-controlled encryption"
→ **Answer**: Use customer-managed keys in Key Vault

**Scenario 4**: "CI/CD pipeline needs to update configuration"
→ **Answer**: Use service principal with "App Configuration Data Owner" role

**Scenario 5**: "Share identity across multiple App Configuration stores"
→ **Answer**: Use user-assigned managed identity

---

## Quick Reference Commands

```bash
# Enable system-assigned identity
az appconfig identity assign --name <name> --resource-group <rg>

# Create user-assigned identity
az identity create --name <name> --resource-group <rg>

# Assign user-assigned identity
az appconfig identity assign --name <name> --identities <identity-id>

# Grant RBAC role
az role assignment create \
  --assignee <principal-id> \
  --role "App Configuration Data Reader" \
  --scope <appconfig-resource-id>

# Create private endpoint
az network private-endpoint create \
  --name <pe-name> \
  --vnet-name <vnet> \
  --subnet <subnet> \
  --private-connection-resource-id <appconfig-id> \
  --group-id configurationStores

# Disable public access
az appconfig update --name <name> --enable-public-network false

# Add firewall rule
az appconfig network-rule add --name <name> --ip-address <ip-or-cidr>

# Get connection string
az appconfig credential list --name <name> --resource-group <rg>
```

---

## Learn More

- [Managed Identities with App Configuration](https://docs.microsoft.com/azure/azure-app-configuration/howto-integrate-azure-managed-service-identity)
- [Use customer-managed keys](https://docs.microsoft.com/azure/azure-app-configuration/concept-customer-managed-keys)
- [Use private endpoints](https://docs.microsoft.com/azure/azure-app-configuration/concept-private-endpoint)
