# Oracle Cloud Infrastructure - Enterprise Cloud Platform

## What is it used for?
Oracle Cloud Infrastructure is used for:
- **Compute services**: Bare Metal, VM instances
- **Database services**: Autonomous Database, MySQL, PostgreSQL
- **Kubernetes**: OKE (Oracle Kubernetes Engine)
- **Container registry**: OCIR (Oracle Cloud Infrastructure Registry)
- **Networking**: VCN, Load Balancing
- **Storage**: Object Storage, Block Volume
- **Analytics**: Oracle Analytics Cloud
- **Enterprise applications**: ERP, HCM integration

## Installation

```bash
# Install OCI CLI
# macOS
brew install oci-cli

# Linux
bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)"

# Windows
# Download from https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/climanualinst.htm

# Configure OCI CLI
oci setup config

# Verify installation
oci --version
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check OCI CLI version
oci --version

# Get user info
oci iam user list

# List compartments
oci iam compartment list

# List compute instances
oci compute instance list

# Create compute instance
oci compute instance launch \
  --availability-domain AD-1 \
  --compartment-id COMPARTMENT_ID \
  --image-id IMAGE_ID \
  --shape VM.Standard.E4.Flex

# Start instance
oci compute instance action --instance-id INSTANCE_ID --action START

# Stop instance
oci compute instance action --instance-id INSTANCE_ID --action STOP

# Get instance details
oci compute instance get --instance-id INSTANCE_ID

# List databases
oci db autonomous-database list

# Create autonomous database
oci db autonomous-database create \
  --compartment-id COMPARTMENT_ID \
  --db-name mydb \
  --admin-password "Password123!"

# List OKE clusters
oci ce cluster-list

# Create OKE cluster
oci ce cluster-create \
  --compartment-id COMPARTMENT_ID \
  --kubernetes-network-config kubernetesNetworkConfig

# Get cluster credentials
oci ce cluster-create-kubeconfig \
  --cluster-id CLUSTER_ID \
  --file $HOME/.kube/config

# List object storage buckets
oci os bucket list --compartment-id COMPARTMENT_ID

# Create object storage bucket
oci os bucket create \
  --compartment-id COMPARTMENT_ID \
  --name my-bucket

# Upload object
oci os object put \
  --bucket-name my-bucket \
  --file-path file.txt

# Download object
oci os object get \
  --bucket-name my-bucket \
  --name file.txt
```

### Common Issues & Resolution

**Issue: Authentication failed**
```bash
# Solution: Reconfigure OCI CLI
oci setup config

# Verify credentials
oci iam user list

# Check configuration file
cat ~/.oci/config
```

**Issue: Compartment not found**
```bash
# Solution: List compartments
oci iam compartment list

# Get root compartment ID
oci iam compartment list --compartment-id-in-subtree true
```

**Issue: Instance creation fails**
```bash
# Solution: Check shape availability
oci compute shape list --compartment-id COMPARTMENT_ID

# Check image availability
oci compute image list --compartment-id COMPARTMENT_ID

# Verify availability domains
oci iam availability-domain list
```

### Debugging
```bash
# Enable debug mode
oci --debug compute instance list

# Check configuration
oci setup repair-file-permissions

# Verify API key
ls -la ~/.oci/oci_api_key.pem

# Get detailed error information
oci compute instance list --debug 2>&1 | head -50

# Check service status
oci limits resource list --service-name compute
```

### Object Storage Management
```bash
# List bucket contents
oci os object list --bucket-name my-bucket

# Set bucket permissions
oci os bucket update --bucket-name my-bucket \
  --public-access-type ObjectRead

# Enable bucket versioning
oci os bucket update --bucket-name my-bucket \
  --versioning Enabled

# Set lifecycle rule
oci os object-lifecycle-policy put \
  --bucket-name my-bucket \
  --items items.json
```

### OKE Cluster Management
```bash
# List node pools
oci ce node-pool list --cluster-id CLUSTER_ID

# Get cluster kubeconfig
oci ce cluster-create-kubeconfig \
  --cluster-id CLUSTER_ID \
  --file kubeconfig.yaml

# Verify kubectl access
kubectl get nodes

# Scale node pool
oci ce node-pool update \
  --node-pool-id NODE_POOL_ID \
  --node-config-details quantityPerSubnet=5
```

### Database Management
```bash
# List autonomous databases
oci db autonomous-database list --compartment-id COMPARTMENT_ID

# Get database details
oci db autonomous-database get --autonomous-database-id DB_ID

# Start database
oci db autonomous-database action --autonomous-database-id DB_ID \
  --action START

# Stop database
oci db autonomous-database action --autonomous-database-id DB_ID \
  --action STOP

# Download wallet
oci db autonomous-database generate-wallet \
  --autonomous-database-id DB_ID \
  --file wallet.zip
```
