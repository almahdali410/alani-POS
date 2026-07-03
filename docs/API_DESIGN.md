# API Design

Base URL: `https://api.alanipos.com/api/v1`. All bodies are JSON. All authenticated
requests send `Authorization: Bearer <access_token>`. Tenant scoping is derived from the
token — endpoints never take a `tenant_id` parameter from the client.

**Conventions**
- Resource collections: `GET /resource`, `POST /resource`, `GET /resource/:id`,
  `PATCH /resource/:id`, `DELETE /resource/:id` (soft delete).
- Pagination: `?page=1&pageSize=20`, response envelope `{ data: [...], meta: { page,
  pageSize, total } }`.
- Errors: `{ error: { code, message, details? } }` with standard HTTP status codes
  (`400` validation, `401` unauthenticated, `403` forbidden/permission or plan-limit,
  `404` not found, `409` conflict, `429` rate-limited).
- Idempotency: mutation endpoints from mobile accept an `Idempotency-Key` header
  (mapped to `client_reference_id` where applicable).

---

## 1. Authentication

| Method | Path | Purpose |
|---|---|---|
| POST | `/auth/login` | Email/password login (tenant users) |
| POST | `/auth/refresh` | Exchange refresh token for new access token |
| POST | `/auth/logout` | Revoke current refresh token |
| POST | `/auth/2fa/verify` | Complete 2FA challenge |
| POST | `/auth/password/forgot` | Send reset email |
| POST | `/auth/password/reset` | Reset with token |
| POST | `/admin/auth/login` | Master Admin login (separate endpoint/realm) |

**`POST /auth/login`**
```json
// Request
{ "email": "owner@cafejoy.com", "password": "••••••••" }

// Response 200
{
  "accessToken": "eyJhbGciOi...",
  "refreshToken": "8f3d1c2a-...",
  "user": {
    "id": "u_123", "fullName": "Alex Rivera", "email": "owner@cafejoy.com",
    "role": "Owner", "tenantId": "t_456", "tenantStatus": "trial"
  }
}
```

---

## 2. Tenant Registration & Onboarding

| Method | Path | Purpose |
|---|---|---|
| POST | `/tenants/register` | Create tenant + owner user, start free trial |
| GET | `/tenants/me` | Current tenant profile |
| PATCH | `/tenants/me` | Update business profile |
| POST | `/tenants/me/stores` | Create first/additional store |

**`POST /tenants/register`**
```json
// Request
{
  "businessName": "Cafe Joy",
  "businessType": "cafe_restaurant",
  "country": "US",
  "currency": "USD",
  "ownerFullName": "Alex Rivera",
  "email": "owner@cafejoy.com",
  "password": "••••••••"
}

// Response 201
{
  "tenant": { "id": "t_456", "slug": "cafe-joy", "status": "trial",
              "trialEndsAt": "2026-08-03T00:00:00Z" },
  "accessToken": "eyJhbGciOi...",
  "refreshToken": "8f3d1c2a-..."
}
```

---

## 3. Subscription Management

| Method | Path | Purpose |
|---|---|---|
| GET | `/billing/packages` | List public subscription packages |
| GET | `/billing/subscription` | Current tenant's subscription status |
| POST | `/billing/subscription/checkout` | Start checkout for a package (creates invoice + gateway session) |
| POST | `/billing/subscription/cancel` | Cancel (effective at period end) |
| GET | `/billing/invoices` | Invoice history |
| GET | `/billing/invoices/:id/pdf` | Download invoice PDF |

**`POST /billing/subscription/checkout`**
```json
// Request
{ "packageId": "pkg_pro_annual", "provider": "ipaymu" }

// Response 200
{
  "invoice": { "id": "inv_789", "amount": 240.00, "currency": "USD", "status": "pending" },
  "payment": {
    "provider": "ipaymu",
    "redirectUrl": "https://my.ipaymu.com/pay/session_abc123"
  }
}
```

---

## 4. Payment Callback (iPaymu)

| Method | Path | Purpose |
|---|---|---|
| POST | `/payments/ipaymu/callback` | Server-to-server webhook from iPaymu |
| GET | `/billing/subscription/checkout/:invoiceId/status` | Client polls/confirms result after redirect back |

**`POST /payments/ipaymu/callback`** (called by iPaymu, verified via HMAC signature header)
```json
// Request (from iPaymu)
{
  "trx_id": "IPM-998877",
  "reference_id": "inv_789",
  "status": "success",
  "amount": 240.00,
  "payment_method": "card",
  "signature": "b3f1...9a"
}

// Response 200
{ "received": true }
```
Effect: `invoice.status = paid`, `payment_transactions` row updated, tenant's
`tenant_subscriptions.status = active`, `current_period_end` extended by the package's
billing cycle.

---

## 5. Product & Catalog Management

| Method | Path | Purpose |
|---|---|---|
| GET | `/products` | List/search products (`?q=`, `?categoryId=`, `?barcode=`) |
| POST | `/products` | Create product |
| GET | `/products/:id` | Product detail (with variants) |
| PATCH | `/products/:id` | Update product |
| DELETE | `/products/:id` | Archive product |
| POST | `/products/:id/variants` | Add variant |
| GET | `/categories` | List categories |
| POST | `/categories` | Create category |
| POST | `/products/import` | Bulk import (CSV/XLSX) |
| GET | `/products/export` | Bulk export |

**`POST /products`**
```json
// Request
{
  "name": "Cappuccino",
  "categoryId": "cat_beverages",
  "sku": "BEV-001",
  "barcode": "8991002100015",
  "type": "good",
  "costPrice": 1.20,
  "sellPrice": 3.50,
  "taxRateId": "tax_vat10",
  "trackStock": true,
  "variants": [{ "sku": "BEV-001-S", "attributes": { "size": "Small" }, "sellPrice": 3.00 },
               { "sku": "BEV-001-L", "attributes": { "size": "Large" }, "sellPrice": 3.50 }]
}

// Response 201 → full product object with generated ids
```

---

## 6. Inventory Management

| Method | Path | Purpose |
|---|---|---|
| GET | `/inventory?storeId=` | Stock levels for a store |
| POST | `/inventory/adjustments` | Manual stock adjustment |
| POST | `/inventory/transfers` | Create stock transfer between stores |
| POST | `/inventory/transfers/:id/complete` | Mark transfer received |
| POST | `/inventory/stock-counts` | Start a stock count |
| PATCH | `/inventory/stock-counts/:id` | Submit counted quantities |
| GET | `/inventory/low-stock` | Products below threshold |
| GET | `/suppliers` | Supplier list |

**`POST /inventory/adjustments`**
```json
// Request
{
  "storeId": "store_1",
  "productVariantId": "var_bev001s",
  "quantityDelta": -2,
  "reason": "Breakage"
}
// Response 201 → resulting stock_movements + updated inventory_levels row
```

---

## 7. POS Transactions

| Method | Path | Purpose |
|---|---|---|
| POST | `/pos/sales` | Create a completed sale |
| POST | `/pos/sales/hold` | Park/hold an order |
| GET | `/pos/sales/held` | List held orders/open tickets for a store |
| POST | `/pos/sales/:id/resume` | Resume a held order into active cart |
| POST | `/pos/sales/:id/refund` | Full/partial refund |
| POST | `/pos/shifts/open` | Open a cashier shift |
| POST | `/pos/shifts/:id/close` | Close shift (cash count) |
| POST | `/pos/sync/push` | Batched offline sale sync from mobile |
| GET | `/pos/sync/pull?since=` | Incremental reference-data pull |

**`POST /pos/sales`**
```json
// Request
{
  "storeId": "store_1",
  "shiftId": "shift_9",
  "clientReferenceId": "b6b1f5b0-...-offline-uuid",
  "customerId": "cust_44",
  "items": [
    { "productVariantId": "var_bev001s", "quantity": 2, "unitPrice": 3.00, "discountAmount": 0 }
  ],
  "payments": [
    { "method": "cash", "amount": 6.60, "tenderedAmount": 10.00 }
  ],
  "note": null
}

// Response 201
{
  "sale": {
    "id": "sale_5001", "saleNumber": "S1-000512", "status": "completed",
    "subtotal": 6.00, "taxTotal": 0.60, "total": 6.60,
    "payments": [{ "method": "cash", "amount": 6.60, "changeAmount": 3.40 }]
  }
}
```

**`POST /pos/sync/push`** (offline batch)
```json
// Request
{ "deviceId": "device_abc", "events": [
  { "clientReferenceId": "uuid-1", "type": "sale", "payload": { "...": "..." } },
  { "clientReferenceId": "uuid-2", "type": "sale", "payload": { "...": "..." } }
]}

// Response 200
{ "results": [
  { "clientReferenceId": "uuid-1", "status": "applied", "serverId": "sale_5002" },
  { "clientReferenceId": "uuid-2", "status": "conflict", "reason": "stock_negative", "serverId": "sale_5003" }
]}
```

---

## 8. Customer Management

| Method | Path | Purpose |
|---|---|---|
| GET | `/customers` | Search/list customers |
| POST | `/customers` | Create customer |
| GET | `/customers/:id` | Profile + purchase history |
| PATCH | `/customers/:id` | Update |
| POST | `/customers/:id/loyalty/adjust` | Manual points adjustment |
| GET | `/customer-groups` | List groups/tiers |

---

## 9. Employee Management

| Method | Path | Purpose |
|---|---|---|
| GET | `/employees` | List staff |
| POST | `/employees` | Invite/create employee |
| PATCH | `/employees/:id` | Update role/status |
| POST | `/employees/:id/clock-in` | Attendance clock-in |
| POST | `/employees/:id/clock-out` | Attendance clock-out |
| GET | `/employees/:id/performance` | Sales performance summary |
| GET | `/roles` | List roles |
| POST | `/roles` | Create custom role |
| PATCH | `/roles/:id/permissions` | Update role's permission set |

---

## 10. Reports

| Method | Path | Purpose |
|---|---|---|
| GET | `/reports/sales-summary?range=` | Gross/net sales, discounts, tax, refunds |
| GET | `/reports/sales-by-product` | Best/worst sellers |
| GET | `/reports/sales-by-category` | Category breakdown |
| GET | `/reports/sales-by-employee` | Cashier performance |
| GET | `/reports/sales-by-store` | Multi-store comparison |
| GET | `/reports/export?type=pdf|xlsx` | Export current report view |

---

## 11. Master Admin Management

All under `/admin/*`, authenticated as an `admin_users` principal.

| Method | Path | Purpose |
|---|---|---|
| GET | `/admin/dashboard` | Platform KPIs (subscribers, MRR/ARR, trials, churn) |
| GET | `/admin/tenants` | List/search all tenants |
| GET | `/admin/tenants/:id` | Tenant detail + usage |
| POST | `/admin/tenants/:id/suspend` | Suspend a tenant |
| POST | `/admin/tenants/:id/reactivate` | Reactivate |
| POST | `/admin/tenants/:id/subscription/extend` | Manually extend subscription |
| GET | `/admin/packages` | List packages |
| POST | `/admin/packages` | Create package |
| PATCH | `/admin/packages/:id` | Update limits/features/pricing |
| GET | `/admin/invoices` | All invoices across tenants |
| POST | `/admin/invoices/:id/mark-paid` | Manual payment verification |
| GET | `/admin/support-tickets` | Support queue |
| POST | `/admin/announcements` | Create announcement |

**`GET /admin/dashboard`**
```json
{
  "totalSubscribers": 4213,
  "activeSubscriptions": 3520,
  "trialUsers": 481,
  "expiredAccounts": 212,
  "mrr": 58230.00,
  "arr": 698760.00,
  "newSubscribersThisMonth": 187,
  "paymentSuccessRate": 0.94
}
```

---

## 12. CMS / Website Layout Management

| Method | Path | Purpose |
|---|---|---|
| GET | `/admin/cms/pages/:page` | Get content blocks for a page |
| PATCH | `/admin/cms/pages/:page/sections/:key` | Update a content section |
| GET | `/admin/cms/layout` | Get global layout config (nav, theme, section order) |
| PATCH | `/admin/cms/layout` | Update layout config |
| GET | `/public/cms/pages/:page` | Public read endpoint consumed by the marketing site |

---

## 13. Integrations & Public API

| Method | Path | Purpose |
|---|---|---|
| GET | `/integrations` | Catalog of available integrations |
| POST | `/integrations/:key/connect` | Connect/configure for tenant |
| DELETE | `/integrations/:key` | Disconnect |
| POST | `/webhooks` | Register outbound webhook |
| GET | `/webhooks/:id/deliveries` | Delivery log |
| GET | `/public-api/v1/products` | Third-party read access (API key auth) |
| GET | `/public-api/v1/sales` | Third-party read access (API key auth) |

All `/public-api/*` routes require `X-Api-Key` and are rate-limited per key
independently of the dashboard session limits.
