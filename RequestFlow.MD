## AI Cloud Cost Detective - Request Flow (AWS)

```
①  User ─·─·─► React ─·─·─► FastAPI Auth ─·─·─► JWT (Amazon RDS PostgreSQL)

②  User selects AWS Account/Region ─·─·─► Python Backend

③  Python ─·─·─► AWS SDK (Boto3) ─·─·─► Fetches all resources in account
                                       · EC2 instances
                                       · RDS databases
                                       · S3 buckets
                                       · Lambda functions
                                       · EBS volumes
                                       · Load Balancers
                                       · CloudWatch metrics

④  Python ─·─·─► FastAPI WebSocket ─·─·─► React (live progress updates)

⑤  Python ─·─·─► OpenAI API ─·─·─► Cost analysis & recommendations

⑥  Python ─·─·─► Amazon RDS PostgreSQL ─·─·─► Stores analysis history

⑦  React ◄·─·─·─ Final report with:
                  · Cost breakdown by service
                  · Right-sizing recommendations
                  · Unused resource cleanup suggestions
                  · Automated fix commands (AWS CLI/Terraform)
```

## Detailed Flow

### Step 1: Authentication
- User signs up/logs in via React frontend
- FastAPI validates credentials against Amazon RDS PostgreSQL
- JWT token issued for session management

### Step 2: Account & Region Selection
- User specifies AWS account ID and region(s) to analyze
- Backend validates AWS credentials from environment

### Step 3: Resource Discovery
- Python backend uses Boto3 (AWS SDK) to call:
  - `ec2.describe_instances()` — All EC2 instances
  - `rds.describe_db_instances()` — All RDS databases
  - `s3.list_buckets()` — All S3 buckets
  - `lambda.list_functions()` — All Lambda functions
  - `ec2.describe_volumes()` — All EBS volumes
  - `elb.describe_load_balancers()` — All load balancers
  - `cloudwatch.describe_alarms()` — All alarms
  - And more...
- WebSocket emits progress: "Found 245 resources..."

### Step 4: Data Enrichment
- Retrieve additional metrics:
  - CloudWatch CPU/memory utilization (last 7/30 days)
  - Cost Explorer data for each resource
  - Security group associations
  - Auto-scaling group configurations

### Step 5: Analysis via OpenAI
- Send resource inventory to OpenAI API
- Prompt: "Analyze these AWS resources for cost optimization opportunities"
- Receive recommendations like:
  - Right-sizing suggestions
  - Reserved Instance opportunities
  - Unused resource cleanup
  - Architectural improvements

### Step 6: Results Storage
- Store analysis in PostgreSQL tables:
  - `analyses` — Analysis metadata
  - `resources` — Detected resources
  - `recommendations` — Cost optimization suggestions
  - `fixes` — Automated commands to implement fixes

### Step 7: Report Generation
- Frontend displays:
  - Cost breakdown by service (EC2, RDS, S3, etc.)
  - Top cost drivers
  - Risk categories (security, compliance, performance)
  - Recommended fixes with estimated savings
  - AWS CLI/Terraform commands for automation

