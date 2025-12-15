# Domain Name System (DNS)

## Table of Contents
- [What is DNS?](#what-is-dns)
- [DNS Hierarchy](#dns-hierarchy)
- [DNS Record Types](#dns-record-types)
- [DNS Resolution Process](#dns-resolution-process)
- [DNS Caching](#dns-caching)
- [DNS Providers and Services](#dns-providers-and-services)
- [Advanced DNS Features](#advanced-dns-features)
- [DNS Security](#dns-security)
- [Performance Optimization](#performance-optimization)
- [Common Issues and Troubleshooting](#common-issues-and-troubleshooting)
- [Interview Tips](#interview-tips)

---

## What is DNS?

The **Domain Name System (DNS)** is often called the "phonebook of the internet." It translates human-readable domain names (like `www.example.com`) into IP addresses (like `192.0.2.1`) that computers use to identify each other on the network.

### Why DNS is Important

```
Without DNS:
User types: http://172.217.160.142
Hard to remember, not user-friendly

With DNS:
User types: http://www.google.com
Easy to remember, brandable
```

### Key Benefits

1. **Human-Friendly**: Easy-to-remember domain names instead of IP addresses
2. **Flexibility**: Change server IPs without changing domain names
3. **Load Distribution**: Route traffic to multiple servers
4. **Redundancy**: Multiple DNS servers for high availability
5. **Geographic Routing**: Direct users to nearest servers

---

## DNS Hierarchy

DNS follows a hierarchical, distributed database structure:

```
                         Root (.)
                            |
        +-------------------+-------------------+
        |                   |                   |
      .com                .org                .net
        |                   |                   |
    example.com         wikipedia.org       cloudflare.net
        |
    +---+---+
    |       |
  www     api
```

### Levels in DNS Hierarchy

#### 1. Root Level (.)
- **13 root server systems** (A-M) distributed globally
- Maintained by different organizations (ICANN, Verisign, NASA, etc.)
- Root servers don't store domain mappings, but know where to find TLD servers
- Anycast routing: Multiple physical servers share same IP

#### 2. Top-Level Domains (TLD)
- **Generic TLDs**: `.com`, `.org`, `.net`, `.info`
- **Country Code TLDs**: `.us`, `.uk`, `.jp`, `.in`
- **New gTLDs**: `.app`, `.dev`, `.cloud`, `.tech`

#### 3. Second-Level Domains
- `example` in `example.com`
- Registered by individuals/organizations through registrars

#### 4. Subdomains
- `www` in `www.example.com`
- `api` in `api.example.com`
- `mail` in `mail.example.com`

---

## DNS Record Types

DNS records store different types of information about a domain.

### Essential Record Types

#### A Record (Address)
Maps domain name to IPv4 address.

```
example.com.  3600  IN  A  192.0.2.1
```

**Example Configuration:**
```bash
# Zone file entry
www.example.com.    IN    A    192.0.2.1
example.com.        IN    A    192.0.2.1
```

#### AAAA Record
Maps domain name to IPv6 address.

```
example.com.  3600  IN  AAAA  2001:0db8:85a3::8a2e:0370:7334
```

#### CNAME Record (Canonical Name)
Creates an alias for another domain name.

```
www.example.com.  3600  IN  CNAME  example.com.
blog.example.com. 3600  IN  CNAME  hosting.platform.com.
```

**Use Cases:**
- Point multiple subdomains to the same destination
- Use third-party services (e.g., CDN)

**Important**: CNAME cannot coexist with other records for the same name.

#### MX Record (Mail Exchange)
Specifies mail servers for the domain.

```
example.com.  3600  IN  MX  10  mail1.example.com.
example.com.  3600  IN  MX  20  mail2.example.com.
```

**Priority**: Lower number = higher priority

#### TXT Record
Stores arbitrary text, often used for verification and security.

```
example.com.  3600  IN  TXT  "v=spf1 include:_spf.google.com ~all"
```

**Common Uses:**
- **SPF (Sender Policy Framework)**: Email authentication
- **DKIM (DomainKeys Identified Mail)**: Email signing
- **Domain verification**: Prove domain ownership
- **DMARC**: Email authentication policy

#### NS Record (Name Server)
Specifies authoritative name servers for the domain.

```
example.com.  3600  IN  NS  ns1.example.com.
example.com.  3600  IN  NS  ns2.example.com.
```

#### SOA Record (Start of Authority)
Contains administrative information about the zone.

```
example.com.  3600  IN  SOA  ns1.example.com. admin.example.com. (
                            2024121501 ; Serial
                            3600       ; Refresh
                            1800       ; Retry
                            604800     ; Expire
                            86400 )    ; Minimum TTL
```

#### PTR Record (Pointer)
Reverse DNS lookup - maps IP to domain name.

```
1.2.0.192.in-addr.arpa.  3600  IN  PTR  example.com.
```

#### SRV Record (Service)
Specifies location of services.

```
_http._tcp.example.com.  3600  IN  SRV  10  60  80  server.example.com.
```

**Format**: `priority weight port target`

#### CAA Record (Certification Authority Authorization)
Specifies which CAs can issue certificates.

```
example.com.  3600  IN  CAA  0  issue  "letsencrypt.org"
```

---

## DNS Resolution Process

### Two Types of DNS Queries

#### 1. Recursive Query
DNS resolver must return a definitive answer (IP or error).

#### 2. Iterative Query
DNS server returns the best answer it has or a referral to another server.

### Complete DNS Resolution Flow

```
User enters: www.example.com

1. Browser Cache
   ↓ (miss)
2. OS Cache
   ↓ (miss)
3. Recursive DNS Resolver (ISP or 8.8.8.8)
   ↓
4. Root Name Server (.)
   → Returns: "Ask .com TLD server"
   ↓
5. TLD Name Server (.com)
   → Returns: "Ask example.com's authoritative server"
   ↓
6. Authoritative Name Server
   → Returns: 192.0.2.1
   ↓
7. Resolver caches result
   ↓
8. Returns IP to client
   ↓
9. Browser connects to 192.0.2.1
```

### Detailed Example

**Step-by-step resolution for `www.example.com`:**

```python
# Pseudo-code for DNS resolution

def resolve_dns(domain):
    # 1. Check browser cache
    if domain in browser_cache:
        return browser_cache[domain]

    # 2. Check OS cache
    if domain in os_cache:
        return os_cache[domain]

    # 3. Query recursive resolver
    resolver = "8.8.8.8"  # Google Public DNS

    # 4. Resolver queries root server
    root_response = query(".", domain)
    # Response: "Ask .com TLD at 192.5.6.30"

    # 5. Resolver queries TLD server
    tld_response = query("192.5.6.30", domain)
    # Response: "Ask ns1.example.com at 192.0.2.2"

    # 6. Resolver queries authoritative server
    auth_response = query("192.0.2.2", domain)
    # Response: "www.example.com is at 192.0.2.1"

    # 7. Cache and return
    cache_result(domain, "192.0.2.1", ttl=3600)
    return "192.0.2.1"
```

### DNS Query Tools

```bash
# Basic DNS lookup
nslookup www.example.com

# Detailed DNS information
dig www.example.com

# Trace full DNS resolution path
dig +trace www.example.com

# Query specific record type
dig example.com MX
dig example.com TXT

# Query specific DNS server
dig @8.8.8.8 www.example.com

# Reverse DNS lookup
dig -x 192.0.2.1
```

---

## DNS Caching

### Cache Levels

```
[Browser Cache] → 5 minutes typical
        ↓
[OS Cache] → Varies by OS
        ↓
[Recursive Resolver Cache] → Based on TTL
        ↓
[Authoritative Server] → Source of truth
```

### Time To Live (TTL)

TTL determines how long a DNS record can be cached.

```
example.com.  3600  IN  A  192.0.2.1
              ^^^^
              TTL in seconds (1 hour)
```

**TTL Trade-offs:**

| TTL Value | Pros | Cons |
|-----------|------|------|
| **Short (60-300s)** | Fast changes, quick failover | More DNS queries, higher load |
| **Medium (3600s)** | Balanced performance | Moderate propagation time |
| **Long (86400s)** | Fewer DNS queries, better performance | Slow changes, difficult rollback |

**Best Practices:**
- **Normal operation**: Use longer TTL (1-24 hours)
- **Before migration**: Lower TTL to 60-300 seconds
- **After migration**: Restore longer TTL

### Cache Example

```python
import time
from datetime import datetime, timedelta

class DNSCache:
    def __init__(self):
        self.cache = {}

    def get(self, domain):
        if domain in self.cache:
            entry = self.cache[domain]
            if datetime.now() < entry['expires']:
                print(f"Cache HIT: {domain}")
                return entry['ip']
            else:
                print(f"Cache EXPIRED: {domain}")
                del self.cache[domain]

        print(f"Cache MISS: {domain}")
        return None

    def set(self, domain, ip, ttl):
        self.cache[domain] = {
            'ip': ip,
            'expires': datetime.now() + timedelta(seconds=ttl)
        }
        print(f"Cached {domain} → {ip} for {ttl}s")

# Usage
cache = DNSCache()
cache.set("example.com", "192.0.2.1", ttl=3600)
print(cache.get("example.com"))  # Cache HIT
```

---

## DNS Providers and Services

### Major DNS Providers

#### 1. Amazon Route 53
**Features:**
- 100% availability SLA
- Low latency
- Integration with AWS services
- Health checks and failover
- Traffic flow policies

**Pricing:**
- $0.50 per hosted zone/month
- $0.40 per million queries

```python
# Route 53 example (boto3)
import boto3

client = boto3.client('route53')

# Create hosted zone
response = client.create_hosted_zone(
    Name='example.com',
    CallerReference=str(hash(datetime.now())),
)

# Create A record
client.change_resource_record_sets(
    HostedZoneId='Z123456789',
    ChangeBatch={
        'Changes': [{
            'Action': 'CREATE',
            'ResourceRecordSet': {
                'Name': 'www.example.com',
                'Type': 'A',
                'TTL': 300,
                'ResourceRecords': [{'Value': '192.0.2.1'}]
            }
        }]
    }
)
```

#### 2. Cloudflare DNS
**Features:**
- Free tier available
- Fastest DNS resolver (1.1.1.1)
- Built-in DDoS protection
- DNSSEC support
- DNS analytics

**Advantages:**
- Global Anycast network
- Free SSL/TLS
- CDN integration

#### 3. Google Cloud DNS
**Features:**
- 100% uptime SLA
- Low latency using Google's network
- DNSSEC support
- Private DNS zones

#### 4. Public DNS Resolvers

| Provider | Primary | Secondary | Features |
|----------|---------|-----------|----------|
| **Google** | 8.8.8.8 | 8.8.4.4 | Fast, reliable |
| **Cloudflare** | 1.1.1.1 | 1.0.0.1 | Privacy-focused, fastest |
| **Quad9** | 9.9.9.9 | 149.112.112.112 | Security filtering |
| **OpenDNS** | 208.67.222.222 | 208.67.220.220 | Content filtering |

---

## Advanced DNS Features

### 1. GeoDNS (Geographic Load Balancing)

Route users to nearest server based on geographic location.

```
User in US → 192.0.2.1 (US server)
User in EU → 203.0.113.1 (EU server)
User in Asia → 198.51.100.1 (Asia server)
```

**Benefits:**
- Reduced latency
- Compliance with data residency laws
- Better user experience

**Route 53 Geolocation Example:**
```json
{
  "Name": "www.example.com",
  "Type": "A",
  "SetIdentifier": "US-East",
  "GeoLocation": {
    "ContinentCode": "NA"
  },
  "TTL": 60,
  "ResourceRecords": [{"Value": "192.0.2.1"}]
}
```

### 2. DNS-based Load Balancing

Distribute traffic across multiple servers.

```
# Multiple A records for round-robin
www.example.com.  300  IN  A  192.0.2.1
www.example.com.  300  IN  A  192.0.2.2
www.example.com.  300  IN  A  192.0.2.3
```

**Limitations:**
- No health checking (use Route 53 health checks)
- Client-side caching reduces effectiveness
- Uneven distribution

### 3. DNS Failover

Automatically route traffic to healthy servers.

```python
# Route 53 Health Check
client.create_health_check(
    Type='HTTPS',
    ResourcePath='/',
    FullyQualifiedDomainName='www.example.com',
    Port=443,
    RequestInterval=30,
    FailureThreshold=3
)

# Failover routing policy
{
  "Name": "www.example.com",
  "Type": "A",
  "SetIdentifier": "Primary",
  "Failover": "PRIMARY",
  "HealthCheckId": "abc123",
  "ResourceRecords": [{"Value": "192.0.2.1"}]
}
```

### 4. Weighted Routing

Control percentage of traffic to each endpoint.

```json
[
  {
    "SetIdentifier": "Server-1",
    "Weight": 70,
    "ResourceRecords": [{"Value": "192.0.2.1"}]
  },
  {
    "SetIdentifier": "Server-2",
    "Weight": 30,
    "ResourceRecords": [{"Value": "192.0.2.2"}]
  }
]
```

**Use Cases:**
- A/B testing
- Blue-green deployments
- Gradual rollouts

### 5. Latency-based Routing

Route users to lowest-latency endpoint.

```json
{
  "SetIdentifier": "us-east-1",
  "Region": "us-east-1",
  "ResourceRecords": [{"Value": "192.0.2.1"}]
}
```

---

## DNS Security

### 1. DNSSEC (DNS Security Extensions)

Adds cryptographic signatures to DNS records to prevent tampering.

```
Zone signing:
1. Generate key pairs (ZSK, KSK)
2. Sign DNS records with private key
3. Publish public key in DNS
4. Validators verify signatures
```

**Benefits:**
- Prevents DNS spoofing
- Ensures data integrity
- Authenticates origin

**Limitations:**
- Increased DNS response size
- More complex management
- Not widely adopted

### 2. DNS over HTTPS (DoH)

Encrypts DNS queries using HTTPS protocol.

```python
import requests

# DoH query using Cloudflare
def dns_over_https(domain):
    url = f"https://1.1.1.1/dns-query?name={domain}&type=A"
    headers = {"accept": "application/dns-json"}
    response = requests.get(url, headers=headers)
    return response.json()

result = dns_over_https("example.com")
print(result['Answer'][0]['data'])  # IP address
```

### 3. DNS over TLS (DoT)

Encrypts DNS queries using TLS on port 853.

```bash
# Configure DNS over TLS
# /etc/systemd/resolved.conf
[Resolve]
DNS=1.1.1.1 1.0.0.1
DNSOverTLS=yes
```

### 4. Common DNS Attacks

#### DNS Spoofing/Cache Poisoning
Attacker injects false DNS records into cache.

**Mitigation:**
- Use DNSSEC
- Randomize source ports
- Use DNS over HTTPS/TLS

#### DNS Amplification DDoS
Attacker uses DNS servers to amplify attack traffic.

**Mitigation:**
- Rate limiting
- Response rate limiting (RRL)
- Disable recursive queries on authoritative servers

#### DNS Tunneling
Encapsulate data in DNS queries to bypass firewalls.

**Mitigation:**
- Monitor DNS query patterns
- Analyze query lengths
- Use DNS security products

---

## Performance Optimization

### 1. DNS Prefetching

```html
<!-- Browser prefetch DNS for external resources -->
<link rel="dns-prefetch" href="//cdn.example.com">
<link rel="dns-prefetch" href="//api.example.com">
```

**Benefits:**
- Reduces latency for future requests
- Parallel DNS resolution

### 2. Reduce DNS Lookups

```
Bad: Multiple domains
<img src="http://cdn1.example.com/logo.png">
<img src="http://cdn2.example.com/banner.png">
<script src="http://js.example.com/app.js"></script>

Good: Fewer domains
<img src="http://cdn.example.com/logo.png">
<img src="http://cdn.example.com/banner.png">
<script src="http://cdn.example.com/app.js"></script>
```

### 3. Optimize TTL Values

```python
# Dynamic TTL based on traffic patterns
def calculate_ttl(domain):
    if is_high_traffic(domain):
        return 86400  # 24 hours - reduce DNS load
    elif is_frequent_changes(domain):
        return 300    # 5 minutes - faster updates
    else:
        return 3600   # 1 hour - balanced
```

### 4. Use Anycast DNS

Multiple servers share same IP address, user routed to nearest.

```
User in New York → DNS server in New York
User in London → DNS server in London
User in Tokyo → DNS server in Tokyo
```

**Benefits:**
- Lower latency
- DDoS mitigation
- High availability

---

## Common Issues and Troubleshooting

### Issue 1: DNS Propagation Delay

**Problem:** Changes not visible globally.

**Solution:**
```bash
# Check DNS propagation
dig @8.8.8.8 example.com  # Google DNS
dig @1.1.1.1 example.com  # Cloudflare DNS
dig @8.8.4.4 example.com  # Google DNS secondary

# Clear local DNS cache
# macOS
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder

# Windows
ipconfig /flushdns

# Linux
sudo systemd-resolve --flush-caches
```

### Issue 2: NXDOMAIN (Domain Not Found)

**Debugging:**
```bash
# Check if domain exists
dig example.com

# Check NS records
dig example.com NS

# Query authoritative server directly
dig @ns1.example.com example.com
```

### Issue 3: Slow DNS Resolution

**Diagnosis:**
```bash
# Measure DNS query time
dig example.com | grep "Query time"

# Test different resolvers
dig @8.8.8.8 example.com | grep "Query time"
dig @1.1.1.1 example.com | grep "Query time"
```

**Solutions:**
- Use faster DNS resolver (1.1.1.1, 8.8.8.8)
- Increase TTL values
- Implement DNS caching

### Issue 4: DNS Loops

**Problem:** CNAME points to another CNAME that points back.

```
# Bad configuration
www.example.com  CNAME  alias.example.com
alias.example.com CNAME  www.example.com  # LOOP!
```

---

## Interview Tips

### Common DNS Interview Questions

#### Q1: Explain DNS resolution process step-by-step
**Answer:**
1. Browser checks cache
2. OS checks cache
3. Query recursive resolver
4. Resolver queries root server (gets TLD server)
5. Resolver queries TLD server (gets authoritative server)
6. Resolver queries authoritative server (gets IP)
7. Result cached and returned

#### Q2: What's the difference between A and CNAME records?
**Answer:**
- **A record**: Maps domain directly to IP address
- **CNAME**: Creates alias to another domain name
- CNAME cannot coexist with other records for same name
- Root domain cannot be CNAME

#### Q3: How would you implement global load balancing?
**Answer:**
Use GeoDNS or latency-based routing:
- Configure DNS records for each region
- Route users to nearest/fastest endpoint
- Implement health checks for failover
- Use low TTL for quick failover

#### Q4: What is DNS cache poisoning and how to prevent it?
**Answer:**
- Attacker injects false DNS records
- Prevention: DNSSEC, random source ports, DNS over HTTPS/TLS
- Use authoritative DNS providers with security features

### System Design Considerations

When designing systems involving DNS:

1. **Low TTL during migrations**: Use 60-300s for quick changes
2. **Health checks**: Implement automated failover
3. **Multiple DNS providers**: Avoid single point of failure
4. **Geographic distribution**: Route users to nearest servers
5. **Monitoring**: Track DNS query performance and errors
6. **Security**: Implement DNSSEC, DoH, or DoT
7. **Cost optimization**: Balance query costs with TTL values

### Real-World Examples

**Netflix:**
- Uses Route 53 with latency-based routing
- Dynamic DNS updates based on server health
- Multiple layers of caching

**Cloudflare:**
- Anycast DNS network (1.1.1.1)
- Sub-10ms query times globally
- DDoS protection at DNS layer

**Facebook:**
- Internal DNS infrastructure
- GeoDNS for global traffic routing
- Custom DNS protocols for optimization

---

## Summary

DNS is a critical component of internet infrastructure that translates domain names to IP addresses.

**Key Takeaways:**

1. **Hierarchical Structure**: Root → TLD → Authoritative servers
2. **Record Types**: A, AAAA, CNAME, MX, TXT, NS, SOA
3. **Resolution Process**: Recursive and iterative queries
4. **Caching**: Multiple levels with TTL-based expiration
5. **Advanced Features**: GeoDNS, failover, load balancing
6. **Security**: DNSSEC, DoH, DoT
7. **Performance**: Optimize TTL, reduce lookups, use Anycast

**For System Design Interviews:**
- Understand DNS resolution flow
- Know when to use different record types
- Consider global traffic routing
- Plan for failover and disaster recovery
- Balance performance vs. flexibility with TTL
