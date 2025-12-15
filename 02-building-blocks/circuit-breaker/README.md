# Circuit Breaker Pattern

## Overview

The **Circuit Breaker** pattern prevents cascading failures in distributed systems by detecting failures and stopping requests to failing services.

Inspired by electrical circuit breakers that prevent overload.

## Problem Statement

**Without Circuit Breaker:**
```
Service A → Service B (down) → Timeout after 30s
              ↓
            1000 requests waiting
              ↓
         Thread pool exhausted
              ↓
         Service A crashes too!
```

**With Circuit Breaker:**
```
Service A → Circuit Breaker → Service B (down)
                ↓
         Detects failures (5/10)
                ↓
         Opens circuit (fail fast)
                ↓
         Service A stays healthy
```

## States

```
┌─────────────┐
│   CLOSED    │  ← Normal operation, requests pass through
│             │    Track failures
└──────┬──────┘
       │ Threshold exceeded
       ↓
┌─────────────┐
│    OPEN     │  ← Fail fast, reject requests immediately
│             │    Start timeout timer
└──────┬──────┘
       │ After timeout
       ↓
┌─────────────┐
│ HALF_OPEN   │  ← Test if service recovered
│             │    Allow limited requests
└──┬─────┬────┘
   │     │
   │     └──→ Success → CLOSED
   │
   └──→ Failure → OPEN
```

## Implementation

### Basic Circuit Breaker

```python
import time
from enum import Enum
from threading import Lock

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreaker:
    """
    Circuit breaker to prevent cascading failures

    Parameters:
    - failure_threshold: Number of failures before opening circuit
    - success_threshold: Successes needed to close circuit
    - timeout: Seconds before attempting recovery (OPEN → HALF_OPEN)
    """

    def __init__(self, failure_threshold=5, success_threshold=2, timeout=60):
        self.failure_threshold = failure_threshold
        self.success_threshold = success_threshold
        self.timeout = timeout

        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None

        self.lock = Lock()

    def call(self, func, *args, **kwargs):
        """
        Execute function through circuit breaker

        Returns: Function result
        Raises: CircuitBreakerError if circuit is OPEN
        """
        with self.lock:
            if self.state == CircuitState.OPEN:
                if self._should_attempt_reset():
                    self.state = CircuitState.HALF_OPEN
                    print("Circuit HALF_OPEN - testing recovery")
                else:
                    raise CircuitBreakerError("Circuit is OPEN")

        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise e

    def _on_success(self):
        """Handle successful call"""
        with self.lock:
            self.failure_count = 0

            if self.state == CircuitState.HALF_OPEN:
                self.success_count += 1
                if self.success_count >= self.success_threshold:
                    self._close_circuit()

    def _on_failure(self):
        """Handle failed call"""
        with self.lock:
            self.failure_count += 1
            self.last_failure_time = time.time()
            self.success_count = 0

            if self.failure_count >= self.failure_threshold:
                self._open_circuit()

    def _open_circuit(self):
        """Open circuit (fail fast mode)"""
        self.state = CircuitState.OPEN
        print(f"Circuit OPENED after {self.failure_count} failures")

    def _close_circuit(self):
        """Close circuit (normal operation)"""
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        print("Circuit CLOSED - service recovered")

    def _should_attempt_reset(self):
        """Check if enough time passed to try recovery"""
        return (time.time() - self.last_failure_time) >= self.timeout

class CircuitBreakerError(Exception):
    """Raised when circuit is open"""
    pass

# Usage Example
def unreliable_api_call():
    """Simulated API that sometimes fails"""
    import random
    if random.random() < 0.5:  # 50% failure rate
        raise Exception("API Error")
    return "Success"

cb = CircuitBreaker(failure_threshold=3, success_threshold=2, timeout=5)

for i in range(20):
    try:
        result = cb.call(unreliable_api_call)
        print(f"Call {i}: {result}")
    except CircuitBreakerError:
        print(f"Call {i}: Circuit OPEN - failing fast")
    except Exception as e:
        print(f"Call {i}: Error - {e}")

    time.sleep(1)
```

### Decorator Pattern

```python
from functools import wraps

def circuit_breaker(failure_threshold=5, timeout=60):
    """
    Decorator to apply circuit breaker to function

    Usage:
        @circuit_breaker(failure_threshold=3, timeout=30)
        def call_external_api():
            ...
    """
    cb = CircuitBreaker(failure_threshold=failure_threshold, timeout=timeout)

    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            return cb.call(func, *args, **kwargs)
        return wrapper

    return decorator

# Usage
@circuit_breaker(failure_threshold=3, timeout=30)
def fetch_user_data(user_id):
    """Fetch user data from external service"""
    import requests
    response = requests.get(f"https://api.example.com/users/{user_id}")
    response.raise_for_status()
    return response.json()

# Calls are automatically protected by circuit breaker
try:
    user = fetch_user_data(123)
except CircuitBreakerError:
    print("Service unavailable - using cached data")
    user = get_cached_user(123)
```

### Advanced Circuit Breaker with Metrics

```python
import time
from collections import deque
from threading import Lock

class AdvancedCircuitBreaker:
    """
    Circuit breaker with sliding window and metrics

    Features:
    - Sliding window for failure rate calculation
    - Slow call detection
    - Metrics (success rate, avg latency)
    """

    def __init__(
        self,
        failure_threshold=50,  # percentage
        slow_call_threshold=1.0,  # seconds
        window_size=100,
        half_open_max_calls=10,
        timeout=60
    ):
        self.failure_threshold = failure_threshold
        self.slow_call_threshold = slow_call_threshold
        self.window_size = window_size
        self.half_open_max_calls = half_open_max_calls
        self.timeout = timeout

        self.state = CircuitState.CLOSED
        self.calls = deque(maxlen=window_size)  # Sliding window
        self.last_state_change = time.time()
        self.half_open_calls = 0

        self.lock = Lock()

    def call(self, func, *args, **kwargs):
        """Execute function with circuit breaker protection"""
        with self.lock:
            if self.state == CircuitState.OPEN:
                if time.time() - self.last_state_change >= self.timeout:
                    self._transition_to_half_open()
                else:
                    raise CircuitBreakerError("Circuit is OPEN")

            if self.state == CircuitState.HALF_OPEN:
                if self.half_open_calls >= self.half_open_max_calls:
                    raise CircuitBreakerError("Too many HALF_OPEN calls")
                self.half_open_calls += 1

        start_time = time.time()
        try:
            result = func(*args, **kwargs)
            duration = time.time() - start_time
            self._record_success(duration)
            return result
        except Exception as e:
            duration = time.time() - start_time
            self._record_failure(duration)
            raise e

    def _record_success(self, duration):
        """Record successful call"""
        with self.lock:
            is_slow = duration >= self.slow_call_threshold
            self.calls.append({
                'success': True,
                'slow': is_slow,
                'duration': duration,
                'timestamp': time.time()
            })

            if self.state == CircuitState.HALF_OPEN:
                # Enough successes to close circuit?
                recent_success_rate = self._calculate_success_rate()
                if recent_success_rate >= (100 - self.failure_threshold):
                    self._transition_to_closed()

    def _record_failure(self, duration):
        """Record failed call"""
        with self.lock:
            self.calls.append({
                'success': False,
                'slow': False,
                'duration': duration,
                'timestamp': time.time()
            })

            # Check if should open circuit
            if self.state in [CircuitState.CLOSED, CircuitState.HALF_OPEN]:
                failure_rate = 100 - self._calculate_success_rate()
                if failure_rate >= self.failure_threshold:
                    self._transition_to_open()

    def _calculate_success_rate(self):
        """Calculate success rate from sliding window"""
        if not self.calls:
            return 100

        successes = sum(1 for call in self.calls if call['success'])
        return (successes / len(self.calls)) * 100

    def _transition_to_open(self):
        """Transition to OPEN state"""
        self.state = CircuitState.OPEN
        self.last_state_change = time.time()
        print(f"Circuit OPEN - failure rate {100 - self._calculate_success_rate():.1f}%")

    def _transition_to_half_open(self):
        """Transition to HALF_OPEN state"""
        self.state = CircuitState.HALF_OPEN
        self.half_open_calls = 0
        self.last_state_change = time.time()
        print("Circuit HALF_OPEN - testing recovery")

    def _transition_to_closed(self):
        """Transition to CLOSED state"""
        self.state = CircuitState.CLOSED
        self.calls.clear()
        self.last_state_change = time.time()
        print("Circuit CLOSED - service recovered")

    def get_metrics(self):
        """Get circuit breaker metrics"""
        with self.lock:
            if not self.calls:
                return {
                    'state': self.state.value,
                    'success_rate': 100,
                    'avg_latency': 0,
                    'total_calls': 0
                }

            successes = sum(1 for call in self.calls if call['success'])
            slow_calls = sum(1 for call in self.calls if call.get('slow', False))
            avg_latency = sum(call['duration'] for call in self.calls) / len(self.calls)

            return {
                'state': self.state.value,
                'success_rate': (successes / len(self.calls)) * 100,
                'avg_latency': avg_latency,
                'slow_call_rate': (slow_calls / len(self.calls)) * 100,
                'total_calls': len(self.calls)
            }
```

## Fallback Strategies

```python
class CircuitBreakerWithFallback:
    """Circuit breaker with fallback logic"""

    def __init__(self, circuit_breaker, fallback_func):
        self.cb = circuit_breaker
        self.fallback_func = fallback_func

    def call(self, func, *args, **kwargs):
        """Try main function, use fallback if circuit open"""
        try:
            return self.cb.call(func, *args, **kwargs)
        except CircuitBreakerError:
            print("Circuit open - using fallback")
            return self.fallback_func(*args, **kwargs)

# Usage
def get_user_from_api(user_id):
    """Primary: Fetch from API"""
    import requests
    response = requests.get(f"https://api.example.com/users/{user_id}")
    return response.json()

def get_user_from_cache(user_id):
    """Fallback: Get from cache"""
    return {"id": user_id, "name": "Cached User", "cached": True}

cb = CircuitBreaker(failure_threshold=3)
cb_with_fallback = CircuitBreakerWithFallback(cb, get_user_from_cache)

user = cb_with_fallback.call(get_user_from_api, user_id=123)
```

## Integration with Popular Libraries

### Using Pybreaker

```python
from pybreaker import CircuitBreaker

# Create circuit breaker
breaker = CircuitBreaker(
    fail_max=5,           # Open after 5 failures
    timeout_duration=60,  # Stay open for 60 seconds
    name='my_service'
)

@breaker
def call_remote_service():
    """Decorated function automatically protected"""
    import requests
    response = requests.get('https://api.example.com/data')
    return response.json()

# Listen to state changes
def on_circuit_open(cb, *args, **kwargs):
    print(f"Circuit {cb.name} opened!")

def on_circuit_close(cb, *args, **kwargs):
    print(f"Circuit {cb.name} closed!")

breaker.add_listener(on_circuit_open, CircuitBreaker.CIRCUIT_OPENED)
breaker.add_listener(on_circuit_close, CircuitBreaker.CIRCUIT_CLOSED)
```

## Monitoring and Metrics

```python
class CircuitBreakerMonitor:
    """Monitor circuit breaker health"""

    def __init__(self):
        self.breakers = {}

    def register(self, name, circuit_breaker):
        """Register circuit breaker for monitoring"""
        self.breakers[name] = circuit_breaker

    def get_status(self):
        """Get status of all circuit breakers"""
        status = {}
        for name, cb in self.breakers.items():
            metrics = cb.get_metrics() if hasattr(cb, 'get_metrics') else {}
            status[name] = {
                'state': cb.state.value,
                'metrics': metrics
            }
        return status

    def get_health_score(self):
        """Calculate overall health (0-100)"""
        if not self.breakers:
            return 100

        open_circuits = sum(
            1 for cb in self.breakers.values()
            if cb.state == CircuitState.OPEN
        )

        health = ((len(self.breakers) - open_circuits) / len(self.breakers)) * 100
        return health

# Usage
monitor = CircuitBreakerMonitor()
monitor.register('user_service', user_service_cb)
monitor.register('payment_service', payment_service_cb)

# Health check endpoint
@app.route('/health')
def health_check():
    return jsonify({
        'status': monitor.get_status(),
        'health_score': monitor.get_health_score()
    })
```

## Use Cases

### 1. Microservices Communication

```python
class MicroserviceClient:
    """Client for calling microservices with circuit breaker"""

    def __init__(self, service_url):
        self.service_url = service_url
        self.circuit_breaker = CircuitBreaker(
            failure_threshold=5,
            timeout=30
        )

    def get_data(self, endpoint):
        """Call microservice endpoint"""
        def api_call():
            import requests
            response = requests.get(f"{self.service_url}/{endpoint}", timeout=5)
            response.raise_for_status()
            return response.json()

        try:
            return self.circuit_breaker.call(api_call)
        except CircuitBreakerError:
            # Service is down - use cached data or return error
            return {'error': 'Service temporarily unavailable'}

# Usage
user_service = MicroserviceClient('https://user-service.example.com')
payment_service = MicroserviceClient('https://payment-service.example.com')

user_data = user_service.get_data('users/123')
```

### 2. Database Connection Pool

```python
class DatabaseWithCircuitBreaker:
    """Database connection with circuit breaker"""

    def __init__(self, connection_string):
        self.connection_string = connection_string
        self.circuit_breaker = CircuitBreaker(failure_threshold=3, timeout=60)

    def query(self, sql, params=None):
        """Execute query through circuit breaker"""
        def execute():
            conn = psycopg2.connect(self.connection_string)
            cursor = conn.cursor()
            cursor.execute(sql, params)
            result = cursor.fetchall()
            conn.close()
            return result

        return self.circuit_breaker.call(execute)
```

## Interview Tips

### Common Questions

**Q: When to use Circuit Breaker?**
- External service calls (APIs, databases)
- Slow/unreliable dependencies
- Prevent cascading failures

**Q: Circuit Breaker vs Retry?**
- Retry: Transient failures, quick recovery
- Circuit Breaker: Prolonged outages, prevent overload
- Use both: Retry with circuit breaker

**Q: How to choose thresholds?**
- Depends on SLA, traffic patterns
- Start conservative (5 failures, 60s timeout)
- Adjust based on metrics

**Q: What about false positives?**
- Use sliding window (not just consecutive failures)
- Monitor slow calls, not just failures
- Gradual recovery (HALF_OPEN state)

## Key Takeaways

1. **Fail Fast**: Stop calling failing services immediately
2. **Three States**: CLOSED → OPEN → HALF_OPEN → CLOSED
3. **Prevent Cascading**: Protect caller from dependency failures
4. **Fallback**: Provide degraded functionality
5. **Monitor**: Track circuit state and metrics

The Circuit Breaker pattern is essential for building resilient distributed systems!
