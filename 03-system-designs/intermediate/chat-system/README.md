# Chat System Design (WhatsApp / Slack)

> **Difficulty:** Intermediate
> **Topics:** WebSockets, Message Queue, Real-time Communication, Group Chat, Read Receipts
> **Companies:** Facebook (WhatsApp), Slack, Discord, Telegram, Signal

---

## 1. Problem Statement

Design a **real-time chat system** like WhatsApp or Slack that supports **one-on-one** and **group messaging**.

### Core Features

**Users:**
- Send and receive messages in real-time
- See online/offline status
- View message history
- Create and join group chats
- Send multimedia (images, files)

**System:**
- Deliver messages with low latency (<100ms)
- Show typing indicators
- Display read receipts
- Support offline users (deliver when online)
- Push notifications

---

## 2. Requirements

### Functional Requirements

1. **One-on-One Chat:**
   - User A sends message to User B
   - Real-time delivery if B is online
   - Store message if B is offline, deliver when online

2. **Group Chat:**
   - Multiple users in a conversation
   - Message delivered to all members
   - Show who sent each message

3. **Online Presence:**
   - Show who's online/offline
   - Last seen timestamp

4. **Message History:**
   - Persist all messages
   - Pagination for old messages

5. **Delivery Status:**
   - Sent (✓)
   - Delivered (✓✓)
   - Read (✓✓ blue)

### Non-Functional Requirements

1. **Low Latency:** <100ms message delivery
2. **High Availability:** 99.99% uptime
3. **Scalability:** 1 billion users, 100 million DAU
4. **Consistency:** Messages delivered in order
5. **Durability:** No message loss

### Out of Scope

- End-to-end encryption (Signal protocol)
- Voice/video calls
- Stickers, reactions
- Message search

---

## 3. Back-of-Envelope Estimation

### Traffic

```
Daily Active Users (DAU): 100M
Messages per user per day: 50
Total messages/day: 5 billion

Messages/second: 5B / 86400 ≈ 58K messages/sec
Peak (3x): 174K messages/sec

WebSocket connections (online users):
Assume 10% online at any time: 10M concurrent connections
```

### Storage

```
Per message:
  message_id: 8 bytes
  sender_id: 8 bytes
  receiver_id/group_id: 8 bytes
  content: 200 bytes (avg)
  timestamp: 8 bytes
  metadata: 20 bytes
  Total: ~250 bytes

Daily storage: 5B messages × 250 bytes = 1.25 TB/day
Yearly: 1.25 TB × 365 = 456 TB/year
With replication (3x): 1.4 PB/year
```

### Bandwidth

```
Incoming: 174K msg/sec × 250 bytes = 43.5 MB/sec = 348 Mbps
Outgoing (fanout to online users): Similar magnitude
Total: ~700 Mbps (manageable)
```

---

## 4. High-Level Design

```
┌─────────────┐          ┌─────────────┐
│  User A     │          │  User B     │
│  (Mobile)   │          │  (Web)      │
└──────┬──────┘          └──────┬──────┘
       │                        │
       │  WebSocket/Long Poll   │  WebSocket
       │                        │
┌──────▼────────────────────────▼──────┐
│         API Gateway / LB             │
│         (Nginx, Kong)                │
└──────┬────────────────────┬──────────┘
       │                    │
       ▼                    ▼
┌─────────────┐      ┌─────────────┐
│ WebSocket   │      │ WebSocket   │
│ Server 1    │      │ Server 2    │  (Stateful)
└──────┬──────┘      └──────┬──────┘
       │                    │
       └────────┬───────────┘
                │
        ┌───────▼────────┐
        │  Message Queue │  (Kafka)
        │  (Fanout)      │
        └───────┬────────┘
                │
       ┌────────┼────────┬────────┐
       │        │        │        │
       ▼        ▼        ▼        ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│Message   │ │Presence  │ │Notif     │
│Service   │ │Service   │ │Service   │
└────┬─────┘ └────┬─────┘ └──────────┘
     │            │
     ▼            ▼
┌──────────┐ ┌──────────┐
│PostgreSQL│ │  Redis   │
│(Messages)│ │(Presence)│
└──────────┘ └──────────┘
```

---

## 5. Detailed Design

### 5.1 WebSocket Connection

**Why WebSockets?**

```
HTTP Long Polling (old):
Client: Request → Server: Wait... → Response after 30s or new message
  ❌ Inefficient, high latency

WebSockets:
Client ←→ Server: Persistent bidirectional connection
  ✅ Real-time, low latency, efficient
```

**Implementation:**

```python
import asyncio
import websockets
import json
from typing import Dict, Set

class WebSocketServer:
    """
    WebSocket server for real-time messaging
    """

    def __init__(self):
        # user_id → Set of WebSocket connections (multiple devices)
        self.connections: Dict[str, Set[websockets.WebSocketServerProtocol]] = {}

    async def register(self, websocket: websockets.WebSocketServerProtocol, user_id: str):
        """Register user's WebSocket connection"""
        if user_id not in self.connections:
            self.connections[user_id] = set()
        self.connections[user_id].add(websocket)
        print(f"User {user_id} connected (total connections: {len(self.connections[user_id])})")

    async def unregister(self, websocket: websockets.WebSocketServerProtocol, user_id: str):
        """Unregister user's WebSocket connection"""
        if user_id in self.connections:
            self.connections[user_id].discard(websocket)
            if not self.connections[user_id]:
                del self.connections[user_id]
        print(f"User {user_id} disconnected")

    async def send_to_user(self, user_id: str, message: dict):
        """Send message to all of user's devices"""
        if user_id in self.connections:
            # Send to all active connections (multiple devices)
            for ws in self.connections[user_id]:
                try:
                    await ws.send(json.dumps(message))
                except Exception as e:
                    print(f"Error sending to {user_id}: {e}")

    async def handle_client(self, websocket: websockets.WebSocketServerProtocol, path: str):
        """Handle client WebSocket connection"""
        user_id = None
        try:
            # First message should be authentication
            auth_msg = await websocket.recv()
            auth_data = json.loads(auth_msg)
            user_id = auth_data.get("user_id")

            if not user_id:
                await websocket.close()
                return

            # Register connection
            await self.register(websocket, user_id)

            # Listen for messages
            async for message in websocket:
                data = json.loads(message)
                await self.handle_message(user_id, data)

        except websockets.exceptions.ConnectionClosed:
            pass
        finally:
            if user_id:
                await self.unregister(websocket, user_id)

    async def handle_message(self, sender_id: str, data: dict):
        """Handle incoming message"""
        msg_type = data.get("type")

        if msg_type == "chat":
            # Send message
            receiver_id = data.get("receiver_id")
            content = data.get("content")

            # Store in database
            message_id = await self.store_message(sender_id, receiver_id, content)

            # Deliver to receiver if online
            await self.send_to_user(receiver_id, {
                "type": "new_message",
                "message_id": message_id,
                "sender_id": sender_id,
                "content": content,
                "timestamp": data.get("timestamp")
            })

            # Send delivery confirmation to sender
            await self.send_to_user(sender_id, {
                "type": "delivered",
                "message_id": message_id
            })

        elif msg_type == "typing":
            # Typing indicator
            receiver_id = data.get("receiver_id")
            await self.send_to_user(receiver_id, {
                "type": "typing",
                "user_id": sender_id
            })

    async def store_message(self, sender_id: str, receiver_id: str, content: str) -> str:
        """Store message in database (simplified)"""
        # In production: INSERT INTO messages (...) VALUES (...)
        import uuid
        return str(uuid.uuid4())

# Start server
async def main():
    server = WebSocketServer()
    async with websockets.serve(server.handle_client, "0.0.0.0", 8765):
        await asyncio.Future()  # Run forever

# asyncio.run(main())
```

---

### 5.2 Message Delivery Flow

**One-on-One Chat:**

```
User A (sender) → WebSocket Server → Message Queue → Message Service
                                                    ↓
                                              Store in DB
                                                    ↓
                        ┌───────────────────────────┴────────┐
                        │                                    │
                        ▼                                    ▼
              User B (online)                       User B (offline)
              WebSocket delivery                    Store for later
```

**Implementation:**

```python
import asyncpg
import aio_pika
import json
from datetime import datetime

class MessageService:
    """
    Service for handling message persistence and delivery
    """

    def __init__(self, db_pool, rabbitmq_connection):
        self.db = db_pool
        self.mq = rabbitmq_connection

    async def send_message(self, sender_id: str, receiver_id: str, content: str):
        """
        Send message (1-on-1 chat)
        """
        # 1. Store message in database
        message_id = await self._store_message(sender_id, receiver_id, content)

        # 2. Publish to message queue for delivery
        await self._publish_message({
            "message_id": message_id,
            "sender_id": sender_id,
            "receiver_id": receiver_id,
            "content": content,
            "timestamp": datetime.utcnow().isoformat()
        })

        return message_id

    async def _store_message(self, sender_id: str, receiver_id: str, content: str) -> str:
        """Store message in PostgreSQL"""
        query = """
            INSERT INTO messages (sender_id, receiver_id, content, created_at)
            VALUES ($1, $2, $3, NOW())
            RETURNING message_id
        """
        message_id = await self.db.fetchval(query, sender_id, receiver_id, content)
        return str(message_id)

    async def _publish_message(self, message: dict):
        """Publish to RabbitMQ for delivery"""
        channel = await self.mq.channel()
        await channel.default_exchange.publish(
            aio_pika.Message(body=json.dumps(message).encode()),
            routing_key=f"user.{message['receiver_id']}"
        )

    async def get_chat_history(self, user_id: str, other_user_id: str, limit: int = 50, offset: int = 0):
        """
        Get message history between two users
        """
        query = """
            SELECT message_id, sender_id, receiver_id, content, created_at, read_at
            FROM messages
            WHERE (sender_id = $1 AND receiver_id = $2)
               OR (sender_id = $2 AND receiver_id = $1)
            ORDER BY created_at DESC
            LIMIT $3 OFFSET $4
        """
        messages = await self.db.fetch(query, user_id, other_user_id, limit, offset)
        return [dict(msg) for msg in messages]

    async def mark_as_read(self, message_id: str, user_id: str):
        """Mark message as read"""
        query = """
            UPDATE messages
            SET read_at = NOW()
            WHERE message_id = $1 AND receiver_id = $2 AND read_at IS NULL
        """
        await self.db.execute(query, message_id, user_id)
```

---

### 5.3 Group Chat

**Challenge:** Fanout to multiple users

```
User A sends message to Group (100 members)
→ Message Service must deliver to 100 users
→ 100 WebSocket deliveries

Optimization: Message Queue with fanout
```

**Implementation:**

```python
class GroupChatService:
    """Handle group chat messaging"""

    def __init__(self, db_pool, ws_server):
        self.db = db_pool
        self.ws = ws_server

    async def send_group_message(self, sender_id: str, group_id: str, content: str):
        """
        Send message to group chat
        """
        # 1. Store message
        message_id = await self._store_group_message(sender_id, group_id, content)

        # 2. Get group members
        members = await self._get_group_members(group_id)

        # 3. Fanout to all members (except sender)
        for member_id in members:
            if member_id != sender_id:
                await self.ws.send_to_user(member_id, {
                    "type": "group_message",
                    "message_id": message_id,
                    "group_id": group_id,
                    "sender_id": sender_id,
                    "content": content,
                    "timestamp": datetime.utcnow().isoformat()
                })

        return message_id

    async def _store_group_message(self, sender_id: str, group_id: str, content: str) -> str:
        """Store group message"""
        query = """
            INSERT INTO group_messages (group_id, sender_id, content, created_at)
            VALUES ($1, $2, $3, NOW())
            RETURNING message_id
        """
        message_id = await self.db.fetchval(query, group_id, sender_id, content)
        return str(message_id)

    async def _get_group_members(self, group_id: str) -> list:
        """Get all members of a group"""
        query = """
            SELECT user_id FROM group_members WHERE group_id = $1
        """
        rows = await self.db.fetch(query, group_id)
        return [row['user_id'] for row in rows]

    async def create_group(self, creator_id: str, name: str, member_ids: list) -> str:
        """Create new group chat"""
        async with self.db.transaction():
            # Create group
            query = """
                INSERT INTO groups (name, created_by, created_at)
                VALUES ($1, $2, NOW())
                RETURNING group_id
            """
            group_id = await self.db.fetchval(query, name, creator_id)

            # Add members (including creator)
            all_members = set(member_ids + [creator_id])
            for member_id in all_members:
                await self.db.execute(
                    "INSERT INTO group_members (group_id, user_id) VALUES ($1, $2)",
                    group_id, member_id
                )

        return str(group_id)
```

---

### 5.4 Online Presence

**Track user online/offline status:**

```python
import redis.asyncio as redis
from datetime import datetime

class PresenceService:
    """
    Track user online/offline status using Redis
    """

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.PRESENCE_TTL = 30  # seconds

    async def user_online(self, user_id: str):
        """Mark user as online"""
        await self.redis.setex(
            f"presence:{user_id}",
            self.PRESENCE_TTL,
            "online"
        )

    async def user_offline(self, user_id: str):
        """Mark user as offline"""
        await self.redis.delete(f"presence:{user_id}")
        # Set last seen
        await self.redis.set(
            f"last_seen:{user_id}",
            datetime.utcnow().isoformat()
        )

    async def is_online(self, user_id: str) -> bool:
        """Check if user is online"""
        status = await self.redis.get(f"presence:{user_id}")
        return status == b"online"

    async def get_last_seen(self, user_id: str) -> str:
        """Get user's last seen timestamp"""
        last_seen = await self.redis.get(f"last_seen:{user_id}")
        return last_seen.decode() if last_seen else None

    async def heartbeat(self, user_id: str):
        """
        Refresh user's online status (called periodically from client)
        """
        await self.user_online(user_id)
```

---

## 6. Database Schema

```sql
-- Users
CREATE TABLE users (
    user_id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    phone VARCHAR(20) UNIQUE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- One-on-one messages
CREATE TABLE messages (
    message_id BIGSERIAL PRIMARY KEY,
    sender_id BIGINT NOT NULL REFERENCES users(user_id),
    receiver_id BIGINT NOT NULL REFERENCES users(user_id),
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    delivered_at TIMESTAMP,
    read_at TIMESTAMP,

    INDEX idx_sender_receiver (sender_id, receiver_id, created_at),
    INDEX idx_receiver (receiver_id, created_at)
);

-- Groups
CREATE TABLE groups (
    group_id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    created_by BIGINT REFERENCES users(user_id),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Group members
CREATE TABLE group_members (
    group_id BIGINT REFERENCES groups(group_id),
    user_id BIGINT REFERENCES users(user_id),
    joined_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (group_id, user_id)
);

-- Group messages
CREATE TABLE group_messages (
    message_id BIGSERIAL PRIMARY KEY,
    group_id BIGINT NOT NULL REFERENCES groups(group_id),
    sender_id BIGINT NOT NULL REFERENCES users(user_id),
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),

    INDEX idx_group_created (group_id, created_at)
);
```

---

## 7. Scaling Considerations

### WebSocket Server Scaling

**Challenge:** WebSockets are stateful

```
User A connected to WS Server 1
User B connected to WS Server 2

A sends to B → Need to route between servers
```

**Solution: Service Discovery + Message Queue**

```python
# User A sends message
WS Server 1 → Publishes to Kafka topic "messages"
            → Message Service consumes, stores in DB
            → Publishes to Kafka topic "user.{receiver_id}"

# Delivery to User B
WS Server 2 → Subscribes to "user.*" topics
            → Receives message for User B
            → Delivers via WebSocket
```

---

## 8. Summary

**Chat System Architecture:**

✅ **WebSockets** for real-time bidirectional communication
✅ **Message Queue** (Kafka) for reliable delivery and scaling
✅ **PostgreSQL** for message persistence
✅ **Redis** for online presence tracking
✅ **Horizontal scaling** with stateless message service

**Key Metrics:**
- Latency: <100ms delivery
- Scale: 100M DAU, 58K messages/sec
- Storage: 456 TB/year

**Trade-offs:**
- WebSockets vs Long Polling (chose WebSockets for real-time)
- Synchronous vs Asynchronous delivery (async for scalability)
- Strong vs Eventual consistency (eventual for availability)

**Next:** Learn about [Search Autocomplete](../search-autocomplete/README.md) with Trie data structure.
