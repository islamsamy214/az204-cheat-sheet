# Azure Blob Storage Lifecycle

## Data Lifecycle Patterns

Data sets have **unique lifecycles**. Understanding these patterns is key to optimizing storage costs:

| Lifecycle Stage | Access Pattern | Optimal Tier | Example |
|----------------|----------------|--------------|---------|
| **Early** | Frequently accessed | **Hot** | Recent user uploads, active logs |
| **Middle** | Infrequently accessed | **Cool/Cold** | Weekly reports, compliance data |
| **Late** | Rarely accessed | **Archive** | Long-term backups, historical records |

### Common Data Lifecycle Scenarios

```
Day 0-14:   [Hot Tier]      → Frequent access (analytics, reports)
Day 15-30:  [Cool Tier]     → Occasional access (weekly review)
Day 31-90:  [Cold Tier]     → Rare access (monthly compliance)
Day 91+:    [Archive Tier]  → Archival (long-term retention)
```

---

## Access Tiers Overview

Azure Storage provides **four access tiers** optimized for different usage patterns.

### Tier Comparison Table

| Tier | Type | Minimum Duration | Storage Cost | Access Cost | Latency | Use Case |
|------|------|------------------|--------------|-------------|---------|----------|
| **Hot** | Online | None | Highest | Lowest | Milliseconds | Frequently accessed data |
| **Cool** | Online | 30 days | Lower | Higher | Milliseconds | Infrequently accessed (>30 days) |
| **Cold** | Online | 90 days | Even Lower | Even Higher | Milliseconds | Rarely accessed (>90 days) |
| **Archive** | Offline | 180 days | Lowest | Highest | Hours | Long-term archival (>180 days) |

### Hot Tier

**Optimization**: Frequently accessed data

#### Characteristics
- **Online tier** (immediate access)
- **No minimum storage duration**
- **Highest storage costs**
- **Lowest access costs**
- Millisecond latency for read/write

#### Best For
- Data accessed or modified frequently
- Data staged for processing
- Active datasets
- Application data in active use

#### Example Scenarios
```
✅ User-uploaded images (social media)
✅ Active application logs
✅ Real-time analytics data
✅ Content delivery (CDN origin)
```

### Cool Tier

**Optimization**: Infrequently accessed data

#### Characteristics
- **Online tier** (immediate access)
- **Minimum 30 days** storage duration
- **Lower storage costs** than Hot
- **Higher access costs** than Hot
- Millisecond latency for read/write

#### Best For
- Short-term backup and disaster recovery
- Older data sets accessed infrequently
- Data held while gathering more data
- Compliance data (monthly review)

#### Example Scenarios
```
✅ 30-day backup retention
✅ Monthly reports
✅ Completed project files
✅ Media archives (occasionally accessed)
```

⚠️ **Early Deletion Fee**: Deleting or moving blobs before 30 days incurs early deletion charges.

### Cold Tier

**Optimization**: Rarely accessed data

#### Characteristics
- **Online tier** (immediate access)
- **Minimum 90 days** storage duration
- **Lower storage costs** than Cool
- **Higher access costs** than Cool
- Millisecond latency for read/write

#### Best For
- Long-term backup (3+ months)
- Quarterly compliance data
- Historical datasets
- Legal hold data

#### Example Scenarios
```
✅ Quarterly compliance data
✅ Long-term backups (90+ days)
✅ Historical records
✅ Archived project data
```

⚠️ **Early Deletion Fee**: Deleting or moving blobs before 90 days incurs early deletion charges.

### Archive Tier

**Optimization**: Long-term archival

#### Characteristics
- **Offline tier** (requires rehydration)
- **Minimum 180 days** storage duration
- **Lowest storage costs**
- **Highest access costs**
- **Hours of latency** (rehydration required)

#### Best For
- Long-term archival storage
- Regulatory compliance data (7+ years)
- Disaster recovery (rarely accessed)
- Historical data preservation

#### Example Scenarios
```
✅ 7-year tax records
✅ Legal compliance archives
✅ Disaster recovery backups
✅ Historical medical records
```

⚠️ **Rehydration Required**: Must be rehydrated to Hot/Cool/Cold tier before accessing (takes hours).

⚠️ **Early Deletion Fee**: Deleting or moving blobs before 180 days incurs early deletion charges.

---

## Data Storage Limits

💡 **Account-Level Limits**: Data storage limits are set at the **account level**, not per access tier.

You can:
- Use all your limit in one tier
- Distribute across multiple tiers
- Move data between tiers as needed

---

## Lifecycle Management

### What is Lifecycle Management?

Azure Blob Storage **lifecycle management** offers a **rule-based policy** to:
- ✅ Transition blobs to appropriate access tiers automatically
- ✅ Expire/delete data at end of lifecycle
- ✅ Optimize costs based on access patterns

### Lifecycle Management Capabilities

| Capability | Description |
|------------|-------------|
| **Tier Transitions** | Automatically move blobs to cooler tiers based on age |
| **Auto-Tier to Hot** | Move blobs from Cool to Hot when accessed (optimize performance) |
| **Delete Blobs** | Delete current versions, previous versions, or snapshots |
| **Scope Flexibility** | Apply to entire account, specific containers, or filtered blobs |
| **Filters** | Use name prefixes or blob index tags to target specific blobs |

### What You Can Do with Lifecycle Policies

```
✅ Transition Cool → Hot immediately when accessed (performance optimization)
✅ Transition current versions to cooler tiers when not accessed
✅ Transition previous versions to cooler tiers  
✅ Transition blob snapshots to cooler tiers
✅ Delete current versions at end of lifecycle
✅ Delete previous versions at end of lifecycle
✅ Delete blob snapshots at end of lifecycle
✅ Apply rules to entire account
✅ Apply rules to specific containers
✅ Apply rules to filtered blobs (prefix/tags)
```

---

## Lifecycle Management Example Scenario

### Business Requirement

A company needs to optimize storage costs for application logs:

**Access Pattern:**
- **First 2 weeks**: Logs accessed frequently for debugging → **Hot tier**
- **Week 3-4**: Logs accessed occasionally for review → **Cool tier**
- **After 1 month**: Logs rarely accessed, kept for compliance → **Archive tier**

### Lifecycle Policy Solution

```json
{
  "rules": [
    {
      "name": "log-lifecycle-policy",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["logs/"]
        },
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 14
            },
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 30
            }
          }
        }
      }
    }
  ]
}
```

### Cost Optimization Result

```
Days 0-14:   Hot tier (frequent access, high storage cost)
Days 15-30:  Cool tier (occasional access, lower storage cost)
Days 31+:    Archive tier (rare access, lowest storage cost)

💰 Result: Significant cost savings while maintaining compliance
```

---

## Cost Optimization Strategy

### The Cost Tradeoff

| As Tier Gets Colder | Storage Cost | Access Cost | Latency |
|----------------------|--------------|-------------|---------|
| Hot → Cool | ↓ Decreases | ↑ Increases | Same |
| Cool → Cold | ↓ Decreases | ↑ Increases | Same |
| Cold → Archive | ↓ Decreases | ↑ Increases | ↑ Hours |

### Decision Framework

```
High Access Frequency → Hot Tier
│
├─ Moderate Access (>30 days) → Cool Tier
│  
├─ Low Access (>90 days) → Cold Tier
│
└─ Rare Access (>180 days) → Archive Tier
```

### Best Practices

✅ **DO:**
- Use lifecycle policies to automate tier transitions
- Choose tiers based on actual access patterns
- Monitor access patterns and adjust policies
- Account for minimum storage durations
- Use Archive tier for long-term retention (7+ years)
- Enable auto-tier to Hot for performance-critical data

❌ **DON'T:**
- Delete or move data before minimum duration (incurs fees)
- Use Archive tier for data needing immediate access
- Ignore access patterns when choosing tiers
- Set unrealistic tier transition timelines

---

## Key Concepts

### Online vs. Offline Tiers

| Category | Tiers | Access | Latency |
|----------|-------|--------|---------|
| **Online** | Hot, Cool, Cold | Immediate | Milliseconds |
| **Offline** | Archive | Requires rehydration | Hours |

### Minimum Storage Durations

⚠️ **Critical for Cost Optimization**:

| Tier | Minimum Duration | Early Deletion Impact |
|------|------------------|----------------------|
| Hot | None | No penalty |
| Cool | 30 days | Early deletion fee charged |
| Cold | 90 days | Early deletion fee charged |
| Archive | 180 days | Early deletion fee charged |

💡 **Best Practice**: Plan data retention to match minimum durations to avoid early deletion fees.

### Rule-Based Policies

Lifecycle management uses **rules** to automate:
- Tier transitions based on **age**
- Deletion based on **age**
- Filtering by **container**, **prefix**, or **tags**

---

## Exam Tips

🎯 **Four access tiers**: Hot, Cool, Cold, Archive (know the differences)

🎯 **Minimum durations**: Cool = 30 days, Cold = 90 days, Archive = 180 days

🎯 **Online vs Offline**: Hot/Cool/Cold are online (immediate access), Archive is offline (rehydration required)

🎯 **Archive latency**: Hours (not milliseconds like other tiers)

🎯 **Lifecycle management**: Rule-based policy for automatic tier transitions and deletions

🎯 **Cost tradeoff**: Storage cost ↓ as tier cools, Access cost ↑ as tier cools

🎯 **Early deletion fees**: Moving/deleting before minimum duration incurs charges

🎯 **Account-level limits**: Storage limits are at account level, not per tier

🎯 **Auto-tier to Hot**: Transition Cool → Hot when accessed (performance optimization)

🎯 **Filters**: Rules can target entire account, containers, or specific blobs (prefix/tags)

---

## Quick Reference Commands

### Set Blob Tier Manually

```bash
# Azure CLI
az storage blob set-tier \
  --account-name <account-name> \
  --container-name <container-name> \
  --name <blob-name> \
  --tier <Hot|Cool|Cold|Archive> \
  --auth-mode login

# PowerShell
Set-AzStorageBlobTier `
  -Container <container-name> `
  -Blob <blob-name> `
  -StandardBlobTier <Hot|Cool|Cold|Archive> `
  -Context $ctx
```

### Check Blob Tier

```bash
# Azure CLI
az storage blob show \
  --account-name <account-name> \
  --container-name <container-name> \
  --name <blob-name> \
  --query "properties.blobTier" \
  --auth-mode login

# PowerShell
(Get-AzStorageBlob `
  -Container <container-name> `
  -Blob <blob-name> `
  -Context $ctx).ICloudBlob.Properties.StandardBlobTier
```

---

## Tier Selection Decision Tree

```
What's the data access pattern?

├─ Accessed frequently (multiple times per day/week)
│  └─ Hot Tier ✅
│
├─ Accessed occasionally (monthly, >30 days old)
│  └─ Cool Tier ✅
│
├─ Accessed rarely (quarterly, >90 days old)
│  └─ Cold Tier ✅
│
└─ Archived (compliance, >180 days, hours to access OK)
   └─ Archive Tier ✅
```

---

## Additional Resources

- [Access tiers for blob data](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview)
- [Blob storage lifecycle management](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview)

[Microsoft Learn - Explore the Azure Blob storage lifecycle](https://learn.microsoft.com/en-us/training/modules/manage-azure-blob-storage-lifecycle/2-blob-storage-lifecycle)
