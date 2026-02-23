# Customer Support Service - Technical Specification

> **Service**: Customer Support Service  
> **Version**: 1.0  
> **Owner**: Platform Team  
> **Last Updated**: November 2025

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Support Service                                     │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────────────────┐  │
│  │ Ticket Manager │  │ Knowledge Base │  │   SLA Tracker            │  │
│  └────────────────┘  └────────────────┘  └──────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Data Model

```sql
CREATE TABLE tickets (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL,
    organization_id UUID NOT NULL,
    
    ticket_number VARCHAR(20) NOT NULL UNIQUE,
    subject VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    
    status VARCHAR(20) NOT NULL DEFAULT 'open',
    priority VARCHAR(20) NOT NULL DEFAULT 'medium',
    category VARCHAR(50),
    
    reporter_user_id UUID NOT NULL,
    assignee_user_id UUID,
    
    -- SLA
    first_response_at TIMESTAMP WITH TIME ZONE,
    first_response_sla TIMESTAMP WITH TIME ZONE,
    resolution_sla TIMESTAMP WITH TIME ZONE,
    resolved_at TIMESTAMP WITH TIME ZONE,
    
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TABLE ticket_comments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    ticket_id UUID NOT NULL REFERENCES tickets(id),
    
    author_id UUID NOT NULL,
    author_type VARCHAR(20) NOT NULL,  -- customer, agent, system
    
    content TEXT NOT NULL,
    is_internal BOOLEAN NOT NULL DEFAULT FALSE,
    
    attachments JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TABLE kb_articles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID,  -- NULL for global articles
    
    title VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    
    category_id UUID,
    tags VARCHAR(50)[],
    
    status VARCHAR(20) NOT NULL DEFAULT 'draft',
    
    view_count INTEGER NOT NULL DEFAULT 0,
    helpful_count INTEGER NOT NULL DEFAULT 0,
    not_helpful_count INTEGER NOT NULL DEFAULT 0,
    
    published_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

---

## 3. API Contracts

### 3.1 Create Ticket

```yaml
POST /v1/tickets

Request:
  {
    "subject": "Unable to access dashboard",
    "description": "Getting 500 error when loading...",
    "priority": "high",
    "category": "technical"
  }

Response: 201 Created
  {
    "success": true,
    "data": {
      "id": "uuid",
      "ticketNumber": "TKT-2025-0001",
      "subject": "Unable to access dashboard",
      "status": "open",
      "priority": "high",
      "sla": {
        "firstResponseDue": "2025-11-29T14:00:00Z",
        "resolutionDue": "2025-11-30T10:00:00Z"
      }
    }
  }
```

### 3.2 List Tickets

```yaml
GET /v1/tickets

Query Parameters:
  status: string
  priority: string
  assignee: UUID

Response: 200 OK
```

### 3.3 Add Comment

```yaml
POST /v1/tickets/{id}/comments

Request:
  {
    "content": "I've reproduced the issue...",
    "isInternal": false,
    "attachments": ["file-uuid"]
  }

Response: 201 Created
```

### 3.4 Search Knowledge Base

```yaml
GET /v1/kb/search

Query Parameters:
  q: string
  category: string

Response: 200 OK
  {
    "success": true,
    "data": {
      "articles": [
        {
          "id": "uuid",
          "title": "How to reset password",
          "slug": "how-to-reset-password",
          "excerpt": "..."
        }
      ]
    }
  }
```

---

## 4. SLA Configuration

```typescript
const SLA_MATRIX = {
  urgent: { firstResponse: 1, resolution: 4 },   // hours
  high: { firstResponse: 4, resolution: 24 },
  medium: { firstResponse: 8, resolution: 48 },
  low: { firstResponse: 24, resolution: 72 }
};
```

---

*FlowMind Technologies | Customer Support Service Technical Specification v1.0*

