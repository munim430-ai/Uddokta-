# Document 4 — Uddokta UI/UX Design Brief

I have full context of the Uddokta blueprint. Now I'll craft Document 4: UI/UX Design Brief — strictly structural, technical, and psychological, with no code or CSS classes, aligned with all constraints (mobile-first PWA, premium minimalist matte black/white aesthetic, Bangla/English, low-end Android sunlight legibility, thumb-zone optimization, wrapping Chatwoot/n8n/Typebot/Twenty headlessly).

---

> A foundational, hallucination-free UI system specification for AI coding agents (Claude Code, Cursor, Windsurf). This document defines visual language, layout law, screen hierarchy, state logic, and behavioral psychology — **not** code, not CSS, not mockups.

Stack context (already locked): Next.js (PWA) · shadcn/ui · Tailwind CSS · Supabase · headless Chatwoot · n8n · Typebot · Twenty CRM.

Brand promise being designed for: *"Turn a messy Facebook/WhatsApp inbox into a mobile order-control system."*
> 

---

## 0. Design North Star

Every screen, component, and microcopy decision in Uddokta must satisfy four non-negotiable filters. If a design choice fails any one filter, it is rejected.

| Filter | Question the designer/agent must answer |
| --- | --- |
| **Sunlight Legibility** | Can a rickshaw-puller-turned-seller read this card outdoors on a 5.5" cracked screen at 11 AM in Dhaka? |
| **Thumb Reach** | Can every critical action be triggered with one right thumb while the left hand holds the phone? |
| **Cognitive Floor Zero** | Would a seller who has never opened a SaaS dashboard understand what to tap *within the first 8 seconds* — without watching a tutorial? |
| **Trust Projection** | Does this view make the seller's business look more credible to the buyer than a screenshot in a Messenger chat? |

The product is not "software"; it is a *professional uniform* for informal sellers. The UI is the uniform.

---

## 1. Design System & Brand Visual Identity

### 1.1 Aesthetic Direction — "Premium Stillness"

Uddokta's visual language is intentionally **quiet, dense in meaning, low in decoration**. The reference emotional benchmark is the interior of a high-end watch boutique: matte surfaces, generous breathing room, one focal object per moment, no neon.

This aesthetic does *deliberate psychological work*:

- **Anti-agency signal.** Bangladesh's SME tech market is saturated with rainbow-gradient agency dashboards. By choosing restraint, Uddokta auto-positions above the noise — sellers perceive it as "this is what real companies use."
- **Pride uplift.** When a Mirpur-based skincare seller opens Uddokta, the interface should feel *one socioeconomic tier above her current self-perception*. Aspirational tools change behavior more than functional ones.
- **Calm under chaos.** The seller's inbox is screaming. The dashboard must be the opposite. Stillness is a UX feature, not a stylistic choice.

**Aesthetic vocabulary (binding):**

- Matte, never glossy. Zero glassmorphism. Zero gradients on primary surfaces.

- Edges are crisp. No blurred shadows used for "depth" — only single-direction, low-opacity hairlines or 1-step elevation.

- One focal element per screen. The eye should never have to choose where to look first.

- Iconography is monoline, geometric, uniform stroke weight. No filled illustrative icons. No emoji used as UI affordance.

- Imagery (product photos, logos) appears inside a calm matte frame — never bleed-to-edge.

### 1.2 Color Palette — Sophisticated Matte System

The palette is built on a **two-color foundation** (matte black + warm white) with **three semantic accents** and **one neutral utility ramp**. No other colors are permitted in the product UI without a written exception.

| Token Role | Purpose | Psychological Job |
| --- | --- | --- |
| **Matte Black (Surface Ink)** | Primary text, primary buttons, header bars, active states | Authority, seriousness, "real software" cue |
| **Warm White (Canvas)** | Main background of every screen | Sunlight legibility, mental quiet, premium boutique feel |
| **Soft Card Gray** | Card backgrounds, dividers, skeleton fills, disabled states | Hierarchy without color, lets accents pop |
| **Emerald (Success Accent)** | Successful payment, confirmed order, delivered status, "online" indicator | Reassurance, "money received" reflex — chosen over generic green to feel more institutional than playful |
| **Deep Indigo (Active Chat Accent)** | Active conversation, unread thread, current staff handler badge, AI/auto-reply marker | Focus, conversational presence — borrows from Messenger/WhatsApp muscle memory without copying their saturated blues |
| **Muted Clay/Amber (Caution Accent)** | Follow-up due, payment pending, COD risk flag, slow staff response | Attention without panic. Deliberately *not* yellow (which reads as cheap/agency) and *not* red (which reads as failure) |
| **Reserved Crimson (Critical Only)** | Failed payment, cancelled order, deleted record, destructive confirmation | Used **sparingly** — fewer than ~5 surfaces in the entire product. Its rarity is its power. |

**Color rules:**

- The matte black/white duo must visually dominate ≥ 80% of any screen's pixel area.

- Accents are *signals*, not decoration. An accent color on a surface always means "something happened here / something needs you."

- No accent may be used to color a card background. Accents live only in: text, dot indicators, status pills, icon glyphs, single-purpose action buttons.

- Dark mode is a **future deliverable, not v1**. Designing in dark first weakens sunlight-mode optimization, which is the actual operating environment.

### 1.3 Typography — Bangla/English Parity

Typography must render *both Latin and Bangla scripts cleanly across the cheap Android device fleet*. This rules out most "designer" sans-serifs that ship without Bangla glyphs or fall back to ugly system fonts mid-paragraph.

**Selection criteria (binding for the type team):**

- Native Bangla glyph coverage with proper conjunct handling (যুক্তাক্ষর) — no fallback to Noto rendering inconsistencies.

- Latin variant from the same designer/foundry so x-height, weight, and color match when languages are mixed in a single sentence ("নতুন order এসেছে — 3 New Orders").

- Variable weight axis (Regular → Semibold → Bold) so the UI uses **weight, not size**, to express hierarchy. Size jumps on small screens cause layout fatigue; weight jumps do not.

- Acceptable rendering down to 13px on 720p Android panels.

**Hierarchy law:**

- Screen title: heaviest weight, calm size — never larger than necessary.

- Card title (e.g., order number, customer name): semibold.

- Body (message preview, address): regular.

- Numerals for money are rendered in tabular figures so columns of ৳ amounts align without the eye doing arithmetic.

- **No italics.** Bangla italics are visually broken on Android system renderers; banning italics also enforces stylistic discipline.

**Language toggling rule:**

- Bangla and English versions of any string must be designed to occupy *the same vertical space* (max ±1 line). Layouts must be tested in both languages before sign-off; a card that looks elegant in English but breaks into 3 lines in Bangla is a defective design.

- A persistent language toggle lives in Settings and as a one-tap header chip on the first-run home screen. After that, language is sticky — never asked again.

### 1.4 Iconography, Imagery, Motion

- **Icons:** one shared monoline family, 24px nominal, 1.5px stroke. Icons are paired with text labels in all primary navigation — never icon-only (low literacy guard).
- **Imagery:** seller-uploaded product photos are auto-fit into a fixed square frame on a soft gray backdrop. Uddokta never shows stock photography of "happy shopkeepers" — that is agency aesthetic.
- **Motion:** motion exists only to communicate *state change* (a card moves between Kanban columns, a drawer slides up, a skeleton resolves into content). No decorative animation. No parallax. No bounce. Motion durations stay short — long animations on 3G feel like the app is frozen.

---

## 2. Layout, Navigation, and Ergonomics

### 2.1 Thumb-Zone Law

Designed against the dominant Bangladeshi mobile reality: **6.5"+ Android phones, held in the right hand, used standing in a shop or on a rickshaw**. The thumb arcs naturally over the lower half of the screen; the upper third is "stretch territory."

Binding zone rules:

| Screen Region | Allowed Content | Forbidden Content |
| --- | --- | --- |
| **Bottom strip (sticky nav)** | The five core pillars | Anything else |
| **Lower 50% (Hot Zone)** | All primary CTAs, swipe targets, send button, "Convert to Order" FAB, status change actions | Decorative content, secondary info |
| **Middle 30% (Scan Zone)** | Card lists, conversation feed, order cards | Critical actions |
| **Top 20% (Stretch Zone)** | Screen title, contextual info, search, filter, back arrow | "Save," "Send," "Confirm" — anything destructive or commit-level |

**Tap target minimum: 48×48 dp.** Smaller targets exist only for non-critical actions (e.g., dismissing a non-blocking toast). Sellers using cheap touchscreens with calibration drift fail on 36px targets — this is observed, not assumed.

### 2.2 Sticky Bottom Navigation — The Five Pillars

Exactly five tabs. No "more" menu. No drawer hamburger hiding core features. If a feature is important enough to exist, it lives inside one of these five pillars; if it doesn't fit, it gets cut.

| Order | Pillar | Mental Model | What's Inside |
| --- | --- | --- | --- |
| 1 | **Inbox** | "Where customers are talking to me" | Unified conversations across Messenger, WhatsApp, Instagram (via headless Chatwoot) |
| 2 | **Orders** | "What I'm fulfilling today" | The Kanban order board |
| 3 | **Customers** | "Who I'm selling to" | Customer database (Twenty CRM headless) |
| 4 | **Products** | "What I sell" | Catalog with quick-reply templates |
| 5 | **Reports** | "How my business is doing" | Weekly snapshots, shareable PDFs |

Each tab uses **icon + Bangla/English label**. The active tab is communicated by the matte black ink color and a small indicator dot — never by a saturated highlight band.

**Floating Action Button (FAB):** A single, persistent matte black circular FAB sits in the lower-right thumb arc. Its function is *contextual*:

- In Inbox → "Convert to Order"

- In Orders → "New Order"

- In Customers → "Add Customer"

- In Products → "Add Product"

- In Reports → "Share Report"

The FAB never opens a menu. One screen, one FAB action. This eliminates decision paralysis.

### 2.3 Data Layout Paradigm — Cards, Not Tables

Tables are banned on mobile screens. Period. Tables cause horizontal scroll, tiny tap targets, and "spreadsheet anxiety" — which directly contradicts the brand promise of replacing the seller's chaotic Google Sheet.

Every data unit is a **card**:

- One card = one entity (one order, one conversation, one customer, one product).

- Each card surfaces 3–5 fields maximum on first glance. Everything else is one tap away in a detail drawer.

- Cards support **horizontal swipe actions**:

  - Order card: swipe right → advance status (New → Confirmed). Swipe left → reveal "Call / WhatsApp / Move."

  - Conversation card: swipe right → mark resolved. Swipe left → reveal "Assign / Snooze / Pin."

  - Customer card: swipe left → reveal "Call / WhatsApp."

- Swipe actions are **paired with redundant tap-to-open menus**. Swipes are an accelerator for repeat users, never the only path.

- Pull-to-refresh is supported on every list. Sellers expect this from WhatsApp.

### 2.4 Zero-Jargon Copy Rules

The UI is a **branded wrapper over Chatwoot, n8n, Typebot, and Twenty**. The seller must never see the seams. Strict copy substitution table:

| Forbidden (developer/SaaS word) | Required UI term (EN) | Required UI term (BN) |
| --- | --- | --- |
| API / Endpoint | Connection | কানেকশন |
| Webhook | Auto-sync | অটো-সিঙ্ক |
| CRM | Customer list | কাস্টমার লিস্ট |
| Workflow / Automation | Auto-work / Business rule | অটো-কাজ |
| Tenant / Workspace | Business account | ব্যবসার অ্যাকাউন্ট |
| Pipeline | Order board | অর্ডার বোর্ড |
| Ticket | Customer issue | কাস্টমার সমস্যা |
| Conversion rate | How many messages became orders | কতগুলো মেসেজ থেকে অর্ডার হয়েছে |
| Agent | Staff / Team member | স্টাফ |
| Inbox (technical) | ইনবক্স গুছানো (curated inbox) | — |
| Order creation from chat | মেসেজ থেকে অর্ডার | — |

**Microcopy laws:**

- Imperatives, not gerunds. "Send receipt," never "Sending receipts."

- Money first, label second. "৳1,850 — COD," never "Payment method: COD, Total amount: ৳1,850."

- Time is humanized in Bangla relative form ("৫ মিনিট আগে," "এইমাত্র") never ISO timestamps in the surface UI.

- Every error message ends with **a verb the user can tap**. Never a dead-end "An error occurred."

---

## 3. Core Screen Breakdowns & Component UX

### 3.1 The Mobile Control Dashboard (Home) — *"Today's Control"*

**Mental model the seller arrives with:***"Did I lose any money since I last opened this?"*

**Layout hierarchy (top to bottom):**

1. **Greeting strip (Stretch Zone).** Time-aware greeting + business name. Hosts the language toggle on first run only. Quiet, never loud.
2. **The Pulse Row** — four matte cards in a 2×2 grid, each surfacing one business-health vital:
3. *New Messages* (count + indigo dot if any are unread > 30 min)
4. *Pending Orders* (count + clay dot if any are stuck > 24 hr)
5. *Today's Estimated Revenue* (large tabular ৳ figure)
6. *Follow-ups Due* (count + tap → opens follow-up queue)
7. **The Attention Strip** — a single horizontal-scroll row of "things only the owner can do right now": *Verify payment proof · Approve outbound campaign · Reassign cold conversation*. If nothing is pending, the entire strip is replaced by a single calm message: *"All clear. Your shop is in control."*
8. **Quick Actions tile** — three large tap tiles in the Hot Zone: *Add Order · Send Follow-up · Create Receipt*. These exist on Home in addition to the FAB because they are the three actions sellers want when *not* coming from a specific conversation.

**Anti-patterns explicitly banned:**

- No line charts on Home. (Sellers don't read line charts under sunlight; they want counts.)

- No "Welcome to your dashboard" cards after first session.

- No notifications bell with a number badge — notifications surface inline as the Attention Strip, where they have context.

**Psychological design intent:** Home is a **calm mirror of business health, not a control surface.** The seller should feel "I see everything in 4 seconds" — not "I have 47 things to do."

---

### 3.2 The Headless Unified Inbox Proxy — *"All inbox in one place"*

This screen is the heart of the product. It wraps Chatwoot conversations + Meta/WhatsApp webhooks into a single thread experience that *imitates WhatsApp's familiarity* while adding seller-side intelligence.

**Inbox list view:**

- Each conversation is a card with: customer name (or phone if name unknown), channel glyph (small monoline Facebook/WhatsApp/Instagram icon), last message preview (1 line, truncated), relative timestamp, and a single status indicator dot.

- Dot color taxonomy:

  - Indigo = unread, active

  - Emerald = order already created from this thread (the win state)

  - Clay = follow-up overdue

  - Gray = resolved

- Cards belonging to the same customer across channels (WhatsApp + Messenger) are visually **stitched** with a thin matte hairline so the seller perceives one buyer, not three.

- The assigned staff initial appears as a tiny circular avatar at the card's bottom-right — present but never visually loud (see §4.3 Staff Accountability).

**Conversation detail view (the chat screen):**

Layout, top to bottom:

1. **Sticky header (Stretch Zone):** Customer name, channel glyph, back arrow. A single chevron pulls down the **Customer Profile Drawer** (§3.4) — this is the seller's "professional memory."

2. **Chat thread:** WhatsApp-style bubbles. Inbound = warm white on soft gray. Outbound = matte black on warm white. Auto-replies are visually distinguished by a small indigo "auto" marker — sellers must always know which messages they did *not* personally write.

3. **Quick-Reply Drawer (slides up from below):** A horizontally scrollable strip of seller-curated templates: *Price · Available · Delivery Charge · Return Policy · Send Order Form · Ask Address · Ask Payment*. Each chip injects a personalized message (using `{{customer_name}}`, `{{product_name}}`, `{{price}}`) into the composer for one-tap edit-and-send. Backed silently by Typebot flows for structured intake.

4. **Product-Picker Drawer (alternate slide-up):** Tap a product → its image, price, variants, and a pre-written quick reply auto-fill the composer.

5. **The "Convert to Order" FAB:** Persistent, matte black, lower-right. One tap opens the **Order Sheet** (a bottom sheet, not a new page — the seller never leaves the conversation). The sheet pre-populates customer name, phone, and any address detected in the recent chat history. The seller confirms variant/quantity, taps "Create Order," and the sheet collapses with a discreet emerald confirmation chip: *"Order #UDD-1048 created."*

6. **Composer (Hot Zone):** Text input + attach + send. Send button is the largest tap target on the screen.

**Behavioral design intent:**

- The seller's existing muscle memory from WhatsApp is preserved (bubble layout, swipe-to-reply gesture, send button placement).

- The *only new behavior to learn* is the FAB — and it does the single most valuable thing the product offers (capture revenue from a conversation).

- Auto-replies are visually transparent because **the seller must trust the bot.** Hidden automation breeds anxiety; marked automation breeds delegation.

---

### 3.3 The Kanban Order Board — *"অর্ডার বোর্ড"*

A horizontally scrollable column board, optimized for one-thumb traversal.

**Column structure (left → right, mirroring real-world order lifecycle):**

`New → Confirmed → Payment Pending → Ready to Ship → Shipped → Delivered` with off-board branches for `Returned` and `Cancelled` (accessible via a "Closed Orders" filter chip at the top, never in the main flow — closed orders are not where attention belongs).

**Interaction model:**

- Columns are **horizontally swipeable** at the screen level (swipe the whole board left/right to traverse).

- Individual order cards are **vertically scrollable within a column**.

- An order card is moved between statuses two ways:

  1. **Long-press + drag** (for deliberate moves).

  2. **Swipe right on the card** → advance to next status (the dominant flow — one thumb, one stroke, status advances).

- A subtle haptic on status change plus a 1-second emerald chip confirms ("Moved to Ready to Ship · Undo").

- "Undo" remains tappable for 5 seconds in the Hot Zone. After that, status changes are committed and logged in `audit_events`.

**Order card content (top to bottom, scannable in <2 seconds):**

- Order number (e.g., `#UDD-1048`) in semibold matte black — the seller's anchor identifier.

- Customer name + small area label (e.g., "Mirpur, Dhaka").

- Money line: `৳1,850` (tabular, large) + payment method pill (`COD` / `Paid` / `Advance`).

- Product summary, single line: "Skincare Set x1."

- Courier status chip if shipped (e.g., "Steadfast — pending").

- Action row in the Hot Zone of the card: Call · WhatsApp · Move.

- **Staff handler avatar** (small, bottom-right corner — see §4.3).

**Column header design:**

- Column title + count badge.

- A faint vertical hairline separates columns — no heavy borders.

- Column backgrounds remain warm white. **Columns are never color-coded** — color carries meaning at the *card* level, not the column level, to keep the board calm.

**Psychological design intent:** The Kanban is the **physical proof that chaos has ended.** Many sellers will spend their first session simply swiping fake orders left to right just to feel the satisfaction of motion = control. The interaction must reward that.

---

### 3.4 The Customer Profile Drawer — *"Professional Memory"*

Triggered by tapping the chevron in any conversation header, or the customer name on any order card. Slides up from the bottom as a **bottom sheet covering ~70% of the screen height** — keeps the conversation context visible above, so the seller never feels "moved away."

**Information hierarchy (top to bottom):**

1. **Identity block:** Customer name, phone (with one-tap call + WhatsApp glyphs), preferred address (latest used).
2. **Trust scoreboard** — three matte chips in a single row:
3. *Lifetime Value:* total ৳ spent (emerald if > median, gray otherwise).
4. *Order Count:* number of confirmed orders.
5. *COD Return Risk:* "Low / Medium / High" pill (clay or crimson if non-low). Risk is computed server-side from return history; the UI never exposes the formula — it exposes the **judgment**, not the math.
6. **Last interactions feed:** Reverse-chronological list of the customer's last 3–5 orders and the last conversation summary. Each is tappable to deep-link into the order or thread.
7. **Tags & notes:** Seller-added tags ("VIP," "Whatsapp only," "Delivers via Pathao only") and a free-form notes field that persists across sessions.
8. **Risk flags (only if present):** "2 previous COD returns," "Phone reported invalid," "Disputed order in last 30 days." These appear only when true; their absence is informative.

**Behavioral design intent — the "split-second pre-reply panel":**

The drawer is engineered to be opened *before* the seller replies. The seller's instinct in chaos is to type immediately; Uddokta's instinct must be to **show the past first**. After 2–3 weeks of use, sellers reflexively chevron-pull before every reply — this is the "professional memory" habit the product is teaching. The drawer dismiss animation is intentionally short (~150 ms) so it doesn't punish the habit.

---

### 3.5 The Public Trust Receipt & Profile (`/public/*` routes)

These pages are the *buyer-facing* face of Uddokta. They exist to do one job: **convert the buyer's "this might be a scam" reflex into "this is a real shop."** They are designed under different visual rules than the seller-facing app, but in the same design language.

### 3.5.1 `/public/receipt/[orderNumber]` — The Digital Invoice

Layout, top to bottom:

1. **Header band:** Seller's logo (or generated monogram if none), business name, "Order Confirmation" label in matte black. The "Powered by Uddokta" mark is present but small and at the very bottom — the buyer must perceive *the seller's brand*, not Uddokta's.

2. **Order ID + Date + Status:** Large, tabular, undecorated. The status pill (Confirmed / Shipped / Delivered) uses the same color taxonomy as the seller-side UI for consistency.

3. **Itemized list:** Product photo (square, framed), name, variant, quantity, unit price, line total. Each row is a card; no table grid lines.

4. **Cost summary:** Subtotal, delivery fee, discount, **Total** in heaviest weight. Tabular alignment.

5. **Customer info:** Name, phone, delivery address — exactly as the seller has it. Discrepancies are the buyer's prompt to correct them ("Wrong address? Message the seller").

6. **Policy strip:** Three matte chips, side by side — *Delivery Policy · Return Policy · Payment Policy*. Each expands inline on tap. Buyers historically don't read policies, but the *visible presence* of structured policies is itself a trust signal, independent of being read.

7. **Action row (Hot Zone):***Message Seller on WhatsApp · Call · Track Parcel* (if courier assigned).

8. **Verification footer:** "This receipt was issued on [date] by [business name] via Uddokta. Verify at [short URL]." A tiny QR code option appears here for sellers who print receipts.

**Design law:** The receipt page must look identical whether viewed on a buyer's phone, the seller's phone, or a courier's phone. Pixel-perfect cross-device alignment is itself the trust signal.

### 3.5.2 `/public/trust/[slug]` — The Shop Profile

The seller's public storefront-of-trust. Not a webshop — a **credibility page** the seller links from their Facebook bio.

Layout:

1. **Identity hero:** Logo + business name + a single tagline (seller-authored, max 80 characters). No carousel. No animation.

2. **Verified badges row:** "Verified phone," "Verified bKash," "X orders fulfilled" (visibility controlled by `visible_order_count` in `trust_profiles`). Badges are matte outlines with a small emerald check — never gold/colorful.

3. **Policy cards:** Delivery, Return, Payment — same structure as the receipt's policy strip, expanded.

4. **Verified reviews:** Cards with star rating, comment, masked customer first name + last-name initial, date. Only reviews approved by the seller (per Workflow G) appear. The total review count and average rating sit above the list as a single calm summary, not a giant star graphic.

5. **Support & contact:** WhatsApp, phone, Facebook page link. One-tap each.

6. **"Order Confirmation Lookup":** A single input — buyer enters their order number → sees their receipt. Removes the seller's burden of resending receipt links.

**Psychological design intent:** A skeptical Bangladeshi buyer arriving from a Facebook ad must, within 5 seconds of scrolling, perceive: *a logo, a real address, verified payment methods, real reviews, a real human to call*. The page does not "sell" — it *certifies*. Restraint is the proof.

---

## 4. System States & Interaction Logic

### 4.1 Empty States — Coaching, Not Decoration

Every empty state in Uddokta is treated as a **first-use lesson**, not a placeholder. The user is, by definition, a non-technical seller who just discovered a feature exists. The empty state's job is to turn confusion into the first successful action.

**Empty state structural law (binding):**

1. **One sentence of empathy** in Bangla-first, plain language.

2. **One concrete next action** — a single tappable button in the Hot Zone, labeled with an imperative verb.

3. **One supporting hint** — a short line explaining what will happen after they tap.

4. **No illustrations of empty boxes, sad faces, or "no data" graphics.** A small monoline glyph at most.

| Screen | Empty State Copy (English / Bangla) | Primary Action |
| --- | --- | --- |
| Products | "No products yet. Add your first item so you can send price replies in one tap." / "এখনো কোনো প্রোডাক্ট নেই। প্রথম প্রোডাক্ট অ্যাড করুন।" | *Add First Product* |
| Orders | "No orders yet. Create an order manually, or convert a chat message into one." / "এখনো কোনো অর্ডার নেই।" | *Create Order* (secondary: *Open Inbox*) |
| Customers | "Your customer list is empty. Every reply you send will start building this." / "কাস্টমার লিস্ট ফাঁকা।" | *Add Customer Manually* |
| Inbox | "No messages yet. Connect your Facebook page or WhatsApp number to start receiving." / "এখনো কোনো মেসেজ আসেনি।" | *Connect a Channel* (opens the simplified connection wizard) |
| Reports | "Your first weekly report will appear here after your first order. Want a sample?" | *View Sample Report* |
| Follow-ups | "Nothing to follow up on right now. Good work." / "এই মুহূর্তে কোনো ফলোআপ নেই। ভালো কাজ।" | (no action — affirmation only) |

The follow-ups example is intentional: empty states can also **reward** the seller for being caught up. This is rare in SaaS and creates emotional attachment.

### 4.2 Loading & Network Resilience UI

Bangladesh's 3G/4G is patchy by neighborhood, hostile in moving vehicles. The UI is designed to **never appear broken**, only "loading" or "saved for later."

**Loading states:**

- **Skeleton screens** for all list views (Inbox cards, Order cards, Customer cards). Skeletons match the exact shape of the eventual card — no spinning circles. Sellers perceive skeletons as "the app is working" rather than "the app is stuck."

- Skeletons use the soft card gray, with a slow, low-contrast shimmer (never aggressive).

- Initial app boot on a cold cache must render the bottom nav and a skeleton Home within ~1 second — before data arrives — so the seller sees structural commitment immediately.

- Spinners are reserved for *modal* actions where the user explicitly waits (e.g., generating a PDF report). They never block a whole screen.

**Network resilience states:**

- **Offline banner:** A single thin matte black strip at the very top of the screen reads *"No internet — your actions will sync when you're back online"* (Bangla equivalent). Non-dismissable while offline; auto-vanishes on reconnect.

- **Optimistic UI on commits:** Order status changes, sent messages, new customer entries appear instantly in the UI with a small clock glyph indicating "queued." On successful sync, the clock turns into an emerald check. On failure, it turns into a clay flag with a *Retry* tap.

- **Send-while-offline contract:** Messages composed offline are visually queued in the chat thread (in soft gray with the clock glyph). They are *never silently dropped*.

- **Image loading:** Product/customer images use progressive low-quality placeholders that resolve. On flaky networks, a single tap on a failed image retries — no error toast.

- **Stale-data marker:** If the seller is on a screen that hasn't refreshed in > 5 minutes due to network loss, a discreet "Last updated 6 min ago — pull to refresh" appears at the top of the list.

**Behavioral design intent:** The seller must believe that **tapping always works**, even when the network doesn't. Trust in the tool degrades catastrophically if a confirmed order silently disappears.

### 4.3 Staff Accountability Triggers

The product solves a real seller pain: *"I don't know who on my team is replying to whom."* But making staff identity visually loud would clutter every card and create surveillance anxiety on the staff side. The design resolves this with **subtle, omnipresent, non-judgmental cues.**

**The Handler Avatar pattern:**

- Every conversation card, every order card, every customer interaction surfaces a **small circular initial-avatar** (24px) in a fixed corner position (typically bottom-right of the card). Color of the ring is generated deterministically from the staff member's `user_id` — consistent per staff, never random.

- "Unassigned" = empty dashed ring. The visual incompleteness creates a natural pull for the owner to assign someone.

- The avatar is tappable → reveals a tiny tooltip: *"Handled by Rahim · last reply 2 min ago."*

**The Live Typing Cue (Inbox only):**

- When another staff member is currently typing in the same conversation, a thin indigo line pulses gently under the conversation card in the inbox list and at the top of the chat detail. This prevents two staff members from double-replying — a frequent source of customer confusion in Bangladeshi SME teams.

**The Response Time Whisper:**

- If a staff member's average response time on conversations they're assigned to is degrading week-over-week, the owner's Reports screen surfaces it gently: "Rahim's average reply time is slower this week than last." No public ranking. No leaderboard shame.

- Staff see their own metrics in their personal report — never each other's.

**The Owner Lens:**

- The owner role has a discreet toggle in Settings: *"Show staff handler on every card."* When off, avatars disappear from staff users' views, reducing visual noise for the people doing the work. The owner can still see them.

**Psychological design intent:** Accountability without surveillance theater. Staff feel trusted; owners feel informed; customers experience consistent service. The design assumes the staff member is competent and the owner is busy — not the inverse.

---

## 5. Cross-Cutting UX Laws (Binding for AI Coding Agents)

These rules apply to every screen and every component. AI coding agents (Claude Code, Cursor, Windsurf) must treat violations as build failures.

1. **No screen ships with more than one primary CTA.** If two actions feel equally important, the design is wrong — one must become secondary.
2. **No setting screen for things that should be invisible.** The seller never configures a webhook, a workflow trigger, an n8n endpoint, or a Chatwoot agent ID. If those need to exist, they live in the admin app, not the seller app.
3. **No data is shown without a unit and a context.** "৳1,850 — Today" is correct; "1850" is forbidden.
4. **No destructive action without a confirmation that names the entity.** "Delete this product?" is wrong. "Delete *Skincare Set*?" is correct.
5. **No success toast without a verb the user just performed.** "Order #UDD-1048 created" is correct. "Success" is forbidden.
6. **No icon without a text label in primary navigation.** Tooltips do not count for non-literate users.
7. **No screen exists that cannot be reached within 3 taps from Home.** If something requires 4+ taps, it is misplaced.
8. **No Bangla translation is allowed to be a literal English translation.** Bangla copy must be drafted Bangla-first by a native speaker — English is the secondary deliverable.
9. **No surface in the seller app may carry the names "Chatwoot," "n8n," "Typebot," "Twenty," or "Supabase."** These tools are infrastructure, never brand.
10. **Every screen must answer one question.** If a designer cannot state in one sentence what question this screen answers for the seller, the screen is not ready to be built.

---

## 6. Handoff Contract to AI Coding Agents

When generating UI from this brief, agents must:

- Treat the **five-pillar bottom nav, the Kanban order board, the conversation FAB, and the Customer Profile Drawer** as the four anchor patterns. All other screens are downstream of these four.
- Generate every component with **Bangla and English string slots**, never English-only with "translate later" comments.
- Default to **shadcn/ui primitives** with the matte black/white tokenization defined in §1.2 — but never hardcode color values inline. All color references go through the design token layer.
- Build **skeleton variants and empty-state variants for every list and detail screen** as first-class components, not afterthoughts.
- Implement **swipe gestures and tap-to-open menus in parallel**, never one without the other.
- Add the **handler avatar slot** to every entity card from the start — even before the staff assignment logic exists. Empty dashed-ring placeholders are acceptable scaffolding.
- Wire the **"Powered by Uddokta" mark** only on `/public/*` routes, in the footer, at the smallest acceptable legible size.

Any component that cannot be confidently mapped back to a section of this brief is not in scope and must be flagged for design review before implementation.

---

## 7. Final Design Philosophy Statement

Uddokta is not a dashboard. It is a quiet promise — to a small seller in Mirpur, Mohakhali, Cumilla, Sylhet — that her business can finally **look** as serious as it already **is**. The UI's job is not to add features. The UI's job is to remove the chaos that currently sits between her phone and her customer.

Every matte surface, every emerald confirmation, every swipe-to-confirm gesture, every empty state that gently coaches the next step — all of it works toward one feeling:

> **"This is real software. This is real business. I am a real entrepreneur."**
> 

Design for that sentence. Build against this brief.

---

*End of Document 4 — UI/UX Design Brief.*