# Kubernetes-Native IDE Reverse Proxy System
## Project Overview
A sophisticated reverse proxy implementation that provides seamless access to containerized development 
environments (RStudio, VSCode, JupyterLab) running on Kubernetes. This system dynamically routes HTTP and 
WebSocket traffic to user-specific IDE instances while maintaining session security and protocol 
compatibility.
## Technical Architecture
### Core Design Pattern
Built on a **composition-based architecture** using Go's embedded structs to create specialized IDE clients 
that share common Kubernetes service discovery and proxy functionality. Each IDE type inherits base proxy 
capabilities while implementing protocol-specific routing logic.
### Key Components
#### 1. **Dynamic Service Discovery**
- **Kubernetes-native resolution** of IDE pod and service endpoints
- **Real-time cluster IP and port detection** for user instances
- **Graceful handling** of service availability states

#### 2. **Multi-Protocol Proxy Support**
- **HTTP/HTTPS proxying** for web-based IDEs
- **WebSocket upgrade handling** for real-time features
- **Protocol-aware header manipulation** for each IDE type

#### 3. **IDE-Specific Routing**
**RStudio Integration:**
- Custom header injection `X-RStudio-Root-Path`
- Base path rewriting for RServer compatibility
- Session-aware proxy configuration

**VSCode Integration:**
- Advanced WebSocket connection handling
- Real-time Language Server Protocol (LSP) support
- Connection upgrade detection and preservation

**JupyterLab Integration:**
- Notebook kernel proxy management
- header configuration `X-Forwarded-Prefix`
- Dynamic path prefix handling

## Technical Implementation Highlights
### Kubernetes Service Discovery
``` go
// Dynamic endpoint resolution
func (c *client) GetAddressHTTP(user string, project string) (string, error) {
    svc, err := c.GetService(user, project)
    if err != nil {
        return "", err
    }
    
    ip := c.serviceIP(svc)
    portNum := c.servicePortNumber(svc)
    addr := fmt.Sprintf("http://%s:%d", ip, portNum)
    return addr, nil
}
```
### WebSocket-Aware Proxying
``` go
// VSCode WebSocket upgrade handling
func (vc *VSCodeClient) director(proxy *reverseProxy, user string, project string) func(*http.Request) {
    return func(req *http.Request) {
        hdr := req.Header
        
        // Detect WebSocket upgrade requests
        if strings.Contains(hdr.Get("Connection"), "Upgrade") && 
           strings.Contains(hdr.Get("Upgrade"), "websocket") {
            hdr.Set("Connection", "Upgrade")
            hdr.Set("Upgrade", "websocket")
        }
        
        hdr.Set("X-Forwarded-Prefix", rootPath)
        vc.setProxyPath(req, rootPath)
    }
}
```
### Real-Time Event Streaming
``` go
// Server-Sent Events for IDE status updates
func stream(c *gin.Context, informer *informerIDE) bool {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()
    
    return c.Stream(func(w io.Writer) bool {
        select {
        case <-ticker.C:
            handleSSE(c, informer)
            return true
        case <-c.Request.Context().Done():
            informer.Stop()
            return false
        }
    })
}
```
## Security & Operational Features
### Multi-Tenant Isolation
- **User-scoped resource naming** prevents cross-user access
- **Namespace-based project isolation** in Kubernetes
- **Session-validated routing** with authentication middleware

### Protocol Security
- **X-Frame-Options injection** for clickjacking protection
- **HTTP method filtering** to block potentially dangerous requests
- **Secure header management** for iframe embedding

### Production Readiness
- **Graceful connection handling** with context-aware termination
- **Resource cleanup** on client disconnection
- **Comprehensive error handling** with proper HTTP status codes

## REST API Design
### Endpoint Structure
``` 
GET    /api/v1/analysis              # Get active IDE
POST   /api/v1/analysis              # Create new IDE instance
DELETE /api/v1/analysis              # Terminate IDE instance
GET    /api/v1/analysis/options      # Get available configurations

# Real-time event streams
GET    /api/v1/analysis/events/{ide-type}

# Reverse proxy endpoints
ANY    /api/v1/connect/{project}/{user}/{ide-type}/*
```
### Dynamic Route Generation
User-specific proxy routes are dynamically generated based on:
- **Project namespace** for multi-tenancy
- for user isolation **Username**
- **IDE type** for protocol-specific handling
- **Wildcard paths** for complete application proxying

## Technical Challenges Solved
### 1. **Path Rewriting Complexity**
Different IDEs expect different base paths and routing structures. The system implements IDE-specific path manipulation while preserving application functionality.
### 2. **WebSocket Protocol Preservation**
VSCode requires persistent WebSocket connections for real-time features. The proxy correctly identifies and preserves WebSocket upgrade requests while maintaining connection state.
### 3. **Multi-Protocol Support**
Each IDE uses different header requirements and routing expectations. The system provides specialized handling while maintaining a unified interface.
### 4. **Kubernetes Service Discovery**
Dynamic resolution of cluster-internal services with proper error handling for unavailable or starting instances.
## Performance & Scalability
### Efficient Resource Management
- **Connection pooling** through Go's HTTP client
- **Context-aware cleanup** preventing resource leaks
- **Minimal proxy overhead** with direct stream forwarding

### Horizontal Scaling Support
- **Stateless proxy design** allows multiple instances
- **Kubernetes-native discovery** works across cluster nodes
- **Session-external state management** supports load balancing

## Development Impact
This reverse proxy system enables:
- **Seamless browser-based development** environments
- **Real-time collaborative coding** through WebSocket support
- **Multi-language development** with specialized IDE containers
- **Secure multi-tenant** development platform
- **Cloud-native development workflows** with Kubernetes integration

The implementation demonstrates advanced Go programming concepts including interface composition, protocol-aware networking, and production-ready error handling while solving complex distributed systems challenges in a cloud-native environment.
