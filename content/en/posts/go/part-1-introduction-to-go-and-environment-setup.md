---
title: "Go from Beginner to Pro · Part 1: Introduction to Go and Environment Setup"
date: 2025-01-12
weight: 1
draft: false
tags: ["go"]
featured: true
cover:
  image: "/images/go-banner.svg"
  caption: "Go from Beginner to Pro"
---

## What is Go?

Go (also known as Golang) is a programming language open-sourced by Google in 2009, designed by Robert Griesemer, Rob Pike, and Ken Thompson.

**Design Goals**:

```
Performance of compiled languages + Development efficiency of dynamic languages + Modern concurrency model
```

### Key Features

| Feature | Description |
|---------|-------------|
| Simple Syntax | Only 25 keywords, no classes/inheritance/generics (generics introduced in 1.18+) |
| Fast Compilation | Extremely fast compilation, completes in seconds |
| Built-in Concurrency | goroutine + channel, lightweight coroutines |
| Static Typing | Type safety checked at compile time |
| Built-in Toolchain | go fmt, go test, pprof, go mod |
| Cross-compilation | Compile to different platforms with one command |
| Memory Safety | GC manages memory automatically, no dangling pointers |

### Who Uses Go?

| Project/Company | Use Case |
|----------------|----------|
| Docker / Kubernetes | Containerization and orchestration |
| Prometheus / Grafana | Monitoring systems |
| TiDB / CockroachDB | Distributed databases |
| Alibaba Cloud / Tencent Cloud | Cloud-native infrastructure |
| ByteDance / Bilibili | Microservices and middleware |

---

## Installing Go

### Linux

```bash
# Download the latest version (using 1.23 as example)
wget https://go.dev/dl/go1.23.0.linux-amd64.tar.gz

# Extract to /usr/local
sudo tar -C /usr/local -xzf go1.23.0.linux-amd64.tar.gz

# Add to PATH
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
```

### macOS

```bash
# Homebrew
brew install go
```

### Windows

Download the MSI installer: https://go.dev/dl/

### Verify Installation

```bash
go version
# go version go1.23.0 linux/amd64
```

---

## Go Environment Configuration

### Go Module (Essential for Modern Go)

Since Go 1.16, `GO111MODULE` defaults to `on`, and using module mode is recommended.

```bash
# Set GOPROXY (essential for users in China)
go env -w GOPROXY=https://goproxy.cn,direct

# View environment
go env
```

### Basic Environment Variables

| Variable | Description | Recommended Value |
|----------|-------------|-------------------|
| `GOROOT` | Go installation path | `/usr/local/go` |
| `GOPATH` | Workspace directory | `~/go` (default) |
| `GOBIN` | Compiled binary path | `$GOPATH/bin` |
| `GOPROXY` | Module proxy | `https://goproxy.cn,direct` |

---

## First Program: Hello World

```go
// main.go
package main

import "fmt"

func main() {
    fmt.Println("Hello, Go!")
}
```

### Running

```bash
# Run directly
go run main.go
# Output: Hello, Go!

# Compile to binary
go build -o hello main.go
./hello
```

### Code Structure Explanation

```
┌─────────────────────────────────────┐
│ package main       // Package declaration│
│                                       │
│ import "fmt"       // Import package   │
│                                       │
│ func main() {      // Entry function    │
│     fmt.Println("Hello, Go!")        │
│ }                   // Brace on same line│
└─────────────────────────────────────┘
```

**Note**: Go requires `{` to be on the same line, it cannot start a new line.

---

## Initializing a Module Project

```bash
# Create project directory
mkdir myapp
cd myapp

# Initialize module
go mod init myapp

# This generates go.mod file
# module myapp
# go 1.23

# Write code
touch main.go
```

### go.mod File

```
module myapp

go 1.23

require (
    github.com/gin-gonic/gin v1.9.1
)
```

### Adding Dependencies

```bash
# Automatically add dependencies and generate go.sum
go get github.com/gin-gonic/gin

# Tidy go.mod
go mod tidy
```

---

## IDE Configuration

### VS Code

Install the Go plugin (gopls language server):

```bash
# Install gopls (VS Code will prompt you to install it automatically)
go install golang.org/x/tools/goud/v2/cmd/goud@latest

# Recommended VS Code settings.json
{
    "go.useLanguageServer": true,
    "go.gopath": "/home/user/go",
    "editor.formatOnSave": true
}
```

### GoLand

JetBrains' professional Go IDE, ready to use out of the box.

---

## Go Tool Introduction

```bash
# Format code (Go has no indentation debates, go fmt unifies style)
go fmt ./...

# Check code issues
go vet ./...

# Build
go build ./...

# Test
go test ./...

# Install dependencies
go mod tidy

# View documentation
go doc fmt.Println
```

---

## Summary: Go Philosophy

| Go Design Principle | Corresponding Practice |
|--------------------|----------------------|
| Explicit over implicit | No exception mechanism, explicit error handling |
| Convention over configuration | go fmt unifies formatting |
| Composition over inheritance | Interfaces + struct composition |
| Concurrency as a built-in capability | goroutine + channel |
| Complete toolchain | go fmt/vet/test/mod |

---

## Wrap-up

This article introduced the background and features of Go, environment setup, and project initialization.

---

