# Observer Pattern

## Intent
Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified automatically.

## Problem
Need to notify multiple objects about state changes without tight coupling.

## Solution
Subject maintains list of observers and notifies them of state changes.

## Python Implementation

```python
from abc import ABC, abstractmethod

class Observer(ABC):
    @abstractmethod
    def update(self, temp: float, humidity: float):
        pass

class Subject(ABC):
    @abstractmethod
    def attach(self, observer: Observer):
        pass

    @abstractmethod
    def detach(self, observer: Observer):
        pass

    @abstractmethod
    def notify(self):
        pass

class WeatherStation(Subject):
    def __init__(self):
        self._observers = []
        self._temperature = 0.0
        self._humidity = 0.0

    def attach(self, observer: Observer):
        self._observers.append(observer)

    def detach(self, observer: Observer):
        self._observers.remove(observer)

    def notify(self):
        for observer in self._observers:
            observer.update(self._temperature, self._humidity)

    def set_measurements(self, temp: float, humidity: float):
        self._temperature = temp
        self._humidity = humidity
        self.notify()  # Auto-notify

class PhoneDisplay(Observer):
    def update(self, temp: float, humidity: float):
        print(f"Phone: {temp}°F, {humidity}% humidity")

class WebDisplay(Observer):
    def update(self, temp: float, humidity: float):
        print(f"Web: {temp}°F, {humidity}% humidity")

# Usage
station = WeatherStation()
phone = PhoneDisplay()
web = WebDisplay()

station.attach(phone)
station.attach(web)

station.set_measurements(75.0, 65.0)  # Both displays update
```

## When to Use
- ✅ Change to one object requires changing unknown number of others
- ✅ Object should notify others without knowing who they are
- ✅ Event handling systems

## Summary
Observer enables loose coupling between subject and observers. Common in MVC, event systems, pub-sub.
