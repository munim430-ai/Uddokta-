# Uddokta — Technical Requirements Document (TRD)
### Document 2 of 6 | Version 1.0

---

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Component Interactions & Data Flow](#2-component-interactions--data-flow)
3. [Data & Security Architecture](#3-data--security-architecture)
4. [Third-Party APIs & Open-Source Integrations](#4-third-party-apis--open-source-integrations)
5. [AI Integration & Growth Architecture](#5-ai-integration--growth-architecture)
6. [Deployment & DevOps Strategy](#6-deployment--devops-strategy)
7. [Error Handling & Resilience](#7-error-handling--resilience)

---

## 1. System Architecture Overview

### 1.1 Macro-Level Architecture

Uddokta is a **multi-layer, multi-runtime system** composed of three distinct deployment zones that must operate as a unified product:

```
┌─────────────────────────────────────────────────────────────────────┐
│  ZONE A: Vercel Edge Network (Global CDN)                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Next.js 14+ App (App Router, PWA)                          │   │
│  │  • React Server Components (RSC) for dashboard rendering    │   │
│  │  • Client Components for real-time inbox and charts         │   │
│  │  • shadcn/ui + Tailwind CSS (mobile-first breakpoints)      │   │
│  │  • API Routes → proxy layer for internal service calls      │   │
│  └─────────────────────────────────────────────────────────────┘   │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ HTTPS / WSS
┌───────────────────────────────▼─────────────────────────────────────┐
│  ZONE B: Supabase Cloud (Managed PostgreSQL + Auth + Realtime)      │
│  ┌──────────────────────┐  ┌───────────────────┐  ┌─────────────┐  │
│  │  PostgreSQL (RLS)    │  │  Supabase Auth     │  │  Realtime   │  │
│  │  • tenants           │  │  • OTP/Magic Link  │  │  • Channel  │  │
│  │  • orders            │  │  • JWT issuance    │  │    pub/sub  │  │
│  │  • conversations     │  │  • Row-level        │  │  • Postgres │  │
│  │  • contacts          │  │    identity bind   │  │    Changes  │  │
│  │  • audit_logs        │  └───────────────────┘  └─────────────┘  │
│  └──────────────────────┘                                           │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ Internal Docker Network
┌───────────────────────────────▼─────────────────────────────────────┐
│  ZONE C: Oracle Cloud Free Tier ARM VPS (4 vCPU / 24 GB RAM)        │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │  Chatwoot         │  │  n8n             │  │  Evolution API   │  │
│  │  (Port 3000)      │  │  (Port 5678)     │  │  (Port 8080)     │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │  Typebot          │  │  Twenty CRM      │  │  Listmonk        │  │
│  │  (Port 3001)      │  │  (Port 3000)     │  │  (Port 9000)     │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │  Dify             │  │  MoneyPrinter    │  │  Caddy Reverse   │  │
│  │  (Port 80/int)    │  │  Turbo (Port     │  │  Proxy           │  │
│  │                   │  │  8501)           │  │  (Ports 80/443)  │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  │
│                                                                      │
│  Shared Docker networks:  uddokta_internal  /  uddokta_external     │
└──────────────────────────────────────────────────────────────────────┘
```

### 1.2 Zone A — Next.js Application (Vercel)

The Next.js app is the **only user-facing surface**. It never communicates directly with Dockerized services from the browser. All cross-zone traffic is proxied through Next.js API routes to avoid CORS exposure and to centralize auth validation.

**Key architectural decisions:**

- **App Router with RSC**: Dashboard pages are server-rendered using RSC, with Supabase SSR client (`@supabase/ssr`) for authenticated data fetching. This eliminates client-side waterfalls for initial page loads on mobile networks.
- **PWA via `next-pwa`**: Configured with a `workbox` service worker for offline shell caching. Sellers on low-bandwidth Bangladeshi networks can view their last-synced order data while offline.
- **No direct service calls from browser**: The browser communicates only with `supabase.co` (via the Supabase JS client for Realtime subscriptions) and `/api/*` routes on Vercel. Chatwoot, n8n, and Evolution API are never exposed to the browser directly.
- **Environment segregation**: `NEXT_PUBLIC_*` vars are safe for the browser. All internal service URLs, API keys for Chatwoot, n8n tokens, and Evolution API keys are server-side only (`process.env.*`), injected at Vercel build/runtime.

### 1.3 Zone B — Supabase (Database Authority)

Supabase is the **single source of truth** for all business data. It is not a thin wrapper — it owns:

- All tenant and user records with full RLS enforcement (see Section 3).
- All orders, contacts, conversation metadata, and audit trails.
- Realtime Change Data Capture (CDC) events, pushed to the Next.js dashboard via Supabase Realtime channels.
- Auth token issuance via JWTs signed with the Supabase project secret.

Supabase's free tier limitation is **500 MB database storage and 2 GB bandwidth/month**. At launch, this is sufficient. The architecture must be designed so that binary assets (images, video files from MoneyPrinterTurbo) are **never stored in Supabase** — they go to Cloudflare R2 or the Oracle VPS's local volume.

### 1.4 Zone C — Oracle VPS (Self-Hosted Engine Room)

The Oracle ARM VPS (4 vCPU, 24 GB RAM, 200 GB block storage) runs all self-hosted services via Docker Compose. **This is the highest operational risk zone.** Oracle's free tier can terminate instances with minimal notice, and ARM architecture creates subtle incompatibilities with some Docker images built for `amd64`.

**Critical constraint**: All Docker images used must support `linux/arm64` or be built with multi-arch manifests. Images that do not have ARM builds must be built from source or substituted.

Confirmed ARM-compatible images:
- `chatwoot/chatwoot` — official multi-arch
- `n8nio/n8n` — official multi-arch
- `atendai/evolution-api` — check manifest before deploy; may require source build
- `twentyhq/twenty` — check manifest; recent versions have ARM support
- `listmonk` — official multi-arch
- `langgenius/dify` — multi-arch available via official Compose file
- `baptisteArno/typebot.io` — multi-arch available

**MoneyPrinterTurbo** (`harry0703/MoneyPrinterTurbo`) relies on Python ML dependencies. GPU acceleration will not be available on Oracle's ARM free tier. It must run in CPU-only mode — video generation will be slow (estimated 3–8 minutes per video clip) and must be invoked as an async background job, not a synchronous request.

---

## 2. Component Interactions & Data Flow

### 2.1 Primary Lifecycle: Incoming WhatsApp Message

This is the most critical data path in the product.

```
1.  WhatsApp User sends message
        │
        ▼
2.  Meta Cloud API / WhatsApp Business API
    POST https://vps.uddokta.com/evolution/webhook/{instanceId}
        │
        ▼
3.  Evolution API (Zone C, Port 8080)
    • Validates webhook signature (X-Hub-Signature-256)
    • Normalizes message payload to a standard envelope
    • Fires event to n8n via outbound webhook
    POST http://n8n:5678/webhook/evolution-inbound
        │
        ▼
4.  n8n Workflow: "evolution-inbound" trigger node
    • Parses contact phone number, message body, media
    • Queries Supabase REST API:
      GET /rest/v1/contacts?phone=eq.{phone}&tenant_id=eq.{tenantId}
    • If contact not found → creates contact record in Supabase
    • Upserts conversation record in Supabase
    • Determines routing rule (assigned agent, tag, label)
        │
        ├──→ 5a. Chatwoot API: Create/update conversation
        │        POST http://chatwoot:3000/api/v1/profile
        │        Creates a conversation in the correct inbox
        │        Associates with contact record
        │
        └──→ 5b. Supabase REST API: Insert into `messages` table
                 { tenant_id, conversation_id, body, direction, created_at }
                 This triggers Supabase Realtime CDC
                       │
                       ▼
             6.  Supabase Realtime
                 Broadcasts change event on channel:
                 `tenant:{tenantId}:conversations`
                       │
                       ▼
             7.  Next.js Dashboard (Zone A)
                 Supabase JS client receives Realtime event
                 React state updates → new message badge / notification
                 No polling. No websocket to VPS required.
```

### 2.2 Outbound Message Lifecycle (Agent Replies)

```
1.  Seller/Agent types reply in Next.js Dashboard
        │
        ▼
2.  Next.js API Route: POST /api/messages/send
    • Validates Supabase JWT (server-side via @supabase/ssr)
    • Extracts tenant_id from JWT claim
    • Forwards to Evolution API via internal service call
        │
        ▼
3.  Evolution API: POST /message/sendText/{instanceId}
    { number: "+8801XXXXXXXXX", text: "..." }
        │
        ▼
4.  WhatsApp Cloud API → End user receives message
        │
        ▼
5.  Evolution API fires delivery webhook back to n8n
    n8n updates message status in Supabase:
    UPDATE messages SET status='delivered' WHERE ...
        │
        ▼
6.  Supabase Realtime broadcasts status update
    Dashboard UI updates message tick status
```

### 2.3 New Seller Onboarding Flow (Lead Magnet → Trial)

```
1.  Paid/organic traffic lands on Typebot-embedded
    lead capture page on Next.js (/audit or /start)
        │
        ▼
2.  Typebot (Zone C, Port 3001)
    Conversational flow asks:
    • Business name
    • Facebook Page URL
    • Average daily DM volume
    • Phone number (WhatsApp)
        │
        ▼
3.  Typebot Webhook on completion fires to n8n:
    POST http://n8n:5678/webhook/typebot-lead
        │
        ▼
4.  n8n Workflow: "typebot-lead"
    • Deduplicates by phone/email in Supabase `leads` table
    • If new: inserts lead record
    • Creates contact in Twenty CRM via Twenty API
      POST http://twenty:3000/api/people { ... }
    • Triggers Apify Actor: "fb-page-auditor" with Facebook URL
    • Sends WhatsApp welcome message via Evolution API
    • Enrolls in Listmonk email sequence: "Trial Onboarding"
        │
        ▼
5.  Apify Actor completes (async, up to 5 minutes)
    POST to n8n callback webhook with scraped data:
    { followers, post_frequency, engagement_rate, ... }
        │
        ▼
6.  n8n Workflow: "apify-audit-callback"
    • Calls Dify API to generate personalized "Inbox Leakage Score"
      report using scraped metrics as context
    • Stores audit report in Supabase `lead_audits` table
    • Sends WhatsApp message to lead with audit summary
    • Updates Twenty CRM deal stage to "Audit Delivered"
```

### 2.4 Internal Sales Pipeline Event Flow

```
Supabase `orders` INSERT/UPDATE
        │ (Supabase Realtime CDC)
        ▼
n8n Workflow: "order-events"
    • On new order: → Notify assigned agent via Chatwoot private note
    • On order status = "cod_failed": → Flag in Twenty CRM, trigger
      follow-up WhatsApp template message via Evolution API
    • On order status = "delivered": → Trigger post-purchase survey
      via Typebot embed link in WhatsApp
    • Daily 09:00 BDT cron: → Aggregate stats, generate PDF report,
      POST to Next.js API route /api/reports/daily
```

---

## 3. Data & Security Architecture

### 3.1 Database Schema — Core Tables

All tables reside in Supabase PostgreSQL. The schema enforces multi-tenancy at the database level, not the application level. Application-level filtering is treated as a convenience, never a security boundary.

```sql
-- Tenant registry (one row per Uddokta business account)
CREATE TABLE public.tenants (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug         TEXT UNIQUE NOT NULL,       -- e.g. "shop-a-fashion"
  name         TEXT NOT NULL,
  plan         TEXT NOT NULL DEFAULT 'trial', -- trial | starter | pro
  meta_waba_id TEXT,                        -- WhatsApp Business Account ID
  created_at   TIMESTAMPTZ DEFAULT now()
);

-- Users (mapped to Supabase Auth users)
CREATE TABLE public.profiles (
  id          UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  tenant_id   UUID NOT NULL REFERENCES public.tenants(id) ON DELETE CASCADE,
  role        TEXT NOT NULL DEFAULT 'agent', -- owner | admin | agent
  full_name   TEXT,
  created_at  TIMESTAMPTZ DEFAULT now()
);

-- Contacts (customers of a tenant)
CREATE TABLE public.contacts (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id   UUID NOT NULL REFERENCES public.tenants(id),
  phone       TEXT NOT NULL,
  name        TEXT,
  platform    TEXT NOT NULL DEFAULT 'whatsapp', -- whatsapp | facebook
  tags        TEXT[],
  created_at  TIMESTAMPTZ DEFAULT now(),
  UNIQUE(tenant_id, phone)
);

-- Conversations
CREATE TABLE public.conversations (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id        UUID NOT NULL REFERENCES public.tenants(id),
  contact_id       UUID NOT NULL REFERENCES public.contacts(id),
  status           TEXT NOT NULL DEFAULT 'open', -- open | resolved | snoozed
  assigned_to      UUID REFERENCES public.profiles(id),
  chatwoot_conv_id BIGINT,   -- foreign key to Chatwoot's internal ID
  last_message_at  TIMESTAMPTZ,
  created_at       TIMESTAMPTZ DEFAULT now()
);

-- Messages
CREATE TABLE public.messages (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id       UUID NOT NULL REFERENCES public.tenants(id),
  conversation_id UUID NOT NULL REFERENCES public.conversations(id),
  direction       TEXT NOT NULL, -- inbound | outbound
  body            TEXT,
  media_url       TEXT,
  status          TEXT NOT NULL DEFAULT 'sent', -- sent | delivered | read | failed
  platform_msg_id TEXT,         -- WhatsApp message ID for dedup
  created_at      TIMESTAMPTZ DEFAULT now()
);

-- Orders
CREATE TABLE public.orders (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id     UUID NOT NULL REFERENCES public.tenants(id),
  contact_id    UUID REFERENCES public.contacts(id),
  amount        NUMERIC(10,2),
  currency      TEXT NOT NULL DEFAULT 'BDT',
  status        TEXT NOT NULL DEFAULT 'pending',
  -- pending | confirmed | shipped | delivered | cod_failed | returned
  notes         TEXT,
  created_at    TIMESTAMPTZ DEFAULT now()
);

-- Audit log (immutable)
CREATE TABLE public.audit_logs (
  id          BIGSERIAL PRIMARY KEY,
  tenant_id   UUID NOT NULL,
  actor_id    UUID,           -- profiles.id, NULL for system events
  action      TEXT NOT NULL,
  target_type TEXT,
  target_id   UUID,
  metadata    JSONB,
  created_at  TIMESTAMPTZ DEFAULT now()
);
```

### 3.2 Row Level Security (RLS) Policies

RLS is **enabled on every table**. There are no exceptions. The policies use a PostgreSQL function to extract `tenant_id` from the authenticated JWT's `app_metadata` claim. This is injected by a Supabase Auth Hook on user creation.

```sql
-- Helper function: extract tenant_id from JWT
CREATE OR REPLACE FUNCTION public.current_tenant_id()
RETURNS UUID
LANGUAGE sql STABLE
AS $$
  SELECT (current_setting('request.jwt.claims', true)::jsonb
          -> 'app_metadata' ->> 'tenant_id')::UUID;
$$;

-- Helper function: current user role within tenant
CREATE OR REPLACE FUNCTION public.current_user_role()
RETURNS TEXT
LANGUAGE sql STABLE
AS $$
  SELECT (current_setting('request.jwt.claims', true)::jsonb
          -> 'app_metadata' ->> 'role')::TEXT;
$$;

-- RLS: tenants
ALTER TABLE public.tenants ENABLE ROW LEVEL SECURITY;

CREATE POLICY "tenant_isolation" ON public.tenants
  FOR ALL USING (id = public.current_tenant_id());

-- RLS: contacts (agents can read all, only admins/owners can delete)
ALTER TABLE public.contacts ENABLE ROW LEVEL SECURITY;

CREATE POLICY "contacts_tenant_isolation" ON public.contacts
  FOR ALL USING (tenant_id = public.current_tenant_id());

-- RLS: conversations
ALTER TABLE public.conversations ENABLE ROW LEVEL SECURITY;

CREATE POLICY "conversations_tenant_isolation" ON public.conversations
  FOR ALL USING (tenant_id = public.current_tenant_id());

-- Agent-scoped read: agents only see assigned conversations
-- Admins/owners see all. This is enforced at app layer for perf;
-- the DB policy is a hard floor, not the primary filter.

-- RLS: messages
ALTER TABLE public.messages ENABLE ROW LEVEL SECURITY;

CREATE POLICY "messages_tenant_isolation" ON public.messages
  FOR ALL USING (tenant_id = public.current_tenant_id());

-- RLS: orders
ALTER TABLE public.orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY "orders_tenant_isolation" ON public.orders
  FOR ALL USING (tenant_id = public.current_tenant_id());

-- RLS: audit_logs (read-only for all authenticated users in tenant)
ALTER TABLE public.audit_logs ENABLE ROW LEVEL SECURITY;

CREATE POLICY "audit_logs_tenant_read" ON public.audit_logs
  FOR SELECT USING (tenant_id = public.current_tenant_id());

-- No INSERT policy on audit_logs for authenticated users.
-- All inserts happen via a service_role key from n8n only.
```

**Service Role Key Usage**: The n8n instance on the Oracle VPS uses the Supabase `service_role` key. This key bypasses RLS entirely. It is stored as a Docker secret and never exposed in any other environment. n8n workflows that write to Supabase are responsible for explicitly including `tenant_id` in every write operation. An n8n workflow that omits `tenant_id` will write a row with a null tenant that no authenticated user can read — this is a data integrity issue, not a security breach, but must be audited.

### 3.3 Webhook Signature Verification

Every inbound webhook to n8n that originates from external services must be verified before processing. Unsigned webhooks from internal Docker services (e.g., Evolution API → n8n on the `uddokta_internal` network) rely on network isolation rather than signatures.

**Meta / WhatsApp Webhooks:**
```
Verification: X-Hub-Signature-256 header
Algorithm: HMAC-SHA256(rawBody, META_APP_SECRET)
Enforcement: n8n "Function" node at the HEAD of every Meta webhook workflow.
Any workflow that does not start with signature validation is considered
a critical security defect.
```

**Typebot → n8n:**
```
Typebot sends a secret token in a custom header: X-Typebot-Secret
n8n verifies this in a Function node before processing.
```

**Apify → n8n callbacks:**
```
Apify supports a `customData` field that is echoed back in the webhook.
A signed HMAC token is embedded in customData on Actor initiation
and verified on callback receipt.
```

### 3.4 Tenant Provisioning

When a new seller completes onboarding, a **Supabase Edge Function** (`provision-tenant`) is invoked with the service role. It:

1. Creates a row in `public.tenants`.
2. Creates the Supabase Auth user via `admin.createUser`.
3. Calls `admin.updateUserById` to set `app_metadata: { tenant_id, role: "owner" }`.
4. Calls Chatwoot API to create an Account and a user within it.
5. Calls Twenty CRM API to create a Workspace.
6. Calls n8n API to enable the tenant-specific workflow set.
7. Writes an `audit_logs` entry for the provisioning event.

All steps in this function are wrapped in a try/catch with a compensating rollback mechanism. If step 4 fails, the Supabase user created in step 2 is deleted before throwing an error.

---

## 4. Third-Party APIs & Open-Source Integrations

### 4.1 Evolution API (WhatsApp Bridge)

**Repository**: `evolution-foundation/evolution-api`  
**Deployment**: Docker container on Oracle VPS, Port 8080  
**Purpose**: Provides a unified REST API over both the official WhatsApp Cloud API (for production tenants) and the Baileys-based WhatsApp Web bridge (for demo/testing environments).

**Configuration per tenant:**
```json
{
  "instanceName": "tenant-{tenantSlug}",
  "integration": "WHATSAPP-CLOUD",
  "token": "{EVOLUTION_GLOBAL_API_KEY}",
  "business_id": "{META_WABA_ID}",
  "access_token": "{META_PERMANENT_TOKEN}",
  "phone_number_id": "{META_PHONE_NUMBER_ID}",
  "webhook": {
    "url": "http://n8n:5678/webhook/evolution-inbound",
    "by_events": true,
    "events": [
      "MESSAGES_UPSERT",
      "MESSAGES_UPDATE",
      "CONNECTION_UPDATE",
      "SEND_MESSAGE"
    ]
  }
}
```

**Critical API calls used:**
```
POST /instance/create              — Create per-tenant WhatsApp instance
POST /instance/connect/{name}      — Get QR code for Baileys demo mode
POST /message/sendText/{name}      — Send text message
POST /message/sendMedia/{name}     — Send image/video/document
GET  /instance/fetchInstances      — List active instances for health check
```

**Rate limit handling**: The WhatsApp Cloud API enforces a 1,000 message/second limit at the WABA level. Evolution API does not internally queue; rate limiting must be implemented in n8n using a "Wait" node with exponential backoff (see Section 7).

### 4.2 Chatwoot (Omnichannel Inbox Engine)

**Repository**: `chatwoot/chatwoot`  
**Deployment**: Docker container, Port 3000  
**Purpose**: Provides the headless inbox engine. Chatwoot is used for its conversation state management, agent assignment logic, and canned response library. The Next.js dashboard renders a custom UI that reads from Supabase (for low-latency Realtime), but conversation actions (assign, label, resolve) are executed via the Chatwoot API to keep Chatwoot as the state authority for inbox operations.

**Integration pattern**: Chatwoot is run in "headless" mode from the seller's perspective — sellers never see the Chatwoot UI. The Next.js dashboard provides the entire UX. Chatwoot serves purely as the backend engine.

**API Endpoints consumed:**
```
POST /api/v1/accounts/{accountId}/conversations
     — Create conversation when new WhatsApp contact initiates
GET  /api/v1/accounts/{accountId}/conversations?status=open
     — Polled by n8n for SLA breach detection
POST /api/v1/accounts/{accountId}/conversations/{convId}/assignments
     — Assign conversation to agent
POST /api/v1/accounts/{accountId}/conversations/{convId}/labels
     — Tag conversations (e.g., "cod-order", "complaint")
POST /api/v1/accounts/{accountId}/conversations/{convId}/messages
     — Create private notes (internal agent notes, not sent to customer)
PATCH /api/v1/accounts/{accountId}/conversations/{convId}
     — Update status (open → resolved)
```

**Auth**: Chatwoot uses a per-account `user_access_token`. This is stored in Supabase `tenants.chatwoot_token` (encrypted at rest via Supabase's `pgsodium` extension). n8n retrieves the token before each API call.

**Webhook from Chatwoot to n8n:**
```
Chatwoot fires outbound webhooks to n8n on:
- conversation_created
- conversation_status_changed
- message_created (outbound only, to avoid duplicate inbound processing)
- conversation_assignment
n8n uses these to keep Supabase `conversations` table in sync.
```

### 4.3 n8n (Workflow Orchestration)

**Repository**: `n8n-io/n8n`  
**Deployment**: Docker container, Port 5678  
**Purpose**: The central nervous system. All business logic, routing decisions, and inter-service communication are encoded as n8n workflows. No business logic lives in Next.js API routes beyond authentication and proxying.

**n8n instance configuration:**
```yaml
environment:
  N8N_BASIC_AUTH_ACTIVE: "true"
  N8N_BASIC_AUTH_USER: "${N8N_ADMIN_USER}"
  N8N_BASIC_AUTH_PASSWORD: "${N8N_ADMIN_PASSWORD}"
  N8N_ENCRYPTION_KEY: "${N8N_ENCRYPTION_KEY}"   # 32-byte random hex
  WEBHOOK_URL: "https://vps.uddokta.com/n8n/"
  DB_TYPE: "postgresdb"
  DB_POSTGRESDB_HOST: "${SUPABASE_DB_HOST}"
  DB_POSTGRESDB_DATABASE: "n8n"
  DB_POSTGRESDB_USER: "${N8N_DB_USER}"
  DB_POSTGRESDB_PASSWORD: "${N8N_DB_PASSWORD}"
  EXECUTIONS_DATA_SAVE_ON_ERROR: "all"
  EXECUTIONS_DATA_SAVE_ON_SUCCESS: "none"   # reduce storage
  EXECUTIONS_DATA_MAX_AGE: "168"            # 7 days retention
```

**n8n runs on a separate PostgreSQL database**, not Supabase's primary DB. This avoids n8n's internal schema polluting the application schema and allows independent backup/restore cycles. This secondary PostgreSQL instance runs as a Docker container on the Oracle VPS.

**Core workflow inventory:**
| Workflow Name | Trigger | Purpose |
|---|---|---|
| `evolution-inbound` | Webhook (Evolution API) | Route inbound WhatsApp messages |
| `chatwoot-sync` | Webhook (Chatwoot) | Sync Chatwoot state changes to Supabase |
| `typebot-lead` | Webhook (Typebot) | Process new leads from onboarding flow |
| `apify-audit-callback` | Webhook (Apify) | Handle Facebook page audit results |
| `order-events` | Supabase Realtime (via HTTP poll) | Trigger alerts on order status changes |
| `daily-report-cron` | Cron (09:00 BDT) | Generate and deliver daily seller reports |
| `sla-breach-check` | Cron (every 15 min) | Alert on unanswered conversations > 2h |
| `moneyprinturbo-trigger` | Manual/Scheduled | Queue video generation jobs |

**n8n API usage from Next.js (admin operations):**
```
GET  /api/v1/workflows          — List workflows for status dashboard
POST /api/v1/workflows/{id}/activate  — Activate workflow for new tenant
GET  /api/v1/executions         — Fetch execution logs for debugging
```

### 4.4 Typebot (Conversational Intake)

**Repository**: `baptisteArno/typebot.io`  
**Deployment**: Docker container, Port 3001 (viewer) + Port 3002 (builder)  
**Purpose**: Powers interactive onboarding flows and the "Free Audit" lead magnet.

**Integration method**: Typebot bots are embedded in the Next.js marketing pages via the official `@typebot-io/js` embed SDK. The bot flows are built in the Typebot builder UI and published to the self-hosted viewer instance.

**Webhook from Typebot:**
When a Typebot flow completes, a Webhook block fires to n8n:
```json
{
  "sessionId": "string",
  "results": {
    "business_name": "string",
    "facebook_url": "string",
    "daily_dm_volume": "string",
    "phone": "string",
    "email": "string"
  }
}
```
This triggers the `typebot-lead` n8n workflow.

**Self-hosting note**: Typebot requires a separate PostgreSQL database and an S3-compatible bucket (use Cloudflare R2 on the free tier) for storing uploaded files in bot flows.

### 4.5 Twenty CRM (Internal Sales Pipeline)

**Repository**: `twentyhq/twenty`  
**Deployment**: Docker container, Port 3000 (conflicts with Chatwoot — Twenty must be remapped to Port 3003)  
**Purpose**: Internal sales pipeline visibility. Used by the Uddokta sales team, not by end-user sellers.

**Integration**: n8n is the only component that writes to Twenty via its API. No direct integration from Next.js.

**Twenty API (GraphQL):**
```graphql
# Create a new lead (Person)
mutation CreatePerson($data: PersonCreateInput!) {
  createPerson(data: $data) {
    id
    name { firstName lastName }
    phones { primaryPhoneNumber }
  }
}

# Create a deal (Opportunity)
mutation CreateOpportunity($data: OpportunityCreateInput!) {
  createOpportunity(data: $data) {
    id stage amount
  }
}

# Update deal stage
mutation UpdateOpportunity($id: ID!, $data: OpportunityUpdateInput!) {
  updateOpportunity(id: $id, data: $data) {
    id stage
  }
}
```

n8n uses the Twenty API token (stored as an n8n credential) to authenticate all GraphQL requests.

### 4.6 Listmonk (Email Automation)

**Repository**: `knadh/listmonk`  
**Deployment**: Docker container, Port 9000  
**Purpose**: Transactional and drip email campaigns for lead nurturing and seller notifications.

**Integration from n8n:**
```
POST /api/subscribers          — Create/update subscriber on new lead
POST /api/campaigns/{id}/start — Start campaign (manual trigger from n8n)
POST /api/tx                   — Send a single transactional email
```

Listmonk requires an outbound SMTP relay. Use AWS SES (free tier: 62,000 emails/month when sent from EC2; otherwise $0.10/1,000) or Brevo's free SMTP tier (300 emails/day).

---

## 5. AI Integration & Growth Architecture

### 5.1 Architecture Overview

The AI growth engine operates as an **autonomous lead-generation pipeline** that runs independently of the core product. It is orchestrated entirely by n8n workflows and produces two outputs: (1) personalized audit reports that convert leads, and (2) video content that drives top-of-funnel traffic.

```
┌─────────────────────────────────────────────────────────┐
│              AI Growth Engine (Zone C)                  │
│                                                         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────┐  │
│  │  Apify Cloud │───▶│  Dify / Dify │───▶│  n8n     │  │
│  │  (Scraping)  │    │  (AI Agents) │    │  (Orch.) │  │
│  └──────────────┘    └──────────────┘    └──────────┘  │
│                                                   │     │
│  ┌──────────────────────────────────────────┐    │     │
│  │  MoneyPrinterTurbo (Video Generation)    │◀───┘     │
│  └──────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────┘
```

### 5.2 Apify — Lead Scraping & Data Extraction

**Library**: `apify/apify-cli` + Apify Cloud  
**Deployment**: Apify Cloud (SaaS). The CLI is used for local development and Actor deployment. Scraping does not run on the Oracle VPS — Apify's infrastructure handles it.

**Apify Actors used:**

**Actor 1: `uddokta/fb-page-auditor`** (custom, to be built)
- Input: Facebook page URL
- Process: Scrapes public page data — follower count, post frequency (last 30 days), average engagement rate, response time indicator, product catalogue presence.
- Output: Structured JSON to n8n callback webhook.
- Trigger: Invoked by n8n via Apify REST API on new lead submission.

```javascript
// n8n HTTP Request node to trigger Apify actor
POST https://api.apify.com/v2/acts/uddokta~fb-page-auditor/runs
Authorization: Bearer {APIFY_API_TOKEN}
{
  "startUrls": [{ "url": "{facebook_page_url}" }],
  "webhooks": [{
    "eventTypes": ["ACTOR.RUN.SUCCEEDED"],
    "requestUrl": "https://vps.uddokta.com/n8n/webhook/apify-audit-callback",
    "headersTemplate": {
      "X-Apify-Secret": "{HMAC_SIGNED_TOKEN}"
    },
    "payloadTemplate": "{\"leadId\": \"{leadId}\", \"datasetId\": \"{{defaultDatasetId}}\"}"
  }]
}
```

**Actor 2: `uddokta/fb-commerce-prospector`** (custom, to be built)
- Input: Bangladesh F-commerce related Facebook group/page search terms.
- Process: Scrapes public posts to find sellers advertising COD products. Extracts business name, Facebook page URL, phone numbers from post text.
- Output: CSV to Apify dataset, fetched by n8n on a daily cron.
- This data feeds directly into Twenty CRM as new Opportunities.

**Apify free tier limitation**: 5 Actor compute units/month. Each audit run consumes approximately 0.05–0.1 compute units. This allows ~50–100 free audits/month before billing begins ($0.25/CU thereafter). This must be tracked, and a circuit breaker in n8n must halt Actor invocations when the monthly usage threshold is reached.

### 5.3 Dify — Agentic AI Orchestration

**Repository**: `langgenius/dify`  
**Deployment**: Docker Compose on Oracle VPS (Dify ships its own `docker-compose.yaml`; it must be merged into the master Uddokta Compose file carefully, as Dify includes its own PostgreSQL, Redis, and Nginx containers that would conflict with the main stack. Use `network_mode: uddokta_internal` and remap ports.)

**Purpose**: Dify provides the AI agent that transforms raw Apify scrape data into personalized "Inbox Leakage Audit" reports.

**Dify application: `inbox-leak-audit-generator`**

This is a Dify Workflow application (not a chatbot) with the following node structure:

```
INPUT: {
  business_name, follower_count, post_frequency_30d,
  avg_engagement_rate, response_time_label,
  daily_dm_volume_estimate
}
  │
  ▼
[LLM Node — GPT-4o-mini or Claude Haiku via API key]
System Prompt: "You are an F-commerce growth consultant for Bangladesh.
  Given the following Facebook page metrics for {business_name},
  calculate an 'Inbox Leakage Score' (0-100 where 100 = maximum leakage).
  Provide: score, 3 key reasons for leakage, 3 specific improvements.
  Respond in Bangla. Format as JSON."
  │
  ▼
[Output: structured JSON]
  {
    score: 78,
    reasons: ["...", "...", "..."],
    improvements: ["...", "...", "..."],
    urgency_label: "High"
  }
```

n8n calls this Dify workflow via HTTP:
```
POST http://dify:80/v1/workflows/run
Authorization: Bearer {DIFY_APP_API_KEY}
{
  "inputs": { ...scrapedMetrics },
  "response_mode": "blocking",
  "user": "{leadId}"
}
```

**Dify self-hosting constraint**: Dify's full stack (PostgreSQL, Redis, Weaviate vector DB, Nginx, API server, worker, web UI) consumes approximately 3–4 GB RAM under idle load. On a 24 GB Oracle VPS shared with the rest of the stack, this is acceptable, but the Weaviate vector store should be disabled initially (set `VECTOR_STORE=none` in Dify's `.env`) to reduce memory footprint. Dify can operate without RAG/vector search for this use case.

### 5.4 MoneyPrinterTurbo — AI Video Marketing Production

**Repository**: `harry0703/MoneyPrinterTurbo`  
**Deployment**: Docker container on Oracle VPS, Port 8501 (Streamlit UI) + internal API  
**Purpose**: Generates short-form video content (Reels/TikToks) in Bangla to educate F-commerce sellers about "Inbox Leakage" — top-of-funnel brand awareness content.

**Technical constraints on Oracle ARM:**
- No GPU. All video generation runs on CPU.
- Python ML dependencies (`torch`, `transformers`, `edge-tts`) must be installed with ARM-compatible builds. Use `torch==2.2.0+cpu` from PyTorch's ARM pip index.
- Expected generation time per 60-second video: 5–15 minutes on 4 vCPU ARM.
- Storage: Each generated video is ~20–80 MB. Videos must be immediately uploaded to Cloudflare R2 (S3-compatible, free tier: 10 GB storage, no egress fees) and deleted from the VPS local volume to avoid disk exhaustion.

**Integration pattern (n8n → MoneyPrinterTurbo):**

MoneyPrinterTurbo exposes a Streamlit UI, not a production API. For programmatic control, a **thin FastAPI wrapper** must be built around the core `VideoService` class in MoneyPrinterTurbo. This wrapper exposes a single endpoint:

```python
# uddokta-mpt-wrapper/main.py
from fastapi import FastAPI, BackgroundTasks
from app.services.video_service import VideoService

app = FastAPI()

@app.post("/generate")
async def generate_video(payload: dict, background_tasks: BackgroundTasks):
    job_id = create_job_id()
    background_tasks.add_task(run_generation, job_id, payload)
    return {"job_id": job_id, "status": "queued"}

@app.get("/status/{job_id}")
def get_status(job_id: str):
    return get_job_status(job_id)  # reads from Redis or SQLite
```

n8n triggers this wrapper endpoint:
```
POST http://moneyprinturbo-wrapper:8502/generate
{
  "script": "আপনার ইনবক্সে প্রতিদিন কতজন ক্রেতা হারাচ্ছেন জানেন?...",
  "voice": "bn-BD-NabanitaNeural",   // Bangla TTS via edge-tts
  "video_terms": ["ecommerce bangladesh", "online shopping"],
  "output_resolution": "1080x1920"   // 9:16 for Reels
}
```

n8n polls `/status/{job_id}` on a 2-minute interval until `status === "completed"`, then fetches the R2 URL and posts it to a Slack/WhatsApp notification for the content team to review before publishing.

**Content pipeline cadence**: Videos are generated on a Monday/Wednesday/Friday cron schedule. Each run generates 1–2 videos. This keeps CPU load isolated to off-peak hours. A `MAX_CONCURRENT_JOBS=1` constraint must be enforced in the wrapper to prevent OOM.

### 5.5 Unified AI Cost Model

| Service | Free Tier | Paid Trigger |
|---|---|---|
| Apify | 5 CU/month | >100 audits/month |
| OpenAI (via Dify) | None | Every Dify LLM call |
| MoneyPrinterTurbo TTS (edge-tts) | Free (local) | Never (runs locally) |
| Cloudflare R2 (video storage) | 10 GB/month | >10 GB stored |

To control OpenAI costs via Dify, use **Claude Haiku** (via Anthropic API) as the default LLM for the audit generator — it is significantly cheaper than GPT-4o-mini for structured JSON output tasks and supports Bangla adequately. Budget estimate: ~$0.002 per audit report at 500 input + 300 output tokens.

---

## 6. Deployment & DevOps Strategy

### 6.1 Repository Structure

```
uddokta/
├── apps/
│   └── web/                    # Next.js application (deployed to Vercel)
├── infra/
│   ├── docker-compose.yml      # Master Compose file (Zone C)
│   ├── docker-compose.override.yml  # Local dev overrides
│   ├── .env.example            # Template for all environment variables
│   ├── caddy/
│   │   └── Caddyfile           # Reverse proxy config
│   ├── n8n/
│   │   └── workflows/          # Exported n8n workflow JSON files
│   └── supabase/
│       └── migrations/         # SQL migration files (supabase db push)
├── packages/
│   └── mpt-wrapper/            # FastAPI wrapper for MoneyPrinterTurbo
└── docs/                       # All 6 planning documents
```

### 6.2 Master Docker Compose

```yaml
# infra/docker-compose.yml
version: "3.8"

networks:
  uddokta_internal:
    driver: bridge
  uddokta_external:
    driver: bridge

volumes:
  postgres_n8n_data:
  chatwoot_storage:
  evolution_instances:
  typebot_db:
  twenty_db:
  listmonk_data:
  dify_data:
  r2_staging:          # local staging before R2 upload

services:

  # ── Reverse Proxy ──────────────────────────────────────────────
  caddy:
    image: caddy:2-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_certs:/config
    networks: [uddokta_external, uddokta_internal]
    restart: unless-stopped

  # ── n8n Database ────────────────────────────────────────────────
  postgres_n8n:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: n8n
      POSTGRES_USER: ${N8N_DB_USER}
      POSTGRES_PASSWORD: ${N8N_DB_PASSWORD}
    volumes:
      - postgres_n8n_data:/var/lib/postgresql/data
    networks: [uddokta_internal]
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${N8N_DB_USER} -d n8n"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── n8n ─────────────────────────────────────────────────────────
  n8n:
    image: n8nio/n8n:latest
    depends_on:
      postgres_n8n:
        condition: service_healthy
    environment:
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres_n8n
      DB_POSTGRESDB_DATABASE: n8n
      DB_POSTGRESDB_USER: ${N8N_DB_USER}
      DB_POSTGRESDB_PASSWORD: ${N8N_DB_PASSWORD}
      N8N_ENCRYPTION_KEY: ${N8N_ENCRYPTION_KEY}
      WEBHOOK_URL: https://vps.uddokta.com/n8n/
      N8N_BASIC_AUTH_ACTIVE: "true"
      N8N_BASIC_AUTH_USER: ${N8N_ADMIN_USER}
      N8N_BASIC_AUTH_PASSWORD: ${N8N_ADMIN_PASSWORD}
      EXECUTIONS_DATA_SAVE_ON_ERROR: all
      EXECUTIONS_DATA_MAX_AGE: "168"
    volumes:
      - n8n_data:/home/node/.n8n
    networks: [uddokta_internal]
    restart: unless-stopped

  # ── Evolution API ───────────────────────────────────────────────
  evolution:
    image: atendai/evolution-api:latest
    environment:
      SERVER_URL: https://vps.uddokta.com/evolution
      AUTHENTICATION_API_KEY: ${EVOLUTION_API_KEY}
      DATABASE_ENABLED: "true"
      DATABASE_CONNECTION_URI: ${EVOLUTION_DB_URI}
      WEBHOOK_GLOBAL_ENABLED: "true"
      WEBHOOK_GLOBAL_URL: http://n8n:5678/webhook/evolution-inbound
    volumes:
      - evolution_instances:/evolution/instances
    networks: [uddokta_internal]
    restart: unless-stopped

  # ── Chatwoot ─────────────────────────────────────────────────────
  chatwoot_app:
    image: chatwoot/chatwoot:latest
    depends_on: [chatwoot_postgres, chatwoot_redis]
    environment:
      RAILS_ENV: production
      SECRET_KEY_BASE: ${CHATWOOT_SECRET_KEY}
      FRONTEND_URL: https://vps.uddokta.com/chatwoot
      DEFAULT_LOCALE: bn
      POSTGRES_HOST: chatwoot_postgres
      POSTGRES_DATABASE: chatwoot
      POSTGRES_USERNAME: ${CHATWOOT_DB_USER}
      POSTGRES_PASSWORD: ${CHATWOOT_DB_PASSWORD}
      REDIS_URL: redis://chatwoot_redis:6379
      ACTIVE_STORAGE_SERVICE: local
    volumes:
      - chatwoot_storage:/app/storage
    networks: [uddokta_internal]
    restart: unless-stopped
    command: bundle exec rails s -p 3000 -b 0.0.0.0

  chatwoot_sidekiq:
    image: chatwoot/chatwoot:latest
    depends_on: [chatwoot_postgres, chatwoot_redis]
    environment: *chatwoot-env   # YAML anchor from chatwoot_app
    networks: [uddokta_internal]
    restart: unless-stopped
    command: bundle exec sidekiq -C config/sidekiq.yml

  chatwoot_postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: chatwoot
      POSTGRES_USER: ${CHATWOOT_DB_USER}
      POSTGRES_PASSWORD: ${CHATWOOT_DB_PASSWORD}
    networks: [uddokta_internal]
    restart: unless-stopped

  chatwoot_redis:
    image: redis:7-alpine
    networks: [uddokta_internal]
    restart: unless-stopped

  # ── Typebot ──────────────────────────────────────────────────────
  typebot_db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: typebot
      POSTGRES_USER: ${TYPEBOT_DB_USER}
      POSTGRES_PASSWORD: ${TYPEBOT_DB_PASSWORD}
    volumes:
      - typebot_db:/var/lib/postgresql/data
    networks: [uddokta_internal]
    restart: unless-stopped

  typebot_builder:
    image: baptistearno/typebot-builder:latest
    depends_on: [typebot_db]
    environment:
      DATABASE_URL: postgresql://${TYPEBOT_DB_USER}:${TYPEBOT_DB_PASSWORD}@typebot_db/typebot
      NEXTAUTH_URL: https://vps.uddokta.com/typebot-builder
      NEXTAUTH_SECRET: ${TYPEBOT_SECRET}
      S3_ENDPOINT: ${R2_ENDPOINT}
      S3_ACCESS_KEY: ${R2_ACCESS_KEY}
      S3_SECRET_KEY: ${R2_SECRET_KEY}
      S3_BUCKET: typebot-assets
    networks: [uddokta_internal]
    restart: unless-stopped

  typebot_viewer:
    image: baptistearno/typebot-viewer:latest
    depends_on: [typebot_db]
    environment:
      DATABASE_URL: postgresql://${TYPEBOT_DB_USER}:${TYPEBOT_DB_PASSWORD}@typebot_db/typebot
      NEXTAUTH_URL: https://vps.uddokta.com/typebot
      NEXTAUTH_SECRET: ${TYPEBOT_SECRET}
      S3_ENDPOINT: ${R2_ENDPOINT}
    networks: [uddokta_internal]
    restart: unless-stopped

  # ── Twenty CRM ───────────────────────────────────────────────────
  twenty:
    image: twentyhq/twenty:latest
    depends_on: [twenty_db, twenty_redis]
    environment:
      PG_DATABASE_URL: postgresql://${TWENTY_DB_USER}:${TWENTY_DB_PASSWORD}@twenty_db/twenty
      REDIS_URL: redis://twenty_redis:6379
      SERVER_URL: https://vps.uddokta.com/twenty
      APP_SECRET: ${TWENTY_APP_SECRET}
    networks: [uddokta_internal]
    restart: unless-stopped

  twenty_db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: twenty
      POSTGRES_USER: ${TWENTY_DB_USER}
      POSTGRES_PASSWORD: ${TWENTY_DB_PASSWORD}
    volumes:
      - twenty_db:/var/lib/postgresql/data
    networks: [uddokta_internal]
    restart: unless-stopped

  twenty_redis:
    image: redis:7-alpine
    networks: [uddokta_internal]
    restart: unless-stopped

  # ── Listmonk ─────────────────────────────────────────────────────
  listmonk:
    image: listmonk/listmonk:latest
    depends_on: [listmonk_db]
    environment:
      LISTMONK_db__host: listmonk_db
      LISTMONK_db__user: ${LISTMONK_DB_USER}
      LISTMONK_db__password: ${LISTMONK_DB_PASSWORD}
      LISTMONK_db__database: listmonk
    volumes:
      - listmonk_data:/listmonk/uploads
    networks: [uddokta_internal]
    restart: unless-stopped

  listmonk_db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: listmonk
      POSTGRES_USER: ${LISTMONK_DB_USER}
      POSTGRES_PASSWORD: ${LISTMONK_DB_PASSWORD}
    networks: [uddokta_internal]
    restart: unless-stopped

  # ── Dify ─────────────────────────────────────────────────────────
  # Dify ships its own docker-compose.yaml. Extract and merge the
  # api, worker, web, sandbox services here. Omit Dify's own nginx,
  # postgres, and redis — reuse chatwoot_redis and create a dify_db.
  dify_api:
    image: langgenius/dify-api:latest
    depends_on: [dify_db, chatwoot_redis]
    environment:
      MODE: api
      SECRET_KEY: ${DIFY_SECRET_KEY}
      DB_USERNAME: ${DIFY_DB_USER}
      DB_PASSWORD: ${DIFY_DB_PASSWORD}
      DB_HOST: dify_db
      DB_DATABASE: dify
      REDIS_HOST: chatwoot_redis
      VECTOR_STORE: none
      STORAGE_TYPE: s3
      S3_ENDPOINT: ${R2_ENDPOINT}
      S3_BUCKET_NAME: dify-assets
      S3_ACCESS_KEY: ${R2_ACCESS_KEY}
      S3_SECRET_KEY: ${R2_SECRET_KEY}
    networks: [uddokta_internal]
    restart: unless-stopped

  dify_worker:
    image: langgenius/dify-api:latest
    depends_on: [dify_db, chatwoot_redis]
    environment:
      MODE: worker
      # ... same env as dify_api
    networks: [uddokta_internal]
    restart: unless-stopped

  dify_db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: dify
      POSTGRES_USER: ${DIFY_DB_USER}
      POSTGRES_PASSWORD: ${DIFY_DB_PASSWORD}
    networks: [uddokta_internal]
    restart: unless-stopped

  # ── MoneyPrinterTurbo Wrapper ─────────────────────────────────────
  mpt_wrapper:
    build:
      context: ../packages/mpt-wrapper
      dockerfile: Dockerfile.arm64
    environment:
      PEXELS_API_KEY: ${PEXELS_API_KEY}
      R2_ENDPOINT: ${R2_ENDPOINT}
      R2_ACCESS_KEY: ${R2_ACCESS_KEY}
      R2_SECRET_KEY: ${R2_SECRET_KEY}
      R2_BUCKET: uddokta-videos
      MAX_CONCURRENT_JOBS: "1"
    volumes:
      - r2_staging:/app/temp
    networks: [uddokta_internal]
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "2.0"    # cap at 2 of 4 vCPUs during generation
          memory: 4G
```

### 6.3 Caddy Reverse Proxy Configuration

All services are exposed under a single domain with path-based routing. No service except Caddy listens on public ports.

```caddyfile
vps.uddokta.com {
  # n8n (webhooks only — restrict builder UI to IP whitelist in prod)
  handle /n8n/* {
    reverse_proxy n8n:5678
  }

  # Evolution API
  handle /evolution/* {
    reverse_proxy evolution:8080
  }

  # Chatwoot (internal use only — consider IP restriction)
  handle /chatwoot/* {
    reverse_proxy chatwoot_app:3000
  }

  # Typebot viewer (public embed)
  handle /typebot/* {
    reverse_proxy typebot_viewer:3000
  }

  # Typebot builder (restrict to admin IPs)
  handle /typebot-builder/* {
    @admin_only remote_ip 103.X.X.X  # team office IP
    handle @admin_only {
      reverse_proxy typebot_builder:3000
    }
    respond "Forbidden" 403
  }

  # Twenty CRM (restrict to admin IPs)
  handle /twenty/* {
    @admin_only remote_ip 103.X.X.X
    handle @admin_only {
      reverse_proxy twenty:3000
    }
    respond "Forbidden" 403
  }

  # Dify
  handle /dify/* {
    reverse_proxy dify_api:5001
  }

  # Listmonk
  handle /listmonk/* {
    reverse_proxy listmonk:9000
  }
}
```

### 6.4 Vercel Deployment (Zone A)

```
apps/web/
├── vercel.json
└── .env.local (gitignored)
```

```json
// vercel.json
{
  "framework": "nextjs",
  "regions": ["sin1"],
  "env": {
    "NEXT_PUBLIC_SUPABASE_URL": "@supabase_url",
    "NEXT_PUBLIC_SUPABASE_ANON_KEY": "@supabase_anon_key",
    "SUPABASE_SERVICE_ROLE_KEY": "@supabase_service_role_key",
    "VPS_N8N_WEBHOOK_BASE": "@vps_n8n_webhook_base",
    "VPS_EVOLUTION_BASE": "@vps_evolution_base",
    "VPS_CHATWOOT_BASE": "@vps_chatwoot_base",
    "CHATWOOT_AGENT_TOKEN": "@chatwoot_agent_token"
  }
}
```

The `sin1` (Singapore) region is selected for lowest latency to Bangladesh.

**Vercel free tier constraints**: 100 GB bandwidth/month, 6,000 minutes serverless function runtime/month. API routes with streaming (e.g., Dify LLM streaming responses) must be implemented as Edge Runtime routes to avoid the 10-second serverless timeout on the free tier.

### 6.5 Supabase Migrations

All database changes are managed via the Supabase CLI:
```bash
supabase db diff --file <migration_name>   # generate migration
supabase db push                           # apply to remote
```

Migration files are committed to `infra/supabase/migrations/` and executed in order. No ad-hoc schema changes via the Supabase Studio UI are permitted in any environment beyond initial development.

---

## 7. Error Handling & Resilience

### 7.1 Idempotency for Webhooks

Every webhook handler in n8n **must be idempotent**. WhatsApp and Meta webhooks can deliver the same event multiple times due to network retries. Without idempotency, messages are duplicated in the database and double-delivered to sellers.

**Implementation:**

Every inbound message webhook from Evolution API contains a `platform_msg_id` (WhatsApp's message ID, e.g., `wamid.XXXX`). Before processing, n8n executes an upsert:

```sql
-- n8n "Supabase" node: Execute Query
INSERT INTO public.messages (
  tenant_id, conversation_id, direction, body, platform_msg_id, status
)
VALUES (
  '{{ $json.tenant_id }}',
  '{{ $json.conversation_id }}',
  'inbound',
  '{{ $json.body }}',
  '{{ $json.platform_msg_id }}',
  'received'
)
ON CONFLICT (platform_msg_id) DO NOTHING;
```

A unique index on `messages(platform_msg_id)` enforces this at the database level:
```sql
CREATE UNIQUE INDEX messages_platform_msg_id_idx
ON public.messages(platform_msg_id)
WHERE platform_msg_id IS NOT NULL;
```

If `ON CONFLICT DO NOTHING` affects zero rows, n8n's subsequent nodes must detect this and halt the workflow branch (check for zero rows returned from the upsert; n8n's Supabase node returns the affected rows).

### 7.2 Retry Queue for Failed Message Delivery

When an outbound message fails (Evolution API returns a non-200 or WhatsApp returns an error code), n8n must not silently drop the failure.

**Retry mechanism using n8n + PostgreSQL:**

```
1. n8n workflow "send-message" attempts Evolution API call
2. On HTTP 4xx/5xx → catches error in "Error Trigger" node
3. Inserts into `public.message_retry_queue` table:
   {
     message_id, tenant_id, payload, attempt_count: 1,
     next_retry_at: NOW() + INTERVAL '30 seconds',
     max_attempts: 5,
     error_code, error_body
   }
4. n8n cron workflow "retry-queue-processor" runs every 30 seconds:
   - Fetches rows WHERE next_retry_at <= NOW() AND attempt_count < max_attempts
   - Attempts resend via Evolution API
   - On success: deletes row, updates message status to 'sent'
   - On failure: updates attempt_count, sets next_retry_at with exponential backoff:
     next_retry_at = NOW() + INTERVAL '(30 * 2^attempt_count) seconds'
     -- 30s → 60s → 120s → 240s → 480s
5. After max_attempts exceeded:
   - Updates message status to 'permanently_failed'
   - Creates Chatwoot private note alerting the assigned agent
   - Deletes retry queue row
```

```sql
CREATE TABLE public.message_retry_queue (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id       UUID NOT NULL,
  message_id      UUID REFERENCES public.messages(id),
  payload         JSONB NOT NULL,
  attempt_count   INT NOT NULL DEFAULT 1,
  max_attempts    INT NOT NULL DEFAULT 5,
  next_retry_at   TIMESTAMPTZ NOT NULL,
  last_error_code TEXT,
  last_error_body TEXT,
  created_at      TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX message_retry_queue_next_retry_idx
ON public.message_retry_queue(next_retry_at)
WHERE attempt_count < max_attempts;
```

### 7.3 WhatsApp Cloud API Rate Limit Handling

The WhatsApp Cloud API rate limits are at two levels:
- **Per-phone-number**: 80 messages/second
- **Per-WABA**: Tiered based on quality rating and business verification

When Evolution API relays a `(131056) Message failed to send because there were too many messages sent from this phone number in a short period of time` error:

n8n must:
1. Detect error code `131056` in the Evolution webhook or HTTP response.
2. Activate "rate limit mode" for the tenant: set a Redis key `rl:{tenantId}` with a 60-second TTL.
3. All outbound messages for that tenant are enqueued in `message_retry_queue` with a forced `next_retry_at = NOW() + 65 seconds` (longer than the cooldown window).
4. Alert the tenant's assigned n8n workflow monitor via a Chatwoot system note.

**Note**: Listmonk campaigns must also be throttled. Listmonk's `batch_size` and `sleep_interval` settings must be configured to respect SMTP provider rate limits. For Brevo free tier (300 emails/day), set `batch_size=25` and `sleep_interval=300` seconds.

### 7.4 Oracle VPS Failure Fallback

The Oracle VPS is a **single point of failure** for all Zone C services. There is no redundancy on the free tier. The following mitigations reduce, but do not eliminate, downtime impact:

**Data durability**: Supabase (Zone B) is the primary data store. If the VPS goes down, the Next.js dashboard can still render from Supabase data. Sellers lose the ability to send/receive messages but retain read access to their order history and contacts.

**VPS backup strategy:**
- Docker volumes are backed up to Cloudflare R2 daily via a cron job on the VPS:
  ```bash
  # runs at 02:00 BDT via crontab
  docker run --rm -v postgres_n8n_data:/data -v /tmp:/backup \
    alpine tar czf /backup/postgres_n8n_$(date +%Y%m%d).tar.gz /data
  # rclone upload to R2
  rclone copy /tmp/postgres_n8n_*.tar.gz r2:uddokta-backups/
  rm /tmp/postgres_n8n_*.tar.gz
  ```
- n8n workflow exports (JSON) are committed to the Git repository after every workflow change. This allows full n8n workflow restoration from source within 30 minutes on a fresh VPS.

**Instance recoverability**: The entire Zone C stack must be restorable from a bare Oracle VPS instance in under 60 minutes by running:
```bash
git clone https://github.com/uddokta/infra && cd infra
cp .env.example .env && vim .env  # fill secrets
docker compose up -d
```

No procedural knowledge beyond populating the `.env` file should be required.

### 7.5 Supabase Realtime Disconnection Handling

Supabase Realtime uses WebSockets. Mobile clients on Bangladeshi networks experience frequent connection drops. The Next.js dashboard must implement automatic reconnection with exponential backoff using the Supabase JS client's built-in reconnection logic:

```typescript
// apps/web/lib/supabase-realtime.ts
const channel = supabase
  .channel(`tenant:${tenantId}:conversations`)
  .on('postgres_changes',
    { event: 'INSERT', schema: 'public', table: 'messages',
      filter: `tenant_id=eq.${tenantId}` },
    (payload) => handleNewMessage(payload)
  )
  .subscribe((status, err) => {
    if (status === 'CHANNEL_ERROR') {
      console.error('Realtime channel error:', err);
      // Supabase client auto-reconnects; show "reconnecting..." indicator in UI
      setRealtimeStatus('reconnecting');
    }
    if (status === 'SUBSCRIBED') {
      setRealtimeStatus('connected');
      // On reconnect: fetch missed messages since last received timestamp
      refetchMissedMessages(lastMessageTimestamp);
    }
  });
```

The `refetchMissedMessages` call on reconnect is critical. The Realtime subscription will have missed events during the disconnection window. The dashboard must query Supabase REST API for messages with `created_at > lastMessageTimestamp` to fill the gap.

### 7.6 Dify / LLM API Failure Handling

When the Dify workflow call fails (LLM API timeout, rate limit, or Dify container restart), n8n must not block the lead generation pipeline.

**Graceful degradation**: If Dify returns a non-200 or times out after 30 seconds, n8n:
1. Logs the failure to `audit_logs` with `action = 'dify_audit_failed'`.
2. Sends the lead a generic WhatsApp message: "আপনার অডিট রিপোর্ট প্রস্তুত হচ্ছে। ২৪ ঘণ্টার মধ্যে পাঠানো হবে।" ("Your audit report is being prepared. It will be sent within 24 hours.")
3. Stores the raw Apify data in Supabase `leads.raw_apify_data` for manual review or retry.
4. Schedules a retry via `message_retry_queue` with a 1-hour `next_retry_at`.

This ensures that a transient Dify failure does not result in a silent lead drop.

---

*End of Document 2: Technical Requirements Document*

*Next document: Document 3 — Product Requirements Document (PRD) / Feature Specification*
