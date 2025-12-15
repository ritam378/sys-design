# Design a Deck of Cards

## Problem Statement

Design a **deck of cards** system that can be used for various card games like Poker, Blackjack, etc.

## Requirements

1. **Standard 52-card deck**: 4 suits × 13 ranks
2. **Shuffle**: Randomize card order
3. **Deal cards**: Remove and return cards from deck
4. **Reset**: Restore deck to original state
5. **Extensible**: Support different card games

## Class Diagram

```
┌──────────────┐
│     Suit     │
│  (Enum)      │
├──────────────┤
│ HEARTS       │
│ DIAMONDS     │
│ CLUBS        │
│ SPADES       │
└──────────────┘

┌──────────────┐
│     Rank     │
│  (Enum)      │
├──────────────┤
│ ACE          │
│ TWO..TEN     │
│ JACK         │
│ QUEEN        │
│ KING         │
└──────────────┘

┌──────────────┐         ┌──────────────┐
│     Card     │         │     Deck     │
├──────────────┤         ├──────────────┤
│ - suit: Suit │    ┌───→│ - cards: []  │
│ - rank: Rank │    │    ├──────────────┤
├──────────────┤    │    │ shuffle()    │
│ __str__()    │────┘    │ deal()       │
│ value()      │         │ reset()      │
└──────────────┘         │ remaining()  │
                         └──────────────┘
```

## Implementation

```python
from enum import Enum
import random

class Suit(Enum):
    """Card suits"""
    HEARTS = '♥'
    DIAMONDS = '♦'
    CLUBS = '♣'
    SPADES = '♠'

class Rank(Enum):
    """Card ranks"""
    ACE = (1, 'A')
    TWO = (2, '2')
    THREE = (3, '3')
    FOUR = (4, '4')
    FIVE = (5, '5')
    SIX = (6, '6')
    SEVEN = (7, '7')
    EIGHT = (8, '8')
    NINE = (9, '9')
    TEN = (10, '10')
    JACK = (11, 'J')
    QUEEN = (12, 'Q')
    KING = (13, 'K')

    def __init__(self, value, symbol):
        self.value = value
        self.symbol = symbol

class Card:
    """Represents a single playing card"""

    def __init__(self, suit: Suit, rank: Rank):
        self.suit = suit
        self.rank = rank

    def __str__(self):
        return f"{self.rank.symbol}{self.suit.value}"

    def __repr__(self):
        return f"Card({self.suit.name}, {self.rank.name})"

    def get_value(self, game='poker'):
        """
        Get card value for specific game
        Can be overridden for different games
        """
        if game == 'blackjack':
            # Ace can be 1 or 11, face cards are 10
            if self.rank in [Rank.JACK, Rank.QUEEN, Rank.KING]:
                return 10
            elif self.rank == Rank.ACE:
                return 11  # or 1, decided by game logic
            else:
                return self.rank.value
        else:
            # Poker: use rank value
            return self.rank.value

class Deck:
    """Standard 52-card deck"""

    def __init__(self):
        self.cards = []
        self.dealt_cards = []
        self.reset()

    def reset(self):
        """Create a fresh deck of 52 cards"""
        self.cards = [
            Card(suit, rank)
            for suit in Suit
            for rank in Rank
        ]
        self.dealt_cards = []

    def shuffle(self):
        """Randomize card order"""
        random.shuffle(self.cards)

    def deal(self, count=1):
        """
        Deal cards from top of deck

        Args:
            count: Number of cards to deal

        Returns:
            List of Card objects

        Raises:
            ValueError: If not enough cards in deck
        """
        if count > len(self.cards):
            raise ValueError(f"Cannot deal {count} cards, only {len(self.cards)} remaining")

        dealt = []
        for _ in range(count):
            card = self.cards.pop()
            self.dealt_cards.append(card)
            dealt.append(card)

        return dealt

    def remaining(self):
        """Number of cards left in deck"""
        return len(self.cards)

    def __len__(self):
        return len(self.cards)

    def __str__(self):
        return f"Deck({len(self.cards)} cards remaining)"

# Example usage
if __name__ == '__main__':
    # Create and shuffle deck
    deck = Deck()
    print(f"New deck: {deck}")
    deck.shuffle()

    # Deal 5 cards
    hand = deck.deal(5)
    print(f"Your hand: {[str(card) for card in hand]}")
    print(f"Remaining: {deck.remaining()}")

    # Reset deck
    deck.reset()
    print(f"After reset: {deck}")
```

## Extended Implementation: Game-Specific Classes

### Hand Class

```python
class Hand:
    """Represents a player's hand"""

    def __init__(self, player_name):
        self.player_name = player_name
        self.cards = []

    def add_card(self, card):
        """Add card to hand"""
        self.cards.append(card)

    def add_cards(self, cards):
        """Add multiple cards to hand"""
        self.cards.extend(cards)

    def remove_card(self, card):
        """Remove specific card from hand"""
        self.cards.remove(card)

    def clear(self):
        """Remove all cards from hand"""
        self.cards.clear()

    def size(self):
        """Number of cards in hand"""
        return len(self.cards)

    def sort(self):
        """Sort cards by rank"""
        self.cards.sort(key=lambda card: card.rank.value)

    def __str__(self):
        cards_str = ', '.join(str(card) for card in self.cards)
        return f"{self.player_name}'s hand: [{cards_str}]"

# Usage
hand = Hand("Alice")
hand.add_cards(deck.deal(5))
print(hand)
```

### Blackjack Example

```python
class BlackjackHand(Hand):
    """Hand for Blackjack game"""

    def calculate_value(self):
        """
        Calculate hand value for Blackjack
        Handle Ace as 1 or 11
        """
        total = 0
        aces = 0

        for card in self.cards:
            if card.rank == Rank.ACE:
                aces += 1
                total += 11
            elif card.rank in [Rank.JACK, Rank.QUEEN, Rank.KING]:
                total += 10
            else:
                total += card.rank.value

        # Adjust for Aces (convert 11 to 1 if busted)
        while total > 21 and aces > 0:
            total -= 10
            aces -= 1

        return total

    def is_blackjack(self):
        """Check if hand is blackjack (21 with 2 cards)"""
        return len(self.cards) == 2 and self.calculate_value() == 21

    def is_busted(self):
        """Check if hand value exceeds 21"""
        return self.calculate_value() > 21

# Usage
bj_hand = BlackjackHand("Bob")
bj_hand.add_cards([Card(Suit.HEARTS, Rank.ACE), Card(Suit.SPADES, Rank.KING)])
print(f"Value: {bj_hand.calculate_value()}")  # 21
print(f"Blackjack: {bj_hand.is_blackjack()}")  # True
```

### Poker Hand Evaluation

```python
from collections import Counter

class PokerHand(Hand):
    """Hand for Poker game with ranking"""

    HAND_RANKINGS = {
        'HIGH_CARD': 1,
        'ONE_PAIR': 2,
        'TWO_PAIR': 3,
        'THREE_OF_A_KIND': 4,
        'STRAIGHT': 5,
        'FLUSH': 6,
        'FULL_HOUSE': 7,
        'FOUR_OF_A_KIND': 8,
        'STRAIGHT_FLUSH': 9,
        'ROYAL_FLUSH': 10
    }

    def evaluate(self):
        """Determine hand ranking"""
        if self.is_royal_flush():
            return 'ROYAL_FLUSH'
        elif self.is_straight_flush():
            return 'STRAIGHT_FLUSH'
        elif self.is_four_of_a_kind():
            return 'FOUR_OF_A_KIND'
        elif self.is_full_house():
            return 'FULL_HOUSE'
        elif self.is_flush():
            return 'FLUSH'
        elif self.is_straight():
            return 'STRAIGHT'
        elif self.is_three_of_a_kind():
            return 'THREE_OF_A_KIND'
        elif self.is_two_pair():
            return 'TWO_PAIR'
        elif self.is_one_pair():
            return 'ONE_PAIR'
        else:
            return 'HIGH_CARD'

    def is_flush(self):
        """All cards same suit"""
        suits = [card.suit for card in self.cards]
        return len(set(suits)) == 1

    def is_straight(self):
        """5 consecutive ranks"""
        values = sorted([card.rank.value for card in self.cards])
        return values == list(range(values[0], values[0] + 5))

    def is_royal_flush(self):
        """A, K, Q, J, 10 of same suit"""
        if not self.is_flush():
            return False
        values = sorted([card.rank.value for card in self.cards])
        return values == [1, 10, 11, 12, 13]  # Ace can be high

    def is_straight_flush(self):
        """Straight and flush"""
        return self.is_straight() and self.is_flush()

    def is_four_of_a_kind(self):
        """4 cards of same rank"""
        ranks = [card.rank for card in self.cards]
        return 4 in Counter(ranks).values()

    def is_full_house(self):
        """3 of a kind + pair"""
        counts = Counter([card.rank for card in self.cards]).values()
        return sorted(counts) == [2, 3]

    def is_three_of_a_kind(self):
        """3 cards of same rank"""
        ranks = [card.rank for card in self.cards]
        return 3 in Counter(ranks).values()

    def is_two_pair(self):
        """2 different pairs"""
        counts = Counter([card.rank for card in self.cards]).values()
        return sorted(counts) == [1, 2, 2]

    def is_one_pair(self):
        """1 pair"""
        ranks = [card.rank for card in self.cards]
        return 2 in Counter(ranks).values()

# Usage
poker_hand = PokerHand("Charlie")
poker_hand.add_cards([
    Card(Suit.HEARTS, Rank.ACE),
    Card(Suit.HEARTS, Rank.KING),
    Card(Suit.HEARTS, Rank.QUEEN),
    Card(Suit.HEARTS, Rank.JACK),
    Card(Suit.HEARTS, Rank.TEN)
])
print(f"Hand ranking: {poker_hand.evaluate()}")  # ROYAL_FLUSH
```

## Design Patterns Used

### 1. Factory Pattern

```python
class CardFactory:
    """Factory for creating cards"""

    @staticmethod
    def create_standard_deck():
        """Create standard 52-card deck"""
        return Deck()

    @staticmethod
    def create_custom_deck(num_decks=1):
        """Create multiple decks (for games like Blackjack)"""
        deck = Deck()
        deck.cards = []
        for _ in range(num_decks):
            deck.cards.extend([
                Card(suit, rank)
                for suit in Suit
                for rank in Rank
            ])
        return deck
```

### 2. Strategy Pattern (Game Rules)

```python
from abc import ABC, abstractmethod

class GameStrategy(ABC):
    """Abstract strategy for card game rules"""

    @abstractmethod
    def deal_initial_hand(self, deck, num_players):
        """Deal initial cards to players"""
        pass

    @abstractmethod
    def calculate_winner(self, hands):
        """Determine winning hand"""
        pass

class PokerStrategy(GameStrategy):
    def deal_initial_hand(self, deck, num_players):
        hands = [PokerHand(f"Player {i}") for i in range(num_players)]
        for _ in range(5):  # 5 cards per player
            for hand in hands:
                hand.add_card(deck.deal(1)[0])
        return hands

    def calculate_winner(self, hands):
        ranked_hands = [(hand, PokerHand.HAND_RANKINGS[hand.evaluate()]) for hand in hands]
        return max(ranked_hands, key=lambda x: x[1])[0]

class BlackjackStrategy(GameStrategy):
    def deal_initial_hand(self, deck, num_players):
        hands = [BlackjackHand(f"Player {i}") for i in range(num_players)]
        for _ in range(2):  # 2 cards per player
            for hand in hands:
                hand.add_card(deck.deal(1)[0])
        return hands

    def calculate_winner(self, hands):
        # Dealer logic, hit/stand, etc.
        valid_hands = [h for h in hands if not h.is_busted()]
        if not valid_hands:
            return None
        return max(valid_hands, key=lambda h: h.calculate_value())
```

## Interview Tips

### Common Questions

**Q: How would you extend this for different games?**
- Use strategy pattern for game-specific rules
- Extend Hand class for game-specific logic
- Keep Card and Deck generic

**Q: How would you handle multiple decks (e.g., Blackjack with 6 decks)?**
- Factory pattern to create multi-deck
- Shuffle all decks together

**Q: How would you handle card counting?**
- Track dealt_cards
- Implement counting algorithm in separate class
- Observer pattern for card tracking

**Q: What design patterns did you use?**
- Enum for Suit and Rank (type safety)
- Factory for deck creation
- Strategy for game rules
- Template method for hand evaluation

## Key Takeaways

1. **Enums**: Use for fixed set of values (Suit, Rank)
2. **Encapsulation**: Card internals hidden
3. **Extensibility**: Easy to add new games
4. **SOLID Principles**: Single responsibility, open/closed
5. **Design Patterns**: Factory, Strategy

This design is clean, maintainable, and easily extensible for various card games!
