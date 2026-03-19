# Flash Ticket System — Đánh Giá Sẵn Sàng Production

> Kiểm chứng từ mã nguồn thực tế: `core-service.yml`, `pom.xml`, `SecurityConfig.java`, `GlobalExceptionHandler.java`, `V2__complete_schema.sql`, và 15+ file Java liên quan.

---

## Dimension 1: Observability (Quan Sát Hệ Thống)

**Score: 1/5** — Gần như chưa có gì.

| Kiểm Tra | Trạng Thái | Mô Tả Gap | Cách Khắc Phục |
|----------|:----------:|------------|----------------|
| Structured Logging (JSON) | ❌ Thiếu | Không có `logback-spring.xml`. Log output dạng plaintext — không thể parse bằng ELK/Loki/CloudWatch. | Tạo `src/main/resources/logback-spring.xml` với `LogstashEncoder` (từ thư viện `logstash-logback-encoder`). Profile `prod` dùng JSON, `dev` giữ plaintext. |
| MDC traceId/userId | ❌ Thiếu | Grep toàn bộ `core-service/src/main/java` cho `MDC` → 0 kết quả. Không thể trace request xuyên suốt các service. | Tạo `MdcFilter.java` implements `OncePerRequestFilter`, đặt `MDC.put("userId", jwt.getSubject())` và `MDC.put("traceId", UUID.randomUUID())` cho mỗi request. |
| Metrics (Micrometer) | ❌ Thiếu | `spring-boot-starter-actuator` có trong `pom.xml` (tự mang Micrometer core), nhưng **không có** `micrometer-registry-*` (Prometheus/InfluxDB) và **không có** custom business metrics (`Counter`, `Timer`, `Gauge`). | Thêm `micrometer-registry-prometheus` vào `pom.xml`. Tạo `BusinessMetricsService.java` với counters: `bookings_created_total`, `payment_success_total`, `payment_failed_total`, `tickets_issued_total`. |
| Distributed Tracing | ❌ Thiếu | Không có `micrometer-tracing-bridge-brave` hoặc `zipkin-reporter-brave` trong `pom.xml`. Không có trace propagation giữa core-service, gateway, user-service. | Thêm `micrometer-tracing-bridge-brave` + `zipkin-reporter-brave` vào `pom.xml`. Cấu hình `management.tracing.sampling.probability=1.0` trong `core-service.yml`. |
| Alerting Rules | ❌ Thiếu | Không có alert rule nào được định nghĩa. `log.error()` là cơ chế duy nhất cho lỗi critical (VD: IPN fail, QR upload fail). | Sau khi có Prometheus metrics: định nghĩa PrometheusRule cho `payment_failed_total > 5 in 5m`, `rabbitmq_dlq_depth > 10`, `http_server_errors_5xx_rate > 1%`. |
| Log Level Production | ⚠️ Chưa phù hợp | `core-service.yml` L130-134: `com.flashticket.core: DEBUG`, `org.springframework.security: DEBUG`, `org.hibernate.SQL: DEBUG`. **Quá verbose cho production** — ảnh hưởng hiệu năng và gây nhiễu log. | Tạo profile `prod` với log level `INFO` cho business, `WARN` cho framework. Giữ `DEBUG` chỉ trong profile `dev`. |
| show-sql | ⚠️ Chưa phù hợp | `jpa.show-sql: true` và `format_sql: true` trong `core-service.yml` L25-28. Mỗi query in ra console → **giảm throughput** nghiêm trọng dưới flash sale. | Đặt `show-sql: false` trong profile prod. Nếu cần debug query, dùng `logging.level.org.hibernate.SQL=DEBUG` có thể toggle runtime qua Actuator. |

---

## Dimension 2: Error Handling (Xử Lý Lỗi)

**Score: 3.5/5** — Nền tảng tốt, thiếu vài case quan trọng.

| Kiểm Tra | Trạng Thái | Mô Tả Gap | Cách Khắc Phục |
|----------|:----------:|------------|----------------|
| `@RestControllerAdvice` | ✅ Đã có | [GlobalExceptionHandler.java](file:///d:/Project/flash-ticket-system/core-service/src/main/java/com/flashticket/core/common/exception/GlobalExceptionHandler.java) — hoạt động đúng. | — |
| Standard `ErrorResponse` | ✅ Đã có | [ErrorResponse.java](file:///d:/Project/flash-ticket-system/core-service/src/main/java/com/flashticket/core/common/exception/ErrorResponse.java) với `timestamp`, `status`, `error`, `message`, `path` — chuẩn RFC 7807-like. | — |
| `ResourceNotFoundException → 404` | ✅ Đã có | Mapped chính xác. | — |
| `InvalidRequestException → 400` | ✅ Đã có | Mapped chính xác. | — |
| `InsufficientStockException → 409` | ✅ Đã có | Mapped chính xác (Conflict). | — |
| `LockAcquisitionException → 503` | ✅ Đã có | Mapped chính xác (Service Unavailable). | — |
| `MethodArgumentNotValidException → 400` | ✅ Đã có | Gộp tất cả field errors thành 1 message. | — |
| Catch-all `Exception → 500` | ✅ Đã có | An toàn — không leak stack trace ra client. | — |
| `AccessDeniedException → 403` | ❌ Thiếu | Spring Security ném `AccessDeniedException` khi `@PreAuthorize` fail. Hiện không được handle → trả về 500 thay vì 403 đúng. | Thêm handler: `@ExceptionHandler(AccessDeniedException.class)` → return `403 Forbidden`. |
| `DataIntegrityViolationException` | ❌ Thiếu | UNIQUE constraint violation (VD: duplicate `ticket_code`, duplicate `order_number`) ném `DataIntegrityViolationException`. Hiện trả 500 generic. | Thêm handler: `@ExceptionHandler(DataIntegrityViolationException.class)` → return `409 Conflict` với message "Dữ liệu bị trùng lặp, vui lòng thử lại". |
| RabbitMQ Consumer Error Handling | ✅ Đã có | `TicketMessageListener` và `EmailMessageListener` đều dùng try/catch + manual NACK → chuyển vào DLQ. | — |
| RabbitMQ DLQ Consumer | ❌ Thiếu | DLQ tồn tại nhưng **không có consumer** — đã được report trong concurrency audit. Không lặp lại chi tiết ở đây. | Xem `concurrency_audit.md` Area D3. |

---

## Dimension 3: Configuration & Secrets Management (Quản Lý Cấu Hình & Bí Mật)

**Score: 3/5** — Kiến trúc tốt (Config Server), nhưng có lỗ hổng quan trọng.

| Kiểm Tra | Trạng Thái | Mô Tả Gap | Cách Khắc Phục |
|----------|:----------:|------------|----------------|
| Spring Cloud Config Server | ✅ Đã có | Tất cả config nằm tập trung tại `configserver/.../config/core-service.yml` — đúng pattern single source of truth. | — |
| DB credentials externalized | ✅ Đã có | `${POSTGRES_URL}`, `${POSTGRES_USER}`, `${POSTGRES_PASSWORD}` — lấy từ environment variable. | — |
| Redis credentials externalized | ✅ Đã có | `${REDIS_HOST}`, `${REDIS_PORT}`, `${REDIS_PASSWORD}`. | — |
| RabbitMQ credentials externalized | ✅ Đã có | `${RABBITMQ_HOST}`, `${RABBITMQ_USERNAME}`, `${RABBITMQ_PASSWORD}`. | — |
| VNPay secrets externalized | ✅ Đã có | `${VNPAY_TMN_CODE}`, `${VNPAY_HASH_SECRET}`. | — |
| Cloudinary secrets externalized | ✅ Đã có | `${CLOUDINARY_CLOUD_NAME}`, `${CLOUDINARY_API_KEY}`, `${CLOUDINARY_API_SECRET}`. | — |
| Mail credentials externalized | ✅ Đã có | `${MAIL_USERNAME}`, `${MAIL_PASSWORD}`. | — |
| Keycloak URI externalized | ✅ Đã có | `${KEYCLOAK_ISSUER_URI}`. | — |
| QR Secret — Hardcoded fallback | 🔴 Nguy hiểm | `core-service.yml` L146: `secret-key: ${QR_SECRET_KEY:flashticket-qr-secret-change-in-prod-min32chars}`. **Nếu env var QR_SECRET_KEY không được set, hệ thống dùng giá trị mặc định — ai cũng biết → giả mạo QR vé.** | Xóa giá trị default: `secret-key: ${QR_SECRET_KEY}`. App sẽ fail fast nếu env var thiếu — đúng behavior cho prod. |
| Environment profiles (dev/prod) | ❌ Thiếu | Chỉ có 1 file `core-service.yml` duy nhất. Không có `core-service-dev.yml`, `core-service-prod.yml`. Log level DEBUG, show-sql true áp dụng cho mọi môi trường. | Tạo `core-service-dev.yml` (giữ debug) và `core-service-prod.yml` (log INFO, show-sql false, Flyway enabled). Dùng `spring.profiles.active=prod` khi deploy. |
| VNPay sandbox URL hardcoded | ⚠️ Cần tách | `core-service.yml` L161-163: VNPay URLs có default trỏ sandbox (`sandbox.vnpayment.vn`). Nếu deploy prod mà quên set env var → **thanh toán đi vào sandbox, mất tiền thật.** | Xóa default cho `VNPAY_PAYMENT_URL` và `VNPAY_API_URL` — bắt buộc phải set env var cho prod. |
| Eureka hardcoded localhost | ⚠️ Cần tách | `core-service.yml` L100: `defaultZone: http://localhost:8761/eureka/` — hardcoded, không dùng env var. | Đổi thành `defaultZone: ${EUREKA_URL:http://localhost:8761/eureka/}`. |
| Actuator endpoints quá mở | 🔴 Nguy hiểm | `management.endpoints.web.exposure.include: "*"` expose **tất cả** endpoints: `/actuator/env` (leak secrets), `/actuator/configprops`, `/actuator/beans`. `SecurityConfig` chỉ permit `/actuator/health` và `/actuator/info` nhưng **Actuator có security riêng** và cấu hình `include: "*"` vẫn expose ở tầng web. | Đổi thành `include: health,info,metrics,prometheus`. Hoặc nếu cần tất cả cho monitoring, thêm `management.server.port: 9081` để Actuator chạy cổng nội bộ. |

---

## Dimension 4: Database Hygiene (Sức Khỏe Cơ Sở Dữ Liệu)

**Score: 2.5/5** — Indexes tốt, nhưng thiếu migration tool và có CASCADE nguy hiểm.

| Kiểm Tra | Trạng Thái | Mô Tả Gap | Cách Khắc Phục |
|----------|:----------:|------------|----------------|
| Flyway migrations | ❌ Bị tắt | `core-service.yml` L32-36: **Flyway bị comment out hoàn toàn**. `hibernate.ddl-auto: validate` nghĩa là app chỉ validate schema, không tạo. Migration được chạy thủ công. Rủi ro: schema drift giữa các môi trường. | Bỏ comment Flyway config. Thêm `spring-boot-starter-data-jpa` đã kéo Flyway tự động. Cần thêm `org.flywaydb:flyway-database-postgresql` vào `pom.xml`. Di chuyển file V1-V5 từ `database/postgresql/` sang `src/main/resources/db/migration/`. |
| Flyway dependency | ❌ Thiếu | `pom.xml` không có `flyway-core` hoặc `flyway-database-postgresql`. | Thêm vào `pom.xml`: `<dependency><groupId>org.flywaydb</groupId><artifactId>flyway-database-postgresql</artifactId></dependency>` (version managed by Spring Boot parent). |
| Indexes — Event schema | ✅ Tốt | `V2__complete_schema.sql`: indexes trên `events.slug`, `events.status`, `events.start_datetime`, `events.is_featured`, `event_schema.categories.slug`, `venues.city` — đều có partial index (`WHERE is_deleted = FALSE`). | — |
| Indexes — Booking schema | ✅ Tốt | Indexes trên `orders.user_id`, `orders.event_id`, `orders.status`, `orders.order_number`, `orders.created_at`, `orders.expires_at`. Entity annotations trong `Order.java` cũng define matching indexes. | — |
| Indexes — Ticket schema | ✅ Tốt | `tickets.ticket_code` (UNIQUE), `tickets.order_id`, `tickets.event_id`, `tickets.status`, `tickets.user_id`. | — |
| Indexes — Promotion schema | ✅ Tốt | `promotions.code` (UNIQUE + partial index), `promotions.status`, `promotion_usages.user_id`. | — |
| Indexes — Transaction schema | ⚠️ Cần kiểm tra | Entity `Transaction.java` define index trên `order_id`, `user_id`, `status`, `provider_transaction_id`. Nhưng cần xác nhận SQL migration có tạo indexes này không. | ⚠️ Cần source: `V2__complete_schema.sql` phần `payment_schema.transactions` — kiểm tra xem indexes có match với Entity annotations. |
| `ON DELETE CASCADE` — order_items | 🔴 Nguy hiểm | `V2__complete_schema.sql` L726: `order_items.order_id REFERENCES orders(id) ON DELETE CASCADE`. Nếu order bị xóa (dù là soft-delete bình thường), một lỗi logic xóa cứng sẽ **xóa sạch order_items** — mất dữ liệu tài chính. | Đổi thành `ON DELETE RESTRICT` cho bảng liên quan đến tiền: `order_items`, `transactions`, `tickets`. Soft-delete là chuẩn duy nhất cho dữ liệu tài chính. |
| `ON DELETE CASCADE` — tickets | 🔴 Nguy hiểm | `V2__complete_schema.sql` L757: `tickets.order_item_id REFERENCES order_items(id) ON DELETE CASCADE`. Chain reaction: xóa order → cascade xóa order_items → cascade xóa tickets. | Đổi thành `ON DELETE RESTRICT`. |
| `ON DELETE CASCADE` — transactions | 🔴 Nguy hiểm | `V2__complete_schema.sql` L791: `transactions.order_id REFERENCES orders(id) ON DELETE CASCADE`. Xóa order → mất toàn bộ lịch sử giao dịch thanh toán. | Đổi thành `ON DELETE RESTRICT`. Dữ liệu thanh toán **không bao giờ được cascade delete**. |
| `ON DELETE CASCADE` — ticket_types | ⚠️ Rủi ro | `V2__complete_schema.sql` L365: `ticket_types.event_id REFERENCES events(id) ON DELETE CASCADE`. Xóa event → xóa tất cả ticket_types → mất config giá vé, stock. | Đổi thành `ON DELETE RESTRICT`. Event nên soft-delete. |
| Full-text search index | ✅ Đã có | `V2__complete_schema.sql` L293: `idx_events_search` dùng `pg_trgm` (trigram) trên `title` — hỗ trợ tìm kiếm. | — |

---

## Dimension 5: Health Checks & Resilience (Kiểm Tra Sức Khỏe & Khả Năng Phục Hồi)

**Score: 1.5/5** — Actuator có nhưng thiếu chiều sâu, không có circuit breaker.

| Kiểm Tra | Trạng Thái | Mô Tả Gap | Cách Khắc Phục |
|----------|:----------:|------------|----------------|
| Actuator /health endpoint | ✅ Đã có | `spring-boot-starter-actuator` trong `pom.xml`. `management.endpoint.health.show-details: always`. SecurityConfig permit `/actuator/health`. | — |
| DB Health (Auto) | ✅ Tự động | Spring Boot tự động thêm `DataSourceHealthIndicator` khi có JPA dependency → kiểm tra connection pool. | — |
| Redis Health (Auto) | ✅ Tự động | `spring-boot-starter-data-redis` (qua Redisson) → `RedisHealthIndicator` tự cấu hình. | — |
| RabbitMQ Health (Auto) | ✅ Tự động | `spring-boot-starter-amqp` → `RabbitHealthIndicator` tự cấu hình. | — |
| Liveness / Readiness Probes | ❌ Thiếu | Kubernetes cần `/actuator/health/liveness` và `/actuator/health/readiness` riêng biệt. Chưa được bật. | Thêm vào `core-service.yml`: `management.endpoint.health.probes.enabled: true` và `management.health.livenessstate.enabled: true`, `management.health.readinessstate.enabled: true`. |
| Circuit Breaker (Cloudinary) | ❌ Thiếu | `QRCodeService` gọi Cloudinary HTTP API trực tiếp. Nếu Cloudinary down → mỗi ticket mất 30s timeout → RabbitMQ consumer bị block. | Thêm `spring-cloud-starter-circuitbreaker-resilience4j` vào `pom.xml`. Wrap `cloudinary.uploader().upload()` trong `CircuitBreaker` instance: `slidingWindowSize=5, failureRateThreshold=50, waitDurationInOpenState=30s`. |
| Circuit Breaker (SMTP) | ❌ Thiếu | `EmailService.sendHtmlEmail()` gọi SMTP trực tiếp. Gmail quota exhausted → mỗi email mất timeout rồi fail. Consumer bị chậm. | Tương tự Cloudinary — wrap `mailSender.send()` trong CircuitBreaker. Khi mạch mở → fast-fail + NACK → message vào DLQ. |
| Circuit Breaker (VNPay) | ⚠️ Ít rủi ro | `VNPayGateway.createPaymentUrl()` chỉ build URL (không gọi HTTP). IPN là VNPay gọi ta → không cần circuit breaker phía ta. QueryDR API (khi implement) sẽ cần. | Chưa cần ngay, nhưng cần khi implement VNPay QueryDR reconciliation. |
| Graceful Shutdown | ❌ Thiếu | Không tìm thấy `server.shutdown: graceful` hoặc `spring.lifecycle.timeout-per-shutdown-phase` trong config. Khi pod restart → in-flight requests (đặc biệt IPN đang xử lý) bị cắt ngang → **mất IPN = mất tiền**. | Thêm vào `core-service.yml`: `server.shutdown: graceful` và `spring.lifecycle.timeout-per-shutdown-phase: 30s`. `AsyncConfig` đã có `setWaitForTasksToCompleteOnShutdown(true)` — tốt nhưng chưa đủ cho Tomcat thread. |
| Rate Limiting | ❌ Thiếu | Không có rate limiting ở tầng core-service. VNPay IPN endpoint public — có thể bị DDoS. | Gateway đã có Resilience4j — nên cấu hình rate limiter ở `gateway-service.yml` cho path `/api/payments/vnpay-ipn`. Hoặc thêm Bucket4j annotation ở controller level core-service. |
| Connection Pool Sizing | ⚠️ Chưa tối ưu | `core-service.yml` không có `spring.datasource.hikari.*` config. HikariCP mặc định `maximumPoolSize=10`. Với flash sale + batch expiry (50 orders × 1 connection mỗi transaction) + concurrent bookings → **pool exhaustion**. | Thêm `spring.datasource.hikari.maximum-pool-size: 30` và `minimum-idle: 10`. Monitor qua Actuator metrics `hikaricp.connections.*`. |

---

## 🔴 Trước Khi Go-Live — 5 Hạng Mục Bắt Buộc

> Không triển khai production nếu chưa hoàn thành 5 mục này. Sắp xếp theo mức độ nguy hiểm giảm dần.

### 1. 🔴 Xóa `ON DELETE CASCADE` trên bảng tài chính

**Rủi ro:** Một bug admin hoặc migration lỗi xóa 1 row `orders` → cascade xóa `order_items` + `tickets` + `transactions` → **mất dữ liệu không thể phục hồi**.

```sql
-- Migration mới: V6__remove_dangerous_cascades.sql
ALTER TABLE booking_schema.order_items
  DROP CONSTRAINT order_items_order_id_fkey,
  ADD CONSTRAINT order_items_order_id_fkey
    FOREIGN KEY (order_id) REFERENCES booking_schema.orders(id) ON DELETE RESTRICT;

ALTER TABLE booking_schema.tickets
  DROP CONSTRAINT tickets_order_item_id_fkey,
  ADD CONSTRAINT tickets_order_item_id_fkey
    FOREIGN KEY (order_item_id) REFERENCES booking_schema.order_items(id) ON DELETE RESTRICT;

ALTER TABLE payment_schema.transactions
  DROP CONSTRAINT transactions_order_id_fkey,
  ADD CONSTRAINT transactions_order_id_fkey
    FOREIGN KEY (order_id) REFERENCES booking_schema.orders(id) ON DELETE RESTRICT;
```

---

### 2. 🔴 Bật Graceful Shutdown

**Rủi ro:** Pod restart cắt ngang VNPay IPN → `Transaction = SUCCESS` nhưng `Order` chưa CONFIRMED → mất tiền.

```yaml
# core-service.yml — thêm block:
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

---

### 3. 🔴 Xóa QR Secret fallback + VNPay sandbox defaults

**Rủi ro:** Deploy prod mà quên set env var → QR secret có thể bị đoán → giả mạo vé. VNPay URL trỏ sandbox → thanh toán vào sandbox.

```yaml
# core-service.yml — sửa:
app:
  qr:
    secret-key: ${QR_SECRET_KEY}              # XÓA default value

vnpay:
  payment-url: ${VNPAY_PAYMENT_URL}           # XÓA sandbox default
  api-url: ${VNPAY_API_URL}                   # XÓA sandbox default
  return-url: ${VNPAY_RETURN_URL}             # XÓA localhost default
```

---

### 4. 🟠 Hạn chế Actuator endpoints + tắt show-sql

**Rủi ro:** `/actuator/env` leak credentials. `show-sql: true` giảm throughput 30-50% dưới tải cao.

```yaml
# core-service.yml — sửa:
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus

spring:
  jpa:
    show-sql: false
    properties:
      hibernate:
        format_sql: false

logging:
  level:
    com.flashticket.core: INFO                # Không DEBUG trong prod
    org.springframework.security: WARN
    org.hibernate.SQL: WARN
```

---

### 5. 🟠 Bật Flyway cho schema versioning

**Rủi ro:** Schema drift — dev/staging/prod có schema khác nhau mà không ai biết. Migration thủ công dễ quên bước.

```xml
<!-- pom.xml — thêm: -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

```yaml
# core-service.yml — bỏ comment:
spring:
  flyway:
    enabled: true
    baseline-on-migrate: true
    locations: classpath:db/migration
    schemas: event_schema,booking_schema,payment_schema,promotion_schema
```

Di chuyển: `database/postgresql/V*.sql` → `core-service/src/main/resources/db/migration/`

---

## Tổng Kết Điểm

| Dimension | Score | Trạng Thái |
|-----------|:-----:|------------|
| 1. Observability | **1/5** | 🔴 Thiếu gần hết — không thể debug sự cố production |
| 2. Error Handling | **3.5/5** | 🟡 Nền tảng tốt, thiếu 2 handler quan trọng |
| 3. Configuration & Secrets | **3/5** | 🟠 Kiến trúc đúng, có lỗ hổng hardcoded secret |
| 4. Database Hygiene | **2.5/5** | 🔴 CASCADE nguy hiểm, Flyway bị tắt |
| 5. Health & Resilience | **1.5/5** | 🔴 Không graceful shutdown, không circuit breaker |
| **Tổng** | **11.5/25** | **Chưa sẵn sàng production** |
