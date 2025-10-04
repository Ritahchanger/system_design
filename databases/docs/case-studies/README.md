# Database Case Studies

This directory contains detailed case studies of popular database systems, examining their architecture, use cases, trade-offs, and real-world applications.

## 📚 Overview

Understanding how different databases are designed and when to use them is crucial for building scalable systems. These case studies provide practical insights into database selection and implementation strategies.

---

## 📖 Available Case Studies

### 1. [Redis](docs/redis.md) - In-Memory Data Store
**Type**: Key-Value Store (In-Memory)

**Key Characteristics**:
- Sub-millisecond latency
- Support for various data structures (strings, lists, sets, hashes, sorted sets)
- Pub/Sub messaging
- Persistence options (RDB, AOF)

**Best For**:
- Caching layers
- Session storage
- Real-time analytics
- Leaderboards and counters
- Message queues

**Learn More**:
- [Relational vs NoSQL](../relational.md) - Understanding when to use cache vs primary storage
- [Scaling Strategies](../scaling_strategies.md) - Redis cluster and replication patterns
- [Consistency Models](../consistency.md) - Redis persistence and durability trade-offs

---

### 2. [MongoDB](docs/mongodb.md) - Document Database
**Type**: Document-Oriented NoSQL

**Key Characteristics**:
- Flexible JSON-like documents (BSON)
- Rich query language
- Horizontal scalability through sharding
- Strong consistency with replica sets

**Best For**:
- Content management systems
- Real-time analytics
- IoT applications
- Product catalogs
- Mobile applications

**Learn More**:
- [NoSQL Databases](../nosql.md) - Document model deep dive
- [Scaling Strategies](../scaling_strategies.md) - MongoDB sharding and replication
- [Consistency Models](../consistency.md) - Replica set consistency guarantees

---

### 3. [Cassandra](docs/cassandra.md) - Wide-Column Store
**Type**: Wide-Column NoSQL

**Key Characteristics**:
- Masterless architecture (peer-to-peer)
- Linear scalability
- High availability with no single point of failure
- Tunable consistency (eventual to strong)

**Best For**:
- Time-series data
- Write-heavy workloads
- Geographically distributed systems
- IoT and sensor data
- Messaging platforms

**Learn More**:
- [NoSQL Databases](../nosql.md) - Wide-column model explained
- [Consistency Models](../consistency.md) - CAP theorem in practice with Cassandra
- [Scaling Strategies](../scaling_strategies.md) - Cassandra's ring architecture

---

### 4. [DynamoDB](docs/dynamodb.md) - Managed NoSQL Database
**Type**: Key-Value and Document Store (Managed Service)

**Key Characteristics**:
- Fully managed by AWS
- Single-digit millisecond latency at any scale
- Built-in security, backup, and restore
- Global tables for multi-region replication

**Best For**:
- Serverless applications
- Gaming applications (session data, leaderboards)
- Ad tech and real-time bidding
- Mobile backends
- Microservices data stores

**Learn More**:
- [NoSQL Databases](../nosql.md) - Key-value model and document model
- [Scaling Strategies](../scaling_strategies.md) - Auto-scaling and partition strategies
- [Consistency Models](../consistency.md) - Eventual vs strong consistency options

---

### 5. [Neo4j](docs/neo4j.md) - Graph Database
**Type**: Graph Database

**Key Characteristics**:
- Native graph storage and processing
- Cypher query language
- ACID compliance
- Relationship-first data model

**Best For**:
- Social networks
- Recommendation engines
- Fraud detection
- Knowledge graphs
- Network and IT operations

**Learn More**:
- [NoSQL Databases](../nosql.md) - Graph model fundamentals
- [Relational Databases](../relational.md) - When to choose graphs over relational joins
- [Consistency Models](../consistency.md) - ACID in graph databases

---

## 🎯 How to Use This Guide

### For Beginners
1. Start with the foundational documents:
   - [Relational Databases](../relational.md) - Understand SQL fundamentals
   - [NoSQL Databases](../nosql.md) - Learn about different NoSQL models
   - [Consistency Models](../consistency.md) - Grasp CAP theorem

2. Then explore case studies based on your use case:
   - Need caching? → **Redis**
   - Flexible schema? → **MongoDB**
   - Write-heavy, time-series? → **Cassandra**
   - AWS-native serverless? → **DynamoDB**
   - Complex relationships? → **Neo4j**

### For Interview Preparation
1. **Understand the trade-offs** for each database
2. **Know the CAP theorem** implications ([Consistency Models](../consistency.md))
3. **Study scaling patterns** ([Scaling Strategies](../scaling_strategies.md))
4. Practice designing systems using these databases

### For System Design
1. Review requirements (read/write patterns, consistency needs)
2. Map requirements to database characteristics
3. Consider hybrid approaches (polyglot persistence)
4. Plan scaling strategy early

---

## 🔄 Cross-References

### By Use Case

#### Caching & Session Management
- **Primary**: [Redis](docs/redis.md)
- **Concepts**: [Scaling Strategies](../scaling_strategies.md) - Cache invalidation patterns

#### Real-Time Analytics
- **Options**: [Redis](docs/redis.md), [MongoDB](docs/mongodb.md), [Cassandra](docs/cassandra.md)
- **Concepts**: [Consistency Models](../consistency.md) - Trade-offs between speed and accuracy

#### Time-Series Data
- **Primary**: [Cassandra](docs/cassandra.md)
- **Alternative**: [MongoDB](docs/mongodb.md) with time-series collections
- **Concepts**: [Scaling Strategies](../scaling_strategies.md) - Partitioning time-series data

#### Social Networks & Graphs
- **Primary**: [Neo4j](docs/neo4j.md)
- **Alternative**: [MongoDB](docs/mongodb.md) for simpler relationships
- **Concepts**: [NoSQL Databases](../nosql.md) - Graph vs document models

#### High Availability Systems
- **Options**: [Cassandra](docs/cassandra.md), [DynamoDB](docs/dynamodb.md)
- **Concepts**: [Consistency Models](../consistency.md) - Eventual consistency patterns

#### Serverless Applications
- **Primary**: [DynamoDB](docs/dynamodb.md)
- **Alternative**: [MongoDB Atlas](docs/mongodb.md)
- **Concepts**: [Scaling Strategies](../scaling_strategies.md) - Auto-scaling strategies

---

## 📊 Comparison Matrix

| Database | Type | Consistency | Scalability | Best Use Case | Complexity |
|----------|------|-------------|-------------|---------------|------------|
| **Redis** | Key-Value | Strong (single node) | Vertical + Clustering | Caching, Real-time | Low |
| **MongoDB** | Document | Strong (default) | Horizontal (Sharding) | Flexible schemas | Medium |
| **Cassandra** | Wide-Column | Tunable | Linear horizontal | Write-heavy, Time-series | High |
| **DynamoDB** | Key-Value/Document | Tunable | Auto-scaling | Serverless, Low-latency | Low-Medium |
| **Neo4j** | Graph | ACID | Vertical + Clustering | Relationship-heavy | Medium-High |

---

## 🏗️ Real-World Architecture Patterns

### Pattern 1: Polyglot Persistence
Use multiple databases for different concerns:
```
User Auth → PostgreSQL (relational.md)
Session Data → Redis (redis.md)
Product Catalog → MongoDB (mongodb.md)
Recommendations → Neo4j (neo4j.md)
Activity Logs → Cassandra (cassandra.md)
```

### Pattern 2: CQRS (Command Query Responsibility Segregation)
- **Write Model**: [Cassandra](docs/cassandra.md) for high-throughput writes
- **Read Model**: [MongoDB](docs/mongodb.md) or [DynamoDB](docs/dynamodb.md) for optimized queries
- **Cache Layer**: [Redis](docs/redis.md)

**Reference**: [Consistency Models](../consistency.md) - Eventual consistency between models

### Pattern 3: Lambda Architecture
- **Speed Layer**: [Redis](docs/redis.md)
- **Batch Layer**: [Cassandra](docs/cassandra.md) or [MongoDB](docs/mongodb.md)
- **Serving Layer**: [DynamoDB](docs/dynamodb.md)

**Reference**: [Scaling Strategies](../scaling_strategies.md) - Batch vs stream processing

---

## 🎓 Learning Path

### Week 1: Foundations
1. Read [Relational Databases](../relational.md)
2. Read [NoSQL Databases](../nosql.md)
3. Study [Consistency Models](../consistency.md)

### Week 2: In-Memory & Document Stores
1. Deep dive: [Redis](docs/redis.md)
2. Deep dive: [MongoDB](docs/mongodb.md)
3. Build a simple cache + MongoDB app

### Week 3: Distributed Systems
1. Deep dive: [Cassandra](docs/cassandra.md)
2. Deep dive: [DynamoDB](docs/dynamodb.md)
3. Study [Scaling Strategies](../scaling_strategies.md)

### Week 4: Specialized Databases
1. Deep dive: [Neo4j](docs/neo4j.md)
2. Compare all databases
3. Design a polyglot persistence system

---

## 💡 Key Takeaways

### When to Use Each Database

**Use Redis when**:
- You need sub-millisecond latency
- Data fits in memory
- You need pub/sub messaging
- Caching is the primary use case

**Use MongoDB when**:
- Schema flexibility is important
- You need rich query capabilities
- Documents naturally model your data
- You want strong consistency with scalability

**Use Cassandra when**:
- Write throughput is critical
- You need linear scalability
- Multi-datacenter deployment
- Time-series or append-only data

**Use DynamoDB when**:
- You're on AWS and want managed service
- Predictable single-digit millisecond latency
- Serverless architecture
- Variable workload with auto-scaling

**Use Neo4j when**:
- Relationships are first-class citizens
- Deep graph traversals are common
- Social networks or recommendation engines
- Fraud detection with connected data

---

## 🔗 Related Documentation

- [Relational Databases](../relational.md) - SQL fundamentals and normalization
- [NoSQL Databases](../nosql.md) - NoSQL categories and models
- [Consistency Models](../consistency.md) - CAP theorem, ACID, BASE
- [Scaling Strategies](../scaling_strategies.md) - Sharding, replication, partitioning

---

## 📝 Practice Exercises

1. **Exercise 1**: Design a social media platform
   - Choose appropriate databases for users, posts, relationships, and feeds
   - Justify your choices using concepts from case studies

2. **Exercise 2**: Design an IoT data collection system
   - Handle millions of writes per second
   - Support real-time dashboards
   - Enable historical analysis

3. **Exercise 3**: Design an e-commerce platform
   - Product catalog with flexible attributes
   - Shopping cart with session management
   - Order history and analytics
   - Recommendation engine

**Hint**: Most solutions will use multiple databases (polyglot persistence)

---

## 🚀 Next Steps

After completing these case studies:

1. **Hands-on Practice**: 
   - Set up each database locally
   - Run example queries
   - Benchmark performance

2. **Advanced Topics**:
   - Study database internals (B-trees, LSM trees)
   - Learn about distributed consensus (Paxos, Raft)
   - Explore database-specific optimization

3. **System Design**:
   - Practice designing systems with these databases
   - Focus on trade-offs and justification
   - Consider failure scenarios

---

## 📚 Additional Resources

- **Books**: 
  - "Designing Data-Intensive Applications" by Martin Kleppmann
  - "Database Internals" by Alex Petrov

- **Documentation**:
  - Official docs for each database
  - Architecture blogs from companies (Netflix, Uber, Airbnb)

- **Practice**:
  - Set up databases using Docker
  - Follow official tutorials
  - Build small projects with each database

---

**Happy Learning! 🎉**

Remember: There's no "best" database—only the right database for your specific use case. Master the art of choosing and combining databases to build robust, scalable systems.