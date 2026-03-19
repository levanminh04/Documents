# Flash Ticket System — Flow Trace & Gap Analysis

> Based on exhaustive source code review of 16+ Java files across `booking`, `payment`, `notification`, and `shared` modules.
> 
> **Constraint:** Does not re-report Bug 1–8 from `AI_CONTEXT.md`.

---

## Flow A: Happy Path

**Browse event → Select ticket type → Create order → Pay (VNPay) → IPN callback → Ticket issued → Email + QR received → Check-in**

### Steps

| # | Step | Class / Method | State Transition |
|---|------|----------------|------------------|
| 1 | Buyer browses events | `EventController.getEvents()` | — |
| 2 | Buyer views event detail + ticket types | `EventController.getEventByIdOrSlug()` | — |
| 3 | Buyer submits booking | `BookingController → BookingService.createBooking()` | — |
| 3a | Redis spam lock (`SetNx`) | `BookingService` L97-100 | — |
| 3b | Duplicate PENDING order check | `BookingService` L133-138 | — |
| 3c | Event + TicketType validation | `BookingService.validateEvent()` + `buildAndValidateContexts()` | — |
| 3d | Redis RLock → double-check stock → atomic decrement | `TicketReservationService.acquireZoneLock()` → `TicketTypeRepository.decrementAvailableAndIncrementReserved()` | `quantityAvailable -= N`, `quantityReserved += N` |
| 3e | Promotion reservation (if voucher) | `PromotionService.reservePromotion()` | `current_uses += 1` |
| 3f | Create Order + OrderItems | `orderRepository.save()` | Order: **→ PENDING**, `expiresAt = now + 15min` |
| 4 | Buyer initiates payment | `PaymentController → PaymentService.initiatePayment()` | — |
| 4a | Redis spam lock (5s TTL) | `PaymentService` L93-95 | — |
| 4b | Validate order (IDOR, status, expiry, amount) | `PaymentValidatorService.validateForPayment()` | — |
| 4c | Create Transaction record | `transactionRepository.save()` | Transaction: **→ PENDING** |
| 4d | Build VNPay URL (HMAC-SHA512 signed) | `VNPayGateway.createPaymentUrl()` | — |
| 4e | Return `paymentUrl` to frontend → redirect | — | — |
| 5 | User completes payment on VNPay | (external) | — |
| 6 | VNPay calls IPN callback | `VNPayIPNController.handleVNPayIPN()` → `VNPayIPNService.processIPN()` | — |
| 6a | Redis idempotency lock (30s TTL) | `VNPayIPNService` L93-95 | — |
| 6b | Find Transaction by `vnp_TxnRef` | `transactionRepository.findByTransactionNumber()` | — |
| 6c | Verify status == PENDING | `VNPayIPNService` L121 | — |
| 6d | Verify HMAC-SHA512 signature | `VNPayGateway.verifyCallback()` | — |
| 6e | Verify amount matches | `PaymentValidatorService.validateAmount()` | — |
| 6f | Update Transaction | `updateTransaction()` | Transaction: **PENDING → SUCCESS** |
| 6g | Confirm Order | `confirmOrder()` | Order: **PENDING → CONFIRMED**, `paidAt = now` |
| 6h | Publish Spring event | `eventPublisher.publishEvent(PaymentSuccessEvent)` | — |
| 6i | Return `{RspCode: "00"}` to VNPay | — | — |
| 7 | After DB commit: async handler fires | `@Async @TransactionalEventListener(AFTER_COMMIT)` `handlePaymentSuccessEvent()` | — |
| 7a | Publish to RabbitMQ `ex.payment` | `rabbitTemplate.convertAndSend()` | Message → `q.ticket.issue` |
| 8 | TicketWorker consumes message | `TicketMessageListener.handlePaymentSuccess()` | — |
| 8a | Issue tickets in DB | `TicketIssuanceService.issueTickets()` | Creates N Ticket records (status: **VALID**), `ticketsSold += N`, `quantityReserved -= N` |
| 8b | Generate QR images + upload Cloudinary | `QRCodeService.generateAndUploadForTickets()` | `qrCodeImageUrl` set per ticket |
| 8c | Publish `TicketIssuedEvent` → `ex.ticket` | `rabbitTemplate.convertAndSend()` | Message → `q.email.send` |
| 8d | Manual ACK | `channel.basicAck()` | Message removed from queue |
| 9 | EmailWorker consumes message | `EmailMessageListener.handleTicketIssued()` | — |
| 9a | Render Thymeleaf HTML + send email | `EmailService.sendTicketConfirmationEmail()` | — |
| 9b | Manual ACK | `channel.basicAck()` | Message removed from queue |
| 10 | Buyer scans QR at gate | `TicketController → TicketIssuanceService.validateAndCheckIn()` | — |
| 10a | Parse QR + verify HMAC-SHA256 | L155-176 | — |
| 10b | Load ticket with `PESSIMISTIC_WRITE` lock | `ticketRepository.findByTicketCodeForUpdate()` | — |
| 10c | Check status == VALID | L183-189 | — |
| 10d | Mark as USED | L192-197 | Ticket: **VALID → USED** |

### Broken/Missing Steps

- 🟡 MEDIUM — **No `confirmPromotion()` call after payment success.** The comment at [BookingService L236](file:///d:/Project/flash-ticket-system/core-service/src/main/java/com/flashticket/core/booking/service/BookingService.java#L236) states _"PaymentService.handleIPN() sẽ gọi promotionService.confirmPromotion()"_, but `VNPayIPNService.processIPN()` does **not** call `confirmPromotion()`. Promotion remains in "reserved" state forever after successful payment. If the promotion has a `max_total_uses` limit, these reserved-but-never-confirmed slots are effectively leaked.
  - **Recommended fix:** Add `promotionService.confirmPromotion(order.getPromotionId())` inside `VNPayIPNService.confirmOrder()` after setting order status to CONFIRMED.

- 🟡 MEDIUM — **No frontend notification mechanism after IPN.** Frontend relies on polling `GET /api/payments/status/{orderId}`. If user closes browser before order is confirmed, they have no way to know payment succeeded until they re-open the app and check `/my-orders`. Email is backup but may be delayed.
  - **Recommended fix:** Implement WebSocket push (Phase 2C roadmap) or at minimum, ensure the polling endpoint clearly returns CONFIRMED status so the next page visit shows correct state.

- 🟢 LOW — **`OrderExpirationService` runs every 60s with batch size 50.** If a massive flash sale creates 1000+ orders simultaneously and most go unpaid, it could take 20+ minutes to expire all of them. During this time, `quantityAvailable` for those ticket types remains artificially low.
  - **Recommended fix:** Reduce interval to 10-15s during flash sale windows, or implement Redis keyspace notification for precise TTL-based expiration.

---

## Flow B: Payment Failure Paths

### B1: Payment Timeout (user never completes VNPay form)

**Steps:**

| # | What happens |
|---|---|
| 1 | User clicks "Pay" → `PaymentService.initiatePayment()` creates Transaction (PENDING), returns VNPay URL |
| 2 | User is redirected to VNPay but never completes the form |
| 3 | VNPay **never sends IPN** (no callback at all) |
| 4 | Transaction stays **PENDING** forever in DB |
| 5 | Order stays **PENDING** until `OrderExpirationService` runs (every 60s, batch 50) |
| 6 | `OrderExpirationHelper.expireOne()`: restoreStock + releasePromotion + set EXPIRED |
| 7 | Stock is restored. User can book again. |

**Broken/Missing Steps:**

- 🟡 MEDIUM — **Transaction stays in PENDING state permanently.** `OrderExpirationHelper.expireOne()` only updates Order status → EXPIRED. It does **not** update the associated Transaction(s) to FAILED/CANCELLED. This creates orphan PENDING transactions in `payment_schema.transactions` that pollute audit data.
  - **Recommended fix:** In `OrderExpirationHelper.expireOne()`, query `transactionRepository.findByOrderIdAndStatus(orderId, PENDING)` and update them to `CANCELLED` with a message like "Order expired before payment completion".

- 🟢 LOW — **No user notification that order expired.** User only discovers expiration when they return to `/my-orders` or try to pay again. No push notification or email.
  - **Recommended fix:** Consider publishing an `OrderExpiredEvent` → email notification (nice-to-have).

---

### B2: IPN Never Arrives (VNPay network issue)

**Steps:**

| # | What happens |
|---|---|
| 1 | User completes payment on VNPay successfully |
| 2 | VNPay tries to call IPN but network fails |
| 3 | VNPay retries IPN (up to 8 retries over 25 hours per VNPay docs) |
| 4 | **If retries eventually succeed** → normal flow continues |
| 5 | **If ALL retries fail** → Transaction stays PENDING, Order stays PENDING |
| 6 | Order eventually expires via `OrderExpirationService` |
| 7 | Stock restored, promotion released — but **user already paid** |

**Broken/Missing Steps:**

- 🔴 CRITICAL — **Money taken but order expired = user paid & got nothing.** If VNPay IPN never arrives and order expires, the user has been charged but receives no tickets. There is no compensating transaction to detect this state and initiate a refund.
  - **Recommended fix:** Add a **payment reconciliation scheduler** that runs periodically (e.g., every 5 minutes). For orders transitioning to EXPIRED that have a Transaction with status PENDING, call VNPay's QueryDR API to check actual payment status. If VNPay confirms payment was successful, reverse the expiration (restore to CONFIRMED) or flag for manual admin review. ⚠️ ADR needed before implementation — this requires access to VNPay's QueryDR endpoint and a decision on auto-reversal vs. admin-approval workflow.

---

### B3: VNPay Returns Error Code (e.g., code 24 = cancelled)

**Steps:**

| # | What happens |
|---|---|
| 1 | User cancels on VNPay payment page |
| 2 | VNPay sends IPN with `vnp_ResponseCode = 24` |
| 3 | `VNPayIPNService.processIPN()` → all validations pass |
| 4 | `result.isSuccess()` = `false` → Transaction updated to **FAILED** |
| 5 | Order **stays PENDING** (no status change for failed payments) |
| 6 | User can retry payment by calling `PaymentService.initiatePayment()` again (creates new Transaction) |
| 7 | If user doesn't retry, order eventually expires via scheduler |

**Broken/Missing Steps:**

- 🟢 LOW — **This path works correctly by design.** User can retry payment multiple times, each attempt creates a new Transaction. Order remains PENDING and holds stock until either payment succeeds or order expires.

---

### B4: Order Expires While Payment is In-Flight

**Steps:**

| # | What happens |
|---|---|
| 1 | User starts payment at `T = 14:50` (order expires at `T = 15:00`) |
| 2 | User completes VNPay payment at `T = 14:58` |
| 3a | **Race A: IPN arrives at `T = 14:59`** (before expiry) — `confirmOrder()` succeeds |
| 3b | **Race B: Scheduler runs at `T = 15:01`** — `expireOne()` runs first |
| 4b | Stock restored, order set to EXPIRED |
| 5b | IPN arrives at `T = 15:02` — `confirmOrder()` checks `order.status == PENDING` → **false** (order is EXPIRED) → **skips** confirm |
| 6b | Transaction marked SUCCESS but Order is EXPIRED |

**Broken/Missing Steps:**

- 🔴 CRITICAL — **User paid successfully but order was expired by scheduler. Transaction = SUCCESS, Order = EXPIRED.** The system took money, restored stock (making it available to other buyers), but never issued tickets. There is no recovery mechanism for this inconsistency.
  - **Recommended fix:** In `VNPayIPNService.confirmOrder()`, handle the case where order status is EXPIRED:
    ```java
    if (order.getStatus() == Order.OrderStatus.EXPIRED) {
        // Payment arrived after expiry — need to re-reserve stock or flag for refund
        log.error("CRITICAL: Payment success for EXPIRED order — orderId={}", orderId);
        // Option 1: Try to re-reserve stock and confirm (complex)
        // Option 2: Flag for admin manual refund
    }
    ```
    ⚠️ ADR needed before implementation — decision between auto-restore (re-reserve stock atomically) vs. auto-refund via VNPay Refund API.

- 🟠 HIGH — **`confirmOrder()` silently logs a warning and returns when order is not PENDING.** The IPN returns `RspCode: "00"` (success) to VNPay regardless, so VNPay stops retrying. The failure is swallowed with only a `log.warn()` at [VNPayIPNService L218-219](file:///d:/Project/flash-ticket-system/core-service/src/main/java/com/flashticket/core/payment/service/VNPayIPNService.java#L218-L219).
  - **Recommended fix:** At minimum, trigger an alert/metric when `order.status != PENDING` during `confirmOrder()` so operations team is notified immediately. Consider adding a `RefundNeeded` table/queue entry.

---

## Flow C: Refund Path (Currently NOT Implemented)

### C1: User Cancels Before Completing Payment

**Steps:**

| # | What happens |
|---|---|
| 1 | User has Order (PENDING) — calls `DELETE /api/orders/{id}` |
| 2 | `BookingService.cancelOrder()` validates status == PENDING |
| 3 | `restoreStock()` → `quantityAvailable += N`, `quantityReserved -= N` |
| 4 | `promotionService.releasePromotion()` → `current_uses -= 1` |
| 5 | Order → **CANCELLED** |

**Broken/Missing Steps:**

- 🟡 MEDIUM — **Outstanding PENDING Transactions are not updated.** If the user initiated payment (creating a Transaction) but then cancels the order before IPN arrives, the Transaction stays PENDING. If VNPay IPN arrives later, `confirmOrder()` will try to confirm a CANCELLED order → silently skipped. But Transaction stays PENDING forever.
  - **Recommended fix:** In `BookingService.cancelOrder()`, also update all PENDING Transactions for this order to CANCELLED.

---

### C2: User Cancels After Successful Payment

**Steps:**

| # | What happens |
|---|---|
| 1 | Order is CONFIRMED, tickets are issued (VALID) |
| 2 | User wants to cancel/refund |
| 3 | `BookingService.cancelOrder()` → **REJECTS** with `"Chỉ có thể hủy đơn hàng đang chờ thanh toán"` |
| 4 | **Dead end — no refund API exists** |

**Broken/Missing Steps:**

- 🔴 CRITICAL — **No refund flow exists.** There is no `POST /api/orders/{id}/refund` endpoint. The `Order` entity has `REFUNDED` status, and `Transaction` entity has refund fields (`isRefunded`, `refundAmount`, `refundedAt`, `refundReason`), but **no service logic uses them**. A user who needs a refund has no self-service path.
  - **Recommended fix:** Create `RefundService.java` with the following flow:
    1. Validate order status == CONFIRMED and tickets not USED
    2. Call VNPay Refund API (`vnp_Command = refund`)
    3. Update Transaction: `isRefunded = true`, `refundAmount`, `refundedAt`
    4. Update Order: status → REFUNDED
    5. Update Tickets: status → CANCELLED
    6. Restore stock: `quantityAvailable += N`
    7. Release promotion slot
    
    ⚠️ ADR needed before implementation — policy decisions required:
    - Full refund only, or partial refund supported?
    - Refund window (e.g., only 24h before event)?
    - Auto-approve or admin-approval workflow?
    - Does buyer-initiated refund require organizer approval?

---

### C3: Organizer Cancels Entire Event

**Steps:**

| # | What happens |
|---|---|
| 1 | Organizer wants to cancel event |
| 2 | **No API exists** to cancel an event (`DELETE /api/organizer/events/{id}` is listed in gap analysis as MUST-HAVE, not implemented) |
| 3 | Dead end |

**Broken/Missing Steps:**

- 🔴 CRITICAL — **No event cancellation + bulk refund mechanism.** If an event is cancelled, there is no way to:
  1. Set event status to CANCELLED
  2. Find all CONFIRMED orders for that event
  3. Initiate bulk refund for all paid orders
  4. Cancel all VALID tickets
  5. Notify all ticket holders via email
  
  - **Recommended fix:** Implement `EventCancellationService.java`:
    1. `eventRepository.updateStatus(eventId, CANCELLED)`
    2. `orderRepository.findByEventIdAndStatus(eventId, CONFIRMED)` → batch refund
    3. `ticketRepository.updateStatusByEventId(eventId, CANCELLED)`
    4. Publish `EventCancelledEvent` → Email notification to all buyers
    
    ⚠️ ADR needed before implementation — requires VNPay bulk refund capability assessment and organizer-initiated vs admin-initiated cancellation workflow.

---

## Flow D: Infrastructure Failure Edge Cases

### D1: Cloudinary Down During QR Image Upload

**Steps:**

| # | What happens |
|---|---|
| 1 | `TicketMessageListener` receives `PaymentSuccessEvent` |
| 2 | `ticketIssuanceService.issueTickets()` → Tickets created in DB (**VALID**) ✅ |
| 3 | `qrCodeService.generateAndUploadForTickets()` → Cloudinary call fails |
| 4 | `QRCodeService` catches exception per-ticket (L59-63), logs error, **continues to next ticket** |
| 5 | Some/all tickets have `qrCodeImageUrl = null` |
| 6 | `TicketIssuedEvent` is published with `qrCodeImageUrl = null` for affected tickets |
| 7 | Email sent with missing QR images |
| 8 | `channel.basicAck()` — message **ACK'd successfully** even though QR upload failed |

**Broken/Missing Steps:**

- 🟠 HIGH — **QR upload failure is silently swallowed and ACK'd.** The message is removed from the queue after partial failure. Tickets exist in DB with `qrCodeImageUrl = null` and `qrCodeData` populated. There is no scheduled job or retry mechanism to fill in missing QR images later.
  - **Recommended fix:** Two options:
    1. **Prefer NACK:** If *any* QR upload fails, NACK the entire message (goes to DLQ). But this requires a DLQ consumer (see D3).
    2. **Background reconciliation job (recommended):** Add a `@Scheduled` method `QRReconciliationService` that queries `ticketRepository.findByQrCodeImageUrlIsNull()` and retries Cloudinary upload.

- 🟡 MEDIUM — **Email sent with broken/missing QR images.** The user receives an email with placeholder or broken image tags. They have no way to get the QR code except through `GET /api/tickets/{id}` (which returns `qrCodeData` but not a rendered image) or the hypothetical `GET /api/tickets/{id}/download-qr` (not implemented per gap analysis).
  - **Recommended fix:** `EmailService.buildEmailBody()` should include fallback text when `qrCodeImageUrl` is null: _"QR code ảnh đang được xử lý, vui lòng kiểm tra lại trong ứng dụng."_

---

### D2: Gmail/SMTP Quota Exhausted During Email Send

**Steps:**

| # | What happens |
|---|---|
| 1 | `EmailMessageListener` receives `TicketIssuedEvent` |
| 2 | `emailService.sendTicketConfirmationEmail()` → SMTP throws `MessagingException` |
| 3 | `EmailService.sendHtmlEmail()` wraps and re-throws as `RuntimeException` |
| 4 | `EmailMessageListener` catches in L49 → `channel.basicNack(deliveryTag, false, false)` |
| 5 | Message moves to `q.email.send.dlq` |
| 6 | **No DLQ consumer exists** — message sits in DLQ indefinitely |

**Broken/Missing Steps:**

- 🟠 HIGH — **Buyer never receives ticket email and there's no retry.** Tickets are issued correctly in DB, but the user doesn't get the email with QR codes. The message is in the DLQ with no consumer to retry it.
  - **Recommended fix:**
    1. **Short-term:** Implement a DLQ consumer for `q.email.send.dlq` with exponential backoff retry (e.g., 1min → 5min → 30min → 1h). After max retries, alert admin. See D3 for details.
    2. **Long-term:** Add `GET /api/tickets/{id}/download-qr` so users can self-serve retrieve QR codes even if email fails.

- 🟢 LOW — **No alert when SMTP quota is exhausted.** The system only logs `log.error()`. No integration with monitoring/alerting (e.g., Prometheus metric, Slack webhook).
  - **Recommended fix:** Increment a Micrometer counter on email failure for alerting.

---

### D3: RabbitMQ DLQ Fills Up (No Consumer)

**Steps:**

| # | What happens |
|---|---|
| 1 | Messages NACK'd from `q.ticket.issue` → `q.ticket.issue.dlq` |
| 2 | Messages NACK'd from `q.email.send` → `q.email.send.dlq` |
| 3 | No consumer on either DLQ |
| 4 | Messages accumulate indefinitely |
| 5 | If disk alarm triggers in RabbitMQ → **all publishers blocked** (RabbitMQ stops accepting new messages) |

**Broken/Missing Steps:**

- 🟠 HIGH — **DLQ has no consumer — confirmed as known architecture decision in project context, but still HIGH risk.** If `q.ticket.issue.dlq` accumulates, those represent **paid orders that never got tickets**. If `q.email.send.dlq` accumulates, those are ticket holders who never got their confirmation email.
  - **Recommended fix:** Create `DLQConsumerService.java`:
    ```java
    @RabbitListener(queues = RabbitMQConstants.QUEUE_TICKET_DLQ)
    public void handleTicketDLQ(PaymentSuccessEvent event, Channel channel,
                                @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
        // 1. Check if tickets were already issued (idempotent)
        // 2. If not, retry issueTickets()
        // 3. If still fails after N retries → log CRITICAL alert
        // 4. ACK to prevent infinite accumulation
    }
    ```
    - For ticket DLQ: retry with backoff. After 3 failures, persist to a `failed_issuances` table and alert admin.
    - For email DLQ: retry with backoff. After 3 failures, mark as `email_pending` and provide admin retry UI.

- 🟡 MEDIUM — **No TTL on DLQ messages.** Without `x-message-ttl` on DLQ queues, messages persist forever. If the system experiences a burst of failures, DLQ grows unbounded.
  - **Recommended fix:** Add `x-message-ttl` to DLQ queues (e.g., 72 hours) and monitor DLQ depth with alerts.

---

### D4: Redis Down Mid-Lock (Lock Not Released)

**Steps:**

| # | What happens |
|---|---|
| 1 | `BookingService.createBooking()` → `spamBucket.setIfAbsent()` → **RedisException** |
| 2a | If Redis is down before acquiring lock → `setIfAbsent()` throws exception → caught by Spring → 500 Internal Server Error to client |
| 2b | If Redis goes down AFTER lock acquired but BEFORE `finally { spamBucket.delete() }` → lock stays in Redis |
| 3 | After Redis recovers, the lock key persists with TTL (10s for spam lock, 30s for IPN lock) |
| 4 | After TTL expires, key auto-deletes → system recovers |

**For `TicketReservationService.acquireZoneLock()` (Redisson RLock):**

| # | What happens |
|---|---|
| 1 | Thread acquires RLock (lease 30s) |
| 2 | Redis goes down |
| 3 | Thread finishes booking, calls `releaseLock()` → **RedisException** |
| 4 | Lock is "orphaned" in Redis |
| 5 | When Redis recovers, lock has lease TTL → auto-expires after 30s |

**Broken/Missing Steps:**

- 🟡 MEDIUM — **Booking entirely blocked when Redis is unavailable.** Redis is a **hard dependency** for the booking flow (spam lock, distributed lock, stock check). If Redis goes down, no bookings can be created. This is a design choice (consistency over availability) but there is no fallback mechanism.
  - **Recommended fix:** The atomic DB `UPDATE ... WHERE available >= qty` (Guard 2) is sufficient as a standalone safety net. Consider making Redis locks **best-effort** with a try/catch that falls back to DB-only path when Redis is unavailable: 
    ```java
    try { lock = ticketReservationService.acquireZoneLock(id); }
    catch (RedisException e) { log.warn("Redis unavailable, falling back to DB-only"); }
    ```
    ⚠️ ADR needed before implementation — this trades consistency guarantees for availability. Acceptable for general sales, but NOT for flash sales.

- 🟢 LOW — **`spamBucket.delete()` in finally block can throw if Redis is down.** The RuntimeException from Redisson would propagate up and could mask the actual business exception.
  - **Recommended fix:** Wrap `spamBucket.delete()` in try/catch inside the finally block:
    ```java
    finally {
        try { spamBucket.delete(); } 
        catch (Exception e) { log.warn("Failed to delete spam lock, TTL will auto-expire", e); }
    }
    ```

---

## Summary Matrix

| ID | Severity | Flow | Issue | Fix Complexity |
|----|----------|------|-------|----------------|
| B2-1 | 🔴 CRITICAL | B2 | IPN never arrives → user paid but order expires, no refund | High (needs VNPay QueryDR) |
| B4-1 | 🔴 CRITICAL | B4 | Payment after order expiry → SUCCESS txn + EXPIRED order | Medium |
| C2-1 | 🔴 CRITICAL | C | No refund flow for paid orders | High (needs VNPay Refund API) |
| C3-1 | 🔴 CRITICAL | C | No event cancellation + bulk refund | High |
| A-1 | 🟡 MEDIUM | A | `confirmPromotion()` never called | Low |
| B1-1 | 🟡 MEDIUM | B1 | Orphan PENDING transactions after order expiry | Low |
| B4-2 | 🟠 HIGH | B4 | `confirmOrder()` silently swallows non-PENDING state | Low |
| C1-1 | 🟡 MEDIUM | C1 | PENDING transactions not cancelled on order cancel | Low |
| D1-1 | 🟠 HIGH | D1 | QR upload failure ACK'd, no retry | Medium |
| D1-2 | 🟡 MEDIUM | D1 | Email sent with missing QR images | Low |
| D2-1 | 🟠 HIGH | D2 | Email failure → DLQ → no retry | Medium |
| D3-1 | 🟠 HIGH | D3 | DLQ has no consumer (paid orders may never get tickets) | Medium |
| D3-2 | 🟡 MEDIUM | D3 | No TTL on DLQ messages | Low |
| D4-1 | 🟡 MEDIUM | D4 | Redis down = all bookings blocked | Medium (ADR needed) |
| D4-2 | 🟢 LOW | D4 | `spamBucket.delete()` can throw in finally | Low |
| A-2 | 🟡 MEDIUM | A | No frontend push notification after IPN | Medium (Phase 2C) |
| A-3 | 🟢 LOW | A | Expiration scheduler batch size may be slow | Low |
| B1-2 | 🟢 LOW | B1 | No user notification on order expiry | Low |
| D2-2 | 🟢 LOW | D2 | No alerting on SMTP quota exhaustion | Low |
