# Amazon Shopping Platform - OOD Design

**Difficulty:** Advanced
**Interview Frequency:** Very High
**Key Concepts:** E-commerce, Inventory, Reviews, Recommendations
**Companies:** Amazon, E-commerce platforms

---

## Problem Statement

Design an e-commerce platform like Amazon with products, cart, orders, inventory, reviews, and recommendations.

---

## Implementation

```python
from enum import Enum
from typing import List, Optional, Dict
from datetime import datetime


class ProductCategory(Enum):
    ELECTRONICS = "Electronics"
    BOOKS = "Books"
    CLOTHING = "Clothing"
    HOME = "Home"


class OrderStatus(Enum):
    PENDING = "Pending"
    CONFIRMED = "Confirmed"
    SHIPPED = "Shipped"
    DELIVERED = "Delivered"
    CANCELLED = "Cancelled"


class Product:
    def __init__(self, product_id: int, name: str, price: float, category: ProductCategory, seller_id: int):
        self.product_id = product_id
        self.name = name
        self.price = price
        self.category = category
        self.seller_id = seller_id
        self.reviews: List['Review'] = []

    def get_average_rating(self) -> float:
        if not self.reviews:
            return 0.0
        return sum(r.rating for r in self.reviews) / len(self.reviews)

    def add_review(self, review: 'Review'):
        self.reviews.append(review)


class Review:
    def __init__(self, user_id: int, rating: float, comment: str):
        self.user_id = user_id
        self.rating = rating  # 1-5
        self.comment = comment
        self.timestamp = datetime.now()


class Inventory:
    def __init__(self):
        self.stock: Dict[int, int] = {}  # product_id -> quantity

    def add_stock(self, product_id: int, quantity: int):
        self.stock[product_id] = self.stock.get(product_id, 0) + quantity

    def check_availability(self, product_id: int, quantity: int) -> bool:
        return self.stock.get(product_id, 0) >= quantity

    def reserve(self, product_id: int, quantity: int) -> bool:
        if self.check_availability(product_id, quantity):
            self.stock[product_id] -= quantity
            return True
        return False

    def release(self, product_id: int, quantity: int):
        self.stock[product_id] = self.stock.get(product_id, 0) + quantity


class CartItem:
    def __init__(self, product: Product, quantity: int):
        self.product = product
        self.quantity = quantity

    def get_total(self) -> float:
        return self.product.price * self.quantity


class ShoppingCart:
    def __init__(self, user_id: int):
        self.user_id = user_id
        self.items: List[CartItem] = []

    def add_item(self, product: Product, quantity: int):
        for item in self.items:
            if item.product.product_id == product.product_id:
                item.quantity += quantity
                return
        self.items.append(CartItem(product, quantity))

    def get_total(self) -> float:
        return sum(item.get_total() for item in self.items)


class Order:
    _order_counter = 1

    def __init__(self, user_id: int, items: List[CartItem]):
        self.order_id = Order._order_counter
        Order._order_counter += 1
        self.user_id = user_id
        self.items = items
        self.total = sum(item.get_total() for item in items)
        self.status = OrderStatus.PENDING
        self.created_at = datetime.now()

    def confirm(self):
        self.status = OrderStatus.CONFIRMED

    def ship(self):
        self.status = OrderStatus.SHIPPED

    def deliver(self):
        self.status = OrderStatus.DELIVERED


class AmazonPlatform:
    def __init__(self):
        self.products: Dict[int, Product] = {}
        self.inventory = Inventory()
        self.orders: List[Order] = []

    def add_product(self, product: Product, quantity: int):
        self.products[product.product_id] = product
        self.inventory.add_stock(product.product_id, quantity)

    def search_products(self, query: str) -> List[Product]:
        return [p for p in self.products.values() if query.lower() in p.name.lower()]

    def checkout(self, cart: ShoppingCart) -> Optional[Order]:
        # Check inventory
        for item in cart.items:
            if not self.inventory.check_availability(item.product.product_id, item.quantity):
                print(f"Insufficient stock for {item.product.name}")
                return None

        # Reserve inventory
        for item in cart.items:
            self.inventory.reserve(item.product.product_id, item.quantity)

        # Create order
        order = Order(cart.user_id, cart.items)
        self.orders.append(order)
        order.confirm()
        print(f"✓ Order #{order.order_id} created - Total: ${order.total:.2f}")
        return order


def main():
    platform = AmazonPlatform()

    # Add products
    laptop = Product(1, "Dell Laptop", 999.99, ProductCategory.ELECTRONICS, seller_id=100)
    book = Product(2, "Design Patterns", 49.99, ProductCategory.BOOKS, seller_id=101)

    platform.add_product(laptop, 10)
    platform.add_product(book, 50)

    # Search
    results = platform.search_products("laptop")
    print(f"Found {len(results)} products")

    # Shopping cart
    cart = ShoppingCart(user_id=1)
    cart.add_item(laptop, 1)
    cart.add_item(book, 2)
    print(f"Cart total: ${cart.get_total():.2f}")

    # Checkout
    order = platform.checkout(cart)

    # Add review
    laptop.add_review(Review(user_id=1, rating=4.5, comment="Great laptop!"))
    print(f"Product rating: {laptop.get_average_rating():.1f}")


if __name__ == "__main__":
    main()
```

---

## Design Patterns
- **Observer:** Inventory updates, price changes
- **Strategy:** Recommendation algorithms
- **Facade:** Simplified checkout process

This tests e-commerce workflows, inventory management, and aggregation.
