---
title: "Go from Beginner to Pro · Part 16: Reflection and Generics"
date: 2025-01-08
weight: 16
draft: false
tags: ["go"]
---

## 1. Reflection

Reflection (reflect) allows a program to examine types and values at runtime.

### reflect.Type and reflect.Value

```go
import "reflect"

func inspect(v interface{}) {
    t := reflect.TypeOf(v)
    fmt.Println("type:", t.Name())
    fmt.Println("kind:", t.Kind())

    val := reflect.ValueOf(v)
    fmt.Println("value:", val)
}

inspect(42)
// type: int
// kind: int

inspect("hello")
// type: string
// kind: string
```

### Reading Struct Fields

```go
type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
    Age  int    `json:"age,omitempty"`
}

u := User{ID: 1, Name: "Zhang San", Age: 25}
t := reflect.TypeOf(u)

for i := 0; i < t.NumField(); i++ {
    field := t.Field(i)
    fmt.Printf("field: %s, type: %v, Tag: %s\n",
        field.Name, field.Type, field.Tag.Get("json"))
}
// field: ID, type: int, Tag: id
// field: Name, type: string, Tag: name
// field: Age, type: int, Tag: age,omitempty
```

### Modifying Values (via Pointer)

```go
x := 42
v := reflect.ValueOf(&x)  // Must be a pointer
v.Elem().SetInt(100)
fmt.Println(x)  // 100
```

### Calling Methods

```go
type Calculator struct{}

func (c Calculator) Add(a, b int) int {
    return a + b
}

c := Calculator{}
v := reflect.ValueOf(c)
method := v.MethodByName("Add")
args := []reflect.Value{reflect.ValueOf(3), reflect.ValueOf(4)}
result := method.Call(args)
fmt.Println(result[0].Int())  // 7
```

### Use Cases

**1. Generic Serialization/Deserialization**

```go
func toMap(v interface{}) map[string]interface{} {
    result := make(map[string]interface{})
    val := reflect.ValueOf(v)

    if val.Kind() == reflect.Ptr {
        val = val.Elem()
    }

    t := val.Type()
    for i := 0; i < val.NumField(); i++ {
        field := t.Field(i)
        result[field.Name] = val.Field(i).Interface()
    }
    return result
}
```

**2. Validator**

```go
func validate(v interface{}) error {
    val := reflect.ValueOf(v)
    if val.Kind() == reflect.Ptr {
        val = val.Elem()
    }

    t := val.Type()
    for i := 0; i < val.NumField(); i++ {
        field := t.Field(i)
        value := val.Field(i)

        // Check required tag
        if field.Tag.Get("validate") == "required" && value.IsZero() {
            return fmt.Errorf("field %s cannot be empty", field.Name)
        }
    }
    return nil
}

type Request struct {
    Name  string `validate:"required"`
    Email string `validate:"required"`
}
```

---

## 2. Generics

Go 1.18 introduced generics.

### Generic Functions

```go
// Regular function — needs a version for each type
func sumInts(nums []int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

func sumFloats(nums []float64) float64 {
    total := 0.0
    for _, n := range nums {
        total += n
    }
    return total
}

// Generic function — one solution for all
func sum[T int | float64](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}

// Usage
fmt.Println(sum([]int{1, 2, 3}))           // 6
fmt.Println(sum([]float64{1.5, 2.5, 3})) // 7.0
```

### Generic Types

```go
// Generic stack
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

func (s *Stack[T]) Peek() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    return s.items[len(s.items)-1], true
}

// Usage
intStack := Stack[int]{}
intStack.Push(10)
intStack.Push(20)
val, _ := intStack.Pop()
fmt.Println(val)  // 20

strStack := Stack[string]{}
strStack.Push("hello")
```

### Generic Collections

```go
// Type-safe Set
type Set[T comparable] struct {
    items map[T]struct{}
}

func NewSet[T comparable]() *Set[T] {
    return &Set[T]{items: make(map[T]struct{})}
}

func (s *Set[T]) Add(item T) {
    s.items[item] = struct{}{}
}

func (s *Set[T]) Remove(item T) {
    delete(s.items, item)
}

func (s *Set[T]) Has(item T) bool {
    _, ok := s.items[item]
    return ok
}

func (s *Set[T]) Items() []T {
    result := make([]T, 0, len(s.items))
    for item := range s.items {
        result = append(result, item)
    }
    return result
}
```

### Generic Constraints

```go
// Define constraint interface
type Number interface {
    int | int64 | float64
}

// Use constraint
func sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}

// ~ in constraint means underlying type
type MyInt int

// Only accepts int
// func add[T int](a, b T) T { return a + b }
// MyInt cannot use this function

// Accepts all types with underlying type int
func add[T ~int](a, b T) T { return a + b }
// MyInt can also use this

// Common constraints
// any = interface{}
// comparable = comparable (== and !=)
// Ordered = sortable (golang.org/x/exp/constraints)
```

### Generic Methods (Limitation)

```go
// Note: Generic methods cannot have additional type parameters
type Container struct{}

// ❌ Methods cannot have additional generic parameters
// func (c Container) Transform[T any](item T) T { ... }

// ✅ Must put generics on the type
type Container2[T any] struct {
    value T
}

func (c Container2[T]) Value() T {
    return c.value
}
```

---

## 3. Reflection vs Generics

| Comparison | Reflection | Generics |
|-----------|-----------|----------|
| Timing | Runtime | Compile time |
| Performance | Has overhead | Zero overhead (compile-time expansion) |
| Type Safety | Unsafe (needs assertion) | Type safe |
| Use Cases | Serialization, ORM, unknown types | Containers, algorithms, data structures |
| Code Complexity | Complex | Concise |

**Recommendation**: If a problem can be solved with generics, don't use reflection. Generics are type-safe at compile time and have no runtime overhead.

---

## 4. Practical Patterns

### Generics + Interface Combination

```go
// Generic cache interface
type Cache[T any] interface {
    Get(key string) (T, bool)
    Set(key string, value T)
}

// In-memory implementation
type MemoryCache[T any] struct {
    data map[string]T
    mu   sync.RWMutex
}

func NewMemoryCache[T any]() *MemoryCache[T] {
    return &MemoryCache[T]{data: make(map[string]T)}
}

func (c *MemoryCache[T]) Get(key string) (T, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.data[key]
    return v, ok
}

func (c *MemoryCache[T]) Set(key string, value T) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[key] = value
}
```

---

## Wrap-up

Reflection gives Go runtime introspection capabilities, suitable for frameworks, ORM, serialization, etc., but has performance overhead. Generics (1.18+) provide compile-time polymorphism, are type-safe and have zero overhead, suitable for containers, algorithms, and data structures. Each has its own use cases — if a problem can be solved with generics, don't use reflection.

---

