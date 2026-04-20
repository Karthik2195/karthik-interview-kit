# Spinnaker - Multi-Cloud Deployment Platform

## What is it used for?
Spinnaker is used for:
- **Multi-cloud deployments**: Deploy to AWS, Azure, GCP, Kubernetes
- **Continuous delivery**: Automate deployment pipelines
- **Canary deployments**: Safely roll out changes
- **Blue-green deployments**: Zero-downtime deployments
- **Pipeline orchestration**: Complex deployment workflows
- **Infrastructure as code**: Manage infrastructure changes
- **Rollback**: Quick rollback capabilities
- **Multi-region**: Deploy across regions

## Installation

```bash
# Install Spinnaker using Halyard
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install.sh
sudo bash install.sh --user ubuntu

# Verify installation
~/.local/bin/hal --version

# Create config directory
mkdir -p ~/.config/spinnaker

# Install Spinnaker
~/.local/bin/hal config version edit --version 1.28.0

# Deploy Spinnaker
~/.local/bin/hal deploy apply

# Or use Docker
docker run -d \
  -p 9000:9000 \
  -v /opt/spinnaker/config:/opt/spinnaker/config \
  spinnaker/deck:latest
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check Spinnaker version
~/.local/bin/hal version list

# Configure storage backend (S3)
~/.local/bin/hal config storage s3 edit \
  --access-key-id <KEY> \
  --secret-access-key <SECRET> \
  --region us-east-1

# Set storage backend
~/.local/bin/hal config storage edit --type s3

# Configure cloud provider (Kubernetes)
~/.local/bin/hal config provider kubernetes enable

# Add Kubernetes account
~/.local/bin/hal config provider kubernetes account add \
  --provider-version v2 \
  --kubeconfig-file ~/.kube/config \
  --context-name kubernetes-master \
  my-k8s-account

# Get service status
~/.local/bin/hal deploy current

# View logs
~/.local/bin/hal log tail

# Access Spinnaker UI
# http://localhost:9000
```

### Pipeline Configuration
```json
{
  "application": "myapp",
  "name": "Deploy to Production",
  "stages": [
    {
      "name": "Build",
      "type": "jenkins",
      "config": {
        "job": "myapp-build",
        "master": "jenkins-master"
      }
    },
    {
      "name": "Deploy to Staging",
      "type": "deploy",
      "config": {
        "cloudProvider": "kubernetes",
        "account": "my-k8s-account",
        "namespace": "staging",
        "manifest": {
          "apiVersion": "apps/v1",
          "kind": "Deployment",
          "metadata": {
            "name": "myapp"
          },
          "spec": {
            "replicas": 2,
            "selector": {
              "matchLabels": {
                "app": "myapp"
              }
            },
            "template": {
              "metadata": {
                "labels": {
                  "app": "myapp"
                }
              },
              "spec": {
                "containers": [
                  {
                    "name": "myapp",
                    "image": "myapp:latest"
                  }
                ]
              }
            }
          }
        }
      }
    },
    {
      "name": "Canary Deployment",
      "type": "canary",
      "config": {
        "canaryConfig": {
          "canaryAnalysisInterval": 600,
          "canaryMetricSetQueryId": "prod"
        }
      }
    }
  ]
}
```

### Common Issues & Resolution

**Issue: Cannot connect to Kubernetes**
```bash
# Solution: Verify kubeconfig
kubectl config current-context
kubectl get nodes

# Update kubeconfig path in Spinnaker
~/.local/bin/hal config provider kubernetes account edit \
  my-k8s-account \
  --kubeconfig-file ~/.kube/config

# Deploy changes
~/.local/bin/hal deploy apply

# Check account credentials
~/.local/bin/hal config provider kubernetes account list
```

**Issue: Pipeline fails to deploy**
```bash
# Solution: Check pipeline execution logs
# Spinnaker UI -> Applications -> Executions -> Click execution

# Verify manifests
kubectl apply -f manifest.yaml --dry-run=client

# Check account permissions
kubectl auth can-i create deployments --as=system:serviceaccount:default:spinnaker

# View Spinnaker logs
~/.local/bin/hal log tail clouddriver
~/.local/bin/hal log tail orca
```

**Issue: Storage backend error**
```bash
# Solution: Verify S3 credentials
aws s3 ls

# Update S3 configuration
~/.local/bin/hal config storage s3 edit \
  --access-key-id <NEW_KEY> \
  --secret-access-key <NEW_SECRET>

# Deploy changes
~/.local/bin/hal deploy apply
```

### Canary Analysis
```bash
# Configure Prometheus for canary analysis
~/.local/bin/hal config metricStore prometheus enable

# Set Prometheus URL
~/.local/bin/hal config metricStore prometheus edit \
  --address http://prometheus:9090

# Create canary config
# Spinnaker UI -> Applications -> Config -> Canary Config

# Define canary metrics
{
  "name": "production-canary",
  "applications": ["myapp"],
  "metrics": [
    {
      "name": "error_rate",
      "query": "rate(errors_total[5m])"
    },
    {
      "name": "latency_p99",
      "query": "histogram_quantile(0.99, latency_bucket)"
    }
  ]
}
```

### Rolling Back
```bash
# Rollback from UI
# Spinnaker UI -> Deployments -> Right-click -> Rollback

# Or via API
curl -X POST \
  http://localhost:8084/applications/myapp/pipelines/<pipeline-id>/rollback \
  -H "Content-Type: application/json"
```

### Debugging
```bash
# View service status
~/.local/bin/hal deploy current

# Check service health
curl http://localhost:8084/health

# View logs
~/.local/bin/hal log tail

# Debug specific service
~/.local/bin/hal log tail orca
~/.local/bin/hal log tail clouddriver
~/.local/bin/hal log tail deck

# Check configuration
cat ~/.config/spinnaker/config
```

### Advanced Features
```bash
# Traffic management with Istio
# Configure Spinnaker with Istio provider

# Webhook integration
# Trigger pipelines from external events

# Jenkins integration
~/.local/bin/hal config ci jenkins enable
~/.local/bin/hal config ci jenkins master add \
  --address http://jenkins:8080 \
  --username admin \
  --password <password> \
  jenkins-master

# GitHub integration
~/.local/bin/hal config notification slack enable
~/.local/bin/hal config notification slack edit \
  --bot-name Spinnaker \
  --token <slack-token>
```

### Multi-Cloud Deployment
```bash
# Enable AWS
~/.local/bin/hal config provider aws enable

# Add AWS account
~/.local/bin/hal config provider aws account add \
  --account-name my-aws-account \
  --assume-role role/spinnakerRole

# Enable Azure
~/.local/bin/hal config provider azure enable

# Add Azure account
~/.local/bin/hal config provider azure account add \
  --account-name my-azure-account \
  --client-id <client-id> \
  --client-secret <client-secret> \
  --default-key-vault <vault-name>

# Deploy across clouds
# Create pipeline with stages targeting different accounts
```

### Monitoring and Alerts
```bash
# Enable monitoring
~/.local/bin/hal config monitoring enable

# Configure alerts
# Spinnaker UI -> Applications -> Alerts -> Create Alert

# View metrics
# Spinnaker UI -> Applications -> Monitoring
```

### Security
```bash
# Enable authentication
~/.local/bin/hal config security authn enable

# Configure OAuth
~/.local/bin/hal config security authn oauth2 enable
~/.local/bin/hal config security authn oauth2 edit \
  --provider github \
  --client-id <client-id> \
  --client-secret <client-secret>

# RBAC
~/.local/bin/hal config security authz enable
~/.local/bin/hal config security authz role-provider edit \
  --type github

# Deploy secure config
~/.local/bin/hal deploy apply
```
