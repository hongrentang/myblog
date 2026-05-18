---
title: "Go from Beginner to Pro · Part 25: Project 6 — URL Shortener"
date: 2025-01-18
weight: 1225
draft: false
tags: ["go"]
---

## Project Introduction

Implement a complete URL Shortener service: efficient short code generation algorithm, Redis caching, Gin framework, production-grade deployment. This is the final project in the series, comprehensively applying all the knowledge learned so far.

## Project Structure

```
shorturl/
├── cmd/server/main.go
├── internal/
│   ├── config/config.go          # Configuration
│   ├── handler/
│   │   ├── shorten.go            # Create short URL
│   │   └── redirect.go           # Redirect
│   ├── service/
│   │   └── shortener.go          # Core business logic
│   ├── storage/
│   │   ├── redis.go              # Redis storage
│   │   └── mysql.go              # MySQL persistence
│   └── model/
│       └── url.go                # Data model
├── pkg/
│   ├── idgen/
│   │   └── snowflake.go          # Snowflake ID
│   └── base62/
│       └── encode.go             # Base62 encoding
├── config.yaml
├── Dockerfile
├── docker-compose.yml
├── go.mod
└── Makefile
```

---

## Step 1: Base62 Encoding

```go
// pkg/base62/encode.go
package base62

const chars = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"

// Encode encodes a number to a Base62 string
func Encode(num uint64) string {
    if num == 0 {
        return string(chars[0])
    }

    var result []byte
    for num > 0 {
        result = append(result, chars[num%62])
        num /= 62
    }

    // Reverse
    for i, j := 0, len(result)-1; i < j; i, j = i+1, j-1 {
        result[i], result[j] = result[j], result[i]
    }

    return string(result)
}

// Decode decodes a Base62 string to a number
func Decode(s string) uint64 {
    var result uint64
    for _, c := range s {
        result = result * 62 + uint64(indexOf(c))
    }
    return result
}

func indexOf(c rune) int {
    switch {
    case c >= '0' && c <= '9':
        return int(c - '0')
    case c >= 'A' && c <= 'Z':
        return int(c-'A') + 10
    case c >= 'a' && c <= 'z':
        return int(c-'a') + 36
    default:
        return 0
    }
}

// Example: Encode(125) → "cb"
//          Decode("cb") → 125
// 6-digit Base62 can represent 62^6 ≈ 56 billion short codes
```

## Step 2: Snowflake ID

```go
// pkg/idgen/snowflake.go
package idgen

import (
    "sync"
    "time"
)

const (
    epoch        = 1700000000000  // Start timestamp
    workerBits   = 10
    sequenceBits = 12
    workerShift  = sequenceBits
    timeShift    = sequenceBits + workerBits
    maxWorker    = -1 ^ (-1 << workerBits)
    maxSequence  = -1 ^ (-1 << sequenceBits)
)

type Snowflake struct {
    mu         sync.Mutex
    timestamp  int64
    workerID   int64
    sequence   int64
}

func New(workerID int64) (*Snowflake, error) {
    if workerID < 0 || workerID > maxWorker {
        return nil, ErrInvalidWorkerID
    }
    return &Snowflake{
        workerID: workerID,
        timestamp: 0,
        sequence: 0,
    }, nil
}

func (s *Snowflake) NextID() uint64 {
    s.mu.Lock()
    defer s.mu.Unlock()

    now := time.Now().UnixMilli()
    if now == s.timestamp {
        s.sequence = (s.sequence + 1) & maxSequence
        if s.sequence == 0 {
            for now <= s.timestamp {
                now = time.Now().UnixMilli()
            }
        }
    } else {
        s.sequence = 0
    }

    s.timestamp = now

    id := uint64((now-epoch)<<timeShift |
        (s.workerID << workerShift) |
        s.sequence)
    return id
}
```

## Step 3: Configuration

```go
// internal/config/config.go
package config

type Config struct {
    Server   ServerConfig   `yaml:"server"`
    Database DatabaseConfig `yaml:"database"`
    Redis    RedisConfig    `yaml:"redis"`
}

type ServerConfig struct {
    Port    int    `yaml:"port"`
    BaseURL string `yaml:"base_url"`
}

type DatabaseConfig struct {
    DSN string `yaml:"dsn"`
}

type RedisConfig struct {
    Addr     string `yaml:"addr"`
    Password string `yaml:"password"`
    DB       int    `yaml:"db"`
}

func Load(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, err
    }
    var cfg Config
    if err := yaml.Unmarshal(data, &cfg); err != nil {
        return nil, err
    }
    return &cfg, nil
}
```

```yaml
# config.yaml
server:
  port: 8080
  base_url: "https://s.url"

database:
  dsn: "root:password@tcp(mysql:3306)/shorturl?charset=utf8mb4&parseTime=True"

redis:
  addr: "redis:6379"
  password: ""
  db: 0
```

## Step 4: Storage Layer

```go
// internal/storage/redis.go
package storage

import (
    "context"
    "time"

    "github.com/redis/go-redis/v9"
)

type RedisStore struct {
    client *redis.Client
}

func NewRedisStore(cfg *config.RedisConfig) *RedisStore {
    return &RedisStore{
        client: redis.NewClient(&redis.Options{
            Addr:     cfg.Addr,
            Password: cfg.Password,
            DB:       cfg.DB,
        }),
    }
}

func (r *RedisStore) Set(ctx context.Context, shortCode, longURL string, ttl time.Duration) error {
    return r.client.Set(ctx, "short:"+shortCode, longURL, ttl).Err()
}

func (r *RedisStore) Get(ctx context.Context, shortCode string) (string, error) {
    return r.client.Get(ctx, "short:"+shortCode).Result()
}

func (r *RedisStore) Exists(ctx context.Context, shortCode string) (bool, error) {
    n, err := r.client.Exists(ctx, "short:"+shortCode).Result()
    return n > 0, err
}
```

```go
// internal/storage/mysql.go
package storage

import (
    "context"
    "time"

    "gorm.io/gorm"
)

type URLMapping struct {
    ID        uint64    `gorm:"primaryKey"`
    ShortCode string    `gorm:"uniqueIndex;size:10;not null"`
    LongURL   string    `gorm:"type:text;not null"`
    ExpireAt  time.Time `gorm:"index"`
    CreatedAt time.Time
}

type MySQLStore struct {
    db *gorm.DB
}

func NewMySQLStore(dsn string) (*MySQLStore, error) {
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
    if err != nil {
        return nil, err
    }
    db.AutoMigrate(&URLMapping{})
    return &MySQLStore{db: db}, nil
}

func (s *MySQLStore) Save(ctx context.Context, mapping *URLMapping) error {
    return s.db.WithContext(ctx).Create(mapping).Error
}

func (s *MySQLStore) FindByShortCode(ctx context.Context, code string) (*URLMapping, error) {
    var m URLMapping
    err := s.db.WithContext(ctx).Where("short_code = ?", code).First(&m).Error
    return &m, err
}
```

## Step 5: Service Layer

```go
// internal/service/shortener.go
package service

import (
    "context"
    "crypto/rand"
    "fmt"
    "time"

    "shorturl/internal/config"
    "shorturl/internal/storage"
    "shorturl/pkg/base62"
    "shorturl/pkg/idgen"
)

type Shortener struct {
    redis     *storage.RedisStore
    mysql     *storage.MySQLStore
    idgen     *idgen.Snowflake
    cfg       *config.Config
}

func NewShortener(redis *storage.RedisStore, mysql *storage.MySQLStore, idgen *idgen.Snowflake, cfg *config.Config) *Shortener {
    return &Shortener{
        redis: redis,
        mysql: mysql,
        idgen: idgen,
        cfg:   cfg,
    }
}

type CreateResult struct {
    ShortCode string `json:"short_code"`
    ShortURL  string `json:"short_url"`
    LongURL   string `json:"long_url"`
}

func (s *Shortener) Create(ctx context.Context, longURL string, ttl time.Duration) (*CreateResult, error) {
    // Generate unique ID and encode
    id := s.idgen.NextID()
    shortCode := base62.Encode(id)

    // Write to database
    mapping := &storage.URLMapping{
        ID:        id,
        ShortCode: shortCode,
        LongURL:   longURL,
        ExpireAt:  time.Now().Add(ttl),
    }

    if err := s.mysql.Save(ctx, mapping); err != nil {
        return nil, fmt.Errorf("save failed: %w", err)
    }

    // Write to Redis (for fast reads)
    if err := s.redis.Set(ctx, shortCode, longURL, ttl); err != nil {
        return nil, fmt.Errorf("cache failed: %w", err)
    }

    return &CreateResult{
        ShortCode: shortCode,
        ShortURL:  fmt.Sprintf("%s/%s", s.cfg.Server.BaseURL, shortCode),
        LongURL:   longURL,
    }, nil
}

func (s *Shortener) Resolve(ctx context.Context, shortCode string) (string, error) {
    // Check Redis first
    longURL, err := s.redis.Get(ctx, shortCode)
    if err == nil {
        return longURL, nil
    }

    // Redis miss, check database
    mapping, err := s.mysql.FindByShortCode(ctx, shortCode)
    if err != nil {
        return "", fmt.Errorf("short URL not found")
    }

    // Check expiration
    if !mapping.ExpireAt.IsZero() && time.Now().After(mapping.ExpireAt) {
        return "", fmt.Errorf("short URL has expired")
    }

    // Write back to Redis
    s.redis.Set(ctx, shortCode, mapping.LongURL, time.Until(mapping.ExpireAt))

    return mapping.LongURL, nil
}
```

## Step 6: Handler

```go
// internal/handler/shorten.go
package handler

type ShortenHandler struct {
    svc *service.Shortener
}

func NewShortenHandler(svc *service.Shortener) *ShortenHandler {
    return &ShortenHandler{svc: svc}
}

type CreateRequest struct {
    URL   string `json:"url" binding:"required,url"`
    TTL   int    `json:"ttl"`   // Expiration time (hours), 0 means never expire
}

func (h *ShortenHandler) Create(c *gin.Context) {
    var req CreateRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.Error(c, http.StatusBadRequest, "please provide a valid URL")
        return
    }

    ttl := 30 * 24 * time.Hour  // Default 30 days
    if req.TTL > 0 {
        ttl = time.Duration(req.TTL) * time.Hour
    }

    result, err := h.svc.Create(c.Request.Context(), req.URL, ttl)
    if err != nil {
        response.Error(c, http.StatusInternalServerError, err.Error())
        return
    }

    response.Created(c, result)
}
```

```go
// internal/handler/redirect.go
package handler

type RedirectHandler struct {
    svc *service.Shortener
}

func NewRedirectHandler(svc *service.Shortener) *RedirectHandler {
    return &RedirectHandler{svc: svc}
}

func (h *RedirectHandler) Redirect(c *gin.Context) {
    shortCode := c.Param("code")
    if shortCode == "" {
        response.Error(c, http.StatusBadRequest, "missing short code")
        return
    }

    longURL, err := h.svc.Resolve(c.Request.Context(), shortCode)
    if err != nil {
        response.Error(c, http.StatusNotFound, "short URL not found")
        return
    }

    // 301 permanent redirect (SEO friendly)
    c.Redirect(http.StatusMovedPermanently, longURL)
}
```

## Step 7: Main Entry

```go
// cmd/server/main.go
package main

import (
    "log"

    "github.com/gin-gonic/gin"
    "shorturl/internal/config"
    "shorturl/internal/handler"
    "shorturl/internal/service"
    "shorturl/internal/storage"
    "shorturl/pkg/idgen"
)

func main() {
    cfg, err := config.Load("config.yaml")
    if err != nil {
        log.Fatal("Failed to load config:", err)
    }

    // Initialize storage
    redis := storage.NewRedisStore(&cfg.Redis)
    mysql, err := storage.NewMySQLStore(cfg.Database.DSN)
    if err != nil {
        log.Fatal("Failed to connect to database:", err)
    }

    // Initialize ID generator
    idgen, err := idgen.New(1)
    if err != nil {
        log.Fatal("Failed to initialize ID generator:", err)
    }

    // Dependency injection
    shortener := service.NewShortener(redis, mysql, idgen, cfg)
    shortenHandler := handler.NewShortenHandler(shortener)
    redirectHandler := handler.NewRedirectHandler(shortener)

    // Routes
    r := gin.Default()

    api := r.Group("/api/v1")
    {
        api.POST("/shorten", shortenHandler.Create)
    }

    r.GET("/:code", redirectHandler.Redirect)

    log.Printf("URL Shortener starting :%d", cfg.Server.Port)
    r.Run(fmt.Sprintf(":%d", cfg.Server.Port))
}
```

## Docker Deployment

```dockerfile
# Dockerfile
FROM golang:1.23-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o server ./cmd/server

FROM alpine:3.19
WORKDIR /app
COPY --from=builder /app/server .
COPY --from=builder /app/config.yaml .
EXPOSE 8080
CMD ["./server"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - mysql
      - redis
  mysql:
    image: mysql:8.4
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: shorturl
  redis:
    image: redis:7-alpine
```

## Testing

```bash
# Create short URL
curl -X POST http://localhost:8080/api/v1/shorten \
  -H "Content-Type: application/json" \
  -d '{"url":"https://example.com/very/long/url/that/needs/shortening"}'

# Response: {"short_code":"abc123","short_url":"https://s.url/abc123"}

# Access short URL
curl -v http://localhost:8080/abc123
# 301 → https://example.com/very/long/url/that/needs/shortening
```

---

## Series Conclusion

25 tutorials have come to an end. From basic syntax to concurrent programming, from the standard library to real-world projects, covering all aspects of Go development.

| Part | Topic | Part | Topic |
|------|-------|------|-------|
| 1-2 | Environment setup, variables & types | 14-15 | Network programming, JSON |
| 3-4 | Flow control, functions | 16-17 | Reflection & generics, testing |
| 5-6 | Slices & maps, structs | 18-19 | Standard library, project structure |
| 7-8 | Interfaces, pointers | **20-25** | **6 Real-world projects** |
| 9-10 | Error handling, package management | | |
| 11-13 | Concurrency, I/O, files | | |

Wishing you continued success on your Go journey!

---

