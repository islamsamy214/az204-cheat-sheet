# Implement Blob Storage Lifecycle Policies

## Implementation Methods

You can add, edit, or remove lifecycle management policies using:

| Method | Use Case | Complexity |
|--------|----------|------------|
| **Azure Portal** | Visual interface, quick setup | Low |
| **Azure PowerShell** | Script automation, bulk operations | Medium |
| **Azure CLI** | Command-line automation, CI/CD | Medium |
| **REST APIs** | Programmatic integration, custom tools | High |

---

## Azure Portal Implementation

### Two Approaches in Portal

1. **List View**: Visual, wizard-style interface
2. **Code View**: Direct JSON editing (more powerful)

### Code View (Recommended)

**Steps:**

1. Navigate to your storage account in Azure Portal
2. Under **Data management** section, select **Lifecycle Management**
3. Select the **Code View** tab
4. Define or edit the lifecycle management policy in JSON
5. Click **Save**

### Example: Move Logs to Cool Tier

**Requirement**: Move block blobs starting with `log` to Cool tier after 30 days

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "move-to-cool",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 30
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["sample-container/log"]
        }
      }
    }
  ]
}
```

### Portal Advantages

✅ Visual feedback
✅ Syntax validation
✅ Easy for single policies
✅ No scripting required

### Portal Limitations

❌ Not ideal for automation
❌ Manual process for multiple accounts
❌ No version control integration

---

## Azure CLI Implementation

### Prerequisites

```bash
# Ensure Azure CLI is installed and logged in
az login

# Verify subscription
az account show
```

### Implementation Steps

#### 1. Create Policy JSON File

Save your policy as a JSON file (e.g., `policy.json`):

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "move-to-cool",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 30
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["sample-container/log"]
        }
      }
    }
  ]
}
```

#### 2. Create/Update Policy

```bash
# Create or update lifecycle management policy
az storage account management-policy create \
    --account-name <storage-account> \
    --policy @policy.json \
    --resource-group <resource-group>
```

⚠️ **Important**: A lifecycle management policy must be **read or written in full**. Partial updates are NOT supported.

#### 3. View Existing Policy

```bash
# Get current lifecycle policy
az storage account management-policy show \
    --account-name <storage-account> \
    --resource-group <resource-group>
```

#### 4. Delete Policy

```bash
# Delete lifecycle policy
az storage account management-policy delete \
    --account-name <storage-account> \
    --resource-group <resource-group>
```

### CLI Advantages

✅ Scriptable and automatable
✅ CI/CD pipeline integration
✅ Version control friendly
✅ Batch operations across accounts

---

## Azure PowerShell Implementation

### Prerequisites

```powershell
# Install Azure PowerShell module (if not installed)
Install-Module -Name Az -AllowClobber -Scope CurrentUser

# Connect to Azure
Connect-AzAccount

# Set subscription context
Set-AzContext -SubscriptionId <subscription-id>
```

### Implementation Steps

#### 1. Create Policy JSON File

Save policy as `policy.json` (same format as CLI).

#### 2. Create/Update Policy

```powershell
# Set variables
$resourceGroup = "<resource-group>"
$storageAccount = "<storage-account>"
$policyFile = "policy.json"

# Read policy from file
$policy = Get-Content -Path $policyFile -Raw

# Create or update lifecycle policy
Set-AzStorageAccountManagementPolicy `
    -ResourceGroupName $resourceGroup `
    -StorageAccountName $storageAccount `
    -Policy $policy
```

#### 3. View Existing Policy

```powershell
# Get current lifecycle policy
Get-AzStorageAccountManagementPolicy `
    -ResourceGroupName $resourceGroup `
    -StorageAccountName $storageAccount
```

#### 4. Delete Policy

```powershell
# Remove lifecycle policy
Remove-AzStorageAccountManagementPolicy `
    -ResourceGroupName $resourceGroup `
    -StorageAccountName $storageAccount
```

### PowerShell Advantages

✅ Integration with existing PowerShell scripts
✅ Windows automation
✅ Azure automation runbooks
✅ Rich object manipulation

---

## Complete Policy Examples

### Example 1: Comprehensive Log Management

**Requirement:**
- Logs in `logs/` container
- Move to Cool after 30 days
- Move to Archive after 90 days
- Delete after 2 years (730 days)
- Delete snapshots after 90 days

**policy.json:**
```json
{
  "rules": [
    {
      "enabled": true,
      "name": "log-management",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 30
            },
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 90
            },
            "delete": {
              "daysAfterModificationGreaterThan": 730
            }
          },
          "snapshot": {
            "delete": {
              "daysAfterCreationGreaterThan": 90
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["logs/"]
        }
      }
    }
  ]
}
```

**Deploy:**
```bash
az storage account management-policy create \
    --account-name mylogstore \
    --policy @policy.json \
    --resource-group myResourceGroup
```

### Example 2: Multi-Container Policy

**Requirement:**
- Backups: Archive after 30 days, delete after 7 years
- Temp files: Delete after 7 days
- Reports: Move to Cool after 60 days

**policy.json:**
```json
{
  "rules": [
    {
      "enabled": true,
      "name": "backup-retention",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 30
            },
            "delete": {
              "daysAfterModificationGreaterThan": 2555
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["backups/"]
        }
      }
    },
    {
      "enabled": true,
      "name": "delete-temp-files",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "delete": {
              "daysAfterModificationGreaterThan": 7
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["temp/"]
        }
      }
    },
    {
      "enabled": true,
      "name": "cool-old-reports",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 60
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["reports/"]
        }
      }
    }
  ]
}
```

### Example 3: Auto-Tier to Hot from Cool

**Requirement:** Automatically move frequently accessed blobs from Cool to Hot

**policy.json:**
```json
{
  "rules": [
    {
      "enabled": true,
      "name": "auto-hot-on-access",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "enableAutoTierToHotFromCool": {
              "daysAfterLastAccessTimeGreaterThan": 0
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"]
        }
      }
    }
  ]
}
```

⚠️ **Note**: Requires **last access time tracking** to be enabled on the storage account.

---

## Workflow for Implementing Policies

### Development Workflow

```
1. Analyze data access patterns
   ↓
2. Design policy rules
   ↓
3. Create policy.json file
   ↓
4. Validate JSON syntax
   ↓
5. Test on non-production account
   ↓
6. Apply to production
   ↓
7. Monitor policy execution
   ↓
8. Adjust as needed
```

### Testing Strategy

✅ **DO:**
1. Start with `"enabled": false` for new rules
2. Test on a clone or dev storage account
3. Use short day values for testing (e.g., 1-2 days)
4. Monitor Azure Monitor metrics
5. Enable rules gradually in production

❌ **DON'T:**
- Deploy directly to production without testing
- Use aggressive deletion rules initially
- Forget to monitor policy effects

---

## Monitoring and Validation

### Check Policy Execution

```bash
# Azure CLI - Check policy
az storage account management-policy show \
    --account-name <storage-account> \
    --resource-group <resource-group> \
    --output json
```

```powershell
# PowerShell - Check policy
Get-AzStorageAccountManagementPolicy `
    -ResourceGroupName <resource-group> `
    -StorageAccountName <storage-account> | ConvertTo-Json
```

### Monitor Blob Tiers

```bash
# Azure CLI - Check blob tier
az storage blob show \
    --account-name <storage-account> \
    --container-name <container> \
    --name <blob-name> \
    --query "properties.blobTier" \
    --auth-mode login
```

```powershell
# PowerShell - Check blob tier
(Get-AzStorageBlob `
    -Container <container> `
    -Blob <blob-name> `
    -Context $ctx).ICloudBlob.Properties.StandardBlobTier
```

### Azure Monitor Integration

Enable diagnostic settings to track:
- Tier transition operations
- Deletion operations
- Policy execution logs

---

## Common Scenarios

### Scenario 1: Cost Optimization for Logs

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "optimize-logs",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {"daysAfterModificationGreaterThan": 14},
            "tierToArchive": {"daysAfterModificationGreaterThan": 30}
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["logs/"]
        }
      }
    }
  ]
}
```

### Scenario 2: Compliance Retention

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "compliance-7-years",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToArchive": {"daysAfterModificationGreaterThan": 90},
            "delete": {"daysAfterModificationGreaterThan": 2555}
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["compliance/"]
        }
      }
    }
  ]
}
```

### Scenario 3: Temporary File Cleanup

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "cleanup-temp",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "delete": {"daysAfterModificationGreaterThan": 7}
          }
        },
        "filters": {
          "blobTypes": ["blockBlob", "appendBlob"],
          "prefixMatch": ["temp/", "cache/"]
        }
      }
    }
  ]
}
```

---

## Best Practices

### JSON Policy Management

✅ **DO:**
- Store policy files in version control (Git)
- Use meaningful file names (`logs-policy.json`, `compliance-policy.json`)
- Add comments in separate documentation
- Validate JSON syntax before deployment
- Keep backup of previous policies

❌ **DON'T:**
- Edit policies directly in Portal for production
- Lose track of policy versions
- Deploy without validation

### Deployment Strategy

✅ **DO:**
- Deploy to dev/test environment first
- Use Infrastructure as Code (Terraform, ARM templates)
- Automate policy deployment in CI/CD pipelines
- Document policy intent and rationale
- Set up alerts for policy execution failures

❌ **DON'T:**
- Deploy manually to multiple accounts
- Skip testing phase
- Forget to document changes

### Maintenance

✅ **DO:**
- Review policies quarterly
- Monitor storage costs and access patterns
- Adjust policies based on actual usage
- Enable/disable rules as needed
- Keep policies simple and focused

❌ **DON'T:**
- Set and forget policies
- Create overly complex rules
- Ignore policy execution logs

---

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Policy not applying | Rule disabled | Set `"enabled": true` |
| Syntax error | Invalid JSON | Validate JSON syntax |
| Blobs not transitioning | Incorrect prefix | Check `prefixMatch` values |
| Permission denied | Insufficient RBAC | Assign Storage Account Contributor role |
| Policy conflicts | Multiple overlapping rules | Review and consolidate rules |

### Validation Checklist

- ✅ JSON syntax is valid
- ✅ Rule names are unique
- ✅ `blobTypes` filter is specified
- ✅ Day values are appropriate
- ✅ Prefixes include container names
- ✅ Enabled flag is set correctly
- ✅ Actions make business sense
- ✅ No more than 100 rules
- ✅ No more than 10 prefixes per rule

---

## Exam Tips

🎯 **Implementation methods**: Portal (Code View), Azure CLI, PowerShell, REST APIs

🎯 **CLI command**: `az storage account management-policy create --policy @policy.json`

🎯 **PowerShell cmdlet**: `Set-AzStorageAccountManagementPolicy -Policy $policy`

🎯 **Full replacement**: Policies must be read/written in **full** (no partial updates)

🎯 **Portal location**: Storage account → Data management → Lifecycle Management

🎯 **Code View vs List View**: Code View provides more control, direct JSON editing

🎯 **Policy file format**: JSON with `rules` array

🎯 **Testing**: Always test on non-production before production deployment

🎯 **Monitoring**: Use Azure Monitor for tracking policy execution

🎯 **Version control**: Store policy JSON files in Git for tracking changes

---

## Quick Reference Commands

### Azure CLI

```bash
# Create/update policy
az storage account management-policy create \
    --account-name <account> \
    --policy @policy.json \
    --resource-group <rg>

# View policy
az storage account management-policy show \
    --account-name <account> \
    --resource-group <rg>

# Delete policy
az storage account management-policy delete \
    --account-name <account> \
    --resource-group <rg>
```

### Azure PowerShell

```powershell
# Create/update policy
$policy = Get-Content -Path policy.json -Raw
Set-AzStorageAccountManagementPolicy `
    -ResourceGroupName <rg> `
    -StorageAccountName <account> `
    -Policy $policy

# View policy
Get-AzStorageAccountManagementPolicy `
    -ResourceGroupName <rg> `
    -StorageAccountName <account>

# Delete policy
Remove-AzStorageAccountManagementPolicy `
    -ResourceGroupName <rg> `
    -StorageAccountName <account>
```

---

## Additional Resources

- [Configure lifecycle management policy](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-policy-configure)
- [Azure CLI storage account management-policy](https://learn.microsoft.com/en-us/cli/azure/storage/account/management-policy)
- [PowerShell Storage Management Policy cmdlets](https://learn.microsoft.com/en-us/powershell/module/az.storage/)

[Microsoft Learn - Implement Blob storage lifecycle policies](https://learn.microsoft.com/en-us/training/modules/manage-azure-blob-storage-lifecycle/4-add-policy-blob-storage)
