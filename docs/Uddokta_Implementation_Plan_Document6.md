# Uddokta — Implementation Plan
### Document 6 of 6 | Version 1.0

> **Execution Model:** Every phase is a hard dependency gate. No phase begins until all acceptance checks in the preceding phase pass. These instructions are written as direct commands for AI coding agents (Claude Code, Cursor, Windsurf). Paste each step sequentially. Do not parallelize across phases.

---

## Table of Contents

1. [Phase 1: Infrastructure Provisioning & Database Seeding](#phase-1-infrastructure-provisioning--database-seeding)
2. [Phase 2: Core Mobile PWA & Manual Order Board](#phase-2-core-mobile-pwa--manual-order-board)
3. [Phase 3: Internal Operations & Lead Funnel](#phase-3-internal-operations--lead-funnel)
4. [Phase 4: Open-Source Engine Orchestration](#phase-4-open-source-engine-orchestration)
5. [Phase 5: Automated AI Marketing Pipeline](#phase-5-automated-ai-marketing-pipeline)
6. [Phase 6: Testing, CI/CD & Launch Prep](#phase-6-testing-cicd--launch-prep)

---

## Phase 1: Infrastructure Provisioning & Database Seeding

> **Gate condition:** Phase 2 cannot start until Supabase responds to an authenticated query with enforced RLS, the Oracle VPS is reachable on ports 80/443 via HTTPS, and all environment variable files are populated and committed to `.gitignore`.

### 1.1 Oracle Cloud Free Tier ARM VPS — Provisioning

```
AGENT TASK 1.1.1
Provision an Oracle Cloud Free Tier ARM Ampere A1 instance with the
following exact specifications:
- Shape: VM.Standard.A1.Flex
- OCPUs: 4
- RAM: 24 GB
- Boot volume: 200 GB
- Image: Canonical Ubuntu 22.04 LTS (ARM64)
- Region: ap-mumbai-1 (lowest latency to Bangladesh)
Output the public IP address and confirm SSH access with key-pair
authentication only. Disable password SSH login.
```

```
AGENT TASK 1.1.2
SSH into the Oracle VPS and run the following bootstrap script verbatim.
Do not skip any step. Each command must return exit code 0 before proceeding.

# Update system
sudo apt-get update && sudo apt-get upgrade -y

# Install Docker Engine (official method for Ubuntu ARM64)
sudo apt-get install -y ca-certificates curl gnupg lsb-release
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

# Verify ARM64 Docker
docker --version
docker compose version
docker run --rm --platform linux/arm64 hello-world

# Create the application user
sudo useradd -m -s /bin/bash uddokta
sudo usermod -aG docker uddokta

# Create directory structure
sudo mkdir -p /opt/uddokta/{infra,backups,logs,secrets}
sudo chown -R uddokta:uddokta /opt/uddokta

# Configure Oracle VPS firewall (iptables — OCI uses host-level rules)
# Open HTTP, HTTPS, and SSH only
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT 7 -m state --state NEW -p tcp --dport 443 -j ACCEPT
sudo netfilter-persistent save

# Install rclone for R2 backup operations
curl https://rclone.org/install.sh | sudo bash
```

```
AGENT TASK 1.1.3
Create the Git monorepo structure on the local developer machine.
Run these commands in the project root:

mkdir -p uddokta/{apps/web,infra/{caddy,n8n/workflows,supabase/migrations,secrets},packages/mpt-wrapper,docs}
cd uddokta
git init
echo "infra/secrets/" >> .gitignore
echo "infra/.env" >> .gitignore
echo "apps/web/.env.local" >> .gitignore
echo "**/.env" >> .gitignore
echo "node_modules/" >> .gitignore
echo ".DS_Store" >> .gitignore
git add .gitignore
git commit -m "chore: initial repo structure"
```

### 1.2 Docker Network & Base Configuration

```
AGENT TASK 1.2.1
Create the master Docker Compose base file at infra/docker-compose.yml.

The file must define exactly two Docker networks:
  - uddokta_internal (driver: bridge) — all inter-service communication
  - uddokta_external (driver: bridge) — only Caddy connects here

Create a named volume declaration block with the following volumes:
  postgres_n8n_data, n8n_data, chatwoot_storage, chatwoot_pg_data,
  chatwoot_redis_data, evolution_instances, typebot_pg_data,
  twenty_pg_data, twenty_redis_data, listmonk_pg_data, dify_pg_data,
  dify_redis_data, caddy_data, caddy_certs, r2_staging, mautic_data,
  mautic_pg_data, mautic_redis_data

Do not define any services yet. Only networks and volumes.
Commit as: infra/docker-compose.yml
```

```
AGENT TASK 1.2.2
Create infra/.env.example with the following variable blocks.
Every value must be a placeholder string in the format {DESCRIBE_ME}.
Do not put real values in this file.

Block 1 — Supabase
  NEXT_PUBLIC_SUPABASE_URL
  NEXT_PUBLIC_SUPABASE_ANON_KEY
  SUPABASE_SERVICE_ROLE_KEY
  SUPABASE_JWT_SECRET
  SUPABASE_DB_HOST
  SUPABASE_DB_PASSWORD

Block 2 — n8n
  N8N_ADMIN_USER
  N8N_ADMIN_PASSWORD
  N8N_ENCRYPTION_KEY (must be 32-byte hex, generate with: openssl rand -hex 32)
  N8N_DB_USER
  N8N_DB_PASSWORD

Block 3 — Chatwoot
  CHATWOOT_SECRET_KEY (generate with: openssl rand -hex 64)
  CHATWOOT_DB_USER
  CHATWOOT_DB_PASSWORD

Block 4 — Evolution API
  EVOLUTION_API_KEY (generate with: openssl rand -hex 32)
  EVOLUTION_DB_URI (postgresql://user:pass@host/evolution)

Block 5 — Typebot
  TYPEBOT_DB_USER
  TYPEBOT_DB_PASSWORD
  TYPEBOT_SECRET (generate with: openssl rand -hex 32)
  TYPEBOT_ADMIN_EMAIL

Block 6 — Twenty CRM
  TWENTY_DB_USER
  TWENTY_DB_PASSWORD
  TWENTY_APP_SECRET (generate with: openssl rand -hex 32)

Block 7 — Listmonk
  LISTMONK_DB_USER
  LISTMONK_DB_PASSWORD

Block 8 — Mautic
  MAUTIC_DB_USER
  MAUTIC_DB_PASSWORD
  MAUTIC_SECRET_KEY

Block 9 — Dify
  DIFY_DB_USER
  DIFY_DB_PASSWORD
  DIFY_SECRET_KEY (generate with: openssl rand -hex 32)
  DIFY_OPENAI_API_KEY
  DIFY_ANTHROPIC_API_KEY

Block 10 — Cloudflare R2
  R2_ENDPOINT (https://{ACCOUNT_ID}.r2.cloudflarestorage.com)
  R2_ACCESS_KEY
  R2_SECRET_KEY
  R2_BUCKET_TYPEBOT=typebot-assets
  R2_BUCKET_VIDEOS=uddokta-videos
  R2_BUCKET_BACKUPS=uddokta-backups
  R2_BUCKET_DIFY=dify-assets

Block 11 — External APIs
  META_APP_SECRET
  META_VERIFY_TOKEN (for webhook verification endpoint)
  APIFY_API_TOKEN
  PEXELS_API_KEY
  SMTP_HOST
  SMTP_PORT
  SMTP_USER
  SMTP_PASSWORD
  SMTP_FROM_EMAIL

Block 12 — VPS / Caddy
  VPS_PUBLIC_IP
  VPS_DOMAIN=vps.uddokta.com
  VERCEL_DOMAIN=app.uddokta.com

Commit as: infra/.env.example
```

```
AGENT TASK 1.2.3
Copy infra/.env.example to infra/.env on the VPS.
SSH into the VPS and populate every variable with real values.
Run: cd /opt/uddokta/infra && cp .env.example .env && nano .env
Verify no placeholder {DESCRIBE_ME} strings remain:
  grep -c "{" .env
The output must be 0. If not, fix remaining gaps before proceeding.
```

### 1.3 Caddy Reverse Proxy — Deploy First

> Caddy must be the first running container because it manages TLS certificates. All other services are proxied through it. Deploy Caddy standalone before adding any other service.

```
AGENT TASK 1.3.1
Create infra/caddy/Caddyfile with the following routing rules.
Replace {VPS_DOMAIN} with the actual domain from .env.

{VPS_DOMAIN} {
  # n8n webhook router (public webhooks must be accessible)
  handle /n8n/* {
    uri strip_prefix /n8n
    reverse_proxy n8n:5678
  }
  # Evolution API (Meta webhook target + internal API)
  handle /evolution/* {
    uri strip_prefix /evolution
    reverse_proxy evolution:8080
  }
  # Chatwoot (internal, restrict to admin IPs in production)
  handle /chatwoot/* {
    uri strip_prefix /chatwoot
    reverse_proxy chatwoot_app:3000
  }
  # Typebot viewer (public — embed on landing page)
  handle /typebot/* {
    uri strip_prefix /typebot
    reverse_proxy typebot_viewer:3000
  }
  # Typebot builder (admin-only, restrict IP in production)
  handle /typebot-admin/* {
    uri strip_prefix /typebot-admin
    reverse_proxy typebot_builder:3000
  }
  # Twenty CRM (admin-only)
  handle /twenty/* {
    uri strip_prefix /twenty
    reverse_proxy twenty:3000
  }
  # Dify (AI agent builder and API)
  handle /dify/* {
    uri strip_prefix /dify
    reverse_proxy dify_web:3000
  }
  handle /dify-api/* {
    uri strip_prefix /dify-api
    reverse_proxy dify_api:5001
  }
  # Listmonk email (admin-only)
  handle /listmonk/* {
    uri strip_prefix /listmonk
    reverse_proxy listmonk:9000
  }
  # Mautic email (admin-only)
  handle /mautic/* {
    uri strip_prefix /mautic
    reverse_proxy mautic:80
  }
  # MPT video generation wrapper
  handle /mpt/* {
    uri strip_prefix /mpt
    reverse_proxy mpt_wrapper:8502
  }
  # Health check endpoint (unauthenticated)
  handle /health {
    respond "OK" 200
  }
}
```

```
AGENT TASK 1.3.2
Add only the Caddy service to infra/docker-compose.yml services block.

services:
  caddy:
    image: caddy:2-alpine
    container_name: uddokta_caddy
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_certs:/config
    networks:
      - uddokta_external
      - uddokta_internal
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:80"]
      interval: 30s
      timeout: 10s
      retries: 3

Deploy with:
  cd /opt/uddokta/infra
  docker compose up -d caddy
  docker compose logs caddy --follow

Verify Caddy is issuing TLS certificate for {VPS_DOMAIN}.
The log must show: certificate obtained successfully
Do not proceed until HTTPS is confirmed working.
```

### 1.4 Supabase Project Initialization

```
AGENT TASK 1.4.1
On the local developer machine, install the Supabase CLI:
  npm install -g supabase

Create a new Supabase project at supabase.com with:
  - Project name: uddokta-production
  - Region: Southeast Asia (Singapore) — closest to Bangladesh
  - Database password: store in infra/.env as SUPABASE_DB_PASSWORD

Link the local repo to the remote project:
  cd uddokta
  supabase login
  supabase link --project-ref {YOUR_PROJECT_REF}

Confirm connection:
  supabase db remote commit
```

```
AGENT TASK 1.4.2
Create the following migration file at
infra/supabase/migrations/0001_core_schema.sql

Write the full SQL for all tables exactly as specified in the TRD
Section 3.1, in this exact creation order (respect foreign key deps):

1. public.tenants
2. public.profiles
3. public.contacts (with UNIQUE constraint on tenant_id + phone)
4. public.conversations
5. public.messages (with UNIQUE INDEX on platform_msg_id WHERE NOT NULL)
6. public.orders
7. public.audit_logs
8. public.message_retry_queue (with INDEX on next_retry_at WHERE attempt_count < max_attempts)

After each CREATE TABLE, add:
  ALTER TABLE public.{table_name} ENABLE ROW LEVEL SECURITY;

Do not add RLS policies yet — that is the next migration.
Run: supabase db push
Verify zero errors in the output.
```

```
AGENT TASK 1.4.3
Create infra/supabase/migrations/0002_rls_policies.sql

Write the full SQL for:
1. The helper function public.current_tenant_id() as specified in TRD 3.2
2. The helper function public.current_user_role() as specified in TRD 3.2
3. All RLS policies for every table as specified in TRD 3.2

Additionally, add these policies not covered in TRD:
  -- Profiles: users can read all profiles in their tenant
  CREATE POLICY "profiles_tenant_read" ON public.profiles
    FOR SELECT USING (tenant_id = public.current_tenant_id());
  -- Profiles: users can only update their own profile
  CREATE POLICY "profiles_self_update" ON public.profiles
    FOR UPDATE USING (id = auth.uid());
  -- message_retry_queue: service_role only (no user policies)
  -- (already protected by service_role key usage in n8n)

Run: supabase db push
Verify zero errors.
```

```
AGENT TASK 1.4.4
Create infra/supabase/migrations/0003_rls_verification_test.sql

Write a SQL script that TESTS RLS enforcement. It must:
1. Create two test tenants: 'tenant-a' and 'tenant-b'
2. Create a test user bound to tenant-a via app_metadata
3. Attempt to SELECT contacts WHERE tenant_id = tenant-b
4. Assert the result is 0 rows (RLS blocked it)
5. Clean up all test data

Run this script against the remote DB using:
  supabase db reset --db-url {SUPABASE_DB_URL}

This test MUST return 0 rows for the cross-tenant query.
If it returns any rows, the RLS policy has a defect. Stop and fix.
```

```
AGENT TASK 1.4.5
Create the Supabase Edge Function for tenant provisioning.

File: infra/supabase/functions/provision-tenant/index.ts

The function must:
1. Accept a POST request with body:
   { tenant_slug, tenant_name, owner_name, owner_email, owner_phone }
2. Verify the request carries the internal service secret header:
   Authorization: Bearer {SUPABASE_SERVICE_ROLE_KEY}
   Reject with 401 if missing or mismatched.
3. Execute these steps in order with rollback on any failure:
   a. INSERT into public.tenants → capture new tenant UUID
   b. Create Supabase Auth user via admin.createUser()
   c. Call admin.updateUserById() to set app_metadata:
      { tenant_id: {uuid}, role: "owner" }
   d. Return { tenant_id, user_id, status: "provisioned" }
4. Wrap all DB operations in a try/catch.
   On any step failure after step (a), DELETE the tenant row
   and DELETE the auth user before re-throwing the error.

Deploy with:
  supabase functions deploy provision-tenant --no-verify-jwt
```

### 1.5 Phase 1 Acceptance Checks

```
ACCEPTANCE CHECK — DO NOT PROCEED TO PHASE 2 UNTIL ALL PASS

[ ] Oracle VPS is accessible via SSH (key-only auth)
[ ] HTTPS is working at https://{VPS_DOMAIN}/health → returns "OK"
[ ] Supabase migrations 0001, 0002, 0003 applied with zero errors
[ ] RLS cross-tenant isolation test returns 0 rows
[ ] provision-tenant Edge Function deployed and returns 200 on valid POST
[ ] infra/.env has no remaining {DESCRIBE_ME} placeholder values
[ ] infra/.env is confirmed in .gitignore (run: git status — must not appear)
[ ] Docker networks uddokta_internal and uddokta_external exist on VPS:
    docker network ls | grep uddokta
[ ] All named volumes declared in docker-compose.yml:
    docker volume ls | grep uddokta (will be empty until services start — just confirm Compose validates cleanly)
    docker compose config --quiet
```

---

## Phase 2: Core Mobile PWA & Manual Order Board

> **Gate condition:** Phase 3 cannot start until the Next.js app is deployed to Vercel, a real seller can log in, create a manual order, and open a public receipt link — entirely without any webhook or Docker service running.

### 2.1 Next.js Monorepo Bootstrap

```
AGENT TASK 2.1.1
Initialize the Next.js application inside apps/web.

Run:
  cd apps/web
  npx create-next-app@latest . \
    --typescript \
    --tailwind \
    --eslint \
    --app \
    --src-dir \
    --import-alias "@/*" \
    --no-turbopack

Then install the exact dependency set:
  npm install @supabase/supabase-js @supabase/ssr
  npm install next-pwa
  npm install @radix-ui/react-icons lucide-react
  npm install class-variance-authority clsx tailwind-merge
  npm install zustand
  npm install react-hook-form @hookform/resolvers zod
  npm install date-fns
  npm install @tanstack/react-query
  npm install next-themes

Install shadcn/ui CLI and initialize:
  npx shadcn@latest init
  When prompted:
    - Style: Default
    - Base color: Neutral
    - CSS variables: Yes

Install the exact shadcn components needed for Phase 2:
  npx shadcn@latest add button card badge input label
  npx shadcn@latest add select textarea dialog sheet
  npx shadcn@latest add dropdown-menu avatar skeleton
  npx shadcn@latest add toast sonner tabs separator
  npx shadcn@latest add form scroll-area

Commit as: feat: bootstrap Next.js PWA with shadcn/ui
```

```
AGENT TASK 2.1.2
Create apps/web/next.config.js with the following configuration:

const withPWA = require('next-pwa')({
  dest: 'public',
  disable: process.env.NODE_ENV === 'development',
  register: true,
  skipWaiting: true,
  runtimeCaching: [
    {
      urlPattern: /^https:\/\/.*\.supabase\.co\/rest\/.*/,
      handler: 'NetworkFirst',
      options: {
        cacheName: 'supabase-api',
        expiration: { maxEntries: 50, maxAgeSeconds: 300 }
      }
    }
  ]
});

module.exports = withPWA({
  reactStrictMode: true,
  images: { domains: ['supabase.co', '{VPS_DOMAIN}'] },
  experimental: { serverActions: { bodySizeLimit: '2mb' } }
});

Create apps/web/public/manifest.json:
{
  "name": "Uddokta",
  "short_name": "Uddokta",
  "description": "Mobile sales and order control for Bangladeshi sellers",
  "start_url": "/app/home",
  "display": "standalone",
  "background_color": "#0f172a",
  "theme_color": "#0f172a",
  "orientation": "portrait",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

```
AGENT TASK 2.1.3
Create the Supabase client utilities at apps/web/src/lib/supabase/:

File: apps/web/src/lib/supabase/client.ts
  — createBrowserClient() using @supabase/ssr for client components
  — Must read NEXT_PUBLIC_SUPABASE_URL and NEXT_PUBLIC_SUPABASE_ANON_KEY

File: apps/web/src/lib/supabase/server.ts
  — createServerClient() using @supabase/ssr for Server Components
  — Must use cookies() from next/headers

File: apps/web/src/lib/supabase/middleware.ts
  — createServerClient() for middleware session refresh
  — Export an updateSession() function

File: apps/web/src/middleware.ts
  — Import updateSession from the middleware utility
  — Apply to all routes matching: /app/(.*), /admin/(.*)
  — Redirect unauthenticated users to /login
  — Redirect authenticated users away from /login to /app/home

Commit as: feat: Supabase SSR client utilities and auth middleware
```

### 2.2 App Shell & Mobile Navigation

```
AGENT TASK 2.2.1
Create the application shell layout at
apps/web/src/app/(app)/layout.tsx

Requirements:
- This is a React Server Component
- It wraps all authenticated /app/* routes
- It reads the session using the server Supabase client
- It fetches the tenant data for the authenticated user
- It provides tenant context via a client-side TenantProvider
- It renders a fixed BottomNavigation component at the bottom
- It renders a mobile-optimized header with business name + avatar
- The main content area must have padding-bottom: 80px to clear the nav

The BottomNavigation component must:
- Be a Client Component (uses usePathname)
- Render exactly 5 tabs: Home, Inbox, Orders, Customers, Reports
- Use lucide-react icons: LayoutDashboard, Inbox, ShoppingBag, Users, BarChart2
- Show active state with brand color on the active tab
- Fixed position at bottom, full width
- z-index: 50
- Background: white/dark-mode aware
- Each tab is a Next.js Link component
```

```
AGENT TASK 2.2.2
Create the TenantProvider and global state at
apps/web/src/providers/TenantProvider.tsx

This must be a Client Component that:
- Accepts initialTenant and initialProfile as props (passed from RSC layout)
- Stores tenant and profile in Zustand store
- Exports useTenant() hook: { tenant, profile, isOwner, isAdmin }
- The isOwner check: profile.role === 'owner'
- The isAdmin check: profile.role is checked against Supabase JWT app_metadata

Create apps/web/src/stores/tenant.store.ts using Zustand:
  - State: { tenant: Tenant | null, profile: Profile | null }
  - Actions: setTenant, setProfile, clearTenant

All subsequent components read from this store.
Never pass tenant_id as props down component trees.
All DB queries must use the tenant_id from this store.
```

### 2.3 Auth Flow

```
AGENT TASK 2.3.1
Build the authentication screens.

File: apps/web/src/app/(auth)/login/page.tsx
- Mobile-optimized login page
- Accepts phone number (Bangladeshi +880 format) or email
- Uses Supabase OTP (magic link to email or phone OTP)
- Form validation with react-hook-form + zod
- On successful OTP verification: redirect to /app/home
- Show "Setup Pending" state if tenant has no workspace yet
- Copy must be in English (Bangla support added in later phase)
- Do not use any Next.js form action — use client-side Supabase auth

File: apps/web/src/app/(auth)/setup-pending/page.tsx
- Simple page shown when auth.user exists but public.profiles has no row
- Copy: "Your workspace is being prepared. You will receive a message when ready."
- Actions: "Contact Support" (opens WhatsApp link), "Re-check Status" button

File: apps/web/src/app/auth/callback/route.ts
- Next.js Route Handler for Supabase auth callback
- Exchanges code for session using server client
- Redirects to /app/home on success
```

### 2.4 Products, Customers, and Orders CRUD

```
AGENT TASK 2.4.1
Build the Products module.

Create the following server actions at
apps/web/src/app/(app)/products/actions.ts:

Action: getProducts(tenantId: string)
  - SELECT * FROM products WHERE tenant_id = $1 ORDER BY created_at DESC
  - Uses server Supabase client (RLS will enforce tenant isolation)

Action: createProduct(data: ProductFormData)
  - INSERT into products with tenant_id from server-side session
  - Never accept tenant_id from the client form body

Action: updateProduct(id: string, data: Partial<ProductFormData>)
  - UPDATE products WHERE id = $1 AND tenant_id = (from session)

Action: deleteProduct(id: string)
  - DELETE FROM products WHERE id = $1 AND tenant_id = (from session)

Create the page at apps/web/src/app/(app)/products/page.tsx:
- RSC: fetch products server-side
- Render mobile product card list (image, name, price, stock badge)
- FAB button: "+ Add Product" → opens Sheet/drawer with ProductForm
- Empty state: "Add products to send faster replies and create orders quickly."

ProductForm fields (in a Sheet component):
  - name (text, required)
  - price (number, required, BDT currency display)
  - sale_price (number, optional)
  - stock_status (select: In Stock / Out of Stock / Limited)
  - variants (dynamic field array: label + price modifier)
  - delivery_note (textarea)
  - quick_reply_template (textarea, pre-filled template)
  - image_url (text, optional for Phase 2)

Add migration: infra/supabase/migrations/0004_products_table.sql
  CREATE TABLE public.products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES public.tenants(id),
    name TEXT NOT NULL,
    price NUMERIC(10,2) NOT NULL,
    sale_price NUMERIC(10,2),
    stock_status TEXT NOT NULL DEFAULT 'in_stock',
    variants JSONB DEFAULT '[]',
    delivery_note TEXT,
    quick_reply_template TEXT,
    image_url TEXT,
    created_at TIMESTAMPTZ DEFAULT now()
  );
  ALTER TABLE public.products ENABLE ROW LEVEL SECURITY;
  CREATE POLICY "products_tenant_isolation" ON public.products
    FOR ALL USING (tenant_id = public.current_tenant_id());
Run: supabase db push
```

```
AGENT TASK 2.4.2
Build the Customers module.

Add migration: infra/supabase/migrations/0005_customers_extended.sql
  The contacts table from migration 0001 is the customers table.
  Add the following columns to public.contacts if not present:
  ALTER TABLE public.contacts ADD COLUMN IF NOT EXISTS tags TEXT[] DEFAULT '{}';
  ALTER TABLE public.contacts ADD COLUMN IF NOT EXISTS total_orders INT DEFAULT 0;
  ALTER TABLE public.contacts ADD COLUMN IF NOT EXISTS total_spent NUMERIC(10,2) DEFAULT 0;
  ALTER TABLE public.contacts ADD COLUMN IF NOT EXISTS last_order_at TIMESTAMPTZ;
  ALTER TABLE public.contacts ADD COLUMN IF NOT EXISTS risk_notes TEXT;
  ALTER TABLE public.contacts ADD COLUMN IF NOT EXISTS area TEXT;
  ALTER TABLE public.contacts ADD COLUMN IF NOT EXISTS city TEXT;
  ALTER TABLE public.contacts ADD COLUMN IF NOT EXISTS full_address TEXT;
Run: supabase db push

Create apps/web/src/app/(app)/customers/page.tsx:
- RSC: paginated customer list (20 per page)
- Search bar (client-side filter on name/phone)
- Customer card: name, phone, order count badge, last order date, city
- Tap → opens /app/customers/[id]
- FAB: "+ Add Customer"

Create apps/web/src/app/(app)/customers/[id]/page.tsx:
- Customer profile detail
- Sections: Profile, Orders (list of past orders), Notes
- Action buttons: "Create Order", "Send Message" (WhatsApp deep link), "Add Note"
- Risk note field (visible to owner/manager only)
- Order history fetched from orders table WHERE contact_id = $1
```

```
AGENT TASK 2.4.3
Build the Orders module — this is the most critical Phase 2 module.

Add migration: infra/supabase/migrations/0006_orders_extended.sql
  ALTER TABLE public.orders ADD COLUMN IF NOT EXISTS order_number TEXT UNIQUE;
  ALTER TABLE public.orders ADD COLUMN IF NOT EXISTS product_lines JSONB DEFAULT '[]';
  ALTER TABLE public.orders ADD COLUMN IF NOT EXISTS payment_method TEXT DEFAULT 'cod';
  ALTER TABLE public.orders ADD COLUMN IF NOT EXISTS delivery_area TEXT;
  ALTER TABLE public.orders ADD COLUMN IF NOT EXISTS delivery_city TEXT;
  ALTER TABLE public.orders ADD COLUMN IF NOT EXISTS delivery_address TEXT;
  ALTER TABLE public.orders ADD COLUMN IF NOT EXISTS courier TEXT;
  ALTER TABLE public.orders ADD COLUMN IF NOT EXISTS tracking_number TEXT;
  ALTER TABLE public.orders ADD COLUMN IF NOT EXISTS discount NUMERIC(10,2) DEFAULT 0;
  ALTER TABLE public.orders ADD COLUMN IF NOT EXISTS delivery_charge NUMERIC(10,2) DEFAULT 0;
  ALTER TABLE public.orders ADD COLUMN IF NOT EXISTS internal_notes TEXT;
  ALTER TABLE public.orders ADD COLUMN IF NOT EXISTS receipt_token TEXT UNIQUE DEFAULT gen_random_uuid()::TEXT;
  ALTER TABLE public.orders ADD COLUMN IF NOT EXISTS conversation_id UUID REFERENCES public.conversations(id);

  -- Auto-generate order number on INSERT
  CREATE OR REPLACE FUNCTION generate_order_number()
  RETURNS TRIGGER AS $$
  BEGIN
    NEW.order_number := 'UDD-' || LPAD(nextval('order_number_seq')::TEXT, 4, '0');
    RETURN NEW;
  END;
  $$ LANGUAGE plpgsql;

  CREATE SEQUENCE IF NOT EXISTS order_number_seq START 1000;

  CREATE TRIGGER set_order_number
    BEFORE INSERT ON public.orders
    FOR EACH ROW EXECUTE FUNCTION generate_order_number();
Run: supabase db push

Create apps/web/src/app/(app)/orders/page.tsx:
- Mobile Kanban board layout
- Tab row at top: All | New | Confirmed | Pending | Shipped | Delivered | Returned
- Order cards: order number, customer name, product summary, amount, status badge, time
- Tap card → opens /app/orders/[id]
- FAB: "+ New Order"
- Empty state per tab

Create apps/web/src/app/(app)/orders/new/page.tsx:
This is the most important form in the product. It must:
- Section 1 "Customer": search existing customers by name/phone,
  or create new inline. Auto-fill address if existing customer found.
- Section 2 "Products": product search from tenant product list,
  select variant, set quantity, show line total. Support multiple lines.
  Show delivery charge field. Show discount field. Show grand total.
- Section 3 "Payment": radio buttons for COD / bKash / Nagad / Bank / Paid
- Section 4 "Delivery": delivery address (pre-fill from customer),
  city, area, courier dropdown (Steadfast / Pathao / REDX / Manual)
- Section 5 "Review": summary card showing all data before submit
- Submit buttons: "Create Order" | "Create + Send Receipt" | "Save Draft"

On successful creation:
- Show success toast with order number
- Offer "Send Receipt" button
- Navigate back to orders list

Create apps/web/src/app/(app)/orders/[id]/page.tsx:
- Full order detail view
- Order header: number, status badge, created at
- Customer section: tap to open customer profile
- Items section: product lines with prices
- Payment section: method, status (paid/unpaid/proof pending)
- Delivery section: address, courier, tracking
- Timeline section: status change history from audit_logs
- Action bar: "Update Status" | "Send Receipt" | "Add Note" | "Call Customer"
- Status update: bottom sheet with allowed transitions per TRD Section 2
```

### 2.5 Public Receipt Page

```
AGENT TASK 2.5.1
Build the public receipt route. This is a critical trust feature and
must work without authentication.

Create apps/web/src/app/public/receipt/[token]/page.tsx:
- This is a public RSC route (no auth required)
- Fetch order by receipt_token (NOT by order ID — token is the safe public key)
- Query must use service role? NO — create a Supabase policy:

Add migration: infra/supabase/migrations/0007_public_receipt_policy.sql
  -- Allow anonymous read of orders via receipt_token
  -- Only expose safe fields, not full address or internal notes
  CREATE POLICY "public_receipt_read" ON public.orders
    FOR SELECT
    USING (receipt_token = current_setting('app.receipt_token', true));
  -- Note: this approach requires passing the token as a setting.
  -- Simpler alternative: use a Supabase Edge Function that reads
  -- the token and returns only safe fields. Use this approach.

REVISED APPROACH — Use Edge Function for public receipt:
  File: infra/supabase/functions/public-receipt/index.ts
  - Accept GET ?token={receipt_token}
  - Query orders + contacts + tenants using service role (server-side only)
  - Return ONLY safe public fields:
    { order_number, created_at, product_lines, amount, currency,
      delivery_charge, discount, payment_method, payment_status,
      delivery_city, status, tenant_name, tenant_logo, tenant_support_phone,
      tenant_delivery_policy, tenant_return_policy,
      customer_first_name (first name only, no surname),
      delivery_area (city only, no street address) }
  - Do NOT return: full_address, phone, internal_notes, staff data, receipt_token
  - Set CORS headers to allow any origin (public page)
  Deploy: supabase functions deploy public-receipt

The Next.js page fetches from this Edge Function.
Design the receipt page as a mobile-optimized public page:
  - Business logo + name at top
  - Order number + status badge
  - Product list table
  - Payment summary
  - Delivery city
  - Policy box (delivery + return)
  - Support contact button
  - "Powered by Uddokta" footer badge
  - No seller dashboard chrome, no navigation
```

### 2.6 Home Dashboard

```
AGENT TASK 2.6.1
Build the Home dashboard at apps/web/src/app/(app)/home/page.tsx

This is a RSC that:
1. Fetches summary stats server-side in parallel using Promise.all:
   - COUNT orders WHERE status = 'new' AND tenant_id = $1 → new_orders
   - COUNT orders WHERE status IN ('payment_pending', 'confirmed')
     AND created_at >= NOW() - INTERVAL '24 hours' → pending_count
   - COUNT orders WHERE status = 'returned' 
     AND updated_at >= NOW() - INTERVAL '7 days' → returns_7d
   - Total revenue: SUM amount WHERE status = 'delivered'
     AND created_at >= date_trunc('month', NOW()) → monthly_revenue
   (All these are safe Supabase server-side queries — RLS enforces tenant scope)

2. Renders the following card grid (2 columns on mobile):
   - New Messages (hardcoded 0 in Phase 2, wired in Phase 4)
   - Pending Orders → taps to /app/orders?status=new
   - Payment Pending → taps to /app/orders?status=payment_pending
   - Returned (7d) → taps to /app/orders?status=returned
   - Monthly Sales (BDT display) → taps to /app/reports
   - Follow-ups Due (hardcoded 0 in Phase 2, wired in Phase 4)

3. Shows business name + greeting at top
4. Shows a "Today's Quick Actions" row: New Order | Add Customer | View Orders

5. Empty state (zero orders): shows the aha-moment prompt:
   "Create your first order or add a test customer to get started."
   Actions: "Create Test Order" | "Add Product" buttons
```

### 2.7 Admin Workspace Management

```
AGENT TASK 2.7.1
Build the Admin console. Admin users have role = 'uddokta_admin'
stored in Supabase auth app_metadata.

Create apps/web/src/app/(admin)/layout.tsx:
- Verify current user has app_metadata.role === 'uddokta_admin'
- Redirect non-admin users to /app/home
- Admin-specific sidebar navigation (desktop-first for admin, unlike seller UI)

Create apps/web/src/app/(admin)/workspaces/page.tsx:
- List all tenants with: slug, name, plan, created_at, status
- Search by name
- "Create Workspace" button

Create apps/web/src/app/(admin)/workspaces/new/page.tsx:
- Form that calls the provision-tenant Edge Function
- Fields: business_name, tenant_slug, owner_name, owner_email, owner_phone, plan
- On success: navigate to /admin/workspaces/{new_id}/setup

Create apps/web/src/app/(admin)/workspaces/[id]/page.tsx:
- Tenant detail: status, plan, health indicators (all green/red icons)
- Action buttons: "Suspend Workspace", "Resend Invite", "View as Owner"
- Health indicators (Phase 2: all static placeholders, wired in Phase 4):
  Last inbound message, Last outbound message, n8n status, Channel status

Create apps/web/src/app/(admin)/workspaces/[id]/setup/page.tsx:
- Guided setup checklist wizard (10-step onboarding from App Flow §19.2)
- Each step marks a boolean in tenants.setup_checklist JSONB column
- Add column: ALTER TABLE public.tenants ADD COLUMN setup_checklist JSONB DEFAULT '{}';
- Steps: business_profile, products, faq, staff, channel, test_customer,
         test_order, receipt, scripts_approved, handover_sent
```

### 2.8 Vercel Deployment — First Deploy

```
AGENT TASK 2.8.1
Deploy the Next.js app to Vercel for the first time.

In the apps/web directory, create vercel.json:
{
  "framework": "nextjs",
  "regions": ["sin1"],
  "buildCommand": "npm run build",
  "outputDirectory": ".next"
}

Create apps/web/.env.local with all NEXT_PUBLIC_* variables
from infra/.env. This file is gitignored.

Deploy:
  npx vercel --cwd apps/web --prod

In the Vercel dashboard, add ALL environment variables from infra/.env
that the Next.js app needs (both NEXT_PUBLIC_* and server-side secrets).

After deployment, verify:
  curl https://app.uddokta.com/health

Confirm all Phase 2 routes are accessible:
  /login → loads login form
  /app/home → redirects to /login (auth guard working)
  /public/receipt/test-token → returns 404 gracefully (no order yet)
```

### 2.9 Phase 2 Acceptance Checks

```
ACCEPTANCE CHECK — DO NOT PROCEED TO PHASE 3 UNTIL ALL PASS

[ ] Login works via Supabase OTP on mobile browser
[ ] Auth middleware blocks /app/* for unauthenticated users
[ ] A test tenant can be provisioned via the admin console
[ ] Products can be created, listed, and edited within a tenant
[ ] Customers can be created and searched
[ ] An order can be created manually with 2+ product lines
[ ] Order number is auto-generated in format UDD-XXXX
[ ] Order status can be moved through valid transitions
[ ] Public receipt page renders at /public/receipt/{token} without login
[ ] Public receipt does NOT expose phone number, full address, or notes
[ ] Admin workspace list shows only tenants (no cross-tenant data leak)
[ ] RLS test: log in as Tenant A user, confirm cannot see Tenant B orders
    (test this manually by crafting a direct Supabase REST call with Tenant A JWT)
[ ] Home dashboard loads in under 3 seconds on a 4G simulated connection
[ ] PWA installs to Android home screen and shows correct app name/icon
[ ] Vercel deployment is live at https://app.uddokta.com
```

---

## Phase 3: Internal Operations & Lead Funnel

> **Gate condition:** Phase 4 cannot start until Typebot is capturing leads, n8n is receiving Typebot webhooks, and Twenty CRM shows incoming leads from the audit form.

### 3.1 Typebot Deployment

```
AGENT TASK 3.1.1
Add Typebot services to infra/docker-compose.yml.

Add service: typebot_db (postgres:16-alpine)
  - POSTGRES_DB: typebot
  - POSTGRES_USER / POSTGRES_PASSWORD from .env
  - Volume: typebot_pg_data
  - Network: uddokta_internal
  - Healthcheck: pg_isready

Add service: typebot_builder
  Image: baptistearno/typebot-builder:latest
  IMPORTANT: Verify this image has a linux/arm64 manifest before deploying.
  Run: docker manifest inspect baptistearno/typebot-builder:latest | grep arm64
  If no arm64 manifest found, build from source:
    git clone https://github.com/baptisteArno/typebot.io.git /opt/src/typebot
    docker build --platform linux/arm64 -t uddokta/typebot-builder:local \
      -f /opt/src/typebot/apps/builder/Dockerfile /opt/src/typebot
  Environment variables (all from .env):
    DATABASE_URL: postgresql://{USER}:{PASS}@typebot_db/typebot
    NEXTAUTH_URL: https://{VPS_DOMAIN}/typebot-admin
    NEXTAUTH_SECRET: {TYPEBOT_SECRET}
    ADMIN_EMAIL: {TYPEBOT_ADMIN_EMAIL}
    S3_ENDPOINT, S3_ACCESS_KEY, S3_SECRET_KEY, S3_BUCKET: {R2 values}
    ENCRYPTION_SECRET: {TYPEBOT_SECRET}
    DISABLE_SIGNUP: true
  Network: uddokta_internal

Add service: typebot_viewer
  Image: baptistearno/typebot-viewer:latest
  Same DATABASE_URL, NEXTAUTH_SECRET, S3 config as builder
  NEXTAUTH_URL: https://{VPS_DOMAIN}/typebot
  Network: uddokta_internal

Deploy:
  docker compose up -d typebot_db
  # Wait for healthcheck to pass:
  docker compose exec typebot_db pg_isready -U {TYPEBOT_DB_USER}
  docker compose up -d typebot_builder typebot_viewer
  docker compose logs typebot_builder --follow
  # Wait for: "ready on http://0.0.0.0:3000" in logs
```

```
AGENT TASK 3.1.2
Build the "Free Inbox Sales Leak Audit" Typebot flow.

Log into the Typebot builder at https://{VPS_DOMAIN}/typebot-admin
Create a new bot named "uddokta-audit-flow".

Build this exact conversation flow using Typebot's visual builder:

Step 1 — Welcome
  Text: "স্বাগতম! আমি আপনার ব্যবসার Inbox Sales Leak Score বের করব।
         মাত্র কয়েকটি প্রশ্নের উত্তর দিন।"
  Button: "শুরু করুন →"

Step 2 — Business Name
  Input type: Text
  Variable: {business_name}
  Question: "আপনার ব্যবসার নাম কী?"

Step 3 — Owner Name
  Input type: Text
  Variable: {owner_name}
  Question: "আপনার নাম কী?"

Step 4 — WhatsApp Number
  Input type: Phone
  Variable: {phone}
  Question: "আপনার WhatsApp নম্বর দিন (আমরা audit report পাঠাব)"
  Validation: Bangladesh +880 format

Step 5 — Facebook Page URL
  Input type: URL
  Variable: {facebook_url}
  Question: "আপনার Facebook Page-এর link দিন"

Step 6 — Business Category
  Input type: Button choice
  Variable: {category}
  Choices: Skincare/Cosmetics | Gadgets/Electronics | Clothing/Fashion |
           Food | Jewelry | Other

Step 7 — Daily Messages
  Input type: Number
  Variable: {daily_messages}
  Question: "প্রতিদিন কতগুলো customer message আসে (আনুমানিক)?"

Step 8 — Biggest Problem
  Input type: Button choice
  Variable: {biggest_pain}
  Choices: Late replies | Missed orders | Staff not following up |
           Fake COD orders | Customer trust issues | Staff management

Step 9 — Confirmation
  Text: "ধন্যবাদ {owner_name}! আমরা {business_name}-এর জন্য
         Inbox Leak Report তৈরি করছি। 30 মিনিটের মধ্যে WhatsApp-এ পাঠাব।"

Step 10 — Webhook (Typebot Webhook block)
  Method: POST
  URL: https://{VPS_DOMAIN}/n8n/webhook/typebot-lead
  Headers: X-Typebot-Secret: {TYPEBOT_WEBHOOK_SECRET from .env}
  Body: all collected variables as JSON

Publish the bot.
Copy the embed code.
```

```
AGENT TASK 3.1.3
Embed the Typebot audit flow on the Next.js marketing page.

Install: npm install @typebot-io/js --save (in apps/web)

Create apps/web/src/app/(marketing)/audit/page.tsx:
- Public marketing page (no auth required)
- Renders the Typebot bot embedded via @typebot-io/js Standard component
- Pass the published bot URL from the Typebot viewer instance
- The page has minimal chrome: Uddokta logo, headline, bot embed
- Headline: "আপনার ব্যবসার বিনামূল্যে Inbox Sales Leak Score জানুন"
- Mobile-first layout

Also create a floating "Get Free Audit" button on the main landing page
(apps/web/src/app/page.tsx) that links to /audit.
```

### 3.2 Twenty CRM Deployment

```
AGENT TASK 3.2.1
Add Twenty CRM to infra/docker-compose.yml.

IMPORTANT: Twenty CRM's default port is 3000, which conflicts with
Chatwoot (also 3000) and Typebot. Remap Twenty to internal port 3000
but give it a unique container name. Caddy routes by container name,
so there is no actual port conflict on the host.

Check ARM64 support:
  docker manifest inspect twentyhq/twenty:latest | grep arm64
If no ARM64 manifest: Twenty may need to be built from source.
  git clone https://github.com/twentyhq/twenty.git /opt/src/twenty
  docker build --platform linux/arm64 -t uddokta/twenty:local \
    --build-arg APP_TARGET=server /opt/src/twenty

Add service: twenty_db (postgres:16-alpine)
  - POSTGRES_DB: twenty
  - Volume: twenty_pg_data
  - Healthcheck: pg_isready

Add service: twenty_redis (redis:7-alpine)
  - Volume: twenty_redis_data

Add service: twenty
  Image: twentyhq/twenty:latest (or uddokta/twenty:local for ARM)
  Depends on: twenty_db, twenty_redis
  Environment:
    PG_DATABASE_URL: postgresql://{USER}:{PASS}@twenty_db/twenty
    REDIS_URL: redis://twenty_redis:6379
    SERVER_URL: https://{VPS_DOMAIN}/twenty
    APP_SECRET: {TWENTY_APP_SECRET}
    STORAGE_TYPE: s3
    STORAGE_S3_REGION: auto
    STORAGE_S3_NAME: {R2_BUCKET_DIFY}
    STORAGE_S3_ENDPOINT: {R2_ENDPOINT}
    STORAGE_S3_ACCESS_KEY_ID: {R2_ACCESS_KEY}
    STORAGE_S3_SECRET_ACCESS_KEY: {R2_SECRET_KEY}
  Network: uddokta_internal

Deploy:
  docker compose up -d twenty_db twenty_redis
  # Wait for DB healthcheck
  docker compose up -d twenty
  docker compose logs twenty --follow
  # Wait for: "Server is running" in logs

Access Twenty at https://{VPS_DOMAIN}/twenty
Create the initial admin account.
```

```
AGENT TASK 3.2.2
Configure Twenty CRM for the Uddokta lead pipeline.

Log into Twenty at https://{VPS_DOMAIN}/twenty.

Create the following custom objects:
1. Object: "Uddokta Lead"
   Fields: business_name (text), facebook_url (url), phone (phone),
   owner_name (text), category (select), daily_messages (number),
   biggest_pain (select), audit_status (select: pending/sent/demo_booked),
   source (select: typebot/manual/referral), lead_score (number)

2. Object: "Uddokta Workspace"
   Fields: tenant_slug (text), plan (select), monthly_revenue (currency),
   setup_status (select: pending/active/suspended), setup_date (date)

Create Pipeline: "Uddokta Sales Pipeline"
  Stages: New Lead → Audit Sent → Demo Scheduled → Demo Done →
          Payment Committed → Setup Started → Active Client → Churned

Generate a Twenty API token:
  Settings → API → Generate new token
  Store this token in infra/.env as TWENTY_API_TOKEN
```

### 3.3 n8n Deployment & Lead Routing

```
AGENT TASK 3.3.1
Add n8n and its PostgreSQL database to infra/docker-compose.yml.

Add service: postgres_n8n (postgres:16-alpine)
  - POSTGRES_DB: n8n
  - POSTGRES_USER / POSTGRES_PASSWORD from .env
  - Volume: postgres_n8n_data
  - Network: uddokta_internal
  - Healthcheck: pg_isready
  NOTE: This is a SEPARATE database from Supabase. n8n's internal schema
  must not pollute the application database.

Add service: n8n
  Image: n8nio/n8n:latest
  Depends on: postgres_n8n (condition: service_healthy)
  Environment:
    DB_TYPE: postgresdb
    DB_POSTGRESDB_HOST: postgres_n8n
    DB_POSTGRESDB_DATABASE: n8n
    DB_POSTGRESDB_USER: {N8N_DB_USER}
    DB_POSTGRESDB_PASSWORD: {N8N_DB_PASSWORD}
    N8N_ENCRYPTION_KEY: {N8N_ENCRYPTION_KEY}
    WEBHOOK_URL: https://{VPS_DOMAIN}/n8n/
    N8N_BASIC_AUTH_ACTIVE: "true"
    N8N_BASIC_AUTH_USER: {N8N_ADMIN_USER}
    N8N_BASIC_AUTH_PASSWORD: {N8N_ADMIN_PASSWORD}
    EXECUTIONS_DATA_SAVE_ON_ERROR: all
    EXECUTIONS_DATA_SAVE_ON_SUCCESS: none
    EXECUTIONS_DATA_MAX_AGE: "168"
    N8N_METRICS: "true"
    GENERIC_TIMEZONE: Asia/Dhaka
  Volume: n8n_data:/home/node/.n8n
  Network: uddokta_internal

Deploy:
  docker compose up -d postgres_n8n
  docker compose up -d n8n
  docker compose logs n8n --follow
  # Wait for: "Editor is now accessible via:" in logs
```

```
AGENT TASK 3.3.2
Build the "typebot-lead" n8n workflow.

Access n8n at https://{VPS_DOMAIN}/n8n

Create workflow: "Typebot Lead Intake"

Node 1 — Webhook trigger
  Path: /typebot-lead
  Method: POST
  Authentication: Header Auth
    Header name: X-Typebot-Secret
    Header value: {TYPEBOT_WEBHOOK_SECRET}
  IMPORTANT: If header is missing or wrong, immediately return 401.
  Add a Function node immediately after the trigger to validate:
    if ($input.headers['x-typebot-secret'] !== process.env.TYPEBOT_WEBHOOK_SECRET) {
      throw new Error('Unauthorized webhook');
    }

Node 2 — Normalize data (Function node)
  Extract: business_name, owner_name, phone, facebook_url,
  category, daily_messages, biggest_pain from $input.body
  Normalize phone to E.164 format (+880...)

Node 3 — Supabase: Check duplicate lead
  HTTP Request to Supabase REST API:
  GET {SUPABASE_URL}/rest/v1/leads?phone=eq.{phone}&select=id
  Headers: apikey: {SUPABASE_SERVICE_ROLE_KEY}
           Authorization: Bearer {SUPABASE_SERVICE_ROLE_KEY}
  Note: Create public.leads table first (migration 0008 below)

Node 4 — IF node: Is new lead?
  Condition: HTTP response body array length === 0

Node 5a (new lead) — Supabase: Insert lead
  POST {SUPABASE_URL}/rest/v1/leads
  Body: all lead data + tenant_id: null (this is a pre-tenant prospect)
  Note: The leads table is in the Uddokta admin schema, not tenant-scoped.

Node 5b (duplicate) — Set: Mark as duplicate update
  UPDATE existing lead with new timestamp (they re-submitted)

Node 6 — Twenty CRM: Create Person
  HTTP Request:
  POST https://{VPS_DOMAIN}/twenty/api/people
  Headers: Authorization: Bearer {TWENTY_API_TOKEN}
  Body: {
    name: { firstName: owner_name, lastName: "" },
    phones: { primaryPhoneNumber: phone },
    city: "",
    company: { name: business_name }
  }
  On error: continue (Twenty is not critical path)

Node 7 — Twenty CRM: Create Opportunity
  Create opportunity in "New Lead" stage with business_name
  Link to person created in Node 6

Node 8 — WhatsApp Acknowledgment (stub for Phase 2, wired in Phase 4)
  Set a flag: send_whatsapp_ack = true
  In Phase 2, log this intention to n8n execution log only.
  Actual message send happens in Phase 4 when Evolution API is deployed.

Node 9 — Respond 200
  Return: { received: true, lead_queued: true }

Add migration: infra/supabase/migrations/0008_leads_table.sql
  CREATE TABLE public.leads (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_name TEXT,
    owner_name TEXT,
    phone TEXT NOT NULL,
    facebook_url TEXT,
    category TEXT,
    daily_messages INT,
    biggest_pain TEXT,
    source TEXT DEFAULT 'typebot',
    audit_status TEXT DEFAULT 'pending',
    raw_apify_data JSONB,
    twenty_person_id TEXT,
    twenty_opportunity_id TEXT,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
  );
  -- No RLS — this is an admin-only table.
  -- Access via service_role key from n8n only.
Run: supabase db push

Export this workflow from n8n as JSON.
Save to: infra/n8n/workflows/typebot-lead-intake.json
Commit to git.
```

### 3.4 Phase 3 Acceptance Checks

```
ACCEPTANCE CHECK — DO NOT PROCEED TO PHASE 4 UNTIL ALL PASS

[ ] Typebot audit flow is accessible at https://{VPS_DOMAIN}/typebot/uddokta-audit-flow
[ ] Completing the Typebot form fires the webhook to n8n
[ ] n8n receives the webhook and creates a row in public.leads
[ ] n8n creates a Person and Opportunity in Twenty CRM
[ ] Twenty CRM shows the lead in the "New Lead" pipeline stage
[ ] Duplicate submission (same phone) does not create a duplicate lead
[ ] n8n execution log shows no errors for a complete typebot-lead run
[ ] Typebot workflow JSON is committed to infra/n8n/workflows/
[ ] Twenty CRM is accessible at https://{VPS_DOMAIN}/twenty
[ ] The /audit page on the Next.js app embeds and runs the Typebot flow
```

---

## Phase 4: Open-Source Engine Orchestration

> **Gate condition:** Phase 5 cannot start until inbound WhatsApp messages flow end-to-end from Evolution API → n8n → Supabase → Realtime → Next.js dashboard, and outbound messages send successfully.

### 4.1 Chatwoot Deployment

```
AGENT TASK 4.1.1
Add Chatwoot and its dependencies to infra/docker-compose.yml.

Chatwoot requires its own PostgreSQL, Redis, and a Sidekiq background
worker. All must be on uddokta_internal network.

Add service: chatwoot_pg (postgres:16-alpine)
  - POSTGRES_DB: chatwoot
  - Volume: chatwoot_pg_data
  - Healthcheck: pg_isready

Add service: chatwoot_redis (redis:7-alpine)
  - Volume: chatwoot_redis_data
  NOTE: This Redis is SHARED with Dify in Phase 5.
  Reference it from Dify as chatwoot_redis (same container).

Add service: chatwoot_app
  Image: chatwoot/chatwoot:latest
  Check ARM64: docker manifest inspect chatwoot/chatwoot:latest | grep arm64
  Depends on: chatwoot_pg (healthy), chatwoot_redis
  Environment:
    RAILS_ENV: production
    SECRET_KEY_BASE: {CHATWOOT_SECRET_KEY}
    FRONTEND_URL: https://{VPS_DOMAIN}/chatwoot
    DEFAULT_LOCALE: en
    FORCE_SSL: false (Caddy handles TLS termination)
    POSTGRES_HOST: chatwoot_pg
    POSTGRES_DATABASE: chatwoot
    POSTGRES_USERNAME: {CHATWOOT_DB_USER}
    POSTGRES_PASSWORD: {CHATWOOT_DB_PASSWORD}
    REDIS_URL: redis://chatwoot_redis:6379
    ACTIVE_STORAGE_SERVICE: local
    RAILS_LOG_TO_STDOUT: true
    INSTALLATION_ENV: self_hosted
  Volume: chatwoot_storage:/app/storage
  Network: uddokta_internal
  Command: bundle exec rails s -p 3000 -b 0.0.0.0

Add service: chatwoot_sidekiq
  Same image and environment as chatwoot_app
  Command: bundle exec sidekiq -C config/sidekiq.yml
  No ports exposed.

Deploy sequence:
  docker compose up -d chatwoot_pg chatwoot_redis
  # Wait for pg healthcheck
  # Run DB migrations (one-time):
  docker compose run --rm chatwoot_app bundle exec rails db:chatwoot_prepare
  docker compose up -d chatwoot_app chatwoot_sidekiq
  docker compose logs chatwoot_app --follow
  # Wait for: "Listening on tcp://0.0.0.0:3000"
```

```
AGENT TASK 4.1.2
Configure Chatwoot for headless operation.

Access Chatwoot at https://{VPS_DOMAIN}/chatwoot
Complete initial setup wizard to create the super admin account.
Store credentials in infra/secrets/chatwoot-admin.txt (gitignored).

For each production tenant, the provision-tenant Edge Function
(created in Phase 1) must be UPDATED to also call the Chatwoot API:

Update infra/supabase/functions/provision-tenant/index.ts:

Add Step 4 after the Supabase user creation:
  // Create Chatwoot Account for this tenant
  const chatwootAccountRes = await fetch(
    `${CHATWOOT_BASE_URL}/auth/sign_in`,  // first get super admin token
    ...
  );
  // Then create account:
  const chatwootAccount = await fetch(
    `${CHATWOOT_BASE_URL}/api/v1/profile`,
    { method: 'POST', ... }
  );
  // Store chatwoot_account_id and chatwoot_api_token in tenants table

Add columns to tenants table:
  infra/supabase/migrations/0009_tenant_chatwoot.sql
  ALTER TABLE public.tenants ADD COLUMN chatwoot_account_id BIGINT;
  ALTER TABLE public.tenants ADD COLUMN chatwoot_api_token TEXT;
  -- Encrypted at rest via pgsodium. For now store as text, encrypt in Phase 6.
Run: supabase db push

Redeploy Edge Function: supabase functions deploy provision-tenant
```

### 4.2 Evolution API Deployment

```
AGENT TASK 4.2.1
Add Evolution API to infra/docker-compose.yml.

Check ARM64: docker manifest inspect atendai/evolution-api:latest | grep arm64
If no ARM64 manifest is found:
  git clone https://github.com/evolution-foundation/evolution-api.git /opt/src/evolution
  docker build --platform linux/arm64 -t uddokta/evolution-api:local /opt/src/evolution
  Use uddokta/evolution-api:local as the image.

Add service: evolution
  Image: atendai/evolution-api:latest (or local ARM build)
  Environment:
    SERVER_URL: https://{VPS_DOMAIN}/evolution
    AUTHENTICATION_TYPE: apikey
    AUTHENTICATION_API_KEY: {EVOLUTION_API_KEY}
    DATABASE_ENABLED: true
    DATABASE_PROVIDER: postgresql
    DATABASE_CONNECTION_URI: postgresql://{N8N_DB_USER}:{N8N_DB_PASSWORD}@postgres_n8n/evolution
    DATABASE_SAVE_DATA_INSTANCE: true
    DATABASE_SAVE_MESSAGE_NEW: true
    DATABASE_SAVE_MESSAGE_UPDATE: true
    WEBHOOK_GLOBAL_ENABLED: false
    WEBHOOK_GLOBAL_URL: ""
    WEBHOOK_EVENTS_MESSAGES_UPSERT: true
    WEBHOOK_EVENTS_MESSAGES_UPDATE: true
    WEBHOOK_EVENTS_CONNECTION_UPDATE: true
    CACHE_REDIS_ENABLED: true
    CACHE_REDIS_URI: redis://chatwoot_redis:6379
    CACHE_REDIS_PREFIX_KEY: evolution
    LOG_LEVEL: ERROR
  Volume: evolution_instances:/evolution/instances
  Network: uddokta_internal
  NOTE: Evolution shares the postgres_n8n database but uses its own
  'evolution' database within that PostgreSQL instance.
  Before starting, create the database:
    docker compose exec postgres_n8n psql -U {N8N_DB_USER} -c "CREATE DATABASE evolution;"

Deploy:
  docker compose up -d evolution
  docker compose logs evolution --follow
  # Wait for: "HTTP Server running on http://0.0.0.0:8080"
```

```
AGENT TASK 4.2.2
Create a WhatsApp instance for the first test tenant via Evolution API.

The provision-tenant Edge Function must be UPDATED again to call
Evolution API when creating a new workspace.

Add to provision-tenant/index.ts after Chatwoot setup:

  // Create Evolution API instance for this tenant
  const evolutionRes = await fetch(
    `${EVOLUTION_BASE_URL}/instance/create`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'apikey': EVOLUTION_API_KEY
      },
      body: JSON.stringify({
        instanceName: `tenant-${tenantSlug}`,
        integration: 'WHATSAPP-BAILEYS', // demo mode for Phase 4 testing
        webhook: {
          url: `${N8N_WEBHOOK_BASE}/evolution-inbound`,
          by_events: true,
          events: ['MESSAGES_UPSERT', 'MESSAGES_UPDATE', 'CONNECTION_UPDATE']
        },
        reject_call: false,
        groups_ignore: true
      })
    }
  );
  // Store evolution_instance_name in tenants table

Add column:
  infra/supabase/migrations/0010_tenant_evolution.sql
  ALTER TABLE public.tenants ADD COLUMN evolution_instance_name TEXT;
Run: supabase db push

For demo/test, QR-based connection:
  GET https://{VPS_DOMAIN}/evolution/instance/connect/tenant-{slug}
  Returns a QR code — scan with WhatsApp to connect.
  In the admin health screen, show the QR when status is 'disconnected'.
```

### 4.3 n8n Core Webhook Workflows

```
AGENT TASK 4.3.1
Build the "evolution-inbound" n8n workflow.
This is the most critical workflow in the system.

Create workflow: "Evolution API — Inbound Message Router"

Node 1 — Webhook trigger
  Path: /evolution-inbound
  Method: POST
  No auth (Evolution API calls this on the internal Docker network)
  IMPORTANT: Add IP allowlist check in Function node:
    Only accept requests from the evolution container's internal IP.

Node 2 — Filter: MESSAGES_UPSERT events only (Function node)
  If event !== 'messages.upsert', respond 200 and stop.
  Extract: from (sender phone), message body, pushName, instanceName
  Derive tenantSlug from instanceName: "tenant-{slug}" → slug

Node 3 — Supabase: Lookup tenant by slug
  GET {SUPABASE_URL}/rest/v1/tenants?slug=eq.{tenantSlug}&select=id,slug,plan
  Use service_role key.
  If no tenant found: log and stop.

Node 4 — Supabase: Upsert contact
  POST {SUPABASE_URL}/rest/v1/contacts
  Headers: Prefer: resolution=merge-duplicates, return=representation
  Body: { tenant_id, phone: normalized_phone, name: pushName, platform: 'whatsapp' }
  This uses the UNIQUE(tenant_id, phone) constraint to upsert safely.

Node 5 — Supabase: Upsert conversation
  POST {SUPABASE_URL}/rest/v1/conversations
  Body: { tenant_id, contact_id, status: 'open', last_message_at: now() }
  Upsert on (tenant_id, contact_id) — needs a unique constraint:
  Add to 0001 migration or new migration:
    CREATE UNIQUE INDEX conversations_tenant_contact_unique
    ON public.conversations(tenant_id, contact_id);

Node 6 — Supabase: Insert message (idempotent)
  POST {SUPABASE_URL}/rest/v1/messages
  Body: { tenant_id, conversation_id, direction: 'inbound',
    body: messageText, platform_msg_id: messageId, status: 'received' }
  Headers: Prefer: resolution=ignore-duplicates
  ON CONFLICT (platform_msg_id) DO NOTHING
  This handles WhatsApp's at-least-once delivery guarantee.

Node 7 — Chatwoot: Create/update conversation
  The tenant's chatwoot_account_id and chatwoot_api_token are fetched
  from Supabase in Node 3.
  POST {CHATWOOT_URL}/api/v1/accounts/{accountId}/conversations
  Body: { inbox_id, contact_id (Chatwoot internal), ... }
  NOTE: This requires Chatwoot contacts and inboxes to be pre-created.
  For Phase 4, log the Chatwoot call and implement it when Chatwoot
  per-tenant configuration is complete.

Node 8 — Respond 200
  Return: { processed: true }

Export and save to: infra/n8n/workflows/evolution-inbound.json
Commit to git.
```

```
AGENT TASK 4.3.2
Build the "outbound-message" n8n workflow.

Create workflow: "Send Outbound WhatsApp Message"

This workflow is triggered by an HTTP call from the Next.js API route.

Node 1 — Webhook trigger
  Path: /send-message
  Method: POST
  Auth: Header { X-Internal-Token: {N8N_INTERNAL_TOKEN from .env} }
  Body schema: { tenant_id, conversation_id, message_body, media_url? }

Node 2 — Supabase: Get tenant + conversation data
  Fetch evolution_instance_name and contact phone from Supabase

Node 3 — Evolution API: Send message
  POST {EVOLUTION_URL}/message/sendText/{instanceName}
  Headers: apikey: {EVOLUTION_API_KEY}
  Body: { number: phone, text: message_body }

Node 4 — Error handling
  On HTTP error from Evolution:
  - If status 429 (rate limit): insert into message_retry_queue
  - If status 4xx (bad request): update message status to 'failed'
  - If status 5xx (server error): insert into message_retry_queue

Node 5 — Supabase: Update message status
  On success: UPDATE messages SET status = 'sent' WHERE id = {message_id}
  On failure: UPDATE messages SET status = 'failed'

Node 6 — Respond 200

Export and save to: infra/n8n/workflows/outbound-message.json
```

```
AGENT TASK 4.3.3
Build the "retry-queue-processor" n8n workflow.

Create workflow: "Message Retry Queue Processor"
Trigger: Cron — every 30 seconds

Node 1 — Supabase: Fetch eligible retries
  GET {SUPABASE_URL}/rest/v1/message_retry_queue
  ?next_retry_at=lte.{NOW()}&attempt_count=lt.max_attempts&select=*
  Use service_role key. Limit 10 rows per run.

Node 2 — SplitInBatches (n8n built-in)
  Process one retry at a time.

Node 3 — Evolution API: Retry send
  POST /message/sendText/{instanceName} with payload from queue row

Node 4 — IF success/failure:
  Success: DELETE from message_retry_queue WHERE id = {id}
           UPDATE messages SET status = 'sent'
  Failure: UPDATE message_retry_queue SET
             attempt_count = attempt_count + 1,
             next_retry_at = NOW() + (INTERVAL '30 seconds' * 2^attempt_count),
             last_error_code = {error_code}
           IF attempt_count >= max_attempts:
             UPDATE messages SET status = 'permanently_failed'
             DELETE from message_retry_queue

Export and save to: infra/n8n/workflows/retry-queue-processor.json
```

### 4.4 Wire Supabase Realtime to Next.js Dashboard

```
AGENT TASK 4.4.1
Add Supabase Realtime subscriptions to the Next.js inbox.

Create apps/web/src/hooks/useRealtimeMessages.ts:

This hook:
1. Subscribes to the Supabase Realtime channel:
   `tenant:{tenantId}:messages`
   Event: postgres_changes, INSERT on public.messages
   Filter: tenant_id=eq.{tenantId}
2. On new message INSERT: updates local React Query cache
3. On CHANNEL_ERROR: shows a "reconnecting" indicator in UI
4. On re-SUBSCRIBE after disconnect: fetches messages since
   lastMessageTimestamp to fill the gap (REST API call)
5. Returns: { realtimeStatus: 'connected' | 'reconnecting' | 'error' }

Create apps/web/src/app/(app)/inbox/page.tsx:
- RSC: initial fetch of conversations (status=open, last 50)
- Client component wrapper that activates the Realtime subscription
- Conversation list: customer name, channel icon, last message preview,
  time, unread count badge, assigned staff avatar
- Filters: All | Unread | Assigned to Me | Has Order
- Each conversation card taps to /app/inbox/[conversationId]
- Shows unread message count as badge on Inbox bottom tab icon

Create apps/web/src/app/(app)/inbox/[id]/page.tsx:
- Conversation detail with message thread (chat bubble layout)
- Customer profile drawer (swipe or button to open)
- Reply input fixed at bottom
- Quick reply button opens product/FAQ template drawer
- "Create Order" button → pre-fills /app/orders/new with this contact
- "Assign" button → staff selector (owner/manager only)
- "Pause Bot" toggle (calls n8n pause workflow)
- Message status indicators: sending / sent / delivered / failed

The reply input posts to:
  Next.js API route: POST /api/messages/send
  which validates the JWT, extracts tenant_id from server session,
  and forwards to n8n's /send-message webhook.
  Never call Evolution API directly from the browser.
```

### 4.5 n8n Follow-Up & Report Workflows

```
AGENT TASK 4.5.1
Build the "daily-report-cron" n8n workflow.

Create workflow: "Daily Business Report Generator"
Trigger: Cron — 09:00 Asia/Dhaka, every day

Node 1 — Supabase: Get all active tenants
  GET /rest/v1/tenants?plan=neq.suspended&select=id,slug,name

Node 2 — SplitInBatches: process each tenant

Node 3 — Supabase: Aggregate daily stats for tenant
  Run a single SQL query via Supabase RPC:
  Create a Supabase database function: get_daily_stats(p_tenant_id, p_date)
  Returns: { total_orders, new_orders, confirmed_orders, shipped,
    delivered, returned, cancelled, total_revenue, follow_ups_sent }

  Add this function in:
  infra/supabase/migrations/0011_reporting_functions.sql

Node 4 — Format report data as JSON

Node 5 — Store in Supabase: INSERT into daily_reports table
  Add migration: 0012_daily_reports.sql
  CREATE TABLE public.daily_reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES public.tenants(id),
    report_date DATE NOT NULL,
    data JSONB NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now(),
    UNIQUE(tenant_id, report_date)
  );
  ALTER TABLE public.daily_reports ENABLE ROW LEVEL SECURITY;
  CREATE POLICY "reports_tenant_read" ON public.daily_reports
    FOR SELECT USING (tenant_id = public.current_tenant_id());

Node 6 — WhatsApp: Send report summary to tenant owner
  Short summary via Evolution API to the owner's WhatsApp number
  Format: "আজকের রিপোর্ট: {X} অর্ডার, {Y} ডেলিভার, {Z} রিটার্ন। Dashboard দেখুন: {link}"

Export and save to: infra/n8n/workflows/daily-report-cron.json
```

```
AGENT TASK 4.5.2
Build the "sla-breach-check" n8n workflow.

Create workflow: "SLA Breach Alert — Unanswered Conversations"
Trigger: Cron — every 15 minutes

For each active tenant:
  GET conversations WHERE status = 'open'
    AND last_message_at < NOW() - INTERVAL '2 hours'
    AND direction of last message = 'inbound'
    (meaning: customer messaged, no staff reply in 2 hours)

For each breached conversation:
  POST to Chatwoot API: create private note on conversation
  "⚠️ এই কথোপকথনে 2 ঘণ্টা ধরে কোনো reply নেই"
  (Create Chatwoot internal note, not a message to the customer)

Export and save to: infra/n8n/workflows/sla-breach-check.json
```

### 4.6 Phase 4 Acceptance Checks

```
ACCEPTANCE CHECK — DO NOT PROCEED TO PHASE 5 UNTIL ALL PASS

[ ] Chatwoot is accessible at https://{VPS_DOMAIN}/chatwoot
[ ] Evolution API is responding at https://{VPS_DOMAIN}/evolution/instance/fetchInstances
[ ] A test WhatsApp instance is connected (QR scanned, status: CONNECTED)
[ ] Sending a WhatsApp message to the test number creates a row in public.messages
[ ] The Supabase Realtime subscription fires within 1 second of INSERT
[ ] The Next.js inbox updates in real-time when a new message arrives (no refresh)
[ ] Replying from the Next.js dashboard sends the message to WhatsApp
[ ] Duplicate message delivery (same platform_msg_id sent twice to n8n) creates only ONE row
    Test: POST the same evolution-inbound payload twice. COUNT public.messages WHERE platform_msg_id = X → must be 1.
[ ] A failed message enters the retry queue correctly
[ ] The retry queue processor runs and retries the failed message
[ ] Daily report cron workflow runs without error (trigger manually to test)
[ ] SLA breach check workflow runs without error
[ ] All n8n workflows are exported and committed to infra/n8n/workflows/
[ ] n8n admin UI is NOT accessible from a browser without Basic Auth credentials
```

---

## Phase 5: Automated AI Marketing Pipeline

> **Gate condition:** This phase can begin only after Phase 4 is fully stable. Phase 5 components are non-blocking for the core product and run in isolation. A failure in any Phase 5 service must never affect the seller's ability to use the dashboard.

### 5.1 Apify Lead Scraping Integration

```
AGENT TASK 5.1.1
Set up Apify cloud integration for Facebook page auditing.

On apify.com:
1. Create a new account and get the API token.
   Store as: APIFY_API_TOKEN in infra/.env

2. Create Actor: "uddokta/fb-page-auditor"
   This is a custom Actor to be built using the Apify SDK.
   Create the Actor source at packages/apify-actors/fb-page-auditor/main.js

   The Actor must:
   - Accept input: { startUrls: [{ url: facebookPageUrl }], leadId: string }
   - Navigate to the Facebook page URL using Apify's PlaywrightCrawler
   - Extract these public fields from the page DOM:
     * Page name (og:title or page header)
     * Follower/like count (from About section or page header)
     * Posts in the last 30 days (count visible posts with dates)
     * Average post engagement (likes + comments on last 10 posts)
     * Response time indicator (if visible on page: "Typically replies within...")
     * Verified badge (boolean)
     * Page category
   - Return structured JSON dataset
   - Handle: private pages (return { error: 'page_private' }),
             not found (return { error: 'page_not_found' })

3. Deploy to Apify cloud:
   cd packages/apify-actors/fb-page-auditor
   npx apify-cli push

4. Test the Actor via Apify console with a real public Facebook page URL.
   Verify the dataset output matches the expected schema.
```

```
AGENT TASK 5.1.2
Build the n8n "apify-audit-trigger" workflow.

This workflow is triggered by the "typebot-lead" workflow
(add a call to this workflow from the end of the typebot-lead workflow
created in Phase 3, but only if facebook_url is provided).

Create workflow: "Apify — Facebook Page Audit Trigger"

Node 1 — Webhook trigger (called internally from typebot-lead workflow)
  Path: /apify-audit-trigger
  Body: { lead_id, facebook_url }

Node 2 — Generate HMAC signature for Apify callback security
  Function node:
  const crypto = require('crypto');
  const token = crypto.createHmac('sha256', process.env.APIFY_CALLBACK_SECRET)
    .update(leadId).digest('hex');

Node 3 — Apify: Start Actor run
  POST https://api.apify.com/v2/acts/uddokta~fb-page-auditor/runs
  Headers: Authorization: Bearer {APIFY_API_TOKEN}
  Body: {
    startUrls: [{ url: facebook_url }],
    webhooks: [{
      eventTypes: ["ACTOR.RUN.SUCCEEDED", "ACTOR.RUN.FAILED"],
      requestUrl: "https://{VPS_DOMAIN}/n8n/webhook/apify-audit-callback",
      headersTemplate: { "X-Apify-Secret": "{hmac_token}" },
      payloadTemplate: '{"leadId": "{lead_id}", "datasetId": "{{defaultDatasetId}}", "status": "{{status}}"}'
    }]
  }

Node 4 — Supabase: Update lead status
  UPDATE leads SET audit_status = 'scraping_started' WHERE id = {lead_id}

Export and save to: infra/n8n/workflows/apify-audit-trigger.json
```

```
AGENT TASK 5.1.3
Build the n8n "apify-audit-callback" workflow.

Create workflow: "Apify — Audit Callback Processor"

Node 1 — Webhook trigger
  Path: /apify-audit-callback
  Method: POST
  Validate X-Apify-Secret header by recomputing HMAC and comparing

Node 2 — IF: status === ACTOR.RUN.FAILED
  → Update leads table: audit_status = 'scrape_failed'
  → Stop workflow

Node 3 — Fetch Apify dataset
  GET https://api.apify.com/v2/datasets/{datasetId}/items
  Headers: Authorization: Bearer {APIFY_API_TOKEN}
  Extract first item from response array.

Node 4 — Supabase: Store raw data
  UPDATE leads SET
    raw_apify_data = {scraped_json},
    audit_status = 'data_collected'
  WHERE id = {leadId}

Node 5 — Dify: Generate audit report
  POST https://{VPS_DOMAIN}/dify-api/v1/workflows/run
  Headers: Authorization: Bearer {DIFY_AUDIT_APP_KEY}
  Body: {
    inputs: {
      business_name: {lead.business_name},
      follower_count: {scraped.follower_count},
      posts_30d: {scraped.post_count_30d},
      avg_engagement: {scraped.avg_engagement},
      response_time: {scraped.response_time},
      daily_messages: {lead.daily_messages},
      biggest_pain: {lead.biggest_pain}
    },
    response_mode: "blocking",
    user: {leadId}
  }
  Timeout: 45 seconds
  On timeout or error: store raw data, set audit_status = 'dify_failed',
  skip to Node 7 (send generic WhatsApp message)

Node 6 — Parse Dify response
  Extract: score, reasons[3], improvements[3], urgency_label
  Supabase: UPDATE leads SET
    audit_score = {score},
    audit_report = {full_dify_response_json},
    audit_status = 'report_generated'

Node 7 — Evolution API: Send WhatsApp audit summary
  Find the evolution instance for the Uddokta admin account
  (The admin sends from the Uddokta business WhatsApp number)
  POST /message/sendText/{adminInstanceName}
  Body: {
    number: {lead.phone},
    text: "আপনার Inbox Leak Score: {score}/100\n
           🔴 সমস্যা: {reasons[0]}\n
           ✅ সমাধান: {improvements[0]}\n\n
           বিস্তারিত রিপোর্ট পেতে এখানে reply করুন 👇"
  }

Node 8 — Twenty CRM: Update opportunity stage
  Move opportunity to "Audit Sent" stage via Twenty GraphQL API

Export and save to: infra/n8n/workflows/apify-audit-callback.json
```

### 5.2 Dify Deployment

```
AGENT TASK 5.2.1
Deploy Dify using the official Docker Compose configuration,
merged carefully into the Uddokta master Compose file.

Clone the Dify repo to inspect its Compose structure:
  git clone https://github.com/langgenius/dify.git /opt/src/dify --depth=1

Read /opt/src/dify/docker/docker-compose.yaml carefully.
Dify ships with its own: nginx, postgres, redis, weaviate, api, worker, web.

MERGE STRATEGY — do NOT run Dify's own docker-compose.yaml.
Extract only these services into the Uddokta master Compose:
  - dify_api (rename from 'api')
  - dify_worker (rename from 'worker')
  - dify_web (rename from 'web')
  - dify_db (new postgres:16-alpine — do NOT reuse Supabase)

SHARE these existing Uddokta services with Dify:
  - Redis: use chatwoot_redis (same instance, different prefix key)
  - Object storage: use Cloudflare R2 (configured via env vars)

DISABLE in Dify:
  - Weaviate (set VECTOR_STORE=none — reduces RAM by ~1.5GB)
  - Dify's own nginx (Caddy handles routing)
  - Dify's own postgres (use dify_db container)
  - Dify's own redis (use chatwoot_redis)

CRITICAL environment variables for dify_api and dify_worker:
  SECRET_KEY: {DIFY_SECRET_KEY}
  DB_USERNAME: {DIFY_DB_USER}
  DB_PASSWORD: {DIFY_DB_PASSWORD}
  DB_HOST: dify_db
  DB_DATABASE: dify
  REDIS_HOST: chatwoot_redis
  REDIS_PORT: 6379
  VECTOR_STORE: none
  STORAGE_TYPE: s3
  S3_USE_AWS_MANAGED_IAM: false
  S3_ENDPOINT: {R2_ENDPOINT}
  S3_BUCKET_NAME: {R2_BUCKET_DIFY}
  S3_ACCESS_KEY: {R2_ACCESS_KEY}
  S3_SECRET_KEY: {R2_SECRET_KEY}
  S3_REGION: auto
  CONSOLE_WEB_URL: https://{VPS_DOMAIN}/dify
  APP_WEB_URL: https://{VPS_DOMAIN}/dify
  SERVICE_API_URL: https://{VPS_DOMAIN}/dify-api

Check ARM64 for each Dify image:
  docker manifest inspect langgenius/dify-api:latest | grep arm64
  docker manifest inspect langgenius/dify-web:latest | grep arm64
If no ARM64 manifests: build from source.

Deploy:
  docker compose up -d dify_db
  docker compose up -d dify_api dify_worker dify_web
  docker compose logs dify_api --follow
  # Wait for: "Application startup complete"
```

```
AGENT TASK 5.2.2
Build the Dify "inbox-leak-audit-generator" workflow application.

Access Dify at https://{VPS_DOMAIN}/dify
Create a new Workflow application named: "inbox-leak-audit-generator"

Configure these nodes in the Dify visual workflow builder:

Start node — Input variables:
  business_name (string), follower_count (number), posts_30d (number),
  avg_engagement (number), response_time (string), daily_messages (number),
  biggest_pain (string)

LLM node — Model: claude-haiku-3 (set Anthropic API key in Dify settings)
  System prompt:
  "You are an expert F-commerce business consultant specializing in
  Bangladeshi social-commerce sellers. Analyze the Facebook page metrics
  provided and generate an Inbox Leakage Audit Report.

  IMPORTANT: Respond ONLY with valid JSON. No preamble, no markdown,
  no explanation outside the JSON structure.

  JSON structure:
  {
    'score': <integer 0-100, where 100 = maximum inbox leakage>,
    'urgency': <'Critical' | 'High' | 'Medium' | 'Low'>,
    'reasons': [<string>, <string>, <string>],
    'improvements': [<string>, <string>, <string>],
    'estimated_monthly_leak': <string, e.g. '15-20 orders/month'>,
    'summary_bangla': <2-sentence summary in Bangla>
  }"

  User message template:
  "Business: {{business_name}}
  Followers: {{follower_count}}
  Posts in last 30 days: {{posts_30d}}
  Average engagement rate: {{avg_engagement}}%
  Response time indicator: {{response_time}}
  Daily estimated messages: {{daily_messages}}
  Biggest reported pain: {{biggest_pain}}"

End node — Output: the LLM response as structured output

Get the API key for this application from Dify settings.
Store as: DIFY_AUDIT_APP_KEY in infra/.env
```

### 5.3 MoneyPrinterTurbo Deployment

```
AGENT TASK 5.3.1
Build the custom FastAPI wrapper for MoneyPrinterTurbo.

Create packages/mpt-wrapper/main.py:

from fastapi import FastAPI, BackgroundTasks
from fastapi.responses import JSONResponse
import subprocess, uuid, json, os, boto3, sqlite3
from pathlib import Path

app = FastAPI()

DB_PATH = "/app/temp/jobs.db"
OUTPUT_DIR = "/app/temp/output"
MAX_CONCURRENT = int(os.getenv("MAX_CONCURRENT_JOBS", "1"))

# Initialize SQLite for job tracking
def init_db():
    conn = sqlite3.connect(DB_PATH)
    conn.execute("""CREATE TABLE IF NOT EXISTS jobs (
      id TEXT PRIMARY KEY, status TEXT, output_url TEXT,
      error TEXT, created_at TEXT, completed_at TEXT
    )""")
    conn.commit()
    conn.close()

@app.on_event("startup")
def startup():
    os.makedirs(OUTPUT_DIR, exist_ok=True)
    init_db()

@app.post("/generate")
async def generate_video(payload: dict, background_tasks: BackgroundTasks):
    # Check concurrent job limit
    conn = sqlite3.connect(DB_PATH)
    running = conn.execute(
      "SELECT COUNT(*) FROM jobs WHERE status='running'"
    ).fetchone()[0]
    conn.close()
    if running >= MAX_CONCURRENT:
        return JSONResponse({"error": "queue_full"}, status_code=429)
    job_id = str(uuid.uuid4())
    _insert_job(job_id, "queued")
    background_tasks.add_task(run_generation, job_id, payload)
    return {"job_id": job_id, "status": "queued"}

@app.get("/status/{job_id}")
def get_status(job_id: str):
    conn = sqlite3.connect(DB_PATH)
    row = conn.execute("SELECT * FROM jobs WHERE id=?", (job_id,)).fetchone()
    conn.close()
    if not row:
        return JSONResponse({"error": "not_found"}, status_code=404)
    return {"job_id": row[0], "status": row[1], "output_url": row[2],
            "error": row[3], "created_at": row[4], "completed_at": row[5]}

def run_generation(job_id: str, payload: dict):
    _update_job(job_id, "running")
    try:
        # Call MoneyPrinterTurbo's VideoService directly via Python import
        # MPT is installed in the same container
        import sys
        sys.path.insert(0, "/app/mpt")
        from app.services.video_service import VideoService
        vs = VideoService()
        # Build task from payload
        task = {
            "video_script": payload.get("script", ""),
            "video_terms": payload.get("video_terms", ["ecommerce"]),
            "video_aspect": payload.get("aspect", "9:16"),
            "video_language": "Bengali",
            "voice_name": payload.get("voice", "bn-BD-NabanitaNeural"),
            "bgm_type": "random",
            "video_count": 1
        }
        output_path = f"{OUTPUT_DIR}/{job_id}.mp4"
        vs.create_video(task, output_file=output_path)
        # Upload to Cloudflare R2
        r2_url = upload_to_r2(output_path, job_id)
        os.remove(output_path)  # Free disk space immediately
        _update_job(job_id, "completed", output_url=r2_url)
    except Exception as e:
        _update_job(job_id, "failed", error=str(e))

def upload_to_r2(file_path: str, job_id: str) -> str:
    s3 = boto3.client("s3",
        endpoint_url=os.environ["R2_ENDPOINT"],
        aws_access_key_id=os.environ["R2_ACCESS_KEY"],
        aws_secret_access_key=os.environ["R2_SECRET_KEY"],
        region_name="auto"
    )
    key = f"videos/{job_id}.mp4"
    s3.upload_file(file_path, os.environ["R2_BUCKET"], key)
    return f"{os.environ['R2_ENDPOINT']}/{os.environ['R2_BUCKET']}/{key}"

Create packages/mpt-wrapper/Dockerfile.arm64:
FROM python:3.11-slim-bullseye
# ARM64-specific PyTorch CPU install
RUN pip install torch==2.2.0 torchvision==0.17.0 torchaudio==2.2.0 \
    --index-url https://download.pytorch.org/whl/cpu
RUN pip install fastapi uvicorn boto3 edge-tts moviepy Pillow requests
# Clone MoneyPrinterTurbo into /app/mpt
RUN apt-get update && apt-get install -y git ffmpeg
RUN git clone https://github.com/harry0703/MoneyPrinterTurbo.git /app/mpt
RUN cd /app/mpt && pip install -r requirements.txt
COPY main.py /app/main.py
WORKDIR /app
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8502"]

Build the ARM64 image on the VPS (NOT locally — must be ARM):
  cd /opt/uddokta
  git pull  # get the latest packages/mpt-wrapper/
  docker build --platform linux/arm64 \
    -t uddokta/mpt-wrapper:local \
    -f packages/mpt-wrapper/Dockerfile.arm64 \
    packages/mpt-wrapper/

WARNING: This build takes 15-30 minutes on the Oracle ARM VPS due to
PyTorch download and compilation. Do not interrupt.

Add service to docker-compose.yml: mpt_wrapper
  Image: uddokta/mpt-wrapper:local (local build, not a registry image)
  Environment: R2_ENDPOINT, R2_ACCESS_KEY, R2_SECRET_KEY, R2_BUCKET from .env
               MAX_CONCURRENT_JOBS: "1"
               PEXELS_API_KEY: {PEXELS_API_KEY}
  Volume: r2_staging:/app/temp
  Networks: uddokta_internal
  Deploy resources limit: cpus 2.0, memory 4G
    (This prevents MPT from consuming all RAM and starving other services)

Deploy:
  docker compose up -d mpt_wrapper
  curl http://localhost:8502/status/test → should return 404 gracefully
```

```
AGENT TASK 5.3.2
Build the n8n "video-generation" workflow.

Create workflow: "MoneyPrinterTurbo — Video Generation Scheduler"
Trigger: Cron — Monday, Wednesday, Friday at 02:00 Asia/Dhaka
(Early morning to use off-peak hours when VPS load is lowest)

Node 1 — Define video scripts (Set node)
  Maintain a rotating array of 6 Bangla educational scripts:
  Script topics:
  1. "প্রতিদিন কতজন ক্রেতা আপনার inbox-এ হারিয়ে যাচ্ছে?"
  2. "COD return কমানোর ৩টি সহজ উপায়"
  3. "WhatsApp-এ professional receipt পাঠান — trust বাড়বে"
  4. "Staff ছাড়া inbox manage করুন — এটাই Uddokta"
  5. "Facebook seller-দের সবচেয়ে বড় ভুল"
  6. "৫ মিনিটে আপনার order board setup করুন"
  Each script is ~200 Bangla characters for a ~60-second video.
  Use a counter in n8n static data to rotate through scripts.

Node 2 — Check MPT queue status
  GET http://mpt_wrapper:8502/status/latest (add a GET /queue endpoint)
  If running jobs >= 1: stop workflow, log "MPT busy, skipping this run"

Node 3 — HTTP Request to MPT wrapper
  POST http://mpt_wrapper:8502/generate
  Body: { script, voice: "bn-BD-NabanitaNeural",
    video_terms: ["bangladesh ecommerce", "online seller"], aspect: "9:16" }
  Store returned job_id in n8n execution data.

Node 4 — Wait 2 minutes (n8n Wait node)

Node 5 — Poll MPT status (loop up to 15 times with 2-minute intervals)
  GET http://mpt_wrapper:8502/status/{job_id}
  If status === "completed": proceed to Node 6
  If status === "failed": log error, notify admin via WhatsApp, stop
  If status === "running": loop back to Wait (max 15 loops = 30 min timeout)

Node 6 — Notify content team
  POST evolution API: send WhatsApp message to admin number
  "✅ নতুন marketing video তৈরি হয়েছে: {output_url}
   Review করুন এবং Facebook/Instagram-এ post করুন।"

Export and save to: infra/n8n/workflows/video-generation.json
```

### 5.4 Mautic Deployment

```
AGENT TASK 5.4.1
Deploy Mautic for email drip campaigns targeting agency leads
and larger SME prospects.

Check ARM64 availability:
  docker manifest inspect mautic/mautic:latest | grep arm64
Mautic does NOT have official ARM64 images as of 2025.
Build from source:
  git clone https://github.com/mautic/mautic.git /opt/src/mautic --depth=1 --branch=5.x
  docker build --platform linux/arm64 \
    -t uddokta/mautic:local /opt/src/mautic
WARNING: Mautic build takes 20-40 minutes. Run in a tmux session.

Add services to docker-compose.yml:
  mautic_db (mysql:8.0-oracle — Mautic requires MySQL, not PostgreSQL)
    Volume: mautic_pg_data (rename to mautic_mysql_data for clarity)
    MYSQL_DATABASE: mautic
    MYSQL_USER / MYSQL_PASSWORD from .env
    Healthcheck: mysqladmin ping

  mautic (uddokta/mautic:local)
    Depends on: mautic_db (healthy)
    Environment:
      MAUTIC_DB_HOST: mautic_db
      MAUTIC_DB_NAME: mautic
      MAUTIC_DB_USER: {MAUTIC_DB_USER}
      MAUTIC_DB_PASSWORD: {MAUTIC_DB_PASSWORD}
      MAUTIC_SITE_URL: https://{VPS_DOMAIN}/mautic
      MAUTIC_SECRET_KEY: {MAUTIC_SECRET_KEY}
    Volume: mautic_data:/var/www/html/var
    Network: uddokta_internal

Deploy:
  docker compose up -d mautic_db
  # Wait for MySQL healthcheck
  docker compose up -d mautic
  # Access https://{VPS_DOMAIN}/mautic
  # Complete initial setup wizard
  # Configure SMTP (Brevo free tier: 300 emails/day)
  #   In Mautic: Settings → SMTP → Host: smtp.brevo.com, Port: 587

Create Mautic campaign: "Uddokta Trial Onboarding Drip"
  Email 1 (immediate): Welcome + link to admin walkthrough video
  Email 2 (+2 days): "3 features most sellers use first"
  Email 3 (+5 days): "Your first week checklist"
  Email 4 (+12 days): "Upgrade to Growth — here's what unlocks"

  NOTE: In Phase 5, leads are added to Mautic via n8n.
  Add to the typebot-lead workflow (Node at the end):
  POST http://mautic:80/api/contacts (with Mautic API token)
  Add lead to "Trial Prospects" segment.
```

### 5.5 Phase 5 Acceptance Checks

```
ACCEPTANCE CHECK — PHASE 5 COMPLETE WHEN ALL PASS

[ ] Apify Actor "uddokta/fb-page-auditor" is deployed and runs successfully
    on a test public Facebook page URL via the Apify console
[ ] n8n apify-audit-trigger workflow fires when a Typebot lead with a
    Facebook URL completes
[ ] n8n apify-audit-callback workflow receives the Apify result,
    calls Dify, and stores the audit report in Supabase leads table
[ ] Dify audit workflow returns valid JSON with score, reasons, improvements
[ ] If Dify fails, the n8n workflow sends a generic fallback WhatsApp message
    and does NOT silently drop the lead
[ ] MPT wrapper responds at https://{VPS_DOMAIN}/mpt/status/test (404 is OK)
[ ] A video generation job can be queued via POST /mpt/generate
[ ] A generated video appears in the Cloudflare R2 bucket
[ ] Video generation is capped at 1 concurrent job (test by submitting 2)
[ ] Mautic is accessible and the SMTP test email sends successfully
[ ] A test lead flows through the full pipeline:
    Typebot form → n8n → Supabase leads → Twenty CRM → Apify scrape →
    Dify audit → WhatsApp message → Mautic email drip
[ ] All 3 n8n workflows (apify-trigger, apify-callback, video-generation)
    are exported and committed to infra/n8n/workflows/
[ ] Memory usage on Oracle VPS with all services running is under 20GB:
    docker stats --no-stream | awk '{print $4}' | grep -v MEM
    If over 20GB: disable Mautic temporarily or reduce Dify worker replicas.
```

---

## Phase 6: Testing, CI/CD & Launch Prep

> **Gate condition:** Every check in this phase must be green before any paying customer is onboarded.

### 6.1 Idempotency & Webhook Security Testing

```
AGENT TASK 6.1.1
Write and execute webhook idempotency tests.

Create: infra/tests/idempotency.sh

Test 1: Duplicate inbound message
  PAYLOAD=$(cat <<'EOF'
  {
    "event": "messages.upsert",
    "instance": "tenant-test",
    "data": {
      "key": { "remoteJid": "+8801XXXXXXXXX@s.whatsapp.net",
               "id": "WAMID_TEST_IDEMPOTENCY_001" },
      "message": { "conversation": "Test message" },
      "pushName": "Test Buyer"
    }
  }
  EOF
  )
  # Send the SAME payload twice to n8n evolution-inbound webhook
  curl -s -X POST https://{VPS_DOMAIN}/n8n/webhook/evolution-inbound \
    -H "Content-Type: application/json" -d "$PAYLOAD"
  sleep 2
  curl -s -X POST https://{VPS_DOMAIN}/n8n/webhook/evolution-inbound \
    -H "Content-Type: application/json" -d "$PAYLOAD"
  sleep 3
  # Query Supabase for count
  COUNT=$(curl -s \
    "${SUPABASE_URL}/rest/v1/messages?platform_msg_id=eq.WAMID_TEST_IDEMPOTENCY_001&select=id" \
    -H "apikey: ${SUPABASE_SERVICE_ROLE_KEY}" \
    -H "Authorization: Bearer ${SUPABASE_SERVICE_ROLE_KEY}" | jq length)
  echo "Message count (expect 1): $COUNT"
  if [ "$COUNT" -ne "1" ]; then
    echo "FAIL: Idempotency test failed — got $COUNT rows"
    exit 1
  fi
  echo "PASS: Idempotency test"

Test 2: Unsigned webhook rejection
  curl -s -X POST https://{VPS_DOMAIN}/n8n/webhook/typebot-lead \
    -H "Content-Type: application/json" \
    -d '{"phone": "+8801XXXXXXXXX"}' \
    -o /dev/null -w "%{http_code}"
  # Expect: 401 or 500 (unauthorized — missing X-Typebot-Secret header)

Test 3: RLS cross-tenant isolation
  # Get a JWT for Tenant A (use Supabase service role to create test session)
  TENANT_A_TOKEN=$(get_test_jwt_for_tenant_a)  # implement this helper
  # Attempt to read Tenant B's orders using Tenant A's token
  RESULT=$(curl -s \
    "${SUPABASE_URL}/rest/v1/orders?tenant_id=eq.{TENANT_B_ID}" \
    -H "apikey: ${SUPABASE_ANON_KEY}" \
    -H "Authorization: Bearer ${TENANT_A_TOKEN}" | jq length)
  echo "Cross-tenant order count (expect 0): $RESULT"
  if [ "$RESULT" -ne "0" ]; then
    echo "CRITICAL FAIL: RLS breach — tenant isolation is broken"
    exit 1
  fi
  echo "PASS: RLS isolation test"

Run: chmod +x infra/tests/idempotency.sh && ./infra/tests/idempotency.sh
ALL tests must pass before proceeding.
```

```
AGENT TASK 6.1.2
Add webhook security headers to the Caddyfile.

Update infra/caddy/Caddyfile to add security headers to ALL routes:

header {
  X-Content-Type-Options "nosniff"
  X-Frame-Options "DENY"
  X-XSS-Protection "1; mode=block"
  Referrer-Policy "strict-origin-when-cross-origin"
  Permissions-Policy "camera=(), microphone=(), geolocation=()"
  -Server
}

Additionally, add IP rate limiting to n8n webhook routes to prevent
webhook flooding:
  rate_limit /n8n/webhook/* {
    zone webhooks 10MB
    rate 60r/m
  }
Note: Caddy's rate_limit requires the caddy-ratelimit plugin.
Build a custom Caddy binary or use xcaddy:
  xcaddy build --with github.com/mholt/caddy-ratelimit
Replace the caddy:2-alpine image with this custom binary.

Reload Caddy: docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

### 6.2 VPS Backup Automation

```
AGENT TASK 6.2.1
Create the automated VPS backup system.

Create infra/scripts/backup.sh:

#!/bin/bash
set -e
DATE=$(date +%Y%m%d_%H%M)
BACKUP_DIR="/opt/uddokta/backups"
R2_BUCKET="uddokta-backups"

echo "[$DATE] Starting Uddokta backup..."

# Export n8n workflows to file system before backup
docker compose -f /opt/uddokta/infra/docker-compose.yml exec -T n8n \
  n8n export:workflow --all --output=/home/node/.n8n/exported_workflows.json

# Backup n8n PostgreSQL
docker compose -f /opt/uddokta/infra/docker-compose.yml exec -T postgres_n8n \
  pg_dump -U {N8N_DB_USER} n8n | gzip > ${BACKUP_DIR}/n8n_${DATE}.sql.gz

# Backup Chatwoot PostgreSQL
docker compose -f /opt/uddokta/infra/docker-compose.yml exec -T chatwoot_pg \
  pg_dump -U {CHATWOOT_DB_USER} chatwoot | gzip > ${BACKUP_DIR}/chatwoot_${DATE}.sql.gz

# Backup Twenty CRM PostgreSQL
docker compose -f /opt/uddokta/infra/docker-compose.yml exec -T twenty_db \
  pg_dump -U {TWENTY_DB_USER} twenty | gzip > ${BACKUP_DIR}/twenty_${DATE}.sql.gz

# Backup Evolution instances (connection data)
docker run --rm -v evolution_instances:/data alpine \
  tar czf - /data > ${BACKUP_DIR}/evolution_instances_${DATE}.tar.gz

# Upload all today's backups to R2
for f in ${BACKUP_DIR}/*_${DATE}*; do
  rclone copy "$f" r2:${R2_BUCKET}/$(basename $f) --s3-provider=Cloudflare
done

# Delete local backups older than 3 days
find ${BACKUP_DIR} -name "*.gz" -mtime +3 -delete

echo "[$DATE] Backup complete."

Configure rclone R2 remote:
  rclone config create r2 s3 \
    provider=Cloudflare \
    access_key_id={R2_ACCESS_KEY} \
    secret_access_key={R2_SECRET_KEY} \
    endpoint={R2_ENDPOINT}

Add crontab entry on the VPS as the uddokta user:
  crontab -e
  # Run backup at 02:00 BDT every day
  0 2 * * * /opt/uddokta/infra/scripts/backup.sh >> /opt/uddokta/logs/backup.log 2>&1

Test the backup:
  bash /opt/uddokta/infra/scripts/backup.sh
  rclone ls r2:uddokta-backups/
  Verify .sql.gz and .tar.gz files are present.
```

### 6.3 CI/CD Pipeline

```
AGENT TASK 6.3.1
Create GitHub Actions CI workflow for the Next.js app.

Create .github/workflows/ci.yml:

name: Uddokta CI

on:
  push:
    branches: [main, develop]
    paths: ['apps/web/**']
  pull_request:
    branches: [main]
    paths: ['apps/web/**']

jobs:
  lint-and-type-check:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: apps/web
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm', cache-dependency-path: apps/web/package-lock.json }
      - run: npm ci
      - run: npm run lint
      - run: npx tsc --noEmit

  security-checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check for secrets in code
        uses: trufflesecurity/trufflehog@main
        with: { path: ./, base: HEAD~1 }
      - name: Verify .gitignore protects .env files
        run: |
          if git ls-files --error-unmatch infra/.env 2>/dev/null; then
            echo "CRITICAL: infra/.env is tracked by git!"
            exit 1
          fi
          echo "PASS: .env files are not tracked"

  deploy-to-vercel:
    needs: [lint-and-type-check, security-checks]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm install -g vercel
      - run: vercel pull --yes --environment=production --token=${{ secrets.VERCEL_TOKEN }}
        working-directory: apps/web
      - run: vercel build --prod --token=${{ secrets.VERCEL_TOKEN }}
        working-directory: apps/web
      - run: vercel deploy --prebuilt --prod --token=${{ secrets.VERCEL_TOKEN }}
        working-directory: apps/web

Add GitHub repository secrets:
  VERCEL_TOKEN: (from Vercel dashboard → Account Settings → Tokens)
  VERCEL_ORG_ID: (from vercel link output)
  VERCEL_PROJECT_ID: (from vercel link output)
```

```
AGENT TASK 6.3.2
Create an n8n workflow export CI check.

Create .github/workflows/n8n-workflows.yml:

name: n8n Workflow Validation

on:
  push:
    paths: ['infra/n8n/workflows/**']

jobs:
  validate-workflows:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Validate all workflow JSON files
        run: |
          for f in infra/n8n/workflows/*.json; do
            echo "Validating: $f"
            python3 -c "import json; json.load(open('$f')); print('  Valid JSON')"
            # Check required fields
            python3 -c "
          import json, sys
          data = json.load(open('$f'))
          required = ['name', 'nodes', 'connections']
          for field in required:
              if field not in data:
                  print(f'FAIL: {field} missing in $f')
                  sys.exit(1)
          print('  Required fields present')
          "
          done
```

### 6.4 Final Tenant Isolation Verification

```
AGENT TASK 6.4.1
Run the complete tenant isolation test suite before any client onboarding.

Create infra/tests/tenant-isolation-full.sh:

This script must test ALL of the following isolation boundaries:

1. DATABASE LEVEL
   - Create 2 test tenants (A and B) with real data in each
   - Verify tenant A JWT cannot read tenant B's:
     * orders, contacts, conversations, messages, products,
       daily_reports, audit_logs

2. API LEVEL
   - Verify Next.js API routes /api/messages/send extract tenant_id
     from server-side JWT ONLY (never from request body)
   - Test: POST /api/messages/send with a spoofed tenant_id in the body
     → must use JWT tenant_id, not the spoofed one

3. PUBLIC ROUTES
   - Verify /public/receipt/{token} returns only safe fields
   - Test: manually check the response contains no phone, no full_address,
     no internal_notes, no platform_msg_id

4. WEBHOOK ROUTES
   - Verify an invalid X-Typebot-Secret returns 401/500
   - Verify a valid webhook with a non-existent tenant slug
     creates no data and returns 200 (safe failure)

5. N8N SERVICE ROLE
   - Verify every Supabase write from n8n includes an explicit tenant_id
   - Query: SELECT id FROM public.messages WHERE tenant_id IS NULL
     → must return 0 rows

6. ADMIN ROUTES
   - Verify /admin/* routes return 403 for non-admin users
   - Verify admin cannot modify data in a way that bypasses RLS
     (admin reads via service_role — verify no accidental cross-tenant writes)

Run all tests and collect results.
ZERO failures allowed before client onboarding.
```

### 6.5 Performance & Mobile Optimization

```
AGENT TASK 6.5.1
Run Lighthouse performance audit and fix critical issues.

Install: npm install -g lighthouse

Test the key seller-facing routes:
  lighthouse https://app.uddokta.com/login --output json --output-path ./lighthouse-login.json
  lighthouse https://app.uddokta.com/public/receipt/test-token --output json --output-path ./lighthouse-receipt.json

Minimum passing scores:
  Performance: 75+ (mobile)
  Accessibility: 90+
  Best Practices: 90+
  PWA: 80+

Common fixes if scores are below target:
  - Add loading="lazy" to all images below the fold
  - Add size hints to all images (width, height props on next/image)
  - Move non-critical fonts to preconnect hints in layout.tsx
  - Ensure all buttons have aria-label
  - Confirm all form inputs have associated labels
  - Add explicit skeleton states for all data-fetching components
    to prevent layout shift (CLS)
```

```
AGENT TASK 6.5.2
Configure Next.js Image Optimization for the Vercel deployment.

In apps/web/next.config.js, configure image domains to allow
images from the VPS and Cloudflare R2.

Ensure all <Image> components from next/image use:
  - width and height props (never omit these)
  - placeholder="blur" for product images
  - priority={true} for above-the-fold images only (logo, hero image)

For the receipt page specifically:
  - The business logo must load quickly — preload via <link rel="preload">
  - Test on a simulated slow 3G connection in Chrome DevTools
  - The receipt must be fully readable within 3 seconds on slow 3G
```

### 6.6 Environment Health Dashboard

```
AGENT TASK 6.6.1
Build the admin health monitoring screen in the Next.js app.

Create apps/web/src/app/(admin)/health/page.tsx

This screen polls the following health endpoints and shows a
red/yellow/green status indicator for each service.

Services to monitor:

VPS Services (polled every 30 seconds via client-side useEffect):
  POST /api/admin/health (Next.js server-side route that aggregates these)
  The server-side route checks:
  - n8n: GET https://{VPS_DOMAIN}/n8n/healthz → expect 200
  - Chatwoot: GET https://{VPS_DOMAIN}/chatwoot/auth/sign_in → expect 200/401
  - Evolution API: GET https://{VPS_DOMAIN}/evolution/instance/fetchInstances → expect 200
  - Typebot viewer: GET https://{VPS_DOMAIN}/typebot → expect 200
  - Dify: GET https://{VPS_DOMAIN}/dify → expect 200
  - Mautic: GET https://{VPS_DOMAIN}/mautic → expect 200

Database Services:
  - Supabase connectivity: simple SELECT 1 via supabase-js
  - Supabase Realtime: test channel subscription status

Business Metrics (from Supabase):
  - Total active tenants
  - Messages in last 24h (aggregate across all tenants)
  - Failed messages in last 24h
  - Retry queue depth
  - Leads in last 7 days

Each row shows: Service name | Status icon | Last checked timestamp
A "Refresh All" button triggers a manual re-check.
Show this screen only to admin users (role check in layout).
```

### 6.7 Pre-Launch Checklist

```
AGENT TASK 6.7.1
Execute the full pre-launch checklist. Every item must be checked
manually by a human and signed off before the first paying client
is onboarded.

INFRASTRUCTURE
[ ] Oracle VPS uptime > 99% in last 7 days (check OCI console)
[ ] All Docker services show "Up" status: docker compose ps
[ ] No container has restarted more than 3 times in the last 24h:
    docker compose ps --format json | jq '.[].Status' | grep -v Up
[ ] Backups ran successfully at least once (check R2 bucket)
[ ] Caddy TLS certificate is valid and not expiring within 30 days:
    docker compose exec caddy caddy certificates

SECURITY
[ ] infra/.env is NOT in git history:
    git log --all --full-history -- infra/.env | wc -l → must be 0
[ ] SUPABASE_SERVICE_ROLE_KEY is not in any source file:
    grep -r "service_role" apps/web/src/ → must return 0 results
[ ] All n8n webhook endpoints require authentication (tested in 6.1)
[ ] RLS isolation tests pass (tested in 6.4)
[ ] Public receipt does not expose private data (tested in 6.4)
[ ] Admin routes require admin role (tested in 6.4)
[ ] Caddy security headers are in place (verify with securityheaders.com)

APPLICATION
[ ] Login works on Android Chrome (real device test)
[ ] Login works on iPhone Safari (real device test)
[ ] PWA installs to home screen on Android
[ ] Order creation works end-to-end on mobile
[ ] Receipt link opens correctly on mobile without app installed
[ ] Inbox real-time updates within 2 seconds (test with two phones)
[ ] All n8n workflows are active (not paused):
    Check n8n dashboard → all workflows show "Active"
[ ] Daily report cron runs without error (trigger manually, verify output)

MONITORING
[ ] Admin health dashboard shows all green
[ ] n8n error execution log is empty (Settings → Executions, filter by error)
[ ] Supabase database is under 400MB (free tier limit: 500MB):
    Check Supabase dashboard → Project Settings → Usage
[ ] Vercel bandwidth is under 80GB for current month (limit: 100GB)

CONTENT
[ ] Typebot audit flow is live and tested end-to-end
[ ] At least 3 test leads have flowed through the full pipeline
[ ] Twenty CRM has the leads from test runs
[ ] Mautic welcome email sends correctly from the SMTP relay
[ ] At least 1 test video has been generated and uploaded to R2

DOCUMENTATION
[ ] All 6 planning documents are committed to docs/
[ ] All n8n workflow JSONs are committed to infra/n8n/workflows/
[ ] infra/.env.example is up to date with all current variables
[ ] README.md in repo root explains how to restore from scratch
[ ] VPS recovery time is tested: can a fresh VPS be fully operational
    in under 60 minutes from git clone + .env population?
    (Test this once in a staging environment before claiming compliance)
```

### 6.8 Phase 6 Final Acceptance

```
FINAL LAUNCH GATE — ALL ITEMS MUST BE GREEN

[ ] idempotency.sh passes all 3 tests with zero failures
[ ] tenant-isolation-full.sh passes all 6 isolation categories
[ ] Lighthouse mobile performance score ≥ 75 on /login
[ ] Lighthouse mobile performance score ≥ 80 on /public/receipt/*
[ ] GitHub Actions CI pipeline passes on a test PR
[ ] Vercel production deployment is live and healthy
[ ] Admin health dashboard shows all green indicators
[ ] Pre-launch checklist (6.7.1) is 100% checked and signed off
[ ] First test tenant onboarded via admin provision-tenant function
[ ] First real order created, receipt sent, and opened on mobile
[ ] First Typebot lead processed end-to-end in under 5 minutes
[ ] Backup restored successfully at least once (test restore, not just backup)

SYSTEM IS READY FOR FIRST PAYING CLIENT.
```

---

## Appendix A: VPS Full Recovery Procedure

> For use when the Oracle VPS is terminated or corrupted.

```bash
# On a fresh Oracle ARM VPS:
# Step 1: Bootstrap (from Task 1.1.2)
sudo apt-get update && sudo apt-get upgrade -y
# ... (install Docker as in Task 1.1.2)

# Step 2: Clone repo and populate secrets
git clone https://github.com/{your-org}/uddokta.git /opt/uddokta
cd /opt/uddokta/infra
cp .env.example .env
# Populate .env with values from your secure secrets store

# Step 3: Restore Docker volumes from R2
rclone copy r2:uddokta-backups/latest/ /opt/uddokta/backups/
# Restore n8n database
docker compose up -d postgres_n8n
docker compose exec -T postgres_n8n \
  psql -U {N8N_DB_USER} n8n < /opt/uddokta/backups/n8n_latest.sql

# Step 4: Start all services
docker compose up -d

# Step 5: Restore n8n workflows from Git
docker compose exec n8n \
  n8n import:workflow --input=/home/node/.n8n/exported_workflows.json

# Step 6: Verify health
curl https://{VPS_DOMAIN}/health
```

## Appendix B: Service RAM Allocation Reference

| Service | Typical RAM | Peak RAM |
|---|---|---|
| Caddy | 30 MB | 60 MB |
| n8n | 300 MB | 600 MB |
| postgres_n8n | 200 MB | 400 MB |
| Evolution API | 200 MB | 400 MB |
| Chatwoot (app) | 500 MB | 900 MB |
| Chatwoot (Sidekiq) | 300 MB | 500 MB |
| chatwoot_pg | 200 MB | 400 MB |
| chatwoot_redis | 50 MB | 100 MB |
| Typebot builder | 400 MB | 700 MB |
| Typebot viewer | 200 MB | 400 MB |
| typebot_db | 150 MB | 300 MB |
| Twenty CRM | 500 MB | 800 MB |
| twenty_db | 200 MB | 400 MB |
| twenty_redis | 50 MB | 100 MB |
| Dify API | 600 MB | 1000 MB |
| Dify Worker | 400 MB | 700 MB |
| Dify Web | 200 MB | 400 MB |
| dify_db | 200 MB | 400 MB |
| Listmonk | 100 MB | 200 MB |
| listmonk_db | 150 MB | 300 MB |
| Mautic | 400 MB | 700 MB |
| mautic_db (MySQL) | 400 MB | 700 MB |
| MPT Wrapper (idle) | 200 MB | 4000 MB (during generation) |
| **TOTAL IDLE** | **~5.6 GB** | **—** |
| **TOTAL PEAK** | **—** | **~13.5 GB** |

Oracle VPS has 24 GB. MPT generation temporarily spikes usage to ~17 GB.
This leaves ~6 GB headroom during video generation.
The `deploy.resources.limits.memory: 4G` on MPT_wrapper is the critical
safeguard preventing OOM during generation.

---

*End of Document 6: Implementation Plan*

*This document, combined with Documents 1–5, constitutes the complete
Uddokta AI-Agent-Ready Blueprint. Begin execution at Phase 1, Step 1.1.1.*
