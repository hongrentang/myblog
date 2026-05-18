---
title: "Go from Beginner to Pro · Part 20: Project 1 — Command Line Tool"
date: 2025-01-13
weight: 1220
draft: false
tags: ["go"]
---

## Project Introduction

Implement an in-memory key-value cache tool with support for data expiration, persistence to file, and command-line interaction. A simplified version of Redis.

## Project Structure

```
gocache/
├── cmd/
│   └── gocache/
│       └── main.go        # Entry
├── internal/
│   ├── cache/
│   │   └── cache.go       # Cache core
│   ├── server/
│   │   └── server.go      # TCP server
│   └── persistence/
│       └── dump.go        # Persistence
└── go.mod
```

## Step 1: Cache Core

```go
// internal/cache/cache.go
package cache

import (
    "sync"
    "time"
)

type Item struct {
    Value      interface{}
    Expiration int64 // Expiration timestamp (nanoseconds)
}

func (i Item) IsExpired() bool {
    if i.Expiration == 0 {
        return false
    }
    return time.Now().UnixNano() > i.Expiration
}

type Cache struct {
    mu    sync.RWMutex
    items map[string]Item
}

func New() *Cache {
    c := &Cache{
        items: make(map[string]Item),
    }
    // Start background cleanup
    go c.cleanup()
    return c
}

func (c *Cache) Set(key string, value interface{}, duration time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()

    var exp int64
    if duration > 0 {
        exp = time.Now().Add(duration).UnixNano()
    }

    c.items[key] = Item{
        Value:      value,
        Expiration: exp,
    }
}

func (c *Cache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    item, ok := c.items[key]
    c.mu.RUnlock()

    if !ok {
        return nil, false
    }

    if item.IsExpired() {
        c.Delete(key)
        return nil, false
    }

    return item.Value, true
}

func (c *Cache) Delete(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    delete(c.items, key)
}

func (c *Cache) Keys() []string {
    c.mu.RLock()
    defer c.mu.RUnlock()

    keys := make([]string, 0, len(c.items))
    for k := range c.items {
        keys = append(keys, k)
    }
    return keys
}

func (c *Cache) Len() int {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return len(c.items)
}

func (c *Cache) Flush() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items = make(map[string]Item)
}

// Background cleanup of expired keys
func (c *Cache) cleanup() {
    ticker := time.NewTicker(1 * time.Minute)
    defer ticker.Stop()

    for range ticker.C {
        c.mu.Lock()
        for k, v := range c.items {
            if v.IsExpired() {
                delete(c.items, k)
            }
        }
        c.mu.Unlock()
    }
}

// Get all data (for persistence)
func (c *Cache) GetAll() map[string]Item {
    c.mu.RLock()
    defer c.mu.RUnlock()

    items := make(map[string]Item, len(c.items))
    for k, v := range c.items {
        items[k] = v
    }
    return items
}

// Reset (for loading persisted data)
func (c *Cache) Reset(items map[string]Item) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items = items
}
```

## Step 2: Command Interface

```go
// Supported commands:
// SET key value [ttl]   — Set key-value pair
// GET key                — Get value
// DEL key                — Delete key
// KEYS                   — List all keys
// LEN                    — Get total count
// FLUSH                  — Clear all
// EXIT                   — Exit
```

## Step 3: Persistence

```go
// internal/persistence/dump.go
package persistence

import (
    "encoding/gob"
    "os"
    "path/filepath"
)

type Dumper struct {
    filePath string
}

func New(path string) *Dumper {
    dir := filepath.Dir(path)
    os.MkdirAll(dir, 0755)
    return &Dumper{filePath: path}
}

func (d *Dumper) Save(data interface{}) error {
    file, err := os.Create(d.filePath)
    if err != nil {
        return err
    }
    defer file.Close()

    encoder := gob.NewEncoder(file)
    return encoder.Encode(data)
}

func (d *Dumper) Load(data interface{}) error {
    file, err := os.Open(d.filePath)
    if err != nil {
        if os.IsNotExist(err) {
            return nil // First run, no backup file
        }
        return err
    }
    defer file.Close()

    decoder := gob.NewDecoder(file)
    return decoder.Decode(data)
}
```

## Step 4: Command-line REPL

```go
// cmd/gocache/main.go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strconv"
    "strings"
    "time"

    "gocache/internal/cache"
    "gocache/internal/persistence"
)

const dumpFile = "data/cache.dump"

func main() {
    c := cache.New()
    dumper := persistence.New(dumpFile)

    // Load persisted data
    items := make(map[string]cache.Item)
    if err := dumper.Load(&items); err == nil {
        c.Reset(items)
        fmt.Printf("Loaded %d cached entries\n", len(items))
    }

    // Periodic persistence
    go func() {
        ticker := time.NewTicker(30 * time.Second)
        defer ticker.Stop()
        for range ticker.C {
            dumper.Save(c.GetAll())
        }
    }()

    fmt.Println("GoCache v1.0 — Enter commands (SET/GET/DEL/KEYS/LEN/FLUSH/EXIT)")
    scanner := bufio.NewScanner(os.Stdin)

    for scanner.Scan() {
        line := strings.TrimSpace(scanner.Text())
        if line == "" {
            continue
        }

        args := strings.Fields(line)
        cmd := strings.ToUpper(args[0])

        switch cmd {
        case "SET":
            if len(args) < 3 {
                fmt.Println("Usage: SET key value [ttl_seconds]")
                continue
            }
            key := args[1]
            value := strings.Join(args[2:], " ")
            var ttl time.Duration
            // Check if last argument is numeric (TTL)
            if last := args[len(args)-1]; isNumeric(last) && len(args) > 3 {
                sec, _ := strconv.Atoi(last)
                ttl = time.Duration(sec) * time.Second
                value = strings.Join(args[2:len(args)-1], " ")
            }
            c.Set(key, value, ttl)
            fmt.Println("OK")

        case "GET":
            if len(args) < 2 {
                fmt.Println("Usage: GET key")
                continue
            }
            val, ok := c.Get(args[1])
            if !ok {
                fmt.Println("(nil)")
            } else {
                fmt.Println(val)
            }

        case "DEL":
            if len(args) < 2 {
                fmt.Println("Usage: DEL key")
                continue
            }
            c.Delete(args[1])
            fmt.Println("OK")

        case "KEYS":
            keys := c.Keys()
            if len(keys) == 0 {
                fmt.Println("(empty)")
            } else {
                for _, k := range keys {
                    fmt.Println(k)
                }
            }

        case "LEN":
            fmt.Println(c.Len())

        case "FLUSH":
            c.Flush()
            fmt.Println("OK")

        case "EXIT", "QUIT":
            fmt.Println("Saving data...")
            dumper.Save(c.GetAll())
            fmt.Println("Goodbye!")
            return

        default:
            fmt.Printf("Unknown command: %s\n", cmd)
        }
    }

    // Save before exit
    dumper.Save(c.GetAll())
}

func isNumeric(s string) bool {
    _, err := strconv.Atoi(s)
    return err == nil
}
```

## Running

```bash
# Initialize
go mod init gocache

# Run
go run cmd/gocache/main.go

# Interaction example
# GoCache v1.0 — Enter commands (SET/GET/DEL/KEYS/LEN/FLUSH/EXIT)
# SET name Zhang San 30
# OK
# GET name
# Zhang San
# KEYS
# name
# LEN
# 1
# EXIT
```

## Extension Ideas

- Add TCP server (support remote access, similar to Redis)
- Add more data structures (List/Set/Sorted Set)
- Add AOF persistence

---

