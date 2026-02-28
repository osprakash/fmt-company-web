# Unit Economics & Credit System Design
## ClipFlow.ai — Detailed Financial Model

**Version:** 2.0  
**Date:** February 28, 2026  
**Status:** Internal — Founder Reference  
**Depends on:** PRD v4.0, TECH_SPEC v3.0

---

## 1. Executive Summary

ClipFlow's **hybrid architecture** creates a structurally advantaged cost model:
- **Browser processing (80% of users)**: Video processing happens on the user's device — near-zero cost to serve
- **Cloud processing (20% of users)**: Larger videos upload to servers — costs credits, generates margin

Revenue comes from:
1. **Output-gating** (exports, resolution, watermark) — subscription tiers
2. **Cloud processing credits** — for videos >20 min / >400MB
3. **AI credits** (Phase 2) — for server-side AI features

| Metric | ClipFlow | OpusClip (est.) |
|--------|----------|-----------------|
| Cost per free user/month | **$0.004** | $0.50–1.50 |
| Cost per cloud processing min | **$0.002** | $0.05–0.10 |
| Gross margin — Starter ($9) | **91%** | 45–55% |
| Gross margin — Pro ($19) | **88%** | 40–50% |
| Gross margin — Business ($49) | **85%** | 35–45% |
| Blended gross margin (at scale) | **~88%** | ~45% |

The hybrid model gives us the best of both worlds: near-zero cost for short videos (browser), and a revenue stream from longer videos (cloud).

---

## 2. Credit System Design

### 2.1 What Is a Credit?

A credit is ClipFlow's universal currency for **server-side operations** that cannot run in the browser. Credits decouple the user-facing pricing from the underlying GPU cost, allowing flexible pricing and bundling.

**Credits are used for two purposes:**
1. **Cloud processing** (MVP) — for videos >20 min / >400MB that can't process in browser
2. **AI features** (Phase 2) — for server-side AI operations like clip detection, B-roll generation

### 2.2 Browser vs Cloud Processing — Credit Usage

| Feature | Browser Processing | Cloud Processing |
|---------|-------------------|------------------|
| **When used** | Videos <20 min, <400MB | Videos >20 min or >400MB |
| **Credit cost** | **0 credits** | **2 credits/min of input video** |
| **Our cost** | $0 | ~$0.002/min |
| **User experience** | Instant, private | Upload wait, then fast |

### 2.3 MVP Features — Browser Processing (Zero Credits)

| MVP Feature | Processing Location | Credit Cost | Rationale |
|-------------|-------------------|-------------|-----------|
| Smart Captions | Browser (Whisper Web) | 0 | Runs on user's device — costs us nothing |
| Aspect Ratio Reframing | Browser (MediaPipe + FFmpeg.wasm) | 0 | Same |
| Video Trimming & Export | Browser (FFmpeg.wasm) | 0 | Same |
| Transcript-Based Editing | Browser (Whisper + FFmpeg.wasm) | 0 | Same |
| Filler Word & Silence Removal | Browser (Web Audio API) | 0 | Same |
| Brand Templates | Browser (IndexedDB) + Supabase sync | 0 | Negligible server cost |

**This is the competitive moat.** For videos under 20 min, we cost nothing to serve. OpusClip charges for every minute.

### 2.4 Cloud Processing — Credit Costs (MVP)

| Operation | Credits | GPU Cost | Margin |
|-----------|---------|----------|--------|
| Cloud processing (per min of input) | 2 | $0.002 | 99% |

**Example:** A 45-minute video costs 90 credits (45 × 2) = $9 at retail ($0.10/credit). Our cost: $0.09. Margin: 99%.

**Why 2 credits/min?**
- Simple mental math for users
- Covers GPU + storage + bandwidth with massive margin
- Competitive with server-based alternatives (OpusClip charges ~$0.15-0.30/min effective)

### 2.3 What Consumes Credits (Phase 2)

| AI Operation | What It Does | Credits Consumed | GPU Time per Unit | GPU Cost per Unit |
|-------------|-------------|-----------------|-------------------|-------------------|
| **AI Clip Detection** | ML model analyzes video for viral-worthy moments | 5 per minute of input video | ~3s A100 per min | $0.0035 |
| **AI B-Roll Generation** | Generates context-relevant stock footage/images | 10 per clip generated | ~8s A100 per clip | $0.0092 |
| **AI Voice Enhancement** | Removes background noise, enhances speech clarity | 3 per minute of audio | ~1.5s A100 per min | $0.0017 |
| **4K Export (server)** | Server-side re-encode at 4K resolution | 2 per minute of output | ~4s A100 per min | $0.0046 |
| **Auto-Chapters** | AI-generated chapter markers from transcript | 2 per video | ~2s A100 per video | $0.0023 |

### 2.4 GPU Cost Assumptions

Based on current (Feb 2026) cloud GPU pricing:

| Provider | GPU | Cost per Second | Cost per Hour | Notes |
|----------|-----|----------------|---------------|-------|
| Replicate | A100 (40GB) | $0.00115 | $4.14 | Pay-per-second, cold start ~5s |
| Modal | A100 (40GB) | $0.001058 | $3.81 | Pay-per-second, warm containers |
| RunPod | A100 (40GB) | $0.00089 | $3.22 | Serverless, variable availability |
| Lambda | A100 (80GB) | $0.00097 | $3.49 | Reserved instances, better for scale |

**Baseline used in this document:** $0.00115/s (Replicate) for conservative modeling. At scale (>$5K/mo GPU spend), negotiate to ~$0.00085/s via committed use.

### 2.5 Cost per Credit — Blended Analysis

Not all credits are equal in cost. The cost depends on which operation the user chooses.

| Operation | Credits | GPU Cost | Cost per Credit |
|-----------|---------|----------|----------------|
| AI Clip Detection (1 min) | 5 | $0.0035 | $0.0007 |
| AI B-Roll (1 clip) | 10 | $0.0092 | $0.00092 |
| Voice Enhancement (1 min) | 3 | $0.0017 | $0.00057 |
| 4K Export (1 min) | 2 | $0.0046 | $0.0023 |
| Auto-Chapters (1 video) | 2 | $0.0023 | $0.00115 |

**Expected usage mix** (based on comparable products):

| Operation | % of Total Credits Used | Weighted Cost/Credit |
|-----------|------------------------|---------------------|
| AI Clip Detection | 35% | $0.000245 |
| AI B-Roll Generation | 20% | $0.000184 |
| Voice Enhancement | 25% | $0.000143 |
| 4K Export | 15% | $0.000345 |
| Auto-Chapters | 5% | $0.0000575 |
| **Blended** | **100%** | **$0.000974** |

**Blended cost per credit: ~$0.001** (one-tenth of a cent).

### 2.6 Setting Credit Retail Price

The retail price of a credit should satisfy three constraints:

1. **Gross margin ≥70%** on credit revenue (target: 90%+)
2. **Psychologically simple** — easy mental math for users
3. **Competitive with server-based alternatives** — must feel cheaper than OpusClip's per-minute pricing

| Pricing Option | Retail per Credit | Gross Margin | 300 Credits Value | Feels Like |
|---------------|-------------------|-------------|-------------------|------------|
| $0.05 | 5¢ | 98.1% | $15 | Cheap — undervalues AI |
| **$0.10** | **10¢** | **99.0%** | **$30** | **Clean math, premium but fair** |
| $0.15 | 15¢ | 99.4% | $45 | Expensive — discourages usage |

**Recommended: $0.10 per credit (base rate).** This gives:
- Simple mental math: 1 credit = 10 cents
- AI Clip Detection on a 10-min video: 50 credits = $5.00 (vs. OpusClip's $15/mo for the same feature locked behind a paywall)
- B-Roll generation: 10 credits = $1.00 per clip (reasonable for AI-generated content)

### 2.7 Credit Allocation per Tier

| | Free | Starter | Pro | Business |
|--|------|---------|-----|----------|
| Monthly credits | 0 | **50** | **300** | **1,000** |
| Retail value of included credits | $0 | $5 | $30 | $100 |
| Cloud processing included | No | Yes (uses credits) | Yes (uses credits) | **Yes (included free)** |
| Actual GPU cost of credits (at 70% usage) | $0 | $0.07 | $0.42 | $1.40 |
| Credits roll over? | — | No | No | No |
| Can purchase top-ups? | No | Yes | Yes | Yes |

**What changed from v1.0:**
- **Starter now gets 50 credits** — enables cloud processing for occasional longer videos
- **Business gets cloud processing included** — no credit cost for cloud, just AI features

**Why Starter gets 50 credits (changed from 0):**
- Cloud processing is an MVP feature, not Phase 2
- Starter users with occasional 30-min videos need a path to process them
- 50 credits = 25 minutes of cloud processing/month (reasonable for occasional use)
- Still maintains clear upgrade path to Pro (300 credits) for heavier users

**Why credits don't roll over:**
- Prevents credit hoarding and unpredictable cost spikes
- Creates monthly urgency ("use it or lose it")
- Standard in SaaS (Canva, Jasper, most AI tools)
- Exception: purchased top-up credits never expire

### 2.8 Credit Top-Up Pricing

| Package | Credits | Price | Per-Credit Price | Discount | Margin |
|---------|---------|-------|-----------------|----------|--------|
| Small | 50 | $5.00 | $0.100 | 0% (base) | 99.0% |
| Medium | 150 | $12.00 | $0.080 | 20% | 98.8% |
| Large | 500 | $35.00 | $0.070 | 30% | 98.6% |

**Top-up credits never expire.** This is important — users who pay cash for credits feel cheated if they disappear. Subscription credits expire monthly (they're included "for free"), but purchased credits persist.

### 2.9 Credit Budget — What Can Users Actually Do?

**Starter (50 credits/month):**

| Scenario | Operations | Credits Used | Remaining |
|----------|-----------|-------------|-----------|
| **Browser-only user** | All videos <20 min | 0 | 50 |
| **Occasional long video** | 1 × 25-min video via cloud | 50 | 0 |
| **Two long videos** | 2 × 20-min videos via cloud | 80 | Needs top-up |

**Pro (300 credits/month):**

| Scenario | Operations | Credits Used | Remaining |
|----------|-----------|-------------|-----------|
| **Mostly browser** | 90% browser + 2 × 30-min cloud | 120 | 180 |
| **Mixed workflow** | 50% browser + 4 × 45-min cloud | 360 | Needs top-up |
| **Heavy cloud user** | 6 × 60-min videos via cloud | 720 | Needs upgrade to Business |

**Business (1,000 credits/month + cloud included):**

| Scenario | Operations | Credits Used | Remaining |
|----------|-----------|-------------|-----------|
| **Cloud processing** | Unlimited (included) | 0 | 1,000 |
| **AI features (Phase 2)** | 10 × AI clip detection (30 min) | 1,500 | Needs top-up |

**Target: 70-80% of users stay within their credit allocation.** The 20-30% who exceed it either:
- Buy a top-up ($5-35 incremental revenue)
- Upgrade to next tier

---

## 3. Infrastructure Cost Breakdown

### 3.1 Fixed Costs (Monthly Baseline)

These costs are incurred regardless of user count.

| Service | Plan | Monthly Cost | Purpose | Scales At |
|---------|------|-------------|---------|-----------|
| Vercel | Pro | $20 | Hosting, edge functions, SSR | 100K+ visitors/mo |
| Supabase | Pro | $25 | Auth, PostgreSQL, 8GB storage | 50K+ users |
| Stripe | — | $0 (usage-based) | Payment processing | — |
| Cloudflare | Free | $0 | CDN for model files (~200MB) | 1M+ downloads |
| PostHog | Cloud Free | $0 | Analytics (1M events/mo free) | 50K MAU |
| Sentry | Developer | $0 | Error tracking (5K events/mo) | 10K errors/mo |
| Domain + DNS | — | $15 | clipflow.ai domain | — |
| Transactional Email | Resend | $0 | Auth emails (3K/mo free) | 3K emails/mo |
| **Total baseline** | | **$60/mo** | | |

**At scale, fixed costs grow to:**

| Stage | Users | Monthly Fixed Cost | Notes |
|-------|-------|--------------------|-------|
| Pre-launch | 0-100 | $60 | Free tiers of everything |
| Early traction | 100-5K | $60-100 | Still mostly on free/starter plans |
| Growth | 5K-50K | $200-600 | Supabase Pro, Vercel Pro, PostHog paid |
| Scale | 50K-200K | $600-2,000 | Higher Supabase tier, Vercel Enterprise negotiation |
| At scale | 200K+ | $2,000-5,000 | Volume discounts kick in |

### 3.2 Variable Costs per User (Monthly)

#### Free User

| Cost Item | Amount | Notes |
|-----------|--------|-------|
| Supabase DB row storage | $0.0001 | ~5 rows per user (profile, 2-3 projects) |
| Supabase Auth | $0.0000 | Included in plan |
| API calls (serverless functions) | $0.0005 | ~50 calls/mo (save project, auth) |
| CDN — model file download (amortized) | $0.003 | ~200MB download, cached after first visit |
| CDN — static assets | $0.0002 | JS bundles, images |
| **Total per free user/month** | **$0.004** | |

A free user costs less than half a cent per month. **This is the structural advantage.** OpusClip's free user costs $0.50-1.50/mo in GPU time for 60 minutes of server-side processing.

#### Starter User ($9/mo)

| Cost Item | Amount | Notes |
|-----------|--------|-------|
| Everything in Free | $0.004 | |
| Stripe processing fee | $0.561 | 2.9% × $9 + $0.30 |
| Supabase Storage (template sync) | $0.001 | <1MB JSON templates |
| Additional API calls | $0.002 | Template sync, project cloud save |
| Cloud processing (35 credits avg used) | $0.035 | 50 × 70% usage × $0.001 cost/credit |
| **Total per Starter user/month** | **$0.603** | |

Almost the entire cost is the Stripe fee. Cloud processing adds minimal cost.

#### Pro User ($19/mo)

| Cost Item | Amount | Notes |
|-----------|--------|-------|
| Everything in Free | $0.004 | |
| Stripe processing fee | $0.851 | 2.9% × $19 + $0.30 |
| Cloud storage (10GB allocated, ~3GB avg used) | $0.075 | Supabase Storage at $0.025/GB |
| AI credits — GPU compute (210 credits avg) | $0.204 | 300 × 70% usage × $0.000974 blended cost |
| Additional API calls | $0.005 | AI endpoints, cloud sync |
| **Total per Pro user/month** | **$1.139** | |

#### Business User ($49/mo)

| Cost Item | Amount | Notes |
|-----------|--------|-------|
| Everything in Free | $0.004 | |
| Stripe processing fee | $1.721 | 2.9% × $49 + $0.30 |
| Cloud storage (50GB allocated, ~15GB avg used) | $0.375 | Supabase Storage at $0.025/GB |
| AI credits — GPU compute (700 credits avg) | $0.682 | 1000 × 70% usage × $0.000974 |
| Team features (additional auth seats) | $0.010 | 3 seats, negligible |
| API access overhead | $0.020 | Higher rate limits, more calls |
| **Total per Business user/month** | **$2.812** | |

### 3.3 Variable Cost Summary

| Tier | Revenue | COGS | Gross Profit | Gross Margin |
|------|---------|------|-------------|-------------|
| Free | $0.00 | $0.004 | -$0.004 | N/A (acquisition) |
| Starter ($9) | $9.00 | $0.60 | **$8.40** | **93.3%** |
| Pro ($19) | $19.00 | $1.14 | **$17.86** | **94.0%** |
| Business ($49) | $49.00 | $2.81 | **$46.19** | **94.3%** |
| Credit Top-Up ($5) | $5.00 | $0.05 | **$4.95** | **99.0%** |
| Credit Top-Up ($12) | $12.00 | $0.12 | **$11.88** | **99.0%** |
| Credit Top-Up ($35) | $35.00 | $0.35 | **$34.65** | **99.0%** |

**The margins are exceptional because:**
1. **Browser processing costs $0** — 80% of users never touch our servers
2. **Cloud processing is highly profitable** — 99% margin on credits
3. **Stripe fees are the main cost** — not compute

---

## 4. Unit Economics Deep Dive

### 4.1 Lifetime Value (LTV) by Tier

**Assumptions:**

| Parameter | Starter | Pro | Business |
|-----------|---------|-----|----------|
| Monthly churn rate | 8% | 5% | 3% |
| Avg lifetime (1/churn) | 12.5 months | 20 months | 33 months |
| Monthly top-up revenue | $0 | $3.50 (30% buy) | $8.00 (40% buy) |
| Expansion rate (upgrades) | 5% upgrade to Pro | 2% upgrade to Business | — |

**LTV Calculation:**

```
LTV = (ARPU × Gross Margin) × Average Lifetime

Starter:
  ARPU = $9.00 + $0.00 (no top-ups) = $9.00
  Gross Margin = 93.7%
  Lifetime = 12.5 months
  LTV = $9.00 × 0.937 × 12.5 = $105.34

Pro:
  ARPU = $19.00 + $3.50 (top-ups) = $22.50
  Gross Margin = 91.8% (blended with top-ups)
  Lifetime = 20 months
  LTV = $22.50 × 0.918 × 20 = $413.10

Business:
  ARPU = $49.00 + $8.00 (top-ups) = $57.00
  Gross Margin = 93.1% (blended with top-ups)
  Lifetime = 33 months
  LTV = $57.00 × 0.931 × 33 = $1,751.15
```

| Tier | LTV | LTV:CAC Target (3:1) | Max CAC |
|------|-----|---------------------|---------|
| Starter | **$105** | 3:1 | **$35** |
| Pro | **$413** | 3:1 | **$138** |
| Business | **$1,751** | 3:1 | **$584** |

### 4.2 Customer Acquisition Cost (CAC) Targets

**Organic Acquisition (Phase 1 — first 6 months):**

| Channel | Est. Monthly Spend | Users Acquired | CAC | Notes |
|---------|-------------------|---------------|-----|-------|
| Product Hunt | $0 (time only) | 2,000-5,000 | $0 | One-time launch spike |
| SEO content (blog) | $500 (writer) | 200-500 | $1-2.50 | Compounds over time |
| Twitter/X organic | $0 | 100-300 | $0 | Founder-led content |
| YouTube tutorials | $200 (production) | 300-800 | $0.25-0.67 | High-intent traffic |
| Reddit (r/NewTubers, etc.) | $0 | 50-200 | $0 | Community engagement |
| **Blended organic** | **$700** | **3,000-6,000** | **$0.12-0.23** | |

**Paid Acquisition (Phase 2 — after product-market fit):**

| Channel | CPC | CVR to Signup | CVR to Paid | CAC (to paid) | Viable? |
|---------|-----|--------------|-------------|---------------|---------|
| Google Ads ("video editor") | $2.50 | 8% | 8% | $390 | No — too expensive |
| Google Ads ("opus clip alternative") | $1.80 | 15% | 10% | $120 | Yes for Pro |
| Facebook/Instagram Ads | $0.80 | 5% | 8% | $200 | Marginal |
| YouTube Pre-roll | $0.15 (CPV) | 3% | 8% | $62.50 | Yes for all tiers |
| TikTok Ads | $0.50 | 4% | 8% | $156 | Yes for Pro/Business |
| Creator partnerships | $500/creator | 500 signups | 8% | $12.50 | Best ROI |
| **Blended paid** | — | — | — | **$50-80** | |

**Target blended CAC (organic + paid): $15-30**

At $15-30 CAC:
- Starter LTV:CAC = 3.5-7.0x (healthy)
- Pro LTV:CAC = 13.8-27.5x (exceptional)
- Business LTV:CAC = 58-117x (exceptional)

### 4.3 Payback Period

How quickly does a paying user's gross profit recover their acquisition cost?

```
Payback Period = CAC / (Monthly Gross Profit)

At $25 blended CAC:
  Starter: $25 / $8.43 = 3.0 months
  Pro:     $25 / $17.86 = 1.4 months
  Business: $25 / $46.19 = 0.5 months
```

| Tier | Payback Period | Target (<12 mo) | Status |
|------|---------------|-----------------|--------|
| Starter | **3.0 months** | < 12 months | Excellent |
| Pro | **1.4 months** | < 12 months | Exceptional |
| Business | **0.5 months** | < 12 months | Exceptional |

---

## 5. Revenue Model at Scale

### 5.1 Revenue Mix Projection (Month 12)

Based on PRD v3.0 projections (150K users, 50K MAE):

| Revenue Stream | Users | Revenue/User | Monthly Revenue | % of MRR |
|---------------|-------|-------------|----------------|----------|
| Starter subscriptions | 4,000 | $9 | $36,000 | 52% |
| Pro subscriptions | 1,000 | $19 | $19,000 | 27% |
| Business subscriptions | 100 | $49 | $4,900 | 7% |
| Credit top-ups (Pro) | 300 | $8.50 avg | $2,550 | 4% |
| Credit top-ups (Business) | 40 | $15 avg | $600 | 1% |
| Annual plan premium (20% discount, paid upfront) | ~500 | — | $6,000 | 9% |
| **Total MRR** | | | **$69,050** | **100%** |

**ARR: $828,600**

### 5.2 Cost Structure at Month 12

| Cost Category | Monthly Cost | % of Revenue |
|--------------|-------------|-------------|
| **COGS** | | |
| Stripe fees (blended) | $4,100 | 5.9% |
| AI GPU compute (credits) | $350 | 0.5% |
| Cloud storage (Supabase) | $200 | 0.3% |
| CDN bandwidth | $100 | 0.1% |
| **Total COGS** | **$4,750** | **6.9%** |
| | | |
| **Fixed Infrastructure** | | |
| Vercel hosting | $150 | 0.2% |
| Supabase (Pro plan) | $75 | 0.1% |
| PostHog analytics | $450 | 0.7% |
| Sentry monitoring | $80 | 0.1% |
| Domain + email services | $30 | 0.0% |
| **Total Fixed Infra** | **$785** | **1.1%** |
| | | |
| **Gross Profit** | **$63,515** | **92.0%** |
| | | |
| **Operating Expenses** | | |
| Engineering (2 FTE) | $25,000 | 36.2% |
| Marketing (content + ads) | $5,000 | 7.2% |
| Customer support (1 part-time) | $2,000 | 2.9% |
| SaaS tools (Figma, Linear, etc.) | $300 | 0.4% |
| **Total OpEx** | **$32,300** | **46.8%** |
| | | |
| **Net Operating Income** | **$31,215** | **45.2%** |

### 5.3 The Infrastructure Cost Advantage — Visualized

How ClipFlow's cost per user compares at different scales:

```
Cost per Active User per Month

         ClipFlow                     OpusClip (estimated)
         ────────                     ────────────────────
Free     $0.004                       $0.50 - $1.50
         ▓                            ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓

Starter  $0.57 (93% = Stripe fee)     $1.20 - $2.50 (GPU + Stripe)
         ▓▓▓▓▓                        ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓

Pro      $1.14                        $4.00 - $8.00 (heavy GPU)
         ▓▓▓▓▓▓▓▓                     ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓

Free users at 100K scale:
  ClipFlow: 100K × $0.004 = $400/mo
  OpusClip: 100K × $1.00  = $100,000/mo
                             ^^^^^^^^^^^^
                             This is why OpusClip limits their free tier.
```

---

## 6. Sensitivity Analysis

### 6.1 What If AI GPU Costs Are 3x Higher?

If blended cost per credit is $0.003 instead of $0.001:

| Tier | New COGS | New Gross Margin | Impact |
|------|----------|-----------------|--------|
| Free | $0.004 (unchanged) | N/A | None |
| Starter | $0.57 (unchanged) | 93.7% | None — no AI credits |
| Pro | $1.55 | 91.8% | -2.2 pts (still excellent) |
| Business | $3.76 | 92.3% | -2.0 pts (still excellent) |

**Verdict:** Even at 3x GPU costs, margins stay above 90%. The credit system has massive headroom.

### 6.2 What If Credit Usage Is 100% (Not 70%)?

If every Pro user burns all 300 credits:

| Tier | Credits Used | GPU Cost | New COGS | Gross Margin |
|------|-------------|----------|----------|-------------|
| Pro | 300 | $0.292 | $1.23 | 93.5% |
| Business | 1,000 | $0.974 | $3.10 | 93.7% |

**Verdict:** Even at 100% credit utilization, margins stay above 93%. Credits are priced with extreme headroom.

### 6.3 What If Stripe Fees Increase to 3.5%?

| Tier | New Stripe Fee | New Gross Margin | Impact |
|------|---------------|-----------------|--------|
| Starter | $0.615 | 93.2% | -0.5 pts |
| Pro | $0.965 | 93.0% | -1.0 pts |
| Business | $2.015 | 93.3% | -1.0 pts |

**Verdict:** Negligible impact. Stripe fees are small relative to the subscription price.

### 6.4 What If Churn Is 2x Worse?

| Tier | Churn | Lifetime | New LTV | LTV:CAC (at $25 CAC) |
|------|-------|----------|---------|----------------------|
| Starter | 16% | 6.25 mo | $52.67 | 2.1x (below target) |
| Pro | 10% | 10 mo | $206.55 | 8.3x (healthy) |
| Business | 6% | 16.7 mo | $875.57 | 35x (excellent) |

**Verdict:** Starter becomes marginal at 2x churn. Invest in onboarding and first-week experience to keep Starter churn below 10%.

---

## 7. Credit System for Starter Tier — Should You Add It?

### 7.1 The Case For Credits on Starter

| Argument | Details |
|----------|---------|
| Revenue expansion | Even 50 credits at $0.10 = $5/mo additional value included |
| Reduces upgrade friction | Users can sample AI features before committing to Pro |
| Competitive | Some competitors include basic AI even on lower tiers |

### 7.2 The Case Against (Recommended)

| Argument | Details |
|----------|---------|
| Muddies the value prop | Starter = "remove watermark + 1080p + 25 exports." Simple. Adding credits confuses this. |
| Creates support burden | "Why did my credits run out? What do credits do?" — support tickets from confused users |
| Reduces Pro upgrade incentive | If Starter gets 50 credits, Pro's 300 credits feels like only 6x more for 2x the price |
| Margin preservation | Starter at 93.7% margin is your cash cow. Don't add COGS. |
| Phase 2 timing | AI features aren't in MVP. Don't pre-announce a credit system that has no features to use it on. |

### 7.3 Recommendation

**Do NOT add credits to Starter in Phase 1 or Phase 2.**

Instead, use a **"taste test" approach** when AI features launch:
- Give Starter users **3 free AI operations per month** (not credits — specific actions)
- Example: "Try AI Clip Detection on 1 video, Voice Enhancement on 1 video, and 1 B-Roll clip — free each month"
- This lets Starter users experience AI features and creates upgrade desire, without introducing credit complexity

If data shows >15% of Starter users want more AI features, then consider adding 50 credits to Starter in Phase 3.

---

## 8. Break-Even Analysis

### 8.1 Monthly Break-Even (Covering Fixed + OpEx)

```
Monthly burn (pre-revenue):
  Engineering (2 FTE):     $25,000
  Fixed infra:                $785
  Marketing:               $3,000
  Misc (tools, legal):       $500
  ────────────────────────────────
  Total monthly burn:      $29,285

Break-even MRR needed:
  $29,285 / 0.92 (gross margin) = $31,832 MRR

Subscribers needed (assuming 70% Starter, 25% Pro, 5% Business):
  Blended ARPU = 0.70 × $9 + 0.25 × $19 + 0.05 × $49 = $6.30 + $4.75 + $2.45 = $13.50
  Paying users needed: $31,832 / $13.50 = ~2,358 paying users

At 8% free-to-paid conversion:
  Total registered users needed: 2,358 / 0.08 = ~29,475 users
```

| Milestone | Users Needed | Timeline (est.) |
|-----------|-------------|----------------|
| Infrastructure break-even (fixed costs only) | ~500 paying users (~6,250 total) | Month 3-4 |
| Full break-even (incl. salaries) | ~2,358 paying users (~29,475 total) | Month 6-8 |
| Profitability ($10K+/mo net) | ~4,000 paying users (~50,000 total) | Month 9-12 |

### 8.2 Runway Analysis

| Funding | Monthly Burn | Runway | Users at End (est.) |
|---------|-------------|--------|-------------------|
| $0 (bootstrapped) | $5,000 (solo founder) | Indefinite if day job | 5K-10K |
| $50K pre-seed | $29,285 | 1.7 months | Not enough |
| $150K pre-seed | $29,285 | 5.1 months | 20K-30K |
| $500K seed | $29,285 | 17 months | 100K-200K |
| $500K seed (lean: $15K/mo) | $15,000 | 33 months | 150K+ |

**Recommendation:** Raise $300-500K seed. Run lean ($15-20K/mo burn) until 1,000 paying users. Then hire and scale marketing.

---

## 9. Pricing Comparison — ClipFlow vs. Competitors

### 9.1 Feature-Adjusted Value Comparison

| Feature | OpusClip ($15/mo) | ClipFlow Starter ($9/mo) | ClipFlow Pro ($19/mo) |
|---------|-------------------|------------------------|-----------------------|
| Exports/month | 90 min processing | 25 exports | Unlimited |
| Captions | Yes | Yes | Yes |
| Transcript editing | No | Yes | Yes |
| Filler removal | No | Yes | Yes |
| Text overlays | Yes (limited) | Yes | Yes |
| Brand kit | $29/mo plan | Yes (10 templates) | Yes (unlimited) |
| AI clip detection | Yes | No | Yes (300 credits) |
| Max resolution | 1080p | 1080p | 1080p (4K via credits) |
| Watermark | Yes (free), No (paid) | No | No |
| Cloud storage | Yes | Sync only | 10GB |
| **Effective $/feature** | **$1.88/feature** | **$1.13/feature** | **$1.73/feature** |

ClipFlow Starter offers **40% more value per dollar** than OpusClip's entry tier. This is the conversion pitch.

---

## 10. Key Metrics to Track

### 10.1 Credit System Health Metrics

| Metric | Target | Red Flag |
|--------|--------|----------|
| Avg credit utilization (Pro) | 60-80% | <30% (credits overpriced/useless) or >95% (underpriced) |
| Avg credit utilization (Business) | 50-70% | <25% or >90% |
| Top-up purchase rate (Pro) | 20-35% | <10% (no perceived value) |
| Top-up purchase rate (Business) | 30-45% | <15% |
| Revenue from top-ups as % of MRR | 5-10% | <2% (credits not driving revenue) |
| Credit-related support tickets | <5% of total | >15% (system too confusing) |

### 10.2 Unit Economics Health Metrics

| Metric | Target | Red Flag |
|--------|--------|----------|
| Blended gross margin | >88% | <80% |
| Starter gross margin | >90% | <85% |
| Pro gross margin | >85% | <75% |
| LTV:CAC ratio (blended) | >3:1 | <2:1 |
| Payback period (blended) | <6 months | >12 months |
| Free → Paid conversion | >8% | <4% |
| Monthly churn (Starter) | <8% | >12% |
| Monthly churn (Pro) | <5% | >8% |
| GPU cost as % of revenue | <3% | >10% |

---

## 11. Risks to Unit Economics

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| GPU costs spike (AI demand) | Pro/Business margins drop 5-10 pts | Low | Pre-negotiate committed use pricing; adjust credit costs if needed |
| Stripe raises fees | All margins drop 1-2 pts | Low | Minimal impact; switch to Paddle/LemonSqueezy if needed |
| Free users never convert | No revenue, just CDN costs | Medium | Output-gating creates natural conversion; monitor export limit hit rate |
| AI credits underutilized | Top-up revenue underperforms | Medium | Make AI features genuinely useful and visible in the UI |
| High churn on Starter | LTV drops below CAC | Medium | Invest in onboarding, first-export experience, weekly email nudges |
| Competition undercuts pricing | Forced to lower prices | Medium | Browser-first cost structure allows price cuts competitors can't match |
| Supabase pricing changes | Fixed costs increase | Low | Easily migrate to self-hosted Postgres + Auth0 |

---

## 12. Summary — Decision Framework

### The Credit Equation

```
Credit Retail Price    = $0.10 (clean, simple, memorable)
Credit Cost (blended)  = $0.001 (one-tenth of a cent)
Credit Gross Margin    = 99%
Included Monthly (Pro) = 300 ($30 retail value, $0.29 actual cost)
Included Monthly (Biz) = 1,000 ($100 retail value, $0.97 actual cost)
```

### The Tier Equation

```
Starter ($9/mo):   All features + no watermark + 1080p + 25 exports
                   COGS: $0.57 | Margin: 93.7% | LTV: $105

Pro ($19/mo):      Everything in Starter + unlimited exports + 300 AI credits + 10GB storage
                   COGS: $1.14 | Margin: 94.0% | LTV: $413

Business ($49/mo): Everything in Pro + 1000 AI credits + 50GB + 3 seats + 4K + API
                   COGS: $2.81 | Margin: 94.3% | LTV: $1,751
```

### The Strategic Advantage

The browser-first architecture gives ClipFlow **90%+ gross margins at every tier** while offering a more generous free product than any competitor. This creates a flywheel:

1. Generous free tier → high signup volume (low CAC)
2. Near-zero cost to serve free users → sustainable growth before revenue
3. Output-gating at moment of highest perceived value → strong conversion
4. 93%+ margins on paid tiers → fast payback, long runway
5. Credits priced at 99% margin → Phase 2 AI features are pure upside

OpusClip's server-based model means every user (free or paid) burns GPU time. They can never offer what ClipFlow offers at ClipFlow's margins. **The cost structure is the moat.**

---

## 13. Hybrid Model Economics — Browser vs Cloud

### 13.1 The 80/20 Split

Based on our target market (short-form creators with 5-20 min source videos):

| Processing Mode | % of Users | % of Processing Minutes | Our Cost |
|-----------------|-----------|------------------------|----------|
| Browser | 80% | 70% | $0 |
| Cloud | 20% | 30% | ~$0.002/min |

**Why 80% browser:**
- Our target is 5-20 min videos (explicitly stated in PRD)
- These fit comfortably in browser limits (200-400MB)
- Only users with occasional longer content need cloud

### 13.2 Cloud Processing Unit Economics

| Input Video Length | Credits | User Pays | Our Cost | Margin |
|-------------------|---------|-----------|----------|--------|
| 30 min | 60 | $6.00 | $0.06 | 99% |
| 60 min | 120 | $12.00 | $0.12 | 99% |
| 2 hours | 240 | $24.00 | $0.24 | 99% |

**Cloud processing is a profit center, not a cost center.**

### 13.3 Why This Model Beats Pure Server-Side

| Metric | ClipFlow (Hybrid) | OpusClip (Server-Only) |
|--------|------------------|----------------------|
| Cost for 10-min video | $0 (browser) | ~$0.50-1.00 (GPU) |
| Cost for 60-min video | $0.12 (cloud) | ~$3.00-6.00 (GPU) |
| Free tier sustainability | Infinite (costs nothing) | Limited (burns GPU) |
| Margin on cloud processing | 99% | 40-60% |

**The hybrid model lets us:**
1. Offer unlimited free browser processing (acquisition)
2. Charge premium for cloud processing (monetization)
3. Maintain 90%+ margins across all tiers

---

*Document maintained by: Founder*  
*Last updated: February 28, 2026*  
*Review quarterly or after any pricing/infrastructure change*
