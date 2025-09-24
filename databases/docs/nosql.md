# NoSQL Databases

> **Part of**: [Database System Design](../README.md) | **Related**: [Relational Databases](./relational.md), [Data Consistency](./consistency.md), [Scaling Strategies](./scaling_strategies.md)

## Introduction

NoSQL databases emerged to address the limitations of relational databases in handling large-scale, distributed, and varied data types. They sacrifice some ACID properties for improved scalability, flexibility, and performance, making them ideal for modern web applications, big data, and real-time systems.

```mermaid
mindmap
  root((NoSQL Databases))
    Document
      MongoDB
      CouchDB
      Amazon DocumentDB
      Flexible Schema
      JSON/BSON
      Nested Documents
    Key-Value
      Redis
      Amazon DynamoDB
      Riak
      Simple Operations
      High Performance
      Distributed Caching
    Column-Family
      Apache Cassandra
      HBase
      Amazon SimpleDB
      Time Series
      Wide Rows
      Compression
    Graph
      Neo4j
      Amazon Neptune
      ArangoDB
      Relationships
      Network Analysis
      Social Networks
```

## NoSQL vs SQL Comparison

```mermaid
graph TB
    subgraph "SQL Databases"
        SQL[Relational/SQL] --> SQLProps[✅ ACID Properties<br/>✅ Complex Queries<br/>✅ Data Consistency<br/>❌ Rigid Schema<br/>❌ Vertical Scaling<br/>❌ Limited Flexibility]
    end
    
    subgraph "NoSQL Databases"
        NoSQL[Non-Relational/NoSQL] --> NoSQLProps[✅ Flexible Schema<br/>✅ Horizontal Scaling<br/>✅ High Performance<br/>❌ Eventual Consistency<br/>❌ Limited Complex Queries<br/>❌ Learning Curve]
    end
    
    subgraph "Use Case Decision"
        OLTP[Complex Transactions<br/>Financial Systems<br/>ERP Applications] --> SQL
        BigData[Big Data<br/>Real-time Analytics<br/>Rapid Development] --> NoSQL
        WebApps[Modern Web Apps<br/>Content Management<br/>IoT Systems] --> Hybrid[Polyglot Persistence<br/>Mix of Both]
    end
    
    style SQL fill:#FFB6C1
    style NoSQL fill:#90EE90
    style Hybrid fill:#87CEEB
```

## Document Databases

Document databases store data in document format, typically JSON or BSON, providing flexibility and intuitive data modeling for developers.

### Characteristics

```mermaid
graph LR
    Doc[Document Database] --> Schema[Flexible Schema<br/>No predefined structure]
    Doc --> Nested[Nested Documents<br/>Complex data structures]
    Doc --> Query[Rich Queries<br/>Field-level operations]
    Doc --> Scale[Horizontal Scaling<br/>Sharding support]
    
    Schema --> Example1[User Profile:<br/>{name, email, preferences}]
    Nested --> Example2[Order:<br/>{items: [...], shipping: {...}}]
    Query --> Example3[Find users where<br/>preferences.theme = 'dark']
    Scale --> Example4[Distribute by<br/>user_id hash]
    
    style Doc fill:#90EE90
```

### MongoDB Deep Dive

**Data Model:**
```javascript
// Flexible document structure
{
  "_id": ObjectId("..."),
  "username": "john_doe",
  "profile": {
    "firstName": "John",
    "lastName": "Doe",
    "avatar": "https://...",
    "preferences": {
      "theme": "dark",
      "notifications": {
        "email": true,
        "push": false
      }
    }
  },
  "orders": [
    {
      "orderId": "ORD-001",
      "items": ["item1", "item2"],
      "total": 299.99,
      "date": ISODate("2024-09-20")
    }
  ]
}
```

**Query Examples:**
```javascript
// Find users with dark theme preference
db.users.find({"profile.preferences.theme": "dark"});

// Update nested document
db.users.updateOne(
  {"username": "john_doe"},
  {"$set": {"profile.preferences.notifications.email": false}}
);

// Aggregation pipeline
db.users.aggregate([
  {"$match": {"profile.preferences.theme": "dark"}},
  {"$group": {"_id": "$profile.preferences.theme", "count": {"$sum": 1}}},
  {"$sort": {"count": -1}}
]);
```

### Sharding Strategy

```mermaid
graph TB
    App[Application] --> Router[MongoDB Router<br/>mongos]
    Router --> ConfigServer[Config Server<br/>Metadata]
    
    Router --> Shard1[Shard 1<br/>Range: user_1 - user_333k]
    Router --> Shard2[Shard 2<br/>Range: user_334k - user_666k]
    Router --> Shard3[Shard 3<br/>Range: user_667k - user_1M]
    
    Shard1 --> Replica1[Primary + 2 Secondaries]
    Shard2 --> Replica2[Primary + 2 Secondaries]
    Shard3 --> Replica3[Primary + 2 Secondaries]
    
    style Router fill:#87CEEB
    style ConfigServer fill:#F0E68C
```

### Use Cases
- **Content Management Systems**: Flexible content structures
- **Product Catalogs**: Varying product attributes
- **User Profiles**: Dynamic user data
- **Real-time Analytics**: Event logging and analysis

## Key-Value Stores

Key-value stores are the simplest NoSQL databases, offering high performance through straightforward key-value operations.

### Architecture

```mermaid
graph LR
    Client[Client Application] --> KV[Key-Value Store]
    
    subgraph "Operations"
        GET[GET key → value]
        PUT[PUT key, value]
        DELETE[DELETE key]
        LIST[LIST keys pattern]
    end
    
    Client --> GET
    Client --> PUT
    Client --> DELETE
    Client --> LIST
    
    subgraph "Storage Engine"
        Memory[In-Memory<br/>Ultra-fast access]
        Persistent[Persistent<br/>Disk-backed]
        Hybrid[Hybrid<br/>Memory + Disk]
    end
    
    KV --> Memory
    KV --> Persistent
    KV --> Hybrid
    
    style KV fill:#F0E68C
    style Memory fill:#FFB6C1
    style Hybrid fill:#90EE90
```

### Redis Implementation

**Data Structures:**
```redis
# String operations
SET user:1001:name "John Doe"
GET user:1001:name

# Hash operations (object-like)
HSET user:1001 name "John Doe" email "john@example.com" age 30
HGET user:1001 name

# List operations (arrays)
LPUSH user:1001:notifications "Welcome message"
LPUSH user:1001:notifications "Friend request"
LRANGE user:1001:notifications 0 -1

# Set operations (unique values)
SADD user:1001:interests "technology" "music" "sports"
SMEMBERS user:1001:interests

# Sorted sets (ranked data)
ZADD leaderboard 1500 "player1" 1200 "player2" 1800 "player3"
ZRANGE leaderboard 0 -1 WITHSCORES
```

**Advanced Features:**
```redis
# Expiration
SETEX session:abc123 3600 "user_data"  # Expires in 1 hour

# Pub/Sub
PUBLISH chat:room1 "Hello everyone!"
SUBSCRIBE chat:room1

# Atomic operations
MULTI
INCR user:1001:points 10
DECR inventory:item123 1
EXEC
```

### DynamoDB Architecture

```mermaid
graph TB
    App[Application] --> DynamoDB[Amazon DynamoDB]
    
    subgraph "Data Distribution"
        DynamoDB --> Partition1[Partition 1<br/>Hash: 0-333]
        DynamoDB --> Partition2[Partition 2<br/>Hash: 334-666]
        DynamoDB --> Partition3[Partition 3<br/>Hash: 667-999]
    end
    
    subgraph "Replication"
        Partition1 --> AZ1A[AZ-1a Copy]
        Partition1 --> AZ1B[AZ-1b Copy]
        Partition1 --> AZ1C[AZ-1c Copy]
    end
    
    subgraph "Features"
        GlobalTables[Global Tables<br/>Multi-region]
        DynamoStreams[DynamoDB Streams<br/>Change capture]
        DAX[DynamoDB DAX<br/>Microsecond latency]
    end
    
    DynamoDB -.-> GlobalTables
    DynamoDB -.-> DynamoStreams
    DynamoDB -.-> DAX
    
    style DynamoDB fill:#F0E68C
    style GlobalTables fill:#87CEEB
```

### Use Cases
- **Session Management**: Web session storage
- **Caching Layer**: Database query results, computed data
- **Real-time Leaderboards**: Gaming scores, rankings
- **Shopping Carts**: Temporary transaction data
- **Rate Limiting**: API throttling, usage tracking

## Column-Family Databases

Column-family databases organize data in column families, optimizing for write-heavy workloads and time-series data.

### Cassandra Architecture

```mermaid
graph TB
    subgraph "Cassandra Cluster"
        Node1[Node 1<br/>Token Range: 0-25%] 
        Node2[Node 2<br/>Token Range: 25-50%]
        Node3[Node 3<br/>Token Range: 50-75%]
        Node4[Node 4<br/>Token Range: 75-100%]
    end
    
    Node1 -.-> Node2
    Node2 -.-> Node3
    Node3 -.-> Node4
    Node4 -.-> Node1
    
    subgraph "Data Distribution"
        PartitionKey[Partition Key<br/>user_id] --> Hash[Consistent Hashing]
        Hash --> TokenRing[Token Ring<br/>Determines Node]
    end
    
    subgraph "Replication"
        RF[Replication Factor = 3]
        RF --> Primary[Primary Node]
        RF --> Replica1[Replica 1]
        RF --> Replica2[Replica 2]
    end
    
    Client[Client] --> LoadBalancer[Load Balancer]
    LoadBalancer --> Node1
    LoadBalancer --> Node2
    LoadBalancer --> Node3
    LoadBalancer --> Node4
    
    style Node1 fill:#90EE90
    style Node2 fill:#87CEEB
    style Node3 fill:#F0E68C
    style Node4 fill:#DDA0DD
```

### Data Model

```cql
-- Create keyspace (database)
CREATE KEYSPACE ecommerce 
WITH REPLICATION = {
  'class': 'SimpleStrategy',
  'replication_factor': 3
};

-- Time-series table for user events
CREATE TABLE user_events (
    user_id UUID,
    event_time TIMESTAMP,
    event_type TEXT,
    event_data MAP<TEXT, TEXT>,
    PRIMARY KEY (user_id, event_time)
) WITH CLUSTERING ORDER BY (event_time DESC);

-- Insert events
INSERT INTO user_events (user_id, event_time, event_type, event_data)
VALUES (
    uuid(),
    '2024-09-20 10:30:00',
    'page_view',
    {'page': '/products', 'referrer': 'google'}
);

-- Query recent events
SELECT * FROM user_events 
WHERE user_id = ? 
  AND event_time > '2024-09-20 00:00:00'
ORDER BY event_time DESC
LIMIT 100;
```

### Consistency Levels

```mermaid
graph LR
    Client[Client Query] --> CL[Consistency Level]
    
    CL --> ONE[ONE<br/>1 node responds<br/>Fastest, least consistent]
    CL --> QUORUM[QUORUM<br/>Majority responds<br/>Balanced]
    CL --> ALL[ALL<br/>All nodes respond<br/>Slowest, most consistent]
    
    subgraph "RF=3 Cluster"
        N1[Node 1] 
        N2[Node 2]
        N3[Node 3]
    end
    
    ONE --> N1
    QUORUM --> N1
    QUORUM --> N2
    ALL --> N1
    ALL --> N2
    ALL --> N3
    
    style QUORUM fill:#90EE90
    style ONE fill:#F0E68C
    style ALL fill:#FFB6C1
```

### Use Cases
- **Time-Series Data**: IoT sensor data, logs, metrics
- **Write-Heavy Applications**: Event logging, audit trails
- **Real-time Analytics**: User behavior tracking
- **Content Management**: Blog posts, comments, media metadata

## Graph Databases

Graph databases excel at managing highly connected data and complex relationships.

### Graph Model

```mermaid
graph TB
    subgraph "Social Network Graph"
        Alice((Alice)) -->|FRIENDS_WITH| Bob((Bob))
        Bob -->|FRIENDS_WITH| Charlie((Charlie))
        Alice -->|LIKES| Post1[Post: Great day!]
        Bob -->|COMMENTED_ON| Post1
        Charlie -->|LIKES| Post1
        Alice -->|WORKS_AT| Company1[Company: TechCorp]
        Bob -->|WORKS_AT| Company1
        Charlie -->|WORKS_AT| Company2[Company: StartupXYZ]
    end
    
    style Alice fill:#FFB6C1
    style Bob fill:#90EE90
    style Charlie fill:#87CEEB
    style Post1 fill:#F0E68C
    style Company1 fill:#DDA0DD
    style Company2 fill:#ADD8E6

```

### Neo4j Implementation

```cypher
// Create nodes
CREATE (alice:Person {name: 'Alice', age: 30, city: 'San Francisco'})
CREATE (bob:Person {name: 'Bob', age: 28, city: 'New York'})
CREATE (techcorp:Company {name: 'TechCorp', industry: 'Technology'})

// Create relationships
CREATE (alice)-[:FRIENDS_WITH {since: '2020-01-15'}]->(bob)
CREATE (alice)-[:WORKS_AT {position: 'Engineer', since: '2022-03-01'}]->(techcorp)

// Complex graph queries
// Find friends of friends
MATCH (me:Person {name: 'Alice'})-[:FRIENDS_WITH]-()-[:FRIENDS_WITH]-(fof:Person)
WHERE fof <> me
RETURN DISTINCT fof.name

// Shortest path between people
MATCH path = shortestPath((alice:Person {name: 'Alice'})-[*]-(charlie:Person {name: 'Charlie'}))
RETURN path

// Recommendation: People who work at the same company as friends
MATCH (me:Person {name: 'Alice'})-[:FRIENDS_WITH]-(friend)-[:WORKS_AT]->(company)
WHERE NOT (me)-[:WORKS_AT]->(company)
RETURN company.name, count(friend) as mutual_connections
ORDER BY mutual_connections DESC
```

### Graph Algorithms

```mermaid
graph TB
    subgraph "Graph Analysis"
        PageRank[PageRank<br/>Node Importance]
        Community[Community Detection<br/>Clustering]
        Centrality[Centrality Measures<br/>Influence Analysis]
        ShortestPath[Shortest Path<br/>Connection Analysis]
    end
    
    subgraph "Use Cases"
        SocialMedia[Social Media<br/>Friend recommendations]
        Fraud[Fraud Detection<br/>Suspicious patterns]
        Logistics[Logistics<br/>Route optimization]
        Knowledge[Knowledge Graphs<br/>Information networks]
    end
    
    PageRank --> SocialMedia
    Community --> SocialMedia
    Centrality --> Fraud
    ShortestPath --> Logistics
    
    style PageRank fill:#90EE90
    style Fraud fill:#FFB6C1
    style Logistics fill:#87CEEB
    style Knowledge fill:#F0E68C
```

### Use Cases
- **Social Networks**: Friend recommendations, influence analysis
- **Fraud Detection**: Pattern recognition in financial transactions
- **Recommendation Engines**: Product and content recommendations
- **Network Analysis**: Infrastructure monitoring, dependency mapping
- **Knowledge Graphs**: Semantic search, AI applications

## CAP Theorem and NoSQL

The CAP theorem fundamentally influences NoSQL database design decisions:

```mermaid
graph TD
    CAP[CAP Theorem<br/>Choose Any Two] --> C[Consistency<br/>All nodes see same data]
    CAP --> A[Availability<br/>System remains operational]
    CAP --> P[Partition Tolerance<br/>Survives network failures]
    
    C & A --> CA[CA Systems<br/>❌ Not suitable for<br/>distributed NoSQL]
    C & P --> CP[CP Systems<br/>MongoDB - default<br/>HBase<br/>Redis Cluster]
    A & P --> AP[AP Systems<br/>Cassandra<br/>DynamoDB<br/>CouchDB<br/>Riak]
    
    subgraph "NoSQL Positioning"
        CP --> CPChar[✅ Strong consistency<br/>❌ May become unavailable<br/>🎯 Financial data, user accounts]
        AP --> APChar[✅ Always available<br/>❌ Eventual consistency<br/>🎯 Social media, analytics]
    end
    
    style CP fill:#90EE90
    style AP fill:#87CEEB
    style CA fill:#FFB6C1

```

## NoSQL Database Comparison

### Feature Matrix

```mermaid
graph TB
    subgraph "NoSQL Database Comparison"
        Document[Document<br/>MongoDB, CouchDB] --> DocFeatures[✅ Flexible schema<br/>✅ Rich queries<br/>✅ Developer friendly<br/>❌ Memory usage<br/>❌ Join limitations]
        
        KeyValue[Key-Value<br/>Redis, DynamoDB] --> KVFeatures[✅ High performance<br/>✅ Simple operations<br/>✅ Horizontal scaling<br/>❌ Limited queries<br/>❌ Data modeling]
        
        ColumnFamily[Column-Family<br/>Cassandra, HBase] --> CFFeatures[✅ Write performance<br/>✅ Time-series data<br/>✅ Compression<br/>❌ Query flexibility<br/>❌ Learning curve]
        
        Graph[Graph<br/>Neo4j, Neptune] --> GFeatures[✅ Relationship queries<br/>✅ Graph algorithms<br/>✅ Pattern matching<br/>❌ Scaling challenges<br/>❌ Specialized use cases]
    end
    
    style Document fill:#90EE90
    style KeyValue fill:#F0E68C
    style ColumnFamily fill:#87CEEB
    style Graph fill:#DDA0DD
```

### Performance Characteristics

| Database Type | Read Performance | Write Performance | Query Complexity | Scaling |
|---------------|------------------|-------------------|------------------|---------|
| **Document** | High | Medium-High | High | Horizontal |
| **Key-Value** | Very High | Very High | Low | Horizontal |
| **Column-Family** | Medium-High | Very High | Medium | Linear |
| **Graph** | Medium | Medium | Very High | Vertical |

### Selection Criteria

```mermaid
flowchart TD
    Start[Data Requirements] --> Structure{Data Structure?}
    
    Structure -->|Hierarchical/Nested| Document[Document Database<br/>MongoDB, CouchDB]
    Structure -->|Simple Key-Value| KeyValue[Key-Value Store<br/>Redis, DynamoDB]
    Structure -->|Time-series/Wide-rows| ColumnFamily[Column-Family<br/>Cassandra, HBase]
    Structure -->|Highly Connected| Graph[Graph Database<br/>Neo4j, Neptune]
    
    Document --> DocDecision{Query Needs?}
    DocDecision -->|Complex Queries| MongoDB[MongoDB<br/>Rich query language]
    DocDecision -->|Simple Queries| CouchDB[CouchDB<br/>HTTP-based]
    
    KeyValue --> KVDecision{Persistence?}
    KVDecision -->|In-Memory| Redis[Redis<br/>Ultra-fast cache]
    KVDecision -->|Persistent| DynamoDB[DynamoDB<br/>Managed service]
    
    ColumnFamily --> CFDecision{Scale Requirements?}
    CFDecision -->|Massive Scale| Cassandra[Cassandra<br/>Linear scaling]
    CFDecision -->|Hadoop Integration| HBase[HBase<br/>Hadoop ecosystem]
    
    Graph --> GraphDecision{Graph Complexity?}
    GraphDecision -->|Complex Algorithms| Neo4j[Neo4j<br/>Rich graph features]
    GraphDecision -->|Simple Relationships| Neptune[Neptune<br/>Managed service]
    
    style MongoDB fill:#90EE90
    style Redis fill:#F0E68C
    style Cassandra fill:#87CEEB
    style Neo4j fill:#DDA0DD
```

## Implementation Best Practices

### Data Modeling Guidelines

#### Document Databases
```javascript
// ✅ Good: Embed related data
{
  "userId": "user123",
  "profile": {
    "name": "John Doe",
    "preferences": {...}
  },
  "recentOrders": [
    {"orderId": "ord1", "total": 99.99}
  ]
}

// ❌ Bad: Over-normalization
{
  "userId": "user123",
  "profileId": "profile456", // Reference instead of embed
  "orderIds": ["ord1", "ord2"] // References instead of recent data
}
```

#### Key-Value Stores
```javascript
// ✅ Good: Hierarchical keys
"user:123:profile" → {"name": "John", "email": "..."}
"user:123:settings" → {"theme": "dark", "notifications": true}
"session:abc123" → {"userId": "123", "expires": "..."}

// ❌ Bad: Flat structure requiring scans
"user_123_name" → "John"
"user_123_email" → "john@example.com"
"user_123_theme" → "dark"
```

### Performance Optimization

#### Indexing Strategies
```javascript
// MongoDB indexing
db.products.createIndex({"category": 1, "price": -1}); // Compound index
db.users.createIndex({"email": 1}, {"unique": true});   // Unique index
db.posts.createIndex({"content": "text"});              // Text search

// Cassandra partitioning
CREATE TABLE user_events (
    user_id UUID,
    event_date DATE,
    event_time TIMESTAMP,
    event_data TEXT,
    PRIMARY KEY ((user_id, event_date), event_time)
) WITH CLUSTERING ORDER BY (event_time DESC);
```

#### Connection Management
```javascript
// MongoDB connection pooling
const client = new MongoClient(uri, {
  maxPoolSize: 100,        // Maximum connections
  minPoolSize: 5,          // Minimum connections
  maxIdleTimeMS: 30000,    // Close idle connections
  serverSelectionTimeoutMS: 5000
});

// Redis connection pooling
const redis = new Redis.Cluster([
  {host: 'redis1', port: 6379},
  {host: 'redis2', port: 6379}
], {
  redisOptions: {
    password: 'password',
    maxRetriesPerRequest: 3
  },
  maxRetriesPerRequest: null
});
```

## Migration Strategies

### From SQL to NoSQL

```mermaid
sequenceDiagram
    participant App as Application
    participant SQL as SQL Database
    participant NoSQL as NoSQL Database
    participant Sync as Data Sync Service
    
    Note over App, NoSQL: Phase 1: Dual Write
    App->>SQL: Write data
    App->>NoSQL: Write same data
    App->>SQL: Read data
    
    Note over App, NoSQL: Phase 2: Background Sync
    Sync->>SQL: Read existing data
    Sync->>NoSQL: Migrate historical data
    
    Note over App, NoSQL: Phase 3: Switch Reads
    App->>SQL: Write data
    App->>NoSQL: Write same data
    App->>NoSQL: Read data (validate)
    
    Note over App, NoSQL: Phase 4: Full Migration
    App->>NoSQL: Write data only
    App->>NoSQL: Read data only
```

### Polyglot Persistence

```mermaid
graph TB
    subgraph "Modern Application Architecture"
        API[API Gateway] --> UserService[User Service]
        API --> ProductService[Product Service]
        API --> OrderService[Order Service]
        API --> AnalyticsService[Analytics Service]
        
        UserService --> PostgreSQL[(PostgreSQL<br/>User accounts, ACID)]
        ProductService --> MongoDB[(MongoDB<br/>Product catalog, flexible)]
        OrderService --> PostgreSQL2[(PostgreSQL<br/>Transactions, consistency)]
        AnalyticsService --> Cassandra[(Cassandra<br/>Time-series events)]
        
        Redis[(Redis<br/>Cache layer)] --> UserService
        Redis --> ProductService
        Redis --> OrderService
        
        Elasticsearch[(Elasticsearch<br/>Search index)] --> ProductService
    end
    
    style PostgreSQL fill:#FFB6C1
    style MongoDB fill:#90EE90
    style Cassandra fill:#87CEEB
    style Redis fill:#F0E68C
    style Elasticsearch fill:#DDA0DD
```

## Related Topics

- **[Data Consistency](./consistency.md)**: Understanding consistency models in NoSQL systems
- **[Scaling Strategies](./scaling_strategies.md)**: NoSQL-specific scaling patterns and techniques
- **[Case Studies](./case-studies.md)**: Real-world NoSQL implementations and architecture decisions
- **[Relational Databases](./relational.md)**: Comparison with traditional SQL databases

## Further Reading

### Books
- "NoSQL Distilled" by Pramod J. Sadalage and Martin Fowler
- "MongoDB: The Definitive Guide" by Kristina Chodorow
- "Cassandra: The Definitive Guide" by Jeff Carpenter and Eben Hewitt
- "Graph Databases" by Ian Robinson, Jim Webber, and Emil Eifrem

### Documentation
- [MongoDB Documentation](https://docs.mongodb.com/)
- [Apache Cassandra Documentation](https://cassandra.apache.org/doc/)
- [Redis Documentation](https://redis.io/documentation)
- [Neo4j Documentation](https://neo4j.com/docs/)

### Online Resources
- [NoSQL Database Patterns](https://highlyscalable.wordpress.com/2012/03/01/nosql-data-modeling-techniques/)
- [CAP Theorem Explained](https://www.ibm.com/cloud/learn/cap-theorem)
- [Database Selection Guide](https://db-engines.com/en/)

---

*Last Updated: September 2025 | [Back to Main Documentation](../README.md)*