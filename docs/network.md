# 🌐 Network Configuration

## Architecture

```mermaid
graph TD
    subgraph "Physical Topology"
        A[Internet Gateway] --> B[Switch]
        B --> C[K3s Node]
    end

    subgraph "Logical Topology"
        D[Internet] --> E[Cloudflare]
        E --> F[Cloudflare Tunnel]
        F --> G[Gateway API]
        G --> H[Cilium Service Mesh]
        H --> I[Kubernetes Service]
        I --> J[Pod]
    end

    style C fill:#f9f,stroke:#333
    style H fill:#bbf,stroke:#333
```

## Traffic Flow

```mermaid
sequenceDiagram
    participant User
    participant Cloudflare
    participant Gateway as Gateway API
    participant Service as K8s Service
    participant Pod

    User->>Cloudflare: HTTPS Request
    Cloudflare->>Gateway: Proxied Request (SSL terminated)
    Gateway->>Service: Route to Service
    Service->>Pod: Forward to Pod
    Pod->>Service: Response
    Service->>Gateway: Return Response
    Gateway->>Cloudflare: Forward Response
    Cloudflare->>User: HTTPS Response
```

## IP Allocation

- **Internal Network**: 192.168.1.0/24
  - Gateway: 192.168.1.1
  - K3s Node: 192.168.1.10

- **Pod Network**: 10.42.0.0/16 (Cilium)
  - Services: 10.43.0.0/16
  - CoreDNS: 10.43.0.10

## Gateway API Configuration

### External Gateway
```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: external-gateway
  namespace: gateway-system
spec:
  gatewayClassName: cilium
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
  - name: https
    port: 443
    protocol: HTTPS
    allowedRoutes:
      namespaces:
        from: All
    tls:
      mode: Terminate
      certificateRefs:
      - name: wildcard-cert
```

### Internal Gateway
```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: internal-gateway
  namespace: gateway-system
spec:
  gatewayClassName: cilium
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
```

## Components

### Cilium
- **Function**: CNI plugin, Service Mesh, Gateway API implementation
- **Installation**: Deployed via Helm in the infrastructure tier
- **Configuration**: Managed through Helm values

### CoreDNS
- **Function**: DNS management for cluster
- **Installation**: Bundled with K3s
- **Configuration**: Custom configmap for internal domains

### Gateway API
- **Function**: Ingress/Gateway management
- **Installation**: CRDs installed separately, implementation by Cilium
- **Configuration**: Gateway and HTTPRoute resources

### Cloudflare Tunnel
- **Function**: Secure external access
- **Installation**: Deployed as a Kubernetes deployment
- **Configuration**: Using tunnel credentials from secrets

## DNS Configuration

### Internal Domains
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  server.conf: |
    home.arpa:53 {
        errors
        cache 30
        forward . 192.168.1.1
    }
```

## Network Flow

### Internal Access
```mermaid
sequenceDiagram
    participant Internal as Internal Client
    participant CoreDNS
    participant Gateway as Internal Gateway
    participant Service
    participant Pod

    Internal->>CoreDNS: DNS Query (service.home.arpa)
    CoreDNS->>Internal: DNS Response (192.168.1.10)
    Internal->>Gateway: HTTP Request
    Gateway->>Service: Route Request
    Service->>Pod: Forward Request
    Pod->>Internal: Response
```

### External Access
```mermaid
sequenceDiagram
    participant External as External Client
    participant Cloudflare
    participant Tunnel as Cloudflare Tunnel
    participant Gateway as External Gateway
    participant Service
    participant Pod

    External->>Cloudflare: DNS Query (service.example.com)
    Cloudflare->>External: DNS Response (Cloudflare IP)
    External->>Cloudflare: HTTPS Request
    Cloudflare->>Tunnel: Proxied Request
    Tunnel->>Gateway: Forward Request
    Gateway->>Service: Route Request
    Service->>Pod: Forward Request
    Pod->>External: Response (reverse path)
```

## Setup Steps

### 1. Install Cilium (Infrastructure Tier)

Cilium is installed via Helm as part of the infrastructure tier:

```bash
# Installing Cilium via Helm is handled automatically by the infrastructure ApplicationSet
# The Helm chart is located at infrastructure/networking/cilium
# You can view the values at infrastructure/networking/cilium/values.yaml

# To manually install Cilium:
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set kubeProxyReplacement=strict \
  --set gatewayAPI.enabled=true \
  --set-string extraConfig.enable-gateway-api=true \
  --set ipam.mode=kubernetes

# Verify Cilium is running
kubectl -n kube-system get pods -l k8s-app=cilium
```

### 2. Configure CoreDNS
```bash
# Apply custom CoreDNS configuration
kubectl apply -f infrastructure/networking/coredns/coredns-custom.yaml

# Restart CoreDNS to apply changes
kubectl rollout restart -n kube-system deployment coredns
```

### 3. Setup Gateways
```bash
# Create gateway namespace
kubectl create namespace gateway-system

# Apply gateway configurations
kubectl apply -f infrastructure/networking/gateway/
```

### 4. Configure Cloudflare
```bash
# Add tunnel secrets (see external-services.md)
kubectl apply -f infrastructure/networking/cloudflared/secrets.yaml

# Deploy cloudflared tunnel
kubectl apply -f infrastructure/networking/cloudflared/deployment.yaml
```

## Validation

### Cilium Status
```bash
# Check Cilium status
cilium status

# Verify connectivity
cilium connectivity test
```

### DNS Resolution
```bash
# Test internal DNS
kubectl run -it --rm debug --image=curlimages/curl -- nslookup kubernetes.default.svc.cluster.local

# Test external DNS
kubectl run -it --rm debug --image=curlimages/curl -- nslookup example.com
```

### Gateway Routing
```bash
# Check gateway status
kubectl get gateway -A

# Test routes
kubectl get httproute -A
```

### Cloudflare Tunnel
```bash
# Check tunnel pods
kubectl get pods -n cloudflared

# Check tunnel logs
kubectl logs -n cloudflared -l app=cloudflared
```

## Troubleshooting

### DNS Issues
1. Check CoreDNS pods:
   ```bash
   kubectl get pods -n kube-system -l k8s-app=kube-dns
   kubectl logs -n kube-system -l k8s-app=kube-dns
   ```

2. Verify custom config:
   ```bash
   kubectl get configmap -n kube-system coredns-custom -o yaml
   ```

### Gateway Issues
1. Check gateway status:
   ```bash
   kubectl describe gateway -n gateway-system external-gateway
   ```

2. Verify routes:
   ```bash
   kubectl describe httproute -A
   ```

### Cloudflare Issues
1. Check tunnel status:
   ```bash
   kubectl get pods -n cloudflared
   kubectl logs -n cloudflared -l app=cloudflared
   ```

2. Verify tunnel connectivity:
   ```bash
   # Port-forward to cloudflared metrics
   kubectl port-forward -n cloudflared svc/cloudflared 8080:2000
   # Access metrics at http://localhost:8080/metrics
   ``` 