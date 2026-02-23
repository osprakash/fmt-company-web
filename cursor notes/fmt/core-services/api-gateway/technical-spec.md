# API Gateway - Technical Specification

> **Service**: API Gateway  
> **Version**: 1.0  
> **Owner**: Platform Team  
> **Implementation**: Kong / AWS API Gateway / Custom  
> **Last Updated**: November 2025

---

## 1. Architecture Overview

```
                        ┌─────────────────────────────────────┐
                        │         Load Balancer               │
                        │    (api.flowmind.io)                │
                        └─────────────────┬───────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           API Gateway                                    │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────────┐    │
│  │   Auth     │  │   Rate     │  │  Request   │  │    Router      │    │
│  │ Validator  │  │  Limiter   │  │  Logger    │  │                │    │
│  └────────────┘  └────────────┘  └────────────┘  └────────────────┘    │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
         ┌───────────────────────────┼───────────────────────────┐
         │                           │                           │
         ▼                           ▼                           ▼
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│ Account Service │       │   IAM Service   │       │ Product Service │
└─────────────────┘       └─────────────────┘       └─────────────────┘
```

---

## 2. Configuration

### 2.1 Route Configuration

```yaml
# Kong declarative config example
services:
  - name: account-service
    url: http://account-service:3000
    routes:
      - name: accounts-route
        paths:
          - /v1/accounts
          - /v1/organizations
        strip_path: false
    plugins:
      - name: jwt
      - name: rate-limiting
        config:
          minute: 100
          policy: redis

  - name: identity-service
    url: http://identity-service:3000
    routes:
      - name: auth-route
        paths:
          - /v1/auth
        plugins:
          - name: rate-limiting
            config:
              minute: 30  # Stricter for auth
```

### 2.2 Rate Limiting Configuration

```yaml
rate_limits:
  default:
    per_tenant:
      requests_per_minute: 1000
      requests_per_hour: 50000
    per_user:
      requests_per_minute: 100
      requests_per_hour: 5000
    per_api_key:
      requests_per_minute: 500

  endpoints:
    "/v1/auth/token":
      per_ip:
        requests_per_minute: 10
    
    "/v1/subscriptions":
      per_tenant:
        requests_per_minute: 30
```

### 2.3 CORS Configuration

```yaml
cors:
  allowed_origins:
    - "https://*.flowmind.io"
    - "http://localhost:*"
  allowed_methods:
    - GET
    - POST
    - PUT
    - PATCH
    - DELETE
    - OPTIONS
  allowed_headers:
    - Authorization
    - Content-Type
    - X-Request-ID
    - X-Tenant-ID
  exposed_headers:
    - X-Request-ID
    - X-RateLimit-Remaining
  max_age: 86400
```

---

## 3. Request Flow

```typescript
async function handleRequest(req: Request): Promise<Response> {
  // 1. Parse request
  const requestId = req.headers['x-request-id'] || generateId();
  
  // 2. Tenant resolution (from subdomain, header, or JWT)
  const tenantId = await resolveTenant(req);
  if (!tenantId) {
    return errorResponse(400, 'Tenant not found');
  }
  
  // 3. Authentication
  const authResult = await authenticate(req);
  if (!authResult.authenticated) {
    return errorResponse(401, 'Unauthorized');
  }
  
  // 4. Rate limiting
  const rateLimitResult = await checkRateLimit(tenantId, authResult.userId, req.path);
  if (rateLimitResult.exceeded) {
    return errorResponse(429, 'Rate limit exceeded', {
      'Retry-After': rateLimitResult.retryAfter,
      'X-RateLimit-Remaining': 0
    });
  }
  
  // 5. Permission check (optional, can be delegated)
  if (requiresPermission(req.path)) {
    const hasPermission = await checkPermission(authResult.userId, tenantId, req.path);
    if (!hasPermission) {
      return errorResponse(403, 'Forbidden');
    }
  }
  
  // 6. Route to service
  const upstream = getUpstreamService(req.path);
  const response = await proxy(upstream, req, {
    headers: {
      'X-Request-ID': requestId,
      'X-Tenant-ID': tenantId,
      'X-User-ID': authResult.userId
    }
  });
  
  // 7. Log request
  await logRequest(req, response, {
    requestId,
    tenantId,
    userId: authResult.userId,
    duration: Date.now() - startTime
  });
  
  return response;
}
```

---

## 4. API Contracts

### 4.1 Standard Headers

#### Request Headers
```
Authorization: Bearer <jwt_token>
X-API-Key: <api_key>  (alternative to JWT)
X-Request-ID: <uuid>  (optional, generated if missing)
X-Tenant-ID: <uuid>   (optional, resolved from JWT or domain)
```

#### Response Headers
```
X-Request-ID: <uuid>
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset: 1701288000
```

### 4.2 Error Responses

```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Try again in 60 seconds.",
    "details": {
      "limit": 100,
      "remaining": 0,
      "resetAt": "2025-11-29T10:05:00Z"
    }
  },
  "requestId": "req-uuid-123"
}
```

### 4.3 Health Check

```yaml
GET /health

Response: 200 OK
  {
    "status": "healthy",
    "version": "1.0.0",
    "uptime": 86400,
    "services": {
      "account-service": "healthy",
      "identity-service": "healthy"
    }
  }
```

---

## 5. Security

### 5.1 JWT Validation
```typescript
async function validateJWT(token: string): Promise<JWTPayload> {
  // 1. Decode header
  const header = decodeHeader(token);
  
  // 2. Get public key from JWKS
  const publicKey = await getPublicKey(header.kid);
  
  // 3. Verify signature
  const payload = jwt.verify(token, publicKey, {
    algorithms: ['RS256'],
    issuer: 'https://auth.flowmind.io',
    audience: 'flowmind-api'
  });
  
  // 4. Check expiration
  if (payload.exp < Date.now() / 1000) {
    throw new Error('Token expired');
  }
  
  return payload;
}
```

### 5.2 API Key Validation
```typescript
async function validateApiKey(apiKey: string): Promise<ApiKeyInfo> {
  // 1. Extract prefix for lookup
  const prefix = apiKey.substring(0, 8);
  
  // 2. Look up in cache or IAM service
  const keyInfo = await iamService.validateApiKey(apiKey);
  
  // 3. Check IP whitelist
  if (keyInfo.ipWhitelist && !keyInfo.ipWhitelist.includes(req.ip)) {
    throw new Error('IP not whitelisted');
  }
  
  return keyInfo;
}
```

---

## 6. Monitoring

### 6.1 Metrics
```
# Request metrics
gateway_requests_total{method, path, status}
gateway_request_duration_seconds{method, path, quantile}
gateway_active_connections

# Rate limiting
gateway_rate_limit_hits{tenant_id, limit_type}
gateway_rate_limit_remaining{tenant_id}

# Upstream
gateway_upstream_requests{service, status}
gateway_upstream_latency{service, quantile}
```

---

*FlowMind Technologies | API Gateway Technical Specification v1.0*

