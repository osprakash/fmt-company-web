# Account Service - Requirements Document

> **Service**: Account Service  
> **Version**: 1.0  
> **Owner**: Platform Team  
> **Last Updated**: November 2025

---

## 1. Business Context

### 1.1 Purpose
The Account Service manages the organizational hierarchy for FlowMind Technologies' multi-tenant platform. It provides the foundational structure for billing, user organization, and resource allocation across all products.

### 1.2 Business Goals
- Enable B2B customers to organize their company structure
- Support complex billing hierarchies for enterprise customers
- Provide foundation for multi-product subscription management
- Enable division-level reporting and resource allocation

### 1.3 Scope
| In Scope | Out of Scope |
|----------|--------------|
| Organization management | User authentication |
| Division hierarchy | Permission management |
| Account lifecycle | Payment processing |
| Billing entity management | Subscription logic |
| Organization metadata | Product entitlements |

---

## 2. Functional Requirements

### 2.1 Organization Management

#### FR-ACC-001: Create Organization
- System shall allow creation of new organizations
- Organization must have a unique name and slug
- System shall generate a unique organization ID
- Organization shall be created in "trial" status by default

#### FR-ACC-002: Organization Profile
- System shall store organization profile information:
  - Legal name
  - Display name
  - Slug (URL-friendly identifier)
  - Logo URL
  - Website
  - Industry
  - Company size
  - Tax ID / VAT number
  - Primary contact email

#### FR-ACC-003: Organization Status Lifecycle
- System shall support the following organization statuses:
  - `trial` - Initial state, limited features
  - `active` - Paid/activated organization
  - `suspended` - Temporarily disabled (payment issues)
  - `churned` - Cancelled subscription
  - `deleted` - Soft-deleted, data retained per policy

#### FR-ACC-004: Organization Settings
- System shall allow organizations to configure:
  - Timezone
  - Date format
  - Currency preference
  - Language preference

### 2.2 Division Management

#### FR-ACC-010: Create Division
- System shall support hierarchical divisions within an organization
- Divisions can have parent divisions (unlimited depth)
- Division must belong to exactly one organization

#### FR-ACC-011: Division Hierarchy
- System shall support:
  - Root divisions (no parent)
  - Child divisions (with parent reference)
  - Moving divisions between parents
  - Division depth limit configurable (default: 10 levels)

#### FR-ACC-012: Division Billing Rollup
- System shall support billing aggregation at division level
- Child division costs shall roll up to parent divisions
- System shall support cost center assignment per division

#### FR-ACC-013: Division Metadata
- System shall store for each division:
  - Name
  - Code (internal reference)
  - Description
  - Cost center
  - Custom metadata (JSON)

### 2.3 Account Management

#### FR-ACC-020: Billing Account
- System shall support multiple billing accounts per organization
- Account represents a billable entity
- Account can span multiple divisions

#### FR-ACC-021: Account Information
- System shall store for each account:
  - Account name
  - Account type (standard, enterprise, reseller)
  - Billing address
  - Billing email
  - Tax exemption status
  - Payment terms (net 30, prepaid, etc.)
  - Currency

#### FR-ACC-022: Account Status
- System shall support account statuses:
  - `active` - Normal operating state
  - `past_due` - Payment overdue
  - `suspended` - Access restricted
  - `closed` - Account terminated

#### FR-ACC-023: Account-Division Association
- System shall allow linking accounts to divisions
- An account can be linked to multiple divisions
- A division can be linked to one account

### 2.4 Search and Query

#### FR-ACC-030: Organization Search
- System shall support searching organizations by:
  - Name (partial match)
  - Slug (exact match)
  - Status
  - Industry
  - Created date range

#### FR-ACC-031: Division Tree Retrieval
- System shall provide full division tree for an organization
- System shall support paginated flat list of divisions
- System shall support filtered division queries

#### FR-ACC-032: Audit Trail
- System shall maintain audit trail for all changes
- Audit shall include: who, what, when, before/after values

---

## 3. Non-Functional Requirements

### 3.1 Performance
| Metric | Target |
|--------|--------|
| API Response Time (p95) | < 100ms |
| API Response Time (p99) | < 250ms |
| Throughput | 1000 requests/second |
| Database Query Time | < 50ms |

### 3.2 Scalability
- Support up to 100,000 organizations
- Support up to 1,000 divisions per organization
- Support up to 100 accounts per organization

### 3.3 Availability
- 99.9% uptime SLA
- Recovery Time Objective (RTO): 1 hour
- Recovery Point Objective (RPO): 5 minutes

### 3.4 Security
- All data encrypted at rest
- PII fields encrypted with tenant-specific keys
- API authentication required for all endpoints
- Tenant isolation enforced at database level

### 3.5 Compliance
- GDPR compliant (data export, deletion)
- SOC2 audit trail requirements
- Data retention configurable per tenant

---

## 4. User Stories

### Epic: Organization Management

#### US-ACC-001: Create Organization
**As a** new customer  
**I want to** create an organization during signup  
**So that** I can start using FlowMind products  

**Acceptance Criteria:**
- [ ] Organization is created with provided name
- [ ] Unique slug is generated or accepted
- [ ] Organization starts in "trial" status
- [ ] Audit event is recorded
- [ ] Welcome notification is triggered

#### US-ACC-002: Update Organization Profile
**As an** organization admin  
**I want to** update my organization's profile  
**So that** our information is accurate for billing and communication  

**Acceptance Criteria:**
- [ ] Can update all editable profile fields
- [ ] Logo can be uploaded/changed
- [ ] Changes are audit logged
- [ ] Validation errors shown for invalid input

#### US-ACC-003: View Organization Dashboard
**As an** organization admin  
**I want to** see an overview of my organization  
**So that** I can understand our usage and structure  

**Acceptance Criteria:**
- [ ] Shows organization details
- [ ] Shows division count and hierarchy summary
- [ ] Shows account summary
- [ ] Shows subscription status

### Epic: Division Management

#### US-ACC-010: Create Division Structure
**As an** organization admin  
**I want to** create divisions and sub-divisions  
**So that** I can mirror our company structure  

**Acceptance Criteria:**
- [ ] Can create root-level divisions
- [ ] Can create child divisions under any division
- [ ] Division names must be unique within parent
- [ ] Maximum depth is enforced

#### US-ACC-011: Move Division
**As an** organization admin  
**I want to** move a division to a different parent  
**So that** I can reorganize as our company changes  

**Acceptance Criteria:**
- [ ] Can move division to new parent
- [ ] All child divisions move with parent
- [ ] Cannot create circular references
- [ ] Billing rollup recalculates

#### US-ACC-012: Assign Cost Centers
**As a** finance admin  
**I want to** assign cost centers to divisions  
**So that** we can track spending by department  

**Acceptance Criteria:**
- [ ] Can assign cost center code to division
- [ ] Cost center is inherited by children if not overridden
- [ ] Cost center appears in billing reports

### Epic: Account Management

#### US-ACC-020: Create Billing Account
**As a** finance admin  
**I want to** create separate billing accounts  
**So that** different parts of our organization can be billed separately  

**Acceptance Criteria:**
- [ ] Can create new billing account
- [ ] Can specify billing address and contact
- [ ] Can link account to divisions
- [ ] Account appears in billing system

#### US-ACC-021: Manage Payment Terms
**As a** finance admin  
**I want to** set payment terms for our account  
**So that** invoicing matches our procurement process  

**Acceptance Criteria:**
- [ ] Can request payment terms change
- [ ] Approved terms apply to future invoices
- [ ] Current terms shown on account

---

## 5. Dependencies

### 5.1 Upstream Dependencies
| Service | Dependency Type | Description |
|---------|----------------|-------------|
| Identity Service | Authentication | User authentication for API calls |
| API Gateway | Routing | Request routing and rate limiting |

### 5.2 Downstream Dependencies
| Service | Dependency Type | Description |
|---------|----------------|-------------|
| IAM Service | Data | User-Organization associations |
| Subscription Service | Data | Billing account for subscriptions |
| All Services | Events | Organization/Account events |

### 5.3 External Dependencies
| System | Purpose |
|--------|---------|
| PostgreSQL | Primary data store |
| Redis | Caching layer |
| Message Broker | Event publishing |

---

## 6. Acceptance Criteria Summary

### 6.1 MVP Criteria
- [ ] CRUD operations for Organizations
- [ ] Basic division hierarchy (3 levels)
- [ ] Single account per organization
- [ ] Organization status management
- [ ] Basic search functionality

### 6.2 V1.0 Criteria
- [ ] Full division hierarchy support
- [ ] Multiple accounts per organization
- [ ] Cost center management
- [ ] Advanced search and filtering
- [ ] Full audit trail
- [ ] Event publishing for all state changes

### 6.3 Future Considerations
- Organization merge/split functionality
- Cross-organization collaboration
- Organization templates
- Bulk import/export

---

*FlowMind Technologies | Account Service Requirements v1.0*

