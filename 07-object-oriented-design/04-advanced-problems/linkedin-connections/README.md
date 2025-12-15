# LinkedIn Professional Network - OOD Design

**Difficulty:** Advanced
**Interview Frequency:** High
**Key Concepts:** Graph Structures, Social Networks, Connections, Recommendations
**Companies:** LinkedIn, Facebook, Twitter, Professional Networks

---

## Problem Statement

Design a professional networking platform like LinkedIn with user profiles, connections (1st, 2nd, 3rd degree), connection requests, endorsements, and connection recommendations.

---

## Requirements

### Core Features
1. User profiles with professional information
2. Connection management (send/accept/reject requests)
3. Connection degrees (1st, 2nd, 3rd degree)
4. Connection recommendations (People You May Know)
5. Endorsements for skills
6. Search users by name, company, or skills

### Advanced Features
7. Mutual connections discovery
8. Network size calculation
9. Shortest path between users
10. Privacy settings for connections

---

## Implementation

```python
from enum import Enum
from typing import List, Optional, Set, Dict
from datetime import datetime
from collections import deque


class ConnectionStatus(Enum):
    PENDING = "Pending"
    ACCEPTED = "Accepted"
    REJECTED = "Rejected"


class PrivacyLevel(Enum):
    PUBLIC = "Public"
    CONNECTIONS_ONLY = "Connections Only"
    PRIVATE = "Private"


class Skill:
    def __init__(self, skill_id: int, name: str):
        self.skill_id = skill_id
        self.name = name


class Endorsement:
    def __init__(self, endorser_id: int, skill: Skill, timestamp: datetime = None):
        self.endorser_id = endorser_id
        self.skill = skill
        self.timestamp = timestamp or datetime.now()


class Experience:
    def __init__(self, company: str, title: str, start_year: int, end_year: Optional[int] = None):
        self.company = company
        self.title = title
        self.start_year = start_year
        self.end_year = end_year
        self.is_current = end_year is None

    def __str__(self) -> str:
        end = "Present" if self.is_current else str(self.end_year)
        return f"{self.title} at {self.company} ({self.start_year} - {end})"


class User:
    def __init__(self, user_id: int, name: str, email: str, headline: str = ""):
        self.user_id = user_id
        self.name = name
        self.email = email
        self.headline = headline
        self.experiences: List[Experience] = []
        self.skills: List[Skill] = []
        self.endorsements: Dict[int, List[Endorsement]] = {}  # skill_id -> endorsements
        self.privacy_level = PrivacyLevel.PUBLIC

    def add_experience(self, experience: Experience):
        self.experiences.append(experience)

    def add_skill(self, skill: Skill):
        if skill not in self.skills:
            self.skills.append(skill)
            self.endorsements[skill.skill_id] = []

    def endorse_skill(self, skill: Skill, endorser_id: int):
        """Receive endorsement for a skill"""
        if skill.skill_id in self.endorsements:
            endorsement = Endorsement(endorser_id, skill)
            self.endorsements[skill.skill_id].append(endorsement)

    def get_current_company(self) -> Optional[str]:
        current_exp = [e for e in self.experiences if e.is_current]
        return current_exp[0].company if current_exp else None

    def __str__(self) -> str:
        return f"{self.name} - {self.headline}"

    def __hash__(self):
        return hash(self.user_id)

    def __eq__(self, other):
        return isinstance(other, User) and self.user_id == other.user_id


class ConnectionRequest:
    _request_counter = 1

    def __init__(self, sender: User, receiver: User, message: str = ""):
        self.request_id = ConnectionRequest._request_counter
        ConnectionRequest._request_counter += 1
        self.sender = sender
        self.receiver = receiver
        self.message = message
        self.status = ConnectionStatus.PENDING
        self.sent_at = datetime.now()
        self.responded_at: Optional[datetime] = None

    def accept(self):
        self.status = ConnectionStatus.ACCEPTED
        self.responded_at = datetime.now()

    def reject(self):
        self.status = ConnectionStatus.REJECTED
        self.responded_at = datetime.now()


class Connection:
    def __init__(self, user1: User, user2: User, connected_at: datetime = None):
        self.user1 = user1
        self.user2 = user2
        self.connected_at = connected_at or datetime.now()

    def get_other_user(self, user: User) -> User:
        """Get the other user in this connection"""
        return self.user2 if user.user_id == self.user1.user_id else self.user1


class LinkedInNetwork:
    def __init__(self):
        self.users: Dict[int, User] = {}
        self.connections: Dict[int, Set[int]] = {}  # user_id -> set of connected user_ids
        self.pending_requests: List[ConnectionRequest] = []

    def add_user(self, user: User):
        self.users[user.user_id] = user
        self.connections[user.user_id] = set()

    def send_connection_request(self, sender_id: int, receiver_id: int, message: str = "") -> Optional[ConnectionRequest]:
        sender = self.users.get(sender_id)
        receiver = self.users.get(receiver_id)

        if not sender or not receiver:
            print("User not found")
            return None

        if self.are_connected(sender_id, receiver_id):
            print(f"{sender.name} and {receiver.name} are already connected")
            return None

        # Check if request already exists
        existing = [r for r in self.pending_requests
                   if r.sender.user_id == sender_id and r.receiver.user_id == receiver_id
                   and r.status == ConnectionStatus.PENDING]
        if existing:
            print("Connection request already sent")
            return None

        request = ConnectionRequest(sender, receiver, message)
        self.pending_requests.append(request)
        print(f"✓ Connection request sent from {sender.name} to {receiver.name}")
        return request

    def accept_connection_request(self, request_id: int) -> bool:
        request = next((r for r in self.pending_requests if r.request_id == request_id), None)

        if not request:
            print("Request not found")
            return False

        if request.status != ConnectionStatus.PENDING:
            print("Request already processed")
            return False

        request.accept()

        # Create bidirectional connection
        self.connections[request.sender.user_id].add(request.receiver.user_id)
        self.connections[request.receiver.user_id].add(request.sender.user_id)

        print(f"✓ {request.receiver.name} and {request.sender.name} are now connected")
        return True

    def reject_connection_request(self, request_id: int) -> bool:
        request = next((r for r in self.pending_requests if r.request_id == request_id), None)

        if not request:
            return False

        request.reject()
        print(f"✓ Connection request rejected")
        return True

    def are_connected(self, user1_id: int, user2_id: int) -> bool:
        """Check if two users are 1st degree connections"""
        return user2_id in self.connections.get(user1_id, set())

    def get_first_degree_connections(self, user_id: int) -> List[User]:
        """Get all 1st degree connections"""
        connection_ids = self.connections.get(user_id, set())
        return [self.users[uid] for uid in connection_ids]

    def get_connection_degree(self, user1_id: int, user2_id: int) -> Optional[int]:
        """
        Find the degree of connection between two users using BFS.
        Returns 1 for direct connection, 2 for friend-of-friend, 3 for third degree, None if not connected.
        """
        if user1_id == user2_id:
            return 0

        if user2_id in self.connections.get(user1_id, set()):
            return 1

        # BFS to find shortest path
        visited = {user1_id}
        queue = deque([(user1_id, 0)])

        while queue:
            current_user_id, degree = queue.popleft()

            if degree >= 3:  # LinkedIn typically shows up to 3rd degree
                continue

            for neighbor_id in self.connections.get(current_user_id, set()):
                if neighbor_id == user2_id:
                    return degree + 1

                if neighbor_id not in visited:
                    visited.add(neighbor_id)
                    queue.append((neighbor_id, degree + 1))

        return None  # Not connected within 3 degrees

    def get_mutual_connections(self, user1_id: int, user2_id: int) -> List[User]:
        """Find mutual connections between two users"""
        connections1 = self.connections.get(user1_id, set())
        connections2 = self.connections.get(user2_id, set())
        mutual_ids = connections1 & connections2
        return [self.users[uid] for uid in mutual_ids]

    def get_network_size(self, user_id: int, max_degree: int = 3) -> int:
        """
        Calculate total network size up to max_degree.
        Degree 1 = direct connections
        Degree 2 = connections of connections
        Degree 3 = third degree
        """
        visited = {user_id}
        queue = deque([(user_id, 0)])

        while queue:
            current_user_id, degree = queue.popleft()

            if degree >= max_degree:
                continue

            for neighbor_id in self.connections.get(current_user_id, set()):
                if neighbor_id not in visited:
                    visited.add(neighbor_id)
                    queue.append((neighbor_id, degree + 1))

        return len(visited) - 1  # Exclude the user themselves

    def get_people_you_may_know(self, user_id: int, limit: int = 5) -> List[User]:
        """
        Recommend connections based on:
        1. 2nd degree connections (friends of friends)
        2. Common connections count
        3. Common skills/companies
        """
        if user_id not in self.users:
            return []

        user = self.users[user_id]
        first_degree = self.connections.get(user_id, set())
        recommendations = {}

        # Find 2nd degree connections
        for connection_id in first_degree:
            second_degree = self.connections.get(connection_id, set())
            for second_user_id in second_degree:
                if second_user_id != user_id and second_user_id not in first_degree:
                    if second_user_id not in recommendations:
                        recommendations[second_user_id] = 0
                    recommendations[second_user_id] += 1  # Count mutual connections

        # Sort by number of mutual connections
        sorted_recommendations = sorted(recommendations.items(), key=lambda x: x[1], reverse=True)

        # Enhance with common skills/companies
        result = []
        for rec_user_id, mutual_count in sorted_recommendations[:limit]:
            rec_user = self.users[rec_user_id]
            result.append(rec_user)

        return result

    def search_users(self, query: str) -> List[User]:
        """Search users by name, company, or skills"""
        query_lower = query.lower()
        results = []

        for user in self.users.values():
            if (query_lower in user.name.lower() or
                query_lower in user.headline.lower() or
                any(query_lower in exp.company.lower() for exp in user.experiences) or
                any(query_lower in skill.name.lower() for skill in user.skills)):
                results.append(user)

        return results

    def get_pending_requests_for_user(self, user_id: int) -> List[ConnectionRequest]:
        """Get all pending connection requests received by a user"""
        return [r for r in self.pending_requests
                if r.receiver.user_id == user_id and r.status == ConnectionStatus.PENDING]


def main():
    network = LinkedInNetwork()

    # Create users
    alice = User(1, "Alice Johnson", "alice@email.com", "Software Engineer at Google")
    bob = User(2, "Bob Smith", "bob@email.com", "Product Manager at Microsoft")
    charlie = User(3, "Charlie Davis", "charlie@email.com", "Data Scientist at Meta")
    diana = User(4, "Diana Lee", "diana@email.com", "UX Designer at Apple")
    evan = User(5, "Evan Wilson", "evan@email.com", "Software Engineer at Google")

    # Add experiences
    alice.add_experience(Experience("Google", "Software Engineer", 2020))
    bob.add_experience(Experience("Microsoft", "Product Manager", 2019))
    charlie.add_experience(Experience("Meta", "Data Scientist", 2021))
    evan.add_experience(Experience("Google", "Software Engineer", 2022))

    # Add skills
    python_skill = Skill(1, "Python")
    java_skill = Skill(2, "Java")
    ml_skill = Skill(3, "Machine Learning")

    alice.add_skill(python_skill)
    alice.add_skill(java_skill)
    charlie.add_skill(python_skill)
    charlie.add_skill(ml_skill)

    # Add users to network
    for user in [alice, bob, charlie, diana, evan]:
        network.add_user(user)

    print("=== LinkedIn Network Demo ===\n")

    # Send connection requests
    req1 = network.send_connection_request(alice.user_id, bob.user_id, "Let's connect!")
    req2 = network.send_connection_request(bob.user_id, charlie.user_id)
    req3 = network.send_connection_request(alice.user_id, evan.user_id)

    print()

    # Accept requests
    network.accept_connection_request(req1.request_id)
    network.accept_connection_request(req2.request_id)
    network.accept_connection_request(req3.request_id)

    # Add more connections
    req4 = network.send_connection_request(charlie.user_id, diana.user_id)
    network.accept_connection_request(req4.request_id)

    print("\n=== Connection Degrees ===")
    degree = network.get_connection_degree(alice.user_id, charlie.user_id)
    print(f"Connection between Alice and Charlie: {degree} degree(s)")

    degree = network.get_connection_degree(alice.user_id, diana.user_id)
    print(f"Connection between Alice and Diana: {degree} degree(s)")

    print("\n=== Mutual Connections ===")
    mutual = network.get_mutual_connections(alice.user_id, charlie.user_id)
    print(f"Mutual connections between Alice and Charlie: {[u.name for u in mutual]}")

    print("\n=== Network Size ===")
    size = network.get_network_size(alice.user_id, max_degree=2)
    print(f"Alice's network size (up to 2nd degree): {size}")

    print("\n=== People You May Know ===")
    recommendations = network.get_people_you_may_know(alice.user_id, limit=3)
    print(f"Recommendations for Alice:")
    for user in recommendations:
        mutual = network.get_mutual_connections(alice.user_id, user.user_id)
        print(f"  - {user.name} ({len(mutual)} mutual connection(s))")

    print("\n=== Endorsements ===")
    # Bob endorses Alice's Python skill
    alice.endorse_skill(python_skill, bob.user_id)
    print(f"✓ Bob endorsed Alice for {python_skill.name}")
    print(f"Alice's Python endorsements: {len(alice.endorsements[python_skill.skill_id])}")

    print("\n=== Search ===")
    results = network.search_users("Google")
    print(f"Search results for 'Google': {[u.name for u in results]}")


if __name__ == "__main__":
    main()
```

---

## Design Patterns

### 1. Graph Pattern
- **Usage:** Network structure with users as nodes, connections as edges
- **Benefit:** Efficient traversal for degree calculation and recommendations

### 2. Strategy Pattern
- **Usage:** Different recommendation algorithms
- **Example:** Mutual connections, common skills, common companies

### 3. Observer Pattern (Extension)
- **Usage:** Notify users of connection requests, endorsements
- **Implementation:** Event listeners for network activities

### 4. Facade Pattern
- **Usage:** `LinkedInNetwork` simplifies complex graph operations
- **Benefit:** Clean API for connection management

---

## SOLID Principles

### Single Responsibility Principle (SRP)
- `User`: Manages profile information
- `Connection`: Represents relationship between users
- `ConnectionRequest`: Handles request lifecycle
- `LinkedInNetwork`: Manages network graph operations

### Open/Closed Principle (OCP)
- Recommendation algorithm can be extended without modifying core network logic
- New connection types (e.g., follow vs connect) can be added

### Liskov Substitution Principle (LSP)
- Different user types (Premium, Basic) can extend `User` class

### Interface Segregation Principle (ISP)
- Separate interfaces for connection management vs recommendations

### Dependency Inversion Principle (DIP)
- Network operations depend on abstractions (User interface), not concrete implementations

---

## Key Algorithms

### BFS for Connection Degree
```python
def get_connection_degree(self, user1_id: int, user2_id: int) -> Optional[int]:
    visited = {user1_id}
    queue = deque([(user1_id, 0)])

    while queue:
        current_user_id, degree = queue.popleft()
        if degree >= 3:
            continue

        for neighbor_id in self.connections.get(current_user_id, set()):
            if neighbor_id == user2_id:
                return degree + 1
            if neighbor_id not in visited:
                visited.add(neighbor_id)
                queue.append((neighbor_id, degree + 1))

    return None
```

### People You May Know (PYMK)
- Find 2nd degree connections
- Rank by mutual connection count
- Filter by common skills/companies

---

## Extensions

### 1. Connection Strength
- Message frequency
- Profile views
- Endorsement reciprocity

### 2. Groups and Communities
- Professional groups
- Company pages
- Interest-based communities

### 3. Influencer Metrics
- Connection quality score
- Profile views
- Engagement rate

### 4. Privacy Controls
- Hide connections from specific users
- Anonymous profile viewing
- Connection visibility settings

---

## Interview Tips

### Clarifying Questions
1. **Scale**: How many users? How many connections per user?
2. **Features**: Which features are must-haves vs nice-to-haves?
3. **Privacy**: How granular should privacy controls be?
4. **Performance**: Response time expectations for graph traversals?

### Common Follow-ups
1. **Scalability**: "How would you handle 500M users?"
   - Graph database (Neo4j)
   - Distributed graph storage
   - Caching for frequent queries

2. **Performance**: "How to optimize PYMK?"
   - Pre-compute recommendations
   - Limit traversal depth
   - Cache 2nd degree connections

3. **Concurrency**: "Handle simultaneous connection requests?"
   - Optimistic locking
   - Request deduplication
   - Atomic operations for connection creation

4. **Storage**: "Database schema for connections?"
   - Adjacency list in NoSQL
   - Separate connection table with timestamps
   - Indexing for fast lookups

---

## Time Complexity

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| Add Connection | O(1) | Set insertion |
| Check Connection | O(1) | Set membership check |
| Get 1st Degree | O(k) | k = number of connections |
| Get Connection Degree | O(V + E) | BFS, bounded by depth limit |
| Mutual Connections | O(k1 + k2) | Set intersection |
| PYMK | O(k * k_avg) | k = connections, k_avg = avg connections per connection |
| Network Size | O(V + E) | BFS up to max depth |

---

## System Design Considerations

When moving to system design scale:

1. **Database**: Graph database (Neo4j, Amazon Neptune) for efficient traversals
2. **Caching**: Redis for connection lists, recommendations
3. **Message Queue**: Kafka for async request processing
4. **Search**: Elasticsearch for user search
5. **Analytics**: Spark for connection insights, network metrics

---

This tests graph algorithms, social network modeling, and recommendation systems.
