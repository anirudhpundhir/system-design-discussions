# URL Shortener - Architecture Diagrams

## High-Level Architecture

```mermaid
graph TB
    subgraph Clients
        Browser[Browser]
        Mobile[Mobile App]
        API_Client[API Client]
    end

    subgraph Edge Layer
        DNS[DNS/GeoDNS]
        CDN[CDN - CloudFlare]
    end

    subgraph Load Balancing
        LB[Load Balancer]
    end

    subgraph Application Layer
        API1[API Server 1]
        API2[API Server 2]
        API3[API Server N]
    end

    subgraph Caching Layer
        Redis1[(Redis Primary)]
        Redis2[(Redis Replica)]
    end

    subgraph Data Layer
        DB_Primary[(PostgreSQL Primary)]
        DB_Replica[(PostgreSQL Replica)]
    end

    subgraph Support Services
        IDGen[ID Generator]
        Analytics[Analytics Service]
        Queue[Message Queue]
    end

    Browser --> DNS
    Mobile --> DNS
    API_Client --> DNS
    DNS --> CDN
    CDN --> LB
    LB --> API1
    LB --> API2
    LB --> API3
    API1 --> Redis1
    API2 --> Redis1
    API3 --> Redis1
    Redis1 --> Redis2
    API1 --> DB_Primary
    DB_Primary --> DB_Replica
    API1 --> IDGen
    API1 --> Queue
    Queue --> Analytics
```

## Write Flow (Create Short URL)

```mermaid
sequenceDiagram
    participant C as Client
    participant LB as Load Balancer
    participant API as API Server
    participant IDGen as ID Generator
    participant Cache as Redis
    participant DB as PostgreSQL

    C->>LB: POST /api/v1/urls
    LB->>API: Forward request

    Note over API: Validate URL format

    API->>Cache: Check if URL exists
    alt URL already shortened
        Cache-->>API: Return existing short_code
        API-->>C: 200 OK (existing)
    else New URL
        API->>IDGen: Get next ID
        IDGen-->>API: unique_id

        Note over API: Encode to short_code

        API->>DB: INSERT url record
        DB-->>API: Success

        API->>Cache: SET url:{short_code}
        Cache-->>API: OK

        API-->>C: 201 Created
    end
```

## Read Flow (Redirect)

```mermaid
sequenceDiagram
    participant B as Browser
    participant CDN as CDN Edge
    participant LB as Load Balancer
    participant API as API Server
    participant Cache as Redis
    participant DB as PostgreSQL
    participant Analytics as Analytics Queue

    B->>CDN: GET /abc1234

    alt CDN Cache Hit
        CDN-->>B: 301 Redirect (cached)
    else CDN Cache Miss
        CDN->>LB: Forward
        LB->>API: Forward

        API->>Cache: GET url:abc1234
        alt Cache Hit
            Cache-->>API: long_url
        else Cache Miss
            API->>DB: SELECT long_url
            DB-->>API: long_url
            API->>Cache: SET url:abc1234 EX 86400
        end

        API->>Analytics: Async: log click
        API-->>CDN: 301 + Cache-Control: max-age=3600
        CDN-->>B: 301 Redirect
    end
```

## Database Sharding Strategy

```mermaid
graph TB
    subgraph Router
        R[Shard Router]
    end

    subgraph Shards
        S0[(Shard 0<br/>a-f)]
        S1[(Shard 1<br/>g-l)]
        S2[(Shard 2<br/>m-r)]
        S3[(Shard 3<br/>s-z, 0-9)]
    end

    R -->|short_code starts with a-f| S0
    R -->|short_code starts with g-l| S1
    R -->|short_code starts with m-r| S2
    R -->|short_code starts with s-z,0-9| S3
```

## ID Generation Service

```mermaid
graph TB
    subgraph Coordinator
        ZK[Zookeeper/etcd]
    end

    subgraph ID Generators
        IG1[Generator 1<br/>Range: 1-1M]
        IG2[Generator 2<br/>Range: 1M-2M]
        IG3[Generator 3<br/>Range: 2M-3M]
    end

    subgraph API Servers
        API1[API 1]
        API2[API 2]
        API3[API 3]
    end

    ZK -->|Allocate range| IG1
    ZK -->|Allocate range| IG2
    ZK -->|Allocate range| IG3

    API1 --> IG1
    API2 --> IG2
    API3 --> IG3
```

## Multi-Region Deployment

```mermaid
graph TB
    subgraph Global
        GDNS[Global DNS<br/>GeoDNS Routing]
    end

    subgraph US Region
        US_CDN[US CDN]
        US_LB[US Load Balancer]
        US_API[US API Cluster]
        US_Cache[(US Redis)]
        US_DB[(US PostgreSQL<br/>Primary)]
    end

    subgraph EU Region
        EU_CDN[EU CDN]
        EU_LB[EU Load Balancer]
        EU_API[EU API Cluster]
        EU_Cache[(EU Redis)]
        EU_DB[(EU PostgreSQL<br/>Replica)]
    end

    subgraph Asia Region
        ASIA_CDN[Asia CDN]
        ASIA_LB[Asia Load Balancer]
        ASIA_API[Asia API Cluster]
        ASIA_Cache[(Asia Redis)]
        ASIA_DB[(Asia PostgreSQL<br/>Replica)]
    end

    GDNS --> US_CDN
    GDNS --> EU_CDN
    GDNS --> ASIA_CDN

    US_CDN --> US_LB --> US_API --> US_Cache --> US_DB
    EU_CDN --> EU_LB --> EU_API --> EU_Cache --> EU_DB
    ASIA_CDN --> ASIA_LB --> ASIA_API --> ASIA_Cache --> ASIA_DB

    US_DB -.->|Async Replication| EU_DB
    US_DB -.->|Async Replication| ASIA_DB
```

## Component Interaction Matrix

```
┌─────────────────┬─────────┬───────┬─────┬──────────┬───────────┐
│                 │ CDN     │ API   │Cache│ Database │ Analytics │
├─────────────────┼─────────┼───────┼─────┼──────────┼───────────┤
│ CDN             │    -    │ sync  │  -  │    -     │     -     │
│ API Server      │  cache  │   -   │sync │   sync   │   async   │
│ Cache (Redis)   │    -    │ data  │  -  │  backup  │     -     │
│ Database        │    -    │ data  │  -  │    -     │     -     │
│ Analytics       │    -    │   -   │  -  │    -     │     -     │
└─────────────────┴─────────┴───────┴─────┴──────────┴───────────┘

Legend:
- sync: synchronous communication
- async: asynchronous communication
- cache: caching relationship
- data: data storage relationship
- backup: cache-aside pattern
```

---

## PlantUML Version (Alternative)

For more detailed diagrams, see `architecture.puml` file.

```plantuml
@startuml URL Shortener Architecture

!define RECTANGLE class

skinparam componentStyle rectangle
skinparam backgroundColor #FEFEFE

package "Client Layer" {
    [Web Browser] as Browser
    [Mobile App] as Mobile
    [API Client] as APIClient
}

package "Edge Layer" {
    [CDN] as CDN
    [Load Balancer] as LB
}

package "Application Layer" {
    [API Server 1] as API1
    [API Server 2] as API2
    [ID Generator] as IDGen
}

package "Cache Layer" {
    database "Redis Primary" as RedisPrimary
    database "Redis Replica" as RedisReplica
}

package "Data Layer" {
    database "PostgreSQL Primary" as DBPrimary
    database "PostgreSQL Replica" as DBReplica
}

Browser --> CDN
Mobile --> CDN
APIClient --> LB
CDN --> LB
LB --> API1
LB --> API2
API1 --> RedisPrimary
API2 --> RedisPrimary
RedisPrimary --> RedisReplica
API1 --> DBPrimary
DBPrimary --> DBReplica
API1 --> IDGen

@enduml
```
