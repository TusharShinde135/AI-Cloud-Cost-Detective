# Prompt 2: OpenAI API Integration for Cost Analysis

Build on top of the existing FastAPI backend. Add AI-powered cost analysis using the OpenAI API.

## What to build

- Create an `ai_analyzer.py` module in `backend/` that:
  - Takes the list of AWS resources (from `aws_scanner.py`) as input.
  - Builds a comprehensive prompt asking the AI to analyze resources for:
    - Over-provisioning (e.g., large EC2 instances with low CPU, over-allocated RDS, over-provisioned Lambda)
    - Unused/idle resources (e.g., unattached EBS volumes, unassociated Elastic IPs, idle load balancers)
    - Misconfigurations (e.g., wrong instance families, missing auto-scaling, no Reserved Instances)
    - Storage & logging costs (e.g., excessive CloudWatch logs, S3 without lifecycle policies)
    - Network costs (e.g., data transfer across regions, NAT gateway usage)
  - Calls the OpenAI chat completions API (`gpt-4o` or `gpt-4-turbo`) and returns structured analysis.
- The AI response should include:
  - Executive summary of findings
  - List of issues found (with severity: high/medium/low and estimated monthly savings)
  - Actionable fix commands (AWS CLI commands or Terraform the user can run)
  - Quick wins (top 3 actions for immediate cost reduction)
- Update `POST /api/analyze` to call `aws_scanner` first, then pass results to `ai_analyzer`, and return the final analysis.
- Store the OpenAI API key in environment variables. Add a `.env.example` file.
- Update `requirements.txt` — add `openai`, `python-dotenv`.

## Project structure update

```
backend/
├── main.py          (updated)
├── aws_scanner.py   (no change)
├── ai_analyzer.py   (new)
├── requirements.txt (updated)
├── .env.example     (new — OPENAI_API_KEY)
```

## Cost Analysis Prompt Tips

Include resource metrics in the prompt:
- EC2: instance type, CPU utilization (from CloudWatch), uptime, region
- RDS: instance class, storage size, read/write IOPS, database engine
- S3: bucket size, storage class, lifecycle policies
- Lambda: function memory, invocation count, duration
- EBS: volume type, size, IOPS, attachment status

Refer to `Architecture.MD`, `RequestFlow.MD`, and `AWS_MIGRATION_GUIDE.md`. This covers step ⑤ of the request flow.
