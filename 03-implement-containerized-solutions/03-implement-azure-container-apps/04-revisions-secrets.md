# Manage Revisions and Secrets in Azure Container Apps

## Key Concepts
- **Revisions** - Immutable snapshots of container app versions
- **Revision-scope changes** - Trigger new revision creation
- **Traffic splitting** - Route traffic across multiple revisions
- **Secrets** - Secure storage for sensitive configuration

## Revisions

### What Are Revisions?

**Immutable snapshots** of container app versions:

- Created automatically on revision-scope changes
- Cannot be modified after creation
- Enable versioning and rollback
- Support traffic splitting
- Customizable revision names

### Revision Lifecycle
```
Create App → Deploy (Revision 1)
           ↓
Update Configuration → Revision 2 (new)
           ↓
Traffic Split: 80% Rev 1, 20% Rev 2
           ↓
Full Rollout → 100% Rev 2
           ↓
Deactivate Rev 1
```

### Revision-Scope Changes

**Changes that trigger new revision**:

✅ Container image changes
✅ Environment variables  
✅ CPU/memory allocation
✅ Scale rules
✅ Dapr configuration
✅ Volume mounts

**Changes that DON'T trigger revision**:
❌ Secrets (application-scoped)
❌ Ingress settings (application-scoped)
❌ Revision mode (application-scoped)

## Revision Naming

### Default Naming
```
<app-name>--<random-suffix>

Example:
myapp--7d7n3xz (automatically generated)
```

### Custom Naming
```bash
# Set custom revision suffix
az containerapp update \
  --name myapp \
  --resource-group myResourceGroup \
  --revision-suffix v2-hotfix \
  --image myapp:v2.1

# Results in: myapp--v2-hotfix
```

### Naming Best Practices
✅ **Descriptive** - Include version or feature name
✅ **Sequential** - v1, v2, v3 or use dates
✅ **Lowercase** - Alphanumeric and hyphens only
✅ **Short** - Keep it concise

Examples:
- `v1-0-0`
- `2026-01-03`
- `feature-payment`
- `hotfix-auth`

## Managing Revisions

### List Revisions
```bash
# List all revisions
az containerapp revision list \
  --name myapp \
  --resource-group myResourceGroup \
  --output table

# Output:
# NAME                      ACTIVE  TRAFFIC  CREATED
# myapp--v1-0-0            True    80%      2026-01-01
# myapp--v2-0-0            True    20%      2026-01-03
# myapp--v1-5-0            False   0%       2026-01-02
```

### Show Revision Details
```bash
# Get specific revision info
az containerapp revision show \
  --name myapp \
  --resource-group myResourceGroup \
  --revision myapp--v2-0-0
```

### Activate Revision
```bash
# Activate inactive revision
az containerapp revision activate \
  --name myapp \
  --resource-group myResourceGroup \
  --revision myapp--v1-0-0
```

### Deactivate Revision
```bash
# Deactivate revision (stop routing traffic)
az containerapp revision deactivate \
  --name myapp \
  --resource-group myResourceGroup \
  --revision myapp--v1-0-0
```

### Copy Revision
```bash
# Create new revision from existing
az containerapp revision copy \
  --name myapp \
  --resource-group myResourceGroup \
  --from-revision myapp--v1-0-0 \
  --revision-suffix v1-0-1 \
  --image myapp:v1.0.1
```

## Traffic Splitting

### Traffic Management Modes

#### Single Revision Mode
**Default** - Only latest revision receives traffic:

```bash
# Single revision mode (default)
az containerapp update \
  --name myapp \
  --resource-group myResourceGroup \
  --image myapp:v2.0

# Old revision deactivated automatically
# 100% traffic to new revision
```

#### Multiple Revisions Mode
**Manual control** - Distribute traffic across revisions:

```bash
# Enable multiple revisions mode
az containerapp revision set-mode \
  --name myapp \
  --resource-group myResourceGroup \
  --mode multiple
```

### Split Traffic

#### Blue/Green Deployment
```bash
# 100% to blue (v1), 0% to green (v2)
az containerapp ingress traffic set \
  --name myapp \
  --resource-group myResourceGroup \
  --revision-weight myapp--v1=100 myapp--v2=0

# Gradually shift to green
az containerapp ingress traffic set \
  --name myapp \
  --resource-group myResourceGroup \
  --revision-weight myapp--v1=50 myapp--v2=50

# Complete rollout to green
az containerapp ingress traffic set \
  --name myapp \
  --resource-group myResourceGroup \
  --revision-weight myapp--v1=0 myapp--v2=100
```

#### Canary Release
```bash
# 95% stable, 5% canary
az containerapp ingress traffic set \
  --name myapp \
  --resource-group myResourceGroup \
  --revision-weight myapp--stable=95 myapp--canary=5

# Increase canary traffic
az containerapp ingress traffic set \
  --name myapp \
  --resource-group myResourceGroup \
  --revision-weight myapp--stable=80 myapp--canary=20
```

#### A/B Testing
```bash
# 50% version A, 50% version B
az containerapp ingress traffic set \
  --name myapp \
  --resource-group myResourceGroup \
  --revision-weight myapp--version-a=50 myapp--version-b=50
```

### Traffic Distribution Example
```
User Requests (100%)
├── 70% → Revision v2.0 (stable)
├── 20% → Revision v2.1 (canary)
└── 10% → Revision v1.9 (fallback)
```

## Secrets Management

### What Are Secrets?

**Secure storage** for sensitive configuration:

- Scoped to application (not revision)
- Encrypted at rest
- Referenced in environment variables or scale rules
- Changes don't trigger new revisions

### Secret Characteristics
✅ **Application-scoped** - Shared across all revisions
✅ **Encrypted** - Secure storage
✅ **No auto-update** - Must restart/redeploy to use updated secrets
✅ **Multiple references** - Multiple revisions can use same secret

### Define Secrets

#### At Creation
```bash
# Create app with secrets
az containerapp create \
  --name myapp \
  --resource-group myResourceGroup \
  --environment myenvironment \
  --image myapp:latest \
  --secrets "queue-connection=$QUEUE_CONNECTION" "api-key=$API_KEY"
```

#### Add Secrets
```bash
# Add secret to existing app
az containerapp secret set \
  --name myapp \
  --resource-group myResourceGroup \
  --secrets "db-password=$DB_PASSWORD"
```

### Reference Secrets

#### In Environment Variables
```bash
# Reference secret in environment variable
az containerapp create \
  --name myapp \
  --resource-group myResourceGroup \
  --environment myenvironment \
  --image myapp:latest \
  --secrets "queue-connection=$CONNECTION_STRING" \
  --env-vars \
    "QueueName=myqueue" \
    "ConnectionString=secretref:queue-connection"

# secretref:<secret-name> references the secret
```

#### In ARM Template
```json
{
  "properties": {
    "configuration": {
      "secrets": [
        {
          "name": "queue-connection",
          "value": "DefaultEndpointsProtocol=https;..."
        }
      ]
    },
    "template": {
      "containers": [
        {
          "env": [
            {
              "name": "ConnectionString",
              "secretRef": "queue-connection"
            }
          ]
        }
      ]
    }
  }
}
```

### List Secrets
```bash
# List secret names (not values)
az containerapp secret list \
  --name myapp \
  --resource-group myResourceGroup \
  --output table

# Output shows names only, not values
# NAME               
# queue-connection
# api-key
# db-password
```

### Update Secrets
```bash
# Update secret value
az containerapp secret set \
  --name myapp \
  --resource-group myResourceGroup \
  --secrets "api-key=$NEW_API_KEY"
```

### Remove Secrets
```bash
# Remove secret
az containerapp secret remove \
  --name myapp \
  --resource-group myResourceGroup \
  --secret-names "old-secret"
```

## Secret Updates and Revisions

### Secret Change Impact

**Secrets are application-scoped**, not revision-scoped:

```
Update Secret → No new revision created
              ↓
Existing revisions don't see change
              ↓
Must take action:
  1. Deploy new revision, OR
  2. Restart existing revision
```

### Responding to Secret Changes

#### Option 1: Deploy New Revision
```bash
# Trigger new revision
az containerapp update \
  --name myapp \
  --resource-group myResourceGroup \
  --image myapp:latest
```

#### Option 2: Restart Revision
```bash
# Restart existing revision
az containerapp revision restart \
  --name myapp \
  --resource-group myResourceGroup \
  --revision myapp--v1-0-0
```

### Before Deleting Secrets

1. **Deploy new revision** without secret reference
2. **Deactivate old revisions** that reference the secret
3. **Delete secret** safely

```bash
# Step 1: Remove secret reference
az containerapp update \
  --name myapp \
  --resource-group myResourceGroup \
  --replace-env-vars "ConnectionString=new-value"

# Step 2: Deactivate old revisions
az containerapp revision deactivate \
  --name myapp \
  --resource-group myResourceGroup \
  --revision myapp--old-version

# Step 3: Delete secret
az containerapp secret remove \
  --name myapp \
  --resource-group myResourceGroup \
  --secret-names "old-connection"
```

## Key Vault Integration

⚠️ **No native Key Vault integration** for secrets:

**Workaround**: Use managed identity + Key Vault SDK in app

```csharp
// App code retrieves secrets from Key Vault
var client = new SecretClient(
    new Uri("https://myvault.vault.azure.net/"),
    new DefaultAzureCredential());

var secret = await client.GetSecretAsync("db-password");
var connectionString = secret.Value.Value;
```

💡 **Recommendation**: Use Key Vault for sensitive secrets, Container Apps secrets for non-critical config

## Best Practices

### 1. Use Descriptive Revision Names
```bash
--revision-suffix v2-0-1-hotfix
```

### 2. Test with Traffic Splitting
```bash
# Start with small percentage
--revision-weight new=5 stable=95
```

### 3. Keep Active Revisions Minimal
```bash
# Deactivate unused revisions
az containerapp revision deactivate ...
```

### 4. Store Secrets Securely
```bash
# Use environment variables or CI/CD secrets
az containerapp secret set --secrets "key=$SECRET_FROM_ENV"
```

### 5. Update Secrets Safely
```bash
# 1. Update secret
# 2. Deploy new revision OR restart existing
# 3. Test
# 4. Deactivate old revisions
```

## Critical Notes
- 💡 **Revisions** - Immutable snapshots created on certain changes
- ⚠️ **Revision-scope** - Image, env vars, resources trigger new revision
- 🎯 **Traffic splitting** - Blue/Green, canary, A/B testing
- ✅ **Secrets** - Application-scoped, not revision-scoped
- 📊 **Secret changes** - Don't create revisions, must restart/redeploy
- 🔄 **Multiple modes** - Single (auto) or multiple (manual) revision mode
- 🔒 **Key Vault** - Not natively integrated, use SDK with managed identity
- ⚠️ **Naming** - Custom revision suffixes for better organization

## Exam Tips
- Revisions: Immutable snapshots of container app version
- Revision-scope changes: Image, env vars, CPU/memory, scale rules, Dapr
- Non-revision changes: Secrets, ingress, revision mode
- Revision naming: `<app>--<suffix>`, customize with --revision-suffix
- Traffic splitting: Distribute across multiple active revisions
- Revision modes: Single (latest only), Multiple (manual control)
- Blue/Green: 100% old → gradually shift → 100% new
- Canary: Small % to new version, monitor, increase gradually
- Secrets: Application-scoped, shared across revisions
- Secret reference: `secretref:<secret-name>` in environment variables
- Secret updates: Don't trigger revisions, must restart/redeploy
- Before deleting secret: Deploy revision without it, deactivate old revisions
- Key Vault: Use managed identity + SDK (not natively integrated)
- List secrets: Shows names only, not values
- Restart revision: `az containerapp revision restart`

[Learn More](https://learn.microsoft.com/en-us/training/modules/implement-azure-container-apps/6-container-apps-revisions-secrets)
