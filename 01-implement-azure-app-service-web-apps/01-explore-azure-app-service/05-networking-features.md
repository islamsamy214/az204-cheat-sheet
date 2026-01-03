# App Service Networking Features

## Key Concepts
- **Multitenant service** - Apps share infrastructure (except Isolated tier)
- **Inbound features** - Control traffic TO your app
- **Outbound features** - Control traffic FROM your app
- **No direct network connection** - Use networking features instead

## Deployment Types

| Type | Tiers | Network |
|------|-------|---------|
| **Multitenant** | Free, Shared, Basic, Standard, Premium, PremiumV2, PremiumV3 | Shared infrastructure |
| **Single-tenant (ASE)** | Isolated, IsolatedV2 | Your Azure VNet |

## Networking Features Overview

### Inbound Features (Control Traffic TO App)

| Feature | Purpose | Use Case |
|---------|---------|----------|
| **App-assigned address** | Dedicated IP for app | IP-based SSL, dedicated inbound address |
| **Access restrictions** | IP/VNet-based filtering | Whitelist specific IPs or subnets |
| **Service endpoints** | Secure access from VNet | Restrict to specific VNet/subnet |
| **Private endpoints** | Private IP in VNet | Fully private app access |

### Outbound Features (Control Traffic FROM App)

| Feature | Purpose | Use Case |
|---------|---------|----------|
| **Hybrid Connections** | Connect to on-premises | Access on-prem databases/APIs |
| **Gateway VNet Integration** | Legacy VNet integration | Older deployments |
| **VNet Integration** | Modern VNet integration | Access VNet resources |

## Common Inbound Use Cases

| Need | Solution |
|------|----------|
| IP-based SSL | App-assigned address |
| Dedicated inbound IP | App-assigned address |
| Restrict by IP addresses | Access restrictions |
| Restrict to VNet | Service endpoints or Private endpoints |
| Fully private app | Private endpoints |

## Outbound IP Addresses

### Understanding Outbound IPs
- **Shared by VM family** - All apps on same VM type share IPs
- **Changes with tier** - Different tiers = different VM families = different IPs
- **Multiple IPs** - Apps use multiple IPs for outbound calls
- **Listed in app properties** - Find in Azure portal or CLI

### Find Outbound IPs

```bash
# Get current outbound IPs
az webapp show \
  --resource-group <rg-name> \
  --name <app-name> \
  --query outboundIpAddresses \
  --output tsv

# Get ALL possible outbound IPs (across all tiers)
az webapp show \
  --resource-group <rg-name> \
  --name <app-name> \
  --query possibleOutboundIpAddresses \
  --output tsv
```

### When Outbound IPs Change
- ⚠️ **Scale between tiers** - Different VM families
- ⚠️ **Delete and recreate** - New resources
- ✅ **Scale within tier** - Same IPs
- ✅ **Scale out/in** - Same IPs

## Default Networking Behavior

### Free and Shared Tiers
- Apps run on **shared multitenant workers**
- Limited networking features
- Share IPs with other customers

### Basic and Higher Tiers
- Apps run on **dedicated workers**
- All apps in same plan share workers
- Deployment slots run on same workers
- Scale out = replicate to new workers

## Access Restrictions

### Configure IP Restrictions

```bash
# Add IP restriction
az webapp config access-restriction add \
  --resource-group <rg-name> \
  --name <app-name> \
  --rule-name AllowOfficeIP \
  --action Allow \
  --ip-address 203.0.113.0/24 \
  --priority 100

# Add VNet restriction
az webapp config access-restriction add \
  --resource-group <rg-name> \
  --name <app-name> \
  --rule-name AllowVNet \
  --action Allow \
  --vnet-name <vnet-name> \
  --subnet <subnet-name> \
  --priority 200
```

### Priority Rules
- **Lower number** = higher priority
- **Allow or Deny** actions
- **Evaluated in order** until match

## VNet Integration

### Regional VNet Integration
```bash
# Enable VNet integration
az webapp vnet-integration add \
  --resource-group <rg-name> \
  --name <app-name> \
  --vnet <vnet-name> \
  --subnet <subnet-name>
```

### Benefits
- ✅ Access VNet resources
- ✅ Access on-premises via ExpressRoute
- ✅ Access service endpoints
- ✅ Route outbound traffic through VNet

### Requirements
- **Standard tier or higher**
- Dedicated subnet (can't be shared with other resources)
- Subnet must have delegation to `Microsoft.Web/serverFarms`

## Private Endpoints

### Key Concepts
- Gives app a **private IP** in your VNet
- App is **not accessible** from internet (unless configured)
- Inbound traffic only through private IP

```bash
# Create private endpoint
az network private-endpoint create \
  --resource-group <rg-name> \
  --name <endpoint-name> \
  --vnet-name <vnet-name> \
  --subnet <subnet-name> \
  --private-connection-resource-id <app-resource-id> \
  --group-ids sites \
  --connection-name <connection-name>
```

### Benefits
- ✅ Secure inbound access
- ✅ No internet exposure
- ✅ Access from VNet or on-premises

## Hybrid Connections

### Key Concepts
- Connect to **on-premises** or **other networks**
- Works with **TCP endpoints** only
- Doesn't require VPN or VNet peering

```bash
# Requires Hybrid Connection Manager on-premises
# Configured through Azure Portal
```

### Use Cases
- Access on-premises SQL Server
- Connect to legacy systems
- Reach TCP endpoints in other networks

## Quick Reference Matrix

| Feature | Tier Required | Inbound/Outbound | Use Case |
|---------|---------------|------------------|----------|
| Access Restrictions | Basic+ | Inbound | IP whitelist |
| Service Endpoints | Basic+ | Inbound | VNet filtering |
| Private Endpoints | Basic+ | Inbound | Private access |
| VNet Integration | Standard+ | Outbound | Access VNet resources |
| Hybrid Connections | Basic+ | Outbound | On-premises access |

## Critical Notes
- 💡 **Outbound IPs change** when scaling between tiers (VM families)
- 🎯 **Inbound features** ≠ **Outbound features** - solve different problems
- ⚠️ **VNet Integration requires Standard+**
- 🔐 **Private Endpoints** make app inaccessible from internet
- 📊 Apps in same plan share **outbound IPs**
- 🌍 **Multitenant = no direct VNet connection** (except Isolated tier)

## Exam Tips
- Know the difference between inbound and outbound features
- Understand when outbound IPs change (tier changes)
- Remember VNet Integration requires dedicated subnet
- Private Endpoints = private IP in VNet, no internet access
- Access Restrictions can filter by IP or VNet
- Service Endpoints vs Private Endpoints distinction

[Learn More](https://learn.microsoft.com/en-us/training/modules/introduction-to-azure-app-service/6-network-features)
