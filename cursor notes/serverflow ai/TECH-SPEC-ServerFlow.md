# ServerFlow.ai - Technical Specification Document

**Product Name:** ServerFlow.ai  
**Company:** Flowmind Technologies  
**Version:** 1.0  
**Last Updated:** February 23, 2026  
**Document Owner:** Engineering Team

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Architecture](#architecture)
3. [Component Specifications](#component-specifications)
4. [Data Models](#data-models)
5. [API Specifications](#api-specifications)
6. [Security Architecture](#security-architecture)
7. [Infrastructure Requirements](#infrastructure-requirements)
8. [Integration Specifications](#integration-specifications)
9. [Development Roadmap](#development-roadmap)
10. [Appendices](#appendices)

---

## 1. System Overview

### 1.1 Purpose

ServerFlow.ai is a multi-tenant SaaS platform that provides AI-powered infrastructure management through a conversational interface. The system abstracts the complexity of SaltStack while leveraging its mature execution framework for reliable remote operations.

### 1.2 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT LAYER                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│  Web Dashboard    │    Mobile App     │    CLI Tool    │    API Clients     │
└────────┬──────────┴─────────┬─────────┴───────┬────────┴─────────┬──────────┘
         │                    │                 │                  │
         └────────────────────┼─────────────────┼──────────────────┘
                              │                 │
                              ▼                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API GATEWAY                                     │
│                    (Authentication, Rate Limiting, Routing)                  │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
         ┌────────────────────────┼────────────────────────┐
         │                        │                        │
         ▼                        ▼                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────────┐
│   AI Service    │    │  Core Platform  │    │     Real-time Service       │
│                 │    │                 │    │                             │
│ • Query Engine  │◄──►│ • User Mgmt     │◄──►│ • WebSocket Gateway         │
│ • NLP Pipeline  │    │ • Org Mgmt      │    │ • Event Streaming           │
│ • LLM Gateway   │    │ • Billing       │    │ • Live Updates              │
│ • Intent Parser │    │ • Audit Logs    │    │ • Notification Dispatch     │
└────────┬────────┘    └────────┬────────┘    └──────────────┬──────────────┘
         │                      │                            │
         └──────────────────────┼────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ORCHESTRATION LAYER                                  │
│                                                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐  │
│  │  Salt Master    │  │  Job Scheduler  │  │    State Manager            │  │
│  │  (per tenant)   │  │                 │  │                             │  │
│  │                 │  │ • Cron jobs     │  │ • Desired state tracking    │  │
│  │ • Command exec  │  │ • Maintenance   │  │ • Drift detection           │  │
│  │ • State apply   │  │ • Batch ops     │  │ • Rollback management       │  │
│  │ • Event bus     │  │                 │  │                             │  │
│  └────────┬────────┘  └─────────────────┘  └─────────────────────────────┘  │
│           │                                                                  │
└───────────┼──────────────────────────────────────────────────────────────────┘
            │
            │ ZeroMQ (encrypted)
            │
            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           AGENT LAYER                                        │
│                                                                              │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐ │
│   │   Agent      │   │   Agent      │   │   Agent      │   │   Agent      │ │
│   │ (Salt Minion)│   │ (Salt Minion)│   │ (Salt Minion)│   │ (Salt Minion)│ │
│   │              │   │              │   │              │   │              │ │
│   │  Linux VM    │   │  Windows VM  │   │  Container   │   │  Cloud VM    │ │
│   └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘ │
│                                                                              │
│   Customer Infrastructure (On-prem / Cloud / Hybrid)                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Technology Stack

| Layer | Technology | Rationale |
|-------|------------|-----------|
| **Frontend** | Next.js 14, React 18, TypeScript | Modern React framework, SSR, great DX |
| **UI Components** | Tailwind CSS, Radix UI | Rapid development, accessible components |
| **API Gateway** | Kong / AWS API Gateway | Rate limiting, auth, routing |
| **Backend Services** | Python 3.12, FastAPI | Async support, Salt compatibility |
| **AI/ML** | LangChain, OpenAI GPT-4, Claude | Flexible LLM orchestration |
| **Orchestration** | SaltStack 3006 | Mature, reliable remote execution |
| **Message Queue** | Redis Streams, RabbitMQ | Event-driven architecture |
| **Database** | PostgreSQL 16 | ACID compliance, JSON support |
| **Time-series DB** | TimescaleDB | Metrics storage, efficient queries |
| **Cache** | Redis 7 | Session, query cache, pub/sub |
| **Search** | Elasticsearch 8 | Log indexing, full-text search |
| **Object Storage** | S3 / MinIO | Artifacts, backups, exports |
| **Container Runtime** | Kubernetes (EKS/GKE) | Scalability, self-healing |
| **Monitoring** | Prometheus, Grafana | Platform observability |
| **CI/CD** | GitHub Actions, ArgoCD | GitOps deployment |

---

## 2. Architecture

### 2.1 Multi-Tenancy Model

ServerFlow uses a **hybrid multi-tenancy** approach:

```
┌─────────────────────────────────────────────────────────────────┐
│                    SHARED INFRASTRUCTURE                         │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │ API Gateway  │  │ AI Service   │  │ Shared Databases     │   │
│  │ (shared)     │  │ (shared)     │  │ (tenant isolation)   │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   TENANT-ISOLATED COMPONENTS                     │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Tenant A                                                 │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │    │
│  │  │ Salt Master  │  │ Event Stream │  │ Secrets Vault│   │    │
│  │  │ (dedicated)  │  │ (isolated)   │  │ (isolated)   │   │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Tenant B                                                 │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │    │
│  │  │ Salt Master  │  │ Event Stream │  │ Secrets Vault│   │    │
│  │  │ (dedicated)  │  │ (isolated)   │  │ (isolated)   │   │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Isolation guarantees:**
- Each tenant gets dedicated Salt Master (no cross-tenant command execution)
- Database row-level security with tenant_id
- Separate encryption keys per tenant
- Network isolation via Kubernetes namespaces

### 2.2 Service Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              SERVICES MAP                                    │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  auth-service   │     │  user-service   │     │  org-service    │
│                 │     │                 │     │                 │
│ • JWT issuance  │     │ • User CRUD     │     │ • Organization  │
│ • OAuth2/OIDC   │     │ • Preferences   │     │ • Teams         │
│ • MFA           │     │ • API keys      │     │ • Invitations   │
│ • Session mgmt  │     │                 │     │ • Roles/Perms   │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                                 ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  chat-service   │     │   ai-service    │     │ execution-svc   │
│                 │     │                 │     │                 │
│ • Conversation  │────►│ • NLP pipeline  │────►│ • Salt bridge   │
│ • History       │     │ • Intent parse  │     │ • Job queue     │
│ • Context mgmt  │◄────│ • LLM calls     │◄────│ • Result proc   │
│ • Streaming     │     │ • Response gen  │     │ • Rollback      │
└─────────────────┘     └─────────────────┘     └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                                 ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ inventory-svc   │     │  metrics-svc    │     │   logs-svc      │
│                 │     │                 │     │                 │
│ • Server list   │     │ • Collection    │     │ • Aggregation   │
│ • Auto-discover │     │ • Storage       │     │ • Search        │
│ • Grouping      │     │ • Alerting      │     │ • Retention     │
│ • Tags/Labels   │     │ • Dashboards    │     │ • Export        │
└─────────────────┘     └─────────────────┘     └─────────────────┘

┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  audit-service  │     │ billing-service │     │ notification-svc│
│                 │     │                 │     │                 │
│ • Change log    │     │ • Usage track   │     │ • Email         │
│ • Compliance    │     │ • Stripe integ  │     │ • Slack         │
│ • Reports       │     │ • Invoicing     │     │ • Webhooks      │
│ • Export        │     │ • Tier enforce  │     │ • In-app        │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

### 2.3 Data Flow: Query to Execution

```
User Query: "Why is prod-db-01 slow?"
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. CHAT SERVICE                                                  │
│    • Receive message via WebSocket                               │
│    • Load conversation context                                   │
│    • Forward to AI Service                                       │
└─────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. AI SERVICE - Intent Recognition                               │
│                                                                  │
│    Input: "Why is prod-db-01 slow?"                             │
│    │                                                             │
│    ├─► Intent: DIAGNOSE_PERFORMANCE                              │
│    ├─► Target: prod-db-01                                        │
│    ├─► Scope: [cpu, memory, disk, network, processes, queries]   │
│    └─► Confidence: 0.94                                          │
└─────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. AI SERVICE - Query Planning                                   │
│                                                                  │
│    Generate execution plan:                                      │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │ {                                                        │  │
│    │   "plan_id": "diag-001",                                │  │
│    │   "steps": [                                             │  │
│    │     {"module": "status.cpuinfo", "target": "prod-db-01"},│  │
│    │     {"module": "status.meminfo", "target": "prod-db-01"},│  │
│    │     {"module": "disk.usage", "target": "prod-db-01"},    │  │
│    │     {"module": "ps.top", "args": {"n": 10}},            │  │
│    │     {"module": "mysql.status", "target": "prod-db-01"},  │  │
│    │     {"module": "mysql.slow_queries", "args": {"n": 5}}   │  │
│    │   ],                                                     │  │
│    │   "parallel": true                                       │  │
│    │ }                                                        │  │
│    └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. EXECUTION SERVICE                                             │
│                                                                  │
│    • Validate plan against permissions                           │
│    • Submit to Salt Master                                       │
│    • Stream results as they arrive                               │
│                                                                  │
│    Salt Commands Executed:                                       │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │ salt 'prod-db-01' status.cpuinfo                        │  │
│    │ salt 'prod-db-01' status.meminfo                        │  │
│    │ salt 'prod-db-01' disk.usage                            │  │
│    │ salt 'prod-db-01' ps.top n=10                           │  │
│    │ salt 'prod-db-01' mysql.status                          │  │
│    │ salt 'prod-db-01' mysql.slow_queries n=5                │  │
│    └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. AI SERVICE - Response Generation                              │
│                                                                  │
│    Raw Results:                                                  │
│    • CPU: 94% usage, mysql process consuming 89%                 │
│    • Memory: 78% used, 2GB swap active                          │
│    • Disk I/O: 450 IOPS (high)                                  │
│    • Slow queries: 3 queries > 10s                              │
│    │                                                             │
│    ▼                                                             │
│    LLM Analysis + Response Generation                            │
│    │                                                             │
│    ▼                                                             │
│    Structured Response:                                          │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │ {                                                        │  │
│    │   "summary": "prod-db-01 is slow due to...",            │  │
│    │   "findings": [...],                                     │  │
│    │   "recommendations": [...],                              │  │
│    │   "actions": [                                           │  │
│    │     {"label": "Kill slow query", "action_id": "..."},   │  │
│    │     {"label": "Add index", "action_id": "..."}          │  │
│    │   ]                                                      │  │
│    │ }                                                        │  │
│    └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. CHAT SERVICE - Response Delivery                              │
│                                                                  │
│    • Stream formatted response to client                         │
│    • Render action buttons                                       │
│    • Store in conversation history                               │
│    • Log to audit trail                                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Component Specifications

### 3.1 ServerFlow Agent (Salt Minion Wrapper)

The agent is a branded Salt minion with custom configuration and additional telemetry collection.

#### 3.1.1 Agent Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    SERVERFLOW AGENT                              │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                   Agent Wrapper                          │    │
│  │                                                          │    │
│  │  • Auto-registration                                     │    │
│  │  • Health reporting                                      │    │
│  │  • Update management                                     │    │
│  │  • Crash recovery                                        │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│                            ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                   Salt Minion Core                       │    │
│  │                                                          │    │
│  │  • ZeroMQ transport (encrypted)                          │    │
│  │  • Module execution engine                               │    │
│  │  • State management                                      │    │
│  │  • Event publishing                                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│                            ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │               Custom ServerFlow Modules                  │    │
│  │                                                          │    │
│  │  • serverflow.inventory    (system discovery)            │    │
│  │  • serverflow.metrics      (telemetry collection)        │    │
│  │  • serverflow.security     (security scanning)           │    │
│  │  • serverflow.logs         (log forwarding)              │    │
│  │  • serverflow.backup       (backup verification)         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### 3.1.2 Agent Installation

**Linux (One-liner):**
```bash
curl -sSL https://get.serverflow.ai | sudo bash -s -- --token=<REGISTRATION_TOKEN>
```

**Windows (PowerShell):**
```powershell
iwr -useb https://get.serverflow.ai/win | iex; Install-ServerFlowAgent -Token "<REGISTRATION_TOKEN>"
```

#### 3.1.3 Agent Configuration

```yaml
# /etc/serverflow/agent.conf (Linux)
# C:\ProgramData\ServerFlow\agent.conf (Windows)

serverflow:
  tenant_id: "tenant_abc123"
  registration_token: "tok_xxxxx"
  
  master:
    host: "master-abc123.serverflow.ai"
    port: 4506
    
  telemetry:
    interval: 60  # seconds
    metrics:
      - cpu
      - memory
      - disk
      - network
      - processes
      
  logs:
    enabled: true
    paths:
      - /var/log/syslog
      - /var/log/auth.log
      - /var/log/nginx/*.log
    max_lines_per_minute: 1000
    
  security:
    scan_interval: 3600  # hourly
    
  updates:
    auto_update: true
    channel: stable
```

#### 3.1.4 Agent Resource Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 0.1 core | 0.25 core |
| Memory | 64 MB | 128 MB |
| Disk | 100 MB | 500 MB |
| Network | Outbound 4505-4506 | Same |

### 3.2 AI Service

#### 3.2.1 NLP Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                      NLP PIPELINE                                │
└─────────────────────────────────────────────────────────────────┘

User Input: "Show me servers with high CPU in the web tier"
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│ Stage 1: PREPROCESSING                                           │
│                                                                  │
│ • Tokenization                                                   │
│ • Spelling correction                                            │
│ • Entity extraction (server names, metrics, thresholds)          │
│                                                                  │
│ Output: {                                                        │
│   "tokens": ["show", "servers", "high", "cpu", "web", "tier"],  │
│   "entities": {                                                  │
│     "metric": "cpu",                                             │
│     "threshold": "high",                                         │
│     "group": "web tier"                                          │
│   }                                                              │
│ }                                                                │
└─────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│ Stage 2: INTENT CLASSIFICATION                                   │
│                                                                  │
│ Classifier: Fine-tuned transformer model                         │
│                                                                  │
│ Intent Categories:                                               │
│ • QUERY_METRICS      (read-only, metrics data)                  │
│ • QUERY_INVENTORY    (read-only, server info)                   │
│ • QUERY_LOGS         (read-only, log search)                    │
│ • DIAGNOSE           (read-only, analysis)                      │
│ • REMEDIATE          (write, requires approval)                 │
│ • CONFIGURE          (write, requires approval)                 │
│ • DEPLOY             (write, requires approval)                 │
│ • REPORT             (read-only, generate report)               │
│                                                                  │
│ Output: {                                                        │
│   "intent": "QUERY_METRICS",                                    │
│   "confidence": 0.92,                                            │
│   "requires_approval": false                                     │
│ }                                                                │
└─────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│ Stage 3: CONTEXT ENRICHMENT                                      │
│                                                                  │
│ Add context from:                                                │
│ • Conversation history (last 10 messages)                        │
│ • User's server inventory                                        │
│ • Recent alerts/incidents                                        │
│ • User preferences and permissions                               │
│                                                                  │
│ Output: {                                                        │
│   "resolved_targets": ["web-01", "web-02", "web-03", "web-04"], │
│   "user_permissions": ["read", "execute"],                      │
│   "recent_context": "User asked about web tier 5 min ago"       │
│ }                                                                │
└─────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│ Stage 4: QUERY PLANNING (LLM)                                    │
│                                                                  │
│ Prompt Template:                                                 │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ You are an infrastructure query planner. Given the user's   │ │
│ │ request and available Salt modules, generate an execution   │ │
│ │ plan.                                                        │ │
│ │                                                              │ │
│ │ User request: {user_input}                                   │ │
│ │ Intent: {intent}                                             │ │
│ │ Target servers: {targets}                                    │ │
│ │ Available modules: {module_list}                             │ │
│ │                                                              │ │
│ │ Generate a JSON execution plan...                            │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Output: Execution plan (see Section 2.3)                         │
└─────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│ Stage 5: RESPONSE GENERATION (LLM)                               │
│                                                                  │
│ After execution results received:                                │
│                                                                  │
│ Prompt Template:                                                 │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ You are a helpful IT assistant. Analyze the execution       │ │
│ │ results and provide a clear, actionable response.           │ │
│ │                                                              │ │
│ │ User question: {user_input}                                  │ │
│ │ Execution results: {results}                                 │ │
│ │                                                              │ │
│ │ Provide:                                                     │ │
│ │ 1. Summary of findings                                       │ │
│ │ 2. Specific issues identified                                │ │
│ │ 3. Recommended actions (if applicable)                       │ │
│ │ 4. Suggested follow-up questions                             │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Output: Formatted response with action buttons                   │
└─────────────────────────────────────────────────────────────────┘
```

#### 3.2.2 LLM Configuration

```python
# ai_service/config.py

LLM_CONFIG = {
    "primary": {
        "provider": "openai",
        "model": "gpt-4-turbo",
        "temperature": 0.1,  # Low for consistency
        "max_tokens": 4096,
        "timeout": 30,
    },
    "fallback": {
        "provider": "anthropic",
        "model": "claude-3-sonnet",
        "temperature": 0.1,
        "max_tokens": 4096,
        "timeout": 30,
    },
    "embedding": {
        "provider": "openai",
        "model": "text-embedding-3-small",
        "dimensions": 1536,
    },
    "rate_limits": {
        "requests_per_minute": 60,
        "tokens_per_minute": 100000,
    },
    "caching": {
        "enabled": True,
        "ttl": 300,  # 5 minutes for similar queries
        "similarity_threshold": 0.95,
    }
}
```

#### 3.2.3 Intent-to-Module Mapping

```python
# ai_service/module_registry.py

MODULE_REGISTRY = {
    "QUERY_METRICS": {
        "cpu": ["status.cpuinfo", "ps.top"],
        "memory": ["status.meminfo", "ps.top"],
        "disk": ["disk.usage", "disk.inodeusage"],
        "network": ["network.interfaces", "network.netstat"],
        "processes": ["ps.top", "ps.pgrep"],
    },
    "QUERY_INVENTORY": {
        "system": ["grains.items", "pkg.list_pkgs"],
        "services": ["service.get_all", "service.status"],
        "users": ["user.list_users", "shadow.info"],
        "network": ["network.interfaces", "network.routes"],
    },
    "DIAGNOSE": {
        "performance": ["status.all_status", "ps.top", "disk.usage"],
        "connectivity": ["network.ping", "network.traceroute"],
        "service_health": ["service.status", "cmd.run_all"],
        "security": ["pkg.list_upgrades", "file.stats"],
    },
    "REMEDIATE": {
        "restart_service": ["service.restart"],
        "kill_process": ["ps.kill_pid"],
        "clear_disk": ["file.remove", "cmd.run"],
        "apply_updates": ["pkg.upgrade"],
        "modify_config": ["file.managed", "file.replace"],
    },
    "CONFIGURE": {
        "install_package": ["pkg.install"],
        "create_user": ["user.present"],
        "firewall": ["iptables.append", "firewalld.add_port"],
        "cron": ["cron.present"],
    },
}
```

### 3.3 Execution Service

#### 3.3.1 Salt Master Integration

```python
# execution_service/salt_bridge.py

import salt.client
import salt.config
from typing import Dict, List, Any
import asyncio

class SaltBridge:
    """Bridge between ServerFlow and Salt Master"""
    
    def __init__(self, tenant_id: str):
        self.tenant_id = tenant_id
        self.master_config = self._load_tenant_config(tenant_id)
        self.local_client = salt.client.LocalClient(
            mopts=self.master_config
        )
        
    async def execute(
        self,
        targets: List[str],
        module: str,
        args: Dict[str, Any] = None,
        timeout: int = 60
    ) -> Dict[str, Any]:
        """Execute Salt module on targets"""
        
        # Validate targets belong to tenant
        validated_targets = await self._validate_targets(targets)
        
        # Build target expression
        target_expr = self._build_target_expression(validated_targets)
        
        # Execute asynchronously
        jid = self.local_client.cmd_async(
            tgt=target_expr,
            fun=module,
            arg=args.get('args', []) if args else [],
            kwarg=args.get('kwargs', {}) if args else {},
            tgt_type='list'
        )
        
        # Wait for results with streaming
        results = await self._collect_results(jid, timeout)
        
        return {
            "job_id": jid,
            "targets": validated_targets,
            "module": module,
            "results": results,
            "timestamp": datetime.utcnow().isoformat()
        }
    
    async def execute_plan(
        self,
        plan: ExecutionPlan,
        stream_callback: callable = None
    ) -> Dict[str, Any]:
        """Execute a multi-step plan"""
        
        results = []
        
        if plan.parallel:
            # Execute all steps in parallel
            tasks = [
                self.execute(
                    targets=step.targets,
                    module=step.module,
                    args=step.args
                )
                for step in plan.steps
            ]
            results = await asyncio.gather(*tasks)
        else:
            # Execute sequentially
            for step in plan.steps:
                result = await self.execute(
                    targets=step.targets,
                    module=step.module,
                    args=step.args
                )
                results.append(result)
                
                if stream_callback:
                    await stream_callback(result)
                    
                # Check for failures if step is critical
                if step.critical and not result.get('success'):
                    break
                    
        return {
            "plan_id": plan.id,
            "results": results,
            "status": "completed"
        }
```

#### 3.3.2 Job Queue Management

```python
# execution_service/job_queue.py

from celery import Celery
from redis import Redis

celery_app = Celery(
    'serverflow',
    broker='redis://redis:6379/0',
    backend='redis://redis:6379/1'
)

celery_app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='UTC',
    enable_utc=True,
    task_routes={
        'execution.*': {'queue': 'execution'},
        'telemetry.*': {'queue': 'telemetry'},
        'ai.*': {'queue': 'ai'},
    },
    task_default_rate_limit='100/m',
)

@celery_app.task(bind=True, max_retries=3)
def execute_salt_command(
    self,
    tenant_id: str,
    targets: List[str],
    module: str,
    args: Dict = None
):
    """Execute Salt command as background task"""
    try:
        bridge = SaltBridge(tenant_id)
        result = asyncio.run(
            bridge.execute(targets, module, args)
        )
        return result
    except Exception as exc:
        self.retry(exc=exc, countdown=2 ** self.request.retries)
```

### 3.4 Real-time Service

#### 3.4.1 WebSocket Gateway

```python
# realtime_service/websocket_gateway.py

from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import Dict, Set
import json

app = FastAPI()

class ConnectionManager:
    """Manage WebSocket connections per tenant/user"""
    
    def __init__(self):
        # tenant_id -> user_id -> Set[WebSocket]
        self.connections: Dict[str, Dict[str, Set[WebSocket]]] = {}
        
    async def connect(
        self,
        websocket: WebSocket,
        tenant_id: str,
        user_id: str
    ):
        await websocket.accept()
        
        if tenant_id not in self.connections:
            self.connections[tenant_id] = {}
        if user_id not in self.connections[tenant_id]:
            self.connections[tenant_id][user_id] = set()
            
        self.connections[tenant_id][user_id].add(websocket)
        
    async def broadcast_to_tenant(
        self,
        tenant_id: str,
        message: dict
    ):
        """Broadcast message to all users in tenant"""
        if tenant_id in self.connections:
            for user_connections in self.connections[tenant_id].values():
                for ws in user_connections:
                    await ws.send_json(message)
                    
    async def send_to_user(
        self,
        tenant_id: str,
        user_id: str,
        message: dict
    ):
        """Send message to specific user"""
        if (tenant_id in self.connections and 
            user_id in self.connections[tenant_id]):
            for ws in self.connections[tenant_id][user_id]:
                await ws.send_json(message)

manager = ConnectionManager()

@app.websocket("/ws/{tenant_id}/{user_id}")
async def websocket_endpoint(
    websocket: WebSocket,
    tenant_id: str,
    user_id: str
):
    await manager.connect(websocket, tenant_id, user_id)
    
    try:
        while True:
            data = await websocket.receive_json()
            
            # Route message to appropriate handler
            message_type = data.get("type")
            
            if message_type == "chat":
                await handle_chat_message(
                    tenant_id, user_id, data, websocket
                )
            elif message_type == "subscribe":
                await handle_subscription(
                    tenant_id, user_id, data
                )
            elif message_type == "action":
                await handle_action(
                    tenant_id, user_id, data, websocket
                )
                
    except WebSocketDisconnect:
        manager.disconnect(websocket, tenant_id, user_id)
```

#### 3.4.2 Event Streaming

```python
# realtime_service/event_stream.py

import redis.asyncio as redis
from typing import AsyncGenerator

class EventStream:
    """Stream events from Salt and internal services"""
    
    def __init__(self, tenant_id: str):
        self.tenant_id = tenant_id
        self.redis = redis.Redis()
        self.stream_key = f"events:{tenant_id}"
        
    async def publish(self, event: dict):
        """Publish event to stream"""
        await self.redis.xadd(
            self.stream_key,
            {"data": json.dumps(event)},
            maxlen=10000  # Keep last 10k events
        )
        
    async def subscribe(
        self,
        last_id: str = "$"
    ) -> AsyncGenerator[dict, None]:
        """Subscribe to event stream"""
        while True:
            events = await self.redis.xread(
                {self.stream_key: last_id},
                block=5000  # 5 second timeout
            )
            
            for stream, messages in events:
                for msg_id, data in messages:
                    last_id = msg_id
                    yield json.loads(data[b"data"])
                    
    async def get_history(
        self,
        count: int = 100,
        start: str = "-",
        end: str = "+"
    ) -> list:
        """Get historical events"""
        events = await self.redis.xrange(
            self.stream_key,
            start,
            end,
            count=count
        )
        return [
            json.loads(data[b"data"])
            for _, data in events
        ]
```

---

## 4. Data Models

### 4.1 Database Schema

```sql
-- Core tenant and user management

CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    plan VARCHAR(50) NOT NULL DEFAULT 'free',
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255),
    name VARCHAR(255),
    role VARCHAR(50) NOT NULL DEFAULT 'member',
    settings JSONB DEFAULT '{}',
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(tenant_id, email)
);

-- Server inventory

CREATE TABLE servers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    minion_id VARCHAR(255) NOT NULL,
    hostname VARCHAR(255),
    ip_addresses JSONB DEFAULT '[]',
    os_family VARCHAR(50),
    os_release VARCHAR(100),
    kernel VARCHAR(100),
    cpu_count INTEGER,
    memory_total BIGINT,
    grains JSONB DEFAULT '{}',
    tags JSONB DEFAULT '[]',
    status VARCHAR(50) DEFAULT 'unknown',
    last_seen_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(tenant_id, minion_id)
);

CREATE INDEX idx_servers_tenant ON servers(tenant_id);
CREATE INDEX idx_servers_status ON servers(tenant_id, status);
CREATE INDEX idx_servers_tags ON servers USING GIN(tags);

-- Conversations and chat history

CREATE TABLE conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(255),
    context JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID REFERENCES conversations(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL, -- 'user', 'assistant', 'system'
    content TEXT NOT NULL,
    metadata JSONB DEFAULT '{}',
    tokens_used INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_messages_conversation ON messages(conversation_id, created_at);

-- Execution history and audit

CREATE TABLE executions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id),
    conversation_id UUID REFERENCES conversations(id),
    job_id VARCHAR(255),
    targets JSONB NOT NULL,
    module VARCHAR(255) NOT NULL,
    args JSONB DEFAULT '{}',
    results JSONB,
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_executions_tenant ON executions(tenant_id, created_at DESC);
CREATE INDEX idx_executions_status ON executions(tenant_id, status);

-- Audit log (immutable)

CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    tenant_id UUID NOT NULL,
    user_id UUID,
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(100),
    resource_id VARCHAR(255),
    details JSONB DEFAULT '{}',
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_audit_tenant_time ON audit_logs(tenant_id, created_at DESC);

-- Alerts and incidents

CREATE TABLE alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    server_id UUID REFERENCES servers(id) ON DELETE CASCADE,
    severity VARCHAR(20) NOT NULL,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    metric_name VARCHAR(100),
    metric_value NUMERIC,
    threshold NUMERIC,
    status VARCHAR(50) DEFAULT 'active',
    acknowledged_by UUID REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ,
    resolved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_alerts_tenant_status ON alerts(tenant_id, status, created_at DESC);

-- Row-level security policies

ALTER TABLE servers ENABLE ROW LEVEL SECURITY;
ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;
ALTER TABLE executions ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_servers ON servers
    USING (tenant_id = current_setting('app.tenant_id')::UUID);

CREATE POLICY tenant_isolation_conversations ON conversations
    USING (tenant_id = current_setting('app.tenant_id')::UUID);

CREATE POLICY tenant_isolation_executions ON executions
    USING (tenant_id = current_setting('app.tenant_id')::UUID);

CREATE POLICY tenant_isolation_audit ON audit_logs
    USING (tenant_id = current_setting('app.tenant_id')::UUID);
```

### 4.2 TimescaleDB Schema (Metrics)

```sql
-- Create hypertable for time-series metrics

CREATE TABLE metrics (
    time TIMESTAMPTZ NOT NULL,
    tenant_id UUID NOT NULL,
    server_id UUID NOT NULL,
    metric_name VARCHAR(100) NOT NULL,
    metric_value DOUBLE PRECISION NOT NULL,
    labels JSONB DEFAULT '{}'
);

SELECT create_hypertable('metrics', 'time');

-- Create continuous aggregates for common queries

CREATE MATERIALIZED VIEW metrics_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    tenant_id,
    server_id,
    metric_name,
    AVG(metric_value) as avg_value,
    MAX(metric_value) as max_value,
    MIN(metric_value) as min_value
FROM metrics
GROUP BY bucket, tenant_id, server_id, metric_name;

CREATE MATERIALIZED VIEW metrics_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    tenant_id,
    server_id,
    metric_name,
    AVG(metric_value) as avg_value,
    MAX(metric_value) as max_value,
    MIN(metric_value) as min_value
FROM metrics
GROUP BY bucket, tenant_id, server_id, metric_name;

-- Retention policy: raw data for 7 days, hourly for 90 days

SELECT add_retention_policy('metrics', INTERVAL '7 days');
SELECT add_retention_policy('metrics_hourly', INTERVAL '90 days');
SELECT add_retention_policy('metrics_daily', INTERVAL '365 days');

-- Compression policy for older data

ALTER TABLE metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, server_id, metric_name'
);

SELECT add_compression_policy('metrics', INTERVAL '1 day');
```

### 4.3 Elasticsearch Mappings (Logs)

```json
{
  "mappings": {
    "properties": {
      "tenant_id": { "type": "keyword" },
      "server_id": { "type": "keyword" },
      "hostname": { "type": "keyword" },
      "timestamp": { "type": "date" },
      "level": { "type": "keyword" },
      "source": { "type": "keyword" },
      "message": { 
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 256 }
        }
      },
      "structured": { "type": "object", "enabled": true },
      "tags": { "type": "keyword" }
    }
  },
  "settings": {
    "index": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "refresh_interval": "5s"
    },
    "index.lifecycle.name": "logs-policy",
    "index.lifecycle.rollover_alias": "logs"
  }
}
```

---

## 5. API Specifications

### 5.1 REST API Overview

**Base URL:** `https://api.serverflow.ai/v1`

**Authentication:** Bearer token (JWT)

```
Authorization: Bearer <access_token>
```

### 5.2 Core Endpoints

#### 5.2.1 Authentication

```yaml
POST /auth/login:
  description: Authenticate user and get tokens
  request:
    body:
      email: string (required)
      password: string (required)
      mfa_code: string (optional)
  response:
    200:
      access_token: string
      refresh_token: string
      expires_in: integer
      user:
        id: string
        email: string
        name: string
        role: string

POST /auth/refresh:
  description: Refresh access token
  request:
    body:
      refresh_token: string (required)
  response:
    200:
      access_token: string
      expires_in: integer

POST /auth/logout:
  description: Invalidate tokens
  response:
    204: No content
```

#### 5.2.2 Servers

```yaml
GET /servers:
  description: List all servers
  parameters:
    status: string (optional) - Filter by status
    tags: array (optional) - Filter by tags
    search: string (optional) - Search hostname/IP
    page: integer (default: 1)
    limit: integer (default: 50, max: 100)
  response:
    200:
      data: array[Server]
      pagination:
        total: integer
        page: integer
        limit: integer

GET /servers/{id}:
  description: Get server details
  response:
    200:
      id: string
      minion_id: string
      hostname: string
      ip_addresses: array
      os_family: string
      os_release: string
      cpu_count: integer
      memory_total: integer
      status: string
      last_seen_at: datetime
      grains: object
      tags: array

POST /servers/{id}/execute:
  description: Execute command on server
  request:
    body:
      module: string (required)
      args: object (optional)
      timeout: integer (optional, default: 60)
  response:
    202:
      job_id: string
      status: "pending"

DELETE /servers/{id}:
  description: Remove server from inventory
  response:
    204: No content
```

#### 5.2.3 Chat

```yaml
POST /chat/conversations:
  description: Create new conversation
  request:
    body:
      title: string (optional)
  response:
    201:
      id: string
      title: string
      created_at: datetime

GET /chat/conversations:
  description: List conversations
  parameters:
    page: integer
    limit: integer
  response:
    200:
      data: array[Conversation]
      pagination: object

POST /chat/conversations/{id}/messages:
  description: Send message (streaming response)
  request:
    body:
      content: string (required)
  response:
    200: (Server-Sent Events stream)
      event: message_start
      data: { message_id: string }
      
      event: content_delta
      data: { delta: string }
      
      event: action_available
      data: { action_id: string, label: string, ... }
      
      event: message_end
      data: { tokens_used: integer }

POST /chat/actions/{action_id}/execute:
  description: Execute suggested action
  request:
    body:
      confirm: boolean (required)
      parameters: object (optional)
  response:
    202:
      execution_id: string
      status: "pending"
```

#### 5.2.4 Metrics & Logs

```yaml
GET /metrics:
  description: Query metrics
  parameters:
    server_ids: array (optional)
    metric_names: array (required)
    start_time: datetime (required)
    end_time: datetime (required)
    aggregation: string (optional) - avg, max, min, sum
    interval: string (optional) - 1m, 5m, 1h, 1d
  response:
    200:
      data: array[MetricSeries]

GET /logs/search:
  description: Search logs
  parameters:
    query: string (required)
    server_ids: array (optional)
    level: string (optional)
    start_time: datetime (required)
    end_time: datetime (required)
    page: integer
    limit: integer
  response:
    200:
      data: array[LogEntry]
      pagination: object
      aggregations:
        by_level: object
        by_server: object
```

### 5.3 WebSocket API

```yaml
Connection: wss://api.serverflow.ai/ws

Authentication:
  # Send after connection
  {
    "type": "auth",
    "token": "<access_token>"
  }

Messages:

  # Chat message
  Client -> Server:
  {
    "type": "chat",
    "conversation_id": "uuid",
    "content": "What's using the most CPU?"
  }

  # Streaming response
  Server -> Client:
  {
    "type": "chat_response",
    "conversation_id": "uuid",
    "message_id": "uuid",
    "delta": "Looking at your servers...",
    "done": false
  }

  # Action available
  Server -> Client:
  {
    "type": "action_available",
    "action_id": "uuid",
    "label": "Restart nginx",
    "description": "This will restart nginx on web-01",
    "requires_confirmation": true
  }

  # Execute action
  Client -> Server:
  {
    "type": "execute_action",
    "action_id": "uuid",
    "confirmed": true
  }

  # Real-time alerts
  Server -> Client:
  {
    "type": "alert",
    "severity": "critical",
    "server_id": "uuid",
    "title": "High CPU on web-01",
    "description": "CPU usage at 95%"
  }

  # Server status update
  Server -> Client:
  {
    "type": "server_status",
    "server_id": "uuid",
    "status": "online",
    "metrics": {
      "cpu": 45.2,
      "memory": 67.8,
      "disk": 34.1
    }
  }
```

---

## 6. Security Architecture

### 6.1 Security Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SECURITY LAYERS                                      │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ Layer 1: NETWORK SECURITY                                                    │
│                                                                              │
│ • WAF (Web Application Firewall) - OWASP rules                              │
│ • DDoS protection (Cloudflare/AWS Shield)                                   │
│ • TLS 1.3 everywhere                                                         │
│ • Private subnets for internal services                                      │
│ • Network policies in Kubernetes                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Layer 2: AUTHENTICATION & AUTHORIZATION                                      │
│                                                                              │
│ • JWT tokens (short-lived, 15 min)                                          │
│ • Refresh token rotation                                                     │
│ • MFA support (TOTP, WebAuthn)                                              │
│ • SSO via SAML/OIDC (Business tier)                                         │
│ • Role-based access control (RBAC)                                          │
│ • API key authentication for automation                                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Layer 3: DATA SECURITY                                                       │
│                                                                              │
│ • Encryption at rest (AES-256)                                              │
│ • Encryption in transit (TLS 1.3)                                           │
│ • Per-tenant encryption keys                                                 │
│ • Secrets management (HashiCorp Vault)                                       │
│ • PII detection and masking in logs                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Layer 4: APPLICATION SECURITY                                                │
│                                                                              │
│ • Input validation on all endpoints                                          │
│ • SQL injection prevention (parameterized queries)                           │
│ • XSS prevention (CSP headers, output encoding)                             │
│ • CSRF protection                                                            │
│ • Rate limiting per user/tenant                                              │
│ • Command injection prevention (allowlist approach)                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Layer 5: AUDIT & COMPLIANCE                                                  │
│                                                                              │
│ • Immutable audit logs                                                       │
│ • All actions attributed to users                                            │
│ • Retention policies (configurable)                                          │
│ • SOC2 Type II compliance                                                    │
│ • GDPR data handling                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Agent-Master Communication Security

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AGENT-MASTER SECURITY                                     │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────┐                              ┌─────────────────┐
│  ServerFlow     │                              │  Salt Master    │
│  Agent          │                              │  (per tenant)   │
│                 │                              │                 │
│  ┌───────────┐  │      1. Key Exchange         │  ┌───────────┐  │
│  │ Minion    │  │ ─────────────────────────►   │  │ Master    │  │
│  │ Private   │  │                              │  │ Public    │  │
│  │ Key       │  │ ◄─────────────────────────   │  │ Key       │  │
│  └───────────┘  │      2. Key Acceptance       │  └───────────┘  │
│                 │         (auto or manual)     │                 │
│  ┌───────────┐  │                              │  ┌───────────┐  │
│  │ AES Key   │  │      3. AES Key Exchange     │  │ AES Key   │  │
│  │ (session) │  │ ◄────────────────────────►   │  │ (session) │  │
│  └───────────┘  │                              │  └───────────┘  │
│                 │                              │                 │
│                 │      4. Encrypted Comms      │                 │
│                 │ ◄════════════════════════►   │                 │
│                 │      (ZeroMQ + AES-256)      │                 │
└─────────────────┘                              └─────────────────┘

Security Features:
• PKI-based authentication (RSA 4096-bit)
• AES-256-CBC for message encryption
• Automatic key rotation (configurable)
• Minion ID verification
• IP allowlisting option
```

### 6.3 Command Execution Security

```python
# execution_service/security.py

class CommandSecurityValidator:
    """Validate and sanitize commands before execution"""
    
    # Modules that are always safe (read-only)
    SAFE_MODULES = {
        "status.*",
        "grains.*",
        "disk.usage",
        "network.interfaces",
        "ps.top",
        "service.status",
        "pkg.list_pkgs",
    }
    
    # Modules requiring explicit approval
    APPROVAL_REQUIRED = {
        "service.restart",
        "service.stop",
        "pkg.install",
        "pkg.remove",
        "file.managed",
        "cmd.run",
        "user.present",
        "user.absent",
    }
    
    # Modules that are never allowed
    BLOCKED_MODULES = {
        "cmd.run_all",  # Use cmd.run with restrictions
        "salt.execute",  # No nested execution
        "state.highstate",  # Use specific states only
    }
    
    # Dangerous patterns in arguments
    DANGEROUS_PATTERNS = [
        r"rm\s+-rf\s+/",
        r">\s*/dev/sd",
        r"mkfs\.",
        r"dd\s+if=",
        r":(){ :|:& };:",  # Fork bomb
    ]
    
    async def validate(
        self,
        module: str,
        args: dict,
        user: User,
        targets: List[str]
    ) -> ValidationResult:
        """Validate command before execution"""
        
        # Check if module is blocked
        if self._is_blocked(module):
            return ValidationResult(
                allowed=False,
                reason=f"Module {module} is not allowed"
            )
        
        # Check user permissions
        if not self._has_permission(user, module):
            return ValidationResult(
                allowed=False,
                reason="Insufficient permissions"
            )
        
        # Check for dangerous patterns
        if dangerous := self._check_dangerous_patterns(args):
            return ValidationResult(
                allowed=False,
                reason=f"Dangerous pattern detected: {dangerous}"
            )
        
        # Check if approval required
        requires_approval = self._requires_approval(module)
        
        return ValidationResult(
            allowed=True,
            requires_approval=requires_approval,
            audit_level="high" if requires_approval else "normal"
        )
```

### 6.4 Secrets Management

```yaml
# vault-config.yaml

vault:
  address: "https://vault.serverflow.internal:8200"
  auth_method: "kubernetes"
  
  secrets_engines:
    - path: "secret/tenants"
      type: "kv-v2"
      description: "Per-tenant secrets"
      
    - path: "pki/agents"
      type: "pki"
      description: "Agent certificates"
      
  policies:
    tenant_read:
      path: "secret/data/tenants/{{identity.entity.metadata.tenant_id}}/*"
      capabilities: ["read"]
      
    agent_cert:
      path: "pki/agents/issue/agent"
      capabilities: ["create", "update"]

# Secret types stored per tenant:
# - Salt master keys
# - Agent registration tokens
# - Database credentials (if self-hosted)
# - Integration API keys (Slack, PagerDuty, etc.)
# - SSH keys for legacy systems
```

---

## 7. Infrastructure Requirements

### 7.1 Kubernetes Architecture

```yaml
# kubernetes/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: serverflow
  labels:
    istio-injection: enabled

---
# kubernetes/deployment-api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: serverflow
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
        - name: api-gateway
          image: serverflow/api-gateway:latest
          ports:
            - containerPort: 8000
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2000m"
              memory: "2Gi"
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 5

---
# kubernetes/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway-hpa
  namespace: serverflow
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

### 7.2 Infrastructure Sizing

#### 7.2.1 Small Deployment (up to 1,000 servers)

```
┌─────────────────────────────────────────────────────────────────┐
│                    SMALL DEPLOYMENT                              │
│                    (1,000 managed servers)                       │
└─────────────────────────────────────────────────────────────────┘

Kubernetes Cluster:
├── Control Plane: Managed (EKS/GKE)
├── Worker Nodes: 3x m5.xlarge (4 vCPU, 16GB RAM)
└── Total: 12 vCPU, 48GB RAM

Services:
├── API Gateway: 2 replicas (1 vCPU, 2GB each)
├── AI Service: 2 replicas (2 vCPU, 4GB each)
├── Chat Service: 2 replicas (1 vCPU, 2GB each)
├── Execution Service: 2 replicas (1 vCPU, 2GB each)
├── Real-time Service: 2 replicas (1 vCPU, 2GB each)
└── Other Services: 1 replica each

Databases:
├── PostgreSQL: db.r5.large (2 vCPU, 16GB RAM, 500GB SSD)
├── TimescaleDB: db.r5.large (2 vCPU, 16GB RAM, 1TB SSD)
├── Redis: cache.r5.large (2 vCPU, 13GB RAM)
└── Elasticsearch: 3x m5.large (6 vCPU, 24GB RAM, 500GB SSD)

Salt Masters:
├── Per 100 servers: 1x t3.medium (2 vCPU, 4GB RAM)
└── Total for 1,000 servers: 10 Salt Masters

Estimated Monthly Cost: $3,000 - $4,000
```

#### 7.2.2 Medium Deployment (up to 10,000 servers)

```
┌─────────────────────────────────────────────────────────────────┐
│                    MEDIUM DEPLOYMENT                             │
│                    (10,000 managed servers)                      │
└─────────────────────────────────────────────────────────────────┘

Kubernetes Cluster:
├── Control Plane: Managed (EKS/GKE)
├── Worker Nodes: 10x m5.2xlarge (8 vCPU, 32GB RAM)
└── Total: 80 vCPU, 320GB RAM

Services:
├── API Gateway: 5 replicas (2 vCPU, 4GB each)
├── AI Service: 5 replicas (4 vCPU, 8GB each)
├── Chat Service: 5 replicas (2 vCPU, 4GB each)
├── Execution Service: 5 replicas (2 vCPU, 4GB each)
├── Real-time Service: 5 replicas (2 vCPU, 4GB each)
└── Other Services: 3 replicas each

Databases:
├── PostgreSQL: db.r5.2xlarge (8 vCPU, 64GB RAM, 2TB SSD)
├── TimescaleDB: db.r5.2xlarge (8 vCPU, 64GB RAM, 5TB SSD)
├── Redis Cluster: 3x cache.r5.xlarge (12 vCPU, 78GB RAM)
└── Elasticsearch: 6x m5.xlarge (24 vCPU, 96GB RAM, 2TB SSD)

Salt Masters:
├── Per 100 servers: 1x t3.large (2 vCPU, 8GB RAM)
└── Total for 10,000 servers: 100 Salt Masters

Estimated Monthly Cost: $15,000 - $25,000
```

### 7.3 Disaster Recovery

```
┌─────────────────────────────────────────────────────────────────┐
│                    DISASTER RECOVERY                             │
└─────────────────────────────────────────────────────────────────┘

RPO (Recovery Point Objective): 1 hour
RTO (Recovery Time Objective): 4 hours

Backup Strategy:
├── PostgreSQL: Continuous WAL archiving to S3, daily snapshots
├── TimescaleDB: Continuous replication, daily snapshots
├── Elasticsearch: Daily snapshots to S3
├── Redis: AOF persistence, hourly RDB snapshots
└── Salt Master configs: Git-based, replicated to S3

Multi-Region Setup:
┌─────────────────┐         ┌─────────────────┐
│  Primary Region │         │ Secondary Region│
│  (us-east-1)    │         │  (us-west-2)    │
│                 │         │                 │
│ ┌─────────────┐ │         │ ┌─────────────┐ │
│ │ Active      │ │ ──────► │ │ Standby     │ │
│ │ Cluster     │ │  Async  │ │ Cluster     │ │
│ └─────────────┘ │  Repli- │ └─────────────┘ │
│                 │  cation │                 │
│ ┌─────────────┐ │         │ ┌─────────────┐ │
│ │ Primary DB  │ │ ──────► │ │ Replica DB  │ │
│ └─────────────┘ │         │ └─────────────┘ │
└─────────────────┘         └─────────────────┘

Failover Process:
1. Detect primary region failure (automated health checks)
2. Promote secondary databases to primary
3. Update DNS to point to secondary region
4. Salt Masters reconnect automatically (agents retry)
5. Notify customers of failover
```

---

## 8. Integration Specifications

### 8.1 Third-Party Integrations

```yaml
integrations:
  
  # Notification channels
  slack:
    type: webhook
    events: [alerts, executions, reports]
    config:
      webhook_url: "https://hooks.slack.com/..."
      channel: "#ops-alerts"
      
  pagerduty:
    type: api
    events: [critical_alerts]
    config:
      routing_key: "..."
      
  email:
    type: smtp
    events: [alerts, reports, invitations]
    config:
      provider: "sendgrid"
      
  # Cloud providers
  aws:
    type: oauth
    capabilities: [ec2_inventory, cloudwatch_metrics]
    config:
      role_arn: "arn:aws:iam::..."
      
  azure:
    type: oauth
    capabilities: [vm_inventory, monitor_metrics]
    config:
      tenant_id: "..."
      
  gcp:
    type: service_account
    capabilities: [compute_inventory, monitoring]
    config:
      project_id: "..."
      
  # Monitoring tools
  datadog:
    type: api_key
    capabilities: [metrics_export, alert_sync]
    
  prometheus:
    type: remote_write
    capabilities: [metrics_export]
    
  # Ticketing systems
  jira:
    type: oauth
    capabilities: [create_tickets, sync_status]
    
  servicenow:
    type: oauth
    capabilities: [create_incidents, cmdb_sync]
```

### 8.2 Webhook Events

```json
{
  "webhook_events": {
    "server.connected": {
      "description": "New server connected",
      "payload": {
        "event": "server.connected",
        "timestamp": "2026-02-23T10:30:00Z",
        "server": {
          "id": "uuid",
          "hostname": "web-01",
          "ip_address": "10.0.1.5"
        }
      }
    },
    "server.disconnected": {
      "description": "Server went offline",
      "payload": {
        "event": "server.disconnected",
        "timestamp": "2026-02-23T10:30:00Z",
        "server": {
          "id": "uuid",
          "hostname": "web-01",
          "last_seen": "2026-02-23T10:25:00Z"
        }
      }
    },
    "alert.triggered": {
      "description": "Alert condition met",
      "payload": {
        "event": "alert.triggered",
        "timestamp": "2026-02-23T10:30:00Z",
        "alert": {
          "id": "uuid",
          "severity": "critical",
          "title": "High CPU on web-01",
          "server_id": "uuid"
        }
      }
    },
    "execution.completed": {
      "description": "Command execution finished",
      "payload": {
        "event": "execution.completed",
        "timestamp": "2026-02-23T10:30:00Z",
        "execution": {
          "id": "uuid",
          "module": "service.restart",
          "targets": ["web-01"],
          "status": "success",
          "user_id": "uuid"
        }
      }
    }
  }
}
```

---

## 9. Development Roadmap

### 9.1 Phase 1: MVP (Weeks 1-8)

```
Week 1-2: Foundation
├── [ ] Project setup (monorepo, CI/CD)
├── [ ] Database schema implementation
├── [ ] Authentication service
├── [ ] Agent installer (Linux)
└── [ ] Basic Salt Master deployment

Week 3-4: Core Features
├── [ ] Server inventory service
├── [ ] Basic telemetry collection
├── [ ] Web dashboard (Next.js)
├── [ ] Server list view
└── [ ] Real-time status updates

Week 5-6: AI Integration
├── [ ] Chat interface
├── [ ] NLP pipeline (intent classification)
├── [ ] LLM integration (OpenAI)
├── [ ] Basic query execution
└── [ ] Response generation

Week 7-8: MVP Polish
├── [ ] First remediation actions
├── [ ] Action approval flow
├── [ ] Basic alerting
├── [ ] Agent installer (Windows)
└── [ ] Documentation
```

### 9.2 Phase 2: Core Product (Weeks 9-16)

```
Week 9-10: Enhanced AI
├── [ ] Advanced diagnostics
├── [ ] Multi-step execution plans
├── [ ] Context-aware responses
└── [ ] Query caching

Week 11-12: Observability
├── [ ] Metrics collection & storage
├── [ ] Log aggregation
├── [ ] Custom dashboards
└── [ ] Alert rules engine

Week 13-14: Security & Compliance
├── [ ] Security scanning
├── [ ] Audit logging
├── [ ] RBAC implementation
└── [ ] MFA support

Week 15-16: Integrations
├── [ ] Slack integration
├── [ ] Email notifications
├── [ ] Webhook system
└── [ ] API documentation
```

### 9.3 Phase 3: Scale & Enterprise (Weeks 17-24)

```
Week 17-18: Enterprise Features
├── [ ] SSO (SAML/OIDC)
├── [ ] Teams & permissions
├── [ ] Custom actions/scripts
└── [ ] API access

Week 19-20: Advanced Automation
├── [ ] Scheduled tasks
├── [ ] Automated remediation
├── [ ] Runbook execution
└── [ ] Change management

Week 21-22: Scale & Performance
├── [ ] Multi-region support
├── [ ] Performance optimization
├── [ ] Load testing
└── [ ] Caching improvements

Week 23-24: Polish & Launch
├── [ ] Mobile app (React Native)
├── [ ] Onboarding improvements
├── [ ] Documentation
└── [ ] Marketing site
```

### 9.4 Technical Milestones

| Milestone | Target | Success Criteria |
|-----------|--------|------------------|
| Alpha | Week 8 | 10 internal servers managed |
| Private Beta | Week 12 | 5 customers, 100 servers |
| Public Beta | Week 18 | 50 customers, 1,000 servers |
| GA | Week 24 | 200 customers, 5,000 servers |

---

## 10. Appendices

### 10.1 Salt Module Reference

```python
# Common Salt modules used by ServerFlow

SALT_MODULES = {
    # System information
    "grains.items": "Get all system grains (OS, hardware, etc.)",
    "status.cpuinfo": "CPU information and usage",
    "status.meminfo": "Memory information and usage",
    "status.loadavg": "System load average",
    "status.uptime": "System uptime",
    
    # Disk operations
    "disk.usage": "Disk usage by mount point",
    "disk.inodeusage": "Inode usage by mount point",
    "file.stats": "File/directory statistics",
    "file.find": "Find files matching criteria",
    
    # Network
    "network.interfaces": "Network interface information",
    "network.netstat": "Network connections",
    "network.ping": "Ping remote host",
    "network.traceroute": "Traceroute to host",
    
    # Process management
    "ps.top": "Top processes by resource usage",
    "ps.pgrep": "Find processes by pattern",
    "ps.kill_pid": "Kill process by PID",
    
    # Service management
    "service.get_all": "List all services",
    "service.status": "Service status",
    "service.start": "Start service",
    "service.stop": "Stop service",
    "service.restart": "Restart service",
    
    # Package management
    "pkg.list_pkgs": "List installed packages",
    "pkg.list_upgrades": "List available upgrades",
    "pkg.install": "Install package",
    "pkg.remove": "Remove package",
    "pkg.upgrade": "Upgrade packages",
    
    # User management
    "user.list_users": "List system users",
    "user.info": "User information",
    "user.present": "Ensure user exists",
    "user.absent": "Ensure user doesn't exist",
    
    # File operations
    "file.managed": "Manage file content",
    "file.replace": "Replace text in file",
    "file.remove": "Remove file/directory",
    
    # Command execution
    "cmd.run": "Run shell command",
    "cmd.script": "Run script from URL/file",
}
```

### 10.2 Error Codes

```json
{
  "error_codes": {
    "AUTH001": "Invalid credentials",
    "AUTH002": "Token expired",
    "AUTH003": "Insufficient permissions",
    "AUTH004": "MFA required",
    
    "SRV001": "Server not found",
    "SRV002": "Server offline",
    "SRV003": "Server connection timeout",
    "SRV004": "Agent version mismatch",
    
    "EXEC001": "Execution failed",
    "EXEC002": "Execution timeout",
    "EXEC003": "Module not allowed",
    "EXEC004": "Approval required",
    "EXEC005": "Target validation failed",
    
    "AI001": "Query parsing failed",
    "AI002": "LLM service unavailable",
    "AI003": "Context limit exceeded",
    "AI004": "Unsafe operation detected",
    
    "RATE001": "Rate limit exceeded",
    "RATE002": "Quota exceeded",
    
    "INT001": "Integration not configured",
    "INT002": "Integration authentication failed",
    "INT003": "Integration rate limited"
  }
}
```

### 10.3 Glossary

| Term | Definition |
|------|------------|
| **Agent** | ServerFlow software installed on managed servers (Salt minion) |
| **Grain** | Salt term for system properties (OS, hardware, etc.) |
| **Master** | Salt Master server that manages agents |
| **Minion** | Salt term for managed server agent |
| **Module** | Salt execution module (e.g., `disk.usage`) |
| **State** | Salt configuration management definition |
| **Pillar** | Salt secure data store for sensitive configuration |
| **Execution** | Single command/action run on one or more servers |
| **Remediation** | Action taken to fix an identified issue |
| **Tenant** | Customer organization in multi-tenant system |

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-23 | Engineering | Initial specification |

---

*This document is confidential and intended for internal use only.*
