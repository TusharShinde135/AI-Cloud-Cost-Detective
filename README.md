# AI Cloud Cost Detective

An AI-powered tool that investigates AWS cloud costs automatically. It scans resources in an AWS account, detects cost issues like over-provisioning and misconfigurations, and provides actionable recommendations and automated fixes.

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React (Vite + TypeScript + Tailwind) |
| Backend | Python (FastAPI) |
| Auth | Custom JWT Auth (bcrypt + PyJWT) |
| Cloud Data | AWS SDK (Boto3) |
| Cloud | AWS |
| AI Analysis | OpenAI API |
| Database | Amazon RDS PostgreSQL |
| Live Updates | FastAPI WebSocket |

## Architecture

```
                               ┌──────────────┐
                               │     USER     │
                               └──────┬───────┘
                                      │
                                      ▼
                            ┌───────────────────┐
                            │  REACT FRONTEND   │
                            └────────┬──────────┘
                                     :
                                     : Login / Signup
                                     ▼
                            ┌───────────────────┐
                            │  PYTHON BACKEND   │
                            │    (FastAPI)      │
                            │                   │
                            │  · Custom JWT Auth│
                            └───┬───────┬───┬───┘
                                :       :   :
                 ┌──────────────┘       :   └──────────────┐
                 :                      :                  :
                 ▼                      ▼                  ▼
          ┌─────────────┐     ┌──────────────┐    ┌──────────────┐
          │  AWS SDK    │     │   FASTAPI    │    │   OPENAI     │
          │  (Boto3)    │     │  WEBSOCKET   │    │    API       │
          │             │     │  (Progress)  │    │              │
          │ describe &  │     └──────┬───────┘    │ Cost Analysis│
          │ list APIs   │            :            └──────┬───────┘
          └──────┬──────┘            : Live updates      :
                 :                   ▼                   :
                 ▼                ┌───────────────┐      :
          ┌─────────────┐   ┌────▶│    REACT      │      :
          │   AWS       │   │     │  (Progress    │      :
          │   Account   │   │     │   Tracker)    │      :
          │ (Resources) │   │     └───────────────┘      :
          └─────────────┘   :                            :
                            :                            ▼
                            :                    ┌──────────────┐
                            :                    │ Amazon RDS   │
                            :                    │  PostgreSQL  │
                            :                    │              │
                            :                    │ · users      │
                            :                    │ · analyses   │
                            :                    └──────┬───────┘
                            :                           :
                            :                  Stored results
                            :                           ▼
                            └──────────────────▶┌───────────────┐
                                                │    REACT      │
                                                │ (Final Report │
                                                │  + Suggestions│
                                                │  + Fixes)     │
                                                └───────────────┘
```

## Request Flow

```
①  User ─·─·─► React ─·─·─► FastAPI Auth ─·─·─► JWT (Amazon RDS PostgreSQL)

②  User selects AWS Account/Region ─·─·─► Python Backend

③  Python ─·─·─► AWS SDK (Boto3) ─·─·─► Fetches all resources in account

④  Python ─·─·─► FastAPI WebSocket ─·─·─► React (live progress)

⑤  Python ─·─·─► OpenAI API ─·─·─► Cost analysis

⑥  Python ─·─·─► Amazon RDS PostgreSQL ─·─·─► Stores analysis history

⑦  React ◄·─·─·─ Final report with suggestions & fixes
```

## What It Detects

- **Over-provisioned resources** — EC2 instances, RDS databases, or ECS tasks sized larger than needed
- **Unused resources** — Unattached EBS volumes, unassociated Elastic IPs, idle load balancers, orphaned security groups
- **Misconfigurations** — Wrong instance types, missing auto-scaling, no Reserved Instances, excessive data transfer
- **Storage & logging costs** — Excessive CloudWatch logs, no S3 lifecycle policies, unoptimized CloudTrail retention
- **Network costs** — Data transfer across regions, NAT gateway usage, VPC endpoint opportunities

## Prerequisites

- AWS Account with appropriate IAM permissions (EC2, RDS, S3, CloudWatch, Cost Explorer)
- AWS credentials configured locally (via `~/.aws/credentials` or environment variables)
- An Amazon RDS PostgreSQL instance
- An OpenAI API key
- Python 3.10+
- Node.js 18+

## How to Run

### Backend

```bash
cd backend
pip install -r requirements.txt
cp .env.example .env   # fill in your credentials (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, OPENAI_API_KEY, etc.)
uvicorn main:app --reload
```

### Frontend

```bash
cd frontend
npm install
npm run dev
```

## How It Works

1. User signs up / logs in via custom JWT auth (credentials stored in Amazon RDS PostgreSQL)
2. Selects an AWS account/region to analyze
3. Python backend fetches all resources using AWS SDK (Boto3)
4. Live progress is streamed to the UI via FastAPI WebSocket
5. Resource data is sent to OpenAI API for cost analysis
6. Analysis results are stored in Amazon RDS PostgreSQL
7. Final report with cost breakdown, suggestions, and fix commands is displayed
