# Netflix Streaming Platform - OOD Design

**Difficulty:** Advanced
**Interview Frequency:** High
**Key Concepts:** Content Management, User Profiles, Recommendations, Watchlists
**Companies:** Netflix, Hulu, Disney+, YouTube

---

## Problem Statement

Design a video streaming platform with content library, user profiles, watchlists, viewing history, and recommendations.

---

## Implementation

```python
from enum import Enum
from typing import List, Optional, Dict
from datetime import datetime, timedelta


class ContentType(Enum):
    MOVIE = "Movie"
    SERIES = "Series"
    DOCUMENTARY = "Documentary"


class Genre(Enum):
    ACTION = "Action"
    COMEDY = "Comedy"
    DRAMA = "Drama"
    SCIFI = "Sci-Fi"


class Content:
    def __init__(self, content_id: int, title: str, content_type: ContentType, genre: Genre, duration_minutes: int):
        self.content_id = content_id
        self.title = title
        self.content_type = content_type
        self.genre = genre
        self.duration_minutes = duration_minutes
        self.rating = 0.0


class Episode:
    def __init__(self, episode_id: int, title: str, season: int, episode_number: int, duration_minutes: int):
        self.episode_id = episode_id
        self.title = title
        self.season = season
        self.episode_number = episode_number
        self.duration_minutes = duration_minutes


class Series(Content):
    def __init__(self, content_id: int, title: str, genre: Genre):
        super().__init__(content_id, title, ContentType.SERIES, genre, 0)
        self.episodes: List[Episode] = []

    def add_episode(self, episode: Episode):
        self.episodes.append(episode)


class WatchHistory:
    def __init__(self, content: Content, timestamp: datetime, duration_watched: int):
        self.content = content
        self.timestamp = timestamp
        self.duration_watched = duration_watched  # seconds

    def is_completed(self) -> bool:
        return self.duration_watched >= (self.content.duration_minutes * 60 * 0.9)  # 90% watched


class UserProfile:
    def __init__(self, profile_id: int, name: str):
        self.profile_id = profile_id
        self.name = name
        self.watch_history: List[WatchHistory] = []
        self.watchlist: List[Content] = []

    def add_to_watchlist(self, content: Content):
        if content not in self.watchlist:
            self.watchlist.append(content)

    def watch_content(self, content: Content, duration_seconds: int):
        history = WatchHistory(content, datetime.now(), duration_seconds)
        self.watch_history.append(history)

    def get_recommended(self, all_content: List[Content]) -> List[Content]:
        # Simple recommendation based on genre preference
        watched_genres = [h.content.genre for h in self.watch_history]
        if not watched_genres:
            return all_content[:5]

        # Most common genre
        most_common = max(set(watched_genres), key=watched_genres.count)
        recommendations = [c for c in all_content if c.genre == most_common and c not in [h.content for h in self.watch_history]]
        return recommendations[:5]


class Subscription:
    def __init__(self, user_id: int, plan: str):
        self.user_id = user_id
        self.plan = plan  # Basic, Standard, Premium
        self.start_date = datetime.now()
        self.is_active = True

    def can_watch(self) -> bool:
        return self.is_active


class NetflixPlatform:
    def __init__(self):
        self.content_library: List[Content] = []
        self.subscriptions: Dict[int, Subscription] = {}
        self.profiles: Dict[int, UserProfile] = {}

    def add_content(self, content: Content):
        self.content_library.append(content)

    def create_profile(self, user_id: int, profile: UserProfile):
        self.profiles[profile.profile_id] = profile

    def search_content(self, query: str) -> List[Content]:
        return [c for c in self.content_library if query.lower() in c.title.lower()]

    def stream_content(self, profile_id: int, content_id: int) -> bool:
        profile = self.profiles.get(profile_id)
        content = next((c for c in self.content_library if c.content_id == content_id), None)

        if not profile or not content:
            return False

        print(f"✓ Streaming '{content.title}' on profile '{profile.name}'")
        return True


def main():
    platform = NetflixPlatform()

    # Add content
    movie1 = Content(1, "Inception", ContentType.MOVIE, Genre.SCIFI, 148)
    movie2 = Content(2, "The Matrix", ContentType.MOVIE, Genre.SCIFI, 136)
    movie3 = Content(3, "Superbad", ContentType.MOVIE, Genre.COMEDY, 113)

    platform.add_content(movie1)
    platform.add_content(movie2)
    platform.add_content(movie3)

    # Create profile
    profile = UserProfile(1, "Alice")
    platform.create_profile(1, profile)

    # Watch content
    profile.watch_content(movie1, 8000)  # watched most of it

    # Get recommendations
    recommendations = profile.get_recommended(platform.content_library)
    print(f"\nRecommendations for {profile.name}:")
    for content in recommendations:
        print(f"  - {content.title} ({content.genre.value})")

    # Add to watchlist
    profile.add_to_watchlist(movie2)
    print(f"\nWatchlist: {[c.title for c in profile.watchlist]}")


if __name__ == "__main__":
    main()
```

---

## Design Patterns
- **Strategy:** Different recommendation algorithms
- **Observer:** Notify on new content
- **Proxy:** Streaming with quality adaptation

This tests content management, user personalization, and recommendation systems.
