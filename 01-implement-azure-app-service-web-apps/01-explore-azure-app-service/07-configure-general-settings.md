# Configure General Settings

## Key Concepts
- Configure **platform and runtime** settings
- Control **security and debugging** options
- Manage **performance features** (Always On, ARR Affinity)
- Settings in **Configuration > General settings**

## Stack Settings

### Configure Runtime
- **Language and SDK versions** (e.g., .NET 8, Node.js 18, Python 3.11)
- **Linux apps**: Optional startup command or file
- Platform-specific configuration

```bash
# Set runtime stack (CLI)
az webapp config set \
  --name <app-name> \
  --resource-group <rg-name> \
  --linux-fx-version "NODE|18-lts"

# Windows app
az webapp config set \
  --name <app-name> \
  --resource-group <rg-name> \
  --net-framework-version "v8.0"
```

## Platform Settings

### Bitness (Windows Only)
| Option | Use Case |
|--------|----------|
| **32-bit** | Legacy apps, lower memory |
| **64-bit** | Modern apps, more memory |

### FTP State
| Option | Description |
|--------|-------------|
| **All allowed** | FTP and FTPS enabled |
| **FTPS only** | Secure FTP only |
| **Disabled** | No FTP access (recommended) |

### HTTP Version
- **HTTP/1.1** - Default, universally supported
- **HTTP/2.0** - Better performance, requires HTTPS/TLS
- ⚠️ Most browsers only support HTTP/2 over TLS

```bash
# Enable HTTP/2
az webapp config set \
  --name <app-name> \
  --resource-group <rg-name> \
  --http20-enabled true
```

### Web Sockets
- ✅ Enable for: **ASP.NET SignalR**, **socket.io**, real-time apps
- ❌ Disable for: Standard REST APIs, static sites

```bash
# Enable Web Sockets
az webapp config set \
  --name <app-name> \
  --resource-group <rg-name> \
  --web-sockets-enabled true
```

### Always On

#### What It Does
- Keeps app **loaded** even with no traffic
- **Prevents cold starts** (20-minute unload timer)
- Sends **GET request** to app root every 5 minutes

#### When to Enable
| Scenario | Always On |
|----------|-----------|
| **Continuous WebJobs** | ✅ Required |
| **CRON-triggered WebJobs** | ✅ Required |
| **Production apps** | ✅ Recommended |
| **Low-traffic apps** | ✅ Prevents unload |
| **Dev/test apps** | ❌ Optional |

```bash
# Enable Always On
az webapp config set \
  --name <app-name> \
  --resource-group <rg-name> \
  --always-on true
```

⚠️ **Not available on Free tier**

### ARR Affinity (Session Affinity)

#### What It Is
- Routes client to **same instance** for entire session
- Uses **cookies** to track sessions
- Also called "Sticky Sessions"

#### When to Use
| App Type | ARR Affinity |
|----------|--------------|
| **Stateful apps** | ✅ On |
| **Session-based apps** | ✅ On |
| **Stateless apps** | ❌ Off (better load distribution) |
| **Microservices** | ❌ Off |
| **APIs** | ❌ Off |

```bash
# Disable ARR Affinity (for stateless apps)
az webapp config set \
  --name <app-name> \
  --resource-group <rg-name> \
  --client-affinity-enabled false
```

💡 **Best Practice**: Turn OFF for stateless apps to improve load balancing

### HTTPS Only
- **Redirects all HTTP traffic** to HTTPS
- ✅ **Always enable for production**
- Required for security compliance

```bash
# Enforce HTTPS
az webapp update \
  --name <app-name> \
  --resource-group <rg-name> \
  --https-only true
```

### Minimum TLS Version
| Version | Status | Use |
|---------|--------|-----|
| **TLS 1.0** | ⚠️ Deprecated | Legacy only |
| **TLS 1.1** | ⚠️ Deprecated | Legacy only |
| **TLS 1.2** | ✅ Recommended | Production standard |
| **TLS 1.3** | ✅ Latest | Best security |

```bash
# Set minimum TLS version
az webapp config set \
  --name <app-name> \
  --resource-group <rg-name> \
  --min-tls-version "1.2"
```

## Debugging

### Remote Debugging
- Enable for: **ASP.NET**, **ASP.NET Core**, **Node.js**
- **Auto-disables after 48 hours** (security feature)
- Not for production use

```bash
# Enable remote debugging
az webapp config set \
  --name <app-name> \
  --resource-group <rg-name> \
  --remote-debugging-enabled true
```

## Client Certificates (Mutual TLS)

### Configuration
- **Require client certificates** for authentication
- Used for **mutual TLS authentication**
- Restricts access via certificate validation

```bash
# Require client certificates
az webapp update \
  --name <app-name> \
  --resource-group <rg-name> \
  --client-cert-enabled true
```

### Access Certificate in Code

```csharp
// C# - Access client certificate
var cert = Request.HttpContext.Connection.ClientCertificate;
var thumbprint = cert?.Thumbprint;
```

## Quick Reference

| Setting | Purpose | Tier Requirement |
|---------|---------|------------------|
| **Always On** | Keep app loaded | Basic+ |
| **ARR Affinity** | Sticky sessions | All tiers |
| **HTTPS Only** | Force HTTPS | All tiers |
| **HTTP/2** | Performance | All tiers |
| **Web Sockets** | Real-time | All tiers |
| **Remote Debugging** | Debugging | All tiers |
| **Client Certificates** | Mutual TLS | All tiers |

## Best Practices Matrix

| App Type | Always On | ARR Affinity | HTTPS Only | TLS Version |
|----------|-----------|--------------|------------|-------------|
| **Production Web App** | ✅ On | ⚠️ Depends | ✅ On | 1.2+ |
| **REST API (Stateless)** | ✅ On | ❌ Off | ✅ On | 1.2+ |
| **SignalR/Real-time** | ✅ On | ✅ On | ✅ On | 1.2+ |
| **Background Jobs** | ✅ On | ❌ Off | ✅ On | 1.2+ |
| **Dev/Test** | ❌ Off | ❌ Off | ⚠️ Optional | 1.2 |

## Critical Notes
- 💡 **Always On required** for Continuous and CRON WebJobs
- ⚠️ **Always On not available** on Free tier
- 🎯 **Disable ARR Affinity** for stateless apps (better performance)
- 🔐 **Always enforce HTTPS Only** in production
- ⏰ **Remote debugging auto-disables** after 48 hours
- 📊 **HTTP/2 requires TLS** - browsers only support HTTP/2 over HTTPS
- 🔄 **ARR Affinity = Sticky Sessions** = Client routed to same instance

## Exam Tips
- Know when to use Always On (WebJobs, production apps)
- Understand ARR Affinity (stateful vs stateless)
- Remember debugging auto-disables after 48 hours
- Know HTTPS Only redirects all HTTP to HTTPS
- HTTP/2 requires TLS for browser support
- Minimum TLS 1.2 recommended for production

[Learn More](https://learn.microsoft.com/en-us/training/modules/configure-web-app-settings/3-configure-general-settings)
