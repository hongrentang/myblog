---
title: "Go from Beginner to Pro · Part 24: Project 5 — WebSocket Chat Room"
date: 2025-01-17
weight: 1224
draft: false
tags: ["go"]
---

## Project Introduction

Build a real-time chat room based on WebSocket, supporting multiple rooms, message broadcasting, and online user list. Uses gorilla/websocket for WebSocket communication.

## Project Structure

```
chatroom/
├── cmd/server/main.go
├── internal/
│   ├── hub/
│   │   └── hub.go        # Central dispatcher
│   ├── client/
│   │   └── client.go     # WebSocket client
│   └── room/
│       └── room.go       # Chat room
├── web/
│   └── index.html        # Frontend page
├── go.mod
└── go.sum
```

## Step 1: Data Model

```go
// Message structure
type Message struct {
    Type     string      `json:"type"`     // join/leave/message/error
    Room     string      `json:"room"`     // Room name
    Username string      `json:"username"` // Username
    Content  string      `json:"content"`  // Message content
    Users    []string    `json:"users,omitempty"` // Online user list
    Time     string      `json:"time"`     // Timestamp
}
```

## Step 2: Chat Room Management

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

## Step 3: WebSocket Client

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
                log.Printf("Read error: %v", err)
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
        // Send buffer full, disconnect
        close(c.done)
    }
}
```

## Step 4: Hub Central Dispatcher

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

            // Broadcast user join
            h.broadcastRoomMessage(room, Message{
                Type:     "join",
                Room:     room.Name,
                Username: client.Username,
                Content:  fmt.Sprintf("%s joined the chat", client.Username),
                Time:     time.Now().Format("15:04:05"),
            })

            // Update user list
            h.broadcastRoomMessage(room, Message{
                Type: "users",
                Room: room.Name,
                Users: room.Users(),
            })

            log.Printf("%s joined room %s (%d online)", client.Username, room.Name, room.Count())

        case client := <-h.Unregister:
            room := client.Room
            if room != nil {
                room.Remove(client)
                close(client.Send)

                // Broadcast user leave
                h.broadcastRoomMessage(room, Message{
                    Type:     "leave",
                    Room:     room.Name,
                    Username: client.Username,
                    Content:  fmt.Sprintf("%s left the chat", client.Username),
                    Time:     time.Now().Format("15:04:05"),
                })

                // Update user list
                h.broadcastRoomMessage(room, Message{
                    Type: "users",
                    Room: room.Name,
                    Users: room.Users(),
                })

                log.Printf("%s left room %s (%d online)", client.Username, room.Name, room.Count())
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

## Step 5: HTTP and WebSocket Entry

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
        return true // Allow all origins in development
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
            log.Println("WebSocket upgrade failed:", err)
            return
        }

        // Read first message (join room info)
        _, msg, err := conn.ReadMessage()
        if err != nil {
            conn.Close()
            return
        }

        var join JoinRequest
        if err := json.Unmarshal(msg, &join); err != nil || join.Username == "" {
            conn.WriteMessage(websocket.TextMessage, []byte(`{"type":"error","content":"please provide a username"}`))
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

    // Frontend page
    http.Handle("/", http.FileServer(http.Dir("./web")))

    log.Println("Chat room server starting :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

## Step 6: Frontend Page

```html
<!-- web/index.html -->
<!DOCTYPE html>
<html>
<head>
    <title>Go Chat Room</title>
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
        <h3>Room: <span id="roomName"></span></h3>
        <h4>Online Users</h4>
        <ul id="userList"></ul>
    </div>
    <div id="chat">
        <div id="messages"></div>
        <div id="input-bar">
            <input type="text" id="msgInput" placeholder="Type a message..." autofocus>
            <button id="sendBtn">Send</button>
        </div>
    </div>

    <script>
        const room = prompt('Enter room name:') || 'general';
        const username = prompt('Enter username:') || 'User' + Math.floor(Math.random() * 1000);
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

## Running

```bash
go mod init chatroom
go get github.com/gorilla/websocket
go run cmd/server/main.go
# Open browser at http://localhost:8080
```

## Extension Ideas

- Add message persistence (chat history)
- WebSocket cluster (use Redis Pub/Sub for broadcasting)
- Private messages/@mentions
- Text formatting and image sharing

---

