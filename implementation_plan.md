# Phase 2B — Payment & Ticket Issuance (Revised After pg_epm Research)

## Giới thiệu

Sau khi phân tích kỹ `pg_epm` (production payment gateway của công ty), phần này tổng hợp:
- Những gì pg_epm làm đúng và cần học theo
- Những điểm pg_epm chưa tối ưu và sẽ cải thiện
- Những gì pg_epm có nhưng **không cần** áp dụng (khác business scope)

---

## Kiến trúc tổng thể pg_epm (đã phân tích)

```
Request → [RequestChecksumConverter] decrypt toàn bộ body
        → [PermissionRequestInterceptor] kiểm tra quyền
        → Controller → Service → Repository
        → [ResponseChecksumConverter] encrypt toàn bộ response

IPN từ VNPay → Controller → VNPayService
  Step 1: Redis SetNx (idempotency lock)
  Step 2: Load transaction từ DB
  Step 3: Verify status = PENDING  ← sớm reject duplicate
  Step 4: Verify HMAC signature
  Step 5: Verify amount (VNPay gửi * 100)
  Step 6: Update transaction + order
  Step 7: RabbitMQ async → issue hàng hóa
  Step 8: Redis release lock
```

---

## Báo cáo chi tiết: pg_epm vs Plan của bạn

### ✅ Áp dụng vào Phase 2B

#### 1. Redis SetNx cho IPN Idempotency (CRITICAL)

pg_epm dùng Redis SetNx thay vì DB unique check vì:
- DB `existsByProviderTransactionId` có race condition khi 2 IPN đến cùng lúc (1ms window)
- Redis SetNx là **atomic** — đảm bảo chỉ 1 thread xử lý

```java
// pg_epm pattern (dòng 190-197 VNPayServiceImpl)
String redisKey = "vnpay:ipn:" + vnp_TxnRef;
long isLock = redisService.setNx(redisKey, redisKey, 30); // 30s lease
if (isLock != 1) {
    // Đang được xử lý ở thread khác — VNPay retry
    return new VnPayCallBackResponse("00", "Processing");
}
try {
    // ... xử lý IPN
} finally {
    redisService.release(redisKey); // luôn release
}
```

**Cải thiện so với pg_epm:** pg_epm dùng `setNx` thủ công. Ta dùng `Redisson RLock` (đã có sẵn) để đảm bảo cleanup tốt hơn.

#### 2. Spam Block khi tạo payment URL

pg_epm chặn user tạo transaction liên tục trong N giây (`CREATE_TRANSACTION_SPAM_BLOCK_TIME = 5s`):

```java
// pattern từ dòng 138-142 CreateTransactionServiceImpl
String spamKey = "payment:spam:" + userId + ":" + orderId;
if (redisService.setNx(spamKey, "1", 5) == 0L) {
    throw new TooManyRequestsException("Vui lòng thử lại sau");
}
// finally: redisService.del(spamKey) — xóa ngay sau khi xử lý
```

#### 3. Amount Verification bắt buộc trong IPN (CRITICAL)

pg_epm verify amount từ VNPay vs amount trong DB:

```java
// dòng 228-235 VNPayServiceImpl
BigDecimal requestAmount = new BigDecimal(request.getVnp_Amount());
BigDecimal entityAmount = entity.getPayAmount().movePointRight(2); // VNPay gửi × 100
if (requestAmount.compareTo(entityAmount) != 0) {
    response.setResponseCode("04"); // Invalid amount
    return response;
}
```

**Quan trọng:** VNPay gửi amount × 100 (ví dụ: 150,000 VND → "15000000"). Phải `movePointRight(2)` trước khi so sánh.

#### 4. Thứ tự xử lý IPN đúng của pg_epm

pg_epm kiểm tra `status != PENDING` **TRƯỚC** khi verify signature:

```java
// dòng 206-210 VNPayServiceImpl — reject sớm nhất có thể
if (!EpmTransStatus.PEND.toString().equals(entity.getPayStatus())) {
    response.setResponseCode("02"); // Already confirmed
    return response; // Không verify signature nữa — tốn CPU không cần thiết
}
// ... rồi mới verify HMAC
```

**Cải thiện:** Logic này đúng nhưng pg_epm verify signature SAU status check — ta sẽ giữ vậy vì nếu transaction đã processed rồi thì không cần tốn CPU verify.

#### 5. QueryDR pattern — Caching transaction để retry

pg_epm lưu transaction vào Redis sau khi tạo:

```java
// dòng 589-593 CreateTransactionServiceImpl
String value = jsonUtils.parseObjectToJson(queryDRAutoDTO);
redisService.set("epm:querydr:" + transactionId, value, 2400); // 40 phút
redisService.zAdd("epm:querydr:list", System.currentTimeMillis(), transactionId);
```

Dùng cho "QueryDR" (Query Debit Result) — cho phép system tự động query lại trạng thái transaction từ VNPay nếu IPN không đến sau N phút.

**Áp dụng phiên bản đơn giản hóa:** Sau khi tạo payment URL, lưu `orderId → transactionNumber` mapping vào Redis (TTL = 20 phút) để:
- Frontend polling `/status` tìm nhanh
- Backup nếu IPN delay

#### 6. TransactionValidator — Validate min/max amount, daily limit

pg_epm có `TransactionValidatorService` với logic validate:
- Số giao dịch tối đa/ngày
- Tổng amount tối đa/ngày
- Min amount / Max amount per transaction

**Phiên bản cho ticket system:**
```java
// PaymentValidatorService (mới)
void validatePaymentRequest(Order order) {
    // 1. Order còn hạn chưa? (expiresAt)
    if (order.getExpiresAt().isBefore(Instant.now())) {
        throw new InvalidRequestException("Đơn hàng đã hết hạn thanh toán");
    }
    // 2. Order đúng trạng thái PENDING?
    if (order.getStatus() != PENDING) {
        throw new InvalidRequestException("Đơn hàng không ở trạng thái chờ thanh toán");
    }
    // 3. Amount > 0?
    if (order.getTotalAmount().compareTo(BigDecimal.ZERO) <= 0) {
        throw new InvalidRequestException("Số tiền thanh toán không hợp lệ");
    }
}
```

#### 7. Strategy Pattern cho Payment Gateway (BETTER THAN pg_epm)

pg_epm dùng `if-else` để route giữa các gateway. Ta dùng **Strategy Pattern** với tính mở rộng tốt hơn:

```
PaymentGateway (interface)
├── createPaymentUrl(order, transaction, ip) → String
└── verifyCallback(params) → PaymentResult

VNPayGateway implements PaymentGateway   ← Phase 2B
MomoGateway implements PaymentGateway    ← Phase tương lai

PaymentGatewayFactory
└── getGateway("VNPAY") → VNPayGateway
```

---

### ❌ KHÔNG áp dụng — Khác business scope

#### RabbitMQ Async Issue
pg_epm cần async vì issue vé viễn thông gọi external SOAP/REST API mất 3-10s. Ticket hệ thống là DB write <100ms → **sync là đủ, đơn giản hơn**.

#### Merchant Routing (`TransactionPolicyService`)
pg_epm phục vụ nhiều merchant (partner) với routing động. Flash ticket chỉ có 1 merchant (admin của app) → **hardcode TMN_CODE là đủ**.

#### Request/Response Body Encryption (`RequestChecksumConverter`)
pg_epm encrypt toàn bộ body vì API được gọi bởi các merchant bên ngoài qua internet công khai. Flash ticket API đã được bảo vệ bởi **HTTPS + JWT** từ Keycloak → **double encryption là over-engineering**.

---

### ⚠️ Điểm chưa tối ưu trong pg_epm (SẼ CẢI THIỆN)

| Vấn đề | pg_epm | Cải thiện trong Phase 2B |
|--------|--------|--------------------------|
| Không dùng `@Transactional` trong IPN | Có thể partial update nếu crash | Bọc toàn bộ IPN handler trong `@Transactional` |
| `buildHashData` không sort đầy đủ | Filter `null` ok, nhưng hardcode fields | Sort tất cả params động theo key |
| `VnPayCallBackResponse` response code magic string | `"00"`, `"97"`, `"04"` hardcode | Dùng enum `VNPayResponseCode` |
| Class `CreateTransactionServiceImpl` quá dài (613 dòng) | 2 method private khổng lồ | Tách thành `PaymentValidatorService` riêng |

---

## Card Tokenization — Tại sao không cần?

**Tokenization là gì?**

Khi user thanh toán lần đầu bằng thẻ ngân hàng, thay vì yêu cầu nhập số thẻ mỗi lần, hệ thống:
1. Gửi số thẻ lên VNPay
2. VNPay trả về `tokenId` (chuỗi ngẫu nhiên đại diện cho thẻ đó)
3. Lần sau user có thể thanh toán 1-click bằng `tokenId` (không cần nhập thẻ lại)

```
pg_epm có: BankTokenEntity { tokenId, subId, cardType }
Consumer:  CreateTokenConsumer, DeleteTokenConsumer (RabbitMQ)
```

**Tại sao Flash Ticket CHƯA CẦN?**

1. **Tần suất mua vé thấp:** User mua vé concert tối đa vài lần/năm → trải nghiệm 1-click không quan trọng bằng Billtop (nạp tiền điện thoại hàng tuần).
2. **Security risk:** Lưu token đòi hỏi thêm security audit, PCI-DSS compliance considerations.
3. **Complexity:** Cần thêm `BankToken` entity, 2 consumers, token lifecycle management.
4. **VNPay Sandbox:** Sandbox của VNPay không hỗ trợ tokenization đầy đủ.

**Khi nào CẦN thêm:** Nếu app có tính năng "Auto-renew" subscription (ví dụ: season pass) hoặc user phàn nàn về UX thì mới triển khai.

---

## Cấu trúc file Phase 2B (Final)

```
payment/
├── config/
│   └── VNPayProperties.java           [NEW] @ConfigurationProperties
├── controller/
│   └── PaymentController.java         [NEW] 4 endpoints
├── dto/
│   ├── PaymentInitRequest.java        [NEW]
│   ├── PaymentInitResponse.java       [NEW]
│   └── PaymentStatusResponse.java     [NEW]
├── gateway/
│   ├── PaymentGateway.java            [NEW] interface (Strategy)
│   ├── PaymentGatewayFactory.java     [NEW] factory
│   └── VNPayGateway.java              [NEW] VNPay implementation
├── service/
│   ├── PaymentService.java            [NEW] orchestration
│   └── PaymentValidatorService.java   [NEW] validation logic
├── entity/
│   └── Transaction.java               [EXISTS]
└── repository/
    └── TransactionRepository.java     [EXISTS]
```

---

## Flow IPN xử lý (production-grade)

```java
@GetMapping("/api/payments/vnpay-ipn") // GET vì VNPay dùng GET
@Transactional
public VNPayIPNResponse handleIPN(@RequestParam Map<String, String> params) {

    String txnRef = params.get("vnp_TxnRef");
    String redisKey = "vnpay:ipn:" + txnRef;

    // Step 1: Redis lock (atomic, tránh race condition)
    RLock lock = redissonClient.getLock(redisKey);
    if (!lock.tryLock(0, 10, TimeUnit.SECONDS)) {
        return VNPayIPNResponse.ok(); // "00" — đang xử lý
    }

    try {
        // Step 2: Load transaction
        Transaction tx = transactionRepository.findByTransactionNumber(txnRef)
            .orElseThrow(...);

        // Step 3: Sớm reject nếu đã xử lý (tránh verify HMAC tốn CPU)
        if (tx.getStatus() != PENDING) {
            return VNPayIPNResponse.alreadyConfirmed(); // "02"
        }

        // Step 4: Verify HMAC
        if (!vnPayGateway.verifySignature(params)) {
            return VNPayIPNResponse.invalidSignature(); // "97"
        }

        // Step 5: Verify amount (VNPay gửi × 100)
        BigDecimal vnpAmount = new BigDecimal(params.get("vnp_Amount")).movePointLeft(2);
        if (vnpAmount.compareTo(tx.getAmount()) != 0) {
            return VNPayIPNResponse.invalidAmount(); // "04"
        }

        // Step 6: Xử lý kết quả
        if ("00".equals(params.get("vnp_ResponseCode"))) {
            tx.setStatus(SUCCESS);
            order.setStatus(CONFIRMED);
            order.setPaymentMethod("VNPAY");
            promotionService.confirmPromotion(order.getPromotionId());
            ticketIssuanceService.issueTickets(order.getId());
        } else {
            tx.setStatus(FAILED);
        }

        return VNPayIPNResponse.ok(); // "00"
    } finally {
        lock.unlock();
    }
}
```

---

## Verification Plan

### Setup
1. Đăng ký VNPay Sandbox tại [sandbox.vnpayment.vn](https://sandbox.vnpayment.vn)
2. `ngrok http 8080` để expose localhost cho IPN
3. Thẻ test: `9704198526191432198` / `07/15` / OTP: `123456`

### Test sequence
```
1. POST /api/bookings           → orderId, PENDING
2. POST /api/payments/initiate  → paymentUrl
3. Browser mở paymentUrl        → thanh toán sandbox
4. VNPay gọi IPN → log xác nhận
5. GET /api/orders/{id}         → CONFIRMED
6. GET /api/tickets/my-tickets  → tickets với QR
7. POST /api/tickets/checkin    → check-in bằng QR
```
