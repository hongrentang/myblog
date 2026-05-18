---
title: "Go from Beginner to Pro · Part 21: Project 2 — Concurrent Crawler"
date: 2025-01-14
weight: 1221
draft: false
tags: ["go"]
---

## Project Introduction

Build a concurrent crawler system: multi-worker concurrent fetching, data pipeline processing, and result storage. This project comprehensively utilizes goroutines, channels, and the sync package.

## Project Structure

```
crawler/
├── cmd/crawler/main.go
├── internal/
│   ├── fetcher/
│   │   └── fetcher.go    # HTTP fetching
│   ├── parser/
│   │   └── parser.go     # Page parsing
│   ├── pipeline/
│   │   └── pipeline.go   # Data pipeline
│   └── storage/
│       └── storage.go    # Result storage
└── go.mod
```

## Step 1: Data Model

```go
// Data structure used throughout the crawler
type CrawlResult struct {
    URL     string
    Title   string
    Content string
    Links   []string      // Page links
    Depth   int
    Status  int           // HTTP status code
    Error   string
}
```

## Step 2: Page Fetcher

```go
// internal/fetcher/fetcher.go
package fetcher

import (
    "fmt"
    "io"
    "net/http"
    "time"
)

type Fetcher struct {
    client *http.Client
}

func New(timeout time.Duration) *Fetcher {
    return &Fetcher{
        client: &http.Client{
            Timeout: timeout,
            Transport: &http.Transport{
                MaxIdleConns:    100,
                IdleConnTimeout: 30 * time.Second,
            },
        },
    }
}

type FetchResult struct {
    URL      string
    Body     string
    Status   int
    Duration time.Duration
    Error    error
}

func (f *Fetcher) Fetch(url string) FetchResult {
    start := time.Now()
    
    req, err := http.NewRequest("GET", url, nil)
    if err != nil {
        return FetchResult{URL: url, Error: err, Duration: time.Since(start)}
    }
    
    req.Header.Set("User-Agent", "GoCrawler/1.0")
    
    resp, err := f.client.Do(req)
    if err != nil {
        return FetchResult{URL: url, Error: err, Duration: time.Since(start)}
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(resp.Body)
    if err != nil {
        return FetchResult{URL: url, Error: err, Status: resp.StatusCode, Duration: time.Since(start)}
    }

    return FetchResult{
        URL:      url,
        Body:     string(body),
        Status:   resp.StatusCode,
        Duration: time.Since(start),
    }
}
```

## Step 3: Page Parser

```go
// internal/parser/parser.go
package parser

import (
    "regexp"
    "strings"
)

var (
    titleRe   = regexp.MustCompile(`<title[^>]*>([^<]+)</title>`)
    linkRe    = regexp.MustCompile(`href=["'](https?://[^"']+)["']`)
)

type ParseResult struct {
    URL     string
    Title   string
    Content string
    Links   []string
    TextLen int
}

func Parse(url, body string) ParseResult {
    result := ParseResult{URL: url}

    // Extract title
    if matches := titleRe.FindStringSubmatch(body); len(matches) > 1 {
        result.Title = strings.TrimSpace(matches[1])
    }

    // Extract links
    links := linkRe.FindAllStringSubmatch(body, -1)
    for _, m := range links {
        result.Links = append(result.Links, m[1])
    }

    // Extract text content
    text := stripTags(body)
    result.Content = truncate(text, 500)
    result.TextLen = len(text)

    return result
}

func stripTags(html string) string {
    re := regexp.MustCompile(`<[^>]*>`)
    text := re.ReplaceAllString(html, " ")
    
    re = regexp.MustCompile(`\s+`)
    text = re.ReplaceAllString(text, " ")
    
    return strings.TrimSpace(text)
}

func truncate(s string, n int) string {
    runes := []rune(s)
    if len(runes) <= n {
        return s
    }
    return string(runes[:n]) + "..."
}
```

## Step 4: Data Pipeline

```go
// internal/pipeline/pipeline.go
package pipeline

import (
    "context"
    "sync"
)

type Processor func(input interface{}) (interface{}, error)

type Pipeline struct {
    stages []Processor
}

func New(stages ...Processor) *Pipeline {
    return &Pipeline{stages: stages}
}

// Stage creates a processing step
func Stage[T, U any](fn func(T) U) Processor {
    return func(input interface{}) (interface{}, error) {
        return fn(input.(T)), nil
    }
}

// Run executes all stages in goroutines
func (p *Pipeline) Run(ctx context.Context, input <-chan interface{}, workers int) <-chan interface{} {
    out := input
    for _, stage := range p.stages {
        out = p.runStage(ctx, out, stage, workers)
    }
    return out
}

func (p *Pipeline) runStage(ctx context.Context, in <-chan interface{}, stage Processor, workers int) <-chan interface{} {
    out := make(chan interface{}, 100)
    
    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for data := range in {
                select {
                case <-ctx.Done():
                    return
                default:
                    result, err := stage(data)
                    if err != nil {
                        continue
                    }
                    out <- result
                }
            }
        }()
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

## Step 5: Result Storage

```go
// internal/storage/storage.go
package storage

import (
    "encoding/json"
    "os"
    "sync"
)

type Storage struct {
    mu       sync.RWMutex
    results  []interface{}
    filePath string
}

func New(filePath string) *Storage {
    return &Storage{
        results:  make([]interface{}, 0),
        filePath: filePath,
    }
}

func (s *Storage) Save(result interface{}) error {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.results = append(s.results, result)
    return nil
}

func (s *Storage) Flush() error {
    s.mu.RLock()
    data, err := json.MarshalIndent(s.results, "", "  ")
    s.mu.RUnlock()
    if err != nil {
        return err
    }
    return os.WriteFile(s.filePath, data, 0644)
}

func (s *Storage) Count() int {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return len(s.results)
}
```

## Step 6: Assembling the Crawler

```go
// cmd/crawler/main.go
package main

import (
    "context"
    "flag"
    "fmt"
    "log"
    "sync"
    "time"

    "crawler/internal/fetcher"
    "crawler/internal/parser"
    "crawler/internal/storage"
)

var (
    startURL = flag.String("url", "https://example.com", "Starting URL")
    maxDepth = flag.Int("depth", 2, "Maximum crawl depth")
    workers  = flag.Int("workers", 5, "Number of concurrent workers")
    output   = flag.String("output", "results.json", "Output file")
)

type CrawlTask struct {
    URL   string
    Depth int
}

func main() {
    flag.Parse()

    fetcher := fetcher.New(10 * time.Second)
    store := storage.New(*output)

    // Task channel
    tasks := make(chan CrawlTask, 1000)
    visited := sync.Map{}  // Deduplication

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    var wg sync.WaitGroup

    // Start workers
    for i := 0; i < *workers; i++ {
        wg.Add(1)
        go worker(ctx, &wg, tasks, fetcher, store, &visited)
    }

    // Send starting task
    tasks <- CrawlTask{URL: *startURL, Depth: 0}

    // Start progress monitoring
    go func() {
        ticker := time.NewTicker(5 * time.Second)
        defer ticker.Stop()
        for range ticker.C {
            log.Printf("Fetched: %d, In queue: %d\n", store.Count(), len(tasks))
        }
    }()

    // Wait for all workers to finish
    wg.Wait()
    close(tasks)

    // Flush storage
    if err := store.Flush(); err != nil {
        log.Fatal("Save failed:", err)
    }

    fmt.Printf("Done! Fetched %d pages, results saved to %s\n", store.Count(), *output)
}

func worker(ctx context.Context, wg *sync.WaitGroup, tasks <-chan CrawlTask,
    f *fetcher.Fetcher, store *storage.Storage, visited *sync.Map) {
    defer wg.Done()

    for task := range tasks {
        select {
        case <-ctx.Done():
            return
        default:
        }

        // Deduplication
        if _, loaded := visited.LoadOrStore(task.URL, true); loaded {
            continue
        }

        // Fetch
        result := f.Fetch(task.URL)
        if result.Error != nil {
            log.Printf("Fetch failed: %s - %v\n", task.URL, result.Error)
            continue
        }

        // Parse
        parsed := parser.Parse(task.URL, result.Body)
        log.Printf("[%d] %s (%s)\n", result.Status, task.URL, parsed.Title)

        // Save
        store.Save(map[string]interface{}{
            "url":    task.URL,
            "title":  parsed.Title,
            "links":  len(parsed.Links),
            "depth":  task.Depth,
            "status": result.Status,
        })

        // Add new crawl tasks (depth limited)
        if task.Depth < *maxDepth {
            for _, link := range parsed.Links {
                tasks <- CrawlTask{URL: link, Depth: task.Depth + 1}
            }
        }
    }
}
```

## Running

```bash
go run cmd/crawler/main.go -url "https://golang.org" -depth 2 -workers 10
```

## Extension Ideas

- Add Robots.txt protocol compliance
- Add request rate limiting (time.Ticker for throttling)
- Smarter URL deduplication (Bloom Filter)
- Support persistent queue (resume after interruption)
- Store to Elasticsearch for full-text search

---

