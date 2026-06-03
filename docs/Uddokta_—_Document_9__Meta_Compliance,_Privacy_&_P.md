# Uddokta — Document 9: Meta Compliance, Privacy & Platform Risk SOP
### Version 1.0 | Status: Strict Defensive Manual

> **Confidential Internal Document:** This SOP is a mandatory defensive manual for Uddokta internal engineering and operations staff. It outlines the protocols required to maintain platform integrity, ensure Meta/WhatsApp compliance, and mitigate legal and operational risks in the Bangladeshi SME context.

---

## 1. Meta & WhatsApp Cloud API Compliance Protocol

### 1.1 WhatsApp Template Message Approval Rules
To prevent Meta from banning the Uddokta Meta App or our clients' WhatsApp Business Account (WABA) numbers, all template messages must adhere to the following strict rules:
*   **Utility & Authentication Only for Cold Leads:** Promotional templates (marketing) must never be sent to a user who has not explicitly opted in.
*   **Bangla/English Clarity:** Templates must use clear, professional language. Avoid aggressive sales jargon or excessive emojis that trigger Meta's spam filters.
*   **Variable Sanitization:** Any dynamic variables (e.g., `{{1}}` for customer name) must be sanitized to prevent the injection of prohibited links or offensive content.
*   **Mandatory Opt-Out:** Every outbound marketing template **must** include a "Stop" or "Unsubscribe" button.

### 1.2 The "Opt-In & Spam Prevention Engine" (n8n Configuration)
We utilize n8n to act as a compliance gatekeeper for all outbound messaging:
*   **Rate Limiting:** n8n workflows are configured with a "Wait" node or a queue system to throttle outbound promotional messages. We limit bursts to no more than 20 messages per minute per WABA number.
*   **Opt-Out Registry:** A dedicated `opt_out` table in Supabase tracks users who have unsubscribed. The n8n "Outbound Message" workflow performs a mandatory lookup; if the recipient's phone number exists in the `opt_out` table, the message is dropped immediately with a `log_reason: "User Opted Out"`.
*   **Engagement Scoring:** n8n monitors "Read" and "Replied" status. If a client's WABA number has a response rate lower than 5% over 100 messages, the system automatically pauses their outbound campaigns and alerts Uddokta Operations for a manual review.

### 1.3 Evolution-API to WhatsApp Cloud API Transition
*   **The Bridge (Starter Tier):** Clients on the Starter tier or in testing use the `evolution-api` (QR-based) bridge. We must warn clients that this method carries a higher risk of account suspension if used for mass messaging.
*   **The Official Path (Growth Tier):** As clients transition to the Growth tier, we migrate them to the official Meta WhatsApp Cloud API.
*   **Migration Protocol:**
    1.  Verify the client's Meta Business Manager and WABA status.
    2.  Provision the official Phone Number ID in the Uddokta Meta App.
    3.  Update the `tenants` table in Supabase to switch the `channel_mode` from `demo` to `production`.
    4.  Redirect n8n webhooks from the Evolution instance to the Meta Webhook endpoint.

---

## 2. Data Security & Multi-Tenancy Lockdown (Supabase)

### 2.1 Zero-Trust Row Level Security (RLS) Auditing
Multi-tenancy is enforced at the database level. Client A must never be able to access Client B's data.
*   **The `tenant_id` Anchor:** Every table (orders, contacts, messages, etc.) **must** contain a `tenant_id` column.
*   **RLS Policy Enforcement:** Every table must have RLS enabled with a policy that checks the user's JWT claim:
    ```sql
    CREATE POLICY "tenant_isolation_policy" ON public.orders
    FOR ALL USING (tenant_id = (auth.jwt() ->> 'tenant_id')::uuid);
    ```
*   **Weekly RLS Audit:** Operations must run the `rls_verification_test.sql` script weekly to ensure no policies have been accidentally altered or dropped.

### 2.2 Secure Credential Management
We never store raw secrets in the application code or unencrypted database fields.
*   **Vault Storage:** Meta API keys, Chatwoot tokens, and WABA secrets are stored in **Supabase Vault** or encrypted using a dedicated service-role-only encryption key managed by n8n.
*   **Environment Segregation:** Vercel environment variables are used for platform-level keys. Tenant-specific secrets are fetched via n8n at runtime and are never exposed to the frontend browser.
*   **Webhook Secret Verification:** All incoming webhooks from Meta or Evolution-API must be verified using the `X-Hub-Signature-256` or equivalent header before processing.

---

## 3. Client-Facing Terms of Service & Liability Shield

### 3.1 Plain-Banglish "Terms of Use" (Core Points)
We provide a simplified version of the legal terms so SME owners understand their responsibilities:
*   **ব্যবসার দায়িত্ব (Business Responsibility):** আপনি (বিক্রেতা) আপনার কাস্টমারের সাথে সকল লেনদেন এবং পণ্যের গুণগত মানের জন্য দায়ী।
*   **স্প্যামিং নিষেধ (No Spamming):** কাস্টমারের অনুমতি ছাড়া মেসেজ পাঠানো যাবে না। বেশি স্প্যাম করলে আপনার WhatsApp নম্বর বন্ধ হতে পারে, যার জন্য Uddokta দায়ী নয়।
*   **ডেটা নিরাপত্তা (Data Security):** আপনার অর্ডারের তথ্য আমরা সুরক্ষিত রাখি, কিন্তু আপনার লগইন ডিটেইলস অন্য কাউকে দেবেন না।

### 3.2 Explicit Liability Shield
Uddokta is a software provider, not a commerce participant. Our liability is limited as follows:
*   **No Liability for Losses:** Uddokta is not liable for lost revenue due to missed webhooks, Meta API downtime, or server outages.
*   **No Liability for Scams:** We are not responsible for fraudulent COD orders, fake payment screenshots, or courier losses.
*   **WhatsApp Ban Indemnity:** If a seller's WhatsApp number is banned due to their messaging behavior (spam, prohibited items), Uddokta will not provide a refund or be held liable for the loss of the communication channel.

---

## 4. Data Continuity & Trust Backups

### 4.1 Automated Google Sheets Export (n8n)
To build absolute trust with sellers who fear "SaaS Lock-in," we provide a portable data copy:
*   **The Protocol:** An n8n cron job runs every 24 hours for each active tenant.
*   **The Action:** It fetches all new orders and contacts from the last 24 hours and appends them to a dedicated Google Sheet owned by the client.
*   **The Benefit:** If Uddokta servers are unreachable, the client has a live, offline-accessible backup of their customer addresses and order statuses.

### 4.2 Supabase Database Snapshot & Backup
*   **Daily Snapshots:** Managed by Supabase's automated backup system.
*   **Off-site Backups:** Every Sunday at 02:00 BDT, an n8n workflow triggers a `pg_dump` of the database (excluding sensitive system tables), encrypts the file, and uploads it to a secure Cloudflare R2 bucket.

---

## 5. Incident Response & Triage Plan

### 5.1 Meta API/Webhook Failure Triage
If Meta changes their API structure or revokes access:
1.  **Detection:** n8n "Error Trigger" nodes alert the internal Slack/WhatsApp channel immediately.
2.  **Notification:** A platform-wide banner is pushed to all Next.js dashboards: *"Meta API is experiencing issues. Outbound messages may be delayed."*
3.  **Triage:** Engineering identifies if the issue is a breaking API change or a token expiration.
4.  **Resolution:** Update n8n nodes or refresh the Meta System User Token.

### 5.2 Prohibited Content & Scam Detection
If a client is caught selling prohibited items (e.g., restricted drugs, counterfeit currency) or running scams:
1.  **Immediate Suspension:** The `is_active` flag in the `tenants` table is set to `false`, revoking all API access and dashboard login.
2.  **Evidence Preservation:** A full database export for that tenant is taken and moved to the `compliance_hold` bucket in R2.
3.  **Communication:** The client is notified via their registered email/WhatsApp: *"Your account has been suspended for violating our Acceptable Use Policy (Section X)."*
4.  **Meta Reporting:** If the violation involves severe fraud, Uddokta will proactively report the WABA ID to Meta to protect the platform's overall reputation.

---
**End of Document 9**
