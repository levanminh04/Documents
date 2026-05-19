# Tài Liệu Seat Map & Booking Plan - Bản Tiếng Việt Theo Code Hiện Tại

## 1. Tóm Tắt Hiện Trạng

Hiện hệ thống đã có nền tảng seat map cho **sự kiện có ghế ngồi**, nhưng chưa xử lý đầy đủ sự kiện **đứng/đi lại tự do** và **mixed event**.

Theo code/schema hiện tại:

- DB đã có `event_layouts`, `event_sectors`, `event_seats`.
- DB đã có `event_seat_inventory` để quản lý trạng thái từng ghế: `AVAILABLE`, `RESERVED`, `SOLD`, `BLOCKED`.
- DB đã có unique constraint `UNIQUE(event_id, event_seat_id)`, giúp tránh tạo trùng inventory cho cùng một ghế trong cùng event.
- `ticket_types.event_sector_id` là optional, nghĩa là một loại vé có thể gắn với sector hoặc không.
- `ticket_types.seat_selection_enabled` đã có, dùng để phân biệt vé có chọn ghế hay không.
- `booking_schema.order_item_seats` đã có trong DB, nhưng Java entity/repository cho bảng này chưa có.
- `SeatMapSyncService` đã có API organizer publish seat map, public get seat map, safe mode chặn xóa/ẩn ghế đã bán/đã giữ.
- `BookingService` hiện vẫn là flow vé thường/zone/general admission, chưa hỗ trợ đặt ghế cụ thể. Hiện nếu buyer gửi `seatIds` cho vé chọn ghế thì backend vẫn báo “chưa hỗ trợ”.
- `TicketReservationService` đã có `acquireSeatLock(eventId, seatId)`, nhưng đang wait tối đa 10s; Phase B có thể thêm fast-fail lock cho trải nghiệm chọn ghế tốt hơn.

Kết luận hiện trạng:

```text
Phase A hiện dùng tốt nhất cho SEATED.
STANDING/MIXED còn cần hardening trước khi làm Phase B booking.
Không cần đập BookingService, chỉ nên tách strategy/service theo từng loại item.
```

---

## 2. Mô Hình Nghiệp Vụ Cho User Non-Tech

Hệ thống cần hỗ trợ 3 loại sự kiện chính.

### Loại 1: Sự kiện có ghế ngồi theo khu vực

Ví dụ:

- Sân bóng đá.
- Nhà hát.
- Rạp biểu diễn.
- Hội trường có số ghế cố định.

Nghiệp vụ:

```text
Khách chọn khu -> chọn ghế cụ thể -> thanh toán -> vé gắn với ghế đó.
```

Ví dụ:

```text
Sector A - Khán đài A
Seat A-01, A-02, A-03
TicketType: Khán đài A
```

Cách quản lý inventory:

```text
Mỗi ghế có 1 trạng thái riêng trong event_seat_inventory.
```

Khi khách chọn ghế A-01:

```text
A-01: AVAILABLE -> RESERVED -> SOLD
```

Loại này gọi là:

```text
sectorType = SEATED
```

---

### Loại 2: Sự kiện mixed, vừa có khu ghế vừa có khu đứng

Ví dụ:

- Concert.
- Festival có VIP ngồi và GA standing.
- Show âm nhạc có khu seated và khu standing.

Nghiệp vụ:

```text
Có khu khách phải chọn ghế.
Có khu khách chỉ chọn số lượng vé, vào khu đó tự đứng/di chuyển.
```

Ví dụ:

```text
VIP Seated -> khách chọn ghế V-A1, V-A2
GA Standing -> khách mua 2 vé, không chọn ghế
```

Cách quản lý inventory:

```text
SEATED sector -> quản lý từng ghế bằng event_seat_inventory.
STANDING sector -> quản lý số lượng bằng ticket_types.quantity_available.
```

Mixed booking phải cho phép một order có cả hai loại:

```json
[
  { "ticketTypeId": "ga-ticket", "quantity": 2 },
  { "ticketTypeId": "vip-ticket", "quantity": 2, "seatIds": ["seat-a1", "seat-a2"] }
]
```

Nguyên tắc quan trọng:

```text
Order phải all-or-nothing.
Nếu giữ ghế VIP thành công nhưng GA hết vé thì rollback toàn bộ.
Nếu trừ GA thành công nhưng ghế VIP bị người khác giữ thì rollback toàn bộ.
```

---

### Loại 3: Sự kiện đi lại tự do

Ví dụ:

- Triển lãm.
- Hội chợ.
- Networking event.
- Workshop không chia ghế.
- Museum/event open-space.

Nghiệp vụ:

```text
Khách chỉ mua loại vé và số lượng.
Không cần chọn ghế.
Không bắt buộc có seat map.
```

Ví dụ:

```text
General Admission Ticket
quantityTotal = 5000
quantityAvailable = 4200
```

Cách quản lý inventory:

```text
Chỉ dùng ticket_types.quantity_available.
Không tạo event_seats.
Không tạo event_seat_inventory.
```

Với loại này, seat map là optional:

```text
Không có seat map: vẫn bán vé bình thường.
Có seat map: chỉ để hiển thị khu vực/đường đi/cổng vào, không dùng để chọn ghế.
```

---

## 3. Seat Map, Sector Type Và Ticket Type

### Quyết định MVP

MVP chỉ hỗ trợ 2 `sectorType`:

```text
SEATED
STANDING
```

Không dùng `ACCESSIBLE` và `VIP_BOX` như sector type chính trong MVP.

### ACCESSIBLE là gì?

`ACCESSIBLE` nên hiểu là ghế/khu dành cho người khuyết tật hoặc wheelchair.

Trong MVP, nên model nó là `seatType`, không phải `sectorType`.

Ví dụ đúng:

```text
Sector A: SEATED
  Seat A-01: REGULAR
  Seat A-02: ACCESSIBLE
```

Không nên làm sớm:

```text
Sector Accessible: ACCESSIBLE
```

Vì accessible thường là thuộc tính của ghế hoặc nhóm ghế trong một khu seated.

### VIP_BOX là gì?

`VIP_BOX` là case nâng cao.

Một VIP box có thể bán theo:

- từng ghế trong box,
- nguyên box,
- bàn/table,
- package nhiều quyền lợi.

MVP không nên xử lý sớm vì sẽ kéo thêm nhiều rule phức tạp.

### Mapping chuẩn cho MVP

```text
SEATED sector:
- Có ghế.
- Có ticket type seatSelectionEnabled=true.
- Có event_seat_inventory.

STANDING sector:
- Không có ghế.
- Có ticket type seatSelectionEnabled=false.
- Không có event_seat_inventory.

GENERAL_ADMISSION event:
- Có thể không có sector.
- Ticket type eventSectorId có thể null.
- Booking chỉ trừ quantity_available.
```

---

## 4. Những Lỗ Hổng Cần Hardening Trước Phase B

### 4.1. `SeatMapSyncService` hiện chỉ lưu `sectorType`, chưa xử lý đúng theo loại sector

Hiện code đang:

```java
sector.setSectorType(defaultIfBlank(sectorPayload.sectorType(), "SEATED"));
```

Vấn đề:

- Chưa normalize `sectorType`.
- Chưa reject `ACCESSIBLE`/`VIP_BOX` trong MVP.
- Chưa tách logic `SEATED` và `STANDING`.
- Vẫn tính `totalCapacity` bằng số ghế active.
- Vẫn gọi inventory sync cho mọi active sector.

Cần sửa:

```text
Nếu SEATED:
- Có seats.
- totalCapacity = số ghế active.
- Tạo/cập nhật event_seats.
- Tạo/cập nhật event_seat_inventory.

Nếu STANDING:
- Không nhận seats trong MVP.
- totalCapacity lấy từ ticketType.quantityTotal hoặc field capacity rõ ràng.
- Không tạo event_seat_inventory.
```

### 4.2. `TicketTypeService` chưa validate `seatSelectionEnabled` khớp sector

Hiện code chỉ check:

```text
eventSectorId nếu có thì sector phải thuộc event.
```

Chưa check:

```text
seatSelectionEnabled=true thì sector phải là SEATED.
seatSelectionEnabled=false thì sector nên là STANDING hoặc null.
```

Cần sửa:

```text
TicketType chọn ghế:
- eventSectorId bắt buộc.
- sectorType phải là SEATED.

TicketType không chọn ghế:
- eventSectorId có thể null.
- nếu có sector thì sectorType phải là STANDING.
```

### 4.3. General admission không nên bắt buộc có seat map

Hiện public seat map endpoint vẫn kỳ vọng có layout.

Nghiệp vụ đúng:

```text
Event đi lại tự do không cần seat map vẫn được publish/bán vé.
```

Cách xử lý MVP:

- `GET /api/events/{idOrSlug}/seat-map` có thể trả 404 nếu event không có layout.
- FE/buyer page hiểu 404 seat map là “event này không dùng seat map”, không phải lỗi event.
- Booking vẫn dùng ticket types để bán vé.

Sau này nếu cần rõ hơn, thêm:

```text
events.admission_mode = GENERAL_ADMISSION | RESERVED_SEATING | MIXED
```

Nhưng Phase B backend chưa bắt buộc thêm field này nếu có thể derive từ ticket types/sectors.

---

## 5. Kế Hoạch Triển Khai Chia Nhỏ

### Phase A.1 - Hardening Sector Type

Mục tiêu: Làm rõ `SEATED` và `STANDING` trước khi đụng booking.

Thay đổi chính:

- Trong `SeatMapSyncService`, thêm normalize/validate `sectorType`.
- Backend MVP chỉ nhận `SEATED` và `STANDING`.
- `ACCESSIBLE` và `VIP_BOX` trả 400 với message rõ.
- Nếu `sectorType` null thì default `SEATED`.

Acceptance criteria:

```text
sectorType=null -> lưu SEATED.
sectorType=seated -> lưu SEATED.
sectorType=STANDING -> lưu STANDING.
sectorType=ACCESSIBLE/VIP_BOX/abc -> 400.
```

---

### Phase A.2 - Tách Logic Publish Cho SEATED Và STANDING

Mục tiêu: Seat map publish không còn xử lý STANDING như SEATED.

SEATED:

- Bắt buộc có ghế active nếu sector active.
- Capacity = số ghế active.
- Tạo/cập nhật `event_seats`.
- Tạo/cập nhật `event_seat_inventory`.
- Chặn hide/delete nếu có ghế `RESERVED/SOLD`.

STANDING:

- Không nhận seats trong MVP.
- Không tạo `event_seats`.
- Không tạo `event_seat_inventory`.
- Capacity lấy từ ticket type quantity hoặc field capacity đã thống nhất.
- Nếu không có layout/seat map, event vẫn có thể bán vé bằng ticket type.

Acceptance criteria:

```text
SEATED sector publish tạo inventory.
STANDING sector publish không tạo inventory.
STANDING sector có seats trong payload -> 400.
Mixed event có 1 SEATED + 1 STANDING publish thành công.
```

---

### Phase A.3 - Validate Ticket Type Khớp Sector

Mục tiêu: Không cho organizer cấu hình sai nghiệp vụ.

Rule:

```text
seatSelectionEnabled=true:
- eventSectorId bắt buộc.
- sectorType phải là SEATED.

seatSelectionEnabled=false:
- eventSectorId có thể null.
- nếu có eventSectorId thì sectorType phải là STANDING.
```

Cần áp dụng ở:

- `TicketTypeService.createTicketType`
- `TicketTypeService.updateTicketType`
- `SeatMapSyncService.publishSeatMap` khi đọc `ticketTypeId` từ sector map data

Acceptance criteria:

```text
Ticket seated gắn STANDING sector -> 400.
Ticket standing gắn SEATED sector -> 400.
Ticket seated không có sector -> 400.
General admission ticket không có sector -> OK.
```

---

### Phase A.4 - General Admission Không Có Seat Map

Mục tiêu: Event đi lại tự do không bị ép tạo layout/seat map.

Hướng MVP:

- Không bắt buộc tạo layout cho event chỉ có ticket type không chọn ghế.
- Buyer page nếu gọi seat map và nhận 404 thì fallback sang UI chọn vé số lượng.
- Backend booking không phụ thuộc seat map với general admission.

Acceptance criteria:

```text
Event chỉ có General Admission ticket, không có layout, vẫn publish/sell được.
GET seat-map có thể 404 nhưng event detail vẫn trả ticket types.
Booking General Admission chạy như flow hiện tại.
```

---

### Phase B.1 - Thêm Java Entity Cho `order_item_seats`

Mục tiêu: Chuẩn bị lưu ghế cụ thể trong order.

DB đã có bảng:

```text
booking_schema.order_item_seats
```

Cần thêm:

```text
OrderItemSeat.java
OrderItemSeatRepository.java
```

Dùng để ghi:

```text
OrderItem VIP x2 -> Seat A1, Seat A2
```

Acceptance criteria:

```text
Hibernate startup OK.
Repository đọc được seats theo orderItemId.
```

---

### Phase B.2 - Giữ Flow Standing/Zone Hiện Tại, Tách Thành Strategy/Service

Mục tiêu: Không đập `BookingService`.

Hiện `BookingService` đã xử lý tốt flow:

```text
ticketType + quantity -> lock ticketType -> trừ quantity_available -> tạo order
```

Giữ logic này cho:

- General admission.
- Standing sector.
- Zone ticket không chọn ghế.

Tái cấu trúc nhẹ:

```text
BookingService = điều phối chung.
ZoneBookingService/Strategy = logic hiện tại.
```

Acceptance criteria:

```text
Booking standing/general admission không đổi behavior.
Cancel/expire vẫn restore ticket_types đúng.
```

---

### Phase B.3 - Thêm Seated Booking Flow

Mục tiêu: Hỗ trợ buyer chọn ghế.

Flow seated:

1. Validate ticket type:
   - `seatSelectionEnabled=true`
   - có `eventSectorId`
2. Validate request:
   - `seatIds` bắt buộc
   - `seatIds.size == quantity`
   - không duplicate seat ids
3. Validate seats:
   - tồn tại
   - active
   - thuộc đúng event
   - thuộc đúng sector của ticket type
4. Acquire seat locks theo thứ tự sorted.
5. Create order + order items trong cùng transaction.
6. Reserve seats bằng atomic update:
   - `AVAILABLE -> RESERVED`
   - set `order_id`
7. Trừ `ticket_types.quantity_available`, tăng reserved.
8. Insert `order_item_seats`.
9. Update Redis seat status best-effort.

Acceptance criteria:

```text
2 user chọn cùng ghế -> chỉ 1 người giữ được.
Ghế đã RESERVED/SOLD -> không đặt được.
Order tạo xong thì inventory ghế là RESERVED.
Order item có danh sách ghế trong order_item_seats.
```

---

### Phase B.4 - Mixed Booking Trong Một Order

Mục tiêu: Concert có thể bán cả GA standing và VIP seated trong cùng order mà không hỏng doanh thu/trải nghiệm.

Rule:

```text
Route theo từng item, không route theo cả event.
```

Ví dụ order:

```json
[
  { "ticketTypeId": "ga", "quantity": 2 },
  { "ticketTypeId": "vip", "quantity": 2, "seatIds": ["A1", "A2"] }
]
```

Cách xử lý:

- Item GA chạy zone flow.
- Item VIP chạy seated flow.
- Toàn bộ nằm trong một DB transaction.
- Nếu bất kỳ item nào fail, rollback toàn bộ.

Để không ảnh hưởng doanh thu:

- Không cho oversell GA nhờ atomic decrement.
- Không cho bán trùng ghế nhờ seat lock + atomic inventory update.
- Không tạo order nửa vời.

Để không ảnh hưởng trải nghiệm:

- Seat lock nên fail nhanh nếu ghế vừa bị người khác giữ.
- Error message nói rõ ghế nào không còn khả dụng.
- FE nên refresh seat status sau khi fail.

Acceptance criteria:

```text
Mixed order thành công -> GA stock giảm, VIP seats RESERVED.
Mixed order fail ở ghế -> GA stock không bị trừ.
Mixed order fail ở GA -> VIP seats không bị giữ.
```

---

### Phase B.5 - Cancel, Expire Và Payment Success

Cancel/expire:

- Existing `restoreStock(orderId)` vẫn restore `ticket_types`.
- Thêm release seat inventory:
  - `RESERVED -> AVAILABLE`
  - clear hoặc giữ audit `order_id` theo policy
  - Redis -> `AVAILABLE`

Payment success:

- Standing:
  - tạo ticket như hiện tại.
- Seated:
  - đọc `order_item_seats`
  - tạo ticket theo từng ghế
  - inventory `RESERVED -> SOLD`
  - set `ticket_id`, `sold_at`
  - Redis -> `SOLD`

Acceptance criteria:

```text
Cancel pending seated order -> ghế quay về AVAILABLE.
Expire pending seated order -> ghế quay về AVAILABLE.
Payment success seated order -> ghế SOLD.
Payment callback retry -> không tạo duplicate ticket.
```

---

### Phase C - Tối Ưu Và Mở Rộng Sau MVP

Không làm trong lần này.

Có thể cân nhắc:

- `events.admission_mode`.
- Redis read-through cho public seat status.
- Fast-fail seat lock.
- `price_levels`.
- Allocation/presale/sponsor hold.
- VIP box/table/package.
- Refund policy cho ghế đã SOLD.

---

## 6. Test Scenarios Cần Có

Phase A hardening:

```text
Publish SEATED sector có ghế -> tạo inventory.
Publish STANDING sector không ghế -> không tạo inventory.
Publish STANDING có seats -> 400.
Publish ACCESSIBLE/VIP_BOX -> 400 trong MVP.
Ticket seated gắn STANDING -> 400.
Ticket standing gắn SEATED -> 400.
Hide/delete ghế SOLD/RESERVED -> 400.
General admission không layout vẫn bán được bằng ticket type.
```

Phase B booking:

```text
Standing-only order thành công.
Seated-only order thành công.
Mixed order thành công.
Mixed order fail ở standing -> rollback seated.
Mixed order fail ở seated -> rollback standing.
2 user chọn cùng 1 ghế -> 1 win, 1 fail.
Order expire release ghế.
Cancel order release ghế.
Payment success chuyển ghế SOLD.
Payment callback retry không duplicate ticket.
```

---

## 7. Assumptions Được Chốt

- Không đập `BookingService`.
- `BookingService` giữ vai trò orchestration.
- Logic booking mới tách theo item: standing/zone strategy và seated strategy.
- MVP chỉ support `SEATED` và `STANDING`.
- `ACCESSIBLE` là `seatType`/attribute, không phải sector type MVP.
- `VIP_BOX` để future.
- General admission không bắt buộc có seat map.
- `ticket_types.event_sector_id` không bỏ.
- DB là source of truth; Redis chỉ là cache/lock hỗ trợ.
- Mixed order phải all-or-nothing để bảo vệ doanh thu và trải nghiệm người dùng.
