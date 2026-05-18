---
title: "Go from Beginner to Pro · Part 4: Functions and Closures"
date: 2025-01-21
weight: 1204
draft: false
tags: ["go"]
---

## 1. Function Definition

```go
func functionName(parameterList) returnType {
    functionBody
}
```

### Simplest Function

```go
func sayHello() {
    fmt.Println("Hello!")
}
```

### Functions with Parameters

```go
func greet(name string) {
    fmt.Printf("Hello, %s!\n", name)
}

// Consecutive parameters of the same type can omit the type
func add(a, b int) int {
    return a + b
}
```

### Multiple Return Values

This is a Go feature, replacing exceptions/tuples in other languages:

```go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("divisor cannot be 0")
    }
    return a / b, nil
}

// Usage
result, err := divide(10, 2)
if err != nil {
    fmt.Println("error:", err)
    return
}
fmt.Println(result)
```

### Named Return Values

```go
// Return values can be named, assigned directly in the function body
func divide(a, b float64) (result float64, err error) {
    if b == 0 {
        err = fmt.Errorf("divisor cannot be 0")
        return  // Equivalent to return result, err
    }
    result = a / b
    return  // Naked return
}
```

**Naked return** is readable in short functions, but explicit returns are recommended in longer functions.

---

## 2. Functions Are First-Class Citizens

In Go, functions are value types and can be assigned to variables, passed as parameters, and returned as values.

### Function Types

```go
// Define function type
type MathFunc func(int, int) int

// Usage
var add MathFunc = func(a, b int) int {
    return a + b
}
fmt.Println(add(1, 2))  // 3
```

### Functions as Parameters

```go
func apply(nums []int, f func(int) int) []int {
    result := make([]int, len(nums))
    for i, v := range nums {
        result[i] = f(v)
    }
    return result
}

// Usage
nums := []int{1, 2, 3, 4}
double := func(x int) int { return x * 2 }
fmt.Println(apply(nums, double))  // [2 4 6 8]
```

### Functions as Return Values

```go
func multiplier(factor int) func(int) int {
    return func(x int) int {
        return x * factor
    }
}

// Usage
double := multiplier(2)
triple := multiplier(3)
fmt.Println(double(5))   // 10
fmt.Println(triple(5))   // 15
```

---

## 3. Closures

A closure is a function that "remembers" the external variables from its surrounding scope:

```go
func counter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}

c1 := counter()
fmt.Println(c1())  // 1
fmt.Println(c1())  // 2
fmt.Println(c1())  // 3

c2 := counter()    // New counter
fmt.Println(c2())  // 1
```

### Closure Pitfall (Common Bug)

```go
// ❌ Common mistake: capturing external variable in a loop
funcs := []func(){}
for i := 0; i < 3; i++ {
    funcs = append(funcs, func() {
        fmt.Println(i)  // Captures the same i
    })
}
for _, f := range funcs {
    f()  // All output 3, because after the loop ends i = 3
}

// ✅ Fix: pass value through parameter
for i := 0; i < 3; i++ {
    i := i  // Declare a new i, each iteration has its own copy
    funcs = append(funcs, func() {
        fmt.Println(i)
    })
}

// Or receive parameter via closure
for i := 0; i < 3; i++ {
    funcs = append(funcs, func(n int) func() {
        return func() { fmt.Println(n) }
    }(i))
}
```

Go 1.22+ has fixed the closure issue in `for range` loops, but the traditional `for i := 0; ...` still requires attention.

---

## 4. Variadic Parameters

```go
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

fmt.Println(sum(1, 2))       // 3
fmt.Println(sum(1, 2, 3, 4)) // 10

// Expand a slice
nums := []int{1, 2, 3}
fmt.Println(sum(nums...))    // 6
```

---

## 5. Anonymous Functions

Defined directly in code, no naming needed:

```go
// Define and immediately execute
func() {
    fmt.Println("immediately executed")
}()

// Assign to a variable
double := func(x int) int {
    return x * 2
}
fmt.Println(double(5))
```

---

## 6. init Function

Each package can have multiple `init` functions, which execute automatically when the package is imported:

```go
// Execution order: global variable initialization → init → main
var version = "1.0.0"

func init() {
    fmt.Println("init 1")
}

func init() {
    fmt.Println("init 2")
}

func main() {
    fmt.Println("main")
}
// init executes before main
```

---

## 7. Recursion

```go
func factorial(n int) int {
    if n <= 1 {
        return 1
    }
    return n * factorial(n-1)
}

// Fibonacci
func fibonacci(n int) int {
    if n <= 1 {
        return n
    }
    return fibonacci(n-1) + fibonacci(n-2)
}
```

---

## 8. Practical Patterns

### Functional Options Pattern

This is very common in project practice:

```go
type ServerConfig struct {
    Host    string
    Port    int
    Timeout time.Duration
}

type ServerOption func(*ServerConfig)

func WithHost(host string) ServerOption {
    return func(c *ServerConfig) {
        c.Host = host
    }
}

func WithPort(port int) ServerOption {
    return func(c *ServerConfig) {
        c.Port = port
    }
}

func NewServer(opts ...ServerOption) *ServerConfig {
    // Default config
    cfg := &ServerConfig{
        Host:    "localhost",
        Port:    8080,
        Timeout: 30 * time.Second,
    }
    // Apply options
    for _, opt := range opts {
        opt(cfg)
    }
    return cfg
}

func main() {
    srv := NewServer(WithPort(9090))
}
```

### Decorator Pattern

```go
func logging(next func(string) error) func(string) error {
    return func(s string) error {
        log.Printf("calling: %s", s)
        err := next(s)
        log.Printf("completed: err=%v", err)
        return err
    }
}

func handler(name string) error {
    fmt.Println("processing:", name)
    return nil
}

func main() {
    wrapped := logging(handler)
    wrapped("test")
}
```

---

## 9. Deferred Call Performance

```go
// defer has a small performance overhead, avoid excessive defer on hot paths

// ❌ Not recommended on hot paths
for i := 0; i < 1000000; i++ {
    mu.Lock()
    defer mu.Unlock()  // Unnecessary
}

// ✅ Manual unlock
for i := 0; i < 1000000; i++ {
    mu.Lock()
    // Operation
    mu.Unlock()
}
```

---

## Wrap-up

This article covered function declarations, multiple return values, closures, variadic parameters, recursion, and practical patterns. Functions are first-class citizens in Go — they can be assigned, passed, and returned, giving Go some of the capabilities of functional programming.

---

