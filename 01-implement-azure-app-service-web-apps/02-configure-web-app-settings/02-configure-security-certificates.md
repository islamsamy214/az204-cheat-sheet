# Configure Security Certificates

## Key Concepts
- **Managed certificates** - Free, auto-renewed by Azure
- **App Service certificates** - Purchased, managed by Azure, stored in Key Vault
- **Private certificates** - Upload your own (PFX format)
- **Public certificates** - For accessing remote resources

## Certificate Options

| Option | Cost | Management | Use Case |
|--------|------|------------|----------|
| **Free managed certificate** | Free | Fully automated | Secure custom domain |
| **App Service certificate** | Paid | Azure-managed, Key Vault | Production with flexibility |
| **Import from Key Vault** | Varies | Self-managed | Existing Key Vault setup |
| **Upload private** | Varies | Self-managed | Third-party certificates |
| **Upload public** | Free | Self-managed | Access remote resources |

## Private Certificate Requirements

### Basic Requirements
- ✅ **Password-protected PFX** file
- ✅ **Triple DES** encryption
- ✅ **Private key**: Minimum 2,048 bits
- ✅ **Certificate chain**: Include intermediate & root certificates

### TLS Binding Requirements (Custom Domain)
- ✅ **Extended Key Usage**: Server authentication (OID = 1.3.6.1.5.5.7.3.1)
- ✅ **Signed by trusted CA**

## Free Managed Certificate

### Requirements
- **Tier**: Basic, Standard, Premium, or Isolated (not Free/Shared)
- **CAA Record**: May need `0 issue digicert.com` for some domains
- **Issuer**: DigiCert

### Features
- ✅ **Auto-renewal**: Every 6 months (45 days before expiration)
- ✅ **Fully managed**: Azure handles everything
- ✅ **TLS/SSL**: Server certificate

### Limitations
- ❌ **No wildcard** certificates
- ❌ **No client certificate** usage (thumbprint-based)
- ❌ **Not exportable**
- ❌ **Not supported in ASE**
- ❌ **No private DNS**
- ❌ **Only alphanumeric, dash, period** characters
- ❌ **Max 64 characters** for custom domain

```bash
# Free managed certificate is created via portal
# Configuration > Custom domains > Add binding
# Select "App Service Managed Certificate"
```

## App Service Certificate

### Azure Manages
- ✅ Purchase process from provider
- ✅ Domain verification
- ✅ Key Vault storage
- ✅ Auto-renewal
- ✅ Sync with App Service apps

### Operations

```bash
# Create App Service Certificate
az appservice certificate create \
  --resource-group <rg-name> \
  --name <cert-name> \
  --hostname <domain-name> \
  --key-vault <vault-name>

# Import to app
az webapp config ssl bind \
  --resource-group <rg-name> \
  --name <app-name> \
  --certificate-thumbprint <thumbprint> \
  --ssl-type SNI

# List certificates
az webapp config ssl list \
  --resource-group <rg-name>
```

### Benefits
- ✅ Automated management
- ✅ Renewal flexibility
- ✅ Export options
- ✅ Key Vault integration

⚠️ **Note**: Not supported in Azure National Clouds

## Upload Private Certificate

```bash
# Upload private certificate
az webapp config ssl upload \
  --resource-group <rg-name> \
  --name <app-name> \
  --certificate-file <path-to-pfx> \
  --certificate-password <password>

# Bind certificate to custom domain
az webapp config ssl bind \
  --resource-group <rg-name> \
  --name <app-name> \
  --certificate-thumbprint <thumbprint> \
  --ssl-type SNI
```

### SSL Types
| Type | Description | Use Case |
|------|-------------|----------|
| **SNI SSL** | Server Name Indication | Modern browsers, multiple domains |
| **IP-based SSL** | Dedicated IP | Legacy browsers, single domain |

## Import from Key Vault

```bash
# Import certificate from Key Vault
az webapp config ssl import \
  --resource-group <rg-name> \
  --name <app-name> \
  --key-vault <vault-name> \
  --key-vault-certificate-name <cert-name>
```

**Requirements**:
- App Service managed identity with Key Vault access
- Certificate stored in Key Vault

## Upload Public Certificate

```bash
# Upload public certificate (via portal)
# Configuration > Certificates > Public Key Certificates (.cer)

# Access in code
var cert = X509Certificate2Collection.Find(
    X509FindType.FindByThumbprint,
    "<thumbprint>",
    validOnly: false
);
```

**Use Cases**:
- Calling external APIs with certificate authentication
- Mutual TLS with external services
- Not for securing custom domains

## Certificate Storage

**Location**: Deployment unit (webspace)
- Bound to resource group + region combination
- Accessible to apps in same RG + region

## Quick Reference

### Tier Requirements
| Feature | Minimum Tier |
|---------|--------------|
| Free managed cert | Basic |
| App Service cert | Basic |
| Custom domain SSL | Basic |
| SNI SSL | All tiers (Basic+) |
| IP-based SSL | Standard+ |

### Certificate Formats
- **PFX**: Private certificates (password-protected)
- **CER**: Public certificates
- **Triple DES**: Encryption for PFX

## Critical Notes
- 💡 **Free certs** auto-renew 45 days before expiration
- ⚠️ **Free certs** have limitations (no wildcard, not exportable)
- 🎯 **App Service certs** stored in Key Vault automatically
- 🔐 **Private key** minimum 2,048 bits
- 📝 **SNI SSL** recommended over IP-based SSL
- ⚠️ **Free tier** doesn't support custom domains/certificates
- 🌍 **National Clouds** don't support App Service Certificates

## Exam Tips
- Know free managed certificate limitations (no wildcard, no export)
- Understand certificate options and when to use each
- Remember minimum tier for SSL (Basic)
- Know private certificate requirements (PFX, 2048-bit key)
- SNI SSL vs IP-based SSL differences
- Free certificates issued by DigiCert
- App Service certificates stored in Key Vault

[Learn More](https://learn.microsoft.com/en-us/training/modules/configure-web-app-settings/6-configure-security-certificates)
