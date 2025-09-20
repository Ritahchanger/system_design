# Caching System API Documentation

## Table of Contents
1. [API Overview](#api-overview)
2. [Authentication](#authentication)
3. [Cache Operations](#cache-operations)
4. [Configuration Endpoints](#configuration-endpoints)
5. [Monitoring Endpoints](#monitoring-endpoints)
6. [Error Handling](#error-handling)
7. [Rate Limiting](#rate-limiting)
8. [Integration Examples](#integration-examples)

---

## API Overview

The Caching System API provides a comprehensive RESTful interface for cache operations, configuration management, and system monitoring. The API is designed following OpenAPI 3.0 specifications and supports multiple serialization formats.

### Base URL Structure

```
https://api.cache.example.com/v1
```

### API Architecture Flow

```mermaid
graph TB
    subgraph "Client Layer"
        WebClient[Web Client]
        MobileApp[Mobile App]
        ServiceClient[Service Client]
    end
    
    subgraph "API Gateway Layer"
        AuthHandler[Authentication Handler]
        RateLimiter[Rate Limiter]  
        RequestRouter[Request Router]
        ResponseFormatter[Response Formatter]
    end
    
    subgraph "API Endpoints"
        CacheOps[Cache Operations<br/>/cache/*]
        ConfigOps[Configuration<br/>/config/*]
        MonitorOps[Monitoring<br/>/monitor/*]
        AdminOps[Administration<br/>/admin/*]
    end
    
    subgraph "Backend Services"
        CacheLayer[Cache Layer<br/>Multi-tier cache]
        ConfigService[Configuration Service]
        MetricsService[Metrics Service]
        AdminService[Admin Service]
    end
    
    WebClient --> AuthHandler
    MobileApp --> AuthHandler
    ServiceClient --> AuthHandler
    
    AuthHandler --> RateLimiter
    RateLimiter --> RequestRouter
    RequestRouter --> ResponseFormatter
    
    ResponseFormatter --> CacheOps
    ResponseFormatter --> ConfigOps
    ResponseFormatter --> MonitorOps
    ResponseFormatter --> AdminOps
    
    CacheOps --> CacheLayer
    ConfigOps --> ConfigService
    MonitorOps --> MetricsService
    AdminOps --> AdminService
    
    classDef client fill:#e3f2fd
    classDef gateway fill:#f1f8e9
    classDef endpoint fill:#fff3e0
    classDef backend fill:#e8f5e8
    
    class WebClient,MobileApp,ServiceClient client
    class AuthHandler,RateLimiter,RequestRouter,ResponseFormatter gateway
    class CacheOps,ConfigOps,MonitorOps,AdminOps endpoint
    class CacheLayer,ConfigService,MetricsService,AdminService backend
```

*Architecture Reference*: [API Gateway Component](architecture.md#api-gateway)

### Supported Formats

| Format | Content-Type | Use Case | Performance |
|--------|-------------|----------|-------------|
| **JSON** | `application/json` | Human-readable, debugging | Standard |
| **MessagePack** | `application/msgpack` | Binary efficiency | 2x faster serialization |
| **Protocol Buffers** | `application/protobuf` | Strongly typed, compact | 3x smaller payload |
| **BSON** | `application/bson` | MongoDB compatibility | Good for complex data |

---

## Authentication

### Authentication Flow

```mermaid
sequenceDiagram
    participant Client
    participant API_Gateway
    participant Auth_Service
    participant Token_Store
    participant Cache_Service
    
    Note over Client,Cache_Service: API Key Authentication
    Client->>API_Gateway: Request with API Key
    API_Gateway->>Auth_Service: Validate API Key
    Auth_Service->>Token_Store: Check key validity
    Token_Store->>Auth_Service: Key details & permissions
    Auth_Service->>API_Gateway: Authorization result
    API_Gateway->>Cache_Service: Authorized request
    Cache_Service->>API_Gateway: Response
    API_Gateway->>Client: Response with rate limit headers
    
    Note over Client,Cache_Service: JWT Authentication  
    Client->>API_Gateway: Request with JWT Token
    API_Gateway->>Auth_Service: Validate JWT
    Auth_Service->>Auth_Service: Verify signature & expiry
    Auth_Service->>API_Gateway: Token claims & permissions
    API_Gateway->>Cache_Service: Request with user context
    Cache_Service->>API_Gateway: Response
    API_Gateway->>Client: Response
```

### API Key Authentication

```bash
# API Key in Header (Recommended)
curl -H "X-API-Key: your-api-key-here" \
     https://api.cache.example.com/v1/cache/mykey

# API Key in Query Parameter (Less secure)
curl https://api.cache.example.com/v1/cache/mykey?api_key=your-api-key-here
```

### JWT Token Authentication

```bash
# JWT Token in Authorization Header
curl -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
     https://api.cache.example.com/v1/cache/mykey
```

### Authentication Endpoints

#### Generate API Key

```http
POST /v1/auth/api-keys
Content-Type: application/json
Authorization: Bearer {jwt_token}

{
  "name": "Production Cache Access",
  "permissions": ["cache:read", "cache:write", "monitor:read"],
  "expires_in": 2592000
}
```

**Response:**
```json
{
  "api_key": "ck_live_abc123def456ghi789",
  "key_id": "key_12345",
  "name": "Production Cache Access", 
  "permissions": ["cache:read", "cache:write", "monitor:read"],
  "created_at": "2025-01-15T10:30:00Z",
  "expires_at": "2025-02-14T10:30:00Z",
  "rate_limit": {
    "requests_per_minute": 1000,
    "burst_size": 100
  }
}
```

#### Refresh JWT Token

```http
POST /v1/auth/refresh
Content-Type: application/json

{
  "refresh_token": "rt_abc123def456"
}
```

*Authentication Setup*: [Security Configuration](setup.md#authentication-setup)

---

## Cache Operations

### Cache Operation Flow

```mermaid
stateDiagram-v2
    [*] --> Validate_Request
    
    Validate_Request --> Authenticate: Valid format
    Validate_Request --> Return_400: Invalid format
    
    Authenticate --> Authorize: Valid credentials
    Authenticate --> Return_401: Invalid credentials
    
    Authorize --> Rate_Check: Sufficient permissions  
    Authorize --> Return_403: Insufficient permissions
    
    Rate_Check --> Process_Request: Within limits
    Rate_Check --> Return_429: Rate exceeded
    
    Process_Request --> Cache_Hit: Data found
    Process_Request --> Cache_Miss: Data not found
    
    Cache_Hit --> Return_200: Return cached data
    Cache_Miss --> Fetch_Source: Retrieve from source
    
    Fetch_Source --> Update_Cache: Data retrieved
    Fetch_Source --> Return_404: Source not found
    
    Update_Cache --> Return_200: Cache updated
    
    Return_200 --> [*]
    Return_400 --> [*]
    Return_401 --> [*]
    Return_403 --> [*]
    Return_404 --> [*]
    Return_429 --> [*]
```

### GET - Retrieve Cache Entry

```http
GET /v1/cache/{key}
```

**Parameters:**
- `key` (path, required): Cache key identifier
- `format` (query, optional): Response format (`json`, `msgpack`, `protobuf`)
- `include_metadata` (query, optional): Include cache metadata (default: `false`)

**Example Request:**
```bash
curl -H "X-API-Key: your-api-key" \
     "https://api.cache.example.com/v1/cache/user:12345?include_metadata=true"
```

**Response (200 OK):**
```json
{
  "key": "user:12345",
  "value": {
    "id": 12345,
    "name": "John Doe",
    "email": "john@example.com",
    "last_login": "2025-01-15T09:30:00Z"
  },
  "metadata": {
    "created_at": "2025-01-15T08:00:00Z",
    "updated_at": "2025-01-15T09:30:00Z",
    "expires_at": "2025-01-15T18:00:00Z",
    "hit_count": 47,
    "size_bytes": 156,
    "cache_layer": "L2"
  }
}
```

**Response Headers:**
```
X-Cache-Status: HIT
X-Cache-Layer: L2
X-Response-Time: 2ms
X-Rate-Limit-Remaining: 999
```

### SET - Store Cache Entry

```http
PUT /v1/cache/{key}
Content-Type: application/json
```

**Parameters:**
- `key` (path, required): Cache key identifier
- `ttl` (query, optional): Time-to-live in seconds
- `strategy` (query, optional): Caching strategy (`write-through`, `write-behind`, `write-around`)

**Request Body:**
```json
{
  "value": {
    "id": 12345,
    "name": "John Doe",
    "email": "john@example.com",
    "preferences": {
      "theme": "dark",
      "notifications": true
    }
  },
  "options": {
    "compress": true,
    "replicate": true,
    "tags": ["user", "session"]
  }
}
```

**Example Request:**
```bash
curl -X PUT \
     -H "X-API-Key: your-api-key" \
     -H "Content-Type: application/json" \
     -d '{"value":{"id":12345,"name":"John Doe"},"options":{"ttl":3600}}' \
     "https://api.cache.example.com/v1/cache/user:12345?strategy=write-through"
```

**Response (201 Created):**
```json
{
  "key": "user:12345",
  "status": "created",
  "metadata": {
    "created_at": "2025-01-15T10:15:00Z",
    "expires_at": "2025-01-15T11:15:00Z",
    "size_bytes": 234,
    "compression_ratio": 0.68,
    "replicated_to": ["node-2", "node-3"]
  },
  "cache_layers": ["L1", "L2", "L3"]
}
```

### DELETE - Remove Cache Entry

```http
DELETE /v1/cache/{key}
```

**Parameters:**
- `key` (path, required): Cache key identifier  
- `cascade` (query, optional): Delete from all cache layers (default: `true`)
- `pattern` (query, optional): Delete keys matching pattern (use with caution)

**Example Request:**
```bash
curl -X DELETE \
     -H "X-API-Key: your-api-key" \
     "https://api.cache.example.com/v1/cache/user:12345?cascade=true"
```

**Response (204 No Content):**
```
X-Deleted-From: L1,L2,L3
X-Deletion-Time: 5ms
```

### PATCH - Update Cache Entry

```http
PATCH /v1/cache/{key}
Content-Type: application/json
```

**Example Request (Partial Update):**
```bash
curl -X PATCH \
     -H "X-API-Key: your-api-key" \
     -H "Content-Type: application/json" \
     -d '{"value":{"last_login":"2025-01-15T10:45:00Z"},"merge_strategy":"deep"}' \
     "https://api.cache.example.com/v1/cache/user:12345"
```

### Bulk Operations

#### Bulk GET

```http
POST /v1/cache/bulk/get
Content-Type: application/json
```

**Request Body:**
```json
{
  "keys": ["user:12345", "user:67890", "session:abc123"],
  "options": {
    "include_metadata": false,
    "max_concurrent": 10
  }
}
```

**Response:**
```json
{
  "results": {
    "user:12345": {
      "status": "hit",
      "value": {"id": 12345, "name": "John Doe"},
      "cache_layer": "L1"
    },
    "user:67890": {
      "status": "hit", 
      "value": {"id": 67890, "name": "Jane Smith"},
      "cache_layer": "L2"
    },
    "session:abc123": {
      "status": "miss",
      "error": "Key not found"
    }
  },
  "summary": {
    "total_keys": 3,
    "hits": 2,
    "misses": 1,
    "hit_ratio": 0.67
  }
}
```

#### Bulk SET

```http
POST /v1/cache/bulk/set
Content-Type: application/json
```

**Request Body:**
```json
{
  "entries": [
    {
      "key": "user:12345",
      "value": {"id": 12345, "name": "John Doe"},
      "ttl": 3600
    },
    {
      "key": "user:67890", 
      "value": {"id": 67890, "name": "Jane Smith"},
      "ttl": 7200
    }
  ],
  "options": {
    "strategy": "write-behind",
    "max_concurrent": 5
  }
}
```

*Bulk Operations Setup*: [Batch Processing Configuration](setup.md#batch-processing)

---

## Configuration Endpoints

### Configuration Management Flow

```mermaid
graph TB
    subgraph "Configuration Sources"
        ConfigFile[Configuration Files]
        EnvVars[Environment Variables]
        ConfigAPI[Configuration API]
        DefaultConfig[Default Configuration]
    end
    
    subgraph "Configuration Management"
        ConfigValidator[Configuration Validator]
        ConfigMerger[Configuration Merger]
        ConfigDistributor[Configuration Distributor]
        ConfigVersioning[Configuration Versioning]
    end
    
    subgraph "Cache Components"
        CacheNodes[Cache Nodes]
        LoadBalancer[Load Balancer]
        Monitoring[Monitoring System]
        SecurityLayer[Security Layer]
    end
    
    ConfigFile --> ConfigValidator
    EnvVars --> ConfigValidator
    ConfigAPI --> ConfigValidator
    DefaultConfig --> ConfigValidator
    
    ConfigValidator --> ConfigMerger
    ConfigMerger --> ConfigVersioning
    ConfigVersioning --> ConfigDistributor
    
    ConfigDistributor --> CacheNodes
    ConfigDistributor --> LoadBalancer
    ConfigDistributor --> Monitoring
    ConfigDistributor --> SecurityLayer
    
    classDef source fill:#e3f2fd
    classDef management fill:#f1f8e9
    classDef component fill:#fff3e0
    
    class ConfigFile,EnvVars,ConfigAPI,DefaultConfig source
    class ConfigValidator,ConfigMerger,ConfigDistributor,ConfigVersioning management
    class CacheNodes,LoadBalancer,Monitoring,SecurityLayer component
```

### Get Current Configuration

```http
GET /v1/config
```

**Query Parameters:**
- `section` (optional): Specific configuration section
- `format` (optional): Response format (`json`, `yaml`, `toml`)
- `include_defaults` (optional): Include default values (default: `false`)

**Example Request:**
```bash
curl -H "X-API-Key: your-api-key" \
     "https://api.cache.example.com/v1/config?section=cache&format=yaml"
```

**Response (200 OK):**
```json
{
  "cache": {
    "layers": {
      "l1": {
        "type": "memory",
        "size_mb": 512,
        "eviction_policy": "lru",
        "ttl_default": 300
      },
      "l2": {
        "type": "redis", 
        "nodes": ["redis-1:6379", "redis-2:6379"],
        "max_connections": 100,
        "ttl_default": 3600
      },
      "l3": {
        "type": "distributed",
        "cluster_size": 3,
        "replication_factor": 2,
        "consistency_level": "eventual"
      }
    },
    "strategies": {
      "default_strategy": "write-through",
      "eviction_policies": ["lru", "lfu", "ttl"],
      "compression": {
        "enabled": true,
        "algorithm": "lz4",
        "min_size_bytes": 1024
      }
    }
  },
  "metadata": {
    "version": "1.2.3",
    "updated_at": "2025-01-15T08:00:00Z",
    "updated_by": "admin@example.com"
  }
}
```

### Update Configuration

```http
PUT /v1/config/{section}
Content-Type: application/json
```

**Parameters:**
- `section` (path, required): Configuration section to update
- `validate_only` (query, optional): Only validate without applying (default: `false`)
- `restart_services` (query, optional): Restart affected services (default: `true`)

**Example Request:**
```bash
curl -X PUT \
     -H "X-API-Key: your-admin-key" \
     -H "Content-Type: application/json" \
     -d '{
       "l1": {
         "size_mb": 1024,
         "eviction_policy": "lfu"
       }
     }' \
     "https://api.cache.example.com/v1/config/cache.layers?validate_only=false"
```

**Response (200 OK):**
```json
{
  "status": "updated",
  "changes": [
    {
      "path": "cache.layers.l1.size_mb",
      "old_value": 512,
      "new_value": 1024
    },
    {
      "path": "cache.layers.l1.eviction_policy", 
      "old_value": "lru",
      "new_value": "lfu"
    }
  ],
  "affected_services": ["cache-node-1", "cache-node-2"],
  "restart_required": true,
  "validation": {
    "status": "passed",
    "warnings": []
  }
}
```

### Configuration Schema

```http
GET /v1/config/schema
```

**Response:**
```json
{
  "schema_version": "1.0",
  "sections": {
    "cache": {
      "layers": {
        "l1": {
          "type": {"type": "string", "enum": ["memory", "redis"]},
          "size_mb": {"type": "integer", "minimum": 64, "maximum": 8192},
          "eviction_policy": {"type": "string", "enum": ["lru", "lfu", "ttl", "random"]}
        }
      }
    },
    "security": {
      "authentication": {
        "methods": {"type": "array", "items": {"enum": ["api_key", "jwt", "oauth2"]}},
        "session_timeout": {"type": "integer", "minimum": 300, "maximum": 86400}
      }
    }
  }
}
```

*Configuration Details*: [Configuration Management](setup.md#configuration-management)

---

## Monitoring Endpoints

### Monitoring Architecture

```mermaid
graph TB
    subgraph "Metrics Collection"
        SystemMetrics[System Metrics<br/>CPU, Memory, Disk]
        CacheMetrics[Cache Metrics<br/>Hit ratio, Latency]
        NetworkMetrics[Network Metrics<br/>Bandwidth, Connections]
        ApplicationMetrics[Application Metrics<br/>Requests, Errors]
    end
    
    subgraph "Metrics Processing"
        MetricsAggregator[Metrics Aggregator]
        TimeSeriesDB[Time Series Database<br/>Prometheus]
        AlertEngine[Alert Engine]
        Dashboard[Dashboard Engine<br/>Grafana]
    end
    
    subgraph "Monitoring API Endpoints"
        HealthEndpoint[/monitor/health]
        MetricsEndpoint[/monitor/metrics]
        AlertsEndpoint[/monitor/alerts]
        StatsEndpoint[/monitor/stats]
    end
    
    SystemMetrics --> MetricsAggregator
    CacheMetrics --> MetricsAggregator
    NetworkMetrics --> MetricsAggregator
    ApplicationMetrics --> MetricsAggregator
    
    MetricsAggregator --> TimeSeriesDB
    TimeSeriesDB --> AlertEngine
    TimeSeriesDB --> Dashboard
    
    TimeSeriesDB --> HealthEndpoint
    TimeSeriesDB --> MetricsEndpoint
    AlertEngine --> AlertsEndpoint
    MetricsAggregator --> StatsEndpoint
    
    classDef collection fill:#e8f5e8
    classDef processing fill:#fff3e0
    classDef endpoint fill:#e1f5fe
    
    class SystemMetrics,CacheMetrics,NetworkMetrics,ApplicationMetrics collection
    class MetricsAggregator,TimeSeriesDB,AlertEngine,Dashboard processing
    class HealthEndpoint,MetricsEndpoint,AlertsEndpoint,StatsEndpoint endpoint
```

### Health Check

```http
GET /v1/monitor/health
```

**Query Parameters:**
- `detailed` (optional): Include detailed component health (default: `false`)
- `timeout` (optional): Health check timeout in seconds (default: `5`)

**Example Request:**
```bash
curl -H "X-API-Key: your-api-key" \
     "https://api.cache.example.com/v1/monitor/health?detailed=true"
```

**Response (200 OK):**
```json
{
  "status": "healthy",
  "timestamp": "2025-01-15T10:30:00Z",
  "uptime": "15d 4h 32m",
  "version": "1.2.3",
  "components": {
    "cache_l1": {
      "status": "healthy",
      "response_time_ms": 1.2,
      "memory_usage_percent": 67
    },
    "cache_l2": {
      "status": "healthy", 
      "response_time_ms": 4.8,
      "connections": {
        "active": 45,
        "max": 100
      }
    },
    "cache_l3": {
      "status": "degraded",
      "response_time_ms": 45.2,
      "nodes": {
        "healthy": 2,
        "unhealthy": 1,
        "total": 3
      }
    },
    "database": {
      "status": "healthy",
      "response_time_ms": 12.5,
      "connection_pool": {
        "active": 8,
        "idle": 12,
        "max": 50
      }
    }
  },
  "overall_performance": {
    "requests_per_second": 1247,
    "average_response_time_ms": 8.3,
    "cache_hit_ratio": 0.89,
    "error_rate": 0.012
  }
}
```

### System Metrics

```http
GET /v1/monitor/metrics
```

**Query Parameters:**
- `metrics` (optional): Comma-separated list of specific metrics
- `timerange` (optional): Time range (`1h`, `24h`, `7d`, `30d`)
- `interval` (optional): Data point interval (`1m`, `5m`, `1h`)
- `format` (optional): Response format (`json`, `prometheus`)

**Example Request:**
```bash
curl -H "X-API-Key: your-api-key" \
     "https://api.cache.example.com/v1/monitor/metrics?metrics=cache_hits,response_time&timerange=1h&interval=5m"
```

**Response (200 OK):**
```json
{
  "timerange": {
    "start": "2025-01-15T09:30:00Z",
    "end": "2025-01-15T10:30:00Z",
    "interval": "5m"
  },
  "metrics": {
    "cache_hits": {
      "type": "counter",
      "unit": "requests",
      "data_points": [
        {"timestamp": "2025-01-15T09:30:00Z", "value": 1250},
        {"timestamp": "2025-01-15T09:35:00Z", "value": 1340},
        {"timestamp": "2025-01-15T09:40:00Z", "value": 1420}
      ]
    },
    "response_time": {
      "type": "histogram",
      "unit": "milliseconds",
      "data_points": [
        {
          "timestamp": "2025-01-15T09:30:00Z",
          "percentiles": {
            "p50": 5.2,
            "p95": 15.8,
            "p99": 45.3
          }
        }
      ]
    }
  }
}
```

### Real-time Statistics

```http
GET /v1/monitor/stats/realtime
```

**Example Response:**
```json
{
  "timestamp": "2025-01-15T10:30:15Z",
  "cache_performance": {
    "requests_per_second": 1247,
    "hit_ratio": 0.894,
    "average_response_time_ms": 8.3,
    "layer_distribution": {
      "l1_hits": 0.45,
      "l2_hits": 0.35,
      "l3_hits": 0.094,
      "misses": 0.106
    }
  },
  "system_resources": {
    "cpu_usage_percent": 34.2,
    "memory_usage_percent": 67.8,
    "disk_io_ops_per_second": 145,
    "network_throughput_mbps": 23.4
  },
  "active_connections": {
    "total": 245,
    "by_client_type": {
      "web_clients": 156,
      "api_clients": 67,
      "admin_clients": 22
    }
  }
}
```

### Performance Analytics

```http
GET /v1/monitor/analytics
```

**Query Parameters:**
- `period` (required): Analysis period (`hourly`, `daily`, `weekly`)
- `metrics` (optional): Specific metrics to analyze
- `compare_to` (optional): Comparison period

**Example Request:**
```bash
curl -H "X-API-Key: your-api-key" \
     "https://api.cache.example.com/v1/monitor/analytics?period=daily&compare_to=previous_week"
```

**Response:**
```json
{
  "analysis_period": "daily",
  "comparison_period": "previous_week",
  "summary": {
    "total_requests": 2847562,
    "average_hit_ratio": 0.887,
    "peak_requests_per_second": 2340,
    "total_data_transferred_gb": 45.7
  },
  "trends": {
    "hit_ratio": {
      "trend": "improving",
      "change_percent": 2.3,
      "confidence": 0.94
    },
    "response_time": {
      "trend": "stable", 
      "change_percent": -0.8,
      "confidence": 0.87
    }
  },
  "recommendations": [
    {
      "type": "optimization",
      "priority": "high",
      "description": "Consider increasing L1 cache size - showing 95% utilization",
      "expected_improvement": "15% better hit ratio"
    }
  ]
}
```

*Monitoring Setup*: [Monitoring Configuration](setup.md#monitoring-setup)

---

## Error Handling

### Error Response Structure

```mermaid
graph TB
    subgraph "Error Classification"
        ClientErrors[4xx Client Errors<br/>Invalid requests]
        ServerErrors[5xx Server Errors<br/>Internal failures]
        CacheErrors[Cache-specific Errors<br/>Domain-specific issues]
    end
    
    subgraph "Error Response Format"
        ErrorCode[Error Code<br/>Machine-readable identifier]
        ErrorMessage[Error Message<br/>Human-readable description]
        ErrorDetails[Error Details<br/>Additional context]
        ErrorMetadata[Error Metadata<br/>Request context]
    end
    
    subgraph "Error Handling Strategy"
        Retry[Retry Logic<br/>Exponential backoff]
        Fallback[Fallback Mechanisms<br/>Alternative data sources]
        Circuit[Circuit Breaker<br/>Prevent cascading failures]
    end
    
    ClientErrors --> ErrorCode
    ServerErrors --> ErrorCode
    CacheErrors --> ErrorCode
    
    ErrorCode --> ErrorMessage
    ErrorMessage --> ErrorDetails
    ErrorDetails --> ErrorMetadata
    
    ErrorMetadata --> Retry
    ErrorMetadata --> Fallback
    ErrorMetadata --> Circuit
    
    classDef errorType fill:#ffcdd2
    classDef response fill:#fff3e0
    classDef handling fill:#e8f5e8
    
    class ClientErrors,ServerErrors,CacheErrors errorType
    class ErrorCode,ErrorMessage,ErrorDetails,ErrorMetadata response
    class Retry,Fallback,Circuit handling
```

### Standard Error Response Format

```json
{
  "error": {
    "code": "CACHE_KEY_NOT_FOUND",
    "message": "The requested cache key does not exist",
    "details": {
      "key": "user:99999",
      "searched_layers": ["L1", "L2", "L3"],
      "suggestions": [
        "Verify the key format",
        "Check if the data has expired"
      ]
    },
    "metadata": {
      "request_id": "req_abc123def456",
      "timestamp": "2025-01-15T10:30:00Z",
      "api_version": "v1",
      "response_time_ms": 12
    }
  }
}
```

### Error Codes Reference

| HTTP Status | Error Code | Description | Retry Strategy |
|-------------|------------|-------------|----------------|
| **400** | `INVALID_REQUEST` | Malformed request syntax | No retry - fix request |
| **400** | `INVALID_KEY_FORMAT` | Cache key format violation | No retry - fix key format |
| **401** | `AUTHENTICATION_FAILED` | Invalid credentials | No retry - check credentials |
| **403** | `INSUFFICIENT_PERMISSIONS` | Operation not allowed | No retry - check permissions |
| **404** | `CACHE_KEY_NOT_FOUND` | Key does not exist | No retry - key doesn't exist |
| **409** | `CACHE_KEY_CONFLICT` | Concurrent modification | Retry with exponential backoff |
| **413** | `PAYLOAD_TOO_LARGE` | Request exceeds size limit | No retry - reduce payload |
| **429** | `RATE_LIMIT_EXCEEDED` | Too many requests | Retry after delay |
| **500** | `INTERNAL_SERVER_ERROR` | Unexpected server error | Retry with exponential backoff |
| **502** | `CACHE_UNAVAILABLE` | Cache backend unreachable | Retry with circuit breaker |
| **503** | `SERVICE_OVERLOADED` | Service temporarily overloaded | Retry with jittered delay |
| **504** | `CACHE_TIMEOUT` | Operation timed out | Retry with longer timeout |

### Client-Side Error Handling Example

```javascript
class CacheClient {
  async get(key, options = {}) {
    const maxRetries = options.maxRetries || 3;
    const baseDelay = options.baseDelay || 1000;
    
    for (let attempt = 0; attempt <= maxRetries; attempt++) {
      try {
        const response = await this.makeRequest('GET', `/cache/${key}`);
        return response;
      } catch (error) {
        if (!this.shouldRetry(error, attempt, maxRetries)) {
          throw error;
        }
        
        const delay = this.calculateDelay(baseDelay, attempt);
        await this.sleep(delay);
      }
    }
  }
  
  shouldRetry(error, attempt, maxRetries) {
    if (attempt >= maxRetries) return false;
    
    const retryableCodes = [
      'CACHE_TIMEOUT',
      'SERVICE_OVERLOADED', 
      'CACHE_UNAVAILABLE',
      'INTERNAL_SERVER_ERROR'
    ];
    
    return retryableCodes.includes(error.code);
  }
  
  calculateDelay(baseDelay, attempt) {
    // Exponential backoff with jitter
    const exponentialDelay = baseDelay * Math.pow(2, attempt);
    const jitter = Math.random() * 0.1 * exponentialDelay;
    return exponentialDelay + jitter;
  }
}
```

---

## Rate Limiting

### Rate Limiting Architecture

```mermaid
graph TB
    subgraph "Rate Limiting Strategies"
        TokenBucket[Token Bucket<br/>Burst handling]
        SlidingWindow[Sliding Window<br/>Precise counting]
        FixedWindow[Fixed Window<br/>Simple implementation]
        LeakyBucket[Leaky Bucket<br/>Smooth rate limiting]
    end
    
    subgraph "Rate Limit Dimensions"
        PerUser[Per User/API Key<br/>Individual quotas]
        PerEndpoint[Per Endpoint<br/>Resource protection]
        PerIP[Per IP Address<br/>Abuse prevention]
        Global[Global Limits<br/>System protection]
    end
    
    subgraph "Rate Limit Storage"
        Memory[In-Memory<br/>Fast, not persistent]
        Redis[Redis<br/>Distributed, persistent]
        Database[Database<br/>Audit trail]
    end
    
    Client[Client Request] --> TokenBucket
    TokenBucket --> PerUser
    PerUser --> Redis
    
    SlidingWindow --> PerEndpoint
    PerEndpoint --> Memory
    
    FixedWindow --> PerIP
    PerIP --> Redis
    
    LeakyBucket --> Global
    Global --> Database
    
    classDef strategy fill:#e1f5fe
    classDef dimension fill:#f1f8e9
    classDef storage fill:#fff3e0
    
    class TokenBucket,SlidingWindow,FixedWindow,LeakyBucket strategy
    class PerUser,PerEndpoint,PerIP,Global dimension
    class Memory,Redis,Database storage
```

### Rate Limit Headers

All API responses include rate limiting information in headers:

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1642248000
X-RateLimit-Window: 3600
X-RateLimit-Policy: sliding-window
```

### Rate Limit Tiers

| Tier | Requests/Hour | Burst Size | Cache Size Limit | Features |
|------|---------------|------------|------------------|----------|
| **Free** | 1,000 | 50 | 100MB | Basic operations |
| **Basic** | 10,000 | 200 | 1GB | Bulk operations, monitoring |
| **Pro** | 100,000 | 1,000 | 10GB | Analytics, custom configs |
| **Enterprise** | 1,000,000+ | 5,000+ | Unlimited | Priority support, SLA |

### Rate Limit Exceeded Response

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Request rate limit exceeded",
    "details": {
      "limit": 1000,
      "window": 3600,
      "reset_at": "2025-01-15T11:00:00Z",
      "retry_after": 1800
    }
  }
}
```

---

## Integration Examples

### SDK Examples

#### Python SDK

```python
from cache_client import CacheClient
import asyncio

# Initialize client
client = CacheClient(
    base_url="https://api.cache.example.com/v1",
    api_key="your-api-key",
    timeout=30,
    retry_config={
        "max_retries": 3,
        "backoff_factor": 2
    }
)

# Basic operations
async def cache_operations():
    # Set a value
    await client.set(
        key="user:12345",
        value={"name": "John Doe", "email": "john@example.com"},
        ttl=3600
    )
    
    # Get a value
    user = await client.get("user:12345")
    print(f"User: {user['name']}")
    
    # Bulk operations
    users = await client.bulk_get([
        "user:12345", 
        "user:67890", 
        "user:11111"
    ])
    
    # Delete a value
    await client.delete("user:12345")

# Run async operations
asyncio.run(cache_operations())
```

#### Node.js SDK

```javascript
const { CacheClient } = require('@cache-system/node-sdk');

const client = new CacheClient({
  baseURL: 'https://api.cache.example.com/v1',
  apiKey: 'your-api-key',
  timeout: 30000,
  retryConfig: {
    retries: 3,
    retryDelay: (retryCount) => Math.pow(2, retryCount) * 1000
  }
});

// Cache operations with error handling
async function cacheOperations() {
  try {
    // Set with options
    await client.set('session:abc123', {
      userId: 12345,
      loginTime: new Date().toISOString()
    }, {
      ttl: 1800,
      tags: ['session', 'user:12345']
    });
    
    // Get with metadata
    const session = await client.get('session:abc123', {
      includeMetadata: true
    });
    
    console.log(`Session created at: ${session.metadata.created_at}`);
    
    // Pattern-based operations
    await client.deletePattern('session:*');
    
  } catch (error) {
    if (error.code === 'CACHE_KEY_NOT_FOUND') {
      console.log('Key not found, creating new entry');
    } else {
      console.error('Cache error:', error.message);
    }
  }
}
```

### cURL Examples

#### Advanced Cache Operations

```bash
#!/bin/bash

API_KEY="your-api-key"
BASE_URL="https://api.cache.example.com/v1"

# Function to make API calls with error handling
cache_request() {
    local method=$1
    local endpoint=$2
    local data=$3
    
    response=$(curl -s -w "\n%{http_code}" \
        -X "$method" \
        -H "X-API-Key: $API_KEY" \
        -H "Content-Type: application/json" \
        ${data:+-d "$data"} \
        "$BASE_URL$endpoint")
    
    http_code=$(tail -n1 <<< "$response")
    response_body=$(sed '$ d' <<< "$response")
    
    if [[ $http_code -ge 200 && $http_code -lt 300 ]]; then
        echo "$response_body" | jq .
    else
        echo "Error $http_code: $response_body" | jq .
        return 1
    fi
}

# Set cache entry with compression
cache_request PUT "/cache/product:12345" '{
  "value": {
    "id": 12345,
    "name": "Laptop Computer",
    "price": 999.99,
    "inventory": 50,
    "specifications": {
      "cpu": "Intel Core i7",
      "ram": "16GB DDR4",
      "storage": "512GB NVMe SSD"
    }
  },
  "options": {
    "ttl": 7200,
    "compress": true,
    "tags": ["product", "electronics", "laptop"]
  }
}'

# Get cache entry with metadata
cache_request GET "/cache/product:12345?include_metadata=true"

# Bulk operations
cache_request POST "/cache/bulk/get" '{
  "keys": ["product:12345", "product:67890", "product:11111"],
  "options": {
    "include_metadata": false,
    "max_concurrent": 5
  }
}'
```

### Integration Patterns

#### Circuit Breaker Pattern

```python
import asyncio
from enum import Enum
from datetime import datetime, timedelta

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open" 
    HALF_OPEN = "half_open"

class CacheCircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60, success_threshold=2):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.success_threshold = success_threshold
        
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED
    
    async def call(self, cache_operation, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if self._should_attempt_reset():
                self.state = CircuitState.HALF_OPEN
                self.success_count = 0
            else:
                raise Exception("Circuit breaker is OPEN")
        
        try:
            result = await cache_operation(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise e
    
    def _on_success(self):
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.success_threshold:
                self.state = CircuitState.CLOSED
                self.failure_count = 0
        else:
            self.failure_count = 0
    
    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = datetime.now()
        
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
    
    def _should_attempt_reset(self):
        return (datetime.now() - self.last_failure_time) > timedelta(seconds=self.timeout)
```

*Integration Guides*: [Client Integration Patterns](setup.md#client-integration)

---

## Related Documentation

This API documentation connects to:

- **[System Architecture](architecture.md)**: Understanding the backend architecture that powers these APIs
- **[Setup & Configuration](setup.md)**: API server deployment and configuration  
- **[Caching Strategies](caching-strategies.md)**: Implementation patterns used by API operations
- **[Main Documentation](../README.md)**: Complete system overview and navigation

## Next Steps

1. **For Development**: Review [Integration Examples](#integration-examples) and start with basic operations
2. **For Configuration**: Check [Configuration Endpoints](#configuration-endpoints) for system setup
3. **For Monitoring**: Implement [Monitoring Endpoints](#monitoring-endpoints) for observability
4. **For Production**: Study [Error Handling](#error-handling) and [Rate Limiting](#rate-limiting) patterns

---

*This API documentation provides comprehensive coverage of all caching system endpoints with practical examples, error handling strategies, and integration patterns. All examples are production-ready and include proper error handling.*