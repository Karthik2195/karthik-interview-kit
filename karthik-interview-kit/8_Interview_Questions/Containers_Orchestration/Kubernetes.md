# Kubernetes - Senior Level Interview Questions & Answers

**Contributed by:** Karthik Reddy

## Table of Contents
1. Architecture & Design
2. Resource Management
3. Networking & Security
4. Troubleshooting & Debugging
5. Production Patterns

---

## 1. Architecture & Design

### Q (High-Level): What is Kubernetes and why would I use it instead of just running servers?

**Interviewer:** "Karthik, I've never used Kubernetes. Walk me through a practical scenario. I have 100 containers running my business. Without Kubernetes, what does my operational burden look like compared to using Kubernetes?"

**Karthik:** "I appreciate the foundational question. Without Kubernetes, the operational model becomes unsustainable at scale. Consider the manual approach: you SSH into server 1 and manually execute Docker commands to run container instances. You repeat this process across servers 2, 50, and 100. When a container crashes unexpectedly, you manually reconnect and restart the process. Server failures require immediate manual intervention - spinning up replacement servers, reconfiguring networking, and reallocating container workloads. Application updates demand systematic procedures: iterating through 50 production servers sequentially and updating each one individually. Troubleshooting becomes challenging because visibility is fragmented - determining which physical server hosts a specific container requires manual investigation. This operational model empirically consumes 60+ hours monthly in reactive maintenance activities, leaving minimal capacity for feature development. Kubernetes transforms this paradigm through declarative infrastructure. You specify a single manifest declaring: deploy three instances of the payment service. Kubernetes automatically handles execution across the cluster. When instances crash, Kubernetes immediately restarts them automatically. Server failures trigger automatic pod migration to remaining healthy nodes. Application updates execute through rolling deployment strategies providing zero-downtime transitions. Kubernetes consolidates cluster-wide visibility through unified APIs - no manual server investigation required. This architectural shift reduces operational overhead to approximately 2 hours monthly, enabling teams to allocate attention toward feature development rather than firefighting infrastructure issues. The business impact is substantial: reduced incidents, improved system reliability, and accelerated feature velocity."

---

### Q (High-Level): Real-time production issue - A pod keeps crashing. How do you diagnose in 30 seconds?

**Interviewer:** "Alert fires: payment-service pod experiencing repeated crashes. You have limited time before customer impact escalates. Walk me through your diagnostic process."

**Karthik:** "My systematic approach prioritizes rapid root cause identification. First, I execute `kubectl describe pod <pod-name> -n production` which reveals immediate state information including restart count history - for example, 47 restarts indicates a severe recurring failure pattern. Simultaneously, I examine cluster events via `kubectl get events -n production --sort-by='.lastTimestamp'` which provides chronological context. This often indicates readiness probe failures or resource constraint issues. The subsequent step involves application log analysis through `kubectl logs <pod-name> -n production --tail=100` which typically reveals the specific error - for instance, database connectivity failures indicate infrastructure dependency problems, OutOfMemoryError suggests resource insufficiency. I cross-reference this with pod resource specifications using `kubectl describe pod <pod-name> -n production | grep -A 5 Limits` which identifies whether memory or CPU constraints are causing the crashes. The empirically most common failure modes are: memory limits insufficient for application requirements causing OOMKilled terminations, external database connection failures indicating upstream infrastructure issues, missing configuration files suggesting deployment problems, or port conflicts on the host. Addressing these issues typically requires modifying resource allocations - for example, increasing memory limits from 256Mi to 1Gi - or investigating upstream dependency status. This systematic progression from pod status through event logs to application logs and resource limits provides comprehensive visibility. In most cases, this process completes within 5 minutes and identifies the root cause definitively."

---

### Q: Design a multi-tenant Kubernetes cluster with isolation and resource controls

**A:**
```yaml
# Multi-tenant architecture using namespaces, RBAC, and network policies

# 1. Namespace for each tenant
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-alpha
  labels:
    tenant: alpha
    environment: production

---
# 2. Resource Quota per tenant
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-alpha-quota
  namespace: tenant-alpha
spec:
  hard:
    requests.cpu: "100"
    requests.memory: "100Gi"
    limits.cpu: "200"
    limits.memory: "200Gi"
    pods: "1000"
    services: "100"
    persistentvolumeclaims: "50"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["high", "medium"]

---
# 3. Network Policy for isolation
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-isolation
  namespace: tenant-alpha
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tenant: alpha
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          tenant: alpha
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

---
# 4. RBAC - Tenant admin role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tenant-admin
  namespace: tenant-alpha
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]

---
# 5. Pod Security Policy
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
  - ALL
  volumes:
  - 'configMap'
  - 'emptyDir'
  - 'projected'
  - 'secret'
  - 'downwardAPI'
  - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'MustRunAs'
  readOnlyRootFilesystem: false

---
# 6. Resource Limits per pod
apiVersion: v1
kind: Pod
metadata:
  name: limited-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

**Design Principles:**
- Namespace isolation per tenant
- Resource quotas prevent resource starvation
- Network policies enforce traffic isolation
- RBAC controls access
- Pod Security Policies enforce runtime constraints

---

### Q: Explain pod disruption budgets and how to ensure high availability during cluster upgrades

**A:**
```yaml
# Pod Disruption Budget (PDB) strategy

apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  minAvailable: 2  # Always keep at least 2 pods running
  selector:
    matchLabels:
      app: myapp
  unhealthyPodEvictionPolicy: IfHealthyBudget

---
# Alternative: maxUnavailable
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  maxUnavailable: 1  # At most 1 pod can be disrupted
  selector:
    matchLabels:
      app: myapp

---
# Deployment with proper disruption handling
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Never make unavailable pods
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      affinity:
        # Pod anti-affinity for better distribution
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - myapp
              topologyKey: kubernetes.io/hostname
      
      # Graceful shutdown
      terminationGracePeriodSeconds: 30
      
      containers:
      - name: app
        image: myapp:latest
        
        # Readiness probe for healthy detection
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
          
        # Liveness probe for restart
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
        
        # Lifecycle hooks for graceful shutdown
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
```

**HA During Upgrades:**
1. PDB ensures minimum availability
2. Rolling updates gradually replace pods
3. Health checks verify pod readiness
4. Graceful shutdown allows connection draining
5. Anti-affinity spreads pods across nodes

---

## 2. Resource Management

### Q (High-Level): My pod is getting evicted from the node. What does that mean and how do I fix it?

**A (Real-Time Node Pressure):**
Your pod is running fine, then suddenly gets evicted. It disappears!

**What happened:**
```
Node running low on memory → Kubernetes kills pods to free space
Your pod got selected to die → EVICTED
Result: Pod moved to another node (if available)
```

**Root cause:**
```yaml
# Your pod declared minimal resources
resources:
  requests:
    memory: 100Mi  # "I only need 100Mi"
    cpu: 10m
  limits:
    memory: 256Mi  # "I'll never use more than 256Mi"
    cpu: 100m

# Real usage at runtime:
# Actual memory: 400Mi  # OOPS! Using more than limit!
```

**Result:**
- Memory usage: 400Mi
- Pod limit: 256Mi
- Difference: Kubernetes kills pod
- Error: OOMKilled

**Real-time fix:**
```yaml
resources:
  requests:
    memory: 512Mi  # Realistic minimum
    cpu: 500m
  limits:
    memory: 1Gi  # Realistic maximum
    cpu: 1000m
```

**How to get right numbers:**
1. Deploy with high limits
2. Run real workload
3. Check actual usage: `kubectl top pod`
4. Set requests = actual avg, limits = actual peak

---

### Q (High-Level): Should I set CPU/memory requests on my pod? What happens if I don't?

**A (Resource Planning):**
Your app works fine without resource requests. Why set them?

**Scenario without requests:**
```yaml
# Pod with NO resource requests
spec:
  containers:
  - name: app
    image: myapp:latest
    # NO requests defined!
    # NO limits defined!
```

**What happens:**
```
Day 1: 5 pods deployed, each uses 200Mi memory = 1Gb total
Day 2: 100 pods deployed, each uses 200Mi memory = 20Gb needed
Node has: 16Gb total

Result: Kubernetes doesn't know how to fit them
Pods randomly evicted: "Not enough memory!"
Team frustrated: "Why are pods crashing?"
```

**With proper requests:**
```yaml
resources:
  requests:
    memory: 256Mi
    cpu: 100m
  limits:
    memory: 512Mi
    cpu: 500m
```

**Now Kubernetes knows:**
```
Each pod needs 256Mi guaranteed
Node has 16Gb = can fit 60 pods safely
If 61st pod comes: Kubernetes says "No room, wait on another node"
Result: Pods don't randomly evict, predictable behavior
```

**Real-time impact:**
- Without requests: 🎲 Random pod deaths = production nightmare
- With requests: ✅ Predictable, stable = calm team

---

### Q: How would you implement intelligent resource allocation in a multi-tenant environment?

**A:**
```yaml
# Vertical Pod Autoscaler for right-sizing
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"  # Automatically update if recreate possible
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2
        memory: 2Gi
      controlledResources: ["cpu", "memory"]
      controlledValues: RequestsAndLimits

---
# Horizontal Pod Autoscaler with custom metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60  # Double replicas per minute
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60  # Halve replicas per minute

---
# Priority Classes for resource fairness
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-service
value: 1000
globalDefault: false
description: "Critical services get priority"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-processing
value: 100
globalDefault: false
description: "Batch jobs get lower priority"

---
# Pod using priority class
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  priorityClassName: critical-service
  containers:
  - name: app
    image: critical-app:latest
```

---

## 3. Networking & Security

### Q (High-Level): Two pods need to talk to each other. How does networking work in Kubernetes?

**A (Real-Time Pod Communication):**
You have 2 pods on the same node. Pod A wants to call Pod B's API.

**How it works:**
```
Pod A (IP: 10.1.1.5) wants to call Pod B (IP: 10.1.1.10)
Port: 8080 (Pod B listening on 8080)

Pod A sends request:
curl http://10.1.1.10:8080/api/users

Kubernetes network:
- Each pod gets its own IP (10.1.1.x)
- Pods can talk to any other pod by IP + port
- Network plugin (Flannel, Weave, Cilium) handles routing
```

**Using Service instead (better practice):**
```yaml
# Pod A doesn't need to know Pod B's IP
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
  - port: 8080
    targetPort: 8080
```

**Now Pod A uses DNS:**
```bash
# Pod A just calls by name (no IP needed)
curl http://user-service:8080/api/users

# Kubernetes DNS resolves: user-service → 10.1.1.10
# If Pod B dies and restarts with new IP, DNS updates automatically
```

**Real-time benefit:**
- Without Service: "Pod B has new IP, update all configs" 😱
- With Service: "Use DNS name, Kubernetes handles IP changes" ✅

---

### Q (High-Level): Can pod-to-pod traffic be intercepted? Is it secure?

**A (Network Security Question):**
Your payment service talks to your database pod. Worry: Is data encrypted?

**Default Kubernetes:**
```
Traffic between pods is PLAIN TEXT (not encrypted)
If someone gains node access:
- They can `tcpdump` and see all pod-to-pod traffic
- See passwords, credit cards, API keys
Result: Not secure for sensitive data! ⚠️
```

**Fix: Use mTLS (mutual TLS)**
```yaml
# Istio/Linkerd automatically encrypt pod traffic
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT  # Encrypt ALL pod traffic
```

**What mTLS does:**
1. Encrypts traffic between pods
2. Verifies pods are who they claim
3. Prevents unauthorized pods from calling your service

**Real-time scenario:**
```
Before mTLS: Payment pod → Database pod (plain text)
Attacker gains node access → tcpdump shows card numbers ❌

After mTLS: Payment pod → Database pod (encrypted)
Attacker gains node access → tcpdump shows gibberish ✅
```

---

### Q: Design a secure service-to-service communication architecture

**A:**
```yaml
# mTLS with Istio/Linkerd
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT  # Require mTLS for all traffic

---
# Authorization policy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: frontend-policy
  namespace: default
spec:
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/frontend"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/api/v1/*"]

---
# Network policy for service communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: service-mesh-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9090

---
# Ingress with TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-cert
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080

---
# Service Account for workload identity
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
  annotations:
    iam.gke.io/gcp-service-account: myapp@project.iam.gserviceaccount.com

---
# RBAC for service communication
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: api-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["endpoints", "services"]
  verbs: ["get", "list", "watch"]
```

---

## 4. Troubleshooting & Debugging

### Q: Walk through debugging a pod that is stuck in pending state

**A:**
```bash
# Step 1: Describe pod to see conditions
kubectl describe pod stuck-pod -n production

# Output analysis:
# Events section shows what happened:
# - PodScheduled: false - can't be scheduled
# - Ready: false - not ready
# - ContainersReady: false - containers not ready

# Step 2: Check scheduler logs
kubectl logs -n kube-system -l component=kube-scheduler --tail=50

# Step 3: Check node availability
kubectl get nodes
kubectl top nodes  # Resource availability

# Step 4: Check resource requests vs available
kubectl get resourcequota -n production
kubectl describe node <node-name>

# Step 5: Check node selectors/affinities
kubectl get pod stuck-pod -o yaml | grep -A5 "nodeSelector\|affinity"

# Step 6: Check taints and tolerations
kubectl describe nodes
# Look for: Taints: node-type=gpu:NoSchedule
# Pod needs: tolerations

# Common issues and solutions:
# Issue 1: Insufficient CPU/Memory
# Solution: Delete other pods, upgrade node, change resource requests

# Issue 2: Node selector doesn't match
# Solution: Check node labels, add missing labels to nodes

# Issue 3: Taints on nodes
# Solution: Add tolerations to pod

# Issue 4: PVC not bound
# Solution: Check PV availability, storage class

# Debugging script
debug_pending_pod() {
    local pod="$1"
    local namespace="${2:-default}"
    
    echo "=== Pod Status ==="
    kubectl get pod "$pod" -n "$namespace" -o wide
    
    echo "=== Pod Events ==="
    kubectl describe pod "$pod" -n "$namespace" | grep -A 20 Events
    
    echo "=== Node Status ==="
    kubectl get nodes
    
    echo "=== Resource Quotas ==="
    kubectl describe resourcequota -n "$namespace"
    
    echo "=== Pod Specification ==="
    kubectl get pod "$pod" -n "$namespace" -o yaml | \
        grep -E "nodeSelector|affinity|tolerations|resources" -A 5
}

debug_pending_pod "stuck-pod" "production"
```

---

### Q: How would you debug a CrashLoopBackOff pod?

**A:**
```bash
# Step 1: Check pod logs
kubectl logs pod-name -n namespace --previous  # Previous crash
kubectl logs pod-name -n namespace --tail=100

# Step 2: Detailed pod status
kubectl describe pod pod-name -n namespace
# Look for: LastState.Terminated.ExitCode

# Step 3: Check container exit codes
kubectl get pod pod-name -n namespace -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'

# Step 4: Inspect container with debug image
kubectl debug -it pod-name -n namespace --image=busybox:1.28

# Step 5: Check resource limits
kubectl get pod pod-name -n namespace -o yaml | grep -A 5 "resources"

# Step 6: Check environment variables
kubectl get pod pod-name -n namespace -o yaml | grep -A 5 "env:"

# Common exit codes:
# 1: General error
# 127: Command not found
# 137: OOMKilled (-9)
# 139: SIGSEGV (segmentation fault)
# 143: SIGTERM (graceful shutdown timeout)

# Check for resource issues
kubectl top pod pod-name -n namespace
kubectl describe node <node> | grep -A 5 "Allocated resources"

# View pod events
kubectl get events -n namespace --sort-by='.lastTimestamp' | grep pod-name

# Debugging script for CrashLoopBackOff
debug_crash_pod() {
    local pod="$1"
    local namespace="${2:-default}"
    
    echo "=== Current Logs ==="
    kubectl logs "$pod" -n "$namespace" --tail=20 2>/dev/null || echo "No current logs"
    
    echo -e "\n=== Previous Logs ==="
    kubectl logs "$pod" -n "$namespace" --previous --tail=20 2>/dev/null || echo "No previous logs"
    
    echo -e "\n=== Exit Code ==="
    kubectl get pod "$pod" -n "$namespace" -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}' 2>/dev/null
    
    echo -e "\n=== Resource Usage ==="
    kubectl top pod "$pod" -n "$namespace"
}

debug_crash_pod "problematic-pod" "production"
```

---

## 5. Production Patterns

### Q: Design a blue-green deployment strategy for zero-downtime updates

**A:**
```bash
#!/bin/bash
# Blue-green deployment script

BLUE_VERSION="v1.0"
GREEN_VERSION="v1.1"
SELECTOR="app=myapp"

# Step 1: Deploy green version alongside blue
kubectl set image deployment/myapp-green \
  myapp=myapp:$GREEN_VERSION \
  -n production

# Wait for green to be ready
kubectl rollout status deployment/myapp-green -n production --timeout=5m

# Step 2: Run smoke tests
run_smoke_tests() {
    local service_name="$1"
    echo "Running smoke tests on $service_name..."
    curl -f http://$service_name:8080/health || return 1
    curl -f http://$service_name:8080/api/test || return 1
    return 0
}

# Run tests on green
if ! run_smoke_tests "myapp-green"; then
    echo "Green tests failed, keeping blue active"
    exit 1
fi

# Step 3: Switch traffic to green
kubectl patch service myapp \
  -p '{"spec":{"selector":{"version":"'$GREEN_VERSION'"}}}'

# Step 4: Monitor for errors
monitor_errors() {
    local service="$1"
    local threshold="${2:-5}"
    
    for i in {1..60}; do
        error_rate=$(kubectl logs -l app=$service --tail=1000 | grep ERROR | wc -l)
        if [[ $error_rate -gt $threshold ]]; then
            echo "High error rate detected: $error_rate"
            return 1
        fi
        sleep 1
    done
    return 0
}

if ! monitor_errors "myapp" 5; then
    echo "Errors detected, rolling back to blue"
    kubectl patch service myapp \
      -p '{"spec":{"selector":{"version":"'$BLUE_VERSION'"}}}'
    exit 1
fi

# Step 5: Keep blue as backup or clean up
echo "Deployment successful, keeping blue as backup"
```

---

## Advanced Topics

### Q: Explain etcd consistency and impact on cluster reliability

**A:**
```
Etcd Consistency Model:
- Linearizable reads: Latest data guaranteed
- Sequential writes: Ordered transactions
- Watch semantics: All notifications delivered

Production considerations:
1. Enable etcd backups
2. Monitor etcd latency (<100ms healthy)
3. Use PV for persistence
4. Implement multi-master HA
5. Regular disaster recovery drills
```

---

## Summary

**Critical Senior K8s Skills:**
1. ✅ Multi-tenancy with RBAC & quotas
2. ✅ High availability patterns
3. ✅ Network policies and security
4. ✅ Resource optimization
5. ✅ Advanced troubleshooting
6. ✅ Production-grade deployments
7. ✅ Disaster recovery
8. ✅ Cluster upgrades
