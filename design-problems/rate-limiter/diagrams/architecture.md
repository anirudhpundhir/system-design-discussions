# Rate Limiter - Architecture Diagrams

## High-Level Architecture

```mermaid
graph TB
    subgraph Clients
        C1[Client 1<br/>API Key: abc]
        C2[Client 2<br/>API Key: xyz]
        C3[Malicious<br/>Bot]
    end

    subgraph Edge Layer
        LB[Load Balancer<br/>L7]
    end

    subgraph Rate Limiting Layer
        RL1[Rate Limiter 1]
        RL2[Rate Limiter 2]
        RL3[Rate Limiter 3]
    end

    subgraph Storage Layer
        RC[(Redis Cluster)]
        Rules[(Rules DB)]
    end

    subgraph Backend Services
        API1[API Server 1]
        API2[API Server 2]
    end

    subgraph Monitoring
        Analytics[Analytics<br/>Service]
        Alerts[Alert<br/>Manager]
    end

    C1 --> LB
    C2 --> LB
    C3 --> LB

    LB --> RL1
    LB --> RL2
    LB --> RL3

    RL1 <--> RC
    RL2 <--> RC
    RL3 <--> RC

    RL1 --> Rules
    RL2 --> Rules
    RL3 --> Rules

    RL1 --> API1
    RL2 --> API2
    RL3 --> API1

    RL1 --> Analytics
    Analytics --> Alerts
```

## Request Flow - Token Bucket Algorithm

```mermaid
sequenceDiagram
    participant C as Client
    participant RL as Rate Limiter
    participant Redis as Redis
    participant API as API Server

    C->>RL: POST /api/resource
    RL->>RL: Extract client_id from API key

    RL->>Redis: GET bucket:{client_id}
    Redis-->>RL: {tokens: 5, last_refill: 1704067200}

    RL->>RL: Calculate tokens to add<br/>(time_elapsed × refill_rate)

    alt Tokens Available
        RL->>Redis: HSET bucket:{client_id}<br/>tokens=4, last_refill=now
        RL->>API: Forward request
        API-->>RL: 200 OK + response
        RL-->>C: 200 OK<br/>X-RateLimit-Remaining: 4
    else No Tokens
        RL-->>C: 429 Too Many Requests<br/>Retry-After: 30
    end
```

## Sliding Window Counter Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant RL as Rate Limiter
    participant Redis as Redis

    C->>RL: Request at 10:30:45

    Note over RL: Current window: 10:30<br/>Previous window: 10:29<br/>Position: 75% into current

    RL->>Redis: MGET client:10:30 client:10:29
    Redis-->>RL: current=30, previous=80

    Note over RL: Weighted = (80 × 0.25) + 30<br/>= 20 + 30 = 50<br/>Limit = 100 ✓

    RL->>Redis: INCR client:10:30
    RL-->>C: 200 OK (50/100 used)
```

## Distributed Rate Limiting Architecture

```mermaid
graph TB
    subgraph Region US
        LB_US[Load Balancer US]
        RL_US1[Rate Limiter]
        RL_US2[Rate Limiter]
        Redis_US[(Redis Primary)]
    end

    subgraph Region EU
        LB_EU[Load Balancer EU]
        RL_EU1[Rate Limiter]
        RL_EU2[Rate Limiter]
        Redis_EU[(Redis Replica)]
    end

    subgraph Region Asia
        LB_ASIA[Load Balancer Asia]
        RL_ASIA1[Rate Limiter]
        RL_ASIA2[Rate Limiter]
        Redis_ASIA[(Redis Replica)]
    end

    LB_US --> RL_US1 & RL_US2
    LB_EU --> RL_EU1 & RL_EU2
    LB_ASIA --> RL_ASIA1 & RL_ASIA2

    RL_US1 & RL_US2 --> Redis_US
    RL_EU1 & RL_EU2 --> Redis_EU
    RL_ASIA1 & RL_ASIA2 --> Redis_ASIA

    Redis_US -.->|Async Replication| Redis_EU
    Redis_US -.->|Async Replication| Redis_ASIA
```

## Token Bucket State Machine

```mermaid
stateDiagram-v2
    [*] --> HasTokens: Initialize with capacity

    HasTokens --> HasTokens: Request arrives,<br/>tokens > 0,<br/>decrement

    HasTokens --> Empty: Request arrives,<br/>tokens = 1,<br/>decrement to 0

    Empty --> Empty: Request arrives,<br/>reject 429

    Empty --> HasTokens: Time passes,<br/>refill tokens

    HasTokens --> HasTokens: Time passes,<br/>refill up to capacity
```

## Rate Limit Decision Tree

```mermaid
flowchart TD
    A[Request Arrives] --> B{Extract Client ID}
    B --> C{Client ID Valid?}

    C -->|No| D[Use IP Address]
    C -->|Yes| E{Get Rate Limit Rule}

    D --> E
    E --> F{Rule Found?}

    F -->|No| G[Apply Default Rule]
    F -->|Yes| H[Apply Specific Rule]

    G --> I{Check Counter}
    H --> I

    I --> J{Under Limit?}

    J -->|Yes| K[Increment Counter]
    K --> L[Forward to API]
    L --> M[Add Rate Limit Headers]
    M --> N[Return Response]

    J -->|No| O[Return 429]
    O --> P[Add Retry-After Header]
```

## Redis Cluster Sharding

```mermaid
graph TB
    subgraph Rate Limiters
        RL1[Rate Limiter 1]
        RL2[Rate Limiter 2]
        RL3[Rate Limiter 3]
    end

    subgraph Redis Cluster
        subgraph Shard 0
            S0M[(Master<br/>slots 0-5460)]
            S0R[(Replica)]
        end

        subgraph Shard 1
            S1M[(Master<br/>slots 5461-10922)]
            S1R[(Replica)]
        end

        subgraph Shard 2
            S2M[(Master<br/>slots 10923-16383)]
            S2R[(Replica)]
        end
    end

    RL1 --> S0M
    RL1 --> S1M
    RL1 --> S2M

    RL2 --> S0M
    RL2 --> S1M
    RL2 --> S2M

    RL3 --> S0M
    RL3 --> S1M
    RL3 --> S2M

    S0M --> S0R
    S1M --> S1R
    S2M --> S2R
```

## Algorithm Comparison Visual

```
TOKEN BUCKET:
┌────────────────────────────────────────────────────────────────┐
│   Bucket (capacity=10)          │  Allows bursts up to 10     │
│   ████████░░                    │  Refills at steady rate     │
│   8 tokens available            │                              │
└────────────────────────────────────────────────────────────────┘

LEAKY BUCKET:
┌────────────────────────────────────────────────────────────────┐
│   ┌─────────────┐               │  Smooths out bursts         │
│   │ ● ● ● ● ●   │ ──► ● ● ●    │  Fixed output rate          │
│   │ Queue       │               │                              │
│   └─────────────┘               │                              │
└────────────────────────────────────────────────────────────────┘

FIXED WINDOW:
┌────────────────────────────────────────────────────────────────┐
│   |────Window 1────|────Window 2────|                         │
│   |     87/100     |     23/100     |  Resets each window    │
│                                      │  Edge case: 2x burst   │
└────────────────────────────────────────────────────────────────┘

SLIDING WINDOW:
┌────────────────────────────────────────────────────────────────┐
│   |=====Previous=====|===Current===|                          │
│   |       80         |     30      | Weighted calculation     │
│   |                  └────40%────┘ | Smooths edge cases       │
│   Weighted = 80×0.6 + 30 = 78                                 │
└────────────────────────────────────────────────────────────────┘
```

## Fail-Open vs Fail-Closed

```mermaid
flowchart TD
    subgraph Fail Open
        A1[Request] --> B1{Redis Available?}
        B1 -->|Yes| C1[Check Limit]
        B1 -->|No| D1[Allow Request]
        C1 --> E1{Under Limit?}
        E1 -->|Yes| F1[Allow]
        E1 -->|No| G1[Reject 429]
        D1 --> H1[Log Warning]
    end

    subgraph Fail Closed
        A2[Request] --> B2{Redis Available?}
        B2 -->|Yes| C2[Check Limit]
        B2 -->|No| D2[Reject 503]
        C2 --> E2{Under Limit?}
        E2 -->|Yes| F2[Allow]
        E2 -->|No| G2[Reject 429]
    end
```

## Local Cache + Redis Architecture

```mermaid
graph TB
    subgraph Rate Limiter Process
        subgraph Local Cache
            LRU[LRU Cache<br/>1000 entries<br/>TTL: 1 second]
        end

        subgraph Logic
            Check[Check Limit]
            Update[Update Counter]
        end
    end

    subgraph Redis Cluster
        RC[(Redis)]
    end

    Request[Incoming Request] --> Check

    Check --> LRU
    LRU -->|Cache Hit| Update
    LRU -->|Cache Miss| RC
    RC --> Update
    Update --> LRU
    Update --> RC
```
