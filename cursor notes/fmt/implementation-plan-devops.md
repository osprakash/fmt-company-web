# FlowMind Technologies - DevOps Implementation Plan

> **Version**: 1.1  
> **Created**: November 2025  
> **Goal**: Production-ready monorepo with GitOps CI/CD pipeline using Helm

---

## 📋 Executive Summary

This document provides a step-by-step implementation plan for setting up:
- **Monorepo** for all 14 core services
- **Keycloak** as Identity Provider (IDP)
- **Kong/APISIX** as API Gateway (open source)
- **GitHub Actions** for CI/CD with GitOps approach
- **ArgoCD + Helm** for Kubernetes deployment
- **Target Environment**: 16GB RAM VM running Kubernetes

---

## 🏗️ Phase 1: Monorepo Setup (Week 1)

### 1.1 Repository Structure

```
flowmind-platform/                    # Main monorepo
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                   # Build, test, lint
│   │   ├── cd.yml                   # Build & push images, update Helm values
│   │   └── pr-checks.yml            # PR validation
│   ├── CODEOWNERS
│   └── pull_request_template.md
│
├── services/                         # All microservices
│   ├── account-service/
│   │   ├── src/
│   │   ├── tests/
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── iam-service/
│   ├── tenant-service/
│   ├── product-service/
│   ├── subscription-service/
│   ├── metering-service/
│   ├── notification-service/
│   ├── audit-service/
│   ├── analytics-service/
│   ├── feature-flags-service/
│   ├── file-service/
│   ├── support-service/
│   └── settings-service/
│
├── libs/                             # Shared libraries
│   ├── common/                       # Utilities, helpers
│   │   ├── src/
│   │   │   ├── database/
│   │   │   ├── messaging/
│   │   │   ├── logging/
│   │   │   ├── auth/
│   │   │   └── utils/
│   │   └── package.json
│   ├── contracts/                    # Shared types, DTOs, events
│   │   ├── src/
│   │   │   ├── events/
│   │   │   ├── dtos/
│   │   │   └── interfaces/
│   │   └── package.json
│   └── testing/                      # Test utilities
│       └── package.json
│
├── tools/                            # Build & dev tools
│   ├── scripts/
│   │   ├── build-affected.sh
│   │   ├── docker-build.sh
│   │   └── generate-api-client.sh
│   └── generators/                   # Service scaffolding
│
├── docker/                           # Docker configurations
│   ├── base/
│   │   └── Dockerfile.node          # Base image for Node services
│   └── compose/
│       ├── docker-compose.yml       # Local development
│       └── docker-compose.infra.yml # Local infra (DB, Redis, etc.)
│
├── nx.json                           # Nx monorepo config
├── package.json                      # Root package.json
├── pnpm-workspace.yaml               # PNPM workspace config
├── tsconfig.base.json                # Shared TS config
├── .env.example
├── .gitignore
├── .dockerignore
├── .eslintrc.js
├── .prettierrc
└── README.md

flowmind-gitops/                      # Separate GitOps repo (Helm-based)
├── charts/                           # Custom Helm charts
│   ├── flowmind-service/            # Reusable chart for all services
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── _helpers.tpl
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       ├── configmap.yaml
│   │       ├── secret.yaml
│   │       ├── hpa.yaml
│   │       └── ingress.yaml
│   │
│   ├── flowmind-infra/              # Infrastructure umbrella chart
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       └── namespaces.yaml
│   │
│   └── kong-config/                  # Kong declarative config chart
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           └── kong-configmap.yaml
│
├── environments/                     # Environment-specific values
│   ├── dev/
│   │   ├── values-infrastructure.yaml
│   │   ├── values-services.yaml
│   │   └── values-kong.yaml
│   ├── staging/
│   │   ├── values-infrastructure.yaml
│   │   ├── values-services.yaml
│   │   └── values-kong.yaml
│   └── production/
│       ├── values-infrastructure.yaml
│       ├── values-services.yaml
│       └── values-kong.yaml
│
├── apps/                             # ArgoCD Applications
│   ├── dev/
│   │   ├── root-app.yaml            # App of Apps
│   │   ├── infrastructure.yaml
│   │   ├── services.yaml
│   │   └── kong.yaml
│   ├── staging/
│   │   └── ...
│   └── production/
│       └── ...
│
└── README.md
```

### 1.2 Initialize Monorepo

```bash
# Step 1: Create and initialize repository
mkdir flowmind-platform && cd flowmind-platform
git init

# Step 2: Initialize with PNPM + Nx
pnpm init
pnpm add -D nx @nx/workspace @nx/node @nx/nest typescript

# Step 3: Initialize Nx
npx nx init

# Step 4: Create workspace configuration
```

**pnpm-workspace.yaml:**
```yaml
packages:
  - 'services/*'
  - 'libs/*'
  - 'tools/*'
```

**nx.json:**
```json
{
  "$schema": "./node_modules/nx/schemas/nx-schema.json",
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["production", "^production"],
      "cache": true
    },
    "test": {
      "inputs": ["default", "^production", "{workspaceRoot}/jest.preset.js"],
      "cache": true
    },
    "lint": {
      "inputs": ["default", "{workspaceRoot}/.eslintrc.js"],
      "cache": true
    }
  },
  "namedInputs": {
    "default": ["{projectRoot}/**/*", "sharedGlobals"],
    "production": [
      "default",
      "!{projectRoot}/**/?(*.)+(spec|test).[jt]s?(x)?(.snap)",
      "!{projectRoot}/tsconfig.spec.json",
      "!{projectRoot}/jest.config.[jt]s"
    ],
    "sharedGlobals": ["{workspaceRoot}/tsconfig.base.json"]
  },
  "plugins": ["@nx/nest"],
  "defaultProject": "account-service"
}
```

### 1.3 Create Service Template

**services/account-service/package.json:**
```json
{
  "name": "@flowmind/account-service",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "build": "nest build",
    "start": "nest start",
    "start:dev": "nest start --watch",
    "start:prod": "node dist/main",
    "test": "jest",
    "test:e2e": "jest --config ./test/jest-e2e.json",
    "lint": "eslint \"{src,test}/**/*.ts\" --fix"
  },
  "dependencies": {
    "@flowmind/common": "workspace:*",
    "@flowmind/contracts": "workspace:*",
    "@nestjs/common": "^10.0.0",
    "@nestjs/core": "^10.0.0",
    "@nestjs/config": "^3.0.0",
    "@nestjs/typeorm": "^10.0.0",
    "pg": "^8.11.0",
    "typeorm": "^0.3.17"
  }
}
```

**services/account-service/Dockerfile:**
```dockerfile
# Build stage
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY services/account-service/package.json ./services/account-service/
COPY libs/common/package.json ./libs/common/
COPY libs/contracts/package.json ./libs/contracts/

# Install dependencies
RUN corepack enable && pnpm install --frozen-lockfile

# Copy source
COPY tsconfig.base.json ./
COPY libs ./libs
COPY services/account-service ./services/account-service

# Build
WORKDIR /app/services/account-service
RUN pnpm build

# Production stage
FROM node:20-alpine AS runner

WORKDIR /app

# Copy built artifacts
COPY --from=builder /app/services/account-service/dist ./dist
COPY --from=builder /app/services/account-service/node_modules ./node_modules
COPY --from=builder /app/services/account-service/package.json ./

# Add non-root user
RUN addgroup -g 1001 -S nodejs && adduser -S nestjs -u 1001
USER nestjs

EXPOSE 3000

ENV NODE_ENV=production

CMD ["node", "dist/main"]
```

### 1.4 Shared Library Setup

**libs/common/src/index.ts:**
```typescript
// Database
export * from './database/base.entity';
export * from './database/tenant.middleware';

// Messaging
export * from './messaging/rabbitmq.module';
export * from './messaging/events';

// Auth
export * from './auth/jwt.guard';
export * from './auth/tenant.decorator';

// Logging
export * from './logging/logger.module';

// Utils
export * from './utils/pagination';
export * from './utils/response';
```

---

## 🔐 Phase 2: Keycloak IDP Setup (Week 2)

### 2.1 Keycloak Helm Chart Configuration

We'll use the official Bitnami Keycloak Helm chart.

**flowmind-gitops/environments/dev/values-infrastructure.yaml:**
```yaml
# Keycloak Configuration
keycloak:
  enabled: true
  replicaCount: 1
  
  auth:
    adminUser: admin
    # adminPassword is set via secret
    existingSecret: keycloak-admin-secret
    passwordSecretKey: password
  
  postgresql:
    enabled: false  # Use external PostgreSQL
  
  externalDatabase:
    host: postgresql.flowmind-infra.svc.cluster.local
    port: 5432
    database: keycloak
    existingSecret: keycloak-db-secret
    existingSecretUserKey: username
    existingSecretPasswordKey: password
  
  production: false  # Set true for production
  proxy: edge
  httpRelativePath: /
  
  extraEnvVars:
    - name: KC_HEALTH_ENABLED
      value: "true"
    - name: KC_METRICS_ENABLED
      value: "true"
    - name: KC_HOSTNAME_STRICT
      value: "false"
  
  resources:
    requests:
      memory: "512Mi"
      cpu: "250m"
    limits:
      memory: "1Gi"
      cpu: "1000m"
  
  service:
    type: ClusterIP
    ports:
      http: 80
  
  ingress:
    enabled: true
    ingressClassName: kong
    hostname: auth.flowmind.local
    path: /
    pathType: Prefix

# PostgreSQL Configuration (Bitnami chart)
postgresql:
  enabled: true
  auth:
    postgresPassword: ""  # Set via secret
    existingSecret: postgresql-secret
    secretKeys:
      adminPasswordKey: postgres-password
  
  primary:
    persistence:
      enabled: true
      size: 20Gi
    
    resources:
      requests:
        memory: "512Mi"
        cpu: "250m"
      limits:
        memory: "1Gi"
        cpu: "1000m"
    
    initdb:
      scripts:
        init-databases.sql: |
          CREATE DATABASE keycloak;
          CREATE DATABASE kong;
          CREATE DATABASE flowmind_accounts;
          CREATE DATABASE flowmind_iam;
          CREATE DATABASE flowmind_tenants;
          CREATE DATABASE flowmind_products;
          CREATE DATABASE flowmind_subscriptions;
          CREATE DATABASE flowmind_notifications;
          CREATE DATABASE flowmind_audit;

# Redis Configuration (Bitnami chart)
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: true
    existingSecret: redis-secret
    existingSecretPasswordKey: password
  
  master:
    persistence:
      enabled: true
      size: 5Gi
    
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "500m"

# RabbitMQ Configuration (Bitnami chart)
rabbitmq:
  enabled: true
  auth:
    username: flowmind
    existingPasswordSecret: rabbitmq-secret
    existingSecretPasswordKey: password
  
  persistence:
    enabled: true
    size: 5Gi
  
  resources:
    requests:
      memory: "256Mi"
      cpu: "200m"
    limits:
      memory: "512Mi"
      cpu: "500m"
```

### 2.2 Keycloak Realm Configuration

**flowmind-gitops/charts/flowmind-infra/files/realm-flowmind.json:**
```json
{
  "realm": "flowmind",
  "enabled": true,
  "sslRequired": "external",
  "registrationAllowed": false,
  "loginWithEmailAllowed": true,
  "duplicateEmailsAllowed": false,
  "resetPasswordAllowed": true,
  "editUsernameAllowed": false,
  "bruteForceProtected": true,
  "permanentLockout": false,
  "maxFailureWaitSeconds": 900,
  "minimumQuickLoginWaitSeconds": 60,
  "waitIncrementSeconds": 60,
  "quickLoginCheckMilliSeconds": 1000,
  "maxDeltaTimeSeconds": 43200,
  "failureFactor": 5,
  "roles": {
    "realm": [
      { "name": "platform_admin", "description": "Platform Administrator" },
      { "name": "org_admin", "description": "Organization Administrator" },
      { "name": "org_member", "description": "Organization Member" }
    ]
  },
  "clients": [
    {
      "clientId": "flowmind-api",
      "enabled": true,
      "clientAuthenticatorType": "client-secret",
      "redirectUris": ["*"],
      "webOrigins": ["*"],
      "standardFlowEnabled": true,
      "directAccessGrantsEnabled": true,
      "serviceAccountsEnabled": true,
      "protocol": "openid-connect",
      "attributes": {
        "access.token.lifespan": "900",
        "client.session.idle.timeout": "1800"
      },
      "protocolMappers": [
        {
          "name": "tenant-mapper",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "config": {
            "claim.name": "tenant_id",
            "user.attribute": "tenant_id",
            "access.token.claim": "true",
            "id.token.claim": "true"
          }
        },
        {
          "name": "org-mapper",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "config": {
            "claim.name": "organization_id",
            "user.attribute": "organization_id",
            "access.token.claim": "true",
            "id.token.claim": "true"
          }
        }
      ]
    },
    {
      "clientId": "flowmind-web",
      "enabled": true,
      "publicClient": true,
      "redirectUris": [
        "http://localhost:3000/*",
        "https://app.flowmind.local/*"
      ],
      "webOrigins": ["+"],
      "standardFlowEnabled": true,
      "directAccessGrantsEnabled": false,
      "protocol": "openid-connect"
    }
  ],
  "browserSecurityHeaders": {
    "contentSecurityPolicy": "frame-src 'self'; frame-ancestors 'self'; object-src 'none';"
  }
}
```

---

## 🌐 Phase 3: API Gateway Setup - Kong (Week 2-3)

### 3.1 Kong Helm Chart Configuration

**flowmind-gitops/environments/dev/values-kong.yaml:**
```yaml
# Kong Gateway (using official Kong Helm chart)
kong:
  enabled: true
  
  env:
    database: postgres
    pg_host: postgresql.flowmind-infra.svc.cluster.local
    pg_user:
      valueFrom:
        secretKeyRef:
          name: kong-db-secret
          key: username
    pg_password:
      valueFrom:
        secretKeyRef:
          name: kong-db-secret
          key: password
    pg_database: kong
    
    # Logging
    proxy_access_log: /dev/stdout
    admin_access_log: /dev/stdout
    proxy_error_log: /dev/stderr
    admin_error_log: /dev/stderr
    
    # Plugins
    plugins: bundled,jwt,cors,rate-limiting,correlation-id
  
  image:
    repository: kong
    tag: "3.5"
  
  ingressController:
    enabled: true
    installCRDs: false
  
  proxy:
    enabled: true
    type: NodePort
    http:
      enabled: true
      nodePort: 30080
    tls:
      enabled: true
      nodePort: 30443
  
  admin:
    enabled: true
    type: ClusterIP
    http:
      enabled: true
  
  resources:
    requests:
      memory: "256Mi"
      cpu: "200m"
    limits:
      memory: "1Gi"
      cpu: "1000m"

  # Migration job
  migrations:
    preUpgrade: true
    postUpgrade: true
```

### 3.2 Kong Configuration Chart

**flowmind-gitops/charts/kong-config/Chart.yaml:**
```yaml
apiVersion: v2
name: kong-config
description: Kong declarative configuration for FlowMind services
type: application
version: 0.1.0
appVersion: "1.0.0"
```

**flowmind-gitops/charts/kong-config/values.yaml:**
```yaml
# Global Kong configuration
global:
  rateLimiting:
    defaultMinute: 100
  cors:
    enabled: true
    origins:
      - "*"
    methods:
      - GET
      - POST
      - PUT
      - PATCH
      - DELETE
      - OPTIONS
    headers:
      - Authorization
      - Content-Type
      - X-Tenant-ID
    exposedHeaders:
      - X-Request-ID
    credentials: true
    maxAge: 3600

# Service routing configuration
services:
  account-service:
    enabled: true
    host: account-service.flowmind-services.svc.cluster.local
    port: 3000
    routes:
      - paths:
          - /api/v1/accounts
          - /api/v1/organizations
          - /api/v1/divisions
    rateLimitMinute: 100
    
  iam-service:
    enabled: true
    host: iam-service.flowmind-services.svc.cluster.local
    port: 3000
    routes:
      - paths:
          - /api/v1/users
          - /api/v1/roles
          - /api/v1/permissions
    rateLimitMinute: 200
    
  tenant-service:
    enabled: true
    host: tenant-service.flowmind-services.svc.cluster.local
    port: 3000
    routes:
      - paths:
          - /api/v1/tenants
    rateLimitMinute: 100
    
  product-service:
    enabled: true
    host: product-service.flowmind-services.svc.cluster.local
    port: 3000
    routes:
      - paths:
          - /api/v1/products
          - /api/v1/features
          - /api/v1/tiers
    rateLimitMinute: 150
    
  subscription-service:
    enabled: true
    host: subscription-service.flowmind-services.svc.cluster.local
    port: 3000
    routes:
      - paths:
          - /api/v1/subscriptions
          - /api/v1/invoices
    rateLimitMinute: 100
    
  metering-service:
    enabled: true
    host: metering-service.flowmind-services.svc.cluster.local
    port: 3000
    routes:
      - paths:
          - /api/v1/metering
          - /api/v1/usage
    rateLimitMinute: 500
    
  notification-service:
    enabled: true
    host: notification-service.flowmind-services.svc.cluster.local
    port: 3000
    routes:
      - paths:
          - /api/v1/notifications
    rateLimitMinute: 200
    
  audit-service:
    enabled: true
    host: audit-service.flowmind-services.svc.cluster.local
    port: 3000
    routes:
      - paths:
          - /api/v1/audit
          - /api/v1/logs
    rateLimitMinute: 100
    
  support-service:
    enabled: true
    host: support-service.flowmind-services.svc.cluster.local
    port: 3000
    routes:
      - paths:
          - /api/v1/tickets
          - /api/v1/knowledge-base
    rateLimitMinute: 100

# JWT configuration
jwt:
  enabled: true
  keyClaimName: kid
  algorithm: RS256
```

**flowmind-gitops/charts/kong-config/templates/kong-ingress.yaml:**
```yaml
{{- range $name, $service := .Values.services }}
{{- if $service.enabled }}
---
apiVersion: configuration.konghq.com/v1
kind: KongIngress
metadata:
  name: {{ $name }}-kong-ingress
  labels:
    {{- include "kong-config.labels" $ | nindent 4 }}
proxy:
  protocol: http
  path: /
  connect_timeout: 60000
  read_timeout: 60000
  write_timeout: 60000
route:
  strip_path: false
  preserve_host: true
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ $name }}-ingress
  annotations:
    konghq.com/strip-path: "false"
    konghq.com/plugins: {{ $name }}-rate-limit,global-cors,global-correlation-id
    {{- if $.Values.jwt.enabled }}
    konghq.com/plugins: {{ $name }}-rate-limit,global-cors,global-correlation-id,jwt-auth
    {{- end }}
  labels:
    {{- include "kong-config.labels" $ | nindent 4 }}
spec:
  ingressClassName: kong
  rules:
    {{- range $service.routes }}
    - http:
        paths:
          {{- range .paths }}
          - path: {{ . }}
            pathType: Prefix
            backend:
              service:
                name: {{ $name }}
                port:
                  number: {{ $service.port }}
          {{- end }}
    {{- end }}
{{- end }}
{{- end }}
```

**flowmind-gitops/charts/kong-config/templates/kong-plugins.yaml:**
```yaml
# Global CORS Plugin
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: global-cors
  labels:
    {{- include "kong-config.labels" . | nindent 4 }}
config:
  origins:
    {{- toYaml .Values.global.cors.origins | nindent 4 }}
  methods:
    {{- toYaml .Values.global.cors.methods | nindent 4 }}
  headers:
    {{- toYaml .Values.global.cors.headers | nindent 4 }}
  exposed_headers:
    {{- toYaml .Values.global.cors.exposedHeaders | nindent 4 }}
  credentials: {{ .Values.global.cors.credentials }}
  max_age: {{ .Values.global.cors.maxAge }}
plugin: cors
---
# Global Correlation ID Plugin
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: global-correlation-id
  labels:
    {{- include "kong-config.labels" . | nindent 4 }}
config:
  header_name: X-Request-ID
  generator: uuid
  echo_downstream: true
plugin: correlation-id
---
{{- if .Values.jwt.enabled }}
# JWT Authentication Plugin
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: jwt-auth
  labels:
    {{- include "kong-config.labels" . | nindent 4 }}
config:
  key_claim_name: {{ .Values.jwt.keyClaimName }}
plugin: jwt
{{- end }}
---
# Per-service rate limiting
{{- range $name, $service := .Values.services }}
{{- if $service.enabled }}
---
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: {{ $name }}-rate-limit
  labels:
    {{- include "kong-config.labels" $ | nindent 4 }}
config:
  minute: {{ $service.rateLimitMinute | default $.Values.global.rateLimiting.defaultMinute }}
  policy: local
plugin: rate-limiting
{{- end }}
{{- end }}
```

---

## 📦 Phase 4: Helm Charts for Services (Week 3)

### 4.1 Reusable Service Chart

**flowmind-gitops/charts/flowmind-service/Chart.yaml:**
```yaml
apiVersion: v2
name: flowmind-service
description: A reusable Helm chart for FlowMind microservices
type: application
version: 0.1.0
appVersion: "1.0.0"
```

**flowmind-gitops/charts/flowmind-service/values.yaml:**
```yaml
# Default values for flowmind-service

# Number of replicas
replicaCount: 1

# Image configuration
image:
  repository: ghcr.io/flowmind/service-name
  pullPolicy: IfNotPresent
  tag: "latest"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

# Service configuration
service:
  type: ClusterIP
  port: 3000

# Container port
containerPort: 3000

# Environment variables
env:
  NODE_ENV: production

# Environment variables from ConfigMaps
envFrom:
  configMaps: []
  secrets: []

# Additional environment variables from secrets/configmaps
envVars: []
# Example:
# - name: DATABASE_URL
#   valueFrom:
#     secretKeyRef:
#       name: my-secret
#       key: database-url

# Resource limits
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"

# Health checks
livenessProbe:
  enabled: true
  httpGet:
    path: /health/live
    port: http
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  enabled: true
  httpGet:
    path: /health/ready
    port: http
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3

# Horizontal Pod Autoscaler
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80

# Node selector
nodeSelector: {}

# Tolerations
tolerations: []

# Affinity rules
affinity: {}

# Pod annotations
podAnnotations: {}

# Pod labels
podLabels: {}

# Security context
securityContext:
  runAsNonRoot: true
  runAsUser: 1001

# Service account
serviceAccount:
  create: true
  annotations: {}
  name: ""

# Ingress (usually handled by Kong, but available if needed)
ingress:
  enabled: false
```

**flowmind-gitops/charts/flowmind-service/templates/_helpers.tpl:**
```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "flowmind-service.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "flowmind-service.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "flowmind-service.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "flowmind-service.labels" -}}
helm.sh/chart: {{ include "flowmind-service.chart" . }}
{{ include "flowmind-service.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
platform: flowmind
{{- end }}

{{/*
Selector labels
*/}}
{{- define "flowmind-service.selectorLabels" -}}
app.kubernetes.io/name: {{ include "flowmind-service.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "flowmind-service.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "flowmind-service.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

**flowmind-gitops/charts/flowmind-service/templates/deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "flowmind-service.fullname" . }}
  labels:
    {{- include "flowmind-service.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "flowmind-service.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "flowmind-service.labels" . | nindent 8 }}
        {{- with .Values.podLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "flowmind-service.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.securityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.containerPort }}
              protocol: TCP
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
            {{- with .Values.envVars }}
            {{- toYaml . | nindent 12 }}
            {{- end }}
          {{- if or .Values.envFrom.configMaps .Values.envFrom.secrets }}
          envFrom:
            {{- range .Values.envFrom.configMaps }}
            - configMapRef:
                name: {{ . }}
            {{- end }}
            {{- range .Values.envFrom.secrets }}
            - secretRef:
                name: {{ . }}
            {{- end }}
          {{- end }}
          {{- if .Values.livenessProbe.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.livenessProbe.httpGet.path }}
              port: {{ .Values.livenessProbe.httpGet.port }}
            initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.livenessProbe.periodSeconds }}
            timeoutSeconds: {{ .Values.livenessProbe.timeoutSeconds }}
            failureThreshold: {{ .Values.livenessProbe.failureThreshold }}
          {{- end }}
          {{- if .Values.readinessProbe.enabled }}
          readinessProbe:
            httpGet:
              path: {{ .Values.readinessProbe.httpGet.path }}
              port: {{ .Values.readinessProbe.httpGet.port }}
            initialDelaySeconds: {{ .Values.readinessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.readinessProbe.periodSeconds }}
            timeoutSeconds: {{ .Values.readinessProbe.timeoutSeconds }}
            failureThreshold: {{ .Values.readinessProbe.failureThreshold }}
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

**flowmind-gitops/charts/flowmind-service/templates/service.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "flowmind-service.fullname" . }}
  labels:
    {{- include "flowmind-service.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "flowmind-service.selectorLabels" . | nindent 4 }}
```

**flowmind-gitops/charts/flowmind-service/templates/hpa.yaml:**
```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "flowmind-service.fullname" . }}
  labels:
    {{- include "flowmind-service.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "flowmind-service.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
    {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
    {{- end }}
{{- end }}
```

**flowmind-gitops/charts/flowmind-service/templates/serviceaccount.yaml:**
```yaml
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "flowmind-service.serviceAccountName" . }}
  labels:
    {{- include "flowmind-service.labels" . | nindent 4 }}
  {{- with .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
{{- end }}
```

### 4.2 Services Values File

**flowmind-gitops/environments/dev/values-services.yaml:**
```yaml
# Global configuration for all services
global:
  imageRegistry: ghcr.io/flowmind
  imagePullSecrets:
    - name: ghcr-secret
  environment: dev
  
  # Shared environment variables
  config:
    keycloakUrl: http://keycloak.flowmind-infra.svc.cluster.local
    keycloakRealm: flowmind
    redisUrl: redis://redis-master.flowmind-infra.svc.cluster.local:6379
    rabbitmqHost: rabbitmq.flowmind-infra.svc.cluster.local

# Individual service configurations
services:
  account-service:
    enabled: true
    replicaCount: 1
    image:
      repository: ghcr.io/flowmind/account-service
      tag: "latest"  # Updated by CI/CD
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "500m"
    envVars:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: account-service-secrets
            key: database-url
      - name: REDIS_URL
        value: "redis://redis-master.flowmind-infra.svc.cluster.local:6379"
      - name: RABBITMQ_URL
        valueFrom:
          secretKeyRef:
            name: rabbitmq-secret
            key: url
      - name: KEYCLOAK_URL
        value: "http://keycloak.flowmind-infra.svc.cluster.local"
      - name: KEYCLOAK_REALM
        value: "flowmind"
  
  iam-service:
    enabled: true
    replicaCount: 1
    image:
      repository: ghcr.io/flowmind/iam-service
      tag: "latest"
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "500m"
    envVars:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: iam-service-secrets
            key: database-url
      - name: REDIS_URL
        value: "redis://redis-master.flowmind-infra.svc.cluster.local:6379"
  
  tenant-service:
    enabled: true
    replicaCount: 1
    image:
      repository: ghcr.io/flowmind/tenant-service
      tag: "latest"
    envVars:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: tenant-service-secrets
            key: database-url
  
  product-service:
    enabled: true
    replicaCount: 1
    image:
      repository: ghcr.io/flowmind/product-service
      tag: "latest"
    envVars:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: product-service-secrets
            key: database-url
  
  subscription-service:
    enabled: true
    replicaCount: 1
    image:
      repository: ghcr.io/flowmind/subscription-service
      tag: "latest"
    envVars:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: subscription-service-secrets
            key: database-url
      - name: STRIPE_SECRET_KEY
        valueFrom:
          secretKeyRef:
            name: stripe-secrets
            key: secret-key
  
  metering-service:
    enabled: true
    replicaCount: 1
    image:
      repository: ghcr.io/flowmind/metering-service
      tag: "latest"
    resources:
      requests:
        memory: "256Mi"
        cpu: "150m"
      limits:
        memory: "512Mi"
        cpu: "750m"
    envVars:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: metering-service-secrets
            key: database-url
  
  notification-service:
    enabled: true
    replicaCount: 1
    image:
      repository: ghcr.io/flowmind/notification-service
      tag: "latest"
    envVars:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: notification-service-secrets
            key: database-url
      - name: SENDGRID_API_KEY
        valueFrom:
          secretKeyRef:
            name: sendgrid-secrets
            key: api-key
  
  audit-service:
    enabled: true
    replicaCount: 1
    image:
      repository: ghcr.io/flowmind/audit-service
      tag: "latest"
    envVars:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: audit-service-secrets
            key: database-url
  
  analytics-service:
    enabled: true
    replicaCount: 1
    image:
      repository: ghcr.io/flowmind/analytics-service
      tag: "latest"
    envVars:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: analytics-service-secrets
            key: database-url
  
  feature-flags-service:
    enabled: true
    replicaCount: 1
    image:
      repository: ghcr.io/flowmind/feature-flags-service
      tag: "latest"
    envVars:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: feature-flags-service-secrets
            key: database-url
  
  file-service:
    enabled: true
    replicaCount: 1
    image:
      repository: ghcr.io/flowmind/file-service
      tag: "latest"
    envVars:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: file-service-secrets
            key: database-url
      - name: S3_BUCKET
        value: "flowmind-files-dev"
      - name: AWS_REGION
        value: "us-east-1"
  
  support-service:
    enabled: true
    replicaCount: 1
    image:
      repository: ghcr.io/flowmind/support-service
      tag: "latest"
    envVars:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: support-service-secrets
            key: database-url
  
  settings-service:
    enabled: true
    replicaCount: 1
    image:
      repository: ghcr.io/flowmind/settings-service
      tag: "latest"
    envVars:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: settings-service-secrets
            key: database-url
```

---

## 🔄 Phase 5: GitHub Actions CI/CD (Week 3)

### 5.1 CI Workflow

**.github/workflows/ci.yml:**
```yaml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  REGISTRY: ghcr.io
  IMAGE_PREFIX: ${{ github.repository_owner }}/flowmind

jobs:
  # Detect affected services
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.detect.outputs.services }}
      libs-changed: ${{ steps.detect.outputs.libs-changed }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Detect affected services
        id: detect
        run: |
          if [ "${{ github.event_name }}" == "pull_request" ]; then
            BASE_SHA=${{ github.event.pull_request.base.sha }}
          else
            BASE_SHA=${{ github.event.before }}
          fi
          
          # Get changed files
          CHANGED_FILES=$(git diff --name-only $BASE_SHA HEAD)
          
          # Check if libs changed
          LIBS_CHANGED=$(echo "$CHANGED_FILES" | grep -q "^libs/" && echo "true" || echo "false")
          echo "libs-changed=$LIBS_CHANGED" >> $GITHUB_OUTPUT
          
          # Get affected services
          SERVICES=()
          for dir in services/*/; do
            SERVICE_NAME=$(basename $dir)
            if echo "$CHANGED_FILES" | grep -q "^services/$SERVICE_NAME/" || [ "$LIBS_CHANGED" == "true" ]; then
              SERVICES+=("$SERVICE_NAME")
            fi
          done
          
          # Output as JSON array
          SERVICES_JSON=$(printf '%s\n' "${SERVICES[@]}" | jq -R -s -c 'split("\n") | map(select(length > 0))')
          echo "services=$SERVICES_JSON" >> $GITHUB_OUTPUT

  # Lint and Test
  lint-test:
    needs: detect-changes
    if: needs.detect-changes.outputs.services != '[]'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: ${{ fromJson(needs.detect-changes.outputs.services) }}
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Setup pnpm
        uses: pnpm/action-setup@v2
        with:
          version: 8

      - name: Get pnpm store directory
        shell: bash
        run: |
          echo "STORE_PATH=$(pnpm store path --silent)" >> $GITHUB_ENV

      - name: Setup pnpm cache
        uses: actions/cache@v3
        with:
          path: ${{ env.STORE_PATH }}
          key: ${{ runner.os }}-pnpm-store-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: |
            ${{ runner.os }}-pnpm-store-

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Lint
        run: pnpm --filter @flowmind/${{ matrix.service }} lint

      - name: Test
        run: pnpm --filter @flowmind/${{ matrix.service }} test -- --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./services/${{ matrix.service }}/coverage/lcov.info
          flags: ${{ matrix.service }}

  # Build Docker images
  build:
    needs: [detect-changes, lint-test]
    if: github.event_name == 'push' && needs.detect-changes.outputs.services != '[]'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: ${{ fromJson(needs.detect-changes.outputs.services) }}
    permissions:
      contents: read
      packages: write
    outputs:
      image-tags: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/${{ matrix.service }}
          tags: |
            type=sha,prefix=
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./services/${{ matrix.service }}/Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            SERVICE_NAME=${{ matrix.service }}

  # Update Helm values in GitOps repo
  update-helm-values:
    needs: [detect-changes, build]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout GitOps repo
        uses: actions/checkout@v4
        with:
          repository: ${{ github.repository_owner }}/flowmind-gitops
          token: ${{ secrets.GITOPS_PAT }}
          path: gitops

      - name: Setup yq
        uses: mikefarah/yq@master

      - name: Update image tags in Helm values
        run: |
          cd gitops
          IMAGE_TAG="${{ github.sha }}"
          SERVICES='${{ needs.detect-changes.outputs.services }}'
          
          # Parse the JSON array and update each service
          echo $SERVICES | jq -r '.[]' | while read SERVICE; do
            echo "Updating $SERVICE to tag $IMAGE_TAG"
            
            # Update the image tag in values-services.yaml
            yq eval -i ".services.${SERVICE}.image.tag = \"${IMAGE_TAG}\"" \
              environments/dev/values-services.yaml
          done

      - name: Commit and push
        run: |
          cd gitops
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add .
          git diff --staged --quiet || git commit -m "chore: update image tags to ${{ github.sha }}"
          git push
```

### 5.2 PR Checks Workflow

**.github/workflows/pr-checks.yml:**
```yaml
name: PR Checks

on:
  pull_request:
    branches: [main, develop]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          ignore-unfixed: true
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  helm-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: ${{ github.repository_owner }}/flowmind-gitops
          token: ${{ secrets.GITOPS_PAT }}

      - name: Setup Helm
        uses: azure/setup-helm@v3
        with:
          version: v3.13.0

      - name: Lint Helm charts
        run: |
          for chart in charts/*/; do
            echo "Linting $chart"
            helm lint "$chart"
          done

  dependency-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Dependency Review
        uses: actions/dependency-review-action@v3
        with:
          fail-on-severity: high

  commitlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install commitlint
        run: |
          npm install -g @commitlint/cli @commitlint/config-conventional

      - name: Validate commits
        run: |
          npx commitlint --from ${{ github.event.pull_request.base.sha }} --to ${{ github.event.pull_request.head.sha }} --verbose
```

---

## 🚀 Phase 6: ArgoCD & Kubernetes Setup (Week 4)

### 6.1 Kubernetes Cluster Setup (16GB VM)

```bash
# Option 1: K3s (Recommended for 16GB VM)
# Install K3s - lightweight Kubernetes
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644

# Verify installation
kubectl get nodes

# Option 2: MicroK8s
# sudo snap install microk8s --classic
# microk8s enable dns storage ingress helm3
```

### 6.2 Resource Allocation (16GB VM)

```yaml
# Recommended resource allocation for 16GB RAM VM
---
# System reserved: ~2GB
# Kubernetes components: ~2GB
# Available for workloads: ~12GB

# Infrastructure (Required):
# - PostgreSQL: 1GB
# - Redis: 512MB
# - RabbitMQ: 512MB
# - Keycloak: 1GB
# - Kong: 1GB
# - ArgoCD: 512MB
# Subtotal: ~4.5GB

# Services (Per service ~256MB-512MB):
# 13 services × 300MB avg = ~4GB

# Buffer/Overhead: ~3.5GB
```

### 6.3 ArgoCD Installation

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward for access
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Install ArgoCD CLI
# Windows (PowerShell):
# Invoke-WebRequest -Uri https://github.com/argoproj/argo-cd/releases/latest/download/argocd-windows-amd64.exe -OutFile argocd.exe
```

### 6.4 ArgoCD Application Definitions

**flowmind-gitops/apps/dev/root-app.yaml:**
```yaml
# Root Application (App of Apps pattern)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: flowmind-root
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/flowmind/flowmind-gitops.git
    targetRevision: main
    path: apps/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**flowmind-gitops/apps/dev/infrastructure.yaml:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: flowmind-infrastructure
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/flowmind/flowmind-gitops.git
    targetRevision: main
    path: charts/flowmind-infra
    helm:
      valueFiles:
        - ../../environments/dev/values-infrastructure.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: flowmind-infra
  syncPolicy:
    automated:
      prune: false  # Don't auto-delete infra
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**flowmind-gitops/apps/dev/services.yaml:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: flowmind-services
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    # Account Service
    - repoURL: https://github.com/flowmind/flowmind-gitops.git
      targetRevision: main
      path: charts/flowmind-service
      helm:
        releaseName: account-service
        valueFiles:
          - $values/environments/dev/values-services.yaml
        parameters:
          - name: nameOverride
            value: account-service
    # IAM Service
    - repoURL: https://github.com/flowmind/flowmind-gitops.git
      targetRevision: main
      path: charts/flowmind-service
      helm:
        releaseName: iam-service
        valueFiles:
          - $values/environments/dev/values-services.yaml
        parameters:
          - name: nameOverride
            value: iam-service
    # Values repository
    - repoURL: https://github.com/flowmind/flowmind-gitops.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: flowmind-services
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**Alternative: Single Application with Multiple Services (Umbrella Chart approach)**

**flowmind-gitops/charts/flowmind-services/Chart.yaml:**
```yaml
apiVersion: v2
name: flowmind-services
description: Umbrella chart for all FlowMind microservices
type: application
version: 0.1.0
appVersion: "1.0.0"

dependencies:
  - name: flowmind-service
    version: "0.1.0"
    repository: "file://../flowmind-service"
    alias: account-service
    condition: services.account-service.enabled
  
  - name: flowmind-service
    version: "0.1.0"
    repository: "file://../flowmind-service"
    alias: iam-service
    condition: services.iam-service.enabled
  
  - name: flowmind-service
    version: "0.1.0"
    repository: "file://../flowmind-service"
    alias: tenant-service
    condition: services.tenant-service.enabled
  
  - name: flowmind-service
    version: "0.1.0"
    repository: "file://../flowmind-service"
    alias: product-service
    condition: services.product-service.enabled
  
  - name: flowmind-service
    version: "0.1.0"
    repository: "file://../flowmind-service"
    alias: subscription-service
    condition: services.subscription-service.enabled
  
  - name: flowmind-service
    version: "0.1.0"
    repository: "file://../flowmind-service"
    alias: metering-service
    condition: services.metering-service.enabled
  
  - name: flowmind-service
    version: "0.1.0"
    repository: "file://../flowmind-service"
    alias: notification-service
    condition: services.notification-service.enabled
  
  - name: flowmind-service
    version: "0.1.0"
    repository: "file://../flowmind-service"
    alias: audit-service
    condition: services.audit-service.enabled
  
  - name: flowmind-service
    version: "0.1.0"
    repository: "file://../flowmind-service"
    alias: analytics-service
    condition: services.analytics-service.enabled
  
  - name: flowmind-service
    version: "0.1.0"
    repository: "file://../flowmind-service"
    alias: feature-flags-service
    condition: services.feature-flags-service.enabled
  
  - name: flowmind-service
    version: "0.1.0"
    repository: "file://../flowmind-service"
    alias: file-service
    condition: services.file-service.enabled
  
  - name: flowmind-service
    version: "0.1.0"
    repository: "file://../flowmind-service"
    alias: support-service
    condition: services.support-service.enabled
  
  - name: flowmind-service
    version: "0.1.0"
    repository: "file://../flowmind-service"
    alias: settings-service
    condition: services.settings-service.enabled
```

**flowmind-gitops/apps/dev/services-umbrella.yaml:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: flowmind-services
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/flowmind/flowmind-gitops.git
    targetRevision: main
    path: charts/flowmind-services
    helm:
      valueFiles:
        - ../../environments/dev/values-services.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: flowmind-services
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**flowmind-gitops/apps/dev/kong.yaml:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: flowmind-kong
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    # Kong Gateway from official Helm repo
    - repoURL: https://charts.konghq.com
      chart: kong
      targetRevision: 2.33.0
      helm:
        releaseName: kong
        valueFiles:
          - $values/environments/dev/values-kong.yaml
    # Kong Configuration
    - repoURL: https://github.com/flowmind/flowmind-gitops.git
      targetRevision: main
      path: charts/kong-config
      helm:
        releaseName: kong-config
    # Values
    - repoURL: https://github.com/flowmind/flowmind-gitops.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: kong
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 📋 Implementation Checklist

### Week 1: Monorepo Foundation
- [ ] Create `flowmind-platform` repository
- [ ] Initialize pnpm + Nx workspace
- [ ] Create shared libraries structure
- [ ] Setup base Dockerfile templates
- [ ] Configure ESLint, Prettier, TypeScript
- [ ] Create first service scaffold (account-service)
- [ ] Setup local docker-compose for development

### Week 2: IDP & API Gateway
- [ ] Create `flowmind-gitops` repository
- [ ] Setup Kubernetes cluster (K3s) on VM
- [ ] Create Helm chart structure
- [ ] Deploy PostgreSQL via Helm
- [ ] Deploy Keycloak with flowmind realm
- [ ] Configure Keycloak clients and mappers
- [ ] Deploy Kong API Gateway via Helm
- [ ] Create Kong configuration chart
- [ ] Test authentication flow

### Week 3: CI/CD Pipeline
- [ ] Create GitHub Actions CI workflow
- [ ] Setup affected service detection
- [ ] Configure Docker image builds
- [ ] Setup GitHub Container Registry
- [ ] Create GitOps Helm values update workflow
- [ ] Setup PR checks (security, Helm lint)
- [ ] Test full CI/CD pipeline

### Week 4: ArgoCD & Deployments
- [ ] Install ArgoCD on cluster
- [ ] Configure ArgoCD with GitOps repo
- [ ] Create App of Apps pattern with Helm
- [ ] Deploy infrastructure via ArgoCD
- [ ] Deploy first service via ArgoCD
- [ ] Setup automated sync policies
- [ ] Configure health checks

### Week 5: Complete Setup
- [ ] Deploy all remaining services
- [ ] Configure service-to-service auth
- [ ] Setup monitoring (basic metrics)
- [ ] Configure log aggregation
- [ ] Create developer documentation
- [ ] Setup local development workflow
- [ ] End-to-end testing

---

## 🎯 Quick Reference Commands

```bash
# Local Development
cd flowmind-platform
pnpm install                           # Install all dependencies
pnpm --filter @flowmind/account-service dev  # Run single service
docker-compose -f docker/compose/docker-compose.infra.yml up -d  # Start local infra

# Build & Test
pnpm nx run-many --target=build --all  # Build all services
pnpm nx run-many --target=test --all   # Test all services
pnpm nx affected --target=build        # Build only affected

# Helm Commands
cd flowmind-gitops
helm dependency update charts/flowmind-services  # Update dependencies
helm template flowmind-services charts/flowmind-services -f environments/dev/values-services.yaml  # Render templates
helm lint charts/flowmind-service                # Lint chart
helm install flowmind-services charts/flowmind-services -f environments/dev/values-services.yaml -n flowmind-services --dry-run  # Test install

# Kubernetes
kubectl get pods -n flowmind-services  # Check service pods
kubectl logs -f deployment/account-service -n flowmind-services  # View logs
kubectl port-forward svc/kong-proxy -n kong 8080:80  # Access API Gateway

# ArgoCD
kubectl port-forward svc/argocd-server -n argocd 8080:443  # ArgoCD UI
argocd app sync flowmind-services      # Manual sync
argocd app list                        # List all apps
argocd app get flowmind-services       # Get app details
argocd app diff flowmind-services      # Show diff

# Debugging
kubectl exec -it deployment/account-service -n flowmind-services -- sh  # Shell into pod
kubectl describe pod <pod-name> -n flowmind-services  # Pod details
helm get values flowmind-services -n flowmind-services  # Get deployed values
```

---

## 🔒 Secrets Management

For production, consider using:
- **Sealed Secrets**: Encrypted secrets stored in Git
- **External Secrets Operator**: Sync from AWS/Azure/GCP secret managers
- **HashiCorp Vault**: Enterprise secret management

**Example: External Secrets with Helm values:**

```yaml
# In values-services.yaml
externalSecrets:
  enabled: true
  secretStore:
    name: vault-backend
    kind: ClusterSecretStore
  
  secrets:
    account-service:
      - secretKey: database-url
        remoteRef:
          key: flowmind/account-service
          property: database-url
```

**External Secrets Helm Chart Template:**

```yaml
{{- if .Values.externalSecrets.enabled }}
{{- range $name, $service := .Values.services }}
{{- if $service.enabled }}
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ $name }}-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: {{ $.Values.externalSecrets.secretStore.kind }}
    name: {{ $.Values.externalSecrets.secretStore.name }}
  target:
    name: {{ $name }}-secrets
  data:
    {{- range $.Values.externalSecrets.secrets }}
    {{- if eq (index . $name) true }}
    - secretKey: {{ .secretKey }}
      remoteRef:
        key: {{ .remoteRef.key }}
        property: {{ .remoteRef.property }}
    {{- end }}
    {{- end }}
{{- end }}
{{- end }}
{{- end }}
```

---

## 📈 Next Steps After Setup

1. **Observability Stack** (via Helm)
   ```bash
   # Add Prometheus community repo
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring
   
   # Add Grafana Loki
   helm repo add grafana https://grafana.github.io/helm-charts
   helm install loki grafana/loki-stack -n monitoring
   ```

2. **Security Hardening**
   - Network policies
   - Pod security policies
   - RBAC refinement
   - TLS everywhere

3. **Scaling Considerations**
   - Horizontal Pod Autoscaler (already in chart)
   - Resource quotas per namespace
   - Database connection pooling

---

*Document Version: 1.1 | FlowMind Technologies | November 2025*
