# Kafka vs RabbitMQ vs SQS: Message Queue Comparison

## Overview

Choosing the right message queue technology is critical for system design. This guide compares the three most popular options: **Apache Kafka**, **RabbitMQ**, and **AWS SQS**.

**Quick Comparison:**

| Feature | Kafka | RabbitMQ | AWS SQS |
|---------|-------|----------|---------|
| **Type** | Distributed log | Message broker | Managed queue service |
| **Best For** | Event streaming, big data | Traditional messaging | AWS-native apps |
| **Throughput** | Very High (millions/sec) | High (tens of thousands/sec) | Medium (configurable) |
| **Latency** | Medium (ms) | Low (µs-ms) | Medium (ms) |
| **Message Ordering** | Per partition | Per queue | FIFO queues only |
| **Message Retention** | Configurable (days/weeks) | Until consumed | Max 14 days |
| **Complexity** | High | Medium | Low (managed) |

---

## Apache Kafka

### What is Kafka?

Kafka is a **distributed event streaming platform** designed for high-throughput, fault-tolerant, real-time data pipelines. Originally developed by LinkedIn, now maintained by Apache.

**Core Concept:** Distributed commit log (append-only)

### Architecture

```
Producers → [Topic: user_events]
               ├─ Partition 0 → Consumer Group A
               ├─ Partition 1 → Consumer Group A
               └─ Partition 2 → Consumer Group A
                              → Consumer Group B (separate copy)
```

**Key Components:**
- **Topics:** Category/feed of messages
- **Partitions:** Shards of a topic for parallelism
- **Brokers:** Kafka servers that store data
- **Producers:** Publish messages to topics
- **Consumers:** Subscribe to topics
- **ZooKeeper/KRaft:** Cluster coordination

### Key Characteristics

#### 1. High Throughput
```
Typical Performance:
- Write: 1-2 million messages/sec per broker
- Read: 10-15 million messages/sec per broker
- Horizontal scaling by adding brokers
```

#### 2. Message Persistence
- Messages stored on disk (not RAM)
- Configurable retention (time or size-based)
- Messages not deleted after consumption
- Multiple consumers can read same message

```properties
# Keep messages for 7 days
retention.ms=604800000

# Or keep 1TB of messages
retention.bytes=1099511627776
```

#### 3. Ordering Guarantee
- Messages within a partition are strictly ordered
- No ordering guarantee across partitions
- Use partition keys for related messages

```python
# Producer with partition key
producer.send('orders',
    key=user_id,  # Same user_id always goes to same partition
    value=order_data
)
```

#### 4. Consumer Groups
- Multiple consumers in a group share work
- Each partition consumed by only one consumer in group
- Automatic rebalancing when consumers join/leave

```python
# Consumer group
consumer = KafkaConsumer(
    'orders',
    group_id='order-processors',
    bootstrap_servers=['localhost:9092']
)

# Multiple instances of this consumer will share the load
for message in consumer:
    process_order(message.value)
```

### When to Use Kafka

✅ **Perfect For:**

**1. Event Streaming**
```python
# Stream processing pipeline
# Real-time analytics, ETL
clickstream_events → Kafka → Spark Streaming → Analytics DB
```

**2. Log Aggregation**
```python
# Centralized logging
App Servers (100s) → Kafka → Log Processor → Elasticsearch
```

**3. Metrics Collection**
```python
# Monitoring pipeline
Metrics Agents → Kafka → Prometheus → Grafana
```

**4. Data Integration (CDC - Change Data Capture)**
```python
# Database replication
MySQL → Debezium → Kafka → PostgreSQL
                        → ElasticSearch
                        → Data Warehouse
```

**5. Event Sourcing**
```python
# Store all state changes
Commands → Event Store (Kafka) → Replay → Current State
```

❌ **Not Ideal For:**
- Simple request-response patterns
- Task queues with acknowledgments
- Low-latency requirements (< 1ms)
- Small teams (operational complexity)

### Pros and Cons

**Pros:**
- ✅ Extremely high throughput (millions of messages/sec)
- ✅ Horizontal scalability
- ✅ Durable storage (disk-based)
- ✅ Replay capability (time-travel)
- ✅ Multiple consumers per message
- ✅ Strong ordering per partition

**Cons:**
- ❌ High operational complexity (ZooKeeper, broker management)
- ❌ Higher latency than RabbitMQ
- ❌ Steep learning curve
- ❌ No built-in message routing (topics only)
- ❌ Requires more infrastructure

### Code Example

```python
from kafka import KafkaProducer, KafkaConsumer
import json

# Producer
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Send message
producer.send('user-signups', {
    'user_id': '12345',
    'email': 'alice@example.com',
    'timestamp': '2024-01-15T10:30:00Z'
})
producer.flush()

# Consumer
consumer = KafkaConsumer(
    'user-signups',
    bootstrap_servers=['localhost:9092'],
    group_id='signup-processor',
    value_deserializer=lambda m: json.loads(m.decode('utf-8')),
    auto_offset_reset='earliest'  # Start from beginning
)

for message in consumer:
    print(f"Received: {message.value}")
    # Process signup
```

---

## RabbitMQ

### What is RabbitMQ?

RabbitMQ is a **message broker** that implements AMQP (Advanced Message Queuing Protocol). Focuses on reliable message delivery with flexible routing.

**Core Concept:** Message broker with exchanges and queues

### Architecture

```
Producer → [Exchange] → [Queue 1] → Consumer A
             (routing)  → [Queue 2] → Consumer B
                        → [Queue 3] → Consumer C
```

**Key Components:**
- **Exchanges:** Route messages to queues (direct, topic, fanout, headers)
- **Queues:** Store messages until consumed
- **Bindings:** Rules connecting exchanges to queues
- **Virtual Hosts:** Logical separation (multi-tenancy)

### Key Characteristics

#### 1. Flexible Routing

**Direct Exchange** (exact match):
```python
# Producer
channel.basic_publish(
    exchange='direct_logs',
    routing_key='error',  # Only goes to 'error' queue
    body='Database connection failed'
)
```

**Topic Exchange** (pattern match):
```python
# Routing key pattern: <facility>.<severity>
# auth.error, auth.info, payment.error, etc.

# Consumer 1: All errors
channel.queue_bind(
    queue='error_queue',
    exchange='logs',
    routing_key='*.error'
)

# Consumer 2: All auth logs
channel.queue_bind(
    queue='auth_queue',
    exchange='logs',
    routing_key='auth.*'
)
```

**Fanout Exchange** (broadcast):
```python
# All bound queues receive message
channel.basic_publish(
    exchange='notifications',
    routing_key='',  # Ignored for fanout
    body='System maintenance in 1 hour'
)
```

#### 2. Message Acknowledgment

```python
def callback(ch, method, properties, body):
    try:
        process_message(body)
        # Acknowledge successful processing
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception as e:
        # Reject and requeue
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)

channel.basic_consume(queue='tasks', on_message_callback=callback)
```

#### 3. Priority Queues

```python
# Create priority queue
channel.queue_declare(
    queue='tasks',
    arguments={'x-max-priority': 10}
)

# Send high-priority message
channel.basic_publish(
    exchange='',
    routing_key='tasks',
    body='Urgent task',
    properties=pika.BasicProperties(priority=9)
)
```

#### 4. TTL and Dead Letter Exchanges

```python
# Queue with TTL and DLX
channel.queue_declare(
    queue='tasks',
    arguments={
        'x-message-ttl': 60000,  # 60 seconds
        'x-dead-letter-exchange': 'dlx'
    }
)
```

### When to Use RabbitMQ

✅ **Perfect For:**

**1. Task Queues**
```python
# Background job processing
Web App → [Tasks Queue] → Workers (image resize, email send)
```

**2. Request-Response (RPC)**
```python
# Synchronous-like communication
Client → [Request Queue] → Worker
  ↑                           ↓
  └──── [Reply Queue] ←───────┘
```

**3. Complex Routing**
```python
# Multi-condition routing
Logs → [Topic Exchange] → Error Queue
                        → Audit Queue
                        → Analytics Queue
```

**4. Microservices Communication**
```python
# Service-to-service messaging
Order Service → [Exchange] → Payment Service
                           → Inventory Service
                           → Notification Service
```

❌ **Not Ideal For:**
- Very high throughput (millions/sec)
- Long-term message storage
- Event streaming / replay
- Large messages (> 128 MB)

### Pros and Cons

**Pros:**
- ✅ Low latency (microseconds to milliseconds)
- ✅ Flexible routing (4 exchange types)
- ✅ Strong delivery guarantees
- ✅ Management UI included
- ✅ Plugin ecosystem
- ✅ Multi-protocol support (AMQP, MQTT, STOMP)

**Cons:**
- ❌ Lower throughput than Kafka
- ❌ Messages deleted after consumption
- ❌ Vertical scaling limits
- ❌ Complex clustering setup
- ❌ No built-in replay

### Code Example

```python
import pika

# Producer
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Declare queue
channel.queue_declare(queue='tasks', durable=True)

# Publish message
channel.basic_publish(
    exchange='',
    routing_key='tasks',
    body='Process image: img123.jpg',
    properties=pika.BasicProperties(
        delivery_mode=2,  # Persistent
    )
)

connection.close()

# Consumer
def callback(ch, method, properties, body):
    print(f"Processing: {body}")
    # Simulate work
    time.sleep(5)
    ch.basic_ack(delivery_tag=method.delivery_tag)

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
channel.queue_declare(queue='tasks', durable=True)

channel.basic_qos(prefetch_count=1)  # Fair dispatch
channel.basic_consume(queue='tasks', on_message_callback=callback)

print('Waiting for messages...')
channel.start_consuming()
```

---

## AWS SQS (Simple Queue Service)

### What is SQS?

AWS SQS is a **fully managed message queue service** in the cloud. No infrastructure to manage, pay-per-use, integrates with AWS ecosystem.

**Core Concept:** Managed queue-as-a-service

### Types

#### 1. Standard Queue
- **Ordering:** Best-effort (not guaranteed)
- **Delivery:** At-least-once (possible duplicates)
- **Throughput:** Unlimited
- **Use case:** High throughput, can tolerate duplicates

#### 2. FIFO Queue
- **Ordering:** Strict FIFO within message group
- **Delivery:** Exactly-once processing
- **Throughput:** 300 msg/sec (3000 with batching)
- **Use case:** Order matters, no duplicates

```python
# FIFO queue naming
queue_name = 'orders.fifo'  # Must end with .fifo

# Send with deduplication
sqs.send_message(
    QueueUrl=queue_url,
    MessageBody='Order data',
    MessageGroupId='user-123',  # Messages with same group ID are ordered
    MessageDeduplicationId='order-456'  # Prevents duplicates
)
```

### Key Characteristics

#### 1. Fully Managed
- No servers to manage
- Auto-scaling
- Built-in security (IAM, encryption)
- High availability (multi-AZ)

#### 2. Dead Letter Queues

```python
# Create DLQ
dlq = sqs.create_queue(QueueName='failed-tasks')

# Configure main queue with DLQ
sqs.set_queue_attributes(
    QueueUrl=main_queue_url,
    Attributes={
        'RedrivePolicy': json.dumps({
            'deadLetterTargetArn': dlq_arn,
            'maxReceiveCount': 3  # Move to DLQ after 3 failed attempts
        })
    }
)
```

#### 3. Visibility Timeout

```python
# Message hidden from other consumers while being processed
# Default: 30 seconds

# Receive message (becomes invisible)
response = sqs.receive_message(
    QueueUrl=queue_url,
    VisibilityTimeout=60  # 60 seconds to process
)

# If processing takes longer, extend timeout
sqs.change_message_visibility(
    QueueUrl=queue_url,
    ReceiptHandle=receipt_handle,
    VisibilityTimeout=120  # Extend to 120 seconds
)

# Delete after successful processing
sqs.delete_message(
    QueueUrl=queue_url,
    ReceiptHandle=receipt_handle
)
```

#### 4. Long Polling

```python
# Reduce costs and latency
response = sqs.receive_message(
    QueueUrl=queue_url,
    WaitTimeSeconds=20,  # Long polling (up to 20 seconds)
    MaxNumberOfMessages=10  # Batch receive
)
```

### When to Use SQS

✅ **Perfect For:**

**1. AWS-Native Applications**
```python
# Integrates seamlessly with AWS services
S3 Event → SQS → Lambda → Process uploaded file
```

**2. Decoupling Microservices**
```python
# Service A and Service B don't need to know about each other
Service A → [SQS] → Service B
```

**3. Simple Task Queues**
```python
# Background job processing
Web App → [SQS] → Workers (send email, resize image)
```

**4. Event-Driven Architectures**
```python
# SNS + SQS fan-out pattern
SNS Topic → SQS Queue 1 → Service A
         → SQS Queue 2 → Service B
         → SQS Queue 3 → Service C
```

❌ **Not Ideal For:**
- Real-time streaming
- Message ordering across all messages (use FIFO for groups)
- Message replay
- Non-AWS environments

### Pros and Cons

**Pros:**
- ✅ Fully managed (no ops overhead)
- ✅ Unlimited scalability
- ✅ Pay-per-use pricing
- ✅ AWS integration (Lambda, S3, SNS)
- ✅ Built-in DLQ support
- ✅ Security (IAM, encryption at rest/in transit)

**Cons:**
- ❌ AWS vendor lock-in
- ❌ Limited throughput for FIFO queues
- ❌ No message routing (use SNS)
- ❌ Max message size 256 KB
- ❌ Max retention 14 days
- ❌ Can be expensive at high volumes

### Code Example

```python
import boto3
import json

sqs = boto3.client('sqs', region_name='us-east-1')
queue_url = 'https://sqs.us-east-1.amazonaws.com/123456789/my-queue'

# Send message
response = sqs.send_message(
    QueueUrl=queue_url,
    MessageBody=json.dumps({
        'order_id': '12345',
        'user_id': 'user_789',
        'total': 99.99
    }),
    MessageAttributes={
        'Priority': {
            'DataType': 'Number',
            'StringValue': '1'
        }
    }
)

# Receive and process messages
while True:
    response = sqs.receive_message(
        QueueUrl=queue_url,
        MaxNumberOfMessages=10,  # Batch processing
        WaitTimeSeconds=20  # Long polling
    )

    messages = response.get('Messages', [])

    for message in messages:
        body = json.loads(message['Body'])
        print(f"Processing order: {body['order_id']}")

        try:
            process_order(body)

            # Delete message after successful processing
            sqs.delete_message(
                QueueUrl=queue_url,
                ReceiptHandle=message['ReceiptHandle']
            )
        except Exception as e:
            print(f"Error: {e}")
            # Message will become visible again after visibility timeout
```

---

## Detailed Comparison

### Performance Comparison

| Metric | Kafka | RabbitMQ | SQS |
|--------|-------|----------|-----|
| **Throughput** | 1-2M msg/sec | 10-50K msg/sec | Variable (high for standard) |
| **Latency** | 2-10 ms | < 1 ms | 10-20 ms |
| **Message Size** | 1 MB default | 128 MB | 256 KB |
| **Retention** | Days/Weeks (configurable) | Until consumed | Max 14 days |
| **Ordering** | Per partition | Per queue | FIFO queues only |
| **Durability** | Disk + replication | Disk + replication | Multi-AZ replication |

### Feature Comparison

| Feature | Kafka | RabbitMQ | SQS |
|---------|-------|----------|-----|
| **Message Replay** | ✅ Yes | ❌ No | ❌ No |
| **Pub/Sub** | ✅ Native | ✅ Via exchanges | ✅ Via SNS |
| **Message Routing** | ❌ Topic-based only | ✅ Flexible (4 types) | ❌ Basic |
| **Priority Queues** | ❌ No | ✅ Yes | ❌ No |
| **Delayed Messages** | ❌ No | ✅ Plugin | ✅ Yes (up to 15 min) |
| **Transaction Support** | ✅ Yes (0.11+) | ✅ Yes | ❌ No |
| **Multi-tenancy** | ❌ Limited | ✅ Virtual hosts | ✅ AWS accounts |
| **Management UI** | ❌ 3rd party | ✅ Built-in | ✅ AWS Console |

### Operational Comparison

| Aspect | Kafka | RabbitMQ | SQS |
|--------|-------|----------|-----|
| **Setup Complexity** | High | Medium | None (managed) |
| **Operational Overhead** | High | Medium | None |
| **Scaling** | Add brokers | Add nodes | Auto-scaled |
| **Monitoring** | JMX, Prometheus | Built-in + plugins | CloudWatch |
| **Clustering** | Native | Native | Multi-AZ built-in |
| **Cost** | Infrastructure cost | Infrastructure cost | Pay-per-request |

### Use Case Matrix

| Use Case | Recommended | Why |
|----------|-------------|-----|
| **Event Streaming** | Kafka | High throughput, replay capability |
| **Task Queue** | RabbitMQ or SQS | Acknowledgments, flexible routing |
| **Log Aggregation** | Kafka | High volume, retention, replay |
| **Microservices** | RabbitMQ | Routing, low latency |
| **AWS Serverless** | SQS | Native Lambda integration |
| **Real-time Analytics** | Kafka | Stream processing, windowing |
| **Background Jobs** | RabbitMQ or SQS | Reliable delivery, DLQ |
| **Pub/Sub Notifications** | RabbitMQ or SNS+SQS | Fanout, filtering |
| **CDC (Database Sync)** | Kafka | Ordered events, replay |
| **Simple Decoupling** | SQS | Managed, no ops |

---

## Real-World Examples

### Companies Using Kafka
- **LinkedIn:** Activity streams, operational metrics (original creator)
- **Netflix:** Real-time monitoring, recommendation engine input
- **Uber:** Trip events, location data, analytics
- **Spotify:** Logging, event processing
- **Twitter:** Stream processing infrastructure

### Companies Using RabbitMQ
- **Instagram:** Photo processing pipeline
- **SoundCloud:** Background job processing
- **9GAG:** Content distribution
- **Reddit:** Vote processing (historically)

### Companies Using SQS
- **Airbnb:** Async task processing
- **BMW:** Connected car data processing
- **Expedia:** Booking confirmation workflows
- **Capital One:** Event-driven architecture

---

## Decision Framework

### Choose Kafka When:
1. ✅ You need very high throughput (millions of messages/sec)
2. ✅ You want to replay messages (event sourcing, debugging)
3. ✅ You're building data pipelines or stream processing
4. ✅ You need message retention for days/weeks
5. ✅ You have dedicated DevOps team for operations
6. ✅ You're doing real-time analytics or ETL

### Choose RabbitMQ When:
1. ✅ You need low latency (microseconds)
2. ✅ You need complex routing patterns
3. ✅ You're building traditional microservices
4. ✅ You need priority queues
5. ✅ You want flexible message acknowledgment
6. ✅ You need protocol flexibility (AMQP, MQTT, STOMP)

### Choose SQS When:
1. ✅ You're already on AWS
2. ✅ You want zero operational overhead
3. ✅ You need unlimited scalability
4. ✅ You want pay-per-use pricing
5. ✅ You're building serverless applications
6. ✅ Your team is small (no dedicated DevOps)

---

## Hybrid Approaches

### Kafka + SQS
```
High-volume events → Kafka (stream processing)
                       ↓
               Filtered events → SQS → Lambda (specific actions)
```

### RabbitMQ + Kafka
```
Microservices ↔ RabbitMQ (low latency)
                    ↓
            Events → Kafka (analytics, long-term storage)
```

### SNS + SQS (AWS)
```
Event Source → SNS Topic → SQS Queue 1 → Service A
                        → SQS Queue 2 → Service B
                        → SQS Queue 3 → Service C
```

---

## Summary

**Kafka:** High-throughput event streaming platform
- **Best for:** Event streaming, analytics, log aggregation
- **Trade-off:** Complexity for throughput and replay

**RabbitMQ:** Flexible message broker
- **Best for:** Microservices, task queues, complex routing
- **Trade-off:** Lower throughput for flexibility and low latency

**AWS SQS:** Managed queue service
- **Best for:** AWS-native apps, simple decoupling, serverless
- **Trade-off:** AWS lock-in for zero operational overhead

**Interview Tip:** Don't say "Kafka is always better" or "RabbitMQ is legacy." Each has valid use cases. Explain trade-offs based on requirements:
- Throughput needs
- Latency requirements
- Operational capacity
- Message replay needs
- Cloud vs on-premise

---

**Next:** Learn about [Pub/Sub Pattern](pub-sub-pattern.md) and [Event-Driven Architecture](event-driven.md) in depth.
