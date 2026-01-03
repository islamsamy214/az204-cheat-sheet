# Create Paired Keys and Values

## Overview

Azure App Configuration stores configuration data as **key-value pairs**. Keys serve as names to store and retrieve corresponding values. Understanding how to structure keys, use labels, and manage values effectively is essential for scalable configuration management.

---

## Keys

### What Are Keys?

Keys are **unique identifiers** for configuration values. They function as names in key-value pairs and are used to store and retrieve corresponding values.

**Key Characteristics**:
- **Case-sensitive**: `AppName` ≠ `appname`
- **Unicode support**: Use any unicode character except `*`, `,`, and `\`
- **Size limit**: 10,000 characters combined for key + value + attributes
- **Treated as whole**: App Configuration doesn't parse key structure

### Key Naming Best Practices

#### Flat vs. Hierarchical Naming

**Flat Naming**:
```
DatabaseConnectionString
CacheEndpoint
LoggingLevel
ApiTimeout
```

**Hierarchical Naming** (Recommended):
```
MyApp:Database:ConnectionString
MyApp:Cache:Endpoint
MyApp:Logging:Level
MyApp:Api:Timeout
```

**Why Hierarchical is Better**:
- ✅ **Easier to read**: Delimiters act as spaces in a sentence
- ✅ **Easier to manage**: Logical grouping of related settings
- ✅ **Easier to use**: Pattern matching for bulk retrieval
- ✅ **Better organization**: Clear structure and relationships

### Design Key Namespaces

#### Component-Based Hierarchy

**Pattern**: `AppName:Component:Setting`

```
MyApp:Database:Server
MyApp:Database:Port
MyApp:Database:Timeout
MyApp:Cache:Redis:Endpoint
MyApp:Cache:Redis:Port
MyApp:Logging:Level
MyApp:Logging:Provider
```

**Benefits**:
- Clear ownership and responsibility
- Easy to query all settings for a component
- Consistent across services

#### Environment-Agnostic Keys

**Don't do this** ❌:
```
MyApp:Database:ConnectionString:Dev
MyApp:Database:ConnectionString:Staging
MyApp:Database:ConnectionString:Prod
```

**Do this instead** ✅:
```
# Same key with different labels
Key: MyApp:Database:ConnectionString, Label: Development
Key: MyApp:Database:ConnectionString, Label: Staging
Key: MyApp:Database:ConnectionString, Label: Production
```

### Framework-Specific Naming

#### .NET / ASP.NET Core

Colon (`:`) separator:
```
AppName:Service1:ApiEndpoint
AppName:Service2:ApiEndpoint
```

Maps to .NET configuration:
```csharp
var endpoint = configuration["AppName:Service1:ApiEndpoint"];

// Or strongly-typed
public class AppSettings
{
    public ServiceSettings Service1 { get; set; }
}

public class ServiceSettings
{
    public string ApiEndpoint { get; set; }
}
```

#### Java Spring Cloud

Spring Cloud `Environment` resources:
```
spring.datasource.url
spring.datasource.username
spring.redis.host
spring.redis.port
```

#### JavaScript/Node.js

Dot (`.`) or colon (`:`) separator:
```
app.database.host
app.database.port
app:api:endpoint
```

### Reserved Characters

| Character | Status | Usage |
|-----------|--------|-------|
| `*` | Reserved | Cannot use in key names |
| `,` | Reserved | Cannot use in key names |
| `\` | Reserved | Cannot use in key names (escape character) |
| `:` | Allowed | Common delimiter |
| `/` | Allowed | Common delimiter |
| `.` | Allowed | Common delimiter |
| `-` | Allowed | Common in kebab-case |
| `_` | Allowed | Common in snake_case |

**Escaping Reserved Characters**:
```
# If you must use reserved character
Key: MyApp\*Special

# Recommended: Avoid reserved characters
Key: MyApp-Special
```

---

## Labels

### What Are Labels?

Labels are **optional attributes** that differentiate key-values with the same key. They enable creating variants of a key without changing the key name.

**Label Characteristics**:
- Optional (default: no label)
- Case-sensitive
- Used to create key variants
- Common use: Environment differentiation

### Label Use Cases

#### 1. Environment Segmentation (Most Common)

**Setup**:
```bash
# Development environment
az appconfig kv set --name myappconfig \
  --key "MyApp:Database:Server" \
  --value "dev-sql.database.windows.net" \
  --label "Development"

# Staging environment
az appconfig kv set --name myappconfig \
  --key "MyApp:Database:Server" \
  --value "staging-sql.database.windows.net" \
  --label "Staging"

# Production environment
az appconfig kv set --name myappconfig \
  --key "MyApp:Database:Server" \
  --value "prod-sql.database.windows.net" \
  --label "Production"
```

**Application Usage**:
```csharp
// Automatically select based on environment
string environment = builder.Environment.EnvironmentName; // "Development", "Staging", "Production"

builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(connectionString)
           .Select(KeyFilter.Any, environment); // Filter by label
});
```

#### 2. Version Management

**Setup**:
```bash
# Version 1.0
az appconfig kv set --key "MyApp:ApiEndpoint" --value "https://api.v1.contoso.com" --label "v1.0"

# Version 2.0
az appconfig kv set --key "MyApp:ApiEndpoint" --value "https://api.v2.contoso.com" --label "v2.0"
```

**Usage**:
```csharp
// Load specific version
options.Select(KeyFilter.Any, "v1.0");
```

#### 3. Feature Branch Testing

**Setup**:
```bash
# Main branch
az appconfig kv set --key "MyApp:Feature:NewUI" --value "false" --label "main"

# Feature branch
az appconfig kv set --key "MyApp:Feature:NewUI" --value "true" --label "feature-new-ui"
```

#### 4. A/B Testing Variants

**Setup**:
```bash
# Variant A
az appconfig kv set --key "MyApp:Checkout:Layout" --value "single-page" --label "variant-a"

# Variant B
az appconfig kv set --key "MyApp:Checkout:Layout" --value "multi-step" --label "variant-b"
```

### Referencing Key-Values Without Labels

**Default (no label)**:
```bash
# Set key without label
az appconfig kv set --key "MyApp:Setting" --value "value"

# Reference in application
var setting = configuration["MyApp:Setting"];
```

**Explicit reference to no label**:
```csharp
// Use \0 to explicitly reference unlabeled key
options.Select(KeyFilter.Any, LabelFilter.Null);
```

In REST API, use URL-encoded `\0` as `%00`:
```
GET https://myappconfig.azconfig.io/kv/MyApp:Setting?label=%00
```

---

## Values

### What Are Values?

Values are **unicode strings** assigned to keys. They represent the actual configuration data.

**Value Characteristics**:
- Unicode strings (any unicode character)
- Optional content-type attribute
- Size limit: 10 KB (combined with key and attributes)
- Encrypted at rest and in transit

### Content Type Attribute

The optional `content-type` attribute helps applications understand how to process the value.

**Common Content Types**:
- `application/json` - JSON data
- `text/plain` - Plain text
- `application/xml` - XML data
- Custom types (e.g., `application/vnd.myapp.config+json`)

**Example**:
```bash
# JSON value with content type
az appconfig kv set \
  --name myappconfig \
  --key "MyApp:ConnectionStrings" \
  --value '{"Database":"Server=...;","Redis":"host:port"}' \
  --content-type "application/json"
```

**Application Usage**:
```csharp
var configValue = await client.GetConfigurationSettingAsync("MyApp:ConnectionStrings");

if (configValue.Value.ContentType == "application/json")
{
    var connectionStrings = JsonSerializer.Deserialize<Dictionary<string, string>>(configValue.Value.Value);
}
```

### Value Examples

#### Simple String
```bash
az appconfig kv set --key "MyApp:AppName" --value "My Application"
```

#### JSON Object
```bash
az appconfig kv set \
  --key "MyApp:Features" \
  --value '{"EnableNewUI":true,"EnableBetaFeatures":false}' \
  --content-type "application/json"
```

#### Connection String
```bash
az appconfig kv set \
  --key "MyApp:Database:ConnectionString" \
  --value "Server=tcp:myserver.database.windows.net,1433;Database=mydb;"
```

#### Comma-Separated List
```bash
az appconfig kv set \
  --key "MyApp:AllowedOrigins" \
  --value "https://app1.contoso.com,https://app2.contoso.com,https://app3.contoso.com"
```

---

## Querying Key-Values

### Pattern Matching

App Configuration supports pattern matching for bulk retrieval.

#### Wildcard Patterns

**All keys**:
```bash
az appconfig kv list --name myappconfig --key "*"
```

**Keys starting with prefix**:
```bash
az appconfig kv list --name myappconfig --key "MyApp:Database:*"
```

**Keys for specific component**:
```bash
az appconfig kv list --name myappconfig --key "MyApp:Cache:*"
```

#### Filter by Label

**Specific label**:
```bash
az appconfig kv list --name myappconfig --label "Production"
```

**No label**:
```bash
az appconfig kv list --name myappconfig --label "\0"
```

**Any label** (including no label):
```bash
az appconfig kv list --name myappconfig
```

### .NET Configuration API

**All keys**:
```csharp
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(connectionString)
           .Select(KeyFilter.Any); // All keys, no label filter
});
```

**Specific prefix**:
```csharp
options.Select("MyApp:Database:*"); // All database settings
```

**Specific label**:
```csharp
options.Select(KeyFilter.Any, "Production"); // All keys with Production label
```

**Multiple filters**:
```csharp
options.Select("MyApp:*", "Production")     // MyApp keys in Production
       .Select("Shared:*", LabelFilter.Null); // Shared keys without label
```

### REST API Queries

**List all key-values**:
```http
GET https://myappconfig.azconfig.io/kv
```

**Filter by key prefix**:
```http
GET https://myappconfig.azconfig.io/kv?key=MyApp:Database:*
```

**Filter by label**:
```http
GET https://myappconfig.azconfig.io/kv?label=Production
```

**Combined filter**:
```http
GET https://myappconfig.azconfig.io/kv?key=MyApp:*&label=Production
```

---

## Versioning Key Values

App Configuration **doesn't automatically version** key-values when modified. Use labels to implement versioning manually.

### Version with Labels

**Approach 1: Explicit Version Labels**
```bash
# Version 1.0
az appconfig kv set --key "MyApp:ApiContract" --value "v1-schema" --label "1.0"

# Version 1.1
az appconfig kv set --key "MyApp:ApiContract" --value "v1.1-schema" --label "1.1"

# Version 2.0
az appconfig kv set --key "MyApp:ApiContract" --value "v2-schema" --label "2.0"
```

**Approach 2: Git Commit ID**
```bash
COMMIT_SHA=$(git rev-parse HEAD)

az appconfig kv set \
  --key "MyApp:BuildConfig" \
  --value "optimized-settings" \
  --label "$COMMIT_SHA"
```

**Approach 3: Timestamp**
```bash
TIMESTAMP=$(date +%Y%m%d%H%M%S)

az appconfig kv set \
  --key "MyApp:DeploymentConfig" \
  --value "latest-config" \
  --label "$TIMESTAMP"
```

### Point-in-Time Snapshots

While labels provide versioning, App Configuration also maintains historical data:

**View key history**:
```bash
# Get key at specific point in time (not directly supported via CLI)
# Use REST API or SDK

# .NET SDK example
var snapshots = client.GetRevisions(new SettingSelector { KeyFilter = "MyApp:Setting" });
foreach (var snapshot in snapshots)
{
    Console.WriteLine($"{snapshot.LastModified}: {snapshot.Value}");
}
```

---

## Practical Examples

### Example 1: Multi-Environment Application

**Setup**:
```bash
APP_CONFIG="myappconfig"

# Shared settings (no label)
az appconfig kv set --name $APP_CONFIG --key "MyApp:AppName" --value "My Application"
az appconfig kv set --name $APP_CONFIG --key "MyApp:Version" --value "2.0"

# Development
az appconfig kv set --name $APP_CONFIG --key "MyApp:Database:Server" --value "dev-sql.database.windows.net" --label "Development"
az appconfig kv set --name $APP_CONFIG --key "MyApp:Logging:Level" --value "Debug" --label "Development"

# Staging
az appconfig kv set --name $APP_CONFIG --key "MyApp:Database:Server" --value "staging-sql.database.windows.net" --label "Staging"
az appconfig kv set --name $APP_CONFIG --key "MyApp:Logging:Level" --value "Information" --label "Staging"

# Production
az appconfig kv set --name $APP_CONFIG --key "MyApp:Database:Server" --value "prod-sql.database.windows.net" --label "Production"
az appconfig kv set --name $APP_CONFIG --key "MyApp:Logging:Level" --value "Warning" --label "Production"
```

**Application Code**:
```csharp
var environment = builder.Environment.EnvironmentName;

builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(connectionString)
           .Select(KeyFilter.Any, LabelFilter.Null)  // Shared settings (no label)
           .Select(KeyFilter.Any, environment);       // Environment-specific
});

// Result: Shared + environment-specific settings merged
```

### Example 2: Microservices Configuration

**Setup**:
```bash
# Shared infrastructure settings
az appconfig kv set --key "Shared:Redis:Host" --value "myredis.redis.cache.windows.net"
az appconfig kv set --key "Shared:ServiceBus:Namespace" --value "myservicebus.servicebus.windows.net"

# Service 1 settings
az appconfig kv set --key "OrderService:Database:Name" --value "orders-db"
az appconfig kv set --key "OrderService:Api:Timeout" --value "30"

# Service 2 settings
az appconfig kv set --key "PaymentService:Database:Name" --value "payments-db"
az appconfig kv set --key "PaymentService:Api:Timeout" --value "60"
```

**Service 1 Code**:
```csharp
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(connectionString)
           .Select("Shared:*")         // Load shared settings
           .Select("OrderService:*");   // Load service-specific settings
});
```

**Service 2 Code**:
```csharp
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(connectionString)
           .Select("Shared:*")           // Load shared settings
           .Select("PaymentService:*");   // Load service-specific settings
});
```

### Example 3: JSON Configuration Values

**Setup**:
```bash
# Store complex configuration as JSON
az appconfig kv set \
  --name myappconfig \
  --key "MyApp:EmailSettings" \
  --value '{
    "SmtpServer": "smtp.office365.com",
    "Port": 587,
    "UseSsl": true,
    "FromAddress": "noreply@contoso.com"
  }' \
  --content-type "application/json"
```

**Application Code**:
```csharp
// Define model
public class EmailSettings
{
    public string SmtpServer { get; set; }
    public int Port { get; set; }
    public bool UseSsl { get; set; }
    public string FromAddress { get; set; }
}

// Bind to model
var emailSettings = new EmailSettings();
configuration.GetSection("MyApp:EmailSettings").Bind(emailSettings);

// Or use Options pattern
services.Configure<EmailSettings>(configuration.GetSection("MyApp:EmailSettings"));
```

---

## Best Practices

### 1. **Use Hierarchical Key Names**

✅ **Do**:
```
MyApp:Database:Server
MyApp:Database:Port
MyApp:Cache:Endpoint
```

❌ **Don't**:
```
db_server
DatabasePort
cache-endpoint
```

### 2. **Use Labels for Environment Variants**

✅ **Do**:
```bash
# Same key, different labels
az appconfig kv set --key "ApiEndpoint" --value "https://dev-api.com" --label "dev"
az appconfig kv set --key "ApiEndpoint" --value "https://api.com" --label "prod"
```

❌ **Don't**:
```bash
# Different keys for environments
az appconfig kv set --key "ApiEndpoint-Dev" --value "https://dev-api.com"
az appconfig kv set --key "ApiEndpoint-Prod" --value "https://api.com"
```

### 3. **Keep Values Small**

- 10 KB limit per key-value pair
- Store large data in Blob Storage, reference URI in App Configuration
- Break large JSON into multiple keys

### 4. **Use Content Type for Structured Data**

```bash
az appconfig kv set \
  --key "MyApp:Settings" \
  --value '{"timeout":30,"retries":3}' \
  --content-type "application/json"
```

### 5. **Avoid Sensitive Data**

❌ **Don't store in App Configuration**:
```bash
az appconfig kv set --key "Database:Password" --value "MyP@ssw0rd123"
```

✅ **Do store in Key Vault, reference from App Configuration**:
```bash
# Store password in Key Vault
az keyvault secret set --vault-name myvault --name DbPassword --value "MyP@ssw0rd123"

# Store Key Vault reference in App Configuration
az appconfig kv set-keyvault \
  --name myappconfig \
  --key "Database:Password" \
  --secret-identifier "https://myvault.vault.azure.net/secrets/DbPassword"
```

---

## Exam Tips

### Key Concepts for AZ-204

1. **Keys are case-sensitive**: `MyApp` ≠ `myapp`

2. **Labels differentiate keys**: Same key, different values based on label

3. **No automatic versioning**: Use labels for manual versioning (e.g., "v1.0", "v2.0")

4. **10 KB size limit**: Combined size of key + value + attributes

5. **Hierarchical naming**: Use delimiters (`:`, `/`, `.`) for organization

6. **Content-type attribute**: Optional, helps applications process values correctly

7. **Pattern matching**: Use `*` wildcard for bulk queries

8. **No label = `\0`**: Explicitly reference unlabeled keys with `\0` (URL: `%00`)

9. **Reserved characters**: Cannot use `*`, `,`, `\` in key names

10. **Encryption**: All data encrypted at rest and in transit

11. **Not for secrets**: Use Key Vault for passwords, App Configuration for non-sensitive settings

12. **Point-in-time replay**: Query historical configuration values

### Common Exam Scenarios

**Scenario 1**: "Store database connection string for dev, staging, prod"
→ **Answer**: Use same key with different labels (Development, Staging, Production)

**Scenario 2**: "Organize settings for multiple microservices"
→ **Answer**: Use hierarchical keys (ServiceName:Component:Setting)

**Scenario 3**: "Store API key securely"
→ **Answer**: Store in Key Vault, reference from App Configuration with `az appconfig kv set-keyvault`

**Scenario 4**: "Query all database settings"
→ **Answer**: Use pattern matching: `MyApp:Database:*`

**Scenario 5**: "Version configuration for rollback capability"
→ **Answer**: Use labels with version numbers or timestamps

---

## Quick Reference Commands

```bash
# Set key-value
az appconfig kv set --name <name> --key <key> --value <value>

# Set with label
az appconfig kv set --name <name> --key <key> --value <value> --label <label>

# Set with content type
az appconfig kv set --name <name> --key <key> --value <value> --content-type <type>

# Get key-value
az appconfig kv show --name <name> --key <key>

# Get with label
az appconfig kv show --name <name> --key <key> --label <label>

# List all
az appconfig kv list --name <name>

# List with pattern
az appconfig kv list --name <name> --key "MyApp:Database:*"

# List with label
az appconfig kv list --name <name> --label "Production"

# Delete key-value
az appconfig kv delete --name <name> --key <key>

# Delete with label
az appconfig kv delete --name <name> --key <key> --label <label>

# Set Key Vault reference
az appconfig kv set-keyvault \
  --name <name> \
  --key <key> \
  --secret-identifier <vault-secret-uri>

# Export to file
az appconfig kv export --name <name> --destination file --path config.json

# Import from file
az appconfig kv import --name <name> --source file --path config.json
```

---

## Learn More

- [Azure App Configuration Keys and Values](https://docs.microsoft.com/azure/azure-app-configuration/concept-key-value)
- [Use labels to enable different configurations](https://docs.microsoft.com/azure/azure-app-configuration/howto-labels-aspnet-core)
- [Best practices for key-value naming](https://docs.microsoft.com/azure/azure-app-configuration/howto-best-practices#key-value-compositions)
