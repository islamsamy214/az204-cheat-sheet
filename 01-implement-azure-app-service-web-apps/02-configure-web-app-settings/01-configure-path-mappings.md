# Configure Path Mappings

## Key Concepts
- **Handler mappings** - Custom script processors for file extensions
- **Virtual applications** - Map URLs to different physical directories
- **Custom storage** - Mount Azure Storage to containerized apps

## Windows Apps (Uncontainerized)

### Handler Mappings
Add custom script processors for specific file extensions.

**Configuration**:
- **Extension**: File extension (e.g., `*.php`, `handler.fcgi`)
- **Script processor**: Absolute path (use `D:\home\site\wwwroot` for app root)
- **Arguments**: Optional command-line arguments

### Virtual Applications & Directories
- **Default root path**: `/` → `D:\home\site\wwwroot`
- Map virtual directories to physical paths relative to `D:\home`
- Clear "Directory" checkbox to mark as web application

**Example**:
| Virtual Path | Physical Path | Is Application |
|--------------|---------------|----------------|
| `/` | `site\wwwroot` | ✅ Yes |
| `/api` | `site\wwwroot\api` | ✅ Yes |
| `/images` | `site\assets\images` | ❌ No (directory) |

## Linux and Containerized Apps

### Custom Storage Mount
Mount Azure Storage (Blobs or Files) to containers.

**Configuration Options**:

| Setting | Description | Options |
|---------|-------------|---------|
| **Name** | Display name | Custom |
| **Configuration** | Basic or Advanced | Basic: Standard storage<br>Advanced: Service endpoints, private endpoints, Key Vault |
| **Storage account** | Azure Storage account | Select from subscription |
| **Storage type** | Blob or File | Blobs: Read-only<br>Files: Read/write |
| **Storage container** | Container name (Basic) | Existing container |
| **Share name** | File share (Advanced) | Existing share |
| **Access key** | Storage key (Advanced) | From storage account |
| **Mount path** | Container path | Absolute path (e.g., `/data`) |
| **Deployment slot setting** | Stick to slot | ✅/❌ |

### Storage Type Restrictions
- **Windows containers**: Only Azure Files supported
- **Linux containers**: Azure Blobs (read-only) or Azure Files
- **Azure Blobs**: Read-only access only

## Quick Reference Commands

```bash
# Configure handler mapping (via portal only)
# Configuration > Path mappings > Handler mappings

# Configure virtual application (via portal only)
# Configuration > Path mappings > Virtual applications and directories

# Add Azure Storage mount (Linux/Container)
az webapp config storage-account add \
  --resource-group <rg-name> \
  --name <app-name> \
  --custom-id <mount-name> \
  --storage-type AzureBlob \
  --account-name <storage-account> \
  --share-name <container-name> \
  --access-key <storage-key> \
  --mount-path /data

# List storage mounts
az webapp config storage-account list \
  --resource-group <rg-name> \
  --name <app-name>

# Remove storage mount
az webapp config storage-account delete \
  --resource-group <rg-name> \
  --name <app-name> \
  --custom-id <mount-name>
```

## Use Cases

| Scenario | Solution |
|----------|----------|
| Multiple apps in one deployment | Virtual applications |
| Custom PHP/Python handler | Handler mappings |
| Shared file storage across instances | Azure Files mount |
| Static assets from Blob Storage | Azure Blobs mount (read-only) |
| Separate app in subdirectory | Virtual application |

## Critical Notes
- 💡 **Windows containers** only support Azure Files (not Blobs)
- ⚠️ **Azure Blobs** are read-only when mounted
- 🎯 **Default root**: `D:\home\site\wwwroot` (Windows)
- 📝 **Virtual applications** can have separate app pools
- 🔐 **Advanced config** needed for private endpoints/service endpoints
- ⚠️ **Mount path** must be absolute (e.g., `/data`, not `data`)

## Exam Tips
- Know handler mappings are for Windows apps only
- Understand virtual applications vs directories
- Remember Azure Blobs are read-only when mounted
- Windows containers can only use Azure Files
- Custom storage useful for shared data across scaled instances
- Slot settings can apply to storage mounts

[Learn More](https://learn.microsoft.com/en-us/training/modules/configure-web-app-settings/4-configure-path-mappings)
