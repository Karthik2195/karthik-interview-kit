# CloudFormation - AWS Infrastructure as Code

## What is it used for?
CloudFormation is used for:
- **Infrastructure as Code**: Define AWS resources in templates
- **Automation**: Automate infrastructure creation
- **Stack management**: Create, update, delete resource stacks
- **Version control**: Track infrastructure changes
- **Reusability**: Create reusable templates
- **Compliance**: Ensure consistent configurations
- **Multi-environment**: Deploy across environments
- **Cost tracking**: Monitor infrastructure costs

## Installation

```bash
# AWS CLI (includes CloudFormation support)
pip install awscli

# Or install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# SAM CLI (AWS Serverless Application Model)
brew install aws-sam-cli

# Configure credentials
aws configure
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Validate template
aws cloudformation validate-template --template-body file://template.yaml

# Create stack
aws cloudformation create-stack \
  --stack-name my-stack \
  --template-body file://template.yaml

# List stacks
aws cloudformation list-stacks

# Describe stack
aws cloudformation describe-stacks --stack-name my-stack

# Get stack resources
aws cloudformation list-stack-resources --stack-name my-stack

# Update stack
aws cloudformation update-stack \
  --stack-name my-stack \
  --template-body file://template.yaml

# Delete stack
aws cloudformation delete-stack --stack-name my-stack

# Get stack events
aws cloudformation describe-stack-events --stack-name my-stack

# Get stack outputs
aws cloudformation describe-stacks --stack-name my-stack \
  --query 'Stacks[0].Outputs'

# Change set (preview changes)
aws cloudformation create-change-set \
  --stack-name my-stack \
  --change-set-name my-changeset \
  --template-body file://template.yaml
```

### Basic CloudFormation Template
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'EC2 instance template'

Parameters:
  InstanceType:
    Type: String
    Default: t2.micro

Resources:
  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0c55b159cbfafe1f0
      InstanceType: !Ref InstanceType
      Tags:
        - Key: Name
          Value: MyInstance

Outputs:
  InstanceId:
    Value: !Ref MyInstance
    Description: Instance ID
  PublicIP:
    Value: !GetAtt MyInstance.PublicIp
    Description: Public IP
```

### Common Issues & Resolution

**Issue: Template validation fails**
```bash
# Solution: Validate syntax
aws cloudformation validate-template --template-body file://template.yaml

# Check JSON format
python -m json.tool template.json

# Check YAML format
yamllint template.yaml
```

**Issue: Stack creation fails**
```bash
# Solution: Check stack events
aws cloudformation describe-stack-events --stack-name my-stack

# View specific error
aws cloudformation describe-stack-events --stack-name my-stack \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'

# Rollback on failure
aws cloudformation delete-stack --stack-name my-stack
```

**Issue: Parameter not recognized**
```bash
# Solution: Use correct parameter format
aws cloudformation create-stack \
  --stack-name my-stack \
  --template-body file://template.yaml \
  --parameters ParameterKey=Key1,ParameterValue=value1

# List template parameters
aws cloudformation get-template-summary \
  --template-body file://template.yaml
```

### Debugging
```bash
# Enable detailed logging
export AWS_DEBUG=true

# Get template summary
aws cloudformation get-template-summary --template-body file://template.yaml

# View current template
aws cloudformation get-template --stack-name my-stack

# Check stack status
aws cloudformation describe-stacks --stack-name my-stack \
  --query 'Stacks[0].StackStatus'

# List all events
aws cloudformation describe-stack-events --stack-name my-stack \
  --query 'StackEvents[*].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]' \
  --output table
```

### Nested Stacks
```yaml
# Parent stack
Resources:
  NestedStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/bucket/nested-template.yaml
      Parameters:
        Param1: value1
      TimeoutInMinutes: 10
```

### Outputs and Exports
```yaml
Outputs:
  VpcId:
    Value: !Ref MyVpc
    Export:
      Name: MyVpc-Id

# Reference exported value
Resources:
  SubnetInVpc:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !ImportValue MyVpc-Id
```

### Conditions
```yaml
Conditions:
  IsProduction: !Equals [!Ref Environment, prod]
  IsProd: !Condition IsProduction

Resources:
  ProdInstance:
    Type: AWS::EC2::Instance
    Condition: IsProduction
    Properties:
      ImageId: ami-0c55b159cbfafe1f0
      InstanceType: t2.large
```

### Mappings
```yaml
Mappings:
  RegionMap:
    us-east-1:
      AMI: ami-0c55b159cbfafe1f0
    us-west-2:
      AMI: ami-0cbf156b7f3924cd9

Resources:
  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !FindInMap [RegionMap, !Ref 'AWS::Region', AMI]
```

### Stack Policies
```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "Update:*",
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "Update:Delete",
      "Resource": "LogicalResourceId/Database"
    }
  ]
}
```

### SAM (Serverless Application Model)
```bash
# Initialize SAM project
sam init

# Build application
sam build

# Run locally
sam local start-api

# Deploy
sam deploy --guided
```
