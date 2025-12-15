# Memento Pattern

## Intent
Without violating encapsulation, capture and externalize an object's internal state so that the object can be restored to this state later.

## Problem
Need to save and restore object state without exposing implementation.

## Solution
Create memento object that stores state; originator creates/restores from memento.

## Python Implementation

```python
class EditorMemento:
    """Memento - stores editor state"""
    def __init__(self, content: str, cursor: int):
        self._content = content
        self._cursor = cursor

    def get_content(self) -> str:
        return self._content

    def get_cursor(self) -> int:
        return self._cursor

class TextEditor:
    """Originator - creates and restores from memento"""
    def __init__(self):
        self._content = ""
        self._cursor = 0

    def type(self, text: str):
        self._content += text
        self._cursor += len(text)

    def set_cursor(self, position: int):
        self._cursor = position

    def save(self) -> EditorMemento:
        """Create memento"""
        return EditorMemento(self._content, self._cursor)

    def restore(self, memento: EditorMemento):
        """Restore from memento"""
        self._content = memento.get_content()
        self._cursor = memento.get_cursor()

    def get_content(self) -> str:
        return self._content

class History:
    """Caretaker - manages mementos"""
    def __init__(self):
        self._mementos = []

    def save(self, memento: EditorMemento):
        self._mementos.append(memento)

    def undo(self) -> EditorMemento:
        if self._mementos:
            return self._mementos.pop()
        return None

# Usage
editor = TextEditor()
history = History()

editor.type("Hello ")
history.save(editor.save())

editor.type("World!")
history.save(editor.save())

editor.type(" More text.")
print(f"Current: {editor.get_content()}")  # Hello World! More text.

# Undo
memento = history.undo()
if memento:
    editor.restore(memento)
print(f"After undo: {editor.get_content()}")  # Hello World!

# Undo again
memento = history.undo()
if memento:
    editor.restore(memento)
print(f"After undo: {editor.get_content()}")  # Hello
```

## When to Use
- ✅ Save and restore object state
- ✅ Implement undo/redo
- ✅ Preserve encapsulation

## Summary
Memento captures and restores object state without violating encapsulation. Common in editors, games, transaction systems.
