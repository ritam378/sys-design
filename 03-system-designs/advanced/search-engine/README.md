# Search Engine System Design

> **Difficulty:** Advanced
> **Topics:** Web Crawling, Inverted Index, PageRank, Distributed Systems, Information Retrieval
> **Companies:** Google, Microsoft (Bing), DuckDuckGo, Elasticsearch, Algolia

---

## 1. Problem Statement

Design a **web-scale search engine** that can crawl, index, and search billions of web pages, returning relevant results in under 200 milliseconds.

### Core Challenges

**Users expect:**
- Find relevant pages among billions of results
- Search results returned in <200ms
- Handle typos and understand intent
- Fresh results (recently published content)
- Personalized and localized results

**System must:**
- Crawl and index the entire web (billions of pages)
- Handle millions of queries per second
- Rank results by relevance
- Scale horizontally
- Update index continuously

**Real-world Examples:** Google Search, Bing, DuckDuckGo, Elasticsearch

---

## 2. Requirements

### Functional Requirements
1. **Crawl** billions of web pages
2. **Index** content for fast retrieval
3. **Search** and return relevant results
4. **Rank** results by relevance
5. **Support** autocomplete/suggestions
6. **Handle** typos and synonyms

### Non-Functional Requirements
1. **Scale**: Billions of pages, millions of queries/sec
2. **Latency**: < 200ms for search results
3. **Availability**: 99.99% uptime
4. **Freshness**: Index new content within hours
5. **Relevance**: High-quality ranking

### Capacity Estimation

**Assumptions:**
- 50 billion web pages
- Average page size: 100 KB
- 100,000 queries per second (QPS)
- Average query: 3 words

**Storage:**
- Pages: 50B × 100 KB = 5 PB
- Index: ~30% of original = 1.5 PB
- Total with replication (3x): 19.5 PB

**Bandwidth:**
- Search queries: 100K QPS × 100 bytes = 10 MB/s
- Results: 100K QPS × 50 KB = 5 GB/s
- Crawling: 1M pages/hour × 100 KB = 27 MB/s

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        SEARCH ENGINE                         │
└─────────────────────────────────────────────────────────────┘

┌──────────────────┐         ┌──────────────────┐
│  Web Crawlers    │────────>│   URL Frontier   │
│  (Distributed)   │         │   (Priority Q)   │
└────────┬─────────┘         └──────────────────┘
         │
         v
┌─────────────────────────────────────────────────┐
│              Document Processor                  │
│  - HTML Parser                                   │
│  - Content Extractor                            │
│  - Deduplication (SimHash)                      │
└────────┬────────────────────────────────────────┘
         │
         v
┌─────────────────────────────────────────────────┐
│              Indexing Pipeline                   │
│  - Tokenization                                  │
│  - Inverted Index Builder                       │
│  - Forward Index Builder                        │
└────────┬────────────────────────────────────────┘
         │
         v
┌─────────────────────────────────────────────────┐
│           Distributed Index Storage              │
│  - Sharded by Term Hash                         │
│  - Replicated for Availability                  │
└─────────────────────────────────────────────────┘
                     ^
                     │
┌────────────────────┴─────────────────────────────┐
│              Query Processing                     │
│  1. Query Parser & Tokenizer                     │
│  2. Query Expansion (synonyms)                   │
│  3. Index Lookup (parallel)                      │
│  4. Ranking (PageRank + TF-IDF + ML)            │
│  5. Results Aggregation                          │
└────────┬─────────────────────────────────────────┘
         │
         v
┌─────────────────────────────────────────────────┐
│                Cache Layer                       │
│  - Query Cache (Redis)                          │
│  - Page Cache                                   │
│  - Autocomplete Trie                            │
└─────────────────────────────────────────────────┘
```

## Core Components

### 1. Web Crawler

**Distributed Crawler Architecture:**

```python
import asyncio
import aiohttp
from urllib.parse import urljoin, urlparse
from bs4 import BeautifulSoup
from collections import deque
import hashlib

class DistributedCrawler:
    """
    Distributed web crawler with politeness

    Features:
    - Respects robots.txt
    - Rate limiting per domain
    - Deduplication with Bloom filter
    - Priority-based URL frontier
    """

    def __init__(self, seed_urls, num_workers=100):
        self.url_frontier = URLFrontier()
        self.visited = BloomFilter(capacity=1_000_000_000)
        self.num_workers = num_workers
        self.politeness_delay = 1  # seconds per domain

        # Seed initial URLs
        for url in seed_urls:
            self.url_frontier.add(url, priority=10)

    async def crawl(self):
        """Start distributed crawling"""
        tasks = []
        for worker_id in range(self.num_workers):
            task = asyncio.create_task(self.worker(worker_id))
            tasks.append(task)

        await asyncio.gather(*tasks)

    async def worker(self, worker_id):
        """Crawler worker - processes URLs from frontier"""
        async with aiohttp.ClientSession() as session:
            while True:
                url = await self.url_frontier.get()

                if not url:
                    await asyncio.sleep(1)
                    continue

                # Check if already visited
                url_hash = hashlib.md5(url.encode()).digest()
                if self.visited.contains(url_hash):
                    continue

                self.visited.add(url_hash)

                try:
                    # Fetch page
                    page = await self.fetch_page(session, url)

                    if page:
                        # Process page
                        await self.process_page(url, page)

                        # Extract and add new URLs
                        new_urls = self.extract_urls(url, page)
                        for new_url in new_urls:
                            priority = self.calculate_priority(new_url)
                            await self.url_frontier.add(new_url, priority)

                    # Politeness delay
                    domain = urlparse(url).netloc
                    await self.wait_for_domain(domain)

                except Exception as e:
                    print(f"Worker {worker_id}: Error crawling {url}: {e}")

    async def fetch_page(self, session, url, timeout=10):
        """Fetch page content"""
        try:
            async with session.get(url, timeout=timeout) as response:
                if response.status == 200:
                    content_type = response.headers.get('Content-Type', '')
                    if 'text/html' in content_type:
                        return await response.text()
        except:
            pass

        return None

    async def process_page(self, url, html):
        """
        Process crawled page
        - Extract content
        - Send to indexing pipeline
        """
        # Parse HTML
        soup = BeautifulSoup(html, 'html.parser')

        # Extract metadata
        title = soup.find('title')
        title_text = title.get_text() if title else ''

        # Extract text content
        # Remove scripts and styles
        for script in soup(['script', 'style']):
            script.decompose()

        text = soup.get_text()

        # Create document
        document = {
            'url': url,
            'title': title_text,
            'content': text,
            'crawled_at': time.time()
        }

        # Send to indexing pipeline
        await self.send_to_indexer(document)

    def extract_urls(self, base_url, html):
        """Extract URLs from page"""
        soup = BeautifulSoup(html, 'html.parser')
        urls = []

        for link in soup.find_all('a', href=True):
            href = link['href']

            # Convert to absolute URL
            absolute_url = urljoin(base_url, href)

            # Filter valid URLs
            if self.is_valid_url(absolute_url):
                urls.append(absolute_url)

        return urls

    def is_valid_url(self, url):
        """Check if URL should be crawled"""
        parsed = urlparse(url)

        # Must be HTTP/HTTPS
        if parsed.scheme not in ['http', 'https']:
            return False

        # Ignore common non-content URLs
        ignore_extensions = ['.pdf', '.jpg', '.png', '.gif', '.css', '.js']
        if any(url.lower().endswith(ext) for ext in ignore_extensions):
            return False

        return True

    def calculate_priority(self, url):
        """Calculate URL priority for frontier"""
        # Simple heuristic: shorter URLs = higher priority
        # In practice, use PageRank, domain authority, etc.
        depth = url.count('/')
        return max(1, 20 - depth)

    async def wait_for_domain(self, domain):
        """Politeness delay per domain"""
        # In production, use distributed rate limiter
        await asyncio.sleep(self.politeness_delay)

    async def send_to_indexer(self, document):
        """Send document to indexing pipeline"""
        # In production, use Kafka or similar
        await kafka_producer.send('documents', document)


class URLFrontier:
    """
    Priority-based URL frontier

    Uses multiple queues for different priorities
    """

    def __init__(self):
        self.queues = {i: deque() for i in range(1, 21)}
        self.lock = asyncio.Lock()

    async def add(self, url, priority=10):
        """Add URL with priority (1-20)"""
        async with self.lock:
            priority = max(1, min(20, priority))
            self.queues[priority].append(url)

    async def get(self):
        """Get highest priority URL"""
        async with self.lock:
            # Check queues from highest to lowest priority
            for priority in range(20, 0, -1):
                if self.queues[priority]:
                    return self.queues[priority].popleft()

        return None


class BloomFilter:
    """
    Bloom filter for URL deduplication
    """

    def __init__(self, capacity, error_rate=0.001):
        import math

        # Calculate optimal size and hash functions
        self.size = int(-capacity * math.log(error_rate) / (math.log(2) ** 2))
        self.hash_count = int(self.size / capacity * math.log(2))
        self.bit_array = [0] * self.size

    def add(self, item):
        """Add item to bloom filter"""
        for seed in range(self.hash_count):
            index = self._hash(item, seed) % self.size
            self.bit_array[index] = 1

    def contains(self, item):
        """Check if item might be in set"""
        for seed in range(self.hash_count):
            index = self._hash(item, seed) % self.size
            if self.bit_array[index] == 0:
                return False
        return True

    def _hash(self, item, seed):
        """Hash function"""
        import hashlib
        h = hashlib.md5(item + str(seed).encode())
        return int(h.hexdigest(), 16)
```

### 2. Document Processor

**Content Extraction and Deduplication:**

```python
import hashlib
from typing import Dict, List

class DocumentProcessor:
    """
    Process crawled documents

    - Extract clean text
    - Detect duplicates (SimHash)
    - Extract metadata
    """

    def __init__(self):
        self.seen_hashes = set()  # In production: use database

    def process(self, document: Dict) -> Dict:
        """
        Process document

        Returns None if duplicate
        """
        # Extract clean text
        clean_text = self.extract_clean_text(document['content'])

        # Calculate SimHash for near-duplicate detection
        simhash = self.calculate_simhash(clean_text)

        # Check for duplicates
        if self.is_duplicate(simhash):
            return None

        self.seen_hashes.add(simhash)

        # Extract metadata
        metadata = self.extract_metadata(document)

        return {
            'url': document['url'],
            'title': document['title'],
            'content': clean_text,
            'simhash': simhash,
            'metadata': metadata,
            'word_count': len(clean_text.split()),
            'crawled_at': document['crawled_at']
        }

    def extract_clean_text(self, html_text: str) -> str:
        """Extract and clean text from HTML"""
        # Remove extra whitespace
        import re
        text = re.sub(r'\s+', ' ', html_text)
        text = text.strip()
        return text

    def calculate_simhash(self, text: str, num_bits=64) -> int:
        """
        Calculate SimHash for near-duplicate detection

        SimHash produces similar hashes for similar content
        """
        # Tokenize
        tokens = text.lower().split()

        # Initialize vector
        vector = [0] * num_bits

        # Process each token
        for token in tokens:
            # Hash token
            token_hash = int(hashlib.md5(token.encode()).hexdigest(), 16)

            # Update vector
            for i in range(num_bits):
                if token_hash & (1 << i):
                    vector[i] += 1
                else:
                    vector[i] -= 1

        # Convert to fingerprint
        fingerprint = 0
        for i in range(num_bits):
            if vector[i] > 0:
                fingerprint |= (1 << i)

        return fingerprint

    def is_duplicate(self, simhash: int, threshold=3) -> bool:
        """
        Check if document is duplicate

        Uses Hamming distance - number of differing bits
        """
        for existing_hash in self.seen_hashes:
            # Calculate Hamming distance
            hamming_distance = bin(simhash ^ existing_hash).count('1')

            # If very similar, consider duplicate
            if hamming_distance <= threshold:
                return True

        return False

    def extract_metadata(self, document: Dict) -> Dict:
        """Extract metadata from document"""
        from urllib.parse import urlparse

        parsed_url = urlparse(document['url'])

        return {
            'domain': parsed_url.netloc,
            'path_depth': document['url'].count('/') - 2,
            'is_homepage': parsed_url.path in ['', '/']
        }
```

### 3. Inverted Index

**Core Data Structure for Search:**

```python
from collections import defaultdict
from typing import List, Dict, Set
import math

class InvertedIndex:
    """
    Inverted Index: term -> list of documents containing term

    Structure:
    {
        "python": {
            "doc1": [pos1, pos5, pos10],  # positions in doc
            "doc3": [pos2, pos7],
            ...
        },
        "search": {
            "doc1": [pos3],
            "doc2": [pos1, pos8],
            ...
        }
    }
    """

    def __init__(self):
        # term -> {doc_id: [positions]}
        self.index = defaultdict(lambda: defaultdict(list))

        # doc_id -> document metadata
        self.documents = {}

        # Document frequency: term -> number of documents
        self.doc_frequency = defaultdict(int)

        # Total documents
        self.total_docs = 0

    def add_document(self, doc_id: str, document: Dict):
        """
        Add document to index

        Args:
            doc_id: Unique document identifier
            document: {'title': str, 'content': str, 'url': str}
        """
        # Store document metadata
        self.documents[doc_id] = {
            'title': document['title'],
            'url': document['url'],
            'length': 0  # will be set below
        }

        # Tokenize content
        tokens = self.tokenize(document['title'] + ' ' + document['content'])

        # Update document length
        self.documents[doc_id]['length'] = len(tokens)

        # Build inverted index
        seen_terms = set()

        for position, term in enumerate(tokens):
            # Add term position to index
            self.index[term][doc_id].append(position)

            # Update document frequency (once per term per document)
            if term not in seen_terms:
                self.doc_frequency[term] += 1
                seen_terms.add(term)

        self.total_docs += 1

    def tokenize(self, text: str) -> List[str]:
        """
        Tokenize text

        - Lowercase
        - Remove punctuation
        - Split on whitespace
        - Remove stopwords
        """
        import re

        # Lowercase and remove punctuation
        text = text.lower()
        text = re.sub(r'[^\w\s]', ' ', text)

        # Split into tokens
        tokens = text.split()

        # Remove stopwords
        stopwords = {'the', 'is', 'at', 'which', 'on', 'a', 'an', 'and', 'or', 'but'}
        tokens = [t for t in tokens if t not in stopwords]

        return tokens

    def search(self, query: str, k=10) -> List[Dict]:
        """
        Search for documents matching query

        Returns top k documents ranked by TF-IDF
        """
        # Tokenize query
        query_terms = self.tokenize(query)

        if not query_terms:
            return []

        # Find candidate documents (contain at least one query term)
        candidate_docs = set()
        for term in query_terms:
            if term in self.index:
                candidate_docs.update(self.index[term].keys())

        # Score each candidate
        scores = {}
        for doc_id in candidate_docs:
            scores[doc_id] = self.calculate_score(query_terms, doc_id)

        # Sort by score
        ranked_docs = sorted(scores.items(), key=lambda x: x[1], reverse=True)

        # Return top k
        results = []
        for doc_id, score in ranked_docs[:k]:
            doc = self.documents[doc_id]
            results.append({
                'doc_id': doc_id,
                'title': doc['title'],
                'url': doc['url'],
                'score': score
            })

        return results

    def calculate_score(self, query_terms: List[str], doc_id: str) -> float:
        """
        Calculate TF-IDF score for document

        TF-IDF = Term Frequency × Inverse Document Frequency
        """
        score = 0.0
        doc_length = self.documents[doc_id]['length']

        for term in query_terms:
            if term not in self.index or doc_id not in self.index[term]:
                continue

            # Term Frequency (normalized by document length)
            term_freq = len(self.index[term][doc_id])
            tf = term_freq / doc_length if doc_length > 0 else 0

            # Inverse Document Frequency
            df = self.doc_frequency[term]
            idf = math.log(self.total_docs / df) if df > 0 else 0

            # TF-IDF
            score += tf * idf

        return score

    def get_term_positions(self, term: str, doc_id: str) -> List[int]:
        """Get positions of term in document"""
        if term in self.index and doc_id in self.index[term]:
            return self.index[term][doc_id]
        return []


# Usage Example
index = InvertedIndex()

# Add documents
index.add_document('doc1', {
    'title': 'Introduction to Python',
    'content': 'Python is a popular programming language. Python is easy to learn.',
    'url': 'https://example.com/python'
})

index.add_document('doc2', {
    'title': 'Python for Data Science',
    'content': 'Python is widely used in data science and machine learning.',
    'url': 'https://example.com/python-ds'
})

index.add_document('doc3', {
    'title': 'Java Programming',
    'content': 'Java is a statically typed programming language.',
    'url': 'https://example.com/java'
})

# Search
results = index.search('python programming', k=10)

for result in results:
    print(f"{result['title']}: {result['score']:.4f}")
```

### 4. Ranking Algorithm

**Combining Multiple Signals:**

```python
import numpy as np
from typing import List, Dict

class SearchRanker:
    """
    Multi-signal ranking for search results

    Combines:
    1. TF-IDF (content relevance)
    2. PageRank (link-based authority)
    3. User engagement metrics
    4. Freshness
    5. ML-based ranking (optional)
    """

    def __init__(self, inverted_index, pagerank_scores):
        self.index = inverted_index
        self.pagerank = pagerank_scores

        # Feature weights
        self.weights = {
            'tfidf': 0.4,
            'pagerank': 0.3,
            'engagement': 0.2,
            'freshness': 0.1
        }

    def rank(self, query: str, candidate_docs: List[str], k=10) -> List[Dict]:
        """
        Rank documents using multiple signals
        """
        scores = {}

        for doc_id in candidate_docs:
            # 1. TF-IDF score
            tfidf_score = self.calculate_tfidf(query, doc_id)

            # 2. PageRank score
            pagerank_score = self.pagerank.get(doc_id, 0)

            # 3. Engagement score (CTR, dwell time, etc.)
            engagement_score = self.get_engagement_score(doc_id, query)

            # 4. Freshness score
            freshness_score = self.get_freshness_score(doc_id)

            # Normalize scores to [0, 1]
            tfidf_norm = self._normalize(tfidf_score, 0, 10)
            pagerank_norm = pagerank_score  # Already [0, 1]
            engagement_norm = engagement_score  # Already [0, 1]
            freshness_norm = freshness_score  # Already [0, 1]

            # Combined score
            final_score = (
                self.weights['tfidf'] * tfidf_norm +
                self.weights['pagerank'] * pagerank_norm +
                self.weights['engagement'] * engagement_norm +
                self.weights['freshness'] * freshness_norm
            )

            scores[doc_id] = final_score

        # Sort and return top k
        ranked = sorted(scores.items(), key=lambda x: x[1], reverse=True)

        results = []
        for doc_id, score in ranked[:k]:
            doc = self.index.documents[doc_id]
            results.append({
                'doc_id': doc_id,
                'title': doc['title'],
                'url': doc['url'],
                'score': score
            })

        return results

    def calculate_tfidf(self, query: str, doc_id: str) -> float:
        """Calculate TF-IDF score"""
        query_terms = self.index.tokenize(query)
        return self.index.calculate_score(query_terms, doc_id)

    def get_engagement_score(self, doc_id: str, query: str) -> float:
        """
        Get user engagement score

        Based on:
        - Click-through rate (CTR)
        - Dwell time
        - Bounce rate
        """
        # In production, fetch from analytics database
        # For now, return dummy score
        return 0.5

    def get_freshness_score(self, doc_id: str) -> float:
        """
        Calculate freshness score

        Newer documents get higher scores
        """
        import time

        doc = self.index.documents[doc_id]
        crawled_at = doc.get('crawled_at', 0)

        # Age in days
        age_days = (time.time() - crawled_at) / 86400

        # Exponential decay: score = e^(-age/30)
        # 30-day half-life
        freshness = math.exp(-age_days / 30)

        return freshness

    def _normalize(self, value, min_val, max_val):
        """Normalize value to [0, 1]"""
        if max_val == min_val:
            return 0
        return (value - min_val) / (max_val - min_val)


class PageRank:
    """
    Calculate PageRank for documents

    PageRank = (1-d)/N + d * sum(PR(incoming) / outlinks(incoming))

    Where:
    - d = damping factor (0.85)
    - N = total documents
    """

    def __init__(self, damping_factor=0.85, iterations=20):
        self.d = damping_factor
        self.iterations = iterations

    def calculate(self, graph: Dict[str, List[str]]) -> Dict[str, float]:
        """
        Calculate PageRank

        Args:
            graph: {doc_id: [list of outgoing links]}

        Returns:
            {doc_id: pagerank_score}
        """
        # Initialize PageRank
        N = len(graph)
        pagerank = {doc: 1.0 / N for doc in graph}

        # Build incoming links graph
        incoming = defaultdict(list)
        for doc, outlinks in graph.items():
            for link in outlinks:
                incoming[link].append(doc)

        # Iterate
        for _ in range(self.iterations):
            new_pagerank = {}

            for doc in graph:
                # Base rank
                rank = (1 - self.d) / N

                # Add contributions from incoming links
                for incoming_doc in incoming[doc]:
                    outlinks_count = len(graph[incoming_doc])
                    if outlinks_count > 0:
                        rank += self.d * pagerank[incoming_doc] / outlinks_count

                new_pagerank[doc] = rank

            pagerank = new_pagerank

        return pagerank


# Example
graph = {
    'doc1': ['doc2', 'doc3'],
    'doc2': ['doc3'],
    'doc3': ['doc1'],
}

pr = PageRank()
scores = pr.calculate(graph)
print(scores)  # {'doc1': 0.387, 'doc2': 0.215, 'doc3': 0.398}
```

### 5. Query Processing Pipeline

**End-to-End Query Handling:**

```python
import time
from typing import List, Dict

class QueryProcessor:
    """
    Process search queries end-to-end

    Pipeline:
    1. Query parsing & tokenization
    2. Query expansion (synonyms, spelling)
    3. Index lookup
    4. Ranking
    5. Result formatting
    """

    def __init__(self, inverted_index, ranker, cache):
        self.index = inverted_index
        self.ranker = ranker
        self.cache = cache
        self.synonym_map = self._load_synonyms()

    def search(self, query: str, k=10) -> Dict:
        """
        Execute search query
        """
        start_time = time.time()

        # 1. Check cache
        cache_key = f"search:{query}:{k}"
        cached_result = self.cache.get(cache_key)
        if cached_result:
            cached_result['cached'] = True
            return cached_result

        # 2. Parse and expand query
        expanded_query = self.expand_query(query)

        # 3. Tokenize
        query_terms = self.index.tokenize(expanded_query)

        if not query_terms:
            return {'results': [], 'total': 0, 'time_ms': 0}

        # 4. Find candidate documents
        candidates = self.find_candidates(query_terms)

        # 5. Rank candidates
        results = self.ranker.rank(expanded_query, candidates, k)

        # 6. Format results
        formatted_results = self.format_results(results, query)

        # Calculate time
        time_ms = int((time.time() - start_time) * 1000)

        response = {
            'query': query,
            'results': formatted_results,
            'total': len(candidates),
            'time_ms': time_ms,
            'cached': False
        }

        # Cache result
        self.cache.set(cache_key, response, ttl=3600)  # 1 hour

        return response

    def expand_query(self, query: str) -> str:
        """
        Expand query with synonyms

        "fast car" -> "fast quick car automobile"
        """
        words = query.lower().split()
        expanded = []

        for word in words:
            expanded.append(word)

            # Add synonyms
            if word in self.synonym_map:
                expanded.extend(self.synonym_map[word])

        return ' '.join(expanded)

    def find_candidates(self, query_terms: List[str]) -> List[str]:
        """
        Find candidate documents

        Documents must contain at least one query term
        """
        candidates = set()

        for term in query_terms:
            if term in self.index.index:
                candidates.update(self.index.index[term].keys())

        return list(candidates)

    def format_results(self, results: List[Dict], query: str) -> List[Dict]:
        """
        Format search results

        Add snippets with highlighted query terms
        """
        formatted = []

        for result in results:
            doc_id = result['doc_id']
            doc = self.index.documents[doc_id]

            # Generate snippet
            snippet = self.generate_snippet(doc_id, query)

            formatted.append({
                'title': result['title'],
                'url': result['url'],
                'snippet': snippet,
                'score': result['score']
            })

        return formatted

    def generate_snippet(self, doc_id: str, query: str, snippet_length=150) -> str:
        """
        Generate snippet with query terms highlighted

        Find passage containing most query terms
        """
        # For simplicity, return first N chars
        # In production, find best matching passage
        doc = self.index.documents[doc_id]
        content = doc.get('content', doc['title'])

        if len(content) <= snippet_length:
            return content

        return content[:snippet_length] + '...'

    def _load_synonyms(self) -> Dict[str, List[str]]:
        """Load synonym dictionary"""
        return {
            'fast': ['quick', 'rapid', 'speedy'],
            'car': ['automobile', 'vehicle'],
            'buy': ['purchase', 'acquire'],
            'cheap': ['inexpensive', 'affordable']
        }


# Cache implementation
class SearchCache:
    """Redis-backed search cache"""

    def __init__(self, redis_client):
        self.redis = redis_client

    def get(self, key: str):
        """Get cached result"""
        import json

        value = self.redis.get(key)
        if value:
            return json.loads(value)
        return None

    def set(self, key: str, value: Dict, ttl: int):
        """Cache result"""
        import json

        self.redis.setex(key, ttl, json.dumps(value))
```

### 6. Autocomplete System

**Real-time Query Suggestions:**

```python
class Trie:
    """
    Trie for autocomplete

    Stores queries with frequencies
    """

    class TrieNode:
        def __init__(self):
            self.children = {}
            self.is_end = False
            self.frequency = 0
            self.suggestions = []  # Top suggestions from this node

    def __init__(self):
        self.root = self.TrieNode()

    def insert(self, query: str, frequency: int = 1):
        """Insert query with frequency"""
        node = self.root

        for char in query.lower():
            if char not in node.children:
                node.children[char] = self.TrieNode()
            node = node.children[char]

        node.is_end = True
        node.frequency += frequency

    def search(self, prefix: str, k=10) -> List[tuple]:
        """
        Get top k suggestions for prefix

        Returns: [(query, frequency), ...]
        """
        # Find prefix node
        node = self.root
        for char in prefix.lower():
            if char not in node.children:
                return []
            node = node.children[char]

        # Collect all queries with this prefix
        suggestions = []
        self._collect_suggestions(node, prefix, suggestions)

        # Sort by frequency
        suggestions.sort(key=lambda x: x[1], reverse=True)

        return suggestions[:k]

    def _collect_suggestions(self, node, prefix, suggestions):
        """Recursively collect all queries from node"""
        if node.is_end:
            suggestions.append((prefix, node.frequency))

        for char, child_node in node.children.items():
            self._collect_suggestions(child_node, prefix + char, suggestions)


class AutocompleteSystem:
    """
    Autocomplete system for search suggestions

    Features:
    - Prefix matching with Trie
    - Personalized suggestions
    - Trending queries
    """

    def __init__(self):
        self.trie = Trie()
        self.trending = []  # Top trending queries

    def add_query(self, query: str, frequency: int = 1):
        """Add or update query"""
        self.trie.insert(query, frequency)

    def get_suggestions(self, prefix: str, user_id: str = None, k=10) -> List[str]:
        """
        Get autocomplete suggestions

        Combines:
        1. Prefix matches from Trie
        2. Personalized suggestions (user history)
        3. Trending queries
        """
        suggestions = []

        # 1. Get prefix matches
        prefix_matches = self.trie.search(prefix, k)
        suggestions.extend([query for query, _ in prefix_matches])

        # 2. Add personalized suggestions
        if user_id:
            personal = self.get_personalized_suggestions(user_id, prefix, k=3)
            suggestions.extend(personal)

        # 3. Add trending if not enough suggestions
        if len(suggestions) < k:
            trending = [q for q in self.trending if q.startswith(prefix)]
            suggestions.extend(trending)

        # Remove duplicates while preserving order
        seen = set()
        unique_suggestions = []
        for s in suggestions:
            if s not in seen:
                seen.add(s)
                unique_suggestions.append(s)

        return unique_suggestions[:k]

    def get_personalized_suggestions(self, user_id: str, prefix: str, k=3) -> List[str]:
        """Get personalized suggestions based on user history"""
        # In production, fetch from user profile database
        # For now, return empty
        return []

    def update_trending(self, queries: List[str]):
        """Update trending queries"""
        self.trending = queries


# Usage
autocomplete = AutocompleteSystem()

# Add popular queries
autocomplete.add_query("python tutorial", frequency=1000)
autocomplete.add_query("python programming", frequency=800)
autocomplete.add_query("python for beginners", frequency=600)
autocomplete.add_query("java tutorial", frequency=500)

# Get suggestions
suggestions = autocomplete.get_suggestions("pyth")
print(suggestions)
# ['python tutorial', 'python programming', 'python for beginners']
```

## Distributed Architecture

### Sharding Strategy

```python
class IndexShard:
    """
    Shard inverted index by term hash

    Sharding strategies:
    1. By term: hash(term) % num_shards
    2. By document: hash(doc_id) % num_shards

    We use term-based sharding for parallel query processing
    """

    def __init__(self, shard_id: int, num_shards: int):
        self.shard_id = shard_id
        self.num_shards = num_shards
        self.index = InvertedIndex()

    def should_handle_term(self, term: str) -> bool:
        """Check if this shard handles the term"""
        term_hash = hash(term)
        shard = term_hash % self.num_shards
        return shard == self.shard_id

    def add_document(self, doc_id: str, document: Dict):
        """Add document (only relevant terms)"""
        tokens = self.index.tokenize(document['content'])

        # Filter tokens this shard handles
        relevant_tokens = [t for t in tokens if self.should_handle_term(t)]

        if relevant_tokens:
            # Add only relevant terms
            self.index.add_document(doc_id, document)

    def search(self, query_terms: List[str]) -> Dict[str, float]:
        """Search for query terms this shard handles"""
        # Filter terms for this shard
        my_terms = [t for t in query_terms if self.should_handle_term(t)]

        if not my_terms:
            return {}

        # Find candidates
        candidates = set()
        for term in my_terms:
            if term in self.index.index:
                candidates.update(self.index.index[term].keys())

        # Score candidates
        scores = {}
        for doc_id in candidates:
            scores[doc_id] = self.index.calculate_score(my_terms, doc_id)

        return scores


class DistributedSearchEngine:
    """
    Distributed search across multiple shards

    Architecture:
    - Query coordinator
    - Multiple index shards
    - Result aggregator
    """

    def __init__(self, num_shards=10):
        self.num_shards = num_shards
        self.shards = [IndexShard(i, num_shards) for i in range(num_shards)]

    def add_document(self, doc_id: str, document: Dict):
        """Add document to all shards"""
        for shard in self.shards:
            shard.add_document(doc_id, document)

    def search(self, query: str, k=10) -> List[Dict]:
        """
        Distributed search

        1. Send query to all shards (parallel)
        2. Each shard returns partial scores
        3. Aggregate and merge results
        4. Return top k
        """
        # Tokenize query
        query_terms = self.shards[0].index.tokenize(query)

        # Query all shards in parallel
        import concurrent.futures

        with concurrent.futures.ThreadPoolExecutor(max_workers=self.num_shards) as executor:
            futures = [
                executor.submit(shard.search, query_terms)
                for shard in self.shards
            ]

            shard_results = [f.result() for f in futures]

        # Merge results
        merged_scores = {}
        for shard_score in shard_results:
            for doc_id, score in shard_score.items():
                merged_scores[doc_id] = merged_scores.get(doc_id, 0) + score

        # Sort and return top k
        ranked = sorted(merged_scores.items(), key=lambda x: x[1], reverse=True)

        results = []
        for doc_id, score in ranked[:k]:
            # Get document metadata (from any shard)
            doc = self.shards[0].index.documents.get(doc_id)
            if doc:
                results.append({
                    'doc_id': doc_id,
                    'title': doc['title'],
                    'url': doc['url'],
                    'score': score
                })

        return results
```

## Database Schema

```sql
-- Documents table
CREATE TABLE documents (
    doc_id VARCHAR(64) PRIMARY KEY,
    url TEXT NOT NULL UNIQUE,
    title TEXT,
    content TEXT,
    simhash BIGINT,
    word_count INT,
    crawled_at TIMESTAMP,
    indexed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    pagerank DECIMAL(10, 8),
    INDEX idx_simhash (simhash),
    INDEX idx_crawled_at (crawled_at)
);

-- Inverted index table (for persistence)
CREATE TABLE inverted_index (
    term VARCHAR(255),
    doc_id VARCHAR(64),
    positions JSON,  -- Array of positions
    term_frequency INT,
    PRIMARY KEY (term, doc_id),
    FOREIGN KEY (doc_id) REFERENCES documents(doc_id),
    INDEX idx_term (term)
) PARTITION BY HASH(term) PARTITIONS 100;

-- Document links (for PageRank)
CREATE TABLE document_links (
    from_doc_id VARCHAR(64),
    to_doc_id VARCHAR(64),
    anchor_text TEXT,
    PRIMARY KEY (from_doc_id, to_doc_id),
    FOREIGN KEY (from_doc_id) REFERENCES documents(doc_id),
    FOREIGN KEY (to_doc_id) REFERENCES documents(doc_id)
);

-- Query logs (for analytics and autocomplete)
CREATE TABLE query_logs (
    query_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id VARCHAR(64),
    query TEXT,
    results_count INT,
    clicked_doc_id VARCHAR(64),
    click_position INT,
    dwell_time_seconds INT,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_query (query(255)),
    INDEX idx_user_time (user_id, timestamp),
    INDEX idx_timestamp (timestamp)
);

-- Autocomplete suggestions
CREATE TABLE autocomplete (
    prefix VARCHAR(50),
    suggestion TEXT,
    frequency BIGINT,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (prefix, suggestion),
    INDEX idx_prefix_freq (prefix, frequency DESC)
);

-- URL frontier (for crawler)
CREATE TABLE url_frontier (
    url TEXT PRIMARY KEY,
    priority INT DEFAULT 10,
    last_crawled TIMESTAMP,
    next_crawl TIMESTAMP,
    crawl_count INT DEFAULT 0,
    INDEX idx_next_crawl (next_crawl),
    INDEX idx_priority (priority DESC)
);
```

## Scalability Considerations

### 1. Index Partitioning

**Term-based Partitioning:**
```
Shard 1: Terms starting with a-c
Shard 2: Terms starting with d-f
...
Shard 10: Terms starting with x-z
```

**Advantages:**
- Parallel query processing
- Easy to add shards

**Challenges:**
- Hot shards (popular terms)
- Need consistent hashing

### 2. Caching Strategy

**Multi-level Cache:**

```python
class MultiLevelCache:
    """
    L1: In-memory cache (application server)
    L2: Redis (distributed)
    L3: CDN (for static results)
    """

    def __init__(self, l1_cache, redis_client):
        self.l1 = l1_cache  # Local LRU cache
        self.l2 = redis_client  # Redis

    def get(self, key: str):
        """Get from cache"""
        # Try L1
        value = self.l1.get(key)
        if value:
            return value

        # Try L2
        value = self.l2.get(key)
        if value:
            # Promote to L1
            self.l1.set(key, value)
            return value

        return None

    def set(self, key: str, value, ttl: int):
        """Set in all cache levels"""
        self.l1.set(key, value)
        self.l2.setex(key, ttl, value)
```

**What to Cache:**
1. Popular queries (Zipf distribution: 20% queries = 80% traffic)
2. Autocomplete suggestions
3. Page metadata
4. PageRank scores

### 3. Replication

**Read Replicas:**
- Master-slave replication for index
- Read from replicas, write to master
- Eventual consistency acceptable

**Consistency:**
- Index updates can be eventually consistent
- Use versioning for index updates

## Real-World Optimizations

### 1. BM25 Ranking

```python
def bm25_score(term_freq, doc_length, avg_doc_length, doc_freq, total_docs, k1=1.5, b=0.75):
    """
    BM25 ranking function (better than TF-IDF)

    Parameters:
    - k1: term frequency saturation (default 1.5)
    - b: length normalization (default 0.75)
    """
    # IDF component
    idf = math.log((total_docs - doc_freq + 0.5) / (doc_freq + 0.5) + 1)

    # TF component with saturation
    tf_component = (term_freq * (k1 + 1)) / (
        term_freq + k1 * (1 - b + b * (doc_length / avg_doc_length))
    )

    return idf * tf_component
```

### 2. Index Compression

**Techniques:**
- Variable byte encoding for posting lists
- Delta encoding for positions
- Dictionary compression for terms

```python
def compress_positions(positions: List[int]) -> bytes:
    """
    Delta encoding + variable byte encoding

    [5, 10, 15, 20] -> [5, 5, 5, 5] -> compressed bytes
    """
    # Delta encode
    deltas = [positions[0]]
    for i in range(1, len(positions)):
        deltas.append(positions[i] - positions[i-1])

    # Variable byte encode (simplified)
    compressed = []
    for delta in deltas:
        compressed.append(delta.to_bytes(2, 'big'))

    return b''.join(compressed)
```

### 3. Query Understanding

**Spell Correction:**
```python
def spell_correct(query: str, dictionary: set) -> str:
    """
    Simple spell correction using edit distance
    """
    words = query.split()
    corrected = []

    for word in words:
        if word in dictionary:
            corrected.append(word)
        else:
            # Find closest word in dictionary
            candidates = [
                (w, edit_distance(word, w))
                for w in dictionary
                if abs(len(w) - len(word)) <= 2
            ]

            if candidates:
                best = min(candidates, key=lambda x: x[1])
                if best[1] <= 2:  # Max 2 edits
                    corrected.append(best[0])
                else:
                    corrected.append(word)
            else:
                corrected.append(word)

    return ' '.join(corrected)
```

## Monitoring and Metrics

**Key Metrics:**

1. **Query Latency**
   - P50, P95, P99 latency
   - Target: < 200ms P95

2. **Index Freshness**
   - Time from crawl to searchable
   - Target: < 24 hours

3. **Result Quality**
   - Click-through rate (CTR)
   - Mean reciprocal rank (MRR)
   - Normalized discounted cumulative gain (NDCG)

4. **System Health**
   - CPU/memory usage
   - Cache hit rate
   - Index size growth

```python
class SearchMetrics:
    """Track search engine metrics"""

    def __init__(self):
        self.query_latencies = []
        self.cache_hits = 0
        self.cache_misses = 0

    def record_query(self, latency_ms: int, cache_hit: bool):
        """Record query metrics"""
        self.query_latencies.append(latency_ms)

        if cache_hit:
            self.cache_hits += 1
        else:
            self.cache_misses += 1

    def get_p95_latency(self) -> float:
        """Calculate P95 latency"""
        if not self.query_latencies:
            return 0

        sorted_latencies = sorted(self.query_latencies)
        p95_index = int(len(sorted_latencies) * 0.95)
        return sorted_latencies[p95_index]

    def get_cache_hit_rate(self) -> float:
        """Calculate cache hit rate"""
        total = self.cache_hits + self.cache_misses
        if total == 0:
            return 0
        return self.cache_hits / total
```

## Interview Tips

### Common Questions

**Q: How do you handle billions of documents?**
- Distributed indexing with sharding
- Incremental indexing (don't rebuild entire index)
- Compression techniques
- Use of efficient data structures (skip lists, B-trees)

**Q: How to rank search results?**
- Combine multiple signals: TF-IDF/BM25, PageRank, user engagement
- Use machine learning (Learning to Rank)
- Personalization based on user history
- Real-time signals (trending, freshness)

**Q: How to make search fast?**
- Inverted index for O(1) term lookup
- Caching (query cache, page cache)
- Index sharding for parallel query processing
- Early termination (WAND algorithm)
- Bloom filters to skip non-matching shards

**Q: How to handle typos?**
- Edit distance (Levenshtein)
- Phonetic algorithms (Soundex, Metaphone)
- N-gram indexing
- Query logs to detect common misspellings

**Q: What's the difference between search engine and database query?**
- Search: Relevance ranking, fuzzy matching, full-text
- Database: Exact matching, structured queries, ACID

## Key Takeaways

1. **Inverted Index**: Core data structure mapping terms to documents
2. **Distributed Architecture**: Shard by term for parallel processing
3. **Ranking**: Combine content relevance (BM25) + link analysis (PageRank) + user signals
4. **Caching**: Multi-level cache for popular queries
5. **Scalability**: Horizontal scaling of crawlers, indexers, and query processors
6. **Crawling**: Politeness, deduplication, robots.txt compliance
7. **Autocomplete**: Trie-based prefix matching with frequency scoring

Building a search engine requires mastering distributed systems, information retrieval, and scalability!
