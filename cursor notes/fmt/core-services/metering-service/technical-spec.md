# Metering & Usage Service - Technical Specification

> **Service**: Metering & Usage Service  
> **Version**: 1.0  
> **Owner**: Platform Team  
> **Last Updated**: November 2025

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       Metering Service                                   │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────────────────┐  │
│  │ Event Ingester │  │  Aggregation   │  │   Quota Enforcer         │  │
│  │ (High Volume)  │  │   Engine       │  │   (Real-time)            │  │
│  └────────────────┘  └────────────────┘  └──────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
         ▼                     ▼                     ▼
   ┌──────────┐         ┌──────────┐         ┌──────────┐
   │  Kafka   │         │ TimescaleDB│        │  Redis   │
   │ (Events) │         │ (Storage)  │        │ (Counters)│
   └──────────┘         └──────────┘         └──────────┘
```

---

## 2. Data Model

### 2.1 Database Schema (TimescaleDB)

```sql
-- Usage events (hypertable for time-series)
CREATE TABLE usage_events (
    id UUID DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL,
    organization_id UUID NOT NULL,
    
    metric_name VARCHAR(100) NOT NULL,
    metric_type VARCHAR(20) NOT NULL,  -- counter, gauge, histogram
    value DECIMAL(20,6) NOT NULL,
    
    -- Dimensions
    product_id UUID,
    feature_name VARCHAR(100),
    resource_id VARCHAR(100),
    tags JSONB NOT NULL DEFAULT '{}',
    
    -- Deduplication
    idempotency_key VARCHAR(100),
    
    recorded_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    PRIMARY KEY (id, recorded_at)
);

SELECT create_hypertable('usage_events', 'recorded_at');

-- Usage aggregates (materialized)
CREATE TABLE usage_aggregates (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL,
    organization_id UUID NOT NULL,
    
    metric_name VARCHAR(100) NOT NULL,
    aggregation_period VARCHAR(20) NOT NULL,  -- hourly, daily, monthly
    period_start TIMESTAMP WITH TIME ZONE NOT NULL,
    period_end TIMESTAMP WITH TIME ZONE NOT NULL,
    
    sum_value DECIMAL(20,6) NOT NULL DEFAULT 0,
    count_value BIGINT NOT NULL DEFAULT 0,
    min_value DECIMAL(20,6),
    max_value DECIMAL(20,6),
    avg_value DECIMAL(20,6),
    
    tags JSONB NOT NULL DEFAULT '{}',
    
    CONSTRAINT usage_aggregates_unique UNIQUE (tenant_id, metric_name, aggregation_period, period_start, tags)
);

-- Current period counters (for real-time)
CREATE TABLE current_usage (
    tenant_id UUID NOT NULL,
    metric_name VARCHAR(100) NOT NULL,
    
    current_value DECIMAL(20,6) NOT NULL DEFAULT 0,
    period_start TIMESTAMP WITH TIME ZONE NOT NULL,
    period_end TIMESTAMP WITH TIME ZONE NOT NULL,
    
    last_updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    PRIMARY KEY (tenant_id, metric_name)
);
```

### 2.2 TypeScript Interfaces

```typescript
interface UsageEvent {
  tenantId: string;
  organizationId: string;
  metricName: string;
  metricType: 'counter' | 'gauge' | 'histogram';
  value: number;
  productId?: string;
  featureName?: string;
  resourceId?: string;
  tags?: Record<string, string>;
  idempotencyKey?: string;
  recordedAt: Date;
}

interface UsageAggregate {
  tenantId: string;
  metricName: string;
  aggregationPeriod: 'hourly' | 'daily' | 'monthly';
  periodStart: Date;
  periodEnd: Date;
  sum: number;
  count: number;
  min?: number;
  max?: number;
  avg?: number;
}

interface QuotaCheck {
  allowed: boolean;
  currentUsage: number;
  limit: number;
  remaining: number;
  percentUsed: number;
  resetAt?: Date;
}
```

---

## 3. API Contracts

### 3.1 Event Ingestion

#### Record Usage Event
```yaml
POST /v1/usage/events

Request:
  {
    "metricName": "api_calls",
    "value": 1,
    "tags": {
      "endpoint": "/api/v1/users",
      "method": "GET"
    },
    "idempotencyKey": "req-uuid-123"
  }

Response: 202 Accepted
```

#### Batch Record Events
```yaml
POST /v1/usage/events/batch

Request:
  {
    "events": [
      { "metricName": "api_calls", "value": 1 },
      { "metricName": "storage_bytes", "value": 1024 }
    ]
  }

Response: 202 Accepted
```

### 3.2 Usage Queries

#### Get Current Usage
```yaml
GET /v1/usage/current

Query Parameters:
  organizationId: UUID
  metrics: string[] (comma-separated)

Response: 200 OK
  {
    "success": true,
    "data": {
      "api_calls": {
        "current": 23456,
        "limit": 100000,
        "percentUsed": 23.46,
        "resetAt": "2025-12-01T00:00:00Z"
      },
      "storage_gb": {
        "current": 12.5,
        "limit": 50,
        "percentUsed": 25
      }
    }
  }
```

#### Get Usage History
```yaml
GET /v1/usage/history

Query Parameters:
  organizationId: UUID
  metric: string
  period: string (hourly, daily, monthly)
  from: datetime
  to: datetime

Response: 200 OK
  {
    "success": true,
    "data": {
      "metric": "api_calls",
      "period": "daily",
      "points": [
        { "timestamp": "2025-11-01", "value": 3456 },
        { "timestamp": "2025-11-02", "value": 4123 }
      ]
    }
  }
```

### 3.3 Quota Check (gRPC)

```protobuf
service MeteringService {
  rpc CheckQuota(CheckQuotaRequest) returns (CheckQuotaResponse);
  rpc IncrementUsage(IncrementUsageRequest) returns (IncrementUsageResponse);
}

message CheckQuotaRequest {
  string tenant_id = 1;
  string metric_name = 2;
  double requested_amount = 3;
}

message CheckQuotaResponse {
  bool allowed = 1;
  double current_usage = 2;
  double limit = 3;
  double remaining = 4;
}
```

---

## 4. Event Processing Pipeline

```typescript
// Kafka consumer for high-volume events
async function processUsageEvent(event: UsageEvent): Promise<void> {
  // 1. Deduplication check
  if (event.idempotencyKey) {
    const exists = await redis.exists(`idemp:${event.idempotencyKey}`);
    if (exists) return;
    await redis.setex(`idemp:${event.idempotencyKey}`, 86400, '1');
  }
  
  // 2. Increment real-time counter
  await redis.incrbyfloat(`usage:${event.tenantId}:${event.metricName}`, event.value);
  
  // 3. Store event for aggregation
  await db.usageEvents.insert(event);
  
  // 4. Check quota thresholds
  await checkAndAlertQuota(event.tenantId, event.metricName);
}
```

---

## 5. Caching Strategy

```
usage:{tenantId}:{metric}          - Real-time counter (Redis INCR)
usage:{tenantId}:current           - Current period summary (TTL: 1min)
idemp:{key}                        - Idempotency check (TTL: 24h)
```

---

*FlowMind Technologies | Metering & Usage Service Technical Specification v1.0*

