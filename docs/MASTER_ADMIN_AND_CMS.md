# Master Admin Panel & Public Website CMS

## Master Admin Panel

The Master Admin Panel is a separate authenticated area (`admin.alanipos.com`) for the
platform operator, built on the same Next.js codebase as a distinct route group with
its own auth realm (`admin_users`, not tenant `users`).

### Dashboard Overview
- Total subscribers, active subscriptions, trial users, expired accounts.
- MRR/ARR, revenue trend chart, payment success rate.
- New signups (day/week/month), churn rate, trial-to-paid conversion rate.
- Alerts panel: failed payments needing review, tickets awaiting response, tenants
  approaching plan limits.

### Subscription Package Management
- CRUD for `subscription_packages`: name, price, billing cycle, limits
  (stores/employees/products/transactions), feature/module toggles, public visibility,
  sort order (controls pricing-page display order).
- Preview how a package renders on the public Pricing page before publishing.
- Archive (not hard-delete) packages already in use by active tenants.

### Subscriber / Tenant Management
- Searchable, filterable tenant list (status, package, country, signup date).
- Tenant detail view: profile, current subscription, usage vs. limits (stores,
  employees, products, transactions this period), payment history, support tickets.
- Actions: suspend, reactivate, manually extend subscription, change package, impersonate
  (support-mode read-only login, fully audit-logged) for troubleshooting.

### Payments & Invoices
- Global invoice list across all tenants, filterable by status/provider/date.
- Manual "mark as paid" / "issue refund" actions (writes `payment_transactions` +
  `audit_logs`).
- Raw gateway payload viewer per transaction for dispute investigation.

### Trial Settings
- Configure trial length (default 1 month), default trial package, whether a payment
  method is required to start a trial, and trial-ending reminder schedule.

### Global Application Settings
- Platform name/branding, supported currencies, default language, support contact
  info, legal entity details used on invoices, email/notification templates.

### Web Profile / Landing Page & CMS
- Section-by-section editor mapped to `website_layout_settings` + `cms_content`:
  hero, features, pricing, testimonials, FAQ, footer, terms, privacy, contact.
- Drag-to-reorder sections, toggle visibility, edit copy/images inline with live
  preview against the actual public site rendering.

### Module Management
- Enable/disable feature modules (loyalty, multi-store, advanced reports, API access,
  integrations) **per package**, which cascades to what each tenant on that package
  can access — the same toggle set that lives in `subscription_packages.features`.

### Tenant Usage Analytics
- Cross-tenant aggregate charts: transactions/month distribution, storage/media usage,
  most-used modules, package popularity, geographic distribution.

### Support Tickets
- Queue view (open/pending/resolved), assignment to admin staff, threaded messages
  (`support_tickets` + `support_ticket_messages`), SLA/priority flags.

### Announcements
- Create platform-wide or package-targeted announcements (`announcements`) shown as a
  banner/notification inside tenant dashboards, with a scheduling window.

---

## Public Website / Web Profile

A marketing site (`alanipos.com`) whose structure and copy are entirely
Master-Admin-editable through the CMS above — no code deploy required to change
content.

### Pages
- **Home** — hero, value proposition, feature highlights, social proof, CTA to trial.
- **Features** — categorized feature breakdown (POS, Inventory, CRM, Multi-store,
  Analytics) driven by `cms_content` for the `features` page.
- **Pricing** — renders live `subscription_packages` where `is_public = true`, with
  annual pricing and a feature-comparison table generated from each package's
  `features` array.
- **Free Trial** — the registration form (`POST /tenants/register`), minimal friction.
- **About** — company story/mission (CMS content).
- **Contact** — contact form (creates a `support_tickets` entry with no tenant, tagged
  `pre_sales`) + company contact details.
- **Login / Register** — auth entry points, register deep-links to the Free Trial flow.
- **Terms of Service / Privacy Policy** — long-form legal content, CMS-managed,
  versioned (`updated_by_admin_id`, timestamp shown to users).

### Design Direction
Modern, responsive, professional SaaS aesthetic: generous whitespace, a confident
single accent color, clear typographic hierarchy, product screenshots/mockups over
stock imagery, sticky nav with a persistent "Start Free Trial" CTA, and full
mobile-responsive breakpoints — see [UI_UX_DESIGN.md](./UI_UX_DESIGN.md) for layout
specifics.
