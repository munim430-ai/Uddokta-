# Uddokta — Setup & Operations SOP
### Document 8 of 8 | Standard Operating Procedure | Version 1.0

> **Who this document is for:** Uddokta Operations Team, Technical Support Lead, and any future team member who onboards, supports, or offboards a client. This is the operational bible. When in doubt, follow this document.

---

## Table of Contents

1. [The Client Onboarding & DFY Setup Protocol](#1-the-client-onboarding--dfy-setup-protocol)
2. [Customer Support & Triage Boundaries](#2-customer-support--triage-boundaries)
3. [Infrastructure Monitoring & Maintenance](#3-infrastructure-monitoring--maintenance)
4. [Compliance & WhatsApp Risk Mitigation](#4-compliance--whatsapp-risk-mitigation)
5. [Offboarding & Data Export Protocol](#5-offboarding--data-export-protocol)

---

## 1. The Client Onboarding & DFY Setup Protocol

> **Operational philosophy:** The DFY setup is the single most margin-destructive part of the business if it is not strictly bounded. Every step in this section exists to eliminate decision loops, reduce back-and-forth, and ensure a client is fully operational after **one structured asset collection session** and **one handover call.** No step is optional. Deviating from this protocol without written approval adds unbudgeted operator hours.

### 1.1 Sales-to-Operations Handover (Twenty CRM)

When a client moves to the **"Paid/Onboarding"** stage in the Twenty CRM pipeline, the Sales Lead must complete the handover card before Operations touches anything.

**Sales Lead's Handover Checklist (done in Twenty CRM, on the client's Opportunity record):**

Mark each field as confirmed before assigning to Operations:

```
[ ] Client's WhatsApp number (the number that will receive the dashboard login OTP)
[ ] Client's Facebook Page URL (exact, not a short link)
[ ] Business category confirmed: Cosmetics | Gadgets | Clothing | Other
[ ] Plan confirmed: Starter | Growth | DFY Premium
[ ] Setup fee payment screenshot attached to Twenty record
[ ] Client language preference confirmed: Bangla | English | Banglish
[ ] Client's current courier(s) confirmed: Steadfast | Pathao | REDX | Manual
[ ] Client's payment method(s) confirmed: COD | bKash | Nagad | Bank Transfer
[ ] Demo mode or Production WhatsApp API?
    → Demo (QR/Baileys): client was briefed on instability risk. Signed off? [ ]
    → Production (Meta Cloud API): WABA ID and Phone Number ID collected? [ ]
[ ] Estimated daily message volume (from audit): ______
[ ] "Biggest Pain" from audit (verbatim from Typebot): ______
[ ] Sales notes: any custom promises made (e.g., "we'll add a custom courier field")
```

**Operations Lead picks up the record ONLY after all boxes are checked.** An incomplete handover card is returned to Sales with a single Chatwoot note: *"Handover card incomplete — please fill fields [X, Y] before reassigning."*

Update the Twenty CRM Opportunity stage to **"Setup Started"** the moment Operations begins work.

---

### 1.2 The Typebot Asset Collection Flow

Before any manual work begins, the client must complete the **Uddokta Setup Intake Bot**. This bot runs on the self-hosted Typebot instance and passively collects all assets needed for provisioning. The Operations Lead sends the bot link via WhatsApp — the client completes it at their own pace.

**Sending the intake link:**

```
WhatsApp message to client (Banglish):

"[Client Name] ভাই/আপু, আপনার Uddokta account setup শুরু হয়ে গেছে! 🎉

নিচের link-এ click করে ছোট একটা form fill করুন। এটাতে আপনার products, 
delivery charge, আর return policy নেওয়া হবে — যা আমরা সরাসরি আপনার 
dashboard-এ add করে দেব।

👉 [TYPEBOT_INTAKE_LINK]

Form complete করলেই আমরা আপনার account তৈরি শুরু করব।
কোনো সমস্যা হলে এখানেই জানান।"
```

**Typebot Intake Bot — Required Flow Nodes:**

The intake bot must collect the following. No asset collection is done by WhatsApp chat if the bot can collect it — the bot is the boundary.

```
Section 1 — Business Identity
  [ ] Business name (as it should appear on receipts and trust profile)
  [ ] Business logo (file upload to R2-backed Typebot storage)
  [ ] Business support WhatsApp number (may differ from owner's number)
  [ ] Business address / area (for receipt page)

Section 2 — Products (up to 10 for DFY onboarding)
  Repeat block up to 10 times:
  [ ] Product name
  [ ] Selling price (BDT)
  [ ] Variants (e.g., "Color: Red, Blue, Green / Size: S, M, L, XL")
  [ ] Stock status: Available / Limited / Pre-order
  [ ] Delivery charge (inside Dhaka / outside Dhaka)
  [ ] Product image (file upload)
  [ ] Quick reply template: "Price question" (pre-written or free text)

Section 3 — Policies (free text, client writes in their own words)
  [ ] Delivery policy (how many days, which couriers)
  [ ] Return/refund policy (e.g., "7 days return, product must be unused")
  [ ] Payment methods accepted
  [ ] Authenticity/warranty note (important for cosmetics, gadgets)
  [ ] Support hours ("আমরা সকাল ১০টা থেকে রাত ১০টা পর্যন্ত available")

Section 4 — Staff (optional)
  [ ] Staff 1 name + WhatsApp number + role (Owner / Manager / Staff)
  [ ] Staff 2 name + WhatsApp number + role (if applicable)

Section 5 — Channel (admin fills after client submits)
  This section is not shown to the client. It is filled by the
  Operations Lead based on the agreed channel mode.
```

**Asset Collection Boundary Rule:** If a client sends product info via WhatsApp chat instead of completing the bot, the Operations Lead replies:

```
"[Name] ভাই, সুবিধার জন্য form-এর মাধ্যমে দিলে আমরা সরাসরি আপনার 
dashboard-এ enter করতে পারব, copy-paste mistake হবে না। 
একটু সময় নিয়ে এখানে দিন 👉 [LINK]"
```

Never manually transcribe from WhatsApp messages. One pass through the bot, one entry into the system.

**When the intake bot is complete**, n8n fires a webhook to the Operations Slack/WhatsApp channel with:
- Client name, business name, Twenty CRM opportunity ID
- Count of products submitted, policy sections completed
- Flag: `assets_complete: true/false`

If `assets_complete: false` (e.g., no logo uploaded, fewer than 3 products), send one follow-up message to the client asking for the missing items. After **two unanswered follow-up attempts**, escalate to Sales to re-engage.

---

### 1.3 Internal Admin Provisioning Checklist

This checklist is executed by the Operations Lead in exact sequence. Do not skip steps. Each step must be confirmed before the next begins.

**Step 1 — Supabase Workspace Provisioning**

```bash
# Call the provision-tenant Edge Function via curl:
curl -X POST https://app.uddokta.com/api/admin/provision-tenant \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {SUPABASE_SERVICE_ROLE_KEY}" \
  -d '{
    "tenant_slug": "{business-name-lowercase-hyphenated}",
    "tenant_name": "{Business Display Name}",
    "owner_name": "{Owner Full Name}",
    "owner_email": "{owner_email_or_generated_placeholder}",
    "owner_phone": "{+880XXXXXXXXXX}",
    "plan": "starter|growth|dfy_premium"
  }'
```

Expected response: `{ "tenant_id": "...", "user_id": "...", "status": "provisioned" }`

If the function returns an error, check:
1. Is the tenant_slug already taken? (`SELECT id FROM tenants WHERE slug = '...'`)
2. Is the phone number already registered? (Check Supabase Auth users)
3. Is the Chatwoot API responding? (`curl https://{VPS_DOMAIN}/chatwoot/auth/sign_in`)

Record the `tenant_id` in the Twenty CRM Opportunity's custom field: `uddokta_tenant_id`.

---

**Step 2 — Seed Products from Typebot Intake Data**

```bash
# Fetch the intake bot submission from the n8n-processed leads table:
# In Supabase Dashboard → Table Editor → leads
# Filter: phone = {client_phone} AND source = 'typebot_intake'
# Copy the raw_typebot_data JSONB field

# Run the product seed script (from the Admin console in the Next.js app):
# /admin/workspaces/{tenant_id}/setup → "Import Products" step
# Paste the product JSON from the Typebot submission
# Review each product row — confirm name, price, variants, image URL
# Click "Import All" — this bulk-inserts into public.products with correct tenant_id
```

**Manual product entry rule:** If a product has a variant structure that is too complex for bulk import (e.g., "Size S in Red, M in Blue, XL in Red and Black"), create it manually in the product form. Flag this in the Twenty CRM note: *"Manual variant entry needed for [product name] — non-standard variant matrix."*

After import, verify: `SELECT COUNT(*) FROM products WHERE tenant_id = '{tenant_id}'`
Expected: matches the number of products the client submitted.

---

**Step 3 — Configure Policies and Quick Reply Templates**

In the Admin setup wizard (`/admin/workspaces/{tenant_id}/setup`):

```
[ ] Delivery policy → paste from Typebot intake, lightly format
[ ] Return policy → paste from Typebot intake, lightly format
[ ] Payment methods → select from checkbox list
[ ] Authenticity note → paste if provided
[ ] Support hours → paste if provided

Auto-Reply Template Seeds (one per category, filled from policies above):
[ ] Price question template (uses {{product_name}}, {{price}}, {{variant}})
[ ] Delivery charge template (uses {{area}}, {{courier}}, {{days}})
[ ] Return policy template
[ ] Payment instruction template (bKash/Nagad number + instructions)
[ ] Out-of-stock template
[ ] Order confirmation template (sent after order creation)
[ ] Receipt delivery template (sent with receipt link)
```

Each template must be **previewed** in both Bangla and Banglish before saving. If the client submitted their policies in English only, translate the key phrases to Bangla — this is not optional. Sellers in Bangladesh respond better when their own templates reflect their natural voice.

---

**Step 4 — Connect WhatsApp Channel via Evolution API**

**For Demo Mode (QR/Baileys):**

```bash
# Create instance via Evolution API:
curl -X POST https://{VPS_DOMAIN}/evolution/instance/create \
  -H "apikey: {EVOLUTION_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "instanceName": "tenant-{tenant_slug}",
    "integration": "WHATSAPP-BAILEYS",
    "webhook": {
      "url": "https://{VPS_DOMAIN}/n8n/webhook/evolution-inbound",
      "by_events": true,
      "events": ["MESSAGES_UPSERT", "MESSAGES_UPDATE", "CONNECTION_UPDATE"]
    }
  }'

# Get the QR code:
curl https://{VPS_DOMAIN}/evolution/instance/connect/tenant-{slug}
# Returns a base64 QR image. Display it to the client via WhatsApp (screenshot).
# The client scans it from their WhatsApp on their phone.
```

After scanning, verify connection:
```bash
curl https://{VPS_DOMAIN}/evolution/instance/fetchInstances \
  -H "apikey: {EVOLUTION_API_KEY}" | jq '.[] | select(.instance.instanceName == "tenant-{slug}") | .instance.state'
# Expected: "open"
```

**Important:** Tell the client verbatim: *"এই connection-টা WhatsApp Web-এর মতো। আপনার phone বন্ধ থাকলে বা internet না থাকলে disconnect হয়ে যাবে। Production WhatsApp API-তে upgrade করলে এই সমস্যা থাকবে না।"*

**For Production Mode (Meta Cloud API):**
This requires the client's WABA ID, Phone Number ID, and a Permanent System User Token from Meta Business Suite. This is done in a **separate Meta Setup Call** with the client. Do not attempt Production API setup without the client present — Meta's two-factor flows require live decisions.

---

**Step 5 — Configure n8n Tenant Webhook Routing**

In n8n, activate the tenant-specific workflow set:

```
[ ] evolution-inbound workflow → verify it handles instanceName "tenant-{slug}"
[ ] outbound-message workflow → confirm tenant lookup returns correct chatwoot_account_id
[ ] daily-report-cron → confirm tenant is included in the active tenant list
[ ] sla-breach-check → confirm tenant conversations are being monitored
```

In Supabase, update the tenant record:
```sql
UPDATE public.tenants SET
  evolution_instance_name = 'tenant-{slug}',
  chatwoot_account_id = {CHATWOOT_ACCOUNT_ID_CREATED_IN_PROVISION},
  setup_checklist = jsonb_set(
    setup_checklist,
    '{channel_connected}',
    'true'
  )
WHERE id = '{tenant_id}';
```

---

**Step 6 — Create Test Customer and Test Order**

In the Admin setup wizard:
1. Create a test customer: Name = "Test Buyer", Phone = any valid number
2. Create a test order with 1 product, COD, pending status
3. Verify order number is generated in format `UDD-XXXX`
4. Generate the receipt link: `/public/receipt/{receipt_token}`
5. Open the receipt link and verify it shows: business name, product, policy

This test receipt link is what you use during the Aha! Moment walkthrough.

Update the Twenty CRM setup checklist:
```
setup_checklist → test_order: complete
setup_checklist → receipt_link_verified: complete
```

---

**Step 7 — Update Twenty CRM and Notify Sales**

```
Update Twenty CRM Opportunity:
  Stage: "Active Client"
  Custom fields:
    uddokta_tenant_id: {tenant_id}
    dashboard_url: https://app.uddokta.com
    first_login_sent: false (will flip when login OTP is sent to client)
    setup_complete_date: {today}
    channel_mode: demo | production

Post a Chatwoot internal note on the client's conversation:
  "Setup complete. Tenant ID: {tenant_id}. Products: {count}.
   Templates: {count}. Channel: {demo/production}.
   Awaiting Aha! Moment handover call."

Notify Sales Lead via WhatsApp/Slack:
  "[Business Name] setup complete. Handover call can be scheduled."
```

---

### 1.4 The Aha! Moment Handover — Script & 5-Minute Mobile Walkthrough

> **The single most important 5 minutes in the client relationship.** The Aha! Moment is the instant the seller realizes: *"আমি এইটা দিয়েই আমার পুরো business চালাতে পারব।"* Everything in the setup was preparation for this moment. Do not rush it. Do not do it over text — it must be a call (WhatsApp call or video call).

**Prerequisites before the call:**
- Dashboard is live with seeded products, policies, and test order
- Receipt link opens correctly on mobile
- Client's WhatsApp is connected (QR scanned or Production API live)
- Operations Lead is on a phone (not desktop) to mirror the client's experience

---

**The 5-Minute Walkthrough Script:**

*Call starts. Client has opened https://app.uddokta.com on their phone.*

**[0:00 — Login & First Look]**

```
Ops Lead: "[Name] ভাই, আপনার OTP গেছে। Enter করুন।"
[Client logs in]
"দেখুন এটাই আপনার business-এর dashboard। 
 এখানে আপনার সব orders, customers, আর inbox একসাথে আছে।
 মোবাইলেই সব হবে — laptop লাগবে না।"
```

**[0:45 — The Order Board]**

```
Ops Lead: "Orders-এ click করুন।"
[Client sees the test order]
"এটা একটা test order আমরা তৈরি করেছি। 
 Real-এ যখন customer message করবে, এখানে একটা order create হবে।
 Status change করতে card-এর উপর tap করুন।"
[Demonstrate status change: New → Confirmed]
"দেখলেন? এক tap। কোনো screenshot, কোনো note নেই।"
```

**[1:30 — The Receipt Link — This is the Aha! Moment]**

```
Ops Lead: "এবার সবচেয়ে important জিনিসটা দেখান।
           এই order-এর উপর tap করুন। নিচে 'Receipt পাঠান' দেখছেন?"
[Client taps]
"এই link টা customer-কে পাঠান। Open করুন।"
[Client opens /public/receipt link]
"দেখুন — আপনার logo, order number, product, দাম, delivery policy — 
 সব একটা professional page-এ। Customer এটা দেখলে trust করবে।
 কোনো screenshot নয়, কোনো PDF নয়।"

[PAUSE. Let the client react.]
```

Most clients say *"এটা তো অনেক সুন্দর"* or *"customer এটা পাঠালে বিশ্বাস করবে।"* That is the Aha! Moment. Do not talk over it.

**[2:30 — Quick Replies in Inbox]**

```
Ops Lead: "এখন Inbox-এ যান।"
[Client opens Inbox]
"এখানে আপনার সব WhatsApp message আসবে।
 কোনো customer 'price কত?' জিজ্ঞেস করলে —
 নিচে Quick Reply button tap করুন।"
[Demonstrate selecting a price template]
"দেখুন — product-এর নাম, দাম, delivery charge সব automatic।
 Edit করে send। Staff-কে একটু train করলেই হবে।"
```

**[3:30 — Products]**

```
Ops Lead: "Products-এ যান — আপনার products আমরা add করে রেখেছি।"
[Client sees their products]
"Edit করতে পারবেন, নতুন add করতে পারবেন — নিজেই।
 কোনো আমাদের কাছে আসতে হবে না।"
```

**[4:15 — The Handover Statement]**

```
Ops Lead: "এই পাঁচটা tab — Inbox, Orders, Customers, Products, Reports —
           এটাই আপনার পুরো business।
           
           আপনার staff-কে শুধু দুইটা জিনিস শেখাতে হবে:
           ১. Message এলে Quick Reply দিন।
           ২. Order হলে Create করুন আর Receipt পাঠান।
           
           কোনো সমস্যা হলে এই WhatsApp-এ message করুন। [Support Number]
           আমরা আছি।"
```

**[4:45 — First Real Action Together]**

```
Ops Lead: "একটা real order create করুন এখন — আমি আছি।"
[Guide the client through creating their first real order, live, together]
"Perfect। এটাই আপনার first Uddokta order।
 এখন থেকে সব এখানেই।"
```

**Post-call action (do immediately after the call ends):**

1. Send a WhatsApp message with the support number, help link, and the test receipt link for reference.
2. Update Twenty CRM: `first_login_sent: true`, stage confirmed as "Active Client."
3. Log in Chatwoot internal note: "Aha! Moment call complete. Client created first order. Sentiment: [positive/neutral/confused]."
4. If client seemed confused: schedule a 10-minute follow-up check-in in 48 hours. Flag in Twenty CRM as `needs_followup_training: true`.

---

### 1.5 DFY Scope Boundaries — What Is and Is Not Included

This section exists to prevent scope creep. Read it before every client interaction.

| Included in DFY Setup | NOT Included — Quoted Separately |
|---|---|
| Up to 10 products seeded from intake form | More than 10 products (charge per 5 additional: [price]) |
| Delivery, return, payment policy templates | Custom legal terms or multi-page policy documents |
| 8 standard auto-reply templates | Custom bot conversation flows with conditional branches |
| 1 WhatsApp channel (demo or production) | Multiple phone numbers or multiple Facebook pages |
| 3 staff users added | More than 3 staff users |
| 1 Aha! Moment walkthrough call | Additional training calls per team member |
| Receipt page configured | Custom receipt HTML design |
| Trust profile page configured | Review campaigns or Google review integration |
| 30 days post-setup support (ticket/WhatsApp) | Ongoing product catalog maintenance |

When a client requests something outside this scope, the response is:

```
"[Name] ভাই, এটা আমাদের standard setup-এর বাইরে একটু।
 আমরা করতে পারব, কিন্তু এটার জন্য আলাদা একটা ছোট charge হবে।
 আমি details পাঠাচ্ছি — দেখুন ঠিক আছে কিনা।"
```

Never say "no" directly. Always say "separately chargeable."

---

## 2. Customer Support & Triage Boundaries

### 2.1 Support Tier Definitions

Support tiers are determined by the client's plan, not by the severity of their mood. A client who is upset but on a Starter plan does not get DFY tier support. Be firm and kind.

| Support Tier | Plans | Channels | Response Commitment | Who Handles |
|---|---|---|---|---|
| **Tier 1 — Standard** | Starter | Chatwoot ticket only (web widget or email) | Next business day | Junior Support via Chatwoot |
| **Tier 2 — Priority** | Growth | Chatwoot ticket + WhatsApp (business hours: 10am–10pm BDT) | Within 4 hours | Senior Support via Chatwoot |
| **Tier 3 — Dedicated** | DFY Premium | WhatsApp direct + Chatwoot | Within 1 hour | Assigned Operations Lead |
| **Internal/Admin** | Uddokta team | Internal Chatwoot inbox | Immediate | Tech Lead |

**Business hours:** 10:00 AM – 10:00 PM Bangladesh Standard Time (BST = UTC+6), 7 days a week. No SLA applies outside these hours unless the issue is Severity 1 (defined below).

---

### 2.2 Using Chatwoot as the Internal Support Desk

Uddokta uses its own self-hosted Chatwoot instance not just for clients' inboxes, but as **our internal support desk.** This means all incoming support requests from clients (across WhatsApp and web widget) flow into the Uddokta team's Chatwoot instance.

**Inbox configuration in Chatwoot (admin setup, one-time):**

```
Inbox 1: "Uddokta Support — Web Widget" (embedded on app.uddokta.com)
  → Source: Web widget using Chatwoot's HTML embed
  → Assigned team: Support
  → Auto-assignment: round-robin among support agents

Inbox 2: "Uddokta Support — WhatsApp" (Evolution API instance: uddokta-support)
  → Source: Evolution API connected to Uddokta's own support WhatsApp number
  → Assigned team: Support
  → Auto-assignment: round-robin

Inbox 3: "Internal Engineering" (for tech team escalations)
  → Source: n8n webhook (auto-creates tickets for Severity 1 events)
  → Assigned team: Tech Lead only
  → No auto-assignment — manual triage
```

**Label taxonomy in Chatwoot (create these before going live):**

```
tier-1-standard
tier-2-priority
tier-3-dedicated
sev-1-critical
sev-2-moderate
sev-3-minor
billing
feature-request
whatsapp-issue
onboarding
bug-confirmed
waiting-client-response
resolved
```

Every ticket must have at minimum: one tier label + one severity label. Unlabelled tickets get auto-assigned a "needs-triage" label via n8n webhook after 30 minutes.

---

### 2.3 Severity Matrix

**Severity 1 — Service Down / Revenue Impact**
> Client cannot send or receive messages, or orders are not saving. Business is blocked.

| Trigger | Example |
|---|---|
| WhatsApp webhook stopped delivering | Client's inbox shows no new messages for >30 min during known active hours |
| Evolution API instance disconnected | QR session expired, client gets "session disconnected" in channel status |
| Supabase database unreachable | Orders not saving, "error" toast on every action |
| n8n workflow stuck | Daily reports not generating, retry queue growing >50 items |
| Vercel app returning 500/502 | Dashboard inaccessible |

**Response procedure for Severity 1:**
1. n8n's `sla-breach-check` and error monitors auto-create a Chatwoot ticket tagged `sev-1-critical` and notify the Tech Lead via WhatsApp within 5 minutes of detection.
2. Tech Lead acknowledges within 15 minutes.
3. Post a status update in the client's Chatwoot conversation within 20 minutes (even if unresolved): *"আমরা জানি সমস্যা হচ্ছে। আমরা এখনই কাজ করছি। Update আসছে।"*
4. Target resolution: within 2 hours.
5. Post-resolution: write a one-paragraph plain-language explanation of what happened and what was fixed. Store in `audit_logs`. Send to client.

---

**Severity 2 — Feature Not Working / Partial Impact**
> Client can use the dashboard but a specific feature is broken or data is missing.

| Trigger | Example |
|---|---|
| Daily report not generated for one tenant | Client asks "কাল কোনো report আসেনি কেন?" |
| Product quick reply templates not sending | Template sends but without variable substitution |
| Receipt link returning 404 | Order created but receipt link is broken |
| Staff user cannot log in | Invited staff member gets OTP but sees wrong workspace |

**Response procedure for Severity 2:**
1. Support agent triages within 4 hours.
2. If reproducible: create a bug report in the `Internal Engineering` Chatwoot inbox with steps to reproduce, tenant_id, and affected order/conversation ID.
3. Tech Lead assesses within 24 hours: is it a config issue (fixable by Operations) or a code bug (requires a Vercel deploy)?
4. Client is updated: *"আমরা দেখেছি। [X] ঘণ্টার মধ্যে fix হয়ে যাবে।"*

---

**Severity 3 — Cosmetic / Low-Impact / Training Issue**
> Client wants a change, has a question, or is confused about how to use a feature.

| Trigger | Example |
|---|---|
| Copy change in template | "Return policy-তে একটা line change করতে চাই" |
| Training question | "Staff কে নতুন order কীভাবে দেখাব?" |
| UI confusion | "Reports-এ কোনটা press করব?" |
| Feature request | "বিকাশের receipt attach করার option চাই" |

**Response procedure for Severity 3:**
1. Tier 1 clients: reply within 1 business day via Chatwoot.
2. Tier 2/3 clients: reply within 2 hours during business hours.
3. Template changes: the client must submit the new text in writing via Chatwoot. Operations applies the change in the Admin console. No back-and-forth over WhatsApp voice notes for copy changes.
4. Feature requests: acknowledge receipt, log in Twenty CRM under the client's record as a Feature Request note, and reply: *"আমরা note করে রাখলাম। আমাদের next update-এ এটা বিবেচনা করব।"* No promise of delivery.

---

### 2.4 Canned Responses — Top 3 Non-Technical User Errors

**These are the three most frequent client errors.** Every support agent must know these by heart. Copy exactly or adapt slightly for natural language — do not paraphrase in a way that loses the key instruction.

---

**Canned Response 1: "আমার inbox-এ নতুন message আসছে না" (No new messages in inbox)**

*This is almost always a QR/Baileys session disconnect. The client's phone went offline or the WhatsApp Web session expired.*

```
Banglish response (WhatsApp):

"[Name] ভাই, সমস্যাটা হলো WhatsApp connection-টা disconnect হয়ে গেছে।
এটা WhatsApp Web-এর মতো — phone বন্ধ থাকলে বা internet না থাকলে 
মাঝেমাঝে এটা হয়।

Fix করতে:
1. Dashboard-এ যান → Settings → Channels
2. WhatsApp-এর পাশে 'Reconnect' button দেখবেন
3. Click করলে একটা QR code আসবে
4. আপনার phone দিয়ে scan করুন (same as before)

Scan হলেই আবার সব message আসা শুরু হবে।

আমরা near future-এ Official WhatsApp API-তে upgrade করব — 
তখন এই সমস্যা আর হবে না। 💪

কোনো সমস্যা হলে screenshot পাঠান।"
```

*If the client is on DFY Premium, offer to reconnect it remotely via Evolution API without asking the client to do it themselves.*

---

**Canned Response 2: "আমার receipt link কাজ করছে না / customer বলছে খুলছে না" (Receipt link not opening)**

*This is usually a network issue on the buyer's end (slow mobile data) or an old/invalid receipt token being reshared.*

```
Banglish response:

"[Name] ভাই, receipt link সাধারণত সব device-এ খুলে। 
কিছু জিনিস check করুন:

1. Link-টা কি পুরোটা গেছে? WhatsApp মাঝেমাঝে long link cut করে।
   Full link: [paste the receipt URL from the order page]

2. Customer-এর internet connection check করতে বলুন — 
   link-টা কিন্তু heavy না, সাধারণ মোবাইল data-তেই খুলবে।

3. যদি এখনও না খুলে, আমাকে order number [UDD-XXXX] বলুন —
   আমি check করি কোনো technical issue আছে কিনা।

আপাতত customer-কে WhatsApp-এ order summary লিখে দিতে পারেন,
পরে receipt link-টা আবার পাঠান।"
```

---

**Canned Response 3: "Staff login হচ্ছে না / Staff dashboard দেখতে পাচ্ছে না" (Staff cannot log in)**

*This is usually an OTP delivery issue (wrong number, typo) or a staff member who was invited but not yet activated in Supabase.*

```
Banglish response:

"[Name] ভাই, staff-এর login issue সাধারণত দুইটা কারণে হয়:

কারণ ১: নম্বর ঠিক আছে কিনা দেখুন।
  → আপনি কোন number দিয়ে invite করেছিলেন?
  → Staff-এর phone-এ কি OTP এসেছে?
  → OTP না এলে: Settings → Staff → [staff name] → 
    'Resend Invite' button press করুন।

কারণ ২: Staff ভুল জায়গায় login করছে।
  → Login page: https://app.uddokta.com/login
  → এখানেই OTP enter করতে হবে। 
  → অন্য কোনো link দিয়ে login হবে না।

এই দুইটা check করুন। তারপরেও সমস্যা হলে staff-এর 
number আর আপনার business name জানান — 
আমরা manually check করব।"
```

---

### 2.5 Support Escalation Path

```
Client contacts support via WhatsApp or Chatwoot web widget
  │
  ▼
Junior Support Agent (Chatwoot)
  │ Can resolve: Sev 3 issues, canned-response scenarios,
  │ template changes, training questions, Sev 2 config issues
  │
  ├── Resolved → Close ticket, tag "resolved"
  │
  └── Cannot resolve within 4 hours → Escalate:
        │
        ▼
      Senior Support / Operations Lead
        │ Can resolve: Sev 2 issues, n8n config changes,
        │ Evolution API reconnects, Supabase data queries
        │
        ├── Resolved → Close ticket, post resolution note
        │
        └── Requires code change or infra fix → Escalate:
              │
              ▼
            Tech Lead (Internal Engineering inbox in Chatwoot)
              │ All Sev 1 issues land here directly
              │ Handles: Vercel deploys, n8n workflow edits,
              │ Docker container restarts, Supabase migrations
              │
              └── Post-resolution: write audit note in tenant record
```

**No client ever contacts the Tech Lead directly.** All escalations go through the Chatwoot internal inbox. The Tech Lead does not have a client-facing WhatsApp number.

---

## 3. Infrastructure Monitoring & Maintenance

> **Oracle VPS operational reality check:** The Oracle Cloud Free Tier ARM VPS is a single point of failure. There is no managed failover. The operational goal is not to achieve five-nines uptime — it is to detect failures fast, recover completely in under 60 minutes, and communicate proactively to affected clients. Every procedure in this section is written against that reality.

### 3.1 Daily Health Check Routine

Run this check every morning before 10:00 AM BST. This is a 5-minute procedure.

**Step 1 — Dashboard Health Screen**

Open `https://app.uddokta.com/admin/health`.

All indicators must be green:
- n8n: ✅
- Chatwoot: ✅
- Evolution API: ✅
- Typebot: ✅
- Dify: ✅
- Supabase connectivity: ✅
- Realtime: ✅

If any indicator is red:
- Note which service
- SSH into the VPS and run Step 2

**Step 2 — VPS Container Status**

```bash
ssh uddokta@{VPS_PUBLIC_IP}
cd /opt/uddokta/infra

# Check all containers are running:
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.RunningFor}}"

# Look for any container with:
#   - "Exit" in status → container crashed
#   - "Restarting" in status → container is crash-looping
#   - "Up X seconds" with X < 60 → just restarted (likely crash-loop recovering)

# Check RAM pressure (critical on 24GB Oracle VPS):
free -h
# WARNING threshold: available < 4 GB
# CRITICAL threshold: available < 2 GB

# Check CPU (should be low at rest — ARM4 vCPU):
top -bn1 | grep "Cpu(s)"
# WARNING: sustained >70% usage without an active video generation job

# Check disk (200GB volume):
df -h /
# WARNING threshold: >75% used
# CRITICAL threshold: >90% used
```

**Step 3 — n8n Execution Log Review**

```
Access n8n at https://{VPS_DOMAIN}/n8n
Navigate: Executions → filter by "Error" status

Review all error executions from the last 24 hours.
Classify each:
  - "Message send failed" (Evolution API timeout) → expected, check retry queue depth
  - "Supabase RLS violation" → URGENT, investigate immediately (may indicate a data bug)
  - "Apify Actor timeout" → non-critical, lead audit delayed
  - "Dify API error" → non-critical, audit fallback triggered
  - "Chatwoot API 401" → URGENT, Chatwoot API token expired for a tenant

For each ERROR execution: note the execution ID, workflow name, and failing node.
If 3+ consecutive failures on the same workflow within 1 hour → Severity 1 event.
```

**Step 4 — Supabase Usage Check**

```
Access Supabase Dashboard → Project Settings → Usage

Check:
  Database size: must be < 400 MB (limit: 500 MB)
    WARNING action at 400 MB: run cleanup query (see 3.3 below)
    CRITICAL action at 480 MB: immediately archive old execution logs,
    delete resolved audit_logs older than 90 days

  Bandwidth: must be < 1.6 GB for the month (limit: 2 GB)
    WARNING action at 1.6 GB: restrict public receipt page image sizes,
    check for any N+1 query patterns in dashboard pages

  Realtime connected clients: note the count
    Sudden spike (>50 at once) may indicate a connection leak in the Next.js app
```

---

### 3.2 Oracle VPS RAM Management — Container Priority

The Oracle ARM VPS has 24 GB RAM. Under normal load, all services consume ~5.6 GB idle. When MoneyPrinterTurbo is actively generating video, total usage can reach ~17 GB. This leaves ~7 GB headroom. However, if multiple services are under simultaneous load, OOM (out-of-memory) events can kill containers.

**Container priority hierarchy (high → low criticality):**

```
TIER 1 — NEVER kill (core product)
  caddy, n8n, postgres_n8n, chatwoot_app, chatwoot_sidekiq,
  chatwoot_pg, chatwoot_redis, evolution

TIER 2 — Kill if OOM imminent (internal ops, non-client-facing)
  twenty, twenty_db, twenty_redis,
  typebot_builder (typebot_viewer must stay up)

TIER 3 — Kill first if OOM (marketing pipeline)
  mpt_wrapper, dify_api, dify_worker, dify_web, dify_db,
  listmonk, mautic, mautic_db
```

**OOM prevention procedure:**

If `free -h` shows available RAM < 3 GB:

```bash
# Immediately stop Tier 3 services:
docker compose stop mpt_wrapper dify_api dify_worker dify_web mautic

# Check if it recovered:
free -h

# If still < 3 GB, stop Tier 2:
docker compose stop twenty typebot_builder

# After the RAM pressure event resolves, restart services:
docker compose start mpt_wrapper dify_api dify_worker dify_web mautic twenty typebot_builder

# Log the OOM event:
echo "$(date): OOM event. Stopped: mpt_wrapper, dify. RAM at event: $(free -h | grep Mem)" \
  >> /opt/uddokta/logs/oom-events.log
```

**Scheduled memory-intensive work:**

MoneyPrinterTurbo video generation (the most RAM-intensive task) is scheduled at **02:00 AM BST** by the n8n cron (as specified in the Implementation Plan). This is the VPS's lowest-traffic period. Do not reschedule it to daytime hours.

---

### 3.3 Supabase Free Tier Management

The Supabase Free Tier has a **500 MB database limit** and **2 GB bandwidth/month**. These limits will eventually be hit as the client base grows. The following procedures keep usage within bounds until a Pro plan upgrade is justified.

**Monthly cleanup query (run on the 1st of each month):**

```sql
-- Delete n8n execution error logs older than 30 days
-- (n8n stores these in its own PostgreSQL, not Supabase)
-- Access via: docker compose exec postgres_n8n psql -U {N8N_DB_USER} n8n
DELETE FROM execution_entity WHERE "startedAt" < NOW() - INTERVAL '30 days';
VACUUM ANALYZE execution_entity;

-- In Supabase: delete resolved audit_logs older than 90 days
DELETE FROM public.audit_logs
WHERE created_at < NOW() - INTERVAL '90 days'
  AND action NOT IN ('tenant_provisioned', 'tenant_suspended', 'data_export');
-- Never delete: provisioning events, suspension events, data export events

-- In Supabase: archive message_retry_queue entries older than 7 days
DELETE FROM public.message_retry_queue
WHERE created_at < NOW() - INTERVAL '7 days'
  AND attempt_count >= max_attempts;

-- Check current database size after cleanup:
SELECT pg_size_pretty(pg_database_size('postgres')) AS db_size;
```

**Bandwidth reduction protocols:**

The most common bandwidth consumers are:
1. Public receipt page loads (images and fonts)
2. Supabase Realtime subscriptions (each connected client generates traffic)
3. Admin bulk queries

```
If monthly bandwidth approaches 1.5 GB:
  Action 1: Enforce lazy loading on all images in the receipt page.
  Action 2: Reduce Realtime subscription polling to every 5 seconds
            (currently 1-2 second via Supabase default).
  Action 3: Paginate admin workspace list to 10 records per page
            (currently 50 — large tenants have large payloads).
  Action 4: Move product images to Cloudflare R2 public bucket
            and serve from R2 directly (zero Supabase bandwidth).
```

**When to upgrade Supabase to Pro ($25/month):**

Trigger the upgrade conversation when **any two** of the following are true:
- Database size > 350 MB consistently for 2 consecutive months
- Monthly bandwidth > 1.5 GB for 2 consecutive months
- Active tenant count > 20
- Daily Realtime event volume > 50,000 events

At that point, the $25/month Supabase Pro plan is self-funding via client subscriptions and is non-negotiable. Do not allow the database to hit 480 MB before upgrading — emergency cleanup at 490 MB is a crisis, not a procedure.

---

### 3.4 n8n Webhook Error Handling Protocol

**n8n is the central nervous system.** A silent failure in n8n means messages are not being routed, orders are not being updated, and reports are not being generated — and the client has no idea. Proactive monitoring is mandatory.

**Monitoring frequency:** Daily (part of the morning health check, Step 3 above).

**Error classification and resolution procedure:**

```
ERROR TYPE 1: "Evolution API — Message send failed"
  Cause: WhatsApp Cloud API rate limit or temporary server error.
  Identify: n8n execution log shows HTTP 429 or 5xx from Evolution API.
  Check: SELECT COUNT(*) FROM message_retry_queue WHERE created_at > NOW() - INTERVAL '1 hour';
  Action:
    - If retry queue count < 20: normal backoff, no action needed.
    - If retry queue count > 50: possible systematic issue.
      Check Evolution API directly:
        curl https://{VPS_DOMAIN}/evolution/instance/fetchInstances \
          -H "apikey: {EVOLUTION_API_KEY}"
      If instance state is not "open": reconnect the instance.
    - If retry queue count > 200: Severity 1 — escalate to Tech Lead.

ERROR TYPE 2: "Chatwoot API — 401 Unauthorized"
  Cause: Chatwoot API token expired for a specific tenant.
  Identify: n8n execution log shows Chatwoot HTTP 401, includes tenant_id.
  Action:
    1. Log into Chatwoot at https://{VPS_DOMAIN}/chatwoot
    2. Find the account for the affected tenant.
    3. Settings → API Access Tokens → regenerate token.
    4. Update the token in Supabase:
       UPDATE public.tenants SET chatwoot_api_token = '{new_token}'
       WHERE id = '{tenant_id}';
    5. Re-run the failed n8n execution manually.

ERROR TYPE 3: "Supabase — Row Level Security violation"
  Cause: An n8n workflow wrote data without a tenant_id, or used the
  wrong tenant_id (data integrity bug, not a security breach yet).
  Identify: n8n execution log shows Supabase 403 with "new row violates
  row-level security policy."
  Action:
    URGENT — escalate immediately.
    Query: SELECT * FROM public.messages WHERE tenant_id IS NULL LIMIT 10;
    If rows found: these are orphaned records. Identify the n8n workflow
    that created them from the execution log. Fix the workflow node that
    is not injecting tenant_id. Delete the orphaned rows.

ERROR TYPE 4: "Typebot webhook — no response"
  Cause: Typebot service is down or the webhook URL changed.
  Identify: n8n execution log shows connection refused or timeout for
  the typebot-lead webhook trigger.
  Action:
    docker compose restart typebot_viewer typebot_builder
    Check logs: docker compose logs typebot_viewer --tail=50

ERROR TYPE 5: "Daily report — zero tenants processed"
  Cause: n8n cron ran but the tenant query returned empty.
  Identify: execution log shows "0 tenants found" in the daily report workflow.
  Action:
    Test query manually:
    SELECT id, slug FROM public.tenants WHERE plan != 'suspended';
    If this returns rows, the issue is in the n8n HTTP Request node (bad header/auth).
    Check that SUPABASE_SERVICE_ROLE_KEY in n8n credentials is still valid.
```

**Weekly n8n workflow audit:**

Every Monday, open n8n and verify these workflows are **Active** (not Paused):
```
[ ] evolution-inbound
[ ] outbound-message
[ ] retry-queue-processor
[ ] typebot-lead-intake
[ ] apify-audit-trigger
[ ] apify-audit-callback
[ ] daily-report-cron
[ ] sla-breach-check
[ ] video-generation
```

If any workflow is Paused, check the `Executions` tab for that workflow to understand why it was paused (n8n auto-pauses workflows that fail 5+ consecutive times). Fix the root cause, then re-activate.

---

## 4. Compliance & WhatsApp Risk Mitigation

> **Non-negotiable context:** Uddokta's core product infrastructure runs on the Meta WhatsApp Cloud API. A single spam incident from a client's number can trigger Meta to ban that phone number from the API. A pattern of spam incidents across multiple tenants can trigger Meta to revoke Uddokta's entire Meta App status — which would take down **all client inboxes simultaneously.** This is an existential risk. The procedures in this section are not guidelines — they are hard operational rules.

### 4.1 Acceptable Use Policy — What Uddokta Allows and Prohibits

Every client signs the following terms before workspace activation (included in the onboarding agreement, not buried in a terms page):

**Permitted uses:**
- Responding to customer-initiated messages (inbound → outbound is always allowed)
- Sending order confirmations and status updates to customers who have ordered
- Sending payment receipts and delivery notifications
- Follow-up messages to customers who have actively engaged in the last 24 hours
- Sending WhatsApp template messages (pre-approved by Meta) for re-engagement after 24-hour window

**Strictly prohibited:**
- Broadcasting promotional messages to customer lists without explicit opt-in records
- Importing phone number lists from external sources and messaging them cold
- Sending more than 5 bulk template messages per day without Meta's Tier 2 messaging approval
- Using Uddokta to send political content, adult content, or financial scheme promotions
- Creating or connecting fake/secondary WhatsApp numbers to bypass API limits
- Using the QR/Baileys demo mode for production-volume messaging (>200 messages/day)

**Enforcement mechanism:** The client signs this policy. If a violation is detected:
1. First offense: immediate suspension of outbound messaging (flip `outbound_paused: true` in the tenant record), WhatsApp warning to the owner, 48-hour grace period to acknowledge.
2. Second offense: workspace suspension. Data preserved, access revoked.
3. Third offense: workspace terminated, data exported and delivered to client, no refund.

---

### 4.2 Spam Detection — n8n Guardrails

The following n8n rules are active on all tenants at all times. These cannot be disabled by client request.

**Rule 1 — Outbound volume cap:**
```
n8n Function node in the outbound-message workflow:

const tenantId = $input.body.tenant_id;
const oneHourAgo = new Date(Date.now() - 60 * 60 * 1000).toISOString();

// Count outbound messages in the last 1 hour for this tenant
const result = await $helpers.httpRequest({
  method: 'GET',
  url: `${SUPABASE_URL}/rest/v1/messages?tenant_id=eq.${tenantId}&direction=eq.outbound&created_at=gte.${oneHourAgo}&select=id`,
  headers: { 'apikey': SUPABASE_SERVICE_ROLE_KEY, ... }
});

const count = result.length;
if (count >= 80) {
  // WhatsApp Cloud API limit is 80 messages/second per WABA
  // Our hourly cap is 80 to stay safe
  throw new Error(`RATE_LIMIT: Tenant ${tenantId} hit hourly outbound cap.`);
}
```

**Rule 2 — Template message approval gate:**
Any outbound message that is initiated outside an active conversation (i.e., the last inbound from this customer was >24 hours ago) must use a Meta-approved template. If a non-template message is attempted for a cold window customer, n8n blocks it and creates a Chatwoot internal note:
*"Message blocked: 24-hour window expired. Use an approved template to re-engage this customer."*

**Rule 3 — New customer cold-message block:**
A customer who has never messaged the tenant cannot receive an outbound message. n8n verifies: `SELECT COUNT(*) FROM messages WHERE contact_id = $1 AND direction = 'inbound'`. If count = 0, the message is blocked with a note: *"Cannot initiate first contact via automated message. Wait for the customer to message first."*

---

### 4.3 Meta Template Rejection Protocol

When a Meta message template is submitted for approval and rejected, this is the response procedure:

**Step 1 — Identify the rejection reason**

Meta provides one of these rejection reasons:
- `INVALID_FORMAT` — template syntax error
- `TAG_CONTENT_MISMATCH` — template category doesn't match content
- `PROMOTIONAL_CONTENT` — content was flagged as promotional when submitted as utility/authentication
- `ABUSIVE_CONTENT` — language triggered a content policy flag
- `SCAM` — template appears deceptive

**Step 2 — Do not re-submit immediately**

Rapid resubmission of a rejected template increases Meta's risk score for the App. Wait at minimum 24 hours before resubmitting.

**Step 3 — Fix and resubmit**

| Rejection Reason | Fix Action |
|---|---|
| `INVALID_FORMAT` | Check double curly braces `{{1}}`, ensure all variables are numbered consecutively |
| `TAG_CONTENT_MISMATCH` | Change the template category in the submission (Utility → Marketing or vice versa) |
| `PROMOTIONAL_CONTENT` | Remove all sale/discount/offer language from Utility templates; move to Marketing category |
| `ABUSIVE_CONTENT` | Rewrite the template entirely; avoid words like "urgent," "limited," "act now," "guaranteed" |
| `SCAM` | This is a serious flag. Do not resubmit without reviewing the template with the Tech Lead. |

**Step 4 — Log the rejection**

```sql
INSERT INTO public.audit_logs (tenant_id, actor_id, action, target_type, metadata)
VALUES (
  '{tenant_id}',
  NULL, -- system event
  'meta_template_rejected',
  'whatsapp_template',
  '{"template_name": "...", "rejection_reason": "...", "submitted_at": "...", "resubmit_after": "..."}'
);
```

**Step 5 — Notify the client (Severity 3 — template change)**

Use Canned Response 1 as the base, adapted for template rejection context.

---

### 4.4 Handling Meta Warning Flags on the App

If Meta sends a quality warning or policy violation notice to the Uddokta Meta App account (visible in Meta Business Suite → WhatsApp Manager → Insights):

**Yellow flag (Quality Pending):**
- Review the message logs for the flagged phone number
- Identify the message or template that triggered the flag
- Immediately pause outbound messaging for that tenant: set `outbound_paused: true`
- Contact the client to review their recent messages
- Submit an appeal through Meta Business Suite if the flag is unwarranted

**Red flag (Quality Flagged / Messaging Limit Reduction):**
- Pause ALL outbound template messages system-wide (not just the flagged tenant)
- Alert the Tech Lead and Founder immediately
- Review n8n logs for any systematic spam pattern across tenants
- Contact Meta through the official support channel within 24 hours
- Do not reinstate messaging until the flag is resolved

**Account Integrity Violation (App suspended):**
This is an existential event. Response:
1. Immediate notification to all affected clients via email/SMS
2. Activate the QR/Baileys fallback for clients currently on Production API
3. File a formal appeal through Meta Business Suite
4. Prepare client communication: *"আমাদের WhatsApp connection temporarily pause হয়েছে। আমরা Meta-র সাথে কাজ করছি। আপনার সব data safe। আপডেট আসছে।"*
5. If appeal fails within 7 days: begin transition to an alternative WhatsApp API provider (WATI, 360Dialog) as a backup.

---

### 4.5 Demo Mode (QR/Baileys) — Client Briefing Requirements

Before activating any client on QR/Baileys demo mode, the Operations Lead must verbally confirm the following with the client and log the confirmation in the Twenty CRM note:

```
"[Client Name] আপনাকে জানাতে চাই যে, আমরা এখন WhatsApp-এর একটা 
demo connection দিচ্ছি। এটা WhatsApp Web-এর মতো কাজ করে।

তিনটা জিনিস মনে রাখবেন:
১. আপনার phone বন্ধ হলে বা internet না থাকলে disconnect হতে পারে।
২. এটা bulk message পাঠানোর জন্য না — শুধু normal customer reply-র জন্য।
৩. দিনে ২০০-এর বেশি message পাঠালে WhatsApp number block হওয়ার risk আছে।

Official WhatsApp API-তে upgrade করলে এই তিনটা সমস্যাই চলে যাবে।
Setup হয়ে গেলে আমরা upgrade-এর process দেখাব।

এটা কি আপনি বুঝলেন? [Confirm: YES/NO]"

Log in Twenty CRM note: "Demo mode risk briefing delivered. Client confirmed: YES. Date: [date]."
```

If the client confirms "NO" or does not respond within 24 hours, do not activate the demo mode connection.

---

## 5. Offboarding & Data Export Protocol

> **Operational philosophy:** A churning client is not a failure to hide. It is a potential case study, a referral risk or opportunity, and a legal data obligation. Every offboarding must leave the client with their complete business data in a portable format, a clear explanation of what has been deactivated, and a genuine offer to return. The way Uddokta handles churning clients is what Bangladesh's F-commerce seller network will talk about.

### 5.1 Churn Detection & Early Intervention

**Churn risk signals (n8n monitors these daily):**

```sql
-- Tenants with no orders in last 14 days (active account, no usage)
SELECT t.slug, t.name, t.plan,
       MAX(o.created_at) AS last_order_date,
       DATEDIFF('day', MAX(o.created_at), NOW()) AS days_since_last_order
FROM public.tenants t
LEFT JOIN public.orders o ON o.tenant_id = t.id
WHERE t.plan != 'suspended'
GROUP BY t.id
HAVING days_since_last_order > 14 OR last_order_date IS NULL;

-- Tenants with high Sev 2+ support ticket rate (frustrated clients)
-- Queried from Chatwoot API: accounts with >3 tickets in last 7 days
```

When a churn signal is detected:
1. n8n auto-creates a Chatwoot internal ticket tagged `churn-risk` and assigns to the Operations Lead.
2. Operations Lead reviews the tenant's usage metrics in Supabase and their support history in Chatwoot.
3. Update Twenty CRM stage to **"Churn Risk"**.
4. Operations Lead initiates a WhatsApp check-in within 24 hours:

```
Banglish WhatsApp check-in message:

"[Name] ভাই, কেমন আছেন? আমরা দেখছি আপনার dashboard এ কিছুদিন 
যাবৎ activity কম। কোনো সমস্যা হচ্ছে কি? 

আপনার কোনো feature দরকার, কোনো change করা দরকার, বা কিছু বুঝতে 
অসুবিধা হচ্ছে — সব বলুন, আমরা fix করব।

আমরা চাই Uddokta আপনার business-এর জন্য সত্যিই কাজে লাগুক।"
```

If the check-in call identifies a fixable issue (training gap, broken feature, missing product), fix it immediately and update the Twenty CRM note. Move stage back to "Active Client."

If the client explicitly requests cancellation, proceed to Section 5.2.

---

### 5.2 Cancellation Request Handling

When a client says they want to cancel, the Operations Lead handles it personally — never a junior support agent.

**Conversation framework (WhatsApp or call):**

```
Phase 1 — Listen without defending (2 minutes):
"বুঝলাম। কি কারণে বন্ধ করতে চাইছেন সেটা একটু বলুন — 
আমরা note করতে চাই।"
[Let the client explain. Do not interrupt. Write it down.]

Phase 2 — Address if addressable (2 minutes):
If the reason is fixable (e.g., "staff use করে না," "WhatsApp disconnect হয়"):
"এটা আমরা fix করতে পারব। আপনাকে cancel করার আগে একটা সুযোগ দিন।
[Specific fix action] করি — ৪৮ ঘণ্টার মধ্যে ঠিক করে দেব।"

If the reason is financial ("monthly charge afford হচ্ছে না"):
"আমরা একটা কম-cost option দেখতে পারি। Starter plan-এ আসলে 
[price] কম হবে। এটা কি কাজে আসবে?"

Phase 3 — Accept gracefully if unresolvable:
If the client insists on cancellation after one fix offer:
"ঠিক আছে ভাই। আমরা বুঝি। আপনার সব data — products, orders, 
customers — সব একটা Google Sheet-এ দিয়ে দেব। কিছুই হারাবে না।
আপনি ভবিষ্যতে ফিরে আসতে চাইলে আমরা এখানে আছি।"
```

**After the cancellation conversation:**
1. Update Twenty CRM: move to "Churned" stage. Add note: reason for churn (verbatim), fix offered (yes/no), reason fix was declined.
2. Begin the data export procedure (Section 5.3) immediately — do not wait for the billing cycle to end.
3. Send the data export to the client before deactivating the workspace.

---

### 5.3 Data Export Procedure — Complete Client Data Package

This procedure must be executed by the Operations Lead, not a junior agent. The client receives their data **before** workspace access is revoked.

**Step 1 — Generate Customer Export from Supabase**

```sql
-- Connect to Supabase via the CLI or psql:
-- psql postgresql://{SUPABASE_DB_HOST}/postgres?sslmode=require

\COPY (
  SELECT
    c.name AS "Customer Name",
    c.phone AS "Phone",
    c.full_address AS "Address",
    c.city AS "City",
    c.area AS "Area",
    c.tags AS "Tags",
    c.total_orders AS "Total Orders",
    c.total_spent AS "Total Spent (BDT)",
    c.last_order_at AS "Last Order Date",
    c.risk_notes AS "Notes",
    c.created_at AS "Added On"
  FROM public.contacts c
  WHERE c.tenant_id = '{tenant_id}'
  ORDER BY c.created_at DESC
) TO '/tmp/export_{slug}_customers.csv' CSV HEADER;
```

**Step 2 — Generate Order Export from Supabase**

```sql
\COPY (
  SELECT
    o.order_number AS "Order Number",
    o.created_at AS "Order Date",
    o.status AS "Status",
    c.name AS "Customer Name",
    c.phone AS "Customer Phone",
    o.delivery_address AS "Delivery Address",
    o.delivery_city AS "City",
    o.delivery_area AS "Area",
    o.product_lines::TEXT AS "Products",
    o.amount AS "Total Amount (BDT)",
    o.discount AS "Discount (BDT)",
    o.delivery_charge AS "Delivery Charge (BDT)",
    o.payment_method AS "Payment Method",
    o.courier AS "Courier",
    o.tracking_number AS "Tracking Number",
    o.internal_notes AS "Notes",
    o.updated_at AS "Last Updated"
  FROM public.orders o
  LEFT JOIN public.contacts c ON c.id = o.contact_id
  WHERE o.tenant_id = '{tenant_id}'
  ORDER BY o.created_at DESC
) TO '/tmp/export_{slug}_orders.csv' CSV HEADER;
```

**Step 3 — Generate Product Export from Supabase**

```sql
\COPY (
  SELECT
    p.name AS "Product Name",
    p.price AS "Price (BDT)",
    p.sale_price AS "Sale Price (BDT)",
    p.stock_status AS "Stock Status",
    p.variants::TEXT AS "Variants",
    p.delivery_note AS "Delivery Note",
    p.quick_reply_template AS "Quick Reply Template",
    p.image_url AS "Image URL",
    p.created_at AS "Added On"
  FROM public.products p
  WHERE p.tenant_id = '{tenant_id}'
  ORDER BY p.created_at DESC
) TO '/tmp/export_{slug}_products.csv' CSV HEADER;
```

**Step 4 — Upload to Google Sheets (preferred) or send as CSV files**

```bash
# Option A: Upload to Google Sheets via n8n
# Trigger the n8n workflow: "client-data-export"
# Input: { tenant_id, tenant_slug, export_files: [customers, orders, products] }
# n8n creates a new Google Sheet, uploads each CSV as a separate tab,
# and returns the shareable Google Sheets URL.

# Option B: Send as CSV files directly via WhatsApp
# Copy the 3 CSV files to a temp location on the VPS,
# then send each file as a WhatsApp document attachment via Evolution API.
```

**Step 5 — Deliver to the client**

```
WhatsApp message to client:

"[Name] ভাই, আপনার সব data তৈরি।

📊 Google Sheet link: [GOOGLE_SHEETS_URL]
(অথবা CSV files attached)

এতে আছে:
✅ আপনার সব customers ([count] জন)
✅ আপনার সব orders ([count] টা)  
✅ আপনার সব products ([count] টা)

এই link/files আপনার কাছে থাকবে। 
Uddokta বন্ধ হলেও আপনার data আপনার কাছে।

কোনো সমস্যা হলে জানান।"
```

Record the delivery confirmation in Chatwoot: `"Data export delivered to client. Google Sheet URL: [URL]. Confirmation received from client: [Yes/No]"`

---

### 5.4 Workspace Deactivation Procedure

Execute this **only after** the client has confirmed receipt of their data export.

**Step 1 — Pause all outbound messaging:**
```sql
UPDATE public.tenants SET
  plan = 'suspended',
  setup_checklist = jsonb_set(setup_checklist, '{suspended_at}', '"' || NOW()::TEXT || '"')
WHERE id = '{tenant_id}';
```

**Step 2 — Disconnect Evolution API instance:**
```bash
curl -X DELETE https://{VPS_DOMAIN}/evolution/instance/tenant-{slug}/deleteInstance \
  -H "apikey: {EVOLUTION_API_KEY}"
```

**Step 3 — Deactivate Supabase Auth user:**
```javascript
// Via Supabase Admin API (call from the Next.js admin console):
await supabase.auth.admin.updateUserById(userId, { ban_duration: 'none' }) // or delete
// Do NOT delete the user — set their profile role to 'suspended'
// Deleting auth users makes audit log entries orphaned
```

Alternatively, update via SQL:
```sql
-- Revoke all active sessions for the tenant's users
-- This is done via Supabase Dashboard → Authentication → Users → [user] → Ban user
-- Do this for ALL users (owner + all staff) in the tenant
```

**Step 4 — Archive (not delete) all tenant data:**
```sql
-- Mark the tenant as archived (data is preserved, not deleted)
UPDATE public.tenants SET
  plan = 'archived',
  slug = slug || '-archived-' || EXTRACT(EPOCH FROM NOW())::INT
WHERE id = '{tenant_id}';
-- Appending -archived-{timestamp} to the slug prevents re-registration conflicts
```

**Step 5 — Update Twenty CRM:**
```
Stage: "Churned"
Custom fields:
  offboarding_complete: true
  data_export_delivered: true
  workspace_deactivated_date: {today}
  churn_reason: {verbatim reason from conversation}
  reactivation_offer_made: true/false
```

**Step 6 — Data Retention Policy:**

Archived tenant data is retained in Supabase for **90 days** after deactivation. After 90 days, the following is deleted:
- All rows in `messages`, `conversations`, `orders` (large tables)
- All rows in `audit_logs` for this tenant

The following is retained **permanently** (legal/compliance):
- `tenants` record (with `archived` status)
- `profiles` records for the owner (with suspended status)
- `contacts` table — kept as-is for 12 months, then anonymized (name and phone replaced with hash)

A deletion cron job in n8n runs on the 1st of each month and processes all tenants where `archived_at < NOW() - INTERVAL '90 days'`.

---

### 5.5 Reactivation Path — Keeping the Door Open

A churned client who wants to return is always welcomed. The reactivation procedure is:

1. Identify their `archived` tenant record in Supabase.
2. If within 90-day data retention window: restore the workspace by setting `plan = 'starter'` (or whichever plan they pay for) and removing the `-archived-{timestamp}` suffix from the slug.
3. If outside the 90-day window: provision a fresh workspace. Customers and orders from the data export can be re-imported via the Admin product import tool.
4. Skip the Typebot intake bot if they can confirm their data is unchanged — go directly to the Aha! Moment handover.
5. In Twenty CRM: move from "Churned" back to "Paid/Onboarding" with a note: "Reactivated — previous tenant data [restored/new]."

The reactivation WhatsApp message from Operations Lead:

```
"[Name] ভাই, স্বাগতম ফিরে! 🎉

আপনার account আবার active করে দিচ্ছি।
[যদি data restore হয়]: আপনার আগের সব data — orders, customers — 
সব ঠিকঠাক আছে।

একটু পরেই login OTP যাবে। সব ready হলে জানাব।"
```

---

## Appendix A: Quick Reference — Critical Credentials & Locations

> Store this table in a password manager (Bitwarden recommended). Never in a WhatsApp group, Google Sheet, or email.

| Credential | Location | Who Has Access |
|---|---|---|
| Supabase Service Role Key | infra/.env + VPS /opt/uddokta/infra/.env | Tech Lead only |
| n8n Admin Password | infra/.env | Tech Lead + Ops Lead |
| Evolution API Key | infra/.env | Tech Lead + Ops Lead |
| Chatwoot Super Admin | infra/secrets/chatwoot-admin.txt (gitignored) | Tech Lead only |
| Twenty CRM Admin | infra/secrets/twenty-admin.txt (gitignored) | Tech Lead + Ops Lead |
| Vercel Deploy Token | GitHub Actions secrets | Tech Lead only |
| Oracle VPS SSH Key | ~/.ssh/uddokta_vps (local machine) | Tech Lead only |
| Meta App Secret | infra/.env + Meta Business Suite | Tech Lead + Founder |
| Cloudflare R2 Keys | infra/.env | Tech Lead only |
| Apify API Token | infra/.env | Tech Lead + Ops Lead |

---

## Appendix B: Twenty CRM — Standard Field Reference

All client records in Twenty CRM must use these exact field values for consistency.

**Opportunity Stage Values:**
```
Scraped → Audit Sent → Demo Watched → Setup Fee Requested →
Paid/Onboarding → Setup Started → Active Client → Churn Risk → Churned
```

**Custom Fields on Opportunity (required before moving to "Active Client"):**
```
uddokta_tenant_id       (UUID from Supabase)
dashboard_url           (https://app.uddokta.com)
channel_mode            (demo | production)
plan                    (starter | growth | dfy_premium)
products_count          (integer)
setup_complete_date     (date)
first_login_sent        (boolean)
aha_moment_completed    (boolean)
churn_reason            (text, filled only on Churned stage)
data_export_delivered   (boolean, filled only on Churned stage)
needs_followup_training (boolean)
```

---

## Appendix C: Emergency Contacts & Escalation Numbers

| Scenario | Contact | Channel |
|---|---|---|
| Oracle VPS down | Oracle Cloud Support Portal (free tier includes basic support) | https://support.oracle.com |
| Meta App suspended | Meta Business Help Center | https://business.facebook.com/help |
| Supabase service outage | Supabase Status Page | https://status.supabase.com |
| Vercel deployment failure | Vercel Support | https://vercel.com/support |
| Cloudflare R2 issue | Cloudflare Status | https://www.cloudflarestatus.com |
| Apify Actor failure | Apify Support | https://apify.com/help |

**Internal escalation order for Severity 1 events:**
```
Junior Support → Operations Lead → Tech Lead → Founder
(Each has 15 minutes to acknowledge before escalating to the next level)
```

---

*End of Document 8: Setup & Operations SOP*

*This document, combined with Documents 1–7, constitutes the complete Uddokta operational and engineering blueprint. Review this SOP before every new client onboarding and after every Severity 1 incident.*
