# Authentication and Authorization in Azure Container Apps

## Key Concepts
- **Built-in auth** - Federated identity with minimal code
- **Identity providers** - Microsoft, Google, Facebook, GitHub, X, OpenID Connect
- **Sidecar architecture** - Auth handled by platform middleware
- **Server-directed flow** - Browser-based auth
- **Client-directed flow** - Mobile/API auth

## Built-In Authentication

### What Is It?

**Platform-level authentication** without custom code:

- Federated identity providers
- Runs as sidecar container
- Intercepts HTTP requests before reaching app
- Manages authentication session
- Injects identity info in headers

### Benefits
✅ **No code required** - Platform handles auth
✅ **Multiple providers** - Support various identity systems
✅ **Session management** - Platform manages tokens
✅ **Identity injection** - User info in request headers
✅ **HTTPS only** - Secure authentication

⚠️ **Important**: Only works with HTTPS. Disable `allowInsecure` on ingress.

## Identity Providers

### Supported Providers

| Provider | Sign-In Endpoint | Description |
|----------|-----------------|-------------|
| **Microsoft Identity Platform** | `/.auth/login/aad` | Microsoft Entra ID (Azure AD) |
| **Facebook** | `/.auth/login/facebook` | Facebook accounts |
| **GitHub** | `/.auth/login/github` | GitHub accounts |
| **Google** | `/.auth/login/google` | Google accounts |
| **X (Twitter)** | `/.auth/login/twitter` | X/Twitter accounts |
| **OpenID Connect** | `/.auth/login/<providerName>` | Any OIDC provider |

### Provider Configuration

#### Microsoft Identity Platform (Entra ID)
```bash
# Configure Microsoft provider
az containerapp auth microsoft update \
  --name myapp \
  --resource-group myResourceGroup \
  --client-id <app-id> \
  --client-secret-setting-name microsoft-provider-authentication-secret \
  --issuer https://login.microsoftonline.com/<tenant-id>/v2.0
```

#### Generic OpenID Connect
```bash
# Configure custom OIDC provider
az containerapp auth openid-connect add \
  --name myapp \
  --resource-group myResourceGroup \
  --provider-name auth0 \
  --client-id <client-id> \
  --client-secret-setting-name auth0-secret \
  --openid-issuer https://<tenant>.auth0.com/
```

## Feature Architecture

### How It Works

**Sidecar container** handles authentication:

```
Internet
    ↓
Ingress (HTTPS)
    ↓
Auth Sidecar Container (middleware)
├── Authenticates user
├── Manages session
├── Validates tokens
└── Injects identity headers
    ↓
Application Container
└── Receives authenticated requests
```

### Auth Middleware Responsibilities

✅ **Authenticates users** - Validates credentials with providers
✅ **Manages sessions** - Handles tokens and cookies
✅ **Injects identity** - Adds user info to request headers
✅ **Token validation** - Verifies JWT tokens

### No In-Process Integration
⚠️ **Separate container** - Auth runs isolated from app code
💡 **Identity via headers** - App reads user info from HTTP headers

## Authentication Flows

### 1. Server-Directed Flow (Browser Apps)

**Platform handles sign-in**:

```
1. User → GET /protected-resource
2. Platform → Redirect to /.auth/login/<provider>
3. User → Sign in at provider
4. Provider → Return to /.auth/login/<provider>/callback
5. Platform → Set auth cookie
6. Platform → Redirect to /protected-resource
7. App → Receives request with identity headers
```

**Use for**:
- Web applications
- Browser-based apps
- Server-side rendered apps

**Example**: User clicks "Sign in with Google" button

### 2. Client-Directed Flow (Mobile/API Apps)

**App handles sign-in, platform validates**:

```
1. Mobile App → Sign in with provider SDK
2. Provider → Return access token to app
3. Mobile App → POST token to /.auth/login/<provider>
4. Platform → Validate token with provider
5. Platform → Return session token
6. Mobile App → Include session token in requests
7. App → Receives authenticated requests
```

**Use for**:
- Native mobile apps
- Single-page applications (SPAs)
- Browser-less apps
- API clients

**Example**: Mobile app uses Google SDK, then validates with Container Apps

## Access Restrictions

### Require Authentication
```bash
# Require authentication for all requests
az containerapp auth update \
  --name myapp \
  --resource-group myResourceGroup \
  --unauthenticated-client-action RedirectToLoginPage

# Options:
# - RedirectToLoginPage: Redirect to sign-in
# - Return401: Return HTTP 401
# - Return403: Return HTTP 403
# - AllowAnonymous: Allow unauthenticated requests
```

### Allow Unauthenticated Access
```bash
# Allow anonymous access (validate tokens if present)
az containerapp auth update \
  --name myapp \
  --resource-group myResourceGroup \
  --unauthenticated-client-action AllowAnonymous
```

### Configuration Comparison

| Mode | Behavior | Use Case |
|------|----------|----------|
| **RedirectToLoginPage** | Auto-redirect to sign-in | Web apps |
| **Return401** | Return unauthorized | APIs, SPAs |
| **Return403** | Return forbidden | APIs |
| **AllowAnonymous** | Optional auth | Mixed public/private content |

## Identity Information in Headers

### Standard Headers Injected

| Header | Description |
|--------|-------------|
| `X-MS-CLIENT-PRINCIPAL-ID` | User's unique ID |
| `X-MS-CLIENT-PRINCIPAL-NAME` | User's name |
| `X-MS-CLIENT-PRINCIPAL-IDP` | Identity provider used |
| `X-MS-CLIENT-PRINCIPAL` | Base64-encoded JSON with user claims |
| `X-MS-TOKEN-<provider>-ACCESS-TOKEN` | Provider access token |
| `X-MS-TOKEN-<provider>-ID-TOKEN` | Provider ID token |

### Reading Identity in Application

#### Node.js/Express
```javascript
app.get('/profile', (req, res) => {
  const userId = req.headers['x-ms-client-principal-id'];
  const userName = req.headers['x-ms-client-principal-name'];
  const provider = req.headers['x-ms-client-principal-idp'];
  
  res.json({
    id: userId,
    name: userName,
    provider: provider
  });
});
```

#### Python/Flask
```python
@app.route('/profile')
def profile():
    user_id = request.headers.get('X-MS-CLIENT-PRINCIPAL-ID')
    user_name = request.headers.get('X-MS-CLIENT-PRINCIPAL-NAME')
    provider = request.headers.get('X-MS-CLIENT-PRINCIPAL-IDP')
    
    return {
        'id': user_id,
        'name': user_name,
        'provider': provider
    }
```

#### .NET/C#
```csharp
[HttpGet("profile")]
public IActionResult GetProfile()
{
    var userId = Request.Headers["X-MS-CLIENT-PRINCIPAL-ID"];
    var userName = Request.Headers["X-MS-CLIENT-PRINCIPAL-NAME"];
    var provider = Request.Headers["X-MS-CLIENT-PRINCIPAL-IDP"];
    
    return Ok(new {
        Id = userId,
        Name = userName,
        Provider = provider
    });
}
```

## Token Management

### Access Tokens
```bash
# Access provider tokens via special endpoint
GET /.auth/me

# Returns:
# - User information
# - Access tokens for configured providers
# - ID tokens
# - Refresh tokens (if available)
```

### Token Refresh
```bash
# Refresh tokens automatically
GET /.auth/refresh

# Platform refreshes tokens with provider
# Returns new tokens
```

### Logout
```bash
# Sign out user
GET /.auth/logout

# Clears session
# Optionally redirects to provider logout
```

## Configuration Example

### Complete Auth Setup
```bash
# 1. Enable Microsoft authentication
az containerapp auth microsoft update \
  --name myapp \
  --resource-group myResourceGroup \
  --client-id $CLIENT_ID \
  --client-secret-setting-name microsoft-secret \
  --issuer https://login.microsoftonline.com/$TENANT_ID/v2.0

# 2. Store client secret
az containerapp secret set \
  --name myapp \
  --resource-group myResourceGroup \
  --secrets microsoft-secret=$CLIENT_SECRET

# 3. Require authentication
az containerapp auth update \
  --name myapp \
  --resource-group myResourceGroup \
  --unauthenticated-client-action RedirectToLoginPage

# 4. Configure allowed redirect URLs in Azure AD:
# https://myapp.<environment-id>.<region>.azurecontainerapps.io/.auth/login/aad/callback
```

## Best Practices

### 1. Always Use HTTPS
```bash
# Disable insecure HTTP
az containerapp ingress update \
  --name myapp \
  --resource-group myResourceGroup \
  --allow-insecure false
```

### 2. Store Secrets Securely
```bash
# Store provider secrets in Container App secrets
az containerapp secret set \
  --name myapp \
  --resource-group myResourceGroup \
  --secrets provider-secret=$SECRET
```

### 3. Use Appropriate Flow
- **Server-directed**: Web apps with browsers
- **Client-directed**: Mobile apps, SPAs

### 4. Configure Token Expiration
```bash
# Set session timeout
az containerapp auth update \
  --name myapp \
  --resource-group myResourceGroup \
  --token-store true \
  --sas-url-secret session-secret
```

### 5. Implement Logout
```html
<!-- Add logout link -->
<a href="/.auth/logout">Sign Out</a>
```

## Limitations

❌ **HTTPS required** - Won't work with HTTP
❌ **External ingress** - Must be externally accessible
❌ **No offline validation** - Requires provider connectivity
❌ **Session cookies** - Some mobile scenarios may need client-directed flow

## Critical Notes
- 💡 **Built-in auth** - No code required, platform handles it
- ⚠️ **HTTPS only** - Disable allowInsecure for security
- 🎯 **Sidecar architecture** - Auth runs in separate container
- ✅ **Identity headers** - User info injected into HTTP headers
- 📊 **Server flow** - Browser apps (redirect to provider)
- 🔄 **Client flow** - Mobile apps (app signs in, platform validates)
- 🔒 **Access control** - Require auth, allow anonymous, return 401/403
- ⚠️ **Multiple providers** - Support multiple identity systems

## Exam Tips
- Built-in authentication: Federated identity without code
- Identity providers: Microsoft, Google, Facebook, GitHub, X, OpenID Connect
- Sign-in endpoints: `/.auth/login/<provider>`
- Sidecar architecture: Auth middleware in separate container
- Server-directed flow: Browser apps, platform handles redirect
- Client-directed flow: Mobile apps, app gets token then validates
- Identity headers: X-MS-CLIENT-PRINCIPAL-ID, X-MS-CLIENT-PRINCIPAL-NAME
- Access restriction: RedirectToLoginPage, Return401, Return403, AllowAnonymous
- HTTPS required: Must disable allowInsecure
- Token endpoints: /.auth/me (info), /.auth/refresh (refresh), /.auth/logout (logout)
- Platform responsibilities: Authenticate, manage session, inject identity, validate tokens
- No in-process integration: Identity via HTTP headers only

[Learn More](https://learn.microsoft.com/en-us/training/modules/implement-azure-container-apps/5-container-apps-authentication)
