# Alani POS — Product & Engineering Design

Alani POS is a multi-tenant, subscription-based Point of Sale platform for small and
mid-size businesses worldwide — retail stores, cafés, restaurants, service businesses,
and multi-branch companies. It ships as three products sharing one backend: a **Web
App**, an **Android App**, and an **iOS App**, plus a **Master Admin Panel** for the
platform owner and a **Public Website** for marketing and self-service sign-up.

This `docs/` folder is the single source of truth for product and technical design.
It is meant to take the project from zero to a build-ready plan.

## Document Map

| Doc | Contents |
|---|---|
| [README.md](./README.md) | This file — product overview, target users, feature summary, SaaS model, roles & permissions |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Tech stack, system architecture, multi-tenancy, auth, security, deployment, backup, monitoring |
| [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md) | Full relational schema — every table, columns, relationships, indexes, tenant isolation |
| [API_DESIGN.md](./API_DESIGN.md) | REST API surface, conventions, and request/response examples |
| [FLOWS.md](./FLOWS.md) | Subscription & payment flow, POS transaction flow, inventory flow, multi-store flow, CRM/loyalty flow |
| [MASTER_ADMIN_AND_CMS.md](./MASTER_ADMIN_AND_CMS.md) | Master Admin Panel features and Public Website CMS features |
| [PLATFORM_FEATURES.md](./PLATFORM_FEATURES.md) | Web vs. Android vs. iOS feature parity matrix, offline sync design |
| [UI_UX_DESIGN.md](./UI_UX_DESIGN.md) | Sitemap, navigation, and screen-by-screen layout specs |
| [ROADMAP.md](./ROADMAP.md) | MVP scope, 7-phase delivery roadmap, and recommended next steps |

---

## 1. Product Overview

**Alani POS** is a cloud-native, subscription-based POS and business-management
platform. It is *not* a clone of any existing product — it borrows only the general
category standard set by tools like Loyverse (simple cashier flow, offline-capable
mobile POS, back-office inventory) and builds an original data model, UI system, and
architecture around a SaaS-first, multi-tenant core.

**Positioning:** "The POS that grows with your business" — a business starts on a free
trial with a single store and a handful of products, and can scale into a multi-branch
operation with role-based staff, loyalty programs, and advanced analytics without ever
migrating platforms.

**Core design principles:**
- **Multi-tenant from day one** — every entity in the system belongs to a tenant; there
  is no "single business" mode to later refactor away from.
- **Offline-first at the till** — a cashier must be able to ring up a sale with no
  internet connection; the backend is the source of truth once connectivity returns.
- **Subscription-gated, not feature-crippled** — packages gate *limits and premium
  modules*, not core usability, so a trial user gets a real product experience.
- **API-first** — the Web, Android, iOS, and Master Admin clients are all consumers of
  the same versioned REST API; there is no hidden server-rendered-only logic.
- **Global by default** — multi-currency, multi-language-ready (English at launch),
  multi-tax-regime, and timezone-aware from the schema up.

---

## 2. Target Users

| Segment | Description | Primary needs |
|---|---|---|
| **Retail stores** | Single or multi-branch shops (fashion, electronics, groceries) | Barcode-driven checkout, inventory, stock transfer |
| **Cafés & restaurants** | Quick-service and table-service food businesses | Open tickets, kitchen-facing order flow, split bills, tips |
| **Service businesses** | Salons, repair shops, small clinics | Item + service line sales, appointments-adjacent CRM, no complex inventory |
| **Multi-branch companies** | Franchises and chains with 3+ locations | Centralized reporting, per-store roles, stock transfer, store comparison |
| **Solo entrepreneurs / market vendors** | One person, one register, low volume | Fast free-trial onboarding, minimal setup, mobile-only usage |
| **SaaS platform owner (internal)** | Alani POS operator | Subscriber lifecycle, billing, plan design, platform health |

**Geography:** international, English-language UI, multi-currency support, VAT/GST/Sales
Tax-configurable tax engine, and a payment abstraction layer so regional gateways can be
added without re-architecting billing.

---

## 3. Main Features (Summary)

- **SaaS subscription engine** — trials, packages, limits, renewals, invoices, grace
  periods, iPaymu billing (Visa/Mastercard) with a pluggable gateway layer.
- **Master Admin Panel** — full control over subscribers, packages, payments, website
  CMS, module toggles, announcements, and support tickets.
- **Public marketing website** — CMS-driven landing page, pricing, free trial signup.
- **Tenant Admin dashboard** — business profile, stores, staff, roles, catalog,
  reporting, CRM, loyalty, tax, receipts, integrations, billing.
- **POS terminal (Web/Android/iOS)** — cart, barcode scan, discounts, tax, split
  payment, refunds, held orders/open tickets, shifts, cash drawer, offline mode.
- **Inventory management** — variants, SKU/barcode, multi-store stock, transfers,
  suppliers, stock counts, low-stock alerts, import/export.
- **Analytics** — sales by product/category/employee/store, P&L-style summaries, charts,
  exports.
- **Employee management** — roles, permissions, shifts, clock-in/out, performance,
  activity logs.
- **CRM & loyalty** — customer profiles, purchase history, points, tiers, groups,
  birthday offers, feedback.
- **Multi-store** — branch-level catalog/staff/stock with centralized roll-up reporting.
- **Integrations** — printers, cash drawers, barcode scanners, payment gateways,
  accounting exports, webhooks, public API.

Full detail for each area is in the linked documents above.

---

## 4. SaaS Business Model

- **Model:** subscription SaaS, tenant = one business (may own multiple stores).
- **Trial:** 1 month free, no credit card required to start (configurable by Master
  Admin), full feature access up to the trial package's limits.
- **Billing cadence at launch:** annual plans only (monthly can be added later — the
  billing engine is cadence-agnostic from day one, see [FLOWS.md](./FLOWS.md)).
- **Packages:** created/edited by Master Admin; each package defines numeric limits
  (stores, employees, products, transactions/month) and a feature/module allowlist
  (e.g., loyalty, multi-store, advanced reports, API access).
- **Subscription statuses:** `trial → active → expired → suspended → cancelled`, plus a
  configurable **grace period** between `expired` and hard lockout.
- **Payments:** iPaymu (Visa/Mastercard) at launch; abstracted `PaymentGatewayProvider`
  interface so Stripe/PayPal/regional gateways plug in without touching billing logic.
- **Manual overrides:** Master Admin can verify a payment, extend a subscription, or
  force a status change (e.g., goodwill extension, chargeback suspension).
- **Usage enforcement:** limits are checked at the API layer (e.g., "create store"
  fails with `402/403` once the package's store limit is hit), and near-limit usage is
  surfaced in both the tenant dashboard and Master Admin analytics.

Full billing/payment sequence diagrams are in [FLOWS.md](./FLOWS.md#1-subscription--payment-flow).

---

## 5. User Roles & Permissions

Roles are split into **platform-level** (not tied to a tenant) and **tenant-level**
(scoped to one tenant, and in some cases one store).

| Role | Scope | Summary |
|---|---|---|
| **Super Admin / Master Admin** | Platform | Full control of the SaaS platform: tenants, packages, payments, CMS, settings. Not part of any tenant's data. |
| **Tenant Owner / Business Owner** | Tenant | Full control within their tenant: stores, staff, catalog, billing, all reports. Created at registration. |
| **Store Manager** | Store | Full operational control of one or more assigned stores: staff scheduling, inventory, store-level reports. No billing/tenant-settings access. |
| **Accountant** | Tenant | Read access to financial reports, taxes, invoices, payouts; no POS or inventory write access. |
| **Inventory Staff** | Store(s) | Manage products, stock levels, stock transfers, purchase orders; no sales/reporting access beyond stock. |
| **Cashier** | Store | POS-only: sell, take payment, hold/open tickets, apply pre-approved discounts, view own shift/till. No settings, no other cashiers' data. |
| **Employee (generic)** | Store | Base role for staff without a specialized function (e.g., clock-in/out, view own sales); customizable via the permission matrix. |
| **Customer** | Tenant (read-only, self) | Optional self-service identity for loyalty balance/history lookup (future customer-facing portal). |

**Permission model:** role = a named bundle of granular permissions
(`resource:action`, e.g. `products:write`, `reports:financial:read`,
`settings:billing:write`). Tenant Owners can create **custom roles** by combining
permissions; Store Manager/Cashier/Inventory Staff/Accountant ship as default templates.
Permissions are enforced both in the API (authorization middleware) and reflected in
client UI (hide/disable actions a role can't perform). See
[ARCHITECTURE.md](./ARCHITECTURE.md#authentication--authorization) for the RBAC model
and [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md#roles--permissions) for the schema.
