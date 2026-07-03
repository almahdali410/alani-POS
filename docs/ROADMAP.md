# MVP Scope & Development Roadmap

## MVP Scope (What Ships in Phase 1)

The MVP proves the core loop: a business can **register → get a free trial → set up a
store and catalog → sell on the Web POS → see basic reports**, on top of a real
multi-tenant, tenant-isolated data model — not a prototype that gets rebuilt later.

**In scope:** tenant registration + trial, single store (multi-store schema exists but
UI limits to 1 store), Web POS (cart, tax, discounts, cash + card-manual payment,
receipts), basic product/category/inventory CRUD, basic employee + role (Owner,
Cashier), basic sales reports, Master Admin skeleton (tenant list, manual
suspend/reactivate), public website skeleton (Home, Pricing static, Free Trial,
Login).

**Explicitly out of scope for MVP:** iPaymu billing (trial only, no paid conversion
yet — see Phase 2), mobile apps (Phase 3), stock transfer/stock count, loyalty/CRM
depth, advanced analytics/export, integrations, full CMS editing (content is
seeded/static, not yet admin-editable).

---

## Phase 1 — MVP
**Features:** Multi-tenant core, tenant registration + 1-month trial (no payment gate
yet), single-store setup, Web POS (sell, cart, tax, discount, cash/card-manual
payment, held orders, receipts), product/category/inventory CRUD (single store),
basic RBAC (Owner, Cashier roles), basic sales reports (daily/summary), Master Admin
skeleton (tenant list + manual status control), static public website.
**Complexity:** High (this phase builds the entire foundation: multi-tenant DB + RLS,
auth/RBAC, API skeleton, Web app shell).
**Dependencies:** None — this is the foundation everything else builds on.
**Priority:** Critical / first.

## Phase 2 — Subscription & Billing
**Features:** Subscription packages (Master Admin CRUD), package limits enforcement,
iPaymu integration (checkout + webhook callback), invoices, payment_transactions,
trial-expiry and renewal-reminder jobs, grace period logic, manual admin overrides,
tenant Billing page, Master Admin payments/invoices views.
**Complexity:** High (payment integration, webhook security, subscription state
machine, background jobs).
**Dependencies:** Phase 1 (tenant/subscription tables, Master Admin shell).
**Priority:** Critical — this is what makes the product an actual SaaS business.

## Phase 3 — Mobile Apps (Android & iOS)
**Features:** Flutter POS app (sell, cart, discounts, split payment, held orders,
refunds), offline-first local storage + sync engine (push/pull), shift & cash drawer
management, barcode scanning (camera + Bluetooth), receipt printing (Bluetooth/network),
push notifications.
**Complexity:** High (offline sync engine and hardware integrations are the hardest
parts of the whole project).
**Dependencies:** Phase 1 API (POS/sync endpoints must be stable); benefits from
Phase 2 being done so mobile can show subscription status, but not a hard blocker.
**Priority:** High — required to fulfill the three-platform commitment; can run
partially in parallel with Phase 2 once the API is stable.

## Phase 4 — Advanced Inventory
**Features:** Product variants at scale, stock transfer between stores, stock count
(opname), supplier management, low-stock alerts, import/export (CSV/XLSX), inventory
history/audit views, multi-store inventory UI.
**Complexity:** Medium.
**Dependencies:** Phase 1 (catalog/inventory schema already in place, this phase is
UI + workflow depth) and benefits from multi-store existing (Phase 6 can be reordered
earlier if multi-store is a launch priority — see note below).
**Priority:** Medium-High — needed before serious retail customers can rely on the
product.

## Phase 5 — CRM and Loyalty
**Features:** Customer directory, purchase history, notes, loyalty points program,
redemption at checkout, customer groups/tiers, birthday offers, feedback capture,
marketing-integration-ready customer segments.
**Complexity:** Medium.
**Dependencies:** Phase 1 (sales/customer schema) and Phase 3 (checkout-time
redemption UX on mobile).
**Priority:** Medium — strong differentiator for retail/café segment, not required
for initial revenue.

## Phase 6 — Multi-store and Integrations
**Features:** Full multi-store UI (store creation gated by package limits,
employee-store assignment, centralized/store-comparison reporting), integrations
framework (`tenant_integrations`, webhooks, public API + API keys), initial
integrations (email, WhatsApp notification, one accounting export, printer/cash
drawer/barcode scanner config UI).
**Complexity:** High (integrations framework + public API surface + auth model for
API keys).
**Dependencies:** Phase 2 (package limits must exist to gate store count), Phase 4
(stock transfer already depends on multi-store — **recommendation: pull the
multi-store schema/UI forward into Phase 4 if any early customer needs >1 store**,
since the schema already supports it from Phase 1).
**Priority:** Medium-High — critical for the "multi-branch companies" target segment.

## Phase 7 — Analytics and Scaling
**Features:** Advanced analytics (customer purchase trends, low-performing product
detection, cross-store benchmarking), visual dashboard upgrades, Excel/PDF export
everywhere, OLAP layer (materialized views → ClickHouse if volume warrants),
performance hardening (read replicas, caching), full Master Admin analytics
(tenant usage, cohort/churn analysis), public API v1 hardening + docs/partner program.
**Complexity:** Medium-High (mostly performance/scale engineering rather than new
product surface).
**Dependencies:** All prior phases generate the data this phase analyzes.
**Priority:** Ongoing/ Medium — prioritize pieces of this earlier if a specific
customer segment demands it (e.g., multi-store benchmarking can move up if enterprise
customers are being pursued early).

---

## Recommended Next Steps

1. **Validate the data model** against 2-3 real target businesses (a café, a retail
   shop, a multi-branch chain) by walking their actual daily workflow through the
   schema in [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md) before writing code.
2. **Stand up the foundation repo structure**: NestJS API skeleton with the
   multi-tenant/RLS pattern proven on 2-3 tables first (`tenants`, `users`, `products`)
   before building out the rest — this de-risks the riskiest architectural bet early.
3. **Build the Master Admin tenant/package CRUD** alongside the tenant registration
   flow so Phase 1 and the start of Phase 2 can be developed together by a small team.
4. **Get iPaymu sandbox credentials early** — payment webhook integration always
   surfaces surprises (signature format, callback timing/ordering); doing this in
   parallel with Phase 1 avoids a late-phase-2 bottleneck.
5. **Design the offline sync protocol (Phase 3) on paper and review it before writing
   Flutter code** — retrofitting offline-first onto an online-only mobile app is far
   more expensive than designing for it from the first mobile commit.
6. **Pick a design system early** (component library + tokens) shared conceptually
   between Next.js (Web/Admin) and Flutter (Mobile) so the three clients feel like one
   product, per [UI_UX_DESIGN.md](./UI_UX_DESIGN.md).
7. **Decide the first two target verticals** (e.g., retail + café) to focus MVP UX
   copy, receipt templates, and default tax/discount presets on, rather than building
   generically for all business types at once.
8. **Set up CI/CD and the staging environment before Phase 1 is "done"**, not after —
   deploy early and often so the multi-tenant/billing risk areas get tested under
   real infrastructure quickly.
