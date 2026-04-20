# Crossplane - Kubernetes-Native Infrastructure Management

## What is it used for?
Crossplane is used for:
- **Infrastructure as Code**: Define cloud resources in Kubernetes
- **Multi-cloud support**: Manage resources across cloud providers
- **GitOps**: Git-based infrastructure management
- **Automation**: Automate resource provisioning
- **Policy enforcement**: Apply organizational policies
- **Composition**: Combine resources into abstractions
- **Self-service**: Enable developer self-service
- **Vendor lock-in prevention**: Multi-cloud flexibility

## Installation

```bash
# Install Crossplane
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system --create-namespace

# Install AWS provider
kubectl apply -f - <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/upbound/provider-aws:v0.46.0
EOF

# Verify installation
kubectl get providers
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check Crossplane version
kubectl get deployment crossplane -n crossplane-system

# List providers
kubectl get providers

# List provider configs
kubectl get providerconfigs

# Create AWS provider config
cat > aws-provider-config.yaml << 'EOF'
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-creds
      key: creds
EOF

# Apply provider config
kubectl apply -f aws-provider-config.yaml

# Create resource
kubectl apply -f resource.yaml

# List resources
kubectl get s3buckets

# Describe resource
kubectl describe s3bucket my-bucket

# Delete resource
kubectl delete s3bucket my-bucket

# Get resource details
kubectl get s3bucket -o yaml
```

### AWS S3 Bucket Example
```yaml
apiVersion: s3.aws.crossplane.io/v1beta1
kind: Bucket
metadata:
  name: my-bucket
spec:
  forProvider:
    region: us-east-1
    acl: private
    versioningConfiguration:
      status: Enabled
    tagging:
      tagSet:
        - key: Environment
          value: dev
  providerConfigRef:
    name: default
```

### Composite Resources (XRDs)
```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xbuckets.example.com
spec:
  group: example.com
  names:
    kind: XBucket
    plural: xbuckets
  claimNames:
    kind: Bucket
    plural: buckets

---
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: bucket-composition
spec:
  compositeTypeRef:
    apiVersion: example.com/v1
    kind: XBucket
  resources:
    - name: bucket
      base:
        apiVersion: s3.aws.crossplane.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-1
```

### Common Issues & Resolution

**Issue: Provider not working**
```bash
# Solution: Check provider status
kubectl describe provider provider-aws

# View provider logs
kubectl logs -n crossplane-system -l pkg.crossplane.io/provider=aws

# Verify provider config
kubectl get providerconfigs

# Check AWS credentials
kubectl describe secret aws-creds -n crossplane-system
```

**Issue: Resource creation fails**
```bash
# Solution: Check resource status
kubectl describe s3bucket my-bucket

# View resource conditions
kubectl get s3bucket -o jsonpath='{.items[0].status.conditions}'

# Check provider logs
kubectl logs -n crossplane-system -l pkg.crossplane.io/provider=aws -f

# Validate resource spec
kubectl apply -f resource.yaml --dry-run=client
```

**Issue: Cannot authenticate to AWS**
```bash
# Solution: Verify credentials
kubectl get secret aws-creds -n crossplane-system -o yaml

# Update AWS credentials
kubectl create secret generic aws-creds -n crossplane-system \
  --from-file=creds=$HOME/.aws/credentials \
  --overwrite -o yaml | kubectl apply -f -

# Patch provider config
kubectl patch providerconfig default --type merge \
  -p '{"spec":{"credentials":{"secretRef":{"name":"aws-creds"}}}}'
```

### Debugging
```bash
# Check Crossplane controller logs
kubectl logs -n crossplane-system -l app=crossplane -f

# View provider controller logs
kubectl logs -n crossplane-system -l pkg.crossplane.io/provider=aws -f

# Inspect resource details
kubectl get s3bucket -o yaml

# Check events
kubectl get events -n default

# Describe Crossplane installation
kubectl describe deployment crossplane -n crossplane-system
```

### Multi-Cloud Example
```yaml
# AWS RDS Database
apiVersion: database.aws.crossplane.io/v1beta2
kind: DBInstance
metadata:
  name: my-db
spec:
  forProvider:
    dbName: mydb
    masterUsername: admin
    engine: postgres
    dbInstanceClass: db.t3.micro
  providerConfigRef:
    name: aws

---
# GCP CloudSQL Database
apiVersion: cloudsql.gcp.crossplane.io/v1beta1
kind: CloudSQLInstance
metadata:
  name: my-gcp-db
spec:
  forProvider:
    databaseVersion: POSTGRES_14
    region: us-central1
    settings:
      tier: db-f1-micro
  providerConfigRef:
    name: gcp
```

### Policy Management
```yaml
# Enforce tagging policy
apiVersion: iam.crossplane.io/v1alpha1
kind: RolePolicy
metadata:
  name: enforce-tagging
spec:
  forProvider:
    statement:
      - effect: Allow
        action:
          - 's3:*'
        resource: '*'
        condition:
          - key: 'aws:RequestedRegion'
            values:
              - 'us-east-1'
```

### GitOps Workflow
```bash
# Store manifests in git
git add infrastructure/
git commit -m "Add S3 bucket resource"
git push

# Crossplane syncs from git
# Resources are automatically created/updated based on git
```

### Observability
```bash
# Monitor resource status
kubectl get managed -A --watch

# View resource conditions
kubectl get managed -A -o custom-columns=NAME:.metadata.name,\
READY:.status.conditions[?(@.type=="Ready")].status,\
SYNCED:.status.conditions[?(@.type=="Synced")].status

# Check controller metrics
kubectl port-forward -n crossplane-system svc/crossplane 8080:8080
# Visit http://localhost:8080/metrics
```

### Advanced Features
```bash
# Custom providers
# Build and publish custom Crossplane providers

# Packages
# Use Crossplane packages for distribution

# Configuration packages
# Share XRDs and compositions
```
