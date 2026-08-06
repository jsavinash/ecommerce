# High-Level & Low-Level Design: Hyperscale Multi-Tenant E-Commerce Platform

**Target Scale:** 100,000 RPS peak (flash sales) · **Availability:** 99.999% · **Multi-tenant:** shared-infrastructure, tenant-isolated data plane
**Version:** 2.0 (integrated with deep research analysis: Shopify BFCM, Shopee C10M, Tencent seckill, 100K RPS best practices)

---

## PART 1: SYSTEM REQUIREMENTS & BOUNDARIES

### 1.1 Functional Requirements (FRs)

#### FR-5: Additional Functional Features (P1-P3)

| Feature | Description | Priority |
|---|---|---|
| **Order tracking** | Real-time via WebSocket/SSE; Kafka `ORDER_STATUS_CHANGED` | P1 |
| **Order history & search** | Paginated, filter by status/date | P1 |
| **Product reviews & ratings** | Moderation, verified-purchase badge | P1 |
| **Product Q&A** | Community Q&A with seller answers | P2 |
| **Tax calculation** | Tax service (Avalara/TaxJar or rules engine) | P1 |
| **Shipping rate calc** | Carrier rate APIs (FedEx/UPS/DHL) with caching | P1 |
| **Split shipment** | Multi-warehouse split fulfillment | P1 |
| **Order cancellation** | Self-service cancel + inventory release + payment void/refund | P1 |
| **Returns / RMA** | Return request, RMA number, return label, refund trigger | P1 |
| **Refund management** | Partial/full refunds, state machine, PSP refund API | P1 |
| **Gift cards** | Issuance, redemption, balance, expiry | P2 |
| **Coupons & promotions** | Discount codes, cart/product rules, stackable, budget caps | P1 |
| **Flash sale / deal engine** | Time-boxed deals, deal inventory, countdown | P1 |
| **Bundle / combo deals** | Product bundles with combined pricing | P2 |
| **Subscriptions** | Recurring billing, payment retry, pause/cancel | P1 |
| **Loyalty & rewards** | Points earning/spending, tiers, cashback | P2 |
| **Wallet** | Prepaid balance, transactions, top-up, payout | P2 |
| **BNPL** | Klarna/Afterpay/Affirm integration | P2 |
| **Multi-currency** | FX conversion, per-tenant currency config | P1 |
| **i18n / l10n** | Per-tenant language, locale, formats | P1 |
| **Guest checkout** | No-account checkout with email capture | P1 |
| **Address book** | Saved addresses, validation, geocoding | P1 |
| **Invoice / receipt** | PDF invoices, GST/VAT, email receipts | P2 |
| **Personalized recommendations** | Collaborative + content-based + real-time behavior | P1 |
| **Trending / popular** | Real-time velocity scoring | P2 |
| **New arrivals** | Freshness-boosted listing | P2 |
| **Deals hub** | Curated deal pages | P2 |
| **Personalized homepage** | Modular personalized sections | P1 |
| **A/B testing** | Experimentation platform | P1 |
| **SSO / social login** | Google/Apple/Facebook OAuth | P1 |
| **MFA / 2FA** | TOTP, SMS, email OTP | P1 |
| **Passwordless login** | Magic link, OTP | P2 |
| **Session management** | Device list, revoke, concurrent limits | P1 |
| **Notification preferences** | Per-channel opt-in/out | P1 |
| **Account deletion (GDPR)** | Right-to-be-forgotten with purge | P1 |
| **Consent management** | GDPR/CCPA consent records | P1 |
| **Customer support** | Tickets, chat, email | P2 |
| **Live chat / chatbot** | Real-time support with AI | P2 |
| **Seller marketplace** | Onboarding, dashboard, catalog, inventory, payouts | P2 |
| **Admin dashboard** | KPIs, sales, orders, users, inventory | P1 |
| **OMS** | Search, edit, cancel, refund, hold | P1 |
| **Inventory management** | Stock levels, adjustments, transfers, alerts | P1 |
| **Catalog management** | Product CRUD, bulk import, approval | P1 |
| **Pricing management** | Price rules, scheduled changes, tenant pricing | P1 |
| **Promotion management** | Create/edit/approve promotions | P1 |
| **Tenant management** | Onboarding, config, quotas | P1 |
| **Audit log** | Immutable admin action trail | P1 |
| **Reporting & analytics** | Sales, inventory, customer, marketing | P1 |
| **CMS** | Landing pages, banners, blog | P2 |
| **SEO management** | Meta tags, sitemap, canonical, structured data | P2 |
| **Webhooks outbound** | Tenant webhooks for order/inventory events | P2 |
| **API keys** | Tenant API key management | P2 |
| **Sandbox environment** | Tenant test environment | P2 |

