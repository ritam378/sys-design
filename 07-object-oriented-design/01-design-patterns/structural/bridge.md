# Bridge Pattern

## Intent
Decouple an abstraction from its implementation so that the two can vary independently.

## Problem
Inheritance creates tight coupling between abstraction and implementation.

## Solution
Separate abstraction and implementation into two hierarchies connected by composition.

## Python Implementation

```python
from abc import ABC, abstractmethod

# Implementation interface
class Device(ABC):
    @abstractmethod
    def turn_on(self):
        pass

    @abstractmethod
    def turn_off(self):
        pass

    @abstractmethod
    def set_volume(self, volume: int):
        pass

# Concrete implementations
class TV(Device):
    def turn_on(self):
        print("TV: Turning on")

    def turn_off(self):
        print("TV: Turning off")

    def set_volume(self, volume: int):
        print(f"TV: Volume set to {volume}")

class Radio(Device):
    def turn_on(self):
        print("Radio: Turning on")

    def turn_off(self):
        print("Radio: Turning off")

    def set_volume(self, volume: int):
        print(f"Radio: Volume set to {volume}")

# Abstraction
class RemoteControl:
    def __init__(self, device: Device):
        self.device = device

    def toggle_power(self):
        self.device.turn_on()

    def volume_up(self):
        self.device.set_volume(15)

# Extended abstraction
class AdvancedRemote(RemoteControl):
    def mute(self):
        self.device.set_volume(0)

# Usage - abstraction and implementation vary independently
tv = TV()
tv_remote = RemoteControl(tv)
tv_remote.toggle_power()  # TV: Turning on

radio = Radio()
adv_remote = AdvancedRemote(radio)
adv_remote.mute()  # Radio: Volume set to 0
```

## When to Use
- ✅ Avoid permanent binding between abstraction and implementation
- ✅ Both should be extensible by subclassing
- ✅ Changes in implementation shouldn't affect clients

## Summary
Bridge separates abstraction from implementation, allowing both to vary independently. Useful when both have multiple variants.
