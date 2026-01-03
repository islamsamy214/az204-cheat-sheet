# Configure Application Settings

## Key Concepts
- **App Settings** = Environment variables for your app
- **Always encrypted at rest**
- **Override Web.config/appsettings.json** in ASP.NET apps
- **Auto-restart on changes** - Settings trigger app restart
- **Slot-specific settings** - Stick to deployment slots

## Application Settings

### Naming Rules
- ✅ Letters, numbers, periods (`.`), underscores (`_`)
- ⚠️ **Linux**: Replace `:` with `__` (double underscore) for nested JSON keys
- ⚠️ **Linux**: Periods `.` become `_` (single underscore)

### Examples

```bash
# Set app setting (CLI)
az webapp config appsettings set \
  --name <app-name> \
  --resource-group <rg-name> \
  --settings KEY1=value1 KEY2=value2

# Get app settings
az webapp config appsettings list \
  --name <app-name> \
  --resource-group <rg-name>

# Delete app setting
az webapp config appsettings delete \
  --name <app-name> \
  --resource-group <rg-name> \
  --setting-names KEY1 KEY2
```

### PowerShell

```powershell
# Set app settings
Set-AzWebApp -ResourceGroupName <rg-name> `
  -Name <app-name> `
  -AppSettings @{"DB_HOST"="server.database.azure.com"}
```

## Slot Settings

### What Are Slot Settings?
- Settings that **don't swap** with deployment slots
- **Stick to specific slot** (staging vs production)
- Prevents production config from moving to staging

### When to Use
- ✅ Connection strings (production DB)
- ✅ Feature flags (enable in staging only)
- ✅ Environment-specific config
- ✅ API keys (different per environment)

### Configuration

```json
[
  {
    "name": "ASPNETCORE_ENVIRONMENT",
    "value": "Production",
    "slotSetting": true  // Doesn't swap
  },
  {
    "name": "FEATURE_FLAG",
    "value": "enabled",
    "slotSetting": false  // Swaps with slot
  }
]
```

## Connection Strings

### Purpose
- Primarily for **ASP.NET and ASP.NET Core**
- **Backed up with app** (unlike app settings)
- Other languages: use app settings instead

### Connection String Types
| Type | Environment Prefix | Example Use |
|------|-------------------|-------------|
| **SQLServer** | `SQLCONNSTR_` | SQL Server databases |
| **MySQL** | `MYSQLCONNSTR_` | MySQL databases |
| **SQLAzure** | `SQLAZURECONNSTR_` | Azure SQL Database |
| **PostgreSQL** | `POSTGRESQLCONNSTR_` | PostgreSQL databases |
| **Custom** | `CUSTOMCONNSTR_` | Custom connections |
| **Redis Cache** | `REDISCACHECONNSTR_` | Redis connections |
| **Document DB** | `DOCDBCONNSTR_` | Cosmos DB |
| **Event Hub** | `EVENTHUBCONNSTR_` | Event Hubs |
| **Service Bus** | `SERVICEBUSCONNSTR_` | Service Bus |
| **Notification Hub** | `NOTIFICATIONHUBCONNSTR_` | Notification Hubs |

### Access in Code

```csharp
// C# - Access connection string
var connString = Environment.GetEnvironmentVariable("SQLCONNSTR_MyDatabase");
```

```javascript
// Node.js - Access connection string
const connString = process.env.SQLCONNSTR_MyDatabase;
```

### Configure Connection String

```bash
# Set connection string
az webapp config connection-string set \
  --name <app-name> \
  --resource-group <rg-name> \
  --connection-string-type SQLAzure \
  --settings MyDb="Server=tcp:server.database.azure.com..."
```

### JSON Format

```json
[
  {
    "name": "MyDatabase",
    "value": "Server=tcp:server.database.azure.com;Database=mydb",
    "type": "SQLAzure",
    "slotSetting": true
  },
  {
    "name": "RedisCache",
    "value": "mycache.redis.cache.windows.net:6380,password=...",
    "type": "Custom",
    "slotSetting": false
  }
]
```

## Custom Containers

### Set Environment Variables

```bash
# Bash
az webapp config appsettings set \
  --resource-group <rg-name> \
  --name <app-name> \
  --settings KEY1=value1 KEY2=value2

# PowerShell
Set-AzWebApp -ResourceGroupName <rg-name> `
  -Name <app-name> `
  -AppSettings @{"DB_HOST"="myserver.database.azure.com"}
```

### Verify Container Environment

```
URL: https://<app-name>.scm.azurewebsites.net/Env
```

### Injection Method
- Settings injected with `--env` flag (Linux containers)
- Available as environment variables at app startup

## Quick Reference

| Setting Type | Encrypted | Backed Up | Naming Restrictions |
|--------------|-----------|-----------|---------------------|
| **App Settings** | ✅ Yes | ❌ No | Letters, numbers, `.`, `_` |
| **Connection Strings** | ✅ Yes | ✅ Yes | Same as above |
| **Slot Settings** | ✅ Yes | ❌ No | Must be flagged explicitly |

## Bulk Edit JSON Format

```json
[
  {
    "name": "ASPNETCORE_ENVIRONMENT",
    "value": "Production",
    "slotSetting": true
  },
  {
    "name": "ApplicationInsights__InstrumentationKey",
    "value": "key-value-here",
    "slotSetting": false
  }
]
```

## Critical Notes
- 💡 **Use Key Vault** for secrets, not app settings
- 🔄 **Settings trigger restart** - plan accordingly
- 🎯 **Slot settings don't swap** - perfect for environment-specific config
- ⚠️ **Linux nested keys**: Use `__` instead of `:`
- 📝 **Connection strings** mainly for .NET apps
- 🔐 **Always encrypted at rest** - but not end-to-end
- ⚠️ **.NET PostgreSQL**: Set connection string type to "Custom" (workaround)

## Best Practices
1. ✅ Use **Key Vault references** for secrets
2. ✅ Mark environment-specific settings as **slot settings**
3. ✅ Use **connection strings for .NET** database connections
4. ✅ Use **app settings for non-.NET** apps
5. ⚠️ **Never commit secrets** to source control

## Exam Tips
- Know the difference between app settings and connection strings
- Understand slot settings and when they swap
- Remember connection string environment prefixes
- Know that settings trigger app restart
- Understand Linux naming rules (`:` → `__`, `.` → `_`)

[Learn More](https://learn.microsoft.com/en-us/training/modules/configure-web-app-settings/2-configure-application-settings)
