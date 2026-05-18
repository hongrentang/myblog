---
title: "Go from Beginner to Pro · Part 18: Common Standard Library"
date: 2025-01-10
weight: 18
draft: false
tags: ["go"]
---

## 1. time

### Basic Operations

```go
// Current time
now := time.Now()

// Structured time
t := time.Date(2025, time.January, 15, 10, 30, 0, 0, time.Local)

// Formatting (Go's reference time is 2006-01-02 15:04:05)
fmt.Println(t.Format("2006-01-02 15:04:05"))
fmt.Println(t.Format("2006/01/02"))
fmt.Println(t.Format("15:04"))

// Parsing
t, _ = time.Parse("2006-01-02", "2025-01-15")
t, _ = time.ParseInLocation("2006-01-02", "2025-01-15", time.Local)
```

### Time Calculations

```go
// Addition/Subtraction
tomorrow := time.Now().Add(24 * time.Hour)
lastWeek := time.Now().AddDate(0, 0, -7)
nextMonth := time.Now().AddDate(0, 1, 0)

// Time difference
elapsed := time.Since(startTime)
duration := time.Until(deadline)

// Comparison
t1.Before(t2)
t1.After(t2)
t1.Equal(t2)

// Timestamps
timestamp := time.Now().Unix()        // Seconds
milli := time.Now().UnixMilli()       // Milliseconds
micro := time.Now().UnixMicro()       // Microseconds
nano := time.Now().UnixNano()         // Nanoseconds
```

### Timers

```go
// Ticker (repeating timer)
ticker := time.NewTicker(5 * time.Second)
go func() {
    for range ticker.C {
        fmt.Println("executes every 5 seconds")
    }
}()
ticker.Stop()  // Stop

// Timer (one-time)
timer := time.NewTimer(10 * time.Second)
<-timer.C
fmt.Println("executes after 10 seconds")
timer.Stop()  // Cancel

// After simplified version
<-time.After(5 * time.Second)

// Sleep
time.Sleep(100 * time.Millisecond)
```

### Timeout

```go
func fetchWithTimeout(url string, timeout time.Duration) ([]byte, error) {
    ch := make(chan []byte, 1)
    
    go func() {
        data, _ := fetch(url)
        ch <- data
    }()
    
    select {
    case data := <-ch:
        return data, nil
    case <-time.After(timeout):
        return nil, fmt.Errorf("request timeout")
    }
}
```

---

## 2. encoding

### base64

```go
// Encode
data := []byte("hello")
encoded := base64.StdEncoding.EncodeToString(data)

// URL safe
urlSafe := base64.URLEncoding.EncodeToString(data)

// Decode
decoded, _ := base64.StdEncoding.DecodeString(encoded)
```

### hex

```go
data := []byte("hello")
encoded := hex.EncodeToString(data)  // "68656c6c6f"
decoded, _ := hex.DecodeString(encoded)
```

### binary

```go
var buf bytes.Buffer

// Write binary (big endian)
binary.Write(&buf, binary.BigEndian, int32(42))
binary.Write(&buf, binary.BigEndian, float64(3.14))

// Read
var x int32
var y float64
binary.Read(&buf, binary.BigEndian, &x)
binary.Read(&buf, binary.BigEndian, &y)
```

---

## 3. crypto

### Hashing

```go
// SHA256
hash := sha256.Sum256([]byte("data"))

// SHA256 streaming
h := sha256.New()
h.Write([]byte("part1"))
h.Write([]byte("part2"))
hash := h.Sum(nil)

// MD5 (not for security, only for checksums)
md5.Sum([]byte("data"))

// HMAC
mac := hmac.New(sha256.New, []byte("key"))
mac.Write([]byte("data"))
signature := mac.Sum(nil)
```

### Random Numbers

```go
// Secure random numbers
func generateToken(length int) (string, error) {
    bytes := make([]byte, length)
    if _, err := rand.Read(bytes); err != nil {
        return "", err
    }
    return hex.EncodeToString(bytes), nil
}
```

### Password Hashing (bcrypt)

```go
import "golang.org/x/crypto/bcrypt"

// Hash password
hash, _ := bcrypt.GenerateFromPassword([]byte("password"), bcrypt.DefaultCost)

// Verify password
err := bcrypt.CompareHashAndPassword(hash, []byte("password"))
if err == nil {
    fmt.Println("password correct")
}
```

---

## 4. flag

```go
func main() {
    // Define flags
    port := flag.Int("port", 8080, "server port")
    host := flag.String("host", "localhost", "host address")
    debug := flag.Bool("debug", false, "debug mode")
    
    // Parse
    flag.Parse()
    
    // Use
    fmt.Printf("Starting %s:%d (debug=%v)\n", *host, *port, *debug)
    
    // Other arguments
    fmt.Println("other args:", flag.Args())
}

// go run main.go -port 9090 -debug=true -- extra1 extra2
```

---

## 5. sort

```go
nums := []int{3, 1, 4, 1, 5}
sort.Ints(nums)              // [1 1 3 4 5]

strs := []string{"c", "a", "b"}
sort.Strings(strs)           // [a b c]

// Custom sorting
users := []User{
    {Name: "Zhang San", Age: 30},
    {Name: "Li Si", Age: 25},
}
sort.Slice(users, func(i, j int) bool {
    return users[i].Age < users[j].Age
})

// Search
index := sort.SearchInts(nums, 4)  // Position of 4

// Reverse
sort.Sort(sort.Reverse(sort.IntSlice(nums)))
```

---

## 6. sync/atomic

```go
var counter atomic.Int64

// Atomic operations
counter.Add(1)
counter.Load()
counter.Store(100)
counter.Swap(200)
counter.CompareAndSwap(200, 300)

// Legacy atomic package
var count int64
atomic.AddInt64(&count, 1)
atomic.LoadInt64(&count)
```

---

## 7. regexp

```go
// Compile (once, use many times)
re := regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)

// Match
matched := re.MatchString("test@example.com")

// Extract
re = regexp.MustCompile(`(\d{4})-(\d{2})-(\d{2})`)
matches := re.FindStringSubmatch("2025-01-15")
// ["2025-01-15", "2025", "01", "15"]

// Replace
result := re.ReplaceAllString("2025-01-15", "$2/$3/$1")
// "01/15/2025"
```

---

## 8. context Additional

```go
// Passing request-scoped values
type contextKey string

const userKey contextKey = "user"

func WithUser(ctx context.Context, user User) context.Context {
    return context.WithValue(ctx, userKey, user)
}

func GetUser(ctx context.Context) (User, bool) {
    user, ok := ctx.Value(userKey).(User)
    return user, ok
}

// Usage
ctx := context.Background()
ctx = WithUser(ctx, User{Name: "Zhang San"})

if user, ok := GetUser(ctx); ok {
    fmt.Println("current user:", user.Name)
}
```

---

## Wrap-up

Go's standard library is feature-rich, covering the vast majority of daily development needs. Packages like time, encoding, crypto, sort, regexp, and context are of high quality — before introducing third-party libraries, check if the standard library can meet your needs.

---

