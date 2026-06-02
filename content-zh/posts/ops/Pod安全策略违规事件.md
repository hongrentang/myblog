---
title: "Pod 安全策略违规事件——当 PSA 的警告模式成为一纸空文"
date: 2026-06-02
weight: 100270
slug: "pod-security-policy-violation"
tags: ["kubernetes", "security", "pod-security", "psa", "troubleshooting"]
categories: ["Security"]
description: "Pod 安全标准违规事件复盘——因 PSA 仅配置为 warn 模式而非 enforce，12 个带有权限提升和宿主机网络的 Pod 在生产命名空间中运行了 6 个月未被发现"
keywords: "kubernetes pod security policy, psa 准入, 容器权限提升, allowPrivilegeEscalation, pod security standards 强制执行"
draft: false
featured: true
cover:
  image: ""
  caption: "Pod 安全策略违规——安全事件排查"
---

# Pod 安全策略违规事件——当 PSA 的警告模式成为一纸空文

## 常见搜索词

- kubernetes pod security admission 违规
- 容器权限提升 pod 安全
- allowPrivilegeEscalation true 安全风险
- pod security standards enforce vs warn
- kubernetes psa audit 模式绕过

---

## 故障经过

**环境**: K8S v1.28, 10 个集群, Pod Security Admission (PSA) 仅配置为 `warn`（警告）模式，未启用 `enforce`（强制执行）。

**时间**: 周四 09:00。安全审计报告标记：生产命名空间中存在以 `allowPrivilegeEscalation: true` 和宿主机网络访问权限运行的容器。

**发现过程**: 使用 `kube-bench` 和 `kubescape` 的季度合规扫描发现多个 Pod 违反 Pod 安全标准。

```bash
kubescape scan --enable-host-scan -n production
# 结果:
# ❌ AllowPrivilegeEscalation: 容器 "worker" 在 Pod "data-processor-7d9f8c6b-x" 中
# ❌ HostNetwork: 容器 "worker" 在 Pod "data-processor-7d9f8c6b-x" 中
# ❌ RunAsRoot: 容器 "worker" 以 UID 0 运行
```

**影响**: 3 个生产命名空间中的 12 个 Pod 以违反 Pod 安全标准的过度权限运行。未检测到主动利用——但这种暴露已持续 6 个月。

---

## 背景

当 Kubernetes 1.23 引入 Pod Security Admission (PSA) 时，团队知道需要采用它。他们配置了 PSA 的 `warn` 模式以避免破坏现有负载：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/warn: restricted       # 仅警告
    pod-security.kubernetes.io/audit: restricted       # 记录违规
    # 注意: 没有 "enforce" 标签——违规行为被允许
```

这意味着 PSA 会对违规行为发出**警告**，但**从不阻止**。这个命名空间的"restricted"配置有名无实。

6 个月来，多个数据处理 Pod 以不断升级的权限被部署。每次开发者都在 Pod 描述中看到 PSA 警告，但选择忽略：

```bash
# 创建 Pod 时的 PSA 警告
Warning: would violate PodSecurity "restricted:latest": privilege escalation
  (containers with "allowPrivilegeEscalation=true" or "privileged=true")
```

"只是警告。部署成功了。以后再说。"——开发者在工单中的评论，这个工单从未被处理。

---

## 排查过程

### 第一步：审计所有命名空间的 PSA 模式

```bash
kubectl get ns -o json | jq -r '
  .items[] | "\(.metadata.name): enforce=\(.metadata.annotations["pod-security.kubernetes.io/enforce"] // "none") warn=\(.metadata.annotations["pod-security.kubernetes.io/warn"] // "none")"'
```

```
production: enforce=none warn=restricted
staging: enforce=none warn=baseline
development: enforce=none warn=baseline
critical-prod: enforce=none warn=restricted
```

零个命名空间启用了 `enforce` 模式。PSA 在整个集群中仅是建议性的。

### 第二步：识别所有违规 Pod

12 个 Pod 违反 PSA restricted 配置：
- 6 个带 `allowPrivilegeEscalation: true`
- 3 个以 Root 运行（`runAsUser: 0`）
- 3 个带宿主机网络访问
- 2 个运行在特权模式

### 第三步：检查是否已被利用

审计日志中未发现主动利用迹象。这些是拥有过度权限的合法负载——安全债务，而非活跃入侵。

---

## 根因

1. **PSA 仅警告模式**：没有任何命名空间设置 `enforce` 标签。PSA 成了一块"请考虑安全"的牌子，而不是安全控制措施
2. **没有准入控制器阻止违规**：没有 OPA/Gatekeeper 或 Kyverno 策略来强制执行 Pod 安全标准
3. **安全警告被视为可选项**：开发者看到 PSA 警告并忽略。没有强制执行，警告就没有约束力
4. **无合规自动化**：违规行为存在 6 个月未被任何自动扫描发现
5. **生产环境中的特权负载**：带有宿主机访问和权限提升的数据处理负载不属于生产命名空间

---

## 解决方案

### 启用 PSA 强制执行

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  --overwrite
```

这会立即阻止任何违反 restricted 配置的新 Pod。现有违规 Pod 继续运行，但无法在不合规的情况下更新。

### 修复现有违规

对每个违规 Pod，与应用团队合作：
1. 移除 `allowPrivilegeEscalation: true`
2. 通过 `runAsNonRoot: true` 切换到非 Root 用户
3. 移除 `hostNetwork: true`
4. 删除不必要的 capabilities

```yaml
# 修复后的 Pod 配置
securityContext:
  allowPrivilegeEscalation: false
  runAsNonRoot: true
  runAsUser: 1001
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]
  seccompProfile:
    type: RuntimeDefault
```

### 必要时使用豁免

对于确实需要提升权限的负载（节点监控、网络插件），使用单独的命名空间或 OPA 豁免：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: system-monitoring
  labels:
    pod-security.kubernetes.io/enforce: privileged
```

### 自动化合规扫描

```bash
kubescape scan framework nsa --exceptions exceptions.json
kubescape scan --fail-threshold critical
```

---

## 经验教训

- **PSA `warn` 不是安全**：警告模式记录违规但不采取任何行动。没有 `enforce`，PSA 只是建议，不是策略
- **安全警告变成噪音**：开发者每天看到数百条警告。没有后果的 PSA 警告几周内就被忽略
- **PSA 本身不够**：将 PSA 与 OPA/Gatekeeper 或 Kyverno 结合，覆盖三个内置配置之外的策略
- **渐进式强制执行有效**：从 `audit` 开始，过渡到 `warn`，最终 `enforce`。但不要停在 `warn`——那不是终点线
- **系统负载需要例外**：某些系统组件（节点导出器、网络插件）确实需要更高权限。为它们使用专用的命名空间标签

---

## 总结

安全缺口：

```
团队采用 PSA 警告模式以避免破坏现有负载
→ 命名空间标记了 "warn: restricted" 但没有 "enforce: restricted"
→ 开发者部署了带有权限提升、Root 访问、宿主机网络的 Pod
→ PSA 显示警告——没有人采取行动
→ 6 个月内累积了 12 个违规 Pod
→ 季度合规扫描发现违规
→ 启用 PSA enforce 模式，修复 Pod
```

修复：改一个标签，从 `warn` 到 `enforce`。工作量：修复 6 个月的安全债务。从第一天起启用 PSA enforce 的成本为零。仅 warning 模式花费了 6 个月的暴露期和一次大修。尽早强制执行，经常强制执行。
