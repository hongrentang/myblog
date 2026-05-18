---
title: "Go from Beginner to Pro · Part 11: Introduction to Concurrency"
date: 2025-01-03
weight: 11
draft: false
tags: ["go"]
---

## Go's Concurrency Model

Go's concurrency is based on the **CSP (Communicating Sequential Processes)** model:

> **Don't communicate by sharing memory; share memory by communicating.**
> — Go Concurrency Philosophy

```
Traditional languages: thread ↔ shared memory ↔ locks
Go:                  goroutine ↔ channel ↔ message passing
```

---

## 1. Goroutine

A goroutine is Go's lightweight thread, managed by the Go runtime.

### Starting a Goroutine

```go
func say(msg string) {
    for i := 0; i < 5; i++ {
        time.Sleep(100 * time.Millisecond)
        fmt.Println(msg)
    }
}

func main() {
    go say("world")  // Runs in a new goroutine
    say("hello")     // Runs in the current goroutine
}
```

### Goroutine Characteristics

```go
// 1. Lightweight — stack starts at just a few KB, can grow dynamically
// 2. Scheduling managed by Go Runtime, no expensive OS kernel context switching
// 3. A single Go process can easily create hundreds of thousands of goroutines

// Starting a goroutine: just add go before the function call
go func() {
    fmt.Println("running in goroutine")
}()
```

### Waiting for Goroutines

```go
// main function doesn't wait for goroutines
func main() {
    go fmt.Println("won't be printed")
}  // main exits directly, goroutine never gets to execute

// Solution: use WaitGroup
func main() {
    var wg sync.WaitGroup
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println("will be printed")
    }()
    wg.Wait()  // Wait for all Done
}
```

---

## 2. Channel

Channel is the communication pipe between goroutines.

### Creation and Basic Operations

```go
// Create
ch := make(chan int)       // Unbuffered channel
ch := make(chan int, 10)   // Buffered channel (buffer of 10)

// Send
ch <- 42

// Receive
x := <-ch
x, ok := <-ch  // ok is false if channel is closed

// Close
close(ch)
```

### Unbuffered Channel (Synchronous)

```go
// Sending and receiving on an unbuffered channel must happen simultaneously in different goroutines
func main() {
    ch := make(chan string)

    go func() {
        fmt.Println("goroutine ready to send...")
        ch <- "hello"  // Blocks until main receives
        fmt.Println("goroutine sent successfully")
    }()

    time.Sleep(time.Second)  // Simulate delay
    fmt.Println("main ready to receive...")
    msg := <-ch  // Blocks until goroutine sends
    fmt.Println("received:", msg)

    time.Sleep(time.Second)
}
```

### Buffered Channel (Asynchronous)

```go
func main() {
    ch := make(chan int, 3)

    ch <- 1  // Doesn't block
    ch <- 2  // Doesn't block
    ch <- 3  // Doesn't block
    // ch <- 4  // Blocks! Buffer is full

    fmt.Println(<-ch)  // 1
    fmt.Println(<-ch)  // 2
    fmt.Println(<-ch)  // 3
}
```

### Iterating Over a Channel

```go
func main() {
    ch := make(chan int)

    go func() {
        for i := 0; i < 5; i++ {
            ch <- i
        }
        close(ch)  // Must close, otherwise range will deadlock
    }()

    for v := range ch {
        fmt.Println(v)
    }
}
```

### Channel Direction

```go
// Read-only channel
func reader(ch <-chan int) {
    for v := range ch {
        fmt.Println(v)
    }
}

// Write-only channel
func writer(ch chan<- int) {
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch)
}
```

---

## 3. Simple Examples

### Producer-Consumer

```go
func producer(ch chan<- int) {
    for i := 0; i < 10; i++ {
        fmt.Printf("producing: %d\n", i)
        ch <- i
        time.Sleep(100 * time.Millisecond)
    }
    close(ch)
}

func consumer(ch <-chan int, name string) {
    for v := range ch {
        fmt.Printf("[%s] consuming: %d\n", name, v)
        time.Sleep(150 * time.Millisecond)
    }
}

func main() {
    ch := make(chan int, 5)
    
    go producer(ch)
    go consumer(ch, "C1")
    go consumer(ch, "C2")
    
    time.Sleep(3 * time.Second)
}
```

### Worker Pool

```go
func worker(id int, jobs <-chan int, results chan<- int) {
    for job := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, job)
        time.Sleep(time.Second)
        results <- job * 2
    }
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)

    // Start 3 workers
    for w := 1; w <= 3; w++ {
        go worker(w, jobs, results)
    }

    // Send 9 jobs
    for j := 1; j <= 9; j++ {
        jobs <- j
    }
    close(jobs)

    // Collect results
    for r := 1; r <= 9; r++ {
        <-results
    }
}
```

---

## 4. Concurrency vs Parallelism

```go
// Concurrency: structure of handling multiple tasks
// Parallelism: executing multiple tasks simultaneously

// Go uses all CPU cores by default
fmt.Println(runtime.NumCPU())  // View number of CPU cores

// Set number of cores to use
runtime.GOMAXPROCS(4)

// GOMAXPROCS controls the number of OS threads executing user-mode Go code
// Default is the number of CPU cores

// View current GOMAXPROCS
fmt.Println(runtime.GOMAXPROCS(0))

// View number of currently running goroutines
fmt.Println(runtime.NumGoroutine())
```

---

## 5. Common Pitfalls

### Pitfall 1: Goroutine Leak

```go
// ❌ Never ends
func leak() {
    ch := make(chan int)
    go func() {
        <-ch  // Waits forever
    }()
}

// ✅ With timeout
func noLeak() {
    ch := make(chan int)
    go func() {
        select {
        case <-ch:
        case <-time.After(5 * time.Second):
        }
    }()
}
```

### Pitfall 2: Sending to a Closed Channel

```go
ch := make(chan int)
close(ch)
ch <- 1  // panic: send on closed channel

// Only the sender should close
// No sends allowed after closing
// Receives are still allowed after closing (until buffer is empty)
```

### Pitfall 3: range on an Unclosed Channel

```go
ch := make(chan int)
go func() {
    for i := 0; i < 3; i++ {
        ch <- i
    }
    // Didn't close
}()

for v := range ch {
    fmt.Println(v)  // Prints 0,1,2 then deadlock
}
```

---

## Wrap-up

Goroutines and channels are the core of Go concurrency. Goroutines are lightweight enough to create hundreds of thousands; channels are used for communication and synchronization between goroutines. Understanding the synchronous nature of unbuffered channels and the asynchronous nature of buffered channels is key to mastering Go concurrency.

---

