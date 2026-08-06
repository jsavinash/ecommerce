# Deep Research Analysis: Hyperscale Multi-Tenant E-Commerce Platform
## Feature Inventory & Optimization Opportunities

**Source analyzed:** `ecommerce-hld-lld.md` (772 lines, HLD/LLD for 100K RPS multi-tenant e-commerce)
**Research basis:** Shopify BFCM engineering (inventory reservations, Kafka), Shopee flash-sale architecture (C10M), Tencent Cloud seckill (1M QPS), industry best practices for 100K RPS systems, multi-tenant SaaS platform patterns.

---

## PART A: GAP ANALYSIS OF EXISTING DOCUMENT

### A.1 What the Document Covers Well (Strengths)

| Area | Assessment |
|---|---|
| Core checkout saga (orchestration) | Strong — step/compensation table, saga state machine, idempotency |
| Inventory oversell prevention | Strong — 3-layer defense (Redlock + FOR UPDATE + version), production-grade Go |
| Database schemas | Good — 7 core tables with PKs/FKs/indexes, ledger pattern |
| API contracts | Good — OpenAPI for checkout + inventory reserve with 200/202/409/422 |
| Failure mitigation matrix | Good — 7 failure modes with detection/mitigation/recovery |
| NFRs | Good — explicit P95/P99 targets, availability, consistency boundaries |

### A.2 Critical Gaps (Missing from the Document)

| # | Gap | Severity | Why It Matters |
|---|---|---|---|
| G1 | **No flash-sale/queueing layer** | 🔴 Critical | At 100K RPS, direct DB writes will melt. Tencent/Shopee both use a queue (CKafka/FLQ) to smooth bursts. The doc has no request queue, no traffic shaping, no backpressure strategy. |
| G2 | **No rate limiting design** | 🔴 Critical | Shopee uses atomic Token Bucket in Redis at the gateway. The doc mentions "rate limiter" in topology but gives zero design. |
| G3 | **No caching strategy detail** | 🔴 Critical | The doc mentions Redis/ES but no multi-layer cache design (CDN → in-process → Redis), no TTL jitter, no cache stampede prevention, no hot-key handling. |
| G4 | **No observability/telemetry** | 🟠 High | No tracing (OpenTelemetry), metrics, logging, SLOs, dashboards, alerting. Impossible to operate at 99.999% without it. |
| G5 | **No feature flags / A/B testing / progressive delivery** | 🟠 High | No way to safely roll out changes at 100K RPS. |
| G6 | **No chaos engineering / resilience testing** | 🟠 High | Failure matrix exists but no mechanism to *prove* the mitigations work. |
| G7 | **No multi-region / DR design** | 🟠 High | NFR says "multi-region active-active for reads" but no topology, no failover procedure, no data replication strategy. |
| G8 | **No tenant isolation / quota / billing design** | 🟠 High | Multi-tenant is claimed but no per-tenant rate limits, quotas, billing metering, or data-plane isolation details. |
| G9 | **No search/ES index design** | 🟠 High | No index mapping, no shard strategy, no reindexing, no query DSL, no relevance tuning. |
| G10 | **No notification service design** | 🟡 Medium | Mentioned but no channels (email/SMS/push), templates, retry, dedupe, or delivery guarantees. |
| G11 | **No analytics/BI pipeline** | 🟡 Medium | No clickstream, no event warehouse, no real-time dashboards. |
| G12 | **No personalization/recommendations** | 🟡 Medium | No recommendation engine, no user profile, no personalization vector. |
| G13 | **No subscription/recurring billing** | 🟡 Medium | Common e-commerce feature missing. |
| G14 | **No loyalty/rewards/wallet** | 🟡 Medium | Wallet balance is mentioned in consistency model but no schema/service. |
| G15 | **No refunds/returns/RMA** | 🟡 Medium | Refunded status exists in orders but no flow, no RMA, no return shipping. |
| G16 | **No tax engine** | 🟡 Medium | Tax field exists but no tax calculation service. |
| G17 | **No shipping/fulfillment integration** | 🟡 Medium | Out of scope but no event contract for 3PL. |
| G18 | **No API versioning / backward compat** | 🟡 Medium | No versioning strategy, no deprecation policy. |
| G19 | **No idempotency key storage schema** | 🟡 Medium | Idempotency mentioned but no `idempotency_keys` table. |
| G20 | **No outbox table schema** | 🟡 Medium | Outbox pattern mentioned but no DDL. |
| G21 | **No schema for cart (Redis)** | 🟡 Medium | Cart described conceptually but no Redis key structure / Lua script. |
| G22 | **No capacity planning / cost model** | 🟡 Medium | No node counts, no storage estimates, no cost. |
| G23 | **No deployment / CI-CD / GitOps** | 🟡 Medium | No deployment strategy, no canary, no rollback. |
| G24 | **No API gateway contract** | 🟡 Medium | No gateway routing, authN/authZ, rate limit, request/response transformation. |
| G25 | **No GraphQL / BFF layer** | 🟢 Low | Mobile/web clients need a BFF to aggregate. |
| G26 | **No edge computing / CDN strategy** | 🟢 Low | No image optimization, no edge caching, no edge personalization. |
| G27 | **No accessibility/i18n/l10n** | 🟢 Low | Multi-tenant global platform needs i18n. |
| G28 | **No compliance (GDPR/CCPA)** | 🟢 Low | Data deletion, consent, audit. |

---

## PART B — COMPLETE FEATURE INVENTORY (ALL POSSIBLE FEATURES)

### B.1 Core Commerce Features (Must-Have)

| # | Feature | Description | Priority | Complexity |
|---|---|---|---|---|
| F1 | **Flash Sale Queue (FLQ)** | Pre-queue requests before inventory/order services. Redis-based queue with atomic Lua enqueue, capacity check, and async drain to order service. Returns "queued" status. | P0 | High |
| F2 | **Rate Limiting (Token Bucket)** | Atomic token bucket in Redis at API Gateway. Per-tenant, per-user, per-IP, per-endpoint. Burst allowance for flash sales. | P0 | Medium |
| F3 | **Inventory Pre-reduction** | Pre-warm hot SKU stock into Redis; atomic Lua `DECR` at request time; DB is source of truth but Redis absorbs 99% of traffic. | P0 | High |
| F4 | **Order Status Tracking** | Real-time order status via WebSocket/SSE + Kafka events. | P1 | Medium |
| F5 | **Order History & Search** | Paginated order list, filter by status/date, order detail. | P1 | Low |
| F6 | **Cart Persistence & Merge** | Anonymous cart → logged-in cart merge on login. | P1 | Medium |
| F7 | **Wishlist / Save for Later** | Redis-backed wishlist per user. | P1 | Low |
| F8 | **Product Reviews & Ratings** | Review service with moderation, rating aggregation, verified-purchase badge. | P1 | Medium |
| F9 | **Product Q&A** | Community Q&A with seller answers. | P2 | Medium |
| F10 | **Multi-currency & Multi-language** | Currency conversion (FX rates), i18n/l10n per tenant. | P1 | Medium |
| F11 | **Tax Calculation** | Tax service (Avalara/TaxJar integration or rules engine) per jurisdiction. | P1 | Medium |
| F12 | **Shipping Rate Calculation** | Carrier rate APIs (FedEx/UPS/DHL) with caching. | P1 | Medium |
| F13 | **Split Shipment** | Multi-warehouse split fulfillment with per-shipment tracking. | P1 | High |
| F14 | **Order Cancellation** | Self-service cancel with inventory release + payment void/refund. | P1 | Medium |
| F15 | **Returns / RMA** | Return request, RMA number, return label, refund trigger. | P1 | High |
| F16 | **Refund Management** | Partial/full refunds, refund state machine, PSP refund API. | P1 | Medium |
| F17 | **Gift Cards** | Issuance, redemption, balance, expiry. | P2 | Medium |
| F18 | **Coupons & Promotions** | Discount codes, cart rules, product rules, stackable, budget caps. | P1 | High |
| F19 | **Flash Sale / Deal Engine** | Time-boxed deals, deal inventory, countdown, deal-specific pricing. | P1 | High |
| F20 | **Bundle / Combo Deals** | Product bundles with combined pricing. | P2 | Medium |
| F21 | **Subscriptions & Recurring Billing** | Subscription plans, recurring orders, payment retry, pause/cancel. | P1 | High |
| F22 | **Loyalty & Rewards** | Points earning/spending, tiers, cashback. | P2 | Medium |
| F23 | **Wallet** | Prepaid wallet with balance, transactions, top-up, payout. | P2 | High |
| F24 | **Buy Now Pay Later (BNPL)** | Integration with Klarna/Afterpay/Affirm. | P2 | Medium |
| F25 | **Multi-PSP Routing** | Smart PSP routing by cost, success rate, region, fallback. | P1 | High |
| F26 | **Payment Methods** | Card, wallet, UPI, bank transfer, COD, crypto (optional). | P1 | Medium |
| F27 | **COD (Cash on Delivery)** | COD-specific order flow with delivery confirmation. | P2 | Medium |
| F28 | **Invoice / Receipt Generation** | PDF invoices, GST/VAT invoices, email receipts. | P2 | Low |
| F29 | **Address Book** | Saved addresses, validation, geocoding. | P1 | Low |
| F30 | **Guest Checkout** | No-account checkout with email capture. | P1 | Low |

### B.2 Discovery & Personalization

| # | Feature | Description | Priority | Complexity |
|---|---|---|---|---|
| F31 | **Search Autocomplete** | Typeahead with suggestions, trending, recent. | P1 | Medium |
| F32 | **Search Analytics** | Click-through, no-result, query logs for tuning. | P2 | Medium |
| F33 | **Personalized Recommendations** | Collaborative filtering + content-based + real-time behavior. | P1 | High |
| F34 | **"Frequently Bought Together"** | Co-occurrence mining from order data. | P2 | Medium |
| F35 | **"Customers Also Viewed"** | Session-based similarity. | P2 | Medium |
| F36 | **Trending / Popular Products** | Real-time velocity scoring. | P2 | Medium |
| F37 | **New Arrivals** | Freshness-boosted listing. | P2 | Low |
| F38 | **Deals / Discounts Hub** | Curated deal pages. | P2 | Low |
| F39 | **Visual Search** | Image-based search (embedding + vector search). | P3 | High |
| F40 | **Voice Search** | ASR + NLU → search query. | P3 | High |
| F41 | **Personalized Homepage** | Modular homepage with personalized sections. | P1 | High |
| F42 | **A/B Testing** | Experimentation platform for UI/pricing/ranking. | P1 | High |

### B.3 Customer & Account

| # | Feature | Description | Priority | Complexity |
|---|---|---|---|---|
| F43 | **SSO / Social Login** | Google/Apple/Facebook OAuth. | P1 | Low |
| F44 | **MFA / 2FA** | TOTP, SMS, email OTP. | P1 | Medium |
| F45 | **Passwordless Login** | Magic link, OTP. | P2 | Low |
| F46 | **Session Management** | Device list, revoke, concurrent session limits. | P1 | Medium |
| F47 | **Profile Management** | Name, avatar, preferences, communication opt-in. | P1 | Low |
| F48 | **Address Book** | Multiple addresses with default. | P1 | Low |
| F49 | **Notification Preferences** | Per-channel (email/SMS/push) opt-in/out. | P1 | Low |
| F50 | **Account Deletion (GDPR)** | Right-to-be-forgotten with data purge. | P1 | Medium |
| F51 | **Consent Management** | GDPR/CCPA consent records. | P1 | Medium |
| F52 | **Customer Support / Ticketing** | Support tickets, chat, email. | P2 | High |
| F53 | **Live Chat / Chatbot** | Real-time support with AI. | P2 | High |
| F54 | **Reviews Moderation** | Spam/abuse detection, seller response. | P2 | Medium |

### B.4 Seller / Marketplace (if multi-vendor)

| # | Feature | Description | Priority | Complexity |
|---|---|---|---|---|
| F55 | **Seller Onboarding** | KYC, bank account, store setup. | P2 | High |
| F56 | **Seller Dashboard** | Sales, inventory, orders, analytics. | P2 | High |
| F57 | **Seller Catalog Management** | Product CRUD, bulk upload, price. | P2 | Medium |
| F58 | **Seller Inventory Sync** | API/CSV/ERP sync. | P2 | Medium |
| F59 | **Seller Payouts** | Settlement, commission, payout schedule. | P2 | High |
| F60 | **Seller Performance** | Rating, defect rate, SLA. | P2 | Medium |
| F61 | **Marketplace Commission** | Commission rules, fee calculation. | P2 | Medium |
| F62 | **Seller Messaging** | Buyer-seller chat. | P3 | Medium |

### B.5 Admin & Operations

| # | Feature | Description | Priority | Complexity |
|---|---|---|---|---|
| F63 | **Admin Dashboard** | KPIs, sales, orders, users, inventory. | P1 | Medium |
| F64 | **Order Management (OMS)** | Search, edit, cancel, refund, hold. | P1 | High |
| F65 | **Inventory Management** | Stock levels, adjustments, transfers, alerts. | P1 | Medium |
| F66 | **Catalog Management** | Product CRUD, bulk import, approval. | P1 | Medium |
| F67 | **Pricing Management** | Price rules, scheduled changes, tenant pricing. | P1 | Medium |
| F68 | **Promotion Management** | Create/edit/approve promotions. | P1 | Medium |
| F69 | **User Management** | Admin RBAC, roles, permissions. | P1 | Medium |
| F70 | **Tenant Management** | Tenant onboarding, config, quotas. | P1 | Medium |
| F71 | **Audit Log** | Immutable audit trail of admin actions. | P1 | Medium |
| F72 | **Reporting & Analytics** | Sales, inventory, customer, marketing reports. | P1 | High |
| F73 | **Data Export** | CSV/Excel export. | P2 | Low |
| F74 | **Content Management (CMS)** | Landing pages, banners, blog. | P2 | Medium |
| F75 | **SEO Management** | Meta tags, sitemap, canonical, structured data. | P2 | Medium |

### B.6 Platform / Infrastructure Features

| # | Feature | Description | Priority | Complexity |
|---|---|---|---|---|
| F76 | **API Gateway** | Routing, auth, rate limit, response caching, request/response. | P0 | High |
| F77 | **Service Mesh** | mTLS, retries, timeouts, circuit breaking, observability. | P1 | High |
| F78 | **Feature Flags** | Progressive delivery, kill switches, canary. | P1 | Medium |
| F79 | **A/B Testing Platform** | Experimentation SDK, assignment, analysis. | P1 | Medium |
| F80 | **Observability** | Metrics (Prometheus), logs (Loki/ELK), traces (OTel/Jaeger), dashboards (Grafana). | P0 | High |
| F81 | **Alerting** | SLO-based alerts, on-call, paging. | P0 | Medium |
| F82 | **Chaos Engineering** | Fault injection, chaos experiments, game days. | P1 | Medium |
| F83 | **CI/CD / GitOps** | ArgoCD, GitHub Actions, blue/green, canary. | P1 | Medium |
| F84 | **Secret Management** | Vault, KMS, rotation. | P0 | Low |
| F85 | **Backup & Restore** | PITR, cross-region backups, restore drills. | P0 | Medium |
| F86 | **Disaster Recovery** | Multi-region failover, RTO/RPO drills. | P1 | High |
| F87 | **Cost Management** | FinOps, cost allocation, autoscaling. | P2 | Medium |
| F88 | **Tenant Quotas** | Per-tenant API, storage, compute quotas. | P1 | Medium |
| F89 | **Tenant Rate Limits** | Per-tenant RPS, burst, concurrency. | P0 | Medium |
| F90 | **Webhook Outbound** | Tenant webhooks for order/inventory events. | P2 | Medium |
| F91 | **API Keys** | Tenant API key management. | P2 | Low |
| F92 | **Sandbox Environment** | Tenant test environment. | P2 | Medium |

---

## PART C — OPTIMIZATION OPPORTUNITIES (BY LAYER)

### C.1 Edge / CDN Optimizations

| # | Optimization | Technique | Impact |
|---|---|---|---|
| O1 | **Edge caching of product pages** | Cache HTML/JSON at CDN edge with `Cache-Control: s-maxage`, `stale-while-revalidate`, `stale-if-error`. | Reduces origin load 80-90% |
| O2 | **Image optimization** | WebP/AVIF, responsive srcset, lazy loading, CDN image resizing (imgix/Cloudinary). | Reduces bandwidth 60-80% |
| O3 | **Edge personalization** | Edge workers (Cloudflare Workers/CloudFront Functions) inject personalized content without origin round-trip. | Reduces latency |
| O4 | **Edge rate limiting** | Token bucket at edge (per IP, per tenant). | Protects origin |
| O5 | **Edge WAF** | Managed WAF rules, bot detection, DDoS mitigation. | Protects origin |
| O6 | **HTTP/3 + QUIC** | Faster TLS handshake, multiplexing. | Reduces latency |
| O7 | **Brotli compression** | Better compression than gzip. | Reduces bandwidth |
| O8 | **Edge cache invalidation** | Purge API, cache tags, `stale-while-revalidate`. | Freshness |

### C.2 API Gateway / BFF Optimizations

| # | Technique | Detail | Impact |
|---|---|---|---|
| O9 | **GraphQL Federation** | Apollo Federation / GraphQL Mesh to compose microservices into one schema. | Reduces N+1 round trips |
| O10 | **BFF per client** | Mobile BFF, Web BFF, Partner BFF. | Tailored payloads |
| O11 | **Response compression** | gzip/brotli at gateway. | Reduces bandwidth |
| O12 | **Request coalescing** | Single-flight for identical concurrent requests. | Reduces backend load |
| O13 | **Connection pooling** | PgBouncer/ProxySQL for DB connections. | Prevents connection exhaustion (Shopify's #1 bottleneck) |
| O14 | **Backpressure** | Reject with 429/503 when queue full. | Protects downstream |
| O15 | **Circuit breakers** | Per-service circuit breaker (Resilience4j/Hystrix). | Fault isolation |
| O16 | **Bulkheads** | Per-tenant, per-service thread pools. | Tenant isolation |

### C.3 Caching Optimizations (Critical for 100K RPS)

| # | Optimization | Technique | Impact |
|---|---|---|---|
| O17 | **Multi-layer cache** | CDN → edge → in-process (Caffeine) → Redis → DB. | 90%+ cache hit |
| O18 | **TTL jitter** | `TTL = base ± random(0-30s)` to prevent thundering herd. | Prevents stampede |
| O19 | **Cache stampede prevention** | Request coalescing (single-flight), soft TTL + background refresh, distributed lock. | Prevents DB meltdown |
| O20 | **Hot key protection** | Replicate hot keys across multiple Redis slots; near cache; randomized response delay. | Prevents hot-spot |
| O21 | **Adaptive TTL** | TTL based on key popularity/update frequency. | Freshness + hit rate |
| O22 | **Predictive cache warming** | Pre-warm cache for flash-sale products before event. | Zero cold misses |
| O23 | **Cache-aside with write-through** | Write-through for hot inventory. | Consistency |
| O24 | **Redis Cluster with replicas** | Read replicas for cache reads. | Read scaling |
| O25 | **Local cache (Caffeine)** | In-process cache for hot keys with tiny TTL (1-5s). | Sub-ms reads |
| O26 | **Cache invalidation via Kafka** | Invalidate on `PRODUCT_UPDATED` event. | Freshness |

### C.4 Database Optimizations

| # | Optimization | Technique | Impact |
|---|---|---|---|
| O27 | **Read replicas** | Route stale-tolerant reads to replicas; follower reads with lag guard. | Scales reads |
| O28 | **Sharding** | Partition by tenant/hash key; avoid cross-shard joins. | Scales writes |
| O29 | **Covering indexes** | Index-only scans for hottest queries. | Reduces IO |
| O30 | **`SKIP LOCKED`** | Shopify's technique: one row per unit, `SELECT ... FOR UPDATE SKIP LOCKED` to reduce contention. | Reduces lock contention |
| O31 | **`READ COMMITTED` isolation** | Avoid gap locks (Shopify's lesson). | Reduces deadlocks |
| O32 | **Consistent lock ordering** | Standardize lock acquisition order to prevent deadlocks. | Reduces deadlocks |
| O33 | **Batching with `UNION ALL`** | Batch multi-line-item queries into one round trip. | Reduces latency |
| O34 | **Connection pooling** | PgBouncer/ProxySQL. | Prevents exhaustion |
| O35 | **Partitioning** | Time-based partitioning for orders/ledger. | Archival + query |
| O36 | **Columnar storage for analytics** | ClickHouse for analytics queries. | Fast analytics |
| O37 | **Connection tagging** | Tag SQL with business process ID for visibility (Shopify). | Debuggability |
| O38 | **InnoDB thread concurrency tuning** | Tune `innodb_thread_concurrency`. | Throughput |

### C.5 Queue / Async Optimizations

| # | Optimization | Detail | Impact |
|---|---|---|---|
| O39 | **Flash Sale Queue (FLQ)** | Redis-backed queue with capacity check, enqueue, drain. | Absorbs bursts |
| O40 | **Kafka with RF=3** | Replication factor 3, acks=all. | Durability |
| O41 | **Consumer groups** | Parallel consumption with partition key = tenant/sku. | Throughput |
| O42 | **DLQ (Dead Letter Queue)** | Failed messages to DLQ with retry policy. | Reliability |
| O43 | **Idempotent consumers** | Consumer-side idempotency via dedupe table. | Exactly-once |
| O44 | **Backpressure** | Consumer lag monitoring, autoscale consumers. | Stability |
| O45 | **Outbox pattern** | Transactional outbox for reliable event publishing. | No event loss |
| O46 | **Event replay** | Replay events to rebuild read models. | Recovery |

### O.6 Inventory Optimizations (Shopify/Shopee/Tencent lessons)

| # | Optimization | Detail | Source |
|---|---|---|---|
| O47 | **Redis pre-reduction** | Pre-warm stock into Redis; atomic Lua `DECR`; DB is source of truth. | Shopee, Tencent |
| O48 | **One row per unit + `SKIP LOCKED`** | Replace quantity column with per-unit rows; `SKIP LOCKED` to skip locked rows. | Shopify |
| O49 | **Bounded pool** | Cap available rows at 1,000 per item/location; inline replenishment. | Shopify |
| O50 | **Composite PK** | `(shop_id, inventory_item_id, inventory_group_id, id)` to reduce row locks. | Shopify |
| O51 | **Shadow mode cutover** | Run new system in parallel with old; validate before cutover. | Shopify |
| O52 | **Soft hold + expiry** | Time-boxed reservation (15 min) with auto-release. | Standard |
| O53 | **Warehouse sourcing** | Distance, stock, split-shipment. | Standard |

### O.7 Payment Optimizations

| # | Optimization | Detail | Impact |
|---|---|---|---|
| O54 | **PSP fallback routing** | Route to backup PSP on failure. | Higher success |
| O55 | **Payment retry** | Retry with backoff on transient failures. | Higher success |
| O56 | **Webhook dedupe** | Unique `psp_event_id`. | Exactly-once |
| O57 | **Idempotency keys** | Unique `idempotency_key` per request. | Exactly-once |
| O58 | **Tokenization** | Never store PANs; use PSP tokens. | PCI scope |
| O59 | **3DS2** | SCA compliance. | Fraud reduction |
| O60 | **Fraud detection** | Risk scoring, velocity checks, AVS/CVV. | Fraud reduction |
| O61 | **Payment analytics** | Success rate, decline reasons, PSP comparison. | Optimization |

### O.8 Observability & Reliability

| # | Optimization | Detail | Impact |
|---|---|---|---|
| O62 | **OpenTelemetry** | Distributed tracing across all services. | Debugging |
| O63 | **RED metrics** | Rate, Errors, Duration per service. | SLOs |
| O64 | **SLOs / SLIs** | Define SLOs, error budgets. | Reliability |
| O65 | **Grafana dashboards** | Per-service, per-tenant dashboards. | Visibility |
| O66 | **Log aggregation** | Loki/ELK with structured logs. | Debugging |
| O67 | **Chaos experiments** | Inject failures (network, DB, Kafka) in staging. | Prove resilience |
| O68 | **Game days** | Simulate flash sale + failure. | Preparedness |
| O69 | **Feature flags** | Kill switches for risky features. | Safe deploys |
| O70 | **Canary deploys** | 1% → 10% → 100% rollout. | Safe deploys |

### O.9 Security Optimizations

| # | Recommendation | Detail | Impact |
|---|---|---|---|
| O71 | **mTLS** | Service-to-service mTLS via service mesh. | Zero trust |
| O72 | **JWT rotation** | Short-lived access + refresh rotation. | Security |
| O73 | **Rate limiting** | Per-tenant, per-user, per-IP. | Abuse prevention |
| O74 | **WAF** | Managed WAF rules. | Attack prevention |
| O75 | **Secret management** | Vault/KMS. | Credential safety |
| O76 | **Audit logging** | Immutable audit trail. | Compliance |
| O77 | **GDPR/CCPA** | Data deletion, consent, export. | Compliance |
| O78 | **Penetration testing** | Regular pentests. | Security |

---

## PART D — INDUSTRY LESSONS APPLIED (Research Synthesis)

### D.1 Shopify: Inventory Reservations (2025-2026)

**Key insight:** Shopify moved from Redis to MySQL for inventory reservations because:
- Redis + separate ledger = no atomicity → oversell/undersell risk.
- MySQL with **one row per unit** + `SKIP LOCKED` + `READ COMMITTED` + composite PK = ACID + low contention.
- **Connection exhaustion** was the real bottleneck, not CPU/query speed — solved with connection visibility (tagged SQL) + PgBouncer + reducing 50% of reads and 33% of transactions on primary.
- **Shadow-mode cutover** — run both systems in parallel, validate, then cut over pod-by-pod.

**Application to our design:**
- Our `inventory_available` with `FOR UPDATE` is correct but will hit **connection exhaustion** at 100K RPS. We must add: PgBouncer, connection tagging, read replicas for inventory reads, and consider `SKIP LOCKED` for multi-unit reservations.
- Consider **shadow-mode validation** for any new inventory algorithm.

### D2: Shopee Flash Sale (C10M)

- **Atomic Redis Lua** for inventory decrement — pre-warm stock, atomic `DECR`, instant out-of-stock failure.
- **Atomic token bucket rate limiting** at gateway — smooth traffic shaping.
- **Flash Sale Queue Service (FLQ)** — queue requests before inventory/order.
- **C10M networking** — DPDK/eBPF/io_uring for millions of concurrent connections.
- **Service mesh** for East-West traffic.

**Application:**
- Add FLQ + Redis pre-reduction + atomic token bucket to our design. These are the **missing P0 pieces**.

### D3: Tencent Cloud Seckill (1M QPS)

- **Layered interception:** Redis pre-reduction → CKafka queue → MySQL transactional deduction.
- **Async processing:** All writes go to queue; user gets "queued" status.
- **Auto-scaling** based on queue backlog.
- **Lightweight validation** (captcha, rate limit) at edge.

**Application:**
- Our design needs the **queue layer** between gateway and order/inventory. This is the single biggest missing piece.

### D4: General 100K RPS Best Practices

- **Multi-layer cache** (CDN → edge → in-process → Redis) with TTL jitter, stampede prevention, hot-key protection.
- **Read replicas + follower reads** with lag guards.
- **Sharding** by tenant/hash.
- **Connection pooling** (PgBouncer).
- **Backpressure** before DB collapse.
- **Idempotency keys + DLQ** for queue safety.

---

## PART E — PRIORITIZED ROADMAP

### Phase 1: Foundation (P0 — Required for 100K RPS)

| Item | Type | Effort |
|---|---|---|
| Flash Sale Queue (FLQ) | Feature | High |
| Redis inventory pre-reduction | Optimization | High |
| Atomic token bucket rate limiting | Feature | High |
| Multi-layer caching (CDN → edge → in-process → Redis) | Optimization | High |
| Cache stampede + hot-key protection | Optimization | Medium |
| PgBouncer connection pooling | Optimization | Low |
| Read replicas + follower reads | Optimization | Medium |
| Outbox table DDL + Kafka | Feature | Medium |
| Idempotency keys table | Schema | Low |
| OpenTelemetry tracing + RED metrics | Observability | High |
| Grafana dashboards + alerting | Observability | Medium |
| Feature flags | Feature | Medium |
| CI/CD + GitOps | Infra | Medium |
| Secret management (Vault/KMS) | Security | Low |
| Backup & restore | Infra | Medium |

### Phase 1: Core Commerce (P1)

| # | Type | Effort |
|---|---|---|
| Order tracking (SSE/WebSocket) | Feature | Medium |
| Cart merge (anonymous → logged-in) | Feature | Medium |
| Wishlist | Feature | Low |
| Product reviews & ratings | Feature | Medium |
| Tax calculation | Feature | Medium |
| Shipping calculation | Feature | Medium |
| Order cancellation | Feature | Medium |
| Returns / RMA | Feature | High |
| Refund management | Feature | Medium |
| Promotions engine | Feature | High |
| Flash sale engine | Feature | High |
| Subscriptions / recurring billing | Feature | High |
| Multi-payment fallback routing | Feature | High |
| Address book | Feature | Low |
| Guest checkout | Feature | Low |
| SSO / OAuth | Feature | Medium |
| MFA / 2FA | Feature | Medium |
| Session management | Feature | Medium |
| Notification preferences | Feature | Low |
| GDPR / CCPA compliance | Feature | Medium |
| Personalized recommendations | Feature | High |
| Personalized search | Feature | High |
| A/B testing | Feature | Medium |
| Admin dashboard | Feature | Medium |
| OMS | Feature | High |
| Inventory management | Feature | Medium |
| Catalog management | Feature | Medium |
| Pricing management | Feature | Medium |
| Promotion management | Feature | Medium |
| Customer management | Feature | Medium |
| Tenant management | Feature | Medium |
| Audit log | Feature | Medium |
| Reporting & analytics | Feature | High |
| Service mesh (mTLS) | Infra | High |
| Chaos engineering | Infra | Medium |
| Multi-region DR | Infra | High |
| Tenant quotas & rate limits | Feature | Medium |
| Load balancing | Infra | Medium |

### Phase 2: Growth (P2)

| # | Type | Effort |
|---|---|---|
| Wishlist | Feature | Low |
| Product Q&A | Feature | Medium |
| Multi-currency | Feature | Medium |
| Gift cards | Feature | Medium |
| Bundle deals | Feature | Medium |
| Loyalty / rewards | Feature | Medium |
| Wallet | Feature | High |
| BNPL | Feature | Medium |
| COD | Feature | Medium |
| Invoice / receipt | Feature | Low |
| Trending / popular | Feature | Medium |
| New arrivals | Feature | Low |
| Deal hub | Feature | Low |
| Passwordless login | Feature | Low |
| Customer support / ticketing | Feature | High |
| Chatbot | Feature | High |
| Seller onboarding | Feature | High |
| Seller dashboard | Feature | High |
| Seller catalog | Feature | Medium |
| Seller inventory | Feature | Medium |
| Seller payouts | Feature | High |
| Seller disputes | Feature | Medium |
| CMS | Feature | Medium |
| SEO management | Feature | Medium |
| Webhooks outbound | Feature | Medium |
| API keys | Feature | Low |
| Sandbox environment | Feature | Medium |
| Cost management / FinOps | Infra | Medium |
| Data export | Feature | Low |

### Phase 3: Advanced (P3)

| # | Type | Effort |
|---|---|---|
| Visual search | Feature | High |
| Voice search | Feature | High |
| Edge personalization | Feature | High |
| Edge GraphQL | Feature | High |
| Real-time analytics | Feature | High |
| Predictive inventory | Feature | High |
| Dynamic pricing | Feature | High |
| AI chatbot | Feature | High |
| AR/VR product preview | Feature | High |

---

## PART F — RECOMMENDED ARCHITECTURE ADDITIONS (Concrete)

### F.1 Flash Sale Queue (FLQ) — Redis-backed

```
Client → API Gateway (rate limit) → FLQ (Redis)
  │  enqueue {user_id, sku, qty, ts}
  │  capacity check (per SKU)
  │  return 202 {queue_position}
  ▼
Drainer (Kafka consumer) → Inventory Service (Redis pre-reduction + DB)
  → Order Service → Payment Service
```

**Redis key structure:**
```
flash:sale:{deal_id}:{sku}:capacity   → INT (remaining)
flash:sale:{deal_id}:{sku}:queue      → LIST (FIFO)
flash:sale:{deal_id}:{sku}:processed  → SET (dedupe)
```

**Lua script (atomic):**
```lua
-- Check capacity, enqueue, return position
local capacity = tonumber(redis.call('GET', KEYS[1]) or '0')
if capacity <= 0 then return -1 end  -- sold out
local pos = redis.call('RPUSH', KEYS[2], ARGV[1])
redis.call('DECR', KEYS[1])
return pos
```

### F.2 Rate Limiting — Atomic Token Bucket (Redis Lua)

```lua
-- KEYS[1] = rate_limit:{tenant}:{user}:{endpoint}
-- ARGV[1] = capacity, ARGV[2] = refill_rate, ARGV[3] = now
local current = tonumber(redis.call('GET', KEYS[1]) or ARGV[1])
local last_refill = tonumber(redis.call('GET', KEYS[1]..':ts') or ARGV[3])
local elapsed = ARGV[3] - last_refill
current = math.min(ARGV[1], current + elapsed * ARGV[2])
if current >= 1 then
  redis.call('SET', KEYS[1], current - 1)
  redis.call('SET', KEYS[1]..':ts', ARGV[3])
  return 1  -- allowed
else
  return 0  -- rate limited
end
```

### F.3 Outbox Table DDL

```sql
CREATE TABLE outbox_events (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id     BIGINT NOT NULL,
    aggregate_type VARCHAR(64) NOT NULL,   -- order, payment, inventory
    aggregate_id  BIGINT NOT NULL,
    event_type    VARCHAR(64) NOT NULL,    -- ORDER_CREATED, PAYMENT_CONFIRMED
    payload       JSONB NOT NULL,
    status        SMALLINT NOT NULL DEFAULT 0,  -- 0=pending, 1=published, 2=failed
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at  TIMESTAMPTZ,
    UNIQUE (aggregate_type, aggregate_id, event_type)
);
CREATE INDEX idx_outbox_status ON outbox (status, created_at) WHERE status = 0;
```

### F.4 Idempotency Keys Table

```sql
CREATE TABLE idempotency_keys (
    tenant_id     BIGINT NOT NULL,
    key           VARCHAR(64) NOT NULL,
    request_hash  CHAR(64) NOT NULL,      -- SHA-256 of request body
    response_code SMALLINT NOT NULL,
    response_body JSONB NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at    TIMESTAMPTZ NOT NULL DEFAULT now() + interval '24 hours',
    PRIMARY KEY (tenant_id, key)
);
```

### F.5 Cart Redis Key Structure

```
cart:{tenant_id}:{user_id}          → Redis Hash
  field: {sku}:{warehouse_id}
  value: {qty, price_snapshot, added_at}

cart:{tenant_id}:{user_id}:meta     → Redis Hash
  {currency, last_updated, item_count}

cart:{tenant_id}:{user_id}:lock     → Redis String (checkout lock)
```

**Lua script (add to cart, atomic):**
```lua
-- KEYS[1] = cart hash, KEYS[2] = cart meta
-- ARGV[1] = sku, ARGV[2] = qty, ARGV[3] = price, ARGV[4] = now
local field = ARGV[1]
local exists = redis.call('HEXISTS', KEYS[1], field)
if exists == 1 then
  local val = cjson.decode(redis.call('HGET', KEYS[1], field))
  val.qty = val.qty + tonumber(ARGV[2])
  redis.call('HSET', KEYS[1], field, cjson.encode(val))
else
  local val = {qty = tonumber(ARGV[2]), price = tonumber(ARGV[3]), added_at = ARGV[4]}
  redis.call('HSET', KEYS[1], field, cjson.encode(val))
end
redis.call('EXPIRE', KEYS[1], 2592000)  -- 30 days
return redis.call('HLEN', KEYS[1])
```

### F.6 Multi-Region / DR Design

```
Region A (primary)          Region B (secondary)
┌──────────────────┐        ┌──────────────────┐
│ API Gateway      │        │ API Gateway      │
│ Services         │        │ Services (read)  │
│ PG (primary)     │◄──────►│ PG (replica)     │
│ Redis (primary)  │        │ Redis (replica)  │
│ Kafka (primary)  │        │ Kafka (mirror)   │
└──────────────────┘        └──────────────────┘
        │                          │
        └─────── Global DNS ───────┘
```

- **Reads:** active-active (both regions serve reads).
- **Writes:** active-passive (Region A primary; Region B standby).
- **Replication:** PG streaming replication (synchronous for orders/payment), Kafka MirrorMaker 2.
- **Failover:** DNS + health checks; RTO ≤ 60s, RPO ≤ 1s.
- **Conflict resolution:** single-writer per shard; no multi-master for orders/payment.

### F.7 Observability Stack

```
┌─────────────────────────────────────────────────┐
│  Grafana (dashboards + alerting)                │
│  ├── Prometheus (metrics)                       │
│  ├── Tempo (tracing)                            │
│  └── Loki (logs)                                │
├─────────────────────────────────────────────────┤
│  OpenTelemetry SDK (instrumentation)            │
│  ├── Service metrics (RED)                      │
│  ├── Distributed traces (OTel)                  │
│  └── Structured logs (JSON)                     │
├─────────────────────────────────────────────────┤
│  SLOs:                                          │
│  ├── Checkout P95 < 1.5s (error budget 0.001%)  │
│  ├── Inventory reserve P95 < 100ms              │
│  └── Search P95 < 50ms                          │
└─────────────────────────────────────────────────┘
```

---

## PART G — SUMMARY OF KEY RECOMMENDATIONS

### Top 10 Highest-Impact Actions

| Rank | Action | Type | Impact |
|---|---|---|---|
| 1 | **Add Flash Sale Queue (FLQ)** | Feature | Absorbs 100K RPS bursts; protects DB |
| 2 | **Add Redis inventory pre-reduction** | Optimization | 99% of stock checks never hit DB |
| 3 | **Add atomic token bucket rate limiting** | Feature | Smooths traffic; prevents abuse |
| 4 | **Add multi-layer cache with stampede/hot-key protection** | Optimization | Prevents cache meltdown |
| 5 | **Add PgBouncer + connection tagging** | Optimization | Prevents connection exhaustion (Shopify's #1 bottleneck) |
| 6 | **Add OpenTelemetry + Grafana + SLOs** | Observability | Required for 99.999% |
| 7 | **Add outbox + idempotency tables** | Schema | Guarantees exactly-once |
| 8 | **Add feature flags + canary deploys** | Infra | Safe changes at scale |
| 9 | **Add multi-region DR design** | Infra | Required for 99.999% |
| 10 | **Add chaos engineering** | Infra | Proves failure mitigations work |

### Key Architectural Corrections to Existing Document

1. **Inventory:** The current `FOR UPDATE` approach is correct but will hit connection exhaustion at 100K RPS. Add PgBouncer, read replicas, and consider Shopify's `SKIP LOCKED` + one-row-per-unit for extreme hot SKUs.
2. **Checkout:** The synchronous saga will not survive 100K RPS. Add the FLQ queue layer between gateway and order/inventory.
3. **Caching:** The document mentions Redis but lacks the multi-layer cache design, TTL jitter, stampede prevention, and hot-key handling that are mandatory at this scale.
4. **Observability:** Completely missing. Add OpenTelemetry, RED metrics, SLOs, and Grafana.
5. **Multi-region:** NFR claims 99.999% but no DR design exists. Add the multi-region topology above.

---

*This analysis is based on research from Shopify Engineering (BFCM 2025, inventory reservations), Shopee's flash-sale architecture, Tencent Cloud's seckill solution, and industry best practices for 100K RPS systems.*