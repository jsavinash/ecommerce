# Microservices Implementation Documentation
## Spring Boot 3.5 + Java 21 + Gradle Kotlin DSL

**Base Blueprint:** `ecommerce-platform-production-blueprint.md` (v3.0)
**Architecture:** Event-Driven Microservices · Orchestration Saga · FLQ Traffic Smoothing
**Target:** 100,000 RPS peak (flash sales) · 99.999% availability

---

## TABLE OF CONTENTS
1. [Technology Stack](#1-technology-stack)
2. [Repository & Multi-Module Structure](#2-repository--multi-module-structure)
3. [Common Modules (Shared Infrastructure)](#3-common-modules)
4. [Microservice Catalog](#4-microservice-catalog)
5. [Service-by-Service Detailed Design](#5-service-by-service-detailed-design)
   - 5.1 [Auth Service](#51-auth-service)
   - 5.2 [Catalog Service](#52-catalog-service)
   - 5.3 [Cart Service](#53-cart-service)
   - 5.4 [Order Service](#54-order-service)
   - 5.5 [Inventory Service](#55-inventory-service)
   - 5.6 [Payment Service](#56-payment-service)
   - 5.7 [Notification Service](#57-notification-service)
   - 5.8 [Flash Sale Queue (FLQ) Service](#58-flash-sale-queue-flq-service)
   - 5.9 [Recommendation Service](#59-recommendation-service)
   - 5.10 [Rate Limit Service](#510-rate-limit-service)
   - 5.11 [Feature Flag Service](#511-feature-flag-service)
6. [Inter-Service Communication](#6-inter-service-communication)
7. [Resilience & Fault Tolerance](#7-resilience--fault-tolerance)
8. [Observability & Monitoring](#8-observability--monitoring)
9. [Security](#9-security)
10. [Kafka Topics & Event Contracts](#10-kafka-topics--event-contracts)
11. [Deployment & DevOps](#11-deployment--devops)
12. [Local Development Setup](#12-local-development-setup)

---

## 1. TECHNOLOGY STACK

| Component | Technology | Version | Rationale |
|---|---|---|---|
| Language | Java | 21 (LTS) | Virtual threads, pattern matching, records, sealed interfaces |
| Framework | Spring Boot | 3.5.x | Latest stable, Spring Framework 6.2, Jakarta EE |
| Build | Gradle | 8.14+ | Kotlin DSL for type-safe build scripts |
| API | Spring Web MVC (REST) + gRPC | springdoc-openapi 2.8+ | REST for external, gRPC for internal |
| Event Bus | Spring Kafka | 3.3+ | Async, exactly-once-ish, replay |
| Cache | Spring Data Redis | 3.5+ | Carts, hot keys, locks, FLQ |
| DB | Spring Data JPA + Flyway | 3.5+ / 11.x | Orders, inventory, payment, users |
| Search | Spring Data Elasticsearch | 5.5+ | Catalog search |
| Resilience | Resilience4j | 2.3+ | Circuit breaker, bulkhead, retry, rate limiter |
| Observability | Micrometer + OpenTelemetry | 1.15+ | Metrics, traces, logs |
| Security | Spring Security + JWT + Spring Cloud Gateway | 6.5+ | AuthN/AuthZ |
| Service Discovery | Spring Cloud Netflix Eureka or Consul | 4.2+ | Service registry |
| Config | Spring Cloud Config Server | 4.2+ | Centralized config |
| Testing | JUnit 5, Testcontainers, Mockito | 5.11+ | Unit + integration tests |

---

## 2. REPOSITORY & MULTI-MODULE STRUCTURE

```
ecommerce-platform/
├── build.gradle.kts                    # Root build (Kotlin DSL)
├── settings.gradle.kts                 # Module declarations
├── gradle/
│   ├── libs.versions.toml              # Version catalog
│   └── wrapper/
├── common/
│   ├── common-core/                    # Shared DTOs, exceptions, HTTP
│   ├── common-observability/           # OTel config, metrics, logging
│   ├── common-resilience/              # Resilience4j config
│   ├── common-security/                # JWT, mTLS, tenant context
│   ├── common-kafka/                   # Kafka producer/consumer config
│   ├── common-redis/                   # Redis config, Lua scripts
│   └── common-persistence/             # JPA config, Flyway, PgBouncer
├── services/
│   ├── auth-service/
│   ├── catalog-service/
│   ├── cart-service/
│   ├── order-service/
│   ├── inventory-service/
│   ├── payment-service/
│   ├── notification-service/
│   ├── flq-service/
│   ├── recommendation-service/
│   ├── rate-limit-service/
│   ├── feature-flag-service/
│   └── api-gateway/
├── infrastructure/
│   ├── k8s/                            # Kubernetes manifests
│   ├── terraform/                      # IaC
│   └── docker-compose/                 # Local dev
└── docs/
```

### settings.gradle.kts

```kotlin
rootProject.name = "ecommerce-platform"

include(
    // Common modules
    ":common:common-core",
    ":common:common-observability",
    ":common:common-resilience",
    ":common:common-security",
    ":common:common-kafka",
    ":common:common-redis",
    ":common:common-persistence",
    // Services
    ":services:auth-service",
    ":services:catalog-service",
    ":services:cart-service",
    ":services:order-service",
    ":services:inventory-service",
    ":services:payment-service",
    ":services:notification-service",
    ":services:flq-service",
    ":services:recommendation-service",
    ":services:rate-limit-service",
    ":services:feature-flag-service",
    ":services:api-gateway"
)
```

### Root build.gradle.kts

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.5.3" apply false
    id("io.spring.dependency-management") version "1.1.7" apply false
    id("org.graalvm.buildtools.native") version "0.10.6" apply false
}

group = "com.ecommerce"
version = "1.0.0"

allprojects {
    repositories {
        mavenCentral()
        maven { url = uri("https://repo.spring.io/milestone") }
    }
}

subprojects {
    apply(plugin = "java")
    apply(plugin = "io.spring.dependency-management")

    java {
        toolchain {
            languageVersion = JavaLanguageVersion.of(21)
        }
    }

    tasks.withType<JavaCompile> {
        options.encoding = "UTF-8"
        options.compilerArgs.add("-parameters")
    }
}

// Shared dependency versions in gradle/libs.versions.toml
```

### gradle/libs.versions.toml

```toml
[versions]
spring-boot = "3.5.3"
spring-cloud = "2025.0.1"
java = "21"
lombok = "1.18.40"
grpc = "1.70.0"
protobuf = "4.30.1"
resilience4j = "2.3.0"
opentelemetry = "1.47.0"
micrometer = "1.15.0"
testcontainers = "1.20.6"
flyway = "11.7.2"
junit = "5.11.4"

[libraries]
spring-boot-starter-web = { module = "org.springframework.boot:spring-boot-starter-web" }
spring-boot-starter-webflux = { module = "org.springframework.boot:spring-boot-starter-webflux" }
spring-boot-starter-data-jpa = { module = "org.springframework.boot:spring-boot-starter-data-jpa" }
spring-boot-starter-data-redis = { module = "org.springframework.boot:spring-boot-starter-data-redis" }
spring-boot-starter-data-elasticsearch = { module = "org.springframework.boot:spring-boot-starter-data-elasticsearch" }
spring-boot-starter-security = { module = "org.springframework.boot:spring-boot-starter-security" }
spring-boot-starter-validation = { module = "org.springframework.boot:spring-boot-starter-validation" }
spring-boot-starter-actuator = { module = "org.springframework.boot:spring-boot-starter-actuator" }
spring-kafka = { module = "org.springframework.kafka:spring-kafka" }
springdoc = { module = "org.springdoc:springdoc-openapi-starter-webmvc-ui", version = "2.8.7" }
flyway-core = { module = "org.flywaydb:flyway-core", version.ref = "flyway" }
flyway-database-postgresql = { module = "org.flywaydb:flyway-database-postgresql", version.ref = "flyway" }
postgresql = { module = "org.postgresql:postgresql" }
lombok = { module = "org.projectlombok:lombok", version.ref = "lombok" }
resilience4j-spring-boot3 = { module = "io.github.resilience4j:resilience4j-spring-boot3", version.ref = "resilience4j" }
resilience4j-circuitbreaker = { module = "io.github.resilience4j:resilience4j-circuitbreaker", version.ref = "resilience4j" }
resilience4j-bulkhead = { module = "io.github.resilience4j:resilience4j-bulkhead", version.ref = "resilience4j" }
resilience4j-retry = { module = "io.github.resilience4j:resilience4j-retry", version.ref = "resilience4j" }
resilience4j-ratelimiter = { module = "io.github.resilience4j:resilience4j-ratelimiter", version.ref = "resilience4j" }
micrometer-registry-prometheus = { module = "io.micrometer:micrometer-registry-prometheus", version.ref = "micrometer" }
opentelemetry-annotations = { module = "io.opentelemetry:opentelemetry-annotations", version.ref = "opentelemetry" }
opentelemetry-exporter-otlp = { module = "io.opentelemetry:opentelemetry-exporter-otlp", version.ref = "opentelemetry" }
testcontainers-bom = { module = "org.testcontainers:testcontainers-bom", version.ref = "testcontainers" }
junit-jupiter = { module = "org.junit.jupiter:junit-jupiter", version.ref = "junit" }

[bundles]
common-logging = ["spring-boot-starter-actuator", "micrometer-registry-prometheus"]
db-postgres = ["spring-boot-starter-data-jpa", "flyway-core", "flyway-database-postgresql", "postgresql"]
resilience = ["resilience4j-spring-boot3", "resilience4j-circuitbreaker", "resilience4j-bulkhead", "resilience4j-retry", "resilience4j-ratelimiter"]
```

---

## 3. COMMON MODULES

### 3.1 common-core — Shared DTOs & Exceptions

```kotlin
// build.gradle.kts
plugins {
    id("java-library")
}

dependencies {
    api("org.springframework.boot:spring-boot-starter-validation")
}
```

```java
// ApiResponse.java
package com.ecommerce.common.core;

public record ApiResponse<T>(
    String code,
    String message,
    T data,
    int status
) {
    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>("SUCCESS", "OK", data, 200);
    }

    public static <T> ApiResponse<T> error(String code, String message, int status) {
        return new ApiResponse<>(code, message, null, status);
    }
}
```

```java
// ErrorCode.java
package com.ecommerce.common.core;

public enum ErrorCode {
    INSUFFICIENT_STOCK("INSUFFICIENT_STOCK", 409),
    PRICE_CHANGED("PRICE_CHANGED", 409),
    CART_NOT_FOUND("CART_NOT_FOUND", 404),
    RATE_LIMITED("RATE_LIMITED", 429),
    QUEUE_FULL("QUEUE_FULL", 503),
    INVALID_REQUEST("INVALID_REQUEST", 422),
    UNAUTHORIZED("UNAUTHORIZED", 401),
    PAYMENT_FAILED("PAYMENT_FAILED", 402),
    INTERNAL_ERROR("INTERNAL_ERROR", 500);

    private final String code;
    private final int httpStatus;

    ErrorCode(String code, int httpStatus) {
        this.code = code;
        this.httpStatus = httpStatus;
    }

    public String getCode() { return code; }
    public int getHttpStatus() { return httpStatus; }
}
```

```java
// BusinessException.java
package com.ecommerce.common.core;

import lombok.Getter;

@Getter
public class BusinessException extends RuntimeException {
    private final ErrorCode errorCode;
    private final Object[] args;

    public BusinessException(ErrorCode errorCode, Object... args) {
        super(errorCode.getCode());
        this.errorCode = errorCode;
        this.args = args;
    }
}
```

```java
// GlobalExceptionHandler.java
package com.ecommerce.common.core;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ApiResponse<Void>> handleBusinessException(BusinessException ex) {
        ErrorCode code = ex.getErrorCode();
        return ResponseEntity
            .status(code.getHttpStatus())
            .body(ApiResponse.error(code.getCode(), ex.getMessage(), code.getHttpStatus()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Void>> handleValidation(MethodArgumentNotValidException ex) {
        return ResponseEntity
            .status(422)
            .body(ApiResponse.error("INVALID_REQUEST", "Validation failed", 422));
    }
}
```

### 3.2 common-security — JWT, Tenant Context

```java
// TenantContext.java
package com.ecommerce.common.security;

public final class TenantContext {
    private static final ThreadLocal<Long> CURRENT_TENANT = new ThreadLocal<>();

    private TenantContext() {}

    public static void setTenantId(Long tenantId) {
        CURRENT_TENANT.set(tenantId);
    }

    public static Long getTenantId() {
        return CURRENT_TENANT.get();
    }

    public static void clear() {
        CURRENT_TENANT.remove();
    }
}
```

```java
// JwtAuthenticationFilter.java
package com.ecommerce.common.security;

@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final String userServiceUrl;

    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
    ) throws ServletException, IOException {

        String tenantHeader = request.getHeader("X-Tenant-Id");
        if (tenantHeader != null) {
            TenantContext.setTenantId(Long.parseLong(tenantHeader));
        }

        String authHeader = request.getHeader("Authorization");
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7);
            try {
                JwtClaims claims = jwtService.validateToken(token);
                request.setAttribute("userId", claims.userId());
                request.setAttribute("roles", claims.roles());
                request.setAttribute("tenantId", claims.tenantId());
            } catch (JwtException e) {
                response.setStatus(401);
                response.getWriter().write("{\"code\":\"UNAUTHORIZED\",\"message\":\"Invalid token\"}");
                return;
            }
        }

        try {
            filterChain.doFilter(request, response);
        } finally {
            TenantContext.clear();
        }
    }
}
```

```java
// JwtService.java
package com.ecommerce.common.security;

import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.Date;
import java.util.List;

@Service
public class JwtService {

    private final SecretKey key;
    private final long accessTokenTtlSeconds;

    public JwtService(
        @Value("${security.jwt.secret}") String secret,
        @Value("${security.jwt.access-token-ttl-seconds:900}") long accessTokenTtlSeconds
    ) {
        this.key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.accessTokenTtlSeconds = accessTokenTtlSeconds;
    }

    public String generateToken(Long userId, Long tenantId, List<String> roles) {
        Instant now = Instant.now();
        return Jwts.builder()
            .subject(userId.toString())
            .claim("tenantId", tenantId)
            .claim("roles", roles)
            .issuedAt(Date.from(now))
            .expiration(Date.from(now.plusSeconds(accessTokenTtlSeconds)))
            .signWith(key)
            .compact();
    }

    public JwtClaims validateToken(String token) {
        var claims = Jwts.parser()
            .verifyWith(key)
            .build()
            .parseSignedClaims(token)
            .getPayload();

        return new JwtClaims(
            Long.parseLong(claims.getSubject()),
            Long.parseLong(claims.get("tenantId", String.class)),
            claims.get("roles", List.class)
        );
    }

    public record JwtClaims(Long userId, Long tenantId, List<String> roles) {}
}
```

### 3.3 common-observability — OpenTelemetry + Micrometer

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.micrometer:micrometer-registry-prometheus")
    implementation("io.opentelemetry:opentelemetry-api")
    implementation("io.opentelemetry.instrumentation:opentelemetry-spring-boot-starter:2.14.0")
    implementation("io.micrometer:micrometer-tracing-bridge-otel")
    implementation("io.opentelemetry:opentelemetry-exporter-otlp")
}
```

```yaml
# application.yml (common observability snippet)
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  metrics:
    tags:
      application: ${spring.application.name}
  tracing:
    sampling:
      probability: 0.1   # 10% for 100K RPS
  otlp:
    tracing:
      endpoint: http://tempo:4317
```

```java
// MetricsRegistry.java
package com.ecommerce.observability;

@Component
public class MetricsRegistry {

    private final MeterRegistry meterRegistry;

    public MetricsRegistry(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    public void recordLatency(String operation, long durationMs, boolean success) {
        Timer.Sample sample = Timer.start(meterRegistry);
        Timer timer = Timer.builder("ecommerce.request.latency")
            .tag("operation", operation)
            .tag("success", String.valueOf(success))
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(meterRegistry);
        timer.record(Duration.ofMillis(durationMs));
    }

    public void incrementCounter(String name, String... tags) {
        meterRegistry.counter(name, tags).increment();
    }

    public void observeGauge(String name, double value, String... tags) {
        meterRegistry.gauge(name, tags, value);
    }
}
```

### 3.4 common-kafka — Kafka Configuration

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.kafka:spring-kafka")
}
```

```java
// KafkaProducerConfig.java
package com.ecommerce.common.kafka;

@Configuration
public class KafkaProducerConfig {

    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate(ProducerFactory<String, Object> producerFactory) {
        return new KafkaTemplate<>(producerFactory);
    }

    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "${kafka.bootstrap-servers}");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        props.put(ProducerConfig.ACKS_CONFIG, "all");                    // RF=3 durability
        props.put(ProducerConfig.RETRIES_CONFIG, 3);
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);       // Exactly-once produce
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");        // Bandwidth efficiency
        props.put(JsonSerializer.ADD_TYPE_INFO_HEADERS, false);
        return new DefaultKafkaProducerFactory<>(props);
    }
}
```

```java
// OutboxEvent.java
package com.ecommerce.common.kafka;

@Entity
@Table(name = "outbox_events")
public class OutboxEvent {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "tenant_id", nullable = false)
    private Long tenantId;

    @Column(name = "aggregate_type", nullable = false)
    private String aggregateType;

    @Column(name = "aggregate_id", nullable = false)
    private Long aggregateId;

    @Column(name = "event_type", nullable = false)
    private String eventType;

    @Column(nullable = false, columnDefinition = "jsonb")
    private String payload;

    @Column(nullable = false)
    private Integer status = 0;  // 0=pending, 1=published, 2=failed

    @Column(name = "created_at")
    private Instant createdAt = Instant.now();
}
```

```java
// OutboxPublisher.java
package com.ecommerce.common.kafka;

@Component
@RequiredArgsConstructor
@Slf4j
public class OutboxPublisher {

    private final OutboxEventRepository outboxRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @Transactional
    public void publishPendingEvents() {
        List<OutboxEvent> pending = outboxRepository.findPendingEvents(100);
        for (OutboxEvent event : pending) {
            try {
                kafkaTemplate.send(
                    event.getEventType().toLowerCase().contains("order") ? "order.events"
                        : event.getEventType().toLowerCase().contains("payment") ? "payment.events"
                        : "inventory.events",
                    event.getAggregateId().toString(),
                    event.getPayload()
                ).get(2, TimeUnit.SECONDS);

                event.setStatus(1);
                outboxRepository.save(event);
            } catch (Exception e) {
                log.error("Failed to publish outbox event {}: {}", event.getId(), e.getMessage());
                event.setStatus(2);
            }
        }
    }
}
```

### 3.5 common-redis — Redis & Lua Scripts

```java
// RedisLuaScripts.java
package com.ecommerce.common.redis;

@Component
public final class RedisLuaScripts {

    // Cart add (atomic, prevents race conditions)
    public static final String CART_ADD = """
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
        redis.call('EXPIRE', KEYS[1], 2592000)
        return redis.call('HLEN', KEYS[1])
        """;

    // FLQ enqueue (atomic, checks capacity + dedupe)
    public static final String FLQ_ENQUEUE = """
        local capacity = tonumber(redis.call('GET', KEYS[1]) or '0')
        if capacity <= 0 then return -1 end
        if redis.call('SISMEMBER', KEYS[3], ARGV[1]) == 1 then return -2 end
        local pos = redis.call('RPUSH', KEYS[2], ARGV[1] .. ':' .. ARGV[2] .. ':' .. ARGV[3])
        redis.call('DECR', KEYS[1])
        redis.call('SADD', KEYS[3], ARGV[1])
        return pos
        """;

    // Rate limit (token bucket)
    public static final String RATE_LIMIT_TOKEN_BUCKET = """
        local current = tonumber(redis.call('GET', KEYS[1]) or ARGV[1])
        local last_refill = tonumber(redis.call('GET', KEYS[1]..':ts') or ARGV[3])
        local elapsed = ARGV[3] - last_refill
        current = math.min(ARGV[1], current + elapsed * ARGV[2])
        if current >= 1 then
          redis.call('SET', KEYS[1], current - 1)
          redis.call('SET', KEYS[1]..':ts', ARGV[3])
          return 1
        else
          return 0
        end
        """;

    // Inventory pre-reduction (hot SKU)
    public static final String INVENTORY_PRE_DEDUCT = """
        local remaining = tonumber(redis.call('GET', KEYS[1]) or '-1')
        if remaining < 0 then return -1 end
        if remaining < tonumber(ARGV[1]) then return -2 end
        return redis.call('DECRBY', KEYS[1], ARGV[1])
        """;
}
```

### 3.6 common-persistence — JPA + Flyway + PgBouncer

```yaml
# application.yml snippet
spring:
  datasource:
    url: jdbc:postgresql://pgbouncer:6432/${db.name}  # PgBouncer pool
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 5000
      pool-name: ${spring.application.name}-pool
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
  flyway:
    enabled: true
    locations: classpath:db/migration
```

---

## 4. MICROSERVICE CATALOG

| # | Service | Port | Database | Key Tech | Purpose |
|---|---|---|---|---|---|
| 1 | Auth Service | 8081 | PostgreSQL | Spring Security, JWT, Redis sessions | AuthN/AuthZ, MFA, RBAC |
| 2 | Catalog Service | 8082 | PostgreSQL + ES + Redis | Elasticsearch, Redis cache | Product catalog, search, facets |
| 3 | Cart Service | 8083 | Redis Cluster | Redis + Lua | Distributed cart, wishlist, merge |
| 4 | Order Service | 8084 | PostgreSQL | Saga orchestrator | Checkout orchestration, order lifecycle |
| 5 | Inventory Service | 8085 | PostgreSQL + Redis | JPA, Redlock, Redis pre-reduction | Stock reservation, ledger |
| 6 | Payment Service | 8086 | PostgreSQL | PSP integrations | Payment state machine, refunds |
| 7 | Notification Service | 8087 | Kafka + PostgreSQL | Kafka consumers | Email, SMS, push |
| 8 | FLQ Service | 8088 | Redis Cluster | Redis Lua | Flash sale queue, traffic smoothing |
| 9 | Recommendation Service | 8089 | Redis + ClickHouse | Machine learning | Personalized recommendations |
| 10 | Rate Limit Service | 8090 | Redis Cluster | Redis Lua | Token bucket rate limiting |
| 11 | Feature Flag Service | 8091 | Redis + PostgreSQL | Redis cache | Feature flags, A/B testing |
| 12 | API Gateway | 8080 | — | Spring Cloud Gateway | Edge routing, rate limit, auth |

---

## 5. SERVICE-BY-SERVICE DETAILED DESIGN

### 5.1 Auth Service

**Purpose:** User authentication, session management, MFA, RBAC, tenant isolation.

#### build.gradle.kts

```kotlin
plugins {
    id("org.springframework.boot")
    id("io.spring.dependency-management")
}

dependencies {
    implementation(project(":common:common-core"))
    implementation(project(":common:common-security"))
    implementation(project(":common:common-observability"))
    implementation(project(":common:common-persistence"))

    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.jsonwebtoken:jjwt-api:0.12.6")
    runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.6")
    runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.12.6")
    implementation("org.postgresql:postgresql")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")

    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")

    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.testcontainers:postgresql")
    testImplementation("org.testcontainers:junit-jupiter")
}

dependencyManagement {
    imports {
        mavenBom("org.testcontainers:testcontainers-bom:1.20.6")
    }
}
```

#### application.yml

```yaml
spring:
  application:
    name: auth-service
  datasource:
    url: jdbc:postgresql://pgbouncer:6432/ecommerce_auth
    username: ${DB_USER}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate    # Flyway manages schema
  data:
    redis:
      host: redis-auth
      port: 6379

server:
  port: 8081

security:
  jwt:
    secret: ${JWT_SECRET}
    access-token-ttl-seconds: 900
    refresh-token-ttl-days: 30

resilience4j.circuitbreaker:
  instances:
    redis:
      slidingWindowSize: 10
      failureRateThreshold: 50
      waitDurationInOpenState: 10s

eureka:
  client:
    service-url:
      defaultZone: http://eureka:8761/eureka/
```

#### Entitites

```java
// User.java
package com.ecommerce.auth.entity;

@Entity
@Table(name = "users", uniqueConstraints = {
    @UniqueConstraint(columnNames = {"tenant_id", "email"})
})
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "tenant_id", nullable = false)
    private Long tenantId;

    @Column(nullable = false)
    private String email;

    @Column(name = "password_hash", nullable = false)
    private String passwordHash;

    private String phone;

    @Column(nullable = false)
    private Integer status = 1;  // 1=active, 2=suspended

    @Column(name = "created_at")
    private Instant createdAt = Instant.now();

    @Column(name = "updated_at")
    private Instant updatedAt = Instant.now();
}
```

```java
// RefreshToken.java
package com.ecommerce.auth.entity;

@Entity
@Table(name = "refresh_tokens")
public class RefreshToken {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(name = "tenant_id", nullable = false)
    private Long tenantId;

    @Column(nullable = false, unique = true)
    private String token;    // JWT refresh token hash

    @Column(name = "expires_at", nullable = false)
    private Instant expiresAt;

    @Column(nullable = false)
    private Boolean revoked = false;

    @Column(name = "device_id")
    private String deviceId;
}
```

#### Controllers & Services

```java
// AuthController.java
package com.ecommerce.auth.controller;

@RestController
@RequestMapping("/api/v1/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthService authService;

    @PostMapping("/login")
    public ResponseEntity<TokenResponse> login(@RequestBody @Valid LoginRequest request,
                                                @RequestHeader("X-Tenant-Id") Long tenantId) {
        return ResponseEntity.ok(authService.login(request, tenantId));
    }

    @PostMapping("/register")
    public ResponseEntity<UserResponse> register(@RequestBody @Valid RegisterRequest request,
                                                  @RequestHeader("X-Tenant-Id") Long tenantId) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(authService.register(request, tenantId));
    }

    @PostMapping("/refresh")
    public ResponseEntity<TokenResponse> refresh(@RequestBody RefreshRequest request,
                                                  @RequestHeader("X-Tenant-Id") Long tenantId) {
        return ResponseEntity.ok(authService.refresh(request.refreshToken(), tenantId));
    }

    @PostMapping("/logout")
    public ResponseEntity<Void> logout(@RequestBody LogoutRequest request,
                                        @RequestHeader("X-Tenant-Id") Long tenantId) {
        authService.logout(request.refreshToken(), tenantId);
        return ResponseEntity.noContent().build();
    }

    @PostMapping("/mfa/enable")
    public ResponseEntity<MfaResponse> enableMfa(@AuthenticationPrincipal Long userId,
                                                  @RequestHeader("X-Tenant-Id") Long tenantId) {
        return ResponseEntity.ok(authService.enableMfa(userId, tenantId));
    }

    @PostMapping("/verify-mfa")
    public ResponseEntity<TokenResponse> verifyMfa(@RequestBody VerifyMfaRequest request) {
        return ResponseEntity.ok(authService.verifyMfa(request));
    }
}
```

```java
// AuthService.java
package com.ecommerce.auth.service;

@Service
@RequiredArgsConstructor
@Slf4j
public class AuthService {

    private final UserRepository userRepository;
    private final RefreshTokenRepository refreshTokenRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtService jwtService;
    private final StringRedisTemplate redisTemplate;
    private final TOTPService totpService;

    @Transactional
    public TokenResponse login(LoginRequest request, Long tenantId) {
        User user = userRepository.findByTenantIdAndEmail(tenantId, request.email())
            .orElseThrow(() -> new BusinessException(ErrorCode.UNAUTHORIZED, "Invalid credentials"));

        if (user.getStatus() != 1) {
            throw new BusinessException(ErrorCode.UNAUTHORIZED, "Account suspended");
        }

        if (!passwordEncoder.matches(request.password(), user.getPasswordHash())) {
            throw new BusinessException(ErrorCode.UNAUTHORIZED, "Invalid credentials");
        }

        // Check MFA
        if (user.isMfaEnabled()) {
            return TokenResponse.pendingMfa(user.getId());
        }

        return issueTokens(user);
    }

    @Transactional
    public TokenResponse register(RegisterRequest request, Long tenantId) {
        boolean exists = userRepository.existsByTenantIdAndEmail(tenantId, request.email());
        if (exists) {
            throw new BusinessException(ErrorCode.INVALID_REQUEST, "Email already registered");
        }

        User user = new User();
        user.setTenantId(tenantId);
        user.setEmail(request.email());
        user.setPasswordHash(passwordEncoder.encode(request.password()));
        user.setPhone(request.phone());
        userRepository.save(user);

        return issueTokens(user);
    }

    @Transactional
    public TokenResponse refresh(String refreshToken, Long tenantId) {
        RefreshToken stored = refreshTokenRepository.findByTokenAndTenantId(refreshToken, tenantId)
            .orElseThrow(() -> new BusinessException(ErrorCode.UNAUTHORIZED, "Invalid refresh token"));

        if (stored.getRevoked() || stored.getExpiresAt().isBefore(Instant.now())) {
            throw new BusinessException(ErrorCode.UNAUTHORIZED, "Refresh token expired");
        }

        User user = userRepository.findById(stored.getUserId())
            .orElseThrow(() -> new BusinessException(ErrorCode.UNAUTHORIZED, "User not found"));

        stored.setRevoked(true);
        refreshTokenRepository.save(stored);

        return issueTokens(user);
    }

    private TokenResponse issueTokens(User user) {
        String accessToken = jwtService.generateToken(user.getId(), user.getTenantId(),
            List.of(user.getRole().name()));

        String refreshToken = UUID.randomUUID().toString();
        RefreshToken rt = new RefreshToken();
        rt.setUserId(user.getId());
        rt.setTenantId(user.getTenantId());
        rt.setToken(refreshToken);
        rt.setExpiresAt(Instant.now().plus(Duration.ofDays(30)));
        refreshTokenRepository.save(rt);

        // Rate limit per user: 100 RPS
        redisTemplate.opsForValue().set(
            "rate_limit:" + user.getTenantId() + ":" + user.getId() + ":*",
            "100", Duration.ofMinutes(5));

        return new TokenResponse(accessToken, refreshToken, 900L, user.getTenantId());
    }
}
```

### 5.2 Catalog Service

**Purpose:** Product catalog, search, facets, autocomplete.

#### build.gradle.kts

```kotlin
plugins {
    id("org.springframework.boot")
    id("io.spring.dependency-management")
}

dependencies {
    implementation(project(":common:common-core"))
    implementation(project(":common:common-observability"))
    implementation(project(":common:common-persistence"))
    implementation(project(":common:common-kafka"))

    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.boot:spring-boot-starter-data-elasticsearch")
    implementation("org.springframework.kafka:spring-kafka")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.7")
    implementation("org.postgresql:postgresql")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")

    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")
}
```

#### application.yml

```yaml
spring:
  application:
    name: catalog-service
  datasource:
    url: jdbc:postgresql://pgbouncer:6432/ecommerce_catalog
  data:
    redis:
      host: redis-catalog
    elasticsearch:
      uris: http://elasticsearch:9200

server:
  port: 8082

ecommerce:
  elasticsearch:
    index: catalog
    shards: 6           # 3 nodes × 2 shards
    replicas: 2
  cache:
    product-ttl-seconds: 300
    search-ttl-seconds: 60
```

#### Elasticsearch Index Mapping (JSON)

```json
{
  "settings": {
    "number_of_shards": 6,
    "number_of_replicas": 2,
    "analysis": {
      "analyzer": {
        "autocomplete": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "edge_ngram_filter"]
        }
      },
      "filter": {
        "edge_ngram_filter": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 20
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "tenant_id": { "type": "long" },
      "sku": { "type": "keyword" },
      "name": {
        "type": "text",
        "analyzer": "autocomplete",
        "fields": { "keyword": { "type": "keyword" } }
      },
      "description": { "type": "text" },
      "category_id": { "type": "long" },
      "brand": { "type": "keyword" },
      "base_price": { "type": "double" },
      "currency": { "type": "keyword" },
      "status": { "type": "integer" },
      "available": { "type": "boolean" },
      "rating_avg": { "type": "float" },
      "created_at": { "type": "date" }
    }
  }
}
```

#### Services

```java
// ProductSearchService.java
package com.ecommerce.catalog.service;

@Service
@RequiredArgsConstructor
public class ProductSearchService {

    private final ElasticsearchOperations esOperations;
    private final StringRedisTemplate redisTemplate;

    /**
     * Search with multi-layer cache:
     * 1. Redis cache (TTL = query hash)
     * 2. Elasticsearch
     * 3. Optional in-process cache with request coalescing
     */
    public SearchResponse search(SearchRequest request, Long tenantId) {
        // Cache key: normalized query
        String cacheKey = "catalog:search:" + tenantId + ":" +
            sha256(request.query() + request.filters() + request.sort() + request.page());

        // 1. Check Redis cache
        String cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return JsonUtils.fromJson(cached, SearchResponse.class);
        }

        // 2. Build ES query (with tenant filter for multi-tenancy)
        NativeSearchQuery query = buildSearchQuery(request, tenantId);
        SearchHits<ProductDocument> hits = esOperations.search(query, ProductDocument.class);

        SearchResponse response = mapToResponse(hits, request);

        // 3. Cache with TTL jitter (prevent thundering herd)
        long ttl = 60 + ThreadLocalRandom.current().nextLong(0, 30);
        redisTemplate.opsForValue().set(cacheKey, JsonUtils.toJson(response), Duration.ofSeconds(ttl));

        return response;
    }

    private NativeSearchQuery buildSearchQuery(SearchRequest request, Long tenantId) {
        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery()
            .filter(QueryBuilders.termQuery("tenant_id", tenantId))
            .filter(QueryBuilders.termQuery("status", 1));

        if (request.query() != null && !request.query().isBlank()) {
            boolQuery.must(QueryBuilders.multiMatchQuery(request.query(),
                "name^3", "description^1", "brand^2"));
        }

        // Facets
        if (request.filters() != null) {
            if (request.filters().categoryId() != null) {
                boolQuery.filter(QueryBuilders.termQuery("category_id", request.filters().categoryId()));
            }
            if (request.filters().brand() != null) {
                boolQuery.filter(QueryBuilders.termQuery("brand", request.filters().brand()));
            }
            if (request.filters().minPrice() != null) {
                boolQuery.filter(QueryBuilders.rangeQuery("base_price").gte(request.filters().minPrice()));
            }
            if (request.filters().available() != null && request.filters().available()) {
                boolQuery.filter(QueryBuilders.termQuery("available", true));
            }
        }

        NativeSearchQueryBuilder builder = new NativeSearchQueryBuilder()
            .withQuery(boolQuery)
            .withPageable(PageRequest.of(request.page() - 1, request.size()))
            .withAggregations(
                AggregationBuilders.terms("brands").field("brand"),
                AggregationBuilders.terms("categories").field("category_id"),
                AggregationBuilders.range("price_ranges").field("base_price")
                    .addRange(0, 50).addRange(50, 100).addRange(100, 200).addRange(200, null)
            );

        // Sorting
        if (request.sort() != null) {
            switch (request.sort()) {
                case "price_asc" -> builder.withSort(Sort.by(Sort.Order.asc("base_price")));
                case "price_desc" -> builder.withSort(Sort.by(Sort.Order.desc("base_price")));
                case "newest" -> builder.withSort(Sort.by(Sort.Order.desc("created_at")));
                case "rating" -> builder.withSort(Sort.by(Sort.Order.desc("rating_avg")));
                default -> builder.withSort(Sort.by(Sort.Order.desc("_score")));
            }
        }

        return builder.build();
    }
}
```

### 5.3 Cart Service

**Purpose:** Distributed cart with atomic Lua operations, wishlist, cart merge.

#### build.gradle.kts

```kotlin
plugins {
    id("org.springframework.boot")
    id("io.spring.dependency-management")
}

dependencies {
    implementation(project(":common:common-core"))
    implementation(project(":common:common-redis"))
    implementation(project(":common:common-observability"))

    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.kafka:spring-kafka")

    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")
}
```

#### application.yml

```yaml
spring:
  application:
    name: cart-service
  data:
    redis:
      cluster:
        nodes:
          - redis-cart-1:6379
          - redis-cart-2:6379
          - redis-cart-3:6379

server:
  port: 8083

kafka:
  bootstrap-servers: ${KAFKA_BOOTSTRAP:localhost:9092}

ecommerce:
  cart:
    ttl-days: 30
    switchOnExpiration: true
  redis:
    scripts:
      cart-add: |
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
        redis.call('EXPIRE', KEYS[1], 2592000)
        return redis.call('HLEN', KEYS[1])
```

#### CartService.java

```java
package com.ecommerce.cart.service;

@Service
@RequiredArgsConstructor
public class CartService {

    private final StringRedisTemplate redisTemplate;
    private final DefaultRedisScript<Long> cartAddScript;
    private final DefaultRedisScript<Long> cartMergeScript;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    private static final String CART_KEY_PREFIX = "cart:";
    private static final String ANON_CART_PREFIX = "cart:anon:";

    /**
     * Add item to cart atomically via Lua script.
     * Handles concurrent add/remove/update races via Redis single-threaded execution.
     */
    public CartResponse addItem(Long tenantId, Long userId, Long sessionId,
                                 CartItemRequest request) {
        String cartKey = userId != null
            ? CART_KEY_PREFIX + tenantId + ":" + userId
            : ANON_CART_PREFIX + tenantId + ":" + sessionId;

        Long itemCount = redisTemplate.execute(
            cartAddScript,
            List.of(cartKey, cartKey + ":meta"),
            request.sku() + ":" + request.warehouseId(),
            String.valueOf(request.quantity()),
            String.valueOf(request.price()),
            String.valueOf(Instant.now().toEpochMilli())
        );

        return new CartResponse(itemCount, getCartItems(tenantId, userId, sessionId));
    }

    /**
     * Merge anonymous cart into user cart on login.
     * Atomic via Lua — no race between merge and concurrent modifications.
     */
    public CartResponse mergeCarts(Long tenantId, Long userId, Long sessionId) {
        String anonKey = ANON_CART_PREFIX + tenantId + ":" + sessionId;
        String userKey = CART_KEY_PREFIX + tenantId + ":" + userId;

        Long itemCount = redisTemplate.execute(
            cartMergeScript,
            List.of(anonKey, userKey)
        );

        // Emit cart merge event for analytics
        kafkaTemplate.send("cart.events", userId.toString(),
            Map.of("type", "CART_MERGED", "tenantId", tenantId, "userId", userId));

        return new CartResponse(itemCount, getCartItems(tenantId, userId, null));
    }

    /**
     * Get cart items with price revalidation callback.
     */
    public List<CartItem> getCartItems(Long tenantId, Long userId, Long sessionId) {
        String cartKey = userId != null
            ? CART_KEY_PREFIX + tenantId + ":" + userId
            : ANON_CART_PREFIX + tenantId + ":" + sessionId;

        Map<Object, Object> entries = redisTemplate.opsForHash().entries(cartKey);
        return entries.entrySet().stream()
            .map(e -> JsonUtils.fromJson(e.getValue().toString(), CartItem.class))
            .toList();
    }

    /**
     * Checkout lock — prevents concurrent checkouts on same cart.
     */
    public boolean acquireCheckoutLock(Long tenantId, Long userId, String orderId) {
        String lockKey = CART_KEY_PREFIX + tenantId + ":" + userId + ":lock";
        return Boolean.TRUE.equals(redisTemplate.opsForValue()
            .setIfAbsent(lockKey, orderId, Duration.ofMinutes(5)));
    }

    public void releaseCheckoutLock(Long tenantId, Long userId) {
        String lockKey = CART_KEY_PREFIX + tenantId + ":" + userId + ":lock";
        redisTemplate.delete(lockKey);
    }
}
```

### 5.4 Order Service (Checkout Orchestrator)

**Purpose:** Checkout saga orchestrator, order lifecycle, state machine.

#### build.gradle.kts

```kotlin
plugins {
    id("org.springframework.boot")
    id("io.spring.dependency-management")
}

dependencies {
    implementation(project(":common:common-core"))
    implementation(project(":common:common-kafka"))
    implementation(project(":common:common-observability"))
    implementation(project(":common:common-persistence"))
    implementation(project(":common:common-resilience"))

    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.kafka:spring-kafka")
    implementation("org.postgresql:postgresql")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")
    implementation("io.github.resilience4j:resilience4j-spring-boot3:2.3.0")
    implementation("io.github.resilience4j:resilience4j-circuitbreaker")
    implementation("io.github.resilience4j:resilience4j-bulkhead")
    implementation("io.github.resilience4j:resilience4j-retry")

    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")
}
```

#### Order Saga Orchestrator

```java
package com.ecommerce.order.saga;

@Service
@Slf4j
@RequiredArgsConstructor
public class CheckoutSagaOrchestrator {

    private final CartServiceClient cartClient;
    private final CatalogServiceClient catalogClient;
    private final InventoryServiceClient inventoryClient;
    private final PaymentServiceClient paymentClient;
    private final OrderRepository orderRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final MetricsRegistry metricsRegistry;

    /**
     * Orchestrated saga:
     * STARTED → QUEUED → RESERVING → PAYMENT_INITIATED → CONFIRMED
     * On failure: COMPENSATING → COMPENSATED
     */
    @Transactional
    public CheckoutResponse checkout(CheckoutRequest request, Long tenantId, Long userId) {
        long startTime = System.currentTimeMillis();

        // Step 0: Create order (PENDING)
        Order order = createPendingOrder(request, tenantId, userId);

        try {
            // Step 1: Lock cart (read snapshot)
            CartSnapshot cart = cartClient.lockCart(tenantId, userId, order.getId());

            // Step 2: Re-validate price
            List<PriceChange> priceChanges = catalogClient.validatePrices(cart.items(), tenantId);
            if (!priceChanges.isEmpty()) {
                throw new PriceChangedException(priceChanges);
            }

            // Step 3: Enqueue to FLQ (traffic smoothing)
            QueueResponse queue = flqClient.enqueue(order.getId(), cart.items(), tenantId);
            updateSagaState(order, SagaState.QUEUED);

            // Step 4: Reserve inventory (Redis pre-reduction + DB)
            ReservationResponse reservation = inventoryClient.reserve(
                tenantId, order.getId(), cart.items(), order.getIdempotencyKey());
            updateSagaState(order, SagaState.RESERVING);

            // Step 5: Initiate payment (PSP routing with fallback)
            PaymentInitiation payment = paymentClient.initiate(
                tenantId, order.getId(), order.getTotal(), request.paymentMethod(),
                order.getIdempotencyKey());
            updateSagaState(order, SagaState.PAYMENT_INITIATED);

            // Step 6: Emit ORDER_CREATED (outbox)
            publishOrderCreated(order, cart);

            metricsRegistry.recordLatency("checkout", System.currentTimeMillis() - startTime, true);

            return new CheckoutResponse(order.getId(), order.getOrderNumber(),
                "PAYMENT_PENDING", payment.paymentIntentId(), priceChanges);

        } catch (Exception e) {
            metricsRegistry.recordLatency("checkout", System.currentTimeMillis() - startTime, false);
            log.error("Checkout saga failed for order {}: {}", order.getId(), e.getMessage());

            // Compensate
            compensate(order, e);
            throw e;
        }
    }

    /**
     * Compensation: reverse already-completed steps.
     */
    private void compensate(Order order, Exception cause) {
        updateSagaState(order, SagaState.COMPENSATING);
        log.warn("Compensating order {} due to: {}", order.getId(), cause.getMessage());

        try {
            // Release cart lock
            cartClient.releaseCartLock(order.getTenantId(), order.getUserId());

            // Release inventory reservation
            if (order.getSagaState() >= SagaState.RESERVING.ordinal()) {
                inventoryClient.releaseReservation(order.getTenantId(), order.getId(),
                    order.getIdempotencyKey());
            }

            // Void payment if initiated
            if (order.getSagaState() >= SagaState.PAYMENT_INITIATED.ordinal()) {
                paymentClient.voidPayment(order.getTenantId(), order.getId(),
                    order.getIdempotencyKey());
            }

            // Mark order cancelled
            order.setStatus(OrderStatus.CANCELLED);
            orderRepository.save(order);

            // Emit ORDER_CANCELLED
            kafkaTemplate.send("order.events", order.getId().toString(),
                Map.of("type", "ORDER_CANCELLED", "orderId", order.getId(),
                    "reason", cause.getMessage()));

            updateSagaState(order, SagaState.COMPENSATED);

        } catch (Exception compensationError) {
            log.error("Compensation failed for order {}: {}",
                order.getId(), compensationError.getMessage());

            // Retry via outbox/retry queue (DLQ)
            kafkaTemplate.send("saga.compensations.retry", order.getId().toString(),
                Map.of("orderId", order.getId(),
                    "compensationCause", cause.getMessage(),
                    "compensationError", compensationError.getMessage()));
        }
    }

    private Order createPendingOrder(CheckoutRequest request, Long tenantId, Long userId) {
        Order order = new Order();
        order.setTenantId(tenantId);
        order.setUserId(userId);
        order.setOrderNumber(generateOrderNumber(tenantId));
        order.setStatus(OrderStatus.PENDING);
        order.setCurrency(request.currency());
        order.setIdempotencyKey(request.idempotencyKey());
        order.setSagaState(SagaState.STARTED);
        return orderRepository.save(order);
    }

    private void publishOrderCreated(Order order, CartSnapshot cart) {
        // Transactional outbox — atomic with DB state
        kafkaTemplate.send("order.events",
            order.getId().toString(),
            new OrderCreatedEvent(order.getId(), order.getOrderNumber(),
                order.getTenantId(), order.getUserId(), cart, order.getTotal()));
    }
}
```

#### Spring State Machine Alternative (finite-state)

```java
package com.ecommerce.order.state;

public enum OrderStatus {
    PENDING(1),
    PAYMENT_PENDING(2),
    CONFIRMED(3),
    SHIPPED(4),
    CANCELLED(5),
    REFUNDED(6);

    private final int code;

    OrderStatus(int code) { this.code = code; }

    public int getCode() { return code; }
}
```

#### application.yml with Resilience4j

```yaml
spring:
  application:
    name: order-service
  datasource:
    url: jdbc:postgresql://pgbouncer:6432/ecommerce_orders
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP:localhost:9092}

server:
  port: 8084

resilience4j:
  circuitbreaker:
    instances:
      inventoryClient:
        slidingWindowSize: 20
        minimumNumberOfCalls: 5
        failureRateThreshold: 50
        waitDurationInOpenState: 20s
        permittedNumberOfCallsInHalfOpenState: 3
      paymentClient:
        slidingWindowSize: 20
        minimumNumberOfCalls: 5
        failureRateThreshold: 40
        waitDurationInOpenState: 15s
      cartClient:
        slidingWindowSize: 10
        failureRateThreshold: 60
  retry:
    instances:
      inventoryRetry:
        maxAttempts: 3
        waitDuration: 500ms
        retryExceptions:
          - java.net.SocketTimeoutException
          - com.ecommerce.common.core.BusinessException
  bulkhead:
    instances:
      checkoutBulkhead:
        maxConcurrentCalls: 100
        maxWaitDuration: 500ms
  ratelimiter:
    instances:
      checkoutRateLimiter:
        limitForPeriod: 1000
        limitRefreshPeriod: 1s
        timeoutDuration: 100ms

grpc:
  clients:
    inventory:
      address: inventory-service:8085
      negotiation-type: TLS
    payment:
      address: payment-service:8086
```

### 5.5 Inventory Service

**Purpose:** Stock reservation, ledger, Redis pre-reduction, oversell prevention.

#### build.gradle.kts

```kotlin
plugins {
    id("org.springframework.boot")
    id("io.spring.dependency-management")
}

dependencies {
    implementation(project(":common:common-core"))
    implementation(project(":common:common-redis"))
    implementation(project(":common:common-observability"))
    implementation(project(":common:common-persistence"))
    implementation(project(":common:common-kafka"))

    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.kafka:spring-kafka")
    implementation("org.redisson:redisson-spring-boot-starter:3.45.0")  // Redlock
    implementation("org.postgresql:postgresql")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")

    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")
}
```

#### InventoryService.java (Redlock + FOR UPDATE + Optimistic)

```java
package com.ecommerce.inventory.service;

@Service
@RequiredArgsConstructor
@Slf4j
public class InventoryService {

    private final InventoryRepository inventoryRepository;
    private final InventoryLedgerRepository ledgerRepository;
    private final RedissonClient redissonClient;
    private final StringRedisTemplate redisTemplate;
    private final MetricsRegistry metricsRegistry;

    private static final DefaultRedisScript<Long> PRE_DEDUCT_SCRIPT;
    static {
        PRE_DEDUCT_SCRIPT = new DefaultRedisScript<>();
        PRE_DEDUCT_SCRIPT.setScriptText("""
            local remaining = tonumber(redis.call('GET', KEYS[1]) or '-1')
            if remaining < 0 then return -1 end
            if remaining < tonumber(ARGV[1]) then return -2 end
            return redis.call('DECRBY', KEYS[1], ARGV[1])
            """);
        PRE_DEDUCT_SCRIPT.setResultType(Long.class);
    }

    /**
     * Reserve inventory with 4-layer oversell prevention:
     * 1. Redis pre-reduction (hot SKU fast path)
     * 2. Redlock distributed lock
     * 3. SELECT ... FOR UPDATE (authoritative)
     * 4. Optimistic version guard
     */
    @Transactional
    public ReservationResponse reserve(ReservationRequest request, Long tenantId) {
        long startTime = System.currentTimeMillis();

        // Layer 1: Redis pre-reduction (hot SKU)
        for (ReservationItem item : request.items()) {
            Long remaining = redisTemplate.execute(PRE_DEDUCT_SCRIPT,
                List.of("inv:hot:" + tenantId + ":" + item.sku()),
                String.valueOf(item.quantity()));
            if (remaining != null && remaining == -2) {
                throw new BusinessException(ErrorCode.INSUFFICIENT_STOCK,
                    "sku=" + item.sku());
            }
        }

        // Layer 2: Redlock (distributed lock per SKU)
        for (ReservationItem item : request.items()) {
            String lockKey = "inv:lock:" + tenantId + ":" + item.sku();
            RLock lock = redissonClient.getLock(lockKey);
            try {
                if (!lock.tryLock(2, 10, TimeUnit.SECONDS)) {
                    throw new BusinessException(ErrorCode.INTERNAL_ERROR, "Lock timeout");
                }
                // Layers 3+4: pessimistic row lock + optimistic version
                reserveInDb(request, tenantId, item);
            } finally {
                if (lock.isHeldByCurrentThread()) {
                    lock.unlock();
                }
            }
        }

        metricsRegistry.recordLatency("inventory.reserve",
            System.currentTimeMillis() - startTime, true);

        return new ReservationResponse(request.reservationId(), "RESERVED", request.items());
    }

    private void reserveInDb(ReservationRequest request, Long tenantId, ReservationItem item) {
        entityManager.createNativeQuery(
            "SET LOCAL lock_timeout = '2s'").executeUpdate();

        InventoryAvailable inventory = inventoryRepository
            .findForUpdate(tenantId, item.sku(), item.warehouseId())
            .orElseThrow(() -> new BusinessException(ErrorCode.INSUFFICIENT_STOCK,
                "sku=" + item.sku()));

        if (inventory.getAvailable() < item.quantity()) {
            throw new BusinessException(ErrorCode.INSUFFICIENT_STOCK,
                "sku=" + item.sku() +
                " requested=" + item.quantity() +
                " available=" + inventory.getAvailable());
        }

        // Optimistic guard
        int updated = inventoryRepository.updateReservation(
            tenantId, item.sku(), item.warehouseId(),
            item.quantity(), inventory.getVersion()
        );

        if (updated == 0) {
            throw new BusinessException(ErrorCode.INTERNAL_ERROR, "Version conflict");
        }

        // Append to ledger (idempotent via unique reference_id)
        InventoryLedger ledger = new InventoryLedger();
        ledger.setTenantId(tenantId);
        ledger.setSku(item.sku());
        ledger.setWarehouseId(item.warehouseId());
        ledger.setChangeType(ChangeType.RESERVE);
        ledger.setQuantity(-item.quantity());
        ledger.setOrderId(request.orderId());
        ledger.setReferenceId(request.reservationId());
        ledgerRepository.save(ledger);
    }

    /**
     * Release reservation (compensation, expiry sweeper, cancellation).
     */
    @Transactional
    public void release(Long tenantId, Long orderId, String referenceId) {
        List<InventoryLedger> reservations = ledgerRepository
            .findByReferenceId(referenceId);

        for (InventoryLedger res : reservations) {
            inventoryRepository.release(
                tenantId, res.getSku(), res.getWarehouseId(),
                -res.getQuantity()
            );

            InventoryLedger release = new InventoryLedger();
            release.setTenantId(tenantId);
            release.setSku(res.getSku());
            release.setWarehouseId(res.getWarehouseId());
            release.setChangeType(ChangeType.RELEASE);
            release.setQuantity(res.getQuantity());
            release.setOrderId(orderId);
            release.setReferenceId(referenceId + ":RELEASE:" + UUID.randomUUID());
            ledgerRepository.save(release);
        }
    }
}
```

### 5.6 Payment Service

**Purpose:** Payment state machine, multi-PSP routing, webhooks, refunds.

#### build.gradle.kts

```kotlin
plugins {
    id("org.springframework.boot")
    id("io.spring.dependency-management")
}

dependencies {
    implementation(project(":common:common-core"))
    implementation(project(":common:common-kafka"))
    implementation(project(":common:common-observability"))
    implementation(project(":common:common-persistence"))

    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.kafka:spring-kafka")
    implementation("org.postgresql:postgresql")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")
    implementation("com.stripe:stripe-java:28.5.0")
    implementation("com.adyen:adyen-java-api-library:21.0.0")

    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")
}
```

#### PaymentStateMachine.java

```java
package com.ecommerce.payment.state;

public enum PaymentStatus {
    PENDING(1, null),
    AUTHORIZED(2, PENDING),
    CAPTURED(3, AUTHORIZED),
    SETTLED(4, CAPTURED),
    FAILED(5, PENDING),
    VOIDED(6, AUTHORIZED),
    EXPIRED(7, PENDING);

    private final int code;
    private final PaymentStatus allowedFrom;

    PaymentStatus(int code, PaymentStatus allowedFrom) {
        this.code = code;
        this.allowedFrom = allowedFrom;
    }

    public boolean canTransitionFrom(PaymentStatus from) {
        if (from == null) {
            return this == PENDING;
        }
        return this.allowedFrom == from;
    }
}
```

```java
// PaymentService.java
package com.ecommerce.payment.service;

@Service
@RequiredArgsConstructor
@Slf4j
public class PaymentService {

    private final PaymentTransactionRepository transactionRepository;
    private final PspRouter pspRouter;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    /**
     * Initiate payment with PSP routing.
     * Primary PSP first; fallback to secondary on failure.
     */
    @Transactional
    public PaymentInitiation initiate(PaymentInitiationRequest request, Long tenantId) {
        PaymentTransaction tx = new PaymentTransaction();
        tx.setTenantId(tenantId);
        tx.setOrderId(request.orderId());
        tx.setUserId(request.userId());
        tx.setAmount(request.amount());
        tx.setCurrency(request.currency());
        tx.setIdempotencyKey(request.idempotencyKey());
        tx.setStatus(PaymentStatus.PENDING);
        transactionRepository.save(tx);

        // Try primary PSP, fallback to secondary
        PspResult result = pspRouter.initiateWithFallback(tx, request.paymentMethod());

        if (result.success()) {
            tx.setPsp(result.psp());
            tx.setPspTransactionId(result.transactionId());
            tx.setStatus(PaymentStatus.PENDING);
            transactionRepository.save(tx);

            return new PaymentInitiation(tx.getId(), result.transactionId(), "PENDING");
        }

        tx.setStatus(PaymentStatus.FAILED);
        transactionRepository.save(tx);
        throw new BusinessException(ErrorCode.PAYMENT_FAILED, result.errorMessage());
    }

    /**
     * Handle webhook (idempotent via psp_event_id).
     */
    @Transactional
    public void handleWebhook(String psp, String eventId, WebhookPayload payload, Long tenantId) {
        // Dedupe — unique (psp, psp_event_id)
        boolean exists = transactionRepository.existsByPspAndPspEventId(psp, eventId);
        if (exists) {
            log.info("Duplicate webhook ignored: psp={}, event={}", psp, eventId);
            return;
        }

        PaymentTransaction tx = transactionRepository
            .findByPspTransactionId(payload.transactionId())
            .orElseThrow(() -> new IllegalArgumentException("Transaction not found"));

        PaymentStatus newStatus = mapWebhookToStatus(payload.eventType());

        // State machine validation
        if (!newStatus.canTransitionFrom(tx.getStatus())) {
            log.warn("Illegal state transition for tx {}: {} → {}",
                tx.getId(), tx.getStatus(), newStatus);
            return;
        }

        tx.setStatus(newStatus);
        tx.setPspEventId(eventId);
        transactionRepository.save(tx);

        // Publish event
        kafkaTemplate.send("payment.events", tx.getOrderId().toString(),
            new PaymentEvent(tx.getId(), tx.getOrderId(), newStatus.name()));

        // If captured → emit order confirmation event
        if (newStatus == PaymentStatus.CAPTURED) {
            kafkaTemplate.send("order.events", tx.getOrderId().toString(),
                Map.of("type", "ORDER_PAYMENT_CONFIRMED", "orderId", tx.getOrderId()));
        }

        // If failed → compensate (release inventory)
        if (newStatus == PaymentStatus.FAILED) {
            kafkaTemplate.send("inventory.events", tx.getOrderId().toString(),
                Map.of("type", "PAYMENT_FAILED_RELEASE", "orderId", tx.getOrderId(),
                    "idempotencyKey", tx.getIdempotencyKey()));
        }
    }
}
```

### 5.7 Notification Service

**Purpose:** Async email/SMS/push notifications with retry and dedupe.

#### build.gradle.kts

```kotlin
plugins {
    id("org.springframework.boot")
    id("io.spring.dependency-management")
}

dependencies {
    implementation(project(":common:common-core"))
    implementation(project(":common:common-observability"))

    implementation("org.springframework.boot:spring-boot-starter")
    implementation("org.springframework.kafka:spring-kafka")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("com.sendgrid:sendgrid-java:4.10.2")
    implementation("com.twilio.sdk:twilio:10.6.6")
    implementation("com.fasterxml.jackson.core:jackson-databind")
    implementation("org.postgresql:postgresql")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")

    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")
}
```

#### NotificationConsumer.java

```java
package com.ecommerce.notification.consumer;

@Component
@RequiredArgsConstructor
@Slf4j
public class NotificationConsumer {

    private final NotificationService notificationService;
    private final NotificationTemplateRepository templateRepository;

    @KafkaListener(topics = "notification.events", groupId = "notification-service",
        concurrency = "10")
    public void consumeOrderEvent(NotificationEvent event) {
        try {
            // 1. Build notification from template
            NotificationTemplate template = templateRepository
                .findByTypeAndLocale(event.type(), event.locale())
                .orElseThrow(() -> new IllegalArgumentException("Template not found"));

            Map<String, Object> content = template.render(event.payload());

            // 2. Deliver based on preferences
            NotificationPreference prefs = event.preferences();

            if (prefs.email()) {
                notificationService.sendEmail(event.tenantId(), event.userId(), content);
            }
            if (prefs.sms()) {
                notificationService.sendSms(event.tenantId(), event.userId(), content);
            }
            if (prefs.push()) {
                notificationService.sendPush(event.tenantId(), event.userId(), content);
            }

            // 3. Record delivery (dedupe)
            notificationService.recordDelivery(event.notificationId());

        } catch (Exception e) {
            log.error("Failed to process notification {}: {}", event.notificationId(), e.getMessage());
            // Retry via DLQ
            throw new RetryableException(e);  // config: maxAttempts=3, backoff
        }
    }
}
```

```yaml
# application.yml
spring:
  application:
    name: notification-service
  kafka:
    consumer:
      group-id: notification-service
      auto-offset-reset: latest
      enable-auto-commit: false
    listener:
      ack-mode: manual_immediate
      concurrency: 10
      retry:
        max-attempts: 3
        backoff: 1000ms

ecommerce:
  notification:
    dlq-topic: notification.events.dlq
    email:
      provider: sendgrid
      from: no-reply@ecommerce.com
    sms:
      provider: twilio
    push:
      provider: fcm
```

### 5.8 Flash Sale Queue (FLQ) Service

**Purpose:** Absorb 100K RPS bursts via Redis queue with atomic capacity check.

#### build.gradle.kts

```kotlin
plugins {
    id("org.springframework.boot")
    id("io.spring.dependency-management")
}

dependencies {
    implementation(project(":common:common-core"))
    implementation(project(":common:common-redis"))
    implementation(project(":common:common-observability"))
    implementation(project(":common:common-kafka"))

    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.kafka:spring-kafka")

    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")
}
```

#### FlqService.java

```java
package com.ecommerce.flq.service;

@Service
@RequiredArgsConstructor
@Slf4j
public class FlqService {

    private final StringRedisTemplate redisTemplate;
    private final DefaultRedisScript<Long> flqEnqueueScript;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final MetricsRegistry metricsRegistry;

    private static final long QUEUE_MAX_LENGTH = 100_000;

    /**
     * Atomically enqueue flash-sale request:
     * - Capacity check (per SKU)
     * - Duplicate prevention (per user per deal)
     * - FIFO enqueue
     */
    public FlqResponse enqueue(FlqEnqueueRequest request, Long tenantId) {
        String capacityKey = "flash:sale:" + request.dealId() + ":" + request.sku() + ":capacity";
        String queueKey = "flash:sale:" + request.dealId() + ":" + request.sku() + ":queue";
        String processedKey = "flash:sale:" + request.dealId() + ":" + request.sku() + ":processed";

        // Check queue length (backpressure)
        Long queueLength = redisTemplate.opsForList().size(queueKey);
        if (queueLength != null && queueLength >= QUEUE_MAX_LENGTH) {
            metricsRegistry.incrementCounter("flq.backpressure");
            throw new BusinessException(ErrorCode.QUEUE_FULL, "Queue full");
        }

        // Atomic enqueue
        Long position = redisTemplate.execute(flqEnqueueScript,
            List.of(capacityKey, queueKey, processedKey),
            String.valueOf(request.userId()),
            request.sku(),
            String.valueOf(request.quantity()));

        if (position != null && position == -1) {
            metricsRegistry.incrementCounter("flq.sold_out");
            throw new BusinessException(ErrorCode.INSUFFICIENT_STOCK, "Sold out");
        }
        if (position != null && position == -2) {
            metricsRegistry.incrementCounter("flq.duplicate");
            throw new BusinessException(ErrorCode.INVALID_REQUEST, "Duplicate request");
        }

        metricsRegistry.incrementCounter("flq.enqueued");
        return new FlqResponse(position, "QUEUED");
    }

    /**
     * Drainer — consumes from Redis queue and forwards to Kafka for async processing.
     * Autoscaled based on Kafka consumer lag.
     */
    @Scheduled(fixedDelay = 100)
    public void drainQueue() {
        List<String> queueKeys = redisTemplate.keys("flash:sale:*:*:queue");
        if (queueKeys == null || queueKeys.isEmpty()) return;

        for (String queueKey : queueKeys) {
            // Batch pop (up to 100 per drain cycle)
            List<String> batch = redisTemplate.opsForList()
                .rightPop(queueKey, 100);

            for (String entry : batch) {
                String[] parts = entry.split(":");
                String userId = parts[0];
                String sku = parts[1];
                int qty = Integer.parseInt(parts[2]);

                // Forward to Kafka for Order Service consumption
                kafkaTemplate.send("flq.drain", sku,
                    Map.of("userId", Long.parseLong(userId), "sku", sku, "qty", qty));
            }

            metricsRegistry.incrementCounter("flq.drained", "batch", String.valueOf(batch.size()));
        }
    }
}
```

### 5.9 Recommendation Service

**Purpose:** Personalized recommendations, trending, "customers also viewed".

#### build.gradle.kts

```kotlin
plugins {
    id("org.springframework.boot")
    id("io.spring.dependency-management")
}

dependencies {
    implementation(project(":common:common-core"))
    implementation(project(":common:common-observability"))

    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.kafka:spring-kafka")
    implementation("dev.morphia.morphia:morphia-core:2.4.14")
    implementation("com.clickhouse:clickhouse-jdbc:0.8.2")

    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")
}
```

#### RecommendationService.java

```java
package com.ecommerce.recommendation.service;

@Service
@RequiredArgsConstructor
public class RecommendationService {

    private final StringRedisTemplate redisTemplate;
    private final ClickHouseDataSource clickHouse;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    /**
     * Get personalized recommendations:
     * 1. Redis cache (per user, TTL 5 min)
     * 2. Collaborative filtering (from ClickHouse co-occurrence)
     * 3. Content-based fallback (same category/brand)
     */
    public RecommendationsResponse getRecommendations(Long tenantId, Long userId,
                                                       String type, int limit) {
        String cacheKey = "rec:" + tenantId + ":" + userId + ":" + type;

        // Cache check
        String cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return JsonUtils.fromJson(cached, RecommendationsResponse.class);
        }

        List<Long> productIds = switch (type) {
            case "related" -> getRelatedProducts(tenantId, userId, limit);
            case "frequently_bought" -> getFrequentlyBought(tenantId, userId, limit);
            case "trending" -> getTrending(tenantId, limit);
            default -> getPersonalized(tenantId, userId, limit);
        };

        RecommendationsResponse response = new RecommendationsResponse(productIds);

        // Cache with TTL jitter
        long ttl = 300 + ThreadLocalRandom.current().nextLong(0, 60);
        redisTemplate.opsForValue().set(cacheKey, JsonUtils.toJson(response),
            Duration.ofSeconds(ttl));

        return response;
    }

    private List<Long> getFrequentlyBought(Long tenantId, Long userId, int limit) {
        // Query ClickHouse co-occurrence matrix
        String sql = """
            SELECT product_b, COUNT(*) as cnt
            FROM order_cooccurrence
            WHERE tenant_id = ? AND product_a IN (
                SELECT product_id FROM user_orders WHERE user_id = ? AND tenant_id = ?
            )
            GROUP BY product_b
            ORDER BY cnt DESC
            LIMIT ?
            """;

        try (var conn = clickHouse.getConnection();
             var stmt = conn.prepareStatement(sql)) {
            stmt.setLong(1, tenantId);
            stmt.setLong(2, userId);
            stmt.setLong(3, tenantId);
            stmt.setInt(4, limit);
            var rs = stmt.executeQuery();

            List<Long> products = new ArrayList<>();
            while (rs.next()) {
                products.add(rs.getLong("product_b"));
            }
            return products;
        } catch (SQLException e) {
            log.error("Failed to query recommendations", e);
            return List.of();
        }
    }

    /**
     * Consume user behavior event for real-time personalization.
     */
    @KafkaListener(topics = "analytics.events", groupId = "recommendation-service")
    public void consumeBehaviorEvent(BehaviorEvent event) {
        // Update user behavior profile in Redis
        String key = "user:profile:" + event.tenantId() + ":" + event.userId();
        redisTemplate.opsForZSet().incrementScore(
            key + ":viewed", event.productId().toString(), 1.0);
        redisTemplate.opsForZSet().incrementScore(
            key + ":purchased", event.productId().toString(), 5.0);

        // Invalidate recommendations cache
        redisTemplate.delete("rec:" + event.tenantId() + ":" + event.userId() + ":*");
    }
}
```

### 5.10 Rate Limit Service

**Purpose:** Atomic token bucket rate limiting per tenant, user, IP, endpoint.

#### build.gradle.kts

```kotlin
plugins {
    id("org.springframework.boot")
    id("io.spring.dependency-management")
}

dependencies {
    implementation(project(":common:common-core"))
    implementation(project(":common:common-redis"))
    implementation(project(":common:common-observability"))

    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")

    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")
}
```

#### RateLimitService.java

```java
package com.ecommerce.ratelimit.service;

@Service
@RequiredArgsConstructor
public class RateLimitService {

    private final StringRedisTemplate redisTemplate;
    private final DefaultRedisScript<Long> tokenBucketScript;

    /**
     * Check rate limit using atomic token bucket.
     * Returns true if allowed, false if rate limited.
     */
    public boolean check(LimitKey key, long nowEpochMillis) {
        String redisKey = "rate_limit:" +
            (key.tenantId() != null ? key.tenantId() : "anon") + ":" +
            (key.userId() != null ? key.userId() : "anon") + ":" +
            (key.endpoint() != null ? key.endpoint() : "all");

        Long result = redisTemplate.execute(
            tokenBucketScript,
            List.of(redisKey),
            String.valueOf(key.capacity()),
            String.valueOf(key.refillRatePerSecond()),
            String.valueOf(nowEpochMillis)
        );

        return result != null && result == 1;
    }

    public record LimitKey(
        Long tenantId,       // per-tenant: 10K RPS
        Long userId,         // per-user: 100 RPS
        String endpoint,     // per-endpoint: checkout 1K RPS
        String ip,           // per-IP: 50 RPS
        int capacity,
        double refillRatePerSecond
    ) {}
}
```

### 5.11 Feature Flag Service

**Purpose:** Feature flags, percentage rollouts, A/B testing assignment.

#### build.gradle.kts

```kotlin
plugins {
    id("org.springframework.boot")
    id("io.spring.dependency-management")
}

dependencies {
    implementation(project(":common:common-core"))
    implementation(project(":common:common-observability"))

    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.kafka:spring-kafka")
    implementation("org.postgresql:postgresql")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")

    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")
}
```

#### FeatureFlagService.java

```java
package com.ecommerce.featureflag.service;

@Service
@RequiredArgsConstructor
public class FeatureFlagService {

    private final StringRedisTemplate redisTemplate;
    private final FeatureFlagRepository flagRepository;

    private LoadingCache<String, FeatureFlag> localCache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofSeconds(5))
        .build(this::loadFlagFromRedis);

    /**
     * Fast path: local Caffeine cache (sub-ms) with Redis fallback.
     */
    public boolean isEnabled(String flagName, Long tenantId, Long userId) {
        FeatureFlag flag = localCache.get(flagName);

        if (flag == null) {
            return false;  // Default safe (feature off)
        }

        // Global kill switch
        if (flag.isKillSwitch()) return false;

        // Percentage rollout
        if (flag.isPercentage()) {
            int hash = Math.abs((tenantId + ":" + userId)
                .hashCode()) % 100;
            return hash < flag.getPercentage();
        }

        // Per-tenant
        if (flag.isTenantScoped()) {
            Set<Long> enabledTenants = flag.getEnabledTenants();
            return enabledTenants.contains(tenantId);
        }

        return flag.isEnabled();
    }

    /**
     * A/B testing assignment: deterministic hash → variant.
     */
    public String assignVariant(Long tenantId, Long userId, String experimentId) {
        int hash = Math.abs((experimentId + ":" + tenantId + ":" + userId)
            .hashCode()) % 100;

        Experiment experiment = experimentRepository.findByKey(experimentId);
        for (Variant v : experiment.getVariants()) {
            if (hash < v.weightPercentage()) {
                return v.name();
            }
        }
        return experiment.getControlVariant();
    }

    private FeatureFlag loadFlagFromRedis(String flagName) {
        String value = redisTemplate.opsForValue().get("flag:" + flagName);
        if (value != null) {
            return JsonUtils.fromJson(value, FeatureFlag.class);
        }
        try {
            return flagRepository.findByName(flagName)
                .orElse(null);
        } catch (Exception e) {
            return null;  // Redis AND DB down → safe default
        }
    }
}
```

---

## 6. INTER-SERVICE COMMUNICATION

### 6.1 HTTP REST (Edge → Internal)

```java
// RestClientConfig.java
package com.ecommerce.common;

@Configuration
public class RestClientConfig {

    @Bean
    public RestClient inventoryClient(RestClient.Builder builder,
                                       @Value("${services.inventory.url}") String url) {
        return builder
            .baseUrl(url)
            .defaultHeader("Content-Type", "application/json")
            .requestFactory(ClientHttpRequestFactorySettings.custom()
                .withConnectTimeout(Duration.ofMillis(500))
                .withReadTimeout(Duration.ofSeconds(2))
                .build())
            .build();
    }
}
```

### 6.2 gRPC (Internal, High-Performance)

```proto
// inventory.proto
syntax = "proto3";

package ecommerce.inventory;

service InventoryService {
    rpc Reserve(ReserveRequest) returns (ReserveResponse);
    rpc Release(ReleaseRequest) returns (ReleaseResponse);
    rpc GetStock(GetStockRequest) returns (GetStockResponse);
}

message ReserveRequest {
    int64 tenant_id = 1;
    int64 order_id = 2;
    string reservation_id = 3;
    repeated ReservationItem items = 4;
}

message ReservationItem {
    string sku = 1;
    int64 warehouse_id = 2;
    int32 quantity = 3;
}

message ReserveResponse {
    string reservation_id = 1;
    string status = 2;
    repeated Allocation allocations = 3;
}

message Allocation {
    string sku = 1;
    int64 warehouse_id = 2;
    int32 quantity = 3;
}
```

```java
// InventoryGrpcClient.java
package com.ecommerce.order.client;

@GrpcClient("inventory")
@RequiredArgsConstructor
public class InventoryGrpcClient {

    private final InventoryServiceGrpc.InventoryServiceBlockingStub inventoryStub;

    public ReservationResponse reserve(Long tenantId, Long orderId, String reservationId,
                                       List<ReservationItemDto> items) {
        ReserveRequest request = ReserveRequest.newBuilder()
            .setTenantId(tenantId)
            .setOrderId(orderId)
            .setReservationId(reservationId)
            .addAllItems(items.stream()
                .map(i -> ReservationItem.newBuilder()
                    .setSku(i.sku())
                    .setWarehouseId(i.warehouseId())
                    .setQuantity(i.quantity())
                    .build())
                .toList())
            .build();

        try {
            ReserveResponse response = inventoryStub.reserve(request);
            return mapToDto(response);
        } catch (StatusRuntimeException e) {
            if (e.getStatus().getCode() == Status.Code.RESOURCE_EXHAUSTED) {
                throw new BusinessException(ErrorCode.INSUFFICIENT_STOCK);
            }
            throw e;
        }
    }
}
```

### 6.3 Kafka (Async Events)

```java
// EventPublisher.java
package com.ecommerce.common.kafka;

@Component
@RequiredArgsConstructor
public class EventPublisher {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    public void publish(Event event) {
        String topic = switch (event.getType()) {
            case ORDER_CREATED, ORDER_CONFIRMED, ORDER_CANCELLED, ORDER_PAYMENT_CONFIRMED ->
                "order.events";
            case INVENTORY_RESERVED, INVENTORY_RELEASED, INVENTORY_RESTOCKED ->
                "inventory.events";
            case PAYMENT_INITIATED, PAYMENT_CAPTURED, PAYMENT_FAILED, PAYMENT_REFUNDED ->
                "payment.events";
            case CART_UPDATED, CART_MERGED, CART_EXPIRED -> "cart.events";
            case NOTIFICATION_REQUESTED -> "notification.events";
            case PRODUCT_VIEWED, ORDER_COMPLETED, SEARCH_QUERIED -> "analytics.events";
            case FLQ_DEQUEUED -> "flq.drain";
            default -> throw new IllegalArgumentException("Unknown event type: " + event.getType());
        };

        kafkaTemplate.send(topic, event.getAggregateId().toString(), event);
    }
}
```

---

## 7. RESILIENCE & FAULT TOLERANCE

### 7.1 Resilience4j Configuration

```java
// ResilienceConfig.java
package com.ecommerce.common.resilience;

@Configuration
public class ResilienceConfig {

    @Bean
    public CircuitBreaker inventoryCircuitBreaker(
        CircuitBreakerRegistry circuitBreakerRegistry) {
        return circuitBreakerRegistry.circuitBreaker("inventoryClient");
    }

    @Bean
    public Bulkhead checkoutBulkhead(BulkheadRegistry bulkheadRegistry) {
        return bulkheadRegistry.bulkhead("checkoutBulkhead");
    }

    @Bean
    public TimeLimiter checkoutTimeLimiter(TimeLimiterRegistry timeLimiterRegistry) {
        return timeLimiterRegistry.timeLimiter("checkoutTimeLimiter");
    }
}
```

```yaml
# application.yml — resilience4j defaults
resilience4j:
  circuitbreaker:
    configs:
      default:
        slidingWindowSize: 100
        minimumNumberOfCalls: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 20s
        permittedNumberOfCallsInHalfOpenState: 5
        recordExceptions:
          - java.io.IOException
          - java.net.SocketTimeoutException
          - com.ecommerce.common.core.BusinessException
        ignoreExceptions:
          - com.ecommerce.common.core.BusinessException  # Business errors don't open circuit
  retry:
    configs:
      default:
        maxAttempts: 3
        waitDuration: 500ms
        retryExceptions:
          - java.net.SocketTimeoutException
          - org.springframework.dao.DeadlockLoserDataAccessException
  bulkhead:
    configs:
      default:
        maxConcurrentCalls: 50
        maxWaitDuration: 500ms
  ratelimiter:
    configs:
      default:
        limitForPeriod: 1000
        limitRefreshPeriod: 1s
        timeoutDuration: 100ms
```

### 7.2 Distributed Lock (Redisson Redlock)

```java
// RedlockConfig.java
package com.ecommerce.common.redis;

@Configuration
public class RedlockConfig {

    @Bean(destroyMethod = "shutdown")
    public RedissonClient redissonClient(RedisProperties properties) {
        Config config = new Config();
        config.useClusterServers()
            .addNodeAddress(properties.getCluster().getNodes().toArray(String[]::new))
            .setMasterConnectionPoolSize(50)
            .setSlaveConnectionPoolSize(50)
            .setTimeout(3000);

        return Redisson.create(config);
    }
}
```

### 7.3 Timeout & Deadline Propagation

```java
// TimeoutConfig.java
package com.ecommerce.common;

@Configuration
public class TimeoutConfig {

    @Bean
    public TaskExecutor serviceTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(50);
        executor.setMaxPoolSize(200);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("svc-");
        executor.initialize();
        return executor;
    }
}
```

---

## 8. OBSERVABILITY & MONITORING

### 8.1 OpenTelemetry Setup

```java
// ObservabilityConfig.java
package com.ecommerce.common.observability;

@Configuration
public class ObservabilityConfig {

    @Bean
    public MeterRegistry meterRegistry() {
        return new SimpleMeterRegistry();
    }

    @Bean
    public MetricsRegistry metricsRegistry(MeterRegistry meterRegistry) {
        return new MetricsRegistry(meterRegistry);
    }

    @Bean
    public GlobalOpenTelemetry openTelemetry(@Value("${spring.application.name}") String serviceName) {
        return GlobalOpenTelemetry.builder()
            .setMeterProvider(
                SdkMeterProvider.builder()
                    .setResource(Resource.create(Attributes.of(
                        AttributeKey.stringKey("service.name"), serviceName)))
                    .build())
            .setTracerProvider(
                SdkTracerProvider.builder()
                    .setResource(Resource.create(Attributes.of(
                        AttributeKey.stringKey("service.name"), serviceName)))
                    .addSpanProcessor(BatchSpanProcessor.builder(
                        OtlpGrpcSpanExporter.builder()
                            .setEndpoint("http://tempo:4317")
                            .build()).build())
                    .build())
            .build();
    }
}
```

### 8.2 Custom Span via Micrometer

```java
package com.ecommerce.common.observability;

@Service
@RequiredArgsConstructor
public class TracingService {

    private final Tracer tracer;

    public <T> T traceSpan(String operationName, Supplier<T> operation) {
        Span span = tracer.spanBuilder(operationName).startSpan();
        try (Scope scope = span.makeCurrent()) {
            return operation.get();
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### 8.3 Logging with Correlation IDs

```yaml
# logback-spring.xml (metrics + correlation IDs)
<configuration>
    <springProperty scope="context" name="appName" source="spring.application.name"/>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"app":"${appName}"}</customFields>
        </encoder>
    </appender>

    <appender name="METRICS" class="io.micrometer.core.instrument.binder.logging.MetricsAwareLogbackAppender"/>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
    </root>
</configuration>
```

---

## 9. SECURITY

### 9.1 Spring Security Configuration

```java
// SecurityConfig.java
package com.ecommerce.common.security;

@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtFilter;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/**", "/api-docs/**", "/swagger-ui/**").permitAll()
                .requestMatchers("/api/v1/auth/**").permitAll()
                .anyRequest().authenticated())
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
}
```

### 9.2 Tenant Isolation

```java
package com.ecommerce.common.security;

@Aspect
@Component
@RequiredArgsConstructor
public class TenantIsolationAspect {

    private final TenantContext tenantContext;

    @Around("@annotation(TenantScoped)")
    public Object enforceTenantScope(ProceedingJoinPoint pjp) throws Throwable {
        Long tenantId = TenantContext.getTenantId();
        if (tenantId == null) {
            throw new BusinessException(ErrorCode.UNAUTHORIZED, "Missing tenant context");
        }
        return pjp.proceed();
    }
}
```

### 9.3 gRPC mTLS

```yaml
# application.yml
grpc:
  server:
    port: 8085-grpc
    security:
      enabled: true
      certificate-chain: classpath:cert/server.crt
      private-key: classpath:cert/server.key
      trust-cert-collection: classpath:cert/ca.crt
```

---

## 10. KAFKA TOPICS & EVENT CONTRACTS

### 10.1 Topic Inventory

| Topic | Partitions | RF | Retention | Producer | Consumers |
|---|---|---|---|---|---|
| `order.events` | 24 | 3 | 7 days | Order, Payment | Notification, Analytics, Recommendation |
| `inventory.events` | 24 | 3 | 7 days | Inventory | Order (compensation), Search indexer |
| `payment.events` | 12 | 3 | 7 days | Payment | Order, Notification, Analytics |
| `cart.events` | 12 | 3 | 7 days | Cart | Analytics |
| `notification.events` | 12 | 3 | 7 days | Order, Payment | Notification Service |
| `analytics.events` | 24 | 3 | 30 days | All services | ClickHouse, Recommendation |
| `flq.drain` | 12 | 3 | 7 days | FLQ Service | Order Service |
| `saga.compensations.retry` | 12 | 3 | 7 days | Order | Order (retry) |
| `*.dlq` | 6 | 3 | 30 days | All | Ops (manual replay) |

### 10.2 Event Contracts (JSON)

```json
// ORDER_CREATED
{
  "type": "ORDER_CREATED",
  "eventId": "uuid",
  "timestamp": "2025-01-01T00:00:00Z",
  "aggregateId": "1001",
  "tenantId": 1,
  "payload": {
    "orderId": 1001,
    "orderNumber": "ORD-2025-000001",
    "userId": 2001,
    "items": [
      { "sku": "PROD-123", "qty": 2, "price": 49.99 }
    ],
    "total": 99.98,
    "currency": "USD"
  }
}

// ORDER_PAYMENT_CONFIRMED
{
  "type": "ORDER_PAYMENT_CONFIRMED",
  "eventId": "uuid",
  "timestamp": "2025-01-01T00:00:05Z",
  "aggregateId": "1001",
  "tenantId": 1,
  "payload": {
    "orderId": 1001,
    "paymentTransactionId": 5001,
    "paymentStatus": "CAPTURED"
  }
}

// INVENTORY_RESERVED
{
  "type": "INVENTORY_RESERVED",
  "eventId": "uuid",
  "timestamp": "2025-01-01T00:00:01Z",
  "aggregateId": "1001",
  "tenantId": 1,
  "payload": {
    "orderId": 1001,
    "reservationId": "res-001",
    "allocations": [
      { "sku": "PROD-123", "warehouseId": 10, "qty": 2 }
    ]
  }
}

// PAYMENT_FAILED
{
  "type": "PAYMENT_FAILED",
  "eventId": "uuid",
  "timestamp": "2025-01-01T00:00:10Z",
  "aggregateId": "1001",
  "tenantId": 1,
  "payload": {
    "orderId": 1001,
    "paymentTransactionId": 5001,
    "reason": "INSUFFICIENT_FUNDS",
    "idempotencyKey": "key-001"
  }
}
```

---

## 11. DEPLOYMENT & DEVOPS

### 11.1 Kubernetes Manifest (Order Service Example)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: ecommerce
spec:
  replicas: 50
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0%   # Zero downtime
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: ecommerce/order-service:1.0.0
          ports:
            - containerPort: 8084
          env:
            - name: DB_URL
              valueFrom:
                secretKeyRef:
                  name: ecommerce-secrets
                  key: order-db-url
            - name: KAFKA_BOOTSTRAP
              value: kafka:9092
            - name: JVM_OPTS
              value: "-Xmx4g -XX:MaxRAMPercentage=75 -XX:+UseZGC -Xlog:gc"
          resources:
            requests:
              cpu: "2"
              memory: "4Gi"
            limits:
              cpu: "4"
              memory: "8Gi"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8084
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8084
          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8084
            failureThreshold: 30
            periodSeconds: 5
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 10"]  # Drain in-flight requests
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: ecommerce
spec:
  selector:
    app: order-service
  ports:
    - port: 8084
      targetPort: 8084
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: ecommerce
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 50
  maxReplicas: 150
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

### 11.2 CI/CD Pipeline (GitHub Actions)

```yaml
name: Build & Deploy
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
      - name: Build with Gradle
        run: ./gradlew build -x test
      - name: Test
        run: ./gradlew test
      - name: Publish Docker image
        run: ./gradlew bootBuildImage
          --imageName=registry.ecommerce.com/${{ matrix.service }}:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Canary deploy 1%
        run: kubectl -n ecommerce set image deployment/$SERVICE $SERVICE=registry.ecommerce.com/$SERVICE:$GITHUB_SHA --record
        env:
          SERVICE: order-service
      - name: Wait for rollout
        run: kubectl -n ecommerce rollout status deployment/$SERVICE --timeout=300s
      - name: Gradual rollout 10% → 50% → 100%
        run: |
          kubectl -n ecommerce scale deployment/$SERVICE --replicas=$(kubectl -n ecommerce get hpa $SERVICE -o jsonpath='{.spec.minReplicas}')
```

### 11.3 Docker Compose (Local Dev)

```yaml
version: "3.9"
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ecommerce
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes

  kafka:
    image: bitnami/kafka:3.9
    ports:
      - "9092:9092"
    environment:
      KAFKA_CFG_NODE_ID: 1
      KAFKA_CFG_PROCESS_ROLES: broker,controller
      KAFKA_CFG_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_CFG_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.16.0
    environment:
      discovery.type: single-node
      xpack.security.enabled: "false"
    ports:
      - "9200:9200"

  pgbouncer:
    image: edoburu/pgbouncer:latest
    environment:
      DB_HOST: postgres
      DB_PORT: "5432"
      DB_USER: dev
      DB_PASSWORD: dev
      POOL_MODE: transaction
      MAX_CLIENT_CONN: "1000"
    ports:
      - "6432:6432"
    depends_on:
      - postgres

  # Services
  auth-service:
    build: ./services/auth-service
    ports: ["8081:8081"]
    environment:
      DB_URL: jdbc:postgresql://pgbouncer:6432/ecommerce_auth
      KAFKA_BOOTSTRAP: kafka:9092
    depends_on: [postgres, redis, kafka]

  catalog-service:
    build: ./services/catalog-service
    ports: ["8082:8082"]
    environment:
      DB_URL: jdbc:postgresql://pgbouncer:6432/ecommerce_catalog
      ES_URIS: http://elasticsearch:9200
    depends_on: [postgres, elasticsearch, redis]
```

---

## 12. LOCAL DEVELOPMENT SETUP

### Commands

```bash
# 1. Clone repo
git clone git@github.com:ecommerce/platform.git
cd platform

# 2. Start infrastructure (Postgres, Redis, Kafka, ES, PgBouncer)
docker-compose up -d postgres redis kafka elasticsearch pgbouncer

# 3. Build all modules
./gradlew build

# 4. Run a specific service
./gradlew :services:order-service:bootRun

# 5. Run with local config override
SPRING_PROFILES_ACTIVE=dev \
DB_USER=dev DB_PASSWORD=dev \
JWT_SECRET=dev-secret \
./gradlew :services:auth-service:bootRun

# 6. Run all tests (unit + integration)
./gradlew test

# 7. Run with Testcontainers (integration tests spin up real deps)
./gradlew :services:inventory-service:test --tests "*IntegrationTest*"
```

### Build & Verify

```bash
# Build Docker image
./gradlew :services:order-service:bootBuildImage \
  --imageName=ecommerce/order-service:latest

# Push
docker push ecommerce/order-service:latest

# Deploy to k8s
kubectl apply -f infrastructure/k8s/order-service.yaml
```

### Swagger/OpenAPI

```bash
# Each service exposes Swagger UI:
# http://localhost:8081/swagger-ui.html  (Auth)
# http://localhost:8082/swagger-ui.html  (Catalog)
# http://localhost:8084/swagger-ui.html  (Order)
# http://localhost:8085/swagger-ui.html  (Inventory)

# OpenAPI JSON:
curl http://localhost:8084/v3/api-docs | jq '.paths' | head
```

### Service Health Checks

```bash
# Actuator endpoints per service
curl http://localhost:8084/actuator/health          # Overall
curl http://localhost:8084/actuator/health/liveness  # Liveness
curl http://localhost:8084/actuator/health/readiness # Readiness
curl http://localhost:8084/actuator/prometheus      # Metrics (Prometheus)

# Check circuit breaker state
curl http://localhost:8084/actuator/circuitbreakers | jq '.circuitBreakers'
```

---

## APPENDIX A: Dependencies Matrix (Per Service)

| Dependency | Auth | Catalog | Cart | Order | Inventory | Payment | Notification | FLQ | Rec | RateLimit | FeatureFlag |
|---|---|---|---|---|---|---|---|---|---|---|---|
| spring-boot-starter-web | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — | ✅ | ✅ | ✅ | ✅ |
| spring-boot-starter-webflux | — | — | — | ✅ | — | — | — | — | — | — | — |
| spring-boot-starter-data-jpa | ✅ | ✅ | — | ✅ | ✅ | ✅ | ✅ | — | — | — | ✅ |
| spring-boot-starter-data-redis | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — | ✅ | ✅ | ✅ | ✅ |
| spring-boot-starter-data-elasticsearch | — | ✅ | — | — | — | — | — | — | — | — | — |
| spring-boot-starter-security | ✅ | ✅ | — | ✅ | — | ✅ | — | — | — | — | — |
| spring-kafka | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — | ✅ |
| resilience4j-spring-boot3 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — | — | — |
| flyway | ✅ | ✅ | — | ✅ | ✅ | ✅ | ✅ | — | — | — | ✅ |
| springdoc-openapi | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — | ✅ | ✅ | ✅ | ✅ |
| grpc-spring-boot-starter | — | ✅ | — | ✅ | — | — | — | — | — | — | — |
| redisson (Redlock) | — | — | — | — | ✅ | — | — | — | — | — | — |
| stripe-java / adyen | — | — | — | — | — | ✅ | — | — | — | — | — |
| sendgrid / twilio | — | — | — | — | — | — | ✅ | — | — | — | — |
| jackson | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| lombok | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| testcontainers | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — | ✅ |

---

## APPENDIX B: Console Commands

```bash
# Watch Kafka topics
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic order.events --from-beginning

# Monitor Redis
redis-cli -h localhost monitor

# Check PgBouncer pool
psql -h localhost -p 6432 -U dev -c "SHOW POOLS;"

# Scale a service
kubectl -n ecommerce scale deployment/inventory-service --replicas=120

# Check Kafka consumer lag
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group order-service --describe

# Run chaos experiment (kill Redis pod)
kubectl -n ecommerce delete pod redis-cluster-node-0 --grace-period=0
```

---

*This documentation translates the HLD/LLD production blueprint into a concrete Spring Boot 3.5 + Java 21 + Gradle Kotlin DSL microservice implementation. All 11 services are covered with build files, application configs, key classes, event contracts, resilience patterns, security, observability, and deployment.*