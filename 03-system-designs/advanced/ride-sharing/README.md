# Design a Ride Sharing System

## Table of Contents
- [Problem Statement](#problem-statement)
- [Requirements](#requirements)
- [High-Level Architecture](#high-level-architecture)
- [Location Tracking](#location-tracking)
- [Driver-Rider Matching](#driver-rider-matching)
- [Geospatial Indexing](#geospatial-indexing)
- [Route Optimization](#route-optimization)
- [Surge Pricing](#surge-pricing)
- [Real-Time Updates](#real-time-updates)
- [Trip Management](#trip-management)
- [Payment Integration](#payment-integration)
- [Database Design](#database-design)
- [Scalability](#scalability)
- [Implementation Examples](#implementation-examples)
- [Real-World Examples](#real-world-examples)
- [Interview Tips](#interview-tips)

---

## Problem Statement

Design a **ride-sharing platform** like Uber or Lyft that can:
- Match riders with nearby drivers in real-time
- Track driver and rider locations continuously
- Calculate optimal routes and ETAs
- Handle surge pricing during high demand
- Process millions of rides per day globally
- Provide real-time trip updates

**Similar to**: Uber, Lyft, Didi, Grab, Ola

---

## Requirements

### Functional Requirements

1. **Rider Features**:
   - Request ride with pickup/destination
   - See nearby drivers on map
   - Get fare estimate
   - Track driver in real-time
   - Rate driver after trip

2. **Driver Features**:
   - Go online/offline
   - Receive ride requests
   - Accept/reject rides
   - Navigate to pickup/destination
   - Track earnings

3. **Matching**:
   - Match rider with nearest available driver
   - Optimize for ETA, rating, ride acceptance rate
   - Handle simultaneous requests

4. **Pricing**:
   - Base fare + distance + time
   - Surge pricing during high demand
   - Promo codes and discounts

### Non-Functional Requirements

1. **Low Latency**: <1s for matching, <500ms for location updates
2. **High Availability**: 99.99% uptime
3. **Scalability**: 10M+ concurrent users
4. **Accuracy**: Precise location tracking (±5 meters)
5. **Real-time**: Live location updates every 3-5 seconds
6. **Fault Tolerance**: Handle network issues gracefully

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Mobile Apps                               │
│         ┌──────────────┐      ┌──────────────┐             │
│         │   Rider App  │      │  Driver App  │             │
│         └──────┬───────┘      └──────┬───────┘             │
└────────────────┼──────────────────────┼──────────────────────┘
                 │                      │
                 ↓                      ↓
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway                               │
│          (Load Balancing, Authentication)                    │
└────────────────┬─────────────────────┬──────────────────────┘
                 │                     │
        ┌────────┴────────┐    ┌──────┴────────┐
        ↓                 ↓    ↓               ↓
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Location   │  │   Matching   │  │    Trip      │
│   Service    │  │   Service    │  │   Service    │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                  │
       ↓                 ↓                  ↓
┌─────────────────────────────────────────────────────────────┐
│              WebSocket Service (Real-time)                   │
│          Push location updates, trip status                  │
└─────────────────────────────────────────────────────────────┘
       │                 │                  │
       ↓                 ↓                  ↓
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Redis       │  │  Geospatial  │  │  PostgreSQL  │
│  (Cache)     │  │  Index       │  │  (Trips DB)  │
│              │  │  (QuadTree)  │  │              │
└──────────────┘  └──────────────┘  └──────────────┘
       │
       ↓
┌──────────────────────────────────────────────────────────────┐
│              Kafka (Event Stream)                             │
│  - location_updates                                          │
│  - trip_events                                               │
│  - driver_status_changes                                     │
└──────────────────────────────────────────────────────────────┘
       │
       ↓
┌──────────────────────────────────────────────────────────────┐
│            Downstream Services                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Pricing  │  │ Payment  │  │Analytics │  │  Fraud   │   │
│  │ Service  │  │ Service  │  │ Service  │  │Detection │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└──────────────────────────────────────────────────────────────┘
```

---

## Location Tracking

### GPS Location Updates

```python
from flask import Flask, request
from flask_socketio import SocketIO, emit
import redis
import json
from datetime import datetime

app = Flask(__name__)
socketio = SocketIO(app, cors_allowed_origins="*")
redis_client = redis.Redis(decode_responses=True)

class LocationService:
    """
    Track real-time locations of drivers and riders
    """

    def __init__(self):
        self.redis = redis_client

    def update_driver_location(self, driver_id, latitude, longitude):
        """
        Update driver's current location

        Stored in Redis with TTL for active drivers
        """
        location = {
            'driver_id': driver_id,
            'latitude': latitude,
            'longitude': longitude,
            'timestamp': datetime.utcnow().isoformat(),
            'heading': request.json.get('heading', 0),  # Direction
            'speed': request.json.get('speed', 0)  # km/h
        }

        # Store in Redis Geo (for spatial queries)
        self.redis.geoadd(
            'drivers:online',
            (longitude, latitude, driver_id)
        )

        # Store detailed location data
        self.redis.setex(
            f'driver:{driver_id}:location',
            300,  # 5 min TTL (expire if no updates)
            json.dumps(location)
        )

        # Publish to real-time subscribers
        self.redis.publish(
            f'driver:{driver_id}:location',
            json.dumps(location)
        )

        return location

    def get_nearby_drivers(self, latitude, longitude, radius_km=5):
        """
        Find drivers within radius using Redis GEORADIUS

        Returns list of driver IDs with distances
        """
        results = self.redis.georadius(
            'drivers:online',
            longitude,
            latitude,
            radius_km,
            unit='km',
            withdist=True,
            sort='ASC'  # Nearest first
        )

        nearby_drivers = []
        for driver_id, distance in results:
            # Get driver details
            driver_location = self.redis.get(f'driver:{driver_id}:location')
            if driver_location:
                location = json.loads(driver_location)
                nearby_drivers.append({
                    'driver_id': driver_id,
                    'distance_km': float(distance),
                    'location': location
                })

        return nearby_drivers

    def update_rider_location(self, rider_id, latitude, longitude):
        """Track rider location during trip"""
        location = {
            'rider_id': rider_id,
            'latitude': latitude,
            'longitude': longitude,
            'timestamp': datetime.utcnow().isoformat()
        }

        self.redis.setex(
            f'rider:{rider_id}:location',
            300,  # 5 min TTL
            json.dumps(location)
        )

        return location

# WebSocket for real-time location streaming
@socketio.on('driver_location_update')
def handle_driver_location(data):
    """
    Receive location update from driver app

    data = {
        'driver_id': 'driver123',
        'latitude': 37.7749,
        'longitude': -122.4194
    }
    """
    location_service = LocationService()
    location = location_service.update_driver_location(
        data['driver_id'],
        data['latitude'],
        data['longitude']
    )

    # Broadcast to riders tracking this driver
    emit('location_update', location, broadcast=True, room=data['driver_id'])

@app.route('/api/drivers/nearby', methods=['POST'])
def get_nearby_drivers():
    """
    POST /api/drivers/nearby
    {
      "latitude": 37.7749,
      "longitude": -122.4194,
      "radius_km": 5
    }
    """
    data = request.json
    location_service = LocationService()

    drivers = location_service.get_nearby_drivers(
        data['latitude'],
        data['longitude'],
        data.get('radius_km', 5)
    )

    return {'drivers': drivers}, 200
```

---

## Driver-Rider Matching

```python
import time
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class Driver:
    id: str
    latitude: float
    longitude: float
    rating: float
    trips_completed: int
    acceptance_rate: float
    is_available: bool

@dataclass
class RideRequest:
    rider_id: str
    pickup_latitude: float
    pickup_longitude: float
    destination_latitude: float
    destination_longitude: float
    ride_type: str  # standard, premium, xl

class MatchingService:
    """
    Match riders with optimal drivers

    Algorithm:
    1. Find nearby available drivers (within 5km)
    2. Calculate score for each driver
    3. Select best driver
    4. Send request to driver
    5. If rejected, try next driver
    """

    def __init__(self):
        self.location_service = LocationService()
        self.eta_service = ETAService()

    def find_match(self, ride_request: RideRequest) -> Optional[Driver]:
        """
        Find best driver for ride request

        Matching criteria:
        - Distance (closer is better)
        - ETA to pickup (faster is better)
        - Driver rating (higher is better)
        - Acceptance rate (higher is better)
        """
        # Get nearby drivers
        nearby_drivers = self.location_service.get_nearby_drivers(
            ride_request.pickup_latitude,
            ride_request.pickup_longitude,
            radius_km=5
        )

        if not nearby_drivers:
            # Expand search radius
            nearby_drivers = self.location_service.get_nearby_drivers(
                ride_request.pickup_latitude,
                ride_request.pickup_longitude,
                radius_km=10
            )

        if not nearby_drivers:
            return None  # No drivers available

        # Filter by availability and ride type
        available_drivers = self._filter_drivers(nearby_drivers, ride_request)

        if not available_drivers:
            return None

        # Score and rank drivers
        scored_drivers = []
        for driver_data in available_drivers:
            driver = self._get_driver_details(driver_data['driver_id'])

            if not driver or not driver.is_available:
                continue

            # Calculate ETA to pickup
            eta_minutes = self.eta_service.calculate_eta(
                driver.latitude,
                driver.longitude,
                ride_request.pickup_latitude,
                ride_request.pickup_longitude
            )

            # Calculate matching score
            score = self._calculate_driver_score(
                driver,
                driver_data['distance_km'],
                eta_minutes
            )

            scored_drivers.append((driver, score, eta_minutes))

        if not scored_drivers:
            return None

        # Sort by score (descending)
        scored_drivers.sort(key=lambda x: x[1], reverse=True)

        # Try drivers in order until one accepts
        for driver, score, eta in scored_drivers[:5]:  # Try top 5
            accepted = self._send_ride_request(driver, ride_request, eta)
            if accepted:
                return driver

        return None

    def _calculate_driver_score(self, driver: Driver, distance_km: float, eta_minutes: float) -> float:
        """
        Calculate driver matching score (0-100)

        Factors:
        - Distance (40%): Closer is better
        - ETA (30%): Faster pickup is better
        - Rating (20%): Higher rating is better
        - Acceptance rate (10%): Higher acceptance is better
        """
        # Distance score (normalize: 0km=100, 5km=0)
        distance_score = max(0, 100 - (distance_km / 5 * 100))

        # ETA score (normalize: 0min=100, 30min=0)
        eta_score = max(0, 100 - (eta_minutes / 30 * 100))

        # Rating score (normalize: 5 stars=100, 0 stars=0)
        rating_score = (driver.rating / 5) * 100

        # Acceptance rate score
        acceptance_score = driver.acceptance_rate * 100

        # Weighted average
        total_score = (
            distance_score * 0.4 +
            eta_score * 0.3 +
            rating_score * 0.2 +
            acceptance_score * 0.1
        )

        return total_score

    def _send_ride_request(self, driver: Driver, ride_request: RideRequest, eta: float) -> bool:
        """
        Send ride request to driver

        Driver has 15 seconds to accept
        """
        request_data = {
            'ride_request_id': str(uuid.uuid4()),
            'rider_id': ride_request.rider_id,
            'pickup_location': {
                'latitude': ride_request.pickup_latitude,
                'longitude': ride_request.pickup_longitude
            },
            'destination': {
                'latitude': ride_request.destination_latitude,
                'longitude': ride_request.destination_longitude
            },
            'eta_to_pickup': eta,
            'estimated_fare': self._calculate_fare(ride_request)
        }

        # Send via WebSocket
        socketio.emit(
            'ride_request',
            request_data,
            room=driver.id
        )

        # Wait for response (with timeout)
        response = self._wait_for_driver_response(
            request_data['ride_request_id'],
            timeout=15
        )

        return response == 'accepted'

    def _filter_drivers(self, nearby_drivers: List, ride_request: RideRequest) -> List:
        """Filter drivers by ride type and availability"""
        filtered = []

        for driver_data in nearby_drivers:
            driver = self._get_driver_details(driver_data['driver_id'])

            # Check if driver supports ride type
            if ride_request.ride_type == 'xl' and not driver.supports_xl:
                continue

            if driver.is_available:
                filtered.append(driver_data)

        return filtered
```

---

## Geospatial Indexing

### QuadTree for Spatial Partitioning

```python
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class Point:
    x: float  # longitude
    y: float  # latitude
    data: any  # driver_id, etc.

class QuadTree:
    """
    QuadTree for efficient spatial indexing

    Divides map into hierarchical quadrants
    Faster than scanning all drivers for nearby search
    """

    def __init__(self, boundary, capacity=4):
        self.boundary = boundary  # (min_x, min_y, max_x, max_y)
        self.capacity = capacity
        self.points = []
        self.divided = False
        self.northwest = None
        self.northeast = None
        self.southwest = None
        self.southeast = None

    def insert(self, point: Point) -> bool:
        """Insert point into quadtree"""
        if not self._contains(point):
            return False

        if len(self.points) < self.capacity:
            self.points.append(point)
            return True

        if not self.divided:
            self._subdivide()

        # Try inserting into children
        if self.northwest.insert(point):
            return True
        if self.northeast.insert(point):
            return True
        if self.southwest.insert(point):
            return True
        if self.southeast.insert(point):
            return True

        return False

    def query_range(self, range_boundary) -> List[Point]:
        """Find all points within range"""
        found = []

        if not self._intersects(range_boundary):
            return found

        for point in self.points:
            if self._point_in_range(point, range_boundary):
                found.append(point)

        if self.divided:
            found.extend(self.northwest.query_range(range_boundary))
            found.extend(self.northeast.query_range(range_boundary))
            found.extend(self.southwest.query_range(range_boundary))
            found.extend(self.southeast.query_range(range_boundary))

        return found

    def _subdivide(self):
        """Split into 4 quadrants"""
        min_x, min_y, max_x, max_y = self.boundary
        mid_x = (min_x + max_x) / 2
        mid_y = (min_y + max_y) / 2

        self.northwest = QuadTree((min_x, mid_y, mid_x, max_y), self.capacity)
        self.northeast = QuadTree((mid_x, mid_y, max_x, max_y), self.capacity)
        self.southwest = QuadTree((min_x, min_y, mid_x, mid_y), self.capacity)
        self.southeast = QuadTree((mid_x, min_y, max_x, mid_y), self.capacity)

        self.divided = True

    def _contains(self, point: Point) -> bool:
        """Check if point is within boundary"""
        min_x, min_y, max_x, max_y = self.boundary
        return (min_x <= point.x < max_x and min_y <= point.y < max_y)

    def _intersects(self, range_boundary) -> bool:
        """Check if range intersects with this boundary"""
        min_x, min_y, max_x, max_y = self.boundary
        r_min_x, r_min_y, r_max_x, r_max_y = range_boundary

        return not (r_max_x < min_x or r_min_x > max_x or
                   r_max_y < min_y or r_min_y > max_y)

    def _point_in_range(self, point: Point, range_boundary) -> bool:
        """Check if point is within range"""
        r_min_x, r_min_y, r_max_x, r_max_y = range_boundary
        return (r_min_x <= point.x < r_max_x and r_min_y <= point.y < r_max_y)

# Usage for driver location indexing
class DriverLocationIndex:
    def __init__(self):
        # San Francisco bay area boundaries
        self.quadtree = QuadTree(
            boundary=(-122.5, 37.2, -122.0, 37.8),
            capacity=10
        )

    def add_driver(self, driver_id, latitude, longitude):
        """Add driver to spatial index"""
        point = Point(x=longitude, y=latitude, data=driver_id)
        self.quadtree.insert(point)

    def find_nearby_drivers(self, latitude, longitude, radius_km=5):
        """Find drivers within radius"""
        # Convert radius to lat/lon degrees (approximate)
        radius_deg = radius_km / 111  # 1 degree ≈ 111 km

        range_boundary = (
            longitude - radius_deg,
            latitude - radius_deg,
            longitude + radius_deg,
            latitude + radius_deg
        )

        points = self.quadtree.query_range(range_boundary)
        return [p.data for p in points]
```

### Geohash Alternative

```python
import geohash2

class GeohashIndex:
    """
    Use geohash for location indexing

    Geohash encodes lat/lon into string
    Nearby locations have common prefixes
    """

    def __init__(self):
        self.redis = redis.Redis()

    def add_driver(self, driver_id, latitude, longitude):
        """Add driver with geohash"""
        # Encode location to geohash (precision 6 ≈ ±0.61 km)
        ghash = geohash2.encode(latitude, longitude, precision=6)

        # Store in Redis set for this geohash cell
        self.redis.sadd(f'geo:{ghash}', driver_id)

        # Also store driver's exact geohash
        self.redis.set(f'driver:{driver_id}:geohash', ghash)

    def find_nearby_drivers(self, latitude, longitude, radius_km=5):
        """Find drivers in nearby geohash cells"""
        center_ghash = geohash2.encode(latitude, longitude, precision=6)

        # Get neighboring cells
        neighbors = geohash2.neighbors(center_ghash)
        neighbors.append(center_ghash)

        # Collect drivers from all cells
        drivers = set()
        for ghash in neighbors:
            cell_drivers = self.redis.smembers(f'geo:{ghash}')
            drivers.update(cell_drivers)

        return list(drivers)
```

---

## Route Optimization

```python
import requests

class ETAService:
    """
    Calculate ETA and optimal routes
    """

    def __init__(self):
        self.google_maps_api_key = "YOUR_API_KEY"

    def calculate_eta(self, from_lat, from_lon, to_lat, to_lon):
        """
        Calculate ETA using Google Maps Directions API

        Returns ETA in minutes
        """
        url = "https://maps.googleapis.com/maps/api/directions/json"

        params = {
            'origin': f'{from_lat},{from_lon}',
            'destination': f'{to_lat},{to_lon}',
            'mode': 'driving',
            'departure_time': 'now',
            'traffic_model': 'best_guess',
            'key': self.google_maps_api_key
        }

        response = requests.get(url, params=params)
        data = response.json()

        if data['status'] == 'OK':
            route = data['routes'][0]
            leg = route['legs'][0]

            # Duration considering traffic
            duration_seconds = leg['duration_in_traffic']['value']
            eta_minutes = duration_seconds / 60

            return eta_minutes
        else:
            # Fallback: Haversine distance / average speed
            distance_km = self._haversine_distance(from_lat, from_lon, to_lat, to_lon)
            avg_speed_kmh = 40  # Assume 40 km/h average
            eta_minutes = (distance_km / avg_speed_kmh) * 60
            return eta_minutes

    def get_route(self, from_lat, from_lon, to_lat, to_lon):
        """
        Get detailed route with turn-by-turn directions

        Returns polyline and waypoints
        """
        url = "https://maps.googleapis.com/maps/api/directions/json"

        params = {
            'origin': f'{from_lat},{from_lon}',
            'destination': f'{to_lat},{to_lon}',
            'mode': 'driving',
            'key': self.google_maps_api_key
        }

        response = requests.get(url, params=params)
        data = response.json()

        if data['status'] == 'OK':
            route = data['routes'][0]

            return {
                'polyline': route['overview_polyline']['points'],
                'distance_meters': route['legs'][0]['distance']['value'],
                'duration_seconds': route['legs'][0]['duration']['value'],
                'steps': route['legs'][0]['steps']
            }

        return None

    def _haversine_distance(self, lat1, lon1, lat2, lon2):
        """Calculate distance between two points (km)"""
        from math import radians, sin, cos, sqrt, atan2

        R = 6371  # Earth radius in km

        lat1, lon1, lat2, lon2 = map(radians, [lat1, lon1, lat2, lon2])

        dlat = lat2 - lat1
        dlon = lon2 - lon1

        a = sin(dlat/2)**2 + cos(lat1) * cos(lat2) * sin(dlon/2)**2
        c = 2 * atan2(sqrt(a), sqrt(1-a))

        return R * c
```

---

## Surge Pricing

```python
class SurgePricingService:
    """
    Dynamic pricing based on supply and demand

    Surge multiplier: 1.0x (normal) to 5.0x (very high demand)
    """

    def __init__(self):
        self.redis = redis.Redis()

    def calculate_surge_multiplier(self, latitude, longitude):
        """
        Calculate surge pricing for location

        Algorithm:
        1. Count active riders in area
        2. Count available drivers in area
        3. Calculate demand/supply ratio
        4. Apply surge multiplier
        """
        # Define area (1 km radius)
        area_geohash = geohash2.encode(latitude, longitude, precision=5)

        # Get demand (active ride requests)
        active_requests = self._count_active_requests(area_geohash)

        # Get supply (available drivers)
        available_drivers = self._count_available_drivers(area_geohash)

        if available_drivers == 0:
            return 3.0  # High surge if no drivers

        # Demand/supply ratio
        ratio = active_requests / available_drivers

        # Calculate surge multiplier
        if ratio < 0.5:
            surge = 1.0  # Normal pricing
        elif ratio < 1.0:
            surge = 1.2
        elif ratio < 1.5:
            surge = 1.5
        elif ratio < 2.0:
            surge = 2.0
        elif ratio < 3.0:
            surge = 3.0
        else:
            surge = 5.0  # Maximum surge

        # Store surge for this area (cache for 2 minutes)
        self.redis.setex(f'surge:{area_geohash}', 120, surge)

        return surge

    def _count_active_requests(self, area_geohash):
        """Count active ride requests in area"""
        count = self.redis.scard(f'requests:{area_geohash}')
        return count

    def _count_available_drivers(self, area_geohash):
        """Count available drivers in area"""
        count = self.redis.scard(f'geo:{area_geohash}')
        return count

    def notify_surge_area(self, latitude, longitude, surge_multiplier):
        """
        Notify nearby drivers of surge area

        Incentivize drivers to move to high-demand areas
        """
        nearby_drivers = location_service.get_nearby_drivers(
            latitude, longitude, radius_km=10
        )

        for driver in nearby_drivers:
            socketio.emit('surge_alert', {
                'location': {'latitude': latitude, 'longitude': longitude},
                'surge_multiplier': surge_multiplier,
                'message': f'{surge_multiplier}x surge pricing in effect!'
            }, room=driver['driver_id'])
```

---

## Real-Time Updates

```python
from flask_socketio import join_room, leave_room

class TripUpdatesService:
    """
    Real-time trip status updates via WebSocket
    """

    @staticmethod
    @socketio.on('join_trip')
    def handle_join_trip(data):
        """
        User joins trip room to receive updates

        data = {'trip_id': 'trip123', 'user_id': 'user456'}
        """
        trip_id = data['trip_id']
        join_room(trip_id)

        # Send current trip status
        trip = db.get_trip(trip_id)
        emit('trip_status', {
            'status': trip['status'],
            'driver_location': trip['driver_location'],
            'eta': trip['eta']
        })

    @staticmethod
    def broadcast_trip_update(trip_id, update_type, data):
        """
        Broadcast update to all users in trip room

        update_type: 'driver_location', 'status_change', 'eta_update'
        """
        socketio.emit(update_type, data, room=trip_id)

# Example: Driver location updates during trip
@socketio.on('driver_location_update')
def handle_driver_location_update(data):
    """
    Driver sends location update during trip
    """
    driver_id = data['driver_id']
    trip = db.get_active_trip_for_driver(driver_id)

    if trip:
        # Update trip location
        db.update_trip_driver_location(
            trip['id'],
            data['latitude'],
            data['longitude']
        )

        # Recalculate ETA
        eta = eta_service.calculate_eta(
            data['latitude'],
            data['longitude'],
            trip['destination_lat'],
            trip['destination_lon']
        )

        # Broadcast to rider
        TripUpdatesService.broadcast_trip_update(
            trip['id'],
            'driver_location',
            {
                'latitude': data['latitude'],
                'longitude': data['longitude'],
                'eta_minutes': eta
            }
        )
```

---

## Trip Management

```python
from enum import Enum

class TripStatus(Enum):
    REQUESTED = 'requested'
    DRIVER_ASSIGNED = 'driver_assigned'
    DRIVER_ARRIVING = 'driver_arriving'
    IN_PROGRESS = 'in_progress'
    COMPLETED = 'completed'
    CANCELLED = 'cancelled'

class TripService:
    """
    Manage trip lifecycle
    """

    def create_trip(self, rider_id, pickup, destination):
        """
        Create new trip request

        Flow:
        1. Create trip record
        2. Find matching driver
        3. Send request to driver
        4. Update trip status
        """
        trip_id = str(uuid.uuid4())

        trip = {
            'id': trip_id,
            'rider_id': rider_id,
            'pickup_lat': pickup['latitude'],
            'pickup_lon': pickup['longitude'],
            'dest_lat': destination['latitude'],
            'dest_lon': destination['longitude'],
            'status': TripStatus.REQUESTED.value,
            'created_at': datetime.utcnow().isoformat()
        }

        db.insert('trips', trip)

        # Find matching driver
        ride_request = RideRequest(
            rider_id=rider_id,
            pickup_latitude=pickup['latitude'],
            pickup_longitude=pickup['longitude'],
            destination_latitude=destination['latitude'],
            destination_longitude=destination['longitude'],
            ride_type='standard'
        )

        driver = matching_service.find_match(ride_request)

        if driver:
            self.assign_driver(trip_id, driver.id)
        else:
            # No drivers available
            db.update('trips', trip_id, {'status': TripStatus.CANCELLED.value})
            return None

        return trip_id

    def assign_driver(self, trip_id, driver_id):
        """Assign driver to trip"""
        db.update('trips', trip_id, {
            'driver_id': driver_id,
            'status': TripStatus.DRIVER_ASSIGNED.value,
            'assigned_at': datetime.utcnow().isoformat()
        })

        # Notify rider
        trip = db.get_trip(trip_id)
        TripUpdatesService.broadcast_trip_update(
            trip_id,
            'driver_assigned',
            {
                'driver_id': driver_id,
                'driver_name': trip['driver']['name'],
                'driver_photo': trip['driver']['photo_url'],
                'driver_rating': trip['driver']['rating'],
                'vehicle': trip['driver']['vehicle']
            }
        )

    def start_trip(self, trip_id):
        """Driver picks up rider and starts trip"""
        db.update('trips', trip_id, {
            'status': TripStatus.IN_PROGRESS.value,
            'started_at': datetime.utcnow().isoformat()
        })

        TripUpdatesService.broadcast_trip_update(
            trip_id,
            'trip_started',
            {'message': 'Trip has started'}
        )

    def complete_trip(self, trip_id):
        """Complete trip at destination"""
        trip = db.get_trip(trip_id)

        # Calculate final fare
        distance_km = self._calculate_trip_distance(trip)
        duration_min = self._calculate_trip_duration(trip)
        surge_multiplier = trip.get('surge_multiplier', 1.0)

        fare = self._calculate_fare(distance_km, duration_min, surge_multiplier)

        db.update('trips', trip_id, {
            'status': TripStatus.COMPLETED.value,
            'completed_at': datetime.utcnow().isoformat(),
            'distance_km': distance_km,
            'duration_min': duration_min,
            'fare': fare
        })

        # Process payment
        payment_service.charge_rider(trip['rider_id'], fare)
        payment_service.pay_driver(trip['driver_id'], fare * 0.8)  # 80% to driver

        # Notify completion
        TripUpdatesService.broadcast_trip_update(
            trip_id,
            'trip_completed',
            {
                'fare': fare,
                'distance_km': distance_km,
                'duration_min': duration_min
            }
        )

        return fare

    def _calculate_fare(self, distance_km, duration_min, surge_multiplier):
        """
        Calculate trip fare

        Formula: Base + (Distance × Rate) + (Time × Rate) × Surge
        """
        BASE_FARE = 2.50
        PER_KM = 1.75
        PER_MIN = 0.35

        base_fare = BASE_FARE + (distance_km * PER_KM) + (duration_min * PER_MIN)
        final_fare = base_fare * surge_multiplier

        return round(final_fare, 2)
```

---

## Database Design

```sql
-- Users
CREATE TABLE users (
    id VARCHAR(255) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20) UNIQUE NOT NULL,
    photo_url VARCHAR(512),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Drivers
CREATE TABLE drivers (
    id VARCHAR(255) PRIMARY KEY,
    user_id VARCHAR(255) REFERENCES users(id),
    license_number VARCHAR(50) NOT NULL,
    vehicle_make VARCHAR(100),
    vehicle_model VARCHAR(100),
    vehicle_year INT,
    vehicle_plate VARCHAR(20),
    rating DECIMAL(3,2) DEFAULT 5.00,
    trips_completed INT DEFAULT 0,
    acceptance_rate DECIMAL(3,2) DEFAULT 1.00,
    is_online BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_online (is_online)
);

-- Trips
CREATE TABLE trips (
    id VARCHAR(255) PRIMARY KEY,
    rider_id VARCHAR(255) REFERENCES users(id),
    driver_id VARCHAR(255) REFERENCES drivers(id),
    pickup_lat DECIMAL(10,8) NOT NULL,
    pickup_lon DECIMAL(11,8) NOT NULL,
    dest_lat DECIMAL(10,8) NOT NULL,
    dest_lon DECIMAL(11,8) NOT NULL,
    status VARCHAR(50) NOT NULL,
    distance_km DECIMAL(10,2),
    duration_min INT,
    fare DECIMAL(10,2),
    surge_multiplier DECIMAL(3,2) DEFAULT 1.00,
    created_at TIMESTAMP DEFAULT NOW(),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    INDEX idx_rider (rider_id),
    INDEX idx_driver (driver_id),
    INDEX idx_status (status),
    INDEX idx_created (created_at)
);

-- Ratings
CREATE TABLE ratings (
    id SERIAL PRIMARY KEY,
    trip_id VARCHAR(255) REFERENCES trips(id),
    rater_id VARCHAR(255) REFERENCES users(id),
    ratee_id VARCHAR(255) REFERENCES users(id),
    rating INT CHECK (rating BETWEEN 1 AND 5),
    comment TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_ratee (ratee_id)
);

-- Payments
CREATE TABLE payments (
    id VARCHAR(255) PRIMARY KEY,
    trip_id VARCHAR(255) REFERENCES trips(id),
    rider_id VARCHAR(255) REFERENCES users(id),
    driver_id VARCHAR(255) REFERENCES drivers(id),
    amount DECIMAL(10,2) NOT NULL,
    payment_method VARCHAR(50),
    status VARCHAR(50) NOT NULL,
    processed_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_trip (trip_id),
    INDEX idx_rider (rider_id)
);
```

---

## Real-World Examples

### Uber
- **Scale**: 131M+ users, 6.3B trips/year
- **Architecture**: Microservices (2000+)
- **Geospatial**: Custom geospatial database (Geofence)
- **Matching**: Machine learning-based dispatch
- **Tech**: Go, Java, Python, Kafka, Cassandra

### Lyft
- **Matching**: Probability-based matching
- **Features**: Shared rides, scheduled rides
- **Tech**: AWS, Kubernetes, Kafka
- **Innovation**: Multi-modal (bikes, scooters)

### Didi (China)
- **Scale**: 550M+ users (largest globally)
- **AI**: Deep learning for ETA prediction
- **Features**: Hitch (carpooling), bike sharing

---

## Interview Tips

### Common Questions

**Q: How do you handle millions of location updates per second?**
- WebSocket for real-time streaming
- Kafka for event processing
- Redis for hot data (current locations)
- Database for persistent storage

**Q: How do you match drivers efficiently?**
- Geospatial indexing (QuadTree, Geohash, Redis Geo)
- Pre-filter by distance (<5km)
- Score drivers (distance, ETA, rating)
- Try top N drivers until acceptance

**Q: How does surge pricing work?**
- Calculate demand/supply ratio per area
- Apply multiplier (1x to 5x)
- Update every 2-5 minutes
- Notify drivers of surge areas

**Q: How do you calculate ETA?**
- Google Maps API for real-time traffic
- Fallback: Haversine distance + average speed
- Update ETA during trip based on current location

**Q: How do you ensure driver safety?**
- Background checks
- Real-time location tracking
- Emergency button (SOS)
- Trip sharing (share trip with contacts)
- Driver rating system

### Key Takeaways

1. **Geospatial indexing**: QuadTree, Geohash, Redis Geo
2. **Real-time communication**: WebSocket for bidirectional updates
3. **Event streaming**: Kafka for location updates, trip events
4. **Matching algorithm**: Score-based with multiple factors
5. **Surge pricing**: Dynamic based on supply/demand
6. **Scalability**: Microservices, horizontal scaling
7. **Fault tolerance**: Graceful degradation, retries

This ride-sharing design demonstrates real-time systems, geospatial algorithms, and high-scale distributed architecture - critical for location-based service interviews!
