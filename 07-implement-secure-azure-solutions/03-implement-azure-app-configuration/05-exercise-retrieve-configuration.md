# Exercise: Retrieve Configuration Settings from Azure App Configuration

## Overview

In this hands-on exercise, you will:
1. Create an Azure App Configuration resource
2. Store configuration key-values using Azure CLI
3. Build a .NET console application
4. Retrieve configuration settings from App Configuration
5. Use hierarchical keys and configuration providers
6. Clean up resources

**Estimated time**: 15-20 minutes

---

## Prerequisites

### Required Tools

- **Azure Subscription**: [Sign up for free](https://azure.microsoft.com/free/)
- **.NET 8.0 SDK**: [Download](https://dotnet.microsoft.com/download)
- **Azure CLI**: [Install](https://docs.microsoft.com/cli/azure/install-azure-cli)
- **Code Editor**: Visual Studio Code or Visual Studio

### Verify Prerequisites

```bash
# Check Azure CLI
az --version

# Check .NET SDK
dotnet --version

# Login to Azure
az login

# Set default subscription (if you have multiple)
az account set --subscription "Your Subscription Name"
```

---

## Part 1: Create Azure App Configuration Store

### Step 1: Set Up Variables

```bash
# Variables
RESOURCE_GROUP="rg-appconfig-lab"
LOCATION="eastus"
APP_CONFIG_NAME="appconfig-lab-$RANDOM"

echo "App Configuration Name: $APP_CONFIG_NAME"
```

### Step 2: Create Resource Group

```bash
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

**Expected Output**:
```json
{
  "id": "/subscriptions/{sub-id}/resourceGroups/rg-appconfig-lab",
  "location": "eastus",
  "name": "rg-appconfig-lab",
  "properties": {
    "provisioningState": "Succeeded"
  }
}
```

### Step 3: Create App Configuration Store

```bash
# Create App Configuration (Free tier for lab)
az appconfig create \
  --name $APP_CONFIG_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Free
```

**Note**: Free tier limits: 1,000 requests/day, 10 MB storage

**Expected Output**:
```json
{
  "creationDate": "2024-01-15T10:30:00+00:00",
  "endpoint": "https://appconfig-lab-12345.azconfig.io",
  "id": "/subscriptions/{sub-id}/resourceGroups/rg-appconfig-lab/providers/Microsoft.AppConfiguration/configurationStores/appconfig-lab-12345",
  "location": "eastus",
  "name": "appconfig-lab-12345",
  "provisioningState": "Succeeded",
  "sku": {
    "name": "Free"
  }
}
```

### Step 4: Get Connection String

```bash
# Get connection string
CONNECTION_STRING=$(az appconfig credential list \
  --name $APP_CONFIG_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "[?name=='Primary'].connectionString" -o tsv)

echo "Connection String: $CONNECTION_STRING"

# Save for later use
echo "export APP_CONFIG_CONNECTION_STRING='$CONNECTION_STRING'" >> ~/.bashrc
export APP_CONFIG_CONNECTION_STRING="$CONNECTION_STRING"
```

---

## Part 2: Store Configuration Settings

### Step 1: Add Simple Key-Values

```bash
# Application name
az appconfig kv set \
  --name $APP_CONFIG_NAME \
  --key "TestApp:Settings:AppName" \
  --value "Configuration Lab App" \
  --yes

# Background color
az appconfig kv set \
  --name $APP_CONFIG_NAME \
  --key "TestApp:Settings:BackgroundColor" \
  --value "Blue" \
  --yes

# Font size
az appconfig kv set \
  --name $APP_CONFIG_NAME \
  --key "TestApp:Settings:FontSize" \
  --value "14" \
  --yes

# Font color
az appconfig kv set \
  --name $APP_CONFIG_NAME \
  --key "TestApp:Settings:FontColor" \
  --value "White" \
  --yes
```

**Note**: `--yes` flag bypasses confirmation prompt

### Step 2: Add Connection Strings

```bash
# Database connection string (non-sensitive for lab)
az appconfig kv set \
  --name $APP_CONFIG_NAME \
  --key "TestApp:ConnectionStrings:Database" \
  --value "Server=myserver.database.windows.net;Database=mydb;" \
  --yes

# Redis connection string
az appconfig kv set \
  --name $APP_CONFIG_NAME \
  --key "TestApp:ConnectionStrings:Redis" \
  --value "myredis.redis.cache.windows.net:6380,ssl=true" \
  --yes
```

### Step 3: Add JSON Value

```bash
# Add complex configuration as JSON
az appconfig kv set \
  --name $APP_CONFIG_NAME \
  --key "TestApp:Features" \
  --value '{"EnableNewUI":true,"EnableBeta":false}' \
  --content-type "application/json" \
  --yes
```

### Step 4: Verify Configuration

```bash
# List all key-values
az appconfig kv list \
  --name $APP_CONFIG_NAME \
  --all

# Count key-values
az appconfig kv list \
  --name $APP_CONFIG_NAME \
  --all \
  --query "length(@)"
```

**Expected Output**: 7 key-values

### Step 5: Query Specific Keys

```bash
# Get all TestApp settings
az appconfig kv list \
  --name $APP_CONFIG_NAME \
  --key "TestApp:*"

# Get only Settings
az appconfig kv list \
  --name $APP_CONFIG_NAME \
  --key "TestApp:Settings:*"

# Get specific key
az appconfig kv show \
  --name $APP_CONFIG_NAME \
  --key "TestApp:Settings:AppName"
```

---

## Part 3: Build .NET Console Application

### Step 1: Create Console Application

```bash
# Create directory
mkdir appconfig-console-lab
cd appconfig-console-lab

# Create .NET console app
dotnet new console
```

### Step 2: Add NuGet Packages

```bash
# Add App Configuration package
dotnet add package Microsoft.Extensions.Configuration.AzureAppConfiguration

# Add configuration extensions
dotnet add package Microsoft.Extensions.Configuration
```

**Packages installed**:
- `Microsoft.Extensions.Configuration.AzureAppConfiguration`: App Configuration provider
- `Microsoft.Extensions.Configuration`: Configuration abstractions

### Step 3: Write Application Code

Create or replace `Program.cs`:

```csharp
using Microsoft.Extensions.Configuration;

class Program
{
    static void Main(string[] args)
    {
        Console.WriteLine("=== Azure App Configuration Lab ===\n");

        // Get connection string from environment variable
        var connectionString = Environment.GetEnvironmentVariable("APP_CONFIG_CONNECTION_STRING");
        
        if (string.IsNullOrEmpty(connectionString))
        {
            Console.WriteLine("ERROR: APP_CONFIG_CONNECTION_STRING environment variable not set");
            Console.WriteLine("Set it using:");
            Console.WriteLine("export APP_CONFIG_CONNECTION_STRING='your-connection-string'");
            return;
        }

        // Build configuration
        var builder = new ConfigurationBuilder();
        builder.AddAzureAppConfiguration(connectionString);
        var config = builder.Build();

        Console.WriteLine("=== Application Settings ===");
        Console.WriteLine($"App Name: {config["TestApp:Settings:AppName"]}");
        Console.WriteLine($"Background Color: {config["TestApp:Settings:BackgroundColor"]}");
        Console.WriteLine($"Font Size: {config["TestApp:Settings:FontSize"]}");
        Console.WriteLine($"Font Color: {config["TestApp:Settings:FontColor"]}");

        Console.WriteLine("\n=== Connection Strings ===");
        Console.WriteLine($"Database: {config["TestApp:ConnectionStrings:Database"]}");
        Console.WriteLine($"Redis: {config["TestApp:ConnectionStrings:Redis"]}");

        Console.WriteLine("\n=== JSON Configuration ===");
        Console.WriteLine($"Features JSON: {config["TestApp:Features"]}");

        // Access all keys with prefix
        Console.WriteLine("\n=== All TestApp Settings ===");
        var settings = config.GetSection("TestApp:Settings");
        foreach (var setting in settings.GetChildren())
        {
            Console.WriteLine($"{setting.Key}: {setting.Value}");
        }

        Console.WriteLine("\n=== Success! Configuration retrieved from Azure App Configuration ===");
    }
}
```

### Step 4: Build Application

```bash
dotnet build
```

**Expected Output**:
```
Build succeeded.
    0 Warning(s)
    0 Error(s)
```

### Step 5: Run Application

```bash
dotnet run
```

**Expected Output**:
```
=== Azure App Configuration Lab ===

=== Application Settings ===
App Name: Configuration Lab App
Background Color: Blue
Font Size: 14
Font Color: White

=== Connection Strings ===
Database: Server=myserver.database.windows.net;Database=mydb;
Redis: myredis.redis.cache.windows.net:6380,ssl=true

=== JSON Configuration ===
Features JSON: {"EnableNewUI":true,"EnableBeta":false}

=== All TestApp Settings ===
AppName: Configuration Lab App
BackgroundColor: Blue
FontSize: 14
FontColor: White

=== Success! Configuration retrieved from Azure App Configuration ===
```

---

## Part 4: Advanced - Dynamic Configuration Refresh

### Step 1: Add Additional Packages

```bash
dotnet add package Microsoft.Extensions.Configuration.AzureAppConfiguration
```

### Step 2: Enhanced Program with Refresh

Create `ProgramRefresh.cs`:

```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Configuration.AzureAppConfiguration;

class ProgramRefresh
{
    private static IConfiguration _configuration;
    private static IConfigurationRefresher _refresher;

    static async Task Main(string[] args)
    {
        Console.WriteLine("=== Azure App Configuration with Refresh ===\n");

        var connectionString = Environment.GetEnvironmentVariable("APP_CONFIG_CONNECTION_STRING");
        
        if (string.IsNullOrEmpty(connectionString))
        {
            Console.WriteLine("ERROR: Connection string not set");
            return;
        }

        // Build configuration with refresh
        var builder = new ConfigurationBuilder();
        builder.AddAzureAppConfiguration(options =>
        {
            options.Connect(connectionString)
                   .ConfigureRefresh(refresh =>
                   {
                       // Register key to watch for changes
                       refresh.Register("TestApp:Settings:BackgroundColor", refreshAll: false)
                              .SetCacheExpiration(TimeSpan.FromSeconds(10));
                   });

            _refresher = options.GetRefresher();
        });

        _configuration = builder.Build();

        // Monitor configuration for 60 seconds
        Console.WriteLine("Monitoring configuration for 60 seconds...");
        Console.WriteLine("Change 'TestApp:Settings:BackgroundColor' in Azure Portal to see updates\n");

        for (int i = 0; i < 12; i++)
        {
            // Trigger refresh (checks if cache expired)
            await _refresher.TryRefreshAsync();

            // Read current value
            var color = _configuration["TestApp:Settings:BackgroundColor"];
            Console.WriteLine($"[{DateTime.Now:HH:mm:ss}] Background Color: {color}");

            await Task.Delay(5000); // Wait 5 seconds
        }

        Console.WriteLine("\nRefresh monitoring complete.");
    }
}
```

### Step 3: Test Dynamic Refresh

**Terminal 1** - Run the application:
```bash
dotnet run
```

**Terminal 2** - Update configuration:
```bash
# Change background color
az appconfig kv set \
  --name $APP_CONFIG_NAME \
  --key "TestApp:Settings:BackgroundColor" \
  --value "Red" \
  --yes

# Wait 10 seconds (cache expiration)

# Change again
az appconfig kv set \
  --name $APP_CONFIG_NAME \
  --key "TestApp:Settings:BackgroundColor" \
  --value "Green" \
  --yes
```

**Expected Terminal 1 Output**:
```
[10:00:00] Background Color: Blue
[10:00:05] Background Color: Blue
[10:00:10] Background Color: Blue
[10:00:15] Background Color: Red   ← Changed!
[10:00:20] Background Color: Red
[10:00:25] Background Color: Red
[10:00:30] Background Color: Green ← Changed again!
```

---

## Part 5: Strongly-Typed Configuration

### Step 1: Create Settings Classes

Create `AppSettings.cs`:

```csharp
public class AppSettings
{
    public Settings Settings { get; set; }
    public ConnectionStrings ConnectionStrings { get; set; }
}

public class Settings
{
    public string AppName { get; set; }
    public string BackgroundColor { get; set; }
    public int FontSize { get; set; }
    public string FontColor { get; set; }
}

public class ConnectionStrings
{
    public string Database { get; set; }
    public string Redis { get; set; }
}
```

### Step 2: Use Strongly-Typed Configuration

Create `ProgramTyped.cs`:

```csharp
using Microsoft.Extensions.Configuration;

class ProgramTyped
{
    static void Main(string[] args)
    {
        Console.WriteLine("=== Strongly-Typed Configuration ===\n");

        var connectionString = Environment.GetEnvironmentVariable("APP_CONFIG_CONNECTION_STRING");
        
        var builder = new ConfigurationBuilder();
        builder.AddAzureAppConfiguration(connectionString);
        var config = builder.Build();

        // Bind to strongly-typed class
        var appSettings = new AppSettings();
        config.GetSection("TestApp").Bind(appSettings);

        // Access with IntelliSense
        Console.WriteLine($"App Name: {appSettings.Settings.AppName}");
        Console.WriteLine($"Background Color: {appSettings.Settings.BackgroundColor}");
        Console.WriteLine($"Font Size: {appSettings.Settings.FontSize}");
        Console.WriteLine($"Font Color: {appSettings.Settings.FontColor}");
        Console.WriteLine($"\nDatabase: {appSettings.ConnectionStrings.Database}");
        Console.WriteLine($"Redis: {appSettings.ConnectionStrings.Redis}");
    }
}
```

### Step 3: Run Strongly-Typed Example

```bash
dotnet run
```

---

## Part 6: Use Labels for Environments

### Step 1: Add Environment-Specific Values

```bash
# Development environment
az appconfig kv set \
  --name $APP_CONFIG_NAME \
  --key "TestApp:Settings:BackgroundColor" \
  --value "Yellow" \
  --label "Development" \
  --yes

# Production environment
az appconfig kv set \
  --name $APP_CONFIG_NAME \
  --key "TestApp:Settings:BackgroundColor" \
  --value "DarkBlue" \
  --label "Production" \
  --yes

# Staging environment
az appconfig kv set \
  --name $APP_CONFIG_NAME \
  --key "TestApp:Settings:BackgroundColor" \
  --value "LightGreen" \
  --label "Staging" \
  --yes
```

### Step 2: Query by Label

```bash
# List all values for BackgroundColor
az appconfig kv list \
  --name $APP_CONFIG_NAME \
  --key "TestApp:Settings:BackgroundColor" \
  --all

# Get Development value
az appconfig kv show \
  --name $APP_CONFIG_NAME \
  --key "TestApp:Settings:BackgroundColor" \
  --label "Development"

# Get Production value
az appconfig kv show \
  --name $APP_CONFIG_NAME \
  --key "TestApp:Settings:BackgroundColor" \
  --label "Production"
```

### Step 3: Use Labels in Application

Create `ProgramLabels.cs`:

```csharp
using Microsoft.Extensions.Configuration;

class ProgramLabels
{
    static void Main(string[] args)
    {
        Console.WriteLine("=== Configuration with Labels ===\n");

        var connectionString = Environment.GetEnvironmentVariable("APP_CONFIG_CONNECTION_STRING");
        string environment = args.Length > 0 ? args[0] : "Development";

        Console.WriteLine($"Loading configuration for: {environment}\n");

        var builder = new ConfigurationBuilder();
        builder.AddAzureAppConfiguration(options =>
        {
            options.Connect(connectionString)
                   .Select("TestApp:*", environment); // Filter by label
        });

        var config = builder.Build();

        Console.WriteLine($"Background Color: {config["TestApp:Settings:BackgroundColor"]}");
    }
}
```

### Step 4: Test with Different Environments

```bash
# Development
dotnet run Development
# Output: Background Color: Yellow

# Staging
dotnet run Staging
# Output: Background Color: LightGreen

# Production
dotnet run Production
# Output: Background Color: DarkBlue
```

---

## Part 7: Verify in Azure Portal

### Step 1: Open Azure Portal

1. Navigate to [Azure Portal](https://portal.azure.com)
2. Search for "App Configuration"
3. Select your App Configuration store (`appconfig-lab-XXXXX`)

### Step 2: View Configuration Explorer

1. In left menu, select **Configuration explorer**
2. View all key-values
3. Filter by:
   - Key prefix: `TestApp:Settings:*`
   - Label: Select environment

### Step 3: Edit Configuration

1. Click on `TestApp:Settings:BackgroundColor`
2. Change value to `Purple`
3. Click **Apply**
4. Run your application to see the change

### Step 4: View Operations

1. In left menu, select **Activity log**
2. See all create/update operations
3. Filter by:
   - Operation: `Set Key Value`
   - Time range: Last hour

---

## Part 8: Clean Up Resources

### Option 1: Delete Resource Group (Recommended)

```bash
# Delete entire resource group (removes all resources)
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait
```

**Deletes**:
- App Configuration store
- All key-values
- Resource group

### Option 2: Delete Only App Configuration

```bash
# Delete just the App Configuration store
az appconfig delete \
  --name $APP_CONFIG_NAME \
  --resource-group $RESOURCE_GROUP \
  --yes
```

### Verify Deletion

```bash
# Check if resource group still exists
az group exists --name $RESOURCE_GROUP
# Output: false (after deletion completes)

# List App Configuration stores
az appconfig list --query "[?name=='$APP_CONFIG_NAME']"
# Output: [] (empty)
```

---

## Troubleshooting

### Issue 1: Connection String Not Found

**Error**:
```
ERROR: APP_CONFIG_CONNECTION_STRING environment variable not set
```

**Solution**:
```bash
# Retrieve connection string again
CONNECTION_STRING=$(az appconfig credential list \
  --name $APP_CONFIG_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "[?name=='Primary'].connectionString" -o tsv)

# Export it
export APP_CONFIG_CONNECTION_STRING="$CONNECTION_STRING"

# Verify
echo $APP_CONFIG_CONNECTION_STRING
```

### Issue 2: App Configuration Name Already Exists

**Error**:
```
(ConflictError) The configuration store name is already taken.
```

**Solution**:
```bash
# Generate unique name with random number
APP_CONFIG_NAME="appconfig-lab-$RANDOM"
echo "New name: $APP_CONFIG_NAME"

# Recreate
az appconfig create --name $APP_CONFIG_NAME --resource-group $RESOURCE_GROUP --location $LOCATION --sku Free
```

### Issue 3: Package Not Found

**Error**:
```
error NU1102: Unable to find package 'Microsoft.Extensions.Configuration.AzureAppConfiguration'
```

**Solution**:
```bash
# Clear NuGet cache
dotnet nuget locals all --clear

# Restore packages
dotnet restore

# Rebuild
dotnet build
```

### Issue 4: Free Tier Request Limit

**Error**:
```
(TooManyRequests) Request rate has been exceeded for subscription
```

**Solution**:
- Free tier allows 1,000 requests/day
- Wait for quota reset (next day)
- Or upgrade to Standard tier:
```bash
az appconfig update --name $APP_CONFIG_NAME --sku Standard
```

---

## Key Takeaways

### What You Learned

1. ✅ **Created** Azure App Configuration store
2. ✅ **Stored** hierarchical key-values
3. ✅ **Built** .NET console application
4. ✅ **Retrieved** configuration programmatically
5. ✅ **Implemented** dynamic refresh
6. ✅ **Used** labels for environment variants
7. ✅ **Applied** strongly-typed configuration
8. ✅ **Verified** configuration in Azure Portal

### Configuration Patterns Explored

| Pattern | Description | Use Case |
|---------|-------------|----------|
| **Hierarchical Keys** | `App:Component:Setting` | Organize related settings |
| **Labels** | Environment variants | Dev/Staging/Prod separation |
| **Dynamic Refresh** | Auto-update without restart | Real-time configuration changes |
| **Strongly-Typed** | Bind to C# classes | Type safety and IntelliSense |
| **JSON Values** | Complex objects | Structured configuration |

### Code Patterns Learned

**Basic retrieval**:
```csharp
var config = builder.Build();
var value = config["TestApp:Settings:AppName"];
```

**Section binding**:
```csharp
var settings = config.GetSection("TestApp:Settings");
foreach (var setting in settings.GetChildren())
{
    Console.WriteLine($"{setting.Key}: {setting.Value}");
}
```

**Dynamic refresh**:
```csharp
.ConfigureRefresh(refresh =>
{
    refresh.Register("TestApp:Settings:BackgroundColor")
           .SetCacheExpiration(TimeSpan.FromSeconds(10));
});
```

**Environment selection**:
```csharp
.Select("TestApp:*", environment)
```

---

## Next Steps

### Explore Further

1. **Managed Identity Authentication**:
   - Deploy application to App Service
   - Enable managed identity
   - Remove connection string

2. **Feature Flags**:
   - Add feature flag to App Configuration
   - Use `Microsoft.FeatureManagement` package
   - Implement percentage rollout

3. **Key Vault Integration**:
   - Store secrets in Key Vault
   - Reference from App Configuration
   - Use Key Vault references

4. **ASP.NET Core Integration**:
   - Create web application
   - Use configuration provider
   - Implement automatic refresh middleware

### Practice Exercises

1. Create separate App Configuration stores for dev/staging/prod
2. Implement feature flag for experimental feature
3. Add Key Vault reference for database password
4. Deploy to Azure App Service with managed identity
5. Set up continuous deployment with configuration updates

---

## Exam Tips

### Key Concepts Demonstrated

1. **Connection String**: Quick setup for development, contains secrets
2. **Hierarchical Keys**: `App:Component:Setting` pattern
3. **Labels**: Enable environment-specific values
4. **ConfigurationBuilder**: Standard .NET configuration pattern
5. **Dynamic Refresh**: Cache expiration and refresh triggers
6. **Strongly-Typed**: Bind configuration to C# classes
7. **Azure CLI**: Create, update, query, and delete operations

### Common Exam Scenarios

**Scenario**: "Retrieve configuration settings in .NET application"
→ **Answer**: Use `ConfigurationBuilder.AddAzureAppConfiguration(connectionString)`

**Scenario**: "Different database connection strings for dev/prod"
→ **Answer**: Use same key with different labels (Development, Production)

**Scenario**: "Update configuration without restarting application"
→ **Answer**: Configure refresh with `ConfigureRefresh()` and `SetCacheExpiration()`

**Scenario**: "Organize related settings"
→ **Answer**: Use hierarchical key naming (e.g., `MyApp:Database:*`)

---

## Additional Resources

- [Azure App Configuration Documentation](https://docs.microsoft.com/azure/azure-app-configuration/)
- [.NET Quickstart](https://docs.microsoft.com/azure/azure-app-configuration/quickstart-dotnet-core-app)
- [ASP.NET Core Tutorial](https://docs.microsoft.com/azure/azure-app-configuration/quickstart-aspnet-core-app)
- [Feature Management](https://docs.microsoft.com/azure/azure-app-configuration/concept-feature-management)

---

**Congratulations!** You've completed the Azure App Configuration exercise. You now know how to centralize application configuration and retrieve it programmatically.
