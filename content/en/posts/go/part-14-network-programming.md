---
title: "Go from Beginner to Pro · Part 14: Network Programming"
date: 2025-01-06
weight: 1214
draft: false
tags: ["go"]
---

## 1. HTTP Client

### Basic GET Request

```go
resp, err := http.Get("https://api.github.com/users/golang")
if err != nil {
    log.Fatal(err)
}
defer resp.Body.Close()

body, err := io.ReadAll(resp.Body)
fmt.Println(string(body))
```

### Requests with Parameters

```go
// Query parameters
req, _ := http.NewRequest("GET", "https://api.example.com/search", nil)
q := req.URL.Query()
q.Add("q", "golang")
q.Add("page", "1")
req.URL.RawQuery = q.Encode()

// Custom Headers
req.Header.Set("Authorization", "Bearer token123")
req.Header.Set("User-Agent", "MyApp/1.0")

// Send
resp, err := http.DefaultClient.Do(req)
```

### POST JSON

```go
data := map[string]interface{}{
    "name":  "Zhang San",
    "email": "zhangsan@example.com",
}

jsonData, _ := json.Marshal(data)
resp, err := http.Post(
    "https://api.example.com/users",
    "application/json",
    bytes.NewBuffer(jsonData),
)
```

### POST Form

```go
resp, err := http.PostForm(
    "https://api.example.com/login",
    url.Values{
        "username": {"admin"},
        "password": {"123456"},
    },
)
```

### Timeout Control

```go
// Method 1: Client-level timeout
client := &http.Client{
    Timeout: 10 * time.Second,
}
resp, err := client.Get("https://api.example.com")

// Method 2: Request-level timeout (Context)
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

req, _ := http.NewRequestWithContext(ctx, "GET", "https://api.example.com", nil)
resp, err := http.DefaultClient.Do(req)
```

### Complete HTTP Client

```go
type Client struct {
    httpClient *http.Client
    baseURL    string
}

func NewClient(baseURL string, timeout time.Duration) *Client {
    return &Client{
        httpClient: &http.Client{
            Timeout: timeout,
            Transport: &http.Transport{
                MaxIdleConns:        100,
                IdleConnTimeout:     90 * time.Second,
                TLSHandshakeTimeout: 10 * time.Second,
            },
        },
        baseURL: baseURL,
    }
}

func (c *Client) Get(path string) ([]byte, error) {
    url := c.baseURL + path
    resp, err := c.httpClient.Get(url)
    if err != nil {
        return nil, fmt.Errorf("GET %s failed: %w", url, err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("unexpected status: %d", resp.StatusCode)
    }

    return io.ReadAll(resp.Body)
}
```

---

## 2. HTTP Server

### Simplest HTTP Server

```go
func helloHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, %s!", r.URL.Path[1:])
}

func main() {
    http.HandleFunc("/", helloHandler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Complete REST Service

```go
var users = []User{
    {ID: 1, Name: "Zhang San"},
    {ID: 2, Name: "Li Si"},
}

func getUsers(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func createUser(w http.ResponseWriter, r *http.Request) {
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    user.ID = len(users) + 1
    users = append(users, user)
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}

// Standard library routing (Go 1.22+ supports method routing)
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("GET /api/users", getUsers)
    mux.HandleFunc("POST /api/users", createUser)

    log.Println("Server starting :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

### Middleware

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}

func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token != "Bearer secret-token" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        next.ServeHTTP(w, r)
    })
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("GET /api/users", getUsers)

    // Chained middleware
    handler := loggingMiddleware(authMiddleware(mux))
    http.ListenAndServe(":8080", handler)
}
```

---

## 3. JSON API

### Returning JSON

```go
func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{"error": message})
}
```

### Reading JSON

```go
func decodeJSON(r *http.Request, v interface{}) error {
    decoder := json.NewDecoder(r.Body)
    decoder.DisallowUnknownFields()  // Reject unknown fields
    return decoder.Decode(v)
}
```

---

## 4. TCP Programming

```go
// TCP Server
func main() {
    listener, err := net.Listen("tcp", ":8080")
    if err != nil {
        log.Fatal(err)
    }
    defer listener.Close()

    for {
        conn, err := listener.Accept()
        if err != nil {
            log.Println(err)
            continue
        }
        go handleConnection(conn)
    }
}

func handleConnection(conn net.Conn) {
    defer conn.Close()

    // Set timeout
    conn.SetDeadline(time.Now().Add(10 * time.Second))

    buf := make([]byte, 1024)
    n, err := conn.Read(buf)
    if err != nil {
        log.Println(err)
        return
    }

    fmt.Printf("received: %s\n", string(buf[:n]))
    conn.Write([]byte("Received!"))
}

// TCP Client
func main() {
    conn, err := net.DialTimeout("tcp", "localhost:8080", 5*time.Second)
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    conn.Write([]byte("Hello, Server!"))
    buf := make([]byte, 1024)
    n, _ := conn.Read(buf)
    fmt.Println(string(buf[:n]))
}
```

---

## 5. Context Timeout Control

```go
// Server-side timeout
srv := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  10 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  60 * time.Second,
}

log.Fatal(srv.ListenAndServe())

// Graceful shutdown
func main() {
    srv := &http.Server{Addr: ":8080"}

    go func() {
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    log.Println("Shutting down server...")
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal("Server forced to shutdown:", err)
    }
    log.Println("Server exited")
}
```

---

## Wrap-up

This article covered HTTP client/server, JSON API, TCP programming, and graceful shutdown. Go's standard library `net/http` is sufficient for building production-grade HTTP services — you don't need third-party frameworks to write clean REST APIs.

---

