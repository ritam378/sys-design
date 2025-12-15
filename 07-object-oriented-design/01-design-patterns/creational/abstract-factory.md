# Abstract Factory Pattern

## Intent
Provide an interface for creating families of related or dependent objects without specifying their concrete classes.

## Problem
Need to create families of related objects that work together (e.g., UI components for different platforms: Windows, Mac, Linux).

## Solution
Create abstract factory interface with methods for creating each product in the family. Concrete factories implement the interface to create platform-specific products.

## Python Implementation

```python
from abc import ABC, abstractmethod

# Abstract Products
class Button(ABC):
    @abstractmethod
    def render(self) -> str:
        pass

class Checkbox(ABC):
    @abstractmethod
    def render(self) -> str:
        pass

# Concrete Products - Windows
class WindowsButton(Button):
    def render(self) -> str:
        return "Rendering Windows Button"

class WindowsCheckbox(Checkbox):
    def render(self) -> str:
        return "Rendering Windows Checkbox"

# Concrete Products - Mac
class MacButton(Button):
    def render(self) -> str:
        return "Rendering Mac Button"

class MacCheckbox(Checkbox):
    def render(self) -> str:
        return "Rendering Mac Checkbox"

# Abstract Factory
class GUIFactory(ABC):
    @abstractmethod
    def create_button(self) -> Button:
        pass

    @abstractmethod
    def create_checkbox(self) -> Checkbox:
        pass

# Concrete Factories
class WindowsFactory(GUIFactory):
    def create_button(self) -> Button:
        return WindowsButton()

    def create_checkbox(self) -> Checkbox:
        return WindowsCheckbox()

class MacFactory(GUIFactory):
    def create_button(self) -> Button:
        return MacButton()

    def create_checkbox(self) -> Checkbox:
        return MacCheckbox()

# Client
def create_ui(factory: GUIFactory):
    button = factory.create_button()
    checkbox = factory.create_checkbox()
    return f"{button.render()}, {checkbox.render()}"

# Usage
win_ui = create_ui(WindowsFactory())
mac_ui = create_ui(MacFactory())
print(win_ui)  # Windows components
print(mac_ui)  # Mac components
```

### Real-World Example: Database Connections

```python
# Abstract products
class Connection(ABC):
    @abstractmethod
    def connect(self) -> str:
        pass

class Query(ABC):
    @abstractmethod
    def execute(self, sql: str) -> str:
        pass

# PostgreSQL family
class PostgreSQLConnection(Connection):
    def connect(self) -> str:
        return "Connected to PostgreSQL"

class PostgreSQLQuery(Query):
    def execute(self, sql: str) -> str:
        return f"PostgreSQL executing: {sql}"

# MySQL family
class MySQLConnection(Connection):
    def connect(self) -> str:
        return "Connected to MySQL"

class MySQLQuery(Query):
    def execute(self, sql: str) -> str:
        return f"MySQL executing: {sql}"

# Abstract Factory
class DatabaseFactory(ABC):
    @abstractmethod
    def create_connection(self) -> Connection:
        pass

    @abstractmethod
    def create_query(self) -> Query:
        pass

# Concrete Factories
class PostgreSQLFactory(DatabaseFactory):
    def create_connection(self) -> Connection:
        return PostgreSQLConnection()

    def create_query(self) -> Query:
        return PostgreSQLQuery()

class MySQLFactory(DatabaseFactory):
    def create_connection(self) -> Connection:
        return MySQLConnection()

    def create_query(self) -> Query:
        return MySQLQuery()

# Usage
def database_operations(factory: DatabaseFactory):
    conn = factory.create_connection()
    query = factory.create_query()
    print(conn.connect())
    print(query.execute("SELECT * FROM users"))

database_operations(PostgreSQLFactory())
database_operations(MySQLFactory())
```

## When to Use
- ✅ System needs to work with multiple families of related products
- ✅ Want to enforce that products from same family are used together
- ✅ Want to provide library without exposing implementations

## Advantages
- Ensures compatibility of products
- Follows Open/Closed Principle
- Isolates concrete classes

## Disadvantages
- Complex to implement
- Adding new products requires changing all factories

## Summary
Abstract Factory is ideal when you need to create families of related objects while ensuring they're compatible. Common in cross-platform applications and plugin systems.
