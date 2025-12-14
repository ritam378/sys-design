# Rate Limiter Building Block

## Overview

A rate limiter restricts the number of requests a client can make to an API within a specified time window. It's essential for:
- Preventing abuse and DDoS attacks
- Ensuring fair resource allocation
- Controlling costs for paid APIs
- Maintaining system stability

## Common Use Cases

- **API throttling:** GitHub allows 5,000 requests/hour per authenticated user
- **Login attempts:** Limit to 5 failed logins per 15 minutes
- **Message sending:** Limit SMS to 10 messages/minute to prevent spam
- **Resource protection:** Prevent database overload from expensive queries

## Rate Limiting Algorithms

### Comparison Table

| Algorithm | Pros | Cons | Use Case |
|-----------|------|------|----------|
| **Token Bucket** | Smooth bursts, flexible | Complex implementation | General purpose (recommended) |
| **Leaky Bucket** | Steady output rate | Rigid, no bursts | Video streaming, data transfer |
| **Fixed Window** | Simple, memory efficient | Burst at window edges | Simple rate limiting |
| **Sliding Window Log** | Accurate, no edge bursts | Memory intensive | Financial APIs, strict limits |
| **Sliding Window Counter** | Accurate, memory efficient | Slightly complex | Production-ready (best) |

---

## 1. Token Bucket Algorithm (Recommended)

### How It Works

Imagine a bucket that holds tokens:
1. Bucket has a **maximum capacity** (e.g., 100 tokens)
2. Tokens are added at a **fixed rate** (e.g., 10 tokens/second)
3. Each request consumes **1 token** (or more for weighted requests)
4. If no tokens available → request is rejected
5. Bucket never exceeds max capacity

### Visual Representation

```
Time: 0s          Time: 1s          Time: 2s
[##########]      [##########]      [##########]
[##########] 100  [##########] 100  [####      ] 40
[##########]  →   [##########]  →   [          ]
  (refill)         (use 60)          (refill +10)

Bucket capacity: 100 tokens
Refill rate: 10 tokens/second
Request consumes: varies per request
```

### Characteristics

**Pros:**
- ✅ Allows **traffic bursts** (use accumulated tokens)
- ✅ Flexible (different costs per request)
- ✅ Smooth traffic shaping
- ✅ Industry standard (used by AWS, Stripe)

**Cons:**
- ❌ Slightly complex to implement
- ❌ Requires atomic operations (race conditions)

### Python Implementation

```python
import time
from threading import Lock

class TokenBucket:
    """
    Token Bucket Rate Limiter

    Parameters:
        capacity: Maximum number of tokens in bucket
        refill_rate: Tokens added per second
    """

    def __init__(self, capacity: int, refill_rate: float):
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.tokens = capacity
        self.last_refill = time.time()
        self.lock = Lock()

    def _refill(self):
        """Refill tokens based on time elapsed"""
        now = time.time()
        elapsed = now - self.last_refill

        # Calculate tokens to add
        tokens_to_add = elapsed * self.refill_rate

        # Update tokens (don't exceed capacity)
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now

    def allow_request(self, tokens: int = 1) -> bool:
        """
        Check if request is allowed

        Args:
            tokens: Number of tokens to consume (default 1)

        Returns:
            True if request allowed, False otherwise
        """
        with self.lock:
            self._refill()

            if self.tokens >= tokens:
                self.tokens -= tokens
                return True

            return False

    def get_tokens(self) -> float:
        """Get current token count"""
        with self.lock:
            self._refill()
            return self.tokens


# Usage Example:
if __name__ == "__main__":
    # Allow 10 requests per second, burst up to 100
    limiter = TokenBucket(capacity=100, refill_rate=10)

    # Simulate requests
    allowed = 0
    rejected = 0

    for i in range(150):
        if limiter.allow_request():
            allowed += 1
            print(f"Request {i+1}: ALLOWED (tokens: {limiter.get_tokens():.2f})")
        else:
            rejected += 1
            print(f"Request {i+1}: REJECTED (tokens: {limiter.get_tokens():.2f})")

        time.sleep(0.05)  # 50ms between requests

    print(f"\nAllowed: {allowed}, Rejected: {rejected}")
```

### Redis Implementation (Distributed)

```python
import redis
import time

class DistributedTokenBucket:
    """
    Distributed Token Bucket using Redis

    Supports multiple servers accessing same rate limit
    """

    def __init__(self, redis_client: redis.Redis, capacity: int, refill_rate: float):
        self.redis = redis_client
        self.capacity = capacity
        self.refill_rate = refill_rate

    def allow_request(self, key: str, tokens: int = 1) -> bool:
        """
        Check if request is allowed for given key

        Args:
            key: Unique identifier (e.g., "user:12345", "ip:192.168.1.1")
            tokens: Number of tokens to consume

        Returns:
            True if request allowed, False otherwise
        """
        now = time.time()

        # Lua script for atomic operations
        lua_script = """
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local refill_rate = tonumber(ARGV[2])
        local requested_tokens = tonumber(ARGV[3])
        local now = tonumber(ARGV[4])

        -- Get current tokens and last refill time
        local current_tokens = tonumber(redis.call('HGET', key, 'tokens'))
        local last_refill = tonumber(redis.call('HGET', key, 'last_refill'))

        -- Initialize if first time
        if current_tokens == nil then
            current_tokens = capacity
            last_refill = now
        end

        -- Refill tokens
        local elapsed = now - last_refill
        local tokens_to_add = elapsed * refill_rate
        current_tokens = math.min(capacity, current_tokens + tokens_to_add)

        -- Check if enough tokens
        if current_tokens >= requested_tokens then
            current_tokens = current_tokens - requested_tokens

            -- Update Redis
            redis.call('HSET', key, 'tokens', current_tokens)
            redis.call('HSET', key, 'last_refill', now)
            redis.call('EXPIRE', key, 3600)  -- Expire after 1 hour

            return 1  -- Allowed
        else
            return 0  -- Rejected
        end
        """

        result = self.redis.eval(
            lua_script,
            1,  # Number of keys
            f"token_bucket:{key}",  # KEYS[1]
            self.capacity,          # ARGV[1]
            self.refill_rate,       # ARGV[2]
            tokens,                 # ARGV[3]
            now                     # ARGV[4]
        )

        return result == 1


# Usage:
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)
limiter = DistributedTokenBucket(redis_client, capacity=100, refill_rate=10)

# Check rate limit for user
if limiter.allow_request("user:alice"):
    print("Request allowed")
else:
    print("Rate limit exceeded")
```

---

## 2. Leaky Bucket Algorithm

### How It Works

Imagine a bucket with a hole at the bottom:
1. Requests enter the bucket as water
2. Water **leaks out at a constant rate**
3. If bucket overflows → request is rejected
4. Ensures **steady output rate**

### Visual Representation

```
       Requests (incoming)
             ↓↓↓
       ┌─────────────┐
       │   ░░░░░░░   │ ← Bucket fills up
       │   ░░░░░░░   │
       │   ░░░░░░░   │
       └──────┬──────┘
              │ Constant leak rate
              ↓ (process requests)
```

### Characteristics

**Pros:**
- ✅ **Smooth output** rate (no bursts)
- ✅ Simple conceptually
- ✅ Good for streaming data

**Cons:**
- ❌ Rigid (no burst allowance)
- ❌ Old requests can delay new ones (queue)

### Python Implementation

```python
import time
from collections import deque
from threading import Lock

class LeakyBucket:
    """
    Leaky Bucket Rate Limiter

    Parameters:
        capacity: Maximum queue size
        leak_rate: Requests processed per second
    """

    def __init__(self, capacity: int, leak_rate: float):
        self.capacity = capacity
        self.leak_rate = leak_rate  # requests per second
        self.queue = deque()
        self.last_leak = time.time()
        self.lock = Lock()

    def _leak(self):
        """Process (leak) requests from queue"""
        now = time.time()
        elapsed = now - self.last_leak

        # Calculate how many requests to leak
        requests_to_leak = int(elapsed * self.leak_rate)

        # Remove requests from queue
        for _ in range(min(requests_to_leak, len(self.queue))):
            self.queue.popleft()

        self.last_leak = now

    def allow_request(self) -> bool:
        """Check if request can be added to queue"""
        with self.lock:
            self._leak()

            if len(self.queue) < self.capacity:
                self.queue.append(time.time())
                return True

            return False

    def get_queue_size(self) -> int:
        """Get current queue size"""
        with self.lock:
            self._leak()
            return len(self.queue)


# Usage:
limiter = LeakyBucket(capacity=50, leak_rate=10)  # Process 10 req/s

for i in range(100):
    if limiter.allow_request():
        print(f"Request {i+1}: QUEUED (queue size: {limiter.get_queue_size()})")
    else:
        print(f"Request {i+1}: REJECTED (queue full)")

    time.sleep(0.02)  # 20ms between requests
```

---

## 3. Fixed Window Counter

### How It Works

1. Divide time into **fixed windows** (e.g., 1-minute windows)
2. Count requests in each window
3. If count exceeds limit → reject request
4. Reset counter at window boundary

### Visual Representation

```
Window 1 (0:00-0:59)    Window 2 (1:00-1:59)    Window 3 (2:00-2:59)
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ Count: 10/100   │     │ Count: 95/100   │     │ Count: 5/100    │
│ ▓▓▓▓▓           │     │ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ │     │ ▓▓              │
└─────────────────┘     └─────────────────┘     └─────────────────┘
     ALLOW                   REJECT                   ALLOW
```

**Problem:** Burst at window edges
```
0:59 - 50 requests (allowed)
1:00 - 50 requests (allowed, new window)
Total: 100 requests in 1 second (2x the limit!)
```

### Characteristics

**Pros:**
- ✅ Simple to implement
- ✅ Memory efficient
- ✅ Fast lookups

**Cons:**
- ❌ **Burst problem** at window edges
- ❌ Not accurate for strict limits

### Python Implementation

```python
import time
from collections import defaultdict
from threading import Lock

class FixedWindowCounter:
    """
    Fixed Window Counter Rate Limiter

    Parameters:
        max_requests: Maximum requests per window
        window_size: Window size in seconds
    """

    def __init__(self, max_requests: int, window_size: int):
        self.max_requests = max_requests
        self.window_size = window_size
        self.windows = defaultdict(int)  # window_start -> count
        self.lock = Lock()

    def _get_window_key(self) -> int:
        """Get current window start time"""
        now = int(time.time())
        return (now // self.window_size) * self.window_size

    def allow_request(self) -> bool:
        """Check if request is allowed"""
        with self.lock:
            window_key = self._get_window_key()

            # Clean old windows (optional, for memory)
            self._cleanup_old_windows(window_key)

            if self.windows[window_key] < self.max_requests:
                self.windows[window_key] += 1
                return True

            return False

    def _cleanup_old_windows(self, current_window: int):
        """Remove windows older than current"""
        old_windows = [k for k in self.windows.keys() if k < current_window - self.window_size]
        for k in old_windows:
            del self.windows[k]

    def get_remaining(self) -> int:
        """Get remaining requests in current window"""
        with self.lock:
            window_key = self._get_window_key()
            return self.max_requests - self.windows[window_key]


# Usage:
limiter = FixedWindowCounter(max_requests=100, window_size=60)  # 100 req/min

for i in range(150):
    if limiter.allow_request():
        print(f"Request {i+1}: ALLOWED (remaining: {limiter.get_remaining()})")
    else:
        print(f"Request {i+1}: REJECTED")
```

### Redis Implementation

```python
def fixed_window_redis(redis_client, key: str, max_requests: int, window_size: int) -> bool:
    """
    Fixed window rate limiter using Redis

    Args:
        redis_client: Redis connection
        key: Unique identifier (e.g., "user:alice")
        max_requests: Max requests per window
        window_size: Window size in seconds

    Returns:
        True if request allowed, False otherwise
    """
    now = int(time.time())
    window_key = (now // window_size) * window_size
    redis_key = f"rate_limit:{key}:{window_key}"

    # Increment counter
    count = redis_client.incr(redis_key)

    # Set expiration on first request
    if count == 1:
        redis_client.expire(redis_key, window_size * 2)

    return count <= max_requests
```

---

## 4. Sliding Window Log

### How It Works

1. Store timestamp of each request in a **sorted set**
2. When new request arrives:
   - Remove timestamps older than the window
   - Count remaining timestamps
   - If count < limit → allow request
3. Accurate, no edge burst problem

### Visual Representation

```
Window: Last 60 seconds from now

Timestamps: [10:00:05, 10:00:12, 10:00:45, 10:01:02, 10:01:15]
Current time: 10:01:20

Remove timestamps before 10:00:20:
Remaining: [10:00:45, 10:01:02, 10:01:15]  ← Count: 3

If limit is 100, allow request (3 < 100)
```

### Characteristics

**Pros:**
- ✅ **Most accurate** algorithm
- ✅ No burst at window edges
- ✅ Precise per-second limiting

**Cons:**
- ❌ **Memory intensive** (store all timestamps)
- ❌ Expensive cleanup operations

### Python Implementation

```python
import time
from threading import Lock

class SlidingWindowLog:
    """
    Sliding Window Log Rate Limiter

    Parameters:
        max_requests: Maximum requests per window
        window_size: Window size in seconds
    """

    def __init__(self, max_requests: int, window_size: int):
        self.max_requests = max_requests
        self.window_size = window_size
        self.requests = []  # List of timestamps
        self.lock = Lock()

    def allow_request(self) -> bool:
        """Check if request is allowed"""
        with self.lock:
            now = time.time()
            window_start = now - self.window_size

            # Remove old requests (outside window)
            self.requests = [ts for ts in self.requests if ts > window_start]

            if len(self.requests) < self.max_requests:
                self.requests.append(now)
                return True

            return False

    def get_remaining(self) -> int:
        """Get remaining requests in window"""
        with self.lock:
            now = time.time()
            window_start = now - self.window_size
            self.requests = [ts for ts in self.requests if ts > window_start]
            return self.max_requests - len(self.requests)


# Usage:
limiter = SlidingWindowLog(max_requests=100, window_size=60)

for i in range(150):
    if limiter.allow_request():
        print(f"Request {i+1}: ALLOWED (remaining: {limiter.get_remaining()})")
    else:
        print(f"Request {i+1}: REJECTED")
```

### Redis Implementation (Sorted Set)

```python
def sliding_window_log_redis(redis_client, key: str, max_requests: int, window_size: int) -> bool:
    """
    Sliding window log using Redis sorted set

    Score = timestamp
    """
    now = time.time()
    window_start = now - window_size
    redis_key = f"rate_limit:log:{key}"

    # Remove old entries
    redis_client.zremrangebyscore(redis_key, 0, window_start)

    # Count requests in window
    count = redis_client.zcard(redis_key)

    if count < max_requests:
        # Add current request
        redis_client.zadd(redis_key, {str(now): now})
        redis_client.expire(redis_key, window_size * 2)
        return True

    return False
```

---

## 5. Sliding Window Counter (Production-Ready)

### How It Works

**Hybrid of Fixed Window + Sliding Window Log:**

1. Keep counters for current and previous window
2. Estimate current requests using weighted formula:

```
requests_in_window =
    previous_window_count × overlap_percentage +
    current_window_count
```

### Example Calculation

```
Current time: 10:01:15 (75% through current window)

Previous window (10:00:00-10:00:59): 84 requests
Current window (10:01:00-10:01:59):  36 requests

Overlap with previous window: 25% (15 seconds)

Estimated requests in last 60 seconds:
= 84 × 25% + 36
= 21 + 36
= 57 requests

If limit is 100: ALLOW (57 < 100)
```

### Characteristics

**Pros:**
- ✅ Accurate (better than fixed window)
- ✅ Memory efficient (only 2 counters)
- ✅ No significant edge burst
- ✅ **Production-ready** (best balance)

**Cons:**
- ❌ Slightly complex logic
- ❌ Not perfectly accurate (estimation)

### Python Implementation

```python
import time
from threading import Lock

class SlidingWindowCounter:
    """
    Sliding Window Counter Rate Limiter (Production-Ready)

    Best balance of accuracy and efficiency
    """

    def __init__(self, max_requests: int, window_size: int):
        self.max_requests = max_requests
        self.window_size = window_size
        self.current_window_count = 0
        self.previous_window_count = 0
        self.current_window_start = int(time.time()) // window_size * window_size
        self.lock = Lock()

    def allow_request(self) -> bool:
        """Check if request is allowed"""
        with self.lock:
            now = time.time()
            current_window = int(now) // self.window_size * self.window_size

            # Check if we're in a new window
            if current_window > self.current_window_start:
                self.previous_window_count = self.current_window_count
                self.current_window_count = 0
                self.current_window_start = current_window

            # Calculate position in current window
            elapsed_in_current = now - current_window
            overlap_percentage = 1 - (elapsed_in_current / self.window_size)

            # Estimate requests in sliding window
            estimated_count = (
                self.previous_window_count * overlap_percentage +
                self.current_window_count
            )

            if estimated_count < self.max_requests:
                self.current_window_count += 1
                return True

            return False


# Usage:
limiter = SlidingWindowCounter(max_requests=100, window_size=60)

for i in range(150):
    if limiter.allow_request():
        print(f"Request {i+1}: ALLOWED")
    else:
        print(f"Request {i+1}: REJECTED")
```

### Redis Implementation

```python
def sliding_window_counter_redis(redis_client, key: str, max_requests: int, window_size: int) -> bool:
    """
    Sliding window counter using Redis
    """
    lua_script = """
    local key = KEYS[1]
    local max_requests = tonumber(ARGV[1])
    local window_size = tonumber(ARGV[2])
    local now = tonumber(ARGV[3])

    local current_window = math.floor(now / window_size) * window_size
    local previous_window = current_window - window_size

    local current_key = key .. ":" .. current_window
    local previous_key = key .. ":" .. previous_window

    local current_count = tonumber(redis.call('GET', current_key) or 0)
    local previous_count = tonumber(redis.call('GET', previous_key) or 0)

    -- Calculate overlap percentage
    local elapsed = now - current_window
    local overlap = 1 - (elapsed / window_size)

    -- Estimate requests in sliding window
    local estimated_count = previous_count * overlap + current_count

    if estimated_count < max_requests then
        redis.call('INCR', current_key)
        redis.call('EXPIRE', current_key, window_size * 2)
        return 1
    else
        return 0
    end
    """

    now = time.time()
    result = redis_client.eval(lua_script, 1, f"rate_limit:{key}", max_requests, window_size, now)
    return result == 1
```

---

## Which Algorithm to Choose?

### Decision Matrix

| Scenario | Recommended Algorithm | Reason |
|----------|----------------------|--------|
| **General API rate limiting** | Token Bucket | Allows bursts, industry standard |
| **Strict financial APIs** | Sliding Window Log | Most accurate, no cheating |
| **Production systems** | Sliding Window Counter | Best balance of accuracy/efficiency |
| **Simple rate limiting** | Fixed Window | Easy to implement, good enough |
| **Streaming/video** | Leaky Bucket | Smooth constant rate |

### Real-World Examples

| Company | Algorithm | Use Case |
|---------|-----------|----------|
| **AWS** | Token Bucket | API Gateway throttling |
| **Stripe** | Token Bucket | Payment API rate limiting |
| **GitHub** | Fixed Window | API rate limits (5000/hour) |
| **Shopify** | Leaky Bucket | API request smoothing |
| **CloudFlare** | Sliding Window Counter | DDoS protection |

---

## Advanced Topics

### 1. Distributed Rate Limiting

**Challenge:** Multiple servers need to share rate limit

**Solution:** Use Redis as centralized counter

```python
class DistributedRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client

    def allow_request(self, user_id: str, algorithm: str = "token_bucket") -> bool:
        if algorithm == "token_bucket":
            return self.token_bucket_redis(user_id)
        elif algorithm == "sliding_window":
            return self.sliding_window_redis(user_id)
        # ... other algorithms
```

### 2. Multi-Tier Rate Limiting

**Different limits for different tiers:**

```python
class TieredRateLimiter:
    def __init__(self):
        self.limits = {
            "free": TokenBucket(capacity=100, refill_rate=1),      # 100/sec burst, 1/sec sustained
            "basic": TokenBucket(capacity=500, refill_rate=10),    # 500/sec burst, 10/sec sustained
            "premium": TokenBucket(capacity=10000, refill_rate=100) # 10k/sec burst, 100/sec sustained
        }

    def allow_request(self, user_tier: str) -> bool:
        limiter = self.limits.get(user_tier, self.limits["free"])
        return limiter.allow_request()
```

### 3. Rate Limiting Headers (HTTP)

Standard headers to inform clients:

```python
def add_rate_limit_headers(response, limiter):
    """Add X-RateLimit headers to HTTP response"""
    response.headers['X-RateLimit-Limit'] = str(limiter.capacity)
    response.headers['X-RateLimit-Remaining'] = str(int(limiter.get_tokens()))
    response.headers['X-RateLimit-Reset'] = str(int(time.time()) + 60)
    return response
```

Example response:
```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1705334400
```

If rate limit exceeded:
```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1705334400
Retry-After: 60
```

---

## Testing Rate Limiters

```python
import unittest
import time

class TestTokenBucket(unittest.TestCase):
    def test_allows_requests_under_limit(self):
        limiter = TokenBucket(capacity=10, refill_rate=1)

        # Should allow 10 requests immediately
        for _ in range(10):
            self.assertTrue(limiter.allow_request())

        # 11th request should be rejected
        self.assertFalse(limiter.allow_request())

    def test_refills_over_time(self):
        limiter = TokenBucket(capacity=10, refill_rate=10)  # 10 tokens/sec

        # Use all tokens
        for _ in range(10):
            limiter.allow_request()

        # Wait 1 second
        time.sleep(1)

        # Should have 10 more tokens
        for _ in range(10):
            self.assertTrue(limiter.allow_request())

    def test_does_not_exceed_capacity(self):
        limiter = TokenBucket(capacity=5, refill_rate=10)

        # Wait for refill
        time.sleep(2)  # 20 tokens would be added, but max is 5

        # Should only allow 5 requests
        for i in range(5):
            self.assertTrue(limiter.allow_request(), f"Request {i+1} failed")

        self.assertFalse(limiter.allow_request(), "6th request should fail")


if __name__ == "__main__":
    unittest.main()
```

---

## Summary

**For System Design Interviews:**

1. **Start with Token Bucket** - Most common, industry standard
2. **Mention trade-offs** - Compare with other algorithms
3. **Discuss distributed scenario** - Use Redis for multi-server
4. **Add rate limit headers** - Show HTTP API knowledge
5. **Consider edge cases** - Clock skew, distributed coordination

**Key Takeaway:** Token Bucket is the recommended default, but understand all five algorithms to discuss trade-offs intelligently.

## References

- [AWS API Gateway Throttling](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html) - Uses Token Bucket
- [Stripe Rate Limiting](https://stripe.com/docs/rate-limits) - Token Bucket implementation
- [System Design Interview Vol 1, Chapter 4](https://bytebytego.com/) - Alex Xu's comprehensive guide

---

**Next:** See [Rate Limiter System Design](../../03-system-designs/intermediate/rate-limiter/) for full system design including API Gateway integration.
