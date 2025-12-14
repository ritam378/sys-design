# Object-Oriented Design Template

Use this template for solving any OOD interview problem.

## Problem: [System Name]

> **Difficulty:** [Beginner/Intermediate/Advanced]
> **Key Concepts:** [List 3-5 design patterns or concepts]
> **Companies Asked:** [List companies if known]

---

## Step 1: Requirements Gathering (5 min)

### Clarifying Questions

Ask the interviewer:

1. **Core functionality:**
   - What are the main features?
   - What should we prioritize?

2. **Users/Actors:**
   - Who uses the system?
   - What are their roles?

3. **Scale/Constraints:**
   - How many users?
   - Any performance requirements?
   - Single-user or multi-user?

4. **Scope:**
   - What features are out of scope?
   - Should we focus on specific aspects?

### Functional Requirements

List what the system MUST do:

- [ ] Requirement 1
- [ ] Requirement 2
- [ ] Requirement 3
- [ ] Requirement 4

### Non-Functional Requirements

- Performance: [latency, throughput requirements]
- Scalability: [concurrent users, data size]
- Extensibility: [future features to consider]

### Out of Scope

Things we won't implement:

- ❌ Feature 1
- ❌ Feature 2
- ❌ Feature 3

---

## Step 2: Core Objects Identification (5 min)

### From Requirements, Extract:

**Nouns → Potential Classes:**
- [Noun 1] → [ClassName1]
- [Noun 2] → [ClassName2]
- [Noun 3] → [ClassName3]
- [Noun 4] → [ClassName4]

**Verbs → Potential Methods:**
- [Verb 1] → method_name_1()
- [Verb 2] → method_name_2()
- [Verb 3] → method_name_3()

**Relationships:**
- [Class A] HAS-A [Class B] (composition)
- [Class C] IS-A [Class D] (inheritance)
- [Class E] USES [Class F] (association)

---

## Step 3: Class Diagram (10 min)

### High-Level UML

```
┌─────────────────────┐
│   MainClass         │
├─────────────────────┤
│ - field1: Type      │
│ - field2: Type      │
├─────────────────────┤
│ + method1()         │
│ + method2()         │
└──────┬──────────────┘
       │
       │ has-a
       ▼
┌─────────────────────┐
│   ComponentClass    │
├─────────────────────┤
│ - field: Type       │
├─────────────────────┤
│ + method()          │
└─────────────────────┘


      ┌───────────────┐
      │  BaseClass    │◁─────────┐
      └───────────────┘          │
              △                  │ is-a
              │                  │
       ┌──────┴──────┐     ┌────┴──────┐
       │             │     │           │
┌──────▼────┐ ┌─────▼─────┐ ┌────────▼──┐
│ Subclass1 │ │ Subclass2 │ │ Subclass3 │
└───────────┘ └───────────┘ └───────────┘
```

### Core Classes

List 4-6 most important classes with their responsibilities:

1. **ClassName1**
   - Responsibility: [What this class does]
   - Key attributes: [field1, field2]
   - Key methods: [method1(), method2()]

2. **ClassName2**
   - Responsibility: [What this class does]
   - Key attributes: [field1, field2]
   - Key methods: [method1(), method2()]

3. **ClassName3**
   - Responsibility: [What this class does]
   - Key attributes: [field1, field2]
   - Key methods: [method1(), method2()]

---

## Step 4: Implementation (20 min)

### Enums (if applicable)

```python
from enum import Enum

class StatusType(Enum):
    """Type-safe status values"""
    STATUS_1 = 1
    STATUS_2 = 2
    STATUS_3 = 3
```

### Core Class 1

```python
class MainClass:
    """
    Main class description

    Attributes:
        field1: Description of field1
        field2: Description of field2
    """

    def __init__(self, param1: Type, param2: Type):
        """Initialize with parameters"""
        self.field1 = param1
        self.field2 = param2

    def method1(self, param: Type) -> ReturnType:
        """
        Description of what this method does

        Args:
            param: Description of parameter

        Returns:
            Description of return value
        """
        # Implementation
        pass

    def method2(self) -> ReturnType:
        """Description of method2"""
        # Implementation
        pass

    def __str__(self):
        """String representation"""
        return f"MainClass({self.field1}, {self.field2})"
```

### Core Class 2

```python
class ComponentClass:
    """
    Component class description
    """

    def __init__(self, param: Type):
        self.field = param

    def method(self) -> ReturnType:
        """Method description"""
        # Implementation
        pass
```

### Core Class 3 (Inheritance Example)

```python
from abc import ABC, abstractmethod

class BaseClass(ABC):
    """Abstract base class"""

    @abstractmethod
    def abstract_method(self) -> ReturnType:
        """Must be implemented by subclasses"""
        pass

    def common_method(self):
        """Common functionality for all subclasses"""
        # Implementation
        pass


class ConcreteClass1(BaseClass):
    """First concrete implementation"""

    def abstract_method(self) -> ReturnType:
        """Implementation specific to ConcreteClass1"""
        # Implementation
        pass


class ConcreteClass2(BaseClass):
    """Second concrete implementation"""

    def abstract_method(self) -> ReturnType:
        """Implementation specific to ConcreteClass2"""
        # Implementation
        pass
```

---

## Step 5: Usage Example

```python
def main():
    """Demonstrate the system"""

    # Create instances
    obj1 = MainClass(param1, param2)
    component = ComponentClass(param)

    # Demonstrate core functionality
    result = obj1.method1(param)
    print(result)

    # Show relationships
    obj1.component = component

    # Demonstrate polymorphism (if applicable)
    objects = [ConcreteClass1(), ConcreteClass2()]
    for obj in objects:
        obj.abstract_method()


if __name__ == "__main__":
    main()
```

### Expected Output

```
[Show what the program outputs]
```

---

## Design Patterns Used

List design patterns applied:

### Pattern 1: [Pattern Name]

**Where:** [Class/method where applied]

**Why:** [Reason for using this pattern]

**Code snippet:**
```python
# Example of pattern usage
```

### Pattern 2: [Pattern Name]

**Where:** [Class/method where applied]

**Why:** [Reason for using this pattern]

---

## SOLID Principles Applied

### Single Responsibility
- Each class has one clear responsibility
- [Example from your code]

### Open/Closed
- Can extend without modifying existing code
- [Example from your code]

### Liskov Substitution
- Subclasses can replace parent classes
- [Example from your code]

### Interface Segregation
- Focused interfaces, no bloat
- [Example from your code]

### Dependency Inversion
- Depend on abstractions, not concretions
- [Example from your code]

---

## Extensions and Follow-up Questions

### Q1: How would you add [Feature X]?

**Answer:**
```
Explain approach:
1. New classes needed: [ClassName]
2. Modified classes: [ClassName]
3. Design pattern used: [Pattern]
4. Code changes:
```

```python
# Code snippet showing extension
```

### Q2: How would you handle [Edge Case]?

**Answer:**
[Explain handling approach]

### Q3: How would you make it thread-safe?

**Answer:**
```python
import threading

class ThreadSafeClass:
    def __init__(self):
        self.lock = threading.Lock()

    def thread_safe_method(self):
        with self.lock:
            # Critical section
            pass
```

### Q4: How would you persist data to database?

**Answer:**
```python
class Repository:
    """Data access layer (separation of concerns)"""
    def save(self, obj): pass
    def find(self, id): pass
    def delete(self, id): pass
```

---

## Interview Tips

### What Interviewers Look For:

✅ Clear thought process
✅ Well-designed classes with single responsibilities
✅ Appropriate use of OOP concepts
✅ Design patterns where applicable
✅ Code organization and naming
✅ Edge case handling
✅ Extensibility discussion

### Common Mistakes to Avoid:

❌ Jumping to code without design
❌ God classes (doing everything)
❌ No use of abstractions/interfaces
❌ Using strings instead of enums
❌ Not discussing trade-offs
❌ Over-engineering

### Time Management (45 min):

- **5 min:** Requirements and clarifications
- **5 min:** Identify core objects
- **10 min:** Draw class diagram, get feedback
- **20 min:** Implement 3-4 key classes
- **5 min:** Discuss extensions, trade-offs, patterns

---

## Complexity Analysis

### Time Complexity

- Operation 1: O(?)
- Operation 2: O(?)
- Operation 3: O(?)

### Space Complexity

- Storage: O(?)
- Auxiliary space: O(?)

---

## Summary

**Key Concepts Demonstrated:**
- [Concept 1]
- [Concept 2]
- [Concept 3]

**Design Patterns Used:**
- [Pattern 1]
- [Pattern 2]

**SOLID Principles:**
- [Which principles are highlighted]

**What This Problem Tests:**
- Ability to model real-world systems
- Understanding of OOP principles
- Design pattern knowledge
- Code organization skills
- Extensibility thinking

---

## Additional Notes

[Any additional observations, alternative approaches, or interview tips specific to this problem]

---

**Related Problems:**
- [Similar OOD problem 1]
- [Similar OOD problem 2]

**References:**
- [Design pattern documentation]
- [Related articles]
