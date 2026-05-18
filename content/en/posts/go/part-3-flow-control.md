---
title: "Go from Beginner to Pro · Part 3: Flow Control"
date: 2025-01-20
weight: 1203
draft: false
tags: ["go"]
---

Go has only 25 keywords; the ones related to flow control are `if`, `for`, `switch`, `defer`, `select` (covered in the concurrency article), and `goto`. There is no `while` or `do-while`.

---

## 1. if

### Basic Syntax

```go
// Go conditions don't need parentheses
age := 18
if age >= 18 {
    fmt.Println("adult")
}
```

### if-else

```go
if age >= 18 {
    fmt.Println("adult")
} else if age >= 12 {
    fmt.Println("teenager")
} else {
    fmt.Println("child")
}
```

**Note**: `else` must be on the same line as `}`.

### if with Initialization

A Go feature — you can execute a short statement in the if condition:

```go
if score := 85; score >= 60 {
    fmt.Println("pass")
} else {
    fmt.Println("fail")
}
// score cannot be accessed here
```

### Using if for Error Checking

```go
// Go standard error handling pattern
if err := doSomething(); err != nil {
    fmt.Println("error:", err)
    return
}

// Error checking with multiple return values
if data, err := fetchData(); err != nil {
    log.Fatal(err)
} else {
    fmt.Println(data)
}
```

---

## 2. for

Go only has `for` loops, no `while` or `do-while`.

### Full Form

```go
for i := 0; i < 10; i++ {
    fmt.Println(i)
}
```

### "while" Form

```go
sum := 0
for sum < 100 {
    sum += sum + 1
}
```

### "while(true)" Form (Infinite Loop)

```go
for {
    // Infinite loop
    break  // Exit
}
```

### range Iteration

```go
// Iterate over a slice
nums := []int{10, 20, 30}
for index, value := range nums {
    fmt.Println(index, value)
}

// Iterate over a map
m := map[string]int{"a": 1, "b": 2}
for key, value := range m {
    fmt.Println(key, value)
}

// Iterate over a string (by rune)
for i, r := range "hello世界" {
    fmt.Printf("%d %c\n", i, r)
}

// Only get index/key, ignore value
for i := range nums {
    fmt.Println(nums[i])
}

// Only get value, ignore index/key
for _, v := range nums {
    fmt.Println(v)
}
```

### break and continue

```go
// break: exit current loop
for i := 0; i < 10; i++ {
    if i == 5 {
        break  // Exit for loop
    }
}

// continue: skip current iteration
for i := 0; i < 10; i++ {
    if i%2 == 0 {
        continue  // Skip even numbers
    }
    fmt.Println(i)
}

// break to label (exit outer loop)
outer:
for i := 0; i < 3; i++ {
    for j := 0; j < 3; j++ {
        if i*j > 2 {
            break outer  // Exit both loops
        }
        fmt.Println(i, j)
    }
}
```

---

## 3. switch

### Basic Syntax

```go
score := 85

switch {
case score >= 90:
    fmt.Println("excellent")
case score >= 80:
    fmt.Println("good")  // Matches here
case score >= 60:
    fmt.Println("pass")
default:
    fmt.Println("fail")
}
```

### Expression switch

```go
switch n := 3; n {
case 1:
    fmt.Println("one")
case 2, 3, 4:  // Match multiple values
    fmt.Println("two three four")
case 5:
    fmt.Println("five")
default:
    fmt.Println("other")
}
```

### No break Needed

Go's switch **does not fall through by default**, it automatically exits after matching a case:

```go
switch n {
case 1:
    fmt.Println("1")
    // Automatically breaks, won't enter case 2
case 2:
    fmt.Println("2")
}

// To fall through, explicitly use fallthrough
switch n := 2; n {
case 1:
    fmt.Println("1")
    fallthrough  // Will continue to execute case 2
case 2:
    fmt.Println("2")  // case 1 fallthrough will also execute this
    fallthrough
case 3:
    fmt.Println("3")
}
// Output: 2 \n 3
```

### type switch

```go
var x interface{} = 42

switch v := x.(type) {
case int:
    fmt.Printf("integer: %d\n", v)
case string:
    fmt.Printf("string: %s\n", v)
case float64:
    fmt.Printf("float: %f\n", v)
default:
    fmt.Printf("unknown type: %T\n", v)
}
```

---

## 4. defer

`defer` delays execution until the function returns. Used for resource release, closing files, unlocking, etc.

### Basic Usage

```go
func readFile() error {
    f, err := os.Open("file.txt")
    if err != nil {
        return err
    }
    defer f.Close()  // Close file before function returns

    // Read file operations
    // ...
    return nil
}
```

### defer Execution Order (LIFO)

```go
func example() {
    defer fmt.Println("1")  // Executed third
    defer fmt.Println("2")  // Executed second
    defer fmt.Println("3")  // Executed first (last in, first out)
}
// Output: 3 2 1
```

### When defer Arguments Are Evaluated

```go
func example() {
    x := 10
    defer fmt.Println(x)  // Output 10 (argument evaluated at defer declaration)
    x = 20
}
// Output: 10
```

### defer Modifying Return Values

```go
// When using named return values, defer can modify the return value
func example() (result int) {
    defer func() {
        result += 10  // Modify return value
    }()
    return 5  // Actually returns 15
}
```

### Common defer Uses

```go
// Close file
f, _ := os.Open("file.txt")
defer f.Close()

// Unlock
mu.Lock()
defer mu.Unlock()

// Record function execution time
func timed() {
    defer func(start time.Time) {
        fmt.Printf("duration: %v\n", time.Since(start))
    }(time.Now())
    // Business logic...
}

// Close HTTP response body (common mistake)
resp, _ := http.Get(url)
defer resp.Body.Close()
```

---

## 5. goto

Use sparingly, but occasionally useful:

```go
func example() {
    i := 0

loop:  // Label
    fmt.Println(i)
    i++
    if i < 3 {
        goto loop  // Jump to label
    }
}
```

---

## 6. Practical Patterns

### Functional Options Pattern (Fowler)

```go
type Server struct {
    Host    string
    Port    int
    Timeout time.Duration
}

type Option func(*Server)

func WithHost(host string) Option {
    return func(s *Server) {
        s.Host = host
    }
}

func WithPort(port int) Option {
    return func(s *Server) {
        s.Port = port
    }
}

func NewServer(opts ...Option) *Server {
    s := &Server{
        Host:    "localhost",
        Port:    8080,
        Timeout: 30 * time.Second,
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Usage
func main() {
    s := NewServer(WithPort(9090))
    fmt.Printf("%+v\n", s)
}
```

---

## Wrap-up

This article covered if, for, switch, defer, and goto. Go's flow control has few keywords but is concise: for unifies all loops, switch doesn't fall through by default, and defer is a powerful resource management tool.

---

