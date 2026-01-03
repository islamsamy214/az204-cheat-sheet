# Blob Storage Lifecycle Policies

## What is a Lifecycle Policy?

A **lifecycle management policy** is a collection of **rules in a JSON document** that automatically:
- Transition blobs to cooler access tiers based on age
- Delete blobs at the end of their lifecycle
- Apply actions based on filters (containers, prefixes, tags)

---

## Policy Structure

### Basic Policy Format

```json
{
  "rules": [
    {
      "name": "rule1",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": {...},
        "actions": {...}
      }
    },
    {
      "name": "rule2",
      "type": "Lifecycle",
      "definition": {...}
    }
  ]
}
```

### Policy Components

| Component | Description | Required |
|-----------|-------------|----------|
| **rules** | Array of rule objects | Yes (at least 1, max 100) |

---

## Rule Parameters

Each rule has the following parameters:

| Parameter | Type | Description | Required | Details |
|-----------|------|-------------|----------|---------|
| **name** | String | Rule identifier | Yes | Up to 256 alphanumeric characters, case-sensitive, must be unique |
| **enabled** | Boolean | Enable/disable rule | No | Default is `true`, allows temporary disabling |
| **type** | Enum | Rule type | Yes | Currently only `"Lifecycle"` is valid |
| **definition** | Object | Rule definition | Yes | Contains `filters` and `actions` |

### Rule Name Best Practices

✅ **DO:**
- Use descriptive names (e.g., `move-logs-to-cool`)
- Follow naming convention (kebab-case or camelCase)
- Make names unique and meaningful

❌ **DON'T:**
- Use generic names (e.g., `rule1`, `rule2`)
- Exceed 256 characters
- Use duplicate names

---

## Rule Definition

Each rule **definition** contains two sets:

### 1. Filter Set
Limits rule actions to a **specific subset of blobs**.

### 2. Action Set
Applies **tier transitions or deletions** to filtered blobs.

---

## Filter Set

Filters define **which blobs** the rule applies to.

### Available Filters

| Filter | Type | Description | Required |
|--------|------|-------------|----------|
| **blobTypes** | Array of enum | Type of blobs (e.g., `blockBlob`, `appendBlob`) | **Yes** |
| **prefixMatch** | Array of strings | Container/blob name prefixes | No |
| **blobIndexMatch** | Array of dictionary | Blob index tag key-value conditions | No |

### Filter Logic

💡 **Multiple Filters**: When multiple filters are defined, they are combined with **logical AND**.

```
(blobTypes = blockBlob) AND (prefix = logs/) AND (tag = status:archived)
```

### 1. blobTypes Filter

**Purpose**: Specify blob types to target.

**Valid Values:**
- `blockBlob` (most common)
- `appendBlob`

```json
"filters": {
  "blobTypes": ["blockBlob"]
}
```

⚠️ **Note**: Page blobs are not supported in lifecycle management policies.

### 2. prefixMatch Filter

**Purpose**: Target blobs with specific name prefixes.

**Characteristics:**
- Array of strings
- Each rule can define **up to 10 prefixes**
- Prefix string must **start with a container name**

**Examples:**

```json
// Single container
"prefixMatch": ["logs/"]

// Multiple containers
"prefixMatch": ["logs/", "backups/"]

// Specific path within container
"prefixMatch": ["container1/folder1/subfolder/"]

// Multiple specific patterns
"prefixMatch": [
  "sample-container/blob1",
  "sample-container/blob2"
]
```

#### Prefix Matching Examples

| Prefix | Matches | Doesn't Match |
|--------|---------|---------------|
| `logs/` | `logs/app.log`, `logs/2024/error.log` | `oldlogs/app.log` |
| `container1/data/` | `container1/data/file.txt` | `container1/file.txt` |
| `images/photos/` | `images/photos/pic1.jpg` | `images/pic1.jpg` |

### 3. blobIndexMatch Filter

**Purpose**: Target blobs with specific index tags.

**Characteristics:**
- Array of dictionary values (key-value pairs)
- Each rule can define **up to 10 tag conditions**

**Examples:**

```json
// Single tag
"blobIndexMatch": [
  {
    "name": "status",
    "op": "==",
    "value": "archived"
  }
]

// Multiple tags (AND logic)
"blobIndexMatch": [
  {
    "name": "status",
    "op": "==",
    "value": "archived"
  },
  {
    "name": "priority",
    "op": "==",
    "value": "low"
  }
]
```

---

## Action Set

Actions define **what happens** to filtered blobs.

### Available Actions

| Action | Supported Blob Types | Current Version | Previous Versions | Snapshots |
|--------|----------------------|-----------------|-------------------|-----------|
| **tierToCool** | Block blobs | ✅ Supported | ✅ Supported | ✅ Supported |
| **tierToCold** | Block blobs | ✅ Supported | ✅ Supported | ✅ Supported |
| **enableAutoTierToHotFromCool** | Block blobs | ✅ Supported | ❌ Not supported | ❌ Not supported |
| **tierToArchive** | Block blobs | ✅ Supported | ✅ Supported | ✅ Supported |
| **delete** | Block blobs, Append blobs | ✅ Supported | ✅ Supported | ✅ Supported |

### Action Priority

⚠️ **Multiple Actions on Same Blob**: Lifecycle management applies the **least expensive action**.

**Cost Order (cheapest to most expensive):**
```
delete < tierToArchive < tierToCold < tierToCool < no action
```

**Example:**
If a blob matches rules for both `tierToCool` and `delete`, the **delete action** is applied.

---

## Run Conditions

Actions are triggered based on **age conditions**.

### Condition Types

| Condition | Description | Used For |
|-----------|-------------|----------|
| **daysAfterModificationGreaterThan** | Days since last modification | Base blob actions |
| **daysAfterCreationGreaterThan** | Days since creation | Blob snapshot actions |
| **daysAfterLastAccessTimeGreaterThan** | Days since last access | Current version (requires access tracking) |
| **daysAfterLastTierChangeGreaterThan** | Days since last tier change | Minimum duration in tier before Archive |

### 1. daysAfterModificationGreaterThan

**Use Case**: Base blob tier transitions and deletions.

**Based On**: Blob's **last modified time**.

```json
"actions": {
  "baseBlob": {
    "tierToCool": {
      "daysAfterModificationGreaterThan": 30
    },
    "tierToArchive": {
      "daysAfterModificationGreaterThan": 90
    },
    "delete": {
      "daysAfterModificationGreaterThan": 365
    }
  }
}
```

**Timeline:**
```
Day 0:   Blob created/modified
Day 31:  Moved to Cool tier
Day 91:  Moved to Archive tier
Day 366: Deleted
```

### 2. daysAfterCreationGreaterThan

**Use Case**: Blob snapshot management.

**Based On**: Snapshot **creation time**.

```json
"actions": {
  "snapshot": {
    "tierToCool": {
      "daysAfterCreationGreaterThan": 90
    },
    "delete": {
      "daysAfterCreationGreaterThan": 365
    }
  }
}
```

### 3. daysAfterLastAccessTimeGreaterThan

**Use Case**: Transition based on actual access patterns.

**Requirements:**
- **Access tracking** must be enabled
- Applies to **current version** only

```json
"actions": {
  "baseBlob": {
    "tierToCool": {
      "daysAfterLastAccessTimeGreaterThan": 30
    }
  }
}
```

💡 **Access Tracking**: Requires enabling last access time tracking on the storage account (additional costs may apply).

### 4. daysAfterLastTierChangeGreaterThan

**Use Case**: Prevent immediate re-archival after rehydration.

**Applies To**: `tierToArchive` actions only.

**Purpose**: Ensures a blob stays in Hot/Cool/Cold tier for a minimum duration after being rehydrated from Archive.

```json
"actions": {
  "baseBlob": {
    "tierToArchive": {
      "daysAfterModificationGreaterThan": 90,
      "daysAfterLastTierChangeGreaterThan": 7
    }
  }
}
```

**Scenario:**
```
1. Blob rehydrated from Archive to Hot
2. Must stay in Hot for at least 7 days
3. Then can be moved back to Archive (if >90 days since modification)
```

---

## Complete Policy Example

### Scenario: Log Management

**Requirements:**
- Logs in `sample-container`
- Blob names start with `blob1`
- Tier to Cool after 30 days
- Tier to Archive after 90 days (but wait 7 days after any tier change)
- Delete after 7 years (2,555 days)
- Delete snapshots after 90 days

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "sample-rule",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 30
            },
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 90,
              "daysAfterLastTierChangeGreaterThan": 7
            },
            "delete": {
              "daysAfterModificationGreaterThan": 2555
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
          "prefixMatch": ["sample-container/blob1"]
        }
      }
    }
  ]
}
```

### Timeline Visualization

```
Day 0:      Blob created (Hot tier)
Day 31:     → Cool tier (after 30 days)
Day 91:     → Archive tier (after 90 days, 7+ days since last tier change)
Day 2556:   → Deleted (after 2,555 days)

Snapshots:
Day 91:     → Deleted (after 90 days from creation)
```

---

## Additional Policy Examples

### Example 1: Simple Archival Policy

Move all logs to Archive after 180 days:

```json
{
  "rules": [
    {
      "name": "archive-old-logs",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 180
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

### Example 2: Multi-Container Policy

Different retention for different containers:

```json
{
  "rules": [
    {
      "name": "archive-backups",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 30
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
      "name": "delete-temp-files",
      "enabled": true,
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
    }
  ]
}
```

### Example 3: Tag-Based Policy

Target blobs with specific tags:

```json
{
  "rules": [
    {
      "name": "archive-tagged-data",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 90
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "blobIndexMatch": [
            {
              "name": "status",
              "op": "==",
              "value": "completed"
            }
          ]
        }
      }
    }
  ]
}
```

---

## Best Practices

### Policy Design

✅ **DO:**
- Use descriptive rule names
- Start with small pilot rules
- Use prefixes for precise targeting
- Enable rules gradually
- Monitor policy execution
- Document policy intent

❌ **DON'T:**
- Create overly complex policies
- Use more than 100 rules (limit)
- Forget about minimum tier durations
- Overlap rules unnecessarily

### Filter Strategy

✅ **DO:**
- Use specific prefixes when possible
- Combine filters for precision
- Test filters before applying actions
- Use blob index tags for flexible categorization

❌ **DON'T:**
- Use overly broad filters
- Forget container name in prefixMatch
- Exceed 10 prefixes or 10 tags per rule

### Action Strategy

✅ **DO:**
- Consider minimum storage durations
- Use `daysAfterLastTierChangeGreaterThan` for Archive actions
- Plan deletion carefully (irreversible)
- Understand action precedence (cost-based)

❌ **DON'T:**
- Archive data that needs frequent access
- Delete without adequate retention period
- Ignore early deletion fees

---

## Exam Tips

🎯 **Policy structure**: JSON document with `rules` array (max 100 rules)

🎯 **Rule components**: name, enabled, type, definition (filters + actions)

🎯 **Three filter types**: blobTypes (required), prefixMatch (optional), blobIndexMatch (optional)

🎯 **Filter logic**: Multiple filters use logical AND

🎯 **Supported blob types**: blockBlob and appendBlob only (no page blobs)

🎯 **Action precedence**: Least expensive action applied (delete > tierToArchive > tierToCold > tierToCool)

🎯 **Four age conditions**: daysAfterModification (base blobs), daysAfterCreation (snapshots), daysAfterLastAccessTime (requires tracking), daysAfterLastTierChange (Archive actions)

🎯 **Prefix limit**: Up to 10 prefixes per rule

🎯 **Tag limit**: Up to 10 blob index tag conditions per rule

🎯 **Rule limit**: Maximum 100 rules per policy

🎯 **Rule name**: Up to 256 characters, case-sensitive, must be unique

🎯 **tierToArchive special condition**: Use daysAfterLastTierChangeGreaterThan to prevent immediate re-archival

---

## Quick Reference

### Rule Template

```json
{
  "name": "<rule-name>",
  "enabled": true,
  "type": "Lifecycle",
  "definition": {
    "filters": {
      "blobTypes": ["blockBlob"],
      "prefixMatch": ["<container>/<prefix>"]
    },
    "actions": {
      "baseBlob": {
        "tierToCool": {
          "daysAfterModificationGreaterThan": <days>
        },
        "tierToArchive": {
          "daysAfterModificationGreaterThan": <days>
        },
        "delete": {
          "daysAfterModificationGreaterThan": <days>
        }
      }
    }
  }
}
```

### Common Day Values

| Retention Period | Days | Tier |
|------------------|------|------|
| 1 week | 7 | Delete temp files |
| 1 month | 30 | Cool tier |
| 3 months | 90 | Cold tier or Archive |
| 6 months | 180 | Archive |
| 1 year | 365 | Delete or Archive |
| 7 years | 2,555 | Compliance retention |

---

## Additional Resources

- [Lifecycle management policy schema](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-policy-configure)
- [Optimize costs with lifecycle management](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview)

[Microsoft Learn - Discover Blob storage lifecycle policies](https://learn.microsoft.com/en-us/training/modules/manage-azure-blob-storage-lifecycle/3-blob-storage-lifecycle-policies)
