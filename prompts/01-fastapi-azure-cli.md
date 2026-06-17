# Prompt 1: FastAPI Backend + AWS Boto3

Create a Python FastAPI backend in a `backend/` folder for the AI Cloud Cost Detective project.

## What to build

- A FastAPI server with a `POST /api/analyze` endpoint that accepts `{ "account_id": "<id>", "region": "<region>" }`.
- A `GET /api/accounts` endpoint that returns the AWS account information.
- A `GET /api/regions` endpoint that returns available AWS regions.
- Use Boto3 (AWS SDK) to interact with AWS services:
  - `ec2.describe_instances()` to fetch EC2 instances
  - `rds.describe_db_instances()` to fetch RDS databases
  - `s3.list_buckets()` to fetch S3 buckets
  - `lambda_client.list_functions()` to fetch Lambda functions
  - `ec2.describe_volumes()` to fetch EBS volumes
  - `elb.describe_load_balancers()` to fetch Load Balancers
  - `cloudwatch.list_metrics()` to fetch CloudWatch metrics
- Parse the AWS API JSON responses and return a structured response with resource type, name, region, size, tags, and metadata.
- Add error handling for missing AWS credentials, invalid regions, and AWS API errors.
- Enable CORS for `http://localhost:5173`.
- Include a `requirements.txt` with `fastapi`, `uvicorn`, `boto3`, `botocore`.

## Project structure

```
backend/
├── main.py
├── aws_scanner.py
├── requirements.txt
```

## Environment Variables

```
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_DEFAULT_REGION=us-east-1
```

Refer to `Architecture.MD`, `RequestFlow.MD`, and `AWS_MIGRATION_GUIDE.md`. This covers step ③ of the request flow.
