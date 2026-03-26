---
title: Cloud Infrastructure Designer
category: devops
version: 1.0.0
works_with: [claude, openai, ollama, gemini]
---

## Purpose
Design a complete, cost-aware cloud infrastructure architecture from a plain-English description of your service — including Terraform snippets, security configuration, and scaling strategy.

## When to Use
- Greenfield project where you need to choose and design your infrastructure
- Existing infra is poorly architected and you need a redesign plan
- You need to explain architecture to stakeholders or write an RFC

## The Prompt

```
You are a cloud architect with expertise in {{CLOUD_PROVIDER}} and infrastructure-as-code. Design a production-ready cloud infrastructure for the following service.

**Cloud Provider:** {{CLOUD_PROVIDER}}
(Options: AWS, GCP, Azure, multi-cloud)

**Service description:**
{{SERVICE_DESCRIPTION}}
(Be specific: type of workload, data sensitivity, user base size, geographic requirements)

**Traffic profile:**
{{TRAFFIC_PROFILE}}
(Examples: "10k req/day with spikes to 100k", "steady 500 req/sec", "batch job runs nightly")

**Data requirements:**
{{DATA_REQUIREMENTS}}
(Examples: "PostgreSQL 200GB, grows 5GB/month, must be encrypted at rest", "S3 storage, no PII")

**Budget constraint:**
{{BUDGET}}
(Examples: "$500/month hard limit", "minimize cost — early stage startup", "cost not primary concern")

**Compliance requirements:**
{{COMPLIANCE}}
(Examples: "SOC 2 Type II required", "HIPAA", "GDPR — data must stay in EU", "none")

**Team size / ops capacity:**
{{OPS_CAPACITY}}
(Examples: "2 devs, no dedicated ops — keep it managed", "have a platform team", "solo founder")

Please provide:
1. **Architecture overview** — ASCII diagram of the full infrastructure
2. **Component breakdown** — Each service selected, why it was chosen over alternatives, and approximate cost
3. **Terraform snippets** — Key resource definitions for the most critical components
4. **Security posture** — IAM roles, network security groups, encryption, secrets management
5. **Scaling strategy** — How this scales from day 1 to 10x traffic
6. **Failure modes** — What breaks first under load, and how to handle it
7. **Cost estimate** — Monthly breakdown by service at current and 10x traffic
```

## Example Input

| Variable | Value |
|---|---|
| `{{CLOUD_PROVIDER}}` | `AWS` |
| `{{SERVICE_DESCRIPTION}}` | `REST API backend for a B2B SaaS app — handles user auth, CRUD operations, and file uploads` |
| `{{TRAFFIC_PROFILE}}` | `500 req/day currently, expect 50k/day in 12 months` |
| `{{DATA_REQUIREMENTS}}` | `PostgreSQL, ~10GB, must be encrypted, needs daily backups` |
| `{{BUDGET}}` | `$200/month now, acceptable to grow with revenue` |
| `{{COMPLIANCE}}` | `No specific requirements, but handle user PII carefully` |
| `{{OPS_CAPACITY}}` | `2 developers, no dedicated ops team` |

## Example Output

```
**Architecture Overview**
```
Internet
    │
    ▼
[CloudFront CDN]
    │
    ▼
[Application Load Balancer]
    │
    ▼
[ECS Fargate Tasks] ──── [ElastiCache Redis (sessions)]
    │
    ▼
[RDS PostgreSQL (Multi-AZ)] ──── [S3 (file uploads)]
    │
    ▼
[Secrets Manager] [CloudWatch Logs]
```

**Component Breakdown**
- ECS Fargate: Serverless containers — no EC2 to manage. Right-sized at 0.5 vCPU/1GB RAM. ~$30/mo.
  (Chose over EC2: no instance management; over Lambda: persistent connections to DB)
- RDS PostgreSQL t3.micro: Managed DB with automated backups, Multi-AZ for HA. ~$25/mo.
- ALB: Handles SSL termination, health checks. ~$18/mo.
- CloudFront: CDN for static assets + API caching. ~$5/mo.
- S3: File uploads. Pay per use. ~$2/mo initially.

**Terraform Snippet**
resource "aws_ecs_service" "api" {
  name            = "api-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.api.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.api.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.api.arn
    container_name   = "api"
    container_port   = 3000
  }
}

**Security Posture**
- ECS tasks run in private subnets — not directly accessible from internet
- ALB in public subnet, only port 443 open
- RDS accessible only from ECS security group (no public endpoint)
- Secrets (DB password, JWT secret) in AWS Secrets Manager — injected at runtime
- S3 bucket: block all public access, use pre-signed URLs for uploads

**Scaling Strategy**
- Day 1: 2 Fargate tasks, t3.micro RDS
- 10x traffic: Add ECS auto-scaling policy (target 70% CPU), upgrade to t3.small RDS
- 100x traffic: Read replicas for RDS, CloudFront for API caching, connection pooling via PgBouncer

**Failure Modes**
- DB connection exhaustion (most likely first): Add PgBouncer before RDS reaches 10k req/day
- Single AZ failure: Fargate auto-replaces tasks; RDS Multi-AZ auto-failover in ~60s

**Monthly Cost Estimate**
Current: ~$80/month | At 50k req/day: ~$150/month | At 500k req/day: ~$600/month
```

## Tips
- Be specific in `{{SERVICE_DESCRIPTION}}` — "web app" tells the AI nothing. "REST API with WebSocket support serving 10k users" triggers very different architecture decisions
- Always mention `{{OPS_CAPACITY}}` — a solo founder should not be managing Kubernetes
- Ask for Terraform output — it makes the design implementable immediately, not just theoretical
