# Connect Functions to Azure Services

## Key Concepts
- **Application settings** - Encrypted key-value pairs for config
- **Connection strings** - Stored as app settings, not hardcoded
- **Identity-based connections** - Use managed identity instead of secrets
- **Managed identity** - System-assigned or user-assigned identity
- **Least privilege** - Grant minimal required permissions

## Application Settings

### Purpose
**Securely store** configuration values:
- Connection strings
- API keys
- Service endpoints
- Feature flags
- Environment-specific config

### Storage
✅ **Encrypted at rest** - Azure encrypts values
✅ **Environment variables** - Accessed at runtime as env vars
✅ **Per environment** - Different values for dev/test/prod

### Configuration Sources

#### Azure (Production)
```
Function App → Configuration → Application settings
├── Name: DatabaseConnection
├── Value: Server=prod-server;Database=...
└── Deployment slot setting: ☐ (optional)
```

#### Local Development
```json
// local.settings.json
{
  "Values": {
    "DatabaseConnection": "Server=localhost;Database=...",
    "ApiKey": "dev-key-123",
    "ExternalApiUrl": "https://dev-api.example.com"
  }
}
```

## Connection String Pattern

### ❌ Don't: Hardcode Connections
```csharp
// BAD - Never do this
public static void Run([QueueTrigger("orders")] string message)
{
    var connectionString = "DefaultEndpointsProtocol=https;AccountName=...";
    // Use connection string
}
```

**Problems**:
- Security risk
- Can't change without redeployment
- No separation between environments

### ✅ Do: Use App Settings
```csharp
// GOOD - Reference app setting
public static void Run(
    [QueueTrigger("orders", Connection = "MyStorageConnection")] string message)
{
    // Connection retrieved from app setting automatically
}
```

### Binding Connection Property
```json
{
  "type": "queueTrigger",
  "direction": "in",
  "name": "message",
  "queueName": "orders",
  "connection": "MyStorageConnection"
}
```

**How it works**:
1. Function looks for app setting named `MyStorageConnection`
2. Retrieves value at runtime
3. Uses value to connect to service

### Set App Settings

#### Portal
```
1. Navigate to Function App
2. Configuration → Application settings
3. Click "+ New application setting"
4. Name: MyStorageConnection
5. Value: DefaultEndpointsProtocol=https;...
6. OK → Save
```

#### CLI
```bash
# Set single setting
az functionapp config appsettings set \
  --name <function-app-name> \
  --resource-group <rg-name> \
  --settings "MyStorageConnection=DefaultEndpointsProtocol=https;..."

# Set multiple settings
az functionapp config appsettings set \
  --name <function-app-name> \
  --resource-group <rg-name> \
  --settings \
    "DatabaseConnection=Server=..." \
    "ApiKey=prod-key-456" \
    "FeatureFlag=enabled"
```

#### Upload from local.settings.json
```bash
# Upload all settings at once
func azure functionapp publish <function-app-name> --publish-settings-only
```

## Identity-Based Connections

### What Is Identity-Based Connection?
**Use managed identity** instead of connection strings/secrets:

Traditional (secret-based):
```
Connection = "AccountKey=supersecretkey123..."
```

Identity-based (no secrets):
```
Connection = "MyStorageConnection"
MyStorageConnection__serviceUri = "https://mystorageaccount.blob.core.windows.net"
```

### Benefits
✅ **No secrets** - No keys to rotate or leak
✅ **Azure AD authentication** - Better security
✅ **Least privilege** - RBAC controls access
✅ **Automatic rotation** - No manual key management

### Supported Services
| Service | Identity Support | Extension |
|---------|------------------|-----------|
| **Azure Storage** | ✅ Yes | Blobs, Queues, Tables |
| **Azure Cosmos DB** | ✅ Yes | SQL API |
| **Azure Service Bus** | ✅ Yes | Queues, Topics |
| **Azure Event Hubs** | ✅ Yes | Event streams |
| **Azure SQL Database** | ✅ Yes | SQL connections |

⚠️ **Azure Files Exception**: Storage account for function app itself (`WEBSITE_AZUREFILESCONNECTIONSTRING`) must use connection string

### Configuration

#### Step 1: Enable Managed Identity
```bash
# Enable system-assigned identity
az functionapp identity assign \
  --name <function-app-name> \
  --resource-group <rg-name>

# Output includes principalId:
# {
#   "principalId": "12345678-1234-1234-1234-123456789012",
#   "tenantId": "...",
#   "type": "SystemAssigned"
# }
```

#### Step 2: Configure Connection Settings
```bash
# Storage Blob identity-based connection
az functionapp config appsettings set \
  --name <function-app-name> \
  --resource-group <rg-name> \
  --settings \
    "MyStorageConnection__serviceUri=https://mystorageaccount.blob.core.windows.net" \
    "MyStorageConnection__credential=managedidentity"
```

**Setting format**:
```
<ConnectionName>__serviceUri = <service-endpoint>
<ConnectionName>__credential = managedidentity
<ConnectionName>__clientId = <client-id>  (optional, for user-assigned)
```

#### Step 3: Grant Permissions
```bash
# Get function app identity
PRINCIPAL_ID=$(az functionapp identity show \
  --name <function-app-name> \
  --resource-group <rg-name> \
  --query principalId -o tsv)

# Grant Storage Blob Data Contributor role
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<storage-account>"
```

### User-Assigned Identity
```bash
# Create user-assigned identity
az identity create \
  --name MyFunctionIdentity \
  --resource-group <rg-name>

# Assign to function app
az functionapp identity assign \
  --name <function-app-name> \
  --resource-group <rg-name> \
  --identities <identity-resource-id>

# Configure connection with clientId
az functionapp config appsettings set \
  --name <function-app-name> \
  --resource-group <rg-name> \
  --settings \
    "MyConnection__serviceUri=https://..." \
    "MyConnection__credential=managedidentity" \
    "MyConnection__clientId=<user-assigned-identity-client-id>"
```

## Common Azure Service Connections

### Azure Storage (Blob)

#### Connection String Method
```bash
az functionapp config appsettings set \
  --name <function-app-name> \
  --resource-group <rg-name> \
  --settings "MyStorageConnection=DefaultEndpointsProtocol=https;AccountName=mystorageaccount;AccountKey=..."
```

#### Identity Method
```bash
# Configure settings
az functionapp config appsettings set \
  --name <function-app-name> \
  --resource-group <rg-name> \
  --settings "MyStorageConnection__serviceUri=https://mystorageaccount.blob.core.windows.net"

# Grant role
az role assignment create \
  --assignee <function-app-principal-id> \
  --role "Storage Blob Data Contributor" \
  --scope <storage-account-resource-id>
```

### Azure Cosmos DB

#### Connection String Method
```bash
az functionapp config appsettings set \
  --name <function-app-name> \
  --resource-group <rg-name> \
  --settings "CosmosDBConnection=AccountEndpoint=https://mycosmosdb.documents.azure.com:443/;AccountKey=..."
```

#### Identity Method
```bash
# Configure settings
az functionapp config appsettings set \
  --name <function-app-name> \
  --resource-group <rg-name> \
  --settings "CosmosDBConnection__accountEndpoint=https://mycosmosdb.documents.azure.com"

# Grant role (Cosmos DB Built-in Data Contributor)
az cosmosdb sql role assignment create \
  --account-name <cosmos-account> \
  --resource-group <rg-name> \
  --role-definition-name "Cosmos DB Built-in Data Contributor" \
  --principal-id <function-app-principal-id> \
  --scope "/"
```

### Azure Service Bus

#### Connection String Method
```bash
az functionapp config appsettings set \
  --name <function-app-name> \
  --resource-group <rg-name> \
  --settings "ServiceBusConnection=Endpoint=sb://myservicebus.servicebus.windows.net/;SharedAccessKeyName=..."
```

#### Identity Method
```bash
# Configure settings
az functionapp config appsettings set \
  --name <function-app-name> \
  --resource-group <rg-name> \
  --settings "ServiceBusConnection__fullyQualifiedNamespace=myservicebus.servicebus.windows.net"

# Grant role
az role assignment create \
  --assignee <function-app-principal-id> \
  --role "Azure Service Bus Data Receiver" \
  --scope <service-bus-namespace-resource-id>
```

### Azure SQL Database

#### Connection String Method
```bash
az functionapp config appsettings set \
  --name <function-app-name> \
  --resource-group <rg-name> \
  --settings "SqlConnection=Server=tcp:myserver.database.windows.net,1433;Database=..."
```

#### Identity Method
```csharp
// In code, use managed identity token
var connectionString = $"Server=tcp:myserver.database.windows.net;Database=mydb;";
var connection = new SqlConnection(connectionString);
connection.AccessToken = await new DefaultAzureCredential().GetTokenAsync(
    new TokenRequestContext(new[] { "https://database.windows.net/.default" }));
```

```bash
# Grant SQL permissions
# In SQL Database:
# CREATE USER [function-app-name] FROM EXTERNAL PROVIDER;
# ALTER ROLE db_datareader ADD MEMBER [function-app-name];
# ALTER ROLE db_datawriter ADD MEMBER [function-app-name];
```

## Local Development with Identity

### DefaultAzureCredential
**Automatic credential chain** for local dev:

1. Environment variables
2. Managed identity (when running in Azure)
3. Visual Studio
4. Azure CLI
5. Azure PowerShell

### Local Testing
```bash
# Login with Azure CLI (for local dev)
az login

# Function uses your Azure CLI credentials locally
# In Azure, switches to managed identity automatically
```

### Code Example (C#)
```csharp
using Azure.Identity;
using Azure.Storage.Blobs;

[FunctionName("BlobFunction")]
public static async Task Run(
    [HttpTrigger(AuthorizationLevel.Function, "get")] HttpRequest req,
    ILogger log)
{
    var credential = new DefaultAzureCredential();
    var blobServiceClient = new BlobServiceClient(
        new Uri("https://mystorageaccount.blob.core.windows.net"),
        credential);
    
    // Use client
}
```

## Required Permissions (RBAC Roles)

### Azure Storage
| Operation | Role |
|-----------|------|
| **Read blobs** | Storage Blob Data Reader |
| **Write blobs** | Storage Blob Data Contributor |
| **Read queues** | Storage Queue Data Reader |
| **Write queues** | Storage Queue Data Contributor |
| **Read tables** | Storage Table Data Reader |
| **Write tables** | Storage Table Data Contributor |

### Azure Cosmos DB
| Operation | Role |
|-----------|------|
| **Read data** | Cosmos DB Built-in Data Reader |
| **Write data** | Cosmos DB Built-in Data Contributor |

### Azure Service Bus
| Operation | Role |
|-----------|------|
| **Receive messages** | Azure Service Bus Data Receiver |
| **Send messages** | Azure Service Bus Data Sender |
| **Full access** | Azure Service Bus Data Owner |

### Azure Event Hubs
| Operation | Role |
|-----------|------|
| **Receive events** | Azure Event Hubs Data Receiver |
| **Send events** | Azure Event Hubs Data Sender |
| **Full access** | Azure Event Hubs Data Owner |

## Best Practices

### 1. Use Identity-Based Connections
✅ **Preferred**: Managed identity
⚠️ **Fallback**: Connection strings (when identity not supported)

### 2. Separate Environments
```
Dev:   MyStorage → dev-storage-account
Test:  MyStorage → test-storage-account
Prod:  MyStorage → prod-storage-account
```

### 3. Least Privilege
Grant **minimum required** permissions:
- Read-only if function only reads
- Specific queue/container scope if possible

### 4. Key Vault References
For secrets that must be stored:
```bash
# Store in Key Vault, reference in app settings
az functionapp config appsettings set \
  --name <function-app-name> \
  --resource-group <rg-name> \
  --settings "ApiKey=@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/ApiKey/)"
```

### 5. Never Commit Secrets
```
# .gitignore
local.settings.json
*.publish.xml
*.user
```

## Troubleshooting

### Connection Issues
```bash
# Check app settings
az functionapp config appsettings list \
  --name <function-app-name> \
  --resource-group <rg-name>

# Check managed identity
az functionapp identity show \
  --name <function-app-name> \
  --resource-group <rg-name>

# Check role assignments
az role assignment list \
  --assignee <principal-id> \
  --output table
```

### Common Errors

#### "Identity not found"
✅ Enable managed identity on function app

#### "Insufficient permissions"
✅ Grant appropriate RBAC role

#### "Connection string missing"
✅ Add app setting with correct name

#### "Identity not supported locally"
✅ Use `az login` or set environment variables

## Critical Notes
- 💡 **App settings** - Encrypted key-value pairs for configuration
- ⚠️ **Connection property** - References app setting NAME, not value
- 🎯 **Identity-based** - Preferred over connection strings
- 📊 **Managed identity** - System-assigned or user-assigned
- ✅ **Least privilege** - Grant minimal required permissions
- 🔄 **DefaultAzureCredential** - Works locally and in Azure
- ⏱️ **RBAC roles** - Required for identity-based connections
- 🔒 **Never commit** - local.settings.json to source control

## Exam Tips
- Application settings stored encrypted, accessed as environment variables
- Connection property in binding references app setting name
- Never hardcode connection strings in code
- Identity-based connections use managed identity (no secrets)
- System-assigned identity: Tied to function app lifecycle
- User-assigned identity: Independent lifecycle, reusable
- DefaultAzureCredential: Automatic credential chain for local/Azure
- Azure Files exception: Must use connection string for function app storage
- Setting format: `<Name>__serviceUri`, `<Name>__credential`
- RBAC roles: Storage Blob Data Contributor, Cosmos DB Built-in Data Contributor
- Grant least privilege permissions only
- Key Vault reference: `@Microsoft.KeyVault(SecretUri=...)`
- Local testing: Use `az login` or Azurite emulator

[Learn More](https://learn.microsoft.com/en-us/training/modules/develop-azure-functions/4-connect-azure-services)
