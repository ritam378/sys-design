# ATM Machine - OOD Design

**Difficulty:** Intermediate
**Interview Frequency:** High
**Key Concepts:** State Pattern, Transaction Management, Security
**Companies:** Banks, Financial Services, Amazon, Google

---

## Problem Statement

Design an ATM that handles cash withdrawals, deposits, balance inquiries, and PIN validation with proper state management and security.

---

## Implementation

```python
from enum import Enum
from typing import Optional


class ATMState(Enum):
    IDLE = "Idle"
    CARD_INSERTED = "Card Inserted"
    PIN_VERIFIED = "PIN Verified"
    TRANSACTION = "Transaction"


class TransactionType(Enum):
    WITHDRAWAL = "Withdrawal"
    DEPOSIT = "Deposit"
    BALANCE_INQUIRY = "Balance Inquiry"


class Card:
    def __init__(self, card_number: str, pin: str):
        self.card_number = card_number
        self.pin = pin

    def validate_pin(self, entered_pin: str) -> bool:
        return self.pin == entered_pin


class Account:
    def __init__(self, account_number: str, balance: float = 0.0):
        self.account_number = account_number
        self.balance = balance

    def withdraw(self, amount: float) -> bool:
        if amount > self.balance:
            return False
        self.balance -= amount
        return True

    def deposit(self, amount: float):
        self.balance += amount

    def get_balance(self) -> float:
        return self.balance


class Transaction:
    _transaction_counter = 1

    def __init__(self, transaction_type: TransactionType, amount: float):
        self.transaction_id = Transaction._transaction_counter
        Transaction._transaction_counter += 1
        self.type = transaction_type
        self.amount = amount

    def __str__(self) -> str:
        return f"Transaction #{self.transaction_id}: {self.type.value} - ${self.amount:.2f}"


class ATM:
    def __init__(self, atm_id: str, cash_available: float = 10000.0):
        self.atm_id = atm_id
        self.cash_available = cash_available
        self.state = ATMState.IDLE
        self.current_card: Optional[Card] = None
        self.current_account: Optional[Account] = None
        self.pin_attempts = 0
        self.max_pin_attempts = 3

    def insert_card(self, card: Card, account: Account):
        if self.state != ATMState.IDLE:
            print("ATM is busy")
            return False

        self.current_card = card
        self.current_account = account
        self.state = ATMState.CARD_INSERTED
        self.pin_attempts = 0
        print(f"Card {card.card_number} inserted")
        return True

    def enter_pin(self, pin: str) -> bool:
        if self.state != ATMState.CARD_INSERTED:
            print("No card inserted")
            return False

        if self.current_card.validate_pin(pin):
            self.state = ATMState.PIN_VERIFIED
            print("PIN verified successfully")
            return True
        else:
            self.pin_attempts += 1
            if self.pin_attempts >= self.max_pin_attempts:
                print(f"Maximum PIN attempts reached. Card blocked.")
                self.eject_card()
            else:
                print(f"Incorrect PIN. {self.max_pin_attempts - self.pin_attempts} attempts remaining")
            return False

    def withdraw(self, amount: float) -> bool:
        if self.state != ATMState.PIN_VERIFIED:
            print("Please verify PIN first")
            return False

        if amount > self.cash_available:
            print(f"Insufficient cash in ATM. Available: ${self.cash_available:.2f}")
            return False

        if self.current_account.withdraw(amount):
            self.cash_available -= amount
            transaction = Transaction(TransactionType.WITHDRAWAL, amount)
            print(f"✓ Withdrawal successful: ${amount:.2f}")
            print(f"  New balance: ${self.current_account.get_balance():.2f}")
            print(f"  {transaction}")
            return True
        else:
            print("Insufficient funds in account")
            return False

    def deposit(self, amount: float) -> bool:
        if self.state != ATMState.PIN_VERIFIED:
            print("Please verify PIN first")
            return False

        self.current_account.deposit(amount)
        self.cash_available += amount
        transaction = Transaction(TransactionType.DEPOSIT, amount)
        print(f"✓ Deposit successful: ${amount:.2f}")
        print(f"  New balance: ${self.current_account.get_balance():.2f}")
        print(f"  {transaction}")
        return True

    def check_balance(self) -> Optional[float]:
        if self.state != ATMState.PIN_VERIFIED:
            print("Please verify PIN first")
            return None

        balance = self.current_account.get_balance()
        print(f"Current balance: ${balance:.2f}")
        return balance

    def eject_card(self):
        if self.current_card:
            print(f"Card {self.current_card.card_number} ejected")

        self.current_card = None
        self.current_account = None
        self.state = ATMState.IDLE
        self.pin_attempts = 0


def main():
    # Create ATM
    atm = ATM("ATM-001", cash_available=5000.0)

    # Create account and card
    account = Account("123456789", balance=1000.0)
    card = Card("4532-1234-5678-9012", pin="1234")

    # Insert card
    atm.insert_card(card, account)

    # Wrong PIN
    atm.enter_pin("0000")

    # Correct PIN
    atm.enter_pin("1234")

    # Check balance
    atm.check_balance()

    # Withdraw
    atm.withdraw(200.0)

    # Deposit
    atm.deposit(500.0)

    # Check balance again
    atm.check_balance()

    # Eject card
    atm.eject_card()


if __name__ == "__main__":
    main()
```

---

## Design Patterns
- **State Pattern:** ATM state transitions
- **Command Pattern:** Transactions as commands (for undo/logging)
- **Strategy Pattern:** Different authentication methods

## SOLID Principles
- Clear state management with State pattern
- Transaction abstraction for different operation types

## Interview Tips
- Handle concurrent access (locking)
- Security considerations (PIN encryption, attempts)
- Transaction logging and audit trail
- Cash denomination management (distribute bills)

This tests state machine design and transactional operations.
