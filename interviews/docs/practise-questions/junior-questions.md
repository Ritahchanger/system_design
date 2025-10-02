# Junior Level System Design Practice Questions

## Table of Contents

1. [Overview](#overview)
2. [Question Categories](#question-categories)
3. [Essential Questions (Must Practice)](#essential-questions-must-practice)
4. [Intermediate Questions](#intermediate-questions)
5. [Bonus Challenge Questions](#bonus-challenge-questions)
6. [Question Templates & Frameworks](#question-templates--frameworks)
7. [Evaluation Criteria](#evaluation-criteria)
8. [Progression to Senior Level](#progression-to-senior-level)

---

## Overview

### Purpose
This collection provides system design practice questions specifically tailored for junior engineers (0-3 years experience). These questions focus on fundamental concepts and gradually build complexity to prepare candidates for real interviews.

### How to Use This Guide
1. **Start with Essential Questions:** Master the basics before moving on
2. **Practice Out Loud:** Explain your thinking as you work through problems
3. **Time Yourself:** Aim for 45-60 minutes per question
4. **Review Solutions:** Compare your approach with the provided guidance
5. **Progress Gradually:** Move to intermediate questions once comfortable

### Connection to Other Guides
- **Interview Process:** See [Interview Process Guide](../engineer-interview-guide/interview-process.md) for complete process
- **Evaluation:** Reference [Junior Guide](../engineer-interview-guide/junior-guide.md) for evaluation criteria
- **Next Level:** Progress to [Senior Questions](./senior-questions.md) when ready
- **Trade-offs:** Use [Trade-offs Guide](../estimation-techniques/trade-offs-decisions.md) for decision-making

---

## Question Categories

### Category A: CRUD Applications (Foundation Level)
**Focus:** Basic operations, simple data modeling, REST APIs
**Time Allocation:** 45 minutes
**Skills Tested:** Database design, API design, basic architecture

### Category B: Content & File Management (Beginner+)
**Focus:** File handling, user interactions, basic relationships
**Time Allocation:** 50 minutes
**Skills Tested:** File storage, user relationships, basic scaling

### Category C: Real-time Features (Intermediate)
**Focus:** Event-driven thinking, basic messaging concepts
**Time Allocation:** 55 minutes
**Skills Tested:** WebSockets, event handling, state management

### Category D: E-commerce Basics (Intermediate+)
**Focus:** Transaction handling, inventory, order processing
**Time Allocation:** 60 minutes
**Skills Tested:** Transaction concepts, business logic, data consistency

```mermaid
graph LR
    A[Category A<br/>CRUD Apps] --> B[Category B<br/>File Management]
    B --> C[Category C<br/>Real-time Features]
    C --> D[Category D<br/>E-commerce Basics]
    
    D -.-> E[Senior Level<br/>Questions]
    
    style A fill:#e8f5e8
    style B fill:#fff3e0
    style C fill:#f3e5f5
    style D fill:#ffebee
    style E fill:#f5f5f5
```

---

## Essential Questions (Must Practice)

### Question 1: URL Shortener Service (Like bit.ly)
**Difficulty:** ⭐⭐☆☆☆
**Time:** 45 minutes
**Category:** A - CRUD Applications

#### Problem Statement
Design a URL shortening service that allows users to:
- Submit long URLs and receive short URLs
- Click short URLs to redirect to original URLs
- View basic analytics (optional)

#### Expected Architecture
```mermaid
graph TB
    User[User] --> Web[Web Interface]
    Web --> API[REST API Server]
    API --> Cache[Redis Cache\nPopular URLs]
    API --> DB[(PostgreSQL\nURL Mappings)]
    
    subgraph "Database Schema"
        URLs["urls table\n- id: bigint\n- short_code: varchar(10)\n- original_url: text\n- created_at: timestamp\n- click_count: integer"]
    end
    
    DB --> URLs
```

#### Key Discussion Points
1. **Short code generation:** Base62 encoding vs random strings
2. **Database design:** Primary key strategy, indexing
3. **Caching strategy:** Which URLs to cache and when
4. **API design:** POST /shorten, GET /{code} endpoints

#### Evaluation Focus
- Can design basic database schema
- Understands REST API principles
- Considers simple performance optimizations
- Thinks about user experience

---

### Question 2: Personal Task Management System
**Difficulty:** ⭐⭐☆☆☆
**Time:** 45 minutes
**Category:** A - CRUD Applications

#### Problem Statement
Design a personal todo/task management application where users can:
- Create, update, delete tasks
- Organize tasks into lists/projects
- Mark tasks as complete
- Set due dates and priorities

#### Expected Data Model
```mermaid
erDiagram
    USERS {
        bigint id PK
        string email UK
        string password_hash
        string first_name
        string last_name
        timestamp created_at
        timestamp updated_at
    }
    
    LISTS {
        bigint id PK
        bigint user_id FK
        string name
        string description
        string color
        timestamp created_at
        timestamp updated_at
    }
    
    TASKS {
        bigint id PK
        bigint list_id FK
        string title
        text description
        boolean completed
        string priority
        timestamp due_date
        integer position
        timestamp created_at
        timestamp updated_at
    }
    
    USERS ||--o{ LISTS : "owns"
    LISTS ||--o{ TASKS : "contains"
```

#### API Design Example
```mermaid
sequenceDiagram
    participant App as Mobile App
    participant API as REST API
    participant DB as Database
    
    App->>API: POST /lists {name: "Work Tasks"}
    API->>DB: INSERT INTO lists
    DB-->>API: list_id: 123
    API-->>App: {id: 123, name: "Work Tasks"}
    
    App->>API: POST /lists/123/tasks {title: "Review PR"}
    API->>DB: INSERT INTO tasks
    DB-->>API: task_id: 456
    API-->>App: {id: 456, title: "Review PR", list_id: 123}
    
    App->>API: PUT /tasks/456 {completed: true}
    API->>DB: UPDATE tasks SET completed = true
    API-->>App: {id: 456, completed: true}
```

#### Key Discussion Points
1. **User authentication:** Session vs JWT tokens
2. **Task organization:** Lists vs tags vs categories
3. **Data relationships:** Foreign keys and referential integrity
4. **Mobile sync:** How to handle offline usage

---

### Question 3: Simple Blogging Platform
**Difficulty:** ⭐⭐⭐☆☆
**Time:** 50 minutes
**Category:** B - Content & File Management

#### Problem Statement
Design a basic blogging platform where users can:
- Create accounts and profiles
- Write and publish blog posts
- Add comments to posts
- Follow other users
- View a personalized feed

#### System Architecture
```mermaid
graph TB
    subgraph "Frontend Layer"
        Web[Web Application<br/>React/Vue]
        Mobile[Mobile App<br/>Optional]
    end
    
    subgraph "Backend Layer"
        Auth[Authentication Service]
        Posts[Posts Service]
        Users[Users Service]
        Feed[Feed Generation Service]
    end
    
    subgraph "Data Layer"
        UserDB[(Users Database)]
        PostDB[(Posts Database)]
        Cache[Redis Cache<br/>Feed Cache]
    end
    
    Web --> Auth
    Web --> Posts
    Web --> Users
    Web --> Feed
    
    Auth --> UserDB
    Posts --> PostDB
    Users --> UserDB
    Feed --> PostDB
    Feed --> Cache
```

#### Database Schema
```mermaid
erDiagram
    USERS {
        bigint id PK
        string username UK
        string email UK
        string password_hash
        string bio
        string avatar_url
        timestamp created_at
    }
    
    POSTS {
        bigint id PK
        bigint author_id FK
        string title
        text content
        string status
        timestamp published_at
        timestamp created_at
        timestamp updated_at
    }
    
    COMMENTS {
        bigint id PK
        bigint post_id FK
        bigint author_id FK
        text content
        timestamp created_at
    }
    
    FOLLOWS {
        bigint id PK
        bigint follower_id FK
        bigint following_id FK
        timestamp created_at
    }
    
    USERS ||--o{ POSTS : "authors"
    USERS ||--o{ COMMENTS : "writes"
    POSTS ||--o{ COMMENTS : "has"
    USERS ||--o{ FOLLOWS : "follower"
    USERS ||--o{ FOLLOWS : "following"
```

#### Feed Generation Strategy
```mermaid
flowchart TD
    A[User Requests Feed] --> B{User Following Count}
    
    B -->|< 100 following| C[Pull Model<br/>Query recent posts from following]
    B -->|> 100 following| D[Hybrid Model<br/>Pre-computed + recent]
    
    C --> E[Sort by timestamp]
    D --> F[Merge cached + fresh posts]
    
    E --> G[Return paginated results]
    F --> G
```

#### Key Discussion Points
1. **Feed generation:** Pull vs push models for different user types
2. **Content storage:** How to handle rich text and images
3. **User relationships:** Following/followers implementation
4. **Performance:** Caching strategies for popular posts

---

## Intermediate Questions

### Question 4: Image Sharing Platform (Like Instagram Basic)
**Difficulty:** ⭐⭐⭐☆☆
**Time:** 55 minutes
**Category:** B - Content & File Management

#### Problem Statement
Design an image sharing platform where users can:
- Upload and share photos with captions
- Like and comment on photos
- Follow other users
- View a timeline feed
- Basic image processing (resize, thumbnails)

#### High-Level Architecture
```mermaid
graph TB
    subgraph "Client Layer"
        Mobile[Mobile Apps<br/>iOS/Android]
        Web[Web Application]
    end
    
    subgraph "API Gateway"
        Gateway[API Gateway<br/>Rate Limiting, Auth]
    end
    
    subgraph "Application Services"
        Upload[Image Upload Service]
        Social[Social Service<br/>Likes, Comments, Follows]
        Feed[Feed Service]
        User[User Service]
    end
    
    subgraph "Storage Layer"
        FileStore[File Storage<br/>AWS S3 / Local Storage]
        Database[(PostgreSQL<br/>Metadata)]
        Cache[Redis<br/>Feed Cache]
    end
    
    subgraph "Processing"
        ImageProcessor[Image Processing<br/>Resize, Thumbnails]
        Queue[Message Queue<br/>Async Processing]
    end
    
    Mobile --> Gateway
    Web --> Gateway
    Gateway --> Upload
    Gateway --> Social
    Gateway --> Feed
    Gateway --> User
    
    Upload --> FileStore
    Upload --> Queue
    Queue --> ImageProcessor
    ImageProcessor --> FileStore
    
    Social --> Database
    Feed --> Database
    Feed --> Cache
    User --> Database
```

#### Image Upload Flow
```mermaid
sequenceDiagram
    participant Client
    participant API
    participant Storage
    participant Queue
    participant Processor
    
    Client->>API: POST /upload (image file)
    API->>Storage: Store original image
    Storage-->>API: File URL
    API->>Queue: Queue processing job
    API-->>Client: Upload successful + image_id
    
    Queue->>Processor: Process image
    Processor->>Storage: Generate thumbnails
    Processor->>Storage: Generate different sizes
    Processor-->>Queue: Processing complete
```

#### Key Discussion Points
1. **File storage:** Local vs cloud storage trade-offs
2. **Image processing:** Synchronous vs asynchronous processing
3. **CDN usage:** Serving images globally
4. **Database vs file system:** Metadata storage strategy

---

### Question 5: Real-time Chat Application
**Difficulty:** ⭐⭐⭐☆☆
**Time:** 55 minutes
**Category:** C - Real-time Features

#### Problem Statement
Design a real-time messaging application where users can:
- Send and receive messages instantly
- Create group chats
- See online/offline status
- Message history and search
- File sharing capabilities

#### Real-time Architecture
```mermaid
graph TB
    subgraph "Client Layer"
        App1[User A<br/>Mobile App]
        App2[User B<br/>Web App]
        App3[User C<br/>Desktop]
    end
    
    subgraph "Connection Layer"
        WS[WebSocket Server<br/>Socket.IO]
        LB[Load Balancer<br/>Sticky Sessions]
    end
    
    subgraph "Application Layer"
        Chat[Chat Service]
        Presence[Presence Service<br/>Online Status]
        Notification[Push Notifications]
    end
    
    subgraph "Message Flow"
        Queue[Message Queue<br/>Redis Pub/Sub]
        History[Message History]
    end
    
    subgraph "Storage"
        ChatDB[(Chat Database<br/>Messages, Rooms)]
        FileStore[File Storage<br/>Shared Files]
    end
    
    App1 --> LB
    App2 --> LB
    App3 --> LB
    LB --> WS
    
    WS --> Chat
    WS --> Presence
    Chat --> Queue
    Queue --> WS
    Chat --> History
    History --> ChatDB
```

#### Message Flow Sequence
```mermaid
sequenceDiagram
    participant A as User A
    participant WS1 as WebSocket Server
    participant Chat as Chat Service
    participant Queue as Message Queue
    participant WS2 as WebSocket Server
    participant B as User B
    
    A->>WS1: Send message
    WS1->>Chat: Process message
    Chat->>Queue: Publish to room
    Chat->>Chat: Store in database
    
    Queue->>WS1: Message for connected users
    Queue->>WS2: Message for connected users
    
    WS1->>A: Message confirmation
    WS2->>B: New message received
```

#### Key Discussion Points
1. **Real-time communication:** WebSockets vs polling vs Server-Sent Events
2. **Message delivery:** At-least-once vs exactly-once delivery
3. **Scaling connections:** Handling thousands of concurrent connections
4. **Message persistence:** Storage and retrieval of chat history

---

### Question 6: Simple E-commerce System
**Difficulty:** ⭐⭐⭐⭐☆
**Time:** 60 minutes
**Category:** D - E-commerce Basics

#### Problem Statement
Design a basic e-commerce platform where customers can:
- Browse products and categories
- Add items to shopping cart
- Complete purchases
- View order history
- Basic inventory management

#### E-commerce Architecture
```mermaid
graph TB
    subgraph "Frontend"
        Web[Web Store]
        Admin[Admin Panel]
    end
    
    subgraph "Core Services"
        Catalog[Product Catalog Service]
        Cart[Shopping Cart Service]
        Order[Order Processing Service]
        Payment[Payment Service]
        Inventory[Inventory Service]
    end
    
    subgraph "External Services"
        PaymentGW[Payment Gateway<br/>Stripe/PayPal]
        Shipping[Shipping APIs]
        Email[Email Service]
    end
    
    subgraph "Data Layer"
        ProductDB[(Product Database)]
        OrderDB[(Order Database)]
        UserDB[(User Database)]
        Cache[Redis Cache<br/>Cart, Sessions]
    end
    
    Web --> Catalog
    Web --> Cart
    Web --> Order
    Admin --> Inventory
    
    Order --> Payment
    Payment --> PaymentGW
    Order --> Inventory
    Order --> Email
    
    Catalog --> ProductDB
    Cart --> Cache
    Order --> OrderDB
    Inventory --> ProductDB
```

#### Order Processing Flow
```mermaid
sequenceDiagram
    participant Customer
    participant Cart
    participant Order
    participant Inventory
    participant Payment
    participant Gateway
    
    Customer->>Cart: Add items to cart
    Customer->>Order: Initiate checkout
    Order->>Inventory: Reserve items
    
    alt Items available
        Inventory-->>Order: Items reserved
        Order->>Payment: Process payment
        Payment->>Gateway: Charge customer
        
        alt Payment successful
            Gateway-->>Payment: Payment confirmed
            Payment-->>Order: Payment successful
            Order->>Inventory: Confirm reservation
            Order-->>Customer: Order confirmed
        else Payment failed
            Gateway-->>Payment: Payment failed
            Payment-->>Order: Payment failed
            Order->>Inventory: Release reservation
            Order-->>Customer: Order failed
        end
    else Items unavailable
        Inventory-->>Order: Insufficient stock
        Order-->>Customer: Items unavailable
    end
```

#### Key Discussion Points
1. **Inventory management:** How to handle concurrent purchases
2. **Payment processing:** PCI compliance and security
3. **Transaction consistency:** Ensuring data integrity across services
4. **Cart persistence:** Session-based vs user-based carts

---

## Bonus Challenge Questions

### Question 7: Event Booking System
**Difficulty:** ⭐⭐⭐⭐☆
**Time:** 60 minutes

Design a system for booking events (concerts, conferences, etc.) with:
- Event creation and management
- Ticket sales with seat selection
- Payment processing
- Waitlist management
- QR code ticket generation

### Question 8: Simple Social Media Feed
**Difficulty:** ⭐⭐⭐⭐☆
**Time:** 60 minutes

Design a basic social media platform with:
- User posts (text, images)
- Following/followers system
- Timeline generation
- Like and comment functionality
- Basic content moderation

### Question 9: File Storage and Sharing Service
**Difficulty:** ⭐⭐⭐☆☆
**Time:** 55 minutes

Design a file storage service (like Google Drive basic) with:
- File upload/download
- Folder organization
- File sharing with permissions
- Version control basics
- Storage optimization

---

## Question Templates & Frameworks

### Universal Question Framework
```mermaid
flowchart TD
    A[Problem Statement] --> B[Requirements Clarification]
    B --> C[Core Entities Identification]
    C --> D[Database Schema Design]
    D --> E[API Design]
    E --> F[Basic Architecture]
    F --> G[Simple Scaling Considerations]
    
    subgraph "Time Allocation"
        B1[5 minutes] -.-> B
        C1[10 minutes] -.-> C
        D1[15 minutes] -.-> D
        E1[10 minutes] -.-> E
        F1[10 minutes] -.-> F
        G1[5 minutes] -.-> G
    end
```

### Requirements Clarification Template
Always ask these questions:
1. **Scale:** How many users? How much data?
2. **Features:** What are the core vs nice-to-have features?
3. **Platform:** Web, mobile, or both?
4. **Authentication:** Do users need accounts?
5. **Real-time:** Any real-time features needed?
6. **Constraints:** Any technology preferences or limitations?

### Database Design Checklist
```mermaid
mindmap
    root((Database Design))
        Entities
            Core Business Objects
            User Management
            Relationships
        Schema
            Primary Keys
            Foreign Keys
            Indexes
            Data Types
        Constraints
            Unique Constraints
            Not Null
            Check Constraints
        Normalization
            1NF, 2NF, 3NF
            When to Denormalize
```

---

## Evaluation Criteria

### Junior Level Success Metrics

#### Essential Skills (Must Demonstrate)
- [ ] Can design basic database schema with proper relationships
- [ ] Understands REST API principles and HTTP methods
- [ ] Identifies major system components (frontend, backend, database)
- [ ] Considers basic user authentication
- [ ] Thinks about data validation

#### Good-to-Have Skills
- [ ] Considers basic performance optimizations (indexing, caching)
- [ ] Thinks about error handling
- [ ] Considers security basics (password hashing, input validation)
- [ ] Shows awareness of scalability concepts
- [ ] Can explain trade-offs in simple terms

#### Red Flags
- [ ] Cannot design basic database relationships
- [ ] No understanding of REST APIs
- [ ] Doesn't consider user authentication
- [ ] No awareness of data persistence
- [ ] Cannot explain their design choices

### Self-Assessment Questions
After each practice session, ask yourself:

1. **Requirements:** Did I ask clarifying questions about scale and features?
2. **Data Model:** Are my database relationships correct and normalized?
3. **API Design:** Are my endpoints RESTful and logically organized?
4. **Architecture:** Can I explain how data flows through my system?
5. **Trade-offs:** Did I consider alternatives and explain my choices?

---

## Progression to Senior Level

### When You're Ready for Senior Questions
You should consistently demonstrate these abilities across multiple junior questions:

#### Technical Readiness
- [ ] **Database Mastery:** Comfortable with complex schemas, joins, and basic optimization
- [ ] **API Design:** Can design comprehensive REST APIs with proper HTTP methods
- [ ] **Architecture Thinking:** Naturally separates concerns and identifies service boundaries
- [ ] **Basic Scaling:** Understands concepts like caching, read replicas, and load balancing
- [ ] **Trade-off Analysis:** Can compare options and explain decisions clearly

#### Process Readiness
- [ ] **Structured Approach:** Follows consistent methodology for solving problems
- [ ] **Communication:** Explains complex concepts clearly and handles questions well
- [ ] **Time Management:** Completes designs within time limits
- [ ] **Evolution Thinking:** Considers how systems might need to change over time

### Bridge Topics to Study
Before moving to senior questions, strengthen these areas:

```mermaid
graph LR
    subgraph "Junior Topics"
        A[Database Design]
        B[REST APIs]
        C[Basic Architecture]
    end
    
    subgraph "Bridge Topics"
        D[Caching Strategies]
        E[Database Scaling]
        F[Message Queues]
        G[Microservices Basics]
    end
    
    subgraph "Senior Topics"
        H[Distributed Systems]
        I[High Availability]
        J[Global Scale]
    end
    
    A --> D
    B --> F
    C --> G
    
    D --> H
    E --> I
    F --> J
    G --> H
```

### Recommended Study Path
1. **Master all essential junior questions** (Questions 1-3)
2. **Complete intermediate questions** (Questions 4-6)
3. **Study bridge topics:**
   - Redis caching patterns
   - Database read replicas
   - Basic message queues (Redis pub/sub)
   - API gateway concepts
4. **Attempt bonus challenge questions**
5. **Move to [Senior Questions](./senior-questions.md)**

### Practice Schedule Recommendation
- **Week 1-2:** Essential questions (1-3), repeat until comfortable
- **Week 3-4:** Intermediate questions (4-6), focus on trade-offs
- **Week 5:** Bonus questions and review weak areas
- **Week 6:** Mock interviews and transition to senior level

---

*Version: 1.0*
*Owner: @codewithmunyao*
*Related Documents:*
- *[Senior Practice Questions](./senior-questions.md)*
- *[Junior Interview Guide](../engineer-interview-guide/junior-guide.md)*
- *[Trade-offs Decision Guide](../estimation-techniques/trade-offs-decisions.md)*