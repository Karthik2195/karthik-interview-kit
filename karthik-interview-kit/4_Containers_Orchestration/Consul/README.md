# Consul - Service Mesh & Networking Platform

## What is it used for?
Consul is used for:
- **Service discovery**: Automatic service registration and discovery
- **Health checking**: Monitor service health
- **Configuration management**: Distributed configuration
- **Multi-datacenter**: Connect across datacenters
- **Service mesh**: Enable service-to-service communication
- **DNS interface**: Built-in DNS
- **Key-value store**: Distributed data store
- **Access control**: Fine-grained access policies

## Installation

```bash
# macOS
brew install consul

# Linux
wget https://releases.hashicorp.com/consul/1.16.0/consul_1.16.0_linux_amd64.zip
unzip consul_1.16.0_linux_amd64.zip
sudo mv consul /usr/local/bin/

# Windows
# Download from https://www.consul.io/downloads

# Docker
docker run -d --name consul consul agent -server -ui -bootstrap-expect=1 -client=0.0.0.0

# Verify installation
consul version
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check Consul version
consul version

# Start Consul server
consul agent -server -ui -bootstrap-expect=1

# Start Consul client
consul agent -ui

# Join cluster
consul join <server-ip>

# List members
consul members

# Register service
cat > service.json << 'EOF'
{
  "service": {
    "name": "web",
    "id": "web-1",
    "port": 8080,
    "check": {
      "http": "http://localhost:8080/health",
      "interval": "10s"
    }
  }
}
EOF

consul services register service.json

# List services
consul catalog services

# Get service details
consul catalog service web

# Query DNS
dig @localhost -p 8600 web.service.consul

# Key-value operations
consul kv put config/app/setting "value"
consul kv get config/app/setting
consul kv list config/app

# Access HTTP API
curl http://localhost:8500/v1/catalog/services

# Get service health
curl http://localhost:8500/v1/health/service/web

# View UI
# http://localhost:8500/ui
```

### Service Configuration
```json
{
  "service": {
    "name": "api",
    "id": "api-1",
    "port": 3000,
    "address": "192.168.1.100",
    "tags": ["v1", "primary"],
    "check": {
      "http": "http://localhost:3000/health",
      "interval": "10s",
      "timeout": "5s",
      "deregister_critical_service_after": "30s"
    },
    "weights": {
      "passing": 10,
      "warning": 5
    }
  }
}
```

### Access Control Lists (ACL)
```bash
# Create ACL token
consul acl token create \
  --description "Service token" \
  --service-identity web

# Apply ACL policy
consul acl policy create \
  --name service-policy \
  --description "Policy for services" \
  --rules @policy.hcl

# List tokens
consul acl token list

# Update service with token
consul services register \
  -token=<token> \
  service.json
```

### Common Issues & Resolution

**Issue: Service not registering**
```bash
# Solution: Check Consul logs
consul monitor

# Verify service configuration
consul validate service.json

# Check service status
consul catalog service <service-name>

# Deregister and re-register
consul services deregister -id=<service-id>
consul services register service.json
```

**Issue: Health check failing**
```bash
# Solution: Test endpoint directly
curl http://localhost:8080/health

# Check health status
consul health service <service-name>

# View check details
curl http://localhost:8500/v1/health/checks/<service-name>

# Increase timeout
# Update check interval and timeout in service.json
```

**Issue: Cannot connect to cluster**
```bash
# Solution: Check cluster membership
consul members

# View leader
consul operator raft list-peers

# Troubleshoot connectivity
consul troubleshoot upstream service-name

# Check network
ping <server-ip>
```

### Debugging
```bash
# Enable debug mode
consul agent -debug

# Monitor logs
consul monitor -log-level=debug

# View metrics
curl http://localhost:8500/v1/agent/metrics

# Check configuration
consul config list

# DNS debug
dig @localhost -p 8600 web.service.consul +short
nslookup web.service.consul localhost:8600

# Trace API calls
curl -v http://localhost:8500/v1/catalog/services
```

### Service Mesh (Consul Connect)
```bash
# Enable Connect
consul agent -server -ui

# Register service with Connect
cat > service-connect.json << 'EOF'
{
  "service": {
    "name": "web",
    "port": 8080,
    "connect": {}
  }
}
EOF

# View intentions
consul intention list

# Create intention
consul intention create web api
```

### Key-Value Store
```bash
# Store configuration
consul kv put app/config '{"debug": true}'

# Read configuration
consul kv get app/config

# Watch for changes
consul watch -type=key -key=app/config

# List all keys
consul kv export > backup.json

# Restore from backup
consul kv import @backup.json
```

### Multi-Datacenter
```bash
# Join secondary datacenter
consul join -wan <primary-dc-ip>

# View federation
consul members -wan

# Register service in different datacenter
# Service registered in DC1 is accessible from DC2

# Query across datacenters
curl http://localhost:8500/v1/catalog/service/web?dc=dc1
```

### Networking
```bash
# List network interfaces
consul members -detailed

# View network status
curl http://localhost:8500/v1/status/leader
curl http://localhost:8500/v1/status/peers

# Check agent configuration
curl http://localhost:8500/v1/agent/self | jq .Config
```
