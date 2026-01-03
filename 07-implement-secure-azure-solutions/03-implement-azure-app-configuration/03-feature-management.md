# Manage Application Features

## What is Feature Management?

**Feature management** is a modern software development practice that decouples feature release from code deployment. It enables quick changes to feature availability on demand without modifying or redeploying code.

Also known as:
- Feature flags
- Feature toggles
- Feature switches
- Dark launches

---

## Why Use Feature Management?

### Traditional Deployment Challenge

**Without Feature Flags**:
```
Code Change → Build → Test → Deploy → Feature Live
```

**Problems**:
- Feature goes live immediately upon deployment
- Cannot selectively enable for subset of users
- Difficult to roll back without redeployment
- Risk of incomplete features affecting all users

### Modern Approach with Feature Flags

**With Feature Flags**:
```
Code Change (with flag) → Build → Deploy → Control Feature via Configuration
```

**Benefits**:
- ✅ Deploy code with feature disabled
- ✅ Enable for specific users/groups
- ✅ Gradual rollout (10% → 50% → 100%)
- ✅ Instant rollback without redeployment
- ✅ A/B testing and experimentation
- ✅ Separate deployment from release

---

## Basic Concepts

### 1. Feature Flag

A **feature flag** is a variable with a binary state (on/off) that determines whether a code block executes.

**Simple Example**:
```csharp
if (featureFlag)
{
    // Run the following code
}
```

**Characteristics**:
- Binary state: `true` (on) or `false` (off)
- Associated with a code block
- State triggers whether code executes
- Can be static or dynamic

### 2. Feature Manager

A **feature manager** is an application package that handles the lifecycle of all feature flags.

**Responsibilities**:
- Load feature flags from configuration source
- Evaluate feature flag state
- Cache feature flags for performance
- Update states dynamically
- Apply filters and rules

**Popular Libraries**:
- **.NET**: `Microsoft.FeatureManagement`
- **Java Spring**: Spring Cloud Feature Management
- **JavaScript**: Custom or third-party libraries
- **Python**: Custom implementations

### 3. Filter

A **filter** is a rule for evaluating the state of a feature flag.

**Filter Types**:
- **Percentage**: Enable for X% of users
- **Targeting**: Enable for specific users/groups
- **Time window**: Enable during specific dates/times
- **Geographic**: Enable for specific regions
- **Browser/Device**: Enable for specific browsers or devices
- **Custom**: Your own business logic

---

## How Feature Flags Work

### Component Interaction

```
┌──────────────────────────────────────────────────────────┐
│  Application Code                                         │
│  ┌────────────────────────────────────────────────┐      │
│  │  if (await featureManager.IsEnabledAsync(      │      │
│  │         "NewCheckout"))                        │      │
│  │  {                                              │      │
│  │      // New checkout process                   │      │
│  │  }                                              │      │
│  │  else                                           │      │
│  │  {                                              │      │
│  │      // Old checkout process                   │      │
│  │  }                                              │      │
│  └─────────────────┬──────────────────────────────┘      │
│                    │                                      │
│                    ▼                                      │
│  ┌────────────────────────────────────────────────┐      │
│  │  Feature Manager                               │      │
│  │  • Loads flags from configuration              │      │
│  │  • Evaluates filters                           │      │
│  │  • Caches results                              │      │
│  │  • Returns true/false                          │      │
│  └─────────────────┬──────────────────────────────┘      │
└────────────────────┼───────────────────────────────────────┘
                     │
                     ▼
   ┌─────────────────────────────────────────┐
   │  Configuration Repository               │
   │  (Azure App Configuration)              │
   │  ┌──────────────────────────────────┐   │
   │  │  FeatureManagement:              │   │
   │  │    NewCheckout:                  │   │
   │  │      EnabledFor:                 │   │
   │  │        - Percentage: 25%         │   │
   │  │    BetaFeatures:                 │   │
   │  │      EnabledFor:                 │   │
   │  │        - Targeting: BetaUsers    │   │
   │  └──────────────────────────────────┘   │
   └─────────────────────────────────────────┘
```

---

## Feature Flag Usage in Code

### 1. Static Boolean Flag

**Simplest form**:
```csharp
bool featureFlag = true;

if (featureFlag)
{
    // Run the following code
}
```

**Use case**: Quick on/off toggle during development

### 2. Rule-Based Evaluation

**Evaluate based on logic**:
```csharp
bool featureFlag = isBetaUser();

if (featureFlag)
{
    // Beta feature code
}
```

### 3. Conditional Branching

**Different behavior based on state**:
```csharp
if (featureFlag)
{
    // This code runs if featureFlag is true
}
else
{
    // This code runs if featureFlag is false
}
```

### 4. .NET Feature Manager (Recommended)

**ASP.NET Core example**:
```csharp
using Microsoft.FeatureManagement;

public class CheckoutController : Controller
{
    private readonly IFeatureManager _featureManager;

    public CheckoutController(IFeatureManager featureManager)
    {
        _featureManager = featureManager;
    }

    public async Task<IActionResult> Process()
    {
        if (await _featureManager.IsEnabledAsync("NewCheckout"))
        {
            return await ProcessNewCheckout();
        }
        else
        {
            return await ProcessOldCheckout();
        }
    }
}
```

### 5. Razor View Example

**Toggle UI elements**:
```html
@inject IFeatureManager FeatureManager

@if (await FeatureManager.IsEnabledAsync("NewUI"))
{
    <div class="new-ui">
        <!-- New UI components -->
    </div>
}
else
{
    <div class="old-ui">
        <!-- Old UI components -->
    </div>
}
```

### 6. Feature Gate Attribute

**Controller-level gating**:
```csharp
[FeatureGate("BetaFeatures")]
public class BetaController : Controller
{
    // Entire controller only accessible if BetaFeatures is enabled
    
    public IActionResult Index()
    {
        return View();
    }
}
```

---

## Feature Flag Declaration

### Configuration Structure

Feature flags have two parts:
1. **Name**: Unique identifier
2. **Filters**: List of rules to evaluate state

### JSON Configuration Format

#### Simple On/Off Flags

```json
{
  "FeatureManagement": {
    "FeatureA": true,      // Always on
    "FeatureB": false      // Always off
  }
}
```

#### Percentage Rollout

```json
{
  "FeatureManagement": {
    "NewCheckout": {
      "EnabledFor": [
        {
          "Name": "Percentage",
          "Parameters": {
            "Value": 50    // Enable for 50% of users
          }
        }
      ]
    }
  }
}
```

#### Time Window Filter

```json
{
  "FeatureManagement": {
    "ChristmasSale": {
      "EnabledFor": [
        {
          "Name": "TimeWindow",
          "Parameters": {
            "Start": "2024-12-20T00:00:00Z",
            "End": "2024-12-26T23:59:59Z"
          }
        }
      ]
    }
  }
}
```

#### Targeting Filter (Specific Users/Groups)

```json
{
  "FeatureManagement": {
    "PremiumFeatures": {
      "EnabledFor": [
        {
          "Name": "Targeting",
          "Parameters": {
            "Audience": {
              "Users": [
                "user1@contoso.com",
                "user2@contoso.com"
              ],
              "Groups": [
                {
                  "Name": "PremiumUsers",
                  "RolloutPercentage": 100
                },
                {
                  "Name": "BetaTesters",
                  "RolloutPercentage": 50
                }
              ],
              "DefaultRolloutPercentage": 0
            }
          }
        }
      ]
    }
  }
}
```

#### Multiple Filters (OR Logic)

When multiple filters are present, they are evaluated in order. **First filter that returns true enables the feature**.

```json
{
  "FeatureManagement": {
    "NewFeature": {
      "EnabledFor": [
        {
          "Name": "Targeting",
          "Parameters": {
            "Audience": {
              "Users": ["admin@contoso.com"]
            }
          }
        },
        {
          "Name": "Percentage",
          "Parameters": {
            "Value": 25
          }
        }
      ]
    }
  }
}
```

**Evaluation logic**:
1. Check if user is `admin@contoso.com` → If yes, enable (stop)
2. Check if user is in 25% rollout → If yes, enable (stop)
3. Otherwise, disable

---

## Feature Flag Repository

### Why Externalize Feature Flags?

**Problems with hard-coded flags**:
- ❌ Require code changes to modify
- ❌ Require redeployment
- ❌ Cannot change at runtime
- ❌ No centralized management

**Benefits of external repository**:
- ✅ Change states without redeployment
- ✅ Real-time feature control
- ✅ Centralized management
- ✅ Audit trail of changes
- ✅ Different states per environment

### Azure App Configuration as Feature Flag Repository

Azure App Configuration is **designed specifically** for feature flag management:

**Features**:
- Centralized repository for all feature flags
- Dedicated UI for feature management
- Real-time state changes
- Multiple filter types supported
- Environment-specific flags (using labels)
- Integration with .NET, Java Spring, and other frameworks
- REST API for custom implementations

**Setup in App Configuration**:
```bash
# Create feature flag
az appconfig feature set \
  --name myappconfig \
  --feature NewCheckout \
  --label Production

# Enable feature flag
az appconfig feature enable \
  --name myappconfig \
  --feature NewCheckout \
  --label Production

# Disable feature flag
az appconfig feature disable \
  --name myappconfig \
  --feature NewCheckout \
  --label Production

# Set percentage filter
az appconfig feature filter add \
  --name myappconfig \
  --feature NewCheckout \
  --filter-name Microsoft.Percentage \
  --filter-parameters Value=50
```

---

## Practical Implementation

### Complete .NET Example

#### 1. Install NuGet Packages

```bash
dotnet add package Microsoft.FeatureManagement.AspNetCore
dotnet add package Microsoft.Extensions.Configuration.AzureAppConfiguration
```

#### 2. Configure Application

**Program.cs**:
```csharp
using Microsoft.FeatureManagement;

var builder = WebApplication.CreateBuilder(args);

// Add Azure App Configuration
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(Environment.GetEnvironmentVariable("APP_CONFIG_CONNECTION_STRING"))
           .UseFeatureFlags(featureFlagOptions =>
           {
               featureFlagOptions.CacheExpirationInterval = TimeSpan.FromMinutes(5);
           });
});

// Add feature management
builder.Services.AddFeatureManagement();

var app = builder.Build();

// Use Azure App Configuration middleware (for dynamic refresh)
app.UseAzureAppConfiguration();

app.MapGet("/checkout", async (IFeatureManager featureManager) =>
{
    if (await featureManager.IsEnabledAsync("NewCheckout"))
    {
        return Results.Ok(new { message = "New checkout process" });
    }
    else
    {
        return Results.Ok(new { message = "Old checkout process" });
    }
});

app.Run();
```

#### 3. Configure App Configuration

**Add to App Configuration**:
```json
{
  "FeatureManagement": {
    "NewCheckout": {
      "EnabledFor": [
        {
          "Name": "Microsoft.Percentage",
          "Parameters": {
            "Value": 25
          }
        }
      ]
    },
    "BetaFeatures": true,
    "MaintenanceMode": false
  }
}
```

#### 4. Use in Controllers

```csharp
[ApiController]
[Route("api/[controller]")]
public class FeaturesController : ControllerBase
{
    private readonly IFeatureManager _featureManager;

    public FeaturesController(IFeatureManager featureManager)
    {
        _featureManager = featureManager;
    }

    [HttpGet("check/{featureName}")]
    public async Task<IActionResult> CheckFeature(string featureName)
    {
        bool isEnabled = await _featureManager.IsEnabledAsync(featureName);
        return Ok(new { feature = featureName, enabled = isEnabled });
    }

    [HttpGet("beta")]
    [FeatureGate("BetaFeatures")]
    public IActionResult BetaEndpoint()
    {
        return Ok(new { message = "This is a beta feature" });
    }
}
```

---

## Common Use Cases

### 1. Gradual Rollout (Canary Release)

**Scenario**: New payment processing feature

**Implementation**:
```json
{
  "FeatureManagement": {
    "NewPaymentProcessor": {
      "EnabledFor": [
        {
          "Name": "Percentage",
          "Parameters": { "Value": 10 }
        }
      ]
    }
  }
}
```

**Rollout Plan**:
- Day 1: 10% of users
- Day 2: Monitor error rates and performance
- Day 3: Increase to 50% if metrics are good
- Day 5: Increase to 100%

**Code**:
```csharp
if (await _featureManager.IsEnabledAsync("NewPaymentProcessor"))
{
    await _newPaymentService.ProcessPayment(order);
}
else
{
    await _legacyPaymentService.ProcessPayment(order);
}
```

### 2. Beta Testing / Early Access

**Scenario**: Premium users get early access to features

**Implementation**:
```json
{
  "FeatureManagement": {
    "AdvancedReporting": {
      "EnabledFor": [
        {
          "Name": "Targeting",
          "Parameters": {
            "Audience": {
              "Groups": [
                {
                  "Name": "PremiumUsers",
                  "RolloutPercentage": 100
                }
              ]
            }
          }
        }
      ]
    }
  }
}
```

### 3. A/B Testing

**Scenario**: Test two different UI layouts

**Implementation**:
```json
{
  "FeatureManagement": {
    "LayoutVariantA": {
      "EnabledFor": [
        {
          "Name": "Percentage",
          "Parameters": { "Value": 50 }
        }
      ]
    }
  }
}
```

**Code**:
```csharp
string layout = await _featureManager.IsEnabledAsync("LayoutVariantA") 
    ? "variant-a" 
    : "variant-b";

// Track which variant user sees
_analytics.TrackVariant(userId, layout);
```

### 4. Kill Switch / Circuit Breaker

**Scenario**: Instantly disable problematic feature

**Implementation**:
```json
{
  "FeatureManagement": {
    "ExternalApiIntegration": true
  }
}
```

**Usage**:
```csharp
if (await _featureManager.IsEnabledAsync("ExternalApiIntegration"))
{
    try
    {
        await _externalApi.CallAsync();
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "External API failed");
        // If errors spike, disable feature in App Configuration
        // No code deployment needed
    }
}
else
{
    // Fallback behavior
    await _fallbackService.HandleAsync();
}
```

**In case of emergency**:
```bash
# Instantly disable feature
az appconfig feature disable --name myappconfig --feature ExternalApiIntegration
```

### 5. Scheduled Feature Release

**Scenario**: Feature goes live at specific time

**Implementation**:
```json
{
  "FeatureManagement": {
    "BlackFridaySale": {
      "EnabledFor": [
        {
          "Name": "TimeWindow",
          "Parameters": {
            "Start": "2024-11-29T00:00:00Z",
            "End": "2024-11-30T23:59:59Z"
          }
        }
      ]
    }
  }
}
```

---

## Best Practices

### 1. **Name Feature Flags Descriptively**

✅ **Good**:
```
NewCheckoutFlow
EnhancedSearchAlgorithm
PremiumReporting
```

❌ **Bad**:
```
Feature1
NewFeature
Test
```

### 2. **Start with Small Percentage**

```
10% → 25% → 50% → 100%
```

Monitor metrics at each stage before increasing.

### 3. **Remove Old Flags**

Feature flags are **temporary**. Remove them after:
- Feature is fully rolled out
- Old code path is removed
- No longer needed

**Technical debt**: Too many feature flags make code hard to maintain.

### 4. **Use Targeting for Internal Testing**

```json
{
  "FeatureManagement": {
    "ExperimentalFeature": {
      "EnabledFor": [
        {
          "Name": "Targeting",
          "Parameters": {
            "Audience": {
              "Users": ["dev-team@contoso.com"],
              "DefaultRolloutPercentage": 0
            }
          }
        }
      ]
    }
  }
}
```

### 5. **Log Feature Flag Decisions**

```csharp
bool isEnabled = await _featureManager.IsEnabledAsync("NewFeature");
_logger.LogInformation("Feature {FeatureName} evaluated to {IsEnabled} for user {UserId}",
    "NewFeature", isEnabled, userId);
```

### 6. **Environment-Specific Flags**

Use labels in App Configuration:

```bash
# Development: Enable all features
az appconfig feature enable --name myappconfig --feature NewFeature --label Development

# Production: Gradual rollout
az appconfig feature filter add \
  --name myappconfig \
  --feature NewFeature \
  --label Production \
  --filter-name Microsoft.Percentage \
  --filter-parameters Value=10
```

---

## Exam Tips

### Key Concepts for AZ-204

1. **Feature flags decouple deployment from release**: Code can be deployed with feature disabled

2. **Feature Manager**: Handles feature flag lifecycle (load, evaluate, cache, update)

3. **Filters**: Rules for evaluating feature state (Percentage, Targeting, Time Window)

4. **Azure App Configuration**: Centralized repository for feature flags

5. **Multiple filters = OR logic**: First filter that returns true enables the feature

6. **Percentage filter**: Enable for X% of users (gradual rollout)

7. **Targeting filter**: Enable for specific users or groups

8. **Time Window filter**: Enable during specific dates/times

9. **Feature flags are temporary**: Should be removed after full rollout

10. **Dynamic refresh**: Applications can detect flag changes without restart

11. **.NET library**: `Microsoft.FeatureManagement.AspNetCore`

12. **FeatureGate attribute**: Apply feature flag to entire controller or action

### Common Exam Scenarios

**Scenario 1**: "Release new feature to 10% of users initially"
→ **Answer**: Use feature flag with Percentage filter (Value: 10)

**Scenario 2**: "Enable feature for beta testers only"
→ **Answer**: Use Targeting filter with specific users/groups

**Scenario 3**: "Need to instantly disable problematic feature"
→ **Answer**: Use feature flag as kill switch, disable in App Configuration

**Scenario 4**: "Test two different checkout flows"
→ **Answer**: Use feature flag with Percentage filter (50%) for A/B testing

**Scenario 5**: "Enable sale feature only during Black Friday"
→ **Answer**: Use Time Window filter with Start and End dates

---

## Quick Reference

### Azure CLI Commands

```bash
# Create feature flag
az appconfig feature set --name <appconfig-name> --feature <feature-name>

# Enable feature
az appconfig feature enable --name <appconfig-name> --feature <feature-name>

# Disable feature
az appconfig feature disable --name <appconfig-name> --feature <feature-name>

# Add percentage filter
az appconfig feature filter add \
  --name <appconfig-name> \
  --feature <feature-name> \
  --filter-name Microsoft.Percentage \
  --filter-parameters Value=50

# List all features
az appconfig feature list --name <appconfig-name>

# Delete feature
az appconfig feature delete --name <appconfig-name> --feature <feature-name>
```

### .NET Code Snippets

```csharp
// Check if feature is enabled
bool isEnabled = await _featureManager.IsEnabledAsync("FeatureName");

// Feature gate attribute
[FeatureGate("FeatureName")]
public IActionResult MyAction() { }

// Conditional execution
if (await _featureManager.IsEnabledAsync("NewFeature"))
{
    // New code
}
else
{
    // Old code
}

// In Razor view
@inject IFeatureManager FeatureManager
@if (await FeatureManager.IsEnabledAsync("NewUI"))
{
    <!-- New UI -->
}
```

---

## Learn More

- [Feature Management Overview](https://docs.microsoft.com/azure/azure-app-configuration/concept-feature-management)
- [Use feature flags in ASP.NET Core](https://docs.microsoft.com/azure/azure-app-configuration/use-feature-flags-dotnet-core)
- [Feature Management Best Practices](https://docs.microsoft.com/azure/azure-app-configuration/howto-best-practices#feature-flag-best-practices)
