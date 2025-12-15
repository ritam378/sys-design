# Template Method Pattern

## Intent
Define the skeleton of an algorithm in a method, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps without changing the algorithm's structure.

## Problem
Multiple classes share similar algorithm with minor variations.

## Solution
Define algorithm skeleton in base class, let subclasses override specific steps.

## Python Implementation

```python
from abc import ABC, abstractmethod

class DataParser(ABC):
    def parse(self, filename: str):
        """Template method - defines algorithm skeleton"""
        data = self.read_file(filename)
        parsed = self.parse_data(data)
        validated = self.validate_data(parsed)
        self.save_data(validated)

    def read_file(self, filename: str) -> str:
        """Common implementation"""
        print(f"Reading file: {filename}")
        return f"raw_data_from_{filename}"

    @abstractmethod
    def parse_data(self, data: str):
        """Subclasses implement"""
        pass

    @abstractmethod
    def validate_data(self, data):
        """Subclasses implement"""
        pass

    def save_data(self, data):
        """Common implementation with hook"""
        print(f"Saving data: {data}")
        self.on_save_complete()  # Hook method

    def on_save_complete(self):
        """Hook - subclasses can override"""
        pass

class CSVParser(DataParser):
    def parse_data(self, data: str):
        print(f"Parsing CSV: {data}")
        return f"csv_parsed_{data}"

    def validate_data(self, data):
        print(f"Validating CSV: {data}")
        return f"validated_{data}"

class JSONParser(DataParser):
    def parse_data(self, data: str):
        print(f"Parsing JSON: {data}")
        return f"json_parsed_{data}"

    def validate_data(self, data):
        print(f"Validating JSON: {data}")
        return f"validated_{data}"

    def on_save_complete(self):
        print("JSON parsing complete!")

# Usage
csv_parser = CSVParser()
csv_parser.parse("data.csv")

print()

json_parser = JSONParser()
json_parser.parse("data.json")
```

## When to Use
- ✅ Common algorithm with variant steps
- ✅ Avoid code duplication
- ✅ Control extension points

## Summary
Template Method defines algorithm skeleton in base class, letting subclasses override specific steps while maintaining overall structure.
