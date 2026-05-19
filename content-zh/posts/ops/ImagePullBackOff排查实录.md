---
title: "ImagePullBackOff 排查实录：镜像仓库鉴权失效导致服务部署失败"
date: 2026-05-19
weight: 100030
slug: "k8s-imagepullbackoff-registry-auth"
tags: ["kubernetes", "troubleshooting"]
categories: ["K8S"]
description: "K8S ImagePullBackOff 解决全过程，从误判镜像标签到找到镜像仓库凭证过期问题"
keywords: "k8s imagepullbackoff解决,pod拉取镜像失败,ErrImagePull,镜像仓库鉴权,kubernetes拉取镜像失败"
draft: false
featured: true
cover:
  image: "/images/imagepullbackoff-banner.svg"
  caption: "ImagePullBackOff 排查与诊断"
---

# ImagePullBackOff 排查实录：镜像仓库鉴权失效导致服务部署失败

## 问题引导

如果你是通过搜索找到这里，这篇文章覆盖以下场景：

- k8s imagepullbackoff 解决
- pod 拉取镜像失败 ErrImagePull
- kubernetes 镜像仓库鉴权失败
- imagePullSecrets 配置错误
- docker registry 限流导致部署失败
- containerd 拉取镜像失败排查
- Harbor 密码轮转后 K8S 拉取异常

---

## 故障现象

**环境**：K8S v1.28.5，containerd 1.7.8，Calico CNI v3.26，Harbor v2.8 私有镜像仓库，集群共 12 个节点（3 Master + 9 Worker），托管 200+ 微服务。

**时间**：周四 10:00，例行上线窗口。前一天刚完成了 Harbor 的季度密码轮转。

**症状**：新版本发布流水线执行完毕后，监控告警——订单服务新版 Pod 全部处于 `ImagePullBackOff` 状态。同时收到开发反馈，QA 环境无法部署最新构建。

```bash
NAMESPACE    NAME                              READY   STATUS              RESTARTS   AGE
production   order-service-v2-7d8f9d4c5f-a1b2  0/1     ImagePullBackOff    0          3m
production   order-service-v2-7d8f9d4c5f-c3d4  0/1     ImagePullBackOff    0          3m
production   order-service-v2-7d8f9d4c5f-e5f6  0/1     ImagePullBackOff    0          3m
staging      order-service-v2-6a8b7c9d0e-f1g2  0/1     ImagePullBackOff    0          5m
```

**影响**：新版本无法上线。切换到旧版本后服务恢复，但新功能无法交付。QA 环境阻塞，影响整个迭代的测试进度。线上旧版本仍能正常运行，说明问题仅限于**新镜像的拉取**。

---

## 误判阶段

### 第一反应：CI 流水线没有推送镜像

看到 `ImagePullBackOff`，最自然的想法就是"镜像没推上去"。我立刻做了三件事：

```bash
# 1. 检查 Harbor 项目仓库
curl -u admin:password "https://harbor.internal/api/v2.0/projects/library/repositories/order-service/artifacts?v=2.1.0" | jq .

# 2. 直接通过 Docker CLI 验证
docker pull harbor.internal/library/order-service:v2.1.0

# 3. 检查 CI 日志
# 在 Jenkins 中搜索 "push" 和 "success"
```

结果：镜像确实存在，tag 正确，CI 日志显示推送成功。Harbor UI 上也看得到，层（layer）大小正常，digest 完整。

### 接着怀疑镜像拉取策略

翻出 Deployment YAML 检查镜像拉取相关配置：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-v2
  namespace: production
spec:
  template:
    spec:
      containers:
      - name: order-service
        image: harbor.internal/library/order-service:v2.1.0
        imagePullPolicy: IfNotPresent
      imagePullSecrets:
      - name: harbor-registry-cred
```

`imagePullPolicy: IfNotPresent` 意味着如果节点上已经有同名镜像，kubelet 会直接用缓存的——但新版本的镜像 tag 是新的，节点上自然没有缓存。所以这条策略并不会帮我们跳过拉取。

### 甚至怀疑过 containerd 配置

由于集群使用 containerd 而不是 Docker，我一度怀疑是 containerd 的 registry mirror 配置有问题：

```bash
# 检查 containerd 配置
cat /etc/containerd/config.toml | grep -A 10 "harbor"
```

但里面根本没有 harbor 的 mirror 配置——containerd 会直接走 HTTPS 连接 harbor，这条路没有问题。

---

## 排查过程

### Step 1: 查看 Pod 详细事件

`describe pod` 是所有 Pod 问题排查的第一步。这比看日志更快，因为 Events 字段直接告诉 kubelet 视角发生了什么。

```bash
kubectl describe pod order-service-v2-7d8f9d4c5f-a1b2 -n production
```

完整输出（关键部分）：

```
Name:             order-service-v2-7d8f9d4c5f-a1b2
Namespace:        production
Priority:         0
Service Account:  order-service
Node:             worker-04/10.0.1.44
Labels:           app=order-service
                  version=v2.1.0
Status:           Pending
IP:               
IPs:              <none>

Conditions:
  Type           Status
  PodScheduled   True 
  Initialized    False
  Ready          False
  ContainersReady False

Containers:
  order-service:
    Container ID:   
    Image:          harbor.internal/library/order-service:v2.1.0
    Image ID:       
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0

Events:
  Type     Reason          Age   From               Message
  ----     ------          ----  ----               -------
  Normal   Scheduled       4m    default-scheduler  Successfully assigned production/order-service-v2-7d8f9d4c5f-a1b2 to worker-04
  Normal   Pulling         3m    kubelet            Pulling image "harbor.internal/library/order-service:v2.1.0"
  Warning  Failed          3m    kubelet            Failed to pull image "harbor.internal/library/order-service:v2.1.0": rpc error: code = Unknown desc = failed to pull and unpack image: unauthorized: authentication required
  Warning  Failed          3m    kubelet            Error: ErrImagePull
  Warning  Failed          2m    kubelet            Error: ImagePullBackOff
  Normal   BackOff         2m    kubelet            Back-off pulling image "harbor.internal/library/order-service:v2.1.0"
```

关键信息非常清晰：**unauthorized: authentication required**。这不是镜像不存在，也不是网络不通——是**身份认证失败**。

注意 Events 中的时间线：
1. `Scheduled` — Pod 被分配到 worker-04
2. `Pulling` — kubelet 尝试拉取镜像
3. `Failed` → `ErrImagePull` → `ImagePullBackOff` — 失败后进入退避
4. `BackOff` — kubelet 进入指数退避，重试间隔会从 5s 增加到 5min

### Step 2: 检查 imagePullSecrets

Pod 的 `imagePullSecrets` 字段指定了拉取镜像时使用的凭证。我们先确认 Pod 确实配置了：

```bash
kubectl get pod order-service-v2-7d8f9d4c5f-a1b2 -n production -o yaml | grep -A 5 "imagePullSecrets"
```

```
imagePullSecrets:
- name: harbor-registry-cred
```

Secret 名称是 `harbor-registry-cred`，看看它的实际内容：

```bash
kubectl get secret harbor-registry-cred -n production -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq .
```

```json
{
  "auths": {
    "harbor.internal": {
      "auth": "YWRtaW46SGFyYm9yMTIzNDU="
    }
  }
}
```

解码 `auth` 字段：

```bash
echo 'YWRtaW46SGFyYm9yMTIzNDU=' | base64 -d
# 输出: admin:Harbor12345
```

`Harbor12345`——这是 Harbor 安装完成后的**默认管理员密码**。这个密码在三个月前的安全审计中被要求轮转，运维团队已经在 Harbor 端改成了 `NewHarbor!Pass2024`，但 K8S 里的 Secret 还是旧的。

### Step 3: 验证 Harbor 连通性和凭证

为了彻底排除网络层面的问题，我在集群节点上做了更全面的测试：

```bash
# 验证 DNS 解析
nslookup harbor.internal

# 验证网络连通性（TCP）
time nc -zv harbor.internal 443

# 验证 HTTPS 证书
openssl s_client -connect harbor.internal:443 -servername harbor.internal < /dev/null 2>/dev/null | openssl x509 -noout -dates

# 验证旧凭证是否有效
echo -n 'admin:Harbor12345' | base64
curl -H "Authorization: Basic YWRtaW46SGFyYm9yMTIzNDU=" https://harbor.internal/v2/_catalog
```

输出：

```
;; ANSWER SECTION:
harbor.internal.  60  IN  A  10.0.1.200

Connection to harbor.internal port 443 succeeded!

notBefore=Mar 15 00:00:00 2026 GMT
notAfter=Jun 13 00:00:00 2026 GMT

{"errors":[{"code":"UNAUTHORIZED","message":"authentication required"}]}
```

DNS 正常、TCP 连通、证书有效、旧凭证被拒——问题确认在凭证上。

用新密码验证：

```bash
echo -n 'admin:NewHarbor!Pass2024' | base64
curl -H "Authorization: Basic $(echo -n 'admin:NewHarbor!Pass2024' | base64)" https://harbor.internal/v2/_catalog
```

成功返回仓库列表。确认新密码有效。

### Step 4: 追溯 Secret 生命周期

通过 Git 历史查找 Secret 的创建记录：

```bash
git log --all --oneline -- "*/harbor-registry-cred*"
git show <commit-hash>
```

```
commit a3f8c2d (initial-cluster-setup)
Date:   Mon Feb 10 14:30:00 2026

    Add initial Kubernetes manifests for production namespace
    
    - Includes harbor-registry-cred Secret
    - Default Harbor password used
```

Secret 是集群初始化时（3 个月前）使用默认密码创建的。期间 Harbor 完成了季度密码轮转，但 Secret 没有得到更新。运维 SOP 中缺少"密码轮转后同步 K8S Secret"这一步骤。

---

## 根因分析

1. **安全策略合规**：安全团队要求每季度轮转 Harbor 管理员密码，运维团队按计划执行了轮转
2. **流程断层**：Harbor 密码变更后，运维团队**没有更新对应的 K8S imagePullSecrets**——这不是技术问题，是流程问题
3. **缓存掩盖问题**：旧版本的 Pod（基于旧镜像版本）在节点上已有缓存，即便 Secret 过期也不会触发重新拉取，导致问题被隐藏了 3 个月
4. **新版本触发暴露**：新版本在 CI 中打了新的镜像标签，节点上没有缓存，kubelet 必须从 Harbor 拉取，立刻暴露凭证失效

**关键认知**：K8S 只在 Pod 调度时拉取镜像，且只在节点无缓存时才会实际执行拉取操作。所以"能跑"不代表"凭证有效"——这是一个很容易被忽视的陷阱。

**最大误判**：前 5 分钟我一直盯着 CI 流水线日志，反复确认构建和推送步骤是否成功。实际上镜像在 Harbor 里完好无损，问题出在集群拉取这一侧，完全不关 CI 的事。

---

## 恢复过程

### 临时恢复（2 分钟）

更新 Secret 使用新的 Harbor 密码。这里使用 `kubectl create secret --dry-run=client` 的技巧——不改 YAML，直接原地更新：

```bash
kubectl create secret docker-registry harbor-registry-cred \
  --namespace=production \
  --docker-server=harbor.internal \
  --docker-username=admin \
  --docker-password='NewHarbor!Pass2024' \
  --dry-run=client -o yaml | kubectl apply -f -
```

验证更新后的 Secret 内容：

```bash
kubectl get secret harbor-registry-cred -n production -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq .
```

```json
{
  "auths": {
    "harbor.internal": {
      "auth": "YWRtaW46TmV3SGFyYm9yIVBhc3MyMDI0"
    }
  }
}
```

强制重建 Pod：

```bash
kubectl delete pod -n production -l app=order-service --force
```

监控 Pod 状态变化：

```bash
kubectl get pods -n production -l app=order-service -w
```

```
NAME                              READY   STATUS              RESTARTS   AGE
order-service-v2-7d8f9d4c5f-x1y2  0/1     Init:ErrImagePull   0          0s
order-service-v2-7d8f9d4c5f-x1y2  0/1     PodInitializing     0          8s
order-service-v2-7d8f9d4c5f-x1y2  1/1     Running             0          12s
```

Pod 成功拉取镜像并进入 Running 状态，前后约 2 分钟。生产环境恢复正常。

### 回滚兼容性处理

如果热修复过程中出现问题，保留旧版本 Deployment 作为回滚方案：

```bash
kubectl rollout undo deployment/order-service-v2 -n production
```

---

## 长期修复方案

### 1. 使用机器人账户替代管理员账户

用 admin 账号拉取镜像是最佳实践的反面教材——权限过大（可以删除镜像、修改项目配置），且一个团队多人共用，密码轮转影响面大。

Harbor 支持机器人账户（Robot Account），可以限制到特定项目、只读权限、长期 token：

```bash
# 在 Harbor UI 中创建机器人账户
# Project → Robots → Add Robot Account
# 名称: robot-order-service-puller
# 权限: pull-only
# 过期时间: never

# 在 K8S 中创建对应 Secret
kubectl create secret docker-registry harbor-registry-cred \
  --namespace=production \
  --docker-server=harbor.internal \
  --docker-username=robot$order-service-puller \
  --docker-password='<harbor-generated-token>' \
  --dry-run=client -o yaml | kubectl apply -f -
```

机器人账户的好处：
- 细粒度权限控制（只读、限定项目）
- token 独立于用户密码，密码轮转不影响
- 可以在 Harbor 审计日志中追踪到具体是哪个服务在拉取

### 2. Secret 自动同步（External Secrets Operator）

手动管理 Secret 不可靠——人总会忘记。通过 External Secrets Operator 从 Vault 自动同步：

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: production
spec:
  provider:
    vault:
      server: "https://vault.internal:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "v1/auth/kubernetes"
          role: "external-secrets"
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: harbor-registry-cred
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: harbor-registry-cred
    creationPolicy: Owner
    deletionPolicy: Delete
  data:
  - secretKey: .dockerconfigjson
    remoteRef:
      key: secrets/harbor
      property: dockerconfigjson
```

当 Harbor 密码轮转时，只需要更新 Vault 中的一个值，ESO 会在 1 小时内自动同步到 K8S Secret。

### 3. CI/CD 流水线预检查

在发布流水线中加入镜像拉取验证步骤，提前发现问题而不是等 Pod 启动失败：

```yaml
# GitLab CI 示例
image-pull-check:
  stage: verify
  script:
    - |
      kubectl run image-pull-test \
        --image=$CI_REGISTRY_IMAGE:$CI_COMMIT_TAG \
        --restart=Never \
        --image-pull-policy=Always \
        --dry-run=client -o yaml | kubectl apply -f -
      
      # 等待最多 30 秒
      for i in $(seq 1 30); do
        STATUS=$(kubectl get pod image-pull-test -o jsonpath='{.status.phase}')
        if [ "$STATUS" = "Running" ]; then
          echo "✓ Image pull successful"
          kubectl delete pod image-pull-test --force --wait=false
          exit 0
        fi
        sleep 1
      done
      
      # 超时，打印详细信息
      echo "✗ Image pull failed!"
      kubectl describe pod image-pull-test
      kubectl delete pod image-pull-test --force --wait=false
      exit 1
```

### 4. 主动监控

配置 Prometheus 告警规则，对 ImagePullBackOff 状态进行主动发现：

```yaml
groups:
- name: kubernetes-pods
  rules:
  - alert: ImagePullBackOff
    expr: kube_pod_status_phase{phase="Pending"} > 0
    for: 3m
    labels:
      severity: critical
    annotations:
      summary: "Pod {{ $labels.pod }} in {{ $labels.namespace }} stuck in ImagePullBackOff"
      description: "Pod has been unable to pull image for over 3 minutes. Check imagePullSecrets and registry accessibility."
      runbook: "https://internal.runbooks/imagepullbackoff"

  - alert: RegistryAuthExpiring
    expr: time() - secret_created_timestamp{type="kubernetes.io/dockerconfigjson"} > 86400 * 80
    for: 1h
    labels:
      severity: warning
    annotations:
      summary: "Docker config secret {{ $labels.secret }} is 80+ days old"
      description: "Registry credentials may expire soon. Verify and rotate if needed."
```

### 5. 将 Secret 版本化并纳入 GitOps

将 imagePullSecrets 的基线版本存在 Git 中，通过 ArgoCD 或 Flux CD 管理，变更需要通过 Merge Request：

```yaml
# gitops/infrastructure/harbor-registry-cred.yaml
apiVersion: v1
kind: Secret
metadata:
  name: harbor-registry-cred
  namespace: production
  annotations:
    sealedsecrets.bitnami.com/managed: "true"
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <encrypted-by-sealed-secrets>
```

### 6. 定期凭证轮转检查

通过脚本定期验证凭证有效性：

```bash
#!/bin/bash
# check-registry-creds.sh
# 定期检查所有 namespace 的镜像拉取凭证

for ns in $(kubectl get ns -o name | cut -d/ -f2); do
  for secret in $(kubectl get secret -n $ns --field-selector type=kubernetes.io/dockerconfigjson -o name 2>/dev/null); do
    echo "检查 $secret in $ns"
    # 提取 registry 地址和凭证
    REGISTRY=$(kubectl get $secret -n $ns -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq -r '.auths | keys[0]')
    AUTH=$(kubectl get $secret -n $ns -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq -r '.auths[].auth')
    
    # 测试凭证有效性
    HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Basic $AUTH" https://$REGISTRY/v2/)
    if [ "$HTTP_CODE" = "401" ]; then
      echo "WARNING: Secret $secret in $ns 凭证无效！"
    fi
  done
done
```

---

## 复现步骤

如果你想在测试环境复现这个场景：

1. 在 Harbor 中修改管理员密码
2. 不更新 K8S 中的 imagePullSecrets
3. 在一个没有缓存镜像的节点上调度新 Pod：
   ```bash
   kubectl cordon <node-with-cache>
   kubectl delete pod <existing-pod>
   ```
   或者直接在新的节点上部署（比如扩容新的 Node Group）
4. 观察 ImagePullBackOff

---

## 排查命令速查

```bash
# 查看 Pod 状态和事件（第一步）
kubectl describe pod <pod> -n <namespace>

# 检查 Pod 声明的 imagePullSecrets
kubectl get pod <pod> -n <namespace> -o yaml | grep -A 5 "imagePullSecrets"

# 解码镜像仓库凭证
kubectl get secret <secret> -n <namespace> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq .

# 验证凭证是否有效
AUTH=$(kubectl get secret <secret> -n <namespace> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq -r '.auths[].auth')
curl -H "Authorization: Basic $AUTH" https://<registry>/v2/_catalog

# 使用新凭证更新 Secret
kubectl create secret docker-registry <name> \
  --docker-server=<registry> \
  --docker-username=<user> \
  --docker-password=<pass> \
  --dry-run=client -o yaml | kubectl apply -f -

# 测试镜像拉取
kubectl run test-pod --image=<image> --restart=Never --image-pull-policy=Always

# 查看所有 dockerconfigjson 类型的 Secret
kubectl get secret --all-namespaces --field-selector type=kubernetes.io/dockerconfigjson

# 检查节点镜像缓存
crictl images | grep <image-name>
```

---

## 延伸：ImagePullBackOff 的常见原因

除了凭证过期，ImagePullBackOff 还可能由以下原因导致：

| 原因 | 特征 | 排查方向 |
|------|------|----------|
| 凭证过期 | `unauthorized` | 检查 Secret |
| 镜像不存在 | `not found` / `manifest unknown` | 检查 CI 和仓库 |
| 仓库限流 | `rate limit` / `too many requests` | 检查 registry 配置 |
| 磁盘空间满 | `no space left on device` | 检查节点磁盘 |
| containerd 异常 | `failed to unpack` | 检查 containerd 日志 |
| 网络策略阻断 | `i/o timeout` / `connection refused` | 检查网络策略 |
| 镜像格式不支持 | `unsupported manifest format` | 检查镜像架构 |

---

## 总结

排查链路：

```
新版本发布失败
  → Pod ImagePullBackOff
  → 误判 CI 未推送镜像（检查 Harbor，镜像存在）
  → describe Pod 发现 unauthorized
  → 检查 Secret 内容（admin:Harbor12345 默认密码）
  → Harbor 密码已轮转但 Secret 未更新
  → 更新 Secret → 重建 Pod → 恢复
```

总耗时：从告警到恢复 8 分钟。其中前 5 分钟浪费在错误的排查方向上。

**核心教训**：

1. **凭证过期是定时炸弹**：只要节点有缓存，Secret 过期不会产生任何症状。直到新镜像部署时才暴露
2. **不要用 admin 账户拉镜像**：机器人账户 + 细粒度权限是标准做法
3. **自动同步，不要靠手动**：External Secrets Operator / Sealed Secrets 等工具消除人为遗漏
4. **describe pod 是关键第一步**：Events 字段直接告诉你 kubelet 的视角，不要跳过这个命令

最后补充一点：`ImagePullBackOff` 的退避时间是指数增长的。第一次重试间隔 5s，然后 10s、20s、40s……直到 5min 上限。如果问题很紧急（比如阻断生产部署），手动重建 Pod 可以立即触发重试，而不是等待退避计时器。
