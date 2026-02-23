# Audit & Logging Service - Technical Specification

> **Service**: Audit & Logging Service  
> **Version**: 1.0  
> **Owner**: Platform Team  
> **Last Updated**: November 2025

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       Audit Service                                      │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────────────────┐  │
│  │ Event Ingester │  │  Query Engine  │  │   Retention Manager      │  │
│  │ (Kafka)        │  │  (Search)      │  │   (Lifecycle)            │  │
│  └────────────────┘  └────────────────┘  └──────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
       ┌──────────┐     ┌──────────┐     ┌──────────┐
       │  Kafka   │     │Elasticsearch│   │ S3/Blob  │
       │ (Ingest) │     │ (Search)   │   │ (Archive)│
       └──────────┘     └──────────┘     └──────────┘
```

---

## 2. Data Model

### 2.1 Audit Event Schema

```typescript
interface AuditEvent {
  id: string;
  timestamp: Date;
  
  // Context
  tenantId: string;
  organizationId: string;
  
  // Actor
  actor: {
    type: 'user' | 'service' | 'system' | 'api_key';
    id: string;
    email?: string;
    name?: string;
    ipAddress?: string;
    userAgent?: string;
  };
  
  // Action
  action: string;  // e.g., "user.created", "settings.updated"
  category: AuditCategory;
  
  // Target
  resource: {
    type: string;  // e.g., "user", "organization"
    id: string;
    name?: string;
  };
  
  // Details
  changes?: {
    field: string;
    oldValue: unknown;
    newValue: unknown;
  }[];
  
  metadata: Record<string, unknown>;
  
  // Request context
  requestId: string;
  sessionId?: string;
}

type AuditCategory = 
  | 'authentication'
  | 'authorization'
  | 'data_access'
  | 'data_modification'
  | 'admin_action'
  | 'security'
  | 'system';
```

### 2.2 Elasticsearch Mapping

```json
{
  "mappings": {
    "properties": {
      "timestamp": { "type": "date" },
      "tenantId": { "type": "keyword" },
      "organizationId": { "type": "keyword" },
      "actor": {
        "properties": {
          "type": { "type": "keyword" },
          "id": { "type": "keyword" },
          "email": { "type": "keyword" },
          "ipAddress": { "type": "ip" }
        }
      },
      "action": { "type": "keyword" },
      "category": { "type": "keyword" },
      "resource": {
        "properties": {
          "type": { "type": "keyword" },
          "id": { "type": "keyword" }
        }
      },
      "changes": { "type": "nested" },
      "metadata": { "type": "object", "enabled": false }
    }
  }
}
```

---

## 3. API Contracts

### 3.1 Log Audit Event (Internal)

```yaml
POST /v1/audit/events

Request:
  {
    "action": "user.invited",
    "category": "admin_action",
    "resource": {
      "type": "user",
      "id": "uuid",
      "name": "john@example.com"
    },
    "changes": [
      { "field": "status", "oldValue": null, "newValue": "pending" }
    ],
    "metadata": {
      "roleAssigned": "member"
    }
  }

Response: 202 Accepted
```

### 3.2 Search Audit Logs

```yaml
GET /v1/audit/events

Query Parameters:
  organizationId: UUID (required)
  category: string
  action: string
  actorId: string
  resourceType: string
  resourceId: string
  from: datetime
  to: datetime
  page: integer
  limit: integer

Response: 200 OK
  {
    "success": true,
    "data": {
      "events": [
        {
          "id": "uuid",
          "timestamp": "2025-11-29T10:00:00Z",
          "actor": {
            "type": "user",
            "email": "admin@example.com",
            "ipAddress": "192.168.1.1"
          },
          "action": "user.invited",
          "category": "admin_action",
          "resource": {
            "type": "user",
            "id": "uuid",
            "name": "john@example.com"
          }
        }
      ],
      "total": 156
    }
  }
```

### 3.3 Export Audit Logs

```yaml
POST /v1/audit/export

Request:
  {
    "organizationId": "uuid",
    "from": "2025-01-01",
    "to": "2025-11-30",
    "format": "csv",
    "filters": {
      "category": ["authentication", "security"]
    }
  }

Response: 202 Accepted
  {
    "success": true,
    "data": {
      "exportId": "uuid",
      "status": "processing",
      "downloadUrl": null
    }
  }
```

---

## 4. Event Collection

```typescript
// Middleware for automatic audit logging
async function auditMiddleware(req: Request, res: Response, next: Next) {
  const startTime = Date.now();
  
  res.on('finish', async () => {
    if (shouldAudit(req, res)) {
      await publishAuditEvent({
        action: getActionFromRoute(req),
        category: getCategoryFromRoute(req),
        actor: {
          type: req.user ? 'user' : 'api_key',
          id: req.user?.id || req.apiKey?.id,
          email: req.user?.email,
          ipAddress: req.ip,
          userAgent: req.headers['user-agent']
        },
        resource: extractResource(req, res),
        changes: extractChanges(req, res),
        requestId: req.id,
        metadata: {
          method: req.method,
          path: req.path,
          statusCode: res.statusCode,
          duration: Date.now() - startTime
        }
      });
    }
  });
  
  next();
}
```

---

## 5. Retention & Archival

```typescript
const RETENTION_POLICIES = {
  authentication: { hot: 90, warm: 365, archive: 2555 },  // 7 years
  security: { hot: 90, warm: 365, archive: 2555 },
  data_access: { hot: 30, warm: 365, archive: 2555 },
  admin_action: { hot: 90, warm: 365, archive: 1825 },  // 5 years
  data_modification: { hot: 30, warm: 90, archive: 365 }
};

// Lifecycle management
async function manageRetention(): Promise<void> {
  for (const [category, policy] of Object.entries(RETENTION_POLICIES)) {
    // Move to warm storage
    await moveToWarm(category, policy.hot);
    
    // Archive to S3
    await archive(category, policy.warm);
    
    // Delete expired
    await deleteExpired(category, policy.archive);
  }
}
```

---

*FlowMind Technologies | Audit & Logging Service Technical Specification v1.0*

