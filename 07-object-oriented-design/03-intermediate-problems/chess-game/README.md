# Chess Game - OOD Design

**Difficulty:** Intermediate-Advanced
**Interview Frequency:** Medium
**Key Concepts:** Complex Inheritance, Move Validation, Game State
**Companies:** Game companies, Google, Microsoft

---

## Problem Statement

Design a chess game with piece movement validation, check/checkmate detection, and turn management.

---

## Implementation

```python
from enum import Enum
from typing import List, Optional, Tuple
from abc import ABC, abstractmethod


class Color(Enum):
    WHITE = "White"
    BLACK = "Black"


class PieceType(Enum):
    KING = "King"
    QUEEN = "Queen"
    ROOK = "Rook"
    BISHOP = "Bishop"
    KNIGHT = "Knight"
    PAWN = "Pawn"


class Position:
    def __init__(self, row: int, col: int):
        self.row = row
        self.col = col

    def is_valid(self) -> bool:
        return 0 <= self.row < 8 and 0 <= self.col < 8

    def __eq__(self, other) -> bool:
        return self.row == other.row and self.col == other.col


class Piece(ABC):
    def __init__(self, color: Color, position: Position):
        self.color = color
        self.position = position
        self.has_moved = False

    @abstractmethod
    def get_valid_moves(self, board: 'Board') -> List[Position]:
        pass

    def __str__(self) -> str:
        return f"{self.color.value[0]}{self.__class__.__name__[0]}"


class King(Piece):
    def get_valid_moves(self, board: 'Board') -> List[Position]:
        moves = []
        directions = [(-1,-1),(-1,0),(-1,1),(0,-1),(0,1),(1,-1),(1,0),(1,1)]

        for dr, dc in directions:
            new_pos = Position(self.position.row + dr, self.position.col + dc)
            if new_pos.is_valid():
                target = board.get_piece(new_pos)
                if target is None or target.color != self.color:
                    moves.append(new_pos)
        return moves


class Queen(Piece):
    def get_valid_moves(self, board: 'Board') -> List[Position]:
        moves = []
        directions = [(-1,-1),(-1,0),(-1,1),(0,-1),(0,1),(1,-1),(1,0),(1,1)]

        for dr, dc in directions:
            for i in range(1, 8):
                new_pos = Position(self.position.row + dr*i, self.position.col + dc*i)
                if not new_pos.is_valid():
                    break
                target = board.get_piece(new_pos)
                if target is None:
                    moves.append(new_pos)
                elif target.color != self.color:
                    moves.append(new_pos)
                    break
                else:
                    break
        return moves


class Rook(Piece):
    def get_valid_moves(self, board: 'Board') -> List[Position]:
        moves = []
        directions = [(-1,0),(1,0),(0,-1),(0,1)]

        for dr, dc in directions:
            for i in range(1, 8):
                new_pos = Position(self.position.row + dr*i, self.position.col + dc*i)
                if not new_pos.is_valid():
                    break
                target = board.get_piece(new_pos)
                if target is None:
                    moves.append(new_pos)
                elif target.color != self.color:
                    moves.append(new_pos)
                    break
                else:
                    break
        return moves


class Knight(Piece):
    def get_valid_moves(self, board: 'Board') -> List[Position]:
        moves = []
        knight_moves = [(-2,-1),(-2,1),(-1,-2),(-1,2),(1,-2),(1,2),(2,-1),(2,1)]

        for dr, dc in knight_moves:
            new_pos = Position(self.position.row + dr, self.position.col + dc)
            if new_pos.is_valid():
                target = board.get_piece(new_pos)
                if target is None or target.color != self.color:
                    moves.append(new_pos)
        return moves


class Board:
    def __init__(self):
        self.grid: List[List[Optional[Piece]]] = [[None]*8 for _ in range(8)]

    def get_piece(self, position: Position) -> Optional[Piece]:
        if position.is_valid():
            return self.grid[position.row][position.col]
        return None

    def set_piece(self, position: Position, piece: Optional[Piece]):
        if position.is_valid():
            self.grid[position.row][position.col] = piece

    def move_piece(self, from_pos: Position, to_pos: Position) -> bool:
        piece = self.get_piece(from_pos)
        if piece is None:
            return False

        valid_moves = piece.get_valid_moves(self)
        if to_pos not in valid_moves:
            return False

        # Perform move
        self.set_piece(to_pos, piece)
        self.set_piece(from_pos, None)
        piece.position = to_pos
        piece.has_moved = True
        return True

    def display(self):
        print("\n  a b c d e f g h")
        for row in range(8):
            print(f"{8-row} ", end="")
            for col in range(8):
                piece = self.grid[row][col]
                print(f"{piece if piece else '.'} ", end="")
            print(f"{8-row}")
        print("  a b c d e f g h\n")


class Game:
    def __init__(self):
        self.board = Board()
        self.current_turn = Color.WHITE
        self._initialize_board()

    def _initialize_board(self):
        # Place kings
        self.board.set_piece(Position(0, 4), King(Color.BLACK, Position(0, 4)))
        self.board.set_piece(Position(7, 4), King(Color.WHITE, Position(7, 4)))

        # Place queens
        self.board.set_piece(Position(0, 3), Queen(Color.BLACK, Position(0, 3)))
        self.board.set_piece(Position(7, 3), Queen(Color.WHITE, Position(7, 3)))

        # Place rooks
        for col in [0, 7]:
            self.board.set_piece(Position(0, col), Rook(Color.BLACK, Position(0, col)))
            self.board.set_piece(Position(7, col), Rook(Color.WHITE, Position(7, col)))

        # Place knights
        for col in [1, 6]:
            self.board.set_piece(Position(0, col), Knight(Color.BLACK, Position(0, col)))
            self.board.set_piece(Position(7, col), Knight(Color.WHITE, Position(7, col)))

    def make_move(self, from_row: int, from_col: int, to_row: int, to_col: int) -> bool:
        from_pos = Position(from_row, from_col)
        to_pos = Position(to_row, to_col)

        piece = self.board.get_piece(from_pos)
        if piece is None or piece.color != self.current_turn:
            print("Invalid move: Not your piece")
            return False

        if self.board.move_piece(from_pos, to_pos):
            self.current_turn = Color.BLACK if self.current_turn == Color.WHITE else Color.WHITE
            return True

        print("Invalid move")
        return False


def main():
    game = Game()
    game.board.display()

    # Example moves
    print(f"{game.current_turn.value}'s turn")
    game.make_move(6, 4, 4, 4)  # White pawn move would go here with full implementation
    game.board.display()


if __name__ == "__main__":
    main()
```

---

## Design Patterns
- **Strategy Pattern:** Different piece movement strategies
- **Command Pattern:** Move history for undo
- **State Pattern:** Game states (playing, check, checkmate)

## Extensions
- Implement all piece types (Bishop, Pawn)
- Add castling, en passant
- Check and checkmate detection
- Move history and undo

This tests complex inheritance hierarchies and move validation logic.
