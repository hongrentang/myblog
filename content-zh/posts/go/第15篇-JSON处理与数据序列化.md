---
title: "Go 从入门到精通 · 第15篇：JSON 处理与数据序列化"
date: 2025-01-07
weight: 15
draft: false
tags: ["go"]
---

## 一、JSON 序列化

### 结构体转 JSON

```go
type User struct {
    ID        int       `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    Password  string    `json:"-"`                 // 忽略
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at,omitempty"` // 为空时忽略
}

user := User{
    ID:        1,
    Name:      "张三",
    Email:     "zhangsan@example.com",
    Password:  "secret123",
    CreatedAt: time.Now(),
}

// 序列化
data, err := json.Marshal(user)
// data, err := json.MarshalIndent(user, "", "  ")  // 格式化输出

fmt.Println(string(data))
// {"id":1,"name":"张三","email":"zhangsan@example.com","created_at":"2025-01-15T10:00:00Z"}
```

### JSON 转结构体

```go
jsonStr := `{
    "id": 1,
    "name": "张三",
    "email": "zhangsan@example.com"
}`

var user User
err := json.Unmarshal([]byte(jsonStr), &user)
if err != nil {
    log.Fatal(err)
}
fmt.Printf("%+v\n", user)
```

### 流式读写

```go
// 从文件读取 JSON（适合大文件）
file, _ := os.Open("users.json")
defer file.Close()

var users []User
decoder := json.NewDecoder(file)
if err := decoder.Decode(&users); err != nil {
    log.Fatal(err)
}

// 写入 JSON 到文件
file, _ = os.Create("output.json")
defer file.Close()

encoder := json.NewEncoder(file)
encoder.SetIndent("", "  ")  // 格式化
for _, u := range users {
    if err := encoder.Encode(u); err != nil {
        log.Fatal(err)
    }
}
```

### 原始消息

```go
// json.RawMessage——延迟解析
type Response struct {
    Code    int              `json:"code"`
    Data    json.RawMessage  `json:"data"`  // 先保留原始数据
}

// 后续再根据条件解析
resp := Response{Code: 200, Data: json.RawMessage(`{"name":"张三"}`)}
var user User
json.Unmarshal(resp.Data, &user)
```

---

## 二、自定义 JSON 处理

### 自定义序列化

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

### 动态 JSON

```go
// map 转 JSON
data := map[string]interface{}{
    "name": "张三",
    "age":  25,
    "tags": []string{"go", "rust"},
}
jsonData, _ := json.Marshal(data)

// 解析动态 JSON
var result map[string]interface{}
json.Unmarshal(jsonData, &result)
name := result["name"].(string)
age := result["age"].(float64)  // JSON 数字默认解析为 float64

// 类型断言
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

## 三、JSON 编码控制

```go
// 时间格式
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

// 数字格式——保留前导零
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

## 四、其他序列化格式

### XML

```go
type User struct {
    XMLName xml.Name `xml:"user"`
    ID      int      `xml:"id,attr"`
    Name    string   `xml:"name"`
    Email   string   `xml:"email"`
}

user := User{ID: 1, Name: "张三", Email: "test@test.com"}

// 序列化
data, _ := xml.MarshalIndent(user, "", "  ")
// <user id="1">
//   <name>张三</name>
//   <email>test@test.com</email>
// </user>

// 反序列化
var u User
xml.Unmarshal(data, &u)
```

### YAML

```go
// 需要第三方库：go get gopkg.in/yaml.v3
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
// 1. 定义 proto 文件
// syntax = "proto3";
// message User {
//   int32 id = 1;
//   string name = 2;
//   string email = 3;
// }

// 2. 生成 Go 代码
// protoc --go_out=. user.proto

// 3. 使用
// user := &pb.User{
//     Id:    1,
//     Name:  "张三",
//     Email: "test@test.com",
// }
// data, _ := proto.Marshal(user)
// 
// var newUser pb.User
// proto.Unmarshal(data, &newUser)
```

---

## 五、Gob 编码

Go 原生二进制格式，适合 Go 程序之间的数据传输：

```go
var buf bytes.Buffer

// 编码
encoder := gob.NewEncoder(&buf)
encoder.Encode(User{ID: 1, Name: "张三"})

// 解码
var user User
decoder := gob.NewDecoder(&buf)
decoder.Decode(&user)
```

---

## 六、性能对比

```go
// JSON（最通用）
encoding/json  // 标准库，反射，速度一般
encoding/json.NewEncoder // 流式

// 高性能 JSON 库
github.com/json-iterator/go   // 更快，兼容标准库
github.com/bytedance/sonic     // 极快（基于 JIT）

// 二进制（适合内部通信）
encoding/gob       // Go 原生
encoding/protobuf  // 跨语言
```

---

## 小结

JSON 是 Web 开发中最常用的数据格式。Go 的 encoding/json 提供 Marshal/Unmarshal 和流式读写，配合 struct tag 控制序列化行为。对于性能敏感场景，可以选用 sonic/json-iterator 等第三方库。

---

