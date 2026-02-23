# IAM Service - Requirements Document

> **Service**: IAM (Identity & Access Management) Service  
> **Version**: 1.0  
> **Owner**: Platform Team  
> **Last Updated**: November 2025

---

## 1. Business Context

### 1.1 Purpose
The IAM Service manages authorization, access control, and user-organization relationships for FlowMind Technologies. It handles user invitations, role assignments, and permission enforcement across all products.

### 1.2 Business Goals
- Provide fine-grained access control for all products
- Support enterprise role management requirements
- Enable self-service user and role management
- Maintain audit trail for compliance

### 1.3 Scope
| In Scope | Out of Scope |
|----------|--------------|
| User-Organization association | User authentication (Identity Service) |
| User invitations | User profile data |
| Role management (RBAC) | Organization structure (Account Service) |
| Permission management | Subscription management |
| API key management | |
| Service accounts | |

---

## 2. Functional Requirements

### 2.1 User Management

#### FR-IAM-001: User-Organization Association
- System shall link users to organizations
- User can belong to multiple organizations
- User has one primary organization (default context)
- System shall track user status per organization (active, suspended, removed)

#### FR-IAM-002: User Invitation
- System shall support email-based user invitations
- Invitation shall include role assignment
- Invitation shall expire after configurable period (default: 7 days)
- System shall support invitation resend
- System shall support bulk invitations (CSV upload)

#### FR-IAM-003: User Status Management
- User status within organization:
  - `pending` - Invited, not yet accepted
  - `active` - Active member
  - `suspended` - Temporarily disabled
  - `removed` - Removed from organization
- Status changes shall be audited

#### FR-IAM-004: User Removal
- Admins shall be able to remove users from organization
- Removal shall revoke all access immediately
- User data shall be retained per policy
- Cannot remove last admin

### 2.2 Role Management (RBAC)

#### FR-IAM-010: System Roles
- System shall provide predefined roles:
  - `owner` - Full access, can delete organization
  - `admin` - Full access except org deletion
  - `member` - Standard access
  - `viewer` - Read-only access
  - `billing` - Billing and subscription access
- System roles cannot be modified

#### FR-IAM-011: Custom Roles
- Organizations shall be able to create custom roles
- Custom roles inherit from a base system role
- Custom roles can have granular permissions
- Custom roles scoped to organization

#### FR-IAM-012: Role Assignment
- Users shall be assigned one or more roles per organization
- Role assignment shall include scope (organization, division, resource)
- Role changes take effect immediately

#### FR-IAM-013: Role Hierarchy
- Roles can inherit from other roles
- Permission inheritance follows role hierarchy
- Explicit denies override inherited allows

### 2.3 Permission Management

#### FR-IAM-020: Permission Model
- Permissions follow format: `resource:action`
- Resources: organizations, users, divisions, subscriptions, etc.
- Actions: create, read, update, delete, manage
- Wildcard support: `resource:*`, `*:read`

#### FR-IAM-021: Permission Scoping
- Permissions can be scoped to:
  - Organization level (org-wide)
  - Division level (specific division and children)
  - Resource level (specific resource instance)

#### FR-IAM-022: Permission Evaluation
- System shall evaluate permissions in order:
  1. Explicit resource-level permissions
  2. Division-level permissions
  3. Organization-level permissions
  4. Role-based permissions
- First match wins; explicit deny always wins

#### FR-IAM-023: ABAC Support (Enterprise)
- Support attribute-based access control for enterprise
- Attributes: user, resource, environment, time
- Policy language for complex rules

### 2.4 API Key Management

#### FR-IAM-030: Create API Key
- Users shall be able to create API keys
- API keys inherit user's permissions or subset
- API keys shall have optional expiration
- API keys shall have optional IP restrictions

#### FR-IAM-031: API Key Scoping
- API keys can be scoped to specific permissions
- API keys can be restricted to specific resources
- API keys can be restricted to specific products

#### FR-IAM-032: API Key Rotation
- System shall support API key rotation
- Grace period for old key after rotation
- System shall track key usage for auditing

### 2.5 Service Accounts

#### FR-IAM-040: Service Account Creation
- Organizations can create service accounts
- Service accounts for machine-to-machine access
- Service accounts have dedicated credentials

#### FR-IAM-041: Service Account Permissions
- Service accounts assigned specific roles
- Permissions should follow least privilege
- Service accounts cannot exceed creator's permissions

---

## 3. Non-Functional Requirements

### 3.1 Performance
| Metric | Target |
|--------|--------|
| Permission Check (p95) | < 5ms |
| Permission Check (p99) | < 20ms |
| User List (p95) | < 100ms |
| Throughput | 10,000 checks/second |

### 3.2 Caching
- Permissions cached per user (TTL: 5 min)
- Cache invalidation on permission change
- Distributed cache for consistency

### 3.3 Security
- Permission changes require authentication
- Self-permission removal prevented
- Cannot grant permissions user doesn't have

---

## 4. User Stories

### Epic: User Invitations

#### US-IAM-001: Invite Team Member
**As an** admin  
**I want to** invite team members to my organization  
**So that** they can access our products  

**Acceptance Criteria:**
- [ ] Enter email address and select role
- [ ] Invitation email sent with link
- [ ] Link leads to signup/login flow
- [ ] User added to organization after accepting

#### US-IAM-002: Bulk Invite
**As an** admin  
**I want to** invite multiple users at once  
**So that** I can onboard my team efficiently  

**Acceptance Criteria:**
- [ ] Upload CSV with emails and roles
- [ ] Validate CSV format and emails
- [ ] Show preview before sending
- [ ] Progress indicator for bulk send
- [ ] Summary of sent invitations

### Epic: Role Management

#### US-IAM-010: Assign Role
**As an** admin  
**I want to** assign roles to users  
**So that** they have appropriate access  

**Acceptance Criteria:**
- [ ] View available roles
- [ ] Select user and assign role
- [ ] Confirm role assignment
- [ ] User sees updated permissions immediately

#### US-IAM-011: Create Custom Role
**As an** admin  
**I want to** create custom roles  
**So that** I can match our team structure  

**Acceptance Criteria:**
- [ ] Create new role with name and description
- [ ] Select base role to inherit from
- [ ] Add/remove specific permissions
- [ ] Save and assign to users

### Epic: API Keys

#### US-IAM-020: Create API Key
**As a** developer  
**I want to** create API keys  
**So that** I can integrate with FlowMind APIs  

**Acceptance Criteria:**
- [ ] Generate new API key
- [ ] Select permissions/scope
- [ ] Set optional expiration
- [ ] Key shown once, must copy
- [ ] Key appears in list after creation

---

## 5. Dependencies

### 5.1 Upstream Dependencies
| Service | Dependency Type | Description |
|---------|----------------|-------------|
| Identity Service | Authentication | Token validation |
| Account Service | Data | Organization/division info |

### 5.2 Downstream Dependencies
| Service | Dependency Type | Description |
|---------|----------------|-------------|
| All Services | Consumer | Permission checks |
| Identity Service | Claims | Role claims in JWT |

---

## 6. Acceptance Criteria Summary

### MVP Criteria
- [ ] User-organization association
- [ ] Basic invitations
- [ ] System roles (owner, admin, member, viewer)
- [ ] Role assignment
- [ ] Permission evaluation

### V1.0 Criteria
- [ ] Custom roles
- [ ] API key management
- [ ] Division-level scoping
- [ ] Service accounts
- [ ] Bulk operations

---

*FlowMind Technologies | IAM Service Requirements v1.0*

