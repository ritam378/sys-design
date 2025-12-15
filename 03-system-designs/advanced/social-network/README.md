# Social Network System Design

## Overview

A **social network** is a platform that enables users to connect, share content, and interact with each other at massive scale.

Examples: Facebook, Twitter/X, Instagram, LinkedIn

## Requirements

### Functional Requirements
1. **User profiles**: Create, update, view profiles
2. **Connections**: Follow/friend users
3. **Posts**: Create, view, like, comment, share
4. **News Feed**: Personalized feed of posts from connections
5. **Notifications**: Real-time updates on interactions
6. **Search**: Find users, posts, hashtags
7. **Messaging**: Direct messages between users
8. **Media**: Upload and share photos/videos

### Non-Functional Requirements
1. **Scale**: 1 billion users, 100 million DAU
2. **Availability**: 99.99% uptime
3. **Latency**: Feed loads < 500ms
4. **Consistency**: Eventual consistency acceptable for feeds
5. **Real-time**: Notifications within seconds

### Capacity Estimation

**Assumptions:**
- 1 billion total users
- 100 million daily active users (DAU)
- Average user has 500 connections
- 50 posts/day per active user
- Average post size: 1 KB (text + metadata)
- 10% posts have media (average 200 KB)

**Storage:**
- Posts per day: 100M × 50 = 5 billion
- Text storage: 5B × 1 KB = 5 TB/day
- Media storage: 5B × 10% × 200 KB = 100 TB/day
- Total per day: ~105 TB
- Yearly: ~38 PB

**Bandwidth:**
- Read-heavy: 100:1 read-to-write ratio
- Feed requests: 100M users × 20 feeds/day = 2B requests/day
- QPS: 2B / 86400 = ~23,000 requests/sec
- Peak (3x): ~70,000 QPS

**Memory (Cache):**
- 20% of users generate 80% of traffic
- Cache hot feeds: 20M × 100 KB = 2 TB

## High-Level Architecture

```
                    ┌─────────────────────────────────┐
                    │         Load Balancer           │
                    │         (Geographic)            │
                    └────────────┬────────────────────┘
                                 │
                ┌────────────────┼────────────────┐
                │                │                │
                ▼                ▼                ▼
    ┌─────────────────┐  ┌─────────────┐  ┌─────────────┐
    │   API Gateway   │  │ WebSocket   │  │   CDN       │
    │   (REST/GraphQL)│  │  Server     │  │ (Media)     │
    └────────┬────────┘  └──────┬──────┘  └─────────────┘
             │                   │
             ▼                   ▼
    ┌────────────────────────────────────────┐
    │        Application Servers              │
    │  - User Service                         │
    │  - Post Service                         │
    │  - Feed Service                         │
    │  - Notification Service                 │
    │  - Search Service                       │
    └───────┬───────────────────┬────────────┘
            │                   │
            ▼                   ▼
    ┌──────────────┐    ┌──────────────┐
    │   Databases  │    │ Cache Layer  │
    │   (Sharded)  │    │   (Redis)    │
    └──────┬───────┘    └──────────────┘
           │
           ▼
    ┌──────────────────────────────────────┐
    │      Message Queue (Kafka)            │
    │  - Post Created Events                │
    │  - Notification Events                │
    │  - Feed Updates                       │
    └───────┬──────────────────────────────┘
            │
            ▼
    ┌──────────────────────────────────────┐
    │     Background Workers                │
    │  - Feed Generator                     │
    │  - Notification Dispatcher            │
    │  - Media Processor                    │
    └──────────────────────────────────────┘
```

## Core Components

### 1. User Service

**User Profile Management:**

```python
from dataclasses import dataclass
from typing import List, Optional
import hashlib
import time

@dataclass
class User:
    user_id: str
    username: str
    email: str
    full_name: str
    bio: str
    profile_picture_url: str
    created_at: int
    verified: bool = False

    def to_dict(self):
        return {
            'user_id': self.user_id,
            'username': self.username,
            'email': self.email,
            'full_name': self.full_name,
            'bio': self.bio,
            'profile_picture_url': self.profile_picture_url,
            'created_at': self.created_at,
            'verified': self.verified
        }


class UserService:
    """
    Handle user profiles and authentication

    Features:
    - CRUD operations on user profiles
    - Password hashing
    - User search
    """

    def __init__(self, db, cache):
        self.db = db
        self.cache = cache

    def create_user(self, username: str, email: str, password: str,
                   full_name: str) -> User:
        """Create new user account"""
        # Validate uniqueness
        if self.username_exists(username):
            raise ValueError("Username already exists")

        if self.email_exists(email):
            raise ValueError("Email already exists")

        # Hash password
        password_hash = self._hash_password(password)

        # Generate user ID
        user_id = self._generate_user_id()

        # Create user
        user = User(
            user_id=user_id,
            username=username,
            email=email,
            full_name=full_name,
            bio='',
            profile_picture_url='',
            created_at=int(time.time())
        )

        # Store in database
        self.db.execute(
            """
            INSERT INTO users (user_id, username, email, password_hash,
                             full_name, bio, created_at)
            VALUES (%s, %s, %s, %s, %s, %s, %s)
            """,
            (user_id, username, email, password_hash, full_name,
             user.bio, user.created_at)
        )

        return user

    def get_user(self, user_id: str) -> Optional[User]:
        """Get user by ID"""
        # Try cache first
        cache_key = f"user:{user_id}"
        cached = self.cache.get(cache_key)
        if cached:
            return User(**cached)

        # Query database
        result = self.db.query_one(
            "SELECT * FROM users WHERE user_id = %s",
            (user_id,)
        )

        if not result:
            return None

        user = User(
            user_id=result['user_id'],
            username=result['username'],
            email=result['email'],
            full_name=result['full_name'],
            bio=result['bio'],
            profile_picture_url=result['profile_picture_url'],
            created_at=result['created_at'],
            verified=result['verified']
        )

        # Cache for 1 hour
        self.cache.set(cache_key, user.to_dict(), ttl=3600)

        return user

    def update_profile(self, user_id: str, updates: dict) -> bool:
        """Update user profile"""
        allowed_fields = ['full_name', 'bio', 'profile_picture_url']

        # Filter to allowed fields
        filtered_updates = {
            k: v for k, v in updates.items()
            if k in allowed_fields
        }

        if not filtered_updates:
            return False

        # Build update query
        set_clause = ', '.join([f"{k} = %s" for k in filtered_updates.keys()])
        values = list(filtered_updates.values()) + [user_id]

        self.db.execute(
            f"UPDATE users SET {set_clause} WHERE user_id = %s",
            values
        )

        # Invalidate cache
        self.cache.delete(f"user:{user_id}")

        return True

    def search_users(self, query: str, limit: int = 20) -> List[User]:
        """Search users by username or name"""
        results = self.db.query(
            """
            SELECT * FROM users
            WHERE username LIKE %s OR full_name LIKE %s
            LIMIT %s
            """,
            (f"%{query}%", f"%{query}%", limit)
        )

        users = []
        for row in results:
            users.append(User(
                user_id=row['user_id'],
                username=row['username'],
                email=row['email'],
                full_name=row['full_name'],
                bio=row['bio'],
                profile_picture_url=row['profile_picture_url'],
                created_at=row['created_at'],
                verified=row['verified']
            ))

        return users

    def username_exists(self, username: str) -> bool:
        """Check if username exists"""
        result = self.db.query_one(
            "SELECT 1 FROM users WHERE username = %s",
            (username,)
        )
        return result is not None

    def email_exists(self, email: str) -> bool:
        """Check if email exists"""
        result = self.db.query_one(
            "SELECT 1 FROM users WHERE email = %s",
            (email,)
        )
        return result is not None

    def _hash_password(self, password: str) -> str:
        """Hash password with salt"""
        import bcrypt
        return bcrypt.hashpw(password.encode(), bcrypt.gensalt()).decode()

    def _generate_user_id(self) -> str:
        """Generate unique user ID"""
        import uuid
        return str(uuid.uuid4())
```

### 2. Connection Service (Follow/Friend)

**Graph-Based Connections:**

```python
from enum import Enum
from typing import List, Set

class ConnectionType(Enum):
    FOLLOW = "follow"  # One-way (Twitter)
    FRIEND = "friend"  # Two-way (Facebook)


class ConnectionService:
    """
    Manage user connections

    Supports:
    - Follow (one-way)
    - Friend (two-way with approval)
    """

    def __init__(self, db, cache):
        self.db = db
        self.cache = cache

    def follow(self, follower_id: str, followee_id: str) -> bool:
        """
        User follows another user (one-way)

        follower_id -> followee_id
        """
        # Validate users exist
        if follower_id == followee_id:
            raise ValueError("Cannot follow yourself")

        # Check if already following
        if self.is_following(follower_id, followee_id):
            return False

        # Create connection
        self.db.execute(
            """
            INSERT INTO connections (follower_id, followee_id, created_at)
            VALUES (%s, %s, %s)
            """,
            (follower_id, followee_id, int(time.time()))
        )

        # Update counters
        self._increment_counter(follower_id, 'following_count')
        self._increment_counter(followee_id, 'followers_count')

        # Invalidate cache
        self._invalidate_connection_cache(follower_id, followee_id)

        # Trigger notification
        self._notify_new_follower(followee_id, follower_id)

        return True

    def unfollow(self, follower_id: str, followee_id: str) -> bool:
        """Unfollow user"""
        result = self.db.execute(
            """
            DELETE FROM connections
            WHERE follower_id = %s AND followee_id = %s
            """,
            (follower_id, followee_id)
        )

        if result.rowcount > 0:
            # Update counters
            self._decrement_counter(follower_id, 'following_count')
            self._decrement_counter(followee_id, 'followers_count')

            # Invalidate cache
            self._invalidate_connection_cache(follower_id, followee_id)

            return True

        return False

    def is_following(self, follower_id: str, followee_id: str) -> bool:
        """Check if user follows another"""
        # Check cache
        cache_key = f"following:{follower_id}:{followee_id}"
        cached = self.cache.get(cache_key)
        if cached is not None:
            return cached == '1'

        # Query database
        result = self.db.query_one(
            """
            SELECT 1 FROM connections
            WHERE follower_id = %s AND followee_id = %s
            """,
            (follower_id, followee_id)
        )

        is_following = result is not None

        # Cache result
        self.cache.set(cache_key, '1' if is_following else '0', ttl=3600)

        return is_following

    def get_followers(self, user_id: str, limit: int = 100,
                     offset: int = 0) -> List[str]:
        """Get list of followers"""
        # Check cache for small limits
        if offset == 0 and limit <= 100:
            cache_key = f"followers:{user_id}"
            cached = self.cache.get(cache_key)
            if cached:
                return cached[:limit]

        # Query database
        results = self.db.query(
            """
            SELECT follower_id FROM connections
            WHERE followee_id = %s
            ORDER BY created_at DESC
            LIMIT %s OFFSET %s
            """,
            (user_id, limit, offset)
        )

        follower_ids = [row['follower_id'] for row in results]

        # Cache first page
        if offset == 0:
            self.cache.set(f"followers:{user_id}", follower_ids, ttl=300)

        return follower_ids

    def get_following(self, user_id: str, limit: int = 100,
                     offset: int = 0) -> List[str]:
        """Get list of users being followed"""
        # Check cache
        if offset == 0 and limit <= 100:
            cache_key = f"following:{user_id}"
            cached = self.cache.get(cache_key)
            if cached:
                return cached[:limit]

        # Query database
        results = self.db.query(
            """
            SELECT followee_id FROM connections
            WHERE follower_id = %s
            ORDER BY created_at DESC
            LIMIT %s OFFSET %s
            """,
            (user_id, limit, offset)
        )

        following_ids = [row['followee_id'] for row in results]

        # Cache first page
        if offset == 0:
            self.cache.set(f"following:{user_id}", following_ids, ttl=300)

        return following_ids

    def get_mutual_connections(self, user_id1: str, user_id2: str) -> Set[str]:
        """Find mutual followers/following"""
        following1 = set(self.get_following(user_id1, limit=10000))
        following2 = set(self.get_following(user_id2, limit=10000))

        return following1 & following2

    def suggest_connections(self, user_id: str, limit: int = 10) -> List[str]:
        """
        Suggest users to follow

        Algorithm:
        1. Find friends of friends
        2. Rank by mutual connections
        3. Filter already following
        """
        # Get users that your connections follow
        following = self.get_following(user_id, limit=1000)

        # Count second-degree connections
        suggestion_scores = {}

        for followee_id in following:
            second_degree = self.get_following(followee_id, limit=100)

            for suggested_id in second_degree:
                # Skip self and already following
                if suggested_id == user_id or suggested_id in following:
                    continue

                suggestion_scores[suggested_id] = \
                    suggestion_scores.get(suggested_id, 0) + 1

        # Sort by score (number of mutual connections)
        suggestions = sorted(
            suggestion_scores.items(),
            key=lambda x: x[1],
            reverse=True
        )

        return [user_id for user_id, _ in suggestions[:limit]]

    def _increment_counter(self, user_id: str, counter: str):
        """Increment user counter"""
        self.db.execute(
            f"UPDATE users SET {counter} = {counter} + 1 WHERE user_id = %s",
            (user_id,)
        )

    def _decrement_counter(self, user_id: str, counter: str):
        """Decrement user counter"""
        self.db.execute(
            f"UPDATE users SET {counter} = {counter} - 1 WHERE user_id = %s",
            (user_id,)
        )

    def _invalidate_connection_cache(self, follower_id: str, followee_id: str):
        """Invalidate connection-related caches"""
        self.cache.delete(f"following:{follower_id}:{followee_id}")
        self.cache.delete(f"followers:{followee_id}")
        self.cache.delete(f"following:{follower_id}")

    def _notify_new_follower(self, user_id: str, follower_id: str):
        """Send notification for new follower"""
        # Send to notification service via Kafka
        pass
```

### 3. Post Service

**Create and Manage Posts:**

```python
from dataclasses import dataclass
from typing import List, Optional
import time

@dataclass
class Post:
    post_id: str
    user_id: str
    content: str
    media_urls: List[str]
    created_at: int
    like_count: int = 0
    comment_count: int = 0
    share_count: int = 0

    def to_dict(self):
        return {
            'post_id': self.post_id,
            'user_id': self.user_id,
            'content': self.content,
            'media_urls': self.media_urls,
            'created_at': self.created_at,
            'like_count': self.like_count,
            'comment_count': self.comment_count,
            'share_count': self.share_count
        }


class PostService:
    """
    Manage posts (create, like, comment, delete)
    """

    def __init__(self, db, cache, event_bus):
        self.db = db
        self.cache = cache
        self.event_bus = event_bus  # Kafka

    def create_post(self, user_id: str, content: str,
                   media_urls: List[str] = None) -> Post:
        """Create new post"""
        # Validate content
        if not content and not media_urls:
            raise ValueError("Post must have content or media")

        if len(content) > 5000:
            raise ValueError("Content too long (max 5000 chars)")

        # Generate post ID
        post_id = self._generate_post_id()

        post = Post(
            post_id=post_id,
            user_id=user_id,
            content=content,
            media_urls=media_urls or [],
            created_at=int(time.time())
        )

        # Store in database
        self.db.execute(
            """
            INSERT INTO posts (post_id, user_id, content, media_urls, created_at)
            VALUES (%s, %s, %s, %s, %s)
            """,
            (post_id, user_id, content,
             json.dumps(media_urls or []), post.created_at)
        )

        # Publish event for feed generation
        self.event_bus.publish('post.created', {
            'post_id': post_id,
            'user_id': user_id,
            'created_at': post.created_at
        })

        return post

    def get_post(self, post_id: str) -> Optional[Post]:
        """Get post by ID"""
        # Try cache
        cache_key = f"post:{post_id}"
        cached = self.cache.get(cache_key)
        if cached:
            return Post(**cached)

        # Query database
        result = self.db.query_one(
            "SELECT * FROM posts WHERE post_id = %s",
            (post_id,)
        )

        if not result:
            return None

        post = Post(
            post_id=result['post_id'],
            user_id=result['user_id'],
            content=result['content'],
            media_urls=json.loads(result['media_urls']),
            created_at=result['created_at'],
            like_count=result['like_count'],
            comment_count=result['comment_count'],
            share_count=result['share_count']
        )

        # Cache
        self.cache.set(cache_key, post.to_dict(), ttl=3600)

        return post

    def delete_post(self, post_id: str, user_id: str) -> bool:
        """Delete post (only by owner)"""
        result = self.db.execute(
            """
            DELETE FROM posts
            WHERE post_id = %s AND user_id = %s
            """,
            (post_id, user_id)
        )

        if result.rowcount > 0:
            # Invalidate cache
            self.cache.delete(f"post:{post_id}")

            # Publish event
            self.event_bus.publish('post.deleted', {
                'post_id': post_id,
                'user_id': user_id
            })

            return True

        return False

    def like_post(self, post_id: str, user_id: str) -> bool:
        """Like a post"""
        # Check if already liked
        if self.has_liked(post_id, user_id):
            return False

        # Record like
        self.db.execute(
            """
            INSERT INTO post_likes (post_id, user_id, created_at)
            VALUES (%s, %s, %s)
            """,
            (post_id, user_id, int(time.time()))
        )

        # Increment like counter
        self.db.execute(
            "UPDATE posts SET like_count = like_count + 1 WHERE post_id = %s",
            (post_id,)
        )

        # Invalidate cache
        self.cache.delete(f"post:{post_id}")

        # Publish event for notification
        self.event_bus.publish('post.liked', {
            'post_id': post_id,
            'user_id': user_id
        })

        return True

    def unlike_post(self, post_id: str, user_id: str) -> bool:
        """Unlike a post"""
        result = self.db.execute(
            """
            DELETE FROM post_likes
            WHERE post_id = %s AND user_id = %s
            """,
            (post_id, user_id)
        )

        if result.rowcount > 0:
            # Decrement counter
            self.db.execute(
                "UPDATE posts SET like_count = like_count - 1 WHERE post_id = %s",
                (post_id,)
            )

            # Invalidate cache
            self.cache.delete(f"post:{post_id}")

            return True

        return False

    def has_liked(self, post_id: str, user_id: str) -> bool:
        """Check if user has liked post"""
        result = self.db.query_one(
            """
            SELECT 1 FROM post_likes
            WHERE post_id = %s AND user_id = %s
            """,
            (post_id, user_id)
        )

        return result is not None

    def add_comment(self, post_id: str, user_id: str, content: str) -> str:
        """Add comment to post"""
        comment_id = self._generate_comment_id()

        self.db.execute(
            """
            INSERT INTO comments (comment_id, post_id, user_id, content, created_at)
            VALUES (%s, %s, %s, %s, %s)
            """,
            (comment_id, post_id, user_id, content, int(time.time()))
        )

        # Increment comment counter
        self.db.execute(
            "UPDATE posts SET comment_count = comment_count + 1 WHERE post_id = %s",
            (post_id,)
        )

        # Invalidate cache
        self.cache.delete(f"post:{post_id}")

        # Publish event
        self.event_bus.publish('post.commented', {
            'post_id': post_id,
            'user_id': user_id,
            'comment_id': comment_id
        })

        return comment_id

    def get_comments(self, post_id: str, limit: int = 50,
                    offset: int = 0) -> List[dict]:
        """Get comments for post"""
        results = self.db.query(
            """
            SELECT * FROM comments
            WHERE post_id = %s
            ORDER BY created_at DESC
            LIMIT %s OFFSET %s
            """,
            (post_id, limit, offset)
        )

        return [dict(row) for row in results]

    def _generate_post_id(self) -> str:
        """Generate unique post ID using Snowflake-like algorithm"""
        import uuid
        return str(uuid.uuid4())

    def _generate_comment_id(self) -> str:
        """Generate unique comment ID"""
        import uuid
        return str(uuid.uuid4())
```

### 4. News Feed Service

**Personalized Feed Generation:**

```python
from typing import List, Dict
import heapq

class FeedService:
    """
    Generate personalized news feed

    Strategies:
    1. Fan-out on write (pre-compute feeds)
    2. Fan-out on read (compute on demand)
    3. Hybrid (celebrities use fan-out on read)
    """

    def __init__(self, db, cache, connection_service, post_service):
        self.db = db
        self.cache = cache
        self.connection_service = connection_service
        self.post_service = post_service
        self.celebrity_threshold = 1_000_000  # 1M followers

    def get_feed(self, user_id: str, limit: int = 20,
                cursor: str = None) -> Dict:
        """
        Get personalized feed for user

        Uses hybrid approach:
        - Regular users: pre-computed feed (fan-out on write)
        - Celebrities: on-demand aggregation (fan-out on read)
        """
        # Try cache first
        cache_key = f"feed:{user_id}:{cursor or 'latest'}"
        cached = self.cache.get(cache_key)
        if cached:
            return cached

        # Check if user follows celebrities
        following = self.connection_service.get_following(user_id, limit=5000)

        celebrity_ids = self._identify_celebrities(following)
        regular_ids = [uid for uid in following if uid not in celebrity_ids]

        # Get posts from regular users (pre-computed)
        regular_posts = self._get_precomputed_feed(user_id, regular_ids, limit)

        # Get posts from celebrities (on-demand)
        celebrity_posts = self._get_celebrity_posts(celebrity_ids, limit)

        # Merge and rank
        all_posts = regular_posts + celebrity_posts
        ranked_posts = self._rank_posts(all_posts, user_id, limit)

        # Add post details
        feed_items = []
        for post_id, score in ranked_posts:
            post = self.post_service.get_post(post_id)
            if post:
                feed_items.append({
                    'post': post.to_dict(),
                    'score': score
                })

        # Generate next cursor
        next_cursor = None
        if len(feed_items) >= limit:
            next_cursor = feed_items[-1]['post']['created_at']

        result = {
            'items': feed_items,
            'next_cursor': next_cursor
        }

        # Cache for 1 minute
        self.cache.set(cache_key, result, ttl=60)

        return result

    def on_post_created(self, event: Dict):
        """
        Handle post creation event (fan-out on write)

        Called by background worker consuming Kafka events
        """
        post_id = event['post_id']
        user_id = event['user_id']

        # Check if user is celebrity
        follower_count = self._get_follower_count(user_id)
        if follower_count > self.celebrity_threshold:
            # Skip fan-out for celebrities (use fan-out on read instead)
            return

        # Get followers
        followers = self.connection_service.get_followers(
            user_id,
            limit=100000
        )

        # Add post to each follower's feed (batch operation)
        self._batch_add_to_feeds(followers, post_id, event['created_at'])

    def _get_precomputed_feed(self, user_id: str, following: List[str],
                             limit: int) -> List[str]:
        """Get pre-computed feed from cache/database"""
        # Query feed table
        results = self.db.query(
            """
            SELECT post_id FROM user_feeds
            WHERE user_id = %s
            ORDER BY created_at DESC
            LIMIT %s
            """,
            (user_id, limit * 2)  # Fetch more for ranking
        )

        return [row['post_id'] for row in results]

    def _get_celebrity_posts(self, celebrity_ids: List[str],
                            limit: int) -> List[str]:
        """Get recent posts from celebrities"""
        if not celebrity_ids:
            return []

        # Query recent posts
        placeholders = ', '.join(['%s'] * len(celebrity_ids))
        results = self.db.query(
            f"""
            SELECT post_id FROM posts
            WHERE user_id IN ({placeholders})
              AND created_at > %s
            ORDER BY created_at DESC
            LIMIT %s
            """,
            (*celebrity_ids, int(time.time()) - 86400 * 7, limit)  # Last 7 days
        )

        return [row['post_id'] for row in results]

    def _rank_posts(self, post_ids: List[str], user_id: str,
                   limit: int) -> List[tuple]:
        """
        Rank posts using engagement signals

        Scoring factors:
        - Recency (time decay)
        - Engagement (likes, comments, shares)
        - User affinity (interaction history)
        - Content type preference
        """
        scored_posts = []

        for post_id in post_ids:
            post = self.post_service.get_post(post_id)
            if not post:
                continue

            score = self._calculate_post_score(post, user_id)
            scored_posts.append((post_id, score))

        # Sort by score (descending)
        scored_posts.sort(key=lambda x: x[1], reverse=True)

        return scored_posts[:limit]

    def _calculate_post_score(self, post: Post, user_id: str) -> float:
        """
        Calculate post relevance score

        Score = recency_score + engagement_score + affinity_score
        """
        import math

        # Recency score (exponential decay)
        age_hours = (time.time() - post.created_at) / 3600
        recency_score = math.exp(-age_hours / 24)  # 24-hour half-life

        # Engagement score (normalized)
        total_engagement = post.like_count + post.comment_count * 2 + post.share_count * 3
        engagement_score = math.log(total_engagement + 1) / 10

        # Affinity score (how much user interacts with post author)
        affinity_score = self._get_user_affinity(user_id, post.user_id)

        # Combined score
        score = (
            0.5 * recency_score +
            0.3 * engagement_score +
            0.2 * affinity_score
        )

        return score

    def _get_user_affinity(self, user_id: str, author_id: str) -> float:
        """
        Calculate affinity between user and author

        Based on:
        - Like frequency
        - Comment frequency
        - Profile views
        """
        # Simplified: return 0.5 for now
        # In production, query interaction history
        return 0.5

    def _identify_celebrities(self, user_ids: List[str]) -> List[str]:
        """Identify celebrity accounts (>1M followers)"""
        if not user_ids:
            return []

        placeholders = ', '.join(['%s'] * len(user_ids))
        results = self.db.query(
            f"""
            SELECT user_id FROM users
            WHERE user_id IN ({placeholders})
              AND followers_count > %s
            """,
            (*user_ids, self.celebrity_threshold)
        )

        return [row['user_id'] for row in results]

    def _get_follower_count(self, user_id: str) -> int:
        """Get follower count for user"""
        result = self.db.query_one(
            "SELECT followers_count FROM users WHERE user_id = %s",
            (user_id,)
        )

        return result['followers_count'] if result else 0

    def _batch_add_to_feeds(self, user_ids: List[str], post_id: str,
                           created_at: int):
        """Add post to multiple user feeds (batch)"""
        # Batch insert for efficiency
        values = [
            (user_id, post_id, created_at)
            for user_id in user_ids
        ]

        # Use batch insert (implementation depends on DB)
        # For MySQL:
        self.db.execute_many(
            """
            INSERT INTO user_feeds (user_id, post_id, created_at)
            VALUES (%s, %s, %s)
            """,
            values
        )

        # Invalidate feed caches
        for user_id in user_ids:
            self.cache.delete(f"feed:{user_id}:latest")
```

### 5. Notification Service

**Real-time Notifications:**

```python
from enum import Enum
from typing import List, Dict
import json

class NotificationType(Enum):
    LIKE = "like"
    COMMENT = "comment"
    FOLLOW = "follow"
    MENTION = "mention"
    SHARE = "share"


class NotificationService:
    """
    Handle real-time notifications

    Delivery channels:
    - WebSocket (in-app)
    - Push notifications (mobile)
    - Email (digest)
    """

    def __init__(self, db, cache, websocket_manager, push_service):
        self.db = db
        self.cache = cache
        self.websocket_manager = websocket_manager
        self.push_service = push_service

    def create_notification(self, user_id: str, notification_type: NotificationType,
                          actor_id: str, entity_id: str,
                          metadata: Dict = None):
        """
        Create and send notification

        Args:
            user_id: Recipient
            notification_type: Type of notification
            actor_id: Who triggered the notification
            entity_id: Related entity (post_id, comment_id, etc.)
            metadata: Additional data
        """
        notification_id = self._generate_notification_id()

        notification = {
            'notification_id': notification_id,
            'user_id': user_id,
            'type': notification_type.value,
            'actor_id': actor_id,
            'entity_id': entity_id,
            'metadata': metadata or {},
            'created_at': int(time.time()),
            'read': False
        }

        # Store in database
        self.db.execute(
            """
            INSERT INTO notifications
            (notification_id, user_id, type, actor_id, entity_id,
             metadata, created_at, read)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
            """,
            (notification_id, user_id, notification_type.value,
             actor_id, entity_id, json.dumps(metadata or {}),
             notification['created_at'], False)
        )

        # Send via WebSocket (if user online)
        self.websocket_manager.send_to_user(user_id, {
            'type': 'notification',
            'data': notification
        })

        # Send push notification (if user has enabled)
        if self._should_send_push(user_id, notification_type):
            self.push_service.send(user_id, self._format_push_message(notification))

        # Increment unread counter
        self._increment_unread_count(user_id)

        return notification_id

    def get_notifications(self, user_id: str, limit: int = 20,
                         cursor: str = None) -> Dict:
        """Get notifications for user"""
        # Build query
        query = """
            SELECT * FROM notifications
            WHERE user_id = %s
        """
        params = [user_id]

        if cursor:
            query += " AND created_at < %s"
            params.append(int(cursor))

        query += " ORDER BY created_at DESC LIMIT %s"
        params.append(limit)

        # Execute
        results = self.db.query(query, params)

        notifications = []
        for row in results:
            notifications.append({
                'notification_id': row['notification_id'],
                'type': row['type'],
                'actor_id': row['actor_id'],
                'entity_id': row['entity_id'],
                'metadata': json.loads(row['metadata']),
                'created_at': row['created_at'],
                'read': row['read']
            })

        # Next cursor
        next_cursor = None
        if len(notifications) >= limit:
            next_cursor = str(notifications[-1]['created_at'])

        return {
            'notifications': notifications,
            'next_cursor': next_cursor
        }

    def mark_as_read(self, user_id: str, notification_ids: List[str]):
        """Mark notifications as read"""
        if not notification_ids:
            return

        placeholders = ', '.join(['%s'] * len(notification_ids))

        self.db.execute(
            f"""
            UPDATE notifications
            SET read = TRUE
            WHERE user_id = %s AND notification_id IN ({placeholders})
            """,
            (user_id, *notification_ids)
        )

        # Decrement unread counter
        self._decrement_unread_count(user_id, len(notification_ids))

    def get_unread_count(self, user_id: str) -> int:
        """Get unread notification count"""
        # Try cache
        cache_key = f"unread_count:{user_id}"
        cached = self.cache.get(cache_key)
        if cached is not None:
            return int(cached)

        # Query database
        result = self.db.query_one(
            """
            SELECT COUNT(*) as count FROM notifications
            WHERE user_id = %s AND read = FALSE
            """,
            (user_id,)
        )

        count = result['count'] if result else 0

        # Cache
        self.cache.set(cache_key, str(count), ttl=300)

        return count

    def _should_send_push(self, user_id: str,
                         notification_type: NotificationType) -> bool:
        """Check if user wants push notifications for this type"""
        # Query user preferences
        # For now, return True
        return True

    def _format_push_message(self, notification: Dict) -> Dict:
        """Format notification for push service"""
        type_messages = {
            'like': 'liked your post',
            'comment': 'commented on your post',
            'follow': 'started following you',
            'mention': 'mentioned you in a post'
        }

        message = type_messages.get(notification['type'], 'new notification')

        return {
            'title': 'Social Network',
            'body': f"@{notification['actor_id']} {message}",
            'data': notification
        }

    def _increment_unread_count(self, user_id: str):
        """Increment unread notification count"""
        cache_key = f"unread_count:{user_id}"
        self.cache.incr(cache_key)
        self.cache.expire(cache_key, 300)

    def _decrement_unread_count(self, user_id: str, count: int):
        """Decrement unread notification count"""
        cache_key = f"unread_count:{user_id}"
        self.cache.decr(cache_key, count)

    def _generate_notification_id(self) -> str:
        """Generate unique notification ID"""
        import uuid
        return str(uuid.uuid4())


class WebSocketManager:
    """Manage WebSocket connections for real-time updates"""

    def __init__(self):
        # user_id -> list of WebSocket connections
        self.connections = {}

    def register(self, user_id: str, websocket):
        """Register WebSocket connection for user"""
        if user_id not in self.connections:
            self.connections[user_id] = []
        self.connections[user_id].append(websocket)

    def unregister(self, user_id: str, websocket):
        """Unregister WebSocket connection"""
        if user_id in self.connections:
            self.connections[user_id].remove(websocket)
            if not self.connections[user_id]:
                del self.connections[user_id]

    def send_to_user(self, user_id: str, message: Dict):
        """Send message to all user's connections"""
        if user_id not in self.connections:
            return

        for ws in self.connections[user_id]:
            try:
                ws.send(json.dumps(message))
            except:
                # Connection closed
                self.unregister(user_id, ws)
```

## Database Schema

```sql
-- Users table
CREATE TABLE users (
    user_id VARCHAR(64) PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(255),
    bio TEXT,
    profile_picture_url TEXT,
    verified BOOLEAN DEFAULT FALSE,
    followers_count INT DEFAULT 0,
    following_count INT DEFAULT 0,
    created_at BIGINT NOT NULL,
    INDEX idx_username (username),
    INDEX idx_email (email)
) PARTITION BY HASH(user_id) PARTITIONS 100;

-- Connections (follow/friend relationships)
CREATE TABLE connections (
    follower_id VARCHAR(64),
    followee_id VARCHAR(64),
    created_at BIGINT NOT NULL,
    PRIMARY KEY (follower_id, followee_id),
    INDEX idx_follower (follower_id),
    INDEX idx_followee (followee_id),
    FOREIGN KEY (follower_id) REFERENCES users(user_id),
    FOREIGN KEY (followee_id) REFERENCES users(user_id)
) PARTITION BY HASH(follower_id) PARTITIONS 100;

-- Posts table
CREATE TABLE posts (
    post_id VARCHAR(64) PRIMARY KEY,
    user_id VARCHAR(64) NOT NULL,
    content TEXT,
    media_urls JSON,
    created_at BIGINT NOT NULL,
    like_count INT DEFAULT 0,
    comment_count INT DEFAULT 0,
    share_count INT DEFAULT 0,
    INDEX idx_user_time (user_id, created_at),
    INDEX idx_created_at (created_at),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
) PARTITION BY RANGE(created_at) (
    PARTITION p_2024_01 VALUES LESS THAN (1706745600),
    PARTITION p_2024_02 VALUES LESS THAN (1709251200),
    -- Add partitions monthly
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Post likes
CREATE TABLE post_likes (
    post_id VARCHAR(64),
    user_id VARCHAR(64),
    created_at BIGINT NOT NULL,
    PRIMARY KEY (post_id, user_id),
    INDEX idx_user_time (user_id, created_at),
    FOREIGN KEY (post_id) REFERENCES posts(post_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
) PARTITION BY HASH(post_id) PARTITIONS 100;

-- Comments
CREATE TABLE comments (
    comment_id VARCHAR(64) PRIMARY KEY,
    post_id VARCHAR(64) NOT NULL,
    user_id VARCHAR(64) NOT NULL,
    content TEXT NOT NULL,
    created_at BIGINT NOT NULL,
    like_count INT DEFAULT 0,
    INDEX idx_post_time (post_id, created_at),
    INDEX idx_user_time (user_id, created_at),
    FOREIGN KEY (post_id) REFERENCES posts(post_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
) PARTITION BY HASH(post_id) PARTITIONS 100;

-- User feeds (fan-out on write)
CREATE TABLE user_feeds (
    user_id VARCHAR(64),
    post_id VARCHAR(64),
    created_at BIGINT NOT NULL,
    PRIMARY KEY (user_id, created_at, post_id),
    INDEX idx_user_time (user_id, created_at)
) PARTITION BY HASH(user_id) PARTITIONS 100;

-- Notifications
CREATE TABLE notifications (
    notification_id VARCHAR(64) PRIMARY KEY,
    user_id VARCHAR(64) NOT NULL,
    type VARCHAR(20) NOT NULL,
    actor_id VARCHAR(64) NOT NULL,
    entity_id VARCHAR(64),
    metadata JSON,
    created_at BIGINT NOT NULL,
    read BOOLEAN DEFAULT FALSE,
    INDEX idx_user_time (user_id, created_at),
    INDEX idx_user_unread (user_id, read, created_at),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (actor_id) REFERENCES users(user_id)
) PARTITION BY HASH(user_id) PARTITIONS 100;
```

## Scalability Strategies

### 1. Database Sharding

**Shard by User ID:**
```
Shard 1: user_id hash % 100 = 0-9
Shard 2: user_id hash % 100 = 10-19
...
Shard 10: user_id hash % 100 = 90-99
```

**Challenges:**
- Cross-shard queries (feed generation)
- Hotspots (celebrity users)

**Solutions:**
- Denormalization
- Cache heavily
- Separate celebrity handling

### 2. Caching Strategy

**Multi-level cache:**

```python
class CacheStrategy:
    """
    L1: Application memory (LRU cache)
    L2: Redis cluster (distributed)
    L3: CDN (media files)
    """

    def __init__(self, l1_cache, redis_cluster):
        self.l1 = l1_cache
        self.l2 = redis_cluster

    def get_user(self, user_id: str):
        # Try L1
        user = self.l1.get(f"user:{user_id}")
        if user:
            return user

        # Try L2
        user = self.l2.get(f"user:{user_id}")
        if user:
            self.l1.set(f"user:{user_id}", user)
            return user

        # Query database
        user = db.get_user(user_id)
        if user:
            self.l1.set(f"user:{user_id}", user)
            self.l2.set(f"user:{user_id}", user, ttl=3600)

        return user
```

**Cache What:**
- User profiles (high read)
- Hot posts (trending)
- Feed data (first page)
- Follower counts

### 3. CDN for Media

**Architecture:**
```
User Upload → App Server → S3 → CloudFront (CDN)
                              ↓
                         Image Processing
                         (resize, compress)
```

### 4. Read Replicas

**Master-Slave Replication:**
- Write to master
- Read from replicas (5-10x read traffic)
- Eventual consistency acceptable

## Interview Tips

### Common Questions

**Q: How do you generate the news feed?**
- **Fan-out on write**: Pre-compute feeds (fast read, slow write, storage intensive)
- **Fan-out on read**: Compute on demand (fast write, slow read, less storage)
- **Hybrid**: Regular users → fan-out on write, Celebrities → fan-out on read

**Q: How do you handle celebrity users (1M+ followers)?**
- Use fan-out on read (don't write to 1M feeds)
- Cache celebrity posts heavily
- Use separate infrastructure

**Q: How to rank posts in feed?**
- Recency (time decay)
- Engagement (likes, comments, shares)
- User affinity (interaction history)
- Content type preference
- ML-based ranking

**Q: How to scale database?**
- Shard by user_id
- Partition posts by time
- Read replicas
- Denormalization

**Q: How to handle hotspots?**
- Consistent hashing with virtual nodes
- Cache aggressively
- Rate limiting
- Separate celebrity handling

## Key Takeaways

1. **Feed Generation**: Hybrid approach (fan-out on write for regular, fan-out on read for celebrities)
2. **Database Sharding**: Partition by user_id, time-based partitioning for posts
3. **Caching**: Multi-level cache (application, Redis, CDN)
4. **Real-time Updates**: WebSocket for notifications
5. **Scalability**: Read replicas, CDN for media, async processing with queues
6. **Ranking**: Combine recency + engagement + affinity

Building a social network requires mastering distributed systems, caching strategies, and real-time data processing!
