# Google Cloud Platform (GCP) - Senior Level Interview Questions & Answers

## Table of Contents
1. Architecture & GCP Services
2. Data & Analytics
3. Kubernetes & Container Strategy
4. Cost Optimization
5. AI/ML Integration

---

## 1. Architecture & GCP Services

### Topic: GCP Architecture & Differentiation

You've worked on AWS and Azure. Our company is considering GCP. What's different? When should we use GCP over the others?

GCP is actually the most advanced from a machine learning and data analytics perspective because Google uses the same infrastructure internally for their own AI. If your company is AI-first, data-heavy, or analytics-focused, GCP is the obvious choice. Technically, they're all similar - compute, networking, databases, but GCP excels in specific areas. First, data processing: BigQuery is genuinely best-in-class for analytics. Faster queries than Redshift or Snowflake, cheaper too. If you're doing data warehousing at scale, GCP wins. Second, Kubernetes: Google invented Kubernetes, so GKE (Google Kubernetes Engine) is the tightest integration. If your team is Kubernetes-heavy, GKE is more native than EKS. Third, machine learning: Vertex AI is better than SageMaker. AutoML, training pipelines, deployment - all seamless. Fourth, streaming: Dataflow (Apache Beam managed) is excellent for real-time data processing. Fifth, infrastructure: Google's network is arguably better than AWS or Azure - lower latency, more direct peering. The downside: smaller marketplace of third-party tools, fewer enterprise relationships than AWS, and Azure's advantage with Microsoft integration doesn't apply. GCP is still playing catch-up in compliance certifications in some regulated industries. Pricing is often cheaper than AWS, but cost management is critical. My rule: If you're doing traditional enterprise IT, use AWS or Azure. If you're AI/data/analytics company, GCP is the best. If you're Microsoft shop, Azure. GCP is 15-20% of enterprises, AWS is 60%, Azure is 20% (rough split)."

---

### Topic: Real-Time Latency Diagnosis

You deployed an application on GCP and it's suddenly slow. Users in Europe are experiencing 500ms latency. What do you check first?

First, I'd check if this is a global problem or regional. GCP's Cloud Monitoring dashboard would show latency by region instantly. If it's Europe-specific while US is fine, it's a geographic issue. Second, I'd check if the app is actually in the right region. Maybe someone deployed to us-central1 but users are in europe-west1. Third, I'd check compute resources - are instances under high CPU or memory pressure? Cloud Monitoring would show CPU at 95%+. If so, we're capacity-constrained. Fourth, I'd check network - Cloud Trace shows request latency breakdown. Is it network latency or application latency? If network, it might be Cloud NAT bottleneck or poor routing. Fifth, check if we're hitting a database bottleneck. Cloud SQL might be struggling. Sixth, check if caching is working - if Cache invalidated, requests go to origin instead of cache, that's slow. In practice, 80% of the time it's one of these: 1) Capacity issue - need to scale up, 2) Wrong region - redeploy, 3) Database bottleneck - optimize queries or scale database, 4) Cache failure - restart cache. My systematic approach: Check monitoring dashboard, look for anomalies, drill into region with highest latency, check compute/database/network metrics, scale or fix bottleneck."

---

### Q: Design a global GCP infrastructure with lowest latency

**A:**
```yaml
# GCP Multi-Region Low-Latency Architecture

# Global Load Balancing
resource "google_compute_global_forwarding_rule" "default" {
  name       = "global-lb"
  target     = google_compute_target_https_proxy.default.id
  port_range = "443"
  ip_protocol = "TCP"
  
  load_balancing_scheme = "EXTERNAL"
  ip_address           = google_compute_global_address.default.id
}

# Backend Service with CloudCDN
resource "google_compute_backend_service" "default" {
  name            = "backend-service"
  protocol        = "HTTPS"
  timeout_sec     = 30
  enable_cdn      = true
  
  cdn_policy {
    cache_mode        = "CACHE_ALL_STATIC"
    default_ttl       = 3600
    max_ttl           = 86400
    negative_caching  = true
    negative_caching_ttl = 120
    
    cache_key_policy {
      include_host           = true
      include_protocol       = true
      include_query_string   = false
    }
  }
  
  # Backends in multiple regions
  backend {
    group          = google_compute_instance_group_manager.us.instance_group
    balancing_mode = "RATE"
    max_rate_per_endpoint = 1000
  }
  
  backend {
    group          = google_compute_instance_group_manager.eu.instance_group
    balancing_mode = "RATE"
    max_rate_per_endpoint = 1000
  }
  
  backend {
    group          = google_compute_instance_group_manager.asia.instance_group
    balancing_mode = "RATE"
    max_rate_per_endpoint = 1000
  }
}

# Health Checks
resource "google_compute_health_check" "default" {
  name = "health-check"
  
  https_health_check {
    port         = "443"
    request_path = "/health"
  }
  
  check_interval_sec = 10
  timeout_sec        = 5
}

# Regional Instance Groups (US)
resource "google_compute_instance_group_manager" "us" {
  name       = "igm-us"
  base_instance_name = "app-us"
  zone       = "us-central1-a"
  target_size = 5
  
  version {
    instance_template = google_compute_instance_template.default.id
  }
  
  auto_healing_policies {
    health_check      = google_compute_health_check.default.id
    initial_delay_sec = 300
  }
}

# Regional Instance Groups (Europe)
resource "google_compute_instance_group_manager" "eu" {
  name       = "igm-eu"
  base_instance_name = "app-eu"
  zone       = "europe-west1-b"
  target_size = 5
  
  version {
    instance_template = google_compute_instance_template.default.id
  }
}

# Regional Instance Groups (Asia)
resource "google_compute_instance_group_manager" "asia" {
  name       = "igm-asia"
  base_instance_name = "app-asia"
  zone       = "asia-southeast1-a"
  target_size = 5
  
  version {
    instance_template = google_compute_instance_template.default.id
  }
}

# Cloud CDN with Cloud Armor
resource "google_compute_security_policy" "default" {
  name = "policy-ddos"
  
  rules {
    action   = "deny(403)"
    priority = "1000"
    match {
      version_specified_rules {
        action = "deny(403)"
        match {
          src_ip_ranges = ["192.0.2.0/24"]
        }
      }
    }
    description = "Block DDoS attacks"
  }
}
```

---

## 2. Data & Analytics

### Topic: Data Processing at Scale

Your company generates 100TB of data daily. How do you process and analyze it on GCP?

100TB daily is significant. AWS would use Redshift, but GCP is actually better here with BigQuery. Here's my pipeline: Ingestion layer - use Cloud Pub/Sub to ingest streaming data in real-time, or Cloud Storage for batch data. Cloud Pub/Sub can handle millions of messages/second. Processing layer - use Dataflow (Apache Beam) for transformation. Dataflow auto-scales, so it'll handle spikes without manual intervention. Alternatively, BigQuery has powerful SQL so you might transform directly in SQL rather than use Dataflow. Storage layer - everything lands in BigQuery. BigQuery stores petabytes if needed, costs are per query not per storage, and it's FAST. A query that would take hours in Redshift takes seconds in BigQuery. Analytics layer - dashboards in Looker (Google owns Looker now). Real-time dashboards can hit BigQuery. ML layer - if you want to do predictive analytics, Vertex AI integrates seamlessly with BigQuery. You literally just do 'CREATE MODEL' in SQL and it trains automatically. The beauty of this architecture is simplicity - no managing clusters, no Spark tuning, no Hadoop operations. Just write SQL. Cost: 100TB daily = 3PB monthly. BigQuery charges $6.25 per TB analyzed, so worst case $18k/month if you analyze everything. But with partitioning and clustering, you'll analyze maybe 10% of data, so $1800/month. That's cheap compared to alternatives. My architecture: Cloud Storage (ingestion) → Dataflow (transformation) → BigQuery (analytics) → Looker (BI)."

---

### Q: Design a real-time analytics pipeline for streaming events

**A:**
```python
# GCP Real-Time Analytics Pipeline

from google.cloud import pubsub_v1
from google.cloud import bigquery
import json
from datetime import datetime

class RealtimeAnalyticsPipeline:
    def __init__(self, project_id):
        self.project_id = project_id
        self.publisher = pubsub_v1.PublisherClient()
        self.bq_client = bigquery.Client(project=project_id)
        self.topic_path = self.publisher.topic_path(
            project_id, 
            'events-topic'
        )
    
    def publish_event(self, event_data):
        """Publish event to Pub/Sub"""
        data = json.dumps(event_data).encode('utf-8')
        
        future = self.publisher.publish(
            self.topic_path,
            data,
            origin='web',
            timestamp=datetime.now().isoformat()
        )
        
        message_id = future.result()
        return message_id
    
    def stream_events_to_bigquery(self):
        """Stream events from Pub/Sub to BigQuery"""
        subscriber = pubsub_v1.SubscriberClient()
        subscription_path = subscriber.subscription_path(
            self.project_id,
            'events-subscription'
        )
        
        def callback(message):
            # Process message
            data = json.loads(message.data.decode('utf-8'))
            
            # Insert into BigQuery
            table_id = f"{self.project_id}.analytics.events"
            errors = self.bq_client.insert_rows_json(
                table_id,
                [data],
                row_ids=[message.message_id]
            )
            
            if not errors:
                print(f"Inserted 1 event")
                message.ack()
            else:
                print(f"Error: {errors}")
                message.nack()
        
        streaming_pull_future = subscriber.subscribe(
            subscription_path,
            callback=callback
        )
        
        return streaming_pull_future
    
    def run_analytics_query(self):
        """Run analytics on streaming data"""
        query = """
            SELECT
                DATE_TRUNC(timestamp, HOUR) as hour,
                event_type,
                COUNT(*) as count,
                AVG(CAST(duration AS FLOAT64)) as avg_duration,
                PERCENTILE_CONT(CAST(duration AS FLOAT64), 0.95) 
                    OVER (PARTITION BY event_type) as p95_duration
            FROM `{}.analytics.events`
            WHERE timestamp > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
            GROUP BY hour, event_type
            ORDER BY hour DESC, count DESC
        """.format(self.project_id)
        
        job = self.bq_client.query(query)
        results = job.result()
        
        return list(results)
    
    def export_to_gcs(self, table_id):
        """Export BigQuery results to Cloud Storage"""
        job_config = bigquery.ExtractJobConfig()
        job_config.compression = bigquery.Compression.SNAPPY
        job_config.destination_format = (
            bigquery.DestinationFormat.NEWLINE_DELIMITED_JSON
        )
        
        destination_uri = f"gs://export-bucket/{table_id}/data-*.json"
        
        extract_job = self.bq_client.extract_table(
            table_id,
            destination_uri,
            job_config=job_config
        )
        
        extract_job.result()
        
        return destination_uri

# Usage
pipeline = RealtimeAnalyticsPipeline('my-project')

# Publish events
pipeline.publish_event({
    'event_type': 'page_view',
    'user_id': 'user123',
    'duration': 2500,
    'timestamp': datetime.now().isoformat()
})

# Stream to BigQuery
future = pipeline.stream_events_to_bigquery()

# Analytics queries
results = pipeline.run_analytics_query()
for row in results:
    print(f"{row.hour}: {row.event_type} - {row.count} events")
```

---

## 3. Kubernetes & Container Strategy

### Topic: Kubernetes Multi-Cluster Management

You're moving 200 microservices to Kubernetes on GCP. How do you manage this scale?

200 microservices is significant. Here's how I'd organize: Infrastructure level - use multiple GKE clusters, not one monolithic cluster. I'd suggest: 1) Prod cluster for critical services, 2) Staging cluster for testing, 3) Dev cluster for development. Each cluster in different regions for resilience. Network level - use Istio or Anthos for service mesh. This gives you unified traffic management, security policies, observability across all services. You don't have to configure networking in each microservice. Organization level - use Kubernetes namespaces to isolate teams. Team A gets namespace 'team-a', Team B gets 'team-b'. RBAC ensures teams only see their namespace. Multi-tenancy without separate clusters. CI/CD level - single pipeline that deploys to all three clusters. GitOps with ArgoCD or Flux - teams commit to Git, controllers sync to Kubernetes automatically. Observability level - Datadog or Prometheus for metrics. Istio automatically exports traces. Every request across all 200 services gets traced. Cost level - GKE Autopilot removes the pain of managing nodes. You pay per pod, not per node. Or use standard GKE with Workload Identity for fine-grained security. My architecture: 3 GKE clusters (prod/staging/dev) → Istio service mesh → Namespaces per team → GitOps deployment → Prometheus/Datadog monitoring. This scales to 500+ microservices without breaking a sweat."

---

### Q: Design a multi-cluster GKE architecture with disaster recovery

**A:**
```yaml
# Multi-Cluster GKE with Failover

# Cluster 1: Primary (us-central1)
resource "google_container_cluster" "primary" {
  name     = "gke-primary"
  location = "us-central1"
  
  # Workload Identity
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }
  
  # Network Policy
  network_policy {
    enabled = true
  }
  
  # Logging and Monitoring
  logging_service    = "logging.googleapis.com/kubernetes"
  monitoring_service = "monitoring.googleapis.com/kubernetes"
  
  # Auto-scaling
  cluster_autoscaling {
    enabled = true
    resource_limits {
      resource_type = "cpu"
      minimum       = 10
      maximum       = 100
    }
  }
}

# Cluster 2: Secondary (europe-west1)
resource "google_container_cluster" "secondary" {
  name     = "gke-secondary"
  location = "europe-west1"
  
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }
  
  network_policy {
    enabled = true
  }
}

# Cluster 3: Disaster Recovery (asia-southeast1)
resource "google_container_cluster" "dr" {
  name     = "gke-dr"
  location = "asia-southeast1"
  
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }
}

# Istio Service Mesh for routing
resource "kubernetes_namespace" "istio" {
  metadata {
    name = "istio-system"
    labels = {
      "istio-injection" = "enabled"
    }
  }
}

# Multi-cluster Ingress
resource "google_compute_backend_service" "multicluster" {
  name = "backend-multicluster"
  
  backends {
    group                 = google_container_node_pool.primary_nodes.managed_instance_group_urls[0]
    balancing_mode        = "RATE"
    max_rate_per_endpoint = 1000
  }
  
  backends {
    group                 = google_container_node_pool.secondary_nodes.managed_instance_group_urls[0]
    balancing_mode        = "RATE"
    max_rate_per_endpoint = 1000
  }
}

# Cross-Cluster VPC Peering
resource "google_compute_network_peering" "primary_secondary" {
  name         = "peering-primary-secondary"
  network      = google_compute_network.primary.self_link
  peer_network = google_compute_network.secondary.self_link
  
  auto_create_routes = true
}
```

---

## 4. Cost Optimization

### Topic: Cost Optimization & Spending Control

Your GCP bill is growing 20% month-over-month. How do you get it under control?

First, understand where the money is going. Use GCP's Cost Management tools - they show you spending by service. Common cost drivers: 1) Compute - GCE instances, GKE, Cloud Run. Are instances appropriately sized? Are you using Committed Use Discounts (CUDs)? 1-year CUD is 25% cheaper, 3-year is 52% cheaper. If your workload is stable, buy CUDs. 2) Data transfer - egress costs money. If you're transferring data between regions, that's expensive. Use Cache, Cloud CDN, or collocate services. 3) Storage - Persistent Disks, Cloud Storage. Archive old data to Coldline or Nearline tiers. 4) BigQuery - if you're analyzing terabytes monthly, costs add up. But you can optimize: partitioning tables, clustering, materialized views, slot reservations instead of on-demand. 5) Networking - especially VPN and dedicated interconnect. If you're not using them, remove them. My systematic approach: Go to Cost Management, sort by service, identify top 3-5 cost drivers, optimize each: Compute → buy CUDs, Data → reduce transfer, Storage → archive old data, BigQuery → optimize queries, Networking → consolidate. You can typically cut costs 30-40% without sacrificing performance. Another trick: GCP has recommendations built in - go to Recommendations tab and let it suggest optimizations. Most engineers don't use that, but it's gold."

---

## 5. AI/ML Integration

### Topic: AI & Machine Learning Integration

You want to add AI to your products starting with vision APIs. How do you do this on GCP without ML expertise?

GCP's AI/ML is actually approachable for non-ML teams. You don't need PhD in machine learning. Here's the progression: Level 1 - Pre-trained APIs. Vision API can detect objects, read text (OCR), identify faces, detect logos. You just call the API with an image, get back structured data. Takes 5 minutes to integrate. Level 2 - AutoML. You provide labeled training data, AutoML trains a custom model. No ML expertise needed. Upload 100 images, tag them, AutoML does the rest. Takes a week. Level 3 - Vertex AI with custom training. For complex models, you write custom Python code using TensorFlow. Vertex AI handles deployment, scaling, monitoring. Specific example: Your app needs to identify defects in photos. Level 1 approach: Use Vision API + custom logic - 'if object detected, send to human for review'. Probably 70% accurate. Level 2 approach: Use AutoML - collect 500 images of defects and 500 non-defects, let AutoML train, 90% accurate. Level 3 approach: Use Vertex AI with custom training - implement YOLOv8 or similar, get 95%+. My recommendation for most startups: Start with pre-trained APIs (Vision), then move to AutoML if you need better accuracy. Avoid custom training unless your problem is unique. GCP handles all the infrastructure, you just focus on data."

---

## Summary

**Critical Senior GCP Skills:**
1. ✅ Multi-region architecture
2. ✅ BigQuery analytics at scale
3. ✅ GKE multi-cluster management
4. ✅ Real-time streaming pipelines
5. ✅ Cost optimization strategies
6. ✅ Istio service mesh
7. ✅ AI/ML service integration
8. ✅ Data processing with Dataflow

*Created by Karthik Reddy*
