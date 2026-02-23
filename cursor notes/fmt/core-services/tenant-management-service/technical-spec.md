# Tenant Management Service - Technical Specification

> **Service**: Tenant Management Service  
> **Version**: 1.0  
> **Owner**: Platform Team  
> **Last Updated**: November 2025

---

## 1. Architecture Overview

### 1.1 Service Context
```
┌─────────────────────────────────────────────────────────────────────────┐
│                           API Gateway                                    │
│              (Tenant Resolution via X-Tenant-ID or Domain)              │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      Tenant Management Service                           │
│                                                                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐  │
│  │ Tenant Resolver │  │ Provisioning    │  │ Domain Manager          │  │
│  │ (gRPC + Cache)  │  │ Orchestrator    │  │ (SSL, DNS)              │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘  │
│                                                                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐  │
│  │ Quota Manager   │  │ Health Monitor  │  │ Isolation Controller    │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
        ┌──────────┐     ┌──────────┐     ┌──────────────┐
        │PostgreSQL│     │  Redis   │     │Let's Encrypt │
        │          │     │ (Cache)  │     │   / Cert Mgr │
        └──────────┘     └──────────┘     └──────────────┘
```

### 1.2 Technology Stack
| Component | Technology |
|-----------|------------|
| Runtime | Node.js 20 / Go |
| Framework | NestJS / Gin |
| Database | PostgreSQL 15 |
| Cache | Redis 7 |
| DNS | Cloudflare API / Route53 |
| SSL | Let's Encrypt / AWS ACM |

---

## 2. Data Model

### 2.1 Database Schema

```sql
-- Enums
CREATE TYPE tenant_status AS ENUM ('provisioning', 'active', 'suspended', 'decommissioning', 'archived');
CREATE TYPE isolation_level AS ENUM ('shared', 'dedicated_compute', 'dedicated_all');
CREATE TYPE domain_status AS ENUM ('pending_verification', 'verified', 'active', 'expired', 'failed');
CREATE TYPE quota_enforcement AS ENUM ('soft', 'hard');

-- Tenants
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL UNIQUE,
    slug VARCHAR(100) NOT NULL UNIQUE,
    
    -- Status
    status tenant_status NOT NULL DEFAULT 'provisioning',
    status_reason TEXT,
    
    -- Isolation
    isolation_level isolation_level NOT NULL DEFAULT 'shared',
    data_region VARCHAR(20) NOT NULL DEFAULT 'us-east-1',
    
    -- Database configuration
    db_host VARCHAR(255),
    db_port INTEGER DEFAULT 5432,
    db_name VARCHAR(100),
    db_schema VARCHAR(100) DEFAULT 'public',
    db_pool_size INTEGER DEFAULT 10,
    
    -- Cache configuration
    cache_prefix VARCHAR(50),
    cache_pool VARCHAR(100),
    
    -- Keycloak
    keycloak_realm VARCHAR(100),
    
    -- Settings
    settings JSONB NOT NULL DEFAULT '{}',
    feature_overrides JSONB NOT NULL DEFAULT '{}',
    
    -- Timestamps
    provisioned_at TIMESTAMP WITH TIME ZONE,
    suspended_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tenants_org ON tenants(organization_id);
CREATE INDEX idx_tenants_slug ON tenants(slug);
CREATE INDEX idx_tenants_status ON tenants(status);

-- Custom Domains
CREATE TABLE tenant_domains (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    
    domain VARCHAR(255) NOT NULL UNIQUE,
    domain_type VARCHAR(20) NOT NULL DEFAULT 'app',  -- app, api, auth
    
    -- Verification
    status domain_status NOT NULL DEFAULT 'pending_verification',
    verification_method VARCHAR(20) DEFAULT 'dns',  -- dns, http
    verification_token VARCHAR(100) NOT NULL,
    verified_at TIMESTAMP WITH TIME ZONE,
    
    -- SSL
    ssl_status VARCHAR(20) DEFAULT 'pending',
    ssl_certificate_arn VARCHAR(255),  -- AWS ACM or stored cert reference
    ssl_expires_at TIMESTAMP WITH TIME ZONE,
    ssl_auto_renew BOOLEAN NOT NULL DEFAULT TRUE,
    
    -- Metadata
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_domains_tenant ON tenant_domains(tenant_id);
CREATE INDEX idx_domains_domain ON tenant_domains(domain);
CREATE INDEX idx_domains_status ON tenant_domains(status);

-- Resource Quotas
CREATE TABLE tenant_quotas (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    
    quota_type VARCHAR(50) NOT NULL,  -- users, api_calls, storage_gb, etc.
    limit_value BIGINT NOT NULL,
    current_value BIGINT NOT NULL DEFAULT 0,
    
    enforcement quota_enforcement NOT NULL DEFAULT 'hard',
    warning_threshold_percent INTEGER DEFAULT 80,
    
    reset_period VARCHAR(20),  -- monthly, daily, never
    last_reset_at TIMESTAMP WITH TIME ZONE,
    next_reset_at TIMESTAMP WITH TIME ZONE,
    
    -- Override from subscription
    subscription_limit BIGINT,
    override_limit BIGINT,  -- Manual override
    override_expires_at TIMESTAMP WITH TIME ZONE,
    override_reason TEXT,
    
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT tenant_quotas_unique UNIQUE (tenant_id, quota_type)
);

CREATE INDEX idx_quotas_tenant ON tenant_quotas(tenant_id);

-- Quota usage history (for trending)
CREATE TABLE quota_usage_history (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL,
    quota_type VARCHAR(50) NOT NULL,
    usage_value BIGINT NOT NULL,
    recorded_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_quota_history_tenant ON quota_usage_history(tenant_id, quota_type, recorded_at DESC);

-- Tenant health snapshots
CREATE TABLE tenant_health (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    
    health_score INTEGER NOT NULL,  -- 0-100
    status VARCHAR(20) NOT NULL,    -- healthy, degraded, unhealthy
    
    metrics JSONB NOT NULL,  -- Detailed metrics
    issues JSONB NOT NULL DEFAULT '[]',  -- Current issues
    
    recorded_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_health_tenant ON tenant_health(tenant_id, recorded_at DESC);

-- Provisioning tasks
CREATE TABLE provisioning_tasks (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    
    task_type VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    
    started_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE,
    error_message TEXT,
    retry_count INTEGER NOT NULL DEFAULT 0,
    
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_provisioning_tenant ON provisioning_tasks(tenant_id);
CREATE INDEX idx_provisioning_status ON provisioning_tasks(status);
```

### 2.2 TypeScript Interfaces

```typescript
// Tenant
interface Tenant {
  id: string;
  organizationId: string;
  slug: string;
  status: TenantStatus;
  statusReason?: string;
  
  // Isolation
  isolationLevel: IsolationLevel;
  dataRegion: string;
  
  // Database config
  dbConfig: DatabaseConfig;
  
  // Cache config
  cachePrefix: string;
  cachePool?: string;
  
  // Keycloak
  keycloakRealm: string;
  
  // Settings
  settings: TenantSettings;
  featureOverrides: Record<string, boolean>;
  
  provisionedAt?: Date;
  suspendedAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}

type TenantStatus = 'provisioning' | 'active' | 'suspended' | 'decommissioning' | 'archived';
type IsolationLevel = 'shared' | 'dedicated_compute' | 'dedicated_all';

interface DatabaseConfig {
  host: string;
  port: number;
  name: string;
  schema: string;
  poolSize: number;
}

interface TenantSettings {
  timezone: string;
  locale: string;
  features: Record<string, unknown>;
}

// Custom Domain
interface TenantDomain {
  id: string;
  tenantId: string;
  domain: string;
  domainType: 'app' | 'api' | 'auth';
  status: DomainStatus;
  verificationMethod: 'dns' | 'http';
  verificationToken: string;
  verifiedAt?: Date;
  sslStatus: 'pending' | 'provisioning' | 'active' | 'expired' | 'failed';
  sslExpiresAt?: Date;
  sslAutoRenew: boolean;
}

type DomainStatus = 'pending_verification' | 'verified' | 'active' | 'expired' | 'failed';

// Quota
interface TenantQuota {
  id: string;
  tenantId: string;
  quotaType: QuotaType;
  limitValue: number;
  currentValue: number;
  enforcement: 'soft' | 'hard';
  warningThresholdPercent: number;
  resetPeriod?: 'monthly' | 'daily' | 'never';
  nextResetAt?: Date;
  subscriptionLimit?: number;
  overrideLimit?: number;
  overrideExpiresAt?: Date;
}

type QuotaType = 
  | 'users'
  | 'api_calls_monthly'
  | 'storage_gb'
  | 'data_transfer_gb'
  | 'divisions'
  | 'integrations'
  | 'projects'
  | 'api_keys';

// Tenant Resolution
interface TenantContext {
  tenantId: string;
  organizationId: string;
  slug: string;
  isolationLevel: IsolationLevel;
  dbConfig: DatabaseConfig;
  cachePrefix: string;
  quotas: Record<QuotaType, QuotaStatus>;
}

interface QuotaStatus {
  limit: number;
  current: number;
  percentUsed: number;
  isExceeded: boolean;
  enforcement: 'soft' | 'hard';
}
```

---

## 3. API Contracts

### 3.1 Tenant Endpoints

#### Get Tenant
```yaml
GET /v1/tenants/{tenantId}

Response: 200 OK
  {
    "success": true,
    "data": {
      "id": "uuid",
      "organizationId": "uuid",
      "slug": "acme-corp",
      "status": "active",
      "isolationLevel": "shared",
      "dataRegion": "us-east-1",
      "keycloakRealm": "flowmind-acme-corp",
      "settings": { ... },
      "quotas": {
        "users": { "limit": 100, "current": 45, "percentUsed": 45 },
        "api_calls_monthly": { "limit": 100000, "current": 23456, "percentUsed": 23 }
      },
      "provisionedAt": "2025-11-29T10:00:00Z"
    }
  }
```

#### Get Tenant by Slug
```yaml
GET /v1/tenants/by-slug/{slug}

Response: 200 OK
```

#### Update Tenant Settings
```yaml
PATCH /v1/tenants/{tenantId}

Request:
  {
    "settings": {
      "timezone": "America/Los_Angeles"
    },
    "featureOverrides": {
      "beta_feature_x": true
    }
  }

Response: 200 OK
```

#### Upgrade Isolation Level
```yaml
POST /v1/tenants/{tenantId}/upgrade-isolation

Request:
  {
    "targetLevel": "dedicated_compute",
    "scheduledAt": "2025-12-01T00:00:00Z",
    "reason": "Enterprise upgrade"
  }

Response: 202 Accepted
  {
    "success": true,
    "data": {
      "migrationId": "uuid",
      "status": "scheduled",
      "scheduledAt": "2025-12-01T00:00:00Z",
      "estimatedDuration": "30 minutes"
    }
  }
```

#### Suspend Tenant
```yaml
POST /v1/tenants/{tenantId}/suspend

Request:
  {
    "reason": "Payment overdue",
    "gracePeriodHours": 72
  }

Response: 200 OK
```

### 3.2 Domain Endpoints

#### Add Custom Domain
```yaml
POST /v1/tenants/{tenantId}/domains

Request:
  {
    "domain": "app.customer.com",
    "domainType": "app"
  }

Response: 201 Created
  {
    "success": true,
    "data": {
      "id": "uuid",
      "domain": "app.customer.com",
      "status": "pending_verification",
      "verification": {
        "method": "dns",
        "recordType": "TXT",
        "recordName": "_flowmind-verification.app.customer.com",
        "recordValue": "flowmind-verify=abc123xyz"
      },
      "instructions": "Add the following DNS record to verify domain ownership..."
    }
  }
```

#### Verify Domain
```yaml
POST /v1/tenants/{tenantId}/domains/{domainId}/verify

Response: 200 OK
  {
    "success": true,
    "data": {
      "status": "verified",
      "sslStatus": "provisioning",
      "estimatedSslTime": "5 minutes"
    }
  }

Errors:
  400: Verification failed - DNS record not found
```

#### List Domains
```yaml
GET /v1/tenants/{tenantId}/domains

Response: 200 OK
  {
    "success": true,
    "data": [
      {
        "id": "uuid",
        "domain": "app.customer.com",
        "domainType": "app",
        "status": "active",
        "sslStatus": "active",
        "sslExpiresAt": "2026-02-28T00:00:00Z"
      }
    ]
  }
```

#### Remove Domain
```yaml
DELETE /v1/tenants/{tenantId}/domains/{domainId}

Response: 204 No Content
```

### 3.3 Quota Endpoints

#### Get Quotas
```yaml
GET /v1/tenants/{tenantId}/quotas

Response: 200 OK
  {
    "success": true,
    "data": {
      "users": {
        "limit": 100,
        "current": 45,
        "percentUsed": 45,
        "enforcement": "hard",
        "resetPeriod": "never"
      },
      "api_calls_monthly": {
        "limit": 100000,
        "current": 23456,
        "percentUsed": 23.46,
        "enforcement": "soft",
        "resetPeriod": "monthly",
        "nextResetAt": "2025-12-01T00:00:00Z"
      },
      "storage_gb": {
        "limit": 50,
        "current": 12.5,
        "percentUsed": 25,
        "enforcement": "hard"
      }
    }
  }
```

#### Update Quota Override
```yaml
PUT /v1/tenants/{tenantId}/quotas/{quotaType}/override

Request:
  {
    "overrideLimit": 200,
    "expiresAt": "2026-01-01T00:00:00Z",
    "reason": "Temporary increase for migration"
  }

Response: 200 OK
```

#### Increment Quota Usage (Internal)
```yaml
POST /v1/tenants/{tenantId}/quotas/{quotaType}/increment

Request:
  {
    "amount": 1
  }

Response: 200 OK
  {
    "success": true,
    "data": {
      "newValue": 46,
      "limit": 100,
      "percentUsed": 46,
      "allowed": true
    }
  }

Errors:
  429: Quota exceeded (for hard enforcement)
```

### 3.4 Tenant Resolution (gRPC - High Performance)

```protobuf
syntax = "proto3";

package flowmind.tenant.v1;

service TenantService {
  // Fast tenant resolution for API Gateway
  rpc ResolveTenant(ResolveTenantRequest) returns (ResolveTenantResponse);
  
  // Resolve by custom domain
  rpc ResolveByDomain(ResolveByDomainRequest) returns (ResolveTenantResponse);
  
  // Check quota (with optional increment)
  rpc CheckQuota(CheckQuotaRequest) returns (CheckQuotaResponse);
  
  // Batch quota increment (for async processing)
  rpc IncrementQuotaBatch(IncrementQuotaBatchRequest) returns (IncrementQuotaBatchResponse);
}

message ResolveTenantRequest {
  string tenant_id = 1;
}

message ResolveByDomainRequest {
  string domain = 1;
}

message ResolveTenantResponse {
  bool found = 1;
  string tenant_id = 2;
  string organization_id = 3;
  string slug = 4;
  string isolation_level = 5;
  string status = 6;
  DatabaseConfig db_config = 7;
  string cache_prefix = 8;
  map<string, QuotaStatus> quotas = 9;
}

message DatabaseConfig {
  string host = 1;
  int32 port = 2;
  string name = 3;
  string schema = 4;
}

message QuotaStatus {
  int64 limit = 1;
  int64 current = 2;
  double percent_used = 3;
  bool is_exceeded = 4;
  string enforcement = 5;
}

message CheckQuotaRequest {
  string tenant_id = 1;
  string quota_type = 2;
  int64 requested_amount = 3;
  bool dry_run = 4;  // Check without incrementing
}

message CheckQuotaResponse {
  bool allowed = 1;
  int64 current_value = 2;
  int64 limit = 3;
  int64 remaining = 4;
  string denial_reason = 5;
}
```

---

## 4. Event Contracts

### 4.1 Published Events

```typescript
// Tenant lifecycle events
interface TenantProvisionedEvent {
  eventType: 'tenant.provisioned';
  tenantId: string;
  organizationId: string;
  slug: string;
  isolationLevel: IsolationLevel;
  dataRegion: string;
}

interface TenantSuspendedEvent {
  eventType: 'tenant.suspended';
  tenantId: string;
  reason: string;
  suspendedBy: string;
  gracePeriodEndsAt?: string;
}

interface TenantReactivatedEvent {
  eventType: 'tenant.reactivated';
  tenantId: string;
}

// Domain events
interface DomainAddedEvent {
  eventType: 'tenant.domain_added';
  tenantId: string;
  domainId: string;
  domain: string;
  domainType: string;
}

interface DomainVerifiedEvent {
  eventType: 'tenant.domain_verified';
  tenantId: string;
  domainId: string;
  domain: string;
}

interface DomainActivatedEvent {
  eventType: 'tenant.domain_activated';
  tenantId: string;
  domainId: string;
  domain: string;
  sslStatus: string;
}

// Quota events
interface QuotaWarningEvent {
  eventType: 'tenant.quota_warning';
  tenantId: string;
  quotaType: QuotaType;
  currentValue: number;
  limitValue: number;
  percentUsed: number;
  threshold: number;
}

interface QuotaExceededEvent {
  eventType: 'tenant.quota_exceeded';
  tenantId: string;
  quotaType: QuotaType;
  enforcement: 'soft' | 'hard';
}

interface QuotaResetEvent {
  eventType: 'tenant.quota_reset';
  tenantId: string;
  quotaType: QuotaType;
  previousValue: number;
}
```

### 4.2 Consumed Events

```typescript
// From Account Service
interface OrganizationCreatedEvent {
  eventType: 'organization.created';
  id: string;
  slug: string;
}
// → Triggers tenant provisioning

// From Subscription Service
interface SubscriptionChangedEvent {
  eventType: 'subscription.changed';
  organizationId: string;
  tierId: string;
  quotas: Record<QuotaType, number>;
}
// → Updates tenant quotas
```

---

## 5. Provisioning Workflow

### 5.1 Tenant Provisioning Steps

```typescript
async function provisionTenant(organizationId: string, slug: string): Promise<Tenant> {
  const tenant = await db.tenants.create({
    organizationId,
    slug,
    status: 'provisioning',
    isolationLevel: 'shared',
    cachePrefix: `t:${slug}:`
  });
  
  // Queue provisioning tasks
  const tasks = [
    { type: 'create_db_schema', order: 1 },
    { type: 'create_keycloak_realm', order: 2 },
    { type: 'setup_default_quotas', order: 3 },
    { type: 'configure_cache', order: 4 },
    { type: 'verify_connectivity', order: 5 }
  ];
  
  await db.provisioningTasks.createMany(
    tasks.map(t => ({ ...t, tenantId: tenant.id }))
  );
  
  // Start provisioning
  await queue.publish('tenant.provisioning.start', { tenantId: tenant.id });
  
  return tenant;
}

async function executeProvisioningTask(task: ProvisioningTask): Promise<void> {
  switch (task.taskType) {
    case 'create_db_schema':
      await createDatabaseSchema(task.tenantId);
      break;
    case 'create_keycloak_realm':
      await keycloak.createRealm(task.tenantId);
      break;
    case 'setup_default_quotas':
      await setupDefaultQuotas(task.tenantId);
      break;
    // ... etc
  }
}
```

### 5.2 Database Schema Creation (Shared Tenancy)

```sql
-- For shared tenancy, we don't create separate schemas
-- Instead, ensure tenant_id column and RLS policies

-- Example: Enable RLS on a product service table
ALTER TABLE products ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON products
    USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- Set tenant context in connection
SET app.tenant_id = 'tenant-uuid';
```

---

## 6. Caching Strategy

### 6.1 Cache Keys
```
tenant:{id}                    - Full tenant context (TTL: 5min)
tenant:slug:{slug}             - Tenant ID lookup (TTL: 5min)
tenant:domain:{domain}         - Domain → Tenant mapping (TTL: 10min)
tenant:{id}:quotas             - Quota summary (TTL: 1min)
```

### 6.2 Tenant Resolution Caching
```typescript
async function resolveTenant(tenantId: string): Promise<TenantContext> {
  // Try cache first
  const cached = await redis.get(`tenant:${tenantId}`);
  if (cached) {
    return JSON.parse(cached);
  }
  
  // Load from DB
  const tenant = await db.tenants.findById(tenantId);
  const quotas = await db.quotas.findByTenant(tenantId);
  
  const context: TenantContext = {
    tenantId: tenant.id,
    organizationId: tenant.organizationId,
    slug: tenant.slug,
    isolationLevel: tenant.isolationLevel,
    dbConfig: tenant.dbConfig,
    cachePrefix: tenant.cachePrefix,
    quotas: formatQuotas(quotas)
  };
  
  // Cache with TTL
  await redis.setex(`tenant:${tenantId}`, 300, JSON.stringify(context));
  
  return context;
}
```

---

## 7. Domain Verification & SSL

### 7.1 DNS Verification Flow
```typescript
async function verifyDomain(domain: TenantDomain): Promise<boolean> {
  const expectedRecord = `flowmind-verify=${domain.verificationToken}`;
  
  // Query DNS
  const txtRecords = await dns.resolveTxt(
    `_flowmind-verification.${domain.domain}`
  );
  
  const verified = txtRecords.flat().some(r => r === expectedRecord);
  
  if (verified) {
    await db.domains.update(domain.id, {
      status: 'verified',
      verifiedAt: new Date()
    });
    
    // Trigger SSL provisioning
    await queue.publish('domain.ssl.provision', { domainId: domain.id });
  }
  
  return verified;
}
```

### 7.2 SSL Certificate Provisioning
```typescript
async function provisionSSL(domain: TenantDomain): Promise<void> {
  // Using Let's Encrypt via ACME
  const cert = await acmeClient.getCertificate({
    domain: domain.domain,
    challenges: ['http-01', 'dns-01']
  });
  
  // Store certificate
  await secretsManager.store(`ssl:${domain.domain}`, {
    certificate: cert.certificate,
    privateKey: cert.privateKey,
    chain: cert.chain
  });
  
  // Update domain status
  await db.domains.update(domain.id, {
    sslStatus: 'active',
    sslExpiresAt: cert.expiresAt
  });
  
  // Update ingress/load balancer
  await ingressController.updateCertificate(domain.domain, cert);
  
  // Publish event
  await eventBus.publish('tenant.domain_activated', {
    tenantId: domain.tenantId,
    domain: domain.domain
  });
}
```

---

## 8. Quota Management

### 8.1 Default Quotas by Tier
```typescript
const DEFAULT_QUOTAS: Record<string, Record<QuotaType, number>> = {
  free: {
    users: 5,
    api_calls_monthly: 10000,
    storage_gb: 1,
    divisions: 1,
    integrations: 2
  },
  pro: {
    users: 50,
    api_calls_monthly: 100000,
    storage_gb: 50,
    divisions: 10,
    integrations: 10
  },
  enterprise: {
    users: -1,  // Unlimited
    api_calls_monthly: -1,
    storage_gb: 1000,
    divisions: -1,
    integrations: -1
  }
};
```

### 8.2 Quota Check Implementation
```typescript
async function checkAndIncrementQuota(
  tenantId: string,
  quotaType: QuotaType,
  amount: number = 1
): Promise<QuotaCheckResult> {
  const quota = await db.quotas.findOne(tenantId, quotaType);
  
  // Unlimited check
  if (quota.limitValue === -1) {
    return { allowed: true, unlimited: true };
  }
  
  const newValue = quota.currentValue + amount;
  const percentUsed = (newValue / quota.limitValue) * 100;
  
  // Check if would exceed
  if (newValue > quota.limitValue) {
    if (quota.enforcement === 'hard') {
      return {
        allowed: false,
        reason: 'quota_exceeded',
        current: quota.currentValue,
        limit: quota.limitValue
      };
    }
    // Soft enforcement - allow but warn
  }
  
  // Increment
  await db.quotas.increment(tenantId, quotaType, amount);
  
  // Check warning thresholds
  if (percentUsed >= quota.warningThresholdPercent) {
    await eventBus.publish('tenant.quota_warning', {
      tenantId,
      quotaType,
      percentUsed
    });
  }
  
  return {
    allowed: true,
    current: newValue,
    limit: quota.limitValue,
    percentUsed
  };
}
```

---

## 9. Health Monitoring

### 9.1 Health Check Implementation
```typescript
async function checkTenantHealth(tenantId: string): Promise<TenantHealth> {
  const checks = await Promise.all([
    checkDatabaseHealth(tenantId),
    checkCacheHealth(tenantId),
    checkKeycloakHealth(tenantId),
    checkQuotaHealth(tenantId)
  ]);
  
  const issues = checks.flatMap(c => c.issues);
  const healthScore = calculateHealthScore(checks);
  
  const status = healthScore >= 90 ? 'healthy' 
    : healthScore >= 70 ? 'degraded' 
    : 'unhealthy';
  
  await db.tenantHealth.create({
    tenantId,
    healthScore,
    status,
    metrics: checks,
    issues
  });
  
  return { healthScore, status, issues };
}
```

---

*FlowMind Technologies | Tenant Management Service Technical Specification v1.0*

