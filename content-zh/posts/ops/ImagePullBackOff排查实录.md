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

# ImagePullBackOff 排查实录

## 问题引导

如果你是通过搜索找到这里，这篇文章覆盖以下场景：

- k8s imagepullbackoff 解决
- pod 拉取镜像失败 ErrImagePull
- kubernetes 镜像仓库鉴权失败
- imagePullSecrets 配置错误
- docker registry 限流导致部署失败

---

## 故障现象

**环境**：K8S v1.28，containerd 1.7，Harbor 私有镜像仓库，集群版本定期升级。

**时间**：周四 10:00，上线窗口。

**症状**：新版本发布流水线执行完毕后，监控显示所有新 Pod 处于 `ImagePullBackOff` 状态。

```bash
NAMESPACE    NAME                              READY   STATUS              RESTARTS   AGE
production   order-service-v2-7d8f9d4c5f-a1b2  0/1     ImagePullBackOff    0          3m
production   order-service-v2-7d8f9d4c5f-c3d4  0/1     ImagePullBackOff    0          3m
production   order-service-v2-7d8f9d4c5f-e5f6  0/1     ImagePullBackOff    0          3m
```

**影响**：新版本无法上线，回滚到旧版本后恢复。但问题是——旧版本的镜像明明还在仓库里，为什么新版本拉不到？

---

## 误判阶段

### 第一反应：镜像标签不存在

看到 `ImagePullBackOff`，我的第一反应是 CI 流水线没有正确推送镜像。我登录 Harbor 查看，镜像确实存在，标签也正确。

### 接着怀疑镜像拉取策略

检查 Deployment 配置，确认 `imagePullPolicy` 是 `IfNotPresent`——按理说节点上如果有同名镜像会用缓存的，但问题依旧。

### 尝试手动拉取

```bash
kubectl describe pod order-service-v2-7d8f9d4c5f-a1b2 -n production
```

Events 输出：

```
Events:
  Type     Reason          Age   From               Message
  ----     ------          ----  ----               -------
  Normal   Scheduled       4m    default-scheduler  Successfully assigned...
  Normal   Pulling         4m    kubelet            Pulling image "harbor.internal/library/order-service:v2.1.0"
  Warning  Failed          4m    kubelet            Failed to pull image "harbor.internal/library/order-service:v2.1.0": rpc error: code = Unknown desc = failed to pull and unpack image: unauthorized: authentication required
  Warning  Failed          4m    kubelet            Error: ErrImagePull
  Warning  Failed          3m    kubelet            Error: ImagePullBackOff
```

关键信息：**unauthorized: authentication required**——不是镜像不存在，而是没有权限拉取。

---

## 排查过程

### Step 1: 检查 imagePullSecrets

```bash
kubectl get pod order-service-v2-7d8f9d4c5f-a1b2 -n production -o yaml | grep imagePullSecrets -A 5
```

```
imagePullSecrets:
- name: harbor-registry-cred
```

Secret 配置了，但内容是什么？

```bash
kubectl get secret harbor-registry-cred -n production -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d
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

解码 `auth` 字段：`echo 'YWRtaW46SGFyYm9yMTIzNDU=' | base64 -d` → `admin:Harbor12345`

密码是 `Harbor12345`——这是 Harbor 安装后的默认密码。但这个密码在三个月前就被管理员轮转过了。

### Step 2: 验证仓库连通性

```bash
# 在集群节点上测试
curl -I https://harbor.internal/v2/
```

```
HTTP/2 401 
```

虽然有 401，但说明网络层是通的——节点能访问 Harbor。问题只在于凭证。

### Step 3: 追溯 Secret 来源

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: harbor-registry-cred
  namespace: production
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64>
```

查看 Git 仓库中该 Secret 的原始定义——发现是三个月前初始化时创建的，之后从未更新。期间 Harbor 密码被轮转，但 K8S Secret 没有同步更新。

---

## 根因

1. 安全管理策略要求每季度轮转 Harbor 管理员密码
2. 运维团队在 Harbor 端完成了密码更改，但**没有更新 K8S 中对应的 imagePullSecrets**
3. 旧版本的 Pod 因为节点上已有缓存镜像，仍然正常运行
4. 新版本发布时，节点上无缓存，需要从 Harbor 拉取，立刻暴露凭证失效问题

**最大误判**：前 5 分钟我一直盯着 CI 流水线日志，认为是构建阶段没有推镜像。实际上镜像在仓库里完好无损，问题是集群拉不下来。

---

## 恢复过程

### 临时恢复（2 分钟）

更新 Secret 为新的 Harbor 密码：

```bash
kubectl create secret docker-registry harbor-registry-cred \
  --namespace=production \
  --docker-server=harbor.internal \
  --docker-username=admin \
  --docker-password='NewHarbor!Pass2024' \
  --dry-run=client -o yaml | kubectl apply -f -
```

删除旧 Pod 触发重建：

```bash
kubectl delete pod -n production -l app=order-service --force
```

新 Pod 成功拉取镜像，服务恢复。

---

## 长期修复

### 1. 使用独立机器人账户

不要用 admin 账户拉取镜像，创建只读机器人账户：

```bash
kubectl create secret docker-registry harbor-registry-cred \
  --namespace=production \
  --docker-server=harbor.internal \
  --docker-username=robot$puller \
  --docker-password='<robot-token>'
```

### 2. Secret 自动同步

通过 External Secrets Operator 从 Vault 自动同步：

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: harbor-registry-cred
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: harbor-registry-cred
    creationPolicy: Owner
  data:
  - secretKey: .dockerconfigjson
    remoteRef:
      key: secrets/harbor
      property: dockerconfigjson
```

### 3. 部署前镜像拉取预检查

在 CI/CD 中增加预检查步骤：

```bash
# 在发布前验证凭证有效性
kubectl run image-pull-test --image=harbor.internal/library/order-service:v2.1.0 \
  --restart=Never --image-pull-policy=Always --dry-run=client -o yaml | kubectl apply -f -

sleep 10
STATUS=$(kubectl get pod image-pull-test -o jsonpath='{.status.phase}')
if [ "$STATUS" != "Running" ]; then
  echo "镜像拉取验证失败！"
  kubectl describe pod image-pull-test
  exit 1
fi
```

### 4. 监控 ImagePullBackOff 事件

配置 Prometheus 告警规则：

```yaml
groups:
- name: image-pull-alerts
  rules:
  - alert: ImagePullBackOff
    expr: kube_pod_status_phase{phase="Pending"} > 0
    for: 2m
    annotations:
      summary: "Pod {{ $labels.pod }} 处于 ImagePullBackOff 状态"
```

---

## 复现步骤

1. 修改 Harbor 密码
2. 不更新 K8S Secret
3. 触发新 Pod 调度（新节点或删除旧 Pod）
4. 观察 ImagePullBackOff

---

## 排查命令速查

```bash
# 查看 Pod 事件
kubectl describe pod <pod> -n <namespace>

# 查看镜像拉取凭证
kubectl get secret <secret> -n <namespace> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d

# 测试镜像拉取
kubectl run test-pod --image=<image> --restart=Never

# 更新镜像仓库凭证
kubectl create secret docker-registry <name> \
  --docker-server=<server> \
  --docker-username=<user> \
  --docker-password=<pass> \
  --dry-run=client -o yaml | kubectl apply -f -
```

---

## 总结

排查链路：

```
新版本发布失败
  → Pod ImagePullBackOff
  → 误判 CI 未推送镜像
  → 检查 Harbor（镜像存在）
  → describe Pod（unauthorized）
  → 检查 Secret（凭证过期）
  → 更新 Secret → Pod 恢复
```

总耗时：从告警到恢复 8 分钟。

**教训**：镜像凭证过期是一个很容易被忽略的「定时炸弹」。基础设施的凭据轮转必须与 K8S 资源同步，否则下次发布就是灾难。建议通过 External Secrets Operator 自动管理，减少人为遗漏。
