# Pastebin Design

> **Difficulty:** Beginner
> **Interview Frequency:** High
> **Key Concepts:** Object storage, Expiration policies, Short URLs, Text storage

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
Design a text-sharing service like Pastebin.com or paste.ubuntu.com where users can paste text content, get a unique URL to share it, and optionally set expiration times. Think of it as a simple, text-focused alternative to file-sharing services.

**Example:**
- User pastes code snippet
- System generates: `https://pastebin.com/XYz9kL2m`
- Anyone with the link can view the paste
- Paste expires after 24 hours (optional)

### Functional Requirements

**Core Features:**
- [x] **Create paste:** User can paste text and get a unique URL
- [x] **View paste:** Anyone with URL can view the paste
- [x] **Expiration:** Pastes can have TTL (1 hour, 1 day, 1 week, never)
- [x] **Raw view:** Option to view raw text (no formatting)
- [x] **Syntax highlighting:** Auto-detect and highlight code
- [x] **Privacy levels:** Public (searchable), Unlisted (only via URL), Private (login required)

**Additional Features:**
- [x] **User accounts:** Optional user registration to manage pastes
- [x] **Edit/Delete:** Users can edit or delete their own pastes
- [x] **Clone/Fork:** Create a copy of existing paste
- [x] **View count:** Track how many times paste was viewed

### Non-Functional Requirements

**Scale:**
- **10 million** pastes created per month
- **100 million** paste views per month (10:1 read/write ratio)
- **50 million** total users (5 million DAU)
- Average paste size: 10 KB
- Max paste size: 10 MB

**Performance:**
- Paste creation: < 500ms
- Paste retrieval: < 200ms (99th percentile)
- Availability: 99.9% uptime

**Other:**
- [x] **Data durability:** Pastes should not be lost
- [x] **Scalable:** Handle traffic spikes
- [x] **Cost-effective:** Optimize storage costs
- [x] **Security:** Prevent spam and malicious content

### Out of Scope

- ❌ Real-time collaborative editing
- ❌ Image/video uploads (text only)
- ❌ Advanced analytics (just basic view counts)
- ❌ Encryption (end-to-end encrypted pastes)
- ❌ API rate limiting (assume covered elsewhere)

### Constraints and Assumptions

**Assumptions:**
- **Read-heavy system:** 10:1 read/write ratio
- **Average paste size:** 10 KB (median), 50 KB (p95)
- **Paste retention:** Default 30 days, can be set to never expire
- **Most pastes are small:** 80% under 5 KB (code snippets)
- **Traffic pattern:** Higher usage during work hours

---

## 2. Back-of-the-Envelope Estimation

### Traffic Estimation

#### Write Operations (Paste Creation)
```
Pastes created per month: 10 million
Pastes created per day: 10M / 30 ≈ 333,000
Pastes created per second (QPS): 333K / 100,000 ≈ 3.3 QPS
Peak write QPS: 3.3 × 3 ≈ 10 QPS (during work hours)
```

#### Read Operations (Paste Views)
```
Paste views per month: 100 million
Paste views per day: 100M / 30 ≈ 3.3 million
Read QPS: 3.3M / 100,000 ≈ 33 QPS
Peak read QPS: 33 × 3 ≈ 100 QPS
```

### Storage Estimation

```
Data per paste:
- Text content: 10 KB (average)
- Metadata (paste_id, user_id, created_at, expiration, privacy): 200 bytes
- Total per paste: ~10.2 KB

Total pastes over 1 year (assuming 30-day retention):
- Per month: 10 million
- Active pastes: 10M (30-day window)

For pastes set to "never expire":
- Assume 20% never expire: 10M × 0.2 = 2M per month
- Over 5 years: 2M × 12 × 5 = 120 million pastes

Storage calculation:
- 30-day window: 10M × 10.2 KB = 102 GB
- 5-year never-expire: 120M × 10.2 KB = 1,224 GB ≈ 1.2 TB
- Total raw storage: ~1.3 TB

With overhead (indexes, metadata tables):
- 1.3 TB × 1.5 = ~2 TB

With replication (3x):
- 2 TB × 3 = 6 TB total
```

### Bandwidth Estimation

```
Write Bandwidth (incoming):
- 3.3 QPS × 10 KB = 33 KB/s ≈ 0.26 Mbps

Read Bandwidth (outgoing):
- 33 QPS × 10 KB = 330 KB/s ≈ 2.64 Mbps

Total bandwidth: ~3 Mbps (very low)
```

### Memory/Cache Estimation

```
Using 80/20 rule (80% of reads from 20% of pastes):

Daily views: 3.3 million
Unique pastes viewed (20%): 3.3M × 0.2 = 660,000 pastes

Cache size needed:
- 660K × 10 KB ≈ 6.6 GB

For safety, provision 10-15 GB of cache (Redis)
Cache TTL: 24 hours
```

### Summary Table

| Metric | Estimate |
|--------|----------|
| **Write QPS (avg)** | 3.3 |
| **Write QPS (peak)** | 10 |
| **Read QPS (avg)** | 33 |
| **Read QPS (peak)** | 100 |
| **Storage (5 years)** | 6 TB |
| **Bandwidth (in)** | 0.26 Mbps |
| **Bandwidth (out)** | 2.64 Mbps |
| **Cache size** | 10-15 GB |

---

## 3. API Design

### REST API Endpoints

#### 1. Create Paste
```http
POST /api/v1/pastes
Content-Type: application/json
Authorization: Bearer <token> (optional)

Request:
{
  "content": "def hello_world():\n    print('Hello, World!')",
  "title": "Python Hello World",
  "syntax": "python",  // auto-detect if not provided
  "expiration": "24h",  // 1h, 24h, 1w, 30d, never
  "privacy": "unlisted"  // public, unlisted, private
}

Response (201 Created):
{
  "paste_id": "XYz9kL2m",
  "url": "https://pastebin.com/XYz9kL2m",
  "raw_url": "https://pastebin.com/raw/XYz9kL2m",
  "created_at": "2024-01-15T10:30:00Z",
  "expires_at": "2024-01-16T10:30:00Z"
}

Error Response (413 Payload Too Large):
{
  "error": "content_too_large",
  "message": "Paste content exceeds 10 MB limit"
}
```

#### 2. Get Paste
```http
GET /api/v1/pastes/{paste_id}

Response (200 OK):
{
  "paste_id": "XYz9kL2m",
  "content": "def hello_world():\n    print('Hello, World!')",
  "title": "Python Hello World",
  "syntax": "python",
  "created_at": "2024-01-15T10:30:00Z",
  "expires_at": "2024-01-16T10:30:00Z",
  "view_count": 42,
  "privacy": "unlisted"
}

Error Response (404 Not Found):
{
  "error": "paste_not_found",
  "message": "Paste not found or has expired"
}

Error Response (403 Forbidden):
{
  "error": "private_paste",
  "message": "This paste is private and requires authentication"
}
```

#### 3. Get Raw Paste
```http
GET /api/v1/pastes/{paste_id}/raw
Content-Type: text/plain

Response (200 OK):
def hello_world():
    print('Hello, World!')
```

#### 4. Update Paste
```http
PUT /api/v1/pastes/{paste_id}
Authorization: Bearer <token>
Content-Type: application/json

Request:
{
  "content": "def hello_world():\n    print('Hello, Universe!')",
  "title": "Updated Python Hello"
}

Response (200 OK):
{
  "paste_id": "XYz9kL2m",
  "url": "https://pastebin.com/XYz9kL2m",
  "updated_at": "2024-01-15T11:00:00Z"
}
```

#### 5. Delete Paste
```http
DELETE /api/v1/pastes/{paste_id}
Authorization: Bearer <token>

Response (204 No Content)

Error Response (403 Forbidden):
{
  "error": "unauthorized",
  "message": "You can only delete your own pastes"
}
```

#### 6. List User's Pastes
```http
GET /api/v1/users/{user_id}/pastes?page=1&limit=20
Authorization: Bearer <token>

Response (200 OK):
{
  "pastes": [
    {
      "paste_id": "XYz9kL2m",
      "title": "Python Hello World",
      "created_at": "2024-01-15T10:30:00Z",
      "view_count": 42,
      "privacy": "unlisted"
    },
    ...
  ],
  "total": 127,
  "page": 1,
  "pages": 7
}
```

### API Design Considerations

- **Versioning:** `/api/v1/` for future changes
- **Authentication:** Optional JWT for user-specific features
- **Content Limits:** 10 MB max paste size
- **Rate Limiting:** 100 pastes/hour per IP (anonymous), 1000/hour per user
- **Caching:** Aggressive caching with Cache-Control headers
- **Compression:** gzip compression for large pastes

---

## 4. Data Model & Database Schema

### Database Choice

**Primary Database:** PostgreSQL

**Justification:**
- **ACID compliance:** Ensure paste data integrity
- **Indexing:** Fast lookups by paste_id
- **JSON support:** Store metadata flexibly
- **Mature:** Well-tested for read-heavy workloads

**Object Storage:** Amazon S3 (or equivalent)

**Justification:**
- **Cost-effective:** $0.023/GB vs database storage
- **Scalable:** Unlimited storage
- **Durability:** 99.999999999% (11 nines)
- **CDN integration:** CloudFront for fast delivery

### Schema Design

#### Approach 1: Database-Only (Small Scale)

```sql
CREATE TABLE pastes (
    id BIGSERIAL PRIMARY KEY,
    paste_id VARCHAR(10) UNIQUE NOT NULL,
    user_id BIGINT,
    title VARCHAR(255),
    content TEXT NOT NULL,
    syntax VARCHAR(50),
    privacy VARCHAR(20) DEFAULT 'unlisted',  -- public, unlisted, private
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP,
    expires_at TIMESTAMP,
    view_count BIGINT DEFAULT 0,

    INDEX idx_paste_id (paste_id),
    INDEX idx_user_id (user_id),
    INDEX idx_created_at (created_at),
    INDEX idx_expires_at (expires_at)
);

CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_username (username),
    INDEX idx_email (email)
);
```

#### Approach 2: Database + Object Storage (Scalable - Chosen)

```sql
CREATE TABLE pastes (
    id BIGSERIAL PRIMARY KEY,
    paste_id VARCHAR(10) UNIQUE NOT NULL,
    user_id BIGINT,
    title VARCHAR(255),
    content_url VARCHAR(500),  -- S3 URL instead of storing content
    size_bytes INT,
    syntax VARCHAR(50),
    privacy VARCHAR(20) DEFAULT 'unlisted',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP,
    expires_at TIMESTAMP,
    view_count BIGINT DEFAULT 0,

    INDEX idx_paste_id (paste_id),
    INDEX idx_user_id (user_id),
    INDEX idx_expires_at (expires_at)
);

CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_username (username),
    INDEX idx_email (email)
);
```

**S3 Object Key Structure:**
```
s3://pastebin-content/{year}/{month}/{paste_id}.txt

Example:
s3://pastebin-content/2024/01/XYz9kL2m.txt
```

### Data Model Diagram

```mermaid
erDiagram
    USERS ||--o{ PASTES : creates

    USERS {
        bigint id PK
        varchar username UK
        varchar email UK
        varchar password_hash
        timestamp created_at
    }

    PASTES {
        bigint id PK
        varchar paste_id UK
        bigint user_id FK
        varchar title
        varchar content_url
        int size_bytes
        varchar syntax
        varchar privacy
        timestamp created_at
        timestamp updated_at
        timestamp expires_at
        bigint view_count
    }
```

### Why Object Storage?

**Cost Comparison (for 1 TB):**
```
PostgreSQL (RDS):
- Storage: $0.115/GB = $115/month
- IOPS: Additional costs
- Backups: Additional costs
- Total: ~$200/month

Amazon S3:
- Storage: $0.023/GB = $23/month
- GET requests: $0.0004 per 1000 = ~$13/month (100M views)
- Total: ~$36/month

Savings: $164/month = 82% cheaper
```

---

## 5. High-Level Design

### Architecture Diagram

```mermaid
graph TB
    Client[Client/Browser]
    CDN[CDN CloudFront]
    LB[Load Balancer]
    API1[API Server 1]
    API2[API Server 2]
    Cache[(Redis Cache)]
    DB[(PostgreSQL<br/>Metadata)]
    S3[(S3 Object Storage<br/>Paste Content)]
    Worker[Background Workers]
    Queue[Message Queue]

    Client --> CDN
    CDN --> LB
    LB --> API1
    LB --> API2

    API1 --> Cache
    API2 --> Cache

    API1 --> DB
    API2 --> DB

    API1 --> S3
    API2 --> S3

    API1 --> Queue
    API2 --> Queue

    Queue --> Worker
    Worker --> DB
    Worker --> S3

    style Client fill:#f9f,stroke:#333
    style CDN fill:#bbf,stroke:#333
    style LB fill:#bbf,stroke:#333
    style API1 fill:#bfb,stroke:#333
    style API2 fill:#bfb,stroke:#333
    style Cache fill:#fbb,stroke:#333
    style DB fill:#ffb,stroke:#333
    style S3 fill:#bff,stroke:#333
```

### Component Overview

1. **Client:** Web browser or API consumer
2. **CDN (CloudFront):** Caches static assets and popular pastes
3. **Load Balancer:** Distributes traffic across API servers (NGINX/ALB)
4. **API Servers:** Stateless application servers (Node.js/Python/Go)
5. **Redis Cache:** Caches metadata and popular paste content
6. **PostgreSQL:** Stores paste metadata, user data
7. **S3 Object Storage:** Stores actual paste content
8. **Message Queue:** Async tasks (Kafka/SQS)
9. **Background Workers:** Cleanup expired pastes, update stats

### Data Flow

#### Create Paste Flow

```mermaid
sequenceDiagram
    participant Client
    participant API as API Server
    participant DB as PostgreSQL
    participant S3
    participant Cache

    Client->>API: POST /api/v1/pastes
    API->>API: Generate paste_id (base62)
    API->>API: Validate content (size, format)

    par Upload to S3
        API->>S3: Upload content
        S3-->>API: Return S3 URL
    and Insert Metadata
        API->>DB: INSERT metadata with S3 URL
        DB-->>API: Confirm insert
    end

    API->>Cache: Cache metadata
    API-->>Client: Return paste URL
```

#### View Paste Flow (Hot Path)

```mermaid
sequenceDiagram
    participant Client
    participant CDN
    participant API as API Server
    participant Cache
    participant DB as PostgreSQL
    participant S3

    Client->>CDN: GET /pastes/XYz9kL2m

    alt CDN Cache Hit
        CDN-->>Client: Return cached page
    else CDN Cache Miss
        CDN->>API: Forward request
        API->>Cache: Get metadata

        alt Cache Hit
            Cache-->>API: Return metadata
        else Cache Miss
            API->>DB: SELECT metadata
            DB-->>API: Return metadata
            API->>Cache: Store metadata
        end

        API->>S3: GET content
        S3-->>API: Return content
        API-->>CDN: Return rendered page
        CDN-->>Client: Return page (and cache)
    end

    Note over API: Async: increment view_count
```

---

## 6. Detailed Component Design

### Component 1: Paste ID Generation

**Similar to URL Shortener, but simpler:**

```python
import random
import string

class PasteIDGenerator:
    # Base62: 0-9, a-z, A-Z = 62 characters
    BASE62 = string.digits + string.ascii_lowercase + string.ascii_uppercase

    def __init__(self):
        self.base = len(self.BASE62)

    def generate(self, length=8):
        """
        Generate random 8-character paste ID
        62^8 = 218 trillion combinations

        Collision probability with 10M pastes:
        p ≈ (10M)^2 / (2 × 62^8) ≈ 0.0002% (very low)
        """
        return ''.join(random.choices(self.BASE62, k=length))

    def generate_from_id(self, id: int) -> str:
        """
        Alternative: Convert numeric ID to base62
        More predictable but guaranteed unique
        """
        if id == 0:
            return self.BASE62[0]

        result = []
        while id > 0:
            result.append(self.BASE62[id % self.base])
            id //= self.base

        return ''.join(reversed(result)).rjust(8, '0')

# Usage:
generator = PasteIDGenerator()
paste_id = generator.generate()  # "k7Hx9mP2"
```

**Collision Handling:**
```python
async def create_paste_id(db):
    """Generate paste ID with collision check"""
    max_attempts = 5

    for _ in range(max_attempts):
        paste_id = PasteIDGenerator().generate()

        # Check if exists
        exists = await db.fetchval(
            "SELECT 1 FROM pastes WHERE paste_id = $1",
            paste_id
        )

        if not exists:
            return paste_id

    # Fallback to timestamp-based ID if random fails
    import time
    return PasteIDGenerator().generate_from_id(int(time.time()))
```

### Component 2: Expiration and Cleanup

**Expiration Strategies:**

**Option 1: Passive Deletion (Lazy)**
```python
async def get_paste(paste_id: str):
    """Check expiration on read"""
    paste = await db.fetchrow(
        "SELECT * FROM pastes WHERE paste_id = $1",
        paste_id
    )

    if not paste:
        raise PasteNotFoundError()

    # Check if expired
    if paste['expires_at'] and paste['expires_at'] < datetime.utcnow():
        # Optionally delete immediately
        await db.execute(
            "DELETE FROM pastes WHERE paste_id = $1",
            paste_id
        )
        raise PasteExpiredError()

    return paste
```

**Option 2: Active Deletion (Background Job)**
```python
# Cron job running daily
async def cleanup_expired_pastes():
    """Delete expired pastes from DB and S3"""
    batch_size = 1000

    while True:
        # Find expired pastes
        expired = await db.fetch(
            """
            SELECT paste_id, content_url
            FROM pastes
            WHERE expires_at < $1
            LIMIT $2
            """,
            datetime.utcnow(),
            batch_size
        )

        if not expired:
            break

        paste_ids = [p['paste_id'] for p in expired]
        s3_keys = [extract_s3_key(p['content_url']) for p in expired]

        # Delete from S3
        await s3_client.delete_objects(
            Bucket='pastebin-content',
            Delete={'Objects': [{'Key': k} for k in s3_keys]}
        )

        # Delete from database
        await db.execute(
            "DELETE FROM pastes WHERE paste_id = ANY($1)",
            paste_ids
        )

        print(f"Deleted {len(expired)} expired pastes")
```

**Chosen Approach:** Hybrid
- Passive deletion on read (immediate feedback)
- Background cleanup daily (free up storage)

### Component 3: Caching Strategy

**Multi-Level Caching:**

```python
# Level 1: CDN (CloudFront)
# - Cache entire HTML pages
# - TTL: 1 hour for public pastes
# - Bypass for private pastes

# Level 2: Application Cache (Redis)
async def get_paste_cached(paste_id: str):
    """Cache metadata and content separately"""

    # Try cache first
    cache_key = f"paste:{paste_id}"
    cached = await redis.get(cache_key)

    if cached:
        return json.loads(cached)

    # Cache miss - fetch from DB + S3
    metadata = await db.fetchrow(
        "SELECT * FROM pastes WHERE paste_id = $1",
        paste_id
    )

    if not metadata:
        raise PasteNotFoundError()

    # Fetch content from S3
    content = await s3_client.get_object(
        Bucket='pastebin-content',
        Key=extract_s3_key(metadata['content_url'])
    )

    paste_data = {
        **dict(metadata),
        'content': content
    }

    # Cache for 1 hour
    await redis.setex(
        cache_key,
        3600,
        json.dumps(paste_data, default=str)
    )

    return paste_data
```

**Cache Invalidation:**
```python
async def update_paste(paste_id: str, new_content: str):
    """Update paste and invalidate cache"""

    # Update S3
    await s3_client.put_object(
        Bucket='pastebin-content',
        Key=f"2024/01/{paste_id}.txt",
        Body=new_content
    )

    # Update database
    await db.execute(
        "UPDATE pastes SET updated_at = $1 WHERE paste_id = $2",
        datetime.utcnow(),
        paste_id
    )

    # Invalidate cache
    await redis.delete(f"paste:{paste_id}")

    # Invalidate CDN (if using CloudFront)
    await cloudfront_invalidate(f"/pastes/{paste_id}")
```

### Component 4: Syntax Highlighting

**Server-Side vs Client-Side:**

**Option 1: Client-Side (Chosen)**
```javascript
// Use highlight.js or Prism.js on the frontend
<script src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/highlight.min.js"></script>
<script>hljs.highlightAll();</script>

<pre><code class="language-python">
{{ paste.content }}
</code></pre>
```

**Pros:**
- No server CPU load
- Fast page rendering
- Many language support (200+)
- CDN-cacheable library

**Option 2: Server-Side**
- Use Pygments (Python) to generate HTML
- Pro: Works without JavaScript
- Con: Server CPU cost, harder to cache

**Auto-Detection:**
```python
from pygments import lexers

def detect_language(content: str) -> str:
    """Auto-detect programming language"""
    try:
        lexer = lexers.guess_lexer(content)
        return lexer.name.lower()
    except:
        return 'text'

# Example:
code = "def hello(): print('Hi')"
language = detect_language(code)  # "python"
```

---

## 7. Identifying and Resolving Bottlenecks

### Potential Bottlenecks

#### 1. Storage Costs (Database)

**Problem:** Storing large pastes in PostgreSQL is expensive

**Solution:** Use S3 object storage
```
Cost savings:
- Database: $115/TB/month
- S3: $23/TB/month
- Savings: 80%

Additional benefits:
- Unlimited scalability
- Built-in redundancy (11 nines durability)
- CDN integration
```

#### 2. Hot Paste Problem

**Problem:** Viral paste gets millions of views, overwhelming servers

**Solutions:**

**A. CDN Caching**
```
- Cache entire HTML page at CloudFront edge
- 99% of requests served from edge
- Origin sees < 1% of traffic
```

**B. Redis Caching**
```python
# Cache popular pastes with higher TTL
async def get_paste_smart_cache(paste_id: str):
    if view_count > 10000:
        ttl = 86400  # 24 hours for viral pastes
    elif view_count > 1000:
        ttl = 3600   # 1 hour for popular
    else:
        ttl = 300    # 5 minutes for normal

    # Cache with dynamic TTL
    await redis.setex(f"paste:{paste_id}", ttl, paste_data)
```

#### 3. Database Write Bottleneck

**Problem:** High write traffic (view count updates) overwhelms DB

**Solution:** Async view count updates
```python
# Don't update view count synchronously
async def view_paste(paste_id: str):
    # Fetch and return paste immediately
    paste = await get_paste_cached(paste_id)

    # Queue view count update asynchronously
    await kafka.produce('paste_views', {
        'paste_id': paste_id,
        'timestamp': datetime.utcnow()
    })

    return paste

# Separate worker processes view count updates
async def process_view_counts():
    # Batch updates every 5 minutes
    views = {}

    async for event in kafka.consume('paste_views'):
        views[event['paste_id']] = views.get(event['paste_id'], 0) + 1

    # Batch update database
    for paste_id, count in views.items():
        await db.execute(
            "UPDATE pastes SET view_count = view_count + $1 WHERE paste_id = $2",
            count, paste_id
        )
```

#### 4. S3 Request Costs

**Problem:** Too many S3 GET requests increase costs

**Solution:** Multi-layer caching
```
1. CloudFront CDN (cache hit ratio: 80%)
2. Redis cache (cache hit ratio: 15%)
3. S3 direct fetch (5% of requests)

Result: 95% reduction in S3 costs
```

### Fault Tolerance

**Strategies:**

1. **Database:** Primary-replica setup with automatic failover (RDS Multi-AZ)
2. **S3:** Built-in 11 nines durability, cross-region replication optional
3. **Cache:** Redis Cluster with Sentinel for automatic failover
4. **API Servers:** Auto-scaling group, health checks, multiple AZs
5. **Graceful Degradation:** Serve from cache even if DB/S3 is down

---

## 8. Trade-offs and Alternatives

### Design Decision 1: S3 vs Database Storage

**Chosen:** S3 for content, Database for metadata

**Rationale:**
- **Cost:** 80% cheaper ($23 vs $115 per TB)
- **Scalability:** Unlimited storage
- **Performance:** CDN integration

**Alternative: Database Only**
- Pros: Simpler architecture, atomic transactions
- Cons: Expensive at scale, limited storage
- Why not: Cost prohibitive for millions of pastes

### Design Decision 2: Random Paste ID vs Sequential

**Chosen:** Random base62 (8 characters)

**Rationale:**
- Unpredictable (security through obscurity)
- No coordination needed (distributed systems)
- 218 trillion combinations (low collision)

**Alternative: Auto-incrementing ID + Base62**
- Pros: Guaranteed unique, shorter IDs possible
- Cons: Predictable, requires coordination
- Why not: Collision probability is acceptably low

### Design Decision 3: Passive vs Active Expiration

**Chosen:** Hybrid (both)

**Rationale:**
- Passive: Immediate user feedback
- Active: Free up storage costs
- Best of both worlds

**Alternative: Only Active**
- Pros: Simpler, consistent
- Cons: Users see expired pastes until cleanup
- Why not: Poor user experience

### Design Decision 4: Syntax Highlighting

**Chosen:** Client-side (highlight.js)

**Rationale:**
- Zero server CPU cost
- Fast page load (JS is cached)
- 200+ languages supported

**Alternative: Server-side (Pygments)**
- Pros: Works without JS, SEO-friendly
- Cons: Server CPU cost, caching complexity
- Why not: Modern browsers support JS universally

---

## 9. Monitoring, Metrics & Alerts

### Key Metrics

#### Application Metrics
- **Paste Creation Rate:** Pastes/second, success rate
- **Paste View Rate:** Views/second, cache hit ratio
- **Response Time:** p50, p95, p99 latency
- **Error Rate:** 404 (not found), 410 (expired), 5xx errors
- **Storage Growth:** GB/day, paste size distribution

#### Infrastructure Metrics
- **Database:**
  - Query latency
  - Connection pool usage
  - Disk usage

- **Cache (Redis):**
  - Hit ratio (target > 80%)
  - Memory usage
  - Eviction rate

- **Object Storage (S3):**
  - GET request count
  - PUT request count
  - Storage used (GB)
  - Transfer out (GB)

#### Business Metrics
- **Daily Active Pastes:** Pastes created/day
- **Popular Syntax:** Most used languages
- **Expiration Rate:** % never-expire vs temporary
- **User Growth:** New users/day

### Logging Strategy

**What to Log:**

**Level: INFO**
- Paste creation (paste_id, size, syntax, privacy)
- Paste views (paste_id, latency)
- Cache hits/misses

**Level: WARN**
- Expired paste access attempts
- Large paste uploads (> 1 MB)
- High view count pastes (potential abuse)

**Level: ERROR**
- S3 upload failures
- Database connection errors
- Cache failures

**Log Format (JSON):**
```json
{
  "timestamp": "2024-01-15T10:30:00.123Z",
  "level": "INFO",
  "service": "pastebin-api",
  "event": "paste_created",
  "paste_id": "XYz9kL2m",
  "user_id": "12345",
  "size_bytes": 10240,
  "syntax": "python",
  "latency_ms": 250,
  "trace_id": "abc-def-ghi"
}
```

### Alerts

| Alert | Condition | Severity | Action |
|-------|-----------|----------|--------|
| **High error rate** | Error rate > 5% for 5 min | Critical | Page on-call |
| **Database down** | DB unreachable | Critical | Auto-failover + page |
| **S3 errors** | S3 error rate > 1% | Critical | Check AWS status |
| **Low cache hit ratio** | Hit ratio < 60% | Warning | Investigate cache config |
| **High latency** | p99 > 1s for 5 min | Warning | Check DB/S3 performance |
| **Storage limit** | S3 storage > 5 TB | Info | Plan capacity |
| **Unusual traffic** | QPS > 5x normal | Info | Monitor for DDoS |

---

## 10. Follow-up Questions & Extensions

### Common Interview Follow-ups

#### Q1: How would you prevent spam and abuse?

**Answer:**

**Rate Limiting:**
```python
# Per IP: 100 pastes/hour for anonymous
# Per user: 1000 pastes/hour for authenticated
# CAPTCHA after 10 pastes in 5 minutes
```

**Content Validation:**
```python
# Max size: 10 MB
# Scan for malicious content (SQL injection attempts in paste)
# Blacklist known spam patterns
# Check against virus scanning API (VirusTotal)
```

**User Reports:**
```python
# Allow users to report inappropriate pastes
# Auto-hide after 5 reports (manual review)
# Ban repeat offenders
```

#### Q2: How would you add collaborative editing (like Google Docs)?

**Answer:**

**Challenges:** Real-time sync, conflict resolution

**Solution:**
- Use **Operational Transformation (OT)** or **CRDTs**
- WebSocket connection for real-time updates
- Store edit history in separate table
- Version control (snapshots every N edits)

**Out of scope for Pastebin**, but could be a premium feature.

#### Q3: How would you scale to 100x traffic?

**Answer:**

**Traffic: 330 QPS reads, 33 QPS writes**

**Strategies:**

1. **Database Sharding:**
   - Shard by paste_id prefix (A-Z)
   - 26 shards, each handles 12.7 QPS reads

2. **Multi-Region Deployment:**
   - Deploy in 3 regions (US, EU, Asia)
   - Route users to nearest region (GeoDNS)
   - Reduce per-region load by 3x

3. **CDN Optimization:**
   - Cache aggressively (90%+ hit ratio)
   - Use edge functions for dynamic content
   - Serve 99% of reads from edge

4. **Read Replicas:**
   - 10 read replicas per shard
   - Each handles ~1.3 QPS (well within limits)

#### Q4: How would you handle large file uploads (100 MB)?

**Answer:**

**Chunked Upload:**
```python
# Client splits file into 5 MB chunks
# Upload chunks to S3 in parallel
# Use S3 multipart upload API

# Server-side:
async def initiate_multipart_upload(paste_id):
    upload_id = await s3.create_multipart_upload(
        Bucket='pastebin-content',
        Key=f'2024/01/{paste_id}.txt'
    )
    return upload_id

async def complete_multipart_upload(paste_id, upload_id, parts):
    await s3.complete_multipart_upload(
        Bucket='pastebin-content',
        Key=f'2024/01/{paste_id}.txt',
        UploadId=upload_id,
        MultipartUpload={'Parts': parts}
    )
```

**Streaming:**
- Stream directly to S3 without buffering in memory
- Use signed URLs for direct client-to-S3 upload

---

## Code Implementation

### Complete Python FastAPI Implementation

```python
from fastapi import FastAPI, HTTPException, UploadFile
from pydantic import BaseModel
import asyncpg
import boto3
from datetime import datetime, timedelta
import random
import string

app = FastAPI()

# Database and S3 clients
db_pool = None
s3_client = boto3.client('s3')
BUCKET_NAME = 'pastebin-content'

class PasteCreate(BaseModel):
    content: str
    title: str = None
    syntax: str = None
    expiration: str = "30d"  # 1h, 24h, 1w, 30d, never
    privacy: str = "unlisted"

class PasteResponse(BaseModel):
    paste_id: str
    url: str
    raw_url: str
    created_at: datetime
    expires_at: datetime = None

def generate_paste_id(length=8):
    """Generate random base62 paste ID"""
    chars = string.digits + string.ascii_letters
    return ''.join(random.choices(chars, k=length))

def parse_expiration(expiration: str) -> datetime:
    """Convert expiration string to datetime"""
    if expiration == "never":
        return None

    units = {"h": "hours", "d": "days", "w": "weeks"}
    amount = int(expiration[:-1])
    unit = units[expiration[-1]]

    return datetime.utcnow() + timedelta(**{unit: amount})

@app.on_event("startup")
async def startup():
    global db_pool
    db_pool = await asyncpg.create_pool(
        "postgresql://user:pass@localhost/pastebin",
        min_size=10,
        max_size=50
    )

@app.on_event("shutdown")
async def shutdown():
    await db_pool.close()

@app.post("/api/v1/pastes", response_model=PasteResponse, status_code=201)
async def create_paste(paste: PasteCreate):
    """Create a new paste"""

    # Validate content size (10 MB limit)
    if len(paste.content.encode()) > 10 * 1024 * 1024:
        raise HTTPException(status_code=413, detail="Content exceeds 10 MB limit")

    # Generate paste ID
    paste_id = generate_paste_id()

    # Upload content to S3
    s3_key = f"{datetime.utcnow().year}/{datetime.utcnow().month}/{paste_id}.txt"
    s3_client.put_object(
        Bucket=BUCKET_NAME,
        Key=s3_key,
        Body=paste.content.encode(),
        ContentType='text/plain'
    )

    # Calculate expiration
    expires_at = parse_expiration(paste.expiration)

    # Store metadata in database
    async with db_pool.acquire() as conn:
        await conn.execute(
            """
            INSERT INTO pastes (paste_id, title, content_url, size_bytes,
                                syntax, privacy, created_at, expires_at)
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
            """,
            paste_id,
            paste.title,
            f"s3://{BUCKET_NAME}/{s3_key}",
            len(paste.content.encode()),
            paste.syntax,
            paste.privacy,
            datetime.utcnow(),
            expires_at
        )

    return PasteResponse(
        paste_id=paste_id,
        url=f"https://pastebin.com/{paste_id}",
        raw_url=f"https://pastebin.com/raw/{paste_id}",
        created_at=datetime.utcnow(),
        expires_at=expires_at
    )

@app.get("/api/v1/pastes/{paste_id}")
async def get_paste(paste_id: str):
    """Retrieve a paste"""

    async with db_pool.acquire() as conn:
        # Fetch metadata
        row = await conn.fetchrow(
            "SELECT * FROM pastes WHERE paste_id = $1",
            paste_id
        )

        if not row:
            raise HTTPException(status_code=404, detail="Paste not found")

        # Check expiration
        if row['expires_at'] and row['expires_at'] < datetime.utcnow():
            raise HTTPException(status_code=410, detail="Paste has expired")

        # Fetch content from S3
        s3_key = row['content_url'].replace(f"s3://{BUCKET_NAME}/", "")
        obj = s3_client.get_object(Bucket=BUCKET_NAME, Key=s3_key)
        content = obj['Body'].read().decode()

        # Async increment view count (fire and forget)
        await conn.execute(
            "UPDATE pastes SET view_count = view_count + 1 WHERE paste_id = $1",
            paste_id
        )

        return {
            "paste_id": row['paste_id'],
            "title": row['title'],
            "content": content,
            "syntax": row['syntax'],
            "created_at": row['created_at'],
            "expires_at": row['expires_at'],
            "view_count": row['view_count'] + 1,
            "privacy": row['privacy']
        }

@app.delete("/api/v1/pastes/{paste_id}", status_code=204)
async def delete_paste(paste_id: str):
    """Delete a paste"""

    async with db_pool.acquire() as conn:
        row = await conn.fetchrow(
            "SELECT content_url FROM pastes WHERE paste_id = $1",
            paste_id
        )

        if not row:
            raise HTTPException(status_code=404, detail="Paste not found")

        # Delete from S3
        s3_key = row['content_url'].replace(f"s3://{BUCKET_NAME}/", "")
        s3_client.delete_object(Bucket=BUCKET_NAME, Key=s3_key)

        # Delete from database
        await conn.execute(
            "DELETE FROM pastes WHERE paste_id = $1",
            paste_id
        )

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## References

### Related Designs
- [URL Shortener](../url-shortener/) - Similar short URL generation
- [File Storage](../file-storage/) - Similar S3 usage pattern
- [Unique ID Generator](../unique-id-generator/) - ID generation strategies

### Real-World Examples
- **Pastebin.com** - Original implementation
- **GitHub Gist** - Code snippet sharing
- **paste.ubuntu.com** - Ubuntu's implementation

---

## Interview Tips

### Key Points to Emphasize

1. **Object Storage Trade-off:** Database vs S3 cost comparison (80% savings)
2. **Caching Strategy:** Multi-level (CDN + Redis) for 95% cache hit ratio
3. **Expiration Handling:** Hybrid passive + active cleanup
4. **Scalability:** S3 provides unlimited storage, CDN handles viral pastes

### Common Mistakes

- Storing large content in database (cost prohibitive)
- Forgetting to handle expired pastes
- Not considering viral paste scenarios
- Ignoring storage costs in design

### How to Stand Out

- Discuss cost optimization (S3 vs DB)
- Mention CDN integration for popular pastes
- Explain async view count updates to reduce DB load
- Consider abuse prevention (rate limiting, content validation)

---

**Summary:** Pastebin is a simpler version of URL shortener focused on text storage. Key differences are using object storage (S3) for cost efficiency and handling expiration policies. The design scales well with CDN caching and can handle millions of pastes cost-effectively.
