# Quy Chuẩn Kỹ Thuật & Đặc Tả Triển Khai Phân Hệ Vé Ghế Ngồi (Seated Ticket Specification)

Tài liệu này là quy chuẩn kỹ thuật chính thức và hướng dẫn triển khai (Implementation Standard) thiết lập cấu trúc dữ liệu, luồng nghiệp vụ, API contract, và cơ chế concurrency cho cả hai hình thức bán vé: **Vé Ghế Ngồi (Seated Ticket)** và **Vé Khu Vực (Zone/Standing Ticket)** trên hệ thống **Flash Ticket**.

---

## 1. Mô Mô Hình Thực Thể & Tiến Trình DB Schema (Phase B)

Để tránh nợ kỹ thuật và mơ hồ trong phát triển, chúng ta thực hiện nâng cấp database schema ngay trong Phase B. Việc nâng cấp được triển khai ở hai cấp độ:

### 1.1. Cập nhật cho Cơ sở Dữ liệu Mới (Fresh Setup)
Cập nhật trực tiếp file [V2__complete_schema.sql](file:///d:/Project/flash-ticket-system/database/postgresql/V2__complete_schema.sql) để khi thiết lập môi trường mới từ đầu, cấu trúc bảng `event_schema.ticket_types` chứa sẵn các cột chuẩn kèm check constraints. 
Để tránh lỗi tham chiếu chéo (circular/forward references) khi bảng `ticket_types` và `event_seat_inventory` được tạo trước bảng `event_sectors` và `event_seats`, chúng ta loại bỏ các ràng buộc khóa ngoại (Foreign Key) khai báo inline tại các bảng này và chuyển chúng xuống cuối file dưới dạng câu lệnh `ALTER TABLE`.

*   **Tại bảng `event_schema.ticket_types`:**
    ```sql
    CREATE TABLE event_schema.ticket_types (
        id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
        event_id UUID NOT NULL REFERENCES event_schema.events(id) ON DELETE CASCADE,
        event_sector_id UUID, -- Loại bỏ REFERENCES inline
        name VARCHAR(100) NOT NULL,
        -- ...
        seat_selection_enabled BOOLEAN DEFAULT FALSE,
        inventory_mode VARCHAR(50) DEFAULT 'QUANTITY' NOT NULL,
        access_scope VARCHAR(50) DEFAULT 'EVENT' NOT NULL,
        -- ...
        CONSTRAINT chk_tt_inventory_mode CHECK (inventory_mode IN ('QUANTITY', 'ASSIGNED_SEAT')),
        CONSTRAINT chk_tt_access_scope CHECK (access_scope IN ('EVENT', 'SECTOR'))
    );
    ```

*   **Tại bảng `event_schema.event_seat_inventory`:**
    ```sql
    CREATE TABLE event_schema.event_seat_inventory (
        id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
        event_id UUID NOT NULL REFERENCES event_schema.events(id) ON DELETE CASCADE,
        event_seat_id UUID NOT NULL, -- Loại bỏ REFERENCES inline
        ticket_type_id UUID REFERENCES event_schema.ticket_types(id) ON DELETE SET NULL,
        status VARCHAR(50) DEFAULT 'AVAILABLE' NOT NULL
        -- ...
    );
    ```

*   **Tại cuối file `V2__complete_schema.sql`:**
    ```sql
    ALTER TABLE event_schema.ticket_types 
        ADD CONSTRAINT fk_ticket_types_sector_id 
        FOREIGN KEY (event_sector_id) 
        REFERENCES event_schema.event_sectors(id) 
        ON DELETE SET NULL;

    ALTER TABLE event_schema.event_seat_inventory 
        ADD CONSTRAINT fk_esi_seat_id 
        FOREIGN KEY (event_seat_id) 
        REFERENCES event_schema.event_seats(id) 
        ON DELETE CASCADE;
    ```

### 1.2. Bản vá cho Hệ thống đã Deploy (Existing Setup & Data Cleanup)
Tạo file migration `database/postgresql/V7__add_ticket_type_fields.sql` sử dụng khối lệnh `DO $$` để đảm bảo Flyway không bị lỗi trùng lặp ràng buộc khi chạy lại trên các cơ sở dữ liệu đã cập nhật:
```sql
-- Thêm trường mới cho ticket_types
ALTER TABLE event_schema.ticket_types 
    ADD COLUMN IF NOT EXISTS inventory_mode VARCHAR(50) DEFAULT 'QUANTITY' NOT NULL,
    ADD COLUMN IF NOT EXISTS access_scope VARCHAR(50) DEFAULT 'EVENT' NOT NULL;

-- Thêm check constraint an toàn (chỉ thêm nếu chưa tồn tại)
DO $$
BEGIN
    IF NOT EXISTS (SELECT 1 FROM pg_constraint WHERE conname = 'chk_tt_inventory_mode') THEN
        ALTER TABLE event_schema.ticket_types
            ADD CONSTRAINT chk_tt_inventory_mode CHECK (inventory_mode IN ('QUANTITY', 'ASSIGNED_SEAT'));
    END IF;
    
    IF NOT EXISTS (SELECT 1 FROM pg_constraint WHERE conname = 'chk_tt_access_scope') THEN
        ALTER TABLE event_schema.ticket_types
            ADD CONSTRAINT chk_tt_access_scope CHECK (access_scope IN ('EVENT', 'SECTOR'));
    END IF;
END $$;

-- Đồng bộ dữ liệu cũ (Backward Compatibility)
UPDATE event_schema.ticket_types
SET inventory_mode = 'ASSIGNED_SEAT'
WHERE seat_selection_enabled = TRUE;

UPDATE event_schema.ticket_types
SET access_scope = 'SECTOR'
WHERE event_sector_id IS NOT NULL;

-- ============================================================================
-- Dọn dẹp dữ liệu rác (Post-Migration Cleanup & Validation)
-- ============================================================================

-- 1. Chuyển ASSIGNED_SEAT mà thiếu sector_id về QUANTITY
UPDATE event_schema.ticket_types
SET inventory_mode = 'QUANTITY', seat_selection_enabled = FALSE
WHERE inventory_mode = 'ASSIGNED_SEAT' AND event_sector_id IS NULL;

-- 2. Chuyển ASSIGNED_SEAT liên kết nhầm với STANDING sector về QUANTITY
UPDATE event_schema.ticket_types tt
SET inventory_mode = 'QUANTITY', seat_selection_enabled = FALSE
FROM event_schema.event_sectors s
WHERE tt.event_sector_id = s.id 
  AND s.sector_type = 'STANDING' 
  AND tt.inventory_mode = 'ASSIGNED_SEAT';

-- 3. Tạm thời ẩn các Ticket Type liên kết với các sector_type chưa hỗ trợ (VIP_BOX, ACCESSIBLE)
UPDATE event_schema.ticket_types tt
SET status = 'HIDDEN', is_visible = false
FROM event_schema.event_sectors s
WHERE tt.event_sector_id = s.id
  AND s.sector_type IN ('VIP_BOX', 'ACCESSIBLE');
```

---

## 2. Giao Diện & Trải Nghiệm Người Dùng (FE Contract)

Đặc tả hoạt động của Frontend trên các luồng nghiệp vụ:

### 2.1. Organizer Portal (Quản lý thiết lập Event)
*   **Tạo/Sửa Vé (Ticket Type Wizard):**
    *   Trường chọn **Access Scope** gồm 2 tùy chọn: `Toàn sự kiện (EVENT)` và `Khán đài cụ thể (SECTOR)`.
    *   Nếu chọn `SECTOR` $\rightarrow$ Dynamic dropdown bắt buộc chọn một Sector đang Active thuộc Event.
    *   Trường chọn **Inventory Mode** gồm 2 tùy chọn: `Mua theo số lượng (QUANTITY)` và `Chọn vị trí ghế (ASSIGNED_SEAT)`.
    *   **Ràng buộc Seated Ticket:** Nếu chọn `ASSIGNED_SEAT`, trường nhập **Total Quantity** (Số lượng vé phát hành) sẽ bị khóa/read-only trên giao diện. Giá trị này sẽ được tự động tính và điền bởi hệ thống dựa trên số lượng ghế thực tế đang hoạt động của khán đài.
*   **Thiết lập Sơ đồ (Seat Map Editor):**
    *   Đối với mỗi Sector được vẽ trên Konva Canvas, Organizer chọn loại khu vực (`sector_type`):
        *   `SEATED`: Bắt buộc vẽ hoặc sinh các ghế vật lý có nhãn tọa độ.
        *   `STANDING`: Không cho phép sinh ghế. Bắt buộc nhập trường **Total Capacity** (Sức chứa tối đa của khu vực).

### 2.2. Buyer Portal (Mua vé)
*   **Trường hợp 1: Vé tự do (Access Scope = EVENT + Inventory Mode = QUANTITY hoặc Access Scope = SECTOR + SectorType = STANDING + Inventory Mode = QUANTITY)**
    *   Giao diện hiển thị nút tăng/giảm số lượng đơn giản (Quantity Selector).
    *   Payload gửi lên Booking API không chứa mảng `seatIds` (hoặc mảng rỗng).
*   **Trường hợp 2: Vé chọn chỗ (Access Scope = SECTOR + SectorType = SEATED + Inventory Mode = ASSIGNED_SEAT)**
    *   Hiển thị canvas sơ đồ ghế ngồi trực quan.
    *   Các ghế có trạng thái từ Cache/DB là `RESERVED`, `SOLD` hoặc `BLOCKED` sẽ hiển thị màu xám khóa (disabled).
    *   Buyer click chọn tối đa `N` ghế (N = giới hạn vé tối đa mỗi đơn hàng).
    *   Payload gửi lên Booking API bắt buộc có mảng `seatIds` có chiều dài bằng đúng trường `quantity`.

---

## 3. Quy Quy Trình Thiết Lập & Quy Tắc Ràng Buộc Khán Đài

### 3.1. Giải quyết Vòng Lặp Thiết Lập (Sector vs TicketType)
Để giải quyết mâu thuẫn "muốn vẽ sơ đồ cần chọn loại vé, muốn tạo loại vé cần liên kết sector", quy trình thiết lập được chuẩn hóa như sau:
1.  **Bước 1 (Vẽ sơ đồ):** Organizer tạo Layout và vẽ các Sector vật lý trong Seat Map Editor. Gán `sector_type` (`SEATED` / `STANDING`), chưa cần gán `ticketTypeId` (để trống). Tiến hành lưu và Publish sơ đồ lần đầu $\rightarrow$ Hệ thống sinh UUID cho các Sector trong DB. Với khán đài `SEATED`, các record trong `event_seat_inventory` được tạo với `ticket_type_id = NULL` và `status = 'AVAILABLE'`.
2.  **Bước 2 (Tạo vé):** Organizer vào danh sách Ticket Type, tạo các Ticket Type với cấu hình `accessScope = SECTOR` và chọn Sector tương ứng từ danh sách dropdown đã sinh ở Bước 1. Lúc này, DB ghi nhận `TicketType.eventSectorId = sectorId` làm **Source of Truth**.
3.  **Bước 3 (Auto-Sync khi liên kết):** Sau khi Ticket Type được lưu, hệ thống tự động chạy câu lệnh SQL để cập nhật giá trị `ticket_type_id` trong bảng `event_seat_inventory` đối với các ghế chưa bán hoặc đang khóa vật lý thuộc Sector đó:
    ```sql
    UPDATE event_schema.event_seat_inventory
    SET ticket_type_id = :ticketTypeId
    WHERE event_seat_id IN (SELECT id FROM event_schema.event_seats WHERE sector_id = :sectorId)
      AND (ticket_type_id IS NULL OR status IN ('AVAILABLE', 'BLOCKED'));
    ```
4.  **Bước 4 (Cập nhật UI Helper):** Hệ thống tự động ghi đè thông tin `map_data -> 'ticketTypeId'` trong bảng `event_sectors` để đồng bộ UI Editor.
    > [!IMPORTANT]
    > **Ignored FE mapData.ticketTypeId:** Khi Frontend thực hiện POST `/publish` sơ đồ ghế, Backend sẽ hoàn toàn bỏ qua hoặc ghi đè thuộc tính `mapData.ticketTypeId` được gửi lên từ client. Source of truth duy nhất để xác định loại vé của Sector luôn là liên kết quan hệ thực thể `TicketType.eventSectorId` trong database.
    >
    > **Publish sau khi đã liên kết Ticket Type:** Nếu sơ đồ ghế được chỉnh sửa và Publish lại sau khi Ticket Type đã được liên kết với Sector, Backend khi tạo các ghế mới sinh ra trong DB sẽ tự động điền `ticket_type_id` bằng ID của Ticket Type đang liên kết với Sector đó.

### 3.2. Giới Hạn Nghiệp Vụ MVP (Sector vs Ticket Type)
*   **MVP Constraint 1:** **Mỗi khán đài ghế ngồi (`SEATED` sector) chỉ được liên kết với tối đa 1 loại vé chọn chỗ (`ASSIGNED_SEAT` ticket type) ở trạng thái hoạt động (active).**
    *   *Ý nghĩa:* Ngăn chặn việc cấu hình nhiều giá vé khác nhau trong cùng một khán đài vật lý mà không có phân lớp hàng ghế trong MVP.
    *   *Giải pháp thay thế:* Nếu Organizer muốn cấu hình vé VIP ở các hàng đầu và Regular ở các hàng sau, họ phải vẽ tách thành 2 sector riêng biệt (ví dụ: `Khán đài A - VIP` và `Khán đài A - Regular`).
*   **MVP Constraint 2 (Seated Quantity Derivation):** Khi Tạo hoặc Cập nhật Ticket Type có `inventoryMode = ASSIGNED_SEAT`, Backend sẽ **bỏ qua hoặc từ chối** trường `quantityTotal` gửi lên từ Client. Số lượng vé phát hành bắt buộc phải được lấy tự động từ số lượng ghế đang hoạt động của sector:
    $$\text{quantityTotal} = \text{event\_seats.count(isActive = true, sectorId = eventSectorId)}$$

### 3.3. Ràng Buộc Đổi Loại Vé Liên Kết của Seated Sector
Để tránh làm sai lệch trạng thái ghế đã giữ hoặc đã bán của một sector khi thay đổi Ticket Type liên kết:
*   **Quy tắc:** Trước khi thay đổi liên kết Ticket Type của một `SEATED` sector:
    *   Nếu trong `event_seat_inventory` của sector đó đã tồn tại bất kỳ ghế nào ở trạng thái `RESERVED` hoặc `SOLD` $\rightarrow$ Hệ thống lập tức từ chối và trả lỗi HTTP 400 (`"Không thể thay đổi loại vé của khán đài khi đã phát sinh vé đặt hoặc vé bán"`).
    *   Nếu toàn bộ ghế ở trạng thái `AVAILABLE` (hoặc `BLOCKED`, `NULL`) $\rightarrow$ Cho phép cập nhật lại `ticket_type_id` mới sang Ticket Type mới.

### 3.4. Định Nghĩa và Ràng Buộc Sức Chứa (Capacity Rules)

#### A. Bản chất trường `events.total_capacity`
*   Theo logic nghiệp vụ hiện có của Core Service (gọi qua `adjustTotalCapacity`), trường `events.total_capacity` được định nghĩa là **Tổng quota vé đã phát hành của tất cả các Ticket Type**.
*   Do đó, hệ thống không dùng `events.total_capacity` làm hard-cap vật lý độc lập tại cấp độ sự kiện, mà nó phản ánh động tổng sức chứa khả dụng của sự kiện dựa trên tổng số vé phát hành:
    $$\text{events.total\_capacity} = \sum (\text{TicketType.quantityTotal của sự kiện})$$

#### B. Đối với Khán Đài Đứng (Access Scope = SECTOR, SectorType = STANDING)
*   Trong API publish seat map, payload `SeatMapPublishRequest.SectorPayload` bổ sung trường `totalCapacity` kiểu `Integer`.
*   Khi publish, hệ thống ghi nhận `totalCapacity` vào bảng `event_sectors`.
*   **Quy tắc nhiều Ticket Type dùng chung Standing Sector:** Tổng số lượng vé của tất cả các Ticket Type liên kết với Standing Sector đó không được vượt quá sức chứa khán đài:
        $$\sum (\text{quantityTotal của các TicketType liên kết với Sector } X) \le \text{event\_sectors.total\_capacity của Sector } X$$
*   Nếu vượt quá $\rightarrow$ Trả lỗi HTTP 400: `INVALID_TICKET_CONFIG` (`"Tổng số vé của các loại vé vượt quá sức chứa tối đa của khán đài STANDING"`).
*   **STANDING sectors không tạo dữ liệu trong bảng `event_seat_inventory`.**

#### C. Đối với Khán Đài Ngồi (Access Scope = SECTOR, SectorType = SEATED)
*   Giá trị `quantityTotal` của Ticket Type có `inventoryMode = ASSIGNED_SEAT` phải đồng bộ với số lượng ghế hoạt động (`isActive = true`) trong khán đài tương ứng.
*   **Công thức Safe-Sync khi Publish Sơ đồ sau khi mở bán:**
    *   Lấy số lượng ghế hoạt động hiện tại trên sơ đồ: `activeSeatCount`.
    *   Đếm số lượng ghế đã bán trong DB: `soldCount = count(event_seat_inventory where status = 'SOLD' for sector)`.
    *   Đếm số lượng ghế đang giữ trong DB: `reservedCount = count(event_seat_inventory where status = 'RESERVED' for sector)`.
    *   Nếu `activeSeatCount < soldCount + reservedCount` $\rightarrow$ Trả lỗi HTTP 400 và chặn Publish (`"Không thể cập nhật sơ đồ vì số ghế hoạt động nhỏ hơn lượng ghế đã bán và đang giữ chỗ"`).
    *   Nếu hợp lệ, Backend tự động cập nhật:
        *   `TicketType.quantityTotal = activeSeatCount`
        *   `TicketType.quantityAvailable = activeSeatCount - soldCount - reservedCount`
        *   `TicketType.quantityReserved = reservedCount`

### 3.5. Ràng Buộc Sửa Đổi Khán Đài (Safe Mode Updates)
Khi thực hiện cập nhật hoặc sửa đổi khán đài đã được bán hoặc giữ vé, hệ thống áp dụng các chốt chặn sau:
*   **Khi giảm Capacity của Standing Sector:** Hệ thống kiểm tra tổng lượng vé đã bán + đang giữ của tất cả các loại vé dùng chung sector đó:
    $$\text{newTotalCapacity} \ge \sum (\text{quantityTotal} - \text{quantityAvailable} \text{ của các TicketType thuộc Sector } X)$$
    Nếu không thỏa mãn $\rightarrow$ Trả lỗi HTTP 400 chặn cập nhật.
*   **Không được thay đổi `sectorType`** (ví dụ: chuyển từ `SEATED` sang `STANDING` hoặc ngược lại) nếu sector đó đã có bất kỳ vé nào ở trạng thái `RESERVED` hoặc `SOLD`.
*   **Không được ẩn hoặc xóa mềm sector/ghế** (chuyển `isActive = false`) nếu sector/ghế đó đang chứa vé ở trạng thái `RESERVED` hoặc `SOLD`.

---

## 4. Ranh Giới Dịch Vụ Đếm Tồn Kho (TicketInventoryCounterService)

Dịch vụ `TicketInventoryCounterService` là cửa ngõ duy nhất thực hiện việc trừ, cộng và hoàn trả tồn kho số lượng trên bảng `ticket_types` (và `inventory_pools` trong tương lai).

### 4.1. Quy tắc phân nhiệm (Single Update Rule)
*   **Chỉ thực hiện trừ counter 1 lần duy nhất:** Việc trừ counter `quantity_available` và tăng `quantity_reserved` chỉ được gọi bởi `BookingService` thông qua `counterService.reserveCounter()` ở pha Orchestration chung (sau khi đã lock toàn bộ tài nguyên thành công nhưng trước khi tạo đơn hàng).
*   **Chiến dịch Seated Reservation:** Khi gọi `seatBookingService.reserveSeats()`, hệ thống chỉ ghi nhận trạng thái dòng trong `event_seat_inventory` và chèn dữ liệu `order_item_seats`, **tuyệt đối không được tác động lại counter lần thứ hai**.

### 4.2. Giao diện Counter Service
```java
@Service
@RequiredArgsConstructor
public class TicketInventoryCounterService {
    private final TicketTypeRepository ticketTypeRepository;

    @Transactional
    public void reserveCounter(UUID ticketTypeId, int amount) {
        int updated = ticketTypeRepository.decrementAvailableAndIncrementReserved(ticketTypeId, amount);
        if (updated == 0) {
            throw new InsufficientStockException("Vé đã hết hàng hoặc không đủ tồn kho");
        }
    }

    @Transactional
    public void restoreCounter(UUID ticketTypeId, int amount) {
        ticketTypeRepository.restoreQuantity(ticketTypeId, amount);
    }

    @Transactional
    public void confirmSold(UUID ticketTypeId, int amount) {
        ticketTypeRepository.decrementReserved(ticketTypeId, amount);
    }
}
```

---

## 5. Ranh Giới Chiến Lược Giữ Chỗ & Khóa Toàn Cục (Lock Ordering)

Hệ thống điều phối xử lý qua `InventoryReservationStrategy` dựa trên `TicketType.inventoryMode`:

*   `BookingService` chịu trách nhiệm mở rộng Transaction, điều phối gọi các pha của Strategy, gọi `TicketInventoryCounterService` và lưu thông tin `Order` gốc.
*   Các Strategy tự quản lý cơ chế khóa (Lock), xác thực sau khóa (Pre-check), và lưu giữ chi tiết chỗ (Record). Không được thiết kế một phương thức `book()` tổng nuốt chửng cả luồng nghiệp vụ của `BookingService`.

### 5.1. Cơ chế Khóa Toàn Cục Tránh Deadlock trên Mixed Order
Để tránh hiện tượng deadlock/circular wait khi một đơn hàng chứa cả vé số lượng (Zone Lock) và vé chọn ghế (Seat Lock), chúng ta thiết lập cơ chế **Sắp xếp khóa toàn cục (Global Lock Ordering)**:
1.  **Thu thập toàn bộ Lock Keys của Đơn hàng:**
    *   Với mỗi item đặt vé số lượng (Fanzone/GA): Tạo key `lock:zone:{ticketTypeId}`
    *   Với mỗi ghế ngồi đặt cụ thể: Tạo key `lock:seat:{eventId}:{seatId}`
2.  **Sắp xếp Thứ Tự Bảng Chữ Cái (Lexicographical Sorting):** Gom toàn bộ danh sách Lock Keys thành một mảng phẳng và tiến hành sort lexicographically:
    ```java
    List<String> sortedLockKeys = allLockKeys.stream().sorted().toList();
    ```
3.  **Acquire Lock Tuần Tự:** Duyệt qua danh sách `sortedLockKeys` và tiến hành acquire lock tuần tự thông qua Redisson client với `waitTime = 100ms` và `leaseTime = 15s`. Nếu có bất kỳ lock nào fail $\rightarrow$ Giải phóng toàn bộ các lock đã lấy trước đó và ném ra ngoại lệ đặt chỗ thất bại (Fail-Fast).

---

## 6. Đặc Tả API Contracts (Request & Response)

### 6.1. Tạo/Cập nhật Ticket Type (POST/PUT)
*   **API Endpoint:**
    *   `POST /api/organizer/events/{eventId}/ticket-types`
    *   `PUT /api/organizer/events/{eventId}/ticket-types/{ticketTypeId}`
*   **Request Body JSON:**
```json
{
  "name": "Khán đài A - VIP",
  "price": 1200000,
  "inventoryMode": "ASSIGNED_SEAT",
  "accessScope": "SECTOR",
  "eventSectorId": "a823e20e-c21b-43d9-a292-96420556e9c4",
  "saleStartDatetime": "2026-06-01T00:00:00Z",
  "saleEndDatetime": "2026-06-15T23:59:59Z"
}
```

### 6.2. DTO Response Contract
*   **TicketTypeDTO (Public & Organizer):** Trả thêm các trường định dạng nghiệp vụ bán để Frontend quyết định hiển thị sơ đồ hoặc số lượng:
```json
{
  "id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "name": "Khán đài A - VIP",
  "price": 1200000,
  "inventoryMode": "ASSIGNED_SEAT",
  "accessScope": "SECTOR",
  "eventSectorId": "a823e20e-c21b-43d9-a292-96420556e9c4",
  "quantityAvailable": 148
}
```
*   **SeatMapResponse.SectorDto (Public & Organizer):** Trả thông tin phân loại khán đài và cấu trúc sức chứa:
```json
{
  "id": "a823e20e-c21b-43d9-a292-96420556e9c4",
  "name": "Khán đài A",
  "sectorType": "STANDING",
  "totalCapacity": 500,
  "isActive": true,
  "seatsData": []
}
```

### 6.3. Publish Seat Map
*   **API Endpoint:** `POST /api/organizer/events/{eventId}/seat-map/publish`
*   **Request Body JSON:**
```json
{
  "name": "Sơ đồ A",
  "backgroundImageUrl": "https://...",
  "backgroundWidth": 800,
  "backgroundHeight": 600,
  "mapConfig": {},
  "sectors": [
    {
      "id": "a823e20e-c21b-43d9-a292-96420556e9c4",
      "name": "Khán đài A",
      "code": "KDA",
      "sectorType": "STANDING",
      "colorCode": "#FF5733",
      "visible": true,
      "displayOrder": 1,
      "totalCapacity": 500,
      "seats": []
    }
  ]
}
```

### 6.4. Đặt vé (Create Booking)
*   **API Endpoint:** `POST /api/bookings`
*   **Request Body JSON:**
```json
{
  "eventId": "f3b392ac-7922-4bb3-bc52-16e6f6630f9a",
  "items": [
    {
      "ticketTypeId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
      "quantity": 2,
      "seatIds": [
        "c3809e2e-2a94-4d8b-967a-1153a5c531d0",
        "d8909e2e-3b94-5d8b-867a-2253a5c531e1"
      ]
    }
  ],
  "promotionCode": "EARLY10"
}
```
*   **Validation & Rejection Rules tại Backend:**
    *   **Chống trùng lặp Request:** Reject nếu cùng một `seatId` xuất hiện nhiều hơn một lần trong cùng một Request Payload.
    *   **Chống trùng lặp Item:** Reject nếu cùng một `ticketTypeId` xuất hiện ở nhiều item khác nhau trong mảng `items`. Giao diện hoặc Controller phải gom nhóm/merge trước khi gọi Service xử lý.
    *   **Kiểm tra phạm vi sự kiện:** Reject nếu bất kỳ `seatId` nào không thuộc về `eventId` đang đặt.
    *   **Kiểm tra phạm vi khán đài:** Reject nếu `seatId` không thuộc về `eventSectorId` liên kết với `ticketTypeId` tương ứng.
    *   **Kiểm tra trạng thái ghế:** Reject nếu trạng thái ghế hiện tại khác `AVAILABLE` (ví dụ: `RESERVED`, `SOLD`, `BLOCKED`, `LOCKED` từ Redisson).

---

## 7. Chi Tiết Cấu Hình Cache & Lock Phân Tán

### 7.1. Lock Spec (Phòng Deadlock & Livelock)
*   **Lexicographical Sorting:** Trước khi acquire lock cho danh sách ghế, Java code bắt buộc phải sort danh sách `seatIds` theo thứ tự bảng chữ cái UUID string.
*   **Wait & Lease Duration:**
    *   `waitTime = 100ms` (Tránh treo luồng Tomcat, chống flaky test do trễ mạng hoặc hạ tầng quá tải).
    *   `leaseTime = 15s` (Tự động giải phóng khóa nếu Node xử lý bị crash đột ngột).

### 7.2. Cache Spec (Redisson RMapCache)
*   **Key Format:** `seat_status:{eventId}`
*   **Kiểu dữ liệu:** `RMapCache<UUID, String>` (Key: Seat UUID, Value: Status String).
*   **Status Values:** `AVAILABLE`, `RESERVED`, `SOLD`, `BLOCKED`.
    *   `BLOCKED`: Dùng cho các ghế bị Organizer tạm khóa/không bán (ghế hư hỏng, ghế khách mời ngoại giao). Ghế này sẽ hiển thị disabled trên sơ đồ.
    *   > [!NOTE]
    *   **Transient Lock Handling:** Trạng thái `LOCKED` (đang xử lý thanh toán tạm thời trong tích tắc 100ms) được quản lý độc lập bởi khóa phân tán của Redisson (`lock:seat:{eventId}:{seatId}`). Trạng thái này không được ghi đè vào cache map dài hạn `seat_status` nhằm giảm tải lưu trữ cho Redis.
*   **Fallback logic khi Cache Miss:**
    *   Nếu Cache trống hoặc key không tồn tại $\rightarrow$ Truy vấn trực tiếp PostgreSQL `event_seat_inventory` theo `eventId`.
    *   Đồng bộ kết quả từ DB vào Redis qua lệnh `putAll()`, thiết lập TTL **48 giờ** cho toàn bộ Map.
*   **Nạp ấm Cache (Warm-up Cache) An Toàn:** Khi thực hiện nạp ấm cache cho event, hệ thống truy vấn trực tiếp trạng thái thực tế từ `event_seat_inventory` trong DB để nạp vào Redis, **tuyệt đối không tự ý ghi đè trạng thái 'AVAILABLE' lên các ghế đã có trạng thái khác**.
*   **Sự kiện Ghi Cache:** Thực hiện bất đồng bộ ngoài Transaction chính bằng `@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)`.

---

## 8. Vòng Đời Đơn Hàng & Ranh Giới Giao Dịch (Transaction Boundary)

### 8.1. Ranh giới Transaction
Toàn bộ quá trình xử lý đặt vé trong phương thức `BookingService.doBooking` phải chạy trong **cùng một DB Transaction**. Nếu bất kỳ bước nào thất bại, toàn bộ dữ liệu phải tự động rollback để đảm bảo tính nhất quán.

### 8.2. Ma Trận Chuyển Đổi Trạng Thái Hệ Thống (State Machine)

| Trạng thái Đơn hàng (`orders.status`) | Hành động trên Counter (`TicketType`) | Trạng thái Ghế DB (`event_seat_inventory`) | Trạng thái Ghế Cache (`Redis`) | Trạng thái Chi tiết Ghế (`order_item_seats.status`) |
| :--- | :--- | :--- | :--- | :--- |
| **1. Khởi tạo đơn (`PENDING`)** | Counter giảm:<br>`available -= N`<br>`reserved += N` | UPDATE status = `RESERVED`, `order_id = currentOrderId` | HSET $\rightarrow$ `RESERVED` *(AFTER_COMMIT)* | INSERT dòng mới với trạng thái `RESERVED` |
| **2. Thanh toán thành công (`CONFIRMED`)** | Giảm reserved:<br>`reserved -= N` | UPDATE status = `SOLD` | HSET $\rightarrow$ `SOLD` *(AFTER_COMMIT)* | UPDATE status = `CONFIRMED` |
| **3. Đơn hết hạn (TTL 15m) (`EXPIRED`)** | Hoàn counter:<br>`available += N`<br>`reserved -= N` | UPDATE status = `AVAILABLE`, `order_id = null` | HSET $\rightarrow$ `AVAILABLE` *(AFTER_COMMIT)* | UPDATE status = `CANCELLED` |
| **4. Hủy đơn chủ động (`CANCELLED`)** | Hoàn counter:<br>`available += N`<br>`reserved -= N` | UPDATE status = `AVAILABLE`, `order_id = null` | HSET $\rightarrow$ `AVAILABLE` *(AFTER_COMMIT)* | UPDATE status = `CANCELLED` |
| **5. IPN Trùng lặp (Idempotent)** | *Bỏ qua* | *Bỏ qua* | *Bỏ qua* | *Bỏ qua* |

*Lưu ý về tính Idempotent:* Trước khi thực hiện bất kỳ cập nhật kho hay ghế nào trong callback payment, bắt buộc kiểm tra trạng thái hiện tại của đơn hàng. Nếu đơn hàng đã ở trạng thái `CONFIRMED`, `CANCELLED` hoặc `EXPIRED` $\rightarrow$ Bỏ qua xử lý để tránh double-count.
*Tương thích ngược:* Để tương thích với các logic cũ, sau khi cập nhật `inventoryMode`, hệ thống tự động đồng bộ giá trị sang trường deprecated:
    $$\text{seat\_selection\_enabled} = (\text{inventoryMode} == \text{ASSIGNED\_SEAT})$$

---

## 9. Ma Trận Tương Thích & Chính Sách Xác Thực (CompatibilityPolicy)

Hệ thống chỉ hỗ trợ hai loại hình khu vực vật lý là `SEATED` và `STANDING` trong Phase B. Các loại hình phụ trợ được cấu hình từ trước sẽ bị loại bỏ hoặc tạm khóa tại tầng Backend.

| accessScope | sectorType (DB) | inventoryMode | Trạng thái | Mã lỗi / Message | Ý nghĩa |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **EVENT** | `null` | **QUANTITY** | ✅ Hợp lệ | | Vé tự do toàn bộ sự kiện. |
| **EVENT** | `null` | **ASSIGNED_SEAT** | ❌ Bị chặn | `INVALID_TICKET_CONFIG`<br>Chọn ghế bắt buộc phải có thông tin khu vực | Không thể chọn số ghế khi không biết ghế nằm ở khán đài nào. |
| **SECTOR** | `STANDING` | **QUANTITY** | ✅ Hợp lệ | | Vé đứng tự do trong một khu vực (Fanzone). |
| **SECTOR** | `STANDING` | **ASSIGNED_SEAT** | ❌ Bị chặn | `INVALID_TICKET_CONFIG`<br>Khu vực đứng không hỗ trợ chọn ghế | Khán đài đứng không có ghế vật lý để chọn. |
| **SECTOR** | `SEATED` | **ASSIGNED_SEAT** | ✅ Hợp lệ | | Vé ngồi chọn số ghế chính xác. |
| **SECTOR** | `SEATED` | **QUANTITY** | ❌ Bị chặn | `UNSUPPORTED_FEATURE`<br>Khán đài SEATED chỉ hỗ trợ chọn ghế | Trong MVP, không cho phép bán vé không số ghế trên khán đài có cấu hình ghế ngồi. |
| **SECTOR** | `VIP_BOX` | *Any* | ❌ Bị chặn | `UNSUPPORTED_FEATURE`<br>Khu vực VIP_BOX chưa hỗ trợ trong MVP | Tạm khóa trong MVP để giữ độ sạch của nghiệp vụ. Trả lỗi từ chối ngay lập tức. |
| **SECTOR** | `ACCESSIBLE`| *Any* | ❌ Bị chặn | `UNSUPPORTED_FEATURE`<br>Khu vực ACCESSIBLE chưa hỗ trợ trong MVP | Tạm khóa. Trong tương lai, `ACCESSIBLE` sẽ là một `seatType` thay vì `sectorType`. |

---

## 10. Cấu Trúc Bảng Ánh Xạ `order_item_seats` (Khớp V2 Schema)

Việc ghi nhận chi tiết ghế đã đặt vào cơ sở dữ liệu phải tuân thủ chính xác 100% cấu trúc của bảng `booking_schema.order_item_seats` được khai báo trong file [V2__complete_schema.sql](file:///d:/Project/flash-ticket-system/database/postgresql/V2__complete_schema.sql):

### 10.1. Giá Trị Cột Price (Giá thực tế trước discount)
*   **Quy tắc:** Để tránh việc phân chia phần trăm giảm giá của Voucher/Promotion cho từng ghế đơn lẻ cực kỳ phức tạp và dễ gây sai số làm tròn số thập phân, giá trị cột `price` trong bảng `order_item_seats` được chốt:
    $$\text{order\_item\_seats.price} = \text{Giá bán gốc của loại vé (TicketType.price) trước khi giảm giá}$$
*   Toàn bộ phần tiền khấu trừ của voucher/promotion sẽ được lưu trữ tập trung ở cấp độ Order (`orders.discount_amount`) hoặc Order Item (`order_items.discount_amount`).

### 10.2. Thực thể Java Mapping (`OrderItemSeat.java`)
```java
@Entity
@Table(name = "order_item_seats", schema = "booking_schema")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class OrderItemSeat {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(name = "order_item_id", nullable = false)
    private UUID orderItemId;

    @Column(name = "seat_id", nullable = false) // Khớp cột seat_id trong V2
    private UUID seatId;

    @Column(name = "seat_label", length = 20)
    private String seatLabel;

    @Column(name = "row_name", length = 10)
    private String rowName;

    @Column(name = "seat_number", length = 10)
    private String seatNumber;

    @Column(name = "price", nullable = false, precision = 15, scale = 2) // Khớp cột price trong V2
    private BigDecimal price;

    @Column(name = "status", length = 50)
    private String status = "RESERVED"; // RESERVED, CONFIRMED, CANCELLED

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt = Instant.now();
}
```

---

## 11. Tiêu Chí Nghiệm Thu Kiến Trúc (Acceptance Criteria)

Hệ thống phải vượt qua 4 bộ Acceptance Criteria dưới đây:

### AC-1: Đặt vé General Admission (Triển lãm/Sự kiện không sơ đồ)
*   **Given:** Một Event có TicketType `inventoryMode = QUANTITY` và `accessScope = EVENT`.
*   **When:** Gửi request đặt vé không truyền `seatIds`.
*   **Then:** Đơn hàng tạo thành công `PENDING`, counter giảm chính xác. Không có bản ghi nào được sinh trong `event_seat_inventory` và `order_item_seats`.

### AC-2: Đặt vé Standing Zone (Khu Fanzone không số ghế)
*   **Given:** Một Event có Sector `STANDING` có total_capacity = 500, TicketType tương ứng có `inventoryMode = QUANTITY` và `accessScope = SECTOR`.
*   **When:** Request đặt 2 vé.
*   **Then:** Đơn hàng tạo thành công. Counter giảm chính xác. Sức chứa của sector được bảo toàn. Không sinh ghế vật lý trong database.

### AC-3: Đặt vé Seated (Chọn số ghế chính xác)
*   **Given:** Một Event có Sector `SEATED`, TicketType tương ứng có `inventoryMode = ASSIGNED_SEAT` và `accessScope = SECTOR`. Ghế `A-01` đang `AVAILABLE`.
*   **When:** Request đặt ghế `A-01`.
*   **Then:** Ghế `A-01` chuyển sang `RESERVED` trong DB và Redis cache. Một bản ghi được insert vào `order_item_seats` với `seat_id` khớp ID ghế, `price` khớp giá vé thực tế, và `status = 'RESERVED'`. Khi hết 15 phút không thanh toán, ghế tự động trả về `AVAILABLE`.

### AC-4: Đặt vé Mixed (Hỗn hợp cả Standing và Seated)
*   **Given:** Buyer đặt 1 vé Standing Fanzone (Quantity = 1) và 1 vé Seated (Quantity = 1, Seat = `B-05`) trong cùng một đơn hàng.
*   **When:** Trực tiếp checkout.
*   **Then:** Hệ thống khóa phân tán thành công cho ghế `B-05`, đồng thời trừ counter cho cả hai loại vé. Sinh 1 bản ghi ghế `B-05` trong `order_item_seats`, vé Fanzone không sinh ghế.
