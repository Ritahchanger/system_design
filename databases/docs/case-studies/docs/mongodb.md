# MongoDB Case Study

> **Part of**: [Database Case Studies](../README.md) | **Related**: [Scaling Strategies](../../scaling_strategies.md), [NoSQL Databases](../../nosql.md)

## Table of Contents
- [Overview](#overview)
- [Architecture Deep Dive](#architecture-deep-dive)
- [Data Modeling](#data-modeling)
- [Scaling Strategies](#scaling-strategies)
- [Performance Optimization](#performance-optimization)
- [Operational Considerations](#operational-considerations)
- [Real-World Examples](#real-world-examples)
- [When to Choose MongoDB](#when-to-choose-mongodb)

## Overview

MongoDB is a document-oriented NoSQL database that stores data in flexible, JSON-like documents. It provides rich querying capabilities, automatic failover, and horizontal scaling through sharding.

### Key Characteristics
- **Document Model**: Stores data as BSON (Binary JSON) documents
- **Dynamic Schema**: No predefined schema requirements
- **Rich Queries**: SQL-like querying with support for complex operations
- **Indexing**: Comprehensive indexing capabilities including compound, geospatial, and text indexes
- **Horizontal Scaling**: Built-in sharding support for distributing data

### Core Strengths
- Flexible schema evolution
- Natural object mapping for applications
- Rich query language and aggregation framework
- Strong consistency within replica sets
- Extensive tooling and ecosystem

### Limitations
- Memory-intensive for large datasets
- Limited multi-document transaction support
- Complex sharding key selection
- Storage overhead compared to relational databases

## Architecture Deep Dive

### MongoDB Deployment Topologies

```mermaid
graph TB
    subgraph "Single Replica Set"
        Primary1[Primary Node<br/>Read/Write]
        Secondary1[Secondary Node<br/>Read-only]
        Secondary2[Secondary Node<br/>Read-only]
        
        Primary1 -->|Replication| Secondary1
        Primary1 -->|Replication| Secondary2
    end
    
    subgraph "Sharded Cluster"
        Client[MongoDB Client] --> Mongos[Mongos Router]
        
        Mongos --> Config[Config Server<br/>Replica Set<br/>Metadata Storage]
        
        Mongos --> Shard1[Shard 1<br/>Replica Set]
        Mongos --> Shard2[Shard 2<br/>Replica Set]
        Mongos --> Shard3[Shard 3<br/>Replica Set]
        
        Shard1 --> RS1P[Primary]
        Shard1 --> RS1S1[Secondary]
        Shard1 --> RS1S2[Secondary]
    end
    
    style Primary1 fill:#FF6B6B
    style Mongos fill:#4ECDC4
    style Config fill:#45B7D1
    style Shard1 fill:#96CEB4
```

### Storage Engine Architecture

```mermaid
graph TB
    subgraph "WiredTiger Storage Engine"
        Application[Application Layer] --> MongoDB[MongoDB Server]
        
        MongoDB --> Cache[WiredTiger Cache<br/>In-Memory Pages]
        MongoDB --> Journal[Journal<br/>Write-Ahead Log]
        
        Cache --> Checkpoint[Checkpoint Process<br/>Periodic Flush to Disk]
        Journal --> Recovery[Crash Recovery<br/>Replay Operations]
        
        Checkpoint --> DataFiles[Data Files<br/>B-Tree Structure]
        Recovery --> DataFiles
        
        subgraph "Compression & Encryption"
            Compression[Block Compression<br/>Snappy/Zlib/Zstd]
            Encryption[Encryption at Rest<br/>AES-256]
        end
        
        DataFiles --> Compression
        DataFiles --> Encryption
    end
    
    style Cache fill:#90EE90
    style Journal fill:#F0E68C
    style DataFiles fill:#87CEEB
    style Compression fill:#FFB6C1
```

### Replication and Consistency

```mermaid
sequenceDiagram
    participant Client
    participant Primary
    participant Secondary1
    participant Secondary2
    participant Arbiter
    
    Note over Client,Arbiter: Write Operation
    Client->>Primary: Write Request
    Primary->>Primary: Write to Journal
    Primary->>Primary: Apply to Data
    Primary->>Secondary1: Replicate (async)
    Primary->>Secondary2: Replicate (async)
    Primary-->>Client: Acknowledge Write
    
    Secondary1->>Primary: Acknowledge Replication
    Secondary2->>Primary: Acknowledge Replication
    
    Note over Client,Arbiter: Primary Election (Failure Scenario)
    Primary->>Primary: Primary Fails
    Secondary1->>Arbiter: Request Vote
    Secondary2->>Arbiter: Request Vote
    Arbiter->>Secondary1: Vote Granted
    Secondary2->>Secondary1: Vote Granted
    Secondary1->>Secondary1: Become Primary
    
    Note over Secondary1: New primary elected
```

## Data Modeling

### Document Structure Design

MongoDB's document model allows for rich, nested data structures:

```javascript
// User profile with embedded and referenced data
const userSchema = {
  _id: ObjectId("..."),
  username: "john_doe",
  email: "john@example.com",
  profile: {
    firstName: "John",
    lastName: "Doe",
    dateOfBirth: ISODate("1990-01-15"),
    avatar: "https://cdn.example.com/avatars/john.jpg",
    bio: "Software developer passionate about databases",
    location: {
      city: "San Francisco",
      state: "CA",
      country: "USA",
      coordinates: {
        type: "Point",
        coordinates: [-122.4194, 37.7749]
      }
    }
  },
  preferences: {
    theme: "dark",
    notifications: {
      email: true,
      push: false,
      sms: false
    },
    privacy: {
      profileVisible: true,
      showLocation: false
    }
  },
  social: {
    followers: 1250,
    following: 890,
    posts: 342
  },
  createdAt: ISODate("2023-01-15T10:30:00Z"),
  lastActive: ISODate("2024-01-15T14:22:33Z"),
  tags: ["developer", "mongodb", "javascript", "node.js"],
  metadata: {
    source: "web",
    referrer: "google.com",
    experiments: ["new_ui", "recommendation_v2"]
  }
};
```

### Schema Design Patterns

#### Embedded Documents vs References

```javascript
// Pattern 1: Embedded Documents (One-to-Few)
const blogPostEmbedded = {
  _id: ObjectId("..."),
  title: "MongoDB Schema Design Patterns",
  content: "Content goes here...",
  author: {
    _id: ObjectId("..."),
    name: "John Doe",
    email: "john@example.com"
  },
  comments: [
    {
      _id: ObjectId("..."),
      author: "Jane Smith",
      content: "Great article!",
      createdAt: ISODate("2024-01-15T10:30:00Z")
    },
    {
      _id: ObjectId("..."),
      author: "Bob Johnson",
      content: "Very helpful, thanks!",
      createdAt: ISODate("2024-01-15T11:45:00Z")
    }
  ],
  tags: ["mongodb", "database", "nosql"],
  createdAt: ISODate("2024-01-15T09:00:00Z")
};

// Pattern 2: References (One-to-Many)
const blogPostReferenced = {
  _id: ObjectId("..."),
  title: "MongoDB Schema Design Patterns",
  content: "Content goes here...",
  authorId: ObjectId("..."),  // Reference to users collection
  commentIds: [               // References to comments collection
    ObjectId("..."),
    ObjectId("...")
  ],
  tags: ["mongodb", "database", "nosql"],
  createdAt: ISODate("2024-01-15T09:00:00Z")
};

// Pattern 3: Hybrid Approach
const blogPostHybrid = {
  _id: ObjectId("..."),
  title: "MongoDB Schema Design Patterns",
  content: "Content goes here...",
  author: {
    _id: ObjectId("..."),
    name: "John Doe",        // Embedded for performance
    avatar: "avatar.jpg"
  },
  commentCount: 15,          // Denormalized count
  lastComments: [            // Most recent comments embedded
    {
      author: "Jane Smith",
      content: "Great article!",
      createdAt: ISODate("2024-01-15T10:30:00Z")
    }
  ],
  tags: ["mongodb", "database", "nosql"],
  createdAt: ISODate("2024-01-15T09:00:00Z")
};
```

### Advanced Schema Patterns

#### Bucket Pattern for Time Series

```javascript
// Temperature sensor data using bucket pattern
const sensorBucket = {
  _id: ObjectId("..."),
  sensorId: "sensor_001",
  date: ISODate("2024-01-15T00:00:00Z"), // Bucket start time
  measurements: [
    {
      timestamp: ISODate("2024-01-15T00:00:00Z"),
      temperature: 23.5,
      humidity: 65.2,
      pressure: 1013.25
    },
    {
      timestamp: ISODate("2024-01-15T00:01:00Z"),
      temperature: 23.6,
      humidity: 65.1,
      pressure: 1013.30
    }
    // ... more measurements (up to 1 hour of data)
  ],
  metaData: {
    location: "Building A, Floor 3",
    deviceType: "DHT22",
    firmware: "1.2.3"
  },
  summary: {
    count: 60,
    avgTemperature: 23.7,
    minTemperature: 23.1,
    maxTemperature: 24.3
  }
};

// Query operations for bucket pattern
async function getBucketPatternOperations() {
  const db = client.db('iot_platform');
  const collection = db.collection('sensor_data');

  // Insert new measurement
  await collection.updateOne(
    {
      sensorId: "sensor_001",
      date: new Date("2024-01-15T00:00:00Z")
    },
    {
      $push: {
        measurements: {
          timestamp: new Date(),
          temperature: 24.1,
          humidity: 64.8,
          pressure: 1013.45
        }
      },
      $inc: { "summary.count": 1 },
      $max: { "summary.maxTemperature": 24.1 },
      $min: { "summary.minTemperature": 24.1 }
    },
    { upsert: true }
  );

  // Query recent data
  const recentData = await collection.find({
    sensorId: "sensor_001",
    date: { 
      $gte: new Date(Date.now() - 24 * 60 * 60 * 1000) // Last 24 hours
    }
  }).toArray();

  return recentData;
}
```

#### Subset Pattern for Large Documents

```javascript
// Product catalog with subset pattern
const productSummary = {
  _id: ObjectId("..."),
  sku: "LAPTOP-001",
  name: "Gaming Laptop Pro",
  price: 1299.99,
  category: "Electronics",
  brand: "TechBrand",
  rating: 4.7,
  reviewCount: 1250,
  primaryImage: "laptop-main.jpg",
  availability: {
    inStock: true,
    quantity: 45
  },
  keyFeatures: [
    "Intel i7 processor",
    "32GB RAM",
    "1TB NVMe SSD",
    "RTX 4070 GPU"
  ],
  createdAt: ISODate("2024-01-15T09:00:00Z")
};

const productDetails = {
  _id: ObjectId("..."), // Same as productSummary._id
  fullDescription: "Long detailed description...",
  specifications: {
    processor: {
      brand: "Intel",
      model: "Core i7-13700H",
      cores: 14,
      baseSpeed: "2.4 GHz",
      maxSpeed: "5.0 GHz"
    },
    memory: {
      size: "32GB",
      type: "DDR5",
      speed: "4800MHz"
    },
    storage: {
      primary: {
        type: "NVMe SSD",
        capacity: "1TB",
        interface: "PCIe 4.0"
      }
    }
    // ... extensive specifications
  },
  images: [
    {
      url: "laptop-main.jpg",
      alt: "Front view",
      type: "main"
    },
    {
      url: "laptop-side.jpg",
      alt: "Side view",
      type: "gallery"
    }
    // ... many more images
  ],
  reviews: ObjectId("..."), // Reference to reviews collection
  manuals: [
    {
      name: "User Manual",
      url: "manuals/laptop-001-manual.pdf",
      language: "en"
    }
  ]
};

// Usage pattern
async function getProductWithDetails(productId) {
  const db = client.db('ecommerce');
  
  // First, get summary for quick display
  const summary = await db.collection('products').findOne({ _id: productId });
  
  // Then, get details if needed
  const details = await db.collection('product_details').findOne({ _id: productId });
  
  return { summary, details };
}
```

## Scaling Strategies

### Replica Set Configuration

```javascript
// Replica set initialization
const { MongoClient } = require('mongodb');

async function setupReplicaSet() {
  const client = new MongoClient('mongodb://localhost:27017');
  
  try {
    await client.connect();
    const admin = client.db('admin');
    
    // Initialize replica set
    const config = {
      _id: "myReplicaSet",
      members: [
        { _id: 0, host: "mongodb-primary:27017", priority: 2 },
        { _id: 1, host: "mongodb-secondary1:27017", priority: 1 },
        { _id: 2, host: "mongodb-secondary2:27017", priority: 1 },
        { _id: 3, host: "mongodb-arbiter:27017", arbiterOnly: true }
      ]
    };
    
    await admin.command({ replSetInitiate: config });
    
    console.log('Replica set initialized');
  } finally {
    await client.close();
  }
}

// Read preference configuration
async function configureReadPreferences() {
  const client = new MongoClient('mongodb://mongodb-primary:27017,mongodb-secondary1:27017,mongodb-secondary2:27017', {
    replicaSet: 'myReplicaSet',
    readPreference: 'secondaryPreferred', // Prefer secondaries for reads
    readConcern: { level: 'majority' }     // Wait for majority acknowledgment
  });
  
  await client.connect();
  const db = client.db('myapp');
  
  // Different read preferences for different operations
  const collection = db.collection('users');
  
  // Critical reads from primary
  const criticalData = await collection.findOne(
    { _id: userId },
    { readPreference: 'primary' }
  );
  
  // Analytics queries from secondaries
  const analyticsData = await collection.aggregate([
    { $group: { _id: "$region", count: { $sum: 1 } } }
  ], { readPreference: 'secondary' }).toArray();
  
  return { criticalData, analyticsData };
}
```

### Sharding Implementation

```javascript
// Enable sharding for database and collection
async function setupSharding() {
  const client = new MongoClient('mongodb://mongos:27017');
  
  try {
    await client.connect();
    const admin = client.db('admin');
    
    // Enable sharding for database
    await admin.command({ enableSharding: 'ecommerce' });
    
    // Create index for shard key
    const db = client.db('ecommerce');
    await db.collection('orders').createIndex({ customerId: 1, orderDate: 1 });
    
    // Shard collection
    await admin.command({
      shardCollection: 'ecommerce.orders',
      key: { customerId: 1, orderDate: 1 } // Compound shard key
    });
    
    // Configure zones for geographic distribution
    await admin.command({
      addShardToZone: 'shard-us-east',
      zone: 'us-east'
    });
    
    await admin.command({
      addShardToZone: 'shard-us-west',
      zone: 'us-west'
    });
    
    // Create zone ranges
    await admin.command({
      updateZoneKeyRange: 'ecommerce.orders',
      min: { customerId: MinKey, orderDate: MinKey },
      max: { customerId: 'us-east-max', orderDate: MaxKey },
      zone: 'us-east'
    });
    
    console.log('Sharding configured successfully');
  } finally {
    await client.close();
  }
}

// Shard key selection strategies
const shardKeyExamples = {
  // Hash-based sharding for even distribution
  hashBased: {
    shardKey: { _id: "hashed" },
    pros: ["Even distribution", "Good write distribution"],
    cons: ["No range queries", "Poor sort performance"]
  },
  
  // Range-based sharding for queries
  rangeBased: {
    shardKey: { customerId: 1, timestamp: 1 },
    pros: ["Good for range queries", "Sorted queries efficient"],
    cons: ["Potential hot spotting", "Uneven growth"]
  },
  
  // Tag-aware sharding for multi-tenancy
  tagAware: {
    shardKey: { tenantId: 1, _id: 1 },
    pros: ["Tenant isolation", "Compliance friendly"],
    cons: ["Complex balancing", "Uneven tenant sizes"]
  }
};
```

### Auto-scaling with MongoDB Atlas

```javascript
// MongoDB Atlas cluster auto-scaling configuration
const atlasConfig = {
  clusterName: "production-cluster",
  providerSettings: {
    providerName: "AWS",
    regionName: "US_EAST_1",
    instanceSizeName: "M30" // Starting size
  },
  autoScaling: {
    diskGBEnabled: true,
    compute: {
      enabled: true,
      scaleDownEnabled: true,
      minInstanceSize: "M10",
      maxInstanceSize: "M80"
    }
  },
  replicationSpecs: [
    {
      numShards: 1,
      regionsConfig: {
        "US_EAST_1": {
          analyticsNodes: 1,
          electableNodes: 3,
          priority: 7,
          readOnlyNodes: 0
        },
        "US_WEST_2": {
          analyticsNodes: 0,
          electableNodes: 2,
          priority: 6,
          readOnlyNodes: 0
        }
      },
      zoneName: "Zone 1"
    }
  ]
};

// Connection with automatic failover
async function createResilientConnection() {
  const connectionString = 'mongodb+srv://username:password@cluster.mongodb.net/mydb?retryWrites=true&w=majority';
  
  const client = new MongoClient(connectionString, {
    maxPoolSize: 10,
    serverSelectionTimeoutMS: 5000,
    socketTimeoutMS: 45000,
    bufferMaxEntries: 0,
    useNewUrlParser: true,
    useUnifiedTopology: true
  });
  
  // Handle connection events
  client.on('serverDescriptionChanged', (event) => {
    console.log('Server description changed:', event);
  });
  
  client.on('topologyDescriptionChanged', (event) => {
    console.log('Topology changed:', event);
  });
  
  return client;
}
```

## Performance Optimization

### Indexing Strategies

```javascript
async function createOptimizedIndexes() {
  const db = client.db('ecommerce');
  const products = db.collection('products');
  const orders = db.collection('orders');
  
  // Compound indexes for common query patterns
  await products.createIndex(
    { category: 1, price: 1, rating: -1 },
    { 
      name: 'category_price_rating_idx',
      background: true // Non-blocking index build
    }
  );
  
  // Text search index
  await products.createIndex(
    {
      name: 'text',
      description: 'text',
      'specifications.$.value': 'text'
    },
    {
      name: 'product_text_search',
      weights: {
        name: 10,
        description: 5,
        'specifications.$.value': 1
      }
    }
  );
  
  // Geospatial index for location-based queries
  await db.collection('stores').createIndex(
    { location: '2dsphere' },
    { name: 'store_location_idx' }
  );
  
  // TTL index for session data
  await db.collection('sessions').createIndex(
    { expiresAt: 1 },
    { 
      name: 'session_ttl_idx',
      expireAfterSeconds: 0 // Use document's expiresAt field
    }
  );
  
  // Partial index for specific conditions
  await orders.createIndex(
    { customerId: 1, orderDate: -1 },
    {
      name: 'active_orders_idx',
      partialFilterExpression: {
        status: { $in: ['pending', 'processing', 'shipped'] }
      }
    }
  );
  
  // Sparse index for optional fields
  await db.collection('users').createIndex(
    { 'socialMedia.twitter': 1 },
    { 
      name: 'twitter_handle_idx',
      sparse: true // Only index documents with this field
    }
  );
}

// Index usage analysis
async function analyzeIndexUsage() {
  const db = client.db('ecommerce');
  const collection = db.collection('products');
  
  // Get index statistics
  const indexStats = await collection.aggregate([
    { $indexStats: {} }
  ]).toArray();
  
  console.log('Index usage statistics:');
  indexStats.forEach(stat => {
    console.log(`Index: ${stat.name}`);
    console.log(`Operations: ${stat.accesses.ops}`);
    console.log(`Since: ${stat.accesses.since}\n`);
  });
  
  // Explain query execution
  const explanation = await collection.find({
    category: 'Electronics',
    price: { $gte: 100, $lte: 500 }
  }).explain('executionStats');
  
  console.log('Query execution stats:');
  console.log(`Documents examined: ${explanation.executionStats.totalDocsExamined}`);
  console.log(`Documents returned: ${explanation.executionStats.totalDocsReturned}`);
  console.log(`Execution time: ${explanation.executionStats.executionTimeMillis}ms`);
  
  return { indexStats, explanation };
}
```

### Aggregation Pipeline Optimization

```javascript
// Optimized aggregation pipelines
async function createOptimizedPipelines() {
  const db = client.db('ecommerce');
  const orders = db.collection('orders');
  
  // Sales analytics with optimization
  const salesAnalytics = await orders.aggregate([
    // Match first to reduce document set early
    {
      $match: {
        orderDate: {
          $gte: new Date('2024-01-01'),
          $lt: new Date('2024-02-01')
        },
        status: 'completed'
      }
    },
    
    // Project only needed fields to reduce memory usage
    {
      $project: {
        customerId: 1,
        orderDate: 1,
        totalAmount: 1,
        region: 1,
        items: {
          $slice: ['$items', 10] // Limit items if not all needed
        }
      }
    },
    
    // Add computed fields
    {
      $addFields: {
        year: { $year: '$orderDate' },
        month: { $month: '$orderDate' },
        dayOfWeek: { $dayOfWeek: '$orderDate' }
      }
    },
    
    // Group by multiple dimensions
    {
      $group: {
        _id: {
          region: '$region',
          month: '$month',
          dayOfWeek: '$dayOfWeek'
        },
        totalSales: { $sum: '$totalAmount' },
        orderCount: { $sum: 1 },
        avgOrderValue: { $avg: '$totalAmount' },
        uniqueCustomers: { $addToSet: '$customerId' }
      }
    },
    
    // Add calculated fields
    {
      $addFields: {
        uniqueCustomerCount: { $size: '$uniqueCustomers' }
      }
    },
    
    // Sort by total sales
    {
      $sort: { totalSales: -1 }
    },
    
    // Limit results
    {
      $limit: 50
    },
    
    // Final projection for clean output
    {
      $project: {
        _id: 0,
        region: '$_id.region',
        month: '$_id.month',
        dayOfWeek: '$_id.dayOfWeek',
        totalSales: { $round: ['$totalSales', 2] },
        orderCount: 1,
        avgOrderValue: { $round: ['$avgOrderValue', 2] },
        uniqueCustomerCount: 1
      }
    }
  ], {
    allowDiskUse: true, // Enable for large datasets
    hint: 'orderDate_region_status_idx' // Force specific index
  }).toArray();
  
  return salesAnalytics;
}

// Real-time aggregation with change streams
async function setupRealTimeAnalytics() {
  const db = client.db('ecommerce');
  const orders = db.collection('orders');
  
  // Watch for new orders
  const changeStream = orders.watch([
    {
      $match: {
        operationType: 'insert',
        'fullDocument.status': 'completed'
      }
    }
  ]);
  
  changeStream.on('change', async (change) => {
    const order = change.fullDocument;
    
    // Update real-time metrics
    await db.collection('metrics').updateOne(
      {
        _id: 'daily_sales',
        date: new Date().toISOString().split('T')[0]
      },
      {
        $inc: {
          totalSales: order.totalAmount,
          orderCount: 1
        },
        $addToSet: {
          customers: order.customerId
        }
      },
      { upsert: true }
    );
    
    console.log(`Updated metrics for order: ${order._id}`);
  });
  
  return changeStream;
}
```

### Connection Pool Management

```javascript
class MongoConnectionManager {
  constructor(uri, options = {}) {
    this.uri = uri;
    this.options = {
      maxPoolSize: 20,
      minPoolSize: 5,
      maxIdleTimeMS: 30000,
      waitQueueTimeoutMS: 5000,
      serverSelectionTimeoutMS: 5000,
      ...options
    };
    this.client = null;
    this.isConnected = false;
  }
  
  async connect() {
    if (this.isConnected) return this.client;
    
    try {
      this.client = new MongoClient(this.uri, this.options);
      await this.client.connect();
      
      // Set up event listeners
      this.client.on('connectionPoolCreated', (event) => {
        console.log('Connection pool created:', event);
      });
      
      this.client.on('connectionCheckedOut', (event) => {
        console.log(`Connection checked out: ${event.connectionId}`);
      });
      
      this.client.on('connectionCheckedIn', (event) => {
        console.log(`Connection checked in: ${event.connectionId}`);
      });
      
      this.client.on('connectionPoolCleared', (event) => {
        console.log('Connection pool cleared:', event);
      });
      
      this.isConnected = true;
      console.log('MongoDB connected successfully');
      return this.client;
      
    } catch (error) {
      console.error('MongoDB connection error:', error);
      throw error;
    }
  }
  
  async getDatabase(dbName) {
    if (!this.isConnected) {
      await this.connect();
    }
    return this.client.db(dbName);
  }
  
  async healthCheck() {
    try {
      const admin = this.client.db('admin');
      await admin.command({ ping: 1 });
      
      // Get connection pool stats
      const stats = await admin.command({ serverStatus: 1 });
      
      return {
        status: 'healthy',
        connections: stats.connections,
        uptime: stats.uptime,
        version: stats.version
      };
      
    } catch (error) {
      return {
        status: 'unhealthy',
        error: error.message
      };
    }
  }
  
  async gracefulShutdown() {
    if (this.client) {
      console.log('Closing MongoDB connections...');
      await this.client.close();
      this.isConnected = false;
      console.log('MongoDB connections closed');
    }
  }
}

// Usage with connection management
const connectionManager = new MongoConnectionManager(
  'mongodb://localhost:27017/myapp',
  {
    maxPoolSize: 50,
    retryWrites: true,
    writeConcern: { w: 'majority', j: true }
  }
);

// Graceful shutdown handling
process.on('SIGINT', async () => {
  await connectionManager.gracefulShutdown();
  process.exit(0);
});

process.on('SIGTERM', async () => {
  await connectionManager.gracefulShutdown();
  process.exit(0);
});
```

## Operational Considerations

### Monitoring and Alerting

```javascript
const { MongoClient } = require('mongodb');

class MongoMonitor {
  constructor(connectionString) {
    this.client = new MongoClient(connectionString);
    this.metrics = new Map();
  }
  
  async connect() {
    await this.client.connect();
  }
  
  async collectServerStats() {
    const admin = this.client.db('admin');
    
    const serverStatus = await admin.command({ serverStatus: 1 });
    const dbStats = await this.client.db().stats();
    
    const metrics = {
      timestamp: new Date(),
      server: {
        uptime: serverStatus.uptime,
        version: serverStatus.version,
        connections: {
          current: serverStatus.connections.current,
          available: serverStatus.connections.available,
          totalCreated: serverStatus.connections.totalCreated
        },
        memory: {
          resident: serverStatus.mem.resident,
          virtual: serverStatus.mem.virtual,
          mapped: serverStatus.mem.mapped || 0
        },
        operations: {
          insert: serverStatus.opcounters.insert,
          query: serverStatus.opcounters.query,
          update: serverStatus.opcounters.update,
          delete: serverStatus.opcounters.delete
        },
        locks: serverStatus.globalLock.currentQueue
      },
      database: {
        collections: dbStats.collections,
        dataSize: dbStats.dataSize,
        storageSize: dbStats.storageSize,
        indexSize: dbStats.indexSize,
        objects: dbStats.objects
      }
    };
    
    return metrics;
  }
  
  async checkReplicaSetHealth() {
    const admin = this.client.db('admin');
    const status = await admin.command({ replSetGetStatus: 1 });
    
    return {
      set: status.set,
      members: status.members.map(member => ({
        name: member.name,
        state: member.stateStr,
        health: member.health,
        uptime: member.uptime
      }))
    };
  }
  
  async getSlowQueries(threshold = 100) {
    const admin = this.client.db('admin');
    const profile = await admin.collection('system.profile')
      .find({ millis: { $gt: threshold } })
      .sort({ ts: -1 })
      .limit(10)
      .toArray();
    
    return profile;
  }
}
```

### Key Metrics to Monitor

**Performance Metrics**:
- Query execution time (>100ms alerts)
- Connection pool utilization (>80% alerts) 
- Memory usage (>85% alerts)
- Disk I/O and storage growth
- Index hit ratio

**Availability Metrics**:
- Replica set member health
- Primary election frequency
- Network connectivity between nodes
- Backup completion status

**Capacity Metrics**:
- Database size growth trends
- Index size vs data size ratio
- Shard balancing status
- Connection count trends

### Backup and Recovery

```javascript
// Automated backup script
const backupConfig = {
  schedule: '0 2 * * *', // Daily at 2 AM
  retention: 30, // Keep 30 days
  compression: true,
  destinations: ['s3://backups/mongodb/', 'local:/backup/']
};

async function performBackup() {
  const timestamp = new Date().toISOString().split('T')[0];
  
  // For replica sets - backup from secondary
  const backupCommand = {
    mongodump: {
      host: 'mongodb-secondary:27017',
      db: 'production',
      archive: `backup-${timestamp}.gz`,
      gzip: true
    }
  };
  
  // Point-in-time recovery setup
  const oplogBackup = {
    command: 'mongodump',
    options: ['--db', 'local', '--collection', 'oplog.rs']
  };
  
  return { backupCommand, oplogBackup };
}
```

### Security Best Practices

**Authentication & Authorization**:
```javascript
// Enable authentication
const authConfig = {
  security: {
    authorization: 'enabled',
    keyFile: '/etc/mongodb/keyfile'
  },
  users: [
    {
      user: 'appUser',
      pwd: 'securePassword',
      roles: [
        { role: 'readWrite', db: 'production' },
        { role: 'read', db: 'analytics' }
      ]
    }
  ]
};
```

**Network Security**:
- Use SSL/TLS for all connections
- Implement IP whitelisting
- Enable authentication logs
- Regular security audits

### Performance Tuning Checklist

**Index Optimization**:
- Monitor unused indexes monthly
- Create compound indexes for common query patterns
- Use partial indexes for sparse data
- Implement TTL indexes for temporary data

**Query Optimization**:
- Enable profiling for slow queries (>100ms)
- Use aggregation pipeline optimization
- Implement proper pagination with cursor-based methods
- Avoid full collection scans

**Storage Optimization**:
- Configure WiredTiger cache size (50-60% of RAM)
- Use compression (snappy for balance, zstd for storage)
- Regular compact operations for fragmentation
- Monitor storage growth patterns

### Disaster Recovery Plan

**Recovery Time Objectives (RTO)**:
- Primary failure: <30 seconds (automatic failover)
- Complete cluster failure: <4 hours
- Data center failure: <8 hours

**Recovery Point Objectives (RPO)**:
- Local failures: 0 (replica sets)
- Disaster scenarios: <1 hour (backup frequency)

**Failover Procedures**:
1. Automatic failover within replica sets
2. Manual intervention for sharded cluster issues
3. Restore from backups for complete data loss
4. Point-in-time recovery using oplog replay

### Troubleshooting Common Issues

**High Memory Usage**:
- Check WiredTiger cache configuration
- Monitor index usage and remove unused indexes
- Implement data archiving strategies

**Slow Queries**:
- Enable profiling and analyze execution plans
- Check index usage with `explain()`
- Optimize aggregation pipelines

**Connection Issues**:
- Monitor connection pool metrics
- Check network connectivity between nodes
- Verify authentication and authorization

**Replication Lag**:
- Monitor secondary lag metrics
- Check network bandwidth between nodes
- Optimize write concern settings

### Maintenance Windows

**Regular Maintenance Tasks**:
- Monthly index analysis and cleanup
- Quarterly storage compaction
- Semi-annual version updates
- Annual disaster recovery testing

**Planned Maintenance Checklist**:
1. Schedule during low-traffic periods
2. Notify stakeholders in advance
3. Perform rolling updates on replica sets
4. Monitor system health post-maintenance
5. Document any issues or improvements