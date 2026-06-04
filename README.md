# 🚀 Express Application - TurboVets DevOps Assessment

[![Build and Deploy](https://github.com/chrisjamaica91/express-app/actions/workflows/deploy.yml/badge.svg)](https://github.com/chrisjamaica91/express-app/actions/workflows/deploy.yml)

Production-ready Express.js + TypeScript application with multi-environment CI/CD pipeline (dev/staging/production).

**Related Repository:** [express-app-iac](https://github.com/chrisjamaica91/express-app-iac) - Infrastructure as Code (CDKTF)

---

## 📋 Overview

### Why Two Repositories?

I used **two separate repos** following industry best practices:
- ✅ **Separation of concerns** - App code vs infrastructure code
- ✅ **Different deployment cadences** - Apps change frequently, infrastructure less often
- ✅ **Better access control** - Different teams, different permissions
- ✅ **Real-world pattern** - How most organizations structure production systems

---

## 🏗️ Key Architectural Decisions

### 1. OIDC Instead of IAM Keys 🔐

**Assessment Requirement:** Store `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` in GitHub Secrets

**My Approach:** GitHub OIDC federation with IAM roles

**Why:**
- ✅ **No long-lived credentials** - Tokens expire automatically (1 hour)
- ✅ **Zero secret management** - No keys to store, rotate, or leak
- ✅ **Better security** - CloudTrail audit logs, automatic rotation
- ✅ **Industry best practice** - Recommended by AWS and GitHub

**Implementation:**
```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-deployment-role
```

### 2. Multi-Stage Docker Builds 📦

**Results:** ~50MB production image (vs 300MB+ with dev dependencies)

```dockerfile
FROM node:20-alpine AS builder
# Build stage with TypeScript compilation

FROM node:20-alpine
# Production stage - only compiled code + runtime deps
USER node  # Non-root for security
```

**Benefits:**
- ✅ 80% smaller images → 5x faster deployments
- ✅ No dev dependencies, TypeScript compiler, or tests in production
- ✅ Non-root user (runs as `node` UID 1000)
- ✅ Alpine Linux base (minimal attack surface)

### 3. Comprehensive Security Scanning 🔍

**Four layers of scanning:**
- **Gitleaks** - Secret detection in code/history
- **Hadolint** - Dockerfile best practices
- **Trivy (filesystem)** - Source code CVE scanning
- **Trivy (image)** - Container image vulnerabilities

All results uploaded to GitHub Security tab (SARIF format).

---

## 💻 Local Development

### Quick Start

```bash
git clone https://github.com/chrisjamaica91/express-app.git
cd express-app
docker-compose up
curl http://localhost:3000/health
# Expected: {"status":"ok","timestamp":"2026-06-04T..."}
```

## 🐳 Docker Configuration

### Dockerfile Optimizations

- **Multi-stage build** - Separate build/runtime environments
- **Alpine Linux** - 5MB base image (vs 200MB+ Debian)
- **Layer caching** - Strategic COPY ordering
- **Security hardening** - Non-root user, minimal packages
- **.dockerignore** - 80% smaller build context

### Docker Compose

```yaml
services:
  app:
    build: .
    ports:
      - "3000:3000"
    healthcheck:
      test: ["CMD", "wget", "--spider", "http://localhost:3000/health"]
      interval: 30s
```

---

## 🔄 CI/CD Pipeline

### Workflow: `.github/workflows/deploy.yml`

**Triggers:** Push to `dev`, `staging`, `main` | Pull requests | Manual dispatch

### Pipeline Stages

**1. Environment Detection**
- `dev` branch → dev environment
- `staging` branch → staging environment
- `main` branch → prod environment
- PRs: Scan only (no deployment)

**2. Security Scanning** (All PRs/Pushes)
- Gitleaks, Hadolint, Trivy scans
- Must pass before proceeding

**3. Build & Push** (Pushes only)
- Build with GitHub Actions cache
- Tag: `dev`, `dev-<sha>`, `latest`
- Push to ECR
- Scan image with Trivy

**4. Deploy to ECS** (Pushes only)
- Force ECS service update
- Wait for service stability
- Verify health endpoint

**5. AI Analysis** (Optional)
- Analyze CloudWatch logs
- Provide deployment insights

### Environment Behavior

| Environment | Branch   | Approvals | Auto-Deploy | Image Tag |
|-------------|----------|-----------|-------------|-----------|
| Dev         | `dev`    | 1         | ✅ Yes      | `dev`     |
| Staging     | `staging`| 1         | ✅ Yes      | `staging` |
| Production  | `main`   | 2         | ✅ Yes      | `prod`    |

---

## 🌍 Multi-Environment Strategy

### Promotion Workflow

```
Feature → PR to dev → Merge → Deploy to Dev
    ↓
  dev → PR to staging (1 approval) → Deploy to Staging
    ↓
staging → PR to main (2 approvals) → Deploy to Production
```

### Image Tagging

```
Dev:        express-app:dev, express-app:dev-abc1234
Staging:    express-app:staging, express-app:staging-def5678
Production: express-app:prod, express-app:prod-ghi9012
```

### Testing Deployments

**Quick test all environments:**
```bash
# Dev
curl http://$(aws elbv2 describe-target-groups --names express-app-dev-alb-tg --region us-east-2 --query 'TargetGroups[0].LoadBalancerArns[0]' --output text | xargs -I {} aws elbv2 describe-load-balancers --load-balancer-arns {} --region us-east-2 --query 'LoadBalancers[0].DNSName' --output text)/health

# Staging
curl http://$(aws elbv2 describe-target-groups --names express-app-staging-alb-tg --region us-east-2 --query 'TargetGroups[0].LoadBalancerArns[0]' --output text | xargs -I {} aws elbv2 describe-load-balancers --load-balancer-arns {} --region us-east-2 --query 'LoadBalancers[0].DNSName' --output text)/health

# Production
curl http://$(aws elbv2 describe-target-groups --names express-app-prod-alb-tg --region us-east-2 --query 'TargetGroups[0].LoadBalancerArns[0]' --output text | xargs -I {} aws elbv2 describe-load-balancers --load-balancer-arns {} --region us-east-2 --query 'LoadBalancers[0].DNSName' --output text)/health
```

---

## 🔒 Security Features

### GitHub OIDC Authentication

**Required Secrets:**
- `AWS_ACCOUNT_ID` - Your 12-digit AWS account ID
- `OPENAI_API_KEY` - (Optional) For AI analysis

**IAM Role** (created by infrastructure repo):
- OIDC provider: `token.actions.githubusercontent.com`
- Role: `github-actions-deployment-role`
- Permissions: Scoped to `express-app-*` resources only

### Branch Protection Rules

**Dev:** PR required, 1 approvals, security scans must pass  
**Staging:** PR required, 1 approval, build + security scans must pass  
**Main:** PR required, 2 approvals, deploy + build + security scans must pass

Configure at: **Settings → Branches → Branch protection rules**

---

## 🚀 Deployment for TurboVets

### Step 1: Deploy Infrastructure First

👉 **Deploy [express-app-iac](https://github.com/chrisjamaica91/express-app-iac) first**

Follow the comprehensive deployment guide in the infrastructure README. This creates: VPC, ECS clusters, ECR, IAM roles, ALBs, CloudWatch logs for all three environments.

### Step 2: Fork This Repository

**On GitHub:**
1. Click **Fork** button on this repository
2. Select your organization (turbovets)
3. Click **Create fork**

**Note:** The fork automatically includes all branches (`main`, `dev`, `staging`), so you don't need to create them manually.

**Clone your fork:**
```bash
git clone https://github.com/YOUR-ORG/express-app.git
cd express-app
```

### Step 3: Configure GitHub Secrets

**Settings → Secrets and variables → Actions → New repository secret**

Add these secrets:
```
AWS_ACCOUNT_ID = 123456789012  # Your 12-digit AWS account ID
OPENAI_API_KEY = sk-...        # Optional (for AI deployment analysis)
```

**Important:** Do NOT add `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` - OIDC handles authentication.

### Step 4: Set Up Branch Protection Rules

**⚠️ Required Step** - Branch protection rules are NOT copied when forking. You must configure these manually.

**Settings → Branches → Add branch protection rule**

**For `dev` branch:**
- Branch name pattern: `dev`
- ✅ Require a pull request before merging
- Number of required approvals: **1**
- ✅ Require status checks to pass: `security-scan`, `build`

**For `staging` branch:**
- Branch name pattern: `staging`
- ✅ Require a pull request before merging
- Number of required approvals: **1**
- ✅ Require status checks to pass: `security-scan`, `build`, `deploy`

**For `main` branch:**
- Branch name pattern: `main`
- ✅ Require a pull request before merging
- Number of required approvals: **2**
- ✅ Require status checks to pass: `security-scan`, `build`, `deploy`

### Step 5: Trigger First Deployment

```bash
# Switch to dev branch (already exists from fork)
git checkout dev

# Make a small change to trigger deployment
echo "# Deployed to TurboVets" >> README.md
git add README.md
git commit -m "Trigger initial deployment"
git push origin dev
```

Monitor deployment in the **Actions** tab. First deployment takes ~5 minutes.

### Step 6: Verify Deployment

After the workflow completes, verify the dev environment is running:

```bash
# Get ALB DNS name and test health endpoint
curl http://$(aws elbv2 describe-target-groups --names express-app-dev-alb-tg --region us-east-2 --query 'TargetGroups[0].LoadBalancerArns[0]' --output text | xargs -I {} aws elbv2 describe-load-balancers --load-balancer-arns {} --region us-east-2 --query 'LoadBalancers[0].DNSName' --output text)/health
```

**Expected response:**
```json
{"status":"ok","timestamp":"2026-06-04T20:02:12.758Z"}
```

### Step 7: Promote Through Environments

- **Dev → Staging:** Create PR, get 1 approval, merge
- **Staging → Main:** Create PR, get 2 approvals, merge

Each merge triggers automatic deployment to that environment.

---

## 📊 Bonus Features Implemented

### ✅ Completed

- ✅ **Multiple environments** (dev/staging/prod with separate VPCs)
- ✅ **CloudWatch logs** for all services
- ✅ **Enhanced security** (4 layers of scanning)
- ✅ **Multi-stage Docker builds** (80% smaller images)
- ✅ **OIDC authentication** (no IAM keys)
- ✅ **AI deployment analysis** (GPT-4o-mini)
- ✅ **Branch protection** with graduated approvals
- ✅ **Container hardening** (non-root, Alpine, minimal packages)
- ✅ **S3 remote backend** with native locking
- ✅ **GitHub Actions cache** for faster builds

### ❌ Not Implemented

- ❌ **Route53 + HTTPS** (requires domain ownership)
- ❌ **CloudWatch alarms** (logs exist, no alerting)

---


## 📝 License

MIT License

---

**Built with ❤️ for TurboVets DevOps Assessment**

*Production-grade DevOps with security-first approach*
