# Prometheus - Monitoring & Alerting System

## What is it used for?
Prometheus is used for:
- **Metrics collection**: Collect time-series metrics from applications
- **Alerting**: Generate alerts based on metric thresholds
- **Monitoring**: Monitor system and application health
- **Performance analysis**: Analyze performance metrics
- **Incident response**: Track and respond to incidents
- **Trend analysis**: Understand system behavior over time
- **Multi-dimensional data**: Label-based metric organization

## Installation

```bash
# Download Prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz
tar -xzf prometheus-2.45.0.linux-amd64.tar.gz
cd prometheus-2.45.0.linux-amd64

# Run Prometheus
./prometheus --config.file=prometheus.yml

# Or install with package manager
sudo apt-get install prometheus prometheus-node-exporter

# Start as service
sudo systemctl start prometheus
sudo systemctl enable prometheus

# Verify installation
curl http://localhost:9090
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check Prometheus version
prometheus --version

# Check configuration syntax
prometheus --config.file=prometheus.yml --check

# Start with specific config
prometheus --config.file=/etc/prometheus/prometheus.yml

# Start with web console templates
prometheus --config.file=prometheus.yml --web.console.libraries=consoles --web.console.templates=consoles

# Query Prometheus API
curl 'http://localhost:9090/api/v1/query?query=up'

# Range query (instant vectors over time)
curl 'http://localhost:9090/api/v1/query_range?query=cpu_usage_percent&start=1609459200&end=1609545600&step=60'

# List available metrics
curl 'http://localhost:9090/api/v1/label/__name__/values'

# List label values
curl 'http://localhost:9090/api/v1/label/job/values'

# Alert status
curl 'http://localhost:9090/api/v1/alerts'

# Query status
curl 'http://localhost:9090/api/v1/status/config'
```

### Configuration File (prometheus.yml)
```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    monitor: 'prometheus'

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'localhost:9093'

rule_files:
  - 'alert.rules.yml'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'kubernetes'
    kubernetes_sd_configs:
      - role: pod
```

### Alert Rules (alert.rules.yml)
```yaml
# alert.rules.yml
groups:
  - name: node_alerts
    interval: 30s
    rules:
      - alert: HighCPUUsage
        expr: node_cpu_seconds_total > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is above 80% for 5 minutes"

      - alert: DiskSpaceWarning
        expr: node_filesystem_avail_bytes / node_filesystem_size_bytes < 0.2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
```

### Common Issues & Resolution

**Issue: Scrape targets down**
```bash
# Solution: Check target endpoint
curl -v http://target:9100/metrics

# View scrape status in UI
# Go to: http://localhost:9090/targets

# Check Prometheus logs
journalctl -u prometheus -n 50

# Verify configuration
prometheus --config.file=prometheus.yml --check
```

**Issue: High memory usage**
```bash
# Solution: Reduce retention time
prometheus --storage.tsdb.retention.time=7d

# Limit sample retention
prometheus --storage.tsdb.retention.size=5GB

# Check current memory usage
ps aux | grep prometheus
```

**Issue: Slow queries**
```bash
# Solution: Check query in UI and optimize
# Dashboard -> Graph -> enter query

# Profile performance
curl 'http://localhost:9090/api/v1/query_range?query=slow_query&start=1609459200&end=1609545600'

# Look for expensive queries in logs
journalctl -u prometheus | grep "query"
```

**Issue: Alerts not firing**
```bash
# Solution: Check alert rules
curl 'http://localhost:9090/api/v1/rules'

# Test alert condition
curl 'http://localhost:9090/api/v1/query?query=YOUR_ALERT_CONDITION'

# Verify alertmanager is configured
curl 'http://localhost:9090/api/v1/status/config' | jq '.data.alerting'

# Check alert rules syntax
prometheus --config.file=prometheus.yml --check
```

### Common Queries
```promql
# CPU usage
rate(node_cpu_seconds_total{mode="user"}[5m])

# Memory usage
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes

# Disk I/O
rate(node_disk_io_time_seconds_total[5m])

# Network interface
rate(node_network_receive_bytes_total[5m])

# Process restart count
changes(process_start_time_seconds[1h])

# HTTP request rate
rate(http_requests_total[5m])

# HTTP error rate
rate(http_requests_total{status=~"5.."}[5m])

# API latency percentile
histogram_quantile(0.95, http_request_duration_seconds_bucket)
```

### Debugging
```bash
# Check Prometheus logs
journalctl -u prometheus -f

# View configuration
curl 'http://localhost:9090/api/v1/status/config'

# Check targets
curl 'http://localhost:9090/api/v1/targets' | jq

# Check alerts
curl 'http://localhost:9090/api/v1/alerts' | jq

# Query execution time
curl 'http://localhost:9090/api/v1/query?query=up' -w '\nTotal time: %{time_total}s\n'

# Test configuration
prometheus --config.file=prometheus.yml --check

# Verbose startup
prometheus --config.file=prometheus.yml --log.level=debug
```

### Node Exporter
```bash
# Install node exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.5.0/node_exporter-1.5.0.linux-amd64.tar.gz
tar -xzf node_exporter-1.5.0.linux-amd64.tar.gz
sudo mv node_exporter-1.5.0.linux-amd64/node_exporter /usr/local/bin/

# Run node exporter
node_exporter

# Check metrics
curl http://localhost:9100/metrics

# Filter specific metrics
curl http://localhost:9100/metrics | grep node_cpu
```
