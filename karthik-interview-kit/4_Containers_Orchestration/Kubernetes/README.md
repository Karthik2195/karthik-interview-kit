# Kubernetes - Container Orchestration Platform

## What is it used for?
Kubernetes is used for:
- **Container orchestration**: Automate deployment, scaling, and management
- **Load balancing**: Distribute traffic across containers
- **Self-healing**: Automatically restart failed containers
- **Auto-scaling**: Scale applications based on demand
- **Rolling updates**: Update applications with zero downtime
- **Resource management**: Efficiently allocate CPU and memory
- **Multi-cloud deployment**: Run on any cloud provider

## Installation

```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install Minikube (for local development)
curl -LO https://github.com/kubernetes/minikube/releases/download/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start Minikube
minikube start

# Install Helm (package manager)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check Kubernetes version
kubectl version

# Get cluster info
kubectl cluster-info

# List nodes
kubectl get nodes

# List all pods
kubectl get pods
kubectl get pods -A  # All namespaces

# List services
kubectl get services

# List deployments
kubectl get deployments

# Create deployment
kubectl create deployment nginx --image=nginx

# Scale deployment
kubectl scale deployment nginx --replicas=3

# Expose service
kubectl expose deployment nginx --type=LoadBalancer --port=80

# Apply manifest file
kubectl apply -f deployment.yaml

# View logs
kubectl logs pod-name
kubectl logs pod-name -c container-name

# Execute command in pod
kubectl exec -it pod-name -- /bin/bash

# Describe resource
kubectl describe pod pod-name
kubectl describe deployment deployment-name

# Delete pod
kubectl delete pod pod-name

# Delete deployment
kubectl delete deployment deployment-name

# Get namespaces
kubectl get namespaces

# Create namespace
kubectl create namespace my-namespace

# Switch namespace
kubectl config set-context --current --namespace=my-namespace

# Port forwarding
kubectl port-forward pod-name 8080:8080
```

### Manifest Examples
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80

---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

### Common Issues & Resolution

**Issue: Pod stuck in Pending state**
```bash
# Solution: Check pod events
kubectl describe pod pod-name

# Check resource availability
kubectl top nodes
kubectl top pods

# Check node status
kubectl get nodes -o wide
```

**Issue: CrashLoopBackOff status**
```bash
# Solution: Check logs
kubectl logs pod-name
kubectl logs pod-name --previous  # Previous instance

# Debug with temporary pod
kubectl run -it --image=busybox debug --restart=Never -- sh
```

**Issue: ImagePullBackOff**
```bash
# Solution: Verify image name and registry
kubectl describe pod pod-name

# Check image availability
kubectl get events -A | grep -i pull

# For private registry, create secret
kubectl create secret docker-registry my-secret \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=password
```

**Issue: Service not accessible**
```bash
# Solution: Check service and endpoints
kubectl get service
kubectl get endpoints

# Check if pods are running
kubectl get pods

# Port forward for testing
kubectl port-forward service/service-name 8080:80
```

**Issue: Node not ready**
```bash
# Solution: Check node status
kubectl describe node node-name

# Check kubelet logs
journalctl -u kubelet -n 50

# Drain and uncordon node
kubectl drain node-name
kubectl uncordon node-name
```

### Debugging Commands
```bash
# Get events for troubleshooting
kubectl get events -A

# Check resource usage
kubectl top nodes
kubectl top pods

# Inspect pod details
kubectl describe pod pod-name

# Get yaml of running resource
kubectl get pod pod-name -o yaml

# Watch pod status
kubectl get pods --watch

# Check pod status across namespaces
kubectl get pods -A -o wide

# View container processes
kubectl exec pod-name -- ps aux

# Check environment variables
kubectl exec pod-name -- env

# View container file system
kubectl exec pod-name -- ls -la /

# Run interactive shell in pod
kubectl exec -it pod-name -- /bin/sh
```

### Configuration Management
```bash
# Get current context
kubectl config current-context

# List contexts
kubectl config get-contexts

# Switch context
kubectl config use-context context-name

# View kubeconfig
kubectl config view

# Set cluster configuration
kubectl config set-cluster cluster-name --server=https://example.com:6443
```

### Advanced Operations
```bash
# Scale statefulset
kubectl scale statefulset statefulset-name --replicas=5

# Update deployment image
kubectl set image deployment/nginx nginx=nginx:1.20

# Rollout history
kubectl rollout history deployment/nginx

# Rollback deployment
kubectl rollout undo deployment/nginx

# Apply all manifests in directory
kubectl apply -f ./manifests/

# Dry-run before applying
kubectl apply -f manifest.yaml --dry-run=client
```
