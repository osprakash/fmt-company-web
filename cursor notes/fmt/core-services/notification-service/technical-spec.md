# Notification Service - Technical Specification

> **Service**: Notification Service  
> **Version**: 1.0  
> **Owner**: Platform Team  
> **Last Updated**: November 2025

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Notification Service                                │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────────────────┐  │
│  │ Notification   │  │   Template     │  │   Preference             │  │
│  │   Router       │  │   Engine       │  │   Manager                │  │
│  └────────────────┘  └────────────────┘  └──────────────────────────┘  │
│                                                                          │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────────────────┐  │
│  │ Email Sender   │  │  SMS Sender    │  │   Push Sender            │  │
│  │ (SendGrid)     │  │  (Twilio)      │  │   (FCM/APNs)             │  │
│  └────────────────┘  └────────────────┘  └──────────────────────────┘  │
│                                                                          │
│  ┌────────────────┐  ┌────────────────┐                                 │
│  │ In-App Handler │  │ Webhook        │                                 │
│  │ (WebSocket)    │  │ Dispatcher     │                                 │
│  └────────────────┘  └────────────────┘                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Data Model

### 2.1 Database Schema

```sql
-- Notification Templates
CREATE TABLE notification_templates (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(100) NOT NULL UNIQUE,
    category VARCHAR(50) NOT NULL,
    
    -- Content by channel
    email_subject VARCHAR(255),
    email_html TEXT,
    email_text TEXT,
    sms_body TEXT,
    push_title VARCHAR(100),
    push_body TEXT,
    in_app_title VARCHAR(100),
    in_app_body TEXT,
    
    -- Config
    channels VARCHAR(20)[] NOT NULL DEFAULT ARRAY['email'],
    priority VARCHAR(20) NOT NULL DEFAULT 'normal',
    
    -- i18n
    translations JSONB NOT NULL DEFAULT '{}',
    
    version INTEGER NOT NULL DEFAULT 1,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Notifications (sent/pending)
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL,
    
    template_id UUID REFERENCES notification_templates(id),
    template_name VARCHAR(100) NOT NULL,
    
    recipient_user_id UUID,
    recipient_email VARCHAR(255),
    recipient_phone VARCHAR(20),
    
    channel VARCHAR(20) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    
    -- Content (rendered)
    subject VARCHAR(255),
    body TEXT NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    
    -- Delivery
    provider VARCHAR(50),
    provider_message_id VARCHAR(100),
    delivered_at TIMESTAMP WITH TIME ZONE,
    opened_at TIMESTAMP WITH TIME ZONE,
    clicked_at TIMESTAMP WITH TIME ZONE,
    
    -- Errors
    error_message TEXT,
    retry_count INTEGER NOT NULL DEFAULT 0,
    next_retry_at TIMESTAMP WITH TIME ZONE,
    
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_tenant ON notifications(tenant_id);
CREATE INDEX idx_notifications_user ON notifications(recipient_user_id);
CREATE INDEX idx_notifications_status ON notifications(status);

-- In-App Notifications
CREATE TABLE in_app_notifications (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL,
    user_id UUID NOT NULL,
    
    title VARCHAR(255) NOT NULL,
    body TEXT NOT NULL,
    icon VARCHAR(50),
    action_url VARCHAR(500),
    
    category VARCHAR(50) NOT NULL,
    priority VARCHAR(20) NOT NULL DEFAULT 'normal',
    
    read_at TIMESTAMP WITH TIME ZONE,
    archived_at TIMESTAMP WITH TIME ZONE,
    
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_in_app_user ON in_app_notifications(user_id, created_at DESC);
CREATE INDEX idx_in_app_unread ON in_app_notifications(user_id) WHERE read_at IS NULL;

-- User Notification Preferences
CREATE TABLE notification_preferences (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    
    category VARCHAR(50) NOT NULL,
    
    email_enabled BOOLEAN NOT NULL DEFAULT TRUE,
    sms_enabled BOOLEAN NOT NULL DEFAULT FALSE,
    push_enabled BOOLEAN NOT NULL DEFAULT TRUE,
    in_app_enabled BOOLEAN NOT NULL DEFAULT TRUE,
    
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT preferences_unique UNIQUE (user_id, tenant_id, category)
);

-- Quiet Hours
CREATE TABLE notification_quiet_hours (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL,
    
    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    timezone VARCHAR(50) NOT NULL,
    days_of_week INTEGER[] NOT NULL DEFAULT ARRAY[0,1,2,3,4,5,6],
    
    enabled BOOLEAN NOT NULL DEFAULT TRUE,
    
    CONSTRAINT quiet_hours_unique UNIQUE (user_id)
);

-- Webhook Endpoints
CREATE TABLE webhook_endpoints (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL,
    organization_id UUID NOT NULL,
    
    url VARCHAR(500) NOT NULL,
    secret VARCHAR(255) NOT NULL,  -- For signature
    
    events VARCHAR(100)[] NOT NULL,  -- Events to send
    
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    
    last_delivery_at TIMESTAMP WITH TIME ZONE,
    failure_count INTEGER NOT NULL DEFAULT 0,
    
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Webhook Deliveries
CREATE TABLE webhook_deliveries (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    endpoint_id UUID NOT NULL REFERENCES webhook_endpoints(id),
    
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    response_status INTEGER,
    response_body TEXT,
    
    attempt_count INTEGER NOT NULL DEFAULT 0,
    next_retry_at TIMESTAMP WITH TIME ZONE,
    
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    delivered_at TIMESTAMP WITH TIME ZONE
);
```

### 2.2 TypeScript Interfaces

```typescript
interface Notification {
  id: string;
  tenantId: string;
  templateName: string;
  recipientUserId?: string;
  recipientEmail?: string;
  recipientPhone?: string;
  channel: NotificationChannel;
  status: NotificationStatus;
  subject?: string;
  body: string;
  metadata: Record<string, unknown>;
  deliveredAt?: Date;
  openedAt?: Date;
}

type NotificationChannel = 'email' | 'sms' | 'push' | 'in_app' | 'webhook';
type NotificationStatus = 'pending' | 'sent' | 'delivered' | 'failed' | 'bounced';

interface NotificationTemplate {
  id: string;
  name: string;
  category: string;
  emailSubject?: string;
  emailHtml?: string;
  emailText?: string;
  smsBody?: string;
  pushTitle?: string;
  pushBody?: string;
  channels: NotificationChannel[];
  priority: 'low' | 'normal' | 'high' | 'urgent';
  translations: Record<string, TranslationContent>;
}

interface SendNotificationRequest {
  templateName: string;
  recipient: {
    userId?: string;
    email?: string;
    phone?: string;
  };
  variables: Record<string, unknown>;
  channels?: NotificationChannel[];
  scheduledAt?: Date;
}
```

---

## 3. API Contracts

### 3.1 Send Notification

```yaml
POST /v1/notifications/send

Request:
  {
    "templateName": "welcome_email",
    "recipient": {
      "userId": "uuid",
      "email": "user@example.com"
    },
    "variables": {
      "userName": "John",
      "productName": "FlowMind Analytics"
    },
    "channels": ["email", "in_app"]
  }

Response: 202 Accepted
  {
    "success": true,
    "data": {
      "notificationIds": ["uuid-1", "uuid-2"],
      "status": "queued"
    }
  }
```

### 3.2 In-App Notifications

```yaml
GET /v1/notifications/in-app

Query Parameters:
  unreadOnly: boolean
  limit: integer
  cursor: string

Response: 200 OK
  {
    "success": true,
    "data": {
      "notifications": [
        {
          "id": "uuid",
          "title": "New team member",
          "body": "John Doe joined your team",
          "icon": "user-plus",
          "actionUrl": "/settings/team",
          "category": "team",
          "readAt": null,
          "createdAt": "2025-11-29T10:00:00Z"
        }
      ],
      "unreadCount": 5,
      "cursor": "xxx"
    }
  }
```

#### Mark as Read
```yaml
POST /v1/notifications/in-app/{id}/read

Response: 200 OK
```

#### Mark All as Read
```yaml
POST /v1/notifications/in-app/read-all

Response: 200 OK
```

### 3.3 Preferences

```yaml
GET /v1/notifications/preferences

Response: 200 OK
  {
    "success": true,
    "data": {
      "categories": {
        "system": { "email": true, "push": true, "in_app": true, "editable": false },
        "product_updates": { "email": true, "push": true, "in_app": true, "editable": true },
        "marketing": { "email": false, "push": false, "in_app": false, "editable": true }
      },
      "quietHours": {
        "enabled": true,
        "startTime": "22:00",
        "endTime": "08:00",
        "timezone": "America/Los_Angeles"
      }
    }
  }
```

```yaml
PATCH /v1/notifications/preferences

Request:
  {
    "categories": {
      "marketing": { "email": false }
    },
    "quietHours": {
      "enabled": true,
      "startTime": "23:00",
      "endTime": "07:00"
    }
  }

Response: 200 OK
```

### 3.4 Webhooks

```yaml
POST /v1/webhooks

Request:
  {
    "url": "https://api.customer.com/webhooks",
    "events": ["subscription.created", "invoice.paid"]
  }

Response: 201 Created
  {
    "success": true,
    "data": {
      "id": "uuid",
      "url": "https://api.customer.com/webhooks",
      "secret": "whsec_xxx",  // Only shown once
      "events": ["subscription.created", "invoice.paid"]
    }
  }
```

---

## 4. Event Processing

```typescript
// Notification router
async function sendNotification(request: SendNotificationRequest): Promise<void> {
  const template = await getTemplate(request.templateName);
  const user = request.recipient.userId 
    ? await getUser(request.recipient.userId) 
    : null;
  
  // Check preferences
  const preferences = user 
    ? await getPreferences(user.id, template.category)
    : getDefaultPreferences();
  
  // Check quiet hours
  const isQuietHours = user 
    ? await checkQuietHours(user.id)
    : false;
  
  const channels = request.channels || template.channels;
  
  for (const channel of channels) {
    if (!preferences[channel]) continue;
    if (isQuietHours && template.priority !== 'urgent') continue;
    
    const rendered = await renderTemplate(template, channel, request.variables);
    
    await queue.publish(`notification.${channel}`, {
      ...rendered,
      recipient: request.recipient
    });
  }
}
```

---

## 5. Provider Integrations

### 5.1 Email (SendGrid)
```typescript
async function sendEmail(notification: EmailNotification): Promise<void> {
  await sendgrid.send({
    to: notification.recipientEmail,
    from: 'noreply@flowmind.io',
    subject: notification.subject,
    html: notification.htmlBody,
    text: notification.textBody,
    trackingSettings: {
      openTracking: { enable: true },
      clickTracking: { enable: true }
    }
  });
}
```

### 5.2 SMS (Twilio)
```typescript
async function sendSms(notification: SmsNotification): Promise<void> {
  await twilio.messages.create({
    to: notification.recipientPhone,
    from: TWILIO_PHONE_NUMBER,
    body: notification.body
  });
}
```

### 5.3 Push (Firebase)
```typescript
async function sendPush(notification: PushNotification): Promise<void> {
  await firebase.messaging().send({
    token: notification.deviceToken,
    notification: {
      title: notification.title,
      body: notification.body
    },
    data: notification.data
  });
}
```

---

*FlowMind Technologies | Notification Service Technical Specification v1.0*

