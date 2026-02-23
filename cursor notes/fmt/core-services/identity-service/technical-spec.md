# Identity Service - Technical Specification

> **Service**: Identity Service  
> **Version**: 1.0  
> **Owner**: Platform Team  
> **Implementation**: BetterAuth  
> **Last Updated**: January 2026

---

## 1. Architecture Overview

### 1.1 Service Context
```
┌─────────────────────────────────────────────────────────────────────┐
│                          API Gateway                                 │
│                     (Routes /auth/* traffic)                         │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       Identity Service                               │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                        BetterAuth                               │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │ │
│  │  │ Auth Core    │  │ OIDC Plugin  │  │ Social Providers     │  │ │
│  │  │ - Sign up    │  │ - /authorize │  │ - Google             │  │ │
│  │  │ - Sign in    │  │ - /token     │  │ - GitHub             │  │ │
│  │  │ - Sessions   │  │ - /userinfo  │  │ - Microsoft          │  │ │
│  │  │ - Password   │  │ - JWKS       │  │ - LinkedIn           │  │ │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                │                                     │
│              ┌─────────────────┴─────────────────┐                   │
│              ▼                                   ▼                   │
│    ┌─────────────────┐                ┌─────────────────┐           │
│    │   PostgreSQL    │                │  Event Emitter  │           │
│    │   (BetterAuth   │                │  (Webhooks)     │           │
│    │    Adapter)     │                │                 │           │
│    └─────────────────┘                └─────────────────┘           │
└─────────────────────────────────────────────────────────────────────┘
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
| Runtime | Node.js 20 LTS | Team expertise, BetterAuth native |
| Framework | Hono / Express | Lightweight, BetterAuth compatible |
| Auth Library | BetterAuth | TypeScript-native, OIDC support, active development |
| Database | PostgreSQL 15 | BetterAuth adapter available |
| ORM | Drizzle | BetterAuth recommended, type-safe |
| API Docs | OpenAPI 3.0 | Industry standard |

### 1.3 BetterAuth Features Used
| Feature | Status | Notes |
|---------|--------|-------|
| Email/Password Auth | ✅ Built-in | Core feature |
| Social OAuth | ✅ Built-in | Google, GitHub, Microsoft, etc. |
| Session Management | ✅ Built-in | DB-backed sessions |
| OIDC Provider | ✅ Plugin | Authorization code + PKCE |
| Email Verification | ✅ Built-in | Requires email adapter |
| Password Reset | ✅ Built-in | Requires email adapter |
| PostgreSQL Adapter | ✅ Built-in | Drizzle/Prisma support |

### 1.4 Design Principles
1. **Use BetterAuth Defaults**: Minimize custom code, leverage built-in features
2. **OIDC-First**: All authentication flows follow OAuth 2.0 / OIDC specs
3. **Stateless Validation**: JWTs validated without database lookup
4. **No Business Logic**: No tenant, org, role, or product concepts
5. **Event-Driven**: Emit events for downstream services via BetterAuth hooks

---

## 2. Data Model

### 2.1 BetterAuth Schema (Auto-generated)
BetterAuth manages its own schema. Key tables:

```
┌──────────────────────────────────────────────────────────────────┐
│                            user                                   │
├──────────────────────────────────────────────────────────────────┤
│ PK │ id: TEXT                                                    │
│    │ email: TEXT UNIQUE                                          │
│    │ emailVerified: BOOLEAN                                      │
│    │ name: TEXT                                                  │
│    │ image: TEXT                                                 │
│    │ createdAt: TIMESTAMP                                        │
│    │ updatedAt: TIMESTAMP                                        │
└──────────────────────────────────────────────────────────────────┘
         │                              │
         │ 1:N                         │ 1:N
         ▼                              ▼
┌─────────────────────┐      ┌─────────────────────┐
│      session        │      │      account        │
├─────────────────────┤      ├─────────────────────┤
│ PK │ id: TEXT       │      │ PK │ id: TEXT       │
│ FK │ userId: TEXT   │      │ FK │ userId: TEXT   │
│    │ token: TEXT    │      │    │ providerId     │
│    │ expiresAt      │      │    │ accountId      │
│    │ ipAddress      │      │    │ accessToken    │
│    │ userAgent      │      │    │ refreshToken   │
│    │ createdAt      │      │    │ expiresAt      │
│    │ updatedAt      │      │    │ createdAt      │
└─────────────────────┘      └─────────────────────┘

┌─────────────────────┐
│   verification      │
├─────────────────────┤
│ PK │ id: TEXT       │
│    │ identifier     │
│    │ value: TEXT    │
│    │ expiresAt      │
│    │ createdAt      │
│    │ updatedAt      │
└─────────────────────┘
```

### 2.2 Database Setup (Drizzle)

```typescript
// schema.ts - BetterAuth auto-generates this
import { pgTable, text, timestamp, boolean } from "drizzle-orm/pg-core";

export const user = pgTable("user", {
  id: text("id").primaryKey(),
  name: text("name"),
  email: text("email").notNull().unique(),
  emailVerified: boolean("emailVerified").notNull().default(false),
  image: text("image"),
  createdAt: timestamp("createdAt").notNull().defaultNow(),
  updatedAt: timestamp("updatedAt").notNull().defaultNow(),
});

export const session = pgTable("session", {
  id: text("id").primaryKey(),
  userId: text("userId").notNull().references(() => user.id, { onDelete: "cascade" }),
  token: text("token").notNull().unique(),
  expiresAt: timestamp("expiresAt").notNull(),
  ipAddress: text("ipAddress"),
  userAgent: text("userAgent"),
  createdAt: timestamp("createdAt").notNull().defaultNow(),
  updatedAt: timestamp("updatedAt").notNull().defaultNow(),
});

export const account = pgTable("account", {
  id: text("id").primaryKey(),
  userId: text("userId").notNull().references(() => user.id, { onDelete: "cascade" }),
  accountId: text("accountId").notNull(),
  providerId: text("providerId").notNull(),
  accessToken: text("accessToken"),
  refreshToken: text("refreshToken"),
  accessTokenExpiresAt: timestamp("accessTokenExpiresAt"),
  refreshTokenExpiresAt: timestamp("refreshTokenExpiresAt"),
  scope: text("scope"),
  idToken: text("idToken"),
  createdAt: timestamp("createdAt").notNull().defaultNow(),
  updatedAt: timestamp("updatedAt").notNull().defaultNow(),
});

export const verification = pgTable("verification", {
  id: text("id").primaryKey(),
  identifier: text("identifier").notNull(),
  value: text("value").notNull(),
  expiresAt: timestamp("expiresAt").notNull(),
  createdAt: timestamp("createdAt").notNull().defaultNow(),
  updatedAt: timestamp("updatedAt").notNull().defaultNow(),
});
```

### 2.3 TypeScript Interfaces

```typescript
// BetterAuth User type
interface User {
  id: string;
  email: string;
  emailVerified: boolean;
  name?: string;
  image?: string;
  createdAt: Date;
  updatedAt: Date;
}

// BetterAuth Session type
interface Session {
  id: string;
  userId: string;
  token: string;
  expiresAt: Date;
  ipAddress?: string;
  userAgent?: string;
  createdAt: Date;
  updatedAt: Date;
}

// JWT Token Claims (OIDC Standard Only)
interface AccessTokenClaims {
  sub: string;      // User ID
  iss: string;      // Issuer URL
  aud: string;      // Client ID
  exp: number;      // Expiration
  iat: number;      // Issued at
  scope: string;    // Granted scopes
}

interface IDTokenClaims {
  sub: string;
  iss: string;
  aud: string;
  exp: number;
  iat: number;
  email?: string;
  email_verified?: boolean;
  name?: string;
  picture?: string;
}
```

---

## 3. BetterAuth Configuration

### 3.1 Core Setup

```typescript
// auth.ts
import { betterAuth } from "better-auth";
import { drizzleAdapter } from "better-auth/adapters/drizzle";
import { oidcProvider } from "better-auth/plugins/oidc-provider";
import { db } from "./db";
import * as schema from "./schema";

export const auth = betterAuth({
  database: drizzleAdapter(db, {
    provider: "pg",
    schema,
  }),
  
  // Email/Password Authentication
  emailAndPassword: {
    enabled: true,
    minPasswordLength: 8,
    requireEmailVerification: true,
  },
  
  // Social Providers
  socialProviders: {
    google: {
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    },
    github: {
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    },
    microsoft: {
      clientId: process.env.MICROSOFT_CLIENT_ID!,
      clientSecret: process.env.MICROSOFT_CLIENT_SECRET!,
    },
  },
  
  // Session Configuration
  session: {
    expiresIn: 60 * 60 * 24 * 7, // 7 days
    updateAge: 60 * 60 * 24,     // Update session every 24 hours
    cookieCache: {
      enabled: true,
      maxAge: 60 * 5, // 5 minutes
    },
  },
  
  // OIDC Provider Plugin
  plugins: [
    oidcProvider({
      // Signing keys
      jwks: {
        keyPairConfig: {
          alg: "RS256",
        },
      },
      // Token lifetimes
      accessTokenLifetime: 15 * 60,      // 15 minutes
      refreshTokenLifetime: 7 * 24 * 60 * 60, // 7 days
      idTokenLifetime: 60 * 60,          // 1 hour
      // Supported scopes
      scopes: ["openid", "profile", "email", "offline_access"],
    }),
  ],
  
  // Email Configuration
  emailVerification: {
    sendVerificationEmail: async ({ user, url }) => {
      await sendEmail({
        to: user.email,
        subject: "Verify your email",
        html: `<a href="${url}">Verify Email</a>`,
      });
    },
  },
  
  // Event Hooks (for downstream services)
  hooks: {
    after: [
      {
        matcher: (ctx) => ctx.path.startsWith("/sign-up"),
        handler: async (ctx) => {
          if (ctx.response?.user) {
            await emitEvent("identity.user.created", {
              userId: ctx.response.user.id,
              email: ctx.response.user.email,
            });
          }
        },
      },
      {
        matcher: (ctx) => ctx.path.startsWith("/sign-in"),
        handler: async (ctx) => {
          if (ctx.response?.session) {
            await emitEvent("identity.login.success", {
              userId: ctx.response.session.userId,
              sessionId: ctx.response.session.id,
            });
          }
        },
      },
    ],
  },
});
```

### 3.2 OIDC Client Registration

```typescript
// Register OIDC clients for your applications
const oidcClients = [
  {
    clientId: "flowmind-web",
    clientSecret: null, // Public client (SPA)
    redirectUris: [
      "http://localhost:3000/callback",
      "https://app.flowmind.io/callback",
    ],
    grantTypes: ["authorization_code", "refresh_token"],
    responseTypes: ["code"],
    scopes: ["openid", "profile", "email", "offline_access"],
    tokenEndpointAuthMethod: "none", // Public client
  },
  {
    clientId: "flowmind-api",
    clientSecret: process.env.API_CLIENT_SECRET,
    redirectUris: ["https://api.flowmind.io/callback"],
    grantTypes: ["authorization_code", "refresh_token", "client_credentials"],
    responseTypes: ["code"],
    scopes: ["openid", "profile", "email"],
    tokenEndpointAuthMethod: "client_secret_basic",
  },
];
```

---

## 4. API Contracts

### 4.1 Base URLs
```
Production:  https://auth.flowmind.io
Staging:     https://auth.staging.flowmind.io
Development: http://localhost:3001
```

### 4.2 OIDC Standard Endpoints

#### Discovery Document
```yaml
GET /.well-known/openid-configuration

Response: 200 OK
{
  "issuer": "https://auth.flowmind.io",
  "authorization_endpoint": "https://auth.flowmind.io/api/auth/authorize",
  "token_endpoint": "https://auth.flowmind.io/api/auth/token",
  "userinfo_endpoint": "https://auth.flowmind.io/api/auth/userinfo",
  "jwks_uri": "https://auth.flowmind.io/.well-known/jwks.json",
  "scopes_supported": ["openid", "profile", "email", "offline_access"],
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "token_endpoint_auth_methods_supported": ["client_secret_basic", "client_secret_post", "none"],
  "code_challenge_methods_supported": ["S256"]
}
```

#### Authorization Endpoint
```yaml
GET /api/auth/authorize

Query Parameters:
  response_type: "code" (required)
  client_id: string (required)
  redirect_uri: string (required)
  scope: string (required, must include "openid")
  state: string (required)
  code_challenge: string (required for public clients)
  code_challenge_method: "S256" (required with code_challenge)

Response: 302 Redirect to login page, then to redirect_uri with code
```

#### Token Endpoint
```yaml
POST /api/auth/token

Headers:
  Content-Type: application/x-www-form-urlencoded

Body (Authorization Code):
  grant_type: "authorization_code"
  code: string
  redirect_uri: string
  client_id: string
  code_verifier: string (if PKCE used)

Body (Refresh Token):
  grant_type: "refresh_token"
  refresh_token: string

Response: 200 OK
{
  "access_token": "eyJ...",
  "token_type": "Bearer",
  "expires_in": 900,
  "refresh_token": "...",
  "id_token": "eyJ...",
  "scope": "openid profile email"
}
```

#### UserInfo Endpoint
```yaml
GET /api/auth/userinfo

Headers:
  Authorization: Bearer {access_token}

Response: 200 OK
{
  "sub": "user_abc123",
  "email": "user@example.com",
  "email_verified": true,
  "name": "John Doe",
  "picture": "https://..."
}
```

#### JWKS Endpoint
```yaml
GET /.well-known/jwks.json

Response: 200 OK
{
  "keys": [
    {
      "kty": "RSA",
      "kid": "...",
      "use": "sig",
      "alg": "RS256",
      "n": "...",
      "e": "AQAB"
    }
  ]
}
```

### 4.3 BetterAuth Endpoints

#### Sign Up
```yaml
POST /api/auth/sign-up/email

Body:
{
  "email": "user@example.com",
  "password": "securepassword",
  "name": "John Doe"
}

Response: 200 OK
{
  "user": {
    "id": "user_abc123",
    "email": "user@example.com",
    "emailVerified": false,
    "name": "John Doe"
  }
}
```

#### Sign In
```yaml
POST /api/auth/sign-in/email

Body:
{
  "email": "user@example.com",
  "password": "securepassword"
}

Response: 200 OK
{
  "user": { ... },
  "session": {
    "id": "session_xyz",
    "userId": "user_abc123",
    "token": "...",
    "expiresAt": "2026-01-25T10:00:00Z"
  }
}
```

#### Social Sign In
```yaml
GET /api/auth/sign-in/social

Query Parameters:
  provider: "google" | "github" | "microsoft"
  callbackURL: string

Response: 302 Redirect to OAuth provider
```

#### Sign Out
```yaml
POST /api/auth/sign-out

Headers:
  Authorization: Bearer {session_token}

Response: 200 OK
```

#### Get Session
```yaml
GET /api/auth/session

Headers:
  Cookie: better-auth.session_token=...

Response: 200 OK
{
  "user": { ... },
  "session": { ... }
}
```

#### List Sessions
```yaml
GET /api/auth/sessions

Headers:
  Authorization: Bearer {session_token}

Response: 200 OK
{
  "sessions": [
    {
      "id": "session_xyz",
      "ipAddress": "192.168.1.1",
      "userAgent": "Mozilla/5.0...",
      "createdAt": "2026-01-18T10:00:00Z",
      "current": true
    }
  ]
}
```

#### Revoke Session
```yaml
POST /api/auth/sessions/revoke

Body:
{
  "sessionId": "session_xyz"
}

Response: 200 OK
```

#### Forgot Password
```yaml
POST /api/auth/forgot-password

Body:
{
  "email": "user@example.com"
}

Response: 200 OK (always, to prevent enumeration)
```

#### Reset Password
```yaml
POST /api/auth/reset-password

Body:
{
  "token": "reset_token_from_email",
  "newPassword": "newsecurepassword"
}

Response: 200 OK
```

#### Verify Email
```yaml
POST /api/auth/verify-email

Body:
{
  "token": "verification_token"
}

Response: 200 OK
```

---

## 5. Event Contracts

### 5.1 Event Format
```typescript
interface IdentityEvent {
  eventId: string;
  eventType: string;
  timestamp: string;
  data: Record<string, unknown>;
}
```

### 5.2 Published Events

```typescript
// Emitted via BetterAuth hooks
const events = {
  "identity.user.created": {
    userId: string,
    email: string,
    emailVerified: boolean,
  },
  
  "identity.user.deleted": {
    userId: string,
    email: string,
  },
  
  "identity.login.success": {
    userId: string,
    sessionId: string,
    method: "email" | "social",
    provider?: string,
  },
  
  "identity.login.failed": {
    email: string,
    reason: string,
  },
  
  "identity.logout": {
    userId: string,
    sessionId: string,
  },
  
  "identity.password.changed": {
    userId: string,
  },
  
  "identity.password.reset": {
    userId: string,
  },
  
  "identity.email.verified": {
    userId: string,
    email: string,
  },
  
  "identity.session.revoked": {
    userId: string,
    sessionId: string,
  },
  
  "identity.account.linked": {
    userId: string,
    provider: string,
  },
  
  "identity.account.unlinked": {
    userId: string,
    provider: string,
  },
};
```

### 5.3 Event Emission

```typescript
// event-emitter.ts
import { EventEmitter } from "events";

const eventBus = new EventEmitter();

export async function emitEvent(type: string, data: Record<string, unknown>) {
  const event = {
    eventId: crypto.randomUUID(),
    eventType: type,
    timestamp: new Date().toISOString(),
    data,
  };
  
  // Local emission (for in-process handlers)
  eventBus.emit(type, event);
  
  // Publish to message broker
  await publishToQueue("flowmind.identity-service.events", event);
}
```

---

## 6. Security Considerations

### 6.1 Password Security
BetterAuth handles password hashing automatically using secure defaults (bcrypt/argon2).

### 6.2 Token Security
```typescript
const securityConfig = {
  accessToken: {
    algorithm: "RS256",
    expiresIn: 900,  // 15 minutes
  },
  refreshToken: {
    expiresIn: 604800,  // 7 days
  },
  session: {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "lax",
  },
};
```

### 6.3 PKCE
- Required for all public clients (SPAs)
- Only `S256` challenge method supported

### 6.4 CORS Configuration
```typescript
const corsConfig = {
  origin: [
    "https://app.flowmind.io",
    "https://admin.flowmind.io",
  ],
  credentials: true,
};
```

---

## 7. Integration Points

### 7.1 Downstream Consumers
| Service | Purpose | Integration |
|---------|---------|-------------|
| All Services | JWT validation | JWKS endpoint |
| Account Service | User-to-org mapping | Events |
| IAM Service | Permission resolution | Events |

### 7.2 External Dependencies
| System | Purpose | Required |
|--------|---------|----------|
| PostgreSQL | User/session storage | Yes |
| SMTP Provider | Emails | Yes |
| Google OAuth | Social login | Optional |
| GitHub OAuth | Social login | Optional |

### 7.3 Service-to-Service Auth
```typescript
// Other services validate JWTs by:
// 1. Fetch JWKS from /.well-known/jwks.json (cache for 24h)
// 2. Validate JWT signature using public key
// 3. Check claims: iss, aud, exp
// 4. Use 'sub' claim to identify user
```

---

## 8. Error Codes

### 8.1 OAuth/OIDC Errors
| Error | HTTP | Description |
|-------|------|-------------|
| invalid_request | 400 | Missing/invalid parameter |
| invalid_client | 401 | Client auth failed |
| invalid_grant | 400 | Invalid code/token |
| access_denied | 403 | User denied |
| invalid_scope | 400 | Invalid scope |

### 8.2 BetterAuth Errors
| Error | HTTP | Description |
|-------|------|-------------|
| USER_NOT_FOUND | 404 | User doesn't exist |
| INVALID_PASSWORD | 401 | Wrong password |
| EMAIL_NOT_VERIFIED | 403 | Email not verified |
| SESSION_EXPIRED | 401 | Session expired |
| INVALID_TOKEN | 400 | Invalid verification token |

---

## 9. Deployment

### 9.1 Environment Variables
```bash
# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/identity

# BetterAuth
BETTER_AUTH_SECRET=your-secret-key
BETTER_AUTH_URL=https://auth.flowmind.io

# OAuth Providers
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GITHUB_CLIENT_ID=...
GITHUB_CLIENT_SECRET=...
MICROSOFT_CLIENT_ID=...
MICROSOFT_CLIENT_SECRET=...

# Email
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=...
SMTP_PASS=...
FROM_EMAIL=noreply@flowmind.io
```

### 9.2 Health Check
```yaml
GET /health

Response: 200 OK
{
  "status": "healthy",
  "database": "connected",
  "version": "1.0.0"
}
```

---

## 10. Future Migration to Keycloak

### 10.1 Migration Triggers
- Need SAML 2.0 support
- Enterprise MFA requirements
- Complex IdP federation
- Advanced security policies

### 10.2 Migration Path
```
1. Deploy Keycloak alongside BetterAuth
2. Configure Keycloak with same OIDC settings
3. Proxy OIDC endpoints through API Gateway
4. Migrate users (password reset required)
5. Switch traffic to Keycloak
6. Decommission BetterAuth
```

### 10.3 Consumer Impact
Because we only expose OIDC contracts:
- **Zero code changes** for consumers
- Same endpoints (proxied)
- Same token format
- Same claims structure

---

*FlowMind Technologies | Identity Service Technical Specification v1.0*
