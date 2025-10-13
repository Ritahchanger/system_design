# Introduction & Architecture

## What are Frontend Microservices?

Frontend microservices (also known as Micro Frontends) is an architectural pattern that extends the microservices concept to frontend development. It involves breaking down a monolithic frontend application into smaller, more manageable pieces that can be developed, tested, and deployed independently.

## Core Principles

### 1. **Be Technology Agnostic**
Teams should be free to choose their own technology stack without being forced into framework decisions made by other teams.

### 2. **Isolate Team Code**
Don't share a runtime between teams. Build independent apps that are self-contained.

### 3. **Establish Team Prefixes**
Agree on naming conventions where isolation is not possible. Namespace CSS, events, and local storage.

### 4. **Favor Native Browser Features**
Use browser APIs over building custom APIs. Use events for communication, not building a shared library.

### 5. **Build a Resilient Site**
Your feature should be useful even if JavaScript fails or hasn't executed yet.

## Architecture Patterns

### 1. Horizontal Split (Page-Based)
In a horizontal split, the application is divided by pages or routes.
Each micro frontend owns one or more entire pages, rather than sharing the same view or layout area.
```mermaid
graph TB
    subgraph "Horizontal Split Architecture"
        Router[Client-Side Router]
        
        subgraph "Pages/Routes"
            Home[Home Page<br/>Team A]
            Products[Products Page<br/>Team B]
            Cart[Cart Page<br/>Team C]
            Profile[Profile Page<br/>Team D]
        end
        
        Router --> Home
        Router --> Products
        Router --> Cart
        Router --> Profile
        
        subgraph "Shared"
            Shell[Shell/Container]
            DS[Design System]
        end
        
        Home --> DS
        Products --> DS
        Cart --> DS
        Profile --> DS
    end
```

### 2. Vertical Split (Feature-Based)
In a vertical split, a single page is composed of multiple micro frontends (components) stacked vertically or in layout sections.
Each MFE owns a feature or a UI fragment rather than an entire page.
```mermaid
graph TB
    subgraph "Vertical Split Architecture"
        Shell[Shell Application]
        
        subgraph "Feature Areas"
            Header[Header & Navigation<br/>Team A]
            Sidebar[Sidebar<br/>Team B]
            Content[Main Content<br/>Team C]
            Footer[Footer<br/>Team D]
        end
        
        Shell --> Header
        Shell --> Sidebar
        Shell --> Content
        Shell --> Footer
        
        subgraph "Cross-Cutting Concerns"
            Auth[Authentication]
            Analytics[Analytics]
            ErrorHandling[Error Handling]
        end
        
        Header --> Auth
        Content --> Analytics
        Sidebar --> ErrorHandling
    end
```

### 3. Hybrid Split
A hybrid split combines:

Horizontal split → Entire pages handled by separate MFEs (page-level ownership).

Vertical split → Individual components or sections on those pages handled by different MFEs (component-level ownership).

Essentially, some MFEs own full pages, while others provide smaller components integrated into those pages.
```mermaid
graph TB
    subgraph "Hybrid Architecture"
        subgraph "Page Level (Horizontal)"
            HomePage[Home Page]
            ProductPage[Product Page]
            CheckoutPage[Checkout Page]
        end
        
        subgraph "Component Level (Vertical)"
            CommonHeader[Shared Header]
            ProductList[Product List Widget]
            ShoppingCart[Cart Widget]
            UserProfile[User Profile Widget]
        end
        
        HomePage --> CommonHeader
        HomePage --> ProductList
        ProductPage --> CommonHeader
        ProductPage --> ShoppingCart
        CheckoutPage --> UserProfile
        CheckoutPage --> ShoppingCart
    end
```

## Integration Approaches

### Build-Time Integration
In Build-Time Integration, all micro frontends are compiled together during build into a single deployable bundle.

Essentially, it’s a monolithic assembly at build-time, even if development is modular.

The shell app and all MFEs are compiled and bundled together.
```mermaid
flowchart LR
    subgraph "Build-Time Integration"
        A[Source Code] --> B[Build Process]
        B --> C[Static Bundle]
        C --> D[Single Deployment]
        
        subgraph "Pros"
            E[Simple Deployment]
            F[Better Performance]
            G[Type Safety]
        end
        
        subgraph "Cons"
            H[Tight Coupling]
            I[Coordinated Releases]
            J[Technology Lock-in]
        end
    end
```

**Use Cases:**
- Small teams
- Shared technology stack
- Performance-critical applications
- Simple deployment requirements

### Run-Time Integration
In Runtime Integration, micro frontends are loaded dynamically in the browser at runtime rather than being bundled together at build time.

This allows each MFE to be developed, deployed, and updated independently.

Often paired with Module Federation, Single-SPA, or import maps.
```mermaid
flowchart LR
    subgraph "Run-Time Integration"
        A[Independent Bundles] --> B[Runtime Composition]
        B --> C[Dynamic Loading]
        C --> D[Independent Deployment]
        
        subgraph "Pros"
            E[True Independence]
            F[Technology Diversity]
            G[Independent Deployment]
        end
        
        subgraph "Cons"
            H[Network Overhead]
            I[Runtime Complexity]
            J[Coordination Challenges]
        end
    end
```

**Use Cases:**
- Large teams
- Different technology requirements
- Independent deployment needs
- Complex business domains

## Communication Patterns

### 1. Event-Driven Communication

```mermaid
sequenceDiagram
    participant MF1 as Microservice A
    participant EventBus as Event Bus
    participant MF2 as Microservice B
    participant MF3 as Microservice C
    
    MF1->>EventBus: Publish "user-login" event
    EventBus->>MF2: Notify subscriber
    EventBus->>MF3: Notify subscriber
    MF2->>EventBus: Publish "cart-updated" event
    EventBus->>MF1: Notify subscriber
```

### 2. Shared State Management

```mermaid
graph TB
    subgraph "Shared State Pattern"
        Store[Central State Store]
        
        subgraph "Microservices"
            MF1[Header Service]
            MF2[Product Service]
            MF3[Cart Service]
        end
        
        MF1 <--> Store
        MF2 <--> Store
        MF3 <--> Store
        
        subgraph "State Slices"
            UserState[User State]
            ProductState[Product State]
            CartState[Cart State]
        end
        
        Store --> UserState
        Store --> ProductState
        Store --> CartState
    end
```

### 3. URL-Based Communication

```mermaid
graph LR
    subgraph "URL-Based Communication"
        A[Route Change] --> B[URL Update]
        B --> C[Route Detection]
        C --> D[Component Update]
        
        subgraph "Examples"
            E["/products/123"] --> F["Product Detail MF"]
            G["/cart"] --> H["Cart MF"]
            I["/profile"] --> J["Profile MF"]
        end
    end
```

## Team Organization Models

### 1. Cross-Functional Teams

```mermaid
graph TB
    subgraph "Cross-Functional Team Structure"
        subgraph "Team A - E-commerce Core"
            A1[Frontend Developer]
            A2[Backend Developer]
            A3[Designer]
            A4[Product Owner]
            A5[QA Engineer]
        end
        
        subgraph "Team B - User Management"
            B1[Frontend Developer]
            B2[Backend Developer]
            B3[Security Engineer]
            B4[Product Owner]
            B5[QA Engineer]
        end
        
        subgraph "Shared Services"
            C1[Platform Team]
            C2[Design System Team]
            C3[DevOps Team]
        end
        
        A1 --> C2
        B1 --> C2
        A2 --> C3
        B2 --> C3
    end
```

### 2. Domain-Driven Teams

```mermaid
graph TB
    subgraph "Domain-Driven Organization"
        subgraph "Customer Domain"
            D1[User Registration MF]
            D2[User Profile MF]
            D3[Authentication MF]
        end
        
        subgraph "Product Domain"
            D4[Product Catalog MF]
            D5[Search MF]
            D6[Recommendations MF]
        end
        
        subgraph "Order Domain"
            D7[Shopping Cart MF]
            D8[Checkout MF]
            D9[Order History MF]
        end
        
        subgraph "Shared Kernel"
            D10[Design System]
            D11[Common Utils]
            D12[Authentication Service]
        end
        
        D1 --> D12
        D2 --> D10
        D4 --> D10
        D7 --> D11
    end
```

## Benefits & Trade-offs

### Benefits

| Benefit | Description | Impact |
|---------|-------------|---------|
| **Independent Development** | Teams work autonomously | Faster development cycles |
| **Technology Diversity** | Choose best tools for the job | Innovation and flexibility |
| **Fault Isolation** | Failures don't cascade | Better reliability |
| **Scalable Teams** | Add teams without coordination | Organizational scalability |
| **Incremental Migration** | Modernize piece by piece | Reduced risk |

### Trade-offs

| Challenge | Description | Mitigation |
|-----------|-------------|------------|
| **Complexity** | More moving parts | Good tooling and practices |
| **Performance** | Network overhead | Caching and optimization |
| **Consistency** | UI/UX fragmentation | Design system and guidelines |
| **Testing** | Integration complexity | Contract testing and E2E |
| **Governance** | Coordination overhead | Clear standards and tooling |

## Decision Framework

### When to Choose Micro Frontends

```mermaid
flowchart TD
    A[Evaluate Your Context] --> B{Large Team?}
    B -->|Yes| C{Independent Deployment Needed?}
    B -->|No| D[Consider Monolith]
    
    C -->|Yes| E{Different Tech Requirements?}
    C -->|No| F[Consider Monorepo]
    
    E -->|Yes| G[✅ Micro Frontends]
    E -->|No| H{Organizational Benefits?}
    
    H -->|Yes| G
    H -->|No| F
    
    D --> I[Single Repo, Single Deploy]
    F --> J[Single Repo, Multiple Apps]
    G --> K[Multiple Repos, Independent Deploy]
```

### Assessment Criteria

Rate each criterion from 1-5 (5 being highest need/complexity):

- **Team Size**: Number of developers working on frontend
- **Team Distribution**: Geographic and organizational spread
- **Technology Diversity**: Need for different frameworks/tools
- **Domain Complexity**: Business domain complexity
- **Deployment Independence**: Need for separate release cycles
- **Performance Requirements**: Strict performance budgets
- **Legacy Integration**: Existing system constraints

**Score Interpretation:**
- **20-25**: Strong candidate for micro frontends
- **15-19**: Consider hybrid approaches
- **10-14**: Evaluate monorepo solutions
- **< 10**: Stick with monolithic frontend

## Getting Started Checklist

- [ ] **Assess your context** using the decision framework
- [ ] **Define boundaries** based on business domains
- [ ] **Choose integration approach** (build-time vs runtime)
- [ ] **Select technology stack** and tooling
- [ ] **Establish team structure** and ownership
- [ ] **Create governance guidelines** and standards
- [ ] **Set up development environment** and workflow
- [ ] **Implement monitoring** and observability
- [ ] **Plan migration strategy** if coming from monolith

## Next Steps

1. **[Set up your Monorepo](02-monorepo-setup.md)** - Configure development environment
2. **[Development Architecture](03-development-architecture.md)** - Local development setup
3. **[Runtime Integration](04-runtime-integration.md)** - Module federation and communication

---

[← Back to Main](../README.md) | [Next: Monorepo Setup →](02-monorepo-setup.md)