# Monolithic Architecture

## Overview

Monolithic architecture is a traditional software design pattern where an application is built as a single, unified unit. All components of the application—user interface, business logic, and data access layers—are interconnected and run as a single service.

## Table of Contents

- [What is Monolithic Architecture?](#what-is-monolithic-architecture)
- [Core Characteristics](#core-characteristics)
- [Architecture Layers](#architecture-layers)
- [How It Works](#how-it-works)
- [Key Components](#key-components)
- [When to Use](#when-to-use)
- [Related Documents](#related-documents)

## What is Monolithic Architecture?

A monolithic application is a single-tiered software application where different components are combined into a single program from a single platform. The application is self-contained, with all functionalities tightly coupled and running in a single process.

### Key Definition

> **Monolithic**: A unified, indivisible application where all functional components are interconnected and interdependent, deployed as a single unit.

## Core Characteristics

### 1. **Single Codebase**
- Entire application exists in one repository
- All modules share the same code base
- Version control is centralized

### 2. **Single Deployment Unit**
- Application deployed as one artifact (WAR, JAR, EXE)
- All-or-nothing deployment approach
- Cannot deploy individual features independently

### 3. **Shared Memory Space**
- All components run in the same process
- Direct method calls between modules
- No network latency for internal communication

### 4. **Unified Database**
- Single database serves entire application
- All modules access same data store
- Strong consistency through ACID transactions

### 5. **Tightly Coupled Components**
- Modules have dependencies on each other
- Changes in one module may affect others
- Shared libraries and dependencies

## Architecture Layers

A typical monolithic application follows a layered architecture pattern:

```mermaid
graph TB
    subgraph "Monolithic Application"
        PL[Presentation Layer<br/>UI, Controllers, APIs]
        BLL[Business Logic Layer<br/>Services, Domain Logic, Rules]
        DAL[Data Access Layer<br/>Repositories, ORM, Queries]
        DB[(Database Layer<br/>Single Relational Database)]
        
        PL --> BLL
        BLL --> DAL
        DAL --> DB
    end
    E
```

### Layer Responsibilities

#### **Presentation Layer**
- Handles user interactions
- Exposes REST APIs or web interfaces
- Request routing and response formatting
- Input validation and error handling

#### **Business Logic Layer**
- Implements core business rules
- Processes and transforms data
- Orchestrates workflows
- Enforces business constraints

#### **Data Access Layer**
- Abstracts database operations
- Implements repository pattern
- Handles data mapping (ORM)
- Manages database connections

#### **Database Layer**
- Stores application data
- Ensures data integrity
- Handles transactions
- Manages relationships

## How It Works

### Request Flow

```mermaid
sequenceDiagram
    participant Client
    participant LB as Load Balancer
    participant Controller as Presentation Layer
    participant Service as Business Logic
    participant Repository as Data Access
    participant DB as Database
    
    Client->>LB: HTTP Request
    LB->>Controller: Route Request
    Controller->>Controller: Validate Input
    Controller->>Service: Process Request
    Service->>Service: Apply Business Rules
    Service->>Repository: Query/Update Data
    Repository->>DB: Execute SQL
    DB-->>Repository: Return Data
    Repository-->>Service: Entity Objects
    Service-->>Controller: Processed Data
    Controller-->>LB: HTTP Response
    LB-->>Client: Response
```

### Example Flow

```mermaid
graph LR
    User[User/Client] --> LB[Load Balancer]
    LB --> Controller
    Controller --> Service[Service Layer]
    Service --> Repo[Repository]
    Repo --> DB[(Database)]
    DB --> Repo
    Repo --> Service
    Service --> Controller
    Controller --> Response[HTTP Response]
    Response --> User
    
    style User fill:#e1f5ff
    style DB fill:#e1ffe1
    style Response fill:#d4edda
```

## Key Components

```mermaid
graph TB
    subgraph "Monolithic Application Components"
        AS[Application Server<br/>Tomcat, IIS, WildFly]
        App[Application Code<br/>Single Deployable Unit]
        Cache[Cache Layer<br/>Redis, Memcached]
        DB[(Database<br/>PostgreSQL, MySQL,<br/>SQL Server, Oracle)]
        
        AS --> App
        App --> Cache
        App --> DB
    end
    
    subgraph "Build & Deployment"
        Build[Build Tools<br/>Maven, Gradle, MSBuild]
        Artifact[Deployable Artifact<br/>JAR, WAR, DLL, EXE]
        Deploy[Deployment<br/>Single Unit Deploy]
        
        Build --> Artifact
        Artifact --> Deploy
    end
    
    style AS fill:#e1f5ff
    style App fill:#fff4e1
    style Cache fill:#f0e1ff
    style DB fill:#e1ffe1
```

## When to Use

```mermaid
graph TB
    Decision{Evaluate Your<br/>Requirements}
    
    Decision -->|Small Scale| Ideal1[Small to Medium Apps<br/>✓ Use Monolithic]
    Decision -->|Quick Launch| Ideal2[Startup MVPs<br/>✓ Use Monolithic]
    Decision -->|Simple Logic| Ideal3[CRUD Applications<br/>✓ Use Monolithic]
    Decision -->|Limited Users| Ideal4[Internal Tools<br/>✓ Use Monolithic]
    Decision -->|Small Team| Ideal5[2-10 Developers<br/>✓ Use Monolithic]
    
    Decision -->|Large Scale| Avoid1[Thousands of Users<br/>✗ Avoid Monolithic]
    Decision -->|Rapid Changes| Avoid2[Frequent Releases<br/>✗ Avoid Monolithic]
    Decision -->|Big Teams| Avoid3[Multiple Teams<br/>✗ Avoid Monolithic]
    Decision -->|Tech Diversity| Avoid4[Different Tech Stacks<br/>✗ Avoid Monolithic]
    Decision -->|Selective Scaling| Avoid5[Independent Scaling<br/>✗ Avoid Monolithic]
    
    style Ideal1 fill:#d4edda
    style Ideal2 fill:#d4edda
    style Ideal3 fill:#d4edda
    style Ideal4 fill:#d4edda
    style Ideal5 fill:#d4edda
    style Avoid1 fill:#f8d7da
    style Avoid2 fill:#f8d7da
    style Avoid3 fill:#f8d7da
    style Avoid4 fill:#f8d7da
    style Avoid5 fill:#f8d7da
```

## Development Workflow

```mermaid
graph LR
    subgraph "Local Development"
        Clone[Clone Repo] --> Install[Install Dependencies]
        Install --> Dev[Run Locally]
        Dev --> Test[Run Tests]
    end
    
    subgraph "Build Process"
        Build[Build Application] --> UnitTest[Run Tests]
        UnitTest --> Package[Generate Artifact<br/>JAR/DLL/Bundle]
    end
    
    subgraph "Deployment"
        Package --> Deploy[Deploy to Server]
        Deploy --> Restart[Restart Service]
        Restart --> Monitor[Monitor & Verify]
    end
    
    Test --> Build
    
```

## Technology Stack Examples

```mermaid
graph TB
    subgraph "Java/Spring Boot Stack"
        J1[Spring Boot Framework]
        J2[PostgreSQL/MySQL]
        J3[Hibernate/JPA ORM]
        J4[Maven/Gradle Build]
        J5[Embedded Tomcat]
        
        J1 --> J3
        J3 --> J2
        J4 --> J1
        J1 --> J5
    end
    
    subgraph ".NET Stack"
        N1[ASP.NET Core]
        N2[SQL Server]
        N3[Entity Framework]
        N4[MSBuild]
        N5[IIS/Kestrel]
        
        N1 --> N3
        N3 --> N2
        N4 --> N1
        N1 --> N5
    end
    
    subgraph "Python/Django Stack"
        P1[Django Framework]
        P2[PostgreSQL]
        P3[Django ORM]
        P4[Gunicorn + Nginx]
        
        P1 --> P3
        P3 --> P2
        P1 --> P4
    end
    
    subgraph "Node.js Stack"
        Node1[Express.js]
        Node2[MongoDB/PostgreSQL]
        Node3[Mongoose/Sequelize]
        Node4[Node.js Server]
        
        Node1 --> Node3
        Node3 --> Node2
        Node1 --> Node4
    end
    
```

## Scaling Strategies

```mermaid
graph TB
    App[Monolithic Application]
    
    subgraph "Vertical Scaling"
        VS[Increase Server Resources]
        CPU[More CPU Cores]
        RAM[More RAM]
        Disk[Faster Storage]
        
        VS --> CPU
        VS --> RAM
        VS --> Disk
    end
    
    subgraph "Horizontal Scaling"
        LB[Load Balancer]
        I1[Instance 1]
        I2[Instance 2]
        I3[Instance 3]
        In[Instance N]
        
        LB --> I1
        LB --> I2
        LB --> I3
        LB --> In
    end
    
    subgraph "Database Scaling"
        Master[(Master DB<br/>Write)]
        Replica1[(Replica 1<br/>Read)]
        Replica2[(Replica 2<br/>Read)]
        Cache[Cache Layer<br/>Redis/Memcached]
        
        Master -.->|Replicate| Replica1
        Master -.->|Replicate| Replica2
    end
    
    App --> VS
    App --> LB
    I1 --> Master
    I2 --> Master
    I3 --> Master
    I1 --> Replica1
    I2 --> Replica2
    I1 --> Cache
    I2 --> Cache
    I3 --> Cache

```

## Testing

```mermaid
graph TB
    subgraph "Testing Pyramid"
        E2E[End-to-End Tests<br/>Few, Slow, Comprehensive]
        INT[Integration Tests<br/>Moderate, Medium Speed]
        UNIT[Unit Tests<br/>Many, Fast, Isolated]
        
        E2E --> INT
        INT --> UNIT
    end
    
    subgraph "Unit Tests"
        UT1[Test Individual Classes]
        UT2[Mock Dependencies]
        UT3[Fast Execution]
        UT4[High Coverage]
    end
    
    subgraph "Integration Tests"
        IT1[Test Multiple Layers]
        IT2[Database Integration]
        IT3[API Endpoints]
        IT4[Service Interaction]
    end
    
    subgraph "End-to-End Tests"
        ET1[Complete User Workflows]
        ET2[UI Automation<br/>Selenium, Playwright]
        ET3[Full Stack Testing]
        ET4[Production-like Environment]
    end
    
    UNIT -.-> UT1
    UNIT -.-> UT2
    UNIT -.-> UT3
    UNIT -.-> UT4
    
    INT -.-> IT1
    INT -.-> IT2
    INT -.-> IT3
    INT -.-> IT4
    
    E2E -.-> ET1
    E2E -.-> ET2
    E2E -.-> ET3
    E2E -.-> ET4
    
```

## Related Documents

- **[pros-cons.md](./pros-cons.md)**: Detailed advantages and disadvantages
- **[use-cases.md](./use-cases.md)**: When to use monolithic architecture
- **[examples.md](./examples.md)**: Real-world application examples

## Further Reading

- [Microservices Architecture](../microservices/readme.md) - Alternative pattern
- [Layered Architecture](../layered-architecture/readme.md) - Architectural style
- [Event-Driven Architecture](../event-driven/readme.md) - Communication pattern

---

**Last Updated**: October 2025  
**Maintainer**: System Design Team