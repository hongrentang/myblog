---
title: "Go from Beginner to Pro · Part 5: Arrays, Slices and Maps"
date: 2025-01-22
weight: 5
draft: false
tags: ["go"]
---

## 1. Arrays

Arrays are fixed-length contiguous memory blocks, and length is part of the type.

### Declaration

```go
// Declare and initialize
var nums [5]int               // [0 0 0 0 0]
var nums = [5]int{1, 2, 3, 4, 5}
nums := [5]int{1, 2, 3, 4, 5}

// Automatic length calculation
nums := [...]int{1, 2, 3, 4, 5}  // Length 5

// Initialize by index
nums := [5]int{0: 10, 3: 40}      // [10 0 0 40 0]
```

### Common Operations

```go
nums := [3]int{10, 20, 30}

// Access
fmt.Println(nums[0])   // 10
nums[1] = 25

// Length
fmt.Println(len(nums)) // 3

// Iteration
for i, v := range nums {
    fmt.Println(i, v)
}

// Comparison (arrays can be compared directly)
a := [3]int{1, 2, 3}
b := [3]int{1, 2, 3}
fmt.Println(a == b)  // true
```

### Note

```go
// Arrays are value types! Assignment and passing copy the entire array
func modify(arr [3]int) {
    arr[0] = 100
}

func main() {
    nums := [3]int{1, 2, 3}
    modify(nums)
    fmt.Println(nums[0])  // 1 (not modified!)
}
```

---

## 2. Slice

Slices are Go's most commonly used data structure — a dynamic view of an array.

### Creating

```go
// Literal
s := []int{1, 2, 3}

// make
s := make([]int, 3)      // [0 0 0], length=capacity=3
s := make([]int, 3, 5)   // length 3, capacity 5

// From array slicing
arr := [5]int{1, 2, 3, 4, 5}
s1 := arr[1:4]            // [2 3 4]
s2 := arr[:3]             // [1 2 3]
s3 := arr[2:]             // [3 4 5]
s4 := arr[:]              // [1 2 3 4 5]

// nil slice (zero value)
var s []int               // nil, len=0, cap=0
```

### Slice Internal Structure

```
Slice = pointer + length + capacity
┌──────────┬──────────┬──────────┐
│  ptr     │  len     │  cap     │
│ (data)   │ (3)      │ (5)      │
└────┬─────┴──────────┴──────────┘
     ↓
  ┌─┬─┬─┬─┬─┐
  │1│2│3│4│5│  ← Underlying array
  └─┴─┴─┴─┴─┘
```

### Common Operations

```go
s := []int{1, 2, 3, 4, 5}

// append
s = append(s, 6)           // [1 2 3 4 5 6]
s = append(s, 7, 8, 9)     // [1 2 3 4 5 6 7 8 9]
s = append(s, []int{10, 11}...)  // Expand and append

// copy
dst := make([]int, 3)
copied := copy(dst, s)     // Returns the number of elements actually copied

// Length and capacity
len(s)  // Number of elements
cap(s)  // Underlying array length

// Sub-slice
sub := s[1:4]               // New slice, shares underlying array
```

### append Growth Mechanism

```go
s := make([]int, 0, 2)
fmt.Println(cap(s))  // 2

s = append(s, 1)
s = append(s, 2)
fmt.Println(cap(s))  // 2 (not full yet)

s = append(s, 3)     // Growth!
fmt.Println(cap(s))  // 4 (capacity doubles)

// Growth strategy:
// Capacity < 1024: double
// Capacity >= 1024: increase by ~25%
```

### Slice Pitfalls

```go
// Pitfall 1: Slices share the underlying array
s1 := []int{1, 2, 3, 4, 5}
s2 := s1[1:3]         // [2 3]
s2[0] = 200           // Modifying s2 also affects s1!
fmt.Println(s1[1])    // 200

// Solution: use copy
s3 := make([]int, 2)
copy(s3, s1[1:3])     // Deep copy

// Pitfall 2: Unexpected side effects after append
s := []int{1, 2, 3, 4, 5}
sub := s[0:2]           // [1 2]
sub = append(sub, 100)  // Modifies s[2]! s becomes [1 2 100 4 5]
```

### Common Slice Operations

```go
// Clear slice
s = s[:0]  // Keep underlying array, reset length

// Clone
clone := append([]int(nil), s...)

// Delete i-th element
i := 2
s = append(s[:i], s[i+1:]...)

// Delete (maintain order, no new slice)
s = append(s[:i], s[i+1:]...)  // Operate directly on the original slice

// Insert
s = append(s, 0)
copy(s[i+1:], s[i:])
s[i] = value
```

---

## 3. Map

Map is a collection of key-value pairs, similar to dictionaries/hash tables in other languages.

### Creating

```go
// Literal
m := map[string]int{
    "a": 1,
    "b": 2,
}

// make
m := make(map[string]int)
m["a"] = 1

// With capacity
m := make(map[string]int, 100)  // Pre-allocate for better performance
```

### Common Operations

```go
m := make(map[string]int)

// CRUD
m["a"] = 1                    // Add/Modify
delete(m, "a")                // Delete
delete(m, "not_exists")       // Delete non-existent key, safe

// Query
v := m["a"]                   // Returns zero value (0) if not exist
v, ok := m["a"]               // ok checks if key exists
if v, ok := m["a"]; ok {
    fmt.Println("exists:", v)
}

// Length
len(m)

// Iteration (unordered!)
for k, v := range m {
    fmt.Println(k, v)
}
```

### Important Map Properties

```go
// 1. Zero value is nil, cannot write directly
var m map[string]int
// m["a"] = 1  // panic: assignment to nil map
m = make(map[string]int)  // Must initialize first

// 2. Reading a non-existent key returns zero value
fmt.Println(m["not_exists"])  // 0 (safe)

// 3. Map is a reference type
m2 := m
m2["a"] = 100    // Affects m

// 4. Maps cannot be compared directly
// fmt.Println(m == m2)  // Compilation error

// 5. Iteration order is not guaranteed
for k, v := range m {
    _ = k; _ = v
}
```

---

## 4. Zero Value vs nil

```go
var (
    arr [3]int    // [0 0 0] not nil
    sl  []int     // nil
    m   map[int]string // nil
)
```

- `nil` slice: length and capacity are both 0, append allocates a new underlying array
- `nil` map: cannot write data

```go
var s []int
fmt.Println(s == nil)  // true
fmt.Println(len(s))    // 0
s = append(s, 1)       // ✅ nil slice can be appended

var m map[string]int
m["a"] = 1             // ❌ panic: cannot write to nil map
```

---

## 5. Performance Tips

```go
// 1. Pre-allocate slice capacity
// ❌
var s []int
for i := 0; i < 10000; i++ {
    s = append(s, i)  // Multiple allocations
}

// ✅
s := make([]int, 0, 10000)
for i := 0; i < 10000; i++ {
    s = append(s, i)  // Zero allocations
}

// 2. Pre-allocate map capacity
// ❌
m := make(map[string]int)
for i := 0; i < 10000; i++ {
    m[fmt.Sprintf("key%d", i)] = i
}

// ✅
m := make(map[string]int, 10000)

// 3. Pass by value for small arrays, use pointers for large structs
// If the slice is large, use a pointer to avoid copying
func process(data *[]int) { ... }
```

---

## Wrap-up

Slices and maps are the two most commonly used data structures in Go. Slices are an abstraction layer over arrays, flexible and efficient; maps provide O(1) key-value lookup. Understanding their underlying structure and growth mechanisms is crucial for writing high-performance code.

---

