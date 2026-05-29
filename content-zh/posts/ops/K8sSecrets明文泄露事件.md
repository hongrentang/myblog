---
title: "K8s Secrets 明文泄露事件——提交到 Git 的凭据如何让整个集群沦陷"
date: 2026-05-29
weight: 100210
slug: "kubernetes-secrets-exposure"
tags: ["kubernetes", "security", "secrets", "troubleshooting"]
categories: ["Security"]
description: "Kubernetes Secrets 泄露事件复盘——明文凭据被提交到公开 Git 仓库导致集群凭据泄露，以及为什么必须使用外部 Secrets 管理方案"
keywords: "kubernetes secrets 泄露 git, k8s secret 明文, sealed secrets, external secrets operator, kubeseal"
draft: false
featured: true
cover:
  image: ""
  caption: "K8s Secrets 明文泄露——安全事件排查"
---

# K8s Secrets 明文泄露事件——提交到 Git 的凭据如何让整个集群沦陷

## 常见搜索词

- kubernetes secrets 在 git 中泄露
- k8s secret base64 不是加密
- 如何安全存储 kubernetes 密钥
- sealed secrets vs external secrets
- git 密钥泄露 kubernetes

---

## 故障经过

**环境**: K8S v1.28，5 个集群（dev/staging/prod × 2 区域），使用 ArgoCD GitOps。

**时间**: 周一 09:45。GitGuardian 安全扫描告警：生产环境凭据出现在公开 GitHub 仓库中。

**发现过程**: 一名实习生将 Kubernetes `Secret` YAML 文件以明文形式提交到了公开的 GitHub 仓库。这个仓库本应是私有的，但三天前的一次设置迁移误将其变更为公开。

```bash
# 扫描器发现的内容
# 文件: infrastructure/secrets/prod-db-credentials.yaml

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

**影响**: 生产集群的数据库凭据、API 密钥、TLS 私钥在公网暴露了 72 小时。紧急凭证轮换需要多团队协调配合。

---

## 背景

团队使用 ArgoCD 实践 GitOps 已超过一年。每个清单都存放在 Git 中，ArgoCD 将它们同步到集群。工作流非常流畅——除了 Secrets 的处理。

Kubernetes 的 Secret **默认不加密**。`data` 字段只是做了 base64 编码，而不是加密。任何能访问集群或 YAML 文件的人都可以解码：

```bash
echo "cHJvZHVjdGlvbl9zdXBlcl9zZWNyZXRfcGFzc3dvcmQ=" | base64 -d
# 输出: production_super_secret_password
```

尽管团队知道这一点，但为了"速度"，他们一直将 Secrets 以原始 YAML 形式提交——反正仓库是私有的。与此同时，一名初级工程师正在入职，需要部署一个新数据库。他遵循了现有模式：创建 Secret YAML，提交，推送。但三天前的一次设置迁移意外将仓库可见性改成了 `public`。

---

## 排查过程

### 第一步：确定暴露范围

GitGuardian 告警列出了 12 个 YAML 文件中的 47 个密钥：

| 服务 | 密钥类型 | 暴露时间 |
|------|---------|---------|
| 生产 PostgreSQL | 密码 | 72 小时 |
| 生产 Redis | 认证 Token | 72 小时 |
| AWS S3 密钥 | Access Key + Secret | 72 小时 |
| Stripe API 密钥 | Live Key | 72 小时 |
| GitHub Deploy Key | SSH 私钥 | 72 小时 |
| TLS 证书 | 私钥 + 证书 | 72 小时 |

### 第二步：检查 Git 历史中的未授权克隆

```bash
# GitHub API：检查公开期间的克隆流量
gh api repos/hongrentang/myblog/traffic/clones
```

遗憾的是，GitHub 只保留 14 天的流量数据，而且在这 72 小时内任何人都可以匿名克隆该仓库。

### 第三步：检查集群是否已被入侵

```bash
# 检查异常 Pod 或 Deployment
kubectl get pods --all-namespaces | grep -iv "Running\|Completed"

# 检查审计日志中是否有异常访问模式
grep -E "secret|credential|password" /var/log/kubernetes/audit.log | grep -v "system:serviceaccount:kube-system" | head -20
```

没有发现集群被入侵的直接迹象——但凭据已暴露，必须假定所有密钥均已泄露。

---

## 根因

1. **Git 中明文存储 Secrets**：Kubernetes Secret 在仓库中没有加密保护
2. **仓库意外公开**：设置迁移更改了仓库可见性
3. **无提交前扫描**：没有使用 `git-secrets`、`truffleHog` 或 pre-commit hooks 阻止密钥提交
4. **未采用外部密钥管理方案**：团队知道 Sealed Secrets 和 External Secrets Operator，但一直未采用
5. **无自动轮换策略**：即使立刻发现，也没有紧急凭证轮换的 Runbook

---

## 解决方案

### 紧急凭证轮换

```bash
# 1. 撤销暴露的 GitHub Deploy Key
gh repo deploy-key list
gh repo deploy-key delete <id>

# 2. 轮换数据库凭据
kubectl exec -it postgres-primary-0 -n production -- psql -c "ALTER USER prod_admin WITH PASSWORD 'new-secure-password';"

# 3. 轮换 Redis 认证 Token
kubectl exec -it redis-master-0 -n production -- redis-cli CONFIG SET requirepass "new-token"

# 4. 通过 IAM 控制台重新生成 AWS Access Key
aws iam create-access-key --user-name prod-s3-user
aws iam update-access-key --access-key-id OLD_KEY --status Inactive
aws iam delete-access-key --access-key-id OLD_KEY

# 5. 通过 Stripe 控制台轮换 API 密钥
# 6. 通过 cert-manager 重新签发 TLS 证书
kubectl delete certificate -n production -l app=api-gateway
```

### 从 Git 历史中清除密钥

```bash
# 使用 git-filter-repo 清除所有历史中的密钥
pip install git-filter-repo

echo "infrastructure/secrets/prod-db-credentials.yaml" > /tmp/secret_files.txt
git filter-repo --paths-from-file /tmp/secret_files.txt --force
git push --force --all
```

**注意**：`git filter-repo` 会重写历史，所有协作者需要重新克隆。

### 永久修复：External Secrets Operator

团队实施了 External Secrets Operator（ESO），从 AWS Secrets Manager 拉取密钥：

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

### 预防措施

1. **Pre-commit hooks**：阻止任何包含 `kind: Secret` 带 base64 编码数据的提交

```bash
#!/bin/bash
if git diff --cached | grep -A5 "kind: Secret" | grep -q "data:"; then
  echo "错误：检测到提交中包含明文 Secret！"
  echo "请使用 Sealed Secrets 或 External Secrets 替代。"
  exit 1
fi
```

2. **GitHub 密钥扫描**：启用推送保护
3. **仓库可见性监控**：设置仓库可见性变更告警
4. **自动化凭证轮换**：使用带自动轮换计划的 Secrets Manager
5. **Sealed Secrets**：使用 `kubeseal` 加密 Secret，安全存入 Git

```bash
kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets \
  < prod-db-credentials.yaml > prod-db-credentials-sealed.yaml
# 只有加密后的 YAML 提交到 Git
```

---

## 经验教训

- **`base64` 不是加密**：Base64 编码是混淆，不是安全。任何 base64 编码的 Secret 都应视为已泄露
- **进了 Git 就是永远**：即使删除了文件，它仍存在于 Git 历史中。使用 `git-filter-repo` 或者从一开始就不要提交 Secrets
- **External Secrets Operator 是正确的模式**：密钥应存放在专用密钥管理器中（AWS Secrets Manager、HashiCorp Vault、GCP Secret Manager），而不是 Git
- **仓库可见性是安全边界**：私有仓库可能意外变成公开。把每次对任何仓库的提交都当作公开的来对待
- **Pre-commit hooks 及早发现错误**：一个简单的 pre-commit hook 就能拦截实习生的提交，避免整起事件

---

## 总结

事件链路：

```
实习生需要部署数据库
→ 遵循现有模式：创建明文 Secret YAML
→ 提交到被意外设为公开的仓库
→ 47 个生产密钥在公开 GitHub 上暴露 72 小时
→ GitGuardian 扫描器触发告警
→ 6 个服务的紧急凭证轮换
→ 使用 git-filter-repo 重写历史
→ 采用 External Secrets Operator 作为永久修复
```

全局凭证轮换：4 小时。流程修复：2 天。使用 `echo "password" | base64` 而不是 `kubeseal` 或 ESO 的代价：不可估量。不要再把 Secrets 放进 Git——从昨天开始。
