# Flash Ticket System — Lịch sử Tiến độ Phát triển

> **Tài liệu sống** — cập nhật mỗi khi hoàn thành feature mới.
> Cập nhật lần cuối: **05/04/2026**

---

## Tổng quan Kiến trúc

```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐
│  API Gateway │───▶│ Core Service │    │  AI Service  │
│ (WebFlux)    │    │ (PostgreSQL) │    │  (PGVector)  │
│ JWT + CORS   │    │ Booking/Pay/ │    │  LangChain4j │
│ Rate Limiter │    │ Event/Promo  │    │  ⬜ SCAFFOLD │
│ Circuit Break│    │ ✅ DONE      │    │              │
└──────┬───────┘    └──────┬───────┘    └──────────────┘
       │                   │
       │            ┌──────┴───────┐
       └───────────▶│ User Service │
                    │  (MongoDB)   │
                    │ Profile/KYC/ │
                    │ Follow/Sync  │
                    │ ✅ DONE      │
                    └──────────────┘
```

**Infrastructure:** Eureka ✅ | Config Server ✅ | Redis ✅ | RabbitMQ ✅ | Keycloak ✅ | Cloudinary ✅ | Docker Compose ✅

---

## Phase 1: Event Management — ✅ DONE

### API Endpoints
| Method | Endpoint | Mô tả |
|--------|----------|-------|
| `GET` | `/api/events` | Danh sách events công khai (paginated) |
| `GET` | `/api/events/{id}` | Chi tiết event (public) |
| `GET` | `/api/categories` | Danh sách categories |
| `GET` | `/api/venues` | Danh sách venues |
| `POST` | `/api/organizer/events` | Tạo event mới (DRAFT) |
| `PUT` | `/api/organizer/events/{id}` | Cập nhật event |
| `GET` | `/api/organizer/events` | My events (paginated) |
| `GET` | `/api/organizer/events/{id}` | Chi tiết event (kể cả DRAFT) |
| `PATCH` | `/api/organizer/events/{id}/publish` | Publish event |
| `PATCH` | `/api/organizer/events/{id}/cancel` | Cancel event |
| `DELETE` | `/api/organizer/events/{id}` | Soft-delete event |
| `POST` | `/api/organizer/events/{id}/images` | Upload image (Cloudinary) |
| `PATCH` | `/api/organizer/events/{id}/images/{imgId}` | Update image metadata |
| `DELETE` | `/api/organizer/events/{id}/images/{imgId}` | Delete image |
| `GET` | `/api/organizer/events/{id}/images` | List images |

### Ticket Type CRUD
| Method | Endpoint | Mô tả |
|--------|----------|-------|
| `POST` | `/api/organizer/events/{id}/ticket-types` | Tạo loại vé |
| `PUT` | `/api/organizer/events/{id}/ticket-types/{ttId}` | Cập nhật loại vé |
| `GET` | `/api/organizer/events/{id}/ticket-types` | List loại vé |
| `DELETE` | `/api/organizer/events/{id}/ticket-types/{ttId}` | Xóa loại vé |

### Event Layout + Seat Map
| Method | Endpoint | Mô tả |
|--------|----------|-------|
| `POST` | `/api/organizer/events/{id}/layout` | Tạo layout |
| `GET` | `/api/organizer/events/{id}/layout` | Get layout |
| `PUT` | `/api/organizer/events/{id}/layout` | Update layout |
| `DELETE` | `/api/organizer/events/{id}/layout` | Delete layout |
| `POST` | `/api/organizer/events/{id}/seat-map/sync` | Bulk sync seats |

### Business Rules đã implement
- Event Lifecycle: DRAFT → PUBLISHED → COMPLETED/CANCELLED
- Publish guard: phải có ≥ 1 TicketType ACTIVE
- Soft-delete: chặn nếu đã có vé bán
- Capacity auto-adjust khi add/remove TicketType
- IDOR protection: Organizer chỉ thao tác event của mình
- Image types: BANNER, GALLERY, SEAT_MAP
- Slug auto-generate từ title

---

## Phase 2A: Booking Core — ✅ DONE

### API Endpoints
| Method | Endpoint | Mô tả |
|--------|----------|-------|
| `POST` | `/api/bookings` | Tạo đơn hàng (Zone ticket mode) |
| `GET` | `/api/orders/my-orders` | Danh sách đơn hàng (paginated, IDOR) |
| `GET` | `/api/orders/{id}` | Chi tiết 1 đơn hàng (IDOR) |
| `DELETE` | `/api/orders/{id}` | Hủy đơn PENDING (stock restore) |
| `GET` | `/api/tickets/my-tickets` | Danh sách vé của tôi (paginated) |
| `GET` | `/api/tickets/{id}` | Chi tiết vé + QR code data |
| `POST` | `/api/tickets/checkin` | Check-in bằng QR (ORGANIZER) |

### Anti-Oversell Architecture (3 lớp)
1. **Redis Distributed Lock** (Redisson RLock) — chặn concurrent access
2. **Double-check stock** sau khi acquire lock — tránh stale read
3. **Atomic SQL** `UPDATE WHERE available >= qty` — safety net cuối

### Business Rules đã implement
- Duplicate booking guard (Redis SetNx spam lock 10s)
- Order expiration: 15 phút auto-cancel + stock restore (Scheduled job)
- Ticket issuance: auto-issue khi payment success (RabbitMQ event-driven)
- QR code: HMAC-SHA256 signed (`{ticketCode}|{eventId}|{ticketTypeId}|{signature}`)
- Check-in: PESSIMISTIC_WRITE lock, VALID → USED, double check-in protection
- Promotion reserve: atomic `current_uses < max_total_uses`, release on cancel/expire

---

## Phase 2B: Payment (VNPay) — ✅ DONE

### API Endpoints
| Method | Endpoint | Mô tả |
|--------|----------|-------|
| `POST` | `/api/payments/initiate` | Tạo VNPay payment URL |
| `GET` | `/api/payments/vnpay-ipn` | VNPay IPN callback (S2S, no JWT) |
| `GET` | `/api/payments/status/{orderId}` | Polling trạng thái thanh toán |

### Architecture Decisions
- **IPN Idempotency:** Redis SetNx lock (30s TTL) per `vnp_TxnRef`
- **Signature Verification:** HMAC-SHA512 (VNPay standard)
- **Amount Validation:** Server-side verify (chống tamper)
- **Async Event:** `@TransactionalEventListener(AFTER_COMMIT)` + `@Async` → RabbitMQ
- **VNPay Return URL:** Configured → `http://localhost:5173/payment/result` (Frontend handles)
- **Transaction Number:** `TXN-{yyyyMMddHHmmss}-{6digits}` — unique per payment attempt
- **Multiple attempts:** 1 Order → N Transactions (VNPay requires unique TxnRef each time)

---

## Phase 2C: Email Notification — ✅ DONE

### Mechanism
- Payment success → RabbitMQ event → Email Service consumer
- Thymeleaf HTML template cho ticket confirmation email
- Backup notification channel khi user tắt browser

---

## Phase 3: User Service — ✅ DONE

### Buyer APIs
| Method | Endpoint | Mô tả |
|--------|----------|-------|
| `GET` | `/api/users/me` | Profile hiện tại |
| `PUT` | `/api/users/me/profile` | Cập nhật profile (partial) |
| `POST` | `/api/users/me/avatar` | Upload avatar (Cloudinary) |

### Organizer KYC Journey
| Method | Endpoint | Mô tả |
|--------|----------|-------|
| `POST` | `/api/organizers/apply` | Nộp đơn đăng ký Organizer |
| `GET` | `/api/organizers/me` | Xem organizer profile |
| `POST` | `/api/organizers/me/logo` | Upload logo |
| `POST` | `/api/organizers/me/banner` | Upload banner |
| `GET` | `/api/organizers/public/{slug}` | Public organizer page |

### Follow System
| Method | Endpoint | Mô tả |
|--------|----------|-------|
| `POST` | `/api/organizers/{id}/follow` | Follow organizer |
| `DELETE` | `/api/organizers/{id}/follow` | Unfollow organizer |
| `GET` | `/api/organizers/{id}/is-following` | Check follow status |

### Admin (User Service)
| Method | Endpoint | Mô tả |
|--------|----------|-------|
| `GET` | `/api/admin/organizers` | List pending organizers |
| `PUT` | `/api/admin/organizers/{id}/verify` | Approve/Reject organizer |

### S2S Internal APIs
| Method | Endpoint | Mô tả |
|--------|----------|-------|
| `GET` | `/api/internal/users/{userId}` | Core-service lấy user info |
| `GET` | `/api/internal/organizers/{id}` | Core-service lấy organizer info |
| `GET` | `/api/internal/organizers/by-user/{id}` | Resolve organizer by userId |

### Architecture Decisions
- **Dual-Sync:** Keycloak SPI → RabbitMQ (proactive) + JwtSelfHealingFilter (reactive)
- **IDOR Protection:** userId LUÔN từ JWT `sub`
- **Email immutable:** Keycloak là SSOT, không cho update qua API
- **Follow:** Tách collection riêng + Atomic `$inc` followerCount
- **Organizer Approval:** MongoDB first → Keycloak second (non-atomic, chấp nhận partial failure)

---

## Phase 4: AI Service — ⬜ SCAFFOLD ONLY

| Status | Detail |
|--------|--------|
| ✅ | `AiServiceApplication.java` tạo và chạy được |
| ❌ | Không có controller, service, repository, hay bất kỳ logic nào |
| ❌ | PGVector chưa setup |
| ❌ | LangChain4j chưa integrate |

---

## Infrastructure — ✅ DONE

| Component | Status | Chi tiết |
|-----------|--------|----------|
| API Gateway | ✅ | WebFlux, JWT auth, CORS, Rate Limiter (Redis), Circuit Breaker |
| Eureka Discovery | ✅ | Service registration tự động |
| Config Server | ✅ | Centralized config (Git-backed) |
| Docker Compose | ✅ | Full stack: PostgreSQL, MongoDB, Redis, RabbitMQ, Keycloak |
| Flyway Migration | ✅ | V1, V2 schemas (event, booking, payment, promotion) |
| Keycloak SPI | ✅ | Custom RabbitMQ SPI cho user event sync |

---

## Ghi chú Kiến trúc Quan trọng

| Quy tắc | Chi tiết |
|---------|----------|
| **Cross-database** | PostgreSQL ↔ MongoDB: chỉ dùng `userId` VARCHAR, KHÔNG FK |
| **Messaging** | Thin Event cho critical flow (Payment→Booking). Fat Event cho notify (Booking→Email) |
| **Concurrency** | Redis Lock → Double-check → Atomic SQL (3 lớp) |
| **Security** | IDOR protection mọi endpoint. JWT `sub` = userId. Internal API = permitAll (network isolation) |
| **Idempotency** | IPN: Redis SetNx. Ticket issuance: check existing count. Event sync: upsert |
