# Azure App Configuration Service Overview

## What is Azure App Configuration?

Azure App Configuration is a **fully managed service** that provides centralized management of application settings and feature flags. It solves the challenge of managing configuration across distributed cloud applications.

### The Problem It Solves

Modern cloud applications often consist of multiple distributed components (microservices, serverless functions, containers) running across different environments. Managing configuration across these components leads to:

- **Configuration sprawl**: Settings scattered across multiple deployment files
- **Hard-to-troubleshoot errors**: Inconsistent configurations between environments
- **Deployment coupling**: Need to redeploy applications to change settings
- **Security concerns**: Credentials stored in code or configuration files

### The Solution

App Configuration provides a centralized configuration store with:
- Single source of truth for application settings
- Dynamic configuration updates without redeployment
- Feature flag management for controlled feature rollouts
- Integration with Azure Key Vault for sensitive data

---

## Key Benefits

### 1. **Fully Managed Service**
- Set up in minutes with Azure Portal or CLI
- No infrastructure to maintain
- Built-in high availability and scalability
- Microsoft-managed updates and patching

### 2. **Flexible Key Representations**
- Hierarchical key naming: `AppName:Service1:ApiEndpoint`
- Label-based variants: Same key, different values per environment
- Query patterns for bulk retrieval
- Unicode character support

### 3. **Tagging with Labels**
- Environment segmentation (dev, staging, production)
- Version management (v1.0, v2.0)
- Feature branch configurations
- A/B testing scenarios

### 4. **Point-in-Time Replay**
- Configuration snapshots at specific timestamps
- Audit historical changes
- Rollback to previous configurations
- Compliance and troubleshooting support

### 5. **Feature Flag Management**
- Dedicated UI for feature management
- Percentage-based rollouts
- Targeted releases (user groups, regions)
- Real-time feature control

### 6. **Enhanced Security**
- Integration with Azure Managed Identities
- Azure RBAC for access control
- Private endpoints for network isolation
- Encryption at rest and in transit
- Customer-managed encryption keys (CMK)

### 7. **Native Framework Integration**
- .NET Configuration Provider
- Spring Cloud integration for Java
- JavaScript/Node.js support
- Python provider
- REST API for custom implementations

---

## Common Use Cases

### Centralized Configuration Management

**Scenario**: Microservices architecture with 20+ services across multiple regions

```yaml
# Hierarchical organization
MyApp:Database:ConnectionString
MyApp:Cache:RedisEndpoint
MyApp:Logging:Level
MyApp:Feature:EnableNewUI

# Environment-specific with labels
Key: MyApp:Database:ConnectionString, Label: Development
Key: MyApp:Database:ConnectionString, Label: Production
```

**Benefits**:
- Single update affects all services
- Consistent configuration across instances
- Easy environment promotion

### Dynamic Configuration Updates

**Scenario**: Change logging level without restarting application

**Traditional Approach**:
1. Edit configuration file
2. Build new container image
3. Deploy to production
4. Wait for pods to restart
5. **Downtime**: 5-10 minutes

**With App Configuration**:
1. Update value in App Configuration
2. Application detects change (polling or push)
3. Apply new configuration dynamically
4. **Downtime**: 0 minutes

```csharp
// .NET Core example with automatic refresh
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(connectionString)
           .ConfigureRefresh(refresh =>
           {
               refresh.Register("MyApp:Logging:Level")
                      .SetCacheExpiration(TimeSpan.FromSeconds(30));
           });
});
```

### Feature Flag Management

**Scenario**: Gradual rollout of new checkout process

**Implementation**:
```json
{
  "FeatureManagement": {
    "NewCheckout": {
      "EnabledFor": [
        {
          "Name": "Percentage",
          "Parameters": { "Value": 10 }
        }
      ]
    }
  }
}
```

**Rollout Plan**:
- Day 1: Enable for 10% of users
- Day 3: Increase to 50% if metrics are good
- Day 7: Enable for 100%
- No code deployment needed

### Multi-Environment Configuration

**Scenario**: Consistent configuration structure across environments with environment-specific values

```bash
# Development
Key: MyApp:ApiEndpoint, Label: dev
Value: https://api-dev.contoso.com

# Staging
Key: MyApp:ApiEndpoint, Label: staging
Value: https://api-staging.contoso.com

# Production
Key: MyApp:ApiEndpoint, Label: prod
Value: https://api.contoso.com
```

**Application Code**:
```csharp
// Select environment automatically
string environment = builder.Environment.EnvironmentName;
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(connectionString)
           .Select(KeyFilter.Any, environment);
});
```

---

## App Configuration vs. Key Vault

| Aspect | App Configuration | Key Vault |
|--------|------------------|-----------|
| **Primary Purpose** | Application settings and feature flags | Secrets, keys, and certificates |
| **Data Sensitivity** | Non-sensitive configuration | Sensitive data (passwords, keys) |
| **Size Limit** | 10 KB per key-value | 25 KB for secrets |
| **Access Patterns** | Frequent reads, bulk retrieval | Infrequent reads, individual access |
| **Feature Flags** | ✅ Native support | ❌ Not designed for this |
| **Dynamic Refresh** | ✅ Built-in support | ⚠️ Manual polling |
| **Best For** | Connection strings, endpoints, settings | Passwords, certificates, API keys |

### Integration Pattern (Recommended)

**Use both services together**:

```csharp
// Store non-sensitive config in App Configuration
MyApp:Database:Server=sql.database.windows.net
MyApp:Database:Name=productiondb

// Store sensitive data in Key Vault
MyApp:Database:Password --> Key Vault Reference

// App Configuration stores reference
{
  "uri": "https://myvault.vault.azure.net/secrets/DbPassword"
}
```

**Benefits**:
- Centralized configuration in App Configuration
- Secrets secured in Key Vault
- Unified access pattern for developers

---

## Client Libraries

App Configuration provides native libraries for popular frameworks:

| Language/Framework | Package | Documentation |
|-------------------|---------|---------------|
| **.NET Core** | `Microsoft.Extensions.Configuration.AzureAppConfiguration` | [Docs](https://docs.microsoft.com/azure/azure-app-configuration/quickstart-dotnet-core-app) |
| **ASP.NET Core** | `Microsoft.Azure.AppConfiguration.AspNetCore` | [Docs](https://docs.microsoft.com/azure/azure-app-configuration/quickstart-aspnet-core-app) |
| **.NET Framework** | `Microsoft.Configuration.ConfigurationBuilders.AzureAppConfiguration` | [Docs](https://docs.microsoft.com/azure/azure-app-configuration/quickstart-dotnet-app) |
| **Java Spring** | `spring-cloud-azure-appconfiguration-config` | [Docs](https://docs.microsoft.com/azure/azure-app-configuration/quickstart-java-spring-app) |
| **JavaScript/Node.js** | `@azure/app-configuration` | [Docs](https://docs.microsoft.com/azure/azure-app-configuration/quickstart-javascript) |
| **Python** | `azure-appconfiguration` | [Docs](https://docs.microsoft.com/azure/azure-app-configuration/quickstart-python) |
| **REST API** | Direct HTTPS | [API Reference](https://docs.microsoft.com/rest/api/appconfiguration/) |

---

## Pricing Tiers

### Free Tier
- **Cost**: Free
- **Requests**: 1,000 per day
- **Storage**: 10 MB
- **Best For**: Development, testing, small applications

### Standard Tier
- **Cost**: $1.20 per day + $0.06 per 10,000 requests
- **Requests**: Unlimited
- **Storage**: 1 GB included (additional storage extra)
- **Features**:
  - Soft delete (7-90 days retention)
  - Customer-managed keys (CMK)
  - Private endpoints
  - 99.9% SLA
- **Best For**: Production applications

**Example Cost Calculation** (Standard Tier):
```
Base cost: $1.20/day × 30 days = $36/month
Requests: 10 million/month = 1,000 × 10,000 requests
Request cost: 1,000 × $0.06 = $60/month
Total: $96/month
```

---

## Quick Start Example

### Create App Configuration Store

```bash
# Variables
RESOURCE_GROUP="rg-appconfig"
LOCATION="eastus"
CONFIG_STORE_NAME="appconfig-myapp-prod"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create App Configuration store (Standard tier)
az appconfig create \
  --name $CONFIG_STORE_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard
```

### Add Configuration Values

```bash
# Add key-values
az appconfig kv set \
  --name $CONFIG_STORE_NAME \
  --key "MyApp:Database:Server" \
  --value "sql.database.windows.net"

# Add with label for environment
az appconfig kv set \
  --name $CONFIG_STORE_NAME \
  --key "MyApp:ApiEndpoint" \
  --value "https://api-dev.contoso.com" \
  --label "Development"

az appconfig kv set \
  --name $CONFIG_STORE_NAME \
  --key "MyApp:ApiEndpoint" \
  --value "https://api.contoso.com" \
  --label "Production"
```

### .NET Application Integration

```csharp
using Microsoft.Extensions.Configuration;
using Azure.Identity;

var builder = WebApplication.CreateBuilder(args);

// Get connection string from environment variable
string connectionString = Environment.GetEnvironmentVariable("APP_CONFIG_CONNECTION_STRING");

// Add Azure App Configuration
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(connectionString)
           .Select(KeyFilter.Any, builder.Environment.EnvironmentName)
           .ConfigureRefresh(refresh =>
           {
               refresh.Register("MyApp:Sentinel", refreshAll: true)
                      .SetCacheExpiration(TimeSpan.FromMinutes(5));
           });
});

var app = builder.Build();

// Access configuration
var apiEndpoint = app.Configuration["MyApp:ApiEndpoint"];
Console.WriteLine($"API Endpoint: {apiEndpoint}");

app.Run();
```

---

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Applications                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Web App  │  │ Function │  │   VM     │  │Container │   │
│  │          │  │   App    │  │          │  │ Instance │   │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘  └─────┬────┘   │
│        │             │              │              │         │
│        └─────────────┴──────────────┴──────────────┘         │
│                           │                                  │
│                           ▼                                  │
│              ┌────────────────────────┐                      │
│              │  Managed Identity      │                      │
│              │  Authentication        │                      │
│              └────────────┬───────────┘                      │
└───────────────────────────┼──────────────────────────────────┘
                            │
                            ▼
          ┌─────────────────────────────────────┐
          │  Azure App Configuration           │
          │  ┌────────────────────────────┐    │
          │  │   Configuration Data       │    │
          │  │   • Key-Value Pairs        │    │
          │  │   • Feature Flags          │    │
          │  │   • Labels (environments)  │    │
          │  └────────────────────────────┘    │
          │  ┌────────────────────────────┐    │
          │  │   Key Vault References     │────┼──┐
          │  │   • Connection strings     │    │  │
          │  │   • API keys               │    │  │
          │  └────────────────────────────┘    │  │
          └─────────────────────────────────────┘  │
                                                    │
                                                    ▼
                                    ┌───────────────────────┐
                                    │  Azure Key Vault      │
                                    │  • Secrets            │
                                    │  • Certificates       │
                                    │  • Keys               │
                                    └───────────────────────┘
```

### Request Flow

1. **Application Startup**:
   - Application initializes with connection string or managed identity
   - Configuration provider connects to App Configuration
   - Initial configuration loaded into memory

2. **Configuration Retrieval**:
   - Application requests configuration values
   - Provider checks local cache
   - If cache expired, fetches from App Configuration
   - Key Vault references resolved automatically

3. **Dynamic Refresh** (Optional):
   - Provider polls App Configuration for changes
   - Sentinel key monitored for bulk refresh
   - Updated values propagated to application
   - No restart required

---

## Security Features

### Authentication Options

1. **Managed Identity** (Recommended)
   ```bash
   # Enable system-assigned managed identity on App Service
   az webapp identity assign --name myapp --resource-group rg
   
   # Grant access to App Configuration
   az role assignment create \
     --assignee <principal-id> \
     --role "App Configuration Data Reader" \
     --scope /subscriptions/<sub-id>/resourceGroups/rg/providers/Microsoft.AppConfiguration/configurationStores/myappconfig
   ```

2. **Connection String** (Development)
   ```bash
   # Get connection string
   az appconfig credential list --name myappconfig --resource-group rg
   
   # Use in application
   export APP_CONFIG_CONNECTION_STRING="Endpoint=https://myappconfig.azconfig.io;Id=xxx;Secret=xxx"
   ```

3. **Azure AD Service Principal** (CI/CD)
   ```bash
   # Create service principal
   az ad sp create-for-rbac --name "AppConfigReader"
   
   # Assign role
   az role assignment create \
     --assignee <app-id> \
     --role "App Configuration Data Reader" \
     --scope <config-store-resource-id>
   ```

### Network Security

**Private Endpoints**:
```bash
# Create private endpoint
az network private-endpoint create \
  --name pe-appconfig \
  --resource-group rg-network \
  --vnet-name vnet-prod \
  --subnet subnet-appconfig \
  --private-connection-resource-id <config-store-resource-id> \
  --group-id configurationStores \
  --connection-name appconfig-connection
```

**Public Access Firewall**:
```bash
# Disable public access
az appconfig update \
  --name myappconfig \
  --resource-group rg \
  --enable-public-network false

# Allow specific IP ranges
az appconfig update \
  --name myappconfig \
  --resource-group rg \
  --enable-public-network true

az appconfig network-rule add \
  --name myappconfig \
  --resource-group rg \
  --ip-address 203.0.113.0/24
```

---

## Best Practices

### 1. **Use Hierarchical Key Naming**

✅ **Good**:
```
MyApp:Database:ConnectionString
MyApp:Database:Timeout
MyApp:Cache:RedisEndpoint
MyApp:Logging:Level
```

❌ **Bad**:
```
db_connection_string
database-timeout
RedisEndpoint
log_level
```

### 2. **Leverage Labels for Environments**

```bash
# Same key, different values per environment
az appconfig kv set --key "MyApp:ApiUrl" --value "https://api-dev.contoso.com" --label "dev"
az appconfig kv set --key "MyApp:ApiUrl" --value "https://api-staging.contoso.com" --label "staging"
az appconfig kv set --key "MyApp:ApiUrl" --value "https://api.contoso.com" --label "prod"
```

### 3. **Implement Configuration Refresh**

```csharp
// Use sentinel key pattern for bulk refresh
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(connectionString)
           .Select(KeyFilter.Any, label)
           .ConfigureRefresh(refresh =>
           {
               // When Sentinel changes, refresh all config
               refresh.Register("MyApp:Sentinel", refreshAll: true)
                      .SetCacheExpiration(TimeSpan.FromMinutes(5));
           });
});
```

### 4. **Store Secrets in Key Vault**

```bash
# Create Key Vault reference
az appconfig kv set-keyvault \
  --name myappconfig \
  --key "MyApp:ConnectionString" \
  --secret-identifier "https://myvault.vault.azure.net/secrets/DbPassword"
```

### 5. **Use Managed Identities**

```csharp
// Production code - no connection strings
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(new Uri("https://myappconfig.azconfig.io"), 
                   new DefaultAzureCredential())
           .Select(KeyFilter.Any, label);
});
```

---

## Exam Tips

### Key Concepts for AZ-204

1. **Service Purpose**: App Configuration is for **application settings and feature flags**, not secrets (use Key Vault for secrets)

2. **Labels**: Enable multiple values for same key (environment variants: dev, staging, prod)

3. **Key Vault Integration**: App Configuration can store Key Vault references for secrets

4. **Authentication**: Managed identities are the recommended authentication method

5. **Dynamic Refresh**: Applications can refresh configuration without restarting using polling or push notifications

6. **Feature Flags**: Native support for feature management with percentage rollouts and targeting filters

7. **Pricing**: Free tier (1,000 requests/day) vs Standard tier (unlimited, SLA, advanced features)

8. **Size Limits**: 10 KB per key-value pair (combine multiple small values, not large datasets)

9. **Client Libraries**: Native providers for .NET, Java Spring, JavaScript, Python

10. **Point-in-Time**: Configuration snapshots support audit, rollback, and compliance

11. **Private Endpoints**: Isolate App Configuration from public internet

12. **No Replacement for Key Vault**: Store connection strings in App Configuration, passwords in Key Vault

### Common Exam Scenarios

**Scenario 1**: "Need to change logging level without redeploying"
→ **Answer**: Use App Configuration with dynamic refresh

**Scenario 2**: "Store database password securely"
→ **Answer**: Store in Key Vault, reference from App Configuration

**Scenario 3**: "Enable feature for 25% of users"
→ **Answer**: Use App Configuration feature flags with percentage filter

**Scenario 4**: "Different API endpoints for dev/staging/prod"
→ **Answer**: Use labels in App Configuration

**Scenario 5**: "Authenticate App Service to App Configuration"
→ **Answer**: Enable managed identity, grant "App Configuration Data Reader" role

---

## Quick Reference Commands

```bash
# Create App Configuration store
az appconfig create --name <name> --resource-group <rg> --location <location> --sku Standard

# Add key-value
az appconfig kv set --name <name> --key <key> --value <value>

# Add with label
az appconfig kv set --name <name> --key <key> --value <value> --label <label>

# List all key-values
az appconfig kv list --name <name>

# List with specific label
az appconfig kv list --name <name> --label <label>

# Delete key-value
az appconfig kv delete --name <name> --key <key>

# Get connection string
az appconfig credential list --name <name> --resource-group <rg>

# Add Key Vault reference
az appconfig kv set-keyvault --name <name> --key <key> --secret-identifier <vault-uri>

# Enable managed identity
az appconfig identity assign --name <name> --resource-group <rg>

# Grant access
az role assignment create \
  --assignee <principal-id> \
  --role "App Configuration Data Reader" \
  --scope <config-store-resource-id>
```

---

## Learn More

- [Azure App Configuration Documentation](https://docs.microsoft.com/azure/azure-app-configuration/)
- [Best Practices](https://docs.microsoft.com/azure/azure-app-configuration/howto-best-practices)
- [Feature Management](https://docs.microsoft.com/azure/azure-app-configuration/concept-feature-management)
- [.NET Quickstart](https://docs.microsoft.com/azure/azure-app-configuration/quickstart-dotnet-core-app)
