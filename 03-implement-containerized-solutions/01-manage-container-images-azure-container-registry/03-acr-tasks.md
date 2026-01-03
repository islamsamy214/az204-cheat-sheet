# Azure Container Registry Tasks (ACR Tasks)

## Key Concepts
- **ACR Tasks** - Build container images in Azure cloud
- **Quick task** - On-demand single image build (`az acr build`)
- **Automated triggers** - Build on Git commit, base image update, schedule
- **Multi-step tasks** - Complex workflows with multiple steps

## What Are ACR Tasks?

**Cloud-based container image building** without local Docker:

- Build images directly in Azure
- No local Docker Engine needed
- Automated build pipelines
- Multi-platform support (Linux, Windows, ARM)
- Integration with CI/CD

### Benefits
✅ **No local Docker** - Build in cloud
✅ **Automation** - Trigger builds automatically
✅ **CI/CD integration** - Part of development lifecycle
✅ **Platform support** - Linux, Windows, ARM architectures
✅ **Multi-step workflows** - Build, test, push in sequence

## Task Scenarios

### 1. Quick Task
**On-demand build and push** without local Docker:

```bash
# Build image from Dockerfile and push to registry
az acr build \
  --registry myregistry \
  --image myapp:v1.0 \
  --file Dockerfile \
  .

# Think: "docker build + docker push" in the cloud
```

**Use cases**:
- Quick builds without local Docker
- CI/CD pipeline builds
- Validate Dockerfile before commit
- Inner-loop development

### 2. Trigger on Source Code Update
**Automatic builds on Git commit**:

```bash
# Create task triggered by Git commit
az acr task create \
  --registry myregistry \
  --name build-on-commit \
  --image myapp:{{.Run.ID}} \
  --context https://github.com/myorg/myrepo.git \
  --file Dockerfile \
  --git-access-token $(cat token.txt) \
  --commit-trigger-enabled true

# Now: Every commit triggers a build
```

**Triggers**:
- Code commit to branch
- Pull request created/updated
- Specific branch or tag pattern

### 3. Trigger on Base Image Update
**Rebuild app when base image changes**:

```bash
# Create task that watches base image
az acr task create \
  --registry myregistry \
  --name rebuild-on-base \
  --image myapp:latest \
  --context https://github.com/myorg/myrepo.git \
  --file Dockerfile \
  --base-image-trigger-enabled true \
  --base-image-trigger-name mybaseimage

# Automatic rebuild when:
# - Base image updated in ACR
# - Base image updated in Docker Hub
```

**Scenario**:
```dockerfile
# Your Dockerfile
FROM myregistry.azurecr.io/baseimage:latest
COPY . /app
CMD ["./app"]

# When baseimage:latest is updated → automatic rebuild
```

### 4. Schedule a Task
**Run builds on a schedule**:

```bash
# Create scheduled task (cron format)
az acr task create \
  --registry myregistry \
  --name nightly-build \
  --image myapp:nightly-{{.Run.Date}} \
  --context https://github.com/myorg/myrepo.git \
  --file Dockerfile \
  --schedule "0 2 * * *"  # 2 AM daily

# Examples:
# "0 2 * * *"      - Daily at 2 AM
# "0 2 * * 0"      - Weekly on Sunday at 2 AM
# "0 */4 * * *"    - Every 4 hours
# "0 2 1 * *"      - Monthly on 1st at 2 AM
```

**Use cases**:
- Nightly builds
- Regular maintenance tasks
- Scheduled image scanning
- Periodic test runs

### 5. Multi-Step Tasks
**Complex workflows** with build, test, deploy:

```yaml
# acr-task.yaml
version: v1.1.0
steps:
  # Step 1: Build web app image
  - build: -t {{.Run.Registry}}/webapp:{{.Run.ID}} -f Dockerfile .
  
  # Step 2: Run web app container
  - cmd: {{.Run.Registry}}/webapp:{{.Run.ID}}
    id: web
    detach: true
    ports:
      - 8080:80
  
  # Step 3: Build test image
  - build: -t {{.Run.Registry}}/tests:{{.Run.ID}} -f Dockerfile.test .
  
  # Step 4: Run tests against web app
  - cmd: {{.Run.Registry}}/tests:{{.Run.ID}}
    env:
      - WEB_URL=http://web:80
  
  # Step 5: Push if tests pass
  - push:
      - {{.Run.Registry}}/webapp:{{.Run.ID}}
      - {{.Run.Registry}}/webapp:latest
```

```bash
# Create multi-step task
az acr task create \
  --registry myregistry \
  --name multi-step-build \
  --context https://github.com/myorg/myrepo.git \
  --file acr-task.yaml \
  --git-access-token $(cat token.txt)

# Run task manually
az acr task run --registry myregistry --name multi-step-build
```

## Quick Task Examples

### Build from Local Context
```bash
# Build from current directory
az acr build --registry myregistry --image myapp:v1.0 .

# Build with specific Dockerfile
az acr build \
  --registry myregistry \
  --image myapp:v1.0 \
  --file Dockerfile.production \
  .
```

### Build from Git Repository
```bash
# Build from GitHub repo
az acr build \
  --registry myregistry \
  --image myapp:v1.0 \
  https://github.com/myorg/myrepo.git

# Build specific branch
az acr build \
  --registry myregistry \
  --image myapp:v1.0 \
  https://github.com/myorg/myrepo.git#develop

# Build with build args
az acr build \
  --registry myregistry \
  --image myapp:v1.0 \
  --build-arg VERSION=1.0.0 \
  .
```

### Build for Multiple Platforms
```bash
# Build for Linux ARM64
az acr build \
  --registry myregistry \
  --image myapp:v1.0-arm64 \
  --platform Linux/arm64 \
  .

# Build for Windows
az acr build \
  --registry myregistry \
  --image myapp:v1.0-windows \
  --platform Windows/amd64 \
  .
```

## Task Management

### Create Task
```bash
# Basic task
az acr task create \
  --registry myregistry \
  --name build-task \
  --image myapp:{{.Run.ID}} \
  --context https://github.com/myorg/myrepo.git \
  --file Dockerfile \
  --git-access-token <token>
```

### List Tasks
```bash
# List all tasks
az acr task list --registry myregistry --output table

# Show task details
az acr task show \
  --registry myregistry \
  --name build-task \
  --output json
```

### Run Task
```bash
# Run task manually
az acr task run --registry myregistry --name build-task

# Run task with overrides
az acr task run \
  --registry myregistry \
  --name build-task \
  --set VERSION=2.0.0
```

### View Task Runs
```bash
# List task runs
az acr task list-runs \
  --registry myregistry \
  --output table

# Show specific run
az acr task show-run \
  --registry myregistry \
  --run-id <run-id>

# Get run logs
az acr task logs \
  --registry myregistry \
  --run-id <run-id>
```

### Update Task
```bash
# Update task image
az acr task update \
  --registry myregistry \
  --name build-task \
  --image myapp:{{.Run.Date}}-{{.Run.ID}}

# Update trigger settings
az acr task update \
  --registry myregistry \
  --name build-task \
  --commit-trigger-enabled false
```

### Delete Task
```bash
# Delete task
az acr task delete \
  --registry myregistry \
  --name build-task
```

## Platform Support

### Supported Platforms

| OS | Architectures |
|-----|---------------|
| **Linux** | amd64, arm, arm64, 386 |
| **Windows** | amd64 |

### Platform Specification
```bash
# Linux AMD64 (default)
--platform Linux/amd64

# Linux ARM
--platform Linux/arm

# Linux ARM64 with variant
--platform Linux/arm64/v8

# Windows
--platform Windows/amd64
```

## Task Variables

### Built-in Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `{{.Run.ID}}` | Unique run ID | `ca1` |
| `{{.Run.Date}}` | Run date (YYYYMMDD) | `20260103` |
| `{{.Run.Registry}}` | Registry name | `myregistry.azurecr.io` |
| `{{.Run.Commit}}` | Git commit SHA | `abc123...` |
| `{{.Run.Branch}}` | Git branch | `main` |

### Usage in Tasks
```bash
# Use variables in image tags
az acr task create \
  --registry myregistry \
  --name build-task \
  --image myapp:{{.Run.Date}}-{{.Run.ID}} \
  --context https://github.com/myorg/myrepo.git
```

## Integration Examples

### Azure DevOps Pipeline
```yaml
# azure-pipelines.yml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: AzureCLI@2
  inputs:
    azureSubscription: 'MyAzureSubscription'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      az acr build \
        --registry myregistry \
        --image myapp:$(Build.BuildId) \
        .
```

### GitHub Actions
```yaml
# .github/workflows/build.yml
name: Build Container Image
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      
      - name: Build and Push
        run: |
          az acr build \
            --registry myregistry \
            --image myapp:${{ github.sha }} \
            .
```

## Critical Notes
- 💡 **No Docker needed** - Build images entirely in Azure cloud
- ⚠️ **Quick task** - `az acr build` = docker build + push in cloud
- 🎯 **Automated triggers** - Git commit, base image update, schedule
- ✅ **Multi-step tasks** - Build, test, push in YAML workflow
- 📊 **Platform support** - Linux (amd64, arm, arm64), Windows (amd64)
- 🔄 **CI/CD integration** - Works with Azure Pipelines, GitHub Actions, Jenkins
- 🔒 **Base image tracking** - Auto-rebuild when base image updates
- ⏱️ **Scheduled tasks** - Cron syntax for recurring builds

## Exam Tips
- ACR Tasks: Build container images in Azure without local Docker
- Quick task: `az acr build` - single image build and push
- Task scenarios: Quick, Git commit, base image update, scheduled, multi-step
- Triggers: Source code commit, base image update, schedule (cron)
- Multi-step tasks: YAML file with build, test, push steps
- Platform support: Linux (amd64, arm, arm64, 386), Windows (amd64)
- Platform syntax: `--platform OS/architecture` or `OS/architecture/variant`
- Task variables: {{.Run.ID}}, {{.Run.Date}}, {{.Run.Registry}}, etc.
- Base image tracking: Auto-rebuild when base image in ACR or Docker Hub updates
- Schedule format: Cron syntax (e.g., "0 2 * * *" for 2 AM daily)
- Git triggers: Commit, pull request, specific branch/tag
- Quick build from Git: `az acr build --registry <name> --image <image> <git-url>`
- Task management: create, list, run, show, update, delete, logs

[Learn More](https://learn.microsoft.com/en-us/training/modules/publish-container-image-to-azure-container-registry/4-azure-container-registry-tasks)
