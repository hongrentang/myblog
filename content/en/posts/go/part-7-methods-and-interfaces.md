---
title: "Go from Beginner to Pro · Part 7: Methods and Interfaces"
date: 2025-01-24
weight: 7
draft: false
tags: ["go"]
---

## 1. Methods

Methods are **functions with a receiver**.

### Definition

```go
type User struct {
    Name string
    Age  int
}

// Value receiver
func (u User) SayHello() {
    fmt.Printf("Hello, I am %s\n", u.Name)
}

// Pointer receiver
func (u *User) Birthday() {
    u.Age++
}
```

### Value Receiver vs Pointer Receiver

```go
type Counter struct {
    value int
}

// Value receiver — does not modify original value
func (c Counter) Value() int {
    return c.value
}

// Pointer receiver — modifies original value
func (c *Counter) Inc() {
    c.value++
}

c := Counter{value: 10}
c.Inc()
fmt.Println(c.Value())  // 11
```

**Selection Principles**:

| Scenario | Receiver Type |
|----------|--------------|
| Does not modify original object | Value receiver |
| Needs to modify original object | Pointer receiver |
| Large struct (expensive to copy) | Pointer receiver |
| Method needs to handle nil | Pointer receiver |

**Consistency Principle**: If a type has pointer receiver methods, typically all methods should use pointer receivers.

### Method Invocation

```go
u := User{Name: "Zhang San", Age: 25}
u.SayHello()           // Value call
u.Birthday()           // Go automatically takes address &u

p := &User{Name: "Li Si"}
p.SayHello()           // Go automatically dereferences (*p)
p.Birthday()
```

## 2. Interfaces

An interface defines a set of method signatures; types that implement these methods automatically satisfy the interface.

### Definition and Implementation

```go
// Define interface
type Animal interface {
    Speak() string
    Move() string
}

// Implement interface (no explicit implements declaration needed)
type Dog struct{}

func (d Dog) Speak() string { return "Woof!" }
func (d Dog) Move() string  { return "Run" }

type Cat struct{}

func (c Cat) Speak() string { return "Meow~" }
func (c Cat) Move() string  { return "Walk" }

// Using the interface
func MakeSound(a Animal) {
    fmt.Printf("%s says: %s\n", a.Move(), a.Speak())
}

func main() {
    dog := Dog{}
    cat := Cat{}
    MakeSound(dog)  // Run says: Woof!
    MakeSound(cat)  // Walk says: Meow~
}
```

### Interfaces Are Implicitly Implemented

This is the core of Go's interface design — types don't need to declare that they implement an interface, as long as the methods match.

```go
// Standard library example: io.Reader
type Reader interface {
    Read(p []byte) (n int, err error)
}

// os.File implements Reader
// No need to write: type File struct ... implements io.Reader
// As long as File has a Read method, it can be used as a Reader
```

### Interface Values

Internally, an interface consists of two parts: **concrete type** + **concrete value**:

```go
var a Animal   // nil interface

a = Dog{}
// a internally: (Dog, Dog{})

a = Cat{}
// a internally: (Cat, Cat{})

// Interface value is nil only when both type and value are nil
var p *Dog = nil
a = p          // a internally: (*Dog, nil)
fmt.Println(a == nil)  // false! Because the type is not nil
```

### Empty Interface

```go
// interface{} can accept any type
var anything interface{}

anything = 42
anything = "hello"
anything = Dog{}

// Type assertion to retrieve concrete type
n := anything.(Dog)    // Unsafe, panics if mismatch
d, ok := anything.(Dog)  // Safe
if ok {
    fmt.Println(d.Speak())
}
```

### type switch

```go
func describe(v interface{}) {
    switch v.(type) {
    case int:
        fmt.Println("integer:", v)
    case string:
        fmt.Println("string:", v)
    case Dog:
        fmt.Println("dog:", v.(Dog).Speak())
    default:
        fmt.Printf("unknown type: %T\n", v)
    }
}
```

### Common Standard Library Interfaces

```go
// io.Reader / io.Writer
type Reader interface {
    Read(p []byte) (n int, err error)
}
type Writer interface {
    Write(p []byte) (n int, err error)
}

// error
type error interface {
    Error() string
}

// Stringer (controls print format)
type Stringer interface {
    String() string
}
```

### Implementing Stringer

```go
type User struct {
    Name string
    Age  int
}

func (u User) String() string {
    return fmt.Sprintf("User(%s, %d)", u.Name, u.Age)
}

u := User{"Zhang San", 25}
fmt.Println(u)  // User(Zhang San, 25)
```

---

## 3. Type Assertion

```go
var v interface{} = "hello"

// Direct assertion (not recommended, panic risk)
s := v.(string)

// Safe assertion (recommended)
s, ok := v.(string)
if ok {
    fmt.Println(s)
} else {
    fmt.Println("not a string")
}

// Assert to interface
var r io.Reader = os.Stdin
if f, ok := r.(*os.File); ok {
    fmt.Println(f.Name())  // Get file name
}
```

---

## 4. Interface Embedding

Interfaces can be composed into larger interfaces through embedding:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}

// Composite interface
type ReadWriter interface {
    Reader
    Writer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

In the standard library, `io.ReadWriter`, `io.ReadWriteCloser` are all composed this way.

---

## 5. Interface Design Principles

### Small Interface Principle

```go
// ✅ Go philosophy: small interfaces
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// ❌ Large interface not recommended
type LargeInterface interface {
    Read(p []byte) (n int, err error)
    Write(p []byte) (n int, err error)
    Close() error
    Flush() error
    Seek(offset int64, whence int) (int64, error)
}
```

### Accept Interfaces, Return Structs

```go
// Function parameters use interfaces, return values use concrete types
func Process(r io.Reader) (*Result, error) {
    data, err := io.ReadAll(r)
    if err != nil {
        return nil, err
    }
    return &Result{Data: data}, nil
}
```

### Checking Interface Implementation

Sometimes you need to statically ensure a type implements an interface:

```go
// Compile-time check: if *File does not implement io.ReadWriter, compilation fails
var _ io.ReadWriter = (*os.File)(nil)
```

---

## 6. Interface Performance

```go
// Interface calls have a small overhead (dynamic dispatch)
// Consider avoiding interfaces on hot paths

// Benchmark
func BenchmarkDirect(b *testing.B) {
    d := Dog{}
    for i := 0; i < b.N; i++ {
        d.Speak()
    }
}

func BenchmarkInterface(b *testing.B) {
    var a Animal = Dog{}
    for i := 0; i < b.N; i++ {
        a.Speak()
    }
}
// Interface calls are about 5-15% slower, negligible in most scenarios
```

---

## Wrap-up

Methods are functions with a receiver; interfaces are collections of method signatures and are Go's way of achieving polymorphism. Go's interfaces are implicitly implemented, loosely coupled, and flexible. The **small interface principle** is the core idea of Go interface design.

---

