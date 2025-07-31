# Remote Python Debugging in OpenShift/Kubernetes

## Overview
This guide shows how to set up remote debugging for `Python` applications deployed in `OpenShift` clusters using 
`VSCode`. The solution enables real-time debugging of containerized `Python` applications without the traditional 
build-deploy-debug cycle.

## Prerequisites
- `Python` application using `Poetry` for dependency management
- `OpenShift` cluster access
- `VSCode` with `Python` extension
- Container registry access

## Benefits
- **Zero rebuild cycles** for debugging
- **Real-time variable inspection**
- **Step-through debugging** in live containers
- **Multiple client support** (team debugging)
- **Production-safe** with environment controls

## Step 1: Prepare Your Python Application for Debugging

### 1.1 Add Debug Dependencies
Add `debugpy` to your `pyproject.toml`:
```toml 
[tool.poetry.dependencies] 
    python = "^3.9" 
    debugpy = "^1.8.0" # Add this for debugging support
    # ... your other dependencies
``` 

### 1.2 Create Debug Entry Point
Create a `debug_main.py` file in your project root:

```python
import os
import sys
import debugpy
from enum import Enum
from your_app import main as app  # Replace with your actual main module


LOCALHOST: str = '127.0.0.1'


class Env:
    class Port:
        NUMBER: str = '5678'
        DEFAULT: str = NUMBER
        ENV: str = 'DEBUG_PORT'
    
        @classmethod
        def get(cls) -> int:
            env = os.getenv(cls.ENV, cls.DEFAULT)
            return int(env)
    
    class Debug:
        ENABLED: str = 'true'
        DISABLED: str = 'false'
        DEFAULT: str = DISABLED
        ENV: str = 'ENABLE_DEBUG'
    
        @classmethod
        def get(cls) -> str:
            env = os.getenv(cls.ENV, cls.DEFAULT)
            return env.lower()
    
        @classmethod
        def enabled(cls) -> bool:
            return cls.get() == cls.ENABLED
    
        @classmethod
        def disabled(cls) -> bool:
            return not cls.enabled()

    class Wait:
        ENABLED: str = 'true'
        DISABLED: str = 'false'
        DEFAULT: str = DISABLED  # Default to not waiting
        ENV: str = 'DEBUG_WAIT_FOR_CLIENT'

        @classmethod
        def get(cls) -> str:
            env = os.getenv(cls.ENV, cls.DEFAULT)
            return env.lower()

        @classmethod
        def enabled(cls) -> bool:
            return cls.get() == cls.ENABLED
        
        @classmethod
        def disabled(cls) -> bool:
            return not cls.enabled()


def main():
    if Env.Debug.disabled(): return
    
    port = Env.Port.get()
    print(f"Starting debugpy server on port {port}")

    debugpy.listen((LOCALHOST, port))
    print("Waiting for debugger to attach...")

    if Env.Wait.enabled():
        print("Waiting for debugger client to connect...")
        debugpy.wait_for_client()
        print("Debugger client connected!")

    print("Debugger attached, continuing...")
    app()


if __name__ == "__main__": 
    main()
``` 

### 1.3 Modify Your Dockerfile
Create a debug-enabled `Dockerfile`:
```dockerfile 
FROM python:3.9-slim AS base

# ------------------------------------------
# Configure Environment
# ------------------------------------------
ENV PYTHONUNBUFFERED=1 
ENV PYTHONDONTWRITEBYTECODE=1

# ------------------------------------------
# Install system dependencies
# ------------------------------------------
ARG APT_CACHE="/var/lib/apt/lists"
RUN \
    apt-get update          && \
    apt-get install -y curl && \
    rm -rf $APT_CACHE/*     && \
    pip install poetry;

# ------------------------------------------
# Set working directory
# ------------------------------------------
WORKDIR /app

# ------------------------------------------
# Copy poetry files
# ------------------------------------------
COPY pyproject.toml poetry.lock ./

# ------------------------------------------
# Configure poetry
# ------------------------------------------
RUN \
    poetry config virtualenvs.create false && \
    poetry install --no-dev;

# ------------------------------------------
# Copy application code
# ------------------------------------------
COPY . .

# ------------------------------------------
# Create non-root user
# ------------------------------------------
ARG USR=appuser
RUN \
  useradd --create-home \
    --shell /bin/bash $USR USER $USR
# Default command (can be overridden)
CMD ["python", "debug_main.py"]
``` 

## Step 2: Configure OpenShift Resources

### 2.1 Deployment Configuration
Create or modify your deployment `YAML` to support debugging:
```yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-python-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-python-app
  template:
    metadata:
      labels:
        app: my-python-app
    spec:
      containers:
      - name: my-python-app
        image: your-registry/my-python-app:latest
        ports:
        - containerPort: 8080
          name: http
        # Conditionally expose debug port for non-production
        # ----------------------------------------------------
        - containerPort: 5678
          name: debug
        env:
        - name: ENABLE_DEBUG
          value: "true"  # Set to "false" for production
        - name: DEBUG_PORT
          value: "5678"
        # ----------------------------------------------------
        # ... Your other environment variables
        securityContext:
          runAsNonRoot: true
          allowPrivilegeEscalation: false
        resources:
          limits:
            cpu: 500m
            memory: 1Gi
          requests:
            cpu: 100m
            memory: 256Mi
``` 

### 2.2 Service Configuration
Create a `Service` that exposes the debug port:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-python-app-service
spec:
  selector:
    app: my-python-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: debug
    port: 5678
    targetPort: 5678
    protocol: TCP
  type: ClusterIP
``` 

### 2.3 Route Configuration (OpenShift)
Create a route for external access (if needed):
```yaml 
apiVersion: route.openshift.io/v1 
kind: Route 
metadata: 
  name: my-python-app-debug 
spec: 
  to: 
    kind: Service 
    name: my-python-app-service 
    port: 
      targetPort: debug 
      tls: 
        termination: edge
``` 

## Step 3: Environment-Specific Configuration

### 3.1 Using ConfigMaps for Environment Control
Create a `ConfigMap` to control debug settings:
```yaml 
apiVersion: v1 
kind: ConfigMap 
metadata: 
  name: debug-config 
data: 
  ENABLE_DEBUG: "true" # Change to "false" for production DEBUG_PORT: "5678"
``` 

Reference it in your deployment:
```yaml 
spec: 
  containers:
    - name: my-python-app 
      envFrom:
        - configMapRef: 
            name: debug-config
``` 

### 3.2 Production Safety
For production deployments, either:
1. Set `ENABLE_DEBUG=false` in the `ConfigMap`
2. Use separate deployment manifests
3. Remove debug port exposure entirely

## Step 4: VSCode Configuration

### 4.1 Create launch.json
Create `.vscode/launch.json` in your project root:
```json 
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Python: Remote Attach",
      "type": "python",
      "request": "attach",
      "connect": {
        "host": "localhost",
        "port": 5678
      },
      "pathMappings": [
        {
          "localRoot": "${workspaceFolder}",
          "remoteRoot": "/app"
        }
      ],
      "justMyCode": true
    }
  ]
}

``` 

### 4.2 Port Forwarding Alternative
Instead of exposing via routes, use port forwarding:
```bash
# Forward the debug port to your local machine
oc port-forward pod/my-python-app-xxxxx 5678:5678
``` 

Then use this `launch.json` configuration:
```json 
{
  "name": "Python: Remote Attach (Port Forward)",
  "type": "python",
  "request": "attach",
  "connect": {
    "host": "localhost",
    "port": 5678
  },
  "pathMappings": [
    {
      "localRoot": "${workspaceFolder}",
      "remoteRoot": "/app"
    }
  ],
  "justMyCode": true
}
``` 

## Step 5: Deployment and Testing

### 5.1 Deploy Your Application
```bash
# If using port forwarding (recommended)
oc port-forward deployment/my-python-app 5678:5678

# or apply your manifests
oc apply -f deployment.yaml oc apply -f service.yaml oc apply -f route.yaml # If using routes
``` 

### 5.2 Connect Debugger
1. Set breakpoints in your `VSCode` editor
2. Start the debug configuration: `F5` or Debug → Start Debugging
3. Select "`Python: Remote Attach`"
4. Your application should continue running normally until a breakpoint is hit

### 5.3 Verify Normal Operation
The application should:
- Start normally even when `ENABLE_DEBUG=true`
- Accept HTTP requests on port 8080
- Only pause execution when the debugger is attached AND breakpoints are hit

## Step 6: Advanced Configuration

### 6.1 Multiple Environment Support
Create environment-specific overlay files:

**`kustomization.yaml`:**
```yaml
resources:
- base-deployment.yaml
- service.yaml

patchesStrategicMerge:
- debug-patch.yaml
``` 

**`debug-patch.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-python-app
spec:
  template:
    spec:
      containers:
      - name: my-python-app
        env:
        - name: ENABLE_DEBUG
          value: "true"
```

### 6.2 Security Considerations
- Never enable debug mode in production
- Use NetworkPolicies to restrict debug port access
- Consider using `OpenShift`'s built-in `RBAC` for debug access control

## Troubleshooting
### Common Issues:
1. **Connection refused**: Check if the debug port is properly exposed
2. **Breakpoints not hit**: Verify path mappings in `launch.json`
3. **Application won't start**: Check container logs for `debugpy` errors
4. **Port conflicts**: Ensure debug port (`5678`) isn't already in use

### Debug Commands:
```bash
# Check pod status
oc get pods

# View logs
oc logs deployment/my-python-app

# Test port accessibility
oc port-forward pod/my-python-app-xxxxx 5678:5678

# Check service endpoints
oc get endpoints my-python-app-service
```
