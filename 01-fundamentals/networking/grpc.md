# gRPC (gRPC Remote Procedure Call)

## Table of Contents
- [What is gRPC?](#what-is-grpc)
- [Core Concepts](#core-concepts)
- [Protocol Buffers](#protocol-buffers)
- [gRPC Communication Patterns](#grpc-communication-patterns)
- [gRPC vs REST](#grpc-vs-rest)
- [Implementation Examples](#implementation-examples)
- [Advanced Features](#advanced-features)
- [Performance and Optimization](#performance-and-optimization)
- [Use Cases](#use-cases)
- [Best Practices](#best-practices)
- [Interview Tips](#interview-tips)

---

## What is gRPC?

**gRPC** is a modern, high-performance RPC (Remote Procedure Call) framework developed by Google. It uses **HTTP/2** for transport, **Protocol Buffers** as the interface description language, and provides features like authentication, load balancing, and more.

### Key Characteristics

```
Traditional REST:
Client → HTTP/1.1 → JSON → Server
- Text-based
- Stateless
- Multiple requests

gRPC:
Client → HTTP/2 → Protocol Buffers → Server
- Binary protocol
- Multiplexed streams
- Bidirectional streaming
```

### Why gRPC?

1. **Performance**: Binary serialization (faster than JSON)
2. **Strongly Typed**: Contract-first API design
3. **Streaming**: Real-time bidirectional communication
4. **Polyglot**: Code generation for multiple languages
5. **HTTP/2**: Multiplexing, header compression, server push
6. **Deadline/Timeouts**: Built-in timeout propagation
7. **Cancellation**: Request cancellation support

---

## Core Concepts

### 1. Service Definition

Define services using Protocol Buffers (.proto files):

```protobuf
syntax = "proto3";

package ecommerce;

// Service definition
service ProductService {
  rpc GetProduct (ProductRequest) returns (Product);
  rpc ListProducts (ListProductsRequest) returns (stream Product);
  rpc CreateProduct (stream Product) returns (ProductResponse);
  rpc ChatAboutProduct (stream Message) returns (stream Message);
}

// Message definitions
message Product {
  string id = 1;
  string name = 2;
  double price = 3;
  string description = 4;
}

message ProductRequest {
  string id = 1;
}
```

### 2. Code Generation

```bash
# Generate Python code
python -m grpc_tools.protoc -I. \
    --python_out=. \
    --grpc_python_out=. \
    product.proto

# Generate Go code
protoc --go_out=. --go-grpc_out=. product.proto

# Generate Java code
protoc --java_out=. --grpc-java_out=. product.proto
```

**Generated files:**
- `product_pb2.py` - Message classes
- `product_pb2_grpc.py` - Service stubs and server interfaces

### 3. Client-Server Architecture

```
┌─────────────┐                    ┌─────────────┐
│   Client    │                    │   Server    │
│             │                    │             │
│ ┌─────────┐ │                    │ ┌─────────┐ │
│ │  Stub   │ │ ──── RPC Call ───→ │ │ Service │ │
│ └─────────┘ │                    │ └─────────┘ │
│             │ ←─── Response ──── │             │
└─────────────┘                    └─────────────┘
      ↓                                   ↓
  HTTP/2 Client                      HTTP/2 Server
```

---

## Protocol Buffers

**Protocol Buffers (protobuf)** is a language-neutral, platform-neutral mechanism for serializing structured data.

### Basic Syntax

```protobuf
syntax = "proto3";

// Scalar types
message User {
  string name = 1;          // string
  int32 age = 2;            // integer
  bool is_active = 3;       // boolean
  double balance = 4;       // double
  bytes avatar = 5;         // binary data
}

// Repeated fields (arrays)
message UserList {
  repeated User users = 1;
}

// Nested messages
message Address {
  string street = 1;
  string city = 2;
  string country = 3;
}

message UserProfile {
  User user = 1;
  Address address = 2;
}

// Enums
enum OrderStatus {
  PENDING = 0;
  CONFIRMED = 1;
  SHIPPED = 2;
  DELIVERED = 3;
}

// Maps
message Inventory {
  map<string, int32> products = 1;  // product_id -> quantity
}

// Oneof (union types)
message Payment {
  oneof payment_method {
    string credit_card = 1;
    string paypal_email = 2;
    string bank_account = 3;
  }
}
```

### Protobuf vs JSON

```python
# JSON serialization
import json

user_json = {
    "name": "Alice",
    "age": 30,
    "email": "alice@example.com"
}
json_data = json.dumps(user_json)  # 50+ bytes
print(json_data)
# {"name": "Alice", "age": 30, "email": "alice@example.com"}

# Protobuf serialization
from user_pb2 import User

user_proto = User(name="Alice", age=30, email="alice@example.com")
proto_data = user_proto.SerializeToString()  # 20-30 bytes
print(proto_data)
# b'\n\x05Alice\x10\x1e\x1a\x13alice@example.com'
```

**Advantages:**
- **Smaller size**: 3-10x smaller than JSON
- **Faster**: 5-10x faster serialization/deserialization
- **Strongly typed**: Schema validation
- **Backward compatible**: Add fields without breaking old clients

---

## gRPC Communication Patterns

gRPC supports **4 types of RPC calls**:

### 1. Unary RPC (Request-Response)

Simple request-response, like REST.

```protobuf
service UserService {
  rpc GetUser (UserRequest) returns (UserResponse);
}
```

**Python Server:**
```python
import grpc
from concurrent import futures
import user_pb2
import user_pb2_grpc

class UserService(user_pb2_grpc.UserServiceServicer):
    def GetUser(self, request, context):
        # Fetch user from database
        user = {
            'id': request.user_id,
            'name': 'Alice',
            'email': 'alice@example.com'
        }
        return user_pb2.UserResponse(
            id=user['id'],
            name=user['name'],
            email=user['email']
        )

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    user_pb2_grpc.add_UserServiceServicer_to_server(UserService(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    server.wait_for_termination()

if __name__ == '__main__':
    serve()
```

**Python Client:**
```python
import grpc
import user_pb2
import user_pb2_grpc

def get_user(user_id):
    with grpc.insecure_channel('localhost:50051') as channel:
        stub = user_pb2_grpc.UserServiceStub(channel)
        request = user_pb2.UserRequest(user_id=user_id)
        response = stub.GetUser(request)
        print(f"User: {response.name}, Email: {response.email}")

get_user("123")
```

### 2. Server Streaming RPC

Server sends stream of messages in response to client request.

```protobuf
service ProductService {
  rpc ListProducts (Empty) returns (stream Product);
}
```

**Use Cases:**
- Fetching large datasets
- Live updates
- Server-side pagination

**Python Server:**
```python
class ProductService(product_pb2_grpc.ProductServiceServicer):
    def ListProducts(self, request, context):
        products = [
            {"id": "1", "name": "Laptop", "price": 999.99},
            {"id": "2", "name": "Mouse", "price": 29.99},
            {"id": "3", "name": "Keyboard", "price": 79.99},
        ]
        for product in products:
            yield product_pb2.Product(
                id=product['id'],
                name=product['name'],
                price=product['price']
            )
```

**Python Client:**
```python
def list_products():
    with grpc.insecure_channel('localhost:50051') as channel:
        stub = product_pb2_grpc.ProductServiceStub(channel)
        products = stub.ListProducts(product_pb2.Empty())
        for product in products:
            print(f"Product: {product.name}, Price: ${product.price}")
```

### 3. Client Streaming RPC

Client sends stream of messages, server responds with single message.

```protobuf
service OrderService {
  rpc CreateOrder (stream OrderItem) returns (OrderResponse);
}
```

**Use Cases:**
- Uploading files
- Batch operations
- Sending metrics/logs

**Python Server:**
```python
class OrderService(order_pb2_grpc.OrderServiceServicer):
    def CreateOrder(self, request_iterator, context):
        total = 0
        items = []
        for item in request_iterator:
            items.append(item.name)
            total += item.price * item.quantity

        return order_pb2.OrderResponse(
            order_id="ORD-123",
            total=total,
            items_count=len(items)
        )
```

**Python Client:**
```python
def create_order():
    items = [
        order_pb2.OrderItem(name="Laptop", price=999.99, quantity=1),
        order_pb2.OrderItem(name="Mouse", price=29.99, quantity=2),
    ]

    with grpc.insecure_channel('localhost:50051') as channel:
        stub = order_pb2_grpc.OrderServiceStub(channel)
        response = stub.CreateOrder(iter(items))
        print(f"Order ID: {response.order_id}, Total: ${response.total}")
```

### 4. Bidirectional Streaming RPC

Both client and server send streams of messages.

```protobuf
service ChatService {
  rpc Chat (stream ChatMessage) returns (stream ChatMessage);
}
```

**Use Cases:**
- Real-time chat
- Live video streaming
- Multiplayer games
- Collaborative editing

**Python Server:**
```python
class ChatService(chat_pb2_grpc.ChatServiceServicer):
    def Chat(self, request_iterator, context):
        for message in request_iterator:
            # Echo message back with timestamp
            response = chat_pb2.ChatMessage(
                user=message.user,
                text=f"Echo: {message.text}",
                timestamp=time.time()
            )
            yield response
```

**Python Client:**
```python
def chat():
    messages = [
        chat_pb2.ChatMessage(user="Alice", text="Hello!"),
        chat_pb2.ChatMessage(user="Alice", text="How are you?"),
    ]

    with grpc.insecure_channel('localhost:50051') as channel:
        stub = chat_pb2_grpc.ChatServiceStub(channel)
        responses = stub.Chat(iter(messages))
        for response in responses:
            print(f"{response.user}: {response.text}")
```

---

## gRPC vs REST

### Comparison Table

| Feature | gRPC | REST |
|---------|------|------|
| **Protocol** | HTTP/2 | HTTP/1.1 (usually) |
| **Payload** | Protocol Buffers (binary) | JSON (text) |
| **API Contract** | Strict (.proto files) | Loose (OpenAPI optional) |
| **Streaming** | Bidirectional | Limited (SSE, WebSocket) |
| **Browser Support** | Limited (needs proxy) | Native |
| **Performance** | High (binary, multiplexing) | Lower (text, multiple connections) |
| **Code Generation** | Built-in | Third-party tools |
| **Human Readable** | No (binary) | Yes (JSON) |
| **Caching** | Complex | HTTP caching |
| **Latency** | Lower | Higher |

### When to Use gRPC

✅ **Use gRPC when:**
- Microservices communication (backend-to-backend)
- Real-time streaming required
- Polyglot environments (multiple languages)
- Performance is critical
- Mobile clients connecting to backend
- Internal APIs

❌ **Don't use gRPC when:**
- Browser-based clients (limited support)
- Public-facing APIs (REST is more common)
- Simple CRUD operations
- Need HTTP caching
- Debugging ease is important

### Performance Comparison

```python
import time
import json
from user_pb2 import User

# JSON serialization
user_dict = {'name': 'Alice', 'age': 30, 'email': 'alice@example.com'}

start = time.time()
for _ in range(100000):
    data = json.dumps(user_dict)
    json.loads(data)
json_time = time.time() - start

# Protobuf serialization
user_proto = User(name='Alice', age=30, email='alice@example.com')

start = time.time()
for _ in range(100000):
    data = user_proto.SerializeToString()
    User().ParseFromString(data)
proto_time = time.time() - start

print(f"JSON: {json_time:.2f}s")
print(f"Protobuf: {proto_time:.2f}s")
print(f"Speedup: {json_time/proto_time:.1f}x")

# Typical output:
# JSON: 2.45s
# Protobuf: 0.38s
# Speedup: 6.4x
```

---

## Implementation Examples

### Complete Example: E-commerce Service

**1. Define Service (ecommerce.proto):**

```protobuf
syntax = "proto3";

package ecommerce;

service EcommerceService {
  // Unary
  rpc GetProduct (ProductRequest) returns (Product);

  // Server streaming
  rpc SearchProducts (SearchRequest) returns (stream Product);

  // Client streaming
  rpc AddToCart (stream CartItem) returns (CartResponse);

  // Bidirectional streaming
  rpc LiveInventory (stream InventoryQuery) returns (stream InventoryUpdate);
}

message Product {
  string id = 1;
  string name = 2;
  double price = 3;
  int32 stock = 4;
}

message ProductRequest {
  string id = 1;
}

message SearchRequest {
  string query = 1;
  int32 max_results = 2;
}

message CartItem {
  string product_id = 1;
  int32 quantity = 2;
}

message CartResponse {
  string cart_id = 1;
  double total = 2;
  int32 item_count = 3;
}

message InventoryQuery {
  string product_id = 1;
}

message InventoryUpdate {
  string product_id = 1;
  int32 stock = 2;
}
```

**2. Server Implementation:**

```python
import grpc
from concurrent import futures
import ecommerce_pb2
import ecommerce_pb2_grpc
import time

class EcommerceService(ecommerce_pb2_grpc.EcommerceServiceServicer):
    def __init__(self):
        self.products = {
            "1": {"name": "Laptop", "price": 999.99, "stock": 10},
            "2": {"name": "Mouse", "price": 29.99, "stock": 50},
            "3": {"name": "Keyboard", "price": 79.99, "stock": 30},
        }

    def GetProduct(self, request, context):
        product = self.products.get(request.id)
        if not product:
            context.abort(grpc.StatusCode.NOT_FOUND, "Product not found")

        return ecommerce_pb2.Product(
            id=request.id,
            name=product['name'],
            price=product['price'],
            stock=product['stock']
        )

    def SearchProducts(self, request, context):
        query = request.query.lower()
        count = 0
        for pid, product in self.products.items():
            if query in product['name'].lower():
                yield ecommerce_pb2.Product(
                    id=pid,
                    name=product['name'],
                    price=product['price'],
                    stock=product['stock']
                )
                count += 1
                if count >= request.max_results:
                    break

    def AddToCart(self, request_iterator, context):
        total = 0
        item_count = 0
        for item in request_iterator:
            product = self.products.get(item.product_id)
            if product:
                total += product['price'] * item.quantity
                item_count += item.quantity

        return ecommerce_pb2.CartResponse(
            cart_id="CART-" + str(int(time.time())),
            total=total,
            item_count=item_count
        )

    def LiveInventory(self, request_iterator, context):
        for query in request_iterator:
            product = self.products.get(query.product_id)
            if product:
                yield ecommerce_pb2.InventoryUpdate(
                    product_id=query.product_id,
                    stock=product['stock']
                )
                time.sleep(1)  # Simulate real-time updates

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    ecommerce_pb2_grpc.add_EcommerceServiceServicer_to_server(
        EcommerceService(), server
    )
    server.add_insecure_port('[::]:50051')
    print("Server started on port 50051")
    server.start()
    server.wait_for_termination()

if __name__ == '__main__':
    serve()
```

**3. Client Implementation:**

```python
import grpc
import ecommerce_pb2
import ecommerce_pb2_grpc

def run_client():
    with grpc.insecure_channel('localhost:50051') as channel:
        stub = ecommerce_pb2_grpc.EcommerceServiceStub(channel)

        # 1. Unary call
        print("=== Get Product ===")
        product = stub.GetProduct(ecommerce_pb2.ProductRequest(id="1"))
        print(f"{product.name}: ${product.price} (Stock: {product.stock})")

        # 2. Server streaming
        print("\n=== Search Products ===")
        search = stub.SearchProducts(
            ecommerce_pb2.SearchRequest(query="", max_results=10)
        )
        for product in search:
            print(f"{product.name}: ${product.price}")

        # 3. Client streaming
        print("\n=== Add to Cart ===")
        items = [
            ecommerce_pb2.CartItem(product_id="1", quantity=1),
            ecommerce_pb2.CartItem(product_id="2", quantity=2),
        ]
        cart = stub.AddToCart(iter(items))
        print(f"Cart ID: {cart.cart_id}, Total: ${cart.total}")

        # 4. Bidirectional streaming
        print("\n=== Live Inventory ===")
        def inventory_queries():
            for pid in ["1", "2", "3"]:
                yield ecommerce_pb2.InventoryQuery(product_id=pid)

        updates = stub.LiveInventory(inventory_queries())
        for update in updates:
            print(f"Product {update.product_id}: {update.stock} in stock")

if __name__ == '__main__':
    run_client()
```

---

## Advanced Features

### 1. Metadata (Headers)

```python
# Server: Read metadata
def GetUser(self, request, context):
    metadata = dict(context.invocation_metadata())
    auth_token = metadata.get('authorization')
    print(f"Auth token: {auth_token}")

    # Send metadata in response
    context.set_trailing_metadata([
        ('response-id', 'xyz-123'),
    ])
    return user_pb2.UserResponse(...)

# Client: Send metadata
metadata = [
    ('authorization', 'Bearer token123'),
    ('client-id', 'web-app'),
]
response = stub.GetUser(request, metadata=metadata)
```

### 2. Error Handling

```python
# Server: Return errors
def GetUser(self, request, context):
    if not user_exists(request.user_id):
        context.abort(
            grpc.StatusCode.NOT_FOUND,
            f"User {request.user_id} not found"
        )
    return user

# Client: Handle errors
try:
    response = stub.GetUser(request)
except grpc.RpcError as e:
    print(f"Error: {e.code()}, {e.details()}")
    if e.code() == grpc.StatusCode.NOT_FOUND:
        print("User not found")
```

**Status Codes:**
- `OK` - Success
- `CANCELLED` - Operation cancelled
- `INVALID_ARGUMENT` - Invalid request
- `NOT_FOUND` - Resource not found
- `UNAUTHENTICATED` - No valid auth
- `PERMISSION_DENIED` - No permission
- `UNAVAILABLE` - Service unavailable
- `INTERNAL` - Internal error

### 3. Deadlines and Timeouts

```python
# Client: Set deadline (5 seconds)
response = stub.GetUser(
    request,
    timeout=5.0  # seconds
)

# Server: Check remaining time
def GetUser(self, request, context):
    if context.time_remaining() < 1.0:
        context.abort(grpc.StatusCode.DEADLINE_EXCEEDED, "Too slow")
    # Process request...
```

### 4. Interceptors (Middleware)

```python
class AuthInterceptor(grpc.ServerInterceptor):
    def intercept_service(self, continuation, handler_call_details):
        metadata = dict(handler_call_details.invocation_metadata)
        token = metadata.get('authorization')

        if not self.validate_token(token):
            context.abort(grpc.StatusCode.UNAUTHENTICATED, "Invalid token")

        return continuation(handler_call_details)

    def validate_token(self, token):
        return token == "Bearer valid_token"

# Add interceptor to server
server = grpc.server(
    futures.ThreadPoolExecutor(max_workers=10),
    interceptors=[AuthInterceptor()]
)
```

### 5. TLS/SSL Security

```python
# Server: Enable TLS
with open('server.crt', 'rb') as f:
    cert = f.read()
with open('server.key', 'rb') as f:
    key = f.read()

server_credentials = grpc.ssl_server_credentials([(key, cert)])
server.add_secure_port('[::]:50051', server_credentials)

# Client: Connect with TLS
with open('server.crt', 'rb') as f:
    trusted_certs = f.read()

credentials = grpc.ssl_channel_credentials(trusted_certs)
channel = grpc.secure_channel('localhost:50051', credentials)
```

---

## Performance and Optimization

### 1. Connection Pooling

```python
# Reuse channels instead of creating new ones
class GrpcClient:
    def __init__(self, address):
        self.channel = grpc.insecure_channel(address)
        self.stub = user_pb2_grpc.UserServiceStub(self.channel)

    def get_user(self, user_id):
        return self.stub.GetUser(user_pb2.UserRequest(user_id=user_id))

    def close(self):
        self.channel.close()

# Use one client for multiple requests
client = GrpcClient('localhost:50051')
user1 = client.get_user("1")
user2 = client.get_user("2")
client.close()
```

### 2. Compression

```python
# Enable compression
response = stub.GetUser(
    request,
    compression=grpc.Compression.Gzip
)
```

### 3. Load Balancing

```python
# Client-side load balancing
channel = grpc.insecure_channel(
    'dns:///my-service:50051',
    options=[('grpc.lb_policy_name', 'round_robin')]
)
```

### 4. Keep-Alive

```python
# Configure keep-alive
channel = grpc.insecure_channel(
    'localhost:50051',
    options=[
        ('grpc.keepalive_time_ms', 10000),
        ('grpc.keepalive_timeout_ms', 5000),
    ]
)
```

---

## Use Cases

### 1. Microservices Communication

```
┌─────────────┐    gRPC     ┌─────────────┐
│   API       │────────────→│   User      │
│  Gateway    │             │  Service    │
└─────────────┘             └─────────────┘
       │                           │
       │ gRPC                     │ gRPC
       ↓                           ↓
┌─────────────┐             ┌─────────────┐
│   Order     │────────────→│  Payment    │
│  Service    │    gRPC     │  Service    │
└─────────────┘             └─────────────┘
```

**Advantages:**
- Fast inter-service communication
- Strongly typed contracts
- Efficient binary protocol

### 2. Mobile to Backend

Mobile apps use gRPC for efficient communication:
- Smaller payloads (saves bandwidth)
- Battery efficient
- Streaming support for real-time features

### 3. Real-Time Features

```python
# Live location tracking
service LocationService {
  rpc TrackLocation (stream Location) returns (stream LocationUpdate);
}

# Stock price updates
service StockService {
  rpc WatchStock (StockRequest) returns (stream StockPrice);
}
```

### 4. IoT Devices

```python
# IoT sensor data collection
service SensorService {
  rpc StreamSensorData (stream SensorReading) returns (Acknowledgment);
}
```

---

## Best Practices

### 1. API Versioning

```protobuf
// Option 1: Package versioning
package myapp.v1;

// Option 2: Service versioning
service UserServiceV1 {
  rpc GetUser (UserRequest) returns (User);
}

service UserServiceV2 {
  rpc GetUser (UserRequestV2) returns (UserV2);
}
```

### 2. Error Handling

```python
# Use appropriate status codes
def GetUser(self, request, context):
    try:
        user = db.get_user(request.user_id)
        if not user:
            context.abort(grpc.StatusCode.NOT_FOUND, "User not found")
        return user
    except ValueError as e:
        context.abort(grpc.StatusCode.INVALID_ARGUMENT, str(e))
    except Exception as e:
        context.abort(grpc.StatusCode.INTERNAL, "Internal error")
```

### 3. Graceful Shutdown

```python
import signal

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    # ... add services ...
    server.start()

    def handle_sigterm(*_):
        print("Shutting down...")
        server.stop(grace=10)  # 10 second grace period

    signal.signal(signal.SIGTERM, handle_sigterm)
    server.wait_for_termination()
```

### 4. Monitoring and Metrics

```python
import prometheus_client

class MetricsInterceptor(grpc.ServerInterceptor):
    def __init__(self):
        self.request_count = prometheus_client.Counter(
            'grpc_requests_total',
            'Total gRPC requests',
            ['method', 'status']
        )

    def intercept_service(self, continuation, handler_call_details):
        method = handler_call_details.method
        response = continuation(handler_call_details)
        self.request_count.labels(method=method, status='ok').inc()
        return response
```

---

## Interview Tips

### Common gRPC Interview Questions

#### Q1: What is gRPC and how is it different from REST?

**Answer:**
- gRPC uses HTTP/2 and Protocol Buffers (binary)
- REST typically uses HTTP/1.1 and JSON (text)
- gRPC supports streaming, REST doesn't natively
- gRPC is faster but less browser-friendly
- gRPC has strict contracts, REST is more flexible

#### Q2: What are the 4 types of gRPC communication?

**Answer:**
1. **Unary**: Single request, single response
2. **Server streaming**: Single request, stream of responses
3. **Client streaming**: Stream of requests, single response
4. **Bidirectional**: Both send streams

#### Q3: When would you choose gRPC over REST?

**Answer:**
- Microservices (backend-to-backend)
- Real-time streaming required
- Performance critical
- Polyglot environment
- Mobile apps to backend
- NOT for browser clients or public APIs

#### Q4: How does gRPC achieve better performance?

**Answer:**
- Binary serialization (Protocol Buffers) vs JSON
- HTTP/2 multiplexing (multiple requests on one connection)
- Header compression
- Efficient streaming
- Generated code (no reflection overhead)

#### Q5: How do you handle errors in gRPC?

**Answer:**
- Use status codes (NOT_FOUND, INVALID_ARGUMENT, etc.)
- `context.abort()` on server
- Catch `grpc.RpcError` on client
- Include detailed error messages
- Use metadata for additional context

### System Design Considerations

When designing with gRPC:

1. **Service Boundaries**: Define clear service contracts
2. **Versioning Strategy**: Plan for API evolution
3. **Error Handling**: Comprehensive status codes
4. **Security**: TLS, authentication, authorization
5. **Monitoring**: Request metrics, error rates, latency
6. **Load Balancing**: Client-side or proxy-based
7. **Backward Compatibility**: Careful with proto changes

### Real-World Examples

**Netflix:**
- Uses gRPC for inter-service communication
- Migrated from REST for better performance
- Supports thousands of microservices

**Google:**
- Internal infrastructure built on gRPC
- YouTube, Google Cloud APIs use gRPC
- 10+ billion gRPC calls per second

**Uber:**
- Real-time location tracking with bidirectional streaming
- Microservices communication
- Mobile apps use gRPC for efficiency

---

## Summary

**gRPC** is a high-performance RPC framework ideal for microservices and real-time applications.

**Key Takeaways:**

1. **HTTP/2 + Protocol Buffers**: Binary, fast, efficient
2. **4 Communication Patterns**: Unary, server streaming, client streaming, bidirectional
3. **Strongly Typed**: Contract-first with .proto files
4. **Performance**: 5-10x faster than JSON/REST
5. **Streaming**: Native support for real-time communication
6. **Polyglot**: Multi-language code generation
7. **Not for Browsers**: Limited browser support

**When to Use:**
- ✅ Microservices communication
- ✅ Real-time streaming
- ✅ Mobile to backend
- ✅ High performance requirements
- ❌ Browser-based clients
- ❌ Public APIs (REST more common)

**For Interviews:**
- Understand 4 communication patterns
- Know gRPC vs REST trade-offs
- Be able to write basic .proto files
- Explain performance benefits
- Discuss use cases and limitations
