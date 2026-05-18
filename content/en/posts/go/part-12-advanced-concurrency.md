---
title: "Go from Beginner to Pro · Part 12: Advanced Concurrency"
date: 2025-01-04
weight: 12
draft: false
tags: ["go"]
---

## 1. select

select allows a single goroutine to wait on multiple channel operations simultaneously.

### Basic Usage

```go
func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        time.Sleep(1 * time.Second)
        ch1 <- "from ch1"
    }()

    go func() {
        time.Sleep(2 * time.Second)
        ch2 <- "from ch2"
    }()

    select {
    case msg := <-ch1:
        fmt.Println(msg)
    case msg := <-ch2:
        fmt.Println(msg)
    case <-time.After(3 * time.Second):
        fmt.Println("timeout")
    default:
        fmt.Println("no channel ready")
    }
}
```

### select Rules

```
1. Receiving from a closed channel returns the zero value immediately (not blocking)
2. Sending to/receiving from a nil channel blocks forever
3. When multiple cases are ready simultaneously, one is chosen at random
4. If all channels are blocked and there's a default, default is executed
5. If all channels are blocked and there's no default, it blocks until one is ready
```

### Timeout Control

```go
// Method 1: time.After
select {
case result := <-ch:
    fmt.Println(result)
case <-time.After(2 * time.Second):
    fmt.Println("operation timed out")
}

// Method 2: context timeout (recommended, more flexible)
func doWithTimeout(ctx context.Context, ch chan int) error {
    select {
    case result := <-ch:
        fmt.Println(result)
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

### Non-blocking Communication

```go
// Non-blocking send
select {
case ch <- value:
    // Send succeeded
default:
    // Send failed (buffer full), doesn't block
}

// Non-blocking receive
select {
case v := <-ch:
    fmt.Println("received:", v)
default:
    fmt.Println("no data")
}
```

### Fan-Out Pattern

```go
// Same data sent to multiple channels
func fanOut(ch <-chan int, outs []chan<- int) {
    for v := range ch {
        for _, out := range outs {
            out <- v
        }
    }
}
```

---

## 2. sync Package

### sync.WaitGroup

```go
func main() {
    var wg sync.WaitGroup

    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            fmt.Printf("Worker %d starting\n", id)
            time.Sleep(time.Second)
            fmt.Printf("Worker %d done\n", id)
        }(i)
    }

    wg.Wait()  // Wait for all workers
    fmt.Println("all done")
}
```

### sync.Mutex

```go
type Counter struct {
    mu    sync.Mutex
    value int
}

func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

func (c *Counter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}
```

### sync.RWMutex

```go
type Cache struct {
    mu    sync.RWMutex
    data  map[string]string
}

func (c *Cache) Get(key string) string {
    c.mu.RLock()        // Read lock, allows concurrent reads
    defer c.mu.RUnlock()
    return c.data[key]
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock()         // Write lock, exclusive
    defer c.mu.Unlock()
    c.data[key] = value
}
```

### sync.Once

```go
var once sync.Once
var config *Config

func GetConfig() *Config {
    once.Do(func() {
        fmt.Println("initializing config (executed once)")
        config = loadConfig()
    })
    return config
}
```

### sync.Pool

```go
var pool = sync.Pool{
    New: func() interface{} {
        return &bytes.Buffer{}
    },
}

func process() {
    buf := pool.Get().(*bytes.Buffer)
    buf.Reset()
    defer pool.Put(buf)

    // Use buf
}
// Return after use, reuse next time, reduce memory allocations
```

### sync.Map

```go
// Concurrent-safe map (suitable for read-heavy, key-non-conflicting scenarios)
var m sync.Map

// Write
m.Store("key", "value")

// Read
v, ok := m.Load("key")

// Delete
m.Delete("key")

// Iterate
m.Range(func(key, value interface{}) bool {
    fmt.Println(key, value)
    return true  // Continue iteration
})
```

---

## 3. Context Package

context is used for passing cancellation signals and request-scoped values.

### Creating Context

```go
// Root context
ctx := context.Background()
ctx := context.TODO()

// With cancellation
ctx, cancel := context.WithCancel(context.Background())
defer cancel()  // Usually deferred

// With timeout
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

// With deadline
ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(2*time.Second))
defer cancel()

// With value
ctx = context.WithValue(ctx, "user_id", 123)
```

### Practical: Timeout Cancellation

```go
func fetchData(ctx context.Context, url string) ([]byte, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    return io.ReadAll(resp.Body)
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    data, err := fetchData(ctx, "https://api.example.com/data")
    if err != nil {
        fmt.Println("request failed:", err)
        return
    }
    fmt.Println(string(data))
}
```

### Cascading Cancellation

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())

    go func() {
        time.Sleep(2 * time.Second)
        cancel()  // Cancel all subtasks
    }()

    var wg sync.WaitGroup
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            select {
            case <-time.After(5 * time.Second):
                fmt.Printf("Task %d completed\n", id)
            case <-ctx.Done():
                fmt.Printf("Task %d cancelled\n", id)
            }
        }(i)
    }

    wg.Wait()
    fmt.Println("main program ended")
}
```

---

## 4. Concurrency Patterns

### 1. Pipeline

```go
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

func main() {
    // Data flow: 1,2,3,4 → square → output
    nums := gen(1, 2, 3, 4)
    squares := square(nums)

    for v := range squares {
        fmt.Println(v)  // 1, 4, 9, 16
    }
}
```

### 2. Fan-Out / Fan-In

```go
// Fan-Out: distribute data to multiple workers
func fanOut(in <-chan int, workers int) []<-chan int {
    channels := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        channels[i] = worker(in)
    }
    return channels
}

// Fan-In: merge multiple channels
func fanIn(chs ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup

    for _, ch := range chs {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                out <- v
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

### 3. Timeout + Retry

```go
func fetchWithRetry(ctx context.Context, url string, retries int) ([]byte, error) {
    for i := 0; i < retries; i++ {
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        default:
        }

        data, err := fetchData(ctx, url)
        if err == nil {
            return data, nil
        }

        // Exponential backoff
        time.Sleep(time.Duration(1<<uint(i)) * 100 * time.Millisecond)
    }
    return nil, fmt.Errorf("failed after %d retries", retries)
}
```

### 4. Concurrency Control (Semaphore)

```go
func main() {
    sem := make(chan struct{}, 3)  // Max 3 concurrent
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            sem <- struct{}{}      // Acquire token
            defer func() { <-sem }() // Release token

            fmt.Printf("Task %d starting\n", id)
            time.Sleep(time.Second)
            fmt.Printf("Task %d done\n", id)
        }(i)
    }

    wg.Wait()
}
```

---

## 5. Race Detection

```go
// Add -race flag at compile/runtime to detect data races
// go run -race main.go
// go build -race ./...

// hello.go
func main() {
    var count int
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            count++  // Data race!
        }()
    }

    wg.Wait()
    fmt.Println(count)
}
```

```bash
go run -race main.go
# Output will show WARNING: DATA RACE
```

**Fixing races**: Use Mutex, Atomic, or channel synchronization.

---

## Wrap-up

This article covered select, the sync package, Context, concurrency patterns, and race detection. With these tools, you can write safe and efficient concurrent programs. Goroutines are tools, channels are the means of communication, but tools need to be combined into patterns (Pipeline, Fan-Out/In) to solve real-world problems.

---

