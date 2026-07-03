# System Architecture

## Recommended Technology Stack

| Layer | Choice | Why |
|---|---|---|
| Backend API | **Node.js + NestJS (TypeScript)** | Batteries-included DI/module system suits a large multi-domain monolith-first backend; strong typing shared with frontend via generated OpenAPI/DTO types |
| Primary database | **PostgreSQL** | Strong relational integrity for financial data, native Row-Level Security for tenant isolation, JSONB for flexible fields (CMS content, product attributes) |
| Cache / queue broker | **Redis** | Session cache, rate limiting, pub/sub, and backing store for BullMQ |
| Background jobs | **BullMQ (Redis-backed)** | Renewal reminders, invoice generation, webhook delivery, report generation, low-stock alert emails |
| Object storage | **S3-compatible (AWS S3 / Cloudflare R2)** | Product images, receipts (PDF), CMS media, import/export files |
| Web frontend | **Next.js (React) + TypeScript + Tailwind CSS** | SSR for the public marketing site/SEO, SPA-like behavior for dashboard & POS, one codebase for both |
| Mobile (Android + iOS) | **Flutter (Dart)** | Single codebase for true native Android & iOS builds; excellent offline storage story (Drift/SQLite) and hardware access (barcode scanners, printers, cash drawers via plugins) |
| Local mobile DB | **SQLite via Drift** | Offline-first POS cache and outbox queue |
| Master Admin Panel | Same Next.js app, separate route group/app, shared design system | Avoids a second frontend stack |
| API contract | **REST + OpenAPI 3.1**, versioned `/api/v1` | Simple to document, cache, and consume from Flutter/Next.js/webhooks; GraphQL can be added later for analytics if needed |
| Auth | **JWT (access + refresh) + OAuth2 for future SSO** | Stateless API auth across web/mobile, refresh-token rotation, per-tenant claims |
| Payments | **iPaymu (subscriptions) behind a `PaymentGatewayProvider` interface** | Visa/Mastercard at launch; Stripe/PayPal pluggable later without touching billing domain logic |
| Search | **PostgreSQL full-text search (Phase 1-4), OpenSearch/Meilisearch (Phase 6+)** | Product/customer search at scale |
| Analytics store | **Postgres read replica + materialized views (Phase 1-6); ClickHouse (Phase 7)** | Keep OLTP simple early, add an OLAP layer only once transaction volume demands it |
| Infra / hosting | **Docker containers on Kubernetes (or managed ECS/Cloud Run for smaller scale)** | Horizontal scaling per service, environment parity |
| CI/CD | **GitHub Actions** | Build/test/lint/deploy pipelines per app (API, web, mobile) |
| Monitoring | **Prometheus + Grafana (metrics), Sentry (errors), Loki/ELK (logs)** | Full observability stack |
| CDN | **CloudFront / Cloudflare** | Static assets, public website, receipt PDFs |
| Notifications | **Email (SES/SendGrid) + WhatsApp Business API (integration-ready) + Push (FCM/APNs)** | Renewal reminders, receipts, low-stock alerts, marketing |

**Why a modular monolith over microservices at launch:** a SaaS POS with a small team
benefits from one deployable backend with clear internal module boundaries
(`tenants`, `billing`, `catalog`, `inventory`, `pos`, `crm`, `reporting`, `cms`,
`admin`). Modules communicate in-process, each owns its own database schema
namespace, and the boundaries are drawn so any module can be extracted into its own
service later (billing and reporting are the most likely first extractions once scale
demands it).

---

## High-Level System Diagram

```
                    ┌─────────────────────────────────────────────┐
                    │              Public Internet                │
                    └─────────────────────────────────────────────┘
                                       │
                     ┌─────────────────┼──────────────────┐
                     ▼                 ▼                  ▼
             ┌───────────────┐ ┌───────────────┐  ┌────────────────┐
             │  Public Website│ │  Web App (SPA)│  │ Android/iOS App│
             │  (Next.js SSR) │ │  (Next.js)     │  │ (Flutter)      │
             └───────┬────────┘ └───────┬────────┘  └────────┬───────┘
                     │                  │                    │
                     └─────────┬────────┴──────────┬─────────┘
                                ▼                   ▼
                        ┌───────────────────────────────┐
                        │        API Gateway / LB        │
                        │  (rate limiting, TLS, WAF)      │
                        └───────────────┬─────────────────┘
                                        ▼
                        ┌───────────────────────────────┐
                        │      NestJS API (modular)      │
                        │  Auth│Tenants│Billing│Catalog   │
                        │  Inventory│POS│CRM│Reporting    │
                        │  CMS│Admin│Integrations│Webhooks │
                        └───┬──────────┬──────────┬───────┘
                            ▼          ▼          ▼
                    ┌───────────┐ ┌─────────┐ ┌───────────┐
                    │ PostgreSQL │ │  Redis  │ │ S3 Storage │
                    │ (multi-    │ │ (cache, │ │ (media,    │
                    │  tenant)   │ │  queue) │ │  receipts) │
                    └───────────┘ └────┬────┘ └───────────┘
                                       ▼
                              ┌─────────────────┐
                              │ Background Workers│
                              │ (BullMQ: billing, │
                              │ webhooks, reports, │
                              │ notifications)     │
                              └─────────────────┘
                                       │
                     ┌─────────────────┼──────────────────┐
                     ▼                 ▼                  ▼
             ┌───────────────┐ ┌───────────────┐  ┌────────────────┐
             │  iPaymu (Visa/ │ │  Email/WhatsApp│  │  Third-party    │
             │  Mastercard)   │ │  Push (FCM/APNs)│ │  integrations   │
             └───────────────┘ └───────────────┘  └────────────────┘
```

---

## Multi-Tenant Strategy

**Chosen model: shared database, shared schema, `tenant_id` on every tenant-scoped
table**, hardened with PostgreSQL **Row-Level Security (RLS)**.

Rationale vs. alternatives:

| Strategy | Pros | Cons | Verdict |
|---|---|---|---|
| Database-per-tenant | Strongest isolation, easy per-tenant backup/restore | Operationally expensive at thousands of tenants; migrations must fan out | Reserved for a future "dedicated/enterprise" tier only |
| Schema-per-tenant | Good isolation, single DB server | Connection pooling and migration complexity grows linearly with tenant count | Not chosen; revisit only if a large enterprise customer requires physical isolation |
| **Shared schema + `tenant_id` + RLS** | Simple migrations, efficient pooling, scales to tens of thousands of tenants, cheapest to operate | Requires discipline: every query must be tenant-scoped | **Chosen** — enforced at three layers below |

**Enforcement layers (defense in depth):**
1. **Application layer:** every authenticated request carries a `tenant_id` resolved
   from the JWT; a NestJS request-scoped `TenantContext` is injected into every
   repository call. Master Admin requests bypass tenant scoping explicitly and are
   audit-logged.
2. **Database layer:** PostgreSQL RLS policies on every tenant-scoped table
   (`USING (tenant_id = current_setting('app.tenant_id')::uuid)`), set per-connection
   via `SET LOCAL app.tenant_id` inside each request's transaction. This means even a
   bug in application code cannot leak cross-tenant rows.
3. **Index layer:** every tenant-scoped table is indexed with `tenant_id` as the
   leading column of its primary lookup indexes, keeping per-tenant queries fast as
   the table grows across all tenants.

**Store-level scoping** works the same way one level down: many tables also carry a
`store_id`, and permission checks combine `tenant_id + store_id` for store-scoped roles
(e.g., a Store Manager's queries are additionally filtered to their assigned stores).

---

## Authentication & Authorization

- **Authentication:** email + password (bcrypt/argon2 hashing) with JWT access tokens
  (short-lived, ~15 min) and refresh tokens (long-lived, rotated on use, revocable).
  Optional TOTP-based 2FA for Tenant Owners and the Master Admin. Mobile apps persist
  refresh tokens in secure device storage (Keychain/Keystore).
- **Tenant resolution:** JWT includes `tenant_id`, `user_id`, `role_id`, and an
  `is_master_admin` flag. Master Admin accounts live in a separate `admin_users` table
  outside tenant scoping entirely.
- **Authorization:** RBAC with a `permissions` table of granular
  `resource:action` strings, `roles` grouping permissions, and `role_id` on each
  tenant user. A `PermissionsGuard` decorator on every controller method declares the
  required permission(s); denial returns `403`.
- **API keys:** tenants can generate scoped API keys (for the public API/integrations)
  tied to a subset of permissions and rate-limited independently of user sessions.
- **Session security:** refresh-token rotation with reuse detection (a reused/stolen
  refresh token invalidates the whole token family), device/session listing and
  remote sign-out from the tenant dashboard.

---

## Subscription & Billing Flow (Architecture View)

See [FLOWS.md](./FLOWS.md#1-subscription--payment-flow) for the full sequence. At the
architecture level, billing is its own NestJS module (`billing`) owning:
`subscription_packages`, `tenant_subscriptions`, `invoices`, `payment_transactions`.
It exposes a `PaymentGatewayProvider` interface:

```ts
interface PaymentGatewayProvider {
  createPayment(input: CreatePaymentInput): Promise<PaymentSession>;
  handleCallback(payload: unknown, signature: string): Promise<PaymentResult>;
  refund(transactionId: string, amount?: number): Promise<RefundResult>;
}
```

`IpaymuProvider` implements this today; `StripeProvider` / `PaypalProvider` can be
added later and selected per-tenant-country via a provider registry, with zero changes
to subscription state-machine logic. A scheduled job (BullMQ cron) evaluates
subscriptions daily for trial-expiry, renewal-due, and grace-period-expiry transitions.

---

## Payment Gateway Integration Flow (iPaymu)

1. Tenant Owner selects a package + annual billing on the Billing page.
2. API creates a `pending` `invoice` + `payment_transaction`, calls iPaymu's
   "create payment" API server-side (never expose iPaymu secret keys to clients),
   receives a redirect/session URL.
3. Client redirects the user to iPaymu's hosted payment page (Visa/Mastercard).
4. iPaymu sends a **server-to-server callback (webhook)** to
   `POST /api/v1/payments/ipaymu/callback` with the transaction status, signed with a
   shared secret/HMAC that the API verifies before trusting the payload.
5. On verified success: `payment_transaction.status = success`, `invoice.status =
   paid`, `tenant_subscription.status = active`, `current_period_end` extended by one
   year.
6. On failure/cancel: `payment_transaction.status = failed`; subscription remains in
   its prior state (`trial`/`expired`); the tenant is shown a retry CTA.
7. All callback payloads are stored raw (`payment_transactions.gateway_payload`
   JSONB) for auditability and dispute handling.
8. Master Admin can manually mark an invoice paid, extend a subscription, or issue a
   refund — every manual action writes to `audit_logs`.

---

## Offline-First Mobile Sync Strategy

The POS must function with zero connectivity at the register. Design:

- **Local-first writes:** every POS action (sale, refund, held order, shift
  open/close) is first written to the local SQLite store (via Drift) inside an
  **outbox table** (`pending_sync_events`) with a client-generated UUID and a
  monotonic local sequence number.
- **Optimistic UI:** the POS UI reflects the local write immediately — the cashier is
  never blocked on network latency.
- **Sync engine:** a background sync worker (foreground service on Android / background
  task on iOS, plus sync-on-resume and sync-on-connectivity-change) pushes queued
  events to `POST /api/v1/sync/push` in batches, idempotent by client UUID.
- **Conflict handling:** sales are append-only (no destructive edits), so most
  conflicts are avoided by design. Stock-quantity conflicts (two devices selling the
  last unit offline) are resolved server-side: stock can go negative temporarily, the
  event is flagged `stock_conflict = true`, and it surfaces in a Master/Tenant
  "sync issues" review queue rather than silently failing.
- **Pull sync:** reference data (catalog, prices, tax rules, customers, employee
  permissions) is pulled incrementally via `GET /api/v1/sync/pull?since=<cursor>`
  using per-table updated-at watermarks, cached locally, and refreshed on app foreground
  and periodically while online.
- **Server as source of truth:** once synced, the server-assigned canonical
  `sale.id`/`sale_number` is written back to the local record; the local UUID is kept
  as `client_reference_id` for idempotency and receipt reprinting.
- **Data retention on device:** a rolling window (e.g., last 30-90 days or N MB) of
  synced sales is kept locally for offline reporting/reprints; older data is pruned
  after confirmed sync.

---

## Security Best Practices

- TLS everywhere (HSTS enabled); mobile apps pin the API's certificate authority.
- Tenant isolation via RLS (above) as the primary defense against IDOR/cross-tenant
  data leaks; every list/detail endpoint additionally asserts tenant/store ownership.
- Password hashing with argon2id; rate-limited login and password-reset endpoints;
  account lockout after repeated failures.
- Input validation via DTO class-validators on every endpoint; parameterized queries
  only (ORM), no raw string SQL concatenation.
- Secrets (iPaymu keys, JWT signing keys, DB credentials) in a secrets manager
  (AWS Secrets Manager / GCP Secret Manager / Vault), never in source control.
- Webhook signature verification (HMAC) for all inbound gateway callbacks.
- Audit logging (`audit_logs`) for all sensitive writes: subscription changes,
  permission changes, refunds, price overrides, Master Admin actions.
- Role-based field-level redaction (e.g., Cashier role never receives cost-price data
  in API responses).
- PCI DSS scope minimization: **card data never touches Alani POS servers** — iPaymu's
  hosted payment page handles PAN entry; only tokens/transaction references are stored.
- Rate limiting and WAF at the gateway layer; brute-force and abuse protection on
  public endpoints (registration, login, trial signup).
- Regular dependency scanning (Dependabot/Snyk) and static analysis in CI.
- GDPR-conscious design: tenant data export and account/data deletion endpoints,
  configurable data retention, clear privacy policy content managed via CMS.

---

## Deployment Architecture

- **Environments:** `dev` → `staging` → `production`, each an isolated
  Kubernetes namespace (or isolated cloud project), isolated databases.
- **API:** stateless NestJS pods behind a load balancer, horizontal pod autoscaling on
  CPU/queue depth; DB connections pooled via PgBouncer.
- **Workers:** separate deployment from the API so background job load never starves
  request-serving capacity.
- **Database:** managed PostgreSQL (RDS/Cloud SQL) with a primary + read replica;
  replica used for heavy reporting queries.
- **Static/public site:** deployed to CDN edge (Next.js SSG/ISR) for fast global
  load times and SEO.
- **Mobile releases:** Flutter builds via CI (Fastlane) to Google Play (internal →
  closed → open → production tracks) and TestFlight → App Store.
- **Blue/green or rolling deploys** for the API to avoid downtime; DB migrations run
  as a pre-deploy step with backward-compatible migration discipline (expand/contract).

## Backup Strategy

- Automated daily full DB snapshots + continuous WAL archiving for point-in-time
  recovery (target: restore to any point within the last 7-35 days depending on plan).
- Cross-region backup replication.
- Quarterly restore drills validated against a staging environment.
- Object storage (receipts, images) versioned with lifecycle rules; accidental
  deletes recoverable for 30 days.
- Tenant-level "export my data" self-service tool as a business-continuity feature,
  independent of infra backups.

## Logging & Monitoring

- Structured JSON logs (request id, tenant id, user id) shipped to Loki/ELK.
- Metrics (request latency, error rate, queue depth, DB pool saturation, sync lag) in
  Prometheus, dashboards in Grafana.
- Error tracking (stack traces, release tagging) in Sentry across API, Web, and
  Flutter clients.
- Uptime/synthetic checks on critical endpoints (login, checkout, payment callback).
- Alerting (PagerDuty/Slack) on error-rate spikes, failed payment callbacks, and
  subscription-job failures.

## API Documentation Plan

- OpenAPI 3.1 spec generated from NestJS decorators (via `@nestjs/swagger`), published
  at `/api/docs` (internal) and a curated public subset for the integrations/API
  program.
- Versioned spec (`/api/v1`), changelog maintained per release.
- Postman/Insomnia collection auto-generated from the OpenAPI spec for partner
  integrations.
- Webhook payload reference documented alongside the public API for third-party
  developers building on Alani POS.
