---
title: "etcd 未加密数据泄露风险——一个备份文件暴露了整个集群的所有秘密"
date: 2026-06-03
weight: 100300
slug: "etcd-unencrypted-data-exposure"
tags: ["kubernetes", "security", "etcd", "encryption", "troubleshooting"]
categories: ["Security"]
description: "etcd 数据泄露事件复盘——因未启用静态加密，etcd 快照文件中所有 Secret、ConfigMap 和 ServiceAccount Token 以明文存储，NAS 意外公网暴露导致全集群凭据泄露"
keywords: "etcd 静态加密, kubernetes etcd 备份安全, etcd 快照未加密, k8s secret 加密, etcd 数据泄露"
draft: false
featured: true
cover:
  image: ""
  caption: "etcd 未加密数据泄露——安全事件排查"
---

# etcd 未加密数据泄露风险——一个备份文件暴露了整个集群的所有秘密

## 常见搜索词

- etcd 静态加密 kubernetes
- etcd 备份暴露密钥
- kubernetes etcd 快照安全
- k8s secret 加密 etcd
- etcd 数据泄露事件

---

## 故障经过

**环境**: K8S v1.27, 3 节点 etcd 集群, 单区域生产集群, 备份存储在共享 NAS 上。

**时间**: 周三 14:00。安全团队收到外部安全研究员的告警：一个 etcd 快照文件在公网暴露的 NAS 共享上可被访问。

**发现过程**: 一名安全研究员在例行 Shodan 扫描中发现了一个无需认证的 NFS 共享。里面：`etcd-snapshot-2026-06-02.db`。他们下载了它，解码，发现了 Kubernetes Secrets，并负责任地披露。

```bash
# 研究员无需凭据即可挂载 NFS 共享
mount -t nfs 203.0.113.50:/backups /mnt/backups
ls /mnt/backups/
etcd-snapshot-2026-06-01.db
etcd-snapshot-2026-06-02.db
kubernetes-manifests/
```

**影响**: Kubernetes 集群中的每一个 Secret——TLS 密钥、数据库密码、API Token、ServiceAccount 凭据——都被任何发现这个 NFS 共享的人获取。估计暴露窗口：3-6 个月。

---

## 背景

etcd 是 Kubernetes 的后端存储。每个集群资源——Secrets、ConfigMaps、ServiceAccounts、RBAC 策略——都存储在 etcd 中。如果有人获取了 etcd 快照，他们就拥有了集群中的一切。

团队通过 cronjob 配置了每日自动 etcd 备份：

```bash
0 2 * * * root ETCDCTL_API=3 etcdctl snapshot save \
  /backups/etcd-snapshot-$(date +\%F).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

备份写入挂载在 `/backups` 的 NAS 共享。NAS 本应仅内网访问，但一个配置错误的防火墙规则将 NFS 服务暴露到了互联网。

更大的问题：**etcd 静态加密未启用**。etcd 中的所有数据都以明文存储：

```bash
kubectl get EncryptionConfiguration -o yaml 2>/dev/null || echo "未找到加密配置"
```

输出：`未找到加密配置`。etcd 中的每个资源——包括 Secrets——都以明文 JSON 或 YAML 存储。

```bash
# 任何有快照的人都可以解码任何 Secret
strings etcd-snapshot-2026-06-02.db | grep -A2 "kind: Secret" | head -20
```

---

## 排查过程

### 第一步：确认暴露

```bash
kubectl describe encryptionconfig 2>/dev/null || echo "集群中不存在 EncryptionConfig 资源"
```

集群中没有 `EncryptionConfiguration`。所有 etcd 数据都未加密存储。

### 第二步：识别暴露数据范围

集群中的每个 Secret、ConfigMap、ServiceAccount Token 和 TLS 证书都已暴露。

### 第三步：检查访问日志

NAS 日志显示来自内部网络范围之外的 IP 连接——确认暴露已被未授权方发现。

---

## 根因

1. **未启用静态加密**：etcd 以明文存储所有数据。Kubernetes Secrets 未配置加密
2. **备份文件未加密**：etcd 快照在 NAS 上未加密存储
3. **NAS 意外公网暴露**：配置错误的防火墙规则使 NFS 共享可从互联网访问
4. **无备份访问监控**：没有来自外部 IP 的 NFS 连接告警
5. **无备份完整性验证**：备份 cronjob 运行了，但没有人验证备份是安全的

---

## 解决方案

### 紧急处置

```bash
# 1. 阻断 NAS 的外部访问
iptables -A INPUT -s 0.0.0.0/0 -p tcp --dport 2049 -j DROP
iptables -A INPUT -s 10.0.0.0/8 -p tcp --dport 2049 -j ACCEPT

# 2. 轮换所有集群 Secret
kubectl delete secret --all-namespaces --all
kubectl delete pod --all-namespaces --all

# 3. 假定所有暴露的凭据已被入侵
# - 轮换数据库密码
# - 重新签发 TLS 证书
# - 重新生成云提供商凭据
```

### 启用 etcd 静态加密

```yaml
# encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-编码的-32-字节密钥>
      - identity: {}
```

```bash
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
kubectl apply -f encryption-config.yaml
# 编辑 kube-apiserver 清单添加:
# --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

### 加密现有数据

启用加密后，etcd 中的现有数据仍未加密。需要重写所有 Secret 来加密它们：

```bash
kubectl get secrets --all-namespaces -o json | jq -r '.items[].metadata.name' | while read secret; do
  kubectl get secret $secret -n $ns -o json | kubectl replace -f -
done
```

### 保护 etcd 备份

```bash
# 用 GPG 加密备份文件
etcdctl snapshot save /tmp/etcd-snapshot.db
gpg --encrypt --recipient admin@company.com /tmp/etcd-snapshot.db
mv /tmp/etcd-snapshot.db.gpg /backups/
```

### 加固 NAS 访问

```bash
# /etc/exports — 仅限内部 IP
/backups 10.0.0.0/8(rw,no_root_squash,no_subtree_check)
```

---

## 经验教训

- **etcd 默认以明文存储一切**：Kubernetes Secrets 在 etcd 中仅 base64 编码但未加密。任何有 etcd 访问权限的人都能拿到所有 Secret
- **静态加密不是可选项**：集群启动时勾选它是有原因的。没有它，对 etcd 数据的物理访问 = 完全集群沦陷
- **备份文件也是秘密**：etcd 快照包含整个集群状态。在任何地方存储之前加密备份
- **NAS 不是安全边界**：NFS 共享往往暴露得比预期更广。定期验证防火墙规则
- **假设所有备份都是公开的**：围绕备份文件最终会泄露的假设来构建安全模型。静态加密 + 加密备份 = 纵深防御

---

## 总结

攻击链路：

```
etcd 未配置静态加密
→ 每日 cronjob 在 NAS 上创建 etcd 快照
→ NAS NFS 共享意外暴露到互联网（防火墙变更）
→ 安全研究员通过 Shodan 发现 NFS 共享
→ 下载 etcd 快照
→ 解码所有 Secret、ConfigMap、ServiceAccount Token、TLS 证书
→ 负责任地向安全团队披露
```

恢复：全凭据轮换，6 小时。修复：静态加密 + 加密备份，1 小时。预防：集群启动时的 10 分钟配置。etcd 是 Kubernetes 的心脏——像对待心脏一样加密它。
