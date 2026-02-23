# Analytics & Telemetry Service - Technical Specification

> **Service**: Analytics & Telemetry Service  
> **Version**: 1.0  
> **Owner**: Platform Team  
> **Last Updated**: November 2025

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Analytics Service                                    │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────────────────┐  │
│  │ Event Ingester │  │  Query Engine  │  │   Dashboard Builder      │  │
│  └────────────────┘  └────────────────┘  └──────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
       ┌──────────┐     ┌──────────┐     ┌──────────┐
       │  Kafka   │     │ClickHouse│     │  Redis   │
       └──────────┘     └──────────┘     └──────────┘
```

---

## 2. Data Model

```sql
-- Events table (ClickHouse)
CREATE TABLE analytics_events (
    event_id UUID,
    tenant_id UUID,
    user_id Nullable(UUID),
    anonymous_id Nullable(String),
    session_id String,
    
    event_type String,
    event_name String,
    
    properties String,  -- JSON
    
    -- Context
    page_url String,
    page_title String,
    referrer String,
    
    -- Device
    device_type String,
    browser String,
    os String,
    
    -- Geo
    country String,
    city String,
    
    timestamp DateTime64(3),
    received_at DateTime64(3)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (tenant_id, timestamp, event_id);
```

---

## 3. API Contracts

### 3.1 Track Event

```yaml
POST /v1/analytics/track

Request:
  {
    "event": "button_clicked",
    "properties": {
      "buttonId": "signup_cta",
      "page": "homepage"
    },
    "context": {
      "page": {
        "url": "https://flowmind.io",
        "title": "FlowMind - Home"
      }
    }
  }

Response: 202 Accepted
```

### 3.2 Identify User

```yaml
POST /v1/analytics/identify

Request:
  {
    "userId": "uuid",
    "traits": {
      "email": "user@example.com",
      "plan": "pro",
      "company": "Acme"
    }
  }

Response: 202 Accepted
```

### 3.3 Query Events

```yaml
POST /v1/analytics/query

Request:
  {
    "metrics": ["count", "unique_users"],
    "dimensions": ["event_name", "day"],
    "filters": [
      { "field": "event_type", "op": "eq", "value": "page_view" }
    ],
    "dateRange": {
      "from": "2025-11-01",
      "to": "2025-11-30"
    }
  }

Response: 200 OK
  {
    "success": true,
    "data": {
      "rows": [
        { "event_name": "page_view", "day": "2025-11-01", "count": 1234, "unique_users": 567 }
      ]
    }
  }
```

---

## 4. JavaScript SDK

```typescript
// Client-side SDK
flowmind.track('button_clicked', {
  buttonId: 'signup_cta',
  page: 'homepage'
});

flowmind.identify('user-123', {
  email: 'user@example.com',
  plan: 'pro'
});

flowmind.page('Homepage', {
  category: 'marketing'
});
```

---

*FlowMind Technologies | Analytics & Telemetry Service Technical Specification v1.0*

