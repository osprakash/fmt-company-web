# FlowMind Technologies - Platform Engineering Plan

> **Company**: FlowMind Technologies  
> **Author Context**: Prakash Subramani

> **Goal**: Multi-tenant SaaS platform supporting B2B, B2C, and hybrid products  
> **Created**: November 2025

---

## 📋 Executive Summary

This document outlines the core platform services, architecture decisions, and technical roadmap for **FlowMind Technologies** - a scalable multi-product SaaS company. The plan prioritizes **build once, reuse everywhere** philosophy while maintaining flexibility for product-specific requirements.

---

## 🏗️ Core Services Architecture

### Your Proposed Services (Reviewed & Enhanced)

| # | Service | Status | Notes |
|---|---------|--------|-------|
| 1 | Account Service | ✅ Good | Org → Division → Account hierarchy is solid |
| 2 | Identity Service (Keycloak) | ✅ Good | Consider Auth0/Cognito as alternatives |
| 3 | IAM Service | ✅ Good | Add RBAC + ABAC support |
| 4 | Product Service | ✅ Good | Add feature catalog integration |
| 5 | Subscriptions & Payment | ✅ Good | Integrate with Stripe/Paddle |
| 6 | Customer Support | ✅ Good | Consider ticketing + knowledge base |
| 7 | Settings Service | ✅ Good | Separate tenant vs user preferences |

### 🚨 Critical Missing Services

| # | Service | Priority | Rationale |
|---|---------|----------|-----------|
| 8 | **Notification Service** | P0 | Email, SMS, Push, In-app - every SaaS needs this |
| 9 | **Audit & Logging Service** | P0 | Compliance (SOC2, GDPR), debugging, security |
| 10 | **API Gateway** | P0 | Rate limiting, auth, routing, versioning |
| 11 | **Tenant Management Service** | P0 | Multi-tenancy config, data isolation policies |
| 12 | **Feature Flags Service** | P1 | Gradual rollouts, A/B testing, kill switches |
| 13 | **Analytics & Telemetry** | P1 | Product usage, funnel analysis, churn prediction |
| 14 | **Metering & Usage Service** | P1 | Usage-based billing, quota management |
| 15 | **File/Asset Service** | P1 | Document storage, media, CDN integration |
| 16 | **Webhook & Integration Service** | P2 | Third-party integrations, event delivery |
| 17 | **Search Service** | P2 | Full-text search, filters (Elasticsearch/Meilisearch) |
| 18 | **Reporting & BI Service** | P2 | Dashboards, exports, scheduled reports |

---

## 🏛️ Recommended Service Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              PRESENTATION LAYER                              │
├─────────────────────────────────────────────────────────────────────────────┤
│  flowmind.io      │  Product Landing Pages  │  Product Apps (per product)   │
│  (Company Site)   │  (per product)          │  (React/Vue/Angular)          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API GATEWAY LAYER                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  Kong / AWS API Gateway / Traefik                                           │
│  • Authentication    • Rate Limiting    • Request Routing                   │
│  • API Versioning    • Request/Response Transform                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          ▼                           ▼                           ▼
┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐
│   CORE PLATFORM     │   │   BUSINESS SUPPORT  │   │   PRODUCT DOMAIN    │
│      SERVICES       │   │      SERVICES       │   │      SERVICES       │
├─────────────────────┤   ├─────────────────────┤   ├─────────────────────┤
│ • Account Service   │   │ • Subscription Svc  │   │ • Product A Svc     │
│ • Identity Service  │   │ • Payment Service   │   │ • Product B Svc     │
│ • IAM Service       │   │ • Customer Support  │   │ • Product C Svc     │
│ • Tenant Service    │   │ • Notification Svc  │   │   (dedicated DB     │
│ • Settings Service  │   │ • Analytics Service │   │    per product)     │
│ • Audit Service     │   │ • Metering Service  │   │                     │
│ • Feature Flags     │   │ • Reporting Service │   │                     │
└─────────────────────┘   └─────────────────────┘   └─────────────────────┘
          │                           │                           │
          └───────────────────────────┼───────────────────────────┘
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DATA & MESSAGING LAYER                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  Message Broker (Kafka/RabbitMQ)  │  Cache (Redis)  │  Search (Elastic)    │
│  Shared DB (PostgreSQL)           │  Blob Storage (S3)                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 📦 Detailed Service Specifications

### 1. Account Service
```yaml
Purpose: Organization hierarchy & account management
Entities:
  - Organization (company/business entity)
  - Division (supports hierarchy for billing rollup)
  - Account (billing account, can span divisions)
Features:
  - Hierarchy management (parent-child relationships)
  - Account status lifecycle (trial → active → suspended → churned)
  - Billing address & tax information
  - Account metadata & custom fields
Multi-tenancy: Shared DB with tenant_id isolation
```

### 2. Identity Service (Keycloak)
```yaml
Purpose: Authentication & identity management
Features:
  - SSO (SAML, OIDC, OAuth2)
  - Social login (Google, Microsoft, GitHub)
  - MFA/2FA support
  - Password policies
  - Session management
  - B2B: Federated identity (customer's IdP)
Enterprise: Self-hosted Keycloak per enterprise tenant
```

### 3. IAM Service
```yaml
Purpose: Authorization & access control
Features:
  - User invitation workflow
  - Role management (system roles + custom roles)
  - Permission sets (RBAC)
  - Attribute-based access (ABAC) for enterprise
  - API key management
  - Service accounts for integrations
Model: Organization → Users → Roles → Permissions → Resources
```

### 4. Tenant Management Service ⭐ NEW
```yaml
Purpose: Multi-tenancy configuration & isolation
Features:
  - Tenant onboarding workflow
  - Data isolation strategy per tenant (shared/silo)
  - Custom domain mapping
  - Tenant-specific configurations
  - Resource quotas & limits
  - Tenant health monitoring
Isolation Levels:
  - Standard: Shared infrastructure, logical isolation
  - Premium: Dedicated compute, shared DB
  - Enterprise: Dedicated everything
```

### 5. Product & Feature Service
```yaml
Purpose: Product catalog & feature management
Entities:
  - Products (your SaaS offerings)
  - Tiers (Free, Pro, Enterprise)
  - Features (granular capabilities)
  - Feature-Tier mapping
Features:
  - Product versioning
  - Feature entitlements
  - Usage limits per tier
  - Add-on management
```

### 6. Subscription & Billing Service
```yaml
Purpose: Subscription lifecycle & payment processing
Features:
  - Plan management
  - Subscription CRUD
  - Proration handling
  - Dunning management
  - Invoice generation
  - Revenue recognition
Integrations: Stripe, Paddle, Chargebee
```

### 7. Metering & Usage Service ⭐ NEW
```yaml
Purpose: Track usage for billing & quotas
Features:
  - Real-time usage tracking
  - Usage aggregation (hourly/daily/monthly)
  - Quota enforcement
  - Overage alerts
  - Usage-based billing triggers
Use Cases:
  - API calls
  - Storage consumed
  - Active users
  - Compute minutes
```

### 8. Notification Service ⭐ NEW
```yaml
Purpose: Multi-channel communication
Channels:
  - Email (transactional & marketing)
  - SMS
  - Push notifications
  - In-app notifications
  - Slack/Teams webhooks
Features:
  - Template management
  - Preference center (user opt-in/out)
  - Delivery tracking
  - Scheduled notifications
  - Batch processing
Providers: SendGrid, Twilio, OneSignal
```

### 9. Audit & Logging Service ⭐ NEW
```yaml
Purpose: Compliance, security & debugging
Features:
  - User activity logs
  - Admin action logs
  - Data access logs
  - Security events
  - Change history (who, what, when)
  - Log retention policies
Compliance: SOC2, GDPR, HIPAA ready
Storage: Time-series DB or Elasticsearch
```

### 10. Feature Flags Service ⭐ NEW
```yaml
Purpose: Controlled feature rollouts
Features:
  - Boolean flags
  - Percentage rollouts
  - User segment targeting
  - A/B testing integration
  - Kill switches
  - Environment-specific flags
Options: Build vs Buy (LaunchDarkly, Flagsmith, Unleash)
```

### 11. Analytics & Telemetry Service ⭐ NEW
```yaml
Purpose: Product analytics & business intelligence
Features:
  - Event tracking (frontend + backend)
  - User journey analysis
  - Funnel analytics
  - Cohort analysis
  - Custom dashboards
  - Data export
Stack: Segment + Amplitude/Mixpanel or self-hosted (PostHog)
```

### 12. API Gateway ⭐ NEW
```yaml
Purpose: Unified API management
Features:
  - Request routing
  - Authentication/Authorization
  - Rate limiting (per tenant/user/API key)
  - Request/Response transformation
  - API versioning
  - Circuit breaker
  - Request logging
Options: Kong, AWS API Gateway, Traefik, Nginx
```

### 13. File & Asset Service ⭐ NEW
```yaml
Purpose: File storage & management
Features:
  - Secure upload/download
  - Virus scanning
  - Image processing (resize, thumbnails)
  - CDN integration
  - Signed URLs
  - Storage quotas
Storage: S3, Azure Blob, GCS
```

### 14. Customer Support Service
```yaml
Purpose: Support ticket management
Features:
  - Ticket lifecycle (create → assign → resolve)
  - SLA management
  - Knowledge base
  - Chat widget integration
  - Customer satisfaction surveys
  - Escalation workflows
Options: Build basic + integrate (Zendesk, Intercom, Freshdesk)
```

### 15. Settings Service
```yaml
Purpose: Configuration management
Scopes:
  - System settings (global)
  - Tenant settings (per organization)
  - User preferences (per user)
Features:
  - Setting inheritance (system → tenant → user)
  - Setting overrides
  - Feature toggles
  - UI customization (themes, branding)
```

---

## 🔄 Multi-Tenancy Strategy

### Data Isolation Models

| Model | Use Case | Pros | Cons |
|-------|----------|------|------|
| **Shared DB, Shared Schema** | SMB customers | Cost efficient | Complex queries, noisy neighbor |
| **Shared DB, Separate Schema** | Mid-market | Good isolation | Schema management overhead |
| **Dedicated Database** | Enterprise | Full isolation | Higher cost, ops complexity |

### Recommended Approach
```
Default: Shared DB with tenant_id (row-level isolation)
├── Free/Starter tiers: Shared everything
├── Pro tier: Shared DB, dedicated compute option
└── Enterprise tier: Dedicated DB, optional dedicated infra
```

### Tenant Resolution Strategy
```
1. Subdomain: tenant1.productname.flowmind.io
2. Custom Domain: app.customer.com (CNAME mapping)
3. Path-based: productname.flowmind.io/tenant1 (less common)
4. Header-based: X-Tenant-ID (for APIs)
```

---

## 🚀 Technical Getting Started Guide

### Phase 1: Foundation (Weeks 1-4)

#### Week 1-2: Infrastructure Setup
```bash
# 1. Repository Structure (Monorepo recommended for startups)
/flowmind-platform
  /services
    /account-service
    /identity-service
    /iam-service
    /notification-service
    ...
  /libs
    /common          # Shared utilities
    /contracts       # API contracts/types
    /db-schemas      # Shared DB schemas
  /infra
    /terraform       # IaC
    /k8s            # Kubernetes manifests
    /docker         # Dockerfiles
  /apps
    /flowmind-website    # flowmind.io
    /product-landing     # Per-product landing pages
    /admin-portal        # Internal admin

# 2. Tech Stack (Recommended)
Backend: Node.js (NestJS) or Go or .NET
Frontend: Next.js or React + Vite
Database: PostgreSQL (primary) + Redis (cache)
Message Broker: RabbitMQ (start) → Kafka (scale)
Search: Meilisearch (start) → Elasticsearch (scale)
```

#### Week 3-4: Core Service Bootstrap
```
Priority Order:
1. API Gateway setup
2. Identity Service (Keycloak setup)
3. Account Service (basic CRUD)
4. IAM Service (users, roles)
5. Tenant Management (basic)
```

### Phase 2: Essential Services (Weeks 5-12)

```
Week 5-6:  Notification Service (email first)
Week 7-8:  Product & Subscription Service
Week 9-10: Feature Flags + Settings Service
Week 11-12: Audit Service + Analytics foundation
```

### Phase 3: Product Development (Weeks 13+)

```
- Build first product domain service
- Build product UI app
- Integration testing
- Beta launch
```

---

## 📐 Key Architecture Decisions

### 1. Communication Patterns
```
Synchronous (REST/gRPC):
  - User-facing APIs
  - Real-time operations
  - Simple CRUD

Asynchronous (Events):
  - Cross-service communication
  - Audit logging
  - Notifications
  - Analytics events
```

### 2. Event-Driven Architecture
```yaml
Event Bus: RabbitMQ / Kafka
Event Types:
  - Domain Events: user.created, subscription.activated
  - Integration Events: payment.received, email.sent
  - System Events: service.health, cache.invalidated

Pattern: Outbox Pattern for reliable event publishing
```

### 3. Database Strategy
```
Core Services: Dedicated DB per service (loose coupling)
Product Services: Dedicated DB per product
Shared Data: Event sourcing for cross-service data
```

### 4. API Design
```yaml
Style: REST (primary) + GraphQL (for complex UIs)
Versioning: URL-based (/v1/, /v2/)
Documentation: OpenAPI 3.0
SDKs: Auto-generate from OpenAPI specs
```

---

## 🔐 Security Considerations

### Authentication & Authorization Flow
```
1. User authenticates via Identity Service (Keycloak)
2. Receives JWT with tenant_id, user_id, roles
3. API Gateway validates JWT
4. IAM Service checks permissions (cached)
5. Service performs action
6. Audit Service logs activity
```

### Security Checklist
- [ ] JWT token validation at gateway
- [ ] Tenant isolation in every query
- [ ] Rate limiting per tenant/user
- [ ] API key rotation mechanism
- [ ] Secrets management (Vault/AWS Secrets)
- [ ] Data encryption at rest & transit
- [ ] GDPR data handling (delete, export)
- [ ] SOC2 audit trail

---

## 💰 Build vs Buy Recommendations

| Service | Recommendation | Reasoning |
|---------|---------------|-----------|
| Identity | **Buy** (Keycloak/Auth0) | Complex, security-critical |
| Payment | **Buy** (Stripe) | Compliance heavy, not core |
| Email | **Buy** (SendGrid) | Deliverability expertise |
| Feature Flags | **Buy** (Flagsmith OSS) | Time saver, not core |
| Analytics | **Buy** (PostHog OSS) | Complex, not core value |
| Customer Support | **Hybrid** | Build basic, integrate advanced |
| Account/IAM | **Build** | Core to your business |
| Tenant Management | **Build** | Core differentiator |
| Product/Subscription | **Build** | Business logic heavy |

---

## 📊 Success Metrics to Track

### Platform Health
- API latency (p50, p95, p99)
- Error rates per service
- Uptime per tenant tier

### Business Metrics
- MRR/ARR
- Customer acquisition cost (CAC)
- Lifetime value (LTV)
- Churn rate
- Net revenue retention

### Product Metrics
- Daily/Monthly active users
- Feature adoption rates
- Time to value (onboarding)

---

## ⚠️ Startup Reality Check

### Do This ✅
1. **Start with monolith** - Extract services as you scale
2. **Focus on 1 product first** - Validate before building platform
3. **Use managed services** - RDS, managed Kubernetes
4. **Automate from day 1** - CI/CD, IaC
5. **Build billing early** - Revenue validates faster

### Avoid This ❌
1. Over-engineering before product-market fit
2. Building all services upfront
3. Custom solutions for commodity problems
4. Premature microservices
5. Complex multi-region before you need it

### Pragmatic Approach
```
MVP Launch Stack:
├── Monolith with modular architecture
├── Keycloak for identity
├── Stripe for payments
├── SendGrid for email
├── PostgreSQL + Redis
├── Single cloud region
└── Simple CI/CD (GitHub Actions)

Scale Later:
├── Extract services as needed
├── Add message broker
├── Multi-region
└── Advanced observability
```

---

## 🗓️ 6-Month Roadmap

```
Month 1: Foundation
├── Monorepo setup
├── CI/CD pipeline
├── Keycloak deployment
├── Core DB schema
└── API Gateway

Month 2: Core Platform
├── Account Service
├── IAM Service
├── Basic Tenant Management
└── Settings Service

Month 3: Business Services
├── Product/Tier Service
├── Subscription Service (Stripe)
├── Notification Service (Email)
└── Basic Admin Portal

Month 4: First Product
├── Product domain service
├── Product UI
├── Landing page
└── Integration testing

Month 5: Polish & Launch Prep
├── Audit logging
├── Analytics integration
├── Documentation
├── Security audit
└── Performance testing

Month 6: Beta Launch
├── Soft launch
├── Customer onboarding
├── Feedback iteration
├── Support processes
└── Monitoring & alerting
```

---

## 🎯 Final Assessment

### Is This a Good Plan? **YES, with caveats**

**Strengths:**
- Solid understanding of core SaaS primitives
- Multi-tenant thinking from day one
- Separation of concerns is correct

**Improvements Needed:**
- Add the missing services (especially Notification, Audit, API Gateway)
- Consider event-driven architecture early
- Plan for observability from start
- Don't over-build before PMF

**Key Success Factors:**
1. Ship first product fast (3-4 months)
2. Build platform incrementally based on real needs
3. Validate with paying customers early
4. Hire/contract DevOps early for platform reliability

---

## 📚 Recommended Resources

- **Book**: "Building Multi-Tenant SaaS Architectures" - Tod Golding
- **Book**: "Designing Data-Intensive Applications" - Martin Kleppmann
- **Course**: AWS SaaS Factory resources
- **Tool**: Draw.io for architecture diagrams
- **Community**: r/SaaS, IndieHackers, SaaStr

---

*Document Version: 1.0 | FlowMind Technologies | Last Updated: November 2025*

