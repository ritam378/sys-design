# Composite Pattern

## Intent
Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions uniformly.

## Problem
Need to represent tree structures where individual objects and groups should be treated the same way.

## Solution
Create component interface, leaf and composite classes. Composite contains children.

## Python Implementation

```python
from abc import ABC, abstractmethod

class FileSystemComponent(ABC):
    @abstractmethod
    def get_size(self) -> int:
        pass

    @abstractmethod
    def display(self, indent: int = 0) -> str:
        pass

# Leaf
class File(FileSystemComponent):
    def __init__(self, name: str, size: int):
        self.name = name
        self.size = size

    def get_size(self) -> int:
        return self.size

    def display(self, indent: int = 0) -> str:
        return " " * indent + f"File: {self.name} ({self.size} bytes)"

# Composite
class Folder(FileSystemComponent):
    def __init__(self, name: str):
        self.name = name
        self.children = []

    def add(self, component: FileSystemComponent):
        self.children.append(component)

    def remove(self, component: FileSystemComponent):
        self.children.remove(component)

    def get_size(self) -> int:
        return sum(child.get_size() for child in self.children)

    def display(self, indent: int = 0) -> str:
        result = " " * indent + f"Folder: {self.name}\n"
        for child in self.children:
            result += child.display(indent + 2) + "\n"
        return result.rstrip()

# Usage
root = Folder("root")
docs = Folder("documents")
pics = Folder("pictures")

docs.add(File("resume.pdf", 500))
docs.add(File("letter.txt", 100))
pics.add(File("photo1.jpg", 2000))

root.add(docs)
root.add(pics)
root.add(File("readme.txt", 50))

print(root.display())
print(f"Total size: {root.get_size()} bytes")
```

## When to Use
- ✅ Represent part-whole hierarchies
- ✅ Treat individual and composite objects uniformly
- ✅ Tree structures

## Summary
Composite enables treating individual objects and compositions uniformly through a common interface. Perfect for file systems, GUI components, organization charts.
