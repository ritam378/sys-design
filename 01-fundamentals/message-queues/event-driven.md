# Event-Driven Architecture (EDA)

## Overview

Event-Driven Architecture (EDA) is a software design pattern where the flow of the program is determined by **events** - changes in state that are significant to the business.

**Core Principle:** Services react to events asynchronously rather than calling each other directly.

**Simple Example:**
```
Traditional (Synchronous):
User clicks "Buy" → Order Service → calls Payment Service → calls Inventory Service

Event-Driven (Asynchronous):
User clicks "Buy" → Order Service emits "OrderCreated" event
                  → Payment Service listens and processes payment
                  → Inventory Service listens and reserves items
```

---

## Key Concepts

### 1. Events
An **event** is a significant change in state.

**Characteristics:**
- **Immutable** - What happened in the past can't change
- **Timestamped** - When did it happen
- **Contains data** - What changed
- **Named in past tense** - `OrderPlaced`, `UserSignedUp`, `PaymentCompleted`

**Event Structure:**
```json
{
  "event_id": "evt_123abc",
  "event_type": "OrderPlaced",
  "event_version": "1.0",
  "timestamp": "2024-01-15T10:30:00Z",
  "aggregate_id": "order_456",
  "data": {
    "order_id": "order_456",
    "user_id": "user_789",
    "total_amount": 99.99,
    "items": [...]
  },
  "metadata": {
    "user_ip": "192.168.1.1",
    "user_agent": "Mozilla/5.0..."
  }
}
```

### 2. Event Producers
Services that **generate events** when something happens.

```python
class OrderService:
    def place_order(self, order_data):
        # Business logic
        order = Order.create(order_data)

        # Emit event
        event_bus.publish('OrderPlaced', {
            'order_id': order.id,
            'user_id': order.user_id,
            'total': order.total,
            'items': order.items
        })

        return order
```

### 3. Event Consumers
Services that **listen for and react to events**.

```python
@subscribe('OrderPlaced')
def handle_order_placed(event):
    # React to event
    send_confirmation_email(event['user_id'], event['order_id'])

@subscribe('OrderPlaced')
def reserve_inventory(event):
    for item in event['items']:
        inventory.reserve(item['product_id'], item['quantity'])
```

### 4. Event Bus / Event Broker
Infrastructure that **routes events** from producers to consumers.

**Technologies:**
- Kafka
- RabbitMQ
- AWS EventBridge
- Azure Event Grid
- Google Cloud Pub/Sub

---

## EDA Patterns

### 1. Event Notification

**What:** Notify other systems that something happened.

**Use Case:** Trigger side effects in other services.

```python
# E-commerce example
class UserService:
    def register_user(self, email, password):
        user = User.create(email, password)

        # Notify other services
        events.publish('UserRegistered', {
            'user_id': user.id,
            'email': user.email,
            'registered_at': datetime.now()
        })

# Email Service (consumer)
@subscribe('UserRegistered')
def send_welcome_email(event):
    email.send(event['email'], 'Welcome!', template='welcome')

# Analytics Service (consumer)
@subscribe('UserRegistered')
def track_registration(event):
    analytics.track('user_registered', event)

# Loyalty Service (consumer)
@subscribe('UserRegistered')
def create_loyalty_account(event):
    loyalty.create_account(event['user_id'], initial_points=100)
```

**Pros:**
- ✅ Decoupled services
- ✅ Easy to add new consumers
- ✅ Scalable

**Cons:**
- ❌ Event contains minimal data (consumers might need to fetch more)
- ❌ Eventual consistency

---

### 2. Event-Carried State Transfer

**What:** Event contains all the data consumers need (no additional lookups).

**Use Case:** Reduce coupling and database load.

```python
# Producer includes full state
class OrderService:
    def ship_order(self, order_id):
        order = Order.find(order_id)
        order.status = 'shipped'
        order.save()

        # Include all relevant data
        events.publish('OrderShipped', {
            'order_id': order.id,
            'user_id': order.user_id,
            'user_email': order.user.email,  # Include denormalized data
            'user_name': order.user.name,
            'items': [
                {'name': item.product.name, 'quantity': item.quantity}
                for item in order.items
            ],
            'shipping_address': order.shipping_address.to_dict(),
            'tracking_number': order.tracking_number
        })

# Consumer doesn't need to call Order Service API
@subscribe('OrderShipped')
def send_shipping_notification(event):
    # All data is in the event
    email.send(
        to=event['user_email'],
        subject='Your order has shipped!',
        body=f"Hi {event['user_name']}, your order {event['order_id']} "
             f"has shipped. Tracking: {event['tracking_number']}"
    )
    # No API call needed!
```

**Pros:**
- ✅ No additional API calls
- ✅ Lower latency
- ✅ Resilient (no cascading failures)

**Cons:**
- ❌ Larger event payloads
- ❌ Data duplication
- ❌ Consumers may have stale data

---

### 3. Event Sourcing

**What:** Store all state changes as a sequence of events.

**Use Case:** Audit trail, time travel, debugging.

```python
# Event Store
class AccountEventStore:
    def __init__(self):
        self.events = []

    def save_event(self, event):
        self.events.append(event)

    def get_events(self, account_id):
        return [e for e in self.events if e['account_id'] == account_id]

# Account aggregate
class Account:
    def __init__(self, account_id):
        self.account_id = account_id
        self.balance = 0
        self.events = []

    def deposit(self, amount):
        event = {
            'event_type': 'MoneyDeposited',
            'account_id': self.account_id,
            'amount': amount,
            'timestamp': datetime.now()
        }
        self.apply_event(event)
        event_store.save_event(event)

    def withdraw(self, amount):
        if self.balance < amount:
            raise InsufficientFundsError()

        event = {
            'event_type': 'MoneyWithdrawn',
            'account_id': self.account_id,
            'amount': amount,
            'timestamp': datetime.now()
        }
        self.apply_event(event)
        event_store.save_event(event)

    def apply_event(self, event):
        if event['event_type'] == 'MoneyDeposited':
            self.balance += event['amount']
        elif event['event_type'] == 'MoneyWithdrawn':
            self.balance -= event['amount']
        self.events.append(event)

    @classmethod
    def rebuild_from_events(cls, account_id):
        account = cls(account_id)
        events = event_store.get_events(account_id)
        for event in events:
            account.apply_event(event)
        return account

# Usage
account = Account('acc_123')
account.deposit(100)
account.withdraw(30)
account.deposit(50)

print(account.balance)  # 120

# Rebuild from events (e.g., after server restart)
rebuilt = Account.rebuild_from_events('acc_123')
print(rebuilt.balance)  # 120
```

**Pros:**
- ✅ Complete audit trail
- ✅ Can rebuild state at any point in time
- ✅ Supports temporal queries ("What was balance on Dec 1st?")
- ✅ Great for debugging

**Cons:**
- ❌ Complex queries (need to replay events)
- ❌ Storage overhead
- ❌ Eventual consistency

---

### 4. CQRS (Command Query Responsibility Segregation)

**What:** Separate read and write models.

**Use Case:** Optimize reads and writes independently.

```python
# Write Model (Commands)
class OrderCommandHandler:
    def handle_create_order(self, command):
        order = Order.create(command.data)
        event_store.save(Event('OrderCreated', order.to_dict()))
        event_bus.publish('OrderCreated', order.to_dict())

# Read Model (Queries)
class OrderQueryModel:
    def __init__(self):
        self.orders_by_user = {}  # Denormalized for fast queries
        self.orders_by_status = {}

    @subscribe('OrderCreated')
    def on_order_created(self, event):
        order = event['data']

        # Update denormalized read models
        if order['user_id'] not in self.orders_by_user:
            self.orders_by_user[order['user_id']] = []
        self.orders_by_user[order['user_id']].append(order)

        if order['status'] not in self.orders_by_status:
            self.orders_by_status[order['status']] = []
        self.orders_by_status[order['status']].append(order)

    def get_user_orders(self, user_id):
        return self.orders_by_user.get(user_id, [])

    def get_orders_by_status(self, status):
        return self.orders_by_status.get(status, [])

# API Layer
@app.post('/orders')
def create_order(order_data):
    # Write path
    command_handler.handle_create_order(order_data)
    return {'status': 'accepted'}

@app.get('/users/{user_id}/orders')
def get_user_orders(user_id):
    # Read path
    return query_model.get_user_orders(user_id)
```

**Pros:**
- ✅ Optimize reads and writes separately
- ✅ Scale read and write independently
- ✅ Simplified queries (denormalized)

**Cons:**
- ❌ Eventual consistency
- ❌ Complexity (two models to maintain)
- ❌ Data duplication

---

### 5. Saga Pattern

**What:** Manage distributed transactions with compensation.

**Use Case:** Multi-step business process across services.

```python
# Choreography-based Saga
class OrderSaga:
    """
    Steps:
    1. Create Order
    2. Process Payment
    3. Reserve Inventory
    4. Ship Order

    Compensations if any step fails:
    - Payment fails → Cancel order
    - Inventory fails → Refund payment, cancel order
    - Shipping fails → Restore inventory, refund payment, cancel order
    """

    # Step 1: Create Order
    @subscribe('CreateOrderCommand')
    def create_order(self, command):
        order = Order.create(command.data)
        event_bus.publish('OrderCreated', order.to_dict())

    # Step 2: Process Payment
    @subscribe('OrderCreated')
    def process_payment(self, event):
        try:
            payment = payment_service.charge(
                event['user_id'],
                event['total']
            )
            event_bus.publish('PaymentProcessed', {
                'order_id': event['order_id'],
                'payment_id': payment.id
            })
        except PaymentError as e:
            event_bus.publish('PaymentFailed', {
                'order_id': event['order_id'],
                'reason': str(e)
            })

    # Compensation: Cancel order if payment fails
    @subscribe('PaymentFailed')
    def cancel_order(self, event):
        Order.find(event['order_id']).cancel()
        event_bus.publish('OrderCancelled', event)

    # Step 3: Reserve Inventory
    @subscribe('PaymentProcessed')
    def reserve_inventory(self, event):
        try:
            inventory_service.reserve(event['order_id'])
            event_bus.publish('InventoryReserved', event)
        except OutOfStockError as e:
            event_bus.publish('InventoryFailed', event)

    # Compensation: Refund if inventory fails
    @subscribe('InventoryFailed')
    def refund_payment(self, event):
        payment_service.refund(event['payment_id'])
        event_bus.publish('PaymentRefunded', event)

    # Step 4: Ship Order
    @subscribe('InventoryReserved')
    def ship_order(self, event):
        tracking = shipping_service.ship(event['order_id'])
        event_bus.publish('OrderShipped', {
            'order_id': event['order_id'],
            'tracking_number': tracking
        })
```

**Pros:**
- ✅ No distributed locks
- ✅ Services remain autonomous
- ✅ Handles failures gracefully

**Cons:**
- ❌ Complex compensation logic
- ❌ Eventual consistency
- ❌ Difficult to debug

---

## Benefits of Event-Driven Architecture

### 1. Loose Coupling
Services don't need to know about each other.

```python
# Traditional (Tight Coupling)
class OrderService:
    def create_order(self, data):
        order = Order.create(data)
        payment_service.charge(order)      # Direct dependency
        inventory_service.reserve(order)   # Direct dependency
        email_service.send_confirmation(order)  # Direct dependency
        # What if we add analytics? Modify OrderService again!

# Event-Driven (Loose Coupling)
class OrderService:
    def create_order(self, data):
        order = Order.create(data)
        events.publish('OrderCreated', order.to_dict())
        # OrderService doesn't know who listens
        # Easy to add analytics, loyalty, etc. without changing OrderService
```

### 2. Scalability
Scale consumers independently.

```python
# High volume of order events
# Scale email service independently from payment service

# 10 instances of Email Service (slow, high volume)
email_service_instances = 10

# 2 instances of Payment Service (fast, critical)
payment_service_instances = 2

# Both consume 'OrderCreated' events at their own pace
```

### 3. Flexibility
Easy to add new functionality.

```python
# Add new feature: Send SMS on order
@subscribe('OrderCreated')
def send_sms_notification(event):
    sms.send(event['user_phone'], f"Order {event['order_id']} received!")

# No change to OrderService needed!
```

### 4. Resilience
Services can fail independently.

```python
# Email service is down?
# Order still created, payment processed, inventory reserved
# Email will be sent when service recovers (from queue backlog)

# Traditional approach would fail entire order if email service is down
```

---

## Challenges and Solutions

### Challenge 1: Eventual Consistency

**Problem:**
```python
# User places order
order_service.create_order()  # OrderCreated event published

# User immediately refreshes page
# Payment service hasn't processed yet!
# UI shows "Order created but payment pending"
```

**Solutions:**
- **Optimistic UI:** Show success immediately, update when confirmed
- **Polling:** Client polls until consistency achieved
- **WebSockets:** Push updates to client when state changes
- **Read-your-writes:** Route reads to same node that handled write

### Challenge 2: Event Ordering

**Problem:**
```python
# Events arrive out of order
events = [
    'OrderShipped',    # Arrived first
    'OrderCreated',    # Arrived second (network delay)
    'PaymentProcessed' # Arrived third
]
# Results in invalid state!
```

**Solutions:**
- **Sequence numbers:** Include sequence number in events
- **Timestamps:** Use timestamps to order events
- **Partition by key:** Same order_id always goes to same partition (Kafka)
- **Idempotency:** Make consumers handle out-of-order gracefully

```python
@subscribe('OrderShipped')
def handle_order_shipped(event):
    order = Order.find(event['order_id'])

    # Idempotent check
    if order is None:
        # Order not created yet, store event for later
        pending_events.store(event)
        return

    if order.status == 'shipped':
        # Already processed, ignore
        return

    order.status = 'shipped'
    order.save()
```

### Challenge 3: Debugging

**Problem:**
- Hard to trace event flow across services
- Difficult to reproduce issues

**Solutions:**
- **Correlation IDs:** Include trace ID in all events
- **Distributed Tracing:** Use tools like Jaeger, Zipkin
- **Event Logging:** Log all events centrally

```python
import uuid

# Generate correlation ID at entry point
correlation_id = str(uuid.uuid4())

# Include in all events
events.publish('OrderCreated', {
    'order_id': order.id,
    'correlation_id': correlation_id  # Track across all services
})

# Consumers propagate correlation ID
@subscribe('OrderCreated')
def process_payment(event):
    correlation_id = event['correlation_id']

    # Include in subsequent events
    events.publish('PaymentProcessed', {
        'order_id': event['order_id'],
        'correlation_id': correlation_id  # Same ID
    })

# Search logs by correlation ID to see full flow
```

### Challenge 4: Schema Evolution

**Problem:**
- Event schema changes over time
- Old consumers break

**Solution:** Versioning

```python
# Version 1
{
  "event_type": "OrderCreated",
  "version": "1.0",
  "data": {
    "order_id": "123",
    "total": 99.99
  }
}

# Version 2 (added items field)
{
  "event_type": "OrderCreated",
  "version": "2.0",
  "data": {
    "order_id": "123",
    "total": 99.99,
    "items": [...]  # New field
  }
}

# Consumer handles both versions
@subscribe('OrderCreated')
def handle_order(event):
    version = event['version']

    if version == '1.0':
        process_v1(event)
    elif version == '2.0':
        process_v2(event)
    else:
        raise UnknownVersionError()
```

---

## Best Practices

### 1. Event Naming
```python
# Good (past tense, specific)
'OrderPlaced'
'PaymentCompleted'
'UserRegistered'
'InventoryReserved'

# Bad (present tense, vague)
'PlaceOrder'
'Payment'
'User'
'Inventory'
```

### 2. Event Size
```python
# Keep events small
# Bad: Include everything
{
  "order": {...},  # Full order object (huge)
  "user": {...},   # Full user object
  "products": [...] # All product details
}

# Good: Include only what's needed
{
  "order_id": "123",
  "user_id": "456",
  "product_ids": ["p1", "p2"],
  "total": 99.99
}
```

### 3. Idempotency
```python
# Track processed events
processed_events = set()

@subscribe('OrderCreated')
def process_order(event):
    event_id = event['event_id']

    # Check if already processed
    if event_id in processed_events:
        return  # Skip duplicate

    # Process
    create_order(event)

    # Mark as processed
    processed_events.add(event_id)
```

### 4. Error Handling
```python
@subscribe('PaymentProcessed')
def reserve_inventory(event):
    try:
        inventory.reserve(event['order_id'])
    except TemporaryError as e:
        # Retry
        raise  # Event will be redelivered
    except PermanentError as e:
        # Don't retry, publish failure event
        events.publish('InventoryReservationFailed', {
            'order_id': event['order_id'],
            'reason': str(e)
        })
```

### 5. Monitoring
```python
# Metrics to track
metrics.increment('events.published', tags=['type:OrderCreated'])
metrics.increment('events.consumed', tags=['type:OrderCreated', 'service:payment'])
metrics.histogram('event.processing_time', processing_time)

# Alerts
if event_lag > 10000:
    alert('Event processing lag is high')

if dlq_size > 100:
    alert('Dead letter queue is growing')
```

---

## Real-World Examples

### Netflix
- **Use case:** Video encoding pipeline
- **Flow:** Video uploaded → Encoding jobs spawned → Multiple quality versions created → CDN populated

### Uber
- **Use case:** Trip events
- **Flow:** Trip requested → Driver matched → Trip started → Location updates → Trip ended → Payment processed

### Amazon
- **Use case:** Order fulfillment
- **Flow:** Order placed → Inventory reserved → Payment processed → Warehouse notified → Item shipped → Tracking updated

---

## Summary

**Event-Driven Architecture:**
- Services communicate via events (not direct calls)
- Producers emit events, consumers react
- Asynchronous, decoupled, scalable

**When to Use:**
- ✅ Microservices architecture
- ✅ Need for loose coupling
- ✅ Complex workflows (sagas)
- ✅ Real-time systems
- ✅ High scalability requirements

**When to Avoid:**
- ❌ Simple CRUD applications
- ❌ Strong consistency required
- ❌ Synchronous request-response needed
- ❌ Small team with limited ops capacity

**Key Patterns:**
1. Event Notification
2. Event-Carried State Transfer
3. Event Sourcing
4. CQRS
5. Saga Pattern

**Challenges:**
- Eventual consistency
- Event ordering
- Debugging complexity
- Schema evolution

**Best Practices:**
- Use correlation IDs
- Version events
- Make consumers idempotent
- Monitor lag and failures
- Keep events small and focused

---

**Next:** Explore specific implementations with [Message Queue Technologies](kafka-rabbitmq-sqs.md).
