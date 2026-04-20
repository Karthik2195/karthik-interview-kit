# GitOps - Infrastructure as Code with Git

## What is it used for?
GitOps is used for:
- **Infrastructure management**: Declare desired state in Git
- **Continuous deployment**: Auto-deploy on Git changes
- **Version control**: Full audit trail of changes
- **Rollback**: Easy rollback by reverting commits
- **Consistency**: Ensure cluster matches Git state
- **Collaboration**: Team-based infrastructure changes
- **Multi-cluster**: Manage multiple clusters
- **Policy enforcement**: Git-based policy control

## Installation & Setup

```bash
# Install Flux
curl -s https://fluxcd.io/install.sh | sudo bash

# Install Argo CD
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

# Initialize GitOps repository
git clone https://github.com/your-org/gitops-repo.git
cd gitops-repo
git init
git add .
git commit -m "Initial commit"
git push origin main
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Flux bootstrap
flux bootstrap github \
  --owner=your-org \
  --repository=gitops-repo \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal

# Flux sync status
flux get all --all-namespaces

# Force sync
flux reconcile source git gitops-repo

# View logs
flux logs -f --all-namespaces

# Argo CD login
argocd login argocd.example.com

# Create Argo application
argocd app create myapp \
  --repo https://github.com/your-org/app-repo.git \
  --path helm \
  --dest-server https://kubernetes.default.svc

# Sync Argo application
argocd app sync myapp

# View Argo applications
argocd app list

# Delete Argo application
argocd app delete myapp
```

### Repository Structure
```
gitops-repo/
├── clusters/
│   ├── my-cluster/
│   │   ├── core/
│   │   │   ├── namespaces.yaml
│   │   │   └── rbac.yaml
│   │   ├── apps/
│   │   │   ├── nginx/
│   │   │   │   ├── kustomization.yaml
│   │   │   │   └── namespace.yaml
│   │   │   └── app-repo.yaml
│   │   └── flux-system/
│   │       └── gotk-components.yaml
│   └── production/
│       └── ...
├── apps/
│   ├── nginx/
│   │   ├── base/
│   │   │   ├── kustomization.yaml
│   │   │   ├── deployment.yaml
│   │   │   └── service.yaml
│   │   └── overlays/
│   │       ├── dev/
│   │       └── prod/
│   └── myapp/
│       └── ...
└── README.md
```

### Flux Configuration
```yaml
# clusters/my-cluster/apps/app-repo.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: app-repo
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/your-org/app-repo.git
  ref:
    branch: main

---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: app-repo
  path: ./k8s
  prune: true
  wait: true
  timeout: 5m
```

### Argo CD Configuration
```yaml
# apps/myapp/argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/app-repo.git
    targetRevision: main
    path: helm/myapp
    helm:
      values: |
        image:
          tag: 1.0.0
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Common Issues & Resolution

**Issue: Flux sync fails**
```bash
# Solution: Check Flux status
flux get helmrelease --all-namespaces

# View Flux logs
flux logs --all-namespaces

# Reconcile manually
flux reconcile source git app-repo

# Check Git credentials
flux secret list -n flux-system
```

**Issue: Argo CD app out of sync**
```bash
# Solution: Check app status
argocd app status myapp

# View diff
argocd app diff myapp

# Refresh app
argocd app refresh myapp

# Sync app
argocd app sync myapp

# Force sync
argocd app sync myapp --force
```

**Issue: Permissions denied on Git**
```bash
# Solution: Update SSH credentials
flux create secret git my-secret \
  --url=ssh://git@github.com/your-org/repo.git \
  --private-key-file=~/.ssh/id_rsa

# For HTTPS
flux create secret git my-secret \
  --url=https://github.com/your-org/repo.git \
  --username=<username> \
  --password=<token>
```

### Debugging
```bash
# Check Git connectivity
flux check

# View Kustomization status
kubectl get kustomizations -n flux-system -o wide

# View HelmRelease status
kubectl get helmreleases -n flux-system -o wide

# Get events
kubectl describe kustomization myapp -n flux-system

# Tail Flux logs
flux logs -f --all-namespaces

# Argo CD health check
argocd admin dashboard

# View Argo logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server -f
```

### Best Practices
```yaml
# Use GitOps-friendly repository structure
# - Keep configurations in YAML
# - Use Kustomize or Helm for templating
# - Separate concerns (apps, infrastructure, policies)
# - Document deployment process

# Enforce reviews
# - Require pull request reviews before merge
# - Run automated checks on PR

# Monitoring
# - Monitor Git sync status
# - Alert on sync failures
# - Track deployment metrics

# Security
# - Use sealed secrets for sensitive data
# - Implement RBAC policies
# - Audit all changes via Git history
```

### Advanced Features
```bash
# Multi-cluster GitOps
# Use Flux/Argo across multiple clusters
# Sync different apps to different clusters

# Progressive delivery
# Use Flagger for canary deployments
# Use Argo Rollouts for deployment strategies

# Policy enforcement
# Use OPA with GitOps
# Enforce policies on Git changes

# Notifications
# Configure Slack/Teams notifications
# Alert on deployment failures
```
