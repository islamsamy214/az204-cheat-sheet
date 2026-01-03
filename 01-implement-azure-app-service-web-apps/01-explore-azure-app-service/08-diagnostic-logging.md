# Enable Diagnostic Logging

## Key Concepts
- **Built-in diagnostics** for debugging and monitoring
- **Multiple log types** for different purposes
- **Flexible storage** - File system or Azure Storage
- **Real-time streaming** and historical access

## Log Types Overview

| Log Type | Platform | Storage Options | Purpose |
|----------|----------|-----------------|---------|
| **Application Logging** | Windows, Linux | File System, Blob Storage (Windows) | App code messages |
| **Web Server Logging** | Windows | File System, Blob Storage | Raw HTTP requests (W3C format) |
| **Detailed Error Messages** | Windows | File System | HTML error pages (HTTP 400+) |
| **Failed Request Tracing** | Windows | File System | IIS request traces |
| **Deployment Logging** | Windows, Linux | File System | Auto-enabled, not configurable |

## Application Logging

### Windows Apps

#### Log Levels
| Level | Includes |
|-------|----------|
| **Disabled** | Nothing |
| **Error** | Error, Critical |
| **Warning** | Warning, Error, Critical |
| **Information** | Info, Warning, Error, Critical |
| **Verbose** | Trace, Debug, Info, Warning, Error, Critical |

#### Storage Options
- **File System**: Temp debugging, **auto-disables after 12 hours**
- **Blob Storage**: Long-term logging, requires container

```bash
# Enable app logging (File System)
az webapp log config \
  --name <app-name> \
  --resource-group <rg-name> \
  --application-logging filesystem \
  --level information

# Enable app logging (Blob Storage)
az webapp log config \
  --name <app-name> \
  --resource-group <rg-name> \
  --application-logging azureblobstorage \
  --level verbose \
  --docker-container-logging filesystem
```

⚠️ **Regenerate storage keys**: Must reconfigure logging after key rotation

### Linux/Container Apps

```bash
# Enable app logging (Linux)
az webapp log config \
  --name <app-name> \
  --resource-group <rg-name> \
  --application-logging true \
  --docker-container-logging filesystem \
  --level information
```

#### Linux-Specific Settings
- **Quota (MB)**: Disk quota for logs
- **Retention Period (Days)**: How long to keep logs

## Web Server Logging (Windows Only)

### Configuration

```bash
# Enable web server logging (File System)
az webapp log config \
  --name <app-name> \
  --resource-group <rg-name> \
  --web-server-logging filesystem

# Enable web server logging (Blob Storage)
az webapp log config \
  --name <app-name> \
  --resource-group <rg-name> \
  --web-server-logging storage
```

### W3C Extended Log Format
Includes:
- HTTP method (GET, POST, etc.)
- Resource URI
- Client IP and port
- User agent
- Response code
- Time taken

## Add Log Messages in Code

### ASP.NET

```csharp
// ASP.NET - System.Diagnostics.Trace
System.Diagnostics.Trace.TraceError("Something bad happened");
System.Diagnostics.Trace.TraceWarning("Warning message");
System.Diagnostics.Trace.TraceInformation("Info message");
```

### ASP.NET Core

```csharp
// ASP.NET Core - ILogger
public class HomeController : Controller
{
    private readonly ILogger<HomeController> _logger;
    
    public HomeController(ILogger<HomeController> logger)
    {
        _logger = logger;
    }
    
    public IActionResult Index()
    {
        _logger.LogInformation("Home page visited");
        _logger.LogWarning("This is a warning");
        _logger.LogError("An error occurred");
        return View();
    }
}
```

### Node.js

```javascript
// Node.js - console
console.log('Information message');
console.warn('Warning message');
console.error('Error message');
```

### Python

```python
# Python - OpenCensus package
import logging
from opencensus.ext.azure.log_exporter import AzureLogHandler

logger = logging.getLogger(__name__)
logger.addHandler(AzureLogHandler())

logger.info("Information message")
logger.warning("Warning message")
logger.error("Error message")
```

## Stream Logs

### Azure Portal
- Navigate to app → **Log stream**
- Real-time log viewing

### Azure CLI

```bash
# Stream logs in Cloud Shell
az webapp log tail \
  --name <app-name> \
  --resource-group <rg-name>

# Stream logs locally
az webapp log tail \
  --name <app-name> \
  --resource-group <rg-name> \
  --provider application
```

### Log Location
- Logs stored in: `/LogFiles` directory (`d:/home/logfiles`)
- Files ending in: `.txt`, `.log`, `.htm`

⚠️ **Out-of-order events**: Buffered writes may cause sequence issues

## Access Log Files

### Download ZIP Archive

| App Type | URL |
|----------|-----|
| **Linux/Container** | `https://<app-name>.scm.azurewebsites.net/api/logs/docker/zip` |
| **Windows** | `https://<app-name>.scm.azurewebsites.net/api/dump` |

```bash
# Download logs (CLI)
az webapp log download \
  --name <app-name> \
  --resource-group <rg-name> \
  --log-file logs.zip
```

### Blob Storage Logs
- Use **Storage Explorer** or Azure portal
- Requires blob storage client tool

## Log File Structure

### Linux/Container Apps
```
/home/LogFiles/
├── docker/          # Docker container logs
├── Application/     # Application logs
└── kudu/           # Deployment logs
```

### Windows Apps
```
d:/home/logfiles/
├── Application/     # Application logs
├── http/           # Web server logs
├── DetailedErrors/ # Detailed error pages
└── W3SVC*/         # Failed request traces
```

## Configuration Quick Reference

```bash
# Complete logging setup
az webapp log config \
  --name <app-name> \
  --resource-group <rg-name> \
  --application-logging filesystem \
  --detailed-error-messages true \
  --failed-request-tracing true \
  --web-server-logging filesystem \
  --level verbose
```

## Log Retention

| Storage | Default Retention | Configurable |
|---------|-------------------|--------------|
| **File System** | Depends on tier | ✅ Yes |
| **Blob Storage** | None (forever) | ✅ Yes (lifecycle policies) |

## Critical Notes
- 💡 **File System logging auto-disables after 12 hours** (Windows app logging)
- 🎯 **Use Blob Storage** for long-term logs
- ⚠️ **Regenerate storage keys** = Reconfigure logging
- 📊 **Deployment logging always on** - no configuration needed
- 🔄 **Buffer causes out-of-order events** in stream
- 🐧 **Linux has quota and retention settings** (File System)
- ⏰ **Stream from /LogFiles** directory only

## Best Practices

### Development
```bash
# Enable verbose logging
az webapp log config \
  --name dev-app \
  --resource-group dev-rg \
  --application-logging filesystem \
  --level verbose
```

### Production
```bash
# Enable error logging to Blob Storage
az webapp log config \
  --name prod-app \
  --resource-group prod-rg \
  --application-logging azureblobstorage \
  --level error \
  --web-server-logging storage
```

## Exam Tips
- Know the difference between File System (temp, 12hr) and Blob Storage (long-term)
- Understand log levels: Disabled < Error < Warning < Information < Verbose
- Remember application logging available on Windows and Linux
- Web server logging, detailed errors, failed request tracing: Windows only
- Deployment logging is automatic and always enabled
- Know how to access logs via URL pattern (`scm.azurewebsites.net`)

[Learn More](https://learn.microsoft.com/en-us/training/modules/configure-web-app-settings/5-enable-diagnostic-logging)
