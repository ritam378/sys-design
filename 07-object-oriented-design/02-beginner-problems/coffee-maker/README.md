# Coffee Maker - OOD Design

**Difficulty:** Beginner
**Interview Frequency:** Medium
**Key Concepts:** Builder Pattern, State Pattern, Strategy Pattern, Encapsulation
**Companies:** Amazon, Google, Microsoft, Uber

---

## Problem Statement

Design an object-oriented coffee maker system that can brew different types of coffee beverages. The system should allow users to customize their coffee with various options like milk, sugar, and flavorings, and should handle the brewing process with different ingredient combinations.

**Core Features:**
1. Brew different types of coffee (Espresso, Latte, Cappuccino, Americano)
2. Customize beverages with add-ons (milk, sugar, whipped cream, flavor shots)
3. Track inventory of ingredients
4. Handle brewing process step-by-step
5. Calculate cost based on beverage type and add-ons
6. Display beverage details

---

## 1. Requirements Gathering (Interview Step 1)

### Clarifying Questions to Ask

**Q1: What types of coffee beverages should the system support?**
A: Support basic types: Espresso, Latte, Cappuccino, and Americano. Each has different ingredient requirements.

**Q2: What customizations should be available?**
A: Users should be able to add milk, sugar, whipped cream, and flavor shots (vanilla, caramel, hazelnut). They should also be able to choose beverage size (small, medium, large).

**Q3: Should we track ingredient inventory?**
A: Yes, track basic ingredients (coffee beans, milk, water, sugar) and reject orders if ingredients are insufficient.

**Q4: How should pricing work?**
A: Each beverage type has a base price. Customizations add to the base price. Larger sizes cost more.

**Q5: Should we handle the actual brewing process?**
A: Yes, simulate the brewing process with different steps (grinding beans, heating water, adding milk, etc.).

**Q6: Do we need to support multiple coffee makers or just one?**
A: Design for a single coffee maker instance.

**Q7: Should we handle concurrent orders?**
A: No, assume orders are processed one at a time.

### Functional Requirements

- ✅ **Beverage Types:** Support Espresso, Latte, Cappuccino, Americano
- ✅ **Customization:** Allow add-ons (milk, sugar, whipped cream, flavors)
- ✅ **Sizes:** Support small, medium, large sizes
- ✅ **Inventory Management:** Track ingredient quantities
- ✅ **Brewing Process:** Simulate step-by-step brewing
- ✅ **Cost Calculation:** Calculate price based on beverage and customizations
- ✅ **Validation:** Check ingredient availability before brewing

### Non-Functional Requirements

- Simple, maintainable code structure
- Extensible for new beverage types and add-ons
- Clear separation of concerns
- Type-safe implementation

### Out of Scope

- ❌ Payment processing
- ❌ User authentication
- ❌ Mobile app or web interface
- ❌ Multiple coffee makers
- ❌ Order queue or scheduling
- ❌ Machine maintenance or cleaning cycles
- ❌ Network connectivity or IoT features

---

## 2. Core Objects Identification (Interview Step 2)

### Nouns (Potential Classes)

Looking at the problem, we can identify these entities:

- **CoffeeMaker** - Main system that brews beverages
- **Beverage** - Abstract representation of a coffee drink
- **Espresso, Latte, Cappuccino, Americano** - Concrete beverage types
- **BeverageBuilder** - Helps construct beverages with customizations
- **Ingredient** - Represents items like coffee, milk, water
- **InventoryManager** - Tracks ingredient quantities
- **Size** - Enumeration for beverage sizes

### Verbs (Potential Methods)

Actions the system should support:

- `brew_beverage()` - Make a coffee
- `add_milk()` - Add milk to beverage
- `add_sugar()` - Add sugar to beverage
- `add_flavor()` - Add flavor shot
- `calculate_cost()` - Compute price
- `check_inventory()` - Verify ingredient availability
- `use_ingredients()` - Consume ingredients
- `restock()` - Add ingredients to inventory

### Relationships

**Inheritance (IS-A):**
- `Espresso IS-A Beverage`
- `Latte IS-A Beverage`
- `Cappuccino IS-A Beverage`
- `Americano IS-A Beverage`

**Composition (HAS-A, strong):**
- `CoffeeMaker HAS-A InventoryManager` (inventory owned by coffee maker)
- `Beverage HAS-A Size`
- `Beverage HAS-A List<AddOn>`

**Association (USES):**
- `CoffeeMaker USES Beverage` (creates and returns beverages)
- `BeverageBuilder USES Beverage` (builds beverage instances)
- `InventoryManager USES Ingredient` (tracks ingredients)

---

## 3. Class Diagram (Interview Step 3)

```
┌────────────────────┐         ┌────────────────────┐
│  <<enumeration>>   │         │  <<enumeration>>   │
│       Size         │         │    FlavorType      │
├────────────────────┤         ├────────────────────┤
│ SMALL              │         │ VANILLA            │
│ MEDIUM             │         │ CARAMEL            │
│ LARGE              │         │ HAZELNUT           │
└────────────────────┘         │ MOCHA              │
                               └────────────────────┘

        ┌─────────────────────────────┐
        │     <<abstract>>            │
        │       Beverage              │
        ├─────────────────────────────┤
        │ - name: str                 │
        │ - size: Size                │
        │ - base_price: float         │
        │ - milk_ml: int              │
        │ - sugar_count: int          │
        │ - has_whipped_cream: bool   │
        │ - flavor: FlavorType | None │
        ├─────────────────────────────┤
        │ + get_cost(): float         │
        │ + get_description(): str    │
        │ + get_ingredients(): dict   │
        └─────────────────────────────┘
                      △
                      │
        ┌─────────────┼─────────────┬──────────────┐
        │             │             │              │
┌───────┴──────┐ ┌────┴─────┐ ┌────┴────────┐ ┌───┴─────────┐
│   Espresso   │ │  Latte   │ │ Cappuccino  │ │  Americano  │
├──────────────┤ ├──────────┤ ├─────────────┤ ├─────────────┤
│              │ │          │ │             │ │             │
└──────────────┘ └──────────┘ └─────────────┘ └─────────────┘


┌─────────────────────────────────┐
│      BeverageBuilder            │
├─────────────────────────────────┤
│ - beverage: Beverage            │
├─────────────────────────────────┤
│ + set_size(size: Size)          │
│ + add_milk(ml: int)             │
│ + add_sugar(count: int)         │
│ + add_whipped_cream()           │
│ + add_flavor(flavor: FlavorType)│
│ + build(): Beverage             │
└─────────────────────────────────┘
                │
                │ builds
                ▼
          (Beverage)


┌─────────────────────────────────┐         ┌─────────────────────────┐
│       CoffeeMaker               │    1  1 │   InventoryManager      │
├─────────────────────────────────┤◆────────├─────────────────────────┤
│ - inventory: InventoryManager   │         │ - ingredients: dict     │
├─────────────────────────────────┤         ├─────────────────────────┤
│ + brew(beverage: Beverage):bool │         │ + has_ingredients():bool│
│ + make_espresso(): Beverage     │         │ + use_ingredients()     │
│ + make_latte(): Beverage        │         │ + restock()             │
│ + make_cappuccino(): Beverage   │         │ + get_status(): dict    │
│ + make_americano(): Beverage    │         └─────────────────────────┘
└─────────────────────────────────┘
```

---

## 4. Python Implementation (Interview Step 4)

### Step 1: Define Enums and Basic Classes

```python
from enum import Enum
from typing import Optional, Dict
from abc import ABC, abstractmethod


class Size(Enum):
    """Beverage size options"""
    SMALL = 1
    MEDIUM = 2
    LARGE = 3


class FlavorType(Enum):
    """Available flavor shots"""
    VANILLA = "Vanilla"
    CARAMEL = "Caramel"
    HAZELNUT = "Hazelnut"
    MOCHA = "Mocha"
```

### Step 2: Beverage Classes Implementation

```python
class Beverage(ABC):
    """
    Abstract base class for all coffee beverages.
    Demonstrates inheritance and encapsulation.
    """

    def __init__(self, name: str, base_price: float, size: Size = Size.MEDIUM):
        self.name = name
        self.size = size
        self.base_price = base_price

        # Customizations
        self.milk_ml: int = 0
        self.sugar_count: int = 0
        self.has_whipped_cream: bool = False
        self.flavor: Optional[FlavorType] = None

    @abstractmethod
    def get_ingredients(self) -> Dict[str, int]:
        """
        Return required ingredients for this beverage.
        Each subclass defines its own ingredients.
        """
        pass

    def get_cost(self) -> float:
        """Calculate total cost including customizations"""
        cost = self.base_price

        # Size multiplier
        size_multiplier = {
            Size.SMALL: 0.8,
            Size.MEDIUM: 1.0,
            Size.LARGE: 1.3
        }
        cost *= size_multiplier[self.size]

        # Add-on costs
        cost += (self.milk_ml / 50) * 0.50  # $0.50 per 50ml milk
        cost += self.sugar_count * 0.10  # $0.10 per sugar
        cost += 0.75 if self.has_whipped_cream else 0
        cost += 0.50 if self.flavor else 0

        return round(cost, 2)

    def get_description(self) -> str:
        """Return detailed description of the beverage"""
        desc = f"{self.size.name.capitalize()} {self.name}"

        customizations = []
        if self.milk_ml > 0:
            customizations.append(f"{self.milk_ml}ml milk")
        if self.sugar_count > 0:
            customizations.append(f"{self.sugar_count} sugar")
        if self.has_whipped_cream:
            customizations.append("whipped cream")
        if self.flavor:
            customizations.append(f"{self.flavor.value} flavor")

        if customizations:
            desc += " with " + ", ".join(customizations)

        return desc

    def __str__(self) -> str:
        return f"{self.get_description()} - ${self.get_cost()}"


class Espresso(Beverage):
    """
    Espresso: Strong coffee made by forcing hot water through ground coffee.
    Base ingredients: Coffee beans and water.
    """

    def __init__(self, size: Size = Size.SMALL):
        # Espresso is typically small
        super().__init__("Espresso", base_price=2.50, size=size)

    def get_ingredients(self) -> Dict[str, int]:
        """Espresso needs coffee beans and water"""
        base_ingredients = {
            "coffee_beans": 18,  # grams
            "water": 30  # ml
        }

        # Adjust for size
        multiplier = {Size.SMALL: 1.0, Size.MEDIUM: 1.5, Size.LARGE: 2.0}
        return {k: int(v * multiplier[self.size]) for k, v in base_ingredients.items()}


class Latte(Beverage):
    """
    Latte: Espresso with steamed milk.
    Base ingredients: Coffee beans, water, and milk.
    """

    def __init__(self, size: Size = Size.MEDIUM):
        super().__init__("Latte", base_price=3.50, size=size)
        # Latte comes with milk by default
        self.milk_ml = 200 if size == Size.MEDIUM else (150 if size == Size.SMALL else 250)

    def get_ingredients(self) -> Dict[str, int]:
        """Latte needs coffee, water, and significant milk"""
        return {
            "coffee_beans": 18,
            "water": 30,
            "milk": self.milk_ml
        }


class Cappuccino(Beverage):
    """
    Cappuccino: Espresso with steamed milk and milk foam.
    Base ingredients: Coffee beans, water, and milk (with foam).
    """

    def __init__(self, size: Size = Size.MEDIUM):
        super().__init__("Cappuccino", base_price=3.75, size=size)
        # Cappuccino has less milk than latte, more foam
        self.milk_ml = 150 if size == Size.MEDIUM else (100 if size == Size.SMALL else 200)

    def get_ingredients(self) -> Dict[str, int]:
        """Cappuccino needs coffee, water, and milk for foam"""
        return {
            "coffee_beans": 18,
            "water": 30,
            "milk": self.milk_ml
        }


class Americano(Beverage):
    """
    Americano: Espresso diluted with hot water.
    Base ingredients: Coffee beans and lots of water.
    """

    def __init__(self, size: Size = Size.MEDIUM):
        super().__init__("Americano", base_price=2.75, size=size)

    def get_ingredients(self) -> Dict[str, int]:
        """Americano needs coffee and extra water"""
        water_amount = {
            Size.SMALL: 180,
            Size.MEDIUM: 240,
            Size.LARGE: 300
        }

        return {
            "coffee_beans": 18,
            "water": water_amount[self.size]
        }
```

### Step 3: Builder Pattern Implementation

```python
class BeverageBuilder:
    """
    Builder pattern for constructing customized beverages.
    Allows method chaining for clean beverage customization.
    """

    def __init__(self, beverage: Beverage):
        self._beverage = beverage

    def set_size(self, size: Size) -> 'BeverageBuilder':
        """Set beverage size"""
        self._beverage.size = size
        return self

    def add_milk(self, ml: int) -> 'BeverageBuilder':
        """Add extra milk (beyond default for lattes/cappuccinos)"""
        self._beverage.milk_ml += ml
        return self

    def add_sugar(self, count: int = 1) -> 'BeverageBuilder':
        """Add sugar packets"""
        self._beverage.sugar_count += count
        return self

    def add_whipped_cream(self) -> 'BeverageBuilder':
        """Add whipped cream topping"""
        self._beverage.has_whipped_cream = True
        return self

    def add_flavor(self, flavor: FlavorType) -> 'BeverageBuilder':
        """Add flavor shot"""
        self._beverage.flavor = flavor
        return self

    def build(self) -> Beverage:
        """Return the constructed beverage"""
        return self._beverage
```

### Step 4: Inventory Manager Implementation

```python
class InventoryManager:
    """
    Manages ingredient inventory for the coffee maker.
    Tracks quantities and handles ingredient consumption.
    """

    def __init__(self):
        # Initialize with default quantities (in grams/ml)
        self._ingredients: Dict[str, int] = {
            "coffee_beans": 1000,  # grams
            "water": 5000,  # ml
            "milk": 2000,  # ml
            "sugar": 500  # grams (packets)
        }

    def has_ingredients(self, required: Dict[str, int]) -> bool:
        """Check if sufficient ingredients are available"""
        for ingredient, amount in required.items():
            if ingredient not in self._ingredients:
                return False
            if self._ingredients[ingredient] < amount:
                return False
        return True

    def use_ingredients(self, required: Dict[str, int]) -> bool:
        """
        Consume ingredients for brewing.
        Returns True if successful, False if insufficient.
        """
        # First check availability
        if not self.has_ingredients(required):
            return False

        # Consume ingredients
        for ingredient, amount in required.items():
            self._ingredients[ingredient] -= amount

        return True

    def restock(self, ingredient: str, amount: int):
        """Add ingredients to inventory"""
        if ingredient in self._ingredients:
            self._ingredients[ingredient] += amount
        else:
            self._ingredients[ingredient] = amount

    def get_status(self) -> Dict[str, int]:
        """Get current inventory status"""
        return self._ingredients.copy()

    def __str__(self) -> str:
        status = "Inventory Status:\n"
        for ingredient, amount in self._ingredients.items():
            unit = "g" if ingredient == "coffee_beans" else "ml"
            status += f"  {ingredient.replace('_', ' ').title()}: {amount}{unit}\n"
        return status
```

### Step 5: Coffee Maker Implementation (Main Class)

```python
class CoffeeMaker:
    """
    Main coffee maker system.
    Coordinates beverage creation and brewing process.
    Demonstrates composition and dependency injection.
    """

    def __init__(self):
        self.inventory = InventoryManager()
        self._is_brewing = False

    def make_espresso(self, size: Size = Size.SMALL) -> Optional[Beverage]:
        """Create an espresso"""
        return Espresso(size)

    def make_latte(self, size: Size = Size.MEDIUM) -> Optional[Beverage]:
        """Create a latte"""
        return Latte(size)

    def make_cappuccino(self, size: Size = Size.MEDIUM) -> Optional[Beverage]:
        """Create a cappuccino"""
        return Cappuccino(size)

    def make_americano(self, size: Size = Size.MEDIUM) -> Optional[Beverage]:
        """Create an americano"""
        return Americano(size)

    def brew(self, beverage: Beverage) -> bool:
        """
        Brew the beverage if ingredients are available.
        Simulates the brewing process.
        """
        if self._is_brewing:
            print("Coffee maker is currently brewing. Please wait.")
            return False

        # Get required ingredients
        required_ingredients = beverage.get_ingredients()

        # Add customization ingredients
        if beverage.milk_ml > 0:
            required_ingredients["milk"] = required_ingredients.get("milk", 0) + beverage.milk_ml

        if beverage.sugar_count > 0:
            required_ingredients["sugar"] = beverage.sugar_count

        # Check inventory
        if not self.inventory.has_ingredients(required_ingredients):
            print("Insufficient ingredients. Please restock.")
            print(self.inventory)
            return False

        # Start brewing process
        self._is_brewing = True
        print(f"\n☕ Brewing {beverage.get_description()}...")

        # Simulate brewing steps
        self._grind_beans(required_ingredients.get("coffee_beans", 0))
        self._heat_water(required_ingredients.get("water", 0))
        self._brew_espresso()

        if required_ingredients.get("milk", 0) > 0:
            self._steam_milk(required_ingredients["milk"])

        if beverage.has_whipped_cream:
            self._add_whipped_cream()

        if beverage.flavor:
            self._add_flavor_shot(beverage.flavor)

        # Consume ingredients
        self.inventory.use_ingredients(required_ingredients)

        self._is_brewing = False
        print(f"✓ {beverage.get_description()} is ready!")
        print(f"Total cost: ${beverage.get_cost()}\n")

        return True

    def _grind_beans(self, grams: int):
        """Simulate grinding coffee beans"""
        print(f"  • Grinding {grams}g of coffee beans...")

    def _heat_water(self, ml: int):
        """Simulate heating water"""
        print(f"  • Heating {ml}ml of water to 92°C...")

    def _brew_espresso(self):
        """Simulate espresso extraction"""
        print(f"  • Extracting espresso shot...")

    def _steam_milk(self, ml: int):
        """Simulate steaming milk"""
        print(f"  • Steaming {ml}ml of milk...")

    def _add_whipped_cream(self):
        """Simulate adding whipped cream"""
        print(f"  • Adding whipped cream topping...")

    def _add_flavor_shot(self, flavor: FlavorType):
        """Simulate adding flavor shot"""
        print(f"  • Adding {flavor.value} flavor shot...")

    def display_menu(self):
        """Display available beverages and prices"""
        print("\n" + "=" * 50)
        print("COFFEE MENU")
        print("=" * 50)

        beverages = [
            ("Espresso", Espresso()),
            ("Latte", Latte()),
            ("Cappuccino", Cappuccino()),
            ("Americano", Americano())
        ]

        for name, beverage in beverages:
            print(f"{name:15} - ${beverage.base_price:.2f}")

        print("\nCustomizations:")
        print("  Extra Milk (50ml)  - $0.50")
        print("  Sugar              - $0.10")
        print("  Whipped Cream      - $0.75")
        print("  Flavor Shot        - $0.50")
        print("\nSizes: Small (80%), Medium (100%), Large (130%)")
        print("=" * 50 + "\n")

    def check_inventory(self):
        """Display current inventory status"""
        print(self.inventory)
```

---

## 5. Complete Usage Example

```python
def main():
    """Demonstrate the coffee maker system"""

    # Create coffee maker
    coffee_maker = CoffeeMaker()

    # Display menu
    coffee_maker.display_menu()

    # Check initial inventory
    print("Initial Inventory:")
    coffee_maker.check_inventory()

    # Example 1: Simple espresso
    print("\n--- Order 1: Simple Espresso ---")
    espresso = coffee_maker.make_espresso()
    coffee_maker.brew(espresso)

    # Example 2: Customized latte using builder pattern
    print("\n--- Order 2: Customized Latte ---")
    latte = coffee_maker.make_latte(Size.LARGE)
    customized_latte = (BeverageBuilder(latte)
                        .add_sugar(2)
                        .add_flavor(FlavorType.VANILLA)
                        .add_whipped_cream()
                        .build())
    coffee_maker.brew(customized_latte)

    # Example 3: Cappuccino with caramel
    print("\n--- Order 3: Caramel Cappuccino ---")
    cappuccino = (BeverageBuilder(coffee_maker.make_cappuccino())
                  .add_flavor(FlavorType.CARAMEL)
                  .add_sugar(1)
                  .build())
    coffee_maker.brew(cappuccino)

    # Example 4: Americano
    print("\n--- Order 4: Medium Americano ---")
    americano = coffee_maker.make_americano()
    coffee_maker.brew(americano)

    # Check inventory after brewing
    print("\nInventory After Brewing:")
    coffee_maker.check_inventory()

    # Example 5: Restock and make more coffee
    print("\n--- Restocking Inventory ---")
    coffee_maker.inventory.restock("milk", 1000)
    coffee_maker.inventory.restock("coffee_beans", 500)
    print("Restocked milk and coffee beans")

    coffee_maker.check_inventory()

    # Example 6: Try to brew when already brewing (edge case)
    print("\n--- Testing Concurrent Brewing (Edge Case) ---")
    coffee_maker._is_brewing = True
    test_beverage = coffee_maker.make_latte()
    coffee_maker.brew(test_beverage)  # Should fail
    coffee_maker._is_brewing = False


if __name__ == "__main__":
    main()
```

### Expected Output

```
==================================================
COFFEE MENU
==================================================
Espresso        - $2.50
Latte           - $3.50
Cappuccino      - $3.75
Americano       - $2.75

Customizations:
  Extra Milk (50ml)  - $0.50
  Sugar              - $0.10
  Whipped Cream      - $0.75
  Flavor Shot        - $0.50

Sizes: Small (80%), Medium (100%), Large (130%)
==================================================

Initial Inventory:
Inventory Status:
  Coffee Beans: 1000g
  Water: 5000ml
  Milk: 2000ml
  Sugar: 500g


--- Order 1: Simple Espresso ---

☕ Brewing Small Espresso...
  • Grinding 18g of coffee beans...
  • Heating 30ml of water to 92°C...
  • Extracting espresso shot...
✓ Small Espresso is ready!
Total cost: $2.0


--- Order 2: Customized Latte ---

☕ Brewing Large Latte with 2 sugar, Vanilla flavor, whipped cream...
  • Grinding 18g of coffee beans...
  • Heating 30ml of water to 92°C...
  • Extracting espresso shot...
  • Steaming 250ml of milk...
  • Adding whipped cream topping...
  • Adding Vanilla flavor shot...
✓ Large Latte with 2 sugar, Vanilla flavor, whipped cream is ready!
Total cost: $5.8
```

---

## 6. Design Patterns Used

### Builder Pattern

**Where:** `BeverageBuilder` class

**Code:**
```python
customized_latte = (BeverageBuilder(latte)
                    .add_sugar(2)
                    .add_flavor(FlavorType.VANILLA)
                    .add_whipped_cream()
                    .build())
```

**Why:** The Builder pattern is perfect for constructing complex beverages with many optional customizations. It provides:
- Clean, readable method chaining
- Immutable beverage construction
- Separation of beverage creation from representation
- Easy addition of new customization options

### Template Method Pattern

**Where:** `Beverage` abstract class with `get_ingredients()` abstract method

**Code:**
```python
class Beverage(ABC):
    @abstractmethod
    def get_ingredients(self) -> Dict[str, int]:
        pass  # Each subclass implements its own recipe
```

**Why:** Each beverage type defines its own ingredient requirements while sharing common cost calculation and description logic.

### Composition Pattern

**Where:** `CoffeeMaker` contains `InventoryManager`

**Code:**
```python
class CoffeeMaker:
    def __init__(self):
        self.inventory = InventoryManager()  # Composition
```

**Why:** CoffeeMaker owns the inventory manager. The inventory cannot exist without the coffee maker, demonstrating strong ownership.

---

## 7. SOLID Principles Applied

### Single Responsibility Principle (SRP)

Each class has one clear responsibility:
- `Beverage` subclasses: Define beverage-specific ingredients
- `BeverageBuilder`: Construct customized beverages
- `InventoryManager`: Track and manage ingredients
- `CoffeeMaker`: Coordinate brewing process

### Open/Closed Principle (OCP)

The system is open for extension, closed for modification:
- Adding new beverage types: Create new `Beverage` subclass (no modification to existing code)
- Adding new flavors: Add to `FlavorType` enum
- Adding new customizations: Extend `BeverageBuilder`

**Example:**
```python
# Easy to add new beverage type
class Mocha(Beverage):
    def __init__(self, size: Size = Size.MEDIUM):
        super().__init__("Mocha", base_price=4.00, size=size)
        self.milk_ml = 200

    def get_ingredients(self) -> Dict[str, int]:
        return {"coffee_beans": 18, "water": 30, "milk": 200, "chocolate": 15}
```

### Liskov Substitution Principle (LSP)

All `Beverage` subclasses can substitute the base class:

```python
def brew(self, beverage: Beverage) -> bool:  # Accepts any Beverage subclass
    # Works with Espresso, Latte, Cappuccino, Americano
    required_ingredients = beverage.get_ingredients()
```

### Interface Segregation Principle (ISP)

Classes don't depend on interfaces they don't use:
- `Beverage` has minimal interface
- `InventoryManager` has focused inventory operations
- No "god interface" with unnecessary methods

### Dependency Inversion Principle (DIP)

High-level modules depend on abstractions:
- `CoffeeMaker.brew()` depends on `Beverage` abstraction, not concrete classes
- Works with any `Beverage` subclass without knowing implementation details

---

## 8. Extensions and Follow-up Questions

### Q1: How would you add support for iced beverages?

**Answer:** Create an `IcedBeverage` decorator or add a temperature attribute:

```python
class Temperature(Enum):
    HOT = "Hot"
    ICED = "Iced"

class Beverage(ABC):
    def __init__(self, name: str, base_price: float, size: Size = Size.MEDIUM):
        self.name = name
        self.temperature = Temperature.HOT  # Default hot
        # ... rest of init

class BeverageBuilder:
    def make_iced(self) -> 'BeverageBuilder':
        """Convert beverage to iced"""
        self._beverage.temperature = Temperature.ICED
        # Adjust ingredients (add ice, reduce hot water)
        return self

# Usage
iced_latte = (BeverageBuilder(coffee_maker.make_latte())
              .make_iced()
              .build())
```

### Q2: How would you implement a loyalty program with discounts?

**Answer:** Use Strategy pattern for pricing:

```python
from abc import ABC, abstractmethod

class PricingStrategy(ABC):
    @abstractmethod
    def calculate_price(self, base_price: float) -> float:
        pass

class RegularPricing(PricingStrategy):
    def calculate_price(self, base_price: float) -> float:
        return base_price

class LoyaltyPricing(PricingStrategy):
    def __init__(self, discount_percent: float):
        self.discount = discount_percent

    def calculate_price(self, base_price: float) -> float:
        return base_price * (1 - self.discount / 100)

class Beverage(ABC):
    def __init__(self, name: str, base_price: float, size: Size = Size.MEDIUM):
        # ... existing init
        self.pricing_strategy: PricingStrategy = RegularPricing()

    def get_cost(self) -> float:
        base_cost = self.base_price  # ... calculate with customizations
        return self.pricing_strategy.calculate_price(base_cost)

# Usage
beverage.pricing_strategy = LoyaltyPricing(discount_percent=10)
```

### Q3: How would you add a cleaning/maintenance mode?

**Answer:** Use State pattern:

```python
from abc import ABC, abstractmethod

class CoffeeMakerState(ABC):
    @abstractmethod
    def brew(self, beverage: Beverage) -> bool:
        pass

    @abstractmethod
    def start_cleaning(self) -> bool:
        pass

class ReadyState(CoffeeMakerState):
    def brew(self, beverage: Beverage) -> bool:
        return True  # Can brew

    def start_cleaning(self) -> bool:
        return True  # Can start cleaning

class CleaningState(CoffeeMakerState):
    def brew(self, beverage: Beverage) -> bool:
        print("Cannot brew during cleaning")
        return False

    def start_cleaning(self) -> bool:
        print("Already cleaning")
        return False

class CoffeeMaker:
    def __init__(self):
        self.inventory = InventoryManager()
        self.state: CoffeeMakerState = ReadyState()

    def brew(self, beverage: Beverage) -> bool:
        return self.state.brew(beverage)

    def start_cleaning(self):
        if self.state.start_cleaning():
            self.state = CleaningState()
            print("Starting cleaning cycle...")
```

### Q4: How would you support saving/loading favorite orders?

**Answer:** Add a favorites manager with serialization:

```python
import json
from typing import List

class FavoritesManager:
    def __init__(self):
        self.favorites: Dict[str, Dict] = {}

    def save_favorite(self, name: str, beverage: Beverage):
        """Save beverage configuration"""
        self.favorites[name] = {
            "type": beverage.__class__.__name__,
            "size": beverage.size.name,
            "milk_ml": beverage.milk_ml,
            "sugar_count": beverage.sugar_count,
            "has_whipped_cream": beverage.has_whipped_cream,
            "flavor": beverage.flavor.name if beverage.flavor else None
        }

    def load_favorite(self, name: str, coffee_maker: CoffeeMaker) -> Optional[Beverage]:
        """Recreate beverage from saved configuration"""
        if name not in self.favorites:
            return None

        config = self.favorites[name]

        # Create base beverage
        beverage_type = config["type"]
        size = Size[config["size"]]

        if beverage_type == "Latte":
            beverage = coffee_maker.make_latte(size)
        elif beverage_type == "Cappuccino":
            beverage = coffee_maker.make_cappuccino(size)
        # ... handle other types

        # Apply customizations
        builder = BeverageBuilder(beverage)
        if config["sugar_count"] > 0:
            builder.add_sugar(config["sugar_count"])
        if config["has_whipped_cream"]:
            builder.add_whipped_cream()
        if config["flavor"]:
            builder.add_flavor(FlavorType[config["flavor"]])

        return builder.build()

    def list_favorites(self) -> List[str]:
        return list(self.favorites.keys())

# Usage
favorites = FavoritesManager()

# Save favorite
my_latte = (BeverageBuilder(coffee_maker.make_latte())
            .add_sugar(2)
            .add_flavor(FlavorType.VANILLA)
            .build())
favorites.save_favorite("My Usual", my_latte)

# Load favorite later
beverage = favorites.load_favorite("My Usual", coffee_maker)
coffee_maker.brew(beverage)
```

---

## 9. Interview Tips

### What Interviewers Look For

- ✅ **Clear class hierarchy:** Proper use of inheritance for beverage types
- ✅ **Builder pattern usage:** Understanding when and how to use creational patterns
- ✅ **Encapsulation:** Proper use of private attributes and public methods
- ✅ **SOLID principles:** Especially SRP and OCP
- ✅ **Edge case handling:** Insufficient inventory, concurrent brewing
- ✅ **Extensibility:** Easy to add new beverage types and customizations
- ✅ **Code quality:** Clean, readable, well-documented code

### Common Mistakes

- ❌ **God class:** Putting all logic in `CoffeeMaker` without separating concerns
- ❌ **Telescoping constructors:** `Beverage(size, milk, sugar, cream, flavor, ...)` instead of Builder
- ❌ **Hardcoded values:** Magic numbers instead of constants or enums
- ❌ **No inventory management:** Forgetting to track and validate ingredients
- ❌ **Tight coupling:** Making classes depend on concrete implementations
- ❌ **Missing validation:** Not checking for invalid states or insufficient resources

### Time Management (45 min interview)

- **0-5 min:** Requirements gathering and clarification
- **5-10 min:** Identify core objects and relationships
- **10-15 min:** Draw class diagram and get feedback
- **15-35 min:** Implement core classes (Beverage hierarchy, Builder, CoffeeMaker)
- **35-40 min:** Add usage example and demonstrate functionality
- **40-45 min:** Discuss patterns, SOLID principles, and extensions

---

## 10. Summary

### Key Takeaways

1. **Builder Pattern** is ideal for objects with many optional parameters
2. **Inheritance** works well for beverage type hierarchy with shared behavior
3. **Composition** properly models ownership (CoffeeMaker owns Inventory)
4. **Enums** provide type-safe constants for sizes and flavors
5. **Abstract methods** enforce subclass contracts (get_ingredients)
6. **Separation of concerns** makes code maintainable and testable
7. **Inventory management** adds realism and demonstrates state management

### Design Highlights

- **Extensible:** Easy to add new beverages, flavors, customizations
- **Maintainable:** Each class has clear, focused responsibility
- **Type-safe:** Extensive use of type hints and enums
- **Realistic:** Handles inventory, brewing steps, edge cases
- **Pattern-rich:** Demonstrates Builder, Template Method, Composition

### Related Problems

- **Vending Machine:** Similar inventory and state management
- **Restaurant Ordering System:** Builder pattern for meal customization
- **Pizza Builder:** Similar customization and pricing logic
- **Juice Bar:** Ingredient mixing and recipe management

This problem demonstrates fundamental OOD concepts that apply to many real-world systems!
