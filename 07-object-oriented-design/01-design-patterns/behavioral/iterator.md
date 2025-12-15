# Iterator Pattern

## Intent
Provide a way to access elements of a collection sequentially without exposing its underlying representation.

## Problem
Need to traverse different collections in a uniform way.

## Solution
Create iterator interface with methods to traverse collection.

## Python Implementation

```python
from abc import ABC, abstractmethod

class Iterator(ABC):
    @abstractmethod
    def has_next(self) -> bool:
        pass

    @abstractmethod
    def next(self):
        pass

class BookIterator(Iterator):
    def __init__(self, books: list):
        self._books = books
        self._index = 0

    def has_next(self) -> bool:
        return self._index < len(self._books)

    def next(self):
        if self.has_next():
            book = self._books[self._index]
            self._index += 1
            return book
        raise StopIteration

class BookCollection:
    def __init__(self):
        self._books = []

    def add_book(self, book: str):
        self._books.append(book)

    def create_iterator(self) -> Iterator:
        return BookIterator(self._books)

# Usage
collection = BookCollection()
collection.add_book("Design Patterns")
collection.add_book("Clean Code")
collection.add_book("Refactoring")

iterator = collection.create_iterator()
while iterator.has_next():
    print(iterator.next())

# Python's built-in iterator
class PythonBookCollection:
    def __init__(self):
        self._books = []

    def add_book(self, book: str):
        self._books.append(book)

    def __iter__(self):
        return iter(self._books)

# Usage with Python's iterator protocol
collection2 = PythonBookCollection()
collection2.add_book("Book 1")
collection2.add_book("Book 2")

for book in collection2:  # Uses __iter__
    print(book)
```

## When to Use
- ✅ Access collection elements without exposing structure
- ✅ Support multiple traversals
- ✅ Uniform interface for different collections

## Summary
Iterator provides standard way to traverse collections. Python has built-in support via `__iter__` and `__next__`.
