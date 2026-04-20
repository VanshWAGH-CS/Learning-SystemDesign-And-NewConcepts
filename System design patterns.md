# System Design Patterns — Complete Study Guide

> Based on key patterns every backend/software engineer must know.  
> Save this for revision or interview prep.

---

## Table of Contents

1. [Microservices vs Monolith](#1-microservices-vs-monolith)
2. [Database Per Service](#2-database-per-service)
3. [Circuit Breaker Pattern](#3-circuit-breaker-pattern)
4. [Event Sourcing](#4-event-sourcing)
5. [CQRS — Command Query Responsibility Segregation](#5-cqrs--command-query-responsibility-segregation)
6. [Pattern Relationship Map](#6-pattern-relationship-map)

---

## 1. Microservices vs Monolith

### Problem Statement
When building a system, the very first decision you make is: **should I go with a monolith or break everything into microservices?**

---

### Monolithic Architecture

All features live inside a **single server and codebase**.

![Monolith vs Microservices](https://microservices.io/i/DecomposingApplications.011.jpg)
*Source: [microservices.io](https://microservices.io/patterns/microservices.html)*

**How it works:**
- One server has all routes: auth routes, notification handlers, comment routes, post routes — all inside the same codebase.
- You can still run multiple instances (horizontal scaling), but you cannot scale individual features.
- If the notification module crashes, the **entire server goes down**.

**✅ Pros:**
- Simple to develop, test, and deploy
- No inter-service network calls — everything is in-process
- Easy to manage for small teams

**❌ Cons:**
- One bug in any module can crash the whole server
- Cannot scale just one feature — you scale all or nothing
- Becomes harder to maintain as the codebase grows

---

### Microservices Architecture

Every feature is its own **independent, isolated service**.

![Microservices Architecture](https://microservices.io/i/Microservice_Architecture.png)
*Source: [microservices.io](https://microservices.io/patterns/microservices.html)*

**How it works:**
- Auth service (Node.js), Notification service (Python), Post service (Rust) — all independent.
- Services communicate via REST APIs, gRPC, or message queues.
- If Notification service is down, Auth service and Post service **continue working**.

**✅ Pros:**
- Independent deployment, scaling, and failure isolation
- Each team can use the best language/tech stack for their service
- Scale only the services under heavy load

**❌ Cons:**
- Managing many servers is complex
- Inter-service communication must be carefully designed
- Higher infrastructure cost

---

### Decision Table

| Factor | Monolith | Microservices |
|---|---|---|
| Team size | Small | Large / multiple teams |
| Speed to prototype | Faster | Slower initial setup |
| Fault tolerance | Low | High |
| Scalability | All-or-nothing | Per-service |
| Complexity | Low | High |

<img width="746" height="545" alt="image" src="https://github.com/user-attachments/assets/8dabd404-0943-4e0e-a170-fbeafa140b71" />

---



## 2. Database Per Service

### Problem Statement
Once you pick microservices, the next question is: **should all services share one database, or should each service own its own?**

---

### Shared Database (Anti-pattern)

```
Auth Service ──┐
               ├──▶  SHARED PostgreSQL DB
Notif. Service─┤       ├── users table
               │       ├── notifications table
File Service ──┘       └── uploads table
```

**Problems:**
- A bug in File Service can accidentally **delete users** from the shared DB.
- The database becomes a **bottleneck** — single point of failure.
- You lose the benefit of true service isolation.

---

### Database Per Service (Recommended)

![Database per service pattern](https://microservices.io/i/databaseperservice.png)
*Source: [microservices.io](https://microservices.io/patterns/data/database-per-service.html)*

**How it works:**
- Auth Service → **MongoDB** (flexible schema for evolving user data)
- Notification Service → **ClickHouse** (analytics, open rates, click rates)
- Post Service → **PostgreSQL** (structured relational data for posts)

Each service picks the **best database for its use case**.

**✅ Pros:**
- True isolation — one service's bugs cannot corrupt another's data
- Freedom to choose any DB type per service
- Independent scaling and indexing per service

**❌ Cons:**
- **No DB-level JOINs** across services
- Joins become **application-level**: call the other service via HTTP → receive data → merge in code
- Higher infrastructure cost (multiple DB instances)

---


### Application-Level Join (How Cross-Service Data Works)

```
Post Service needs user details:

  Post Service ──HTTP GET /users/{id}──▶ Auth Service
                                               │
                                        Returns user data
                                               │
  Post Service ◀───────────────────────────────┘
       │
  Merges post data + user data in application code
       │
  Returns combined response to client
```
<img width="790" height="627" alt="image" src="https://github.com/user-attachments/assets/3e97ea0f-32e1-4449-8bb2-d6c843329b73" />

---



## 3. Circuit Breaker Pattern

### Problem Statement
In microservices, services depend on each other. **What happens when one service goes down and others keep calling it?**

---

### Cascading Failure (Without Circuit Breaker)

```
Service A ──▶ Service B ──▶ Service C ──▶ Service D ❌ (DOWN)
                                                │
                                          Returns error
                                                │
                                  Service C crashes ❌
                                         │
                               Service B crashes ❌
                                        │
                              Service A crashes ❌
```

One service failure brings the **entire system down**. This is called a **Cascading Failure**.

---

### Circuit Breaker Pattern

![Circuit Breaker Pattern diagram](https://learn.microsoft.com/en-us/azure/architecture/patterns/_images/circuit-breaker-diagram.png)
*Source: [Microsoft Azure Architecture Docs](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)*

A **Circuit Breaker** is a proxy placed between services that monitors requests and blocks calls to a failing service.

---

### Three States

```
          CLOSED ──(failures exceed threshold)──▶ OPEN
            ▲                                       │
            │                                    (timeout)
            │                                       ▼
    (test succeeds)                           HALF-OPEN
            └──────────────────────────────────────┘
```

| State | Behaviour |
|---|---|
| **CLOSED** | All requests flow normally. Service is healthy. |
| **OPEN** | All requests blocked immediately. Returns a fallback response. Gives the failing service time to recover. |
| **HALF-OPEN** | After a timeout, one test request is sent through. If it succeeds → CLOSED. If it fails → back to OPEN. |

**Why it matters:**
- Without circuit breaker: failing service gets hammered with retries → harder to recover.
- With circuit breaker: failing service gets breathing room → recovers faster, dependent services degrade gracefully.

> **Popular Libraries:** Netflix Hystrix · Resilience4j (Java) · `opossum` (Node.js) · Polly (.NET)
<img width="793" height="662" alt="image" src="https://github.com/user-attachments/assets/e442affd-5a16-4772-ad73-b5bf4c7782da" />


---


## 4. Event Sourcing

### Problem Statement
In high-throughput systems (banking, e-commerce), **how do you track state changes reliably without database locks becoming a bottleneck?**

---

### Traditional Approach — Mutable State

```
orders table:
┌──────────┬──────────────────┐
│ order_id │ status           │
├──────────┼──────────────────┤
│  #1234   │ DELIVERED        │  ← state mutated 4 times, history lost
└──────────┴──────────────────┘
```

**Problems:**
- You lose history — you can only see the current state.
- Concurrent writes require **locks**, which become a bottleneck at scale.

---

### Event Sourcing — Immutable Event Log

![Event Sourcing diagram](https://martinfowler.com/eaaDev/eventSourcing/accountBalance.gif)
*Source: [Martin Fowler — Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)*

**How it works:**
- Instead of mutating a state column, you **append events** to an immutable log.
- Events are **never deleted or modified**.
- Current state is derived by **replaying** the events.

```
Event Log (immutable, append-only):
┌────────────┬───────────────────┐
│ 12:00      │ ORDER_PLACED      │
│ 12:10      │ READY_TO_SHIP     │
│ 12:30      │ SHIPPING          │
│ 14:00      │ DELIVERED         │
└────────────┴───────────────────┘
```

To get current status → read the latest event → return "DELIVERED".

---

### Real-World Example: Banking

Banks **never** store just your balance as a single number. They maintain a complete transaction log:

```
+$1000  (initial deposit)
-$200   (Amazon purchase)
+$500   (salary credit)
-$100   (utility bill)
──────────────────────────
= $1200  (computed on the fly by replaying events)
```

This is Event Sourcing in the real world.

---

### Benefits

| Benefit | Why It Matters |
|---|---|
| **Full audit trail** | Every change recorded — great for compliance |
| **No locks needed** | Append-only writes avoid lock contention |
| **Time travel** | Replay events to see system state at any past point |
| **Debugging** | Reproduce bugs by replaying exact event sequence |
| **High throughput** | Appending is extremely fast |

<img width="767" height="621" alt="image" src="https://github.com/user-attachments/assets/7b6a112f-0afc-4dfc-a336-a9f4968960b7" />


---

## 5. CQRS — Command Query Responsibility Segregation

### Problem Statement
At massive scale (Amazon, Flipkart), a single database handles both reads and writes. **Can we separate these concerns to scale each side independently?**

---

### The Core Idea

> Split your system into two sides:
> - **Command Side** → handles writes (Create, Update, Delete)
> - **Query Side** → handles reads (GET, Search, List)

![CQRS Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/_images/command-and-query-responsibility-segregation-cqrs-basic.png)
*Source: [Microsoft Azure Architecture Docs](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)*

---

### How It Works

```
                     CLIENT
                    /       \
          Command side       Query side
          (writes/mutations) (reads)
                |                 |
         Event store          Read DB
         (write DB)           (optimised, denormalised)
                |
         Event projector
          (async sync)
                |
            Read DB ──────────────▶ Client gets fast reads
```

**Step-by-step:**
1. User places an order → request goes to **Command Side**
2. Command validates and appends an event to the **Event Store**
3. **Event Projector** (async) reads that event and updates the **Read DB** in a denormalised, query-optimised format
4. When user queries their order → goes directly to the **Query Side** → reads from Read DB instantly

---

### CQRS + Event Sourcing Together

These two patterns pair naturally:

```
Write path:  Command ──▶ Validate ──▶ Append to Event Log ──▶ Done ✅

Read path:   Event Log ──▶ Projector ──▶ Read DB (pre-computed) ──▶ Query returns instantly ✅
```

---

### Trade-offs

| Aspect | Detail |
|---|---|
| **Consistency** | **Eventual** — reads may lag slightly behind writes |
| **Read performance** | Very fast — data is pre-computed and ready |
| **Write performance** | Independent — write side scales separately |
| **Complexity** | Higher — two databases, event pipeline, projection logic |

> ⚠️ **Use CQRS only if eventual consistency is acceptable for your system.**

<img width="806" height="606" alt="image" src="https://github.com/user-attachments/assets/38a6fbe8-17da-418e-9914-177991fa6a4e" />


---

## 6. Pattern Relationship Map

Each pattern builds on the previous decision:

```
┌──────────────────────────────────────────────────────┐
│  Step 1: Monolith or Microservices?                  │
│  → Based on team size, scale needs, fault tolerance  │
└──────────────────────┬───────────────────────────────┘
                       │ (chose microservices)
┌──────────────────────▼───────────────────────────────┐
│  Step 2: Database per service?                       │
│  → True isolation? Or shared DB to save cost?        │
└──────────────────────┬───────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────┐
│  Step 3: Circuit Breaker                             │
│  → Prevent cascading failures between services       │
└──────────────────────┬───────────────────────────────┘
                       │ (high throughput system)
┌──────────────────────▼───────────────────────────────┐
│  Step 4: Event Sourcing                              │
│  → Immutable append-only log as source of truth      │
└──────────────────────┬───────────────────────────────┘
                       │ (read/write imbalance)
┌──────────────────────▼───────────────────────────────┐
│  Step 5: CQRS                                        │
│  → Separate read and write models for max scale      │
└──────────────────────────────────────────────────────┘
```

---

## Quick Revision Table

| Pattern | Solves | Key Trade-off |
|---|---|---|
| **Microservices** | Feature isolation, independent scaling | Inter-service complexity |
| **Monolith** | Simple development and deployment | Single point of failure |
| **DB Per Service** | True data isolation, DB flexibility | No DB-level JOINs; higher cost |
| **Circuit Breaker** | Cascading failures, system resilience | Threshold tuning, added proxy layer |
| **Event Sourcing** | Audit trail, lock-free high-throughput writes | Storage growth, complex replay |
| **CQRS** | Independent read/write scaling | Eventual consistency, added complexity |

<img width="704" height="441" alt="image" src="https://github.com/user-attachments/assets/46eb2ee3-caca-4d2c-accb-34587b5a4477" />

---

## References & Further Reading

- [Microservices Patterns — microservices.io](https://microservices.io/patterns/microservices.html)
- [Database Per Service — microservices.io](https://microservices.io/patterns/data/database-per-service.html)
- [Circuit Breaker — Microsoft Azure Docs](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)
- [Event Sourcing — Martin Fowler](https://martinfowler.com/eaaDev/EventSourcing.html)
- [CQRS — Microsoft Azure Docs](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [CQRS — Martin Fowler](https://martinfowler.com/bliki/CQRS.html)



---

> 💡 **Interview Tip:** Always mention trade-offs when discussing patterns.  
> Every pattern solves a problem but introduces new ones — that's the essence of system design.

*Happy Revising! 🚀*
