# AWS Backend Implementation Guide - Prompt 1

Complete code examples and setup instructions for building the FastAPI backend with AWS Boto3.

## 1. Project Setup

### Create Virtual Environment
```bash
cd backend
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

### Install Dependencies
```bash
pip install fastapi uvicorn boto3 botocore python-dotenv python-multipart
pip freeze > requirements.txt
```

## 2. Environment Configuration

### Create `.env` file
```env
AWS_ACCESS_KEY_ID=your_access_key_here
AWS_SECRET_ACCESS_KEY=your_secret_key_here
AWS_DEFAULT_REGION=us-east-1
OPENAI_API_KEY=your_openai_key_here
DATABASE_URL=postgresql://user:password@rds-endpoint:5432/dbname
JWT_SECRET=your_jwt_secret_here
```

### Create `.env.example`
```env
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=us-east-1
OPENAI_API_KEY=
DATABASE_URL=
JWT_SECRET=
```

## 3. Main Backend File

### `backend/main.py`
```python
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from dotenv import load_dotenv
import os
from aws_scanner import AWSScanner
from ai_analyzer import AIAnalyzer

load_dotenv()

app = FastAPI(title="AI Cloud Cost Detective", version="1.0.0")

# Enable CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173", "http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize services
aws_scanner = AWSScanner()
ai_analyzer = AIAnalyzer()

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    return {"status": "healthy", "service": "AI Cloud Cost Detective"}

@app.get("/api/regions")
async def get_regions():
    """Get available AWS regions"""
    try:
        regions = aws_scanner.get_available_regions()
        return {"regions": regions}
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))

@app.get("/api/accounts")
async def get_accounts():
    """Get AWS account information"""
    try:
        account_info = aws_scanner.get_account_info()
        return account_info
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))

@app.post("/api/analyze")
async def analyze(request: dict):
    """
    Analyze AWS resources for cost optimization
    Expected payload: { "account_id": "123456789012", "region": "us-east-1" }
    """
    try:
        account_id = request.get("account_id")
        region = request.get("region")
        
        if not account_id or not region:
            raise HTTPException(status_code=400, detail="account_id and region required")
        
        # Scan resources
        resources = await aws_scanner.scan_resources(account_id, region)
        
        # Analyze with AI
        analysis = await ai_analyzer.analyze(resources)
        
        return {
            "account_id": account_id,
            "region": region,
            "resources_scanned": len(resources),
            "analysis": analysis
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000, reload=True)
```

## 4. AWS Scanner Module

### `backend/aws_scanner.py`
```python
import boto3
from botocore.exceptions import ClientError, NoCredentialsError
import os

class AWSScanner:
    def __init__(self):
        self.ec2_client = None
        self.rds_client = None
        self.s3_client = None
        self.lambda_client = None
        self.cloudwatch_client = None
        self.elb_client = None
        self.sts_client = boto3.client('sts')
    
    def _get_client(self, service_name, region):
        """Get AWS service client for a specific region"""
        try:
            return boto3.client(service_name, region_name=region)
        except NoCredentialsError:
            raise Exception("AWS credentials not found. Configure AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY")
    
    def get_account_info(self):
        """Get AWS account ID and user"""
        try:
            identity = self.sts_client.get_caller_identity()
            return {
                "account_id": identity['Account'],
                "user_arn": identity['Arn'],
                "user_id": identity['UserId']
            }
        except NoCredentialsError:
            raise Exception("AWS credentials not configured")
    
    def get_available_regions(self):
        """Get list of available AWS regions"""
        ec2 = self._get_client('ec2', 'us-east-1')
        regions = ec2.describe_regions()['Regions']
        return [r['RegionName'] for r in regions]
    
    async def scan_resources(self, account_id, region):
        """Scan all resources in a region"""
        resources = {
            'ec2_instances': await self._scan_ec2(region),
            'rds_databases': await self._scan_rds(region),
            's3_buckets': await self._scan_s3(),
            'lambda_functions': await self._scan_lambda(region),
            'ebs_volumes': await self._scan_ebs(region),
            'load_balancers': await self._scan_load_balancers(region),
            'cloudwatch_metrics': await self._scan_cloudwatch(region),
        }
        return resources
    
    async def _scan_ec2(self, region):
        """Scan EC2 instances"""
        try:
            ec2 = self._get_client('ec2', region)
            response = ec2.describe_instances()
            instances = []
            
            for reservation in response['Reservations']:
                for instance in reservation['Instances']:
                    instances.append({
                        'id': instance['InstanceId'],
                        'type': instance['InstanceType'],
                        'state': instance['State']['Name'],
                        'launch_time': str(instance['LaunchTime']),
                        'tags': {tag['Key']: tag['Value'] for tag in instance.get('Tags', [])},
                        'region': region
                    })
            
            return instances
        except ClientError as e:
            print(f"Error scanning EC2: {e}")
            return []
    
    async def _scan_rds(self, region):
        """Scan RDS databases"""
        try:
            rds = self._get_client('rds', region)
            response = rds.describe_db_instances()
            databases = []
            
            for db in response['DBInstances']:
                databases.append({
                    'id': db['DBInstanceIdentifier'],
                    'engine': db['Engine'],
                    'instance_class': db['DBInstanceClass'],
                    'storage_gb': db['AllocatedStorage'],
                    'status': db['DBInstanceStatus'],
                    'region': region
                })
            
            return databases
        except ClientError as e:
            print(f"Error scanning RDS: {e}")
            return []
    
    async def _scan_s3(self):
        """Scan S3 buckets"""
        try:
            s3 = self._get_client('s3', 'us-east-1')
            response = s3.list_buckets()
            buckets = []
            
            for bucket in response['Buckets']:
                buckets.append({
                    'name': bucket['Name'],
                    'created': str(bucket['CreationDate'])
                })
            
            return buckets
        except ClientError as e:
            print(f"Error scanning S3: {e}")
            return []
    
    async def _scan_lambda(self, region):
        """Scan Lambda functions"""
        try:
            lambda_client = self._get_client('lambda', region)
            response = lambda_client.list_functions()
            functions = []
            
            for func in response['Functions']:
                functions.append({
                    'name': func['FunctionName'],
                    'runtime': func['Runtime'],
                    'memory_mb': func['MemorySize'],
                    'timeout_sec': func['Timeout'],
                    'region': region
                })
            
            return functions
        except ClientError as e:
            print(f"Error scanning Lambda: {e}")
            return []
    
    async def _scan_ebs(self, region):
        """Scan EBS volumes"""
        try:
            ec2 = self._get_client('ec2', region)
            response = ec2.describe_volumes()
            volumes = []
            
            for volume in response['Volumes']:
                volumes.append({
                    'id': volume['VolumeId'],
                    'size_gb': volume['Size'],
                    'type': volume['VolumeType'],
                    'state': volume['State'],
                    'iops': volume.get('Iops', 'N/A'),
                    'region': region
                })
            
            return volumes
        except ClientError as e:
            print(f"Error scanning EBS: {e}")
            return []
    
    async def _scan_load_balancers(self, region):
        """Scan Load Balancers"""
        try:
            elb = self._get_client('elbv2', region)  # ALB/NLB
            response = elb.describe_load_balancers()
            load_balancers = []
            
            for lb in response['LoadBalancers']:
                load_balancers.append({
                    'name': lb['LoadBalancerName'],
                    'arn': lb['LoadBalancerArn'],
                    'type': lb['Type'],
                    'state': lb['State']['Code'],
                    'region': region
                })
            
            return load_balancers
        except ClientError as e:
            print(f"Error scanning Load Balancers: {e}")
            return []
    
    async def _scan_cloudwatch(self, region):
        """Scan CloudWatch metrics"""
        try:
            cloudwatch = self._get_client('cloudwatch', region)
            response = cloudwatch.list_metrics(MaxItems=50)
            metrics = []
            
            for metric in response['Metrics']:
                metrics.append({
                    'namespace': metric['Namespace'],
                    'name': metric['MetricName'],
                    'dimensions': metric.get('Dimensions', [])
                })
            
            return metrics[:20]  # Limit to first 20
        except ClientError as e:
            print(f"Error scanning CloudWatch: {e}")
            return []
```

## 5. Error Handling Best Practices

```python
from botocore.exceptions import ClientError, NoCredentialsError, PartialCredentialsError

class AWSError(Exception):
    """Custom AWS error"""
    pass

def handle_aws_error(error):
    """Handle AWS errors gracefully"""
    if isinstance(error, NoCredentialsError):
        return {"error": "AWS credentials not found"}
    elif isinstance(error, PartialCredentialsError):
        return {"error": "Incomplete AWS credentials"}
    elif isinstance(error, ClientError):
        error_code = error.response['Error']['Code']
        if error_code == 'UnauthorizedOperation':
            return {"error": "Unauthorized: Check IAM permissions"}
        elif error_code == 'InvalidRegion':
            return {"error": "Invalid AWS region"}
        else:
            return {"error": f"AWS Error: {error_code}"}
    else:
        return {"error": str(error)}
```

## 6. Testing

### `backend/test_scanner.py`
```python
import asyncio
from aws_scanner import AWSScanner

async def test_scanner():
    scanner = AWSScanner()
    
    # Test account info
    account = scanner.get_account_info()
    print(f"Account ID: {account['account_id']}")
    
    # Test regions
    regions = scanner.get_available_regions()
    print(f"Available regions: {len(regions)}")
    
    # Test scanning
    resources = await scanner.scan_resources("123456789012", "us-east-1")
    print(f"EC2 Instances: {len(resources['ec2_instances'])}")
    print(f"RDS Databases: {len(resources['rds_databases'])}")
    print(f"S3 Buckets: {len(resources['s3_buckets'])}")

asyncio.run(test_scanner())
```

## 7. Running the Backend

```bash
# Development
python -m uvicorn main:app --reload

# Production
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
```

## 8. Testing API Endpoints

```bash
# Health check
curl http://localhost:8000/health

# Get regions
curl http://localhost:8000/api/regions

# Get account info
curl http://localhost:8000/api/accounts

# Analyze (requires JWT token)
curl -X POST http://localhost:8000/api/analyze \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -d '{"account_id": "123456789012", "region": "us-east-1"}'
```

## 9. IAM Permissions Required

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeVolumes",
        "rds:DescribeDBInstances",
        "s3:ListAllMyBuckets",
        "lambda:ListFunctions",
        "elasticloadbalancing:DescribeLoadBalancers",
        "cloudwatch:ListMetrics",
        "sts:GetCallerIdentity"
      ],
      "Resource": "*"
    }
  ]
}
```

---

**Ready to implement Prompt 1!** Follow this guide step-by-step, then proceed to Prompt 2 (AI Analysis).
