---
title: "Go 从入门到精通 · 第22篇：项目三 —— RESTful API 服务"
date: 2025-01-15
weight: 22
draft: false
tags: ["go"]
---

## 项目介绍

使用 Gin + GORM + JWT 构建完整的用户管理系统 RESTful API，涵盖认证、CRUD、中间件、Swagger 文档。

## 项目结构

```
userapi/
├── cmd/server/main.go
├── internal/
│   ├── config/
│   │   └── config.go         # 配置
│   ├── middleware/
│   │   ├── auth.go           # JWT 认证
│   │   └── cors.go           # 跨域
│   ├── handler/
│   │   ├── user.go           # 用户处理器
│   │   └── auth.go           # 认证处理器
│   ├── service/
│   │   ├── user.go
│   │   └── auth.go
│   ├── repository/
│   │   └── user.go
│   └── model/
│       └── user.go
├── pkg/
│   └── response/response.go  # 统一响应
├── go.mod
└── Makefile
```

## 第一步：数据模型

```go
// internal/model/user.go
package model

import "time"

type User struct {
    ID        uint      `gorm:"primarykey" json:"id"`
    Username  string    `gorm:"uniqueIndex;size:50;not null" json:"username" binding:"required"`
    Email     string    `gorm:"uniqueIndex;size:100;not null" json:"email" binding:"required,email"`
    Password  string    `gorm:"not null" json:"-"`  // JSON 忽略密码
    Nickname  string    `gorm:"size:50" json:"nickname"`
    Avatar    string    `gorm:"size:255" json:"avatar"`
    Status    int       `gorm:"default:1" json:"status"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}
```

## 第二步：配置

```go
// internal/config/config.go
package config

import (
    "os"
    "time"
)

type Config struct {
    Server   ServerConfig
    Database DatabaseConfig
    JWT      JWTConfig
}

type ServerConfig struct {
    Port string
}

type DatabaseConfig struct {
    DSN string
}

type JWTConfig struct {
    Secret  string
    Expires time.Duration
}

func Load() *Config {
    return &Config{
        Server: ServerConfig{
            Port: getEnv("SERVER_PORT", "8080"),
        },
        Database: DatabaseConfig{
            DSN: getEnv("DB_DSN", "root:password@tcp(127.0.0.1:3306)/userapi?charset=utf8mb4&parseTime=True"),
        },
        JWT: JWTConfig{
            Secret:  getEnv("JWT_SECRET", "your-secret-key"),
            Expires: 24 * time.Hour,
        },
    }
}

func getEnv(key, fallback string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return fallback
}
```

## 第三步：统一响应

```go
// pkg/response/response.go
package response

import (
    "net/http"

    "github.com/gin-gonic/gin"
)

type Response struct {
    Code    int         `json:"code"`
    Message string      `json:"message"`
    Data    interface{} `json:"data,omitempty"`
}

func Success(c *gin.Context, data interface{}) {
    c.JSON(http.StatusOK, Response{Code: 0, Message: "success", Data: data})
}

func Created(c *gin.Context, data interface{}) {
    c.JSON(http.StatusCreated, Response{Code: 0, Message: "created", Data: data})
}

func Error(c *gin.Context, status int, msg string) {
    c.JSON(status, Response{Code: status, Message: msg})
}
```

## 第四步：JWT 中间件

```go
// internal/middleware/auth.go
package middleware

import (
    "net/http"
    "strings"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/golang-jwt/jwt/v5"
)

type Claims struct {
    UserID   uint   `json:"user_id"`
    Username string `json:"username"`
    jwt.RegisteredClaims
}

func GenerateToken(secret string, userID uint, username string, expires time.Duration) (string, error) {
    claims := Claims{
        UserID:   userID,
        Username: username,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(expires)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(secret))
}

func ParseToken(secret, tokenStr string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenStr, &Claims{}, func(t *jwt.Token) (interface{}, error) {
        return []byte(secret), nil
    })
    if err != nil {
        return nil, err
    }
    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }
    return nil, jwt.ErrSignatureInvalid
}

// Gin 中间件
func AuthMiddleware(secret string) gin.HandlerFunc {
    return func(c *gin.Context) {
        auth := c.GetHeader("Authorization")
        if !strings.HasPrefix(auth, "Bearer ") {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "未提供令牌"})
            return
        }

        claims, err := ParseToken(secret, auth[7:])
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "令牌无效"})
            return
        }

        c.Set("user_id", claims.UserID)
        c.Set("username", claims.Username)
        c.Next()
    }
}
```

## 第五步：认证处理器

```go
// internal/handler/auth.go
package handler

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "userapi/internal/config"
    "userapi/internal/middleware"
    "userapi/internal/model"
    "userapi/internal/service"
    "userapi/pkg/response"
)

type AuthHandler struct {
    svc *service.UserService
    cfg *config.Config
}

func NewAuthHandler(svc *service.UserService, cfg *config.Config) *AuthHandler {
    return &AuthHandler{svc: svc, cfg: cfg}
}

type LoginRequest struct {
    Username string `json:"username" binding:"required"`
    Password string `json:"password" binding:"required"`
}

type RegisterRequest struct {
    Username string `json:"username" binding:"required,min=3,max=50"`
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=6"`
}

func (h *AuthHandler) Register(c *gin.Context) {
    var req RegisterRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.Error(c, http.StatusBadRequest, err.Error())
        return
    }

    user, err := h.svc.Register(c.Request.Context(), req.Username, req.Email, req.Password)
    if err != nil {
        response.Error(c, http.StatusConflict, err.Error())
        return
    }

    token, _ := middleware.GenerateToken(h.cfg.JWT.Secret, user.ID, user.Username, h.cfg.JWT.Expires)
    response.Created(c, gin.H{"user": user, "token": token})
}

func (h *AuthHandler) Login(c *gin.Context) {
    var req LoginRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.Error(c, http.StatusBadRequest, err.Error())
        return
    }

    user, err := h.svc.Login(c.Request.Context(), req.Username, req.Password)
    if err != nil {
        response.Error(c, http.StatusUnauthorized, err.Error())
        return
    }

    token, _ := middleware.GenerateToken(h.cfg.JWT.Secret, user.ID, user.Username, h.cfg.JWT.Expires)
    response.Success(c, gin.H{"user": user, "token": token})
}
```

## 第六步：Service 层

```go
// internal/service/user.go
package service

import (
    "errors"
    "strings"

    "golang.org/x/crypto/bcrypt"
    "userapi/internal/model"
    "userapi/internal/repository"
)

type UserService struct {
    repo *repository.UserRepo
}

func NewUserService(repo *repository.UserRepo) *UserService {
    return &UserService{repo: repo}
}

func (s *UserService) Register(ctx interface{}, username, email, password string) (*model.User, error) {
    username = strings.TrimSpace(username)
    email = strings.TrimSpace(email)

    if s.repo.ExistsByUsername(username) {
        return nil, errors.New("用户名已存在")
    }
    if s.repo.ExistsByEmail(email) {
        return nil, errors.New("邮箱已被注册")
    }

    hashed, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    if err != nil {
        return nil, err
    }

    user := &model.User{
        Username: username,
        Email:    email,
        Password: string(hashed),
        Nickname: username,
        Status:   1,
    }

    if err := s.repo.Create(user); err != nil {
        return nil, err
    }

    user.Password = ""  // 不要返回密码
    return user, nil
}

func (s *UserService) Login(ctx interface{}, username, password string) (*model.User, error) {
    user, err := s.repo.FindByUsername(username)
    if err != nil {
        return nil, errors.New("用户名或密码错误")
    }

    if err := bcrypt.CompareHashAndPassword([]byte(user.Password), []byte(password)); err != nil {
        return nil, errors.New("用户名或密码错误")
    }

    user.Password = ""
    return user, nil
}
```

## 第七步：主入口

```go
// cmd/server/main.go
package main

import (
    "log"

    "github.com/gin-gonic/gin"
    "gorm.io/driver/mysql"
    "gorm.io/gorm"
    "userapi/internal/config"
    "userapi/internal/handler"
    "userapi/internal/middleware"
    "userapi/internal/model"
    "userapi/internal/repository"
    "userapi/internal/service"
)

func main() {
    cfg := config.Load()

    // 连接数据库
    db, err := gorm.Open(mysql.Open(cfg.Database.DSN), &gorm.Config{})
    if err != nil {
        log.Fatal("数据库连接失败:", err)
    }
    db.AutoMigrate(&model.User{})

    // 依赖注入
    userRepo := repository.NewUserRepo(db)
    userSvc := service.NewUserService(userRepo)
    authHandler := handler.NewAuthHandler(userSvc, cfg)
    userHandler := handler.NewUserHandler(userSvc)

    // 路由
    r := gin.Default()
    r.Use(middleware.CORS())

    api := r.Group("/api/v1")
    {
        // 公开接口
        api.POST("/auth/register", authHandler.Register)
        api.POST("/auth/login", authHandler.Login)

        // 需要认证的接口
        auth := api.Group("")
        auth.Use(middleware.AuthMiddleware(cfg.JWT.Secret))
        {
            auth.GET("/users/me", userHandler.GetProfile)
            auth.PUT("/users/me", userHandler.UpdateProfile)
            auth.GET("/users", userHandler.List)
        }
    }

    log.Printf("服务启动 :%s", cfg.Server.Port)
    r.Run(":" + cfg.Server.Port)
}
```

## Makefile

```makefile
.PHONY: run build test

run:
	go run cmd/server/main.go

build:
	go build -o bin/server cmd/server/main.go

test:
	go test ./... -v

migrate:
	go run cmd/migrate/main.go
```

## 运行

```bash
go mod init userapi
go mod tidy
go run cmd/server/main.go

# 测试
curl -X POST http://localhost:8080/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","email":"admin@test.com","password":"123456"}'
```

---

