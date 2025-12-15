# Flyweight Pattern

## Intent
Use sharing to support large numbers of fine-grained objects efficiently.

## Problem
Need many similar objects, causing high memory usage.

## Solution
Share common state (intrinsic) among objects, store unique state (extrinsic) separately.

## Python Implementation

```python
class TreeType:
    """Flyweight - shared intrinsic state"""
    def __init__(self, name: str, color: str, texture: str):
        self.name = name
        self.color = color
        self.texture = texture

    def render(self, x: int, y: int):
        print(f"Drawing {self.color} {self.name} at ({x}, {y})")

class TreeFactory:
    """Manages flyweight pool"""
    _tree_types = {}

    @classmethod
    def get_tree_type(cls, name: str, color: str, texture: str) -> TreeType:
        key = f"{name}_{color}_{texture}"
        if key not in cls._tree_types:
            cls._tree_types[key] = TreeType(name, color, texture)
            print(f"Creating new TreeType: {key}")
        return cls._tree_types[key]

    @classmethod
    def get_total_types(cls) -> int:
        return len(cls._tree_types)

class Tree:
    """Context - stores extrinsic state"""
    def __init__(self, x: int, y: int, tree_type: TreeType):
        self.x = x  # Unique state
        self.y = y  # Unique state
        self.type = tree_type  # Shared state

    def render(self):
        self.type.render(self.x, self.y)

# Usage - 1000 trees with only a few shared types
forest = []
for i in range(1000):
    if i % 3 == 0:
        tree_type = TreeFactory.get_tree_type("Oak", "green", "rough")
    elif i % 3 == 1:
        tree_type = TreeFactory.get_tree_type("Pine", "dark green", "smooth")
    else:
        tree_type = TreeFactory.get_tree_type("Birch", "white", "papery")

    tree = Tree(i * 10, i * 10, tree_type)
    forest.append(tree)

print(f"Total trees: {len(forest)}")
print(f"Total tree types (flyweights): {TreeFactory.get_total_types()}")  # Only 3!
```

## When to Use
- ✅ Large number of similar objects
- ✅ Most object state can be extrinsic
- ✅ Memory usage is concern

## Summary
Flyweight reduces memory by sharing common state among many objects. Common in games, text editors, graphics systems.
