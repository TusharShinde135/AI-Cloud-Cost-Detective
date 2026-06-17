# Prompt 3: Amazon RDS PostgreSQL + WebSocket Progress Tracking

Build on top of the existing FastAPI backend. Add Amazon RDS PostgreSQL for storing users and analysis history, and FastAPI WebSocket for live progress updates.

## What to build

### Database (Amazon RDS PostgreSQL)

- Connect to an Amazon RDS PostgreSQL instance using `asyncpg` or `psycopg2`.
- Store the database connection string in `.env` (`DATABASE_URL` — use RDS endpoint).
- Create tables on startup:
  - `users` — id, email, password_hash, created_at
  - `analyses` — id, user_id, aws_account_id, aws_region, resources_scanned (int), issues_found (int), estimated_savings (decimal), analysis_result (jsonb), status, created_at
  - `resources` — id, analysis_id, resource_type (ec2/rds/s3/lambda/ebs/elb/etc), resource_id, resource_name, resource_config (jsonb), cost_estimate (decimal), created_at
  - `recommendations` — id, analysis_id, severity (high/medium/low), title, description, estimated_savings (decimal), fix_command (text), created_at
- After AI analysis completes, store the full result in the `analyses` table along with individual resources and recommendations.
- Add a `GET /api/history` endpoint that returns past analyses for the authenticated user (paginated).

### WebSocket Progress

- Add a WebSocket endpoint `ws://localhost:8000/ws/progress/{analysis_id}`.
- During the `POST /api/analyze` flow, push progress messages through the WebSocket at each stage:
  - `"Authenticating AWS credentials..."`
  - `"Fetching EC2 instances..."`
  - `"Fetching RDS databases..."`
  - `"Fetching S3 buckets..."`
  - `"Fetching Lambda functions..."`
  - `"Fetching EBS volumes..."`
  - `"Fetching Load Balancers..."`
  - `"Fetching CloudWatch metrics..."`
  - `"Scanned <total> resources in <region>"`
  - `"Analyzing costs with AI..."`
  - `"Storing results in database..."`
  - `"Analysis complete — <issues_found> issues found, estimated savings: $<amount>/month"`
- The frontend will connect to this WebSocket to show live progress.

### Update .env.example

Add these environment variables:
```
DATABASE_URL=postgresql://user:password@rds-endpoint.region.rds.amazonaws.com:5432/dbname
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_DEFAULT_REGION=us-east-1
```

## Project structure update

```
backend/
├── main.py          (updated — history endpoint, WebSocket, DB init)
├── aws_scanner.py   (no change)
├── ai_analyzer.py   (no change)
├── db.py            (new — RDS connection, table creation, queries)
├── models.py        (new — Pydantic models for Analysis, Resource, Recommendation)
├── requirements.txt (updated — add asyncpg/psycopg2, websockets, sqlalchemy)
├── .env.example     (updated)
```

## RDS Setup Checklist

- [ ] Create RDS PostgreSQL instance in AWS Console
- [ ] Note the endpoint URL (e.g., `mydb.123abc.us-east-1.rds.amazonaws.com`)
- [ ] Create master username and password
- [ ] Allow inbound traffic from your backend security group (port 5432)
- [ ] Add endpoint to DATABASE_URL in .env

Refer to `Architecture.MD`, `RequestFlow.MD`, and `AWS_MIGRATION_GUIDE.md`. This covers steps ④ and ⑥ of the request flow.
