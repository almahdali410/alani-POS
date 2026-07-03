# UI/UX Design

Design language: modern SaaS — clean neutral surface (off-white/near-black in dark
mode), one confident accent color, rounded-medium corners, generous spacing, clear
data hierarchy for numbers-heavy screens (reports, POS totals), large touch targets on
POS screens (44px+ minimum) for speed and glove/finger tolerance.

## Sitemap

```
alanipos.com (Public Website)
├── / (Home)
├── /features
├── /pricing
├── /free-trial
├── /about
├── /contact
├── /login
├── /register            → redirects into /free-trial flow
├── /terms
└── /privacy

app.alanipos.com (Tenant App — Web)
├── /onboarding                (first-run: business profile → first store → first products)
├── /dashboard                 (home/overview)
├── /pos                       (POS terminal, full-screen mode)
├── /products
│   ├── /products/:id
│   └── /categories
├── /inventory
│   ├── /inventory/adjustments
│   ├── /inventory/transfers
│   └── /inventory/stock-counts
├── /customers
│   └── /customers/:id
├── /loyalty
├── /employees
│   └── /employees/roles
├── /reports
│   ├── /reports/sales
│   ├── /reports/products
│   └── /reports/employees
├── /stores
├── /settings
│   ├── /settings/business
│   ├── /settings/tax
│   ├── /settings/receipts
│   ├── /settings/payment-methods
│   └── /settings/integrations
└── /billing
    ├── /billing/plan
    └── /billing/invoices

admin.alanipos.com (Master Admin)
├── /admin/dashboard
├── /admin/tenants
│   └── /admin/tenants/:id
├── /admin/packages
├── /admin/payments
├── /admin/cms
│   ├── /admin/cms/home
│   ├── /admin/cms/pricing
│   └── /admin/cms/legal
├── /admin/announcements
├── /admin/support
└── /admin/settings
```

## Main Navigation

**Tenant Web App:** left sidebar (collapsible), grouped as: *Sell* (POS), *Catalog*
(Products, Categories, Inventory), *People* (Customers, Employees), *Insights*
(Reports), *Manage* (Stores, Settings, Billing). Top bar: store switcher, current
shift indicator, search, notifications bell, account menu. Cashiers with a
POS-only role land directly on `/pos` with the sidebar hidden for focus.

**Master Admin:** left sidebar: Dashboard, Tenants, Packages, Payments, CMS,
Announcements, Support, Settings. Top bar: global search (find tenant by name/email),
admin account menu.

**Public Website:** sticky top nav (Home, Features, Pricing, About, Contact,
Login, **Start Free Trial** button styled as primary CTA), footer with sitemap +
legal links + social.

**Mobile (Android/iOS):** bottom tab bar: **Sell** (POS, default tab), **Orders**
(held/history), **Products** (lookup + quick edit), **Customers**, **More** (shift,
settings, sync status, account). POS-only staff see a reduced tab set (Sell, Orders,
More).

## Dashboard Layout (Tenant Overview)

- Top: date-range selector + store filter.
- KPI row (stat tiles): Today's Sales, Transactions, Average Sale, Low Stock Items.
- Sales trend chart (line, selectable period).
- Two-column: "Top Products" list + "Recent Transactions" table.
- Sidebar widget: trial/subscription status card with upgrade CTA when relevant.

## POS Screen Layout

- **Left/main pane (≈65%):** category tabs across the top, product grid below
  (image, name, price tiles); search bar with barcode-scan icon pinned above the grid.
- **Right pane (≈35%, or bottom sheet on mobile):** running cart — line items
  (qty stepper, per-item discount icon), subtotal/tax/discount/total summary, customer
  chip (tap to attach), **Hold** and **Charge** buttons pinned at the bottom.
- **Payment screen** (modal/full-screen on mobile): payment method tabs (Cash, Card,
  Split, Other), numeric keypad for cash tendered with quick-amount buttons, running
  "amount due" always visible, **Complete Sale** confirms and shows the receipt screen
  (print/email/done actions).
- **Header bar:** store name, current shift/cashier, held-orders count badge, sync
  status indicator (mobile: online/offline + pending-sync count).

## Product Management Screen

- Table view: image thumbnail, name, SKU, category, stock (per store or total),
  price, status toggle — with column-based sort/filter and bulk-select actions
  (bulk category change, bulk export, archive).
- Product detail/edit: tabbed layout — *General* (name, category, images), *Pricing*
  (cost/sell price, tax), *Variants* (attribute matrix), *Inventory* (per-store stock,
  thresholds), *Suppliers*.

## Inventory Screen

- Store selector at top; stock table (SKU, name, on-hand, low-stock flag, last
  movement date).
- Tabs: *Stock Levels*, *Transfers* (kanban-like: Draft → In Transit → Completed),
  *Stock Counts* (session list → count-entry grid), *Suppliers*.
- Low-stock items surfaced as a filterable quick-view and a dashboard widget.

## Sales Report Screen

- Filter bar: date range, store(s), employee, category — persists as URL state for
  shareable/bookmarkable report views.
- Summary cards: Gross Sales, Discounts, Refunds, Tax, Net Sales.
- Chart area (toggle: by day/week/month; toggle: by product/category/employee/store).
- Data table beneath the chart mirroring the current breakdown, with **Export
  (Excel/PDF)** action.

## Employee Screen

- Staff directory table (avatar, name, role, assigned store(s), status).
- Employee detail: profile, role/permissions, assigned stores, shift history,
  attendance log, sales performance summary.
- Roles sub-screen: permission matrix editor (checkbox grid: modules × actions) for
  building custom roles.

## Customer & Loyalty Screen

- Customer table: name, phone/email, group, loyalty points, total spent, last visit.
- Customer detail: profile + notes, purchase history timeline, loyalty ledger
  (earn/redeem entries), quick actions (adjust points, add note).
- Loyalty program settings: earn rate, redemption rate, tier/group configuration.

## Subscription & Billing Screen

- Current plan card: package name, limits with usage bars (e.g., "3 of 5 stores
  used"), renewal date, status badge (trial/active/expired/grace).
- Plan comparison table with **Upgrade/Change Plan** action.
- Invoice history table with download-PDF action per row.
- Payment method section (iPaymu-hosted management link).

## Master Admin Dashboard

- KPI row: Total Subscribers, MRR/ARR, Trial Users, Expired Accounts.
- Revenue trend chart + subscription funnel (Trial → Active → Cancelled) visualization.
- Recent signups table + alerts panel (failed payments, open tickets nearing SLA).
- Quick links into Tenants, Packages, Payments, CMS.

## Public Website Layout

- **Home:** full-width hero (headline, subcopy, primary "Start Free Trial" + secondary
  "See Pricing" CTAs, product screenshot/mockup), logo strip (optional social proof),
  feature highlight grid (3-4 cards), testimonials carousel, pricing teaser, final CTA
  band, footer.
- **Pricing:** plan cards (monthly/annual toggle placeholder, annual live at launch),
  feature-comparison table below the fold, FAQ accordion, CTA band.
- All marketing pages share the same header/footer chrome and a consistent 12-column
  responsive grid, breaking to a single column under 768px.

## Responsive & Accessibility Notes
- Breakpoints: mobile (<768px), tablet (768-1199px, primary POS-till target),
  desktop (≥1200px, primary back-office target).
- WCAG 2.1 AA color contrast targets; all interactive controls keyboard-navigable on
  web; POS numeric entry supports hardware keyboard input for speed.
- Dark mode supported across tenant app and Master Admin (not required for the public
  marketing site at launch).
