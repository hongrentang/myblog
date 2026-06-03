---
title: "Cloud Credential Leak — When Your AWS Key in a Public Repo Cost $50,000 Overnight"
date: 2026-06-03
weight: 100330
slug: "cloud-credential-leak-response"
tags: ["kubernetes", "security", "cloud", "credential", "troubleshooting"]
categories: ["Security"]
description: "A cloud credential leak incident — how an AWS access key committed to a public GitHub repo was discovered by automated scanners within minutes, leading to cryptomining and data exfiltration costing $50,000"
keywords: "cloud credential leak, aws key exposed github, secret scanning, credential rotation incident, cloud security breach"
draft: false
featured: true
cover:
  image: ""
  caption: "Cloud Credential Leak — Incident Response"
---

# Cloud Credential Leak — When Your AWS Key in a Public Repo Cost $50,000 Overnight

## Common Search Queries

- aws access key exposed in github public repository
- cloud credential leak incident response
- aws key compromised by automated scanner
- kubernetes service account token leaked
- secret scanning pre-commit hook prevention

---

## The Incident

**Environment**: Kubernetes cluster running on AWS (EKS), 3 production clusters across us-east-1 and eu-west-1, 150+ microservices, Terraform-managed infrastructure in a **public** GitHub monorepo.

**Time**: Tuesday 14:30 UTC. An engineer on the infrastructure team pushed a commit containing infrastructure configurations. Within 12 minutes, AWS Cost Explorer showed a vertical spike. By 14:50 UTC — just 20 minutes later — the company's monitoring Slack channel lit up with "AWS Budget Alert: $5,000 exceeded."

**Initial Symptoms**:

```bash
# AWS Budget Alert — first sign of trouble
Budget Name: monthly-soft-limit
Threshold: 80% ($4,000 / $5,000)
Current Spend: $5,320
Forecast: $38,000
```

The on-call engineer pulled up AWS CloudTrail and saw API calls from IPs in Russia, China, and Nigeria — regions where the company had zero business operations.

```bash
# CloudTrail console — unusual API activity
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::EC2::Instance \
  --start-time "2026-06-03T14:30:00Z" \
  --query 'Events[?contains(SourceIPAddress, `185.`) || contains(SourceIPAddress, `102.`)]' \
  --max-items 50
```

**Impact**: $50,300 in unauthorized AWS charges within 18 hours. 3 S3 buckets with customer PII data accessed. 1,200+ GPU spot instances provisioned across 6 AWS regions. Full AWS account credential rotation. Production deployment freeze for 36 hours. 2 customer data breach notifications required.

---

## Background

Six months prior, the infrastructure team had set up cross-account IAM roles for Kubernetes service integrations. An AWS access key (`AKIAIOSFODNN7EXAMPLE`) with `ReadOnlyAccess` + `S3:GetObject` was generated for a monitoring tool that needed to read S3 bucket metrics.

The key was never intended for production use. It was created for a quick proof-of-concept. But it worked, so it stayed. A developer added it to a Kubernetes ConfigMap for convenience:

```yaml
# configmap-infra.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: monitoring-config
data:
  aws_access_key_id: AKIAIOSFODNN7EXAMPLE
  aws_secret_access_key: wJalrXUtpnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
  s3_bucket: production-customer-backups
```

The ConfigMap was checked into the infrastructure monorepo. The repository had been **public** since day one — "open source our infrastructure patterns" was the team's philosophy.

An automated scanner run by a credential harvesting group scanned the public repository's commit history. The regex `AKIA[0-9A-Z]{16}` matched immediately. The attacker had the working key pair within 3 minutes of the commit.

---

## Investigation

### Step 1: Confirm Credential Exposure

```bash
# Check GitHub for public access to the repository
# The repo was public — anyone could clone it

# Search commit history for credential patterns
git log --all --oneline | head -5
git show HEAD --stat | grep -i "configmap"

# Use GitHub's secret scanning API to confirm exposure
gh api repos/org/infra-config/secret-scanning/alerts \
  --jq '.[] | select(.secret_type == "aws_access_key")'
```

One alert confirmed: the AWS access key was detected by GitHub's secret scanner. But the alert was sent to an email alias that nobody monitored.

### Step 2: Enumerate Compromised Resources

```bash
# List all EC2 instances launched after the compromise
aws ec2 describe-instances \
  --filters "Name=launch-time,After=2026-06-03T14:30:00Z" \
  --query 'Reservations[].Instances[].[InstanceId,InstanceType,LaunchTime,Placement.AvailabilityZone,State.Name]' \
  --output table
```

The output was catastrophic:

| Instance Type | Count | Region | Launch Time |
|--------------|-------|--------|-------------|
| p3.2xlarge   | 480   | us-east-1 | 14:35 - 14:50 UTC |
| p3.8xlarge   | 320   | us-west-2 | 14:38 - 15:02 UTC |
| p4d.24xlarge | 200   | eu-west-1 | 14:42 - 15:10 UTC |
| g4dn.xlarge  | 120   | ap-southeast-1 | 14:45 - 15:05 UTC |
| p3.2xlarge   | 80    | sa-east-1 | 14:50 - 15:08 UTC |

**Total: 1,200 instances** — all GPU-optimized for cryptomining.

```bash
# Check IAM user activity
aws iam get-credential-report --output text | grep AKIAIOSFODNN7EXAMPLE
```

```bash
# List all S3 buckets accessed by the compromised key
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIAIOSFODNN7EXAMPLE \
  --query 'Events[?EventName==`GetObject`].[Resources[0].ARN,EventTime,SourceIPAddress]' \
  --output table
```

3 S3 buckets were accessed — one containing unencrypted customer PII (names, email addresses, phone numbers).

### Step 3: Trace the Attacker's Actions

CloudTrail logs and EC2 console activity revealed the attacker's timeline:

| Time (UTC) | Action | Detail |
|-----------|--------|--------|
| 14:33 | Credential discovery | GitHub repo scan, regex match on AKIA key |
| 14:34 | Credential validation | `sts:GetCallerIdentity` from IP 185.xx.xx.xx |
| 14:35 - 14:50 | Resource enumeration | `ec2:DescribeInstances`, `s3:ListBuckets`, `iam:GetUser` |
| 14:35 - 15:10 | Cryptomining deployment | 1,200 GPU spot instances across 6 regions |
| 14:40 | Persistence | Created IAM user `support-engineer-02` with `AdministratorAccess` |
| 14:42 - 14:48 | Data exfiltration | Downloaded 12 GB from S3 buckets (customer backups) |
| 14:55 | Lateral movement | Attempted `eks:DescribeCluster` — blocked by network policy |

### Step 4: Identify the Leak Vector

```bash
# Find the commit that introduced the credential
git log --all --oneline --source -S "AKIAIOSFODN"
```

The credential was introduced in commit `a3f2b1c` on the `main` branch, merged 12 minutes before the first CloudTrail anomaly. The file `configmap-infra.yaml` contained the plaintext AWS access key and secret.

---

## Root Cause

1. **Plaintext cloud credentials in ConfigMap**: AWS access key and secret key were hardcoded directly in a Kubernetes ConfigMap YAML file. No secrets management tool (AWS Secrets Manager, HashiCorp Vault, or Sealed Secrets) was used

2. **Public repository with sensitive infrastructure code**: The infrastructure monorepo was public. Any file committed to `main` became immediately visible to automated scanners. No branch protection rules prevented direct pushes to `main`

3. **No pre-commit hook for secret detection**: No `pre-commit` hooks, `talisman`, or `gitleaks` were configured to block commits containing credential patterns

4. **IAM policy was overly permissive**: Although the key was labeled "read-only," the IAM policy allowed `s3:GetObject` on **all** S3 buckets (`arn:aws:s3:::*`), not scoped to specific buckets. This enabled data exfiltration from customer-data buckets

5. **No IAM access key usage constraints**: The key had no `aws:SourceIp` or `aws:SourceArn` condition — no restriction on where it could be used from. Attackers used it from their own infrastructure

6. **No resource-level budget alerts**: AWS Budget alerts were set at the account level but the threshold was too high ($5,000). The first alert arrived when $5,000 had already been spent. There were no per-service or per-region budget limits

---

## Resolution

### Emergency Response (First 30 Minutes)

```bash
# Step 1: Deactivate the compromised access key
aws iam update-access-key \
  --access-key-id AKIAIOSFODNN7EXAMPLE \
  --status Inactive

# Step 2: Delete the attacker-created IAM user
aws iam delete-user --user-name support-engineer-02
# (First need to detach policies and delete access keys)

# Step 3: Detach and delete unauthorized policies
aws iam list-attached-user-policies --user-name support-engineer-02
aws iam detach-user-policy --user-name support-engineer-02 \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Step 4: Terminate all unauthorized EC2 instances
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Value=running" \
  --query 'Reservations[].Instances[?LaunchTime>`2026-06-03T14:30:00Z`].[InstanceId,Placement.Region]' \
  --output text | awk '{print $2, $1}' | while read region id; do
    aws ec2 terminate-instances --instance-ids "$id" --region "$region"
  done

# Step 5: Delete unauthorized security group rules
aws ec2 describe-security-groups \
  --query 'SecurityGroups[?GroupName!=`default`].[GroupId,GroupName]' \
  --output text | xargs -r -I{} aws ec2 delete-security-group --group-id {}
```

### Containment Verification

```bash
# Verify the key is no longer active
aws iam get-access-key-last-used \
  --access-key-id AKIAIOSFODNN7EXAMPLE

# Check for any remaining unauthorized resources
aws resourcegroupstaggingapi get-resources \
  --query 'ResourceTagMappingList[].ResourceARN'

# Rotate all remaining IAM keys in the account
aws iam list-users --query 'Users[].UserName' --output text | \
  while read user; do
    aws iam list-access-keys --user-name "$user" --query 'AccessKeyMetadata[].AccessKeyId' --output text | \
    while read key; do
      aws iam create-access-key --user-name "$user"
      aws iam update-access-key --access-key-id "$key" --status Inactive
    done
  done
```

### Long-Term Remediation

**1. Implement Secrets Management**

```yaml
# Use AWS Secrets Manager + External Secrets Operator
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: aws-credentials
spec:
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: aws-credentials
  data:
    - secretKey: aws_access_key_id
      remoteRef:
        key: /infra/aws/monitoring-key
        property: access_key_id
    - secretKey: aws_secret_access_key
      remoteRef:
        key: /infra/aws/monitoring-key
        property: secret_access_key
```

**2. Enforce IAM Least Privilege**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::monitoring-metrics/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "10.0.0.0/8"
        }
      }
    }
  ]
}
```

**3. Add Pre-Commit Hooks**

```bash
# .gitleaks.toml — prevent credential commits
[[rules]]
id = "aws-access-key"
description = "AWS Access Key"
regex = '''AKIA[0-9A-Z]{16}'''
tags = ["aws", "credentials"]

[[rules]]
id = "aws-secret-key"
description = "AWS Secret Key"
regex = '''(?i)aws(.{0,20})?['\"][0-9a-zA-Z\/+]{40}['\"]'''
tags = ["aws", "credentials"]
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: detect-private-key
```

**4. CI/CD Secret Scanning**

```yaml
# GitHub Actions — scan every push
name: Secret Scanning
on: [push, pull_request]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**5. IAM Conditions and SCPs**

```json
// Service Control Policy — enforce constraints on all IAM users
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RestrictEC2InstanceTypes",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringNotEquals": {
          "ec2:InstanceType": ["t3.medium", "t3.large", "m5.large", "c5.large"]
        }
      }
    },
    {
      "Sid": "RequireIAMConditions",
      "Effect": "Deny",
      "Action": "iam:CreateAccessKey",
      "Resource": "*",
      "Condition": {
        "Null": {
          "aws:RequestedRegion": "true"
        }
      }
    }
  ]
}
```

**6. AWS Budgets with Automated Response**

```bash
# Create per-region budget alerts
aws budgets create-budget \
  --budget '{
    "BudgetName": "us-east-1-monthly",
    "BudgetLimit": {"Amount": "2000", "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST",
    "CostFilters": {"Region": ["us-east-1"]}
  }' \
  --notifications-with-subscribers '[
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 80
      },
      "Subscribers": [{"SubscriptionType": "SNS", "Address": "arn:aws:sns:us-east-1:123456789:security-alerts"}]
    }
  ]'

# Enable AWS CloudWatch anomaly detection on spend
aws cloudwatch put-anomaly-detector \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --stat Average
```

**7. Make the Infrastructure Repo Private**

The repository was immediately made private. All existing forks were locked.

---

## Lessons Learned

- **Plaintext credentials in Kubernetes manifests are bombs**: ConfigMaps and Secrets should **never** contain raw cloud credentials. Use External Secrets Operator, Sealed Secrets, or Vault. If you see `aws_access_key_id: AKIA...` in a YAML file, assume it is already compromised

- **Public repo + infrastructure code = existential risk**: The team's philosophy of "open source infrastructure patterns" directly caused this breach. Security through obscurity is not security, but making your IAM configs publicly searchable is actively dangerous

- **IAM conditions save accounts**: Adding `aws:SourceIp` and `aws:SourceArn` conditions to IAM policies would have prevented the attacker from using the key outside the VPC even if it was compromised

- **Budget alerts need to be aggressive and multi-layered**: A single $5,000 account-level budget alert is not sufficient. Set per-region, per-service, and per-account budgets at 50/75/90/100% thresholds. CloudWatch anomaly detection on billing metrics catches spikes in near-real-time

- **GitHub secret scanning alerts need active monitoring**: GitHub's built-in secret scanner found the key but sent the alert to an unmonitored email. Integrate it with PagerDuty or Slack. Treat secret scanning alerts as P0 incidents

- **Pre-commit hooks are mandatory**: A simple `gitleaks` pre-commit hook would have rejected the commit containing the AWS key. There is no excuse not to have this. It takes 5 minutes to set up and prevents incidents like this

- **Immutable infrastructure tags are valuable**: Tagging all authorized EC2 instances with `Environment: Production` and `ManagedBy: Terraform` made it easy to script the termination of 1,200 unauthorized instances

---

## Summary

The attack chain:

```
Developer hardcodes AWS credentials in ConfigMap YAML
→ Commits configmap-infra.yaml to main branch of public repository
→ GitHub secret scanner detects the key, sends email to unmonitored alias
→ Attacker's automated bot scans public GitHub commits for AKIA pattern
→ Bot validates credentials via sts:GetCallerIdentity (3 min post-commit)
→ Enumerates resources: EC2, S3, IAM (5 min post-commit)
→ Launches 1,200 GPU spot instances across 6 regions for cryptomining
→ Creates IAM persistence user with AdministratorAccess
→ Exfiltrates 12 GB of customer PII data from S3 (10 min post-commit)
→ AWS Budget alert fires at $5,000 — 18 minutes post-commit
→ On-call engineer deactivates key, terminates instances
→ $50,300 bill, 3 data breach notifications, 36-hour deployment freeze
```

**Remediation**: 36 hours. **Cost**: $50,300 + legal/compliance fallout from 3 data breach notifications. **Root cause**: one ConfigMap with hardcoded credentials committed to a public repository.

The fix chain:

```
Hardcoded key in ConfigMap
→ Replace with External Secrets Operator + AWS Secrets Manager
→ Add gitleaks pre-commit hook (blocks AKIA patterns)
→ Add gitleaks CI job on every push
→ Scope IAM policy to specific resource ARN + SourceIp condition
→ Apply SCP to restrict instance types and enforce IAM conditions
→ Set per-region budgets at 50/75/90/100% with SNS to PagerDuty
→ Make infrastructure repository private
→ Integrate GitHub secret scanning alerts with incident management
```

A single hardcoded credential. Twelve minutes to discovery. Eighteen hours of cryptomining. Three customer data breaches. Fifty thousand dollars. All of it preventable with a pre-commit hook and an IAM condition.
