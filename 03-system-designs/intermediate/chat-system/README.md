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

## 8. Message Delivery Guarantees

### At-Least-Once vs Exactly-Once

**Challenge:** Ensure messages are delivered reliably

```python
# Scenario 1: Message sent but ACK lost
User A → Message → Server (stored)
                  ↓
         Server → Message → User B (delivered)
                  ↓
         Server → ACK lost in network
                  ↓
User A retries (thinks it failed)
         → Duplicate message!
```

**Solution: Idempotency with Message IDs**

```python
class IdempotentMessageService:
    """
    Ensures messages are delivered at-least-once but processed exactly-once
    """

    def __init__(self, db_pool, redis_client):
        self.db = db_pool
        self.redis = redis_client

    async def send_message(self, message_id: str, sender_id: str, receiver_id: str, content: str):
        """
        Send message with idempotency guarantee
        message_id is generated by the client
        """
        # Check if already processed (in last 24 hours)
        cache_key = f"msg:{message_id}"
        if await self.redis.exists(cache_key):
            # Already processed - return success without storing again
            return message_id

        # Process message (store in DB)
        query = """
            INSERT INTO messages (message_id, sender_id, receiver_id, content, created_at)
            VALUES ($1, $2, $3, $4, NOW())
            ON CONFLICT (message_id) DO NOTHING
            RETURNING message_id
        """
        result = await self.db.fetchval(query, message_id, sender_id, receiver_id, content)

        # Cache for 24 hours to handle retries
        await self.redis.setex(cache_key, 86400, "processed")

        return message_id

# Client-side: Generate message_id before sending
import uuid
message_id = str(uuid.uuid4())  # Client generates ID
# Now safe to retry with same message_id
```

---

### Message Ordering

**Challenge:** Messages arrive out of order

```python
User A sends:
  1. "Hello" (timestamp: 100ms)
  2. "How are you?" (timestamp: 101ms)

Network delays:
  Message 2 arrives first (via faster route)
  Message 1 arrives second

User B sees: "How are you?" then "Hello" ❌
```

**Solution 1: Sequence Numbers**

```python
class OrderedMessageService:
    """
    Ensures messages are delivered in order using sequence numbers
    """

    def __init__(self, redis_client):
        self.redis = redis_client

    async def send_message(self, conversation_id: str, sender_id: str, content: str):
        """
        Assign incrementing sequence number per conversation
        """
        # Get next sequence number for this conversation
        seq_key = f"seq:{conversation_id}"
        sequence_number = await self.redis.incr(seq_key)

        message = {
            "conversation_id": conversation_id,
            "sender_id": sender_id,
            "content": content,
            "sequence": sequence_number,
            "timestamp": datetime.utcnow().isoformat()
        }

        # Store and deliver
        await self._store_message(message)
        return message

    async def receive_messages(self, conversation_id: str, last_sequence: int):
        """
        Client requests messages after last_sequence
        Server returns messages in order
        """
        query = """
            SELECT * FROM messages
            WHERE conversation_id = $1 AND sequence > $2
            ORDER BY sequence ASC
        """
        messages = await self.db.fetch(query, conversation_id, last_sequence)
        return messages
```

**Solution 2: Lamport Timestamps (for distributed ordering)**

```python
class LamportClock:
    """
    Lamport logical clock for distributed message ordering
    """

    def __init__(self):
        self.time = 0

    def increment(self):
        """Increment on local event"""
        self.time += 1
        return self.time

    def update(self, received_time: int):
        """Update on receiving message from another process"""
        self.time = max(self.time, received_time) + 1
        return self.time

# Usage in chat system
class DistributedChatServer:
    def __init__(self, server_id):
        self.server_id = server_id
        self.clock = LamportClock()

    async def send_message(self, sender_id: str, receiver_id: str, content: str):
        """Send message with Lamport timestamp"""
        lamport_time = self.clock.increment()

        message = {
            "sender_id": sender_id,
            "receiver_id": receiver_id,
            "content": content,
            "lamport_time": lamport_time,
            "server_id": self.server_id
        }

        await self._publish_message(message)

    async def receive_message(self, message: dict):
        """Receive message and update clock"""
        self.clock.update(message["lamport_time"])
        await self._deliver_message(message)

# Messages can now be sorted by (lamport_time, server_id) for total ordering
```

---

## 9. Advanced Features

### 9.1 Typing Indicators

**Real-time notification when someone is typing**

```python
class TypingIndicatorService:
    """
    Efficient typing indicators with minimal overhead
    """

    def __init__(self, redis_client, ws_server):
        self.redis = redis_client
        self.ws = ws_server
        self.TYPING_TTL = 3  # seconds

    async def user_typing(self, conversation_id: str, user_id: str):
        """
        User started typing
        Set short-lived key in Redis (3 seconds TTL)
        """
        key = f"typing:{conversation_id}:{user_id}"

        # Only send notification if not already typing
        if not await self.redis.exists(key):
            # Broadcast to conversation members
            await self._broadcast_typing(conversation_id, user_id, is_typing=True)

        # Set/refresh TTL
        await self.redis.setex(key, self.TYPING_TTL, "1")

    async def user_stopped_typing(self, conversation_id: str, user_id: str):
        """
        User stopped typing (explicit)
        """
        key = f"typing:{conversation_id}:{user_id}"
        await self.redis.delete(key)
        await self._broadcast_typing(conversation_id, user_id, is_typing=False)

    async def _broadcast_typing(self, conversation_id: str, user_id: str, is_typing: bool):
        """
        Broadcast typing status to conversation members
        """
        # Get conversation members
        members = await self._get_conversation_members(conversation_id)

        for member_id in members:
            if member_id != user_id:  # Don't send to self
                await self.ws.send_to_user(member_id, {
                    "type": "typing",
                    "conversation_id": conversation_id,
                    "user_id": user_id,
                    "is_typing": is_typing
                })

    async def get_typing_users(self, conversation_id: str) -> list:
        """
        Get list of users currently typing in conversation
        """
        pattern = f"typing:{conversation_id}:*"
        keys = await self.redis.keys(pattern)

        # Extract user_ids from keys
        typing_users = [key.decode().split(":")[-1] for key in keys]
        return typing_users
```

**Client-side optimization: Debouncing**

```javascript
// Client sends typing indicator with debouncing
let typingTimeout;

function onKeyPress() {
    // Clear existing timeout
    clearTimeout(typingTimeout);

    // Send "typing" notification (if not already sent)
    if (!isTyping) {
        socket.send(JSON.stringify({type: "typing", conversation_id: conversationId}));
        isTyping = true;
    }

    // Auto-stop after 3 seconds of inactivity
    typingTimeout = setTimeout(() => {
        socket.send(JSON.stringify({type: "stopped_typing", conversation_id: conversationId}));
        isTyping = false;
    }, 3000);
}
```

---

### 9.2 Read Receipts

**Track message delivery and read status**

```python
class ReadReceiptService:
    """
    Track message delivery status: sent → delivered → read
    """

    def __init__(self, db_pool, ws_server):
        self.db = db_pool
        self.ws = ws_server

    async def mark_delivered(self, message_id: str, user_id: str):
        """
        Mark message as delivered (arrived at user's device)
        """
        query = """
            UPDATE messages
            SET delivered_at = NOW()
            WHERE message_id = $1
              AND receiver_id = $2
              AND delivered_at IS NULL
            RETURNING sender_id
        """
        sender_id = await self.db.fetchval(query, message_id, user_id)

        if sender_id:
            # Notify sender: message delivered
            await self.ws.send_to_user(sender_id, {
                "type": "receipt",
                "message_id": message_id,
                "status": "delivered",
                "timestamp": datetime.utcnow().isoformat()
            })

    async def mark_read(self, message_id: str, user_id: str):
        """
        Mark message as read (user opened the chat)
        """
        query = """
            UPDATE messages
            SET read_at = NOW(),
                delivered_at = COALESCE(delivered_at, NOW())
            WHERE message_id = $1
              AND receiver_id = $2
              AND read_at IS NULL
            RETURNING sender_id
        """
        sender_id = await self.db.fetchval(query, message_id, user_id)

        if sender_id:
            # Notify sender: message read
            await self.ws.send_to_user(sender_id, {
                "type": "receipt",
                "message_id": message_id,
                "status": "read",
                "timestamp": datetime.utcnow().isoformat()
            })

    async def mark_conversation_read(self, conversation_id: str, user_id: str):
        """
        Mark all messages in conversation as read (when user opens chat)
        """
        query = """
            UPDATE messages
            SET read_at = NOW(),
                delivered_at = COALESCE(delivered_at, NOW())
            WHERE receiver_id = $1
              AND (sender_id, receiver_id) IN (
                  SELECT user1, user2 FROM conversations WHERE conversation_id = $2
              )
              AND read_at IS NULL
            RETURNING message_id, sender_id
        """
        updated = await self.db.fetch(query, user_id, conversation_id)

        # Notify senders in batch
        for row in updated:
            await self.ws.send_to_user(row['sender_id'], {
                "type": "receipt",
                "message_id": row['message_id'],
                "status": "read"
            })
```

**Optimization: Batch read receipts**

```python
# Instead of sending individual receipt for each message,
# send one receipt with last_read_message_id

await self.ws.send_to_user(sender_id, {
    "type": "receipt_batch",
    "conversation_id": conversation_id,
    "last_read_message_id": last_message_id,
    "read_count": 15  # 15 messages marked as read
})
```

---

### 9.3 Offline Message Queue

**Store messages for offline users, deliver when online**

```python
class OfflineMessageQueue:
    """
    Queue messages for offline users using Redis Sorted Set
    """

    def __init__(self, redis_client):
        self.redis = redis_client

    async def enqueue_message(self, user_id: str, message: dict):
        """
        Add message to user's offline queue
        Sorted by timestamp for ordered delivery
        """
        queue_key = f"offline_queue:{user_id}"

        # Add to sorted set (score = timestamp)
        timestamp = datetime.utcnow().timestamp()
        await self.redis.zadd(
            queue_key,
            {json.dumps(message): timestamp}
        )

        # Set expiry on queue (30 days)
        await self.redis.expire(queue_key, 30 * 86400)

    async def dequeue_messages(self, user_id: str, limit: int = 100) -> list:
        """
        Retrieve messages for user who came online
        Return in chronological order
        """
        queue_key = f"offline_queue:{user_id}"

        # Get messages (oldest first)
        messages_json = await self.redis.zrange(queue_key, 0, limit - 1)

        # Parse messages
        messages = [json.loads(msg) for msg in messages_json]

        # Remove delivered messages
        if messages:
            await self.redis.zremrangebyrank(queue_key, 0, len(messages) - 1)

        return messages

    async def get_queue_size(self, user_id: str) -> int:
        """Get number of pending offline messages"""
        queue_key = f"offline_queue:{user_id}"
        return await self.redis.zcard(queue_key)

# Usage in WebSocket handler
async def on_user_connect(user_id: str):
    """When user connects, deliver offline messages"""
    offline_queue = OfflineMessageQueue(redis_client)

    # Get pending messages
    messages = await offline_queue.dequeue_messages(user_id)

    # Deliver messages
    for message in messages:
        await ws_server.send_to_user(user_id, message)

    print(f"Delivered {len(messages)} offline messages to {user_id}")
```

---

### 9.4 Message Reactions

**Support emoji reactions like Slack/WhatsApp**

```python
class MessageReactionService:
    """
    Support reactions on messages (👍, ❤️, 😂, etc.)
    """

    def __init__(self, db_pool, ws_server):
        self.db = db_pool
        self.ws = ws_server

    async def add_reaction(self, message_id: str, user_id: str, emoji: str):
        """
        Add emoji reaction to message
        """
        query = """
            INSERT INTO message_reactions (message_id, user_id, emoji, created_at)
            VALUES ($1, $2, $3, NOW())
            ON CONFLICT (message_id, user_id, emoji) DO NOTHING
            RETURNING reaction_id
        """
        reaction_id = await self.db.fetchval(query, message_id, user_id, emoji)

        if reaction_id:
            # Broadcast to conversation members
            await self._broadcast_reaction(message_id, user_id, emoji, action="added")

    async def remove_reaction(self, message_id: str, user_id: str, emoji: str):
        """
        Remove emoji reaction from message
        """
        query = """
            DELETE FROM message_reactions
            WHERE message_id = $1 AND user_id = $2 AND emoji = $3
        """
        await self.db.execute(query, message_id, user_id, emoji)

        # Broadcast removal
        await self._broadcast_reaction(message_id, user_id, emoji, action="removed")

    async def get_reactions(self, message_id: str) -> dict:
        """
        Get all reactions for a message
        Returns: {"👍": ["user1", "user2"], "❤️": ["user3"]}
        """
        query = """
            SELECT emoji, user_id
            FROM message_reactions
            WHERE message_id = $1
            ORDER BY created_at
        """
        rows = await self.db.fetch(query, message_id)

        # Group by emoji
        reactions = {}
        for row in rows:
            emoji = row['emoji']
            if emoji not in reactions:
                reactions[emoji] = []
            reactions[emoji].append(row['user_id'])

        return reactions

# Database schema
"""
CREATE TABLE message_reactions (
    reaction_id BIGSERIAL PRIMARY KEY,
    message_id BIGINT REFERENCES messages(message_id) ON DELETE CASCADE,
    user_id BIGINT REFERENCES users(user_id),
    emoji VARCHAR(10) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE (message_id, user_id, emoji)
);

CREATE INDEX idx_message_reactions ON message_reactions(message_id);
"""
```

---

## 10. Push Notifications

**Notify offline users via mobile push**

```python
import httpx
from typing import List

class PushNotificationService:
    """
    Send push notifications to offline users
    Using FCM (Firebase Cloud Messaging) for example
    """

    def __init__(self, fcm_server_key: str, db_pool):
        self.fcm_server_key = fcm_server_key
        self.db = db_pool
        self.fcm_url = "https://fcm.googleapis.com/fcm/send"

    async def send_message_notification(self, user_id: str, sender_name: str, message: str):
        """
        Send push notification for new message
        """
        # Get user's device tokens
        device_tokens = await self._get_device_tokens(user_id)

        if not device_tokens:
            return  # No devices to notify

        # Prepare notification payload
        notification = {
            "title": sender_name,
            "body": message[:100],  # Truncate long messages
            "click_action": "OPEN_CHAT",
            "sound": "default"
        }

        # Send to all devices
        for token in device_tokens:
            await self._send_fcm_notification(token, notification)

    async def _get_device_tokens(self, user_id: str) -> List[str]:
        """Get all registered device tokens for user"""
        query = """
            SELECT device_token FROM user_devices
            WHERE user_id = $1 AND push_enabled = true
        """
        rows = await self.db.fetch(query, user_id)
        return [row['device_token'] for row in rows]

    async def _send_fcm_notification(self, device_token: str, notification: dict):
        """Send notification via Firebase Cloud Messaging"""
        async with httpx.AsyncClient() as client:
            response = await client.post(
                self.fcm_url,
                headers={
                    "Authorization": f"key={self.fcm_server_key}",
                    "Content-Type": "application/json"
                },
                json={
                    "to": device_token,
                    "notification": notification,
                    "priority": "high"
                }
            )

            if response.status_code != 200:
                print(f"FCM error: {response.text}")

# Integration with message delivery
async def send_message_with_notification(sender_id, receiver_id, content):
    # Store message
    message_id = await message_service.send_message(sender_id, receiver_id, content)

    # Check if receiver is online
    is_online = await presence_service.is_online(receiver_id)

    if not is_online:
        # Send push notification
        sender = await get_user(sender_id)
        await push_service.send_message_notification(
            receiver_id,
            sender['name'],
            content
        )

    return message_id
```

---

## 11. Media Messages (Images, Files)

**Handle file uploads efficiently**

```python
import boto3
from io import BytesIO

class MediaMessageService:
    """
    Handle media messages (images, videos, files)
    Store in S3/CDN, send URL instead of content
    """

    def __init__(self, s3_client, db_pool):
        self.s3 = s3_client
        self.db = db_pool
        self.bucket_name = "chat-media-bucket"

    async def send_media_message(
        self,
        sender_id: str,
        receiver_id: str,
        file_content: bytes,
        file_type: str,
        filename: str
    ):
        """
        Send media message:
        1. Upload file to S3
        2. Store metadata in DB with S3 URL
        3. Send message with URL
        """
        # 1. Upload to S3
        file_key = f"media/{sender_id}/{uuid.uuid4()}/{filename}"
        s3_url = await self._upload_to_s3(file_content, file_key, file_type)

        # 2. Store message metadata
        query = """
            INSERT INTO messages (sender_id, receiver_id, message_type, media_url, media_filename, created_at)
            VALUES ($1, $2, $3, $4, $5, NOW())
            RETURNING message_id
        """
        message_id = await self.db.fetchval(
            query, sender_id, receiver_id, file_type, s3_url, filename
        )

        # 3. Send message with URL
        await ws_server.send_to_user(receiver_id, {
            "type": "media_message",
            "message_id": message_id,
            "sender_id": sender_id,
            "media_type": file_type,
            "media_url": s3_url,
            "filename": filename,
            "thumbnail_url": await self._generate_thumbnail(file_content, file_type) if file_type.startswith("image/") else None
        })

        return message_id

    async def _upload_to_s3(self, file_content: bytes, file_key: str, content_type: str) -> str:
        """Upload file to S3 and return URL"""
        self.s3.put_object(
            Bucket=self.bucket_name,
            Key=file_key,
            Body=file_content,
            ContentType=content_type
        )

        # Return presigned URL (valid for 7 days)
        url = self.s3.generate_presigned_url(
            'get_object',
            Params={'Bucket': self.bucket_name, 'Key': file_key},
            ExpiresIn=7 * 86400
        )

        return url

    async def _generate_thumbnail(self, image_content: bytes, content_type: str) -> str:
        """Generate thumbnail for image (optional optimization)"""
        from PIL import Image

        # Resize image to thumbnail
        image = Image.open(BytesIO(image_content))
        image.thumbnail((200, 200))

        # Upload thumbnail
        thumb_buffer = BytesIO()
        image.save(thumb_buffer, format='JPEG')
        thumb_buffer.seek(0)

        thumb_key = f"thumbnails/{uuid.uuid4()}.jpg"
        return await self._upload_to_s3(thumb_buffer.read(), thumb_key, "image/jpeg")

# Database schema update
"""
ALTER TABLE messages ADD COLUMN message_type VARCHAR(20) DEFAULT 'text';
ALTER TABLE messages ADD COLUMN media_url TEXT;
ALTER TABLE messages ADD COLUMN media_filename VARCHAR(255);
ALTER TABLE messages ADD COLUMN thumbnail_url TEXT;
"""
```

---

## 12. Scaling WebSocket Servers

### Service Discovery Pattern

**Problem:** Route messages between WebSocket servers

```python
import etcd3

class WebSocketServiceDiscovery:
    """
    Register WebSocket servers with etcd for service discovery
    """

    def __init__(self, etcd_client, server_id, server_host):
        self.etcd = etcd_client
        self.server_id = server_id
        self.server_host = server_host

    async def register(self):
        """
        Register this server in etcd with TTL
        Automatically renew lease (heartbeat)
        """
        lease = self.etcd.lease(ttl=10)

        # Store server info: ws_servers/{server_id} -> host:port
        self.etcd.put(
            f"ws_servers/{self.server_id}",
            self.server_host,
            lease=lease
        )

        # Keep lease alive
        asyncio.create_task(self._keep_alive(lease))

    async def _keep_alive(self, lease):
        """Keep etcd lease alive"""
        while True:
            await asyncio.sleep(5)
            lease.refresh()

    async def find_user_server(self, user_id: str) -> str:
        """
        Find which WebSocket server a user is connected to
        """
        # Check Redis for user's server
        server_id = await redis_client.get(f"user_location:{user_id}")

        if not server_id:
            return None

        # Look up server host in etcd
        server_host = self.etcd.get(f"ws_servers/{server_id.decode()}")
        return server_host[0].decode() if server_host[0] else None

# When user connects
async def on_user_connect(user_id: str, ws_server_id: str):
    """Track which server user is connected to"""
    await redis_client.setex(
        f"user_location:{user_id}",
        3600,  # 1 hour TTL
        ws_server_id
    )
```

### Cross-Server Message Routing

```python
class CrossServerRouter:
    """
    Route messages between WebSocket servers using pub/sub
    """

    def __init__(self, redis_client, ws_server):
        self.redis = redis_client
        self.ws = ws_server

    async def start_subscriber(self, server_id: str):
        """
        Subscribe to messages for this server
        """
        pubsub = self.redis.pubsub()
        await pubsub.subscribe(f"ws_server:{server_id}")

        async for message in pubsub.listen():
            if message['type'] == 'message':
                data = json.loads(message['data'])
                await self._deliver_local(data)

    async def _deliver_local(self, message: dict):
        """Deliver message to local WebSocket connection"""
        user_id = message['receiver_id']
        await self.ws.send_to_user(user_id, message)

    async def route_message(self, user_id: str, message: dict):
        """
        Route message to correct WebSocket server
        """
        # Find user's server
        server_id = await redis_client.get(f"user_location:{user_id}")

        if not server_id:
            # User offline - queue message
            await offline_queue.enqueue_message(user_id, message)
            return

        # Publish to user's server
        await self.redis.publish(
            f"ws_server:{server_id.decode()}",
            json.dumps(message)
        )
```

---

## 13. Summary

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

**Trade-offs:**
- WebSockets vs Long Polling (chose WebSockets for real-time)
- Synchronous vs Asynchronous delivery (async for scalability)
- Strong vs Eventual consistency (eventual for availability)

---

## 14. Trade-offs & Design Decisions

### 1. WebSockets vs Alternatives

| Approach | Pros | Cons | Use Case |
|----------|------|------|----------|
| **HTTP Long Polling** | Simple, works everywhere | High latency, inefficient | Legacy browsers |
| **Server-Sent Events (SSE)** | Simpler than WebSocket, HTTP-based | One-way (server→client) | Notifications only |
| **WebSockets** | Bidirectional, low latency, efficient | More complex, stateful | Real-time chat ✅ |
| **HTTP/2 Push** | Multiplexing, header compression | Limited browser support | Modern apps |

**Decision: WebSockets** for full-duplex real-time communication

---

### 2. Message Queue: Kafka vs RabbitMQ vs Redis Pub/Sub

| Feature | Kafka | RabbitMQ | Redis Pub/Sub |
|---------|-------|----------|---------------|
| **Persistence** | ✅ Durable | ✅ Durable | ❌ In-memory |
| **Throughput** | Very high (millions/sec) | High (hundreds of K/sec) | Very high |
| **Ordering** | ✅ Per partition | ⚠️ With single consumer | ❌ No guarantee |
| **Replay** | ✅ Yes | ❌ No | ❌ No |
| **Complexity** | High | Medium | Low |

**Decision:**
- **Kafka** for message persistence and high throughput
- **Redis Pub/Sub** for ephemeral events (typing indicators, presence)

---

### 3. Database: SQL vs NoSQL

**PostgreSQL (SQL):**
- ✅ ACID transactions
- ✅ Rich queries (search chat history)
- ✅ Joins (user + messages)
- ❌ Harder to shard

**Cassandra (NoSQL):**
- ✅ Easy horizontal scaling
- ✅ High write throughput
- ✅ Natural partitioning by user_id
- ❌ No joins, limited queries

**Decision:** **PostgreSQL with read replicas** for simplicity, migrate to Cassandra at massive scale

---

### 4. Consistency vs Availability (CAP Theorem)

**Scenario:** User sends message to group

**Strong Consistency (CP):**
```python
# Wait for all members to ACK before returning success
# ❌ Slow, low availability
# ✅ Guaranteed delivery
```

**Eventual Consistency (AP):**
```python
# Return success immediately, deliver asynchronously
# ✅ Fast, high availability
# ⚠️ Messages may arrive out of order temporarily
```

**Decision:** **Eventual consistency (AP)** - Messages delivered at-least-once, idempotency handles duplicates

---

## 15. Interview Tips & Common Questions

### Q1: "How do you handle message delivery when both users are online?"

**Answer:**
"When both users are online:
1. Sender sends message via WebSocket to Server A
2. Server A stores message in database immediately
3. Server A publishes to message queue (Kafka topic)
4. If receiver is on same server, deliver directly via WebSocket
5. If receiver is on different server (Server B), route via Redis Pub/Sub
6. Server B delivers to receiver via WebSocket
7. Send delivery receipt back to sender

Key: Store-first ensures durability, async delivery ensures low latency."

### Q2: "What happens if a message fails to deliver?"

**Answer:**
"Multi-layered approach:
1. **At-rest durability**: Message is stored in database before delivery attempt
2. **Offline queue**: If user is offline, message goes to Redis sorted set
3. **Retry mechanism**: Delivery failures trigger exponential backoff retries
4. **Dead letter queue**: After max retries, move to DLQ for manual investigation
5. **Push notification**: If online delivery fails, send mobile push as fallback

This ensures at-least-once delivery guarantee."

### Q3: "How do you ensure message ordering in a distributed system?"

**Answer:**
"Message ordering challenges exist at multiple levels:

**Within a conversation:**
- Use sequence numbers per conversation (Redis INCR)
- Client displays messages sorted by sequence number
- If message N+1 arrives before N, client buffers and waits

**Across servers with clock skew:**
- Use Lamport timestamps or vector clocks for causal ordering
- Don't rely solely on server timestamps due to clock drift

**In group chats:**
- Total ordering not required (unlike transactions)
- Per-sender ordering is sufficient
- Display timestamps for user context"

### Q4: "How would you design end-to-end encryption?"

**Answer:**
"Signal Protocol approach:
1. **Key exchange**: Each user has identity key pair (public/private)
2. **Session keys**: Establish session keys using Double Ratchet algorithm
3. **Message encryption**: Encrypt message on sender's device before sending
4. **Server role**: Server only routes encrypted blobs, cannot read content
5. **Key management**: Store keys on device, use secure enclave

Trade-off: Server can't do:
- Message search
- Content moderation
- Read receipts (without metadata leakage)

But gains: Privacy, security against server compromise"

### Q5: "How do you handle large group chats (10K+ members)?"

**Answer:**
"Large groups require different approach:

**Fanout optimization:**
1. **Batch fanout**: Group members into batches, deliver in waves
2. **Lazy delivery**: Only deliver to active members immediately
3. **Read replicas**: Distribute reads across multiple DB replicas

**Architecture changes:**
1. **Broadcast channels**: For 10K+ members, switch to pub/sub model
2. **No typing indicators**: Too expensive to broadcast to everyone
3. **Sampled read receipts**: Only show 'read by 500+', not individual names
4. **Message workers**: Dedicated workers for large group fanout

**Example: Telegram channels:**
- One-way broadcast (no replies)
- Optimized for massive scale (millions of subscribers)
- Different tech stack than regular chats"

---

## 16. Performance Optimization Checklist

### Database Optimizations

- [ ] Index on `(sender_id, receiver_id, created_at)` for chat history
- [ ] Index on `receiver_id` for unread message count
- [ ] Partition messages table by date (monthly/yearly)
- [ ] Use read replicas for message history queries
- [ ] Archive old messages (>1 year) to cold storage (S3)

### Caching Strategy

- [ ] Cache recent messages (last 50) in Redis per conversation
- [ ] Cache user online status in Redis (30s TTL)
- [ ] Cache group member lists in Redis
- [ ] Cache unread message counts per user

### WebSocket Optimizations

- [ ] Connection pooling for database queries
- [ ] Message batching (send multiple messages in one frame)
- [ ] Compression (WebSocket permessage-deflate extension)
- [ ] Heartbeat/ping-pong to detect dead connections

### Monitoring & Metrics

```python
# Key metrics to track:
- WebSocket connections count (gauge)
- Messages sent per second (counter)
- Message delivery latency p50/p99 (histogram)
- Failed deliveries (counter)
- Offline queue size per user (gauge)
- Database query latency (histogram)
- WebSocket reconnections (counter)
```

---

## 17. Production Checklist

**Before Launch:**
- [ ] Load testing (10K concurrent WebSocket connections per server)
- [ ] Chaos testing (kill random servers, ensure graceful degradation)
- [ ] Message delivery tests (online, offline, reconnection scenarios)
- [ ] Database failover testing
- [ ] Idempotency testing (retry message sends)
- [ ] Group chat testing (100, 1000, 10000 members)
- [ ] Media upload/download testing (images, large files)
- [ ] Push notification testing (iOS, Android)
- [ ] Monitoring dashboards (Grafana)
- [ ] Alerting rules (PagerDuty)
- [ ] Runbook for on-call engineers

**Capacity Planning:**
- Provision for 3x peak load
- Database: 10K writes/sec, 50K reads/sec
- WebSocket servers: 10K connections each
- Redis: <1ms latency, 100K ops/sec
- Kafka: 1M messages/sec throughput

**Next Steps:**
- Learn about [Notification System](../notification-system/README.md)
- Study [Collaborative Docs](../collaborative-docs/README.md) with OT/CRDT
- Explore [Video Streaming](../../advanced/video-streaming/README.md)
