# Google Cloud - Google Cloud Platform

## What is it used for?
Google Cloud Platform is used for:
- **Compute services**: Compute Engine (VMs), App Engine
- **Kubernetes**: GKE (Google Kubernetes Engine)
- **Data analytics**: BigQuery, Dataflow, Pub/Sub
- **Machine learning**: Vertex AI, TensorFlow on Cloud
- **Storage**: Cloud Storage, Firestore, Cloud SQL
- **Serverless**: Cloud Functions, Cloud Run
- **DevOps**: Cloud Build, Cloud Deploy
- **Networking**: VPC, Cloud Load Balancing, Cloud CDN

## Installation

```bash
# Install Google Cloud SDK
# macOS
curl https://sdk.cloud.google.com | bash
exec -l $SHELL

# Linux
curl https://sdk.cloud.google.com | bash

# Windows
# Download installer from https://cloud.google.com/sdk/docs/install

# Initialize gcloud
gcloud init

# Verify installation
gcloud --version
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check gcloud version
gcloud --version

# List projects
gcloud projects list

# Set project
gcloud config set project PROJECT_ID

# Get current project
gcloud config get-value project

# List compute instances
gcloud compute instances list

# Create VM instance
gcloud compute instances create my-instance \
  --zone=us-central1-a \
  --machine-type=e2-micro

# Start instance
gcloud compute instances start my-instance --zone=us-central1-a

# Stop instance
gcloud compute instances stop my-instance --zone=us-central1-a

# SSH into instance
gcloud compute ssh my-instance --zone=us-central1-a

# List GKE clusters
gcloud container clusters list

# Create GKE cluster
gcloud container clusters create my-cluster \
  --zone=us-central1-a \
  --num-nodes=3

# Get cluster credentials
gcloud container clusters get-credentials my-cluster --zone=us-central1-a

# List storage buckets
gsutil ls

# Create storage bucket
gsutil mb gs://my-bucket

# Copy to bucket
gsutil cp file.txt gs://my-bucket/

# List datasets
bq ls

# Query BigQuery
bq query --use_legacy_sql=false 'SELECT * FROM `project.dataset.table` LIMIT 10'

# Deploy Cloud Function
gcloud functions deploy my-function \
  --runtime python39 \
  --trigger-http \
  --allow-unauthenticated

# Deploy to Cloud Run
gcloud run deploy my-service \
  --image gcr.io/my-project/my-image \
  --platform managed \
  --region us-central1
```

### Common Issues & Resolution

**Issue: Project not found**
```bash
# Solution: List available projects
gcloud projects list

# Set correct project
gcloud config set project PROJECT_ID

# Verify setting
gcloud config list
```

**Issue: Permission denied**
```bash
# Solution: Check IAM roles
gcloud projects get-iam-policy PROJECT_ID

# Grant role to user
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=user:email@example.com \
  --role=roles/editor
```

**Issue: VM creation fails**
```bash
# Solution: Check quotas
gcloud compute project-info describe --project=PROJECT_ID

# Check zone availability
gcloud compute zones list

# Try different zone
gcloud compute instances create my-instance \
  --zone=us-central1-b \
  --machine-type=e2-micro
```

**Issue: GKE cluster not accessible**
```bash
# Solution: Get credentials again
gcloud container clusters get-credentials my-cluster --zone=us-central1-a

# Verify kubectl context
kubectl config current-context

# Check cluster status
gcloud container clusters describe my-cluster --zone=us-central1-a
```

### Debugging
```bash
# Enable debug logging
gcloud --debug compute instances list

# Check authentication
gcloud auth list

# Show current configuration
gcloud config list

# Get detailed error information
gcloud compute instances create my-instance --debug

# Check service status
gcloud services list --enabled

# Enable required service
gcloud services enable compute.googleapis.com

# View audit logs
gcloud logging read "severity >= ERROR" --limit 10
```

### Cloud Storage Operations
```bash
# List bucket contents
gsutil ls gs://my-bucket

# Set bucket permissions
gsutil iam ch user:email@example.com:objectViewer gs://my-bucket

# Enable versioning
gsutil versioning set on gs://my-bucket

# Set lifecycle policy
gsutil lifecycle set policy.json gs://my-bucket

# Enable logging
gsutil logging set on -b gs://log-bucket -o log-prefix gs://my-bucket
```

### GKE Management
```bash
# List node pools
gcloud container node-pools list --cluster=my-cluster

# Scale cluster
gcloud container clusters update my-cluster \
  --num-nodes=5 \
  --zone=us-central1-a

# Upgrade cluster
gcloud container clusters upgrade my-cluster \
  --master \
  --zone=us-central1-a

# Get cluster info
gcloud container clusters describe my-cluster --zone=us-central1-a
```

### Monitoring & Logging
```bash
# View logs
gcloud logging read "resource.type=gce_instance" --limit 50

# Create alert policy
gcloud alpha monitoring policies create \
  --notification-channels=CHANNEL_ID \
  --display-name="CPU Alert"

# List metrics
gcloud monitoring metrics list

# Get metric data
gcloud monitoring time-series list \
  --filter='metric.type="compute.googleapis.com/instance/cpu/usage_time"'
```
