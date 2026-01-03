# Azure Functions Development

## Key Concepts
- **Function app** - Container for multiple functions
- **host.json** - Configuration for entire function app
- **local.settings.json** - Local development settings
- **function.json** - Function-level configuration (non-C#)
- **Local development** - Test locally, deploy to Azure

## Function App Structure

### What Is a Function App?
**Container** that manages, deploys, and scales functions together:

- **Execution context** - All functions share same runtime environment
- **Deployment unit** - Deploy entire function app at once
- **Pricing plan** - All functions share same billing/plan
- **Runtime version** - All functions use same runtime (e.g., 4.x)
- **Language** - Functions 2.x+: All functions must use same language

### Organizational Benefits
```
Function App: OrderProcessing
├── ValidateOrder (HTTP trigger)
├── ProcessPayment (Queue trigger)
├── SendConfirmation (Queue trigger)
└── UpdateInventory (Queue trigger)

Benefits:
- Logical grouping of related functions
- Shared configuration (host.json)
- Single deployment
- Unified monitoring
```

## Project Structure

### Essential Files

#### host.json
**Function app-wide configuration**:

```json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "maxTelemetryItemsPerSecond": 20
      }
    }
  },
  "functionTimeout": "00:05:00",
  "extensions": {
    "http": {
      "routePrefix": "api",
      "maxConcurrentRequests": 100
    },
    "queues": {
      "batchSize": 16,
      "maxDequeueCount": 5
    }
  }
}
```

**Key settings**:
- `version` - Schema version (always "2.0")
- `functionTimeout` - Max execution time
- `extensions` - Extension-specific config
- `logging` - Application Insights settings

#### local.settings.json
**Local development configuration** (NOT deployed):

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "node",
    "MyDatabaseConnection": "Server=localhost;...",
    "ApiKey": "local-dev-key-123"
  },
  "Host": {
    "LocalHttpPort": 7071,
    "CORS": "*"
  },
  "ConnectionStrings": {
    "SQLConnectionString": "..."
  }
}
```

**Structure**:
- `Values` - App settings (connection strings, API keys)
- `Host` - Local Functions host settings
- `ConnectionStrings` - Database connections
- `IsEncrypted` - Whether values are encrypted

⚠️ **Important**: Never commit to source control (contains secrets)

#### function.json (JavaScript/Python/PowerShell)
**Function-level configuration**:

```json
{
  "disabled": false,
  "bindings": [
    {
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": ["get", "post"],
      "authLevel": "function"
    },
    {
      "type": "http",
      "direction": "out",
      "name": "res"
    }
  ]
}
```

**C# equivalent** (attributes instead of function.json):
```csharp
[FunctionName("HttpExample")]
public static IActionResult Run(
    [HttpTrigger(AuthorizationLevel.Function, "get", "post")] HttpRequest req,
    ILogger log)
{
    // Function code
}
```

### Directory Structure

#### Node.js / Python / PowerShell
```
MyFunctionApp/
├── host.json
├── local.settings.json
├── package.json (Node.js)
├── requirements.txt (Python)
├── HttpTriggerFunction/
│   ├── function.json
│   └── index.js
├── QueueTriggerFunction/
│   ├── function.json
│   └── index.js
└── TimerTriggerFunction/
    ├── function.json
    └── index.js
```

#### C# (.NET)
```
MyFunctionApp/
├── MyFunctionApp.csproj
├── host.json
├── local.settings.json
├── HttpTriggerFunction.cs
├── QueueTriggerFunction.cs
└── TimerTriggerFunction.cs
```

## Local Development

### Benefits
✅ **Full runtime** - Complete Functions runtime locally
✅ **Live connections** - Connect to Azure services
✅ **Debugging** - Full debugging support
✅ **Fast iteration** - Test without deployment

### Prerequisites
| Tool | Purpose |
|------|---------|
| **Azure Functions Core Tools** | Local runtime and CLI |
| **Visual Studio Code** | Code editor (recommended) |
| **Azure Functions extension** | VS Code integration |
| **Language runtime** | Node.js, Python, .NET, etc. |

### Installation

#### Azure Functions Core Tools
```bash
# macOS (Homebrew)
brew tap azure/functions
brew install azure-functions-core-tools@4

# Windows (Chocolatey)
choco install azure-functions-core-tools

# Ubuntu/Debian
curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg
sudo mv microsoft.gpg /etc/apt/trusted.gpg.d/microsoft.gpg
sudo sh -c 'echo "deb [arch=amd64] https://packages.microsoft.com/repos/microsoft-ubuntu-$(lsb_release -cs)-prod $(lsb_release -cs) main" > /etc/apt/sources.list.d/dotnetdev.list'
sudo apt-get update
sudo apt-get install azure-functions-core-tools-4
```

#### VS Code Extension
```
1. Open VS Code
2. Extensions (Ctrl+Shift+X)
3. Search "Azure Functions"
4. Install Microsoft's Azure Functions extension
```

### Create New Project

#### Using Core Tools
```bash
# Initialize new function app
func init MyFunctionApp --worker-runtime node

# Navigate to project
cd MyFunctionApp

# Create function
func new --name HttpExample --template "HTTP trigger"

# Run locally
func start
```

#### Using VS Code
```
1. Command Palette (Ctrl+Shift+P)
2. "Azure Functions: Create New Project"
3. Select folder
4. Choose language
5. Select runtime version
6. Choose template
7. Provide function name
```

### Local Testing

#### Start Local Runtime
```bash
# Start Functions host
func start

# Output:
# Azure Functions Core Tools
# Core Tools Version:       4.0.5455
# Function Runtime Version: 4.27.5.21554
#
# Functions:
#   HttpExample: [GET,POST] http://localhost:7071/api/HttpExample
#
# For detailed output, run func with --verbose flag.
```

#### Test HTTP Function
```bash
# Test with curl
curl http://localhost:7071/api/HttpExample?name=Azure

# Test with body
curl -X POST http://localhost:7071/api/HttpExample \
  -H "Content-Type: application/json" \
  -d '{"name":"Azure"}'
```

#### Debug in VS Code
```
1. Set breakpoints in code
2. Press F5 (Start Debugging)
3. Functions host starts with debugger attached
4. Trigger function (HTTP request, queue message, etc.)
5. Breakpoint hits, inspect variables
```

### Testing Triggers Locally

#### HTTP Trigger
```bash
# Direct call to localhost endpoint
curl http://localhost:7071/api/HttpExample
```

#### Storage Queue Trigger
**Option 1**: Use Azurite emulator
```bash
# Install Azurite
npm install -g azurite

# Start Azurite
azurite --silent --location c:\azurite --debug c:\azurite\debug.log

# In local.settings.json:
"AzureWebJobsStorage": "UseDevelopmentStorage=true"
```

**Option 2**: Use live Azure Storage
```json
// local.settings.json
{
  "Values": {
    "AzureWebJobsStorage": "DefaultEndpointsProtocol=https;AccountName=..."
  }
}
```

#### Timer Trigger
```bash
# Runs automatically on schedule
# Use short interval for testing: "*/10 * * * * *" (every 10 seconds)
```

#### Manual Trigger
```bash
# Admin endpoint for non-HTTP triggers
curl -X POST http://localhost:7071/admin/functions/QueueTriggerFunction \
  -H "Content-Type: application/json" \
  -d '{"input":"test data"}'
```

## Settings Synchronization

### Deploy Settings to Azure
```bash
# Upload local.settings.json to Azure
func azure functionapp publish <function-app-name> --publish-settings-only

# Or manually add each setting
az functionapp config appsettings set \
  --name <function-app-name> \
  --resource-group <rg-name> \
  --settings \
    "DatabaseConnection=Server=..." \
    "ApiKey=prod-key-456"
```

### Download Settings from Azure
```bash
# Download app settings to local.settings.json
func azure functionapp fetch-app-settings <function-app-name>

# Decrypt settings (if encrypted)
func settings decrypt
```

### Best Practices
✅ **Separate environments** - Different settings for dev/test/prod
✅ **Use Key Vault** - Store secrets in Azure Key Vault
✅ **Never commit secrets** - Add local.settings.json to .gitignore
✅ **Connection naming** - Use descriptive names for connection strings

## Portal Development Limitations

### Limited Portal Editing
❌ **C# compiled** - Cannot edit in portal
❌ **Python** - Cannot edit in portal
❌ **Java** - Cannot edit in portal
❌ **TypeScript** - Cannot edit in portal

✅ **C# Script (.csx)** - Can edit in portal
✅ **JavaScript** - Can edit in portal
✅ **PowerShell** - Can edit in portal

### Recommendation
💡 **Develop locally** for all production scenarios:
- Better IDE support
- Full debugging
- Source control integration
- CI/CD pipelines
- Unit testing

## Configuration Examples

### host.json - Production Settings
```json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "maxTelemetryItemsPerSecond": 20
      }
    },
    "logLevel": {
      "default": "Information",
      "Host.Results": "Error",
      "Function": "Error",
      "Host.Aggregator": "Trace"
    }
  },
  "functionTimeout": "00:05:00",
  "extensions": {
    "http": {
      "routePrefix": "api",
      "maxConcurrentRequests": 100,
      "maxOutstandingRequests": 200,
      "dynamicThrottlesEnabled": true
    },
    "queues": {
      "batchSize": 16,
      "maxDequeueCount": 5,
      "newBatchThreshold": 8,
      "visibilityTimeout": "00:00:30"
    },
    "serviceBus": {
      "prefetchCount": 100,
      "messageHandlerOptions": {
        "maxConcurrentCalls": 32,
        "autoComplete": true
      }
    }
  }
}
```

### local.settings.json - Development
```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "node",
    "APPINSIGHTS_INSTRUMENTATIONKEY": "",
    "DatabaseConnection": "Server=localhost;Database=TestDB;",
    "ExternalApiUrl": "https://dev-api.example.com",
    "ApiKey": "dev-key-123"
  },
  "Host": {
    "LocalHttpPort": 7071,
    "CORS": "*",
    "CORSCredentials": false
  }
}
```

## Critical Notes
- 💡 **Function app** - Deployment unit containing multiple functions
- ⚠️ **Same language** - Functions 2.x+ requires same language per app
- 🎯 **host.json** - App-wide configuration
- 📊 **local.settings.json** - Never commit to source control
- ✅ **Local development** - Recommended for all scenarios
- 🔄 **Settings sync** - Use func CLI to upload/download settings
- ⏱️ **Azurite** - Local storage emulator for testing
- 🔒 **Portal limitations** - C#, Python, Java not editable in portal

## Exam Tips
- Function app = container for multiple related functions
- All functions in app share: runtime, pricing plan, deployment
- Functions 2.x+: All functions must use same language
- host.json: App-wide configuration (timeout, extensions, logging)
- local.settings.json: Local dev settings, not deployed
- Never commit local.settings.json (contains secrets)
- C# uses attributes, other languages use function.json
- Portal editing limited (C# Script, JavaScript, PowerShell only)
- Local development recommended (better tooling, debugging)
- Azure Functions Core Tools: CLI for local development
- Azurite: Local storage emulator for testing
- Settings sync: `func azure functionapp publish --publish-settings-only`
- Manual trigger admin endpoint: http://localhost:7071/admin/functions/{name}

[Learn More](https://learn.microsoft.com/en-us/training/modules/develop-azure-functions/2-azure-function-development-overview)
