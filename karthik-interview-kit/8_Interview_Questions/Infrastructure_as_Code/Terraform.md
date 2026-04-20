# Terraform - Senior Level Interview Questions & Answers

## Table of Contents
1. State Management & Backend
2. Module Design & Patterns
3. Complex Configurations
4. Scaling & Performance
5. Production Best Practices

---

## 1. State Management & Backend

### Q (High-Level): What is Terraform state and why do I need to care about it?

**A (Real-Time Infrastructure):**
You run `terraform apply`. Terraform creates 10 AWS resources. Where does Terraform remember what it created?

**Terraform State File:**
```json
{
  "resources": [
    {
      "type": "aws_instance",
      "name": "web-server",
      "id": "i-1234567890abcdef"
    },
    {
      "type": "aws_security_group",
      "name": "web-sg",
      "id": "sg-1234567890abcdef"
    }
  ]
}
```

**Why it matters:**
```
Scenario: You run terraform destroy
Terraform needs to know what to delete
It reads state file: "I created these 10 resources"
Terraform: "Delete all 10"
Result: Infrastructure deleted ✅

Without state:
Terraform: "I don't know what I created"
Result: Resources left orphaned in AWS, still costing money! ❌
```

**Real-time example:**
```
You created: VPC, 3 subnets, 2 EC2s, 1 RDS, 1 ALB (10 resources)
State file has all 10 in its records

6 months later: Run terraform destroy
Result: All 10 deleted automatically ✅
```

**Key point:**
- State file = Terraform's memory of what it created
- Without it: Can't update or destroy safely

---

### Q (High-Level): Two teammates run terraform apply at the same time. What breaks?

**A (Concurrency Nightmare):**
```
Teammate 1 at 10:00:01: terraform apply (starts)
Teammate 2 at 10:00:02: terraform apply (also starts)

State file contents at 10:00:01:
{AWS instance: "i-123", RDS: "db-456"}

Teammate 1 reads state: Changes RDS password
Teammate 2 reads state: Adds new security group

Result: Conflict!
State file corrupted = DISASTER ❌
```

**Solution: State Locking**
```hcl
# Remote state with DynamoDB locking
terraform {
  backend "s3" {
    bucket           = "myorg-terraform-state"
    dynamodb_table   = "terraform-locks"  # ← THIS prevents conflicts
  }
}
```

**How locking works:**
```
Teammate 1 at 10:00:01: terraform apply (locks state file)
Teammate 2 at 10:00:02: terraform apply (waits... state locked)
Teammate 1 at 10:00:15: apply done (releases lock)
Teammate 2 at 10:00:16: can now proceed (lock acquired)
Result: Safe, sequential, no conflicts ✅
```

**Real-time rule:**
- Always use remote state (not local file)
- Always enable locking
- Never manually edit state file

---

### Q: Design a production-grade Terraform state management strategy

**A:**
```hcl
# Remote state with locking and encryption
terraform {
  backend "s3" {
    bucket           = "myorg-terraform-state"
    key              = "prod/infrastructure/terraform.tfstate"
    region           = "us-east-1"
    encrypt          = true
    dynamodb_table   = "terraform-locks"
    skip_credentials_validation = false
    skip_metadata_api_check     = false
  }
}

# State file structure (best practices)
# prod/
#   ├── core/
#   │   └── terraform.tfstate
#   ├── networking/
#   │   └── terraform.tfstate
#   ├── kubernetes/
#   │   └── terraform.tfstate
#   └── applications/
#       └── terraform.tfstate

# Backend security configuration
resource "aws_s3_bucket" "terraform_state" {
  bucket = "myorg-terraform-state"
}

# Enable versioning for recovery
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  versioning_configuration {
    status     = "Enabled"
    mfa_delete = "Enabled"
  }
}

# Enable encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform_state.arn
    }
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# DynamoDB for state locking
resource "aws_dynamodb_table" "terraform_locks" {
  name           = "terraform-locks"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  point_in_time_recovery {
    enabled = true
  }

  tags = {
    Name = "terraform-state-lock"
  }
}

# State file backup
resource "aws_backup_vault" "terraform_state" {
  name = "terraform-state-backup"
}

resource "aws_backup_plan" "terraform_state" {
  name = "terraform-state-backup-plan"

  rule {
    rule_name            = "daily_backup"
    target_backup_vault_name = aws_backup_vault.terraform_state.name
    schedule             = "cron(0 5 * * ? *)"  # Daily at 5 AM UTC

    lifecycle {
      delete_after = 30
    }
  }
}
```

**State Management Best Practices:**
1. Use remote state (S3, Terraform Cloud)
2. Enable encryption and versioning
3. Implement state locking (DynamoDB)
4. Regular backups
5. Restrict access with IAM
6. Use separate state files per environment
7. Never commit tfstate to Git

---

### Q: How would you handle state migration and merge conflicts?

**A:**
```hcl
# State migration scenario
# Migrating from local state to remote state

# Step 1: Configure new backend
terraform {
  backend "s3" {
    bucket = "new-state-bucket"
    key    = "terraform.tfstate"
    region = "us-east-1"
  }
}

# Step 2: Migrate state
# terraform init  # Follow prompts to migrate

# State merge conflict resolution
# When working in teams, conflicts can occur

# Use remote state to prevent conflicts
# But if conflicts do occur:

resource "aws_instance" "server" {
  # If state conflict, use refresh
  # terraform refresh
  # terraform state show aws_instance.server
  
  ami           = var.ami_id
  instance_type = var.instance_type
}

# State manipulation (use with caution)
# terraform state list                    # List all resources
# terraform state show <resource>        # Show specific resource
# terraform state mv <old> <new>         # Rename resource
# terraform state rm <resource>          # Remove from state
# terraform state replace-provider       # Replace provider

# Resolving merge conflicts
locals {
  # Document all state changes
  state_version = "1.0.0"
  last_updated  = "2024-01-15"
  
  # Track all major changes
  change_log = {
    "migration_to_remote_state" = "2024-01-15"
    "vpc_restructure"           = "2024-01-10"
  }
}
```

---

## 2. Module Design & Patterns

### Q: Design a reusable, composable Terraform module system

**A:**
```hcl
# Module structure (AWS VPC example)

# modules/networking/vpc/main.tf
module "vpc" {
  source = "./modules/networking/vpc"
  
  environment     = var.environment
  region          = var.region
  cidr_block      = var.vpc_cidr
  azs             = data.aws_availability_zones.available.names
  private_subnets = var.private_subnets
  public_subnets  = var.public_subnets
  
  enable_nat_gateway = true
  single_nat_gateway = var.environment != "production"
  
  tags = local.tags
}

# modules/networking/vpc/variables.tf
variable "environment" {
  type        = string
  description = "Environment name"
  
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be dev, staging, or production"
  }
}

variable "vpc_cidr" {
  type        = string
  description = "VPC CIDR block"
  default     = "10.0.0.0/16"
}

# modules/networking/vpc/outputs.tf
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "VPC ID"
}

output "private_subnet_ids" {
  value       = aws_subnet.private[*].id
  description = "Private subnet IDs"
}

# Root module: main.tf
module "core_networking" {
  source = "./modules/networking/vpc"
  
  environment     = var.environment
  vpc_cidr        = "10.0.0.0/16"
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.10.0/24", "10.0.11.0/24"]
}

module "kubernetes_cluster" {
  source = "./modules/kubernetes/eks"
  
  environment         = var.environment
  vpc_id              = module.core_networking.vpc_id
  private_subnets    = module.core_networking.private_subnet_ids
  cluster_version    = var.k8s_version
}

# Composition pattern: Multiple modules working together
locals {
  # Central configuration
  common_tags = {
    Environment = var.environment
    Terraform   = true
    Owner       = "platform-team"
    CreatedBy   = "Karthik Reddy Vaddepal"
  }
}

# Advanced: Dynamic module configuration
resource "aws_instance" "web" {
  for_each      = var.web_servers
  
  ami           = data.aws_ami.ubuntu.id
  instance_type = each.value.instance_type
  
  tags = merge(local.common_tags, {
    Name = each.key
  })
}
```

---

## 3. Complex Configurations

### Q: Design a multi-region, multi-environment infrastructure

**A:**
```hcl
# Multi-region strategy

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Primary region
provider "aws" {
  alias  = "primary"
  region = "us-east-1"
}

# Secondary region (disaster recovery)
provider "aws" {
  alias  = "secondary"
  region = "us-west-2"
}

# Multi-environment support
variable "environment" {
  type = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Invalid environment"
  }
}

locals {
  environment_config = {
    dev = {
      instance_type = "t3.small"
      multi_az      = false
      backup_retention = 7
    }
    staging = {
      instance_type = "t3.medium"
      multi_az      = true
      backup_retention = 30
    }
    prod = {
      instance_type = "t3.large"
      multi_az      = true
      backup_retention = 90
    }
  }
  
  config = local.environment_config[var.environment]
}

# Primary region resources
resource "aws_db_instance" "primary" {
  provider          = aws.primary
  identifier        = "mydb-primary-${var.environment}"
  instance_class    = local.config.instance_type
  multi_az          = local.config.multi_az
  backup_retention_period = local.config.backup_retention
  
  # Enable backup for replication
  skip_final_snapshot       = false
  backup_window             = "03:00-04:00"
  copy_tags_to_snapshot     = true
}

# Secondary region read replica
resource "aws_db_instance" "secondary" {
  provider               = aws.secondary
  replicate_source_db    = aws_db_instance.primary.identifier
  skip_final_snapshot    = true
  auto_minor_version_upgrade = false
}

# Failover mechanism
resource "aws_rds_cluster" "failover" {
  provider            = aws.primary
  cluster_identifier  = "mydb-cluster-${var.environment}"
  engine              = "aurora"
  master_username     = var.db_username
  master_password     = random_password.db_password.result
  backup_retention_period = local.config.backup_retention
  
  # Enable backtrack for Aurora
  backtrack_window    = 24
  
  enabled_cloudwatch_logs_exports = ["error", "general", "slowquery"]
}

# Global Accelerator for failover
resource "aws_globalaccelerator_accelerator" "main" {
  enabled       = true
  ip_address_type = "IPV4"
  attributes {
    flow_logs_enabled = true
    flow_logs_s3_bucket = aws_s3_bucket.flow_logs.bucket
    flow_logs_s3_prefix = "global-accelerator/"
  }
}
```

---

## 4. Scaling & Performance

### Q: How would you optimize Terraform performance for large infrastructure?

**A:**
```hcl
# Performance optimization strategies

# 1. Parallel resource creation
terraform {
  required_version = ">= 1.3"
  
  # Increase parallelism
  # terraform apply -parallelism=20
}

# 2. Resource targeting (for debugging/updating)
# terraform plan -target=aws_instance.web[0]
# terraform apply -target=aws_instance.web

# 3. Optimize data sources
data "aws_ami" "ubuntu" {
  # Cache data source results
  most_recent = true
  
  owners = ["099720109477"]  # Canonical
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# 4. Split large modules into smaller ones
# Instead of 500-resource module, split into:
# - networking (100 resources)
# - compute (150 resources)
# - storage (100 resources)
# - monitoring (50 resources)

# 5. Use count/for_each for bulk operations
resource "aws_instance" "web" {
  for_each      = toset(var.web_servers)
  
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"
  
  tags = {
    Name = each.value
  }
}

# 6. Backend optimization
terraform {
  backend "s3" {
    # Use state locking to prevent concurrent operations
    dynamodb_table = "terraform-locks"
    
    # S3 bucket optimization
    # - Enable versioning
    # - Use S3 Fast Retrieval storage class
  }
}

# 7. Local module caching
module "networking" {
  source = "./modules/networking"  # Local = fastest
  # vs
  # source = "git::https://github.com/org/modules.git//networking"  # Slower
}

# 8. Performance monitoring
output "performance_metrics" {
  value = {
    total_resources = length(data.terraform_state.state.values.root_module.resources)
    state_size_bytes = filesize("${path.module}/terraform.tfstate")
  }
}
```

---

## 5. Production Best Practices

### Q: Design a GitOps CI/CD pipeline for Terraform

**A:**
```hcl
# Terraform CI/CD workflow

# 1. GitHub Actions workflow
# .github/workflows/terraform.yml
name: Terraform

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  terraform:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: 1.5.0
    
    - name: Terraform Format
      run: terraform fmt -check
    
    - name: Terraform Init
      run: terraform init
    
    - name: Terraform Validate
      run: terraform validate
    
    - name: TFLint
      uses: terraform-linters/setup-tflint@v3
    
    - name: Plan
      if: github.event_name == 'pull_request'
      run: terraform plan
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    
    - name: Apply
      if: github.ref == 'refs/heads/main' && github.event_name == 'push'
      run: terraform apply -auto-approve
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

# 2. Pre-commit hooks
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/hashicorp/pre-commit-terraform
    rev: v1.81.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_docs
      - id: terraform_tflint
      - id: terragrunt_fmt
      - id: terraform_checkov

# 3. Policy enforcement with Sentinel
# (Terraform Cloud/Enterprise feature)
policy "require_owner_tag" {
  enforcement_level = "mandatory"
}

# 4. Versioning strategy
variable "terraform_version" {
  type    = string
  default = "1.5.0"
}

terraform {
  required_version = ">= 1.5.0"
}

# 5. Documentation
# auto-generate docs with terraform-docs
# terraform-docs markdown table --output-file README.md .

output "infrastructure_summary" {
  value = {
    created_by  = "Karthik Reddy Vaddepal"
    environment = var.environment
    region      = var.region
  }
}
```

---

## Summary

**Critical Senior Terraform Skills:**
1. ✅ State management & locking
2. ✅ Module design patterns
3. ✅ Multi-region/multi-environment
4. ✅ Performance optimization
5. ✅ GitOps CI/CD integration
6. ✅ Disaster recovery
7. ✅ Security best practices
8. ✅ Team collaboration patterns
