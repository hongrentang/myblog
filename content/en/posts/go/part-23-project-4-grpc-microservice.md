---
title: "Go from Beginner to Pro · Part 23: Project 4 — gRPC Microservice"
date: 2025-01-16
weight: 1223
draft: false
tags: ["go"]
---

## Project Introduction

Build a complete gRPC microservice — User Service. Includes Protocol Buffers definitions, server implementation, client calls, health checks, and interceptors.

## Project Structure

```
usersvc/
├── cmd/
│   ├── server/main.go          # Server entry
│   └── client/main.go          # Client entry
├── internal/
│   └── service/
│       └── user.go             # Business logic
├── api/
│   └── proto/
│       ├── user/v1/
│       │   └── user.proto      # Proto definition
│       └── health/
│           └── health.proto
├── pkg/
│   └── interceptors/
│       └── logging.go          # Interceptor
├── gen/                        # Generated Go code
│   └── proto/                  (protoc generated)
├── buf.yaml                    # Buf configuration
├── buf.gen.yaml
├── go.mod
└── Makefile
```

## Step 1: Proto Definition

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

## Step 2: Buf Configuration

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

## Step 3: Generate Code

```bash
# Install tools
go install github.com/bufbuild/buf/cmd/buf@latest
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Generate code
buf generate
```

## Step 4: Logging Interceptor

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

## Step 5: Server Implementation

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
        return nil, fmt.Errorf("user %d not found", req.Id)
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
        return nil, fmt.Errorf("user %d not found", req.Id)
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

## Step 6: Server Entry

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
        log.Fatal("Failed to listen:", err)
    }

    store := service.NewUserStore()
    userServer := service.NewUserServer(store)

    s := grpc.NewServer(
        grpc.UnaryInterceptor(interceptors.LoggingInterceptor),
    )

    userv1.RegisterUserServiceServer(s, userServer)
    reflection.Register(s)  // Debug convenience

    log.Println("gRPC server starting :50051")
    if err := s.Serve(lis); err != nil {
        log.Fatal("Failed to serve:", err)
    }
}
```

## Step 7: Client

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
        log.Fatal("Failed to connect:", err)
    }
    defer conn.Close()

    client := userv1.NewUserServiceClient(conn)
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // 1. Create user
    createResp, err := client.CreateUser(ctx, &userv1.CreateUserRequest{
        Username: "Zhang San",
        Email:    "zhangsan@example.com",
    })
    if err != nil {
        log.Fatal("Failed to create user:", err)
    }
    fmt.Printf("Created user: %+v\n", createResp.User)

    // 2. Get user
    getResp, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: createResp.User.Id})
    if err != nil {
        log.Fatal("Failed to get user:", err)
    }
    fmt.Printf("User info: %+v\n", getResp.User)

    // 3. List users
    listResp, err := client.ListUsers(ctx, &userv1.ListUsersRequest{Page: 1, PageSize: 10})
    if err != nil {
        log.Fatal("Failed to list users:", err)
    }
    fmt.Printf("Users: %d total\n", listResp.Total)
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

## Extension Ideas

- Integrate etcd for service discovery and registration
- Add TLS encryption
- Add streaming RPC (Server/Client/Bidirectional)
- Integrate database instead of in-memory storage

---

