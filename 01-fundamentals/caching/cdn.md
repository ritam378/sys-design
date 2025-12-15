# Content Delivery Network (CDN)

## Overview

A **Content Delivery Network (CDN)** is a geographically distributed network of servers that caches and serves content to users from the nearest location, reducing latency and improving performance.

**Key Principle:** Bring content closer to users.

**Typical Latency:**
- Same city: 10-20ms
- Same country: 20-50ms
- Same continent: 50-100ms
- Different continent: 100-300ms

**With CDN:**
- Users served from nearest edge location: 10-30ms globally

---

## How CDN Works

### Architecture

```
User (New York) → CDN Edge (New York) → Origin Server (California)
                      ↓ (cached)
                  Returns content
                  (20ms instead of 150ms)

User (London) → CDN Edge (London) → Origin Server (California)
                   ↓ (cached)
               Returns content
               (15ms instead of 200ms)
```

### Request Flow

**First Request (Cache Miss):**
```
1. User requests example.com/image.jpg
2. DNS resolves to nearest CDN edge
3. Edge doesn't have content (cache miss)
4. Edge fetches from origin server
5. Edge caches content
6. Edge returns to user
```

**Subsequent Requests (Cache Hit):**
```
1. User requests example.com/image.jpg
2. DNS resolves to nearest CDN edge
3. Edge has content (cache hit)
4. Edge returns immediately (no origin request)
```

**Visual:**
```mermaid
graph LR
    User[User] --> DNS[DNS]
    DNS --> Edge[CDN Edge<br/>Closest Location]
    Edge --> |Cache Miss| Origin[Origin Server]
    Edge --> |Cache Hit| User
    Origin --> Edge
```

---

## Key Features

### 1. Geographic Distribution

**Edge Locations (PoPs - Points of Presence):**
```
Cloudflare: 300+ cities worldwide
AWS CloudFront: 450+ edge locations
Akamai: 4,000+ PoPs
Fastly: 70+ PoPs
```

**Example Routing:**
```python
# User in Tokyo
request from tokyo.example.com
    ↓
Routed to Tokyo edge server (10ms)
Instead of California origin (150ms)
```

### 2. Caching

**What CDN Caches:**
- Static files (images, CSS, JavaScript, fonts)
- Videos and media files
- API responses (with headers)
- HTML pages (configurable)
- Downloads (PDFs, executables)

**Cache Headers:**
```http
# Response from origin
HTTP/1.1 200 OK
Cache-Control: public, max-age=86400
Expires: Wed, 21 Oct 2024 07:28:00 GMT
ETag: "abc123"
Content-Type: image/jpeg

# CDN caches for 24 hours
```

**Cache-Control Directives:**
```http
Cache-Control: public, max-age=3600           # Cache for 1 hour
Cache-Control: private, no-cache              # Don't cache (user-specific)
Cache-Control: public, max-age=31536000       # Cache for 1 year (static assets)
Cache-Control: no-store                       # Never cache (sensitive data)
Cache-Control: s-maxage=3600, max-age=600     # CDN: 1h, Browser: 10min
```

### 3. Load Balancing

CDN distributes traffic across multiple servers:

```
1 million requests
    ↓
Distributed across 100 edge servers
    ↓
Each handles 10,000 requests
(Origin server only handles cache misses)
```

### 4. DDoS Protection

**Without CDN:**
```
Attacker → 1 million req/sec → Origin Server (crashes)
```

**With CDN:**
```
Attacker → 1 million req/sec → CDN (absorbs attack)
    ↓
CDN filters malicious traffic
    ↓
Only legitimate requests reach origin
```

### 5. SSL/TLS Termination

```
User ← HTTPS → CDN Edge ← HTTP/HTTPS → Origin
       (encrypted)         (can be unencrypted)
```

**Benefits:**
- Offload SSL processing from origin
- Reduce latency (SSL handshake at edge)
- Free SSL certificates (Let's Encrypt)

---

## CDN Configuration

### Static Asset Optimization

```html
<!-- Before CDN -->
<img src="https://example.com/images/logo.png">
<link rel="stylesheet" href="https://example.com/css/styles.css">
<script src="https://example.com/js/app.js"></script>

<!-- After CDN -->
<img src="https://cdn.example.com/images/logo.png">
<link rel="stylesheet" href="https://cdn.example.com/css/styles.css">
<script src="https://cdn.example.com/js/app.js"></script>

<!-- Or use CDN subdomain -->
<img src="https://d1a2b3c4.cloudfront.net/images/logo.png">
```

### Cache-Control Headers

```python
# Flask example
from flask import Flask, send_file, make_response

app = Flask(__name__)

@app.route('/static/<path:filename>')
def serve_static(filename):
    response = make_response(send_file(f'static/{filename}'))

    # Cache for 1 year (immutable assets)
    if filename.endswith(('.jpg', '.png', '.css', '.js')):
        response.headers['Cache-Control'] = 'public, max-age=31536000, immutable'

    # Cache for 1 hour (dynamic content)
    elif filename.endswith('.json'):
        response.headers['Cache-Control'] = 'public, max-age=3600'

    return response
```

### Cache Busting / Versioning

**Problem:** Users have old cached version after update.

**Solution 1: Query String Versioning**
```html
<!-- Old -->
<link rel="stylesheet" href="/css/styles.css?v=1">

<!-- Updated -->
<link rel="stylesheet" href="/css/styles.css?v=2">
```

**Solution 2: Filename Hashing (Best Practice)**
```html
<!-- Webpack/build tools generate hashed filenames -->
<link rel="stylesheet" href="/css/styles.a1b2c3d4.css">
<script src="/js/app.e5f6g7h8.js"></script>

<!-- When files change, hash changes, CDN fetches new version -->
```

**Solution 3: Path Versioning**
```html
<link rel="stylesheet" href="/v2/css/styles.css">
```

---

## CDN Providers Comparison

| Provider | Edge Locations | Free Tier | Key Features | Best For |
|----------|----------------|-----------|--------------|----------|
| **Cloudflare** | 300+ | ✅ Generous | Security, DDoS protection | General purpose |
| **AWS CloudFront** | 450+ | ✅ 1TB/month | AWS integration | AWS users |
| **Fastly** | 70+ | ❌ Paid | Real-time purge, edge compute | High control |
| **Akamai** | 4,000+ | ❌ Paid | Enterprise, largest network | Large enterprises |
| **Bunny CDN** | 114 | ✅ Pay-as-go | Affordable, simple | Startups |
| **Vercel** | Global | ✅ Yes | Next.js optimized | Web apps |

---

## Common Patterns

### 1. Static Website Hosting

```
S3/Storage → CloudFront → Users
              ↓
        HTTPS, custom domain
        Cache static files
```

**Setup:**
```bash
# AWS example
aws s3 mb s3://my-website
aws s3 sync ./build s3://my-website
aws cloudfront create-distribution \
  --origin-domain-name my-website.s3.amazonaws.com
```

### 2. API Response Caching

```python
# Django REST Framework
from rest_framework.decorators import api_view
from django.views.decorators.cache import cache_page

@api_view(['GET'])
@cache_page(60 * 15)  # Cache for 15 minutes
def get_products(request):
    products = Product.objects.all()
    return Response(ProductSerializer(products, many=True).data)
```

**Response Headers:**
```http
GET /api/products HTTP/1.1

HTTP/1.1 200 OK
Cache-Control: public, max-age=900
Vary: Accept-Encoding
X-Cache: HIT from cloudfront
```

### 3. Dynamic Content Acceleration

```
User → CDN Edge
        ↓
    Optimized route to origin (not public internet)
        ↓
    Origin Server
```

**Benefits:**
- Connection pooling
- Route optimization
- TCP optimization
- Compression

### 4. Video Streaming (Adaptive Bitrate)

```
Video file → Transcoded to multiple bitrates
              ↓
          HLS/DASH segments
              ↓
          Cached at CDN edge
              ↓
          Delivered to users
```

**Example HLS Manifest:**
```m3u8
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
https://cdn.example.com/video/360p.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1400000,RESOLUTION=842x480
https://cdn.example.com/video/480p.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2800000,RESOLUTION=1280x720
https://cdn.example.com/video/720p.m3u8
```

---

## Cache Invalidation

### 1. TTL Expiration
Wait for cache to expire naturally.

```http
Cache-Control: max-age=3600  # Expires in 1 hour
```

**Pros:** Simple, automatic
**Cons:** Slow, can't force immediate update

### 2. Manual Purge/Invalidation

**Cloudflare:**
```bash
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Authorization: Bearer {api_token}" \
  -d '{"files":["https://example.com/image.jpg"]}'
```

**AWS CloudFront:**
```python
import boto3

cloudfront = boto3.client('cloudfront')

cloudfront.create_invalidation(
    DistributionId='E1234567890ABC',
    InvalidationBatch={
        'Paths': {
            'Quantity': 2,
            'Items': [
                '/images/logo.png',
                '/css/*'  # Wildcard
            ]
        },
        'CallerReference': str(time.time())
    }
)
```

**Pros:** Immediate
**Cons:** Costs money (CloudFront: $0.005 per path), impacts performance

### 3. Versioned URLs (Best Practice)

```html
<!-- Old -->
<link href="/css/styles.css">

<!-- New version (different URL, no purge needed) -->
<link href="/css/styles.v2.css">
```

**Pros:** No purge cost, instant updates, can rollback
**Cons:** Requires build process

---

## CDN Performance Optimization

### 1. Image Optimization

**On-the-fly transformation:**
```html
<!-- Cloudflare Image Resizing -->
<img src="https://example.com/cdn-cgi/image/width=800,quality=85/photo.jpg">

<!-- Cloudinary -->
<img src="https://res.cloudinary.com/demo/image/upload/w_800,q_auto/photo.jpg">

<!-- imgix -->
<img src="https://demo.imgix.net/photo.jpg?w=800&auto=compress">
```

### 2. Compression

```http
# Response headers
Content-Encoding: gzip
Content-Type: text/html; charset=utf-8
Vary: Accept-Encoding
```

**Brotli vs Gzip:**
```
Original:     1000 KB
Gzip:         250 KB (75% reduction)
Brotli:       200 KB (80% reduction, slower)
```

### 3. HTTP/2 and HTTP/3

**HTTP/2 Benefits:**
- Multiplexing (multiple requests on one connection)
- Header compression
- Server push

**HTTP/3 Benefits:**
- QUIC protocol (UDP-based)
- Faster handshake
- Better on poor networks

### 4. Edge Computing

Run code at CDN edge (serverless functions).

**Cloudflare Workers:**
```javascript
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  // Run at edge, modify response
  const response = await fetch(request)
  const newHeaders = new Headers(response.headers)
  newHeaders.set('X-Custom-Header', 'Added at edge')

  return new Response(response.body, {
    status: response.status,
    headers: newHeaders
  })
}
```

**Lambda@Edge (AWS):**
```python
def lambda_handler(event, context):
    request = event['Records'][0]['cf']['request']

    # Redirect mobile users
    if 'Mobile' in request['headers'].get('user-agent', [{}])[0].get('value', ''):
        return {
            'status': '302',
            'headers': {
                'location': [{
                    'key': 'Location',
                    'value': 'https://m.example.com'
                }]
            }
        }

    return request
```

---

## CDN Security

### 1. Signed URLs

Prevent hotlinking and unauthorized access.

```python
import hashlib
import time

def generate_signed_url(url, secret, expiration=3600):
    expires = int(time.time()) + expiration
    token = hashlib.md5(f"{secret}{url}{expires}".encode()).hexdigest()
    return f"{url}?token={token}&expires={expires}"

# Usage
signed_url = generate_signed_url(
    "https://cdn.example.com/video.mp4",
    "my-secret-key",
    expiration=3600  # 1 hour
)
# https://cdn.example.com/video.mp4?token=abc123&expires=1705334400
```

**Cloudfront Signed URLs:**
```python
import boto3
from datetime import datetime, timedelta

cloudfront_signer = boto3.client('cloudfront').get_signer(
    key_pair_id='APKAXXXXXXXXXX',
    private_key='-----BEGIN RSA PRIVATE KEY-----...'
)

url = cloudfront_signer.generate_presigned_url(
    'https://d1234.cloudfront.net/private/video.mp4',
    date_less_than=datetime.now() + timedelta(hours=1)
)
```

### 2. Geo-Blocking

```javascript
// Cloudflare Worker
addEventListener('fetch', event => {
  const country = event.request.headers.get('CF-IPCountry')

  if (['CN', 'RU', 'KP'].includes(country)) {
    event.respondWith(new Response('Access denied', { status: 403 }))
  } else {
    event.respondWith(fetch(event.request))
  }
})
```

### 3. Rate Limiting at Edge

```javascript
// Cloudflare rate limiting
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  const ip = request.headers.get('CF-Connecting-IP')

  // Check rate limit
  const count = await incrementCounter(ip)

  if (count > 100) {  // 100 requests per minute
    return new Response('Rate limit exceeded', { status: 429 })
  }

  return fetch(request)
}
```

---

## Real-World Examples

### Netflix
- **CDN:** Custom CDN (Open Connect) + 3rd party
- **Content:** Streams from nearest edge location
- **Scale:** Petabytes of video cached globally

### Facebook
- **CDN:** Custom CDN + Akamai
- **Content:** Images, videos, static assets
- **Optimization:** Adaptive image compression

### Spotify
- **CDN:** Google Cloud CDN + Fastly
- **Content:** Audio streams, album art
- **Optimization:** Audio compression at edge

### GitHub
- **CDN:** Fastly
- **Content:** Git repos, static pages
- **Feature:** Instant cache purge on push

---

## Cost Optimization

### 1. Origin Shield

```
Users → CDN Edge → Origin Shield → Origin
                      ↓ (reduces origin requests)
                   Cache layer
```

**Benefits:**
- Reduce origin load
- Improve cache hit ratio
- Lower egress costs

### 2. Compression

```
Transfer 1TB uncompressed: $85 (AWS CloudFront)
Transfer 250GB compressed: $21 (75% savings)
```

### 3. Smart Caching

```python
# Cache expensive operations longer
@app.route('/api/analytics')
def analytics():
    response = expensive_query()
    response.headers['Cache-Control'] = 'public, max-age=86400'  # 24 hours
    return response

# Cache user-specific data shorter
@app.route('/api/profile')
def profile():
    response = get_user_profile()
    response.headers['Cache-Control'] = 'private, max-age=300'  # 5 minutes
    return response
```

---

## Monitoring

### Key Metrics

```python
# Track CDN performance
metrics = {
    'cache_hit_ratio': 0.85,      # Target: > 80%
    'avg_latency_ms': 25,          # Target: < 100ms
    'bandwidth_saved': '75%',      # vs origin
    'origin_requests': 15,         # per 100 user requests
    'error_rate': 0.01,            # Target: < 1%
}
```

### Alerts

```
Alert: Cache hit ratio < 70%
Action: Investigate cache headers, TTL settings

Alert: High error rate (5xx) from origin
Action: Check origin server health

Alert: High latency from specific region
Action: Check edge location health
```

---

## When to Use CDN

### ✅ Use CDN When:
1. Global user base
2. Serving static content (images, videos, CSS, JS)
3. High traffic website
4. Need DDoS protection
5. Want to reduce origin load
6. Latency-sensitive application

### ❌ Don't Need CDN When:
1. Local-only users (single city/region)
2. Fully dynamic content (user-specific)
3. Very low traffic
4. Real-time collaboration (WebSockets)
5. Large file uploads

---

## Best Practices

### 1. Set Appropriate Cache Headers
```http
# Static assets (images, fonts)
Cache-Control: public, max-age=31536000, immutable

# HTML pages
Cache-Control: public, max-age=3600, must-revalidate

# API responses
Cache-Control: public, max-age=300, s-maxage=3600

# User-specific data
Cache-Control: private, max-age=0, no-cache
```

### 2. Use Versioned URLs
```html
<!-- Include hash in filename -->
<script src="/js/app.a1b2c3.js"></script>
```

### 3. Monitor Cache Performance
```python
# Log cache status
@app.after_request
def log_cache_status(response):
    cache_status = request.headers.get('X-Cache', 'UNKNOWN')
    logger.info(f"Cache status: {cache_status}")
    return response
```

### 4. Implement Fallbacks
```javascript
// If CDN fails, fallback to origin
<script src="https://cdn.example.com/lib.js"
        onerror="this.onerror=null; this.src='https://origin.example.com/lib.js'">
</script>
```

---

## Summary

**CDN Benefits:**
- ✅ Reduced latency (50-90%)
- ✅ Lower origin load (70-95%)
- ✅ DDoS protection
- ✅ Better availability
- ✅ Cost savings (bandwidth)

**CDN Trade-offs:**
- ❌ Eventual consistency (cache invalidation lag)
- ❌ Additional cost (though often offset by savings)
- ❌ Configuration complexity
- ❌ Debugging challenges

**Key Concepts:**
- Edge locations bring content close to users
- Cache-Control headers determine caching behavior
- Versioned URLs enable instant updates
- Edge computing enables serverless at edge

**Interview Tips:**
- Explain cache hit ratio and optimization
- Discuss cache invalidation strategies
- Mention specific CDN providers and use cases
- Cover security (signed URLs, geo-blocking)

---

**Next:** Learn about [Redis vs Memcached](redis-vs-memcached.md) for in-memory caching.
