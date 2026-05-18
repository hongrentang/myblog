---
title: "Docker Container Deployment Pitfalls Guide"
date: 2025-01-27
draft: false
tags: ["ops"]
featured: true
cover:
  image: "/images/docker-banner.svg"
  caption: "Docker Container Deployment Pitfalls Guide"
---

## Preface

Docker has become the de facto standard for modern application deployment. However, in practice, there are many pitfalls lurking between "it works on my machine" and "running stably in production." This article summarizes the traps I've personally fallen into during containerized deployment and their solutions, hoping to help you avoid the same detours.

---

## Pitfall 1: Oversized Images — Slow Builds and Transfers

### Problem

A simple Node.js application, without any optimization, can easily reach 1GB in image size. Building takes minutes, and pushing to a registry is even more painful.

### Solution

**1. Use Multi-Stage Builds**

```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Runtime stage
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

The final image only contains files needed at runtime — build tools and intermediate artifacts are all discarded.

**2. Choose Smaller Base Images**

| Image | Size | Use Case |
|------|------|---------|
| `ubuntu:22.04` | ~80MB | General purpose |
| `debian:bookworm-slim` | ~80MB | General purpose |
| `alpine` | ~5MB | Pursuing minimal size |
| `distroless` | ~10MB | High security requirements |

**3. Optimize Layer Caching**

```dockerfile
# Place infrequently changed operations first to leverage Docker layer caching
COPY package*.json ./
RUN npm ci --only=production  # Only reinstalls when package.json changes
COPY . .                       # Source code changes don't affect upper layer cache
```

### Result

Multi-stage builds + Alpine base image reduce a Node.js application from 1.2GB to ~150MB.

---

## Pitfall 2: Running Containers as root

### Problem

By default, processes inside containers run as root. If an attacker exploits an application vulnerability to enter the container, they gain the highest privileges within it. More critically, if the container is configured with privileged mode or there are kernel vulnerabilities, the attacker may escape to the host.

### Solution

```dockerfile
FROM node:20-alpine

# Create a non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

WORKDIR /app
COPY --chown=appuser:appgroup . .

# Switch user
USER appuser

EXPOSE 3000
CMD ["node", "index.js"]
```

Also specify explicitly in `docker-compose.yml`:

```yaml
services:
  app:
    user: "1001:1001"
```

### Additional Notes

- If your application needs to listen on ports 80/443 when using a non-root user, configure port forwarding or map the container's internal port to a high-numbered port on the host
- In Kubernetes, you can further restrict via `securityContext`:

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

## Pitfall 3: Timezone and Locale Issues

### Problem

The default timezone inside containers is UTC. Log timestamps don't match business time, making troubleshooting extremely painful.

### Solution

```dockerfile
# Debian/Ubuntu based
RUN apt-get update && \
    apt-get install -y tzdata && \
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    dpkg-reconfigure -f noninteractive tzdata

# Alpine based
RUN apk add --no-cache tzdata && \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone
```

Or set environment variables in `docker-compose.yml` (recommended):

```yaml
services:
  app:
    environment:
      - TZ=Asia/Shanghai
```

---

## Pitfall 4: Improper Log Management

### Problem

By default, container logs are written to JSON files with no rotation. Long-running containers can fill up disk space.

### Solution

**1. Limit Log Size**

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

**2. Output Logs to stdout/stderr Within the Application**

**Principle: Do not write logs to files — let Docker's logging driver handle them.**

```javascript
// ❌ Wrong approach
const fs = require('fs');
fs.appendFileSync('/var/log/app.log', message);

// ✅ Correct approach
console.log(JSON.stringify({ level: 'info', message, timestamp: new Date().toISOString() }));
console.error(JSON.stringify({ level: 'error', message, timestamp: new Date().toISOString() }));
```

**3. Use Professional Log Collection in Production**

```
stdout/stderr → Docker → json-file → Filebeat/fluentd → Elasticsearch/Kafka
```

---

## Pitfall 5: Missing Health Checks

### Problem

The container process is still running, but the application has deadlocked or hung. Docker considers the container healthy, traffic continues to pour in, and users see a blank page or a 500 error.

### Solution

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

Or in `docker-compose.yml`:

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

**What the health check endpoint should do:**

```javascript
app.get('/health', async (req, res) => {
  try {
    // Check database connection
    await db.raw('SELECT 1');
    // Check cache connection
    await redis.ping();
    // Check disk space (if needed)
    res.json({ status: 'healthy' });
  } catch (err) {
    res.status(503).json({ status: 'unhealthy', reason: err.message });
  }
});
```

---

## Pitfall 6: Messy Environment Management

### Problem

Configuration is hard-coded into the image, requiring different images for different environments. Or, the application relies on many environment variables but performs no validation, only to find missing configuration at runtime.

### Solution

**1. Build Once, Deploy Anywhere**

Images should not contain environment-specific configuration. All environment differences are injected via environment variables.

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

**2. Validate Environment Variables at Startup**

```javascript
const requiredEnv = ['DB_HOST', 'DB_PORT', 'REDIS_URL', 'JWT_SECRET'];
for (const key of requiredEnv) {
  if (!process.env[key]) {
    throw new Error(`Missing required environment variable: ${key}`);
  }
}
```

**3. Document with `.env.example`**

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

## Pitfall 7: Missing Memory and Resource Limits

### Problem

A memory leak in one container can bring down all services on the entire host.

### Solution

**Always set resource limits:**

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

With Docker run:

```bash
docker run -d --memory="512m" --cpus="0.5" myapp
```

### Monitoring Tips

- Set container memory usage alerts (e.g., >80% for 5 consecutive minutes)
- Java applications: ensure `-Xmx` is set lower than the container memory limit
- Node.js applications: pay attention to `--max-old-space-size`

---

## Pitfall 8: Neglecting Data Persistence

### Problem

Data is lost after container restart, or multiple containers writing to the same volume simultaneously causes data corruption.

### Solution

**1. Clearly Distinguish Between Stateful and Stateless Services**

Stateless services (API, Worker): do not rely on local storage
Stateful services (database, cache): use volumes or external storage

**2. Use Named Volumes Instead of Bind Mounts**

```yaml
services:
  postgres:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data  # ✅ Named volume, managed by Docker

volumes:
  pgdata:
```

**3. Consider Object Storage for Non-Database Scenarios**

Don't store file uploads inside containers -> use S3/MinIO / Alibaba Cloud OSS

---

## Pitfall 9: Network Configuration Chaos

### Problem

Communication between containers using `localhost` fails, or everything works fine on macOS but breaks when deployed to Linux.

### Solution

**1. Use Custom Networks**

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

Containers communicate via service names: `redis:6379` instead of `localhost:6379`

**2. Understand network_mode**

- `bridge` (default): Each container has its own network stack, accessed via port mapping
- `host`: Containers share the host's network (better performance but security risks)
- `none`: No network

---

## Pitfall 10: Cache Invalidation in CI/CD

### Problem

Every CI build installs dependencies from scratch, and build times keep getting longer.

### Solution

**1. Leverage Caching in GitHub Actions**

```yaml
- name: Cache Docker layers
  uses: actions/cache@v3
  with:
    path: /tmp/.buildx-cache
    key: ${{ runner.os }}-buildx-${{ github.sha }}
    restore-keys: |
      ${{ runner.os }}-buildx-
```

**2. Optimize Dockerfile Layer Order**

Refer back to the layer caching strategy in Pitfall 1: **Place infrequently changed operations first.**

---

## Summary Checklist

| Item | Check Points |
|------|---------|
| Image Size | Multi-stage builds, Alpine, optimize layer order |
| Security | Non-root user, principle of least privilege |
| Timezone | Set TZ environment variable or install tzdata |
| Logs | stdout/stderr, limit log size |
| Health Check | Configure HEALTHCHECK, check dependent services |
| Configuration | Environment variable injection, validate at startup |
| Resource Limits | Set CPU/memory limits, configure alerts |
| Data Persistence | Named volumes, stateless design |
| Network | Custom network, service name communication |
| CI/CD | Cache build layers, optimize build speed |

---

Containerized deployment is not just about running `docker run` and calling it done. I hope this checklist helps you avoid some common pitfalls and turns containers into an accelerator for your deployment workflow rather than a source of trouble.

If you've encountered other Docker deployment pitfalls, feel free to share them!
