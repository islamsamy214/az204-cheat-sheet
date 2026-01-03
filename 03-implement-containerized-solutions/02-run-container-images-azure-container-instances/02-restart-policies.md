# Container Restart Policies

## Key Concepts
- **Restart policy** - Controls container restart behavior
- **Always** - Restart on any exit (default)
- **Never** - Never restart (run once)
- **OnFailure** - Restart only on failure (non-zero exit)

## Restart Policy Overview

**Controls what happens when container exits**:

- Determines if container should restart
- Affects billing (stopped containers don't incur charges)
- Perfect for batch jobs and tasks
- Per-second billing means you pay only while running

## Restart Policy Options

| Policy | Behavior | Use Case | Status After Exit |
|--------|----------|----------|-------------------|
| **Always** | Always restart on exit | Long-running services, web apps | Running |
| **Never** | Never restart, run once | One-time tasks, batch jobs | Terminated |
| **OnFailure** | Restart on non-zero exit | Tasks that may fail and retry | Running or Terminated |

### Always (Default)
```bash
# Containers always restart (even on success)
az container create \
  --resource-group myResourceGroup \
  --name web-app \
  --image nginx \
  --restart-policy Always

# Use for:
# - Web servers
# - APIs
# - Long-running services
# - Continuous monitoring
```

**Behavior**:
- Exit code 0 (success) → Restart
- Exit code != 0 (failure) → Restart
- Default policy if not specified

### Never
```bash
# Container never restarts
az container create \
  --resource-group myResourceGroup \
  --name batch-job \
  --image myapp \
  --restart-policy Never

# Use for:
# - One-time batch jobs
# - Data processing tasks
# - Image rendering
# - Build jobs
# - Database migrations
```

**Behavior**:
- Exit code 0 (success) → Terminated
- Exit code != 0 (failure) → Terminated
- Status set to **Terminated** after exit

### OnFailure
```bash
# Container restarts only on failure
az container create \
  --resource-group myResourceGroup \
  --name retry-task \
  --image myapp \
  --restart-policy OnFailure

# Use for:
# - Tasks that may fail temporarily
# - Network-dependent operations
# - API calls with retry logic
# - Data sync tasks
```

**Behavior**:
- Exit code 0 (success) → Terminated
- Exit code != 0 (failure) → Restart and try again
- Runs at least once

## Run-to-Completion Tasks

### Perfect for Batch Jobs

**Benefits**:
- Start quickly (seconds)
- Run task to completion
- Pay only for compute time used
- Automatically terminated when done

### Example: Data Processing
```bash
# Process data file and exit
az container create \
  --resource-group myResourceGroup \
  --name data-processor \
  --image myprocessor:v1.0 \
  --restart-policy Never \
  --environment-variables \
    'INPUT_FILE'='data.csv' \
    'OUTPUT_FILE'='results.json' \
  --azure-file-volume-account-name mystorage \
  --azure-file-volume-account-key <key> \
  --azure-file-volume-share-name data \
  --azure-file-volume-mount-path /data

# Container runs once and stops
# Status becomes "Terminated"
```

### Example: Build Job
```bash
# Build container image
az container create \
  --resource-group myResourceGroup \
  --name build-job \
  --image docker:dind \
  --restart-policy OnFailure \
  --command-line "docker build -t myapp:latest ." \
  --azure-file-volume-mount-path /workspace

# If build fails → Restart
# If build succeeds → Terminate
```

## Container Status

### Status Lifecycle

```
Create → Waiting → Running → Succeeded/Failed → Terminated
                              ↓ (OnFailure)
                            Restart
```

### Check Status
```bash
# View container status
az container show \
  --resource-group myResourceGroup \
  --name mycontainer \
  --query instanceView.state

# Possible states:
# - Waiting
# - Running
# - Succeeded (exit code 0)
# - Failed (exit code != 0)
# - Terminated
```

### View Logs After Termination
```bash
# Even after container stops, logs are available
az container logs \
  --resource-group myResourceGroup \
  --name mycontainer

# View last exit code
az container show \
  --resource-group myResourceGroup \
  --name mycontainer \
  --query "instanceView.currentState.exitCode"
```

## Practical Examples

### Example 1: Web Server (Always)
```bash
# NGINX web server - always running
az container create \
  --resource-group myResourceGroup \
  --name nginx-server \
  --image nginx:alpine \
  --dns-name-label mywebsite \
  --ports 80 \
  --restart-policy Always

# Stays running indefinitely
# Restarts if crashes
```

### Example 2: Nightly Backup (Never)
```bash
# Backup job runs once per schedule
az container create \
  --resource-group myResourceGroup \
  --name backup-job \
  --image backup-tool:latest \
  --restart-policy Never \
  --environment-variables \
    'SOURCE'='/data' \
    'DESTINATION'='https://backup.blob.core.windows.net' \
  --azure-file-volume-mount-path /data

# Runs once, then stops
# Triggered by external scheduler (Logic Apps, Azure Automation)
```

### Example 3: Message Processor (OnFailure)
```bash
# Process queue messages with retry
az container create \
  --resource-group myResourceGroup \
  --name message-processor \
  --image processor:v1.0 \
  --restart-policy OnFailure \
  --environment-variables \
    'QUEUE_NAME'='messages' \
    'CONNECTION_STRING'='...'

# Success: Process messages → Exit 0 → Terminate
# Failure: Network error → Exit 1 → Restart
```

### Example 4: Image Rendering (OnFailure)
```bash
# Render video frame
az container create \
  --resource-group myResourceGroup \
  --name renderer \
  --image renderer:latest \
  --restart-policy OnFailure \
  --gpu-count 1 \
  --gpu-sku K80 \
  --environment-variables \
    'FRAME'='42' \
    'SCENE'='explosion'

# Render success → Exit 0 → Terminate
# Render failure → Exit 1 → Restart
```

## YAML Configuration

### Always
```yaml
apiVersion: '2021-09-01'
location: eastus
name: always-running
properties:
  containers:
  - name: webapp
    properties:
      image: nginx
      ports:
      - port: 80
      resources:
        requests:
          cpu: 1
          memoryInGB: 1.5
  osType: Linux
  restartPolicy: Always
```

### Never
```yaml
apiVersion: '2021-09-01'
location: eastus
name: one-time-task
properties:
  containers:
  - name: batch-job
    properties:
      image: batch-processor:latest
      resources:
        requests:
          cpu: 2
          memoryInGB: 4
  osType: Linux
  restartPolicy: Never
```

### OnFailure
```yaml
apiVersion: '2021-09-01'
location: eastus
name: retry-on-failure
properties:
  containers:
  - name: data-sync
    properties:
      image: sync-tool:v1.0
      environmentVariables:
      - name: RETRY_COUNT
        value: '3'
      resources:
        requests:
          cpu: 1
          memoryInGB: 2
  osType: Linux
  restartPolicy: OnFailure
```

## Exit Codes

### Standard Exit Codes
| Exit Code | Meaning | Restart with OnFailure? |
|-----------|---------|-------------------------|
| **0** | Success | No (Terminated) |
| **1** | General error | Yes |
| **2** | Misuse | Yes |
| **126** | Cannot execute | Yes |
| **127** | Command not found | Yes |
| **137** | SIGKILL (OOM) | Yes |
| **139** | Segmentation fault | Yes |

### Setting Exit Codes in Your App
```bash
# Shell script
#!/bin/bash
if [ -f "/data/input.txt" ]; then
    # Process file
    process_file
    exit 0  # Success
else
    echo "Input file not found"
    exit 1  # Failure - will trigger restart with OnFailure
fi
```

```python
# Python
import sys

try:
    process_data()
    sys.exit(0)  # Success
except Exception as e:
    print(f"Error: {e}")
    sys.exit(1)  # Failure - will trigger restart with OnFailure
```

## Cost Implications

### Billing by Restart Policy

| Policy | Running Time | Cost Pattern |
|--------|--------------|--------------|
| **Always** | Continuous | Constant (until deleted) |
| **Never** | Once | Single execution cost |
| **OnFailure** | Until success | Variable (depends on failures) |

### Example Cost Calculation
```
Container: 1 CPU, 2 GB memory
Region: East US
CPU: $0.0000012/core/second
Memory: $0.0000001/GB/second

Always (24 hours):
= 86,400 seconds
= (1 × $0.0000012 × 86,400) + (2 × $0.0000001 × 86,400)
= $0.12/day

Never (5 minutes):
= 300 seconds
= (1 × $0.0000012 × 300) + (2 × $0.0000001 × 300)
= $0.0004 per run

OnFailure (3 retries, 2 min each):
= 360 seconds total
= (1 × $0.0000012 × 360) + (2 × $0.0000001 × 360)
= $0.0005 per task
```

## Best Practices

### 1. Choose Right Policy
```bash
# Long-running services
--restart-policy Always

# Batch jobs, migrations
--restart-policy Never

# Tasks with transient failures
--restart-policy OnFailure
```

### 2. Implement Proper Exit Codes
```python
# Return correct exit codes
sys.exit(0)  # Success
sys.exit(1)  # Failure
```

### 3. Monitor Container Status
```bash
# Check final status
az container show \
  --resource-group myResourceGroup \
  --name mycontainer \
  --query "instanceView.state"
```

### 4. Use Logs for Debugging
```bash
# View logs even after termination
az container logs \
  --resource-group myResourceGroup \
  --name mycontainer
```

### 5. Set Timeouts
```bash
# Prevent infinite retries with OnFailure
# Implement timeout logic in your application
```

## Critical Notes
- 💡 **Default policy** - Always (restarts on any exit)
- ⚠️ **Never** - Run once, then terminate (perfect for batch jobs)
- 🎯 **OnFailure** - Restart on failure (exit code != 0)
- ✅ **Per-second billing** - Stopped containers don't incur charges
- 📊 **Exit code 0** - Success (no restart with OnFailure)
- 🔄 **Exit code != 0** - Failure (restart with OnFailure)
- 🔒 **Terminated state** - Container stopped, logs still available
- ⏱️ **Run-to-completion** - Perfect for batch jobs, data processing

## Exam Tips
- Restart policies: Always, Never, OnFailure
- Always: Default, restarts on any exit (long-running services)
- Never: Run once, never restart (batch jobs, migrations)
- OnFailure: Restart only on failure (exit code != 0)
- Exit code 0: Success (OnFailure won't restart)
- Exit code != 0: Failure (OnFailure will restart)
- Run-to-completion: Perfect for batch jobs, billed per second
- Status after exit with Never/OnFailure (success): Terminated
- Logs available even after termination
- CLI: `--restart-policy Always|Never|OnFailure`
- YAML: `restartPolicy: Always|Never|OnFailure`
- Billing: Pay only while container is running
- OnFailure runs at least once (even if succeeds first time)

[Learn More](https://learn.microsoft.com/en-us/training/modules/create-run-container-images-azure-container-instances/4-run-containerized-tasks-restart-policies)
