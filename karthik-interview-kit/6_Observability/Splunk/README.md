# Splunk - Enterprise Security & Analytics Platform

## What is it used for?
Splunk is used for:
- **Log aggregation**: Collect logs from all sources
- **Security analytics**: Detect threats and anomalies
- **Operational intelligence**: Monitor infrastructure
- **Compliance reporting**: Generate compliance reports
- **Application performance**: Monitor app metrics
- **Business analytics**: Analyze business data
- **Incident response**: Respond to security incidents
- **Alerting**: Create sophisticated alerts

## Installation

```bash
# Download Splunk
wget -O splunk-9.0.0-linux-2.6-amd64.rpm 'https://www.splunk.com/en_US/download/splunk/enterprise'

# Install
sudo rpm -i splunk-9.0.0-linux-2.6-amd64.rpm

# Accept license and start
cd /opt/splunk/bin
sudo ./splunk start --accept-license

# Access Splunk
# http://localhost:8000
# Default: admin / changeme

# Create Splunk user
sudo useradd -r -m -d /opt/splunk splunk

# Enable boot-start
sudo /opt/splunk/bin/splunk enable boot-start -user splunk
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check Splunk status
sudo /opt/splunk/bin/splunk status

# Start Splunk
sudo /opt/splunk/bin/splunk start

# Stop Splunk
sudo /opt/splunk/bin/splunk stop

# Restart Splunk
sudo /opt/splunk/bin/splunk restart

# Add data input
# Web UI -> Settings -> Data Inputs -> New

# View indexed data
# Web UI -> Search & Reporting -> Search

# Create index
# Web UI -> Settings -> Indexes -> New Index

# Configure forwarder
# Edit /opt/splunkforwarder/etc/system/local/outputs.conf

# Restart forwarder
sudo /opt/splunkforwarder/bin/splunk restart

# Test connection
/opt/splunkforwarder/bin/splunk list forward-server -auth admin:password
```

### Search Queries
```splunk
# Search for errors
error OR failed OR exception

# Search by index and source
index=main source="/var/log/syslog"

# Search by time
earliest=-24h@h latest=now

# Count events
index=main | stats count

# Top results
index=main | stats count by host | sort - count

# Search and chart
index=main | stats count by status | chart count by status

# Timechart
index=main | timechart count

# Field extraction
index=main | fields host, status, message

# Regex matching
index=main | regex message="ERROR.*timeout"

# Case sensitivity
index=main message=Error OR message=error

# Transaction
index=main | transaction host status

# Join
index=main | join host [search index=secondary_index]
```

### Common Issues & Resolution

**Issue: Forwarder not connecting to Splunk**
```bash
# Solution: Check forwarding configuration
sudo /opt/splunkforwarder/bin/splunk list forward-server

# Verify connection
sudo /opt/splunkforwarder/bin/splunk test forward-server

# Check forwarding logs
sudo tail -f /opt/splunkforwarder/var/log/splunk/metrics.log

# Restart forwarder
sudo /opt/splunkforwarder/bin/splunk restart
```

**Issue: Data not showing in Splunk**
```bash
# Solution: Check if data is indexed
index=_internal group=queue

# Verify input configuration
# Web UI -> Settings -> Data Inputs

# Check for indexing errors
index=_internal source=*splunkd.log ERROR

# Rebuild index
sudo /opt/splunk/bin/splunk rebuild
```

**Issue: High memory usage**
```bash
# Solution: Limit index cache
# Edit /opt/splunk/etc/system/local/limits.conf
[indexer]
maxKBps = 10000

# Restart Splunk
sudo /opt/splunk/bin/splunk restart

# Monitor resource usage
# Web UI -> Monitoring Console -> Resource Usage
```

### Alerts and Reports
```bash
# Create alert
# Web UI -> Settings -> Alerts -> New Alert

# Create report
# Web UI -> Search & Reporting -> Save as

# Schedule report
# Search results -> Save as -> Schedule Report

# View alert logs
index=_internal group=schedulers log_level=WARN
```

### Data Inputs Configuration
```
# /opt/splunk/etc/system/local/inputs.conf
[syslog://9514]
connection_host = dns
source = syslog

[monitor:///var/log/myapp/error.log]
index = main
sourcetype = myapp_error

[http://5000]
auth = admin:password
index = main
sourcetype = json
```

### Debugging
```bash
# Check Splunk logs
tail -f /opt/splunk/var/log/splunk/splunkd.log

# Check metrics
/opt/splunk/bin/splunk list inputstatus -auth admin:password

# Verify index health
index=_internal source=*indexer.log status

# Monitor queue usage
index=_internal group=queue

# Check indexing performance
index=_internal group=indexer processor=http_in
```

### Advanced Features
```bash
# Use Lookup tables
# Web UI -> Settings -> Lookups -> New Lookup Table

# Create saved searches
# Web UI -> Search & Reporting -> Save Search

# Create knowledge objects
# Web UI -> Settings -> Knowledge objects

# Configure props.conf for parsing
# Set LINE_BREAKER, TRUNCATE, etc.
```

### Performance Optimization
```bash
# Limit search results
earliest=-7d

# Exclude unwanted data
NOT (source=*/access.log OR source=*/debug.log)

# Use accelerated reports
# Web UI -> Settings -> Reports -> Acceleration

# Optimize index structure
# Small indexes for fast searches
# Archive old data

# Use balanced tstats
index=main | tstats count
```

### Cluster Management
```bash
# Configure clustering
# Edit server.conf
[clustering]
mode = master
cluster_label = my_cluster
replication_factor = 3

# Add cluster nodes
# Configure peer nodes with manager_uri

# View cluster status
# Web UI -> Monitoring Console -> Clustering
```

### User and Authentication
```bash
# Create user
/opt/splunk/bin/splunk add user username \
  -password password \
  -role admin \
  -auth admin:password

# Change password
/opt/splunk/bin/splunk edit user username \
  -password newpassword

# Configure LDAP authentication
# Edit authentication.conf
[ldap]
...
```
