# Shopping Cart - OOD Design

**Difficulty:** Intermediate
**Interview Frequency:** Very High
**Key Concepts:** Cart Management, Pricing, Discounts, Checkout
**Companies:** Amazon, E-commerce platforms, Stripe

---

## Problem Statement

Design a shopping cart system with products, pricing, discounts, tax calculation, and checkout functionality.

---

## Implementation

```python
from enum import Enum
from typing import List, Optional
from abc import ABC, abstractmethod


class Product:
    def __init__(self, product_id: int, name: str, price: float, category: str):
        self.product_id = product_id
        self.name = name
        self.price = price
        self.category = category

    def __str__(self) -> str:
        return f"{self.name} (${self.price:.2f})"


class CartItem:
    def __init__(self, product: Product, quantity: int):
        self.product = product
        self.quantity = quantity

    def get_subtotal(self) -> float:
        return self.product.price * self.quantity

    def __str__(self) -> str:
        return f"{self.product.name} x{self.quantity} = ${self.get_subtotal():.2f}"


class Discount(ABC):
    @abstractmethod
    def apply(self, amount: float) -> float:
        pass


class PercentageDiscount(Discount):
    def __init__(self, percentage: float):
        self.percentage = percentage

    def apply(self, amount: float) -> float:
        return amount * (self.percentage / 100)


class FixedDiscount(Discount):
    def __init__(self, amount: float):
        self.amount = amount

    def apply(self, amount: float) -> float:
        return min(self.amount, amount)


class ShoppingCart:
    def __init__(self):
        self.items: List[CartItem] = []
        self.discount: Optional[Discount] = None

    def add_item(self, product: Product, quantity: int = 1):
        # Check if product already in cart
        for item in self.items:
            if item.product.product_id == product.product_id:
                item.quantity += quantity
                print(f"Updated: {item}")
                return

        item = CartItem(product, quantity)
        self.items.append(item)
        print(f"Added: {item}")

    def remove_item(self, product_id: int):
        self.items = [item for item in self.items if item.product.product_id != product_id]

    def update_quantity(self, product_id: int, quantity: int):
        for item in self.items:
            if item.product.product_id == product_id:
                item.quantity = quantity
                break

    def apply_discount(self, discount: Discount):
        self.discount = discount

    def get_subtotal(self) -> float:
        return sum(item.get_subtotal() for item in self.items)

    def get_discount_amount(self) -> float:
        if self.discount:
            return self.discount.apply(self.get_subtotal())
        return 0.0

    def get_tax(self, tax_rate: float = 0.08) -> float:
        taxable = self.get_subtotal() - self.get_discount_amount()
        return taxable * tax_rate

    def get_total(self, tax_rate: float = 0.08) -> float:
        subtotal = self.get_subtotal()
        discount = self.get_discount_amount()
        tax = self.get_tax(tax_rate)
        return subtotal - discount + tax

    def checkout(self, tax_rate: float = 0.08):
        print("\n" + "="*50)
        print("CHECKOUT SUMMARY")
        print("="*50)

        for item in self.items:
            print(item)

        subtotal = self.get_subtotal()
        discount = self.get_discount_amount()
        tax = self.get_tax(tax_rate)
        total = self.get_total(tax_rate)

        print("-"*50)
        print(f"Subtotal:        ${subtotal:.2f}")
        if discount > 0:
            print(f"Discount:       -${discount:.2f}")
        print(f"Tax (8%):        ${tax:.2f}")
        print("="*50)
        print(f"Total:           ${total:.2f}")
        print("="*50)

    def clear(self):
        self.items.clear()


def main():
    cart = ShoppingCart()

    # Create products
    laptop = Product(1, "Laptop", 999.99, "Electronics")
    mouse = Product(2, "Mouse", 29.99, "Electronics")
    keyboard = Product(3, "Keyboard", 79.99, "Electronics")

    # Add to cart
    cart.add_item(laptop, 1)
    cart.add_item(mouse, 2)
    cart.add_item(keyboard, 1)

    # Update quantity
    cart.update_quantity(2, 3)

    # Apply discount
    cart.apply_discount(PercentageDiscount(10))  # 10% off

    # Checkout
    cart.checkout()


if __name__ == "__main__":
    main()
```

---

## Design Patterns
- **Strategy Pattern:** Discount strategies
- **Observer Pattern:** Cart update notifications
- **Decorator Pattern:** Additional cart features

## Extensions
- Add coupon codes
- Implement buy-one-get-one offers
- Support gift wrapping
- Add shipping cost calculation

This tests aggregation, pricing logic, and discount strategies.
