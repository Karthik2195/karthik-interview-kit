# Papertrail - Cloud Log Management Service

## What is it used for?
Papertrail is used for:
- **Log management**: Cloud-based log aggregation
- **Real-time search**: Search logs instantly
- **Log retention**: Long-term log storage
- **Alerts**: Alert on log patterns
- **Integration**: Connect to popular tools
- **Log parsing**: Automatic log parsing
- **Compliance**: Meet compliance requirements
- **Team collaboration**: Share logs across team

## Installation & Setup

```bash
# Install remote syslog daemon (rsyslog)
sudo apt-get install rsyslog

# Install nxlog (Windows alternative)
# Download from https://nxlog.io/

# Configure rsyslog to forward logs
# Edit /etc/rsyslog.d/papertrail.conf
*.* @logs1.papertrailapp.com:12345

# Restart rsyslog
sudo systemctl restart rsyslog

# For Docker
docker run -d \
  -v /dev/log:/dev/log \
  remote-syslog:latest \
  -d logs1.papertrailapp.com:12345

# For Kubernetes
# Add sidecar container to forward logs
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Test syslog forwarding
logger -t test "Test message"

# View Papertrail logs
# Visit: app.papertrailapp.com

# Configure log source
# Web UI -> Settings -> Log Destinations

# Create saved search
# Web UI -> Searches -> Save Search

# Set up alert
# Web UI -> Alerts -> Create Alert

# API access
curl -H "X-Papertrail-Token: YOUR_TOKEN" \
  https://api.papertrailapp.com/api/v1/events/search.json

# Search logs via API
curl -H "X-Papertrail-Token: YOUR_TOKEN" \
  'https://api.papertrailapp.com/api/v1/events/search.json?q=ERROR'

# Export logs
curl -H "X-Papertrail-Token: YOUR_TOKEN" \
  'https://api.papertrailapp.com/api/v1/events/search.json?q=*' \
  | jq > logs.json
```

### Configuration Examples
```bash
# Rsyslog configuration for Papertrail
# /etc/rsyslog.d/papertrail.conf

$ModLoad imfile
$InputFileName /var/log/app.log
$InputFileTag app:
$InputFileStateFile stat-app
$InputFileSeverity info
$InputFileFacility local3
$InputRunFileMonitor

*.* @logs1.papertrailapp.com:12345

# Forward only errors
:msg, contains, "ERROR" @logs1.papertrailapp.com:12345

# Forward from specific source
:hostname, isequal, "myhost" @logs1.papertrailapp.com:12345
```

### Common Issues & Resolution

**Issue: Logs not appearing in Papertrail**
```bash
# Solution: Check rsyslog status
sudo systemctl status rsyslog

# Verify configuration
cat /etc/rsyslog.d/papertrail.conf

# Test connectivity
nc -zv logs1.papertrailapp.com 12345

# Send test message
logger "Test message"

# Check rsyslog logs
tail -f /var/log/syslog | grep rsyslog
```

**Issue: Connection refused**
```bash
# Solution: Verify port number
# Check Papertrail settings for correct port

# Firewall check
sudo ufw allow to logs1.papertrailapp.com

# Update configuration with correct endpoint
sudo nano /etc/rsyslog.d/papertrail.conf
# Update to correct: @logs1.papertrailapp.com:XXXXX

# Restart rsyslog
sudo systemctl restart rsyslog
```

**Issue: TLS certificate errors**
```bash
# Solution: Enable TLS
# /etc/rsyslog.d/papertrail.conf
$ActionSendStreamDriver gtls
$ActionSendStreamDriverMode 1
$ActionSendStreamDriverAuthMode x509/name
$ActionSendStreamDriverPermittedPeer *.papertrailapp.com

# Restart rsyslog
sudo systemctl restart rsyslog
```

### Search Syntax
```
# Basic search
ERROR

# Multiple keywords
ERROR AND timeout

# Exclude
ERROR NOT timeout

# Regular expression
ERROR regex="timeout|failure"

# Specific host
host:myhost ERROR

# Specific program
program:nginx ERROR

# Severity levels
severity:error
severity:warn
severity:info

# Time range
after:2024-01-01 before:2024-01-02

# Source IP
source_ip:192.168.1.1
```

### Alerts
```bash
# Create email alert
# Web UI -> Alerts -> New Alert
# Set trigger: when log matches pattern
# Set action: send email

# Create webhook alert
# Web UI -> Alerts -> New Alert
# Set action: send to webhook URL

# Create PagerDuty alert
# Web UI -> Alerts -> New Alert
# Integrate with PagerDuty
# Trigger incident on match
```

### API Usage
```bash
# Get events
curl -H "X-Papertrail-Token: TOKEN" \
  'https://api.papertrailapp.com/api/v1/events/search.json?q=ERROR&limit=100'

# Get saved searches
curl -H "X-Papertrail-Token: TOKEN" \
  'https://api.papertrailapp.com/api/v1/searches.json'

# Create saved search
curl -X POST \
  -H "X-Papertrail-Token: TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"My Search","query":"ERROR"}' \
  'https://api.papertrailapp.com/api/v1/searches.json'

# Fetch API info
curl -H "X-Papertrail-Token: TOKEN" \
  'https://api.papertrailapp.com/api/v1/systems.json'
```

### Integration Examples
```bash
# Slack integration
# Web UI -> Integrations -> Slack

# PagerDuty integration
# Web UI -> Integrations -> PagerDuty

# Custom webhook
# Web UI -> Alerts -> Custom webhook

# Email integration
# Web UI -> Email Notifications
```

### Debugging
```bash
# Test rsyslog
rsyslogd -N1 -f /etc/rsyslog.conf

# Show rsyslog version
rsyslogd -version

# Monitor rsyslog in real-time
tail -f /var/log/syslog | grep rsyslog

# Debug forwarding
logger -d -t test "Debug message"

# Check iptables/firewall
sudo iptables -L | grep 12345
```

### Log Retention
```bash
# Configure retention
# Web UI -> Settings -> Retention Policies

# Set retention for specific log sources
# Web UI -> Log Destinations -> Retention

# Export old logs before deletion
# Use API to export logs
# Then delete
```

### Advanced Features
```bash
# Log parsing
# Papertrail auto-parses common log formats

# Custom parsing
# Web UI -> Log Destinations -> Edit Source
# Set regex pattern for parsing

# Event correlation
# Saved searches can reference multiple sources

# Real-time dashboard
# Web UI -> Dashboard -> Real-time
```

### Performance Tips
```bash
# Filter logs at source
# Only send relevant logs to Papertrail

# Use severity levels
# Forward only warn+ level

# Batch logs
# Configure rsyslog to batch messages

# Monitor bandwidth
# Web UI -> Settings -> Usage
```
