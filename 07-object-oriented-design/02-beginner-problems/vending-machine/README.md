# Design a Vending Machine

## Problem Statement

Design a **vending machine** that dispenses products when users insert money and make selections.

## Requirements

1. **Display products**: Show available products and prices
2. **Accept money**: Coins and bills
3. **Dispense product**: After payment received
4. **Return change**: Calculate and return change
5. **Cancel transaction**: Return money
6. **Refill**: Add inventory
7. **State management**: Handle states (idle, has money, dispensing, etc.)

## Class Diagram

```
┌────────────────────┐
│  VendingMachine    │
├────────────────────┤
│ - state: State     │
│ - inventory: {}    │
│ - current_amount   │
├────────────────────┤
│ insert_money()     │
│ select_product()   │
│ dispense()         │
│ return_change()    │
│ cancel()           │
└─────────┬──────────┘
          │
          │ uses
          ↓
┌──────────────────────────────────────────────────┐
│                    State                         │
│                 (Abstract)                       │
├──────────────────────────────────────────────────┤
│ + insert_money()                                 │
│ + select_product()                               │
│ + dispense()                                     │
│ + cancel()                                       │
└──────────────────────────────────────────────────┘
          ↑
          │ implements
    ┌─────┴────────┬───────────┬────────────┐
    │              │           │            │
┌───────────┐ ┌─────────┐ ┌──────────┐ ┌──────────┐
│ IdleState │ │HasMoney │ │Dispensing│ │ NoChange │
│           │ │  State  │ │  State   │ │  State   │
└───────────┘ └─────────┘ └──────────┘ └──────────┘
```

## Implementation

### Product Class

```python
class Product:
    """Represents a product in the vending machine"""

    def __init__(self, name, price, code):
        self.name = name
        self.price = price
        self.code = code

    def __str__(self):
        return f"{self.code}: {self.name} - ${self.price}"

    def __repr__(self):
        return f"Product('{self.name}', {self.price}, '{self.code}')"
```

### Inventory Management

```python
class Inventory:
    """Manages product inventory"""

    def __init__(self):
        self.products = {}  # product_code -> (Product, quantity)

    def add_product(self, product, quantity=1):
        """Add product to inventory"""
        if product.code in self.products:
            current_product, current_qty = self.products[product.code]
            self.products[product.code] = (current_product, current_qty + quantity)
        else:
            self.products[product.code] = (product, quantity)

    def is_available(self, product_code):
        """Check if product is in stock"""
        if product_code not in self.products:
            return False
        _, quantity = self.products[product_code]
        return quantity > 0

    def get_product(self, product_code):
        """Get product by code"""
        if product_code in self.products:
            product, _ = self.products[product_code]
            return product
        return None

    def dispense_product(self, product_code):
        """Remove one unit from inventory"""
        if not self.is_available(product_code):
            raise ValueError(f"Product {product_code} out of stock")

        product, quantity = self.products[product_code]
        self.products[product_code] = (product, quantity - 1)
        return product

    def display_inventory(self):
        """Display all products and quantities"""
        for code, (product, quantity) in self.products.items():
            status = f"({quantity} left)" if quantity > 0 else "(OUT OF STOCK)"
            print(f"{product} {status}")
```

### State Pattern Implementation

```python
from abc import ABC, abstractmethod

class State(ABC):
    """Abstract state class"""

    @abstractmethod
    def insert_money(self, machine, amount):
        """Handle money insertion"""
        pass

    @abstractmethod
    def select_product(self, machine, product_code):
        """Handle product selection"""
        pass

    @abstractmethod
    def dispense(self, machine):
        """Handle product dispensing"""
        pass

    @abstractmethod
    def cancel(self, machine):
        """Handle transaction cancellation"""
        pass

class IdleState(State):
    """Machine is idle, waiting for money"""

    def insert_money(self, machine, amount):
        print(f"Money inserted: ${amount}")
        machine.current_amount += amount
        machine.set_state(machine.has_money_state)
        print(f"Total: ${machine.current_amount}")

    def select_product(self, machine, product_code):
        print("Please insert money first")

    def dispense(self, machine):
        print("Please insert money and select product")

    def cancel(self, machine):
        print("No transaction to cancel")

class HasMoneyState(State):
    """Machine has money, waiting for product selection"""

    def insert_money(self, machine, amount):
        print(f"Additional money inserted: ${amount}")
        machine.current_amount += amount
        print(f"Total: ${machine.current_amount}")

    def select_product(self, machine, product_code):
        # Check if product exists and is in stock
        if not machine.inventory.is_available(product_code):
            print(f"Product {product_code} not available")
            return

        product = machine.inventory.get_product(product_code)

        # Check if enough money
        if machine.current_amount < product.price:
            needed = product.price - machine.current_amount
            print(f"Insufficient funds. Need ${needed} more")
            return

        # Proceed to dispensing
        machine.selected_product = product_code
        machine.set_state(machine.dispensing_state)
        machine.dispense()

    def dispense(self, machine):
        print("Please select a product first")

    def cancel(self, machine):
        print(f"Transaction cancelled. Returning ${machine.current_amount}")
        machine.return_change(machine.current_amount)
        machine.current_amount = 0
        machine.set_state(machine.idle_state)

class DispensingState(State):
    """Machine is dispensing product"""

    def insert_money(self, machine, amount):
        print("Please wait, dispensing product")

    def select_product(self, machine, product_code):
        print("Already dispensing a product")

    def dispense(self, machine):
        try:
            # Dispense product
            product = machine.inventory.dispense_product(machine.selected_product)
            print(f"Dispensing {product.name}")

            # Calculate change
            change = machine.current_amount - product.price

            if change > 0:
                print(f"Returning change: ${change}")
                machine.return_change(change)

            # Reset machine
            machine.current_amount = 0
            machine.selected_product = None
            machine.set_state(machine.idle_state)
            print("Thank you!")

        except ValueError as e:
            print(f"Error: {e}")
            print(f"Returning ${machine.current_amount}")
            machine.return_change(machine.current_amount)
            machine.current_amount = 0
            machine.selected_product = None
            machine.set_state(machine.idle_state)

    def cancel(self, machine):
        print("Cannot cancel during dispensing")

class NoChangeState(State):
    """Machine has no change"""

    def insert_money(self, machine, amount):
        print("Exact change only - cannot accept money")

    def select_product(self, machine, product_code):
        print("Exact change only")

    def dispense(self, machine):
        print("Exact change only")

    def cancel(self, machine):
        print("No transaction in progress")
```

### Vending Machine Class

```python
class VendingMachine:
    """Main vending machine class"""

    def __init__(self):
        self.inventory = Inventory()
        self.current_amount = 0
        self.selected_product = None

        # Initialize states
        self.idle_state = IdleState()
        self.has_money_state = HasMoneyState()
        self.dispensing_state = DispensingState()
        self.no_change_state = NoChangeState()

        # Set initial state
        self.current_state = self.idle_state

        # Change management
        self.change_available = 100  # Assume $100 in change

    def set_state(self, state):
        """Change machine state"""
        self.current_state = state

    def insert_money(self, amount):
        """Insert money into machine"""
        self.current_state.insert_money(self, amount)

    def select_product(self, product_code):
        """Select product by code"""
        self.current_state.select_product(self, product_code)

    def dispense(self):
        """Dispense selected product"""
        self.current_state.dispense(self)

    def cancel(self):
        """Cancel transaction"""
        self.current_state.cancel(self)

    def return_change(self, amount):
        """Return change to user"""
        if amount > self.change_available:
            print(f"Warning: Low on change. Can only return ${self.change_available}")
            self.change_available = 0
            self.set_state(self.no_change_state)
        else:
            self.change_available -= amount
            print(f"Change dispensed: ${amount}")

    def add_product(self, product, quantity):
        """Refill inventory"""
        self.inventory.add_product(product, quantity)

    def display_products(self):
        """Show all available products"""
        print("\n=== Available Products ===")
        self.inventory.display_inventory()
        print()

# Usage Example
if __name__ == '__main__':
    # Create vending machine
    vm = VendingMachine()

    # Add products
    vm.add_product(Product("Coke", 1.50, "A1"), quantity=10)
    vm.add_product(Product("Chips", 1.00, "A2"), quantity=5)
    vm.add_product(Product("Candy", 0.75, "A3"), quantity=20)
    vm.add_product(Product("Water", 1.25, "B1"), quantity=15)

    # Display products
    vm.display_products()

    # Scenario 1: Successful purchase
    print("=== Scenario 1: Buy Coke ===")
    vm.insert_money(2.00)
    vm.select_product("A1")
    print()

    # Scenario 2: Insufficient funds
    print("=== Scenario 2: Insufficient funds ===")
    vm.insert_money(1.00)
    vm.select_product("A1")  # Coke costs $1.50
    vm.cancel()
    print()

    # Scenario 3: Exact change
    print("=== Scenario 3: Exact change ===")
    vm.insert_money(0.75)
    vm.select_product("A3")  # Candy costs $0.75
    print()

    # Scenario 4: Product out of stock
    print("=== Scenario 4: Multiple purchases ===")
    for i in range(6):
        vm.insert_money(1.00)
        vm.select_product("A2")  # Chips
    print()
```

## Extended Features

### Payment Processing

```python
class Payment:
    """Handle different payment types"""

    @staticmethod
    def accept_coins(coins):
        """
        Accept coins: quarters (0.25), dimes (0.10), nickels (0.05)
        coins: dict like {0.25: 4, 0.10: 2, 0.05: 1} = $1.45
        """
        total = sum(denomination * count for denomination, count in coins.items())
        return total

    @staticmethod
    def accept_bills(bills):
        """
        Accept bills: $1, $5, $10, $20
        bills: dict like {1: 2, 5: 1} = $7
        """
        total = sum(denomination * count for denomination, count in bills.items())
        return total

    @staticmethod
    def calculate_change(amount):
        """
        Calculate optimal coin/bill combination for change
        Returns: dict of {denomination: count}
        """
        denominations = [20, 10, 5, 1, 0.25, 0.10, 0.05, 0.01]
        change = {}

        for denom in denominations:
            count = int(amount / denom)
            if count > 0:
                change[denom] = count
                amount -= denom * count
                amount = round(amount, 2)

        return change

# Usage
coins = {0.25: 6, 0.10: 2}  # 6 quarters + 2 dimes
total = Payment.accept_coins(coins)
print(f"Total: ${total}")  # $1.70

change = Payment.calculate_change(3.67)
print(f"Change: {change}")
# {1: 3, 0.25: 2, 0.10: 1, 0.05: 1, 0.01: 2}
```

### Admin Interface

```python
class VendingMachineAdmin:
    """Admin interface for vending machine"""

    def __init__(self, machine):
        self.machine = machine

    def refill_product(self, product_code, quantity):
        """Refill specific product"""
        product = self.machine.inventory.get_product(product_code)
        if product:
            self.machine.add_product(product, quantity)
            print(f"Refilled {product.name} (+{quantity})")
        else:
            print(f"Product {product_code} not found")

    def add_change(self, amount):
        """Add change to machine"""
        self.machine.change_available += amount
        print(f"Added ${amount} in change. Total: ${self.machine.change_available}")

    def view_sales_report(self):
        """Display sales statistics"""
        # In real implementation, track sales in database
        print("=== Sales Report ===")
        print("Total sales: $X")
        print("Products sold: Y")
        print("Most popular: Z")

    def set_price(self, product_code, new_price):
        """Update product price"""
        product = self.machine.inventory.get_product(product_code)
        if product:
            old_price = product.price
            product.price = new_price
            print(f"{product.name}: ${old_price} → ${new_price}")
```

## Design Patterns Used

### 1. State Pattern
- Different states for machine (Idle, HasMoney, Dispensing)
- Each state handles actions differently
- Clean state transitions

### 2. Strategy Pattern (Payment)
- Different payment strategies (coins, bills, card)
- Easily extensible to new payment methods

### 3. Singleton Pattern (Optional)

```python
class VendingMachineSingleton:
    """Ensure only one vending machine instance"""

    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

## Interview Tips

### Common Questions

**Q: How would you handle concurrent users?**
- Add locking mechanism
- Queue requests
- Prevent race conditions on inventory

**Q: What if change runs out?**
- Check change availability before accepting payment
- Enter "exact change only" mode
- Alert maintenance

**Q: How would you add card payment?**
- Create CardPayment strategy
- Integrate with payment gateway
- Handle authorization/settlement

**Q: How do you track sales?**
- Add Sales database
- Log each transaction
- Generate reports

**Q: What design patterns did you use?**
- State pattern (machine states)
- Strategy pattern (payment methods)
- Possibly Singleton (one machine instance)

## Key Takeaways

1. **State Pattern**: Perfect for state machines like vending machines
2. **Encapsulation**: Inventory and payment logic separated
3. **Extensibility**: Easy to add products, payment methods
4. **SOLID Principles**: Single responsibility for each class
5. **Error Handling**: Graceful handling of edge cases

This design demonstrates real-world state management and is a common OOD interview question!
