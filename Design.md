# Trade-Off Strategies in Software Architecture

---

## 1. What Are Architecture Trade-Offs?

Every architectural decision is a **trade-off** — choosing one quality attribute often comes at the cost of another. There is no "perfect" architecture; there is only the **right architecture for a given context**.

> "Architecture is the stuff you can't Google." — Mark Richards
>
> "Everything in software architecture is a trade-off. If you think you've found something that isn't a trade-off, you just haven't identified the trade-off yet." — Neal Ford

### The Fundamental Truth

```
You CANNOT maximize ALL quality attributes simultaneously.

Performance ◄──────────────────────────────► Maintainability
Security    ◄──────────────────────────────► Usability
Scalability ◄──────────────────────────────► Simplicity
Consistency ◄──────────────────────────────► Availability
Cost        ◄──────────────────────────────► Quality
Speed to Market ◄──────────────────────────► Technical Excellence
```

### Why Trade-Offs Matter

| Reason | Explanation |
|---|---|
| **Finite resources** | Budget, time, and team skills are always limited |
| **Conflicting qualities** | Improving one attribute often degrades another |
| **Context-dependent** | What's right for Netflix is wrong for a hospital system |
| **Irreversible decisions** | Some choices (database, language, architecture style) are costly to reverse |
| **Business alignment** | Architecture must serve business goals, not engineer preferences |

---

## 2. Key Quality Attributes (The "-ilities")

Architects evaluate trade-offs against these **quality attributes** (also called architectural characteristics or "-ilities"):

```
┌────────────────────────────────────────────────────────────────┐
│                    Quality Attributes                          │
├──────────────────┬──────────────────┬──────────────────────────┤
│  Operational      │  Structural       │  Cross-Cutting           │
├──────────────────┼──────────────────┼──────────────────────────┤
│  Performance      │  Maintainability  │  Security                │
│  Scalability      │  Extensibility    │  Observability           │
│  Availability     │  Testability      │  Cost                    │
│  Reliability      │  Modularity       │  Compliance              │
│  Fault Tolerance  │  Simplicity       │  Accessibility           │
│  Elasticity       │  Reusability      │  Auditability            │
└──────────────────┴──────────────────┴──────────────────────────┘
```

### Definitions

| Attribute | Definition | Measured By |
|---|---|---|
| **Performance** | How fast the system responds | Latency (ms), throughput (req/sec) |
| **Scalability** | Ability to handle increased load | Users, requests, data volume |
| **Availability** | System uptime | % uptime (99.9% = 8.76 hrs downtime/year) |
| **Reliability** | System produces correct results | Error rate, MTBF (Mean Time Between Failures) |
| **Fault Tolerance** | System continues operating despite failures | Recovery time, blast radius |
| **Maintainability** | Ease of changing the system | Time to implement a change |
| **Extensibility** | Ease of adding new features | Lines of code / files changed for new feature |
| **Testability** | Ease of testing the system | Code coverage, test execution time |
| **Simplicity** | Ease of understanding the system | Onboarding time for new developers |
| **Security** | Protection against threats | Vulnerabilities, compliance score |
| **Cost** | Total cost of ownership | Infrastructure + development + maintenance |

---

## 3. The Most Common Architecture Trade-Offs

### Trade-Off 1: Performance vs. Maintainability

```
High Performance                              High Maintainability
(Optimized, complex code)                     (Clean, readable code)
◄─────────────────────────────────────────────►

Example:
  Hand-tuned SQL with stored procedures       EF Core with LINQ
  Inline everything, avoid abstraction        Clean Architecture layers
  Raw byte manipulation                       High-level string operations
  Custom binary protocol                      REST with JSON
```

```csharp
// HIGH PERFORMANCE — harder to maintain
public unsafe int SumArray(int[] arr)
{
    fixed (int* p = arr)
    {
        int sum = 0;
        int* end = p + arr.Length;
        for (int* ptr = p; ptr < end; ptr++)
            sum += *ptr;
        return sum;
    }
}

// HIGH MAINTAINABILITY — slower but clear
public int SumArray(int[] arr) => arr.Sum();
```

| Choose Performance When | Choose Maintainability When |
|---|---|
| Real-time systems (trading, gaming) | Business CRUD applications |
| Processing millions of records/sec | Team changes frequently |
| Every millisecond of latency costs money | Feature velocity matters more |
| System is at hardware limits | System is far from hardware limits |

### Trade-Off 2: Consistency vs. Availability (CAP Theorem)

```
In a distributed system, during a network partition (P),
you MUST choose between:

    Consistency (C)                    Availability (A)
    Every read gets the                Every request gets a
    most recent write                  response (even if stale)
    ◄─────────────────────────────────►

    CP Systems:                        AP Systems:
    ─ MongoDB (default)                ─ Cassandra
    ─ HBase                            ─ DynamoDB
    ─ Redis Cluster                    ─ CouchDB

    Sacrifice:                         Sacrifice:
    Availability during partitions     Consistency during partitions
    (return error rather than          (return stale data rather
     stale data)                        than error)
```

```
                    ┌───────────────┐
                    │  Consistency  │
                    │     (C)       │
                    └───────┬───────┘
                           ╱ ╲
                          ╱   ╲
                    CP   ╱     ╲   CA
                        ╱       ╲
                       ╱  PICK 2  ╲
                      ╱     ONLY    ╲
     ┌───────────────┴─────────────┴───────────────┐
     │  Availability (A)                            │
     │              AP              Partition        │
     │                              Tolerance (P)   │
     └──────────────────────────────────────────────┘

CA = Not realistic in distributed systems (network always fails)
CP = Consistent + Partition Tolerant (sacrifice availability)
AP = Available + Partition Tolerant (sacrifice consistency)
```

**Real-World Examples:**

| System | Choice | Why |
|---|---|---|
| **Banking / Payment** | CP (Consistency) | Wrong balance = financial loss |
| **Social media feed** | AP (Availability) | Showing a slightly old post is fine |
| **Inventory count** | CP (Consistency) | Overselling = revenue loss |
| **Product reviews** | AP (Availability) | Missing one review briefly is acceptable |
| **Flight booking** | CP (Consistency) | Double-booking a seat is unacceptable |
| **Shopping cart** | AP (Availability) | Cart should always be accessible |

### Trade-Off 3: Scalability vs. Simplicity

```
Monolith (Simple)                           Microservices (Scalable)
─────────────────                           ────────────────────────
✅ One codebase                              ✅ Scale individual services
✅ Simple deployment                         ✅ Independent team ownership
✅ Easy debugging                            ✅ Fault isolation
✅ No network latency between modules        ✅ Technology diversity
❌ Scales as a whole unit                    ❌ Distributed system complexity
❌ Single point of failure                   ❌ Network latency, partial failures
❌ Entire team coupled                       ❌ Data consistency challenges
❌ Deployment risk (all or nothing)          ❌ Operational overhead (DevOps)
```

| Team Size | System Complexity | Recommendation |
|---|---|---|
| 1-5 developers | Low-Medium | Monolith |
| 5-15 developers | Medium | Modular Monolith |
| 15-50 developers | High | Microservices |
| 50+ developers | Very High | Microservices + Platform Team |

### Trade-Off 4: Security vs. Usability

```
Fort Knox Security                          Frictionless Experience
──────────────────                          ──────────────────────
MFA on every action                         Single sign-on, remember me
Session timeout = 2 minutes                 Session timeout = 30 days
Complex password rules                      Social login (one click)
Re-authenticate for every page              Authenticate once, browse freely
CAPTCHA everywhere                          No CAPTCHA
```

| Domain | Lean Towards |
|---|---|
| Banking, Healthcare | Security (accept friction) |
| E-commerce, Social media | Usability (accept some risk) |
| Government, Defense | Security (maximum) |
| Entertainment, Gaming | Usability (maximum) |

### Trade-Off 5: Speed to Market vs. Technical Excellence

```
Ship Fast (Startup Mode)                    Build Right (Enterprise Mode)
────────────────────────                    ───────────────────────────
MVP with shortcuts                          Clean Architecture from day 1
Minimal testing                             Full test coverage
Single database, no caching                 Distributed caching, read replicas
Hardcoded config values                     Configuration management
Console.WriteLine for logging               Structured logging + monitoring
"We'll refactor later"                      "Build it right the first time"
```

```
Technical Debt Curve:

Quality │
        │     "Build Right"
        │    ╱─────────────────────────── Sustainable pace
        │   ╱
        │  ╱
        │ ╱    "Ship Fast"
        │╱─────────╲
        │           ╲
        │            ╲──────── Slowdown from debt
        │                  ╲
        │                   ╲── Rewrite needed
        └──────────────────────────────────
                    Time →
```

### Trade-Off 6: Cost vs. Reliability

| Reliability | Architecture | Approximate Monthly Cost |
|---|---|---|
| 99% (7.3 hrs downtime/month) | Single server, no redundancy | $50-200 |
| 99.9% (43 min downtime/month) | Load balancer + 2 servers | $500-2,000 |
| 99.99% (4.3 min downtime/month) | Multi-AZ, auto-scaling, failover DB | $5,000-20,000 |
| 99.999% (26 sec downtime/month) | Multi-region, active-active, chaos engineering | $50,000+ |

```
Each additional "9" of availability roughly costs 10x more.

Cost ($) │
         │                                        ╱ 99.999%
         │                                      ╱
         │                                   ╱
         │                                ╱
         │                          ╱──
         │                     ╱──       99.99%
         │               ╱──
         │          ╱──              99.9%
         │     ╱──
         │──                     99%
         └──────────────────────────────────────
                    Availability →
```

### Trade-Off 7: Coupling vs. Autonomy (Data Management)

```
Shared Database                             Database-per-Service
(Strong Consistency, Tight Coupling)        (Loose Coupling, Eventual Consistency)
───────────────────────────                 ─────────────────────────────────────
┌──────────┐  ┌──────────┐                ┌──────────┐     ┌──────────┐
│ Order    │  │ Product  │                │ Order    │     │ Product  │
│ Service  │  │ Service  │                │ Service  │     │ Service  │
└────┬─────┘  └────┬─────┘                └────┬─────┘     └────┬─────┘
     │             │                           │                │
     ▼             ▼                           ▼                ▼
┌──────────────────────┐                ┌──────────┐     ┌──────────┐
│    Shared Database   │                │ OrderDB  │     │ProductDB │
│  (All tables here)   │                └──────────┘     └──────────┘
└──────────────────────┘

✅ Simple queries, JOINs              ✅ Independent deployment
✅ Strong consistency (ACID)          ✅ Scale databases independently
✅ No data duplication                ✅ Technology freedom per service
❌ Schema change = all services       ❌ No cross-service JOINs
❌ Single point of failure            ❌ Eventual consistency (Saga pattern)
❌ Can't scale independently          ❌ Data duplication may be needed
```

---

## 4. Trade-Off Analysis Framework — How to Decide

### Step 1: Identify the Top 3 Quality Attributes

Not all attributes matter equally. Prioritize based on business context.

```
Example: E-Commerce Platform

Must Have:     Availability, Performance, Security
Nice to Have:  Maintainability, Extensibility
Can Sacrifice: Simplicity (acceptable complexity for scale)

Example: Hospital Patient Records System

Must Have:     Security, Consistency, Reliability
Nice to Have:  Performance, Usability
Can Sacrifice: Scalability (limited user base)
```

### Step 2: Use an Architecture Trade-Off Matrix

Rate each option against your prioritized attributes (1-5 scale):

```
Attribute (Weight)        │ Monolith │ Microservices │ Modular Monolith │
──────────────────────────┼──────────┼───────────────┼──────────────────┤
Performance (3)           │    5     │      3        │        4         │
Scalability (5)           │    2     │      5        │        3         │
Maintainability (4)       │    3     │      4        │        5         │
Simplicity (2)            │    5     │      1        │        4         │
Fault Tolerance (4)       │    1     │      5        │        2         │
Cost (3)                  │    5     │      2        │        4         │
──────────────────────────┼──────────┼───────────────┼──────────────────┤
Weighted Score            │   63     │     73        │       76         │

Calculation:
Monolith:          (5×3)+(2×5)+(3×4)+(5×2)+(1×4)+(5×3) = 15+10+12+10+4+15 = 66
Microservices:     (3×3)+(5×5)+(4×4)+(1×2)+(5×4)+(2×3) = 9+25+16+2+20+6  = 78
Modular Monolith:  (4×3)+(3×5)+(5×4)+(4×2)+(2×4)+(4×3) = 12+15+20+8+8+12 = 75
```

### Step 3: Document Your Decision (Architecture Decision Record — ADR)

```markdown
# ADR-001: Use Microservices Architecture

## Status: Accepted

## Context
We are building an e-commerce platform expected to serve 100K+ concurrent
users with 5 independent development teams.

## Decision
We will use microservices architecture with one service per bounded context.

## Trade-offs Accepted
- ✅ Independent scaling (critical for flash sales)
- ✅ Team autonomy (5 teams can work in parallel)
- ✅ Fault isolation (payment failure won't break browsing)
- ❌ Increased operational complexity (need DevOps investment)
- ❌ Distributed data management (eventual consistency via Saga)
- ❌ Network latency for inter-service calls

## Alternatives Considered
1. Monolith — rejected: can't scale individual components
2. Modular Monolith — rejected: team coupling, single deployment

## Consequences
- Must invest in container orchestration (Kubernetes)
- Must implement distributed tracing (Jaeger/Zipkin)
- Must implement API Gateway for client communication
- Must use message broker (RabbitMQ) for async communication
```

---

## 5. Common Architecture Trade-Off Patterns

### Pattern 1: CQRS — Separate Read and Write Models

```
Problem: One model can't optimize for both reads and writes.

Trade-off: Complexity ↑ in exchange for Performance ↑ and Scalability ↑

                    ┌──────────────┐
                    │   Client     │
                    └──────┬───────┘
                           │
                    ┌──────┴───────┐
                    │  API Layer   │
                    └──────┬───────┘
                   ╱               ╲
          Command Side             Query Side
          (Write Model)            (Read Model)
                │                      │
         ┌──────┴──────┐        ┌──────┴──────┐
         │ Normalized   │  Sync  │ Denormalized│
         │ Write DB     │──────▶│ Read DB     │
         │ (SQL Server) │       │ (Redis/Mongo)│
         └─────────────┘        └─────────────┘

✅ Gains: Read performance (10x-100x), independent scaling
❌ Costs: Two models, sync complexity, eventual consistency
```

### Pattern 2: Event Sourcing — Store Events, Not State

```
Problem: Need complete audit trail + ability to rebuild state.

Trade-off: Storage ↑ + Complexity ↑ in exchange for Auditability ↑ + Flexibility ↑

Traditional (Store Current State):
┌───────────────────────────────────┐
│ Account: #12345                   │
│ Balance: $750                     │  ← Only the final state
└───────────────────────────────────┘

Event Sourcing (Store All Events):
┌───────────────────────────────────┐
│ AccountCreated    → Balance: $0   │
│ MoneyDeposited    → +$1000        │
│ MoneyWithdrawn    → -$200         │
│ MoneyDeposited    → +$500         │  ← Full history, replay anytime
│ MoneyWithdrawn    → -$550         │
│ Current Balance   = $750          │
└───────────────────────────────────┘

✅ Gains: Complete audit trail, temporal queries, replay/rebuild
❌ Costs: Storage growth, complex querying (need projections), eventual consistency
```

### Pattern 3: Saga Pattern — Distributed Transactions

```
Problem: Need transactions across multiple microservices (no distributed ACID).

Trade-off: Consistency ↓ (eventual) in exchange for Availability ↑ + Autonomy ↑

Choreography Saga (Event-Driven):
─────────────────────────────────
Order Service          Payment Service        Inventory Service
     │                       │                       │
     │ OrderCreated ─────────┼──────────────────────▶│
     │                       │                       │ StockReserved
     │                       │◀──────────────────────│
     │                       │ PaymentProcessed      │
     │◀──────────────────────│                       │
     │ OrderConfirmed        │                       │

Orchestration Saga (Central Coordinator):
──────────────────────────────────────────
                ┌──────────────────┐
                │  Order Saga      │
                │  Orchestrator    │
                └───────┬──────────┘
                        │
           ┌────────────┼────────────┐
           ▼            ▼            ▼
      Reserve       Process       Send
      Stock        Payment       Notification
           │            │            │
     (compensate   (compensate
      if fails)     if fails)

✅ Gains: Cross-service "transaction", each service autonomous
❌ Costs: Complexity, compensating transactions, eventual consistency
```

### Pattern 4: API Gateway — Single Entry Point

```
Problem: Clients need to talk to many microservices.

Trade-off: Latency ↑ (extra hop) in exchange for Security ↑ + Simplicity ↑ (for client)

Without Gateway:                    With Gateway:
Client knows all services           Client knows one endpoint

Client ──▶ User Service             Client ──▶ API Gateway ──▶ User Service
Client ──▶ Product Service                         │──▶ Product Service
Client ──▶ Order Service                           │──▶ Order Service
Client ──▶ Payment Service                         └──▶ Payment Service

✅ Gains: Centralized auth, rate limiting, request aggregation, SSL termination
❌ Costs: Single point of failure, extra network hop, added complexity
```

### Pattern 5: Cache-Aside — Trade Freshness for Speed

```
Problem: Database reads are slow for frequently accessed data.

Trade-off: Freshness ↓ (stale data risk) in exchange for Performance ↑ + Load ↓

    Client
      │
      ▼
  ┌────────┐  1. Check cache   ┌─────────┐
  │  App   │───────────────────▶│  Cache  │
  │ Server │                    │ (Redis) │
  │        │◀───────────────────│         │
  │        │  2a. Cache HIT     └─────────┘
  │        │     → return
  │        │
  │        │  2b. Cache MISS
  │        │───────────────────▶┌──────────┐
  │        │  3. Query DB       │ Database │
  │        │◀───────────────────│          │
  │        │  4. Store in cache └──────────┘
  │        │───────────────────▶ Cache
  └────────┘

✅ Gains: Sub-millisecond reads, reduce DB load by 80-90%
❌ Costs: Stale data (TTL-dependent), cache invalidation complexity, extra infrastructure
```

---

## 6. Real-World Trade-Off Decisions by Major Companies

| Company | Decision | Trade-Off Accepted |
|---|---|---|
| **Netflix** | Microservices + eventual consistency | Complexity ↑, but independent scaling for 200M+ users |
| **Amazon** | "Two-pizza teams" with service ownership | Data duplication ↑, but team autonomy ↑ |
| **Twitter** | Moved from Ruby monolith to JVM microservices | Rewrite cost ↑, but 10x performance improvement |
| **Shopify** | Stayed with modular monolith (Ruby on Rails) | Scalability limit, but simplicity + developer speed |
| **Uber** | Domain-oriented microservices (DOMA) | Platform investment ↑, but 2000+ services manageable |
| **Facebook** | PHP monolith + custom compiler (HHVM → Hack) | Unusual choice, but kept developer velocity |
| **Google** | Monorepo with billions of lines of code | Tooling investment ↑, but atomic changes + code sharing |
| **Basecamp** | "Majestic Monolith" — single Rails app | Scaling limits, but 15-person team ships fast |

---

## 7. Architecture Trade-Off Anti-Patterns (Mistakes to Avoid)

### Anti-Pattern 1: Resume-Driven Architecture

```
❌ "Let's use Kubernetes, microservices, event sourcing, and GraphQL"
    → Because it looks good on my resume

✅ "Our 3-person team building an internal tool should use a monolith"
    → Because it fits our context
```

### Anti-Pattern 2: Premature Optimization

```
❌ Building for 1 million users on day 1 when you have 100 users
❌ Implementing distributed caching before measuring actual DB load
❌ Splitting into microservices before understanding domain boundaries

✅ "Optimize when you have evidence, not when you have anxiety"
```

### Anti-Pattern 3: Ignoring Reversibility

```
Trade-Off Reversibility:

Easy to Reverse (Low Risk):
  ─ Caching strategy
  ─ Logging framework
  ─ CI/CD pipeline choice
  ─ Frontend framework

Hard to Reverse (High Risk):     ← Spend more time on these decisions
  ─ Programming language
  ─ Primary database
  ─ Architecture style (monolith vs. microservices)
  ─ Cloud provider
  ─ Communication protocol (sync vs. async)
```

### Anti-Pattern 4: Analysis Paralysis

```
❌ Spending 3 months debating Redis vs. Memcached for a startup MVP
❌ Evaluating 15 message brokers when any of the top 3 would work
❌ Designing for every possible future scenario

✅ "Make the decision reversible, then decide fast"
✅ "The best architecture is the one that allows you to defer decisions"
```

### Anti-Pattern 5: Ignoring Team Capabilities

```
Architecture must match team skills:

Team of 3 junior developers:
  ❌ Kubernetes + microservices + event sourcing + CQRS
  ✅ Monolith + simple deployment + SQL Server

Team of 50 experienced engineers:
  ❌ Single monolith (becomes bottleneck)
  ✅ Microservices + platform team + CI/CD automation
```

---

## 8. Decision Framework — Architecture Trade-Off Checklist

### Before Making a Decision, Ask:

```
1. CONTEXT
   □ What are the top 3 business goals?
   □ What is the expected user/data scale (now and in 2 years)?
   □ What is the team size and experience level?
   □ What is the budget and timeline?

2. TRADE-OFFS
   □ What do we GAIN with this choice?
   □ What do we LOSE or make harder?
   □ Can we live with the downsides?
   □ Is this decision reversible? At what cost?

3. ALTERNATIVES
   □ What are the other options?
   □ Why are we rejecting them?
   □ Is there a simpler approach that's "good enough"?

4. VALIDATION
   □ Have we prototyped or benchmarked?
   □ Are we solving a real problem or an imagined one?
   □ Does the team have the skills to execute this?

5. DOCUMENTATION
   □ Have we written an ADR (Architecture Decision Record)?
   □ Will someone in 6 months understand WHY we made this choice?
```

---

## 9. Trade-Off Summary Table — Quick Reference

| Trade-Off | Option A | Option B | Choose A When | Choose B When |
|---|---|---|---|---|
| **Performance vs. Maintainability** | Optimized code | Clean code | Real-time, high-frequency | CRUD apps, changing requirements |
| **Consistency vs. Availability** | CP (strong consistency) | AP (eventual consistency) | Financial, inventory | Social feeds, carts |
| **Scalability vs. Simplicity** | Microservices | Monolith | Large teams, high scale | Small teams, MVPs |
| **Security vs. Usability** | Strict controls | Frictionless UX | Banking, healthcare | Entertainment, social |
| **Speed vs. Quality** | Ship fast (tech debt) | Build right | Startup validation | Established product |
| **Cost vs. Reliability** | Single instance | Multi-region HA | Internal tools, dev/test | Revenue-critical production |
| **Coupling vs. Autonomy** | Shared database | DB-per-service | Small team, simple domain | Large team, complex domain |
| **Sync vs. Async** | HTTP (synchronous) | Message queue (async) | Need immediate response | Fire-and-forget, resilience |
| **Build vs. Buy** | Custom solution | Third-party/SaaS | Core differentiator | Commodity feature |
| **Caching vs. Freshness** | Aggressive caching | Always-fresh reads | Read-heavy, tolerance for staleness | Write-heavy, must be current |

---

## 10. The "Good Enough" Principle

```
                    ┌───────────────────────────────┐
                    │                               │
  ❌ Over-          │     ✅ "Good Enough"           │     ❌ Under-
  Engineered        │        Architecture            │     Engineered
                    │                               │
  Kubernetes for    │  Matches business needs        │  No error handling
  a blog            │  Team can maintain it          │  No logging
  Microservices     │  Affordable to run             │  No security
  for 100 users     │  Can evolve as needs change    │  Single point of failure
                    │                               │
                    └───────────────────────────────┘

"The goal is not the best architecture.
 The goal is the least worst architecture for YOUR context."
```

### Final Decision Heuristic

```
1. Start simple (monolith, single DB, synchronous calls)
2. Measure (observe bottlenecks with real data)
3. Split only where needed (strangler fig pattern)
4. Document every major decision (ADRs)
5. Revisit trade-offs as context changes (quarterly)
```

---
---

# PII Data Security in Software Architecture Design

---

## 1. What Is PII (Personally Identifiable Information)?

**PII** is any data that can be used — alone or combined with other data — to **identify, contact, or locate** a specific individual.

### PII Classification

```
┌──────────────────────────────────────────────────────────────────────┐
│                         PII Data Categories                         │
├──────────────────────────────┬───────────────────────────────────────┤
│    Direct Identifiers         │    Quasi-Identifiers (Indirect)      │
│    (Identify alone)           │    (Identify when combined)          │
├──────────────────────────────┼───────────────────────────────────────┤
│  • Full name                  │  • Date of birth                     │
│  • Social Security Number     │  • ZIP / Postal code                 │
│  • Passport number            │  • Gender                            │
│  • Driver's license number    │  • Race / Ethnicity                  │
│  • Email address              │  • Job title                         │
│  • Phone number               │  • Employer name                     │
│  • Credit card number         │  • IP address                        │
│  • Bank account number        │  • Device fingerprint                │
│  • Biometric data             │  • Browser cookies                   │
│  • Medical record number      │  • Purchase history                  │
│  • Aadhaar number (India)     │  • Location data                     │
│  • PAN number (India)         │  • Login timestamps                  │
└──────────────────────────────┴───────────────────────────────────────┘
```

### Sensitivity Levels

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  🔴 HIGH SENSITIVITY (Regulatory / Financial / Health)       │
│     SSN, Aadhaar, PAN, passport, biometrics,                │
│     medical records, credit card, bank account               │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  🟡 MEDIUM SENSITIVITY (Personal Contact)                    │
│     Full name, email, phone, address, date of birth,         │
│     employment info                                          │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  🟢 LOW SENSITIVITY (Behavioral / Indirect)                  │
│     IP address, cookies, preferences, purchase history,      │
│     device info                                              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. Why PII Security Matters in Architecture

### Consequences of PII Breach

| Impact | Example |
|---|---|
| **Financial penalties** | GDPR fines up to €20M or 4% of global revenue |
| **Lawsuits** | Class-action from affected users |
| **Reputation damage** | Customer trust destroyed — users leave |
| **Regulatory action** | Business license revoked, mandated audits |
| **Criminal liability** | Officers/developers can face charges in some jurisdictions |

### Real-World Breach Costs

```
Equifax (2017):    148M records → $700M settlement
Marriott (2018):   500M records → $124M GDPR fine
Facebook (2019):   533M records → $5B FTC fine
Uber (2016):       57M records  → $148M settlement
Capital One (2019): 100M records → $190M settlement
```

### Key Regulations

| Regulation | Region | Key PII Requirements |
|---|---|---|
| **GDPR** | EU | Consent, right to erasure, breach notification (72 hrs), data minimization |
| **CCPA/CPRA** | California, US | Right to know, delete, opt-out of sale |
| **HIPAA** | US (Healthcare) | Protected Health Information (PHI) encryption, access controls |
| **PCI DSS** | Global (Payments) | Credit card data encryption, tokenization, network segmentation |
| **IT Act / DPDP** | India | Consent, data localization, breach notification |
| **PDPA** | Singapore | Consent, purpose limitation, data protection officers |
| **LGPD** | Brazil | Similar to GDPR — consent, minimization, breach notification |

---

## 3. PII Security Principles for Architecture

### The 7 Pillars of PII Protection

```
┌─────────────────────────────────────────────────────────────────┐
│                 7 Pillars of PII Protection                     │
├─────────────────────┬───────────────────────────────────────────┤
│ 1. Data Minimization│ Collect only what you absolutely need     │
│ 2. Encryption       │ Encrypt at rest AND in transit            │
│ 3. Access Control   │ Least privilege — need-to-know basis      │
│ 4. Anonymization    │ Remove/mask PII when full data not needed │
│ 5. Audit Logging    │ Track who accessed what PII and when      │
│ 6. Data Retention   │ Delete PII when purpose is fulfilled      │
│ 7. Breach Response  │ Detect, contain, notify within SLA        │
└─────────────────────┴───────────────────────────────────────────┘
```

---

## 4. Architecture Strategies for PII Protection

### Strategy 1: Data Minimization — Collect Less, Risk Less

```
❌ WRONG: Collect everything "just in case"
────────────────────────────────────────
Registration form:
  Full Name, Email, Phone, DOB, Gender, SSN,
  Mother's Maiden Name, Pet's Name, Shoe Size...

✅ RIGHT: Collect only what the feature requires
────────────────────────────────────────
Registration form:
  Email, Password (that's it for signup)
  Name, Address → only when placing first order
  PAN/Tax ID → only when legally required
```

```csharp
// ❌ BAD — Storing unnecessary PII
public class User
{
    public string FullName { get; set; }
    public string Email { get; set; }
    public string SSN { get; set; }           // Why does an e-commerce site need SSN?
    public string MotherMaidenName { get; set; } // Security question anti-pattern
    public DateTime DateOfBirth { get; set; }     // Needed only for age verification
    public string CreditCardNumber { get; set; }  // NEVER store full card numbers
}

// ✅ GOOD — Minimal PII, collected progressively
public class User
{
    public Guid Id { get; set; }
    public string Email { get; set; } = "";
    public string PasswordHash { get; set; } = "";  // Never store plain passwords
    public string? FirstName { get; set; }           // Collected when needed
    public string? PhoneNumber { get; set; }         // Collected for 2FA opt-in only
}
```

### Strategy 2: Encryption — At Rest and In Transit

```
┌─────────────────────────────────────────────────────────────────┐
│                    Encryption Strategy                           │
├─────────────────────────────┬───────────────────────────────────┤
│   In Transit                 │   At Rest                         │
│   (Network)                  │   (Storage)                       │
├─────────────────────────────┼───────────────────────────────────┤
│ • TLS 1.2+ for all APIs     │ • AES-256 for database columns   │
│ • HTTPS enforced (no HTTP)   │ • Transparent Data Encryption    │
│ • mTLS for service-to-service│   (TDE) for full database        │
│ • Certificate pinning (mobile)│ • Encrypted backups              │
│ • Encrypt message queues     │ • Encrypted file storage          │
│ • VPN for internal traffic   │ • Key management (Azure Key Vault,│
│                              │   AWS KMS, HashiCorp Vault)       │
└─────────────────────────────┴───────────────────────────────────┘
```

#### Column-Level Encryption in EF Core

```csharp
// Encrypt sensitive fields before saving to database
public class EncryptionService : IEncryptionService
{
    private readonly byte[] _key;
    private readonly byte[] _iv;

    public EncryptionService(IOptions<EncryptionSettings> settings)
    {
        _key = Convert.FromBase64String(settings.Value.Key);
        _iv = Convert.FromBase64String(settings.Value.IV);
    }

    public string Encrypt(string plainText)
    {
        using var aes = Aes.Create();
        aes.Key = _key;
        aes.IV = _iv;

        using var encryptor = aes.CreateEncryptor();
        var plainBytes = Encoding.UTF8.GetBytes(plainText);
        var encryptedBytes = encryptor.TransformFinalBlock(plainBytes, 0, plainBytes.Length);

        return Convert.ToBase64String(encryptedBytes);
    }

    public string Decrypt(string cipherText)
    {
        using var aes = Aes.Create();
        aes.Key = _key;
        aes.IV = _iv;

        using var decryptor = aes.CreateDecryptor();
        var cipherBytes = Convert.FromBase64String(cipherText);
        var plainBytes = decryptor.TransformFinalBlock(cipherBytes, 0, cipherBytes.Length);

        return Encoding.UTF8.GetString(plainBytes);
    }
}
```

#### EF Core Value Converter for Automatic Encryption

```csharp
// Automatically encrypt/decrypt PII fields when reading/writing to DB
public class EncryptedConverter : ValueConverter<string, string>
{
    public EncryptedConverter(IEncryptionService encryptionService)
        : base(
            v => encryptionService.Encrypt(v),       // Writing to DB → encrypt
            v => encryptionService.Decrypt(v))        // Reading from DB → decrypt
    { }
}

// Apply to entity configuration
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    private readonly IEncryptionService _encryption;

    public UserConfiguration(IEncryptionService encryption)
    {
        _encryption = encryption;
    }

    public void Configure(EntityTypeBuilder<User> builder)
    {
        var converter = new EncryptedConverter(_encryption);

        builder.Property(u => u.Email)
            .HasConversion(converter);       // Email encrypted at rest

        builder.Property(u => u.PhoneNumber)
            .HasConversion(converter);       // Phone encrypted at rest

        builder.Property(u => u.SSN)
            .HasConversion(converter);       // SSN encrypted at rest
    }
}
```

#### Always Encrypted (SQL Server — Database-Level)

```sql
-- SQL Server Always Encrypted: data encrypted in DB, decrypted only by app
-- Even DBAs cannot read the plaintext

CREATE TABLE Users (
    Id UNIQUEIDENTIFIER PRIMARY KEY,
    Email NVARCHAR(256)
        ENCRYPTED WITH (
            COLUMN_ENCRYPTION_KEY = CEK_Auto1,
            ENCRYPTION_TYPE = DETERMINISTIC,       -- Allows equality comparisons
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        ),
    SSN NVARCHAR(11)
        ENCRYPTED WITH (
            COLUMN_ENCRYPTION_KEY = CEK_Auto1,
            ENCRYPTION_TYPE = RANDOMIZED,          -- Maximum security, no comparisons
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        )
);
```

```
Encryption Type Comparison:

Deterministic Encryption:
  Same plaintext → same ciphertext (always)
  ✅ Allows: equality checks, JOINs, GROUP BY, indexing
  ❌ Pattern leakage: attacker can see which rows have same value
  Use for: Email (need to search by email)

Randomized Encryption:
  Same plaintext → different ciphertext (each time)
  ✅ Maximum security: no pattern leakage
  ❌ Cannot: search, filter, join, index on this column
  Use for: SSN, medical records, passwords (no need to search)
```

### Strategy 3: Tokenization — Replace PII with Non-Sensitive Tokens

```
Tokenization is different from encryption:
  Encryption: Reversible with a key (ciphertext → plaintext)
  Tokenization: Lookup table / vault (token → original value)

Original Data              Tokenized Data              Token Vault (Secure)
──────────────             ──────────────              ────────────────────
Card: 4111-1111-1111-1111  Card: tok_a8f3k2           tok_a8f3k2 → 4111...1111
SSN:  123-45-6789          SSN:  tok_m9x2p7           tok_m9x2p7 → 123-45-6789
Email: alice@example.com   Email: tok_q3w8z1          tok_q3w8z1 → alice@...

The application works with tokens.
Only the Token Vault can resolve tokens back to real values.
Even if the main database is breached, attackers get meaningless tokens.
```

```csharp
// Tokenization Service
public interface ITokenizationService
{
    Task<string> TokenizeAsync(string sensitiveValue, string dataType);
    Task<string> DetokenizeAsync(string token);
}

public class TokenizationService : ITokenizationService
{
    private readonly ITokenVaultClient _vault;  // Secure external vault

    public TokenizationService(ITokenVaultClient vault)
    {
        _vault = vault;
    }

    public async Task<string> TokenizeAsync(string sensitiveValue, string dataType)
    {
        // Store in vault, get back a token
        var token = await _vault.StoreAsync(sensitiveValue, dataType);
        return token;  // e.g., "tok_a8f3k2m9x2"
    }

    public async Task<string> DetokenizeAsync(string token)
    {
        // Retrieve from vault (requires authorization)
        return await _vault.RetrieveAsync(token);
    }
}

// Usage in Payment Service
public class PaymentService
{
    private readonly ITokenizationService _tokenizer;

    public async Task<Payment> ProcessPaymentAsync(PaymentRequest request)
    {
        // Tokenize the card number — never store the real one
        var cardToken = await _tokenizer.TokenizeAsync(
            request.CardNumber, "credit_card");

        var payment = new Payment
        {
            CardToken = cardToken,                // Store only the token
            CardLast4 = request.CardNumber[^4..], // Store last 4 for display
            CardType = DetectCardType(request.CardNumber),
            Amount = request.Amount
        };

        // Process payment using token
        // The payment gateway resolves the token to the real card
        await _gateway.ChargeAsync(cardToken, request.Amount);

        return payment;
    }
}
```

### Strategy 4: Data Masking — Show Partial PII

```
Full PII (Internal/Admin)     Masked PII (Displayed to User/Logs)
─────────────────────────     ──────────────────────────────────
alice@example.com             a***e@example.com
+91-9876543210                +91-****3210
4111-1111-1111-1111           ****-****-****-1111
123-45-6789 (SSN)             ***-**-6789
Rahul Kumar                   R***l K***r
```

```csharp
// Data Masking Utility
public static class PiiMasker
{
    public static string MaskEmail(string email)
    {
        if (string.IsNullOrEmpty(email)) return "";
        var parts = email.Split('@');
        if (parts.Length != 2) return "***@***";

        var local = parts[0];
        var masked = local.Length <= 2
            ? "***"
            : $"{local[0]}***{local[^1]}";

        return $"{masked}@{parts[1]}";
    }

    public static string MaskPhone(string phone)
    {
        if (string.IsNullOrEmpty(phone) || phone.Length < 4) return "****";
        return $"{"".PadLeft(phone.Length - 4, '*')}{phone[^4..]}";
    }

    public static string MaskCreditCard(string cardNumber)
    {
        if (string.IsNullOrEmpty(cardNumber) || cardNumber.Length < 4) return "****";
        return $"****-****-****-{cardNumber[^4..]}";
    }

    public static string MaskName(string name)
    {
        if (string.IsNullOrEmpty(name) || name.Length <= 2) return "***";
        return $"{name[0]}{"".PadLeft(name.Length - 2, '*')}{name[^1]}";
    }
}

// Usage
var masked = PiiMasker.MaskEmail("alice@example.com");  // "a***e@example.com"
var masked = PiiMasker.MaskPhone("+919876543210");       // "********3210"
```

#### Masking in API Responses (DTOs)

```csharp
// Return masked PII in public-facing DTOs
public class UserProfileDto
{
    public Guid Id { get; set; }
    public string FullName { get; set; } = "";
    public string MaskedEmail { get; set; } = "";      // a***e@example.com
    public string MaskedPhone { get; set; } = "";       // ****3210
}

// Mapping — mask during DTO creation
public UserProfileDto MapToPublicDto(User user)
{
    return new UserProfileDto
    {
        Id = user.Id,
        FullName = user.FirstName + " " + user.LastName,
        MaskedEmail = PiiMasker.MaskEmail(user.Email),
        MaskedPhone = PiiMasker.MaskPhone(user.PhoneNumber)
    };
}
```

### Strategy 5: Anonymization & Pseudonymization

```
┌────────────────────────────────────────────────────────────────────┐
│  Technique       │  Description                │  Reversible?      │
├──────────────────┼─────────────────────────────┼───────────────────┤
│  Anonymization   │  Permanently remove PII     │  ❌ No (one-way)  │
│                  │  Cannot re-identify          │                   │
├──────────────────┼─────────────────────────────┼───────────────────┤
│  Pseudonymization│  Replace PII with pseudonym │  ✅ Yes (with key)│
│                  │  Can re-identify with key    │                   │
├──────────────────┼─────────────────────────────┼───────────────────┤
│  Masking         │  Hide parts of PII          │  ❌ No (partial)  │
│                  │  Show only what's needed     │                   │
└──────────────────┴─────────────────────────────┴───────────────────┘
```

```csharp
// Anonymization — for analytics databases
// Replace real PII with irreversible values
public class AnonymizationService
{
    public AnonymizedUser Anonymize(User user)
    {
        return new AnonymizedUser
        {
            // Hash the ID — can group records but can't reverse to original
            AnonymousId = ComputeSha256Hash(user.Id.ToString()),

            // Keep non-PII for analytics
            AgeGroup = GetAgeGroup(user.DateOfBirth),   // "25-34" instead of exact DOB
            Region = GetRegion(user.PostalCode),         // "North India" instead of address
            AccountCreatedMonth = user.CreatedAt.ToString("yyyy-MM"),

            // Remove all direct identifiers
            // No name, no email, no phone, no address
        };
    }

    private string GetAgeGroup(DateTime dob)
    {
        var age = DateTime.UtcNow.Year - dob.Year;
        return age switch
        {
            < 18 => "Under 18",
            < 25 => "18-24",
            < 35 => "25-34",
            < 45 => "35-44",
            < 55 => "45-54",
            _ => "55+"
        };
    }
}

// Pseudonymization — for dev/test databases
public class PseudonymizationService
{
    public void PseudonymizeForTesting(User user)
    {
        user.FirstName = $"User_{Guid.NewGuid().ToString()[..8]}";
        user.LastName = "TestUser";
        user.Email = $"test_{Guid.NewGuid():N}@example.com";
        user.PhoneNumber = $"+1555{Random.Shared.Next(1000000, 9999999)}";
        user.PasswordHash = BCrypt.Net.BCrypt.HashPassword("TestPassword123!");
    }
}
```

---

## 5. Access Control for PII

### Principle of Least Privilege

```
Role-Based Access to PII:

┌─────────────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│  PII Field       │ Customer │ Support  │ Manager  │ DBA      │ Admin    │
├─────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ Own Email        │ ✅ Full  │ ✅ Masked│ ✅ Masked│ ❌       │ ✅ Full  │
│ Own Phone        │ ✅ Full  │ ✅ Masked│ ✅ Masked│ ❌       │ ✅ Full  │
│ Own Address      │ ✅ Full  │ ✅ Full  │ ✅ Full  │ ❌       │ ✅ Full  │
│ Other's Email    │ ❌       │ ✅ Masked│ ✅ Full  │ ❌       │ ✅ Full  │
│ Payment Card     │ ✅ Last 4│ ✅ Last 4│ ✅ Last 4│ ❌       │ ✅ Last 4│
│ SSN / Aadhaar    │ ❌       │ ❌       │ ❌       │ ❌       │ ✅ Full  │
│ Raw DB Access    │ ❌       │ ❌       │ ❌       │ ✅*      │ ✅       │
└─────────────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
* DBA sees encrypted columns — cannot decrypt without app-level keys
```

### Field-Level Authorization in ASP.NET Core

```csharp
// Different DTO projections based on role
public class UserResponseFactory
{
    public object CreateResponse(User user, ClaimsPrincipal currentUser)
    {
        // Admin sees everything
        if (currentUser.IsInRole("Admin"))
        {
            return new AdminUserDto
            {
                Id = user.Id,
                Email = user.Email,
                Phone = user.PhoneNumber,
                FullName = $"{user.FirstName} {user.LastName}",
                SSN = user.SSN,
                Addresses = user.Addresses.Select(MapAddress).ToList(),
                CreatedAt = user.CreatedAt,
                LastLoginAt = user.LastLoginAt
            };
        }

        // Support sees masked PII
        if (currentUser.IsInRole("Support"))
        {
            return new SupportUserDto
            {
                Id = user.Id,
                MaskedEmail = PiiMasker.MaskEmail(user.Email),
                MaskedPhone = PiiMasker.MaskPhone(user.PhoneNumber),
                FirstName = user.FirstName,
                // No SSN, no full email, no full phone
            };
        }

        // Customer sees only their own data
        return new CustomerUserDto
        {
            Id = user.Id,
            Email = user.Email,
            Phone = user.PhoneNumber,
            FirstName = user.FirstName,
            LastName = user.LastName
        };
    }
}
```

---

## 6. PII in Logs — The Silent Leak

### The Problem

```
❌ DANGEROUS — PII leaking into logs

logger.LogInformation("User registered: {Email}, {Phone}", user.Email, user.Phone);
// Log output: "User registered: alice@example.com, +919876543210"

logger.LogError("Payment failed for card {CardNumber}", request.CardNumber);
// Log output: "Payment failed for card 4111111111111111"

// These logs go to:
//   → Console → captured by container runtime
//   → Files → accessible to ops team
//   → Centralized logging (ELK, Seq, Application Insights)
//   → Error tracking (Sentry, Raygun)
//   → ALL are potential breach vectors
```

### The Solution: PII-Safe Logging

```csharp
// ✅ SAFE — Never log raw PII

// Option 1: Log IDs, not PII
logger.LogInformation("User registered: {UserId}", user.Id);

// Option 2: Log masked values
logger.LogInformation("User registered: {MaskedEmail}", PiiMasker.MaskEmail(user.Email));

// Option 3: Use structured logging with PII redaction
logger.LogInformation("Payment failed for card ending {Last4}", request.CardNumber[^4..]);
```

#### Automatic PII Redaction Middleware for Serilog

```csharp
// Serilog destructuring policy — automatically mask PII properties
public class PiiDestructuringPolicy : IDestructuringPolicy
{
    private static readonly HashSet<string> PiiProperties = new(StringComparer.OrdinalIgnoreCase)
    {
        "Email", "Phone", "PhoneNumber", "SSN", "SocialSecurityNumber",
        "CardNumber", "CreditCard", "Password", "PasswordHash",
        "DateOfBirth", "DOB", "Address", "Aadhaar", "PAN"
    };

    public bool TryDestructure(
        object value, ILogEventPropertyValueFactory factory, out LogEventPropertyValue? result)
    {
        result = null;
        if (value is null) return false;

        var type = value.GetType();
        if (type.IsPrimitive || type == typeof(string)) return false;

        var properties = new List<LogEventProperty>();
        foreach (var prop in type.GetProperties())
        {
            var propValue = prop.GetValue(value);
            if (PiiProperties.Contains(prop.Name))
            {
                // Redact PII fields
                properties.Add(new LogEventProperty(
                    prop.Name, factory.CreatePropertyValue("[REDACTED]")));
            }
            else
            {
                properties.Add(new LogEventProperty(
                    prop.Name, factory.CreatePropertyValue(propValue, destructureObjects: true)));
            }
        }

        result = new StructureValue(properties);
        return true;
    }
}

// Registration
Log.Logger = new LoggerConfiguration()
    .Destructure.With<PiiDestructuringPolicy>()
    .WriteTo.Console()
    .CreateLogger();

// Now this is safe:
logger.LogInformation("User registered: {@User}", user);
// Output: "User registered: { Id: guid, FirstName: Alice, Email: [REDACTED], Phone: [REDACTED] }"
```

---

## 7. PII in Microservices Architecture

### Challenge: PII Scattered Across Services

```
User Service:     Email, Phone, Name, Address, SSN
Order Service:    Shipping Name, Shipping Address, Phone
Payment Service:  Card Number, Billing Address, Email
Notification:     Email, Phone (for sending)
Analytics:        User behavior + potentially PII

Problem: PII is duplicated across 5 databases.
         A breach in ANY service exposes PII.
```

### Solution: PII Vault / Centralized PII Service

```
                         ┌──────────────────┐
                         │    PII Vault      │
                         │  (User Service)   │
                         │                   │
                         │ Stores ALL PII:   │
                         │  - Email          │
                         │  - Phone          │
                         │  - Name           │
                         │  - Address        │
                         │  - SSN            │
                         │                   │
                         │ Returns: Tokens   │
                         │  + Masked views   │
                         └────────┬──────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
              ▼                   ▼                   ▼
     ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
     │ Order Service │   │ Payment Svc  │   │ Notification │
     │               │   │              │   │   Service    │
     │ Stores only:  │   │ Stores only: │   │              │
     │ - UserId      │   │ - UserId     │   │ Asks PII     │
     │ - AddressToken│   │ - CardToken  │   │ Vault for    │
     │               │   │ - Last4      │   │ email/phone  │
     │ NO raw PII    │   │ NO raw PII   │   │ at send time │
     └──────────────┘   └──────────────┘   └──────────────┘
```

```csharp
// Order Service — stores tokens, not PII
public class Order
{
    public Guid Id { get; set; }
    public Guid UserId { get; set; }                  // Reference to User Service
    public string ShippingAddressToken { get; set; }   // Token from PII Vault
    // NO shipping name, NO address, NO phone stored here
}

// When displaying order to customer:
public async Task<OrderDetailDto> GetOrderDetailAsync(Guid orderId)
{
    var order = await _orderRepository.GetByIdAsync(orderId);

    // Resolve PII from vault at display time
    var address = await _piiVault.DetokenizeAddressAsync(order.ShippingAddressToken);
    var user = await _userClient.GetContactInfoAsync(order.UserId);

    return new OrderDetailDto
    {
        OrderNumber = order.OrderNumber,
        CustomerName = $"{user.FirstName} {user.LastName}",
        ShippingAddress = address,
        Items = order.Items.Select(MapItem).ToList()
    };
}
```

---

## 8. Data Retention & Right to Erasure (GDPR Article 17)

### Retention Policy

```
Data Type              Retention Period        After Expiry
─────────────          ────────────────        ─────────────
Account data           Active + 30 days        Anonymize or delete
Order history          7 years (tax law)       Anonymize PII, keep transaction
Payment records        7 years (financial law)  Delete card tokens, keep amount
Support tickets        3 years                 Anonymize customer details
Login/activity logs    1 year                  Delete
Marketing preferences  Until consent withdrawn Delete
Abandoned carts        30 days                 Delete
```

### Right to Erasure Implementation

```csharp
// User requests account deletion → must erase PII across ALL services

public class UserDeletionService
{
    private readonly IUserRepository _userRepo;
    private readonly IOrderServiceClient _orderClient;
    private readonly IPaymentServiceClient _paymentClient;
    private readonly INotificationServiceClient _notificationClient;

    public async Task ProcessDeletionRequestAsync(Guid userId)
    {
        // 1. Anonymize orders (keep for tax compliance, remove PII)
        await _orderClient.AnonymizeUserOrdersAsync(userId);

        // 2. Delete payment tokens
        await _paymentClient.DeleteUserPaymentDataAsync(userId);

        // 3. Delete notification history
        await _notificationClient.DeleteUserNotificationsAsync(userId);

        // 4. Anonymize the user record (don't hard-delete for referential integrity)
        var user = await _userRepo.GetByIdAsync(userId);
        if (user is not null)
        {
            user.FirstName = "Deleted";
            user.LastName = "User";
            user.Email = $"deleted_{userId}@anonymized.local";
            user.PhoneNumber = null;
            user.PasswordHash = "";
            user.Status = AccountStatus.Deleted;
            user.Addresses.Clear();
            user.RefreshTokens.Clear();

            await _userRepo.SaveChangesAsync();
        }

        // 5. Log the deletion (without PII)
        _logger.LogInformation("User {UserId} data erasure completed", userId);
    }
}
```

### Automated Retention Job

```csharp
// Background service to enforce retention policies
public class DataRetentionJob : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = _scopeFactory.CreateScope();
            var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();

            // Delete abandoned carts older than 30 days
            var staleCartsCutoff = DateTime.UtcNow.AddDays(-30);
            await dbContext.Carts
                .Where(c => c.UpdatedAt < staleCartsCutoff)
                .ExecuteDeleteAsync(stoppingToken);

            // Delete activity logs older than 1 year
            var logsCutoff = DateTime.UtcNow.AddYears(-1);
            await dbContext.UserActivities
                .Where(a => a.Timestamp < logsCutoff)
                .ExecuteDeleteAsync(stoppingToken);

            // Delete expired refresh tokens
            await dbContext.RefreshTokens
                .Where(t => t.ExpiresAt < DateTime.UtcNow.AddDays(-7))
                .ExecuteDeleteAsync(stoppingToken);

            // Run daily at 2 AM
            await Task.Delay(TimeSpan.FromHours(24), stoppingToken);
        }
    }
}
```

---

## 9. PII Security Audit Trail

### What to Log (Without Logging PII)

```
✅ Log:                           ❌ Never Log:
─ Who accessed PII (user ID)      ─ The PII values themselves
─ What type of PII was accessed   ─ Passwords (even hashed)
─ When it was accessed            ─ Full card numbers
─ Why (API endpoint / action)     ─ SSN / Aadhaar numbers
─ From where (IP, service name)   ─ Session tokens / JWTs
─ Success or failure              ─ Encryption keys
```

```csharp
// PII Access Audit Log
public class PiiAccessLog
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public Guid AccessedByUserId { get; set; }
    public string AccessedByRole { get; set; } = "";
    public Guid TargetUserId { get; set; }            // Whose PII was accessed
    public string PiiFieldAccessed { get; set; } = ""; // "Email", "Phone", "SSN"
    public string Action { get; set; } = "";           // "View", "Export", "Modify", "Delete"
    public string Endpoint { get; set; } = "";         // "/api/users/123"
    public string IpAddress { get; set; } = "";
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
    public bool WasAuthorized { get; set; }
}
```

```csharp
// Middleware to automatically log PII access
public class PiiAuditMiddleware
{
    private readonly RequestDelegate _next;
    private static readonly HashSet<string> PiiEndpoints = new()
    {
        "/api/users",
        "/api/users/{id}",
        "/api/users/{id}/addresses",
        "/api/internal/users"
    };

    public async Task InvokeAsync(HttpContext context, IPiiAuditService auditService)
    {
        var path = context.Request.Path.Value ?? "";
        var isPiiEndpoint = PiiEndpoints.Any(e => path.StartsWith(
            e.Split('{')[0], StringComparison.OrdinalIgnoreCase));

        if (isPiiEndpoint && context.User.Identity?.IsAuthenticated == true)
        {
            await auditService.LogAccessAsync(new PiiAccessLog
            {
                AccessedByUserId = Guid.Parse(context.User.FindFirst("sub")?.Value ?? ""),
                AccessedByRole = context.User.FindFirst(ClaimTypes.Role)?.Value ?? "",
                Endpoint = path,
                Action = context.Request.Method,
                IpAddress = context.Connection.RemoteIpAddress?.ToString() ?? "",
                WasAuthorized = true
            });
        }

        await _next(context);
    }
}
```

---

## 10. PII Security Architecture — Complete Design

```
┌───────────────────────────────────────────────────────────────────────┐
│                    PII Security Architecture                         │
│                                                                       │
│  Client (Browser/Mobile)                                              │
│     │  HTTPS (TLS 1.3)                                               │
│     ▼                                                                 │
│  ┌──────────────────┐                                                │
│  │   API Gateway     │ ← Rate limiting, JWT validation, IP filtering │
│  └────────┬─────────┘                                                │
│           │                                                           │
│  ┌────────┴──────────────────────────────────────────────────┐       │
│  │                   Application Services                     │       │
│  │                                                            │       │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │       │
│  │  │ User Service │  │Order Service│  │Payment Svc  │       │       │
│  │  │             │  │             │  │             │       │       │
│  │  │ PII Vault:  │  │ Tokens only │  │ Tokens only │       │       │
│  │  │ Encrypted   │  │ No raw PII  │  │ No raw PII  │       │       │
│  │  │ at rest     │  │             │  │             │       │       │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘       │       │
│  │         │                │                │               │       │
│  │  ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐       │       │
│  │  │  UserDB     │  │  OrderDB    │  │ PaymentDB   │       │       │
│  │  │ (Encrypted) │  │ (No PII)   │  │ (No PII)   │       │       │
│  │  │ AES-256     │  │             │  │             │       │       │
│  │  └─────────────┘  └─────────────┘  └─────────────┘       │       │
│  └───────────────────────────────────────────────────────────┘       │
│                                                                       │
│  Cross-Cutting Concerns:                                              │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │ • PII-safe logging (auto-redaction)                             │ │
│  │ • Audit trail (who accessed what PII, when, why)                │ │
│  │ • Data retention policies (auto-delete expired PII)             │ │
│  │ • Right to erasure (cross-service PII deletion)                 │ │
│  │ • Key management (Azure Key Vault / AWS KMS)                    │ │
│  │ • Breach detection (anomaly alerts on PII access patterns)      │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 11. PII Security Checklist for Architects

```
DATA COLLECTION
  □ Collect only necessary PII (data minimization)
  □ Get explicit consent before collecting PII
  □ Document purpose for each PII field collected
  □ Provide privacy policy explaining data usage

STORAGE
  □ Encrypt PII at rest (AES-256 or Always Encrypted)
  □ Tokenize payment card data (PCI DSS)
  □ Store PII in a centralized vault (not scattered across services)
  □ Use separate encryption keys per sensitivity level
  □ Rotate encryption keys periodically

TRANSIT
  □ TLS 1.2+ for all external communication
  □ mTLS for service-to-service calls carrying PII
  □ Never pass PII in URL query strings
  □ Encrypt PII fields in message queues

ACCESS CONTROL
  □ Role-based access to PII (least privilege)
  □ Field-level authorization (mask PII based on role)
  □ Break-glass procedure for emergency PII access
  □ Regular access reviews (quarterly)

LOGGING & MONITORING
  □ Never log raw PII values
  □ Auto-redact PII in log output
  □ Audit trail for all PII access
  □ Alerts for unusual PII access patterns (bulk export, off-hours)

DATA LIFECYCLE
  □ Define retention period for each PII category
  □ Automated deletion of expired PII
  □ Right to erasure (GDPR Article 17) — cross-service deletion
  □ Anonymize PII for analytics (never use real PII)
  □ Pseudonymize PII for dev/test environments

INCIDENT RESPONSE
  □ Breach notification plan (72 hours under GDPR)
  □ Forensic logging to determine breach scope
  □ Ability to revoke all tokens / sessions on breach
  □ Regular penetration testing on PII-handling endpoints
  □ Tabletop exercises for PII breach scenarios
```

---

## 12. Quick Reference — PII Protection Techniques

| Technique | What It Does | Reversible? | Use When |
|---|---|---|---|
| **Encryption** | Scramble data with a key | ✅ Yes (with key) | Storing PII you need to read back |
| **Hashing** | One-way transform | ❌ No | Passwords, integrity checks |
| **Tokenization** | Replace with random token | ✅ Yes (via vault) | Credit cards, SSNs stored externally |
| **Masking** | Show partial data | ❌ No | Displaying PII to support staff / UI |
| **Anonymization** | Permanently remove identity | ❌ No | Analytics, data warehousing |
| **Pseudonymization** | Replace with fake but consistent data | ✅ Yes (with mapping) | Dev/test environments |
| **Redaction** | Remove entirely from output | ❌ No | Logs, error messages, exports |

---
---

# Business Continuity Plan (BCP) in Software Architecture Design

---

## 1. What Is BCP (Business Continuity Planning)?

**BCP** is a proactive strategy that ensures **critical business operations continue** during and after a disaster — whether it's a server crash, data center outage, cyberattack, natural disaster, or even a key-person departure.

```
┌──────────────────────────────────────────────────────────────────────┐
│                  BCP vs DR vs HA — Key Differences                   │
├──────────────────┬───────────────────────────────────────────────────┤
│  BCP             │ Business strategy — keeps the entire BUSINESS     │
│  (Business       │ running. Covers people, processes, technology,    │
│   Continuity)    │ communication, and facilities.                    │
├──────────────────┼───────────────────────────────────────────────────┤
│  DR              │ Technology strategy — recovers IT SYSTEMS after   │
│  (Disaster       │ failure. Covers backups, replication, failover,   │
│   Recovery)      │ and restoring services.                           │
├──────────────────┼───────────────────────────────────────────────────┤
│  HA              │ Design strategy — prevents DOWNTIME through       │
│  (High           │ redundancy, load balancing, and automatic         │
│   Availability)  │ failover. Goal: zero perceived outage.            │
└──────────────────┴───────────────────────────────────────────────────┘

Relationship:
  BCP ⊃ DR ⊃ HA
  BCP is the umbrella — DR and HA are subsets of BCP.

  BCP = "How does the business survive?"
  DR  = "How do we restore systems?"
  HA  = "How do we prevent systems from going down?"
```

---

## 2. Key BCP Metrics Every Architect Must Know

```
┌────────────────────────────────────────────────────────────────────┐
│                      Critical BCP Metrics                          │
├──────────────┬─────────────────────────────────────────────────────┤
│              │                                                     │
│   RPO        │  Recovery Point Objective                           │
│              │  "How much data can we afford to LOSE?"             │
│              │                                                     │
│              │  RPO = 0    → Zero data loss (synchronous repl.)    │
│              │  RPO = 1 hr → Can lose up to 1 hour of data         │
│              │  RPO = 24 hr→ Daily backups are sufficient          │
│              │                                                     │
├──────────────┼─────────────────────────────────────────────────────┤
│              │                                                     │
│   RTO        │  Recovery Time Objective                            │
│              │  "How fast must we be BACK ONLINE?"                 │
│              │                                                     │
│              │  RTO = 0    → No downtime (hot standby / HA)        │
│              │  RTO = 1 hr → System restored within 1 hour         │
│              │  RTO = 24 hr→ Can tolerate a day of downtime        │
│              │                                                     │
├──────────────┼─────────────────────────────────────────────────────┤
│              │                                                     │
│   MTTR       │  Mean Time To Recover                               │
│              │  Average actual time to restore after failure.       │
│              │  MTTR should be ≤ RTO.                              │
│              │                                                     │
├──────────────┼─────────────────────────────────────────────────────┤
│              │                                                     │
│   MTBF       │  Mean Time Between Failures                         │
│              │  Average time system runs without failure.           │
│              │  Higher = more reliable.                             │
│              │                                                     │
├──────────────┼─────────────────────────────────────────────────────┤
│              │                                                     │
│   SLA        │  Service Level Agreement                            │
│              │  Contractual uptime commitment.                     │
│              │  99.9% = 8.76 hrs downtime/year                     │
│              │  99.99% = 52.6 min downtime/year                    │
│              │  99.999% = 5.26 min downtime/year                   │
│              │                                                     │
└──────────────┴─────────────────────────────────────────────────────┘
```

### RPO vs RTO Visual Timeline

```
          Data Loss Window              Recovery Window
         ◄──────────────────►      ◄──────────────────────►
         │                  │      │                       │
─────────┼──────────────────┼──────┼───────────────────────┼──────►
     Last Backup         DISASTER  Recovery Starts     System Online
                          Event                         Again
         │                  │      │                       │
         └──────RPO─────────┘      └─────────RTO───────────┘
```

### SLA to Downtime Mapping

| SLA (Uptime %) | Annual Downtime | Monthly Downtime | Weekly Downtime |
|---|---|---|---|
| 99% (two nines) | 3.65 days | 7.31 hours | 1.68 hours |
| 99.9% (three nines) | 8.76 hours | 43.8 minutes | 10.1 minutes |
| 99.95% | 4.38 hours | 21.9 minutes | 5.04 minutes |
| 99.99% (four nines) | 52.6 minutes | 4.38 minutes | 1.01 minutes |
| 99.999% (five nines) | 5.26 minutes | 26.3 seconds | 6.05 seconds |

---

## 3. BCP Threat Categories for Software Systems

```
┌────────────────────────────────────────────────────────────────────┐
│                    Threat Classification                           │
├────────────────────┬───────────────────────────────────────────────┤
│ INFRASTRUCTURE     │ • Server / VM failure                         │
│                    │ • Data center / AZ outage                     │
│                    │ • Region-wide outage (cloud provider)         │
│                    │ • Network partition                           │
│                    │ • DNS failure                                 │
│                    │ • Load balancer failure                       │
├────────────────────┼───────────────────────────────────────────────┤
│ DATA               │ • Database corruption                         │
│                    │ • Accidental data deletion                    │
│                    │ • Ransomware / data encryption attack         │
│                    │ • Replication lag / split-brain               │
│                    │ • Backup failure (discovered during restore)  │
├────────────────────┼───────────────────────────────────────────────┤
│ APPLICATION        │ • Bad deployment (broke production)           │
│                    │ • Memory leak / resource exhaustion           │
│                    │ • Cascading failure across services           │
│                    │ • Third-party API/service outage              │
│                    │ • Certificate expiration                      │
├────────────────────┼───────────────────────────────────────────────┤
│ SECURITY           │ • DDoS attack                                 │
│                    │ • Data breach / exfiltration                  │
│                    │ • Credential compromise                       │
│                    │ • Supply chain attack (compromised package)   │
├────────────────────┼───────────────────────────────────────────────┤
│ HUMAN / PROCESS    │ • Key person unavailable (bus factor)         │
│                    │ • Configuration error by operator             │
│                    │ • Vendor/cloud account suspension             │
│                    │ • Regulatory compliance shutdown              │
├────────────────────┼───────────────────────────────────────────────┤
│ NATURAL DISASTER   │ • Earthquake, flood, fire at data center     │
│                    │ • Power grid failure                          │
│                    │ • Pandemic (remote work disruption)           │
└────────────────────┴───────────────────────────────────────────────┘
```

---

## 4. BCP Architecture Strategies

### Strategy 1: Multi-Availability Zone (Multi-AZ) Deployment

```
Goal: Survive a single data center failure
RPO: 0 (synchronous replication)
RTO: Seconds to minutes (automatic failover)

Region: US-East
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  AZ-1 (Data Center A)         AZ-2 (Data Center B)            │
│  ┌──────────────────┐         ┌──────────────────┐            │
│  │  App Server (1)  │         │  App Server (2)  │            │
│  │  App Server (3)  │         │  App Server (4)  │            │
│  └────────┬─────────┘         └────────┬─────────┘            │
│           │                            │                       │
│  ┌────────┴─────────┐         ┌────────┴─────────┐            │
│  │  DB Primary       │ ←sync→ │  DB Standby       │            │
│  │  (Read/Write)     │ repli. │  (Read replica +   │            │
│  │                   │        │   failover target)  │            │
│  └──────────────────┘         └──────────────────┘            │
│                                                                 │
│  ┌──────────────────┐         ┌──────────────────┐            │
│  │  Redis Primary    │ ←sync→ │  Redis Replica    │            │
│  └──────────────────┘         └──────────────────┘            │
│                                                                 │
│              ┌───────────────────────┐                         │
│              │    Load Balancer       │                         │
│              │  (Routes to healthy    │                         │
│              │   instances in any AZ) │                         │
│              └───────────────────────┘                         │
└─────────────────────────────────────────────────────────────────┘
```

### Strategy 2: Multi-Region Deployment (Disaster Recovery)

```
Goal: Survive an entire region going down
RPO: Seconds to minutes (async replication)
RTO: Minutes to hours (DNS failover + data sync)

                    ┌──────────────────┐
                    │   Global Load     │
                    │   Balancer / DNS  │
                    │   (Route 53 /     │
                    │   Traffic Manager) │
                    └────────┬─────────┘
                             │
             ┌───────────────┼───────────────┐
             │                               │
             ▼                               ▼
  ┌─────────────────────┐       ┌─────────────────────┐
  │  PRIMARY REGION      │       │  SECONDARY REGION    │
  │  (US-East)           │       │  (US-West)           │
  │                      │       │                      │
  │  App: Active         │       │  App: Standby/Active │
  │  DB:  Read/Write     │──async──▶ DB: Read Replica   │
  │  Cache: Active       │ repl. │  Cache: Warm/Cold    │
  │  Queue: Active       │       │  Queue: Standby      │
  │                      │       │                      │
  │  Handles 100% traffic│       │  Handles 0% (or read)│
  └─────────────────────┘       └─────────────────────┘

Failover Modes:
  Active-Passive: Secondary idle until primary fails → promote
  Active-Active:  Both regions serve traffic → if one fails, other absorbs
```

### Strategy 3: Active-Active vs Active-Passive

```
┌──────────────────────────────────────────────────────────────────────┐
│                Active-Passive vs Active-Active                       │
├──────────────────┬──────────────────────┬────────────────────────────┤
│ Aspect           │ Active-Passive       │ Active-Active              │
├──────────────────┼──────────────────────┼────────────────────────────┤
│ Traffic handling │ Primary: 100%        │ Both: ~50% each            │
│                  │ Secondary: 0%        │                            │
├──────────────────┼──────────────────────┼────────────────────────────┤
│ Failover time    │ Minutes (promote +   │ Seconds (redirect traffic) │
│                  │ DNS propagation)     │                            │
├──────────────────┼──────────────────────┼────────────────────────────┤
│ Data consistency │ Easier (one writer)  │ Harder (conflict           │
│                  │                      │ resolution needed)         │
├──────────────────┼──────────────────────┼────────────────────────────┤
│ Cost             │ Lower (standby is    │ Higher (both fully         │
│                  │ minimal resources)   │ provisioned)               │
├──────────────────┼──────────────────────┼────────────────────────────┤
│ Complexity       │ Simpler              │ Complex (data sync,        │
│                  │                      │ conflict resolution)       │
├──────────────────┼──────────────────────┼────────────────────────────┤
│ Data loss risk   │ Async replication →  │ Lower (both have recent    │
│                  │ some data loss       │ data)                      │
├──────────────────┼──────────────────────┼────────────────────────────┤
│ Best for         │ Cost-sensitive apps, │ Mission-critical (banking, │
│                  │ RTO of minutes OK    │ healthcare, e-commerce)    │
└──────────────────┴──────────────────────┴────────────────────────────┘
```

---

## 5. Database Continuity Strategies

### Backup Strategy: The 3-2-1 Rule

```
┌──────────────────────────────────────────────────────────────────┐
│                    3-2-1 Backup Rule                              │
│                                                                  │
│    3  copies of data                                             │
│       (1 primary + 2 backups)                                    │
│                                                                  │
│    2  different storage media / types                             │
│       (e.g., SSD + Object Storage)                               │
│                                                                  │
│    1  copy offsite / different region                             │
│       (survives regional disaster)                               │
└──────────────────────────────────────────────────────────────────┘
```

### Database Replication Patterns

```
┌──────────────────────────────────────────────────────────────────────┐
│  Pattern               │  RPO      │  Use Case                      │
├────────────────────────┼───────────┼────────────────────────────────┤
│ Synchronous Repl.      │  0        │  Zero data loss (banking)      │
│ (Commit waits for      │           │  Higher latency on writes      │
│  replica ACK)          │           │                                │
├────────────────────────┼───────────┼────────────────────────────────┤
│ Asynchronous Repl.     │  Seconds  │  Most applications             │
│ (Commit returns        │  to mins  │  Good performance, slight      │
│  before replica ACK)   │           │  data loss risk                │
├────────────────────────┼───────────┼────────────────────────────────┤
│ Semi-Synchronous       │  0-1 txn  │  Balanced (at least 1 replica  │
│ (At least 1 replica    │           │  confirms before commit)       │
│  must ACK)             │           │                                │
├────────────────────────┼───────────┼────────────────────────────────┤
│ Log Shipping           │  Minutes  │  Budget DR, SQL Server         │
│ (Periodic backup       │  to hours │  Older systems                 │
│  restore to standby)   │           │                                │
├────────────────────────┼───────────┼────────────────────────────────┤
│ Point-in-Time Recovery │  Seconds  │  Accidental deletion recovery  │
│ (Transaction log       │           │  "Undo" to any point in time   │
│  continuous backup)    │           │                                │
└────────────────────────┴───────────┴────────────────────────────────┘
```

### SQL Server Always On Availability Groups

```
                    ┌───────────────────┐
                    │   App / EF Core   │
                    │                   │
                    │ Connection String: │
                    │ Listener endpoint  │
                    └────────┬──────────┘
                             │
                    ┌────────┴──────────┐
                    │   AG Listener      │
                    │  (Virtual IP/DNS)  │
                    │  Routes to Primary │
                    └────────┬──────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
            ▼                ▼                ▼
   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
   │  Primary      │ │  Sync        │ │  Async       │
   │  Replica      │ │  Secondary   │ │  Secondary   │
   │  (Read/Write) │ │  (Same AZ)   │ │  (Cross-     │
   │               │ │  Auto-       │ │   Region)    │
   │               │ │  failover    │ │  Manual      │
   │               │ │  RPO = 0     │ │  failover    │
   │               │ │              │ │  RPO > 0     │
   └──────────────┘ └──────────────┘ └──────────────┘
```

```csharp
// EF Core connection string for Always On AG
// Automatically routes to the current primary
{
  "ConnectionStrings": {
    "Default": "Server=ag-listener.example.com;Database=ECommerceDB;MultiSubnetFailover=True;ApplicationIntent=ReadWrite;Encrypt=True;TrustServerCertificate=False"
  }
}

// Read-only routing for reports (routes to secondary)
{
  "ConnectionStrings": {
    "ReadOnly": "Server=ag-listener.example.com;Database=ECommerceDB;MultiSubnetFailover=True;ApplicationIntent=ReadOnly;Encrypt=True;TrustServerCertificate=False"
  }
}

// Using read replicas in EF Core
public class AppDbContext : DbContext
{
    // Primary context for writes
}

public class ReadOnlyDbContext : DbContext
{
    // Points to read replica — for queries, reports, analytics
    // Reduces load on primary
}

// Register both
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(config.GetConnectionString("Default")));

services.AddDbContext<ReadOnlyDbContext>(options =>
    options.UseSqlServer(config.GetConnectionString("ReadOnly")));
```

---

## 6. Application-Level BCP Patterns

### Pattern 1: Circuit Breaker (Prevent Cascading Failures)

```
When a downstream service is failing, stop sending requests —
don't let one failure cascade into total system failure.

States:
┌──────────┐    Failure threshold    ┌──────────┐
│  CLOSED   │ ──────────────────────▶ │   OPEN   │
│ (Normal)  │    reached (e.g., 5    │ (Failing) │
│           │    failures in 10 sec)  │           │
│ Requests  │                        │ Requests  │
│ pass      │                        │ fail-fast │
│ through   │ ◀────────────────────── │ (no call) │
└──────────┘    Success in           └─────┬─────┘
                half-open                  │
                     ▲                     │ Timer expires
                     │                     ▼
                ┌────┴──────┐
                │ HALF-OPEN  │
                │            │
                │ Allow 1    │
                │ test       │
                │ request    │
                └────────────┘
```

```csharp
// Using Polly for Circuit Breaker in ASP.NET Core
builder.Services.AddHttpClient("PaymentService", client =>
{
    client.BaseAddress = new Uri("https://payment-api.internal");
})
.AddTransientHttpErrorPolicy(policy =>
    policy.CircuitBreakerAsync(
        handledEventsAllowedBeforeBreaking: 5,   // Open after 5 failures
        durationOfBreak: TimeSpan.FromSeconds(30) // Stay open for 30 sec
    ))
.AddTransientHttpErrorPolicy(policy =>
    policy.WaitAndRetryAsync(new[]
    {
        TimeSpan.FromMilliseconds(200),
        TimeSpan.FromMilliseconds(500),
        TimeSpan.FromSeconds(1)
    }));
```

### Pattern 2: Bulkhead Isolation

```
Isolate failures to prevent one component from consuming all resources.

Without Bulkhead:
┌──────────────────────────────────┐
│         Shared Thread Pool       │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐   │
│  │ Req│ │ Req│ │ Req│ │ Req│   │   ← Payment Service is slow
│  │Pay │ │Pay │ │Pay │ │Pay │   │   ← All threads consumed by Payment
│  └────┘ └────┘ └────┘ └────┘   │
│  Order and User requests?        │   ← BLOCKED! No threads available.
│  Sorry, system overloaded.       │
└──────────────────────────────────┘

With Bulkhead:
┌──────────────────────────────────────────────────────┐
│  Payment Pool (5)    │  Order Pool (10)  │ User (10) │
│  ┌────┐ ┌────┐      │  ┌────┐ ┌────┐   │ ┌────┐   │
│  │Slow│ │Slow│      │  │ OK │ │ OK │   │ │ OK │   │
│  └────┘ └────┘      │  └────┘ └────┘   │ └────┘   │
│  Payment is slow     │  Orders still     │ Users    │
│  → only Payment      │  work fine!       │ work!    │
│    affected          │                   │          │
└──────────────────────────────────────────────────────┘
```

```csharp
// Polly Bulkhead — limit concurrent calls to a downstream service
builder.Services.AddHttpClient("InventoryService")
    .AddPolicyHandler(Policy.BulkheadAsync<HttpResponseMessage>(
        maxParallelization: 10,     // Max 10 concurrent calls
        maxQueuingActions: 20,      // Max 20 waiting in queue
        onBulkheadRejectedAsync: context =>
        {
            Log.Warning("Inventory service bulkhead full — request rejected");
            return Task.CompletedTask;
        }
    ));
```

### Pattern 3: Retry with Exponential Backoff + Jitter

```csharp
// Polly Retry — transient failure recovery
builder.Services.AddHttpClient("CatalogService")
    .AddTransientHttpErrorPolicy(policy =>
        policy.WaitAndRetryAsync(
            retryCount: 3,
            sleepDurationProvider: attempt =>
                TimeSpan.FromSeconds(Math.Pow(2, attempt))       // 2s, 4s, 8s
                + TimeSpan.FromMilliseconds(Random.Shared.Next(0, 1000)), // Jitter
            onRetry: (outcome, delay, attempt, _) =>
            {
                Log.Warning(
                    "Catalog retry #{Attempt} after {Delay}ms. Status: {Status}",
                    attempt, delay.TotalMilliseconds, outcome.Result?.StatusCode);
            }
        ));

// Why jitter?
// Without jitter: 100 clients retry at exactly 2s, 4s, 8s → thundering herd
// With jitter:    Clients retry at 2.3s, 1.8s, 2.7s → spread out, reduces load
```

### Pattern 4: Timeout Policy

```csharp
// Don't wait forever for a slow service
builder.Services.AddHttpClient("ShippingService")
    .AddPolicyHandler(Policy.TimeoutAsync<HttpResponseMessage>(
        TimeSpan.FromSeconds(5))) // Fail after 5 seconds
    .AddTransientHttpErrorPolicy(policy =>
        policy.CircuitBreakerAsync(3, TimeSpan.FromSeconds(30)));

// Combined: If Shipping takes > 5s → timeout → counts as failure
//           After 3 timeouts → circuit opens → fail fast for 30s
```

### Pattern 5: Fallback (Graceful Degradation)

```csharp
// When a service is down, provide degraded but functional experience
public class ProductService
{
    private readonly HttpClient _catalogClient;
    private readonly IDistributedCache _cache;

    public async Task<List<Product>> GetFeaturedProductsAsync()
    {
        try
        {
            // Try live catalog service
            var response = await _catalogClient.GetAsync("/api/products/featured");
            response.EnsureSuccessStatusCode();

            var products = await response.Content.ReadFromJsonAsync<List<Product>>();

            // Cache for fallback use
            await _cache.SetStringAsync("featured_products",
                JsonSerializer.Serialize(products),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
                });

            return products!;
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Catalog service unavailable — using cached data");

            // Fallback: return stale cached data (better than nothing)
            var cached = await _cache.GetStringAsync("featured_products");
            if (cached is not null)
                return JsonSerializer.Deserialize<List<Product>>(cached)!;

            // Ultimate fallback: return empty list with message
            return new List<Product>();
        }
    }
}
```

### Pattern 6: Queue-Based Load Leveling

```
Problem: Sudden traffic spike overwhelms the service.
Solution: Put a message queue between producer and consumer.

Without Queue:
  Client ──▶ API ──▶ Database     ← 10,000 req/sec, DB collapses

With Queue:
  Client ──▶ API ──▶ Queue ──▶ Worker ──▶ Database
                      (buffer)   (processes at
                                  safe rate)

  Queue absorbs bursts. Workers consume at sustainable pace.
  Even if workers are temporarily down, messages wait in queue.
```

```csharp
// Order placement → queue → async processing
public class OrderController : ControllerBase
{
    private readonly IMessagePublisher _publisher;

    [HttpPost]
    public async Task<IActionResult> PlaceOrder(OrderRequest request)
    {
        // Validate and create order record (status = Pending)
        var orderId = Guid.NewGuid();
        await _orderRepo.CreatePendingOrderAsync(orderId, request);

        // Publish to queue for async processing (payment, inventory, shipping)
        await _publisher.PublishAsync(new OrderPlacedEvent
        {
            OrderId = orderId,
            UserId = request.UserId,
            Items = request.Items,
            Timestamp = DateTime.UtcNow
        });

        // Return immediately — don't make user wait for payment + inventory
        return Accepted(new { OrderId = orderId, Status = "Processing" });
    }
}
```

---

## 7. Health Checks & Monitoring for BCP

### ASP.NET Core Health Checks

```csharp
// Program.cs — Register health checks for all dependencies
builder.Services.AddHealthChecks()
    // Database connectivity
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("Default")!,
        name: "sqlserver",
        failureStatus: HealthStatus.Unhealthy,
        tags: new[] { "db", "critical" })

    // Redis cache
    .AddRedis(
        redisConnectionString: builder.Configuration["Redis:Connection"]!,
        name: "redis",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "cache" })

    // RabbitMQ message broker
    .AddRabbitMQ(
        rabbitConnectionString: builder.Configuration["RabbitMQ:Connection"]!,
        name: "rabbitmq",
        failureStatus: HealthStatus.Unhealthy,
        tags: new[] { "messaging", "critical" })

    // Downstream service
    .AddUrlGroup(
        new Uri("https://payment-api.internal/health"),
        name: "payment-service",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "downstream" })

    // Disk space
    .AddDiskStorageHealthCheck(
        setup: opt => opt.AddDrive("C:\\", 1024),  // Min 1GB free
        name: "disk-space",
        failureStatus: HealthStatus.Degraded);

// Map endpoints
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    // Liveness: Is the app running? (No dependency checks)
    Predicate = _ => false,
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    // Readiness: Are all dependencies available?
    Predicate = check => check.Tags.Contains("critical"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/full", new HealthCheckOptions
{
    // Full: All health checks
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

### Health Check Response

```json
{
  "status": "Degraded",
  "totalDuration": "00:00:00.234",
  "entries": {
    "sqlserver": { "status": "Healthy", "duration": "00:00:00.045" },
    "redis": { "status": "Degraded", "duration": "00:00:00.120",
               "description": "High latency detected" },
    "rabbitmq": { "status": "Healthy", "duration": "00:00:00.032" },
    "payment-service": { "status": "Healthy", "duration": "00:00:00.067" }
  }
}
```

### Kubernetes Probes Integration

```yaml
# Kubernetes deployment — uses health check endpoints
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: order-service
          image: order-service:latest
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 15
            failureThreshold: 3       # Restart after 3 failures
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 2       # Remove from LB after 2 failures
          startupProbe:
            httpGet:
              path: /health/live
              port: 8080
            failureThreshold: 30
            periodSeconds: 5          # Allow 150s for slow startup
```

---

## 8. Deployment Strategies for Zero-Downtime

```
┌────────────────────────────────────────────────────────────────────────┐
│                    Deployment Strategies                                │
├─────────────────┬──────────────────────────────────────────────────────┤
│                 │                                                      │
│  Rolling        │  Replace instances one by one.                       │
│  Update         │  Old → New → Old → New → Done                       │
│                 │  ✅ Zero downtime  ❌ Mixed versions during deploy   │
│                 │                                                      │
├─────────────────┼──────────────────────────────────────────────────────┤
│                 │                                                      │
│  Blue-Green     │  Two identical environments.                         │
│                 │  Blue (current) serves traffic.                      │
│                 │  Green (new version) deployed and tested.            │
│                 │  Switch load balancer: Blue → Green.                 │
│                 │  ✅ Instant rollback  ❌ Double infrastructure cost  │
│                 │                                                      │
├─────────────────┼──────────────────────────────────────────────────────┤
│                 │                                                      │
│  Canary         │  Route 5% of traffic to new version.                │
│                 │  Monitor errors, latency, business metrics.          │
│                 │  If good → gradually increase to 100%.              │
│                 │  If bad → roll back the 5%.                         │
│                 │  ✅ Low risk  ❌ More complex routing                │
│                 │                                                      │
├─────────────────┼──────────────────────────────────────────────────────┤
│                 │                                                      │
│  Feature        │  Deploy code with feature behind a flag.             │
│  Flags          │  Toggle on/off without redeployment.                 │
│                 │  ✅ Decouple deploy from release  ❌ Flag debt       │
│                 │                                                      │
└─────────────────┴──────────────────────────────────────────────────────┘
```

### Blue-Green Deployment Flow

```
Step 1: Blue is live
  LB ──▶ Blue (v1.0) ✅    Green (idle)

Step 2: Deploy v1.1 to Green
  LB ──▶ Blue (v1.0) ✅    Green (v1.1) 🔄 deploying...

Step 3: Test Green (smoke tests, health checks)
  LB ──▶ Blue (v1.0) ✅    Green (v1.1) ✅ tested

Step 4: Switch traffic to Green
  LB ──▶ Green (v1.1) ✅   Blue (v1.0) standby

Step 5: If problems → instant rollback
  LB ──▶ Blue (v1.0) ✅    Green (v1.1) ❌ rolled back

Step 6: If stable → decommission Blue or keep as next target
```

### Canary Deployment with YARP (Reverse Proxy)

```csharp
// YARP (Yet Another Reverse Proxy) for canary routing in ASP.NET Core
// appsettings.json
{
  "ReverseProxy": {
    "Routes": {
      "order-route": {
        "ClusterId": "order-cluster",
        "Match": { "Path": "/api/orders/{**catch-all}" }
      }
    },
    "Clusters": {
      "order-cluster": {
        "LoadBalancingPolicy": "Random",
        "Destinations": {
          "stable": {
            "Address": "https://order-v1.internal/",
            "Metadata": { "Weight": "95" }   // 95% traffic
          },
          "canary": {
            "Address": "https://order-v2.internal/",
            "Metadata": { "Weight": "5" }    // 5% canary
          }
        }
      }
    }
  }
}
```

---

## 9. Data Backup & Recovery Patterns

### Automated Backup Strategy

```csharp
// Backup orchestration service
public class BackupOrchestrator : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var now = DateTime.UtcNow;

            // Full backup: Weekly (Sunday 2 AM UTC)
            if (now.DayOfWeek == DayOfWeek.Sunday && now.Hour == 2)
            {
                await PerformFullBackupAsync("full", stoppingToken);
            }
            // Differential backup: Daily (2 AM UTC, except Sunday)
            else if (now.Hour == 2)
            {
                await PerformDifferentialBackupAsync("diff", stoppingToken);
            }
            // Transaction log backup: Every 15 minutes
            if (now.Minute % 15 == 0)
            {
                await PerformLogBackupAsync("log", stoppingToken);
            }

            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
        }
    }

    private async Task PerformFullBackupAsync(string type, CancellationToken ct)
    {
        var backupName = $"ECommerceDB_{type}_{DateTime.UtcNow:yyyyMMdd_HHmmss}.bak";

        // Backup to local storage
        await _dbAdmin.ExecuteBackupAsync(backupName, BackupType.Full);

        // Copy to remote storage (3-2-1 rule)
        await _blobStorage.UploadAsync($"backups/{backupName}", localPath);

        // Copy to different region
        await _crossRegionStorage.ReplicateAsync($"backups/{backupName}");

        // Verify backup integrity
        var isValid = await _dbAdmin.VerifyBackupAsync(backupName);
        if (!isValid)
        {
            await _alertService.SendCriticalAlertAsync(
                $"Backup verification FAILED: {backupName}");
        }

        // Clean old backups (keep 4 weekly, 30 daily, 96 log)
        await CleanOldBackupsAsync();

        _logger.LogInformation("Backup completed: {Name}, Valid: {IsValid}",
            backupName, isValid);
    }
}
```

### Backup Schedule Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Backup Schedule                                    │
├──────────────┬────────────┬─────────────┬────────────────────────────┤
│  Type         │  Frequency │  Retention  │  RPO if Restored           │
├──────────────┼────────────┼─────────────┼────────────────────────────┤
│  Full         │  Weekly    │  4 weeks    │  Up to 7 days data loss    │
│  Differential │  Daily     │  30 days    │  Up to 24 hours data loss  │
│  Log          │  Every 15m │  4 days     │  Up to 15 minutes          │
│  Snapshot     │  Hourly    │  48 hours   │  Up to 1 hour              │
├──────────────┴────────────┴─────────────┴────────────────────────────┤
│  Restore Strategy:                                                    │
│  Last Full + Last Differential + All Logs since differential         │
│  = Point-in-time recovery to within 15 minutes                       │
└──────────────────────────────────────────────────────────────────────┘
```

### Test Your Backups! (Restore Drill)

```
❌ Common Mistake:
  "We have backups" → Never tested restore → Backup is corrupt → Data lost

✅ Best Practice — Monthly Restore Drill:
  1. Pick a random backup from last month
  2. Restore to a test environment
  3. Run data integrity checks
  4. Verify row counts match expected
  5. Test application connectivity to restored DB
  6. Measure actual restore time (verify RTO is achievable)
  7. Document results
```

---

## 10. Chaos Engineering — Test Your BCP

### What Is Chaos Engineering?

```
"Deliberately inject failures into production (or staging) to verify
 that the system handles them correctly BEFORE a real disaster happens."

                          — Netflix (inventors of Chaos Monkey)

Principle: "If you haven't tested your failover, you don't have failover."
```

### Types of Chaos Experiments

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Chaos Experiments                                 │
├────────────────────┬────────────────────────────────────────────────┤
│ Kill a service     │ Stop a random service instance                 │
│ instance           │ → Does LB route to healthy instances?          │
├────────────────────┼────────────────────────────────────────────────┤
│ Network latency    │ Add 2 seconds latency to service calls         │
│                    │ → Do timeouts and retries work?                │
├────────────────────┼────────────────────────────────────────────────┤
│ Network partition  │ Block traffic between two services             │
│                    │ → Does circuit breaker open? Fallback?         │
├────────────────────┼────────────────────────────────────────────────┤
│ Disk full          │ Fill up disk on a node                         │
│                    │ → Do alerts fire? Does app handle gracefully?  │
├────────────────────┼────────────────────────────────────────────────┤
│ Database failover  │ Force primary → secondary failover             │
│                    │ → How long? Any data loss? App reconnects?     │
├────────────────────┼────────────────────────────────────────────────┤
│ DNS failure        │ Corrupt DNS resolution for a service           │
│                    │ → Does cache help? Retry? Alert?               │
├────────────────────┼────────────────────────────────────────────────┤
│ Clock skew         │ Set time 5 minutes ahead on one node           │
│                    │ → JWT validation still works? Cert valid?      │
├────────────────────┼────────────────────────────────────────────────┤
│ Memory pressure    │ Consume 90% of memory on a node                │
│                    │ → OOM kill? Graceful degradation?              │
├────────────────────┼────────────────────────────────────────────────┤
│ AZ outage          │ Simulate entire availability zone going down   │
│                    │ → Traffic routes to surviving AZ?              │
└────────────────────┴────────────────────────────────────────────────┘
```

### Chaos Engineering Maturity Levels

```
Level 0: "We don't test failures"
  → Discover failures in production at 3 AM

Level 1: Manual testing in staging
  → Kill processes, test recovery manually

Level 2: Automated chaos in staging
  → Scheduled chaos experiments, automated verification

Level 3: Automated chaos in production
  → Chaos Monkey (Netflix), Gremlin, LitmusChaos
  → Auto-abort if blast radius exceeds threshold

Level 4: Continuous chaos + GameDays
  → Regular full-scale disaster simulations
  → Cross-team incident response exercises
```

---

## 11. Incident Response Plan (When BCP Activates)

### Incident Severity Levels

```
┌───────┬───────────────────────────────┬────────────────┬───────────────┐
│ Level │ Description                   │ Response Time  │ Example       │
├───────┼───────────────────────────────┼────────────────┼───────────────┤
│ SEV-1 │ Complete service outage        │ 15 minutes     │ All users     │
│       │ Revenue impact, data loss risk │                │ cannot access │
├───────┼───────────────────────────────┼────────────────┼───────────────┤
│ SEV-2 │ Major feature degraded         │ 30 minutes     │ Payments      │
│       │ Significant user impact        │                │ failing       │
├───────┼───────────────────────────────┼────────────────┼───────────────┤
│ SEV-3 │ Minor feature impacted         │ 4 hours        │ Search slow   │
│       │ Workaround available           │                │ but working   │
├───────┼───────────────────────────────┼────────────────┼───────────────┤
│ SEV-4 │ Cosmetic / Low impact          │ Next business  │ Typo on       │
│       │ No revenue impact              │ day            │ error page    │
└───────┴───────────────────────────────┴────────────────┴───────────────┘
```

### Incident Response Flow

```
    ┌─────────────┐
    │  DETECT     │  Monitoring alert fires / User reports issue
    │  (0-5 min)  │  PagerDuty / OpsGenie → On-call engineer
    └──────┬──────┘
           │
    ┌──────┴──────┐
    │  TRIAGE     │  Determine severity (SEV 1-4)
    │  (5-15 min) │  Assemble incident response team
    │             │  Create incident channel (Slack/Teams)
    └──────┬──────┘
           │
    ┌──────┴──────┐
    │  CONTAIN    │  Stop the bleeding:
    │  (15-60 min)│  → Rollback bad deployment
    │             │  → Failover to secondary
    │             │  → Enable circuit breakers
    │             │  → Scale up / rate limit
    └──────┬──────┘
           │
    ┌──────┴──────┐
    │  RESOLVE    │  Fix the root cause:
    │  (varies)   │  → Deploy hotfix
    │             │  → Restore from backup
    │             │  → Patch vulnerability
    └──────┬──────┘
           │
    ┌──────┴──────┐
    │  RECOVER    │  Verify full recovery:
    │             │  → All health checks green
    │             │  → Data consistency verified
    │             │  → Performance back to baseline
    └──────┬──────┘
           │
    ┌──────┴──────┐
    │  POST-      │  Blameless post-mortem:
    │  MORTEM     │  → Timeline of events
    │  (within    │  → Root cause analysis
    │   48 hours) │  → Action items to prevent recurrence
    │             │  → Update runbooks
    └─────────────┘
```

### Runbook Template (for On-Call Engineers)

```markdown
## Runbook: Database Failover

**Trigger:** Primary DB health check fails for > 2 minutes

**Auto-remediation:** 
- AG listener auto-fails over to sync secondary (< 30 sec)
- Application reconnects automatically (MultiSubnetFailover=True)

**If auto-failover fails → Manual steps:**
1. Verify primary is truly down: `SELECT @@SERVERNAME` on both nodes
2. Check AG dashboard: `SELECT * FROM sys.dm_hadr_availability_group_states`
3. Force manual failover:
   ```sql
   ALTER AVAILABILITY GROUP [ECommerceAG] FORCE_FAILOVER_ALLOW_DATA_LOSS;
   ```
4. Update DNS if listener IP changed
5. Verify application connectivity
6. Notify stakeholders via incident channel

**Escalation:** If not resolved in 30 min → page Database Team Lead
**Communication:** Update status page every 15 minutes
```

---

## 12. BCP for Microservices — Service Mesh Resilience

```
Service Mesh (Istio / Linkerd) — BCP at the infrastructure level:

┌─────────────────────────────────────────────────────────────────┐
│                    Service Mesh BCP Features                     │
├─────────────────────┬───────────────────────────────────────────┤
│ Automatic retries   │ Mesh retries failed requests (configurable│
│                     │ per route, with backoff)                   │
├─────────────────────┼───────────────────────────────────────────┤
│ Circuit breaking    │ Mesh opens circuit when error rate spikes │
│                     │ (no code changes needed)                   │
├─────────────────────┼───────────────────────────────────────────┤
│ Timeout enforcement │ Global timeout policies at mesh level      │
├─────────────────────┼───────────────────────────────────────────┤
│ Traffic shifting    │ Canary: 5% to new version, 95% to stable  │
│                     │ Instant rollback by shifting back to 100% │
├─────────────────────┼───────────────────────────────────────────┤
│ Outlier detection   │ Eject unhealthy pods from load balancer   │
│                     │ (e.g., 5 consecutive 5xx → eject for 30s) │
├─────────────────────┼───────────────────────────────────────────┤
│ mTLS everywhere     │ Encrypted service-to-service communication│
│                     │ Zero trust networking by default           │
├─────────────────────┼───────────────────────────────────────────┤
│ Observability       │ Distributed tracing, metrics, access logs │
│                     │ across all services — no code changes      │
└─────────────────────┴───────────────────────────────────────────┘
```

### Istio Resilience Configuration

```yaml
# Istio DestinationRule — circuit breaker + outlier detection
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service.default.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: DEFAULT
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
    outlierDetection:
      consecutive5xxErrors: 3        # Eject after 3 consecutive 5xx
      interval: 10s                  # Check every 10 seconds
      baseEjectionTime: 30s          # Eject for 30 seconds
      maxEjectionPercent: 50         # Never eject more than 50% of pods
    circuitBreaker:
      maxConnections: 100
      maxPendingRequests: 50
      maxRetries: 3

---
# Istio VirtualService — retry + timeout
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
    - payment-service
  http:
    - route:
        - destination:
            host: payment-service
      timeout: 5s
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: 5xx,reset,connect-failure,retriable-4xx
```

---

## 13. BCP Cost-Benefit Analysis

### Cost of Downtime vs Cost of BCP

```
Estimated Downtime Cost by Business Type:

Business Type          Revenue/Hour     1-Hour Outage     24-Hour Outage
──────────────         ────────────     ─────────────     ──────────────
Small e-commerce       $500             $500              $12,000
Mid-size SaaS          $10,000          $10,000           $240,000
Large e-commerce       $100,000         $100,000          $2,400,000
Financial services     $1,000,000       $1,000,000        $24,000,000
Cloud provider (AWS)   $10,000,000+     $10,000,000+      $240,000,000+

Hidden Costs (add 3-5x multiplier):
  + Customer churn (lost trust)
  + SLA violation penalties
  + Employee overtime for recovery
  + Regulatory fines
  + PR/communication costs
  + Legal fees (if data loss)
```

### BCP Investment Tiers

```
┌───────────────────────────────────────────────────────────────────────┐
│  Tier  │ Investment  │ Achievable SLA │ Capabilities                  │
├────────┼─────────────┼────────────────┼───────────────────────────────┤
│        │             │                │                               │
│  Basic │ Low         │ 99% - 99.9%   │ • Daily backups               │
│        │ ($)         │ (8.7 hrs/yr)   │ • Single region               │
│        │             │                │ • Manual failover             │
│        │             │                │ • Basic monitoring            │
│        │             │                │ • Documented runbooks         │
│        │             │                │                               │
├────────┼─────────────┼────────────────┼───────────────────────────────┤
│        │             │                │                               │
│  Mid   │ Medium      │ 99.9% - 99.99%│ • Multi-AZ deployment         │
│        │ ($$)        │ (52 min/yr)    │ • Auto-failover DB            │
│        │             │                │ • Health checks + alerting    │
│        │             │                │ • Circuit breakers            │
│        │             │                │ • Blue-green deployments      │
│        │             │                │ • 15-min log backups          │
│        │             │                │                               │
├────────┼─────────────┼────────────────┼───────────────────────────────┤
│        │             │                │                               │
│  High  │ High        │ 99.99% -      │ • Multi-region active-active  │
│        │ ($$$)       │  99.999%       │ • Global load balancing       │
│        │             │ (5 min/yr)     │ • Chaos engineering           │
│        │             │                │ • Service mesh resilience     │
│        │             │                │ • Real-time replication       │
│        │             │                │ • Auto-scaling                │
│        │             │                │ • 24/7 on-call rotation       │
│        │             │                │ • Regular disaster drills     │
│        │             │                │                               │
└────────┴─────────────┴────────────────┴───────────────────────────────┘
```

---

## 14. BCP Architect Checklist

```
AVAILABILITY & REDUNDANCY
  □ Multi-AZ deployment (survive single data center failure)
  □ Multi-region strategy defined (for critical services)
  □ Load balancer with health-based routing
  □ Database replication configured (sync for HA, async for DR)
  □ No single points of failure (SPOF audit completed)

BACKUP & RECOVERY
  □ 3-2-1 backup rule followed
  □ Automated backups (full + differential + log)
  □ Backup verification (automated integrity checks)
  □ Monthly restore drill documented and passed
  □ Point-in-time recovery tested
  □ RPO and RTO defined per service and tested

RESILIENCE PATTERNS
  □ Circuit breakers on all downstream calls
  □ Retry with exponential backoff + jitter
  □ Timeout policies on all external calls
  □ Bulkhead isolation for critical services
  □ Fallback / graceful degradation for non-critical features
  □ Queue-based load leveling for write-heavy operations

DEPLOYMENT
  □ Zero-downtime deployment strategy (rolling / blue-green / canary)
  □ Rollback capability tested (< 5 minutes)
  □ Feature flags for risky features
  □ Database migrations are backward-compatible

MONITORING & ALERTING
  □ Health check endpoints (liveness + readiness)
  □ Alerts for SLA breach risk (before it happens)
  □ Distributed tracing across services
  □ Dashboard for key business + system metrics
  □ PagerDuty / OpsGenie on-call rotation configured

INCIDENT RESPONSE
  □ Severity levels defined (SEV 1-4)
  □ Runbooks for top 10 failure scenarios
  □ Incident communication plan (status page, stakeholder updates)
  □ Blameless post-mortem template and process
  □ Escalation matrix documented

TESTING
  □ Chaos experiments in staging (monthly)
  □ GameDay exercises (quarterly)
  □ Failover drills tested (DB, region, service)
  □ Load testing to identify breaking point
  □ DR simulation end-to-end (annually)

PEOPLE & PROCESS
  □ On-call rotation with documented handoff
  □ Bus factor ≥ 2 for every critical service
  □ Knowledge base with architecture decisions (ADRs)
  □ BCP plan reviewed and updated quarterly
  □ Training for new team members on incident response
```

---

## 15. BCP Quick Reference — Recovery Strategy Decision Matrix

| Scenario | Strategy | RPO | RTO | Cost |
|---|---|---|---|---|
| Server crash | Multi-AZ + auto-failover | 0 | Seconds | $$ |
| Data center outage | Multi-AZ + DB replication | 0 | Minutes | $$ |
| Region outage | Multi-region active-passive | Minutes | 30-60 min | $$$ |
| Region outage (critical) | Multi-region active-active | 0 | Seconds | $$$$ |
| Bad deployment | Blue-green rollback | 0 | Minutes | $$ |
| Accidental data deletion | Point-in-time restore | 15 min | 30-60 min | $ |
| Ransomware attack | Air-gapped backup restore | Hours | Hours-Days | $$ |
| Cascading service failure | Circuit breaker + bulkhead | 0 | Seconds | $ |
| Traffic spike | Auto-scaling + queue leveling | 0 | Minutes | $$ |
| DNS failure | Multi-CDN + DNS failover | 0 | Minutes | $$ |
| Key person leaves | Documentation + bus factor ≥ 2 | N/A | N/A | $ |

### The BCP Golden Rule

```
"A disaster recovery plan that hasn't been tested
 is just a document that gives you false confidence."

→ Test your backups.
→ Test your failovers.
→ Test your runbooks.
→ Test them regularly.
→ Test them under pressure (GameDays).
```

---
---

# Security Architecture in Software Design

---

## 1. Security Architecture Overview

**Security Architecture** defines how security controls are woven into every layer of the system — from network to application to data — ensuring confidentiality, integrity, and availability (CIA triad).

```
┌──────────────────────────────────────────────────────────────────────┐
│                    CIA Triad + AAA Model                             │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  CIA Triad:                          AAA Model:                      │
│  ┌─────────────────┐                 ┌─────────────────┐            │
│  │ Confidentiality │                 │ Authentication  │            │
│  │ (only authorized│                 │ (Who are you?)  │            │
│  │  can read)      │                 ├─────────────────┤            │
│  ├─────────────────┤                 │ Authorization   │            │
│  │ Integrity       │                 │ (What can you   │            │
│  │ (data not       │                 │  do?)           │            │
│  │  tampered)      │                 ├─────────────────┤            │
│  ├─────────────────┤                 │ Accounting      │            │
│  │ Availability    │                 │ (What did you   │            │
│  │ (system is      │                 │  do? Audit)     │            │
│  │  accessible)    │                 └─────────────────┘            │
│  └─────────────────┘                                                │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Defense in Depth — Layered Security

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Defense in Depth                                  │
│                                                                     │
│  Layer 1: NETWORK          Firewall, WAF, DDoS protection, VPN     │
│     │                                                               │
│  Layer 2: PERIMETER        API Gateway, rate limiting, IP filtering │
│     │                                                               │
│  Layer 3: IDENTITY         OAuth2, JWT, MFA, SSO                   │
│     │                                                               │
│  Layer 4: APPLICATION      Input validation, CORS, CSP headers     │
│     │                                                               │
│  Layer 5: DATA             Encryption at rest, column-level,       │
│     │                      tokenization                             │
│  Layer 6: MONITORING       Audit logs, SIEM, anomaly detection     │
│                                                                     │
│  Each layer assumes the layer above it has been breached.           │
│  No single layer is sufficient alone.                               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Zero Trust Architecture

```
Traditional Model (Castle & Moat):
  "Trust everything inside the network perimeter"
  ❌ Once an attacker is inside, they move freely

Zero Trust Model:
  "Never trust, always verify"
  ✅ Every request is authenticated and authorized, regardless of origin

                Traditional                    Zero Trust
                ──────────                     ──────────
  Internal      Trusted by default             Must authenticate
  traffic       (no auth needed)               (same as external)

  VPN users     Full network access            Access only specific
                                               resources

  Microservice  Service A calls B freely       Service A must present
  calls                                        identity + be authorized

  Database      App server connects freely     App uses managed identity
  access                                       + least privilege role
```

### Zero Trust Principles

```
┌──────────────────────────────────────────────────────────────────┐
│              Zero Trust — 5 Core Principles                      │
├──────────────────────┬───────────────────────────────────────────┤
│ 1. Verify explicitly │ Always authenticate and authorize based   │
│                      │ on all available data points              │
├──────────────────────┼───────────────────────────────────────────┤
│ 2. Least privilege   │ Limit access to just-in-time and         │
│    access            │ just-enough-access (JIT/JEA)             │
├──────────────────────┼───────────────────────────────────────────┤
│ 3. Assume breach     │ Minimize blast radius, segment access,   │
│                      │ verify end-to-end encryption              │
├──────────────────────┼───────────────────────────────────────────┤
│ 4. Micro-            │ Divide the network into small zones,     │
│    segmentation      │ enforce policies per zone                 │
├──────────────────────┼───────────────────────────────────────────┤
│ 5. Continuous        │ Monitor and validate security posture     │
│    validation        │ continuously, not just at login           │
└──────────────────────┴───────────────────────────────────────────┘
```

### Zero Trust in Microservices

```csharp
// mTLS: Every service has a certificate — mutual authentication
// Service A calling Service B:

// 1. API Gateway validates external JWT
// 2. Gateway issues internal service token (or forwards validated claims)
// 3. Service B verifies the token + checks authorization

// Service-to-service auth using HttpClient + certificate
builder.Services.AddHttpClient("PaymentService")
    .ConfigurePrimaryHttpMessageHandler(() =>
    {
        var handler = new HttpClientHandler();
        handler.ClientCertificates.Add(
            new X509Certificate2("service-a-cert.pfx", "password"));
        handler.ServerCertificateCustomValidationCallback =
            (message, cert, chain, errors) =>
            {
                // Validate Payment Service's certificate
                return cert?.Thumbprint == expectedThumbprint;
            };
        return handler;
    });

// Or using Azure Managed Identity (no secrets to manage)
builder.Services.AddHttpClient("PaymentService")
    .AddHttpMessageHandler(() =>
    {
        var credential = new DefaultAzureCredential();
        return new BearerTokenHandler(credential, 
            new[] { "api://payment-service/.default" });
    });
```

---

## 3. Threat Modeling — STRIDE

### What Is Threat Modeling?

```
Threat modeling is a structured process to:
  1. Identify WHAT you're protecting (assets)
  2. Identify WHO might attack (threat actors)
  3. Identify HOW they might attack (threats)
  4. Decide WHAT to do about it (mitigations)

Do it BEFORE writing code, not after a breach.
```

### STRIDE Framework

```
┌──────────────────────────────────────────────────────────────────────┐
│  Threat             │ Violates      │ Example                       │
├─────────────────────┼───────────────┼───────────────────────────────┤
│ S — Spoofing        │ Authentication│ Attacker pretends to be       │
│                     │               │ another user (stolen JWT)     │
├─────────────────────┼───────────────┼───────────────────────────────┤
│ T — Tampering       │ Integrity     │ Modify request payload, SQL   │
│                     │               │ injection, man-in-the-middle  │
├─────────────────────┼───────────────┼───────────────────────────────┤
│ R — Repudiation     │ Non-repud.    │ User denies performing an     │
│                     │               │ action (no audit trail)       │
├─────────────────────┼───────────────┼───────────────────────────────┤
│ I — Information     │ Confidential. │ Sensitive data exposed in     │
│     Disclosure      │               │ logs, error messages, APIs    │
├─────────────────────┼───────────────┼───────────────────────────────┤
│ D — Denial of       │ Availability  │ DDoS attack, resource         │
│     Service         │               │ exhaustion, infinite loops    │
├─────────────────────┼───────────────┼───────────────────────────────┤
│ E — Elevation of    │ Authorization │ Regular user gains admin      │
│     Privilege       │               │ access (broken access control)│
└─────────────────────┴───────────────┴───────────────────────────────┘
```

### STRIDE Applied to E-Commerce API

```
Asset: Order API (/api/orders)

┌─────────────────┬───────────────────────────────┬──────────────────────────────┐
│ Threat           │ Attack Scenario               │ Mitigation                   │
├─────────────────┼───────────────────────────────┼──────────────────────────────┤
│ Spoofing         │ Stolen JWT used to place      │ Short JWT expiry (15 min),   │
│                  │ orders as another user        │ refresh token rotation,      │
│                  │                               │ device fingerprint           │
├─────────────────┼───────────────────────────────┼──────────────────────────────┤
│ Tampering        │ Modify order total in         │ Server-side price calc,      │
│                  │ request payload               │ input validation,            │
│                  │                               │ HMAC on critical fields      │
├─────────────────┼───────────────────────────────┼──────────────────────────────┤
│ Repudiation      │ Customer claims they never    │ Audit log with timestamp,    │
│                  │ placed the order              │ IP, device info              │
├─────────────────┼───────────────────────────────┼──────────────────────────────┤
│ Info Disclosure  │ Order details of other users  │ Authorize per resource       │
│                  │ visible via IDOR              │ (check userId == orderId     │
│                  │                               │  owner)                      │
├─────────────────┼───────────────────────────────┼──────────────────────────────┤
│ DoS              │ Attacker places 10,000 orders │ Rate limiting (100/min),     │
│                  │ per minute                    │ CAPTCHA, throttling          │
├─────────────────┼───────────────────────────────┼──────────────────────────────┤
│ Elevation        │ Regular user changes role     │ Never trust client-side      │
│                  │ claim in JWT                  │ roles, validate server-side  │
│                  │                               │ on every request             │
└─────────────────┴───────────────────────────────┴──────────────────────────────┘
```

### DREAD Risk Scoring

```
After identifying threats with STRIDE, prioritize using DREAD:

D — Damage Potential:     How much damage if exploited? (1-10)
R — Reproducibility:      How easy to reproduce? (1-10)
E — Exploitability:       How easy to exploit? (1-10)
A — Affected Users:       How many users impacted? (1-10)
D — Discoverability:      How easy to discover? (1-10)

Risk Score = (D + R + E + A + D) / 5

Example: SQL Injection on Login
  Damage:          10 (full database access)
  Reproducibility: 8  (easy with tools like sqlmap)
  Exploitability:  9  (well-known attack)
  Affected Users:  10 (all users)
  Discoverability: 7  (automated scanners find it)
  Score = (10+8+9+10+7)/5 = 8.8 → CRITICAL — fix immediately

Score Ranges:
  1-3:  Low risk    → Accept or fix in next sprint
  4-6:  Medium risk → Fix in current release
  7-10: High risk   → Fix immediately, block release
```

---

## 4. Authentication Architecture — OAuth 2.0 & OpenID Connect

### OAuth 2.0 Flows

```
┌──────────────────────────────────────────────────────────────────────┐
│                    OAuth 2.0 Flow Selection                          │
├──────────────────┬───────────────────┬───────────────────────────────┤
│ Client Type      │ Flow              │ Use Case                      │
├──────────────────┼───────────────────┼───────────────────────────────┤
│ Web App          │ Authorization     │ Server-side app that can      │
│ (server-side)    │ Code + PKCE       │ securely store secrets        │
├──────────────────┼───────────────────┼───────────────────────────────┤
│ SPA (React,      │ Authorization     │ Public client, no secret      │
│  Angular)        │ Code + PKCE       │ (PKCE prevents interception)  │
├──────────────────┼───────────────────┼───────────────────────────────┤
│ Mobile App       │ Authorization     │ Public client, uses custom    │
│                  │ Code + PKCE       │ URI scheme for redirect       │
├──────────────────┼───────────────────┼───────────────────────────────┤
│ Machine-to-      │ Client            │ Backend service calling       │
│ Machine (M2M)    │ Credentials       │ another service (no user)     │
├──────────────────┼───────────────────┼───────────────────────────────┤
│ Legacy (avoid)   │ Resource Owner    │ ❌ DEPRECATED — sends         │
│                  │ Password          │ username/password to client   │
├──────────────────┼───────────────────┼───────────────────────────────┤
│ Never use        │ Implicit          │ ❌ DEPRECATED — token in      │
│                  │                   │ URL fragment (insecure)       │
└──────────────────┴───────────────────┴───────────────────────────────┘
```

### Authorization Code Flow with PKCE

```
User        Browser/App     Auth Server      API (Resource Server)
 │              │                │                    │
 │──Login──────▶│                │                    │
 │              │──Auth Request──▶                    │
 │              │  + code_verifier                    │
 │              │  + code_challenge                   │
 │              │  (PKCE)        │                    │
 │◀─Login Page──│◀───────────────│                    │
 │──Credentials─▶                │                    │
 │              │◀─Auth Code─────│                    │
 │              │                │                    │
 │              │──Exchange Code─▶                    │
 │              │  + code_verifier                    │
 │              │◀─Access Token──│                    │
 │              │  + Refresh Token                    │
 │              │  + ID Token    │                    │
 │              │                                     │
 │              │──API Call + Bearer Token────────────▶
 │              │◀─Protected Resource─────────────────│
```

### JWT Token Architecture

```csharp
// JWT structure: header.payload.signature

// Access Token (short-lived: 15 min)
{
  "header": { "alg": "RS256", "typ": "JWT", "kid": "key-id-1" },
  "payload": {
    "sub": "user-123",           // Subject (user ID)
    "iss": "https://auth.example.com",  // Issuer
    "aud": "https://api.example.com",   // Audience
    "exp": 1718000000,           // Expires (15 min from now)
    "iat": 1717999100,           // Issued at
    "scope": "orders.read orders.write",
    "roles": ["customer"],
    "tenant_id": "tenant-456"
  },
  "signature": "RS256(header + payload, private_key)"
}

// ASP.NET Core JWT validation
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://auth.example.com";
        options.Audience = "https://api.example.com";
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ClockSkew = TimeSpan.FromSeconds(30),  // Tight clock skew
            RequireExpirationTime = true
        };
    });
```

### Refresh Token Rotation (Prevent Token Theft)

```csharp
public class TokenService
{
    public async Task<TokenResponse> RefreshAsync(string refreshToken)
    {
        var storedToken = await _tokenRepo.FindByTokenAsync(refreshToken);

        // 1. Check if token exists and is valid
        if (storedToken is null || storedToken.IsRevoked || storedToken.ExpiresAt < DateTime.UtcNow)
            throw new SecurityException("Invalid refresh token");

        // 2. Check for token reuse (theft detection)
        if (storedToken.IsUsed)
        {
            // Token was already used! Possible theft.
            // Revoke ALL tokens for this user (nuclear option)
            await _tokenRepo.RevokeAllUserTokensAsync(storedToken.UserId);
            _logger.LogWarning("Refresh token reuse detected for user {UserId}", storedToken.UserId);
            throw new SecurityException("Token reuse detected — all sessions revoked");
        }

        // 3. Mark old token as used
        storedToken.IsUsed = true;
        await _tokenRepo.UpdateAsync(storedToken);

        // 4. Issue new access + refresh token pair
        var newAccessToken = GenerateAccessToken(storedToken.UserId);
        var newRefreshToken = GenerateRefreshToken();

        await _tokenRepo.CreateAsync(new RefreshToken
        {
            Token = newRefreshToken,
            UserId = storedToken.UserId,
            ExpiresAt = DateTime.UtcNow.AddDays(7),
            CreatedAt = DateTime.UtcNow
        });

        return new TokenResponse(newAccessToken, newRefreshToken);
    }
}
```

---

## 5. OWASP Top 10 — Architecture Mitigations

```
┌──────────────────────────────────────────────────────────────────────┐
│                    OWASP Top 10 (2021)                               │
├────┬─────────────────────────┬───────────────────────────────────────┤
│ #  │ Vulnerability            │ Architectural Mitigation              │
├────┼─────────────────────────┼───────────────────────────────────────┤
│ 1  │ Broken Access Control   │ RBAC/ABAC, resource-level authz,     │
│    │                         │ deny by default                       │
├────┼─────────────────────────┼───────────────────────────────────────┤
│ 2  │ Cryptographic Failures  │ TLS 1.2+, AES-256, key rotation,    │
│    │                         │ never roll your own crypto            │
├────┼─────────────────────────┼───────────────────────────────────────┤
│ 3  │ Injection               │ Parameterized queries (EF Core),     │
│    │                         │ input validation, ORM                 │
├────┼─────────────────────────┼───────────────────────────────────────┤
│ 4  │ Insecure Design         │ Threat modeling (STRIDE), secure by  │
│    │                         │ design principles, abuse cases        │
├────┼─────────────────────────┼───────────────────────────────────────┤
│ 5  │ Security Misconfig      │ Hardened defaults, no debug in prod, │
│    │                         │ infrastructure as code                │
├────┼─────────────────────────┼───────────────────────────────────────┤
│ 6  │ Vulnerable Components   │ Dependency scanning (Dependabot),    │
│    │                         │ SCA tools, regular updates            │
├────┼─────────────────────────┼───────────────────────────────────────┤
│ 7  │ Auth Failures           │ MFA, rate limit login, secure        │
│    │                         │ password hashing (BCrypt/Argon2)      │
├────┼─────────────────────────┼───────────────────────────────────────┤
│ 8  │ Software & Data         │ Signed builds, CI/CD integrity,     │
│    │ Integrity Failures      │ verify dependencies                   │
├────┼─────────────────────────┼───────────────────────────────────────┤
│ 9  │ Logging & Monitoring    │ Structured logging, SIEM, alerting, │
│    │ Failures                │ audit trails (without logging PII)    │
├────┼─────────────────────────┼───────────────────────────────────────┤
│ 10 │ SSRF (Server-Side       │ Allowlist outbound URLs, block       │
│    │ Request Forgery)        │ internal IPs, network segmentation   │
└────┴─────────────────────────┴───────────────────────────────────────┘
```

### ASP.NET Core Security Headers

```csharp
// Security headers middleware
app.Use(async (context, next) =>
{
    var headers = context.Response.Headers;

    // Prevent clickjacking
    headers["X-Frame-Options"] = "DENY";

    // Prevent MIME sniffing
    headers["X-Content-Type-Options"] = "nosniff";

    // XSS protection
    headers["X-XSS-Protection"] = "1; mode=block";

    // Referrer policy
    headers["Referrer-Policy"] = "strict-origin-when-cross-origin";

    // Content Security Policy
    headers["Content-Security-Policy"] =
        "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; " +
        "img-src 'self' data:; font-src 'self'; connect-src 'self' https://api.example.com";

    // HSTS (HTTP Strict Transport Security)
    headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains; preload";

    // Permissions Policy
    headers["Permissions-Policy"] = "camera=(), microphone=(), geolocation=()";

    await next();
});
```

### Rate Limiting in ASP.NET Core

```csharp
// .NET 7+ built-in rate limiting
builder.Services.AddRateLimiter(options =>
{
    // Global rate limit: 100 requests per minute
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(
        context => RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.Connection.RemoteIpAddress?.ToString() ?? "unknown",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1),
                QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
                QueueLimit = 10
            }));

    // Specific policy for login (strict: 5 attempts per minute)
    options.AddFixedWindowLimiter("login", opt =>
    {
        opt.PermitLimit = 5;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.QueueLimit = 0;  // No queuing — reject immediately
    });

    options.OnRejected = async (context, ct) =>
    {
        context.HttpContext.Response.StatusCode = 429;
        await context.HttpContext.Response.WriteAsJsonAsync(
            new { Error = "Too many requests. Try again later." }, ct);
    };
});

app.UseRateLimiter();

// Apply to endpoints
app.MapPost("/api/auth/login", LoginHandler)
    .RequireRateLimiting("login");
```

---

## 6. Security Architecture Checklist

```
AUTHENTICATION
  □ OAuth 2.0 / OpenID Connect with PKCE
  □ JWT access tokens (short-lived: 15 min)
  □ Refresh token rotation with reuse detection
  □ MFA for admin and sensitive operations
  □ Password hashing: BCrypt/Argon2 (never MD5/SHA1)
  □ Account lockout after N failed attempts

AUTHORIZATION
  □ RBAC or ABAC enforced server-side
  □ Resource-level authorization (not just role checks)
  □ Deny by default — explicitly grant permissions
  □ API Gateway validates JWT before forwarding

TRANSPORT
  □ TLS 1.2+ everywhere (no plain HTTP)
  □ mTLS for service-to-service communication
  □ Certificate pinning for mobile apps
  □ HSTS header with preload

APPLICATION
  □ OWASP Top 10 mitigations implemented
  □ Input validation on all endpoints
  □ Security headers (CSP, X-Frame-Options, etc.)
  □ CORS properly configured (no wildcard *)
  □ Rate limiting on all public endpoints
  □ CSRF protection for state-changing operations

INFRASTRUCTURE
  □ Secrets in Key Vault (never in code/config)
  □ Managed identities for cloud resources
  □ Network segmentation / micro-segmentation
  □ WAF in front of public endpoints
  □ Container image scanning in CI/CD

MONITORING
  □ Failed auth attempt alerting
  □ Privilege escalation detection
  □ Anomalous API usage patterns
  □ Dependency vulnerability scanning (automated)
```

---
---

# Observability in Software Architecture

---

## 1. What Is Observability?

**Observability** is the ability to understand the **internal state** of a system by examining its **external outputs** — logs, metrics, and traces. It answers: "Why is my system behaving this way?"

```
Monitoring vs. Observability:

Monitoring:    "Is the system UP or DOWN?" (known unknowns)
               → Dashboard turns red when CPU > 90%

Observability: "WHY is the system slow for users in Mumbai?" (unknown unknowns)
               → Trace a request across 5 services to find the bottleneck
```

---

## 2. The Three Pillars of Observability

```
┌──────────────────────────────────────────────────────────────────────┐
│                  Three Pillars of Observability                       │
├──────────────────┬──────────────────┬────────────────────────────────┤
│                  │                  │                                │
│  📋 LOGS         │  📊 METRICS      │  🔗 TRACES                    │
│                  │                  │                                │
│  What happened   │  How much / how  │  Where time was spent          │
│  (events)        │  fast (numbers)  │  (request flow)                │
│                  │                  │                                │
│  "User 123       │  "Avg latency:   │  "Request took 850ms:         │
│   login failed   │   245ms"         │   API Gateway: 5ms            │
│   at 14:32:05    │  "Error rate:    │   Auth Service: 45ms          │
│   from IP x.x"   │   2.3%"         │   Order Service: 200ms        │
│                  │  "Active users:  │   Payment Service: 600ms ←    │
│                  │   12,450"        │   bottleneck!"                │
│                  │                  │                                │
│  Tools:          │  Tools:          │  Tools:                        │
│  ELK, Seq,       │  Prometheus,     │  Jaeger, Zipkin,              │
│  Serilog,        │  Grafana,        │  OpenTelemetry,               │
│  App Insights    │  Datadog         │  App Insights                  │
└──────────────────┴──────────────────┴────────────────────────────────┘
```

---

## 3. Structured Logging

```csharp
// ❌ BAD — Unstructured logs (hard to search/analyze)
logger.LogInformation($"Order {orderId} placed by user {userId} for ${amount}");
// Output: "Order 123 placed by user 456 for $99.99"
// → Can't filter by orderId, can't aggregate by userId

// ✅ GOOD — Structured logging with Serilog
logger.LogInformation(
    "Order {OrderId} placed by user {UserId} for {Amount}",
    orderId, userId, amount);
// Output JSON: { "OrderId": 123, "UserId": 456, "Amount": 99.99, ... }
// → Searchable: WHERE OrderId = 123
// → Aggregatable: COUNT BY UserId

// Serilog setup with enrichers
Log.Logger = new LoggerConfiguration()
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithEnvironmentName()
    .Enrich.WithProperty("ServiceName", "OrderService")
    .WriteTo.Console(new JsonFormatter())
    .WriteTo.Seq("http://seq-server:5341")       // Centralized log server
    .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(
        new Uri("http://elasticsearch:9200")))
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .CreateLogger();
```

### Correlation ID — Track Requests Across Services

```csharp
// Middleware to propagate correlation ID across all services
public class CorrelationIdMiddleware
{
    private const string CorrelationHeader = "X-Correlation-Id";
    private readonly RequestDelegate _next;

    public async Task InvokeAsync(HttpContext context)
    {
        // Get or generate correlation ID
        var correlationId = context.Request.Headers[CorrelationHeader].FirstOrDefault()
            ?? Guid.NewGuid().ToString();

        // Add to response headers
        context.Response.Headers[CorrelationHeader] = correlationId;

        // Push to log context (all logs in this request will include it)
        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await _next(context);
        }
    }
}

// Now every log entry includes CorrelationId:
// { "CorrelationId": "abc-123", "Message": "Order placed", "OrderId": 456 }
// Search across ALL services: WHERE CorrelationId = "abc-123"
```

### Log Levels — When to Use Each

```
┌────────────┬──────────────────────────────────────────────────────┐
│ Level      │ When to Use                                          │
├────────────┼──────────────────────────────────────────────────────┤
│ Trace      │ Very detailed diagnostic info (loop iterations,      │
│            │ variable values). NEVER in production.                │
├────────────┼──────────────────────────────────────────────────────┤
│ Debug      │ Detailed info useful during development.             │
│            │ Disabled in production (enable temporarily to debug). │
├────────────┼──────────────────────────────────────────────────────┤
│ Information│ Normal application flow events.                       │
│            │ "Order placed", "User logged in", "Payment processed" │
├────────────┼──────────────────────────────────────────────────────┤
│ Warning    │ Something unexpected but recoverable.                 │
│            │ "Retry #2 to Payment Service", "Cache miss",          │
│            │ "Deprecated API called"                                │
├────────────┼──────────────────────────────────────────────────────┤
│ Error      │ Operation failed but service continues.               │
│            │ "Payment failed for order 123",                       │
│            │ "Database timeout on query X"                          │
├────────────┼──────────────────────────────────────────────────────┤
│ Critical   │ Unrecoverable failure, app may crash.                 │
│            │ "Database connection pool exhausted",                  │
│            │ "Out of memory", "Configuration missing"               │
└────────────┴──────────────────────────────────────────────────────┘
```

---

## 4. Metrics — SLIs, SLOs, SLAs

### Key Metrics Categories

```
┌──────────────────────────────────────────────────────────────────────┐
│  Category        │  Metric                   │  Example              │
├──────────────────┼───────────────────────────┼───────────────────────┤
│  RED Metrics     │  Rate                     │  500 req/sec          │
│  (Request-       │  Errors                   │  2% error rate        │
│   focused)       │  Duration                 │  p99 = 450ms          │
├──────────────────┼───────────────────────────┼───────────────────────┤
│  USE Metrics     │  Utilization              │  CPU at 65%           │
│  (Resource-      │  Saturation               │  Queue depth: 1200    │
│   focused)       │  Errors                   │  Disk I/O errors: 3   │
├──────────────────┼───────────────────────────┼───────────────────────┤
│  Business        │  Orders/minute            │  42 orders/min        │
│  Metrics         │  Revenue/hour             │  $12,500/hr           │
│                  │  Cart abandonment rate    │  68%                  │
│                  │  Signup conversion         │  3.2%                 │
└──────────────────┴───────────────────────────┴───────────────────────┘
```

### SLI → SLO → SLA Chain

```
SLI (Service Level Indicator):
  The MEASUREMENT — what you actually measure
  Example: "Percentage of requests completing in < 500ms"

SLO (Service Level Objective):
  The TARGET — internal goal for the SLI
  Example: "99.9% of requests must complete in < 500ms"

SLA (Service Level Agreement):
  The CONTRACT — external promise to customers (with penalties)
  Example: "99.9% uptime or we give 10% credit"

  SLO should be STRICTER than SLA:
    SLO = 99.95% (internal target)
    SLA = 99.9%  (customer promise)
    Buffer = 0.05% for safety

Error Budget:
  If SLO = 99.9%, then Error Budget = 0.1%
  = 43.8 minutes/month of allowed downtime
  → Team can use error budget for risky deployments
  → If budget is spent, freeze deployments until next month
```

### Prometheus Metrics in ASP.NET Core

```csharp
// Custom Prometheus metrics
public class OrderMetrics
{
    private static readonly Counter OrdersPlaced = Metrics.CreateCounter(
        "orders_placed_total",
        "Total number of orders placed",
        new CounterConfiguration { LabelNames = new[] { "status", "payment_method" } });

    private static readonly Histogram OrderLatency = Metrics.CreateHistogram(
        "order_processing_duration_seconds",
        "Time to process an order",
        new HistogramConfiguration
        {
            Buckets = Histogram.ExponentialBuckets(0.01, 2, 10) // 10ms to 10s
        });

    private static readonly Gauge ActiveOrders = Metrics.CreateGauge(
        "orders_active_count",
        "Number of orders currently being processed");

    public async Task<Order> PlaceOrderAsync(OrderRequest request)
    {
        ActiveOrders.Inc();
        using var timer = OrderLatency.NewTimer();

        try
        {
            var order = await ProcessOrderAsync(request);
            OrdersPlaced.WithLabels("success", request.PaymentMethod).Inc();
            return order;
        }
        catch (Exception)
        {
            OrdersPlaced.WithLabels("failed", request.PaymentMethod).Inc();
            throw;
        }
        finally
        {
            ActiveOrders.Dec();
        }
    }
}
```

---

## 5. Distributed Tracing — OpenTelemetry

```
Trace: End-to-end journey of a single request

Trace ID: abc-123
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Span: API Gateway (5ms)                                        │
│  ├── Span: Auth Service — Validate JWT (45ms)                   │
│  ├── Span: Order Service — Create Order (200ms)                 │
│  │   ├── Span: DB Query — Insert Order (25ms)                   │
│  │   └── Span: RabbitMQ — Publish OrderPlaced (10ms)            │
│  └── Span: Payment Service — Charge Card (600ms)  ← BOTTLENECK │
│      ├── Span: External API — Stripe (550ms)       ← ROOT CAUSE│
│      └── Span: DB Query — Update Payment (15ms)                 │
│                                                                  │
│  Total: 850ms                                                    │
└──────────────────────────────────────────────────────────────────┘
```

```csharp
// OpenTelemetry setup in ASP.NET Core
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .SetResourceBuilder(ResourceBuilder.CreateDefault()
            .AddService("OrderService", serviceVersion: "1.0.0"))
        .AddAspNetCoreInstrumentation()         // HTTP incoming requests
        .AddHttpClientInstrumentation()          // HTTP outgoing calls
        .AddSqlClientInstrumentation(options =>
        {
            options.SetDbStatementForText = true; // Capture SQL queries
            options.RecordException = true;
        })
        .AddSource("OrderService.Activities")    // Custom spans
        .AddJaegerExporter(options =>
        {
            options.AgentHost = "jaeger";
            options.AgentPort = 6831;
        })
        .AddOtlpExporter(options =>              // OpenTelemetry Collector
        {
            options.Endpoint = new Uri("http://otel-collector:4317");
        }))
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddPrometheusExporter());

// Custom spans for business operations
public class OrderService
{
    private static readonly ActivitySource Activity = new("OrderService.Activities");

    public async Task<Order> PlaceOrderAsync(OrderRequest request)
    {
        using var span = Activity.StartActivity("PlaceOrder");
        span?.SetTag("order.user_id", request.UserId.ToString());
        span?.SetTag("order.item_count", request.Items.Count);

        // Each downstream call creates a child span automatically
        var inventory = await _inventoryClient.ReserveAsync(request.Items);
        var payment = await _paymentClient.ChargeAsync(request.Amount);

        span?.SetTag("order.status", "completed");
        return order;
    }
}
```

---

## 6. Observability Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                Observability Architecture                           │
│                                                                    │
│   Services (emit telemetry)                                        │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐                       │
│   │ Order    │  │ Payment  │  │ User     │                       │
│   │ Service  │  │ Service  │  │ Service  │                       │
│   └────┬─────┘  └────┬─────┘  └────┬─────┘                       │
│        │             │             │                               │
│        └─────────────┼─────────────┘                               │
│                      │                                             │
│                      ▼                                             │
│        ┌─────────────────────────┐                                │
│        │  OpenTelemetry Collector │ ← Receives, processes, exports │
│        │  (Central aggregation)   │                                │
│        └────────┬────────────────┘                                │
│                 │                                                   │
│    ┌────────────┼────────────┬────────────────┐                   │
│    │            │            │                │                    │
│    ▼            ▼            ▼                ▼                    │
│ ┌────────┐ ┌────────┐ ┌──────────┐ ┌──────────────┐             │
│ │ Jaeger │ │Promethe│ │ Elastic  │ │ Grafana      │             │
│ │(Traces)│ │  us    │ │  search  │ │ (Dashboards) │             │
│ │        │ │(Metrics)│ │  (Logs) │ │              │             │
│ └────────┘ └────────┘ └──────────┘ └──────┬───────┘             │
│                                           │                       │
│                                    ┌──────┴───────┐               │
│                                    │  Alert       │               │
│                                    │  Manager     │               │
│                                    │  → PagerDuty │               │
│                                    │  → Slack     │               │
│                                    │  → Email     │               │
│                                    └──────────────┘               │
└────────────────────────────────────────────────────────────────────┘
```

### Alerting Strategy

```
Alert on SYMPTOMS, not CAUSES:

❌ BAD Alerts (cause-based):
  "CPU > 80%"         → So what? Maybe it's normal during batch processing.
  "Memory > 90%"      → Is it affecting users? Maybe GC will clean it up.
  "Disk > 85%"        → Is it growing? Or stable?

✅ GOOD Alerts (symptom-based):
  "p99 latency > 2s for 5 min"              → Users are experiencing slowness
  "Error rate > 5% for 2 min"               → Something is broken
  "Orders/min dropped 50% vs. last week"    → Revenue impact
  "Zero successful payments in 5 min"       → Payment system is down

Alert Severity:
  🔴 Critical: Page on-call NOW (e.g., zero payments)
  🟡 Warning:  Slack notification (e.g., latency rising)
  🟢 Info:     Dashboard only (e.g., deployment completed)
```

---
---

# API Design & Governance in Software Architecture

---

## 1. API Design Principles

```
┌──────────────────────────────────────────────────────────────────────┐
│                    API Design Principles                             │
├──────────────────────┬───────────────────────────────────────────────┤
│ 1. Contract-First    │ Design the API contract (OpenAPI spec) before │
│                      │ writing code. Agreement between teams.        │
├──────────────────────┼───────────────────────────────────────────────┤
│ 2. Consistency       │ Same naming conventions, error formats, and   │
│                      │ pagination across ALL APIs.                   │
├──────────────────────┼───────────────────────────────────────────────┤
│ 3. Backward Compat.  │ Never break existing clients when evolving   │
│                      │ the API. Additive changes only.               │
├──────────────────────┼───────────────────────────────────────────────┤
│ 4. Idempotency       │ Same request sent twice → same result.        │
│                      │ Critical for retries and reliability.         │
├──────────────────────┼───────────────────────────────────────────────┤
│ 5. Discoverability   │ HATEOAS, OpenAPI docs, self-documenting      │
│                      │ responses with links and schemas.             │
├──────────────────────┼───────────────────────────────────────────────┤
│ 6. Least Surprise    │ API behaves as developers expect.             │
│                      │ GET = read, POST = create, DELETE = remove.   │
└──────────────────────┴───────────────────────────────────────────────┘
```

---

## 2. RESTful API Design Standards

### Resource Naming Conventions

```
✅ GOOD:
  GET    /api/v1/orders                    → List orders
  GET    /api/v1/orders/123                → Get order by ID
  POST   /api/v1/orders                    → Create order
  PUT    /api/v1/orders/123                → Replace order
  PATCH  /api/v1/orders/123                → Partially update order
  DELETE /api/v1/orders/123                → Delete order
  GET    /api/v1/orders/123/items          → List items in order
  GET    /api/v1/users/456/orders          → Orders for a user

❌ BAD:
  GET    /api/getOrders                    → Verb in URL
  POST   /api/createOrder                  → Verb in URL
  GET    /api/v1/order                     → Singular (use plural)
  GET    /api/v1/Orders                    → PascalCase (use kebab/lower)
  POST   /api/v1/orders/create             → Redundant verb
  GET    /api/v1/orders?action=delete      → Using GET for mutation
```

### Standard HTTP Status Codes

```
┌──────┬───────────────────────────────────────────────────────────────┐
│ Code │ Meaning & When to Use                                        │
├──────┼───────────────────────────────────────────────────────────────┤
│ 200  │ OK — Successful GET, PUT, PATCH                              │
│ 201  │ Created — Successful POST (return Location header)           │
│ 202  │ Accepted — Async operation started (not completed yet)       │
│ 204  │ No Content — Successful DELETE (nothing to return)           │
├──────┼───────────────────────────────────────────────────────────────┤
│ 400  │ Bad Request — Validation failed, malformed request           │
│ 401  │ Unauthorized — Not authenticated (no/invalid token)          │
│ 403  │ Forbidden — Authenticated but not authorized                 │
│ 404  │ Not Found — Resource doesn't exist                           │
│ 409  │ Conflict — Duplicate resource, concurrency conflict          │
│ 422  │ Unprocessable Entity — Valid JSON but business rule violated │
│ 429  │ Too Many Requests — Rate limit exceeded                      │
├──────┼───────────────────────────────────────────────────────────────┤
│ 500  │ Internal Server Error — Unhandled exception (bug)            │
│ 502  │ Bad Gateway — Upstream service returned invalid response     │
│ 503  │ Service Unavailable — Server overloaded or in maintenance    │
│ 504  │ Gateway Timeout — Upstream service didn't respond in time    │
└──────┴───────────────────────────────────────────────────────────────┘
```

### Standard Error Response Format (RFC 7807)

```csharp
// Consistent error format across all APIs
public class ProblemDetails
{
    public string Type { get; set; }     // URI to error documentation
    public string Title { get; set; }    // Short description
    public int Status { get; set; }      // HTTP status code
    public string Detail { get; set; }   // Human-readable explanation
    public string Instance { get; set; } // Request path
    public Dictionary<string, string[]> Errors { get; set; } // Validation errors
}

// Example response:
{
  "type": "https://api.example.com/errors/validation",
  "title": "Validation Failed",
  "status": 422,
  "detail": "One or more fields have invalid values",
  "instance": "/api/v1/orders",
  "errors": {
    "amount": ["Amount must be greater than 0"],
    "shippingAddress.zip": ["Invalid ZIP code format"]
  }
}
```

---

## 3. API Versioning Strategies

```
┌────────────────────────────────────────────────────────────────────────┐
│                    API Versioning Strategies                           │
├─────────────────┬──────────────────────────┬──────────────────────────┤
│ Strategy        │ Example                  │ Pros / Cons              │
├─────────────────┼──────────────────────────┼──────────────────────────┤
│ URL Path        │ /api/v1/orders           │ ✅ Obvious, cacheable    │
│ (most common)   │ /api/v2/orders           │ ❌ URL pollution         │
├─────────────────┼──────────────────────────┼──────────────────────────┤
│ Query String    │ /api/orders?v=2          │ ✅ Simple to add         │
│                 │                          │ ❌ Easy to forget, messy │
├─────────────────┼──────────────────────────┼──────────────────────────┤
│ Header          │ X-Api-Version: 2         │ ✅ Clean URLs            │
│                 │ Api-Version: 2024-01-15  │ ❌ Not visible in browser│
├─────────────────┼──────────────────────────┼──────────────────────────┤
│ Media Type      │ Accept: application/     │ ✅ RESTful purist        │
│ (content neg.)  │   vnd.example.v2+json    │ ❌ Complex, rarely used  │
├─────────────────┼──────────────────────────┼──────────────────────────┤
│ Date-based      │ Api-Version: 2024-01-15  │ ✅ Clear timeline        │
│ (Stripe style)  │                          │ ❌ Need version registry │
└─────────────────┴──────────────────────────┴──────────────────────────┘

Recommendation: URL Path (/api/v1/) for public APIs
                Header-based for internal microservice APIs
```

### Backward Compatibility Rules

```
✅ NON-BREAKING (safe to deploy):
  + Add new optional field to response
  + Add new optional query parameter
  + Add new endpoint
  + Add new enum value (if client handles unknown gracefully)

❌ BREAKING (requires new version):
  - Remove or rename a field
  - Change field type (string → int)
  - Change required/optional status
  - Change URL path
  - Change HTTP method
  - Change error response format
  - Remove an endpoint
```

---

## 4. Idempotency — Critical for Reliability

```
Problem: Client sends POST /orders, network drops → client retries
         → Two orders created (duplicate)!

Solution: Idempotency key — same key = same result

Client ──POST /orders──▶  API receives, checks idempotency key
         Idempotency-Key:  │
         "abc-123"         ├─ First time → process, store result
                           ├─ Second time → return stored result (no reprocessing)
                           └─ Third time → return stored result
```

```csharp
// Idempotency middleware
public class IdempotencyMiddleware
{
    private readonly IDistributedCache _cache;

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        // Only apply to state-changing methods
        if (context.Request.Method is not ("POST" or "PUT" or "PATCH"))
        {
            await next(context);
            return;
        }

        var idempotencyKey = context.Request.Headers["Idempotency-Key"].FirstOrDefault();
        if (string.IsNullOrEmpty(idempotencyKey))
        {
            await next(context);
            return;
        }

        var cacheKey = $"idempotency:{idempotencyKey}";
        var cachedResponse = await _cache.GetStringAsync(cacheKey);

        if (cachedResponse is not null)
        {
            // Return cached response (no re-processing)
            var cached = JsonSerializer.Deserialize<CachedResponse>(cachedResponse);
            context.Response.StatusCode = cached!.StatusCode;
            context.Response.ContentType = "application/json";
            await context.Response.WriteAsync(cached.Body);
            return;
        }

        // Process request and cache the response
        var originalBody = context.Response.Body;
        using var memStream = new MemoryStream();
        context.Response.Body = memStream;

        await next(context);

        memStream.Position = 0;
        var responseBody = await new StreamReader(memStream).ReadToEndAsync();

        // Cache for 24 hours
        await _cache.SetStringAsync(cacheKey,
            JsonSerializer.Serialize(new CachedResponse(context.Response.StatusCode, responseBody)),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(24)
            });

        memStream.Position = 0;
        await memStream.CopyToAsync(originalBody);
    }
}
```

---

## 5. Pagination, Filtering & Sorting

```csharp
// Standard pagination response
{
  "data": [ ... ],
  "pagination": {
    "page": 2,
    "pageSize": 20,
    "totalCount": 487,
    "totalPages": 25,
    "hasNext": true,
    "hasPrevious": true
  },
  "links": {
    "self": "/api/v1/orders?page=2&pageSize=20",
    "first": "/api/v1/orders?page=1&pageSize=20",
    "last": "/api/v1/orders?page=25&pageSize=20",
    "next": "/api/v1/orders?page=3&pageSize=20",
    "previous": "/api/v1/orders?page=1&pageSize=20"
  }
}

// Usage:
// GET /api/v1/orders?page=2&pageSize=20&sort=createdAt:desc&status=completed
```

```csharp
// Cursor-based pagination (better for large datasets / real-time feeds)
// Uses last item's ID as cursor — no offset skipping

// GET /api/v1/orders?cursor=eyJpZCI6MTIzfQ&limit=20

{
  "data": [ ... ],
  "pagination": {
    "nextCursor": "eyJpZCI6MTQzfQ",    // Base64 encoded { "id": 143 }
    "hasMore": true
  }
}

// Cursor vs Offset:
// Offset (page=50): DB must scan and skip 1000 rows → SLOW on large tables
// Cursor (after=id): DB seeks directly to the row → FAST, consistent
```

---

## 6. API Gateway Pattern

```
┌────────────────────────────────────────────────────────────────────┐
│                    API Gateway Responsibilities                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Client Request ──▶ API Gateway                                   │
│                     │                                              │
│                     ├── Authentication (validate JWT)              │
│                     ├── Authorization (check scopes/roles)         │
│                     ├── Rate Limiting (100 req/min per IP)         │
│                     ├── Request Validation (schema check)          │
│                     ├── Request Transformation (add headers)       │
│                     ├── Load Balancing (round-robin to instances)  │
│                     ├── Circuit Breaking (fail fast if service down)│
│                     ├── Caching (cache GET responses)              │
│                     ├── Logging & Tracing (add trace ID)          │
│                     ├── TLS Termination (HTTPS → HTTP internal)   │
│                     └── Response Transformation (filter fields)    │
│                                                                    │
│  Tools: YARP, Ocelot, Kong, AWS API Gateway, Azure APIM           │
└────────────────────────────────────────────────────────────────────┘
```

---
---

# Data Architecture in Software Design

---

## 1. Database Selection — SQL vs NoSQL Decision Matrix

```
┌──────────────────────────────────────────────────────────────────────┐
│                    SQL vs NoSQL Decision Matrix                       │
├──────────────────┬──────────────────────┬────────────────────────────┤
│ Criteria         │ SQL (Relational)     │ NoSQL                      │
├──────────────────┼──────────────────────┼────────────────────────────┤
│ Data model       │ Structured, fixed    │ Flexible, schema-less      │
│                  │ schema, relations    │ (document, key-value, etc) │
├──────────────────┼──────────────────────┼────────────────────────────┤
│ Consistency      │ Strong (ACID)        │ Eventual (BASE) or tunable │
├──────────────────┼──────────────────────┼────────────────────────────┤
│ Scaling          │ Vertical (scale up)  │ Horizontal (scale out)     │
├──────────────────┼──────────────────────┼────────────────────────────┤
│ Joins            │ Native, efficient    │ No joins (denormalize)     │
├──────────────────┼──────────────────────┼────────────────────────────┤
│ Transactions     │ Multi-table ACID     │ Single-document atomic     │
├──────────────────┼──────────────────────┼────────────────────────────┤
│ Query language   │ SQL (standard)       │ Varies per DB              │
├──────────────────┼──────────────────────┼────────────────────────────┤
│ Best for         │ Financial, ERP,      │ Catalog, IoT, social,      │
│                  │ inventory, reports   │ real-time analytics         │
└──────────────────┴──────────────────────┴────────────────────────────┘
```

### NoSQL Types — When to Use Each

```
┌──────────────────────────────────────────────────────────────────────┐
│  Type           │  Example         │  Best For                       │
├─────────────────┼──────────────────┼─────────────────────────────────┤
│ Document        │ MongoDB,         │ Product catalogs, user profiles,│
│                 │ CosmosDB         │ content management, flexible    │
│                 │                  │ schema                          │
├─────────────────┼──────────────────┼─────────────────────────────────┤
│ Key-Value       │ Redis,           │ Caching, session storage,       │
│                 │ DynamoDB         │ shopping carts, leaderboards    │
├─────────────────┼──────────────────┼─────────────────────────────────┤
│ Column-Family   │ Cassandra,       │ Time-series data, IoT sensor   │
│                 │ HBase            │ data, write-heavy workloads     │
├─────────────────┼──────────────────┼─────────────────────────────────┤
│ Graph           │ Neo4j,           │ Social networks, recommendation │
│                 │ CosmosDB Gremlin │ engines, fraud detection        │
├─────────────────┼──────────────────┼─────────────────────────────────┤
│ Search          │ Elasticsearch,   │ Full-text search, log analysis, │
│                 │ OpenSearch       │ autocomplete                    │
├─────────────────┼──────────────────┼─────────────────────────────────┤
│ Time-Series     │ InfluxDB,        │ Monitoring metrics, financial   │
│                 │ TimescaleDB      │ tick data, IoT telemetry        │
└─────────────────┴──────────────────┴─────────────────────────────────┘
```

### Polyglot Persistence — Use the Right DB for Each Job

```
E-Commerce Platform:

  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │ User Service │     │ Product Svc  │     │ Order Service│
  │              │     │              │     │              │
  │  PostgreSQL  │     │  MongoDB     │     │ SQL Server   │
  │  (ACID for   │     │  (Flexible   │     │ (ACID for    │
  │   accounts)  │     │   catalog    │     │  orders,     │
  │              │     │   schema)    │     │  inventory)  │
  └──────────────┘     └──────────────┘     └──────────────┘

  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │ Cart Service │     │ Search       │     │ Analytics    │
  │              │     │              │     │              │
  │  Redis       │     │ Elasticsearch│     │ ClickHouse   │
  │  (Fast,      │     │ (Full-text   │     │ (Columnar,   │
  │   ephemeral) │     │  search)     │     │  OLAP)       │
  └──────────────┘     └──────────────┘     └──────────────┘
```

---

## 2. Database Sharding Strategies

```
Sharding = Splitting data across multiple databases to handle scale

WHY: Single database can't handle the load (writes, storage, connections)
WHEN: > 1TB of data OR > 10,000 write operations/sec

Types of Sharding:
┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  Range-Based Sharding:                                            │
│    Shard 1: Orders 1 - 1,000,000                                  │
│    Shard 2: Orders 1,000,001 - 2,000,000                          │
│    ✅ Simple, range queries efficient                              │
│    ❌ Hotspot: most recent shard gets all writes                   │
│                                                                    │
│  Hash-Based Sharding:                                             │
│    Shard = hash(userId) % num_shards                              │
│    ✅ Even distribution                                            │
│    ❌ Hard to add shards (re-hashing required)                     │
│                                                                    │
│  Geographic Sharding:                                             │
│    Shard 1: US customers → US-East DB                              │
│    Shard 2: EU customers → EU-West DB                              │
│    ✅ Data locality, compliance (GDPR)                             │
│    ❌ Cross-region queries expensive                               │
│                                                                    │
│  Tenant-Based Sharding:                                           │
│    Shard 1: Tenants A-M                                           │
│    Shard 2: Tenants N-Z                                           │
│    ✅ Natural isolation for multi-tenant                           │
│    ❌ Uneven if one tenant is very large                           │
└────────────────────────────────────────────────────────────────────┘
```

### Partitioning vs Sharding

```
Partitioning: Split data within ONE database (single server)
  → Table partitioning in SQL Server / PostgreSQL
  → Example: Orders partitioned by month

Sharding: Split data across MULTIPLE databases (multiple servers)
  → Each shard is an independent database
  → Example: US orders in US-DB, EU orders in EU-DB

Partitioning = vertical/horizontal split inside 1 DB
Sharding = horizontal split across many DBs
```

---

## 3. CQRS & Read/Write Separation

```
Problem: One model can't optimize for both complex writes AND fast reads

Solution: CQRS — Command Query Responsibility Segregation

  Write Path (Commands):                Read Path (Queries):
  ┌──────────────┐                      ┌──────────────┐
  │ POST/PUT/    │                      │ GET /orders  │
  │ DELETE       │                      │ GET /reports │
  │ /orders      │                      └──────┬───────┘
  └──────┬───────┘                             │
         │                                     │
  ┌──────┴───────┐   ──sync/async──▶   ┌──────┴───────┐
  │ Write DB     │   (event or CDC)    │ Read DB      │
  │ (Normalized, │                     │ (Denormalized,│
  │  ACID, SQL   │                     │  materialized │
  │  Server)     │                     │  views, Redis)│
  └──────────────┘                     └──────────────┘

Benefits:
  ✅ Scale reads independently (10x more reads than writes typical)
  ✅ Optimize read models for specific query patterns
  ✅ Write model stays clean and normalized

Costs:
  ❌ Eventual consistency between write and read models
  ❌ Complexity of keeping models in sync
  ❌ More infrastructure (2 databases + sync mechanism)
```

---

## 4. Data Migration Patterns

```
┌──────────────────────────────────────────────────────────────────────┐
│  Pattern           │  Description                │  Risk Level       │
├────────────────────┼─────────────────────────────┼───────────────────┤
│ Expand-Contract    │ Add new column → migrate    │  Low              │
│ (recommended)      │ data → remove old column    │  Zero downtime    │
├────────────────────┼─────────────────────────────┼───────────────────┤
│ Blue-Green DB      │ Two databases, switch after │  Medium           │
│                    │ migration + verification     │  Requires sync    │
├────────────────────┼─────────────────────────────┼───────────────────┤
│ Dual-Write         │ Write to old + new DB during│  Medium           │
│                    │ migration, switch reads      │  Consistency risk │
├────────────────────┼─────────────────────────────┼───────────────────┤
│ CDC (Change Data   │ Capture changes from old DB  │  Low-Medium       │
│  Capture)          │ and replay to new DB         │  Debezium, etc.   │
├────────────────────┼─────────────────────────────┼───────────────────┤
│ Big Bang           │ Stop system, migrate,        │  HIGH             │
│                    │ restart                       │  Downtime required│
└────────────────────┴─────────────────────────────┴───────────────────┘
```

### Expand-Contract Migration (Zero Downtime)

```
Phase 1: EXPAND — Add new column alongside old
  ALTER TABLE Users ADD EmailNormalized NVARCHAR(256) NULL;
  → Old code still works (ignores new column)
  → Deploy new code that writes to BOTH columns

Phase 2: MIGRATE — Backfill existing data
  UPDATE Users SET EmailNormalized = LOWER(Email) WHERE EmailNormalized IS NULL;
  → Run in batches to avoid locking

Phase 3: SWITCH — New code reads from new column only
  → Deploy code that reads from EmailNormalized

Phase 4: CONTRACT — Remove old column
  ALTER TABLE Users DROP COLUMN Email;
  → Only after all code is using the new column

Total downtime: ZERO
```

---
---

# Domain-Driven Design (DDD) in Software Architecture

---

## 1. What Is DDD?

**Domain-Driven Design** is an approach to software development that focuses on modeling software to match the **business domain**. The code structure reflects the real-world business, not technical concerns.

```
Traditional Approach:               DDD Approach:
──────────────────                  ─────────────
Project/                            Project/
├── Controllers/                    ├── Ordering/          ← Bounded Context
│   ├── OrderController             │   ├── Domain/
│   ├── ProductController           │   │   ├── Order.cs     (Aggregate Root)
│   └── UserController              │   │   ├── OrderItem.cs (Entity)
├── Models/                         │   │   ├── Money.cs     (Value Object)
│   ├── Order.cs                    │   │   └── OrderStatus.cs (Enum)
│   ├── Product.cs                  │   ├── Application/
│   └── User.cs                     │   │   ├── PlaceOrderCommand.cs
├── Services/                       │   │   └── PlaceOrderHandler.cs
│   ├── OrderService.cs             │   └── Infrastructure/
│   └── PaymentService.cs           │       ├── OrderRepository.cs
└── Data/                           │       └── OrderDbContext.cs
    └── AppDbContext.cs             ├── Catalog/           ← Bounded Context
                                    │   ├── Domain/
Organized by TECHNICAL layer        │   ├── Application/
                                    │   └── Infrastructure/
                                    Organized by BUSINESS domain
```

---

## 2. Strategic DDD — Bounded Contexts

### What Is a Bounded Context?

```
A Bounded Context is a boundary within which a domain model has a specific,
unambiguous meaning. The same real-world concept can mean different things
in different contexts.

Example: "Product" means different things in different contexts:

┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  Catalog Context │  │ Ordering Context│  │ Shipping Context│
│                  │  │                 │  │                 │
│  Product:        │  │  Product:       │  │  Product:       │
│  - Name          │  │  - ProductId    │  │  - ProductId    │
│  - Description   │  │  - Price        │  │  - Weight       │
│  - Images[]      │  │  - Quantity     │  │  - Dimensions   │
│  - Categories[]  │  │  - Discount     │  │  - IsFragile    │
│  - Reviews[]     │  │                 │  │                 │
│  - SEO metadata  │  │  (Only cares    │  │  (Only cares    │
│                  │  │   about pricing) │  │   about shipping│
│  (Rich product   │  │                 │  │   attributes)   │
│   information)   │  │                 │  │                 │
└─────────────────┘  └─────────────────┘  └─────────────────┘

Each context has its OWN model of Product.
They share a ProductId but nothing else.
```

### Context Mapping — How Bounded Contexts Interact

```
┌──────────────────────────────────────────────────────────────────────┐
│  Relationship          │  Description                                │
├────────────────────────┼────────────────────────────────────────────┤
│ Shared Kernel          │ Two contexts share a small common model    │
│                        │ (both teams must agree on changes)         │
├────────────────────────┼────────────────────────────────────────────┤
│ Customer-Supplier      │ Upstream supplies data, downstream consumes│
│                        │ Downstream can request changes from upstream│
├────────────────────────┼────────────────────────────────────────────┤
│ Conformist             │ Downstream conforms to upstream's model    │
│                        │ (no negotiation power)                     │
├────────────────────────┼────────────────────────────────────────────┤
│ Anti-Corruption Layer  │ Downstream translates upstream's model     │
│ (ACL)                  │ to protect its own domain model            │
├────────────────────────┼────────────────────────────────────────────┤
│ Open Host Service      │ Upstream exposes a well-defined protocol   │
│                        │ (API / events) for all consumers           │
├────────────────────────┼────────────────────────────────────────────┤
│ Published Language     │ Shared schema (JSON, Protobuf, Avro)       │
│                        │ used between contexts                      │
├────────────────────────┼────────────────────────────────────────────┤
│ Separate Ways          │ Contexts are completely independent        │
│                        │ (no integration needed)                     │
└────────────────────────┴────────────────────────────────────────────┘
```

---

## 3. Tactical DDD — Building Blocks

### Entities vs Value Objects vs Aggregates

```
┌──────────────────────────────────────────────────────────────────────┐
│  Building Block    │  Identity?  │  Mutable? │  Example              │
├────────────────────┼─────────────┼───────────┼───────────────────────┤
│ Entity             │ ✅ Has ID   │ ✅ Yes    │ Order, User, Product  │
│                    │ (identity   │           │ (tracked over time)   │
│                    │  matters)   │           │                       │
├────────────────────┼─────────────┼───────────┼───────────────────────┤
│ Value Object       │ ❌ No ID    │ ❌ Immutable│ Money, Address,     │
│                    │ (equality   │           │ Email, DateRange      │
│                    │  by value)  │           │ (replaced, not changed│
├────────────────────┼─────────────┼───────────┼───────────────────────┤
│ Aggregate          │ ✅ Root ID  │ ✅ Yes    │ Order (root) +        │
│                    │ (cluster of │           │ OrderItems (children) │
│                    │  entities)  │           │ Modified only through │
│                    │             │           │ the root              │
├────────────────────┼─────────────┼───────────┼───────────────────────┤
│ Domain Event       │ ✅ Event ID │ ❌ Immutable│ OrderPlaced,        │
│                    │             │           │ PaymentCompleted      │
├────────────────────┼─────────────┼───────────┼───────────────────────┤
│ Domain Service     │ ❌ No state │ N/A       │ PricingService,       │
│                    │ (stateless  │           │ TaxCalculator         │
│                    │  operation) │           │ (logic that doesn't   │
│                    │             │           │  belong to one entity) │
└────────────────────┴─────────────┴───────────┴───────────────────────┘
```

```csharp
// Value Object — immutable, equality by value
public record Money(decimal Amount, string Currency)
{
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add different currencies");
        return new Money(Amount + other.Amount, Currency);
    }
}

// Entity — has identity
public class OrderItem
{
    public Guid Id { get; private set; }
    public Guid ProductId { get; private set; }
    public int Quantity { get; private set; }
    public Money UnitPrice { get; private set; }
    public Money TotalPrice => new(UnitPrice.Amount * Quantity, UnitPrice.Currency);

    public void UpdateQuantity(int newQuantity)
    {
        if (newQuantity <= 0)
            throw new DomainException("Quantity must be positive");
        Quantity = newQuantity;
    }
}

// Aggregate Root — entry point for the entire aggregate
public class Order
{
    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    private readonly List<IDomainEvent> _events = new();
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _events.AsReadOnly();

    public static Order Create(Guid customerId)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            Status = OrderStatus.Draft
        };
        order._events.Add(new OrderCreatedEvent(order.Id, customerId));
        return order;
    }

    public void AddItem(Guid productId, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Can only add items to draft orders");

        var existing = _items.FirstOrDefault(i => i.ProductId == productId);
        if (existing is not null)
            existing.UpdateQuantity(existing.Quantity + quantity);
        else
            _items.Add(new OrderItem(productId, quantity, unitPrice));
    }

    public void Place()
    {
        if (!_items.Any())
            throw new DomainException("Cannot place an empty order");
        if (Status != OrderStatus.Draft)
            throw new DomainException("Order is not in draft status");

        Status = OrderStatus.Placed;
        _events.Add(new OrderPlacedEvent(Id, CustomerId, TotalAmount));
    }

    public Money TotalAmount => _items
        .Aggregate(new Money(0, "USD"), (sum, item) => sum.Add(item.TotalPrice));
}
```

### Aggregate Design Rules

```
Rule 1: Reference other aggregates by ID only
  ❌ Order has a Customer navigation property
  ✅ Order has a CustomerId (Guid)

Rule 2: Keep aggregates small
  ❌ Order aggregate contains Customer, Products, Payments, Shipments
  ✅ Order aggregate contains only OrderItems

Rule 3: One transaction = one aggregate
  ❌ Save Order + update Inventory + charge Payment in one transaction
  ✅ Save Order → publish event → Inventory subscribes → Payment subscribes

Rule 4: Enforce invariants inside the aggregate
  ❌ Service checks "can add item?" then adds item (two steps)
  ✅ Order.AddItem() validates and adds atomically
```

---

## 4. DDD with Clean Architecture

```
┌────────────────────────────────────────────────────────┐
│                                                        │
│  Presentation (API / Controllers)                      │
│  ┌──────────────────────────────────────────────────┐ │
│  │                                                  │ │
│  │  Application (Use Cases / Commands / Queries)    │ │
│  │  ┌──────────────────────────────────────────┐   │ │
│  │  │                                          │   │ │
│  │  │  Domain (Entities, Value Objects,         │   │ │
│  │  │         Aggregates, Domain Events,        │   │ │
│  │  │         Repository Interfaces)            │   │ │
│  │  │                                          │   │ │
│  │  └──────────────────────────────────────────┘   │ │
│  │                                                  │ │
│  └──────────────────────────────────────────────────┘ │
│                                                        │
│  Infrastructure (EF Core, External APIs, Message Bus)  │
│                                                        │
└────────────────────────────────────────────────────────┘

Dependency Rule: Inner layers NEVER depend on outer layers
  Domain → depends on nothing
  Application → depends on Domain
  Infrastructure → depends on Domain + Application
  Presentation → depends on Application
```

---
---

# Event-Driven Architecture (EDA)

---

## 1. What Is Event-Driven Architecture?

```
EDA is an architecture pattern where components communicate through
EVENTS — something that happened in the past (immutable facts).

Request-Driven (Synchronous):
  Order Service ──HTTP──▶ Payment Service ──HTTP──▶ Inventory Service
  (Coupled, blocking, cascading failures)

Event-Driven (Asynchronous):
  Order Service ──publishes──▶ "OrderPlaced" event
                               ├──▶ Payment Service (subscribes, charges card)
                               ├──▶ Inventory Service (subscribes, reserves stock)
                               ├──▶ Notification Service (subscribes, sends email)
                               └──▶ Analytics Service (subscribes, records metric)
  (Decoupled, non-blocking, each service independent)
```

---

## 2. Event Types

```
┌──────────────────────────────────────────────────────────────────────┐
│  Event Type        │  Description                │  Example          │
├────────────────────┼─────────────────────────────┼───────────────────┤
│ Domain Event       │ Something meaningful that    │ OrderPlaced       │
│                    │ happened in the domain       │ PaymentCompleted  │
├────────────────────┼─────────────────────────────┼───────────────────┤
│ Integration Event  │ Cross-service event          │ UserRegistered    │
│                    │ (published to message broker) │ ProductPriceChanged│
├────────────────────┼─────────────────────────────┼───────────────────┤
│ Notification Event │ "Something happened" — thin  │ { orderId: 123 } │
│ (thin)             │ (consumer fetches details)   │ (no payload)      │
├────────────────────┼─────────────────────────────┼───────────────────┤
│ Event-Carried      │ "Something happened + here's │ { orderId: 123,  │
│ State Transfer     │ all the data you need"       │   items: [...],   │
│ (fat)              │ (consumer doesn't call back) │   total: 99.99 } │
└────────────────────┴─────────────────────────────┴───────────────────┘
```

---

## 3. Messaging Patterns

### Pub/Sub vs Point-to-Point

```
Point-to-Point (Queue):
  Producer ──▶ Queue ──▶ ONE Consumer
  (Each message processed by exactly one consumer)
  Use for: Order processing, task distribution

Pub/Sub (Topic):
  Publisher ──▶ Topic ──▶ Subscriber A
                     ├──▶ Subscriber B
                     └──▶ Subscriber C
  (Each message delivered to ALL subscribers)
  Use for: Event notification, data sync
```

### Message Broker Selection

```
┌──────────────────────────────────────────────────────────────────────┐
│  Broker          │ Strengths                 │ Best For              │
├──────────────────┼───────────────────────────┼───────────────────────┤
│ RabbitMQ         │ Flexible routing, mature, │ Task queues, RPC,     │
│                  │ easy to operate            │ moderate throughput   │
├──────────────────┼───────────────────────────┼───────────────────────┤
│ Apache Kafka     │ High throughput, durable, │ Event streaming, log  │
│                  │ replay, ordering           │ aggregation, CDC      │
├──────────────────┼───────────────────────────┼───────────────────────┤
│ Azure Service Bus│ Enterprise features, DLQ, │ Enterprise async      │
│                  │ sessions, transactions     │ messaging, .NET apps  │
├──────────────────┼───────────────────────────┼───────────────────────┤
│ AWS SQS/SNS      │ Serverless, managed,      │ Cloud-native, low     │
│                  │ no infrastructure          │ maintenance           │
├──────────────────┼───────────────────────────┼───────────────────────┤
│ Redis Streams    │ Low latency, in-memory    │ Real-time events,     │
│                  │                           │ lightweight pub/sub   │
└──────────────────┴───────────────────────────┴───────────────────────┘
```

---

## 4. Transactional Outbox Pattern — Reliable Event Publishing

```
Problem: How to atomically save to DB AND publish an event?

❌ WRONG — Two separate operations (can fail independently):
  1. Save order to DB           ← succeeds
  2. Publish OrderPlaced event  ← fails (broker down)
  Result: Order saved but event never published → inconsistency

✅ SOLUTION — Outbox Pattern:
  1. Save order + outbox message in SAME database transaction
  2. Background worker reads outbox and publishes to broker
  3. Mark outbox message as published

  ┌─────────────────────────────┐
  │    Database Transaction      │
  │                              │
  │  INSERT INTO Orders (...)    │
  │  INSERT INTO Outbox (        │
  │    EventType: 'OrderPlaced', │
  │    Payload: '{...}',         │
  │    Published: false          │
  │  )                           │
  │  COMMIT                      │
  └──────────────┬───────────────┘
                 │
  ┌──────────────┴───────────────┐
  │  Outbox Worker (background)   │
  │  1. SELECT FROM Outbox        │
  │     WHERE Published = false   │
  │  2. Publish to message broker │
  │  3. UPDATE Published = true   │
  └───────────────────────────────┘
```

```csharp
// Outbox implementation
public class OutboxMessage
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string EventType { get; set; } = "";
    public string Payload { get; set; } = "";
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public bool Published { get; set; } = false;
    public DateTime? PublishedAt { get; set; }
}

// Save order + outbox message in same transaction
public async Task PlaceOrderAsync(Order order)
{
    await using var transaction = await _dbContext.Database.BeginTransactionAsync();

    _dbContext.Orders.Add(order);

    _dbContext.OutboxMessages.Add(new OutboxMessage
    {
        EventType = nameof(OrderPlacedEvent),
        Payload = JsonSerializer.Serialize(new OrderPlacedEvent(
            order.Id, order.CustomerId, order.TotalAmount))
    });

    await _dbContext.SaveChangesAsync();
    await transaction.CommitAsync();
}

// Background worker publishes outbox messages
public class OutboxPublisher : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var messages = await _dbContext.OutboxMessages
                .Where(m => !m.Published)
                .OrderBy(m => m.CreatedAt)
                .Take(100)
                .ToListAsync(ct);

            foreach (var message in messages)
            {
                await _messageBroker.PublishAsync(message.EventType, message.Payload);
                message.Published = true;
                message.PublishedAt = DateTime.UtcNow;
            }

            await _dbContext.SaveChangesAsync(ct);
            await Task.Delay(TimeSpan.FromSeconds(1), ct);
        }
    }
}
```

---

## 5. Dead Letter Queue (DLQ) — Handling Poison Messages

```
When a message fails processing repeatedly, move it to a DLQ
instead of blocking the entire queue.

Normal Flow:
  Queue ──▶ Consumer ──▶ Process ──▶ ACK (success)

Failure Flow:
  Queue ──▶ Consumer ──▶ Process FAILS
        ◀── NACK (retry) ──┘
  Queue ──▶ Consumer ──▶ Process FAILS (2nd time)
        ◀── NACK (retry) ──┘
  Queue ──▶ Consumer ──▶ Process FAILS (3rd time)
        ──▶ Dead Letter Queue (DLQ) ──▶ Manual investigation

DLQ Investigation:
  1. Alert fires: "5 messages in DLQ for OrderProcessing"
  2. Engineer inspects the message payload
  3. Fix the bug or data issue
  4. Replay the message from DLQ → original queue
```

---

## 6. Idempotent Consumers

```
Problem: Broker guarantees at-least-once delivery → consumer may get
         the same message twice → double-charging customer!

Solution: Make consumers idempotent

1. Track processed message IDs:

   IF NOT EXISTS (SELECT 1 FROM ProcessedEvents WHERE EventId = @eventId)
   BEGIN
       -- Process the message
       INSERT INTO ProcessedEvents (EventId, ProcessedAt) VALUES (@eventId, GETUTCDATE())
   END
   -- ELSE: Skip (already processed)

2. Use natural idempotency keys:
   OrderPlaced → check if payment already exists for OrderId
   PaymentCompleted → check if shipment already initiated for PaymentId
```

---
---

# Multi-Tenancy Architecture

---

## 1. What Is Multi-Tenancy?

```
Multi-tenancy: ONE application instance serves MULTIPLE customers (tenants).
Each tenant's data is isolated from others.

Single-Tenant:                    Multi-Tenant:
┌──────────┐ ┌──────────┐       ┌──────────────────────────────────┐
│ App for  │ │ App for  │       │  Shared Application              │
│ Tenant A │ │ Tenant B │       │  ┌──────┐ ┌──────┐ ┌──────┐    │
│          │ │          │       │  │Tenant│ │Tenant│ │Tenant│    │
│ Own DB   │ │ Own DB   │       │  │  A   │ │  B   │ │  C   │    │
└──────────┘ └──────────┘       │  └──────┘ └──────┘ └──────┘    │
(Expensive, simple)              └──────────────────────────────────┘
                                 (Cost-effective, complex isolation)
```

---

## 2. Data Isolation Strategies

```
┌──────────────────────────────────────────────────────────────────────┐
│  Strategy            │  Isolation │  Cost   │  Complexity │ Example  │
├──────────────────────┼───────────┼─────────┼─────────────┼──────────┤
│ Database-per-Tenant  │  Highest  │  High   │  Medium     │ Salesforce│
│ (separate DB)        │           │         │             │ (enterprise)│
├──────────────────────┼───────────┼─────────┼─────────────┼──────────┤
│ Schema-per-Tenant    │  High     │  Medium │  Medium     │ PostgreSQL│
│ (same DB, diff       │           │         │             │ schemas  │
│  schema/namespace)   │           │         │             │          │
├──────────────────────┼───────────┼─────────┼─────────────┼──────────┤
│ Shared DB +          │  Medium   │  Low    │  Low        │ Most SaaS│
│ TenantId column      │           │         │ (but risky) │ products │
│ (row-level isolation)│           │         │             │          │
├──────────────────────┼───────────┼─────────┼─────────────┼──────────┤
│ Hybrid               │  Varies   │  Medium │  High       │ Premium  │
│ (big tenants get own │           │         │             │ = own DB │
│  DB, small share)    │           │         │             │          │
└──────────────────────┴───────────┴─────────┴─────────────┴──────────┘
```

### EF Core Global Query Filter for Multi-Tenancy

```csharp
// Shared DB with TenantId column — EF Core automatically filters by tenant

public class MultiTenantDbContext : DbContext
{
    private readonly ITenantProvider _tenantProvider;

    public MultiTenantDbContext(DbContextOptions options, ITenantProvider tenantProvider)
        : base(options)
    {
        _tenantProvider = tenantProvider;
    }

    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        // Global query filter — ALL queries automatically filter by TenantId
        builder.Entity<Product>()
            .HasQueryFilter(p => p.TenantId == _tenantProvider.TenantId);
        builder.Entity<Order>()
            .HasQueryFilter(o => o.TenantId == _tenantProvider.TenantId);
    }

    public override Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        // Auto-set TenantId on new entities
        foreach (var entry in ChangeTracker.Entries<ITenantEntity>()
            .Where(e => e.State == EntityState.Added))
        {
            entry.Entity.TenantId = _tenantProvider.TenantId;
        }
        return base.SaveChangesAsync(ct);
    }
}

// Tenant resolution from JWT / subdomain / header
public class TenantProvider : ITenantProvider
{
    private readonly IHttpContextAccessor _httpContext;

    public Guid TenantId =>
        Guid.Parse(_httpContext.HttpContext?.User
            .FindFirst("tenant_id")?.Value ?? throw new UnauthorizedAccessException());
}
```

---

## 3. Tenant Resolution Strategies

```
┌──────────────────────────────────────────────────────────────────────┐
│  Strategy         │  Example                    │  Best For           │
├───────────────────┼─────────────────────────────┼─────────────────────┤
│ Subdomain         │ acme.app.com                │ B2B SaaS products  │
│                   │ contoso.app.com             │ Clean URL branding │
├───────────────────┼─────────────────────────────┼─────────────────────┤
│ Path prefix       │ app.com/acme/orders         │ Simple routing     │
│                   │ app.com/contoso/orders       │                    │
├───────────────────┼─────────────────────────────┼─────────────────────┤
│ Header            │ X-Tenant-Id: acme-123       │ API-only (no UI)   │
├───────────────────┼─────────────────────────────┼─────────────────────┤
│ JWT claim         │ { "tenant_id": "acme-123" } │ Token-based auth   │
├───────────────────┼─────────────────────────────┼─────────────────────┤
│ Custom domain     │ orders.acme.com             │ White-label SaaS   │
│                   │ (CNAME to app)              │                    │
└───────────────────┴─────────────────────────────┴─────────────────────┘
```

---
---

# Migration Strategies in Software Architecture

---

## 1. Migration Strategy Selection

```
┌──────────────────────────────────────────────────────────────────────┐
│  Strategy              │ Risk  │ Downtime │ When to Use              │
├────────────────────────┼───────┼──────────┼──────────────────────────┤
│ Strangler Fig          │ Low   │ Zero     │ Monolith → microservices │
│ (incremental replace)  │       │          │ Large legacy systems     │
├────────────────────────┼───────┼──────────┼──────────────────────────┤
│ Parallel Run           │ Low   │ Zero     │ Critical systems where   │
│ (old + new together)   │       │          │ correctness is paramount │
├────────────────────────┼───────┼──────────┼──────────────────────────┤
│ Branch by Abstraction  │ Low   │ Zero     │ Replacing internal       │
│ (swap implementation)  │       │          │ components / libraries   │
├────────────────────────┼───────┼──────────┼──────────────────────────┤
│ Feature Toggle         │ Low   │ Zero     │ Gradual rollout of new   │
│ Migration              │       │          │ functionality            │
├────────────────────────┼───────┼──────────┼──────────────────────────┤
│ Big Bang               │ HIGH  │ Yes      │ Small systems, complete  │
│ (replace everything)   │       │          │ rewrite (last resort)    │
└────────────────────────┴───────┴──────────┴──────────────────────────┘
```

---

## 2. Strangler Fig Pattern (Most Important)

```
Named after strangler fig trees that grow around a host tree until it dies.

Phase 1: Identify a bounded context to extract
  ┌────────────────────────────────────┐
  │         Monolith                   │
  │  ┌──────┐ ┌──────┐ ┌──────┐      │
  │  │ User │ │Order │ │Product│      │
  │  │      │ │      │ │  ←── │ Extract this first
  │  └──────┘ └──────┘ └──────┘      │
  └────────────────────────────────────┘

Phase 2: Build new service alongside the monolith
  ┌────────────────────────┐     ┌──────────────┐
  │     Monolith           │     │ Product      │
  │  ┌──────┐ ┌──────┐    │     │ Microservice │
  │  │ User │ │Order │    │     │ (new)        │
  │  └──────┘ └──────┘    │     └──────────────┘
  └────────────────────────┘

Phase 3: Route traffic to new service via proxy/gateway
  Client ──▶ API Gateway
              ├── /products/* ──▶ Product Microservice (new)
              └── /users/*    ──▶ Monolith (old)
              └── /orders/*   ──▶ Monolith (old)

Phase 4: Repeat for each bounded context until monolith is empty

Key Rules:
  ✅ Extract one context at a time
  ✅ Run old and new in parallel for validation
  ✅ Use an anti-corruption layer between old and new
  ❌ Never do a full rewrite (Big Bang) — it almost always fails
```

---

## 3. Parallel Run Strategy

```
Run old and new systems simultaneously, compare results:

  Request ──▶ Router/Proxy
              ├── Old System ──▶ Response A (returned to user)
              └── New System ──▶ Response B (logged for comparison)

  Compare A and B:
    Match → new system is correct ✅
    Mismatch → investigate and fix new system

  Gradually increase trust:
    Week 1: Return old response, log new
    Week 2: Return old response, alert on mismatch
    Week 3: Return NEW response, log old for safety
    Week 4: Retire old system

  Used by: Banks, payment processors, healthcare systems
  (Where getting it wrong is catastrophic)
```

---
---

# Technical Debt Management

---

## 1. What Is Technical Debt?

```
Technical debt = shortcuts taken during development that create
future work. Like financial debt: you pay interest over time.

Types:
┌──────────────────────────────────────────────────────────────────────┐
│  Type              │  Description               │  Example           │
├────────────────────┼────────────────────────────┼────────────────────┤
│ Deliberate /       │ Known shortcut taken        │ "Skip unit tests  │
│ Prudent            │ consciously for speed       │  to hit deadline"  │
├────────────────────┼────────────────────────────┼────────────────────┤
│ Deliberate /       │ Known bad practice done     │ "We don't do      │
│ Reckless           │ anyway                      │  code reviews"     │
├────────────────────┼────────────────────────────┼────────────────────┤
│ Inadvertent /      │ Didn't know better at time  │ "Now I see how    │
│ Prudent            │ but learned later           │  we should have    │
│                    │                             │  designed it"      │
├────────────────────┼────────────────────────────┼────────────────────┤
│ Inadvertent /      │ Team lacks knowledge or     │ "What's a design  │
│ Reckless           │ skills                      │  pattern?"         │
└────────────────────┴────────────────────────────┴────────────────────┘
```

---

## 2. Technical Debt Prioritization Matrix

```
┌────────────────────────────────────────────────────────────────────┐
│           High Impact                                              │
│     ┌──────────────────┬───────────────────┐                      │
│     │  Fix NOW         │  Plan for next    │                      │
│     │  (Sprint 0)      │  quarter          │                      │
│     │                  │                   │                      │
│     │  • Security      │  • Architecture   │                      │
│     │    vulnerabilities│   refactoring    │                      │
│     │  • Data integrity│  • Performance    │                      │
│     │    bugs          │    optimization   │                      │
│     │  • Production    │  • Test coverage  │                      │
│     │    stability     │    gaps           │                      │
│     ├──────────────────┼───────────────────┤                      │
│     │  Opportunistic   │  Ignore / Accept  │                      │
│     │  (Fix when       │  (Not worth the   │                      │
│     │   touching area) │   investment)     │                      │
│     │                  │                   │                      │
│     │  • Code style    │  • Legacy module  │                      │
│     │    inconsistency │    nobody touches │                      │
│     │  • Minor         │  • Cosmetic code  │                      │
│     │    duplication   │    issues         │                      │
│     └──────────────────┴───────────────────┘                      │
│           Low Impact                                               │
│     Low Effort ◄──────────────────────────▶ High Effort           │
└────────────────────────────────────────────────────────────────────┘
```

### The 20% Rule

```
Sustainable approach to managing tech debt:

  Sprint capacity: 100%
  ├── 80% → Feature development
  └── 20% → Technical debt repayment

  This prevents debt from accumulating beyond control.
  Track tech debt items in the backlog with "tech-debt" label.
  Review and prioritize quarterly.
```

---
---

# Cost Optimization / FinOps in Architecture

---

## 1. Cloud Cost Optimization Strategies

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Cost Optimization Strategies                       │
├──────────────────────┬───────────────────────────────────────────────┤
│ Right-Sizing         │ Match instance size to actual usage.          │
│                      │ Most apps are OVER-provisioned.               │
│                      │ Monitor CPU/memory for 2 weeks, then resize. │
├──────────────────────┼───────────────────────────────────────────────┤
│ Reserved Instances   │ Commit to 1-3 years for 30-60% discount.     │
│                      │ Good for: databases, steady-state services.  │
├──────────────────────┼───────────────────────────────────────────────┤
│ Spot / Preemptible   │ Use surplus capacity at 60-90% discount.     │
│                      │ Good for: batch jobs, stateless workers.     │
│                      │ Risk: can be terminated with 2 min notice.   │
├──────────────────────┼───────────────────────────────────────────────┤
│ Auto-Scaling         │ Scale down during off-peak hours.             │
│                      │ E-commerce: 10 instances at noon, 2 at 3 AM. │
├──────────────────────┼───────────────────────────────────────────────┤
│ Serverless           │ Pay per execution (Lambda, Azure Functions).  │
│                      │ Good for: event-driven, low/variable traffic.│
│                      │ Zero cost when idle.                          │
├──────────────────────┼───────────────────────────────────────────────┤
│ Data Tiering         │ Hot data → SSD, Warm data → HDD,             │
│                      │ Cold data → Archive (S3 Glacier).            │
│                      │ Move data automatically based on access age.  │
├──────────────────────┼───────────────────────────────────────────────┤
│ Caching              │ Cache frequent reads → fewer DB calls         │
│                      │ → smaller DB instance needed → lower cost.   │
└──────────────────────┴───────────────────────────────────────────────┘
```

---

## 2. Architecture Decisions That Impact Cost

```
Decision                    Cheap Option              Expensive Option
────────                    ────────────              ────────────────
Deployment model            Shared / multi-tenant     Dedicated per customer
Database                    Single DB, shared schema  DB-per-service
Communication               Synchronous HTTP          Message broker cluster
Caching                     In-memory (local)         Distributed Redis cluster
Observability               Basic logging             Full OTel + Jaeger + Grafana
Environments                Dev + Prod                Dev + QA + Staging + Prod
Region                      Single region             Multi-region active-active
Availability target         99.9% (basic)             99.999% (10x cost per 9)
```

### Cost Per Request Thinking

```
Know your cost-per-request for each service:

Order Service:
  Infrastructure cost / month:  $2,000
  Requests / month:             1,000,000
  Cost per request:             $0.002

Payment Service:
  Infrastructure cost / month:  $5,000
  Requests / month:             500,000
  Cost per request:             $0.01

→ Identify expensive services
→ Optimize the highest cost-per-request services first
→ Can the $0.01 Payment service use a cheaper alternative?
```

---
---

# Performance Engineering in Architecture

---

## 1. Performance Budgets

```
Define performance budgets BEFORE development — just like financial budgets.

API Performance Budget:
┌─────────────────────┬────────────┬──────────────┬──────────────┐
│ Endpoint            │ p50 Target │ p95 Target   │ p99 Target   │
├─────────────────────┼────────────┼──────────────┼──────────────┤
│ GET /products       │ < 50ms     │ < 200ms      │ < 500ms      │
│ GET /products/{id}  │ < 30ms     │ < 100ms      │ < 200ms      │
│ POST /orders        │ < 200ms    │ < 500ms      │ < 1000ms     │
│ POST /payments      │ < 500ms    │ < 2000ms     │ < 3000ms     │
│ GET /search         │ < 100ms    │ < 300ms      │ < 500ms      │
└─────────────────────┴────────────┴──────────────┴──────────────┘

p50 = median (50% of requests are faster)
p95 = 95th percentile (5% of requests are slower than this)
p99 = 99th percentile (1% of requests are slower — tail latency)
```

---

## 2. Performance Anti-Patterns

```
┌────────────────────────────────────────────────────────────────────────┐
│  Anti-Pattern         │  Problem                │  Solution            │
├───────────────────────┼─────────────────────────┼──────────────────────┤
│ N+1 Query             │ 1 query for list +      │ .Include() in EF     │
│                       │ N queries for details   │ Core, batch loading  │
├───────────────────────┼─────────────────────────┼──────────────────────┤
│ Over-fetching         │ SELECT * when you need  │ .Select() projection │
│                       │ 3 columns               │ specific columns     │
├───────────────────────┼─────────────────────────┼──────────────────────┤
│ Chatty APIs           │ 10 API calls for 1 page │ BFF pattern,         │
│                       │ render                  │ GraphQL, aggregation │
├───────────────────────┼─────────────────────────┼──────────────────────┤
│ Synchronous chains    │ A → B → C → D (each     │ Parallel calls,      │
│                       │ waits for previous)     │ async, message queue │
├───────────────────────┼─────────────────────────┼──────────────────────┤
│ No caching            │ Same data computed on   │ Cache-aside,         │
│                       │ every request           │ HTTP cache headers   │
├───────────────────────┼─────────────────────────┼──────────────────────┤
│ Large payloads        │ Returning 10MB JSON     │ Pagination, field    │
│                       │ response                │ selection, compress  │
├───────────────────────┼─────────────────────────┼──────────────────────┤
│ Missing indexes       │ Full table scan on      │ Add proper indexes,  │
│                       │ every query             │ query plan analysis  │
├───────────────────────┼─────────────────────────┼──────────────────────┤
│ Unbounded queries     │ SELECT * FROM Orders    │ Always paginate,     │
│                       │ (returns 1M rows)       │ always LIMIT         │
└───────────────────────┴─────────────────────────┴──────────────────────┘
```

---

## 3. Capacity Planning

```
Formula:
  Required capacity = (Peak requests/sec) × (Avg response time) × (Safety factor)

Example:
  Peak: 5,000 requests/sec
  Avg response: 200ms = 0.2 seconds
  Safety factor: 2x (for burst headroom)

  Concurrent connections = 5,000 × 0.2 × 2 = 2,000

  If each instance handles 200 concurrent connections:
  Instances needed = 2,000 / 200 = 10 instances

Load Testing Tools:
  • k6 (JavaScript-based, modern)
  • Apache JMeter (Java, GUI-based)
  • NBomber (.NET native)
  • Locust (Python)
  • Artillery (Node.js)
```

---
---

# 12-Factor App / Cloud-Native Principles

---

## 1. The 12 Factors

```
┌────┬─────────────────────┬───────────────────────────────────────────┐
│ #  │ Factor              │ Principle                                 │
├────┼─────────────────────┼───────────────────────────────────────────┤
│ 1  │ Codebase            │ One codebase tracked in version control,  │
│    │                     │ many deploys (dev, staging, prod)         │
├────┼─────────────────────┼───────────────────────────────────────────┤
│ 2  │ Dependencies        │ Explicitly declare and isolate            │
│    │                     │ dependencies (NuGet, npm)                 │
├────┼─────────────────────┼───────────────────────────────────────────┤
│ 3  │ Config              │ Store config in environment variables     │
│    │                     │ (never in code)                           │
├────┼─────────────────────┼───────────────────────────────────────────┤
│ 4  │ Backing Services    │ Treat databases, queues, caches as        │
│    │                     │ attached resources (swappable via config) │
├────┼─────────────────────┼───────────────────────────────────────────┤
│ 5  │ Build, Release, Run │ Strictly separate build, release, and    │
│    │                     │ run stages (CI/CD pipeline)               │
├────┼─────────────────────┼───────────────────────────────────────────┤
│ 6  │ Processes           │ Execute app as stateless processes        │
│    │                     │ (no sticky sessions, no local file state) │
├────┼─────────────────────┼───────────────────────────────────────────┤
│ 7  │ Port Binding        │ Export services via port binding           │
│    │                     │ (self-contained, no app server needed)    │
├────┼─────────────────────┼───────────────────────────────────────────┤
│ 8  │ Concurrency         │ Scale out via process model               │
│    │                     │ (more instances, not bigger instances)    │
├────┼─────────────────────┼───────────────────────────────────────────┤
│ 9  │ Disposability       │ Fast startup, graceful shutdown           │
│    │                     │ (handle SIGTERM, drain connections)       │
├────┼─────────────────────┼───────────────────────────────────────────┤
│ 10 │ Dev/Prod Parity     │ Keep dev, staging, and prod as similar   │
│    │                     │ as possible (same DB, same config)        │
├────┼─────────────────────┼───────────────────────────────────────────┤
│ 11 │ Logs                │ Treat logs as event streams               │
│    │                     │ (write to stdout, let platform capture)   │
├────┼─────────────────────┼───────────────────────────────────────────┤
│ 12 │ Admin Processes     │ Run admin/management tasks as one-off     │
│    │                     │ processes (migrations, scripts)           │
└────┴─────────────────────┴───────────────────────────────────────────┘
```

### Cloud-Native Beyond 12 Factors

```
Additional principles for modern cloud-native apps:

13. API-First         → Design APIs before implementation
14. Telemetry         → Built-in observability (logs, metrics, traces)
15. Security          → Security baked in, not bolted on (DevSecOps)
16. Resilience        → Handle failures gracefully (circuit breakers, retries)
17. Infrastructure    → Infrastructure as Code (Terraform, Pulumi)
    as Code
```

---
---

# Evolutionary Architecture

---

## 1. What Is Evolutionary Architecture?

```
"An evolutionary architecture supports guided, incremental change
 across multiple dimensions."
    — Neal Ford, Rebecca Parsons, Patrick Kua

Traditional:  Design everything upfront → build → hope it works
Evolutionary: Start simple → measure → evolve based on real data

Key Concept: Architecture should change as requirements change.
             Design for changeability, not for the final state.
```

---

## 2. Fitness Functions — Automated Architecture Validation

```
Fitness functions are automated tests that verify architectural
characteristics are maintained as the system evolves.

Examples:
┌────────────────────────┬────────────────────────────────────────────┐
│ Architectural Goal     │ Fitness Function                           │
├────────────────────────┼────────────────────────────────────────────┤
│ Performance            │ p99 latency < 500ms (run in CI/CD)        │
│                        │ Fail build if performance degrades         │
├────────────────────────┼────────────────────────────────────────────┤
│ Modularity             │ No circular dependencies between modules  │
│                        │ (ArchUnit / NetArchTest)                   │
├────────────────────────┼────────────────────────────────────────────┤
│ Security               │ No high-severity CVEs in dependencies     │
│                        │ (Dependabot / Snyk in pipeline)            │
├────────────────────────┼────────────────────────────────────────────┤
│ Code quality           │ Test coverage > 80%                       │
│                        │ Cyclomatic complexity < 15 per method     │
├────────────────────────┼────────────────────────────────────────────┤
│ Coupling               │ Domain layer has zero dependency on       │
│                        │ infrastructure (ArchUnit test)             │
├────────────────────────┼────────────────────────────────────────────┤
│ API compatibility      │ No breaking changes in public API         │
│                        │ (contract testing with Pact)               │
└────────────────────────┴────────────────────────────────────────────┘
```

```csharp
// NetArchTest — enforce architectural rules in C#
[Fact]
public void Domain_Should_Not_Depend_On_Infrastructure()
{
    var result = Types.InAssembly(typeof(Order).Assembly)
        .ShouldNot()
        .HaveDependencyOn("Infrastructure")
        .GetResult();

    Assert.True(result.IsSuccessful);
}

[Fact]
public void Controllers_Should_Not_Directly_Access_Repositories()
{
    var result = Types.InAssembly(typeof(OrderController).Assembly)
        .That().ResideInNamespace("Controllers")
        .ShouldNot()
        .HaveDependencyOn("Repositories")
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

---

## 3. Architecture Runway

```
Architecture Runway = the set of infrastructure, patterns, and
frameworks that allow feature teams to build without being blocked
by architectural concerns.

  Runway Too Short → Teams blocked, need architectural work before features
  Runway Too Long  → Over-engineering, YAGNI, wasted effort

Ideal: 2-3 sprints of runway ahead of feature teams

  Sprint 1-2 (Platform team):
    → Set up CI/CD pipeline
    → Configure observability (OTel + Grafana)
    → Create API Gateway + auth
    → Create shared libraries (error handling, logging)

  Sprint 3+ (Feature teams):
    → Build features on top of the runway
    → Platform team extends runway as needed
```

---
---

# Compliance Architecture

---

## 1. Key Compliance Frameworks

```
┌──────────────────────────────────────────────────────────────────────┐
│  Framework   │  Focus              │  Architecture Impact            │
├──────────────┼─────────────────────┼─────────────────────────────────┤
│ SOC 2        │ Security, privacy,  │ Audit logging, access control, │
│ (Type II)    │ availability        │ change management, encryption  │
├──────────────┼─────────────────────┼─────────────────────────────────┤
│ ISO 27001    │ Information security│ Risk assessment, ISMS, controls│
│              │ management          │ monitoring, incident mgmt      │
├──────────────┼─────────────────────┼─────────────────────────────────┤
│ GDPR         │ Data privacy (EU)   │ PII encryption, consent mgmt, │
│              │                     │ data minimization, erasure     │
├──────────────┼─────────────────────┼─────────────────────────────────┤
│ HIPAA        │ Healthcare data (US)│ PHI encryption, access audit,  │
│              │                     │ BAA with vendors               │
├──────────────┼─────────────────────┼─────────────────────────────────┤
│ PCI DSS      │ Payment card data   │ Network segmentation,          │
│              │                     │ tokenization, key management   │
├──────────────┼─────────────────────┼─────────────────────────────────┤
│ FedRAMP      │ US Government cloud │ Data residency, FIPS 140-2    │
│              │                     │ encryption, continuous ATO     │
└──────────────┴─────────────────────┴─────────────────────────────────┘
```

---

## 2. Compliance-Driven Architecture Decisions

```
Separation of Duties:
  ❌ Developer can deploy to production AND access production data
  ✅ Developer deploys → pipeline → approval gate → production
     Production DB access requires separate privileged identity

Audit Trail Requirements:
  WHO did WHAT, WHEN, from WHERE — immutable, tamper-proof
  → Append-only audit log (never delete or modify)
  → Write to separate audit database (different access controls)
  → Retain for 7 years (SOC 2 / financial regulations)

Data Residency:
  GDPR → EU citizen data must stay in EU
  → Deploy EU-region infrastructure for EU customers
  → Use geographic sharding or tenant-based routing

Change Management:
  Every production change must be:
  □ Reviewed (PR with at least 2 approvers)
  □ Tested (automated tests pass)
  □ Approved (change advisory board for major changes)
  □ Documented (ADR + deployment log)
  □ Reversible (rollback plan)
```

---

## 3. Compliance as Code

```csharp
// Automated compliance checks in CI/CD pipeline

// 1. Infrastructure compliance (Terraform + OPA)
//    → Verify no public S3 buckets, encryption enabled, etc.

// 2. Code compliance (SonarQube + custom rules)
//    → No hardcoded secrets, no SQL concatenation, etc.

// 3. Dependency compliance (Snyk / Dependabot)
//    → No high-severity CVEs, approved licenses only

// 4. Access compliance (Azure Policy / AWS Config)
//    → MFA enforced, no root account usage, etc.

// Pipeline:
//   Build → Test → Security Scan → Compliance Check → Deploy
//   If ANY compliance check fails → block deployment
```

---
---

# Platform Engineering

---

## 1. What Is Platform Engineering?

```
Platform Engineering = building an INTERNAL DEVELOPER PLATFORM (IDP)
that provides self-service infrastructure and tools to product teams.

Without Platform Team:                With Platform Team:
Each team reinvents the wheel         Teams use shared platform

Team A: Sets up own CI/CD            Platform provides:
Team B: Sets up own CI/CD            → One CI/CD for all teams
Team C: Sets up own CI/CD            → One observability stack
Team A: Configures own monitoring    → One deployment pipeline
Team B: Configures own monitoring    → One secret management
Team C: Configures own monitoring    → One database provisioning
(3x effort, 3 different standards)   (Consistent, self-service)
```

---

## 2. Golden Paths — Paved Roads for Developers

```
A Golden Path is a well-supported, recommended way to accomplish
a common task. Developers CAN deviate, but the golden path is easier.

Example Golden Paths:
┌────────────────────────────────────────────────────────────────────┐
│  Task                    │  Golden Path                            │
├──────────────────────────┼─────────────────────────────────────────┤
│ Create a new service     │ Service template (Cookiecutter/Yeoman)  │
│                          │ Pre-configured: CI/CD, Docker, health   │
│                          │ checks, logging, metrics, auth          │
├──────────────────────────┼─────────────────────────────────────────┤
│ Deploy to production     │ GitOps pipeline (push to main → auto   │
│                          │ deploy to staging → promote to prod)    │
├──────────────────────────┼─────────────────────────────────────────┤
│ Add a database           │ Self-service form → provisions managed │
│                          │ DB with backups, monitoring, secrets    │
├──────────────────────────┼─────────────────────────────────────────┤
│ Add observability        │ Import shared Grafana dashboard         │
│                          │ + pre-configured alerts                 │
├──────────────────────────┼─────────────────────────────────────────┤
│ Secret management        │ Azure Key Vault / HashiCorp Vault       │
│                          │ + auto-injected into pods               │
└──────────────────────────┴─────────────────────────────────────────┘
```

---

## 3. Internal Developer Portal (Backstage)

```
Backstage (by Spotify) = Central catalog for all services, APIs,
documentation, and infrastructure.

Features:
┌──────────────────────────────────────────────────────────────────────┐
│  Feature              │  What It Provides                           │
├───────────────────────┼─────────────────────────────────────────────┤
│ Service Catalog       │ List of ALL services with ownership, health,│
│                       │ dependencies, API docs                      │
├───────────────────────┼─────────────────────────────────────────────┤
│ Software Templates    │ Create new service from template in minutes │
│ (Scaffolder)          │ (pre-configured CI/CD, Docker, etc.)       │
├───────────────────────┼─────────────────────────────────────────────┤
│ TechDocs              │ Docs-as-code alongside the service code    │
│                       │ (Markdown → rendered in portal)             │
├───────────────────────┼─────────────────────────────────────────────┤
│ API Registry          │ Discover and browse all API definitions     │
│                       │ (OpenAPI, gRPC, GraphQL)                    │
├───────────────────────┼─────────────────────────────────────────────┤
│ Dependency Graph      │ Visualize service dependencies              │
│                       │ "Who calls whom?"                           │
├───────────────────────┼─────────────────────────────────────────────┤
│ Scorecards            │ Rate services on quality metrics            │
│                       │ (test coverage, docs, SLO compliance)      │
└───────────────────────┴─────────────────────────────────────────────┘
```

---
---

# Architecture Design — Master Checklist

This is a comprehensive checklist combining ALL topics covered above.

```
TRADE-OFFS & DECISIONS
  □ Top 3 quality attributes identified and prioritized
  □ Trade-off analysis documented (weighted matrix)
  □ Architecture Decision Records (ADRs) maintained
  □ "Good enough" principle applied (no over-engineering)

SECURITY
  □ Zero Trust principles applied
  □ Threat model completed (STRIDE + DREAD scoring)
  □ OAuth 2.0 + PKCE for authentication
  □ JWT with short expiry + refresh token rotation
  □ OWASP Top 10 mitigations in place
  □ Security headers configured
  □ Rate limiting on all public endpoints
  □ mTLS for service-to-service calls

PII & COMPLIANCE
  □ PII classification completed (high/medium/low)
  □ Encryption at rest (AES-256) and in transit (TLS 1.3)
  □ Tokenization for payment data
  □ PII-safe logging (auto-redaction)
  □ Right to erasure implemented (cross-service)
  □ Data retention policies automated
  □ Compliance framework requirements mapped to controls

BCP & RESILIENCE
  □ RPO and RTO defined per service
  □ Multi-AZ deployment (minimum)
  □ 3-2-1 backup rule followed
  □ Circuit breakers, retries, timeouts, bulkheads
  □ Health checks + Kubernetes probes
  □ Zero-downtime deployment strategy
  □ Incident response plan with runbooks
  □ Chaos engineering experiments (monthly)

OBSERVABILITY
  □ Structured logging with correlation IDs
  □ Metrics: RED (request) + USE (resource) + Business
  □ Distributed tracing (OpenTelemetry)
  □ SLIs, SLOs defined with error budgets
  □ Alerting on symptoms (not causes)

API DESIGN
  □ RESTful conventions followed
  □ Versioning strategy decided
  □ Idempotency keys for POST/PUT operations
  □ Standard error format (RFC 7807)
  □ Pagination (cursor-based for large datasets)
  □ API Gateway pattern for cross-cutting concerns

DATA ARCHITECTURE
  □ Database selection justified (SQL vs NoSQL vs polyglot)
  □ Sharding strategy defined (if needed)
  □ CQRS applied where read/write patterns diverge
  □ Data migration strategy (expand-contract preferred)

DOMAIN-DRIVEN DESIGN
  □ Bounded contexts identified
  □ Context map drawn
  □ Aggregates designed (small, one-per-transaction)
  □ Domain events for cross-aggregate communication

EVENT-DRIVEN ARCHITECTURE
  □ Message broker selected and justified
  □ Transactional outbox for reliable publishing
  □ Idempotent consumers (handle duplicates)
  □ Dead letter queue with alerting

MULTI-TENANCY
  □ Data isolation strategy chosen
  □ Tenant resolution strategy defined
  □ Global query filters for data isolation
  □ Tenant-specific configuration support

PLATFORM & OPERATIONS
  □ Golden paths for common developer tasks
  □ Service templates (new service in < 30 min)
  □ Internal developer portal (service catalog)
  □ 12-factor app principles followed
  □ Infrastructure as Code

COST & PERFORMANCE
  □ Performance budgets defined (p50, p95, p99)
  □ N+1 queries eliminated
  □ Capacity planning completed
  □ Cloud cost optimization (right-sizing, reserved instances)
  □ Cost-per-request tracked per service

EVOLUTION & DEBT
  □ Fitness functions in CI/CD pipeline
  □ Architecture runway 2-3 sprints ahead
  □ Technical debt tracked and prioritized
  □ 20% of sprint capacity for debt repayment
  □ Migration strategy defined (strangler fig preferred)
```
