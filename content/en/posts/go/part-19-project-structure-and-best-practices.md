---
title: "Go from Beginner to Pro · Part 19: Project Structure and Best Practices"
date: 2025-01-11
weight: 19
draft: false
tags: ["go"]
---

## 1. Standard Project Layout

```
myapp/
├── cmd/                       # Entry files
│   └── server/
│       └── main.go
├── internal/                  # Private code (cannot be imported externally)
│   ├── handler/               # HTTP handlers
│   ├── service/               # Business logic
│   ├── repository/            # Data access
│   └── model/                 # Data models
├── pkg/                       # Exportable public code
│   ├── response/
│   └── middleware/
├── config/                    # Configuration
│   └── config.go
├── migrations/                # Database migrations
├── scripts/                   # Scripts
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

---

## 2. Layered Architecture

```
handler (HTTP layer) → service (Business layer) → repository (Data layer)
```

### Handler Layer

```go
// internal/handler/user.go
type UserHandler struct {
    service *service.UserService
}

func NewUserHandler(svc *service.UserService) *UserHandler {
    return &UserHandler{service: svc}
}

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    user, err := h.service.GetUser(r.Context(), id)
    if err != nil {
        response.Error(w, http.StatusNotFound, err.Error())
        return
    }
    response.JSON(w, http.StatusOK, user)
}
```

### Service Layer

```go
// internal/service/user.go
type UserService struct {
    repo *repository.UserRepo
}

func NewUserService(repo *repository.UserRepo) *UserService {
    return &UserService{repo: repo}
}

func (s *UserService) GetUser(ctx context.Context, id string) (*model.User, error) {
    return s.repo.FindByID(ctx, id)
}
```

### Repository Layer

```go
// internal/repository/user.go
type UserRepo struct {
    db *sql.DB
}

func NewUserRepo(db *sql.DB) *UserRepo {
    return &UserRepo{db: db}
}

func (r *UserRepo) FindByID(ctx context.Context, id string) (*model.User, error) {
    // SQL query
}
```

---

## 3. Dependency Injection

### Constructor Injection

```go
// main.go
func main() {
    db := initDB()
    userRepo := repository.NewUserRepo(db)
    userSvc := service.NewUserService(userRepo)
    userHandler := handler.NewUserHandler(userSvc)

    mux := http.NewServeMux()
    mux.HandleFunc("GET /users/{id}", userHandler.GetUser)
    
    http.ListenAndServe(":8080", mux)
}
```

### Wire (Auto Dependency Injection)

```go
// wire.go
//go:build wireinject
// +build wireinject

func InitializeApp() *App {
    wire.Build(
        initDB,
        repository.NewUserRepo,
        service.NewUserService,
        handler.NewUserHandler,
        NewApp,
    )
    return nil
}
```

---

## 4. Configuration Management

```go
// config/config.go
type Config struct {
    Server   ServerConfig   `yaml:"server"`
    Database DatabaseConfig `yaml:"database"`
    Redis    RedisConfig    `yaml:"redis"`
}

type ServerConfig struct {
    Port    int           `yaml:"port"`
    Timeout time.Duration `yaml:"timeout"`
}

type DatabaseConfig struct {
    DSN string `yaml:"dsn"`
}

// Load configuration
func Load(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, err
    }

    var cfg Config
    if err := yaml.Unmarshal(data, &cfg); err != nil {
        return nil, err
    }
    return &cfg, nil
}

// Environment variable override
func (c *Config) ApplyEnv() {
    if port := os.Getenv("SERVER_PORT"); port != "" {
        if p, err := strconv.Atoi(port); err == nil {
            c.Server.Port = p
        }
    }
}
```

---

## 5. Unified Error Handling

```go
// pkg/response/error.go
type AppError struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
    Err     error  `json:"-"`
}

func (e *AppError) Error() string {
    return e.Message
}

func (e *AppError) Unwrap() error {
    return e.Err
}

var (
    ErrNotFound      = &AppError{Code: 404, Message: "resource not found"}
    ErrUnauthorized  = &AppError{Code: 401, Message: "unauthorized"}
    ErrForbidden     = &AppError{Code: 403, Message: "forbidden"}
    ErrInternal      = &AppError{Code: 500, Message: "internal server error"}
)

// Handler unified handling
func handleError(w http.ResponseWriter, err error) {
    var appErr *AppError
    if errors.As(err, &appErr) {
        response.JSON(w, appErr.Code, appErr)
        return
    }
    response.JSON(w, 500, ErrInternal)
}
```

---

## 6. Hot Reload

```bash
# Install air
go install github.com/air-verse/air@latest

# Initialize config
air init

# Start (auto restarts on code changes)
air
```

### .air.toml

```toml
[build]
  cmd = "go build -o ./tmp/main ./cmd/server"
  bin = "./tmp/main"
  include_ext = ["go", "tpl", "tmpl", "html"]
  exclude_dir = ["tmp", "vendor"]
```

---

## 7. Logging Best Practices

```go
// Recommend using structured logging libraries (zerolog/zap)
import "github.com/rs/zerolog/log"

func main() {
    // Initialize
    log.Logger = log.Output(zerolog.ConsoleWriter{
        Out:        os.Stderr,
        TimeFormat: time.RFC3339,
    })
    
    // Usage
    log.Info().
        Str("user", "Zhang San").
        Int("age", 25).
        Msg("user logged in")
    
    log.Error().
        Err(err).
        Str("request_id", reqID).
        Msg("request failed")
}
```

Output:

```
3:00PM INF user logged in user=Zhang San age=25
3:00PM ERR request failed request_id=abc123 error="connection timeout"
```

---

## 8. Programming Habits

```go
// 1. Return early, reduce nesting
// ❌
func process(data []byte) error {
    if data != nil {
        if len(data) < 100 {
            return handle(data)
        }
        return errors.New("data too long")
    }
    return errors.New("data is empty")
}

// ✅
func process(data []byte) error {
    if data == nil {
        return errors.New("data is empty")
    }
    if len(data) >= 100 {
        return errors.New("data too long")
    }
    return handle(data)
}

// 2. Use interfaces to isolate dependencies
type UserRepo interface {
    FindByID(ctx context.Context, id string) (*User, error)
}

// 3. Avoid global state
var db *sql.DB  // ❌

// 4. Naming conventions
// Interface: er suffix / method name
// Receiver: short (1-2 chars)
func (s *Server) Handle() {}

// 5. Don't panic — handle every error
```

---

## Wrap-up

Good project structure makes code maintainable. Layered architecture (handler → service → repository) is the standard pattern for Go web projects. Dependency injection, unified error handling, structured logging, hot reload, and other tools and practices form the foundation of a high-quality Go project. Starting from the next article, we move into project practice.

---

