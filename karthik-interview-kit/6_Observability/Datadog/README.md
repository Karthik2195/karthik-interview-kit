# Datadog - Full-Stack Monitoring & Observability Platform

## What is it used for?
Datadog is used for:
- **Infrastructure monitoring**: Monitor servers, containers, and cloud infrastructure
- **APM (Application Performance Monitoring)**: Track application performance and dependencies
- **Distributed tracing**: Monitor requests across microservices
- **Log aggregation**: Collect and analyze logs from all sources
- **Real User Monitoring (RUM)**: Monitor actual user experience
- **Alerting & notifications**: Intelligent alerting with multiple channels
- **Cost monitoring**: Track cloud spend and optimization
- **Security monitoring**: Detect threats and security anomalies

## Installation

```bash
# Install Datadog Agent on Linux
bash -c "$(curl -L https://s3.amazonaws.com/dd-agent/scripts/install_agent.sh)"

# Or with package manager (Ubuntu/Debian)
sudo apt-get install datadog-agent

# Or with package manager (RHEL/CentOS)
sudo yum install datadog-agent

# Start the agent
sudo systemctl start datadog-agent
sudo systemctl enable datadog-agent

# Verify agent status
sudo systemctl status datadog-agent

# Verify installation
sudo datadog-agent status
```

## Docker Installation

```dockerfile
# Docker
docker run -d \
  --name datadog \
  -e DD_API_KEY=<your-api-key> \
  -e DD_SITE=datadoghq.com \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v /proc/:/host/proc/:ro \
  -v /sys/fs/cgroup/:/host/sys/fs/cgroup:ro \
  datadog/agent:latest
```

## Kubernetes Installation

```bash
# Using Helm
helm repo add datadog https://helm.datadoghq.com
helm install datadog datadog/datadog --set datadog.apiKey=<your-api-key>

# Verify installation
kubectl get pods | grep datadog
kubectl logs -f deploy/datadog-agent
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check agent status
sudo datadog-agent status

# View agent configuration
sudo datadog-agent config

# Check integrations
sudo datadog-agent check integration_name

# Verify connectivity
sudo datadog-agent check datadog_agent

# View logs
sudo tail -f /var/log/datadog/agent.log

# Restart agent
sudo systemctl restart datadog-agent

# Flush logs manually
sudo datadog-agent flare
```

## Configuration

```yaml
# /etc/datadog-agent/datadog.yaml

api_key: <your-api-key>
site: datadoghq.com

# Enable specific integrations
logs_enabled: true
apm_enabled: true
process_config:
  enabled: true

# Custom tags
tags:
  - env:production
  - service:payment
  - team:backend

# Exclude patterns
exclude_paths:
  - /var/log/datadog/agent.log
  - /var/log/syslog
```

## Interview Questions Reference
For senior-level interview questions and answers, see [8_Interview_Questions/Observability/Datadog.md](../../8_Interview_Questions/Observability/Datadog.md)
