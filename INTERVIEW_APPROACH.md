# The System Design Interview Approach

> A structured framework for tackling any system design problem

---

## The Mindset

Before diving into frameworks, understand this:

1. **There is no perfect answer** - Only trade-offs aligned with requirements
2. **Communication > Solution** - Interviewers evaluate your thinking process
3. **Drive the conversation** - Don't wait for prompts; lead the discussion
4. **It's iterative** - Start simple, then evolve the design
5. **Requirements shape architecture** - Business needs drive technical decisions

---

## The 6-Step Framework

### Step 1: Clarify Requirements (2-3 minutes)

**Goal:** Understand WHAT we're building and for WHOM

#### Functional Requirements
Ask: "What are the core features users need?"

```
Example - Twitter:
✓ Post tweets (text, images, videos)
✓ Follow/unfollow users
✓ View home timeline
✓ Like, retweet, reply
✗ Direct messages (clarify if in scope)
✗ Trending topics (clarify if in scope)
```

#### Non-Functional Requirements
Ask: "What qualities must the system have?"

| Requirement | Question to Ask |
|-------------|-----------------|
| **Scale** | How many users? DAU? |
| **Availability** | Can we tolerate downtime? |
| **Consistency** | Must reads be immediately consistent? |
| **Latency** | What's acceptable response time? |
| **Durability** | Can we lose any data? |

#### Constraints
Ask: "What are the limitations?"

- Budget constraints
- Tech stack preferences
- Regulatory requirements (GDPR, HIPAA)
- Geographic distribution

#### Template Conversation
```
"Before I dive in, let me understand the requirements:
- Are we building the full Twitter or focusing on core features?
- What scale are we targeting? Millions or billions of users?
- Is this read-heavy or write-heavy?
- Do we need real-time updates or near-real-time is okay?
- Any geographic constraints?"
```

---

### Step 2: Estimate Scale (2-3 minutes)

**Goal:** Quantify the system to make informed decisions

#### Back-of-Envelope Calculations

```
Example - Twitter Clone:

Users:
- Total users: 500M
- Daily Active Users (DAU): 200M
- Average tweets per user per day: 2

Traffic:
- Tweets per day: 200M × 2 = 400M tweets/day
- Tweets per second: 400M / 86400 ≈ 4,600 TPS
- Peak TPS: 4,600 × 3 = ~14,000 TPS

Read/Write Ratio:
- Each user reads 100 tweets/day
- Reads per day: 200M × 100 = 20B reads/day
- Read TPS: ~230,000 TPS
- Read:Write ratio = 50:1 (READ HEAVY)

Storage:
- Tweet size: ~500 bytes (text + metadata)
- Daily storage: 400M × 500B = 200GB/day
- Yearly storage: 200GB × 365 = 73TB/year
- 5-year storage: ~365TB
```

#### Quick Reference Numbers

| Metric | Value |
|--------|-------|
| Seconds in a day | 86,400 (~100K) |
| Seconds in a month | 2.5M |
| 1 Million requests/day | ~12 TPS |
| 1 Billion requests/day | ~12,000 TPS |

#### Storage Quick Math

| Data Type | Typical Size |
|-----------|--------------|
| Character | 1-4 bytes |
| UUID | 16 bytes |
| Timestamp | 8 bytes |
| Short text (tweet) | 280 bytes |
| Image (compressed) | 200KB - 1MB |
| Video (1 min, compressed) | 10-50MB |

---

### Step 3: Define API (3-5 minutes)

**Goal:** Define the contract between client and server

#### RESTful API Design

```
Example - Twitter API:

POST /api/v1/tweets
Request:
{
    "content": "Hello World!",
    "media_ids": ["abc123"],
    "reply_to": null
}
Response: 201 Created
{
    "tweet_id": "tweet_12345",
    "created_at": "2026-01-15T10:30:00Z"
}

GET /api/v1/timeline?page=1&limit=20
Response: 200 OK
{
    "tweets": [...],
    "next_cursor": "cursor_xyz"
}

POST /api/v1/users/{user_id}/follow
Response: 200 OK

DELETE /api/v1/users/{user_id}/follow
Response: 200 OK
```

#### Key Considerations
- Pagination strategy (cursor vs offset)
- Rate limiting approach
- Authentication (OAuth, JWT)
- Versioning strategy
- Error response format

---

### Step 4: High-Level Design (10-15 minutes)

**Goal:** Draw the architecture showing major components

#### Start Simple

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Client  │────▶│   API    │────▶│ Database │
└──────────┘     └──────────┘     └──────────┘
```

#### Evolve Based on Requirements

```
                                    ┌─────────────┐
                                    │   CDN       │
                                    └──────┬──────┘
                                           │
┌──────────┐     ┌──────────────┐    ┌─────▼─────┐
│  Client  │────▶│Load Balancer │───▶│API Gateway│
└──────────┘     └──────────────┘    └─────┬─────┘
                                           │
                      ┌────────────────────┼────────────────────┐
                      │                    │                    │
               ┌──────▼──────┐     ┌───────▼──────┐    ┌───────▼──────┐
               │Tweet Service│     │User Service  │    │Timeline Svc  │
               └──────┬──────┘     └───────┬──────┘    └───────┬──────┘
                      │                    │                    │
               ┌──────▼──────┐     ┌───────▼──────┐    ┌───────▼──────┐
               │  Tweet DB   │     │   User DB    │    │Timeline Cache│
               └─────────────┘     └──────────────┘    └──────────────┘
```

#### Component Checklist

| Component | When to Use |
|-----------|-------------|
| Load Balancer | Multiple servers, high availability |
| API Gateway | Multiple services, auth, rate limiting |
| Cache | Read-heavy, reduce DB load |
| CDN | Static content, global users |
| Message Queue | Async processing, decoupling |
| Search Engine | Full-text search, complex queries |
| Object Storage | Media files, large blobs |

---

### Step 5: Deep Dive (10-15 minutes)

**Goal:** Detail critical components; show expertise

#### Pick 2-3 Components to Deep Dive

**Example: Timeline Service Deep Dive**

```
Approach 1: Pull Model (Fan-out on Read)
─────────────────────────────────────────
When user opens app:
1. Fetch list of users they follow
2. Query each user's recent tweets
3. Merge and sort by timestamp
4. Return to user

Pros: Simple, no pre-computation
Cons: Slow for users following many accounts

Approach 2: Push Model (Fan-out on Write)
─────────────────────────────────────────
When user posts tweet:
1. Get list of followers
2. Push tweet to each follower's timeline cache

Pros: Fast reads (pre-computed)
Cons: Expensive for celebrities (millions of followers)

Approach 3: Hybrid Model (Twitter's Actual Approach)
────────────────────────────────────────────────────
- Regular users: Push model
- Celebrities (>10K followers): Pull model
- Combine at read time

Why: Best of both worlds
Trade-off: More complex system
```

#### Deep Dive Template

```
For each component, discuss:
1. Data model / Schema
2. Algorithm / Approach
3. Scaling strategy
4. Failure handling
5. Trade-offs made
```

---

### Step 6: Address Bottlenecks & Scale (5-10 minutes)

**Goal:** Identify problems and propose solutions

#### Common Bottlenecks & Solutions

| Bottleneck | Solutions |
|------------|-----------|
| Single DB | Read replicas, sharding |
| Hot partitions | Better partition key, caching |
| Slow writes | Async processing, write-behind cache |
| Slow reads | Caching layers, CDN |
| Single point of failure | Redundancy, failover |
| Large media files | Object storage, CDN |

#### Scaling Patterns

```
Horizontal Scaling:
├── Stateless services (easy to scale)
├── Database sharding
├── Cache clustering
└── Load balancer distribution

Vertical Scaling:
├── Bigger machines (quick fix)
├── Better algorithms (O(n) → O(log n))
└── Query optimization

Caching Layers:
├── CDN (edge caching)
├── Application cache (Redis)
├── Database query cache
└── Client-side cache
```

#### Monitoring & Alerting
Always mention:
- Key metrics to track (latency, error rate, throughput)
- Alerting thresholds
- Logging and tracing
- Dashboard requirements

---

## Design Thinking Examples

### Scenario 1: Read-Heavy System (News Feed)

```
Requirement: 100:1 read to write ratio
Thinking:
→ Heavy caching is essential
→ Pre-compute content where possible
→ Push updates to cache on write
→ Accept eventual consistency
→ Use CDN for static content

Architecture Implications:
- Multi-layer caching (CDN → Redis → DB)
- Read replicas for database
- Async write processing
```

### Scenario 2: Write-Heavy System (Logging)

```
Requirement: 1000:1 write to read ratio
Thinking:
→ Optimize for write throughput
→ Batch writes
→ Use append-only storage
→ Background indexing
→ Eventual consistency acceptable

Architecture Implications:
- Message queue for buffering
- Time-series database
- Async processing workers
- Columnar storage for analytics
```

### Scenario 3: Strong Consistency Required (Banking)

```
Requirement: Money transfers must be atomic
Thinking:
→ ACID transactions needed
→ Single source of truth
→ Synchronous replication
→ Accept higher latency
→ Use distributed locks

Architecture Implications:
- SQL database with transactions
- Synchronous replication
- Saga pattern for distributed transactions
- Idempotency keys
```

### Scenario 4: Global Availability (E-commerce)

```
Requirement: Users worldwide, 99.99% uptime
Thinking:
→ Multi-region deployment
→ Data replication across regions
→ Handle network partitions
→ Balance consistency vs availability

Architecture Implications:
- CDN for static content
- Regional databases with cross-region replication
- Eventually consistent for non-critical data
- Strong consistency for orders/payments
```

---

## Common Mistakes to Avoid

| Mistake | Better Approach |
|---------|-----------------|
| Jumping to solution | Clarify requirements first |
| Over-engineering | Start simple, evolve as needed |
| Ignoring trade-offs | Explicitly discuss pros/cons |
| Single database | Consider sharding early |
| Forgetting failure modes | Design for failure |
| Not estimating | Always do back-of-envelope math |
| Monolithic thinking | Consider service boundaries |

---

## Quick Reference: CAP Theorem in Practice

```
            Consistency
               /\
              /  \
             /    \
            / CP   \
           /        \
          /          \
         /     CA     \
        /______________\
Availability ─────── Partition Tolerance

CA: Traditional RDBMS (single node)
CP: MongoDB, HBase, Redis Cluster
AP: Cassandra, DynamoDB, CouchDB
```

**In distributed systems, P is mandatory. Choose between C and A:**
- Banking/Finance → CP (consistency critical)
- Social Media → AP (availability critical)
- E-commerce → Depends on feature (orders: CP, feeds: AP)

---

## The Interview Checklist

Before you finish, ensure you've covered:

- [ ] Functional requirements clarified
- [ ] Non-functional requirements (scale, latency, availability)
- [ ] Back-of-envelope calculations done
- [ ] API endpoints defined
- [ ] High-level architecture drawn
- [ ] Data model discussed
- [ ] Storage decisions explained (SQL/NoSQL, why)
- [ ] Caching strategy
- [ ] Scaling approach
- [ ] Failure handling
- [ ] Trade-offs explicitly stated

---

*Remember: The goal is to demonstrate structured thinking, not to build a perfect system.*
