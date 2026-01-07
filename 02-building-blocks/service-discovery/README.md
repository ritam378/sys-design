# Service Discovery - System Design Building Block

**Difficulty:** Intermediate-Advanced
**Category:** Distributed Systems, Microservices
**Use Cases:** Microservices, Load Balancing, Auto-scaling, Cloud Infrastructure
**Key Concepts:** Dynamic Registration, Health Checks, Service Registry

---

## What is Service Discovery?

**Service Discovery** is a mechanism that allows services in a distributed system to find and communicate with each other without hardcoding network locations.

**Problem it Solves:**
```
❌ Hardcoded IPs:
  PaymentService → http://192.168.1.50:8080/inventory

✅ Service Discovery:
  PaymentService → "inventory-service" → Discover → http://192.168.1.50:8080
```

**Why Needed:** In cloud/microservices, IPs change dynamically (auto-scaling, failures, deployments)

---

## Core Components

### 1. Service Registry
**Central database of available services**

```
Registry:
├── inventory-service
│   ├── Instance 1: 192.168.1.50:8080 (healthy)
│   ├── Instance 2: 192.168.1.51:8080 (healthy)
│   └── Instance 3: 192.168.1.52:8080 (unhealthy)
├── payment-service
│   └── Instance 1: 192.168.1.60:9000 (healthy)
└── user-service
    ├── Instance 1: 192.168.1.70:7000 (healthy)
    └── Instance 2: 192.168.1.71:7000 (healthy)
```

### 2. Registration
**Services register themselves on startup**

```
Service starts → Register with registry:
POST /registry/register
{
  "service_name": "inventory-service",
  "host": "192.168.1.50",
  "port": 8080,
  "health_check_url": "/health"
}
```

### 3. Discovery
**Clients query registry to find services**

```
Client needs inventory-service:
GET /registry/discover/inventory-service

Response:
[
  {"host": "192.168.1.50", "port": 8080},
  {"host": "192.168.1.51", "port": 8080}
]

Client picks one (round-robin, random, etc.)
```

### 4. Health Checks
**Registry monitors service health**

```
Every 10 seconds:
  For each registered service:
    Call health_check_url
    If success: Mark healthy
    If fail 3 times: Mark unhealthy, remove from discovery
```

### 5. De-registration
**Services unregister on shutdown**

```
Service shutdown → Unregister:
DELETE /registry/unregister/inventory-service/192.168.1.50:8080

Or: Use heartbeats (if heartbeat stops, auto-remove)
```

---

## Service Discovery Patterns

### 1. Client-Side Discovery

**Flow:**
```
1. Client queries Service Registry
2. Registry returns list of instances
3. Client picks one (load balancing logic in client)
4. Client makes direct request to service
```

**Pros:**
- Simple registry (just lookup)
- Client controls load balancing
- One less network hop

**Cons:**
- Client logic more complex
- Each client needs discovery library
- Client needs to handle failures

**Example:** Netflix Eureka with Ribbon

### 2. Server-Side Discovery

**Flow:**
```
1. Client requests Load Balancer
2. Load Balancer queries Service Registry
3. Load Balancer picks instance
4. Load Balancer forwards request
5. Service responds through Load Balancer
```

**Pros:**
- Simple clients
- Centralized load balancing
- Easy to change LB strategy

**Cons:**
- Load balancer is single point of failure
- Extra network hop
- Load balancer can be bottleneck

**Example:** AWS ELB, Kubernetes Services

---

## Popular Tools

| Tool | Type | Key Features | Best For |
|------|------|--------------|----------|
| **Consul** | Both | Service mesh, KV store, health checks | Full-featured microservices |
| **Eureka** | Client-side | Netflix OSS, Java-focused | Spring Cloud apps |
| **Zookeeper** | Both | Strong consistency, leader election | Hadoop ecosystem |
| **etcd** | Both | Kubernetes native, Raft consensus | Kubernetes, CoreOS |
| **Kubernetes DNS** | Server-side | Built-in, DNS-based | Kubernetes clusters |

---

## Health Check Strategies

### 1. **HTTP Health Endpoint**
```
GET /health
Response: 200 OK {"status": "healthy"}

Registry calls this every 10s
If 3 consecutive failures → mark unhealthy
```

### 2. **TCP Connection Check**
```
Try to open TCP connection to service port
If successful → healthy
If refused/timeout → unhealthy
```

### 3. **Heartbeat**
```
Service sends heartbeat every 10s:
POST /heartbeat {"service": "inventory-service", "instance": "192.168.1.50:8080"}

If no heartbeat for 30s → mark unhealthy
```

### 4. **Custom Health Logic**
```
Health endpoint checks:
- Database connectivity
- Disk space
- Memory usage
- Downstream dependencies

Return 200 only if all checks pass
```

---

## Load Balancing Algorithms

When multiple instances exist, how to choose?

| Algorithm | How It Works | Use Case |
|-----------|--------------|----------|
| **Round Robin** | Rotate through instances | Equal servers, simple |
| **Random** | Pick random instance | Stateless services |
| **Least Connections** | Send to server with fewest connections | Long-running requests |
| **Weighted** | Assign weights to instances | Heterogeneous hardware |
| **IP Hash** | Hash client IP to instance | Session affinity |
| **Least Response Time** | Send to fastest instance | Performance-sensitive |

---

## Service Registry Data Model

### Basic Schema

```json
{
  "services": {
    "inventory-service": {
      "instances": [
        {
          "instance_id": "inv-001",
          "host": "192.168.1.50",
          "port": 8080,
          "status": "healthy",
          "last_heartbeat": "2024-01-07T10:30:00Z",
          "metadata": {
            "version": "1.2.0",
            "region": "us-west",
            "weight": 100
          }
        },
        {
          "instance_id": "inv-002",
          "host": "192.168.1.51",
          "port": 8080,
          "status": "healthy",
          "last_heartbeat": "2024-01-07T10:30:05Z",
          "metadata": {
            "version": "1.2.0",
            "region": "us-west",
            "weight": 100
          }
        }
      ]
    }
  }
}
```

---

## Advanced Features

### 1. **Service Mesh Integration**
```
Service Discovery + Service Mesh (Istio, Linkerd)
- Automatic service registration
- Intelligent routing
- Circuit breaking
- Observability
```

### 2. **Multi-Region Discovery**
```
Prefer local region instances:
1. Check local region registry
2. If not available, check other regions
3. Consider latency in selection
```

### 3. **Canary Deployments**
```
Metadata: {"version": "v1", "traffic": 90%}
Metadata: {"version": "v2", "traffic": 10%}

Route 10% traffic to new version
Monitor metrics
Gradually increase if stable
```

### 4. **Blue-Green Deployment**
```
Blue instances: version 1.0 (active)
Green instances: version 2.0 (standby)

Test green → Switch traffic → Blue becomes standby
```

---

## Failure Scenarios

### 1. **Service Registry Down**

**Problem:** Services can't discover each other

**Solutions:**
- **Client-side caching:** Cache discovered instances
- **Fallback:** Use last known good configuration
- **Multiple registries:** Run registry cluster (Consul, etcd)

### 2. **Stale Data**

**Problem:** Registry returns unhealthy instance

**Solutions:**
- **Aggressive health checks:** Check frequently
- **Client-side retries:** Try another instance on failure
- **Circuit breakers:** Detect and skip bad instances

### 3. **Network Partition**

**Problem:** Registry can't reach service, but service is healthy

**Solutions:**
- **Grace period:** Don't immediately remove
- **Heartbeat from service:** Service actively reports health
- **Quorum-based decisions:** Multiple registry nodes agree

---

## Implementation Example (Conceptual)

### Service Registration
```python
class ServiceRegistry:
    def register(self, service_name, instance_info):
        if service_name not in self.services:
            self.services[service_name] = []

        instance = {
            "id": generate_id(),
            "host": instance_info["host"],
            "port": instance_info["port"],
            "status": "healthy",
            "last_seen": now()
        }

        self.services[service_name].append(instance)
        self.start_health_check(instance)

    def discover(self, service_name):
        instances = self.services.get(service_name, [])
        healthy = [i for i in instances if i["status"] == "healthy"]
        return healthy
```

### Client Discovery
```python
class ServiceClient:
    def call(self, service_name, endpoint):
        instances = registry.discover(service_name)
        if not instances:
            raise ServiceUnavailableError()

        instance = self.load_balancer.select(instances)
        url = f"http://{instance['host']}:{instance['port']}{endpoint}"

        try:
            return requests.get(url)
        except RequestException:
            # Retry with different instance
            return self.call(service_name, endpoint)
```

---

## Comparison: DNS vs Service Discovery

| Feature | DNS | Service Discovery |
|---------|-----|-------------------|
| **Update speed** | Slow (TTL, caching) | Fast (real-time) |
| **Health checks** | No | Yes |
| **Load balancing** | Basic (round-robin) | Advanced algorithms |
| **Metadata** | Limited | Rich (version, region, tags) |
| **Complexity** | Simple | More complex |
| **Best for** | Stable services | Dynamic microservices |

---

## Interview Questions

### Conceptual
1. **Q:** What problem does service discovery solve?
   - **A:** Dynamically find service instances without hardcoded IPs, handle auto-scaling and failures

2. **Q:** Client-side vs server-side discovery?
   - **A:** Client-side: client queries registry and picks instance; Server-side: load balancer does discovery

3. **Q:** How do you handle service registry failure?
   - **A:** Client-side caching, multiple registry replicas, fallback to last known config

### Technical
4. **Q:** Design a health check system?
   - **A:** HTTP /health endpoint, poll every 10s, mark unhealthy after 3 failures, auto-remove after 30s

5. **Q:** How to prevent thundering herd when new instances register?
   - **A:** Gradual traffic increase, warm-up period, health check grace period

6. **Q:** Implement service discovery with caching?
   - **A:** Cache discovered instances, TTL 30s, background refresh, invalidate on failure

### System Design
7. **Q:** Design service discovery for multi-region deployment?
   - **A:** Registry per region, prefer local region, fallback to other regions, consider latency

8. **Q:** How would you implement canary deployments with service discovery?
   - **A:** Tag instances with version, metadata with traffic percentage, route based on weights

---

## Key Takeaways

### When to Use
✅ Microservices architecture
✅ Auto-scaling environments
✅ Cloud-native applications
✅ Container orchestration (Kubernetes)

### When NOT to Use
❌ Monolithic applications
❌ Static infrastructure (fixed IPs)
❌ Very small deployments (2-3 services)
❌ Services that rarely change

---

## Summary

**In One Sentence:** Service discovery enables services in distributed systems to dynamically find and communicate with each other without hardcoded network locations.

**Key Benefits:**
1. **Dynamic:** Handle auto-scaling, failures, deployments
2. **Resilient:** Health checks remove unhealthy instances
3. **Flexible:** Load balancing, routing, canary deployments
4. **Decoupled:** Services don't need to know each other's locations

**Popular Pattern:**
```
Service → Register → Registry → Discover ← Client
              ↓
        Health Checks
```

---

**Pro Tip for Interviews:** Always mention the dynamic nature (IPs change), health checks (detect failures), and compare client-side vs server-side discovery. Give examples: Netflix Eureka, Kubernetes DNS, Consul. Discuss trade-offs between consistency (Zookeeper) and availability (Eureka). Mention how it enables auto-scaling and canary deployments!
