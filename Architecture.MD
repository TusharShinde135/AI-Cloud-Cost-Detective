## AI Cloud Cost Detective - Architecture (AWS)

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
          │ · EC2       │     └──────┬───────┘    │ Cost Analysis│
          │ · RDS       │            :            └──────┬───────┘
          │ · S3        │     Live updates             :
          │ · CloudWatch│            ▼                   :
          └──────┬──────┘   ┌───────────────┐            :
                 :          │    REACT      │            :
                 ▼          │  (Progress    │            :
          ┌─────────────┐   │   Tracker)    │            :
          │   AWS       │   └───────────────┘            :
          │  Account    │                                :
          │ (Resources) │                                ▼
          └─────────────┘                        ┌──────────────┐
                                                  │ Amazon RDS   │
                                                  │  PostgreSQL  │
                                                  │              │
                                                  │ · users      │
                                                  │ · analyses   │
                                                  └──────┬───────┘
                                                         :
                                                 Stored results
                                                         ▼
                                                  ┌───────────────┐
                                                  │    REACT      │
                                                  │ (Final Report │
                                                  │  + Suggestions│
                                                  │  + Fixes)     │
                                                  └───────────────┘
```

## Key AWS Services

### Compute
- **EC2** — Virtual machine instances (detect over-provisioning, right-sizing opportunities)
- **Lambda** — Serverless functions (detect unused/inefficient functions)
- **ECS/EKS** — Container services (detect resource waste)

### Storage
- **S3** — Object storage (detect lifecycle policy opportunities, unused buckets)
- **EBS** — Block storage (detect unattached volumes, wrong volume types)
- **EFS** — File storage (detect unused mounts)

### Database
- **RDS** — Managed relational databases (detect over-provisioned instances, unused databases)
- **DynamoDB** — NoSQL (detect provisioned capacity waste)

### Networking
- **VPC** — Virtual network (detect unused security groups, NAT gateway costs)
- **Elastic IP** — Static IPs (detect unassociated IPs)
- **Load Balancer** — ALB/NLB (detect unused load balancers)

### Monitoring & Cost
- **CloudWatch** — Logs and monitoring (detect excessive log retention, unused alarms)
- **AWS Cost Explorer** — Cost analysis integration
- **CloudTrail** — Audit logs (detect excessive retention)

## Detection Categories

| Category | Examples |
|----------|----------|
| **Over-Provisioning** | Large EC2 instances running at low CPU, oversized RDS databases, over-provisioned Lambda memory |
| **Unused Resources** | Unattached EBS volumes, unassociated Elastic IPs, orphaned security groups, empty S3 buckets |
| **Misconfigurations** | Wrong instance families, missing Reserved Instances, no auto-scaling policies, NAT gateway in single AZ |
| **Storage Costs** | S3 with no lifecycle policies, excessive CloudWatch log retention, CloudTrail without expiration |
| **Network Costs** | Data transfer across regions/AZs, expensive NAT gateway usage, missing VPC endpoints |

