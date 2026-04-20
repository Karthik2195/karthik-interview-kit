# ELK Stack - Elasticsearch, Logstash, Kibana

## What is it used for?
ELK Stack is used for:
- **Log aggregation**: Collect logs from multiple sources
- **Log analysis**: Search and analyze log data
- **Visualization**: Create dashboards and visualizations
- **Alerting**: Alert on log patterns
- **Performance monitoring**: Monitor system performance
- **Security analysis**: Detect security threats
- **Business analytics**: Analyze business metrics
- **Troubleshooting**: Debug issues across systems

## Installation

```bash
# Docker Compose setup
cat > docker-compose.yml << 'EOF'
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.0.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data

  logstash:
    image: docker.elastic.co/logstash/logstash:8.0.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    ports:
      - "5000:5000"
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.0.0
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

volumes:
  elasticsearch_data:
EOF

docker-compose up -d
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check Elasticsearch status
curl http://localhost:9200

# List indices
curl http://localhost:9200/_cat/indices

# Create index
curl -X PUT http://localhost:9200/my-index

# Index a document
curl -X POST http://localhost:9200/my-index/_doc/ \
  -H 'Content-Type: application/json' \
  -d '{"message": "test log"}'

# Search documents
curl -X GET http://localhost:9200/my-index/_search

# Delete index
curl -X DELETE http://localhost:9200/my-index

# View cluster health
curl http://localhost:9200/_cluster/health

# Access Kibana
# http://localhost:5601
```

### Logstash Configuration
```ruby
# logstash.conf
input {
  syslog {
    port => 514
    type => syslog
  }
  file {
    path => "/var/log/application/*.log"
    start_position => "beginning"
  }
}

filter {
  if [type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGLINE}" }
    }
  }
  
  mutate {
    add_field => { "[@metadata][index_name]" => "logs-%{+YYYY.MM.dd}" }
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "%{[@metadata][index_name]}"
  }
  stdout { codec => rubydebug }
}
```

### Common Issues & Resolution

**Issue: Elasticsearch not responding**
```bash
# Solution: Check service status
sudo systemctl status elasticsearch

# Restart service
sudo systemctl restart elasticsearch

# Check logs
tail -f /var/log/elasticsearch/elasticsearch.log

# Verify connectivity
curl -v http://localhost:9200
```

**Issue: Logstash not processing logs**
```bash
# Solution: Test Logstash configuration
logstash -f logstash.conf -t

# Enable debug
logstash -f logstash.conf --log.level=debug

# Check Logstash logs
tail -f /var/log/logstash/logstash-plain.log

# Verify input source
tail -f /var/log/syslog
```

**Issue: Kibana cannot connect to Elasticsearch**
```bash
# Solution: Check Elasticsearch status
curl http://localhost:9200

# Update Kibana config
# Edit config/kibana.yml
# elasticsearch.hosts: ["http://elasticsearch:9200"]

# Restart Kibana
sudo systemctl restart kibana
```

### Kibana Usage
```bash
# Create index pattern
# Kibana UI -> Management -> Index Patterns -> Create

# Create visualization
# Kibana UI -> Visualize -> Create

# Create dashboard
# Kibana UI -> Dashboard -> Create

# Create alert
# Kibana UI -> Management -> Alerts -> Create alert
```

### Advanced Configurations
```ruby
# Add multiple outputs
output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
  s3 {
    bucket => "my-bucket"
    region => "us-east-1"
  }
}

# Parse JSON logs
filter {
  json {
    source => "message"
  }
}

# Add GeoIP data
filter {
  geoip {
    source => "clientip"
  }
}
```

### Debugging
```bash
# Check index mapping
curl http://localhost:9200/my-index/_mapping

# View shard information
curl http://localhost:9200/_cat/shards

# Check node status
curl http://localhost:9200/_nodes

# View pending tasks
curl http://localhost:9200/_cluster/pending_tasks

# Monitor performance
curl http://localhost:9200/_stats
```

### Backup and Restore
```bash
# Create snapshot repository
curl -X PUT http://localhost:9200/_snapshot/my-repo \
  -H 'Content-Type: application/json' \
  -d '{
    "type": "fs",
    "settings": {
      "location": "/mnt/snapshots"
    }
  }'

# Create snapshot
curl -X PUT http://localhost:9200/_snapshot/my-repo/snapshot-1

# List snapshots
curl http://localhost:9200/_snapshot/my-repo/_all

# Restore snapshot
curl -X POST http://localhost:9200/_snapshot/my-repo/snapshot-1/_restore
```

### Security
```bash
# Enable X-Pack security
# Edit elasticsearch.yml
xpack.security.enabled: true

# Create users
POST /_security/user/myuser
{
  "password": "password123",
  "roles": ["superuser"]
}

# Configure HTTPS
# Generate certificates
bin/elasticsearch-certutil cert -out config/certs/node1.p12

# Add authentication to Kibana
# kibana.yml
elasticsearch.username: "kibana_system"
elasticsearch.password: "password"
```

### Performance Tuning
```bash
# Adjust number of shards
PUT /my-index
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1
  }
}

# Monitor heap usage
curl http://localhost:9200/_nodes/stats | jq '.nodes[].jvm.mem'

# Optimize index
POST /my-index/_forcemerge

# Delete old indices
curl -X DELETE http://localhost:9200/logs-2023*
```
