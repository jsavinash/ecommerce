# Complete Microservices Catalog — E-Commerce Platform

**Architecture:** Event-Driven Microservices · **Scale:** 100K RPS · **Availability:** 99.999%
**Tech:** Java 21 · Spring Boot 3.5 · Gradle Kotlin DSL · Kafka · Redis · PostgreSQL · Elasticsearch

---

## COMPLETE MICROSERVICE LIST (18 Services)

### CORE COMMERCE SERVICES (8)

| # | Microservice | Port | Database | Responsibility | Key Features |
|---|---|---|---|---|---|
| 1 | **API Gateway** | 8080 | — | Edge routing, authN, rate limiting, request/response transformation, WAF | F76, F2, F89, F88, F91 |
| 2 | **Auth Service** | 8081 | PostgreSQL + Redis | User authentication, JWT, MFA, RBAC, session management, GDPR | F43-F47, F50, F51, F69 |
| 3 | **Catalog Service** | 8082 | PostgreSQL + Elasticsearch + Redis | Product catalog, search, facets, reviews, Q&A, CMS, SEO, i18n | F8-F10, F31, F32, F37-F40, F54, F66, F74, F75 |
| 4 | **Cart Service** | 8083 | Redis Cluster | Distributed cart, wishlist, cart merge, atomic Lua operations | F6, F7 |
| 5 | **Order Service** | 8084 | PostgreSQL | Checkout saga orchestrator, order lifecycle, OMS, refunds, returns, subscriptions, promotions, wallet, loyalty, gift cards, tax, shipping | F4, F5, F11-F18, F20-F24, F28, F29, F30, F64 |
| 6 | **Inventory Service** | 8085 | PostgreSQL + Redis | Stock reservation, ledger, multi-warehouse allocation, oversell prevention | F3, F65 |
| 7 | **Payment Service** | 8086 | PostgreSQL | Payment state machine, multi-PSP routing, webhooks, refunds, COD | F25, F26, F27 |
| 8 | **Notification Service** | 8087 | Kafka + PostgreSQL | Email/SMS/push notifications, templates, delivery tracking | F49 |

### SCALE & PERFORMANCE SERVICES (4)

| # | Microservice | Port | Database | Responsibility | Key Features |
|---|---|---|---|---|---|
| 9 | **Flash Sale Queue (FLQ) Service** | 8088 | Redis Cluster | Traffic smoothing, capacity check, FIFO queue, drainer, backpressure | F1, F19 |
| 10 | **Rate Limit Service** | 8090 | Redis Cluster | Atomic token bucket rate limiting per tenant/user/IP/endpoint | F2, F89 |
| 11 | **Recommendation Service** | 8089 | Redis + ClickHouse | Personalized recommendations, trending, frequently-bought-together, homepage | F33-F36, F41 |
| 12 | **Feature Flag Service** | 8091 | Redis + PostgreSQL | Feature flags, percentage rollouts, A/B testing assignment, kill switches | F42, F78, F79 |

### SELLER & MARKETPLACE SERVICES (2)

| # | Microservice | Port | Database | Responsibility | Key Features |
|---|---|---|---|---|---|
| 13 | **Seller Service** | 8092 | PostgreSQL | Seller onboarding, KYC, dashboard, catalog management, inventory sync | F55-F58 |
| 14 | **Payout Service** | 8093 | PostgreSQL | Seller payouts, settlement, commission calculation, payout schedules | F59-F61 |

### ADMIN & OPERATIONS SERVICES (2)

| # | Microservice | Port | Database | Responsibility | Key Features |
|---|---|---|---|---|---|
| 15 | **Admin Service** | 8094 | PostgreSQL | Admin dashboard, tenant management, audit log, reporting, data export | F63, F67-F73 |
| 16 | **Support Service** | 8095 | PostgreSQL + Redis | Customer support tickets, live chat, chatbot, reviews moderation | F52, F53, F30 |

### PLATFORM & INFRASTRUCTURE SERVICES (2)

| # | Microservice | Port | Database | Responsibility | Key Features |
|---|---|---|---|---|---|
| 17 | **Analytics Service** | 8096 | ClickHouse | Real-time analytics, event ingestion, dashboards, BI reporting | F72, F73, F32 |
| 18 | **Webhook Service** | 8097 | PostgreSQL | Outbound webhooks to tenants, retry, delivery tracking | F90, F92 |

---

## SERVICE DEPENDENCY GRAPH

```
                    ┌─────────────────┐
                    │   API Gateway   │
                    │      (8080)     │
                    └────────┬────────┘
                             │
          ┌──────────┬───────┼────────┬──────────┬──────────┐
          ▼          ▼       ▼        ▼          ▼          ▼
    ┌─────────┐ ┌─────────┐ ┌──────┐ ┌──────┐ ┌────────┐ ┌────────┐
    │  Auth   │ │ Catalog │ │ Cart │ │ FLQ  │ │  Rate  │ │  Rec   │
    │ (8081)  │ │ (8082)  │ │(8083)│ │(8088)│ │ Limit  │ │ (8089) │
    └────┬────┘ └────┬────┘ └──┬───┘ └──┬───┘ │ (8090) │ └───┬────┘
         │           │        │        │     └────────┘     │
         │           │        │        │                    │
         ▼           ▼        ▼        ▼                    ▼
    ┌──────────────────────────────────────────────────────────┐
    │                     Order Service                        │
    │                    (8084 — Saga Orchestrator)            │
    └──┬──────────┬──────────┬──────────┬──────────┬───────────┘
       ▼          ▼          ▼          ▼          ▼
  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
  │Inventory│ │ Payment │ │  Seller │ │ Payout  │ │  Admin  │
  │ (8085)  │ │ (8086)  │ │ (8092)  │ │ (8093)  │ │ (8094)  │
  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘

  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ Notification │  │  Analytics   │  │   Webhook    │
  │    (8087)    │  │    (8096)    │  │    (8097)    │
  └──────────────┘  └──────────────┘  └──────────────┘
        ▲                  ▲                  ▲
        └────── Kafka Events (async) ────────┘
```

---

## COMMUNICATION PATTERNS

### Synchronous (gRPC/REST — low latency, strong consistency)

| Caller | → | Callee | Purpose | Timeout |
|---|---|---|---|---|
| API Gateway | → | Auth | Token validation | 500ms |
| API Gateway | → | Rate Limit | Token bucket check | 100ms |
| Order | → | Cart | Lock cart, read snapshot | 500ms |
| Order | → | Catalog | Price re-validation | 500ms |
| Order | → | FLQ | Enqueue checkout | 100ms |
| Order | → | Inventory | Reserve stock | 2s |
| Order | → | Payment | Initiate payment | 2s |
| Catalog | → | Elasticsearch | Search query | 100ms |
| Recommendation | → | ClickHouse | Co-occurrence query | 500ms |

### Asynchronous (Kafka — eventual consistency, decoupled)

| Producer | → | Topic | Consumers |
|---|---|---|---|
| Order | → | `order.events` | Notification, Analytics, Recommendation |
| Inventory | → | `inventory.events` | Order (compensation), Catalog (indexer) |
| Payment | → | `payment.events` | Order, Notification, Analytics |
| Cart | → | `cart.events` | Analytics |
| FLQ | → | `flq.drain` | Order |
| All | → | `analytics.events` | Analytics, Recommendation |
| Order/Payment | → | `notification.events` | Notification |
| Order | → | `saga.compensations.retry` | Order (DLQ) |

---

## DATABASE OWNERSHIP

| Database | Owner(s) | Purpose |
|---|---|---|
| `ecommerce_auth` | Auth Service | Users, refresh tokens, sessions |
| `ecommerce_catalog` | Catalog Service | Products, categories, reviews |
| `ecommerce_orders` | Order Service | Orders, order items, refunds, returns, subscriptions, promotions, wallet, loyalty, gift cards |
| `ecommerce_inventory` | Inventory Service | Inventory ledger, inventory available |
| `ecommerce_payment` | Payment Service | Payment transactions, refunds |
| `ecommerce_notification` | Notification Service | Templates, delivery records |
| `ecommerce_seller` | Seller Service | Sellers, KYC, seller products |
| `ecommerce_payout` | Payout Service | Settlements, payouts, commission |
| `ecommerce_admin` | Admin Service | Tenants, audit logs, RBAC |
| `ecommerce_support` | Support Service | Tickets, chat history |
| `ecommerce_webhook` | Webhook Service | Webhook configs, delivery logs |
| `ecommerce_featureflag` | Feature Flag Service | Flags, experiments, variants |
| **Redis Cluster** | Cart, FLQ, Rate Limit, Auth, Catalog, Inventory, Feature Flag | Carts, queues, rate limits, sessions, hot keys, locks, flags |
| **Elasticsearch** | Catalog Service | Product search index |
| **ClickHouse** | Analytics, Recommendation | Event analytics, co-occurrence |

---

## DEPLOYMENT TOPOLOGY

```
┌─────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                      │
│                                                             │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │
│  │ Gateway │ │  Auth   │ │ Catalog │ │  Cart   │  ...      │
│  │  ×40    │ │  ×20    │ │  ×60    │ │  ×30    │           │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘           │
│                                                             │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │
│  │  Order  │ │Inventory│ │ Payment │ │   FLQ   │  ...      │
│  │  ×50    │ │  ×40    │ │  ×30    │ │  ×20    │           │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘           │
│                                                             │
│  ┌──────────────────────┐  ┌──────────────────────┐         │
│  │   PostgreSQL × 12    │  │   Redis Cluster × 6 │         │
│  │  (PgBouncer pools)   │  │  (carts, locks, FLQ)│         │
│  └──────────────────────┘  └──────────────────────┘         │
│                                                             │
│  ┌──────────────────────┐  ┌──────────────────────┐         │
│  │   Kafka × 3 brokers  │  │ Elasticsearch × 3    │         │
│  │   (RF=3)            │  │  (6 shards, 2 repl)  │         │
│  └──────────────────────┘  └──────────────────────┘         │
│                                                             │
│  ┌──────────────────────┐  ┌──────────────────────┐         │
│  │   ClickHouse × 3    │  │  Observability       │         │
│  │  (analytics)        │  │  (Prom/Grafana/Tempo)│         │
│  └──────────────────────┘  └──────────────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

---

## PRIORITY-BASED DEPLOYMENT ORDER

### Phase 0 — Foundation (required for 100K RPS)
1. API Gateway (8080)
2. Auth Service (8081)
3. Rate Limit Service (8090)
4. FLQ Service (8088)
5. Inventory Service (8085)

### Phase 1 — Core Commerce
6. Catalog Service (8082)
7. Cart Service (8083)
8. Order Service (8084)
9. Payment Service (8086)
10. Notification Service (8087)

### Phase 2 — Intelligence & Growth
11. Recommendation Service (8089)
12. Feature Flag Service (8091)
13. Analytics Service (8096)

### Phase 3 — Seller & Admin
14. Seller Service (8092)
15. Payout Service (8093)
16. Admin Service (8094)
17. Support Service (8095)
18. Webhook Service (8097)

---

## SUMMARY

| Category | Services | Count |
|---|---|---|
| Core Commerce | API Gateway, Auth, Catalog, Cart, Order, Inventory, Payment, Notification | 8 |
| Scale & Performance | FLQ, Rate Limit, Recommendation, Feature Flag | 4 |
| Seller & Marketplace | Seller, Payout | 2 |
| Admin & Operations | Admin, Support | 2 |
| Platform & Infrastructure | Analytics, Webhook | 2 |
| **Total** | | **18** |