# Core Business Flows

## 1. Subscription & Payment Flow

### 1.1 Trial signup
1. Visitor fills out **Free Trial** form on the public website (business name, email,
   password, country).
2. `POST /tenants/register` creates `tenants` (status `trial`), `users` (Owner role),
   and `tenant_subscriptions` (`status = trial`, `trial_ends_at = now() + 1 month`,
   `package_id` = the Master-Admin-configured default trial package).
3. Owner is logged in and redirected to onboarding (business profile → first store →
   first products, or "skip and explore").

### 1.2 Trial → paid conversion
1. From day 1, the tenant dashboard shows a persistent trial countdown and an
   **Upgrade** CTA once inside the last 7 days (configurable) of the trial.
2. Owner goes to **Billing**, selects a package (annual pricing), clicks **Subscribe**.
3. `POST /billing/subscription/checkout` creates a `pending` `invoice` +
   `payment_transaction`, calls iPaymu to create a hosted payment session, returns a
   `redirectUrl`.
4. Owner completes card payment (Visa/Mastercard) on iPaymu's hosted page.
5. iPaymu redirects the browser back to Alani POS (`return_url`) **and** independently
   POSTs a signed webhook to `/payments/ipaymu/callback`.
6. The webhook handler verifies the HMAC signature, looks up the `invoice` by
   `reference_id`, and:
   - on `success`: `payment_transaction.status = success`, `invoice.status = paid`,
     `tenant_subscriptions.status = active`,
     `current_period_end = now() + package.billing_cycle`.
   - on `failed`/`cancelled`: `payment_transaction.status = failed`; subscription
     stays as-is; tenant sees a "payment failed, try again" banner.
7. The client-side return page polls `GET /billing/subscription/checkout/:invoiceId/status`
   until the webhook has resolved the invoice (webhook may arrive before or after the
   redirect), then shows success/failure.

### 1.3 Renewal
1. A daily BullMQ job scans `tenant_subscriptions` where `current_period_end` is
   within N days (e.g., 14/7/1 days) and sends renewal reminder emails.
2. On the renewal date, if `auto_renew = true` and a saved payment method exists, the
   system attempts an automatic charge via iPaymu; otherwise the tenant is prompted to
   manually renew.
3. If `current_period_end` passes with no successful payment:
   `status = expired`, and a `grace_period_ends_at` (e.g., +7 days, Master-Admin
   configurable) is set. During grace, the tenant retains read/limited access with an
   in-app banner; after grace expires, POS/write actions are blocked and only the
   Billing page remains accessible.
4. `status = suspended` is reserved for Master-Admin-initiated suspension (e.g.,
   abuse, chargeback) and is independent of the billing-cycle state machine.

### 1.4 Manual override (Master Admin)
`POST /admin/tenants/:id/subscription/extend` or `/admin/invoices/:id/mark-paid` lets
platform staff extend a period, force-activate, or reconcile a payment that succeeded
out-of-band. Every such action writes an `audit_logs` row with before/after state.

### 1.5 State machine

```
        register            checkout success           period ends, no payment
 (none) ─────────► trial ─────────────────────► active ─────────────────────► expired
                     │                              ▲   \                         │
                     │ trial ends, no payment        \   \ renewal payment        │ grace period
                     ▼                                \   \  succeeds             ▼ expires
                  expired ◄───────────────────────────┘    \                  (locked, billing-
                     │            checkout success           \                 only access)
                     │                                         ▼
                     └──────────────► cancelled ◄────── active/trial (owner cancels)
                                          ▲
                     Master Admin ───────►│ suspended (can be applied from active/trial/expired)
                     suspend/reactivate ──┘
```

---

## 2. POS Transaction Flow

1. Cashier opens app/web POS, must have an **open shift** (`POST /pos/shifts/open`
   with opening cash float) before selling.
2. Cashier builds a cart: scans barcode / searches / taps category tiles → adds
   product/variant → adjusts quantity → applies item-level discount if permitted.
3. Optionally attaches a **customer** (search or create) for loyalty/history.
4. Applies transaction-level discount/note if needed; tax is computed live from the
   product/category tax rate (inclusive or exclusive per tenant setting).
5. Cashier proceeds to payment: chooses cash, card, digital, or **splits** across
   multiple methods until the balance reaches zero.
6. On confirm, the client persists the sale **locally first** (outbox), immediately
   shows the receipt/confirmation, then syncs to the server in the background
   (`POST /pos/sync/push` on mobile; direct `POST /pos/sales` when online on web).
7. Server validates stock, prices (re-derives against current catalog for anti-tamper,
   but preserves the client's snapshot for display), writes `sales`, `sale_items`,
   `payments`, and one `stock_movements` row per line item; updates `inventory_levels`.
8. Receipt is offered as print (connected printer), email, or on-screen only.
9. **Held orders / open tickets:** instead of step 5, cashier taps **Hold** — sale is
   saved with `status = held` (retail "park sale") or `status = open_ticket`
   (restaurant/café "table stays open, add more items later"); it reappears in the
   Held Orders list for any cashier at that store to resume.
10. **Refund/return:** from Sales History, select a completed sale → choose items/qty
    to refund → `POST /pos/sales/:id/refund` → compensating `stock_movements` +
    `refunds`/`refund_items` created; refund payment method mirrors or is chosen
    independently (cash back vs. store credit).
11. **Shift close:** cashier counts the drawer, enters closing cash;
    `POST /pos/shifts/:id/close` compares `expected_cash_amount` (opening float + cash
    sales − cash paid-out) against the entered amount and records `cash_difference`
    for manager review.

---

## 3. Inventory Flow

1. **Receiving stock:** Inventory Staff records a purchase receipt against a supplier
   (quantity + cost price) → `stock_movements(type=purchase_receipt)` →
   `inventory_levels.quantity` increases at that store.
2. **Selling stock:** every `sale_item` on a completed sale writes a
   `stock_movements(type=sale, quantity_delta=-qty)`.
3. **Low stock alert:** a scheduled job compares `inventory_levels.quantity` against
   `low_stock_threshold` per SKU per store; breaches trigger an in-app notification and
   optional email to Inventory Staff/Owner, and surface on the Inventory dashboard.
4. **Stock transfer between stores:** Store A creates a `stock_transfers` draft with
   line items → marks `in_transit` (writes `transfer_out` movement at Store A,
   decrementing its stock) → Store B receives and marks `completed` (writes
   `transfer_in` movement at Store B, incrementing its stock). Discrepancies during
   receiving (e.g., damaged goods) are logged as a follow-up adjustment.
5. **Stock count (opname):** a `stock_counts` session is opened for a store, staff
   enter physical counts per SKU (`stock_count_items.counted_quantity`); on submission,
   variances write `adjustment` movements that reconcile `inventory_levels` to the
   counted truth, and a variance report is generated for the manager.
6. **Import/export:** bulk product + starting-stock import via CSV/XLSX
   (`POST /products/import`), validated row-by-row with an error report; export mirrors
   the same schema for round-tripping and backups.

---

## 4. Multi-Store Flow

1. Tenant Owner creates additional `stores` (gated by the package's `max_stores`
   limit — attempting to exceed it returns `403` with an upgrade prompt).
2. Products are tenant-wide by default (shared catalog); `inventory_levels` and
   pricing overrides (if enabled) are per-store, so the same SKU can have different
   stock — and optionally different price — at each branch.
3. Employees are assigned to one or more stores via `employee_store_assignments`;
   Store Managers/Cashiers only see and operate within their assigned store(s).
4. **Centralized reporting:** the Owner's dashboard aggregates across all stores by
   default, with a store filter/breakdown available on every report
   (`/reports/sales-by-store` powers the store-comparison view).
5. **Stock transfer** (above) is the mechanism for moving inventory between branches;
   all transfers are tenant-scoped (both stores must belong to the same tenant).
6. Store-level settings (receipt footer, default tax rate, operating hours) can
   override tenant defaults where relevant, falling back to tenant-level config
   otherwise.

---

## 5. CRM & Loyalty Flow

1. **Customer capture:** at checkout or via the Customers screen, a customer is
   created/looked up by phone/email/name and optionally assigned to a
   `customer_group`.
2. **Purchase history & profile:** every sale linked to a customer contributes to
   `customers.total_spent` (denormalized, updated transactionally) and is queryable
   via `GET /customers/:id`.
3. **Earning points:** if the tenant's `loyalty_programs` is active, completing a sale
   with a linked customer writes a `loyalty_transactions(type=earn)` row
   (`points = amount_spent × points_per_currency_unit`) and updates
   `customers.loyalty_points_balance`.
4. **Redeeming points:** at checkout, cashier can apply a points redemption as a
   payment/discount component; writes `loyalty_transactions(type=redeem)` with a
   negative points delta, validated against the customer's current balance.
5. **Membership tiers/groups:** `customer_groups` can carry a default discount
   percentage and are used to segment customers for targeted offers and reporting.
6. **Birthday/special offers:** a scheduled job matches `customers.birthday` against
   the current date window and triggers a configured email/notification (via the
   integrations layer — WhatsApp/email) offering a discount code.
7. **Feedback:** post-sale, an optional feedback prompt (email link or in-app for a
   future customer portal) captures a rating/comment tied to the `sale_id`/`customer_id`
   for the Owner's review.
8. **Marketing-ready:** customer segments (by group, spend, recency) are exportable
   and structured so a future WhatsApp/email marketing integration can target them
   directly through the `tenant_integrations` layer without schema changes.
