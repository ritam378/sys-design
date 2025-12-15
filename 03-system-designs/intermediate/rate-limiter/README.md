# Design a Rate Limiter

## Table of Contents
- [Problem Statement](#problem-statement)
- [Requirements](#requirements)
- [Rate Limiting Algorithms](#rate-limiting-algorithms)
- [High-Level Design](#high-level-design)
- [Detailed Design](#detailed-design)
- [Distributed Rate Limiting](#distributed-rate-limiting)
- [API Design](#api-design)
- [Implementation Examples](#implementation-examples)
- [Response Headers](#response-headers)
- [Rate Limiting Rules](#rate-limiting-rules)
- [Failure Handling](#failure-handling)
- [Optimizations](#optimizations)
- [Real-World Examples](#real-world-examples)
- [Interview Tips](#interview-tips)

---

## Problem Statement

Design a **rate limiter** that:
- Limits the number of requests a user/client can make in a given time window
- Prevents abuse and protects backend services from overload
- Provides fair resource allocation among users
- Scales to handle millions of users and billions of requests

**Similar to**: Stripe API rate limiter, GitHub API, Twitter API, CloudFlare rate limiting

---

## Requirements

### Functional Requirements

1. **Limit requests** based on:
   - User ID
   - IP address
   - API key
   - Geographic region
2. **Configurable limits**: Different limits for different APIs
3. **Return clear feedback**: Rate limit headers, error messages
4. **Multiple time windows**: Per second, minute, hour, day
5. **Tiered limits**: Different limits for different user tiers (free, premium)

### Non-Functional Requirements

1. **Low Latency**: < 5ms overhead per request
2. **Highly Available**: 99.99% uptime
3. **Fault Tolerant**: Continue operating if rate limiter fails
4. **Scalable**: Handle millions of users concurrently
5. **Accurate**: Minimal false positives/negatives
6. **Distributed**: Work across multiple servers

### Extended Requirements

1. **Analytics**: Track rate limit violations
2. **Dynamic limits**: Adjust based on system load
3. **Whitelist/Blacklist**: Bypass or block specific users
4. **Gradual rollout**: Percentage-based enabling

---

## Rate Limiting Algorithms

### 1. Token Bucket Algorithm

**Most Popular** - Used by Amazon, Stripe

**Concept:**
- Bucket holds tokens (max capacity)
- Tokens refilled at constant rate
- Each request consumes one token
- If no tokens available, request rejected

```
Bucket capacity: 10 tokens
Refill rate: 5 tokens/second

Time  Tokens  Request  Result
0s    10      -        -
1s    10      ✓        Accept (9 tokens left)
2s    10      ✓        Accept (9 tokens left, refilled to 10)
3s    10      ✓×10     Accept all (0 tokens left)
3.5s  2       ✓        Accept (1 token left, refilled 2.5)
3.6s  2       ✓        Accept (1 token left)
3.7s  2       ✓        Accept (0 tokens left)
3.8s  2       ✓        Reject (0 tokens, only 0.5 refilled)
```

**Implementation:**

```python
import time

class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity  # Max tokens
        self.tokens = capacity    # Current tokens
        self.refill_rate = refill_rate  # Tokens per second
        self.last_refill = time.time()

    def allow_request(self):
        """Check if request is allowed"""
        self._refill()

        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False

    def _refill(self):
        """Refill tokens based on time elapsed"""
        now = time.time()
        elapsed = now - self.last_refill
        tokens_to_add = elapsed * self.refill_rate

        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now

# Usage
bucket = TokenBucket(capacity=10, refill_rate=5)

for i in range(15):
    if bucket.allow_request():
        print(f"Request {i}: Allowed")
    else:
        print(f"Request {i}: Rejected")
    time.sleep(0.1)
```

**Pros:**
- Burst handling (accumulate tokens during idle)
- Memory efficient (per-user state is small)
- Smooth rate limiting

**Cons:**
- Challenging to tune (capacity vs refill rate)
- Can allow bursts exceeding desired rate

---

### 2. Leaky Bucket Algorithm

**Concept:**
- Requests enter bucket at any rate
- Requests processed at constant rate
- If bucket full, requests rejected
- Like water dripping from bucket with hole

```
Bucket capacity: 10 requests
Processing rate: 5 requests/second

Incoming: 20 req/s → [Bucket: 10] → Outgoing: 5 req/s
                     (overflow rejected)
```

**Implementation:**

```python
from collections import deque
import time
import threading

class LeakyBucket:
    def __init__(self, capacity, leak_rate):
        self.capacity = capacity
        self.leak_rate = leak_rate  # Requests per second
        self.queue = deque()
        self.last_leak = time.time()
        self.lock = threading.Lock()

    def allow_request(self):
        """Add request to bucket"""
        with self.lock:
            self._leak()

            if len(self.queue) < self.capacity:
                self.queue.append(time.time())
                return True
            return False

    def _leak(self):
        """Process requests at constant rate"""
        now = time.time()
        elapsed = now - self.last_leak
        leaks = int(elapsed * self.leak_rate)

        for _ in range(min(leaks, len(self.queue))):
            self.queue.popleft()

        self.last_leak = now

# Usage
bucket = LeakyBucket(capacity=10, leak_rate=5)
```

**Pros:**
- Constant output rate (smooths bursts)
- Simple implementation

**Cons:**
- Doesn't handle bursts well
- Recent requests can be rejected even if system idle

---

### 3. Fixed Window Counter

**Simplest** - Easy to implement

**Concept:**
- Divide time into fixed windows (e.g., 1 minute)
- Count requests in each window
- Reset counter at window boundary

```
Limit: 100 requests per minute

12:00:00 - 12:00:59  →  100 requests (allowed)
12:01:00 - 12:01:59  →  100 requests (allowed)

Problem: Burst at boundary
12:00:30 - 12:00:59  →  100 requests
12:01:00 - 12:01:29  →  100 requests
Total in 60s: 200 requests (double the limit!)
```

**Implementation:**

```python
import time
from collections import defaultdict

class FixedWindowCounter:
    def __init__(self, limit, window_size):
        self.limit = limit
        self.window_size = window_size  # seconds
        self.windows = defaultdict(int)  # window_start -> count

    def allow_request(self, user_id):
        """Check if request is allowed"""
        now = time.time()
        window_start = int(now // self.window_size) * self.window_size
        key = f"{user_id}:{window_start}"

        if self.windows[key] < self.limit:
            self.windows[key] += 1
            return True
        return False

    def cleanup_old_windows(self):
        """Remove expired windows"""
        now = time.time()
        current_window = int(now // self.window_size) * self.window_size
        keys_to_delete = [
            k for k in self.windows
            if int(k.split(':')[1]) < current_window - self.window_size
        ]
        for key in keys_to_delete:
            del self.windows[key]

# Usage
limiter = FixedWindowCounter(limit=100, window_size=60)

if limiter.allow_request('user123'):
    print("Request allowed")
else:
    print("Rate limit exceeded")
```

**Pros:**
- Very simple to implement
- Memory efficient
- Easy to understand

**Cons:**
- Boundary burst issue
- Can allow 2x limit in edge cases

---

### 4. Sliding Window Log

**Most Accurate** - No boundary issues

**Concept:**
- Keep timestamp of each request
- Count requests in sliding window
- Remove timestamps outside window

```
Limit: 10 requests per minute
Current time: 12:01:30

Request timestamps:
12:00:45, 12:00:50, 12:01:00, 12:01:10, 12:01:20, 12:01:25

Sliding window: [12:00:30 - 12:01:30]
Valid requests: 12:00:45, 12:00:50, 12:01:00, 12:01:10, 12:01:20, 12:01:25 = 6
Can accept: 10 - 6 = 4 more requests
```

**Implementation:**

```python
import time
from collections import defaultdict, deque

class SlidingWindowLog:
    def __init__(self, limit, window_size):
        self.limit = limit
        self.window_size = window_size  # seconds
        self.logs = defaultdict(deque)  # user_id -> [timestamps]

    def allow_request(self, user_id):
        """Check if request is allowed"""
        now = time.time()
        window_start = now - self.window_size

        # Remove old timestamps
        while self.logs[user_id] and self.logs[user_id][0] < window_start:
            self.logs[user_id].popleft()

        # Check limit
        if len(self.logs[user_id]) < self.limit:
            self.logs[user_id].append(now)
            return True
        return False

# Usage
limiter = SlidingWindowLog(limit=10, window_size=60)

for i in range(15):
    if limiter.allow_request('user123'):
        print(f"Request {i}: Allowed")
    else:
        print(f"Request {i}: Rejected")
    time.sleep(1)
```

**Pros:**
- Very accurate (no boundary issues)
- Precise rate limiting

**Cons:**
- High memory usage (store all timestamps)
- Expensive for high traffic (sorting, cleanup)

---

### 5. Sliding Window Counter

**Best Balance** - Combines fixed window + sliding window

**Concept:**
- Approximate sliding window using two fixed windows
- Weighted count based on overlap

```
Limit: 100 requests per minute
Current time: 12:01:30 (50% into window)

Previous window [12:00:00-12:01:00]: 80 requests
Current window [12:01:00-12:02:00]: 40 requests

Approximate count:
= (prev_count × overlap%) + current_count
= (80 × 50%) + 40
= 40 + 40
= 80 requests

Can accept: 100 - 80 = 20 more requests
```

**Implementation:**

```python
import time
from collections import defaultdict

class SlidingWindowCounter:
    def __init__(self, limit, window_size):
        self.limit = limit
        self.window_size = window_size
        self.windows = defaultdict(lambda: {'prev': 0, 'curr': 0, 'curr_start': 0})

    def allow_request(self, user_id):
        """Check if request is allowed"""
        now = time.time()
        window_start = int(now // self.window_size) * self.window_size
        window_data = self.windows[user_id]

        # New window started
        if window_start != window_data['curr_start']:
            window_data['prev'] = window_data['curr']
            window_data['curr'] = 0
            window_data['curr_start'] = window_start

        # Calculate weighted count
        time_in_window = now - window_start
        prev_weight = 1 - (time_in_window / self.window_size)
        estimated_count = (
            window_data['prev'] * prev_weight +
            window_data['curr']
        )

        if estimated_count < self.limit:
            window_data['curr'] += 1
            return True
        return False

# Usage
limiter = SlidingWindowCounter(limit=100, window_size=60)
```

**Pros:**
- Memory efficient (only 2 counters)
- Smooth rate limiting
- Handles bursts better than fixed window

**Cons:**
- Approximation (not 100% accurate)
- Slightly more complex than fixed window

---

## Algorithm Comparison

| Algorithm | Accuracy | Memory | Performance | Burst Handling | Complexity |
|-----------|----------|--------|-------------|----------------|------------|
| **Token Bucket** | Good | Low | High | Excellent | Medium |
| **Leaky Bucket** | Good | Low | High | Poor | Medium |
| **Fixed Window** | Poor | Low | High | Poor | Low |
| **Sliding Log** | Excellent | High | Low | Excellent | High |
| **Sliding Counter** | Good | Low | High | Good | Medium |

**Recommendation:**
- **General purpose**: Token Bucket or Sliding Window Counter
- **Strict rate limiting**: Sliding Window Log
- **Simple requirements**: Fixed Window Counter
- **Smooth traffic**: Leaky Bucket

---

## High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│                         Client                               │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────────┐
│                      Load Balancer                           │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────────┐
│                  API Gateway / Rate Limiter                  │
│                                                              │
│  1. Extract user ID / IP / API key                          │
│  2. Check rate limit (Redis)                                │
│  3. If allowed → forward request                            │
│  4. If rejected → return 429 error                          │
└────────┬───────────────────────┬────────────────────────────┘
         │                       │
         ↓                       ↓
   ┌─────────┐            ┌──────────────┐
   │  Redis  │            │  Rules Store │
   │ Cluster │            │  (Config DB) │
   │         │            │              │
   │ Counter │            │ - Limits     │
   │  Data   │            │ - Whitelist  │
   └─────────┘            │ - Blacklist  │
                          └──────────────┘
         │
         ↓
┌─────────────────────────────────────────────────────────────┐
│                    Backend Services                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │
│  │ Service A│  │ Service B│  │ Service C│                  │
│  └──────────┘  └──────────┘  └──────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

---

## Detailed Design

### Rate Limiter Middleware

```python
from flask import Flask, request, jsonify
import redis
import time

app = Flask(__name__)
redis_client = redis.Redis(host='localhost', port=6379, db=0)

class RateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client

    def check_rate_limit(self, user_id, limit=100, window=60):
        """
        Sliding window counter algorithm using Redis

        Args:
            user_id: User identifier
            limit: Max requests per window
            window: Time window in seconds

        Returns:
            (allowed: bool, retry_after: int)
        """
        key = f"rate_limit:{user_id}"
        now = time.time()
        window_start = now - window

        # Remove old entries
        self.redis.zremrangebyscore(key, 0, window_start)

        # Count requests in window
        request_count = self.redis.zcard(key)

        if request_count < limit:
            # Add current request
            self.redis.zadd(key, {now: now})
            self.redis.expire(key, window)
            return True, 0
        else:
            # Get oldest request timestamp
            oldest = self.redis.zrange(key, 0, 0, withscores=True)
            if oldest:
                retry_after = int(oldest[0][1] + window - now)
                return False, retry_after
            return False, window

limiter = RateLimiter(redis_client)

@app.before_request
def rate_limit_check():
    """Rate limit middleware"""
    # Extract identifier (user_id, IP, API key)
    user_id = request.headers.get('X-User-ID') or request.remote_addr

    # Check rate limit
    allowed, retry_after = limiter.check_rate_limit(user_id)

    if not allowed:
        return jsonify({
            'error': 'Rate limit exceeded',
            'retry_after': retry_after
        }), 429, {
            'X-RateLimit-Limit': '100',
            'X-RateLimit-Remaining': '0',
            'X-RateLimit-Reset': str(int(time.time()) + retry_after),
            'Retry-After': str(retry_after)
        }

@app.route('/api/users')
def get_users():
    return jsonify({'users': ['Alice', 'Bob', 'Charlie']})

if __name__ == '__main__':
    app.run(debug=True)
```

---

## Distributed Rate Limiting

### Challenges

1. **Race Conditions**: Multiple servers incrementing counter simultaneously
2. **Synchronization**: Keeping counters in sync across servers
3. **Network Latency**: Delays in updating central counter

### Solution 1: Redis Atomic Operations

```python
class DistributedRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client

    def sliding_window_counter(self, user_id, limit=100, window=60):
        """
        Distributed sliding window using Redis sorted sets

        Uses sorted set with timestamps as scores
        Atomic operations prevent race conditions
        """
        key = f"rate_limit:{user_id}"
        now = time.time()
        window_start = now - window

        # Use Lua script for atomicity
        lua_script = """
        local key = KEYS[1]
        local now = tonumber(ARGV[1])
        local window = tonumber(ARGV[2])
        local limit = tonumber(ARGV[3])
        local window_start = now - window

        -- Remove old entries
        redis.call('ZREMRANGEBYSCORE', key, 0, window_start)

        -- Get current count
        local count = redis.call('ZCARD', key)

        if count < limit then
            -- Add new entry
            redis.call('ZADD', key, now, now)
            redis.call('EXPIRE', key, window)
            return {1, limit - count - 1}
        else
            return {0, 0}
        end
        """

        result = self.redis.eval(
            lua_script,
            1,  # Number of keys
            key,  # KEYS[1]
            now, window, limit  # ARGV[1], ARGV[2], ARGV[3]
        )

        allowed = bool(result[0])
        remaining = result[1]
        return allowed, remaining

# Usage
limiter = DistributedRateLimiter(redis_client)
allowed, remaining = limiter.sliding_window_counter('user123')
```

### Solution 2: Token Bucket with Redis

```python
class RedisTokenBucket:
    def __init__(self, redis_client):
        self.redis = redis_client

    def allow_request(self, user_id, capacity=10, refill_rate=1):
        """
        Token bucket implemented in Redis

        Stores: {tokens: float, last_refill: timestamp}
        """
        key = f"token_bucket:{user_id}"

        lua_script = """
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local refill_rate = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])

        -- Get current state
        local bucket = redis.call('HGETALL', key)
        local tokens = capacity
        local last_refill = now

        if #bucket > 0 then
            tokens = tonumber(bucket[2])
            last_refill = tonumber(bucket[4])
        end

        -- Refill tokens
        local elapsed = now - last_refill
        local new_tokens = math.min(capacity, tokens + (elapsed * refill_rate))

        -- Try to consume token
        if new_tokens >= 1 then
            new_tokens = new_tokens - 1
            redis.call('HSET', key, 'tokens', new_tokens, 'last_refill', now)
            redis.call('EXPIRE', key, 3600)
            return {1, math.floor(new_tokens)}
        else
            redis.call('HSET', key, 'tokens', new_tokens, 'last_refill', now)
            redis.call('EXPIRE', key, 3600)
            return {0, 0}
        end
        """

        result = self.redis.eval(
            lua_script,
            1,
            key,
            capacity, refill_rate, time.time()
        )

        return bool(result[0]), result[1]
```

### Solution 3: Sticky Sessions

Route same user to same server (reduces synchronization).

```python
class StickySessionRateLimiter:
    """
    Use consistent hashing to route users to same server
    Each server maintains local counters
    """
    def __init__(self, server_id, total_servers):
        self.server_id = server_id
        self.total_servers = total_servers
        self.local_limiters = {}

    def get_server(self, user_id):
        """Hash user to server"""
        return hash(user_id) % self.total_servers

    def allow_request(self, user_id):
        """Check rate limit on assigned server"""
        if self.get_server(user_id) != self.server_id:
            # Route to correct server
            return self.forward_to_server(user_id)

        # Check local limiter
        if user_id not in self.local_limiters:
            self.local_limiters[user_id] = TokenBucket(10, 1)

        return self.local_limiters[user_id].allow_request()
```

---

## API Design

### Rate Limiter Configuration

```json
{
  "rules": [
    {
      "name": "default",
      "limit": 1000,
      "window": 3600,
      "unit": "hour",
      "scope": "user"
    },
    {
      "name": "login",
      "endpoint": "/api/auth/login",
      "limit": 5,
      "window": 60,
      "unit": "minute",
      "scope": "ip"
    },
    {
      "name": "premium_user",
      "limit": 10000,
      "window": 3600,
      "unit": "hour",
      "scope": "user",
      "condition": "user.tier == 'premium'"
    }
  ],
  "whitelist": ["admin-api-key", "monitoring-service"],
  "blacklist": ["abusive-user-123"]
}
```

### Dynamic Configuration

```python
class RateLimitConfig:
    def __init__(self, config_store):
        self.config_store = config_store  # Database or file
        self.cache = {}
        self.load_config()

    def load_config(self):
        """Load rules from config store"""
        self.cache = self.config_store.get_all_rules()

    def get_limit(self, user_id, endpoint):
        """Get limit for user and endpoint"""
        # Check whitelist
        if user_id in self.cache.get('whitelist', []):
            return float('inf'), 0  # No limit

        # Check blacklist
        if user_id in self.cache.get('blacklist', []):
            return 0, 0  # Complete block

        # Find matching rule
        for rule in self.cache.get('rules', []):
            if self.matches_rule(rule, user_id, endpoint):
                return rule['limit'], rule['window']

        # Default limit
        return 1000, 3600

    def matches_rule(self, rule, user_id, endpoint):
        """Check if rule applies"""
        if 'endpoint' in rule and rule['endpoint'] != endpoint:
            return False

        if 'condition' in rule:
            # Evaluate condition (e.g., user tier)
            return self.evaluate_condition(rule['condition'], user_id)

        return True
```

---

## Response Headers

### Standard Headers

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000           # Max requests per window
X-RateLimit-Remaining: 987        # Requests remaining
X-RateLimit-Reset: 1640000000     # Unix timestamp when limit resets
X-RateLimit-RetryAfter: 13        # Seconds until retry (if rejected)
```

### 429 Too Many Requests Response

```
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640000060
Retry-After: 60

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please try again later.",
    "retry_after": 60
  }
}
```

### Implementation

```python
def add_rate_limit_headers(response, limit, remaining, reset_time):
    """Add rate limit headers to response"""
    response.headers['X-RateLimit-Limit'] = str(limit)
    response.headers['X-RateLimit-Remaining'] = str(remaining)
    response.headers['X-RateLimit-Reset'] = str(reset_time)

    if remaining == 0:
        retry_after = reset_time - int(time.time())
        response.headers['Retry-After'] = str(retry_after)

    return response
```

---

## Rate Limiting Rules

### Multi-Tier Rate Limiting

```python
class TieredRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.tiers = {
            'free': {'limit': 100, 'window': 3600},
            'basic': {'limit': 1000, 'window': 3600},
            'premium': {'limit': 10000, 'window': 3600},
            'enterprise': {'limit': float('inf'), 'window': 3600}
        }

    def get_user_tier(self, user_id):
        """Fetch user tier from database"""
        # In production, query database
        return 'basic'

    def check_limit(self, user_id):
        tier = self.get_user_tier(user_id)
        config = self.tiers[tier]

        # Apply tier-specific limit
        return self.check_rate_limit(
            user_id,
            limit=config['limit'],
            window=config['window']
        )
```

### Endpoint-Specific Limits

```python
class EndpointRateLimiter:
    def __init__(self):
        self.limits = {
            '/api/auth/login': {'limit': 5, 'window': 60},
            '/api/auth/signup': {'limit': 3, 'window': 3600},
            '/api/search': {'limit': 100, 'window': 60},
            '/api/users': {'limit': 1000, 'window': 3600},
            'default': {'limit': 1000, 'window': 3600}
        }

    def get_limit_for_endpoint(self, endpoint):
        return self.limits.get(endpoint, self.limits['default'])

    def check_limit(self, user_id, endpoint):
        config = self.get_limit_for_endpoint(endpoint)
        key = f"rate_limit:{user_id}:{endpoint}"

        # Check rate limit with endpoint-specific config
        return self.check_rate_limit(
            key,
            limit=config['limit'],
            window=config['window']
        )
```

### Geographic Rate Limiting

```python
class GeoRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.region_limits = {
            'US': 1000,
            'EU': 1000,
            'ASIA': 500,    # Lower limit due to abuse
            'OTHER': 100
        }

    def get_region(self, ip_address):
        """Get region from IP (use GeoIP database)"""
        # In production, use MaxMind GeoIP2 or similar
        return 'US'

    def check_limit(self, ip_address):
        region = self.get_region(ip_address)
        limit = self.region_limits.get(region, 100)

        return self.check_rate_limit(
            f"geo:{region}:{ip_address}",
            limit=limit,
            window=3600
        )
```

---

## Failure Handling

### Fail-Open Strategy

**If rate limiter fails, allow requests through** (prefer availability)

```python
class FailOpenRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client

    def check_rate_limit(self, user_id):
        try:
            # Try to check rate limit
            return self._check_limit_redis(user_id)
        except redis.RedisError:
            # Redis down - fail open (allow request)
            print("Rate limiter unavailable - allowing request")
            return True, 0
        except Exception as e:
            # Unknown error - fail open
            print(f"Rate limiter error: {e} - allowing request")
            return True, 0
```

### Fail-Closed Strategy

**If rate limiter fails, reject requests** (prefer security)

```python
class FailClosedRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client

    def check_rate_limit(self, user_id):
        try:
            return self._check_limit_redis(user_id)
        except Exception as e:
            # Rate limiter error - fail closed (reject request)
            print(f"Rate limiter error: {e} - rejecting request")
            return False, 60  # Reject with retry after 60s
```

### Circuit Breaker

**Temporarily disable rate limiter if Redis keeps failing**

```python
class CircuitBreakerRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.failure_count = 0
        self.circuit_open = False
        self.last_failure = 0

    def check_rate_limit(self, user_id):
        # If circuit open, bypass rate limiting
        if self.circuit_open:
            if time.time() - self.last_failure > 60:  # 60s cooldown
                self.circuit_open = False
                self.failure_count = 0
            else:
                return True, 0  # Bypass

        try:
            result = self._check_limit_redis(user_id)
            self.failure_count = 0  # Reset on success
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure = time.time()

            # Open circuit after 5 failures
            if self.failure_count >= 5:
                self.circuit_open = True
                print("Circuit breaker OPEN - bypassing rate limiter")

            return True, 0  # Fail open
```

---

## Optimizations

### 1. Caching Rate Limit Rules

```python
from functools import lru_cache

class CachedRateLimiter:
    @lru_cache(maxsize=10000)
    def get_user_limit(self, user_id):
        """Cache user limits to avoid database queries"""
        return self.db.get_user_limit(user_id)
```

### 2. Local Rate Limiting (Before Redis)

```python
class LocalCacheRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.local_cache = {}  # user_id -> (count, window_start)

    def check_rate_limit(self, user_id, limit=100, window=60):
        """
        Check local cache first (fast)
        If uncertain, check Redis (accurate)
        """
        now = time.time()

        # Check local cache
        if user_id in self.local_cache:
            count, window_start = self.local_cache[user_id]

            # Window expired
            if now - window_start >= window:
                self.local_cache[user_id] = (1, now)
                return True, limit - 1

            # Definitely over limit
            if count >= limit:
                return False, 0

            # Under 90% of limit - allow locally
            if count < limit * 0.9:
                self.local_cache[user_id] = (count + 1, window_start)
                return True, limit - count - 1

        # Check Redis for accurate count
        return self._check_redis(user_id, limit, window)
```

### 3. Batch Processing

```python
class BatchRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.pending_requests = []

    def queue_request(self, user_id):
        """Queue request for batch processing"""
        self.pending_requests.append(user_id)

        # Process batch when size reaches threshold
        if len(self.pending_requests) >= 100:
            self.process_batch()

    def process_batch(self):
        """Process multiple rate limit checks in one Redis call"""
        pipeline = self.redis.pipeline()

        for user_id in self.pending_requests:
            key = f"rate_limit:{user_id}"
            pipeline.incr(key)
            pipeline.expire(key, 60)

        results = pipeline.execute()
        self.pending_requests.clear()
        return results
```

---

## Real-World Examples

### 1. Stripe API

**Rate Limits:**
- 100 read requests per second
- 25 write requests per second
- Burst allowance with token bucket

**Implementation:**
```python
# Stripe-style rate limiter
class StripeRateLimiter:
    def __init__(self):
        self.read_limiter = TokenBucket(capacity=100, refill_rate=100)
        self.write_limiter = TokenBucket(capacity=25, refill_rate=25)

    def check_limit(self, endpoint, method):
        if method in ['GET', 'HEAD']:
            return self.read_limiter.allow_request()
        else:  # POST, PUT, DELETE
            return self.write_limiter.allow_request()
```

### 2. GitHub API

**Rate Limits:**
- 5,000 requests per hour (authenticated)
- 60 requests per hour (unauthenticated)
- Per-user limits

**Headers:**
```
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4999
X-RateLimit-Reset: 1640000000
X-RateLimit-Used: 1
X-RateLimit-Resource: core
```

### 3. Twitter API

**Rate Limits:**
- Different limits per endpoint
- 15-minute windows
- User-based and app-based limits

**Example:**
```
/api/statuses/user_timeline: 900 requests per 15 minutes
/api/search/tweets: 450 requests per 15 minutes
/api/followers/list: 15 requests per 15 minutes
```

### 4. Cloudflare Rate Limiting

**Features:**
- Zone-level rate limiting
- Rule-based (URL, headers, method)
- Block or challenge
- Analytics and alerts

```json
{
  "id": "rule_id",
  "description": "Protect login endpoint",
  "match": {
    "request": {
      "url": "*/login",
      "methods": ["POST"]
    }
  },
  "threshold": 5,
  "period": 60,
  "action": {
    "mode": "challenge",
    "timeout": 300
  }
}
```

---

## Interview Tips

### Key Concepts to Explain

1. **Algorithm Choice**:
   - Token Bucket: Burst handling, smooth rate
   - Sliding Window: Accurate, memory-efficient
   - Fixed Window: Simple but has boundary issues

2. **Distributed Challenges**:
   - Race conditions (use Redis Lua scripts)
   - Synchronization overhead
   - Eventual consistency

3. **Trade-offs**:
   - Accuracy vs Performance
   - Memory vs Precision
   - Fail-open vs Fail-closed

### Common Interview Questions

**Q1: How would you implement a distributed rate limiter?**
- Use Redis with atomic operations (Lua scripts)
- Consistent hashing for partitioning
- Choose algorithm (token bucket or sliding window)

**Q2: How do you handle race conditions?**
- Redis atomic operations (INCR, Lua scripts)
- Optimistic locking with WATCH/MULTI/EXEC
- Accept minor inconsistency for performance

**Q3: What happens if Redis goes down?**
- Fail-open: Allow all requests (availability)
- Fail-closed: Reject all requests (security)
- Circuit breaker: Temporarily bypass rate limiter

**Q4: How do you rate limit across multiple regions?**
- Global: Centralized Redis (higher latency)
- Regional: Local Redis per region (inconsistent)
- Hybrid: Local + eventual sync

**Q5: How do you handle different user tiers?**
- Store tier in database
- Look up tier before applying limit
- Cache tier to reduce database queries

### Design Interview Template

1. **Clarify requirements**: QPS, user count, accuracy needed
2. **Choose algorithm**: Token bucket or sliding window counter
3. **Design storage**: Redis with atomic operations
4. **Handle distribution**: Lua scripts, race conditions
5. **Define rules**: Multi-tier, endpoint-specific, geographic
6. **Error handling**: Fail-open vs fail-closed
7. **Optimizations**: Caching, local rate limiting
8. **Monitoring**: Track violations, adjust limits dynamically

---

## Summary

**Rate limiting** is essential for API protection, resource management, and fair usage enforcement.

**Key Takeaways:**

1. **Algorithms**:
   - Token Bucket: Best for burst handling
   - Sliding Window Counter: Best balance of accuracy and performance
   - Sliding Window Log: Most accurate but expensive

2. **Implementation**:
   - Use Redis for distributed state
   - Lua scripts for atomic operations
   - Consistent hashing for partitioning

3. **Advanced Features**:
   - Multi-tier limits (free, premium, enterprise)
   - Endpoint-specific rules
   - Geographic rate limiting
   - Dynamic configuration

4. **Production Considerations**:
   - Fail-open for availability
   - Circuit breaker for resilience
   - Monitoring and analytics
   - Clear error messages and headers

**Real-World Usage:**
- **Stripe**: Token bucket with read/write separation
- **GitHub**: Fixed window with per-user tracking
- **Twitter**: Endpoint-specific sliding windows
- **Cloudflare**: Rule-based with DDoS protection

This design is a critical component of modern API infrastructure and appears frequently in system design interviews.
