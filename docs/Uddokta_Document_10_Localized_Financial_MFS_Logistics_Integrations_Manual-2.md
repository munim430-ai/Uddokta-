# Uddokta — Document 10: Localized Financial, MFS & Logistics Integrations Manual

**Document type:** Localized Financial, MFS & Logistics Integrations Manual  
**Product:** Uddokta  
**Audience:** AI coding agents, integrations engineers, fintech engineer, operations lead, support lead  
**Status:** Build-ready technical planning document  
**Primary stack:** Next.js PWA, Supabase/PostgreSQL, n8n, Chatwoot, Typebot, WhatsApp Cloud API, Evolution API for controlled demos, Google Sheets export, local courier adapters  

---

## 0. Source Discipline and Anti-Hallucination Rules

This document is intentionally strict because local Bangladesh integrations are inconsistent. Courier and MFS APIs may be private, account-specific, unstable, undocumented publicly, or changed without notice. AI coding agents must **never invent external provider payloads**.

### 0.1 Grounded source base

This manual is aligned with the existing Uddokta documents:

- **PRD:** Uddokta is a mobile-first sales, order, follow-up, customer-record, and trust-control platform for Bangladeshi SMEs using Facebook, WhatsApp, screenshots, bKash/Nagad proof, COD delivery, and manual staff communication.
- **Business Blueprint:** Uddokta hides API/webhook complexity from sellers and exposes only a mobile dashboard, order cards, customer records, follow-up controls, reports, and trust profile.
- **TRD:** n8n is the orchestration layer; Supabase is the source of truth; Next.js is the customer-facing wrapper.
- **Setup SOP:** client onboarding collects courier preference, payment methods, channel mode, products, policies, and staff through a structured intake flow.
- **Meta Compliance SOP:** all outbound messaging must remain compliant, throttled, opt-out-aware, logged, and pauseable.

### 0.2 Public provider evidence level

| Provider | Public API detail reliability | Build rule |
|---|---:|---|
| **bKash PRA** | Official public product page confirms PRA use, limits, statement/SMS formats, and retail/f-commerce fit. | Manual/semi-automated verification first. |
| **bKash Payment Gateway** | Official developer docs expose token, create payment, execute/query, and IPN concepts. | Future gateway adapter can be specified abstractly. |
| **Nagad** | Public API docs are not reliably discoverable from official sources. | Treat as manual/semi-automated until official merchant credentials/docs are obtained. |
| **SSLCommerz** | Public/merchant docs exist but must be verified with merchant credentials before coding exact production payloads. | Build adapter interface, not final hardcoded payloads. |
| **Steadfast** | Official site confirms ecommerce delivery, COD, tracking, dashboard; public exact API docs are not consistently official. Community packages expose observed fields. | Use canonical adapter + verified credentialed docs before production. |
| **Pathao Courier** | Official site confirms courier, merchant onboarding, tracking, COD, return charges. Community package exposes merchant API shape and success-rate lookup. | Use canonical adapter + verified credentialed docs before production. |
| **REDX** | Public API details are weak/not consistently available. | CSV/export fallback first; adapter after official docs/access. |

### 0.3 Non-negotiable rules for AI agents

1. **Do not hardcode undocumented provider payloads.** Put provider payload mapping in versioned adapter config and fixtures.
2. **Do not make n8n the source of truth.** n8n orchestrates; Supabase owns state.
3. **Do not trust client-provided order totals.** Recalculate server-side from `order_items`, `delivery_fee`, `discount`, and `cod_amount`.
4. **Do not auto-verify screenshots.** OCR may assist, but human verification remains authoritative until official payment APIs are integrated.
5. **Do not use cross-tenant COD risk data in the client UI unless consent, privacy terms, and legal review approve it.**
6. **Do not expose courier/MFS credentials to the browser.**
7. **Every external request must have idempotency, retry, logging, raw payload storage, and human override.**
8. **Every outbound customer message must pass through the Meta compliance gate and opt-out registry.**
9. **Every table must be workspace/tenant scoped.**
10. **Every automation must be pauseable at tenant and order level.**

---

# 1. Integration Architecture Overview

## 1.1 System of record

Supabase/PostgreSQL is the only source of truth for:

- orders
- customers
- payments
- courier consignments
- verification events
- outbound messages
- webhook events
- risk scores
- audit logs

n8n must never store unique business truth. n8n may store temporary execution data and workflow state, but final state must be written to Supabase.

## 1.2 Integration topology

```text
Seller Mobile Dashboard
        ↓
Next.js API Route
        ↓
Supabase permission check
        ↓
n8n workflow trigger
        ↓
Provider adapter: bKash / Nagad / SSLCommerz / Steadfast / Pathao / REDX / Meta
        ↓
Provider response or webhook
        ↓
n8n normalization
        ↓
Supabase state update
        ↓
Realtime UI update + audit log + optional customer notification
```

## 1.3 Canonical integration record model

Every provider interaction must create or update these record types:

- `integration_requests`
- `integration_responses`
- `webhook_events`
- `payment_attempts`
- `payment_verification_events`
- `courier_shipments`
- `courier_events`
- `outbound_messages`
- `audit_events`

The normalized records must survive even if the provider call fails.

---

# 2. Database Additions for Local Integrations

The existing Uddokta schema already contains workspaces/tenants, customers, conversations, messages, products, orders, followups, trust profiles, reviews, and audit events. Document 10 adds integration-specific tables.

Use `tenant_id` or `workspace_id` consistently depending on the final schema naming. If the project already uses `tenant_id`, use `tenant_id`. If it uses `workspace_id`, use `workspace_id`. Do not mix both inside one codebase.

The examples below use `tenant_id`.

## 2.1 Provider accounts

```sql
create type provider_type as enum (
  'bkash_pra',
  'bkash_gateway',
  'nagad_manual',
  'nagad_gateway',
  'sslcommerz',
  'steadfast',
  'pathao',
  'redx',
  'manual_courier',
  'meta_whatsapp',
  'evolution_demo'
);

create type provider_account_status as enum (
  'draft',
  'pending_credentials',
  'active',
  'paused',
  'error',
  'revoked'
);

create table public.provider_accounts (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid not null references public.tenants(id) on delete cascade,
  provider provider_type not null,
  display_name text not null,
  status provider_account_status not null default 'draft',
  mode text not null default 'manual', -- manual | api | csv | demo
  credential_ref text, -- pointer to vault/encrypted secret, never raw key
  config jsonb not null default '{}',
  last_health_check_at timestamptz,
  last_error text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create index provider_accounts_tenant_provider_idx
on public.provider_accounts (tenant_id, provider, status);
```

## 2.2 Payment attempts

```sql
create type payment_method_type as enum (
  'cod',
  'bkash_pra',
  'bkash_gateway',
  'nagad_manual',
  'nagad_gateway',
  'sslcommerz',
  'bank_transfer',
  'cash',
  'other'
);

create type payment_status_type as enum (
  'unpaid',
  'needs_verification',
  'verified',
  'failed_match',
  'rejected',
  'refunded',
  'cancelled'
);

create table public.payment_attempts (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid not null references public.tenants(id) on delete cascade,
  order_id uuid not null references public.orders(id) on delete cascade,
  customer_id uuid references public.customers(id),
  method payment_method_type not null,
  status payment_status_type not null default 'unpaid',
  expected_amount numeric(12,2) not null default 0,
  claimed_amount numeric(12,2),
  verified_amount numeric(12,2),
  currency text not null default 'BDT',
  sender_phone text,
  receiver_account text,
  trx_id text,
  customer_reference text,
  screenshot_url text,
  raw_text text,
  normalized_text jsonb not null default '{}',
  gateway_payment_id text,
  gateway_transaction_id text,
  provider_account_id uuid references public.provider_accounts(id),
  verification_notes text,
  verified_by uuid,
  verified_at timestamptz,
  failed_reason text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create unique index payment_attempts_unique_trx_per_tenant
on public.payment_attempts (tenant_id, method, trx_id)
where trx_id is not null;
```

## 2.3 Payment verification events

```sql
create table public.payment_verification_events (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid not null references public.tenants(id) on delete cascade,
  payment_attempt_id uuid not null references public.payment_attempts(id) on delete cascade,
  order_id uuid not null references public.orders(id) on delete cascade,
  event_type text not null, -- uploaded | ocr_parsed | manual_verified | failed_match | gateway_ipn | gateway_query
  actor_type text not null default 'system', -- system | staff | admin | provider
  actor_user_id uuid,
  previous_status payment_status_type,
  new_status payment_status_type,
  evidence jsonb not null default '{}',
  raw_payload jsonb not null default '{}',
  created_at timestamptz not null default now()
);

create index payment_verification_events_order_idx
on public.payment_verification_events (tenant_id, order_id, created_at desc);
```

## 2.4 Courier shipments

```sql
create type courier_provider_type as enum (
  'steadfast',
  'pathao',
  'redx',
  'manual'
);

create type shipment_status_type as enum (
  'not_created',
  'draft',
  'queued',
  'api_submitted',
  'provider_review',
  'pickup_requested',
  'picked_up',
  'in_transit',
  'out_for_delivery',
  'delivered',
  'partial_delivered',
  'return_requested',
  'returned',
  'cancelled',
  'failed',
  'unknown'
);

create table public.courier_shipments (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid not null references public.tenants(id) on delete cascade,
  order_id uuid not null references public.orders(id) on delete cascade,
  customer_id uuid references public.customers(id),
  provider courier_provider_type not null,
  provider_account_id uuid references public.provider_accounts(id),
  status shipment_status_type not null default 'not_created',
  internal_reference text not null,
  provider_consignment_id text,
  provider_tracking_code text,
  provider_invoice text,
  tracking_url text,
  recipient_name text not null,
  recipient_phone text not null,
  recipient_address text not null,
  recipient_city text,
  recipient_zone text,
  recipient_area text,
  item_description text,
  item_weight numeric(8,2),
  item_quantity int default 1,
  cod_amount numeric(12,2) not null default 0,
  delivery_fee numeric(12,2) default 0,
  special_instruction text,
  outbound_payload jsonb not null default '{}',
  last_provider_payload jsonb not null default '{}',
  created_by uuid,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create unique index courier_shipments_provider_consignment_unique
on public.courier_shipments (tenant_id, provider, provider_consignment_id)
where provider_consignment_id is not null;

create unique index courier_shipments_order_provider_unique
on public.courier_shipments (tenant_id, order_id, provider);
```

## 2.5 Courier events

```sql
create table public.courier_events (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid not null references public.tenants(id) on delete cascade,
  shipment_id uuid references public.courier_shipments(id) on delete cascade,
  order_id uuid not null references public.orders(id) on delete cascade,
  provider courier_provider_type not null,
  provider_event_id text,
  provider_consignment_id text,
  provider_status_raw text,
  normalized_status shipment_status_type not null default 'unknown',
  event_time timestamptz,
  location text,
  actor text,
  raw_payload jsonb not null default '{}',
  created_at timestamptz not null default now()
);

create unique index courier_events_dedupe_idx
on public.courier_events (tenant_id, provider, provider_event_id)
where provider_event_id is not null;

create index courier_events_order_idx
on public.courier_events (tenant_id, order_id, created_at desc);
```

## 2.6 Webhook events

```sql
create type webhook_source_type as enum (
  'meta',
  'evolution',
  'bkash_ipn',
  'sslcommerz_ipn',
  'courier',
  'typebot',
  'internal'
);

create table public.webhook_events (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references public.tenants(id) on delete set null,
  source webhook_source_type not null,
  provider text,
  event_key text not null,
  external_event_id text,
  signature_valid boolean,
  idempotency_key text not null,
  processing_status text not null default 'received', -- received | processed | duplicate | failed | ignored
  raw_headers jsonb not null default '{}',
  raw_payload jsonb not null default '{}',
  normalized_payload jsonb not null default '{}',
  error_message text,
  received_at timestamptz not null default now(),
  processed_at timestamptz
);

create unique index webhook_events_idempotency_unique
on public.webhook_events (source, idempotency_key);
```

## 2.7 Integration request log

```sql
create table public.integration_requests (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid not null references public.tenants(id) on delete cascade,
  provider provider_type not null,
  provider_account_id uuid references public.provider_accounts(id),
  operation text not null,
  related_order_id uuid references public.orders(id),
  related_payment_attempt_id uuid references public.payment_attempts(id),
  related_shipment_id uuid references public.courier_shipments(id),
  idempotency_key text not null,
  request_payload jsonb not null default '{}',
  response_payload jsonb not null default '{}',
  http_status int,
  status text not null default 'pending', -- pending | success | retrying | failed | cancelled
  error_code text,
  error_message text,
  attempt_count int not null default 0,
  next_retry_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create unique index integration_requests_idempotency_unique
on public.integration_requests (tenant_id, provider, operation, idempotency_key);
```

## 2.8 COD risk profile

```sql
create type risk_level_type as enum (
  'unknown',
  'low',
  'medium',
  'high',
  'blocked'
);

create table public.customer_risk_profiles (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid not null references public.tenants(id) on delete cascade,
  customer_id uuid not null references public.customers(id) on delete cascade,
  phone_normalized text not null,
  risk_level risk_level_type not null default 'unknown',
  internal_order_count int not null default 0,
  internal_delivered_count int not null default 0,
  internal_returned_count int not null default 0,
  internal_cancelled_count int not null default 0,
  internal_return_rate numeric(5,2),
  external_success_rate numeric(5,2),
  external_source text,
  external_checked_at timestamptz,
  last_calculated_at timestamptz,
  calculation_version text not null default 'v1',
  reasons jsonb not null default '[]',
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create unique index customer_risk_profiles_tenant_phone_unique
on public.customer_risk_profiles (tenant_id, phone_normalized);
```

---

# 3. MFS Payment Verification Workflow: bKash & Nagad

## 3.1 Early-state reality

Most early Uddokta clients will not have merchant gateway credentials. Many will use:

- bKash Personal Retail Account
- personal bKash number
- Nagad personal/merchant number
- bank transfer
- COD
- screenshot-based confirmation
- TrxID typed by customer

Therefore the first implementation must use a **manual-to-semi-automated payment verification queue**.

## 3.2 Manual screenshot / TrxID upload flow

### Customer-side entry points

A customer may provide payment proof through:

1. WhatsApp image upload
2. Messenger image upload
3. typed TrxID
4. typed amount
5. payment screenshot attached by seller/staff
6. manual note by seller after checking bKash/Nagad app
7. future gateway IPN callback

### Internal data flow

```text
Customer sends payment screenshot or TrxID
        ↓
Message received via Meta/Evolution
        ↓
n8n inbound message router detects payment intent
        ↓
Create or update payment_attempt:
  status = needs_verification
        ↓
Attach screenshot/media URL or raw text
        ↓
Optional OCR/text parsing into normalized fields
        ↓
Seller/staff sees Verification Queue
        ↓
Staff manually compares against bKash/Nagad app or statement
        ↓
Staff marks:
  verified OR failed_match OR rejected
        ↓
Order payment_status updates
        ↓
Receipt/follow-up message may be sent
```

## 3.3 Verification queue lifecycle

### Payment status lifecycle

```text
unpaid
  → needs_verification
  → verified
  → failed_match
  → rejected
  → refunded
  → cancelled
```

### Meaning

| Status | Meaning | Who can set | Customer-facing effect |
|---|---|---|---|
| `unpaid` | No proof received. | system/staff | No payment confirmation. |
| `needs_verification` | Proof/TrxID received but not verified. | system/staff | Staff sees yellow warning. |
| `verified` | Payment matched manually or by gateway. | staff/admin/system gateway | Order can move to confirmed/ready. |
| `failed_match` | Proof does not match amount/phone/date/order. | staff/admin | Customer may receive polite correction request. |
| `rejected` | Fraudulent/invalid/duplicate proof. | owner/admin | Order locked until owner action. |
| `refunded` | Refund completed. | owner/admin | Order marked resolved. |
| `cancelled` | Payment no longer relevant. | staff/admin | Removed from active queue. |

## 3.4 Verification queue UI requirements

Verification Queue screen must show:

- order number
- customer name and phone
- expected amount
- claimed amount
- payment method
- TrxID
- screenshot preview
- extracted OCR text if available
- date/time of proof
- related conversation link
- duplicate TrxID warning
- buttons:
  - `Mark Verified`
  - `Failed Match`
  - `Ask Customer Again`
  - `Reject`
  - `Attach to Order`
  - `View Customer`

## 3.5 Payment proof parser

### Parser inputs

```json
{
  "tenant_id": "uuid",
  "order_id": "uuid",
  "message_id": "uuid",
  "method": "bkash_pra|nagad_manual",
  "raw_text": "You have received Tk 1500 from ... TrxID ABC123 at 03/06/2026 14:12",
  "screenshot_url": "https://...",
  "customer_claimed_trx_id": "ABC123",
  "customer_claimed_amount": 1500
}
```

### Normalized payment proof object

```json
{
  "provider": "bkash",
  "transaction_type": "payment|send_money|merchant_payment|unknown",
  "amount": 1500,
  "currency": "BDT",
  "sender_phone_masked_or_full": "017********",
  "receiver_account": "018********",
  "trx_id": "ABC123",
  "reference": "UDD-1048",
  "timestamp_local": "2026-06-03T14:12:00+06:00",
  "confidence": 0.82,
  "parser_version": "mfs-proof-parser-v1",
  "warnings": ["ocr_low_confidence", "receiver_account_missing"]
}
```

### Parser rule

OCR and regex parsing may only assist staff. It cannot directly mark payment as verified unless:

- official provider IPN/gateway status confirms it, or
- staff manually confirms it, or
- future authorized statement API confirms it.

## 3.6 bKash PRA manual verification logic

bKash PRA is designed for micro and small businesses operating in retail and f-commerce to collect bKash payments. It supports customer payments by PRA QR, payment gateway, or manual PRA number entry. PRA records and SMS/statement formats include fields such as amount, reference, TrxID, and time.

### Required Uddokta fields for bKash PRA

| Uddokta field | Source |
|---|---|
| `receiver_account` | seller PRA number |
| `expected_amount` | order total / advance amount |
| `trx_id` | customer-provided TrxID or screenshot parse |
| `customer_reference` | order number if customer entered it |
| `screenshot_url` | proof attachment |
| `verified_by` | staff/admin who checked bKash app |
| `verified_at` | verification time |
| `failed_reason` | mismatch reason |

### bKash manual verification checklist

Staff must verify:

```text
[ ] TrxID exists in bKash app / PRA statement
[ ] amount matches order expected amount
[ ] receiver account matches seller account
[ ] transaction time is reasonable
[ ] reference/order number matches where available
[ ] screenshot is not obviously edited
[ ] TrxID not already used in another order
```

### bKash duplicate prevention

When a TrxID is entered:

```sql
select id, order_id, status
from payment_attempts
where tenant_id = :tenant_id
and method in ('bkash_pra', 'bkash_gateway')
and trx_id = :trx_id;
```

If found:

- show **Duplicate TrxID** warning
- do not allow staff to verify without owner/admin override
- log `payment_verification_events.event_type = 'duplicate_trx_detected'`

## 3.7 Nagad manual verification logic

No reliable public official Nagad merchant API documentation should be assumed at planning time. Treat Nagad as manual/semi-automated unless a client provides official merchant gateway credentials and docs.

### Required Uddokta fields for Nagad

| Uddokta field | Meaning |
|---|---|
| `method = nagad_manual` | manual verification lane |
| `receiver_account` | seller Nagad number |
| `expected_amount` | order amount |
| `trx_id` | transaction ID from customer |
| `screenshot_url` | proof screenshot |
| `verified_by` | seller/staff verifier |
| `verification_notes` | notes from Nagad app check |

### Nagad duplicate prevention

Same logic as bKash, scoped to `method = nagad_manual`.

## 3.8 Payment-proof mismatch states

### Mismatch reasons

```text
amount_mismatch
trx_id_duplicate
receiver_mismatch
date_time_mismatch
unreadable_screenshot
no_trx_id
customer_sent_wrong_order
suspected_fake_screenshot
staff_unable_to_verify
```

### Customer message templates

**Failed amount match**

```text
আপনার payment proof পেয়েছি, কিন্তু amount/order total match হচ্ছে না।
দয়া করে সঠিক TrxID বা screenshot আবার পাঠান।

Order: {{order_number}}
Expected: ৳{{expected_amount}}
```

**Unclear screenshot**

```text
Screenshot clear না হওয়ায় payment verify করা যাচ্ছে না।
দয়া করে bKash/Nagad TrxID লিখে পাঠান অথবা clear screenshot দিন।
```

**Verified**

```text
Payment verified ✅

Order: {{order_number}}
Amount: ৳{{verified_amount}}

আমরা এখন order process করছি।
```

## 3.9 Future gateway integration: bKash / SSLCommerz / Nagad

### Gateway adapter abstraction

```ts
interface PaymentGatewayAdapter {
  provider: 'bkash_gateway' | 'sslcommerz' | 'nagad_gateway';
  createPayment(input: CreatePaymentInput): Promise<CreatePaymentResult>;
  executePayment?(input: ExecutePaymentInput): Promise<ExecutePaymentResult>;
  queryPayment(input: QueryPaymentInput): Promise<QueryPaymentResult>;
  refundPayment?(input: RefundPaymentInput): Promise<RefundPaymentResult>;
  verifyWebhook(input: WebhookVerificationInput): Promise<boolean>;
  normalizeWebhook(input: unknown): NormalizedPaymentEvent;
}
```

### Token lifecycle for bKash gateway

bKash official developer docs describe a grant-token API that returns an access token (`id_token`) to access other APIs, plus a refresh token and expiry. Therefore the bKash adapter must implement:

```text
provider_account credential_ref
  → fetch app_key/app_secret/username/password from vault
  → request grant token
  → store token metadata encrypted or in provider_token_cache
  → use id_token in create/query/execute payment calls
  → refresh token before expiry
  → never expose id_token to browser
```

### bKash create payment canonical mapping

Internal Uddokta payment request:

```json
{
  "tenant_id": "uuid",
  "order_id": "uuid",
  "order_number": "UDD-1048",
  "amount": "1850.00",
  "currency": "BDT",
  "payer_reference": "017XXXXXXXX",
  "callback_url": "https://app.uddokta.com/api/webhooks/payments/bkash",
  "intent": "Sale"
}
```

bKash tokenized payment docs include fields such as `mode`, `payerReference`, `callbackURL`, `amount`, `currency`, `intent`, and `merchantInvoiceNumber`. AI agents must use bKash's current official docs and merchant credentials before implementation.

### SSLCommerz future abstraction

The SSLCommerz adapter must support:

- payment session/initiation
- transaction validation
- IPN listener
- order/payment ID correlation
- failed/cancelled transaction status
- settlement reconciliation export

Do not code exact SSLCommerz payloads until merchant docs/credentials are available. Build only the `PaymentGatewayAdapter` interface and test fixtures.

### Gateway payment state map

```text
gateway_created
  → customer_redirected
  → gateway_success_callback
  → query_verified
  → verified

gateway_created
  → gateway_failed_callback
  → failed_match

gateway_created
  → gateway_cancelled_callback
  → cancelled

gateway_created
  → no_callback
  → query_pending
  → expired_or_failed
```

---

# 4. Local Courier API Normalization: Steadfast, Pathao, REDX

## 4.1 Courier integration principle

Uddokta must not treat each courier as a completely different product surface. The seller sees one courier workflow:

```text
Order ready
→ select courier
→ create shipment / export CSV
→ tracking code saved
→ customer receives tracking
→ courier event updates order
```

Provider differences are hidden inside adapters.

## 4.2 Canonical shipment creation input

```json
{
  "tenant_id": "uuid",
  "order_id": "uuid",
  "order_number": "UDD-1048",
  "provider": "steadfast|pathao|redx|manual",
  "recipient": {
    "name": "Nusrat Jahan",
    "phone": "01712345678",
    "alternative_phone": "01812345678",
    "address": "House 12, Road 5, Mirpur 10, Dhaka",
    "city": "Dhaka",
    "zone": "Mirpur",
    "area": "Mirpur 10"
  },
  "merchant": {
    "store_id": "provider-store-id-if-required",
    "sender_name": "Seller Name",
    "sender_phone": "018XXXXXXXX",
    "pickup_address": "Seller pickup address"
  },
  "parcel": {
    "item_description": "Skincare Set x1",
    "item_quantity": 1,
    "item_weight": 1.0,
    "item_type": "parcel",
    "delivery_type": "normal",
    "special_instruction": "Call before delivery"
  },
  "cash": {
    "cod_amount": 1850.00,
    "delivery_fee": 130.00,
    "payment_status": "cod"
  }
}
```

## 4.3 Canonical shipment creation result

```json
{
  "success": true,
  "provider": "steadfast",
  "order_id": "uuid",
  "provider_consignment_id": "1424107",
  "provider_tracking_code": "15BAEB8A",
  "provider_invoice": "UDD-1048",
  "tracking_url": "https://...",
  "provider_status_raw": "in_review",
  "normalized_status": "provider_review",
  "raw_response": {}
}
```

## 4.4 Status normalization map

| Canonical Uddokta status | Meaning | Order status effect |
|---|---|---|
| `not_created` | No courier shipment created. | no change |
| `draft` | Shipment prepared but not submitted. | no change |
| `queued` | Waiting for n8n submission. | delivery_pending |
| `api_submitted` | API request accepted, awaiting provider state. | delivery_pending |
| `provider_review` | Provider reviewing/processing. | delivery_pending |
| `pickup_requested` | Pickup requested. | delivery_pending |
| `picked_up` | Courier picked up parcel. | shipped |
| `in_transit` | Parcel moving through network. | shipped |
| `out_for_delivery` | Rider/agent attempting delivery. | shipped |
| `delivered` | Delivered to customer. | delivered |
| `partial_delivered` | Partial delivered/exchanged. | delivered_with_issue |
| `return_requested` | Return process started. | return_pending |
| `returned` | Parcel returned. | returned |
| `cancelled` | Shipment cancelled. | cancelled |
| `failed` | Provider error/failure. | shipment_failed |
| `unknown` | Status cannot be mapped. | no auto change; flag manual review |

## 4.5 Steadfast outbound payload blueprint

### Evidence level

- Official Steadfast site confirms ecommerce delivery, COD collection, online management, real-time tracking, and tracking codes.
- Community Laravel package documentation lists observed Steadfast API fields including `invoice`, `recipient_name`, `recipient_phone`, `recipient_address`, `cod_amount`, `note`, `recipient_email`, `alternative_phone`, `item_description`, `total_lot`, and `delivery_type`.

### Uddokta to Steadfast mapping

| Uddokta canonical field | Steadfast field | Required? | Notes |
|---|---|---:|---|
| `order_number` | `invoice` | Yes | Use Uddokta order number. |
| `recipient.name` | `recipient_name` | Yes | sanitize Bangla/English text. |
| `recipient.phone` | `recipient_phone` | Yes | normalize BD phone. |
| `recipient.address` | `recipient_address` | Yes | minimum useful address length. |
| `cash.cod_amount` | `cod_amount` | Yes | for COD; 0 if prepaid. |
| `parcel.special_instruction` | `note` | Optional | customer/courier instruction. |
| `recipient.alternative_phone` | `alternative_phone` | Optional | if available. |
| `parcel.item_description` | `item_description` | Optional | product summary only. |
| `parcel.item_quantity` | `total_lot` | Optional | confirm with official docs. |
| `parcel.delivery_type` | `delivery_type` | Optional | confirm provider values. |

### n8n adapter output: Steadfast

```json
{
  "invoice": "UDD-1048",
  "recipient_name": "Nusrat Jahan",
  "recipient_phone": "01712345678",
  "recipient_address": "House 12, Road 5, Mirpur 10, Dhaka",
  "cod_amount": 1850,
  "note": "Call before delivery",
  "alternative_phone": "01812345678",
  "item_description": "Skincare Set x1",
  "total_lot": 1,
  "delivery_type": 0
}
```

**Implementation rule:** This object is a blueprint. Final field names/values must be verified against Steadfast credentialed API docs before production.

## 4.6 Pathao outbound payload blueprint

### Evidence level

- Official Pathao Courier page confirms courier delivery, merchant onboarding, live parcel tracking, COD, parcel returns, and merchant registration.
- A community Laravel package for the Pathao Merchant Courier API lists functions for cities/zones/areas/stores, create store, create order, price calculation, order view, and user success rate by phone. It lists create-order fields such as `store_id`, `merchant_order_id`, `sender_name`, `sender_phone`, `recipient_name`, `recipient_phone`, `recipient_address`, `recipient_city`, `recipient_zone`, `recipient_area`, `delivery_type`, `item_type`, `special_instruction`, `item_quantity`, `item_weight`, `amount_to_collect`, and `item_description`.

### Uddokta to Pathao mapping

| Uddokta canonical field | Pathao field | Required? | Notes |
|---|---|---:|---|
| `merchant.store_id` | `store_id` | Yes | configured per tenant/provider. |
| `order_number` | `merchant_order_id` | Optional/Yes | use Uddokta order number. |
| `merchant.sender_name` | `sender_name` | Yes | store/seller contact. |
| `merchant.sender_phone` | `sender_phone` | Yes | pickup contact. |
| `recipient.name` | `recipient_name` | Yes | customer name. |
| `recipient.phone` | `recipient_phone` | Yes | normalize phone. |
| `recipient.address` | `recipient_address` | Yes | minimum viable address. |
| `recipient.city` | `recipient_city` | Yes | provider city ID, not raw text. |
| `recipient.zone` | `recipient_zone` | Yes | provider zone ID. |
| `recipient.area` | `recipient_area` | Yes | provider area ID. |
| `parcel.delivery_type` | `delivery_type` | Yes | provider enum; verify. |
| `parcel.item_type` | `item_type` | Yes | provider enum; verify. |
| `parcel.special_instruction` | `special_instruction` | Optional | delivery note. |
| `parcel.item_quantity` | `item_quantity` | Yes | numeric. |
| `parcel.item_weight` | `item_weight` | Yes | numeric kg. |
| `cash.cod_amount` | `amount_to_collect` | Yes | COD collection amount. |
| `parcel.item_description` | `item_description` | Optional | product summary. |

### Location normalization requirement

Pathao requires structured location IDs, not only free-text address. Uddokta must maintain provider location mapping tables:

```sql
create table public.courier_locations (
  id uuid primary key default gen_random_uuid(),
  provider courier_provider_type not null,
  type text not null, -- city | zone | area
  provider_location_id text not null,
  parent_provider_location_id text,
  name_en text,
  name_bn text,
  aliases text[] default '{}',
  raw_payload jsonb not null default '{}',
  updated_at timestamptz not null default now(),
  unique(provider, type, provider_location_id)
);
```

### n8n adapter output: Pathao

```json
{
  "store_id": 12345,
  "merchant_order_id": "UDD-1048",
  "sender_name": "Seller Name",
  "sender_phone": "01812345678",
  "recipient_name": "Nusrat Jahan",
  "recipient_phone": "01712345678",
  "recipient_address": "House 12, Road 5, Mirpur 10, Dhaka",
  "recipient_city": 1,
  "recipient_zone": 12,
  "recipient_area": 144,
  "delivery_type": 48,
  "item_type": 2,
  "special_instruction": "Call before delivery",
  "item_quantity": 1,
  "item_weight": 1.0,
  "amount_to_collect": 1850,
  "item_description": "Skincare Set x1"
}
```

**Implementation rule:** Use this only as an adapter blueprint from community-observed docs. Verify current Pathao Merchant API payload and enum values with official credentials before production.

## 4.7 REDX outbound payload blueprint

Public REDX API payloads are not reliably available. Therefore REDX starts as:

1. manual courier selection,
2. CSV/Excel export,
3. tracking-code manual entry,
4. API adapter only after official docs/access.

### REDX manual shipment fields

| Field | Required |
|---|---:|
| order number | Yes |
| customer name | Yes |
| customer phone | Yes |
| customer address | Yes |
| area/city | Yes |
| product summary | Yes |
| COD amount | Yes |
| delivery instruction | Optional |
| tracking code | entered after booking |

## 4.8 Inbound courier webhook handlers

Because public webhook structures are not reliable, Uddokta must use a **provider-agnostic webhook envelope**.

### Expected inbound envelope after n8n normalization

```json
{
  "tenant_id": "uuid",
  "provider": "steadfast|pathao|redx",
  "provider_event_id": "string",
  "provider_consignment_id": "string",
  "provider_tracking_code": "string",
  "merchant_order_id": "UDD-1048",
  "provider_status_raw": "parcel_delivered",
  "normalized_status": "delivered",
  "event_time": "2026-06-03T14:32:00+06:00",
  "location": "Mirpur Hub",
  "raw_payload": {}
}
```

### Matching priority

When webhook arrives, match in this order:

1. `tenant_id + provider + provider_consignment_id`
2. `tenant_id + provider + provider_tracking_code`
3. `tenant_id + provider + merchant_order_id`
4. manual review queue if no match

Never update an order from a courier webhook if tenant cannot be resolved confidently.

### Courier event flow

```text
Courier webhook received
        ↓
Verify signature/token if provider supports it
        ↓
Compute idempotency_key
        ↓
Insert into webhook_events
        ↓
If duplicate: mark duplicate and stop
        ↓
Normalize provider status
        ↓
Find shipment/order
        ↓
Insert courier_event
        ↓
Update courier_shipments.status
        ↓
Update orders.status where allowed
        ↓
Trigger customer alert if configured
        ↓
Audit log
```

## 4.9 Portable export fallback: CSV/Excel

The fallback is mandatory. Many sellers will prefer manual courier upload or may not have API credentials.

### Export principles

- Export must work even if all courier APIs are unavailable.
- Export must be tenant-scoped.
- Export must be downloadable from dashboard and backup to Google Sheets.
- Each provider has a separate export template.
- Fields must be mapped from canonical order fields.
- Export must validate required fields before download.

### Standard export columns

```csv
order_number,customer_name,customer_phone,customer_address,city,zone,area,product_description,item_quantity,item_weight,cod_amount,delivery_fee,special_instruction,payment_status
```

### Steadfast export columns

```csv
invoice,recipient_name,recipient_phone,recipient_address,cod_amount,note,item_description,total_lot,delivery_type
```

### Pathao export columns

```csv
merchant_order_id,sender_name,sender_phone,recipient_name,recipient_phone,recipient_address,recipient_city,recipient_zone,recipient_area,delivery_type,item_type,special_instruction,item_quantity,item_weight,amount_to_collect,item_description
```

### REDX export columns

```csv
merchant_order_id,recipient_name,recipient_phone,recipient_address,area,city,item_description,amount_to_collect,special_instruction
```

### Export validation

Before CSV is generated:

```text
[ ] customer name exists
[ ] phone normalized
[ ] address length adequate
[ ] COD amount is numeric
[ ] order status is ready_to_ship or confirmed
[ ] duplicate export warning checked
[ ] provider required city/zone/area mapping exists where needed
```

---

# 5. COD Return Risk Engine

## 5.1 Risk engine objective

The COD Return Risk Engine helps sellers decide whether to ship a parcel, request advance delivery fee, verify customer again, or block suspicious orders.

It must not claim certainty. It provides a risk signal, not a legal/fraud verdict.

## 5.2 Risk calculation input

```json
{
  "tenant_id": "uuid",
  "customer_id": "uuid",
  "phone_normalized": "01712345678",
  "order_id": "uuid",
  "order_total": 1850,
  "payment_method": "cod",
  "city": "Dhaka",
  "area": "Mirpur",
  "product_category": "skincare",
  "internal_history": {
    "orders": 4,
    "delivered": 3,
    "returned": 1,
    "cancelled": 0
  },
  "external_history": {
    "source": "pathao_success_rate|steadfast_status|none",
    "success_rate": 60,
    "checked_at": "2026-06-03T10:00:00+06:00"
  }
}
```

## 5.3 Internal tenant-only risk signals

Internal signals are safe within the same tenant:

| Signal | Risk effect |
|---|---:|
| Previous delivered orders with same phone | lowers risk |
| Previous return with same phone | raises risk |
| Multiple cancelled orders | raises risk |
| Missing address | raises risk |
| address too short | raises risk |
| COD amount unusually high | raises risk |
| repeat customer with reviews | lowers risk |
| verified advance payment | lowers risk |
| phone duplicate across multiple customer names in same tenant | raises risk |

## 5.4 Cross-tenant risk signals

Cross-tenant risk is sensitive.

### Allowed by default

- aggregate platform-level risk model without exposing other seller identities
- only if privacy policy permits
- only if customer phone is hashed or normalized securely
- only show risk bands, not other merchant history

### Not allowed by default

- showing “this customer returned parcel from another seller”
- exposing seller names, order details, products, addresses, or timestamps from another tenant
- cross-tenant blacklist without legal review
- irreversible “fraud” labeling

### Safer display language

Use:

```text
High COD risk signal
```

Do not use:

```text
Fraud customer
```

## 5.5 External courier success-rate signals

Pathao community docs reference a “user success rate using phone number” function in merchant API tooling. If official Pathao credentials expose this capability, Uddokta may use it as an external signal after verifying API terms.

Steadfast or other providers may expose status/history capabilities through merchant portals or APIs. Do not assume any external fraud/success endpoint unless the credentialed provider docs confirm it.

## 5.6 Risk scoring v1

```text
base_score = 0

Internal history:
+25 if previous returned count >= 2
+15 if previous returned count = 1
-25 if previous delivered count >= 3
-10 if previous delivered count = 1 or 2

Order quality:
+15 if delivery address length < 20 characters
+10 if no area/city
+10 if phone used by multiple customer profiles in same tenant
+15 if COD amount > tenant average order value * 2
-10 if partial advance payment verified
-20 if full payment verified

External signal:
+25 if external success rate < 50
+10 if external success rate >= 50 and < 70
-15 if external success rate >= 85

Manual staff input:
+20 if staff flagged suspicious
-10 if staff confirmed known customer
```

### Risk level thresholds

```text
0 or below        → low
1 to 24           → medium
25 to 49          → high
50+               → blocked/review required
unknown/no signal → unknown
```

## 5.7 Risk profile output

```json
{
  "risk_level": "high",
  "risk_score": 42,
  "reasons": [
    "Previous returned order found",
    "Address is incomplete",
    "COD amount is higher than normal"
  ],
  "recommended_action": "Request delivery charge advance before shipment",
  "display_copy_bn": "এই order-এ COD return risk বেশি। parcel পাঠানোর আগে delivery charge advance নিতে পারেন।"
}
```

## 5.8 Visual alert flags on order board

### UI alert rules

| Risk level | Order card indicator | Tap behavior |
|---|---|---|
| `unknown` | gray dot: “Risk unknown” | open risk detail |
| `low` | no warning or green microtext | no interruption |
| `medium` | amber dot: “Check before send” | show reason |
| `high` | amber card strip: “COD risk high” | require confirmation |
| `blocked` | red lock: “Owner approval needed” | disable courier submit until owner override |

### Owner approval for high-risk COD

If risk is `blocked`:

```text
[ ] Owner reviewed customer
[ ] Customer reconfirmed order
[ ] Advance delivery charge requested OR owner waived
[ ] Reason for override entered
```

Store override in `audit_events`.

---

# 6. Automated Customer Alert Flows

## 6.1 Notification engine principles

Outbound customer alerts must pass through:

1. tenant bot-pause check
2. opt-out check
3. order-level notification setting
4. WhatsApp 24-hour/session or template rule check
5. rate limit check
6. message log insert
7. provider send
8. delivery-status update

## 6.2 Canonical outbound message object

```json
{
  "tenant_id": "uuid",
  "customer_id": "uuid",
  "order_id": "uuid",
  "conversation_id": "uuid",
  "channel": "whatsapp",
  "provider": "meta_whatsapp|evolution_demo",
  "message_type": "service|utility|template|manual",
  "template_key": "order_confirmed_receipt_bn",
  "language": "bn",
  "body": "আপনার order confirmed...",
  "variables": {
    "order_number": "UDD-1048",
    "receipt_url": "https://app.uddokta.com/public/receipt/UDD-1048"
  },
  "idempotency_key": "tenant:order:trigger:template"
}
```

## 6.3 Trigger A: Order Confirmed → Digital Receipt Link

### Trigger condition

```text
orders.status changes to confirmed
AND receipt_url exists
AND notification not already sent for order_confirmed_receipt
```

### n8n loop

```text
Supabase trigger / n8n schedule / webhook
→ fetch order + customer + tenant channel
→ check bot pause
→ check opt-out
→ compose receipt message
→ insert outbound_messages queued
→ send via Meta/Evolution
→ update outbound message status
→ audit event
```

### Message template

```text
আপনার order confirmed ✅

Order: {{order_number}}
Total: ৳{{total}}
Receipt: {{receipt_url}}

Delivery update আমরা এখানেই জানাব।
```

## 6.4 Trigger B: Courier Picked Up → Live Tracking URL

### Trigger condition

```text
courier_shipments.status changes to picked_up OR in_transit
AND tracking_url or tracking_code exists
AND tracking message not already sent
```

### Message template

```text
আপনার parcel courier pickup করেছে 🚚

Order: {{order_number}}
Tracking: {{tracking_url_or_code}}

Delivery সময় courier update অনুযায়ী পরিবর্তন হতে পারে।
```

## 6.5 Trigger C: Parcel Returned → Failed Delivery Follow-up

### Trigger condition

```text
courier_shipments.status changes to returned
OR courier_events.normalized_status = returned
```

### Internal action

- mark order status `returned`
- create followup type `failed_delivery`
- flag customer risk profile
- notify owner/staff
- queue customer follow-up only if policy allows

### Customer message

```text
আপনার order delivery complete হয়নি / return হয়েছে।

Order: {{order_number}}

আপনি কি আবার delivery নিতে চান?
1. হ্যাঁ, আবার পাঠান
2. Address change করব
3. Order cancel
```

### Staff alert

```text
Returned parcel alert

Order: {{order_number}}
Customer: {{customer_name}} / {{phone}}
Courier: {{provider}}
Reason/status: {{provider_status_raw}}

Action needed: call customer or request advance delivery charge before resend.
```

## 6.6 Trigger D: Payment Verified → Order Processing

```text
payment_attempt.status changes to verified
AND order.payment_status changes to paid/advance_paid
```

Message:

```text
Payment verified ✅
Order: {{order_number}}

আমরা এখন order process করছি।
```

## 6.7 Trigger E: Payment Failed Match → Ask Again

```text
payment_attempt.status changes to failed_match
```

Message:

```text
Payment proof match হচ্ছে না।
দয়া করে correct TrxID বা clear screenshot আবার পাঠান।

Order: {{order_number}}
Expected amount: ৳{{expected_amount}}
```

## 6.8 Trigger F: Delivered → Review Request

```text
order.status changes to delivered
AND review request not sent
```

Message:

```text
আপনার order delivered হয়েছে ✅

আপনার experience কেমন ছিল?
Review দিন: {{review_url}}

আপনার review আমাদের business trust বাড়াতে সাহায্য করে।
```

---

# 7. Integration Error Resilience & Idempotency

## 7.1 Idempotency principle

Every external event or action must be safe to process multiple times without duplicate orders, duplicate messages, duplicate payments, or duplicate courier events.

## 7.2 Idempotency keys

### Payment proof

```text
tenant_id + method + trx_id
```

If no `trx_id` exists:

```text
tenant_id + order_id + sha256(raw_text + screenshot_hash + claimed_amount)
```

### Payment gateway

```text
tenant_id + provider + gateway_transaction_id
tenant_id + provider + payment_id
tenant_id + provider + merchant_invoice_number
```

### Courier create shipment

```text
tenant_id + provider + order_id + operation:create_shipment
```

### Courier webhook

```text
source:courier + provider + provider_event_id
```

Fallback:

```text
source:courier + provider + provider_consignment_id + provider_status_raw + event_time
```

### Meta inbound message

```text
source:meta + provider_message_id
```

### Evolution inbound message

```text
source:evolution + instance_name + remote_jid + message_id
```

## 7.3 Duplicate event prevention flow

```text
Webhook received
→ Compute idempotency_key
→ Try insert into webhook_events
→ Unique constraint succeeds:
     process event
→ Unique constraint fails:
     mark/log duplicate
     return HTTP 200
```

Important: duplicates should return **HTTP 200** to the provider after being safely ignored, otherwise providers may retry again.

## 7.4 Retry architecture

### Retryable errors

```text
HTTP 408 timeout
HTTP 429 rate limit
HTTP 500 provider error
HTTP 502/503/504 gateway/server unavailable
network timeout
DNS temporary failure
connection reset
```

### Non-retryable errors

```text
HTTP 400 invalid payload
HTTP 401 invalid credentials
HTTP 403 forbidden
HTTP 404 consignment not found
validation error
missing required customer phone
missing address
invalid COD amount
duplicate consignment already exists
```

### Retry schedule

```text
attempt 1: immediate
attempt 2: after 2 minutes
attempt 3: after 10 minutes
attempt 4: after 30 minutes
attempt 5: after 2 hours
then: manual review
```

### Retry queue table

Use existing `message_retry_queue` for messages, but create a separate `integration_retry_queue` for courier/MFS.

```sql
create table public.integration_retry_queue (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid not null references public.tenants(id) on delete cascade,
  integration_request_id uuid references public.integration_requests(id) on delete cascade,
  provider provider_type not null,
  operation text not null,
  payload jsonb not null default '{}',
  attempt_count int not null default 0,
  max_attempts int not null default 5,
  next_retry_at timestamptz not null,
  last_error text,
  status text not null default 'queued', -- queued | processing | completed | dead_letter
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create index integration_retry_queue_due_idx
on public.integration_retry_queue (next_retry_at)
where status = 'queued';
```

## 7.5 n8n retry processor

```text
Schedule trigger / every few minutes
→ Query integration_retry_queue where next_retry_at <= now and status = queued
→ Lock job by setting status = processing
→ Call provider adapter
→ If success:
     mark integration_request success
     mark queue completed
→ If retryable fail:
     increment attempt_count
     compute next_retry_at
     status = queued
→ If non-retryable or max attempts:
     status = dead_letter
     alert operations
```

## 7.6 Dead-letter queue

Dead-letter events require human review.

### Required fields in admin UI

- provider
- operation
- order number
- customer phone
- payload preview
- error message
- attempt count
- last response
- retry button
- mark resolved
- escalate to provider

## 7.7 Provider health checks

Each provider account should have a health check.

| Provider | Health check |
|---|---|
| bKash gateway | grant/refresh token test |
| SSLCommerz | validation endpoint / credential check |
| Steadfast | authenticated status/profile endpoint if available |
| Pathao | token expiry or cities/stores endpoint |
| REDX | credential/test endpoint if available |
| Meta WhatsApp | phone number status / test send where allowed |
| Evolution demo | instance state |

Do not run health checks that create real payments or shipments.

---

# 8. n8n Workflow Catalog for Document 10

## 8.1 Payment proof intake workflow

```text
Workflow: mfs-payment-proof-intake

Trigger:
- inbound message has media/screenshot
- inbound text contains trx/trxid/reference/payment keyword
- staff manually uploads proof

Steps:
1. Normalize tenant/order/customer context.
2. Save raw message/media.
3. Create payment_attempt if not exists.
4. Extract OCR/text if possible.
5. Parse TrxID/amount/reference.
6. Check duplicate TrxID.
7. Set status = needs_verification.
8. Notify seller/staff.
```

## 8.2 Manual verification workflow

```text
Workflow: mfs-manual-verification-action

Trigger:
- staff clicks Mark Verified / Failed Match / Reject

Steps:
1. Validate user role and tenant access.
2. Fetch payment_attempt.
3. Check duplicate constraints.
4. Update payment_attempt.status.
5. Update order.payment_status.
6. Insert payment_verification_event.
7. Queue customer message if needed.
8. Audit event.
```

## 8.3 Gateway payment callback workflow

```text
Workflow: gateway-payment-callback

Trigger:
- /api/webhooks/payments/:provider

Steps:
1. Verify provider signature where available.
2. Compute idempotency key.
3. Save webhook_events raw payload.
4. Normalize payment event.
5. Match payment_attempt/order.
6. Query provider if callback alone is insufficient.
7. Update status.
8. Queue confirmation/failure message.
9. Audit.
```

## 8.4 Courier create shipment workflow

```text
Workflow: courier-create-shipment

Trigger:
- seller clicks Create Shipment
- order.status = ready_to_ship
- bulk selection submitted

Steps:
1. Fetch order/customer/products.
2. Validate required courier fields.
3. Calculate COD amount.
4. Calculate risk score.
5. If high risk, require owner override.
6. Map to provider payload.
7. Insert integration_request.
8. Call provider API or create CSV export.
9. Save courier_shipment.
10. Queue customer tracking message if accepted.
11. Audit.
```

## 8.5 Courier webhook workflow

```text
Workflow: courier-webhook-normalizer

Trigger:
- provider webhook or polling result

Steps:
1. Verify webhook token/signature if available.
2. Compute idempotency key.
3. Insert webhook_events.
4. Normalize provider status.
5. Match shipment.
6. Insert courier_event.
7. Update shipment/order status.
8. Trigger customer alert.
9. Trigger staff alert if returned/failed.
10. Audit.
```

## 8.6 Courier polling workflow

Use polling only where webhooks are unavailable or unreliable.

```text
Workflow: courier-status-poller

Trigger:
- scheduled

Steps:
1. Fetch active shipments not final.
2. For each provider account, group requests.
3. Query provider status endpoint where available.
4. Normalize status.
5. Process as courier event.
6. Respect provider rate limits.
```

## 8.7 COD risk calculation workflow

```text
Workflow: cod-risk-calculation

Trigger:
- order created
- customer phone changed
- order marked ready_to_ship
- shipment create attempted

Steps:
1. Normalize phone.
2. Fetch internal customer/order history.
3. Fetch external success-rate if provider supports it and tenant has credentials.
4. Calculate risk score.
5. Save customer_risk_profile.
6. Add UI flag to order.
7. If blocked/high, require owner review before courier submit.
```

---

# 9. API Contracts for AI Coding Agents

## 9.1 Create payment proof

`POST /api/payments/proof`

```json
{
  "order_id": "uuid",
  "method": "bkash_pra",
  "claimed_amount": 1850,
  "trx_id": "ABC123",
  "screenshot_url": "https://...",
  "raw_text": "optional"
}
```

Response:

```json
{
  "payment_attempt_id": "uuid",
  "status": "needs_verification",
  "duplicate_warning": false
}
```

## 9.2 Verify payment

`POST /api/payments/:id/verify`

```json
{
  "action": "verified|failed_match|rejected",
  "verified_amount": 1850,
  "notes": "Matched in bKash PRA app",
  "failed_reason": null
}
```

Response:

```json
{
  "payment_attempt_id": "uuid",
  "order_id": "uuid",
  "new_status": "verified",
  "order_payment_status": "paid"
}
```

## 9.3 Create shipment

`POST /api/courier/shipments`

```json
{
  "order_id": "uuid",
  "provider": "steadfast",
  "cod_amount": 1850,
  "special_instruction": "Call before delivery",
  "force_owner_override": false
}
```

Response:

```json
{
  "shipment_id": "uuid",
  "status": "api_submitted",
  "provider_consignment_id": "1424107",
  "provider_tracking_code": "15BAEB8A",
  "tracking_url": "https://..."
}
```

## 9.4 Export courier CSV

`POST /api/courier/export`

```json
{
  "provider": "pathao",
  "order_ids": ["uuid", "uuid"],
  "format": "csv"
}
```

Response:

```json
{
  "download_url": "https://...",
  "rows": 25,
  "validation_errors": []
}
```

## 9.5 Calculate COD risk

`POST /api/risk/cod`

```json
{
  "order_id": "uuid",
  "force_refresh_external": false
}
```

Response:

```json
{
  "risk_level": "high",
  "risk_score": 42,
  "reasons": [
    "Previous returned order found",
    "Address incomplete"
  ],
  "recommended_action": "Request advance delivery charge"
}
```

---

# 10. Security, Privacy, and Compliance Requirements

## 10.1 Credential storage

- Provider credentials must live in Supabase Vault, encrypted DB fields, or environment secrets.
- Browser must never receive provider credentials.
- n8n must use credential references, not hardcoded tokens inside nodes.
- Rotate credentials when staff changes or provider warns of compromise.

## 10.2 Webhook verification

- Meta webhooks: verify challenge and signature.
- bKash/SSLCommerz: verify provider signature/IPN validation where available.
- Courier webhooks: verify bearer token, secret header, IP allowlist, or provider-supplied signature where available.
- Evolution demo: verify API key and instance mapping.

If webhook cannot be verified, store raw payload with `signature_valid = false`, do not update orders automatically, and alert admin.

## 10.3 Sensitive data rules

Sensitive fields:

- phone numbers
- payment screenshots
- TrxID
- courier address
- buyer order history
- provider credentials
- webhook raw payloads

Controls:

- tenant RLS
- role-based access
- signed media URLs
- audit logs
- retention rules
- export controls
- no public indexing of receipt/trust pages with sensitive data

## 10.4 Customer-facing privacy copy

For receipt and trust pages:

```text
এই order link শুধুমাত্র order confirmation এবং support-এর জন্য।
আপনার phone/address public করা হবে না।
```

---

# 11. Testing Requirements

## 11.1 Payment tests

```text
[ ] Create payment_attempt with screenshot
[ ] Duplicate TrxID blocked
[ ] Staff can mark verified
[ ] failed_match triggers customer message
[ ] verified updates order payment status
[ ] OCR parser never auto-verifies
[ ] Gateway webhook duplicate is ignored
[ ] Cross-tenant TrxID does not leak
```

## 11.2 Courier tests

```text
[ ] Create shipment payload maps correctly for Steadfast
[ ] Create shipment payload maps correctly for Pathao
[ ] Required field validation blocks incomplete address
[ ] COD amount is recalculated server-side
[ ] Duplicate create shipment does not create second consignment
[ ] Provider 500 moves request to retry queue
[ ] Non-retryable validation error goes to manual review
[ ] Courier webhook duplicate is ignored
[ ] Delivered event updates order
[ ] Returned event updates order and creates follow-up
```

## 11.3 Risk engine tests

```text
[ ] Unknown phone returns unknown risk
[ ] Repeat delivered customer lowers score
[ ] Repeat returned customer raises score
[ ] Incomplete address raises score
[ ] Verified payment lowers score
[ ] High risk blocks courier submission without owner override
[ ] Cross-tenant data is not exposed in risk reasons
```

## 11.4 Customer alert tests

```text
[ ] Order confirmed sends receipt once
[ ] Picked up sends tracking once
[ ] Returned creates failed delivery follow-up
[ ] Opt-out blocks outbound
[ ] Tenant bot pause blocks outbound
[ ] Retry queue handles 429/5xx
[ ] Manual message override works
```

---

# 12. Required Fixtures Before Coding Real Integrations

AI coding agents must create fixtures first.

## 12.1 Payment fixtures

```text
fixtures/payments/bkash_pra_sms_received.json
fixtures/payments/bkash_pra_statement_row.json
fixtures/payments/nagad_manual_sms_received.json
fixtures/payments/payment_duplicate_trx.json
fixtures/payments/bkash_gateway_ipn_success.json
fixtures/payments/bkash_gateway_query_success.json
```

## 12.2 Courier fixtures

```text
fixtures/courier/steadfast_create_success.json
fixtures/courier/steadfast_create_error_validation.json
fixtures/courier/steadfast_status_delivered.json
fixtures/courier/pathao_create_success.json
fixtures/courier/pathao_success_rate_low.json
fixtures/courier/pathao_status_returned.json
fixtures/courier/redx_manual_export_row.json
```

## 12.3 Normalized fixtures

```text
fixtures/normalized/payment_event_verified.json
fixtures/normalized/courier_event_delivered.json
fixtures/normalized/courier_event_returned.json
fixtures/normalized/cod_risk_high.json
```

---

# 13. Agent Instructions: Local Integrations Build Prompt

Paste this into Claude Code / Cursor / Windsurf when implementing Document 10.

```text
You are building Uddokta Document 10 integrations.

Goal:
Implement the localized MFS payment verification and courier integration layer for Uddokta without hallucinating provider payloads.

Stack:
- Next.js API routes
- Supabase/PostgreSQL
- n8n workflows
- TypeScript
- Zod validation
- Meta WhatsApp Cloud API / Evolution demo
- provider adapters for bKash PRA/manual, Nagad/manual, Steadfast, Pathao, REDX/manual export

Rules:
1. Supabase is the source of truth.
2. n8n orchestrates but does not own business state.
3. Every provider request has an idempotency key.
4. Every inbound webhook is stored raw before processing.
5. Every external payload is normalized into Uddokta canonical schema.
6. Do not hardcode undocumented provider payloads.
7. Use provider-specific adapter files and fixtures.
8. Browser must never receive provider credentials.
9. Every order/customer/payment/courier record must be tenant-scoped.
10. Add tests for duplicate webhook prevention, duplicate TrxID prevention, and cross-tenant isolation.

Build:
- provider_accounts
- payment_attempts
- payment_verification_events
- courier_shipments
- courier_events
- webhook_events
- integration_requests
- integration_retry_queue
- customer_risk_profiles
- payment proof APIs
- courier shipment APIs
- export CSV API
- COD risk API
- n8n workflow JSON stubs
- fixture files
- README for local testing

Do not implement real provider production calls until official credentials and current provider docs are available. Implement adapters with interfaces, mocks, fixtures, and config-driven field mapping first.
```

---

# 14. Provider Documentation Checklist

Before enabling a provider in production, collect and store:

```text
[ ] Official provider docs PDF/link
[ ] API base URL
[ ] Sandbox URL if available
[ ] Production URL
[ ] Auth mechanism
[ ] Required headers
[ ] Rate limits
[ ] Create order/payment payload
[ ] Status/query payload
[ ] Webhook payload
[ ] Signature verification method
[ ] Retry policy recommended by provider
[ ] Error code list
[ ] Merchant account ID/store ID
[ ] Contact person at provider
[ ] Credential rotation procedure
```

If this checklist is incomplete, provider remains in `manual` or `csv` mode.

---

# 15. Final Integration Position

Uddokta’s local integration strategy is:

```text
Manual proof first
→ Semi-automated verification queue
→ CSV/export courier fallback
→ Provider adapter abstraction
→ Official API only after credentialed docs
→ Full gateway/courier automation only after reliability is proven
```

This prevents the two major failures common in Bangladesh SME integrations:

1. pretending unofficial/local APIs are more stable than they are;
2. letting automation destroy trust when manual verification would have been safer.

Uddokta wins by making local chaos visible, structured, auditable, and recoverable — not by pretending every provider integration is clean from day one.

---

# 16. Source Notes

- bKash Developer Product Overview: https://developer.bka.sh/docs/product-overview
- bKash Developer Grant Token: https://developer.bka.sh/docs/grant-token-3
- bKash Developer Tokenized Create Payment: https://developer.bka.sh/docs/create-payment-1
- bKash Developer IPN/Webhooks: https://developer.bka.sh/docs/webhooks
- bKash Personal Retail Account: https://www.bkash.com/en/page/personal-retail-account
- Pathao Courier: https://pathao.com/courier/
- Pathao Courier community package: https://github.com/enuenan/pathao-courier
- Steadfast Courier: https://steadfast.com.bd/
- Steadfast community package: https://packagist.org/packages/nayemuf/steadfast-courier
- n8n Webhook Node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/
- n8n Error Handling: https://docs.n8n.io/flow-logic/error-handling/
- n8n Queue Mode: https://docs.n8n.io/hosting/scaling/queue-mode/
