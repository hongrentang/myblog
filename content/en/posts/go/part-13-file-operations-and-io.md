---
title: "Go from Beginner to Pro · Part 13: File Operations and I/O"
date: 2025-01-05
weight: 13
draft: false
tags: ["go"]
---

## 1. File Reading and Writing

### Reading Entire File

```go
// ioutil.ReadFile (before Go 1.16)
// data, err := ioutil.ReadFile("file.txt")

// os.ReadFile (Go 1.16+ recommended)
data, err := os.ReadFile("file.txt")
if err != nil {
    log.Fatal(err)
}
fmt.Println(string(data))
```

### Reading Line by Line

```go
func readLines(path string) ([]string, error) {
    file, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    defer file.Close()

    var lines []string
    scanner := bufio.NewScanner(file)
    for scanner.Scan() {
        lines = append(lines, scanner.Text())
    }
    return lines, scanner.Err()
}
```

### Writing to Files

```go
// Write (overwrite)
err := os.WriteFile("output.txt", []byte("Hello, Go!"), 0644)

// Append
func appendToFile(path, text string) error {
    f, err := os.OpenFile(path, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
    if err != nil {
        return err
    }
    defer f.Close()

    _, err = f.WriteString(text + "\n")
    return err
}

// Buffered write
f, _ := os.Create("output.txt")
defer f.Close()

w := bufio.NewWriter(f)
w.WriteString("first line\n")
w.WriteString("second line\n")
w.Flush()  // Flush buffer content to the underlying writer
```

---

## 2. Reader and Writer Interfaces

### io.Reader

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

### io.Writer

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

### Standard Input/Output

```go
// os.Stdin implements Reader
// os.Stdout and os.Stderr implement Writer

var buf bytes.Buffer
io.Copy(&buf, os.Stdin)  // Read from stdin, write to buf
io.Copy(os.Stdout, &buf) // Read from buf, write to stdout
```

### io.Copy

```go
// Copy data without allocating a large buffer
func copyFile(src, dst string) error {
    sourceFile, err := os.Open(src)
    if err != nil {
        return err
    }
    defer sourceFile.Close()

    destFile, err := os.Create(dst)
    if err != nil {
        return err
    }
    defer destFile.Close()

    _, err = io.Copy(destFile, sourceFile)
    return err
}
```

### Chained Operations

```go
// Read a compressed JSON file
file, _ := os.Open("data.json.gz")
defer file.Close()

gzReader, _ := gzip.NewReader(file)
defer gzReader.Close()

var data map[string]interface{}
json.NewDecoder(gzReader).Decode(&data)
// Data flow: file → gzip.Reader → json.Decoder → map
```

---

## 3. bufio Package

```go
// bufio.Reader — buffered reading
f, _ := os.Open("large.txt")
defer f.Close()

r := bufio.NewReader(f)
for {
    line, err := r.ReadString('\n')
    if err == io.EOF {
        break
    }
    fmt.Print(line)
}

// bufio.Scanner — easier line reading
scanner := bufio.NewScanner(f)
for scanner.Scan() {
    line := scanner.Text()
    // Process line
}

// bufio.Writer — buffered writing
var buf bytes.Buffer
w := bufio.NewWriter(&buf)
w.WriteString("hello")
w.Flush()
```

---

## 4. bytes and strings Packages

### bytes.Buffer

```go
var buf bytes.Buffer

// Use it like a file
buf.WriteString("hello ")
buf.WriteString("world")
fmt.Println(buf.String())  // "hello world"

// Reset for reuse
buf.Reset()
```

### strings

```go
s := "hello, world"

// Checking
strings.Contains(s, "hello")          // true
strings.HasPrefix(s, "hello")         // true
strings.HasSuffix(s, "world")         // true

// Splitting and joining
parts := strings.Split(s, ", ")       // ["hello", "world"]
joined := strings.Join(parts, "-")    // "hello-world"

// Replacing
strings.Replace(s, "world", "Go", 1)  // "hello, Go"
strings.ReplaceAll(s, "l", "L")       // "heLLo, worLd"

// Trimming
strings.TrimSpace("  hello  ")        // "hello"
strings.Trim(s, "hdl")                // "ello, wor"

// Builder (efficient string concatenation)
var b strings.Builder
b.Grow(100)  // Pre-allocate
for i := 0; i < 1000; i++ {
    b.WriteString("a")
}
s := b.String()
```

---

## 5. Common os Package Operations

```go
// File info
info, _ := os.Stat("file.txt")
info.Name()       // File name
info.Size()       // File size
info.IsDir()      // Whether it's a directory
info.ModTime()    // Modification time
info.Mode()       // Permissions

// Directory operations
os.Mkdir("dir", 0755)           // Create single directory
os.MkdirAll("a/b/c", 0755)      // Create nested directories
os.Remove("file.txt")           // Delete file
os.RemoveAll("dir")             // Delete directory and contents

// Rename/Move
os.Rename("old.txt", "new.txt")

// Temp files
f, _ := os.CreateTemp("", "example-*.txt")
fmt.Println(f.Name())
os.Remove(f.Name())

// Environment variables
os.Getenv("HOME")           // Get
os.Setenv("MY_VAR", "val")  // Set
os.LookupEnv("MY_VAR")      // Safe get (with existence check)
os.Environ()                // All environment variables

// Working directory
dir, _ := os.Getwd()
os.Chdir("/tmp")

// Exit
os.Exit(1)     // Immediate exit
os.Exit(0)     // Normal exit
```

---

## 6. path/filepath

```go
import "path/filepath"

// Path operations
path := filepath.Join("dir", "sub", "file.txt")  // "dir/sub/file.txt"
dir := filepath.Dir(path)        // "dir/sub"
base := filepath.Base(path)      // "file.txt"
ext := filepath.Ext(path)        // ".txt"

// Directory traversal
filepath.WalkDir("root", func(path string, d fs.DirEntry, err error) error {
    if err != nil {
        return err
    }
    fmt.Println(path, d.IsDir())
    return nil
})

// Pattern matching
matched, _ := filepath.Match("*.txt", "file.txt")  // true
```

---

## 7. Logging

### log Package

```go
import "log"

// Basic usage
log.Println("this is a log")
log.Printf("user %s logged in successfully", "admin")

// Fatal errors
log.Fatal("unable to start server")    // Print log and os.Exit(1)
log.Fatalf("error: %v", err)

// Conditional panic
log.Panic("unrecoverable error")    // Print log and panic
```

### Custom Logger

```go
// Custom log format
logger := log.New(os.Stdout, "[APP] ", log.Ldate|log.Ltime|log.Lshortfile)
logger.Println("custom log")

// Log to file
f, _ := os.OpenFile("app.log", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
logger := log.New(f, "[INFO] ", log.LstdFlags)
logger.Println("log written to file")
```

### Structured Logging

```go
// Simple structured logging
type Logger struct {
    *log.Logger
}

func (l *Logger) Info(msg string, fields ...interface{}) {
    l.Printf("[INFO] %s %v", msg, fields)
}

// Third-party log libraries like zerolog / zap are recommended
```

---

## 8. Practical: File Watching

```go
func watchFile(path string) {
    initialStat, _ := os.Stat(path)

    for {
        stat, err := os.Stat(path)
        if err != nil {
            fmt.Println("file deleted")
            return
        }

        if stat.ModTime() != initialStat.ModTime() {
            fmt.Println("file modified")
            initialStat = stat
        }

        time.Sleep(1 * time.Second)
    }
}
```

---

## Wrap-up

I/O is the interface between a program and the outside world. Go's io.Reader/Writer interfaces provide a unified abstraction, and together with the bufio, bytes, strings, and os packages, they cover the vast majority of file operation scenarios. Remember: the interfaces are simple but the combinations are powerful.

---

