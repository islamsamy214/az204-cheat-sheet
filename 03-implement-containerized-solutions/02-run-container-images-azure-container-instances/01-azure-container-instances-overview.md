# Azure Container Instances Overview

## Key Concepts
- **ACI** - Fastest way to run containers in Azure
- **Container groups** - Collection of containers on same host (like Kubernetes pod)
- **Isolated containers** - No orchestration needed
- **Per-second billing** - Pay only for compute time used

## What Is Azure Container Instances?

**Fastest and simplest way** to run containers in Azure:

- No VM management required
- Start containers in seconds
- Run single containers or container groups
- Perfect for isolated workloads
- Hypervisor-level security

### When to Use ACI
✅ **Simple applications** - Single containers, no orchestration
✅ **Task automation** - Batch jobs, build tasks
✅ **Build jobs** - CI/CD pipeline tasks
✅ **Dev/test** - Quick container testing
✅ **Event-driven workloads** - Process messages, respond to events

### When to Use AKS Instead
❌ **Full orchestration** - Service discovery, auto-scaling
❌ **Complex apps** - Multiple interdependent services
❌ **Long-running services** - Continuous high availability

## Benefits

| Feature | Description |
|---------|-------------|
| **Fast startup** | Start containers in seconds (not minutes) |
| **Public IP + FQDN** | Expose containers with public IP and DNS name |
| **Hypervisor-level security** | Complete isolation (like VMs) |
| **Custom sizes** | Specify exact CPU cores and memory |
| **Persistent storage** | Mount Azure Files shares |
| **Linux & Windows** | Support both OS types |
| **No infrastructure** | No VMs to manage |

## Container Groups

### What Is a Container Group?

**Collection of containers** scheduled on the same host machine:

- Similar to Kubernetes **pod**
- Share lifecycle, resources, network, storage
- Top-level resource in ACI
- Co-located containers work together

### Container Group Architecture
```
Container Group (pod-like unit)
├── Container 1 (nginx)
│   └── Port 80 exposed
├── Container 2 (log collector)
│   └── Port 5000 exposed
├── Shared IP: 40.112.23.145
├── DNS: myapp.eastus.azurecontainer.io
├── Volume 1: Azure Files share → Container 1
└── Volume 2: Azure Files share → Container 2
```

### Key Characteristics
✅ **Single host** - All containers on same machine
✅ **Shared IP** - One public IP for all containers
✅ **Shared lifecycle** - Start/stop together
✅ **Shared network** - Containers communicate via localhost
✅ **Shared volumes** - Mount volumes to specific containers

⚠️ **Linux only** - Multi-container groups support Linux only (Windows = single container)

## Deployment Methods

### Azure CLI
```bash
# Single container
az container create \
  --resource-group myResourceGroup \
  --name mycontainer \
  --image nginx \
  --ports 80 \
  --dns-name-label myapp \
  --location eastus
```

### Resource Manager Template
```json
{
  "type": "Microsoft.ContainerInstance/containerGroups",
  "apiVersion": "2021-09-01",
  "name": "mycontainergroup",
  "location": "eastus",
  "properties": {
    "containers": [...],
    "osType": "Linux"
  }
}
```

### YAML File (Recommended for Multi-Container)
```yaml
apiVersion: '2021-09-01'
location: eastus
name: mycontainergroup
properties:
  containers:
  - name: nginx
    properties:
      image: nginx
      ports:
      - port: 80
  osType: Linux
```

**Recommendation**:
- **YAML** - For container-only deployments
- **ARM template** - When deploying other Azure resources too

## Resource Allocation

### How Resources Are Allocated

**Sum of all container requests** in the group:

```
Container Group CPU/Memory = Sum of all containers

Example:
- Container 1: 1 CPU, 1.5 GB memory
- Container 2: 0.5 CPU, 1 GB memory
- Total allocated: 1.5 CPU, 2.5 GB memory
```

### CPU and Memory Specs
```bash
# Specify resources
az container create \
  --resource-group myResourceGroup \
  --name mycontainer \
  --image myapp \
  --cpu 2 \
  --memory 4

# 2 CPU cores, 4 GB memory
```

### GPU Support (Preview)
```bash
# GPU-enabled container
az container create \
  --resource-group myResourceGroup \
  --name gpu-container \
  --image tensorflow/tensorflow:latest-gpu \
  --gpu-count 1 \
  --gpu-sku K80
```

## Networking

### IP Address & Ports
- **Single IP** - Shared by all containers in group
- **Port namespace** - Shared across containers
- **External access** - Expose ports on IP address
- **Internal communication** - Use localhost

### Port Configuration
```bash
# Expose multiple ports
az container create \
  --resource-group myResourceGroup \
  --name mycontainer \
  --image myapp \
  --ports 80 443 8080
```

### DNS Name Label
```bash
# Get FQDN
az container create \
  --resource-group myResourceGroup \
  --name mycontainer \
  --image nginx \
  --dns-name-label myapp \
  --location eastus

# FQDN: myapp.eastus.azurecontainer.io
```

⚠️ **Port mapping not supported** - Can't map host port 8080 to container port 80

### Localhost Communication
```yaml
# Container 1 can access Container 2 via localhost
apiVersion: '2021-09-01'
name: multi-container
properties:
  containers:
  - name: frontend
    properties:
      image: nginx
      ports:
      - port: 80
  - name: backend
    properties:
      image: api
      ports:
      - port: 5000  # Not exposed externally
      environmentVariables:
      - name: FRONTEND_URL
        value: http://localhost:80
  osType: Linux
```

## Storage Volumes

### Supported Volume Types

| Volume Type | Description | Use Case |
|-------------|-------------|----------|
| **Azure Files** | SMB file share | Persistent data |
| **Secret** | Sensitive data | Passwords, keys |
| **Empty directory** | Temporary storage | Scratch space |
| **Git repo** | Clone repository | Source code |

### Volume Example
```bash
# Mount Azure Files share
az container create \
  --resource-group myResourceGroup \
  --name mycontainer \
  --image myapp \
  --azure-file-volume-account-name mystorageaccount \
  --azure-file-volume-account-key <key> \
  --azure-file-volume-share-name myshare \
  --azure-file-volume-mount-path /data
```

## Common Scenarios

### 1. Web App + Content Puller
```
- Container 1: Nginx (serve web app)
- Container 2: Git sync (pull latest content)
```

### 2. App + Logging Sidecar
```
- Container 1: Application (write logs)
- Container 2: Log collector (ship logs to Azure Monitor)
```

### 3. App + Monitoring Sidecar
```
- Container 1: Application
- Container 2: Health checker (monitor and alert)
```

### 4. Front-End + Back-End
```
- Container 1: Web UI (public port 80)
- Container 2: API service (localhost port 5000)
```

## CLI Commands

### Create Container
```bash
# Basic container
az container create \
  --resource-group myResourceGroup \
  --name mycontainer \
  --image nginx \
  --ports 80

# With DNS and environment variables
az container create \
  --resource-group myResourceGroup \
  --name mycontainer \
  --image myapp:v1.0 \
  --dns-name-label myapp \
  --ports 80 443 \
  --environment-variables 'API_URL'='https://api.example.com' \
  --cpu 2 \
  --memory 4
```

### List Containers
```bash
# List container groups
az container list \
  --resource-group myResourceGroup \
  --output table

# Show container details
az container show \
  --resource-group myResourceGroup \
  --name mycontainer \
  --output json
```

### Container Logs
```bash
# View logs
az container logs \
  --resource-group myResourceGroup \
  --name mycontainer

# Follow logs
az container attach \
  --resource-group myResourceGroup \
  --name mycontainer
```

### Container State
```bash
# Start container
az container start \
  --resource-group myResourceGroup \
  --name mycontainer

# Stop container
az container stop \
  --resource-group myResourceGroup \
  --name mycontainer

# Restart container
az container restart \
  --resource-group myResourceGroup \
  --name mycontainer
```

### Delete Container
```bash
# Delete container group
az container delete \
  --resource-group myResourceGroup \
  --name mycontainer \
  --yes
```

### Exec Into Container
```bash
# Run command in container
az container exec \
  --resource-group myResourceGroup \
  --name mycontainer \
  --exec-command "/bin/bash"
```

## Pricing

### Billing Model
**Per-second billing** for CPU and memory:

```
Cost = (CPU cores × CPU price × seconds) + (GB memory × memory price × seconds)

Example (East US):
- CPU: $0.0000012/vCPU/second
- Memory: $0.0000001/GB/second

2 vCPU, 4 GB, 1 hour:
= (2 × $0.0000012 × 3600) + (4 × $0.0000001 × 3600)
= $0.01 per hour
```

### Cost Optimization
✅ **Stop when not needed** - Pay only while running
✅ **Right-size resources** - Don't over-provision
✅ **Restart policy** - Use `OnFailure` or `Never` for tasks
✅ **Spot containers** (Preview) - Lower cost for interruptible workloads

## Critical Notes
- 💡 **Fastest way** - Start containers in seconds without VMs
- ⚠️ **Container groups** - Like Kubernetes pods (shared host/network/lifecycle)
- 🎯 **Multi-container groups** - Linux only (Windows = single container)
- ✅ **Per-second billing** - Pay only for running time
- 📊 **Resource allocation** - Sum of all container requests
- 🔄 **Port namespace** - Shared, no port mapping
- 🔒 **Localhost communication** - Containers reach each other via localhost
- ⚠️ **Use AKS for orchestration** - ACI is for simple, isolated containers

## Exam Tips
- ACI = Fastest way to run containers in Azure (seconds, no VMs)
- Container group = Top-level resource (like Kubernetes pod)
- Multi-container groups: Linux only, Windows = single container only
- Container group features: Shared IP, lifecycle, network, storage volumes
- Resource allocation: Sum of all container CPU/memory requests
- Networking: Single IP per group, shared port namespace, no port mapping
- Localhost: Containers communicate within group via localhost
- Storage volumes: Azure Files, Secret, Empty directory, Git repo
- Deployment: CLI, ARM template, YAML (YAML for container-only)
- Billing: Per-second for CPU and memory (only while running)
- Use cases: Simple apps, task automation, build jobs, dev/test
- Use AKS instead: Full orchestration, service discovery, auto-scaling
- DNS: `--dns-name-label` creates FQDN like `name.region.azurecontainer.io`

[Learn More](https://learn.microsoft.com/en-us/training/modules/create-run-container-images-azure-container-instances/2-azure-container-instances-overview)
