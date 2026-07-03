# Database Schema

Engine: **PostgreSQL 15+**. All primary keys are `UUID` (generated via `gen_random_uuid()`)
unless noted. All tables include `created_at`, `updated_at` (both `timestamptz`), and
tenant-scoped tables include `tenant_id` + a `deleted_at` (soft delete) unless noted.
Multi-tenant tables are protected by Row-Level Security policies keyed on `tenant_id`
(see [ARCHITECTURE.md](./ARCHITECTURE.md#multi-tenant-strategy)).

Legend for "Multi-tenant": **Platform** = no `tenant_id` (global/shared);
**Tenant** = scoped by `tenant_id`; **Store** = scoped by `tenant_id` + `store_id`.

---

## Platform / Identity

### `admin_users`
Master Admin operators (platform staff), fully separate from tenant users.
- **Columns:** `id`, `email` (unique), `password_hash`, `full_name`, `role`
  (`super_admin`, `support`, `finance`), `is_active`, `two_factor_enabled`, `last_login_at`.
- **Relationships:** referenced by `audit_logs.actor_admin_id`, `support_tickets.assigned_admin_id`.
- **Indexes:** unique on `email`.
- **Multi-tenant:** Platform.

### `tenants`
One row per subscribing business — the tenant root.
- **Columns:** `id`, `name`, `slug` (unique, used in subdomain/URLs), `business_type`
  (`retail`, `cafe_restaurant`, `service`, `other`), `country`, `default_currency`,
  `default_timezone`, `default_language` (`en` at launch), `status`
  (`trial`, `active`, `expired`, `suspended`, `cancelled`), `owner_user_id`, `logo_url`.
- **Relationships:** parent of `stores`, `users`, `products`, `customers`,
  `tenant_subscriptions`, everything tenant-scoped.
- **Indexes:** unique on `slug`; index on `status`.
- **Multi-tenant:** root of the tenant hierarchy (this table itself is the tenant).

### `users`
Staff/user accounts belonging to a tenant (owners, managers, cashiers, etc.).
- **Columns:** `id`, `tenant_id`, `email`, `password_hash`, `full_name`, `phone`,
  `role_id`, `status` (`active`, `invited`, `suspended`), `two_factor_enabled`,
  `last_login_at`, `avatar_url`.
- **Relationships:** `role_id → roles.id`; referenced by `sales.cashier_id`,
  `shifts.user_id`, `employee_store_assignments`.
- **Indexes:** unique on `(tenant_id, email)`; index on `role_id`.
- **Multi-tenant:** Tenant.

### `roles`
Named permission bundles; system default roles are tenant-cloned templates, but
tenants can also define custom roles.
- **Columns:** `id`, `tenant_id` (nullable = system template), `name`
  (`Owner`, `Store Manager`, `Cashier`, `Inventory Staff`, `Accountant`, `Employee`),
  `is_system_default`, `description`.
- **Relationships:** many-to-many with `permissions` via `role_permissions`.
- **Indexes:** index on `tenant_id`.
- **Multi-tenant:** Tenant (nullable for global templates).

### `permissions`
Static catalog of granular permission strings (seeded, not user-editable).
- **Columns:** `id`, `key` (e.g. `products:write`, `reports:financial:read`, unique),
  `module` (`pos`, `inventory`, `reports`, `settings`, `billing`, `admin`), `description`.
- **Multi-tenant:** Platform (shared catalog).

### `role_permissions`
Join table.
- **Columns:** `role_id`, `permission_id`.
- **Indexes:** composite PK `(role_id, permission_id)`.

### `employee_store_assignments`
Which store(s) a user can operate in (for Store Manager/Cashier/Inventory Staff scoping).
- **Columns:** `id`, `tenant_id`, `user_id`, `store_id`, `is_primary`.
- **Indexes:** unique `(user_id, store_id)`.
- **Multi-tenant:** Store.

---

## Stores & Business Structure

### `stores`
A physical or logical branch/location.
- **Columns:** `id`, `tenant_id`, `name`, `code`, `address`, `city`, `country`,
  `timezone`, `currency`, `phone`, `is_active`, `manager_user_id`.
- **Relationships:** parent of `inventory_levels`, `sales`, `shifts`, `cash_drawers`.
- **Indexes:** index on `tenant_id`; unique `(tenant_id, code)`.
- **Multi-tenant:** Tenant (a store belongs to exactly one tenant).

### `tax_rates`
Configurable tax rules per tenant (and optionally per store/product).
- **Columns:** `id`, `tenant_id`, `name` (e.g. "VAT 15%"), `rate_percent`,
  `is_inclusive`, `is_default`, `is_active`.
- **Multi-tenant:** Tenant.

### `discounts`
Reusable discount definitions (manual or rule-based).
- **Columns:** `id`, `tenant_id`, `name`, `type` (`percentage`, `fixed_amount`),
  `value`, `scope` (`item`, `transaction`), `requires_approval`, `starts_at`, `ends_at`,
  `is_active`.
- **Multi-tenant:** Tenant.

---

## Catalog & Inventory

### `categories`
Product categories (supports nesting via `parent_category_id`).
- **Columns:** `id`, `tenant_id`, `name`, `parent_category_id`, `sort_order`, `image_url`.
- **Indexes:** index on `(tenant_id, parent_category_id)`.
- **Multi-tenant:** Tenant.

### `suppliers`
Vendors products are purchased from.
- **Columns:** `id`, `tenant_id`, `name`, `contact_name`, `email`, `phone`, `address`,
  `is_active`.
- **Multi-tenant:** Tenant.

### `units`
Units of measure (piece, kg, liter, box, etc.), tenant-customizable.
- **Columns:** `id`, `tenant_id`, `name`, `abbreviation`.
- **Multi-tenant:** Tenant.

### `products`
Core catalog item (may have variants).
- **Columns:** `id`, `tenant_id`, `category_id`, `supplier_id`, `unit_id`, `name`,
  `description`, `sku`, `barcode`, `type` (`good`, `service`), `has_variants`,
  `cost_price`, `sell_price`, `tax_rate_id`, `track_stock`, `image_url`, `is_active`.
- **Relationships:** parent of `product_variants`, `inventory_levels`; referenced by
  `sale_items.product_id`.
- **Indexes:** index on `tenant_id`; unique `(tenant_id, sku)`; index on `(tenant_id, barcode)`;
  full-text index on `name` for search.
- **Multi-tenant:** Tenant.

### `product_variants`
Variant dimension (size/color/etc.) of a product; when `has_variants = false`, a
product implicitly has one default variant row for uniform inventory logic.
- **Columns:** `id`, `tenant_id`, `product_id`, `sku`, `barcode`, `attributes` (JSONB,
  e.g. `{"size":"M","color":"Red"}`), `cost_price`, `sell_price`, `image_url`, `is_active`.
- **Indexes:** unique `(tenant_id, sku)`; index on `product_id`.
- **Multi-tenant:** Tenant.

### `inventory_levels`
Current stock quantity of a variant **at a specific store**.
- **Columns:** `id`, `tenant_id`, `store_id`, `product_variant_id`, `quantity`
  (numeric, allows fractional for weighted items), `reserved_quantity`,
  `low_stock_threshold`.
- **Relationships:** unique `(store_id, product_variant_id)` — one row per SKU per store.
- **Indexes:** composite unique `(store_id, product_variant_id)`; index on `tenant_id`.
- **Multi-tenant:** Store.

### `stock_movements`
Immutable ledger of every stock change (append-only, auditable).
- **Columns:** `id`, `tenant_id`, `store_id`, `product_variant_id`, `type`
  (`sale`, `refund`, `adjustment`, `transfer_in`, `transfer_out`, `stock_count`,
  `purchase_receipt`), `quantity_delta`, `reference_type` (`sale`, `stock_transfer`,
  `stock_count`, `manual`), `reference_id`, `note`, `created_by_user_id`.
- **Indexes:** index on `(tenant_id, product_variant_id, created_at)`; index on
  `(store_id, created_at)`.
- **Multi-tenant:** Store.
- **Note:** `inventory_levels.quantity` is a derived/cached total maintained
  transactionally alongside each `stock_movements` insert — `stock_movements` is the
  source of truth, `inventory_levels` is the fast-read projection.

### `stock_transfers`
Movement of stock between two stores of the same tenant.
- **Columns:** `id`, `tenant_id`, `from_store_id`, `to_store_id`, `status`
  (`draft`, `in_transit`, `completed`, `cancelled`), `requested_by_user_id`,
  `completed_at`.
- **Relationships:** parent of `stock_transfer_items`.
- **Multi-tenant:** Tenant (spans two stores).

### `stock_transfer_items`
- **Columns:** `id`, `stock_transfer_id`, `product_variant_id`, `quantity`.

### `stock_counts` (stock opname)
Periodic physical count reconciliation.
- **Columns:** `id`, `tenant_id`, `store_id`, `status` (`open`, `completed`),
  `started_by_user_id`, `completed_at`.
- **Relationships:** parent of `stock_count_items`.
- **Multi-tenant:** Store.

### `stock_count_items`
- **Columns:** `id`, `stock_count_id`, `product_variant_id`, `expected_quantity`,
  `counted_quantity`, `variance` (generated).

---

## Sales / POS

### `shifts`
A cashier's till session at a store.
- **Columns:** `id`, `tenant_id`, `store_id`, `user_id`, `opened_at`, `closed_at`,
  `opening_cash_amount`, `closing_cash_amount`, `expected_cash_amount`,
  `cash_difference`, `status` (`open`, `closed`).
- **Relationships:** parent of `sales`, `cash_drawer_movements`.
- **Indexes:** index on `(store_id, status)`.
- **Multi-tenant:** Store.

### `cash_drawer_movements`
Cash in/out events within a shift (paid-in, paid-out, opening float, closing count).
- **Columns:** `id`, `tenant_id`, `shift_id`, `type` (`open`, `cash_in`, `cash_out`,
  `close`), `amount`, `reason`, `created_by_user_id`.
- **Multi-tenant:** Store.

### `customers`
Tenant's customer directory (for CRM/loyalty and receipt attribution).
- **Columns:** `id`, `tenant_id`, `full_name`, `email`, `phone`, `birthday`,
  `customer_group_id`, `loyalty_points_balance`, `total_spent` (denormalized),
  `notes`, `marketing_opt_in`.
- **Indexes:** index on `tenant_id`; index on `(tenant_id, phone)`, `(tenant_id, email)`.
- **Multi-tenant:** Tenant (customers can transact at any store of the tenant).

### `customer_groups`
Segmentation/membership tiers.
- **Columns:** `id`, `tenant_id`, `name`, `discount_percent_default`.
- **Multi-tenant:** Tenant.

### `loyalty_programs`
Tenant-level loyalty configuration.
- **Columns:** `id`, `tenant_id`, `points_per_currency_unit`, `redemption_rate`,
  `is_active`.
- **Multi-tenant:** Tenant.

### `loyalty_transactions`
Ledger of points earned/redeemed (mirrors `stock_movements` pattern for auditability).
- **Columns:** `id`, `tenant_id`, `customer_id`, `type` (`earn`, `redeem`, `adjust`,
  `expire`), `points_delta`, `sale_id` (nullable), `note`.
- **Indexes:** index on `(customer_id, created_at)`.
- **Multi-tenant:** Tenant.

### `sales`
A completed (or held) transaction.
- **Columns:** `id`, `tenant_id`, `store_id`, `shift_id`, `cashier_id`, `customer_id`
  (nullable), `sale_number` (human-readable, tenant+store sequential), `client_reference_id`
  (UUID from offline client, unique — idempotency key), `status`
  (`open_ticket`, `held`, `completed`, `refunded`, `partially_refunded`, `void`),
  `subtotal`, `discount_total`, `tax_total`, `total`, `note`, `synced_at`.
- **Relationships:** parent of `sale_items`, `payments`, `refunds`.
- **Indexes:** unique `(tenant_id, client_reference_id)`; index on `(store_id, created_at)`;
  index on `(customer_id)`; unique `(store_id, sale_number)`.
- **Multi-tenant:** Store.

### `sale_items`
Line items of a sale.
- **Columns:** `id`, `sale_id`, `product_variant_id`, `description` (snapshot of
  product name at sale time), `quantity`, `unit_price` (snapshot), `discount_amount`,
  `tax_amount`, `line_total`.
- **Indexes:** index on `sale_id`; index on `product_variant_id` (for sales-by-product
  reporting).
- **Note:** price/name are snapshotted at time of sale so later catalog edits never
  alter historical transactions.

### `payments`
One or more payments applied to a sale (supports split payment).
- **Columns:** `id`, `tenant_id`, `sale_id`, `method` (`cash`, `card`, `digital_wallet`,
  `store_credit`, `other`), `amount`, `tendered_amount` (cash), `change_amount`,
  `gateway_reference` (nullable, for card/digital), `status`
  (`completed`, `failed`, `voided`).
- **Indexes:** index on `sale_id`.
- **Multi-tenant:** Store (inherits from sale).

### `refunds`
Refund/return against a completed sale.
- **Columns:** `id`, `tenant_id`, `sale_id`, `processed_by_user_id`, `reason`,
  `total_refunded`, `status` (`completed`, `pending`).
- **Relationships:** parent of `refund_items`.
- **Multi-tenant:** Store.

### `refund_items`
- **Columns:** `id`, `refund_id`, `sale_item_id`, `quantity`, `amount`. Each refund
  item also writes a compensating `stock_movements` row when the product is restocked.

---

## SaaS Billing / Subscriptions

### `subscription_packages`
Master-Admin-defined plans.
- **Columns:** `id`, `name`, `description`, `billing_cycle` (`monthly`, `annual`),
  `price_amount`, `price_currency`, `max_stores`, `max_employees`, `max_products`,
  `max_transactions_per_month`, `features` (JSONB array of feature flags, e.g.
  `["loyalty","multi_store","advanced_reports","api_access"]`), `is_public`,
  `is_active`, `sort_order`.
- **Relationships:** referenced by `tenant_subscriptions.package_id`.
- **Multi-tenant:** Platform (shared across all tenants).

### `tenant_subscriptions`
The tenant's current/historical subscription record.
- **Columns:** `id`, `tenant_id`, `package_id`, `status`
  (`trial`, `active`, `expired`, `suspended`, `cancelled`), `trial_ends_at`,
  `current_period_start`, `current_period_end`, `grace_period_ends_at`,
  `cancel_at_period_end`, `auto_renew`.
- **Indexes:** index on `tenant_id`; index on `status` (for the daily billing-status job).
- **Multi-tenant:** Tenant (one active row per tenant; history kept via `invoices`).

### `invoices`
Billing documents generated per subscription period.
- **Columns:** `id`, `tenant_id`, `tenant_subscription_id`, `invoice_number`, `amount`,
  `currency`, `status` (`draft`, `pending`, `paid`, `overdue`, `void`), `issued_at`,
  `due_at`, `paid_at`, `pdf_url`.
- **Indexes:** unique `invoice_number`; index on `tenant_id`.
- **Multi-tenant:** Tenant.

### `payment_transactions`
Raw record of every gateway attempt (subscription payments).
- **Columns:** `id`, `tenant_id`, `invoice_id`, `provider` (`ipaymu`, `stripe`,
  `paypal`, `manual`), `provider_transaction_id`, `amount`, `currency`, `status`
  (`pending`, `success`, `failed`, `refunded`), `gateway_payload` (JSONB, raw
  callback), `verified_by_admin_id` (nullable, for manual verification).
- **Indexes:** index on `invoice_id`; index on `provider_transaction_id`.
- **Multi-tenant:** Tenant.

---

## Platform CMS & Support

### `website_layout_settings`
Master-Admin-editable structure of the public site (which sections appear, order,
theme tokens).
- **Columns:** `id`, `key` (e.g. `homepage_layout`), `value` (JSONB), `updated_by_admin_id`.
- **Multi-tenant:** Platform.

### `cms_content`
Editable content blocks (hero, features, pricing copy, testimonials, FAQ, footer,
legal pages).
- **Columns:** `id`, `page` (`home`, `features`, `pricing`, `about`, `terms`,
  `privacy`, `contact`), `section_key`, `content` (JSONB, supports rich text/i18n-ready
  structure), `is_published`, `updated_by_admin_id`.
- **Indexes:** index on `(page, section_key)`.
- **Multi-tenant:** Platform.

### `announcements`
System-wide or package-targeted announcements shown in tenant dashboards.
- **Columns:** `id`, `title`, `body`, `target` (`all`, `package_id`, `trial_only`),
  `starts_at`, `ends_at`, `is_active`.
- **Multi-tenant:** Platform.

### `support_tickets`
Subscriber inquiries handled by platform staff.
- **Columns:** `id`, `tenant_id`, `created_by_user_id`, `subject`, `status`
  (`open`, `pending`, `resolved`, `closed`), `priority`, `assigned_admin_id`.
- **Relationships:** parent of `support_ticket_messages`.
- **Multi-tenant:** Tenant (owned by a tenant, visible to platform staff).

### `support_ticket_messages`
- **Columns:** `id`, `support_ticket_id`, `author_type` (`tenant_user`, `admin`),
  `author_id`, `body`, `attachment_url`.

---

## Integrations

### `integrations`
Available third-party integration types (catalog, platform-managed).
- **Columns:** `id`, `key` (`accounting_quickbooks`, `whatsapp`, `email_smtp`,
  `webhook`, `printer`, `cash_drawer`, `barcode_scanner`, `ecommerce_shopify`), `name`,
  `category`.
- **Multi-tenant:** Platform (catalog).

### `tenant_integrations`
A tenant's configured instance of an integration.
- **Columns:** `id`, `tenant_id`, `integration_id`, `config` (JSONB, credentials/settings
  — encrypted at rest), `is_active`, `connected_by_user_id`.
- **Multi-tenant:** Tenant.

### `webhooks`
Tenant-configured outbound webhooks for the public API.
- **Columns:** `id`, `tenant_id`, `url`, `event_types` (text array, e.g.
  `["sale.completed","product.low_stock"]`), `secret`, `is_active`.
- **Multi-tenant:** Tenant.

### `webhook_deliveries`
Delivery attempts/log for observability and retries.
- **Columns:** `id`, `webhook_id`, `event_type`, `payload` (JSONB), `response_status`,
  `attempt_count`, `delivered_at`.
- **Multi-tenant:** Tenant (inherits from webhook).

---

## Cross-Cutting

### `audit_logs`
Immutable log of sensitive actions across the whole platform.
- **Columns:** `id`, `tenant_id` (nullable for platform-level actions), `actor_type`
  (`user`, `admin_user`, `system`), `actor_id`, `action` (e.g.
  `subscription.status_changed`, `refund.created`, `role.permissions_updated`),
  `entity_type`, `entity_id`, `before` (JSONB), `after` (JSONB), `ip_address`.
- **Indexes:** index on `(tenant_id, created_at)`; index on `(entity_type, entity_id)`.
- **Multi-tenant:** Tenant-nullable (platform actions have `tenant_id = null`).

### `pending_sync_events` *(mobile-originated, mirrored server-side for observability)*
- **Columns:** `id`, `tenant_id`, `store_id`, `device_id`, `event_type`, `payload`
  (JSONB), `client_created_at`, `status` (`applied`, `conflict`, `rejected`).
- **Multi-tenant:** Store.

---

## Entity Relationship Summary

```
tenants 1─N stores 1─N inventory_levels N─1 product_variants N─1 products N─1 categories
tenants 1─N users N─1 roles N─N permissions
tenants 1─N tenant_subscriptions N─1 subscription_packages
tenant_subscriptions 1─N invoices 1─N payment_transactions
stores 1─N shifts 1─N sales 1─N sale_items N─1 product_variants
sales 1─N payments
sales 1─N refunds 1─N refund_items
tenants 1─N customers 1─N loyalty_transactions
stores 1─N stock_movements N─1 product_variants
stores(from) 1─N stock_transfers N─1 stores(to) 1─N stock_transfer_items
tenants 1─N support_tickets 1─N support_ticket_messages
```

All tenant-scoped tables share one non-negotiable rule: **no query executes without a
`tenant_id` filter**, enforced structurally by RLS so this rule survives future
engineers, not just code review discipline.
