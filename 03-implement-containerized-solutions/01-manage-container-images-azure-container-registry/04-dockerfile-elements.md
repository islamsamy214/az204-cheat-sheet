# Dockerfile Elements

## Key Concepts
- **Dockerfile** - Script with instructions to build Docker image
- **Layered architecture** - Each instruction creates a layer
- **Base image** - Starting point (FROM instruction)
- **Build context** - Files sent to Docker daemon for build

## What Is a Dockerfile?

**Text file with instructions** to build a container image:

- Series of commands to assemble image
- Defines base image, dependencies, app code
- Each instruction creates a new layer
- Layers are cached for faster builds

## Common Dockerfile Instructions

### Essential Instructions

| Instruction | Purpose | Example |
|-------------|---------|---------|
| `FROM` | Set base image | `FROM node:18` |
| `WORKDIR` | Set working directory | `WORKDIR /app` |
| `COPY` | Copy files to image | `COPY . /app` |
| `ADD` | Copy + extract archives | `ADD app.tar.gz /app` |
| `RUN` | Execute command during build | `RUN npm install` |
| `CMD` | Default command to run | `CMD ["node", "app.js"]` |
| `ENTRYPOINT` | Main executable | `ENTRYPOINT ["python"]` |
| `EXPOSE` | Document port | `EXPOSE 8080` |
| `ENV` | Set environment variable | `ENV NODE_ENV=production` |
| `ARG` | Build-time variable | `ARG VERSION=1.0` |
| `LABEL` | Add metadata | `LABEL version="1.0"` |
| `USER` | Set user for RUN/CMD | `USER appuser` |
| `VOLUME` | Create mount point | `VOLUME /data` |
| `HEALTHCHECK` | Container health check | `HEALTHCHECK CMD curl` |

## Example Dockerfile (.NET)

### Basic .NET Application
```dockerfile
# Use the .NET 6 runtime as a base image
FROM mcr.microsoft.com/dotnet/runtime:6.0

# Set the working directory to /app
WORKDIR /app

# Copy the contents of the published app to the container's /app directory
COPY bin/Release/net6.0/publish/ .

# Document that the application listens on port 80 (does not publish it)
EXPOSE 80

# Set the command to run when the container starts
CMD ["dotnet", "MyApp.dll"]
```

**Line-by-line explanation**:
1. `FROM mcr.microsoft.com/dotnet/runtime:6.0` - Start with .NET 6 runtime base image
2. `WORKDIR /app` - Create and set `/app` as working directory
3. `COPY bin/Release/net6.0/publish/ .` - Copy published app files to `/app`
4. `EXPOSE 80` - Document that app listens on port 80
5. `CMD ["dotnet", "MyApp.dll"]` - Run app when container starts

⚠️ **EXPOSE** does not publish the port - use `docker run -p` to map ports

## Detailed Instruction Examples

### FROM - Base Image
```dockerfile
# Official image from Docker Hub
FROM node:18

# Microsoft image from MCR
FROM mcr.microsoft.com/dotnet/aspnet:6.0

# Multi-stage build (multiple FROM)
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
# ...
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS runtime
```

### WORKDIR - Set Working Directory
```dockerfile
# Create and switch to /app
WORKDIR /app

# Subsequent commands run from /app
COPY package.json .
RUN npm install

# Equivalent to: COPY package.json /app
```

### COPY vs ADD
```dockerfile
# COPY - Simple file copy (preferred)
COPY package.json /app/
COPY src/ /app/src/

# ADD - Copy + auto-extract tar files
ADD archive.tar.gz /app/

# ADD - Download from URL (not recommended)
ADD https://example.com/file.txt /app/
```

💡 **Best Practice**: Use `COPY` unless you need `ADD` features

### RUN - Execute Commands
```dockerfile
# Install packages (Linux)
RUN apt-get update && apt-get install -y curl

# Install dependencies (Node.js)
RUN npm install

# Multiple commands in one layer
RUN apt-get update && \
    apt-get install -y curl git && \
    apt-get clean

# Create user
RUN adduser --disabled-password --gecos '' appuser
```

### CMD vs ENTRYPOINT

#### CMD - Default Command
```dockerfile
# Exec form (preferred)
CMD ["node", "server.js"]

# Can be overridden
docker run myimage python app.py  # Runs python instead
```

#### ENTRYPOINT - Fixed Executable
```dockerfile
# Fixed entry point
ENTRYPOINT ["python", "app.py"]

# Cannot be overridden (only args can)
docker run myimage --debug  # Runs: python app.py --debug
```

#### Combined ENTRYPOINT + CMD
```dockerfile
# Fixed command with default args
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8080"]

# Default: python app.py --port 8080
# Override args: docker run myimage --port 9000
```

### ENV - Environment Variables
```dockerfile
# Set environment variables
ENV NODE_ENV=production
ENV PORT=8080
ENV DATABASE_URL=postgres://localhost/db

# Use in subsequent commands
RUN echo "Environment: $NODE_ENV"
```

### ARG - Build Arguments
```dockerfile
# Define build argument with default
ARG VERSION=1.0.0
ARG BUILD_DATE

# Use in Dockerfile
LABEL version=$VERSION
RUN echo "Building version $VERSION"

# Set at build time
docker build --build-arg VERSION=2.0.0 .
```

**Difference: ARG vs ENV**
- `ARG` - Available during build only
- `ENV` - Available during build AND runtime

### EXPOSE - Document Ports
```dockerfile
# Single port
EXPOSE 8080

# Multiple ports
EXPOSE 8080 8443

# Port + protocol
EXPOSE 3000/tcp
EXPOSE 53/udp
```

⚠️ **Important**: Does not actually publish ports - only documentation

```bash
# Publish port at runtime
docker run -p 8080:8080 myimage
docker run -p 9000:8080 myimage  # Host 9000 → Container 8080
```

### USER - Set User
```dockerfile
# Run as non-root user
RUN adduser --disabled-password appuser
USER appuser

# All subsequent commands run as appuser
RUN whoami  # Returns: appuser
CMD ["./app"]  # Runs as appuser
```

### VOLUME - Mount Points
```dockerfile
# Create mount point for persistent data
VOLUME /data

# Multiple volumes
VOLUME ["/data", "/logs"]

# Use at runtime
docker run -v /host/path:/data myimage
```

## Multi-Stage Builds

### Why Multi-Stage?
✅ **Smaller images** - Don't include build tools in final image
✅ **Separate build/runtime** - Build in SDK, run in runtime
✅ **Security** - Fewer tools = smaller attack surface

### Example: .NET Multi-Stage Build
```dockerfile
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["MyApp.csproj", "./"]
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

# Stage 2: Runtime
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

**Result**: Final image contains only runtime + published app (no SDK)

### Example: Node.js Multi-Stage Build
```dockerfile
# Stage 1: Build
FROM node:18 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

## Layer Caching

### How Caching Works
1. Docker caches each layer
2. If instruction unchanged, reuse cached layer
3. First changed instruction invalidates cache for all subsequent layers

### Optimization: Order Instructions by Change Frequency
```dockerfile
# ❌ BAD - Changes to source invalidate npm install cache
FROM node:18
WORKDIR /app
COPY . .              # Changes frequently
RUN npm install       # Must rebuild every time

# ✅ GOOD - Cache npm install layer
FROM node:18
WORKDIR /app
COPY package*.json ./ # Changes rarely
RUN npm install       # Cached unless package.json changes
COPY . .              # Changes frequently
```

## Best Practices

### 1. Use Specific Base Image Tags
```dockerfile
# ❌ Avoid: Latest tag
FROM node:latest

# ✅ Use: Specific version
FROM node:18.15.0-alpine
```

### 2. Minimize Layers
```dockerfile
# ❌ Multiple layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# ✅ Single layer
RUN apt-get update && \
    apt-get install -y curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### 3. Use .dockerignore
```
# .dockerignore
node_modules
.git
*.log
.env
.vscode
```

### 4. Run as Non-Root User
```dockerfile
RUN adduser --disabled-password appuser
USER appuser
```

### 5. Multi-Stage Builds
```dockerfile
# Build stage with all tools
FROM node:18 AS build
# ...

# Production stage with minimal footprint
FROM node:18-alpine AS production
COPY --from=build /app/dist .
```

### 6. Use Alpine Images
```dockerfile
# Standard: ~900 MB
FROM node:18

# Alpine: ~100 MB
FROM node:18-alpine
```

### 7. Label Images
```dockerfile
LABEL maintainer="team@example.com"
LABEL version="1.0.0"
LABEL description="My application"
```

## Common Dockerfile Patterns

### Node.js Application
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
USER node
CMD ["node", "server.js"]
```

### Python Application
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
USER nobody
CMD ["python", "app.py"]
```

### Java Spring Boot
```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## Build Commands

### Basic Build
```bash
# Build image from Dockerfile
docker build -t myapp:v1.0 .

# Build with specific Dockerfile
docker build -t myapp:v1.0 -f Dockerfile.prod .

# Build with build args
docker build --build-arg VERSION=1.0.0 -t myapp:v1.0 .
```

### ACR Build
```bash
# Build in Azure
az acr build \
  --registry myregistry \
  --image myapp:v1.0 \
  --file Dockerfile \
  .
```

## Critical Notes
- 💡 **Dockerfile** - Text script with build instructions
- ⚠️ **Layers** - Each instruction creates a layer (cached for speed)
- 🎯 **FROM** - Required first instruction (base image)
- ✅ **Multi-stage** - Build in SDK, run in runtime (smaller images)
- 📊 **Order matters** - Instructions ordered by change frequency
- 🔄 **COPY preferred** - Use COPY instead of ADD (unless extracting)
- 🔒 **Non-root user** - Run as non-root for security
- ⚠️ **EXPOSE** - Documentation only, doesn't publish ports

## Exam Tips
- Dockerfile: Script to build container image
- FROM: First instruction, sets base image
- WORKDIR: Sets working directory (creates if doesn't exist)
- COPY: Copy files to image (preferred over ADD)
- ADD: Copy + extract tar files (use COPY for simple copies)
- RUN: Execute command during build (each creates layer)
- CMD: Default command (can be overridden at runtime)
- ENTRYPOINT: Main executable (args can be overridden)
- EXPOSE: Document port (doesn't publish - use `docker run -p`)
- ENV: Environment variable (available during build and runtime)
- ARG: Build argument (only during build)
- USER: Set user for subsequent RUN/CMD commands
- Multi-stage builds: Separate build and runtime stages (smaller images)
- Layer caching: Instructions ordered by change frequency
- Best practices: Specific tags, minimize layers, .dockerignore, non-root user, alpine images
- Build command: `docker build -t <name>:<tag> .`
- ACR build: `az acr build --registry <name> --image <image> .`

[Learn More](https://learn.microsoft.com/en-us/training/modules/publish-container-image-to-azure-container-registry/5-dockerfile-components)
