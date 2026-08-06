# Complete Microservices Catalog — E-Commerce Platform (Expanded)
## Deep Research Analysis — All Possible Services

**Base:** `microservices-catalog.md` (18 services)
**Research:** 21-Service Go Blueprint (tanhdev), Composable Commerce Architecture (vesviet), industry best practices
**Total Services:** **40 microservices** + 2 frontend applications

---

## COMPLETE MICROSERVICE LIST (40 Services)

### DOMAIN 1: EDGE & API (2)

| # | Microservice | Port | Database | Responsibility | New? |
|---|---|---|---|---|---|
| 1 | **API Gateway** | 8080 | — | Edge routing, authN, rate limiting, WAF, BFF, request/response transformation | Existing |
| 2 | **GraphQL Federation / BFF** | 8080-gql | — | GraphQL schema federation, client-specific BFF (mobile/web/partner), request coalescing | **NEW** |

---

### DOMAIN 2: IDENTITY & ACCESS (4)

| # | Microservice | Port | Database | Responsibility | New? |
|---|---|---|---|---|---|
| 3 | **Auth Service** | 8081 | PostgreSQL + Redis | JWT, OAuth2, MFA, session management, passwordless | Existing |
| 4 | **User Service** | 8098 | PostgreSQL | Internal staff accounts, RBAC, admin roles, permissions | **NEW** |
| 5 | **Customer Profile Service** | 8099 | PostgreSQL + Redis | External customer data, LTV, segmentation, preferences, GDPR | **NEW** |
| 6 | **Consent Service** | 8100 | PostgreSQL | GDPR/CCPA consent records, data deletion requests, data export | **NEW** |

---

### DOMAIN 3: PRODUCT & CONTENT (6)

| # | Microservice | Port | Database | Responsibility | New? |
|---|---|---|---|---|---|
| 7 | **Catalog Service** | 8082 | PostgreSQL + ES + Redis | Product catalog, PIM, categories, attributes, reviews, Q&A | Existing |
| 8 | **Pricing Service** | 8101 | PostgreSQL + Redis | Multi-currency pricing, tax-aware pricing, dynamic pricing rules, scheduled price changes | **NEW** |
| 9 | **Promotion Service** | 8102 | PostgreSQL + Redis | Coupons, BOGO, cart rules, product rules, budget caps, stackable promotions | **NEW** |
| 10 | **Search Service** | 8103 | Elasticsearch | CQRS pattern, sub-50ms search, facets, autocomplete, relevance tuning | **NEW** |
| 11 | **Search Worker** | 8104 | — | Kafka consumer, rebuilds ES documents on product/pricing/stock events | **NEW** |
| 12 | **CMS Service** | 8105 | PostgreSQL + Redis | Landing pages, banners, blog, content blocks, A/B content variants | **NEW** |

---

### DOMAIN 4: COMMERCE FLOW (5)

| # | Microservice | Port | Database | Responsibility | New? |
|---|---|---|---|---|---|
| 13 | **Cart Service** | 8083 | Redis Cluster | Distributed cart, wishlist, cart merge, atomic Lua | Existing |
| 14 | **Checkout Orchestrator** | 8084 | PostgreSQL | Saga pattern, coordinates cart → FLQ → inventory → payment | Existing (Order) |
| 15 | **Order Lifecycle Service** | 8106 | PostgreSQL | 8-state FSM for order processing, order history, OMS | **NEW** (split from Order) |
| 16 | **Payment Service** | 8086 | PostgreSQL | Multi-PSP (Stripe, Adyen, PayPal, VNPay, MoMo), authorization, capture, refunds, webhooks | Existing |
| 17 | **Fraud Detection Service** | 8107 | Redis + ClickHouse | Risk scoring, velocity checks, AVS/CVV, device fingerprinting, rule engine | **NEW** |

---

### DOMAIN 5: LOGISTICS & FULFILLMENT (5)

| # | Microservice | Port | Database | Responsibility | New? |
|---|---|---|---|---|---|
| 18 | **Inventory / Warehouse Service** | 8085 | PostgreSQL + Redis | Stock tracking, reservations, restock, OCC, idempotency, multi-warehouse | Existing |
| 19 | **Fulfillment Service** | 8108 | PostgreSQL | Orchestrates pick, pack, hand-off; shipment creation; tracking updates | **NEW** |
| 20 | **Shipping Hub Service** | 8109 | PostgreSQL + Redis | Normalizes multi-carrier communication (FedEx, UPS, DHL), rate calc, label generation, tracking | **NEW** |
| 21 | **Location Service** | 8110 | PostgreSQL | Geographical locations, zones, warehouses, delivery areas, geocoding | **NEW** |
| 22 | **Returns Service** | 8111 | PostgreSQL | RMA, returns, exchanges, automated refunds, restock, return labels | **NEW** |

---

### DOMAIN 6: POST-PURCHASE & ENGAGEMENT (5)

| # | Microservice | Port | Database | Responsibility | New? |
|---|---|---|---|---|---|
| 23 | **Loyalty & Rewards Service** | 8112 | PostgreSQL + Redis | Points ledger, tiers, referrals, cashback, outbox pattern | **NEW** |
| 24 | **Wallet Service** | 8113 | PostgreSQL | Prepaid balance, transactions, top-up, payout, ledger | **NEW** |
| 25 | **Subscription Service** | 8114 | PostgreSQL | Recurring billing, plans, payment retry, pause/cancel, dunning | **NEW** |
| 26 | **Gift Card Service** | 8115 | PostgreSQL | Issuance, redemption, balance, expiry, bulk purchase | **NEW** |
| 27 | **Review Service** | 8116 | PostgreSQL + Redis | Product reviews, ratings, moderation, verified-purchase badge, Q&A | **NEW** |

---

### DOMAIN 7: SELLER & MARKETPLACE (4)

| # | Microservice | Port | Database | Responsibility | New? |
|---|---|---|---|---|---|
| 28 | **Seller Service** | 8092 | PostgreSQL | Onboarding, KYC, dashboard, catalog management, inventory sync | Existing |
| 29 | **Payout Service** | 8093 | PostgreSQL | Settlements, commission, payout schedules, tax forms | Existing |
| 30 | **Marketplace Commission Service** | 8117 | PostgreSQL | Commission rules, fee calculation, tiered rates, category-based fees | **NEW** |
| 31 | **Seller Messaging Service** | 8118 | PostgreSQL + Redis | Buyer-seller chat, dispute resolution, notifications | **NEW** |

---

### DOMAIN 8: CUSTOMER EXPERIENCE (3)

| # | Microservice | Port | Database | Responsibility | New? |
|---|---|---|---|---|---|
| 32 | **Recommendation Service** | 8089 | Redis + ClickHouse | Personalized recs, trending, frequently-bought-together, homepage | Existing |
| 33 | **Support / Ticketing Service** | 8095 | PostgreSQL + Redis | Support tickets, live chat, chatbot, knowledge base | Existing |
| 34 | **Notification Hub** | 8087 | Kafka + PostgreSQL | Email/SMS/push, templates, delivery tracking, retry, dedupe | Existing |

---

### DOMAIN 9: PLATFORM & OPERATIONS (6)

| # | Microservice | Port | Database | Responsibility | New? |
|---|---|---|---|---|---|
| 35 | **FLQ Service** | 8088 | Redis Cluster | Flash sale queue, traffic smoothing, capacity check | Existing |
| 36 | **Rate Limit Service** | 8090 | Redis Cluster | Atomic token bucket per tenant/user/IP/endpoint | Existing |
| 37 | **Feature Flag Service** | 8091 | Redis + PostgreSQL | Feature flags, A/B testing, kill switches | Existing |
| 38 | **Analytics Service** | 8096 | ClickHouse | Real-time analytics, event ingestion, dashboards, BI | Existing |
| 39 | **Webhook Service** | 8097 | PostgreSQL | Outbound webhooks, retry, delivery tracking | Existing |
| 40 | **Admin Service** | 8094 | PostgreSQL | Admin dashboard, tenant management, audit log, reporting | Existing |

---

### FRONTEND APPLICATIONS (2)

| # | Application | Tech | Responsibility |
|---|---|---|---|
| 41 | **Customer Website** | Next.js | Customer-facing frontend: browse, search, purchase, track orders |
| 42 | **Admin Dashboard** | React | Admin-facing frontend: manage products, orders, users, analytics |

---

## SERVICE COUNT COMPARISON

| Source | Count |
|---|---|
| Original `microservices-catalog.md` | 18 |
| 21-Service Go Blueprint (tanhdev) | 21 |
| Composable Commerce Architecture (vesviet) | 23 |
| **Expanded Catalog (this document)** | **40 + 2 frontends** |

---

## NEW SERVICES ADDED (22)

| # | New Service | Why It's Needed |
|---|---|---|
| 1 | **GraphQL Federation / BFF** | Reduces N+1 round trips; client-specific payloads |
| 2 | **User Service** | Separates internal staff/RBAC from external customer auth |
| 3 | **Customer Profile Service** | LTV, segmentation, preferences — separate from auth |
| 4 | **Consent Service** | GDPR/CCPA compliance — dedicated service for consent records |
| 5 | **Pricing Service** | Multi-currency, tax-aware, dynamic pricing — separate from catalog |
| 6 | **Promotion Service** | Coupons, BOGO, budget caps — complex rules engine |
| 7 | **Search Service** | CQRS pattern — dedicated search with sub-50ms SLA |
| 8 | **Search Worker** | Kafka consumer rebuilding ES documents |
| 9 | **CMS Service** | Content management — landing pages, banners, blog |
| 10 | **Order Lifecycle Service** | 8-state FSM — split from monolithic Order Service |
| 11 | **Fraud Detection Service** | Risk scoring, velocity checks, device fingerprinting |
| 12 | **Fulfillment Service** | Pick, pack, hand-off orchestration |
| 13 | **Shipping Hub Service** | Multi-carrier normalization, labels, tracking |
| 14 | **Location Service** | Geocoding, zones, warehouse locations |
| 15 | **Returns Service** | RMA, exchanges, automated refunds, restock |
| 16 | **Loyalty & Rewards Service** | Points ledger, tiers, referrals |
| 17 | **Wallet Service** | Prepaid balance, ledger, top-up |
| 18 | **Subscription Service** | Recurring billing, dunning |
| 19 | **Gift Card Service** | Issuance, redemption, balance |
| 20 | **Review Service** | Reviews, ratings, moderation |
| 21 | **Marketplace Commission Service** | Commission rules, tiered rates |
| 22 | **Seller Messaging Service** | Buyer-seller chat, disputes |

---

## UPDATED SERVICE DEPENDENCY GRAPH

```
                    ┌─────────────────────────┐
                    │   API Gateway + BFF     │
                    │      (8080)             │
                    └────────────┬────────────┘
                                 │
     ┌──────────┬──────────┬─────┼─────┬──────────┬──────────┐
     ▼          ▼          ▼     ▼     ▼          ▼          ▼
┌─────────┐ ┌─────────┐ ┌──────┐ ┌──────┐ ┌────────┐ ┌────────┐
│  Auth   │ │  User   │ │Customer│ │Cart  │ │  FLQ   │ │  Rate  │
│ (8081)  │ │ (8098)  │ │(8099) │ │(8083)│ │ (8088) │ │ Limit  │
└────┬────┘ └────┬────┘ └──┬───┘ └──┬───┘ └───┬────┘ │ (8090) │
     │           │        │        │         │      └────────┘
     │           │        │        │         │
     ▼           ▼        ▼        ▼         ▼
┌──────────────────────────────────────────────────────────────┐
│              Checkout Orchestrator (8084)                    │
│         (Saga: cart → FLQ → inventory → payment)            │
└──┬──────────┬──────────┬──────────┬──────────┬───────────────┘
   ▼          ▼          ▼          ▼          ▼
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────────┐
│Inventory│ │Payment  │ │ Fraud   │ │Fulfill- │ │  Shipping   │
│ (8085)  │ │ (8086)  │ │ (8107)  │ │ment(8108)│ │  Hub(8109) │
└─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────────┘
     │           │                        │            │
     ▼           ▼                        ▼            ▼
┌─────────┐ ┌─────────┐            ┌─────────┐  ┌─────────────┐
│Returns  │ │Loyalty  │            │Location │  │  Order      │
│ (8111)  │ │ (8112)  │            │ (8110)  │  │ Lifecycle   │
└─────────┘ └─────────┘            └─────────┘  │   (8106)    │
                                                └─────────────┘
     ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
     │ Notification │  │  Analytics   │  │   Webhook    │
     │    (8087)    │  │    (8096)    │  │    (8097)    │
     └──────────────┘  └──────────────┘  └──────────────┘
           ▲                  ▲                  ▲
           └────── Kafka Events (async) ────────┘

     ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
     │   Catalog    │  │   Pricing    │  │  Promotion   │
     │    (8082)    │  │    (8101)    │  │   (8102)     │
     └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
            │                 │                 │
            ▼                 ▼                 ▼
     ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
     │   Search     │  │  Search      │  │     CMS      │
     │   (8103)     │  │  Worker(8104)│  │    (8105)    │
     └──────────────┘  └──────────────┘  └──────────────┘
```

---

## DATABASE OWNERSHIP (UPDATED)

| Database | Owner(s) | Purpose |
|---|---|---|
| `ecommerce_auth` | Auth Service | Users, refresh tokens, sessions |
| `ecommerce_user` | User Service | Staff accounts, RBAC |
| `ecommerce_customer` | Customer Profile Service | Customer data, LTV, segmentation |
| `ecommerce_consent` | Consent Service | GDPR/CCPA consent records |
| `ecommerce_catalog` | Catalog Service | Products, categories, reviews |
| `ecommerce_pricing` | Pricing Service | Price rules, currency, tax |
| `ecommerce_promotion` | Promotion Service | Coupons, BOGO, budget caps |
| `ecommerce_cms` | CMS Service | Content, landing pages, banners |
| `ecommerce_orders` | Checkout + Order Lifecycle | Orders, order items, FSM state |
| `ecommerce_payment` | Payment Service | Payment transactions, refunds |
| `ecommerce_fraud` | Fraud Detection Service | Risk scores, rules, blacklists |
| `ecommerce_inventory` | Inventory Service | Ledger, available stock |
| `ecommerce_fulfillment` | Fulfillment Service | Pick/pack, shipments |
| `ecommerce_shipping` | Shipping Hub Service | Carrier configs, labels, tracking |
| `ecommerce_location` | Location Service | Zones, warehouses, geocoding |
| `ecommerce_returns` | Returns Service | RMA, exchanges, restock |
| `ecommerce_loyalty` | Loyalty Service | Points ledger, tiers |
| `ecommerce_wallet` | Wallet Service | Balance, transactions |
| `ecommerce_subscription` | Subscription Service | Plans, billing, dunning |
| `ecommerce_giftcard` | Gift Card Service | Cards, balance, redemption |
| `ecommerce_review` | Review Service | Reviews, ratings, moderation |
| `ecommerce_seller` | Seller Service | Sellers, KYC |
| `ecommerce_payout` | Payout Service | Settlements, payouts |
| `ecommerce_commission` | Commission Service | Commission rules, fees |
| `ecommerce_messaging` | Seller Messaging Service | Chat, disputes |
| `ecommerce_support` | Support Service | Tickets, chat history |
| `ecommerce_notification` | Notification Service | Templates, delivery |
| `ecommerce_admin` | Admin Service | Tenants, audit logs |
| `ecommerce_webhook` | Webhook Service | Webhook configs, delivery |
| `ecommerce_featureflag` | Feature Flag Service | Flags, experiments |
| **Redis Cluster** | Cart, FLQ, Rate Limit, Auth, Catalog, Inventory, Feature Flag, Fraud, Loyalty, Wallet | Carts, queues, rate limits, sessions, hot keys, locks, flags, risk scores |
| **Elasticsearch** | Search Service | Product search index |
| **ClickHouse** | Analytics, Recommendation | Event analytics, co-occurrence |

---

## KAFKA TOPICS (UPDATED)

| Topic | Producer | Consumers |
|---|---|---|
| `order.events` | Checkout, Order Lifecycle, Payment | Notification, Analytics, Recommendation, Search Worker |
| `inventory.events` | Inventory | Order (compensation), Search Worker, Fulfillment |
| `payment.events` | Payment | Order, Notification, Analytics, Fraud |
| `cart.events` | Cart | Analytics |
| `flq.drain` | FLQ | Checkout |
| `product.events` | Catalog, Pricing, Promotion | Search Worker, Recommendation |
| `fulfillment.events` | Fulfillment, Shipping Hub | Order, Notification, Analytics |
| `returns.events` | Returns | Inventory (restock), Payment (refund), Analytics |
| `loyalty.events` | Loyalty | Analytics, Notification |
| `wallet.events` | Wallet | Analytics, Notification |
| `subscription.events` | Subscription | Payment, Notification, Analytics |
| `fraud.events` | Fraud Detection | Order (block/allow), Analytics |
| `notification.events` | All | Notification Hub |
| `analytics.events` | All | Analytics, Recommendation |
| `saga.compensations.retry` | Checkout | Checkout (DLQ) |
| `*.dlq` | All | Ops (manual replay) |

---

## DEPLOYMENT ORDER (UPDATED — 6 Phases)

### Phase 0 — Foundation (5 services)
1. API Gateway (8080)
2. Auth Service (8081)
3. Rate Limit Service (8090)
4. FLQ Service (8088)
5. Inventory Service (8085)

### Phase 1 — Core Commerce (8 services)
6. Catalog Service (8082)
7. Cart Service (8083)
8. Checkout Orchestrator (8084)
9. Order Lifecycle Service (8106)
10. Payment Service (8086)
11. Notification Hub (8087)
12. Search Service (8103)
13. Search Worker (8104)

### Phase 2 — Product & Pricing (4 services)
14. Pricing Service (8101)
15. Promotion Service (8102)
16. CMS Service (8105)
17. Review Service (8116)

### Phase 3 — Logistics (4 services)
18. Fulfillment Service (8108)
19. Shipping Hub Service (8109)
20. Location Service (8110)
21. Returns Service (8111)

### Phase 4 — Engagement & Post-Purchase (5 services)
22. Loyalty & Rewards Service (8112)
23. Wallet Service (8113)
24. Subscription Service (8114)
25. Gift Card Service (8115)
26. Fraud Detection Service (8107)

### Phase 5 — Seller, Admin & Intelligence (6 services)
27. Seller Service (8092)
28. Payout Service (8093)
29. Marketplace Commission Service (8117)
30. Seller Messaging Service (8118)
31. Recommendation Service (8089)
32. Feature Flag Service (8091)
33. Analytics Service (8096)
34. Webhook Service (8097)
35. Admin Service (8094)
36. Support Service (8095)
37. User Service (8098)
38. Customer Profile Service (8099)
39. Consent Service (8100)
40. GraphQL Federation / BFF (8080-gql)

---

## SUMMARY

| Domain | Services | Count |
|---|---|---|
| Edge & API | API Gateway, GraphQL Federation/BFF | 2 |
| Identity & Access | Auth, User, Customer Profile, Consent | 4 |
| Product & Content | Catalog, Pricing, Promotion, Search, Search Worker, CMS | 6 |
| Commerce Flow | Cart, Checkout Orchestrator, Order Lifecycle, Payment, Fraud Detection | 5 |
| Logistics & Fulfillment | Inventory, Fulfillment, Shipping Hub, Location, Returns | 5 |
| Post-Purchase & Engagement | Loyalty, Wallet, Subscription, Gift Card, Review | 5 |
| Seller & Marketplace | Seller, Payout, Commission, Messaging | 4 |
| Customer Experience | Recommendation, Support, Notification Hub | 3 |
| Platform & Operations | FLQ, Rate Limit, Feature Flag, Analytics, Webhook, Admin | 6 |
| Frontend Applications | Customer Website, Admin Dashboard | 2 |
| **Total** | | **42** |
