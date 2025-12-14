# URL Shortener Design (TinyURL, bit.ly)

> **Difficulty:** Beginner
> **Interview Frequency:** Very High
> **Key Concepts:** Hashing, Base62 encoding, Database indexing, Caching, Rate limiting

## Table of Contents
1. [Problem Statement & Requirements](#1-problem-statement--requirements)
2. [Back-of-the-Envelope Estimation](#2-back-of-the-envelope-estimation)
3. [API Design](#3-api-design)
4. [Data Model & Database Schema](#4-data-model--database-schema)
5. [High-Level Design](#5-high-level-design)
6. [Detailed Component Design](#6-detailed-component-design)
7. [Identifying and Resolving Bottlenecks](#7-identifying-and-resolving-bottlenecks)
8. [Trade-offs and Alternatives](#8-trade-offs-and-alternatives)
9. [Monitoring, Metrics & Alerts](#9-monitoring-metrics--alerts)
10. [Follow-up Questions & Extensions](#10-follow-up-questions--extensions)

---

## 1. Problem Statement & Requirements

### Problem Description
Design a URL shortening service like TinyURL or bit.ly that converts long URLs into short, manageable links. When users access the short URL, they should be redirected to the original long URL.

**Example:**
- Long URL: `https://www.example.com/very/long/path/to/resource?param1=value1&param2=value2`
- Short URL: `https://short.ly/abc123`

### Functional Requirements

**Core Features:**
- [x] **Create short URL:** Given a long URL, return a unique short URL
- [x] **Redirect:** When user visits short URL, redirect to original long URL
- [x] **Custom aliases:** Users can optionally provide custom short URLs
- [x] **Expiration:** URLs can have TTL (time-to-live)
- [x] **Analytics:** Track click count and basic analytics

**Additional Features:**
- [x] **User accounts:** Users can manage their URLs
- [x] **URL validation:** Validate URLs before shortening
- [x] **QR code generation:** Generate QR codes for short URLs

### Non-Functional Requirements

**Scale:**
- **100 million** URLs created per month
- **10 billion** redirects per month (100:1 read/write ratio)
- **500 million** total users
- **100 million** DAU (Daily Active Users)

**Performance:**
- Latency: < 200ms for redirection (99th percentile)
- Availability: 99.99% uptime

**Other:**
- [x] **Highly available** - Service should not have downtime
- [x] **Scalable** - Should handle traffic spikes
- [x] **Short URLs should be unpredictable** - For security
- [x] **Data durability** - URLs should not be lost

### Out of Scope

- ❌ URL editing after creation
- ❌ Complex user management (OAuth, SSO)
- ❌ Detailed analytics (geo-location, device type, browser)
- ❌ Link preview generation

### Constraints and Assumptions

**Assumptions:**
- **Read-heavy system:** 100:1 read/write ratio
- **URL length:** Average 100 characters for long URL
- **Short URL length:** 7 characters (62^7 = 3.5 trillion combinations)
- **Data retention:** 5 years
- **Traffic is not evenly distributed:** 80/20 rule applies

---

## 2. Back-of-the-Envelope Estimation

### Traffic Estimation

#### Write Operations (URL Creation)
```
URLs created per month: 100 million
URLs created per day: 100M / 30 ≈ 3.3 million
URLs created per second (QPS): 3.3M / 100,000 ≈ 33 QPS
Peak write QPS: 33 × 2 = 66 QPS
```

#### Read Operations (URL Redirection)
```
Read/write ratio: 100:1
Redirects per month: 100M × 100 = 10 billion
Redirects per day: 10B / 30 ≈ 333 million
Redirect QPS: 333M / 100,000 ≈ 3,300 QPS
Peak redirect QPS: 3,300 × 2 = 6,600 QPS
```

### Storage Estimation

```
Data per URL:
- Long URL: 100 bytes (average)
- Short URL: 7 bytes
- User ID: 8 bytes (BIGINT)
- Created timestamp: 8 bytes
- Expiration: 8 bytes
- Metadata (JSON): 50 bytes
- Total per record: ~180 bytes

Total URLs over 5 years:
- Per month: 100 million
- Per year: 100M × 12 = 1.2 billion
- Over 5 years: 1.2B × 5 = 6 billion URLs

Total storage:
- 6 billion × 180 bytes = 1,080 GB ≈ 1.1 TB

With overhead (indexes, replication):
- 1.1 TB × 3 = 3.3 TB
```

### Bandwidth Estimation

```
Write Bandwidth (incoming):
- 33 requests/sec × 180 bytes = 5,940 bytes/sec ≈ 6 KB/s

Read Bandwidth (outgoing):
- 3,300 requests/sec × 180 bytes = 594,000 bytes/sec ≈ 600 KB/s

Total bandwidth: ~606 KB/s (very low)
```

### Memory/Cache Estimation

```
Using 80/20 rule (80% of traffic from 20% of URLs):

Daily redirects: 333 million
Unique URLs (20%): 333M × 0.2 = 66.6 million URLs per day

Cache size needed:
- 66.6M × 180 bytes ≈ 12 GB

For safety, provision 20-30 GB of cache
Use Redis or Memcached for caching
```

### Summary Table

| Metric | Estimate |
|--------|----------|
| **Write QPS (avg)** | 33 |
| **Write QPS (peak)** | 66 |
| **Read QPS (avg)** | 3,300 |
| **Read QPS (peak)** | 6,600 |
| **Storage (5 years)** | 3.3 TB |
| **Bandwidth (in)** | 6 KB/s |
| **Bandwidth (out)** | 600 KB/s |
| **Cache size** | 20-30 GB |

---

## 3. API Design

### REST API Endpoints

#### 1. Create Short URL
```http
POST /api/v1/shorten
Content-Type: application/json
Authorization: Bearer <token>

Request:
{
  "long_url": "https://www.example.com/very/long/url",
  "custom_alias": "mylink",  // optional
  "expiration_date": "2025-12-31T23:59:59Z"  // optional
}

Response (201 Created):
{
  "short_url": "https://short.ly/abc123",
  "long_url": "https://www.example.com/very/long/url",
  "created_at": "2024-01-15T10:30:00Z",
  "expiration_date": "2025-12-31T23:59:59Z"
}

Error Response (400 Bad Request):
{
  "error": "invalid_url",
  "message": "The provided URL is not valid"
}

Error Response (409 Conflict):
{
  "error": "alias_taken",
  "message": "The custom alias is already in use"
}
```

#### 2. Redirect to Long URL
```http
GET /{short_code}

Response (301 Moved Permanently or 302 Found):
Location: https://www.example.com/very/long/url

Error Response (404 Not Found):
{
  "error": "not_found",
  "message": "Short URL not found or expired"
}
```

#### 3. Get URL Analytics
```http
GET /api/v1/analytics/{short_code}
Authorization: Bearer <token>

Response (200 OK):
{
  "short_url": "https://short.ly/abc123",
  "long_url": "https://www.example.com/very/long/url",
  "created_at": "2024-01-15T10:30:00Z",
  "click_count": 1523,
  "last_accessed": "2024-01-20T15:45:00Z"
}
```

#### 4. Delete Short URL
```http
DELETE /api/v1/urls/{short_code}
Authorization: Bearer <token>

Response (204 No Content)

Error Response (404 Not Found):
{
  "error": "not_found",
  "message": "Short URL not found"
}
```

### API Design Considerations

- **Versioning:** `/api/v1/` for future-proofing
- **Authentication:** JWT tokens for user-specific operations
- **Rate Limiting:** 100 requests per minute per user (creation), unlimited redirects
- **Idempotency:** Creating same long URL returns existing short URL
- **HTTPS Only:** All API calls must use HTTPS

---

## 4. Data Model & Database Schema

### Database Choice

**Selected Database:** PostgreSQL (or MySQL)

**Justification:**
- **ACID compliance** - Important for URL creation consistency
- **Strong indexing** - Fast lookups by short URL
- **Mature and battle-tested** - Used by many similar services
- **Relational data** - User-URL relationships

**Alternative:** NoSQL like Cassandra for even higher scale, but adds complexity

### Schema Design

#### Table 1: `urls`
```sql
CREATE TABLE urls (
    id BIGSERIAL PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    user_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expiration_date TIMESTAMP,
    click_count BIGINT DEFAULT 0,
    last_accessed TIMESTAMP,

    INDEX idx_short_code (short_code),
    INDEX idx_user_id (user_id),
    INDEX idx_created_at (created_at)
);
```

**Columns:**
- `id` - Primary key, auto-incrementing
- `short_code` - The short URL identifier (unique, indexed)
- `long_url` - Original URL (text to support very long URLs)
- `user_id` - Foreign key to users table (nullable for anonymous)
- `created_at` - Creation timestamp
- `expiration_date` - Optional expiration
- `click_count` - Number of times URL was accessed
- `last_accessed` - Last access timestamp

**Indexes:**
- **Primary index on `id`** - Auto-created with PRIMARY KEY
- **Unique index on `short_code`** - Fast lookup for redirects (most common query)
- **Index on `user_id`** - Fast lookup of user's URLs
- **Index on `created_at`** - For analytics and cleanup

#### Table 2: `users`
```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_email (email)
);
```

#### Table 3: `url_analytics` (Optional - for detailed analytics)
```sql
CREATE TABLE url_analytics (
    id BIGSERIAL PRIMARY KEY,
    url_id BIGINT NOT NULL,
    accessed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address VARCHAR(45),
    user_agent TEXT,

    FOREIGN KEY (url_id) REFERENCES urls(id),
    INDEX idx_url_id (url_id),
    INDEX idx_accessed_at (accessed_at)
);
```

### Data Model Diagram

```mermaid
erDiagram
    USERS ||--o{ URLS : creates
    URLS ||--o{ URL_ANALYTICS : tracks

    USERS {
        bigint id PK
        varchar email
        varchar password_hash
        timestamp created_at
    }

    URLS {
        bigint id PK
        varchar short_code UK
        text long_url
        bigint user_id FK
        timestamp created_at
        timestamp expiration_date
        bigint click_count
        timestamp last_accessed
    }

    URL_ANALYTICS {
        bigint id PK
        bigint url_id FK
        timestamp accessed_at
        varchar ip_address
        text user_agent
    }
```

### Sharding Strategy

**For larger scale (beyond single DB):**

- **Sharding Key:** First character of `short_code`
- **Partitioning Method:** Hash-based on short_code
- **Number of Shards:** Start with 10, can grow to 100+

**Example:**
- Shard 1: short_codes starting with [0-9a-f]
- Shard 2: short_codes starting with [g-m]
- Shard 3: short_codes starting with [n-z, A-Z]

---

## 5. High-Level Design

### Architecture Diagram

```mermaid
graph TB
    Client[Client/Browser]
    CDN[CDN for Static Assets]
    LB[Load Balancer]
    API1[API Server 1]
    API2[API Server 2]
    API3[API Server N]
    Cache[(Redis Cache)]
    ZK[ZooKeeper<br/>ID Generator]
    DB[(Primary Database<br/>PostgreSQL)]
    Replica1[(Read Replica 1)]
    Replica2[(Read Replica 2)]
    Analytics[Analytics Service]
    Queue[Message Queue<br/>Kafka]

    Client --> CDN
    CDN --> LB
    LB --> API1
    LB --> API2
    LB --> API3

    API1 --> Cache
    API2 --> Cache
    API3 --> Cache

    API1 --> DB
    API2 --> DB
    API3 --> DB

    API1 --> Replica1
    API2 --> Replica1
    API3 --> Replica2

    API1 --> ZK
    API2 --> ZK
    API3 --> ZK

    API1 --> Queue
    API2 --> Queue
    API3 --> Queue

    Queue --> Analytics
    Analytics --> DB

    style Client fill:#f9f,stroke:#333
    style LB fill:#bbf,stroke:#333
    style API1 fill:#bfb,stroke:#333
    style API2 fill:#bfb,stroke:#333
    style API3 fill:#bfb,stroke:#333
    style Cache fill:#fbb,stroke:#333
    style DB fill:#ffb,stroke:#333
```

### Component Overview

1. **Client:** Web browser or mobile app
2. **CDN:** Serves static content (HTML, CSS, JS)
3. **Load Balancer:** Layer 7 load balancer (NGINX/HAProxy)
4. **API Servers:** Stateless application servers (Node.js/Go/Java)
5. **Redis Cache:** Caches hot short URL mappings
6. **ZooKeeper:** Distributed ID generation coordination
7. **Primary Database:** Writes go here
8. **Read Replicas:** Distribute read load
9. **Message Queue:** Async analytics processing
10. **Analytics Service:** Processes click events

### Data Flow

#### Create Short URL Flow

```mermaid
sequenceDiagram
    participant Client
    participant LB as Load Balancer
    participant API as API Server
    participant ZK as ZooKeeper
    participant DB as Database
    participant Cache

    Client->>LB: POST /api/v1/shorten
    LB->>API: Route request
    API->>API: Validate URL

    alt Custom Alias Provided
        API->>DB: Check if alias exists
        alt Alias Available
            API->>DB: Insert with custom alias
        else Alias Taken
            API-->>Client: 409 Conflict
        end
    else Generate Short Code
        API->>ZK: Get unique ID
        ZK-->>API: Return ID
        API->>API: Encode ID to base62
        API->>DB: Insert URL mapping
    end

    DB-->>API: Confirm insert
    API->>Cache: Cache the mapping
    API-->>LB: Return short URL
    LB-->>Client: 201 Created
```

#### Redirect Flow (Hot Path - Most Critical)

```mermaid
sequenceDiagram
    participant Client
    participant LB as Load Balancer
    participant API as API Server
    participant Cache
    participant DB as Read Replica
    participant Queue as Message Queue

    Client->>LB: GET /abc123
    LB->>API: Route request
    API->>Cache: Get long_url for 'abc123'

    alt Cache Hit
        Cache-->>API: Return long_url
    else Cache Miss
        API->>DB: SELECT long_url WHERE short_code='abc123'
        DB-->>API: Return long_url
        API->>Cache: Store in cache
    end

    API->>Queue: Log click event (async)
    API-->>LB: 301 Redirect to long_url
    LB-->>Client: 301 Redirect
    Client->>Client: Navigate to long_url
```

---

## 6. Detailed Component Design

### Component 1: Short URL Generation

**Purpose:** Generate unique, short, unpredictable identifiers

**Two Approaches:**

#### Approach A: Hash-based Generation

**Algorithm:**
```
1. Compute hash of long URL: hash = SHA256(long_url)
2. Take first 43 bits of hash
3. Encode to base62 (gives 7 characters)
4. Check for collision in database
5. If collision, append counter and re-hash
```

**Pros:**
- Same long URL always generates same short URL (idempotent)
- Simple to implement

**Cons:**
- Hash collisions require handling
- Potential for hash collision attacks

#### Approach B: Auto-incrementing ID with Base62 Encoding (Chosen)

**Algorithm:**
```python
def id_to_short_code(id: int) -> str:
    """
    Convert numeric ID to base62 short code

    Base62: [0-9a-zA-Z] = 62 characters
    7 characters = 62^7 = 3.5 trillion combinations
    """
    chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    base = len(chars)
    short_code = []

    while id > 0:
        short_code.append(chars[id % base])
        id //= base

    return ''.join(reversed(short_code)).rjust(7, '0')

# Example:
# ID 1 -> "0000001"
# ID 62 -> "000001a"
# ID 125 -> "0000021"
```

**Pros:**
- No collisions (unique IDs)
- Predictable length
- Simple and fast

**Cons:**
- Sequential IDs are predictable (can be mitigated with random offset)
- Same long URL creates different short URLs each time

**Making IDs Unpredictable:**
```python
# Use ZooKeeper or distributed counter with random offset
RANDOM_OFFSET = 10000000  # Start IDs from random number

def get_next_id():
    id = zookeeper.get_and_increment("url_counter")
    return id + RANDOM_OFFSET
```

**Code Sample (Python):**
```python
class URLShortener:
    BASE62 = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

    def __init__(self):
        self.base = len(self.BASE62)

    def encode(self, id: int) -> str:
        """Encode integer ID to base62 string"""
        if id == 0:
            return self.BASE62[0]

        result = []
        while id > 0:
            result.append(self.BASE62[id % self.base])
            id //= self.base

        return ''.join(reversed(result)).rjust(7, '0')

    def decode(self, short_code: str) -> int:
        """Decode base62 string to integer ID"""
        id = 0
        for char in short_code:
            id = id * self.base + self.BASE62.index(char)
        return id

# Usage:
shortener = URLShortener()
short_code = shortener.encode(123456789)  # "8M0kX"
original_id = shortener.decode(short_code)  # 123456789
```

### Component 2: Caching Strategy

**Purpose:** Minimize database hits for popular URLs (80/20 rule)

**Implementation:**
- **Cache:** Redis (single-threaded, fast, supports TTL)
- **Cache Key:** `url:{short_code}` → `{long_url}`
- **TTL:** 24 hours (cache invalidation on expiration)
- **Eviction Policy:** LRU (Least Recently Used)

**Cache-Aside Pattern:**
```python
async def get_long_url(short_code: str) -> str:
    # Try cache first
    cache_key = f"url:{short_code}"
    long_url = await redis.get(cache_key)

    if long_url:
        return long_url  # Cache hit

    # Cache miss - query database
    url_record = await db.query(
        "SELECT long_url, expiration_date FROM urls WHERE short_code = ?",
        short_code
    )

    if not url_record:
        raise URLNotFoundError()

    # Check expiration
    if url_record.expiration_date and url_record.expiration_date < now():
        raise URLExpiredError()

    # Store in cache with TTL
    await redis.setex(cache_key, 86400, url_record.long_url)

    return url_record.long_url
```

**Cache Warming:**
- Pre-load top 1000 most popular URLs on startup
- Use analytics data to identify hot URLs

### Component 3: Rate Limiting

**Purpose:** Prevent abuse and ensure fair usage

**Implementation:** Token Bucket Algorithm

```python
from time import time

class RateLimiter:
    def __init__(self, redis_client, max_requests=100, window_seconds=60):
        self.redis = redis_client
        self.max_requests = max_requests
        self.window = window_seconds

    async def is_allowed(self, user_id: str) -> bool:
        """Check if user is within rate limit"""
        key = f"ratelimit:{user_id}"
        now = int(time())
        window_start = now - self.window

        # Use Redis sorted set for sliding window
        pipe = self.redis.pipeline()

        # Remove old entries
        pipe.zremrangebyscore(key, 0, window_start)

        # Count requests in current window
        pipe.zcard(key)

        # Add current request
        pipe.zadd(key, {str(now): now})

        # Set expiration
        pipe.expire(key, self.window)

        results = await pipe.execute()
        request_count = results[1]

        return request_count < self.max_requests
```

### Component 4: Analytics Processing

**Purpose:** Track clicks without slowing down redirects

**Implementation:** Asynchronous event processing

```python
# In API server (non-blocking)
async def redirect(short_code: str):
    long_url = await get_long_url(short_code)

    # Fire-and-forget analytics event
    await kafka.produce('url_clicks', {
        'short_code': short_code,
        'timestamp': now(),
        'ip': request.ip,
        'user_agent': request.headers['User-Agent']
    })

    return RedirectResponse(long_url, status_code=301)

# In analytics service (separate process)
async def process_click_events():
    async for event in kafka.consume('url_clicks'):
        await db.execute(
            "UPDATE urls SET click_count = click_count + 1, "
            "last_accessed = ? WHERE short_code = ?",
            event['timestamp'], event['short_code']
        )
```

---

## 7. Identifying and Resolving Bottlenecks

### Potential Bottlenecks

#### 1. Single Point of Failure (SPOF)

**Problem:** Single database, single cache, single load balancer

**Solutions:**

**Database:**
- Primary-Replica setup (1 primary for writes, multiple replicas for reads)
- Automatic failover with tools like Patroni or ProxySQL
- Cross-region replication for disaster recovery

**Cache:**
- Redis Cluster with multiple nodes
- Redis Sentinel for automatic failover
- Consistent hashing for distributing keys

**Load Balancer:**
- Multiple load balancers in active-passive or active-active mode
- DNS-based failover

#### 2. Database Write Bottleneck

**Problem:** High write traffic overwhelms single primary database

**Solutions:**

**Approach 1: Sharding**
```
Partition data by short_code prefix
- Shard 1: [0-9a-f]*
- Shard 2: [g-m]*
- Shard 3: [n-zA-Z]*

Each shard handles 1/3 of traffic
```

**Approach 2: Database Optimization**
- Use batch inserts for better throughput
- Optimize indexes (only index what's necessary)
- Use connection pooling
- Increase write capacity (vertical scaling)

#### 3. Hot Key Problem in Cache

**Problem:** A few viral URLs get millions of requests, overwhelming cache

**Solutions:**
- **Local cache** on each API server for extremely hot keys
- **Replica caching** - Cache same key on multiple Redis instances
- **CDN caching** - Cache 301 redirects at CDN level for viral links

#### 4. ID Generation Bottleneck

**Problem:** Centralized ID generator becomes bottleneck

**Solutions:**

**Approach 1: Twitter Snowflake**
```
64-bit ID structure:
- 1 bit: unused
- 41 bits: timestamp (milliseconds since epoch)
- 10 bits: machine ID (1024 machines)
- 12 bits: sequence number (4096 IDs per millisecond per machine)

Each server generates IDs independently, no coordination needed
```

**Approach 2: Range-based allocation**
```
Server 1: IDs 1-1,000,000
Server 2: IDs 1,000,001-2,000,000
Server 3: IDs 2,000,001-3,000,000

Servers request new ranges from ZooKeeper when exhausted
```

### Fault Tolerance

**Strategies:**

1. **Replication:** 3x replication for all data (primary + 2 replicas)
2. **Health Checks:** Load balancer health checks every 5 seconds
3. **Circuit Breakers:** Prevent cascading failures when DB is down
4. **Graceful Degradation:** Return cached data even if stale when DB is unavailable
5. **Retry Logic:** Retry failed DB queries with exponential backoff

**Example Circuit Breaker:**
```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN

    async def call(self, func):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitBreakerOpenError()

        try:
            result = await func()
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise e

    def on_success(self):
        self.failure_count = 0
        self.state = "CLOSED"

    def on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"
```

---

## 8. Trade-offs and Alternatives

### Design Decision 1: Base62 Encoding vs Hashing

**Chosen Approach:** Base62 encoding of auto-incrementing IDs

**Rationale:**
- No collision handling needed
- Simple and predictable
- Fast encoding/decoding

**Alternatives Considered:**

**1. MD5/SHA Hash of URL**
- Pros: Same URL → same short code (idempotent), simple
- Cons: Hash collisions, collision handling adds complexity, predictable if URL is known
- Why not chosen: Collision handling adds latency and complexity

**2. Random String Generation**
- Pros: Unpredictable, simple
- Cons: Need to check for collisions, potential infinite loop if space is exhausted
- Why not chosen: Collision checking adds DB query on every creation

### Design Decision 2: SQL vs NoSQL

**Chosen Approach:** PostgreSQL (SQL)

**Rationale:**
- ACID guarantees prevent duplicate short codes
- Strong indexing for fast lookups
- Simpler for read-heavy workloads with indexes
- Mature tooling and monitoring

**Alternative: Cassandra (NoSQL)**
- Pros: Horizontal scalability, high write throughput, multi-datacenter replication
- Cons: Eventual consistency, more complex, overkill for this scale
- Why not chosen: Added complexity not needed for 3,300 QPS reads

### Design Decision 3: 301 vs 302 Redirect

**Chosen Approach:** 301 (Permanent Redirect)

**Rationale:**
- Better for SEO
- Browsers cache 301 redirects
- Reduces server load for repeated visits

**Alternative: 302 (Temporary Redirect)**
- Pros: Guarantees analytics tracking on every visit (no browser cache)
- Cons: Higher server load, more database queries
- Why not chosen: Analytics can be tracked via JavaScript on landing page

**Hybrid Approach:**
- Use 302 for first visit (track analytics)
- Use 301 for subsequent visits (serve from cache/CDN)

### CAP Theorem Considerations

**Our Choice:** **Consistency + Partition Tolerance** (CP system)

**Justification:**
- URL creation requires strong consistency (no duplicate short codes)
- It's okay if URL creation is briefly unavailable during partition
- Redirects can serve stale data from cache (eventual consistency acceptable)

---

## 9. Monitoring, Metrics & Alerts

### Key Metrics

#### Application Metrics
- **URL Creation Rate:** Creates per second, success vs failure rate
- **Redirect Rate:** Redirects per second, 404 rate (broken links)
- **Redirect Latency:** p50, p95, p99, p999 latency
- **Cache Hit Ratio:** % of requests served from cache (target > 80%)
- **API Error Rate:** 4xx and 5xx errors per minute

#### Infrastructure Metrics
- **Database:**
  - Query latency (SELECT, INSERT)
  - Connection pool utilization
  - Replication lag (replica behind primary)
  - Disk usage

- **Cache (Redis):**
  - Memory usage
  - Eviction rate
  - Cache hit/miss ratio
  - Network I/O

- **Load Balancer:**
  - Requests per second
  - Active connections
  - Backend health status

#### Business Metrics
- **Daily Active URLs:** URLs created per day
- **Top URLs:** Most clicked short URLs
- **Expiration Rate:** % of URLs that expire vs are used
- **User Growth:** New users per day

### Logging Strategy

**What to Log:**

**Level: INFO**
- URL creation (short_code, long_url, user_id)
- Successful redirects (short_code, latency)
- Cache hits/misses

**Level: WARN**
- High redirect latency (> 500ms)
- Cache connection errors (fallback to DB)
- Rate limit exceeded

**Level: ERROR**
- Database connection failures
- URL creation failures
- 500 errors

**Log Format (JSON):**
```json
{
  "timestamp": "2024-01-15T10:30:00.123Z",
  "level": "INFO",
  "service": "url-shortener",
  "host": "api-server-1",
  "event": "url_created",
  "short_code": "abc123",
  "user_id": "12345",
  "latency_ms": 45,
  "trace_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

### Alerts

| Alert | Condition | Severity | Action |
|-------|-----------|----------|--------|
| **High error rate** | Error rate > 1% for 5 min | Critical | Page on-call engineer |
| **Database down** | Primary DB unreachable | Critical | Auto-failover + page |
| **High latency** | p99 > 1s for 5 min | Warning | Investigate performance |
| **Cache down** | Redis unreachable | Warning | System degraded, investigate |
| **Low cache hit ratio** | Hit ratio < 70% | Warning | Review cache strategy |
| **Disk space low** | Disk > 85% full | Warning | Provision more storage |
| **Unusual traffic spike** | QPS > 2x normal | Info | Monitor for DDoS |

### Monitoring Dashboard

**Tools:**
- **Metrics:** Prometheus + Grafana
- **Logging:** ELK Stack (Elasticsearch, Logstash, Kibana) or Datadog
- **Tracing:** Jaeger for distributed tracing
- **Alerts:** PagerDuty for on-call management

**Dashboard Panels:**
1. Request rate (QPS) - Last 24h
2. Latency percentiles (p50, p95, p99) - Last 1h
3. Error rate - Last 1h
4. Cache hit ratio - Last 24h
5. Database query latency - Last 1h
6. Top 10 most-clicked URLs - Last 24h

---

## 10. Follow-up Questions & Extensions

### Common Interview Follow-ups

#### Q1: How would you scale this to 100x the current load?

**Answer:**

**Traffic: 330,000 QPS reads, 3,300 QPS writes**

**Strategies:**

1. **Database Sharding:**
   - Shard by `short_code` prefix (consistent hashing)
   - 100 shards, each handling 3,300 QPS reads
   - Vitess or Citus for automatic sharding

2. **Multi-Region Deployment:**
   - Deploy in 5 regions (US-East, US-West, EU, Asia, Australia)
   - Route users to nearest region (GeoDNS)
   - Each region handles 20x load instead of 100x

3. **CDN for Redirects:**
   - Cache 301 redirects at CDN edge (Cloudflare, Fastly)
   - Reduces origin traffic by 90%+
   - Handle 300,000 QPS at edge, only 30,000 at origin

4. **Read Replicas:**
   - 20-50 read replicas per shard
   - Each replica handles ~1,650 QPS (well within capability)

#### Q2: How would you handle a viral URL with millions of requests per second?

**Answer:**

**Problem:** Single URL getting 10 million QPS

**Solutions:**

1. **CDN Caching:**
   - Cache 301 redirect at CDN level
   - CDN can handle millions of QPS
   - Origin never sees most requests

2. **Local Cache on API Servers:**
   - Each API server caches top 100 URLs in memory
   - No Redis/DB hit for viral URLs
   - Update every 1 minute

3. **Dedicated Cache Tier:**
   - Separate Redis cluster just for viral URLs
   - More memory, better SLA
   - Auto-detect viral URLs (click rate spike)

#### Q3: How would you prevent abuse (spam, malicious links)?

**Answer:**

**Strategies:**

1. **Rate Limiting:**
   - IP-based: 10 URLs/hour for anonymous users
   - User-based: 100 URLs/hour for authenticated users
   - Use Redis for distributed rate limiting

2. **URL Validation:**
   - Check against blacklist of malicious domains
   - Integrate with Google Safe Browsing API
   - Block URL shorteners (prevent chaining)

3. **CAPTCHA:**
   - Require CAPTCHA after 5 URL creations from same IP
   - Use reCAPTCHA v3 (invisible)

4. **Link Scanning:**
   - Asynchronously scan target URL for malware
   - Flag or block malicious links
   - Show warning page before redirect for flagged links

5. **Authentication:**
   - Require login for URL creation
   - Ban abusive accounts

#### Q4: How would you add custom domains (bit.ly/mycompany/abc123)?

**Answer:**

**Implementation:**

1. **Domain Verification:**
   - User adds TXT record to DNS
   - Verify ownership before allowing custom domain

2. **SSL Certificates:**
   - Auto-generate SSL certs using Let's Encrypt
   - Store certs in secure storage (AWS Secrets Manager)

3. **Routing:**
   - Load balancer routes by Host header
   - Map domain to user/organization in database

4. **Schema Change:**
```sql
ALTER TABLE urls ADD COLUMN custom_domain VARCHAR(255);
CREATE INDEX idx_domain_code ON urls(custom_domain, short_code);
```

5. **Lookup Logic:**
```python
async def get_long_url(domain: str, short_code: str):
    if domain == "short.ly":
        # Default domain lookup
        return await db.query("SELECT long_url FROM urls WHERE short_code = ?", short_code)
    else:
        # Custom domain lookup
        return await db.query(
            "SELECT long_url FROM urls WHERE custom_domain = ? AND short_code = ?",
            domain, short_code
        )
```

### Possible Extensions

1. **Link Expiration & Auto-deletion:**
   - Background job to delete expired URLs
   - Run daily via cron job
   - Free up short codes for reuse

2. **QR Code Generation:**
   - Generate QR code on-demand or pre-generate
   - Store in S3, serve via CDN
   - Library: python-qrcode or external API

3. **A/B Testing for Marketers:**
   - Create multiple short URLs for same long URL
   - Track which performs better
   - Useful for marketing campaigns

4. **Browser Extension:**
   - One-click URL shortening
   - Automatically shorten URLs on copy

5. **API for Bulk Creation:**
   - Upload CSV of URLs
   - Return CSV of short URLs
   - Background processing via queue

### Advanced Considerations

**Multi-Region Active-Active Setup:**
```
Problem: Avoid ID collisions across regions

Solution: Assign ID ranges to each region
- US-East: IDs starting with 0-1
- US-West: IDs starting with 2-3
- EU: IDs starting with 4-5
- Asia: IDs starting with 6-7

Or use Snowflake IDs with region ID embedded
```

**Data Migration Strategy:**
```
Scenario: Migrating from MySQL to Cassandra

Approach:
1. Dual-write: Write to both MySQL and Cassandra
2. Backfill: Copy historical data from MySQL to Cassandra
3. Verify: Compare data between systems
4. Switch reads: Gradually move reads to Cassandra
5. Deprecate: Stop writing to MySQL once stable
```

---

## Code Implementation

### Complete Python Implementation (FastAPI)

```python
from fastapi import FastAPI, HTTPException, Request
from pydantic import BaseModel, HttpUrl
from typing import Optional
import asyncpg
import redis.asyncio as redis
from datetime import datetime, timedelta
import hashlib

app = FastAPI()

# Database and cache connections
db_pool = None
redis_client = None

class URLCreate(BaseModel):
    long_url: HttpUrl
    custom_alias: Optional[str] = None
    expiration_days: Optional[int] = None

class URLResponse(BaseModel):
    short_url: str
    long_url: str
    created_at: datetime
    expiration_date: Optional[datetime] = None

class URLShortener:
    BASE62 = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

    @classmethod
    def encode(cls, num: int) -> str:
        if num == 0:
            return cls.BASE62[0]

        result = []
        base = len(cls.BASE62)
        while num > 0:
            result.append(cls.BASE62[num % base])
            num //= base
        return ''.join(reversed(result))

    @classmethod
    def decode(cls, code: str) -> int:
        num = 0
        base = len(cls.BASE62)
        for char in code:
            num = num * base + cls.BASE62.index(char)
        return num

@app.on_event("startup")
async def startup():
    global db_pool, redis_client
    db_pool = await asyncpg.create_pool(
        "postgresql://user:pass@localhost/urlshortener",
        min_size=10,
        max_size=50
    )
    redis_client = await redis.from_url("redis://localhost")

@app.on_event("shutdown")
async def shutdown():
    await db_pool.close()
    await redis_client.close()

@app.post("/api/v1/shorten", response_model=URLResponse, status_code=201)
async def create_short_url(url_data: URLCreate):
    async with db_pool.acquire() as conn:
        # Calculate expiration date
        expiration_date = None
        if url_data.expiration_days:
            expiration_date = datetime.utcnow() + timedelta(days=url_data.expiration_days)

        if url_data.custom_alias:
            # Check if custom alias exists
            existing = await conn.fetchrow(
                "SELECT id FROM urls WHERE short_code = $1",
                url_data.custom_alias
            )
            if existing:
                raise HTTPException(status_code=409, detail="Custom alias already taken")

            short_code = url_data.custom_alias

            # Insert with custom alias
            row = await conn.fetchrow(
                """
                INSERT INTO urls (short_code, long_url, expiration_date, created_at)
                VALUES ($1, $2, $3, $4)
                RETURNING id, short_code, long_url, created_at, expiration_date
                """,
                short_code, str(url_data.long_url), expiration_date, datetime.utcnow()
            )
        else:
            # Generate short code from auto-increment ID
            row = await conn.fetchrow(
                """
                INSERT INTO urls (long_url, expiration_date, created_at, short_code)
                VALUES ($1, $2, $3, '')
                RETURNING id, long_url, created_at, expiration_date
                """,
                str(url_data.long_url), expiration_date, datetime.utcnow()
            )

            # Generate short code from ID
            short_code = URLShortener.encode(row['id'])

            # Update with generated short code
            await conn.execute(
                "UPDATE urls SET short_code = $1 WHERE id = $2",
                short_code, row['id']
            )

        # Cache the URL
        cache_key = f"url:{short_code}"
        await redis_client.setex(cache_key, 86400, str(url_data.long_url))

        return URLResponse(
            short_url=f"https://short.ly/{short_code}",
            long_url=str(url_data.long_url),
            created_at=row['created_at'],
            expiration_date=expiration_date
        )

@app.get("/{short_code}")
async def redirect_to_long_url(short_code: str):
    # Try cache first
    cache_key = f"url:{short_code}"
    long_url = await redis_client.get(cache_key)

    if long_url:
        # Cache hit
        long_url = long_url.decode('utf-8')
    else:
        # Cache miss - query database
        async with db_pool.acquire() as conn:
            row = await conn.fetchrow(
                """
                SELECT long_url, expiration_date
                FROM urls
                WHERE short_code = $1
                """,
                short_code
            )

            if not row:
                raise HTTPException(status_code=404, detail="URL not found")

            # Check expiration
            if row['expiration_date'] and row['expiration_date'] < datetime.utcnow():
                raise HTTPException(status_code=410, detail="URL expired")

            long_url = row['long_url']

            # Update cache
            await redis_client.setex(cache_key, 86400, long_url)

            # Increment click count (async, non-blocking)
            await conn.execute(
                """
                UPDATE urls
                SET click_count = click_count + 1, last_accessed = $1
                WHERE short_code = $2
                """,
                datetime.utcnow(), short_code
            )

    # Return 301 permanent redirect
    from fastapi.responses import RedirectResponse
    return RedirectResponse(url=long_url, status_code=301)

@app.get("/api/v1/analytics/{short_code}")
async def get_analytics(short_code: str):
    async with db_pool.acquire() as conn:
        row = await conn.fetchrow(
            """
            SELECT short_code, long_url, created_at, click_count, last_accessed
            FROM urls
            WHERE short_code = $1
            """,
            short_code
        )

        if not row:
            raise HTTPException(status_code=404, detail="URL not found")

        return {
            "short_code": row['short_code'],
            "long_url": row['long_url'],
            "created_at": row['created_at'],
            "click_count": row['click_count'],
            "last_accessed": row['last_accessed']
        }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## References

### Papers & Articles
- [Designing a URL Shortening Service](https://www.educative.io/courses/grokking-the-system-design-interview/m2ygV4E81AR)
- [How URL Shorteners Work](https://stackoverflow.blog/2020/03/02/best-practices-for-rest-api-design/)

### Real-World Implementations
- [TinyURL Architecture](https://tinyurl.com/)
- [Bitly Engineering Blog](https://bitly.engineering/)
- [YouTube: System Design: URL Shortener](https://www.youtube.com/watch?v=fMZMm_0ZhK4)

### Related Designs
- [Pastebin Design](../pastebin/) - Similar storage and retrieval pattern
- [Key-Value Store](../key-value-store/) - Core data structure
- [Rate Limiter](../../02-building-blocks/rate-limiter/) - Used in this design

---

## Interview Tips

### Common Mistakes to Avoid

1. **Not asking clarifying questions** - Always ask about scale, requirements
2. **Jumping to implementation** - Start with high-level design first
3. **Ignoring edge cases** - Handle expiration, collisions, empty URLs
4. **Not considering scalability** - Think about 10x, 100x growth
5. **Forgetting about monitoring** - Always discuss metrics and alerts

### How to Approach This Question

**First 5 minutes:**
- Clarify requirements (functional and non-functional)
- Estimate scale (traffic, storage, bandwidth)
- Agree on scope with interviewer

**Next 15-20 minutes:**
- Draw high-level architecture
- Discuss API design
- Design database schema
- Explain data flow (read and write paths)

**Final 20-25 minutes:**
- Deep dive into 2-3 components (interviewer's choice)
- Discuss bottlenecks and solutions
- Cover monitoring and operations
- Handle follow-up questions

### Key Talking Points

1. **Base62 encoding** - Explain why and how it works
2. **Caching strategy** - Cache-aside pattern, 80/20 rule
3. **Database choice** - SQL for consistency, NoSQL for scale
4. **Sharding strategy** - How to partition data at very large scale
5. **CAP theorem** - Consistency vs Availability trade-off

---

**Congratulations!** You've completed the URL Shortener system design. This is one of the most common interview questions - practice explaining each section clearly and concisely.

**Next Steps:**
- Practice drawing the architecture diagram from memory
- Implement a simple version in your favorite language
- Study [Pastebin](../pastebin/) for a similar but slightly different design
