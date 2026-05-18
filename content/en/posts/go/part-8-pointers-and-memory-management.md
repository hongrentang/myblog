---
title: "Go from Beginner to Pro · Part 8: Pointers and Memory Management"
date: 2025-01-25
weight: 1208
draft: false
tags: ["go"]
---

## 1. Pointers

Go has pointers, but no pointer arithmetic (unlike C where you can do `p++`).

### Basic Usage

```go
var x int = 10
var p *int = &x  // p points to the address of x

fmt.Println(p)   // 0xc0000a0000 (address)
fmt.Println(*p)  // 10 (dereference)
```

### new Function

```go
// new allocates memory and returns a pointer
p := new(int)    // *int, default value 0
*p = 100

// Equivalent to
var x int
p := &x
```

### Pointers as Parameters

```go
func swap(a, b *int) {
    *a, *b = *b, *a
}

x, y := 1, 2
swap(&x, &y)
fmt.Println(x, y)  // 2 1
```

### Pointer vs Value

```go
// Value passing — copy
func byValue(u User) {
    u.Name = "Cannot change"
}

// Pointer passing — modify original data
func byPointer(u *User) {
    u.Name = "Can change"
}

u := User{Name: "Zhang San"}
byValue(u)
fmt.Println(u.Name)    // Zhang San

byPointer(&u)
fmt.Println(u.Name)    // Can change
```

---

## 2. new vs make

| Comparison | new | make |
|-----------|-----|------|
| Purpose | Allocate memory | Initialize built-in reference types |
| Returns | Pointer (*T) | Value (slice/map/channel) |
| Applicable | Any type | Only slice, map, channel |
| Initialization | Zero value | Allocate underlying data structure |

```go
p := new([]int)      // *[]int (nil slice)
fmt.Println(*p == nil)  // true

v := make([]int, 5)  // []int value, underlying allocated
fmt.Println(v)       // [0 0 0 0 0]

// Common pattern: create and return pointer to struct
func NewUser(name string) *User {
    return &User{Name: name}  // Go compiler automatically allocates to heap
}
```

---

## 3. Stack vs Heap

### Escape Analysis

The Go compiler analyzes whether variables should be allocated on the stack or heap:

```go
// Stack allocation — variables don't escape
func sum() int {
    x := 10
    y := 20
    return x + y  // x, y don't escape, allocated on stack
}

// Heap allocation — variable escapes to heap
func newUser() *User {
    u := User{Name: "Zhang San"}  // Because a pointer is returned, u escapes to heap
    return &u
}
```

### Viewing Escape Analysis

```bash
go build -gcflags '-m' main.go

# Example output:
# ./main.go:10:2: moved to heap: u
```

### Why Care About Escape?

| Allocation | Advantages | Disadvantages |
|-----------|------------|---------------|
| Stack | Zero GC overhead, automatic reclamation | Short lifetime |
| Heap | Flexible lifetime | GC overhead |

### Reducing Unnecessary Heap Allocations

```go
// ❌ Returning a pointer causes escape
func getValue(m map[string]int, key string) *int {
    v := m[key]
    return &v  // v escapes to heap
}

// ✅ Return value type
func getValue(m map[string]int, key string) int {
    return m[key]  // Stack allocation
}
```

---

## 4. GC (Garbage Collection)

Go uses the **concurrent tri-color mark-sweep** algorithm.

### GC Trigger Conditions

```go
// 1. Heap memory grows to a certain proportion
// 2. No GC for over 2 minutes
// 3. Manual call
runtime.GC()
```

### GC Debugging

```go
// View GC statistics
var m runtime.MemStats
runtime.ReadMemStats(&m)

fmt.Printf("Allocated memory: %d MB\n", m.Alloc/1024/1024)
fmt.Printf("Total allocated: %d MB\n", m.TotalAlloc/1024/1024)
fmt.Printf("GC count: %d\n", m.NumGC)
fmt.Printf("Next GC trigger: %d MB\n", m.NextGC/1024/1024)

// GODEBUG environment variable
// GODEBUG=gctrace=1 ./myapp
// gc 1 @0.003s 4%: 0.022+0.18+0.009 ms clock, 0.17+0.23/0.37/0+0.073 ms cpu, 4->4->0 MB, 5 MB goal, 4 P
```

---

## 5. unsafe Package

**Avoid using it** in daily development; only use it in special scenarios:

```go
import "unsafe"

func main() {
    x := int32(10)
    size := unsafe.Sizeof(x)
    fmt.Println(size)  // 4

    // Pointer conversion (dangerous!)
    var f float64 = 3.14
    // Convert float64 pointer to int64 pointer (not recommended)
    i := *(*int64)(unsafe.Pointer(&f))
    fmt.Println(i)
}
```

---

## 6. Common Memory Issues

### 1. Memory Leak

```go
// Slice referencing large array, cannot be freed
func process() {
    data := readLargeFile()  // Large array
    result := data[:10]      // Small slice referencing large array
    // data cannot be GC'd because it's referenced by result
}

// Fix: use copy
func process() {
    data := readLargeFile()
    result := make([]byte, 10)
    copy(result, data[:10])  // result is independent, data can be GC'd
}
```

### 2. Goroutine Leak

```go
// ❌ Goroutine blocks forever
func leak() {
    ch := make(chan int)
    go func() {
        <-ch  // Waits forever
    }()
}

// ✅ Use context timeout control
func noLeak(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case <-ch:
        case <-ctx.Done():
        }
    }()
}
```

---

## 7. Memory Optimization Tips

```go
// 1. Pre-allocate slice capacity
// ❌
var s []int
for i := 0; i < 10000; i++ {
    s = append(s, i)
}

// ✅
s := make([]int, 0, 10000)
for i := 0; i < 10000; i++ {
    s = append(s, i)
}

// 2. Reuse objects (sync.Pool)
var pool = sync.Pool{
    New: func() interface{} {
        return &bytes.Buffer{}
    },
}

func write(w io.Writer) {
    buf := pool.Get().(*bytes.Buffer)
    buf.Reset()
    defer pool.Put(buf)

    // Use buf...
    w.Write(buf.Bytes())
}

// 3. Avoid large stack copies
// Pass large structs with pointers

// 4. Reduce string concatenation
// ❌
s := ""
for i := 0; i < 1000; i++ {
    s += "a"
}
// O(n²), creates a new string each time

// ✅
var b strings.Builder
for i := 0; i < 1000; i++ {
    b.WriteString("a")
}
s := b.String()  // O(n), pre-allocates buffer
```

---

## Wrap-up

Go's pointers are simple and safe (no pointer arithmetic), combined with escape analysis and GC for automatic memory management. Daily development doesn't require excessive focus on memory allocation, but in performance-sensitive scenarios, understanding escape analysis and reducing heap allocations can be very helpful.

---

