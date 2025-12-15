# Design a Web Crawler

## Table of Contents
- [Problem Statement](#problem-statement)
- [Requirements](#requirements)
- [High-Level Design](#high-level-design)
- [URL Frontier](#url-frontier)
- [Fetcher and Parser](#fetcher-and-parser)
- [Politeness and Rate Limiting](#politeness-and-rate-limiting)
- [Deduplication](#deduplication)
- [Distributed Crawling](#distributed-crawling)
- [Storage](#storage)
- [Implementation](#implementation)
- [Optimizations](#optimizations)
- [Real-World Examples](#real-world-examples)
- [Interview Tips](#interview-tips)

---

## Problem Statement

Design a **web crawler** that can:
- Download billions of web pages
- Follow links to discover new pages
- Respect robots.txt and crawl politeness
- Handle failures and retries
- Scale horizontally

**Similar to**: Googlebot, Bingbot, web scraping systems

---

## Requirements

### Functional Requirements

1. **Crawl web pages**: Download HTML content
2. **Follow links**: Extract and crawl linked pages
3. **Respect robots.txt**: Honor crawl directives
4. **Politeness**: Rate limit requests per domain
5. **Prioritization**: Crawl important pages first
6. **Store content**: Save HTML and metadata

### Non-Functional Requirements

1. **Scalability**: Billions of pages
2. **Efficiency**: Millions of pages per day
3. **Politeness**: Don't overload servers
4. **Fault Tolerance**: Handle failures gracefully
5. **Extensibility**: Support different content types
6. **Deduplication**: Avoid crawling same page multiple times

---

## High-Level Design

```
┌────────────────────────────────────────────────────────────┐
│                      Seed URLs                              │
│            (Starting points for crawl)                      │
└────────────────┬───────────────────────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────────────────────┐
│                    URL Frontier                             │
│          (Priority queue of URLs to crawl)                  │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │ High Priority│  │ Medium Prior.│  │  Low Prior.  │    │
│  │   (News)     │  │  (Popular)   │  │   (Other)    │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
└────────────────┬───────────────────────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────────────────────┐
│                  URL Deduplicator                           │
│            (Bloom filter + URL fingerprints)                │
└────────────────┬───────────────────────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────────────────────┐
│                 Robots.txt Checker                          │
│              (Check if URL is crawlable)                    │
└────────────────┬───────────────────────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────────────────────┐
│                  Politeness Manager                         │
│          (Rate limit per domain: 1 req/sec)                │
└────────────────┬───────────────────────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────────────────────┐
│                   HTML Fetcher                              │
│              (Download page content)                        │
└────────────────┬───────────────────────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────────────────────┐
│                    DNS Resolver                             │
│                (Cache DNS lookups)                          │
└────────────────┬───────────────────────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────────────────────┐
│                   HTML Parser                               │
│          (Extract links and content)                        │
└────────────────┬───────────────────────────────────────────┘
                 │
            ┌────┴────┐
            ↓         ↓
    ┌────────────┐  ┌─────────────┐
    │New URLs    │  │  Content    │
    │(to frontier)│  │  Storage    │
    └────────────┘  └─────────────┘
```

---

## URL Frontier

### Priority Queue Implementation

```python
import heapq
from collections import defaultdict
from urllib.parse import urlparse

class URLFrontier:
    def __init__(self):
        # Priority queues per domain
        self.queues = defaultdict(list)  # domain -> [(priority, url)]

        # Track last fetch time per domain (politeness)
        self.last_fetch = {}  # domain -> timestamp

        # Domain priorities
        self.domain_priority = {}  # domain -> priority

    def add_url(self, url, priority=5):
        """
        Add URL to frontier
        Priority: 1 (highest) to 10 (lowest)
        """
        domain = urlparse(url).netloc

        # Store in domain's queue
        heapq.heappush(self.queues[domain], (priority, url))

    def get_next_url(self):
        """
        Get next URL to crawl (politeness-aware)

        Strategy:
        1. Round-robin across domains (avoid overwhelming one domain)
        2. Respect minimum delay between requests to same domain
        3. Higher priority URLs first within domain
        """
        import time

        # Find domain that can be crawled now
        for domain in self.queues:
            if not self.queues[domain]:
                continue

            # Check if we can crawl this domain (politeness)
            last_fetch = self.last_fetch.get(domain, 0)
            if time.time() - last_fetch < 1.0:  # 1 second delay
                continue

            # Get highest priority URL from this domain
            priority, url = heapq.heappop(self.queues[domain])

            # Update last fetch time
            self.last_fetch[domain] = time.time()

            return url

        return None  # No URLs available

    def size(self):
        """Total URLs in frontier"""
        return sum(len(q) for q in self.queues.values())

# Example usage
frontier = URLFrontier()
frontier.add_url('https://example.com/page1', priority=1)  # High priority
frontier.add_url('https://example.com/page2', priority=5)  # Medium
frontier.add_url('https://other.com/page', priority=3)

next_url = frontier.get_next_url()
print(f"Crawling: {next_url}")
```

### BFS vs DFS Strategy

```python
class CrawlStrategy:
    """
    BFS: Breadth-First Search (discover many sites)
    DFS: Depth-First Search (deep crawl of one site)
    """

    def bfs_crawl(self, seed_urls):
        """
        BFS: Good for discovering diverse content
        """
        from collections import deque

        queue = deque(seed_urls)
        visited = set()

        while queue:
            url = queue.popleft()  # FIFO

            if url in visited:
                continue

            visited.add(url)
            content, links = self.fetch_and_parse(url)

            # Add discovered links to queue
            for link in links:
                if link not in visited:
                    queue.append(link)

    def dfs_crawl(self, seed_urls):
        """
        DFS: Good for comprehensive site crawling
        """
        stack = list(seed_urls)
        visited = set()

        while stack:
            url = stack.pop()  # LIFO

            if url in visited:
                continue

            visited.add(url)
            content, links = self.fetch_and_parse(url)

            # Add discovered links to stack
            for link in links:
                if link not in visited:
                    stack.append(link)
```

---

## Fetcher and Parser

### HTTP Fetcher

```python
import requests
from urllib.robotparser import RobotFileParser

class HTTPFetcher:
    def __init__(self):
        self.session = requests.Session()
        self.session.headers.update({
            'User-Agent': 'MyCrawler/1.0 (+http://mycrawler.com/bot.html)'
        })
        self.timeout = 10  # seconds

    def fetch(self, url):
        """
        Fetch URL with error handling

        Returns:
            (status_code, content, headers)
        """
        try:
            response = self.session.get(
                url,
                timeout=self.timeout,
                allow_redirects=True
            )

            return {
                'status_code': response.status_code,
                'content': response.text,
                'headers': dict(response.headers),
                'url': response.url  # Final URL after redirects
            }

        except requests.Timeout:
            return {'status_code': 408, 'error': 'Timeout'}
        except requests.ConnectionError:
            return {'status_code': 503, 'error': 'Connection failed'}
        except Exception as e:
            return {'status_code': 500, 'error': str(e)}

class RobotsTxtChecker:
    def __init__(self):
        self.parsers = {}  # domain -> RobotFileParser

    def can_fetch(self, url):
        """Check if URL can be crawled per robots.txt"""
        from urllib.parse import urlparse

        parsed = urlparse(url)
        domain = parsed.netloc
        robots_url = f"{parsed.scheme}://{domain}/robots.txt"

        # Cache robots.txt parser
        if domain not in self.parsers:
            rp = RobotFileParser()
            rp.set_url(robots_url)
            try:
                rp.read()
                self.parsers[domain] = rp
            except:
                # If robots.txt fails, allow crawling
                return True

        return self.parsers[domain].can_fetch('*', url)
```

### HTML Parser

```python
from bs4 import BeautifulSoup
from urllib.parse import urljoin, urlparse

class HTMLParser:
    def __init__(self):
        self.allowed_domains = set()  # Optional: restrict to domains

    def parse(self, url, html_content):
        """
        Parse HTML and extract:
        - Links
        - Title
        - Meta description
        - Text content
        """
        soup = BeautifulSoup(html_content, 'html.parser')

        # Extract links
        links = self.extract_links(url, soup)

        # Extract metadata
        metadata = {
            'title': soup.title.string if soup.title else '',
            'description': self.get_meta_description(soup),
            'text': soup.get_text(strip=True)
        }

        return links, metadata

    def extract_links(self, base_url, soup):
        """Extract and normalize all links"""
        links = set()

        for tag in soup.find_all('a', href=True):
            href = tag['href']

            # Convert relative URLs to absolute
            absolute_url = urljoin(base_url, href)

            # Normalize URL (remove fragments, normalize case)
            normalized = self.normalize_url(absolute_url)

            # Filter out non-HTTP links
            if normalized.startswith(('http://', 'https://')):
                links.add(normalized)

        return list(links)

    def normalize_url(self, url):
        """
        Normalize URL:
        - Remove fragment (#section)
        - Lowercase scheme and domain
        - Remove trailing slash
        - Sort query parameters
        """
        from urllib.parse import urlparse, urlunparse, parse_qs, urlencode

        parsed = urlparse(url)

        # Lowercase scheme and netloc
        scheme = parsed.scheme.lower()
        netloc = parsed.netloc.lower()

        # Remove fragment
        fragment = ''

        # Sort query parameters
        query_params = parse_qs(parsed.query)
        sorted_query = urlencode(sorted(query_params.items()))

        # Remove trailing slash from path
        path = parsed.path.rstrip('/')

        normalized = urlunparse((
            scheme, netloc, path, parsed.params, sorted_query, fragment
        ))

        return normalized

    def get_meta_description(self, soup):
        """Extract meta description"""
        meta = soup.find('meta', attrs={'name': 'description'})
        return meta['content'] if meta and meta.get('content') else ''
```

---

## Politeness and Rate Limiting

### Domain-Level Rate Limiting

```python
import time
from collections import defaultdict
import threading

class PolitenessManager:
    def __init__(self, delay_seconds=1.0):
        self.delay_seconds = delay_seconds  # Delay between requests to same domain
        self.last_access = defaultdict(float)  # domain -> last_access_time
        self.locks = defaultdict(threading.Lock)  # domain -> lock

    def wait_if_needed(self, url):
        """
        Wait if necessary to respect politeness policy

        Ensures minimum delay between requests to same domain
        """
        from urllib.parse import urlparse

        domain = urlparse(url).netloc

        with self.locks[domain]:
            last_time = self.last_access[domain]
            elapsed = time.time() - last_time

            if elapsed < self.delay_seconds:
                wait_time = self.delay_seconds - elapsed
                time.sleep(wait_time)

            self.last_access[domain] = time.time()

# Usage
politeness = PolitenessManager(delay_seconds=1.0)

def crawl_url(url):
    politeness.wait_if_needed(url)
    # Now safe to crawl
    content = fetcher.fetch(url)
```

---

## Deduplication

### Bloom Filter for URL Deduplication

```python
from bitarray import bitarray
import mmh3

class BloomFilter:
    def __init__(self, size=10000000, hash_count=7):
        """
        Bloom filter for efficient URL deduplication

        False positive rate: ~0.01% with these params
        """
        self.size = size
        self.hash_count = hash_count
        self.bit_array = bitarray(size)
        self.bit_array.setall(0)

    def add(self, url):
        """Add URL to filter"""
        for i in range(self.hash_count):
            index = mmh3.hash(url, i) % self.size
            self.bit_array[index] = 1

    def contains(self, url):
        """Check if URL might have been seen (probabilistic)"""
        for i in range(self.hash_count):
            index = mmh3.hash(url, i) % self.size
            if not self.bit_array[index]:
                return False  # Definitely not seen
        return True  # Probably seen

class URLDeduplicator:
    def __init__(self):
        self.bloom_filter = BloomFilter()
        self.exact_urls = set()  # For exact deduplication (smaller set)

    def is_duplicate(self, url):
        """
        Two-level deduplication:
        1. Bloom filter (fast, probabilistic)
        2. Exact set (slower, definitive for positives)
        """
        # Quick bloom filter check
        if not self.bloom_filter.contains(url):
            # Definitely new
            self.bloom_filter.add(url)
            self.exact_urls.add(url)
            return False

        # Bloom filter says "maybe" - check exact set
        if url in self.exact_urls:
            return True  # Duplicate

        # Bloom filter false positive
        self.exact_urls.add(url)
        return False

# Usage
dedup = URLDeduplicator()

for url in urls:
    if not dedup.is_duplicate(url):
        crawl(url)
```

### Content Fingerprinting

```python
import hashlib

class ContentDeduplicator:
    def __init__(self):
        self.content_hashes = set()

    def is_duplicate_content(self, html_content):
        """
        Check if content is duplicate (different URL, same content)

        Use SimHash or MinHash for near-duplicate detection
        """
        # Simple MD5 hash for exact duplicates
        content_hash = hashlib.md5(html_content.encode()).hexdigest()

        if content_hash in self.content_hashes:
            return True

        self.content_hashes.add(content_hash)
        return False

    def simhash(self, text):
        """
        SimHash for near-duplicate detection
        Find documents that are 90%+ similar
        """
        # Simplified SimHash implementation
        tokens = text.lower().split()
        vector = [0] * 64

        for token in tokens:
            token_hash = int(hashlib.md5(token.encode()).hexdigest(), 16)
            for i in range(64):
                if token_hash & (1 << i):
                    vector[i] += 1
                else:
                    vector[i] -= 1

        # Convert to binary fingerprint
        fingerprint = 0
        for i in range(64):
            if vector[i] > 0:
                fingerprint |= (1 << i)

        return fingerprint

    def hamming_distance(self, hash1, hash2):
        """Count differing bits"""
        return bin(hash1 ^ hash2).count('1')
```

---

## Distributed Crawling

### Consistent Hashing for URL Distribution

```python
import hashlib

class DistributedCrawler:
    def __init__(self, num_workers=10):
        self.num_workers = num_workers

    def assign_worker(self, url):
        """
        Assign URL to worker using consistent hashing

        Ensures same domain goes to same worker (simplifies politeness)
        """
        from urllib.parse import urlparse

        domain = urlparse(url).netloc

        # Hash domain to worker
        domain_hash = int(hashlib.md5(domain.encode()).hexdigest(), 16)
        worker_id = domain_hash % self.num_workers

        return worker_id

# Coordinator distributes URLs to workers
class CrawlCoordinator:
    def __init__(self, num_workers=10):
        self.distributor = DistributedCrawler(num_workers)
        self.worker_queues = [[] for _ in range(num_workers)]

    def distribute_url(self, url):
        """Send URL to appropriate worker"""
        worker_id = self.distributor.assign_worker(url)
        self.worker_queues[worker_id].append(url)

    def get_urls_for_worker(self, worker_id):
        """Worker pulls URLs from its queue"""
        return self.worker_queues[worker_id]
```

---

## Storage

### Database Schema

```sql
-- URLs to crawl
CREATE TABLE url_frontier (
    url_id BIGSERIAL PRIMARY KEY,
    url TEXT UNIQUE NOT NULL,
    domain VARCHAR(255),
    priority INT DEFAULT 5,
    added_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_domain_priority (domain, priority)
);

-- Crawled pages
CREATE TABLE crawled_pages (
    page_id BIGSERIAL PRIMARY KEY,
    url TEXT UNIQUE NOT NULL,
    domain VARCHAR(255),
    title TEXT,
    content TEXT,
    html_content TEXT,
    crawled_at TIMESTAMP DEFAULT NOW(),
    status_code INT,
    INDEX idx_domain (domain),
    INDEX idx_crawled_at (crawled_at)
);

-- Links between pages
CREATE TABLE page_links (
    from_page_id BIGINT REFERENCES crawled_pages(page_id),
    to_url TEXT,
    anchor_text TEXT,
    PRIMARY KEY (from_page_id, to_url)
);
```

---

## Implementation

### Complete Crawler

```python
class WebCrawler:
    def __init__(self):
        self.frontier = URLFrontier()
        self.dedup = URLDeduplicator()
        self.fetcher = HTTPFetcher()
        self.parser = HTMLParser()
        self.politeness = PolitenessManager(delay_seconds=1.0)
        self.robots_checker = RobotsTxtChecker()

    def crawl(self, seed_urls, max_pages=1000):
        """Main crawl loop"""
        # Add seed URLs
        for url in seed_urls:
            self.frontier.add_url(url, priority=1)

        pages_crawled = 0

        while pages_crawled < max_pages:
            # Get next URL
            url = self.frontier.get_next_url()
            if not url:
                break

            # Check if already crawled
            if self.dedup.is_duplicate(url):
                continue

            # Check robots.txt
            if not self.robots_checker.can_fetch(url):
                print(f"Blocked by robots.txt: {url}")
                continue

            # Politeness delay
            self.politeness.wait_if_needed(url)

            # Fetch page
            response = self.fetcher.fetch(url)
            if response['status_code'] != 200:
                print(f"Failed to fetch {url}: {response['status_code']}")
                continue

            # Parse HTML
            links, metadata = self.parser.parse(url, response['content'])

            # Store page
            self.store_page(url, metadata, response['content'])

            # Add discovered links to frontier
            for link in links:
                if not self.dedup.is_duplicate(link):
                    self.frontier.add_url(link, priority=5)

            pages_crawled += 1
            print(f"Crawled {pages_crawled}/{max_pages}: {url}")

    def store_page(self, url, metadata, content):
        """Store crawled page"""
        # Save to database or file
        pass

# Usage
crawler = WebCrawler()
crawler.crawl(['https://example.com'], max_pages=100)
```

---

## Optimizations

### 1. DNS Caching

```python
import socket
from functools import lru_cache

@lru_cache(maxsize=10000)
def cached_dns_lookup(hostname):
    """Cache DNS lookups"""
    return socket.gethostbyname(hostname)
```

### 2. Connection Pooling

```python
import requests

session = requests.Session()
adapter = requests.adapters.HTTPAdapter(
    pool_connections=100,
    pool_maxsize=100
)
session.mount('http://', adapter)
session.mount('https://', adapter)
```

### 3. Focused Crawling

```python
class FocusedCrawler:
    """
    Crawl only pages related to specific topics
    Use ML to classify page relevance
    """
    def is_relevant(self, content):
        # Simple keyword-based
        keywords = ['machine learning', 'AI', 'neural network']
        return any(kw in content.lower() for kw in keywords)

    def crawl_focused(self, seed_urls):
        for url in self.crawl(seed_urls):
            content = fetch(url)
            if self.is_relevant(content):
                # High priority for relevant pages
                self.frontier.add_url(url, priority=1)
```

---

## Real-World Examples

### Googlebot
- **Crawl budget**: Limit pages per site per day
- **PageRank**: Prioritize important pages
- **Rendering**: JavaScript execution for SPA
- **Distributed**: Thousands of servers

### Common Crawl
- **Open dataset**: Billions of pages monthly
- **Petabyte-scale**: Distributed on S3
- **WARC format**: Web ARChive format

---

## Interview Tips

### Key Points

1. **URL Frontier**: Priority queue, politeness-aware
2. **Deduplication**: Bloom filter + exact set
3. **Robots.txt**: Respect crawl directives
4. **Politeness**: Rate limit per domain
5. **Distributed**: Consistent hashing for workers
6. **Scalability**: Queue-based, horizontal scaling

### Common Questions

**Q: How do you avoid crawling the same page twice?**
- Bloom filter for quick check
- Exact URL set for confirmation
- Content fingerprinting for duplicates

**Q: How do you handle millions of URLs in frontier?**
- Distributed queue (Kafka, RabbitMQ)
- Partition by domain
- Priority-based dequeuing

**Q: How do you respect site politeness?**
- Delay between requests to same domain (1 sec)
- Check robots.txt
- Limit concurrent requests per domain

**Q: How do you scale horizontally?**
- Consistent hashing to assign domains to workers
- Each worker maintains politeness for its domains
- Centralized frontier with distributed workers

This web crawler design demonstrates distributed systems, politeness, and scalability - essential for large-scale web data collection.
