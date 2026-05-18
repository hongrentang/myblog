---
title: "Go from Beginner to Pro · Part 15: JSON and Data Serialization"
date: 2025-01-07
weight: 1215
draft: false
tags: ["go"]
---

## 1. JSON Serialization

### Struct to JSON

```go
type User struct {
    ID        int       `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    Password  string    `json:"-"`                 // Ignore
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at,omitempty"` // Omit if empty
}

user := User{
    ID:        1,
    Name:      "Zhang San",
    Email:     "zhangsan@example.com",
    Password:  "secret123",
    CreatedAt: time.Now(),
}

// Serialize
data, err := json.Marshal(user)
// data, err := json.MarshalIndent(user, "", "  ")  // Pretty print

fmt.Println(string(data))
// {"id":1,"name":"Zhang San","email":"zhangsan@example.com","created_at":"2025-01-15T10:00:00Z"}
```

### JSON to Struct

```go
jsonStr := `{
    "id": 1,
    "name": "Zhang San",
    "email": "zhangsan@example.com"
}`

var user User
err := json.Unmarshal([]byte(jsonStr), &user)
if err != nil {
    log.Fatal(err)
}
fmt.Printf("%+v\n", user)
```

### Streaming Read/Write

```go
// Read JSON from file (suitable for large files)
file, _ := os.Open("users.json")
defer file.Close()

var users []User
decoder := json.NewDecoder(file)
if err := decoder.Decode(&users); err != nil {
    log.Fatal(err)
}

// Write JSON to file
file, _ = os.Create("output.json")
defer file.Close()

encoder := json.NewEncoder(file)
encoder.SetIndent("", "  ")  // Format
for _, u := range users {
    if err := encoder.Encode(u); err != nil {
        log.Fatal(err)
    }
}
```

### Raw Message

```go
// json.RawMessage — lazy parsing
type Response struct {
    Code    int              `json:"code"`
    Data    json.RawMessage  `json:"data"`  // Preserve raw data first
}

// Parse later based on conditions
resp := Response{Code: 200, Data: json.RawMessage(`{"name":"Zhang San"}`)}
var user User
json.Unmarshal(resp.Data, &user)
```

---

## 2. Custom JSON Handling

### Custom Serialization

```go
type Duration struct {
    time.Duration
}

func (d Duration) MarshalJSON() ([]byte, error) {
    return json.Marshal(d.String())  // "1s", "5m0s"
}

func (d *Duration) UnmarshalJSON(data []byte) error {
    var s string
    if err := json.Unmarshal(data, &s); err != nil {
        return err
    }
    dur, err := time.ParseDuration(s)
    if err != nil {
        return err
    }
    d.Duration = dur
    return nil
}

type Config struct {
    Timeout Duration `json:"timeout"`
}

// JSON: {"timeout": "30s"}
```

### Dynamic JSON

```go
// map to JSON
data := map[string]interface{}{
    "name": "Zhang San",
    "age":  25,
    "tags": []string{"go", "rust"},
}
jsonData, _ := json.Marshal(data)

// Parse dynamic JSON
var result map[string]interface{}
json.Unmarshal(jsonData, &result)
name := result["name"].(string)
age := result["age"].(float64)  // JSON numbers are parsed as float64 by default

// Type assertion
for k, v := range result {
    switch val := v.(type) {
    case string:
        fmt.Printf("%s: string=%s\n", k, val)
    case float64:
        fmt.Printf("%s: number=%f\n", k, val)
    case []interface{}:
        fmt.Printf("%s: array=%v\n", k, val)
    }
}
```

---

## 3. JSON Encoding Control

```go
// Time format
type CustomTime struct {
    time.Time
}

func (ct CustomTime) MarshalJSON() ([]byte, error) {
    return json.Marshal(ct.Format("2006-01-02 15:04:05"))
}

func (ct *CustomTime) UnmarshalJSON(data []byte) error {
    var s string
    if err := json.Unmarshal(data, &s); err != nil {
        return err
    }
    t, err := time.Parse("2006-01-02 15:04:05", s)
    if err != nil {
        return err
    }
    ct.Time = t
    return nil
}

// Number format — preserve leading zeros
type ZipCode string

func (z ZipCode) MarshalJSON() ([]byte, error) {
    return json.Marshal(string(z))
}

func (z *ZipCode) UnmarshalJSON(data []byte) error {
    var s string
    if err := json.Unmarshal(data, &s); err != nil {
        return err
    }
    *z = ZipCode(s)
    return nil
}
```

---

## 4. Other Serialization Formats

### XML

```go
type User struct {
    XMLName xml.Name `xml:"user"`
    ID      int      `xml:"id,attr"`
    Name    string   `xml:"name"`
    Email   string   `xml:"email"`
}

user := User{ID: 1, Name: "Zhang San", Email: "test@test.com"}

// Serialize
data, _ := xml.MarshalIndent(user, "", "  ")
// <user id="1">
//   <name>Zhang San</name>
//   <email>test@test.com</email>
// </user>

// Deserialize
var u User
xml.Unmarshal(data, &u)
```

### YAML

```go
// Requires third-party library: go get gopkg.in/yaml.v3
import "gopkg.in/yaml.v3"

config := `
server:
  host: localhost
  port: 8080
database:
  host: localhost
  port: 3306
`

var cfg struct {
    Server   struct {
        Host string `yaml:"host"`
        Port int    `yaml:"port"`
    } `yaml:"server"`
    Database struct {
        Host string `yaml:"host"`
        Port int    `yaml:"port"`
    } `yaml:"database"`
}

yaml.Unmarshal([]byte(config), &cfg)
```

### Protocol Buffers

```go
// 1. Define proto file
// syntax = "proto3";
// message User {
//   int32 id = 1;
//   string name = 2;
//   string email = 3;
// }

// 2. Generate Go code
// protoc --go_out=. user.proto

// 3. Usage
// user := &pb.User{
//     Id:    1,
//     Name:  "Zhang San",
//     Email: "test@test.com",
// }
// data, _ := proto.Marshal(user)
// 
// var newUser pb.User
// proto.Unmarshal(data, &newUser)
```

---

## 5. Gob Encoding

Go's native binary format, suitable for data transfer between Go programs:

```go
var buf bytes.Buffer

// Encode
encoder := gob.NewEncoder(&buf)
encoder.Encode(User{ID: 1, Name: "Zhang San"})

// Decode
var user User
decoder := gob.NewDecoder(&buf)
decoder.Decode(&user)
```

---

## 6. Performance Comparison

```go
// JSON (most versatile)
encoding/json  // Standard library, uses reflection, average speed
encoding/json.NewEncoder // Streaming

// High-performance JSON libraries
github.com/json-iterator/go   // Faster, compatible with standard library
github.com/bytedance/sonic     // Extremely fast (JIT-based)

// Binary (suitable for internal communication)
encoding/gob       // Go native
encoding/protobuf  // Cross-language
```

---

## Wrap-up

JSON is the most commonly used data format in web development. Go's encoding/json provides Marshal/Unmarshal and streaming read/write, with struct tags controlling serialization behavior. For performance-sensitive scenarios, consider third-party libraries like sonic/json-iterator.

---

