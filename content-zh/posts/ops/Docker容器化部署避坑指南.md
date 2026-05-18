---
title: "Docker 容器化部署避坑指南"
date: 2025-01-27
weight: 1401
draft: false
tags: ["ops"]
featured: true
cover:
  image: "/images/docker-banner.svg"
  caption: "Docker 容器化部署避坑指南"
---

## 前言

Docker 已经成为了现代应用部署的事实标准。但在实际落地过程中，从"能在本地跑"到"在生产环境稳定运行"，中间藏着不少坑。这篇文章总结了我自己在容器化部署中踩过的坑和解决方案，希望能帮你少走弯路。

---

## 坑一：镜像过大，构建慢、传输慢

### 问题

一个简单的 Node.js 应用，不加任何优化，镜像体积轻松上 1GB。构建一次几分钟，推送到仓库更是煎熬。

### 解决方案

**1. 使用多阶段构建**

```dockerfile
# 构建阶段
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# 运行阶段
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

最终镜像只包含运行所需文件，构建工具和中间产物全部丢弃。

**2. 选择更小的基础镜像**

| 镜像 | 体积 | 适用场景 |
|------|------|---------|
| `ubuntu:22.04` | ~80MB | 通用 |
| `debian:bookworm-slim` | ~80MB | 通用 |
| `alpine` | ~5MB | 追求极致体积 |
| `distroless` | ~10MB | 安全要求高 |

**3. 优化层缓存**

```dockerfile
# 把不常变动的操作放在前面，利用 Docker 层缓存
COPY package*.json ./
RUN npm ci --only=production  # 只有 package.json 变动才重新安装
COPY . .                       # 源码变动不影响上层缓存
```

### 效果

多阶段构建 + Alpine 基础镜像，Node.js 应用从 1.2GB 降到 ~150MB。

---

## 坑二：容器以 root 运行

### 问题

默认情况下容器内以 root 运行。如果攻击者通过应用漏洞进入容器，就能获得容器内最高权限。更严重的是，如果容器被配置了特权模式或存在内核漏洞，攻击者可能逃逸到宿主机。

### 解决方案

```dockerfile
FROM node:20-alpine

# 创建非 root 用户
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

WORKDIR /app
COPY --chown=appuser:appgroup . .

# 切换用户
USER appuser

EXPOSE 3000
CMD ["node", "index.js"]
```

在 `docker-compose.yml` 中也显式指定：

```yaml
services:
  app:
    user: "1001:1001"
```

### 补充

- 如果应用需要监听 80/443 端口，使用非 root 用户时需配置端口转发或将镜像内端口映射到宿主机高位端口
- 在 Kubernetes 中可以通过 `securityContext` 进一步限制：

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1001
  runAsGroup: 1001
  capabilities:
    drop: ["ALL"]
  readOnlyRootFilesystem: true
```

---

## 坑三：时区与 locale 问题

### 问题

容器默认时区是 UTC，日志时间戳和业务时间对不上，排查问题时极其痛苦。

### 解决方案

```dockerfile
# Debian/Ubuntu 系
RUN apt-get update && \
    apt-get install -y tzdata && \
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    dpkg-reconfigure -f noninteractive tzdata

# Alpine 系
RUN apk add --no-cache tzdata && \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone
```

或者在 `docker-compose.yml` 中设置环境变量（更推荐）：

```yaml
services:
  app:
    environment:
      - TZ=Asia/Shanghai
```

---

## 坑四：日志管理不当

### 问题

默认情况下容器日志写入 JSON 文件且无轮转，长时间运行会占满磁盘。

### 解决方案

**1. 限制日志大小**

```yaml
# docker-compose.yml
services:
  app:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

**2. 应用内日志输出到 stdout/stderr**

**原则：不要将日志写入文件，让 Docker 的日志驱动来处理。**

```javascript
// ❌ 错误做法
const fs = require('fs');
fs.appendFileSync('/var/log/app.log', message);

// ✅ 正确做法
console.log(JSON.stringify({ level: 'info', message, timestamp: new Date().toISOString() }));
console.error(JSON.stringify({ level: 'error', message, timestamp: new Date().toISOString() }));
```

**3. 生产环境使用专业日志收集**

```
stdout/stderr → Docker → json-file → Filebeat/fluentd → Elasticsearch/Kafka
```

---

## 坑五：健康检查缺失

### 问题

容器进程还在运行，但应用已经死锁或卡死。Docker 认为容器是 healthy，流量照常涌入，用户看到白屏或 500。

### 解决方案

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

或者在 `docker-compose.yml` 中：

```yaml
services:
  app:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 10s
```

**健康检查接口应该做什么：**

```javascript
app.get('/health', async (req, res) => {
  try {
    // 检查数据库连接
    await db.raw('SELECT 1');
    // 检查缓存连接
    await redis.ping();
    // 检查磁盘空间（如果需要）
    res.json({ status: 'healthy' });
  } catch (err) {
    res.status(503).json({ status: 'unhealthy', reason: err.message });
  }
});
```

---

## 坑六：环境管理混乱

### 问题

配置写死在镜像里，导致不同环境要打不同的镜像。或者依赖大量环境变量但不做校验，运行时才发现缺配置。

### 解决方案

**1. 构建一次，到处部署**

镜像不包含环境特定配置，所有环境差异通过环境变量注入。

```yaml
# docker-compose.dev.yml
services:
  app:
    image: myapp:latest
    environment:
      - DB_HOST=localhost
      - DB_PORT=5432
      - LOG_LEVEL=debug

# docker-compose.prod.yml
services:
  app:
    image: myapp:latest
    environment:
      - DB_HOST=prod-db.internal
      - DB_PORT=5432
      - LOG_LEVEL=warn
```

**2. 启动时校验环境变量**

```javascript
const requiredEnv = ['DB_HOST', 'DB_PORT', 'REDIS_URL', 'JWT_SECRET'];
for (const key of requiredEnv) {
  if (!process.env[key]) {
    throw new Error(`Missing required environment variable: ${key}`);
  }
}
```

**3. 使用 `.env.example` 文档化**

```
# .env.example
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=
REDIS_URL=redis://localhost:6379
LOG_LEVEL=debug
JWT_SECRET=
```

---

## 坑七：内存与资源限制缺失

### 问题

一个容器的内存泄漏拖垮整台宿主机上的所有服务。

### 解决方案

**始终设置资源限制：**

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

在 Docker run 中：

```bash
docker run -d --memory="512m" --cpus="0.5" myapp
```

### 监控建议

- 设置容器内存使用率告警（如 >80% 持续 5 分钟）
- Java 应用特别注意设置 `-Xmx` 要小于容器内存限制
- Node.js 应用注意 `--max-old-space-size`

---

## 坑八：数据持久化疏忽

### 问题

容器重启后数据丢失，或者多个容器同时写同一卷导致数据损坏。

### 解决方案

**1. 明确区分有状态和无状态服务**

无状态服务（API、Worker）：不要依赖本地存储
有状态服务（数据库、缓存）：使用 volumes 或外部存储

**2. 使用命名卷而非绑定挂载**

```yaml
services:
  postgres:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data  # ✅ 命名卷，Docker 管理

volumes:
  pgdata:
```

**3. 非数据库场景考虑对象存储**

文件上传不要存容器内 -> 用 S3/MinIO / 阿里云 OSS

---

## 坑九：网络配置混乱

### 问题

容器间通信用 `localhost` 失败，或者在 macOS 上一切正常但部署到 Linux 就出问题。

### 解决方案

**1. 使用自定义网络**

```yaml
services:
  app:
    networks:
      - app-net
  redis:
    networks:
      - app-net

networks:
  app-net:
    driver: bridge
```

容器间通过服务名通信：`redis:6379` 而不是 `localhost:6379`

**2. 理解 network_mode**

- `bridge`（默认）：容器有自己的网络栈，通过端口映射访问
- `host`：容器共享宿主机网络（性能好但有安全隐患）
- `none`：无网络

---

## 坑十：CI/CD 中构建缓存失效

### 问题

每次 CI 构建都从头安装依赖，构建时间越来越长。

### 解决方案

**1. GitHub Actions 中利用缓存**

```yaml
- name: Cache Docker layers
  uses: actions/cache@v3
  with:
    path: /tmp/.buildx-cache
    key: ${{ runner.os }}-buildx-${{ github.sha }}
    restore-keys: |
      ${{ runner.os }}-buildx-
```

**2. 优化 Dockerfile 层顺序**

回顾坑一中的层缓存策略：**把变更频率低的操作放在前面**。

---

## 总结清单

| 项目 | 检查要点 |
|------|---------|
| 镜像体积 | 多阶段构建、Alpine、优化层顺序 |
| 安全 | 非 root 用户、最小权限原则 |
| 时区 | 设置 TZ 环境变量或安装 tzdata |
| 日志 | stdout/stderr、限制日志大小 |
| 健康检查 | 配置 HEALTHCHECK、检查依赖服务 |
| 配置 | 环境变量注入、启动时校验 |
| 资源限制 | 设置 CPU/内存上限、配置告警 |
| 数据持久化 | 命名卷、无状态设计 |
| 网络 | 自定义网络、服务名通信 |
| CI/CD | 缓存构建层、优化构建速度 |

---

容器化部署不是简单的 `docker run` 就完事了。希望这份清单能帮你少踩一些坑，让容器真正成为你部署流程的加速器，而不是麻烦制造者。

如果你有遇到其他 Docker 部署的坑，欢迎补充！
