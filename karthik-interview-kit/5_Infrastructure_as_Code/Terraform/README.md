# Terraform - Infrastructure as Code

## What is it used for?
Terraform is used for:
- **Infrastructure provisioning**: Define and create cloud resources
- **Multi-cloud support**: Deploy to AWS, Azure, Google Cloud, etc.
- **Version control**: Track infrastructure changes in git
- **Automation**: Automate infrastructure creation and updates
- **Disaster recovery**: Recreate infrastructure quickly
- **Documentation**: Infrastructure documented in code
- **Collaboration**: Team-based infrastructure management

## Installation

```bash
# Linux
wget https://releases.hashicorp.com/terraform/1.5.0/terraform_1.5.0_linux_amd64.zip
unzip terraform_1.5.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# macOS
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Windows
# Download from https://www.terraform.io/downloads

# Verify installation
terraform version
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Initialize Terraform working directory
terraform init

# Validate configuration
terraform validate

# Format code
terraform fmt

# Plan changes
terraform plan

# Plan with output to file
terraform plan -out=plan.tfplan

# Apply changes
terraform apply

# Apply saved plan
terraform apply plan.tfplan

# Apply without confirmation
terraform apply -auto-approve

# Destroy infrastructure
terraform destroy

# Destroy without confirmation
terraform destroy -auto-approve

# Show state
terraform show

# Refresh state
terraform refresh

# List resources
terraform state list

# Show specific resource
terraform state show aws_instance.example

# Taint resource (force recreation)
terraform taint aws_instance.example

# Untaint resource
terraform untaint aws_instance.example

# Import existing resource
terraform import aws_instance.example i-1234567890abcdef0

# Get remote state
terraform state pull

# Push state
terraform state push
```

### Basic Configuration Example
```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "Example Instance"
  }
}

output "instance_ip" {
  value = aws_instance.example.public_ip
}

variable "instance_type" {
  type    = string
  default = "t2.micro"
}
```

### Common Issues & Resolution

**Issue: Backend already initialized**
```bash
# Solution: Reinitialize with upgrade
terraform init -upgrade

# Or remove .terraform directory
rm -rf .terraform
terraform init
```

**Issue: Resource already exists**
```bash
# Solution: Import existing resource
terraform import aws_instance.example instance-id

# Or modify resource and apply
terraform apply
```

**Issue: Insufficient permissions**
```bash
# Solution: Check AWS credentials
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"

# Or use AWS profile
export AWS_PROFILE=myprofile

# Verify credentials
aws sts get-caller-identity
```

**Issue: State lock timeout**
```bash
# Solution: Force unlock (use with caution)
terraform force-unlock lock-id

# Or increase lock timeout
terraform init -lock-timeout=5m
```

**Issue: Module not found**
```bash
# Solution: Re-initialize modules
terraform init -upgrade

# Or manually download modules
terraform get
```

### Debugging
```bash
# Enable debug logging
export TF_LOG=DEBUG
terraform plan

# Trace mode (very verbose)
export TF_LOG=TRACE
terraform apply

# Save debug output to file
export TF_LOG_PATH=terraform.log
terraform plan

# Show all resources
terraform state list

# Show detailed state of resource
terraform state show aws_instance.example

# Create backup of state
terraform state pull > backup.tfstate

# Validate syntax
terraform validate

# Format check
terraform fmt -check
```

### State Management
```bash
# Show complete state
terraform show

# Show specific resource state
terraform state show aws_instance.example

# List all resources in state
terraform state list

# Remove resource from state (but not from cloud)
terraform state rm aws_instance.example

# Move resource in state
terraform state mv aws_instance.old aws_instance.new

# Pull remote state locally
terraform state pull > local.tfstate

# Push local state to remote
terraform state push local.tfstate
```

### Working with Workspaces
```bash
# Create workspace
terraform workspace new staging

# List workspaces
terraform workspace list

# Select workspace
terraform workspace select staging

# Delete workspace
terraform workspace delete staging

# Show current workspace
terraform workspace show

# Use workspace in code
environment = terraform.workspace
```

### Advanced Commands
```bash
# Get module documentation
terraform get -update

# Validate and format all files
terraform fmt -recursive

# Generate plan in JSON format
terraform plan -json > plan.json

# Show specific output
terraform output instance_ip

# Refresh state from actual resources
terraform refresh

# Check if plan would cause changes
terraform plan -detailed-exitcode
```
