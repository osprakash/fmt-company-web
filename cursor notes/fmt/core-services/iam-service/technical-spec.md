# IAM Service - Technical Specification

> **Service**: IAM Service  
> **Version**: 1.0  
> **Owner**: Platform Team  
> **Last Updated**: November 2025

---

## 1. Architecture Overview

### 1.1 Service Context
```
┌─────────────────────────────────────────────────────────────────────────┐
│                             API Gateway                                  │
│                    (JWT Validation + IAM Check)                         │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
         ┌──────────────────────┼──────────────────────┐
         │                      │                      │
         ▼                      ▼                      ▼
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│  IAM Service    │   │ Other Services  │   │ Other Services  │
│                 │   │ (Permission     │   │                 │
│  • User Mgmt    │◄──│  checks via     │   │                 │
│  • Roles        │   │  gRPC/REST)     │   │                 │
│  • Permissions  │   │                 │   │                 │
│  • API Keys     │   └─────────────────┘   └─────────────────┘
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌────────┐ ┌────────┐
│Postgres│ │ Redis  │
│        │ │(Cache) │
└────────┘ └────────┘
```

### 1.2 Technology Stack
| Component | Technology |
|-----------|------------|
| Runtime | Node.js 20 LTS / Go 1.21 |
| Framework | NestJS / Gin |
| Database | PostgreSQL 15 |
| Cache | Redis 7 |
| IPC | gRPC (for high-frequency permission checks) |

---

## 2. Data Model

### 2.1 Entity Relationship Diagram
```
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│      USER       │       │   ORGANIZATION  │       │      ROLE       │
│   (external)    │       │   (external)    │       │                 │
└────────┬────────┘       └────────┬────────┘       │ id              │
         │                         │                │ org_id (null=   │
         │                         │                │   system role)  │
         │                         │                │ name            │
         └───────────┬─────────────┘                │ type            │
                     │                              │ permissions[]   │
                     ▼                              │ parent_role_id  │
         ┌─────────────────────┐                    └────────┬────────┘
         │   ORG_MEMBERSHIP    │                             │
         │                     │                             │
         │ id                  │                             │
         │ user_id             │─────────────────────────────┘
         │ organization_id     │
         │ status              │       ┌─────────────────────┐
         │ roles[]             │       │   ROLE_ASSIGNMENT   │
         │ invited_by          │       │                     │
         │ invited_at          │◄──────│ id                  │
         │ joined_at           │       │ membership_id       │
         └─────────────────────┘       │ role_id             │
                     │                 │ scope_type          │
                     │                 │ scope_id            │
                     │                 │ granted_by          │
                     │                 │ granted_at          │
                     │                 └─────────────────────┘
                     │
                     ▼
         ┌─────────────────────┐       ┌─────────────────────┐
         │     API_KEY         │       │   SERVICE_ACCOUNT   │
         │                     │       │                     │
         │ id                  │       │ id                  │
         │ membership_id       │       │ organization_id     │
         │ name                │       │ name                │
         │ key_hash            │       │ description         │
         │ prefix              │       │ credentials_hash    │
         │ scopes[]            │       │ roles[]             │
         │ ip_whitelist[]      │       │ last_used_at        │
         │ expires_at          │       └─────────────────────┘
         │ last_used_at        │
         └─────────────────────┘
```

### 2.2 Database Schema

```sql
-- Enums
CREATE TYPE membership_status AS ENUM ('pending', 'active', 'suspended', 'removed');
CREATE TYPE role_type AS ENUM ('system', 'custom');
CREATE TYPE scope_type AS ENUM ('organization', 'division', 'resource');
CREATE TYPE invitation_status AS ENUM ('pending', 'accepted', 'expired', 'cancelled');

-- Roles
CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID,  -- NULL for system roles
    name VARCHAR(100) NOT NULL,
    display_name VARCHAR(255) NOT NULL,
    description TEXT,
    type role_type NOT NULL DEFAULT 'custom',
    permissions JSONB NOT NULL DEFAULT '[]',
    parent_role_id UUID REFERENCES roles(id),
    is_default BOOLEAN NOT NULL DEFAULT FALSE,
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT roles_unique_name UNIQUE (organization_id, name),
    CONSTRAINT roles_system_no_org CHECK (
        (type = 'system' AND organization_id IS NULL) OR
        (type = 'custom' AND organization_id IS NOT NULL)
    )
);

-- System roles (seeded)
INSERT INTO roles (id, name, display_name, type, permissions, is_default) VALUES
('00000000-0000-0000-0000-000000000001', 'owner', 'Owner', 'system', '["*:*"]', false),
('00000000-0000-0000-0000-000000000002', 'admin', 'Administrator', 'system', 
 '["organization:read", "organization:update", "users:*", "roles:*", "divisions:*", "subscriptions:*", "settings:*"]', false),
('00000000-0000-0000-0000-000000000003', 'member', 'Member', 'system',
 '["organization:read", "users:read", "divisions:read"]', true),
('00000000-0000-0000-0000-000000000004', 'viewer', 'Viewer', 'system',
 '["organization:read", "users:read", "divisions:read"]', false),
('00000000-0000-0000-0000-000000000005', 'billing', 'Billing Admin', 'system',
 '["organization:read", "subscriptions:*", "invoices:*"]', false);

CREATE INDEX idx_roles_org ON roles(organization_id) WHERE organization_id IS NOT NULL;

-- Organization Memberships
CREATE TABLE organization_memberships (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL,
    organization_id UUID NOT NULL,
    status membership_status NOT NULL DEFAULT 'pending',
    is_primary BOOLEAN NOT NULL DEFAULT FALSE,
    invited_by UUID,
    invited_at TIMESTAMP WITH TIME ZONE,
    joined_at TIMESTAMP WITH TIME ZONE,
    suspended_at TIMESTAMP WITH TIME ZONE,
    suspended_reason TEXT,
    removed_at TIMESTAMP WITH TIME ZONE,
    removed_reason TEXT,
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT memberships_unique UNIQUE (user_id, organization_id)
);

CREATE INDEX idx_memberships_user ON organization_memberships(user_id);
CREATE INDEX idx_memberships_org ON organization_memberships(organization_id);
CREATE INDEX idx_memberships_status ON organization_memberships(status);

-- Role Assignments
CREATE TABLE role_assignments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    membership_id UUID NOT NULL REFERENCES organization_memberships(id) ON DELETE CASCADE,
    role_id UUID NOT NULL REFERENCES roles(id),
    scope_type scope_type NOT NULL DEFAULT 'organization',
    scope_id UUID,  -- division_id or resource_id if scoped
    granted_by UUID NOT NULL,
    granted_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE,
    metadata JSONB NOT NULL DEFAULT '{}',
    
    CONSTRAINT assignments_unique UNIQUE (membership_id, role_id, scope_type, scope_id)
);

CREATE INDEX idx_assignments_membership ON role_assignments(membership_id);
CREATE INDEX idx_assignments_role ON role_assignments(role_id);
CREATE INDEX idx_assignments_scope ON role_assignments(scope_type, scope_id);

-- Invitations
CREATE TABLE invitations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL,
    email VARCHAR(255) NOT NULL,
    role_ids UUID[] NOT NULL,
    status invitation_status NOT NULL DEFAULT 'pending',
    token VARCHAR(100) NOT NULL UNIQUE,
    invited_by UUID NOT NULL,
    accepted_by UUID,
    message TEXT,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    accepted_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT invitations_pending_unique UNIQUE (organization_id, email) 
        WHERE status = 'pending'
);

CREATE INDEX idx_invitations_token ON invitations(token) WHERE status = 'pending';
CREATE INDEX idx_invitations_org ON invitations(organization_id);
CREATE INDEX idx_invitations_email ON invitations(email);

-- API Keys
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    membership_id UUID NOT NULL REFERENCES organization_memberships(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    key_prefix VARCHAR(10) NOT NULL,  -- First 8 chars for identification
    key_hash VARCHAR(255) NOT NULL,   -- bcrypt hash of full key
    scopes JSONB NOT NULL DEFAULT '[]',  -- Subset of user's permissions
    ip_whitelist INET[],
    expires_at TIMESTAMP WITH TIME ZONE,
    last_used_at TIMESTAMP WITH TIME ZONE,
    last_used_ip INET,
    revoked_at TIMESTAMP WITH TIME ZONE,
    revoked_reason TEXT,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_api_keys_membership ON api_keys(membership_id);
CREATE INDEX idx_api_keys_prefix ON api_keys(key_prefix) WHERE revoked_at IS NULL;

-- Service Accounts
CREATE TABLE service_accounts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    client_id VARCHAR(100) NOT NULL UNIQUE,
    client_secret_hash VARCHAR(255) NOT NULL,
    role_ids UUID[] NOT NULL,
    scopes JSONB NOT NULL DEFAULT '[]',
    ip_whitelist INET[],
    last_used_at TIMESTAMP WITH TIME ZONE,
    disabled_at TIMESTAMP WITH TIME ZONE,
    created_by UUID NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT service_accounts_unique_name UNIQUE (organization_id, name)
);

CREATE INDEX idx_service_accounts_org ON service_accounts(organization_id);
CREATE INDEX idx_service_accounts_client ON service_accounts(client_id);

-- Permissions cache table (for complex permission evaluations)
CREATE TABLE permission_cache (
    user_id UUID NOT NULL,
    organization_id UUID NOT NULL,
    permissions JSONB NOT NULL,
    computed_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    
    PRIMARY KEY (user_id, organization_id)
);
```

### 2.3 TypeScript Interfaces

```typescript
// Membership
interface OrganizationMembership {
  id: string;
  userId: string;
  organizationId: string;
  status: MembershipStatus;
  isPrimary: boolean;
  invitedBy?: string;
  invitedAt?: Date;
  joinedAt?: Date;
  roles: RoleAssignment[];
  metadata: Record<string, unknown>;
}

type MembershipStatus = 'pending' | 'active' | 'suspended' | 'removed';

// Role
interface Role {
  id: string;
  organizationId?: string;
  name: string;
  displayName: string;
  description?: string;
  type: 'system' | 'custom';
  permissions: Permission[];
  parentRoleId?: string;
  isDefault: boolean;
}

// Permission format: "resource:action" or "resource:action:scope"
type Permission = string;

interface RoleAssignment {
  id: string;
  membershipId: string;
  roleId: string;
  role?: Role;
  scopeType: 'organization' | 'division' | 'resource';
  scopeId?: string;
  grantedBy: string;
  grantedAt: Date;
  expiresAt?: Date;
}

// Invitation
interface Invitation {
  id: string;
  organizationId: string;
  email: string;
  roleIds: string[];
  status: InvitationStatus;
  token: string;
  invitedBy: string;
  message?: string;
  expiresAt: Date;
  acceptedAt?: Date;
}

type InvitationStatus = 'pending' | 'accepted' | 'expired' | 'cancelled';

// API Key
interface ApiKey {
  id: string;
  membershipId: string;
  name: string;
  description?: string;
  keyPrefix: string;
  scopes: Permission[];
  ipWhitelist?: string[];
  expiresAt?: Date;
  lastUsedAt?: Date;
  createdAt: Date;
}

// Service Account
interface ServiceAccount {
  id: string;
  organizationId: string;
  name: string;
  description?: string;
  clientId: string;
  roleIds: string[];
  scopes: Permission[];
  ipWhitelist?: string[];
  lastUsedAt?: Date;
  disabledAt?: Date;
}

// Permission Check Request/Response
interface PermissionCheckRequest {
  userId: string;
  organizationId: string;
  permission: string;
  resourceId?: string;
  resourceType?: string;
}

interface PermissionCheckResponse {
  allowed: boolean;
  reason?: string;
  matchedRole?: string;
  matchedScope?: string;
}
```

---

## 3. API Contracts

### 3.1 Base URL
```
REST: https://api.flowmind.io/v1/iam
gRPC: iam.flowmind.io:443
```

### 3.2 Membership Endpoints

#### List Organization Members
```yaml
GET /organizations/{orgId}/members

Query Parameters:
  page: integer (default: 1)
  limit: integer (default: 20)
  status: string (filter by status)
  search: string (search by email/name)
  role: string (filter by role)

Response: 200 OK
  {
    "success": true,
    "data": [
      {
        "id": "uuid",
        "userId": "uuid",
        "user": {
          "id": "uuid",
          "email": "john@example.com",
          "name": "John Doe",
          "avatarUrl": "https://..."
        },
        "status": "active",
        "isPrimary": false,
        "roles": [
          {
            "id": "uuid",
            "roleId": "uuid",
            "role": {
              "name": "admin",
              "displayName": "Administrator"
            },
            "scopeType": "organization"
          }
        ],
        "joinedAt": "2025-11-29T10:00:00Z"
      }
    ],
    "meta": { "page": 1, "limit": 20, "total": 45 }
  }
```

#### Get Member Details
```yaml
GET /organizations/{orgId}/members/{memberId}

Response: 200 OK
  {
    "success": true,
    "data": {
      "id": "uuid",
      "userId": "uuid",
      "user": { ... },
      "status": "active",
      "roles": [ ... ],
      "invitedBy": { "id": "uuid", "name": "Jane Doe" },
      "invitedAt": "2025-11-01T10:00:00Z",
      "joinedAt": "2025-11-02T10:00:00Z"
    }
  }
```

#### Update Member Status
```yaml
PATCH /organizations/{orgId}/members/{memberId}

Request:
  {
    "status": "suspended",
    "reason": "Policy violation"
  }

Response: 200 OK
```

#### Remove Member
```yaml
DELETE /organizations/{orgId}/members/{memberId}

Request:
  {
    "reason": "Left the company"
  }

Response: 204 No Content

Errors:
  400: Cannot remove last owner
  404: Member not found
```

### 3.3 Invitation Endpoints

#### Create Invitation
```yaml
POST /organizations/{orgId}/invitations

Request:
  {
    "email": "newuser@example.com",
    "roleIds": ["uuid-admin-role"],
    "message": "Welcome to the team!",
    "expiresInDays": 7
  }

Response: 201 Created
  {
    "success": true,
    "data": {
      "id": "uuid",
      "email": "newuser@example.com",
      "status": "pending",
      "expiresAt": "2025-12-06T10:00:00Z"
    }
  }

Errors:
  400: Invalid email / User already member
  409: Pending invitation exists
```

#### Bulk Invite
```yaml
POST /organizations/{orgId}/invitations/bulk

Request:
  Content-Type: multipart/form-data
  Body:
    file: CSV file (email, role)
    defaultRoleIds: UUID[] (if role not in CSV)

Response: 202 Accepted
  {
    "success": true,
    "data": {
      "jobId": "uuid",
      "total": 50,
      "statusUrl": "/organizations/{orgId}/invitations/bulk/{jobId}"
    }
  }
```

#### Accept Invitation
```yaml
POST /invitations/{token}/accept

Headers:
  Authorization: Bearer <user_token>

Response: 200 OK
  {
    "success": true,
    "data": {
      "organizationId": "uuid",
      "organizationName": "Acme Corp",
      "membershipId": "uuid"
    }
  }

Errors:
  400: Invitation expired
  404: Invalid token
  409: Already a member
```

#### List Invitations
```yaml
GET /organizations/{orgId}/invitations

Query Parameters:
  status: string (pending, accepted, expired)

Response: 200 OK
```

#### Resend Invitation
```yaml
POST /organizations/{orgId}/invitations/{invitationId}/resend

Response: 200 OK
```

#### Cancel Invitation
```yaml
DELETE /organizations/{orgId}/invitations/{invitationId}

Response: 204 No Content
```

### 3.4 Role Endpoints

#### List Roles
```yaml
GET /organizations/{orgId}/roles

Query Parameters:
  includeSystem: boolean (default: true)

Response: 200 OK
  {
    "success": true,
    "data": [
      {
        "id": "uuid",
        "name": "owner",
        "displayName": "Owner",
        "type": "system",
        "description": "Full access to organization",
        "permissions": ["*:*"],
        "memberCount": 2
      },
      {
        "id": "uuid",
        "name": "developer",
        "displayName": "Developer",
        "type": "custom",
        "description": "Access to development resources",
        "permissions": ["projects:*", "deployments:read"],
        "parentRoleId": "uuid-member",
        "memberCount": 15
      }
    ]
  }
```

#### Create Custom Role
```yaml
POST /organizations/{orgId}/roles

Request:
  {
    "name": "developer",
    "displayName": "Developer",
    "description": "Access to development resources",
    "parentRoleId": "uuid-member-role",
    "permissions": [
      "projects:create",
      "projects:read",
      "projects:update",
      "deployments:read"
    ]
  }

Response: 201 Created
```

#### Update Role
```yaml
PATCH /organizations/{orgId}/roles/{roleId}

Request:
  {
    "displayName": "Senior Developer",
    "permissions": ["projects:*", "deployments:*"]
  }

Response: 200 OK

Errors:
  400: Cannot modify system role
```

#### Delete Role
```yaml
DELETE /organizations/{orgId}/roles/{roleId}

Query Parameters:
  reassignTo: UUID (role to reassign members to)

Response: 204 No Content

Errors:
  400: Cannot delete system role / Role has members
```

#### Assign Role to Member
```yaml
POST /organizations/{orgId}/members/{memberId}/roles

Request:
  {
    "roleId": "uuid",
    "scopeType": "division",
    "scopeId": "uuid-division",
    "expiresAt": "2026-01-01T00:00:00Z"
  }

Response: 201 Created
```

#### Remove Role from Member
```yaml
DELETE /organizations/{orgId}/members/{memberId}/roles/{assignmentId}

Response: 204 No Content

Errors:
  400: Cannot remove last owner role
```

### 3.5 Permission Check Endpoint

#### Check Permission (REST)
```yaml
POST /permissions/check

Request:
  {
    "userId": "uuid",
    "organizationId": "uuid",
    "permission": "projects:create",
    "resourceType": "project",
    "resourceId": "uuid"
  }

Response: 200 OK
  {
    "success": true,
    "data": {
      "allowed": true,
      "matchedRole": "developer",
      "matchedScope": "organization"
    }
  }
```

#### Batch Permission Check
```yaml
POST /permissions/check/batch

Request:
  {
    "userId": "uuid",
    "organizationId": "uuid",
    "checks": [
      { "permission": "projects:create" },
      { "permission": "projects:delete", "resourceId": "uuid" },
      { "permission": "users:invite" }
    ]
  }

Response: 200 OK
  {
    "success": true,
    "data": {
      "results": [
        { "permission": "projects:create", "allowed": true },
        { "permission": "projects:delete", "allowed": false },
        { "permission": "users:invite", "allowed": true }
      ]
    }
  }
```

#### Get User Permissions
```yaml
GET /organizations/{orgId}/members/{memberId}/permissions

Response: 200 OK
  {
    "success": true,
    "data": {
      "permissions": [
        "organization:read",
        "users:read",
        "users:invite",
        "projects:*"
      ],
      "roles": ["admin", "developer"],
      "scopes": {
        "organization": ["organization:read", "users:*"],
        "division:uuid": ["projects:*"]
      }
    }
  }
```

### 3.6 API Key Endpoints

#### Create API Key
```yaml
POST /organizations/{orgId}/api-keys

Request:
  {
    "name": "CI/CD Pipeline",
    "description": "For automated deployments",
    "scopes": ["deployments:create", "projects:read"],
    "ipWhitelist": ["192.168.1.0/24"],
    "expiresInDays": 365
  }

Response: 201 Created
  {
    "success": true,
    "data": {
      "id": "uuid",
      "name": "CI/CD Pipeline",
      "keyPrefix": "fm_live_",
      "key": "fm_live_abc123xyz789...",  // Only shown once!
      "expiresAt": "2026-11-29T10:00:00Z",
      "createdAt": "2025-11-29T10:00:00Z"
    }
  }
```

#### List API Keys
```yaml
GET /organizations/{orgId}/api-keys

Response: 200 OK
  {
    "success": true,
    "data": [
      {
        "id": "uuid",
        "name": "CI/CD Pipeline",
        "keyPrefix": "fm_live_",
        "scopes": ["deployments:create", "projects:read"],
        "lastUsedAt": "2025-11-29T08:00:00Z",
        "expiresAt": "2026-11-29T10:00:00Z"
      }
    ]
  }
```

#### Revoke API Key
```yaml
DELETE /organizations/{orgId}/api-keys/{keyId}

Request:
  {
    "reason": "Compromised"
  }

Response: 204 No Content
```

### 3.7 gRPC Service Definition

```protobuf
syntax = "proto3";

package flowmind.iam.v1;

service IAMService {
  // High-performance permission check
  rpc CheckPermission(CheckPermissionRequest) returns (CheckPermissionResponse);
  rpc CheckPermissionBatch(CheckPermissionBatchRequest) returns (CheckPermissionBatchResponse);
  
  // Get user's effective permissions (cached)
  rpc GetEffectivePermissions(GetEffectivePermissionsRequest) returns (GetEffectivePermissionsResponse);
  
  // Validate API key
  rpc ValidateApiKey(ValidateApiKeyRequest) returns (ValidateApiKeyResponse);
}

message CheckPermissionRequest {
  string user_id = 1;
  string organization_id = 2;
  string permission = 3;
  optional string resource_type = 4;
  optional string resource_id = 5;
}

message CheckPermissionResponse {
  bool allowed = 1;
  optional string matched_role = 2;
  optional string matched_scope = 3;
  optional string denial_reason = 4;
}

message CheckPermissionBatchRequest {
  string user_id = 1;
  string organization_id = 2;
  repeated PermissionCheck checks = 3;
}

message PermissionCheck {
  string permission = 1;
  optional string resource_type = 2;
  optional string resource_id = 3;
}

message CheckPermissionBatchResponse {
  repeated CheckPermissionResponse results = 1;
}

message ValidateApiKeyRequest {
  string api_key = 1;
  string ip_address = 2;
}

message ValidateApiKeyResponse {
  bool valid = 1;
  optional string user_id = 2;
  optional string organization_id = 3;
  repeated string scopes = 4;
  optional string error = 5;
}
```

---

## 4. Event Contracts

### 4.1 Published Events

```typescript
// Member events
interface MemberInvitedEvent {
  eventType: 'member.invited';
  organizationId: string;
  email: string;
  roleIds: string[];
  invitedBy: string;
  invitationId: string;
}

interface MemberJoinedEvent {
  eventType: 'member.joined';
  organizationId: string;
  userId: string;
  membershipId: string;
  roles: string[];
}

interface MemberRemovedEvent {
  eventType: 'member.removed';
  organizationId: string;
  userId: string;
  membershipId: string;
  removedBy: string;
  reason?: string;
}

interface MemberSuspendedEvent {
  eventType: 'member.suspended';
  organizationId: string;
  userId: string;
  suspendedBy: string;
  reason?: string;
}

// Role events
interface RoleAssignedEvent {
  eventType: 'role.assigned';
  organizationId: string;
  userId: string;
  roleId: string;
  roleName: string;
  scopeType: string;
  scopeId?: string;
  grantedBy: string;
}

interface RoleRevokedEvent {
  eventType: 'role.revoked';
  organizationId: string;
  userId: string;
  roleId: string;
  roleName: string;
  revokedBy: string;
}

// API Key events
interface ApiKeyCreatedEvent {
  eventType: 'api_key.created';
  organizationId: string;
  userId: string;
  apiKeyId: string;
  name: string;
  scopes: string[];
}

interface ApiKeyRevokedEvent {
  eventType: 'api_key.revoked';
  organizationId: string;
  apiKeyId: string;
  reason?: string;
}
```

---

## 5. Permission Evaluation Algorithm

```typescript
async function checkPermission(request: PermissionCheckRequest): Promise<boolean> {
  const { userId, organizationId, permission, resourceId, resourceType } = request;
  
  // 1. Check cache first
  const cached = await cache.get(`perms:${userId}:${organizationId}`);
  if (cached && hasPermission(cached, permission)) {
    return true;
  }
  
  // 2. Get user's role assignments
  const assignments = await db.roleAssignments.findByMembership(userId, organizationId);
  
  // 3. Evaluate each assignment
  for (const assignment of assignments) {
    // Check scope match
    if (!scopeMatches(assignment, resourceType, resourceId)) {
      continue;
    }
    
    // Get role permissions (including inherited)
    const rolePerms = await getExpandedPermissions(assignment.roleId);
    
    // Check if permission matches
    if (permissionMatches(rolePerms, permission)) {
      // Update cache
      await cache.set(`perms:${userId}:${organizationId}`, rolePerms, TTL_5_MIN);
      return true;
    }
  }
  
  return false;
}

function permissionMatches(permissions: string[], required: string): boolean {
  const [resource, action] = required.split(':');
  
  for (const perm of permissions) {
    const [permResource, permAction] = perm.split(':');
    
    // Wildcard matches
    if (permResource === '*' || permResource === resource) {
      if (permAction === '*' || permAction === action) {
        return true;
      }
    }
  }
  
  return false;
}
```

---

## 6. Caching Strategy

### 6.1 Cache Keys
```
perms:{userId}:{orgId}           - User's effective permissions (TTL: 5min)
role:{roleId}                     - Role with expanded permissions (TTL: 10min)
membership:{userId}:{orgId}       - Membership details (TTL: 5min)
api_key:{prefix}                  - API key validation cache (TTL: 1min)
```

### 6.2 Cache Invalidation
Invalidate on:
- Role assignment change
- Role permission change
- Membership status change
- API key revocation

Publish invalidation event:
```typescript
await eventBus.publish('cache.invalidate', {
  pattern: `perms:${userId}:*`
});
```

---

## 7. Security Considerations

### 7.1 Permission Escalation Prevention
- Cannot grant permissions you don't have
- Cannot create role with more permissions than own
- Cannot assign role with higher privileges

### 7.2 API Key Security
- Keys hashed with bcrypt (cost factor 12)
- Key shown only once at creation
- IP whitelist enforced at gateway
- Rate limiting per key

### 7.3 Audit Requirements
All actions logged:
- Role changes
- Permission changes
- Member status changes
- API key operations

---

*FlowMind Technologies | IAM Service Technical Specification v1.0*

