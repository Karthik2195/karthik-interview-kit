# Azure DevOps - CI/CD and Project Management

## What is it used for?
Azure DevOps is used for:
- **Pipelines**: CI/CD automation
- **Repos**: Git repositories
- **Boards**: Project management
- **Test Plans**: Test management
- **Artifacts**: Package management
- **Multi-cloud**: Deploy to any cloud
- **Integrations**: Connect with third-party tools
- **Enterprise**: Large-scale DevOps

## Installation & Setup

```bash
# Install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Login to Azure
az login

# List DevOps projects
az devops project list

# Create project
az devops project create --name myproject --description "My project"

# Install Azure Pipelines extension for GitHub
# Visit: https://github.com/apps/azure-pipelines

# Create service connection
az devops service-endpoint create \
  --service-endpoint-type github \
  --name github-connection
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# List organizations
az devops organization list

# Set default organization
az devops configure --defaults organization=<org-url>

# Get project info
az devops project show --project myproject

# List repositories
az repos list --project myproject

# Create repository
az repos create --project myproject --name myrepo

# Create pipeline
az pipelines create --project myproject \
  --name myapp-pipeline \
  --repository myrepo

# Run pipeline
az pipelines run --project myproject \
  --name myapp-pipeline

# Get pipeline status
az pipelines runs list --project myproject --pipeline myapp-pipeline

# View build logs
az pipelines runs show --project myproject \
  --id <run-id> \
  --open
```

### Azure Pipelines (YAML)
```yaml
# azure-pipelines.yml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - task: UseDotNet@2
            inputs:
              version: '6.0.x'
          - script: dotnet build --configuration $(buildConfiguration)
          - script: dotnet test
          - script: dotnet publish --configuration $(buildConfiguration)
          - task: PublishBuildArtifacts@1
            inputs:
              pathToPublish: $(Build.ArtifactStagingDirectory)

  - stage: Deploy
    dependsOn: Build
    condition: succeeded()
    jobs:
      - deployment: Deploy
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploying..."
```

### Common Issues & Resolution

**Issue: Pipeline fails authentication**
```bash
# Solution: Create PAT token
# Azure DevOps -> User settings -> Personal access tokens

# Use token in pipeline
az devops login --org <org-url> --token <pat-token>

# Update service connection
az devops service-endpoint create \
  --service-endpoint-type github \
  --name github-connection \
  --github-url https://github.com \
  --github-token <token>
```

**Issue: Build fails with permissions**
```bash
# Solution: Grant pipeline permissions
az pipelines permissions update \
  --project myproject \
  --pipeline-id <pipeline-id> \
  --subject <user-or-group> \
  --role Admin

# Check current permissions
az pipelines permissions show \
  --project myproject \
  --pipeline-id <pipeline-id>
```

**Issue: Deployment fails**
```bash
# Solution: Check environment status
az devops invoke --api-version 6.0 \
  --area pipelines \
  --resource environments

# View deployment logs
az pipelines runs show \
  --project myproject \
  --id <run-id> \
  --open

# Check service connections
az devops service-endpoint list --project myproject
```

### Boards (Project Management)
```bash
# Create work item
az boards work-item create \
  --title "New feature" \
  --type Story \
  --project myproject

# List work items
az boards query --project myproject \
  --wiql "SELECT [System.Id] FROM WorkItems WHERE [System.State] <> 'Closed'"

# Update work item
az boards work-item update \
  --id <work-item-id> \
  --state "In Progress" \
  --project myproject

# Link work items
az boards work-item link create \
  --id <source-id> \
  --relation-type "parent" \
  --target-id <target-id>
```

### Artifacts (Package Management)
```bash
# Create feed
az artifacts universal publish \
  --feed myfeeed \
  --name mypackage \
  --version 1.0.0 \
  --path ./dist

# List feeds
az artifacts universal show \
  --feed myfeed \
  --name mypackage \
  --version 1.0.0

# Delete package
az artifacts universal delete \
  --feed myfeed \
  --name mypackage \
  --version 1.0.0 \
  --yes
```

### Test Plans
```bash
# Create test plan
az devops invoke --api-version 5.1 \
  --area test \
  --resource plans \
  --route-parameters project=myproject

# List test cases
az devops invoke --api-version 5.1 \
  --area test \
  --resource cases \
  --route-parameters project=myproject
```

### Debugging
```bash
# Enable debug output
export SYSTEM_DEBUG=true
az pipelines run --project myproject --name myapp-pipeline

# View raw logs
az pipelines runs show --project myproject --id <run-id> --json

# Check agent status
az pipelines agent list --project myproject

# View pipeline definition
az pipelines show --project myproject --name myapp-pipeline
```

### Release Pipelines (Classic)
```bash
# Create release definition
az devops invoke \
  --api-version 5.1 \
  --area Release \
  --resource definitions \
  --route-parameters project=myproject \
  --http-method POST

# List releases
az devops invoke \
  --api-version 5.1 \
  --area Release \
  --resource releases \
  --route-parameters project=myproject

# Deploy release
az devops invoke \
  --api-version 5.1 \
  --area Release \
  --resource releases \
  --route-parameters project=myproject \
  --http-method POST \
  --body @release.json
```

### Integration with Kubernetes
```yaml
# Deploy to AKS
- task: KubernetesManifest@0
  inputs:
    action: deploy
    kubernetesServiceConnection: 'myCluster'
    namespace: 'default'
    manifests: |
      $(Pipeline.Workspace)/manifests/deployment.yaml
    imagePullSecrets: |
      $(imagePullSecret)
    containers: |
      myregistry.azurecr.io/myapp:$(Build.BuildId)

# Deploy Helm chart
- task: HelmDeploy@0
  inputs:
    connectionType: 'Kubernetes Service Connection'
    kubernetesServiceConnection: 'myCluster'
    namespace: 'default'
    command: 'upgrade'
    chartType: 'FilePath'
    chartPath: '$(Pipeline.Workspace)/chart'
    chartVersion: '1.0.0'
```

### Security and Secrets
```yaml
# Use variable groups
variables:
  - group: 'Production Secrets'

# Reference in pipeline
steps:
  - script: echo $(MY_SECRET_VAR)
    env:
      MY_SECRET_VAR: $(MySecret)

# Store secrets securely
# Azure DevOps -> Pipelines -> Library -> Variable groups
```
