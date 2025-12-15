# Tic-Tac-Toe - OOD Design

**Difficulty:** Beginner-Intermediate
**Interview Frequency:** Very High
**Key Concepts:** Game State, Win Detection, Turn Management
**Companies:** Common warm-up question at many companies

---

## Problem Statement

Design a Tic-Tac-Toe game with player management, move validation, and win condition checking.

---

## Implementation

```python
from enum import Enum
from typing import Optional, List


class Symbol(Enum):
    X = "X"
    O = "O"
    EMPTY = " "


class Player:
    def __init__(self, name: str, symbol: Symbol):
        self.name = name
        self.symbol = symbol


class Board:
    def __init__(self, size: int = 3):
        self.size = size
        self.grid = [[Symbol.EMPTY]*size for _ in range(size)]

    def is_valid_move(self, row: int, col: int) -> bool:
        return (0 <= row < self.size and 0 <= col < self.size and
                self.grid[row][col] == Symbol.EMPTY)

    def make_move(self, row: int, col: int, symbol: Symbol) -> bool:
        if self.is_valid_move(row, col):
            self.grid[row][col] = symbol
            return True
        return False

    def is_full(self) -> bool:
        return all(cell != Symbol.EMPTY for row in self.grid for cell in row)

    def check_winner(self) -> Optional[Symbol]:
        # Check rows
        for row in self.grid:
            if row[0] != Symbol.EMPTY and all(cell == row[0] for cell in row):
                return row[0]

        # Check columns
        for col in range(self.size):
            if (self.grid[0][col] != Symbol.EMPTY and
                all(self.grid[row][col] == self.grid[0][col] for row in range(self.size))):
                return self.grid[0][col]

        # Check diagonals
        if (self.grid[0][0] != Symbol.EMPTY and
            all(self.grid[i][i] == self.grid[0][0] for i in range(self.size))):
            return self.grid[0][0]

        if (self.grid[0][self.size-1] != Symbol.EMPTY and
            all(self.grid[i][self.size-1-i] == self.grid[0][self.size-1] for i in range(self.size))):
            return self.grid[0][self.size-1]

        return None

    def display(self):
        print("\n" + "  " + " ".join(str(i) for i in range(self.size)))
        for i, row in enumerate(self.grid):
            print(f"{i} " + "|".join(cell.value for cell in row))
            if i < self.size - 1:
                print("  " + "-"*(self.size*2-1))
        print()


class TicTacToe:
    def __init__(self, player1_name: str, player2_name: str):
        self.board = Board()
        self.player1 = Player(player1_name, Symbol.X)
        self.player2 = Player(player2_name, Symbol.O)
        self.current_player = self.player1
        self.winner: Optional[Player] = None
        self.game_over = False

    def play_move(self, row: int, col: int) -> bool:
        if self.game_over:
            print("Game is over!")
            return False

        if not self.board.make_move(row, col, self.current_player.symbol):
            print("Invalid move!")
            return False

        print(f"{self.current_player.name} plays at ({row}, {col})")

        # Check for winner
        winner_symbol = self.board.check_winner()
        if winner_symbol:
            self.winner = self.player1 if winner_symbol == self.player1.symbol else self.player2
            self.game_over = True
            print(f"\n🎉 {self.winner.name} wins!")
            return True

        # Check for draw
        if self.board.is_full():
            self.game_over = True
            print("\n🤝 Game is a draw!")
            return True

        # Switch player
        self.current_player = self.player2 if self.current_player == self.player1 else self.player1
        return True

    def display(self):
        self.board.display()
        if not self.game_over:
            print(f"{self.current_player.name}'s turn ({self.current_player.symbol.value})")


def main():
    game = TicTacToe("Alice", "Bob")

    moves = [
        (0, 0),  # X
        (1, 1),  # O
        (0, 1),  # X
        (0, 2),  # O
        (1, 0),  # X
        (2, 2),  # O
        (2, 0),  # X wins
    ]

    game.display()

    for row, col in moves:
        game.play_move(row, col)
        game.display()

        if game.game_over:
            break


if __name__ == "__main__":
    main()
```

---

## Design Patterns
- **Strategy Pattern:** AI player strategies
- **State Pattern:** Game states (playing, won, draw)
- **Observer Pattern:** UI updates on moves

## Extensions
- Add AI player with minimax algorithm
- Support larger grids (4x4, 5x5)
- Implement undo functionality
- Add move timer

This tests basic game logic, state management, and win condition checking.
