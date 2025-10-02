# Horizontal Scaling for API Gateways

## Overview

Horizontal scaling (scaling out) involves adding more instances of API gateway servers to distribute load across multiple machines. Instead of making one server more powerful, you deploy multiple identical servers that work together to handle incoming traffic.

## Theory and Concepts

### What is Horizontal Scaling?

Horizontal scaling distributes workload by:
- **Adding more instances**: Deploy additional gateway servers
- **Load distribution**: Traffic is spread across all instances
- **Independent operation**: Each instance operates autonomously
- **Shared-nothing architecture**: Instances don't share memory or local state

```mermaid
graph TD
    A[Client Requests] --> B[Load Balancer]
    B --> C[Gateway Instance 1]
    B --> D[Gateway Instance 2]
    B --> E[Gateway Instance 3]
    B --> F[Gateway Instance N]
    
    C --> G[Backend Services]
    D --> G
    E --> G
    F --> G
    
    style B fill:#f9f,stroke:#333,stroke-width:4px
```

### Core Principles

1. **Statelessness**: Each request can be handled by any instance
2. **Redundancy**: Multiple instances provide fault tolerance
3. **Elasticity**: Add or remove instances based on demand
4. **Linear scalability**: Performance scales proportionally with instances

## When to Use Horizontal Scaling

Horizontal scaling is ideal when:
- **High availability is critical**: No single point of failure
- **Traffic is unpredictable**: Can scale dynamically
- **Growth is unlimited**: No hardware ceiling
- **Global distribution**: Deploy instances in multiple regions
- **Cost optimization at scale**: Commodity hardware is cost-effective

### Advantages Over Vertical Scaling

```mermaid
graph LR
    A[Horizontal Scaling Benefits] --> B[Unlimited Growth]
    A --> C[High Availability]
    A --> D[Fault Tolerance]
    A --> E[Rolling Updates]
    A --> F[Geographic Distribution]
    
    B --> B1[Add infinite instances]
    C --> C1[No single point of failure]
    D --> D1[Instance failure doesn't kill system]
    E --> E1[Zero downtime deployments]
    F --> F1[Reduced latency globally]
```

## Load Distribution Strategies

### 1. Round Robin

Distributes requests sequentially across instances.

```mermaid
graph TD
    A[Request 1] --> B[Load Balancer]
    C[Request 2] --> B
    D[Request 3] --> B
    E[Request 4] --> B
    
    B -->|Round Robin| F[Instance 1]
    B --> G[Instance 2]
    B --> H[Instance 3]
    
    A -.->|Routes to| F
    C -.->|Routes to| G
    D -.->|Routes to| H
    E -.->|Routes to| F
```

**Use Case**: Simple, works well when all instances and requests are similar.

### 2. Least Connections

Routes to the instance with fewest active connections.

```javascript
// Conceptual pattern - not production code
function selectInstance(instances) {
  return instances.reduce((least, current) => {
    return current.activeConnections < least.activeConnections 
      ? current 
      : least;
  });
}
```

**Use Case**: Long-lived connections or varying request processing times.

### 3. Weighted Distribution

Assigns more traffic to more powerful instances.

```mermaid
graph TD
    A[Load Balancer] --> B[Instance 1<br/>Weight: 3]
    A --> C[Instance 2<br/>Weight: 2]
    A --> D[Instance 3<br/>Weight: 1]
    
    A -.->|50% traffic| B
    A -.->|33% traffic| C
    A -.->|17% traffic| D
```

**Use Case**: Mixed instance sizes or gradual migration scenarios.

### 4. IP Hash / Session Affinity

Routes based on client IP to maintain sticky sessions.

```javascript
// Conceptual pattern
function selectInstanceByIP(clientIP, instances) {
  const hash = hashFunction(clientIP);
  const index = hash % instances.length;
  return instances[index];
}
```

**Use Case**: When sessions must be maintained (though prefer stateless - see [patterns.md](./patterns.md)).

### 5. Geographic Routing

Routes to nearest instance based on client location.

```mermaid
graph TD
    A[Global Load Balancer] --> B[US East Region]
    A --> C[EU West Region]
    A --> D[Asia Pacific Region]
    
    B --> B1[Instance Pool 1]
    C --> C1[Instance Pool 2]
    D --> D1[Instance Pool 3]
    
    E[Client in NYC] -.->|Routed to| B
    F[Client in London] -.->|Routed to| C
    G[Client in Tokyo] -.->|Routed to| D
```

**Use Case**: Global applications requiring low latency (see [architecture.md](./architecture.md)).

## Scaling Patterns

### Auto-Scaling

Automatically adjust instance count based on metrics.

```mermaid
graph TD
    A[Monitoring System] --> B{Check Metrics}
    B -->|CPU > 70%| C[Scale Up]
    B -->|CPU < 30%| D[Scale Down]
    B -->|Within Range| E[No Action]
    
    C --> F[Add Instance]
    D --> G[Remove Instance]
    
    F --> H[Update Load Balancer]
    G --> H
    
    H --> I[Continue Monitoring]
    I --> A
```

**Scaling Triggers**:
- CPU utilization
- Memory usage
- Request rate
- Response time
- Queue depth

### Predictive Scaling

Scale proactively based on historical patterns.

```javascript
// Conceptual pattern
function calculateRequiredInstances(timeOfDay, dayOfWeek, historicalData) {
  const predictedTraffic = historicalData
    .filter(d => d.timeOfDay === timeOfDay && d.dayOfWeek === dayOfWeek)
    .reduce((sum, d) => sum + d.traffic, 0) / historicalData.length;
  
  return Math.ceil(predictedTraffic / REQUESTS_PER_INSTANCE);
}
```

**Use Case**: Predictable traffic patterns (e.g., business hours, seasonal events).

### Manual Scaling

Deliberately add instances for known events.

```mermaid
graph LR
    A[Normal Load<br/>5 instances] --> B[Event Prep<br/>20 instances]
    B --> C[During Event<br/>50 instances]
    C --> D[Post-Event<br/>10 instances]
    D --> E[Normal Load<br/>5 instances]
```

**Use Case**: Product launches, Black Friday, scheduled maintenance.

## State Management Challenges

### The Stateless Imperative

```mermaid
graph TD
    A[Stateless Gateway Design] --> B[Session Data]
    A --> C[Rate Limiting]
    A --> D[Caching]
    
    B --> B1[External Session Store<br/>Redis/Memcached]
    C --> C1[Distributed Rate Limiter<br/>Redis with Lua]
    D --> D1[Shared Cache Layer<br/>Redis/Memcached]
    
    style B1 fill:#ccffcc
    style C1 fill:#ccffcc
    style D1 fill:#ccffcc
```

**Key Principle**: Any state must be externalized so any instance can handle any request.

### External Session Storage

```javascript
// Conceptual pattern for session management
class SessionManager {
  constructor(redisClient) {
    this.redis = redisClient;
  }
  
  async getSession(sessionId) {
    return await this.redis.get(`session:${sessionId}`);
  }
  
  async setSession(sessionId, data, ttl) {
    await this.redis.setex(`session:${sessionId}`, ttl, JSON.stringify(data));
  }
}
```

**Theory**: Session data stored in Redis is accessible from any gateway instance.

### Distributed Caching

For caching strategies that benefit horizontal scaling, see [caching.md](./caching.md).

```mermaid
graph TD
    A[Gateway Instance 1] --> D[Shared Redis Cache]
    B[Gateway Instance 2] --> D
    C[Gateway Instance 3] --> D
    
    D --> E[Cache Hit]
    D --> F[Cache Miss]
    
    F --> G[Fetch from Backend]
    G --> H[Store in Cache]
```

### Distributed Rate Limiting

```javascript
// Conceptual pattern using Redis
async function checkRateLimit(clientId, limit, window) {
  const key = `rate:${clientId}:${Math.floor(Date.now() / window)}`;
  const current = await redis.incr(key);
  
  if (current === 1) {
    await redis.expire(key, window);
  }
  
  return current <= limit;
}
```

See [security.md](./security.md) for comprehensive rate limiting strategies.

## Health Checks and Load Balancer Integration

### Health Check Types

```mermaid
graph TD
    A[Load Balancer] --> B[Active Health Checks]
    A --> C[Passive Health Checks]
    
    B --> B1[Periodic HTTP Ping]
    B --> B2[Deep Health Endpoint]
    B --> B3[Dependency Checks]
    
    C --> C1[Monitor Response Codes]
    C --> C2[Track Response Times]
    C --> C3[Connection Failures]
    
    B1 --> D{Healthy?}
    B2 --> D
    B3 --> D
    C1 --> D
    C2 --> D
    C3 --> D
    
    D -->|Yes| E[Keep in Pool]
    D -->|No| F[Remove from Pool]
```

### Health Endpoint Design

```javascript
// Conceptual health check endpoint
app.get('/health', async (req, res) => {
  const checks = {
    server: true,
    redis: await checkRedis(),
    database: await checkDatabase(),
    backend: await checkBackend()
  };
  
  const healthy = Object.values(checks).every(c => c === true);
  
  res.status(healthy ? 200 : 503).json({
    status: healthy ? 'healthy' : 'unhealthy',
    checks
  });
});
```

**Theory**: Load balancer removes unhealthy instances, preventing cascading failures.

## Deployment Strategies

### Rolling Updates

Update instances gradually to maintain availability.

```mermaid
graph LR
    A[10 Instances v1.0] --> B[8 v1.0 + 2 v1.1]
    B --> C[6 v1.0 + 4 v1.1]
    C --> D[4 v1.0 + 6 v1.1]
    D --> E[2 v1.0 + 8 v1.1]
    E --> F[10 Instances v1.1]
    
    style A fill:#ffcccc
    style F fill:#ccffcc
```

**Benefits**: Zero downtime, gradual rollout, easy rollback.

### Blue-Green Deployment

Maintain two identical environments.

```mermaid
graph TD
    A[Load Balancer] --> B[Blue Environment<br/>v1.0 - ACTIVE]
    A -.->|Not routing| C[Green Environment<br/>v1.1 - STANDBY]
    
    D[Deploy v1.1] --> C
    E[Test Green] --> C
    
    F[Switch Traffic] --> G[Load Balancer]
    G --> C2[Green Environment<br/>v1.1 - ACTIVE]
    G -.->|Not routing| B2[Blue Environment<br/>v1.0 - STANDBY]
```

**Benefits**: Instant rollback, full testing before cutover.

### Canary Deployment

Route small percentage of traffic to new version.

```mermaid
graph TD
    A[Load Balancer] --> B[95% traffic]
    A --> C[5% traffic]
    
    B --> D[Stable Version<br/>10 instances]
    C --> E[Canary Version<br/>1 instance]
    
    F[Monitor Metrics] --> E
    F --> G{Success?}
    
    G -->|Yes| H[Increase Canary %]
    G -->|No| I[Rollback]
    
    H --> J[Eventually 100%]
```

**Benefits**: Risk mitigation, real-world testing, gradual rollout.

## Performance Considerations

### Network Latency

```mermaid
graph TD
    A[Client] -->|10ms| B[Load Balancer]
    B -->|5ms| C[Gateway Instance]
    C -->|20ms| D[Backend Service]
    
    E[Total Latency] --> F[10ms + 5ms + 20ms = 35ms]
    
    G[Add Load Balancer Overhead] --> H[Consider Health Checks]
    G --> I[SSL Termination]
    G --> J[Connection Pooling]
```

**Optimization**: Use HTTP/2, connection pooling, and minimize load balancer hops.

### Connection Management

Each instance maintains connection pools to backends:

```javascript
// Conceptual pattern
const connectionPool = {
  maxConnections: 100,
  idleTimeout: 30000,
  keepAlive: true
};

// With 10 instances, total backend connections = 10 * 100 = 1000
```

**Theory**: More instances = more backend connections. Ensure backends can handle the load (see [architecture.md](./architecture.md)).

### Throughput Scaling

```mermaid
graph LR
    A[1 Instance<br/>1000 req/s] --> B[2 Instances<br/>2000 req/s]
    B --> C[4 Instances<br/>4000 req/s]
    C --> D[8 Instances<br/>8000 req/s]
    D --> E[16 Instances<br/>16000 req/s]
    
    style A fill:#ffcccc
    style E fill:#ccffcc
```

**Ideal**: Linear scaling. **Reality**: Slightly sub-linear due to load balancer overhead.

## Monitoring Horizontal Scaling

### Key Metrics

Track these metrics across all instances:

```mermaid
graph TD
    A[Monitoring Dashboard] --> B[Per-Instance Metrics]
    A --> C[Aggregate Metrics]
    A --> D[Load Balancer Metrics]
    
    B --> B1[CPU/Memory Usage]
    B --> B2[Request Rate]
    B --> B3[Error Rate]
    
    C --> C1[Total Throughput]
    C --> C2[Average Response Time]
    C --> C3[P95/P99 Latency]
    
    D --> D1[Connection Distribution]
    D --> D2[Health Check Status]
    D --> D3[Backend Health]
```

See [monitoring.md](./monitoring.md) for detailed observability strategies.

### Scaling Indicators

**Scale Out When**:
- Average CPU across instances > 70%
- Request queue growing
- Response time degrading
- Error rate increasing

**Scale In When**:
- Average CPU < 30% for sustained period
- Over-provisioned capacity
- Cost optimization needed

## Cost Optimization

### Right-Sizing Instances

```javascript
// Conceptual cost calculator
function calculateInstanceCost(traffic, instanceCapacity, instanceCost) {
  const requiredInstances = Math.ceil(traffic / instanceCapacity);
  const totalCost = requiredInstances * instanceCost;
  const utilizationRate = traffic / (requiredInstances * instanceCapacity);
  
  return { requiredInstances, totalCost, utilizationRate };
}
```

**Strategy**: Balance between performance and cost efficiency.

### Reserved vs Spot Instances

```mermaid
graph TD
    A[Instance Strategy] --> B[Base Load<br/>Reserved Instances]
    A --> C[Variable Load<br/>On-Demand]
    A --> D[Burst Capacity<br/>Spot Instances]
    
    B --> B1[70% cost savings]
    C --> C1[Flexibility]
    D --> D1[90% cost savings<br/>Risk of termination]
```

**Best Practice**: Mix instance types to optimize cost while maintaining reliability.

## Integration with Other Components

### Routing Complexity

More instances require sophisticated routing (see [routing.md](./routing.md)):
- Path-based routing
- Header-based routing  
- Service discovery
- Circuit breakers per instance

### Caching Strategy

Horizontal scaling affects caching (see [caching.md](./caching.md)):
- Distributed cache required
- Cache warming strategies
- Consistency considerations

### Security Implications

Multiple instances complicate security (see [security.md](./security.md)):
- Distributed rate limiting
- Centralized authentication
- Certificate management
- DDoS protection

## Hybrid Scaling Approach

Combine vertical and horizontal scaling:

```mermaid
graph TD
    A[Load Balancer] --> B[Instance 1<br/>8 cores, 16GB]
    A --> C[Instance 2<br/>8 cores, 16GB]
    A --> D[Instance 3<br/>8 cores, 16GB]
    A --> E[Instance N<br/>8 cores, 16GB]
    
    F[Vertical Scaling] -.->|Optimize each instance| B
    G[Horizontal Scaling] -.->|Add more instances| E
    
    style F fill:#ffffcc
    style G fill:#ccffff
```

**Strategy**: Vertically scale instances to optimal size, then horizontally scale by adding more instances.

## Best Practices

### 1. Design for Failure

Assume instances will fail:
- Implement graceful shutdown
- Use health checks
- Enable automatic recovery
- Plan for partial outages

### 2. Maintain Statelessness

```javascript
// BAD: Instance-local state
let requestCounter = 0; // Lost when instance dies

// GOOD: External state
await redis.incr('global:request:counter');
```

### 3. Use Service Discovery

Dynamic instance registration:

```mermaid
graph TD
    A[New Instance Starts] --> B[Register with Service Registry]
    B --> C[Load Balancer Discovers]
    C --> D[Start Receiving Traffic]
    
    E[Instance Fails] --> F[Deregister from Registry]
    F --> G[Load Balancer Removes]
    G --> H[Traffic Stops]
```

### 4. Implement Graceful Degradation

When scaling can't keep up:
- Enable circuit breakers (see [patterns.md](./patterns.md))
- Apply rate limiting (see [security.md](./security.md))
- Return cached responses
- Show degraded functionality

### 5. Test Scalability

Regular load testing:
- Identify bottlenecks before production
- Validate auto-scaling policies
- Ensure linear scaling behavior
- Test instance failure scenarios

## Challenges and Solutions

### Challenge 1: Distributed Tracing

**Problem**: Request spans multiple instances.

**Solution**: Implement correlation IDs:
```javascript
// Conceptual pattern
function generateTraceId() {
  return `${instanceId}-${Date.now()}-${randomId()}`;
}
```

### Challenge 2: Log Aggregation

**Problem**: Logs scattered across instances.

**Solution**: Centralized logging system (see [monitoring.md](./monitoring.md)).

### Challenge 3: Configuration Management

**Problem**: Keeping all instances in sync.

**Solution**: 
- External configuration service
- Environment variables
- Configuration management tools

### Challenge 4: Database Connection Limits

**Problem**: Each instance creates connections.

**Solution**:
- Connection pooling
- Database proxy (e.g., PgBouncer)
- Read replicas for scaling reads

## When NOT to Use Horizontal Scaling

Horizontal scaling isn't always the answer:
- **Very low traffic**: Overhead exceeds benefits
- **Tightly coupled state**: Requires expensive synchronization
- **Hardware underutilized**: Vertical scaling is cheaper
- **Development complexity**: Team lacks distributed systems expertise

## Summary

Horizontal scaling provides:
- **Unlimited growth potential**: No hardware ceiling
- **High availability**: Redundancy and fault tolerance
- **Flexibility**: Dynamic scaling with demand
- **Geographic distribution**: Lower latency globally

Key requirements:
- Stateless architecture
- External session/cache storage
- Load balancer infrastructure
- Proper monitoring and auto-scaling

Combined with vertical scaling, horizontal scaling enables systems to handle massive scale while maintaining reliability and performance.

## References

- [scaling.md](./scaling.md) - Overall scaling strategies including vertical
- [architecture.md](./architecture.md) - System design principles
- [patterns.md](./patterns.md) - Design patterns for distributed systems
- [monitoring.md](./monitoring.md) - Observability and metrics
- [caching.md](./caching.md) - Distributed caching strategies
- [security.md](./security.md) - Security in distributed environments
- [routing.md](./routing.md) - Traffic routing strategies
- [pros-cons.md](./pros-cons.md) - Trade-offs and decision framework