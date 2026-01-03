# Authentication and Authorization in App Service

## Key Concepts
- **Built-in authentication** - No code changes required
- **Platform middleware** - Runs before app code
- **Federated identity** - Third-party providers manage authentication
- **Token store** - Automatic token management
- **Zero SDK requirement** - Works with any language/framework

## Why Use Built-in Authentication?
- ✅ Save development time - no auth code to write
- ✅ Multiple providers supported
- ✅ Built into platform - no SDKs needed
- ✅ Token management included
- ✅ Works with any language

## Identity Providers

| Provider | Sign-in Endpoint | Acronym |
|----------|------------------|---------|
| **Microsoft Entra ID** | `/.auth/login/aad` | Azure AD |
| **Facebook** | `/.auth/login/facebook` | FB |
| **Google** | `/.auth/login/google` | - |
| **X (Twitter)** | `/.auth/login/x` | - |
| **GitHub** | `/.auth/login/github` | - |
| **Apple** | `/.auth/login/apple` | Preview |
| **OpenID Connect** | `/.auth/login/<providerName>` | Custom |

## Authentication Flow

### Server-Directed Flow (Without Provider SDK)
**Best for**: Browser apps

| Step | Action | Details |
|------|--------|---------|
| 1 | Sign user in | Redirect to `/.auth/login/<provider>` |
| 2 | Post-authentication | Provider redirects to `/.auth/login/<provider>/callback` |
| 3 | Establish session | App Service adds authenticated cookie |
| 4 | Serve content | Browser includes auth cookie automatically |

### Client-Directed Flow (With Provider SDK)
**Best for**: Mobile apps, SPAs, REST APIs

| Step | Action | Details |
|------|--------|---------|
| 1 | Sign user in | Client SDK signs in directly |
| 2 | Post-authentication | Client posts token to `/.auth/login/<provider>` |
| 3 | Establish session | App Service returns auth token |
| 4 | Serve content | Client includes `X-ZUMO-AUTH` header |

## Authorization Behavior

### Option 1: Allow Unauthenticated Requests
```
Configuration: "Allow anonymous"
```
- ✅ App code handles authorization
- ✅ More flexibility
- ✅ Auth info passed in HTTP headers
- ✅ Multiple sign-in providers available
- 📝 Good for public sites with optional login

### Option 2: Require Authentication
```
Configuration: "Require authentication"
```
- ❌ Rejects all unauthenticated traffic
- 🔄 Redirects browser to `/.auth/login/<provider>`
- ⚠️ Returns `HTTP 401 Unauthorized` for APIs/mobile apps
- ⚠️ Can return `HTTP 403 Forbidden` (configurable)
- 📝 Good for internal apps/authenticated-only apps

⚠️ **Caution**: Requiring auth blocks **all** requests, including publicly accessible pages (e.g., home page for SPAs)

## How the Middleware Works

The auth middleware:
1. ✅ Authenticates users with configured provider
2. ✅ Validates, stores, refreshes OAuth tokens
3. ✅ Manages authenticated session
4. ✅ Injects identity info into HTTP headers

**Key Characteristics**:
- Runs on same VM as your app (Windows)
- Runs in separate container (Linux/containers)
- No code changes required
- Configured via Azure Resource Manager or config file
- Language-agnostic

## Token Store

### What It Is
- **Built-in repository** of tokens for authenticated users
- Available for web apps, APIs, mobile apps
- Automatically enabled with any provider

### Access Tokens
- Available via **environment variables**
- Available via **HTTP headers**
- Only with built-in authentication enabled

## Logging and Tracing

```bash
# Enable application logging
az webapp log config \
  --name <app-name> \
  --resource-group <rg-name> \
  --application-logging filesystem

# View logs
az webapp log tail \
  --name <app-name> \
  --resource-group <rg-name>
```

- ✅ Auth traces collected in app logs
- ✅ Find auth errors in existing logs
- ✅ Convenient debugging

## Configuration Examples

### Configure Microsoft Entra ID

```bash
# Enable auth with Azure AD
az webapp auth update \
  --name <app-name> \
  --resource-group <rg-name> \
  --enabled true \
  --action LoginWithAzureActiveDirectory

# Configure Azure AD
az webapp auth microsoft update \
  --name <app-name> \
  --resource-group <rg-name> \
  --client-id <app-id> \
  --tenant-id <tenant-id>
```

### Access User Claims in Code

```csharp
// C# - Access claims
var identity = (ClaimsIdentity)User.Identity;
var claims = identity.Claims;

// Get user info from headers
var userId = Request.Headers["X-MS-CLIENT-PRINCIPAL-ID"];
var userName = Request.Headers["X-MS-CLIENT-PRINCIPAL-NAME"];
```

```javascript
// Node.js - Access user info
app.get('/api/user', (req, res) => {
  const userId = req.headers['x-ms-client-principal-id'];
  const userName = req.headers['x-ms-client-principal-name'];
  res.json({ userId, userName });
});
```

## Quick Reference

| Feature | Details |
|---------|---------|
| **Endpoints** | `/.auth/login/<provider>` |
| **Token validation** | Automatic |
| **Session management** | Built-in |
| **Identity in headers** | `X-MS-CLIENT-PRINCIPAL-*` |
| **Token header** | `X-ZUMO-AUTH` (client flow) |
| **Configuration** | Portal, ARM, or config file |

## Critical Notes
- 💡 **No code changes needed** - pure configuration
- 🎯 **Multiple providers supported** - mix and match
- ⚠️ **Linux = separate container** - no in-process integration
- 🔐 **Token store automatic** - refresh tokens managed
- 📝 **Auth middleware runs first** - before app code
- 🚀 **Works with any language** - no SDK required
- ⚠️ **Require auth blocks all traffic** - including home page

## Exam Tips
- Know the two auth flows: server-directed vs client-directed
- Understand auth middleware runs before app code
- Remember `/.auth/login/<provider>` pattern
- Token store is automatic when auth is enabled
- No code changes required - configuration only
- Auth info available in HTTP headers (`X-MS-CLIENT-PRINCIPAL-*`)

[Learn More](https://learn.microsoft.com/en-us/training/modules/introduction-to-azure-app-service/5-authentication-authorization-app-service)
