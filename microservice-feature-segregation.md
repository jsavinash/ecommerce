# Feature Segregation by Microservice
## Multi-Tenant E-Commerce Platform — Service Ownership Map

**Source:** `multi-tenant-ecommerce-platform.md` (92 features · 57 FRs)
**Architecture:** 11 Microservices + API Gateway

---

## SERVICE OWNERSHIP MAP — ALL 92 FEATURES

### 1. API GATEWAY (Edge — Port 8080)

| Feature ID | Feature | FR | Priority |
|---|---|---|---|
| F76 | API Gateway (routing, auth, rate limit, response caching) | FR-3 | P0 |
| F2 | Rate Limiting (Token Bucket) — per-tenant/user/IP/endpoint | FR-3 | P0 |
| F89 | Tenant Rate Limits (per-tenant RPS, burst, concurrency) | FR-50 | P0 |
| F88 | Tenant Quotas (API, storage, compute) | FR-50 | P1 |
| F91 | API Keys (tenant API key management) | FR-56 | P2 |

**Total: 5 features**

---

### 2. AUTH SERVICE (Port 8081)

| Feature ID | Feature | FR | Priority |
|---|---|---|---|
| F43 | SSO / Social Login (Google, Apple, Facebook OAuth) | FR-34 | P1 |
| F44 | MFA / 2FA (TOTP, SMS, email OTP) | FR-35 | P1 |
| F45 | Passwordless Login (magic link, OTP) | FR-36 | P2 |
| F46 | Session Management (device list, revoke, concurrent limits) | FR-37 | P1 |
| F47 | Profile Management (name, avatar, preferences, opt-in) | FR-37 | P1 |
| F50 | Account Deletion (GDPR right-to-be-forgotten) | FR-39 | P1 |
| F51 | Consent Management (GDPR/CCPA consent records) | FR-40 | P1 |
| F69 | User Management / RBAC (admin roles, permissions) | FR-44 | P1 |

**Total: 8 features**

---

### 3. CATALOG SERVICE (Port 8082)

| Feature ID | Feature | FR | Priority |
|---|---|---|---|
| F31 | Search Autocomplete (typeahead, trending, recent) | FR-1 | P1 |
| F32 | Search Analytics (click-through, no-result, query logs) | FR-1 | P2 |
| F39 | Visual Search (image-based, embeddings + vector search) | FR-1 | P3 |
| F40 | Voice Search (ASR + NLU → search query) | FR-1 | P3 |
| F66 | Catalog Management (product CRUD, bulk import, approval) | FR-47 | P1 |
| F75 | SEO Management (meta tags, sitemap, canonical, structured data) | FR-54 | P2 |
| F74 | CMS (landing pages, banners, blog) | FR-53 | P2 |
| F8 | Product Reviews & Ratings (moderation, verified-purchase badge) | FR-7 | P1 |
| F54 | Reviews Moderation (spam/abuse detection) | FR-7 | P2 |
| F9 | Product Q&A (community Q&A with seller answers) | FR-8 | P2 |
| F37 | New Arrivals (freshness-boosted listing) | FR-30 | P2 |
| F38 | Deals Hub (curated deal pages) | FR-31 | P2 |
| F10 | Multi-currency (FX conversion, per-tenant config) | FR-23 | P1 |
| F10 | i18n / l10n (per-tenant language, locale, formats) | FR-24 | P1 |

**Total: 14 features**

---

### 4. CART SERVICE (Port 8083)

| Feature ID | Feature | FR | Priority |
|---|---|---|---|
| F6 | Cart Persistence & Merge (anonymous → user cart) | FR-2 | P1 |
| F7 | Wishlist / Save for Later (Redis Set, move-to-cart) | FR-2 | P1 |

**Total: 2 features**

---

### 5. ORDER SERVICE (Port 8084)

| Feature ID | Feature | FR | Priority |
|---|---|---|---|
| F4 | Order Status Tracking (WebSocket/SSE, Kafka events) | FR-5 | P1 |
| F5 | Order History & Search (paginated, filter by status/date) | FR-6 | P1 |
| F12 | Order Cancellation (self-service, inventory release, payment void) | FR-12 | P1 |
| F13 | Split Shipment (multi-warehouse split fulfillment) | FR-11 | P1 |
| F14 | Order Management / OMS (search, edit, cancel, refund, hold) | FR-45 | P1 |
| F16 | Refund Management (partial/full, state machine, PSP API) | FR-14 | P1 |
| F17 | Flash Sale / Deal Engine (time-boxed deals, countdown) | FR-17 | P1 |
| F18 | Coupons & Promotions (discount codes, cart/product rules) | FR-16 | P1 |
| F20 | Subscriptions & Recurring Billing (payment retry, pause/cancel) | FR-19 | P1 |
| F25 | Invoice / Receipt Generation (PDF, GST/VAT) | FR-27 | P2 |
| F28 | Returns / RMA (return request, label, refund trigger) | FR-13 | P1 |
| F5x | Guest Checkout (no-account checkout with email capture) | FR-25 | P1 |
| F11 | Tax Calculation (Avalara/TaxJar or rules engine) | FR-9 | P1 |
| F29 | Address Book (saved addresses, validation, geocoding) | FR-26 | P1 |
| F24 | BNPL (Klarna/Afterpay/Affirm integration) | FR-22 | P2 |
| F21 | Loyalty & Rewards (points earning/spending, tiers) | FR-20 | P2 |
| F22 | Wallet (prepaid balance, transactions, top-up) | FR-21 | P2 |
| F23 | Gift Cards (issuance, redemption, balance, expiry) | FR-15 | P2 |

**Total: 18 features**

---

### 6. INVENTORY SERVICE (Port 8085)

| Feature ID | Feature | FR | Priority |
|---|---|---|---|
| F3 | Inventory Pre-reduction (Redis hot SKU, atomic Lua DECR) | FR-4 | P0 |
| F65 | Inventory Management (stock levels, adjustments, transfers, alerts) | FR-46 | P1 |

**Total: 2 features**

---

### 7. PAYMENT SERVICE (Port 8086)

| Feature ID | Feature | FR | Priority |
|---|---|---|---|
| F25 | Multi-PSP Routing (primary + fallback, cost/success weighted) | FR-3 | P1 |
| F26 | Payment Methods (card, wallet, UPI, bank transfer) | FR-3 | P1 |
| F27 | COD (Cash on Delivery — no PSP authorization) | FR-3 | P2 |

**Total: 3 features**

---

### 8. NOTIFICATION SERVICE (Port 8087)

| Feature ID | Feature | FR | Priority |
|---|---|---|---|
| F49 | Notification Preferences (per-channel email/SMS/push opt-in) | FR-38 | P1 |

**Total: 1 feature**

---

### 9. FLQ SERVICE — Flash Sale Queue (Port 8088)

| Feature ID | Feature | FR | Priority |
|---|---|---|---|
| F1 | Flash Sale Queue (atomic Lua enqueue, capacity check, FIFO, drainer) | FR-3 | P0 |
| F19 | Deal Engine integration (deal inventory, countdown, deal pricing) | FR-17 | P1 |

**Total: 2 features**

---

### 10. RECOMMENDATION SERVICE (Port 8089)

| Feature ID | Feature | FR | Priority |
|---|---|---|---|
| F33 | Personalized Recommendations (collaborative + content-based) | FR-28 | P1 |
| F34 | Frequently Bought Together (co-occurrence mining) | FR-28 | P2 |
| F35 | Customers Also Viewed (session-based similarity) | FR-28 | P2 |
| F36 | Trending / Popular (real-time velocity scoring) | FR-29 | P2 |
| F41 | Personalized Homepage (modular personalized sections) | FR-32 | P1 |

**Total: 5 features**

---

### 11. RATE LIMIT SERVICE (Port 8090)

| Feature ID | Feature | FR | Priority |
|---|---|---|---|
| F2 | Rate Limiting (atomic token bucket — Redis Lua) | FR-3 | P0 |
| F89 | Tenant Rate Limits (per-tenant RPS, burst, concurrency) | FR-50 | P0 |

**Total: 2 features**

---

### 12. FEATURE FLAG SERVICE (Port 8091)

| Feature ID | Feature | FR | Priority |
|---|---|---|---|
| F78 | Feature Flags (progressive delivery, kill switches, canary) | Platform | P1 |
| F42 | A/B Testing (experimentation platform) | FR-33 | P1 |
| F79 | A/B Testing Platform (SDK, assignment, analysis) | FR-33 | P1 |

**Total: 3 features**

---

### 13. PLATFORM / INFRASTRUCTURE (Cross-Cutting)

| Feature ID | Feature | FR | Priority |
|---|---|---|---|
| F77 | Service Mesh (mTLS, retries, timeouts, circuit breaking) | Platform | P1 |
| F80 | Observability (OTel, Prometheus, Grafana) | Platform | P0 |
| F81 | Alerting (SLO-based alerts, on-call, paging) | Platform | P0 |
| F82 | Chaos Engineering (fault injection, game days) | Platform | P1 |
| F83 | CI/CD / GitOps (ArgoCD, GitHub Actions, canary) | Platform | P1 |
| F84 | Secret Management (Vault, KMS, rotation) | Platform | P0 |
| F85 | Backup & Restore (PITR, cross-region, drills) | Platform | P0 |
| F86 | Disaster Recovery (multi-region failover, RPO/RTO drills) | Platform | P1 |
| F87 | Cost Management / FinOps (cost allocation, autoscaling) | Platform | P2 |
| F90 | Webhook Outbound (tenant webhooks for order/inventory events) | FR-55 | P2 |
| F92 | Sandbox Environment (tenant test environment) | FR-57 | P2 |
| F68 | Promotion Management (create/edit/approve promotions) | FR-49 | P1 |
| F67 | Pricing Management (price rules, scheduled changes) | FR-48 | P1 |
| F70 | Tenant Management (onboarding, config, quotas) | FR-50 | P1 |
| F71 | Audit Log (immutable admin action trail) | FR-51 | P1 |
| F72 | Reporting & Analytics (sales, inventory, customer, marketing) | FR-52 | P1 |
| F73 | Data Export (CSV/Excel export) | FR-52 | P2 |
| F63 | Admin Dashboard (KPIs, sales, orders, users, inventory) | FR-44 | P1 |
| F55 | Seller Onboarding (KYC, bank account, store setup) | FR-43 | P2 |
| F56 | Seller Dashboard (sales, inventory, orders, analytics) | FR-43 | P2 |
| F57 | Seller Catalog Management (product CRUD, bulk upload) | FR-43 | P2 |
| F58 | Seller Inventory Sync (API/CSV/ERP sync) | FR-43 | P2 |
| F59 | Seller Payouts (settlement, commission, payout schedule) | FR-43 | P2 |
| F60 | Seller Performance (rating, defect rate, SLA) | FR-43 | P2 |
| F61 | Marketplace Commission (commission rules, fee calc) | FR-43 | P2 |
| F62 | Seller Messaging (buyer-seller chat) | FR-43 | P3 |
| F30 | Customer Support / Ticketing (tickets, chat, email) | FR-41 | P2 |
| F53 | Live Chat / Chatbot (real-time support with AI) | FR-42 | P2 |

**Total: 28 features (cross-cutting / admin / seller)**

---

## CONSOLIDATED SERVICE OWNERSHIP TALLY

| # | Microservice | Port | Feature Count | P0 | P1 | P2 | P3 |
|---|---|---|---|---|---|---|---|
| 1 | API Gateway | 8080 | 5 | 3 | 1 | 1 | 0 |
| 2 | Auth Service | 8081 | 8 | 0 | 7 | 1 | 0 |
| 3 | Catalog Service | 8082 | 14 | 0 | 8 | 4 | 2 |
| 4 | Cart Service | 8083 | 2 | 0 | 2 | 0 | 0 |
| 5 | Order Service | 8084 | 18 | 0 | 12 | 6 | 0 |
| 6 | Inventory Service | 8085 | 2 | 1 | 1 | 0 | 0 |
| 7 | Payment Service | 8086 | 3 | 0 | 2 | 1 | 0 |
| 8 | Notification Service | 8087 | 1 | 0 | 1 | 0 | 0 |
| 9 | FLQ Service | 8088 | 2 | 1 | 1 | 0 | 0 |
| 10 | Recommendation Service | 8089 | 5 | 0 | 2 | 3 | 0 |
| 11 | Rate Limit Service | 8090 | 2 | 2 | 0 | 0 | 0 |
| 12 | Feature Flag Service | 8091 | 3 | 0 | 3 | 0 | 0 |
| 13 | Cross-Cutting / Admin / Seller | — | 28 | 4 | 13 | 10 | 1 |
| **TOTAL** | | | **92** | **8** | **38** | **40** | **6** |

---

## FUNCTIONAL REQUIREMENTS BY SERVICE

### Auth Service (FR-34, 35, 36, 37, 39, 40, 44)
| FR | Requirement |
|---|---|
| FR-34 | SSO / Social Login |
| FR-35 | MFA / 2FA |
| FR-36 | Passwordless Login |
| FR-37 | Session Management & Profile |
| FR-39 | Account Deletion (GDPR) |
| FR-40 | Consent Management |
| FR-44 | Admin Dashboard (partially — RBAC) |

### Catalog Service (FR-1, 7, 8, 23, 24, 30, 31, 47, 53, 54)
| FR | Requirement |
|---|---|
| FR-1 | Dynamic Catalog Search & Discovery (search, autocomplete, analytics, visual, voice) |
| FR-7 | Product Reviews & Ratings |
| FR-8 | Product Q&A |
| FR-23 | Multi-currency |
| FR-24 | i18n / l10n |
| FR-30 | New Arrivals |
| FR-31 | Deals Hub |
| FR-47 | Catalog Management |
| FR-53 | CMS |
| FR-54 | SEO Management |

### Cart Service (FR-2)
| FR | Requirement |
|---|---|
| FR-2 | Distributed Cart Management (persistence, merge, wishlist) |

### Order Service (FR-5, 6, 9, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 25, 26, 27, 45)
| FR | Requirement |
|---|---|
| FR-5 | Order Tracking |
| FR-6 | Order History & Search |
| FR-9 | Tax Calculation |
| FR-11 | Split Shipment |
| FR-12 | Order Cancellation |
| FR-13 | Returns / RMA |
| FR-14 | Refund Management |
| FR-15 | Gift Cards |
| FR-16 | Coupons & Promotions |
| FR-17 | Flash Sale / Deal Engine |
| FR-18 | Bundle / Combo Deals |
| FR-19 | Subscriptions |
| FR-20 | Loyalty & Rewards |
| FR-21 | Wallet |
| FR-22 | BNPL |
| FR-25 | Guest Checkout |
| FR-26 | Address Book |
| FR-27 | Invoice / Receipt |
| FR-45 | OMS |

### Inventory Service (FR-4, 46)
| FR | Requirement |
|---|---|
| FR-4 | Multi-Warehouse Inventory Allocation |
| FR-46 | Inventory Management |

### Payment Service (FR-3 partially)
| FR | Requirement |
|---|---|
| FR-3 | Checkout Orchestration & Payment State Tracking (payment portion: multi-PSP, methods, COD) |

### FLQ Service (FR-3 partially, FR-17)
| FR | Requirement |
|---|---|
| FR-3 | Checkout Orchestration (queueing portion: FLQ) |
| FR-17 | Flash Sale / Deal Engine (queue integration) |

### Recommendation Service (FR-28, 29, 32)
| FR | Requirement |
|---|---|
| FR-28 | Personalized Recommendations (incl. frequent-bought-together, also-viewed) |
| FR-29 | Trending / Popular |
| FR-32 | Personalized Homepage |

### Notification Service (FR-38, 41, 42)
| FR | Requirement |
|---|---|
| FR-38 | Notification Preferences |
| FR-41 | Customer Support (notification portion) |
| FR-42 | Live Chat / Chatbot (push notifications) |

### Feature Flag Service (FR-33)
| FR | Requirement |
|---|---|
| FR-33 | A/B Testing |

### API Gateway + Rate Limit (FR-3, 50, 56)
| FR | Requirement |
|---|---|
| FR-3 | Checkout Orchestration (rate limiting portion) |
| FR-50 | Tenant Management (quotas, rate limits) |
| FR-56 | API Keys |

---

## SUMMARY

The 92 features and 57 FRs have been segregated across **13 logical service groups**:

| Category | Services | Feature Count | % of Total |
|---|---|---|---|
| Core Services (API Gateway, Auth, Catalog, Cart, Order, Inventory, Payment, Notification) | 8 | 53 | 57.6% |
| Flash-Sale & Scale Services (FLQ, Rate Limit) | 2 | 4 | 4.3% |
| Intelligence Services (Recommendation, Feature Flag) | 2 | 8 | 8.7% |
| Cross-Cutting / Admin / Seller / Platform | — | 28 | 30.4% |