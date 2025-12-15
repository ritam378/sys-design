# Builder Pattern

## Intent
Separate the construction of a complex object from its representation, allowing the same construction process to create different representations.

## Problem
- Object has many optional parameters (telescoping constructor problem)
- Construction requires multiple steps
- Want immutable objects with many fields

## Solution
Use a builder class to construct the object step-by-step with a fluent interface (method chaining).

## Python Implementation

```python
class Pizza:
    def __init__(self):
        self.size: str = ""
        self.crust: str = ""
        self.toppings: list = []
        self.cheese: bool = False
        self.sauce: str = ""

    def __str__(self) -> str:
        return (f"{self.size} pizza with {self.crust} crust, "
                f"{self.sauce} sauce, toppings: {', '.join(self.toppings)}, "
                f"cheese: {self.cheese}")

class PizzaBuilder:
    def __init__(self):
        self._pizza = Pizza()

    def set_size(self, size: str) -> 'PizzaBuilder':
        self._pizza.size = size
        return self

    def set_crust(self, crust: str) -> 'PizzaBuilder':
        self._pizza.crust = crust
        return self

    def set_sauce(self, sauce: str) -> 'PizzaBuilder':
        self._pizza.sauce = sauce
        return self

    def add_topping(self, topping: str) -> 'PizzaBuilder':
        self._pizza.toppings.append(topping)
        return self

    def add_cheese(self) -> 'PizzaBuilder':
        self._pizza.cheese = True
        return self

    def build(self) -> Pizza:
        return self._pizza

# Usage - method chaining
pizza = (PizzaBuilder()
         .set_size("Large")
         .set_crust("Thin")
         .set_sauce("Marinara")
         .add_topping("Pepperoni")
         .add_topping("Mushrooms")
         .add_cheese()
         .build())

print(pizza)
```

### Example 2: SQL Query Builder

```python
class SQLQuery:
    def __init__(self):
        self.select_fields = []
        self.table = ""
        self.where_conditions = []
        self.order_by = []
        self.limit_value = None

    def to_sql(self) -> str:
        sql = f"SELECT {', '.join(self.select_fields)} FROM {self.table}"
        if self.where_conditions:
            sql += f" WHERE {' AND '.join(self.where_conditions)}"
        if self.order_by:
            sql += f" ORDER BY {', '.join(self.order_by)}"
        if self.limit_value:
            sql += f" LIMIT {self.limit_value}"
        return sql

class QueryBuilder:
    def __init__(self):
        self._query = SQLQuery()

    def select(self, *fields: str) -> 'QueryBuilder':
        self._query.select_fields.extend(fields)
        return self

    def from_table(self, table: str) -> 'QueryBuilder':
        self._query.table = table
        return self

    def where(self, condition: str) -> 'QueryBuilder':
        self._query.where_conditions.append(condition)
        return self

    def order_by(self, field: str) -> 'QueryBuilder':
        self._query.order_by.append(field)
        return self

    def limit(self, count: int) -> 'QueryBuilder':
        self._query.limit_value = count
        return self

    def build(self) -> SQLQuery:
        return self._query

# Usage
query = (QueryBuilder()
         .select("id", "name", "email")
         .from_table("users")
         .where("age > 18")
         .where("active = true")
         .order_by("name")
         .limit(10)
         .build())

print(query.to_sql())
# SELECT id, name, email FROM users WHERE age > 18 AND active = true ORDER BY name LIMIT 10
```

### Example 3: HTTP Request Builder

```python
class HttpRequest:
    def __init__(self):
        self.method = "GET"
        self.url = ""
        self.headers = {}
        self.body = None
        self.timeout = 30

    def __str__(self) -> str:
        return f"{self.method} {self.url} (timeout: {self.timeout}s)"

class RequestBuilder:
    def __init__(self):
        self._request = HttpRequest()

    def get(self, url: str) -> 'RequestBuilder':
        self._request.method = "GET"
        self._request.url = url
        return self

    def post(self, url: str) -> 'RequestBuilder':
        self._request.method = "POST"
        self._request.url = url
        return self

    def add_header(self, key: str, value: str) -> 'RequestBuilder':
        self._request.headers[key] = value
        return self

    def with_body(self, body: dict) -> 'RequestBuilder':
        self._request.body = body
        return self

    def timeout(self, seconds: int) -> 'RequestBuilder':
        self._request.timeout = seconds
        return self

    def build(self) -> HttpRequest:
        return self._request

# Usage
request = (RequestBuilder()
           .post("https://api.example.com/users")
           .add_header("Content-Type", "application/json")
           .add_header("Authorization", "Bearer token123")
           .with_body({"name": "John", "email": "john@example.com"})
           .timeout(60)
           .build())

print(request)
```

## When to Use
- ✅ Object has many optional parameters
- ✅ Avoid telescoping constructors
- ✅ Want fluent, readable object construction
- ✅ Need different representations of same object

## Advantages
- Avoids telescoping constructors
- Clean, readable code with method chaining
- Immutable objects can be constructed step-by-step
- Single Responsibility Principle

## Disadvantages
- More code (separate builder class)
- Overkill for simple objects

## vs. Other Patterns
- **Factory Method:** Simple object creation, not step-by-step
- **Abstract Factory:** Creates families of objects, not complex single objects

## Summary
Builder pattern is ideal for constructing complex objects with many optional parameters. Promotes clean, readable code through method chaining.
