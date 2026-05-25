# Merchant Hub — Platform Design Spec

**Status:** Draft for review
**Date:** 2026-05-12
**Scope:** Platform-level architecture only. Per-service feature specs (Website Builder, AI Concierge, AI Management, Delivery CMS, Contactless CMS, Table Management) will be written as separate documents against the contracts defined here.

---

## 1. Overview

Merchant Hub is a single web application that hosts a catalogue of **service modules** on top of a shared **platform core**. A merchant subscribes to a tier (one) and zero-or-more add-ons; the hub gates UI and APIs based on the resulting entitlements. New services can be added later without re-onboarding or re-login.

### 1.1 Goals
- One coherent surface for merchants regardless of which services they bought.
- Easy to start small (a few services) and grow into the full platform.
- Each service module is independently buildable, releasable, and replaceable.
- Shared data (menu, customers, staff, promotion codes) lives in the platform core — one record, used by every service.

### 1.2 Non-goals (for this spec)
- Per-service feature specs (each service gets its own doc).
- Specific tier contents and pricing — owned by the business team; the platform treats them as configuration.
- Choice of message bus, warehouse vendor, or AI provider — the spec assumes these exist.

---

## 2. Service Catalogue

| Module | Audience | Owns | Reads from |
|---|---|---|---|
| Restaurant Website Builder | Public diners | Site config, pages, theme | Content Engine, Booking Engine, Promotion Code |
| AI Concierge | Public diners (web / Zalo / WhatsApp / Messenger) | Conversations, escalation rules | Content Engine, Booking Engine (writes), CDP (writes), Promotion Code |
| AI Management System | Owners / managers (mobile-first) | KPI views, recommendations, anomalies, benchmarks | All service telemetry via events |
| Merchant CMS — Delivery | Staff | Off-premise orders, drivers, delivery zones, ETAs | Content Engine, CDP, Promotion Code, Staff |
| Merchant CMS — Contactless | Staff | On-premise QR orders, table-runner workflow | Content Engine, CDP, Promotion Code, Staff |
| Table Management | Staff | Tables, floor plan, Booking Engine (owns the data; multiple UIs write to it) | Content Engine, CDP, Promotion Code, Staff |

**Boundary note — Booking Engine:** owned by Table Management (data model, business rules, source of truth). Driven by multiple UIs: Table Management's own surface, AI Concierge, Website Builder's embedded widget, the public booking page. No service ever reads the booking table directly; all access goes through Table Management's contracts.

---

## 2a. Service Summaries

Each service is described in four blocks: **Who it's for**, **Pricing model**, **What it does**, **Key features**. AI Concierge content is canonical (provided by product); the other five are first drafts based on this spec and to be refined by product before sign-off.

### Restaurant Website Builder *(canonical)*
*Drag-and-drop website builder tuned for the restaurant use case.*

- **Who it's for** — Restaurants without an existing website, or with a generic template that doesn't convert. Starter and Operator tiers especially.
- **Pricing model** — Subscription — included with every Hub tier. No PAYG component; the value is constant month-over-month.
- **What it does** — Lets the merchant build and maintain their own restaurant website without a developer. Pre-fills menu, hours, photos, and reviews from existing online presence (Foody / Google) so the merchant starts at 80% rather than zero. Hosts on Savyu infrastructure with SSL, mobile-responsive design, and basic SEO.
- **Key features**
  - Templates designed for the restaurant use case (menu, gallery, location, reservations).
  - Auto-fills from existing presence (Foody, Google Business Profile).
  - Embedded reservation and order CTAs that route to Savyu products.
  - SEO basics: structured data, sitemap, mobile responsive.
  - Content updates flow from the Merchant CMS — update once, propagates here.

### AI Concierge *(canonical)*
- **Who it's for** — Restaurants with high inquiry volume — popular spots, hotel restaurants, places with extensive menus or dietary complexity. Operator and Grower tiers.
- **Pricing model** — Hybrid: subscription for the Concierge product + PAYG for AI inference beyond included monthly quota. Included quota scales with Hub tier.
- **What it does** — Sits on the merchant's website, Zalo OA, WhatsApp, or other channels. Answers questions about menu, dietary options, hours, location, parking, kid-friendliness. Books reservations directly into the Booking Engine. Knows the menu cold (powered by the Content Engine). Hands off to humans for complex requests.
- **Key features**
  - Multi-channel: web widget, Zalo, WhatsApp, Facebook Messenger.
  - Books reservations end-to-end (writes to the Booking Engine).
  - Knows menu, allergens, hours, policies (reads from the Content Engine).
  - Conversation transcripts feed the Unified CDP for end-customer profiles.
  - Escalation rules per merchant (e.g. "private events → always to a human").

### AI Management System *(canonical)*
*Operations dashboard with intelligence layered on top — not just metrics, but recommendations.*

- **Who it's for** — Owner-operators and managers who run the day-to-day. Operator tier and above. The "how is my business doing" daily check-in.
- **Pricing model** — Subscription — included with Operator tier and above. No PAYG; value scales with how often the merchant logs in, not with usage volume.
- **What it does** — The default dashboard view inside the Hub. Shows yesterday's performance, today's outlook, and recommended actions. Uses **Aggregated Data Intelligence** to compare merchant performance against peers in their cuisine type and area. Surfaces "the next best action this week" rather than dumping raw metrics.
- **Key features**
  - KPIs: covers, average ticket, online vs walk-in mix, repeat customer rate.
  - Peer benchmarks: "how do I compare to similar restaurants nearby?"
  - Daily recommendations: surfaced by the rule engine in Part 3.
  - Anomaly detection: flag drops or spikes the merchant should know about.
  - Mobile-first — designed for the owner checking on their phone.

### Merchant CMS — Delivery
- **Who it's for** — Restaurants running off-premise orders (own delivery, pickup, or 3rd-party logistics fallback).
- **Pricing model** — Subscription. Add-on (or included in higher tiers — TBD by business).
- **What it does** — Manages every off-premise order from intake through delivery: live order kanban, driver dispatch, delivery zones, ETAs. Wired into the shared menu, customer, and promotion-code services so a delivery order is just another channel.
- **Key features**
  - Live order kanban: new → preparing → ready → out for delivery → delivered.
  - Driver roster, live status and location, assignment rules.
  - Delivery zones (polygon), per-zone fees, minimum order, ETA rules.
  - Auto-dispatch with manual override and 3rd-party logistics fallback.
  - Refunds, reprints, full order history.

### Merchant CMS — Contactless
- **Who it's for** — Restaurants accepting dine-in orders via QR at the table.
- **Pricing model** — Subscription. Add-on (or included in higher tiers — TBD by business).
- **What it does** — Diners scan a QR at their table and order without flagging a waiter. Orders fire straight to the kitchen tagged with the table; runners deliver; the bill closes against the table session. Reduces wait, raises ticket size on add-ons.
- **Key features**
  - Live orders by table; "table 12 wants the bill" runner queue.
  - QR generation and printing per table; table grouping into sections.
  - Configurable diner actions: call waiter, request bill, modify items.
  - Split bills, table-session timeout, service charge and tipping policy.
  - Kitchen routing rules per station.

### Table Management System *(canonical · owns Booking Engine)*
*Floor plan, reservation management, and table turnover tracking.*

- **Who it's for** — Restaurants with reservations and managed seating — sit-down restaurants, fine dining, popular casual spots. Grower tier and above.
- **Pricing model** — Hybrid: subscription for the product + PAYG fee per reservation processed beyond included monthly quota. Included quota scales with Hub tier.
- **What it does** — Visual floor plan that staff use during service to seat parties, track turnover, and coordinate with the kitchen. Reservations from any source (AI Concierge, website, Booking app, Giaodoan, walk-in) flow into one view. Staff see the night's plan and current state in one screen.
- **Key features**
  - Drag-and-drop floor plan editor; multiple floor plans per merchant (lunch vs dinner, indoor vs patio).
  - Real-time table status: occupied, paying, turning, available.
  - Reservation queue from all sources unified (writes to and reads from the Booking Engine).
  - Turnover analytics: average seating duration by table size, day, time.
  - Mobile-friendly for hosts working off a tablet at the door.

---

## 3. Platform Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  Merchant Hub Shell  (auth · nav · entitlements · dashboard)        │
├──────────────┬──────────────┬─────────────┬─────────────┬───────────┤
│  Website     │  AI          │  AI         │  Merchant   │  Table    │
│  Builder     │  Concierge   │  Management │  CMS        │  Mgmt     │
│              │  (customer)  │  (operator) │  ├─Delivery │           │
│              │              │             │  └─Contactl.│           │
└──────────────┴──────────────┴─────────────┴─────────────┴───────────┘
┌─────────────────────────────────────────────────────────────────────┐
│  Shared Platform Services                                           │
│  ─ Content Engine     (menu, allergens, hours, policies)            │
│  ─ Unified CDP        (customers, conversations, history)           │
│  ─ Booking Engine     (owned by Table Mgmt; cross-service writes)   │
│  ─ Staff & Roles      (RBAC, per-service permissions)               │
│  ─ Promotion Code     (codes, redemptions, scoping by service)      │
│  ─ Merchant & Locs    (account, locations, branding)                │
│  ─ Billing & Entitl.  (tier + add-ons → feature flags)              │
└─────────────────────────────────────────────────────────────────────┘
```

Service modules never reach into each other's storage. Cross-service interaction is via typed contracts (synchronous) or domain events (asynchronous). See §6.

---

## 4. Service Purchase & Entitlement Model

### 4.1 Commercial model
**Tier (one per merchant) + zero-or-more add-ons.** Tiers are **dynamic** — defined and edited at runtime in billing configuration, not in code. The platform does not hardcode tier names, contents, prices, or quotas. Adding a new tier or a new add-on, renaming an existing tier, or shifting a service between bundle and add-on is a config change.

Two pricing shapes are supported per service:
- **Subscription** — flat fee for access, included in a tier or purchased as an add-on. (Website Builder, Delivery CMS, Contactless CMS, and AI Management use this shape — AI Management's value scales with how often the merchant logs in, not with usage volume, so it has no PAYG component.)
- **Hybrid (subscription + PAYG overage)** — subscription unlocks the service and includes a monthly **quota** of a metered unit; consumption beyond the quota bills as PAYG at a defined overage rate. The included quota can vary by tier. Two services use this shape today:
  - **AI Concierge** — metered units: AI conversations and AI inference tokens (scales with the diner population, not the merchant).
  - **Table Management** — metered unit: reservations processed (scales with restaurant volume, so the service grows with the business).

### 4.2 Entitlement record (single source of truth)

```ts
type ServiceKey =
  | "website_builder" | "ai_concierge" | "ai_management"
  | "delivery_cms" | "contactless_cms" | "table_management";

type MeteredUnit =
  | "ai_conversations" | "ai_inference_tokens"
  | "locations" | "staff_seats";

type MeteredQuota = {
  unit: MeteredUnit;
  included: number;        // resets each billing period
  overagePolicy: "block" | "payg";
  overageRatePerUnit?: number; // required when overagePolicy === "payg"
};

type Entitlement = {
  merchantId: string;
  tierId: string;                 // opaque ID; mapping defined dynamically in billing config
  addOns: ServiceKey[];
  quotas: Record<ServiceKey, MeteredQuota[]>; // empty array = service has no metered units
  status: "active" | "past_due" | "locked";
  periodStart: Date;
  periodEndsAt: Date;
};

type UsageCounter = {
  merchantId: string;
  serviceKey: ServiceKey;
  unit: MeteredUnit;
  consumedThisPeriod: number;
  lastUpdatedAt: Date;
};
```

### 4.3 Resolution
**Service active:** `isServiceActive(merchantId, serviceKey)` =
`status === "active"` **AND** (`serviceKey ∈ tier.includedServices` **OR** `serviceKey ∈ addOns`).

**Metered consumption:** before a service performs a metered action (e.g. AI Concierge starts a conversation), it calls `canConsume(merchantId, serviceKey, unit, amount)` which returns:
- `allow` — under quota, or PAYG and account is current.
- `meter` — over quota; PAYG path; usage recorded and billed.
- `block` — over quota; non-PAYG policy; action denied with structured reason (UI shows upgrade prompt).

Every gate in the UI (sidebar item, dashboard widget, deep link) **and** every API endpoint consults `isServiceActive`. Every metered action also consults `canConsume`. Service modules never check tier IDs — they ask for their own key.

### 4.4 Adding a service later
1. Merchant clicks a locked entry in the sidebar or opens **Browse services**.
2. Sees the service's value prop and price.
3. Confirms purchase → billing webhook updates `Entitlement.addOns` → hub refetches entitlements → sidebar/dashboard reflect the new service immediately. No re-login.
4. First-run wizard for the newly-unlocked service runs on first open.

### 4.5 Non-payment lifecycle (the only path that removes a service)

```
active ──payment fails──▶ past_due ──grace 7 days──▶ locked
   ▲                          │                          │
   └────payment succeeds──────┴──────payment succeeds────┘
```

- **active** — full access.
- **past_due** — non-blocking banner across the hub; all services remain usable; merchant prompted to update payment.
- **locked** — service UIs hidden from sidebar/dashboard; deep links route to a "Payment required" page; data is **retained, never deleted** while locked. Public-facing surfaces (merchant website, AI Concierge widget, public booking page) degrade to **read-only**: menu visible, ordering/booking disabled.
- A successful payment at any point returns the account to **active** immediately.

**Defaults (configurable):** grace period = 7 days; locked-state public surfaces = read-only (not full offline).

---

## 5. Hub Navigation & UX

### 5.1 Shell layout
```
┌────────────────────────────────────────────────────────────────────┐
│  Top bar:  [Logo]  [Location switcher ▾]   [🔍]  [🔔]  [Avatar ▾] │
├──────────┬─────────────────────────────────────────────────────────┤
│ Hub      │   Service header   ┌──────────────────────────────────┐ │
│ sidebar  │   ─────────────────│ Tab1 │ Tab2 │ … │ Settings        │ │
│ (always) │                    └──────────────────────────────────┘ │
│          │                                                         │
│          │   Service content                                       │
└──────────┴─────────────────────────────────────────────────────────┘
```

### 5.2 Hub sidebar groups
1. **Dashboard** — always shown.
2. **Services** — one entry per service module; active services normal, inactive dimmed with 🔒 and an "Add" badge.
3. **Shared data** — Menu, Customers, Staff, Promotion Code; always visible.
4. **Browse services** + **Settings** — at the bottom.

### 5.3 Navigation pattern (in-service)
**Persistent hub sidebar + per-service top tabs.** The hub sidebar never disappears so service switching is always one click. Each service's own sub-features live in top tabs inside the service area. The last tab is always **Settings** (per-service configuration). If a service grows past ~8 features, that is a signal to regroup features — not to switch to a different nav pattern.

### 5.4 Dashboard (default landing)
- Personalized greeting + location selector.
- One **widget** per active service (KPIs + quick actions):
  - Website Builder: site status, last published, traffic snapshot.
  - Table Management: today's bookings, cover count, no-shows.
  - Contactless CMS: open tables, orders in queue, average ticket.
  - Delivery CMS: live orders, driver status, ETA breaches.
  - AI Concierge: conversations today, bookings created, escalations.
  - AI Management: top recommendation card + KPI snapshot.
- A **"You don't have…"** card showing 1–2 inactive services with the strongest value prop for this merchant (driven by AI Management signals when available; default copy otherwise).

### 5.5 Widget contract
Every service module ships a `DashboardWidget` component implementing a fixed contract:
- Title, three size variants, defined loading / empty / error states.
- Optional "Open full app" link.
- No knowledge of dashboard layout.

The hub composes widgets; it does not know what they render. This keeps services decoupled from the dashboard and makes adding a new service additive.

### 5.6 Locked / inactive service behaviour
- Sidebar entry visible but dimmed with 🔒.
- Click → in-hub value-prop page with "Start trial" / "Add to plan" CTAs (not a separate marketing site).
- Deep link to a locked service → the same value-prop page (not 404).

### 5.7 Onboarding edge case
A merchant with **zero active services** (just signed up, hasn't picked a tier) lands on a guided onboarding: choose tier → pick add-ons → chained first-run wizards.

### 5.8 Multi-location
- Location switcher in top bar is global.
- Each service module respects the active location where data is location-scoped.
- Some surfaces are merchant-wide (Merchant Account, Billing, AI Management benchmarks) and ignore the switcher.

### 5.9 Responsive behaviour
- **AI Management** is mobile-first by design.
- The rest of the hub is responsive but optimized for tablet/desktop (kitchen, floor, back-office).
- Mobile collapses the hub sidebar into a bottom nav with the four most-used services.

---

## 6. In-Service IA (feature map per module)

Tabs listed left-to-right. Each tab is one screen / feature area. Sub-features inside a tab are filters / drawers / panels — not separate tabs. The last tab is always **Settings**.

### 6.1 Restaurant Website Builder
- **Site** — preview, publish controls, status (draft/live), publish history.
- **Pages** — landing, menu, about, contact, custom; per-page editor with sections (hero, menu embed, gallery, booking widget embed, testimonials).
- **Theme** — colors, typography, logo, imagery, brand presets.
- **Domain** — custom domain connection, DNS, SSL.
- **SEO & Analytics** — metadata, OG images, GA / Pixel connections.
- **Settings** — locale, footer, legal pages, redirects.

### 6.2 AI Concierge
- **Conversations** — live + history; filter by channel; escalation badges; assignable to staff.
- **Channels** — web widget, Zalo, WhatsApp, Messenger; connection state; per-channel customization.
- **Knowledge** — what the AI knows: menu (linked from Content Engine), hours, policies, FAQs, allergen rules.
- **Behavior** — persona, greeting, fallback messages, escalation rules, allowed actions (book / recommend / take order).
- **Insights** — conversation volume, top intents, booking conversion, escalation rate.
- **Settings** — operating hours, languages, human-handoff routing, transcript retention.

### 6.3 AI Management System *(mobile-first)*
- **Today** — KPIs at-a-glance (covers, average ticket, online vs walk-in mix, repeat rate); large numbers, sparklines.
- **Recommendations** — daily AI-generated cards: "do X because Y"; accept / dismiss / snooze.
- **Anomalies** — flagged drops or spikes with context and likely cause.
- **Benchmarks** — peer comparison (similar cuisine, area, size).
- **Reports** — drill-down weekly / monthly views; exports.
- **Settings** — alert thresholds, notification channels (push/email/SMS), peer-group preferences.

### 6.4 Merchant CMS — Delivery
- **Live orders** — kanban (new → preparing → ready → out for delivery → delivered); countdowns, alerts.
- **Orders** — full history; filters; refunds; reprints.
- **Drivers** — roster, live status/location, assignment, performance.
- **Zones** — delivery polygons; per-zone fees; min order; ETA rules.
- **Dispatch rules** — auto-assign, manual override, fallback to 3rd-party logistics.
- **Settings** — order channels, prep-time defaults, packaging fees, holiday hours.

### 6.5 Merchant CMS — Contactless
- **Live orders** — by table; "table 12 wants the bill"; runner queue; fire-to-kitchen.
- **Orders** — history per table/session; split bills.
- **Tables & QR** — table list; QR generation/printing; table grouping (sections).
- **Service flow** — what diners can do (call waiter, request bill, modifiers); kitchen routing rules.
- **Settings** — service charge, tipping policy, table-session timeout, language.

### 6.6 Table Management *(owns Booking Engine)*
- **Calendar** — day / week / month timeline; bookings, walk-ins, blocks; drag to move.
- **Floor plan** — visual editor; tables with status (free / seated / holding / dirty); seat from waitlist.
- **Bookings** — list view; filters; search; party tags; deposits; cancellations; no-show tracking.
- **Waitlist & walk-ins** — current queue; estimated wait; SMS notify when ready.
- **Availability rules** — service windows; slot duration; max covers per slot; blackout dates.
- **Booking sources** — channels writing to the engine: AI Concierge, public booking page, Website Builder widget, manual; per-source toggle + analytics.
- **Settings** — confirmation messages, deposit policy, no-show fees, integrations (e.g. Google Reserve).

---

## 7. Shared Data (platform core)

Entries always visible in the hub sidebar, owned by platform services and accessed by all modules.

### 7.1 Menu *(Content Engine)*
- **Items** — name, description, photos, allergens, tags, dietary flags.
- **Categories** — grouping for ordering surfaces.
- **Modifiers & options** — sizes, add-ons, modifier groups.
- **Pricing** — per-channel (dine-in / delivery / contactless can differ); per-location.
- **Availability** — schedules (breakfast menu, weekend specials); out-of-stock toggle.
- **Settings** — currency, tax rules, allergen taxonomy.

### 7.2 Customers *(Unified CDP)*
- **Profiles** — search/list; unified view per customer (orders, bookings, conversations, notes).
- **Segments** — saved filters (VIPs, lapsed diners, allergy-flagged).
- **Marketing consents** — channel-level opt-in / opt-out.
- **Settings** — PII retention, merge rules.

### 7.3 Staff
- **Members** — invite, deactivate, reset password.
- **Roles** — per-service permission grants (e.g. "Cashier" gets Contactless live orders only).
- **Activity** — audit log of who did what.
- **Settings** — SSO config (where entitlement allows), session policy.

### 7.4 Promotion Code
- **Codes** — list; create code with rules.
- **Scopes** — which services a code applies to (Delivery only, Contactless only, all; specific items; specific customer segment).
- **Redemptions** — history; who used what.
- **Settings** — code generation defaults, stacking rules.

---

## 8. Cross-Cutting Concerns

### 8.1 Identity & Auth
- **Merchant tenancy:** every record is scoped to a `merchantId`. Staff accounts belong to a merchant; one staff record does not span merchants. The platform itself is multi-tenant; users are not.
- **Auth model:** email/password + magic link by default. SSO (Google / Microsoft / SAML) gated by entitlement.
- **Session:** JWT short-lived + refresh; one session per device; "sign out everywhere" available.
- **Authorization:** RBAC. Roles are platform-defined templates (Owner, Manager, Cashier, Kitchen, Host); each role is a set of `(serviceKey, permission)` grants. Owner can clone and customize templates. Permission checks happen at the **API** level — UI gating is convenience, not security.

### 8.2 Data Ownership & Contracts
Each service owns its data and exposes typed read/write contracts. Cross-service access goes through contracts, never direct DB reads.

| Owner | Data | Read by | Write by |
|---|---|---|---|
| Content Engine | Menu, allergens, hours, policies | All services | Menu UI, merchant admin |
| Unified CDP | Customers, conversations, order/booking history | All services | Delivery, Contactless, Table Mgmt, AI Concierge |
| Booking Engine *(Table Mgmt)* | Bookings, slots, availability | Website Builder, AI Concierge, public booking page | Table Mgmt UI, AI Concierge, public widgets |
| Delivery CMS | Off-premise orders, drivers, zones | AI Management (telemetry, read-only) | Delivery UI |
| Contactless CMS | On-premise orders, table sessions | AI Management (telemetry, read-only) | Contactless UI |
| Promotion Code | Codes, redemptions | Delivery, Contactless, Booking | Promotion UI; services record redemptions |
| Staff & Roles | Staff, permissions | All services | Staff UI |
| Merchant & Locations | Account, locations, branding | All services | Settings |
| Entitlements | Tier, add-ons, status, limits | All services (read); Hub (gate) | Billing webhook only |

### 8.3 Events (loose coupling)
Services publish domain events to a shared bus — e.g. `order.completed`, `booking.created`, `customer.identified`, `conversation.escalated`. AI Management subscribes broadly for KPIs and anomaly detection; the CDP subscribes for profile unification. No synchronous fan-out across services.

### 8.4 Observability & Audit
- Every write produces an **audit log** entry: actor, action, target, before/after, timestamp, merchant + location scope. Surfaced in Staff → Activity.
- Application telemetry: traces per request tagged with `merchantId` and `serviceKey` so per-service performance and errors can be sliced.
- AI surfaces (Concierge, Management) log prompts, responses, model, latency, and cost — for debugging and per-merchant cost attribution.

### 8.5 Extensibility — adding a new service later
A future service module (e.g. "Loyalty") requires only:
1. A `ServiceKey` entry.
2. A `DashboardWidget` implementation.
3. A sidebar entry + tabbed feature surface.
4. Optionally: read/write contracts against existing platform services; events emitted / subscribed.

No changes to entitlements wiring, billing, navigation shell, or shared data — all configuration-driven.

### 8.6 Internationalization
- Locale at merchant level (default) with per-user override.
- Customer-facing surfaces (website, Concierge, QR ordering, booking page) support multi-language; language list configured per merchant.
- Currency at location level.

### 8.7 Public-facing surfaces
Three distinct customer-touching deployments fed by the platform:
- **Merchant website** (Website Builder output) — served at the merchant's domain.
- **AI Concierge** widget + channel adapters — embeddable script plus Zalo / WhatsApp / Messenger webhooks.
- **Public booking page** — hosted booking flow at a hub-owned subdomain, or embedded.

All three respect the locked-state degradation rule in §4.5.

---

## 9. Tech Foundation (recommended defaults)

These are starting defaults; adjust if team or platform constraints differ.

- **Frontend:** React + TypeScript. Single hub application; service modules live as feature-foldered sections inside one repo (modular monolith) initially, with the contract boundaries from §8.2 so they can be split out later.
- **Routing:** App-shell route owns sidebar/topbar; nested routes per service own their tabs.
- **State:** server state via React Query (per-service query keys). Entitlements cached in a hub-level provider with optimistic refresh on the billing webhook.
- **Backend:** service-per-domain API surface — one logical service per module plus one for each shared platform service. Each speaks the contracts in §8.2.
- **Events:** message bus (vendor TBD); spec assumes one exists.
- **Storage:** relational primary store per service domain; object storage for media (menu photos, website assets); separate analytics warehouse fed by events, for AI Management.
- **Auth:** centralized identity service issuing JWTs; entitlements stored alongside and embedded as claims on token refresh.
- **AI infra:** AI Gateway pattern in front of model providers; per-merchant prompt / cost tracking; rate limits per `Entitlement.limits.aiConversationsPerMonth`.
- **Public deployments:** Website Builder publishes to an edge-served static + dynamic hybrid; AI Concierge widget served from CDN with a thin auth handshake.

---

## 10. Open Items (deferred to follow-up specs)

These are intentionally **not** decided in this spec:

- Specific tier names, contents, prices — owned by business.
- Per-service feature specs (one document per service, against the contracts here).
- Message-bus vendor, AI-provider mix, warehouse vendor.
- Detailed RBAC permission catalog per service (defined in each service's spec).
- Migration / import flows (e.g. menu import from third-party POS) — per-service.
- Notification channels and templates (transactional + marketing).
- Mobile native app strategy beyond responsive web for AI Management.

---

## 11. Glossary

- **Service module** — one of the six top-level products in the catalogue (§2).
- **Platform core / shared platform services** — Content Engine, Unified CDP, Booking Engine, Staff & Roles, Promotion Code, Merchant & Locations, Billing & Entitlements.
- **Entitlement** — the record describing what a merchant is allowed to use (§4.2).
- **Booking Engine** — the data + rules layer for reservations; owned by Table Management, written to by multiple UIs.
- **CDP** — Customer Data Platform; the unified customer record.
- **Content Engine** — the menu / hours / policies data layer.
