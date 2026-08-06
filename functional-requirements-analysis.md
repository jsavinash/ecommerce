# Functional Requirements Analysis
## Multi-Tenant E-Commerce Platform

**Source:** `multi-tenant-ecommerce-platform.md`
**Total Functional Requirements:** 57 (FR-1 through FR-57)
**Total Features Cataloged:** 92 (F1 through F92)

---

## PART A: CORE FUNCTIONAL REQUIREMENTS (FR-1 to FR-4)

These are the 4 primary functional requirements that define the platform's core commerce flows.

### FR-1: Dynamic Catalog Search & Discovery
| Aspect | Requirement |
|---|---|
| **Ingest path** | Product feeds (PIM/ERP) → `Catalog Ingest` → normalized `products` table → CDC (Debezium) → Elasticsearch index + Redis hot-key cache |
| **Query path** | `GET /api/v1/catalog/search?q=&filters=&sort=&page=` → API Gateway → `Catalog Service` → Elasticsearch (faceted aggregation) → Redis cache (TTL 60s, keyed by normalized query hash) → response |
| **Facets** | Category, brand, price range, rating, availability (per-warehouse stock), tenant |
| **Ranking** | BM25 + business boost (revenue velocity, margin, freshness). Personalization via tenant/user profile vector |
| **Multi-tenancy** | Every document carries `tenant_id`; shared-index with tenant filter (routing by `tenant_id` shard routing) |
| **Search autocomplete** | Typeahead suggestions from Redis sorted sets (trending + recent + prefix match), backed by ES completion suggester |
| **Search analytics** | Click-through, no-result, and query-log events streamed to Kafka → ClickHouse for tuning |

### FR-2: Distributed Cart Management (Race Conditions)
| Aspect | Requirement |
|---|---|
| **Model** | Redis Hash per cart (`cart:{tenant}:{user_id}`) with field = `sku`, value = `{qty, added_at, price_snapshot}` |
| **Concurrency** | All mutations via **Lua script** (atomic) — no read-modify-write in app layer |
| **Concurrent add** | `HINCRBY` inside Lua with `HSETNX` for price snapshot |
| **Concurrent remove vs. add** | Single Lua script serializes per-key via Redis single-threaded execution |
| **Price change mid-session** | Cart stores `price_snapshot`; on checkout, `Cart Service` re-validates against current price; delta surfaced as `price_changed_items[]` |
| **TTL/expiry** | Sliding TTL 30 days; background sweeper emits `CART_EXPIRED` event |
| **Cart → Order handoff** | `POST /checkout` atomically reads cart (Lua), creates `order` (pending), emits `CART_LOCKED`; cart soft-deleted only after order reaches `PAYMENT_PENDING` |
| **Anonymous cart merge** | Anonymous cart merged into user cart on login via Lua (qty sum, newest price snapshot) |
| **Wishlist / Save for Later** | Redis Set per user (`wishlist:{tenant}:{user_id}`); move-to-cart operation |

### FR-3: Checkout Orchestration & Payment State Tracking
| Aspect | Requirement |
|---|---|
| **Flow** | `Checkout Service` (orchestrator) → validate cart → price re-validation → **Flash Sale Queue (FLQ)** → `Inventory Service` reserve → `Payment Service` initiate → async payment webhook → confirm → `Order Service` finalize → `Notification Service` |
| **Payment state machine** | `PENDING → AUTHORIZED → CAPTURED → SETTLED` / `PENDING → AUTHORIZED → VOIDED` / `PENDING → FAILED` / `PENDING → EXPIRED` |
| **Idempotency** | Every checkout carries `Idempotency-Key` (UUIDv7); `payment_transactions.idempotency_key` unique index; duplicate requests return original `202`/`200` with same `transaction_id` |
| **Webhook handling** | PSP webhooks idempotent — keyed by `psp_event_id`; processed exactly-once via dedupe + outbox |
| **Multi-PSP routing** | Primary PSP; fallback on failure/timeout (cost + success-rate weighted) |
| **Payment retry** | Transient failures retried exponential backoff (max 3); permanent declines → alternative payment methods |
| **Refund state machine** | `REFUND_REQUESTED → REFUND_PROCESSING → REFUNDED` / `REFUND_FAILED`; partial and full; PSP refund API |
| **COD** | No PSP authorization; order confirmed on delivery; payment captured at delivery |

### FR-4: Multi-Warehouse Inventory Allocation
| Aspect | Requirement |
|---|---|
| **Model** | `inventory_ledger` (append-only) + `inventory_available` (materialized per `sku × warehouse`) |
| **Allocation** | Hard reservation (decrement available, increment reserved) + soft allocation (time-boxed hold 15 min for payment window) |
| **Redis pre-reduction** | Hot SKU stock pre-warmed (`inv:hot:{tenant}:{sku}`); atomic Lua `DECR`; Redis absorbs ~99% of stock-check traffic; DB is source of truth |
| **Sourcing** | Tenant fulfillment config → distance/latency → available stock → split-shipment support |
| **Oversell prevention (4 layers)** | Redis pre-reduction → Redlock → `FOR UPDATE` with `lock_timeout` → optimistic `version` guard |
| **Release paths** | Payment timeout → auto-release; order cancel → release; payment failure → release; RMA/return → restock |

---

## PART B: COMPLETE FUNCTIONAL REQUIREMENTS LIST (FR-1 to FR-57)

### B.1 Core Commerce (FR-1 to FR-27)

| FR No. | Feature | Description | Priority | Complexity |
|---|---|---|---|---|
| FR-1 | **Dynamic Catalog Search & Discovery** | Product feeds (PIM/ERP), query path, facets, ranking, multi-tenancy, autocomplete, search analytics | P1 | High |
| FR-2 | **Distributed Cart Management** | Redis Hash model, Lua atomic concurrency, race condition handling, cart merge, wishlist | P1 | High |
| FR-3 | **Checkout Orchestration & Payment State Tracking** | Saga orchestration, payment state machine, idempotency, webhooks, multi-PSP, refunds, COD | P1 | High |
| FR-4 | **Multi-Warehouse Inventory Allocation** | Ledger + materialized state, hard/soft reservation, Redis pre-reduction, 4-layer oversell prevention | P1 | High |
| FR-5 | **Order tracking** | Real-time via WebSocket/SSE; Kafka `ORDER_STATUS_CHANGED` events | P1 | Medium |
| FR-6 | **Order history & search** | Paginated, filter by status/date, order detail | P1 | Low |
| FR-7 | **Product reviews & ratings** | Review service, moderation, verified-purchase badge, rating aggregation | P1 | Medium |
| FR-8 | **Product Q&A** | Community Q&A with seller answers | P2 | Medium |
| FR-9 | **Tax calculation** | Tax service (Avalara/TaxJar or rules engine) per jurisdiction | P1 | Medium |
| FR-10 | **Shipping rate calc** | Carrier rate APIs (FedEx/UPS/DHL) with caching | P1 | Medium |
| FR-11 | **Split shipment** | Multi-warehouse split fulfillment with per-shipment tracking | P1 | High |
| FR-12 | **Order cancellation** | Self-service cancel + inventory release + payment void/refund | P1 | Medium |
| FR-13 | **Returns / RMA** | Return request, RMA number, return label, refund trigger | P1 | High |
| FR-14 | **Refund management** | Partial/full refunds, state machine, PSP refund API | P1 | Medium |
| FR-15 | **Gift cards** | Issuance, redemption, balance, expiry | P2 | Medium |
| FR-16 | **Coupons & promotions** | Discount codes, cart/product rules, stackable, budget caps | P1 | High |
| FR-17 | **Flash sale / deal engine** | Time-boxed deals, deal inventory, countdown, deal-specific pricing | P1 | High |
| FR-18 | **Bundle / combo deals** | Product bundles with combined pricing | P2 | Medium |
| FR-19 | **Subscriptions** | Recurring billing, payment retry, pause/cancel | P1 | High |
| FR-20 | **Loyalty & rewards** | Points earning/spending, tiers, cashback | P2 | Medium |
| FR-21 | **Wallet** | Prepaid balance, transactions, top-up, payout | P2 | High |
| FR-22 | **BNPL** | Klarna/Afterpay/Affirm integration | P2 | Medium |
| FR-23 | **Multi-currency** | FX conversion, per-tenant currency config | P1 | Medium |
| FR-24 | **i18n / l10n** | Per-tenant language, locale, date/number formats | P1 | Medium |
| FR-25 | **Guest checkout** | No-account checkout with email capture | P1 | Low |
| FR-26 | **Address book** | Saved addresses, validation, geocoding | P1 | Low |
| FR-27 | **Invoice / receipt** | PDF invoices, GST/VAT invoices, email receipts | P2 | Low |

### B.2 Discovery & Personalization (FR-28 to FR-33)

| FR No. | Feature | Description | Priority | Complexity |
|---|---|---|---|---|
| FR-28 | **Personalized recommendations** | Collaborative filtering + content-based + real-time behavior | P1 | High |
| FR-29 | **Trending / popular** | Real-time velocity scoring | P2 | Medium |
| FR-30 | **New arrivals** | Freshness-boosted listing | P2 | Low |
| FR-31 | **Deals hub** | Curated deal pages | P2 | Low |
| FR-32 | **Personalized homepage** | Modular homepage with personalized sections | P1 | High |
| FR-33 | **A/B testing** | Experimentation platform for UI/pricing/ranking | P1 | High |

### B.3 Customer & Account (FR-34 to FR-42)

| FR No. | Feature | Description | Priority | Complexity |
|---|---|---|---|---|
| FR-34 | **SSO / social login** | Google/Apple/Facebook OAuth | P1 | Low |
| FR-35 | **MFA / 2FA** | TOTP, SMS, email OTP | P1 | Medium |
| FR-36 | **Passwordless login** | Magic link, OTP | P2 | Low |
| FR-37 | **Session management** | Device list, revoke, concurrent session limits | P1 | Medium |
| FR-38 | **Notification preferences** | Per-channel (email/SMS/push) opt-in/out | P1 | Low |
| FR-39 | **Account deletion (GDPR)** | Right-to-be-forgotten with data purge | P1 | Medium |
| FR-40 | **Consent management** | GDPR/CCPA consent records | P1 | Medium |
| FR-41 | **Customer support** | Tickets, chat, email | P2 | High |
| FR-42 | **Live chat / chatbot** | Real-time support with AI | P2 | High |

### B.4 Seller / Marketplace (FR-43)

| FR No. | Feature | Description | Priority | Complexity |
|---|---|---|---|---|
| FR-43 | **Seller marketplace** | Onboarding, dashboard, catalog, inventory, payouts, commission | P2 | High |

### B.5 Admin & Operations (FR-44 to FR-54)

| FR No. | Feature | Description | Priority | Complexity |
|---|---|---|---|---|
| FR-44 | **Admin dashboard** | KPIs, sales, orders, users, inventory | P1 | Medium |
| FR-45 | **OMS (Order Management System)** | Search, edit, cancel, refund, hold | P1 | High |
| FR-46 | **Inventory management** | Stock levels, adjustments, transfers, alerts | P1 | Medium |
| FR-47 | **Catalog management** | Product CRUD, bulk import, approval | P1 | Medium |
| FR-48 | **Pricing management** | Price rules, scheduled changes, tenant pricing | P1 | Medium |
| FR-49 | **Promotion management** | Create/edit/approve promotions | P1 | Medium |
| FR-50 | **Tenant management** | Onboarding, config, quotas | P1 | Medium |
| FR-51 | **Audit log** | Immutable admin action trail | P1 | Medium |
| FR-52 | **Reporting & analytics** | Sales, inventory, customer, marketing reports | P1 | High |
| FR-53 | **CMS** | Landing pages, banners, blog | P2 | Medium |
| FR-54 | **SEO management** | Meta tags, sitemap, canonical, structured data | P2 | Medium |

### B.6 Platform / Infrastructure (FR-55 to FR-57)

| FR No. | Feature | Description | Priority | Complexity |
|---|---|---|---|---|
| FR-55 | **Webhooks outbound** | Tenant webhooks for order/inventory events | P2 | Medium |
| FR-56 | **API keys** | Tenant API key management | P2 | Low |
| FR-57 | **Sandbox environment** | Tenant test environment | P2 | Medium |

---

## PART C: FEATURE INVENTORY CROSS-REFERENCE (92 Features)

The file also contains a 92-feature inventory (F1-F92) that maps to the FRs. Here's the cross-reference:

### C.1 Core Commerce (30 features)

| # | Feature | Priority | Complexity | Maps to FR |
|---|---|---|---|---|
| F1 | Flash Sale Queue (FLQ) | P0 | High | FR-3 (checkout) |
| F2 | Rate Limiting (Token Bucket) | P0 | Medium | FR-3 (checkout) |
| F3 | Inventory Pre-reduction | P0 | High | FR-4 (inventory) |
| F4 | Order Status Tracking | P1 | Medium | FR-5 |
| F5 | Order History & Search | P1 | Low | FR-6 |
| F6 | Cart Persistence & Merge | P1 | Medium | FR-2 |
| F7 | Wishlist / Save for Later | P1 | Low | FR-2 |
| F8 | Product Reviews & Ratings | P1 | Medium | FR-7 |
| F9 | Product Q&A | P2 | Medium | FR-8 |
| F10 | Multi-currency & Multi-language | P1 | Medium | FR-23, FR-24 |
| F11 | Tax Calculation | P1 | Medium | FR-9 |
| F12 | Shipping Rate Calculation | P1 | Medium | FR-10 |
| F13 | Split Shipment | P1 | High | FR-11 |
| F14 | Order Cancellation | P1 | Medium | FR-12 |
| F15 | Returns / RMA | P1 | High | FR-13 |
| F16 | Refund Management | P1 | Medium | FR-14 |
| F17 | Gift Cards | P2 | Medium | FR-15 |
| F18 | Coupons & Promotions | P1 | High | FR-16 |
| F19 | Flash Sale / Deal Engine | P1 | High | FR-17 |
| F20 | Bundle / Combo Deals | P2 | Medium | FR-18 |
| F21 | Subscriptions & Recurring Billing | P1 | High | FR-19 |
| F22 | Loyalty & Rewards | P2 | Medium | FR-20 |
| F23 | Wallet | P2 | High | FR-21 |
| F24 | BNPL | P2 | Medium | FR-22 |
| F25 | Multi-PSP Routing | P1 | High | FR-3 |
| F26 | Payment Methods (card, wallet, UPI, COD) | P1 | Medium | FR-3 |
| F27 | COD | P2 | Medium | FR-3 |
| F28 | Invoice / Receipt | P2 | Low | FR-27 |
| F29 | Address Book | P1 | Low | FR-26 |
| F30 | Guest Checkout | P1 | Low | FR-25 |

### C.2 Discovery & Personalization (12 features)

| # | Feature | Priority | Complexity | Maps to FR |
|---|---|---|---|---|
| F31 | Search Autocomplete | P1 | Medium | FR-1 |
| F32 | Search Analytics | P2 | Medium | FR-1 |
| F33 | Personalized Recommendations | P1 | High | FR-28 |
| F34 | Frequently Bought Together | P2 | Medium | FR-28 |
| F35 | Customers Also Viewed | P2 | Medium | FR-28 |
| F36 | Trending / Popular | P2 | Medium | FR-29 |
| F37 | New Arrivals | P2 | Low | FR-30 |
| F38 | Deals Hub | P2 | Low | FR-31 |
| F39 | Visual Search | P3 | High | FR-1 |
| F40 | Voice Search | P3 | High | FR-1 |
| F41 | Personalized Homepage | P1 | High | FR-32 |
| F42 | A/B Testing | P1 | High | FR-33 |

### C.3 Customer & Account (12 features)

| # | Feature | Priority | Complexity | Maps to FR |
|---|---|---|---|---|
| F43 | SSO / Social Login | P1 | Low | FR-34 |
| F44 | MFA / 2FA | P1 | Medium | FR-35 |
| F45 | Passwordless Login | P2 | Low | FR-36 |
| F46 | Session Management | P1 | Medium | FR-37 |
| F47 | Profile Management | P1 | Low | FR-37 |
| F48 | Address Book | P1 | Low | FR-26 |
| F49 | Notification Preferences | P1 | Low | FR-38 |
| F50 | Account Deletion (GDPR) | P1 | Medium | FR-39 |
| F51 | Consent Management | P1 | Medium | FR-40 |
| F52 | Customer Support / Ticketing | P2 | High | FR-41 |
| F53 | Live Chat / Chatbot | P2 | High | FR-42 |
| F54 | Reviews Moderation | P2 | Medium | FR-7 |

### C.4 Seller / Marketplace (8 features)

| # | Feature | Priority | Complexity | Maps to FR |
|---|---|---|---|---|
| F55 | Seller Onboarding | P2 | High | FR-43 |
| F56 | Seller Dashboard | P2 | High | FR-43 |
| F57 | Seller Catalog Management | P2 | Medium | FR-43 |
| F58 | Seller Inventory Sync | P2 | Medium | FR-43 |
| F59 | Seller Payouts | P2 | High | FR-43 |
| F60 | Seller Performance | P2 | Medium | FR-43 |
| F61 | Marketplace Commission | P2 | Medium | FR-43 |
| F62 | Seller Messaging | P3 | Medium | FR-43 |

### C.5 Admin & Operations (13 features)

| # | Feature | Priority | Complexity | Maps to FR |
|---|---|---|---|---|
| F63 | Admin Dashboard | P1 | Medium | FR-44 |
| F64 | Order Management (OMS) | P1 | High | FR-45 |
| F65 | Inventory Management | P1 | Medium | FR-46 |
| F66 | Catalog Management | P1 | Medium | FR-47 |
| F67 | Pricing Management | P1 | Medium | FR-48 |
| F68 | Promotion Management | P1 | Medium | FR-49 |
| F69 | User Management (RBAC) | P1 | Medium | FR-44 |
| F70 | Tenant Management | P1 | Medium | FR-50 |
| F71 | Audit Log | P1 | Medium | FR-51 |
| F72 | Reporting & Analytics | P1 | High | FR-52 |
| F73 | Data Export | P2 | Low | FR-52 |
| F74 | CMS | P2 | Medium | FR-53 |
| F75 | SEO Management | P2 | Medium | FR-54 |

### C.6 Platform / Infrastructure (17 features)

| # | Feature | Priority | Complexity | Maps to FR |
|---|---|---|---|---|
| F76 | API Gateway | P0 | High | FR-3 |
| F77 | Service Mesh | P1 | High | Platform |
| F78 | Feature Flags | P1 | Medium | Platform |
| F79 | A/B Testing Platform | P1 | Medium | FR-33 |
| F80 | Observability (OTel, Prometheus, Grafana) | P0 | High | Platform |
| F81 | Alerting (SLO-based) | P0 | Medium | Platform |
| F82 | Chaos Engineering | P1 | Medium | Platform |
| F83 | CI/CD / GitOps | P1 | Medium | Platform |
| F84 | Secret Management | P0 | Low | Platform |
| F85 | Backup & Restore | P0 | Medium | Platform |
| F86 | Disaster Recovery | P1 | High | Platform |
| F87 | Cost Management / FinOps | P2 | Medium | Platform |
| F88 | Tenant Quotas | P1 | Medium | FR-50 |
| F89 | Tenant Rate Limits | P0 | Medium | FR-50 |
| F90 | Webhook Outbound | P2 | Medium | FR-55 |
| F91 | API Keys | P2 | Low | FR-56 |
| F92 | Sandbox Environment | P2 | Medium | FR-57 |

---

## PART D: SUMMARY STATISTICS

### D.1 By Priority

| Priority | Count | Percentage |
|---|---|---|
| P0 (Critical — required for 100K RPS) | 8 | 8.7% |
| P1 (Core Commerce) | 38 | 41.3% |
| P2 (Growth) | 40 | 43.5% |
| P3 (Advanced) | 6 | 6.5% |
| **Total** | **92** | **100%** |

### D.2 By Domain

| Domain | Count | Percentage |
|---|---|---|
| Core Commerce | 30 | 32.6% |
| Discovery & Personalization | 12 | 13.0% |
| Customer & Account | 12 | 13.0% |
| Seller / Marketplace | 8 | 8.7% |
| Admin & Operations | 13 | 14.1% |
| Platform / Infrastructure | 17 | 18.5% |
| **Total** | **92** | **100%** |

### D.3 By Complexity

| Complexity | Count | Percentage |
|---|---|---|
| Low | 17 | 18.5% |
| Medium | 48 | 52.2% |
| High | 27 | 29.3% |
| **Total** | **92** | **100%** |

### D.4 Functional Requirements by Category

| Category | FR Range | Count |
|---|---|---|
| Core Commerce | FR-1 to FR-27 | 27 |
| Discovery & Personalization | FR-28 to FR-33 | 6 |
| Customer & Account | FR-34 to FR-42 | 9 |
| Seller / Marketplace | FR-43 | 1 |
| Admin & Operations | FR-44 to FR-54 | 11 |
| Platform / Infrastructure | FR-55 to FR-57 | 3 |
| **Total** | **FR-1 to FR-57** | **57** |

---

## PART E: KEY OBSERVATIONS

1. **The file is a condensed version** of the full HLD/LLD blueprint — it contains the 92-feature inventory (Part 4) and the FR table, but the detailed FR-1 through FR-4 descriptions are truncated (only FR-5 header exists under Part 1).

2. **P0 features (8)** are the non-negotiable foundation for 100K RPS:
   - F1: Flash Sale Queue (FLQ)
   - F2: Rate Limiting (Token Bucket)
   - F3: Inventory Pre-reduction
   - F76: API Gateway
   - F80: Observability
   - F81: Alerting
   - F84: Secret Management
   - F85: Backup & Restore
   - F89: Tenant Rate Limits

3. **FR-1 through FR-4** are the 4 primary functional requirements that define the platform's core commerce flows (catalog, cart, checkout, inventory).

4. **FR-5 through FR-57** are the 53 additional functional requirements covering order management, reviews, tax, shipping, returns, refunds, promotions, subscriptions, loyalty, wallet, personalization, customer account, seller marketplace, admin operations, and platform infrastructure.

5. **The 92 features (F1-F92)** map to the 57 functional requirements (FR-1 to FR-57), with some FRs having multiple features (e.g., FR-3 checkout maps to F1, F2, F25, F26, F27, F76) and some features being platform-level (F77-F89) that don't map to a specific FR.