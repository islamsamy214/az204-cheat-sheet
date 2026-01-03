# Environment Variables in Container Instances

## Key Concepts
- **Environment variables** - Dynamic configuration for containers
- **Standard variables** - Visible in portal and CLI
- **Secure values** - Hidden sensitive data (passwords, keys)
- **secureValue** - Property for sensitive information

## Why Use Environment Variables?

**Dynamic configuration** without rebuilding images:

- Configure apps at runtime
- Different settings per environment (dev/prod)
- Pass secrets securely
- Similar to `docker run --env`

### Benefits
✅ **No image rebuild** - Change config without rebuilding
✅ **Environment-specific** - Different values per deployment
✅ **Secure secrets** - Hide sensitive data
✅ **12-factor app** - Follow cloud-native principles

## Setting Environment Variables

### Azure CLI
```bash
# Single environment variable
az container create \
  --resource-group myResourceGroup \
  --name mycontainer \
  --image myapp \
  --environment-variables 'API_URL'='https://api.example.com'

# Multiple variables
az container create \
  --resource-group myResourceGroup \
  --name mycontainer \
  --image myapp \
  --environment-variables \
    'API_URL'='https://api.example.com' \
    'PORT'='8080' \
    'LOG_LEVEL'='info'
```

⚠️ **Shell-specific syntax**:
```bash
# Bash/Cloud Shell
--environment-variables 'KEY'='value' 'KEY2'='value2'

# Windows Command Prompt
--environment-variables "KEY"="value" "KEY2"="value2"

# PowerShell
--environment-variables 'KEY'='value','KEY2'='value2'
```

### YAML Configuration
```yaml
apiVersion: '2021-09-01'
location: eastus
name: mycontainer
properties:
  containers:
  - name: myapp
    properties:
      image: myapp:latest
      environmentVariables:
      - name: 'API_URL'
        value: 'https://api.example.com'
      - name: 'PORT'
        value: '8080'
      - name: 'LOG_LEVEL'
        value: 'info'
      resources:
        requests:
          cpu: 1
          memoryInGB: 1.5
  osType: Linux
```

## Secure Values

### What Are Secure Values?

**Hidden environment variables** for sensitive data:

- Not visible in Azure Portal
- Not shown in `az container show`
- Only accessible from within container
- Use for: passwords, API keys, connection strings

### Standard vs Secure Values

| Feature | Standard (`value`) | Secure (`secureValue`) |
|---------|-------------------|------------------------|
| **Visibility** | Visible in portal/CLI | Hidden |
| **In container** | Accessible | Accessible |
| **Use for** | Non-sensitive config | Passwords, keys |

### Setting Secure Values (CLI)
```bash
# Secure environment variables
az container create \
  --resource-group myResourceGroup \
  --name mycontainer \
  --image myapp \
  --environment-variables \
    'API_URL'='https://api.example.com' \
  --secure-environment-variables \
    'API_KEY'='super-secret-key-123' \
    'DB_PASSWORD'='MyP@ssw0rd!'
```

### Setting Secure Values (YAML)
```yaml
apiVersion: '2021-09-01'
location: eastus
name: secure-container
properties:
  containers:
  - name: myapp
    properties:
      image: myapp:latest
      environmentVariables:
      # Regular variable (visible)
      - name: 'API_URL'
        value: 'https://api.example.com'
      
      # Secure variable (hidden)
      - name: 'API_KEY'
        secureValue: 'super-secret-key-123'
      
      # Another secure variable
      - name: 'DB_PASSWORD'
        secureValue: 'MyP@ssw0rd!'
      
      resources:
        requests:
          cpu: 1
          memoryInGB: 1.5
  osType: Linux
  restartPolicy: Always
```

## Accessing Variables in Container

### Linux Container (Bash)
```bash
#!/bin/bash
echo "API URL: $API_URL"
echo "Port: $PORT"
echo "API Key: $API_KEY"  # Works even though secureValue

# Use in application
curl -H "Authorization: Bearer $API_KEY" $API_URL
```

### Node.js
```javascript
// process.env contains all environment variables
const apiUrl = process.env.API_URL;
const apiKey = process.env.API_KEY;  // Accessible
const port = process.env.PORT || 3000;

console.log(`Connecting to ${apiUrl}`);
console.log(`API Key: ${apiKey}`);  // Can access secure values
```

### Python
```python
import os

api_url = os.environ.get('API_URL')
api_key = os.environ.get('API_KEY')  # Accessible
port = int(os.environ.get('PORT', 8080))

print(f"API URL: {api_url}")
print(f"API Key: {api_key}")  # Can access secure values
```

### .NET/C#
```csharp
string apiUrl = Environment.GetEnvironmentVariable("API_URL");
string apiKey = Environment.GetEnvironmentVariable("API_KEY");  // Accessible
int port = int.Parse(Environment.GetEnvironmentVariable("PORT") ?? "8080");

Console.WriteLine($"API URL: {apiUrl}");
Console.WriteLine($"API Key: {apiKey}");  # Can access secure values
```

## Viewing Variables

### List Environment Variables (Standard Only)
```bash
# Show container details (secure values hidden)
az container show \
  --resource-group myResourceGroup \
  --name mycontainer \
  --query "properties.containers[0].properties.environmentVariables"

# Output shows:
# - Regular variables: name + value
# - Secure variables: name only (no value)
```

### Example Output
```json
[
  {
    "name": "API_URL",
    "value": "https://api.example.com"
  },
  {
    "name": "API_KEY",
    "secureValue": null  // Value is hidden
  },
  {
    "name": "DB_PASSWORD",
    "secureValue": null  // Value is hidden
  }
]
```

### Cannot View Secure Values
```bash
# These commands won't show secure values:
az container show ...
az container export ...

# Only accessible from within the running container
```

## Practical Examples

### Example 1: Web App with Database
```bash
az container create \
  --resource-group myResourceGroup \
  --name webapp \
  --image webapp:latest \
  --dns-name-label mywebapp \
  --ports 80 \
  --environment-variables \
    'ENVIRONMENT'='production' \
    'LOG_LEVEL'='warn' \
  --secure-environment-variables \
    'DB_CONNECTION_STRING'='Server=mydb.database.windows.net;Database=mydb;User=admin;Password=P@ssw0rd;' \
    'JWT_SECRET'='my-super-secret-jwt-key'
```

### Example 2: Batch Job with API
```yaml
apiVersion: '2021-09-01'
location: eastus
name: batch-processor
properties:
  containers:
  - name: processor
    properties:
      image: batch-processor:v1.0
      environmentVariables:
      - name: 'BATCH_SIZE'
        value: '100'
      - name: 'API_ENDPOINT'
        value: 'https://api.example.com/v2'
      - name: 'API_KEY'
        secureValue: 'sk-1234567890abcdef'
      - name: 'STORAGE_CONNECTION'
        secureValue: 'DefaultEndpointsProtocol=https;AccountName=...'
      resources:
        requests:
          cpu: 2
          memoryInGB: 4
  osType: Linux
  restartPolicy: OnFailure
```

### Example 3: Multi-Container with Shared Config
```yaml
apiVersion: '2021-09-01'
location: eastus
name: app-with-sidecar
properties:
  containers:
  # Main application
  - name: webapp
    properties:
      image: webapp:latest
      ports:
      - port: 80
      environmentVariables:
      - name: 'APP_NAME'
        value: 'My Web App'
      - name: 'API_KEY'
        secureValue: 'secret-api-key'
      resources:
        requests:
          cpu: 1
          memoryInGB: 1.5
  
  # Logging sidecar
  - name: log-collector
    properties:
      image: fluentd:latest
      environmentVariables:
      - name: 'LOG_DESTINATION'
        value: 'https://logs.example.com'
      - name: 'AUTH_TOKEN'
        secureValue: 'log-collector-token'
      resources:
        requests:
          cpu: 0.5
          memoryInGB: 0.5
  
  osType: Linux
  restartPolicy: Always
```

## Common Patterns

### Pattern 1: Environment-Specific Config
```bash
# Development
az container create \
  --name webapp-dev \
  --image webapp:latest \
  --environment-variables \
    'ENVIRONMENT'='development' \
    'DEBUG'='true' \
  --secure-environment-variables \
    'API_KEY'='dev-api-key'

# Production
az container create \
  --name webapp-prod \
  --image webapp:latest \
  --environment-variables \
    'ENVIRONMENT'='production' \
    'DEBUG'='false' \
  --secure-environment-variables \
    'API_KEY'='prod-api-key'
```

### Pattern 2: Feature Flags
```yaml
environmentVariables:
- name: 'FEATURE_NEW_UI'
  value: 'true'
- name: 'FEATURE_BETA_API'
  value: 'false'
- name: 'MAX_CONNECTIONS'
  value: '100'
```

### Pattern 3: Service Discovery
```yaml
# Frontend container
environmentVariables:
- name: 'BACKEND_URL'
  value: 'http://localhost:5000'  # Backend in same container group
- name: 'CACHE_URL'
  value: 'myredis.redis.cache.windows.net'
```

## Best Practices

### 1. Use Secure Values for Secrets
```yaml
# ✅ GOOD
- name: 'API_KEY'
  secureValue: 'secret-key'

# ❌ BAD
- name: 'API_KEY'
  value: 'secret-key'  # Visible in portal/CLI
```

### 2. Use Key Vault References (Better)
```bash
# Best: Store secrets in Key Vault
az keyvault secret set \
  --vault-name myvault \
  --name api-key \
  --value 'super-secret-key'

# Reference in container (requires managed identity)
# Not natively supported by ACI - use init script to fetch
```

### 3. Don't Commit Secrets to Source Control
```yaml
# ❌ DON'T commit this file with secrets
apiVersion: '2021-09-01'
...
environmentVariables:
- name: 'PASSWORD'
  secureValue: 'hardcoded-password'  # Bad!

# ✅ DO use parameter files or CI/CD variables
```

### 4. Validate Environment Variables
```python
import os
import sys

# Validate required variables
required_vars = ['API_URL', 'API_KEY', 'DB_CONNECTION']
missing = [var for var in required_vars if not os.environ.get(var)]

if missing:
    print(f"Missing required variables: {', '.join(missing)}")
    sys.exit(1)
```

### 5. Use Default Values
```javascript
// Provide sensible defaults
const port = process.env.PORT || 3000;
const logLevel = process.env.LOG_LEVEL || 'info';
const apiUrl = process.env.API_URL || 'http://localhost:8080';
```

## Updating Environment Variables

⚠️ **Cannot update existing container** - Must recreate:

```bash
# Delete old container
az container delete \
  --resource-group myResourceGroup \
  --name mycontainer \
  --yes

# Create new with updated variables
az container create \
  --resource-group myResourceGroup \
  --name mycontainer \
  --image myapp \
  --environment-variables \
    'API_URL'='https://new-api.example.com'  # Updated
```

## Critical Notes
- 💡 **Environment variables** - Dynamic configuration without rebuilding
- ⚠️ **secureValue** - Use for sensitive data (hidden from portal/CLI)
- 🎯 **value vs secureValue** - Regular vs secure environment variables
- ✅ **Accessible in container** - Both types accessible via env vars
- 📊 **Cannot update** - Must delete and recreate container
- 🔒 **Best practice** - Use secure values for secrets
- 🔑 **Key Vault better** - Store secrets in Azure Key Vault (fetch at startup)
- ⚠️ **Shell syntax** - Different quote styles for Bash vs CMD vs PowerShell

## Exam Tips
- Environment variables: Dynamic configuration without rebuilding image
- Set with: `--environment-variables` (CLI) or `environmentVariables` (YAML)
- Standard variables: Use `value` property (visible in portal/CLI)
- Secure variables: Use `secureValue` property (hidden from portal/CLI)
- Both types accessible from within container via environment variables
- Secure values: Only visible inside running container
- CLI secure flag: `--secure-environment-variables`
- Cannot view secure values after creation (only container can access)
- Cannot update variables: Must delete and recreate container
- Shell syntax: Bash `'KEY'='value'`, CMD `"KEY"="value"`
- Use for: API keys, passwords, connection strings (secure values)
- Best practice: Store secrets in Key Vault, not environment variables
- 12-factor app: Configuration via environment variables

[Learn More](https://learn.microsoft.com/en-us/training/modules/create-run-container-images-azure-container-instances/5-set-environment-variables-azure-container-instances)
