# Flash Ticket System — Concurrency Deep-Dive Audit

> Source-verified against 20+ Java files. Does not re-report Bug 1–8 from `AI_CONTEXT.md`.

---

## Area 1: Order Expiry Scheduler vs. IPN Race Condition

### Concurrency Model Analysis

The expiry flow is: `OrderExpirationService` (every 60s) → `findExpiredPendingOrders()` (no row lock) → for each order → `OrderExpirationHelper.expireOne()` → `@Transactional { restoreStock() + releasePromotion() + status → EXPIRED }`.

The IPN flow is: `VNPayIPNService.processIPN()` → `@Transactional { Transaction → SUCCESS + confirmOrder() → status → CONFIRMED + publishEvent(PaymentSuccessEvent) }`.

**Critical timing window:** Both flows operate on the same Order row with **no cross-process coordination**. The expiry scheduler reads the order with a plain `SELECT` (no `FOR UPDATE`), then later updates it in `expireOne()`. The IPN handler reads the order with `findById()` (also no `FOR UPDATE`), then updates status.

### Bug Identified: **Yes — Race Condition**

### Scenario That Triggers It

```
T=0ms    OrderExpirationService: SELECT orders WHERE status=PENDING AND expires_at < now
         → Returns Order #123 (expired 30s ago)

T=5ms    VNPay IPN arrives for Order #123
         → VNPayIPNService.processIPN() starts @Transactional
         → Redis idempotency lock acquired
         → Transaction → SUCCESS
         → confirmOrder(): findById(#123) → status=PENDING → set CONFIRMED, paidAt=now
         → COMMIT ✅

T=50ms   OrderExpirationHelper.expireOne(Order #123) starts @Transactional
         → order object was loaded at T=0ms, still has status=PENDING in memory
         → restoreStock(): quantityAvailable += N, quantityReserved -= N  ← WRONG! Stock double-counted
         → releasePromotion(): currentUses -= 1                           ← WRONG! Slot leaked
         → order.setStatus(EXPIRED)
         → orderRepository.save(order)  ← Hibernate merges stale entity
         → COMMIT — overwrites CONFIRMED with EXPIRED ❌
```

**Result:** User paid successfully, but order is EXPIRED. Stock was restored (making those tickets available for sale to someone else), yet tickets may have already been issued. **Double-selling** + **money taken with no ticket** simultaneously.

### Impact: 🔴 CRITICAL — Money at risk + data corruption (double-sold inventory)

### Recommended Fix

**Option A (Lowest risk, immediate):** Use optimistic locking or conditional UPDATE in `expireOne()`:

```java
// In OrderExpirationHelper.expireOne()
@Transactional
public void expireOne(Order order) {
    // Re-read with status check to prevent overwriting concurrent IPN confirm
    int updated = orderRepository.updateStatusToExpired(order.getId());
    // Returns 0 if order is no longer PENDING (IPN confirmed it)
    if (updated == 0) {
        log.info("Order {} already processed (likely IPN confirmed), skipping expiry",
                 order.getOrderNumber());
        return; // Don't restore stock — IPN already handled it
    }
    
    bookingService.restoreStock(order.getId());
    promotionService.releasePromotion(order.getPromotionId());
}
```

Add to `OrderRepository.java`:
```java
@Modifying
@Query("""
    UPDATE Order o SET o.status = 'EXPIRED'
    WHERE o.id = :id AND o.status = 'PENDING'
""")
int updateStatusToExpired(@Param("id") UUID id);
```

**Option B (Stronger, also recommended):** Add `@Version` (JPA optimistic locking) to `Order.java`:
```java
@Version
private Long version;
```
This causes `ObjectOptimisticLockingFailureException` if two transactions attempt to update the same order, which can be caught and retried.

---

## Area 2: Promotion Code Race Condition

### Concurrency Model Analysis

`PromotionService.reservePromotion()` has two limits to enforce:

1. **Global limit** (`max_total_uses`): Protected by `atomicIncrementUsage()` — `UPDATE ... SET current_uses = current_uses + 1 WHERE current_uses < max_total_uses`. This is **correctly atomic** at the SQL level. ✅

2. **Per-user limit** (`max_uses_per_user`): Checked at [PromotionService L82](file:///d:/Project/flash-ticket-system/core-service/src/main/java/com/flashticket/core/promotion/service/PromotionService.java#L82) via `promotionUsageRepository.countByUserIdAndPromotionId()`. This is a **read-then-check** (TOCTOU) pattern.

However, the per-user count queries `PromotionUsage` records, which are only created by `confirmPromotion()` — called **after payment success**. During the booking phase, only `reservePromotion()` runs (incrementing `current_uses`). So the per-user check counts **confirmed usages only**, not pending reservations.

### Bug Identified: **Potential — Per-User Limit Bypass**

### Scenario That Triggers It

```
State: Voucher "FLASH50" — max_uses_per_user = 1. User A has 0 confirmed usages.

T=0ms    User A, Tab 1: createBooking(promoCode="FLASH50")
         → reservePromotion() → countByUserIdAndPromotionId = 0 → passes check
         → atomicIncrementUsage() → current_uses = 1
         → Order #1 created (PENDING)

T=100ms  User A, Tab 2: createBooking(promoCode="FLASH50", different event)
         → BUT: blocked by Guard 0 (spam lock) per EVENT ← key = "booking:spam:{userId}:{eventId}"
         → Different eventId → spam lock does NOT block
         → BUT: blocked by Guard 1 (duplicate PENDING check) per EVENT
         → Different eventId → Guard 1 does NOT block
         → reservePromotion() → countByUserIdAndPromotionId = 0
           (still 0 because Order #1 hasn't been paid yet → no PromotionUsage record)
         → atomicIncrementUsage() → current_uses = 2
         → Order #2 created (PENDING)

T=later  Both orders paid → both get discount → user bypassed per-user limit
```

**However:** This is **partially mitigated** by the fact that `confirmPromotion()` is never actually called (as identified in Flow A analysis). So `PromotionUsage` records are never created, meaning this per-user check is **always 0** — effectively broken for a different reason.

### Impact: 🟡 MEDIUM — Per-user promotion limit can be bypassed when user books for different events concurrently. The blast radius is limited by the global `max_total_uses` atomic check, which works correctly. *(Becomes 🟠 HIGH once `confirmPromotion()` bug is fixed — because then the TOCTOU window still exists.)*

### Recommended Fix

Add Redis guard **before** the DB per-user check:

```java
// In reservePromotion(), before the per-user DB check:
String userPromoKey = "promo:user:" + userId + ":" + promo.getId();
RBucket<String> bucket = redissonClient.getBucket(userPromoKey);
boolean isFirst = bucket.setIfAbsent("1", Duration.ofMinutes(20)); // > order TTL
if (!isFirst && promo.getMaxUsesPerUser() != null && promo.getMaxUsesPerUser() <= 1) {
    throw new InvalidRequestException("Bạn đã sử dụng mã voucher này");
}
```

And release the key in `releasePromotion()` / after expiry. This creates an atomic guard similar to the booking spam lock — existing pattern in the codebase.

---

## Area 3: Ticket Transfer Logic

### Concurrency Model Analysis

`Ticket.java` entity has three transfer-related fields:
- `isTransferable` (Boolean, default `true`) — set in [TicketIssuanceService L117](file:///d:/Project/flash-ticket-system/core-service/src/main/java/com/flashticket/core/booking/service/TicketIssuanceService.java#L117)
- `transferredFromTicketId` (UUID, nullable) — never written anywhere
- `transferredAt` (Instant, nullable) — never written anywhere

Grep for `transfer` across all service files: **zero results** outside of entity field declarations and the `isTransferable(true)` setter in ticket issuance.

Also noted: `TicketStatus` enum includes `TRANSFERRED` value — never used.

### Bug Identified: **No — But Data Integrity Risk Exists**

There is no transfer logic at all, so there are no concurrency bugs. However:

### Scenario That Triggers Data Integrity Risk

1. **Field defaulting to `true`:** Every ticket is created with `isTransferable = true`. If a future developer or API consumer reads this field, they would believe transfers are supported. A naive frontend could show a "Transfer" button with no backend to handle it.

2. **Schema coupling:** The `transferredFromTicketId` field is a nullable UUID column in the `tickets` table. It has no FK constraint (by design — cross-module), but also no index. A future transfer implementation could accidentally create circular references (ticket A transferred from B, B transferred from A) without a guard.

3. **Status transition gap:** `Ticket.TicketStatus.TRANSFERRED` exists but check-in validation at [TicketIssuanceService L187](file:///d:/Project/flash-ticket-system/core-service/src/main/java/com/flashticket/core/booking/service/TicketIssuanceService.java#L187) checks `status != VALID` → throws generic error. A TRANSFERRED ticket would get the message _"Vé không hợp lệ (trạng thái: TRANSFERRED)"_ but no explanation about who holds the new ticket.

### Impact: 🟢 LOW — No current runtime bug. Future implementation risk only.

### Recommended Fix

Short-term: Set `isTransferable = false` as default until transfer feature is implemented, to prevent misleading API consumers:

```java
// In TicketIssuanceService.issueTickets()
.isTransferable(false)  // Changed from true — transfer feature not yet implemented
```

Long-term: When implementing transfer, use `PESSIMISTIC_WRITE` lock on the ticket row (same pattern as check-in) plus a `UNIQUE` constraint on `(transferredFromTicketId)` to prevent double-transfer.

---

## Area 4: Payment Status Polling

### Concurrency Model Analysis

`PaymentService.getPaymentStatus()` at [PaymentService L181](file:///d:/Project/flash-ticket-system/core-service/src/main/java/com/flashticket/core/payment/service/PaymentService.java#L181) is a `@Transactional(readOnly = true)` method with IDOR protection. It returns `order.status` + list of transactions.

The frontend calls `GET /api/payments/status/{orderId}` after redirect from VNPay. It polls until `orderStatus != PENDING`.

### Bug Identified: **Potential — Frontend Stuck Indefinitely**

### Scenario That Triggers It

```
1. User pays on VNPay → VNPay redirects user back to frontend
2. Frontend starts polling GET /api/payments/status/{orderId} every 2-3s
3. VNPay IPN is delayed (network issue) or never arrives
4. Polling keeps returning { status: "PENDING" } forever
5. User stares at spinner indefinitely — no timeout or fallback messaging
```

**What the backend provides:** `PaymentStatusResponse` contains `orderStatus` and list of `TransactionSummary`. When the IPN arrives and order becomes CONFIRMED, the next poll returns CONFIRMED. When the order expires, the next poll returns EXPIRED (after scheduler runs).

**What's missing:** There is no server-side mechanism to distinguish between "waiting for IPN" and "IPN will never come". The `expiresAt` field is available in the Order but is **not included** in `PaymentStatusResponse` — the frontend has no way to know when to stop polling and show a timeout message.

### Impact: 🟠 HIGH — User journey blocked with no workaround when IPN is delayed

### Recommended Fix

1. **Include `expiresAt` in `PaymentStatusResponse`:**

```java
public record PaymentStatusResponse(
    UUID orderId,
    String orderStatus,
    BigDecimal totalAmount,
    Instant expiresAt,        // ← ADD THIS
    List<TransactionSummary> transactions
) { ... }
```

Frontend can then show: *"Đang chờ xác nhận thanh toán. Tự động hết hạn lúc {expiresAt}."* and stop polling after expiry.

2. **Add transaction-level status to help frontend distinguish states:**
   - If latest Transaction = FAILED → show "Thanh toán thất bại" + retry button
   - If latest Transaction = PENDING + `completedAt = null` → show "Đang chờ xác nhận"
   - If no Transaction exists → show "Chưa bắt đầu thanh toán"

3. **(Phase 2C, per AI_CONTEXT roadmap):** Replace polling with WebSocket push via `IPN success → Kafka/RabbitMQ → WebSocket push → frontend`.

---

## Area 5: Multi-Instance Deployment

### Concurrency Model Analysis

If two `core-service` pods run simultaneously, three subsystems need analysis:

**5A — Redis Locks:**

All Redis keys are globally keyed:
- Booking spam: `booking:spam:{userId}:{eventId}` ← per user+event, globally unique ✅
- Payment spam: `payment:spam:{userId}:{orderId}` ← per user+order, globally unique ✅
- IPN idempotency: `vnpay:ipn:{txnRef}` ← per transaction number, globally unique ✅
- Zone lock: `lock:tickettype:{ticketTypeId}` ← per ticket type, globally unique ✅

Redisson `RLock` internally uses `{clientId}:{threadId}` as the lock value, so cross-pod locks never conflict on ownership. **No bug here.** ✅

### Bug Identified (5A): **No**

---

**5B — RabbitMQ Consumers:**

Both pods register `@RabbitListener` on the same queues (`q.ticket.issue`, `q.email.send`). RabbitMQ with default round-robin dispatch + `prefetchCount=1` creates **competing consumers** — each message goes to exactly one pod. This is correct.

The `AcknowledgeMode.MANUAL` ensures messages are not lost.

### Bug Identified (5B): **No — Correctly Competing Consumers** ✅

However, one edge case worth noting:

If Pod A crashes mid-processing (after `issueTickets()` but before `channel.basicAck()`), RabbitMQ will re-deliver to Pod B. `TicketIssuanceService.issueTickets()` has idempotency check (`countByOrderIdAndIsDeletedFalse > 0`), so Pod B will return existing tickets. But `QRCodeService.generateAndUploadForTickets()` also has idempotency (`if qrCodeImageUrl != null → skip`). **This is correctly handled.** ✅

---

**5C — Order Expiry Scheduler:**

### Bug Identified (5C): **Yes — Duplicate Processing**

`@Scheduled(fixedDelay = 60_000)` runs on **every pod independently**. There is no leader election (`ShedLock`, `@SchedulerLock`, etc.) and no `SELECT FOR UPDATE SKIP LOCKED` in `findExpiredPendingOrders()`.

### Scenario That Triggers It

```
T=0s     Pod A: SELECT expired orders → [Order #1, #2, #3]
T=0.1s   Pod B: SELECT expired orders → [Order #1, #2, #3]  (same result set)

T=0.5s   Pod A: expireOne(Order #1) → restoreStock(+5) → EXPIRED ✅
T=0.6s   Pod B: expireOne(Order #1) → restoreStock(+5) → save(EXPIRED)

Result: quantityAvailable incremented TWICE (+10 instead of +5) for Order #1
        Stock inflated → more tickets can be sold than physically exist = OVERSELL
```

The `restoreQuantity()` SQL is `SET quantityAvailable = quantityAvailable + :amount` — it does **not** have a guard like `WHERE status = 'PENDING'`. It blindly adds, so a double-call doubles the restoration.

### Impact: 🔴 CRITICAL — Stock inflation leading to overselling

### Recommended Fix

**Option A (Simplest, immediate):** Add [ShedLock](https://github.com/lukas-krecan/ShedLock) to prevent concurrent scheduler execution:

```xml
<!-- pom.xml -->
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
    <version>5.x</version>
</dependency>
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-provider-redis-spring</artifactId>
    <version>5.x</version>
</dependency>
```

```java
@Scheduled(fixedDelay = 60_000)
@SchedulerLock(name = "orderExpiration", lockAtMostFor = "5m", lockAtLeastFor = "30s")
public void expireOrders() { ... }
```

**Option B (DB-level, complements Option A):** Use `SELECT FOR UPDATE SKIP LOCKED` to prevent two pods processing the same row:

```java
@Query(value = """
    SELECT * FROM booking_schema.orders
    WHERE status = 'PENDING' AND expires_at < :now AND is_deleted = false
    ORDER BY expires_at ASC
    LIMIT :limit
    FOR UPDATE SKIP LOCKED
    """, nativeQuery = true)
List<Order> findExpiredPendingOrdersForUpdate(@Param("now") Instant now, @Param("limit") int limit);
```

**Recommended: Use both** — ShedLock as primary guard, `SKIP LOCKED` as defense-in-depth.

---

## Summary Matrix

| Area | Bug? | Severity | Root Cause | Fix Effort |
|------|------|----------|------------|------------|
| 1 — Expiry vs IPN Race | ✅ Yes | 🔴 CRITICAL | No conditional UPDATE or optimistic lock on Order status transition | Low (add `WHERE status = PENDING` to expiry UPDATE) |
| 2 — Promotion Per-User | ⚠️ Potential | 🟡 MEDIUM | Read-then-check TOCTOU on `PromotionUsage` count (different-event scenario) | Low (Redis `SetNx` guard) |
| 3 — Ticket Transfer | ❌ No | 🟢 LOW | Fields exist without logic — misleading defaults | Trivial (change default) |
| 4 — Payment Polling | ⚠️ Potential | 🟠 HIGH | `expiresAt` not exposed to frontend; no IPN-never-arrives fallback | Low (add field to DTO) |
| 5A — Redis Locks | ❌ No | ✅ OK | Globally-keyed, Redisson handles cross-pod correctly | — |
| 5B — RabbitMQ Consumers | ❌ No | ✅ OK | Competing consumers with manual ACK + idempotency checks | — |
| 5C — Scheduler Dual-Fire | ✅ Yes | 🔴 CRITICAL | No ShedLock/leader election; no `SKIP LOCKED`; `restoreQuantity` unconditionally adds | Medium (add ShedLock + SKIP LOCKED) |
