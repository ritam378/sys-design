# Design a Recommendation System

## Table of Contents
- [Problem Statement](#problem-statement)
- [Requirements](#requirements)
- [Recommendation Approaches](#recommendation-approaches)
- [High-Level Architecture](#high-level-architecture)
- [Collaborative Filtering](#collaborative-filtering)
- [Content-Based Filtering](#content-based-filtering)
- [Hybrid Approaches](#hybrid-approaches)
- [Cold Start Problem](#cold-start-problem)
- [Real-Time Recommendations](#real-time-recommendations)
- [Model Training Pipeline](#model-training-pipeline)
- [Scalability](#scalability)
- [A/B Testing](#ab-testing)
- [Implementation Examples](#implementation-examples)
- [Real-World Examples](#real-world-examples)
- [Interview Tips](#interview-tips)

---

## Problem Statement

Design a **recommendation system** that can:
- Suggest relevant items to users (products, movies, videos, etc.)
- Personalize recommendations based on user behavior
- Handle millions of users and items
- Provide real-time recommendations
- Support multiple recommendation strategies
- Continuously learn and improve

**Similar to**: Netflix recommendations, Amazon product recommendations, YouTube video suggestions, Spotify playlists

---

## Requirements

### Functional Requirements

1. **Generate Recommendations**:
   - Personalized recommendations per user
   - Similar items (if you like X, try Y)
   - Trending/popular items
   - Category-based recommendations

2. **User Interactions**:
   - Track views, clicks, purchases, ratings
   - Explicit feedback (ratings, likes)
   - Implicit feedback (watch time, scrolling)

3. **Content Management**:
   - Item metadata (title, category, tags, description)
   - User profiles (demographics, preferences)

### Non-Functional Requirements

1. **Scalability**: 100M+ users, 1M+ items
2. **Performance**: <100ms for recommendation retrieval
3. **Accuracy**: High precision and recall
4. **Freshness**: Incorporate recent interactions
5. **Diversity**: Avoid filter bubbles
6. **Explainability**: Why was this recommended?

---

## Recommendation Approaches

### 1. Collaborative Filtering
Find users with similar tastes and recommend what they liked.

```
User A likes: [Item 1, Item 2, Item 3]
User B likes: [Item 1, Item 2, Item 4]
→ Recommend Item 4 to User A (similar users)
```

**Types**:
- **User-based**: Find similar users
- **Item-based**: Find similar items
- **Matrix Factorization**: SVD, ALS

### 2. Content-Based Filtering
Recommend items similar to what user liked before.

```
User liked: Action movies with Tom Cruise
→ Recommend: Other action movies or Tom Cruise films
```

### 3. Hybrid Approaches
Combine collaborative and content-based.

```
Netflix: 80% collaborative + 20% content-based
```

---

## High-Level Architecture

```
┌────────────────────────────────────────────────────────────┐
│                         Users                               │
│                 (Web, Mobile, TV Apps)                     │
└────────────────┬───────────────────────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────────────────────┐
│                   API Gateway                               │
│              (Authentication, Rate Limiting)                │
└────────────────┬───────────────────────────────────────────┘
                 │
        ┌────────┴────────┐
        ↓                 ↓
┌──────────────┐   ┌──────────────────┐
│ Recommendation│   │  User Activity   │
│    Service    │   │    Tracker       │
└───────┬───────┘   └────────┬─────────┘
        │                    │
        │                    ↓
        │          ┌─────────────────┐
        │          │  Event Stream   │
        │          │    (Kafka)      │
        │          └────────┬────────┘
        │                   │
        ↓                   ↓
┌────────────────────────────────────────────────────────────┐
│              Recommendation Engine                          │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────┐│
│  │ Collaborative   │  │  Content-Based  │  │  Popular   ││
│  │   Filtering     │  │    Filtering    │  │   Items    ││
│  └────────┬────────┘  └────────┬────────┘  └─────┬──────┘│
└───────────┼────────────────────┼──────────────────┼───────┘
            │                    │                  │
            ↓                    ↓                  ↓
   ┌──────────────────────────────────────────────────┐
   │           Candidate Generation                    │
   │  (Generate 1000s of candidates per user)         │
   └──────────────────┬───────────────────────────────┘
                      │
                      ↓
   ┌──────────────────────────────────────────────────┐
   │              Ranking Model                        │
   │  (ML model to rank and score candidates)         │
   └──────────────────┬───────────────────────────────┘
                      │
                      ↓
   ┌──────────────────────────────────────────────────┐
   │           Re-ranking & Filtering                  │
   │  (Diversity, Business rules, A/B testing)        │
   └──────────────────┬───────────────────────────────┘
                      │
                      ↓
   ┌──────────────────────────────────────────────────┐
   │              Cache (Redis)                        │
   │  Store pre-computed recommendations              │
   └──────────────────────────────────────────────────┘
            │                    │
            ↓                    ↓
   ┌──────────────┐     ┌──────────────┐
   │  User-Item   │     │    Item      │
   │ Interaction  │     │   Metadata   │
   │  Database    │     │   Database   │
   └──────────────┘     └──────────────┘

   OFFLINE PROCESSING:
   ┌──────────────────────────────────────────────────┐
   │       Model Training Pipeline                     │
   │  (Spark/Airflow - Daily/Weekly batch jobs)      │
   │                                                   │
   │  1. Feature Engineering                          │
   │  2. Model Training (ALS, Neural CF)              │
   │  3. Model Evaluation                             │
   │  4. Deploy to Production                         │
   └──────────────────────────────────────────────────┘
```

---

## Collaborative Filtering

### User-Based Collaborative Filtering

```python
import numpy as np
from scipy.spatial.distance import cosine

class UserBasedCF:
    """
    User-based collaborative filtering

    Find similar users and recommend items they liked
    """

    def __init__(self):
        self.user_item_matrix = None  # users × items matrix
        self.user_similarity = None

    def fit(self, interactions):
        """
        Build user-item matrix from interactions

        interactions: [(user_id, item_id, rating), ...]
        """
        users = list(set([u for u, _, _ in interactions]))
        items = list(set([i for _, i, _ in interactions]))

        # Create user-item matrix
        n_users = len(users)
        n_items = len(items)
        self.user_item_matrix = np.zeros((n_users, n_items))

        user_idx = {u: i for i, u in enumerate(users)}
        item_idx = {i: j for j, i in enumerate(items)}

        for user, item, rating in interactions:
            self.user_item_matrix[user_idx[user], item_idx[item]] = rating

        # Calculate user similarity matrix
        self.user_similarity = self._calculate_similarity()

        self.user_idx = user_idx
        self.item_idx = item_idx
        self.items = items

    def _calculate_similarity(self):
        """
        Calculate cosine similarity between users
        """
        n_users = self.user_item_matrix.shape[0]
        similarity = np.zeros((n_users, n_users))

        for i in range(n_users):
            for j in range(i, n_users):
                if i == j:
                    similarity[i, j] = 1.0
                else:
                    # Cosine similarity
                    sim = 1 - cosine(
                        self.user_item_matrix[i],
                        self.user_item_matrix[j]
                    )
                    similarity[i, j] = sim
                    similarity[j, i] = sim

        return similarity

    def recommend(self, user_id, n=10):
        """
        Recommend top N items for user

        Algorithm:
        1. Find K most similar users
        2. Get items they rated highly
        3. Filter out items user already interacted with
        4. Return top N
        """
        if user_id not in self.user_idx:
            return self._recommend_popular(n)

        user_index = self.user_idx[user_id]

        # Find similar users (top 50)
        similar_users = np.argsort(self.user_similarity[user_index])[::-1][1:51]

        # Get items user hasn't interacted with
        user_items = self.user_item_matrix[user_index]
        unrated_items = np.where(user_items == 0)[0]

        # Calculate predicted ratings for unrated items
        predictions = []
        for item_idx in unrated_items:
            # Weighted average of similar users' ratings
            similar_ratings = self.user_item_matrix[similar_users, item_idx]
            similar_scores = self.user_similarity[user_index, similar_users]

            # Filter users who rated this item
            rated_mask = similar_ratings > 0
            if not rated_mask.any():
                continue

            predicted_rating = np.average(
                similar_ratings[rated_mask],
                weights=similar_scores[rated_mask]
            )

            predictions.append((self.items[item_idx], predicted_rating))

        # Sort by predicted rating
        predictions.sort(key=lambda x: x[1], reverse=True)

        return [item for item, _ in predictions[:n]]

# Example usage
interactions = [
    ('user1', 'item1', 5),
    ('user1', 'item2', 4),
    ('user2', 'item1', 5),
    ('user2', 'item3', 4),
    ('user3', 'item2', 3),
    ('user3', 'item3', 5),
]

cf = UserBasedCF()
cf.fit(interactions)
recommendations = cf.recommend('user1', n=5)
print(recommendations)
```

### Item-Based Collaborative Filtering

```python
class ItemBasedCF:
    """
    Item-based collaborative filtering

    Find similar items and recommend them
    More scalable than user-based (items change less frequently)
    """

    def __init__(self):
        self.item_similarity = None

    def fit(self, interactions):
        """Build item similarity matrix"""
        users = list(set([u for u, _, _ in interactions]))
        items = list(set([i for _, i, _ in interactions]))

        # Create user-item matrix (transpose of user-based)
        n_users = len(users)
        n_items = len(items)
        user_item_matrix = np.zeros((n_users, n_items))

        user_idx = {u: i for i, u in enumerate(users)}
        item_idx = {i: j for j, i in enumerate(items)}

        for user, item, rating in interactions:
            user_item_matrix[user_idx[user], item_idx[item]] = rating

        # Calculate item similarity (items × items)
        self.item_similarity = self._calculate_item_similarity(user_item_matrix)

        self.user_idx = user_idx
        self.item_idx = item_idx
        self.items = items
        self.user_item_matrix = user_item_matrix

    def _calculate_item_similarity(self, user_item_matrix):
        """Calculate cosine similarity between items"""
        n_items = user_item_matrix.shape[1]
        similarity = np.zeros((n_items, n_items))

        for i in range(n_items):
            for j in range(i, n_items):
                if i == j:
                    similarity[i, j] = 1.0
                else:
                    # Items rated by same users
                    item_i = user_item_matrix[:, i]
                    item_j = user_item_matrix[:, j]

                    # Only consider users who rated both items
                    mask = (item_i > 0) & (item_j > 0)
                    if mask.sum() == 0:
                        continue

                    sim = 1 - cosine(item_i[mask], item_j[mask])
                    similarity[i, j] = sim
                    similarity[j, i] = sim

        return similarity

    def recommend(self, user_id, n=10):
        """
        Recommend items similar to what user liked
        """
        if user_id not in self.user_idx:
            return []

        user_index = self.user_idx[user_id]
        user_ratings = self.user_item_matrix[user_index]

        # Get items user rated highly (>= 4)
        liked_items = np.where(user_ratings >= 4)[0]

        if len(liked_items) == 0:
            return []

        # Find similar items
        item_scores = {}
        for item_idx in liked_items:
            # Get similar items
            similar_items = self.item_similarity[item_idx]

            for other_idx, similarity in enumerate(similar_items):
                if user_ratings[other_idx] > 0:  # Already rated
                    continue

                if other_idx not in item_scores:
                    item_scores[other_idx] = 0

                # Weighted by user's rating and item similarity
                item_scores[other_idx] += user_ratings[item_idx] * similarity

        # Sort by score
        ranked_items = sorted(item_scores.items(), key=lambda x: x[1], reverse=True)

        return [self.items[idx] for idx, _ in ranked_items[:n]]
```

### Matrix Factorization (ALS)

```python
from pyspark.ml.recommendation import ALS
from pyspark.sql import SparkSession

class MatrixFactorization:
    """
    Matrix factorization using Alternating Least Squares (ALS)

    Decompose user-item matrix into:
    User matrix (users × latent_factors)
    Item matrix (items × latent_factors)

    Used by Netflix, Spotify
    """

    def __init__(self, rank=10, max_iter=10, reg_param=0.01):
        self.spark = SparkSession.builder.appName("Recommendations").getOrCreate()
        self.rank = rank  # Number of latent factors
        self.max_iter = max_iter
        self.reg_param = reg_param
        self.model = None

    def fit(self, interactions_df):
        """
        Train ALS model

        interactions_df: Spark DataFrame with columns [user_id, item_id, rating]
        """
        als = ALS(
            rank=self.rank,
            maxIter=self.max_iter,
            regParam=self.reg_param,
            userCol="user_id",
            itemCol="item_id",
            ratingCol="rating",
            coldStartStrategy="drop"
        )

        self.model = als.fit(interactions_df)

    def recommend_for_user(self, user_id, n=10):
        """Generate top N recommendations for user"""
        user_df = self.spark.createDataFrame([(user_id,)], ["user_id"])
        recommendations = self.model.recommendForUserSubset(user_df, n)

        return recommendations.collect()[0]['recommendations']

    def recommend_for_all_users(self, n=10):
        """
        Generate recommendations for all users (batch)

        Can be run offline and cached
        """
        recommendations = self.model.recommendForAllUsers(n)
        return recommendations

# Example usage
"""
from pyspark.sql import Row

interactions = [
    Row(user_id=1, item_id=1, rating=5.0),
    Row(user_id=1, item_id=2, rating=4.0),
    Row(user_id=2, item_id=1, rating=5.0),
    Row(user_id=2, item_id=3, rating=4.0),
]

spark = SparkSession.builder.getOrCreate()
df = spark.createDataFrame(interactions)

mf = MatrixFactorization(rank=10)
mf.fit(df)

# Get recommendations for user 1
recs = mf.recommend_for_user(1, n=5)
"""
```

---

## Content-Based Filtering

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity

class ContentBasedRecommender:
    """
    Content-based filtering using item features

    Recommend items similar to what user liked based on:
    - Category, tags, genre
    - Description text
    - Actors, director, etc.
    """

    def __init__(self):
        self.vectorizer = TfidfVectorizer(stop_words='english')
        self.item_features = None
        self.item_similarity = None

    def fit(self, items):
        """
        Build item feature vectors

        items: [
            {'id': 'item1', 'category': 'action', 'tags': ['thriller', 'crime'], 'description': '...'},
            ...
        ]
        """
        self.items = items
        self.item_ids = [item['id'] for item in items]

        # Combine text features
        item_texts = []
        for item in items:
            text = f"{item.get('category', '')} {' '.join(item.get('tags', []))} {item.get('description', '')}"
            item_texts.append(text)

        # TF-IDF vectorization
        self.item_features = self.vectorizer.fit_transform(item_texts)

        # Calculate item similarity matrix
        self.item_similarity = cosine_similarity(self.item_features)

    def recommend(self, user_liked_items, n=10):
        """
        Recommend items similar to what user liked

        user_liked_items: List of item IDs user interacted with
        """
        if not user_liked_items:
            return []

        # Get indices of liked items
        liked_indices = [
            self.item_ids.index(item_id)
            for item_id in user_liked_items
            if item_id in self.item_ids
        ]

        if not liked_indices:
            return []

        # Calculate similarity scores
        scores = self.item_similarity[liked_indices].mean(axis=0)

        # Get top N similar items (excluding already liked)
        top_indices = scores.argsort()[::-1]

        recommendations = []
        for idx in top_indices:
            item_id = self.item_ids[idx]
            if item_id not in user_liked_items:
                recommendations.append(item_id)
                if len(recommendations) >= n:
                    break

        return recommendations

# Example usage
items = [
    {
        'id': 'movie1',
        'category': 'action',
        'tags': ['thriller', 'crime'],
        'description': 'A thrilling action movie with intense car chases'
    },
    {
        'id': 'movie2',
        'category': 'action',
        'tags': ['adventure', 'sci-fi'],
        'description': 'Space adventure with amazing special effects'
    },
    {
        'id': 'movie3',
        'category': 'comedy',
        'tags': ['romance', 'feel-good'],
        'description': 'A heartwarming romantic comedy'
    },
]

cb = ContentBasedRecommender()
cb.fit(items)

user_liked = ['movie1']
recommendations = cb.recommend(user_liked, n=5)
print(recommendations)  # Will recommend movie2 (similar action genre)
```

---

## Hybrid Approaches

```python
class HybridRecommender:
    """
    Combine collaborative and content-based filtering

    Strategies:
    1. Weighted: Score = 0.7 * CF + 0.3 * CB
    2. Switching: Use CF if enough data, else CB
    3. Feature combination: Use CF scores as features in CB
    """

    def __init__(self, cf_model, cb_model, cf_weight=0.7):
        self.cf_model = cf_model
        self.cb_model = cb_model
        self.cf_weight = cf_weight
        self.cb_weight = 1 - cf_weight

    def recommend_weighted(self, user_id, user_liked_items, n=10):
        """
        Weighted hybrid approach
        """
        # Get CF recommendations with scores
        cf_recs = self.cf_model.recommend(user_id, n=50)

        # Get CB recommendations
        cb_recs = self.cb_model.recommend(user_liked_items, n=50)

        # Combine scores
        combined_scores = {}

        # CF scores (normalized)
        for i, item in enumerate(cf_recs):
            score = (len(cf_recs) - i) / len(cf_recs)  # Normalize by position
            combined_scores[item] = self.cf_weight * score

        # CB scores
        for i, item in enumerate(cb_recs):
            score = (len(cb_recs) - i) / len(cb_recs)
            if item in combined_scores:
                combined_scores[item] += self.cb_weight * score
            else:
                combined_scores[item] = self.cb_weight * score

        # Sort by combined score
        ranked = sorted(combined_scores.items(), key=lambda x: x[1], reverse=True)

        return [item for item, _ in ranked[:n]]

    def recommend_switching(self, user_id, user_interaction_count, user_liked_items, n=10):
        """
        Switching hybrid: Use CF if enough data, else CB
        """
        if user_interaction_count >= 10:
            # Enough data for collaborative filtering
            return self.cf_model.recommend(user_id, n=n)
        else:
            # Not enough data, use content-based
            return self.cb_model.recommend(user_liked_items, n=n)
```

---

## Cold Start Problem

### New User (No interaction history)

```python
class ColdStartHandler:
    """
    Handle cold start for new users and items
    """

    def recommend_for_new_user(self, user_demographics, n=10):
        """
        Recommend to new user based on:
        1. Demographics (age, gender, location)
        2. Onboarding preferences
        3. Popular items in user's demographic
        """
        # Get popular items for similar demographic
        similar_users = db.get_users_by_demographics(
            age_range=(user_demographics['age'] - 5, user_demographics['age'] + 5),
            location=user_demographics['location']
        )

        # Get items popular among similar users
        popular_items = db.get_popular_items_for_users(similar_users, limit=n)

        return popular_items

    def onboarding_flow(self):
        """
        Show new user a few items to get initial preferences
        """
        # Show diverse set of items from different categories
        categories = ['action', 'comedy', 'drama', 'sci-fi', 'horror']
        sample_items = []

        for category in categories:
            # Get 2 popular items from each category
            items = db.get_popular_items_by_category(category, limit=2)
            sample_items.extend(items)

        return sample_items

    def recommend_for_new_item(self, item_metadata, n=10):
        """
        Recommend new item to users based on:
        1. Content similarity to items user liked
        2. Users who like similar items
        """
        # Find similar existing items using metadata
        similar_items = self.content_based.find_similar(item_metadata, n=20)

        # Find users who liked these similar items
        target_users = db.get_users_who_liked(similar_items)

        return target_users
```

---

## Real-Time Recommendations

```python
import redis
from kafka import KafkaConsumer

class RealTimeRecommender:
    """
    Update recommendations in real-time based on user actions
    """

    def __init__(self):
        self.redis = redis.Redis()
        self.kafka_consumer = KafkaConsumer('user-events')

    def process_user_event(self, event):
        """
        Process real-time user event

        event = {
            'user_id': 'user123',
            'event_type': 'view',
            'item_id': 'item456',
            'timestamp': 1234567890
        }
        """
        user_id = event['user_id']
        item_id = event['item_id']

        # Update user's recent interactions
        self.redis.zadd(
            f"user:{user_id}:recent_views",
            {item_id: event['timestamp']}
        )

        # Trim to last 50 interactions
        self.redis.zremrangebyrank(
            f"user:{user_id}:recent_views",
            0, -51
        )

        # Get real-time recommendations based on this item
        similar_items = self.get_similar_items(item_id, n=10)

        # Cache updated recommendations
        self.redis.setex(
            f"user:{user_id}:realtime_recs",
            3600,  # 1 hour TTL
            json.dumps(similar_items)
        )

    def get_recommendations(self, user_id):
        """
        Get recommendations (real-time + batch)

        Strategy:
        1. Real-time recs (based on last few actions)
        2. Batch recs (pre-computed daily)
        3. Blend both
        """
        # Real-time recommendations
        realtime_recs = self.redis.get(f"user:{user_id}:realtime_recs")
        realtime_recs = json.loads(realtime_recs) if realtime_recs else []

        # Batch recommendations (pre-computed)
        batch_recs = self.redis.get(f"user:{user_id}:batch_recs")
        batch_recs = json.loads(batch_recs) if batch_recs else []

        # Blend: 30% real-time, 70% batch
        final_recs = []
        final_recs.extend(realtime_recs[:3])  # Top 3 real-time
        final_recs.extend(batch_recs[:7])  # Top 7 batch

        return final_recs

    def listen_to_events(self):
        """Consume user events from Kafka"""
        for message in self.kafka_consumer:
            event = json.loads(message.value)
            self.process_user_event(event)
```

---

## Model Training Pipeline

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

class RecommenderTrainingPipeline:
    """
    Offline batch training pipeline

    Runs daily/weekly to update recommendation models
    """

    def extract_features(self):
        """
        Extract features from user interactions

        Features:
        - User demographics
        - Item metadata
        - Interaction history
        - Temporal features (time of day, day of week)
        """
        interactions = db.get_interactions(
            start_date=datetime.now() - timedelta(days=90)
        )

        features = []
        for interaction in interactions:
            feature = {
                'user_id': interaction['user_id'],
                'item_id': interaction['item_id'],
                'rating': interaction['rating'],
                'timestamp': interaction['timestamp'],
                'user_age': user_profiles[interaction['user_id']]['age'],
                'item_category': items[interaction['item_id']]['category'],
                # ... more features
            }
            features.append(feature)

        return features

    def train_model(self, features):
        """
        Train recommendation model
        """
        # Convert to Spark DataFrame
        df = spark.createDataFrame(features)

        # Train ALS model
        als = ALS(rank=50, maxIter=20, regParam=0.01)
        model = als.fit(df)

        return model

    def evaluate_model(self, model, test_data):
        """
        Evaluate model performance

        Metrics:
        - Precision@K
        - Recall@K
        - NDCG
        - MAP (Mean Average Precision)
        """
        predictions = model.transform(test_data)

        # Calculate metrics
        from pyspark.ml.evaluation import RegressionEvaluator

        evaluator = RegressionEvaluator(
            metricName="rmse",
            labelCol="rating",
            predictionCol="prediction"
        )

        rmse = evaluator.evaluate(predictions)

        return {'rmse': rmse}

    def deploy_model(self, model):
        """
        Deploy model to production

        1. Save model to S3
        2. Update model version in database
        3. Trigger re-computation of recommendations
        """
        model.save("s3://models/als_model_v2")

        # Update model metadata
        db.update_model_version('als', 'v2', datetime.now())

        # Trigger batch recommendation generation
        self.generate_batch_recommendations(model)

    def generate_batch_recommendations(self, model):
        """
        Generate recommendations for all users (offline)
        Store in Redis for fast serving
        """
        # Generate for all users
        all_user_recs = model.recommendForAllUsers(10)

        # Store in Redis
        for row in all_user_recs.collect():
            user_id = row['user_id']
            recommendations = [rec['item_id'] for rec in row['recommendations']]

            self.redis.setex(
                f"user:{user_id}:batch_recs",
                86400 * 7,  # 7 days TTL
                json.dumps(recommendations)
            )

# Airflow DAG
default_args = {
    'owner': 'data-science',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'retries': 1,
}

dag = DAG(
    'recommendation_training',
    default_args=default_args,
    schedule_interval='@daily',
)

extract_task = PythonOperator(
    task_id='extract_features',
    python_callable=pipeline.extract_features,
    dag=dag,
)

train_task = PythonOperator(
    task_id='train_model',
    python_callable=pipeline.train_model,
    dag=dag,
)

evaluate_task = PythonOperator(
    task_id='evaluate_model',
    python_callable=pipeline.evaluate_model,
    dag=dag,
)

deploy_task = PythonOperator(
    task_id='deploy_model',
    python_callable=pipeline.deploy_model,
    dag=dag,
)

extract_task >> train_task >> evaluate_task >> deploy_task
```

---

## Scalability

### Distributed Computing with Spark

```python
from pyspark.sql import SparkSession
from pyspark.ml.recommendation import ALS

# Initialize Spark
spark = SparkSession.builder \
    .appName("Recommendations") \
    .config("spark.driver.memory", "4g") \
    .config("spark.executor.memory", "8g") \
    .config("spark.executor.cores", "4") \
    .getOrCreate()

# Load data (can be billions of rows)
interactions = spark.read.parquet("s3://data/interactions/")

# Train model (distributed across cluster)
als = ALS(rank=100, maxIter=20)
model = als.fit(interactions)

# Generate recommendations (distributed)
user_recs = model.recommendForAllUsers(10)

# Save results
user_recs.write.parquet("s3://recommendations/batch/")
```

### Caching Strategy

```python
class RecommendationCache:
    """
    Multi-tier caching for recommendations
    """

    def __init__(self):
        self.redis = redis.Redis()
        self.local_cache = {}  # In-memory LRU cache

    def get_recommendations(self, user_id, n=10):
        """
        Get recommendations with caching

        L1: Local memory (fastest)
        L2: Redis (fast)
        L3: Database/Model (slow)
        """
        # L1: Local cache
        if user_id in self.local_cache:
            return self.local_cache[user_id][:n]

        # L2: Redis cache
        cached = self.redis.get(f"user:{user_id}:recs")
        if cached:
            recs = json.loads(cached)
            self.local_cache[user_id] = recs  # Populate L1
            return recs[:n]

        # L3: Generate from model (cache miss)
        recs = self.model.recommend(user_id, n=n)

        # Populate caches
        self.redis.setex(f"user:{user_id}:recs", 3600, json.dumps(recs))
        self.local_cache[user_id] = recs

        return recs
```

---

## A/B Testing

```python
class ABTestingFramework:
    """
    A/B test different recommendation algorithms
    """

    def __init__(self):
        self.experiments = {}

    def create_experiment(self, experiment_id, control_model, treatment_model, traffic_split=0.5):
        """
        Create A/B test

        control_model: Current model (baseline)
        treatment_model: New model to test
        traffic_split: % of traffic to treatment (0-1)
        """
        self.experiments[experiment_id] = {
            'control': control_model,
            'treatment': treatment_model,
            'split': traffic_split,
            'metrics': {
                'control': {'views': 0, 'clicks': 0, 'purchases': 0},
                'treatment': {'views': 0, 'clicks': 0, 'purchases': 0}
            }
        }

    def get_variant(self, user_id, experiment_id):
        """
        Assign user to variant (control or treatment)

        Use consistent hashing for stable assignment
        """
        experiment = self.experiments[experiment_id]

        # Hash user_id to get consistent variant
        hash_val = hash(f"{user_id}:{experiment_id}") % 100

        if hash_val < experiment['split'] * 100:
            return 'treatment'
        else:
            return 'control'

    def get_recommendations(self, user_id, experiment_id, n=10):
        """Get recommendations based on A/B test variant"""
        variant = self.get_variant(user_id, experiment_id)
        experiment = self.experiments[experiment_id]

        if variant == 'treatment':
            recs = experiment['treatment'].recommend(user_id, n=n)
        else:
            recs = experiment['control'].recommend(user_id, n=n)

        # Track that user saw recommendations
        experiment['metrics'][variant]['views'] += 1

        return recs, variant

    def track_click(self, experiment_id, variant, item_id):
        """Track when user clicks on recommendation"""
        self.experiments[experiment_id]['metrics'][variant]['clicks'] += 1

    def analyze_results(self, experiment_id):
        """
        Analyze A/B test results

        Calculate:
        - Click-through rate (CTR)
        - Conversion rate
        - Statistical significance (t-test)
        """
        experiment = self.experiments[experiment_id]
        metrics = experiment['metrics']

        # CTR
        control_ctr = metrics['control']['clicks'] / max(metrics['control']['views'], 1)
        treatment_ctr = metrics['treatment']['clicks'] / max(metrics['treatment']['views'], 1)

        # Lift
        lift = (treatment_ctr - control_ctr) / control_ctr if control_ctr > 0 else 0

        return {
            'control_ctr': control_ctr,
            'treatment_ctr': treatment_ctr,
            'lift': lift,
            'winner': 'treatment' if treatment_ctr > control_ctr else 'control'
        }
```

---

## Real-World Examples

### Netflix
- **Algorithms**: 80% collaborative filtering + 20% content-based
- **Scale**: 200M+ users, billions of ratings
- **Features**: Watch history, ratings, time of day, device
- **Metrics**: Retention, watch time

### Amazon
- **Approaches**: Item-to-item collaborative filtering
- **Features**: "Customers who bought X also bought Y"
- **Personalization**: Browse history, cart, wish list
- **Business Impact**: 35% of revenue from recommendations

### YouTube
- **Two-stage**: Candidate generation → Ranking
- **Features**: Watch history, search history, demographics
- **Real-time**: Immediate updates based on current session
- **Scale**: 1B+ users, 500+ hours uploaded/minute

### Spotify
- **Hybrid**: Collaborative filtering + Audio analysis
- **Playlists**: Discover Weekly, Release Radar
- **Features**: Listening history, skips, playlists
- **Innovation**: Audio features (tempo, energy, mood)

---

## Interview Tips

### Common Questions

**Q: Collaborative vs Content-based filtering?**
- **Collaborative**: Uses user behavior (ratings, clicks). Good for discovering new types. Suffers from cold start.
- **Content-based**: Uses item features. Works for new items. May create filter bubbles.
- **Hybrid**: Combine both for best results.

**Q: How to handle cold start problem?**
- **New users**: Popular items, demographic-based, onboarding flow
- **New items**: Content-based similarity, promoted items
- **Long-tail items**: Explore-exploit trade-off

**Q: How to evaluate recommendation quality?**
- **Offline**: Precision@K, Recall@K, NDCG, MAP
- **Online**: CTR, conversion rate, engagement time
- **Business**: Revenue, retention, user satisfaction

**Q: How to scale to billions of users?**
- **Batch processing**: Spark/Hadoop for model training
- **Caching**: Redis for pre-computed recommendations
- **Approximate nearest neighbors**: For real-time similarity
- **Sampling**: Train on subset, apply to all

**Q: How to ensure diversity?**
- **Re-ranking**: Penalize similar items
- **Exploration**: Random items (10-20%)
- **Categories**: Ensure multiple categories in recommendations
- **Freshness**: Include recent items

### Key Takeaways

1. **Two-stage pipeline**: Candidate generation → Ranking
2. **Hybrid approaches**: Combine collaborative + content-based
3. **Scalability**: Batch pre-computation + real-time updates
4. **A/B testing**: Continuous experimentation
5. **Cold start**: Demographics, onboarding, popular items
6. **Metrics**: Both offline (precision, recall) and online (CTR, revenue)

This recommendation system design demonstrates ML, distributed computing, and real-time processing - critical for modern tech companies!
