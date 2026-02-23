# Feature Flags Service - Technical Specification

> **Service**: Feature Flags Service  
> **Version**: 1.0  
> **Owner**: Platform Team  
> **Last Updated**: November 2025

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Feature Flags Service                                 │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────────────────┐  │
│  │ Flag Evaluator │  │ Admin Manager  │  │   Targeting Engine       │  │
│  │ (gRPC/REST)    │  │                │  │                          │  │
│  └────────────────┘  └────────────────┘  └──────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Data Model

```sql
CREATE TABLE feature_flags (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    key VARCHAR(100) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    
    flag_type VARCHAR(20) NOT NULL DEFAULT 'boolean',
    default_value JSONB NOT NULL,
    
    status VARCHAR(20) NOT NULL DEFAULT 'draft',
    
    targeting_rules JSONB NOT NULL DEFAULT '[]',
    percentage_rollout INTEGER,
    
    environments JSONB NOT NULL DEFAULT '{"production": false, "staging": true}',
    
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TABLE flag_overrides (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    flag_id UUID NOT NULL REFERENCES feature_flags(id),
    
    override_type VARCHAR(20) NOT NULL,  -- tenant, user, segment
    target_id UUID NOT NULL,
    
    value JSONB NOT NULL,
    
    expires_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

---

## 3. API Contracts

### 3.1 Evaluate Flag

```yaml
POST /v1/flags/evaluate

Request:
  {
    "flagKey": "new_dashboard",
    "context": {
      "userId": "uuid",
      "tenantId": "uuid",
      "attributes": {
        "plan": "enterprise",
        "country": "US"
      }
    }
  }

Response: 200 OK
  {
    "success": true,
    "data": {
      "key": "new_dashboard",
      "enabled": true,
      "value": true,
      "variant": null,
      "reason": "targeting_rule"
    }
  }
```

### 3.2 Batch Evaluate

```yaml
POST /v1/flags/evaluate/batch

Request:
  {
    "flagKeys": ["new_dashboard", "beta_feature", "dark_mode"],
    "context": { ... }
  }

Response: 200 OK
  {
    "success": true,
    "data": {
      "new_dashboard": { "enabled": true },
      "beta_feature": { "enabled": false },
      "dark_mode": { "enabled": true, "value": "auto" }
    }
  }
```

### 3.3 gRPC Service

```protobuf
service FeatureFlagsService {
  rpc Evaluate(EvaluateRequest) returns (EvaluateResponse);
  rpc EvaluateBatch(EvaluateBatchRequest) returns (EvaluateBatchResponse);
}
```

---

## 4. Evaluation Logic

```typescript
async function evaluateFlag(flagKey: string, context: EvalContext): Promise<FlagResult> {
  const flag = await getFlag(flagKey);
  
  // Check if flag is active
  if (flag.status !== 'active') {
    return { enabled: false, reason: 'flag_inactive' };
  }
  
  // Check environment
  if (!flag.environments[context.environment]) {
    return { enabled: false, reason: 'environment_disabled' };
  }
  
  // Check overrides
  const override = await checkOverrides(flag.id, context);
  if (override) {
    return { enabled: true, value: override.value, reason: 'override' };
  }
  
  // Check targeting rules
  for (const rule of flag.targetingRules) {
    if (matchesRule(rule, context)) {
      return { enabled: true, value: rule.value, reason: 'targeting_rule' };
    }
  }
  
  // Check percentage rollout
  if (flag.percentageRollout !== null) {
    const hash = hashContext(flagKey, context.userId || context.tenantId);
    if (hash <= flag.percentageRollout) {
      return { enabled: true, value: flag.defaultValue, reason: 'percentage_rollout' };
    }
  }
  
  return { enabled: false, value: flag.defaultValue, reason: 'default' };
}
```

---

## 5. Caching

```
flag:{key}                    - Flag definition (TTL: 1min)
flag:eval:{key}:{hash}        - Evaluation result (TTL: 30s)
```

---

*FlowMind Technologies | Feature Flags Service Technical Specification v1.0*

