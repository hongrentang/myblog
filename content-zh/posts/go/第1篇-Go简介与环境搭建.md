---
title: "Go 从入门到精通 · 第1篇：Go 简介与环境搭建"
date: 2025-01-12
weight: 1201
draft: false
tags: ["go"]
featured: true
cover:
  image: "/images/go-banner.svg"
  caption: "Go 从入门到精通"
---

## Go 是什么？

Go（又称 Golang）是 Google 于 2009 年开源的一门编程语言，由 Robert Griesemer、Rob Pike、Ken Thompson 设计。

**设计目标**：

```
编译型语言的性能 + 动态语言的开发效率 + 现代化的并发模型
```

### 核心特点

| 特点 | 说明 |
|------|------|
| 简洁语法 | 只有 25 个关键字，没有类/继承/泛型（1.18+ 引入泛型）|
| 快速编译 | 编译速度极快，秒级完成 |
| 原生并发 | goroutine + channel，轻量级协程 |
| 静态类型 | 编译时检查类型安全 |
| 内置工具链 | go fmt、go test、pprof、go mod |
| 交叉编译 | 一行命令编译到不同平台 |
| 内存安全 | GC 自动管理内存，无野指针 |

### 谁在用 Go？

| 项目/公司 | 用途 |
|-----------|------|
| Docker / Kubernetes | 容器化和编排 |
| Prometheus / Grafana | 监控系统 |
| TiDB / CockroachDB | 分布式数据库 |
| 阿里云 / 腾讯云 | 云原生基础设施 |
| 字节跳动 / 哔哩哔哩 | 微服务和中间件 |

---

## 安装 Go

### Linux

```bash
# 下载最新版本（以 1.23 为例）
wget https://go.dev/dl/go1.23.0.linux-amd64.tar.gz

# 解压到 /usr/local
sudo tar -C /usr/local -xzf go1.23.0.linux-amd64.tar.gz

# 添加到 PATH
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
```

### macOS

```bash
# Homebrew
brew install go
```

### Windows

下载 MSI 安装包：https://go.dev/dl/

### 验证安装

```bash
go version
# go version go1.23.0 linux/amd64
```

---

## Go 环境配置

### Go Module（现代 Go 必备）

从 Go 1.16 起，`GO111MODULE` 默认为 `on`，推荐使用 module 模式。

```bash
# 设置 GOPROXY（国内必配）
go env -w GOPROXY=https://goproxy.cn,direct

# 查看环境
go env
```

### 基础环境变量

| 变量 | 说明 | 推荐值 |
|------|------|--------|
| `GOROOT` | Go 安装路径 | `/usr/local/go` |
| `GOPATH` | 工作目录 | `~/go`（默认）|
| `GOBIN` | 编译后的二进制路径 | `$GOPATH/bin` |
| `GOPROXY` | 模块代理 | `https://goproxy.cn,direct` |

---

## 第一个程序：Hello World

```go
// main.go
package main

import "fmt"

func main() {
    fmt.Println("Hello, Go!")
}
```

### 运行

```bash
# 直接运行
go run main.go
# 输出：Hello, Go!

# 编译为二进制
go build -o hello main.go
./hello
```

### 代码结构说明

```
┌─────────────────────────────────────┐
│ package main       // 包声明         │
│                                       │
│ import "fmt"       // 导入包          │
│                                       │
│ func main() {      // 入口函数        │
│     fmt.Println("Hello, Go!")        │
│ }                   // 大括号换行风格  │
└─────────────────────────────────────┘
```

**注意**：Go 强制 `{` 放在行尾，不能另起一行。

---

## 初始化模块项目

```bash
# 创建项目目录
mkdir myapp
cd myapp

# 初始化模块
go mod init myapp

# 这会生成 go.mod 文件
# module myapp
# go 1.23

# 编写代码
touch main.go
```

### go.mod 文件

```
module myapp

go 1.23

require (
    github.com/gin-gonic/gin v1.9.1
)
```

### 添加依赖

```bash
# 自动添加依赖并生成 go.sum
go get github.com/gin-gonic/gin

# 整理 go.mod
go mod tidy
```

---

## IDE 配置

### VS Code

安装 Go 插件（gopls 语言服务器）：

```bash
# 安装 gopls（VS Code 会自动提示安装）
go install golang.org/x/tools/goud/v2/cmd/goud@latest

# 推荐的 VS Code settings.json
{
    "go.useLanguageServer": true,
    "go.gopath": "/home/user/go",
    "editor.formatOnSave": true
}
```

### GoLand

JetBrains 出品的专业 Go IDE，开箱即用。

---

## Go Tool 入门

```bash
# 格式化代码（Go 没有缩进之争，go fmt 统一风格）
go fmt ./...

# 检查代码问题
go vet ./...

# 构建
go build ./...

# 测试
go test ./...

# 安装依赖
go mod tidy

# 查看文档
go doc fmt.Println
```

---

## 总结：Go 的哲学

| Go 的设计理念 | 对应实践 |
|-------------|---------|
| 显式优于隐式 | 没有异常机制，显式处理错误 |
| 约定优于配置 | go fmt 统一格式|
| 组合优于继承 | 接口 + 结构体组合 |
| 并发是原生能力 | goroutine + channel |
| 工具链完善 | go fmt/vet/test/mod |

---

## 小结

这一篇介绍了 Go 语言的背景和特点、环境搭建以及项目初始化。

---

