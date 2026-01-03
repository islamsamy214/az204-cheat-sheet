# Azure Container Registry Storage Capabilities

## Key Concepts
- **Encryption-at-rest** - All images encrypted automatically
- **Regional storage** - Data stored in registry region
- **Geo-replication** - Multi-region registry (Premium only)
- **Zone redundancy** - Availability zones (Premium only)
- **Scalable storage** - Unlimited repositories within tier limits

## Storage Features

### Encryption-at-Rest
**Automatic encryption** for all registry content:

- Images encrypted before storage
- Automatic decryption on pull
- Azure-managed encryption keys
- Optional: Customer-managed keys (CMK)

```bash
# Enable customer-managed key (Premium only)
az acr encryption set \
  --resource-group myResourceGroup \
  --name myregistry \
  --key-encryption-key <key-identifier> \
  --identity <managed-identity-id>
```

### Regional Storage
**Data residency** for compliance:

| Region | Storage Location | Paired Region |
|--------|-----------------|---------------|
| **Most regions** | Primary + paired region | Yes |
| **Brazil South** | Brazil South only | No |
| **Southeast Asia** | Southeast Asia only | No |

⚠️ **Regional outage**: Data may become unavailable and is not automatically recovered

## Geo-Replication (Premium Only)

### Benefits
✅ **High availability** - Guard against regional failures
✅ **Network-close storage** - Faster pushes/pulls
✅ **Single registry** - Manage one registry across regions
✅ **Regional deployment** - Deploy closer to users

### How It Works
```
Primary Registry (East US)
├── Replica (West Europe)
├── Replica (Southeast Asia)
└── Replica (Australia East)

- Single registry name: myregistry.azurecr.io
- Automatic image replication
- Regional endpoints for fast pulls
- Centralized management
```

### CLI Commands
```bash
# Enable geo-replication (Premium required)
az acr replication create \
  --resource-group myResourceGroup \
  --registry myregistry \
  --location westeurope

# List replicas
az acr replication list \
  --registry myregistry \
  --output table

# Delete replica
az acr replication delete \
  --resource-group myResourceGroup \
  --registry myregistry \
  --name westeurope
```

### Replication Workflow
```bash
# 1. Push to primary region
docker push myregistry.azurecr.io/myapp:v1.0

# 2. Automatic replication to all replicas
# myregistry-eastus.azurecr.io/myapp:v1.0
# myregistry-westeurope.azurecr.io/myapp:v1.0
# myregistry-southeastasia.azurecr.io/myapp:v1.0

# 3. Pull from nearest region automatically
docker pull myregistry.azurecr.io/myapp:v1.0
```

## Zone Redundancy (Premium Only)

### Availability Zones
**Replicate registry within a region**:

- Minimum 3 separate zones per region
- Protects against zone failures
- Available in [supported regions](https://learn.microsoft.com/en-us/azure/availability-zones/az-overview)

```bash
# Enable zone redundancy at creation
az acr create \
  --resource-group myResourceGroup \
  --name myregistry \
  --sku Premium \
  --zone-redundancy enabled

# Enable on existing registry
az acr update \
  --resource-group myResourceGroup \
  --name myregistry \
  --zone-redundancy enabled
```

## Scalable Storage

### Storage Limits by Tier

| Tier | Included Storage | Max Storage | Bandwidth |
|------|-----------------|-------------|-----------|
| **Basic** | 10 GB | 2 TB | Limited |
| **Standard** | 100 GB | 2 TB | Medium |
| **Premium** | 500 GB | 2 TB | High |

### Unlimited Resources
✅ **Repositories** - Create as many as needed
✅ **Images** - No image count limit
✅ **Layers** - Store all image layers
✅ **Tags** - Unlimited tags per repository

⚠️ **Performance impact**: High numbers of repositories/tags can affect performance

### Storage Management
```bash
# Check registry usage
az acr show-usage --name myregistry --output table

# Example output:
# NAME                CURRENT    LIMIT
# ------------------  ---------  -------
# Size                5.2 GB     10 GB
# Webhooks            2          10
```

## Maintenance Best Practices

### Regular Cleanup
```bash
# Delete unused images (older than 30 days)
az acr run \
  --registry myregistry \
  --cmd "acr purge --filter 'myrepo:.*' --ago 30d" \
  /dev/null

# Delete untagged manifests
az acr manifest delete \
  --name myregistry \
  --repository myrepo \
  --manifest <manifest-digest>
```

### Retention Policies (Preview)
```bash
# Set retention policy
az acr config retention update \
  --registry myregistry \
  --status enabled \
  --days 30 \
  --type UntaggedManifests
```

### Lifecycle Management
```bash
# Create task to clean old images weekly
az acr task create \
  --registry myregistry \
  --name weekly-cleanup \
  --cmd "acr purge --filter 'myrepo:.*' --ago 30d" \
  --schedule "0 2 * * 0" \
  --context /dev/null
```

## Data Recovery

### Soft Delete (Preview)
```bash
# Enable soft delete (Premium)
az acr config soft-delete update \
  --registry myregistry \
  --status enabled \
  --days 7

# List deleted artifacts
az acr manifest list-deleted \
  --registry myregistry \
  --repository myrepo

# Restore deleted artifact
az acr manifest restore \
  --registry myregistry \
  --repository myrepo \
  --manifest <manifest-digest>
```

⚠️ **Important**: Deleted resources cannot be recovered without soft delete enabled

## Performance Optimization

### Network Proximity
**Use geo-replication** for global deployments:

```bash
# Replicate to regions where you deploy
az acr replication create --registry myregistry --location westus2
az acr replication create --registry myregistry --location eastasia
az acr replication create --registry myregistry --location northeurope

# Benefit: Automatic routing to nearest replica
```

### Throughput Limits

| Tier | Concurrent Operations | ReadOps/sec | WriteOps/sec |
|------|----------------------|-------------|--------------|
| **Basic** | 10 | 300 | 100 |
| **Standard** | 20 | 600 | 200 |
| **Premium** | 500 | 10,000 | 2,000 |

### Optimization Tips
✅ **Layer caching** - Use multi-stage builds
✅ **Parallel pulls** - Premium supports 500 concurrent ops
✅ **Compression** - Docker automatically compresses layers
✅ **Regional replicas** - Deploy close to compute resources

## Cost Management

### Storage Costs
```
Basic:    $0.167/day (~$5/month) + $0.10/GB/month
Standard: $0.667/day (~$20/month) + $0.10/GB/month
Premium:  $1.667/day (~$50/month) + $0.10/GB/month

Geo-replication: Additional $0.10/GB/month per replica
```

### Cost Optimization
✅ **Delete unused images** - Reduce storage costs
✅ **Use Basic for dev/test** - Lower daily cost
✅ **Optimize image size** - Use multi-stage builds
✅ **Monitor usage** - `az acr show-usage`
✅ **Retention policies** - Auto-delete old images

## Critical Notes
- 💡 **Encryption-at-rest** - Automatic for all images
- ⚠️ **Regional outage** - Data may be unavailable, not auto-recovered
- 🎯 **Geo-replication** - Premium only, multi-region high availability
- ✅ **Zone redundancy** - Premium only, minimum 3 zones per region
- 📊 **Storage limits** - 2 TB max per tier
- 🔄 **Cleanup required** - Delete unused images for performance
- ⚠️ **Deletion permanent** - Use soft delete (Preview) for recovery
- 🔒 **CMK** - Customer-managed keys for extra encryption layer

## Exam Tips
- All tiers: Encryption-at-rest (automatic), Azure-managed keys
- Regional storage: Data in registry region (+ paired region in most cases)
- Brazil South & Southeast Asia: Data confined to single region
- Geo-replication: Premium only, multi-region HA, faster pulls
- Zone redundancy: Premium only, minimum 3 zones per region
- Storage limits: Basic (10GB), Standard (100GB), Premium (500GB included)
- Max storage: 2 TB per tier
- Unlimited: Repositories, images, layers, tags (within storage limit)
- Performance impact: High repository/tag count affects performance
- Soft delete: Preview feature, Premium only, restore deleted artifacts
- Cleanup: `az acr purge`, retention policies, scheduled tasks
- Cost: Daily rate + $0.10/GB storage + geo-replication costs
- Throughput: Premium (500 concurrent ops), Standard (20), Basic (10)

[Learn More](https://learn.microsoft.com/en-us/training/modules/publish-container-image-to-azure-container-registry/3-azure-container-registry-storage)
