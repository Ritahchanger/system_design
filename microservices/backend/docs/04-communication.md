# 04 — Communication

## Table of Contents

- [The Two Communication Modes](#the-two-communication-modes)
- [REST over HTTP](#rest-over-http)
- [gRPC](#grpc)
- [REST vs gRPC — Decision Guide](#rest-vs-grpc-decision-guide)
- [Asynchronous Messaging](#asynchronous-messaging)
- [Message Brokers](#message-brokers)
- [Kafka vs RabbitMQ](#kafka-vs-rabbitmq)
- [Event-Driven Patterns](#event-driven-patterns)
- [Service Mesh](#service-mesh)
- [Communication Resilience](#communication-resilience)
- [Choosing the Right Pattern](#choosing-the-right-pattern)
- [Summary & Next Steps](#summary--next-steps)

---

## The Two Communication Modes

Every inter-service call falls into one of two modes. Choosing the wrong one for a given interaction is one of the most common sources of both performance problems and architectural coupling.

### Synchronous

The caller sends a request and blocks until it receives a response. Both services must be available at the same time.

```
Client ──── GET /users/123 ──►  User Service
       ◄─── 200 { user } ────
```

Characteristics:
- Immediate, consistent response
- Caller is coupled to the availability of the callee
- Timeout required — the caller must decide how long to wait
- Failure propagates: if User Service is down, the caller fails too

Use for:
- Queries where the result is needed to proceed
- User-facing operations where latency is visible
- Simple CRUD that needs synchronous confirmation

### Asynchronous

The producer publishes a message and continues immediately. The consumer processes it independently, possibly much later.

```
Order Service ──── OrderPlaced ──►  Broker  ──►  Notification Service
                                              ──►  Inventory Service
                                              ──►  Analytics Service
```

Characteristics:
- Producer and consumer are decoupled in time
- Multiple consumers can independently react to the same event
- Higher throughput — no blocking waits
- Trade-off: eventual consistency, harder to debug, no immediate confirmation

Use for:
- Workflows that span multiple services
- Fan-out to multiple consumers
- Operations where eventual consistency is acceptable
- Buffering spikes in load

> **The decision rule**: if the caller needs the answer right now, use sync. If it doesn't, use async.

---

## REST over HTTP

REST (Representational State Transfer) over HTTP is the universal baseline for service communication. Its strengths are ubiquity, human-readability, and tooling support — not performance.

### Anatomy of a Good REST Call

```http
POST /api/v1/orders HTTP/1.1
Host: order-service
Content-Type: application/json
Authorization: Bearer eyJhbGci...
X-Request-ID: 4bf92f35-77b3-4da6
X-Correlation-ID: a3ce929d-0e0e-4736

{
  "userId": "usr_01HQTM7K",
  "items": [
    { "productId": "prod_9K2X", "quantity": 2 }
  ],
  "shippingAddressId": "addr_3P9W"
}
```

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/v1/orders/ord_01HQTM8Z
X-Request-ID: 4bf92f35-77b3-4da6

{
  "id": "ord_01HQTM8Z",
  "status": "PENDING",
  "createdAt": "2024-01-15T10:30:00Z"
}
```

Key headers every service should send and respect:

| Header | Purpose |
|--------|---------|
| `X-Request-ID` | Unique ID for this specific HTTP request — for deduplication |
| `X-Correlation-ID` | ID that follows a business transaction across all services — for tracing |
| `Authorization` | JWT or service token |
| `Content-Type` | Always `application/json` for API responses |
| `Retry-After` | Seconds to wait before retrying (on 429 and 503) |
| `Idempotency-Key` | Client-supplied key for safe retries of POST/PATCH |

### Idempotency

Any request that mutates state must be safe to retry. Network failures happen. Clients retry. Without idempotency, retries cause duplicate operations — double charges, duplicate orders.

```typescript
// Client: attach an idempotency key to every mutating request
const response = await fetch('/api/v1/payments', {
  method: 'POST',
  headers: {
    'Idempotency-Key': crypto.randomUUID(), // generate once, retry with same key
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ orderId, amount }),
});

// Server: check if this key was already processed
async function processPayment(req: Request): Promise<Response> {
  const idempotencyKey = req.headers.get('Idempotency-Key');

  if (idempotencyKey) {
    const cached = await redis.get(`idempotency:${idempotencyKey}`);
    if (cached) {
      // Return the original response — don't process again
      return Response.json(JSON.parse(cached), { status: 200 });
    }
  }

  const result = await paymentService.charge(req.body);

  if (idempotencyKey) {
    // Cache result for 24 hours
    await redis.setex(`idempotency:${idempotencyKey}`, 86400, JSON.stringify(result));
  }

  return Response.json(result, { status: 201 });
}
```

### Timeouts — Always Set Them

Every outbound HTTP call must have an explicit timeout. The default is often "wait forever", which starves connection pools and causes cascading failures.

```typescript
// Node.js with fetch — set timeouts explicitly
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 3000); // 3 second timeout

try {
  const response = await fetch('http://user-service/api/v1/users/123', {
    signal: controller.signal,
  });
  return await response.json();
} catch (err) {
  if (err.name === 'AbortError') {
    throw new ServiceTimeoutError('user-service timed out after 3000ms');
  }
  throw err;
} finally {
  clearTimeout(timeoutId);
}
```

A reasonable starting point for internal service calls:
- **Connection timeout**: 500ms (how long to wait for a TCP connection)
- **Read timeout**: 2–5s (how long to wait for a response body)
- **Total timeout**: 5–10s (hard ceiling for the entire request)

Tune based on your P99 latency measurements in production — see [07-observability.md](./07-observability.md).

---

## gRPC

gRPC is a high-performance RPC framework from Google built on HTTP/2 and Protocol Buffers (Protobuf). It's the right choice for high-throughput internal service-to-service communication where REST overhead is measurable.

### Why gRPC is Faster

| Factor | REST/JSON | gRPC/Protobuf |
|--------|-----------|--------------|
| Serialisation | Text-based JSON | Binary Protobuf (~5-10x smaller) |
| Protocol | HTTP/1.1 (one request per connection) | HTTP/2 (multiplexed, many requests per connection) |
| Code generation | Manual or OpenAPI | Auto-generated from `.proto` — type-safe |
| Contract enforcement | Optional (schemas rarely enforced) | Enforced at compile time |

### Defining a Service Contract

Everything in gRPC starts with a `.proto` file. This file is the contract between client and server — both generate their code from it.

```protobuf
// user.proto
syntax = "proto3";

package user.v1;

service UserService {
  // Unary — one request, one response
  rpc GetUser(GetUserRequest) returns (GetUserResponse);

  // Server-side streaming — one request, stream of responses
  rpc ListUserOrders(ListOrdersRequest) returns (stream Order);

  // Client-side streaming — stream of requests, one response
  rpc BatchCreateUsers(stream CreateUserRequest) returns (BatchCreateResponse);

  // Bidirectional streaming — stream in, stream out
  rpc SyncUserActivity(stream ActivityEvent) returns (stream ActivityAck);
}

message GetUserRequest {
  string user_id = 1;
}

message GetUserResponse {
  string id        = 1;
  string email     = 2;
  string name      = 3;
  int64  created_at = 4;  // Unix timestamp
}
```

### Server Implementation (Go)

```go
// server.go
type UserServer struct {
    pb.UnimplementedUserServiceServer
    repo UserRepository
}

func (s *UserServer) GetUser(
    ctx context.Context,
    req *pb.GetUserRequest,
) (*pb.GetUserResponse, error) {

    user, err := s.repo.FindByID(ctx, req.UserId)
    if err != nil {
        // Map domain errors to gRPC status codes
        if errors.Is(err, ErrNotFound) {
            return nil, status.Errorf(codes.NotFound, "user %s not found", req.UserId)
        }
        return nil, status.Errorf(codes.Internal, "failed to fetch user: %v", err)
    }

    return &pb.GetUserResponse{
        Id:        user.ID,
        Email:     user.Email,
        Name:      user.Name,
        CreatedAt: user.CreatedAt.Unix(),
    }, nil
}

func main() {
    lis, _ := net.Listen("tcp", ":50051")
    s := grpc.NewServer(
        grpc.UnaryInterceptor(otelgrpc.UnaryServerInterceptor()), // tracing
    )
    pb.RegisterUserServiceServer(s, &UserServer{repo: NewPostgresRepo()})
    s.Serve(lis)
}
```

### Client Implementation (Node.js)

```typescript
// order-service calling user-service via gRPC
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';

const packageDef = protoLoader.loadSync('user.proto');
const userProto = grpc.loadPackageDefinition(packageDef) as any;

const client = new userProto.user.v1.UserService(
  'user-service:50051',
  grpc.credentials.createInsecure(), // use createSsl() in production
);

// Promisify the callback-based client
function getUser(userId: string): Promise<UserResponse> {
  return new Promise((resolve, reject) => {
    client.GetUser({ user_id: userId }, (err: Error, response: UserResponse) => {
      if (err) reject(err);
      else resolve(response);
    });
  });
}

// Usage
const user = await getUser('usr_01HQTM7K');
```

### gRPC Status Codes

gRPC has its own set of status codes. Map domain errors to them consistently.

| Code | HTTP Equivalent | Use case |
|------|----------------|----------|
| `OK` | 200 | Success |
| `NOT_FOUND` | 404 | Resource doesn't exist |
| `ALREADY_EXISTS` | 409 | Duplicate resource |
| `INVALID_ARGUMENT` | 400 | Bad request data |
| `PERMISSION_DENIED` | 403 | Lacks permission |
| `UNAUTHENTICATED` | 401 | Missing / invalid credentials |
| `RESOURCE_EXHAUSTED` | 429 | Rate limit hit |
| `UNAVAILABLE` | 503 | Service temporarily down |
| `DEADLINE_EXCEEDED` | 504 | Timeout |
| `INTERNAL` | 500 | Unexpected server error |

---

## REST vs gRPC — Decision Guide

```
Is this a public API (consumed by browsers, third parties, mobile apps)?
  YES → REST. gRPC has limited browser support without a proxy.

Is this an internal service-to-service call?
  Does it need high throughput (>1k req/s) or very low latency?
    YES → gRPC. The binary encoding and HTTP/2 multiplexing make a measurable difference.
    NO  → REST is fine. Simpler to debug, wider tooling.

Does the contract need to be strongly typed and enforced at compile time?
  YES → gRPC (Protobuf is better at this than OpenAPI in practice).

Does the team already use OpenAPI / Swagger and has good REST tooling?
  YES → Stay with REST unless performance demands a change.
```

---

## Asynchronous Messaging

Asynchronous messaging decouples services in time and lets multiple consumers independently process the same event. It is the backbone of event-driven architectures.

### Core Concepts

**Message**: A discrete unit of data sent between services. May be a command ("do this") or an event ("this happened").

**Queue**: A FIFO buffer. One consumer processes each message. Used for task distribution and load levelling.

**Topic / Exchange**: A broadcast channel. All subscribers receive every message. Used for event fan-out.

**Consumer Group**: A set of service instances that share the work of consuming from a topic — each message is processed by exactly one instance in the group.

**Dead Letter Queue (DLQ)**: Where messages go after repeated processing failures. Essential for debugging and recovery.

### Message Design

Events should be self-contained — a consumer must not need to call back to the producer to understand the event.

```typescript
// Bad — consumer must call User Service to get user data
{
  "type": "OrderPlaced",
  "orderId": "ord_01HQTM8Z",
  "userId": "usr_01HQTM7K"  // consumer has to look this up
}

// Good — event carries everything consumers need
{
  "type": "OrderPlaced",
  "specVersion": "1.0",
  "id": "evt_4bf92f35",
  "source": "order-service",
  "time": "2024-01-15T10:30:00Z",
  "data": {
    "orderId": "ord_01HQTM8Z",
    "userId": "usr_01HQTM7K",
    "userEmail": "alice@example.com",  // denormalised at publish time
    "items": [
      { "productId": "prod_9K2X", "name": "Widget Pro", "qty": 2, "unitPrice": 49.99 }
    ],
    "totalAmount": 99.98,
    "currency": "USD"
  }
}
```

Events should follow the [CloudEvents specification](https://cloudevents.io/) for `specVersion`, `id`, `source`, `time` envelope fields — this standardises tooling across brokers.

### Consumer Guarantees

| Guarantee | Meaning | Implication |
|-----------|---------|-------------|
| **At-most-once** | Message delivered ≤1 time, may be lost | Fastest, no duplicates, data loss possible |
| **At-least-once** | Message delivered ≥1 time, may be duplicated | Most common — consumers must be idempotent |
| **Exactly-once** | Message delivered exactly 1 time | Expensive, rarely necessary in practice |

Design consumers to be **idempotent** under at-least-once delivery — processing the same message twice should produce the same result.

```typescript
// Idempotent consumer — safe to process the same event multiple times
async function handleOrderPlaced(event: OrderPlacedEvent): Promise<void> {
  // Check if we've already processed this event ID
  const alreadyProcessed = await db.processedEvents.exists(event.id);
  if (alreadyProcessed) {
    logger.info({ eventId: event.id }, 'Skipping duplicate event');
    return;
  }

  await db.transaction(async (tx) => {
    // Process the event
    await tx.inventory.reserveItems(event.data.orderId, event.data.items);
    // Record that we processed this event
    await tx.processedEvents.insert({ eventId: event.id, processedAt: new Date() });
  });
}
```

---

## Message Brokers

### RabbitMQ

A traditional message broker built around the AMQP protocol. Strong support for complex routing, priority queues, and flexible exchange topologies.

```typescript
import amqp from 'amqplib';

// Producer
const conn = await amqp.connect('amqp://localhost');
const channel = await conn.createChannel();

await channel.assertExchange('orders', 'topic', { durable: true });
await channel.publish(
  'orders',
  'orders.placed',  // routing key
  Buffer.from(JSON.stringify(event)),
  { persistent: true, contentType: 'application/json' }
);

// Consumer with manual acknowledgement
await channel.assertQueue('inventory-service', { durable: true });
await channel.bindQueue('inventory-service', 'orders', 'orders.placed');

channel.consume('inventory-service', async (msg) => {
  if (!msg) return;
  try {
    const event = JSON.parse(msg.content.toString());
    await handleOrderPlaced(event);
    channel.ack(msg);               // acknowledge — remove from queue
  } catch (err) {
    logger.error(err, 'Failed to process message');
    channel.nack(msg, false, false); // reject — send to DLQ
  }
}, { noAck: false });
```

### Apache Kafka

A distributed event streaming platform. Kafka is a persistent, ordered, replayable log — not just a message queue.

```typescript
import { Kafka } from 'kafkajs';

const kafka = new Kafka({ brokers: ['kafka:9092'] });

// Producer
const producer = kafka.producer();
await producer.connect();
await producer.send({
  topic: 'orders.placed',
  messages: [
    {
      key: event.data.orderId,     // partition key — same order always same partition
      value: JSON.stringify(event),
      headers: { 'content-type': 'application/json' },
    },
  ],
});

// Consumer group
const consumer = kafka.consumer({ groupId: 'inventory-service' });
await consumer.connect();
await consumer.subscribe({ topic: 'orders.placed', fromBeginning: false });

await consumer.run({
  eachMessage: async ({ message }) => {
    const event = JSON.parse(message.value!.toString());
    await handleOrderPlaced(event);
    // Kafka auto-commits offset — or use manual commits for stronger guarantees
  },
});
```

Kafka's key advantage: messages are retained on disk (configurable retention, often days or weeks). Consumers can replay from any offset — invaluable for rebuilding read models (CQRS) or recovering from a bug.

---

## Kafka vs RabbitMQ

| Factor | RabbitMQ | Kafka |
|--------|---------|-------|
| **Model** | Message queue (push) | Event log (pull) |
| **Message retention** | Deleted after ACK | Persisted for retention period |
| **Replay** | No | Yes — rewind to any offset |
| **Throughput** | Moderate (~50k msg/s) | Very high (~1M msg/s per partition) |
| **Ordering** | Per-queue | Per-partition |
| **Routing** | Rich (topic/fanout/direct/headers exchanges) | Simple (topic, partition by key) |
| **Consumer model** | Push (broker delivers) | Pull (consumer fetches) |
| **Operational complexity** | Low–medium | High (ZooKeeper/KRaft, partition tuning) |
| **Best for** | Task queues, complex routing, RPC patterns | Event streaming, audit logs, CQRS projections |

Choose RabbitMQ for task distribution and complex routing. Choose Kafka when you need message persistence, high throughput, or replayable event streams.

---

## Event-Driven Patterns

### Event Notification

The simplest pattern. A service publishes a minimal event telling other services that something happened. Consumers fetch additional details if needed.

```json
{ "type": "UserCreated", "userId": "usr_123", "timestamp": "..." }
```

Low coupling — the producer doesn't need to know what consumers will do. Trade-off: consumers make extra API calls.

### Event-Carried State Transfer

The event carries all the data consumers need — no callbacks required. Higher payload, but consumers are fully self-sufficient.

```json
{
  "type": "UserCreated",
  "userId": "usr_123",
  "email": "alice@example.com",
  "name": "Alice",
  "plan": "pro",
  "timestamp": "..."
}
```

This is the recommended default. Design events to be self-contained.

### Request-Reply over Messaging

Async request-response — the caller publishes a command and waits on a reply queue for the response. Used for async workflows that still need a result.

```typescript
// Caller
const correlationId = crypto.randomUUID();
const replyQueue = `reply.${correlationId}`;

await channel.assertQueue(replyQueue, { exclusive: true, autoDelete: true });
await channel.publish('commands', 'payments.charge', Buffer.from(JSON.stringify({
  orderId, amount, correlationId, replyTo: replyQueue,
})));

// Wait for reply with timeout
const reply = await waitForReply(channel, replyQueue, 5000);
```

### Event Sourcing as Communication

When combined with event sourcing (see [03-database-patterns.md](./03-database-patterns.md#event-sourcing)), the event log itself becomes the communication medium. Services subscribe to the event store directly rather than a separate broker.

---

## Service Mesh

At scale, managing retries, timeouts, mutual TLS, and observability across every service-to-service call becomes unmanageable if each service implements it independently. A **service mesh** moves these cross-cutting concerns out of application code and into the infrastructure.

### Architecture

A service mesh injects a **sidecar proxy** (typically Envoy) alongside every service instance. All traffic flows through the sidecar — not directly between services.

```
Service A  ←→  Envoy Sidecar A
                      ↕
               Control Plane
               (Istio / Linkerd)
                      ↕
Service B  ←→  Envoy Sidecar B
```

### What the Mesh Handles

| Concern | Without mesh | With mesh |
|---------|-------------|-----------|
| mTLS between services | Each service implements it | Automatic, transparent |
| Retries & timeouts | Coded in each service | Policy in control plane |
| Circuit breaking | Library per language | Universal policy |
| Distributed tracing | Instrumentation per service | Automatic header propagation |
| Traffic splitting (canary) | Custom routing code | Declarative weight rules |
| Rate limiting | Per-service implementation | Mesh-level policy |

### When to Adopt a Service Mesh

A service mesh adds significant operational complexity. Don't adopt one prematurely.

Adopt when:
- You have 20+ services and cross-cutting concerns are becoming a maintenance burden
- You need mTLS for compliance (PCI-DSS, HIPAA)
- You need fine-grained traffic control (canary releases, A/B routing at the infrastructure level)

Defer when:
- Your team is still learning microservices fundamentals
- You have fewer than 10 services
- Your current tooling handles retries and TLS adequately

Popular choices: **Istio** (most features, highest complexity), **Linkerd** (simpler, lighter), **Consul Connect** (good if you already use Consul for service discovery).

---

## Communication Resilience

Failures in service communication are normal, not exceptional. Design for them explicitly.

### Retries with Exponential Backoff

Never retry immediately — a failing service is usually overloaded, and instant retries make it worse. Use exponential backoff with jitter.

```typescript
async function callWithRetry<T>(
  fn: () => Promise<T>,
  maxAttempts = 3,
  baseDelayMs = 100,
): Promise<T> {
  let lastError: Error;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err as Error;

      // Don't retry on client errors (4xx) — they won't succeed
      if (err instanceof HttpError && err.status < 500) throw err;

      if (attempt < maxAttempts) {
        const delay = baseDelayMs * Math.pow(2, attempt - 1)
          + Math.random() * 100; // jitter
        await sleep(delay);
      }
    }
  }

  throw lastError!;
}
```

Only retry on transient errors (network timeouts, 429, 503). Never retry 4xx errors.

### Circuit Breaker

Stops calling a service that is clearly failing, giving it time to recover and preventing cascading failures. See [08-resilience.md](./08-resilience.md#circuit-breaker) for a full implementation.

```
CLOSED (normal) → too many failures → OPEN (reject calls) → timeout → HALF-OPEN (test) → success → CLOSED
```

### Bulkhead

Isolate communication with different downstream services into separate connection pools / thread pools. A slow or failing downstream service can only exhaust its own pool, not starve calls to all other services.

```typescript
// Separate pools per downstream service
const userServicePool    = new ConnectionPool({ maxConnections: 20 });
const paymentServicePool = new ConnectionPool({ maxConnections: 10 });
const inventoryPool      = new ConnectionPool({ maxConnections: 15 });

// Payment going down can't exhaust User Service connections
```

### Timeout Hierarchy

Set timeouts at every layer, decreasing from outer to inner.

```
Client request timeout:   10s
  └─ Service A timeout:   8s
       └─ Service B timeout:  5s
            └─ DB query timeout: 3s
```

If inner timeouts are longer than outer timeouts, the inner timeout never fires — the outer timeout cuts the connection first while the inner operation keeps running, leaking resources.

---

## Choosing the Right Pattern

A quick decision reference for common scenarios.

| Scenario | Pattern | Protocol |
|----------|---------|----------|
| User submits a form, needs immediate feedback | Synchronous | REST |
| Service needs data from another service to proceed | Synchronous | REST or gRPC |
| High-frequency internal calls between services | Synchronous | gRPC |
| Order placed → notify, reserve stock, log analytics | Asynchronous | Kafka topic |
| Background job processing (resize images, send email) | Asynchronous | RabbitMQ queue |
| Long-running saga across 4 services | Asynchronous | Kafka + Saga orchestrator |
| Rebuild a read model after a bug | Async with replay | Kafka (replay from offset) |
| External partner API integration | Synchronous | REST (with webhook callbacks) |
| Two services chatting at ~1M msg/s | Asynchronous | Kafka |
| Complex routing (priority queues, dead-lettering) | Asynchronous | RabbitMQ |

---

## Summary & Next Steps

REST for external and simple internal APIs. gRPC for high-throughput internal calls where performance is measurable. Async messaging for anything that doesn't need an immediate response — which is most things. Make every consumer idempotent, set timeouts on every sync call, and design events to be self-contained.

The most important rule: async is the default for workflows that span services. Synchronous chains of more than 2 hops are an architecture smell.

### Recommended Reading Order

| Step | Document | What you'll learn |
|------|----------|------------------|
| Next | [05-deployment-strategies.md](./05-deployment-strategies.md) | Getting services into production safely |
| Also | [06-security.md](./06-security.md) | Securing service-to-service communication with mTLS and JWT |
| Also | [08-resilience.md](./08-resilience.md) | Circuit breakers, retries, bulkheads — full implementations |
| Also | [07-observability.md](./07-observability.md) | Tracing async event flows across services |

---

*Part of the [Microservices Architecture Guide](../../README.md)*  
*Previous: [03-database-patterns.md](./03-database-patterns.md)*  
*Next: [05-deployment-strategies.md](./05-deployment-strategies.md)*