# Stack and Queue - OOD Design

**Difficulty:** Beginner
**Interview Frequency:** High
**Key Concepts:** Encapsulation, Iterator Pattern, Template Method Pattern, Generics
**Companies:** Google, Amazon, Microsoft, Facebook, Apple

---

## Problem Statement

Design custom Stack and Queue data structures from scratch using object-oriented principles. Implement core operations, handle edge cases, and demonstrate different implementation approaches (array-based and linked-list-based).

**Core Features:**
1. Stack with push, pop, peek, isEmpty operations
2. Queue with enqueue, dequeue, peek, isEmpty operations
3. Handle capacity limits and empty structure edge cases
4. Support both array-based and linked-list implementations
5. Iterator support for traversal
6. Generic type support

---

## 1. Requirements Gathering

### Functional Requirements

- ✅ **Stack Operations:** push, pop, peek, isEmpty, size
- ✅ **Queue Operations:** enqueue, dequeue, peek, isEmpty, size
- ✅ **Capacity Management:** Support both fixed and dynamic sizing
- ✅ **Error Handling:** Handle overflow and underflow gracefully
- ✅ **Type Safety:** Generic support for different data types
- ✅ **Iterator Support:** Enable iteration through elements

### Non-Functional Requirements

- O(1) time complexity for all operations
- Clear error messages for invalid operations
- Memory efficient implementations
- Thread-safe operations (optional extension)

### Out of Scope

- ❌ Persistence to disk
- ❌ Distributed implementation
- ❌ Priority queue
- ❌ Double-ended queue (deque)

---

## 2. Core Objects Identification

### Classes

- **Stack** - LIFO data structure (abstract base)
- **ArrayStack** - Array-based stack implementation
- **LinkedStack** - Linked-list-based stack implementation
- **Queue** - FIFO data structure (abstract base)
- **ArrayQueue** - Circular array-based queue implementation
- **LinkedQueue** - Linked-list-based queue implementation
- **Node** - Building block for linked implementations
- **StackIterator** - Iterator for stack traversal

### Methods

- Stack: `push()`, `pop()`, `peek()`, `is_empty()`, `size()`
- Queue: `enqueue()`, `dequeue()`, `peek()`, `is_empty()`, `size()`
- Shared: `clear()`, `__str__()`, `__iter__()`

---

## 3. Class Diagram

```
          ┌─────────────────────┐
          │   <<abstract>>      │
          │       Stack         │
          ├─────────────────────┤
          │ + push(item)        │
          │ + pop(): T          │
          │ + peek(): T         │
          │ + is_empty(): bool  │
          │ + size(): int       │
          └─────────────────────┘
                    △
                    │
          ┌─────────┴─────────┐
          │                   │
  ┌───────┴────────┐  ┌───────┴────────┐
  │  ArrayStack    │  │  LinkedStack   │
  ├────────────────┤  ├────────────────┤
  │ - items: List  │  │ - top: Node    │
  │ - capacity: int│  │ - count: int   │
  └────────────────┘  └────────────────┘
                              │
                              │ uses
                              ▼
                      ┌───────────────┐
                      │     Node      │
                      ├───────────────┤
                      │ - data: T     │
                      │ - next: Node  │
                      └───────────────┘


          ┌─────────────────────┐
          │   <<abstract>>      │
          │       Queue         │
          ├─────────────────────┤
          │ + enqueue(item)     │
          │ + dequeue(): T      │
          │ + peek(): T         │
          │ + is_empty(): bool  │
          │ + size(): int       │
          └─────────────────────┘
                    △
                    │
          ┌─────────┴─────────┐
          │                   │
  ┌───────┴────────┐  ┌───────┴────────┐
  │  ArrayQueue    │  │  LinkedQueue   │
  ├────────────────┤  ├────────────────┤
  │ - items: List  │  │ - front: Node  │
  │ - front: int   │  │ - rear: Node   │
  │ - rear: int    │  │ - count: int   │
  └────────────────┘  └────────────────┘
```

---

## 4. Python Implementation

### Step 1: Node Class for Linked Implementations

```python
from typing import Generic, TypeVar, Optional, List
from abc import ABC, abstractmethod

T = TypeVar('T')


class Node(Generic[T]):
    """Node for linked list implementation"""

    def __init__(self, data: T, next_node: Optional['Node[T]'] = None):
        self.data = data
        self.next = next_node

    def __str__(self) -> str:
        return str(self.data)
```

### Step 2: Abstract Stack Class

```python
class Stack(ABC, Generic[T]):
    """Abstract base class for Stack implementations"""

    @abstractmethod
    def push(self, item: T) -> None:
        """Add item to top of stack"""
        pass

    @abstractmethod
    def pop(self) -> T:
        """Remove and return top item"""
        pass

    @abstractmethod
    def peek(self) -> T:
        """Return top item without removing"""
        pass

    @abstractmethod
    def is_empty(self) -> bool:
        """Check if stack is empty"""
        pass

    @abstractmethod
    def size(self) -> int:
        """Return number of items"""
        pass

    @abstractmethod
    def clear(self) -> None:
        """Remove all items"""
        pass
```

### Step 3: Array-Based Stack

```python
class ArrayStack(Stack[T]):
    """
    Array-based stack implementation.
    Uses Python list with optional capacity limit.
    """

    def __init__(self, capacity: Optional[int] = None):
        self._items: List[T] = []
        self._capacity = capacity

    def push(self, item: T) -> None:
        """Push item onto stack"""
        if self._capacity and len(self._items) >= self._capacity:
            raise OverflowError(f"Stack overflow: capacity {self._capacity} reached")
        self._items.append(item)

    def pop(self) -> T:
        """Pop item from stack"""
        if self.is_empty():
            raise IndexError("Stack underflow: cannot pop from empty stack")
        return self._items.pop()

    def peek(self) -> T:
        """Peek at top item"""
        if self.is_empty():
            raise IndexError("Stack is empty")
        return self._items[-1]

    def is_empty(self) -> bool:
        """Check if empty"""
        return len(self._items) == 0

    def size(self) -> int:
        """Get stack size"""
        return len(self._items)

    def clear(self) -> None:
        """Clear all items"""
        self._items.clear()

    def __iter__(self):
        """Iterate from top to bottom"""
        return iter(reversed(self._items))

    def __str__(self) -> str:
        items_str = " -> ".join(str(item) for item in reversed(self._items))
        return f"Stack[top -> {items_str} -> bottom]"
```

### Step 4: Linked-List-Based Stack

```python
class LinkedStack(Stack[T]):
    """
    Linked-list-based stack implementation.
    No capacity limit, more memory overhead per element.
    """

    def __init__(self):
        self._top: Optional[Node[T]] = None
        self._count = 0

    def push(self, item: T) -> None:
        """Push item onto stack"""
        new_node = Node(item, self._top)
        self._top = new_node
        self._count += 1

    def pop(self) -> T:
        """Pop item from stack"""
        if self.is_empty():
            raise IndexError("Stack underflow: cannot pop from empty stack")

        data = self._top.data
        self._top = self._top.next
        self._count -= 1
        return data

    def peek(self) -> T:
        """Peek at top item"""
        if self.is_empty():
            raise IndexError("Stack is empty")
        return self._top.data

    def is_empty(self) -> bool:
        """Check if empty"""
        return self._top is None

    def size(self) -> int:
        """Get stack size"""
        return self._count

    def clear(self) -> None:
        """Clear all items"""
        self._top = None
        self._count = 0

    def __iter__(self):
        """Iterate from top to bottom"""
        current = self._top
        while current:
            yield current.data
            current = current.next

    def __str__(self) -> str:
        items = [str(item) for item in self]
        items_str = " -> ".join(items)
        return f"Stack[top -> {items_str} -> bottom]"
```

### Step 5: Abstract Queue Class

```python
class Queue(ABC, Generic[T]):
    """Abstract base class for Queue implementations"""

    @abstractmethod
    def enqueue(self, item: T) -> None:
        """Add item to rear of queue"""
        pass

    @abstractmethod
    def dequeue(self) -> T:
        """Remove and return front item"""
        pass

    @abstractmethod
    def peek(self) -> T:
        """Return front item without removing"""
        pass

    @abstractmethod
    def is_empty(self) -> bool:
        """Check if queue is empty"""
        pass

    @abstractmethod
    def size(self) -> int:
        """Return number of items"""
        pass

    @abstractmethod
    def clear(self) -> None:
        """Remove all items"""
        pass
```

### Step 6: Circular Array-Based Queue

```python
class ArrayQueue(Queue[T]):
    """
    Circular array-based queue implementation.
    Efficient use of space with wrap-around.
    """

    def __init__(self, capacity: int = 10):
        self._items: List[Optional[T]] = [None] * capacity
        self._capacity = capacity
        self._front = 0
        self._rear = 0
        self._count = 0

    def enqueue(self, item: T) -> None:
        """Enqueue item"""
        if self._count >= self._capacity:
            self._resize(self._capacity * 2)

        self._items[self._rear] = item
        self._rear = (self._rear + 1) % self._capacity
        self._count += 1

    def dequeue(self) -> T:
        """Dequeue item"""
        if self.is_empty():
            raise IndexError("Queue underflow: cannot dequeue from empty queue")

        item = self._items[self._front]
        self._items[self._front] = None  # Help garbage collection
        self._front = (self._front + 1) % self._capacity
        self._count -= 1

        # Shrink if necessary
        if 0 < self._count < self._capacity // 4:
            self._resize(self._capacity // 2)

        return item

    def peek(self) -> T:
        """Peek at front item"""
        if self.is_empty():
            raise IndexError("Queue is empty")
        return self._items[self._front]

    def is_empty(self) -> bool:
        """Check if empty"""
        return self._count == 0

    def size(self) -> int:
        """Get queue size"""
        return self._count

    def clear(self) -> None:
        """Clear all items"""
        self._items = [None] * self._capacity
        self._front = 0
        self._rear = 0
        self._count = 0

    def _resize(self, new_capacity: int) -> None:
        """Resize internal array"""
        new_items: List[Optional[T]] = [None] * new_capacity

        # Copy items in order
        for i in range(self._count):
            new_items[i] = self._items[(self._front + i) % self._capacity]

        self._items = new_items
        self._capacity = new_capacity
        self._front = 0
        self._rear = self._count

    def __iter__(self):
        """Iterate from front to rear"""
        for i in range(self._count):
            yield self._items[(self._front + i) % self._capacity]

    def __str__(self) -> str:
        items_str = " <- ".join(str(item) for item in self)
        return f"Queue[front -> {items_str} <- rear]"
```

### Step 7: Linked-List-Based Queue

```python
class LinkedQueue(Queue[T]):
    """
    Linked-list-based queue implementation.
    No capacity limit, O(1) operations.
    """

    def __init__(self):
        self._front: Optional[Node[T]] = None
        self._rear: Optional[Node[T]] = None
        self._count = 0

    def enqueue(self, item: T) -> None:
        """Enqueue item"""
        new_node = Node(item)

        if self.is_empty():
            self._front = new_node
            self._rear = new_node
        else:
            self._rear.next = new_node
            self._rear = new_node

        self._count += 1

    def dequeue(self) -> T:
        """Dequeue item"""
        if self.is_empty():
            raise IndexError("Queue underflow: cannot dequeue from empty queue")

        data = self._front.data
        self._front = self._front.next

        if self._front is None:  # Queue became empty
            self._rear = None

        self._count -= 1
        return data

    def peek(self) -> T:
        """Peek at front item"""
        if self.is_empty():
            raise IndexError("Queue is empty")
        return self._front.data

    def is_empty(self) -> bool:
        """Check if empty"""
        return self._front is None

    def size(self) -> int:
        """Get queue size"""
        return self._count

    def clear(self) -> None:
        """Clear all items"""
        self._front = None
        self._rear = None
        self._count = 0

    def __iter__(self):
        """Iterate from front to rear"""
        current = self._front
        while current:
            yield current.data
            current = current.next

    def __str__(self) -> str:
        items = [str(item) for item in self]
        items_str = " <- ".join(items)
        return f"Queue[front -> {items_str} <- rear]"
```

---

## 5. Complete Usage Example

```python
def test_stack():
    """Test stack implementations"""
    print("=" * 60)
    print("TESTING ARRAY STACK")
    print("=" * 60)

    stack = ArrayStack[int](capacity=5)

    # Push items
    for i in range(1, 6):
        stack.push(i * 10)
        print(f"Pushed {i * 10}: {stack}")

    # Peek
    print(f"\nPeek: {stack.peek()}")

    # Pop items
    print("\nPopping items:")
    while not stack.is_empty():
        item = stack.pop()
        print(f"Popped {item}: {stack}")

    # Test overflow
    print("\nTesting overflow:")
    try:
        limited_stack = ArrayStack[str](capacity=2)
        limited_stack.push("A")
        limited_stack.push("B")
        limited_stack.push("C")  # Should fail
    except OverflowError as e:
        print(f"✓ Caught overflow: {e}")

    # Test underflow
    print("\nTesting underflow:")
    try:
        empty_stack = ArrayStack[int]()
        empty_stack.pop()  # Should fail
    except IndexError as e:
        print(f"✓ Caught underflow: {e}")

    print("\n" + "=" * 60)
    print("TESTING LINKED STACK")
    print("=" * 60)

    linked_stack = LinkedStack[str]()

    for letter in "HELLO":
        linked_stack.push(letter)
        print(f"Pushed {letter}: {linked_stack}")

    print(f"\nSize: {linked_stack.size()}")

    # Iteration
    print("\nIterating through stack:")
    for item in linked_stack:
        print(f"  {item}")


def test_queue():
    """Test queue implementations"""
    print("\n" + "=" * 60)
    print("TESTING ARRAY QUEUE")
    print("=" * 60)

    queue = ArrayQueue[int](capacity=3)

    # Enqueue items
    for i in range(1, 6):
        queue.enqueue(i * 10)
        print(f"Enqueued {i * 10}: {queue}")

    # Peek
    print(f"\nPeek: {queue.peek()}")

    # Dequeue items
    print("\nDequeuing items:")
    for _ in range(3):
        item = queue.dequeue()
        print(f"Dequeued {item}: {queue}")

    # Test circular behavior
    print("\nTesting circular behavior:")
    queue.enqueue(100)
    queue.enqueue(200)
    print(f"After enqueues: {queue}")

    print("\n" + "=" * 60)
    print("TESTING LINKED QUEUE")
    print("=" * 60)

    linked_queue = LinkedQueue[str]()

    for word in ["First", "Second", "Third", "Fourth"]:
        linked_queue.enqueue(word)
        print(f"Enqueued {word}: {linked_queue}")

    print(f"\nSize: {linked_queue.size()}")

    # Iteration
    print("\nIterating through queue:")
    for item in linked_queue:
        print(f"  {item}")

    # Clear queue
    linked_queue.clear()
    print(f"\nAfter clear: is_empty = {linked_queue.is_empty()}")


def main():
    """Run all tests"""
    test_stack()
    test_queue()


if __name__ == "__main__":
    main()
```

---

## 6. Design Patterns Used

### Template Method Pattern

**Where:** Abstract base classes define algorithm structure

**Code:**
```python
class Stack(ABC):
    # Template methods define the interface
    @abstractmethod
    def push(self, item: T) -> None:
        pass  # Subclasses implement specific behavior
```

**Why:** Allows different implementations while maintaining consistent interface.

### Iterator Pattern

**Where:** `__iter__()` methods in all implementations

**Code:**
```python
for item in stack:  # Works seamlessly
    print(item)
```

**Why:** Enables uniform traversal regardless of internal structure.

### Strategy Pattern

**Where:** Choosing between array-based and linked implementations

**Why:** Different algorithms (array vs linked) with same interface.

---

## 7. SOLID Principles Applied

### Single Responsibility Principle (SRP)
- `Node`: Only manages node data and link
- `ArrayStack`: Only manages array-based stack logic
- `LinkedStack`: Only manages linked-list-based stack logic

### Open/Closed Principle (OCP)
- Easy to add new implementations (e.g., `ThreadSafeStack`) without modifying base class

### Liskov Substitution Principle (LSP)
- All Stack implementations can substitute `Stack` base class
- All Queue implementations can substitute `Queue` base class

### Interface Segregation Principle (ISP)
- Stack and Queue have minimal, focused interfaces
- No unnecessary methods

### Dependency Inversion Principle (DIP)
- Client code depends on `Stack`/`Queue` abstractions, not concrete classes

---

## 8. Extensions and Follow-up Questions

### Q1: How would you make these thread-safe?

```python
import threading

class ThreadSafeStack(Stack[T]):
    def __init__(self, implementation: Stack[T]):
        self._stack = implementation
        self._lock = threading.Lock()

    def push(self, item: T) -> None:
        with self._lock:
            self._stack.push(item)

    def pop(self) -> T:
        with self._lock:
            return self._stack.pop()

    # ... other methods with locks
```

### Q2: How would you implement a min/max stack?

```python
class MinStack(ArrayStack[int]):
    def __init__(self):
        super().__init__()
        self._min_stack = ArrayStack[int]()

    def push(self, item: int) -> None:
        super().push(item)
        if self._min_stack.is_empty() or item <= self._min_stack.peek():
            self._min_stack.push(item)

    def pop(self) -> int:
        item = super().pop()
        if item == self._min_stack.peek():
            self._min_stack.pop()
        return item

    def get_min(self) -> int:
        return self._min_stack.peek()
```

### Q3: Implement a queue using two stacks

```python
class StackQueue(Queue[T]):
    def __init__(self):
        self._inbox = ArrayStack[T]()
        self._outbox = ArrayStack[T]()

    def enqueue(self, item: T) -> None:
        self._inbox.push(item)

    def dequeue(self) -> T:
        if self._outbox.is_empty():
            while not self._inbox.is_empty():
                self._outbox.push(self._inbox.pop())
        return self._outbox.pop()

    # ... other methods
```

---

## 9. Interview Tips

### What Interviewers Look For

- ✅ Understanding of LIFO vs FIFO
- ✅ Proper error handling (overflow/underflow)
- ✅ Generic type support
- ✅ O(1) time complexity awareness
- ✅ Iterator implementation
- ✅ Clean abstraction with ABC

### Common Mistakes

- ❌ Not handling empty structure edge cases
- ❌ Inefficient queue implementation (not using circular array)
- ❌ Missing capacity checks
- ❌ Not using generics/type hints
- ❌ Incorrect linked list pointer management

### Time Management (45 min)

- **0-5 min:** Clarify requirements (LIFO/FIFO, capacity limits)
- **5-10 min:** Design class hierarchy
- **10-25 min:** Implement one stack and one queue variant
- **25-35 min:** Add error handling and edge cases
- **35-45 min:** Discuss patterns, extensions, trade-offs

---

## 10. Summary

### Key Takeaways

1. **Abstract base classes** define consistent interfaces
2. **Array vs Linked** trade-offs: memory, flexibility, performance
3. **Circular arrays** optimize queue space usage
4. **Generic types** enable type-safe, reusable code
5. **Iterator pattern** provides uniform traversal
6. **Error handling** prevents invalid states

### Complexity Analysis

| Operation | Array Stack | Linked Stack | Array Queue | Linked Queue |
|-----------|-------------|--------------|-------------|--------------|
| Push/Enqueue | O(1)* | O(1) | O(1)* | O(1) |
| Pop/Dequeue | O(1) | O(1) | O(1)* | O(1) |
| Peek | O(1) | O(1) | O(1) | O(1) |
| Space | O(n) | O(n) | O(n) | O(n) |

*Amortized O(1) due to occasional resize

### Related Problems

- **Min/Max Stack:** Track minimum/maximum efficiently
- **Implement Queue using Stacks:** Two-stack technique
- **Valid Parentheses:** Stack application
- **LRU Cache:** Combined hash map and queue

This problem demonstrates fundamental data structure design!
