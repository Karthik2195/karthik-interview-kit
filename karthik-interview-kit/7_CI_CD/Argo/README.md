# Argo - Workflow Engine for Kubernetes

## What is it used for?
Argo is used for:
- **Workflow orchestration**: Complex multi-step workflows
- **CI/CD pipelines**: Container-native CI/CD
- **Batch processing**: Process large amounts of data
- **Machine learning**: ML pipeline orchestration
- **DAG workflows**: Directed acyclic graph workflows
- **Parallel execution**: Run tasks in parallel
- **Event-driven**: Trigger on events
- **Managed services**: Use Argo as managed service

## Installation

```bash
# Install Argo Workflows
kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.4.0/install.yaml

# Install Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Install Argo Events
kubectl create namespace argo-events
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/install.yaml

# Install Argo CLI
curl -sLO https://github.com/argoproj/argo-workflows/releases/download/v3.4.0/argo-linux-amd64.gz
gunzip argo-linux-amd64.gz
chmod +x argo-linux-amd64
sudo mv argo-linux-amd64 /usr/local/bin/argo
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Submit workflow
argo submit workflow.yaml

# List workflows
argo list

# Get workflow status
argo get workflow-name

# View workflow logs
argo logs workflow-name

# Watch workflow
argo watch workflow-name

# Delete workflow
argo delete workflow-name

# Resubmit workflow
argo resubmit workflow-name

# Stop workflow
argo stop workflow-name
```

### Workflow Definition
```yaml
# workflow.yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-
spec:
  entrypoint: hello
  templates:
  - name: hello
    steps:
    - - name: hello-step
        template: hello-template

  - name: hello-template
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo 'Hello World'"]
```

### DAG Workflow
```yaml
# dag-workflow.yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: dag-
spec:
  entrypoint: dag
  templates:
  - name: dag
    dag:
      tasks:
      - name: task-a
        template: task
        arguments:
          parameters:
          - name: message
            value: "Task A"
      - name: task-b
        template: task
        dependencies: task-a
        arguments:
          parameters:
          - name: message
            value: "Task B"

  - name: task
    inputs:
      parameters:
      - name: message
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo {{inputs.parameters.message}}"]
```

### Common Issues & Resolution

**Issue: Workflow fails with ImagePullBackOff**
```bash
# Solution: Check image availability
kubectl describe pod <pod-name> -n argo

# Verify image exists
docker pull image-name

# Update workflow with correct image
# Or pull image to cluster first

# Check image pull secrets
kubectl get secrets -n argo
```

**Issue: Workflow timeout**
```yaml
# Solution: Set timeout
spec:
  activeDeadlineSeconds: 3600
  ttlStrategy:
    secondsAfterFinished: 300

# Extend timeout for steps
spec:
  templates:
  - name: my-template
    container:
      image: alpine:latest
    activeDeadlineSeconds: 600
```

**Issue: Permission denied**
```bash
# Solution: Check RBAC
kubectl get rolebindings -n argo

# Create service account
kubectl create serviceaccount argo-workflow -n argo

# Grant permissions
kubectl create rolebinding argo-workflow \
  --clusterrole=argo-workflows \
  --serviceaccount=argo:argo-workflow \
  -n argo
```

### Advanced Workflows
```yaml
# Parallel tasks
- name: parallel-tasks
  parallelism: 5
  template: task

# Retry policy
- name: task-with-retry
  template: task
  retryStrategy:
    limit: 3
    backoff:
      duration: 5s
      factor: 2

# Conditional execution
- name: conditional-task
  when: "{{workflow.parameters.debug}} == 'true'"
  template: debug-task

# Artifact passing
- name: artifact-task
  artifacts:
  - name: output
    path: /tmp/output.txt
```

### Debugging
```bash
# Get workflow details
argo get <workflow-name> -o json

# Check pod logs
kubectl logs <pod-name> -n argo

# View events
kubectl get events -n argo

# Describe workflow
kubectl describe workflow <workflow-name> -n argo

# Check node status
argo get <workflow-name> | grep -A 5 Status
```

### Integration with Argo CD
```yaml
# Create Argo CD application for workflows
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-workflows
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argo-workflows.git
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: argo
  syncPolicy:
    automated:
      prune: true
```

### Artifact Management
```yaml
# Define artifacts
spec:
  artifactRepositories:
  - name: default
    s3:
      bucket: my-bucket
      key: argo-artifacts
      endpoint: s3.amazonaws.com
      accessKey:
        name: aws-creds
        key: accessKey
      secretKey:
        name: aws-creds
        key: secretKey

# Use in workflow
- name: artifact-task
  container:
    image: alpine:latest
  artifacts:
  - name: output
    path: /tmp/output.txt
```

### Triggers and Events
```yaml
# Cron trigger
apiVersion: argoproj.io/v1alpha1
kind: CronWorkflow
metadata:
  name: cron-workflow
spec:
  schedule: "0 * * * *"
  concurrencyPolicy: "Allow"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  workflowSpec:
    entrypoint: main
    templates:
    - name: main
      container:
        image: alpine:latest
```
