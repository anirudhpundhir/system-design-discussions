# Scalability Patterns

## Understanding Scale

### Types of Scaling

```
┌─────────────────────────────────────────────────────────────────┐
│                     VERTICAL SCALING                             │
│                      (Scale Up)                                  │
│                                                                  │
│    ┌─────────┐         ┌─────────────────┐                      │
│    │ Server  │   →     │ Bigger Server   │                      │
│    │  4 CPU  │         │    32 CPU       │                      │
│    │  8 GB   │         │   256 GB        │                      │
│    └─────────┘         └─────────────────┘                      │
│                                                                  │
│    Pros: Simple, no code changes                                │
│    Cons: Limited ceiling, expensive, single point of failure    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    HORIZONTAL SCALING                            │
│                      (Scale Out)                                 │
│                                                                  │
│    ┌─────────┐         ┌─────────┐ ┌─────────┐ ┌─────────┐     │
│    │ Server  │   →     │ Server  │ │ Server  │ │ Server  │     │
│    └─────────┘         └─────────┘ └─────────┘ └─────────┘     │
│                                                                  │
│    Pros: Unlimited scale, redundancy, cost-effective            │
│    Cons: Requires distributed design, more complex              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Load Balancing

### Types of Load Balancers

```
Layer 4 (Transport):
┌─────────┐     ┌─────────┐
│ Client  │────►│   L4    │────► Server
└─────────┘     │   LB    │      (TCP/UDP level)
                └─────────┘

Layer 7 (Application):
┌─────────┐     ┌─────────┐
│ Client  │────►│   L7    │────► Server
└─────────┘     │   LB    │      (HTTP level, can route by URL, headers)
                └─────────┘
```

### Load Balancing Algorithms

| Algorithm | How It Works | Best For |
|-----------|--------------|----------|
| **Round Robin** | Rotate through servers | Homogeneous servers |
| **Weighted Round Robin** | Rotate with weights | Different server capacities |
| **Least Connections** | Route to server with fewest connections | Long-lived connections |
| **Least Response Time** | Route to fastest server | Varying server loads |
| **IP Hash** | Hash client IP to server | Session affinity |
| **Consistent Hashing** | Minimize remapping on changes | Distributed caches |

### Health Checks

```
Load Balancer
     │
     ├──► Server 1 ✓ (healthy)
     │    GET /health → 200 OK
     │
     ├──► Server 2 ✗ (unhealthy, removed from pool)
     │    GET /health → 503 or timeout
     │
     └──► Server 3 ✓ (healthy)
          GET /health → 200 OK
```

---

## Caching Strategies

### Cache Hierarchy

```
            ┌──────────────────┐
            │  Browser Cache   │  ~1ms
            │  (localStorage)  │
            └────────┬─────────┘
                     │
            ┌────────▼─────────┐
            │       CDN        │  ~10ms
            │  (Edge servers)  │
            └────────┬─────────┘
                     │
            ┌────────▼─────────┐
            │  Application     │  ~1ms
            │  Cache (Redis)   │
            └────────┬─────────┘
                     │
            ┌────────▼─────────┐
            │ Database Query   │  ~10ms
            │     Cache        │
            └────────┬─────────┘
                     │
            ┌────────▼─────────┐
            │    Database      │  ~100ms
            └──────────────────┘
```

### Caching Patterns

#### Cache-Aside (Lazy Loading)

```
Read:
1. Check cache
2. If miss, read from DB
3. Write to cache
4. Return data

┌─────────┐    1. get(key)    ┌─────────┐
│   App   │◄─────────────────►│  Cache  │
└────┬────┘    3. set(key)    └─────────┘
     │
     │ 2. SELECT (on miss)
     ▼
┌─────────┐
│   DB    │
└─────────┘
```

#### Read-Through

```
1. App requests from cache
2. Cache fetches from DB if miss
3. Cache returns data

┌─────────┐    1. get(key)    ┌─────────┐
│   App   │◄─────────────────►│  Cache  │
└─────────┘                   └────┬────┘
                                   │ 2. SELECT (on miss)
                                   ▼
                              ┌─────────┐
                              │   DB    │
                              └─────────┘
```

#### Write-Through

```
1. App writes to cache
2. Cache synchronously writes to DB
3. Both updated together

┌─────────┐    1. set(key)    ┌─────────┐
│   App   │──────────────────►│  Cache  │
└─────────┘                   └────┬────┘
                                   │ 2. INSERT/UPDATE
                                   ▼
                              ┌─────────┐
                              │   DB    │
                              └─────────┘
```

#### Write-Behind (Write-Back)

```
1. App writes to cache
2. Cache immediately returns
3. Cache asynchronously writes to DB

┌─────────┐    1. set(key)    ┌─────────┐
│   App   │──────────────────►│  Cache  │
└─────────┘                   └────┬────┘
                                   │ 3. Async batch write
                                   ▼
                              ┌─────────┐
                              │   DB    │
                              └─────────┘

Pros: Fast writes
Cons: Risk of data loss if cache fails before flush
```

### Cache Eviction Policies

| Policy | Description | Use Case |
|--------|-------------|----------|
| **LRU** | Least Recently Used | General purpose |
| **LFU** | Least Frequently Used | When access patterns vary |
| **TTL** | Time To Live | Data with known expiry |
| **FIFO** | First In First Out | Simple, predictable |

### Cache Invalidation

```
The Two Hard Problems in Computer Science:
1. Cache invalidation
2. Naming things
3. Off-by-one errors

Strategies:
┌─────────────────────────────────────────────────────────────┐
│ TTL (Time-based): Cache expires after duration              │
│ Pros: Simple, automatic                                     │
│ Cons: May serve stale data, wasteful refresh                │
├─────────────────────────────────────────────────────────────┤
│ Event-based: Invalidate on writes                           │
│ Pros: Always fresh                                          │
│ Cons: Complex, tight coupling                               │
├─────────────────────────────────────────────────────────────┤
│ Hybrid: TTL + event invalidation                            │
│ Pros: Best of both worlds                                   │
│ Cons: More complex                                          │
└─────────────────────────────────────────────────────────────┘
```

---

## Database Scaling

### Replication

```
Master-Slave (Primary-Replica):
┌──────────┐
│  Primary │◄─── All writes
└────┬─────┘
     │ Replication
     ▼
┌────┴────┬─────────┐
│Replica 1│Replica 2│◄─── Reads distributed
└─────────┴─────────┘

Benefits: Read scaling, high availability
Trade-off: Replication lag (eventual consistency)
```

```
Multi-Master:
┌──────────┐     ┌──────────┐
│ Master 1 │◄───►│ Master 2 │
└──────────┘     └──────────┘
Both accept writes, replicate to each other

Benefits: Write availability, geographic distribution
Trade-off: Conflict resolution needed
```

### Sharding (Partitioning)

```
Horizontal Sharding:
┌─────────────────────────────────────────────┐
│                 Users Table                  │
│  user_id │ name  │ email                     │
│     1    │ Alice │ alice@...                 │
│     2    │ Bob   │ bob@...                   │
│    ...   │  ...  │ ...                       │
└─────────────────────────────────────────────┘
                    │
     ┌──────────────┼──────────────┐
     ▼              ▼              ▼
┌─────────┐   ┌─────────┐   ┌─────────┐
│ Shard 0 │   │ Shard 1 │   │ Shard 2 │
│ id % 3=0│   │ id % 3=1│   │ id % 3=2│
└─────────┘   └─────────┘   └─────────┘
```

#### Sharding Strategies

| Strategy | Description | Pros/Cons |
|----------|-------------|-----------|
| **Range-based** | Shard by range (A-M, N-Z) | Easy, but hot spots |
| **Hash-based** | Shard by hash(key) % n | Even distribution, but range queries hard |
| **Directory-based** | Lookup service maps key → shard | Flexible, but lookup overhead |
| **Geographic** | Shard by region | Low latency, but uneven load |

#### Consistent Hashing

```
Without Consistent Hashing:
Adding server remaps most keys

hash(key) % 3 → shard
hash(key) % 4 → different shard (75% keys move!)

With Consistent Hashing:
Only K/N keys move when adding/removing nodes

┌────────────────────────────────┐
│     Virtual Ring              │
│          N1                    │
│        ┌───┐                   │
│     N4 │   │ N2                │
│        └───┘                   │
│          N3                    │
│                                │
│  Key hashed to position        │
│  Assigned to next node         │
│  Adding node: only moves keys  │
│  from one adjacent node        │
└────────────────────────────────┘
```

---

## Message Queues

### Why Queues?

```
Synchronous (Tight Coupling):
Service A ─────► Service B ─────► Service C
                 (blocked)         (blocked)

If B is slow or down, A waits or fails.

Asynchronous (Decoupled):
Service A ─────► Queue ─────► Service B
                       └────► Service C

A doesn't wait. B and C process independently.
```

### Queue Patterns

#### Point-to-Point

```
Producer ─────► Queue ─────► Consumer
                  │
One message, one consumer
```

#### Publish-Subscribe

```
Producer ─────► Topic ─────► Consumer 1
                     └────► Consumer 2
                     └────► Consumer 3

One message, multiple consumers (broadcast)
```

#### Fan-out

```
                    ┌────► Queue 1 ─────► Consumer A
Producer ─────► ────┼────► Queue 2 ─────► Consumer B
                    └────► Queue 3 ─────► Consumer C

Message copied to multiple queues
```

### Delivery Guarantees

| Guarantee | Description | Use Case |
|-----------|-------------|----------|
| **At-most-once** | May lose messages | Logs, metrics |
| **At-least-once** | May duplicate | Most use cases (with idempotency) |
| **Exactly-once** | No loss, no duplicates | Financial, complex transactions |

---

## Content Delivery Networks (CDN)

```
Without CDN:
User (Tokyo) ─────────────────────► Server (US)
             High latency, slow

With CDN:
User (Tokyo) ─────► Edge (Tokyo) ─── cache hit ──► Content
                         │
                    cache miss
                         │
                         ▼
                    Server (US)
```

### CDN Caching Strategy

```
Static Content:
• Images, CSS, JS, fonts
• Long TTL (days/weeks)
• Cache-Control: max-age=31536000

Dynamic Content:
• API responses, personalized pages
• Short TTL or no cache
• Cache-Control: private, no-store
```

---

## Auto Scaling

```
┌─────────────────────────────────────────────────────────────┐
│                    Auto Scaling Group                        │
│                                                              │
│   Metrics: CPU > 70%    →    Scale Out (add instances)      │
│            CPU < 30%    →    Scale In (remove instances)    │
│                                                              │
│   ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐           │
│   │Instance│  │Instance│  │Instance│  │Instance│           │
│   │   1    │  │   2    │  │   3    │  │   4    │  ← Added  │
│   └────────┘  └────────┘  └────────┘  └────────┘           │
│                                                              │
│   min: 2, max: 10, desired: 3                               │
└─────────────────────────────────────────────────────────────┘
```

### Scaling Triggers

| Metric | Scale Out | Scale In |
|--------|-----------|----------|
| CPU | > 70% | < 30% |
| Memory | > 80% | < 40% |
| Request count | > threshold | < threshold |
| Queue depth | Growing | Empty |
| Custom metric | Business-specific | Business-specific |

---

## Summary: When to Use What

| Challenge | Pattern |
|-----------|---------|
| Read-heavy load | Caching, Read Replicas, CDN |
| Write-heavy load | Sharding, Write-behind cache |
| High availability | Replication, Load Balancing |
| Uneven load | Consistent Hashing |
| Tight coupling | Message Queues |
| Global users | CDN, Geo-replication |
| Burst traffic | Auto Scaling, Queues |
| Large files | Object Storage + CDN |

---

## Further Reading
- [Scalability for Dummies](https://www.lecloud.net/tagged/scalability)
- [High Scalability Blog](http://highscalability.com/)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
