# Prometheus - Senior Level Interview Questions & Answers

## Table of Contents
1. Architecture & Design
2. Advanced Queries & Analysis
3. Scalability & Performance
4. Alerting Strategies
5. Multi-cluster Monitoring

---

## 1. Architecture & Design

### High-Level: What is Prometheus and how does it help you manage infrastructure?

**Interviewer:** "Karthik, tell me about a time when your service went down and you had no idea why. What would Prometheus have done differently?"

**Karthik:** "I appreciate the question. Early in my career, I experienced exactly that scenario. Service crashed at 2 AM, stakeholders were inquiring, and I was reviewing logs without clear visibility into root cause. Prometheus would have fundamentally transformed that situation. The mechanism is straightforward: your application exposes metrics at a designated endpoint, and Prometheus performs periodic scraping at 15-second intervals. This establishes continuous collection of infrastructure telemetry - CPU utilization, memory consumption, request metrics, error rates, and more. When incidents occur, rather than operating without visibility, you possess a comprehensive time-series database with historical context. You can retrospectively analyze when the problem initiated, CPU behavior patterns, potential memory leaks, or whether deployment preceded the failure. It provides a complete forensic analysis of your infrastructure. The architectural advantage of Prometheus is its pull-based model rather than push-based architecture, meaning your services need not actively transmit data - they simply expose it and Prometheus ingests it. This approach proves more efficient and reduces application-level resource consumption."

---

### High-Level: Walk me through how you'd diagnose a production incident at 3 AM using Prometheus.

**Interviewer:** "Alright, it's 3 AM, an alert fires saying your payment service is down. You have maybe 5 minutes before stakeholders are inquiring. Walk me through exactly what you'd do with Prometheus to determine the root cause."

**Karthik:** "My initial approach would involve examining the `up` metric for the payment service. This metric provides a definitive state - a value of 1 indicates the service is operational, while 0 indicates a failure state. Once confirmed, I'd examine CPU and memory utilization metrics on the infrastructure nodes hosting the service. Service failures frequently result from memory exhaustion leading to OOMKilled events or CPU saturation causing request timeouts. I'd simultaneously assess request volume patterns to determine whether we're experiencing anomalous traffic or typical load patterns. The subsequent step involves correlating the service failure timestamp with deployment history. By examining Prometheus data from the incident timeframe and cross-referencing deployment logs, if a deployment occurred 2-3 minutes preceding the failure, that represents a strong causal indicator. I would then either initiate a rollback procedure or analyze the specific code changes. The significant advantage is that Prometheus consolidates all this telemetry in a unified interface - I avoid the need for manual server access or extensive log file analysis. Several targeted queries provide comprehensive insight into the incident."

---

### Q: Design a global Prometheus monitoring architecture for multi-region deployment

**A:**
```yaml
# Global Monitoring Architecture

# Primary: Prometheus in each region (HA)
# Secondary: Central aggregator
# Tertiary: Long-term storage (Cortex/Thanos)

---
# Regional Prometheus (HA setup)
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config-us-east
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
      external_labels:
        cluster: us-east-1
        region: us-east
        environment: production
    
    remote_write:
      - url: http://thanos-receive:10901/api/v1/receive
        queue_config:
          capacity: 100000
          max_shards: 50
    
    scrape_configs:
      - job_name: 'kubernetes-cluster'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace]
            action: keep
            regex: 'default|monitoring'
      
      - job_name: 'nodes'
        static_configs:
          - targets: ['localhost:9100']

---
# Central aggregator (Thanos Query)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thanos-query
spec:
  replicas: 3
  selector:
    matchLabels:
      app: thanos-query
  template:
    metadata:
      labels:
        app: thanos-query
    spec:
      containers:
      - name: thanos-query
        image: quay.io/thanos/thanos:latest
        args:
          - query
          - --grpc-address=0.0.0.0:10901
          - --http-address=0.0.0.0:9090
          - --query.replica-label=replica
          - --store=dnssrv+_grpc._tcp.prometheus-us-east.monitoring.svc.cluster.local
          - --store=dnssrv+_grpc._tcp.prometheus-us-west.monitoring.svc.cluster.local
          - --store=dnssrv+_grpc._tcp.prometheus-eu-west.monitoring.svc.cluster.local
        ports:
        - name: grpc
          containerPort: 10901
        - name: http
          containerPort: 9090

---
# Long-term storage (S3)
apiVersion: v1
kind: ConfigMap
metadata:
  name: thanos-storage-config
data:
  objstore.yml: |
    type: S3
    config:
      bucket: thanos-metrics
      endpoint: s3.amazonaws.com
      region: us-east-1
      access_key: ${AWS_ACCESS_KEY_ID}
      secret_key: ${AWS_SECRET_ACCESS_KEY}
```

---

### Q: Design an advanced alerting strategy with intelligent thresholding

**A:**
```yaml
# Advanced Alerting Configuration

groups:
  - name: application
    interval: 30s
    rules:
      # Threshold-based alerts
      - alert: HighErrorRate
        expr: |
          rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        annotations:
          summary: "High error rate detected"
          dashboard: "http://grafana/d/app-dashboard"

      # Anomaly detection (uses recording rules)
      - alert: AnomalousHighMemory
        expr: |
          (container_memory_usage_bytes - container_memory_usage_bytes offset 1h) 
          / container_memory_usage_bytes > 0.5
        for: 10m
        annotations:
          summary: "Memory usage increased by 50%"

      # Forecast-based alerts
      - alert: DiskSpaceFullPrediction
        expr: |
          predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 24*3600) < 0
        for: 30m
        annotations:
          summary: "Disk will be full in 24 hours"

      # Multi-metric composite alerts
      - alert: ServiceDegraded
        expr: |
          (rate(http_requests_total[5m]) < 10)
          and (rate(http_requests_total{status="500"}[5m]) > 0.1)
        for: 10m
        annotations:
          summary: "Service is degraded"

  # Recording rules for performance
  - name: recording_rules
    interval: 1m
    rules:
      - record: instance:node_cpu:rate5m
        expr: rate(node_cpu_seconds_total[5m])

      - record: instance:node_memory:ratio
        expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes

      - record: instance:requests:rate5m
        expr: rate(http_requests_total[5m])
```

---

## 2. Advanced Queries & Analysis

### High-Level: How would you write a PromQL query to assess service health status?

**Interviewer:** "You're reviewing service status before a standup meeting. You want a rapid assessment of operational state across your infrastructure. What queries would you construct?"

**Karthik:** "I would implement a tiered query approach. The foundational query examines the `up` metric for specific services, providing immediate operational state visibility. Beyond basic availability, I construct queries measuring error rates using `rate(http_requests_total{status=~\"5..\"}[1m])` which quantifies 5xx errors per second over a one-minute window. This reveals whether the service is degraded or completely unavailable. Subsequently, I analyze latency characteristics using `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[1m]))` which provides the 95th percentile response latency. This metric identifies performance degradation that doesn't manifest as errors but impacts user experience. By executing these three queries sequentially, I obtain comprehensive health assessment: operational status, error frequency, and performance characteristics. When all three indicate normal parameters, stakeholders have confidence in system health."

---

### High-Level: An incident is occurring with elevated error rates. How would you identify the affected component using Prometheus?

**Interviewer:** "You receive an alert indicating error rates are critically elevated. Multiple services appear involved. Your objective is to isolate which specific components are experiencing degradation. How would you approach this investigation?"

**Karthik:** "My methodology involves progressively narrowing scope through dimensional analysis. The initial query, `topk(5, rate(http_requests_total{status=~\"5..\"}[1m]) by (service))`, identifies which services are generating the highest error volumes. This dimensional breakdown reveals whether the incident is systemic or localized to specific services. Once I've identified the affected service, I further segment by endpoint using `rate(http_requests_total{status=~\"5..\",service=\"payment\"}[1m]) by (endpoint)`. This reveals whether the entire service is degraded or if specific endpoints are experiencing issues. For example, the payment checkout endpoint might be failing while inventory endpoints function normally. This specificity allows rapid remediation focusing on the precise failure point. Subsequently, I examine infrastructure metrics for the affected service - CPU utilization, memory consumption, database connection pool status, and any external dependency health. By correlating temporal aspects - when the errors commenced versus when resource constraints appeared - I can identify whether the failure stems from code defects, infrastructure limitations, or external dependency failures. This systematic dimensional analysis transforms a broad problem statement into a specific, addressable issue."

---

### Q: Write complex PromQL queries for real-world scenarios

**A:**
```promql
# 1. Request latency percentiles with labels
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service, method)
)

# 2. Top N services by error rate
topk(5,
  rate(http_requests_total{status=~"5.."}[5m])
  / on(service) group_left sum(rate(http_requests_total[5m])) by (service)
)

# 3. Memory usage with trend
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024
unless on(instance) topk by (instance) (1, increase(node_memory_MemAvailable_bytes[1h])) < 0

# 4. Request rate normalized by replicas
rate(http_requests_total[5m]) / count(up{job="api"} == 1)

# 5. CPU saturation (useful vs actual)
100 * (1 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))))

# 6. Time series growth
count(ALERTS) - count(ALERTS offset 24h)

# 7. Container resource efficiency
container_memory_usage_bytes / container_spec_memory_limit_bytes

# 8. API endpoint SLO compliance
sum(rate(http_requests_total{status=~"2.."}[5m])) 
/ on(endpoint) sum(rate(http_requests_total[5m])) by (endpoint) >= 0.999

# 9. Database connection pool exhaustion rate
rate(mysql_global_variables_max_connections[5m]) * 0.8 - 
avg(mysql_global_status_threads_connected)

# 10. Predict when metric reaches threshold
predict_linear(disk_free_bytes[1h], 7*24*3600) < 1e9
```

---

## 3. Scalability & Performance

### High-Level: When planning infrastructure monitoring, how do you determine appropriate storage allocation for Prometheus?

**Interviewer:** "You're designing monitoring infrastructure for a microservices architecture containing 100 Kubernetes pods, each exposing 50 metrics. Your retention policy requires 30 days of historical data. What storage capacity would you provision?"

**Karthik:** "This calculation requires understanding compression ratios and temporal aspects of time-series data. With 100 pods and 50 metrics per pod, we're maintaining 5,000 distinct time series. Prometheus performs scrapes at 15-second intervals, yielding approximately 5,760 scrapes daily. Each compressed data point consumes roughly 1-2 bytes. The mathematical projection: 5,000 time series × 2 bytes × 5,760 daily scrapes × 30 days yields approximately 1.7 gigabytes. However, practical considerations suggest higher allocation. As your infrastructure scales and service count increases, you'll collect additional metrics. Furthermore, operational practices often require retention periods exceeding initial estimates. I would recommend provisioning 10-15 gigabytes for this scenario. An equally important consideration involves storage characteristics - Prometheus demands high IOPS throughput due to continuous time-series writes. I would prioritize solid-state storage over mechanical drives. In Kubernetes environments, I ensure persistent volumes are backed by performant storage classes. Monitoring storage utilization metrics themselves - via `prometheus_tsdb_data_directory_size_bytes` - enables proactive capacity management before exhaustion occurs."

---

### High-Level: Your Prometheus dashboard queries are executing slowly. Team members are frustrated waiting for results during incident investigation. What is your diagnostic approach?

**Interviewer:** "It's Monday morning. Your team is investigating a production incident. They access the Prometheus dashboard and every query requires 30 seconds to complete. The incident timeline is slipping. What is your immediate assessment and remediation strategy?"

**Karthik:** "This situation typically indicates one of two fundamental issues: either excessive cardinality in your stored metrics, or inefficient query construction. My initial diagnostic involves executing `count(up)` to determine total time series volume. If that count exceeds 1 million, cardinality explosion is probable. This frequently results from uncontrolled label dimensionality - for example, including user IDs, request IDs, or other high-cardinality attributes as metric labels. The second issue involves query inefficiency. Overly broad queries like `rate(http_requests_total[5m])` without label filtering force Prometheus to evaluate every single request metric across all services, endpoints, and instances. The immediate remediation involves implementing recording rules - pre-computed queries that Prometheus calculates on a regular schedule, typically every minute. Rather than dashboards querying raw metrics which requires real-time aggregation, recording rules pre-aggregate results so dashboard queries return instantly. For example, a recording rule `instance:requests:rate5m = rate(http_requests_total[5m])` computed once per minute means dashboard queries simply reference the pre-computed result. Long-term solutions involve architectural segmentation - deploying Prometheus instances per region, team, or workload type, or implementing distributed query layers like Thanos which enable federated querying across multiple Prometheus instances."

---

## 3. Scalability & Performance

### Q: Design Prometheus for handling billions of metrics

**A:**
```yaml
# Scalable Prometheus Architecture

---
# Sidecar to push metrics to remote storage
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-sidecar
data:
  config.yml: |
    storage_clients:
      - type: S3
        config:
          bucket: prometheus-remote
          region: us-east-1
    
    grpc_listen_addr: "0.0.0.0:10901"
    http_listen_addr: "0.0.0.0:9090"
    
    # Sharding configuration
    sharding:
      enabled: true
      replicas: 3

---
# Prometheus with sidecar
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-with-sidecar
spec:
  replicas: 3
  template:
    spec:
      containers:
      # Main Prometheus
      - name: prometheus
        image: prom/prometheus:latest
        resources:
          requests:
            memory: "2Gi"
            cpu: "1"
          limits:
            memory: "4Gi"
            cpu: "2"
        args:
          - --storage.tsdb.retention.time=24h
          - --storage.tsdb.max-block-duration=2h
          - --storage.tsdb.min-block-duration=2h
          - --query.max-samples=1000000
          - --query.timeout=2m

      # Cortex/Loki for long-term storage
      - name: cortex-sidecar
        image: cortexproject/cortex:latest
        args:
          - -config.file=/etc/cortex/config.yml
```

**Optimization Techniques:**
1. Remote storage (Cortex, Thanos)
2. Metric relabeling to reduce cardinality
3. Recording rules for expensive queries
4. Sampling for high-cardinality metrics
5. Retention policies by metric type
6. Sharding across instances

---

## 4. Alerting Strategies

### High-Level: How do you design an alert strategy that balances responsiveness with operational sanity during on-call rotations?

**Interviewer:** "Tell me about your alert strategy. You're designing monitoring for a production system where you'll be on-call overnight. I want to understand how you prevent alert fatigue while ensuring critical issues receive immediate attention."

**Karthik:** "The foundational principle I apply is restricting alerts to conditions requiring immediate human intervention. I've observed teams creating excessive alerts that generate alert fatigue, diminishing team effectiveness during actual incidents. The mechanism involves two key decisions: first, defining which conditions genuinely necessitate wake-up calls versus those that can queue in communication channels, and second, establishing appropriate severity stratification. A critical alert threshold might be: service completely unavailable for more than 60 seconds. This avoids false positives from transient pod restarts which typically resolve within 10 seconds. The timing parameter - `for: 1m` - proves essential. It prevents trivial fluctuations from triggering notifications, while ensuring genuine problems receive attention. Beyond binary firing, I implement severity levels. Critical alerts trigger immediate escalation via on-call platforms like PagerDuty. Warning alerts accumulate in team communication channels where they receive business-hours attention. This graduated approach ensures operational team sustainability while preserving incident response capability. Ongoing refinement involves analyzing alert history. Alerts that consistently fire without corresponding action items or real customer impact represent noise rather than signals and should be disabled. Teams should actively tune their alert configuration based on incident postmortem feedback rather than accepting default configurations."

---

### High-Level: You receive multiple simultaneous alerts during a 3 AM incident. How do you implement rapid triage to identify and remediate the root cause?

**Interviewer:** "Your phone receives notifications for multiple alerts in rapid succession at 3 AM. CPU elevated, memory elevated, disk utilization rising, request latency increasing, and error rates climbing. You have approximately five minutes before customer impact escalates. What is your triage and remediation methodology?"

**Karthik:** "My approach prioritizes understanding direct customer impact versus leading indicators of future impact. Alerts that directly affect service availability or functionality represent P1 - immediate priority. If the service is completely unavailable or the majority of requests are failing, I dedicate resources to resolution within minutes. Alerts representing resource constraints that will become problems within an hour constitute P2. For example, CPU at 85% with continued upward trajectory warrants investigation and potential resource scaling, but currently maintains customer availability. P3 alerts involve future concerns that don't require immediate action - disk space adequate for several hours before exhaustion, for example. Beyond prioritization, I examine alert correlation. When CPU, memory, and latency spike simultaneously, this typically indicates a single root cause rather than three independent problems. I investigate the foundational cause - perhaps a code deployment introduced a memory leak - rather than addressing symptoms individually. The methodology involves systematic examination: reviewing deployment history to correlate timing, analyzing application logs for error patterns, examining resource utilization trends on affected infrastructure, and checking external dependency status. By isolating the root cause, remediation becomes targeted and effective. Treating symptoms individually often proves unsuccessful and delays true resolution."

---

### Q: Design sophisticated alert routing with escalation

**A:**
```yaml
# Alert Manager Configuration with Routing Tree

global:
  resolve_timeout: 5m
  slack_api_url: ${SLACK_WEBHOOK_URL}

route:
  receiver: 'team-default'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  
  # Route by severity
  routes:
    - match:
        severity: critical
      receiver: 'ops-critical'
      group_wait: 10s
      repeat_interval: 1h
      routes:
        - match:
            service: database
          receiver: 'dba-oncall'
        - match:
            service: api
          receiver: 'backend-team'
    
    - match:
        severity: warning
      receiver: 'ops-warning'
      group_wait: 5m
      repeat_interval: 1h
    
    - match:
        severity: info
      receiver: 'logging'
      group_wait: 10m

receivers:
  - name: 'ops-critical'
    slack_configs:
      - channel: '#alerts-critical'
        title: 'Critical Alert'
        send_resolved: true
    pagerduty_configs:
      - routing_key: ${PAGERDUTY_ROUTING_KEY}
        description: '{{ .GroupLabels.alertname }}'

  - name: 'dba-oncall'
    slack_configs:
      - channel: '#database-alerts'
    opsgenie_configs:
      - api_key: ${OPSGENIE_API_KEY}
        priority: 'P1'

  - name: 'logging'
    webhook_configs:
      - url: 'http://logging-system:8080/alerts'

inhibit_rules:
  # Silence warnings when critical is firing
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster']
```

---

## 5. Multi-cluster Monitoring

### High-Level: You have Kubernetes clusters in three regions. How do you monitor them all?

**Interviewer:** "Your company has data centers in US, Europe, and Asia. Each has its own Kubernetes cluster with its own Prometheus. But your boss wants one dashboard that shows the global picture. How do you set that up?"

**Karthik:** "This is where Prometheus gets interesting. The naive approach is to have a central Prometheus that scrapes all three regional Prometheus instances. This works and is simple to set up, but it's not ideal because if your central Prometheus goes down, you lose all visibility. A better approach is to use Thanos, which is basically a distributed query layer on top of Prometheus. Each region has its own Prometheus with a Thanos sidecar. The sidecar uploads historical data to S3 or GCS. Then you have a central Thanos Query component that can query across all three regions. When you run a query, Thanos fan-outs to all three regional Prometheus instances in parallel and aggregates the results. The cool part is if one regional Prometheus is down, the other two still work. And Thanos handles data deduplication automatically. So if you have metrics from multiple replicas, Thanos knows they're the same metric and deduplicates. It's pretty elegant actually. The trade-off is there's more moving parts to manage, but at scale it's worth it."

---

### High-Level: Your Prometheus crashes for 2 hours. Is all your historical data lost?

### High-Level: If your Prometheus server experiences catastrophic failure and requires rebuild, is historical metric data permanently lost?

**Interviewer:** "Worst-case scenario: your Prometheus server experiences critical hardware failure requiring complete rebuild. The outage persists for two hours. After recovery and restart, do you retain the month's prior historical data, or is it irretrievably lost?"

**Karthik:** "The architecture provides data protection through local persistent storage. Prometheus maintains its database on disk in its proprietary time-series format. As long as storage infrastructure remains intact, data persists across service restarts. When Prometheus restarts, it reconstructs its database from disk storage and resumes normal operation. There will be a temporal gap during the outage period when no metrics were collected, but historical data preceding and following the incident remains accessible. The critical implementation detail involves persistent volume configuration in Kubernetes environments. Ephemeral storage attached to pods results in complete data loss upon pod termination or restart. Proper architecture requires persistent volume claims backed by reliable storage classes. This ensures data survival across node failures or pod rescheduling. For truly comprehensive data retention, I implement a tiered storage strategy: Prometheus handles short-term storage - perhaps 30 days - on local high-performance storage. For long-term retention and disaster recovery, I employ external storage solutions like Thanos or Cortex which continuously push historical data to object storage such as AWS S3 or Google Cloud Storage. This architecture provides redundancy at multiple levels. Even catastrophic local disk failure doesn't result in complete data loss because archive copies exist in object storage. Prometheus transitions from a single point of failure to a resilient multi-tier system."

---

### Q: Design monitoring for Kubernetes multi-cluster setup

**A:**
```yaml
# Multi-cluster Prometheus setup

---
# Cluster 1: Prometheus with federation
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-fed-us-east
data:
  prometheus.yml: |
    global:
      external_labels:
        cluster: us-east-1
        environment: production
    
    scrape_configs:
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod

---
# Central Prometheus that scrapes from all clusters
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-central
data:
  prometheus.yml: |
    global:
      external_labels:
        cluster: global
    
    scrape_configs:
      # Federate metrics from each cluster
      - job_name: 'federate-us-east'
        honor_labels: true
        metrics_path: '/federate'
        scrape_interval: 15s
        static_configs:
          - targets: ['prometheus-us-east:9090']
        params:
          'match[]':
            - '{job="kubernetes-pods"}'
            - '{job="node"}'
      
      - job_name: 'federate-eu-west'
        honor_labels: true
        metrics_path: '/federate'
        static_configs:
          - targets: ['prometheus-eu-west:9090']
      
      - job_name: 'federate-ap-south'
        honor_labels: true
        metrics_path: '/federate'
        static_configs:
          - targets: ['prometheus-ap-south:9090']
```

---

## Summary

**Critical Senior Prometheus Skills:**
1. ✅ Multi-region architecture
2. ✅ Advanced PromQL queries
3. ✅ Scalability patterns
4. ✅ Alert routing & escalation
5. ✅ Performance optimization
6. ✅ Multi-cluster monitoring
7. ✅ Recording rules
8. ✅ Storage solutions

*Created by Karthik Reddy*
