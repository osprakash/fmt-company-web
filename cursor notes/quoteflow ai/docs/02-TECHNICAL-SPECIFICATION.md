# QuoteFlow AI - Technical Specification Document

**Version:** 1.0  
**Date:** February 23, 2026  
**Status:** Draft  

---

## 1. System Overview

### 1.1 Architecture Philosophy

QuoteFlow AI follows a **Chat-to-UI Pipeline** architecture where the LLM acts as an intelligent parser/generator that outputs structured data, which then populates an editable UI form with live PDF preview. Both the AI Agent and the React UI consume the same backend APIs. This ensures:

- **Consistency**: AI and UI always produce identical results
- **Determinism**: AI actions are predictable and auditable
- **Maintainability**: Single source of truth for business logic
- **User Control**: Chat accelerates creation, UI enables precision editing
- **Error Recovery**: LLM mistakes are easily corrected in the form

### 1.2 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           CLIENT LAYER                                   │
├─────────────────┬─────────────────┬─────────────────┬───────────────────┤
│   React Web UI  │  WhatsApp Bot   │  Telegram Bot   │   Mobile App      │
│   (SPA)         │  (Webhook)      │  (Webhook)      │   (React Native)  │
└────────┬────────┴────────┬────────┴────────┬────────┴─────────┬─────────┘
         │                 │                 │                   │
         ▼                 ▼                 ▼                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         API GATEWAY (Express.js)                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │ Auth        │  │ Rate        │  │ Request     │  │ Response    │    │
│  │ Middleware  │  │ Limiting    │  │ Validation  │  │ Transform   │    │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘    │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         ▼                       ▼                       ▼
┌─────────────────┐  ┌─────────────────────┐  ┌─────────────────────────┐
│  REST API       │  │  Mastra AI Agent    │  │  MCP Server             │
│  Controllers    │  │  (Orchestration)    │  │  (Tool Definitions)     │
│                 │  │                     │  │                         │
│  - Quotes       │  │  - Chat Handler     │  │  - quote_create         │
│  - Products     │  │  - Context Manager  │  │  - quote_update         │
│  - Contacts     │  │  - Tool Executor    │  │  - product_search       │
│  - Users        │  │  - Memory Store     │  │  - contact_lookup       │
│  - Templates    │  │                     │  │  - pdf_generate         │
└────────┬────────┘  └──────────┬──────────┘  └───────────┬─────────────┘
         │                      │                         │
         └──────────────────────┼─────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        SERVICE LAYER                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │ Quote        │  │ Product      │  │ Contact      │  │ PDF         │ │
│  │ Service      │  │ Service      │  │ Service      │  │ Service     │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └─────────────┘ │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │ User         │  │ Template     │  │ Tax          │  │ Notification│ │
│  │ Service      │  │ Service      │  │ Service      │  │ Service     │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └─────────────┘ │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
┌────────────────────────────────┼────────────────────────────────────────┐
│                        DATA LAYER                                        │
│  ┌──────────────────────┐  ┌──────────────────────┐                     │
│  │     PostgreSQL       │  │       Redis          │                     │
│  │  (Primary Database)  │  │  (Cache & Sessions)  │                     │
│  └──────────────────────┘  └──────────────────────┘                     │
│  ┌──────────────────────┐  ┌──────────────────────┐                     │
│  │        S3            │  │   Vector Store       │                     │
│  │  (File Storage)      │  │  (RAG/Embeddings)    │                     │
│  └──────────────────────┘  └──────────────────────┘                     │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Chat-to-UI Pipeline Architecture

The core user experience follows a **Chat → Structured Output → Editable UI → Live Preview** pipeline:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      CHAT-TO-UI PIPELINE FLOW                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                        CHAT PANEL                                │    │
│  │  User: "Create a quote for John at ABC Corp for 5 hours         │    │
│  │         consulting and 2 logo designs"                           │    │
│  └──────────────────────────┬──────────────────────────────────────┘    │
│                             │                                            │
│                             ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                     LLM PROCESSING                               │    │
│  │  1. Parse natural language intent                                │    │
│  │  2. Look up products from catalog (MCP tool call)                │    │
│  │  3. Look up/create contact (MCP tool call)                       │    │
│  │  4. Generate structured JSON output                              │    │
│  └──────────────────────────┬──────────────────────────────────────┘    │
│                             │                                            │
│                             ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                 STRUCTURED OUTPUT (JSON)                         │    │
│  │  {                                                               │    │
│  │    "customer": { "name": "John", "company": "ABC Corp" },        │    │
│  │    "lineItems": [                                                │    │
│  │      { "description": "Consulting", "qty": 5, "rate": 150 },     │    │
│  │      { "description": "Logo Design", "qty": 2, "rate": 500 }     │    │
│  │    ],                                                            │    │
│  │    "currency": "USD", "validDays": 30                            │    │
│  │  }                                                               │    │
│  └──────────────────────────┬──────────────────────────────────────┘    │
│                             │                                            │
│           ┌─────────────────┴─────────────────┐                         │
│           ▼                                   ▼                         │
│  ┌─────────────────────────┐    ┌─────────────────────────────────┐    │
│  │   EDITABLE FORM UI      │    │      LIVE PDF PREVIEW           │    │
│  │  ┌───────────────────┐  │    │  ┌───────────────────────────┐  │    │
│  │  │ Customer: [John▼] │  │◄──►│  │  [Logo]                   │  │    │
│  │  │ Company: [ABC Co] │  │    │  │  QUOTATION #Q-2026-001    │  │    │
│  │  ├───────────────────┤  │    │  │  To: John, ABC Corp       │  │    │
│  │  │ LINE ITEMS        │  │    │  │  ─────────────────────    │  │    │
│  │  │ Consulting 5h $750│  │    │  │  Consulting  5hr   $750   │  │    │
│  │  │ Logo      2x $1000│  │    │  │  Logo Design 2x    $1000  │  │    │
│  │  │ [+ Add Item]      │  │    │  │  ─────────────────────    │  │    │
│  │  ├───────────────────┤  │    │  │  Total: $1,925            │  │    │
│  │  │ Total: $1,925     │  │    │  └───────────────────────────┘  │    │
│  │  └───────────────────┘  │    └─────────────────────────────────┘    │
│  └─────────────────────────┘                                            │
│           │                                                              │
│           │  Real-time bidirectional sync                               │
│           │  Form changes → Preview updates instantly (<100ms)          │
│           │  Chat commands → Form + Preview update                      │
│           ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                      ACTIONS                                     │    │
│  │  [Send via Email]  [Share Link]  [WhatsApp]  [Download PDF]     │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key Implementation Details:**

| Component | Technology | Purpose |
|-----------|------------|---------|
| Chat Panel | React + WebSocket | Real-time conversation |
| LLM Processing | Mastra AI + OpenRouter | Intent parsing, structured output |
| Structured Output | Zod Schema Validation | Type-safe JSON contract |
| Editable Form | React Hook Form + Zustand | State management, validation |
| Live Preview | react-pdf / @react-pdf/renderer | Client-side PDF rendering |
| Final PDF | Puppeteer (server-side) | Pixel-perfect export |

**Data Flow:**
1. User types in chat → WebSocket to backend
2. Backend processes via Mastra AI Agent → Returns structured JSON
3. Frontend receives JSON → Populates form state (Zustand)
4. Form state changes → Triggers live preview re-render
5. User can edit form directly OR continue chatting
6. On "Send" → Server generates final PDF, stores, and distributes

---

## 2. Technology Stack

### 2.1 Core Technologies

| Layer | Technology | Version | Purpose |
|-------|------------|---------|---------|
| **Runtime** | Node.js | 20 LTS | Server runtime |
| **Language** | TypeScript | 5.x | Type safety |
| **Framework** | Express.js | 4.x | HTTP server |
| **Database** | PostgreSQL | 16 | Primary data store |
| **Cache** | Redis | 7.x | Caching, sessions, queues |
| **AI Framework** | Mastra.ai | Latest | Agent orchestration |
| **LLM Router** | OpenRouter | - | Multi-model access |
| **Frontend** | React | 18.x | Web UI |
| **State Management** | Zustand | 4.x | Client state |
| **Styling** | Tailwind CSS | 3.x | UI styling |
| **PDF Generation** | Puppeteer / React-PDF | - | PDF creation |

### 2.2 Infrastructure

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Hosting** | AWS / Vercel | Application hosting |
| **CDN** | CloudFlare | Static assets, DDoS protection |
| **File Storage** | AWS S3 | PDFs, logos, uploads |
| **Email** | SendGrid / Resend | Transactional email |
| **SMS** | Twilio | SMS notifications |
| **Monitoring** | Datadog / Sentry | APM, error tracking |
| **CI/CD** | GitHub Actions | Automated deployment |

### 2.3 Third-Party Integrations

| Service | Purpose |
|---------|---------|
| **WhatsApp Business API** | Messaging channel |
| **Telegram Bot API** | Messaging channel |
| **Stripe** | Payment processing |
| **OpenRouter** | LLM API gateway |

---

## 3. Database Schema

### 3.1 Entity Relationship Diagram

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│   tenants    │       │    users     │       │   contacts   │
├──────────────┤       ├──────────────┤       ├──────────────┤
│ id (PK)      │◄──────│ tenant_id(FK)│       │ id (PK)      │
│ name         │       │ id (PK)      │       │ tenant_id(FK)│──┐
│ slug         │       │ email        │       │ name         │  │
│ settings     │       │ password     │       │ email        │  │
│ subscription │       │ role         │       │ phone        │  │
│ created_at   │       │ created_at   │       │ company      │  │
└──────────────┘       └──────────────┘       │ address      │  │
                                              │ metadata     │  │
                                              └──────────────┘  │
                                                     │          │
┌──────────────┐       ┌──────────────┐              │          │
│  products    │       │   quotes     │◄─────────────┘          │
├──────────────┤       ├──────────────┤                         │
│ id (PK)      │       │ id (PK)      │◄────────────────────────┘
│ tenant_id(FK)│       │ tenant_id(FK)│
│ name         │       │ contact_id   │       ┌──────────────┐
│ description  │       │ quote_number │       │ quote_items  │
│ sku          │       │ status       │       ├──────────────┤
│ price        │       │ subtotal     │       │ id (PK)      │
│ cost         │       │ tax_amount   │       │ quote_id(FK) │──┐
│ uom          │       │ total        │       │ product_id   │  │
│ category     │       │ currency     │       │ description  │  │
│ tax_rate     │       │ valid_until  │       │ quantity     │  │
│ active       │       │ notes        │       │ unit_price   │  │
└──────────────┘       │ terms        │       │ discount     │  │
                       │ pdf_url      │       │ tax_rate     │  │
                       │ created_at   │       │ line_total   │  │
                       │ sent_at      │       └──────────────┘  │
                       │ viewed_at    │              │          │
                       └──────────────┘◄─────────────┘          │
                              │                                 │
                              ▼                                 │
                       ┌──────────────┐                         │
                       │  templates   │                         │
                       ├──────────────┤                         │
                       │ id (PK)      │                         │
                       │ tenant_id(FK)│                         │
                       │ name         │                         │
                       │ html_content │                         │
                       │ css_styles   │                         │
                       │ is_default   │                         │
                       └──────────────┘                         │
                                                                │
┌──────────────┐       ┌──────────────┐                         │
│    leads     │       │conversations │                         │
├──────────────┤       ├──────────────┤                         │
│ id (PK)      │       │ id (PK)      │                         │
│ tenant_id(FK)│       │ tenant_id(FK)│                         │
│ contact_id   │───────│ user_id      │                         │
│ status       │       │ channel      │                         │
│ source       │       │ external_id  │                         │
│ value        │       │ context      │                         │
│ notes        │       │ created_at   │                         │
│ created_at   │       └──────────────┘                         │
└──────────────┘              │                                 │
                              ▼                                 │
                       ┌──────────────┐                         │
                       │  messages    │                         │
                       ├──────────────┤                         │
                       │ id (PK)      │                         │
                       │ conv_id (FK) │                         │
                       │ role         │                         │
                       │ content      │                         │
                       │ tool_calls   │                         │
                       │ created_at   │                         │
                       └──────────────┘                         │
```

### 3.2 Core Tables DDL

```sql
-- Tenants (Multi-tenancy support)
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    settings JSONB DEFAULT '{}',
    subscription_tier VARCHAR(50) DEFAULT 'free',
    subscription_status VARCHAR(50) DEFAULT 'active',
    stripe_customer_id VARCHAR(255),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(255),
    role VARCHAR(50) DEFAULT 'member',
    avatar_url VARCHAR(500),
    preferences JSONB DEFAULT '{}',
    email_verified_at TIMESTAMPTZ,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(tenant_id, email)
);

-- Products/Services
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    sku VARCHAR(100),
    price DECIMAL(15, 4) NOT NULL,
    cost DECIMAL(15, 4),
    currency VARCHAR(3) DEFAULT 'USD',
    uom VARCHAR(50) DEFAULT 'unit',
    category VARCHAR(100),
    tax_rate DECIMAL(5, 2) DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(tenant_id, sku)
);

-- Contacts
CREATE TABLE contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255),
    phone VARCHAR(50),
    company VARCHAR(255),
    address JSONB DEFAULT '{}',
    tags TEXT[] DEFAULT '{}',
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Quotes
CREATE TABLE quotes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    contact_id UUID REFERENCES contacts(id),
    quote_number VARCHAR(50) NOT NULL,
    status VARCHAR(50) DEFAULT 'draft',
    subtotal DECIMAL(15, 4) DEFAULT 0,
    discount_amount DECIMAL(15, 4) DEFAULT 0,
    tax_amount DECIMAL(15, 4) DEFAULT 0,
    total DECIMAL(15, 4) DEFAULT 0,
    currency VARCHAR(3) DEFAULT 'USD',
    valid_until DATE,
    notes TEXT,
    terms TEXT,
    internal_notes TEXT,
    template_id UUID,
    pdf_url VARCHAR(500),
    share_token VARCHAR(100) UNIQUE,
    created_by UUID REFERENCES users(id),
    sent_at TIMESTAMPTZ,
    viewed_at TIMESTAMPTZ,
    accepted_at TIMESTAMPTZ,
    rejected_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(tenant_id, quote_number)
);

-- Quote Line Items
CREATE TABLE quote_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    quote_id UUID REFERENCES quotes(id) ON DELETE CASCADE,
    product_id UUID REFERENCES products(id),
    description VARCHAR(500) NOT NULL,
    quantity DECIMAL(15, 4) DEFAULT 1,
    uom VARCHAR(50) DEFAULT 'unit',
    unit_price DECIMAL(15, 4) NOT NULL,
    discount_percent DECIMAL(5, 2) DEFAULT 0,
    tax_rate DECIMAL(5, 2) DEFAULT 0,
    line_total DECIMAL(15, 4) NOT NULL,
    sort_order INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Templates
CREATE TABLE templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    html_content TEXT NOT NULL,
    css_styles TEXT,
    thumbnail_url VARCHAR(500),
    is_default BOOLEAN DEFAULT false,
    is_system BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Leads (CRM)
CREATE TABLE leads (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    contact_id UUID REFERENCES contacts(id),
    status VARCHAR(50) DEFAULT 'new',
    source VARCHAR(100),
    estimated_value DECIMAL(15, 4),
    notes TEXT,
    assigned_to UUID REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Conversations (Chat History)
CREATE TABLE conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id),
    channel VARCHAR(50) DEFAULT 'web',
    external_id VARCHAR(255),
    context JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Messages
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID REFERENCES conversations(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL,
    content TEXT NOT NULL,
    tool_calls JSONB,
    tokens_used INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_products_tenant ON products(tenant_id);
CREATE INDEX idx_products_category ON products(tenant_id, category);
CREATE INDEX idx_contacts_tenant ON contacts(tenant_id);
CREATE INDEX idx_contacts_email ON contacts(tenant_id, email);
CREATE INDEX idx_quotes_tenant ON quotes(tenant_id);
CREATE INDEX idx_quotes_status ON quotes(tenant_id, status);
CREATE INDEX idx_quotes_contact ON quotes(contact_id);
CREATE INDEX idx_quote_items_quote ON quote_items(quote_id);
CREATE INDEX idx_messages_conversation ON messages(conversation_id);
CREATE INDEX idx_leads_tenant_status ON leads(tenant_id, status);
```

---

## 4. API Design

### 4.1 API Structure

```
/api/v1
├── /auth
│   ├── POST   /register
│   ├── POST   /login
│   ├── POST   /logout
│   ├── POST   /refresh
│   └── POST   /forgot-password
│
├── /users
│   ├── GET    /me
│   ├── PATCH  /me
│   └── PUT    /me/password
│
├── /products
│   ├── GET    /                    # List products
│   ├── POST   /                    # Create product
│   ├── GET    /:id                 # Get product
│   ├── PATCH  /:id                 # Update product
│   ├── DELETE /:id                 # Delete product
│   └── POST   /import              # Bulk import
│
├── /contacts
│   ├── GET    /                    # List contacts
│   ├── POST   /                    # Create contact
│   ├── GET    /:id                 # Get contact
│   ├── PATCH  /:id                 # Update contact
│   ├── DELETE /:id                 # Delete contact
│   └── GET    /:id/quotes          # Contact's quotes
│
├── /quotes
│   ├── GET    /                    # List quotes
│   ├── POST   /                    # Create quote
│   ├── GET    /:id                 # Get quote
│   ├── PATCH  /:id                 # Update quote
│   ├── DELETE /:id                 # Delete quote
│   ├── POST   /:id/items           # Add line item
│   ├── PATCH  /:id/items/:itemId   # Update line item
│   ├── DELETE /:id/items/:itemId   # Remove line item
│   ├── POST   /:id/send            # Send quote
│   ├── POST   /:id/duplicate       # Clone quote
│   └── GET    /:id/pdf             # Generate PDF
│
├── /templates
│   ├── GET    /                    # List templates
│   ├── POST   /                    # Create template
│   ├── GET    /:id                 # Get template
│   ├── PATCH  /:id                 # Update template
│   └── DELETE /:id                 # Delete template
│
├── /leads
│   ├── GET    /                    # List leads
│   ├── POST   /                    # Create lead
│   ├── GET    /:id                 # Get lead
│   ├── PATCH  /:id                 # Update lead
│   └── DELETE /:id                 # Delete lead
│
├── /chat
│   ├── POST   /message             # Send chat message
│   ├── GET    /conversations       # List conversations
│   └── GET    /conversations/:id   # Get conversation
│
├── /webhooks
│   ├── POST   /whatsapp            # WhatsApp webhook
│   └── POST   /telegram            # Telegram webhook
│
└── /public
    └── GET    /quotes/:shareToken  # Public quote view
```

### 4.2 API Request/Response Examples

#### Create Quote

```typescript
// POST /api/v1/quotes
// Request
{
  "contact_id": "uuid",
  "valid_until": "2026-03-23",
  "currency": "USD",
  "notes": "Thank you for your business!",
  "items": [
    {
      "product_id": "uuid",
      "quantity": 5,
      "discount_percent": 10
    },
    {
      "description": "Custom consulting work",
      "quantity": 8,
      "uom": "hours",
      "unit_price": 150,
      "tax_rate": 10
    }
  ]
}

// Response
{
  "success": true,
  "data": {
    "id": "uuid",
    "quote_number": "QT-2026-0001",
    "status": "draft",
    "contact": {
      "id": "uuid",
      "name": "John Smith",
      "company": "ABC Corp"
    },
    "items": [...],
    "subtotal": 1950.00,
    "discount_amount": 75.00,
    "tax_amount": 187.50,
    "total": 2062.50,
    "currency": "USD",
    "valid_until": "2026-03-23",
    "created_at": "2026-02-23T10:30:00Z"
  }
}
```

#### Chat Message (Returns Structured Output for UI)

```typescript
// POST /api/v1/chat/message
// Request
{
  "conversation_id": "uuid", // optional, creates new if not provided
  "message": "Create a quote for ABC Corp for 10 hours of consulting"
}

// Response - includes structured_output for UI rendering
{
  "success": true,
  "data": {
    "conversation_id": "uuid",
    "response": "I've created a draft quote for ABC Corp. You can review and edit it in the form, or tell me what to change.",
    "actions_taken": [
      {
        "tool": "quote_create",
        "result": { "quote_id": "uuid", "quote_number": "QT-2026-0002" }
      }
    ],
    // STRUCTURED OUTPUT - populates the editable form UI
    "structured_output": {
      "type": "quote_draft",
      "data": {
        "id": "uuid",
        "quote_number": "QT-2026-0002",
        "customer": {
          "id": "uuid-or-null",
          "name": "ABC Corp",
          "company": "ABC Corp",
          "email": null
        },
        "line_items": [
          {
            "id": "temp-uuid",
            "product_id": "uuid-or-null",
            "description": "Consulting Services",
            "quantity": 10,
            "uom": "hour",
            "unit_price": 150.00,
            "discount_percent": 0,
            "tax_rate": 10,
            "line_total": 1500.00
          }
        ],
        "subtotal": 1500.00,
        "discount_amount": 0,
        "tax_amount": 150.00,
        "total": 1650.00,
        "currency": "USD",
        "valid_until": "2026-03-30",
        "notes": "",
        "terms": ""
      }
    },
    // Legacy format for backward compatibility
    "quote": {
      "id": "uuid",
      "quote_number": "QT-2026-0002",
      "total": 1650.00
    }
  }
}
```

**Structured Output Schema (Zod):**

```typescript
// src/schemas/chatOutput.ts
import { z } from 'zod';

export const QuoteDraftSchema = z.object({
  type: z.literal('quote_draft'),
  data: z.object({
    id: z.string().uuid().optional(),
    quote_number: z.string(),
    customer: z.object({
      id: z.string().uuid().nullable(),
      name: z.string(),
      company: z.string().optional(),
      email: z.string().email().nullable(),
    }),
    line_items: z.array(z.object({
      id: z.string(),
      product_id: z.string().uuid().nullable(),
      description: z.string(),
      quantity: z.number().positive(),
      uom: z.string().default('unit'),
      unit_price: z.number().nonnegative(),
      discount_percent: z.number().min(0).max(100).default(0),
      tax_rate: z.number().min(0).max(100).default(0),
      line_total: z.number(),
    })).min(1),
    subtotal: z.number(),
    discount_amount: z.number().default(0),
    tax_amount: z.number(),
    total: z.number(),
    currency: z.string().length(3).default('USD'),
    valid_until: z.string(), // ISO date
    notes: z.string().default(''),
    terms: z.string().default(''),
  }),
});

export type QuoteDraft = z.infer<typeof QuoteDraftSchema>;
```

---

## 5. Mastra AI Agent Architecture

### 5.1 Agent Configuration

```typescript
// src/agents/quoteAgent.ts
import { Agent } from '@mastra/core';
import { openrouter } from '@mastra/openrouter';

export const quoteAgent = new Agent({
  name: 'QuoteFlow Assistant',
  description: 'AI assistant for creating and managing quotations',
  
  model: openrouter({
    model: 'anthropic/claude-3.5-sonnet',
    temperature: 0.1, // Low temperature for deterministic outputs
  }),
  
  systemPrompt: `You are QuoteFlow Assistant, an AI that helps users create and manage professional quotations.

CRITICAL RULES:
1. ALWAYS use the provided tools to perform actions - never make up data
2. When creating quotes, ALWAYS look up products from the catalog first
3. ALWAYS confirm with the user before sending quotes
4. Format currency values according to the user's locale settings
5. If a product isn't found, ask the user if they want to add it

AVAILABLE CONTEXT:
- User's company: {{company_name}}
- Default currency: {{default_currency}}
- Tax rate: {{default_tax_rate}}%

When creating quotes:
1. First, identify or create the contact
2. Look up products/services from the catalog
3. Calculate totals including applicable taxes
4. Present a summary and ask for confirmation
5. Only send when explicitly requested`,

  tools: [
    contactLookupTool,
    contactCreateTool,
    productSearchTool,
    productCreateTool,
    quoteCreateTool,
    quoteUpdateTool,
    quoteItemAddTool,
    quoteSendTool,
    pdfGenerateTool,
  ],
  
  memory: {
    type: 'conversation',
    maxMessages: 50,
  },
});
```

### 5.2 MCP Tool Definitions

```typescript
// src/mcp/tools/quoteTools.ts
import { createTool } from '@mastra/core';
import { z } from 'zod';
import { quoteService } from '../../services/quoteService';

export const quoteCreateTool = createTool({
  name: 'quote_create',
  description: 'Create a new quotation for a customer',
  
  parameters: z.object({
    contact_id: z.string().uuid().describe('The contact/customer ID'),
    valid_days: z.number().default(30).describe('Quote validity in days'),
    currency: z.string().length(3).default('USD').describe('ISO currency code'),
    notes: z.string().optional().describe('Notes to include on quote'),
    items: z.array(z.object({
      product_id: z.string().uuid().optional(),
      description: z.string().describe('Line item description'),
      quantity: z.number().positive().describe('Quantity'),
      uom: z.string().default('unit').describe('Unit of measure'),
      unit_price: z.number().positive().describe('Price per unit'),
      discount_percent: z.number().min(0).max(100).default(0),
      tax_rate: z.number().min(0).max(100).optional(),
    })).min(1).describe('Quote line items'),
  }),
  
  execute: async ({ context, params }) => {
    const { tenantId, userId } = context;
    
    const quote = await quoteService.create({
      tenantId,
      createdBy: userId,
      ...params,
    });
    
    return {
      success: true,
      quote_id: quote.id,
      quote_number: quote.quoteNumber,
      total: quote.total,
      currency: quote.currency,
      share_url: `${process.env.APP_URL}/q/${quote.shareToken}`,
    };
  },
});

export const productSearchTool = createTool({
  name: 'product_search',
  description: 'Search for products/services in the catalog',
  
  parameters: z.object({
    query: z.string().describe('Search query (name, SKU, or description)'),
    category: z.string().optional().describe('Filter by category'),
    limit: z.number().default(10).describe('Max results to return'),
  }),
  
  execute: async ({ context, params }) => {
    const { tenantId } = context;
    
    const products = await productService.search({
      tenantId,
      ...params,
    });
    
    return {
      success: true,
      count: products.length,
      products: products.map(p => ({
        id: p.id,
        name: p.name,
        sku: p.sku,
        price: p.price,
        uom: p.uom,
        category: p.category,
      })),
    };
  },
});

export const contactLookupTool = createTool({
  name: 'contact_lookup',
  description: 'Find a contact by name, email, or company',
  
  parameters: z.object({
    query: z.string().describe('Search by name, email, or company'),
  }),
  
  execute: async ({ context, params }) => {
    const { tenantId } = context;
    
    const contacts = await contactService.search({
      tenantId,
      query: params.query,
      limit: 5,
    });
    
    if (contacts.length === 0) {
      return {
        success: true,
        found: false,
        message: 'No contacts found. Would you like to create a new contact?',
      };
    }
    
    return {
      success: true,
      found: true,
      contacts: contacts.map(c => ({
        id: c.id,
        name: c.name,
        email: c.email,
        company: c.company,
      })),
    };
  },
});

export const quoteSendTool = createTool({
  name: 'quote_send',
  description: 'Send a quote to the customer via email or generate share link',
  
  parameters: z.object({
    quote_id: z.string().uuid().describe('The quote ID to send'),
    method: z.enum(['email', 'link', 'whatsapp']).describe('Delivery method'),
    message: z.string().optional().describe('Custom message to include'),
  }),
  
  execute: async ({ context, params }) => {
    const { tenantId } = context;
    
    const result = await quoteService.send({
      tenantId,
      quoteId: params.quote_id,
      method: params.method,
      message: params.message,
    });
    
    return {
      success: true,
      method: params.method,
      ...(params.method === 'email' && { sent_to: result.email }),
      ...(params.method === 'link' && { share_url: result.shareUrl }),
      ...(params.method === 'whatsapp' && { whatsapp_url: result.whatsappUrl }),
    };
  },
});

export const pdfGenerateTool = createTool({
  name: 'pdf_generate',
  description: 'Generate a PDF for a quote',
  
  parameters: z.object({
    quote_id: z.string().uuid().describe('The quote ID'),
    template_id: z.string().uuid().optional().describe('Template to use'),
  }),
  
  execute: async ({ context, params }) => {
    const { tenantId } = context;
    
    const pdfUrl = await pdfService.generate({
      tenantId,
      quoteId: params.quote_id,
      templateId: params.template_id,
    });
    
    return {
      success: true,
      pdf_url: pdfUrl,
      download_url: `${process.env.APP_URL}/api/v1/quotes/${params.quote_id}/pdf`,
    };
  },
});
```

### 5.3 MCP Server Setup

```typescript
// src/mcp/server.ts
import { MCPServer } from '@mastra/mcp';
import {
  quoteCreateTool,
  quoteUpdateTool,
  quoteSendTool,
  productSearchTool,
  productCreateTool,
  contactLookupTool,
  contactCreateTool,
  pdfGenerateTool,
} from './tools';

export const mcpServer = new MCPServer({
  name: 'quoteflow-mcp',
  version: '1.0.0',
  
  tools: [
    quoteCreateTool,
    quoteUpdateTool,
    quoteSendTool,
    productSearchTool,
    productCreateTool,
    contactLookupTool,
    contactCreateTool,
    pdfGenerateTool,
  ],
  
  // Context injection for multi-tenancy
  contextProvider: async (request) => {
    const { tenantId, userId } = request.auth;
    const tenant = await tenantService.getById(tenantId);
    
    return {
      tenantId,
      userId,
      company_name: tenant.name,
      default_currency: tenant.settings.currency || 'USD',
      default_tax_rate: tenant.settings.taxRate || 0,
      locale: tenant.settings.locale || 'en-US',
    };
  },
});
```

---

## 6. Chat Flow Architecture

### 6.1 Message Processing Pipeline

```typescript
// src/services/chatService.ts
import { quoteAgent } from '../agents/quoteAgent';
import { conversationRepository } from '../repositories';

export class ChatService {
  async processMessage(params: {
    tenantId: string;
    userId: string;
    conversationId?: string;
    message: string;
    channel: 'web' | 'whatsapp' | 'telegram';
  }) {
    // 1. Get or create conversation
    let conversation = params.conversationId
      ? await conversationRepository.getById(params.conversationId)
      : await conversationRepository.create({
          tenantId: params.tenantId,
          userId: params.userId,
          channel: params.channel,
        });

    // 2. Load conversation history for context
    const history = await conversationRepository.getMessages(conversation.id);

    // 3. Build context for agent
    const context = await this.buildContext(params.tenantId, params.userId);

    // 4. Execute agent with message
    const response = await quoteAgent.execute({
      messages: [
        ...history.map(m => ({ role: m.role, content: m.content })),
        { role: 'user', content: params.message },
      ],
      context,
    });

    // 5. Save messages
    await conversationRepository.addMessage({
      conversationId: conversation.id,
      role: 'user',
      content: params.message,
    });

    await conversationRepository.addMessage({
      conversationId: conversation.id,
      role: 'assistant',
      content: response.content,
      toolCalls: response.toolCalls,
      tokensUsed: response.usage?.totalTokens,
    });

    // 6. Return response
    return {
      conversationId: conversation.id,
      response: response.content,
      actionsTaken: response.toolCalls?.map(tc => ({
        tool: tc.name,
        result: tc.result,
      })),
    };
  }

  private async buildContext(tenantId: string, userId: string) {
    const tenant = await tenantService.getById(tenantId);
    const user = await userService.getById(userId);

    return {
      tenantId,
      userId,
      company_name: tenant.name,
      user_name: user.name,
      default_currency: tenant.settings.currency || 'USD',
      default_tax_rate: tenant.settings.taxRate || 0,
      locale: tenant.settings.locale || 'en-US',
    };
  }
}
```

### 6.2 WhatsApp Integration

```typescript
// src/webhooks/whatsapp.ts
import { Router } from 'express';
import { WhatsAppClient } from '../integrations/whatsapp';
import { chatService } from '../services/chatService';

const router = Router();

// Webhook verification
router.get('/whatsapp', (req, res) => {
  const mode = req.query['hub.mode'];
  const token = req.query['hub.verify_token'];
  const challenge = req.query['hub.challenge'];

  if (mode === 'subscribe' && token === process.env.WHATSAPP_VERIFY_TOKEN) {
    res.status(200).send(challenge);
  } else {
    res.sendStatus(403);
  }
});

// Message webhook
router.post('/whatsapp', async (req, res) => {
  try {
    const { entry } = req.body;
    
    for (const e of entry) {
      for (const change of e.changes) {
        if (change.value.messages) {
          for (const message of change.value.messages) {
            await handleWhatsAppMessage(message, change.value.metadata);
          }
        }
      }
    }
    
    res.sendStatus(200);
  } catch (error) {
    console.error('WhatsApp webhook error:', error);
    res.sendStatus(500);
  }
});

async function handleWhatsAppMessage(
  message: WhatsAppMessage,
  metadata: WhatsAppMetadata
) {
  // 1. Find tenant by WhatsApp business number
  const tenant = await tenantService.getByWhatsAppNumber(
    metadata.phone_number_id
  );
  
  if (!tenant) {
    console.error('No tenant found for WhatsApp number');
    return;
  }

  // 2. Find or create contact by phone number
  let contact = await contactService.getByPhone(tenant.id, message.from);
  if (!contact) {
    contact = await contactService.create({
      tenantId: tenant.id,
      phone: message.from,
      name: message.from, // Will be updated later
    });
  }

  // 3. Get or create conversation
  let conversation = await conversationRepository.getByExternalId(
    tenant.id,
    `whatsapp:${message.from}`
  );
  
  if (!conversation) {
    conversation = await conversationRepository.create({
      tenantId: tenant.id,
      channel: 'whatsapp',
      externalId: `whatsapp:${message.from}`,
      context: { contactId: contact.id },
    });
  }

  // 4. Process message through AI
  const response = await chatService.processMessage({
    tenantId: tenant.id,
    userId: tenant.defaultUserId, // System user for channel messages
    conversationId: conversation.id,
    message: message.text?.body || '',
    channel: 'whatsapp',
  });

  // 5. Send response back via WhatsApp
  const whatsapp = new WhatsAppClient(tenant.whatsappConfig);
  await whatsapp.sendMessage(message.from, response.response);

  // 6. If a PDF was generated, send it
  if (response.actionsTaken?.some(a => a.tool === 'pdf_generate')) {
    const pdfAction = response.actionsTaken.find(
      a => a.tool === 'pdf_generate'
    );
    await whatsapp.sendDocument(message.from, pdfAction.result.pdf_url);
  }
}

export default router;
```

---

## 7. PDF Generation System

### 7.1 Template Engine

```typescript
// src/services/pdfService.ts
import puppeteer from 'puppeteer';
import Handlebars from 'handlebars';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';

export class PDFService {
  private browser: puppeteer.Browser | null = null;

  async generate(params: {
    tenantId: string;
    quoteId: string;
    templateId?: string;
  }): Promise<string> {
    // 1. Fetch quote with all relations
    const quote = await quoteRepository.getById(params.quoteId, {
      include: ['contact', 'items', 'items.product', 'tenant'],
    });

    // 2. Get template
    const template = params.templateId
      ? await templateRepository.getById(params.templateId)
      : await templateRepository.getDefault(params.tenantId);

    // 3. Prepare template data
    const data = this.prepareTemplateData(quote);

    // 4. Compile template
    const html = this.compileTemplate(template, data);

    // 5. Generate PDF
    const pdfBuffer = await this.renderPDF(html, template.cssStyles);

    // 6. Upload to S3
    const pdfUrl = await this.uploadToS3(
      pdfBuffer,
      `quotes/${params.tenantId}/${quote.quoteNumber}.pdf`
    );

    // 7. Update quote with PDF URL
    await quoteRepository.update(params.quoteId, { pdfUrl });

    return pdfUrl;
  }

  private prepareTemplateData(quote: Quote) {
    return {
      quote: {
        number: quote.quoteNumber,
        date: this.formatDate(quote.createdAt, quote.tenant.settings.locale),
        validUntil: this.formatDate(quote.validUntil, quote.tenant.settings.locale),
        status: quote.status,
        notes: quote.notes,
        terms: quote.terms,
      },
      company: {
        name: quote.tenant.name,
        logo: quote.tenant.settings.logoUrl,
        address: quote.tenant.settings.address,
        phone: quote.tenant.settings.phone,
        email: quote.tenant.settings.email,
        website: quote.tenant.settings.website,
        taxId: quote.tenant.settings.taxId,
      },
      customer: {
        name: quote.contact.name,
        company: quote.contact.company,
        email: quote.contact.email,
        phone: quote.contact.phone,
        address: quote.contact.address,
      },
      items: quote.items.map(item => ({
        description: item.description,
        quantity: this.formatNumber(item.quantity, quote.tenant.settings.locale),
        uom: item.uom,
        unitPrice: this.formatCurrency(item.unitPrice, quote.currency, quote.tenant.settings.locale),
        discount: item.discountPercent > 0 ? `${item.discountPercent}%` : null,
        tax: item.taxRate > 0 ? `${item.taxRate}%` : null,
        total: this.formatCurrency(item.lineTotal, quote.currency, quote.tenant.settings.locale),
      })),
      totals: {
        subtotal: this.formatCurrency(quote.subtotal, quote.currency, quote.tenant.settings.locale),
        discount: quote.discountAmount > 0 
          ? this.formatCurrency(quote.discountAmount, quote.currency, quote.tenant.settings.locale)
          : null,
        tax: this.formatCurrency(quote.taxAmount, quote.currency, quote.tenant.settings.locale),
        total: this.formatCurrency(quote.total, quote.currency, quote.tenant.settings.locale),
      },
      payment: quote.tenant.settings.paymentInfo,
      currency: quote.currency,
    };
  }

  private compileTemplate(template: Template, data: any): string {
    // Register helpers
    Handlebars.registerHelper('ifCond', function(v1, v2, options) {
      return v1 === v2 ? options.fn(this) : options.inverse(this);
    });

    const compiled = Handlebars.compile(template.htmlContent);
    return compiled(data);
  }

  private async renderPDF(html: string, css: string): Promise<Buffer> {
    if (!this.browser) {
      this.browser = await puppeteer.launch({
        headless: true,
        args: ['--no-sandbox', '--disable-setuid-sandbox'],
      });
    }

    const page = await this.browser.newPage();
    
    await page.setContent(`
      <!DOCTYPE html>
      <html>
        <head>
          <meta charset="UTF-8">
          <style>${css}</style>
        </head>
        <body>${html}</body>
      </html>
    `, { waitUntil: 'networkidle0' });

    const pdf = await page.pdf({
      format: 'A4',
      margin: { top: '20mm', right: '15mm', bottom: '20mm', left: '15mm' },
      printBackground: true,
    });

    await page.close();
    return Buffer.from(pdf);
  }

  private async uploadToS3(buffer: Buffer, key: string): Promise<string> {
    const s3 = new S3Client({ region: process.env.AWS_REGION });
    
    await s3.send(new PutObjectCommand({
      Bucket: process.env.S3_BUCKET,
      Key: key,
      Body: buffer,
      ContentType: 'application/pdf',
    }));

    return `https://${process.env.S3_BUCKET}.s3.${process.env.AWS_REGION}.amazonaws.com/${key}`;
  }

  private formatCurrency(amount: number, currency: string, locale: string): string {
    return new Intl.NumberFormat(locale, {
      style: 'currency',
      currency,
    }).format(amount);
  }

  private formatNumber(num: number, locale: string): string {
    return new Intl.NumberFormat(locale).format(num);
  }

  private formatDate(date: Date, locale: string): string {
    return new Intl.DateTimeFormat(locale, {
      year: 'numeric',
      month: 'long',
      day: 'numeric',
    }).format(new Date(date));
  }
}
```

### 7.2 Default Template Example

```html
<!-- templates/default.html -->
<div class="quote-document">
  <header class="header">
    <div class="company-info">
      {{#if company.logo}}
        <img src="{{company.logo}}" alt="{{company.name}}" class="logo" />
      {{/if}}
      <div class="company-details">
        <h1>{{company.name}}</h1>
        <p>{{company.address.street}}</p>
        <p>{{company.address.city}}, {{company.address.state}} {{company.address.zip}}</p>
        <p>{{company.phone}} | {{company.email}}</p>
        {{#if company.taxId}}<p>Tax ID: {{company.taxId}}</p>{{/if}}
      </div>
    </div>
    <div class="quote-info">
      <h2>QUOTATION</h2>
      <table>
        <tr><td>Quote #:</td><td>{{quote.number}}</td></tr>
        <tr><td>Date:</td><td>{{quote.date}}</td></tr>
        <tr><td>Valid Until:</td><td>{{quote.validUntil}}</td></tr>
      </table>
    </div>
  </header>

  <section class="customer-section">
    <h3>Bill To:</h3>
    <p><strong>{{customer.name}}</strong></p>
    {{#if customer.company}}<p>{{customer.company}}</p>{{/if}}
    {{#if customer.address}}
      <p>{{customer.address.street}}</p>
      <p>{{customer.address.city}}, {{customer.address.state}} {{customer.address.zip}}</p>
    {{/if}}
    {{#if customer.email}}<p>{{customer.email}}</p>{{/if}}
  </section>

  <section class="items-section">
    <table class="items-table">
      <thead>
        <tr>
          <th>Description</th>
          <th>Qty</th>
          <th>Unit</th>
          <th>Price</th>
          {{#if hasDiscounts}}<th>Discount</th>{{/if}}
          {{#if hasTax}}<th>Tax</th>{{/if}}
          <th>Total</th>
        </tr>
      </thead>
      <tbody>
        {{#each items}}
        <tr>
          <td>{{description}}</td>
          <td class="center">{{quantity}}</td>
          <td class="center">{{uom}}</td>
          <td class="right">{{unitPrice}}</td>
          {{#if ../hasDiscounts}}<td class="center">{{discount}}</td>{{/if}}
          {{#if ../hasTax}}<td class="center">{{tax}}</td>{{/if}}
          <td class="right">{{total}}</td>
        </tr>
        {{/each}}
      </tbody>
    </table>
  </section>

  <section class="totals-section">
    <table class="totals-table">
      <tr>
        <td>Subtotal:</td>
        <td>{{totals.subtotal}}</td>
      </tr>
      {{#if totals.discount}}
      <tr>
        <td>Discount:</td>
        <td>-{{totals.discount}}</td>
      </tr>
      {{/if}}
      <tr>
        <td>Tax:</td>
        <td>{{totals.tax}}</td>
      </tr>
      <tr class="total-row">
        <td><strong>Total:</strong></td>
        <td><strong>{{totals.total}}</strong></td>
      </tr>
    </table>
  </section>

  {{#if quote.notes}}
  <section class="notes-section">
    <h3>Notes:</h3>
    <p>{{quote.notes}}</p>
  </section>
  {{/if}}

  {{#if quote.terms}}
  <section class="terms-section">
    <h3>Terms & Conditions:</h3>
    <p>{{quote.terms}}</p>
  </section>
  {{/if}}

  {{#if payment}}
  <section class="payment-section">
    <h3>Payment Information:</h3>
    <p>{{payment.instructions}}</p>
    {{#if payment.bankName}}<p>Bank: {{payment.bankName}}</p>{{/if}}
    {{#if payment.accountNumber}}<p>Account: {{payment.accountNumber}}</p>{{/if}}
    {{#if payment.routingNumber}}<p>Routing: {{payment.routingNumber}}</p>{{/if}}
  </section>
  {{/if}}

  <footer class="footer">
    <p>Thank you for your business!</p>
    {{#if company.website}}<p>{{company.website}}</p>{{/if}}
  </footer>
</div>
```

---

## 8. Frontend Architecture

### 8.1 Project Structure

```
frontend/
├── src/
│   ├── app/                    # Next.js App Router
│   │   ├── (auth)/
│   │   │   ├── login/
│   │   │   └── register/
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx        # Dashboard home
│   │   │   ├── quotes/
│   │   │   │   ├── page.tsx    # Quote list
│   │   │   │   ├── new/
│   │   │   │   └── [id]/
│   │   │   ├── products/
│   │   │   ├── contacts/
│   │   │   ├── templates/
│   │   │   └── settings/
│   │   ├── chat/               # Full-page chat interface
│   │   └── api/                # API routes (if needed)
│   │
│   ├── components/
│   │   ├── ui/                 # Shadcn/UI components
│   │   ├── chat/
│   │   │   ├── ChatWindow.tsx
│   │   │   ├── MessageBubble.tsx
│   │   │   ├── ChatInput.tsx
│   │   │   └── QuotePreview.tsx
│   │   ├── quotes/
│   │   │   ├── QuoteForm.tsx
│   │   │   ├── QuoteTable.tsx
│   │   │   ├── LineItemEditor.tsx
│   │   │   └── QuotePreview.tsx
│   │   ├── products/
│   │   ├── contacts/
│   │   └── templates/
│   │       └── TemplateEditor.tsx
│   │
│   ├── hooks/
│   │   ├── useChat.ts
│   │   ├── useQuotes.ts
│   │   ├── useProducts.ts
│   │   └── useContacts.ts
│   │
│   ├── stores/
│   │   ├── authStore.ts
│   │   ├── chatStore.ts
│   │   └── uiStore.ts
│   │
│   ├── lib/
│   │   ├── api.ts              # API client
│   │   ├── utils.ts
│   │   └── validators.ts
│   │
│   └── types/
│       └── index.ts
│
├── public/
├── tailwind.config.js
└── package.json
```

### 8.2 Chat-to-UI Main Layout

The main quote creation interface combines chat, editable form, and live PDF preview:

```tsx
// src/components/quote-builder/QuoteBuilderLayout.tsx
'use client';

import { useQuoteBuilderStore } from '@/stores/quoteBuilderStore';
import { ChatPanel } from './ChatPanel';
import { QuoteEditorForm } from './QuoteEditorForm';
import { LivePDFPreview } from './LivePDFPreview';

export function QuoteBuilderLayout() {
  const { quoteDraft, isLoading } = useQuoteBuilderStore();

  return (
    <div className="flex h-screen">
      {/* Left: Chat Panel */}
      <div className="w-80 border-r flex flex-col bg-gray-50">
        <ChatPanel />
      </div>

      {/* Center: Editable Form */}
      <div className="flex-1 overflow-y-auto p-6">
        {quoteDraft ? (
          <QuoteEditorForm />
        ) : (
          <EmptyState />
        )}
      </div>

      {/* Right: Live PDF Preview */}
      <div className="w-[400px] border-l bg-white overflow-y-auto">
        {quoteDraft && <LivePDFPreview />}
      </div>
    </div>
  );
}

function EmptyState() {
  return (
    <div className="flex items-center justify-center h-full text-gray-500">
      <div className="text-center">
        <p className="text-lg font-medium">Start a quote</p>
        <p className="mt-2">Type in the chat or use the form below</p>
        <button className="mt-4 px-4 py-2 bg-primary text-white rounded-lg">
          Create New Quote
        </button>
      </div>
    </div>
  );
}
```

### 8.3 Quote Builder Store (Zustand)

Central state management for the chat-to-UI pipeline:

```tsx
// src/stores/quoteBuilderStore.ts
import { create } from 'zustand';
import { QuoteDraft } from '@/schemas/chatOutput';

interface Message {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  timestamp: Date;
  structuredOutput?: QuoteDraft;
}

interface QuoteBuilderState {
  // Chat state
  messages: Message[];
  conversationId: string | null;
  isLoading: boolean;
  
  // Quote draft state (populated from chat OR form edits)
  quoteDraft: QuoteDraft['data'] | null;
  
  // Actions
  sendMessage: (message: string) => Promise<void>;
  updateQuoteDraft: (updates: Partial<QuoteDraft['data']>) => void;
  updateLineItem: (index: number, updates: Partial<QuoteDraft['data']['line_items'][0]>) => void;
  addLineItem: () => void;
  removeLineItem: (index: number) => void;
  resetQuote: () => void;
}

export const useQuoteBuilderStore = create<QuoteBuilderState>((set, get) => ({
  messages: [],
  conversationId: null,
  isLoading: false,
  quoteDraft: null,

  sendMessage: async (message: string) => {
    set({ isLoading: true });
    
    // Add user message
    const userMessage: Message = {
      id: crypto.randomUUID(),
      role: 'user',
      content: message,
      timestamp: new Date(),
    };
    
    set((state) => ({ messages: [...state.messages, userMessage] }));

    try {
      const response = await fetch('/api/v1/chat/message', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          conversation_id: get().conversationId,
          message,
        }),
      });

      const data = await response.json();
      
      // Add assistant message
      const assistantMessage: Message = {
        id: crypto.randomUUID(),
        role: 'assistant',
        content: data.data.response,
        timestamp: new Date(),
        structuredOutput: data.data.structured_output,
      };
      
      set((state) => ({
        messages: [...state.messages, assistantMessage],
        conversationId: data.data.conversation_id,
      }));

      // If structured output contains quote draft, update the form
      if (data.data.structured_output?.type === 'quote_draft') {
        set({ quoteDraft: data.data.structured_output.data });
      }
    } catch (error) {
      console.error('Chat error:', error);
    } finally {
      set({ isLoading: false });
    }
  },

  updateQuoteDraft: (updates) => {
    set((state) => ({
      quoteDraft: state.quoteDraft 
        ? { ...state.quoteDraft, ...updates }
        : null,
    }));
    // Recalculate totals
    get().recalculateTotals();
  },

  updateLineItem: (index, updates) => {
    set((state) => {
      if (!state.quoteDraft) return state;
      
      const newLineItems = [...state.quoteDraft.line_items];
      newLineItems[index] = { ...newLineItems[index], ...updates };
      
      // Recalculate line total
      const item = newLineItems[index];
      const subtotal = item.quantity * item.unit_price;
      const discount = subtotal * (item.discount_percent / 100);
      const afterDiscount = subtotal - discount;
      const tax = afterDiscount * (item.tax_rate / 100);
      newLineItems[index].line_total = afterDiscount + tax;
      
      return {
        quoteDraft: { ...state.quoteDraft, line_items: newLineItems },
      };
    });
    get().recalculateTotals();
  },

  addLineItem: () => {
    set((state) => {
      if (!state.quoteDraft) return state;
      
      const newItem = {
        id: crypto.randomUUID(),
        product_id: null,
        description: '',
        quantity: 1,
        uom: 'unit',
        unit_price: 0,
        discount_percent: 0,
        tax_rate: 0,
        line_total: 0,
      };
      
      return {
        quoteDraft: {
          ...state.quoteDraft,
          line_items: [...state.quoteDraft.line_items, newItem],
        },
      };
    });
  },

  removeLineItem: (index) => {
    set((state) => {
      if (!state.quoteDraft || state.quoteDraft.line_items.length <= 1) return state;
      
      const newLineItems = state.quoteDraft.line_items.filter((_, i) => i !== index);
      return {
        quoteDraft: { ...state.quoteDraft, line_items: newLineItems },
      };
    });
    get().recalculateTotals();
  },

  recalculateTotals: () => {
    set((state) => {
      if (!state.quoteDraft) return state;
      
      const subtotal = state.quoteDraft.line_items.reduce(
        (sum, item) => sum + (item.quantity * item.unit_price),
        0
      );
      
      const discountAmount = state.quoteDraft.line_items.reduce(
        (sum, item) => sum + (item.quantity * item.unit_price * item.discount_percent / 100),
        0
      );
      
      const taxAmount = state.quoteDraft.line_items.reduce(
        (sum, item) => {
          const afterDiscount = item.quantity * item.unit_price * (1 - item.discount_percent / 100);
          return sum + (afterDiscount * item.tax_rate / 100);
        },
        0
      );
      
      return {
        quoteDraft: {
          ...state.quoteDraft,
          subtotal,
          discount_amount: discountAmount,
          tax_amount: taxAmount,
          total: subtotal - discountAmount + taxAmount,
        },
      };
    });
  },

  resetQuote: () => {
    set({ quoteDraft: null, messages: [], conversationId: null });
  },
}));
```

### 8.4 Live PDF Preview Component

Real-time PDF preview using react-pdf:

```tsx
// src/components/quote-builder/LivePDFPreview.tsx
'use client';

import { useMemo } from 'react';
import { Document, Page, Text, View, StyleSheet, PDFViewer } from '@react-pdf/renderer';
import { useQuoteBuilderStore } from '@/stores/quoteBuilderStore';
import { useCompanySettings } from '@/hooks/useCompanySettings';

const styles = StyleSheet.create({
  page: { padding: 40, fontSize: 10, fontFamily: 'Helvetica' },
  header: { flexDirection: 'row', justifyContent: 'space-between', marginBottom: 30 },
  logo: { width: 100, height: 40 },
  title: { fontSize: 24, fontWeight: 'bold', color: '#333' },
  section: { marginBottom: 20 },
  table: { display: 'flex', flexDirection: 'column', marginTop: 10 },
  tableHeader: { flexDirection: 'row', backgroundColor: '#f3f4f6', padding: 8, fontWeight: 'bold' },
  tableRow: { flexDirection: 'row', padding: 8, borderBottomWidth: 1, borderBottomColor: '#e5e7eb' },
  col1: { width: '40%' },
  col2: { width: '15%', textAlign: 'center' },
  col3: { width: '15%', textAlign: 'right' },
  col4: { width: '15%', textAlign: 'right' },
  col5: { width: '15%', textAlign: 'right' },
  totals: { marginTop: 20, alignItems: 'flex-end' },
  totalRow: { flexDirection: 'row', justifyContent: 'flex-end', marginBottom: 4 },
  totalLabel: { width: 100, textAlign: 'right', marginRight: 10 },
  totalValue: { width: 80, textAlign: 'right' },
  grandTotal: { fontSize: 14, fontWeight: 'bold', marginTop: 8, paddingTop: 8, borderTopWidth: 2 },
});

export function LivePDFPreview() {
  const { quoteDraft } = useQuoteBuilderStore();
  const { company } = useCompanySettings();

  const QuoteDocument = useMemo(() => {
    if (!quoteDraft) return null;

    return (
      <Document>
        <Page size="A4" style={styles.page}>
          {/* Header */}
          <View style={styles.header}>
            <View>
              <Text style={styles.title}>QUOTATION</Text>
              <Text>#{quoteDraft.quote_number}</Text>
              <Text>Date: {new Date().toLocaleDateString()}</Text>
              <Text>Valid Until: {quoteDraft.valid_until}</Text>
            </View>
            <View>
              <Text style={{ fontWeight: 'bold' }}>{company?.name}</Text>
              <Text>{company?.address}</Text>
              <Text>{company?.email}</Text>
            </View>
          </View>

          {/* Bill To */}
          <View style={styles.section}>
            <Text style={{ fontWeight: 'bold', marginBottom: 4 }}>Bill To:</Text>
            <Text>{quoteDraft.customer.name}</Text>
            {quoteDraft.customer.company && <Text>{quoteDraft.customer.company}</Text>}
            {quoteDraft.customer.email && <Text>{quoteDraft.customer.email}</Text>}
          </View>

          {/* Line Items Table */}
          <View style={styles.table}>
            <View style={styles.tableHeader}>
              <Text style={styles.col1}>Description</Text>
              <Text style={styles.col2}>Qty</Text>
              <Text style={styles.col3}>Unit Price</Text>
              <Text style={styles.col4}>Discount</Text>
              <Text style={styles.col5}>Total</Text>
            </View>
            {quoteDraft.line_items.map((item, index) => (
              <View key={index} style={styles.tableRow}>
                <Text style={styles.col1}>{item.description}</Text>
                <Text style={styles.col2}>{item.quantity} {item.uom}</Text>
                <Text style={styles.col3}>${item.unit_price.toFixed(2)}</Text>
                <Text style={styles.col4}>{item.discount_percent > 0 ? `${item.discount_percent}%` : '-'}</Text>
                <Text style={styles.col5}>${item.line_total.toFixed(2)}</Text>
              </View>
            ))}
          </View>

          {/* Totals */}
          <View style={styles.totals}>
            <View style={styles.totalRow}>
              <Text style={styles.totalLabel}>Subtotal:</Text>
              <Text style={styles.totalValue}>${quoteDraft.subtotal.toFixed(2)}</Text>
            </View>
            {quoteDraft.discount_amount > 0 && (
              <View style={styles.totalRow}>
                <Text style={styles.totalLabel}>Discount:</Text>
                <Text style={styles.totalValue}>-${quoteDraft.discount_amount.toFixed(2)}</Text>
              </View>
            )}
            <View style={styles.totalRow}>
              <Text style={styles.totalLabel}>Tax:</Text>
              <Text style={styles.totalValue}>${quoteDraft.tax_amount.toFixed(2)}</Text>
            </View>
            <View style={[styles.totalRow, styles.grandTotal]}>
              <Text style={styles.totalLabel}>TOTAL ({quoteDraft.currency}):</Text>
              <Text style={styles.totalValue}>${quoteDraft.total.toFixed(2)}</Text>
            </View>
          </View>

          {/* Notes */}
          {quoteDraft.notes && (
            <View style={[styles.section, { marginTop: 30 }]}>
              <Text style={{ fontWeight: 'bold', marginBottom: 4 }}>Notes:</Text>
              <Text>{quoteDraft.notes}</Text>
            </View>
          )}
        </Page>
      </Document>
    );
  }, [quoteDraft, company]);

  if (!quoteDraft) return null;

  return (
    <div className="h-full">
      <div className="p-4 border-b bg-gray-50">
        <h3 className="font-medium">Live Preview</h3>
        <p className="text-sm text-gray-500">Updates as you edit</p>
      </div>
      <PDFViewer width="100%" height="calc(100% - 60px)" showToolbar={false}>
        {QuoteDocument}
      </PDFViewer>
    </div>
  );
}
```

### 8.5 Chat Panel Component

```tsx
// src/components/quote-builder/ChatPanel.tsx
'use client';

import { useState, useRef, useEffect } from 'react';
import { useQuoteBuilderStore } from '@/stores/quoteBuilderStore';
import { MessageBubble } from './MessageBubble';

export function ChatPanel() {
  const { messages, isLoading, sendMessage } = useQuoteBuilderStore();
  const [input, setInput] = useState('');
  const messagesEndRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!input.trim() || isLoading) return;
    
    const message = input;
    setInput('');
    await sendMessage(message);
  };

  return (
    <div className="flex flex-col h-full">
      {/* Messages */}
      <div className="flex-1 overflow-y-auto p-4 space-y-3">
        {messages.length === 0 && (
          <div className="text-center text-gray-500 text-sm mt-4">
            <p className="font-medium">Chat with QuoteFlow AI</p>
            <p className="mt-2">Try:</p>
            <ul className="mt-1 space-y-1 text-xs">
              <li>"Quote for John at ABC Corp"</li>
              <li>"Add 5 hours consulting"</li>
              <li>"Apply 10% discount"</li>
            </ul>
          </div>
        )}
        
        {messages.map((message) => (
          <MessageBubble key={message.id} message={message} />
        ))}
        
        {isLoading && (
          <div className="flex items-center space-x-1 text-gray-400 text-sm">
            <span className="animate-bounce">●</span>
            <span className="animate-bounce delay-100">●</span>
            <span className="animate-bounce delay-200">●</span>
          </div>
        )}
        
        <div ref={messagesEndRef} />
      </div>

      {/* Input */}
      <form onSubmit={handleSubmit} className="p-3 border-t bg-white">
        <div className="flex gap-2">
          <input
            type="text"
            value={input}
            onChange={(e) => setInput(e.target.value)}
            placeholder="Type a message..."
            className="flex-1 px-3 py-2 border rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-primary"
            disabled={isLoading}
          />
          <button
            type="submit"
            disabled={isLoading || !input.trim()}
            className="px-4 py-2 bg-primary text-white rounded-lg text-sm disabled:opacity-50"
          >
            Send
          </button>
        </div>
      </form>
    </div>
  );
}
```

### 8.3 Quote Form Component

```tsx
// src/components/quotes/QuoteForm.tsx
'use client';

import { useForm, useFieldArray } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Select } from '@/components/ui/select';
import { ContactSelector } from './ContactSelector';
import { ProductSelector } from './ProductSelector';

const quoteSchema = z.object({
  contactId: z.string().uuid(),
  validDays: z.number().min(1).max(365),
  currency: z.string().length(3),
  notes: z.string().optional(),
  items: z.array(z.object({
    productId: z.string().uuid().optional(),
    description: z.string().min(1),
    quantity: z.number().positive(),
    uom: z.string(),
    unitPrice: z.number().positive(),
    discountPercent: z.number().min(0).max(100),
    taxRate: z.number().min(0).max(100),
  })).min(1),
});

type QuoteFormData = z.infer<typeof quoteSchema>;

export function QuoteForm({ onSubmit, initialData }: {
  onSubmit: (data: QuoteFormData) => Promise<void>;
  initialData?: Partial<QuoteFormData>;
}) {
  const {
    register,
    control,
    handleSubmit,
    watch,
    setValue,
    formState: { errors, isSubmitting },
  } = useForm<QuoteFormData>({
    resolver: zodResolver(quoteSchema),
    defaultValues: {
      validDays: 30,
      currency: 'USD',
      items: [{ description: '', quantity: 1, uom: 'unit', unitPrice: 0, discountPercent: 0, taxRate: 0 }],
      ...initialData,
    },
  });

  const { fields, append, remove } = useFieldArray({
    control,
    name: 'items',
  });

  const items = watch('items');
  
  const calculateTotals = () => {
    let subtotal = 0;
    let totalDiscount = 0;
    let totalTax = 0;

    items.forEach(item => {
      const lineSubtotal = item.quantity * item.unitPrice;
      const discount = lineSubtotal * (item.discountPercent / 100);
      const afterDiscount = lineSubtotal - discount;
      const tax = afterDiscount * (item.taxRate / 100);
      
      subtotal += lineSubtotal;
      totalDiscount += discount;
      totalTax += tax;
    });

    return {
      subtotal,
      discount: totalDiscount,
      tax: totalTax,
      total: subtotal - totalDiscount + totalTax,
    };
  };

  const totals = calculateTotals();

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-6">
      {/* Contact Selection */}
      <div>
        <label className="block text-sm font-medium mb-1">Customer</label>
        <ContactSelector
          value={watch('contactId')}
          onChange={(id) => setValue('contactId', id)}
        />
        {errors.contactId && (
          <p className="text-red-500 text-sm mt-1">{errors.contactId.message}</p>
        )}
      </div>

      {/* Quote Settings */}
      <div className="grid grid-cols-2 gap-4">
        <div>
          <label className="block text-sm font-medium mb-1">Valid For (days)</label>
          <Input type="number" {...register('validDays', { valueAsNumber: true })} />
        </div>
        <div>
          <label className="block text-sm font-medium mb-1">Currency</label>
          <Select {...register('currency')}>
            <option value="USD">USD - US Dollar</option>
            <option value="EUR">EUR - Euro</option>
            <option value="GBP">GBP - British Pound</option>
            <option value="INR">INR - Indian Rupee</option>
          </Select>
        </div>
      </div>

      {/* Line Items */}
      <div>
        <div className="flex justify-between items-center mb-2">
          <label className="block text-sm font-medium">Line Items</label>
          <Button
            type="button"
            variant="outline"
            size="sm"
            onClick={() => append({
              description: '',
              quantity: 1,
              uom: 'unit',
              unitPrice: 0,
              discountPercent: 0,
              taxRate: 0,
            })}
          >
            Add Item
          </Button>
        </div>

        <div className="space-y-3">
          {fields.map((field, index) => (
            <div key={field.id} className="border rounded-lg p-4 bg-gray-50">
              <div className="grid grid-cols-6 gap-3">
                <div className="col-span-3">
                  <ProductSelector
                    onSelect={(product) => {
                      setValue(`items.${index}.productId`, product.id);
                      setValue(`items.${index}.description`, product.name);
                      setValue(`items.${index}.unitPrice`, product.price);
                      setValue(`items.${index}.uom`, product.uom);
                      setValue(`items.${index}.taxRate`, product.taxRate);
                    }}
                  />
                  <Input
                    {...register(`items.${index}.description`)}
                    placeholder="Description"
                    className="mt-2"
                  />
                </div>
                <div>
                  <Input
                    type="number"
                    {...register(`items.${index}.quantity`, { valueAsNumber: true })}
                    placeholder="Qty"
                  />
                </div>
                <div>
                  <Input
                    type="number"
                    step="0.01"
                    {...register(`items.${index}.unitPrice`, { valueAsNumber: true })}
                    placeholder="Price"
                  />
                </div>
                <div className="flex items-center gap-2">
                  <Input
                    type="number"
                    {...register(`items.${index}.discountPercent`, { valueAsNumber: true })}
                    placeholder="%"
                    className="w-16"
                  />
                  <Button
                    type="button"
                    variant="ghost"
                    size="sm"
                    onClick={() => remove(index)}
                    disabled={fields.length === 1}
                  >
                    ✕
                  </Button>
                </div>
              </div>
            </div>
          ))}
        </div>
      </div>

      {/* Totals */}
      <div className="bg-gray-100 rounded-lg p-4">
        <div className="space-y-2 text-right">
          <div className="flex justify-end">
            <span className="w-32">Subtotal:</span>
            <span className="w-32 font-medium">${totals.subtotal.toFixed(2)}</span>
          </div>
          {totals.discount > 0 && (
            <div className="flex justify-end text-green-600">
              <span className="w-32">Discount:</span>
              <span className="w-32">-${totals.discount.toFixed(2)}</span>
            </div>
          )}
          <div className="flex justify-end">
            <span className="w-32">Tax:</span>
            <span className="w-32">${totals.tax.toFixed(2)}</span>
          </div>
          <div className="flex justify-end text-lg font-bold border-t pt-2">
            <span className="w-32">Total:</span>
            <span className="w-32">${totals.total.toFixed(2)}</span>
          </div>
        </div>
      </div>

      {/* Notes */}
      <div>
        <label className="block text-sm font-medium mb-1">Notes</label>
        <textarea
          {...register('notes')}
          className="w-full border rounded-lg p-3"
          rows={3}
          placeholder="Additional notes for the customer..."
        />
      </div>

      {/* Actions */}
      <div className="flex justify-end gap-3">
        <Button type="button" variant="outline">
          Save as Draft
        </Button>
        <Button type="submit" disabled={isSubmitting}>
          {isSubmitting ? 'Creating...' : 'Create Quote'}
        </Button>
      </div>
    </form>
  );
}
```

---

## 9. Security Architecture

### 9.1 Authentication Flow

```typescript
// src/middleware/auth.ts
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { userService } from '../services/userService';

export interface AuthRequest extends Request {
  user?: {
    id: string;
    tenantId: string;
    email: string;
    role: string;
  };
}

export const authenticate = async (
  req: AuthRequest,
  res: Response,
  next: NextFunction
) => {
  try {
    const authHeader = req.headers.authorization;
    
    if (!authHeader?.startsWith('Bearer ')) {
      return res.status(401).json({ error: 'No token provided' });
    }

    const token = authHeader.substring(7);
    const decoded = jwt.verify(token, process.env.JWT_SECRET!) as {
      userId: string;
      tenantId: string;
    };

    const user = await userService.getById(decoded.userId);
    
    if (!user || user.tenantId !== decoded.tenantId) {
      return res.status(401).json({ error: 'Invalid token' });
    }

    req.user = {
      id: user.id,
      tenantId: user.tenantId,
      email: user.email,
      role: user.role,
    };

    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
};

export const requireRole = (...roles: string[]) => {
  return (req: AuthRequest, res: Response, next: NextFunction) => {
    if (!req.user || !roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
};
```

### 9.2 Rate Limiting

```typescript
// src/middleware/rateLimit.ts
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import { redis } from '../lib/redis';

export const apiLimiter = rateLimit({
  store: new RedisStore({
    sendCommand: (...args: string[]) => redis.call(...args),
  }),
  windowMs: 60 * 1000, // 1 minute
  max: 100, // 100 requests per minute
  message: { error: 'Too many requests, please try again later' },
  keyGenerator: (req) => req.user?.tenantId || req.ip,
});

export const chatLimiter = rateLimit({
  store: new RedisStore({
    sendCommand: (...args: string[]) => redis.call(...args),
  }),
  windowMs: 60 * 1000,
  max: 20, // 20 chat messages per minute
  message: { error: 'Too many messages, please slow down' },
  keyGenerator: (req) => `chat:${req.user?.id}`,
});

export const pdfLimiter = rateLimit({
  store: new RedisStore({
    sendCommand: (...args: string[]) => redis.call(...args),
  }),
  windowMs: 60 * 1000,
  max: 10, // 10 PDF generations per minute
  message: { error: 'Too many PDF requests' },
  keyGenerator: (req) => `pdf:${req.user?.tenantId}`,
});
```

### 9.3 Input Validation

```typescript
// src/middleware/validate.ts
import { Request, Response, NextFunction } from 'express';
import { AnyZodObject, ZodError } from 'zod';

export const validate = (schema: AnyZodObject) => {
  return async (req: Request, res: Response, next: NextFunction) => {
    try {
      await schema.parseAsync({
        body: req.body,
        query: req.query,
        params: req.params,
      });
      next();
    } catch (error) {
      if (error instanceof ZodError) {
        return res.status(400).json({
          error: 'Validation failed',
          details: error.errors.map(e => ({
            field: e.path.join('.'),
            message: e.message,
          })),
        });
      }
      next(error);
    }
  };
};
```

---

## 10. Deployment Architecture

### 10.1 Infrastructure Diagram

```
                                    ┌─────────────────┐
                                    │   CloudFlare    │
                                    │   (CDN + WAF)   │
                                    └────────┬────────┘
                                             │
                    ┌────────────────────────┼────────────────────────┐
                    │                        │                        │
                    ▼                        ▼                        ▼
           ┌────────────────┐      ┌────────────────┐      ┌────────────────┐
           │  Web Frontend  │      │   API Server   │      │  Webhook Server│
           │   (Vercel)     │      │   (AWS ECS)    │      │   (AWS ECS)    │
           └────────────────┘      └───────┬────────┘      └───────┬────────┘
                                           │                       │
                    ┌──────────────────────┼───────────────────────┘
                    │                      │
                    ▼                      ▼
           ┌────────────────┐      ┌────────────────┐
           │   PostgreSQL   │      │     Redis      │
           │   (AWS RDS)    │      │ (AWS ElastiC.) │
           └────────────────┘      └────────────────┘
                    │
                    ▼
           ┌────────────────┐      ┌────────────────┐
           │    AWS S3      │      │   OpenRouter   │
           │  (File Store)  │      │   (LLM API)    │
           └────────────────┘      └────────────────┘
```

### 10.2 Environment Configuration

```bash
# .env.example

# Application
NODE_ENV=production
APP_URL=https://app.quoteflow.ai
API_URL=https://api.quoteflow.ai

# Database
DATABASE_URL=postgresql://user:pass@host:5432/quoteflow

# Redis
REDIS_URL=redis://host:6379

# Authentication
JWT_SECRET=your-jwt-secret
JWT_EXPIRES_IN=7d

# AWS
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=xxx
AWS_SECRET_ACCESS_KEY=xxx
S3_BUCKET=quoteflow-files

# AI/LLM
OPENROUTER_API_KEY=xxx
DEFAULT_MODEL=anthropic/claude-3.5-sonnet

# WhatsApp
WHATSAPP_API_URL=https://graph.facebook.com/v18.0
WHATSAPP_ACCESS_TOKEN=xxx
WHATSAPP_VERIFY_TOKEN=xxx

# Telegram
TELEGRAM_BOT_TOKEN=xxx

# Email
SENDGRID_API_KEY=xxx
FROM_EMAIL=quotes@quoteflow.ai

# Stripe
STRIPE_SECRET_KEY=xxx
STRIPE_WEBHOOK_SECRET=xxx

# Monitoring
SENTRY_DSN=xxx
DATADOG_API_KEY=xxx
```

### 10.3 Docker Configuration

```dockerfile
# Dockerfile
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runner

WORKDIR /app
ENV NODE_ENV=production

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

# Install Puppeteer dependencies for PDF generation
RUN apk add --no-cache \
    chromium \
    nss \
    freetype \
    harfbuzz \
    ca-certificates \
    ttf-freefont

ENV PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium-browser

EXPOSE 3000

CMD ["node", "dist/server.js"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/quoteflow
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis

  db:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=quoteflow
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

---

## 11. Testing Strategy

### 11.1 Test Structure

```
tests/
├── unit/
│   ├── services/
│   │   ├── quoteService.test.ts
│   │   ├── productService.test.ts
│   │   └── pdfService.test.ts
│   └── utils/
│       └── calculations.test.ts
│
├── integration/
│   ├── api/
│   │   ├── quotes.test.ts
│   │   ├── products.test.ts
│   │   └── contacts.test.ts
│   └── chat/
│       └── chatFlow.test.ts
│
├── e2e/
│   ├── quoteCreation.test.ts
│   ├── chatInterface.test.ts
│   └── pdfGeneration.test.ts
│
└── fixtures/
    ├── quotes.json
    ├── products.json
    └── contacts.json
```

### 11.2 Example Tests

```typescript
// tests/integration/chat/chatFlow.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { chatService } from '../../../src/services/chatService';
import { setupTestDB, cleanupTestDB, createTestTenant } from '../../helpers';

describe('Chat Flow Integration', () => {
  let tenant: any;
  let user: any;

  beforeEach(async () => {
    await setupTestDB();
    tenant = await createTestTenant();
    user = tenant.users[0];
  });

  afterEach(async () => {
    await cleanupTestDB();
  });

  it('should create a quote via chat', async () => {
    // First message: Create quote
    const response1 = await chatService.processMessage({
      tenantId: tenant.id,
      userId: user.id,
      message: 'Create a quote for John Smith at ABC Corp',
      channel: 'web',
    });

    expect(response1.response).toContain('quote');
    expect(response1.actionsTaken).toContainEqual(
      expect.objectContaining({ tool: 'contact_lookup' })
    );

    // Second message: Add items
    const response2 = await chatService.processMessage({
      tenantId: tenant.id,
      userId: user.id,
      conversationId: response1.conversationId,
      message: 'Add 5 hours of consulting at $150/hour',
      channel: 'web',
    });

    expect(response2.response).toContain('$750');
    expect(response2.actionsTaken).toContainEqual(
      expect.objectContaining({ tool: 'quote_item_add' })
    );
  });

  it('should handle product lookup', async () => {
    const response = await chatService.processMessage({
      tenantId: tenant.id,
      userId: user.id,
      message: 'What products do I have for web design?',
      channel: 'web',
    });

    expect(response.actionsTaken).toContainEqual(
      expect.objectContaining({ tool: 'product_search' })
    );
  });
});
```

---

## 12. Monitoring & Observability

### 12.1 Logging Configuration

```typescript
// src/lib/logger.ts
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: process.env.NODE_ENV === 'development'
    ? { target: 'pino-pretty' }
    : undefined,
  base: {
    service: 'quoteflow-api',
    version: process.env.APP_VERSION,
  },
});

export const requestLogger = pino({
  level: 'info',
  base: { type: 'request' },
  redact: ['req.headers.authorization', 'req.body.password'],
});
```

### 12.2 Metrics Collection

```typescript
// src/lib/metrics.ts
import { StatsD } from 'hot-shots';

const statsd = new StatsD({
  host: process.env.DATADOG_HOST,
  port: 8125,
  prefix: 'quoteflow.',
});

export const metrics = {
  increment: (metric: string, tags?: string[]) => {
    statsd.increment(metric, 1, tags);
  },
  
  timing: (metric: string, value: number, tags?: string[]) => {
    statsd.timing(metric, value, tags);
  },
  
  gauge: (metric: string, value: number, tags?: string[]) => {
    statsd.gauge(metric, value, tags);
  },
};

// Usage examples:
// metrics.increment('quotes.created', ['tier:pro']);
// metrics.timing('pdf.generation_time', 2500);
// metrics.gauge('active_users', 150);
```

---

## 13. Development Workflow

### 13.1 Git Workflow

```
main (production)
  │
  └── develop (staging)
        │
        ├── feature/quote-templates
        ├── feature/whatsapp-integration
        └── fix/pdf-generation-bug
```

### 13.2 CI/CD Pipeline

```yaml
# .github/workflows/ci.yml
name: CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
      redis:
        image: redis:7
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm run test:unit
      - run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/test
          REDIS_URL: redis://localhost:6379

  deploy-staging:
    needs: test
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Staging
        run: |
          # Deploy to staging environment

  deploy-production:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Production
        run: |
          # Deploy to production environment
```

---

**Document Owner:** Engineering Team  
**Last Updated:** February 23, 2026  
**Next Review:** March 23, 2026
