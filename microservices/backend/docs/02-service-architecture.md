# 02 — Service Architecture

## Table of Contents

- [Service Decomposition Strategies](#service-decomposition-strategies)
- [Defining Service Boundaries](#defining-service-boundaries)
- [API Design Principles](#api-design-principles)
- [The API Gateway Pattern](#the-api-gateway-pattern)
- [Service Communication Overview](#service-communication-overview)
- [Service Templates and Structure](#service-templates-and-structure)
- [Versioning Strategy](#versioning-strategy)
- [Anti-patterns to Avoid](#anti-patterns-to-avoid)
- [Summary & Next Steps](#summary--next-steps)

---

## Service Decomposition Strategies

Decomposing a system into the *right* services is the most important — and hardest — architectural decision. Cut too coarse and you recreate the monolith. Cut too fine and you end up with distributed spaghetti. There are three reliable strategies.

### Strategy 1 — Decompose by Domain (Domain-Driven Design)

The most principled approach. Identify the **bounded contexts** in your domain: areas where a consistent domain model applies and terms mean the same thing without ambiguity.

```
Bounded Context: "Order"
  - In this context, "customer" means the person placing the order
  - "product" means an orderable SKU with a price and stock level
  - These definitions don't bleed into the Payment context

Bounded Context: "Payment"
  - "customer" here means a billing account with a payment method
  - "amount" here means a financial ledger entry
```

Each bounded context becomes a service candidate. The language boundary is your service boundary.

**How to find bounded contexts:**
1. Run Event Storming workshops — map domain events, commands, and actors on a wall
2. Look for places where the same word means different things to different teams
3. Follow Conway's Law — your org chart is already telling you where the boundaries are

**Read more**: [Domain-Driven Design — Eric Evans](https://www.domainlanguage.com/ddd/)

### Strategy 2 — Decompose by Business Capability

Identify the distinct things your business *does*, not what it *is*. Each capability maps to a service.

```
E-commerce capabilities:
  - Product catalogue management   → Catalogue Service
  - Inventory & fulfilment          → Fulfilment Service
  - Pricing & promotions            → Pricing Service
  - Customer recommendations        → Recommendation Service
  - Analytics & reporting           → Analytics Service
```

A business capability is stable — it doesn't change when the technology changes. It maps cleanly to a team that owns it end-to-end.

### Strategy 3 — Decompose by Subdomain

From DDD: classify every part of your domain by strategic value.

| Subdomain type | Description | Strategy |
|---------------|-------------|----------|
| **Core** | Your competitive advantage — what makes you different | Build in-house, invest heavily, own every line |
| **Supporting** | Necessary but not differentiating | Build in-house, keep lean |
| **Generic** | Commodity functionality (auth, payments, email) | Buy or use SaaS — don't reinvent |

```
Ride-hailing company:
  Core:       Matching algorithm, surge pricing, driver routing
  Supporting: Trip history, ratings, support ticketing
  Generic:    SMS notifications (Twilio), payments (Stripe), auth (Auth0)
```

Spending engineering effort on generic subdomains is waste. Use off-the-shelf solutions and integrate.

---

## Defining Service Boundaries

A well-drawn service boundary has these properties:

### High Cohesion

Everything inside the service belongs together — it serves a single business purpose, owns a single data model, and changes for a single reason.

```
✅  User Service: user registration, login, profile, preferences, account deletion
❌  User Service: user registration, login, profile + sends order confirmation emails
                  (order emails belong to Notification, triggered by Order events)
```

### Loose Coupling

Services depend on each other as little as possible. When they must communicate, they do so via stable, versioned APIs — never via shared databases, shared libraries with business logic, or tight synchronous chains.

```
✅  Order Service publishes OrderPlaced event → Notification Service consumes it
❌  Order Service directly calls Notification Service, which calls Template Service,
    which calls User Service to get the email — 4-service chain for one operation
```

### Data Ownership

Each service is the **single source of truth** for its own data. Other services request that data via API. They do not query the database directly.

```
✅  GET /api/users/{id}          → returns user data to callers
❌  SELECT * FROM users_db.users → other services bypass the API
```

If two services need to join data frequently, consider whether they should be one service.

### The "Two-Pizza Team" Test

A service should be manageable by a team that could be fed with two pizzas (~6–8 people). If a service has grown too large to be owned by a small team, it's time to decompose further.

### Avoiding Distributed Monolith

The worst outcome is a distributed monolith — microservices that are so tightly coupled they must be deployed together. Signs you're building one:

- Services share a database
- Deploying service A requires deploying service B
- A request touches 5+ services in a synchronous chain
- Services import each other's internal domain objects

---

## API Design Principles

Every service exposes a contract. Design it with the assumption that it will be consumed by teams you've never spoken to and that it will outlive the current implementation.

### RESTful Resource Design

Model APIs around resources (nouns), not actions (verbs).

```http
# Resources
GET    /api/v1/users              # list users
POST   /api/v1/users              # create user
GET    /api/v1/users/{id}         # get user
PUT    /api/v1/users/{id}         # replace user
PATCH  /api/v1/users/{id}         # partial update
DELETE /api/v1/users/{id}         # delete user

# Nested resources
GET    /api/v1/users/{id}/orders  # get orders for a user
```

### HTTP Status Codes — Use Them Correctly

| Code | Meaning | Use case |
|------|---------|----------|
| `200` | OK | Successful GET, PUT, PATCH |
| `201` | Created | Successful POST that creates a resource |
| `204` | No Content | Successful DELETE |
| `400` | Bad Request | Invalid input, validation failure |
| `401` | Unauthorized | Missing or invalid authentication |
| `403` | Forbidden | Authenticated but not permitted |
| `404` | Not Found | Resource does not exist |
| `409` | Conflict | Duplicate resource, optimistic lock conflict |
| `422` | Unprocessable Entity | Semantically invalid input |
| `429` | Too Many Requests | Rate limit exceeded |
| `500` | Internal Server Error | Unexpected server-side failure |

Never return `200` with an error in the body — that breaks every client that checks status codes.

### Consistent Error Response Schema

All errors across all services must follow the same shape. Define this contract organisation-wide and never deviate.

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "message": "Must be a valid email address"
      }
    ],
    "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

The `traceId` links the error back to a distributed trace in your observability stack — see [07-observability.md](./07-observability.md).

### Request / Response DTOs

Never expose your internal domain objects or database models directly via API. Use explicit Data Transfer Objects (DTOs) that you control independently.

```typescript
// Internal domain entity — never expose directly
interface UserEntity {
  id: string;
  hashedPassword: string;  // must never leave the service
  internalScore: number;   // internal implementation detail
  createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null;
}

// External DTO — the public contract
interface UserResponse {
  id: string;
  email: string;
  displayName: string;
  createdAt: string; // ISO 8601 string, not a Date object
}

// Mapper keeps them separate
function toUserResponse(entity: UserEntity): UserResponse {
  return {
    id: entity.id,
    email: entity.email,
    displayName: entity.displayName,
    createdAt: entity.createdAt.toISOString(),
  };
}
```

### Pagination

All list endpoints must paginate. Never return unbounded collections.

```http
GET /api/v1/orders?page=2&limit=20&sort=createdAt&order=desc

{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 847,
    "hasNext": true,
    "hasPrev": true
  }
}
```

For large datasets or real-time feeds, prefer cursor-based pagination:

```http
GET /api/v1/events?cursor=eyJpZCI6IjEyMyJ9&limit=50

{
  "data": [...],
  "nextCursor": "eyJpZCI6IjE3MyJ9",
  "hasMore": true
}
```

---

## The API Gateway Pattern

The API Gateway is the single entry point for all external clients. It handles cross-cutting concerns so that individual services don't have to.

```
External clients (web, mobile, third-party)
         |
    API Gateway
    ├── Authentication & JWT validation
    ├── Authorisation (RBAC/ABAC)
    ├── Rate limiting
    ├── Request routing
    ├── SSL termination
    ├── Request/response transformation
    ├── Logging & tracing injection
    └── Circuit breaking (optional)
         |
    ┌────┼────┐
 User   Order  Payment
  Svc    Svc    Svc
```

### What the Gateway Should NOT Do

The gateway handles infrastructure concerns only. It must not contain business logic.

```
✅  Gateway validates JWT signature and extracts user claims
✅  Gateway enforces rate limits per API key
✅  Gateway routes /api/v1/orders/* to the Order Service
❌  Gateway calculates order totals
❌  Gateway checks whether a user is eligible for a discount
❌  Gateway orchestrates calls across multiple services
```

Business logic that lives in the gateway is untestable, unversioned, and invisible to the service that owns it.

### BFF — Backend for Frontend

For complex client requirements (mobile vs web vs third-party), introduce a **Backend for Frontend (BFF)** — a thin gateway tailored to one client type.

```
Mobile App  ──►  Mobile BFF  ──►  Core Services
Web App     ──►  Web BFF     ──►  Core Services
Partners    ──►  Public API  ──►  Core Services
```

Each BFF aggregates and shapes responses for its specific client without polluting the core services with client-specific logic.

---

## Service Communication Overview

Services communicate in two modes. The choice between them is one of the most consequential decisions in your architecture.

### Synchronous (REST / gRPC)

```
Client ──── POST /orders ──►  Order Service
                              │
                          GET /users/{id}
                              │
                          ◄── User Service
                              │
            ◄── 201 Created ──┘
```

Use when:
- The caller needs the response before it can continue
- The operation is a query (reads don't need to be async)
- Latency matters for the user experience

Technology choices:
- **REST/HTTP** — universal, human-readable, easy to debug, use for external APIs
- **gRPC** — binary protocol, strongly typed via Protobuf, ~7–10x faster than REST, use for internal service-to-service calls with high throughput

### Asynchronous (Event-Driven)

```
Order Service ──── OrderPlaced ──►  Message Broker
                                     │
                          ┌──────────┼──────────┐
                    Inventory     Notification  Analytics
                      Svc            Svc          Svc
```

Use when:
- The producer does not need an immediate response
- Multiple services need to react to the same event
- You need to decouple services in time (the consumer can be offline)
- You want to avoid synchronous call chains

**Deep dive**: [04-communication.md](./04-communication.md)

### The Golden Rule

> If removing the dependency between two services would make either one simpler, you've drawn the boundary in the wrong place.

---

## Service Templates and Structure

Every service in your organisation should follow the same internal structure. This is not about micromanaging — it's about making every service instantly navigable for any engineer.

### Recommended Directory Structure

```
{service-name}/
├── cmd/
│   └── server/
│       └── main.go              # Entrypoint — wires dependencies, starts server
├── internal/
│   ├── api/
│   │   ├── handlers/            # HTTP handlers — thin, delegate to service layer
│   │   ├── middleware/          # Auth, logging, tracing, rate limiting
│   │   └── dto/                 # Request and response types
│   ├── domain/
│   │   ├── entities/            # Domain objects — no framework dependencies
│   │   ├── events/              # Domain event definitions
│   │   └── errors/              # Domain-specific error types
│   ├── service/
│   │   └── user_service.go      # Business logic — orchestrates domain + repo
│   └── repository/
│       └── user_repository.go   # Data access — SQL queries, cache reads/writes
├── pkg/
│   └── validator/               # Shared, importable utilities
├── migrations/
│   ├── 001_create_users.up.sql
│   └── 001_create_users.down.sql
├── config/
│   └── config.go                # Environment variable parsing
├── Dockerfile
├── docker-compose.yml           # Local dev: DB, cache, broker
├── .env.example
└── README.md
```

### Dependency Flow

Dependencies flow inward only. Inner layers know nothing about outer layers.

```
HTTP Handler  →  Service Layer  →  Repository  →  Database
     ↑               ↑                ↑
  (thin)         (business          (data
  delegates       logic)            access)
  to service
```

The domain layer has zero external dependencies — no frameworks, no ORMs, no HTTP clients. This makes it trivially unit-testable.

### Every Service Must Expose

```go
// Liveness — is the process alive?
GET /health
→ { "status": "healthy", "service": "user-service", "version": "1.4.2" }

// Readiness — is the service ready to accept traffic?
GET /ready
→ { "status": "ready" }         // checks DB, cache, broker connections
→ { "status": "not ready" }     // returns 503 — k8s will stop sending traffic

// Metrics — Prometheus format
GET /metrics
→ # HELP http_requests_total The total number of HTTP requests
  # TYPE http_requests_total counter
  http_requests_total{method="GET",code="200"} 1234
```

The readiness probe is your safety valve during deployments — see [05-deployment-strategies.md](./05-deployment-strategies.md).

---

## Versioning Strategy

APIs must evolve without breaking existing consumers. The discipline of versioning is what makes independent deployments possible.

### URL Versioning (Recommended)

```
/api/v1/users/{id}   ← stable, unchanged
/api/v2/users/{id}   ← new version with breaking changes
```

Both versions run concurrently until v1 is deprecated and all consumers have migrated.

### What Constitutes a Breaking Change

| Change | Breaking? |
|--------|-----------|
| Adding a new optional field to a response | No |
| Adding a new optional query parameter | No |
| Removing a field from a response | **Yes** |
| Renaming a field | **Yes** |
| Changing a field's type | **Yes** |
| Changing a required field to optional | No |
| Changing an optional field to required | **Yes** |
| Adding a new endpoint | No |
| Removing an endpoint | **Yes** |

### Deprecation Policy

1. Announce deprecation with a timeline (minimum 90 days for internal, 12 months for public APIs)
2. Add `Deprecation` and `Sunset` response headers to all v1 responses
3. Log all v1 calls to measure consumer migration progress
4. Remove only when traffic drops to zero or deadline passes

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 31 Dec 2024 23:59:59 GMT
Link: </api/v2/users/123>; rel="successor-version"
```

---

## Anti-patterns to Avoid

These patterns seem reasonable but consistently cause pain at scale.

### Chatty Services

A single user action triggering a chain of 8 synchronous service calls. Each hop adds latency and a new failure point.

```
Client → Gateway → Order Svc → User Svc → Address Svc
                             → Inventory Svc → Warehouse Svc
                             → Payment Svc → Card Svc
                             → Notification Svc
```

**Fix**: Denormalise data where it's needed. Have services publish events that others consume and cache locally. Use async patterns — [04-communication.md](./04-communication.md).

### Shared Database

Two services reading and writing the same database tables.

```
❌  User Service    ──┐
                      ├── shared_db.users table
   Profile Service ──┘
```

**Fix**: One database per service. If Profile needs user data, call `GET /api/v1/users/{id}`.

### God Service

A service that knows about every other service and orchestrates all business workflows. This is a distributed monolith in disguise.

```
❌  Orchestrator Service
    ├── creates user (User Svc)
    ├── creates order (Order Svc)
    ├── processes payment (Payment Svc)
    ├── updates inventory (Inventory Svc)
    └── sends notification (Notification Svc)
```

**Fix**: Use choreography-based sagas where services react to events independently. Reserve orchestration for genuinely complex, long-running workflows — [03-database-patterns.md](./03-database-patterns.md#saga-pattern).

### Leaking Domain Logic Into the Gateway

Business rules that live in the API Gateway become invisible to the service that should own them, impossible to test, and impossible to version.

**Fix**: The gateway routes and authenticates. Services contain all business logic.

### Synchronous Everything

Treating async operations (sending an email, updating analytics, refreshing a cache) as synchronous calls. This makes the critical path brittle and slow.

**Fix**: Any operation that doesn't need an immediate response should be async — [04-communication.md](./04-communication.md).

---

## Summary & Next Steps

Service architecture is about drawing the right lines. Cut by domain, keep data ownership strict, design APIs as public contracts, and keep the gateway as infrastructure — not as a business logic home. The patterns here lay the foundation; the database and communication patterns in the next two documents are where the real distributed systems complexity lives.

### Recommended Reading Order

| Step | Document | What you'll learn |
|------|----------|------------------|
| Next | [03-database-patterns.md](./03-database-patterns.md) | Database per service, CQRS, sagas, event sourcing |
| Then | [04-communication.md](./04-communication.md) | REST vs gRPC vs messaging — patterns and trade-offs |
| Also | [06-security.md](./06-security.md) | Securing inter-service communication with mTLS and JWT |
| Also | [08-resilience.md](./08-resilience.md) | Circuit breakers, retries, bulkheads for service calls |

---

*Part of the [Microservices Architecture Guide](../../README.md)*  
*Previous: [01-introduction.md](./01-introduction.md)*  
*Next: [03-database-patterns.md](./03-database-patterns.md)*