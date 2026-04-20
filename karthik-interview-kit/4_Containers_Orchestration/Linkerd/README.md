# Linkerd - Lightweight Service Mesh

## What is it used for?
Linkerd is used for:
- **Service mesh**: Manage service-to-service communication
- **Automatic mTLS**: Secure service communication
- **Traffic management**: Control traffic flow
- **Observability**: Metrics and tracing
- **Reliability**: Retries, timeouts, circuit breakers
- **Traffic splitting**: Canary deployments
- **Load balancing**: Intelligent load balancing
- **Kubernetes-native**: Deep Kubernetes integration

## Installation

```bash
# Install Linkerd CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh

# Add to PATH
export PATH=$PATH:$HOME/.linkerd2/bin

# Install Linkerd on cluster
linkerd install | kubectl apply -f -

# Install add-ons
linkerd viz install | kubectl apply -f -
linkerd jaeger install | kubectl apply -f -

# Verify installation
linkerd check
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check Linkerd version
linkerd version

# Verify installation
linkerd check

# Check specific components
linkerd check --pre
linkerd check -o short

# Enable namespace for automatic mTLS
kubectl annotate namespace default linkerd.io/inject=enabled --overwrite

# Check injected proxies
kubectl get pods -l linkerd.io/control-plane

# View proxy status
linkerd proxy-control-plane-versions

# Access Linkerd dashboard
linkerd viz dashboard

# View metrics
linkerd viz stat

# View services
linkerd viz top

# Check service topology
linkerd viz edges deployment

# View live traffic
linkerd viz live

# Trace requests
linkerd viz trace deployment/my-app
```

### Namespace Configuration
```bash
# Enable automatic sidecar injection
kubectl annotate namespace my-ns linkerd.io/inject=enabled

# Verify annotation
kubectl get ns my-ns --show-labels

# Label pods for injection
kubectl label pod my-pod linkerd.io/inject=enabled

# Check injected pods
kubectl get pods --show-labels | grep linkerd.io/inject
```

### Traffic Management
```yaml
# TrafficSplit for canary deployment
apiVersion: split.smi-spec.io/v1alpha3
kind: TrafficSplit
metadata:
  name: my-app-split
spec:
  service: my-app
  backends:
  - service: my-app-v1
    weight: 90
  - service: my-app-v2
    weight: 10

---
# ServiceProfile for per-route metrics
apiVersion: linkerd.io/v1alpha1
kind: ServiceProfile
metadata:
  name: my-app
spec:
  service:
    name: my-app
    port: 8080
  routes:
  - name: POST /api/users
    condition:
      method: POST
      pathRegex: /api/users
  - name: GET /api/users
    condition:
      method: GET
      pathRegex: /api/users
    timeout: 30s
    retries:
      limit: 5
      backoff: exponential
```

### Policies
```yaml
# AuthorizationPolicy for mTLS
apiVersion: policy.linkerd.io/v1alpha1
kind: AuthorizationPolicy
metadata:
  name: allow-web-to-api
spec:
  targetRef:
    group: core
    kind: ServiceAccount
    name: api
  rules:
  - from:
    - principalName: web
    to:
    - method: GET
      path: /api/v1/users

# NetworkPolicy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-api
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - protocol: TCP
      port: 8080
```

### Common Issues & Resolution

**Issue: Sidecar injection not working**
```bash
# Solution: Enable automatic injection
kubectl annotate namespace default linkerd.io/inject=enabled --overwrite

# Verify annotation
kubectl get ns default --show-labels

# Restart pods to trigger injection
kubectl rollout restart deployment/my-app

# Check injected pods
kubectl get pods my-app-* -o jsonpath='{.items[0].spec.containers[*].name}'
```

**Issue: mTLS connection fails**
```bash
# Solution: Check certificate validity
linkerd diagnose --proxy-metrics=detailed

# Verify proxy is running
kubectl get pods -l app=my-app -o jsonpath='{.items[0].spec.containers[*].name}'

# Check proxy logs
kubectl logs -l app=my-app -c linkerd-proxy

# Validate service profile
kubectl get serviceprofile
```

**Issue: High latency with Linkerd**
```bash
# Solution: Check proxy resource limits
kubectl get pod -o json | jq '.items[].spec.containers[] | select(.name=="linkerd-proxy") | .resources'

# Increase proxy resources
kubectl set resources daemonset linkerd-proxy --limits=cpu=500m,memory=256Mi -n linkerd

# Check metrics
linkerd viz stat
```

**Issue: Dashboard not accessible**
```bash
# Solution: Check viz installation
linkerd viz check

# Port forward to dashboard
kubectl port-forward -n linkerd svc/web 3000:8084

# Access dashboard
# http://localhost:3000
```

### Debugging
```bash
# Verbose logging
linkerd check -o short --verbose

# Check proxy metrics
linkerd viz metrics

# View proxy config
kubectl describe pod <pod-name>

# Get proxy logs
kubectl logs <pod-name> -c linkerd-proxy

# Test connectivity between pods
kubectl exec -it <pod1> -- curl http://<pod2>:8080

# Check mTLS status
linkerd diagnose --proxy-metrics=detailed

# Trace traffic
kubectl exec -it <pod-name> -- ss -netp
```

### Monitoring Integration

**Prometheus Integration**
```bash
# Prometheus is automatically installed with viz
# Access Prometheus
kubectl port-forward -n linkerd svc/prometheus 9090

# Query Prometheus
# http://localhost:9090
```

**Grafana Integration**
```bash
# Install Grafana for Linkerd
kubectl apply -f https://raw.githubusercontent.com/linkerd/linkerd2/main/viz/charts/linkerd-viz/templates/grafana.yaml

# Access Grafana
kubectl port-forward -n linkerd svc/grafana 3000
```

### Production Deployment
```bash
# Install with HA control plane
linkerd install --ha | kubectl apply -f -

# Increase replicas
kubectl scale deployment -l app.kubernetes.io/part-of=linkerd \
  --replicas=3 -n linkerd

# Configure certificate rotation
linkerd check --output short
```

### Traffic Analysis
```bash
# View traffic metrics
linkerd viz stat services

# Check error rates
linkerd viz stat -o json | jq '.statTables[].podGroup.rows[] | select(.errorRate > "0")'

# Analyze latency
linkerd viz stat -l p95

# View traffic topology
linkerd viz edges resources
```

### Advanced Features
```bash
# Mutual TLS across namespaces
linkerd install --tls=external | kubectl apply -f -

# Policy enforcement
linkerd check policy

# Network policies with Linkerd
kubectl apply -f network-policy.yaml

# Custom metrics
kubectl apply -f custom-metrics.yaml
```
