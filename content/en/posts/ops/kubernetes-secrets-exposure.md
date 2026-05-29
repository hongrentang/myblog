---
title: "Kubernetes Secrets Exposure — How Plaintext Secrets in Git Leaked Your Cluster Away"
date: 2026-05-29
weight: 100210
slug: "kubernetes-secrets-exposure"
tags: ["kubernetes", "security", "secrets", "troubleshooting"]
categories: ["Security"]
description: "A Kubernetes secrets exposure incident — how secrets committed as YAML to a public Git repo led to cluster compromise, and why external secret management is essential"
keywords: "kubernetes secrets exposed git, k8s secret plaintext, sealed secrets, external secrets operator, kubeseal"
draft: false
featured: true
cover:
  image: ""
  caption: "Kubernetes Secrets Exposure — Incident Response"
---

# Kubernetes Secrets Exposure — How Plaintext Secrets in Git Leaked Your Cluster Away

## Common Search Queries

- kubernetes secrets exposed in git
- k8s secret base64 not encrypted
- how to store kubernetes secrets safely
- sealed secrets vs external secrets
- git secrets leaked kubernetes

---

## The Incident

**Environment**: K8S v1.28, 5 clusters (dev/staging/prod x 2 regions), GitOps with ArgoCD.

**Time**: Monday 09:45. A security scanner (GitGuardian) alert: production secrets found in a public GitHub repository.

**Initial Discovery**: An intern had committed a Kubernetes `Secret` YAML file to a public GitHub repo — in plaintext. The repo was meant to be private, but a misconfigured repository setting had made it public for three days.

```bash
# What the scanner found
# File: infrastructure/secrets/prod-db-credentials.yaml

apiVersion: v1
kind: Secret
metadata:
  name: prod-db-credentials
  namespace: production
type: Opaque
data:
  DB_PASSWORD: cHJvZHVjdGlvbl9zdXBlcl9zZWNyZXRfcGFzc3dvcmQ=  # "production_super_secret_password"
  DB_USERNAME: cHJvZF9hZG1pbiA=  # "prod_admin"
  DB_CONNECTION_STRING: cG9zdGdyZXNxbDovL3Byb2RfYWRtaW46cHJvZHVjdGlvbl9zdXBlcl9zZWNyZXRfcGFzc3dvcmRAZGIuZXhhbXBsZS5jb206NTQzMi9wcm9kX2Ri
```

**Impact**: Database credentials, API keys, and TLS private keys for the production cluster were exposed on the public internet for 72 hours. Emergency credential rotation required a coordinated multi-team effort.

---

## Background

The team had been practicing GitOps with ArgoCD for over a year. Every manifest lived in Git, and ArgoCD synced them to the cluster. The workflow was smooth — except for secrets.

Secrets in Kubernetes are **not encrypted** by default. The `data` field is only base64-encoded, not encrypted. Anyone with access to the cluster — or the YAML file — can decode them:

```bash
echo "cHJvZHVjdGlvbl9zdXBlcl9zZWNyZXRfcGFzc3dvcmQ=" | base64 -d
# Output: production_super_secret_password
```

Despite knowing this, the team had been committing secrets as raw YAML for "speed" — the Git repo was supposed to be private. Meanwhile, a junior engineer was onboarding and needed to deploy a new database. They followed the existing pattern: create a Secret YAML, commit, push. But the repo visibility was accidentally set to `public` during a settings migration three days earlier.

---

## Investigation

### Step 1: Identify What Was Exposed

The GitGuardian alert listed 47 secrets across 12 YAML files:

```bash
# Search for all Secret YAML files in git history
git -C /home/kali/桌面/code/claude/myblog log --all --diff-filter=A --name-only --format="" | grep -E 'secrets|credentials|\.secret\.'
```

```bash
# Check all files ever committed that contain "kind: Secret"
git log --all --diff-filter=A -p --full-history | grep -B5 "kind: Secret" | head -20
```

### Step 2: Extract the Exposed Credentials

```bash
# List all affected files
grep -rl "kind: Secret" /path/to/repo/secrets/
```

Each file contained plaintext base64-encoded secrets:

| Service | Secret Type | Exposure Window |
|---------|------------|-----------------|
| Production PostgreSQL | Password | 72 hours |
| Production Redis | Auth Token | 72 hours |
| AWS S3 Key | Access Key + Secret | 72 hours |
| Stripe API Key | Live Key | 72 hours |
| GitHub Deploy Key | SSH Private Key | 72 hours |
| TLS Certificates | Private Key + Cert | 72 hours |

### Step 3: Check Git History for Unauthorized Clones

```bash
# GitHub API: check clone traffic for the public period
gh api repos/hongrentang/myblog/traffic/clones
```

Unfortunately, GitHub only retains traffic data for 14 days, and anyone could have cloned the repo anonymously during those 72 hours.

### Step 4: Check Cluster for Signs of Compromise

```bash
# Check for unusual pods or deployments
kubectl get pods --all-namespaces | grep -iv "Running\|Completed"

# Check audit logs for unusual access patterns
grep -E "secret|credential|password" /var/log/kubernetes/audit.log | grep -v "system:serviceaccount:kube-system" | head -20
```

No immediate signs of cluster compromise were found — but with exposed credentials, the assumption was that all secrets were compromised.

---

## Root Cause

1. **Plaintext secrets in Git**: No encryption at rest for Kubernetes Secrets in the repository
2. **Repo accidentally public**: A settings migration changed the repo visibility from private to public
3. **No pre-commit scanning**: No tools like `git-secrets`, `truffleHog`, or pre-commit hooks to block secret commits
4. **No external secrets management**: The team knew about Sealed Secrets and External Secrets Operator but hadn't adopted them
5. **No automated rotation policy**: Even if discovered immediately, there was no runbook for emergency credential rotation

---

## Resolution

### Emergency Credential Rotation

```bash
# 1. Revoke the exposed GitHub deploy key
gh repo deploy-key list
gh repo deploy-key delete <id>

# 2. Rotate database credentials
# Connect to PostgreSQL and change password
kubectl exec -it postgres-primary-0 -n production -- psql -c "ALTER USER prod_admin WITH PASSWORD 'new-secure-password';"

# 3. Rotate Redis auth token
kubectl exec -it redis-master-0 -n production -- redis-cli CONFIG SET requirepass "new-token"

# 4. Regenerate AWS access keys via IAM console
aws iam create-access-key --user-name prod-s3-user
aws iam update-access-key --access-key-id OLD_KEY --status Inactive
aws iam delete-access-key --access-key-id OLD_KEY

# 5. Rotate Stripe API key via Stripe dashboard
# 6. Reissue TLS certificates via cert-manager
kubectl delete certificate -n production -l app=api-gateway
# cert-manager will automatically reissue
```

### Remove Secrets from Git History

```bash
# Use git filter-repo to purge secrets from all history
pip install git-filter-repo

# Create a list of files to remove
echo "infrastructure/secrets/prod-db-credentials.yaml" > /tmp/secret_files.txt

# Purge them from history
git filter-repo --paths-from-file /tmp/secret_files.txt --force

# Force push to overwrite remote history
git push --force --all
```

**Warning**: `git filter-repo` rewrites history. All collaborators must clone fresh afterward.

### Permanent Fix: External Secrets Operator

The team implemented External Secrets Operator (ESO) to pull secrets from AWS Secrets Manager:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: prod-db-credentials
  namespace: production
spec:
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: prod-db-credentials
  data:
  - secretKey: DB_PASSWORD
    remoteRef:
      key: /production/db/password
```

### Prevention Measures

1. **Pre-commit hooks**: Block any commit containing `kind: Secret` with base64-encoded data

```bash
# .git/hooks/pre-commit — block plaintext secrets
#!/bin/bash
if git diff --cached | grep -A5 "kind: Secret" | grep -q "data:"; then
  echo "ERROR: Plaintext secrets detected in commit!"
  echo "Use Sealed Secrets or External Secrets instead."
  exit 1
fi
```

2. **GitHub secret scanning**: Enable push protection for the repository
3. **Repository visibility monitoring**: Set up alerts for any change to repo visibility settings
4. **Automated credential rotation**: Use a Secrets Manager with automatic rotation schedules
5. **Sealed Secrets for GitOps**: Encrypt secrets with `kubeseal` so they're safe in Git

```bash
# Encrypt a secret
kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets \
  < prod-db-credentials.yaml > prod-db-credentials-sealed.yaml
# Only the sealed YAML is committed to Git
```

---

## Lessons Learned

- **`base64` is not encryption**: Base64 encoding is obfuscation, not security. Treat any base64-encoded Secret as compromised
- **If it's in Git, it's forever**: Even after deleting a file, it remains in git history. Use `git-filter-repo` or never commit secrets in the first place
- **External Secrets Operator is the right pattern**: Secrets should live in a dedicated secrets manager (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager), not in Git
- **Repo visibility is a security boundary**: Private repos can accidentally become public. Treat every commit to any repo as potentially public
- **Pre-commit hooks catch mistakes early**: A simple pre-commit hook would have blocked the intern's commit and prevented the entire incident

---

## Summary

The incident chain:

```
Intern needs to deploy database
→ Follows existing pattern: creates plaintext Secret YAML
→ Commits to repo that was accidentally set to public
→ 47 production secrets exposed on public GitHub for 72 hours
→ GitGuardian scanner triggers alert
→ Emergency credential rotation across 6 services
→ History rewritten with git-filter-repo
→ External Secrets Operator adopted as permanent fix
```

Global credential rotation: 4 hours. Fixing the process: 2 days. The cost of using `echo "password" | base64` instead of `kubeseal` or ESO: incalculable. Stop putting secrets in Git — yesterday.
