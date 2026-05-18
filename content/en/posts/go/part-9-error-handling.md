---
title: "Go from Beginner to Pro · Part 9: Error Handling"
date: 2025-01-26
weight: 1209
draft: false
tags: ["go"]
---

## Error Handling Philosophy

Go has no exceptions (try-catch); errors are just ordinary values.

**Design Principles**:

```
Errors are values, not exceptions
Explicit error handling, not implicit throwing
Every place that can go wrong should be handled
```

---

## 1. error Interface

error is a built-in interface in Go:

```go
type error interface {
    Error() string
}
```

Any type that implements the `Error() string` method is an error.

### Basic Usage

```go
// Function returning error
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("divisor cannot be 0")
    }
    return a / b, nil
}

// Handling on call
result, err := divide(10, 0)
if err != nil {
    fmt.Println("error:", err)
    return
}
fmt.Println(result)
```

---

## 2. Creating Errors

### errors.New

```go
import "errors"

var ErrNotFound = errors.New("record not found")
var ErrPermission = errors.New("permission denied")

func findUser(id int) (*User, error) {
    if id <= 0 {
        return nil, ErrNotFound
    }
    // ...
}
```

### fmt.Errorf

```go
// Formatted error
func validateAge(age int) error {
    if age < 0 || age > 150 {
        return fmt.Errorf("invalid age: %d", age)
    }
    return nil
}

// %w wraps the error (Go 1.13+)
func process() error {
    err := doSomething()
    if err != nil {
        return fmt.Errorf("processing failed: %w", err)  // %w wraps the original error
    }
    return nil
}
```

---

## 3. Custom Error Types

### Errors with Additional Information

```go
type ValidationError struct {
    Field   string
    Value   interface{}
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("field %s validation failed: %s (value: %v)",
        e.Field, e.Message, e.Value)
}

func validateUser(u User) error {
    if u.Name == "" {
        return &ValidationError{
            Field:   "Name",
            Value:   u.Name,
            Message: "name cannot be empty",
        }
    }
    return nil
}

// Use type assertion to get detailed information
err := validateUser(user)
var ve *ValidationError
if errors.As(err, &ve) {
    fmt.Printf("validation failed: %s (%s)\n", ve.Field, ve.Message)
}
```

### Errors with Error Codes

```go
type AppError struct {
    Code    int
    Message string
    Err     error
}

func (e *AppError) Error() string {
    return fmt.Sprintf("[%d] %s: %v", e.Code, e.Message, e.Err)
}

func (e *AppError) Unwrap() error {
    return e.Err
}

func NewAppError(code int, msg string, err error) *AppError {
    return &AppError{Code: code, Message: msg, Err: err}
}
```

---

## 4. Error Handling Patterns

### Basic Pattern

```go
// Standard pattern: if err != nil
data, err := fetchData()
if err != nil {
    log.Printf("fetchData error: %v", err)
    return nil, err
}
```

### Wrapping Pattern

```go
// Wrap layer by layer, preserving stack information
func LoadConfig(path string) (*Config, error) {
    file, err := os.Open(path)
    if err != nil {
        return nil, fmt.Errorf("failed to open config file %s: %w", path, err)
    }
    defer file.Close()

    var cfg Config
    if err := json.NewDecoder(file).Decode(&cfg); err != nil {
        return nil, fmt.Errorf("failed to parse config: %w", err)
    }
    return &cfg, nil
}
```

### Fail Fast

```go
// If startup configuration is wrong, panic directly
func init() {
    cfg, err := LoadConfig("config.json")
    if err != nil {
        panic(fmt.Sprintf("failed to load config: %v", err))
    }
}
```

### Retry Pattern

```go
func fetchWithRetry(url string, maxRetries int) (*Response, error) {
    var lastErr error
    for i := 0; i < maxRetries; i++ {
        resp, err := http.Get(url)
        if err == nil {
            return resp, nil
        }
        lastErr = err
        time.Sleep(time.Second * time.Duration(i+1))
    }
    return nil, fmt.Errorf("failed after %d retries: %w", maxRetries, lastErr)
}
```

---

## 5. errors Package

Go 1.13+ errors package provides three key functions:

### errors.Is

```go
if errors.Is(err, ErrNotFound) {
    // Check if ErrNotFound is in the error chain
}
```

### errors.As

```go
var ve *ValidationError
if errors.As(err, &ve) {
    // Find the first error matching the type in the chain
    fmt.Println(ve.Field)
}
```

### errors.Unwrap

```go
// Unwrap one level
innerErr := errors.Unwrap(err)
```

### Complete Example

```go
// Wrapping at each layer
func readConfig() error {
    return fmt.Errorf("read config: %w", ErrNotFound)
}

func process() error {
    return fmt.Errorf("processing failed: %w", readConfig())
}

func main() {
    err := process()
    fmt.Println(err)                    // processing failed: read config: record not found

    fmt.Println(errors.Is(err, ErrNotFound))  // true (traverses error chain)
    fmt.Println(errors.Is(err, os.ErrNotExist)) // false
}
```

---

## 6. panic and recover

### panic

Panic is Go's runtime exception mechanism, typically used for **unrecoverable errors**:

```go
// When to use panic:
// 1. The program state is severely abnormal, continuing makes no sense
// 2. Programmer errors (e.g., index out of bounds)
// 3. Missing dependencies at startup

func must(err error) {
    if err != nil {
        panic(err)
    }
}

func init() {
    cfg, err := LoadConfig()
    must(err)  // Startup fails, panic directly
}
```

### recover

```go
// recover can only be used in defer
func safeRun(f func()) {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("program crashed, recovered:", r)
        }
    }()
    f()
}

// Practical: recover in HTTP handler
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("panic recovered: %v\n", err)
                http.Error(w, "Internal Server Error", 500)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

### recover in Chain Calls

```go
type Chain struct {
    steps []func() error
}

func (c *Chain) Add(step func() error) {
    c.steps = append(c.steps, step)
}

func (c *Chain) Run() (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("pipeline panic: %v", r)
        }
    }()
    for _, step := range c.steps {
        if err := step(); err != nil {
            return fmt.Errorf("step failed: %w", err)
        }
    }
    return nil
}
```

---

## 7. Error Handling Best Practices

```
Error handling principles:
1. Errors are values — handle them like ordinary values
2. Always handle errors — don't use _
3. Wrap errors to preserve context — use %w
4. Distinguish between recoverable and unrecoverable errors
5. Create sentinel errors with Err prefix
6. Don't log errors repeatedly — each layer either logs or wraps, not both
```

### Good vs Bad

```go
// ❌ Ignore errors
data, _ := doSomething()

// ❌ Multiple logging
func read() error {
    err := os.ReadFile("file")
    log.Println(err)   // Lower layer logs
    return err
}
func main() {
    err := read()
    log.Println(err)   // Upper layer logs again
}

// ✅ Correct approach: log only once
func read() error {
    return fmt.Errorf("read file: %w", os.ReadFile("file"))
}
func main() {
    err := read()
    if err != nil {
        log.Println(err)  // Log only at the outermost layer
    }
}
```

---

## Wrap-up

Go's error handling is simple and straightforward — errors are values. Core habits: **if err != nil**, **wrap errors with context**, **log only at the outermost layer**. Panic is only for truly exceptional situations, with recover as a safety net.

---

