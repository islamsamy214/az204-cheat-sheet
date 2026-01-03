# Mount Azure File Share in Container Instances

## Key Concepts
- **Persistent storage** - Data survives container restarts
- **Azure Files** - Fully managed SMB file shares
- **Volume mount** - Attach storage to container paths
- **Linux only** - File share mounting (Linux containers)

## Why Mount Azure Files?

**Persist data beyond container lifecycle**:

- Container storage is ephemeral (lost on restart)
- Share data between containers
- Store application data, logs, config
- Backup and recovery
- Share files across container instances

### Problem Without Persistent Storage
```
Container created → Data written → Container stops → Data LOST
```

### Solution With Azure Files
```
Container created → Data written to Azure Files → Container stops → Data PERSISTS
```

## Azure Files Overview

**Fully managed file shares** in the cloud:

- **SMB protocol** - Industry standard (CIFS)
- **Accessible** - From anywhere (VMs, containers, on-premises)
- **Shared** - Multiple containers can mount same share
- **Persistent** - Data survives container lifecycle

## Limitations

### Important Constraints
❌ **Linux only** - File share mounting not supported for Windows containers
❌ **Root required** - Linux container must run as root
❌ **CIFS only** - Limited to CIFS support (no NFS)
❌ **Single share per container** - Can mount multiple shares to different containers in group

| Feature | Supported |
|---------|-----------|
| **Linux containers** | ✅ Yes |
| **Windows containers** | ❌ No |
| **Root user** | ✅ Required |
| **CIFS/SMB** | ✅ Yes |
| **NFS** | ❌ No |

## Prerequisites

### Create Storage Account and File Share
```bash
# Create storage account
az storage account create \
  --resource-group myResourceGroup \
  --name mystorageaccount \
  --location eastus \
  --sku Standard_LRS

# Get storage account key
STORAGE_KEY=$(az storage account keys list \
  --resource-group myResourceGroup \
  --account-name mystorageaccount \
  --query "[0].value" \
  --output tsv)

# Create file share
az storage share create \
  --name myshare \
  --account-name mystorageaccount \
  --account-key $STORAGE_KEY
```

## Mount File Share (Azure CLI)

### Basic Mount
```bash
# Create container with mounted Azure Files
az container create \
  --resource-group myResourceGroup \
  --name mycontainer \
  --image nginx \
  --azure-file-volume-account-name mystorageaccount \
  --azure-file-volume-account-key $STORAGE_KEY \
  --azure-file-volume-share-name myshare \
  --azure-file-volume-mount-path /data

# Files in /data persist across container restarts
```

### Full Example
```bash
# Variables
RESOURCE_GROUP="myResourceGroup"
STORAGE_ACCOUNT="mystorageaccount"
SHARE_NAME="myshare"
STORAGE_KEY=$(az storage account keys list \
  --resource-group $RESOURCE_GROUP \
  --account-name $STORAGE_ACCOUNT \
  --query "[0].value" --output tsv)

# Create container with Azure Files
az container create \
  --resource-group $RESOURCE_GROUP \
  --name webapp \
  --image webapp:latest \
  --dns-name-label mywebapp \
  --ports 80 \
  --azure-file-volume-account-name $STORAGE_ACCOUNT \
  --azure-file-volume-account-key $STORAGE_KEY \
  --azure-file-volume-share-name $SHARE_NAME \
  --azure-file-volume-mount-path /app/data

# Application writes to /app/data
# Data persists in Azure Files myshare
```

## Mount File Share (YAML)

### Single Volume
```yaml
apiVersion: '2021-09-01'
location: eastus
name: file-share-demo
properties:
  containers:
  - name: webapp
    properties:
      image: nginx
      ports:
      - port: 80
      resources:
        requests:
          cpu: 1.0
          memoryInGB: 1.5
      volumeMounts:
      - name: filesharevolume
        mountPath: /data
  
  osType: Linux
  restartPolicy: Always
  
  ipAddress:
    type: Public
    ports:
    - port: 80
    dnsNameLabel: webapp-demo
  
  volumes:
  - name: filesharevolume
    azureFile:
      shareName: myshare
      storageAccountName: mystorageaccount
      storageAccountKey: <storage-account-key>
```

### Deploy YAML
```bash
# Deploy container group with YAML
az container create \
  --resource-group myResourceGroup \
  --file container-with-volume.yaml
```

## Mount Multiple Volumes

### Multiple File Shares to Same Container
```yaml
apiVersion: '2021-09-01'
location: eastus
name: multi-volume-demo
properties:
  containers:
  - name: myapp
    properties:
      image: myapp:latest
      resources:
        requests:
          cpu: 1.0
          memoryInGB: 1.5
      volumeMounts:
      # Mount share1 to /app/data
      - name: datavolume
        mountPath: /app/data
      # Mount share2 to /app/logs
      - name: logvolume
        mountPath: /app/logs
  
  osType: Linux
  
  volumes:
  # First volume
  - name: datavolume
    azureFile:
      shareName: share1
      storageAccountName: mystorageaccount
      storageAccountKey: <key>
  # Second volume
  - name: logvolume
    azureFile:
      shareName: share2
      storageAccountName: mystorageaccount
      storageAccountKey: <key>
```

### Multiple Containers with Different Mounts
```yaml
apiVersion: '2021-09-01'
location: eastus
name: multi-container-volumes
properties:
  containers:
  # Container 1: Write data
  - name: writer
    properties:
      image: writer:latest
      volumeMounts:
      - name: shareddata
        mountPath: /output
      resources:
        requests:
          cpu: 1
          memoryInGB: 1
  
  # Container 2: Read data
  - name: reader
    properties:
      image: reader:latest
      volumeMounts:
      - name: shareddata
        mountPath: /input
      resources:
        requests:
          cpu: 1
          memoryInGB: 1
  
  osType: Linux
  
  volumes:
  - name: shareddata
    azureFile:
      shareName: shared
      storageAccountName: mystorageaccount
      storageAccountKey: <key>
```

## Practical Examples

### Example 1: Web App with Persistent Uploads
```bash
# Web app that stores uploaded files
az container create \
  --resource-group myResourceGroup \
  --name webapp \
  --image webapp:latest \
  --dns-name-label mywebapp \
  --ports 80 443 \
  --azure-file-volume-account-name mystorageaccount \
  --azure-file-volume-account-key $STORAGE_KEY \
  --azure-file-volume-share-name uploads \
  --azure-file-volume-mount-path /app/uploads \
  --environment-variables \
    'UPLOAD_DIR'='/app/uploads'

# Users upload files → Stored in Azure Files
# Container restarts → Files still available
```

### Example 2: Log Collection
```yaml
apiVersion: '2021-09-01'
location: eastus
name: app-with-logging
properties:
  containers:
  # Application
  - name: webapp
    properties:
      image: webapp:latest
      ports:
      - port: 80
      volumeMounts:
      - name: logs
        mountPath: /var/log/app
      resources:
        requests:
          cpu: 1
          memoryInGB: 1.5
  
  # Log collector sidecar
  - name: log-collector
    properties:
      image: fluentd:latest
      volumeMounts:
      - name: logs
        mountPath: /logs
      resources:
        requests:
          cpu: 0.5
          memoryInGB: 0.5
  
  osType: Linux
  
  volumes:
  - name: logs
    azureFile:
      shareName: applogs
      storageAccountName: mystorageaccount
      storageAccountKey: <key>
```

### Example 3: Configuration Files
```bash
# Upload config file to file share first
az storage file upload \
  --account-name mystorageaccount \
  --account-key $STORAGE_KEY \
  --share-name config \
  --source config.json \
  --path config.json

# Container reads config from mounted share
az container create \
  --resource-group myResourceGroup \
  --name myapp \
  --image myapp:latest \
  --azure-file-volume-account-name mystorageaccount \
  --azure-file-volume-account-key $STORAGE_KEY \
  --azure-file-volume-share-name config \
  --azure-file-volume-mount-path /etc/config

# Application reads /etc/config/config.json
```

### Example 4: Data Processing Pipeline
```yaml
apiVersion: '2021-09-01'
location: eastus
name: data-pipeline
properties:
  containers:
  - name: processor
    properties:
      image: data-processor:latest
      volumeMounts:
      - name: input
        mountPath: /input
      - name: output
        mountPath: /output
      resources:
        requests:
          cpu: 2
          memoryInGB: 4
  
  osType: Linux
  restartPolicy: OnFailure
  
  volumes:
  # Input data from file share
  - name: input
    azureFile:
      shareName: input-data
      storageAccountName: mystorageaccount
      storageAccountKey: <key>
  # Write results to file share
  - name: output
    azureFile:
      shareName: output-data
      storageAccountName: mystorageaccount
      storageAccountKey: <key>
```

## Accessing Files

### From Container
```bash
# Inside container
ls /data
cat /data/file.txt
echo "Hello" > /data/newfile.txt
```

### From Azure Portal
1. Go to Storage Account
2. Navigate to File shares
3. Select your share
4. Browse/upload/download files

### Using Azure CLI
```bash
# List files
az storage file list \
  --account-name mystorageaccount \
  --account-key $STORAGE_KEY \
  --share-name myshare \
  --output table

# Upload file
az storage file upload \
  --account-name mystorageaccount \
  --account-key $STORAGE_KEY \
  --share-name myshare \
  --source localfile.txt \
  --path remotefile.txt

# Download file
az storage file download \
  --account-name mystorageaccount \
  --account-key $STORAGE_KEY \
  --share-name myshare \
  --path remotefile.txt \
  --dest localfile.txt
```

### Using Storage Explorer
- Install Azure Storage Explorer
- Connect to storage account
- Browse file shares
- Upload/download files

## Volume Types Comparison

| Volume Type | Persistence | Use Case |
|-------------|-------------|----------|
| **Azure Files** | Persistent | App data, logs, uploads |
| **emptyDir** | Temporary | Scratch space, caching |
| **gitRepo** | Read-only | Source code, config |
| **secret** | Sensitive data | Passwords, certificates |

### Azure Files Volume
```yaml
volumes:
- name: persistent
  azureFile:
    shareName: myshare
    storageAccountName: mystorageaccount
    storageAccountKey: <key>
```

### Empty Directory (Temporary)
```yaml
volumes:
- name: scratch
  emptyDir: {}
```

### Git Repo
```yaml
volumes:
- name: code
  gitRepo:
    repository: https://github.com/user/repo.git
    directory: .
```

### Secret
```yaml
volumes:
- name: secrets
  secret:
    secretName: my-secret
```

## Security Considerations

### Storage Account Key
⚠️ **Storage account key** is highly sensitive:

```yaml
# ❌ BAD: Hardcoded in YAML
storageAccountKey: "abcd1234..."  # Don't commit to source control

# ✅ BETTER: Use parameter or CI/CD variable
storageAccountKey: ${STORAGE_KEY}

# ✅ BEST: Use managed identity (not yet supported for ACI file shares)
```

### Read-Only Mounts
```yaml
# Mount as read-only (prevents writes)
volumeMounts:
- name: configvolume
  mountPath: /etc/config
  readOnly: true
```

### Access Control
```bash
# Limit access with SAS token (future)
# Currently requires storage account key
```

## Troubleshooting

### Common Issues

#### Mount Fails
```bash
# Check storage account key
az storage account keys list \
  --resource-group myResourceGroup \
  --account-name mystorageaccount

# Verify file share exists
az storage share show \
  --name myshare \
  --account-name mystorageaccount \
  --account-key $STORAGE_KEY
```

#### Permission Denied
```yaml
# Ensure container runs as root
securityContext:
  runAsUser: 0  # Root user
```

#### Windows Container Error
```
# Error: Volume mount not supported for Windows containers
# Solution: Use Linux containers
osType: Linux  # Not Windows
```

## Best Practices

### 1. Separate Shares for Different Purposes
```bash
# Separate shares
az storage share create --name app-data
az storage share create --name app-logs
az storage share create --name app-uploads
```

### 2. Use Descriptive Mount Paths
```yaml
volumeMounts:
- name: datavolume
  mountPath: /app/data  # Clear purpose
- name: logvolume
  mountPath: /var/log/app  # Standard location
```

### 3. Protect Storage Keys
```bash
# Store in Key Vault
az keyvault secret set \
  --vault-name myvault \
  --name storage-key \
  --value $STORAGE_KEY

# Reference in CI/CD (don't hardcode)
```

### 4. Monitor Storage Usage
```bash
# Check file share usage
az storage share stats \
  --name myshare \
  --account-name mystorageaccount \
  --account-key $STORAGE_KEY
```

### 5. Backup Important Data
```bash
# Snapshot file share
az storage share snapshot \
  --name myshare \
  --account-name mystorageaccount \
  --account-key $STORAGE_KEY
```

## Critical Notes
- 💡 **Persistent storage** - Azure Files survive container restarts
- ⚠️ **Linux only** - File share mounting not supported for Windows containers
- 🎯 **Root required** - Container must run as root user
- ✅ **SMB/CIFS** - Standard protocol (not NFS)
- 📊 **Multiple mounts** - Can mount different shares to different paths
- 🔄 **Shared storage** - Multiple containers can mount same share
- 🔒 **Storage key** - Required for mount (highly sensitive)
- ⚠️ **Read-only** - Can mount as read-only with readOnly: true

## Exam Tips
- Azure Files: Persistent storage for containers (SMB/CIFS)
- Linux containers only (Windows not supported)
- Container must run as root to mount file share
- Mount with: `--azure-file-volume-*` flags (CLI)
- YAML: `volumes` (define), `volumeMounts` (use in container)
- Multiple volumes: Define multiple in `volumes` array
- Storage account key required (sensitive - protect it)
- Mount path: Where volume appears in container filesystem
- Shared storage: Multiple containers can mount same share
- Use cases: App data, logs, uploads, configuration, shared data
- Alternative volumes: emptyDir (temp), gitRepo (read-only), secret (sensitive)
- Read-only: Set `readOnly: true` in volumeMount
- Troubleshooting: Verify key, share exists, Linux container, root user
- Best practice: Separate shares for different data types

[Learn More](https://learn.microsoft.com/en-us/training/modules/create-run-container-images-azure-container-instances/6-mount-azure-file-share-azure-container-instances)
