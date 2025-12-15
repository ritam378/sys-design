# Facade Pattern

## Intent
Provide a unified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes the subsystem easier to use.

## Problem
Complex subsystem with many interdependent classes is hard to use.

## Solution
Create a facade class that provides simple methods that delegate to subsystem.

## Python Implementation

```python
# Complex subsystem
class CPU:
    def freeze(self): print("CPU freezing...")
    def jump(self, addr): print(f"CPU jumping to {addr}")
    def execute(self): print("CPU executing...")

class Memory:
    def load(self, addr, data): print(f"Loading {data} to {addr}")

class HardDrive:
    def read(self, addr, size): return f"Data from {addr}"

# Facade
class ComputerFacade:
    def __init__(self):
        self.cpu = CPU()
        self.memory = Memory()
        self.hd = HardDrive()

    def start(self):
        """Simple interface to complex boot process"""
        self.cpu.freeze()
        data = self.hd.read(0, 1024)
        self.memory.load(0, data)
        self.cpu.jump(0)
        self.cpu.execute()

# Usage - simple interface
computer = ComputerFacade()
computer.start()  # Hides complexity
```

## When to Use
- ✅ Simplify complex subsystem
- ✅ Layer architecture
- ✅ Reduce coupling between subsystem and clients

## Summary
Facade provides simple interface to complex subsystem, promoting loose coupling and easier maintenance.
