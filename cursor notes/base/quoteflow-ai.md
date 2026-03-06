# QuoteFlow AI - Product Requirements Document (PRD)

**Version:** 2.0
**Date:** March 6, 2026
**Status:** Draft

---

## 1. Executive Summary

QuoteFlow AI is a WhatsApp-first, AI-powered quotation tool built for freelancers, contractors, and small service businesses in markets where WhatsApp is the primary channel of business communication — India, Southeast Asia, MENA, and LATAM.

The core insight: millions of contractors and service providers already conduct business on WhatsApp. They send quotes as voice notes, hand-typed messages, or blurry photos of handwritten papers. QuoteFlow AI meets them where they already are — no new app to install, no behavior change required — and upgrades their quoting from unprofessional to instant and polished.

On the web side, the platform uses a **Chat-to-UI Pipeline** architecture: natural language chat generates structured data that populates an editable form with a live PDF preview. This eliminates the core failure mode of AI tools — misinterpretations are fixed with one click in the form, not by re-prompting the AI.

### 1.1 Vision Statement

"Enable any contractor or freelancer to send a professional quote in under 2 minutes, from WhatsApp or the web, in their language."

### 1.2 Core Value Proposition

- **Zero friction**: Create quotes via WhatsApp — no app download, no learning curve
- **Speed**: Chat generates a professional quote in under 2 minutes
- **Precision**: Chat → Editable form → Live PDF preview eliminates AI error risk
- **Emerging market-first**: Priced and designed for India, MENA, LATAM — not retrofitted
- **Professional output**: Contractors who quote on paper look as professional as agencies

### 1.3 Why This Market, Why Now

| Factor | Reality |
|--------|---------|
| WhatsApp Business users | 200M+ globally as of 2024 |
| WhatsApp penetration in India | 500M+ users; primary business communication tool |
| Current quoting method | Voice notes, typed text, paper photos, Google Docs |
| Existing SaaS solutions for this | Near zero — PandaDoc/Proposify are built for Western SMBs |
| Willingness to pay | Demonstrated by Zoho, Freshworks success in these markets |

No major quoting SaaS has built a WhatsApp-native product for this segment. This is the gap.

### 1.4 Positioning vs. Competition

| Product | Primary Market | WhatsApp | Chat-to-Quote | Price |
|---------|---------------|----------|---------------|-------|
| **QuoteFlow AI** | India, MENA, LATAM | Native | Yes | $5–15/mo |
| PandaDoc | US, EU | No | No | $19–49/mo |
| Proposify | US, EU | No | No | $49+/mo |
| Better Proposals | US, UK | No | No | $19–49/mo |
| Quotient | US, AU | No | No | $25/mo |
| Zoho Quote | Global | No | No | $14+/mo |
| Invoice Ninja | Global | No | No | Free/OSS |

The incumbents will not build WhatsApp-native products for emerging markets. Their pricing, infrastructure, and go-to-market are optimized for Western SMBs. This is a structural, not temporary, gap.

---

### 1.5 Core Architecture: Chat-to-UI Pipeline

QuoteFlow AI uses a **Chat-to-UI Pipeline** where the LLM is an accelerator, not the interface. This is the key architectural decision that separates it from pure chatbot approaches.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CHAT-TO-UI PIPELINE                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   User Input (Web Chat OR WhatsApp)                                      │
│   "Quote for Raj at Infra Ltd, 3 days civil survey, rate 8500/day"      │
│         │                                                                │
│         ▼                                                                │
│   ┌─────────────────┐                                                   │
│   │  LLM Processing │  ← Parses intent, matches product catalog         │
│   └────────┬────────┘                                                   │
│            │                                                             │
│            ▼                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Structured JSON Output (validated against schema)             │   │
│   │  { customer, lineItems, discount, currency, validDays }        │   │
│   └────────┬────────────────────────────────────────────────────────┘   │
│            │                                                             │
│            ▼                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  EDITABLE FORM UI                   LIVE PDF PREVIEW            │   │
│   │  ┌───────────────────────────┐      ┌──────────────────────┐   │   │
│   │  │ Customer: [Raj Kumar  ▼]  │      │ [Company Logo]       │   │   │
│   │  │ Company:  [Infra Ltd    ] │      │ QUOTATION #Q-0042    │   │   │
│   │  ├───────────────────────────┤      │ To: Raj Kumar        │   │   │
│   │  │ LINE ITEMS                │  --> │ ─────────────────    │   │   │
│   │  │ Civil Survey  3d  ₹8,500  │      │ Civil Survey  3d     │   │   │
│   │  │ [+ Add Line Item]         │      │ ₹8,500/d  ₹25,500   │   │   │
│   │  ├───────────────────────────┤      │ ─────────────────    │   │   │
│   │  │ Total: ₹25,500            │      │ Total: ₹25,500       │   │   │
│   │  └───────────────────────────┘      └──────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│            │                                                             │
│            ▼                                                             │
│   User shares via WhatsApp link, email, or PDF download                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Why this beats pure chat:**

| Pure Chat Approach | Chat-to-UI Pipeline |
|--------------------|---------------------|
| LLM is the interface | LLM is the accelerator |
| Errors fixed through more prompting | Errors fixed with one click in form |
| "Did it understand me?" anxiety | What you see is what you get |
| Hard to review 10+ line items | Table view handles complexity visually |
| No visual output until sent | PDF preview builds confidence before sending |
| Chat history = implicit state | Form = explicit, visible state |

---

## 2. Target Market

### 2.1 Primary Segment

**Profile:** Self-employed contractors, tradespeople, and freelancers in WhatsApp-dominant markets who currently quote on paper, voice note, or typed WhatsApp text.

| Characteristic | Detail |
|----------------|--------|
| Business size | 1–10 people |
| Quotes per month | 5–50 |
| Current tool | Paper, WhatsApp text, Google Docs |
| Tech comfort | Low to medium |
| Primary device | Android smartphone |
| Language | Not necessarily English |

### 2.2 Secondary Segment

**Profile:** Small agencies and service businesses (10–50 employees) seeking to standardize quoting across their team, with a preference for web-first workflows but needing WhatsApp for client-facing delivery.

### 2.3 Geographic Strategy

| Phase | Markets | Rationale |
|-------|---------|-----------|
| **Phase 1 (MVP)** | India, UAE, Singapore | Large WhatsApp penetration, English-comfortable, established SaaS payment habits |
| **Phase 2** | Saudi Arabia, Malaysia, Indonesia, Philippines | High WhatsApp usage, growing SaaS adoption |
| **Phase 3** | LATAM (Brazil, Mexico, Colombia) | WhatsApp dominant, large contractor market |
| **Phase 4** | Africa, broader MENA | Mobile-first, underserved by SaaS |

English-speaking Western markets (US, UK, Canada) are not the primary target. They are served adequately by PandaDoc and Proposify.

### 2.4 User Personas

#### Persona 1: Arjun — Electrical Contractor, Pune, India
- **Age:** 38
- **Tech:** Android phone, uses WhatsApp for all business comms
- **Pain:** Quotes customers verbally or via voice note; clients don't take him seriously; loses track of quotes
- **Goal:** Send a professional PDF quote via WhatsApp while still on-site
- **Willingness to pay:** ₹500–800/month (~$6–10)
- **Channel:** WhatsApp-first

#### Persona 2: Fatima — Interior Designer, Dubai, UAE
- **Age:** 31
- **Tech:** iPhone, web-comfortable, uses WhatsApp for client comms
- **Pain:** Spends 45 min per quote in Canva/Google Docs; no quote tracking
- **Goal:** Create quotes in under 5 minutes, track if clients viewed them
- **Willingness to pay:** AED 70–120/month (~$19–33)
- **Channel:** Web chat + WhatsApp delivery

#### Persona 3: Marcus — IT Services Agency, Singapore
- **Age:** 44
- **Tech:** High, manages a 12-person team
- **Pain:** Team uses different formats, no pricing consistency, no conversion data
- **Goal:** Standardized templates, team management, basic analytics
- **Willingness to pay:** SGD 60–100/month (~$45–75)
- **Channel:** Web UI primarily

---

## 3. Feature Requirements

### 3.1 MVP Features (Month 1–3)

The MVP must prove one thing: **users can create and send a professional quote in under 2 minutes.** Everything else is secondary.

#### 3.1.1 AI Chat Interface + Editable Form + Live Preview

| Feature | Description | Priority |
|---------|-------------|----------|
| Conversational quote creation | Create quotes via natural language in web chat | P0 |
| Structured JSON output | LLM outputs validated JSON, not free-form text | P0 |
| Editable form UI | AI output populates an editable line-item form | P0 |
| Live PDF preview | Preview updates within 200ms of any form change | P0 |
| Multi-turn conversation | Handle corrections and additions across messages | P0 |
| MCP tool integration | AI reads product catalog and contacts via the same APIs as the UI | P0 |
| Direct form usage | Users can skip chat and fill form manually | P0 |

**Structured Output Schema:**
```json
{
  "customer": {
    "id": "uuid-or-null",
    "name": "Raj Kumar",
    "company": "Infra Ltd",
    "email": "raj@infra.com",
    "phone": "+91-9876543210"
  },
  "lineItems": [
    {
      "productId": "uuid-or-null",
      "description": "Civil Survey",
      "quantity": 3,
      "unit": "day",
      "unitPrice": 8500,
      "discountPercent": 0,
      "taxRate": 18
    }
  ],
  "discount": {
    "type": "percent",
    "value": 0
  },
  "validDays": 14,
  "notes": "",
  "currency": "INR"
}
```

#### 3.1.2 Product/Service Catalog

| Feature | Description | Priority |
|---------|-------------|----------|
| Product catalog | Define services with name, description, default price | P0 |
| Unit of measure | Hours, days, units, sq ft, kg, custom | P0 |
| Pricing rules | Base price, simple volume tiers | P0 |
| Categories | Organize by service type | P1 |
| Cost tracking (margin) | Track cost vs. selling price | P2 |

#### 3.1.3 Quote Generation

| Feature | Description | Priority |
|---------|-------------|----------|
| Line items | Multiple products/services per quote | P0 |
| Quantity and UOM | Per-item quantities with units | P0 |
| Line-level discounts | Discount per line item | P0 |
| Quote-level discount | Overall percentage or fixed discount | P0 |
| Tax calculation | Manual rate entry; GST/VAT label per region | P0 |
| Validity period | Set expiration date | P0 |
| Notes field | Customer-facing notes on quote | P0 |
| Internal notes | Notes not visible on PDF | P1 |
| Terms & conditions | Default or custom T&C block | P1 |

#### 3.1.4 PDF Generation and Live Preview

| Feature | Description | Priority |
|---------|-------------|----------|
| Live PDF preview | Client-side rendering (react-pdf), real-time update | P0 |
| PDF export | Download final PDF | P0 |
| 1 professional template | Clean, locale-appropriate default | P0 |
| Company logo upload | Logo placement in header | P0 |
| Company contact details | Address, phone, email, website | P0 |
| Payment info block | Bank details or payment terms | P0 |
| Quote numbering | Auto-increment, configurable prefix | P0 |
| 2nd template option | Alternative design | P1 |
| Custom fields | Additional labeled fields | P2 |
| Visual template editor | Drag-drop customization | Out of scope (V2) |

#### 3.1.5 Sharing and Distribution

| Feature | Description | Priority |
|---------|-------------|----------|
| WhatsApp share | One-tap share: PDF + link to web view via WhatsApp | P0 |
| Email sharing | Send quote via email with PDF attachment | P0 |
| Shareable link | Unique URL for client to view quote in browser | P0 |
| PDF download | Direct download | P0 |
| View tracking | Know when client opened the link | P1 |
| Quote acceptance | Client can click "Accept" on web view | P1 |
| E-signature | Digital signature on acceptance | Out of scope (V2) |

#### 3.1.6 Contact Management (Lite CRM)

| Feature | Description | Priority |
|---------|-------------|----------|
| Contact records | Name, company, email, phone, country | P0 |
| Quote history per contact | All quotes sent to this contact | P0 |
| Quote status | Draft, Sent, Viewed, Accepted, Rejected, Expired | P0 |
| Quote search | Search by customer name, date, amount | P0 |
| Quote duplication | Clone existing quote as starting point | P0 |
| CSV contact import | Bulk import existing contacts | P1 |
| Lead pipeline | Kanban view: lead stages | Out of scope (V2) |

#### 3.1.7 Business Profile and Settings

| Feature | Description | Priority |
|---------|-------------|----------|
| Company profile | Name, logo, address, industry | P0 |
| Default currency | Set primary currency | P0 |
| Default tax rate | Pre-fill tax on new quotes | P0 |
| Default validity period | Pre-fill days valid | P0 |
| Preferred language | UI language (English for MVP) | P0 |

---

### 3.2 Phase 2 Features (Month 4–6)

#### 3.2.1 WhatsApp Business API Integration

This is the primary growth vector. Moving from "share to WhatsApp" (a link) to a true two-way WhatsApp experience.

| Feature | Description | Priority |
|---------|-------------|----------|
| WhatsApp inbound | Receive quote requests in WhatsApp, process via AI | P0 |
| WhatsApp outbound | Send formatted quote card + PDF via WhatsApp | P0 |
| Conversation context | AI maintains context across multi-message WA threads | P0 |
| WhatsApp bot for clients | Clients can reply "accept" or "reject" on WhatsApp | P1 |

**WhatsApp MVP flow:**
```
Client WhatsApp: "Can you quote me for 50m2 floor tiling?"
     ↓
QuoteFlow AI processes via LLM (reads user's catalog)
     ↓
AI replies in WhatsApp: "Here's your quote [PDF link].
  50m2 floor tiling at ₹180/sqft = ₹9,000. Valid 14 days."
     ↓
Client replies: "Accepted"
     ↓
Quote status updates to Accepted in dashboard
```

#### 3.2.2 Multi-Language Support

| Phase | Languages |
|-------|-----------|
| Phase 2 | Hindi, Arabic |
| Phase 3 | Tamil, Bahasa, Tagalog, Portuguese |
| Phase 4 | 10+ languages |

#### 3.2.3 Team Features

| Feature | Description | Priority |
|---------|-------------|----------|
| Multiple users | Add team members with assigned roles | P1 |
| Role-based access | Admin, Sales Rep, View-only | P1 |
| Shared product catalog | Team uses consistent pricing | P1 |
| Quote assignment | Assign quotes to team members | P1 |

#### 3.2.4 Analytics

| Feature | Description | Priority |
|---------|-------------|----------|
| Quote conversion rate | Sent → Accepted % | P1 |
| Revenue pipeline | Value of open quotes | P1 |
| Top products | Most-quoted services | P1 |
| Response time tracking | Time from sent to viewed/accepted | P1 |

---

### 3.3 Phase 3 Features (Month 7–12)

- Telegram bot integration
- Mobile app (PWA first, then React Native)
- Document training (upload past quotes to train AI on terminology/pricing)
- Accounting integrations (Zoho Books, QuickBooks, Tally)
- API for third-party integrations
- E-signatures
- Invoice conversion (quote → invoice)
- Multi-currency per quote

---

## 4. Pricing Strategy

### 4.1 Pricing Philosophy

Priced for emerging markets. The Western SaaS default of $19–49/mo is inaccessible for a contractor in Pune or Jakarta. We use local-currency pricing at purchasing-power-adjusted rates.

### 4.2 Tier Structure

| Tier | Price (USD) | India (INR) | UAE (AED) | Quotes/Month | Key Features |
|------|-------------|-------------|-----------|--------------|--------------|
| **Free** | $0 | ₹0 | AED 0 | 5 | Web chat, 1 template, WhatsApp share link |
| **Solo** | $7/mo | ₹499/mo | AED 25/mo | Unlimited | All templates, email, WA link, contact CRM |
| **Pro** | $15/mo | ₹999/mo | AED 55/mo | Unlimited | + View tracking, acceptance, basic analytics |
| **Team** | $35/mo | ₹2,499/mo | AED 130/mo | Unlimited | + 5 users, shared catalog, team analytics |
| **Enterprise** | Custom | Custom | Custom | Unlimited | + API, SSO, white-label, dedicated support |

### 4.3 WhatsApp API Pricing

WhatsApp Business API (two-way inbound/outbound) is a Pro+ feature, not an add-on. It is included in Pro and Team tiers with a message quota:

| Tier | Included WhatsApp Conversations/Month | Overage |
|------|---------------------------------------|---------|
| Solo | 0 (WA share link only) | N/A |
| Pro | 200 conversations | $0.03/conversation |
| Team | 500 conversations | $0.02/conversation |

**Note:** Meta charges ~$0.01–0.04 per conversation depending on country. We absorb this into the plan pricing at moderate volumes and pass overage through at cost + 50% margin.

---

## 5. Localization

### 5.1 Currency Support

- 50 currencies at launch covering Phase 1–2 markets
- ISO 4217 standard
- Correct symbol placement and number formatting per locale
- Real-time exchange rates (optional, Phase 2)

### 5.2 Tax and Compliance

| Region | Tax System | MVP Support |
|--------|-----------|-------------|
| India | GST (5%, 12%, 18%, 28%) + HSN codes | Phase 1 |
| UAE | VAT 5% | Phase 1 |
| Singapore | GST 9% | Phase 1 |
| Saudi Arabia | VAT 15% | Phase 2 |
| EU | VAT (variable by country) | Phase 3 |
| US | State sales tax | Phase 3 |

### 5.3 Date and Number Formats

- DD/MM/YYYY default for Phase 1 markets (configurable)
- Indian number system support (1,00,000 vs 100,000)
- RTL layout support for Arabic (Phase 2)

---

## 6. Non-Functional Requirements

### 6.1 Performance

| Metric | Target |
|--------|--------|
| Chat response time | < 2 seconds |
| Chat → form render | < 500ms |
| Live preview update | < 200ms |
| PDF export generation | < 5 seconds |
| Page load (mobile 4G) | < 3 seconds |
| API response time | < 500ms (p95) |
| Uptime | 99.9% |

### 6.2 Mobile-First

- All web UI designed mobile-first (primary device is Android smartphone)
- Touch-friendly form controls
- PDF preview optimized for small screens
- WhatsApp deep links work on both iOS and Android

### 6.3 Security

- GDPR compliant (for EU users)
- Data encryption at rest (AES-256)
- Data encryption in transit (TLS 1.3)
- Role-based access control
- Audit logging for team accounts
- SOC 2 Type II (target Year 2)

### 6.4 Scalability

- Target: 10,000 MAU at end of Year 1
- Multi-region deployment: India region (Asia South) from launch
- Horizontal scaling on compute; managed DB with read replicas

---

## 7. Go-To-Market Strategy

### 7.1 Launch Market: India

India is the launch market for three reasons:
1. 500M+ WhatsApp users; WhatsApp is the default business communication tool
2. Massive contractor and freelancer economy (construction, IT services, creative)
3. Established SaaS payment habits (Razorpay, UPI); demonstrated by Zoho and Freshworks success

### 7.2 Acquisition Channels

| Channel | Tactic | Target |
|---------|--------|--------|
| WhatsApp communities | Share in contractor/freelancer WhatsApp groups | Viral, zero cost |
| YouTube (vernacular) | Short demo videos in Hindi, Tamil, Telugu | SEO + awareness |
| Facebook groups | Indian contractor/freelancer communities | Paid + organic |
| Referral program | 1 month free for each referred paying user | Viral loop |
| Partner onboarding | Partner with CA firms, trade associations | B2B2C |

### 7.3 Conversion Strategy

- Free tier is genuinely useful (5 quotes/month covers many early-stage users)
- Upgrade trigger: quote limit reached OR view tracking needed
- Annual pricing at 20% discount to improve LTV
- Local payment methods: UPI, Razorpay, PayPal for international

---

## 8. Success Metrics

### 8.1 Product Metrics (6-Month Targets)

| Metric | Target |
|--------|--------|
| Monthly Active Users | 3,000 |
| Quotes Generated | 25,000/month |
| Median time to first quote | < 3 minutes |
| Chat-initiated quotes | 55% |
| Direct form usage | 45% |
| Quote-to-send rate | 75% |
| 30-day retention | 60% |

### 8.2 Business Metrics (Year 1 Targets)

| Metric | Target |
|--------|--------|
| Free users | 8,000 |
| Paid subscribers | 800 |
| MRR | $12,000 |
| Avg revenue per paid user | $15/mo |
| CAC | < $25 |
| LTV (3-year) | > $400 |
| Churn (monthly) | < 5% |

---

## 9. Release Roadmap

### Phase 1: MVP (Months 1–3) — Prove Core Value

**Goal:** Users can create and send a professional quote in under 2 minutes.

- Web chat → editable form → live PDF preview
- Product catalog (manual entry)
- Quote management (status, search, duplicate)
- Basic contact records + quote history
- Sharing: WhatsApp link, email, PDF download
- 1 PDF template
- Business profile + settings (currency, tax, logo)
- India GST + UAE VAT tax modes
- INR and AED currency support

**Cut from MVP:** WhatsApp Business API inbound, team features, analytics, multi-language, Telegram

### Phase 2: Growth (Months 4–6) — Activate WhatsApp Channel

**Goal:** WhatsApp is a first-class quoting channel, not just a share button.

- WhatsApp Business API (inbound + outbound)
- Hindi and Arabic language support
- Team features (multi-user, roles)
- Quote acceptance flow (client-facing web view)
- View tracking + notifications
- Basic analytics dashboard
- 2nd PDF template
- Expand currencies to 50

### Phase 3: Scale (Months 7–12) — Deepen and Expand

**Goal:** Expand to LATAM and Southeast Asia; increase ARPU.

- Portuguese and Bahasa language support
- Mobile app (PWA → React Native)
- Invoice conversion (quote → invoice)
- Document training (upload past quotes)
- Accounting integrations (Zoho Books, Tally)
- Telegram bot
- API for integrations
- E-signatures

---

## 10. Risks and Mitigations

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Meta WhatsApp API cost increases or policy changes | High | Medium | Multi-channel fallback (SMS, email); don't over-index on WA API for monetization |
| LLM output errors on financial data | High | Medium | Chat-to-UI Pipeline makes errors non-critical; validation on JSON schema; user confirms before sending |
| Incumbents (PandaDoc, Zoho) build WhatsApp features | Medium | Medium | Speed advantage; they won't price for ₹499/mo; community moat |
| Low willingness to pay in target markets | High | Low | Validated by Zoho/Freshworks; freemium model; UPI/low-friction payments |
| WhatsApp Business API verification delays | Medium | High | Onboard with WA share link first; API as Phase 2; use verified BSP partner |
| Tax compliance complexity per region | Medium | Medium | Phased rollout; manual rate entry for MVP; auto-calculation in Phase 2 |
| AI hallucination in quote pricing | High | Low | MCP reads actual product catalog; structured output validated against schema; form review before send |

---

## 11. Technical Stack Guidance

| Layer | Recommendation | Rationale |
|-------|---------------|-----------|
| Frontend | Next.js + Tailwind | SSR for mobile performance; familiar ecosystem |
| PDF preview | react-pdf (client-side) | Instant preview without server round-trip |
| PDF export | Puppeteer or react-pdf server | Pixel-perfect final output |
| AI framework | Mastra.ai + OpenRouter | MCP tool integration; model flexibility |
| LLM | Claude Sonnet (via OpenRouter) | Strong structured output; function calling |
| Backend | Node.js + Hono or Fastify | Lightweight; good for serverless |
| Database | PostgreSQL (Supabase or Neon) | Relational model fits quote/line-item structure |
| Auth | Clerk or Supabase Auth | Fast to ship; supports social login |
| WhatsApp | Meta Cloud API via 360dialog or Twilio | BSP partnership for faster verification |
| Payments | Razorpay (India) + Stripe (international) | Local payment methods critical for India |
| Hosting | Vercel + Fly.io (Asia South region) | Low latency for primary market |

---

## 12. Appendix

### 12.1 Glossary

| Term | Definition |
|------|------------|
| MCP | Model Context Protocol — structured AI tool calling; allows LLM to read/write product catalog and contacts via the same APIs as the UI |
| UOM | Unit of Measure |
| BSP | Business Solution Provider — Meta-authorized WhatsApp API partner |
| Quote | Price proposal sent to a potential customer |
| Invoice | Bill issued after work is completed (Phase 3 feature) |
| Chat-to-UI Pipeline | Architecture where LLM output populates an editable form, not a free-text response |
| GST | Goods and Services Tax (India, Singapore, Australia) |
| WA API | WhatsApp Business API — enables two-way programmatic messaging |

### 12.2 Key References

- Meta WhatsApp Business API Documentation
- Mastra.ai Framework Documentation
- OpenRouter API Documentation
- ISO 4217 Currency Codes
- India GST Rate Schedule

---

**Document Owner:** Product Team
**Last Updated:** March 6, 2026
**Next Review:** April 6, 2026
