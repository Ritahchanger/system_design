# Event-Driven Architecture Flow

## Overview

This document provides detailed explanations and diagrams of how events flow through an event-driven architecture system, from generation to consumption and error handling.

## Table of Contents

1. [Basic Event Flow](#basic-event-flow)
2. [Publish-Subscribe Pattern](#publish-subscribe-pattern)
3. [Point-to-Point Pattern](#point-to-point-pattern)
4. [Event Streaming Flow](#event-streaming-flow)
5. [Error Handling and Retry Flow](#error-handling-and-retry-flow)
6. [Complex Multi-Service Flow](#complex-multi-service-flow)
7. [Event Choreography](#event-choreography)
8. [Event Orchestration](#event-orchestration)

---

## Basic Event Flow

The fundamental flow in an event-driven architecture involves three main stages: event generation, event routing, and event processing.

```mermaid
sequenceDiagram
    participant Producer
    participant EventBus as Event Bus/Broker
    participant Consumer1 as Consumer 1
    participant Consumer2 as Consumer 2

    Producer->>EventBus: 1. Publish Event
    EventBus->>EventBus: 2. Store Event
    EventBus->>Consumer1: 3. Deliver Event
    EventBus->>Consumer2: 3. Deliver Event
    Consumer1->>Consumer1: 4. Process Event
    Consumer2->>Consumer2: 4. Process Event
    Consumer1->>EventBus: 5. Acknowledge
    Consumer2->>EventBus: 5. Acknowledge
```

### Flow Steps

1. **Event Generation**: Producer detects a state change or action and creates an event
2. **Event Publication**: Producer publishes the event to the event bus/broker
3. **Event Storage**: Event bus persists the event (optional, depends on broker)
4. **Event Routing**: Event bus routes the event to all interested subscribers
5. **Event Delivery**: Consumers receive the event asynchronously
6. **Event Processing**: Each consumer processes the event independently
7. **Acknowledgment**: Consumers acknowledge successful processing

---

## Publish-Subscribe Pattern

In this pattern, multiple consumers can subscribe to the same event type, enabling one-to-many communication.

```mermaid
graph LR
    A[Order Service] -->|OrderPlaced Event| B[Event Bus]
    B -->|Subscribe| C[Inventory Service]
    B -->|Subscribe| D[Payment Service]
    B -->|Subscribe| E[Notification Service]
    B -->|Subscribe| F[Analytics Service]
    
    style A fill:#e1f5ff
    style B fill:#fff4e1
    style C fill:#e8f5e9
    style D fill:#e8f5e9
    style E fill:#e8f5e9
    style F fill:#e8f5e9
```

### Characteristics

- **Decoupled**: Producers don't know about consumers
- **Scalable**: Easy to add new subscribers
- **Flexible**: Each consumer can process events differently
- **Parallel Processing**: All consumers receive events simultaneously

### Example Flow: E-commerce Order

```mermaid
sequenceDiagram
    participant OS as Order Service
    participant EB as Event Bus
    participant IS as Inventory Service
    participant PS as Payment Service
    participant NS as Notification Service

    OS->>EB: Publish: OrderPlaced
    Note over EB: Topic: orders.placed
    
    par Parallel Processing
        EB->>IS: OrderPlaced Event
        IS->>IS: Reserve Inventory
        IS->>EB: Publish: InventoryReserved
    and
        EB->>PS: OrderPlaced Event
        PS->>PS: Process Payment
        PS->>EB: Publish: PaymentProcessed
    and
        EB->>NS: OrderPlaced Event
        NS->>NS: Send Confirmation Email
    end
```

---

## Point-to-Point Pattern

In this pattern, each event is consumed by exactly one consumer, typically used for work distribution.

```mermaid
graph LR
    A[Producer] -->|Send Message| B[Queue]
    B -->|Consume| C[Consumer 1]
    B -->|Consume| D[Consumer 2]
    B -->|Consume| E[Consumer 3]
    
    style A fill:#e1f5ff
    style B fill:#fff4e1
    style C fill:#e8f5e9
    style D fill:#e8f5e9
    style E fill:#e8f5e9
```

### Load Balancing Example

```mermaid
sequenceDiagram
    participant P as Producer
    participant Q as Message Queue
    participant C1 as Worker 1
    participant C2 as Worker 2
    participant C3 as Worker 3

    P->>Q: Message 1
    P->>Q: Message 2
    P->>Q: Message 3
    P->>Q: Message 4
    P->>Q: Message 5
    
    Q->>C1: Message 1
    Q->>C2: Message 2
    Q->>C3: Message 3
    Q->>C1: Message 4
    Q->>C2: Message 5
    
    C1->>Q: ACK
    C2->>Q: ACK
    C3->>Q: ACK
```

---

## Event Streaming Flow

Event streaming involves continuous processing of events as they arrive, often with the ability to replay events.

```mermaid
graph TB
    subgraph "Event Stream"
        E1[Event 1<br/>t=0]
        E2[Event 2<br/>t=1]
        E3[Event 3<br/>t=2]
        E4[Event 4<br/>t=3]
        E5[Event 5<br/>t=4]
    end
    
    subgraph "Consumers"
        C1[Consumer A<br/>Reading from t=0]
        C2[Consumer B<br/>Reading from t=2]
        C3[Consumer C<br/>Reading from t=4]
    end
    
    E1 --> C1
    E2 --> C1
    E3 --> C1
    E3 --> C2
    E4 --> C1
    E4 --> C2
    E5 --> C1
    E5 --> C2
    E5 --> C3
    
    style E1 fill:#e3f2fd
    style E2 fill:#e3f2fd
    style E3 fill:#e3f2fd
    style E4 fill:#e3f2fd
    style E5 fill:#e3f2fd
```

### Stream Processing Flow

```mermaid
sequenceDiagram
    participant P as Producer
    participant K as Kafka Topic
    participant SP as Stream Processor
    participant DB as Database
    participant A as Analytics

    loop Continuous Stream
        P->>K: Publish Events
        K->>SP: Stream Events
        SP->>SP: Transform/Aggregate
        SP->>DB: Store Results
        SP->>A: Send Metrics
    end
```

---

## Error Handling and Retry Flow

Robust error handling is crucial in event-driven systems. This includes retry mechanisms and dead letter queues.

```mermaid
graph TB
    A[Event Bus] -->|Deliver Event| B[Consumer]
    B -->|Process| C{Success?}
    C -->|Yes| D[Acknowledge]
    C -->|No| E{Retry Count < Max?}
    E -->|Yes| F[Retry with Backoff]
    F --> B
    E -->|No| G[Dead Letter Queue]
    G --> H[Manual Review]
    
    style C fill:#fff9c4
    style E fill:#fff9c4
    style G fill:#ffcdd2
```

### Detailed Retry Flow

```mermaid
sequenceDiagram
    participant EB as Event Bus
    participant C as Consumer
    participant DLQ as Dead Letter Queue
    participant M as Monitor/Alert

    EB->>C: Deliver Event (Attempt 1)
    C->>C: Process
    C--xEB: Processing Failed
    
    Note over EB,C: Wait 1 second
    EB->>C: Deliver Event (Attempt 2)
    C->>C: Process
    C--xEB: Processing Failed
    
    Note over EB,C: Wait 4 seconds (exponential backoff)
    EB->>C: Deliver Event (Attempt 3)
    C->>C: Process
    C--xEB: Processing Failed
    
    Note over EB,C: Wait 9 seconds
    EB->>C: Deliver Event (Attempt 4)
    C->>C: Process
    C--xEB: Processing Failed
    
    EB->>DLQ: Move to DLQ
    DLQ->>M: Trigger Alert
    M->>M: Notify Operations Team
```

### Circuit Breaker Pattern

```mermaid
stateDiagram-v2
    [*] --> Closed
    Closed --> Open: Failure threshold exceeded
    Open --> HalfOpen: Timeout expires
    HalfOpen --> Closed: Success
    HalfOpen --> Open: Failure
    
    note right of Closed
        Normal operation
        Requests processed
    end note
    
    note right of Open
        Fast fail
        No requests sent
    end note
    
    note right of HalfOpen
        Limited requests
        Testing recovery
    end note
```

---

## Complex Multi-Service Flow

Real-world scenarios often involve multiple services coordinating through events.

### E-commerce Order Processing

```mermaid
sequenceDiagram
    participant U as User
    participant OS as Order Service
    participant EB as Event Bus
    participant IS as Inventory Service
    participant PS as Payment Service
    participant SS as Shipping Service
    participant NS as Notification Service

    U->>OS: Place Order
    OS->>EB: Publish: OrderCreated
    
    par Check Inventory
        EB->>IS: OrderCreated
        IS->>IS: Check Stock
        alt Stock Available
            IS->>EB: Publish: InventoryReserved
        else Out of Stock
            IS->>EB: Publish: InventoryUnavailable
            EB->>NS: InventoryUnavailable
            NS->>U: Send "Out of Stock" Email
        end
    end
    
    EB->>PS: InventoryReserved
    PS->>PS: Process Payment
    alt Payment Success
        PS->>EB: Publish: PaymentCompleted
        EB->>SS: PaymentCompleted
        SS->>SS: Create Shipment
        SS->>EB: Publish: ShipmentCreated
        EB->>NS: ShipmentCreated
        NS->>U: Send "Order Shipped" Email
    else Payment Failed
        PS->>EB: Publish: PaymentFailed
        EB->>IS: PaymentFailed
        IS->>IS: Release Inventory
        EB->>NS: PaymentFailed
        NS->>U: Send "Payment Failed" Email
    end
```

---

## Event Choreography

In choreography, services react to events without a central coordinator. Each service knows what to do when specific events occur.

```mermaid
graph TB
    subgraph "Order Flow - Choreography"
        A[Order Service] -->|OrderPlaced| B[Event Bus]
        B --> C[Inventory Service]
        C -->|InventoryReserved| B
        B --> D[Payment Service]
        D -->|PaymentProcessed| B
        B --> E[Shipping Service]
        E -->|OrderShipped| B
        B --> F[Notification Service]
    end
    
    style A fill:#e1f5ff
    style C fill:#e8f5e9
    style D fill:#e8f5e9
    style E fill:#e8f5e9
    style F fill:#e8f5e9
    style B fill:#fff4e1
```

### Characteristics

- **Decentralized**: No single point of control
- **Loosely Coupled**: Services don't know about each other
- **Flexible**: Easy to add new services
- **Complex**: Can be difficult to understand overall flow

---

## Event Orchestration

In orchestration, a central orchestrator manages the workflow and tells services what to do.

```mermaid
sequenceDiagram
    participant U as User
    participant O as Orchestrator
    participant IS as Inventory Service
    participant PS as Payment Service
    participant SS as Shipping Service
    participant NS as Notification Service

    U->>O: Place Order
    
    O->>IS: Reserve Inventory
    IS-->>O: Inventory Reserved
    
    O->>PS: Process Payment
    PS-->>O: Payment Success
    
    O->>SS: Create Shipment
    SS-->>O: Shipment Created
    
    O->>NS: Send Notification
    NS-->>O: Notification Sent
    
    O->>U: Order Completed
```

### Orchestration with Saga Pattern

```mermaid
graph TB
    Start([Start Order]) --> Step1[Reserve Inventory]
    Step1 --> Check1{Success?}
    Check1 -->|Yes| Step2[Process Payment]
    Check1 -->|No| Comp1[End - Failed]
    
    Step2 --> Check2{Success?}
    Check2 -->|Yes| Step3[Create Shipment]
    Check2 -->|No| Comp2[Compensate: Release Inventory]
    Comp2 --> Comp1
    
    Step3 --> Check3{Success?}
    Check3 -->|Yes| Step4[Send Notification]
    Check3 -->|No| Comp3[Compensate: Refund Payment]
    Comp3 --> Comp4[Compensate: Release Inventory]
    Comp4 --> Comp1
    
    Step4 --> End([Order Complete])
    
    style Check1 fill:#fff9c4
    style Check2 fill:#fff9c4
    style Check3 fill:#fff9c4
    style Comp1 fill:#ffcdd2
    style Comp2 fill:#ffcdd2
    style Comp3 fill:#ffcdd2
    style Comp4 fill:#ffcdd2
    style End fill:#c8e6c9
```

---

## Event Sourcing Integration

When combining event-driven architecture with event sourcing, events become the source of truth.

```mermaid
sequenceDiagram
    participant C as Command Handler
    participant ES as Event Store
    participant EB as Event Bus
    participant R as Read Model
    participant Q as Query Handler

    C->>ES: Save Events
    ES->>ES: Append to Stream
    ES->>EB: Publish Events
    
    par Update Read Models
        EB->>R: Event 1
        R->>R: Update Projection
    and
        EB->>R: Event 2
        R->>R: Update Projection
    end
    
    Note over Q,R: Later...
    Q->>R: Query Data
    R-->>Q: Return Result
```

### Event Store Structure

```mermaid
graph LR
    subgraph "Event Store"
        E1[Event 1<br/>OrderCreated<br/>v1]
        E2[Event 2<br/>ItemAdded<br/>v2]
        E3[Event 3<br/>PaymentProcessed<br/>v3]
        E4[Event 4<br/>OrderShipped<br/>v4]
    end
    
    E1 --> E2
    E2 --> E3
    E3 --> E4
    
    E4 -.->|Replay| R[Current State<br/>Reconstruction]
    
    style E1 fill:#e3f2fd
    style E2 fill:#e3f2fd
    style E3 fill:#e3f2fd
    style E4 fill:#e3f2fd
    style R fill:#c8e6c9
```

---

## Key Takeaways

### Flow Principles

1. **Asynchronous Communication**: Events are processed independently
2. **Loose Coupling**: Producers and consumers don't depend on each other
3. **Scalability**: Add consumers without modifying producers
4. **Resilience**: Failed consumers don't affect others
5. **Flexibility**: Change event handlers without affecting producers

### Best Practices

- **Event Naming**: Use past tense (OrderCreated, PaymentProcessed)
- **Event Size**: Keep events small and focused
- **Idempotency**: Handle duplicate events gracefully
- **Ordering**: Don't rely on strict ordering unless necessary
- **Monitoring**: Track event flow and processing times

### Common Pitfalls

- **Event Chain Complexity**: Too many chained events become hard to debug
- **Missing Error Handling**: Always plan for failures
- **Tight Coupling**: Events with too much detail couple services
- **Lost Events**: Ensure reliable delivery and persistence
- **Version Management**: Plan for event schema evolution

---

## Related Documentation

- [README.md](./readme.md) - Overview of Event-Driven Architecture
- [pros-cons.md](./pros-cons.md) - Advantages and disadvantages
- [use-cases.md](./use-cases.md) - Real-world implementation examples