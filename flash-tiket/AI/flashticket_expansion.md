# 🚀 Flash Ticket System — Đề Xuất Mở Rộng (Graduation Edition)

> Tham khảo: Ticketmaster (TM1, SafeTix), Eventbrite, vivenu, Dice, StubHub, TicketSwap

---

## Phần 1: Tách Microservices — Vừa đủ cho Saga

### Từ Modular Monolith → 4 Services

```
HIỆN TẠI (core-service monolith):
┌──────────────────────────────┐
│        core-service          │
│  ┌──────┐ ┌───────┐ ┌─────┐ │
│  │Event │ │Booking│ │Pay- │ │
│  │      │ │       │ │ment │ │
│  └──────┘ └───────┘ └─────┘ │
│  ┌──────────┐ ┌───────────┐ │
│  │Promotion │ │Notification│ │
│  └──────────┘ └───────────┘ │
└──────────────────────────────┘

SAU KHI TÁCH:
┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐
│  Event  │  │ Booking  │  │ Payment  │  │ Notification │
│ Service │  │ Service  │  │ Service  │  │   Service    │
│ :8081   │  │ :8083    │  │ :8084    │  │   :8086      │
│         │  │          │  │          │  │              │
│PostgreSQL│  │PostgreSQL│  │PostgreSQL│  │  RabbitMQ    │
│event_    │  │booking_  │  │payment_  │  │  + SMTP      │
│schema   │  │schema    │  │schema    │  │              │
└─────────┘  └──────────┘  └──────────┘  └──────────────┘
      + user-service (:8082, MongoDB) — giữ nguyên
      + discovery-service (:8085, PGVector) — giữ nguyên
      + promotion tạm gộp vào Booking hoặc tách riêng nếu đủ thời gian
```

### Saga Flow: Booking → Payment → Ticket Issuance

```
BookingSaga Orchestrator (trong Booking Service):

Step 1: ValidateEvent
  → Event Service: check event published, sale window, ticket availability
  → Compensate: không cần (read-only)

Step 2: ReserveInventory
  → Event Service: atomic decrement quantity_available
  → Compensate: RestoreInventory (increment lại)

Step 3: ReservePromotion (nếu có voucher)
  → Booking Service: atomic increment current_uses
  → Compensate: ReleasePromotion

Step 4: CreateOrder
  → Booking Service: insert order + order_items (PENDING)
  → Compensate: CancelOrder

Step 5: ProcessPayment
  → Payment Service: create transaction, redirect VNPay
  → Compensate: RefundPayment

Step 6: ConfirmOrder + IssueTickets
  → Booking Service: order → CONFIRMED, generate tickets + QR
  → Compensate: RevokeTickets

Step 7: SendNotification
  → Notification Service: email confirmation (best-effort, no compensate)

Step 8: UpdateAnalytics
  → Event Service: increment tickets_sold
  → No compensate (eventual consistency OK)
```

**Giao tiếp giữa services:** RabbitMQ (event-driven) + REST (synchronous cho validate/query).

---

## Phần 2: Features Mới — BUYER

### 🟢 B1. Virtual Queue (Phòng chờ ảo cho Flash Sale)
> *Tham khảo: Ticketmaster Queue-it, Dice*

**Vấn đề:** 10,000 người đổ vào mua vé cùng lúc → server sập.

**Giải pháp:**
- Khi event sắp mở bán, buyer vào "phòng chờ" (WebSocket connection)
- Hệ thống random vị trí queue (chống bot advantage)
- Thả từng batch (50 người/lần) vào trang mua vé
- Hiển thị: vị trí trong queue, estimated wait time, real-time counter

**Kỹ thuật:**
- Redis Sorted Set: `ZADD queue:{eventId} {randomScore} {userId}`
- WebSocket (STOMP): push position updates realtime
- Rate control: Spring Cloud Gateway throttle per-batch

---

### 🟢 B2. Ticket Transfer (Chuyển vé cho bạn bè)
> *Tham khảo: Ticketmaster SafeTix, vivenu*

**Nghiệp vụ:**
- Buyer đã có vé → muốn chuyển cho người khác
- Nhập email/SĐT người nhận → hệ thống gửi link
- Người nhận accept → **vô hiệu hóa QR cũ**, sinh QR mới cho người nhận
- Giới hạn: max 2 lần transfer/vé, chỉ trước event 24h

**Kỹ thuật:** QR rotation (HMAC-SHA256 resign với recipient userId)

---

### 🟢 B3. Waitlist (Danh sách chờ khi hết vé)
> *Tham khảo: Eventbrite, Dice*

**Nghiệp vụ:**
- Vé sold out → hiện nút "Tham gia danh sách chờ"
- Khi có vé trả lại (cancel/refund/transfer) → auto-notify người đầu waitlist
- Người đầu waitlist có 15 phút để mua, hết hạn → chuyển người tiếp theo

**Kỹ thuật:** Redis List (FIFO queue) + Scheduled job check

---

### 🟢 B4. Wishlist & Reminders (Yêu thích + Nhắc nhở)

- Buyer "yêu thích" event → nhận notification khi sắp mở bán, sắp hết vé, giảm giá
- Calendar integration: "Thêm vào Google Calendar" khi mua vé thành công
- Price drop alert: thông báo khi loại vé giảm giá

---

### 🟢 B5. Social Features
> *Tham khảo: Eventbrite Social*

- **"Bạn bè đang quan tâm"**: Hiện avatar bạn bè (từ follow list) đã mua/xem event này
- **Group Booking**: Tạo nhóm → share link → mọi người chọn vé → checkout chung hoặc riêng
- **Share Event**: Deep link chia sẻ lên MXH kèm preview card (Open Graph)
- **Referral**: Share link → bạn mua vé → bạn được giảm 5%, bạn bè giảm 5%

---

### 🟢 B6. Event Reviews & Ratings (Sau sự kiện)

- Sau event kết thúc → gửi notification "Đánh giá sự kiện"
- Rating 1-5 sao + text review + upload ảnh
- Hiện trên trang organizer profile (tính average rating)
- AI: Sentiment analysis tự động cho admin/organizer dashboard

---

### 🟢 B7. My Tickets — Enhanced Wallet
> *Tham khảo: Ticketmaster App*

- Giao diện "Ví vé" đẹp: hiện QR lớn + thông tin event + countdown
- **Offline mode**: Cache QR code cho trường hợp mất mạng tại venue
- **Live event info**: Map venue, schedule, lineup (nếu music festival)
- **Post-event**: Vé chuyển thành "kỷ niệm số" (collectible) — không xóa

---

### 🔵 B8. Ticket Resale Marketplace (Nâng cao)
> *Tham khảo: StubHub, TicketSwap, Ticketmaster Resale*

- Buyer không đi được → đăng bán lại vé trên platform (controlled resale)
- **Giá cap**: Organizer set max resale price (ví dụ: max 120% face value) — chống phe vé
- **Platform commission**: 10% phí resale
- Khi bán thành công: QR cũ vô hiệu, QR mới cho buyer mới
- **Saga flow**: List → Match Buyer → Escrow Payment → Transfer Ticket → Release Payment

---

## Phần 3: Features Mới — ORGANIZER

### 🟢 O1. Analytics Dashboard (Real-time)
> *Tham khảo: Eventbrite Analytics, TM1*

**Metrics:**
- **Revenue**: Tổng doanh thu, doanh thu theo loại vé, theo ngày
- **Sales Velocity**: Biểu đồ bán vé theo thời gian (line chart)
- **Conversion Funnel**: Views → Clicks → Add to Cart → Checkout → Paid
- **Traffic Sources**: Từ đâu đến (direct, social, search, referral)
- **Demographic**: Phân bố buyer theo thành phố, độ tuổi (nếu có)
- **Seat Map Heatmap**: Khu vực nào bán nhanh nhất

**Kỹ thuật:** WebSocket cho live counter + Chart.js/Recharts trên FE

---

### 🟢 O2. Marketing Tools
> *Tham khảo: Eventbrite Marketing Suite, TM Promoted*

- **Email Campaign**: Organizer gửi email cho danh sách followers
- **Discount Codes**: Tạo batch mã giảm giá (bulk generate)
- **Early Access**: Tạo link "Pre-sale" cho fan cứng (trước ngày mở bán chính thức)
- **Affiliate Links**: Tracking link cho KOL/influencer → commission khi bán được vé
- **UTM Tracking**: Theo dõi nguồn traffic từ các campaign marketing

---

### 🟢 O3. Multi-Event Management
> *Tham khảo: TM1 Event Groups*

- **Clone Event**: Copy event cũ → sửa ngày/giờ (cho tour, recurring shows)
- **Event Series**: Gom nhiều events thành 1 series (ví dụ: "Rock Storm Tour 2026")
- **Bulk Edit**: Sửa giá/thời gian cho nhiều events cùng lúc
- **Template**: Lưu layout/pricing config → dùng lại cho events sau

---

### 🟢 O4. Add-ons & Upsell (Doanh thu phụ)
> *Tham khảo: Eventbrite Add-ons, vivenu Ancillary Revenue*

- Organizer thêm **sản phẩm kèm** khi mua vé:
  - Parking pass (vé giữ xe)
  - Merchandise (áo, mũ, poster)
  - F&B package (combo đồ ăn)
  - VIP upgrade
  - Meet & Greet pass
- Buyer chọn add-ons trong checkout flow → tăng revenue per transaction

---

### 🟢 O5. Check-in Dashboard (Ngày diễn ra)
> *Tham khảo: Eventbrite Organizer App*

- **Real-time check-in counter**: Bao nhiêu người đã vào / tổng vé bán
- **Scan QR từ web**: Organizer/staff mở camera trên web → scan QR
- **Manual check-in**: Tìm theo tên/email/mã vé → check-in thủ công
- **Multi-gate stats**: Mỗi cổng vào bao nhiêu người (nếu nhiều staff)
- **Anomaly alert**: Cảnh báo nếu cùng 1 vé scan 2 lần ở 2 cổng khác nhau

---

### 🟢 O6. Payout & Financial Reports

- **Revenue breakdown**: Doanh thu - Platform fee - Payment gateway fee = Net payout
- **Payout schedule**: Auto payout sau event 7 ngày (giữ để xử lý refund)
- **Tax report**: Xuất PDF/Excel cho kế toán
- **Refund history**: Tracking mọi refund đã xử lý

---

### 🔵 O7. Dynamic Pricing (AI-Powered, Nâng cao)
> *Tham khảo: Ticketmaster Dynamic Pricing*

- AI tự động điều chỉnh giá dựa trên:
  - Sales velocity (bán nhanh → tăng giá, bán chậm → giảm)
  - Thời gian còn lại trước event
  - Demand prediction (based on views, wishlist count)
  - Competitor events cùng thời điểm
- Organizer set min/max price range → AI optimize trong khoảng đó
- Transparency: Hiện cho buyer "Giá hiện tại" + lý do (demand cao)

---

## Phần 4: Features Mới — ADMIN

### 🟢 A1. Platform Dashboard

- **Tổng quan**: Tổng events, tổng users, tổng revenue, active events
- **Real-time**: Events đang bán hôm nay, transactions/giây
- **Growth metrics**: User growth, event growth, revenue growth (MoM, YoY)

---

### 🟢 A2. Content Moderation

- **Event Review Queue**: Events mới publish → Admin review trước khi public (optional)
- **AI Auto-Flag**: Detect event spam, content vi phạm (ảnh không phù hợp, text spam)
- **Report System**: Buyer/Organizer report event/review → Admin xử lý

---

### 🟢 A3. Refund Management

- **Refund Request Queue**: Buyer request refund → Admin/Organizer approve/reject
- **Auto-Refund Rules**: Organizer set policy (VD: full refund trước 48h, 50% trước 24h, 0% sau đó)
- **Batch Refund**: Event cancelled → Admin 1-click refund tất cả orders
- **Refund to Original Payment**: Gọi VNPay Refund API

---

### 🟢 A4. User & Organizer Management

- **User list**: Search, filter, view details, suspend/ban account
- **Organizer verification** (đã có): Mở rộng thêm re-verification khi vi phạm
- **Activity logs**: Ai làm gì, khi nào (audit trail)

---

### 🟢 A5. Platform Configuration

- **Fee settings**: Platform commission %, payment gateway fee
- **Category management**: CRUD categories, reorder, set icon
- **Venue management**: CRUD venues, verify venues
- **Feature flags**: Bật/tắt features (resale, dynamic pricing, AI chatbot) per environment

---

## Phần 5: AI Features — Tích hợp tự nhiên

| Feature | Vai trò | Kỹ thuật |
|---|---|---|
| **AI Event Recommendation** | Homepage personalized cho buyer | Collaborative Filtering + Content-Based (PGVector embeddings) |
| **AI Chatbot** (đã có) | Tư vấn event, hỗ trợ booking | RAG + Agentic Tools (LangChain4j) |
| **AI Auto-Tag** | Organizer upload event → AI suggest categories, tags | Gemini API text classification |
| **AI Description Generator** | Organizer nhập vài keywords → AI viết mô tả event | Gemini API text generation |
| **AI Review Sentiment** | Phân tích sentiment reviews → dashboard organizer | NLP sentiment analysis |
| **AI Fraud Detection** | Detect bot purchases, suspicious patterns | Isolation Forest trên transaction features |
| **AI Dynamic Pricing** | Tự điều chỉnh giá theo demand | XGBoost regression + business rules |
| **AI Demand Forecast** | Dự đoán lượng vé bán cho event mới | Historical data + event features → prediction |
| **AI SEO Optimization** | Auto-generate meta tags, slugs tối ưu | Gemini API |
| **AI Smart Search** | "Tìm concert rock ở Hà Nội cuối tuần này" (NLP) | Semantic search (PGVector) + NLP intent parsing |

---

## Phần 6: Công nghệ mới gợi ý

| Công nghệ | Áp dụng cho feature nào |
|---|---|
| **WebSocket (STOMP/SockJS)** | Virtual Queue, Live check-in counter, Real-time dashboard |
| **Elasticsearch** | Smart Search (thay SQL LIKE hiện tại), full-text + faceted |
| **Kafka** (thay RabbitMQ cho event streaming) | High-throughput event tracking, analytics pipeline |
| **Redis Streams** | Activity feed, notification queue |
| **Spring Batch** | Batch refund, batch payout, scheduled reports |
| **Temporal.io / Camunda** | Saga orchestration engine (thay tự code) |
| **MinIO / S3** | Object storage cho event media (thay Cloudinary nếu cần) |
| **Prometheus + Grafana** | Platform health monitoring, SLA tracking |
| **OpenTelemetry** | Distributed tracing across microservices |
| **Docker Compose → Kubernetes** | Production deployment, auto-scaling |

---

## Phần 7: Roadmap đề xuất

### MVP cho đồ án (8-12 tuần)

**Tuần 1-2: Tách Microservices + Saga**
- [ ] Tách Event Service, Booking Service, Payment Service
- [ ] Implement Saga Orchestrator cho booking flow
- [ ] Inter-service communication (RabbitMQ + REST)

**Tuần 3-4: Buyer Features (Core)**
- [ ] Ticket Transfer
- [ ] Waitlist
- [ ] Wishlist & Reminders
- [ ] Enhanced My Tickets wallet

**Tuần 5-6: Organizer Features**
- [ ] Analytics Dashboard (revenue, sales velocity, funnel)
- [ ] Check-in Dashboard
- [ ] Add-ons & Upsell
- [ ] Marketing tools (discount codes, early access)

**Tuần 7-8: Admin + Refund**
- [ ] Refund system (request → approve → VNPay refund)
- [ ] Platform dashboard
- [ ] Content moderation

**Tuần 9-10: AI Integration**
- [ ] AI Recommendation Engine (homepage personalization)
- [ ] AI Smart Search (semantic search)
- [ ] AI Auto-tag + Description generator
- [ ] Polish AI Chatbot (đã có)

**Tuần 11-12: Polish + Demo**
- [ ] Virtual Queue (nếu đủ thời gian)
- [ ] Event Reviews & Ratings
- [ ] Testing, documentation, demo prep

### Nice-to-have (nếu dư thời gian)
- Ticket Resale Marketplace
- Dynamic Pricing AI
- Social features (group booking, referral)
- Loyalty / Points program
