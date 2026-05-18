---
title: "Go from Beginner to Pro · Part 17: Testing and Performance Analysis"
date: 2025-01-09
weight: 17
draft: false
tags: ["go"]
---

## 1. Unit Testing

### Test File Naming

```
xxx_test.go
```

Test files should be placed in the same package as the code being tested.

### Basic Tests

```go
// math.go
package math

func Add(a, b int) int {
    return a + b
}

func Divide(a, b int) (int, error) {
    if b == 0 {
        return 0, fmt.Errorf("divisor cannot be 0")
    }
    return a / b, nil
}
```

```go
// math_test.go
package math

import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    expected := 5
    if result != expected {
        t.Errorf("Add(2,3) = %d; want %d", result, expected)
    }
}

func TestDivide(t *testing.T) {
    result, err := Divide(10, 2)
    if err != nil {
        t.Fatal("should not error:", err)
    }
    if result != 5 {
        t.Errorf("Divide(10,2) = %d; want 5", result)
    }
}

func TestDivideByZero(t *testing.T) {
    _, err := Divide(10, 0)
    if err == nil {
        t.Error("divide by zero should error")
    }
}
```

```bash
# Run tests
go test                     # Current package
go test ./...               # All packages
go test -v                  # Verbose output
go test -run TestAdd        # Run specific test
go test -count=1            # Disable cache
```

### Table-Driven Tests

```go
func TestAddTable(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive numbers", 2, 3, 5},
        {"negative numbers", -1, -2, -3},
        {"zero values", 0, 0, 0},
        {"positive and negative", 5, -3, 2},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Add(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("Add(%d,%d) = %d; want %d",
                    tt.a, tt.b, result, tt.expected)
            }
        })
    }
}
```

### Setup and Teardown

```go
func TestMain(m *testing.M) {
    fmt.Println("=== Before all tests ===")
    
    // Initialize database, create files, etc.
    setup()
    
    code := m.Run()  // Run all tests
    
    // Clean up resources
    teardown()
    
    fmt.Println("=== After all tests ===")
    os.Exit(code)
}
```

---

## 2. Subtests

```go
func TestUser(t *testing.T) {
    // Parallel subtests
    t.Run("create user", func(t *testing.T) {
        t.Parallel()
        // Test...
    })
    
    t.Run("get user", func(t *testing.T) {
        t.Parallel()
        // Test...
    })
}
```

---

## 3. Test Helper Functions

### Using Helpers

```go
func assertEqual(t *testing.T, got, want interface{}, msg ...string) {
    t.Helper()  // Mark as helper, reports caller's line number on failure
    if got != want {
        defaultMsg := fmt.Sprintf("got %v, want %v", got, want)
        if len(msg) > 0 {
            defaultMsg = msg[0] + ": " + defaultMsg
        }
        t.Error(defaultMsg)
    }
}

func TestAdd(t *testing.T) {
    assertEqual(t, Add(2, 3), 5)
    assertEqual(t, Add(-1, 1), 0, "positive and negative addition")
}
```

### Temporary Directories

```go
func TestFile(t *testing.T) {
    dir := t.TempDir()  // Auto cleanup
    path := filepath.Join(dir, "test.txt")
    
    os.WriteFile(path, []byte("hello"), 0644)
    data, _ := os.ReadFile(path)
    
    if string(data) != "hello" {
        t.Error("content mismatch")
    }
}
```

---

## 4. Mocking and Interfaces

```go
// Dependency interface definition
type EmailSender interface {
    Send(to, subject, body string) error
}

type UserService struct {
    sender EmailSender
}

func (s *UserService) Register(name, email string) error {
    // Registration logic...
    return s.sender.Send(email, "Welcome", "Welcome, "+name)
}

// Use mock in testing
type mockSender struct {
    calls []string
}

func (m *mockSender) Send(to, subject, body string) error {
    m.calls = append(m.calls, to)
    return nil
}

func TestRegister(t *testing.T) {
    mock := &mockSender{}
    svc := &UserService{sender: mock}
    
    svc.Register("Zhang San", "zs@test.com")
    
    if len(mock.calls) != 1 {
        t.Error("should have sent one email")
    }
}
```

---

## 5. Benchmarking

```go
func BenchmarkAdd(b *testing.B) {
    // b.N is determined by the framework
    for i := 0; i < b.N; i++ {
        Add(2, 3)
    }
}

func BenchmarkStringConcat(b *testing.B) {
    // Reset timer
    b.ResetTimer()
    
    for i := 0; i < b.N; i++ {
        var s string
        for j := 0; j < 100; j++ {
            s += "a"
        }
    }
}

func BenchmarkStringBuilder(b *testing.B) {
    b.ResetTimer()
    
    for i := 0; i < b.N; i++ {
        var sb strings.Builder
        for j := 0; j < 100; j++ {
            sb.WriteString("a")
        }
        _ = sb.String()
    }
}
```

```bash
# Run benchmarks
go test -bench=. -benchmem
# BenchmarkAdd-8               1000000000  0.25 ns/op   0 B/op  0 allocs/op
# BenchmarkStringConcat-8         100000  20000 ns/op  5000 B/op  100 allocs/op
# BenchmarkStringBuilder-8       1000000   1000 ns/op   500 B/op  1 allocs/op
```

---

## 6. Performance Profiling (pprof)

```go
import (
    "net/http"
    _ "net/http/pprof"  // Import pprof
)

func main() {
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
    
    // Application logic...
}
```

```bash
# CPU profiling
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Memory profiling
go tool pprof http://localhost:6060/debug/pprof/heap

# Goroutine profiling
go tool pprof http://localhost:6060/debug/pprof/goroutine

# Generate flame graph
go tool pprof -http=:8080 ~/pprof/pprof.samples.cpu.001.pb.gz
```

---

## 7. Code Coverage

```bash
# View test coverage
go test -cover
# ok      example/math   0.002s   coverage: 80.0% of statements

# Generate coverage report
go test -coverprofile=coverage.out
go tool cover -html=coverage.out  # View in browser
```

---

## Wrap-up

Testing is part of Go culture. Table-driven tests are the standard pattern; the testing package provides complete functionality for tests, benchmarks, subtests, etc. pprof is a powerful tool for performance analysis, and combined with flame graphs, it can quickly identify performance bottlenecks.

---

