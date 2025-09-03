## Runtime Integration Patterns {#runtime-integration}

### 1. Build-Time Integration

```mermaid
flowchart TD
    A[Source Code] --> B[Build Process]
    B --> C[Bundle Creation]
    C --> D[Static Assets]
    D --> E[CDN Deployment]
    
    subgraph "Build-Time Integration"
        F[NPM Packages]
        G[Git Submodules]
        H[Shared Libraries]
    end
    
    B --> F
    B --> G
    B --> H
```

### 2. Runtime Integration

```mermaid
flowchart TD
    subgraph "Runtime Integration Methods"
        A[Module Federation] --> A1[Webpack 5]
        A --> A2[Dynamic Imports]
        
        B[Single-SPA] --> B1[Framework Agnostic]
        B --> B2[Lifecycle Management]
        
        C[Web Components] --> C1[Native Standards]
        C --> C2[Shadow DOM]
        
        D[Server-Side Includes] --> D1[Edge Side Includes]
        D --> D2[Template Composition]
    end
    
    A1 --> E[Production App]
    B1 --> E
    C1 --> E
    D1 --> E
```

### Module Federation Implementation

```typescript
// Shell application component
import React, { Suspense } from 'react';

const HeaderService = React.lazy(() => import('headerService/Header'));
const ProductCatalog = React.lazy(() => import('productCatalog/ProductList'));

function App() {
  return (
    <div className="app">
      <Suspense fallback={<div>Loading header...</div>}>
        <HeaderService />
      </Suspense>
      
      <main>
        <Suspense fallback={<div>Loading products...</div>}>
          <ProductCatalog />
        </Suspense>
      </main>
    </div>
  );
}

export default App;
```

### State Management Across Microservices

```mermaid
graph TB
    subgraph "State Management Strategies"
        A[Local State] --> A1[Component State]
        A --> A2[Service State]
        
        B[Shared State] --> B1[Event Bus]
        B --> B2[State Store]
        B --> B3[URL State]
        
        C[External State] --> C1[APIs]
        C --> C2[Databases]
        C --> C3[Browser Storage]
    end
    
    subgraph "Implementation"
        D[Microservice 1] --> E[Shared Store]
        F[Microservice 2] --> E
        G[Microservice 3] --> E
        E --> H[State Synchronization]
        H --> I[UI Updates]
    end
```



---

For more information, see:
- [Cross framework](05-cross-framework.md)
- [Deployment strategies](06-deployment-strategies.md)
- [Performance & Monitoring](07-performance-monitoring.md)