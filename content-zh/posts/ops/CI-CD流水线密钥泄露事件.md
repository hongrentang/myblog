---
title: "CI/CD 流水线密钥泄露事件——一行调试日志烧掉 47000 美元"
date: 2026-06-02
weight: 100250
slug: "cicd-pipeline-secret-leak"
tags: ["security", "ci-cd", "devops", "cloud", "troubleshooting"]
categories: ["Security"]
description: "CI/CD 流水线安全事件复盘——GitHub Actions 构建日志中暴露的 AWS 密钥被爬虫抓取，导致 347 台 GPU 实例被启动用于挖矿，产生 47000 美元账单"
keywords: "ci cd 流水线密钥泄露, github actions secret 暴露, aws 密钥被打印到日志, 挖矿云账单, devsecops 流水线安全"
draft: false
featured: true
cover:
  image: ""
  caption: "CI/CD 流水线密钥泄露——安全事件排查"
---

# CI/CD 流水线密钥泄露事件——一行调试日志烧掉 47000 美元

## 常见搜索词

- ci cd 流水线密钥在日志中暴露
- github actions aws 凭据泄露
- aws 密钥在构建日志中泄露
- 挖矿攻击云账单
- 如何保护 ci cd 流水线密钥

---

## 故障经过

**环境**: GitHub Actions CI/CD, AWS 账户 5 个区域 200+ 资源, 40+ 微服务。

**时间**: 周六凌晨 03:15。AWS 成本异常检测告警：预测未来 24 小时内支出将超过 $10,000——正常是 $800/天。

**初始症状**: 财务团队的电话响了。AWS 自动发送了"消费警告"——已超过日平均 $5,000。当值班工程师查看时，账单已达 $12,000 且还在攀升。

```bash
# 工程师在 AWS Cost Explorer 中看到的
区域: us-east-1
服务: EC2 (Spot 实例)
从 02:00 UTC 开始异常飙升
实例数: 347（正常 50）
实例类型: p3.2xlarge, p3.8xlarge（GPU 实例——挖矿最爱）
```

**影响**: 36 小时内 $47,000 的非授权 AWS 费用。全账户凭证轮换。48 小时生产部署冻结。

---

## 背景

团队使用 GitHub Actions 做 CI/CD。AWS 凭据存储为 GitHub Actions Secrets——配置正确，日志中会被屏蔽，代码中从不出现。安全性被认为"足够好"。

但有一个漏洞：一名开发在编写新部署脚本时，在构建步骤中添加了打印环境变量的调试日志：

```yaml
steps:
  - name: 配置 AWS 凭证
    uses: aws-actions/configure-aws-credentials@v1
    with:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      aws-region: us-east-1

  - name: 调试——打印环境变量
    run: env | grep AWS_            # ← 这一行导致事故
```

GitHub Actions 会在日志中屏蔽密钥——但仅限于 `${{ secrets.* }}` 语法。`env` 命令打印的是未屏蔽的值，因为在运行时密钥会被导出为真实的环境变量。日志中出现了：

```
AWS_ACCESS_KEY_ID=AKIAXXXXXXXXXXXX
AWS_SECRET_ACCESS_KEY=xxxxxxxxxxxxxx
```

攻击者的爬虫一直在扫描公开的 GitHub Actions 日志。提交推送到公开仓库后 7 分钟内，凭据就被盗了。

---

## 排查过程

### 第一步：定位泄露源

```bash
gh run list --workflow=deploy.yml --limit=10 --json=databaseId,conclusion,headBranch
```

最近一次 `feature/debug-logging` 分支的运行在原始日志中显示完整环境变量。

### 第二步：评估云账户损失

```bash
aws ec2 describe-instances --filters "Name=launch-time,After=2026-06-01T00:00:00Z" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,LaunchTime:LaunchTime}' \
  --output table
```

347 台实例：
- 280 台 p3.2xlarge（GPU）
- 67 台 p3.8xlarge（GPU）
- 全部运行在账户从未使用过的区域

```bash
aws iam get-credential-report --output text | cut -d',' -f1,4,5,16 | head -20
```

被泄露的密钥从东欧和中国的 IP 使用。

```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=arn:aws:s3:::production-data-backup
```

攻击者已访问了包含生产数据库备份的 S3 存储桶。

### 第三步：追溯攻击路径

攻击者的爬虫：
1. 扫描公开 GitHub Actions 日志中的 AWS Access Key（正则: `AKIA[0-9A-Z]{16}`）
2. 7 分钟内找到暴露的密钥
3. 使用密钥启动 GPU Spot 实例挖矿
4. 创建具有管理员权限的新 IAM 用户以持久化
5. 从可访问的 S3 存储桶窃取数据
6. 设置额外的 IAM Access Key 作为备用

---

## 根因

1. **CI 中打印环境变量调试**：`env | grep AWS_` 暴露了 GitHub Actions 本应屏蔽的密钥
2. **公开仓库 + 敏感 CI/CD**：仓库是公开的，CI 日志公开可访问
3. **AWS 密钥权限过大**：CI/CD 使用的 IAM 用户有 `AdministratorAccess`——远超所需
4. **无 CI 日志扫描**：没有自动扫描构建日志中密钥的工具
5. **无 AWS 资源限制**：没有 SCP 或预算限制来阻止资源失控

---

## 解决方案

### 紧急处置

```bash
# 1. 立即停用被泄露的 AWS 密钥
aws iam update-access-key --access-key-id AKIAXXXXXXXXXXXX --status Inactive

# 2. 终止所有非授权 EC2 实例
aws ec2 describe-instances --filters "Name=instance-state-name,Value=running" \
  --query 'Reservations[].Instances[?LaunchTime>`2026-06-01`].[InstanceId]' \
  --output text | xargs -r aws ec2 terminate-instances --instance-ids

# 3. 删除非授权 IAM 用户
aws iam list-users | jq -r '.Users[] | select(.CreateDate > "2026-06-01") | .UserName' | \
  xargs -r -I{} aws iam delete-user --user-name {}

# 4. 撤销并重新签发所有 AWS 凭据
aws iam create-access-key --user-name cicd-user
```

### 预算和资源控制

```bash
aws budgets create-budget \
  --budget-file '{"BudgetName":"monthly-budget","BudgetLimit":{"Amount":"5000","Unit":"USD"},"TimeUnit":"MONTHLY","BudgetType":"COST"}' \
  --notifications-with-subscribers '[]'
```

### 修复 CI/CD 流水线

```yaml
steps:
  - name: 配置 AWS 凭证
    uses: aws-actions/configure-aws-credentials@v4
    with:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      aws-region: us-east-1

  - name: 部署应用
    run: |
      # 使用特定变量，决不转储所有变量
      ./deploy.sh
```

### CI/CD 最小权限原则

限制 CI/CD IAM 策略仅允许 ECR 推送、ECS 更新和特定 S3 存储桶的写入，而非 `AdministratorAccess`。

### 流水线密钥扫描

```yaml
steps:
  - name: 密钥扫描
    uses: trufflesecurity/trufflehog@v3
    with:
      path: .
      base: ${{ github.event.before }}
      head: HEAD
```

### 监控告警

```yaml
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

## 经验教训

- **GitHub Actions 密钥屏蔽不是加密**：用 `***` 屏蔽只是 UI 层面的隐藏。运行时密钥是真实的环境变量。任何打印 env 的命令都会暴露它们
- **CI/CD 密钥需要最小权限**：CI 中使用 `AdministratorAccess` 意味着每次提交都可能摧毁整个账户。将 CI 凭据限制在流水线精确需要的权限
- **公开仓库 + CI 日志 = 公开密钥**：如果仓库是公开的，CI 日志就是公开的。假设攻击者有爬虫在监控每一次新运行
- **预算告警能省钱**：$500/$1000/$5000 的 AWS 预算告警会在 10 分钟内发现异常，而不是 12 小时
- **IAM 凭据报告是好朋友**：`aws iam generate-credential-report` 显示密钥的最后使用时间。每周审计一次

---

## 总结

攻击链路：

```
开发者添加调试日志: env | grep AWS_
→ 提交并推送到公开仓库
→ GitHub Actions 运行工作流，将密钥打印到日志
→ 攻击者的爬虫扫描公共 CI 日志，发现 AKIA 密钥
→ 启动 347 台 GPU Spot 实例挖矿
→ 创建持久化 IAM 用户
→ 窃取 S3 数据
→ 36 小时内产生 $47,000 账单
```

修复：48 小时。损失：$47,000 + 48 小时部署冻结。起因：一行调试代码。代价：四万七千美元。提交前删掉你的调试日志。
