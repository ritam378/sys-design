# Uber / Ride-Sharing System Design

> **Difficulty:** Advanced
> **Topics:** Geo-location, QuadTree, Real-time Matching, Surge Pricing, ETA Calculation, WebSockets
> **Companies:** Uber, Lyft, DoorDash, Grab, Didi

---

## 1. Problem Statement

Design a **ride-sharing platform** like Uber that connects riders with nearby drivers in real-time.

### Core Features

**Riders:**
- Request a ride (pickup location, destination)
- See nearby drivers on map
- Get fare estimate
- Track driver in real-time
- Rate driver after ride

**Drivers:**
- Go online/offline
- Accept/reject ride requests
- Navigate to pickup/destination
- Complete ride, get payment

**System:**
- Match riders with closest available drivers
- Calculate ETA (estimated time of arrival)
- Dynamic pricing (surge pricing during high demand)
- Real-time location tracking
- Payment processing

---

## 2. Requirements

### Functional Requirements

1. **Rider Flow:**
   - Request ride with pickup and destination
   - View nearby available drivers on map
   - Receive driver assignment and ETA
   - Track driver location in real-time
   - Pay for ride, rate driver

2. **Driver Flow:**
   - Toggle online/offline status
   - Receive ride requests
   - Accept/reject requests (timeout if no response)
   - Navigate to pickup, then destination
   - Complete ride, receive payment

3. **Matching Algorithm:**
   - Find closest available drivers (within radius)
   - Assign driver based on proximity, rating, acceptance rate
   - Handle rejections (reassign to next driver)

4. **Pricing:**
   - Base fare + distance + time
   - Surge pricing during high demand
   - Fare estimate before ride

5. **Real-time Tracking:**
   - Driver location updates every 3-5 seconds
   - Rider sees driver approaching
   - ETA updates dynamically

### Non-Functional Requirements

1. **Scalability:**
   - 500M riders, 10M drivers
   - 10M active rides per day
   - 1M concurrent rides during peak

2. **Availability:**
   - 99.99% uptime (critical for safety)
   - Graceful degradation (show cached driver locations if real-time fails)

3. **Low Latency:**
   - Ride request to driver match: <5 seconds
   - Location update latency: <1 second
   - ETA calculation: <2 seconds

4. **Accuracy:**
   - Driver location accurate within 10 meters
   - ETA accurate within 20% (traffic considered)

### Out of Scope

- Multi-stop rides
- Ride scheduling (book for later)
- Carpooling (UberPool)
- Food delivery (Uber Eats)
- Driver background checks, onboarding

---

## 3. Back-of-Envelope Estimation

### Traffic

**Assumptions:**
- 500M riders globally
- 10M active drivers
- 10M rides per day
- Peak: 1M concurrent rides

**Requests:**

```
Ride requests:
  10M rides/day = 10M / 86400 = ~120 rides/sec
  Peak (3x): 360 rides/sec

Location updates (drivers):
  10M drivers × 30% active = 3M active drivers
  Update every 4 seconds = 3M / 4 = 750K updates/sec

WebSocket connections:
  1M concurrent rides × 2 (rider + driver) = 2M WebSocket connections
```

### Storage

**Driver locations (hot data):**

```
Per driver location:
  driver_id: 8 bytes (BIGINT)
  lat: 8 bytes (DOUBLE)
  lng: 8 bytes (DOUBLE)
  timestamp: 8 bytes
  heading: 2 bytes
  speed: 2 bytes
  Total: ~36 bytes

Active drivers: 3M × 36 bytes = 108 MB  (easily fits in memory)
```

**Ride data (persistent):**

```
Per ride:
  ride_id, rider_id, driver_id, timestamps, locations, fare: ~500 bytes

Daily: 10M rides × 500 bytes = 5 GB/day
Yearly: 5 GB × 365 = 1.8 TB/year
With replication (3x): 5.4 TB/year
```

**Trip history (cold storage):**

```
5 years of data: 1.8 TB × 5 = 9 TB
Compressed (S3): ~3 TB
```

### Bandwidth

**Location updates:**

```
Incoming (drivers → server):
  750K updates/sec × 36 bytes = 27 MB/sec = 216 Mbps

Outgoing (server → riders):
  1M riders tracking × 36 bytes × 0.25 updates/sec = 9 MB/sec = 72 Mbps

Total: ~300 Mbps (manageable)
```

---

## 4. High-Level Design

### Architecture Diagram

```
┌─────────────┐          ┌─────────────┐
│   Rider     │          │   Driver    │
│   Mobile    │          │   Mobile    │
└──────┬──────┘          └──────┬──────┘
       │                        │
       │  HTTPS/WebSocket       │  HTTPS/WebSocket
       │                        │
       └────────┬───────────────┘
                │
         ┌──────▼──────────┐
         │   API Gateway   │  (Load Balancer, Rate Limiting)
         │   (Kong/Nginx)  │
         └────────┬────────┘
                  │
      ┌───────────┼───────────┬──────────────┬────────────┐
      │           │           │              │            │
      ▼           ▼           ▼              ▼            ▼
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│  Rider   │ │  Driver  │ │ Matching │ │Location  │ │ Payment  │
│ Service  │ │ Service  │ │ Service  │ │ Service  │ │ Service  │
└────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘
     │            │            │            │            │
     └────────────┴────────────┴────────────┴────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
        ┌──────────┐     ┌──────────┐    ┌──────────┐
        │PostgreSQL│     │  Redis   │    │ Kafka    │
        │(Rides DB)│     │(Location)│    │(Events)  │
        └──────────┘     └──────────┘    └──────────┘
                              │
                              ▼
                         ┌──────────┐
                         │ QuadTree │  (In-memory geo-spatial index)
                         │ (Redis)  │
                         └──────────┘
```

### Key Components

1. **API Gateway** - Load balancing, authentication, rate limiting
2. **Rider Service** - Ride requests, tracking, ratings
3. **Driver Service** - Driver status, ride acceptance, navigation
4. **Matching Service** - Match riders with drivers using QuadTree
5. **Location Service** - Track driver locations in real-time
6. **Payment Service** - Fare calculation, surge pricing, transactions
7. **Notification Service** - Push notifications, SMS
8. **Redis** - Cache driver locations, QuadTree index
9. **Kafka** - Event streaming (ride events, location updates)
10. **PostgreSQL** - Persistent storage (users, rides, payments)

---

## 5. Detailed Design

### 5.1 Geo-Spatial Indexing (QuadTree)

**Problem:** How to efficiently find nearby drivers?

**Naive Approach (Too Slow):**

```python
# Query all drivers, calculate distance to rider
SELECT driver_id, lat, lng
FROM drivers
WHERE status = 'available';

# In application:
for driver in drivers:
    distance = haversine(rider_lat, rider_lng, driver.lat, driver.lng)
    if distance < 5km:
        nearby_drivers.append(driver)

# ❌ O(N) for N drivers (millions of calculations!)
```

**QuadTree Approach (Fast):**

**QuadTree** recursively divides 2D space into quadrants.

**Structure:**

```
Root (entire world)
├── NW (Northwest quadrant)
│   ├── NW
│   │   ├── Drivers: [D1, D5]
│   │   └── ...
│   ├── NE
│   ├── SW
│   └── SE
├── NE (Northeast quadrant)
├── SW (Southwest quadrant)
└── SE (Southeast quadrant)
```

**Example:**

```
World Map:
┌───────────────────┬───────────────────┐
│        NW         │        NE         │
│                   │                   │
│  ┌─────┬─────┐    │                   │
│  │ NW  │ NE  │    │    D7             │
│  ├─────┼─────┤    │                   │
│  │D1,D5│  D2 │    │                   │
│  │ SW  │ SE  │    │                   │
│  └─────┴─────┘    │                   │
│       D3          │                   │
├───────────────────┼───────────────────┤
│        SW         │        SE         │
│                   │                   │
│    D4             │      D6           │
│                   │                   │
└───────────────────┴───────────────────┘

Rider R requests ride at (lat, lng) in NW.NW quadrant.

QuadTree search:
  1. Start at root → go to NW
  2. NW → go to NW (contains R)
  3. Find drivers in NW.NW and neighboring quadrants: [D1, D5, D2, D3]
  4. Calculate distance only for these 4 drivers (vs. all 7)

✅ O(log N) to find quadrant + O(k) for k nearby drivers
```

**Implementation:**

```python
from typing import List, Optional
from dataclasses import dataclass
import math

@dataclass
class Location:
    lat: float
    lng: float

@dataclass
class Driver:
    driver_id: str
    location: Location
    status: str  # 'available', 'busy', 'offline'

class BoundingBox:
    """Represents a geographic rectangle"""
    def __init__(self, min_lat: float, max_lat: float, min_lng: float, max_lng: float):
        self.min_lat = min_lat
        self.max_lat = max_lat
        self.min_lng = min_lng
        self.max_lng = max_lng

    def contains(self, location: Location) -> bool:
        """Check if location is within this box"""
        return (self.min_lat <= location.lat <= self.max_lat and
                self.min_lng <= location.lng <= self.max_lng)

    def intersects(self, other: 'BoundingBox') -> bool:
        """Check if this box intersects with another"""
        return not (other.max_lat < self.min_lat or
                    other.min_lat > self.max_lat or
                    other.max_lng < self.min_lng or
                    other.min_lng > self.max_lng)

class QuadTreeNode:
    """
    QuadTree node for efficient geo-spatial indexing

    Each node represents a geographic region and can contain:
    - Up to MAX_CAPACITY drivers
    - 4 child quadrants (NW, NE, SW, SE) if capacity exceeded
    """

    MAX_CAPACITY = 50  # Max drivers per leaf node
    MAX_DEPTH = 10     # Max tree depth

    def __init__(self, boundary: BoundingBox, depth: int = 0):
        self.boundary = boundary
        self.depth = depth
        self.drivers: List[Driver] = []
        self.divided = False

        # Child quadrants (None until subdivision)
        self.nw: Optional[QuadTreeNode] = None
        self.ne: Optional[QuadTreeNode] = None
        self.sw: Optional[QuadTreeNode] = None
        self.se: Optional[QuadTreeNode] = None

    def insert(self, driver: Driver) -> bool:
        """Insert driver into QuadTree"""
        # Ignore if driver outside boundary
        if not self.boundary.contains(driver.location):
            return False

        # If capacity not exceeded, add here
        if len(self.drivers) < self.MAX_CAPACITY and not self.divided:
            self.drivers.append(driver)
            return True

        # Subdivide if not already divided
        if not self.divided and self.depth < self.MAX_DEPTH:
            self.subdivide()

        # Insert into appropriate child quadrant
        if self.divided:
            return (self.nw.insert(driver) or
                    self.ne.insert(driver) or
                    self.sw.insert(driver) or
                    self.se.insert(driver))

        # Max depth reached, force insert
        self.drivers.append(driver)
        return True

    def subdivide(self):
        """Split this node into 4 quadrants"""
        mid_lat = (self.boundary.min_lat + self.boundary.max_lat) / 2
        mid_lng = (self.boundary.min_lng + self.boundary.max_lng) / 2

        # Create 4 child quadrants
        self.nw = QuadTreeNode(
            BoundingBox(mid_lat, self.boundary.max_lat, self.boundary.min_lng, mid_lng),
            self.depth + 1
        )
        self.ne = QuadTreeNode(
            BoundingBox(mid_lat, self.boundary.max_lat, mid_lng, self.boundary.max_lng),
            self.depth + 1
        )
        self.sw = QuadTreeNode(
            BoundingBox(self.boundary.min_lat, mid_lat, self.boundary.min_lng, mid_lng),
            self.depth + 1
        )
        self.se = QuadTreeNode(
            BoundingBox(self.boundary.min_lat, mid_lat, mid_lng, self.boundary.max_lng),
            self.depth + 1
        )

        # Move existing drivers to child quadrants
        for driver in self.drivers:
            self.nw.insert(driver) or self.ne.insert(driver) or \
            self.sw.insert(driver) or self.se.insert(driver)

        self.drivers = []  # Clear parent node
        self.divided = True

    def query(self, search_area: BoundingBox) -> List[Driver]:
        """Find all drivers within search area"""
        found = []

        # No overlap, return empty
        if not self.boundary.intersects(search_area):
            return found

        # Check drivers in this node
        for driver in self.drivers:
            if search_area.contains(driver.location):
                found.append(driver)

        # Recursively search child quadrants
        if self.divided:
            found.extend(self.nw.query(search_area))
            found.extend(self.ne.query(search_area))
            found.extend(self.sw.query(search_area))
            found.extend(self.se.query(search_area))

        return found

def haversine(lat1: float, lng1: float, lat2: float, lng2: float) -> float:
    """
    Calculate distance between two points on Earth using Haversine formula

    Returns: distance in kilometers
    """
    R = 6371  # Earth's radius in km

    # Convert to radians
    lat1_rad = math.radians(lat1)
    lat2_rad = math.radians(lat2)
    delta_lat = math.radians(lat2 - lat1)
    delta_lng = math.radians(lng2 - lng1)

    # Haversine formula
    a = (math.sin(delta_lat / 2) ** 2 +
         math.cos(lat1_rad) * math.cos(lat2_rad) *
         math.sin(delta_lng / 2) ** 2)
    c = 2 * math.asin(math.sqrt(a))

    return R * c

class DriverMatcher:
    """Matches riders with nearby available drivers using QuadTree"""

    def __init__(self):
        # QuadTree covering entire world
        world_boundary = BoundingBox(
            min_lat=-90, max_lat=90,
            min_lng=-180, max_lng=180
        )
        self.quadtree = QuadTreeNode(world_boundary)

    def update_driver_location(self, driver: Driver):
        """Update driver's location in QuadTree"""
        # In production: Remove old location, insert new
        # For simplicity: Rebuild tree periodically
        self.quadtree.insert(driver)

    def find_nearby_drivers(
        self,
        rider_location: Location,
        radius_km: float = 5.0,
        max_drivers: int = 10
    ) -> List[Driver]:
        """
        Find available drivers within radius of rider

        Args:
            rider_location: Rider's current location
            radius_km: Search radius in kilometers
            max_drivers: Maximum number of drivers to return

        Returns:
            List of nearby available drivers, sorted by distance
        """
        # Convert radius to lat/lng degrees (approximation)
        # 1 degree lat ≈ 111 km
        # 1 degree lng ≈ 111 km * cos(lat)
        lat_delta = radius_km / 111.0
        lng_delta = radius_km / (111.0 * math.cos(math.radians(rider_location.lat)))

        # Create search bounding box
        search_area = BoundingBox(
            min_lat=rider_location.lat - lat_delta,
            max_lat=rider_location.lat + lat_delta,
            min_lng=rider_location.lng - lng_delta,
            max_lng=rider_location.lng + lng_delta
        )

        # Query QuadTree
        candidates = self.quadtree.query(search_area)

        # Filter: only available drivers within exact radius
        nearby = []
        for driver in candidates:
            if driver.status != 'available':
                continue

            distance = haversine(
                rider_location.lat, rider_location.lng,
                driver.location.lat, driver.location.lng
            )

            if distance <= radius_km:
                nearby.append((driver, distance))

        # Sort by distance
        nearby.sort(key=lambda x: x[1])

        # Return top N drivers
        return [driver for driver, _ in nearby[:max_drivers]]

# Example Usage
if __name__ == "__main__":
    matcher = DriverMatcher()

    # Add drivers to QuadTree
    drivers = [
        Driver("D1", Location(37.7749, -122.4194), "available"),  # SF
        Driver("D2", Location(37.7849, -122.4094), "available"),  # 1 km from D1
        Driver("D3", Location(37.7649, -122.4294), "busy"),       # Busy
        Driver("D4", Location(40.7128, -74.0060), "available"),   # NYC (far away)
    ]

    for driver in drivers:
        matcher.update_driver_location(driver)

    # Rider in SF requests ride
    rider = Location(37.7750, -122.4195)
    nearby_drivers = matcher.find_nearby_drivers(rider, radius_km=5.0)

    print(f"Found {len(nearby_drivers)} nearby drivers:")
    for driver in nearby_drivers:
        distance = haversine(rider.lat, rider.lng, driver.location.lat, driver.location.lng)
        print(f"  {driver.driver_id}: {distance:.2f} km away")

# Output:
# Found 2 nearby drivers:
#   D1: 0.01 km away
#   D2: 1.23 km away
# (D3 excluded: busy, D4 excluded: too far)
```

**Optimizations:**

1. **Rebuild tree periodically** (every 30 seconds):
   - Drivers move constantly
   - Rebuilding ensures accuracy
   - Alternative: Track driver movements, update incrementally

2. **Cache in Redis:**
   - Serialize QuadTree to Redis
   - Each geo-region can have its own tree (US-West, EU, Asia)

3. **Geohash alternative:**
   - Use Geohash (string encoding of lat/lng)
   - Redis GEO commands: `GEOADD`, `GEORADIUS`

---

### 5.2 Ride Matching Algorithm

**Flow:**

```
1. Rider requests ride
   ↓
2. Find nearby available drivers (QuadTree)
   ↓
3. Rank drivers by:
   - Distance (closest first)
   - Rating (higher better)
   - Acceptance rate (higher better)
   ↓
4. Send request to top driver
   ↓
5. Driver has 30 seconds to accept/reject
   ↓
6. If accept: Assign ride, notify rider
   If reject/timeout: Send to next driver
   ↓
7. Repeat until match found or no drivers left
```

**Implementation:**

```python
import asyncio
from enum import Enum
from typing import Optional
import time

class RideStatus(Enum):
    REQUESTED = "requested"
    DRIVER_ASSIGNED = "driver_assigned"
    DRIVER_ARRIVED = "driver_arrived"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    CANCELLED = "cancelled"

class Ride:
    def __init__(self, ride_id: str, rider_id: str, pickup: Location, destination: Location):
        self.ride_id = ride_id
        self.rider_id = rider_id
        self.pickup = pickup
        self.destination = destination
        self.driver_id: Optional[str] = None
        self.status = RideStatus.REQUESTED
        self.created_at = time.time()
        self.fare: Optional[float] = None

class MatchingService:
    """Matches riders with drivers"""

    ACCEPT_TIMEOUT = 30  # seconds

    def __init__(self, driver_matcher: DriverMatcher):
        self.driver_matcher = driver_matcher
        self.active_requests = {}  # ride_id → pending request

    async def request_ride(self, ride: Ride) -> bool:
        """
        Request a ride, find and assign driver

        Returns: True if driver assigned, False if no drivers available
        """
        # Find nearby drivers
        nearby_drivers = self.driver_matcher.find_nearby_drivers(
            ride.pickup,
            radius_km=5.0,
            max_drivers=10
        )

        if not nearby_drivers:
            return False  # No drivers available

        # Try each driver in order
        for driver in nearby_drivers:
            assigned = await self._try_assign_driver(ride, driver)
            if assigned:
                return True

        return False  # All drivers rejected

    async def _try_assign_driver(self, ride: Ride, driver: Driver) -> bool:
        """
        Send ride request to driver, wait for acceptance

        Returns: True if driver accepted, False if rejected/timeout
        """
        # Send notification to driver
        await self._notify_driver(driver.driver_id, ride)

        # Wait for response (with timeout)
        try:
            response = await asyncio.wait_for(
                self._wait_for_driver_response(ride.ride_id, driver.driver_id),
                timeout=self.ACCEPT_TIMEOUT
            )

            if response == "accept":
                # Assign driver to ride
                ride.driver_id = driver.driver_id
                ride.status = RideStatus.DRIVER_ASSIGNED
                driver.status = "busy"

                # Notify rider
                await self._notify_rider(ride.rider_id, f"Driver {driver.driver_id} assigned!")

                return True
            else:
                # Driver rejected
                return False

        except asyncio.TimeoutError:
            # Driver didn't respond in time
            return False

    async def _notify_driver(self, driver_id: str, ride: Ride):
        """Send ride request to driver (push notification + WebSocket)"""
        # In production: Send via WebSocket or push notification
        print(f"[Notify] Driver {driver_id}: New ride request {ride.ride_id}")

    async def _notify_rider(self, rider_id: str, message: str):
        """Send update to rider"""
        print(f"[Notify] Rider {rider_id}: {message}")

    async def _wait_for_driver_response(self, ride_id: str, driver_id: str) -> str:
        """Wait for driver to accept/reject (simulated)"""
        # In production: Listen to driver's response via WebSocket/API
        await asyncio.sleep(5)  # Simulate driver thinking
        return "accept"  # Simplified: always accept

# Example Usage
async def main():
    matcher = DriverMatcher()

    # Add drivers
    matcher.update_driver_location(
        Driver("D1", Location(37.7749, -122.4194), "available")
    )
    matcher.update_driver_location(
        Driver("D2", Location(37.7849, -122.4094), "available")
    )

    # Rider requests ride
    ride = Ride(
        ride_id="R123",
        rider_id="U456",
        pickup=Location(37.7750, -122.4195),
        destination=Location(37.8044, -122.2711)  # Oakland
    )

    matching_service = MatchingService(matcher)
    success = await matching_service.request_ride(ride)

    if success:
        print(f"Ride {ride.ride_id} assigned to driver {ride.driver_id}")
    else:
        print(f"No drivers available for ride {ride.ride_id}")

# Run
# asyncio.run(main())
```

---

### 5.3 ETA Calculation

**Estimate time for driver to reach pickup location.**

**Factors:**

1. **Distance** - Straight-line vs. road distance
2. **Traffic** - Current traffic conditions
3. **Speed** - Average speed for road type

**Approaches:**

**1. Simple (Haversine + Average Speed):**

```python
def calculate_simple_eta(driver_loc: Location, pickup_loc: Location) -> int:
    """
    Simple ETA: straight-line distance / average speed

    Returns: ETA in seconds
    """
    distance_km = haversine(driver_loc.lat, driver_loc.lng, pickup_loc.lat, pickup_loc.lng)

    # Assume average city speed: 30 km/h
    AVERAGE_SPEED_KMH = 30
    time_hours = distance_km / AVERAGE_SPEED_KMH
    time_seconds = int(time_hours * 3600)

    return time_seconds

# Example:
driver = Location(37.7749, -122.4194)
pickup = Location(37.7849, -122.4094)
eta = calculate_simple_eta(driver, pickup)
print(f"ETA: {eta // 60} minutes")  # ~3 minutes
```

**2. Advanced (Google Maps / External API):**

```python
import requests

async def calculate_accurate_eta(driver_loc: Location, pickup_loc: Location) -> int:
    """
    Accurate ETA using Google Maps Directions API

    Considers:
    - Actual road network
    - Current traffic
    - Road restrictions

    Returns: ETA in seconds
    """
    # Google Maps Directions API
    url = "https://maps.googleapis.com/maps/api/directions/json"
    params = {
        "origin": f"{driver_loc.lat},{driver_loc.lng}",
        "destination": f"{pickup_loc.lat},{pickup_loc.lng}",
        "mode": "driving",
        "departure_time": "now",  # Use current traffic
        "key": "YOUR_API_KEY"
    }

    response = requests.get(url, params=params)
    data = response.json()

    if data["status"] == "OK":
        route = data["routes"][0]["legs"][0]
        duration_seconds = route["duration_in_traffic"]["value"]
        return duration_seconds
    else:
        # Fallback to simple ETA
        return calculate_simple_eta(driver_loc, pickup_loc)
```

**3. Machine Learning (Uber's Approach):**

```python
# Uber uses ML models trained on historical data:
# - Time of day
# - Day of week
# - Weather
# - Historical traffic patterns
# - Events (concerts, sports games)

def calculate_ml_eta(driver_loc, pickup_loc, timestamp, weather, events):
    # Features
    features = [
        haversine(driver_loc, pickup_loc),
        timestamp.hour,
        timestamp.weekday(),
        weather['rain'],  # Boolean
        len(events),  # Number of nearby events
        # ... more features
    ]

    # ML model (trained offline)
    eta = ml_model.predict(features)
    return eta
```

---

### 5.4 Surge Pricing

**Dynamic pricing based on supply and demand.**

**Algorithm:**

```python
class SurgePricingCalculator:
    """Calculate surge multiplier based on demand/supply"""

    def calculate_surge(self, zone: str, timestamp: float) -> float:
        """
        Calculate surge multiplier for a geographic zone

        Args:
            zone: Geographic zone ID (e.g., "SF_downtown")
            timestamp: Current time

        Returns:
            Surge multiplier (1.0 = no surge, 2.5 = 2.5x price)
        """
        # Get recent ride requests and available drivers in zone
        ride_requests = self._count_recent_requests(zone, window_minutes=10)
        available_drivers = self._count_available_drivers(zone)

        if available_drivers == 0:
            return 3.0  # Max surge (no drivers!)

        # Demand/supply ratio
        demand_supply_ratio = ride_requests / available_drivers

        # Surge tiers
        if demand_supply_ratio < 1.0:
            return 1.0  # No surge (plenty of drivers)
        elif demand_supply_ratio < 2.0:
            return 1.5  # Mild surge
        elif demand_supply_ratio < 3.0:
            return 2.0  # Moderate surge
        elif demand_supply_ratio < 5.0:
            return 2.5  # High surge
        else:
            return 3.0  # Extreme surge

    def _count_recent_requests(self, zone: str, window_minutes: int) -> int:
        """Count ride requests in zone within time window"""
        # In production: Query from Redis (time-series data)
        # Simplified:
        return 50  # 50 requests in last 10 min

    def _count_available_drivers(self, zone: str) -> int:
        """Count available drivers in zone"""
        # In production: Query from QuadTree/Redis
        return 20  # 20 available drivers

def calculate_fare(
    distance_km: float,
    duration_minutes: int,
    surge_multiplier: float
) -> float:
    """
    Calculate ride fare

    Formula: (Base + Distance × Rate + Time × Rate) × Surge
    """
    BASE_FARE = 2.50        # $2.50
    COST_PER_KM = 1.50      # $1.50/km
    COST_PER_MIN = 0.30     # $0.30/min

    base_fare = BASE_FARE
    distance_fare = distance_km * COST_PER_KM
    time_fare = duration_minutes * COST_PER_MIN

    total = (base_fare + distance_fare + time_fare) * surge_multiplier

    return round(total, 2)

# Example:
surge_calculator = SurgePricingCalculator()
surge = surge_calculator.calculate_surge("SF_downtown", time.time())

fare = calculate_fare(
    distance_km=10,
    duration_minutes=20,
    surge_multiplier=surge
)

print(f"Surge: {surge}x")
print(f"Fare: ${fare}")

# Output:
# Surge: 2.5x
# Fare: $56.25  (normally $22.50, but 2.5x surge!)
```

---

### 5.5 Real-Time Location Tracking

**WebSocket-based location updates.**

**Driver App:**

```python
# Driver sends location every 4 seconds
import asyncio
import websockets
import json

async def driver_location_sender(driver_id: str):
    uri = f"ws://location-service.uber.com/driver/{driver_id}"

    async with websockets.connect(uri) as websocket:
        while True:
            # Get current GPS location
            current_location = get_gps_location()  # From device GPS

            # Send to server
            message = {
                "driver_id": driver_id,
                "lat": current_location.lat,
                "lng": current_location.lng,
                "heading": 45,  # Degrees
                "speed": 12.5,  # km/h
                "timestamp": time.time()
            }

            await websocket.send(json.dumps(message))

            # Wait 4 seconds
            await asyncio.sleep(4)
```

**Location Service (Server):**

```python
import asyncio
import websockets
import json

class LocationService:
    """Receives driver locations, updates QuadTree, broadcasts to riders"""

    def __init__(self):
        self.driver_matcher = DriverMatcher()
        self.rider_connections = {}  # rider_id → WebSocket

    async def handle_driver_connection(self, websocket, driver_id: str):
        """Handle driver WebSocket connection"""
        async for message in websocket:
            data = json.loads(message)

            # Update driver location in QuadTree
            driver = Driver(
                driver_id=data["driver_id"],
                location=Location(data["lat"], data["lng"]),
                status="busy" if data.get("on_ride") else "available"
            )
            self.driver_matcher.update_driver_location(driver)

            # If driver is on a ride, broadcast location to rider
            if data.get("ride_id"):
                await self.broadcast_to_rider(data["ride_id"], data)

    async def broadcast_to_rider(self, ride_id: str, location_data: dict):
        """Send driver location to rider tracking this ride"""
        rider_id = self._get_rider_for_ride(ride_id)

        if rider_id in self.rider_connections:
            rider_ws = self.rider_connections[rider_id]
            await rider_ws.send(json.dumps(location_data))

    def _get_rider_for_ride(self, ride_id: str) -> str:
        # In production: Query from database
        return "U456"  # Simplified

# Start WebSocket server
async def main():
    location_service = LocationService()

    async def driver_handler(websocket, path):
        driver_id = path.split("/")[-1]  # Extract driver_id from URL
        await location_service.handle_driver_connection(websocket, driver_id)

    async with websockets.serve(driver_handler, "0.0.0.0", 8765):
        await asyncio.Future()  # Run forever

# asyncio.run(main())
```

**Rider App (Receives Updates):**

```python
# Rider receives driver location updates
async def rider_tracking_receiver(ride_id: str):
    uri = f"ws://location-service.uber.com/rider/{ride_id}"

    async with websockets.connect(uri) as websocket:
        async for message in websocket:
            data = json.loads(message)

            # Update map with driver's location
            print(f"Driver at ({data['lat']}, {data['lng']})")
            print(f"Speed: {data['speed']} km/h")

            # Recalculate ETA
            eta = calculate_simple_eta(
                Location(data['lat'], data['lng']),
                pickup_location
            )
            print(f"ETA: {eta // 60} minutes")
```

---

## 6. Database Schema

### PostgreSQL (Persistent Data)

```sql
-- Users (riders and drivers)
CREATE TABLE users (
    user_id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    user_type VARCHAR(10) NOT NULL,  -- 'rider' or 'driver'
    rating DECIMAL(3, 2),  -- Average rating (0.00 - 5.00)
    created_at TIMESTAMP DEFAULT NOW()
);

-- Drivers (additional info)
CREATE TABLE drivers (
    driver_id BIGINT PRIMARY KEY REFERENCES users(user_id),
    license_plate VARCHAR(20) NOT NULL,
    vehicle_model VARCHAR(50) NOT NULL,
    vehicle_color VARCHAR(20) NOT NULL,
    status VARCHAR(20) DEFAULT 'offline',  -- 'available', 'busy', 'offline'
    current_lat DOUBLE PRECISION,
    current_lng DOUBLE PRECISION,
    last_location_update TIMESTAMP
);

-- Rides
CREATE TABLE rides (
    ride_id BIGSERIAL PRIMARY KEY,
    rider_id BIGINT NOT NULL REFERENCES users(user_id),
    driver_id BIGINT REFERENCES users(user_id),
    status VARCHAR(20) NOT NULL,  -- 'requested', 'driver_assigned', 'in_progress', 'completed', 'cancelled'

    -- Locations
    pickup_lat DOUBLE PRECISION NOT NULL,
    pickup_lng DOUBLE PRECISION NOT NULL,
    dropoff_lat DOUBLE PRECISION NOT NULL,
    dropoff_lng DOUBLE PRECISION NOT NULL,

    -- Pricing
    fare DECIMAL(10, 2),
    surge_multiplier DECIMAL(3, 2) DEFAULT 1.0,

    -- Timestamps
    created_at TIMESTAMP DEFAULT NOW(),
    driver_assigned_at TIMESTAMP,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,

    -- Ratings
    rider_rating SMALLINT,  -- 1-5
    driver_rating SMALLINT,  -- 1-5

    INDEX idx_rider_id (rider_id),
    INDEX idx_driver_id (driver_id),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
);

-- Payments
CREATE TABLE payments (
    payment_id BIGSERIAL PRIMARY KEY,
    ride_id BIGINT NOT NULL REFERENCES rides(ride_id),
    amount DECIMAL(10, 2) NOT NULL,
    payment_method VARCHAR(20) NOT NULL,  -- 'credit_card', 'paypal', 'cash'
    status VARCHAR(20) NOT NULL,  -- 'pending', 'completed', 'failed'
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Redis (Real-Time Data)

```
# Driver locations (expires after 30 seconds if not updated)
GEOADD drivers:locations <lng> <lat> driver_id
GEORADIUS drivers:locations <lng> <lat> 5 km WITHDIST

# Driver status
HSET driver:D123 status "available"
HSET driver:D123 lat "37.7749"
HSET driver:D123 lng "-122.4194"
EXPIRE driver:D123 30

# Active rides (for quick lookups)
HSET ride:R456 driver_id "D123"
HSET ride:R456 rider_id "U789"
HSET ride:R456 status "in_progress"

# Surge pricing (by zone)
HSET surge:SF_downtown multiplier "2.5"
EXPIRE surge:SF_downtown 300  # Recalculate every 5 min
```

---

## 7. API Design

### Rider APIs

```python
# Request a ride
POST /v1/rides
{
  "rider_id": "U456",
  "pickup": {"lat": 37.7749, "lng": -122.4194},
  "destination": {"lat": 37.8044, "lng": -122.2711}
}

Response 201:
{
  "ride_id": "R789",
  "status": "requested",
  "estimated_fare": 22.50,
  "surge_multiplier": 1.0
}

# Get ride status
GET /v1/rides/{ride_id}

Response 200:
{
  "ride_id": "R789",
  "status": "driver_assigned",
  "driver": {
    "driver_id": "D123",
    "name": "John Doe",
    "vehicle": "Toyota Prius - White",
    "rating": 4.8,
    "current_location": {"lat": 37.7750, "lng": -122.4200}
  },
  "eta_seconds": 180
}

# Cancel ride
PUT /v1/rides/{ride_id}/cancel

# Rate driver
POST /v1/rides/{ride_id}/rating
{
  "rating": 5,
  "feedback": "Great driver!"
}
```

### Driver APIs

```python
# Update status (go online/offline)
PUT /v1/drivers/{driver_id}/status
{
  "status": "available"  # or "offline"
}

# Accept ride
PUT /v1/rides/{ride_id}/accept
{
  "driver_id": "D123"
}

# Reject ride
PUT /v1/rides/{ride_id}/reject
{
  "driver_id": "D123"
}

# Complete ride
PUT /v1/rides/{ride_id}/complete
{
  "driver_id": "D123",
  "final_location": {"lat": 37.8044, "lng": -122.2711}
}
```

---

## 8. System Optimizations

### 1. Geo-Sharding

**Shard by geographic region:**

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  US-West    │  │  US-East    │  │  EU-West    │
│  Shard      │  │  Shard      │  │  Shard      │
│             │  │             │  │             │
│ CA, WA, OR  │  │ NY, MA, FL  │  │ UK, FR, DE  │
└─────────────┘  └─────────────┘  └─────────────┘

Benefits:
- Low latency (query local shard)
- Fault isolation (US outage doesn't affect EU)
- Compliance (GDPR data residency)
```

### 2. Caching

```python
# Cache frequent queries
# - Driver profiles
# - Surge pricing (recalculated every 5 min)
# - Recent ride history

@cache(ttl=300)  # 5 minutes
def get_surge_multiplier(zone: str) -> float:
    return calculate_surge(zone)

@cache(ttl=60)  # 1 minute
def get_nearby_drivers(location: Location) -> List[Driver]:
    return quadtree.query(location, radius=5)
```

### 3. Read Replicas

```
Primary (Writes):
  - Ride creation
  - Payment processing

Replicas (Reads):
  - Ride history
  - Driver profiles
  - Analytics
```

---

## 9. Failure Handling

### Driver Doesn't Accept

```python
# Timeout after 30 seconds → Try next driver
if not driver_accepted:
    next_driver = get_next_best_driver()
    send_request_to_driver(next_driver)
```

### No Drivers Available

```python
# Expand search radius incrementally
radii = [5, 10, 15, 20]  # km
for radius in radii:
    drivers = find_drivers(location, radius)
    if drivers:
        break
else:
    return "No drivers available"
```

### WebSocket Disconnection

```python
# Rider's WebSocket drops
# - Keep sending location updates (buffer)
# - Reconnect automatically
# - Fetch latest state on reconnect

if websocket_disconnected:
    buffer_updates(driver_location)
    # Client reconnects
    send_buffered_updates()
```

---

## 10. Summary

**System Highlights:**

✅ **QuadTree** for O(log n) nearby driver search
✅ **WebSockets** for real-time location tracking
✅ **Surge pricing** based on demand/supply ratio
✅ **ETA calculation** using Haversine + traffic data
✅ **Matching algorithm** with timeout and fallback
✅ **Redis** for low-latency location queries
✅ **Geo-sharding** for global scale and compliance
✅ **Kafka** for event streaming (analytics, audit logs)

**Key Metrics:**

- **Latency:** Driver match < 5s, location update < 1s
- **Scale:** 1M concurrent rides, 750K location updates/sec
- **Availability:** 99.99% uptime
- **Storage:** 1.8 TB/year (rides), 108 MB RAM (active driver locations)

**Trade-offs:**

- **QuadTree rebuild:** Periodic (every 30s) vs. incremental updates
- **ETA accuracy:** Simple (fast) vs. ML-based (accurate but complex)
- **Consistency:** Eventual (high availability) vs. strong (financial data)

**Real-World Examples:**
- Uber uses QuadTree + S2 geometry for geo-indexing
- Lyft uses real-time ML for ETA prediction
- Grab (Southeast Asia) geo-shards by country/city

---

**Next:** Explore [Payment System Design](../payment-system/README.md) for handling ride payments at scale.
