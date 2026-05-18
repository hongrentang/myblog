---
title: "Go 从入门到精通 · 第24篇：项目五 —— WebSocket 实时聊天室"
date: 2025-01-17
weight: 24
draft: false
tags: ["go"]
---

## 项目介绍

构建一个基于 WebSocket 的实时聊天室，支持多房间、消息广播、在线用户列表。使用 gorilla/websocket 实现 WebSocket 通信。

## 项目结构

```
chatroom/
├── cmd/server/main.go
├── internal/
│   ├── hub/
│   │   └── hub.go        # 中央调度器
│   ├── client/
│   │   └── client.go     # WebSocket 客户端
│   └── room/
│       └── room.go       # 聊天室
├── web/
│   └── index.html        # 前端页面
├── go.mod
└── go.sum
```

## 第一步：数据模型

```go
// 消息结构
type Message struct {
    Type     string      `json:"type"`     // join/leave/message/error
    Room     string      `json:"room"`     // 房间名
    Username string      `json:"username"` // 用户名
    Content  string      `json:"content"`  // 消息内容
    Users    []string    `json:"users,omitempty"` // 在线用户列表
    Time     string      `json:"time"`     // 时间戳
}
```

## 第二步：聊天室管理

```go
// internal/room/room.go
package room

import (
    "sync"
)

type Room struct {
    Name    string
    clients map[*Client]bool
    mu      sync.RWMutex
}

func NewRoom(name string) *Room {
    return &Room{
        Name:    name,
        clients: make(map[*Client]bool),
    }
}

func (r *Room) Add(client *Client) {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.clients[client] = true
}

func (r *Room) Remove(client *Client) {
    r.mu.Lock()
    defer r.mu.Unlock()
    delete(r.clients, client)
}

func (r *Room) Broadcast(msg []byte, sender *Client) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    for client := range r.clients {
        if client != sender {
            client.Send(msg)
        }
    }
}

func (r *Room) Users() []string {
    r.mu.RLock()
    defer r.mu.RUnlock()

    var users []string
    for client := range r.clients {
        users = append(users, client.Username)
    }
    return users
}

func (r *Room) Count() int {
    r.mu.RLock()
    defer r.mu.RUnlock()
    return len(r.clients)
}
```

## 第三步：WebSocket 客户端

```go
// internal/client/client.go
package client

import (
    "log"
    "time"

    "github.com/gorilla/websocket"
)

const (
    writeWait      = 10 * time.Second
    pongWait       = 60 * time.Second
    pingPeriod     = (pongWait * 9) / 10
    maxMessageSize = 4096
)

type Client struct {
    Hub      *Hub
    Room     *Room
    Conn     *websocket.Conn
    Username string
    Send     chan []byte
    done     chan struct{}
}

func NewClient(hub *Hub, room *Room, conn *websocket.Conn, username string) *Client {
    return &Client{
        Hub:      hub,
        Room:     room,
        Conn:     conn,
        Username: username,
        Send:     make(chan []byte, 256),
        done:     make(chan struct{}),
    }
}

func (c *Client) ReadPump() {
    defer func() {
        c.Hub.Unregister <- c
        c.Conn.Close()
    }()

    c.Conn.SetReadLimit(maxMessageSize)
    c.Conn.SetReadDeadline(time.Now().Add(pongWait))
    c.Conn.SetPongHandler(func(string) error {
        c.Conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })

    for {
        _, msg, err := c.Conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseNormalClosure) {
                log.Printf("读取错误: %v", err)
            }
            break
        }
        c.Hub.Broadcast <- &HubMessage{
            Client:  c,
            Message: msg,
        }
    }
}

func (c *Client) WritePump() {
    ticker := time.NewTicker(pingPeriod)
    defer func() {
        ticker.Stop()
        c.Conn.Close()
    }()

    for {
        select {
        case msg, ok := <-c.Send:
            if !ok {
                c.Conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }
            c.Conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := c.Conn.WriteMessage(websocket.TextMessage, msg); err != nil {
                return
            }

        case <-ticker.C:
            c.Conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := c.Conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}

func (c *Client) Send(msg []byte) {
    select {
    case c.Send <- msg:
    default:
        // 发送缓冲区满，断开
        close(c.done)
    }
}
```

## 第四步：Hub 中央调度

```go
// internal/hub/hub.go
package hub

import (
    "encoding/json"
    "fmt"
    "log"
    "time"
)

type Message struct {
    Type     string   `json:"type"`
    Room     string   `json:"room"`
    Username string   `json:"username"`
    Content  string   `json:"content"`
    Users    []string `json:"users,omitempty"`
    Time     string   `json:"time"`
}

type Client struct {
    Hub      *Hub
    Room     *Room
    Conn     *websocket.Conn
    Username string
    Send     chan []byte
}

type HubMessage struct {
    Client  *Client
    Message []byte
}

type Hub struct {
    Rooms      map[string]*Room
    Register   chan *Client
    Unregister chan *Client
    Broadcast  chan *HubMessage
}

func New() *Hub {
    return &Hub{
        Rooms:      make(map[string]*Room),
        Register:   make(chan *Client),
        Unregister: make(chan *Client),
        Broadcast:  make(chan *HubMessage, 256),
    }
}

func (h *Hub) Run() {
    for {
        select {
        case client := <-h.Register:
            room := h.getOrCreateRoom(client.Room.Name)
            room.Add(client)
            client.Room = room

            // 广播用户加入
            h.broadcastRoomMessage(room, Message{
                Type:     "join",
                Room:     room.Name,
                Username: client.Username,
                Content:  fmt.Sprintf("%s 加入了聊天室", client.Username),
                Time:     time.Now().Format("15:04:05"),
            })

            // 更新用户列表
            h.broadcastRoomMessage(room, Message{
                Type: "users",
                Room: room.Name,
                Users: room.Users(),
            })

            log.Printf("%s 加入了房间 %s (%d 人在线)", client.Username, room.Name, room.Count())

        case client := <-h.Unregister:
            room := client.Room
            if room != nil {
                room.Remove(client)
                close(client.Send)

                // 广播用户离开
                h.broadcastRoomMessage(room, Message{
                    Type:     "leave",
                    Room:     room.Name,
                    Username: client.Username,
                    Content:  fmt.Sprintf("%s 离开了聊天室", client.Username),
                    Time:     time.Now().Format("15:04:05"),
                })

                // 更新用户列表
                h.broadcastRoomMessage(room, Message{
                    Type: "users",
                    Room: room.Name,
                    Users: room.Users(),
                })

                log.Printf("%s 离开了房间 %s (%d 人在线)", client.Username, room.Name, room.Count())
            }

        case broadcast := <-h.Broadcast:
            var msg Message
            if err := json.Unmarshal(broadcast.Message, &msg); err != nil {
                continue
            }

            msg.Time = time.Now().Format("15:04:05")
            msg.Username = broadcast.Client.Username
            msg.Room = broadcast.Client.Room.Name

            data, _ := json.Marshal(msg)
            broadcast.Client.Room.Broadcast(data, broadcast.Client)

            log.Printf("[%s] %s: %s", msg.Room, msg.Username, msg.Content)
        }
    }
}

func (h *Hub) getOrCreateRoom(name string) *Room {
    if room, ok := h.Rooms[name]; ok {
        return room
    }
    room := NewRoom(name)
    h.Rooms[name] = room
    return room
}

func (h *Hub) broadcastRoomMessage(room *Room, msg Message) {
    data, err := json.Marshal(msg)
    if err != nil {
        return
    }
    room.Broadcast(data, nil)
}
```

## 第五步：HTTP 与 WebSocket 入口

```go
// cmd/server/main.go
package main

import (
    "encoding/json"
    "log"
    "net/http"

    "github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    CheckOrigin: func(r *http.Request) bool {
        return true // 开发环境允许所有来源
    },
}

type JoinRequest struct {
    Room     string `json:"room"`
    Username string `json:"username"`
}

func main() {
    hub := New()
    go hub.Run()

    http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
        conn, err := upgrader.Upgrade(w, r, nil)
        if err != nil {
            log.Println("WebSocket 升级失败:", err)
            return
        }

        // 读取第一条消息（加入房间信息）
        _, msg, err := conn.ReadMessage()
        if err != nil {
            conn.Close()
            return
        }

        var join JoinRequest
        if err := json.Unmarshal(msg, &join); err != nil || join.Username == "" {
            conn.WriteMessage(websocket.TextMessage, []byte(`{"type":"error","content":"请提供用户名"}`))
            conn.Close()
            return
        }

        if join.Room == "" {
            join.Room = "general"
        }

        client := NewClient(hub, NewRoom(join.Room), conn, join.Username)
        hub.Register <- client

        go client.WritePump()
        go client.ReadPump()
    })

    // 前端页面
    http.Handle("/", http.FileServer(http.Dir("./web")))

    log.Println("聊天室服务启动 :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

## 第六步：前端页面

```html
<!-- web/index.html -->
<!DOCTYPE html>
<html>
<head>
    <title>Go 聊天室</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: sans-serif; display: flex; height: 100vh; }
        #sidebar { width: 200px; background: #f5f5f5; padding: 20px; }
        #chat { flex: 1; display: flex; flex-direction: column; }
        #messages { flex: 1; overflow-y: auto; padding: 20px; }
        #input-bar { display: flex; padding: 10px 20px; border-top: 1px solid #ddd; }
        #input-bar input { flex: 1; padding: 10px; border: 1px solid #ddd; border-radius: 4px; }
        #input-bar button { margin-left: 10px; padding: 10px 20px; background: #007bff; color: white; border: none; border-radius: 4px; cursor: pointer; }
        .msg { margin: 10px 0; }
        .msg .user { font-weight: bold; color: #007bff; }
        .msg .time { color: #999; font-size: 12px; margin-left: 10px; }
        .system { color: #28a745; font-style: italic; }
    </style>
</head>
<body>
    <div id="sidebar">
        <h3>房间: <span id="roomName"></span></h3>
        <h4>在线用户</h4>
        <ul id="userList"></ul>
    </div>
    <div id="chat">
        <div id="messages"></div>
        <div id="input-bar">
            <input type="text" id="msgInput" placeholder="输入消息..." autofocus>
            <button id="sendBtn">发送</button>
        </div>
    </div>

    <script>
        const room = prompt('输入房间名:') || 'general';
        const username = prompt('输入用户名:') || '用户' + Math.floor(Math.random() * 1000);
        document.getElementById('roomName').textContent = room;

        const ws = new WebSocket(`ws://${location.host}/ws`);

        ws.onopen = () => {
            ws.send(JSON.stringify({ room, username }));
        };

        ws.onmessage = (e) => {
            const msg = JSON.parse(e.data);
            displayMessage(msg);
        };

        function displayMessage(msg) {
            const div = document.createElement('div');
            div.className = 'msg';

            if (msg.type === 'users') {
                updateUserList(msg.users);
                return;
            }

            if (msg.type === 'join' || msg.type === 'leave') {
                div.className = 'msg system';
                div.textContent = msg.content;
            } else {
                div.innerHTML = `<span class="user">${msg.username}</span>` +
                    `<span class="time">${msg.time}</span>` +
                    `<div>${msg.content}</div>`;
            }

            document.getElementById('messages').appendChild(div);
            div.scrollIntoView();
        }

        function updateUserList(users) {
            const list = document.getElementById('userList');
            list.innerHTML = users.map(u => `<li>${u}</li>`).join('');
        }

        document.getElementById('sendBtn').onclick = sendMessage;
        document.getElementById('msgInput').onkeypress = (e) => {
            if (e.key === 'Enter') sendMessage();
        };

        function sendMessage() {
            const input = document.getElementById('msgInput');
            const content = input.value.trim();
            if (!content) return;

            ws.send(JSON.stringify({ type: 'message', content }));
            input.value = '';
        }
    </script>
</body>
</html>
```

## 运行

```bash
go mod init chatroom
go get github.com/gorilla/websocket
go run cmd/server/main.go
# 浏览器访问 http://localhost:8080
```

## 扩展方向

- 增加消息持久化（聊天记录）
- WebSocket 集群（使用 Redis Pub/Sub 广播）
- 私聊/@功能
- 文字样式和图片发送

---

