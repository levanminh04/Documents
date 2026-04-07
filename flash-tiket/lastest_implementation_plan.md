# Lộ trình phát triển — Flash Ticket System (Backend)

## Mục tiêu

Xây dựng hệ thống hoàn chỉnh ở mức **Demo-ready** — cả 3 service (Core, User, AI) đều hoạt động được cơ bản, đủ để khách hàng tham khảo và hình dung sản phẩm. AI Service sẽ được phát triển đầy đủ, không MVP.

---

## Kiểm kê chính xác: Những gì ĐÃ CÓ (tính đến 05/04/2026)

### ✅ Core Service — Event Module (Hoàn chỉnh)
| API | Endpoint | Status |
|-----|----------|--------|
| Get Categories | `GET /api/categories` | ✅ |
| Get All Events (Public) | `GET /api/events` | ✅ |
| Get Event By ID (Public) | `GET /api/events/{id}` | ✅ |
| Organizer: CRUD Event | `POST/PUT/GET/DELETE /api/organizer/events` | ✅ |
| Organizer: Publish/Cancel | `PATCH .../publish`, `.../cancel` | ✅ |
| Organizer: Image Upload | `POST /api/organizer/events/{id}/images` | ✅ |
| Organizer: Image Metadata | `PATCH/DELETE .../images/{imgId}` | ✅ |
| Organizer: Layout CRUD | `POST/GET/PUT/DELETE .../layout` | ✅ |
| Organizer: Seat Map Sync | `POST .../seat-map/sync` | ✅ |
| Organizer: Ticket Type CRUD | `POST/PUT/GET/DELETE .../ticket-types` | ✅ |
| Venues | `GET /api/venues` | ✅ |
| Categories | `GET /api/categories` | ✅ |

### ✅ Core Service — Booking Module (Hoàn chỉnh)
| API | Endpoint | Status |
|-----|----------|--------|
| Create Booking | `POST /api/bookings` | ✅ |
| Create Booking + Promotion | `POST /api/bookings` (with `promotionCode`) | ✅ |
| Get My Orders | `GET /api/orders/my-orders` (paginated) | ✅ |
| Get Order Detail | `GET /api/orders/{id}` (IDOR protected) | ✅ |
| Cancel Order | `DELETE /api/orders/{id}` (PENDING only, stock restore) | ✅ |
| Get My Tickets | `GET /api/tickets/my-tickets` (paginated) | ✅ |
| Get Ticket Detail | `GET /api/tickets/{id}` (IDOR, có QR data) | ✅ |
| Check-in (Organizer) | `POST /api/tickets/checkin` (HMAC verify, PESSIMISTIC_WRITE) | ✅ |
| Order Expiration | Scheduled job + Stock restore | ✅ |
| Ticket Issuance | Auto-issue on payment success (RabbitMQ event) | ✅ |

### ✅ Core Service — Payment Module (Hoàn chỉnh cơ bản)
| API | Endpoint | Status |
|-----|----------|--------|
| Initiate Payment | `POST /api/payments/initiate` (VNPay URL creation) | ✅ |
| VNPay IPN Callback | `GET /api/payments/vnpay-ipn` (HMAC verify, idempotent) | ✅ |
| Payment Status Polling | `GET /api/payments/status/{orderId}` (IDOR) | ✅ |
| VNPay Return URL | Config `vnp_ReturnUrl` → `http://localhost:5173/payment/result` | ✅ (Frontend handles) |

### ✅ Core Service — Promotion Module
| Feature | Status |
|---------|--------|
| Reserve/Confirm/Release lifecycle | ✅ |
| Atomic counter (`WHERE current_uses < max_total_uses`) | ✅ |
| Scope: ALL events (TODO cho SPECIFIC_EVENTS) | ⚠️ Partial |
| Promotion CRUD cho Admin/Organizer | ❌ Chưa có Controller |

### ✅ User Service (Hoàn chỉnh)
| API | Endpoint | Status |
|-----|----------|--------|
| Get My Profile | `GET /api/users/me` | ✅ |
| Update Profile | `PUT /api/users/me/profile` | ✅ |
| Upload Avatar | `POST /api/users/me/avatar` | ✅ |
| Apply Organizer (KYC) | `POST /api/organizers/apply` | ✅ |
| Get Organizer Profile | `GET /api/organizers/me` | ✅ |
| Upload Logo/Banner | `POST /api/organizers/me/logo`, `.../banner` | ✅ |
| Public Organizer Page | `GET /api/organizers/public/{slug}` | ✅ |
| Follow/Unfollow | `POST/DELETE /api/organizers/{id}/follow` | ✅ |
| Check Follow Status | `GET /api/organizers/{id}/is-following` | ✅ |
| Admin: List Pending Organizers | `GET /api/admin/organizers` | ✅ |
| Admin: Verify Organizer | `PUT /api/admin/organizers/{id}/verify` | ✅ |
| S2S: Get User/Organizer | `GET /api/internal/users/{id}`, `.../organizers/{id}` | ✅ |
| Keycloak Sync (SPI + Self-Healing) | RabbitMQ + JwtSelfHealingFilter | ✅ |
| Email Notification (Thymeleaf) | Auto-send on ticket issuance | ✅ |

### ⬜ AI Service (Chỉ có Application class, chưa có logic)
| Feature | Status |
|---------|--------|
| AiServiceApplication.java | ✅ (Chạy được, nhưng rỗng) |
| Event recommendation | ❌ |
| Similar events | ❌ |
| Search / Discovery | ❌ |
| Popular ranking | ❌ |

### ✅ Infrastructure
| Component | Status |
|-----------|--------|
| API Gateway (Routes, Rate Limiter, Circuit Breaker, JWT Auth, CORS) | ✅ |
| Eureka Discovery | ✅ |
| Config Server | ✅ |
| Docker Compose | ✅ |
| RabbitMQ (Payment events, Keycloak SPI) | ✅ |
| Redis (Distributed Lock, Spam Prevention, IPN Idempotency) | ✅ |
| Cloudinary (Image upload) | ✅ |

---

## Phân tích GAP — Những gì CÒN THIẾU để Demo-ready

### 🔴 Thiếu hẳn (Blocking cho Demo)
| # | Thiếu sót | Ảnh hưởng | Thuộc service |
|---|-----------|-----------|---------------|
| 1 | **AI Service hoàn chỉnh** | Không demo được tính năng AI — 1/3 service rỗng | ai-service |
| 2 | **Organizer Dashboard APIs** | Organizer không có thống kê doanh thu/vé bán | core-service |
| 3 | **Promotion CRUD APIs** | Admin/Organizer không tạo/quản lý được voucher | core-service |
| 4 | **Organizer Stats Sync** | `OrganizerProfile.statistics` luôn trả zero | user-service |

### 🟡 Nên có (Important nhưng không blocking)
| # | Thiếu sót | Ảnh hưởng |
|---|-----------|-----------|
| 5 | **Admin User Management** | Admin không quản lý được users (list/suspend/ban) |
| 6 | **Admin Event Moderation** | Admin không review/feature events |
| 7 | **Error handling thống nhất** | user-service và core-service format lỗi khác nhau |
| 8 | **VNPay QueryDR** | Không chủ động kiểm tra được giao dịch khi IPN miss |
| 9 | **Organizer: xem orders/attendees của event** | Organizer không biết ai mua vé |

### 🟢 Đã đủ tốt cho Demo
- ✅ Full Buyer journey: Xem event → Đặt vé → Thanh toán → Nhận vé + QR → Check-in
- ✅ Full Organizer journey: Apply KYC → Được duyệt → Tạo event → Publish
- ✅ Follow system, Profile management, Upload media
- ✅ Payment flow hoàn chỉnh (VNPay sandbox)

---

## Lộ trình đề xuất — 4 Sprint

> [!IMPORTANT]
> Sprint 1-3 song song nếu bạn muốn. Sprint 1 (AI) có thể làm độc lập vì service tách biệt.

---

### Sprint 1: AI Service — Event Recommendation (Full implementation, KHÔNG MVP)

*Mục tiêu: AI Service hoạt động đầy đủ — tìm kiếm sự kiện, gợi ý tương tự, xếp hạng phổ biến*

> [!IMPORTANT]
> Bạn muốn làm **chi tiết và hết sức**, không MVP. Dưới đây là scope đầy đủ.

#### 1.1 Foundation — PGVector + Embedding Pipeline
- [ ] Thêm PGVector extension vào PostgreSQL (Flyway migration)
- [ ] Entity `EventEmbedding` — lưu vector embedding của event (title + description + tags)
- [ ] `EmbeddingService` — gọi LLM API (OpenAI/Gemini/Ollama) để tạo embedding vector
- [ ] Event Sync mechanism — khi event được tạo/update → auto-generate embedding
  - Lựa chọn: RabbitMQ event từ core-service hoặc polling job
- [ ] Batch script để generate embeddings cho existing events

#### 1.2 API Endpoints — Intelligent Search & Discovery
- [ ] `GET /api/ai/events/search?q=...` — Semantic search (PGVector cosine similarity)
  - Không chỉ keyword match, mà hiểu ngữ nghĩa ("nhạc EDM" → tìm được "đại nhạc hội điện tử")
- [ ] `GET /api/ai/events/{eventId}/similar` — Sự kiện tương tự (PGVector KNN)
- [ ] `GET /api/ai/events/popular` — Xếp hạng phổ biến (scoring algorithm: views + bookings + recency)
- [ ] `GET /api/ai/events/personalized?userId=...` — Gợi ý cá nhân hóa
  - Dựa trên booking history + follow organizers + event categories đã xem
- [ ] `GET /api/ai/events/trending` — Xu hướng (sự kiện có tốc độ bán vé tăng nhanh)

#### 1.3 LangChain4j Integration — Chatbot / Q&A (nếu muốn)
- [ ] Event Q&A endpoint — hỏi bằng ngôn ngữ tự nhiên về events
  - "Cuối tuần này có sự kiện nào ở Hà Nội?"
  - "Tìm concert nhạc rock giá dưới 500K"
- [ ] RAG pipeline: User query → Embedding → PGVector search → LLM aggregate answer

#### 1.4 Infrastructure
- [ ] Kafka consumer để nhận event data từ core-service (hoặc REST polling)
- [ ] Caching cho popular/trending results (Redis, TTL 5-10 min)
- [ ] Rate limiting cho AI endpoints (tốn tài nguyên hơn CRUD)

---

### Sprint 2: Organizer Dashboard + Orders/Attendees

*Mục tiêu: Organizer có đầy đủ công cụ quản lý sự kiện*

#### 2.1 core-service — Organizer Dashboard APIs
- [ ] `GET /api/organizer/dashboard` — Tổng quan:
  - Tổng sự kiện (theo trạng thái: DRAFT/PUBLISHED/COMPLETED/CANCELLED)
  - Tổng doanh thu (sum totalAmount WHERE status = CONFIRMED)
  - Tổng vé đã bán
  - Biểu đồ doanh thu theo tháng (12 tháng gần nhất)
- [ ] `GET /api/organizer/events/{eventId}/stats` — Thống kê chi tiết 1 event:
  - Vé bán theo loại vé (mỗi TicketType: sold/total/revenue)
  - Đơn hàng breakdown (PENDING/CONFIRMED/CANCELLED/EXPIRED)
  - Check-in rate (checked/total tickets)
  - Doanh thu theo ngày (chart data)
- [ ] `GET /api/organizer/events/{eventId}/orders` — Danh sách đơn hàng cho event (phân trang, lọc status)
- [ ] `GET /api/organizer/events/{eventId}/attendees` — Danh sách người tham dự (ticket holders)

#### 2.2 user-service — Organizer Statistics Sync
- [ ] Internal API `GET /api/internal/organizer-stats/{userId}` — core-service expose thống kê
- [ ] Cron job hoặc on-demand sync khi gọi `GET /api/organizers/me`
- [ ] Update `OrganizerProfile.statistics` (totalEvents, totalTicketsSold, totalRevenue)

---

### Sprint 3: Promotion CRUD + Admin Panel

*Mục tiêu: Admin và Organizer quản lý được hệ thống*

#### 3.1 core-service — Promotion CRUD
- [ ] `POST /api/organizer/promotions` — Tạo voucher cho events của mình
- [ ] `GET /api/organizer/promotions` — Danh sách voucher (phân trang, lọc)
- [ ] `PUT /api/organizer/promotions/{id}` — Cập nhật voucher
- [ ] `DELETE /api/organizer/promotions/{id}` — Vô hiệu hóa voucher
- [ ] `GET /api/promotions/validate?code=...&eventId=...` — Public: check voucher hợp lệ (cho frontend preview discount)

#### 3.2 Admin APIs
- [ ] `GET /api/admin/users` — Danh sách users (phân trang, lọc role/status)
- [ ] `PUT /api/admin/users/{userId}/suspend` — Đình chỉ tài khoản (MongoDB + Keycloak)
- [ ] `GET /api/admin/events` — Tất cả events (cho Admin review)
- [ ] `PUT /api/admin/events/{eventId}/featured` — Đánh dấu featured
- [ ] `GET /api/admin/transactions` — Danh sách giao dịch (reconciliation)

---

### Sprint 4: Polish & Production-Ready

*Mục tiêu: Hệ thống sạch sẽ, monitoring, security*

- [ ] Thống nhất Error Response format giữa services
- [ ] Actuator endpoints: `/health`, `/info`, `/metrics`
- [ ] VNPay QueryDR API (chủ động query khi IPN miss)
- [ ] CORS production config
- [ ] Internal API authentication (service-account JWT thay permitAll)
- [ ] API Gateway: thêm routes cho ai-service
- [ ] Swagger/OpenAPI documentation tự động

---

## Open Questions

> [!IMPORTANT]
> Cần xác nhận trước khi code:

1. **Sprint ưu tiên:** Bạn muốn bắt đầu từ Sprint nào? Sprint 1 (AI) có thể làm song song với Sprint 2/3 vì service tách biệt. Bạn muốn làm tuần tự hay song song?

2. **AI LLM Provider:** Bạn muốn dùng LLM nào cho embedding?
   - **OpenAI** (text-embedding-3-small) — chất lượng tốt, tốn phí
   - **Google Gemini** (text-embedding-004) — free tier có, chất lượng tốt
   - **Ollama** (local, miễn phí) — chậm hơn, không cần API key
   - **Sentence Transformers** (Java-based, local) — không phụ thuộc external API

3. **AI Chatbot (LangChain4j RAG):** Có muốn implement phần chatbot Q&A hay chỉ cần search + recommendation?

4. **Promotion scope:** Organizer tạo voucher cho event CỦA MÌNH, hay Admin tạo voucher platform-wide? Hay cả hai?

5. **Admin: suspend user** — Khi suspend user thì orders PENDING có tự động cancel không?

---

## Verification Plan

### Automated Tests
- `mvn test` cho mỗi service sau khi implement
- Integration test cho AI search accuracy (precision@5)
- Load test cho AI endpoints (embedding computation tốn tài nguyên)

### Manual Verification
- Postman collection update cho tất cả API mới
- Demo scenario: Full user journey + AI recommendations
- Organizer dashboard accuracy check vs actual DB data
