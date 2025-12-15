# Elevator System - OOD Design

**Difficulty:** Intermediate
**Interview Frequency:** Very High
**Key Concepts:** State Pattern, Strategy Pattern, Scheduling Algorithms
**Companies:** Google, Microsoft, Amazon, Uber

---

## Problem Statement

Design an elevator control system for a building with multiple elevators. The system should efficiently handle requests, optimize travel time, and manage elevator state transitions.

---

## Implementation

```python
from enum import Enum
from typing import List, Optional
from abc import ABC, abstractmethod


class Direction(Enum):
    UP = 1
    DOWN = -1
    IDLE = 0


class ElevatorState(Enum):
    IDLE = "Idle"
    MOVING_UP = "Moving Up"
    MOVING_DOWN = "Moving Down"
    STOPPED = "Stopped"


class Request:
    def __init__(self, floor: int, direction: Direction):
        self.floor = floor
        self.direction = direction

    def __str__(self) -> str:
        return f"Floor {self.floor} - {self.direction.name}"


class Elevator:
    def __init__(self, elevator_id: int, total_floors: int):
        self.elevator_id = elevator_id
        self.current_floor = 1
        self.direction = Direction.IDLE
        self.state = ElevatorState.IDLE
        self.requests: List[int] = []  # Internal button presses
        self.total_floors = total_floors

    def move_to_floor(self, floor: int):
        print(f"Elevator {self.elevator_id}: Moving from floor {self.current_floor} to {floor}")

        if floor > self.current_floor:
            self.direction = Direction.UP
            self.state = ElevatorState.MOVING_UP
        elif floor < self.current_floor:
            self.direction = Direction.DOWN
            self.state = ElevatorState.MOVING_DOWN

        # Simulate movement
        while self.current_floor != floor:
            self.current_floor += self.direction.value
            print(f"  At floor {self.current_floor}")

        self.state = ElevatorState.STOPPED
        print(f"  Arrived at floor {floor}")

    def add_request(self, floor: int):
        if 1 <= floor <= self.total_floors and floor not in self.requests:
            self.requests.append(floor)
            self.requests.sort()

    def process_requests(self):
        if not self.requests:
            self.direction = Direction.IDLE
            self.state = ElevatorState.IDLE
            return

        # Find next floor based on current direction
        if self.direction == Direction.UP or self.direction == Direction.IDLE:
            next_floors = [f for f in self.requests if f >= self.current_floor]
            if next_floors:
                next_floor = min(next_floors)
            else:
                next_floor = max([f for f in self.requests if f < self.current_floor], default=self.current_floor)
        else:
            next_floors = [f for f in self.requests if f <= self.current_floor]
            if next_floors:
                next_floor = max(next_floors)
            else:
                next_floor = min([f for f in self.requests if f > self.current_floor], default=self.current_floor)

        if next_floor != self.current_floor:
            self.move_to_floor(next_floor)
            self.requests.remove(next_floor)

    def __str__(self) -> str:
        return f"Elevator {self.elevator_id}: Floor {self.current_floor} - {self.state.value}"


class ElevatorController:
    def __init__(self, num_elevators: int, total_floors: int):
        self.elevators = [Elevator(i, total_floors) for i in range(num_elevators)]
        self.total_floors = total_floors
        self.external_requests: List[Request] = []

    def request_elevator(self, floor: int, direction: Direction):
        print(f"\n[REQUEST] Floor {floor} - {direction.name}")
        request = Request(floor, direction)
        self.external_requests.append(request)

        # Find best elevator (simple strategy: closest idle or moving in same direction)
        best_elevator = self._find_best_elevator(request)
        if best_elevator:
            best_elevator.add_request(floor)
            print(f"Assigned to Elevator {best_elevator.elevator_id}")

    def _find_best_elevator(self, request: Request) -> Optional[Elevator]:
        # Simple strategy: find closest elevator
        idle_elevators = [e for e in self.elevators if e.state == ElevatorState.IDLE]

        if idle_elevators:
            return min(idle_elevators, key=lambda e: abs(e.current_floor - request.floor))

        # Find elevator moving in same direction
        moving_elevators = [
            e for e in self.elevators
            if e.direction == request.direction and (
                (request.direction == Direction.UP and e.current_floor <= request.floor) or
                (request.direction == Direction.DOWN and e.current_floor >= request.floor)
            )
        ]

        if moving_elevators:
            return min(moving_elevators, key=lambda e: abs(e.current_floor - request.floor))

        # Default: closest elevator
        return min(self.elevators, key=lambda e: abs(e.current_floor - request.floor))

    def step(self):
        for elevator in self.elevators:
            elevator.process_requests()

    def display_status(self):
        print(f"\n{'='*50}")
        print("Elevator System Status")
        print(f"{'='*50}")
        for elevator in self.elevators:
            print(elevator)
        print(f"{'='*50}\n")


def main():
    # 3 elevators, 10 floors
    controller = ElevatorController(num_elevators=3, total_floors=10)

    controller.display_status()

    # Simulate requests
    controller.request_elevator(5, Direction.UP)
    controller.step()

    controller.request_elevator(3, Direction.DOWN)
    controller.step()

    controller.request_elevator(8, Direction.UP)
    controller.step()

    controller.display_status()


if __name__ == "__main__":
    main()
```

---

## Design Patterns

**State Pattern:** Elevator states (Idle, Moving, Stopped)
**Strategy Pattern:** Different scheduling algorithms
**Singleton:** ElevatorController (optional)

## SOLID Principles
- **SRP:** Elevator manages movement, Controller manages assignment
- **OCP:** Easy to add new scheduling strategies
- **Strategy Pattern:** Pluggable scheduling algorithms

## Interview Tips
- Discuss SCAN, LOOK scheduling algorithms
- Handle edge cases (same floor requests, out of range)
- Optimize for minimal wait time
- Consider energy efficiency

This problem tests understanding of state machines, scheduling, and system coordination.
