---
title: "Go from Beginner to Pro · Part 2: Variables, Constants and Data Types"
date: 2025-01-19
weight: 1202
draft: false
tags: ["go"]
---

## 1. Variable Declaration

Go has four ways to declare variables:

### 1. var Declaration

```go
var name string = "Zhang San"
var age int = 25
```

### 2. Type Inference

```go
var name = "Zhang San"  // Automatically inferred as string
var age = 25            // Automatically inferred as int
```

### 3. Batch Declaration

```go
var (
    name   string = "Zhang San"
    age    int    = 25
    height        = 175.5  // float64
    active        = true    // bool
)
```

### 4. Short Variable Declaration (Recommended)

```go
name := "Zhang San"    // Equivalent to var name = "Zhang San"
age := 25
```

Short variable declaration is the most common way in Go, but can only be used inside functions.

### Variable Rules

```go
// 1. Declared variables must be used, otherwise compilation error
// var x int  // ❌ Compilation error: x declared but not used

// 2. Multiple variable assignment
x, y := 1, 2
x, y = y, x  // Swap values, no temporary variable needed

// 3. _ ignores return values
_, err := doSomething()

// 4. Zero value initialization
var a int       // 0
var b string    // ""
var c bool      // false
var d float64   // 0
var e []int     // nil
```

---

## 2. Constants

```go
// Single constant
const Pi = 3.14159

// Batch constants
const (
    StatusOK    = 200
    StatusNotFound = 404
    StatusError    = 500
)

// Constants can be used as enums
const (
    Monday    = 1
    Tuesday   = 2
    Wednesday = 3
)
```

### iota Enum

```go
const (
    Monday = iota       // 0
    Tuesday             // 1
    Wednesday           // 2
    Thursday            // 3
    Friday              // 4
    Saturday            // 5
    Sunday              // 6
)

const (
    _  = iota             // Ignore 0
    KB = 1 << (10 * iota) // 1 << 10 = 1024
    MB                    // 1 << 20 = 1048576
    GB                    // 1 << 30
    TB                    // 1 << 40
)
```

Constants are determined at compile time and cannot use runtime expressions:

```go
const x = len("hello")  // ✅ Compile-time function
const y = time.Now()    // ❌ Runtime function, compilation error
```

---

## 3. Basic Data Types

### Boolean

```go
var active bool = true
var done bool   // Defaults to false

// Logical operations
!active         // NOT
active && done  // AND
active || done  // OR
```

### Integer Types

```go
// Signed
var a int     // Platform-dependent: 32-bit system = 32bit, 64-bit system = 64bit
var b int8    // -128 ~ 127
var c int16   // -32768 ~ 32767
var d int32   // -2^31 ~ 2^31-1
var e int64   // -2^63 ~ 2^63-1

// Unsigned
var f uint    // Same as int, but non-negative
var g uint8   // 0 ~ 255 (alias for byte)
var h uint16  // 0 ~ 65535
var i uint32  // 0 ~ 2^32-1
var j uint64  // 0 ~ 2^64-1

// Special
var k byte = 255       // Alias for uint8
var l rune = '中'      // Alias for int32, represents Unicode code point
```

**Common Choices**:
- Use `int` for integers
- Use `byte` for binary bytes
- Use `rune` for Unicode characters

### Floating Point Types

```go
var f1 float32        // ~7 digits precision
var f2 float64        // ~15 digits precision (recommended)

// Scientific notation
var f3 = 1.5e10       // 15000000000
var f4 = 1.5E-3       // 0.0015

// Special values
var inf = math.Inf(1)  // Positive infinity
var nan = math.NaN()   // Not a number
```

**Floating Point Comparison Pitfall**:

```go
// ❌ Don't compare floating point numbers directly
if 0.1 + 0.2 == 0.3 {
    fmt.Println("equal")  // Won't execute!
}

// ✅ Use epsilon comparison
const epsilon = 1e-9
if math.Abs((0.1+0.2)-0.3) < epsilon {
    fmt.Println("approximately equal")
}
```

### Strings

```go
// Double-quoted strings
var s1 = "hello"

// Backtick raw strings (no escaping, supports newlines)
var s2 = `first line
second line
third line`

// String concatenation
s := "hello, " + "world"

// Length
len(s)  // Returns byte count, not character count

// Index (get byte)
s[0]    // 'h'

// Substring
s[0:5]  // "hello"

// Iteration (by byte)
for i := 0; i < len(s); i++ {
    fmt.Printf("%x ", s[i])
}

// Iteration (by rune)
for i, r := range "hello世界" {
    fmt.Printf("%d %c\n", i, r)
}
```

### Strings Are Immutable

```go
s := "hello"
// s[0] = 'H'  // ❌ Compilation error

// Convert to []byte when modification is needed
b := []byte(s)
b[0] = 'H'
s = string(b)  // "Hello"
```

---

## 4. Type Conversion

Go has **no** implicit type conversion, explicit conversion is required:

```go
var a int = 10
var b int64 = 20

// a = b          // ❌ Compilation error: type mismatch
a = int(b)        // ✅ Explicit conversion

// Safe conversion range
var x int32 = 1000
var y int8 = int8(x)  // Compiles but may overflow (if x > 127)

// Number to string
s := strconv.Itoa(100)          // "100"
n, _ := strconv.Atoi("100")     // 100

// String and []byte
s := "hello"
b := []byte(s)      // string → []byte
s2 := string(b)     // []byte → string

// Float to integer (truncates decimal)
f := 3.14
i := int(f)         // 3
```

---

## 5. Zero Values

Go variables are automatically initialized to zero values after declaration — **there are no uninitialized variables**:

```go
var i int       // 0
var f float64   // 0
var b bool      // false
var s string    // ""
var p *int      // nil
var sl []int    // nil
var m map[int]string // nil
var ch chan int // nil
```

---

## 6. Naming Conventions

```go
// Camel case naming
userName := "Zhang San"
userAge := 25

// Package-level variables
var version = "1.0.0"

// Capitalized first letter = public (exported)
var PublicVar = "I'm public"

// Lowercase first letter = private (unexported)
var privateVar = "I'm private"

// Abbreviations keep consistent casing
var userID int           // ✅
var userId int           // ❌ Not used
var urlString string     // URL as a whole, uppercase as URLString
```

---

## Wrap-up

This article covered Go's variables, constants, data types, and type conversion. Go's type system is simple but rigorous — no implicit conversion, no uninitialized variables, and type inference reduces declaration burden.

---

