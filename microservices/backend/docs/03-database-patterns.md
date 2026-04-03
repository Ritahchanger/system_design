# 03 — Database Patterns

## Table of Contents

- [Database per Service](#database-per-service)
- [Choosing the Right Database](#choosing-the-right-database)
- [Data Consistency in Distributed Systems](#data-consistency-in-distributed-systems)
- [The Saga Pattern](#the-saga-pattern)
- [CQRS — Command Query Responsibility Segregation](#cqrs)
- [Event Sourcing](#event-sourcing)
- [The Outbox Pattern](#the-outbox-pattern)
- [Shared Database Anti-pattern](#shared-database-anti-pattern)
- [Data Migration Strategy](#data-migration-strategy)
- [Summary & Next Steps](#summary--next-steps)

---

## Database per Service

The single most important data rule in microservices: **each service owns its own database and no other service touches it directly**.

```
✅  Order Service  →  orders_db   (MongoDB)
    User Service   →  users_db    (PostgreSQL)
    Payment Service → payments_db (PostgreSQL)

❌  Order Service  ──┐
                     ├──  shared_db  ← distributed monolith
    User Service   ──┘
```

This isn't about technology preference — it's about enforcing the service boundary at the data layer. A service that can directly read or write another service's database is coupled to that service's schema. Schema changes become coordinated deployments. The whole point of microservices collapses.

### How Services Access Each Other's Data

The only correct way is via the owning service's API.

```typescript
// Order Service needs user details — it calls the API
const user = await fetch(`http://user-service/api/v1/users/${userId}`)
  .then(r => r.json());

// Order Service caches what it needs locally to avoid chatty calls
await redis.setex(`user:${userId}`, 300, JSON.stringify({
  id: user.id,
  email: user.email,
  displayName: user.displayName,
}));
```

**What about joins?** You don't. Services that need to query across domains either:
1. Call each service API and join in application code (for small result sets)
2. Maintain a local read-model populated by events (for high-frequency queries)
3. Use the CQRS pattern to build a dedicated query projection (see below)

---

## Choosing the Right Database

Polyglot persistence — using the right database for each service's access patterns — is one of microservices' genuine advantages. Don't default to PostgreSQL everywhere out of habit.

| Database type | Best for | Examples |
|--------------|----------|---------|
| **Relational (SQL)** | Structured data, complex queries, ACID transactions, financial records | PostgreSQL, MySQL |
| **Document** | Flexible schemas, nested/hierarchical data, rapid iteration | MongoDB, DynamoDB |
| **Key-value** | Session data, caching, simple lookups by ID | Redis, DynamoDB |
| **Wide-column** | High write throughput, time-series, event logs, IoT | Cassandra, ScyllaDB |
| **Search** | Full-text search, faceted filtering, analytics | Elasticsearch, OpenSearch |
| **Graph** | Relationships between entities, recommendations, fraud | Neo4j, Amazon Neptune |
| **Time-series** | Metrics, monitoring data, financial ticks | InfluxDB, TimescaleDB |
| **Message/Event store** | Ordered event logs, event sourcing | Kafka, EventStoreDB |

### Decision Framework

Ask three questions before choosing:

1. **What is the primary access pattern?** By ID? By range? Full-text? Graph traversal?
2. **What consistency guarantees do you need?** Strong consistency (financial ledger) or eventual (analytics)?
3. **What are the scale characteristics?** Read-heavy? Write-heavy? Time-series?

```
User Service:
  Access: by ID, by email (unique), by auth token
  Consistency: strong (login must reflect latest state)
  Scale: moderate reads, low writes
  → PostgreSQL

Order Service:
  Access: by ID, by user, nested line items in one document
  Consistency: eventual ok for reporting, strong for checkout
  Scale: variable writes during sales peaks
  → MongoDB

Analytics Service:
  Access: aggregations over time windows, full-text on descriptions
  Consistency: eventual is fine (reports can lag seconds)
  Scale: high write throughput, complex read queries
  → Elasticsearch + ClickHouse
```

---

## Data Consistency in Distributed Systems

In a monolith, wrapping multiple operations in a single database transaction gives you ACID guarantees: either everything commits, or nothing does. In microservices, each service has its own database — cross-service operations cannot use a single ACID transaction.

### The Two Failure Modes

**Scenario**: Place an order (Order Service), charge the card (Payment Service), reserve stock (Inventory Service). What happens when step 2 succeeds but step 3 fails?

Without a coordination pattern, you have charged money for an order you cannot fulfil. This is a partial failure — the worst kind.

### Two-Phase Commit (2PC) — Why Not to Use It

2PC is the traditional distributed transaction protocol. It works but it's a terrible fit for microservices:

- Requires all services to implement a transaction coordinator protocol
- All participants must lock resources until the coordinator commits or rolls back
- If the coordinator fails mid-transaction, participants are locked indefinitely
- It creates tight runtime coupling — exactly what microservices are trying to avoid

**Use 2PC only** when you are using a single database with multiple schemas, or with databases that natively support distributed transactions (e.g. CockroachDB, Spanner). Not across independent service boundaries.

### Eventual Consistency

The correct default model for cross-service data. Accept that data will be consistent *eventually*, design around it:

- Each service executes its local transaction atomically
- State is propagated via events
- Downstream services update their local state when they process the event
- The system converges to a consistent state without locking anything globally

```
Order placed at T=0
  Order Service marks order as PENDING (T=0)
  Publishes OrderPlaced event

  Inventory Service receives event at T=10ms
    Reserves stock, publishes InventoryReserved

  Payment Service receives InventoryReserved at T=25ms
    Charges card, publishes PaymentProcessed

  Order Service receives PaymentProcessed at T=40ms
    Marks order as CONFIRMED
```

The system is consistent at T=40ms. No global locks. No single point of failure.

---

## The Saga Pattern

A saga is a sequence of local transactions where each step publishes an event (or sends a command) that triggers the next step. If any step fails, compensating transactions undo the work already done.

There are two implementation styles.

### Choreography-Based Saga

Services react to events independently. There is no central coordinator — each service knows what to do when it sees a specific event.

```javascript
// Order Service
async function placeOrder(orderData) {
  const order = await orderRepo.create({
    ...orderData,
    status: 'PENDING'
  });

  // Publish event — Inventory Service will pick this up
  await eventBus.publish('OrderPlaced', {
    orderId: order.id,
    items: order.items,
    userId: order.userId,
  });

  return order;
}

// Inventory Service (listening independently)
eventBus.subscribe('OrderPlaced', async (event) => {
  try {
    await inventoryRepo.reserveItems(event.orderId, event.items);
    await eventBus.publish('InventoryReserved', { orderId: event.orderId });
  } catch (err) {
    await eventBus.publish('InventoryReservationFailed', {
      orderId: event.orderId,
      reason: err.message,
    });
  }
});

// Order Service listens for failure and compensates
eventBus.subscribe('InventoryReservationFailed', async (event) => {
  await orderRepo.update(event.orderId, { status: 'FAILED' });
  await eventBus.publish('OrderFailed', { orderId: event.orderId });
});
```

**Pros**: Simple, loose coupling, no central point of failure.
**Cons**: Business logic is spread across services, hard to visualise the full workflow, debugging requires tracing events across multiple logs.

### Orchestration-Based Saga

A dedicated orchestrator service drives the workflow step by step. It tracks state and issues commands to each participant.

```javascript
class OrderSagaOrchestrator {
  async execute(orderData) {
    const sagaId = crypto.randomUUID();

    await this.sagaRepo.create({
      id: sagaId,
      status: 'STARTED',
      steps: ['RESERVE_INVENTORY', 'PROCESS_PAYMENT', 'CONFIRM_ORDER'],
      currentStep: 0,
    });

    try {
      // Step 1: Reserve inventory
      await this.inventoryService.reserveItems(sagaId, orderData.items);
      await this.sagaRepo.advanceStep(sagaId);

      // Step 2: Process payment
      await this.paymentService.charge(sagaId, orderData.payment);
      await this.sagaRepo.advanceStep(sagaId);

      // Step 3: Confirm order
      const order = await this.orderService.confirm(sagaId, orderData);
      await this.sagaRepo.complete(sagaId);

      return order;
    } catch (err) {
      await this.compensate(sagaId, err);
      throw new SagaFailedError(sagaId, err.message);
    }
  }

  async compensate(sagaId, err) {
    const saga = await this.sagaRepo.find(sagaId);

    // Roll back in reverse order of what completed
    if (saga.currentStep >= 2) {
      await this.paymentService.refund(sagaId);
    }
    if (saga.currentStep >= 1) {
      await this.inventoryService.releaseReservation(sagaId);
    }

    await this.sagaRepo.fail(sagaId, err.message);
  }
}
```

**Pros**: Explicit workflow, easier to monitor, compensations are centralised and visible.
**Cons**: Orchestrator is a dependency — if it fails, workflows stall. Risk of the orchestrator becoming a god service.

### Choosing Between Them

| Factor | Choreography | Orchestration |
|--------|-------------|---------------|
| Workflow complexity | Simple, linear | Complex, conditional branches |
| Number of steps | 2–4 | 5+ |
| Team ownership | Single team | Multiple teams |
| Observability needs | Low | High (regulatory, auditable) |
| Failure tolerance | Must be stateless | Orchestrator must be durable |

For most workflows, start with choreography. Introduce an orchestrator when you need conditional branching, timeouts, or audit trails.

---

## CQRS

Command Query Responsibility Segregation splits your data model into two: a write model that handles mutations, and one or more read models optimised for specific queries.

### The Core Problem CQRS Solves

In a standard service, the same data model serves both writes (commands) and reads (queries). This creates tension:

- Writes need normalised data to avoid anomalies
- Reads need denormalised data to avoid expensive joins
- Optimising for one degrades the other

CQRS separates them completely.

```
Write side:
  POST /orders  →  Command handler  →  Normalised write store
                                        (validates, enforces invariants)

Read side:
  GET /orders/{id}  →  Query handler  →  Denormalised read model
                                          (pre-joined, pre-aggregated, fast)
```

### Implementation

```typescript
// Command handler — write side
class PlaceOrderCommand {
  async execute(command: PlaceOrderDTO): Promise<string> {
    // Validate business rules against the write model
    const user = await this.userRepo.findById(command.userId);
    if (!user.isActive) throw new Error('User account is inactive');

    const inventory = await this.inventoryRepo.checkAvailability(command.items);
    if (!inventory.allAvailable) throw new Error('Items out of stock');

    // Write to the normalised store
    const order = await this.orderRepo.create(command);

    // Publish event to update read models
    await this.eventBus.publish('OrderCreated', order);

    return order.id;
  }
}

// Event handler — updates the read model asynchronously
eventBus.subscribe('OrderCreated', async (event) => {
  // Denormalised view — pre-join user data so queries are instant
  await readDb.upsert('order_summaries', {
    orderId: event.id,
    userId: event.userId,
    userEmail: event.userEmail,       // denormalised
    userDisplayName: event.userName,  // denormalised
    itemCount: event.items.length,
    totalAmount: event.totalAmount,
    status: event.status,
    createdAt: event.createdAt,
  });
});

// Query handler — read side, never touches the write store
class GetOrderSummaryQuery {
  async execute(orderId: string): Promise<OrderSummaryDTO> {
    // Single table scan, no joins, pre-computed aggregates
    return this.readDb.findOne('order_summaries', { orderId });
  }
}
```

### When to Use CQRS

CQRS adds complexity — don't apply it everywhere. Use it when:

- Read and write loads are significantly different in volume or scale
- You have multiple consumer-specific views of the same data (admin dashboard, mobile app, reporting)
- Your read queries are too slow because they require complex joins
- You need independent scaling of read vs write infrastructure

Do not use it for simple CRUD services where a single model works fine.

---

## Event Sourcing

Instead of storing the current state of an entity, store the sequence of events that led to that state. Current state is derived by replaying events.

```
Traditional (state storage):
  orders table: { id, status: "SHIPPED", amount: 99.99, updatedAt: "..." }
  — history is gone, only current state survives

Event sourcing (event storage):
  events table:
    { orderId, type: "OrderPlaced",   data: {...}, timestamp: T1 }
    { orderId, type: "PaymentTaken",  data: {...}, timestamp: T2 }
    { orderId, type: "OrderShipped",  data: {...}, timestamp: T3 }
  — full history preserved, current state = replay of all events
```

### Implementation

```typescript
class Order {
  id: string;
  status: 'PENDING' | 'PAID' | 'SHIPPED' | 'CANCELLED';
  amount: number;
  events: DomainEvent[] = [];

  // Apply an event — mutates state and records the event
  apply(event: DomainEvent): void {
    switch (event.type) {
      case 'OrderPlaced':
        this.status = 'PENDING';
        this.amount = event.data.amount;
        break;
      case 'PaymentTaken':
        this.status = 'PAID';
        break;
      case 'OrderShipped':
        this.status = 'SHIPPED';
        break;
      case 'OrderCancelled':
        this.status = 'CANCELLED';
        break;
    }
    this.events.push(event);
  }

  // Rebuild state from stored events (rehydration)
  static fromHistory(events: DomainEvent[]): Order {
    const order = new Order();
    events.forEach(e => order.apply(e));
    order.events = []; // clear — these are already stored
    return order;
  }
}

// Repository saves events, not state
class OrderEventSourcedRepository {
  async save(order: Order): Promise<void> {
    for (const event of order.events) {
      await this.eventStore.append(order.id, event);
    }
  }

  async findById(orderId: string): Promise<Order> {
    const events = await this.eventStore.getEvents(orderId);
    return Order.fromHistory(events);
  }
}
```

### Snapshots

Replaying thousands of events on every load is expensive. Use snapshots to cache the state at a point in time and only replay events since the snapshot.

```typescript
async findById(orderId: string): Promise<Order> {
  const snapshot = await this.snapshotStore.getLatest(orderId);

  const events = snapshot
    ? await this.eventStore.getEventsSince(orderId, snapshot.version)
    : await this.eventStore.getEvents(orderId);

  const order = snapshot
    ? Order.fromSnapshot(snapshot)
    : new Order();

  events.forEach(e => order.apply(e));
  return order;
}
```

### When to Use Event Sourcing

Event sourcing solves specific problems. Apply it when you genuinely have those problems, not as a default.

Use it when:
- You need a complete, immutable audit trail (financial transactions, healthcare, compliance)
- You need temporal queries ("what was the state of this order at 2pm on Tuesday?")
- You are combining with CQRS and want the event log to be the single source of truth for all projections
- You need to replay events to rebuild read models or fix projection bugs

Do not use it when:
- You just need CRUD with a simple history log (use a separate audit table)
- Your team is not familiar with it — the learning curve is steep and bugs are subtle
- Your entities have very high event rates — event stores under heavy write load need careful management

---

## The Outbox Pattern

The most common consistency bug in event-driven microservices: you write to your database and then publish an event, but the publish fails. Your database has the new state. The rest of the system never heard about it.

```
❌ Classic race condition:
  1. INSERT order INTO orders_db  ✓
  2. PUBLISH OrderPlaced to broker ✗  ← network hiccup
  Result: order exists, no downstream processing ever happens
```

The Outbox Pattern fixes this by making event publishing part of the database transaction.

```
✅ With outbox:
  1. BEGIN TRANSACTION
     INSERT order INTO orders           ✓
     INSERT event INTO outbox_events    ✓
  2. COMMIT TRANSACTION                 ✓
  3. Outbox worker polls outbox_events  ← separate process
     PUBLISH OrderPlaced to broker      ✓ (retried until success)
     DELETE or mark event as published  ✓
```

### Implementation

```sql
-- Schema
CREATE TABLE outbox_events (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  aggregate_id VARCHAR(255) NOT NULL,
  event_type   VARCHAR(255) NOT NULL,
  payload      JSONB        NOT NULL,
  created_at   TIMESTAMPTZ  DEFAULT NOW(),
  published_at TIMESTAMPTZ  -- NULL = not yet published
);
```

```typescript
// In your service — both writes happen in one transaction
async function placeOrder(orderData: PlaceOrderDTO): Promise<Order> {
  return this.db.transaction(async (tx) => {
    const order = await tx.orders.create(orderData);

    // Write the event to the outbox in the same transaction
    await tx.outboxEvents.create({
      aggregateId: order.id,
      eventType: 'OrderPlaced',
      payload: {
        orderId: order.id,
        userId: order.userId,
        items: order.items,
        totalAmount: order.totalAmount,
      },
    });

    return order;
  });
}

// Outbox worker — runs separately, polls for unpublished events
async function processOutbox(): Promise<void> {
  const unpublished = await db.outboxEvents.findMany({
    where: { publishedAt: null },
    orderBy: { createdAt: 'asc' },
    take: 100,
  });

  for (const event of unpublished) {
    await broker.publish(event.eventType, event.payload);
    await db.outboxEvents.update(event.id, {
      publishedAt: new Date(),
    });
  }
}

// Run every 500ms
setInterval(processOutbox, 500);
```

### Transactional Outbox via CDC

For high-throughput systems, instead of a polling worker, use **Change Data Capture (CDC)** — tools like Debezium read the database's transaction log (WAL for PostgreSQL) and stream changes directly to the broker. Zero polling lag, no extra database load.

```
PostgreSQL WAL → Debezium → Kafka → Consumer services
```

This is the production-grade approach for systems processing thousands of events per second.

---

## Shared Database Anti-pattern

For completeness — what you must never do, and what to do instead when you've inherited it.

### The Problem

```
❌  Service A  ──┐
                 ├── shared_db.orders
    Service B  ──┘

- Service A changes the orders schema
- Service B breaks silently
- Both services must be deployed together
- You've built a distributed monolith
```

### How to Migrate Away

If you inherit a shared database, don't try to split it in one go. Use the **Strangler Fig** at the data layer:

1. Identify which tables each service *logically* owns
2. Create a new schema/database for one service
3. Add a synchronisation layer (dual-writes + eventual consistency) during migration
4. Redirect the migrating service to its own database
5. Remove the synchronisation layer once traffic has fully migrated
6. Repeat for the next service

```
Phase 1: Both services use shared_db
Phase 2: Service A writes to both shared_db AND new service_a_db (dual-write)
         Service A reads from service_a_db
         Service B still reads from shared_db (unchanged)
Phase 3: Service B migrates to call Service A's API for the data it needs
Phase 4: Dual-write removed, shared_db tables for Service A are dropped
```

---

## Data Migration Strategy

Schema migrations in microservices need to be backward compatible. You cannot migrate and redeploy atomically — old and new versions of a service run simultaneously during deployment.

### Expand / Contract Pattern

The safest way to make breaking schema changes.

```
Step 1 — Expand (backward compatible):
  Add the new column as nullable
  ALTER TABLE users ADD COLUMN display_name VARCHAR(255);

Step 2 — Migrate data (backfill):
  UPDATE users SET display_name = first_name || ' ' || last_name
  WHERE display_name IS NULL;

Step 3 — Deploy new code:
  New code reads display_name, still writes both old and new columns

Step 4 — Contract (remove old column):
  Once all instances run new code, drop the old columns
  ALTER TABLE users DROP COLUMN first_name;
  ALTER TABLE users DROP COLUMN last_name;
```

Never do Step 1 and Step 4 in the same deployment. Always leave a deploy between expanding and contracting.

### Migration Tooling

```bash
# Node.js — use db-migrate, Flyway, or Prisma migrate
npx prisma migrate deploy          # Apply pending migrations
npx prisma migrate status          # Show migration history

# Go — use golang-migrate
migrate -path ./migrations -database $DATABASE_URL up

# Java — use Flyway or Liquibase
./mvnw flyway:migrate
```

Always run migrations as part of the deployment pipeline, never manually in production.

---

## Summary & Next Steps

Data is where microservices complexity concentrates. Database per service enforces the boundary. Eventual consistency is the default model — accept it and design around it rather than fighting it with distributed transactions. Sagas handle multi-service workflows. CQRS and event sourcing are power tools for specific problems — apply them surgically, not universally. The outbox pattern is non-negotiable if you're publishing events — use it from day one.

### Recommended Reading Order

| Step | Document | What you'll learn |
|------|----------|------------------|
| Next | [04-communication.md](./04-communication.md) | REST, gRPC, message queues — how services actually talk |
| Also | [08-resilience.md](./08-resilience.md) | Handling failures in service calls and event processing |
| Also | [07-observability.md](./07-observability.md) | Tracing distributed transactions across services |
| Also | [09-performance-optimization.md](./09-performance-optimization.md) | Caching strategies to reduce cross-service data fetching |

---

*Part of the [Microservices Architecture Guide](../../README.md)*  
*Previous: [02-service-architecture.md](./02-service-architecture.md)*  
*Next: [04-communication.md](./04-communication.md)*