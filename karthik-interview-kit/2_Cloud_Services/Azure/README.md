# Azure - Microsoft Cloud Platform

## What is it used for?
Azure is used for:
- **Cloud computing**: Virtual machines and compute services
- **App services**: Host web apps and APIs
- **Database services**: Managed databases (SQL, CosmosDB)
- **Container services**: AKS (Azure Kubernetes Service)
- **DevOps tools**: Azure DevOps, Pipelines
- **AI/ML services**: Cognitive services, Machine Learning
- **Data analytics**: Data lakes, Synapse Analytics
- **Security & compliance**: Enterprise-grade security

## Installation

```bash
# Install Azure CLI
# macOS
brew install azure-cli

# Linux (Ubuntu/Debian)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Windows
# Download MSI from https://aka.ms/azurecli

# Verify installation
az --version

# Login to Azure
az login

# Set subscription
az account set --subscription "subscription-id"
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check Azure version
az --version

# List subscriptions
az account list

# Show current subscription
az account show

# Get all resource groups
az group list

# Create resource group
az group create --name myResourceGroup --location eastus

# List all resources
az resource list

# List VM images
az vm image list

# Create virtual machine
az vm create --resource-group myResourceGroup \
  --name myVM --image UbuntuLTS

# Start VM
az vm start --resource-group myResourceGroup --name myVM

# Stop VM
az vm stop --resource-group myResourceGroup --name myVM

# Get VM details
az vm show --resource-group myResourceGroup --name myVM

# List app services
az appservice plan list

# Create app service
az appservice plan create --name myAppServicePlan \
  --resource-group myResourceGroup

# Create web app
az webapp create --resource-group myResourceGroup \
  --plan myAppServicePlan --name myWebApp

# Deploy code to web app
az webapp up --resource-group myResourceGroup \
  --name myWebApp --runtime "PYTHON:3.9"

# List Azure Kubernetes Service (AKS) clusters
az aks list

# Create AKS cluster
az aks create --resource-group myResourceGroup \
  --name myCluster --node-count 3

# Get AKS credentials
az aks get-credentials --resource-group myResourceGroup \
  --name myCluster

# List storage accounts
az storage account list

# Create storage account
az storage account create --name mystorageaccount \
  --resource-group myResourceGroup --location eastus

# List key vaults
az keyvault list

# Create key vault
az keyvault create --resource-group myResourceGroup \
  --name myKeyVault

# Get storage account keys
az storage account keys list --resource-group myResourceGroup \
  --account-name mystorageaccount
```

### Azure Database Operations
```bash
# List databases
az sql server list

# Create SQL server
az sql server create --resource-group myResourceGroup \
  --name myServer --admin-user azureuser \
  --admin-password P@ssw0rd123!

# Create database
az sql db create --resource-group myResourceGroup \
  --server myServer --name myDatabase

# List CosmosDB accounts
az cosmosdb list

# Create CosmosDB account
az cosmosdb create --resource-group myResourceGroup \
  --name myCosmosDB --kind GlobalDocumentDB
```

### Azure DevOps
```bash
# List projects
az devops project list

# Create project
az devops project create --name MyProject \
  --organization https://dev.azure.com/myorganization

# List pipelines
az pipelines list --project MyProject

# Run pipeline
az pipelines run --id <pipeline-id> --project MyProject

# List builds
az pipelines build list --project MyProject

# Get build details
az pipelines build show --id <build-id> --project MyProject
```

### Common Issues & Resolution

**Issue: Login failed**
```bash
# Solution: Re-authenticate
az login

# Or use device code login
az login --use-device-code

# Clear cached credentials
az cache purge

# Check current credentials
az account show
```

**Issue: Subscription not found**
```bash
# Solution: List subscriptions
az account list

# Set correct subscription
az account set --subscription "subscription-id"

# Or specify in command
az vm list --subscription "subscription-id"
```

**Issue: Resource creation failed**
```bash
# Solution: Check error details
az vm create ... --debug

# Verify resource group exists
az group exists --name myResourceGroup

# Check resource quotas
az compute vm usage list --location eastus

# View operation logs
az monitor activity-log list
```

**Issue: Access denied**
```bash
# Solution: Check role assignments
az role assignment list --assignee user@example.com

# Assign role
az role assignment create --assignee user@example.com \
  --role "Contributor" --scope /subscriptions/subscription-id

# Get current user info
az ad signed-in-user show
```

### Debugging Commands
```bash
# Enable debug logging
az vm create --debug

# Get detailed error information
az vm create ... 2>&1 | tee output.log

# Check Azure CLI configuration
az config list

# View all available commands
az --help
az vm --help

# Get operation status
az operation list

# Monitor resource health
az resource invoke-action --action runCommand \
  --ids <resource-id>

# Check diagnostic settings
az monitor diagnostic-settings list \
  --resource <resource-id>

# View activity logs
az monitor activity-log list \
  --resource-group myResourceGroup \
  --max-records 50
```

### Azure Resource Manager (ARM) Templates
```bash
# Deploy ARM template
az deployment group create --resource-group myResourceGroup \
  --template-file template.json --parameters parameters.json

# Validate template
az deployment group validate --resource-group myResourceGroup \
  --template-file template.json

# Export template from existing resource group
az group export --name myResourceGroup > template.json

# Create resource from template
az resource create --resource-group myResourceGroup \
  --template-file template.json
```

### Managing Credentials & Secrets
```bash
# Store secret in Key Vault
az keyvault secret set --vault-name myKeyVault \
  --name mySecret --value mySecretValue

# Get secret from Key Vault
az keyvault secret show --vault-name myKeyVault \
  --name mySecret

# List all secrets
az keyvault secret list --vault-name myKeyVault

# Delete secret
az keyvault secret delete --vault-name myKeyVault \
  --name mySecret

# Configure storage account access keys
az storage account keys renew --resource-group myResourceGroup \
  --account-name mystorageaccount
```

### Monitoring & Alerts
```bash
# Get metrics
az monitor metrics list --resource <resource-id> \
  --metric "Percentage CPU"

# Create alert rule
az monitor metrics alert create --name myAlert \
  --resource-group myResourceGroup \
  --scopes <resource-id> \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m

# List alerts
az monitor metrics alert list --resource-group myResourceGroup

# Get logs
az monitor log-analytics workspace list
```

### AKS Cluster Management
```bash
# List nodes in cluster
az aks nodepool list --resource-group myResourceGroup \
  --cluster-name myCluster

# Scale cluster
az aks scale --resource-group myResourceGroup \
  --name myCluster --node-count 5

# Update cluster version
az aks upgrade --resource-group myResourceGroup \
  --name myCluster --kubernetes-version 1.28

# Enable addons
az aks enable-addons --resource-group myResourceGroup \
  --name myCluster --addons monitoring

# Get cluster admin credentials
az aks get-credentials --resource-group myResourceGroup \
  --name myCluster --admin
```

### Azure Policy
```bash
# List policies
az policy definition list

# Create custom policy
az policy definition create --name myPolicy \
  --rules myPolicy.json

# Assign policy to resource group
az policy assignment create --name myAssignment \
  --policy-id myPolicy \
  --scope /subscriptions/subscription-id/resourceGroups/myResourceGroup

# List policy assignments
az policy assignment list

# Get compliance state
az policy state summarize --filter "isCompliant eq true"
```

## Pricing & Cost Management
```bash
# Get cost estimation
az consumption usage list

# View billing
az billing invoice list

# Cost analysis
az billing invoice download --invoice-name invoice-name
```
