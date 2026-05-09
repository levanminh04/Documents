# Tư vấn chi tiết — 5 câu hỏi trước khi triển khai

---

## Câu 1: Gemini API Key

### Tình trạng hiện tại
Trong `configserver/.../discovery-service.yml` (plan), ta cần:
```yaml
langchain4j:
  google-ai-gemini:
    chat-model:
      api-key: ${GEMINI_API_KEY}
```

### Hướng dẫn lấy API Key (miễn phí)

**Bước 1**: Truy cập [Google AI Studio](https://aistudio.google.com/apikey)

**Bước 2**: Đăng nhập bằng Google Account

**Bước 3**: Click **"Create API Key"** → Chọn project hoặc tạo mới → Copy key

**Bước 4**: Thêm vào file `.env` (hoặc environment variables):
```env
GEMINI_API_KEY=AIzaSy...your_key_here
```

### Giới hạn Free Tier (đủ cho dev + test)

| Model | Requests/phút | Requests/ngày | Token/phút |
|---|---|---|---|
| **gemini-2.0-flash** | 15 RPM | 1,500 RPD | 1M TPM |
| gemini-1.5-pro | 2 RPM | 50 RPD | 32k TPM |
| text-embedding-004 | 1,500 RPM | Unlimited | N/A |

> [!TIP]
> **Khuyến nghị**: Dùng `gemini-2.0-flash` cho cả chat + classification (mood, RAG router). Free tier 15 RPM đủ cho dev. Production chuyển sang Pay-as-you-go ($0.10/1M input tokens).

### Verify API Key hoạt động
```bash
curl "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"contents":[{"parts":[{"text":"Hello"}]}]}'
```

Nếu trả về JSON có `candidates` → key hoạt động ✅

### Fallback OpenAI (optional)
Nếu muốn dual-provider (Gemini primary + OpenAI fallback):
```env
OPENAI_API_KEY=sk-...your_key_here
```
OpenAI không có free tier, nhưng GPT-4o-mini rất rẻ ($0.15/1M input tokens). Không bắt buộc — có thể để trống, hệ thống chỉ dùng Gemini.

### ✅ Khuyến nghị của tôi
> Lấy **Gemini API key miễn phí** từ Google AI Studio. Đủ cho toàn bộ quá trình dev + test. Không cần OpenAI key ở giai đoạn này.

---

## Câu 2: PGVector Extension

### Tình trạng hiện tại
Trong `docker-compose.yml` dự án đã dùng:
```yaml
postgres:
  image: pgvector/pgvector:pg14  # ← Image này đã CÀI SẴN pgvector extension
```

**Tuy nhiên**, có extension trong image ≠ đã ENABLE trong database. Cần chạy `CREATE EXTENSION` một lần.

### Bước kiểm tra (chạy qua pgAdmin hoặc psql)

**Bước 1**: Kết nối PostgreSQL
```bash
# Qua docker
docker exec -it postgres_flash_ticket psql -U postgres

# Hoặc qua pgAdmin: http://localhost:5050
```

**Bước 2**: Kiểm tra extension đã có chưa
```sql
SELECT * FROM pg_extension WHERE extname = 'vector';
```
- Nếu trả về 1 row → đã enable ✅
- Nếu trả về 0 row → cần enable ⬇️

**Bước 3**: Enable extension (nếu chưa có)
```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

**Bước 4**: Verify
```sql
-- Phải trả về thành công, không lỗi
SELECT '[1,2,3]'::vector;
```

### Schema migration cho discovery-service

Dự án hiện tại **chưa dùng Flyway** (core-service comment out flyway config, dùng `ddl-auto: validate`). Có 2 lựa chọn:

**Option A — Chạy SQL thủ công (đơn giản, phù hợp giai đoạn dev):**
```sql
-- Chạy 1 lần trước khi start discovery-service
CREATE EXTENSION IF NOT EXISTS vector;
CREATE SCHEMA IF NOT EXISTS discovery_schema;

CREATE TABLE discovery_schema.chat_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id VARCHAR(255) NOT NULL,
    title VARCHAR(500),
    mood VARCHAR(20) DEFAULT 'NEUTRAL',
    message_count INTEGER DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    is_active BOOLEAN DEFAULT TRUE
);
CREATE INDEX idx_chat_sessions_user ON discovery_schema.chat_sessions(user_id);

CREATE TABLE discovery_schema.chat_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES discovery_schema.chat_sessions(id),
    role VARCHAR(20) NOT NULL,
    content TEXT NOT NULL,
    tool_name VARCHAR(100),
    tool_result TEXT,
    mood VARCHAR(20),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_chat_messages_session ON discovery_schema.chat_messages(session_id);

-- PGVector embedding table tự tạo bởi LangChain4j (createTable=true)
```

**Option B — Dùng Flyway (best practice cho production):**
- Thêm dependency `flyway-core` + `flyway-database-postgresql` vào `pom.xml`
- Tạo file `src/main/resources/db/migration/V1__init_discovery_schema.sql`
- Config trong `discovery-service.yml`:
```yaml
spring:
  flyway:
    enabled: true
    schemas: discovery_schema
    locations: classpath:db/migration
```

### ✅ Khuyến nghị của tôi
> **Giai đoạn dev**: Option A (chạy SQL thủ công). Nhanh, đơn giản, nhất quán với cách core-service đang làm.
> **Trước khi lên production**: Chuyển sang Option B (Flyway) cho cả core-service và discovery-service.

---

## Câu 3: Sửa đổi core-service (Event Publisher)

### Scope sửa đổi — CHỈ 3 files mới + 1 dòng sửa

> [!IMPORTANT]
> **Nguyên tắc**: KHÔNG thay đổi bất kỳ business logic nào. Chỉ thêm event publishing SAU KHI transaction commit thành công.

#### File 1: `core-service/.../shared/messaging/RabbitMQConstants.java` — Thêm 3 constants

```diff
 public final class RabbitMQConstants {
     // ... existing constants ...

+    // ── Discovery Service sync ──
+    public static final String EXCHANGE_DISCOVERY = "ex.discovery";
+    public static final String RK_EVENT_SYNC = "event.sync";
+    public static final String QUEUE_EVENT_SYNC = "q.discovery.event.sync";
+    public static final String QUEUE_EVENT_SYNC_DLQ = "q.discovery.event.sync.dlq";
 }
```

#### File 2: `core-service/.../shared/event/EventSyncSpringEvent.java` [MỚI]

```java
package com.flashticket.core.shared.event;

import com.flashticket.core.event.entity.Event;

/**
 * Spring ApplicationEvent nội bộ.
 * Trigger RabbitMQ publish SAU KHI DB transaction commit (AFTER_COMMIT).
 * Không ảnh hưởng transaction hiện tại.
 */
public record EventSyncSpringEvent(Event event, String action) {
    // action: "PUBLISHED", "UPDATED", "DELETED"
}
```

#### File 3: `core-service/.../shared/event/EventSyncPublisher.java` [MỚI]

```java
package com.flashticket.core.shared.event;

import com.flashticket.core.event.entity.Event;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;
import org.springframework.transaction.event.TransactionPhase;
import org.springframework.transaction.event.TransactionalEventListener;

import static com.flashticket.core.shared.messaging.RabbitMQConstants.*;

/**
 * Publish event data → discovery-service qua RabbitMQ.
 *
 * Sử dụng @TransactionalEventListener(AFTER_COMMIT):
 * - Chỉ publish SAU KHI DB commit thành công
 * - Nếu transaction rollback → KHÔNG publish (tránh stale data)
 * - @Async: không block thread chính
 *
 * Pattern nhất quán với TicketMessageListener đã có.
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class EventSyncPublisher {

    private final RabbitTemplate rabbitTemplate;

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleEventSync(EventSyncSpringEvent springEvent) {
        Event event = springEvent.event();
        try {
            var message = buildSyncMessage(event, springEvent.action());
            rabbitTemplate.convertAndSend(EXCHANGE_DISCOVERY, RK_EVENT_SYNC, message);
            log.info("[EventSync] Published to discovery: eventId={}, action={}",
                    event.getId(), springEvent.action());
        } catch (Exception e) {
            // Log only — không throw để không ảnh hưởng main flow
            log.error("[EventSync] Failed to publish: eventId={}", event.getId(), e);
        }
    }

    private java.util.Map<String, Object> buildSyncMessage(Event event, String action) {
        return java.util.Map.ofEntries(
            java.util.Map.entry("eventId", event.getId()),
            java.util.Map.entry("title", event.getTitle()),
            java.util.Map.entry("description", nullSafe(event.getDescription())),
            java.util.Map.entry("status", action),
            java.util.Map.entry("minPrice", event.getMinPrice()),
            java.util.Map.entry("startDatetime", String.valueOf(event.getStartDatetime())),
            java.util.Map.entry("venueName", event.getVenue() != null ? event.getVenue().getName() : ""),
            java.util.Map.entry("organizerId", nullSafe(event.getOrganizerId()))
        );
    }

    private String nullSafe(String s) { return s != null ? s : ""; }
}
```

#### Dòng sửa duy nhất: `OrganizerEventService.java`

```diff
 @Service
 public class OrganizerEventService {
     // ... existing fields ...
+    private final ApplicationEventPublisher eventPublisher;

     // Trong method publishEvent() — SAU dòng eventRepository.save(event):
+    eventPublisher.publishEvent(new EventSyncSpringEvent(event, "PUBLISHED"));

     // Trong method updateEvent() — SAU dòng eventRepository.save(event):
+    eventPublisher.publishEvent(new EventSyncSpringEvent(event, "UPDATED"));
 }
```

### Phân tích an toàn

| Câu hỏi | Trả lời |
|---|---|
| Có thay đổi BookingService không? | ❌ KHÔNG |
| Có thay đổi PaymentService không? | ❌ KHÔNG |
| Có thay đổi database schema không? | ❌ KHÔNG |
| Có thay đổi API response không? | ❌ KHÔNG |
| Nếu RabbitMQ down thì sao? | Log error, main flow vẫn chạy bình thường (catch Exception + @Async) |
| Nếu discovery-service down thì sao? | Message nằm trong queue, consumer xử lý khi service up lại |

### ✅ Khuyến nghị
> **Hoàn toàn an toàn**. `@TransactionalEventListener(AFTER_COMMIT)` + `@Async` + try-catch đảm bảo zero impact lên business logic hiện tại. Đây là pattern đã proven trong hệ thống (tương tự `TicketMessageListener`).

---

## Câu 4: Chat Memory — Persist vs In-Memory

### So sánh chi tiết

| Tiêu chí | In-Memory | Persist (PostgreSQL) |
|---|---|---|
| **Tốc độ** | ⚡ Cực nhanh (HashMap) | 🟢 Nhanh (indexed query) |
| **Restart survive** | ❌ Mất hết khi restart | ✅ Giữ nguyên |
| **Multi-instance** | ❌ Mỗi instance riêng biệt | ✅ Shared across instances |
| **Conversation history** | ❌ Không xem lại được | ✅ User xem lại chat cũ |
| **Debug/Audit** | ❌ Không trace được | ✅ Query DB để debug |
| **RAM usage** | ⚠️ Tăng theo số user | ✅ Offload xuống DB |
| **Complexity** | ✅ 0 code thêm | 🟡 Thêm entity + repository |
| **Phù hợp giai đoạn** | Dev/prototype | Production |

### Cách triển khai từng option

**Option A — In-Memory (LangChain4j built-in):**
```java
@Bean
ChatMemoryProvider chatMemoryProvider() {
    return sessionId -> MessageWindowChatMemory.builder()
            .id(sessionId)
            .maxMessages(20)
            .build();
    // Dùng InMemoryChatMemoryStore mặc định
}
```
- Ưu điểm: Zero config, 3 dòng code
- Nhược điểm: Restart = mất hết, không scale

**Option B — Persist (PostgreSQL):**
```java
@Component
public class PersistentChatMemory implements ChatMemoryStore {

    private final ChatMessageRepository repository;

    @Override
    public List<ChatMessage> getMessages(Object sessionId) {
        return repository.findBySessionIdOrderByCreatedAt(sessionId.toString())
                .stream()
                .map(this::toChatMessage)
                .toList();
    }

    @Override
    public void updateMessages(Object sessionId, List<ChatMessage> messages) {
        // Lưu message mới nhất vào DB
        var latest = messages.get(messages.size() - 1);
        repository.save(ChatMessageEntity.from(sessionId.toString(), latest));
    }

    @Override
    public void deleteMessages(Object sessionId) {
        repository.deleteBySessionId(sessionId.toString());
    }
}
```

### ✅ Khuyến nghị của tôi

> **Chiến lược 2 giai đoạn:**
> 1. **Phase 1 (dev)**: Dùng **In-Memory** — nhanh, đơn giản, tập trung vào RAG + Agent logic
> 2. **Phase 2 (trước production)**: Chuyển sang **Persist** — chỉ cần implement `ChatMemoryStore` interface, không thay đổi code khác
>
> Lý do: Schema `chat_sessions` + `chat_messages` đã thiết kế sẵn (Part 1). Khi chuyển sang Persist chỉ mất ~2 tiếng code.

---

## Câu 5: Streaming (SSE) vs Batch Response

### So sánh

| Tiêu chí | Batch (đợi hết rồi trả) | SSE (stream từng token) |
|---|---|---|
| **UX trải nghiệm** | ⚠️ User đợi 3-8s blank | ✅ Thấy chữ hiện dần (ChatGPT-like) |
| **Perceived latency** | Cao (user nghĩ bị lag) | Thấp (feedback ngay lập tức) |
| **Implementation** | ✅ Đơn giản (REST POST/Response) | 🟡 Phức tạp hơn (SSE/WebFlux) |
| **Tool execution** | ✅ Đơn giản (chờ tool xong rồi trả) | ⚠️ Phức tạp (tool blocking giữa stream) |
| **Error handling** | ✅ Dễ (HTTP status code) | 🟡 Khó hơn (stream đã mở) |
| **Frontend** | ✅ `fetch()` thường | 🟡 `EventSource` hoặc `fetch` + ReadableStream |
| **LangChain4j support** | ✅ Có sẵn | ✅ Có `StreamingChatLanguageModel` |

### Cách triển khai SSE (nếu chọn)

```java
// Controller
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> streamChat(@RequestParam String sessionId,
                                @RequestParam String message,
                                @AuthenticationPrincipal Jwt jwt) {
    return chatService.streamResponse(sessionId, message, jwt.getTokenValue());
}

// Service — dùng StreamingChatLanguageModel
StreamingChatLanguageModel streamingModel; // Bean riêng

public Flux<String> streamResponse(...) {
    return Flux.create(sink -> {
        streamingModel.generate(messages, new StreamingResponseHandler<>() {
            @Override public void onNext(String token) { sink.next(token); }
            @Override public void onComplete(Response<AiMessage> r) { sink.complete(); }
            @Override public void onError(Throwable e) { sink.error(e); }
        });
    });
}
```

### Vấn đề kỹ thuật với SSE + Agentic Tools

> [!WARNING]
> Khi LLM gọi `BookingTool` → tool cần 1-3s để REST call → **stream bị pause** giữa chừng.
> User thấy: `"Đang tìm sự kiện..."` → im lặng 3s → `"Đã tạo đơn hàng..."`.
> Giải pháp: Gửi "thinking" events: `data: {"type":"thinking","text":"Đang tạo đơn hàng..."}`

### ✅ Khuyến nghị của tôi

> **Chiến lược 2 giai đoạn (giống chat memory):**
> 1. **Phase 1**: Dùng **Batch Response** — tập trung vào core RAG + Agent logic hoạt động đúng
> 2. **Phase 2**: Thêm **SSE endpoint** song song — không xóa batch endpoint, thêm `/api/chat/stream`
>
> Lý do: Tool execution (booking, payment) + ReAct loop tạo ra latency không đều. Batch response dễ debug hơn rất nhiều trong giai đoạn đầu. SSE thêm sau chỉ mất ~4 tiếng.

---

## Tổng hợp quyết định khuyến nghị

| # | Câu hỏi | Khuyến nghị | Action cần làm |
|---|---|---|---|
| 1 | Gemini API key | Lấy free key từ Google AI Studio | Truy cập link, tạo key, thêm `.env` |
| 2 | PGVector | Đã có sẵn trong Docker image, chỉ cần `CREATE EXTENSION` | Chạy 1 lệnh SQL |
| 3 | Sửa core-service | ✅ An toàn, 3 files mới + 1 dòng inject | Approve scope |
| 4 | Chat memory | In-Memory (Phase 1) → Persist (Phase 2) | Không cần action ngay |
| 5 | Streaming | Batch (Phase 1) → SSE (Phase 2) | Không cần action ngay |

Khi bạn xác nhận, tôi sẽ bắt đầu **Phase 1: Scaffold & Rename** ngay lập tức. 🚀
