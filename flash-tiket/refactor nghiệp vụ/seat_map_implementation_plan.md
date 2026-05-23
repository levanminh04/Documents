# 🎯 Implementation Plan v5 — Phase 5: Seated Ticket
*Revised: v5 — aligned with final architecture spec (21/05/2026)*

> Tuân thủ: AI_CONTEXT.md §6, §7, §20
> Kiến trúc & Quy chuẩn: [seated_ticket_architecture_spec.md](file:///C:/Users/84583/.gemini/antigravity/brain/a1cb11c7-9cae-4941-84e6-000d8b49efd6/artifacts/seated_ticket_architecture_spec.md)
> Kỹ thuật hiệu năng: SYSTEM_DESIGN_NOTES.md §8 (Locks), §9 (HSET Cache), §10 (Architecture Specs)

---

## Assumptions

1. DB schema nâng cấp: Cập nhật `V2__complete_schema.sql` (cho fresh DB), tạo `V7__add_ticket_type_fields.sql` (cho existing DB) để đưa `inventory_mode`/`access_scope` thành cột thực, và tạo `V8__add_event_seat_ticket_type.sql` để hỗ trợ gán TicketType theo từng ghế trong SEATED sector.
2. Tạo thêm `TicketInventoryCounterService` làm boundary bảo vệ counter.
3. Tạo thêm `CompatibilityPolicy` làm ma trận check logic.
4. `SeatBookingService` và các strategy triển khai đúng interface `InventoryReservationStrategy` tách biệt 2 pha: `precheckAfterLocks` và `recordReservation`.
5. Lock per-seat sử dụng lexicographical sorting và `waitTime = 100ms` để phòng tránh deadlock/livelock và giảm thiểu flaky test.
6. **KHÔNG sửa** `BookingService.restoreStock()`.
7. Items `[PERF]` = không bắt buộc cho MVP, **bắt buộc trước flash sale**.
8. Items `[VERIFIED]` = đã xác nhận tồn tại trong code, chỉ cần tham chiếu.


---

## Đã Có Trong Code — KHÔNG Cần Làm Lại

| Method/Field | File | Ghi chú |
|---|---|---|
| `findByIdAndOrganizerIdAndIsDeletedFalse()` | `EventRepository.java:114` | ✅ Đã có |
| `TicketTypeOrganizerDTO.eventSectorId` | `TicketTypeOrganizerDTO.java:17` | ✅ Đã có + mapper |
| `EventSector.isActive` | `EventSector.java:55` | ✅ Dùng `setIsActive(false)` — KHÔNG có `setVisible()` |
| `EventSeat.isActive` | `EventSeat.java:59` | ✅ Tương tự |

---

## Phase A — Event Module

### A0. `[PERF]` Hibernate Batch — **đúng file config**
```
Sửa: configserver/src/main/resources/config/core-service.yml
     (KHÔNG phải core-service/src/main/resources/application.yml — file đó chỉ import configserver)

Thêm vào block spring.jpa.properties.hibernate (đã có sẵn):
  jdbc:
    batch_size: 500
  order_inserts: true
  order_updates: true

Kết quả sau sửa (core-service.yml lines 26-29):
  properties:
    hibernate:
      format_sql: false
      dialect: org.hibernate.dialect.PostgreSQLDialect
      jdbc:
        batch_size: 500
      order_inserts: true
      order_updates: true

→ verify: startup OK, không có regression
```

### A1. EventSectorRepository — thêm query
```
Sửa: EventSectorRepository.java
Thêm: List<EventSector> findAllByLayoutId(UUID layoutId)
(trả cả inactive — publishSeatMap cần load toàn bộ hiện trạng)
→ verify: compile OK, findByLayoutIdAndIsActiveTrue() không bị đổi
```

### A2. EventSeatRepository — thêm query
```
Sửa: EventSeatRepository.java
Thêm: List<EventSeat> findAllBySectorId(UUID sectorId)
(trả cả inactive)
→ verify: compile OK
```

### A3. EventSeatInventoryRepository — thêm queries
```
Sửa: EventSeatInventoryRepository.java
Thêm:
  Optional<EventSeatInventory> findByEventIdAndEventSeatId(UUID eventId, UUID seatId)
  List<EventSeatInventory> findAllByOrderId(UUID orderId)
  long countByEventSeatIdInAndStatusIn(Collection<UUID> seatIds, Collection<String> statuses)

[PERF] Bulk INSERT native query:
  @Modifying
  @Query(value = """
      INSERT INTO event_schema.event_seat_inventory
          (id, event_id, event_seat_id, ticket_type_id, status, created_at, updated_at)
      SELECT uuid_generate_v4(), :eventId, s.id, s.ticket_type_id, 'AVAILABLE', NOW(), NOW()
      FROM event_schema.event_seats s
      WHERE s.sector_id = :sectorId AND s.is_active = true
        AND NOT EXISTS (
            SELECT 1 FROM event_schema.event_seat_inventory i
            WHERE i.event_seat_id = s.id AND i.event_id = :eventId
        )
      """, nativeQuery = true)
  int bulkInsertInventoryForSector(UUID eventId, UUID sectorId);

[SAFETY] Atomic conditional UPDATE (chỉ update khi vẫn AVAILABLE):
  @Modifying
  @Query(value = """
      UPDATE event_schema.event_seat_inventory
      SET status = 'RESERVED', order_id = :orderId, updated_at = NOW()
      WHERE event_seat_id = :seatId
        AND event_id = :eventId
        AND ticket_type_id = :ticketTypeId
        AND status = 'AVAILABLE'
      """, nativeQuery = true)
  int reserveSeatIfAvailable(UUID seatId, UUID eventId, UUID ticketTypeId, UUID orderId);
  -- Trả về 0 nếu ghế không còn AVAILABLE → báo lỗi oversell ngay

→ verify: compile OK
```

### A4. EventSeatInventory entity — map ticketTypeId
```
Sửa: EventSeatInventory.java
Thêm: @Column(name = "ticket_type_id") private UUID ticketTypeId;
→ verify: Hibernate startup không báo schema mismatch
```

### A4b. EventSeat entity — map ticketTypeId + colorCode
```
Sửa: EventSeat.java
Thêm:
  @Column(name = "ticket_type_id") private UUID ticketTypeId;
  @Column(name = "color_code", length = 7) private String colorCode;

Ý nghĩa:
  event_seats.ticket_type_id = source of truth ghế thuộc TicketType/vùng giá nào.
  event_seat_inventory.ticket_type_id = snapshot để booking validate nhanh.
  colorCode chỉ phục vụ FE render, không dùng cho booking validation.
→ verify: Hibernate startup không báo schema mismatch
```

### A5. SeatMapSyncService — Fix IDOR + tách getSeatMap

**A5a — Fix IDOR (Organizer getSeatMap):**
```
Sửa: SeatMapSyncService.java line 40-44
Đổi: findById() + manual compare
Thành: findByIdAndOrganizerIdAndIsDeletedFalse() [VERIFIED — đã có sẵn]
```

**A5b — Organizer thấy cả INACTIVE:**
```
Sửa: SeatMapSyncService.java lines 52, 67
  Sector: findByLayoutIdAndIsActiveTrue → findAllByLayoutId (A1)
  Seat:   findBySectorIdAndIsActiveTrue → findAllBySectorId (A2)
→ verify: hide ghế → publish → reload → ghế vẫn hiện (hidden=true trong FE)
```

**A5c — getPublicSeatMap() cho Buyer:**
```
Thêm method: SeatMapResponse getPublicSeatMap(String idOrSlug)
  1. Resolve event (UUID → fallback slug)
  2. Check PUBLISHED else 404
  3. Sectors/Seats: chỉ isActiveTrue
  [PERF] Đọc seat status từ Redis HGETALL trước → fallback DB + re-warm

→ verify: DRAFT → 404; PUBLISHED → data đúng
```

### A6. publishSeatMap() — full validation

```
@Transactional
publishSeatMap(UUID eventId, SeatMapPublishRequest payload, String organizerId):

  Bước 0: IDOR — findByIdAndOrganizerIdAndIsDeletedFalse() [VERIFIED]

  Bước 1: Load hiện trạng
    dbSectorMap = findAllByLayoutId → HashMap
    dbSeatMap   = findAllBySectorId (per sector) → HashMap

  Bước 2: Validate payload
    - Duplicate sectorId/seatId → 400
    - Duplicate (sector, row, seatNumber) → 400
    - Bỏ qua `event_sectors.mapData.ticketTypeId` nếu FE cũ còn gửi lên.
      TicketType của ghế phải lấy từ per-seat `seat.ticketTypeId`.

  Classify: toInsert/toUpdate/toHide/toDelete cho sectors và seats

  Bước 3: Safe Mode Validation
    Chốt 1: Với SEATED sector, candidateSeats có SOLD/RESERVED → 400
    Chốt 2: Với SEATED sector, candidateSectors có vé đã bán → 400
    Chốt 3 (Standing Capacity Safe Update): 
      Nếu sectorType = STANDING và đang giảm totalCapacity:
        Lấy tất cả TicketType thuộc sectorId đó từ DB.
        Đảm bảo newTotalCapacity >= sum(sold + reserved across all ticket types in sector).
        Nếu không thỏa mãn → 400 "Sức chứa mới nhỏ hơn lượng vé đã bán và giữ chỗ".

  Bước 4: Persist
    saveAll(toInsertSectors, toUpdateSectors)
    toDeleteSectors: setIsActive(false)   ← KHÔNG setVisible (không tồn tại)
    toHideSectors:   setIsActive(false)   ← hide = isActive=false trong entity
    (tương tự cho seats)

    Xử lý điền ticket_type_id (Chỉ áp dụng cho SEATED sector):
      - Không đọc `event_sectors.mapData.ticketTypeId` làm source of truth.
      - Với từng ghế SEATED, lưu `seat.ticketTypeId` vào `event_seats.ticket_type_id`.
      - Validate `seat.ticketTypeId` thuộc TicketType active của cùng event và cùng sector.
      - Nếu ghế đang RESERVED/SOLD thì không cho đổi `event_seats.ticket_type_id`.
      - Nếu ghế AVAILABLE/BLOCKED/null inventory thì cho đổi và sync snapshot sang `event_seat_inventory.ticket_type_id`.
      - Trước khi mở bán, mọi ghế active trong SEATED sector phải có ticketTypeId hợp lệ.

    [PERF] Bulk INSERT inventory for all active SEATED sectors in layout (chèn ghế mới):
      for each active SEATED sectorId in layout:
        bulkInsertInventoryForSector(eventId, sectorId)
        -- SQL: INSERT INTO event_seat_inventory (event_id, event_seat_id, ticket_type_id, status)
        -- SELECT :eventId, s.id, s.ticket_type_id, 'AVAILABLE' FROM event_seats s
        -- WHERE s.sector_id = :sectorId AND s.is_active = true
        -- AND NOT EXISTS (SELECT 1 FROM event_seat_inventory esi WHERE esi.event_seat_id = s.id)

    [PERF] Warm-up Redis HSET (Tránh ghi đè AVAILABLE):
      Lấy tất cả các ghế hoạt động từ event_seat_inventory cho eventId.
      putAll(Redis map 'seat_status:{eventId}', seatId -> dbStatus)
      expire(48h)

  return getSeatMapForOrganizer(eventId)

→ verify: Publish 5,000 ghế < 2s; xóa ghế SOLD → 400; đổi ticketType có RESERVED → 400; nạp ấm cache giữ đúng trạng thái DB
```

### A7. EventSeatMapController — POST /publish
```
Thêm: @PostMapping("/publish")
  → publishSeatMap(eventId, payload, jwt.getSubject())
  → HTTP 200 SeatMapResponse
```

### A8. EventController — GET /{idOrSlug}/seat-map
```
Thêm: @GetMapping("/{idOrSlug}/seat-map") Auth: permitAll
  → getPublicSeatMap(idOrSlug)
```

### A9. TicketTypeDTO (public & organizer) — thêm fields
```
Sửa: TicketTypeDTO.java (public buyer-facing DTO) và các Mapper
Thêm: UUID eventSectorId, InventoryMode inventoryMode, AccessScope accessScope, String colorCode
Lý do: Cần thiết để FE Buyer phân biệt được luồng số lượng hay chọn ghế trên sơ đồ.

Sửa thêm SeatMapResponse/Seat DTO:
  - Thêm `ticketTypeId`
  - Thêm `price`
  - Thêm `colorCode`

Sửa SeatMapPublishRequest.SeatPayload:
  - Nhận per-seat `ticketTypeId`
  - Nhận per-seat `colorCode`
```


### A10. TicketTypeService — validate & force-set fields
```
Sửa: TicketTypeService.java — createTicketType() và updateTicketType()

1. Thêm validation khi eventSectorId != null:
   EventSector sector = eventSectorRepository.findById(eventSectorId)
       .orElseThrow(() -> 404);
   if (!sector.getLayout().getEvent().getId().equals(eventId))
       throw new InvalidRequestException("Sector không thuộc event này");

2. Xử lý nghiệp vụ ASSIGNED_SEAT:
   if (req.inventoryMode() == InventoryMode.ASSIGNED_SEAT) {
       // Validate 1: eventSectorId bắt buộc và sectorType phải là SEATED
       if (eventSectorId == null || sector.getSectorType() != SEATED) throw 400;

       // Validate 2: Cho phép nhiều ASSIGNED_SEAT TicketType trong cùng SEATED sector.
       // TicketType đóng vai trò vùng giá/hạng ghế trong MVP.

       // Validate 3: Đọc số lượng ghế active đang gán vào ticket type hiện tại để làm quantityTotal.
       int activeSeats = eventSeatRepository.countByTicketTypeIdAndIsActiveTrue(ticketTypeId);
       int soldCount = eventSeatInventoryRepository.countByTicketTypeIdAndStatus(ticketTypeId, "SOLD");
       int reservedCount = eventSeatInventoryRepository.countByTicketTypeIdAndStatus(ticketTypeId, "RESERVED");
       
       // Override / Ghi đè các trường tồn kho trong Entity trước khi lưu.
       // Với create mới, activeSeats có thể = 0 cho đến khi organizer gán ghế vào TicketType trên seat map.
       ticketType.setQuantityTotal(activeSeats);
       ticketType.setQuantityAvailable(activeSeats - soldCount - reservedCount);
       ticketType.setQuantityReserved(reservedCount);
   }

3. Đồng bộ tương thích ngược (Backward Compatibility):
   ticketType.setSeatSelectionEnabled(req.inventoryMode() == InventoryMode.ASSIGNED_SEAT);

Lý do: Chặn organizer cấu hình sai, tự động tính toán quota dựa trên sơ đồ ghế vật lý.
→ verify: 
  - Gán sector event khác → 400
  - Cấu hình ASSIGNED_SEAT cho STANDING sector → 400
  - Cấu hình nhiều ASSIGNED_SEAT ticket type trong cùng SEATED sector → OK
  - Tạo/sửa vé ASSIGNED_SEAT có quantityTotal khác số ghế active gán vào ticket type → Tự động override theo số ghế
```


---

## Phase B — Booking Module

### B0. OrderItemSeat — entity + repository mới
```
Tạo: com.flashticket.core.booking.entity.OrderItemSeat.java
  @Entity @Table("booking_schema.order_item_seats")
  Fields: id, orderItemId, seatId, seatLabel, rowName, seatNumber, price, status, createdAt

Tạo: com.flashticket.core.booking.repository.OrderItemSeatRepository.java
  List<OrderItemSeat> findByOrderItemId(UUID orderItemId)
  void deleteByOrderItemId(UUID orderItemId)  ← không dùng nhưng cần cho completeness

Lý do: B1, B3, B4 đều cần entity này để ghi audit trail "vé này → ghế nào".
→ verify: Hibernate startup OK, bảng tồn tại trong V2 schema
```

### B1. `[PERF]` TicketReservationService — acquireGlobalLocks()
```
Tạo: public List<RLock> acquireGlobalLocks(UUID eventId, List<BookingItem> items)
  - Thu thập tất cả Lock Keys cần thiết cho đơn hàng:
    - Với mỗi item có ticketType là QUANTITY: thêm "lock:zone:" + ticketTypeId
    - Với mỗi item có ticketType là ASSIGNED_SEAT: với mỗi seatId, thêm "lock:seat:" + eventId + ":" + seatId
  - Sắp xếp lexicographically (bảng chữ cái):
    List<String> sortedKeys = allKeys.stream().sorted().toList();
  - Acquire locks tuần tự:
    List<RLock> acquiredLocks = new ArrayList<>();
    for (String key : sortedKeys) {
        RLock lock = tryAcquireLock(key, 100, 15); // waitTime=100ms, leaseTime=15s
        if (lock == null) {
            // Giải phóng ngay tất cả các lock đã acquire trước đó (Fail-Fast)
            acquiredLocks.forEach(RLock::unlock);
            throw new InsufficientStockException("Tài nguyên (vé/ghế) hiện đã bị giữ bởi giao dịch khác.");
        }
        acquiredLocks.add(lock);
    }
    return acquiredLocks;

Lý do: Phase B reject mixed-zone order, nhưng vẫn cần tránh Deadlock / Livelock khi một request seated khóa nhiều ghế, và giữ nền tảng mở rộng nếu sau này cho cart nhiều zone.
→ verify: acquireGlobalLocks regression PASS
```

### B2. SeatBookingService — service mới

**Flow booking seated đúng thứ tự (fix Rủi ro #2):**

```
⚠️ QUAN TRỌNG: orderId chỉ tồn tại SAU KHI Order được tạo (BookingService line 231).
   reserveSeats() PHẢI được gọi SAU buildOrder() + orderRepository.save().
   Thứ tự đúng trong doBooking():
     1. Lock toàn cục (Zone + Seat) theo thứ tự sorted lexicographically
     2. Validate (post-lock double-check cho cả Seated và Zone)
     3. Decrement ticket_types counter qua TicketInventoryCounterService
     4. Calculate promotion
     5. buildOrder() + save() → orderId có
     6. buildOrderItems() + save() → orderItemId có
     7. reserveSeats(seatIds, orderId, orderItemId, ...) → ghi inventory + order_item_seats
```

```java
Tạo: SeatBookingService.java

validateSeatedItems(seatIds, ticketTypeId, eventId, userId)
  - Mỗi seatId: tồn tại, isActive=true, đúng event, đúng ticketType
  - Verify `event_seats.ticket_type_id == ticketTypeId`
  - Verify snapshot `event_seat_inventory.ticket_type_id == ticketTypeId`
  - Không trùng seatId trong 1 order
  - inventoryStatus == AVAILABLE (post-lock double-check)

reserveSeats(seatIds, orderId, orderItemId, ticketTypeId, eventId)
  *** Gọi SAU KHI orderId và orderItemId đã có ***
  (a) Atomic conditional UPDATE per seat (A3):
      int updated = inventoryRepo.reserveSeatIfAvailable(seatId, eventId, ticketTypeId, orderId)
      if (updated == 0) throw oversell exception
      Nếu bất kỳ ghế nào fail → rollback (trong @Transactional)
  (b) Insert OrderItemSeat per seatId (B0)
  (c) Redis HSET: seatIds → "RESERVED"

restoreSeatedStock(orderId)
  *** CHỈ 2 việc — KHÔNG touch ticket_types ***
  (a) UPDATE inventory → AVAILABLE WHERE order_id=orderId
  (b) Redis HSET: affectedSeatIds → "AVAILABLE"

confirmSeatsSold(orderId)
  (a) UPDATE inventory → SOLD
  (b) Redis HSET: affectedSeatIds → "SOLD"

→ verify:
  reserveSeats(): inventory=RESERVED + counter đúng + order_item_seats đúng
  2 users cùng ghế → 1 win, 1 fail <150ms
  Deadlock test: lock ordering fix → không deadlock
```

### B3. BookingService — routing (3 chỗ, thứ tự đúng)

```java
// (a) Inject SeatBookingService
private final SeatBookingService seatBookingService;
private final TicketReservationService ticketReservationService;

// (b) buildAndValidateContexts() — thay throw "not supported":
validateSingleBookingZone(contexts);

if (ctx.ticketType().getInventoryMode() == InventoryMode.ASSIGNED_SEAT) {
    if (seatIds == null || seatIds.size() != quantity)
        throw new InvalidRequestException("Số ghế phải khớp số lượng vé");
}

// validateSingleBookingZone():
// - Phase B chỉ cho phép 1 booking request thuộc 1 vùng bán.
// - Reject nếu trộn EVENT-level + SECTOR-level.
// - Reject nếu request chứa nhiều eventSectorId khác nhau.
// - Reject nếu trộn STANDING sector và SEATED sector trong cùng order.

// (c) doBooking() — THÊM seated branch với thứ tự đúng:
List<RLock> acquiredLocks = new ArrayList<>();
try {
    // 1. Acquire Lock Toàn Cục theo thứ tự sắp xếp lexicographical
    acquiredLocks = ticketReservationService.acquireGlobalLocks(eventId, items);

    // 2. Validate & Reserve Counter
    for (BookingItemContext ctx : contexts) {
        if (ctx.ticketType().getInventoryMode() == InventoryMode.ASSIGNED_SEAT) {
            seatBookingService.validateSeatedItems(seatIds, ticketTypeId, eventId, userId);
            // Decrement counter qua counterService (trước buildOrder)
            ticketInventoryCounterService.reserveCounter(ticketTypeId, count);
        } else {
            // Zone flow double check
            ticketReservationService.validateZoneStock(ctx.ticketType().getId(), count);
            ticketInventoryCounterService.reserveCounter(ctx.ticketType().getId(), count);
        }
    }



    // Promotion + buildOrder() + save() → orderId
    Order order = buildOrder(...);
    order = orderRepository.save(order);

    // buildOrderItems() + save() → orderItemId
    List<OrderItem> items = buildOrderItems(order, contexts, event);
    orderItemRepository.saveAll(items);

    // Seated: reserveSeats() SAU KHI có orderId + orderItemId
    for (OrderItem item : items) {
        if (isSeatedItem(item)) {
            seatBookingService.reserveSeats(
                seatIdsForItem, order.getId(), item.getId(), ticketTypeId, eventId);
        }
    }
} finally {
    acquiredLocks.forEach(ticketReservationService::releaseLock);
}

KHÔNG SỬA logic zone hiện có của restoreStock(), cancelOrder() hay buildOrder(), chỉ bổ sung bọc luồng xử lý vòng đời Seated sau khi hoàn tất zone stock restore.
→ verify: Zone regression PASS; Seated: order + inventory + order_item_seats đúng
```


### B4. TicketIssuanceService — seated branch
```
Sửa: TicketIssuanceService.java
Với seated items:
  1. Lấy OrderItemSeat theo orderItemId (B0 repository)
  2. Mỗi seat → Ticket với seatId + seatLabel
  3. Gọi seatBookingService.confirmSeatsSold(orderId)
  4. Zone: giữ nguyên

→ verify: ticket có seatId+seatLabel; inventory=SOLD; Redis=SOLD
```

### B5. cancelOrder() + OrderExpiration
```
Sửa: BookingService.cancelOrder() sau restoreStock(orderId):
  boolean hasSeated = items.stream()
      .anyMatch(i -> i.getTicketType().getInventoryMode() == InventoryMode.ASSIGNED_SEAT);
  if (hasSeated) seatBookingService.restoreSeatedStock(orderId);

Sửa: OrderExpirationHelper — tương tự.


Anti double-count:
  restoreStock()       → ticket_types += N          ✅
  restoreSeatedStock() → inventory + Redis=AVAILABLE ✅ (KHÔNG đụng ticket_types)
```

---

## Phase FE (FE Contract Reference)

> [!NOTE]
> Các cấu phần này mô tả phạm vi refactor Frontend để đồng bộ với API Phase B. Frontend team sẽ thực hiện các nhiệm vụ này dựa theo API Contract.

### FE1. Form Quản Lý Ticket Type (Organizer Wizard)
*   **Mô tả:** Cập nhật form Tạo/Sửa Vé.
*   **Refactor:**
    *   Thêm trường select **Access Scope** (`EVENT` / `SECTOR`).
    *   Nếu chọn `SECTOR`, hiển thị dynamic dropdown lấy danh sách Sector đang active của event.
    *   Thêm trường select **Inventory Mode** (`QUANTITY` / `ASSIGNED_SEAT`).
    *   Validate trên client: Nếu chọn sector có loại `STANDING` $\rightarrow$ Khóa trường chọn vị trí ghế và ép về `QUANTITY`.
    *   Với sector `SEATED`, cho phép tạo nhiều TicketType `ASSIGNED_SEAT` trong cùng một sector. Mỗi TicketType là một vùng giá/hạng ghế.
    *   Với TicketType `ASSIGNED_SEAT`, trường số lượng phải read-only và lấy từ số ghế active đang gán vào TicketType đó.
    *   Thêm `colorCode` cho TicketType để FE tô màu ghế theo vùng giá.

### FE2. Seat Map Editor (Organizer Canvas)
*   **Mô tả:** Chỉnh sửa thuộc tính của Sector trên Konva Canvas.
*   **Refactor:**
    *   Khi vẽ Sector mới hoặc click vào Sector có sẵn, hiển thị form tùy chỉnh:
        *   Chọn `sectorType` (`SEATED` / `STANDING`).
        *   Nếu chọn `STANDING`: Hiển thị trường nhập số **Sức chứa tối đa (Total Capacity)**, ẩn toàn bộ chức năng vẽ ghế vật lý.
        *   Nếu chọn `SEATED`: Ẩn trường nhập Capacity (tự động tính bằng số ghế vật lý vẽ trên canvas), hiển thị bảng cấu hình hàng ghế.
    *   Với `SEATED`, bổ sung tool gán ghế vào TicketType:
        *   Organizer chọn TicketType/vùng giá, rồi click/drag chọn ghế để gán `seat.ticketTypeId`.
        *   Ghế lấy màu từ `ticketTypes.colorCode` hoặc snapshot `seat.colorCode`.
        *   Payload publish phải gửi per-seat `ticketTypeId` và `colorCode`.
        *   Không dùng `sector.mapData.ticketTypeId` làm source of truth vì một sector có thể có nhiều TicketType.
        *   Chặn publish nếu còn ghế active trong SEATED sector chưa có `ticketTypeId`.
    *   Bảo toàn thuộc tính `isActive` khi ẩn/xóa ghế (`seat.isActive = false`) thay vì xóa cứng.

### FE3. Flow Mua Vé (Buyer Booking Flow)
*   **Mô tả:** Phân nhánh giao diện hiển thị cho Buyer dựa trên `TicketTypeDTO` trả về từ API.
*   **Refactor:**
    *   Buyer xem danh sách vé: Đọc `inventoryMode` và `accessScope`.
    *   Nếu `inventoryMode == 'QUANTITY'`: Hiển thị ô chọn số lượng vé (Quantity Selector).
    *   Nếu `inventoryMode == 'ASSIGNED_SEAT'`: Khi buyer chọn vé $\rightarrow$ Mở popup hiển thị Sơ đồ ghế (Seat Map Canvas), cho phép click trực tiếp để chọn vị trí ghế (truyền `seatIds` vào Payload booking).
    *   Phase B: Chỉ cho phép giỏ hàng/booking request thuộc một vùng bán. Nếu buyer đã chọn vé ở một sector, UI không cho thêm vé từ sector khác hoặc EVENT-level ticket vào cùng checkout.

### FE4. Trang Checkout & Chi Tiết Đơn Hàng (Order Detail)
*   **Mô tả:** Hiển thị thông tin ghế đã chọn.
*   **Refactor:**
    *   Hiển thị chi tiết dòng sản phẩm trong giỏ hàng: `Khán đài A - VIP [Ghế A-01, A-02]`.
    *   Đối với vé đứng/GA: Chỉ hiển thị số lượng, ví dụ `Khán đài Fanzone x 2`.

### FE5. Xử lý lỗi Publish & Hidden Seat (Fix bug cũ)
*   **OrganizerSeatMapPage.tsx:**
    ```typescript
    } catch (err: any) {
      const status = err?.response?.status;
      if (status === 400 || status === 403) {
        toast.error(err?.response?.data?.message || "Không thể publish.");
        setPublishing(false);
        return;  // Không chạy poll loop
      }
    }
    ```
*   **seatMapEditorUtils.ts:**
    ```typescript
    hidden: seat.isActive === false,  // Thay vì: hidden: false
    ```
*   **organizerWorkspaceService.ts (nhận response trực tiếp):**
    ```typescript
    // Sửa publishSeatMap() trả SeatMapResponse thay vì void:
    const response = await api.post(`/organizer/events/${eventId}/seat-map/publish`, payload);
    return response.data;  // SeatMapResponse để cập nhật UI ngay lập tức
    ```


---

## Phân Công Restore

| Trường hợp | Method | Table/Cache |
|---|---|---|
| Zone cancel | `restoreStock()` | `ticket_types` ✅ |
| Seated cancel | `restoreStock()` | `ticket_types` ✅ |
| Seated cancel | `restoreSeatedStock()` | `event_seat_inventory` + Redis ✅ |
| ❌ TUYỆT ĐỐI KHÔNG | `restoreSeatedStock()` | `ticket_types` |

---

## Thứ Tự Thực Hiện

```
DB0                       (apply V7 + V8 cho database hiện hữu; V2 dùng cho fresh DB)
A0                        (config — 5 phút, làm đầu)
A1 → A2 → A3 → A4        (Repository/Entity)
A4b                       (EventSeat.ticketTypeId + colorCode)
A5(a,b,c)                 (SeatMapSyncService — cần A1, A2)
A6                         (publishSeatMap — cần A1–A5)
A7 → A8                   (Controller)
A9                         (TicketTypeDTO public)
A10                        (TicketTypeService validation)
    ↓
✅ Deploy Phase A
    ↓
B0                         (OrderItemSeat entity/repo — bắt buộc trước B2)
B1                         (TicketReservationService — nhanh)
B2                         (SeatBookingService — cần A3, A4, B0, B1)
B3                         (BookingService routing — cần B2)
B4 → B5                   (Issuance + Restore)
    ↓
✅ Deploy Phase B
    ↓
FE1 + FE2 + FE3
```

> [!WARNING]
> **B3 là điểm rủi ro nhất.** Thứ tự `reserveSeats()` PHẢI sau `orderRepository.save()`. Chạy zone regression test ngay sau B3.

> [!IMPORTANT]
> `[PERF]` items không bắt buộc cho MVP nhưng bắt buộc trước flash sale. Xem SYSTEM_DESIGN_NOTES.md §8, §9.

---

## Edge Cases

| Edge Case | Xử lý |
|---|---|
| Xóa ghế SOLD/RESERVED | A6 — Chốt 1 |
| Xóa sector có vé | A6 — Chốt 2 |
| Ẩn sector/seat | A6 — `setIsActive(false)` (không có `setVisible`) |
| Đổi TicketType của ghế có RESERVED/SOLD | Chặn đổi `event_seats.ticket_type_id` |
| Đổi TicketType của ghế chỉ AVAILABLE/BLOCKED | Cho đổi và sync snapshot `event_seat_inventory.ticket_type_id` |
| ticketTypeId thuộc event khác | A10 (validation lúc save Ticket Type) |
| Ghế active SEATED chưa gán ticketTypeId | A6/A10 — reject publish hoặc sale-readiness |
| Một SEATED sector có nhiều ASSIGNED_SEAT TicketType | Hợp lệ — phân biệt bằng `event_seats.ticket_type_id`, không dùng `sector.mapData.ticketTypeId` |
| Hidden seat mất khi reload | A5b + FE2 |
| FE nuốt lỗi 400 | FE1 |
| IDOR getSeatMap | A5a + `findByIdAndOrganizerIdAndIsDeletedFalse` [VERIFIED] |
| Counter UI lệch | BookingService gọi counterService.reserveCounter() một lần duy nhất |
| Double-count restore | B2.restoreSeatedStock() không touch ticket_types |
| orderId không có khi reserveSeats | B3 — reserveSeats() gọi SAU save() |
| OrderItemSeat không có entity | B0 — tạo mới trước B2 |
| Oversell ghế (2 user cùng ghế) | A3 atomic WHERE status=AVAILABLE + B2 check updated=0 |
| Deadlock | B1/B3 — Global Lock Ordering (lexicographical sorting toàn bộ seat & zone keys) |
| Livelock | Lock key sorting + waitTime = 100ms |
| Throughput thấp flash sale | B1 waitTime = 100ms (§8.2) |
| Hot read seat status | A5c + A6 Redis HSET cache (§9) |
| saveAll() chậm | A0 + A3 bulk SQL |
| Mixed order / nhiều zone trong cùng booking | B3 — Phase B reject bằng `validateSingleBookingZone()` |
| Buyer đọc DRAFT | A5c check PUBLISHED |
| Redis chết | A5c fallback DB + re-warm |
| DB-Redis lệch | DB = source of truth |
| setVisible không compile | Fix: dùng setIsActive(false) [VERIFIED from entity] |
| Config sai file | Fix: core-service.yml in configserver [A0] |
| eventSectorId cross-event | A10 — validate scope |
| FE poll sau publish thành công | FE3 — dùng response trực tiếp |
| Fresh setup Flyway fail | Fix: V2 FK constraints ở cuối file, V7 bọc DO $$ IF NOT EXISTS |
| Seated quantity nhập sai | A10 override từ active seats count, FE read-only |
