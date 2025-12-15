# Message Queue Patterns

## Overview

Message queues enable asynchronous communication between services in distributed systems. They decouple producers (senders) from consumers (receivers), providing reliability, scalability, and fault tolerance.

**Key Benefits:**
- **Decoupling:** Services don't need to know about each other
- **Reliability:** Messages persist until processed
- **Scalability:** Add more consumers to handle load
- **Fault Tolerance:** Messages survive service crashes

---

## Common Message Queue Patterns

### 1. Point-to-Point (Queue) Pattern

**How It Works:**
- One message → One consumer
- Messages are consumed and removed from queue
- Multiple consumers compete for messages (load balancing)

**Visual:**
```
Producer → [Queue] → Consumer 1
                  → Consumer 2  (whichever gets it first)
                  → Consumer 3
```

**Use Cases:**
- Task processing (background jobs)
- Order processing
- Email sending
- Image processing

**Example: Order Processing**
```python
# Producer (Web Server)
order_queue.send({
    "order_id": "12345",
    "user_id": "user_789",
    "items": [...],
    "total": 99.99
})

# Consumer (Order Processor)
message = order_queue.receive()
process_order(message.body)
message.ack()  # Remove from queue
```

**Characteristics:**
- ✅ Load balancing across consumers
- ✅ Each message processed exactly once
- ✅ Simple and predictable
- ❌ No message broadcasting

---

### 2. Publish-Subscribe (Pub/Sub) Pattern

**How It Works:**
- One message → Multiple consumers
- Each subscriber gets a copy of every message
- Publishers don't know about subscribers

**Visual:**
```
Publisher → [Topic] → Subscriber 1 (gets copy)
                   → Subscriber 2 (gets copy)
                   → Subscriber 3 (gets copy)
```

**Use Cases:**
- Event notifications
- Real-time updates (stock prices, sports scores)
- Log aggregation
- Broadcasting system events

**Example: User Signup Event**
```python
# Publisher (Auth Service)
event_bus.publish("user.signup", {
    "user_id": "user_123",
    "email": "alice@example.com",
    "timestamp": "2024-01-15T10:30:00Z"
})

# Subscriber 1: Email Service (sends welcome email)
@subscribe("user.signup")
def send_welcome_email(event):
    send_email(event['email'], "Welcome!")

# Subscriber 2: Analytics Service (tracks signup)
@subscribe("user.signup")
def track_signup(event):
    analytics.track("user_signup", event)

# Subscriber 3: CRM Service (creates customer record)
@subscribe("user.signup")
def create_crm_record(event):
    crm.create_customer(event)
```

**Characteristics:**
- ✅ Multiple consumers per message
- ✅ Decoupled services
- ✅ Easy to add new subscribers
- ❌ Higher message volume (duplicates)

---

### 3. Request-Reply Pattern

**How It Works:**
- Producer sends request to queue
- Consumer processes and sends reply to another queue
- Producer waits for reply (correlation ID links them)

**Visual:**
```
Client → [Request Queue] → Worker
  ↑                           ↓
  └──── [Reply Queue] ←───────┘
       (correlation_id matches)
```

**Use Cases:**
- RPC (Remote Procedure Call) over message queue
- Distributed transactions
- Query services
- Microservice communication

**Example: Price Calculation**
```python
import uuid

# Client (sends request)
correlation_id = str(uuid.uuid4())
request_queue.send({
    "correlation_id": correlation_id,
    "reply_to": "price_replies",
    "cart_items": [...]
})

# Wait for reply
reply = reply_queue.receive(
    filter=f"correlation_id = '{correlation_id}'",
    timeout=5000  # 5 seconds
)
total_price = reply.body['total']

# Worker (processes and replies)
request = request_queue.receive()
total = calculate_price(request.body['cart_items'])

reply_queue.send({
    "correlation_id": request.body['correlation_id'],
    "total": total
}, queue=request.body['reply_to'])
```

**Characteristics:**
- ✅ Synchronous-like behavior over async queues
- ✅ Can handle slow operations
- ❌ More complex (correlation tracking)
- ❌ Timeout handling needed

---

### 4. Priority Queue Pattern

**How It Works:**
- Messages have priority levels (high, medium, low)
- High-priority messages processed first
- Useful for SLA-driven processing

**Visual:**
```
Producer → [Priority Queue]
              ├─ High Priority (processed first)
              ├─ Medium Priority
              └─ Low Priority
```

**Use Cases:**
- Customer support tickets (VIP users first)
- Order processing (express shipping first)
- Alerts and notifications
- Task scheduling

**Example: Support Tickets**
```python
# VIP customer ticket (high priority)
ticket_queue.send({
    "ticket_id": "T-12345",
    "user_tier": "VIP",
    "issue": "Payment failed"
}, priority=9)

# Regular customer (normal priority)
ticket_queue.send({
    "ticket_id": "T-12346",
    "user_tier": "Free",
    "issue": "Feature request"
}, priority=5)

# Consumer processes high-priority first
message = ticket_queue.receive()  # Gets T-12345 first
```

**Implementation Tips:**
- Use separate queues per priority level
- Or use message priority attribute (if supported)
- Monitor queue depths to prevent starvation

**Characteristics:**
- ✅ SLA-driven processing
- ✅ Important messages processed faster
- ❌ Risk of low-priority starvation
- ❌ More complex queue management

---

### 5. Dead Letter Queue (DLQ) Pattern

**How It Works:**
- Failed messages moved to separate "dead letter" queue
- Prevents poison messages from blocking processing
- Allows investigation and manual retry

**Visual:**
```
Producer → [Main Queue] → Consumer
                ↓ (failed N times)
           [Dead Letter Queue]
                ↓
           Manual Investigation
```

**Use Cases:**
- Error handling
- Failed payment processing
- Invalid message format
- Service unavailability

**Example: Order Processing with DLQ**
```python
# Consumer with retry logic
MAX_RETRIES = 3

message = order_queue.receive()
retry_count = message.attributes.get('retry_count', 0)

try:
    process_order(message.body)
    message.ack()
except Exception as e:
    if retry_count < MAX_RETRIES:
        # Retry: Put back in queue with incremented counter
        order_queue.send(
            message.body,
            attributes={'retry_count': retry_count + 1},
            delay_seconds=60 * (2 ** retry_count)  # Exponential backoff
        )
    else:
        # Move to DLQ after max retries
        dead_letter_queue.send({
            "original_message": message.body,
            "error": str(e),
            "retry_count": retry_count,
            "timestamp": now()
        })

    message.ack()  # Remove from main queue
```

**Monitoring:**
```python
# Alert if DLQ has messages
dlq_size = dead_letter_queue.get_message_count()
if dlq_size > 10:
    alert("Dead Letter Queue has {} messages".format(dlq_size))
```

**Characteristics:**
- ✅ Prevents poison messages
- ✅ Preserves failed messages for analysis
- ✅ Unblocks queue processing
- ❌ Requires monitoring and manual intervention

---

### 6. Competing Consumers Pattern

**How It Works:**
- Multiple consumers read from same queue
- Each message processed by only one consumer
- Scales horizontally to handle load

**Visual:**
```
Producer → [Queue] ⇄ Consumer 1 (idle)
                  ⇄ Consumer 2 (processing msg 1)
                  ⇄ Consumer 3 (processing msg 2)
                  ⇄ Consumer 4 (idle)
```

**Use Cases:**
- Background job processing
- Image resizing
- Video transcoding
- Batch processing

**Example: Image Processing**
```python
# Producer (uploads)
for image_path in uploaded_images:
    image_queue.send({
        "image_path": image_path,
        "user_id": user_id
    })

# Multiple consumers (workers)
# Worker 1, Worker 2, Worker 3, etc.
while True:
    message = image_queue.receive()

    # Process image
    image = load_image(message.body['image_path'])
    resized = resize_image(image)
    save_image(resized)

    message.ack()
```

**Scaling:**
```bash
# Start multiple workers
docker-compose up --scale image-worker=10
```

**Characteristics:**
- ✅ Horizontal scalability
- ✅ Load balancing automatic
- ✅ Fault tolerance (workers can fail)
- ❌ Message ordering not guaranteed

---

### 7. Message Filtering Pattern

**How It Works:**
- Consumers subscribe with filters/selectors
- Only receive messages matching criteria
- Reduces unnecessary message processing

**Visual:**
```
Publisher → [Topic]
              ├─ Filter: region=US → Consumer A
              ├─ Filter: region=EU → Consumer B
              └─ Filter: priority=high → Consumer C
```

**Use Cases:**
- Regional processing
- Customer segmentation
- Event filtering
- Selective notifications

**Example: Regional Order Processing**
```python
# Publisher
order_bus.publish("orders", {
    "order_id": "O-123",
    "region": "US",
    "amount": 99.99
}, attributes={"region": "US"})

# Consumer 1: US orders only
@subscribe("orders", filter="region = 'US'")
def process_us_order(order):
    # Process with US tax rules
    pass

# Consumer 2: EU orders only
@subscribe("orders", filter="region = 'EU'")
def process_eu_order(order):
    # Process with EU tax rules (VAT)
    pass
```

**SQL-like Filters (AWS SQS, SNS):**
```python
subscription.set_filter_policy({
    "region": ["US", "CA"],
    "order_value": [{"numeric": [">", 100]}],
    "customer_type": ["VIP"]
})
```

**Characteristics:**
- ✅ Reduced network traffic
- ✅ Consumer-side filtering
- ✅ Flexible subscription rules
- ❌ Filter complexity limits

---

### 8. Saga Pattern (Distributed Transactions)

**How It Works:**
- Break distributed transaction into local transactions
- Each step publishes event to trigger next step
- Compensating transactions for rollback

**Visual:**
```
Order Service → Payment Service → Inventory Service → Shipping Service
     ↓               ↓                  ↓                   ↓
  [Reserve]      [Charge]          [Allocate]          [Ship]
     ↓ (fail)        ↓                  ↓                   ↓
[Compensate] ← [Refund] ← [Deallocate] ← [Cancel]
```

**Use Cases:**
- E-commerce order flow
- Travel booking
- Financial transactions
- Multi-service workflows

**Example: E-commerce Order Saga**
```python
# Step 1: Create Order
order = create_order(cart)
event_bus.publish("order.created", order)

# Step 2: Payment Service
@subscribe("order.created")
def charge_payment(event):
    try:
        payment = charge_card(event['order_id'], event['total'])
        event_bus.publish("payment.succeeded", {
            "order_id": event['order_id'],
            "payment_id": payment.id
        })
    except PaymentError as e:
        event_bus.publish("payment.failed", {
            "order_id": event['order_id'],
            "reason": str(e)
        })

# Step 3: Inventory Service
@subscribe("payment.succeeded")
def allocate_inventory(event):
    try:
        allocate_items(event['order_id'])
        event_bus.publish("inventory.allocated", event)
    except OutOfStockError:
        # Compensate: refund payment
        event_bus.publish("inventory.failed", event)

# Compensation: Refund
@subscribe("inventory.failed")
def refund_payment(event):
    refund(event['payment_id'])
    event_bus.publish("order.cancelled", event)
```

**Characteristics:**
- ✅ No distributed locks needed
- ✅ Services remain independent
- ✅ Eventual consistency
- ❌ Complex compensation logic
- ❌ Hard to debug failures

---

### 9. Event Sourcing Pattern

**How It Works:**
- Store all state changes as sequence of events
- Rebuild state by replaying events
- Event log is source of truth

**Visual:**
```
Commands → [Event Store]
             ├─ Event 1: Account Created
             ├─ Event 2: $100 Deposited
             ├─ Event 3: $30 Withdrawn
             └─ Event 4: $50 Deposited
                    ↓
          Replay → Current State: $120
```

**Use Cases:**
- Audit trails
- Banking/financial systems
- Undo/redo functionality
- Time-travel debugging

**Example: Bank Account**
```python
# Event Store
events = [
    {"type": "AccountCreated", "account_id": "A-123", "balance": 0},
    {"type": "MoneyDeposited", "account_id": "A-123", "amount": 100},
    {"type": "MoneyWithdrawn", "account_id": "A-123", "amount": 30},
    {"type": "MoneyDeposited", "account_id": "A-123", "amount": 50}
]

# Rebuild state by replaying events
def get_balance(account_id):
    balance = 0
    for event in events:
        if event['account_id'] != account_id:
            continue

        if event['type'] == 'AccountCreated':
            balance = event['balance']
        elif event['type'] == 'MoneyDeposited':
            balance += event['amount']
        elif event['type'] == 'MoneyWithdrawn':
            balance -= event['amount']

    return balance

# Result: 120
```

**Snapshots (Optimization):**
```python
# Instead of replaying 1 million events, store snapshot every 1000 events
snapshot = {"balance": 5000, "event_sequence": 1000}

# Replay only from snapshot
def get_balance_optimized(account_id):
    snapshot = load_snapshot(account_id)
    balance = snapshot['balance']

    # Replay only events after snapshot
    events = load_events_after(account_id, snapshot['event_sequence'])
    for event in events:
        # Apply events...
        pass

    return balance
```

**Characteristics:**
- ✅ Complete audit trail
- ✅ Time-travel queries
- ✅ Event replay for testing
- ❌ Storage overhead (all events)
- ❌ Query complexity increases

---

### 10. CQRS (Command Query Responsibility Segregation)

**How It Works:**
- Separate read and write models
- Commands update write model → publish events
- Events update read model (materialized views)

**Visual:**
```
Command (Write) → [Write DB] → Events → [Read DB 1] (optimized for queries)
                                      → [Read DB 2] (analytics)
                                      → [Read DB 3] (search)
```

**Use Cases:**
- High read/write ratio systems
- Complex reporting requirements
- Different consistency requirements
- Performance optimization

**Example: E-commerce Product**
```python
# Write Model (Commands)
class ProductWriteModel:
    def create_product(self, name, price):
        product = Product(id=uuid4(), name=name, price=price)
        db.save(product)

        # Publish event
        event_bus.publish("product.created", {
            "product_id": product.id,
            "name": name,
            "price": price
        })

# Read Model (Queries)
class ProductReadModel:
    def __init__(self):
        # Optimized read database (denormalized)
        self.products = {}

    # Update read model when event received
    @subscribe("product.created")
    def on_product_created(self, event):
        self.products[event['product_id']] = {
            "name": event['name'],
            "price": event['price'],
            "reviews_count": 0,  # Precomputed
            "average_rating": 0  # Precomputed
        }

    # Fast queries
    def get_product(self, product_id):
        return self.products[product_id]

    def search_products(self, query):
        # Use ElasticSearch or similar for fast search
        return search_index.search(query)
```

**Characteristics:**
- ✅ Optimized read and write paths
- ✅ Scalable independently
- ✅ Flexible read models
- ❌ Eventual consistency
- ❌ Increased complexity

---

## Message Queue Guarantees

### At-Most-Once Delivery
- Message delivered 0 or 1 times
- Fast but can lose messages
- **Use case:** Metrics, logs (lossy is acceptable)

### At-Least-Once Delivery (Most Common)
- Message delivered 1 or more times
- Reliable but can duplicate
- **Use case:** Most applications (idempotent processing)

### Exactly-Once Delivery
- Message delivered exactly once
- Hardest to implement, slower
- **Use case:** Financial transactions, billing

**Example: Idempotent Consumer (At-Least-Once)**
```python
# Track processed message IDs to avoid duplicates
processed_ids = set()

while True:
    message = queue.receive()
    message_id = message.attributes['message_id']

    if message_id in processed_ids:
        # Duplicate - skip processing
        message.ack()
        continue

    # Process message
    process(message.body)

    # Mark as processed
    processed_ids.add(message_id)
    message.ack()
```

---

## Best Practices

### 1. Message Design
- **Keep messages small** (< 256 KB)
- **Use JSON or Protocol Buffers** for serialization
- **Include metadata** (timestamp, version, correlation ID)
- **Make messages self-contained** (no external dependencies)

### 2. Error Handling
- **Implement retries** with exponential backoff
- **Use Dead Letter Queues** for poison messages
- **Log all errors** with full context
- **Monitor DLQ depth**

### 3. Performance
- **Batch messages** when possible
- **Use message prefetching** (receive multiple at once)
- **Tune visibility timeout** (how long message is hidden)
- **Compress large messages**

### 4. Reliability
- **Acknowledge only after processing**
- **Use transactions** for critical flows
- **Implement circuit breakers** for downstream services
- **Set appropriate TTL** (time-to-live) for messages

---

## Comparison Table

| Pattern | Use Case | Pros | Cons |
|---------|----------|------|------|
| **Point-to-Point** | Task processing | Simple, load balancing | No broadcasting |
| **Pub/Sub** | Event notifications | Decoupled, broadcast | Message duplication |
| **Request-Reply** | RPC-style calls | Synchronous-like | Complex correlation |
| **Priority Queue** | SLA-driven processing | Important tasks first | Starvation risk |
| **Dead Letter Queue** | Error handling | Unblocks processing | Manual intervention |
| **Competing Consumers** | Scaling | Horizontal scaling | No ordering |
| **Message Filtering** | Selective processing | Reduced traffic | Filter complexity |
| **Saga** | Distributed transactions | No locks needed | Complex compensation |
| **Event Sourcing** | Audit trails | Complete history | Storage overhead |
| **CQRS** | Read/write optimization | Scalable | Eventual consistency |

---

## Summary

**Key Takeaways:**
- Message queues decouple services and improve reliability
- Choose pattern based on use case (broadcasting vs task processing)
- Always implement error handling (DLQ, retries)
- Consider delivery guarantees (at-least-once vs exactly-once)
- Monitor queue depths and processing lag

**Interview Tips:**
- Explain why you'd use a queue (decoupling, scalability)
- Discuss trade-offs (ordering vs scalability)
- Mention specific technologies (Kafka, RabbitMQ, SQS)
- Cover error handling and monitoring

---

**Next:** Learn about specific [Message Queue Technologies](kafka-rabbitmq-sqs.md) and their trade-offs.
