# Stack Overflow Q&A Platform - OOD Design

**Difficulty:** Advanced
**Interview Frequency:** High
**Key Concepts:** Reputation System, Voting, Tags, Search
**Companies:** Stack Overflow, Quora, Reddit, Q&A Platforms

---

## Problem Statement

Design a Q&A platform like Stack Overflow with questions, answers, voting, reputation system, tags, search, and content moderation. Support accepted answers, bounties, and user privileges based on reputation.

---

## Requirements

### Core Features
1. User accounts with reputation points
2. Ask questions with title, body, and tags
3. Post answers to questions
4. Upvote/downvote questions and answers
5. Accept answer (by question author)
6. Comment on questions and answers
7. Search questions by tags, keywords
8. Reputation-based privileges

### Advanced Features
9. Bounty system for questions
10. Question closing/reopening
11. Edit history and versioning
12. Badge system for achievements

---

## Implementation

```python
from enum import Enum
from typing import List, Optional, Set, Dict
from datetime import datetime, timedelta
from collections import defaultdict


class VoteType(Enum):
    UPVOTE = 1
    DOWNVOTE = -1


class QuestionStatus(Enum):
    OPEN = "Open"
    CLOSED = "Closed"
    DELETED = "Deleted"


class Privilege(Enum):
    COMMENT = 50
    VOTE_UP = 15
    VOTE_DOWN = 125
    EDIT_OTHERS = 2000
    CLOSE_QUESTIONS = 3000
    DELETE_QUESTIONS = 10000


class Tag:
    def __init__(self, tag_id: int, name: str, description: str = ""):
        self.tag_id = tag_id
        self.name = name.lower()
        self.description = description
        self.question_count = 0

    def __str__(self) -> str:
        return f"[{self.name}]"

    def __hash__(self):
        return hash(self.tag_id)

    def __eq__(self, other):
        return isinstance(other, Tag) and self.tag_id == other.tag_id


class Vote:
    def __init__(self, user_id: int, vote_type: VoteType):
        self.user_id = user_id
        self.vote_type = vote_type
        self.timestamp = datetime.now()


class Comment:
    _comment_counter = 1

    def __init__(self, user_id: int, content: str):
        self.comment_id = Comment._comment_counter
        Comment._comment_counter += 1
        self.user_id = user_id
        self.content = content
        self.created_at = datetime.now()
        self.score = 0

    def __str__(self) -> str:
        return f"Comment #{self.comment_id}: {self.content[:50]}"


class Answer:
    _answer_counter = 1

    def __init__(self, author_id: int, question_id: int, content: str):
        self.answer_id = Answer._answer_counter
        Answer._answer_counter += 1
        self.author_id = author_id
        self.question_id = question_id
        self.content = content
        self.created_at = datetime.now()
        self.updated_at = datetime.now()

        self.votes: Dict[int, Vote] = {}  # user_id -> Vote
        self.comments: List[Comment] = []
        self.is_accepted = False
        self.is_deleted = False

    def vote(self, user_id: int, vote_type: VoteType):
        """Add or update vote"""
        self.votes[user_id] = Vote(user_id, vote_type)

    def remove_vote(self, user_id: int):
        """Remove vote"""
        self.votes.pop(user_id, None)

    def get_score(self) -> int:
        """Calculate score from votes"""
        return sum(v.vote_type.value for v in self.votes.values())

    def add_comment(self, comment: Comment):
        self.comments.append(comment)

    def accept(self):
        """Mark as accepted answer"""
        self.is_accepted = True

    def __str__(self) -> str:
        return f"Answer #{self.answer_id} (Score: {self.get_score()})"


class Question:
    _question_counter = 1

    def __init__(self, author_id: int, title: str, content: str, tags: List[Tag]):
        self.question_id = Question._question_counter
        Question._question_counter += 1
        self.author_id = author_id
        self.title = title
        self.content = content
        self.tags = tags
        self.created_at = datetime.now()
        self.updated_at = datetime.now()

        self.status = QuestionStatus.OPEN
        self.votes: Dict[int, Vote] = {}  # user_id -> Vote
        self.answers: List[Answer] = []
        self.comments: List[Comment] = []
        self.views = 0

        # Bounty system
        self.bounty: Optional[int] = None
        self.bounty_expires_at: Optional[datetime] = None

    def vote(self, user_id: int, vote_type: VoteType):
        """Add or update vote"""
        self.votes[user_id] = Vote(user_id, vote_type)

    def remove_vote(self, user_id: int):
        """Remove vote"""
        self.votes.pop(user_id, None)

    def get_score(self) -> int:
        """Calculate score from votes"""
        return sum(v.vote_type.value for v in self.votes.values())

    def add_answer(self, answer: Answer):
        self.answers.append(answer)

    def add_comment(self, comment: Comment):
        self.comments.append(comment)

    def get_accepted_answer(self) -> Optional[Answer]:
        accepted = [a for a in self.answers if a.is_accepted]
        return accepted[0] if accepted else None

    def set_bounty(self, amount: int, days: int = 7):
        """Set bounty for the question"""
        self.bounty = amount
        self.bounty_expires_at = datetime.now() + timedelta(days=days)

    def close(self):
        self.status = QuestionStatus.CLOSED

    def reopen(self):
        self.status = QuestionStatus.OPEN

    def increment_views(self):
        self.views += 1

    def __str__(self) -> str:
        tags_str = " ".join(str(tag) for tag in self.tags)
        return f"Q #{self.question_id}: {self.title} {tags_str} (Score: {self.get_score()})"


class User:
    def __init__(self, user_id: int, name: str, email: str):
        self.user_id = user_id
        self.name = name
        self.email = email
        self.reputation = 1  # Starting reputation
        self.created_at = datetime.now()

        self.questions_asked: List[int] = []
        self.answers_posted: List[int] = []

    def has_privilege(self, privilege: Privilege) -> bool:
        """Check if user has enough reputation for a privilege"""
        return self.reputation >= privilege.value

    def add_reputation(self, points: int):
        self.reputation += points
        self.reputation = max(1, self.reputation)  # Minimum 1

    def __str__(self) -> str:
        return f"{self.name} (Rep: {self.reputation})"


class ReputationSystem:
    """
    Manages reputation points for different actions.
    Based on Stack Overflow's reputation system.
    """
    QUESTION_UPVOTE = 5
    QUESTION_DOWNVOTE = -2
    ANSWER_UPVOTE = 10
    ANSWER_DOWNVOTE = -2
    ANSWER_ACCEPTED = 15
    ACCEPT_ANSWER = 2
    DOWNVOTE_COST = -1  # Cost to downvoter

    @staticmethod
    def award_question_upvote(user: User):
        user.add_reputation(ReputationSystem.QUESTION_UPVOTE)

    @staticmethod
    def award_question_downvote(user: User):
        user.add_reputation(ReputationSystem.QUESTION_DOWNVOTE)

    @staticmethod
    def award_answer_upvote(user: User):
        user.add_reputation(ReputationSystem.ANSWER_UPVOTE)

    @staticmethod
    def award_answer_downvote(user: User):
        user.add_reputation(ReputationSystem.ANSWER_DOWNVOTE)

    @staticmethod
    def award_answer_accepted(user: User):
        user.add_reputation(ReputationSystem.ANSWER_ACCEPTED)

    @staticmethod
    def award_accept_answer(user: User):
        user.add_reputation(ReputationSystem.ACCEPT_ANSWER)


class StackOverflow:
    def __init__(self):
        self.users: Dict[int, User] = {}
        self.questions: Dict[int, Question] = {}
        self.answers: Dict[int, Answer] = {}
        self.tags: Dict[str, Tag] = {}
        self.reputation_system = ReputationSystem()

    def add_user(self, user: User):
        self.users[user.user_id] = user

    def create_tag(self, tag_id: int, name: str, description: str = "") -> Tag:
        tag = Tag(tag_id, name, description)
        self.tags[name.lower()] = tag
        return tag

    def get_or_create_tag(self, name: str) -> Tag:
        """Get existing tag or create new one"""
        name_lower = name.lower()
        if name_lower not in self.tags:
            tag_id = len(self.tags) + 1
            return self.create_tag(tag_id, name)
        return self.tags[name_lower]

    def ask_question(self, author_id: int, title: str, content: str, tag_names: List[str]) -> Optional[Question]:
        if author_id not in self.users:
            return None

        # Get or create tags
        tags = [self.get_or_create_tag(name) for name in tag_names]

        question = Question(author_id, title, content, tags)
        self.questions[question.question_id] = question
        self.users[author_id].questions_asked.append(question.question_id)

        # Update tag counts
        for tag in tags:
            tag.question_count += 1

        print(f"✓ Question #{question.question_id} created by {self.users[author_id].name}")
        return question

    def post_answer(self, author_id: int, question_id: int, content: str) -> Optional[Answer]:
        user = self.users.get(author_id)
        question = self.questions.get(question_id)

        if not user or not question or question.status != QuestionStatus.OPEN:
            return None

        answer = Answer(author_id, question_id, content)
        self.answers[answer.answer_id] = answer
        question.add_answer(answer)
        user.answers_posted.append(answer.answer_id)

        print(f"✓ Answer #{answer.answer_id} posted by {user.name}")
        return answer

    def vote_question(self, question_id: int, voter_id: int, vote_type: VoteType) -> bool:
        question = self.questions.get(question_id)
        voter = self.users.get(voter_id)

        if not question or not voter:
            return False

        # Check privileges
        if vote_type == VoteType.UPVOTE and not voter.has_privilege(Privilege.VOTE_UP):
            print(f"User needs {Privilege.VOTE_UP.value} reputation to upvote")
            return False

        if vote_type == VoteType.DOWNVOTE and not voter.has_privilege(Privilege.VOTE_DOWN):
            print(f"User needs {Privilege.VOTE_DOWN.value} reputation to downvote")
            return False

        # Apply vote
        question.vote(voter_id, vote_type)

        # Award reputation to question author
        author = self.users[question.author_id]
        if vote_type == VoteType.UPVOTE:
            self.reputation_system.award_question_upvote(author)
        else:
            self.reputation_system.award_question_downvote(author)
            # Cost to downvoter
            voter.add_reputation(ReputationSystem.DOWNVOTE_COST)

        return True

    def vote_answer(self, answer_id: int, voter_id: int, vote_type: VoteType) -> bool:
        answer = self.answers.get(answer_id)
        voter = self.users.get(voter_id)

        if not answer or not voter:
            return False

        # Check privileges
        if vote_type == VoteType.UPVOTE and not voter.has_privilege(Privilege.VOTE_UP):
            print(f"User needs {Privilege.VOTE_UP.value} reputation to upvote")
            return False

        if vote_type == VoteType.DOWNVOTE and not voter.has_privilege(Privilege.VOTE_DOWN):
            print(f"User needs {Privilege.VOTE_DOWN.value} reputation to downvote")
            return False

        # Apply vote
        answer.vote(voter_id, vote_type)

        # Award reputation to answer author
        author = self.users[answer.author_id]
        if vote_type == VoteType.UPVOTE:
            self.reputation_system.award_answer_upvote(author)
        else:
            self.reputation_system.award_answer_downvote(author)
            # Cost to downvoter
            voter.add_reputation(ReputationSystem.DOWNVOTE_COST)

        return True

    def accept_answer(self, question_id: int, answer_id: int, accepter_id: int) -> bool:
        question = self.questions.get(question_id)
        answer = self.answers.get(answer_id)

        if not question or not answer:
            return False

        # Only question author can accept
        if question.author_id != accepter_id:
            print("Only question author can accept an answer")
            return False

        # Unaccept previous answer if exists
        previous_accepted = question.get_accepted_answer()
        if previous_accepted:
            previous_accepted.is_accepted = False
            # Remove reputation from previously accepted answer author
            prev_author = self.users[previous_accepted.author_id]
            prev_author.add_reputation(-ReputationSystem.ANSWER_ACCEPTED)

        # Accept new answer
        answer.accept()

        # Award reputation
        answer_author = self.users[answer.author_id]
        self.reputation_system.award_answer_accepted(answer_author)

        question_author = self.users[question.author_id]
        self.reputation_system.award_accept_answer(question_author)

        print(f"✓ Answer #{answer_id} accepted")
        return True

    def add_comment(self, user_id: int, question_id: int = None, answer_id: int = None,
                   content: str = "") -> Optional[Comment]:
        user = self.users.get(user_id)
        if not user or not user.has_privilege(Privilege.COMMENT):
            print(f"User needs {Privilege.COMMENT.value} reputation to comment")
            return None

        comment = Comment(user_id, content)

        if question_id:
            question = self.questions.get(question_id)
            if question:
                question.add_comment(comment)
        elif answer_id:
            answer = self.answers.get(answer_id)
            if answer:
                answer.add_comment(comment)

        return comment

    def search_by_tags(self, tag_names: List[str]) -> List[Question]:
        """Search questions by tags"""
        tag_names_lower = [name.lower() for name in tag_names]
        results = []

        for question in self.questions.values():
            if question.status != QuestionStatus.DELETED:
                question_tags = [tag.name for tag in question.tags]
                if any(tag in question_tags for tag in tag_names_lower):
                    results.append(question)

        return sorted(results, key=lambda q: q.created_at, reverse=True)

    def search_questions(self, query: str) -> List[Question]:
        """Search questions by title or content"""
        query_lower = query.lower()
        results = []

        for question in self.questions.values():
            if question.status != QuestionStatus.DELETED:
                if (query_lower in question.title.lower() or
                    query_lower in question.content.lower()):
                    results.append(question)

        return sorted(results, key=lambda q: q.get_score(), reverse=True)

    def get_top_questions(self, limit: int = 10) -> List[Question]:
        """Get top questions by score"""
        active_questions = [q for q in self.questions.values()
                          if q.status == QuestionStatus.OPEN]
        return sorted(active_questions, key=lambda q: q.get_score(), reverse=True)[:limit]

    def get_unanswered_questions(self) -> List[Question]:
        """Get questions with no answers"""
        return [q for q in self.questions.values()
                if q.status == QuestionStatus.OPEN and len(q.answers) == 0]


def main():
    so = StackOverflow()

    # Create users
    alice = User(1, "Alice", "alice@email.com")
    bob = User(2, "Bob", "bob@email.com")
    charlie = User(3, "Charlie", "charlie@email.com")

    # Give them some initial reputation for testing
    alice.reputation = 200
    bob.reputation = 150
    charlie.reputation = 100

    for user in [alice, bob, charlie]:
        so.add_user(user)

    print("=== Stack Overflow Demo ===\n")

    # Ask questions
    q1 = so.ask_question(alice.user_id, "How to reverse a list in Python?",
                        "I want to reverse a Python list. What's the best way?",
                        ["python", "list"])

    q2 = so.ask_question(bob.user_id, "What is the time complexity of dict lookup?",
                        "Is Python dict lookup O(1) or O(n)?",
                        ["python", "data-structures"])

    print("\n=== Posting Answers ===")
    # Post answers
    a1 = so.post_answer(bob.user_id, q1.question_id, "Use list.reverse() or reversed()")
    a2 = so.post_answer(charlie.user_id, q1.question_id, "You can also use slicing: my_list[::-1]")

    print("\n=== Voting ===")
    # Vote on question
    so.vote_question(q1.question_id, bob.user_id, VoteType.UPVOTE)
    so.vote_question(q1.question_id, charlie.user_id, VoteType.UPVOTE)
    print(f"Question score: {q1.get_score()}")
    print(f"Alice's reputation: {alice.reputation}")

    # Vote on answers
    so.vote_answer(a1.answer_id, alice.user_id, VoteType.UPVOTE)
    so.vote_answer(a1.answer_id, charlie.user_id, VoteType.UPVOTE)
    so.vote_answer(a2.answer_id, alice.user_id, VoteType.UPVOTE)

    print(f"\nAnswer 1 score: {a1.get_score()}")
    print(f"Answer 2 score: {a2.get_score()}")
    print(f"Bob's reputation: {bob.reputation}")
    print(f"Charlie's reputation: {charlie.reputation}")

    print("\n=== Accept Answer ===")
    # Accept answer
    so.accept_answer(q1.question_id, a2.answer_id, alice.user_id)
    print(f"Charlie's reputation after acceptance: {charlie.reputation}")
    print(f"Alice's reputation after accepting: {alice.reputation}")

    print("\n=== Comments ===")
    # Add comments
    comment = so.add_comment(bob.user_id, question_id=q1.question_id,
                           content="Great question!")
    if comment:
        print(f"✓ Comment added: {comment.content}")

    print("\n=== Search by Tags ===")
    python_questions = so.search_by_tags(["python"])
    print(f"Questions tagged 'python': {len(python_questions)}")
    for q in python_questions:
        print(f"  - {q.title}")

    print("\n=== Top Questions ===")
    top = so.get_top_questions(limit=3)
    for i, q in enumerate(top, 1):
        author = so.users[q.author_id]
        print(f"{i}. {q.title} by {author.name} (Score: {q.get_score()}, Answers: {len(q.answers)})")

    print("\n=== User Stats ===")
    print(f"Alice: {alice.reputation} rep, {len(alice.questions_asked)} questions, "
          f"{len(alice.answers_posted)} answers")
    print(f"Bob: {bob.reputation} rep, {len(bob.questions_asked)} questions, "
          f"{len(bob.answers_posted)} answers")
    print(f"Charlie: {charlie.reputation} rep, {len(charlie.questions_asked)} questions, "
          f"{len(charlie.answers_posted)} answers")


if __name__ == "__main__":
    main()
```

---

## Design Patterns

### 1. Strategy Pattern
- **Usage:** Different reputation calculation strategies
- **Benefit:** Easy to modify reputation rules

### 2. Observer Pattern (Extension)
- **Usage:** Notify users of new answers, comments, badges
- **Implementation:** Event listeners for content updates

### 3. Factory Pattern
- **Usage:** Create different content types (Question, Answer, Comment)
- **Benefit:** Extensible content creation

### 4. Composite Pattern
- **Usage:** Comments can be nested (replies to comments)
- **Benefit:** Tree structure for discussions

---

## SOLID Principles

### Single Responsibility Principle (SRP)
- `Question`: Manages question data and metadata
- `Answer`: Handles answer functionality
- `ReputationSystem`: Encapsulates reputation logic
- `StackOverflow`: Platform coordination

### Open/Closed Principle (OCP)
- New reputation rules can be added without modifying existing code
- New privilege levels can be introduced easily

### Liskov Substitution Principle (LSP)
- Different user types (Moderator, Admin) can extend `User` class

### Interface Segregation Principle (ISP)
- Separate interfaces for voting, commenting, and moderation

### Dependency Inversion Principle (DIP)
- Platform depends on abstractions, not concrete implementations

---

## Key Features

### Reputation System
- Question upvote: +5 to author
- Answer upvote: +10 to author
- Accepted answer: +15 to answerer, +2 to accepter
- Downvote: -2 to receiver, -1 to downvoter

### Privilege System
- Comment: 50 reputation
- Upvote: 15 reputation
- Downvote: 125 reputation
- Edit others' posts: 2000 reputation
- Close questions: 3000 reputation

---

## Extensions

### 1. Badge System
- Bronze/Silver/Gold badges
- Achievement tracking (first question, helpful answer, etc.)

### 2. Advanced Search
- Full-text search with Elasticsearch
- Search operators (tag:python score:>5)
- Autocomplete for tags

### 3. Moderation
- Flag system for inappropriate content
- Review queues
- Moderator tools (lock, delete, migrate)

### 4. Edit History
- Version control for questions/answers
- Rollback capability
- Edit suggestions from low-rep users

### 5. Analytics
- Question view tracking
- User activity stats
- Tag popularity trends

---

## Interview Tips

### Clarifying Questions
1. **Scale**: How many questions/answers per day?
2. **Features**: Which features are critical (voting, comments, bounties)?
3. **Reputation**: Should reputation be real-time or eventually consistent?
4. **Search**: Simple keyword search or advanced full-text search?
5. **Moderation**: How important is content moderation?

### Common Follow-ups

1. **Scalability**: "How would you scale to millions of questions?"
   - Separate services: Question Service, Answer Service, Vote Service
   - Database sharding by question ID
   - Caching (Redis) for hot questions
   - Search index (Elasticsearch)

2. **Performance**: "How to handle vote counts efficiently?"
   - Denormalize vote counts in question/answer tables
   - Update asynchronously via queue
   - Cache popular questions

3. **Consistency**: "Handle concurrent votes on same question?"
   - Optimistic locking
   - Atomic increment operations
   - Event sourcing for vote history

4. **Search**: "How to search millions of questions quickly?"
   - Elasticsearch with inverted index
   - Tag-based indexing
   - Scoring algorithm (relevance + recency + votes)

---

## Time Complexity

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| Ask Question | O(T) | T = number of tags |
| Post Answer | O(1) | Append to list |
| Vote | O(1) | Update vote map |
| Accept Answer | O(A) | A = number of answers (to find previous accepted) |
| Search by Tags | O(Q * T) | Q = questions, T = tags per question |
| Search Questions | O(Q) | Linear scan (use search index in production) |
| Top Questions | O(Q log K) | Q = questions, K = limit |

---

## System Design Considerations

When scaling to production:

1. **Database**:
   - Questions/Answers: PostgreSQL with partitioning
   - Votes: Separate table with composite index (user_id, question_id)
   - Tags: Many-to-many relationship table
   - Cache: Redis for hot questions, user sessions

2. **Architecture**:
   - Content Service: CRUD for questions/answers
   - Vote Service: Handle voting, reputation updates
   - Search Service: Elasticsearch for full-text search
   - Notification Service: Email/push notifications

3. **Optimization**:
   - CDN for static assets
   - Message queue (Kafka) for async vote processing
   - Read replicas for search queries
   - Denormalized view counts, vote counts

4. **Ranking Algorithms**:
   - Hot questions: Wilson score interval
   - Trending: Recent activity + vote velocity
   - Personalized: User's tag preferences

---

## Database Schema (Conceptual)

```
Users: user_id, name, email, reputation, created_at
Questions: question_id, author_id, title, content, status, created_at, view_count
Answers: answer_id, question_id, author_id, content, is_accepted, created_at
Votes: vote_id, user_id, votable_id, votable_type, vote_type, created_at
Tags: tag_id, name, description, question_count
QuestionTags: question_id, tag_id
Comments: comment_id, user_id, commentable_id, commentable_type, content, created_at
```

---

This tests reputation systems, voting mechanisms, and community-driven content platforms.
