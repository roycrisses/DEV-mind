---
title: CI/CD Pipeline Generator
category: devops
version: 1.0.0
works_with: [claude, openai, ollama, gemini]
---

## Purpose
Generate a complete, environment-aware CI/CD pipeline config for your chosen platform — with test gates, security scanning, and deployment strategies built in.

## When to Use
- Setting up a new project's pipeline from scratch
- Migrating between CI platforms (e.g., Jenkins → GitHub Actions)
- Existing pipeline is flaky, slow, or missing quality gates

## The Prompt

```
You are a DevOps engineer specializing in CI/CD pipelines. Generate a complete pipeline configuration for the following project.

**CI/CD Platform:** {{PLATFORM}}
(Options: GitHub Actions, GitLab CI, CircleCI, Bitbucket Pipelines, Jenkins (Declarative))

**Project type:** {{PROJECT_TYPE}}
(Examples: "Node.js API with PostgreSQL", "Python ML service", "React frontend", "Go microservice")

**Deployment target:** {{DEPLOYMENT_TARGET}}
(Examples: "AWS ECS via ECR", "GCP Cloud Run", "Kubernetes cluster", "Vercel", "S3 + CloudFront")

**Environments:**
{{ENVIRONMENTS}}
(Examples: "staging (pushes to main), production (manual approval on tags)", "dev, staging, prod")

**Test suite:**
{{TEST_SUITE}}
(Examples: "Jest unit + integration tests", "pytest with coverage", "none yet — just add placeholder")

**Security requirements:**
{{SECURITY_REQUIREMENTS}}
(Examples: "SAST scan with Semgrep", "container vulnerability scan with Trivy", "dependency audit", "none")

**Additional requirements:**
{{ADDITIONAL_REQUIREMENTS}}
(Examples: "Slack notification on failure", "cache node_modules between runs", "build Docker image and push to ECR")

Generate:
1. **Pipeline config file** — Complete, ready-to-commit configuration
2. **Required secrets** — Exact list of secrets to add to the CI platform
3. **Pipeline diagram** — ASCII diagram showing the job flow
4. **Estimated pipeline time** — Approximate run time for each stage
5. **Optimization notes** — How to speed up the pipeline if it becomes slow
```

## Example Input

| Variable | Value |
|---|---|
| `{{PLATFORM}}` | `GitHub Actions` |
| `{{PROJECT_TYPE}}` | `Node.js API with PostgreSQL` |
| `{{DEPLOYMENT_TARGET}}` | `AWS ECS via ECR` |
| `{{ENVIRONMENTS}}` | `staging (push to main), production (manual approval on git tag)` |
| `{{TEST_SUITE}}` | `Jest unit + integration tests` |
| `{{SECURITY_REQUIREMENTS}}` | `Trivy container scan, npm audit` |
| `{{ADDITIONAL_REQUIREMENTS}}` | `Slack notification on production deploy failure` |

## Example Output

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

env:
  ECR_REPOSITORY: my-api
  ECS_SERVICE: my-api-service
  ECS_CLUSTER: my-cluster

jobs:
  test:
    name: Test & Lint
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run lint
      - run: npm test -- --coverage
        env:
          DATABASE_URL: postgres://postgres:test@localhost:5432/testdb
      - run: npm audit --audit-level=high

  security-scan:
    name: Security Scan
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build image for scanning
        run: docker build -t scan-target:${{ github.sha }} .
      - name: Trivy vulnerability scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: scan-target:${{ github.sha }}
          severity: CRITICAL,HIGH
          exit-code: '1'

  deploy-staging:
    name: Deploy to Staging
    needs: security-scan
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      - name: Build, tag, push to ECR
        run: |
          aws ecr get-login-password | docker login --username AWS --password-stdin ${{ secrets.ECR_REGISTRY }}
          docker build -t $ECR_REPOSITORY:${{ github.sha }} .
          docker push ${{ secrets.ECR_REGISTRY }}/$ECR_REPOSITORY:${{ github.sha }}
      - name: Deploy to ECS
        run: |
          aws ecs update-service --cluster $ECS_CLUSTER \
            --service $ECS_SERVICE-staging \
            --force-new-deployment

  deploy-production:
    name: Deploy to Production
    needs: deploy-staging
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://api.example.com
    steps:
      - name: Deploy to ECS Production
        run: |
          aws ecs update-service --cluster $ECS_CLUSTER \
            --service $ECS_SERVICE-production \
            --force-new-deployment
      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1.25.0
        with:
          payload: '{"text":"🚨 Production deploy FAILED for ${{ github.ref }}"}'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

**Required Secrets**
- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` — IAM user with ECS + ECR permissions
- `ECR_REGISTRY` — Your ECR registry URL
- `SLACK_WEBHOOK_URL` — Slack Incoming Webhook URL

**Pipeline Diagram**
```
PR/Push → [test] → [security-scan] → [deploy-staging] → (manual gate) → [deploy-production]
```

**Estimated Times:** test: 3min | security-scan: 4min | deploy: 2min | Total: ~9min

**Optimization Notes**
- Cache npm and Docker layers with `actions/cache@v4` to cut test time by ~40%
- Run lint and test as parallel matrix jobs if test suite grows beyond 5 minutes
```

## Tips
- Always specify your `{{DEPLOYMENT_TARGET}}` precisely — "AWS" alone is too vague; ECS, Lambda, and App Runner all need different pipeline steps
- Add `environment: production` with required reviewers in GitHub Settings to enforce manual approval gates
- For monorepos, add path filters (`paths:`) to only trigger the pipeline when relevant files change
