# Back-of-the-Envelope Calculations

## Overview

Back-of-the-envelope (BOTE) calculations are rough estimations used to quickly assess system requirements during interviews. They demonstrate your ability to think quantitatively about scale.

## Why It Matters

In interviews, BOTE calculations help you:
- **Size infrastructure appropriately** - Don't over or under-engineer
- **Identify bottlenecks early** - Spot potential issues
- **Make informed decisions** - Choose technologies based on scale
- **Show technical rigor** - Demonstrate systematic thinking

## Standard Assumptions

### Time-Based Conversions

```
1 day = 24 hours = 86,400 seconds ≈ 100,000 seconds (10^5)
1 month = 30 days ≈ 2.6 million seconds (2.6 × 10^6)
1 year = 365 days ≈ 31.5 million seconds (3 × 10^7)
```

### Traffic Conversions

```
1 million requests/day ≈ 12 requests/second
10 million requests/day ≈ 120 requests/second
100 million requests/day ≈ 1,200 requests/second
1 billion requests/day ≈ 12,000 requests/second

Formula: QPS = (Daily requests) / 100,000
Peak QPS ≈ 2-3 × Average QPS
```

### Storage Multipliers

```
1 KB = 1,000 bytes (10^3)
1 MB = 1,000 KB = 1 million bytes (10^6)
1 GB = 1,000 MB = 1 billion bytes (10^9)
1 TB = 1,000 GB = 1 trillion bytes (10^12)
1 PB = 1,000 TB = 1 quadrillion bytes (10^15)
```

### Data Size Assumptions

```
1 character (ASCII) = 1 byte
1 character (Unicode UTF-8) = 1-4 bytes (avg ~2 bytes)
1 integer (32-bit) = 4 bytes
1 long (64-bit) = 8 bytes
1 timestamp = 8 bytes
1 UUID = 16 bytes
```

### Common Object Sizes

```
Short URL: ~10 bytes
Tweet (280 chars): ~300 bytes
Small image (thumbnail): ~50 KB
Medium image: ~500 KB
High-res photo: ~2-5 MB
1-minute video (720p): ~10-20 MB
1-hour HD video: ~1-2 GB
```

### User Activity Assumptions

```
Daily Active Users (DAU) = 70% of Monthly Active Users (MAU)
Peak users = 2-3 × Average concurrent users
Average session duration = 5-10 minutes
Actions per session = 5-20 (varies by app)
```

## The 4-Step Estimation Framework

### Step 1: Understand the Scale
- How many users?
- What's the read/write ratio?
- What's the data retention period?

### Step 2: Estimate Traffic
- Daily Active Users (DAU)
- Actions per user per day
- Total requests per day
- QPS (average and peak)

### Step 3: Estimate Storage
- Size of each record
- Number of records per day/month/year
- Total storage needed
- Growth rate over time

### Step 4: Estimate Bandwidth
- Average request size
- Average response size
- Total data in/out per second

## Example: URL Shortener

### Given Requirements
- 100 million URLs created per month
- Read/write ratio: 100:1
- URL stored for 5 years
- Average URL length: 100 characters

### Traffic Estimation

```
Write Operations:
- URLs created/month: 100 million
- URLs created/day: 100M / 30 ≈ 3.3 million
- Write QPS: 3.3M / 100,000 ≈ 33 QPS
- Peak write QPS: 33 × 2 ≈ 66 QPS

Read Operations:
- Read/write ratio: 100:1
- Read QPS: 33 × 100 = 3,300 QPS
- Peak read QPS: 3,300 × 2 ≈ 6,600 QPS
```

### Storage Estimation

```
Data per URL:
- Original URL: 100 bytes
- Short URL: 7 bytes
- Metadata (created_at, user_id): 20 bytes
- Total per record: ~130 bytes

Total URLs over 5 years:
- Per month: 100 million
- Per year: 100M × 12 = 1.2 billion
- Over 5 years: 1.2B × 5 = 6 billion URLs

Total storage:
- 6 billion × 130 bytes = 780 GB ≈ 800 GB

With overhead (indexes, replication):
- 800 GB × 3 = 2.4 TB
```

### Bandwidth Estimation

```
Write Bandwidth:
- 33 requests/sec × 130 bytes = 4,290 bytes/sec ≈ 4 KB/s

Read Bandwidth:
- 3,300 requests/sec × 130 bytes = 429,000 bytes/sec ≈ 420 KB/s

Total bandwidth: ~424 KB/s (negligible - no CDN needed)
```

### Memory/Cache Estimation

```
Using 80/20 rule (80% of traffic from 20% of URLs):
- Daily reads: 3,300 QPS × 86,400 = 285 million reads
- Unique URLs (20%): 285M × 0.2 = 57 million URLs
- Cache size: 57M × 130 bytes ≈ 7.4 GB

With safety margin: 10-15 GB of cache
```

## Example: Social Media Feed

### Given Requirements
- 500 million Daily Active Users (DAU)
- Each user views feed 5 times per day
- Each feed shows 20 posts
- Each post: 1 KB (text + metadata)

### Traffic Estimation

```
Feed Views:
- Views per day: 500M × 5 = 2.5 billion views
- Average QPS: 2.5B / 100,000 = 25,000 QPS
- Peak QPS: 25,000 × 3 = 75,000 QPS
```

### Bandwidth Estimation

```
Data per feed view:
- 20 posts × 1 KB = 20 KB per feed

Total outgoing bandwidth:
- 25,000 QPS × 20 KB = 500,000 KB/s = 500 MB/s

Daily bandwidth:
- 2.5B views × 20 KB = 50 TB per day
```

### Storage Estimation (30-day retention)

```
Posts created per day:
- Assume 10% of DAU create posts
- 500M × 0.1 = 50 million posts/day
- Average post size: 1 KB

Storage for 30 days:
- 50M posts/day × 30 days = 1.5 billion posts
- 1.5B × 1 KB = 1.5 TB

With media (images/videos):
- Average media per post: 500 KB
- 1.5B × 500 KB = 750 TB

Total: ~750 TB (use object storage like S3)
```

## Example: Video Streaming Service

### Given Requirements
- 10 million DAU
- Average 30 minutes of video watched per user per day
- Video quality: 720p at 2 Mbps

### Traffic Estimation

```
Concurrent viewers:
- Total watch time: 10M × 30 min = 300M minutes/day
- Per second: 300M min / (24 × 60) = 208,333 minutes/sec
- Concurrent viewers: 208,333 / 60 ≈ 3,500 concurrent streams
- Peak concurrent: 3,500 × 3 = 10,500 streams
```

### Bandwidth Estimation

```
Per stream: 2 Mbps = 0.25 MB/s

Total bandwidth:
- Average: 3,500 × 0.25 MB/s = 875 MB/s ≈ 7 Gbps
- Peak: 10,500 × 0.25 MB/s = 2,625 MB/s ≈ 21 Gbps

Daily data transfer:
- 300M minutes × 60 sec × 0.25 MB = 4.5 PB/day
```

### Storage Estimation

```
Assume 100,000 total videos in library:
- Average video length: 10 minutes
- Size: 10 min × 60 sec × 0.25 MB = 150 MB per video

Total storage:
- 100,000 × 150 MB = 15 TB (original quality)
- With multiple resolutions (360p, 480p, 720p, 1080p): 15 TB × 4 = 60 TB
- With redundancy and backups: 60 TB × 3 = 180 TB
```

## Common Mistakes to Avoid

### 1. Using Exact Numbers
❌ Bad: "86,400 seconds in a day"
✅ Good: "~100,000 seconds in a day"

### 2. Not Accounting for Peak Traffic
❌ Bad: Only calculating average QPS
✅ Good: Peak QPS = 2-3 × Average QPS

### 3. Forgetting Overhead
❌ Bad: Only calculating raw data size
✅ Good: Include indexes, replication, metadata (multiply by 2-3)

### 4. Ignoring Data Growth
❌ Bad: Only calculating current storage
✅ Good: Project growth over 3-5 years

### 5. Not Stating Assumptions
❌ Bad: Jump directly to numbers
✅ Good: "Assuming 70% DAU/MAU ratio..."

## Quick Reference Table

| Metric | Formula | Example |
|--------|---------|---------|
| **QPS from daily requests** | Daily requests / 100,000 | 10M requests/day ÷ 100K = 100 QPS |
| **Peak QPS** | Average QPS × 2-3 | 100 QPS × 2 = 200 peak QPS |
| **Storage over time** | Records/day × Days × Size | 1M/day × 365 × 1KB = 365 GB/year |
| **Bandwidth** | QPS × Data size | 1000 QPS × 10 KB = 10 MB/s |
| **DAU from MAU** | MAU × 0.7 | 100M MAU × 0.7 = 70M DAU |
| **Concurrent users** | DAU × Avg session time / 86,400 | 10M × 600s / 86,400 = 69,444 |

## Interview Tips

### Do's
✅ **Round numbers aggressively** - 86,400 → 100,000
✅ **State assumptions clearly** - "Assuming 100:1 read/write ratio"
✅ **Think out loud** - Explain your reasoning
✅ **Use powers of 10** - Easier to calculate
✅ **Verify reasonableness** - Does 1 PB/day make sense?

### Don'ts
❌ **Don't use a calculator** - Show mental math skills
❌ **Don't aim for precision** - Order of magnitude is enough
❌ **Don't skip estimation** - It shows technical thinking
❌ **Don't forget units** - Always specify KB/MB/GB/TB

## Practice Problems

### Problem 1: Instagram-like Photo Sharing
```
Given:
- 100M DAU
- Each user uploads 2 photos/day
- Average photo size: 2 MB
- Users view 50 photos/day

Calculate:
1. Upload QPS
2. View QPS
3. Storage needed for 3 years
4. Daily bandwidth
```

<details>
<summary>Solution</summary>

```
Upload Traffic:
- Photos/day: 100M × 2 = 200M photos
- Upload QPS: 200M / 100,000 = 2,000 QPS

View Traffic:
- Views/day: 100M × 50 = 5B views
- View QPS: 5B / 100,000 = 50,000 QPS

Storage (3 years):
- Photos/year: 200M × 365 = 73B photos
- 3 years: 73B × 3 = 219B photos
- Storage: 219B × 2 MB = 438 TB ≈ 440 TB

Bandwidth:
- Upload: 2,000 QPS × 2 MB = 4 GB/s
- Download: 50,000 QPS × 2 MB = 100 GB/s
```
</details>

## Summary

BOTE calculations are about:
1. **Making reasonable assumptions**
2. **Using simple math** (powers of 10, rounding)
3. **Showing systematic thinking**
4. **Arriving at order-of-magnitude estimates**

**Remember:** In interviews, the process matters more than the exact answer. Show your work, state assumptions, and explain your reasoning.

## Next Steps

- Practice [QPS Estimation](qps-estimation.md)
- Learn [Storage Estimation](storage-estimation.md)
- Study [Capacity Planning](capacity-planning.md)

---

**Pro Tip:** Practice BOTE calculations for every system you design. It becomes second nature after 10-15 practice sessions.
