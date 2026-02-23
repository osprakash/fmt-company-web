# Account Service - Technical Specification

> **Service**: Account Service  
> **Version**: 1.0  
> **Owner**: Platform Team  
> **Last Updated**: November 2025

---

## 1. Architecture Overview

### 1.1 Service Context
```
┌─────────────────────────────────────────────────────────────┐
│                        API Gateway                          │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Account Service                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │   REST API  │  │   GraphQL   │  │   Event Publisher   │  │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘  │
│         │                │                     │             │
│         └────────────────┼─────────────────────┘             │
│                          │                                   │
│                          ▼                                   │
│              ┌───────────────────────┐                       │
│              │    Business Logic     │                       │
│              │  ┌─────────────────┐  │                       │
│              │  │ OrganizationSvc │  │                       │
│              │  │ DivisionSvc     │  │                       │
│              │  │ AccountSvc      │  │                       │
│              │  └─────────────────┘  │                       │
│              └───────────┬───────────┘                       │
│                          │                                   │
│              ┌───────────┴───────────┐                       │
│              ▼                       ▼                       │
│    ┌─────────────────┐    ┌─────────────────┐               │
│    │   PostgreSQL    │    │     Redis       │               │
│    │   (Primary)     │    │    (Cache)      │               │
│    └─────────────────┘    └─────────────────┘               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  Message Broker │
                    │ (RabbitMQ/Kafka)│
                    └─────────────────┘
```

### 1.2 Technology Stack
| Component | Technology | Rationale |
|-----------|------------|-----------|
| Runtime | Node.js 20 LTS | Team expertise, async performance |
| Framework | NestJS | Enterprise patterns, TypeScript |
| Database | PostgreSQL 15 | ACID, JSON support, mature |
| Cache | Redis 7 | Performance, pub/sub |
| ORM | Prisma | Type safety, migrations |
| API Docs | OpenAPI 3.0 | Industry standard |

---

## 2. Data Model

### 2.1 Entity Relationship Diagram
```
┌──────────────────────────────────────────────────────────────────┐
│                         ORGANIZATION                              │
├──────────────────────────────────────────────────────────────────┤
│ PK │ id: UUID                                                    │
│    │ tenant_id: UUID (for future multi-tenant scenarios)         │
│    │ name: VARCHAR(255)                                          │
│    │ slug: VARCHAR(100) UNIQUE                                   │
│    │ legal_name: VARCHAR(255)                                    │
│    │ status: ENUM                                                │
│    │ settings: JSONB                                             │
│    │ metadata: JSONB                                             │
│    │ created_at: TIMESTAMP                                       │
│    │ updated_at: TIMESTAMP                                       │
│    │ deleted_at: TIMESTAMP (soft delete)                         │
└──────────────────────────────────────────────────────────────────┘
                              │
                              │ 1:N
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                           DIVISION                                │
├──────────────────────────────────────────────────────────────────┤
│ PK │ id: UUID                                                    │
│ FK │ organization_id: UUID                                       │
│ FK │ parent_id: UUID (self-reference, nullable)                  │
│ FK │ account_id: UUID (nullable)                                 │
│    │ name: VARCHAR(255)                                          │
│    │ code: VARCHAR(50)                                           │
│    │ description: TEXT                                           │
│    │ cost_center: VARCHAR(50)                                    │
│    │ level: INTEGER (computed depth)                             │
│    │ path: LTREE (materialized path)                             │
│    │ metadata: JSONB                                             │
│    │ created_at: TIMESTAMP                                       │
│    │ updated_at: TIMESTAMP                                       │
│    │ deleted_at: TIMESTAMP                                       │
└──────────────────────────────────────────────────────────────────┘
                              │
                              │ N:1
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                           ACCOUNT                                 │
├──────────────────────────────────────────────────────────────────┤
│ PK │ id: UUID                                                    │
│ FK │ organization_id: UUID                                       │
│    │ name: VARCHAR(255)                                          │
│    │ type: ENUM (standard, enterprise, reseller)                 │
│    │ status: ENUM (active, past_due, suspended, closed)          │
│    │ billing_email: VARCHAR(255)                                 │
│    │ billing_address: JSONB                                      │
│    │ tax_id: VARCHAR(50) ENCRYPTED                               │
│    │ tax_exempt: BOOLEAN                                         │
│    │ currency: CHAR(3)                                           │
│    │ payment_terms: VARCHAR(20)                                  │
│    │ metadata: JSONB                                             │
│    │ created_at: TIMESTAMP                                       │
│    │ updated_at: TIMESTAMP                                       │
│    │ deleted_at: TIMESTAMP                                       │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 Database Schema (PostgreSQL)

```sql
-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "ltree";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Enums
CREATE TYPE organization_status AS ENUM (
    'trial', 'active', 'suspended', 'churned', 'deleted'
);

CREATE TYPE account_status AS ENUM (
    'active', 'past_due', 'suspended', 'closed'
);

CREATE TYPE account_type AS ENUM (
    'standard', 'enterprise', 'reseller'
);

-- Organizations Table
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    legal_name VARCHAR(255),
    logo_url VARCHAR(500),
    website VARCHAR(255),
    industry VARCHAR(100),
    company_size VARCHAR(50),
    tax_id VARCHAR(50),  -- Encrypted at application level
    primary_email VARCHAR(255) NOT NULL,
    status organization_status NOT NULL DEFAULT 'trial',
    settings JSONB NOT NULL DEFAULT '{}',
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT organizations_name_length CHECK (char_length(name) >= 2),
    CONSTRAINT organizations_slug_format CHECK (slug ~ '^[a-z0-9-]+$')
);

-- Indexes for Organizations
CREATE INDEX idx_organizations_slug ON organizations(slug) WHERE deleted_at IS NULL;
CREATE INDEX idx_organizations_status ON organizations(status) WHERE deleted_at IS NULL;
CREATE INDEX idx_organizations_tenant ON organizations(tenant_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_organizations_created ON organizations(created_at DESC);

-- Divisions Table
CREATE TABLE divisions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    parent_id UUID REFERENCES divisions(id),
    account_id UUID,  -- FK added after accounts table
    name VARCHAR(255) NOT NULL,
    code VARCHAR(50),
    description TEXT,
    cost_center VARCHAR(50),
    level INTEGER NOT NULL DEFAULT 0,
    path LTREE,
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT divisions_name_length CHECK (char_length(name) >= 1),
    CONSTRAINT divisions_level_max CHECK (level <= 10),
    CONSTRAINT divisions_no_self_parent CHECK (id != parent_id)
);

-- Indexes for Divisions
CREATE INDEX idx_divisions_organization ON divisions(organization_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_divisions_parent ON divisions(parent_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_divisions_path ON divisions USING GIST (path);
CREATE INDEX idx_divisions_account ON divisions(account_id) WHERE deleted_at IS NULL;

-- Accounts Table
CREATE TABLE accounts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    type account_type NOT NULL DEFAULT 'standard',
    status account_status NOT NULL DEFAULT 'active',
    billing_email VARCHAR(255) NOT NULL,
    billing_address JSONB NOT NULL DEFAULT '{}',
    tax_id VARCHAR(50),  -- Encrypted
    tax_exempt BOOLEAN NOT NULL DEFAULT FALSE,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    payment_terms VARCHAR(20) NOT NULL DEFAULT 'immediate',
    external_id VARCHAR(100),  -- Stripe customer ID
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT accounts_name_length CHECK (char_length(name) >= 2),
    CONSTRAINT accounts_currency_valid CHECK (currency ~ '^[A-Z]{3}$')
);

-- Add FK from divisions to accounts
ALTER TABLE divisions 
ADD CONSTRAINT fk_divisions_account 
FOREIGN KEY (account_id) REFERENCES accounts(id);

-- Indexes for Accounts
CREATE INDEX idx_accounts_organization ON accounts(organization_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_accounts_status ON accounts(status) WHERE deleted_at IS NULL;
CREATE INDEX idx_accounts_external ON accounts(external_id) WHERE external_id IS NOT NULL;

-- Audit Log Table (service-specific)
CREATE TABLE account_audit_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    organization_id UUID NOT NULL,
    action VARCHAR(50) NOT NULL,
    actor_id UUID,
    actor_type VARCHAR(20) NOT NULL DEFAULT 'user',
    changes JSONB NOT NULL DEFAULT '{}',
    metadata JSONB NOT NULL DEFAULT '{}',
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_entity ON account_audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_organization ON account_audit_logs(organization_id);
CREATE INDEX idx_audit_created ON account_audit_logs(created_at DESC);

-- Updated timestamp trigger
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_organizations_updated_at
    BEFORE UPDATE ON organizations
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_divisions_updated_at
    BEFORE UPDATE ON divisions
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_accounts_updated_at
    BEFORE UPDATE ON accounts
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### 2.3 TypeScript Interfaces

```typescript
// Organization Types
interface Organization {
  id: string;
  tenantId: string;
  name: string;
  slug: string;
  legalName?: string;
  logoUrl?: string;
  website?: string;
  industry?: string;
  companySize?: string;
  taxId?: string;
  primaryEmail: string;
  status: OrganizationStatus;
  settings: OrganizationSettings;
  metadata: Record<string, unknown>;
  createdAt: Date;
  updatedAt: Date;
  deletedAt?: Date;
}

type OrganizationStatus = 'trial' | 'active' | 'suspended' | 'churned' | 'deleted';

interface OrganizationSettings {
  timezone: string;
  dateFormat: string;
  currency: string;
  language: string;
  features?: Record<string, boolean>;
}

// Division Types
interface Division {
  id: string;
  organizationId: string;
  parentId?: string;
  accountId?: string;
  name: string;
  code?: string;
  description?: string;
  costCenter?: string;
  level: number;
  path: string;
  metadata: Record<string, unknown>;
  createdAt: Date;
  updatedAt: Date;
  deletedAt?: Date;
  // Virtual fields
  children?: Division[];
  parent?: Division;
}

// Account Types
interface Account {
  id: string;
  organizationId: string;
  name: string;
  type: AccountType;
  status: AccountStatus;
  billingEmail: string;
  billingAddress: BillingAddress;
  taxId?: string;
  taxExempt: boolean;
  currency: string;
  paymentTerms: string;
  externalId?: string;
  metadata: Record<string, unknown>;
  createdAt: Date;
  updatedAt: Date;
  deletedAt?: Date;
}

type AccountType = 'standard' | 'enterprise' | 'reseller';
type AccountStatus = 'active' | 'past_due' | 'suspended' | 'closed';

interface BillingAddress {
  line1: string;
  line2?: string;
  city: string;
  state?: string;
  postalCode: string;
  country: string;
}
```

---

## 3. API Contracts

### 3.1 Base URL
```
Production: https://api.flowmind.io/v1/account-service
Staging:    https://api.staging.flowmind.io/v1/account-service
```

### 3.2 Authentication
All endpoints require Bearer token authentication:
```
Authorization: Bearer <jwt_token>
```

### 3.3 Organization Endpoints

#### Create Organization
```yaml
POST /organizations

Request:
  Content-Type: application/json
  Body:
    name: string (required, min: 2, max: 255)
    slug: string (optional, auto-generated if not provided)
    legalName: string (optional)
    primaryEmail: string (required, email format)
    industry: string (optional)
    companySize: string (optional, enum: 1-10, 11-50, 51-200, 201-500, 500+)
    settings: object (optional)

Response: 201 Created
  {
    "success": true,
    "data": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "tenantId": "550e8400-e29b-41d4-a716-446655440001",
      "name": "Acme Corporation",
      "slug": "acme-corporation",
      "status": "trial",
      "primaryEmail": "admin@acme.com",
      "settings": {
        "timezone": "UTC",
        "dateFormat": "YYYY-MM-DD",
        "currency": "USD",
        "language": "en"
      },
      "createdAt": "2025-11-29T10:00:00Z",
      "updatedAt": "2025-11-29T10:00:00Z"
    }
  }

Errors:
  400: Validation error
  409: Slug already exists
  401: Unauthorized
```

#### Get Organization
```yaml
GET /organizations/{id}

Path Parameters:
  id: UUID (required)

Response: 200 OK
  {
    "success": true,
    "data": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "Acme Corporation",
      "slug": "acme-corporation",
      "legalName": "Acme Corporation Inc.",
      "logoUrl": "https://cdn.flowmind.io/logos/acme.png",
      "website": "https://acme.com",
      "industry": "Technology",
      "companySize": "51-200",
      "primaryEmail": "admin@acme.com",
      "status": "active",
      "settings": { ... },
      "metadata": { ... },
      "createdAt": "2025-11-29T10:00:00Z",
      "updatedAt": "2025-11-29T10:00:00Z",
      "_links": {
        "self": "/organizations/550e8400-e29b-41d4-a716-446655440000",
        "divisions": "/organizations/550e8400-e29b-41d4-a716-446655440000/divisions",
        "accounts": "/organizations/550e8400-e29b-41d4-a716-446655440000/accounts"
      }
    }
  }

Errors:
  404: Organization not found
  401: Unauthorized
  403: Forbidden (not member of organization)
```

#### Update Organization
```yaml
PATCH /organizations/{id}

Path Parameters:
  id: UUID (required)

Request:
  Content-Type: application/json
  Body: (all fields optional)
    name: string
    legalName: string
    logoUrl: string
    website: string
    industry: string
    companySize: string
    settings: object (merged with existing)
    metadata: object (merged with existing)

Response: 200 OK
  {
    "success": true,
    "data": { ... updated organization ... }
  }

Errors:
  400: Validation error
  404: Organization not found
  401: Unauthorized
  403: Forbidden
```

#### List Organizations
```yaml
GET /organizations

Query Parameters:
  page: integer (default: 1)
  limit: integer (default: 20, max: 100)
  status: string (filter by status)
  search: string (search by name)
  sort: string (default: -createdAt)

Response: 200 OK
  {
    "success": true,
    "data": [ ... organizations ... ],
    "meta": {
      "page": 1,
      "limit": 20,
      "total": 150,
      "totalPages": 8
    }
  }
```

#### Update Organization Status
```yaml
POST /organizations/{id}/status

Request:
  Content-Type: application/json
  Body:
    status: string (required, enum: trial, active, suspended, churned)
    reason: string (optional, required for suspend/churn)

Response: 200 OK
  {
    "success": true,
    "data": { ... updated organization ... }
  }
```

#### Delete Organization (Soft Delete)
```yaml
DELETE /organizations/{id}

Response: 204 No Content

Errors:
  404: Organization not found
  403: Forbidden
  409: Organization has active subscriptions
```

### 3.4 Division Endpoints

#### Create Division
```yaml
POST /organizations/{orgId}/divisions

Path Parameters:
  orgId: UUID (required)

Request:
  Content-Type: application/json
  Body:
    name: string (required)
    parentId: UUID (optional, null for root division)
    code: string (optional)
    description: string (optional)
    costCenter: string (optional)
    accountId: UUID (optional)
    metadata: object (optional)

Response: 201 Created
  {
    "success": true,
    "data": {
      "id": "660e8400-e29b-41d4-a716-446655440000",
      "organizationId": "550e8400-e29b-41d4-a716-446655440000",
      "parentId": null,
      "name": "Engineering",
      "code": "ENG",
      "level": 0,
      "path": "660e8400-e29b-41d4-a716-446655440000",
      "createdAt": "2025-11-29T10:00:00Z"
    }
  }

Errors:
  400: Validation error / Max depth exceeded
  404: Organization or parent division not found
  409: Division name already exists under parent
```

#### Get Division Tree
```yaml
GET /organizations/{orgId}/divisions/tree

Path Parameters:
  orgId: UUID (required)

Query Parameters:
  rootId: UUID (optional, get subtree from this division)
  maxDepth: integer (optional, default: all levels)

Response: 200 OK
  {
    "success": true,
    "data": [
      {
        "id": "660e8400-e29b-41d4-a716-446655440000",
        "name": "Engineering",
        "code": "ENG",
        "level": 0,
        "children": [
          {
            "id": "770e8400-e29b-41d4-a716-446655440000",
            "name": "Frontend",
            "code": "ENG-FE",
            "level": 1,
            "children": []
          },
          {
            "id": "880e8400-e29b-41d4-a716-446655440000",
            "name": "Backend",
            "code": "ENG-BE",
            "level": 1,
            "children": []
          }
        ]
      }
    ]
  }
```

#### List Divisions (Flat)
```yaml
GET /organizations/{orgId}/divisions

Query Parameters:
  page: integer
  limit: integer
  parentId: UUID (filter by parent)
  search: string

Response: 200 OK
  {
    "success": true,
    "data": [ ... divisions ... ],
    "meta": { ... pagination ... }
  }
```

#### Update Division
```yaml
PATCH /organizations/{orgId}/divisions/{id}

Request:
  Body:
    name: string
    code: string
    description: string
    costCenter: string
    metadata: object

Response: 200 OK
```

#### Move Division
```yaml
POST /organizations/{orgId}/divisions/{id}/move

Request:
  Body:
    newParentId: UUID (null for root level)

Response: 200 OK
  {
    "success": true,
    "data": { ... updated division with new path ... }
  }

Errors:
  400: Would create circular reference / Max depth exceeded
  404: Division or new parent not found
```

#### Delete Division
```yaml
DELETE /organizations/{orgId}/divisions/{id}

Query Parameters:
  cascade: boolean (default: false, if true deletes children)

Response: 204 No Content

Errors:
  409: Division has children (when cascade=false)
```

### 3.5 Account Endpoints

#### Create Account
```yaml
POST /organizations/{orgId}/accounts

Request:
  Body:
    name: string (required)
    type: string (default: standard)
    billingEmail: string (required)
    billingAddress: object (required)
      line1: string (required)
      line2: string
      city: string (required)
      state: string
      postalCode: string (required)
      country: string (required, ISO 3166-1 alpha-2)
    taxId: string
    taxExempt: boolean
    currency: string (default: USD)
    paymentTerms: string (default: immediate)

Response: 201 Created
```

#### Get Account
```yaml
GET /organizations/{orgId}/accounts/{id}

Response: 200 OK
```

#### List Accounts
```yaml
GET /organizations/{orgId}/accounts

Query Parameters:
  status: string
  type: string

Response: 200 OK
```

#### Update Account
```yaml
PATCH /organizations/{orgId}/accounts/{id}

Request:
  Body: (partial update)
    name: string
    billingEmail: string
    billingAddress: object
    taxExempt: boolean
    paymentTerms: string

Response: 200 OK
```

#### Update Account Status
```yaml
POST /organizations/{orgId}/accounts/{id}/status

Request:
  Body:
    status: string (enum: active, suspended, closed)
    reason: string

Response: 200 OK
```

---

## 4. Event Contracts

### 4.1 Event Format
```typescript
interface DomainEvent<T> {
  eventId: string;           // UUID
  eventType: string;         // e.g., "organization.created"
  aggregateType: string;     // e.g., "Organization"
  aggregateId: string;       // Entity ID
  tenantId: string;          // Organization's tenant ID
  timestamp: string;         // ISO 8601
  version: number;           // Event schema version
  correlationId: string;     // Request correlation ID
  causationId?: string;      // ID of event that caused this
  actor: {
    id: string;
    type: 'user' | 'system' | 'service';
  };
  data: T;                   // Event-specific payload
  metadata: Record<string, unknown>;
}
```

### 4.2 Published Events

#### Organization Events
```typescript
// organization.created
interface OrganizationCreatedEvent {
  id: string;
  name: string;
  slug: string;
  primaryEmail: string;
  status: 'trial';
}

// organization.updated
interface OrganizationUpdatedEvent {
  id: string;
  changes: {
    field: string;
    oldValue: unknown;
    newValue: unknown;
  }[];
}

// organization.status_changed
interface OrganizationStatusChangedEvent {
  id: string;
  previousStatus: OrganizationStatus;
  newStatus: OrganizationStatus;
  reason?: string;
}

// organization.deleted
interface OrganizationDeletedEvent {
  id: string;
  deletedAt: string;
}
```

#### Division Events
```typescript
// division.created
interface DivisionCreatedEvent {
  id: string;
  organizationId: string;
  name: string;
  parentId?: string;
  path: string;
}

// division.moved
interface DivisionMovedEvent {
  id: string;
  organizationId: string;
  previousParentId?: string;
  newParentId?: string;
  previousPath: string;
  newPath: string;
  affectedDivisionIds: string[];  // IDs of all divisions whose path changed
}

// division.deleted
interface DivisionDeletedEvent {
  id: string;
  organizationId: string;
  cascadeDeleted: boolean;
  deletedChildrenIds?: string[];
}
```

#### Account Events
```typescript
// account.created
interface AccountCreatedEvent {
  id: string;
  organizationId: string;
  name: string;
  type: AccountType;
  currency: string;
}

// account.status_changed
interface AccountStatusChangedEvent {
  id: string;
  organizationId: string;
  previousStatus: AccountStatus;
  newStatus: AccountStatus;
  reason?: string;
}
```

### 4.3 Event Topics/Exchanges
```
Topic: flowmind.account-service.events
Routing Keys:
  - organization.created
  - organization.updated
  - organization.status_changed
  - organization.deleted
  - division.created
  - division.updated
  - division.moved
  - division.deleted
  - account.created
  - account.updated
  - account.status_changed
```

---

## 5. Security Considerations

### 5.1 Authentication
- All API endpoints require valid JWT token
- Token must contain `tenantId` claim
- Service-to-service calls use API keys with service identity

### 5.2 Authorization
```typescript
// Permission model
enum AccountServicePermission {
  // Organization
  ORGANIZATION_READ = 'organization:read',
  ORGANIZATION_UPDATE = 'organization:update',
  ORGANIZATION_DELETE = 'organization:delete',
  
  // Division
  DIVISION_CREATE = 'division:create',
  DIVISION_READ = 'division:read',
  DIVISION_UPDATE = 'division:update',
  DIVISION_DELETE = 'division:delete',
  
  // Account
  ACCOUNT_CREATE = 'account:create',
  ACCOUNT_READ = 'account:read',
  ACCOUNT_UPDATE = 'account:update',
  ACCOUNT_MANAGE = 'account:manage',  // Status changes
}
```

### 5.3 Data Protection
- Tax IDs encrypted using AES-256-GCM
- Encryption keys stored in AWS KMS / HashiCorp Vault
- PII fields marked for GDPR compliance

### 5.4 Rate Limiting
| Endpoint Pattern | Rate Limit |
|-----------------|------------|
| GET /organizations/* | 100/min |
| POST /organizations | 10/min |
| PATCH /organizations/* | 30/min |
| DELETE /organizations/* | 5/min |

---

## 6. Integration Points

### 6.1 Consumed Services
| Service | Purpose | Protocol |
|---------|---------|----------|
| Identity Service | Token validation | REST |
| IAM Service | Permission checks | REST/gRPC |
| Feature Flags | Feature toggles | REST |

### 6.2 Consumed By
| Service | Purpose |
|---------|---------|
| IAM Service | User-org association |
| Subscription Service | Billing account lookup |
| Product Service | Org entitlements |
| All Services | Tenant context |

### 6.3 Event Consumers
| Service | Events Consumed | Purpose |
|---------|----------------|---------|
| IAM Service | organization.created | Create default admin role |
| Notification Service | organization.created | Send welcome email |
| Subscription Service | account.created | Sync to payment provider |
| Audit Service | All events | Compliance logging |

---

## 7. Caching Strategy

### 7.1 Cache Keys
```
org:{id}                    - Organization by ID (TTL: 5min)
org:slug:{slug}             - Organization by slug (TTL: 5min)
org:{id}:divisions          - Division tree (TTL: 2min)
org:{id}:accounts           - Accounts list (TTL: 5min)
```

### 7.2 Cache Invalidation
- Invalidate on any write operation
- Publish cache invalidation events for distributed cache
- Use cache-aside pattern

---

## 8. Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| ACC_001 | 400 | Invalid organization name |
| ACC_002 | 409 | Organization slug already exists |
| ACC_003 | 404 | Organization not found |
| ACC_004 | 403 | Cannot modify organization |
| ACC_010 | 400 | Invalid division hierarchy |
| ACC_011 | 400 | Maximum division depth exceeded |
| ACC_012 | 409 | Division name conflict |
| ACC_013 | 409 | Cannot delete division with children |
| ACC_020 | 400 | Invalid billing address |
| ACC_021 | 409 | Cannot close account with active subscriptions |

---

*FlowMind Technologies | Account Service Technical Specification v1.0*

