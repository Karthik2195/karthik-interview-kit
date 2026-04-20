# Prisma - Container Security Platform

## What is it used for?
Prisma is used for:
- **Container security**: Scan containers for vulnerabilities
- **Compliance**: Ensure compliance with standards
- **Runtime protection**: Monitor container behavior at runtime
- **Supply chain security**: Secure software supply chain
- **Vulnerability management**: Track and remediate vulnerabilities
- **Access control**: Enforce container access policies
- **Secrets management**: Detect and protect secrets
- **Incident response**: Respond to security incidents

## Installation

```bash
# Install Prisma Cloud CLI
curl -L https://api.twistlock.com/v2/api/v1/util/twistcli_download/linux/twistcli -o twistcli
chmod +x twistcli
sudo mv twistcli /usr/local/bin/

# Docker integration
docker scan <image>  # If Prisma enabled

# Deploy on Kubernetes
helm repo add twistlock https://twistlock.azurecr.io/helm/v1
helm install prisma twistlock/twistlock \
  --set console.exposeService=LoadBalancer

# Verify installation
twistcli --version
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Scan image for vulnerabilities
twistcli images scan --address https://console.example.com \
  --user <username> --password <password> \
  myimage:latest

# Scan container registry
twistcli images scan-registry \
  --address https://console.example.com \
  --registry-type docker \
  --registry docker.io

# Policy validation
twistcli images validate --address https://console.example.com \
  myimage:latest

# Console status
curl -k -u <user>:<pass> https://console.example.com/api/v1/status

# List vulnerabilities
curl -k -u <user>:<pass> \
  https://console.example.com/api/v1/images

# View runtime alerts
curl -k -u <user>:<pass> \
  https://console.example.com/api/v1/runtime/alerts
```

### Policy Configuration
```yaml
# Container runtime policy
apiVersion: k8s.tcwp.io/v1
kind: ContainerRuntimePolicy
metadata:
  name: runtime-policy
spec:
  capabilities: "audit"
  processes:
    - ps
    - ls
  network:
    - 10.0.0.0/8
  files:
    - /etc/passwd
```

### Scanning Configuration
```bash
# Create scanning policy
# Web Console -> Vulnerability -> Scanning -> Create

# Scan registry periodically
# Web Console -> Registries -> Schedule Scan

# Set compliance standards
# Web Console -> Compliance -> Select Standards
# CIS, PCI-DSS, HIPAA, etc.
```

### Common Issues & Resolution

**Issue: Scanning fails with authentication error**
```bash
# Solution: Verify credentials
twistcli images scan --address https://console.example.com \
  --user <username> --password <password> \
  --dry-run

# Test console connectivity
curl -k -u <user>:<pass> https://console.example.com/api/v1/status

# Check user permissions
# Web Console -> User Management -> Verify Roles
```

**Issue: Runtime agent not connecting**
```bash
# Solution: Check agent status
kubectl get daemonset -n twistlock

# View agent logs
kubectl logs -n twistlock -l app=twistlock-defender

# Verify network connectivity
kubectl exec -it <defender-pod> -n twistlock \
  -- curl -k https://console.example.com

# Restart agents
kubectl rollout restart daemonset/twistlock-defender -n twistlock
```

**Issue: Vulnerability database not updating**
```bash
# Solution: Check database sync
# Web Console -> System -> Updates

# Trigger manual update
curl -k -u <user>:<pass> -X POST \
  https://console.example.com/api/v1/feeds/update

# Check feed status
curl -k -u <user>:<pass> \
  https://console.example.com/api/v1/feeds/status
```

### Compliance Scanning
```bash
# CIS benchmark scan
twistcli images scan --address https://console.example.com \
  --user <username> --password <password> \
  --compliance cis \
  myimage:latest

# View compliance report
curl -k -u <user>:<pass> \
  https://console.example.com/api/v1/compliance

# Export compliance report
curl -k -u <user>:<pass> \
  https://console.example.com/api/v1/compliance/export \
  -H "Accept: application/pdf" > report.pdf
```

### Runtime Protection
```bash
# Enable runtime protection
# Web Console -> Runtime -> Create Rule

# Monitor process execution
# Web Console -> Runtime -> Incidents -> View Details

# Block suspicious processes
# Web Console -> Runtime -> Rules -> Add Block Rule

# View runtime metrics
curl -k -u <user>:<pass> \
  https://console.example.com/api/v1/runtime/metrics
```

### Secrets Detection
```bash
# Scan for secrets in image
twistcli images scan --address https://console.example.com \
  --user <username> --password <password> \
  --secrets-scan \
  myimage:latest

# View detected secrets
curl -k -u <user>:<pass> \
  https://console.example.com/api/v1/secrets

# Remediate secrets
# Update image, remove secrets, rebuild
```

### Debugging
```bash
# Enable debug logging
# Web Console -> System -> Debug Logging

# View console logs
docker logs <console-container>

# View defender logs
docker logs <defender-container>

# Test scanning
twistcli images scan --address https://console.example.com \
  --user <username> --password <password> \
  --dry-run \
  myimage:latest

# API test
curl -k -I https://console.example.com/api/v1/status

# Check connectivity
telnet console.example.com 8084
```

### Integration Examples

**GitHub Integration**
```bash
# Enable GitHub scanning
# Web Console -> Integrations -> GitHub

# Scan pull requests automatically
# Repository settings -> Configure Prisma
```

**CI/CD Pipeline Integration**
```bash
# Jenkins plugin
# Install: Jenkins -> Manage Plugins -> Search "Twistlock"

# GitLab integration
# .gitlab-ci.yml
scan_image:
  script:
    - twistcli images scan --address $PRISMA_CONSOLE \
      --user $PRISMA_USER --password $PRISMA_PASSWORD \
      $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

**Kubernetes Policy**
```yaml
# Deploy with Prisma enforcement
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  labels:
    prisma: enabled
spec:
  containers:
  - name: app
    image: myimage:latest
    securityContext:
      runAsNonRoot: true
      readOnlyRootFilesystem: true
```

### Advanced Features
```bash
# Machine learning-based anomaly detection
# Enabled by default

# Behavioral threat protection
# Web Console -> Runtime -> Advanced

# Custom policy rules
# Web Console -> Policies -> Custom Rules

# Incident correlation
# Web Console -> Incidents -> Timeline
```
