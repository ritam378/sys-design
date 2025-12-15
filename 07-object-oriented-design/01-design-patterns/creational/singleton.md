# Singleton Pattern

## Intent
Ensure a class has only one instance and provide a global point of access to it.

## Problem
- Need exactly one instance of a class (e.g., database connection, logger, config manager)
- Multiple instances would cause problems (conflicting state, resource waste)
- Need global access point

## Solution
- Make constructor private
- Provide static method that returns the single instance
- Lazy or eager initialization

## Structure
```
┌─────────────────────────┐
│      Singleton          │
├─────────────────────────┤
│ - _instance: Singleton  │  (static)
├─────────────────────────┤
│ + get_instance()        │  (static)
│ - __init__()            │  (private)
└─────────────────────────┘
```

## Python Implementation

### Method 1: Using `__new__` (Thread-Safe)
```python
class DatabaseConnection:
    """Singleton database connection"""
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._initialize()
        return cls._instance

    def _initialize(self):
        """Initialize connection (called only once)"""
        self.connection = "Database Connection Established"
        print("Initializing database connection...")

    def query(self, sql: str):
        return f"Executing: {sql}"

# Usage
db1 = DatabaseConnection()
db2 = DatabaseConnection()
assert db1 is db2  # Same instance
print(f"Same instance: {db1 is db2}")  # True
```

### Method 2: Metaclass Approach
```python
class SingletonMeta(type):
    """Metaclass for creating singletons"""
    _instances = {}

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Logger(metaclass=SingletonMeta):
    """Singleton logger"""
    def __init__(self):
        self.logs = []

    def log(self, message: str):
        self.logs.append(message)
        print(f"[LOG] {message}")

# Usage
logger1 = Logger()
logger2 = Logger()
logger1.log("First message")
logger2.log("Second message")
assert logger1 is logger2
print(logger1.logs)  # ['First message', 'Second message']
```

### Method 3: Decorator Approach
```python
def singleton(cls):
    """Singleton decorator"""
    instances = {}

    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return get_instance

@singleton
class ConfigManager:
    """Singleton configuration manager"""
    def __init__(self):
        self.config = {"debug": False, "timeout": 30}

    def get(self, key: str):
        return self.config.get(key)

    def set(self, key: str, value):
        self.config[key] = value

# Usage
config1 = ConfigManager()
config2 = ConfigManager()
config1.set("debug", True)
assert config2.get("debug") == True  # Shared state
```

### Method 4: Thread-Safe Singleton with Lock
```python
import threading

class ThreadSafeSingleton:
    """Thread-safe singleton using double-checked locking"""
    _instance = None
    _lock = threading.Lock()

    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                # Double-check after acquiring lock
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance

# Usage
def create_singleton():
    singleton = ThreadSafeSingleton()
    print(f"Thread {threading.current_thread().name}: {id(singleton)}")

threads = [threading.Thread(target=create_singleton) for _ in range(5)]
for t in threads:
    t.start()
for t in threads:
    t.join()
# All threads get same instance ID
```

## When to Use
- ✅ Exactly one instance needed (database connection, thread pool)
- ✅ Global access point required
- ✅ Lazy initialization beneficial
- ✅ Control over instantiation needed

## When NOT to Use
- ❌ Need multiple instances
- ❌ Testing requires fresh instances
- ❌ Violates Single Responsibility Principle (manages own lifecycle)
- ❌ Makes code harder to test (global state)

## Advantages
- **Controlled access:** Single instance guaranteed
- **Lazy initialization:** Created only when needed
- **Global access:** Available throughout application
- **Memory efficient:** One instance only

## Disadvantages
- **Global state:** Can cause hidden dependencies
- **Testing difficulties:** Hard to mock/isolate
- **Violates SRP:** Class manages both logic and lifecycle
- **Concurrency issues:** Requires thread-safety measures
- **Tight coupling:** Clients depend on concrete class

## Real-World Examples

### 1. Database Connection Pool
```python
class ConnectionPool:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.connections = []
            cls._instance._initialize_pool()
        return cls._instance

    def _initialize_pool(self, size=5):
        for i in range(size):
            self.connections.append(f"Connection-{i}")

    def get_connection(self):
        if self.connections:
            return self.connections.pop()
        raise Exception("No connections available")

    def release_connection(self, conn):
        self.connections.append(conn)

# Usage
pool = ConnectionPool()
conn = pool.get_connection()
print(f"Got: {conn}")
pool.release_connection(conn)
```

### 2. Application Settings
```python
class Settings:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._load_settings()
        return cls._instance

    def _load_settings(self):
        self.settings = {
            "app_name": "MyApp",
            "version": "1.0.0",
            "debug_mode": False
        }

    def get(self, key: str, default=None):
        return self.settings.get(key, default)

    def set(self, key: str, value):
        self.settings[key] = value

# Usage across modules
settings = Settings()
settings.set("debug_mode", True)

# In another module
settings2 = Settings()
print(settings2.get("debug_mode"))  # True (same instance)
```

### 3. Logger System
```python
from datetime import datetime

class ApplicationLogger:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.log_file = "app.log"
        return cls._instance

    def info(self, message: str):
        self._write_log("INFO", message)

    def error(self, message: str):
        self._write_log("ERROR", message)

    def _write_log(self, level: str, message: str):
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        log_entry = f"[{timestamp}] {level}: {message}"
        print(log_entry)

# Usage
logger = ApplicationLogger()
logger.info("Application started")
logger.error("An error occurred")

# Same logger everywhere
another_logger = ApplicationLogger()
assert logger is another_logger
```

## Alternatives to Singleton

### Dependency Injection
```python
class Database:
    def __init__(self, connection_string: str):
        self.connection_string = connection_string

class UserService:
    def __init__(self, database: Database):
        self.database = database  # Injected, not singleton

# Usage
db = Database("postgresql://localhost")
user_service = UserService(db)  # Explicit dependency
```

### Module-Level Instance
```python
# config.py
class Config:
    def __init__(self):
        self.settings = {}

# Create single instance at module level
config = Config()

# other_module.py
from config import config  # Import the instance
config.settings["key"] = "value"
```

## Related Patterns
- **Factory Method:** Can use singleton to ensure single factory
- **Facade:** Often implemented as singleton
- **Flyweight:** Often uses singleton for shared instances

## Interview Questions

**Q: How do you make a singleton thread-safe?**
A: Use locks (double-checked locking), metaclasses, or `__new__` method.

**Q: What are the disadvantages of Singleton?**
A: Global state, testing difficulties, violates SRP, tight coupling.

**Q: How would you test code that uses a singleton?**
A: Use dependency injection, create reset method, or use mocking frameworks.

**Q: Can you have multiple singletons in inheritance hierarchy?**
A: Yes, but each class should maintain its own instance if truly separate.

## Summary

**Use Singleton when:**
- Need exactly one instance (database, logger, config)
- Global access required
- Controlling instantiation is important

**Avoid Singleton when:**
- Testing is priority (use DI instead)
- Multiple instances might be needed later
- Causes tight coupling issues

**Remember:** Singleton is often considered an anti-pattern. Consider alternatives like dependency injection for better testability and flexibility.
