---
title: "Go from Beginner to Pro · Part 10: Package Management and Modules"
date: 2025-01-02
weight: 10
draft: false
tags: ["go"]
---

## 1. Go Module Introduction

Go Module is the official dependency management solution introduced in Go 1.11, and has been the default since Go 1.16.

**Core Files**:

```
myproject/
├── go.mod      # Module definition and dependency declarations
├── go.sum      # Dependency checksums (auto-managed)
└── main.go
```

---

## 2. go.mod File

### Initialization

```bash
# Create new project
mkdir myapp && cd myapp
go mod init myapp

# Generated go.mod:
module myapp

go 1.23
```

### Adding Dependencies

```bash
# Method 1: go get
go get github.com/gin-gonic/gin

# Method 2: import then go mod tidy
# Import "github.com/gin-gonic/gin" in code
go mod tidy  # Automatically adds missing dependencies, removes unused ones

# Install specific version
go get github.com/gin-gonic/gin@v1.10.0
```

### go.mod Content Example

```
module github.com/user/myapp

go 1.23

require (
    github.com/gin-gonic/gin v1.10.0
    github.com/go-sql-driver/mysql v1.8.1
)

require (
    github.com/bytedance/sonic v1.11.6 // indirect
    github.com/gabriel-vasile/mimetype v1.4.3 // indirect
)
```

- `indirect`: Indirect dependencies (not directly imported)
- Comments after `//` are auto-generated

---

## 3. Package Import

### Import Syntax

```go
// Single import
import "fmt"

// Grouped import
import (
    "fmt"
    "os"

    "github.com/gin-gonic/gin"
    "github.com/go-sql-driver/mysql"
)

// Alias
import f "fmt"

// Dot import (not recommended, pollutes namespace)
import . "fmt"

// Anonymous import (only executes init, doesn't use package identifiers)
import _ "github.com/go-sql-driver/mysql"
```

### Import Paths

```go
// Standard library
import "fmt"
import "net/http"
import "encoding/json"

// Third-party
import "github.com/gin-gonic/gin"

// Internal packages
import "myapp/internal/database"
import "myapp/pkg/utils"
```

---

## 4. Package Visibility

Go controls visibility through **capitalization of the first letter**:

```go
// Package myapp/pkg/utils

// Capitalized first letter = exported (public)
func PublicFunc() {}

// Lowercase first letter = unexported (private)
func privateFunc() {}

// Same applies to variables, constants, structs, interfaces
var PublicVar = 1
var privateVar = 2

type PublicStruct struct{}
type privateStruct struct{}
```

### Within Package vs Outside Package

```go
// Same package, can access everything
// a.go
package utils
var internalVar = 10

// b.go (same package)
package utils
func getVar() int {
    return internalVar  // ✅ Same package, private is accessible
}
```

---

## 5. Package Organization

### Standard Go Project Layout

```
myapp/
├── go.mod
├── go.sum
├── main.go                    # Entry file
├── cmd/                       # Executables
│   └── server/
│       └── main.go
├── internal/                  # Private packages (cannot be imported externally)
│   ├── database/
│   │   └── db.go
│   └── config/
│       └── config.go
├── pkg/                       # Public packages
│   └── response/
│       └── response.go
├── api/                       # API definitions
├── web/                       # Frontend resources
└── scripts/                   # Build scripts
```

### The Special internal Package

Packages under the `internal` directory can only be imported by code within its parent directory tree:

```
myapp/
├── internal/     # Only code within myapp can import
│   └── secret/
└── main.go       # ✅ Can import internal/secret

other/
└── main.go       # ❌ Cannot import myapp/internal/secret
```

---

## 6. Dependency Management

### Common Commands

```bash
# Add dependency
go get github.com/gin-gonic/gin

# Update dependency
go get -u github.com/gin-gonic/gin
go get -u ./...       # Update all dependencies

# Clean dependencies
go mod tidy

# Download all dependencies to local cache
go mod download

# View dependencies
go list -m all                 # All dependencies
go list -m -versions github.com/gin-gonic/gin  # View available versions

# View why a package is depended on
go mod why github.com/gin-gonic/gin

# Verify dependencies
go mod verify

# Visualize dependency graph
go mod graph
```

### Version Management

```bash
# Upgrade to minor version
go get github.com/gin-gonic/gin@latest

# Specify version
go get github.com/gin-gonic/gin@v1.10.0

# Upgrade to v2 major version
go get github.com/gin-gonic/gin/v2
```

### Replacing Local Dependencies

```go
// Using replace directive in go.mod
module myapp

go 1.23

require (
    github.com/example/mypackage v0.0.0
)

// Replace with local path
replace github.com/example/mypackage => ../mypackage
```

---

## 7. vendor Directory

```bash
# Copy dependencies to vendor directory
go mod vendor

# Build using vendor
go build -mod=vendor

# Useful when the project needs an offline environment for building
```

**go vendor and go.sum together**: vendor directory records a complete copy of dependencies, go.sum records integrity hashes.

---

## 7. Common Issues

### 1. Import Conflicts

```go
// Two different packages with the same name
import (
    "crypto/rand"
    "math/rand"
)

// Use aliases to resolve
import (
    cryptorand "crypto/rand"
    "math/rand"
)
```

### 2. Circular Imports

```
a.go import b  →  b.go import a  →  compilation error
```

**Solutions**:
1. Extract common interfaces to a third package
2. Adjust package responsibilities to eliminate circular dependencies

### 3. Private Repositories

```bash
# Go proxy does not support private repositories
GOPROXY=direct

# Or configure private repos not to use proxy
go env -w GOPRIVATE=github.com/mycompany/*
go env -w GONOSUMCHECK=github.com/mycompany/*  # Skip checksum verification
```

---

## 8. Go Proxy

```bash
# Default proxy
go env -w GOPROXY=https://proxy.golang.org,direct

# China mirror (recommended for users in China)
go env -w GOPROXY=https://goproxy.cn,direct

# Direct connection (no proxy)
go env -w GOPROXY=direct

# Multiple proxies
go env -w GOPROXY=https://goproxy.cn,https://goproxy.io,direct
```

---

## 9. Build and Cross-Platform Compilation

```bash
# Build for current platform
go build -o myapp main.go

# Linux
GOOS=linux GOARCH=amd64 go build -o myapp-linux main.go

# Windows
GOOS=windows GOARCH=amd64 go build -o myapp.exe main.go

# macOS
GOOS=darwin GOARCH=amd64 go build -o myapp-mac main.go
GOOS=darwin GOARCH=arm64 go build -o myapp-mac-m1 main.go  # Apple Silicon

# Disable CGO
CGO_ENABLED=0 go build -o myapp main.go

# Reduce binary size
go build -ldflags="-s -w" -o myapp main.go

# View supported platforms
go tool dist list
```

---

## Wrap-up

Package management is the foundation of Go project organization. Go Module makes dependency management simple: `go mod init` to initialize, `go get` to add dependencies, `go mod tidy` to clean up. A sensible package structure (internal, pkg) makes code organization clearer.

---

