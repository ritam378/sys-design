# Facebook News Feed - OOD Design

**Difficulty:** Advanced
**Interview Frequency:** Very High
**Key Concepts:** Feed Generation, Post Ranking, Timeline, Aggregation
**Companies:** Meta (Facebook), Twitter, Instagram, Social Media Platforms

---

## Problem Statement

Design a news feed system like Facebook that generates personalized feeds based on user connections, post engagement, and ranking algorithms. Support posts, comments, likes, shares, and content moderation.

---

## Requirements

### Core Features
1. User profiles and friendships
2. Create posts (text, images, videos)
3. Interactions: likes, comments, shares
4. Personalized news feed generation
5. Post visibility controls (public, friends, custom)
6. Chronological and ranked feed algorithms

### Advanced Features
7. Trending posts detection
8. Feed refresh and pagination
9. Content moderation and reporting
10. Post pinning and highlighting

---

## Implementation

```python
from enum import Enum
from typing import List, Optional, Set, Dict
from datetime import datetime, timedelta
from collections import defaultdict
import heapq


class PostType(Enum):
    TEXT = "Text"
    IMAGE = "Image"
    VIDEO = "Video"
    LINK = "Link"


class Visibility(Enum):
    PUBLIC = "Public"
    FRIENDS = "Friends Only"
    PRIVATE = "Only Me"
    CUSTOM = "Custom"


class InteractionType(Enum):
    LIKE = "Like"
    LOVE = "Love"
    HAHA = "Haha"
    WOW = "Wow"
    SAD = "Sad"
    ANGRY = "Angry"


class User:
    def __init__(self, user_id: int, name: str, email: str):
        self.user_id = user_id
        self.name = name
        self.email = email
        self.friends: Set[int] = set()
        self.created_at = datetime.now()

    def add_friend(self, friend_id: int):
        self.friends.add(friend_id)

    def remove_friend(self, friend_id: int):
        self.friends.discard(friend_id)

    def is_friend(self, user_id: int) -> bool:
        return user_id in self.friends

    def __str__(self) -> str:
        return f"{self.name} (ID: {self.user_id})"


class Interaction:
    def __init__(self, user_id: int, interaction_type: InteractionType):
        self.user_id = user_id
        self.interaction_type = interaction_type
        self.timestamp = datetime.now()


class Comment:
    _comment_counter = 1

    def __init__(self, user_id: int, content: str):
        self.comment_id = Comment._comment_counter
        Comment._comment_counter += 1
        self.user_id = user_id
        self.content = content
        self.timestamp = datetime.now()
        self.likes: Set[int] = set()
        self.replies: List['Comment'] = []

    def add_like(self, user_id: int):
        self.likes.add(user_id)

    def add_reply(self, reply: 'Comment'):
        self.replies.append(reply)


class Post:
    _post_counter = 1

    def __init__(self, author_id: int, content: str, post_type: PostType = PostType.TEXT,
                 visibility: Visibility = Visibility.FRIENDS):
        self.post_id = Post._post_counter
        Post._post_counter += 1
        self.author_id = author_id
        self.content = content
        self.post_type = post_type
        self.visibility = visibility
        self.created_at = datetime.now()
        self.updated_at = datetime.now()

        # Engagement
        self.interactions: List[Interaction] = []
        self.comments: List[Comment] = []
        self.shares: Set[int] = set()  # user_ids who shared

        # Moderation
        self.is_deleted = False
        self.is_pinned = False

    def add_interaction(self, user_id: int, interaction_type: InteractionType):
        # Remove previous interaction from same user
        self.interactions = [i for i in self.interactions if i.user_id != user_id]
        self.interactions.append(Interaction(user_id, interaction_type))

    def remove_interaction(self, user_id: int):
        self.interactions = [i for i in self.interactions if i.user_id != user_id]

    def add_comment(self, comment: Comment):
        self.comments.append(comment)

    def share(self, user_id: int):
        self.shares.add(user_id)

    def get_engagement_score(self) -> float:
        """
        Calculate engagement score based on interactions, comments, and shares.
        Used for ranking posts in feed.
        """
        # Weight different engagement types
        interaction_score = len(self.interactions) * 1.0
        comment_score = len(self.comments) * 2.0  # Comments are more valuable
        share_score = len(self.shares) * 3.0  # Shares are most valuable

        # Time decay: newer posts get higher scores
        hours_old = (datetime.now() - self.created_at).total_seconds() / 3600
        time_decay = 1.0 / (1.0 + hours_old / 24.0)  # Decay over days

        return (interaction_score + comment_score + share_score) * time_decay

    def get_like_count(self) -> int:
        return len([i for i in self.interactions if i.interaction_type == InteractionType.LIKE])

    def get_comment_count(self) -> int:
        return len(self.comments)

    def get_share_count(self) -> int:
        return len(self.shares)

    def __str__(self) -> str:
        return f"Post #{self.post_id} by User {self.author_id}: {self.content[:50]}..."


class FeedRankingStrategy:
    """Strategy pattern for different feed ranking algorithms"""

    def rank_posts(self, posts: List[Post], user: User) -> List[Post]:
        raise NotImplementedError


class ChronologicalRanking(FeedRankingStrategy):
    """Simple chronological ordering (newest first)"""

    def rank_posts(self, posts: List[Post], user: User) -> List[Post]:
        return sorted(posts, key=lambda p: p.created_at, reverse=True)


class EngagementRanking(FeedRankingStrategy):
    """Rank by engagement score (likes, comments, shares, recency)"""

    def rank_posts(self, posts: List[Post], user: User) -> List[Post]:
        return sorted(posts, key=lambda p: p.get_engagement_score(), reverse=True)


class PersonalizedRanking(FeedRankingStrategy):
    """Personalized ranking based on user's interaction history"""

    def rank_posts(self, posts: List[Post], user: User) -> List[Post]:
        def personalized_score(post: Post) -> float:
            base_score = post.get_engagement_score()

            # Boost posts from close friends (users the current user interacts with often)
            author_boost = 1.5 if post.author_id in user.friends else 1.0

            # Boost posts user has interacted with
            user_interacted = any(i.user_id == user.user_id for i in post.interactions)
            interaction_boost = 1.3 if user_interacted else 1.0

            return base_score * author_boost * interaction_boost

        return sorted(posts, key=personalized_score, reverse=True)


class NewsFeed:
    def __init__(self):
        self.users: Dict[int, User] = {}
        self.posts: Dict[int, Post] = {}
        self.user_posts: Dict[int, List[int]] = defaultdict(list)  # user_id -> post_ids
        self.ranking_strategy: FeedRankingStrategy = EngagementRanking()

    def add_user(self, user: User):
        self.users[user.user_id] = user
        self.user_posts[user.user_id] = []

    def create_friendship(self, user1_id: int, user2_id: int):
        """Create bidirectional friendship"""
        user1 = self.users.get(user1_id)
        user2 = self.users.get(user2_id)

        if user1 and user2:
            user1.add_friend(user2_id)
            user2.add_friend(user1_id)
            print(f"✓ {user1.name} and {user2.name} are now friends")

    def create_post(self, author_id: int, content: str, post_type: PostType = PostType.TEXT,
                   visibility: Visibility = Visibility.FRIENDS) -> Optional[Post]:
        if author_id not in self.users:
            return None

        post = Post(author_id, content, post_type, visibility)
        self.posts[post.post_id] = post
        self.user_posts[author_id].append(post.post_id)

        print(f"✓ Post #{post.post_id} created by {self.users[author_id].name}")
        return post

    def like_post(self, post_id: int, user_id: int) -> bool:
        post = self.posts.get(post_id)
        if not post or post.is_deleted:
            return False

        post.add_interaction(user_id, InteractionType.LIKE)
        return True

    def comment_on_post(self, post_id: int, user_id: int, content: str) -> Optional[Comment]:
        post = self.posts.get(post_id)
        if not post or post.is_deleted:
            return None

        comment = Comment(user_id, content)
        post.add_comment(comment)
        return comment

    def share_post(self, post_id: int, user_id: int) -> bool:
        post = self.posts.get(post_id)
        if not post or post.is_deleted:
            return False

        post.share(user_id)
        return True

    def get_feed(self, user_id: int, limit: int = 20) -> List[Post]:
        """
        Generate personalized news feed for user.
        Shows posts from friends and own posts.
        """
        user = self.users.get(user_id)
        if not user:
            return []

        eligible_posts = []

        # Get all friends' posts and own posts
        relevant_users = user.friends | {user_id}

        for uid in relevant_users:
            for post_id in self.user_posts.get(uid, []):
                post = self.posts.get(post_id)
                if post and not post.is_deleted and self._can_view_post(user_id, post):
                    eligible_posts.append(post)

        # Apply ranking strategy
        ranked_posts = self.ranking_strategy.rank_posts(eligible_posts, user)

        # Return top posts (pagination)
        return ranked_posts[:limit]

    def _can_view_post(self, viewer_id: int, post: Post) -> bool:
        """Check if viewer can see the post based on visibility settings"""
        if post.visibility == Visibility.PUBLIC:
            return True

        if post.visibility == Visibility.PRIVATE:
            return viewer_id == post.author_id

        if post.visibility == Visibility.FRIENDS:
            author = self.users.get(post.author_id)
            return viewer_id == post.author_id or (author and author.is_friend(viewer_id))

        return False

    def set_ranking_strategy(self, strategy: FeedRankingStrategy):
        """Change feed ranking algorithm"""
        self.ranking_strategy = strategy
        print(f"✓ Feed ranking strategy updated to {strategy.__class__.__name__}")

    def get_trending_posts(self, limit: int = 10, hours: int = 24) -> List[Post]:
        """
        Get trending posts based on recent engagement.
        Only considers posts from last N hours.
        """
        cutoff_time = datetime.now() - timedelta(hours=hours)
        recent_posts = [p for p in self.posts.values()
                       if p.created_at >= cutoff_time and not p.is_deleted]

        # Sort by engagement score
        trending = sorted(recent_posts, key=lambda p: p.get_engagement_score(), reverse=True)
        return trending[:limit]

    def search_posts(self, query: str) -> List[Post]:
        """Search posts by content"""
        query_lower = query.lower()
        results = [p for p in self.posts.values()
                  if query_lower in p.content.lower() and not p.is_deleted]
        return sorted(results, key=lambda p: p.created_at, reverse=True)


def main():
    feed_system = NewsFeed()

    # Create users
    alice = User(1, "Alice Johnson", "alice@email.com")
    bob = User(2, "Bob Smith", "bob@email.com")
    charlie = User(3, "Charlie Davis", "charlie@email.com")
    diana = User(4, "Diana Lee", "diana@email.com")

    for user in [alice, bob, charlie, diana]:
        feed_system.add_user(user)

    print("=== Facebook News Feed Demo ===\n")

    # Create friendships
    feed_system.create_friendship(alice.user_id, bob.user_id)
    feed_system.create_friendship(alice.user_id, charlie.user_id)
    feed_system.create_friendship(bob.user_id, charlie.user_id)

    print("\n=== Creating Posts ===")
    # Create posts
    post1 = feed_system.create_post(alice.user_id, "Just finished a great book! 📚", PostType.TEXT)
    post2 = feed_system.create_post(bob.user_id, "Beautiful sunset today 🌅", PostType.IMAGE)
    post3 = feed_system.create_post(charlie.user_id, "Excited about the new project!", PostType.TEXT)
    post4 = feed_system.create_post(alice.user_id, "Anyone want to grab coffee?", PostType.TEXT)

    print("\n=== Interactions ===")
    # Add interactions
    feed_system.like_post(post1.post_id, bob.user_id)
    feed_system.like_post(post1.post_id, charlie.user_id)
    feed_system.like_post(post2.post_id, alice.user_id)
    feed_system.like_post(post2.post_id, charlie.user_id)
    feed_system.like_post(post3.post_id, alice.user_id)

    print(f"✓ Post #{post1.post_id} has {post1.get_like_count()} likes")
    print(f"✓ Post #{post2.post_id} has {post2.get_like_count()} likes")

    # Add comments
    comment1 = feed_system.comment_on_post(post1.post_id, bob.user_id, "Which book?")
    comment2 = feed_system.comment_on_post(post1.post_id, charlie.user_id, "I'd love to read it too!")
    print(f"✓ Post #{post1.post_id} has {post1.get_comment_count()} comments")

    # Share post
    feed_system.share_post(post2.post_id, alice.user_id)
    print(f"✓ Post #{post2.post_id} has {post2.get_share_count()} shares")

    print("\n=== Alice's News Feed (Engagement Ranking) ===")
    alice_feed = feed_system.get_feed(alice.user_id, limit=5)
    for i, post in enumerate(alice_feed, 1):
        author = feed_system.users[post.author_id]
        print(f"{i}. {author.name}: {post.content}")
        print(f"   Engagement: {post.get_like_count()} likes, {post.get_comment_count()} comments, "
              f"{post.get_share_count()} shares")
        print(f"   Score: {post.get_engagement_score():.2f}")

    print("\n=== Switch to Chronological Ranking ===")
    feed_system.set_ranking_strategy(ChronologicalRanking())
    alice_feed_chrono = feed_system.get_feed(alice.user_id, limit=5)
    for i, post in enumerate(alice_feed_chrono, 1):
        author = feed_system.users[post.author_id]
        print(f"{i}. {author.name}: {post.content} (posted {post.created_at.strftime('%H:%M')})")

    print("\n=== Trending Posts ===")
    trending = feed_system.get_trending_posts(limit=3)
    print("Top trending posts:")
    for i, post in enumerate(trending, 1):
        author = feed_system.users[post.author_id]
        print(f"{i}. {author.name}: {post.content}")
        print(f"   Engagement score: {post.get_engagement_score():.2f}")

    print("\n=== Search Posts ===")
    results = feed_system.search_posts("book")
    print(f"Search results for 'book': {len(results)} post(s)")
    for post in results:
        author = feed_system.users[post.author_id]
        print(f"  - {author.name}: {post.content}")


if __name__ == "__main__":
    main()
```

---

## Design Patterns

### 1. Strategy Pattern
- **Usage:** Different feed ranking algorithms (Chronological, Engagement, Personalized)
- **Benefit:** Easy to switch between ranking strategies
```python
class FeedRankingStrategy:
    def rank_posts(self, posts: List[Post], user: User) -> List[Post]:
        raise NotImplementedError
```

### 2. Observer Pattern (Extension)
- **Usage:** Notify users of new posts, comments, likes from friends
- **Implementation:** Event-driven notifications

### 3. Composite Pattern
- **Usage:** Comments can have replies (tree structure)
- **Benefit:** Nested comment threads

### 4. Factory Pattern
- **Usage:** Create different post types (Text, Image, Video)
- **Benefit:** Extensible post creation

---

## SOLID Principles

### Single Responsibility Principle (SRP)
- `Post`: Manages post content and engagement
- `Comment`: Handles comment functionality
- `NewsFeed`: Feed generation and management
- `FeedRankingStrategy`: Ranking algorithm logic

### Open/Closed Principle (OCP)
- New ranking strategies can be added without modifying existing code
- New post types can be added by extending `Post`

### Liskov Substitution Principle (LSP)
- All ranking strategies implement `FeedRankingStrategy` interface
- Can substitute any strategy without breaking functionality

### Interface Segregation Principle (ISP)
- Separate interfaces for feed generation vs post management

### Dependency Inversion Principle (DIP)
- `NewsFeed` depends on `FeedRankingStrategy` abstraction, not concrete implementations

---

## Key Algorithms

### Engagement Score Calculation
```python
def get_engagement_score(self) -> float:
    interaction_score = len(self.interactions) * 1.0
    comment_score = len(self.comments) * 2.0
    share_score = len(self.shares) * 3.0

    hours_old = (datetime.now() - self.created_at).total_seconds() / 3600
    time_decay = 1.0 / (1.0 + hours_old / 24.0)

    return (interaction_score + comment_score + share_score) * time_decay
```

### Feed Generation
1. Collect posts from friends and self
2. Filter by visibility permissions
3. Apply ranking strategy
4. Paginate results

---

## Extensions

### 1. Advanced Ranking
- Machine learning-based personalization
- User affinity scores
- Content type preferences
- Negative feedback (hide, report)

### 2. Real-time Updates
- WebSocket connections for live feed updates
- Push notifications for interactions

### 3. Content Moderation
- Spam detection
- Hate speech filtering
- Report and review system
- Automated content flagging

### 4. Analytics
- Impression tracking
- Click-through rates
- Engagement metrics per post
- A/B testing for ranking algorithms

### 5. Stories and Ephemeral Content
- Time-limited posts (24 hours)
- Story viewer tracking
- Highlight creation

---

## Interview Tips

### Clarifying Questions
1. **Scale**: How many users? How many posts per day?
2. **Feed Size**: Average number of friends per user?
3. **Real-time**: Should feed update in real-time or on refresh?
4. **Features**: Which features are critical (likes, comments, shares)?
5. **Ranking**: Simple chronological or complex algorithmic ranking?

### Common Follow-ups

1. **Scalability**: "How would you scale to 1B users?"
   - Separate services: Feed Generation, Post Storage, Engagement Service
   - Sharding by user ID
   - Caching (Redis) for hot feeds
   - CDN for media content

2. **Performance**: "How to generate feed quickly?"
   - Pre-compute feeds asynchronously
   - Cache recent posts per user
   - Fan-out on write vs fan-out on read tradeoff
   - Limit feed generation to top N friends

3. **Feed Generation Approaches**:
   - **Fan-out on write**: Pre-generate feeds when post is created (fast reads, slow writes)
   - **Fan-out on read**: Generate feed on demand (fast writes, slow reads)
   - **Hybrid**: Pre-compute for active users, on-demand for others

4. **Consistency**: "Handle post deletion/updates?"
   - Soft deletes with `is_deleted` flag
   - Eventual consistency acceptable for feeds
   - Immediate update for post author

---

## Time Complexity

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| Create Post | O(1) | Simple insertion |
| Like Post | O(n) | n = existing interactions (to remove duplicate) |
| Comment | O(1) | Append to list |
| Generate Feed | O(F * P + P log P) | F = friends, P = posts, sorting |
| Trending Posts | O(N log K) | N = all posts, K = limit (heap) |
| Search Posts | O(N) | Linear scan (use search index in production) |

---

## System Design Considerations

When scaling to production:

1. **Database**:
   - Posts: Cassandra (wide-column, time-series)
   - User graph: Neo4j or adjacency list
   - Feed cache: Redis (TTL-based)

2. **Architecture**:
   - Post Service: Handle CRUD operations
   - Feed Service: Generate and rank feeds
   - Engagement Service: Track likes, comments, shares
   - Notification Service: Push notifications

3. **Optimization**:
   - CDN for images/videos
   - Message queue (Kafka) for async processing
   - Elasticsearch for post search
   - ML models for ranking (TensorFlow Serving)

4. **Monitoring**:
   - Feed latency metrics
   - Engagement rates
   - Cache hit rates
   - Error tracking

---

This tests feed aggregation, ranking algorithms, and social media data modeling.
