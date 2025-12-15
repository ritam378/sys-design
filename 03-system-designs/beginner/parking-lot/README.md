# Parking Lot System Design

A comprehensive guide to designing a scalable parking lot management system for multi-level parking facilities.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Requirements](#2-requirements)
3. [Back-of-the-Envelope Estimation](#3-back-of-the-envelope-estimation)
4. [API Design](#4-api-design)
5. [Data Model & Database Schema](#5-data-model--database-schema)
6. [High-Level Design](#6-high-level-design)
7. [Detailed Component Design](#7-detailed-component-design)
8. [Identifying and Resolving Bottlenecks](#8-identifying-and-resolving-bottlenecks)
9. [Trade-offs and Alternatives](#9-trade-offs-and-alternatives)
10. [Monitoring, Metrics & Alerts](#10-monitoring-metrics--alerts)
11. [Follow-up Questions & Extensions](#11-follow-up-questions--extensions)
12. [Code Implementation](#12-code-implementation)
13. [References](#13-references)

---

## 1. Problem Statement

Design a parking lot management system that can:
- Track available parking spots across multiple floors and spot types
- Handle vehicle entry and exit operations
- Calculate parking fees based on duration
- Support different vehicle types (motorcycle, car, truck, electric vehicle)
- Provide real-time availability information
- Generate parking tickets with unique identifiers
- Handle concurrent parking operations

**Real-world Examples:**
- Shopping mall parking systems
- Airport parking facilities
- Office building parking garages
- Smart city parking solutions

---

## 2. Requirements

### Functional Requirements

1. **Vehicle Entry**
   - Issue parking tickets with unique IDs
   - Assign appropriate parking spot based on vehicle type
   - Record entry timestamp
   - Update spot availability

2. **Vehicle Exit**
   - Calculate parking fee based on duration
   - Process payment
   - Mark spot as available
   - Record exit timestamp

3. **Spot Management**
   - Support multiple spot types (compact, regular, large, electric)
   - Track spot status (available, occupied, reserved, maintenance)
   - Support multiple floors/levels
   - Handle spot reservations

4. **Pricing**
   - Hourly rate pricing
   - Different rates for different vehicle types
   - Support for daily/monthly passes
   - Peak hour pricing (optional)

5. **Availability Display**
   - Real-time available spots per floor
   - Available spots per vehicle type
   - Estimated parking time remaining

### Non-Functional Requirements

1. **Scalability**
   - Support 1,000+ parking spots
   - Handle 100+ concurrent entry/exit operations
   - Scale across multiple parking facilities

2. **Availability**
   - 99.9% uptime
   - System should degrade gracefully
   - Entry/exit should work even if payment system is down

3. **Consistency**
   - Strong consistency for spot assignments (no double-booking)
   - Eventually consistent for availability displays

4. **Performance**
   - Entry processing: < 2 seconds
   - Exit processing: < 5 seconds
   - Availability query: < 500ms

5. **Security**
   - Ticket validation to prevent fraud
   - Payment security compliance (PCI DSS)
   - Access control for admin operations

### Out of Scope

- Automated parking (robotic systems)
- License plate recognition (can be added as extension)
- Mobile app development (focus on backend)
- Valet parking management

---

## 3. Back-of-the-Envelope Estimation

### Assumptions

- **Parking Facility Size:**
  - 5 floors
  - 200 spots per floor
  - Total: 1,000 parking spots

- **Spot Distribution:**
  - Compact: 30% (300 spots)
  - Regular: 50% (500 spots)
  - Large: 15% (150 spots)
  - Electric: 5% (50 spots)

- **Usage Patterns:**
  - Average occupancy: 70%
  - Average parking duration: 3 hours
  - Peak occupancy: 90% (weekends)
  - Daily turnover: 2.5x (2,500 vehicles/day)

### Traffic Estimates

**Daily Operations:**
- Entries per day: 2,500
- Exits per day: 2,500
- Total operations: 5,000/day

**Peak Load:**
- Peak hour entries: 250 (10% of daily in 1 hour)
- Peak hour exits: 250
- Operations per second: ~0.14 ops/sec (average)
- Peak ops/sec: ~1 op/sec

**QPS (Queries Per Second):**
- Entry API: 1 QPS (peak)
- Exit API: 1 QPS (peak)
- Availability check: 10 QPS (many users checking)
- Total: ~15 QPS (very manageable)

### Storage Estimates

**Ticket Records:**
- Ticket size: ~500 bytes (ID, vehicle info, timestamps, spot)
- Daily tickets: 2,500
- Monthly tickets: 75,000
- Annual storage: 75,000 × 12 × 500 bytes = 450 MB/year
- 5-year storage: ~2.25 GB

**Parking Spot Metadata:**
- 1,000 spots × 200 bytes = 200 KB (negligible)

**Total Storage:** < 5 GB for 5 years (very small)

### Bandwidth Estimates

**Data Transfer:**
- Entry request: ~1 KB
- Entry response (ticket): ~1 KB
- Exit request: ~1 KB
- Exit response: ~2 KB

**Peak Bandwidth:**
- Peak operations: 1 op/sec × 2 KB average = 2 KB/sec
- With 10x safety margin: 20 KB/sec
- **Conclusion:** Bandwidth is not a concern

### Cost Estimates (Annual)

**Infrastructure:**
- Database (PostgreSQL): $50/month = $600/year
- Application servers (2 instances): $100/month = $1,200/year
- Load balancer: $25/month = $300/year
- Total: ~$2,100/year

**Conclusion:** A single database and 2 application servers can easily handle this load. The system is CPU-light and data-light.

---

## 4. API Design

We'll design a RESTful API for the parking lot system.

### 4.1 Core APIs

#### 1. Get Available Spots

```http
GET /api/v1/parking/availability
```

**Query Parameters:**
- `floor` (optional): Filter by floor number
- `vehicle_type` (optional): Filter by vehicle type

**Response:**
```json
{
  "total_spots": 1000,
  "available_spots": 300,
  "by_floor": [
    {
      "floor": 1,
      "total": 200,
      "available": 60,
      "by_type": {
        "compact": 15,
        "regular": 30,
        "large": 10,
        "electric": 5
      }
    }
  ],
  "by_type": {
    "compact": 90,
    "regular": 150,
    "large": 45,
    "electric": 15
  }
}
```

#### 2. Vehicle Entry

```http
POST /api/v1/parking/entry
```

**Request Body:**
```json
{
  "vehicle_number": "ABC-1234",
  "vehicle_type": "car",
  "entry_time": "2024-01-15T10:30:00Z"
}
```

**Response:**
```json
{
  "ticket_id": "TKT-20240115-001234",
  "vehicle_number": "ABC-1234",
  "spot": {
    "id": "A-1-045",
    "floor": 1,
    "type": "regular",
    "location": "Section A, Spot 45"
  },
  "entry_time": "2024-01-15T10:30:00Z",
  "qr_code": "base64_encoded_qr_code"
}
```

**Error Responses:**
```json
{
  "error": "NO_AVAILABLE_SPOTS",
  "message": "No parking spots available for vehicle type: car",
  "retry_after": 300
}
```

#### 3. Vehicle Exit

```http
POST /api/v1/parking/exit
```

**Request Body:**
```json
{
  "ticket_id": "TKT-20240115-001234",
  "exit_time": "2024-01-15T13:45:00Z"
}
```

**Response:**
```json
{
  "ticket_id": "TKT-20240115-001234",
  "vehicle_number": "ABC-1234",
  "entry_time": "2024-01-15T10:30:00Z",
  "exit_time": "2024-01-15T13:45:00Z",
  "duration_minutes": 195,
  "fee": {
    "base_fee": 15.00,
    "tax": 1.50,
    "total": 16.50,
    "currency": "USD"
  },
  "payment_url": "/api/v1/payments/PAY-123456"
}
```

#### 4. Process Payment

```http
POST /api/v1/payments
```

**Request Body:**
```json
{
  "ticket_id": "TKT-20240115-001234",
  "amount": 16.50,
  "payment_method": "credit_card",
  "card_token": "tok_visa_4242"
}
```

**Response:**
```json
{
  "payment_id": "PAY-123456",
  "status": "completed",
  "amount": 16.50,
  "receipt_url": "/receipts/PAY-123456.pdf"
}
```

### 4.2 Admin APIs

#### 5. Update Spot Status

```http
PATCH /api/v1/admin/spots/{spot_id}
```

**Request Body:**
```json
{
  "status": "maintenance",
  "reason": "Cleaning in progress"
}
```

#### 6. Get Parking Statistics

```http
GET /api/v1/admin/statistics?start_date=2024-01-01&end_date=2024-01-31
```

**Response:**
```json
{
  "total_vehicles": 75000,
  "total_revenue": 112500.00,
  "average_duration_minutes": 180,
  "occupancy_rate": 0.72,
  "peak_hours": [
    {"hour": 14, "avg_occupancy": 0.89},
    {"hour": 15, "avg_occupancy": 0.91}
  ]
}
```

### 4.3 API Design Principles

1. **Idempotency:** Use idempotency keys for payment operations
2. **Versioning:** All APIs under `/api/v1/` for future compatibility
3. **Rate Limiting:** 100 requests/minute per IP
4. **Authentication:**
   - Public APIs: API key
   - Admin APIs: OAuth 2.0 + role-based access
5. **Error Handling:** Consistent error format with error codes

---

## 5. Data Model & Database Schema

### 5.1 Entity Relationship Diagram

```mermaid
erDiagram
    ParkingLot ||--o{ Floor : contains
    Floor ||--o{ ParkingSpot : contains
    ParkingSpot ||--o{ Ticket : assigned_to
    Vehicle ||--o{ Ticket : owns
    Ticket ||--o| Payment : has

    ParkingLot {
        uuid id PK
        string name
        string address
        int total_floors
        jsonb pricing_config
        timestamp created_at
    }

    Floor {
        uuid id PK
        uuid parking_lot_id FK
        int floor_number
        int total_spots
        string display_name
    }

    ParkingSpot {
        uuid id PK
        uuid floor_id FK
        string spot_number
        enum spot_type
        enum status
        int section
        timestamp updated_at
    }

    Vehicle {
        uuid id PK
        string license_plate UK
        enum vehicle_type
        string owner_name
        string owner_phone
    }

    Ticket {
        uuid id PK
        string ticket_number UK
        uuid vehicle_id FK
        uuid spot_id FK
        timestamp entry_time
        timestamp exit_time
        enum status
        decimal fee_amount
    }

    Payment {
        uuid id PK
        uuid ticket_id FK
        decimal amount
        enum payment_method
        enum status
        string transaction_id
        timestamp paid_at
    }
```

### 5.2 Database Schema (PostgreSQL)

```sql
-- Parking Lot
CREATE TABLE parking_lots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    address TEXT NOT NULL,
    total_floors INT NOT NULL,
    pricing_config JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Floors
CREATE TABLE floors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parking_lot_id UUID NOT NULL REFERENCES parking_lots(id),
    floor_number INT NOT NULL,
    total_spots INT NOT NULL,
    display_name VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(parking_lot_id, floor_number)
);

CREATE INDEX idx_floors_parking_lot ON floors(parking_lot_id);

-- Parking Spots
CREATE TYPE spot_type_enum AS ENUM ('compact', 'regular', 'large', 'electric');
CREATE TYPE spot_status_enum AS ENUM ('available', 'occupied', 'reserved', 'maintenance');

CREATE TABLE parking_spots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    floor_id UUID NOT NULL REFERENCES floors(id),
    spot_number VARCHAR(20) NOT NULL,
    spot_type spot_type_enum NOT NULL,
    status spot_status_enum DEFAULT 'available',
    section VARCHAR(10),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(floor_id, spot_number)
);

CREATE INDEX idx_spots_floor ON parking_spots(floor_id);
CREATE INDEX idx_spots_status_type ON parking_spots(status, spot_type) WHERE status = 'available';

-- Vehicles
CREATE TYPE vehicle_type_enum AS ENUM ('motorcycle', 'car', 'suv', 'truck', 'electric_car');

CREATE TABLE vehicles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    license_plate VARCHAR(20) NOT NULL UNIQUE,
    vehicle_type vehicle_type_enum NOT NULL,
    owner_name VARCHAR(255),
    owner_phone VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_vehicles_license_plate ON vehicles(license_plate);

-- Tickets
CREATE TYPE ticket_status_enum AS ENUM ('active', 'paid', 'exited', 'cancelled');

CREATE TABLE tickets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_number VARCHAR(50) NOT NULL UNIQUE,
    vehicle_id UUID NOT NULL REFERENCES vehicles(id),
    spot_id UUID NOT NULL REFERENCES parking_spots(id),
    entry_time TIMESTAMP NOT NULL,
    exit_time TIMESTAMP,
    status ticket_status_enum DEFAULT 'active',
    fee_amount DECIMAL(10, 2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_tickets_vehicle ON tickets(vehicle_id);
CREATE INDEX idx_tickets_spot ON tickets(spot_id);
CREATE INDEX idx_tickets_status ON tickets(status);
CREATE INDEX idx_tickets_entry_time ON tickets(entry_time);

-- Payments
CREATE TYPE payment_method_enum AS ENUM ('cash', 'credit_card', 'debit_card', 'mobile_wallet', 'monthly_pass');
CREATE TYPE payment_status_enum AS ENUM ('pending', 'completed', 'failed', 'refunded');

CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id UUID NOT NULL REFERENCES tickets(id),
    amount DECIMAL(10, 2) NOT NULL,
    payment_method payment_method_enum NOT NULL,
    status payment_status_enum DEFAULT 'pending',
    transaction_id VARCHAR(100),
    paid_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_payments_ticket ON payments(ticket_id);
CREATE INDEX idx_payments_status ON payments(status);
CREATE INDEX idx_payments_paid_at ON payments(paid_at);
```

### 5.3 Data Access Patterns

**Hot Queries:**
1. Find available spots by type: O(1) with index
2. Get ticket by ticket_number: O(1) with unique index
3. Calculate parking fee: O(1) with ticket_id
4. Update spot status: O(1) with primary key

**Optimization Strategies:**
- Composite index on `(status, spot_type)` for availability queries
- Partial index for available spots only (saves space)
- Use PostgreSQL JSONB for flexible pricing configuration
- Partition tickets table by month for historical data

---

## 6. High-Level Design

### 6.1 Architecture Diagram

```mermaid
graph TB
    subgraph "Client Layer"
        A[Entry Terminal]
        B[Exit Terminal]
        C[Display Boards]
        D[Admin Dashboard]
    end

    subgraph "Load Balancer"
        LB[NGINX/HAProxy]
    end

    subgraph "Application Layer"
        API1[API Server 1]
        API2[API Server 2]
    end

    subgraph "Service Layer"
        PS[Parking Service]
        TS[Ticket Service]
        PYS[Payment Service]
        NS[Notification Service]
    end

    subgraph "Data Layer"
        DB[(PostgreSQL<br/>Primary)]
        DBR[(PostgreSQL<br/>Replica)]
        CACHE[(Redis Cache)]
    end

    subgraph "External Services"
        PAYMENT[Payment Gateway<br/>Stripe/Square]
        SMS[SMS Service]
    end

    A --> LB
    B --> LB
    C --> LB
    D --> LB

    LB --> API1
    LB --> API2

    API1 --> PS
    API1 --> TS
    API1 --> PYS
    API2 --> PS
    API2 --> TS
    API2 --> PYS

    PS --> CACHE
    PS --> DB
    TS --> DB
    PYS --> PAYMENT
    PYS --> DB
    NS --> SMS

    DB --> DBR
    DBR --> C

    style DB fill:#f9f,stroke:#333,stroke-width:2px
    style CACHE fill:#bbf,stroke:#333,stroke-width:2px
```

### 6.2 Component Overview

**1. Entry/Exit Terminals:**
- Physical hardware with barcode/QR scanners
- Print ticket with QR code
- Display available spots
- Simple UI for vehicle type selection

**2. Load Balancer:**
- Distributes traffic across API servers
- Health checks and failover
- SSL termination
- Rate limiting

**3. API Servers:**
- Stateless FastAPI application
- RESTful endpoints
- JWT authentication for admin
- Request validation

**4. Service Layer:**

- **Parking Service:**
  - Spot allocation algorithm
  - Availability tracking
  - Floor/section management

- **Ticket Service:**
  - Ticket generation
  - Fee calculation
  - Ticket validation

- **Payment Service:**
  - Payment processing
  - Receipt generation
  - Refund handling

- **Notification Service:**
  - SMS notifications
  - Email receipts
  - Alert system

**5. Data Layer:**
- **PostgreSQL Primary:** All write operations
- **PostgreSQL Replica:** Read operations for displays
- **Redis Cache:**
  - Available spots count (TTL: 10s)
  - Active ticket lookups
  - Session data

**6. External Services:**
- Payment gateway (Stripe/Square)
- SMS service (Twilio)
- Email service (SendGrid)

---

## 7. Detailed Component Design

### 7.1 Spot Allocation Algorithm

**Strategy:** First-Available with Type Matching

```python
def find_available_spot(vehicle_type: str, preferred_floor: int = None) -> Optional[ParkingSpot]:
    """
    Find available parking spot using optimized algorithm.

    Priority:
    1. Exact type match on preferred floor
    2. Exact type match on any floor
    3. Larger spot type (if available)
    """

    # Map vehicle to acceptable spot types (in priority order)
    SPOT_MAPPING = {
        'motorcycle': ['compact', 'regular'],
        'car': ['regular', 'large'],
        'electric_car': ['electric', 'regular', 'large'],
        'suv': ['large', 'regular'],
        'truck': ['large']
    }

    acceptable_types = SPOT_MAPPING.get(vehicle_type, ['regular'])

    # Try preferred floor first
    if preferred_floor:
        spot = find_spot_on_floor(preferred_floor, acceptable_types)
        if spot:
            return spot

    # Try all floors, prioritizing lower floors
    for floor in range(1, total_floors + 1):
        spot = find_spot_on_floor(floor, acceptable_types)
        if spot:
            return spot

    return None

def find_spot_on_floor(floor_num: int, spot_types: List[str]) -> Optional[ParkingSpot]:
    """Find available spot on specific floor."""
    query = """
        SELECT id, spot_number, spot_type, floor_id
        FROM parking_spots
        WHERE floor_id = (SELECT id FROM floors WHERE floor_number = $1)
          AND status = 'available'
          AND spot_type = ANY($2)
        ORDER BY spot_type = $2[1] DESC, section, spot_number
        LIMIT 1
        FOR UPDATE SKIP LOCKED
    """
    return db.execute(query, floor_num, spot_types)
```

**Key Features:**
- `FOR UPDATE SKIP LOCKED`: Prevents race conditions
- Type priority ordering
- Floor preference support
- Atomic spot reservation

### 7.2 Fee Calculation Engine

```python
class FeeCalculator:
    """Calculate parking fees based on duration and vehicle type."""

    def __init__(self, pricing_config: dict):
        self.config = pricing_config
        # Example config:
        # {
        #   "hourly_rates": {
        #     "motorcycle": 2.0,
        #     "car": 5.0,
        #     "suv": 7.0,
        #     "truck": 10.0
        #   },
        #   "daily_max": 50.0,
        #   "grace_period_minutes": 15,
        #   "tax_rate": 0.10
        # }

    def calculate_fee(
        self,
        vehicle_type: str,
        entry_time: datetime,
        exit_time: datetime
    ) -> dict:
        """Calculate total parking fee."""

        # Calculate duration
        duration = exit_time - entry_time
        total_minutes = duration.total_seconds() / 60

        # Grace period (free for first 15 minutes)
        if total_minutes <= self.config['grace_period_minutes']:
            return {
                'base_fee': 0.0,
                'tax': 0.0,
                'total': 0.0,
                'duration_minutes': total_minutes
            }

        # Calculate billable hours (round up)
        billable_hours = math.ceil(total_minutes / 60)

        # Get hourly rate
        hourly_rate = self.config['hourly_rates'].get(vehicle_type, 5.0)

        # Calculate base fee
        base_fee = billable_hours * hourly_rate

        # Apply daily maximum
        daily_max = self.config.get('daily_max', float('inf'))
        base_fee = min(base_fee, daily_max)

        # Calculate tax
        tax_rate = self.config.get('tax_rate', 0.0)
        tax = base_fee * tax_rate

        return {
            'base_fee': round(base_fee, 2),
            'tax': round(tax, 2),
            'total': round(base_fee + tax, 2),
            'duration_minutes': int(total_minutes),
            'billable_hours': billable_hours
        }
```

### 7.3 Ticket Generation

```python
class TicketGenerator:
    """Generate unique parking tickets."""

    def generate_ticket_number(self, entry_time: datetime) -> str:
        """
        Generate unique ticket number.
        Format: TKT-YYYYMMDD-NNNNNN
        """
        date_part = entry_time.strftime('%Y%m%d')

        # Get daily sequence number (thread-safe)
        sequence = self._get_next_sequence(date_part)

        return f"TKT-{date_part}-{sequence:06d}"

    def _get_next_sequence(self, date_part: str) -> int:
        """Get next sequence number for the day using Redis."""
        key = f"ticket:sequence:{date_part}"
        sequence = redis.incr(key)
        redis.expire(key, 86400 * 2)  # Keep for 2 days
        return sequence

    def generate_qr_code(self, ticket_number: str) -> str:
        """Generate QR code for ticket."""
        import qrcode
        import base64
        from io import BytesIO

        qr = qrcode.QRCode(version=1, box_size=10, border=5)
        qr.add_data(ticket_number)
        qr.make(fit=True)

        img = qr.make_image(fill_color="black", back_color="white")
        buffer = BytesIO()
        img.save(buffer, format='PNG')

        return base64.b64encode(buffer.getvalue()).decode()
```

### 7.4 Availability Tracking with Redis

```python
class AvailabilityTracker:
    """Track available spots using Redis for real-time updates."""

    def __init__(self, redis_client):
        self.redis = redis_client

    def update_spot_status(self, spot_id: str, is_available: bool, spot_type: str, floor: int):
        """Update spot availability in Redis."""
        pipeline = self.redis.pipeline()

        # Update total available count
        key_total = "parking:available:total"
        pipeline.incr(key_total, 1 if is_available else -1)

        # Update by type
        key_type = f"parking:available:type:{spot_type}"
        pipeline.incr(key_type, 1 if is_available else -1)

        # Update by floor
        key_floor = f"parking:available:floor:{floor}"
        pipeline.incr(key_floor, 1 if is_available else -1)

        # Update by floor and type
        key_floor_type = f"parking:available:floor:{floor}:type:{spot_type}"
        pipeline.incr(key_floor_type, 1 if is_available else -1)

        # Set TTL to auto-sync with DB
        for key in [key_total, key_type, key_floor, key_floor_type]:
            pipeline.expire(key, 60)

        pipeline.execute()

    def get_availability(self, floor: int = None, vehicle_type: str = None) -> dict:
        """Get availability from Redis with DB fallback."""
        if floor and vehicle_type:
            key = f"parking:available:floor:{floor}:type:{vehicle_type}"
        elif floor:
            key = f"parking:available:floor:{floor}"
        elif vehicle_type:
            key = f"parking:available:type:{vehicle_type}"
        else:
            key = "parking:available:total"

        count = self.redis.get(key)

        if count is None:
            # Cache miss - fetch from database
            count = self._fetch_from_db(floor, vehicle_type)
            self.redis.setex(key, 60, count)

        return int(count)

    def _fetch_from_db(self, floor: int = None, vehicle_type: str = None) -> int:
        """Fetch availability from database."""
        query = "SELECT COUNT(*) FROM parking_spots WHERE status = 'available'"
        params = []

        if floor:
            query += " AND floor_id = (SELECT id FROM floors WHERE floor_number = $1)"
            params.append(floor)

        if vehicle_type:
            query += f" AND spot_type = ${len(params) + 1}"
            params.append(vehicle_type)

        return db.fetchone(query, *params)[0]
```

### 7.5 Concurrency Control

**Problem:** Multiple vehicles trying to park simultaneously might get assigned the same spot.

**Solution:** Database-level locking with `SELECT ... FOR UPDATE SKIP LOCKED`

```python
async def assign_spot_atomically(vehicle_type: str) -> Optional[ParkingSpot]:
    """
    Atomically assign a parking spot with row-level locking.
    """
    async with db.transaction():
        # Find and lock the spot in one query
        query = """
            UPDATE parking_spots
            SET status = 'occupied', updated_at = NOW()
            WHERE id = (
                SELECT id FROM parking_spots
                WHERE status = 'available'
                  AND spot_type = $1
                ORDER BY floor_id, spot_number
                LIMIT 1
                FOR UPDATE SKIP LOCKED
            )
            RETURNING id, spot_number, spot_type, floor_id
        """

        spot = await db.fetchone(query, vehicle_type)

        if spot:
            # Update cache
            availability_tracker.update_spot_status(
                spot['id'],
                is_available=False,
                spot_type=spot['spot_type'],
                floor=spot['floor_id']
            )

        return spot
```

**Key Points:**
- `FOR UPDATE SKIP LOCKED`: Skip already-locked rows (no waiting)
- Transaction ensures atomicity
- Cache updated after DB commit
- Returns None if no spots available

---

## 8. Identifying and Resolving Bottlenecks

### 8.1 Potential Bottlenecks

| Bottleneck | Impact | Solution |
|------------|--------|----------|
| **Database Write Contention** | High concurrent entry/exit | Connection pooling, read replicas |
| **Spot Assignment Race Conditions** | Double-booking | Row-level locking, optimistic locking |
| **Availability Query Load** | Slow display board updates | Redis caching, read replicas |
| **Payment Processing Delays** | Exit gate delays | Async payment processing, pre-payment |
| **Single Point of Failure** | System downtime | Database replication, API server redundancy |

### 8.2 Scalability Solutions

#### 8.2.1 Database Scaling

**Problem:** Single database becomes bottleneck at 10,000+ spots across multiple facilities.

**Solutions:**

1. **Read Replicas:**
```python
# Route reads to replica
@router.get("/availability")
async def get_availability():
    return await db_replica.query("SELECT ...")

# Route writes to primary
@router.post("/entry")
async def vehicle_entry():
    return await db_primary.query("INSERT ...")
```

2. **Connection Pooling:**
```python
# PostgreSQL connection pool
DATABASE_POOL = {
    'min_size': 10,
    'max_size': 50,
    'max_queries': 50000,
    'max_inactive_connection_lifetime': 300
}
```

3. **Sharding by Facility:**
```python
# Shard database by parking_lot_id
def get_shard(parking_lot_id: str) -> Database:
    shard_number = hash(parking_lot_id) % NUM_SHARDS
    return database_shards[shard_number]
```

#### 8.2.2 Caching Strategy

**Multi-Layer Caching:**

```python
class CacheManager:
    """Multi-layer caching for parking system."""

    def __init__(self):
        self.l1_cache = {}  # In-memory (local to API server)
        self.l2_cache = redis_client  # Shared Redis

    async def get_availability(self, key: str) -> Optional[int]:
        # L1: In-memory cache (100ms TTL)
        if key in self.l1_cache:
            value, timestamp = self.l1_cache[key]
            if time.time() - timestamp < 0.1:  # 100ms
                return value

        # L2: Redis cache (10s TTL)
        value = await self.l2_cache.get(key)
        if value:
            self.l1_cache[key] = (int(value), time.time())
            return int(value)

        # L3: Database
        value = await self._fetch_from_db(key)
        await self.l2_cache.setex(key, 10, value)
        self.l1_cache[key] = (value, time.time())
        return value
```

**Cache Invalidation:**
```python
# Invalidate on spot status change
def on_spot_status_change(spot_id, old_status, new_status):
    if old_status == 'available' or new_status == 'available':
        # Invalidate all related cache keys
        keys = [
            "parking:available:total",
            f"parking:available:floor:{spot.floor_id}",
            f"parking:available:type:{spot.spot_type}",
            f"parking:available:floor:{spot.floor_id}:type:{spot.spot_type}"
        ]
        redis.delete(*keys)
```

#### 8.2.3 Load Balancing

```nginx
# NGINX configuration for load balancing
upstream parking_api {
    least_conn;  # Send to server with fewest connections

    server api-server-1:8000 weight=1 max_fails=3 fail_timeout=30s;
    server api-server-2:8000 weight=1 max_fails=3 fail_timeout=30s;
    server api-server-3:8000 weight=1 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;

    location /api/ {
        proxy_pass http://parking_api;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_connect_timeout 2s;
        proxy_read_timeout 10s;
    }
}
```

### 8.3 Handling Peak Load

**Scenario:** Shopping mall on Black Friday - 5x normal traffic

**Strategies:**

1. **Rate Limiting:**
```python
from fastapi_limiter import FastAPILimiter
from fastapi_limiter.depends import RateLimiter

@app.get("/availability", dependencies=[Depends(RateLimiter(times=10, seconds=1))])
async def get_availability():
    pass  # Limited to 10 requests/second per IP
```

2. **Request Throttling:**
```python
# Prioritize entry/exit over availability checks
PRIORITY_QUEUES = {
    'entry': 1,      # Highest priority
    'exit': 1,
    'payment': 2,
    'availability': 3  # Lowest priority
}
```

3. **Graceful Degradation:**
```python
async def get_availability_with_fallback():
    try:
        # Try real-time data
        return await get_realtime_availability()
    except DatabaseTimeout:
        # Fall back to cached data (may be stale)
        return await get_cached_availability()
    except Exception:
        # Ultimate fallback
        return {"message": "Availability data temporarily unavailable"}
```

---

## 9. Trade-offs and Alternatives

### 9.1 Spot Assignment Strategies

| Strategy | Pros | Cons | Use Case |
|----------|------|------|----------|
| **First-Available** | Simple, fast | Uneven floor usage | Small facilities |
| **Round-Robin by Floor** | Even distribution | Complex logic | Multi-floor garages |
| **Nearest-to-Exit** | Faster exits | Entry overhead | Valet parking |
| **Dynamic Pricing** | Revenue optimization | Complex implementation | Premium facilities |

**Our Choice:** First-Available with floor hints
- **Why:** Simple, fast, works for 95% of use cases
- **Trade-off:** May fill lower floors first (acceptable)

### 9.2 Database Choices

| Database | Pros | Cons | Verdict |
|----------|------|------|---------|
| **PostgreSQL** | ACID, rich features, JSON support | Vertical scaling limits | ✅ **Chosen** |
| **MySQL** | Simple, widely supported | Weaker JSON support | Alternative |
| **MongoDB** | Flexible schema, horizontal scaling | No ACID across documents | Not suitable |
| **Cassandra** | High write throughput | Complex consistency | Overkill |

**Our Choice:** PostgreSQL
- **Why:** ACID guarantees critical for no double-booking
- **Trade-off:** Harder to scale horizontally (acceptable for < 1M spots)

### 9.3 Caching Strategies

**Strategy 1: Cache-Aside (Lazy Loading)**
```python
# Read from cache, fall back to DB
data = cache.get(key)
if not data:
    data = db.query(...)
    cache.set(key, data, ttl=60)
return data
```
- ✅ Pros: Only cache what's needed
- ❌ Cons: Cache miss penalty, stale data risk

**Strategy 2: Write-Through**
```python
# Write to DB and cache simultaneously
db.insert(...)
cache.set(key, data, ttl=60)
```
- ✅ Pros: Cache always fresh
- ❌ Cons: Write latency, cache pollution

**Our Choice:** Hybrid
- Write-through for availability counts (critical)
- Cache-aside for statistics (less critical)

### 9.4 Payment Processing

**Option 1: Synchronous Payment**
```
Entry → Park → Exit → Pay (wait) → Gate Opens
```
- ✅ Simple implementation
- ❌ Exit delays if payment slow

**Option 2: Async Payment**
```
Entry → Park → Exit (immediate) → Pay Later (online/app)
```
- ✅ No exit delays
- ❌ Revenue loss risk (3-5% non-payment)

**Option 3: Prepayment (Our Choice)**
```
Entry → Park → Prepay (kiosk/app) → Exit (scan receipt)
```
- ✅ No exit delays
- ✅ Guaranteed payment
- ❌ User friction

**Recommendation:** Prepayment with grace period (pay within 10 minutes of exit)

---

## 10. Monitoring, Metrics & Alerts

### 10.1 Key Metrics

#### Business Metrics

```python
# Track in Prometheus/CloudWatch
business_metrics = {
    # Revenue
    'revenue_per_hour': Gauge('parking_revenue_per_hour'),
    'average_parking_duration': Histogram('parking_duration_minutes'),

    # Occupancy
    'occupancy_rate': Gauge('parking_occupancy_rate'),
    'available_spots_by_type': Gauge('available_spots', ['spot_type']),

    # Operations
    'vehicles_per_hour': Counter('vehicles_total', ['operation']),  # entry/exit
    'payment_success_rate': Gauge('payment_success_rate'),

    # Customer Experience
    'entry_processing_time': Histogram('entry_processing_seconds'),
    'exit_processing_time': Histogram('exit_processing_seconds'),
}
```

#### Technical Metrics

```python
technical_metrics = {
    # API Performance
    'api_request_duration': Histogram('api_request_duration_seconds', ['endpoint']),
    'api_request_total': Counter('api_requests_total', ['endpoint', 'status']),

    # Database
    'db_connection_pool_size': Gauge('db_pool_connections', ['state']),
    'db_query_duration': Histogram('db_query_duration_seconds', ['query_type']),

    # Cache
    'cache_hit_rate': Gauge('cache_hit_rate', ['cache_type']),
    'cache_memory_usage': Gauge('cache_memory_bytes'),

    # System
    'cpu_usage': Gauge('cpu_usage_percent'),
    'memory_usage': Gauge('memory_usage_bytes'),
}
```

### 10.2 Alerts Configuration

```yaml
# Prometheus Alerting Rules
groups:
  - name: parking_lot_alerts
    interval: 30s
    rules:
      # Business Alerts
      - alert: LowAvailability
        expr: parking_occupancy_rate > 0.95
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Parking almost full ({{ $value }}% occupied)"

      - alert: HighPaymentFailureRate
        expr: payment_success_rate < 0.90
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Payment failure rate at {{ $value }}"

      # Technical Alerts
      - alert: HighAPILatency
        expr: histogram_quantile(0.95, api_request_duration_seconds) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "95th percentile API latency > 2s"

      - alert: DatabaseConnectionPoolExhausted
        expr: db_pool_connections{state="idle"} < 5
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Database connection pool nearly exhausted"

      - alert: CacheLowHitRate
        expr: cache_hit_rate < 0.70
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Cache hit rate below 70% ({{ $value }})"
```

### 10.3 Logging Strategy

```python
import structlog

logger = structlog.get_logger()

# Structured logging for better querying
def log_vehicle_entry(ticket_id, vehicle_number, spot_id):
    logger.info(
        "vehicle_entered",
        event="entry",
        ticket_id=ticket_id,
        vehicle_number=vehicle_number,
        spot_id=spot_id,
        timestamp=datetime.utcnow().isoformat()
    )

def log_payment_processed(ticket_id, amount, payment_method, status):
    logger.info(
        "payment_processed",
        event="payment",
        ticket_id=ticket_id,
        amount=amount,
        payment_method=payment_method,
        status=status,
        timestamp=datetime.utcnow().isoformat()
    )
```

**Log Levels:**
- `DEBUG`: Spot allocation attempts, cache hits/misses
- `INFO`: Entry/exit events, payments
- `WARNING`: High occupancy, payment retries
- `ERROR`: Payment failures, database errors
- `CRITICAL`: System unavailability, data corruption

### 10.4 Dashboards

**Grafana Dashboard Layout:**

```
┌─────────────────────────────────────────┐
│  Real-Time Occupancy by Floor           │
│  ▮▮▮▮▮▮▮▮▮▮▮▮▮▮▮░░░ Floor 1: 85%      │
│  ▮▮▮▮▮▮▮▮▮▮▮░░░░░░░ Floor 2: 65%      │
├─────────────────────────────────────────┤
│  Revenue Today        Vehicles Today    │
│  $12,450 (+15%)       1,234 (+8%)      │
├─────────────────────────────────────────┤
│  API Response Times (p95)               │
│  Entry: 0.8s  Exit: 1.2s  Payment: 2.1s│
├─────────────────────────────────────────┤
│  Payment Success Rate    Cache Hit Rate │
│  98.5% ✓                 87.2% ✓       │
└─────────────────────────────────────────┘
```

---

## 11. Follow-up Questions & Extensions

### Common Interview Follow-ups

#### Q1: "How would you handle license plate recognition (LPR)?"

**Answer:**
```python
class LPRIntegration:
    """Integrate with License Plate Recognition camera."""

    async def process_entry_with_lpr(self, image: bytes) -> dict:
        # Call LPR service (AWS Rekognition, OpenALPR)
        license_plate = await lpr_service.recognize(image)

        # Check for existing vehicle
        vehicle = await db.get_vehicle_by_plate(license_plate)

        if vehicle and vehicle.has_monthly_pass:
            # Auto-entry for pass holders
            spot = await assign_spot(vehicle.vehicle_type)
            ticket = await create_ticket(vehicle.id, spot.id)
            await open_gate()
            return {"status": "auto_entry", "ticket": ticket}
        else:
            # Regular flow
            return await regular_entry_flow(license_plate)
```

**Changes Required:**
- Add `LPR cameras` at entry/exit gates
- Add `monthly_pass` table
- Add `vehicle_images` table for audit trail
- Implement gate automation APIs

#### Q2: "How would you implement reserved parking?"

**Answer:**
```sql
-- Add reservations table
CREATE TABLE reservations (
    id UUID PRIMARY KEY,
    spot_id UUID REFERENCES parking_spots(id),
    vehicle_id UUID REFERENCES vehicles(id),
    reserved_from TIMESTAMP NOT NULL,
    reserved_until TIMESTAMP NOT NULL,
    status reservation_status_enum DEFAULT 'active'
);

-- Update spot allocation
CREATE INDEX idx_reservations_active ON reservations(spot_id, status)
WHERE status = 'active'
AND reserved_from <= NOW()
AND reserved_until >= NOW();
```

**Spot Allocation Logic:**
```python
def find_available_spot_with_reservations(vehicle_id, vehicle_type):
    # First check if vehicle has a reservation
    reserved_spot = db.query("""
        SELECT spot_id FROM reservations
        WHERE vehicle_id = $1
          AND status = 'active'
          AND reserved_from <= NOW()
          AND reserved_until >= NOW()
    """, vehicle_id)

    if reserved_spot:
        return reserved_spot

    # Otherwise find non-reserved spot
    return find_available_spot(vehicle_type, exclude_reserved=True)
```

#### Q3: "How would you scale this to 100 parking facilities?"

**Answer:**

**Multi-Facility Architecture:**

```mermaid
graph TB
    subgraph "Facility 1"
        F1[Local API Server]
        F1DB[(Local DB)]
    end

    subgraph "Facility 2"
        F2[Local API Server]
        F2DB[(Local DB)]
    end

    subgraph "Central System"
        CENTRAL[Central API]
        CENTRALDB[(Central DB)]
        CACHE[(Redis Cluster)]
    end

    APP[Mobile App] --> CENTRAL
    F1 --> F1DB
    F2 --> F2DB
    F1 -.Sync.-> CENTRAL
    F2 -.Sync.-> CENTRAL
    CENTRAL --> CENTRALDB
    CENTRAL --> CACHE
```

**Key Changes:**
1. **Database per Facility:** Each facility has local DB for resilience
2. **Central Aggregation:** Central system aggregates data for reporting
3. **Async Sync:** Facilities sync to central system every 5 minutes
4. **Federated Search:** Mobile app searches across all facilities

```python
# Multi-facility availability
async def get_availability_across_facilities(city: str, radius_km: float):
    facilities = await get_nearby_facilities(city, radius_km)

    # Query all facilities in parallel
    tasks = [get_facility_availability(f.id) for f in facilities]
    results = await asyncio.gather(*tasks)

    return {
        "facilities": [
            {
                "id": f.id,
                "name": f.name,
                "distance_km": f.distance,
                "available_spots": r.available,
                "pricing": f.pricing
            }
            for f, r in zip(facilities, results)
        ]
    }
```

#### Q4: "How would you implement dynamic pricing?"

**Answer:**
```python
class DynamicPricingEngine:
    """Calculate prices based on demand."""

    def calculate_price(self, vehicle_type: str, entry_time: datetime) -> float:
        base_rate = self.get_base_rate(vehicle_type)

        # Demand multiplier (1.0 - 3.0)
        occupancy = self.get_current_occupancy()
        demand_multiplier = self._calculate_demand_multiplier(occupancy)

        # Time multiplier
        time_multiplier = self._get_time_multiplier(entry_time)

        # Event multiplier (concerts, games, etc.)
        event_multiplier = self._get_event_multiplier(entry_time)

        final_rate = base_rate * demand_multiplier * time_multiplier * event_multiplier

        return round(final_rate, 2)

    def _calculate_demand_multiplier(self, occupancy: float) -> float:
        """Surge pricing based on occupancy."""
        if occupancy < 0.50:
            return 1.0  # Normal price
        elif occupancy < 0.75:
            return 1.5  # 50% increase
        elif occupancy < 0.90:
            return 2.0  # Double price
        else:
            return 3.0  # Triple price (high demand)

    def _get_time_multiplier(self, entry_time: datetime) -> float:
        """Peak hour pricing."""
        hour = entry_time.hour
        if 9 <= hour < 17:  # Business hours
            return 1.5
        elif 17 <= hour < 20:  # Evening peak
            return 1.8
        else:
            return 1.0  # Off-peak
```

### Extensions

1. **Electric Vehicle Charging:**
   - Track charging status
   - Calculate electricity fees
   - Reserve charging spots

2. **Valet Parking:**
   - Add valet staff assignment
   - Track key handoff
   - Optimize car movement

3. **Mobile App:**
   - Find parking before arrival
   - Navigate to assigned spot
   - Pay via app (skip kiosk)

4. **Analytics & ML:**
   - Predict peak hours
   - Optimize pricing
   - Forecast maintenance needs

---

## 12. Code Implementation

Complete FastAPI implementation of the parking lot system.

```python
# main.py
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager
import asyncpg
import redis.asyncio as redis
from datetime import datetime
from typing import Optional, List
from uuid import UUID
import os

from models import (
    VehicleEntry, VehicleExit, TicketResponse,
    AvailabilityResponse, PaymentRequest, PaymentResponse
)
from services import (
    ParkingService, TicketService, PaymentService,
    AvailabilityTracker, FeeCalculator
)

# Database configuration
DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://user:pass@localhost/parking")
REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379")

# Global instances
db_pool: asyncpg.Pool = None
redis_client: redis.Redis = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    global db_pool, redis_client

    db_pool = await asyncpg.create_pool(
        DATABASE_URL,
        min_size=10,
        max_size=50,
        command_timeout=60
    )

    redis_client = await redis.from_url(
        REDIS_URL,
        encoding="utf-8",
        decode_responses=True
    )

    print("✓ Database and Redis connected")

    yield

    # Shutdown
    await db_pool.close()
    await redis_client.close()
    print("✓ Connections closed")

app = FastAPI(
    title="Parking Lot Management System",
    version="1.0.0",
    lifespan=lifespan
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Dependency injection
async def get_db():
    async with db_pool.acquire() as connection:
        yield connection

async def get_redis():
    return redis_client

async def get_parking_service(
    db: asyncpg.Connection = Depends(get_db),
    redis: redis.Redis = Depends(get_redis)
) -> ParkingService:
    return ParkingService(db, redis)

async def get_ticket_service(
    db: asyncpg.Connection = Depends(get_db),
    redis: redis.Redis = Depends(get_redis)
) -> TicketService:
    return TicketService(db, redis)

# ============================================================================
# API Endpoints
# ============================================================================

@app.get("/")
async def root():
    return {
        "service": "Parking Lot Management System",
        "version": "1.0.0",
        "status": "operational"
    }

@app.get("/health")
async def health_check(db: asyncpg.Connection = Depends(get_db)):
    """Health check endpoint."""
    try:
        await db.fetchval("SELECT 1")
        return {"status": "healthy", "database": "connected"}
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}

# ============================================================================
# Parking Operations
# ============================================================================

@app.get("/api/v1/parking/availability", response_model=AvailabilityResponse)
async def get_availability(
    floor: Optional[int] = None,
    vehicle_type: Optional[str] = None,
    service: ParkingService = Depends(get_parking_service)
):
    """
    Get real-time parking availability.

    - **floor**: Filter by specific floor (1-5)
    - **vehicle_type**: Filter by vehicle type (motorcycle, car, suv, truck, electric_car)
    """
    return await service.get_availability(floor, vehicle_type)

@app.post("/api/v1/parking/entry", response_model=TicketResponse)
async def vehicle_entry(
    entry: VehicleEntry,
    service: ParkingService = Depends(get_parking_service)
):
    """
    Process vehicle entry and issue parking ticket.

    Returns ticket with assigned parking spot.
    """
    try:
        ticket = await service.process_entry(
            vehicle_number=entry.vehicle_number,
            vehicle_type=entry.vehicle_type,
            entry_time=entry.entry_time or datetime.utcnow()
        )
        return ticket
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    except Exception as e:
        raise HTTPException(status_code=500, detail="Internal server error")

@app.post("/api/v1/parking/exit")
async def vehicle_exit(
    exit_data: VehicleExit,
    service: ParkingService = Depends(get_parking_service)
):
    """
    Process vehicle exit and calculate parking fee.
    """
    try:
        exit_info = await service.process_exit(
            ticket_number=exit_data.ticket_number,
            exit_time=exit_data.exit_time or datetime.utcnow()
        )
        return exit_info
    except ValueError as e:
        raise HTTPException(status_code=404, detail=str(e))
    except Exception as e:
        raise HTTPException(status_code=500, detail="Internal server error")

@app.get("/api/v1/tickets/{ticket_number}")
async def get_ticket(
    ticket_number: str,
    service: TicketService = Depends(get_ticket_service)
):
    """Get ticket details by ticket number."""
    ticket = await service.get_ticket(ticket_number)
    if not ticket:
        raise HTTPException(status_code=404, detail="Ticket not found")
    return ticket

# ============================================================================
# Payment Operations
# ============================================================================

@app.post("/api/v1/payments", response_model=PaymentResponse)
async def process_payment(
    payment: PaymentRequest,
    db: asyncpg.Connection = Depends(get_db)
):
    """Process parking fee payment."""
    payment_service = PaymentService(db)

    try:
        result = await payment_service.process_payment(
            ticket_id=payment.ticket_id,
            amount=payment.amount,
            payment_method=payment.payment_method,
            card_token=payment.card_token
        )
        return result
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    except Exception as e:
        raise HTTPException(status_code=500, detail="Payment processing failed")

# ============================================================================
# Admin Operations
# ============================================================================

@app.patch("/api/v1/admin/spots/{spot_id}")
async def update_spot_status(
    spot_id: UUID,
    status: str,
    reason: Optional[str] = None,
    db: asyncpg.Connection = Depends(get_db)
):
    """Update parking spot status (admin only)."""
    valid_statuses = ['available', 'occupied', 'reserved', 'maintenance']
    if status not in valid_statuses:
        raise HTTPException(
            status_code=400,
            detail=f"Invalid status. Must be one of: {valid_statuses}"
        )

    await db.execute(
        """
        UPDATE parking_spots
        SET status = $1, updated_at = NOW()
        WHERE id = $2
        """,
        status, spot_id
    )

    return {"status": "updated", "spot_id": str(spot_id), "new_status": status}

@app.get("/api/v1/admin/statistics")
async def get_statistics(
    start_date: datetime,
    end_date: datetime,
    db: asyncpg.Connection = Depends(get_db)
):
    """Get parking statistics for date range."""
    stats = await db.fetchrow(
        """
        SELECT
            COUNT(*) as total_vehicles,
            SUM(fee_amount) as total_revenue,
            AVG(EXTRACT(EPOCH FROM (exit_time - entry_time))/60) as avg_duration_minutes,
            COUNT(*) FILTER (WHERE exit_time IS NULL) as currently_parked
        FROM tickets
        WHERE entry_time BETWEEN $1 AND $2
        """,
        start_date, end_date
    )

    return dict(stats)

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

```python
# models.py
from pydantic import BaseModel, Field
from datetime import datetime
from typing import Optional
from uuid import UUID
from enum import Enum

class VehicleType(str, Enum):
    MOTORCYCLE = "motorcycle"
    CAR = "car"
    SUV = "suv"
    TRUCK = "truck"
    ELECTRIC_CAR = "electric_car"

class SpotType(str, Enum):
    COMPACT = "compact"
    REGULAR = "regular"
    LARGE = "large"
    ELECTRIC = "electric"

class PaymentMethod(str, Enum):
    CASH = "cash"
    CREDIT_CARD = "credit_card"
    DEBIT_CARD = "debit_card"
    MOBILE_WALLET = "mobile_wallet"
    MONTHLY_PASS = "monthly_pass"

class VehicleEntry(BaseModel):
    vehicle_number: str = Field(..., example="ABC-1234")
    vehicle_type: VehicleType = Field(..., example="car")
    entry_time: Optional[datetime] = None

class VehicleExit(BaseModel):
    ticket_number: str = Field(..., example="TKT-20240115-001234")
    exit_time: Optional[datetime] = None

class ParkingSpotInfo(BaseModel):
    id: str
    floor: int
    spot_number: str
    spot_type: str
    location: str

class FeeBreakdown(BaseModel):
    base_fee: float
    tax: float
    total: float
    currency: str = "USD"
    duration_minutes: int

class TicketResponse(BaseModel):
    ticket_id: str
    ticket_number: str
    vehicle_number: str
    spot: ParkingSpotInfo
    entry_time: datetime
    qr_code: Optional[str] = None

class AvailabilityResponse(BaseModel):
    total_spots: int
    available_spots: int
    by_type: dict
    by_floor: list

class PaymentRequest(BaseModel):
    ticket_id: UUID
    amount: float
    payment_method: PaymentMethod
    card_token: Optional[str] = None

class PaymentResponse(BaseModel):
    payment_id: str
    status: str
    amount: float
    receipt_url: Optional[str] = None
```

```python
# services.py
import asyncpg
import redis.asyncio as redis
from datetime import datetime
from typing import Optional, Dict, List
import math
import qrcode
import base64
from io import BytesIO
from uuid import UUID, uuid4

class FeeCalculator:
    """Calculate parking fees based on duration and vehicle type."""

    PRICING_CONFIG = {
        "hourly_rates": {
            "motorcycle": 2.0,
            "car": 5.0,
            "suv": 7.0,
            "truck": 10.0,
            "electric_car": 5.0
        },
        "daily_max": 50.0,
        "grace_period_minutes": 15,
        "tax_rate": 0.10
    }

    def calculate_fee(
        self,
        vehicle_type: str,
        entry_time: datetime,
        exit_time: datetime
    ) -> Dict:
        """Calculate total parking fee."""
        duration = exit_time - entry_time
        total_minutes = duration.total_seconds() / 60

        # Grace period
        if total_minutes <= self.PRICING_CONFIG['grace_period_minutes']:
            return {
                'base_fee': 0.0,
                'tax': 0.0,
                'total': 0.0,
                'duration_minutes': int(total_minutes),
                'currency': 'USD'
            }

        # Calculate billable hours (round up)
        billable_hours = math.ceil(total_minutes / 60)

        # Get hourly rate
        hourly_rate = self.PRICING_CONFIG['hourly_rates'].get(vehicle_type, 5.0)

        # Calculate base fee
        base_fee = billable_hours * hourly_rate
        base_fee = min(base_fee, self.PRICING_CONFIG['daily_max'])

        # Calculate tax
        tax = base_fee * self.PRICING_CONFIG['tax_rate']

        return {
            'base_fee': round(base_fee, 2),
            'tax': round(tax, 2),
            'total': round(base_fee + tax, 2),
            'duration_minutes': int(total_minutes),
            'currency': 'USD'
        }

class TicketGenerator:
    """Generate unique parking tickets."""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def generate_ticket_number(self, entry_time: datetime) -> str:
        """Generate unique ticket number."""
        date_part = entry_time.strftime('%Y%m%d')

        # Get daily sequence number (thread-safe)
        key = f"ticket:sequence:{date_part}"
        sequence = self.redis.incr(key)
        self.redis.expire(key, 86400 * 2)

        return f"TKT-{date_part}-{sequence:06d}"

    def generate_qr_code(self, ticket_number: str) -> str:
        """Generate QR code for ticket."""
        qr = qrcode.QRCode(version=1, box_size=10, border=5)
        qr.add_data(ticket_number)
        qr.make(fit=True)

        img = qr.make_image(fill_color="black", back_color="white")
        buffer = BytesIO()
        img.save(buffer, format='PNG')

        return base64.b64encode(buffer.getvalue()).decode()

class AvailabilityTracker:
    """Track available spots using Redis."""

    def __init__(self, db: asyncpg.Connection, redis_client: redis.Redis):
        self.db = db
        self.redis = redis_client

    async def get_availability(
        self,
        floor: Optional[int] = None,
        vehicle_type: Optional[str] = None
    ) -> Dict:
        """Get availability from Redis with DB fallback."""

        # Try cache first
        cache_key = self._get_cache_key(floor, vehicle_type)
        cached = await self.redis.get(cache_key)

        if cached is not None:
            return int(cached)

        # Fetch from database
        count = await self._fetch_from_db(floor, vehicle_type)

        # Cache for 10 seconds
        await self.redis.setex(cache_key, 10, count)

        return count

    def _get_cache_key(self, floor: Optional[int], vehicle_type: Optional[str]) -> str:
        """Generate cache key."""
        if floor and vehicle_type:
            return f"parking:available:floor:{floor}:type:{vehicle_type}"
        elif floor:
            return f"parking:available:floor:{floor}"
        elif vehicle_type:
            return f"parking:available:type:{vehicle_type}"
        else:
            return "parking:available:total"

    async def _fetch_from_db(
        self,
        floor: Optional[int] = None,
        vehicle_type: Optional[str] = None
    ) -> int:
        """Fetch availability from database."""
        query = "SELECT COUNT(*) FROM parking_spots WHERE status = 'available'"
        params = []

        if floor:
            params.append(floor)
            query += f" AND floor_id = (SELECT id FROM floors WHERE floor_number = ${len(params)})"

        if vehicle_type:
            params.append(vehicle_type)
            query += f" AND spot_type = ${len(params)}"

        return await self.db.fetchval(query, *params)

class ParkingService:
    """Main parking operations service."""

    def __init__(self, db: asyncpg.Connection, redis_client: redis.Redis):
        self.db = db
        self.redis = redis_client
        self.ticket_generator = TicketGenerator(redis_client)
        self.fee_calculator = FeeCalculator()
        self.availability_tracker = AvailabilityTracker(db, redis_client)

    async def get_availability(
        self,
        floor: Optional[int] = None,
        vehicle_type: Optional[str] = None
    ) -> Dict:
        """Get parking availability."""

        # Get total and available
        total_spots = await self.db.fetchval("SELECT COUNT(*) FROM parking_spots")
        available_spots = await self.availability_tracker.get_availability()

        # Get by type
        by_type = {}
        for spot_type in ['compact', 'regular', 'large', 'electric']:
            count = await self.availability_tracker.get_availability(vehicle_type=spot_type)
            by_type[spot_type] = count

        # Get by floor
        by_floor = []
        for floor_num in range(1, 6):
            floor_available = await self.availability_tracker.get_availability(floor=floor_num)
            by_floor.append({
                'floor': floor_num,
                'available': floor_available
            })

        return {
            'total_spots': total_spots,
            'available_spots': available_spots,
            'by_type': by_type,
            'by_floor': by_floor
        }

    async def process_entry(
        self,
        vehicle_number: str,
        vehicle_type: str,
        entry_time: datetime
    ) -> Dict:
        """Process vehicle entry."""

        async with self.db.transaction():
            # Find or create vehicle
            vehicle = await self.db.fetchrow(
                "SELECT id, vehicle_type FROM vehicles WHERE license_plate = $1",
                vehicle_number
            )

            if not vehicle:
                vehicle_id = uuid4()
                await self.db.execute(
                    """
                    INSERT INTO vehicles (id, license_plate, vehicle_type)
                    VALUES ($1, $2, $3)
                    """,
                    vehicle_id, vehicle_number, vehicle_type
                )
            else:
                vehicle_id = vehicle['id']

            # Find available spot
            spot = await self._find_and_assign_spot(vehicle_type)

            if not spot:
                raise ValueError(f"No available spots for vehicle type: {vehicle_type}")

            # Generate ticket
            ticket_number = await self.ticket_generator.generate_ticket_number(entry_time)
            ticket_id = uuid4()

            await self.db.execute(
                """
                INSERT INTO tickets
                (id, ticket_number, vehicle_id, spot_id, entry_time, status)
                VALUES ($1, $2, $3, $4, $5, 'active')
                """,
                ticket_id, ticket_number, vehicle_id, spot['id'], entry_time
            )

            # Generate QR code
            qr_code = self.ticket_generator.generate_qr_code(ticket_number)

            return {
                'ticket_id': str(ticket_id),
                'ticket_number': ticket_number,
                'vehicle_number': vehicle_number,
                'spot': {
                    'id': str(spot['id']),
                    'floor': spot['floor_number'],
                    'spot_number': spot['spot_number'],
                    'spot_type': spot['spot_type'],
                    'location': f"Floor {spot['floor_number']}, Spot {spot['spot_number']}"
                },
                'entry_time': entry_time,
                'qr_code': qr_code
            }

    async def _find_and_assign_spot(self, vehicle_type: str) -> Optional[Dict]:
        """Find and atomically assign a parking spot."""

        # Map vehicle to acceptable spot types
        SPOT_MAPPING = {
            'motorcycle': ['compact', 'regular'],
            'car': ['regular', 'large'],
            'electric_car': ['electric', 'regular', 'large'],
            'suv': ['large', 'regular'],
            'truck': ['large']
        }

        acceptable_types = SPOT_MAPPING.get(vehicle_type, ['regular'])

        # Find and lock spot
        spot = await self.db.fetchrow(
            """
            UPDATE parking_spots
            SET status = 'occupied', updated_at = NOW()
            WHERE id = (
                SELECT ps.id
                FROM parking_spots ps
                JOIN floors f ON ps.floor_id = f.id
                WHERE ps.status = 'available'
                  AND ps.spot_type = ANY($1)
                ORDER BY f.floor_number, ps.spot_number
                LIMIT 1
                FOR UPDATE SKIP LOCKED
            )
            RETURNING
                parking_spots.id,
                parking_spots.spot_number,
                parking_spots.spot_type,
                (SELECT floor_number FROM floors WHERE id = parking_spots.floor_id) as floor_number
            """,
            acceptable_types
        )

        return spot

    async def process_exit(
        self,
        ticket_number: str,
        exit_time: datetime
    ) -> Dict:
        """Process vehicle exit and calculate fee."""

        async with self.db.transaction():
            # Get ticket
            ticket = await self.db.fetchrow(
                """
                SELECT t.id, t.entry_time, t.spot_id, v.vehicle_type, v.license_plate
                FROM tickets t
                JOIN vehicles v ON t.vehicle_id = v.id
                WHERE t.ticket_number = $1 AND t.status = 'active'
                """,
                ticket_number
            )

            if not ticket:
                raise ValueError("Ticket not found or already processed")

            # Calculate fee
            fee = self.fee_calculator.calculate_fee(
                ticket['vehicle_type'],
                ticket['entry_time'],
                exit_time
            )

            # Update ticket
            await self.db.execute(
                """
                UPDATE tickets
                SET exit_time = $1, fee_amount = $2, status = 'paid', updated_at = NOW()
                WHERE id = $3
                """,
                exit_time, fee['total'], ticket['id']
            )

            # Free the spot
            await self.db.execute(
                """
                UPDATE parking_spots
                SET status = 'available', updated_at = NOW()
                WHERE id = $1
                """,
                ticket['spot_id']
            )

            # Invalidate cache
            await self.redis.delete("parking:available:*")

            return {
                'ticket_number': ticket_number,
                'vehicle_number': ticket['license_plate'],
                'entry_time': ticket['entry_time'],
                'exit_time': exit_time,
                'fee': fee
            }

class TicketService:
    """Ticket management service."""

    def __init__(self, db: asyncpg.Connection, redis_client: redis.Redis):
        self.db = db
        self.redis = redis_client

    async def get_ticket(self, ticket_number: str) -> Optional[Dict]:
        """Get ticket details."""
        ticket = await self.db.fetchrow(
            """
            SELECT
                t.ticket_number, t.entry_time, t.exit_time, t.fee_amount, t.status,
                v.license_plate, v.vehicle_type,
                ps.spot_number, ps.spot_type,
                f.floor_number
            FROM tickets t
            JOIN vehicles v ON t.vehicle_id = v.id
            JOIN parking_spots ps ON t.spot_id = ps.id
            JOIN floors f ON ps.floor_id = f.id
            WHERE t.ticket_number = $1
            """,
            ticket_number
        )

        return dict(ticket) if ticket else None

class PaymentService:
    """Payment processing service."""

    def __init__(self, db: asyncpg.Connection):
        self.db = db

    async def process_payment(
        self,
        ticket_id: UUID,
        amount: float,
        payment_method: str,
        card_token: Optional[str] = None
    ) -> Dict:
        """Process payment for parking ticket."""

        # In production, integrate with Stripe/Square
        # For now, simulate payment

        payment_id = uuid4()
        transaction_id = f"TXN-{payment_id.hex[:12].upper()}"

        await self.db.execute(
            """
            INSERT INTO payments
            (id, ticket_id, amount, payment_method, status, transaction_id, paid_at)
            VALUES ($1, $2, $3, $4, 'completed', $5, NOW())
            """,
            payment_id, ticket_id, amount, payment_method, transaction_id
        )

        return {
            'payment_id': str(payment_id),
            'status': 'completed',
            'amount': amount,
            'transaction_id': transaction_id,
            'receipt_url': f"/receipts/{payment_id}.pdf"
        }
```

```python
# seed_data.py
"""Seed database with initial parking lot data."""
import asyncpg
import asyncio
from uuid import uuid4

async def seed_database():
    """Create initial parking lot structure."""

    conn = await asyncpg.connect("postgresql://user:pass@localhost/parking")

    try:
        # Create parking lot
        parking_lot_id = uuid4()
        await conn.execute(
            """
            INSERT INTO parking_lots (id, name, address, total_floors, pricing_config)
            VALUES ($1, $2, $3, $4, $5)
            """,
            parking_lot_id,
            "Downtown Parking Garage",
            "123 Main Street, City, State 12345",
            5,
            '{"hourly_rates": {"motorcycle": 2.0, "car": 5.0, "suv": 7.0, "truck": 10.0}, "daily_max": 50.0}'
        )

        print(f"✓ Created parking lot: {parking_lot_id}")

        # Create floors and spots
        spot_types = ['compact', 'regular', 'large', 'electric']
        spot_distribution = [0.3, 0.5, 0.15, 0.05]  # 30% compact, 50% regular, etc.

        for floor_num in range(1, 6):
            # Create floor
            floor_id = uuid4()
            await conn.execute(
                """
                INSERT INTO floors (id, parking_lot_id, floor_number, total_spots, display_name)
                VALUES ($1, $2, $3, $4, $5)
                """,
                floor_id, parking_lot_id, floor_num, 200, f"Floor {floor_num}"
            )

            print(f"✓ Created floor {floor_num}")

            # Create 200 spots per floor
            spot_idx = 0
            for spot_type, distribution in zip(spot_types, spot_distribution):
                num_spots = int(200 * distribution)

                for i in range(num_spots):
                    spot_number = f"{chr(65 + floor_num - 1)}-{spot_idx + 1:03d}"
                    section = chr(65 + (spot_idx // 50))  # A, B, C, D sections

                    await conn.execute(
                        """
                        INSERT INTO parking_spots
                        (id, floor_id, spot_number, spot_type, status, section)
                        VALUES ($1, $2, $3, $4, $5, $6)
                        """,
                        uuid4(), floor_id, spot_number, spot_type, 'available', section
                    )

                    spot_idx += 1

            print(f"✓ Created {spot_idx} spots on floor {floor_num}")

        print("\n✅ Database seeded successfully!")
        print("Total: 5 floors, 1000 parking spots")

    finally:
        await conn.close()

if __name__ == "__main__":
    asyncio.run(seed_database())
```

```bash
# requirements.txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
asyncpg==0.29.0
redis==5.0.1
pydantic==2.5.0
qrcode[pil]==7.4.2
python-multipart==0.0.6
structlog==23.2.0
prometheus-client==0.19.0
```

```bash
# Run the application
pip install -r requirements.txt
python seed_data.py  # Initialize database
uvicorn main:app --reload --port 8000
```

---

## 13. References

### System Design Resources
1. **"System Design Interview" by Alex Xu**
   - Chapter on parking lot system design
   - Scalability patterns

2. **"Designing Data-Intensive Applications" by Martin Kleppmann**
   - Database consistency models
   - Caching strategies

### Technical Documentation
1. **PostgreSQL Documentation**
   - Row-level locking: https://www.postgresql.org/docs/current/explicit-locking.html
   - SKIP LOCKED: https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE

2. **FastAPI Documentation**
   - https://fastapi.tiangolo.com/

3. **Redis Documentation**
   - Caching patterns: https://redis.io/docs/manual/patterns/

### Similar Real-World Systems
1. **ParkWhiz** - Parking reservation platform
2. **SpotHero** - Smart parking solution
3. **Bosch Parking Lot Sensor** - IoT parking management

---

## Interview Tips

### What Interviewers Look For

1. **Clarifying Requirements:**
   - Ask about scale (spots, facilities, traffic)
   - Confirm functional requirements (reservation, payment, etc.)
   - Identify critical vs. nice-to-have features

2. **Concurrency Handling:**
   - Explain how you prevent double-booking
   - Discuss database locking strategies
   - Consider race conditions

3. **Scalability Thinking:**
   - Start simple (single facility)
   - Explain how to scale to multiple facilities
   - Discuss database partitioning

4. **Trade-off Analysis:**
   - Consistency vs. availability (choose consistency for spot assignment)
   - Synchronous vs. asynchronous payment
   - Cache freshness vs. performance

### Common Mistakes to Avoid

1. **Over-engineering:** Don't start with distributed systems for 1000 spots
2. **Ignoring concurrency:** Must handle simultaneous entry/exit
3. **Weak data model:** Parking spots must be first-class entities
4. **No error handling:** What if payment fails? Spot unavailable?

### Time Management (45-minute interview)

- **0-5 min:** Clarify requirements, scope
- **5-10 min:** Back-of-envelope estimation
- **10-20 min:** High-level design + API
- **20-35 min:** Deep dive (spot assignment, concurrency)
- **35-40 min:** Bottlenecks, scalability
- **40-45 min:** Q&A, extensions

---

**Last Updated:** December 2024
**Difficulty:** Beginner
**Estimated Interview Time:** 45-60 minutes
