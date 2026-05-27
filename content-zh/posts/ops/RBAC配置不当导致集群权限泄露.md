---
title: "RBAC 配置不当导致集群权限泄露——一个只读 ServiceAccount 如何拿到集群管理员权限"
date: 2026-05-27
weight: 100190
slug: "rbac-misconfiguration-cluster-privilege-escalation"
tags: ["kubernetes", "security", "rbac", "troubleshooting"]
categories: ["Security"]
description: "Kubernetes RBAC 配置不当事件复盘——一个本该只拥有只读权限的监控 ServiceAccount，因配置失误被赋予了 cluster-admin，最终导致 15 个 Pod 被误删"
keywords: "kubernetes rbac 配置不当, k8s serviceaccount 权限过大, clusterrolebinding 安全, kubernetes 权限提升, kubectl auth can-i"
draft: false
featured: true
cover:
  image: ""
  caption: "RBAC 配置不当——权限泄露排查"
---

# RBAC 配置不当导致集群权限泄露——一个只读 ServiceAccount 如何拿到集群管理员权限

## 常见搜索词

- kubernetes rbac 配置不当案例
- k8s serviceaccount 权限过大怎么排查
- clusterrolebinding 安全事件
- 如何审计 kubernetes rbac 权限
- kubectl auth can-i 检查权限

---

## 故障经过

**环境**: K8S v1.27, kubeadm 部署, 3 Master + 7 Worker, 200+ 命名空间。

**时间**: 14:30，业务高峰期。

**症状**: 监控系统使用的 ServiceAccount（`monitoring/metrics-collector`）居然删除了 15 个生产 Pod，分布在 5 个 namespace 中。监控管道瘫痪，连带数个关键业务中断。

```bash
kubectl get events --all-namespaces --field-selector reason=Killing
NAMESPACE      LAST SEEN   TYPE    REASON   OBJECT                  MESSAGE
api-prod       14:29:53    Normal  Killing  pod/api-v2-7d9f8c6b-*   Stopping container api-server
api-prod       14:29:54    Normal  Killing  pod/api-v2-7d9f8c6b-*   Stopping container sidecar
cache-prod     14:29:55    Normal  Killing  pod/redis-sentinel-*    Stopping container redis
...
```

**影响**: 15 个 Pod 被删，6 分钟部分服务降级。

---

## 背景

三周前，团队部署了一套新的监控系统。DevOps 工程师需要一个能列出 Pod 和读取指标的 ServiceAccount。他创建了一个 Role：

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: monitoring
  name: metrics-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
```

然后他创建了 ClusterRoleBinding 来绑定这个 Role……但出了差错：

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring-metrics-reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: metrics-reader    # ← 找不到名为 "metrics-reader" 的 ClusterRole
subjects:
- kind: ServiceAccount
  name: metrics-collector
  namespace: monitoring
```

问题在于：`metrics-reader` 被定义成了 **Role**（命名空间级别），而不是 **ClusterRole**（集群级别）。这个 ClusterRoleBinding 引用的 `metrics-reader` ClusterRole 并不存在。但 Kubernetes 在创建绑定时**不会验证 RoleRef 是否存在**，只是保存了引用。

接下来的操作让问题升级：工程师发现 ServiceAccount 没有权限列出 Pod，于是想"干脆用内置的只读 ClusterRole 算了"。他找到了 `view`——一个内置的只读 ClusterRole。

但在编辑 YAML 的过程中，他犯了两个错误：

1. 为了测试是否奏效，他临时把 `roleRef` 改成了 `cluster-admin`，然后忘了改回来
2. 旧的 `metrics-reader` Role 也从未清理

最终生效的绑定：

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring-metrics-reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin    # ← 灾难
subjects:
- kind: ServiceAccount
  name: metrics-collector
  namespace: monitoring
```

这个监控 ServiceAccount 从此拥有了整个集群的 **cluster-admin** 权限。

---

## 排查过程

### 第一步：确认谁干的

```bash
# 在审计日志中搜索 Pod 删除事件
grep "metrics-collector" /var/log/kubernetes/audit.log | grep "delete" | head -5
```

```
{"kind":"Event","level":"RequestResponse","user":{"username":"system:serviceaccount:monitoring:metrics-collector",...},"verb":"delete","objectRef":{"resource":"pods","namespace":"api-prod","name":"api-v2-7d9f8c6b-abc"}}
```

确认为 ServiceAccount `monitoring/metrics-collector` 所为。

### 第二步：查看 ServiceAccount 的实际权限

```bash
kubectl auth can-i --list --as=system:serviceaccount:monitoring:metrics-collector
```

```
Resources                                       Non-Resource URLs   Verbs
*.*                                             []                  [*]
[*]                                             []                  [*]
```

通配符权限，什么都能做。至此确认：有人把 `cluster-admin` 绑给了一个监控账号。

### 第三步：定位绑定

```bash
kubectl get clusterrolebinding monitoring-metrics-reader -o yaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-metrics-reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin    # ← 根源在这里
subjects:
- kind: ServiceAccount
  name: metrics-collector
  namespace: monitoring
```

### 第四步：回溯变更记录

```bash
# 审计日志中查找谁创建/修改了这个绑定
grep "monitoring-metrics-reader" /var/log/kubernetes/audit.log | head -3
```

```
...
"user":{"username":"devops-user"},"verb":"create","objectRef":{"resource":"clusterrolebindings","name":"monitoring-metrics-reader"},"responseStatus":{"code":201}}
...
"user":{"username":"devops-user"},"verb":"update","objectRef":{"resource":"clusterrolebindings","name":"monitoring-metrics-reader"},"responseStatus":{"code":200}}
```

绑定从一开始就是 `cluster-admin`——不是被攻击者利用，而是配置阶段的人为失误。

---

## 根因

DevOps 工程师在编辑 YAML 时临时把角色改为 `cluster-admin` 做测试，改完忘了恢复。监控系统在时间压力下仓促上线，问题未被发现，因为：

- RBAC 清单在应用前没有人做代码审查
- 没有自动化 RBAC 策略扫描
- 监控系统正常工作，没人深究它的权限
- 没有对高权限绑定的创建设置告警

---

## 解决方案

### 紧急修复

```bash
# 删除错误的 ClusterRoleBinding
kubectl delete clusterrolebinding monitoring-metrics-reader

# 创建正确的绑定（使用 view 角色）
cat <<EOF | kubectl apply -f -
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring-metrics-reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
- kind: ServiceAccount
  name: metrics-collector
  namespace: monitoring
EOF
```

### 全面审计所有绑定

```bash
# 列出所有 ClusterRoleBinding 及其角色
kubectl get clusterrolebinding -o custom-columns='NAME:.metadata.name,ROLE:.roleRef.name,KIND:.roleRef.kind,SUBJECTS:.subjects[*].name' | sort
```

检查哪些 ServiceAccount 绑定了 `cluster-admin`：

```bash
kubectl get clusterrolebinding -o json | jq -r '
  .items[] | select(.roleRef.name == "cluster-admin") |
  "\(.metadata.name) → \(.subjects[]?.kind)/\(.subjects[]?.name) [\(.subjects[]?.namespace // "cluster")]"
'
```

### 长期预防措施

1. **最小权限原则**：永远不要从 `cluster-admin` 开始缩减——从最小权限开始，按需增加
2. **专用 Role**：不要为特定需求复用共享的 ClusterRole（`admin`、`edit`、`view`），创建自定义 Role
3. **CI 中审查 RBAC**：使用 `kube-linter`、`checkov`、`kube-bench` 等工具扫描越权绑定
4. **告警规则**：为高权限 ClusterRoleBinding 的创建设置告警

```yaml
# Prometheus 规则示例
groups:
- name: rbac-security
  rules:
  - alert: HighPrivilegeBindingCreated
    expr: count(kube_clusterrolebinding_info{role="cluster-admin"}) > 3
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "检测到新的 cluster-admin 绑定"
```

---

## 经验教训

- **`roleRef` 区分大小写和类型**：绑定 `Role` 还是 `ClusterRole` 完全不同。类型不对会静默失败，导致"先让它能跑再说"式的权限滥用
- **`cluster-admin` 永远不是临时测试选项**：一旦加上就很容易忘记移除。始终在专用测试命名空间中使用受限 Role
- **监控谁能做什么**：`kubectl auth can-i --list --as=<user>` 显示有效权限。每次部署完关键 ServiceAccount 后都跑一遍
- **RBAC 不会在创建绑定时验证 RoleRef**：你可以绑定一个不存在的 ClusterRole，它静默不授予任何权限——直到有人创建了那个 ClusterRole

---

## 总结

排查链路：

```
监控管道报权限不足
→ DevOps 工程师临时用 cluster-admin 测试
→ 改完 YAML 忘记恢复就 apply 了
→ ServiceAccount 获得集群管理员权限
→ 没有代码审查，没有 RBAC 扫描发现
→ 几周后监控容器中的错误脚本执行了 kubectl delete pods
→ 15 个 Pod 被删，6 分钟部分服务中断
```

修复耗时：15 分钟。越权绑定存在时间：3 周。修复本身很简单——代价是一次本不该发生的生产事故。

审计你的 RBAC。在事故发生之前，而不是之后。
