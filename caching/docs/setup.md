# Caching System Setup & Configuration Guide

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Installation](#installation)
3. [Environment Configuration](#environment-configuration)
4. [Cache Type Configuration](#cache-type-configuration)
5. [Strategy Configuration](#strategy-configuration)
6. [Monitoring Setup](#monitoring-setup)
7. [Security Configuration](#security-configuration)
8. [Deployment Strategies](#deployment-strategies)

---

## Prerequisites

### System Requirements

```mermaid
graph TB
    subgraph "Development Environment"
        DevOS[Operating System<br/>Linux/macOS/Windows]
        DevNode[Node.js 18+<br/>JavaScript Runtime]
        DevDocker[Docker 20+<br/>Containerization]
    end
    
    subgraph "Production Environment"
        ProdOS[Linux Distribution<br/>Ubuntu 20.04+]
        ProdNode[Node.js 18 LTS<br/>Production Runtime]
        ProdMem[Memory<br/>16GB+ RAM]
        ProdStorage[Storage<br/>SSD 100GB+]
    end
    
    subgraph "Infrastructure"
        Redis[Redis 6.0+<br/>Primary cache store]
        PostgreSQL[PostgreSQL 13+<br/>Configuration storage]
        Monitoring[Prometheus + Grafana<br/>Monitoring stack]
    end
    
    DevOS --> ProdOS
    DevNode --> ProdNode
    DevDocker --> ProdMem
    
    ProdNode --> Redis
    ProdMem --> PostgreSQL
    ProdStorage --> Monitoring
    
    classDef dev fill:#e8f5e8
    classDef prod fill:#fff3e0
    classDef infra fill:#e1f5fe
    
    class DevOS,DevNode,DevDocker dev
    class ProdOS,ProdNode,ProdMem,ProdStorage prod
    class Redis,PostgreSQL,Monitoring infra
```

### Hardware Specifications

| Component | Development | Production | High-Load |
|-----------|-------------|------------|-----------|
| **CPU** | 4 cores | 8-16 cores | 32+ cores |
| **RAM** | 8GB | 32GB | 128GB+ |
| **Storage** | 50GB SSD | 500GB NVMe | 2TB+ NVMe |
| **Network** | 100 Mbps | 1 Gbps | 10 Gbps+ |

---

## Installation

### Quick Start Installation

```bash
# Clone and setup
git clone https://github.com/your-org/caching-system.git
cd caching-system
npm install

# Copy environment configuration
cp .env.example .env

# Start with Docker Compose
docker-compose up -d

# Initialize system
npm run setup:dev
npm run health-check
```

### Docker Compose Setup

```yaml
version: '3.8'

services:
  cache-service:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - REDIS_URL=redis://redis:6379
      - POSTGRES_URL=postgres://postgres:password@postgres:5432/cachedb
    depends_on:
      - redis
      - postgres
    volumes:
      - ./config:/app/config
      - ./logs:/app/logs

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
      - ./config/redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf

  postgres:
    image: postgres:15-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=cachedb
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    volumes:
      - postgres-data:/var/lib/postgresql/data

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  redis-data:
  postgres-data:
  grafana-data:

networks:
  default:
    name: cache-network
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cache-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: cache-service
  template:
    metadata:
      labels:
        app: cache-service
    spec:
      containers:
      - name: cache-service
        image: cache-service:latest
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: REDIS_URL
          value: "redis://redis-service:6379"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

---

## Environment Configuration

### Configuration File Structure

```javascript
// config/default.js
module.exports = {
  server: {
    port: process.env.PORT || 3000,
    host: process.env.HOST || '0.0.0.0'
  },
  
  cache: {
    layers: {
      l1: {
        type: 'memory',
        size_mb: parseInt(process.env.L1_CACHE_SIZE_MB) || 512,
        eviction_policy: process.env.L1_EVICTION_POLICY || 'lru',
        ttl_default: parseInt(process.env.L1_DEFAULT_TTL) || 300
      },
      l2: {
        type: 'redis',
        url: process.env.REDIS_URL || 'redis://localhost:6379',
        max_connections: parseInt(process.env.REDIS_MAX_CONNECTIONS) || 100,
        ttl_default: parseInt(process.env.L2_DEFAULT_TTL) || 3600
      }
    },
    
    strategies: {
      default_strategy: process.env.DEFAULT_CACHE_STRATEGY || 'cache-aside',
      write_behind: {
        batch_size: parseInt(process.env.WRITE_BEHIND_BATCH_SIZE) || 100,
        flush_interval: parseInt(process.env.WRITE_BEHIND_FLUSH_INTERVAL) || 5000
      }
    }
  },
  
  monitoring: {
    enabled: process.env.MONITORING_ENABLED !== 'false',
    metrics_port: parseInt(process.env.PROMETHEUS_PORT) || 9090,
    log_level: process.env.LOG_LEVEL || 'info'
  },
  
  security: {
    api_key_required: process.env.API_KEY_REQUIRED === 'true',
    rate_limit_rpm: parseInt(process.env.RATE_LIMIT_RPM) || 1000
  }
};
```

### Environment Variables

```bash
# .env.example
NODE_ENV=development
PORT=3000

# Cache Configuration
L1_CACHE_SIZE_MB=512
L1_EVICTION_POLICY=lru
L2_DEFAULT_TTL=3600
REDIS_URL=redis://localhost:6379

# Strategy Configuration
DEFAULT_CACHE_STRATEGY=cache-aside
WRITE_BEHIND_BATCH_SIZE=100

# Monitoring
MONITORING_ENABLED=true
LOG_LEVEL=info

# Security
API_KEY_REQUIRED=false
RATE_LIMIT_RPM=1000
```

---

## Cache Type Configuration

### Memory Cache Setup

```javascript
// config/cache-types/memory.js
const memoryConfig = {
  l1_memory: {
    type: 'memory',
    size_mb: 512,
    max_entries: 100000,
    eviction_policy: {
      algorithm: 'lru',
      check_interval: 60000,
      max_age_seconds: 300
    }
  }
};

module.exports = memoryConfig;
```

### Redis Configuration

```javascript
// config/cache-types/redis.js
const redisConfig = {
  single: {
    host: 'localhost',
    port: 6379,
    db: 0,
    keyPrefix: 'cache:',
    connectTimeout: 10000,
    maxRetriesPerRequest: 3
  },
  
  cluster: {
    nodes: [
      { host: 'redis-1', port: 6379 },
      { host: 'redis-2', port: 6379 },
      { host: 'redis-3', port: 6379 }
    ],
    options: {
      enableOfflineQueue: false,
      scaleReads: 'slave'
    }
  }
};

module.exports = redisConfig;
```

---

## Strategy Configuration

### Cache-Aside Setup

```javascript
// config/strategies/cache-aside.js
module.exports = {
  name: 'cache-aside',
  settings: {
    default_ttl: 3600,
    retry_attempts: 3,
    retry_delay: 1000,
    
    key_patterns: {
      user: 'user:{id}',
      session: 'session:{token}',
      product: 'product:{id}'
    },
    
    invalidation: {
      on_write: true,
      cascade_deletes: false,
      batch_invalidation: true
    }
  }
};
```

### Write-Behind Setup

```javascript
// config/strategies/write-behind.js
module.exports = {
  name: 'write-behind',
  settings: {
    batch_size: 100,
    flush_interval: 5000,
    max_queue_size: 10000,
    
    failure_handling: {
      retry_attempts: 3,
      retry_backoff: 'exponential',
      dead_letter_queue: true
    },
    
    consistency: {
      eventual_consistency_window: 30000,
      conflict_resolution: 'last_write_wins'
    }
  }
};
```

---

## Monitoring Setup

### Prometheus Configuration

```yaml
# config/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'cache-service'
    static_configs:
      - targets: ['cache-service:3000']
    metrics_path: /metrics
    scrape_interval: 15s

  - job_name: 'redis'
    static_configs:
      - targets: ['redis:6379']
```

### Grafana Dashboard Setup

```json
{
  "dashboard": {
    "title": "Cache Performance",
    "panels": [
      {
        "title": "Hit Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(cache_hits_total[5m]) / (rate(cache_hits_total[5m]) + rate(cache_misses_total[5m])) * 100"
          }
        ]
      },
      {
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(cache_operation_duration_seconds_bucket[5m]))"
          }
        ]
      }
    ]
  }
}
```

### Application Metrics

```javascript
// monitoring/metrics.js
const prometheus = require('prom-client');

const metrics = {
  cacheHits: new prometheus.Counter({
    name: 'cache_hits_total',
    help: 'Total number of cache hits',
    labelNames: ['layer', 'key_pattern']
  }),
  
  cacheMisses: new prometheus.Counter({
    name: 'cache_misses_total',
    help: 'Total number of cache misses',
    labelNames: ['layer', 'key_pattern']
  }),
  
  operationDuration: new prometheus.Histogram({
    name: 'cache_operation_duration_seconds',
    help: 'Duration of cache operations',
    labelNames: ['operation', 'layer'],
    buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1]
  }),
  
  memoryUsage: new prometheus.Gauge({
    name: 'cache_memory_usage_bytes',
    help: 'Memory usage by cache layer',
    labelNames: ['layer']
  })
};

module.exports = metrics;
```

---

## Security Configuration

### Authentication Setup

```javascript
// config/security/auth.js
const authConfig = {
  methods: ['api_key', 'jwt'],
  
  api_key: {
    header_name: 'X-API-Key',
    required: process.env.API_KEY_REQUIRED === 'true',
    keys: process.env.API_KEYS?.split(',') || []
  },
  
  jwt: {
    secret: process.env.JWT_SECRET || 'default-secret',
    algorithm: 'HS256',
    expiry: '1h',
    issuer: 'cache-system'
  },
  
  rate_limiting: {
    enabled: true,
    window_ms: 60000,
    max_requests: 1000,
    skip_successful: false
  }
};

module.exports = authConfig;
```

### TLS Configuration

```javascript
// config/security/tls.js
const tlsConfig = {
  enabled: process.env.TLS_ENABLED === 'true',
  
  server: {
    key: process.env.TLS_KEY_PATH || './certs/server.key',
    cert: process.env.TLS_CERT_PATH || './certs/server.crt',
    ca: process.env.TLS_CA_PATH,
    requestCert: false,
    rejectUnauthorized: true
  },
  
  client: {
    key: process.env.CLIENT_KEY_PATH,
    cert: process.env.CLIENT_CERT_PATH,
    ca: process.env.CLIENT_CA_PATH,
    checkServerIdentity: true
  },
  
  ciphers: [
    'ECDHE-RSA-AES128-GCM-SHA256',
    'ECDHE-RSA-AES256-GCM-SHA384',
    'ECDHE-RSA-AES128-SHA256',
    'ECDHE-RSA-AES256-SHA384'
  ].join(':'),
  
  secureProtocol: 'TLSv1_2_method'
};

module.exports = tlsConfig;
```

---

## Deployment Strategies

### Production Deployment

```mermaid
graph TB
    subgraph "Load Balancer"
        LB[NGINX Load Balancer<br/>SSL Termination]
    end
    
    subgraph "Application Tier"
        App1[Cache Service 1<br/>Node.js + PM2]
        App2[Cache Service 2<br/>Node.js + PM2]
        App3[Cache Service 3<br/>Node.js + PM2]
    end
    
    subgraph "Cache Tier"
        Redis1[Redis Primary<br/>Master Node]
        Redis2[Redis Replica<br/>Slave Node]
        Redis3[Redis Sentinel<br/>Failover Monitor]
    end
    
    subgraph "Storage Tier"
        DB1[PostgreSQL Primary<br/>Configuration DB]
        DB2[PostgreSQL Replica<br/>Read Replica]
    end
    
    subgraph "Monitoring"
        Prometheus[Prometheus<br/>Metrics Collection]
        Grafana[Grafana<br/>Dashboards]
        AlertManager[AlertManager<br/>Notifications]
    end
    
    LB --> App1
    LB --> App2
    LB --> App3
    
    App1 --> Redis1
    App2 --> Redis1
    App3 --> Redis1
    
    Redis1 --> Redis2
    Redis3 --> Redis1
    Redis3 --> Redis2
    
    App1 --> DB1
    App2 --> DB1
    App3 --> DB1
    DB1 --> DB2
    
    App1 --> Prometheus
    App2 --> Prometheus
    App3 --> Prometheus
    Redis1 --> Prometheus
    
    Prometheus --> Grafana
    Prometheus --> AlertManager
    
    classDef lb fill:#ff9999
    classDef app fill:#99ccff
    classDef cache fill:#99ff99
    classDef db fill:#ffcc99
    classDef monitor fill:#cc99ff
    
    class LB lb
    class App1,App2,App3 app
    class Redis1,Redis2,Redis3 cache
    class DB1,DB2 db
    class Prometheus,Grafana,AlertManager monitor
```

### Docker Production Configuration

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/etc/nginx/certs
    depends_on:
      - cache-service

  cache-service:
    build: .
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 2G
          cpus: '1'
        reservations:
          memory: 1G
          cpus: '0.5'
    environment:
      - NODE_ENV=production
      - REDIS_URL=redis://redis-primary:6379
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  redis-primary:
    image: redis:7-alpine
    command: redis-server --appendonly yes --replica-read-only no
    volumes:
      - redis-primary-data:/data

  redis-replica:
    image: redis:7-alpine
    command: redis-server --replicaof redis-primary 6379
    depends_on:
      - redis-primary

volumes:
  redis-primary-data:
```

### Health Check Implementation

```javascript
// health/health-check.js
class HealthChecker {
  constructor(cacheService, database) {
    this.cache = cacheService;
    this.database = database;
  }
  
  async checkHealth() {
    const checks = {
      timestamp: new Date().toISOString(),
      status: 'healthy',
      checks: {}
    };
    
    try {
      // Check cache connectivity
      checks.checks.cache = await this.checkCache();
      
      // Check database connectivity
      checks.checks.database = await this.checkDatabase();
      
      // Check memory usage
      checks.checks.memory = this.checkMemoryUsage();
      
      // Overall status
      const allHealthy = Object.values(checks.checks)
        .every(check => check.status === 'healthy');
      
      checks.status = allHealthy ? 'healthy' : 'degraded';
      
    } catch (error) {
      checks.status = 'unhealthy';
      checks.error = error.message;
    }
    
    return checks;
  }
  
  async checkCache() {
    const testKey = `health-check:${Date.now()}`;
    const testValue = 'ping';
    
    try {
      await this.cache.set(testKey, testValue, 10);
      const result = await this.cache.get(testKey);
      await this.cache.delete(testKey);
      
      return {
        status: result === testValue ? 'healthy' : 'unhealthy',
        response_time: Date.now()
      };
    } catch (error) {
      return {
        status: 'unhealthy',
        error: error.message
      };
    }
  }
  
  async checkDatabase() {
    try {
      const result = await this.database.query('SELECT 1');
      return {
        status: 'healthy',
        connection_count: result.connectionCount || 0
      };
    } catch (error) {
      return {
        status: 'unhealthy',
        error: error.message
      };
    }
  }
  
  checkMemoryUsage() {
    const memUsage = process.memoryUsage();
    const totalMem = require('os').totalmem();
    const freeMem = require('os').freemem();
    const usedPercent = ((totalMem - freeMem) / totalMem) * 100;
    
    return {
      status: usedPercent < 90 ? 'healthy' : 'warning',
      heap_used_mb: Math.round(memUsage.heapUsed / 1024 / 1024),
      heap_total_mb: Math.round(memUsage.heapTotal / 1024 / 1024),
      system_memory_usage_percent: Math.round(usedPercent)
    };
  }
}

module.exports = HealthChecker;
```

### Deployment Scripts

```bash
#!/bin/bash
# scripts/deploy.sh

deploy_production() {
    echo "Starting production deployment..."
    
    # Build application
    npm run build
    
    # Run tests
    npm test
    if [ $? -ne 0 ]; then
        echo "Tests failed, aborting deployment"
        exit 1
    fi
    
    # Build Docker image
    docker build -t cache-service:latest .
    
    # Deploy with zero downtime
    docker-compose -f docker-compose.prod.yml up -d --scale cache-service=0
    sleep 30
    docker-compose -f docker-compose.prod.yml up -d --scale cache-service=3
    
    # Wait for health check
    for i in {1..30}; do
        if curl -f http://localhost/health > /dev/null 2>&1; then
            echo "Deployment successful!"
            break
        fi
        sleep 10
    done
    
    # Cleanup old images
    docker image prune -f
}

rollback() {
    echo "Rolling back deployment..."
    docker-compose -f docker-compose.prod.yml down
    docker-compose -f docker-compose.prod.yml up -d
}

case "$1" in
    deploy)
        deploy_production
        ;;
    rollback)
        rollback
        ;;
    *)
        echo "Usage: $0 {deploy|rollback}"
        exit 1
        ;;
esac
```

---

## Configuration Validation

### Validation Script

```javascript
// scripts/validate-config.js
const config = require('../config');

function validateConfig() {
  const errors = [];
  
  // Required configurations
  if (!config.cache?.layers?.l2?.url) {
    errors.push('Redis URL is required');
  }
  
  if (!config.database?.url) {
    errors.push('Database URL is required');
  }
  
  // Validate cache sizes
  const l1Size = config.cache?.layers?.l1?.size_mb;
  if (l1Size && (l1Size < 64 || l1Size > 4096)) {
    errors.push('L1 cache size must be between 64MB and 4GB');
  }
  
  // Validate TTL values
  const defaultTTL = config.cache?.layers?.l1?.ttl_default;
  if (defaultTTL && (defaultTTL < 60 || defaultTTL > 86400)) {
    errors.push('Default TTL must be between 1 minute and 24 hours');
  }
  
  if (errors.length > 0) {
    console.error('Configuration validation failed:');
    errors.forEach(error => console.error(`- ${error}`));
    process.exit(1);
  }
  
  console.log('Configuration validation passed!');
}

if (require.main === module) {
  validateConfig();
}

module.exports = validateConfig;
```

---

## Quick Setup Commands

```bash
# Development setup
npm run setup:dev

# Production deployment
npm run deploy:prod

# Health check
npm run health

# Configuration validation
npm run validate:config

# Performance test
npm run test:performance

# Generate certificates
npm run generate:certs
```

---

This setup guide provides comprehensive configuration and deployment strategies for the caching system, covering development through production environments with proper monitoring, security, and health checking capabilities.