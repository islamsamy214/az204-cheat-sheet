# Azure Functions Overview

## Key Concepts
- **Serverless compute** - No infrastructure management required
- **Event-driven** - Code runs in response to triggers
- **Triggers** - Events that start function execution
- **Bindings** - Simplified input/output data connections
- **Pay-per-execution** - Only charged when code runs

## What Is Azure Functions?

### Definition
**Azure Functions** is a serverless compute service that lets you run event-driven code without managing infrastructure:

- **Write less code** - Focus on business logic, not boilerplate
- **Maintain less infrastructure** - Cloud handles servers
- **Save on costs** - Pay only for execution time
- **Auto-scale** - Automatically scales based on demand

### Use Cases
- **Web APIs** - Build HTTP-triggered RESTful APIs
- **Data processing** - Process files, images, streams
- **System integration** - Connect disparate systems
- **IoT data streams** - Process device telemetry
- **Message queues** - Handle queue-based workflows
- **Scheduled tasks** - Run code on timer triggers
- **Microservices** - Build small, focused services

## Core Concepts

### Triggers
**Start execution** of function code:

| Trigger Type | Description | Example |
|--------------|-------------|---------|
| **HTTP** | Web request | REST API endpoint |
| **Timer** | Schedule (cron) | Nightly cleanup job |
| **Queue** | Message in queue | Process order |
| **Blob** | File uploaded | Image resize |
| **Event Hub** | Stream events | IoT telemetry |
| **Event Grid** | Event notification | Resource change |
| **Service Bus** | Enterprise messaging | Order processing |
| **Cosmos DB** | Database change | Data sync |

**Rule**: Each function has **exactly one trigger**

### Bindings
**Simplify input/output** connections:

- **Input binding** - Read data from external source
- **Output binding** - Write data to external destination
- **Declarative** - Configured in function.json or attributes

**Rule**: Function can have **multiple bindings**

### Example: Bindings in Action
```csharp
[FunctionName("ProcessOrder")]
public static void Run(
    [QueueTrigger("orders")] string order,          // Trigger
    [Blob("receipts/{id}")] out string receipt,      // Output binding
    [CosmosDB("orders", "history")] out Order record // Output binding
)
{
    // Process order
    receipt = GenerateReceipt(order);
    record = SaveToHistory(order);
}
```

**Result**: Function triggered by queue, outputs to Blob Storage and Cosmos DB automatically

## Azure Functions vs Azure Logic Apps

### Comparison Table

| Feature | Azure Functions | Azure Logic Apps |
|---------|-----------------|------------------|
| **Development** | Code-first (imperative) | Designer-first (declarative) |
| **Skill set** | Developers (C#, JS, Python) | Business analysts, developers |
| **Orchestration** | Durable Functions extension | Visual workflow designer |
| **Connectivity** | ~12 built-in bindings, custom code | 400+ connectors, B2B pack |
| **Actions** | Write code for each action | Large collection ready-made |
| **Monitoring** | Application Insights | Azure portal, Monitor logs |
| **Management** | REST API, Visual Studio, CLI | Portal, REST API, PowerShell |
| **Execution** | Azure, locally | Azure, locally, on-premises |
| **Pricing** | Pay per execution | Pay per action/connector usage |
| **Source control** | Git integration | JSON definition files |

### When to Choose Azure Functions
✅ Need to write custom code logic
✅ Have developer skill set
✅ Need fine-grained control over execution
✅ Processing high volumes at low cost
✅ Integration with Application Insights

### When to Choose Logic Apps
✅ Business process workflow (approval, routing)
✅ Visual designer preferred
✅ Enterprise integration (SAP, Oracle, etc.)
✅ B2B scenarios (EDI, AS2, X12)
✅ Low-code/no-code requirements

## Azure Functions vs WebJobs

### Comparison Table

| Feature | Azure Functions | WebJobs + WebJobs SDK |
|---------|-----------------|----------------------|
| **Serverless model** | ✅ Yes | ❌ No |
| **Automatic scaling** | ✅ Yes | ❌ No (manual scale App Service) |
| **Browser development** | ✅ Portal editor | ❌ Requires IDE |
| **Pay-per-use pricing** | ✅ Consumption plan | ❌ Pay for App Service Plan |
| **Logic Apps integration** | ✅ Yes | ❌ No |
| **Trigger events** | HTTP, Timer, Storage, Service Bus, Cosmos DB, Event Hubs, Event Grid, Webhooks | Timer, Storage, Service Bus, Cosmos DB, Event Hubs, File system (no HTTP, Event Grid) |
| **Host** | Managed Functions host | App Service Plan |
| **Deployment** | Independent function apps | Tied to web app |

### When to Choose Azure Functions
✅ Need serverless architecture
✅ Want automatic scaling
✅ Pay-per-execution cost model
✅ HTTP-triggered APIs
✅ Event Grid integration
✅ Quick development/testing in portal

### When to Choose WebJobs
✅ Already have App Service Plan
✅ Need full control over host
✅ Background processing for existing web app
✅ File system trigger needed
✅ Long-running continuous jobs

## Architecture Components

### Function App
- **Container** for one or more functions
- **Share** same hosting plan, deployment, runtime
- **Organize** related functions together
- **Unit of deployment** and scale

### Functions Runtime
- **Executes** function code
- **Manages** triggers and bindings
- **Handles** scaling and lifecycle
- **Versions**: 4.x (current), 3.x, 2.x, 1.x

### Host Configuration
Configured in `host.json`:
```json
{
  "version": "2.0",
  "functionTimeout": "00:05:00",
  "extensions": {
    "http": {
      "routePrefix": "api",
      "maxConcurrentRequests": 100
    }
  }
}
```

## Supported Languages

### In-Portal Development
| Language | Runtime | Portal Support |
|----------|---------|----------------|
| **C# Script** | .NET 6, 7, 8 | ✅ Yes |
| **JavaScript** | Node.js 18, 20 | ✅ Yes |
| **PowerShell** | PowerShell 7.2, 7.4 | ✅ Yes |

### Local Development Only
| Language | Runtime | Portal Support |
|----------|---------|----------------|
| **C# (compiled)** | .NET 6, 7, 8, Isolated | ❌ No |
| **Python** | 3.8, 3.9, 3.10, 3.11 | ❌ No |
| **Java** | Java 8, 11, 17 | ❌ No |
| **TypeScript** | Node.js 18, 20 | ❌ No |

⚠️ **Note**: Develop C#, Python, Java, TypeScript locally, deploy to Azure

## Development Workflow

### 1. Create Function App
```bash
# CLI
az functionapp create \
  --name <function-app-name> \
  --resource-group <rg-name> \
  --consumption-plan-location <region> \
  --runtime node \
  --runtime-version 18 \
  --storage-account <storage-name>
```

### 2. Create Function
```bash
# Using Azure Functions Core Tools
func init MyFunctionApp --worker-runtime node
cd MyFunctionApp
func new --name HttpExample --template "HTTP trigger"
```

### 3. Test Locally
```bash
func start

# Output:
# HttpExample: [GET,POST] http://localhost:7071/api/HttpExample
```

### 4. Deploy
```bash
func azure functionapp publish <function-app-name>
```

## Function Execution Flow

```
1. Event occurs (HTTP request, timer, message, etc.)
   ↓
2. Trigger fires
   ↓
3. Function runtime invokes function
   ↓
4. Input bindings retrieve data (if configured)
   ↓
5. Function code executes
   ↓
6. Output bindings write data (if configured)
   ↓
7. Function completes
   ↓
8. Response returned (for synchronous triggers)
```

## Integration Services Comparison

| Service | Purpose | Best For | Development |
|---------|---------|----------|-------------|
| **Azure Functions** | Serverless compute | Custom code, data processing | Code-first |
| **Logic Apps** | Workflow orchestration | Business processes, integration | Designer-first |
| **Event Grid** | Event distribution | Reactive programming, pub/sub | Configuration |
| **Service Bus** | Enterprise messaging | Reliable message delivery | Messaging |
| **Power Automate** | Business automation | Office 365, citizen developers | Low-code |

## Quick Command Reference

```bash
# Create function app
az functionapp create \
  --name <app-name> \
  --resource-group <rg> \
  --consumption-plan-location <region> \
  --runtime <node|python|dotnet|java> \
  --storage-account <storage>

# List function apps
az functionapp list \
  --resource-group <rg> \
  --output table

# Get function app URL
az functionapp show \
  --name <app-name> \
  --resource-group <rg> \
  --query "defaultHostName" -o tsv

# View logs
az functionapp log tail \
  --name <app-name> \
  --resource-group <rg>

# Update app settings
az functionapp config appsettings set \
  --name <app-name> \
  --resource-group <rg> \
  --settings KEY=VALUE
```

## Critical Notes
- 💡 **Serverless** - No infrastructure management
- ⚠️ **One trigger** - Each function has exactly one trigger
- 🎯 **Multiple bindings** - Functions can have many input/output bindings
- 📊 **Event-driven** - Code runs in response to events
- ✅ **Auto-scale** - Automatically scales based on load
- 🔄 **Integrated** - Works with Logic Apps, Event Grid, etc.
- ⏱️ **Pay-per-execution** - Charged only when code runs (Consumption plan)
- 🔒 **Managed runtime** - Microsoft handles patching, updates

## Exam Tips
- Azure Functions = serverless, event-driven compute
- Each function has exactly ONE trigger
- Functions can have multiple input/output bindings
- Functions vs Logic Apps: Code-first vs Designer-first
- Functions vs WebJobs: Serverless + autoscale vs App Service hosted
- WebJobs don't support HTTP triggers or Event Grid
- Triggers start execution, bindings simplify I/O
- Supported languages: C#, JavaScript, Python, Java, PowerShell, TypeScript
- Portal development: C# Script, JavaScript, PowerShell only
- Function app = container for multiple related functions
- host.json configures function app behavior
- Application Insights for monitoring (recommended)

[Learn More](https://learn.microsoft.com/en-us/training/modules/explore-azure-functions/2-azure-functions-overview)
