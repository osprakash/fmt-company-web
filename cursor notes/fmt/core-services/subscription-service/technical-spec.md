# Subscription & Billing Service - Technical Specification

> **Service**: Subscription & Billing Service  
> **Version**: 1.0  
> **Owner**: Platform Team  
> **Last Updated**: November 2025

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Subscription Service                                 │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────────────────┐  │
│  │ Subscription   │  │   Billing      │  │    Webhook Handler       │  │
│  │   Manager      │  │   Manager      │  │    (Stripe Events)       │  │
│  └────────────────┘  └────────────────┘  └──────────────────────────┘  │
│                                                                          │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────────────────┐  │
│  │   Invoice      │  │   Dunning      │  │    Payment Method        │  │
│  │   Generator    │  │   Manager      │  │    Manager               │  │
│  └────────────────┘  └────────────────┘  └──────────────────────────┘  │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
                           ┌─────────────────┐
                           │     Stripe      │
                           │      API        │
                           └─────────────────┘
```

---

## 2. Data Model

### 2.1 Database Schema

```sql
-- Subscriptions
CREATE TABLE subscriptions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL,
    account_id UUID NOT NULL,  -- Billing account
    
    -- Plan
    product_id UUID NOT NULL,
    tier_id UUID NOT NULL,
    billing_cycle VARCHAR(20) NOT NULL DEFAULT 'monthly',  -- monthly, annual
    
    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    status_reason TEXT,
    
    -- Dates
    current_period_start TIMESTAMP WITH TIME ZONE NOT NULL,
    current_period_end TIMESTAMP WITH TIME ZONE NOT NULL,
    trial_end TIMESTAMP WITH TIME ZONE,
    cancelled_at TIMESTAMP WITH TIME ZONE,
    cancel_at_period_end BOOLEAN NOT NULL DEFAULT FALSE,
    
    -- Stripe
    stripe_subscription_id VARCHAR(100) UNIQUE,
    stripe_customer_id VARCHAR(100) NOT NULL,
    
    -- Pricing (snapshot)
    price_amount DECIMAL(10,2) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_subscriptions_org ON subscriptions(organization_id);
CREATE INDEX idx_subscriptions_stripe ON subscriptions(stripe_subscription_id);
CREATE INDEX idx_subscriptions_status ON subscriptions(status);

-- Subscription Items (for usage-based or multi-item subscriptions)
CREATE TABLE subscription_items (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    subscription_id UUID NOT NULL REFERENCES subscriptions(id) ON DELETE CASCADE,
    
    item_type VARCHAR(50) NOT NULL,  -- base, addon, usage
    price_id VARCHAR(100) NOT NULL,  -- Stripe price ID
    quantity INTEGER NOT NULL DEFAULT 1,
    
    stripe_subscription_item_id VARCHAR(100),
    
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Payment Methods
CREATE TABLE payment_methods (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL,
    account_id UUID NOT NULL,
    
    type VARCHAR(20) NOT NULL,  -- card, bank_transfer
    is_default BOOLEAN NOT NULL DEFAULT FALSE,
    
    -- Card details (non-sensitive)
    card_brand VARCHAR(20),
    card_last4 CHAR(4),
    card_exp_month INTEGER,
    card_exp_year INTEGER,
    
    stripe_payment_method_id VARCHAR(100) NOT NULL,
    
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payment_methods_org ON payment_methods(organization_id);

-- Invoices
CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL,
    subscription_id UUID REFERENCES subscriptions(id),
    
    -- Invoice details
    invoice_number VARCHAR(50) NOT NULL UNIQUE,
    status VARCHAR(20) NOT NULL DEFAULT 'draft',  -- draft, open, paid, void, uncollectible
    
    -- Amounts
    subtotal DECIMAL(10,2) NOT NULL,
    tax DECIMAL(10,2) NOT NULL DEFAULT 0,
    total DECIMAL(10,2) NOT NULL,
    amount_paid DECIMAL(10,2) NOT NULL DEFAULT 0,
    amount_due DECIMAL(10,2) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    
    -- Dates
    invoice_date DATE NOT NULL,
    due_date DATE NOT NULL,
    paid_at TIMESTAMP WITH TIME ZONE,
    
    -- Stripe
    stripe_invoice_id VARCHAR(100) UNIQUE,
    stripe_invoice_pdf VARCHAR(500),
    stripe_hosted_invoice_url VARCHAR(500),
    
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_invoices_org ON invoices(organization_id);
CREATE INDEX idx_invoices_status ON invoices(status);
CREATE INDEX idx_invoices_stripe ON invoices(stripe_invoice_id);

-- Invoice Line Items
CREATE TABLE invoice_line_items (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    invoice_id UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    
    description TEXT NOT NULL,
    quantity DECIMAL(10,4) NOT NULL DEFAULT 1,
    unit_amount DECIMAL(10,4) NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    
    period_start DATE,
    period_end DATE,
    proration BOOLEAN NOT NULL DEFAULT FALSE,
    
    metadata JSONB NOT NULL DEFAULT '{}'
);

-- Payment Attempts (for dunning)
CREATE TABLE payment_attempts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    invoice_id UUID NOT NULL REFERENCES invoices(id),
    payment_method_id UUID REFERENCES payment_methods(id),
    
    amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) NOT NULL,  -- pending, succeeded, failed
    failure_reason TEXT,
    failure_code VARCHAR(50),
    
    stripe_payment_intent_id VARCHAR(100),
    
    attempted_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payment_attempts_invoice ON payment_attempts(invoice_id);

-- Subscription Events (history)
CREATE TABLE subscription_events (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    subscription_id UUID NOT NULL REFERENCES subscriptions(id),
    
    event_type VARCHAR(50) NOT NULL,
    previous_state JSONB,
    new_state JSONB,
    
    triggered_by UUID,  -- User ID or null for system
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sub_events_subscription ON subscription_events(subscription_id, created_at DESC);
```

### 2.2 TypeScript Interfaces

```typescript
interface Subscription {
  id: string;
  organizationId: string;
  accountId: string;
  productId: string;
  tierId: string;
  billingCycle: 'monthly' | 'annual';
  status: SubscriptionStatus;
  statusReason?: string;
  currentPeriodStart: Date;
  currentPeriodEnd: Date;
  trialEnd?: Date;
  cancelledAt?: Date;
  cancelAtPeriodEnd: boolean;
  stripeSubscriptionId?: string;
  stripeCustomerId: string;
  priceAmount: number;
  currency: string;
}

type SubscriptionStatus = 
  | 'trialing'
  | 'active'
  | 'past_due'
  | 'cancelled'
  | 'paused'
  | 'incomplete'
  | 'incomplete_expired';

interface Invoice {
  id: string;
  organizationId: string;
  subscriptionId?: string;
  invoiceNumber: string;
  status: InvoiceStatus;
  subtotal: number;
  tax: number;
  total: number;
  amountPaid: number;
  amountDue: number;
  currency: string;
  invoiceDate: Date;
  dueDate: Date;
  paidAt?: Date;
  stripeInvoiceId?: string;
  stripePdfUrl?: string;
  lineItems: InvoiceLineItem[];
}

type InvoiceStatus = 'draft' | 'open' | 'paid' | 'void' | 'uncollectible';

interface PaymentMethod {
  id: string;
  type: 'card' | 'bank_transfer';
  isDefault: boolean;
  card?: {
    brand: string;
    last4: string;
    expMonth: number;
    expYear: number;
  };
}
```

---

## 3. API Contracts

### 3.1 Subscription Endpoints

#### Create Subscription
```yaml
POST /v1/subscriptions

Request:
  {
    "organizationId": "uuid",
    "accountId": "uuid",
    "tierId": "uuid",
    "billingCycle": "annual",
    "paymentMethodId": "uuid",
    "trialDays": 14
  }

Response: 201 Created
  {
    "success": true,
    "data": {
      "id": "uuid",
      "status": "trialing",
      "currentPeriodStart": "2025-11-29T00:00:00Z",
      "currentPeriodEnd": "2025-12-13T00:00:00Z",
      "trialEnd": "2025-12-13T00:00:00Z",
      "tier": {
        "id": "uuid",
        "name": "Pro",
        "priceAnnual": 490
      }
    }
  }
```

#### Get Subscription
```yaml
GET /v1/subscriptions/{id}

Response: 200 OK
  {
    "success": true,
    "data": {
      "id": "uuid",
      "organizationId": "uuid",
      "tier": { ... },
      "status": "active",
      "billingCycle": "annual",
      "currentPeriodStart": "2025-11-29T00:00:00Z",
      "currentPeriodEnd": "2026-11-29T00:00:00Z",
      "priceAmount": 490,
      "nextInvoiceDate": "2026-11-29",
      "paymentMethod": {
        "type": "card",
        "card": { "brand": "visa", "last4": "4242" }
      }
    }
  }
```

#### Change Plan
```yaml
POST /v1/subscriptions/{id}/change-plan

Request:
  {
    "tierId": "uuid",
    "billingCycle": "annual",
    "prorationBehavior": "create_prorations"  // or "none", "always_invoice"
  }

Response: 200 OK
  {
    "success": true,
    "data": {
      "subscription": { ... },
      "proration": {
        "creditAmount": 25.00,
        "chargeAmount": 150.00,
        "netAmount": 125.00
      }
    }
  }
```

#### Cancel Subscription
```yaml
POST /v1/subscriptions/{id}/cancel

Request:
  {
    "cancelAtPeriodEnd": true,
    "reason": "too_expensive",
    "feedback": "Need lower tier option"
  }

Response: 200 OK
  {
    "success": true,
    "data": {
      "id": "uuid",
      "status": "active",
      "cancelAtPeriodEnd": true,
      "cancellationDate": "2026-11-29T00:00:00Z"
    }
  }
```

#### Reactivate Subscription
```yaml
POST /v1/subscriptions/{id}/reactivate

Response: 200 OK
```

### 3.2 Payment Method Endpoints

#### Add Payment Method
```yaml
POST /v1/payment-methods

Request:
  {
    "organizationId": "uuid",
    "stripePaymentMethodId": "pm_xxx",  // From Stripe.js
    "setAsDefault": true
  }

Response: 201 Created
```

#### List Payment Methods
```yaml
GET /v1/organizations/{orgId}/payment-methods

Response: 200 OK
```

#### Set Default Payment Method
```yaml
POST /v1/payment-methods/{id}/set-default

Response: 200 OK
```

#### Remove Payment Method
```yaml
DELETE /v1/payment-methods/{id}

Response: 204 No Content
```

### 3.3 Invoice Endpoints

#### List Invoices
```yaml
GET /v1/organizations/{orgId}/invoices

Query Parameters:
  status: string
  from: date
  to: date
  page: integer
  limit: integer

Response: 200 OK
  {
    "success": true,
    "data": [
      {
        "id": "uuid",
        "invoiceNumber": "INV-2025-0001",
        "status": "paid",
        "total": 490.00,
        "invoiceDate": "2025-11-29",
        "paidAt": "2025-11-29T10:00:00Z",
        "pdfUrl": "https://..."
      }
    ],
    "meta": { ... }
  }
```

#### Get Invoice
```yaml
GET /v1/invoices/{id}

Response: 200 OK
```

#### Download Invoice PDF
```yaml
GET /v1/invoices/{id}/pdf

Response: 302 Redirect to PDF URL
```

#### Pay Invoice (Manual)
```yaml
POST /v1/invoices/{id}/pay

Request:
  {
    "paymentMethodId": "uuid"
  }

Response: 200 OK
```

### 3.4 Webhook Endpoint

```yaml
POST /v1/webhooks/stripe

Headers:
  Stripe-Signature: t=xxx,v1=xxx

Handled Events:
  - customer.subscription.created
  - customer.subscription.updated
  - customer.subscription.deleted
  - invoice.created
  - invoice.paid
  - invoice.payment_failed
  - payment_intent.succeeded
  - payment_intent.payment_failed
  - customer.updated

Response: 200 OK (always, to prevent retries on handled events)
```

---

## 4. Event Contracts

### 4.1 Published Events

```typescript
interface SubscriptionCreatedEvent {
  eventType: 'subscription.created';
  subscriptionId: string;
  organizationId: string;
  tierId: string;
  tierName: string;
  status: SubscriptionStatus;
  trialEnd?: string;
}

interface SubscriptionStatusChangedEvent {
  eventType: 'subscription.status_changed';
  subscriptionId: string;
  organizationId: string;
  previousStatus: SubscriptionStatus;
  newStatus: SubscriptionStatus;
  reason?: string;
}

interface SubscriptionPlanChangedEvent {
  eventType: 'subscription.plan_changed';
  subscriptionId: string;
  organizationId: string;
  previousTierId: string;
  newTierId: string;
  changeType: 'upgrade' | 'downgrade';
  effectiveAt: string;
}

interface SubscriptionCancelledEvent {
  eventType: 'subscription.cancelled';
  subscriptionId: string;
  organizationId: string;
  reason?: string;
  effectiveAt: string;
  immediate: boolean;
}

interface InvoicePaidEvent {
  eventType: 'invoice.paid';
  invoiceId: string;
  organizationId: string;
  amount: number;
  currency: string;
}

interface PaymentFailedEvent {
  eventType: 'payment.failed';
  organizationId: string;
  invoiceId: string;
  amount: number;
  failureReason: string;
  attemptCount: number;
  nextRetryAt?: string;
}
```

---

## 5. Stripe Integration

### 5.1 Stripe Object Mapping

```typescript
// Sync strategy: Stripe as source of truth for payment state
// Our DB stores denormalized data for queries

async function syncFromStripe(stripeSubscription: Stripe.Subscription): Promise<void> {
  await db.subscriptions.upsert({
    where: { stripeSubscriptionId: stripeSubscription.id },
    update: {
      status: mapStripeStatus(stripeSubscription.status),
      currentPeriodStart: new Date(stripeSubscription.current_period_start * 1000),
      currentPeriodEnd: new Date(stripeSubscription.current_period_end * 1000),
      cancelAtPeriodEnd: stripeSubscription.cancel_at_period_end
    }
  });
}

function mapStripeStatus(status: string): SubscriptionStatus {
  const mapping: Record<string, SubscriptionStatus> = {
    'trialing': 'trialing',
    'active': 'active',
    'past_due': 'past_due',
    'canceled': 'cancelled',
    'unpaid': 'past_due',
    'incomplete': 'incomplete',
    'incomplete_expired': 'incomplete_expired',
    'paused': 'paused'
  };
  return mapping[status] || 'active';
}
```

### 5.2 Dunning Configuration

```typescript
const DUNNING_CONFIG = {
  retrySchedule: [1, 3, 7, 14],  // Days after failure
  gracePeriodDays: 14,
  suspendAfterDays: 14,
  cancelAfterDays: 30,
  
  notifications: [
    { day: 0, template: 'payment_failed_initial' },
    { day: 3, template: 'payment_failed_reminder' },
    { day: 7, template: 'payment_failed_urgent' },
    { day: 14, template: 'account_suspension_warning' }
  ]
};
```

---

## 6. Security Considerations

### 6.1 PCI Compliance
- Never store full card numbers
- Use Stripe.js for card collection
- Only store Stripe tokens/IDs

### 6.2 Webhook Security
- Verify Stripe signatures
- Idempotency via event ID
- Process webhooks asynchronously

### 6.3 Access Control
- Billing admin role required
- Audit all payment actions
- Rate limit payment attempts

---

*FlowMind Technologies | Subscription & Billing Service Technical Specification v1.0*

