# Azure → AWS Migration Guide

This document outlines the changes required to convert the AI Cloud Cost Detective from Azure to AWS.

## Service Mapping

| Azure Service | AWS Service | Purpose |
|---|---|---|
| Azure CLI | AWS CLI / Boto3 | Infrastructure queries |
| Azure Resource Groups | AWS Account/Region | Resource organization |
| Azure VMs | EC2 Instances | Compute resources |
| Azure App Services | Lambda / App Runner | Serverless/app hosting |
| Azure SQL Database | Amazon RDS | Managed database |
| Azure Blob Storage | Amazon S3 | Object storage |
| Azure PostgreSQL (Managed) | Amazon RDS PostgreSQL | Managed PostgreSQL |
| Azure Monitor | CloudWatch | Monitoring & logs |
| Azure DevOps | AWS CodePipeline | CI/CD |

## Backend Changes Required

### 1. Dependencies (`backend/requirements.txt`)

**Remove:**
```
azure-cli
azure-identity
azure-common
```

**Add:**
```
boto3>=1.28.0
botocore>=1.31.0
```

### 2. Environment Variables (`.env`)

**Azure:**
```
AZURE_SUBSCRIPTION_ID=...
AZURE_RESOURCE_GROUP=...
AZURE_CLIENT_ID=...
AZURE_CLIENT_SECRET=...
AZURE_TENANT_ID=...
DATABASE_URL=postgresql://user:pass@azure-postgres:5432/dbname
```

**AWS:**
```
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_DEFAULT_REGION=us-east-1
DATABASE_URL=postgresql://user:pass@rds-endpoint:5432/dbname
OPENAI_API_KEY=...
```

### 3. Resource Discovery (`backend/services/cloud_service.py`)

**Change from Azure CLI to Boto3:**

```python
# OLD (Azure)
import subprocess
result = subprocess.run(['az', 'resource', 'list', '--rg', rg_name], capture_output=True)

# NEW (AWS)
import boto3
ec2 = boto3.client('ec2')
instances = ec2.describe_instances()
```

### 4. Key Boto3 Calls to Implement

```python
import boto3

# EC2 instances
ec2 = boto3.client('ec2', region_name='us-east-1')
instances = ec2.describe_instances()

# RDS databases
rds = boto3.client('rds', region_name='us-east-1')
databases = rds.describe_db_instances()

# S3 buckets
s3 = boto3.client('s3')
buckets = s3.list_buckets()

# Lambda functions
lambda_client = boto3.client('lambda', region_name='us-east-1')
functions = lambda_client.list_functions()

# EBS volumes
volumes = ec2.describe_volumes()

# CloudWatch metrics
cloudwatch = boto3.client('cloudwatch', region_name='us-east-1')
metrics = cloudwatch.list_metrics()

# Cost Explorer (for cost analysis)
ce = boto3.client('ce', region_name='us-east-1')
costs = ce.get_cost_and_usage(TimePeriod={...})
```

### 5. Cost Analysis Updates

**Azure Cost:** Query Azure Cost Management API
**AWS Cost:** Use Cost Explorer API or AWS Pricing API

```python
# AWS Cost Explorer query
ce = boto3.client('ce')
response = ce.get_cost_and_usage(
    TimePeriod={'Start': start_date, 'End': end_date},
    Granularity='DAILY',
    Filter={'Dimensions': {'Key': 'SERVICE', 'Values': ['Amazon Elastic Compute Cloud - Compute']}},
    Metrics=['BlendedCost']
)
```

## Database Schema Updates

### PostgreSQL Tables (No major changes needed)

```sql
-- Existing tables remain mostly the same:
-- users, analyses, recommendations, fixes

-- Add AWS-specific fields:
ALTER TABLE analyses ADD COLUMN aws_account_id VARCHAR(12);
ALTER TABLE analyses ADD COLUMN aws_region VARCHAR(20);
ALTER TABLE resources ADD COLUMN aws_resource_id VARCHAR(255);
ALTER TABLE resources ADD COLUMN aws_service VARCHAR(50);
```

## Frontend Changes

### Minimal Changes Required

1. Update labels/icons from "Azure Resource Group" to "AWS Account"
2. Change region selector to AWS regions
3. Update documentation links to AWS services
4. Update resource type icons to AWS service icons

### Example Components Update:

```javascript
// OLD: Azure region list
const regions = ['eastus', 'westus', 'centralus'];

// NEW: AWS region list
const regions = ['us-east-1', 'us-west-2', 'eu-west-1', 'ap-southeast-1'];
```

## Testing Checklist

- [ ] Authenticate with AWS credentials
- [ ] List EC2 instances from a region
- [ ] List RDS databases
- [ ] List S3 buckets
- [ ] Query CloudWatch metrics
- [ ] Store analysis results in RDS PostgreSQL
- [ ] Generate cost recommendations via OpenAI
- [ ] Stream WebSocket progress updates
- [ ] Verify JWT authentication still works
- [ ] Test with multiple AWS regions
- [ ] Validate security (IAM permissions, credential handling)

## Deployment Considerations

### AWS Permissions Required (IAM Policy)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "rds:Describe*",
        "s3:List*",
        "lambda:List*",
        "cloudwatch:List*",
        "ce:GetCostAndUsage",
        "pricing:GetProducts"
      ],
      "Resource": "*"
    }
  ]
}
```

### Deployment Options

1. **EC2 + RDS** — Deploy backend on EC2, use managed RDS
2. **Lambda + RDS** — Deploy backend as Lambda functions (serverless)
3. **ECS + RDS** — Deploy backend on ECS containers
4. **CloudFront + S3** — Host React frontend on S3 + CloudFront

## Security Best Practices

1. **Credentials:**
   - Use IAM roles instead of access keys (when on EC2)
   - Store sensitive config in AWS Secrets Manager
   - Never commit `.env` files with real credentials

2. **Database:**
   - Enable RDS encryption at rest
   - Use VPC security groups to restrict access
   - Enable automated backups

3. **API Keys:**
   - Store OpenAI API key in AWS Secrets Manager
   - Rotate keys regularly
   - Use separate keys for dev/prod

## Estimated Effort

- **Backend refactoring:** 4-6 hours
- **Testing:** 2-3 hours
- **Documentation:** 1-2 hours
- **Total:** ~8-11 hours

