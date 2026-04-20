# Flux - GitOps for Kubernetes

## What is it used for?
Flux is used for:
- **GitOps**: Declarative infrastructure in Git
- **Continuous deployment**: Auto-deploy on Git changes
- **Multi-tenancy**: Isolate workloads
- **Helm integration**: Deploy Helm charts from Git
- **Kustomize support**: Customize deployments
- **Notifications**: Alert on events
- **Image automation**: Auto-update container images
- **Multi-cluster**: Manage multiple clusters

## Installation

```bash
# Install Flux
curl -s https://fluxcd.io/install.sh | sudo bash

# Or via Homebrew
brew install fluxcd/tap/flux

# Verify installation
flux --version

# Check prerequisites
flux check --pre

# Bootstrap Flux with GitHub
flux bootstrap github \
  --owner=<your-org> \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal

# Bootstrap with GitLab
flux bootstrap gitlab \
  --owner=<your-group> \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check Flux status
flux check

# Get all resources
flux get all --all-namespaces

# Get sources
flux get sources git

# Get Helm charts
flux get helmcharts

# Get Kustomizations
flux get kustomizations

# Sync manually
flux reconcile source git fleet-infra

# Watch reconciliation
flux logs -f --all-namespaces

# Suspend reconciliation
flux suspend kustomization myapp

# Resume reconciliation
flux resume kustomization myapp

# Delete Flux
flux uninstall --keep-namespace
```

### Repository Structure
```
fleet-infra/
├── clusters/
│   └── my-cluster/
│       ├── flux-system/
│       │   ├── gotk-components.yaml
│       │   └── gotk-sync.yaml
│       └── apps/
│           ├── kustomization.yaml
│           ├── nginx/
│           │   └── kustomization.yaml
│           └── database/
│               ├── kustomization.yaml
│               └── helmrelease.yaml
└── README.md
```

### GitRepository Configuration
```yaml
# clusters/my-cluster/apps/git-repo.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: fleet-infra
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/your-org/fleet-infra.git
  ref:
    branch: main
  ignore: |
    # Exclude all files
    /*
    # Include cluster dir
    !/clusters/
    !/clusters/my-cluster/
```

### Kustomization Configuration
```yaml
# clusters/my-cluster/apps/kustomization.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m0s
  path: ./clusters/my-cluster/apps
  prune: true
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  validation: client
  dependsOn:
    - name: core
```

### Helm Integration
```yaml
# HelmRepository
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: stable
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.helm.sh/stable

---
# HelmRelease
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: nginx
  namespace: default
spec:
  interval: 5m
  chart:
    spec:
      chart: nginx-ingress
      sourceRef:
        kind: HelmRepository
        name: stable
      interval: 1m
  values:
    replicaCount: 2
    service:
      type: LoadBalancer
```

### Common Issues & Resolution

**Issue: Source fetch fails**
```bash
# Solution: Check Git connectivity
flux get sources git

# Describe source for details
kubectl describe gitrepository fleet-infra -n flux-system

# Check credentials
flux create secret git my-secret \
  --url=https://github.com/your-org/repo.git \
  --username=<username> \
  --password=<token>
```

**Issue: Kustomization not reconciling**
```bash
# Solution: Check Kustomization status
flux get kustomizations -n flux-system

# Describe resource
kubectl describe kustomization apps -n flux-system

# Check events
kubectl get events -n flux-system

# Force reconciliation
flux reconcile kustomization apps --with-source
```

**Issue: HelmRelease fails**
```bash
# Solution: Check HelmRelease status
kubectl describe helmrelease nginx -n default

# View Helm values being used
kubectl get helmrelease nginx -n default -o jsonpath='{.spec.values}' | jq

# Check Helm repository
flux get sources helm

# Force reconcile
flux reconcile helmrelease nginx --with-source
```

### Image Automation
```yaml
# Enable automated image updates
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: nginx
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: nginx
  policy:
    semver:
      range: ^1.0

---
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m0s
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  git:
    commit:
      author:
        email: flux@example.com
        name: Flux
      messageTemplate: 'Automated image update'
  push:
    branch: main
```

### Notifications
```yaml
# Configure Slack notifications
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: on-deployment-failure
  namespace: flux-system
spec:
  providerRef:
    name: slack
  eventSeverity: error
  eventSources:
  - kind: Kustomization
    name: '*'

---
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: slack
  namespace: flux-system
spec:
  type: slack
  address: https://hooks.slack.com/services/XXX/YYY/ZZZ
```

### Debugging
```bash
# Check Flux logs
flux logs --all-namespaces -f

# Get controller logs
kubectl logs -n flux-system deployment/source-controller -f

# Get kustomize-controller logs
kubectl logs -n flux-system deployment/kustomize-controller -f

# Export logs for analysis
kubectl logs -n flux-system deployment/source-controller > flux-logs.txt

# Describe all resources
kubectl describe all -n flux-system
```

### Best Practices
```bash
# Use separate branches for different environments
clusters/
├── staging/
└── production/

# Implement protection rules
# - Require PR reviews
# - Run CI checks
# - Require status checks to pass

# Monitor reconciliation
flux get all --all-namespaces -o wide

# Use notifications for visibility
# - Alert on failures
# - Track changes

# Implement RBAC
# - Use service accounts
# - Limit permissions
```

### Advanced Features
```yaml
# Cross-cluster replication
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps-prod
  namespace: flux-system
spec:
  # Production specific values
  patches:
  - target:
      kind: Deployment
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 3

# Multi-source deployments
dependsOn:
  - name: infrastructure
  - name: monitoring
```
