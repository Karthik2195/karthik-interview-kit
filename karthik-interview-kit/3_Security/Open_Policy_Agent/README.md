# Open Policy Agent (OPA) - Policy as Code Engine

## What is it used for?
OPA is used for:
- **Policy enforcement**: Enforce organizational policies
- **Authorization**: Fine-grained access control
- **Configuration validation**: Validate configurations
- **Kubernetes admission control**: Enforce pod policies
- **Infrastructure compliance**: Ensure compliance
- **Multi-cloud governance**: Policy across clouds
- **Real-time decisions**: Fast policy evaluation
- **Audit trails**: Track policy decisions

## Installation

```bash
# macOS
brew install opa

# Linux
curl -L -o opa https://openpolicyagent.org/downloads/latest/opa_linux_amd64
chmod +x opa
sudo mv opa /usr/local/bin/

# Windows
# Download from https://openpolicyagent.org/docs/latest/#running-opa

# Verify installation
opa version

# Start OPA server
opa run -s
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check OPA version
opa version

# Run OPA REPL
opa

# Test policy
opa test -v policy.rego

# Evaluate policy
opa eval -d policy.rego 'data.example.allow'

# Compile policy
opa build -b policy.rego

# Format Rego files
opa fmt -w *.rego

# Deploy OPA
opa run -s -l localhost:8181

# Query OPA API
curl http://localhost:8181/v1/data
```

### Basic Rego Policy
```rego
# example.rego
package example

# Simple allow rule
allow {
    input.user.role == "admin"
}

# Deny rule
deny[msg] {
    input.user.age < 18
    msg := "User must be 18 or older"
}

# Complex rules
allow {
    input.method == "GET"
    input.path == "/public"
}

allow {
    input.method == "GET"
    input.path == "/users"
    input.user.authenticated
}

# Conditional allow
allow {
    input.user.role in ["admin", "moderator"]
    input.action == "delete"
}
```

### Kubernetes Integration
```yaml
# apiVersion: constraints.gatekeeper.sh/v1beta1
# kind: ConstraintTemplate
# metadata:
#   name: k8srequiredlabels
# spec:
#   crd:
#     spec:
#       names:
#         kind: K8sRequiredLabels
#       validation:
#         openAPIV3Schema:
#           type: object
#           properties:
#             labels:
#               type: array
#               items:
#                 type: string
#   targets:
#     - target: admission.k8s.gatekeeper.sh
#       rego: |
#         package k8srequiredlabels
#         deny[msg] {
#             not input.review.object.metadata.labels
#             msg := "Metadata labels required"
#         }

# Install Gatekeeper
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.12/deploy/gatekeeper.yaml
```

### Policy Testing
```bash
# Create test file
cat > policy_test.rego << 'EOF'
package example

test_allow_admin {
    allow with input as {"user": {"role": "admin"}}
}

test_deny_minor {
    deny[msg] with input as {"user": {"age": 16}}
    msg == "User must be 18 or older"
}
EOF

# Run tests
opa test -v policy_test.rego
```

### Data Management
```bash
# Load data
opa eval -d data.json -d policy.rego 'data.example'

# Query with data
curl -X POST http://localhost:8181/v1/data/example \
  -H "Content-Type: application/json" \
  -d '{"input": {"user": {"role": "admin"}}}'

# Bundle policies
opa build -b policy.rego -o bundle.tar.gz
```

### Common Issues & Resolution

**Issue: Policy evaluation returns undefined**
```bash
# Solution: Check policy package
opa eval -d policy.rego 'data.example'

# Debug evaluation
opa eval -d policy.rego -f pretty 'data.example'

# Check policy syntax
opa fmt -c policy.rego
```

**Issue: Kubernetes admission webhook fails**
```bash
# Solution: Check webhook configuration
kubectl get validatingwebhookconfigurations

# View webhook logs
kubectl logs -n gatekeeper-system -l control-plane=controller-manager

# Test webhook
curl -X POST http://localhost:8181/v1/data/kubernetes/admission
```

**Issue: Policy performance slow**
```bash
# Solution: Optimize policy
# Use indexing:
admin[id] {
    data.users[id].role == "admin"
}

# Use early exit
allow {
    input.user.role == "public"
    fail
}
```

### Debugging
```bash
# Enable debug mode
opa run -s -l localhost:8181 --log-level=debug

# Trace evaluation
opa eval -d policy.rego -t 'data.example'

# Profile policy
opa eval -d policy.rego --profile 'data.example'

# View data storage
opa eval -d data.json 'data'

# Check bundle contents
tar -tzf bundle.tar.gz
```

### Policy Examples

**Network Policy**
```rego
package network

allow_traffic {
    input.source.namespace == "trusted"
    input.destination.port == 443
    input.protocol == "TCP"
}

deny_traffic[msg] {
    input.source.namespace == "untrusted"
    input.destination.port in [22, 23]
    msg := sprintf("SSH not allowed from %s", [input.source.namespace])
}
```

**Resource Quota Policy**
```rego
package resources

deny[msg] {
    input.resource.requests.cpu > "2000m"
    msg := "CPU request exceeds 2000m"
}

deny[msg] {
    input.resource.requests.memory > "4Gi"
    msg := "Memory request exceeds 4Gi"
}
```

**Compliance Policy**
```rego
package compliance

deny[msg] {
    not input.metadata.labels.environment
    msg := "environment label required"
}

deny[msg] {
    not input.metadata.labels.owner
    msg := "owner label required"
}

deny[msg] {
    input.spec.securityContext.runAsRoot == true
    msg := "Running as root not allowed"
}
```

### Advanced Features
```bash
# HTTP Server
opa run -s \
  --bundle /path/to/bundle \
  --set decision_logs.console=true

# Distributed Tracing
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

# RBAC with OPA
# Define roles, permissions, and resources
# Query: can(user, action, resource)
```
