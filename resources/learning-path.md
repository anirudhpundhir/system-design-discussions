# System Design Learning Path

A structured approach to mastering system design for interviews and career growth.

---

## Learning Stages

### Stage 1: Foundation (2-4 weeks)

**Goal:** Understand core concepts and vocabulary

```
Week 1-2: Distributed Systems Basics
├── Read: fundamentals/cap-theorem/
├── Read: fundamentals/design-principles/
├── Practice: Explain CAP to someone
└── Quiz yourself on terminology

Week 3-4: Scaling Fundamentals
├── Read: fundamentals/scalability-patterns/
├── Read: fundamentals/data-patterns/
├── Practice: Draw basic architectures
└── Understand: When to use what
```

**Milestone:** Can explain basic concepts like:
- Horizontal vs vertical scaling
- Caching strategies
- Database replication
- Load balancing

---

### Stage 2: Component Deep Dive (4-6 weeks)

**Goal:** Understand how individual components work

```
Week 1-2: Databases
├── SQL vs NoSQL trade-offs
├── Sharding strategies
├── Replication modes
└── technologies/databases/

Week 3-4: Caching & Messaging
├── Cache patterns (aside, through, behind)
├── Message queues vs event streams
├── Kafka, RabbitMQ, Redis
└── technologies/messaging-systems/

Week 5-6: Infrastructure
├── Load balancers (L4 vs L7)
├── CDN and edge computing
├── Container orchestration basics
└── technologies/cloud-infrastructure/
```

**Milestone:** Can explain trade-offs between:
- PostgreSQL vs Cassandra vs MongoDB
- Kafka vs RabbitMQ
- Redis vs Memcached

---

### Stage 3: Design Practice (4-8 weeks)

**Goal:** Apply knowledge to real problems

```
Beginner Problems (Week 1-2):
├── URL Shortener
├── Pastebin
├── Rate Limiter
└── Key-Value Store

Intermediate Problems (Week 3-4):
├── Twitter Timeline
├── Instagram Feed
├── Chat System
└── Notification Service

Advanced Problems (Week 5-6):
├── YouTube/Netflix
├── Uber/Lyft
├── Search Engine
└── Distributed Cache

Expert Problems (Week 7-8):
├── Google Docs
├── Stock Exchange
├── Ad Serving
└── Distributed Database
```

**Milestone:** Can design any of these in 45 minutes following the framework

---

### Stage 4: Interview Preparation (2-4 weeks)

**Goal:** Polish presentation and handle curveballs

```
Week 1: Framework Mastery
├── Practice the 6-step approach
├── Time yourself (45 min total)
├── Record yourself explaining
└── Review: INTERVIEW_APPROACH.md

Week 2: Mock Interviews
├── Practice with peers
├── Get feedback on communication
├── Work on trade-off articulation
└── Handle follow-up questions

Week 3-4: Edge Cases & Deep Dives
├── Study failure scenarios
├── Practice scaling discussions
├── Learn monitoring/alerting answers
└── Prepare for "what if" questions
```

**Milestone:** Comfortable with any design question in interview setting

---

## Topic Priority Matrix

### Must Know (Essential)

| Topic | Why Essential |
|-------|---------------|
| Load Balancing | Every system needs it |
| Caching (Redis) | Core optimization technique |
| Database Sharding | Scale past single machine |
| Message Queues | Async processing |
| API Design | Interface everything |
| CAP Theorem | Fundamental trade-off |

### Should Know (Important)

| Topic | Why Important |
|-------|---------------|
| Consistent Hashing | Distributed caching |
| CDN | Global performance |
| Microservices | Modern architecture |
| Event Sourcing | Audit and replay |
| Database Replication | High availability |
| Rate Limiting | System protection |

### Good to Know (Advanced)

| Topic | Why Useful |
|-------|------------|
| Consensus (Raft/Paxos) | Distributed coordination |
| CRDT | Conflict resolution |
| Bloom Filters | Probabilistic data structures |
| B-Trees/LSM | Database internals |
| Vector Clocks | Causality tracking |
| Two-Phase Commit | Distributed transactions |

---

## Resources by Type

### Books

| Book | Level | Focus |
|------|-------|-------|
| Designing Data-Intensive Applications | Intermediate | Comprehensive |
| System Design Interview (Vol 1 & 2) | Interview | Practice problems |
| Building Microservices | Intermediate | Architecture |
| Database Internals | Advanced | Deep dive |
| Release It! | Intermediate | Production systems |

### Online Courses

| Course | Platform | Focus |
|--------|----------|-------|
| Grokking System Design | Educative | Interview prep |
| MIT 6.824 Distributed Systems | MIT OCW | Academic depth |
| System Design Primer | GitHub | Free overview |

### YouTube Channels

| Channel | Style |
|---------|-------|
| ByteByteGo | Visual explanations |
| Tech Dummies | Interview walkthroughs |
| Gaurav Sen | Deep dives |
| Hussein Nasser | Backend focus |

### Blogs

| Blog | Type |
|------|------|
| High Scalability | Case studies |
| Martin Kleppmann | Academic rigor |
| Uber Engineering | Real-world |
| Netflix Tech | Streaming focus |
| Stripe Engineering | Payment systems |

---

## Daily Practice Routine

### 30-Minute Daily Routine

```
5 min: Review one concept from fundamentals/
10 min: Read one section of a design problem
10 min: Practice explaining a component
5 min: Note questions for deeper study
```

### 2-Hour Weekend Deep Dive

```
30 min: Full design problem walkthrough
30 min: Study one technology deeply
30 min: Compare approaches/trade-offs
30 min: Write notes/contribute to repo
```

---

## Assessment Checklist

### Foundation Level
- [ ] Can explain horizontal vs vertical scaling
- [ ] Understand CAP theorem trade-offs
- [ ] Know when to use SQL vs NoSQL
- [ ] Can describe basic caching strategies
- [ ] Understand load balancing basics

### Intermediate Level
- [ ] Can design a URL shortener in 30 min
- [ ] Understand sharding strategies
- [ ] Know message queue patterns
- [ ] Can estimate capacity requirements
- [ ] Understand consistency models

### Advanced Level
- [ ] Can design any common system in 45 min
- [ ] Deep understanding of trade-offs
- [ ] Can handle scaling follow-ups
- [ ] Understand failure scenarios
- [ ] Can discuss monitoring and alerting

### Expert Level
- [ ] Can design novel systems
- [ ] Deep component knowledge
- [ ] Can optimize existing designs
- [ ] Understand cost trade-offs
- [ ] Can lead design discussions

---

## Common Mistakes to Avoid

| Mistake | How to Avoid |
|---------|--------------|
| Jumping to solution | Always clarify requirements first |
| Over-engineering | Start simple, add complexity as needed |
| Ignoring non-functional requirements | Ask about scale, latency, consistency |
| Not considering failure | Design for failure from the start |
| Poor time management | Practice with timer |
| Not driving the conversation | Lead, don't wait for prompts |

---

## Interview Day Tips

1. **Before:**
   - Review common patterns
   - Prepare clarifying questions
   - Get good sleep

2. **During:**
   - Think out loud
   - Draw as you explain
   - Ask clarifying questions early
   - State assumptions
   - Discuss trade-offs explicitly

3. **After:**
   - Note what went well
   - Identify improvement areas
   - Add learnings to your notes

---

*Learning system design is a marathon, not a sprint. Consistency beats intensity.*
