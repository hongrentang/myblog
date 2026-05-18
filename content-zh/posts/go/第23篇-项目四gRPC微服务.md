---
title: "Go 从入门到精通 · 第23篇：项目四 —— gRPC 微服务"
date: 2025-01-16
weight: 1223
draft: false
tags: ["go"]
---

## 项目介绍

构建一个完整的 gRPC 微服务——用户服务（User Service）。包含 Protocol Buffers 定义、服务端实现、客户端调用、健康检查和拦截器。

## 项目结构

```
usersvc/
├── cmd/
│   ├── server/main.go          # 服务端入口
│   └── client/main.go          # 客户端入口
├── internal/
│   └── service/
│       └── user.go             # 业务逻辑
├── api/
│   └── proto/
│       ├── user/v1/
│       │   └── user.proto      # Proto 定义
│       └── health/
│           └── health.proto
├── pkg/
│   └── interceptors/
│       └── logging.go          # 拦截器
├── gen/                        # 生成的 Go 代码
│   └── proto/                  （protoc 生成）
├── buf.yaml                    # Buf 配置
├── buf.gen.yaml
├── go.mod
└── Makefile
```

## 第一步：Proto 定义

```protobuf
// api/proto/user/v1/user.proto
syntax = "proto3";

package user.v1;

option go_package = "usersvc/gen/proto/user/v1;userv1";

service UserService {
    rpc GetUser(GetUserRequest) returns (GetUserResponse);
    rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
    rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
    rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse);
    rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
}

message User {
    int64 id = 1;
    string username = 2;
    string email = 3;
    string nickname = 4;
    int32 status = 5;
    string created_at = 6;
}

message GetUserRequest {
    int64 id = 1;
}

message GetUserResponse {
    User user = 1;
}

message ListUsersRequest {
    int32 page = 1;
    int32 page_size = 2;
}

message ListUsersResponse {
    repeated User users = 1;
    int32 total = 2;
}

message CreateUserRequest {
    string username = 1;
    string email = 2;
    string password = 3;
}

message CreateUserResponse {
    User user = 1;
}

message UpdateUserRequest {
    int64 id = 1;
    string nickname = 2;
    int32 status = 3;
}

message UpdateUserResponse {
    User user = 1;
}

message DeleteUserRequest {
    int64 id = 1;
}

message DeleteUserResponse {
    bool success = 1;
}
```

## 第二步：Buf 配置

```yaml
# buf.yaml
version: v2
modules:
  - path: api/proto
lint:
  use:
    - STANDARD
breaking:
  use:
    - FILE
```

```yaml
# buf.gen.yaml
version: v2
plugins:
  - local: protoc-gen-go
    out: gen
    opt: paths=source_relative
  - local: protoc-gen-go-grpc
    out: gen
    opt: paths=source_relative
```

## 第三步：生成代码

```bash
# 安装工具
go install github.com/bufbuild/buf/cmd/buf@latest
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# 生成代码
buf generate
```

## 第四步：日志拦截器

```go
// pkg/interceptors/logging.go
package interceptors

import (
    "context"
    "log"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/status"
)

func LoggingInterceptor(ctx context.Context, req interface{},
    info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {

    start := time.Now()

    resp, err := handler(ctx, req)

    duration := time.Since(start)
    code := status.Code(err)

    log.Printf("[gRPC] %s | %v | %v", info.FullMethod, code, duration)

    return resp, err
}
```

## 第五步：服务端实现

```go
// internal/service/user.go
package service

import (
    "context"
    "fmt"
    "sync"

    userv1 "usersvc/gen/proto/user/v1"
)

type UserStore struct {
    mu     sync.RWMutex
    users  map[int64]*userv1.User
    nextID int64
}

func NewUserStore() *UserStore {
    return &UserStore{
        users:  make(map[int64]*userv1.User),
        nextID: 1,
    }
}

type UserServer struct {
    userv1.UnimplementedUserServiceServer
    store *UserStore
}

func NewUserServer(store *UserStore) *UserServer {
    return &UserServer{store: store}
}

func (s *UserServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
    s.store.mu.RLock()
    user, ok := s.store.users[req.Id]
    s.store.mu.RUnlock()

    if !ok {
        return nil, fmt.Errorf("用户 %d 不存在", req.Id)
    }

    return &userv1.GetUserResponse{User: user}, nil
}

func (s *UserServer) ListUsers(ctx context.Context, req *userv1.ListUsersRequest) (*userv1.ListUsersResponse, error) {
    s.store.mu.RLock()
    defer s.store.mu.RUnlock()

    var users []*userv1.User
    for _, u := range s.store.users {
        users = append(users, u)
    }

    return &userv1.ListUsersResponse{
        Users: users,
        Total: int32(len(users)),
    }, nil
}

func (s *UserServer) CreateUser(ctx context.Context, req *userv1.CreateUserRequest) (*userv1.CreateUserResponse, error) {
    s.store.mu.Lock()
    defer s.store.mu.Unlock()

    id := s.nextID
    s.nextID++

    user := &userv1.User{
        Id:       id,
        Username: req.Username,
        Email:    req.Email,
        Status:   1,
    }

    s.store.users[id] = user

    return &userv1.CreateUserResponse{User: user}, nil
}

func (s *UserServer) UpdateUser(ctx context.Context, req *userv1.UpdateUserRequest) (*userv1.UpdateUserResponse, error) {
    s.store.mu.Lock()
    defer s.store.mu.Unlock()

    user, ok := s.store.users[req.Id]
    if !ok {
        return nil, fmt.Errorf("用户 %d 不存在", req.Id)
    }

    if req.Nickname != "" {
        user.Nickname = req.Nickname
    }
    if req.Status != 0 {
        user.Status = req.Status
    }

    return &userv1.UpdateUserResponse{User: user}, nil
}

func (s *UserServer) DeleteUser(ctx context.Context, req *userv1.DeleteUserRequest) (*userv1.DeleteUserResponse, error) {
    s.store.mu.Lock()
    defer s.store.mu.Unlock()

    delete(s.store.users, req.Id)

    return &userv1.DeleteUserResponse{Success: true}, nil
}
```

## 第六步：服务端入口

```go
// cmd/server/main.go
package main

import (
    "log"
    "net"

    "google.golang.org/grpc"
    "google.golang.org/grpc/reflection"
    userv1 "usersvc/gen/proto/user/v1"
    "usersvc/internal/service"
    "usersvc/pkg/interceptors"
)

func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatal("监听失败:", err)
    }

    store := service.NewUserStore()
    userServer := service.NewUserServer(store)

    s := grpc.NewServer(
        grpc.UnaryInterceptor(interceptors.LoggingInterceptor),
    )

    userv1.RegisterUserServiceServer(s, userServer)
    reflection.Register(s)  // 方便调试

    log.Println("gRPC 服务启动 :50051")
    if err := s.Serve(lis); err != nil {
        log.Fatal("服务启动失败:", err)
    }
}
```

## 第七步：客户端

```go
// cmd/client/main.go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    userv1 "usersvc/gen/proto/user/v1"
)

func main() {
    conn, err := grpc.NewClient("localhost:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        log.Fatal("连接失败:", err)
    }
    defer conn.Close()

    client := userv1.NewUserServiceClient(conn)
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // 1. 创建用户
    createResp, err := client.CreateUser(ctx, &userv1.CreateUserRequest{
        Username: "张三",
        Email:    "zhangsan@example.com",
    })
    if err != nil {
        log.Fatal("创建用户失败:", err)
    }
    fmt.Printf("创建用户: %+v\n", createResp.User)

    // 2. 查询用户
    getResp, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: createResp.User.Id})
    if err != nil {
        log.Fatal("查询用户失败:", err)
    }
    fmt.Printf("用户信息: %+v\n", getResp.User)

    // 3. 用户列表
    listResp, err := client.ListUsers(ctx, &userv1.ListUsersRequest{Page: 1, PageSize: 10})
    if err != nil {
        log.Fatal("查询列表失败:", err)
    }
    fmt.Printf("用户列表: %d 个\n", listResp.Total)
}
```

## Makefile

```makefile
.PHONY: gen server client

gen:
	buf generate

server:
	go run cmd/server/main.go

client:
	go run cmd/client/main.go
```

## 扩展方向

- 集成 etcd 实现服务注册与发现
- 添加 TLS 加密传输
- 添加流式 RPC（Server/Client/Bidirectional streaming）
- 集成数据库替代内存存储

---

