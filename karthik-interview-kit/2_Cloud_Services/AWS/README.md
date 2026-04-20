# AWS - Amazon Web Services

## What is it used for?
AWS is used for:
- **Cloud computing**: Scalable compute resources (EC2)
- **Storage solutions**: Object storage (S3), Databases (RDS, DynamoDB)
- **Networking**: VPC, Load Balancing, CDN (CloudFront)
- **Container orchestration**: ECS, EKS for running containers
- **Serverless computing**: Lambda for event-driven workloads
- **DevOps tooling**: CodeBuild, CodeDeploy, CodePipeline
- **Security & compliance**: IAM, KMS, Secrets Manager
- **Analytics**: CloudWatch, Athena, Elasticsearch

## Installation

```bash
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# macOS
brew install awscli

# Windows
# Download MSI from https://aws.amazon.com/cli/

# Verify installation
aws --version

# Configure credentials
aws configure
# Enter: Access Key ID, Secret Access Key, Default region, Output format
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check AWS version
aws --version

# List AWS CLI configuration
aws configure list

# List all S3 buckets
aws s3 ls

# Create S3 bucket
aws s3 mb s3://my-bucket-name

# Upload file to S3
aws s3 cp file.txt s3://my-bucket-name/

# Download file from S3
aws s3 cp s3://my-bucket-name/file.txt ./

# List EC2 instances
aws ec2 describe-instances

# Start EC2 instance
aws ec2 start-instances --instance-ids i-1234567890abcdef0

# Stop EC2 instance
aws ec2 stop-instances --instance-ids i-1234567890abcdef0

# Describe security groups
aws ec2 describe-security-groups

# List RDS databases
aws rds describe-db-instances

# Create RDS database
aws rds create-db-instance --db-instance-identifier mydb --engine mysql

# List Lambda functions
aws lambda list-functions

# Invoke Lambda function
aws lambda invoke --function-name my-function output.json

# Get caller identity
aws sts get-caller-identity

# List IAM users
aws iam list-users

# Create IAM user
aws iam create-user --user-name newuser

# Get CloudWatch metrics
aws cloudwatch get-metric-statistics --namespace AWS/EC2 \
  --metric-name CPUUtilization --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z --period 3600 \
  --statistics Average

# Create CloudWatch alarm
aws cloudwatch put-metric-alarm --alarm-name cpu-alarm \
  --metric-name CPUUtilization --namespace AWS/EC2 \
  --statistic Average --period 300 --threshold 70 \
  --comparison-operator GreaterThanThreshold
```

### AWS EC2 Management
```bash
# List EC2 instances with details
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PublicIpAddress]' \
  --output table

# Create security group
aws ec2 create-security-group --group-name my-sg --description "My security group"

# Add inbound rule
aws ec2 authorize-security-group-ingress --group-id sg-12345678 \
  --protocol tcp --port 22 --cidr 0.0.0.0/0

# Create key pair
aws ec2 create-key-pair --key-name my-key --query 'KeyMaterial' \
  --output text > my-key.pem
chmod 400 my-key.pem

# Launch EC2 instance
aws ec2 run-instances --image-id ami-0c55b159cbfafe1f0 --count 1 \
  --instance-type t2.micro --key-name my-key \
  --security-groups my-sg

# Describe instance details
aws ec2 describe-instances --instance-ids i-1234567890abcdef0
```

### AWS S3 Operations
```bash
# List S3 buckets
aws s3 ls

# List objects in bucket
aws s3 ls s3://my-bucket/

# Sync local to S3
aws s3 sync ./local-folder s3://my-bucket/

# Sync S3 to local
aws s3 sync s3://my-bucket/ ./local-folder

# Remove object from S3
aws s3 rm s3://my-bucket/file.txt

# Empty bucket (remove all objects)
aws s3 rm s3://my-bucket/ --recursive

# Enable versioning
aws s3api put-bucket-versioning --bucket my-bucket \
  --versioning-configuration Status=Enabled

# Set bucket policy
aws s3api put-bucket-policy --bucket my-bucket \
  --policy file://policy.json

# Get bucket lifecycle configuration
aws s3api get-bucket-lifecycle-configuration --bucket my-bucket
```

### AWS IAM Management
```bash
# List IAM users
aws iam list-users

# Create IAM user
aws iam create-user --user-name newuser

# Attach policy to user
aws iam attach-user-policy --user-name newuser \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Create access key for user
aws iam create-access-key --user-name newuser

# List user access keys
aws iam list-access-keys --user-name newuser

# Delete access key
aws iam delete-access-key --user-name newuser --access-key-id AKIAIOSFODNN7EXAMPLE

# Create IAM role
aws iam create-role --role-name my-role --assume-role-policy-document file://trust-policy.json

# Get current user info
aws iam get-user --user-name newuser
```

### Common Issues & Resolution

**Issue: Access denied error**
```bash
# Solution: Check AWS credentials
aws sts get-caller-identity

# Verify permissions
aws iam list-user-policies --user-name $USER

# Configure correct credentials
aws configure
# or
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="us-east-1"
```

**Issue: Invalid bucket name**
```bash
# Solution: S3 bucket naming rules
# - Must be 3-63 characters
# - Only lowercase letters, numbers, hyphens, periods
# - Must not start with number or hyphen
# - Must not be an IP address

# Valid example
aws s3 mb s3://my-project-bucket-2024
```

**Issue: EC2 instance launch fails**
```bash
# Solution: Check instance limits
aws service-quotas get-service-quota --service-code ec2 \
  --quota-code L-1216C47A

# Check security group configuration
aws ec2 describe-security-groups --group-ids sg-12345678

# Check key pair exists
aws ec2 describe-key-pairs --key-names my-key
```

**Issue: S3 bucket access denied**
```bash
# Solution: Check bucket policy
aws s3api get-bucket-policy --bucket my-bucket

# Check object ACL
aws s3api get-object-acl --bucket my-bucket --key file.txt

# Add public read access (use with caution)
aws s3api put-object-acl --bucket my-bucket --key file.txt --acl public-read
```

### Debugging Commands
```bash
# Enable debug logging
aws s3 ls --debug

# Get detailed error information
aws ec2 describe-instances --debug 2>&1 | grep error

# Query with specific output format
aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId,State.Name]' --output table

# Check CloudTrail logs for API calls
aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=my-resource

# Get service health
aws health describe-events

# Check cost and usage
aws ce get-cost-and-usage --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY --metrics BlendedCost
```

### AWS CloudFormation
```bash
# Create stack
aws cloudformation create-stack --stack-name my-stack \
  --template-body file://template.yaml

# List stacks
aws cloudformation list-stacks

# Describe stack
aws cloudformation describe-stacks --stack-name my-stack

# Update stack
aws cloudformation update-stack --stack-name my-stack \
  --template-body file://template.yaml

# Delete stack
aws cloudformation delete-stack --stack-name my-stack

# Validate template
aws cloudformation validate-template --template-body file://template.yaml
```

### AWS Credentials Management
```bash
# Set profile
export AWS_PROFILE=myprofile

# List configured profiles
cat ~/.aws/config

# Use specific profile in command
aws s3 ls --profile myprofile

# Set temporary credentials
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...

# Clear credentials
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_SESSION_TOKEN
```
