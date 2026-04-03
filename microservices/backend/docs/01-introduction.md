# 01 — Introduction to Backend Microservices

## Table of Contents

- [What Are Microservices?](#what-are-microservices)
- [Monolith vs Microservices](#monolith-vs-microservices)
- [Core Principles](#core-principles)
- [When to Use Microservices](#when-to-use-microservices)
- [Trade-offs](#trade-offs)
- [Service Anatomy](#service-anatomy)
- [Communication Overview](#communication-overview)
- [The Evolution Path](#the-evolution-path)
- [Key Terminology](#key-terminology)
- [Summary & Next Steps](#summary--next-steps)

---

## What Are Microservices?

Microservices is an architectural style where a single application is composed of **small, independently deployable services**, each responsible for a specific business capability. Each service:

- Runs in its own process
- Communicates via well-defined APIs (REST, gRPC, or message queues)
- Owns its own data store
- Can be deployed, scaled, and updated independently

```
"Do one thing and do it well."
— Unix Philosophy, applied to services
```

Microservices emerged as a response to the pain of large, monolithic codebases — where a single change could require deploying the entire application, teams stepped on each other's work, and scaling meant scaling everything rather than just what needed it.

---

## Monolith vs Microservices

Understanding microservices starts with understanding what they replace.

### The Monolith

A monolithic architecture packages all application functionality — UI, business logic, and data access — into a single deployable unit.

```
┌─────────────────────────────────┐
│         Monolithic App          │
│  ┌──────────┐  ┌─────────────┐  │
│  │  User    │  │   Orders    │  │
│  │  Module  │  │   Module    │  │
│  └──────────┘  └─────────────┘  │
│  ┌──────────┐  ┌─────────────┐  │
│  │ Payment  │  │ Notification│  │
│  │  Module  │  │   Module    │  │
│  └──────────┘  └─────────────┘  │
│         Shared Database         │
└─────────────────────────────────┘
```

**Monolith strengths:**
- Simple to develop and deploy (one artifact)
- Easy to test end-to-end locally
- No network latency between modules
- Transactions are trivially consistent
- Lower operational overhead

**Monolith weaknesses:**
- Codebase grows unwieldy as teams scale
- Deploying one small fix requires redeploying everything
- A bug in one module can crash the whole application
- Scaling requires scaling everything, not just the bottleneck
- Technology choices are locked in for the whole application

### Microservices

Each business domain becomes its own independently deployed service.

```
┌──────────────┐   ┌──────────────┐
│  User Svc    │   │  Order Svc   │
│  (Node.js)   │   │    (Go)      │
│  PostgreSQL  │   │  MongoDB     │
└──────────────┘   └──────────────┘

┌──────────────┐   ┌──────────────┐
│  Payment Svc │   │  Notify Svc  │
│   (Java)     │   │  (Python)    │
│  PostgreSQL  │   │  RabbitMQ    │
└──────────────┘   └──────────────┘

         ↑ API Gateway ↑
              Client
```

**Microservices strengths:**
- Independent deployments — change one service without touching others
- Independent scaling — scale only the services under load
- Technology flexibility — each service can use the right tool for the job
- Fault isolation — a failing service doesn't bring everything down
- Team autonomy — teams own their services end to end
- Easier to understand — smaller, focused codebases

**Microservices weaknesses:**
- Distributed systems complexity (network failures, latency, partial failures)
- Data consistency is harder — no shared ACID transactions
- More infrastructure to manage (service discovery, load balancing, etc.)
- Observability requires more effort (distributed tracing, log aggregation)
- Testing is more complex (contract tests, integration across services)

---

## Core Principles

Microservices architectures that work well share a common set of principles. Violate them and the architecture fights back.

### 1. Single Responsibility

Each service owns exactly one bounded context. It does not leak logic or data into other services.

```
✅  User Service   → manages users, authentication, profiles
✅  Order Service  → manages orders, cart, fulfilment
❌  User Service   → also emails users about their orders (that's Notification's job)
```

### 2. Decentralised Data Management

Services do not share databases. Each service is the single source of truth for its own data. Other services request data via API — they do not join across databases.

```
✅  Order Service calls GET /users/{id} to get user info
❌  Order Service connects directly to the Users database
```

### 3. Smart Endpoints, Dumb Pipes

Business logic lives in the services, not in the infrastructure connecting them. HTTP, gRPC, and message brokers are transport — they carry messages; they do not process them.

### 4. Design for Failure

Every network call can fail. Every dependency can be slow. Services must handle failures gracefully: timeouts, retries, circuit breakers, and fallbacks are not optional.

### 5. Infrastructure Automation

Services are only viable at scale if deployment, monitoring, and rollbacks are automated. This is non-negotiable.

### 6. Evolutionary Design

Start simple. Extract services when the pain of not doing so outweighs the cost of the split. Don't pre-emptively decompose into microservices.

---

## When to Use Microservices

Microservices are not universally better. They are the right tool in specific contexts.

### Use Microservices When

| Condition | Why It Matters |
|-----------|---------------|
| Team > 15–20 engineers | Conway's Law — your architecture mirrors your org chart. Microservices let teams own boundaries. |
| Multiple distinct business domains | Natural seams reduce coupling. |
| Different scaling requirements per domain | Scale the Order Service during flash sales without scaling Notifications. |
| Polyglot requirements | One team is better in Go, another in Python — let them use what's right. |
| High release velocity | Multiple teams deploy independently without blocking each other. |
| Strict fault isolation requirements | Payment failures should never crash the product catalogue. |

### Stay With a Monolith When

| Condition | Why It Matters |
|-----------|---------------|
| Early-stage startup / MVP | You don't know your domain boundaries yet. Premature decomposition is expensive. |
| Team < 10 engineers | Microservices overhead (CI pipelines, service mesh, tracing) costs more than it saves. |
| Simple CRUD application | The operational complexity is not justified. |
| Limited DevOps capability | Without automation and observability, microservices become a liability. |
| Domain boundaries unclear | Incorrect service cuts are worse than a monolith — you get distributed coupling. |

### The Strangler Fig Pattern

For existing monoliths, the practical migration path is the **Strangler Fig**: incrementally extract services at the edges, routing specific functionality to new services while the monolith handles the rest. The monolith shrinks as the new services grow around it — never do a big-bang rewrite.

```
Phase 1: Monolith handles everything
Phase 2: Extract Auth → User Service (behind the API Gateway)
Phase 3: Extract Payments → Payment Service
Phase 4: Extract Notifications → Notification Service
Phase N: Monolith is gone
```

---

## Trade-offs

Every microservices decision involves trade-offs. Being explicit about them prevents nasty surprises.

### The CAP Theorem Reality

Distributed systems cannot simultaneously guarantee consistency, availability, and partition tolerance. In a microservices world you are always choosing between **CP** (consistent but potentially unavailable) and **AP** (available but potentially stale). Design for this explicitly.

### Operational Complexity Budget

| Monolith | Microservices |
|----------|--------------|
| 1 deployment pipeline | N deployment pipelines |
| Single log stream | Distributed log aggregation (ELK, Loki) |
| In-process calls (µs) | Network calls (ms) |
| ACID transactions | Sagas / eventual consistency |
| 1 runtime | N runtimes (Docker, Kubernetes) |
| Simple debugging | Distributed tracing (Jaeger, Zipkin) |

You are trading **simplicity** for **scalability and autonomy**. Only accept that trade when the returns justify it.

### The Two-Pizza Rule (Heuristic)

A healthy microservice team should be feedable with two pizzas (~6–8 people). If a service needs more than that to maintain, it may be too large and should be decomposed further. If a team maintains dozens of services, they may be too small and some should be merged.

---

## Service Anatomy

A well-structured microservice follows a consistent internal layout regardless of language.

```
user-service/
├── cmd/                  # Entrypoints (HTTP server, worker)
│   └── server/
│       └── main.go
├── internal/
│   ├── api/              # HTTP handlers, request/response DTOs
│   ├── domain/           # Business logic, entities, value objects
│   ├── repository/       # Data access layer (DB queries)
│   └── service/          # Orchestrates domain + repository
├── pkg/                  # Shared, importable packages
├── migrations/           # Database migrations
├── Dockerfile
├── docker-compose.yml    # Local development environment
└── README.md             # Service-specific documentation
```

### Every Service Exposes

| Endpoint | Purpose |
|----------|---------|
| `GET /health` | Liveness probe — is the process alive? |
| `GET /ready` | Readiness probe — is the service ready to take traffic? |
| `GET /metrics` | Prometheus-format metrics |
| `GET /api/v1/...` | Business API |

```javascript
// Minimum viable health endpoint (Node.js)
app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    service: 'user-service',
    version: process.env.SERVICE_VERSION,
    timestamp: new Date().toISOString(),
  });
});

// Readiness endpoint (checks downstream dependencies)
app.get('/ready', async (req, res) => {
  try {
    await db.query('SELECT 1');
    res.json({ status: 'ready' });
  } catch (err) {
    res.status(503).json({ status: 'not ready', reason: err.message });
  }
});
```

---

## Communication Overview

Services communicate in two fundamental modes. Choosing the right one for each interaction is one of the most important architectural decisions.

### Synchronous (Request / Response)

The caller waits for a response before proceeding.

```
Client ──── GET /users/123 ──►  User Service
       ◄─── 200 { user } ───
```

Use for:
- Queries where the client needs the result immediately
- Simple CRUD operations
- Low-latency user-facing interactions

Technologies: REST (HTTP/1.1), gRPC (HTTP/2), GraphQL

### Asynchronous (Event-Driven)

The producer publishes an event and does not wait for a consumer.

```
Order Service ──── OrderPlaced event ──►  Message Broker
                                               │
                          Notification Svc ◄───┘
                          Inventory Svc   ◄───┘
                          Analytics Svc  ◄───┘
```

Use for:
- Workflows that span multiple services (sagas)
- Operations where eventual consistency is acceptable
- Fan-out to multiple consumers
- Decoupling producer from consumer availability

Technologies: RabbitMQ, Apache Kafka, AWS SQS/SNS

> **Rule of thumb:** If the response matters *right now* to the caller, use sync. If not, use async.

---

## The Evolution Path

Most successful microservices architectures didn't start that way. They evolved.

```
Stage 1 — Monolith
  All modules in one deployable unit.
  Suitable for teams < 10, early product.

Stage 2 — Modular Monolith
  Clear internal module boundaries with no cross-module DB access.
  Prepares for future extraction without the operational cost.

Stage 3 — Macro-services (2–4 services)
  Extract the most distinct domain (e.g. Payments) first.
  Learn the operational mechanics before multiplying services.

Stage 4 — Microservices
  Full decomposition aligned to team ownership.
  Requires mature CI/CD, observability, and service mesh.
```

The worst mistake in microservices adoption is going from Stage 1 to Stage 4 in a single sprint. The second worst is never leaving Stage 1 when the team has grown past 20.

---

## Key Terminology

| Term | Definition |
|------|-----------|
| **Bounded Context** | A logical boundary within which a particular domain model applies and is consistent (from Domain-Driven Design) |
| **Service Mesh** | Infrastructure layer that handles service-to-service communication (e.g. Istio, Linkerd) |
| **API Gateway** | Single entry point for external clients — handles routing, auth, rate limiting |
| **Sidecar** | A helper container deployed alongside a service to handle cross-cutting concerns (logging, proxying) |
| **Circuit Breaker** | Pattern that stops calling a failing service temporarily to prevent cascading failures |
| **Saga** | Pattern for managing distributed transactions across multiple services using compensating actions |
| **Service Discovery** | Mechanism by which services locate each other's network address (Consul, Kubernetes DNS) |
| **Idempotency** | Property where repeating the same operation produces the same result — critical for safe retries |
| **Eventual Consistency** | A consistency model where, given no new updates, all replicas converge to the same value over time |
| **Polyglot Persistence** | Using different data storage technologies for different services based on their access patterns |

---

## Summary & Next Steps

Microservices offer genuine advantages for large-scale, multi-team systems: independent deployability, fault isolation, and the freedom to scale and evolve each part of the system on its own terms. They come at real cost: operational complexity, distributed systems challenges, and a higher floor for engineering maturity.

The most important question is not *"should we use microservices?"* but *"are we ready to operate them responsibly?"*

### Recommended Reading Order

| Step | Document | What You'll Learn |
|------|----------|------------------|
| 1 | [02-service-architecture.md](./02-service-architecture.md) | How to draw the right service boundaries |
| 2 | [03-database-patterns.md](./03-database-patterns.md) | Data management in a distributed system |
| 3 | [04-communication.md](./04-communication.md) | REST, gRPC, events — when and how |
| 4 | [08-resilience.md](./08-resilience.md) | Circuit breakers, retries, bulkheads |
| 5 | [07-observability.md](./07-observability.md) | Distributed tracing, metrics, logging |
| 6 | [05-deployment-strategies.md](./05-deployment-strategies.md) | Getting to production reliably |

---

*Part of the [Microservices Architecture Guide](../../README.md)*