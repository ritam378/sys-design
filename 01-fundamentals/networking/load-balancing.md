# Load Balancing

## Overview

**Load Balancing** distributes incoming network traffic across multiple servers to ensure no single server is overwhelmed, improving availability, reliability, and performance.

**Key Question:** "How do you distribute traffic across multiple servers efficiently?"

**Short Answer:**
- **Load Balancer** sits between clients and servers
- Distributes requests using algorithms (round-robin, least connections, etc.)
- Provides high availability through health checks
- Operates at Layer 4 (TCP) or Layer 7 (HTTP)

---

## Why Load Balancing?

### Without Load Balancer

```
All traffic → Single Server
              (Overloaded, single point of failure)

┌─────────┐
│ Client  │──┐
└─────────┘  │
             │
┌─────────┐  │      ┌──────────┐
│ Client  │──┼─────►│  Server  │  ← Overwhelmed!
└─────────┘  │      │ (100% CPU)│
             │      └──────────┘
┌─────────┐  │
│ Client  │──┘
└─────────┘

Problems:
❌ Server overload → slow response
❌ Single point of failure
❌ No scalability
```

### With Load Balancer

```
Traffic → Load Balancer → Distributes across servers

┌─────────┐
│ Client  │──┐
└─────────┘  │      ┌──────────────┐
             │      │ Load Balancer │
┌─────────┐  ├─────►│ (Distributor) │
│ Client  │──┤      └───────┬───────┘
└─────────┘  │              │
             │        ┌─────┼─────┐
┌─────────┐  │        │     │     │
│ Client  │──┘        ▼     ▼     ▼
└─────────┘       ┌────┐ ┌────┐ ┌────┐
                  │ S1 │ │ S2 │ │ S3 │
                  │50% │ │50% │ │50% │
                  └────┘ └────┘ └────┘

Benefits:
✅ Evenly distributed load
✅ High availability (failover)
✅ Horizontal scalability
```

---

## Load Balancing Layers

### Layer 4 (Transport Layer)

**Operates at:** TCP/UDP level

**Routes based on:**
- Source/destination IP
- Source/destination Port

**Characteristics:**
- ✅ Fast (no packet inspection)
- ✅ Protocol-agnostic (works with any TCP/UDP)
- ❌ No content-based routing
- ❌ Can't read HTTP headers

**Example:**

```
Client connects to load balancer:
  Client IP: 192.168.1.10:5000
  Load Balancer IP: 10.0.0.1:80

Load balancer forwards to backend:
  Backend IP: 10.0.1.5:8080

Decision based on: IP addresses only
```

### Layer 7 (Application Layer)

**Operates at:** HTTP/HTTPS level

**Routes based on:**
- HTTP headers
- Cookies
- URL paths
- Request methods

**Characteristics:**
- ✅ Content-based routing
- ✅ SSL termination
- ✅ Advanced features (caching, compression)
- ❌ Slower (must parse HTTP)

**Example:**

```
Load balancer inspects HTTP request:

GET /api/users HTTP/1.1
Host: example.com
Cookie: user_id=123

Routes based on:
- Path: /api/* → API servers
- Cookie: user_id=123 → Sticky session to same server
- Header: X-Version=2 → Version 2 servers
```

---

## Load Balancing Algorithms

### 1. Round Robin

**Distributes requests sequentially**

```python
class RoundRobinBalancer:
    def __init__(self, servers: list):
        self.servers = servers
        self.current = 0

    def get_server(self):
        server = self.servers[self.current]
        self.current = (self.current + 1) % len(self.servers)
        return server

# Usage
lb = RoundRobinBalancer(["server1", "server2", "server3"])

lb.get_server()  # → server1
lb.get_server()  # → server2
lb.get_server()  # → server3
lb.get_server()  # → server1 (wraps around)
```

**Pros:**
- ✅ Simple
- ✅ Fair distribution

**Cons:**
- ❌ Ignores server capacity
- ❌ Ignores current load

**When to use:** Servers have equal capacity

---

### 2. Weighted Round Robin

**Assigns more requests to powerful servers**

```python
class WeightedRoundRobinBalancer:
    def __init__(self, servers: dict):
        # servers = {"server1": 5, "server2": 3, "server3": 2}
        # Total weight = 10
        self.weighted_servers = []

        for server, weight in servers.items():
            self.weighted_servers.extend([server] * weight)

        self.current = 0

    def get_server(self):
        server = self.weighted_servers[self.current]
        self.current = (self.current + 1) % len(self.weighted_servers)
        return server

# Usage
lb = WeightedRoundRobinBalancer({
    "server1": 5,  # 50% (high capacity)
    "server2": 3,  # 30%
    "server3": 2   # 20% (low capacity)
})

# Distribution: 5 to S1, 3 to S2, 2 to S3, repeat
```

**When to use:** Servers have different capacities

---

### 3. Least Connections

**Routes to server with fewest active connections**

```python
class LeastConnectionsBalancer:
    def __init__(self, servers: list):
        self.connections = {server: 0 for server in servers}

    def get_server(self):
        # Find server with minimum connections
        server = min(self.connections, key=self.connections.get)
        self.connections[server] += 1
        return server

    def release_connection(self, server):
        self.connections[server] -= 1

# Usage
lb = LeastConnectionsBalancer(["server1", "server2", "server3"])

# Initially: {server1: 0, server2: 0, server3: 0}
lb.get_server()  # → server1 (connections: {server1: 1, ...})
lb.get_server()  # → server2 (connections: {server1: 1, server2: 1, ...})
lb.get_server()  # → server3
lb.get_server()  # → server1 (ties go to first)

lb.release_connection("server1")  # Connection finished
```

**When to use:** Long-lived connections (WebSocket, database connections)

---

### 4. Least Response Time

**Routes to server with lowest response time**

```python
class LeastResponseTimeBalancer:
    def __init__(self, servers: list):
        self.servers = servers
        self.response_times = {server: 0.0 for server in servers}

    def get_server(self):
        return min(self.response_times, key=self.response_times.get)

    def update_response_time(self, server, response_time_ms):
        # Exponential moving average
        alpha = 0.2
        self.response_times[server] = (
            alpha * response_time_ms +
            (1 - alpha) * self.response_times[server]
        )

# Usage
lb = LeastResponseTimeBalancer(["server1", "server2", "server3"])

# After requests:
lb.update_response_time("server1", 50)   # Fast
lb.update_response_time("server2", 200)  # Slow
lb.update_response_time("server3", 100)

lb.get_server()  # → server1 (lowest response time)
```

**When to use:** Servers have varying performance characteristics

---

### 5. IP Hash (Consistent Hashing)

**Routes same client IP to same server**

```python
import hashlib

class IPHashBalancer:
    def __init__(self, servers: list):
        self.servers = servers

    def get_server(self, client_ip: str):
        hash_value = int(hashlib.md5(client_ip.encode()).hexdigest(), 16)
        index = hash_value % len(self.servers)
        return self.servers[index]

# Usage
lb = IPHashBalancer(["server1", "server2", "server3"])

lb.get_server("192.168.1.10")  # → server2
lb.get_server("192.168.1.10")  # → server2 (same server!)
lb.get_server("192.168.1.20")  # → server1 (different client)
```

**Pros:**
- ✅ Session affinity (sticky sessions)
- ✅ Cache hits (same user → same server → cached data)

**Cons:**
- ❌ Uneven distribution if few clients
- ❌ All requests from same client to one server

**When to use:** Stateful applications, session management

---

## Health Checks

**Ensure requests only go to healthy servers**

```python
import asyncio
import aiohttp

class HealthCheckBalancer:
    def __init__(self, servers: list, check_interval: int = 10):
        self.servers = servers
        self.healthy = {server: True for server in servers}
        self.check_interval = check_interval

    async def health_check_loop(self):
        while True:
            await self.check_all_servers()
            await asyncio.sleep(self.check_interval)

    async def check_all_servers(self):
        tasks = [self.check_server(server) for server in self.servers]
        await asyncio.gather(*tasks)

    async def check_server(self, server):
        try:
            async with aiohttp.ClientSession() as session:
                async with session.get(
                    f"http://{server}/health",
                    timeout=aiohttp.ClientTimeout(total=2)
                ) as response:
                    if response.status == 200:
                        self.healthy[server] = True
                    else:
                        self.healthy[server] = False
        except Exception:
            self.healthy[server] = False

    def get_healthy_servers(self):
        return [s for s in self.servers if self.healthy[s]]

    def get_server(self):
        healthy_servers = self.get_healthy_servers()

        if not healthy_servers:
            raise Exception("No healthy servers available")

        # Use round-robin among healthy servers
        return healthy_servers[0]

# Usage
lb = HealthCheckBalancer(["server1", "server2", "server3"])

# Start health checks
asyncio.create_task(lb.health_check_loop())

# Only routes to healthy servers
server = lb.get_server()  # Won't route to failed servers
```

**Health Check Types:**

1. **HTTP/HTTPS:** GET /health → 200 OK
2. **TCP:** Connect to port → Success
3. **Custom:** Application-specific logic

---

## Real-World Load Balancers

### Hardware Load Balancers

**F5 Big-IP, Citrix ADC**

**Pros:**
- ✅ Very high performance (millions QPS)
- ✅ Dedicated hardware

**Cons:**
- ❌ Expensive ($10K-$100K+)
- ❌ Vendor lock-in

### Software Load Balancers

**1. Nginx**

```nginx
http {
    upstream backend {
        least_conn;  # Algorithm

        server backend1.example.com weight=5;
        server backend2.example.com;
        server backend3.example.com;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend;
        }
    }
}
```

**2. HAProxy**

```
frontend http-in
    bind *:80
    default_backend servers

backend servers
    balance roundrobin
    option httpchk GET /health
    server server1 192.168.1.10:8080 check
    server server2 192.168.1.11:8080 check
    server server3 192.168.1.12:8080 check
```

**3. Cloud Load Balancers**

- **AWS ELB** (Elastic Load Balancer)
- **Google Cloud Load Balancing**
- **Azure Load Balancer**

---

## Advanced Features

### SSL Termination

**Load balancer handles SSL/TLS encryption**

```
Client ──HTTPS──► Load Balancer ──HTTP──► Backend Servers
         (encrypted)              (unencrypted)

Benefits:
✅ Offload CPU-intensive encryption from backends
✅ Centralized certificate management
✅ Backends can focus on business logic
```

### Session Persistence (Sticky Sessions)

**Same client always routes to same server**

```python
# Cookie-based
Set-Cookie: SERVER_ID=server2; Path=/

# All subsequent requests with this cookie → server2
```

**Methods:**
1. **Cookie-based:** Load balancer sets cookie
2. **IP-based:** Hash client IP
3. **Application session ID:** Use session token

---

## Comparison Table

| Algorithm | Use Case | Pros | Cons |
|-----------|----------|------|------|
| **Round Robin** | Equal servers | Simple | Ignores load |
| **Weighted RR** | Different capacities | Fair distribution | Static weights |
| **Least Connections** | Long connections | Dynamic load balancing | Overhead |
| **Least Response Time** | Varying performance | Performance-aware | Requires monitoring |
| **IP Hash** | Stateful apps | Session affinity | Uneven distribution |

---

## Summary

**Load Balancing** is essential for:
- ✅ High availability (failover)
- ✅ Scalability (distribute load)
- ✅ Performance (optimal routing)

**Key Concepts:**
1. **Layer 4 vs Layer 7** - Speed vs Features
2. **Algorithms** - Round-robin, least connections, IP hash
3. **Health checks** - Only route to healthy servers
4. **SSL termination** - Offload encryption

**Popular tools:** Nginx, HAProxy, AWS ELB, Google Cloud LB

**Next:** Learn about [Cache Invalidation](../caching/cache-invalidation.md) strategies.
