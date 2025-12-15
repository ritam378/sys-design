# State Pattern

## Intent
Allow an object to alter its behavior when its internal state changes. The object will appear to change its class.

## Problem
Object behavior depends on state, leading to large conditionals.

## Solution
Create separate state classes, each implementing behavior for that state.

## Python Implementation

```python
from abc import ABC, abstractmethod

class State(ABC):
    @abstractmethod
    def insert_coin(self, machine: 'VendingMachine'):
        pass

    @abstractmethod
    def select_product(self, machine: 'VendingMachine'):
        pass

    @abstractmethod
    def dispense(self, machine: 'VendingMachine'):
        pass

class NoCoinState(State):
    def insert_coin(self, machine):
        print("Coin inserted")
        machine.set_state(machine.has_coin_state)

    def select_product(self, machine):
        print("Insert coin first")

    def dispense(self, machine):
        print("Insert coin first")

class HasCoinState(State):
    def insert_coin(self, machine):
        print("Coin already inserted")

    def select_product(self, machine):
        print("Product selected")
        machine.set_state(machine.dispense_state)

    def dispense(self, machine):
        print("Select product first")

class DispenseState(State):
    def insert_coin(self, machine):
        print("Please wait, dispensing product")

    def select_product(self, machine):
        print("Please wait, dispensing product")

    def dispense(self, machine):
        print("Dispensing product...")
        machine.set_state(machine.no_coin_state)

class VendingMachine:
    def __init__(self):
        self.no_coin_state = NoCoinState()
        self.has_coin_state = HasCoinState()
        self.dispense_state = DispenseState()
        self._state = self.no_coin_state

    def set_state(self, state: State):
        self._state = state

    def insert_coin(self):
        self._state.insert_coin(self)

    def select_product(self):
        self._state.select_product(self)

    def dispense(self):
        self._state.dispense(self)

# Usage
machine = VendingMachine()
machine.insert_coin()     # Coin inserted
machine.select_product()  # Product selected
machine.dispense()        # Dispensing product...
```

## When to Use
- ✅ Object behavior depends on state
- ✅ Large conditionals based on state
- ✅ State transitions are explicit

## Summary
State eliminates large conditionals by delegating behavior to state-specific classes. Common in workflows, games, UI.
