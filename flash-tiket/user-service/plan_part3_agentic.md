# Part 3 — Nhóm 2: Agentic (Booking, Payment, Full ReAct)

## Tổng quan kiến trúc Agentic

```
User: "Đặt 2 vé VIP concert Sơn Tùng"
         │
         ▼
┌─ DiscoveryAssistant (AiServices interface) ──────────┐
│  System prompt + Chat memory + Tools                  │
│                                                       │
│  LLM reasoning (ReAct):                              │
│   Thought: User muốn đặt vé, cần tìm event trước    │
│   Action:  eventSearchTool("concert Sơn Tùng")       │
│   Observe: Found event ID=xxx, VIP ticketTypeId=yyy  │
│   Thought: Cần xác nhận với user trước khi đặt       │
│   → "Tìm thấy Concert Sơn Tùng 15/6. VIP 1.5M.     │
│      Bạn muốn đặt 2 vé VIP không?"                   │
│                                                       │
│  User: "Đặt luôn"                                    │
│   Thought: User confirm, gọi booking tool            │
│   Action:  bookingTool(eventId, ticketTypeId, qty=2)  │
│   Observe: Order TB-20260615-123456 created (PENDING)│
│   Thought: Cần hỏi user muốn thanh toán không       │
│   → "Đã tạo đơn TB-20260615-123456. Tổng 3M VND.   │
│      Bạn muốn thanh toán qua VNPay không?"           │
│                                                       │
│  User: "Thanh toán đi"                               │
│   Action:  paymentTool(orderId)                       │
│   → "Link thanh toán: https://vnpay.vn/..."          │
└───────────────────────────────────────────────────────┘
```

> [!IMPORTANT]
> **Ràng buộc bắt buộc**: Mọi write action (booking, payment) **PHẢI** có user confirmation trong chat trước khi thực hiện. Tool KHÔNG tự ý tạo booking.

---

## Feature 5: Booking Creation Tool

### Mô tả kỹ thuật
`@Tool` gọi REST API `POST /api/bookings` trên core-service, **forward JWT gốc của user**. Chỉ tạo Order(PENDING) — không confirm trực tiếp.

### File: `shared/client/CoreServiceClient.java`
```java
package com.flashticket.discovery.shared.client;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.*;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;
import org.springframework.web.client.RestClientResponseException;

/**
 * REST client gọi core-service qua Eureka load balancer.
 * Forward JWT gốc của user để core-service validate IDOR.
 *
 * Dùng RestClient (Spring 6.1+) — nhất quán với pattern hiện tại.
 */
@Component
@Slf4j
public class CoreServiceClient {

    private final RestClient restClient;

    public CoreServiceClient(
            RestClient.Builder builder,
            @Value("${app.core-service.base-url}") String baseUrl) {
        this.restClient = builder.baseUrl(baseUrl).build();
    }

    /** POST /api/bookings — Tạo booking mới */
    public String createBooking(String jwt, CreateBookingPayload payload) {
        try {
            return restClient.post()
                    .uri("/api/bookings")
                    .header(HttpHeaders.AUTHORIZATION, "Bearer " + jwt)
                    .contentType(MediaType.APPLICATION_JSON)
                    .body(payload)
                    .retrieve()
                    .body(String.class); // Return raw JSON
        } catch (RestClientResponseException e) {
            log.error("[CoreClient] Booking failed: {} — {}", e.getStatusCode(), e.getResponseBodyAsString());
            return "ERROR: " + extractErrorMessage(e);
        }
    }

    /** POST /api/payments/initiate — Tạo payment URL */
    public String initiatePayment(String jwt, PaymentInitPayload payload) {
        try {
            return restClient.post()
                    .uri("/api/payments/initiate")
                    .header(HttpHeaders.AUTHORIZATION, "Bearer " + jwt)
                    .contentType(MediaType.APPLICATION_JSON)
                    .body(payload)
                    .retrieve()
                    .body(String.class);
        } catch (RestClientResponseException e) {
            log.error("[CoreClient] Payment failed: {} — {}", e.getStatusCode(), e.getResponseBodyAsString());
            return "ERROR: " + extractErrorMessage(e);
        }
    }

    /** GET /api/events (public) — Search events */
    public String searchEvents(String query, int page, int size) {
        return restClient.get()
                .uri(uriBuilder -> uriBuilder
                        .path("/api/events")
                        .queryParam("keyword", query)
                        .queryParam("page", page)
                        .queryParam("size", size)
                        .build())
                .retrieve()
                .body(String.class);
    }

    /** GET /api/events/{id} (public) — Event detail */
    public String getEventDetail(String eventId) {
        return restClient.get()
                .uri("/api/events/{id}", eventId)
                .retrieve()
                .body(String.class);
    }

    private String extractErrorMessage(RestClientResponseException e) {
        try {
            // Parse ErrorResponse format from core-service GlobalExceptionHandler
            var body = e.getResponseBodyAsString();
            if (body.contains("message")) {
                return body; // Return full error for LLM to interpret
            }
            return e.getStatusText();
        } catch (Exception ex) {
            return e.getMessage();
        }
    }

    // ── Payload records ──
    public record CreateBookingPayload(
        java.util.UUID eventId,
        java.util.List<BookingItem> items,
        String customerName, String customerEmail,
        String customerPhone, String promotionCode,
        String customerNote
    ) {
        public record BookingItem(java.util.UUID ticketTypeId, int quantity) {}
    }

    public record PaymentInitPayload(
        java.util.UUID orderId,
        String bankCode,
        String provider
    ) {}
}
```

### File: `agent/tool/EventSearchTool.java`
```java
package com.flashticket.discovery.agent.tool;

import com.flashticket.discovery.shared.client.CoreServiceClient;
import dev.langchain4j.agent.tool.P;
import dev.langchain4j.agent.tool.Tool;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

/**
 * Tool tìm kiếm sự kiện qua core-service REST API.
 * Kết quả trả về JSON cho LLM parse.
 */
@Component
@RequiredArgsConstructor
public class EventSearchTool {

    private final CoreServiceClient coreClient;

    @Tool("Tìm kiếm sự kiện theo từ khóa. Trả về danh sách sự kiện với id, title, date, price, venue.")
    public String searchEvents(
            @P("Từ khóa tìm kiếm sự kiện") String keyword) {
        return coreClient.searchEvents(keyword, 0, 5);
    }

    @Tool("Xem chi tiết một sự kiện cụ thể bằng eventId. Trả về thông tin đầy đủ gồm loại vé, giá, số lượng còn.")
    public String getEventDetail(
            @P("UUID của sự kiện cần xem chi tiết") String eventId) {
        return coreClient.getEventDetail(eventId);
    }
}
```

### File: `agent/tool/BookingTool.java`
```java
package com.flashticket.discovery.agent.tool;

import com.flashticket.discovery.shared.client.CoreServiceClient;
import com.flashticket.discovery.shared.client.CoreServiceClient.CreateBookingPayload;
import com.flashticket.discovery.shared.client.CoreServiceClient.CreateBookingPayload.BookingItem;
import dev.langchain4j.agent.tool.P;
import dev.langchain4j.agent.tool.Tool;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.UUID;

/**
 * Tool tạo đơn đặt vé — gọi POST /api/bookings trên core-service.
 *
 * RÀNG BUỘC QUAN TRỌNG:
 * - Chỉ gọi SAU KHI user confirm rõ ràng trong chat
 * - Chỉ tạo Order(PENDING) — không confirm trực tiếp
 * - Forward JWT gốc → core-service validate IDOR + TOCTOU protection
 * - Không bypass BookingService security guards
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class BookingTool {

    private final CoreServiceClient coreClient;

    @Tool("""
        Tạo đơn đặt vé cho user. CHỈ gọi khi user đã XÁC NHẬN muốn đặt.
        Trả về thông tin đơn hàng nếu thành công, hoặc lỗi nếu thất bại.
        Đơn sẽ có trạng thái PENDING và hết hạn sau 15 phút nếu chưa thanh toán.
        """)
    public String createBooking(
            @P("UUID của sự kiện") String eventId,
            @P("UUID của loại vé") String ticketTypeId,
            @P("Số lượng vé (1-10)") int quantity,
            @P("Tên khách hàng") String customerName,
            @P("Email khách hàng") String customerEmail,
            @P("Số điện thoại (optional)") String customerPhone,
            @P("Mã voucher (optional, null nếu không có)") String promotionCode,
            @P("JWT token của user") String jwtToken) {

        log.info("[BookingTool] Creating booking: event={}, ticket={}, qty={}",
                eventId, ticketTypeId, quantity);

        var payload = new CreateBookingPayload(
                UUID.fromString(eventId),
                List.of(new BookingItem(UUID.fromString(ticketTypeId), quantity)),
                customerName, customerEmail, customerPhone,
                promotionCode, null
        );

        return coreClient.createBooking(jwtToken, payload);
    }
}
```

---

## Feature 6: Payment Redirect Tool

### File: `agent/tool/PaymentTool.java`
```java
package com.flashticket.discovery.agent.tool;

import com.flashticket.discovery.shared.client.CoreServiceClient;
import com.flashticket.discovery.shared.client.CoreServiceClient.PaymentInitPayload;
import dev.langchain4j.agent.tool.P;
import dev.langchain4j.agent.tool.Tool;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.UUID;

/**
 * Tool tạo VNPay payment link — gọi POST /api/payments/initiate trên core-service.
 *
 * RÀNG BUỘC:
 * - Chỉ gọi SAU KHI user confirm muốn thanh toán
 * - Chỉ hoạt động với Order status = PENDING
 * - PaymentService trên core-service tự validate IDOR + spam lock
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class PaymentTool {

    private final CoreServiceClient coreClient;

    @Tool("""
        Tạo link thanh toán VNPay cho đơn hàng. CHỈ gọi khi user XÁC NHẬN muốn thanh toán.
        Trả về URL thanh toán để user click vào.
        Đơn hàng phải đang ở trạng thái PENDING.
        """)
    public String createPaymentLink(
            @P("UUID của đơn hàng cần thanh toán") String orderId,
            @P("Mã ngân hàng (optional, null để user tự chọn trên VNPay)") String bankCode,
            @P("JWT token của user") String jwtToken) {

        log.info("[PaymentTool] Creating payment for order: {}", orderId);

        var payload = new PaymentInitPayload(
                UUID.fromString(orderId), bankCode, "VNPAY");

        return coreClient.initiatePayment(jwtToken, payload);
    }
}
```

### File: `agent/tool/VoucherTool.java`
```java
package com.flashticket.discovery.agent.tool;

import com.flashticket.discovery.shared.client.CoreServiceClient;
import dev.langchain4j.agent.tool.P;
import dev.langchain4j.agent.tool.Tool;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

/**
 * Tool tra cứu voucher/promotion — gọi core-service API.
 * Read-only: chỉ kiểm tra voucher có hợp lệ không, không reserve.
 */
@Component
@RequiredArgsConstructor
public class VoucherTool {

    private final CoreServiceClient coreClient;

    @Tool("Kiểm tra mã voucher có hợp lệ không, trả về thông tin giảm giá nếu có.")
    public String checkVoucher(
            @P("Mã voucher cần kiểm tra") String voucherCode) {
        // TODO: Thêm endpoint GET /api/promotions/check/{code} vào core-service
        // Đây là read-only, không thay đổi business logic
        return "Tính năng kiểm tra voucher đang được phát triển.";
    }
}
```

---

## Feature 7: Full Agentic — ReAct Loop

### File: `chat/service/DiscoveryAssistant.java` (AiServices Interface)
```java
package com.flashticket.discovery.chat.service;

import dev.langchain4j.service.*;

/**
 * LangChain4j AiServices interface — entry point cho toàn bộ AI.
 *
 * ReAct loop tự động: LLM nhận tools → quyết định gọi tool nào →
 * quan sát kết quả → tiếp tục reasoning → trả lời user.
 *
 * System prompt chỉ dẫn:
 * 1. LUÔN xác nhận trước khi booking/payment
 * 2. Trả lời bằng tiếng Việt
 * 3. Gợi ý voucher nếu có
 * 4. Tone phù hợp mood user
 */
@SystemMessage("""
    Bạn là trợ lý AI của FlashTicket — nền tảng bán vé sự kiện trực tuyến.

    NGUYÊN TẮC BẮT BUỘC:
    1. Trả lời bằng TIẾNG VIỆT, thân thiện, chuyên nghiệp.
    2. Khi user muốn ĐẶT VÉ: tìm sự kiện → hiển thị thông tin → HỎI XÁC NHẬN → mới gọi bookingTool.
    3. Khi user muốn THANH TOÁN: hiển thị tổng tiền → HỎI XÁC NHẬN → mới gọi paymentTool.
    4. KHÔNG BAO GIỜ tự ý đặt vé hay thanh toán mà chưa được user đồng ý.
    5. Nếu gặp lỗi từ tool, giải thích dễ hiểu cho user, đề xuất giải pháp.
    6. Gợi ý voucher/khuyến mãi nếu biết có.

    THÔNG TIN HỆ THỐNG:
    - Đơn hàng PENDING sẽ hết hạn sau 15 phút
    - Thanh toán qua VNPay (link redirect)
    - Mỗi user chỉ có 1 đơn PENDING per event
    """)
public interface DiscoveryAssistant {

    @UserMessage("{{message}}")
    String chat(@MemoryId String sessionId,
                @V("message") String message);
}
```

### File: `config/LangChainConfig.java` — Thêm AiServices bean
```java
// Thêm vào LangChainConfig.java (bổ sung từ Part 2)

@Bean
DiscoveryAssistant discoveryAssistant(
        ChatLanguageModel chatModel,
        EventSearchTool eventSearchTool,
        BookingTool bookingTool,
        PaymentTool paymentTool,
        VoucherTool voucherTool,
        ContentRetriever eventContentRetriever,
        ChatMemoryProvider chatMemoryProvider) {

    return AiServices.builder(DiscoveryAssistant.class)
            .chatLanguageModel(chatModel)
            .chatMemoryProvider(chatMemoryProvider)
            .contentRetriever(eventContentRetriever)  // RAG
            .tools(eventSearchTool, bookingTool, paymentTool, voucherTool)
            .build();
}

@Bean
ChatMemoryProvider chatMemoryProvider(PersistentChatMemory persistentMemory) {
    return sessionId -> MessageWindowChatMemory.builder()
            .id(sessionId)
            .maxMessages(20)  // Giữ 20 messages gần nhất
            .chatMemoryStore(persistentMemory)
            .build();
}
```

### File: `chat/service/ChatService.java`
```java
package com.flashticket.discovery.chat.service;

import com.flashticket.discovery.chat.dto.*;
import com.flashticket.discovery.mood.MoodDetector;
import com.flashticket.discovery.rag.strategy.AdaptiveRagRouter;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

/**
 * Main entry point — orchestrate mood detection + RAG + Agent.
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class ChatService {

    private final DiscoveryAssistant assistant;
    private final MoodDetector moodDetector;
    private final AdaptiveRagRouter ragRouter;

    public ChatResponse processMessage(String userId, String sessionId,
                                        String message, String jwtToken) {
        // 1. Detect mood
        var mood = moodDetector.detect(message);
        log.info("[Chat] User={}, Mood={}, Message='{}'", userId, mood, message);

        // 2. Enhance message with JWT context (tool cần JWT)
        // JWT được inject vào tool context thông qua ThreadLocal
        JwtContextHolder.set(jwtToken);

        try {
            // 3. Call assistant (ReAct loop tự động)
            String response = assistant.chat(sessionId, message);

            return new ChatResponse(
                    sessionId, response, mood.name(),
                    ragRouter.route(message).strategy()
            );
        } finally {
            JwtContextHolder.clear();
        }
    }
}
```

### File: `chat/controller/ChatController.java`
```java
package com.flashticket.discovery.chat.controller;

import com.flashticket.discovery.chat.dto.*;
import com.flashticket.discovery.chat.service.ChatService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/chat")
@RequiredArgsConstructor
public class ChatController {

    private final ChatService chatService;

    @PostMapping
    public ResponseEntity<ChatResponse> chat(
            @Valid @RequestBody ChatRequest request,
            @AuthenticationPrincipal Jwt jwt) {
        String userId = jwt.getSubject();
        String jwtToken = jwt.getTokenValue();

        ChatResponse response = chatService.processMessage(
                userId, request.sessionId(), request.message(), jwtToken);

        return ResponseEntity.ok(response);
    }
}
```

### File: `chat/dto/ChatRequest.java` + `ChatResponse.java`
```java
// ChatRequest.java
public record ChatRequest(
    @jakarta.validation.constraints.NotBlank String sessionId,
    @jakarta.validation.constraints.NotBlank @jakarta.validation.constraints.Size(max = 2000) String message
) {}

// ChatResponse.java
public record ChatResponse(
    String sessionId,
    String message,
    String mood,
    String ragStrategy
) {}
```

### JWT Context cho Tools — ThreadLocal Pattern
```java
package com.flashticket.discovery.chat.service;

/**
 * ThreadLocal holder để pass JWT token từ ChatService → Tools.
 * BookingTool và PaymentTool cần JWT để forward qua core-service.
 */
public final class JwtContextHolder {
    private static final ThreadLocal<String> HOLDER = new ThreadLocal<>();
    public static void set(String jwt) { HOLDER.set(jwt); }
    public static String get() { return HOLDER.get(); }
    public static void clear() { HOLDER.remove(); }
    private JwtContextHolder() {}
}
```

> [!WARNING]
> **BookingTool/PaymentTool** phải gọi `JwtContextHolder.get()` thay vì nhận JWT qua LLM parameter. LLM KHÔNG nên xử lý JWT token — security risk.

---

## Sửa đổi BookingTool để dùng JwtContextHolder

```java
// Cập nhật BookingTool.createBooking() — bỏ param jwtToken
@Tool("Tạo đơn đặt vé. CHỈ gọi khi user XÁC NHẬN.")
public String createBooking(
        @P("UUID sự kiện") String eventId,
        @P("UUID loại vé") String ticketTypeId,
        @P("Số lượng vé") int quantity,
        @P("Tên khách hàng") String customerName,
        @P("Email khách hàng") String customerEmail,
        @P("Số điện thoại") String customerPhone,
        @P("Mã voucher (null nếu không)") String promotionCode) {

    String jwt = JwtContextHolder.get(); // Lấy từ ThreadLocal
    if (jwt == null) return "ERROR: Phiên đăng nhập hết hạn, vui lòng đăng nhập lại.";
    // ... rest of logic
}
```

---

## Files core-service cần thêm (KHÔNG sửa business logic)

> [!IMPORTANT]
> Chỉ thêm 1 dòng publish event. Không thay đổi flow hiện có.

### `core-service/.../event/service/OrganizerEventService.java`
```java
// Thêm inject:
private final ApplicationEventPublisher eventPublisher;

// Trong method publishEvent() hoặc updateEvent() — SAU KHI save thành công:
eventPublisher.publishEvent(new EventSyncSpringEvent(savedEvent));
```

### `core-service/.../shared/event/EventSyncSpringEvent.java` [NEW]
```java
/** Spring ApplicationEvent nội bộ — trigger RabbitMQ publish SAU DB commit */
public record EventSyncSpringEvent(Event event) {}
```

### `core-service/.../shared/event/EventSyncPublisher.java` [NEW]
```java
@Component @RequiredArgsConstructor
public class EventSyncPublisher {
    private final RabbitTemplate rabbitTemplate;

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleEventSync(EventSyncSpringEvent event) {
        // Build DTO from Event entity → publish to ex.discovery
        rabbitTemplate.convertAndSend("ex.discovery", "event.sync", buildSyncMessage(event.event()));
    }
}
```

---

## Checklist nhóm 2 — Trước khi chạy

- [ ] `CoreServiceClient` connect được tới core-service qua Eureka
- [ ] JWT forward: discovery-service extract JWT → pass header → core-service validate OK
- [ ] BookingTool: test tạo booking PENDING thành công
- [ ] PaymentTool: test tạo VNPay URL thành công
- [ ] ReAct loop: LLM tự search → show info → ask confirm → book
- [ ] JwtContextHolder ThreadLocal không leak (finally clear)
- [ ] core-service `EventSyncPublisher` publish message sau DB commit

## Potential Errors — Nhóm 2

| Error | Dấu hiệu | Fix |
|---|---|---|
| `RestClientResponseException 401` | Core-service reject JWT | Verify JWT token format, check `issuer-uri` match |
| `InvalidRequestException: Bạn đang có 1 đơn PENDING` | User đã có order | LLM nên guide user hủy đơn cũ hoặc thanh toán |
| `InsufficientStockException` | Hết vé | LLM trả lời "Rất tiếc, loại vé X đã hết" |
| Tool timeout > 10s | Core-service cold start | Tăng timelimiter, thêm retry |
| LLM gọi bookingTool mà chưa confirm | System prompt bị ignore | Thêm guard check trong tool: kiểm tra conversation history |
| ThreadLocal leak | JWT từ user A lọt vào request user B | Đảm bảo `finally { JwtContextHolder.clear() }` |

---

> **Next**: Part 4 — Mood-Aware Layer + Part 5 — Thứ tự triển khai, Rủi ro, Verification
