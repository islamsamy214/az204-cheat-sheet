# Azure App Service - Complete Module

## 📚 Quick Navigation

| # | Topic | Key Exam Points | Status |
|---|-------|----------------|--------|
| 1 | [Examine Azure App Service](./01-examine-azure-app-service.md) | PaaS features, Linux support, ASE | ✅ |
| 2 | [App Service Plans](./02-app-service-plans.md) | Pricing tiers, scaling strategies | ✅ |
| 3 | [Deploy to App Service](./03-deploy-to-app-service.md) | CI/CD, deployment slots, containers | ✅ |
| 4 | [Authentication & Authorization](./04-authentication-authorization.md) | Built-in auth, identity providers | ✅ |
| 5 | [Networking Features](./05-networking-features.md) | VNet integration, outbound IPs | ✅ |
| 6 | [Configure App Settings](./06-configure-app-settings.md) | Environment variables, slot settings | ✅ |
| 7 | [Configure General Settings](./07-configure-general-settings.md) | Always On, ARR Affinity, HTTPS | ✅ |
| 8 | [Diagnostic Logging](./08-diagnostic-logging.md) | Log types, streaming, storage | ✅ |

## 🎯 Exam Focus Areas

### Critical Commands to Memorize
```bash
# Create web app with deployment
az webapp up --name <app> --runtime "NODE:18-lts"

# Deployment slot operations
az webapp deployment slot create --name <app> --slot staging
az webapp deployment slot swap --name <app> --slot staging

# Configuration
az webapp config appsettings set --name <app> --settings KEY=value
az webapp config set --name <app> --always-on true

# Logging
az webapp log tail --name <app>
az webapp log download --name <app> --log-file logs.zip
```

### Tier Requirements Quick Reference

| Feature | Minimum Tier |
|---------|--------------|
| Custom Domains | Basic |
| SSL/TLS | Basic |
| Deployment Slots | Standard |
| Always On | Basic |
| VNet Integration | Standard |
| Auto Scale | Standard |
| Managed Identity | All tiers |

### Common Exam Scenarios

#### Scenario 1: Zero-Downtime Deployment
**Answer**: Use deployment slots (Standard tier+)
1. Deploy to staging slot
2. Test in staging
3. Swap to production

#### Scenario 2: Environment-Specific Configuration
**Answer**: Use slot settings
- Mark settings as "slot setting"
- Won't swap with deployment

#### Scenario 3: Keep App Always Running
**Answer**: Enable Always On (Basic tier+)
- Required for WebJobs
- Prevents 20-minute unload

#### Scenario 4: Stateless API Performance
**Answer**: Disable ARR Affinity
- Better load distribution
- No session stickiness needed

#### Scenario 5: Secure Production App
**Answer**: Multiple settings
- HTTPS Only = true
- Minimum TLS = 1.2
- Use Key Vault for secrets

## 📊 Decision Matrices

### Which Pricing Tier?

| Requirement | Tier |
|-------------|------|
| Just testing | Free/Shared |
| Basic production | Basic |
| Need deployment slots | Standard+ |
| High performance | PremiumV3 |
| Network isolation | IsolatedV2 |

### Which Deployment Method?

| Scenario | Method |
|----------|--------|
| Modern CI/CD | GitHub Actions, Azure DevOps |
| Quick test | `az webapp up` |
| Container app | Azure Container Registry + Slots |
| Legacy system | FTP/S (not recommended) |

### Which Logging?

| Purpose | Type | Storage |
|---------|------|---------|
| Temp debugging | Application (File System) | 12 hours auto-expire |
| Long-term | Application (Blob) | Permanent |
| HTTP requests | Web Server | File or Blob |
| Production errors | Application (Blob, Error level) | Blob Storage |

## 🔍 Search Patterns

### Find Commands
```bash
# Find all CLI commands
grep -r "az webapp" .

# Find specific settings
grep -r "always-on" .

# Find connection strings
grep -r "SQLCONNSTR" .
```

### Find Topics
```bash
# Find deployment info
grep -r "deployment slot" .

# Find authentication info
grep -r "/.auth/login" .

# Find networking info
grep -r "VNet" .
```

## 💡 Pro Tips Summary

1. **Always use deployment slots** for production (requires Standard+)
2. **Mark environment configs** as slot settings
3. **Disable ARR Affinity** for stateless apps
4. **Enable Always On** for WebJobs and production
5. **Use Blob Storage** for long-term logs
6. **Enforce HTTPS and TLS 1.2+** for production
7. **Use Key Vault references** instead of storing secrets
8. **Tag container images** with versions, not "latest"
9. **Outbound IPs change** when scaling between tiers
10. **Connection strings prefixed** by type (SQLCONNSTR_, etc.)

## 📝 Gotchas to Remember

- ⚠️ Free/Shared tiers are dev/test only
- ⚠️ File System logging auto-disables after 12 hours
- ⚠️ Remote debugging auto-disables after 48 hours
- ⚠️ Always On not available on Free tier
- ⚠️ Deployment slots require Standard tier or higher
- ⚠️ Linux nested JSON keys use `__` instead of `:`
- ⚠️ Connection strings mainly for .NET apps
- ⚠️ Outbound IPs change when scaling between VM families

## 🎓 Study Recommendations

### Pass 1: Skim (30 minutes)
- Read all "Key Concepts" sections
- Review all "Quick Reference" tables
- Note topics that need deeper study

### Pass 2: Deep Dive (2 hours)
- Read full summaries for weak areas
- Try commands in Azure CLI
- Review code examples

### Pass 3: Practice (1 hour)
- Create test app in Azure
- Practice deployment slots
- Configure settings and logging
- Test authentication

### Pass 4: Review (30 minutes)
- Review "Critical Notes"
- Review "Exam Tips"
- Quiz yourself on decision matrices

---

**Module Complete**: 8/8 units ✅  
**Estimated Study Time**: 4 hours total  
**Hands-On Labs**: 1-2 hours recommended  

[← Back to Main Index](../../README.md) | [Next: Azure Functions →](../02-azure-functions/)
