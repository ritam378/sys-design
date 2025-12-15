# Stock Trading System Design

## Overview

A **stock trading system** enables users to buy and sell securities with low latency, high throughput, and strong consistency guarantees.

Examples: Robinhood, E*TRADE, Interactive Brokers, Nasdaq

## Requirements

### Functional Requirements
1. **Place orders**: Market, limit, stop-loss orders
2. **Order matching**: Match buy/sell orders
3. **Portfolio management**: Track holdings, P&L
4. **Real-time quotes**: Live stock prices
5. **Order book**: View market depth
6. **Trade history**: Audit trail of all trades
7. **Risk management**: Margin checks, circuit breakers

### Non-Functional Requirements
1. **Latency**: < 10ms for order matching (ultra-low latency)
2. **Throughput**: 100,000+ orders per second
3. **Availability**: 99.999% during trading hours
4. **Consistency**: ACID guarantees for trades
5. **Fairness**: FIFO order matching (price-time priority)
6. **Compliance**: Audit trail, regulatory reporting

### Capacity Estimation

**Assumptions:**
- 10 million users
- 1 million active traders per day
- Average 10 orders per trader per day
- Peak load: 5x average
- Average order size: 200 bytes

**Throughput:**
- Orders per day: 1M × 10 = 10 million
- Orders per second (avg): 10M / 86400 = ~116 QPS
- Peak: 116 × 5 = ~580 QPS per symbol
- Total (100 symbols): 58,000 QPS

**Storage:**
- Orders per day: 10M × 200 bytes = 2 GB/day
- Trades (50% match rate): 5M × 500 bytes = 2.5 GB/day
- Total per day: ~5 GB
- Yearly: ~1.8 TB

**Latency Requirements:**
- Order placement to matching: < 10ms
- Market data updates: < 100ms
- Portfolio updates: < 1s

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    STOCK TRADING SYSTEM                       │
└──────────────────────────────────────────────────────────────┘

┌─────────────────┐         ┌─────────────────┐
│   Web/Mobile    │────────>│  Load Balancer  │
│     Clients     │         │    (Layer 7)    │
└─────────────────┘         └────────┬─────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
                    ▼                ▼                ▼
          ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
          │ API Gateway  │  │ WebSocket    │  │ Market Data  │
          │   (REST)     │  │   Server     │  │   Server     │
          └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
                 │                 │                  │
                 ▼                 ▼                  ▼
          ┌────────────────────────────────────────────────┐
          │          Application Services                  │
          │  - Order Service                               │
          │  - Matching Engine (per symbol)                │
          │  - Portfolio Service                           │
          │  - Risk Management Service                     │
          └────────┬──────────────────────┬────────────────┘
                   │                      │
                   ▼                      ▼
          ┌──────────────┐        ┌──────────────┐
          │   Primary    │───────>│   Replica    │
          │   Database   │        │   Database   │
          │  (PostgreSQL)│        │              │
          └──────┬───────┘        └──────────────┘
                 │
                 ▼
          ┌──────────────────────────────────────┐
          │       Message Queue (Kafka)          │
          │  - Order Events                      │
          │  - Trade Events                      │
          │  - Market Data Updates               │
          └──────┬───────────────────────────────┘
                 │
                 ▼
          ┌──────────────────────────────────────┐
          │      Background Workers              │
          │  - Settlement                        │
          │  - Reporting                         │
          │  - Analytics                         │
          └──────────────────────────────────────┘
```

## Core Components

### 1. Order Types

```python
from enum import Enum
from dataclasses import dataclass
from decimal import Decimal
from typing import Optional

class OrderSide(Enum):
    BUY = "buy"
    SELL = "sell"


class OrderType(Enum):
    MARKET = "market"      # Execute immediately at best price
    LIMIT = "limit"        # Execute at specified price or better
    STOP = "stop"          # Becomes market order when price reached
    STOP_LIMIT = "stop_limit"  # Becomes limit order when price reached


class OrderStatus(Enum):
    PENDING = "pending"
    OPEN = "open"
    PARTIALLY_FILLED = "partially_filled"
    FILLED = "filled"
    CANCELLED = "cancelled"
    REJECTED = "rejected"


@dataclass
class Order:
    """Represents a trading order"""
    order_id: str
    user_id: str
    symbol: str
    side: OrderSide
    order_type: OrderType
    quantity: int
    price: Optional[Decimal]  # None for market orders
    stop_price: Optional[Decimal]  # For stop orders
    filled_quantity: int = 0
    status: OrderStatus = OrderStatus.PENDING
    created_at: int = 0
    updated_at: int = 0

    def remaining_quantity(self) -> int:
        """Get unfilled quantity"""
        return self.quantity - self.filled_quantity

    def is_complete(self) -> bool:
        """Check if order is fully filled"""
        return self.filled_quantity >= self.quantity

    def average_fill_price(self, fills: list) -> Decimal:
        """Calculate average fill price"""
        if not fills:
            return Decimal(0)

        total_value = sum(f.price * f.quantity for f in fills)
        total_quantity = sum(f.quantity for f in fills)

        return total_value / total_quantity if total_quantity > 0 else Decimal(0)


@dataclass
class Trade:
    """Represents an executed trade"""
    trade_id: str
    buy_order_id: str
    sell_order_id: str
    symbol: str
    quantity: int
    price: Decimal
    buyer_id: str
    seller_id: str
    timestamp: int
```

### 2. Order Book (Matching Engine)

**Price-Time Priority Matching:**

```python
from collections import defaultdict
from sortedcontainers import SortedDict
import time
from decimal import Decimal
from typing import List, Optional

class OrderBook:
    """
    Order book with price-time priority matching

    Data structure:
    - Buy orders: Max heap (highest price first)
    - Sell orders: Min heap (lowest price first)
    - Same price level: FIFO queue
    """

    def __init__(self, symbol: str):
        self.symbol = symbol

        # Price level -> List of orders (FIFO)
        # Buy side: descending price order
        self.buy_orders = SortedDict(lambda x: -x)

        # Sell side: ascending price order
        self.sell_orders = SortedDict()

        # Order ID -> Order (for fast lookup)
        self.orders = {}

        # Last traded price
        self.last_price = Decimal(0)

        # Trade history
        self.trades = []

    def add_order(self, order: Order) -> List[Trade]:
        """
        Add order to book and attempt matching

        Returns list of trades generated
        """
        trades = []

        if order.order_type == OrderType.MARKET:
            # Match market order immediately
            trades = self._match_market_order(order)
        elif order.order_type == OrderType.LIMIT:
            # Try to match limit order
            trades = self._match_limit_order(order)

            # If not fully filled, add to book
            if not order.is_complete():
                self._add_to_book(order)

        # Update last price
        if trades:
            self.last_price = trades[-1].price

        return trades

    def cancel_order(self, order_id: str) -> bool:
        """Cancel order"""
        if order_id not in self.orders:
            return False

        order = self.orders[order_id]

        # Remove from book
        if order.side == OrderSide.BUY:
            if order.price in self.buy_orders:
                self.buy_orders[order.price] = [
                    o for o in self.buy_orders[order.price]
                    if o.order_id != order_id
                ]
                if not self.buy_orders[order.price]:
                    del self.buy_orders[order.price]
        else:
            if order.price in self.sell_orders:
                self.sell_orders[order.price] = [
                    o for o in self.sell_orders[order.price]
                    if o.order_id != order_id
                ]
                if not self.sell_orders[order.price]:
                    del self.sell_orders[order.price]

        # Remove from orders map
        del self.orders[order_id]

        order.status = OrderStatus.CANCELLED
        return True

    def _match_market_order(self, order: Order) -> List[Trade]:
        """
        Match market order against best available prices

        Market buy: match against lowest sell prices
        Market sell: match against highest buy prices
        """
        trades = []

        if order.side == OrderSide.BUY:
            # Match against sell orders (ascending price)
            while not order.is_complete() and self.sell_orders:
                best_price = self.sell_orders.keys()[0]
                trades.extend(self._match_at_price(
                    order,
                    self.sell_orders[best_price],
                    best_price
                ))

        else:  # SELL
            # Match against buy orders (descending price)
            while not order.is_complete() and self.buy_orders:
                best_price = self.buy_orders.keys()[0]
                trades.extend(self._match_at_price(
                    order,
                    self.buy_orders[best_price],
                    best_price
                ))

        return trades

    def _match_limit_order(self, order: Order) -> List[Trade]:
        """
        Match limit order

        Only match at specified price or better
        """
        trades = []

        if order.side == OrderSide.BUY:
            # Match against sell orders at or below limit price
            while not order.is_complete() and self.sell_orders:
                best_price = self.sell_orders.keys()[0]

                # Stop if price too high
                if best_price > order.price:
                    break

                trades.extend(self._match_at_price(
                    order,
                    self.sell_orders[best_price],
                    best_price
                ))

        else:  # SELL
            # Match against buy orders at or above limit price
            while not order.is_complete() and self.buy_orders:
                best_price = self.buy_orders.keys()[0]

                # Stop if price too low
                if best_price < order.price:
                    break

                trades.extend(self._match_at_price(
                    order,
                    self.buy_orders[best_price],
                    best_price
                ))

        return trades

    def _match_at_price(self, incoming_order: Order,
                       price_level_orders: List[Order],
                       price: Decimal) -> List[Trade]:
        """
        Match incoming order against orders at specific price level

        FIFO matching within price level
        """
        trades = []

        # Match against orders in FIFO order
        orders_to_remove = []

        for i, resting_order in enumerate(price_level_orders):
            if incoming_order.is_complete():
                break

            # Calculate trade quantity
            trade_quantity = min(
                incoming_order.remaining_quantity(),
                resting_order.remaining_quantity()
            )

            # Create trade
            trade = self._create_trade(
                incoming_order,
                resting_order,
                trade_quantity,
                price
            )

            trades.append(trade)
            self.trades.append(trade)

            # Update order fill quantities
            incoming_order.filled_quantity += trade_quantity
            resting_order.filled_quantity += trade_quantity

            # Update statuses
            if incoming_order.is_complete():
                incoming_order.status = OrderStatus.FILLED

            if resting_order.is_complete():
                resting_order.status = OrderStatus.FILLED
                orders_to_remove.append(i)
            else:
                resting_order.status = OrderStatus.PARTIALLY_FILLED

        # Remove filled orders from price level
        for i in reversed(orders_to_remove):
            filled_order = price_level_orders.pop(i)
            del self.orders[filled_order.order_id]

        # Remove price level if empty
        if not price_level_orders:
            if incoming_order.side == OrderSide.BUY:
                del self.sell_orders[price]
            else:
                del self.buy_orders[price]

        return trades

    def _add_to_book(self, order: Order):
        """Add order to order book"""
        if order.side == OrderSide.BUY:
            if order.price not in self.buy_orders:
                self.buy_orders[order.price] = []
            self.buy_orders[order.price].append(order)
        else:
            if order.price not in self.sell_orders:
                self.sell_orders[order.price] = []
            self.sell_orders[order.price].append(order)

        # Add to orders map
        self.orders[order.order_id] = order
        order.status = OrderStatus.OPEN

    def _create_trade(self, buy_order: Order, sell_order: Order,
                     quantity: int, price: Decimal) -> Trade:
        """Create trade from matched orders"""
        import uuid

        # Determine which is buyer and which is seller
        if buy_order.side == OrderSide.BUY:
            buyer_id = buy_order.user_id
            seller_id = sell_order.user_id
            buy_order_id = buy_order.order_id
            sell_order_id = sell_order.order_id
        else:
            buyer_id = sell_order.user_id
            seller_id = buy_order.user_id
            buy_order_id = sell_order.order_id
            sell_order_id = buy_order.order_id

        return Trade(
            trade_id=str(uuid.uuid4()),
            buy_order_id=buy_order_id,
            sell_order_id=sell_order_id,
            symbol=self.symbol,
            quantity=quantity,
            price=price,
            buyer_id=buyer_id,
            seller_id=seller_id,
            timestamp=int(time.time() * 1_000_000)  # microseconds
        )

    def get_market_depth(self, levels: int = 10) -> dict:
        """
        Get market depth (order book snapshot)

        Returns top N price levels for buy and sell
        """
        buy_depth = []
        for i, (price, orders) in enumerate(self.buy_orders.items()):
            if i >= levels:
                break
            total_quantity = sum(o.remaining_quantity() for o in orders)
            buy_depth.append({
                'price': float(price),
                'quantity': total_quantity,
                'orders': len(orders)
            })

        sell_depth = []
        for i, (price, orders) in enumerate(self.sell_orders.items()):
            if i >= levels:
                break
            total_quantity = sum(o.remaining_quantity() for o in orders)
            sell_depth.append({
                'price': float(price),
                'quantity': total_quantity,
                'orders': len(orders)
            })

        return {
            'symbol': self.symbol,
            'buy': buy_depth,
            'sell': sell_depth,
            'last_price': float(self.last_price)
        }

    def get_best_bid(self) -> Optional[Decimal]:
        """Get best buy price"""
        if self.buy_orders:
            return self.buy_orders.keys()[0]
        return None

    def get_best_ask(self) -> Optional[Decimal]:
        """Get best sell price"""
        if self.sell_orders:
            return self.sell_orders.keys()[0]
        return None

    def get_spread(self) -> Optional[Decimal]:
        """Get bid-ask spread"""
        bid = self.get_best_bid()
        ask = self.get_best_ask()

        if bid and ask:
            return ask - bid
        return None
```

### 3. Order Service

**Handle Order Lifecycle:**

```python
import uuid
from typing import List, Optional
from decimal import Decimal

class OrderService:
    """
    Manage order lifecycle

    - Validate orders
    - Route to matching engine
    - Update portfolio
    - Persist orders and trades
    """

    def __init__(self, db, cache, order_books, portfolio_service,
                 risk_service, event_bus):
        self.db = db
        self.cache = cache
        self.order_books = order_books  # symbol -> OrderBook
        self.portfolio_service = portfolio_service
        self.risk_service = risk_service
        self.event_bus = event_bus

    def place_order(self, user_id: str, symbol: str, side: OrderSide,
                   order_type: OrderType, quantity: int,
                   price: Optional[Decimal] = None) -> Order:
        """
        Place trading order

        Steps:
        1. Validate order
        2. Check risk limits
        3. Route to matching engine
        4. Update portfolio
        5. Persist order and trades
        """
        # Validate order
        self._validate_order(user_id, symbol, side, order_type, quantity, price)

        # Pre-trade risk checks
        if not self.risk_service.check_order(user_id, symbol, side, quantity, price):
            raise ValueError("Order rejected by risk management")

        # Create order
        order = Order(
            order_id=str(uuid.uuid4()),
            user_id=user_id,
            symbol=symbol,
            side=side,
            order_type=order_type,
            quantity=quantity,
            price=price,
            stop_price=None,
            created_at=int(time.time() * 1_000_000)
        )

        # Reserve funds/shares
        if side == OrderSide.BUY:
            # Reserve cash
            required_amount = quantity * price if price else Decimal(0)
            self.portfolio_service.reserve_cash(user_id, required_amount)
        else:
            # Reserve shares
            self.portfolio_service.reserve_shares(user_id, symbol, quantity)

        # Route to matching engine
        order_book = self._get_order_book(symbol)
        trades = order_book.add_order(order)

        # Process trades
        for trade in trades:
            self._process_trade(trade)

        # Persist order
        self._persist_order(order)

        # Publish event
        self.event_bus.publish('order.placed', {
            'order_id': order.order_id,
            'user_id': user_id,
            'symbol': symbol,
            'side': side.value,
            'quantity': quantity,
            'price': float(price) if price else None
        })

        return order

    def cancel_order(self, order_id: str, user_id: str) -> bool:
        """Cancel pending order"""
        # Get order
        order = self.get_order(order_id)

        if not order:
            raise ValueError("Order not found")

        # Verify ownership
        if order.user_id != user_id:
            raise ValueError("Unauthorized")

        # Check if cancellable
        if order.status not in [OrderStatus.OPEN, OrderStatus.PARTIALLY_FILLED]:
            raise ValueError(f"Cannot cancel order in {order.status} state")

        # Cancel in matching engine
        order_book = self._get_order_book(order.symbol)
        success = order_book.cancel_order(order_id)

        if success:
            # Release reserved funds/shares
            if order.side == OrderSide.BUY:
                unfilled_amount = order.remaining_quantity() * order.price
                self.portfolio_service.release_cash(user_id, unfilled_amount)
            else:
                self.portfolio_service.release_shares(
                    user_id,
                    order.symbol,
                    order.remaining_quantity()
                )

            # Update in database
            self._update_order_status(order_id, OrderStatus.CANCELLED)

            # Publish event
            self.event_bus.publish('order.cancelled', {
                'order_id': order_id,
                'user_id': user_id
            })

        return success

    def get_order(self, order_id: str) -> Optional[Order]:
        """Get order by ID"""
        # Try cache
        cache_key = f"order:{order_id}"
        cached = self.cache.get(cache_key)
        if cached:
            return self._deserialize_order(cached)

        # Query database
        result = self.db.query_one(
            "SELECT * FROM orders WHERE order_id = %s",
            (order_id,)
        )

        if not result:
            return None

        order = self._row_to_order(result)

        # Cache
        self.cache.set(cache_key, self._serialize_order(order), ttl=300)

        return order

    def get_user_orders(self, user_id: str, status: OrderStatus = None,
                       limit: int = 100) -> List[Order]:
        """Get user's orders"""
        query = "SELECT * FROM orders WHERE user_id = %s"
        params = [user_id]

        if status:
            query += " AND status = %s"
            params.append(status.value)

        query += " ORDER BY created_at DESC LIMIT %s"
        params.append(limit)

        results = self.db.query(query, params)

        return [self._row_to_order(row) for row in results]

    def _validate_order(self, user_id: str, symbol: str, side: OrderSide,
                       order_type: OrderType, quantity: int,
                       price: Optional[Decimal]):
        """Validate order parameters"""
        # Quantity must be positive
        if quantity <= 0:
            raise ValueError("Quantity must be positive")

        # Limit orders must have price
        if order_type == OrderType.LIMIT and price is None:
            raise ValueError("Limit order must specify price")

        # Price must be positive
        if price is not None and price <= 0:
            raise ValueError("Price must be positive")

        # Check trading hours
        if not self._is_market_open():
            raise ValueError("Market is closed")

        # Check if symbol exists
        if not self._symbol_exists(symbol):
            raise ValueError(f"Invalid symbol: {symbol}")

    def _process_trade(self, trade: Trade):
        """
        Process executed trade

        - Update portfolios
        - Settle trade
        - Record transaction
        """
        # Update buyer portfolio
        self.portfolio_service.add_position(
            trade.buyer_id,
            trade.symbol,
            trade.quantity,
            trade.price
        )

        # Update seller portfolio
        self.portfolio_service.remove_position(
            trade.seller_id,
            trade.symbol,
            trade.quantity,
            trade.price
        )

        # Persist trade
        self._persist_trade(trade)

        # Publish event
        self.event_bus.publish('trade.executed', {
            'trade_id': trade.trade_id,
            'symbol': trade.symbol,
            'quantity': trade.quantity,
            'price': float(trade.price),
            'timestamp': trade.timestamp
        })

    def _get_order_book(self, symbol: str) -> OrderBook:
        """Get or create order book for symbol"""
        if symbol not in self.order_books:
            self.order_books[symbol] = OrderBook(symbol)
        return self.order_books[symbol]

    def _persist_order(self, order: Order):
        """Persist order to database"""
        self.db.execute(
            """
            INSERT INTO orders
            (order_id, user_id, symbol, side, order_type, quantity, price,
             filled_quantity, status, created_at, updated_at)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            """,
            (order.order_id, order.user_id, order.symbol, order.side.value,
             order.order_type.value, order.quantity, order.price,
             order.filled_quantity, order.status.value, order.created_at,
             order.updated_at)
        )

    def _persist_trade(self, trade: Trade):
        """Persist trade to database"""
        self.db.execute(
            """
            INSERT INTO trades
            (trade_id, buy_order_id, sell_order_id, symbol, quantity, price,
             buyer_id, seller_id, timestamp)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
            """,
            (trade.trade_id, trade.buy_order_id, trade.sell_order_id,
             trade.symbol, trade.quantity, trade.price, trade.buyer_id,
             trade.seller_id, trade.timestamp)
        )

    def _update_order_status(self, order_id: str, status: OrderStatus):
        """Update order status"""
        self.db.execute(
            """
            UPDATE orders
            SET status = %s, updated_at = %s
            WHERE order_id = %s
            """,
            (status.value, int(time.time() * 1_000_000), order_id)
        )

        # Invalidate cache
        self.cache.delete(f"order:{order_id}")

    def _is_market_open(self) -> bool:
        """Check if market is open"""
        # Simplified - check trading hours
        # In production, check exchange calendar
        from datetime import datetime

        now = datetime.now()
        # Example: 9:30 AM - 4:00 PM EST
        if now.weekday() >= 5:  # Weekend
            return False

        hour = now.hour
        return 9 <= hour < 16

    def _symbol_exists(self, symbol: str) -> bool:
        """Check if symbol is tradeable"""
        # Query symbols table
        result = self.db.query_one(
            "SELECT 1 FROM symbols WHERE symbol = %s",
            (symbol,)
        )
        return result is not None

    def _row_to_order(self, row: dict) -> Order:
        """Convert database row to Order object"""
        return Order(
            order_id=row['order_id'],
            user_id=row['user_id'],
            symbol=row['symbol'],
            side=OrderSide(row['side']),
            order_type=OrderType(row['order_type']),
            quantity=row['quantity'],
            price=row['price'],
            stop_price=row.get('stop_price'),
            filled_quantity=row['filled_quantity'],
            status=OrderStatus(row['status']),
            created_at=row['created_at'],
            updated_at=row['updated_at']
        )

    def _serialize_order(self, order: Order) -> dict:
        """Serialize order for caching"""
        return {
            'order_id': order.order_id,
            'user_id': order.user_id,
            'symbol': order.symbol,
            'side': order.side.value,
            'order_type': order.order_type.value,
            'quantity': order.quantity,
            'price': float(order.price) if order.price else None,
            'filled_quantity': order.filled_quantity,
            'status': order.status.value,
            'created_at': order.created_at
        }

    def _deserialize_order(self, data: dict) -> Order:
        """Deserialize order from cache"""
        return Order(
            order_id=data['order_id'],
            user_id=data['user_id'],
            symbol=data['symbol'],
            side=OrderSide(data['side']),
            order_type=OrderType(data['order_type']),
            quantity=data['quantity'],
            price=Decimal(str(data['price'])) if data['price'] else None,
            filled_quantity=data['filled_quantity'],
            status=OrderStatus(data['status']),
            created_at=data['created_at']
        )
```

### 4. Portfolio Service

**Track User Holdings:**

```python
from typing import Dict, List
from decimal import Decimal

class Position:
    """Represents a stock position"""

    def __init__(self, symbol: str, quantity: int, average_price: Decimal):
        self.symbol = symbol
        self.quantity = quantity
        self.average_price = average_price

    def market_value(self, current_price: Decimal) -> Decimal:
        """Calculate current market value"""
        return self.quantity * current_price

    def unrealized_pnl(self, current_price: Decimal) -> Decimal:
        """Calculate unrealized profit/loss"""
        return (current_price - self.average_price) * self.quantity

    def cost_basis(self) -> Decimal:
        """Calculate total cost basis"""
        return self.quantity * self.average_price


class Portfolio:
    """User's trading portfolio"""

    def __init__(self, user_id: str, cash_balance: Decimal):
        self.user_id = user_id
        self.cash_balance = cash_balance
        self.reserved_cash = Decimal(0)
        self.positions: Dict[str, Position] = {}
        self.reserved_shares: Dict[str, int] = {}

    def available_cash(self) -> Decimal:
        """Get available cash (not reserved)"""
        return self.cash_balance - self.reserved_cash

    def total_value(self, market_prices: Dict[str, Decimal]) -> Decimal:
        """Calculate total portfolio value"""
        positions_value = sum(
            pos.market_value(market_prices.get(symbol, Decimal(0)))
            for symbol, pos in self.positions.items()
        )

        return self.cash_balance + positions_value

    def total_pnl(self, market_prices: Dict[str, Decimal]) -> Decimal:
        """Calculate total unrealized P&L"""
        return sum(
            pos.unrealized_pnl(market_prices.get(symbol, Decimal(0)))
            for pos in self.positions.values()
        )


class PortfolioService:
    """
    Manage user portfolios

    - Track positions
    - Calculate P&L
    - Handle cash and share reservations
    """

    def __init__(self, db, cache, market_data_service):
        self.db = db
        self.cache = cache
        self.market_data_service = market_data_service

    def get_portfolio(self, user_id: str) -> Portfolio:
        """Get user's portfolio"""
        # Try cache
        cache_key = f"portfolio:{user_id}"
        cached = self.cache.get(cache_key)
        if cached:
            return self._deserialize_portfolio(cached)

        # Query database
        portfolio = self._load_portfolio_from_db(user_id)

        # Cache
        self.cache.set(cache_key, self._serialize_portfolio(portfolio), ttl=60)

        return portfolio

    def add_position(self, user_id: str, symbol: str, quantity: int,
                    price: Decimal):
        """
        Add to position (buy)

        Update average price using weighted average
        """
        portfolio = self.get_portfolio(user_id)

        # Calculate cost
        cost = quantity * price

        # Update cash
        portfolio.cash_balance -= cost
        portfolio.reserved_cash -= cost

        # Update position
        if symbol in portfolio.positions:
            pos = portfolio.positions[symbol]

            # Weighted average price
            total_quantity = pos.quantity + quantity
            total_cost = pos.cost_basis() + cost
            pos.average_price = total_cost / total_quantity
            pos.quantity = total_quantity
        else:
            # New position
            portfolio.positions[symbol] = Position(symbol, quantity, price)

        # Persist
        self._save_portfolio(portfolio)

    def remove_position(self, user_id: str, symbol: str, quantity: int,
                       price: Decimal):
        """
        Remove from position (sell)
        """
        portfolio = self.get_portfolio(user_id)

        if symbol not in portfolio.positions:
            raise ValueError(f"No position in {symbol}")

        pos = portfolio.positions[symbol]

        if pos.quantity < quantity:
            raise ValueError(f"Insufficient shares: have {pos.quantity}, need {quantity}")

        # Update position
        pos.quantity -= quantity

        # Remove position if fully sold
        if pos.quantity == 0:
            del portfolio.positions[symbol]

        # Release reserved shares
        portfolio.reserved_shares[symbol] = \
            portfolio.reserved_shares.get(symbol, 0) - quantity

        # Add proceeds to cash
        proceeds = quantity * price
        portfolio.cash_balance += proceeds

        # Persist
        self._save_portfolio(portfolio)

    def reserve_cash(self, user_id: str, amount: Decimal):
        """Reserve cash for pending buy order"""
        portfolio = self.get_portfolio(user_id)

        if portfolio.available_cash() < amount:
            raise ValueError("Insufficient funds")

        portfolio.reserved_cash += amount

        self._save_portfolio(portfolio)

    def release_cash(self, user_id: str, amount: Decimal):
        """Release reserved cash (order cancelled)"""
        portfolio = self.get_portfolio(user_id)

        portfolio.reserved_cash = max(Decimal(0), portfolio.reserved_cash - amount)

        self._save_portfolio(portfolio)

    def reserve_shares(self, user_id: str, symbol: str, quantity: int):
        """Reserve shares for pending sell order"""
        portfolio = self.get_portfolio(user_id)

        if symbol not in portfolio.positions:
            raise ValueError(f"No position in {symbol}")

        pos = portfolio.positions[symbol]
        reserved = portfolio.reserved_shares.get(symbol, 0)
        available = pos.quantity - reserved

        if available < quantity:
            raise ValueError(f"Insufficient shares: have {available}, need {quantity}")

        portfolio.reserved_shares[symbol] = reserved + quantity

        self._save_portfolio(portfolio)

    def release_shares(self, user_id: str, symbol: str, quantity: int):
        """Release reserved shares (order cancelled)"""
        portfolio = self.get_portfolio(user_id)

        reserved = portfolio.reserved_shares.get(symbol, 0)
        portfolio.reserved_shares[symbol] = max(0, reserved - quantity)

        self._save_portfolio(portfolio)

    def get_position(self, user_id: str, symbol: str) -> Optional[Position]:
        """Get position for specific symbol"""
        portfolio = self.get_portfolio(user_id)
        return portfolio.positions.get(symbol)

    def _load_portfolio_from_db(self, user_id: str) -> Portfolio:
        """Load portfolio from database"""
        # Get cash balance
        user_result = self.db.query_one(
            "SELECT cash_balance, reserved_cash FROM users WHERE user_id = %s",
            (user_id,)
        )

        if not user_result:
            raise ValueError(f"User not found: {user_id}")

        portfolio = Portfolio(
            user_id=user_id,
            cash_balance=user_result['cash_balance']
        )
        portfolio.reserved_cash = user_result.get('reserved_cash', Decimal(0))

        # Get positions
        positions = self.db.query(
            """
            SELECT symbol, quantity, average_price
            FROM positions
            WHERE user_id = %s AND quantity > 0
            """,
            (user_id,)
        )

        for row in positions:
            portfolio.positions[row['symbol']] = Position(
                symbol=row['symbol'],
                quantity=row['quantity'],
                average_price=row['average_price']
            )

        # Get reserved shares
        reserved = self.db.query(
            """
            SELECT symbol, SUM(quantity) as reserved
            FROM orders
            WHERE user_id = %s
              AND side = 'sell'
              AND status IN ('open', 'partially_filled')
            GROUP BY symbol
            """,
            (user_id,)
        )

        for row in reserved:
            portfolio.reserved_shares[row['symbol']] = row['reserved']

        return portfolio

    def _save_portfolio(self, portfolio: Portfolio):
        """Save portfolio to database"""
        # Update cash balance
        self.db.execute(
            """
            UPDATE users
            SET cash_balance = %s, reserved_cash = %s
            WHERE user_id = %s
            """,
            (portfolio.cash_balance, portfolio.reserved_cash, portfolio.user_id)
        )

        # Update positions
        for symbol, pos in portfolio.positions.items():
            self.db.execute(
                """
                INSERT INTO positions (user_id, symbol, quantity, average_price)
                VALUES (%s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE
                    quantity = VALUES(quantity),
                    average_price = VALUES(average_price)
                """,
                (portfolio.user_id, symbol, pos.quantity, pos.average_price)
            )

        # Invalidate cache
        self.cache.delete(f"portfolio:{portfolio.user_id}")

    def _serialize_portfolio(self, portfolio: Portfolio) -> dict:
        """Serialize portfolio for caching"""
        return {
            'user_id': portfolio.user_id,
            'cash_balance': float(portfolio.cash_balance),
            'reserved_cash': float(portfolio.reserved_cash),
            'positions': {
                symbol: {
                    'quantity': pos.quantity,
                    'average_price': float(pos.average_price)
                }
                for symbol, pos in portfolio.positions.items()
            },
            'reserved_shares': portfolio.reserved_shares
        }

    def _deserialize_portfolio(self, data: dict) -> Portfolio:
        """Deserialize portfolio from cache"""
        portfolio = Portfolio(
            user_id=data['user_id'],
            cash_balance=Decimal(str(data['cash_balance']))
        )
        portfolio.reserved_cash = Decimal(str(data['reserved_cash']))

        for symbol, pos_data in data['positions'].items():
            portfolio.positions[symbol] = Position(
                symbol=symbol,
                quantity=pos_data['quantity'],
                average_price=Decimal(str(pos_data['average_price']))
            )

        portfolio.reserved_shares = data['reserved_shares']

        return portfolio
```

### 5. Risk Management Service

**Pre-trade and Post-trade Checks:**

```python
from decimal import Decimal
from typing import Optional

class RiskService:
    """
    Risk management and compliance

    - Position limits
    - Margin requirements
    - Circuit breakers
    - Pattern day trading checks
    """

    def __init__(self, db, portfolio_service, market_data_service):
        self.db = db
        self.portfolio_service = portfolio_service
        self.market_data_service = market_data_service

        # Risk limits
        self.max_position_size = 10000  # shares
        self.max_order_value = Decimal(100000)  # dollars
        self.max_daily_trades = 100
        self.margin_requirement = Decimal(0.25)  # 25% initial margin

    def check_order(self, user_id: str, symbol: str, side: OrderSide,
                   quantity: int, price: Optional[Decimal]) -> bool:
        """
        Pre-trade risk checks

        Returns True if order passes all checks
        """
        # Check position limit
        if not self._check_position_limit(user_id, symbol, side, quantity):
            return False

        # Check order value limit
        if not self._check_order_value(quantity, price):
            return False

        # Check buying power
        if side == OrderSide.BUY:
            if not self._check_buying_power(user_id, quantity, price):
                return False

        # Check daily trade limit
        if not self._check_daily_trade_limit(user_id):
            return False

        # Check pattern day trading
        if not self._check_pattern_day_trading(user_id):
            return False

        return True

    def _check_position_limit(self, user_id: str, symbol: str,
                             side: OrderSide, quantity: int) -> bool:
        """Check position size limit"""
        position = self.portfolio_service.get_position(user_id, symbol)

        if side == OrderSide.BUY:
            current_quantity = position.quantity if position else 0
            new_quantity = current_quantity + quantity

            if new_quantity > self.max_position_size:
                return False

        return True

    def _check_order_value(self, quantity: int,
                          price: Optional[Decimal]) -> bool:
        """Check single order value limit"""
        if price is None:
            return True  # Market order - can't check

        order_value = quantity * price

        return order_value <= self.max_order_value

    def _check_buying_power(self, user_id: str, quantity: int,
                           price: Optional[Decimal]) -> bool:
        """Check if user has sufficient buying power"""
        if price is None:
            # For market orders, use conservative estimate
            price = self.market_data_service.get_last_price(symbol)

        required_amount = quantity * price

        # Check for margin account
        is_margin = self._is_margin_account(user_id)

        if is_margin:
            # Margin account: only need margin_requirement % cash
            required_cash = required_amount * self.margin_requirement
        else:
            # Cash account: need full amount
            required_cash = required_amount

        portfolio = self.portfolio_service.get_portfolio(user_id)

        return portfolio.available_cash() >= required_cash

    def _check_daily_trade_limit(self, user_id: str) -> bool:
        """Check daily trade count limit"""
        # Count trades today
        from datetime import datetime

        today_start = datetime.now().replace(
            hour=0, minute=0, second=0, microsecond=0
        ).timestamp()

        result = self.db.query_one(
            """
            SELECT COUNT(*) as count
            FROM trades
            WHERE (buyer_id = %s OR seller_id = %s)
              AND timestamp >= %s
            """,
            (user_id, user_id, int(today_start * 1_000_000))
        )

        count = result['count'] if result else 0

        return count < self.max_daily_trades

    def _check_pattern_day_trading(self, user_id: str) -> bool:
        """
        Check pattern day trading rule

        Pattern day trader: 4+ day trades in 5 business days
        Requires $25,000 minimum account balance
        """
        # Count day trades in last 5 business days
        day_trades = self._count_day_trades(user_id, days=5)

        if day_trades >= 4:
            # Check account balance
            portfolio = self.portfolio_service.get_portfolio(user_id)
            market_prices = {}  # Get current prices
            total_value = portfolio.total_value(market_prices)

            if total_value < Decimal(25000):
                return False

        return True

    def _is_margin_account(self, user_id: str) -> bool:
        """Check if user has margin account"""
        result = self.db.query_one(
            "SELECT margin_enabled FROM users WHERE user_id = %s",
            (user_id,)
        )

        return result['margin_enabled'] if result else False

    def _count_day_trades(self, user_id: str, days: int) -> int:
        """Count day trades (buy and sell same stock same day)"""
        # Simplified implementation
        # In production, track day trades properly
        return 0
```

## Database Schema

```sql
-- Users/Accounts table
CREATE TABLE users (
    user_id VARCHAR(64) PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    cash_balance DECIMAL(20, 2) DEFAULT 0,
    reserved_cash DECIMAL(20, 2) DEFAULT 0,
    margin_enabled BOOLEAN DEFAULT FALSE,
    created_at BIGINT NOT NULL,
    INDEX idx_email (email)
);

-- Symbols (tradeable securities)
CREATE TABLE symbols (
    symbol VARCHAR(10) PRIMARY KEY,
    name VARCHAR(255),
    exchange VARCHAR(20),
    sector VARCHAR(50),
    active BOOLEAN DEFAULT TRUE
);

-- Orders table
CREATE TABLE orders (
    order_id VARCHAR(64) PRIMARY KEY,
    user_id VARCHAR(64) NOT NULL,
    symbol VARCHAR(10) NOT NULL,
    side ENUM('buy', 'sell') NOT NULL,
    order_type ENUM('market', 'limit', 'stop', 'stop_limit') NOT NULL,
    quantity INT NOT NULL,
    price DECIMAL(20, 4),
    stop_price DECIMAL(20, 4),
    filled_quantity INT DEFAULT 0,
    status ENUM('pending', 'open', 'partially_filled', 'filled', 'cancelled', 'rejected') NOT NULL,
    created_at BIGINT NOT NULL,
    updated_at BIGINT NOT NULL,
    INDEX idx_user_time (user_id, created_at),
    INDEX idx_symbol_status (symbol, status),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (symbol) REFERENCES symbols(symbol)
) PARTITION BY HASH(user_id) PARTITIONS 100;

-- Trades table
CREATE TABLE trades (
    trade_id VARCHAR(64) PRIMARY KEY,
    buy_order_id VARCHAR(64) NOT NULL,
    sell_order_id VARCHAR(64) NOT NULL,
    symbol VARCHAR(10) NOT NULL,
    quantity INT NOT NULL,
    price DECIMAL(20, 4) NOT NULL,
    buyer_id VARCHAR(64) NOT NULL,
    seller_id VARCHAR(64) NOT NULL,
    timestamp BIGINT NOT NULL,
    INDEX idx_symbol_time (symbol, timestamp),
    INDEX idx_buyer_time (buyer_id, timestamp),
    INDEX idx_seller_time (seller_id, timestamp),
    INDEX idx_timestamp (timestamp),
    FOREIGN KEY (buy_order_id) REFERENCES orders(order_id),
    FOREIGN KEY (sell_order_id) REFERENCES orders(order_id),
    FOREIGN KEY (symbol) REFERENCES symbols(symbol)
) PARTITION BY RANGE(timestamp) (
    -- Partition by day/week
    PARTITION p_2024_01 VALUES LESS THAN (1706745600000000),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Positions table
CREATE TABLE positions (
    user_id VARCHAR(64),
    symbol VARCHAR(10),
    quantity INT NOT NULL,
    average_price DECIMAL(20, 4) NOT NULL,
    updated_at BIGINT,
    PRIMARY KEY (user_id, symbol),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (symbol) REFERENCES symbols(symbol)
) PARTITION BY HASH(user_id) PARTITIONS 100;

-- Market data (OHLCV)
CREATE TABLE market_data (
    symbol VARCHAR(10),
    timestamp BIGINT,
    open DECIMAL(20, 4),
    high DECIMAL(20, 4),
    low DECIMAL(20, 4),
    close DECIMAL(20, 4),
    volume BIGINT,
    PRIMARY KEY (symbol, timestamp),
    INDEX idx_symbol_time (symbol, timestamp)
) PARTITION BY HASH(symbol) PARTITIONS 50;
```

## Performance Optimizations

### 1. In-Memory Matching Engine

**Use memory-mapped structures:**
- Lock-free data structures
- CPU cache optimization
- NUMA-aware allocation

### 2. Low-Latency Networking

**Bypass kernel networking:**
- Use DPDK (Data Plane Development Kit)
- Kernel bypass with RDMA
- Direct NIC-to-application communication

### 3. Database Optimizations

**Write-Ahead Log (WAL):**
- Batch writes
- Async replication
- SSD optimization

**Time-series optimization:**
- Time-based partitioning
- Compression (delta encoding)
- Hot/cold data separation

## Interview Tips

### Common Questions

**Q: How to achieve low latency (< 10ms)?**
- In-memory matching engine
- Co-location (servers near exchange)
- Optimized data structures (sorted containers)
- Lock-free algorithms
- Kernel bypass networking

**Q: How to ensure ACID properties?**
- Use database transactions
- Two-phase commit for distributed transactions
- Write-Ahead Logging (WAL)
- Idempotency for retry safety

**Q: How to handle high throughput?**
- Horizontal scaling (one matching engine per symbol)
- Sharding by symbol
- Async processing with message queues
- Batch operations where possible

**Q: How to prevent race conditions in matching?**
- Single-threaded matching engine per symbol
- Lock-free queues for incoming orders
- Optimistic locking for database updates

**Q: How to implement circuit breakers?**
- Track price movements per symbol
- Halt trading if price moves > threshold (e.g., 10%) in short time
- Require manual intervention to resume
- Regulatory requirement to prevent flash crashes

## Key Takeaways

1. **Low Latency**: In-memory matching, optimized data structures, co-location
2. **Consistency**: ACID transactions, idempotency, audit trails
3. **Fairness**: Price-time priority matching (FIFO at same price)
4. **Risk Management**: Pre-trade checks, position limits, margin requirements
5. **Scalability**: Shard by symbol, separate matching engines
6. **Compliance**: Audit logs, circuit breakers, regulatory reporting

Building a stock trading system requires expertise in low-latency systems, concurrency, and financial regulations!
