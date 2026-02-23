# Identity Service - Requirements Document

> **Service**: Identity Service  
> **Version**: 1.0  
> **Owner**: Platform Team  
> **Implementation**: BetterAuth  
> **Contract**: OAuth 2.0 / OIDC Standard  
> **Last Updated**: January 2026

---

## 1. Business Context

### 1.1 Purpose
The Identity Service provides **authentication and identity management only** for all FlowMind Technologies products. It serves as the single source of truth for **user identities** and handles all authentication flows.

**Key Principle**: This service exposes **only OAuth 2.0 / OIDC standard contracts**. No product, organization, or account-specific logic resides here.

### 1.2 Business Goals
- Provide secure, scalable authentication for all products
- Enable social login for B2C products
- Maintain compliance with security standards (SOC2, GDPR)
- **Provider-agnostic**: Depend only on OAuth2/OIDC contracts
- **Migration-ready**: Easy swap to Keycloak when enterprise features needed

### 1.3 Scope
| In Scope | Out of Scope |
|----------|--------------|
| User authentication (email/password) | Authorization/permissions (IAM Service) |
| Social login (OAuth2 providers) | User profile management (Account Service) |
| Session management | Organizational structure (Account Service) |
| Basic OIDC endpoints | Subscription management |
| Password reset/change | Tenant configuration |
| Email verification | Product-specific logic |
| Standard OIDC claims only | Custom business claims |
| | SAML 2.0 (Future - Keycloak) |
| | MFA/2FA (Future - Keycloak) |
| | Enterprise SSO (Future - Keycloak) |

### 1.4 Architecture Principle
```
┌─────────────────────────────────────────────────────────────────┐
│                      Consumer Services                          │
│         (API Gateway, Account Service, IAM Service, etc.)       │
└─────────────────────────────────────────────────────────────────┘
                              │
                    OAuth2/OIDC Standard Contracts
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Identity Service                            │
│                       (BetterAuth)                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Functional Requirements

### 2.1 Authentication Methods

#### FR-IDN-001: Email/Password Authentication
- System shall support email + password authentication
- System shall enforce password policies (min length, complexity)
- System shall support password reset via email
- System shall support password change for authenticated users

#### FR-IDN-002: Social Login (OAuth2 Providers)
- System shall support OAuth2 social providers:
  - Google
  - GitHub
  - Microsoft (Azure AD)
  - LinkedIn
  - Apple (for mobile apps)
- Social accounts shall be linkable to existing accounts
- Implementation via standard OAuth2 flows

### 2.2 Session Management

#### FR-IDN-010: Session Handling
- System shall create sessions on successful authentication
- System shall support session expiration (configurable)
- System shall support manual session revocation
- System shall support "remember me" functionality
- System shall track session metadata (IP, user agent)

#### FR-IDN-011: Token Management (OIDC Standard)
- System shall issue JWT access tokens
- System shall issue refresh tokens for session extension
- System shall issue ID tokens per OIDC Core spec
- Access token lifetime: configurable (default 15 min)
- Refresh token lifetime: configurable (default 7 days)
- System shall support PKCE for public clients

#### FR-IDN-012: Token Claims (Standard OIDC Only)
Tokens shall include **only standard OIDC claims**:
```
Required Claims:
  - sub       : Subject identifier (unique user ID)
  - iss       : Issuer identifier
  - aud       : Audience
  - exp       : Expiration time
  - iat       : Issued at time

Standard Profile Claims (ID Token / UserInfo):
  - email           : User's email address
  - email_verified  : Email verification status
  - name            : Full name (if provided)
  - picture         : Profile picture URL (if provided)
```

**Note**: Business-specific claims (tenant_id, org_id, roles, permissions) are **NOT** included. These are resolved by downstream services using the `sub` claim.

### 2.3 Password Management

#### FR-IDN-020: Password Policies
- System shall enforce minimum password length (8 chars)
- System shall enforce complexity requirements (configurable)

#### FR-IDN-021: Password Reset
- System shall send password reset emails
- Reset links shall expire (configurable, default 1 hour)
- Reset links shall be single-use
- System shall notify user of password changes

#### FR-IDN-022: Password Change
- Authenticated users shall be able to change password
- Current password verification required

### 2.4 User Identity Lifecycle

#### FR-IDN-030: User Registration
- System shall support self-registration via email/password
- System shall support registration via social providers
- System shall verify email addresses
- Registration creates identity only (no org/account association)

#### FR-IDN-031: Account Linking
- System shall support linking multiple OAuth providers to one user
- User can link/unlink social accounts
- System shall handle email conflicts gracefully

#### FR-IDN-032: Account Deletion
- System shall support user-initiated account deletion
- System shall support admin-initiated account deletion
- Deletion shall emit event for downstream cleanup

### 2.5 OIDC Provider Endpoints

#### FR-IDN-040: Standard OIDC Endpoints
- System shall expose `/.well-known/openid-configuration`
- System shall expose `/authorize` endpoint
- System shall expose `/token` endpoint
- System shall expose `/userinfo` endpoint
- System shall expose JWKS endpoint for token validation

---

## 3. Non-Functional Requirements

### 3.1 Performance
| Metric | Target |
|--------|--------|
| Login Response Time (p95) | < 500ms |
| Token Validation (p95) | < 10ms |
| Throughput | 1000 auth/second |

### 3.2 Availability
- 99.9% uptime target
- Graceful degradation (cached JWKS for token validation)

### 3.3 Security
- OWASP Top 10 compliance
- Encrypted password storage (Argon2/bcrypt)
- PKCE required for all public clients
- Secrets in environment variables / secret manager

### 3.4 Compliance
- GDPR: Data portability, right to deletion
- SOC2: Audit logging, access controls

---

## 4. User Stories

### Epic: User Authentication

#### US-IDN-001: Email Login
**As a** user  
**I want to** log in with my email and password  
**So that** I can access my account  

**Acceptance Criteria:**
- [ ] Login form accepts email and password
- [ ] Invalid credentials show generic error
- [ ] Successful login returns OIDC tokens
- [ ] Session created with metadata

#### US-IDN-002: Social Login
**As a** user  
**I want to** log in with my Google/GitHub account  
**So that** I don't need to remember another password  

**Acceptance Criteria:**
- [ ] OAuth buttons shown on login page
- [ ] First-time users have identity created
- [ ] Existing users with same email are linked
- [ ] User can unlink social account later

#### US-IDN-003: Forgot Password
**As a** user  
**I want to** reset my forgotten password  
**So that** I can regain access to my account  

**Acceptance Criteria:**
- [ ] Forgot password link on login page
- [ ] Email with reset link sent
- [ ] Reset link expires after 1 hour
- [ ] New password must meet policy

#### US-IDN-004: Email Verification
**As a** user  
**I want to** verify my email address  
**So that** I can prove ownership of my email  

**Acceptance Criteria:**
- [ ] Verification email sent on registration
- [ ] Clicking link marks email as verified
- [ ] Can resend verification email

---

## 5. API Contracts

### 5.1 OIDC Standard Endpoints
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/.well-known/openid-configuration` | GET | Discovery document |
| `/api/auth/authorize` | GET | Authorization endpoint |
| `/api/auth/token` | POST | Token endpoint |
| `/api/auth/userinfo` | GET | UserInfo endpoint |
| `/.well-known/jwks.json` | GET | JSON Web Key Set |

### 5.2 BetterAuth Endpoints
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/auth/sign-up` | POST | User registration |
| `/api/auth/sign-in` | POST | Email/password login |
| `/api/auth/sign-out` | POST | Logout |
| `/api/auth/forgot-password` | POST | Request password reset |
| `/api/auth/reset-password` | POST | Complete password reset |
| `/api/auth/verify-email` | POST | Verify email address |
| `/api/auth/session` | GET | Get current session |
| `/api/auth/sessions` | GET | List all sessions |

### 5.3 Event Contracts
Identity events emitted for downstream services:

```yaml
Events:
  - identity.user.created
  - identity.user.deleted
  - identity.login.success
  - identity.login.failed
  - identity.logout
  - identity.password.changed
  - identity.password.reset
  - identity.email.verified
  - identity.session.revoked
  - identity.account.linked
  - identity.account.unlinked
```

---

## 6. Dependencies

### 6.1 Upstream Dependencies
| Service | Dependency Type | Description |
|---------|----------------|-------------|
| API Gateway | Routing | Routes /auth/* to Identity Service |

**Note**: No upstream business service dependencies. Identity Service is foundational.

### 6.2 Downstream Consumers
| Service | Consumption Type | Description |
|---------|-----------------|-------------|
| Account Service | Token validation, user lookup by `sub` | Links identity to org/account |
| IAM Service | Token validation | Resolves permissions for identity |
| All Services | JWKS for token validation | Stateless JWT validation |

### 6.3 External Dependencies
| System | Purpose |
|--------|---------|
| PostgreSQL | User identity storage (BetterAuth adapter) |
| Redis | Session storage (optional) |
| Email Provider | Password reset, email verification |

---

## 7. Implementation Strategy

### 7.1 Current: BetterAuth
BetterAuth provides out-of-the-box:
- ✅ Email/password authentication
- ✅ OAuth social login (Google, GitHub, Microsoft, etc.)
- ✅ Session management
- ✅ OIDC provider plugin
- ✅ PKCE support
- ✅ PostgreSQL adapter
- ✅ Email verification
- ✅ Password reset

### 7.2 What This Service Does NOT Do (MVP)
- ❌ MFA/2FA (defer to Keycloak)
- ❌ SAML 2.0 SSO (defer to Keycloak)
- ❌ Enterprise IdP federation (defer to Keycloak)
- ❌ Advanced rate limiting (add separately if needed)
- ❌ Store organization/tenant data
- ❌ Manage user roles or permissions
- ❌ Include business claims in tokens

### 7.3 Future: Keycloak Migration
Migration triggers:
- Need for SAML 2.0 support
- Enterprise MFA requirements
- Complex IdP federation
- Advanced security policies

**Migration Path:**
Because we expose only OIDC contracts:
- No consumer code changes required
- Same token format (JWT)
- Same endpoints (proxied if needed)
- Same claims structure

---

## 8. Acceptance Criteria Summary

### MVP Criteria (BetterAuth)
- [ ] Email/password authentication
- [ ] Google social login
- [ ] GitHub social login
- [ ] JWT token issuance (OIDC compliant)
- [ ] Password reset flow
- [ ] Email verification
- [ ] Session management (list, revoke)
- [ ] OIDC discovery endpoint
- [ ] JWKS endpoint
- [ ] PKCE support
- [ ] Event emission for user lifecycle

### V1.1 Criteria
- [ ] All social providers (Microsoft, LinkedIn, Apple)
- [ ] Session metadata tracking (IP, user agent)
- [ ] Account linking/unlinking UI

### Future (Keycloak Migration)
- [ ] TOTP/MFA
- [ ] SAML 2.0 SSO
- [ ] Enterprise IdP federation
- [ ] WebAuthn/FIDO2
- [ ] Advanced security policies
- [ ] SCIM provisioning

---

## 9. Glossary

| Term | Definition |
|------|------------|
| Subject (sub) | Unique identifier for a user identity |
| Identity | Authentication credentials and basic profile (email, name) |
| Account | Business entity in Account Service, linked to identity via `sub` |
| OIDC | OpenID Connect - authentication layer on OAuth 2.0 |
| PKCE | Proof Key for Code Exchange - OAuth security extension |
| JWK | JSON Web Key - public key for token verification |

---

*FlowMind Technologies | Identity Service Requirements v1.0*
