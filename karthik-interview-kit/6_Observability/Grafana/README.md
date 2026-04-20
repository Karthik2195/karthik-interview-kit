# Grafana - Visualization & Analytics Platform

## What is it used for?
Grafana is used for:
- **Metrics visualization**: Create interactive dashboards
- **Data exploration**: Analyze metrics and logs
- **Alerting**: Set up sophisticated alerts
- **Multi-source support**: Query multiple data sources
- **Real-time monitoring**: Monitor system and application metrics
- **Performance analysis**: Analyze historical data
- **Team collaboration**: Share dashboards across teams
- **Custom dashboards**: Create tailored monitoring views

## Installation

```bash
# Ubuntu/Debian
sudo apt-get install -y adduser libfontconfig1
wget https://dl.grafana.com/oss/release/grafana_10.0.0_amd64.deb
sudo dpkg -i grafana_10.0.0_amd64.deb

# RHEL/CentOS
wget https://dl.grafana.com/oss/release/grafana-10.0.0-1.x86_64.rpm
sudo rpm -ivh grafana-10.0.0-1.x86_64.rpm

# macOS
brew install grafana

# Docker
docker run -d -p 3000:3000 --name=grafana grafana/grafana

# Start Grafana
sudo systemctl start grafana-server
sudo systemctl enable grafana-server

# Access Grafana
# Browser: http://localhost:3000
# Default credentials: admin/admin
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Start Grafana
sudo systemctl start grafana-server

# Stop Grafana
sudo systemctl stop grafana-server

# Restart Grafana
sudo systemctl restart grafana-server

# Check status
sudo systemctl status grafana-server

# View logs
sudo tail -f /var/log/grafana/grafana.log

# Query API for dashboards
curl -H "Authorization: Bearer APITOKEN" http://localhost:3000/api/dashboards

# List datasources via API
curl -H "Authorization: Bearer APITOKEN" http://localhost:3000/api/datasources

# Create dashboard via API
curl -X POST -H "Authorization: Bearer APITOKEN" \
  -H "Content-Type: application/json" \
  -d @dashboard.json \
  http://localhost:3000/api/dashboards/db

# Get dashboard by ID
curl -H "Authorization: Bearer APITOKEN" \
  http://localhost:3000/api/dashboards/uid/DASHBOARD-UID

# Find alert by ID
curl -H "Authorization: Bearer APITOKEN" \
  http://localhost:3000/api/alerts/1
```

### Adding Data Sources
```bash
# Add Prometheus datasource via API
curl -X POST -H "Authorization: Bearer APITOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Prometheus",
    "type": "prometheus",
    "url": "http://prometheus:9090",
    "access": "proxy",
    "isDefault": true
  }' \
  http://localhost:3000/api/datasources

# Add Elasticsearch datasource
curl -X POST -H "Authorization: Bearer APITOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Elasticsearch",
    "type": "elasticsearch",
    "url": "http://elasticsearch:9200",
    "access": "proxy",
    "database": "logs-*"
  }' \
  http://localhost:3000/api/datasources

# List all datasources
curl -H "Authorization: Bearer APITOKEN" \
  http://localhost:3000/api/datasources
```

### Creating Dashboards
```json
{
  "dashboard": {
    "title": "System Monitoring",
    "tags": ["monitoring"],
    "timezone": "UTC",
    "panels": [
      {
        "title": "CPU Usage",
        "targets": [
          {
            "expr": "node_cpu_seconds_total"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Memory Usage",
        "targets": [
          {
            "expr": "node_memory_MemAvailable_bytes"
          }
        ],
        "type": "stat"
      }
    ]
  }
}
```

### Common Issues & Resolution

**Issue: Data source connection failed**
```bash
# Solution: Test connection in UI
# Configuration -> Data Sources -> Edit -> Test

# Or test via curl
curl -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"test"}' \
  http://localhost:3000/api/datasources/test

# Check Grafana logs
sudo tail -f /var/log/grafana/grafana.log

# Verify datasource URL is accessible
curl http://prometheus:9090  # Test connectivity
```

**Issue: Dashboard not saving**
```bash
# Solution: Check Grafana permissions
sudo chown -R grafana:grafana /var/lib/grafana

# Restart Grafana
sudo systemctl restart grafana-server

# Check database connectivity
sudo systemctl status grafana-server

# Check logs for errors
sudo grep -i error /var/log/grafana/grafana.log
```

**Issue: Alerts not triggering**
```bash
# Solution: Enable alerting
# Configuration -> Alerting -> Enable

# Check alert status
curl -H "Authorization: Bearer TOKEN" \
  http://localhost:3000/api/alerts

# Test alert condition
# Go to: Alert Rules -> Test Rule

# Check notification channel
# Configuration -> Notification channels

# Verify alertmanager configuration
```

**Issue: Performance issues / slow dashboards**
```bash
# Solution: Check query performance
# Dashboard Settings -> Inspect -> Query Inspector

# Optimize queries
# Reduce time range, add aggregation

# Check Grafana memory usage
ps aux | grep grafana

# Limit data points
# Panel settings -> Options -> Limit data points

# Use caching
# Configuration -> Data Sources -> Edit -> Cache
```

**Issue: Authentication failed**
```bash
# Solution: Reset admin password
sudo grafana-cli admin reset-admin-password password

# Or edit config
sudo nano /etc/grafana/grafana.ini
# Change: disable_login_form = false

# Restart service
sudo systemctl restart grafana-server
```

### API Authentication
```bash
# Create API token
curl -X POST -H "Content-Type: application/json" \
  -d '{"name":"my-api-token"}' \
  -u admin:admin \
  http://localhost:3000/api/auth/keys

# Use token in requests
curl -H "Authorization: Bearer APITOKEN" \
  http://localhost:3000/api/dashboards

# List API tokens
curl -H "Authorization: Bearer APITOKEN" \
  http://localhost:3000/api/auth/keys

# Revoke token
curl -X DELETE -H "Authorization: Bearer APITOKEN" \
  http://localhost:3000/api/auth/keys/1
```

### Debugging
```bash
# Enable debug logging
# /etc/grafana/grafana.ini
[log]
level = debug

# Restart with debug
sudo systemctl restart grafana-server
sudo tail -f /var/log/grafana/grafana.log

# Test datasource connection
# UI: Configuration -> Data Sources -> Select datasource -> Test

# API datasource test
curl -X POST -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"url":"http://prometheus:9090"}' \
  http://localhost:3000/api/datasources/test

# Check health
curl http://localhost:3000/api/health

# Get version
curl http://localhost:3000/api/health | jq .version

# List installed plugins
curl http://localhost:3000/api/plugins
```

### Common Plugins
```bash
# Install plugin
sudo grafana-cli plugins install grafana-worldmap-panel

# List installed plugins
sudo grafana-cli plugins ls

# Install from URL
sudo grafana-cli --pluginUrl URL plugins install PLUGIN

# Update plugin
sudo grafana-cli plugins update PLUGIN

# Uninstall plugin
sudo grafana-cli plugins uninstall PLUGIN

# Restart after installing
sudo systemctl restart grafana-server
```

### Dashboard Export/Import
```bash
# Export dashboard to JSON
curl -H "Authorization: Bearer TOKEN" \
  http://localhost:3000/api/dashboards/uid/DASHBOARD-UID | jq '.dashboard' > dashboard.json

# Import dashboard
curl -X POST -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"dashboard":'"$(cat dashboard.json)"',"overwrite":true}' \
  http://localhost:3000/api/dashboards/db
```
