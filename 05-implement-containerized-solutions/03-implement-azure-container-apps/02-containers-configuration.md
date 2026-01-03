# Containers in Azure Container Apps

## Key Concepts
- **Linux containers** - Linux x86-64 (linux/amd64) only
- **Multiple containers** - Sidecar pattern support
- **Configuration** - ARM templates, environment variables, probes
- **No privileged mode** - Containers run without root access

## Container Support

### Supported Container Types
✅ **Linux x86-64** - linux/amd64 architecture
✅ **Any runtime** - Node.js, Python, Java, .NET, Go, etc.
✅ **Any registry** - Docker Hub, ACR, private registries
✅ **Multiple containers** - Sidecar pattern

### Limitations
❌ **No privileged containers** - No root access required processes
❌ **Windows not supported** - Linux containers only
❌ **No ARM architecture** - Only x86-64 (amd64)

| Feature | Supported |
|---------|-----------|
| Linux containers | ✅ Yes |
| Windows containers | ❌ No |
| x86-64 (amd64) | ✅ Yes |
| ARM architecture | ❌ No |
| Privileged mode | ❌ No |
| Root access | ❌ Causes runtime error |

## Container Configuration

### ARM Template Example
```json
{
  "containers": [
    {
      "name": "main",
      "image": "myregistry.azurecr.io/myapp:v1.0",
      "env": [
        {
          "name": "HTTP_PORT",
          "value": "80"
        },
        {
          "name": "SECRET_VAL",
          "secretRef": "mysecret"
        }
      ],
      "resources": {
        "cpu": 0.5,
        "memory": "1Gi"
      },
      "volumeMounts": [
        {
          "mountPath": "/myfiles",
          "volumeName": "azure-files-volume"
        }
      ],
      "probes": [
        {
          "type": "liveness",
          "httpGet": {
            "path": "/health",
            "port": 8080
          },
          "initialDelaySeconds": 7,
          "periodSeconds": 3
        }
      ]
    }
  ]
}
```

### Configuration Options

| Setting | Description | Example |
|---------|-------------|---------|
| **name** | Container name | "main" |
| **image** | Container image | "nginx:alpine" |
| **env** | Environment variables | [{"name": "PORT", "value": "80"}] |
| **resources** | CPU and memory | {"cpu": 0.5, "memory": "1Gi"} |
| **volumeMounts** | Volume mounts | [{"mountPath": "/data", "volumeName": "vol"}] |
| **probes** | Health checks | Liveness, readiness, startup |

## Environment Variables

### Standard Variables
```json
{
  "env": [
    {
      "name": "API_URL",
      "value": "https://api.example.com"
    },
    {
      "name": "PORT",
      "value": "8080"
    }
  ]
}
```

### Secret References
```json
{
  "env": [
    {
      "name": "CONNECTION_STRING",
      "secretRef": "db-connection"
    },
    {
      "name": "API_KEY",
      "secretRef": "api-key-secret"
    }
  ]
}
```

## Resource Allocation

### CPU and Memory Limits
```json
{
  "resources": {
    "cpu": 0.5,        // 0.5 vCPU
    "memory": "1Gi"    // 1 GB memory
  }
}
```

### Common Configurations

| App Type | CPU | Memory |
|----------|-----|--------|
| **Lightweight API** | 0.25 | 0.5 Gi |
| **Standard app** | 0.5 | 1 Gi |
| **Heavy processing** | 1.0 | 2 Gi |
| **Large workload** | 2.0 | 4 Gi |

## Health Probes

### Liveness Probe
**Checks if container is alive**:

```json
{
  "type": "liveness",
  "httpGet": {
    "path": "/health",
    "port": 8080,
    "httpHeaders": [
      {
        "name": "Custom-Header",
        "value": "liveness probe"
      }
    ]
  },
  "initialDelaySeconds": 7,
  "periodSeconds": 3
}
```

### Readiness Probe
**Checks if container is ready to serve traffic**:

```json
{
  "type": "readiness",
  "httpGet": {
    "path": "/ready",
    "port": 8080
  },
  "initialDelaySeconds": 5,
  "periodSeconds": 5
}
```

### Startup Probe
**Checks if container has started**:

```json
{
  "type": "startup",
  "httpGet": {
    "path": "/startup",
    "port": 8080
  },
  "initialDelaySeconds": 0,
  "periodSeconds": 3,
  "failureThreshold": 30
}
```

## Multiple Containers (Sidecar Pattern)

### When to Use Sidecars

✅ **Log collector** - Forward logs from main app
✅ **Cache refresher** - Update cache in shared volume
✅ **Proxy** - Service mesh, authentication
✅ **Monitoring** - Health checks, metrics collection

### Example: App + Log Collector
```json
{
  "containers": [
    {
      "name": "webapp",
      "image": "webapp:latest",
      "resources": {
        "cpu": 0.5,
        "memory": "1Gi"
      },
      "volumeMounts": [
        {
          "mountPath": "/var/log/app",
          "volumeName": "logs"
        }
      ]
    },
    {
      "name": "log-collector",
      "image": "fluentd:latest",
      "resources": {
        "cpu": 0.25,
        "memory": "0.5Gi"
      },
      "volumeMounts": [
        {
          "mountPath": "/logs",
          "volumeName": "logs"
        }
      ]
    }
  ]
}
```

### Sidecar Communication
- **Shared volumes** - Exchange files
- **Localhost** - Network communication
- **Same lifecycle** - Start and stop together

⚠️ **Note**: For independent microservices, use separate container apps (not sidecars)

## Container Registries

### Azure Container Registry
```json
{
  "registries": [
    {
      "server": "myregistry.azurecr.io",
      "username": "myregistry",
      "passwordSecretRef": "registry-password"
    }
  ]
}
```

### Docker Hub (Private)
```json
{
  "registries": [
    {
      "server": "docker.io",
      "username": "my-docker-username",
      "passwordSecretRef": "docker-password"
    }
  ]
}
```

### Using Managed Identity (ACR)
```bash
# Create container app with managed identity for ACR
az containerapp create \
  --name myapp \
  --resource-group myResourceGroup \
  --environment myenvironment \
  --image myregistry.azurecr.io/myapp:latest \
  --registry-server myregistry.azurecr.io \
  --registry-identity system
```

## Volume Mounts

### Azure Files Volume
```json
{
  "volumeMounts": [
    {
      "mountPath": "/data",
      "volumeName": "azure-files-volume"
    }
  ],
  "volumes": [
    {
      "name": "azure-files-volume",
      "storageType": "AzureFile",
      "storageName": "myfilestorage"
    }
  ]
}
```

### EmptyDir Volume (Temporary)
```json
{
  "volumeMounts": [
    {
      "mountPath": "/tmp",
      "volumeName": "temp-storage"
    }
  ],
  "volumes": [
    {
      "name": "temp-storage",
      "storageType": "EmptyDir"
    }
  ]
}
```

## Automatic Restart

### Restart on Crash
✅ **Automatic** - Containers restart automatically on crash
✅ **Health probes** - Restart if liveness probe fails
✅ **No manual intervention** - Self-healing

## Critical Notes
- 💡 **Linux only** - Linux x86-64 (amd64) containers only
- ⚠️ **No privileged mode** - Runtime error if root access needed
- 🎯 **Multiple containers** - Sidecar pattern for related containers
- ✅ **Health probes** - Liveness, readiness, startup
- 📊 **Resource limits** - CPU (vCPU) and memory (Gi)
- 🔄 **Auto restart** - Containers restart on crash
- 🔒 **Secret refs** - Reference secrets in environment variables
- ⚠️ **Volume mounts** - Azure Files or EmptyDir

## Exam Tips
- Linux containers only (Windows not supported)
- Architecture: x86-64 (amd64) only
- No privileged containers (no root access)
- Multiple containers: Sidecar pattern (shared lifecycle, volumes, network)
- Configuration changes: Trigger new revision
- Environment variables: Standard (value) or secret reference (secretRef)
- Resources: CPU (vCPU), memory (Gi notation)
- Health probes: Liveness (alive?), readiness (ready?), startup (started?)
- Probe types: httpGet, tcpSocket, exec
- Container registries: Public, ACR (with managed identity), private (with credentials)
- Volume mounts: Azure Files (persistent), EmptyDir (temporary)
- Auto restart: On crash or failed liveness probe
- Sidecar use cases: Logging, caching, monitoring, proxy

[Learn More](https://learn.microsoft.com/en-us/training/modules/implement-azure-container-apps/4-container-apps-containers)
