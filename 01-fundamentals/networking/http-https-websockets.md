# HTTP, HTTPS, and WebSockets

## Overview

Understanding web protocols is fundamental to system design. This guide covers HTTP, HTTPS, and WebSockets - the protocols that power modern web applications.

---

## HTTP (HyperText Transfer Protocol)

### What is HTTP?

HTTP is an **application-layer protocol** for transmitting hypermedia documents (HTML, JSON, images, etc.). It's the foundation of data communication on the web.

**Key Characteristics:**
- **Request-Response model:** Client sends request, server sends response
- **Stateless:** Each request is independent (no memory of previous requests)
- **Text-based:** Human-readable headers and methods
- **Port:** 80 (default)

### HTTP Request Structure

```http
GET /api/users/123 HTTP/1.1
Host: api.example.com
User-Agent: Mozilla/5.0
Accept: application/json
Authorization: Bearer token123
Cookie: session_id=abc123

[Optional Request Body]
```

**Components:**
1. **Request Line:** `METHOD /path HTTP/version`
2. **Headers:** Metadata about the request
3. **Body:** Data sent to server (POST, PUT, PATCH)

### HTTP Methods

| Method | Purpose | Idempotent | Safe | Has Body |
|--------|---------|------------|------|----------|
| **GET** | Retrieve resource | ✅ Yes | ✅ Yes | ❌ No |
| **POST** | Create resource | ❌ No | ❌ No | ✅ Yes |
| **PUT** | Update/replace resource | ✅ Yes | ❌ No | ✅ Yes |
| **PATCH** | Partial update | ❌ No | ❌ No | ✅ Yes |
| **DELETE** | Remove resource | ✅ Yes | ❌ No | ❌ No |
| **HEAD** | Get headers only | ✅ Yes | ✅ Yes | ❌ No |
| **OPTIONS** | Get allowed methods | ✅ Yes | ✅ Yes | ❌ No |

**Idempotent:** Multiple identical requests have same effect as single request.

**Examples:**
```python
import requests

# GET - Retrieve user
response = requests.get('https://api.example.com/users/123')

# POST - Create user
response = requests.post('https://api.example.com/users', json={
    'name': 'Alice',
    'email': 'alice@example.com'
})

# PUT - Replace user
response = requests.put('https://api.example.com/users/123', json={
    'name': 'Alice Updated',
    'email': 'alice.new@example.com'
})

# PATCH - Partial update
response = requests.patch('https://api.example.com/users/123', json={
    'email': 'alice.new@example.com'
})

# DELETE - Remove user
response = requests.delete('https://api.example.com/users/123')
```

### HTTP Status Codes

**1xx - Informational:**
- `100 Continue` - Continue with request
- `101 Switching Protocols` - Upgrading to WebSocket

**2xx - Success:**
- `200 OK` - Request succeeded
- `201 Created` - Resource created
- `202 Accepted` - Request accepted, processing async
- `204 No Content` - Success but no response body

**3xx - Redirection:**
- `301 Moved Permanently` - Resource moved, update bookmarks
- `302 Found` - Temporary redirect
- `304 Not Modified` - Use cached version

**4xx - Client Errors:**
- `400 Bad Request` - Invalid request syntax
- `401 Unauthorized` - Authentication required
- `403 Forbidden` - No permission
- `404 Not Found` - Resource doesn't exist
- `429 Too Many Requests` - Rate limit exceeded

**5xx - Server Errors:**
- `500 Internal Server Error` - Server crashed
- `502 Bad Gateway` - Upstream server error
- `503 Service Unavailable` - Server overloaded
- `504 Gateway Timeout` - Upstream server timeout

### HTTP Headers

**Request Headers:**
```http
Host: api.example.com                  # Required in HTTP/1.1
User-Agent: Mozilla/5.0                # Client information
Accept: application/json               # Expected response format
Accept-Encoding: gzip, deflate         # Supported compression
Authorization: Bearer token123         # Authentication
Cookie: session_id=abc; user=alice     # Cookies
Content-Type: application/json         # Body format
Content-Length: 1234                   # Body size in bytes
```

**Response Headers:**
```http
Content-Type: application/json
Content-Length: 5678
Cache-Control: public, max-age=3600
Set-Cookie: session_id=xyz; HttpOnly; Secure
ETag: "abc123"
Location: /api/users/456               # For redirects
Access-Control-Allow-Origin: *         # CORS
```

### HTTP Versions

**HTTP/1.0 (1996):**
- New connection per request
- No persistent connections

**HTTP/1.1 (1997):**
- Persistent connections (keep-alive)
- Pipelining (send multiple requests without waiting)
- Chunked transfer encoding
- Host header (virtual hosting)

**HTTP/2 (2015):**
- Binary protocol (not text)
- Multiplexing (multiple requests on one connection)
- Server push
- Header compression (HPACK)
- Stream prioritization

**HTTP/3 (2022):**
- QUIC protocol (UDP-based, not TCP)
- Faster handshake
- Better on poor networks
- No head-of-line blocking

**Performance Comparison:**
```
HTTP/1.1: 6 connections × 1 request each = 6 requests
HTTP/2:   1 connection × 6 requests = 6 requests (faster)
HTTP/3:   1 connection × 6 requests (even faster, UDP-based)
```

---

## HTTPS (HTTP Secure)

### What is HTTPS?

HTTP over **TLS/SSL** (Transport Layer Security). Encrypts data between client and server.

**Port:** 443 (default)

### How HTTPS Works

**TLS Handshake:**
```
1. Client → Server: "Hello, I support TLS 1.3, these cipher suites"
2. Server → Client: "Let's use TLS 1.3, this cipher suite, here's my certificate"
3. Client verifies certificate (signed by trusted CA)
4. Client generates session key, encrypts with server's public key
5. Server decrypts with private key
6. Both have shared session key
7. Encrypted communication begins
```

**Visual:**
```
Client                              Server
   |                                   |
   |--- ClientHello ------------------>|
   |<-- ServerHello, Certificate ------|
   |--- Key Exchange ----------------->|
   |<-- Finished ----------------------|
   |--- Encrypted Data --------------->|
   |<-- Encrypted Data ----------------|
```

### Benefits of HTTPS

1. **Encryption:** Data can't be read by third parties
2. **Authentication:** Verify server identity
3. **Data Integrity:** Detect if data was tampered with
4. **SEO:** Google ranks HTTPS sites higher
5. **Required for:** Modern APIs, PWAs, geolocation, camera access

### SSL/TLS Certificates

**Types:**
- **Domain Validated (DV):** Basic, automated (Let's Encrypt)
- **Organization Validated (OV):** Verify organization identity
- **Extended Validation (EV):** Highest validation (green bar)

**Free Certificates:**
```bash
# Let's Encrypt (free, automated)
sudo certbot --nginx -d example.com -d www.example.com
```

**Certificate Content:**
```
Subject: CN=example.com
Issuer: Let's Encrypt
Valid from: 2024-01-01
Valid to: 2024-04-01
Public key: [RSA 2048-bit key]
Signature: [CA signature]
```

### HTTP vs HTTPS

| Aspect | HTTP | HTTPS |
|--------|------|-------|
| **Port** | 80 | 443 |
| **Encryption** | None | TLS/SSL |
| **Speed** | Slightly faster | Slightly slower (TLS overhead) |
| **SEO** | Lower ranking | Higher ranking |
| **Cost** | Free | Free (Let's Encrypt) or paid |
| **Browser Indicator** | "Not Secure" | Padlock icon |

---

## WebSockets

### What are WebSockets?

A **full-duplex, bidirectional communication protocol** over a single TCP connection. Allows server to push data to client without client requesting it.

**Port:** 80 (ws://) or 443 (wss://)

### HTTP vs WebSocket

**HTTP (Request-Response):**
```
Client: "Give me data" → Server: "Here's data"
Client: "Give me data" → Server: "Here's data"
Client: "Give me data" → Server: "Here's data"
(New request each time)
```

**WebSocket (Persistent Connection):**
```
Client: "Open connection" → Server: "Connected"
Server → Client: "Here's new data"
Server → Client: "Here's more data"
Client → Server: "I'm sending data"
(Connection stays open)
```

### WebSocket Handshake

**Upgrade from HTTP to WebSocket:**

**Client Request:**
```http
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

**Server Response:**
```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**After handshake:** Connection upgraded to WebSocket, full-duplex communication.

### WebSocket Frames

Unlike HTTP (text headers), WebSocket uses **binary frames**:

```
Frame structure:
- FIN bit (is this the final fragment?)
- Opcode (text, binary, ping, pong, close)
- Payload length
- Masking key (client to server only)
- Payload data
```

### Use Cases

✅ **Perfect For:**
1. **Real-time chat** - Instant messaging
2. **Live notifications** - Push notifications
3. **Collaborative editing** - Google Docs-style editing
4. **Live feeds** - Stock prices, sports scores
5. **Multiplayer games** - Real-time game state
6. **Live streaming** - Audio/video streaming
7. **IoT devices** - Sensor data streaming

❌ **Not For:**
1. **Simple API requests** - Use HTTP
2. **Large file transfers** - Use HTTP with chunking
3. **SEO-critical content** - Use HTTP
4. **Stateless operations** - Use HTTP

### Client-Side Example (JavaScript)

```javascript
// Connect to WebSocket server
const ws = new WebSocket('wss://server.example.com/chat');

// Connection opened
ws.addEventListener('open', (event) => {
    console.log('Connected to server');
    ws.send('Hello Server!');
});

// Listen for messages
ws.addEventListener('message', (event) => {
    console.log('Message from server:', event.data);
});

// Connection closed
ws.addEventListener('close', (event) => {
    console.log('Disconnected from server');
});

// Error handling
ws.addEventListener('error', (event) => {
    console.error('WebSocket error:', event);
});

// Send message
function sendMessage(message) {
    if (ws.readyState === WebSocket.OPEN) {
        ws.send(message);
    }
}

// Close connection
ws.close();
```

### Server-Side Example (Python - FastAPI)

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import List

app = FastAPI()

# Active connections
class ConnectionManager:
    def __init__(self):
        self.active_connections: List[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)

    async def broadcast(self, message: str):
        for connection in self.active_connections:
            await connection.send_text(message)

manager = ConnectionManager()

@app.websocket("/ws/chat")
async def websocket_endpoint(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            # Receive message from client
            data = await websocket.receive_text()

            # Broadcast to all clients
            await manager.broadcast(f"Message: {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)
        await manager.broadcast("User disconnected")
```

### Server-Side Example (Node.js - Socket.IO)

```javascript
const express = require('express');
const http = require('http');
const socketIO = require('socket.io');

const app = express();
const server = http.createServer(app);
const io = socketIO(server);

// Connection event
io.on('connection', (socket) => {
    console.log('User connected:', socket.id);

    // Listen for messages
    socket.on('message', (data) => {
        console.log('Received:', data);

        // Broadcast to all clients
        io.emit('message', data);

        // Or send to specific room
        socket.to('room1').emit('message', data);
    });

    // Disconnection
    socket.on('disconnect', () => {
        console.log('User disconnected:', socket.id);
    });
});

server.listen(3000, () => {
    console.log('Server listening on port 3000');
});
```

### WebSocket vs HTTP Long Polling

**HTTP Long Polling:**
```
Client: "Any updates?" → Server: (waits... waits...) → "Yes, here's update"
Client: "Any updates?" → Server: (waits... waits...) → "Yes, here's update"
(Repeatedly open/close connections)
```

**WebSocket:**
```
Client: "Connect" → Server: "Connected"
(Connection stays open, messages flow both ways)
Server → Client: "Update 1"
Server → Client: "Update 2"
Client → Server: "My update"
```

**Comparison:**

| Aspect | HTTP Long Polling | WebSocket |
|--------|-------------------|-----------|
| **Latency** | Higher (new request) | Lower (persistent) |
| **Overhead** | High (HTTP headers each time) | Low (small frames) |
| **Server Load** | Higher (many connections) | Lower (efficient) |
| **Complexity** | Moderate | Higher |
| **Firewall Friendly** | More compatible | Sometimes blocked |

### WebSocket Scaling

**Challenge:** WebSockets are **stateful** (connection tied to specific server).

**Solution 1: Sticky Sessions**
```
Load Balancer (sticky sessions)
    ↓ (same user always to same server)
Server 1 (WebSocket connections for users A, B, C)
Server 2 (WebSocket connections for users D, E, F)
```

**Solution 2: Pub/Sub with Redis**
```
Server 1 (WebSocket connections)
    ↓ publish
Redis Pub/Sub
    ↓ subscribe
Server 2 (WebSocket connections)

User on Server 1 sends message → Redis → All servers broadcast to their connected users
```

**Implementation:**
```python
import redis
import asyncio

# Redis pub/sub
redis_client = redis.Redis()
pubsub = redis_client.pubsub()
pubsub.subscribe('chat_messages')

@app.websocket("/ws/chat")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()

    # Listen to Redis
    async def redis_listener():
        for message in pubsub.listen():
            if message['type'] == 'message':
                await websocket.send_text(message['data'])

    # Start background task
    task = asyncio.create_task(redis_listener())

    try:
        while True:
            data = await websocket.receive_text()

            # Publish to Redis (all servers will receive)
            redis_client.publish('chat_messages', data)
    except WebSocketDisconnect:
        task.cancel()
```

---

## Comparison Table

| Feature | HTTP | HTTPS | WebSocket |
|---------|------|-------|-----------|
| **Protocol** | Application layer | HTTP + TLS | Application layer |
| **Port** | 80 | 443 | 80 (ws) / 443 (wss) |
| **Encryption** | None | TLS/SSL | Optional (wss) |
| **Connection** | Request-response | Request-response | Persistent |
| **Direction** | Client → Server | Client → Server | Bidirectional |
| **Overhead** | High (headers) | Higher (TLS + headers) | Low (small frames) |
| **Latency** | Higher | Slightly higher | Lower |
| **Use Case** | APIs, websites | Secure APIs, websites | Real-time apps |
| **Stateful** | No | No | Yes |

---

## Best Practices

### HTTP/HTTPS

1. **Use HTTPS everywhere** - Even for non-sensitive data
2. **Set proper headers:**
```http
Cache-Control: public, max-age=3600
Content-Security-Policy: default-src 'self'
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Strict-Transport-Security: max-age=31536000
```

3. **Compress responses:**
```python
# Flask example
from flask_compress import Compress

app = Flask(__name__)
Compress(app)  # Auto-compress responses
```

4. **Use HTTP/2** - Enable on server
5. **Implement rate limiting** - Prevent abuse

### WebSockets

1. **Authenticate connections:**
```javascript
// Send token in initial message
ws.addEventListener('open', () => {
    ws.send(JSON.stringify({ type: 'auth', token: 'jwt_token' }));
});
```

2. **Handle reconnections:**
```javascript
function connect() {
    const ws = new WebSocket('wss://server.example.com');

    ws.addEventListener('close', () => {
        // Reconnect after 5 seconds
        setTimeout(connect, 5000);
    });
}
```

3. **Heartbeat/ping-pong:**
```python
# Server sends ping every 30 seconds
async def heartbeat(websocket):
    while True:
        await asyncio.sleep(30)
        await websocket.send_text('ping')
```

4. **Limit message size:**
```python
MAX_MESSAGE_SIZE = 1024 * 64  # 64 KB

data = await websocket.receive_text()
if len(data) > MAX_MESSAGE_SIZE:
    await websocket.close(code=1009)  # Message too big
```

5. **Monitor connections:**
```python
# Track active connections
active_connections = 0

@websocket_connect
def on_connect():
    global active_connections
    active_connections += 1
    metrics.gauge('websocket.connections', active_connections)
```

---

## Summary

**HTTP:**
- Request-response protocol
- Stateless
- Text-based
- Use for: APIs, websites, file downloads

**HTTPS:**
- HTTP + TLS encryption
- Secure, authenticated
- Slightly slower than HTTP
- Use for: Everything (mandatory for modern web)

**WebSockets:**
- Full-duplex, persistent connection
- Bidirectional communication
- Low latency, low overhead
- Use for: Real-time applications

**Interview Tips:**
- Explain when to use each protocol
- Discuss HTTP/2 multiplexing
- Cover WebSocket scaling challenges (sticky sessions, pub/sub)
- Mention security (HTTPS, WSS)

---

**Next:** Learn about [TCP vs UDP](tcp-vs-udp.md) - the transport layer protocols.
