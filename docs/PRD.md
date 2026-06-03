# Uddokta Product Requirements Document (PRD)

**Product:** Uddokta  
**Document:** Product Requirements Document  
**Version:** v1.0  
**Generated:** 2026-06-03  
**Status:** Builder-ready draft  

---

## Source Integrity Notice

This PRD is based on the uploaded **Uddokta Business Blueprint**, the founder-provided feasibility direction, and current public sources. It avoids unsupported claims about exact Bangladesh BotSailor users, guaranteed revenue, guaranteed conversion lifts, or guaranteed willingness to pay. Where public data is weak or unavailable, assumptions are clearly marked.

---

# 1. Executive Summary

## 1.1 What Uddokta is

**Uddokta** is a mobile-first sales, order, follow-up, customer-record, and trust-control platform for Bangladeshi SMEs that sell through Facebook, Messenger, WhatsApp, Instagram, comments, screenshots, bKash/Nagad payment proof, COD delivery, and manual staff communication.

Uddokta is **not** a generic chatbot, digital marketing agency, BotSailor reseller, or white-label SaaS. It is a proprietary customer-facing wrapper and operational system built by combining custom product design with open-source/public infrastructure such as Next.js, Supabase/PostgreSQL, n8n, Chatwoot, Typebot, official WhatsApp Cloud API, Google Sheets export, and later local courier/payment integrations.

The customer should experience Uddokta as a simple mobile control panel:

- Inbox
- Orders
- Customers
- Products
- Auto replies
- Follow-ups
- Reports
- Trust profile
- Settings

The customer must not see API keys, webhooks, n8n, Chatwoot, Typebot, Supabase, Meta developer settings, server logs, or technical setup.

## 1.2 Who it is for

The first users are Bangladeshi F-commerce, e-commerce, and social-commerce sellers, especially:

- Premium cosmetics and skincare sellers
- Gadget/accessory sellers
- Active Facebook-page sellers
- WhatsApp-heavy sellers
- Sellers using COD courier delivery
- Sellers using bKash/Nagad screenshots or bKash PRA
- Sellers with staff replying manually from phones

The first wedge should avoid low-margin, low-volume sellers who demand heavy support but resist monthly fees.

## 1.3 What problem it solves

Uddokta solves two bundled problems.

### Problem 1: Sales/order leakage

Customers message “price?”, “available?”, “size?”, “delivery charge?”, “original?”, or “warranty?” but sellers reply late or forget follow-up. Orders remain buried in Messenger, WhatsApp, staff phones, screenshots, notebooks, courier portals, or Google Sheets. Owners cannot clearly see how many inquiries became orders, which staff handled which customer, or where the sale was lost.

### Problem 2: Trust and proof failure

Buyers distrust informal Facebook sellers, and sellers lack structured proof. Payment, invoice, courier, refund, review, and complaint records are scattered across screenshots and chats. COD return/fake-order risk is high. Sellers need customer history, digital receipts, review collection, issue records, and public trust pages.

## 1.4 Why Bangladesh is the right market

Bangladesh is mobile/social-first. DataReportal’s Digital 2026 Bangladesh report states that Bangladesh had **186 million cellular mobile connections**, **82.8 million internet users**, and **64 million social media user identities** in late 2025. It also reports **64 million Facebook ad-reachable users**, with Facebook reach equivalent to **77.3% of the local internet user base**. [S1]

bKash states that its **Personal Retail Account** is designed for micro and small businesses operating in **retail and f-commerce** to collect bKash payments from customers. [S3]

Meta’s WhatsApp Business Platform is officially positioned for lead generation, conversational commerce, order confirmations, shipment updates, cart abandonment re-engagement, product browsing, order placement, automated flows, routing, and integration with back-end systems. [S2]

## 1.5 Why this should not be positioned as a chatbot

The term “chatbot” is too narrow and weak for the buyer psychology. Bangladeshi SME owners want visible control and trust, not an abstract AI bot. Uddokta’s core positioning should be:

> **Uddokta turns messy Facebook, WhatsApp, and Instagram selling into a clean mobile sales, order, follow-up, and trust system.**

Chat automation is a feature. **Order control and trust proof** are the product.

---

# 2. Market Evidence

## 2.1 Bangladesh mobile and social-commerce environment

| Metric | Figure | Source |
|---|---:|---|
| Cellular mobile connections | 186 million | DataReportal Digital 2026 Bangladesh [S1] |
| Internet users | 82.8 million | DataReportal Digital 2026 Bangladesh [S1] |
| Internet penetration | 47.0% | DataReportal Digital 2026 Bangladesh [S1] |
| Social media user identities | 64.0 million | DataReportal Digital 2026 Bangladesh [S1] |
| Facebook ad reach | 64.0 million | DataReportal Digital 2026 Bangladesh [S1] |
| Facebook reach vs internet users | 77.3% | DataReportal Digital 2026 Bangladesh [S1] |
| Instagram ad reach | 9.15 million | DataReportal Digital 2026 Bangladesh [S1] |
| TikTok adult ad reach | 56.2 million | DataReportal Digital 2026 Bangladesh [S1] |

**Product implication:** Uddokta should start with Facebook/Messenger/WhatsApp behavior, not a website-first model. Instagram and TikTok matter for marketing and later channels, but Facebook and WhatsApp are the operational core.

## 2.2 SME/MSME scale

The uploaded feasibility base states that Bangladesh has approximately **7.81–7.82 million MSMEs**, with retail, trade, and services dominating the SME economy. Treat this as directionally useful but verify against primary BBS/SME Foundation data before investor-grade use.

**Assumption:** Even if the exact MSME count varies by source, the market is large enough that niche prioritization matters more than total-addressable-market inflation.

## 2.3 F-commerce/e-commerce relevance

The uploaded feasibility base identifies F-commerce as a major informal digital-commerce channel in Bangladesh. Public sources consistently recognize Facebook-based commerce and social-commerce behavior in Bangladesh, but exact live counts of active F-commerce pages vary and are not reliably published in one authoritative public source.

**Data integrity note:** No reliable current public source was found that definitively verifies the exact number of active F-commerce pages in Bangladesh. Do not use a precise F-commerce page count in public pitch decks unless verified.

## 2.4 WhatsApp/Facebook/Messenger behavior

Meta’s WhatsApp Business Platform page states that businesses can use WhatsApp APIs for lead generation, conversational commerce, order confirmations, shipment updates, cart abandonment re-engagement, product browsing, order placement, automated flows, routing, and back-end integrations. [S2]

**Product implication:** Uddokta’s use cases should stay inside business support, commerce operations, order updates, and customer care. Avoid positioning it as a general-purpose AI chatbot on WhatsApp.

## 2.5 Payments and MFS behavior

bKash states that Personal Retail Account allows micro and small businesses in retail and f-commerce to collect payment from bKash customers. It lists payment receive limits of BDT 30,000 per transaction and BDT 500,000 monthly for PRA, and states that customers can pay by QR, payment gateway, or manual PRA number entry. [S3]

bKash also lists fee details on the PRA page, including 0.20% for specified Personal Retail to Merchant payment use cases. [S4]

**Product implication:** Uddokta should include payment-proof tracking, bKash/Nagad reference fields, and payment verification states even if full payment gateway integration is excluded from MVP.

## 2.6 Trust, fraud, and informal-market dispute behavior

A 2026 arXiv study on Bangladesh informal e-market trust states that buyers and sellers often deal with fraud and financial harm by sharing posts, screenshots, and warnings in social media groups, but these reports are scattered, hard to verify, and rarely lead to resolution. The study used a survey of 124 participants, interviews with 36 buyers/sellers/stakeholders, and evaluation with 32 participants. [S5]

**Product implication:** Uddokta’s trust layer is not optional. Digital receipts, order lookup, issue logs, review collection, and seller trust pages directly address a documented informal-commerce behavior.

## 2.7 SME digital adoption barriers

A 2024 arXiv study on 4IR adoption among Bangladesh SMEs found that stakeholders cited lack of technical knowledge, financial constraints, inadequate training, and safety/security concerns as challenges during digital transition. [S6]

**Product implication:** Uddokta must use done-for-you onboarding, mobile-first design, Bangla/Banglish copy, clear support boundaries, and visible data export/backup.

## 2.8 Competitor market signal

BotSailor publicly positions itself around WhatsApp, Facebook, Instagram, Telegram, webchat, e-commerce, retail, AI agents, Google Sheets, SSLCommerz, n8n, webhooks, WhatsApp catalog, WhatsApp flows, and reseller programs. It publicly claims **1.34M+ global entities**, but no public Bangladesh-specific user count was found. [S7]

Manychat positions itself around Instagram, WhatsApp, Messenger, TikTok, automation, ecommerce, creators, agencies, and brands, and states it is loved by 1M+ creators, marketers, and brands. [S10]

WATI positions itself around WhatsApp Business API, no-code chatbots, Instagram automation, click-to-WhatsApp ads, campaigns, team inbox, catalog, multiple WhatsApp numbers, and ecommerce/agency/healthcare/e-learning use cases. [S11]

Respond.io positions itself as a customer conversation management platform that unifies WhatsApp, TikTok ads, Facebook messages, store visits, and CRM-linked profiles into a team inbox so opportunities are not overlooked. [S12]

GoHighLevel offers a broad agency platform with CRM, forms, websites/funnels, chat widget, inbound SMS/social DMs, social planner, missed-call text-back, ad manager, invoicing, payment integrations, calendars, order forms, and upsells/downsells. [S9]

**Product implication:** The category is validated, but Uddokta should not copy feature breadth. It should win through Bangladesh-specific mobile order operations, trust/proof workflows, local payment/courier behavior, and done-for-you setup.

---

# 3. Target Customer Segments

Scoring: 1 = weak, 5 = strong.

| Segment | Pain Intensity | Ability to Pay | FB/WA Dependency | Urgency | Adoption Difficulty | Support Burden | Best Fit Score | Recommendation |
|---|---:|---:|---:|---:|---:|---:|---:|---|
| Cosmetics/skincare sellers | 5 | 4 | 5 | 5 | 3 | 4 | 9/10 | Target first |
| Gadget/accessory sellers | 5 | 4 | 5 | 5 | 3 | 3 | 9/10 | Target first |
| Clothing sellers | 5 | 2–3 | 5 | 4 | 4 | 5 | 6.5/10 | Later |
| Education/training sellers | 3 | 4 | 3 | 3 | 3 | 3 | 6.5/10 | Secondary |
| Home decor sellers | 3 | 3 | 3 | 2 | 3 | 4 | 5/10 | Later |
| Food/cloud kitchen sellers | 5 | 2 | 4 | 5 | 4 | 3 | 4.5/10 | Avoid initially |
| Small service businesses | 2–3 | 3 | 3 | 2 | 3 | 3 | 4.5/10 | Later |

## 3.1 Cosmetics/skincare sellers

**Why strong:** Skincare sales are consultative. Buyers ask about skin type, authenticity, usage, side effects, ingredients, and return/exchange policies. Repeat purchase potential is high.

**Main pains:** repetitive questions, authenticity concerns, delayed follow-up, repeat-purchase reminders, product recommendation history, staff inconsistency.

**Recommendation:** Primary wedge.

## 3.2 Gadget/accessory sellers

**Why strong:** Higher average order values, warranty concerns, counterfeit fear, after-sales questions, and delivery/payment trust issues.

**Main pains:** “original?” questions, warranty tracking, variant/stock confusion, support complaints, fake COD orders.

**Recommendation:** Primary wedge.

## 3.3 Clothing sellers

**Why attractive:** Huge raw volume and constant chat/order flow.

**Why risky:** Low margins, size/color complexity, high COD return risk, heavy support burden, and lower willingness to pay.

**Recommendation:** Later, after product flow is proven.

## 3.4 Food/cloud kitchen sellers

**Why pain is high:** Late replies immediately lose orders.

**Why not first:** Food requires instant fulfillment, local delivery radius, live menu availability, timing, and different operations.

**Recommendation:** Avoid initially.

## 3.5 Education/training sellers

**Why possible:** Lead capture and follow-up matter, but logistics/order proof are less central.

**Recommendation:** Secondary niche.

## 3.6 Home decor sellers

**Why possible:** Higher AOV and trust concerns, but lower daily order volume.

**Recommendation:** Later.

## 3.7 Small service businesses

**Why possible:** General inquiry control, but less repeatable product-template fit.

**Recommendation:** Later.

---

# 4. User Personas

## 4.1 Owner/operator using mobile only

**Profile:** Owner of a Facebook-based cosmetics, skincare, or gadget shop. Operates mostly from Android phone. Uses Facebook Page inbox, Messenger, WhatsApp, bKash/Nagad, courier dashboard, staff calls, and screenshots.

**Goals:** reply faster, stop losing orders, track staff, see orders from mobile, look professional, reduce fake/COD losses, increase repeat customers.

**Fears:** software will be complex, data will disappear, WhatsApp will get banned, staff will not use it, monthly fee will not produce value, customers will dislike bot replies.

**Current workflow:** customer comments/messages, staff replies manually, order details are copied from chat, payment proof arrives as screenshot, courier entry happens separately, follow-up depends on memory.

**Trigger to buy:** missed order, high ad spend with low conversion, fake COD return, staff confusion, repeated customer complaints, desire to look professional.

**Objections:** “My staff already replies,” “Software বুঝি না,” “Monthly fee কেন?”, “WhatsApp ban হবে না তো?”, “Customer robot পছন্দ করবে না.”

**Trust signals needed:** mobile demo, free inbox audit, founder video, real dashboard screenshot, clear cancellation, data export, manual support, Bangla walkthrough.

## 4.2 Staff/admin replying to inbox

**Goals:** reply quickly, find product answers, convert chats to orders, avoid owner pressure.

**Fears:** monitoring feels punitive, dashboard slows work, order creation takes extra time, bot replies wrongly.

**Pain points:** repetitive questions, many open chats, hard to find old customers, no quick product reply library.

**Adoption trigger:** quick replies save typing, one-tap order creation, less confusion.

## 4.3 Customer placing order through chat

**Goals:** know price, availability, authenticity, delivery charge, return policy, payment instructions, and order status.

**Fears:** fake page, wrong product, no delivery, refund problem, payment screenshot misuse.

**Trust signals needed:** professional receipt, order number, clear seller policy, support contact, reviews, delivery confirmation.

## 4.4 Uddokta founder/admin team

**Goals:** onboard clients quickly, keep support burden controlled, monitor failures, prevent data leaks, build case studies, scale with templates.

**Fears:** too many support calls, Meta API policy issues, data leak, low-paying churn, overbuilding, infrastructure failure.

**Needs:** admin console, logs, bot pause, workspace isolation, backup, repeatable onboarding checklist.

---

# 5. Core Product Thesis

## 5.1 Mobile-first

Uddokta must be designed for a low-end Android phone first. Bangladesh’s mobile/social data supports this product direction. [S1]

Requirements:

- PWA installable to home screen
- Bottom navigation
- Card layout
- Minimal tables
- Large tap targets
- Fast perceived loading
- Clear empty states

## 5.2 Bangla/Banglish-friendly

Uddokta must support Bangla, English, and Banglish templates. Stiff English-only SaaS copy will reduce trust and adoption.

## 5.3 Done-for-you onboarding

Bangladesh SME adoption barriers include technical knowledge, financial constraints, training, and security concerns. [S6]

Uddokta must include setup service, product import, FAQ writing, training, and handover.

## 5.4 Order-control-first

The strongest object in the product is not the chat. It is the **Order Card**.

Every conversation should be convertible into:

- Customer
- Product
- Variant
- Price
- Address
- Payment state
- Courier state
- Follow-up state
- Receipt link

## 5.5 Trust/proof-first

Because informal-market trust problems are documented in Bangladesh, Uddokta must treat trust proof as a product layer. [S5]

Required:

- Digital receipt
- Public trust profile
- Review collection
- Issue/refund log
- Customer history
- Payment/delivery proof records

## 5.6 Human-assisted, not fully AI-autonomous

Uddokta should help humans sell better. It should not attempt to fully replace staff.

Required stance:

- Smart templates handle repetitive questions
- Humans close sales and handle trust-sensitive cases
- Staff can pause bot/manual takeover anytime
- Owner sees performance without technical complexity

## 5.7 Premium tech brand

Uddokta must look serious and clean:

- Product screenshots
- Transparent pricing
- Help center
- Founder-led education
- No fake income claims
- No cheap agency-style graphics
- No “AI magic” exaggeration

---

# 6. MVP Scope

## 6.1 Must-have MVP features

| Feature | User Story | Business Value | Acceptance Criteria | Priority | Complexity | Dependency | Existing Tool vs Custom |
|---|---|---|---|---|---|---|---|
| Mobile dashboard | Owner sees orders, messages, follow-ups from phone. | Tangible value. | Loads on mobile; key cards; bottom nav. | P0 | Medium | Next.js/Supabase | Custom |
| Workspace login | Owner/staff access own business only. | SaaS foundation. | Auth works; data scoped. | P0 | Medium | Supabase | Supabase + custom |
| Inbox proxy | Staff sees conversations in one place. | Reduces leakage. | Conversation list, source, status, customer profile. | P0/P1 | High | Chatwoot/Meta | Chatwoot headless + custom |
| Order capture from chat | Staff converts chat to order. | Core aha moment. | Button creates order with customer/items/status. | P0 | Medium | Orders/customers | Custom |
| Order board | Owner tracks orders. | Operational control. | Status board works on mobile. | P0 | Medium | Orders | Custom |
| Customer profile | Staff sees history before replying. | Repeat sales/trust. | Shows phone, address, notes, orders. | P0 | Medium | Customers/orders | Custom |
| Product/FAQ quick replies | Staff replies without typing. | Faster consistent replies. | Product cards and templates usable from chat. | P0 | Medium | Products/templates | Custom |
| Follow-up reminders | Owner recovers pending leads. | Reduces leakage. | Follow-ups due list; manual send/done. | P0 | Medium | n8n/Supabase | n8n + custom |
| Digital receipt link | Customer gets proof. | Trust differentiator. | Public URL with order data and policy. | P0 | Medium | Orders/public routes | Custom |
| Google Sheet export | Owner gets backup. | Reduces data fear. | CSV/Sheet export per workspace. | P1 | Medium | Google/n8n | n8n + custom |
| Weekly report | Owner sees performance. | Retention. | Shows inquiries/orders/returns/followups. | P1 | Medium | Data tables | n8n + custom |
| Admin dashboard | Uddokta creates/manages clients. | Scalable onboarding. | Create workspace, invite owner, view status. | P0 | Medium | Supabase | Custom |
| Basic onboarding form | Prospect submits business data. | Faster setup. | Captures business, links, products, pain. | P0 | Low/Medium | Typebot/custom | Typebot or custom |
| Bot pause/manual takeover | Owner can stop automation. | Trust/risk control. | Workspace/channel/conversation pause. | P0 | Medium | Message router | Custom |
| Logs/error visibility | Admin sees failures. | Reliability. | Message/workflow failure logs. | P0 | Medium | APIs/n8n | Custom |

## 6.2 MVP aha moment

The first real aha moment:

> Seller opens a customer chat, taps **Create Order**, sees a structured order card, sends a professional receipt link, and sees the order appear on the board.

If this flow is not easy on mobile, the product fails.

---

# 7. Out-of-Scope for MVP

| Excluded Feature | Reason |
|---|---|
| Native mobile app | PWA is faster and easier to iterate. |
| Full accounting | Payment proof and basic totals are enough. |
| Full inventory/warehouse | Product list and stock status are enough initially. |
| Advanced AI chatbot | Wrong replies can damage trust; templates are safer. |
| Complex payment gateway routing | Manual proof/reference tracking is enough initially. |
| Full social scheduler | Not core to order-control problem. |
| Multi-location enterprise | Early ICP is small social-commerce seller. |
| Full email marketing | Bangladesh F-commerce is chat/social-first. |
| Full ad manager | Too complex and already served by Meta tools. |
| White-label reseller marketplace | Focus on direct customer success first. |
| Full GoHighLevel clone | Overbuilt and unfocused. |

---

# 8. Customer Journey

1. **Discovery:** founder video, Facebook/Reels/TikTok post, group value post, free tool, referral, case study.
2. **Free inbox audit:** prospect submits page link, category, daily inquiry/order estimate, screenshots if willing, biggest pain.
3. **Demo:** show their category-specific flow: message → quick reply → order card → receipt → follow-up → report.
4. **Payment/setup commitment:** manual bKash/Nagad/PRA screenshot or invoice confirmation.
5. **Onboarding:** collect business, product, FAQ, staff, payment, delivery, policies, language tone.
6. **Initial configuration:** workspace, product list, templates, receipt, trust profile, follow-ups, reports.
7. **First order captured:** seller creates first real order.
8. **First report:** owner sees order/follow-up/return summary.
9. **Retention:** continued order use, customer history, reports, staff accountability.
10. **Upsell:** official WhatsApp API setup, courier integration, advanced reports, review campaigns, DFY optimization.

---

# 9. Client Onboarding Requirements

## 9.1 Onboarding principle

Onboarding must feel like a guided business setup, not software configuration.

The client should think:

> “They are setting up my business control panel.”

## 9.2 Required onboarding data

| Category | Data Needed | Required? |
|---|---|---|
| Business info | Business name, category, logo, address, support phone | Yes |
| Owner info | Owner name, phone, email if available | Yes |
| Social links | Facebook page, Instagram, WhatsApp number | Yes where applicable |
| Channel info | Preferred message channel, production/demo mode | Yes |
| Product/service list | Name, price, variants, stock, image | Yes |
| FAQ | Delivery charge, return policy, payment, location, warranty/original proof | Yes |
| Pricing/delivery | Delivery charge by area if known | Yes |
| Payment methods | COD, bKash, Nagad, PRA, bank | Yes |
| Courier preference | Steadfast, Pathao, REDX, manual | Optional in MVP |
| Staff users | Names, roles, phones | Optional but recommended |
| Tone/language | Bangla, English, Banglish, formal/informal | Yes |
| Approval | Auto-reply script, receipt template, trust profile | Yes |
| Handover | Test order, walkthrough, support channel | Yes |

## 9.3 Onboarding flow

1. Prospect fills audit form.
2. Admin qualifies prospect.
3. Prospect receives demo.
4. Prospect pays setup fee or enters pilot.
5. Admin creates workspace.
6. Admin imports business/product/FAQ data.
7. Admin configures templates and receipt.
8. Admin creates test order.
9. Client approves scripts.
10. Client receives login.
11. Client receives mobile walkthrough.
12. Client creates or handles first order.

---

# 10. Functional Requirements

## 10.1 Auth and workspace

| ID | Requirement | Priority |
|---|---|---|
| PRD-FR-001 | The system shall allow secure user login. | P0 |
| PRD-FR-002 | The system shall support workspace-based tenancy. | P0 |
| PRD-FR-003 | The system shall support owner, manager, staff, viewer, and admin roles. | P0 |
| PRD-FR-004 | Users shall not access records outside their workspace. | P0 |
| PRD-FR-005 | Admin users shall create, suspend, and update workspaces. | P0 |

## 10.2 Inbox

| ID | Requirement | Priority |
|---|---|---|
| PRD-FR-006 | The system shall display customer conversations in a mobile inbox list. | P0 |
| PRD-FR-007 | The inbox shall show customer name/phone, channel, last message, unread count, status, and assigned staff. | P0 |
| PRD-FR-008 | The system shall support manual/demo conversation data before live integrations. | P0 |
| PRD-FR-009 | The system shall support inbound webhook integrations when channels are connected. | P1 |
| PRD-FR-010 | Staff shall assign conversations to users. | P1 |
| PRD-FR-011 | Staff shall pause automation for a conversation. | P0 |
| PRD-FR-012 | Staff shall use quick replies inside conversations. | P0 |

## 10.3 Order board

| ID | Requirement | Priority |
|---|---|---|
| PRD-FR-013 | Staff shall create an order from a conversation or customer page. | P0 |
| PRD-FR-014 | Orders shall include product, quantity, price, customer, phone, address, payment method, and status. | P0 |
| PRD-FR-015 | Orders shall display as mobile cards. | P0 |
| PRD-FR-016 | Supported statuses: New, Confirmed, Payment Pending, Ready to Ship, Shipped, Delivered, Returned, Cancelled. | P0 |
| PRD-FR-017 | Users shall move orders between statuses. | P0 |
| PRD-FR-018 | Users shall add notes and attachments to orders. | P1 |

## 10.4 Customer records

| ID | Requirement | Priority |
|---|---|---|
| PRD-FR-019 | The system shall create and update customer profiles. | P0 |
| PRD-FR-020 | Customer profiles shall show phone, address, notes, tags, orders, and last interaction. | P0 |
| PRD-FR-021 | Staff shall edit customer address and notes. | P0 |
| PRD-FR-022 | The system shall show order history before reply. | P1 |
| PRD-FR-023 | The system shall support risk notes such as fake order, return issue, or payment issue. | P1 |

## 10.5 Product and FAQ

| ID | Requirement | Priority |
|---|---|---|
| PRD-FR-024 | Users shall create products with name, price, image, description, variants, and stock status. | P0 |
| PRD-FR-025 | Products shall support quick reply templates. | P0 |
| PRD-FR-026 | The system shall support FAQ templates for delivery, return, payment, location, warranty, and authenticity. | P0 |
| PRD-FR-027 | Staff shall send product replies from the conversation screen. | P0 |

## 10.6 Automation and follow-up

| ID | Requirement | Priority |
|---|---|---|
| PRD-FR-028 | The system shall allow follow-up reminders for interested customers, pending payment, missing address, delivery confirmation, and review request. | P0 |
| PRD-FR-029 | Sensitive outbound messages shall require manual approval in MVP. | P0 |
| PRD-FR-030 | Every outbound message shall have status tracking. | P0 |
| PRD-FR-031 | The system shall allow workspace-level bot pause. | P0 |
| PRD-FR-032 | The system shall support n8n workflow triggers for follow-up and reports. | P1 |

## 10.7 Reports

| ID | Requirement | Priority |
|---|---|---|
| PRD-FR-033 | The system shall show daily and weekly summaries. | P1 |
| PRD-FR-034 | Reports shall include inquiries, orders, delivered orders, returns, cancellations, follow-ups, and pending items. | P1 |
| PRD-FR-035 | Reports shall be exportable/shareable later as PDF/image/CSV. | P1 |
| PRD-FR-036 | Reports shall avoid unsupported revenue claims. | P0 |

## 10.8 Trust profile and receipt

| ID | Requirement | Priority |
|---|---|---|
| PRD-FR-037 | The system shall generate public order receipt links. | P0 |
| PRD-FR-038 | Receipts shall show order number, business name, customer, product, total, payment status, delivery status, and policy links. | P0 |
| PRD-FR-039 | The system shall support a public trust profile page for each business. | P1 |
| PRD-FR-040 | Trust profile shall show business name, contact, delivery policy, return policy, payment methods, and approved reviews. | P1 |
| PRD-FR-041 | Reviews shall require approval before public display. | P1 |

## 10.9 Admin console

| ID | Requirement | Priority |
|---|---|---|
| PRD-FR-042 | Admin shall create and configure workspaces. | P0 |
| PRD-FR-043 | Admin shall view client health, plan, status, and errors. | P0 |
| PRD-FR-044 | Admin shall pause workspace/channel automation. | P0 |
| PRD-FR-045 | Admin shall configure default templates by niche. | P1 |
| PRD-FR-046 | Admin shall trigger export/backup manually. | P1 |

## 10.10 Settings, notifications, export

| ID | Requirement | Priority |
|---|---|---|
| PRD-FR-047 | Owners shall edit business profile, policies, payment methods, and language preference. | P0 |
| PRD-FR-048 | Owners shall manage staff users. | P1 |
| PRD-FR-049 | Owners shall configure auto-reply templates without workflow logic. | P0 |
| PRD-FR-050 | The system shall notify staff/owner of new orders and follow-up due items. | P1 |
| PRD-FR-051 | Critical integration failures shall notify Uddokta admin. | P0 |
| PRD-FR-052 | The system shall support CSV export for orders/customers. | P1 |
| PRD-FR-053 | The system shall support Google Sheets backup where possible. | P1 |
| PRD-FR-054 | Exports shall respect workspace permissions. | P0 |

---

# 11. Non-Functional Requirements

| ID | Requirement | Priority |
|---|---|---|
| PRD-NFR-001 | The application must be mobile-first and usable on common Android devices. | P0 |
| PRD-NFR-002 | Core screens should feel fast on mobile networks; target perceived load under 2 seconds where possible. | P0 |
| PRD-NFR-003 | All client data must be isolated by workspace. | P0 |
| PRD-NFR-004 | Supabase RLS or equivalent server-side authorization must be mandatory. | P0 |
| PRD-NFR-005 | No service role keys or provider secrets may be exposed to the client. | P0 |
| PRD-NFR-006 | Every external webhook event must be logged with raw payload where lawful and necessary. | P0 |
| PRD-NFR-007 | Every outbound message must have status tracking. | P0 |
| PRD-NFR-008 | Failed external API calls must be visible to admin and eligible for retry. | P0 |
| PRD-NFR-009 | The system must include bot pause/manual takeover controls. | P0 |
| PRD-NFR-010 | The UI must support Bangla, English, and Banglish templates. | P0 |
| PRD-NFR-011 | The UI must avoid technical jargon in customer-facing screens. | P0 |
| PRD-NFR-012 | Public receipt/trust pages must not expose private data. | P0 |
| PRD-NFR-013 | The system must maintain audit logs for critical actions. | P0 |
| PRD-NFR-014 | The product should support poor-network conditions with clear loading, retry, and draft behavior where possible. | P1 |
| PRD-NFR-015 | The system must scale across workspaces without data mixing. | P0 |
| PRD-NFR-016 | Backups/exports must be available to reduce client fear of lock-in. | P1 |
| PRD-NFR-017 | The product must follow Meta/WhatsApp messaging policy boundaries and avoid spam-enabling UX. | P0 |
| PRD-NFR-018 | The product must maintain a professional premium interface standard. | P0 |
| PRD-NFR-019 | The product should use readable fonts, contrast, and large tap targets. | P1 |
| PRD-NFR-020 | Open-source backend tools should be replaceable without changing the client UI. | P1 |

---

# 12. UI/UX Principles

- **Mobile-first:** build from phone screens upward.
- **Card-based:** use order cards, customer cards, product cards, report cards.
- **Large tap targets:** the owner may operate while busy.
- **Minimal typing:** use quick replies, dropdowns, templates, tap-to-call, tap-to-WhatsApp.
- **Familiar patterns:** conversation screen should feel like WhatsApp/Messenger; order board should feel simple, not enterprise CRM.
- **Bangla/Banglish copy:** avoid stiff English-only language.
- **Trust visuals:** order number, receipt link, policy box, review card, backup indicator.
- **Owner dashboard:** show new messages, pending orders, payment pending, follow-ups due, returned/cancelled orders.
- **Staff accountability:** show assigned staff and follow-up due without making the UI feel punitive.
- **No technical jargon:** no API/webhook/CRM/pipeline/tenant wording in customer screens.

---

# 13. Competitive Positioning

| Competitor | What they do well | Weakness for Bangladesh SMEs | Uddokta differentiation |
|---|---|---|---|
| BotSailor | Broad WhatsApp/social automation, ecommerce, AI agents, reseller tools, integrations. [S7] | Broad/technical; exact Bangladesh user count unavailable; not specifically local order-trust OS. | Bangladesh-specific order-control, trust profile, local language, mobile wrapper, DFY setup. |
| WATI | WhatsApp API, team inbox, campaigns, chatbot, catalog, Instagram/Facebook Messenger. [S11] | Foreign SaaS feel; may be too formal or costly for informal sellers. | Local onboarding, Bangla/Banglish templates, order board + receipt/trust layer. |
| Manychat | Instagram/WhatsApp/Messenger/TikTok marketing automation and ecommerce/creator workflows. [S10] | Marketing-first, not Bangladesh COD/order-proof-first. | Operations-first, not campaign-first. |
| Respond.io | Team inbox, customer profiles, lead management, multi-channel conversation handling. [S12] | More global/enterprise; may be too complex. | Simpler mobile Bangladesh seller OS. |
| GoHighLevel | Broad agency CRM/funnels/social/ads/payments/invoicing platform. [S9] | Overbuilt and US/agency-centric. | Narrow, local, mobile-first. |
| Chatwoot | Strong open-source conversation engine. | Direct UI not enough for Bangladesh order/trust workflows. | Use headlessly behind Uddokta wrapper. |
| Courier dashboards | Strong post-sale delivery flow. | Do not solve pre-sale chat/order leakage. | Uddokta connects pre-sale chat to order and trust proof. |
| Manual WhatsApp/Facebook | Familiar and free. | Chaotic, untracked, no reports, no structured proof. | Keeps human selling while adding business control. |

## Ethical differentiation

Uddokta must not copy competitor code, branding, paid templates, proprietary UI, or customer data. It can ethically differentiate by building its own wrapper, using open-source tools within license rules, and creating Bangladesh-specific workflows.

---

# 14. Business Model Requirements

## 14.1 Setup fee logic

A setup fee is required because early customers are non-technical and need guided setup. The setup fee also filters unserious users and funds onboarding labor.

## 14.2 Monthly subscription logic

The monthly fee covers dashboard hosting, storage, reports, workflow maintenance, support, monitoring, backups, and continued product updates.

## 14.3 Done-for-you service layer

DFY setup should be a core product layer, not an afterthought.

Includes:

- Product import
- FAQ writing
- Staff training
- Trust profile setup
- Receipt setup
- Channel connection guidance
- Test order
- Handover

## 14.4 Message/API cost pass-through

Meta/WhatsApp message costs should be shown transparently. Do not hide high messaging costs inside vague plans. This protects trust.

## 14.5 Support boundaries

- Starter: chat/ticket support, help center, basic setup.
- Growth: priority support and setup assistance.
- DFY Premium: full onboarding, training, optimization.

Avoid unlimited phone support for low-tier users.

## 14.6 Package concepts

### Starter

- Mobile order dashboard
- Customer list
- Product list
- Manual order creation
- Basic quick replies
- Receipt link
- Basic reports
- Export

### Growth

- Inbox integration/proxy
- Follow-ups
- Staff assignment
- Customer history
- Google Sheets backup
- Review request
- Advanced reports

### DFY Premium

- Full setup
- Product import
- FAQ writing
- Staff training
- Trust profile setup
- Channel connection support
- Workflow customization

## 14.7 Premium positioning

Uddokta should not compete as the cheapest bot. It should compete as the simplest serious operating system for Bangladeshi sellers.

---

# 15. Success Metrics

## 15.1 Activation metrics

- Workspace setup completed
- Owner first login
- First product added
- First customer added
- First order created
- First receipt sent
- First follow-up scheduled
- First report viewed

## 15.2 First value moment

Primary first value moment:

> Seller creates an order from a conversation/customer and sends a professional receipt link.

## 15.3 Product usage metrics

- Orders created per workspace
- Conversations converted to orders
- Follow-ups scheduled
- Follow-ups completed
- Receipts generated
- Product quick replies used
- Customer profiles updated
- Staff response activity
- Report views

## 15.4 Business metrics

- Setup fee collection rate
- Monthly subscription conversion
- Active paying workspaces
- Average revenue per workspace
- Support tickets per workspace
- Churn risk accounts
- Refund/cancellation requests
- Upsell rate

## 15.5 Pilot validation metrics

- Do sellers understand dashboard without long training?
- Do they create orders inside Uddokta instead of reverting to screenshots?
- Do they value receipt/trust links?
- Do they use product quick replies?
- Do they view reports?
- Are they willing to pay setup/monthly fee?
- Which niche reaches aha moment fastest?

---

# 16. Risks and Mitigation

| Risk | Severity | Description | Mitigation |
|---|---|---|---|
| Meta API policy risk | High | Spam or template misuse can harm messaging reliability. | Official API for production, manual approval, opt-out, logs. |
| Unofficial WhatsApp bridge risk | High | QR/web automation can disconnect or create number risk. | Demo/testing only; disclose risk. |
| Data leak risk | Critical | Cross-tenant data leak would destroy trust. | RLS, server-side checks, audit logs, security tests. |
| Support overload | High | Non-technical SMEs may require too much support. | Setup fee, qualification, help center, support tiers. |
| Wrong niche | High | Low-margin sellers may churn or refuse payment. | Start with cosmetics/skincare and gadgets/accessories. |
| Low willingness to pay | Medium/High | Sellers may resist subscriptions. | Anchor against staff cost and lost-order recovery; show audit proof. |
| Technical fragility | High | Multiple tools create failure points. | Logs, retries, queues, fallback manual mode. |
| Trust failure | High | Cheap UI or broken workflows damage brand. | Premium UI, clear limitations, exports, founder support. |
| Competitor reaction | Medium | Existing tools can copy positioning. | Local execution, templates, service, courier/payment workflows. |
| Overbuilding | High | Full GoHighLevel clone delays validation. | Keep MVP order-control-first. |

---

# 17. Open Questions

1. Which segment pays fastest: cosmetics/skincare or gadgets/accessories?
2. What setup fee is acceptable before brand trust exists?
3. Should dashboard be Bangla-only or Bangla + English?
4. Will sellers use an installable PWA regularly?
5. How much manual onboarding is needed for first activation?
6. Do sellers value receipt/trust links enough to pay?
7. Which report drives retention: orders, missed messages, returns, staff response, or follow-up?
8. How hard is WhatsApp Cloud API onboarding for Bangladesh SMEs?
9. Which courier integration matters first: Steadfast, Pathao, REDX, or export-first?
10. Can live inbox integration wait while order board and receipt prove value?
11. What level of staff monitoring is accepted?
12. What support boundary can be enforced without reducing trust?
13. What marketing claims are safe and ethical?
14. Which product categories must be blocked for compliance reasons?

---

# 18. PRD Verdict

## 18.1 Recommended first wedge

**Premium cosmetics/skincare sellers and gadget/accessory sellers.**

Reason:

- High message volume
- High trust friction
- High consultation need
- Better ability to pay
- Repeat-customer value
- Clear need for order/customer history

## 18.2 Recommended first MVP

> **Mobile inbox-to-order dashboard with customer records, product quick replies, order board, digital receipt link, follow-up reminders, admin setup, and basic reports.**

## 18.3 What to build first

1. Mobile app shell
2. Auth/workspace tenancy
3. Products
4. Customers
5. Orders/order board
6. Receipt/trust link
7. Basic reports
8. Admin setup
9. Onboarding form
10. Demo inbox
11. Follow-up reminders
12. Export/backup
13. Messaging integration stubs
14. Official API integration after core flow is validated

## 18.4 What to test before code

- Manual free inbox audit
- Seller response to dashboard demo
- Willingness to pay setup fee
- Order creation usability on mobile
- Receipt/trust link reaction
- Product quick reply usefulness
- Support burden per seller

## 18.5 What to avoid

- Full GoHighLevel clone
- Advanced AI bot
- Mass broadcast marketing
- Native app
- Full accounting
- Full inventory
- Full payment gateway routing
- Low-margin, high-support clients too early

## 18.6 Feasibility verdict

**Verdict: YES — feasible, but only if Uddokta stays narrow, mobile-first, order-control-first, and trust-proof-first.**

The market behavior is already chat/social-first, the trust problem is documented, and open-source/public tooling can reduce build cost. The business will fail if it is positioned as another chatbot, targets the lowest-paying sellers first, overbuilds into a GoHighLevel clone, or launches without strong onboarding and support boundaries.

---

# References

[S1] DataReportal, **Digital 2026: Bangladesh**. https://datareportal.com/reports/digital-2026-bangladesh

[S2] WhatsApp Business, **Business Platform**. https://business.whatsapp.com/products/business-platform

[S3] bKash, **Personal Retail Account for small businesses**. https://www.bkash.com/en/page/personal-retail-account

[S4] bKash, **PRA fees and limits**. https://www.bkash.com/en/page/personal-retail-account

[S5] ATM Mizanur Rahman and Sharifa Sultana, **Bonik Somiti: A Social-market Tool for Safe, Accountable, and Harmonious Informal E-Market Ecosystem in Bangladesh**, arXiv, 2026. https://arxiv.org/abs/2602.12650

[S6] Toukir Ahammed, Moumita Asad, Kazi Sakib, **Impact of Fourth Industrial Revolution (4IR) on Small and Medium Enterprises (SMEs) and Employment in Bangladesh: Opportunities and Challenges**, arXiv, 2024. https://arxiv.org/abs/2412.21106

[S7] BotSailor official website. https://botsailor.com/

[S8] BotSailor pricing page. https://botsailor.com/pricing

[S9] GoHighLevel official website. https://www.gohighlevel.com/

[S10] Manychat official website. https://manychat.com/

[S11] WATI official website. https://wati.io/

[S12] Respond.io official website. https://respond.io/

[S13] Uploaded Uddokta Business Blueprint, founder-provided strategic source.

---

**Should I continue to Document 2: Technical Requirements Document?**
