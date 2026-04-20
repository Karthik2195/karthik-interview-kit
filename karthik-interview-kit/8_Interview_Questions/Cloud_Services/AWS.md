# AWS - Senior Level Interview Questions & Answers

**Contributed by:** Karthik Reddy

## Table of Contents
1. Architecture & Design
2. Cost Optimization
3. Security & Compliance
4. Performance & Scaling
5. Disaster Recovery

---

## 1. Architecture & Design

### Q (High-Level): My website sometimes goes down. How do I make it always available?

**A (Real-Time High Availability):**
Your e-commerce site crashed on Black Friday. Lost $100k in sales.

**What went wrong:**
```
Single server in us-east-1a:
- Server dies → Website down
- Network issue → Website down
- Whole data center fails → Website down
Result: No redundancy = WILL fail ❌
```

**Fix: Multi-region setup**
```
1. Run in us-east-1a (primary)
2. Run in us-east-1b (backup, same region)
3. Run in us-west-2 (different region, 2000 miles away)

If us-east-1a data center burns down:
- Customers in US East routed to us-east-1b (still in same region, <1ms)
- No traffic loss ✅
- All $100k saved ✅
```

**AWS tools:**
```
Route 53: Smart DNS routing
- Health checks every 10 seconds
- If region dies: Automatically route to healthy region
- Users don't even notice (transparent failover)
```

**Real-time result:**
- Before: One regional failure = business down
- After: One region dies, customers see no impact

---

### Q (High-Level): My AWS bill is $50k/month. How do I find where money is going?

**A (Cost Crisis Real-Time):**
CFO asks: "Why is AWS so expensive?"

**Common cost culprits:**
```
1. Running instance 24/7 when only needed 9-5
   - Solution: Stop at 5 PM, restart at 9 AM = 40% savings

2. Data transfer costs (super expensive, $0.02/GB out)
   - Old website transferred 1TB/day out = $600/day = $18k/month!
   - Solution: Use CloudFront CDN = 90% cheaper

3. Old EBS volumes you forgot about
   - $5/month × 500 forgotten volumes = $2500/month waste
   - Solution: Audit and delete unused volumes

4. 30 RDS databases for testing
   - Each costs $500/month = $15k/month for dev/test
   - Solution: One dev database, start/stop for testing
```

**Quick AWS audit:**
```bash
# Find expensive resources
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,LaunchTime]'

# Estimate monthly bill
aws ce get-cost-and-usage --time-period Start=2024-01-01,End=2024-01-31
```

**Real-time savings:**
- Before optimization: $50k/month
- Delete unused resources: $50k → $35k (-30%)
- Add auto-scaling: $35k → $25k (-30%)
- Use RIs for guaranteed workload: $25k → $15k (-40%)
- Total: $50k → $15k! That's $420k/year saved!

---

### Q: Design a highly available, scalable e-commerce platform on AWS

**A:**
```
Architecture Overview:
┌─────────────────────────────────────────────────────────────┐
│                     CloudFront CDN (Global)                  │
└─────────────────────────────────────────────────────────────┘
                                │
┌─────────────────────────────────────────────────────────────┐
│              Route 53 (DNS & Health Checks)                  │
│          (Multi-region failover & latency routing)          │
└─────────────────────────────────────────────────────────────┘
                                │
    ┌──────────────────────────┼──────────────────────────┐
    │                          │                          │
┌─────────┐              ┌─────────┐              ┌─────────┐
│ us-east │              │ eu-west │              │ ap-south│
│  ALB    │              │  ALB    │              │  ALB    │
└─────────┘              └─────────┘              └─────────┘
    │                          │                          │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  ECS Cluster    │    │  ECS Cluster    │    │  ECS Cluster    │
│  - Auto Scaling │    │  - Auto Scaling │    │  - Auto Scaling │
│  - Services     │    │  - Services     │    │  - Services     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
    │                          │                          │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  ElastiCache    │    │  ElastiCache    │    │  ElastiCache    │
│  (Session)      │    │  (Session)      │    │  (Session)      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
    │                          │                          │
┌───────────────────────────────────────────────────────────────┐
│              Aurora Global Database (Multi-region)            │
│          (Automatic replication, instant failover)            │
└───────────────────────────────────────────────────────────────┘
    │                          │                          │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  S3 Bucket      │    │  S3 Bucket      │    │  S3 Bucket      │
│  (Replication)  │    │  (Replication)  │    │  (Replication)  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
    │
    ├── S3 Event Notifications → SQS/SNS
    ├── S3 Lifecycle Policies
    └── S3 Access Logs
```

**Implementation Details:**
1. **Global Edge Locations**: CloudFront for static content caching
2. **DNS**: Route 53 with health checks for failover
3. **Compute**: ECS on Fargate for containers (serverless)
4. **Database**: Aurora Global Database for multi-region
5. **Cache**: ElastiCache Redis for sessions
6. **Storage**: S3 with cross-region replication
7. **Messaging**: SQS for async processing
8. **Monitoring**: CloudWatch, X-Ray tracing

---

### Q: Explain AWS service mesh architecture using service discovery

**A:**
```yaml
# Service Discovery with Consul in AWS

# ECS Service Definition
{
  "family": "product-service",
  "containerDefinitions": [
    {
      "name": "product-api",
      "image": "myrepo/product-api:latest",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "CONSUL_HOST",
          "value": "consul.service.consul"
        },
        {
          "name": "SERVICE_NAME",
          "value": "product-service"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/product-service",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}

# Service Registry with Consul
service "product-service" {
  id      = "product-service-1"
  name    = "product-service"
  port    = 8080
  address = "10.0.1.100"
  
  check {
    id       = "product-health"
    http     = "http://10.0.1.100:8080/health"
    interval = "10s"
    timeout  = "5s"
  }
  
  tags = ["v1", "prod"]
}
```

---

## 2. Cost Optimization

### Q: Design a cost optimization strategy for multi-environment AWS infrastructure

**A:**
```python
# Cost optimization analysis and recommendation engine

import boto3
import json
from datetime import datetime, timedelta

class AWSCostOptimizer:
    def __init__(self):
        self.ec2 = boto3.client('ec2')
        self.ce = boto3.client('ce')  # Cost Explorer
        self.cloudwatch = boto3.client('cloudwatch')
    
    def identify_underutilized_instances(self):
        """Find EC2 instances with low CPU/memory utilization"""
        instances = self.ec2.describe_instances()
        underutilized = []
        
        for reservation in instances['Reservations']:
            for instance in reservation['Instances']:
                metrics = self.cloudwatch.get_metric_statistics(
                    Namespace='AWS/EC2',
                    MetricName='CPUUtilization',
                    Dimensions=[{'Name': 'InstanceId', 'Value': instance['InstanceId']}],
                    StartTime=datetime.utcnow() - timedelta(days=7),
                    EndTime=datetime.utcnow(),
                    Period=3600,
                    Statistics=['Average']
                )
                
                avg_cpu = sum(m['Average'] for m in metrics['Datapoints']) / len(metrics['Datapoints'])
                
                # Flag instances with <5% average CPU
                if avg_cpu < 5:
                    underutilized.append({
                        'instance_id': instance['InstanceId'],
                        'avg_cpu': avg_cpu,
                        'type': instance['InstanceType'],
                        'recommendation': 'Consider terminating or downsizing'
                    })
        
        return underutilized
    
    def recommend_reserved_instances(self):
        """Recommend Reserved Instances for cost savings"""
        response = self.ce.get_reservation_purchase_recommendation(
            Service='EC2',
            LookbackPeriod='THIRTY_DAYS',
            TermInYears='THREE_YEARS',
            PaymentOption='ALL_UPFRONT'
        )
        
        savings = []
        for recommendation in response['Recommendations']:
            monthly_savings = recommendation['EstimatedMonthlyOnDemandCost'] - recommendation['EstimatedMonthlyAmortizedCost']
            savings.append({
                'instance_type': recommendation['InstanceDetails'],
                'monthly_savings': monthly_savings,
                'annual_savings': monthly_savings * 12
            })
        
        return sorted(savings, key=lambda x: x['annual_savings'], reverse=True)
    
    def optimize_storage_costs(self):
        """Identify expensive storage and recommend lifecycle policies"""
        s3 = boto3.client('s3')
        buckets = s3.list_buckets()
        
        optimizations = []
        for bucket in buckets['Buckets']:
            bucket_name = bucket['Name']
            
            # Get bucket size and recommend lifecycle
            versioning = s3.get_bucket_versioning(Bucket=bucket_name)
            
            if versioning.get('Status') == 'Enabled':
                # Recommend cleanup of old versions
                optimizations.append({
                    'bucket': bucket_name,
                    'issue': 'Versioning enabled - old versions accumulating',
                    'recommendation': 'Configure lifecycle policy to delete old versions after 90 days',
                    'estimated_savings': 'Potentially 30-50% of storage costs'
                })
        
        return optimizations
    
    def analyze_spending_trends(self):
        """Analyze spending trends by service"""
        response = self.ce.get_cost_and_usage(
            TimePeriod={
                'Start': (datetime.utcnow() - timedelta(days=30)).strftime('%Y-%m-%d'),
                'End': datetime.utcnow().strftime('%Y-%m-%d')
            },
            Granularity='MONTHLY',
            Metrics=['UnblendedCost'],
            GroupBy=[{'Type': 'DIMENSION', 'Key': 'SERVICE'}]
        )
        
        spending = {}
        for result in response['ResultsByTime']:
            for group in result['Groups']:
                service = group['Keys'][0]
                cost = float(group['Metrics']['UnblendedCost']['Amount'])
                spending[service] = cost
        
        return sorted(spending.items(), key=lambda x: x[1], reverse=True)

# Usage
optimizer = AWSCostOptimizer()
print("Underutilized Instances:", optimizer.identify_underutilized_instances())
print("Reserved Instance Recommendations:", optimizer.recommend_reserved_instances())
print("Storage Optimization:", optimizer.optimize_storage_costs())
print("Spending by Service:", optimizer.analyze_spending_trends())
```

**Cost Optimization Strategies:**
1. **Reserved Instances**: 40% savings vs On-Demand
2. **Spot Instances**: 90% savings for non-critical workloads
3. **Savings Plans**: Flexibility across instance types
4. **Right-sizing**: Eliminate unused resources
5. **Storage Lifecycle**: Archive old data to Glacier
6. **Network Optimization**: Reduce data transfer costs
7. **Auto-scaling**: Scale based on demand

---

## 3. Security & Compliance

### Q: Design a security architecture for a regulated industry (HIPAA, PCI-DSS)

**A:**
```hcl
# Terraform for HIPAA-compliant AWS infrastructure

# 1. VPC Isolation
resource "aws_vpc" "hipaa" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "hipaa-vpc"
    Compliance  = "HIPAA"
  }
}

# 2. Private subnets only (no internet gateway for PHI)
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.hipaa.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
}

# 3. Encryption at rest
resource "aws_kms_key" "hipaa" {
  description             = "HIPAA-compliant KMS key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  
  tags = {
    Compliance = "HIPAA"
  }
}

# 4. RDS with encryption and backups
resource "aws_db_instance" "hipaa" {
  engine                   = "postgres"
  allocated_storage        = 100
  instance_class           = "db.t3.medium"
  
  # Encryption
  storage_encrypted        = true
  kms_key_id              = aws_kms_key.hipaa.arn
  
  # Backups
  backup_retention_period = 90
  backup_window           = "03:00-04:00"
  copy_tags_to_snapshot   = true
  
  # Security
  skip_final_snapshot     = false
  multi_az                = true
  publicly_accessible     = false
  
  # Logging
  enabled_cloudwatch_logs_exports = ["postgresql"]
  
  # HIPAA BAA required
  tags = {
    Compliance = "HIPAA"
  }
}

# 5. IAM policy for least privilege
resource "aws_iam_policy" "hipaa_minimal" {
  name = "hipaa-minimal-policy"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = "${aws_s3_bucket.phi_data.arn}/*"
        Condition = {
          StringEquals = {
            "s3:x-amz-server-side-encryption" = "aws:kms"
          }
        }
      }
    ]
  })
}

# 6. CloudTrail for audit logging
resource "aws_cloudtrail" "hipaa" {
  name                          = "hipaa-audit-trail"
  s3_bucket_name               = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true
  
  depends_on = [aws_s3_bucket_policy.cloudtrail]
}

# 7. VPC Flow Logs
resource "aws_flow_log" "hipaa" {
  traffic_type       = "REJECT"
  iam_role_arn      = aws_iam_role.flow_log.arn
  log_destination   = aws_cloudwatch_log_group.flow_log.arn
  vpc_id            = aws_vpc.hipaa.id
}
```

---

## 4. Performance & Scaling

### Q: Design auto-scaling strategy for unpredictable workloads

**A:**
```json
// Target Tracking Scaling Policy
{
  "TargetTrackingScalingPolicyConfiguration": {
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "ScaleOutCooldown": 300,
    "ScaleInCooldown": 900
  }
}

// Custom Metric Scaling
{
  "TargetTrackingScalingPolicyConfiguration": {
    "TargetValue": 50.0,
    "CustomizedMetricSpecification": {
      "MetricName": "MyCustomMetric",
      "Namespace": "MyApp",
      "Statistic": "Average"
    }
  }
}

// Step Scaling for more control
{
  "AdjustmentType": "ChangeInCapacity",
  "MetricAggregationType": "Average",
  "StepAdjustments": [
    {
      "MetricIntervalLowerBound": 0,
      "MetricIntervalUpperBound": 10,
      "ScalingAdjustment": 1
    },
    {
      "MetricIntervalLowerBound": 10,
      "MetricIntervalUpperBound": 20,
      "ScalingAdjustment": 3
    },
    {
      "MetricIntervalLowerBound": 20,
      "ScalingAdjustment": 5
    }
  ]
}
```

---

## 5. Disaster Recovery

### Q: Design a disaster recovery strategy (RPO=1 hour, RTO=15 minutes)

**A:**
```
DR Strategy Matrix:

┌─────────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│     Tier        │     RPO      │     RTO      │   Cost       │   Use Case   │
├─────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│  Backup/Restore │  24 hours    │  8-24 hours  │  Low         │  Non-critical│
│  Pilot Light    │  1 hour      │  1-4 hours   │  Medium      │  Standard    │
│  Warm Standby   │  15 minutes  │  15-30 min   │  High        │  Mission     │
│  Hot Standby    │  Real-time   │  <5 minutes  │  Very High   │  Critical    │
└─────────────────┴──────────────┴──────────────┴──────────────┴──────────────┘

Implementation (Target: RPO=1h, RTO=15min = Warm Standby):

1. Database:
   - Multi-AZ RDS with cross-region read replicas
   - Automated backups every 15 minutes
   - Cross-region restore capability

2. Application:
   - Active-passive setup across regions
   - Route 53 health checks (30s intervals)
   - Auto-scaling groups in DR region

3. Execution:
   - Automated failover via Lambda
   - Manual failover verification (15 minutes)
   - Full recovery capability within 1 hour
```

---

## Summary

**Critical Senior AWS Skills:**
1. ✅ Multi-region architectures
2. ✅ Cost optimization
3. ✅ Security compliance (HIPAA, PCI-DSS)
4. ✅ Auto-scaling strategies
5. ✅ Disaster recovery
6. ✅ Service integration
7. ✅ Performance tuning
8. ✅ Large-scale infrastructure

**Created by:** Karthik Reddy Vaddepal
