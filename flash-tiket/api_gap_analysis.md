# Flash Ticket System — API Gap Analysis

> Ngày: 19/03/2026 | Dựa trên rà soát 10 Controllers hiện có + toàn bộ Entity/Service layer

---

## Hiện trạng API (Existing Endpoints)

| Controller | Endpoints |
|---|---|
| `EventController` | `GET /api/events`, `GET /api/events/featured`, `GET /api/events/{idOrSlug}` |
| `CategoryController` | `GET /api/categories` |
| `ImageUploadController` | `POST/GET/DELETE /api/organizer/events/{id}/images` (ORGANIZER) |
| `BookingController` | `POST /api/bookings`, `GET /api/orders/my-orders`, `GET /api/orders/{id}`, `DELETE /api/orders/{id}` |
| `TicketController` | `GET /api/tickets/my-tickets`, `GET /api/tickets/{id}`, `POST /api/tickets/checkin` |
| `PaymentController` | `POST /api/payments/initiate`, `GET /api/payments/status/{orderId}` |
| `VNPayIPNController` | `GET /api/payments/vnpay-ipn` (permitAll) |
| `OrganizerController` | `GET /api/organizers/{id}`, `GET /api/organizers/by-user/{userId}` (user-service) |
| `UserController` | ⚠️ **TRỐNG** — Không có endpoint nào |

---

## 1. BUYER APIs — Thiếu sót

| Priority | Method + Path | Request | Response | Justification |
|---|---|---|---|---|
| **MUST-HAVE** | `POST /api/orders/{id}/apply-promo` | `{ promoCode }` | `{ discountAmount, newTotal }` | Buyer hiện không có cách áp dụng mã giảm giá khi đặt vé |
| **MUST-HAVE** | `GET /api/users/me` | JWT | `{ id, name, email, phone, avatar }` | Buyer cần xem thông tin profile (user-service hiện trống) |
| **MUST-HAVE** | `PUT /api/users/me` | `{ name, phone, avatar }` | `{ updatedProfile }` | Cập nhật thông tin cá nhân (vé sẽ dùng name/phone này) |
| **SHOULD-HAVE** | `POST /api/orders/{id}/refund` | `{ reason }` | `{ refundStatus, refundAmount }` | Buyer cần yêu cầu hoàn tiền cho đơn đã thanh toán |
| **SHOULD-HAVE** | `GET /api/tickets/{id}/download-qr` | — | QR image bytes | Tải ảnh QR riêng (ngoài email) để sử dụng offline |
| **SHOULD-HAVE** | `POST /api/tickets/{id}/transfer` | `{ recipientEmail }` | `{ transferId, status }` | Entity Ticket đã có `isTransferable` nhưng chưa có logic |
| **NICE-TO-HAVE** | `GET /api/events/{id}/reviews` | `page, size` | `Page<ReviewDTO>` | Đánh giá sự kiện sau khi tham gia |
| **NICE-TO-HAVE** | `POST /api/events/{id}/reviews` | `{ rating, comment }` | `{ review }` | Gửi đánh giá |

---

## 2. ORGANIZER APIs — Thiếu sót (Nghiêm trọng nhất)

| Priority | Method + Path | Request | Response | Justification |
|---|---|---|---|---|
| **MUST-HAVE** | `POST /api/organizer/events` | `EventCreateDTO` (title, desc, datetime, venue, ticketTypes) | `{ eventId, slug }` | ⛔ **Hiện tại không có API tạo Event!** Organizer không thể làm gì cả |
| **MUST-HAVE** | `PUT /api/organizer/events/{id}` | `EventUpdateDTO` | `{ updatedEvent }` | Chỉnh sửa thông tin sự kiện |
| **MUST-HAVE** | `DELETE /api/organizer/events/{id}` | — | `204 No Content` | Hủy/Ẩn sự kiện (soft delete) |
| **MUST-HAVE** | `GET /api/organizer/events` | `page, size, status` | `Page<OrganizerEventDTO>` | Danh sách sự kiện của Organizer (dashboard) |
| **MUST-HAVE** | `GET /api/organizer/events/{id}/orders` | `page, size, status` | `Page<OrderSummaryDTO>` | Xem danh sách đơn hàng theo sự kiện |
| **MUST-HAVE** | `GET /api/organizer/events/{id}/stats` | — | `{ totalSold, totalRevenue, checkedIn, occupancyRate }` | Thống kê doanh thu & vé — Organizer cần Dashboard |
| **MUST-HAVE** | `POST /api/organizer/events/{id}/ticket-types` | `{ name, price, quantity }` | `{ ticketTypeId }` | Thêm loại vé mới cho sự kiện |
| **MUST-HAVE** | `PUT /api/organizer/events/{id}/ticket-types/{ttId}` | `{ price, quantity }` | `{ updated }` | Cập nhật giá/số lượng vé |
| **SHOULD-HAVE** | `GET /api/organizer/events/{id}/checkin-stats` | — | `{ total, checkedIn, pending }` | Thống kê check-in realtime |
| **SHOULD-HAVE** | `GET /api/organizer/events/{id}/attendees` | `page, size` | `Page<AttendeeDTO>` | Danh sách người mua để check-in thủ công |
| **SHOULD-HAVE** | `POST /api/organizer/promotions` | `{ code, discount, eventId, maxUses, expiry }` | `{ promotionId }` | Tạo mã khuyến mãi (Promotion module có Entity nhưng không có Controller) |
| **SHOULD-HAVE** | `GET /api/organizer/promotions` | `eventId, page` | `Page<PromotionDTO>` | Xem danh sách mã khuyến mãi |
| **SHOULD-HAVE** | `PATCH /api/organizer/events/{id}/publish` | — | `{ status: PUBLISHED }` | Duyệt & xuất bản sự kiện (chuyển từ DRAFT → PUBLISHED) |
| **NICE-TO-HAVE** | `GET /api/organizer/revenue/export` | `eventId, format` | CSV/Excel | Xuất báo cáo doanh thu |

---

## 3. ADMIN APIs — Hoàn toàn chưa có

| Priority | Method + Path | Request | Response | Justification |
|---|---|---|---|---|
| **MUST-HAVE** | `GET /api/admin/events` | `page, status, search` | `Page<AdminEventDTO>` | Quản lý sự kiện toàn hệ thống |
| **MUST-HAVE** | `PATCH /api/admin/events/{id}/status` | `{ status: APPROVED/REJECTED }` | `{ updated }` | Phê duyệt sự kiện trước khi công khai |
| **MUST-HAVE** | `GET /api/admin/users` | `page, role, search` | `Page<UserDTO>` | Quản lý tài khoản người dùng |
| **MUST-HAVE** | `PATCH /api/admin/users/{id}/status` | `{ status: ACTIVE/BANNED }` | `{ updated }` | Khóa/Mở tài khoản vi phạm |
| **SHOULD-HAVE** | `GET /api/admin/orders` | `page, status, dateRange` | `Page<OrderDTO>` | Xem toàn bộ đơn hàng (tra cứu/hỗ trợ khách) |
| **SHOULD-HAVE** | `GET /api/admin/dashboard` | — | `{ totalUsers, totalEvents, revenue, activeOrders }` | Tổng quan hệ thống |
| **SHOULD-HAVE** | `POST /api/admin/refunds/{orderId}` | `{ amount, reason }` | `{ refundId }` | Admin xử lý hoàn tiền |
| **SHOULD-HAVE** | `GET /api/admin/dlq-messages` | `queue, page` | `Page<DLQMessageDTO>` | Monitor DLQ messages (hiện chỉ nằm yên trong queue) |
| **NICE-TO-HAVE** | `POST /api/admin/announcements` | `{ title, content, target }` | `{ id }` | Gửi thông báo toàn hệ thống |
| **NICE-TO-HAVE** | `GET /api/admin/audit-logs` | `userId, action, dateRange` | `Page<AuditLog>` | Kiểm tra lịch sử thao tác |

---

## 4. PUBLIC / ANONYMOUS APIs — Thiếu sót

| Priority | Method + Path | Request | Response | Justification |
|---|---|---|---|---|
| **SHOULD-HAVE** | `GET /api/events/{id}/ticket-types` | — | `List<TicketTypeDTO>` | ⚠️ Hiện đang trả cùng EventDetail nhưng frontend có thể cần gọi riêng để refresh tồn kho |
| **SHOULD-HAVE** | `GET /api/venues` | `city, page` | `Page<VenueDTO>` | Tìm sự kiện theo địa điểm |
| **SHOULD-HAVE** | `GET /api/events/upcoming` | `limit` | `List<EventResponse>` | "Sự kiện sắp diễn ra" (khác w/ featured) |
| **NICE-TO-HAVE** | `GET /api/search/suggest` | `q` | `List<SuggestionDTO>` | Gợi ý tìm kiếm khi gõ (autocomplete) |

---

## ⛔ 5 Critical Gaps — Làm "gãy" luồng End-to-End

| # | Gap | Impact | Severity |
|---|---|---|---|
| 1 | **Không có API tạo Event** (`POST /api/organizer/events`) | Organizer không thể tạo sự kiện mới → hệ thống không có dữ liệu → Buyer không có gì để mua | 🔴 CRITICAL |
| 2 | **Không có API áp dụng Promotion** (`POST /api/orders/{id}/apply-promo`) | Promotion module có Entity + Service nhưng không có Controller → mã giảm giá vô dụng | 🔴 CRITICAL |
| 3 | **user-service trống** (không có `GET /api/users/me`) | Buyer không thể xem/cập nhật profile → thông tin trên vé có thể sai | 🟠 HIGH |
| 4 | **Không có Refund flow** (`POST /api/orders/{id}/refund`) | Chỉ cancel được đơn PENDING. Đơn đã thanh toán (CONFIRMED) không thể hoàn tiền → Buyer kẹt tiền | 🟠 HIGH |
| 5 | **Không có Organizer Dashboard Stats** (`GET /api/organizer/events/{id}/stats`) | Organizer không biết bán được bao nhiêu vé, thu được bao nhiêu tiền → không có lý do dùng platform | 🟠 HIGH |

---

## ⚠️ Security Gaps Spotted

| Issue | File | Severity |
|---|---|---|
| `POST /api/tickets/checkin` thiếu `@PreAuthorize("hasRole('ORGANIZER')")` — bất kỳ user có JWT nào cũng gọi được | [TicketController.java](file:///d:/Project/flash-ticket-system/core-service/src/main/java/com/flashticket/core/booking/controller/TicketController.java#L83) | 🔴 CRITICAL |
| `POST /api/bookings` và các Order endpoints không có `@PreAuthorize` — dù IDOR được protect bằng JWT subject, nhưng không bắt buộc role BUYER cụ thể | [BookingController.java](file:///d:/Project/flash-ticket-system/core-service/src/main/java/com/flashticket/core/booking/controller/BookingController.java) | 🟡 MEDIUM |
