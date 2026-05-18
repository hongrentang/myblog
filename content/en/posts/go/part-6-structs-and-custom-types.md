---
title: "Go from Beginner to Pro · Part 6: Structs and Custom Types"
date: 2025-01-23
weight: 6
draft: false
tags: ["go"]
---

## 1. Structs

Go has no concept of "classes"; it uses structs to organize data.

### Definition and Declaration

```go
// Definition
type User struct {
    ID       int
    Name     string
    Email    string
    Age      int
    CreatedAt time.Time
}

// Declaration method 1: literal
u := User{
    ID:    1,
    Name:  "Zhang San",
    Email: "zhangsan@example.com",
    Age:   25,
}

// Declaration method 2: positional (not recommended, breaks if field order changes)
u := User{1, "Zhang San", "zhangsan@example.com", 25}

// Declaration method 3: zero value + individual assignment
var u User
u.ID = 1
u.Name = "Zhang San"
```

### Nested Structs

```go
type Address struct {
    Province string
    City     string
    Detail   string
}

type User struct {
    ID      int
    Name    string
    Address Address  // Nesting
}

// Initialization
u := User{
    ID:   1,
    Name: "Zhang San",
    Address: Address{
        Province: "Guangdong",
        City:     "Shenzhen",
        Detail:   "Nanshan District",
    },
}

// Access nested fields
fmt.Println(u.Address.City)
```

### Anonymous Structs

```go
// Temporary struct, no type definition needed
u := struct {
    Name string
    Age  int
}{
    Name: "Zhang San",
    Age:  25,
}

// Anonymous nesting
type User struct {
    ID   int
    Name string
    Address struct {  // Anonymous embedded field
        Province string
        City     string
    }
}
```

### Structs Are Value Types

```go
u1 := User{Name: "Zhang San"}
u2 := u1           // Copies the entire struct
u2.Name = "Li Si"
fmt.Println(u1.Name)  // "Zhang San" (unchanged)

// Use pointers to modify
u3 := &User{Name: "Zhang San"}
u4 := u3
u4.Name = "Li Si"
fmt.Println(u3.Name)  // "Li Si"
```

---

## 2. Struct Tags

Tags are metadata for struct fields, commonly used for JSON serialization/ORM mapping:

```go
type User struct {
    ID        int       `json:"id"`
    Name      string    `json:"name,omitempty"`  // Omit if empty
    Email     string    `json:"email"`
    Password  string    `json:"-"`               // Ignore this field
    CreatedAt time.Time `json:"created_at"`
}

u := User{
    ID:    1,
    Name:  "Zhang San",
    Email: "test@test.com",
}

data, _ := json.Marshal(u)
fmt.Println(string(data))
// {"id":1,"name":"Zhang San","email":"test@test.com","created_at":"0001-01-01T00:00:00Z"}
```

### Common Tag Formats

```go
// JSON
`json:"field_name,omitempty"`
`json:"-"`

// Database ORM (GORM)
`gorm:"column:user_name;type:varchar(100);not null"`

// Form validation (validator)
`validate:"required,min=3,max=50"`

// YAML
`yaml:"field_name"`

// XML
`xml:"field_name,attr"`
```

---

## 3. Custom Types

### type Definition

```go
// Define new types based on existing types
type MyInt int
type UserList []User
type Handler func(string) error

var age MyInt = 25
fmt.Printf("%T\n", age)  // main.MyInt

// Different types with the same underlying type cannot be implicitly converted
var a MyInt = 10
var b int = 20
// a = b  // ❌ Compilation error
a = MyInt(b)  // ✅ Explicit conversion
```

### Uses of Custom Types

```go
// Add methods to a type
type Counter int

func (c *Counter) Inc() {
    *c++
}

func (c *Counter) Value() int {
    return int(*c)
}

// Add methods to a slice
type Queue []int

func (q *Queue) Push(v int) {
    *q = append(*q, v)
}

func (q *Queue) Pop() (int, bool) {
    if len(*q) == 0 {
        return 0, false
    }
    v := (*q)[0]
    *q = (*q)[1:]
    return v, true
}
```

---

## 4. Factory Functions

Go has no constructors; factory functions are used instead:

```go
type User struct {
    id   int    // lowercase = private
    name string
}

// Factory function
func NewUser(name string) *User {
    return &User{
        id:   generateID(),
        name: name,
    }
}

// Factory function with options
func NewUserWithOpts(name, email string, age int) *User {
    u := &User{
        id:    generateID(),
        name:  name,
        email: email,
        age:   age,
    }
    u.validate()
    return u
}
```

---

## 5. Struct Comparison

```go
type Point struct {
    X, Y int
}

// Comparable
p1 := Point{1, 2}
p2 := Point{1, 2}
p3 := Point{3, 4}
fmt.Println(p1 == p2)  // true
fmt.Println(p1 == p3)  // false
```

If a struct contains non-comparable fields (slice, map, func), it cannot be compared directly:

```go
type Bad struct {
    Data []int  // slice cannot be compared
}
// b1 == b2  // Compilation error

// Use reflect.DeepEqual
b1 := Bad{Data: []int{1, 2}}
b2 := Bad{Data: []int{1, 2}}
fmt.Println(reflect.DeepEqual(b1, b2))  // true
```

---

## 6. Empty Struct

Empty structs use zero memory and are often used for signaling:

```go
// Empty struct occupies 0 bytes
empty := struct{}{}

// Using map to implement a set
type Set struct {
    m map[string]struct{}
}

func (s *Set) Add(key string) {
    s.m[key] = struct{}{}
}

func (s *Set) Has(key string) bool {
    _, ok := s.m[key]
    return ok
}

// Channel signaling
ch := make(chan struct{})
go func() {
    // do something
    ch <- struct{}{}
}()
<-ch  // Wait for completion
```

---

## 7. Struct Memory Alignment

The Go compiler inserts padding bytes between struct fields for memory alignment:

```go
// Large fields first, then small fields (more compact)
type WellAligned struct {
    A int64    // 8 bytes
    B float64  // 8 bytes
    C int32    // 4 bytes
    D int16    // 2 bytes
    E bool     // 1 byte
}
// Total 24 bytes (8+8+4+2+1+1padding)

// Out of order (wasteful)
type BadAligned struct {
    A bool     // 1 byte + 7 padding
    B int64    // 8 bytes
    C bool     // 1 byte + 3 padding
    D int32    // 4 bytes
    E bool     // 1 byte + 7 padding
}
// Total 32 bytes

// Use unsafe.Sizeof to check
fmt.Println(unsafe.Sizeof(WellAligned{}))  // 24
fmt.Println(unsafe.Sizeof(BadAligned{}))   // 32
```

---

## Wrap-up

Structs are the primary way Go organizes and encapsulates data. Combined with tags for serialization, factory functions replacing constructors, empty structs saving memory, and memory alignment improving performance — these are all advanced techniques in Go development.

---

