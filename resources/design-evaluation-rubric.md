# System Design Evaluation Rubric

A comprehensive scoring framework to evaluate system designs for optimality, performance, and scalability.

---

## Quick Score Calculator

Use this rubric to score any system design from 0-100.

```
Total Score = Σ (Category Score × Weight)

Categories:
├── Requirements & Scope (15%)
├── Architecture Quality (25%)
├── Scalability (20%)
├── Reliability (15%)
├── Performance (10%)
├── Security (10%)
└── Operability (5%)
```

---

## Detailed Scoring Rubric

### 1. Requirements & Scope (15 points)

| Score | Criteria |
|-------|----------|
| **15** | Functional and non-functional requirements clearly defined; explicit assumptions; constraints documented |
| **12** | Most requirements captured; some assumptions implicit |
| **9** | Basic requirements identified; missing key non-functional requirements |
| **6** | Vague requirements; no scale estimates |
| **3** | Requirements not clarified; jumped to solution |
| **0** | No requirements discussion |

**Checklist:**
- [ ] Functional requirements listed
- [ ] Non-functional requirements (latency, availability, consistency)
- [ ] Scale estimates (users, requests/sec, storage)
- [ ] Constraints and assumptions stated
- [ ] Out-of-scope items identified

---

### 2. Architecture Quality (25 points)

| Score | Criteria |
|-------|----------|
| **25** | Clean separation of concerns; appropriate service boundaries; follows SOLID principles; diagrams clear |
| **20** | Good architecture; minor coupling issues; mostly follows best practices |
| **15** | Reasonable architecture; some questionable decisions; works but not optimal |
| **10** | Over-engineered or under-engineered; significant design issues |
| **5** | Major architectural flaws; hard to understand or maintain |
| **0** | No coherent architecture |

**Checklist:**
- [ ] Clear service/component boundaries
- [ ] Single responsibility for each component
- [ ] Loose coupling between services
- [ ] Data flow is clear and logical
- [ ] API design follows best practices
- [ ] Appropriate use of patterns (not over-engineered)

---

### 3. Scalability (20 points)

| Score | Criteria |
|-------|----------|
| **20** | Horizontal scaling at all layers; sharding strategy defined; handles 10x growth |
| **16** | Good scalability; minor bottlenecks; handles expected growth |
| **12** | Basic scalability; some layers don't scale well |
| **8** | Significant scaling limitations; single points of failure |
| **4** | Minimal scalability consideration |
| **0** | Won't scale beyond trivial load |

**Scalability Checklist:**

| Layer | Scales? | How? |
|-------|---------|------|
| Load Balancer | [ ] | |
| Application | [ ] | Stateless + auto-scaling |
| Cache | [ ] | Cluster/sharding |
| Database | [ ] | Read replicas + sharding |
| Storage | [ ] | Distributed/cloud storage |

**Scoring Formula:**
```
Scalability Score = (Stateless × 4) + (DB Scaling × 6) +
                   (Caching × 4) + (Async Processing × 3) +
                   (No SPOF × 3)
```

---

### 4. Reliability (15 points)

| Score | Criteria |
|-------|----------|
| **15** | No SPOF; graceful degradation; comprehensive failure handling; chaos-ready |
| **12** | Good redundancy; most failures handled; minor gaps |
| **9** | Basic redundancy; common failures handled |
| **6** | Some reliability measures; SPOF present |
| **3** | Minimal reliability; many failure modes unhandled |
| **0** | No reliability consideration |

**Reliability Checklist:**
- [ ] No single point of failure
- [ ] Database replication configured
- [ ] Retry logic with backoff
- [ ] Circuit breakers for external calls
- [ ] Graceful degradation paths
- [ ] Health checks and auto-recovery
- [ ] Multi-AZ/region consideration

---

### 5. Performance (10 points)

| Score | Criteria |
|-------|----------|
| **10** | Meets all latency targets; optimized data access; efficient algorithms |
| **8** | Good performance; minor optimization opportunities |
| **6** | Acceptable performance; some slow paths |
| **4** | Performance issues in key flows |
| **2** | Significant performance problems |
| **0** | Performance not considered |

**Performance Checklist:**
- [ ] Caching strategy defined
- [ ] Database queries optimized
- [ ] Network hops minimized
- [ ] Async where appropriate
- [ ] CDN for static content
- [ ] Pagination implemented

---

### 6. Security (10 points)

| Score | Criteria |
|-------|----------|
| **10** | Defense in depth; encryption everywhere; proper auth/authz; audit logging |
| **8** | Good security; minor gaps |
| **6** | Basic security; some risks |
| **4** | Minimal security measures |
| **2** | Significant security gaps |
| **0** | Security not considered |

**Security Checklist:**
- [ ] Authentication mechanism defined
- [ ] Authorization (RBAC/ABAC) considered
- [ ] Data encrypted in transit (TLS)
- [ ] Data encrypted at rest
- [ ] Input validation mentioned
- [ ] Secrets management approach

---

### 7. Operability (5 points)

| Score | Criteria |
|-------|----------|
| **5** | Full observability; clear deployment strategy; runbooks mentioned |
| **4** | Good monitoring; basic deployment plan |
| **3** | Some monitoring; deployment approach unclear |
| **2** | Minimal operations consideration |
| **0** | Operations not discussed |

**Operability Checklist:**
- [ ] Key metrics identified
- [ ] Logging strategy
- [ ] Alerting approach
- [ ] Deployment strategy (blue-green, canary)
- [ ] Rollback plan

---

## Score Interpretation

| Total Score | Rating | Interpretation |
|-------------|--------|----------------|
| **90-100** | Excellent | Production-ready design, interview ace |
| **80-89** | Good | Solid design, minor improvements possible |
| **70-79** | Acceptable | Decent design, some gaps to address |
| **60-69** | Needs Work | Fundamental issues to resolve |
| **Below 60** | Insufficient | Major redesign needed |

---

## Design Review Template

### Summary Scores

```
┌─────────────────────────────────────────────────────────────┐
│                 DESIGN EVALUATION SUMMARY                    │
├─────────────────────────────────────────────────────────────┤
│ Design:        [System Name]                                 │
│ Reviewer:      [Name]                                        │
│ Date:          [YYYY-MM-DD]                                  │
├─────────────────────────────────────────────────────────────┤
│ Category              │ Score │ Max │ Weight │ Weighted     │
├───────────────────────┼───────┼─────┼────────┼──────────────┤
│ Requirements          │   /15 │  15 │  1.0   │              │
│ Architecture          │   /25 │  25 │  1.0   │              │
│ Scalability           │   /20 │  20 │  1.0   │              │
│ Reliability           │   /15 │  15 │  1.0   │              │
│ Performance           │   /10 │  10 │  1.0   │              │
│ Security              │   /10 │  10 │  1.0   │              │
│ Operability           │    /5 │   5 │  1.0   │              │
├───────────────────────┼───────┼─────┼────────┼──────────────┤
│ TOTAL                 │       │ 100 │        │      /100    │
└─────────────────────────────────────────────────────────────┘
```

### Detailed Feedback

```markdown
## Strengths
1.
2.
3.

## Areas for Improvement
1.
2.
3.

## Critical Issues (Must Fix)
1.
2.

## Recommendations
1.
2.
3.
```

---

## Performance Benchmarks

### Latency Targets

| Operation | Good | Acceptable | Needs Improvement |
|-----------|------|------------|-------------------|
| API Response | < 100ms | < 500ms | > 500ms |
| Page Load | < 2s | < 5s | > 5s |
| Database Query | < 50ms | < 200ms | > 200ms |
| Cache Hit | < 5ms | < 20ms | > 20ms |

### Availability Targets

| Target | Downtime/Year | Suitable For |
|--------|---------------|--------------|
| 99% | 3.65 days | Internal tools |
| 99.9% | 8.76 hours | Business apps |
| 99.99% | 52.6 minutes | Customer-facing |
| 99.999% | 5.26 minutes | Critical systems |

### Throughput Guidelines

| System Type | Expected TPS | Design Consideration |
|-------------|--------------|----------------------|
| Blog/CMS | 100-1K | Single server OK |
| E-commerce | 1K-10K | Load balancing needed |
| Social Media | 10K-100K | Caching critical |
| Real-time | 100K+ | Specialized architecture |

---

## Interview Scoring (Additional)

For interview evaluation, add these criteria:

### Communication (Bonus 10 points)

| Score | Criteria |
|-------|----------|
| **10** | Clear explanation, drove conversation, asked great questions |
| **7** | Good communication, mostly led discussion |
| **4** | Adequate explanation, needed prompting |
| **0** | Poor communication, hard to follow |

### Trade-off Discussion (Bonus 10 points)

| Score | Criteria |
|-------|----------|
| **10** | Explicitly discussed multiple options with pros/cons |
| **7** | Mentioned alternatives, explained choices |
| **4** | Some trade-off awareness |
| **0** | No trade-off discussion |

---

## Using This Rubric

### For Self-Assessment
1. Design your solution
2. Score each category honestly
3. Identify lowest-scoring areas
4. Iterate and improve

### For Peer Review
1. Have reviewer score independently
2. Discuss scoring differences
3. Create action items for improvements

### For Interview Practice
1. Record yourself designing
2. Score using rubric
3. Focus on weakest areas
4. Practice until consistently scoring 80+

---

*Fair evaluation leads to better designs. Use this rubric consistently.*
