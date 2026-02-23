# Product & Feature Service - Technical Specification

> **Service**: Product & Feature Service  
> **Version**: 1.0  
> **Owner**: Platform Team  
> **Last Updated**: November 2025

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Product Service                           │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Products   │  │   Tiers     │  │     Features        │  │
│  │  Manager    │  │   Manager   │  │     & Entitlements  │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Data Model

### 2.1 Database Schema

```sql
-- Products
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    short_description VARCHAR(255),
    icon_url VARCHAR(500),
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Pricing Tiers
CREATE TABLE tiers (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    product_id UUID NOT NULL REFERENCES products(id),
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) NOT NULL,
    description TEXT,
    
    -- Pricing
    price_monthly DECIMAL(10,2),
    price_annual DECIMAL(10,2),
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    
    -- Display
    display_order INTEGER NOT NULL DEFAULT 0,
    is_popular BOOLEAN NOT NULL DEFAULT FALSE,
    is_public BOOLEAN NOT NULL DEFAULT TRUE,
    
    -- Stripe
    stripe_price_id_monthly VARCHAR(100),
    stripe_price_id_annual VARCHAR(100),
    
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT tiers_unique_slug UNIQUE (product_id, slug)
);

-- Features
CREATE TABLE features (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(100) NOT NULL UNIQUE,
    display_name VARCHAR(255) NOT NULL,
    description TEXT,
    category VARCHAR(50),
    
    feature_type VARCHAR(20) NOT NULL DEFAULT 'boolean',  -- boolean, limit, metered
    unit VARCHAR(50),  -- For limits: 'users', 'GB', etc.
    
    is_public BOOLEAN NOT NULL DEFAULT TRUE,
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Tier Entitlements (feature access per tier)
CREATE TABLE tier_entitlements (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tier_id UUID NOT NULL REFERENCES tiers(id) ON DELETE CASCADE,
    feature_id UUID NOT NULL REFERENCES features(id),
    
    enabled BOOLEAN NOT NULL DEFAULT TRUE,
    limit_value INTEGER,  -- For limit features
    
    metadata JSONB NOT NULL DEFAULT '{}',
    
    CONSTRAINT entitlements_unique UNIQUE (tier_id, feature_id)
);

-- Organization Entitlement Overrides
CREATE TABLE organization_entitlements (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL,
    feature_id UUID NOT NULL REFERENCES features(id),
    
    enabled BOOLEAN,
    limit_value INTEGER,
    
    granted_by UUID NOT NULL,
    reason TEXT,
    expires_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT org_entitlements_unique UNIQUE (organization_id, feature_id)
);

-- Add-ons
CREATE TABLE addons (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    product_id UUID REFERENCES products(id),  -- NULL if cross-product
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    
    price DECIMAL(10,2) NOT NULL,
    billing_type VARCHAR(20) NOT NULL DEFAULT 'recurring',  -- one_time, recurring
    
    stripe_price_id VARCHAR(100),
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Addon-Tier compatibility
CREATE TABLE addon_tier_compatibility (
    addon_id UUID NOT NULL REFERENCES addons(id),
    tier_id UUID NOT NULL REFERENCES tiers(id),
    included BOOLEAN NOT NULL DEFAULT FALSE,  -- TRUE if included in tier
    
    PRIMARY KEY (addon_id, tier_id)
);
```

### 2.2 TypeScript Interfaces

```typescript
interface Product {
  id: string;
  name: string;
  slug: string;
  description?: string;
  shortDescription?: string;
  iconUrl?: string;
  status: 'active' | 'coming_soon' | 'deprecated' | 'retired';
  tiers?: Tier[];
}

interface Tier {
  id: string;
  productId: string;
  name: string;
  slug: string;
  description?: string;
  priceMonthly?: number;
  priceAnnual?: number;
  currency: string;
  displayOrder: number;
  isPopular: boolean;
  isPublic: boolean;
  stripePriceIdMonthly?: string;
  stripePriceIdAnnual?: string;
  entitlements?: TierEntitlement[];
}

interface Feature {
  id: string;
  name: string;
  displayName: string;
  description?: string;
  category?: string;
  featureType: 'boolean' | 'limit' | 'metered';
  unit?: string;
  isPublic: boolean;
}

interface TierEntitlement {
  tierId: string;
  featureId: string;
  feature?: Feature;
  enabled: boolean;
  limitValue?: number;
}

interface EntitlementCheck {
  hasAccess: boolean;
  limitValue?: number;
  currentUsage?: number;
  source: 'tier' | 'override' | 'addon';
  upgradeInfo?: {
    requiredTier: string;
    tierName: string;
  };
}
```

---

## 3. API Contracts

### 3.1 Product Endpoints

#### List Products (Public)
```yaml
GET /v1/products

Query Parameters:
  status: string (filter: active, coming_soon)
  includePrivate: boolean (admin only)

Response: 200 OK
  {
    "success": true,
    "data": [
      {
        "id": "uuid",
        "name": "FlowMind Analytics",
        "slug": "analytics",
        "shortDescription": "Product analytics platform",
        "iconUrl": "https://...",
        "status": "active"
      }
    ]
  }
```

#### Get Product with Tiers
```yaml
GET /v1/products/{slug}

Query Parameters:
  includeTiers: boolean (default: true)
  includeFeatures: boolean (default: true)

Response: 200 OK
  {
    "success": true,
    "data": {
      "id": "uuid",
      "name": "FlowMind Analytics",
      "slug": "analytics",
      "description": "...",
      "tiers": [
        {
          "id": "uuid",
          "name": "Free",
          "slug": "free",
          "priceMonthly": 0,
          "priceAnnual": 0,
          "features": [
            { "name": "users", "displayName": "Users", "type": "limit", "value": 3 },
            { "name": "dashboards", "displayName": "Dashboards", "type": "limit", "value": 5 },
            { "name": "data_export", "displayName": "Data Export", "type": "boolean", "value": false }
          ]
        },
        {
          "id": "uuid",
          "name": "Pro",
          "slug": "pro",
          "priceMonthly": 49,
          "priceAnnual": 490,
          "isPopular": true,
          "features": [
            { "name": "users", "value": 25 },
            { "name": "dashboards", "value": -1 },  // Unlimited
            { "name": "data_export", "value": true }
          ]
        }
      ]
    }
  }
```

### 3.2 Entitlement Endpoints

#### Check Feature Access
```yaml
GET /v1/entitlements/check

Query Parameters:
  organizationId: UUID (required)
  feature: string (required)

Response: 200 OK
  {
    "success": true,
    "data": {
      "hasAccess": true,
      "limitValue": 25,
      "currentUsage": 12,
      "remainingQuota": 13,
      "source": "tier"
    }
  }
```

#### Batch Check Features
```yaml
POST /v1/entitlements/check/batch

Request:
  {
    "organizationId": "uuid",
    "features": ["users", "dashboards", "data_export", "api_access"]
  }

Response: 200 OK
  {
    "success": true,
    "data": {
      "users": { "hasAccess": true, "limitValue": 25 },
      "dashboards": { "hasAccess": true, "limitValue": -1 },
      "data_export": { "hasAccess": true },
      "api_access": { "hasAccess": false, "upgradeInfo": { "requiredTier": "enterprise" } }
    }
  }
```

#### Get Organization Entitlements
```yaml
GET /v1/organizations/{orgId}/entitlements

Response: 200 OK
  {
    "success": true,
    "data": {
      "tier": {
        "id": "uuid",
        "name": "Pro",
        "slug": "pro"
      },
      "features": {
        "users": { "enabled": true, "limit": 25, "source": "tier" },
        "custom_feature": { "enabled": true, "limit": 100, "source": "override" }
      },
      "addons": [
        { "id": "uuid", "name": "Extra Storage", "slug": "extra-storage" }
      ]
    }
  }
```

#### Grant Feature Override (Admin)
```yaml
POST /v1/organizations/{orgId}/entitlements/override

Request:
  {
    "featureId": "uuid",
    "enabled": true,
    "limitValue": 100,
    "reason": "Enterprise pilot program",
    "expiresAt": "2026-01-01T00:00:00Z"
  }

Response: 201 Created
```

### 3.3 Tier Comparison

#### Get Tier Comparison Matrix
```yaml
GET /v1/products/{productSlug}/compare

Response: 200 OK
  {
    "success": true,
    "data": {
      "tiers": ["free", "starter", "pro", "enterprise"],
      "categories": [
        {
          "name": "Core Features",
          "features": [
            {
              "name": "users",
              "displayName": "Team Members",
              "values": {
                "free": { "type": "limit", "value": 3 },
                "starter": { "type": "limit", "value": 10 },
                "pro": { "type": "limit", "value": 50 },
                "enterprise": { "type": "unlimited" }
              }
            }
          ]
        }
      ]
    }
  }
```

### 3.4 gRPC Service (High Performance)

```protobuf
syntax = "proto3";

package flowmind.product.v1;

service ProductService {
  rpc CheckEntitlement(CheckEntitlementRequest) returns (CheckEntitlementResponse);
  rpc CheckEntitlementBatch(CheckEntitlementBatchRequest) returns (CheckEntitlementBatchResponse);
  rpc GetOrganizationTier(GetOrganizationTierRequest) returns (GetOrganizationTierResponse);
}

message CheckEntitlementRequest {
  string organization_id = 1;
  string feature_name = 2;
}

message CheckEntitlementResponse {
  bool has_access = 1;
  int32 limit_value = 2;  // -1 for unlimited
  string source = 3;      // tier, override, addon
}
```

---

## 4. Event Contracts

```typescript
interface TierCreatedEvent {
  eventType: 'tier.created';
  productId: string;
  tierId: string;
  name: string;
  slug: string;
}

interface EntitlementOverrideGrantedEvent {
  eventType: 'entitlement.override_granted';
  organizationId: string;
  featureId: string;
  featureName: string;
  grantedBy: string;
  expiresAt?: string;
}

interface EntitlementOverrideExpiredEvent {
  eventType: 'entitlement.override_expired';
  organizationId: string;
  featureId: string;
}
```

---

## 5. Caching Strategy

```
product:{slug}                      - Product with tiers (TTL: 10min)
tier:{id}:entitlements              - Tier entitlements (TTL: 10min)
org:{id}:entitlements               - Organization entitlements (TTL: 5min)
feature:check:{orgId}:{feature}     - Entitlement check result (TTL: 1min)
```

---

*FlowMind Technologies | Product & Feature Service Technical Specification v1.0*

