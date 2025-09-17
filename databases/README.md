# Database Systems in System Design

## Overview

Databases are the backbone of modern applications, providing persistent storage, data integrity, and query capabilities. Understanding different database types, their trade-offs, and when to use them is crucial for system design interviews and real-world applications.

## Table of Contents

- [Database Types](#database-types)
- [ACID Properties](#acid-properties)
- [CAP Theorem](#cap-theorem)
- [Database Scaling](#database-scaling)
- [SQL vs NoSQL](#sql-vs-nosql)
- [Database Selection Criteria](#database-selection-criteria)
- [Common Database Patterns](#common-database-patterns)
- [Performance Optimization](#performance-optimization)
- [Database Architecture Patterns](#database-architecture-patterns)

## Database Types

### Relational Databases (RDBMS)
- **Examples**: PostgreSQL, MySQL, SQL Server, Oracle
- **Use Cases**: Complex queries, transactions, data consistency
- **Strengths**: ACID compliance, mature ecosystem, standardized SQL
- **Weaknesses**: Vertical scaling limitations, schema rigidity

### Document Databases
- **Examples**: MongoDB, CouchDB, Amazon DocumentDB
- **Use Cases**: Content management, catalogs, user profiles
- **Strengths**: Flexible schema, natural object mapping
- **Weaknesses**: Limited complex queries, eventual consistency

### Key-Value Stores
- **Examples**: Redis, DynamoDB, Riak
- **Use Cases**: Caching, session storage, real-time recommendations
- **Strengths**: High performance, simple model, horizontal scaling
- **Weaknesses**: Limited query capabilities, no complex relationships

### Column-Family
- **Examples**: Cassandra, HBase, Amazon SimpleDB
- **Use Cases**: Time-series data, IoT data, analytics
- **Strengths**: High write throughput, compression, scalability
- **Weaknesses**: Limited ad-hoc queries, eventual consistency

### Graph Databases
- **Examples**: Neo4j, Amazon Neptune, ArangoDB
- **Use Cases**: Social networks, recommendation engines, fraud detection
- **Strengths**: Relationship queries, pattern matching
- **Weaknesses**: Scaling challenges, specialized use cases

### Time-Series Databases
- **Examples**: InfluxDB, TimescaleDB, Prometheus
- **Use Cases**: Monitoring, IoT sensors, financial data
- **Strengths**: Optimized for time-based data, compression
- **Weaknesses**: Limited general-purpose queries

## ACID Properties

```mermaid
graph LR
    A[ACID Properties] --> B[Atomicity]
    A --> C[Consistency]
    A --> D[Isolation]
    A --> E[Durability]
    
    B --> B1[All or Nothing]
    C --> C1[Valid State]
    D --> D1[Concurrent Safety]
    E --> E1[Persistent Storage]
```

### Atomicity
Transactions are all-or-nothing operations. Either all changes are applied or none are.

### Consistency
Database remains in a valid state before and after transactions.

### Isolation
Concurrent transactions don't interfere with each other.

### Durability
Committed transactions survive system failures.

## CAP Theorem

```mermaid
triangle CAP
    Consistency --- Availability
    Consistency --- Partition-Tolerance
    Availability --- Partition-Tolerance
    
    CA[CA Systems<br/>Traditional RDBMS<br/>Single Node]
    CP[CP Systems<br/>MongoDB<br/>HBase]
    AP[AP Systems<br/>Cassandra<br/>DynamoDB]
```

You can only guarantee 2 out of 3 properties:
- **Consistency**: All nodes see the same data simultaneously
- **Availability**: System remains operational
- **Partition Tolerance**: System continues despite network failures

## Database Scaling

### Vertical Scaling (Scale Up)
- Add more power to existing machine
- **Pros**: Simple, maintains ACID properties
- **Cons**: Hardware limits, single point of failure, expensive

### Horizontal Scaling (Scale Out)

#### Read Replicas
```mermaid
graph TB
    App[Application] --> Master[Master DB<br/>Writes]
    App --> Replica1[Read Replica 1]
    App --> Replica2[Read Replica 2]
    App --> Replica3[Read Replica 3]
    
    Master -.->|Replication| Replica1
    Master -.->|Replication| Replica2
    Master -.->|Replication| Replica3
```

#### Sharding
```mermaid
graph TB
    App[Application] --> Router[Shard Router]
    Router --> Shard1[Shard 1<br/>Users A-F]
    Router --> Shard2[Shard 2<br/>Users G-M]
    Router --> Shard3[Shard 3<br/>Users N-S]
    Router --> Shard4[Shard 4<br/>Users T-Z]
```

**Sharding Strategies:**
- **Range-based**: Partition by key ranges
- **Hash-based**: Use hash function to determine shard
- **Directory-based**: Lookup service maps keys to shards

## SQL vs NoSQL

| Aspect | SQL | NoSQL |
|--------|-----|--------|
| Schema | Fixed schema | Flexible schema |
| ACID | Full ACID support | Eventually consistent |
| Scaling | Vertical scaling | Horizontal scaling |
| Queries | Complex SQL queries | Simple queries |
| Relationships | Strong relationships | Weak relationships |
| Use Cases | Complex transactions | Big data, real-time |

## Database Selection Criteria

### Data Structure
- **Structured**: SQL databases
- **Semi-structured**: Document databases
- **Unstructured**: Key-value or document stores

### Scalability Requirements
- **Read-heavy**: Read replicas, caching
- **Write-heavy**: Sharding, NoSQL
- **Both**: Distributed databases

### Consistency Requirements
- **Strong consistency**: SQL databases
- **Eventual consistency**: NoSQL databases
- **Mixed**: Hybrid approaches

### Query Complexity
- **Complex queries**: SQL databases
- **Simple lookups**: Key-value stores
- **Analytics**: Column-family or time-series

## Common Database Patterns

### Master-Slave Replication
```mermaid
sequenceDiagram
    participant App as Application
    participant Master as Master DB
    participant Slave1 as Slave DB 1
    participant Slave2 as Slave DB 2
    
    App->>Master: Write Request
    Master-->>App: Write Response
    Master->>Slave1: Replicate Data
    Master->>Slave2: Replicate Data
    
    App->>Slave1: Read Request
    Slave1-->>App: Read Response
```

### Master-Master Replication
- Multiple writable nodes
- Conflict resolution required
- Higher availability but complexity

### Federation (Functional Partitioning)
- Split databases by function
- Users DB, Products DB, Orders DB
- Reduces read/write traffic to each database

### Denormalization
- Trade storage for query performance
- Duplicate data to avoid joins
- Common in NoSQL and read-heavy systems

## Performance Optimization

### Indexing
```mermaid
graph TB
    Table[Table Data] --> Index1[B-Tree Index<br/>Primary Key]
    Table --> Index2[Hash Index<br/>Exact Matches]
    Table --> Index3[Bitmap Index<br/>Low Cardinality]
    Table --> Index4[Full-Text Index<br/>Text Search]
```

**Index Types:**
- **B-Tree**: Range queries, sorting
- **Hash**: Exact matches
- **Bitmap**: Low cardinality data
- **Full-text**: Text search

### Query Optimization
- Use appropriate indexes
- Avoid N+1 queries
- Optimize JOIN operations
- Use query execution plans

### Connection Pooling
```mermaid
graph LR
    App1[App Instance 1] --> Pool[Connection Pool<br/>Size: 10]
    App2[App Instance 2] --> Pool
    App3[App Instance 3] --> Pool
    
    Pool --> DB[Database<br/>Max Connections: 100]
```

### Caching Strategies
- **Cache-aside**: Application manages cache
- **Write-through**: Write to cache and database
- **Write-behind**: Write to cache first, database later
- **Refresh-ahead**: Proactively refresh cache

## Database Architecture Patterns

### Microservices Data Architecture
```mermaid
graph TB
    subgraph "User Service"
        US[User Service] --> UDB[User Database]
    end
    
    subgraph "Order Service"
        OS[Order Service] --> ODB[Order Database]
    end
    
    subgraph "Product Service"
        PS[Product Service] --> PDB[Product Database]
    end
    
    API[API Gateway] --> US
    API --> OS
    API --> PS
```

### CQRS (Command Query Responsibility Segregation)
```mermaid
graph TB
    Command[Command Side] --> WriteDB[Write Database<br/>Optimized for Writes]
    Query[Query Side] --> ReadDB[Read Database<br/>Optimized for Reads]
    
    WriteDB -.->|Event Stream| ReadDB
    
    App[Application] --> Command
    App --> Query
```

### Event Sourcing
```mermaid
sequenceDiagram
    participant App as Application
    participant ES as Event Store
    participant View as Read Model
    
    App->>ES: Store Event
    ES->>View: Project Event
    App->>View: Query State
    View-->>App: Current State
```

### Polyglot Persistence
- Use different databases for different purposes
- User profiles: Document database
- Analytics: Column-family database
- Caching: Key-value store
- Relationships: Graph database

## Database Security

### Access Control
- Authentication and authorization
- Role-based access control (RBAC)
- Principle of least privilege

### Data Protection
- Encryption at rest and in transit
- Data masking and anonymization
- Audit logging

### Network Security
- VPCs and private subnets
- Database firewalls
- SSL/TLS connections

## Monitoring and Maintenance

### Key Metrics
- **Performance**: Query response time, throughput
- **Availability**: Uptime, connection success rate
- **Resource Usage**: CPU, memory, disk I/O
- **Replication Lag**: Time delay in data replication

### Backup and Recovery
- **Full backups**: Complete database copy
- **Incremental backups**: Changes since last backup
- **Point-in-time recovery**: Restore to specific timestamp
- **Cross-region replication**: Disaster recovery

### Maintenance Tasks
- Index rebuilding and optimization
- Statistics updates
- Log file management
- Security patch updates

## Best Practices

1. **Choose the right database** for your use case
2. **Design for scalability** from the beginning
3. **Implement proper indexing** strategy
4. **Monitor performance** continuously
5. **Plan for disaster recovery**
6. **Secure sensitive data**
7. **Document database schema** and changes
8. **Test backup and recovery** procedures
9. **Consider data lifecycle** management
10. **Stay updated** with database versions and security patches

## Common Interview Questions

1. **How would you design a database for a social media platform?**
2. **When would you choose NoSQL over SQL?**
3. **How do you handle database scaling?**
4. **Explain database sharding and its challenges**
5. **How do you ensure data consistency in distributed systems?**
6. **What are the trade-offs between different database types?**
7. **How do you optimize database performance?**
8. **Explain ACID properties and their importance**
9. **How do you handle database migrations?**
10. **What are the security considerations for databases?**

## Advanced Database Concepts

### Database Transactions and Concurrency

#### Transaction Isolation Levels
```mermaid
graph TB
    A[Transaction Isolation Levels] --> B[Read Uncommitted]
    A --> C[Read Committed]
    A --> D[Repeatable Read]
    A --> E[Serializable]
    
    B --> B1[Dirty Reads Allowed]
    C --> C1[No Dirty Reads]
    D --> D1[No Phantom Reads]
    E --> E1[Complete Isolation]
```

- **Read Uncommitted**: Lowest isolation, allows dirty reads
- **Read Committed**: Prevents dirty reads, allows non-repeatable reads
- **Repeatable Read**: Prevents dirty and non-repeatable reads
- **Serializable**: Highest isolation, prevents all phenomena

#### Concurrency Control Mechanisms

**Optimistic Locking**
```mermaid
sequenceDiagram
    participant T1 as Transaction 1
    participant T2 as Transaction 2
    participant DB as Database
    
    T1->>DB: Read data (version 1)
    T2->>DB: Read data (version 1)
    T1->>DB: Update with version check
    DB-->>T1: Success (version 2)
    T2->>DB: Update with version check
    DB-->>T2: Failure (version mismatch)
```

**Pessimistic Locking**
```mermaid
sequenceDiagram
    participant T1 as Transaction 1
    participant T2 as Transaction 2
    participant DB as Database
    
    T1->>DB: Read with lock
    DB-->>T1: Data + Lock acquired
    T2->>DB: Read with lock
    Note over T2,DB: Waits for lock release
    T1->>DB: Update and release lock
    DB-->>T2: Data + Lock acquired
```

### Database Partitioning Strategies

#### Horizontal Partitioning (Sharding)
```mermaid
graph TB
    subgraph "Original Table"
        Table[Users Table<br/>1M records]
    end
    
    subgraph "Sharded Tables"
        Shard1[Shard 1<br/>Users 1-250K]
        Shard2[Shard 2<br/>Users 250K-500K]
        Shard3[Shard 3<br/>Users 500K-750K]
        Shard4[Shard 4<br/>Users 750K-1M]
    end
    
    Table --> Shard1
    Table --> Shard2
    Table --> Shard3
    Table --> Shard4
```

#### Vertical Partitioning
```mermaid
graph LR
    subgraph "Original Table"
        UserTable[Users<br/>ID, Name, Email, Profile, Preferences, Analytics]
    end
    
    subgraph "Partitioned Tables"
        Basic[Basic Info<br/>ID, Name, Email]
        Profile[Profile Data<br/>ID, Profile, Preferences]
        Analytics[Analytics<br/>ID, Analytics Data]
    end
    
    UserTable --> Basic
    UserTable --> Profile
    UserTable --> Analytics
```

### Data Modeling Patterns

#### Aggregation Pattern
Store pre-computed aggregations for faster queries:
```json
{
  "_id": "product_123",
  "name": "Laptop",
  "daily_stats": {
    "2024-01-15": {
      "views": 245,
      "purchases": 12,
      "revenue": 14388
    },
    "2024-01-16": {
      "views": 198,
      "purchases": 8,
      "revenue": 9592
    }
  }
}
```

#### Bucket Pattern
Group related data into buckets to optimize storage and queries:
```json
{
  "_id": "sensor_data_2024_01_15_00",
  "sensor_id": "temp_001",
  "bucket_start": "2024-01-15T00:00:00Z",
  "bucket_end": "2024-01-15T01:00:00Z",
  "measurements": [
    {"timestamp": "2024-01-15T00:00:15Z", "temperature": 22.5},
    {"timestamp": "2024-01-15T00:01:15Z", "temperature": 22.7},
    // ... more measurements
  ]
}
```

#### Schema Versioning Pattern
Handle schema evolution gracefully:
```json
{
  "_id": "user_123",
  "schema_version": "2.1",
  "name": "John Doe",
  "email": "john@example.com",
  "profile": {
    "created_at": "2024-01-15T10:00:00Z",
    "preferences": {
      "theme": "dark",
      "notifications": true
    }
  }
}
```

### Database Consistency Models

#### Strong Consistency
```mermaid
sequenceDiagram
    participant Client as Client
    participant Primary as Primary Node
    participant Secondary as Secondary Node
    
    Client->>Primary: Write Request
    Primary->>Secondary: Replicate Write
    Secondary-->>Primary: Acknowledge
    Primary-->>Client: Write Complete
    
    Note over Client,Secondary: All reads return latest write
```

#### Eventual Consistency
```mermaid
sequenceDiagram
    participant Client as Client
    participant Node1 as Node 1
    participant Node2 as Node 2
    participant Node3 as Node 3
    
    Client->>Node1: Write Request
    Node1-->>Client: Write Complete
    
    Node1->>Node2: Async Replication
    Node1->>Node3: Async Replication
    
    Note over Client,Node3: Reads may return stale data temporarily
```

### Advanced Indexing Strategies

#### Composite Indexes
```sql
-- Multi-column index for common query patterns
CREATE INDEX idx_user_activity ON user_logs(user_id, timestamp, action_type);

-- Query that uses the composite index efficiently
SELECT * FROM user_logs 
WHERE user_id = 12345 
  AND timestamp BETWEEN '2024-01-01' AND '2024-01-31'
  AND action_type = 'login';
```

#### Partial Indexes
```sql
-- Index only active users to save space
CREATE INDEX idx_active_users ON users(last_login) 
WHERE status = 'active';

-- Index only recent data
CREATE INDEX idx_recent_orders ON orders(created_at) 
WHERE created_at > '2024-01-01';
```

#### Covering Indexes
```sql
-- Index includes all columns needed for query
CREATE INDEX idx_user_summary ON users(user_id) 
INCLUDE (name, email, created_at);

-- Query can be satisfied entirely from index
SELECT user_id, name, email, created_at 
FROM users WHERE user_id = 12345;
```

### Database Performance Tuning

#### Query Execution Plans
```mermaid
graph TB
    A[Query] --> B[Parser]
    B --> C[Query Optimizer]
    C --> D[Execution Plan]
    D --> E[Plan Cache]
    D --> F[Execution Engine]
    F --> G[Results]
    
    C --> C1[Statistics]
    C --> C2[Indexes]
    C --> C3[Cost Model]
```

#### Database Statistics
- **Cardinality**: Number of distinct values
- **Selectivity**: Fraction of rows returned by predicate
- **Distribution**: How values are distributed
- **Correlation**: Relationship between columns

#### Performance Monitoring Queries
```sql
-- Find slow queries
SELECT query_id, mean_time, calls, total_time
FROM pg_stat_statements 
ORDER BY mean_time DESC 
LIMIT 10;

-- Index usage statistics
SELECT schemaname, tablename, indexname, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes 
ORDER BY idx_tup_read DESC;

-- Table size analysis
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size,
    pg_total_relation_size(schemaname||'.'||tablename) as size_bytes
FROM pg_tables 
ORDER BY size_bytes DESC;
```

### Distributed Database Patterns

#### Multi-Master Replication
```mermaid
graph TB
    subgraph "Region 1"
        M1[Master 1] --> S1[Slave 1]
        M1 --> S2[Slave 2]
    end
    
    subgraph "Region 2"
        M2[Master 2] --> S3[Slave 3]
        M2 --> S4[Slave 4]
    end
    
    M1 <-.->|Bidirectional<br/>Replication| M2
```

#### Consensus Algorithms
```mermaid
sequenceDiagram
    participant L as Leader
    participant F1 as Follower 1
    participant F2 as Follower 2
    participant F3 as Follower 3
    
    L->>F1: Propose Value
    L->>F2: Propose Value
    L->>F3: Propose Value
    
    F1-->>L: Accept
    F2-->>L: Accept
    F3-->>L: Reject
    
    Note over L,F3: Majority agrees, value committed
    
    L->>F1: Commit Value
    L->>F2: Commit Value
    L->>F3: Commit Value
```

### Database Security Deep Dive

#### Row-Level Security (RLS)
```sql
-- Enable RLS on table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Create policy for user access
CREATE POLICY user_orders ON orders
FOR ALL TO application_users
USING (user_id = current_setting('app.current_user_id')::int);

-- Users can only see their own orders
SELECT * FROM orders; -- Automatically filtered
```

#### Column-Level Encryption
```sql
-- Encrypt sensitive columns
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255),
    encrypted_ssn BYTEA, -- Encrypted at application level
    ssn_key_id UUID      -- Reference to encryption key
);
```

#### Database Auditing
```sql
-- Create audit log table
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    table_name VARCHAR(50),
    operation VARCHAR(10),
    old_values JSONB,
    new_values JSONB,
    user_id INTEGER,
    timestamp TIMESTAMPTZ DEFAULT NOW()
);

-- Trigger function for auditing
CREATE OR REPLACE FUNCTION audit_trigger_function()
RETURNS TRIGGER AS $
BEGIN
    INSERT INTO audit_log (table_name, operation, old_values, new_values, user_id)
    VALUES (TG_TABLE_NAME, TG_OP, 
            CASE WHEN TG_OP = 'DELETE' THEN row_to_json(OLD) ELSE NULL END,
            CASE WHEN TG_OP = 'INSERT' THEN row_to_json(NEW) 
                 WHEN TG_OP = 'UPDATE' THEN row_to_json(NEW) 
                 ELSE NULL END,
            current_setting('app.current_user_id')::int);
    RETURN COALESCE(NEW, OLD);
END;
$ LANGUAGE plpgsql;
```

### Database Testing Strategies

#### Unit Testing Database Code
```python
import pytest
from unittest.mock import Mock, patch
from your_app.database import UserRepository

class TestUserRepository:
    def test_create_user_success(self):
        # Arrange
        mock_db = Mock()
        repo = UserRepository(mock_db)
        user_data = {"name": "John", "email": "john@test.com"}
        
        # Act
        result = repo.create_user(user_data)
        
        # Assert
        mock_db.execute.assert_called_once()
        assert result["name"] == "John"
    
    def test_find_user_not_found(self):
        # Arrange
        mock_db = Mock()
        mock_db.fetchone.return_value = None
        repo = UserRepository(mock_db)
        
        # Act & Assert
        with pytest.raises(UserNotFoundError):
            repo.find_user(999)
```

#### Integration Testing
```python
@pytest.fixture
def test_database():
    # Setup test database
    engine = create_engine("postgresql://test:test@localhost/testdb")
    TestBase.metadata.create_all(engine)
    session = sessionmaker(bind=engine)()
    
    yield session
    
    # Cleanup
    session.close()
    TestBase.metadata.drop_all(engine)

def test_user_creation_integration(test_database):
    # Test actual database operations
    user = User(name="Test User", email="test@example.com")
    test_database.add(user)
    test_database.commit()
    
    retrieved_user = test_database.query(User).filter_by(email="test@example.com").first()
    assert retrieved_user.name == "Test User"
```

### Database Migration Strategies

#### Zero-Downtime Migrations
```mermaid
graph TB
    A[Current Schema v1] --> B[Add New Column<br/>with Default Value]
    B --> C[Deploy App Code<br/>Handles Both Schemas]
    C --> D[Migrate Data<br/>Populate New Column]
    D --> E[Update App Code<br/>Use New Column Only]
    E --> F[Remove Old Column<br/>Schema v2 Complete]
```

#### Migration Scripts Example
```sql
-- migrations/001_add_user_status.sql
BEGIN;

-- Add new column with default value
ALTER TABLE users ADD COLUMN status VARCHAR(20) DEFAULT 'active';

-- Create index for new column
CREATE INDEX CONCURRENTLY idx_users_status ON users(status);

-- Update migration tracking
INSERT INTO schema_migrations (version, applied_at) 
VALUES (1, NOW());

COMMIT;
```

#### Rollback Strategy
```sql
-- rollbacks/001_remove_user_status.sql
BEGIN;

-- Remove index
DROP INDEX IF EXISTS idx_users_status;

-- Remove column
ALTER TABLE users DROP COLUMN IF EXISTS status;

-- Update migration tracking
DELETE FROM schema_migrations WHERE version = 1;

COMMIT;
```

### Data Warehousing and Analytics

#### Star Schema Design
```mermaid
graph TB
    subgraph "Fact Table"
        Sales[Sales Fact<br/>- sale_id<br/>- product_id<br/>- customer_id<br/>- date_id<br/>- store_id<br/>- quantity<br/>- revenue]
    end
    
    subgraph "Dimension Tables"
        Product[Product Dimension<br/>- product_id<br/>- name<br/>- category<br/>- brand]
        
        Customer[Customer Dimension<br/>- customer_id<br/>- name<br/>- age<br/>- location]
        
        Date[Date Dimension<br/>- date_id<br/>- date<br/>- month<br/>- quarter<br/>- year]
        
        Store[Store Dimension<br/>- store_id<br/>- name<br/>- location<br/>- manager]
    end
    
    Sales --> Product
    Sales --> Customer
    Sales --> Date
    Sales --> Store
```

#### ETL Process
```mermaid
graph LR
    subgraph "Extract"
        DB1[Source DB 1]
        DB2[Source DB 2]
        API[External API]
        Files[CSV Files]
    end
    
    subgraph "Transform"
        Clean[Data Cleaning]
        Valid[Validation]
        Agg[Aggregation]
        Enrich[Data Enrichment]
    end
    
    subgraph "Load"
        DW[Data Warehouse]
        Mart[Data Marts]
    end
    
    DB1 --> Clean
    DB2 --> Valid
    API --> Agg
    Files --> Enrich
    
    Clean --> DW
    Valid --> DW
    Agg --> Mart
    Enrich --> Mart
```

### Database Disaster Recovery

#### Recovery Point Objective (RPO) vs Recovery Time Objective (RTO)
```mermaid
graph TB
    subgraph "RPO (Data Loss)"
        RPO1[RPO = 0<br/>Synchronous Replication<br/>No Data Loss]
        RPO2[RPO = 5 min<br/>Frequent Backups<br/>Max 5 min data loss]
        RPO3[RPO = 24 hours<br/>Daily Backups<br/>Max 24 hour data loss]
    end
    
    subgraph "RTO (Downtime)"
        RTO1[RTO = 0<br/>Hot Standby<br/>Instant Failover]
        RTO2[RTO = 1 hour<br/>Warm Standby<br/>Quick Recovery]
        RTO3[RTO = 24 hours<br/>Cold Backup<br/>Manual Recovery]
    end
```

#### Backup Strategies
```bash
#!/bin/bash
# Full backup script
pg_dump -h localhost -U postgres -d myapp > backup_$(date +%Y%m%d_%H%M%S).sql

# Incremental backup using WAL archiving
# In postgresql.conf:
# wal_level = replica
# archive_mode = on
# archive_command = 'cp %p /backup/wal/%f'

# Point-in-time recovery
pg_basebackup -h localhost -D /backup/base -U postgres -P -W

# Continuous archiving
# Recovery command in recovery.conf:
# restore_command = 'cp /backup/wal/%f %p'
# recovery_target_time = '2024-01-15 10:30:00'
```

### Database Observability

#### Metrics to Monitor
```mermaid
graph TB
    subgraph "Performance Metrics"
        QPS[Queries Per Second]
        Latency[Query Latency]
        Throughput[Throughput]
    end
    
    subgraph "Resource Metrics"
        CPU[CPU Usage]
        Memory[Memory Usage]
        Disk[Disk I/O]
        Network[Network I/O]
    end
    
    subgraph "Database Metrics"
        Connections[Active Connections]
        LockWaits[Lock Waits]
        DeadLocks[Deadlocks]
        BufferHit[Buffer Hit Ratio]
    end
    
    subgraph "Business Metrics"
        ErrorRate[Error Rate]
        Availability[Availability]
        DataGrowth[Data Growth Rate]
    end
```

#### Alerting Rules
```yaml
# Example Prometheus alerting rules
groups:
- name: database_alerts
  rules:
  - alert: HighDatabaseConnections
    expr: postgres_connections_active / postgres_connections_max > 0.8
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High database connection usage"
      description: "Database connections are above 80% of maximum"
      
  - alert: SlowQueries
    expr: postgres_query_duration_seconds{quantile="0.95"} > 5
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Slow database queries detected"
      description: "95th percentile query time is above 5 seconds"
```

### Emerging Database Technologies

#### NewSQL Databases
- **Examples**: CockroachDB, TiDB, Spanner
- **Benefits**: ACID + Horizontal Scaling
- **Use Cases**: Global applications, financial systems

#### Multi-Model Databases
- **Examples**: ArangoDB, CosmosDB, OrientDB
- **Benefits**: Support multiple data models in one system
- **Use Cases**: Complex applications with varied data types

#### Serverless Databases
- **Examples**: Aurora Serverless, FaunaDB, PlanetScale
- **Benefits**: Auto-scaling, pay-per-use
- **Use Cases**: Variable workloads, development environments

## Resources

- [Database Internals by Alex Petrov](https://www.databass.dev/)
- [Designing Data-Intensive Applications by Martin Kleppmann](https://dataintensive.net/)
- [SQL Performance Explained](https://sql-performance-explained.com/)
- [NoSQL Databases Explained](https://www.mongodb.com/nosql-explained)
- [CAP Theorem Explained](https://www.ibm.com/cloud/learn/cap-theorem)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [MongoDB Manual](https://docs.mongodb.com/manual/)
- [Redis Documentation](https://redis.io/documentation)
- [Cassandra Documentation](https://cassandra.apache.org/doc/)
- [Database Reliability Engineering](https://www.oreilly.com/library/view/database-reliability-engineering/9781491925935/)

## Practice Exercises

1. **Design a database schema** for a multi-tenant SaaS application
2. **Implement sharding logic** for a user table with 100M records
3. **Create a data migration strategy** for adding a new required column
4. **Design a backup and recovery plan** for a critical e-commerce database
5. **Optimize slow queries** using execution plans and indexing strategies
6. **Implement database monitoring** with custom metrics and alerting
7. **Design a data warehouse schema** for business analytics
8. **Create a disaster recovery plan** with RTO/RPO requirements
9. **Implement database security** with encryption and access controls
10. **Design a polyglot persistence architecture** for a microservices system

---

Remember: The best database choice depends on your specific requirements, data patterns, scalability needs, and consistency requirements. Always consider the trade-offs and plan for future growth. Database decisions are often the hardest to change later, so invest time upfront in understanding your requirements and constraints.