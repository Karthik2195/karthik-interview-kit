# Istio - Service Mesh Platform

## What is it used for?
Istio is used for:
- **Traffic management**: Intelligent request routing and load balancing
- **Security**: Mutual TLS, authorization policies, certificate management
- **Observability**: Distributed tracing, monitoring, metrics collection
- **Service discovery**: Automatic service discovery
- **Resilience**: Retries, circuit breakers, rate limiting
- **Multi-cluster management**: Connect and manage multiple clusters
- **Canary deployments**: Gradual traffic shifting
- **Network policies**: Fine-grained network security

## Installation

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*

# Add istioctl to PATH
export PATH=$PWD/bin:$PATH

# Install Istio
istioctl install --set profile=demo -y

# Label namespace for sidecar injection
kubectl label namespace default istio-injection=enabled

# Verify installation
istioctl verify-install

# Check running Istio components
kubectl get pods -n istio-system
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check Istio version
istioctl version

# Install Istio
istioctl install --set profile=production -y

# Uninstall Istio
istioctl uninstall --purge

# Verify Istio installation
istioctl verify-install

# Analyze configuration
istioctl analyze

# Check mesh configuration
istioctl meshconfig

# Get virtual services
kubectl get virtualservices

# Get destination rules
kubectl get destinationrules

# Get gateway
kubectl get gateways

# Get service entries
kubectl get serviceentries

# Apply configuration
kubectl apply -f destination-rule.yaml

# View traffic
istioctl dashboard kiali
```

### VirtualService Configuration
```yaml
# virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: nginx
spec:
  hosts:
  - nginx
  http:
  - match:
    - uri:
        prefix: "/v2"
    route:
    - destination:
        host: nginx
        subset: v2
  - route:
    - destination:
        host: nginx
        subset: v1
```

### DestinationRule Configuration
```yaml
# destination-rule.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: nginx
spec:
  host: nginx
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

### Gateway Configuration
```yaml
# gateway.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: nginx-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "example.com"
```

### Common Issues & Resolution

**Issue: Sidecar injection not working**
```bash
# Solution: Label namespace correctly
kubectl label namespace default istio-injection=enabled

# Verify label
kubectl get namespace default --show-labels

# Restart pods to trigger injection
kubectl delete pod -l app=nginx

# Check for injection
kubectl get pods nginx-* -o jsonpath='{.items[0].spec.containers[*].name}'
```

**Issue: Traffic not reaching pods**
```bash
# Solution: Check VirtualService
kubectl describe virtualservice nginx

# Check DestinationRule
kubectl describe destinationrule nginx

# Verify service exists
kubectl get svc nginx

# Check endpoints
kubectl get endpoints nginx

# Test connectivity
kubectl exec -it <pod-name> -- curl http://nginx:80
```

**Issue: High latency with Istio**
```bash
# Solution: Check sidecar resource limits
kubectl describe pod <pod-name>

# Adjust Istio resource allocation
istioctl upgrade

# Check pilot CPU usage
kubectl top pod -n istio-system -l app=istiod

# Optimize configuration
kubectl apply -f optimized-config.yaml
```

**Issue: mTLS certificate issues**
```bash
# Solution: Check certificate authority
kubectl get secret -n istio-system

# Verify mTLS status
istioctl authn tls-check

# Enable mTLS
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT
EOF

# Check certificate validity
kubectl get cert -n istio-system
```

### Observability

**Kiali Dashboard**
```bash
# Access Kiali
istioctl dashboard kiali

# View traffic flows
# Kiali Dashboard -> Graph

# Check service metrics
# Kiali Dashboard -> Services
```

**Jaeger Tracing**
```bash
# Access Jaeger
istioctl dashboard jaeger

# View traces
# Select service and time range

# Filter by operation
# Use search functionality
```

**Prometheus Metrics**
```bash
# Access Prometheus
istioctl dashboard prometheus

# Query metrics
# http_requests_total{destination_service="nginx"}
# request_duration_milliseconds

# View graphs
# Use Prometheus query interface
```

### Traffic Management

**Canary Deployment**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: nginx
spec:
  hosts:
  - nginx
  http:
  - match:
    - headers:
        user-agent:
          regex: ".*Chrome.*"
    route:
    - destination:
        host: nginx
        subset: v2
  - route:
    - destination:
        host: nginx
        subset: v1
      weight: 90
    - destination:
        host: nginx
        subset: v2
      weight: 10
```

**Circuit Breaker**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: nginx
spec:
  host: nginx
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

**Rate Limiting**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: ratelimit
spec:
  workloadSelector:
    labels:
      app: nginx
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
    patch:
      operation: INSERT_BEFORE
      value:
        name: envoy.ext_authz
        typedConfig:
          "@type": type.googleapis.com/udpa.type.v1.TypedStruct
```

### Security

**PeerAuthentication (mTLS)**
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT
```

**AuthorizationPolicy**
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: nginx-policy
spec:
  selector:
    matchLabels:
      app: nginx
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/frontend"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/api/v1/*"]
```

### Debugging

```bash
# Analyze Istio configuration
istioctl analyze

# Get configuration summary
istioctl analyze --all-namespaces

# View sidecar configuration
istioctl proxy-config cluster <pod-name>

# Dump entire config
istioctl proxy-config all <pod-name>

# Check routes
istioctl proxy-config routes <pod-name>

# View listeners
istioctl proxy-config listener <pod-name>

# Check endpoints
istioctl proxy-config endpoint <pod-name>

# View sidecar logs
kubectl logs <pod-name> -c istio-proxy

# Enable debug logging
kubectl exec <pod-name> -c istio-proxy -- \
  curl localhost:15000/logging?level=debug

# Check pilot logs
kubectl logs -n istio-system -l app=istiod

# Get mesh configuration
kubectl get cm istio -n istio-system -o yaml

# Test connectivity
kubectl exec <source-pod> -- \
  curl http://<destination-service>:80
```

### Advanced Features

**Multi-Cluster Service Entry**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-service
spec:
  hosts:
  - external.example.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
```

**Request Routing with Headers**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: nginx
spec:
  hosts:
  - nginx
  http:
  - match:
    - headers:
        x-user-id:
          exact: "admin"
    route:
    - destination:
        host: nginx
        subset: v2
  - route:
    - destination:
        host: nginx
        subset: v1
```

### Performance Tuning

```bash
# Reduce Istio overhead
# 1. Use minimum required features
# 2. Disable unused features in Istio ConfigMap
# 3. Monitor resource usage:
kubectl top pod -n istio-system

# 4. Optimize pilot configuration
kubectl edit cm istio -n istio-system
```
