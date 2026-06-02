---
title: "CI/CD Pipeline Secret Leak — How a CI Log Exposed AWS Keys and Led to a Cryptomining Disaster"
date: 2026-06-02
weight: 100250
slug: "cicd-pipeline-secret-leak"
tags: ["security", "ci-cd", "devops", "cloud", "troubleshooting"]
categories: ["Security"]
description: "A CI/CD pipeline security incident — how AWS keys printed in a CI build log were scraped by a bot, leading to a $47,000 cryptomining bill and a complete cloud account rebuild"
keywords: "ci cd pipeline secret leak, github actions secret exposure, aws keys leaked in ci log, cryptomining cloud bill, devsecops pipeline security"
draft: false
featured: true
cover:
  image: ""
  caption: "CI/CD Pipeline Secret Leak — Incident Response"
---

# CI/CD Pipeline Secret Leak — How a CI Log Exposed AWS Keys and Led to a Cryptomining Disaster

## Common Search Queries

- ci cd pipeline secret exposed in logs
- github actions aws credentials leak
- aws keys leaked in build logs
- cryptomining attack cloud bill
- how to secure ci cd pipeline secrets

---

## The Incident

**Environment**: GitHub Actions CI/CD, AWS account with 200+ resources across 5 regions, 40+ microservices.

**Time**: Saturday 03:15 AM. AWS Cost Anomaly Detection alert: estimated spend projected to exceed $10,000 in the next 24 hours — normally $800/day.

**Initial Symptoms**: The finance team's phone started ringing. AWS had auto-sent a "Spend Warning" notification at $5,000 over the daily average. By the time the on-call engineer checked, the bill had hit $12,000 and was climbing.

```bash
# What the engineer saw in AWS Cost Explorer
Region: us-east-1
Service: EC2 (Spot Instances)
Unusual spike starting at 02:00 UTC
Instance count: 347 (normally 50)
Instance types: p3.2xlarge, p3.8xlarge (GPU instances — cryptominer favorites)
```

**Impact**: $47,000 in unauthorized AWS charges over 36 hours. Full cloud account credential rotation. 48 hours of production deployment freeze.

---

## Background

The team used GitHub Actions for CI/CD. AWS credentials were stored as GitHub Actions secrets — properly configured, masked in logs, never exposed in code. The security was considered "good enough."

But there was a gap: a developer working on a new deployment script added debug logging to print environment variables during a build step:

```yaml
# deploy.yml (simplified)
steps:
  - name: Configure AWS credentials
    uses: aws-actions/configure-aws-credentials@v1
    with:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      aws-region: us-east-1

  - name: Debug — print env
    run: env | grep AWS_            # ← This line caused the incident
```

GitHub Actions masks secrets in logs — but only for the `${{ secrets.* }}` syntax. The `env` command printed the unmasked values because at runtime, secrets are exported as real environment variables. The log showed:

```
AWS_ACCESS_KEY_ID=AKIAXXXXXXXXXXXX
AWS_SECRET_ACCESS_KEY=xxxxxxxxxxxxxx
```

An attacker had a bot scanning public GitHub Actions logs. Within 7 minutes of the commit being pushed to a public repository, the credentials were compromised.

---

## Investigation

### Step 1: Identify the Leak Source

```bash
# Check recent GitHub Actions runs for exposed secrets
gh run list --workflow=deploy.yml --limit=10 --json=databaseId,conclusion,headBranch
```

The most recent run on the `feature/debug-logging` branch showed the full environment variables in the raw log output.

### Step 2: Assess Cloud Account Damage

```bash
# List all EC2 instances launched in the last 24 hours
aws ec2 describe-instances --filters "Name=launch-time,After=2026-06-01T00:00:00Z" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,LaunchTime:LaunchTime}' \
  --output table
```

347 instances launched:
- 280 p3.2xlarge (GPU)
- 67 p3.8xlarge (GPU)
- All running in regions the account never used (us-west-2, eu-west-1)

```bash
# Check IAM activity
aws iam get-credential-report --output text | cut -d',' -f1,4,5,16 | head -20
```

The compromised key was used from IPs in Eastern Europe and China.

```bash
# Check S3 access
aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=arn:aws:s3:::production-data-backup
```

The attacker had accessed S3 buckets containing production database backups.

### Step 3: Trace the Attack Path

The attacker's bot:
1. Scanned public GitHub Actions logs for AWS access keys (regex: `AKIA[0-9A-Z]{16}`)
2. Found the exposed keys within 7 minutes
3. Used the keys to launch GPU spot instances for cryptomining
4. Created a new IAM user with admin privileges for persistence
5. Exfiltrated data from accessible S3 buckets
6. Set up additional IAM access keys for backup access

---

## Root Cause

1. **Debug print of environment variables in CI**: The `env | grep AWS_` line exposed masked secrets because GitHub Actions exports them as real env vars
2. **Public repository with sensitive CI/CD**: The repository was public. CI logs were publicly accessible
3. **Over-permissive AWS keys**: The CI/CD IAM user had `AdministratorAccess` — far more than needed
4. **No CI log scanning**: No automated scanning for secrets in build logs (e.g., truffleHog, GitGuardian CI)
5. **No AWS resource limits**: No Service Control Policies (SCPs) or budget limits to prevent runaway resource creation

---

## Resolution

### Emergency (Containment)

```bash
# 1. Immediately deactivate the compromised AWS key
aws iam update-access-key --access-key-id AKIAXXXXXXXXXXXX --status Inactive

# 2. Terminate all unauthorized EC2 instances
aws ec2 describe-instances --filters "Name=instance-state-name,Value=running" \
  --query 'Reservations[].Instances[?LaunchTime>`2026-06-01`].[InstanceId]' \
  --output text | xargs -r aws ec2 terminate-instances --instance-ids

# 3. Delete unauthorized IAM users and keys
aws iam list-users | jq -r '.Users[] | select(.CreateDate > "2026-06-01") | .UserName' | \
  xargs -r -I{} aws iam delete-user --user-name {}

# 4. Revoke and reissue all AWS credentials
aws iam create-access-key --user-name cicd-user
# Update GitHub secrets with new key
```

### Budget and Resource Controls

```bash
# Set AWS Budget alert at $500
aws budgets create-budget \
  --budget-file '{"BudgetName":"monthly-budget","BudgetLimit":{"Amount":"5000","Unit":"USD"},"TimeUnit":"MONTHLY","BudgetType":"COST"}' \
  --notifications-with-subscribers '[]'

# Apply Service Control Policy to restrict instance types
# Organizations SCP to allow only approved instance families
```

### Fix the CI/CD Pipeline

```yaml
# Fixed deploy.yml — no debug logging of env vars
steps:
  - name: Configure AWS credentials
    uses: aws-actions/configure-aws-credentials@v4
    with:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      aws-region: us-east-1

  - name: Deploy application
    run: |
      # Use specific env vars, never dump all
      ./deploy.sh
```

### Implement Least Privilege for CI/CD

```bash
# Create scoped IAM policy for CI/CD
# Instead of AdministratorAccess, allow only:
# - ecr:PushImage, ecr:PullImage
# - ecs:UpdateService, ecs:RegisterTaskDefinition
# - s3:PutObject (specific bucket)
```

### Add Secret Scanning in CI/CD Pipeline

```yaml
# Add truffleHog to detect secrets in build logs
steps:
  - name: Secret scanning
    uses: trufflesecurity/trufflehog@v3
    with:
      path: .
      base: ${{ github.event.before }}
      head: HEAD
```

### Monitoring and Alerting

```yaml
# AWS CloudWatch alarm for unusual instance count
aws cloudwatch put-metric-alarm \
  --alarm-name high-instance-count \
  --metric-name RunningInstanceCount \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 100 \
  --comparison-operator GreaterThanThreshold
```

---

## Lessons Learned

- **GitHub Actions secret masking is not encryption**: Secrets masked with `***` in the UI are still real env vars at runtime. Any command that prints env vars will expose them
- **CI/CD keys need least privilege**: `AdministratorAccess` in CI means every commit can destroy the account. Scope CI credentials to exactly what the pipeline needs
- **Public repo + CI logs = public secrets**: If the repository is public, CI logs are public. Assume attackers have bots watching every new run
- **Budget alerts save money**: AWS budget alerts at $500/$1000/$5000 would have caught this within 10 minutes, not 12 hours
- **IAM credential report is your friend**: `aws iam generate-credential-report` shows when keys were last used. Audit it weekly

---

## Summary

The attack chain:

```
Developer adds debug logging in CI/CD workflow: env | grep AWS_
→ Commits and pushes to public repository
→ GitHub Actions runs workflow, prints AWS credentials in log
→ Attacker's bot scans public CI logs, finds AKIA key
→ Launches 347 GPU spot instances for cryptomining
→ Creates persistence IAM user
→ Exfiltrates S3 data
→ $47,000 AWS bill in 36 hours
```

Remediation: 48 hours. Cost: $47,000 + 48 hours of deployment freeze. Prevention: never print env vars in CI, scope IAM permissions to minimum, set budget alerts. The debug line that caused this: one line of code. The cost: $47,000. Delete your debug logging before you commit.
