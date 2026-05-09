# Part 2 — Nhóm 1: RAG Nâng Cao

## Feature 1: Simple RAG (Base — prerequisite cho tất cả)

### Mô tả kỹ thuật
Simple RAG là nền tảng: User hỏi → Embed query → Tìm k documents gần nhất trong PGVector → Inject vào prompt → LLM trả lời.

### Files cần tạo mới

#### 1. `config/LangChainConfig.java` — Bean configuration
```java
package com.flashticket.discovery.config;

import dev.langchain4j.data.segment.TextSegment;
import dev.langchain4j.model.embedding.EmbeddingModel;
import dev.langchain4j.model.googleai.GoogleAiEmbeddingModel;
import dev.langchain4j.store.embedding.EmbeddingStore;
import dev.langchain4j.store.embedding.pgvector.PgVectorEmbeddingStore;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class LangChainConfig {

    @Bean
    EmbeddingModel embeddingModel(
            @Value("${langchain4j.google-ai-gemini.chat-model.api-key}") String apiKey) {
        return GoogleAiEmbeddingModel.builder()
                .apiKey(apiKey)
                .modelName("text-embedding-004")
                .build();
    }

    @Bean
    EmbeddingStore<TextSegment> embeddingStore(
            @Value("${spring.datasource.url}") String url,
            @Value("${spring.datasource.username}") String user,
            @Value("${spring.datasource.password}") String password) {
        // Parse JDBC URL → host, port, database
        return PgVectorEmbeddingStore.builder()
                .host(extractHost(url))
                .port(extractPort(url))
                .database(extractDatabase(url))
                .user(user)
                .password(password)
                .table("discovery_schema.event_embeddings")
                .dimension(768) // text-embedding-004 = 768 dims
                .createTable(true)
                .build();
    }
    // extractHost/Port/Database helpers omitted for brevity
}
```

#### 2. `rag/retriever/EventContentRetriever.java`
```java
package com.flashticket.discovery.rag.retriever;

import dev.langchain4j.data.segment.TextSegment;
import dev.langchain4j.model.embedding.EmbeddingModel;
import dev.langchain4j.rag.content.Content;
import dev.langchain4j.rag.content.retriever.ContentRetriever;
import dev.langchain4j.rag.query.Query;
import dev.langchain4j.store.embedding.EmbeddingSearchRequest;
import dev.langchain4j.store.embedding.EmbeddingStore;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

import java.util.List;

/**
 * Simple RAG retriever — tìm events gần nhất trong PGVector.
 * maxResults = 5, minScore = 0.7 (chỉ trả kết quả đủ relevant).
 */
@Component
@RequiredArgsConstructor
public class EventContentRetriever implements ContentRetriever {

    private final EmbeddingStore<TextSegment> embeddingStore;
    private final EmbeddingModel embeddingModel;

    private static final int MAX_RESULTS = 5;
    private static final double MIN_SCORE = 0.7;

    @Override
    public List<Content> retrieve(Query query) {
        var embedding = embeddingModel.embed(query.text()).content();
        var searchRequest = EmbeddingSearchRequest.builder()
                .queryEmbedding(embedding)
                .maxResults(MAX_RESULTS)
                .minScore(MIN_SCORE)
                .build();
        return embeddingStore.search(searchRequest).matches().stream()
                .map(match -> Content.from(match.embedded()))
                .toList();
    }
}
```

#### 3. `ingest/service/EmbeddingIngestionService.java`
```java
package com.flashticket.discovery.ingest.service;

import dev.langchain4j.data.document.Document;
import dev.langchain4j.data.document.Metadata;
import dev.langchain4j.data.document.splitter.DocumentSplitters;
import dev.langchain4j.data.segment.TextSegment;
import dev.langchain4j.model.embedding.EmbeddingModel;
import dev.langchain4j.store.embedding.EmbeddingStore;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

/**
 * Chuyển Event data thành embeddings và lưu vào PGVector.
 *
 * Document format:
 * "[Title] - [Description] | Thể loại: [categories] | Giá từ: [minPrice]
 *  | Ngày: [startDate] | Địa điểm: [venue] | Tags: [tags]"
 *
 * Metadata: eventId, organizerId, status, minPrice — dùng cho filtering.
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class EmbeddingIngestionService {

    private final EmbeddingStore<TextSegment> embeddingStore;
    private final EmbeddingModel embeddingModel;

    public void upsertEvent(EventSyncMessage event) {
        // 1. Xóa embeddings cũ của event này (by metadata filter)
        embeddingStore.removeAll(
            dev.langchain4j.store.embedding.filter.MetadataFilterBuilder
                .metadataKey("eventId").isEqualTo(event.eventId().toString()));

        // 2. Build document text
        String text = buildEventText(event);

        // 3. Split thành segments (chunk 500 chars, overlap 50)
        Document doc = Document.from(text, Metadata.from("eventId", event.eventId().toString())
                .put("organizerId", event.organizerId())
                .put("status", event.status())
                .put("minPrice", String.valueOf(event.minPrice())));

        var segments = DocumentSplitters
                .recursive(500, 50)
                .split(doc);

        // 4. Embed & store
        var embeddings = embeddingModel.embedAll(segments).content();
        embeddingStore.addAll(embeddings, segments);

        log.info("[Ingest] Upserted {} segments for event: {}", segments.size(), event.title());
    }

    public void deleteEvent(String eventId) {
        embeddingStore.removeAll(
            dev.langchain4j.store.embedding.filter.MetadataFilterBuilder
                .metadataKey("eventId").isEqualTo(eventId));
        log.info("[Ingest] Deleted embeddings for event: {}", eventId);
    }

    private String buildEventText(EventSyncMessage e) {
        return String.format("""
            %s - %s
            Thể loại: %s | Giá từ: %s VND
            Ngày diễn ra: %s | Địa điểm: %s
            Tags: %s | Tổ chức bởi: %s
            """,
            e.title(), e.description(),
            String.join(", ", e.categories()), e.minPrice(),
            e.startDatetime(), e.venueName(),
            String.join(", ", e.tags()), e.organizerName());
    }

    /** DTO nhận từ RabbitMQ */
    public record EventSyncMessage(
        java.util.UUID eventId, String title, String description,
        String shortDescription, java.util.List<String> categories,
        java.util.List<String> tags, String organizerId, String organizerName,
        String status, java.math.BigDecimal minPrice,
        String startDatetime, String endDatetime, String venueName,
        String venueAddress
    ) {}
}
```

#### 4. `ingest/listener/EventDataListener.java`
```java
package com.flashticket.discovery.ingest.listener;

import com.flashticket.discovery.ingest.service.EmbeddingIngestionService;
import com.flashticket.discovery.shared.messaging.DiscoveryMQConstants;
import com.rabbitmq.client.Channel;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.amqp.support.AmqpHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Component;

/**
 * Consumer RabbitMQ — nhận event data từ core-service.
 * Manual ACK pattern (nhất quán với core-service).
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class EventDataListener {

    private final EmbeddingIngestionService ingestionService;

    @RabbitListener(queues = DiscoveryMQConstants.QUEUE_EVENT_SYNC)
    public void handleEventSync(
            EmbeddingIngestionService.EventSyncMessage message,
            Channel channel,
            @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) {
        try {
            if ("DELETED".equals(message.status())) {
                ingestionService.deleteEvent(message.eventId().toString());
            } else {
                ingestionService.upsertEvent(message);
            }
            channel.basicAck(deliveryTag, false);
        } catch (Exception e) {
            log.error("[EventDataListener] Failed to process: {}", message.eventId(), e);
            try { channel.basicNack(deliveryTag, false, false); } // → DLQ
            catch (Exception ignored) {}
        }
    }
}
```

---

## Feature 2: Multi-hop RAG

### Mô tả kỹ thuật
Xử lý câu hỏi phức tạp cần nhiều bước tra cứu. Ví dụ: *"Có sự kiện nhạc rock nào ở Hà Nội vào tháng 6 mà giá dưới 500k không? Nếu có thì organizer nào tổ chức?"*

**Cơ chế**: Decompose câu hỏi → nhiều sub-queries → tìm kiếm song song → merge kết quả → LLM tổng hợp.

### File: `rag/strategy/MultiHopRagStrategy.java`
```java
package com.flashticket.discovery.rag.strategy;

import dev.langchain4j.model.chat.ChatLanguageModel;
import dev.langchain4j.model.input.PromptTemplate;
import dev.langchain4j.rag.content.Content;
import com.flashticket.discovery.rag.retriever.EventContentRetriever;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.*;

/**
 * Multi-hop RAG: Phân rã câu hỏi phức tạp → nhiều sub-queries → merge.
 *
 * Flow:
 * 1. LLM decompose: "Tìm concert rock ở HN giá rẻ" → ["concert rock Hà Nội", "sự kiện giá dưới 500k"]
 * 2. Mỗi sub-query → EventContentRetriever → List<Content>
 * 3. Deduplicate by eventId
 * 4. Trả về merged context cho LLM final answer
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class MultiHopRagStrategy implements RagStrategy {

    private final ChatLanguageModel chatModel;
    private final EventContentRetriever retriever;

    private static final String DECOMPOSE_PROMPT = """
        Phân tích câu hỏi sau thành tối đa 3 sub-queries đơn giản để tìm kiếm sự kiện.
        Mỗi sub-query trên 1 dòng riêng. Không đánh số. Không giải thích.
        Câu hỏi: {{query}}
        """;

    @Override
    public List<Content> retrieve(String query) {
        // Step 1: Decompose
        String decomposed = chatModel.generate(
            PromptTemplate.from(DECOMPOSE_PROMPT)
                .apply(Map.of("query", query))
                .toUserMessage()
        ).content().text();

        List<String> subQueries = Arrays.stream(decomposed.split("\n"))
                .map(String::trim)
                .filter(s -> !s.isBlank())
                .limit(3)
                .toList();

        log.info("[MultiHop] Decomposed into {} sub-queries: {}", subQueries.size(), subQueries);

        // Step 2: Retrieve for each sub-query
        Set<String> seenEventIds = new HashSet<>();
        List<Content> mergedResults = new ArrayList<>();

        for (String subQuery : subQueries) {
            var results = retriever.retrieve(
                dev.langchain4j.rag.query.Query.from(subQuery));
            for (Content content : results) {
                String eventId = content.textSegment().metadata().getString("eventId");
                if (eventId != null && seenEventIds.add(eventId)) {
                    mergedResults.add(content);
                }
            }
        }

        log.info("[MultiHop] Total unique results: {}", mergedResults.size());
        return mergedResults;
    }

    @Override
    public String strategyName() { return "MULTI_HOP"; }
}
```

---

## Feature 3: CRAG (Corrective RAG)

### Mô tả kỹ thuật
Sau khi RAG trả kết quả, thêm bước **đánh giá chất lượng**. Nếu kết quả không đủ tốt → tự động sửa query và retry.

**Flow**: Query → Retrieve → Evaluate relevance → nếu score < threshold → rewrite query → retry (max 2 lần).

### File: `rag/evaluator/RelevanceEvaluator.java`
```java
package com.flashticket.discovery.rag.evaluator;

import dev.langchain4j.model.chat.ChatLanguageModel;
import dev.langchain4j.model.input.PromptTemplate;
import dev.langchain4j.rag.content.Content;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Map;

/**
 * Đánh giá mức độ liên quan của retrieved documents với câu hỏi.
 * Trả về score 0.0–1.0. Dùng LLM để judge.
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class RelevanceEvaluator {

    private final ChatLanguageModel chatModel;

    private static final String EVAL_PROMPT = """
        Đánh giá mức độ liên quan giữa CÂU HỎI và KẾT QUẢ TÌM KIẾM.
        Trả về CHỈ MỘT số từ 0.0 đến 1.0 (0=không liên quan, 1=hoàn toàn phù hợp).
        Không giải thích.

        CÂU HỎI: {{query}}
        KẾT QUẢ: {{context}}
        """;

    public double evaluate(String query, List<Content> contents) {
        if (contents.isEmpty()) return 0.0;

        String context = contents.stream()
                .map(c -> c.textSegment().text())
                .reduce("", (a, b) -> a + "\n---\n" + b);

        try {
            String response = chatModel.generate(
                PromptTemplate.from(EVAL_PROMPT)
                    .apply(Map.of("query", query, "context", context))
                    .toUserMessage()
            ).content().text().trim();
            return Double.parseDouble(response);
        } catch (Exception e) {
            log.warn("[CRAG] Evaluation failed, defaulting to 0.5: {}", e.getMessage());
            return 0.5;
        }
    }
}
```

### File: `rag/strategy/CorrectiveRagStrategy.java`
```java
package com.flashticket.discovery.rag.strategy;

import dev.langchain4j.model.chat.ChatLanguageModel;
import dev.langchain4j.model.input.PromptTemplate;
import dev.langchain4j.rag.content.Content;
import com.flashticket.discovery.rag.evaluator.RelevanceEvaluator;
import com.flashticket.discovery.rag.retriever.EventContentRetriever;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Map;

/**
 * CRAG: Retrieve → Evaluate → nếu score thấp → Rewrite query → Retry.
 * Max 2 lần retry. Threshold: 0.6.
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class CorrectiveRagStrategy implements RagStrategy {

    private final EventContentRetriever retriever;
    private final RelevanceEvaluator evaluator;
    private final ChatLanguageModel chatModel;

    private static final double RELEVANCE_THRESHOLD = 0.6;
    private static final int MAX_RETRIES = 2;

    private static final String REWRITE_PROMPT = """
        Câu hỏi sau không tìm được kết quả tốt. Viết lại câu hỏi bằng cách:
        - Dùng từ đồng nghĩa
        - Mở rộng/thu hẹp phạm vi
        - Thêm context về sự kiện/vé

        Câu hỏi gốc: {{query}}
        Chỉ trả lại câu hỏi mới, không giải thích.
        """;

    @Override
    public List<Content> retrieve(String query) {
        String currentQuery = query;

        for (int attempt = 0; attempt <= MAX_RETRIES; attempt++) {
            var results = retriever.retrieve(
                dev.langchain4j.rag.query.Query.from(currentQuery));
            double score = evaluator.evaluate(query, results); // luôn eval vs original query

            log.info("[CRAG] Attempt {}: query='{}', score={}, results={}",
                    attempt, currentQuery, score, results.size());

            if (score >= RELEVANCE_THRESHOLD || attempt == MAX_RETRIES) {
                return results;
            }

            // Rewrite query
            currentQuery = chatModel.generate(
                PromptTemplate.from(REWRITE_PROMPT)
                    .apply(Map.of("query", currentQuery))
                    .toUserMessage()
            ).content().text().trim();

            log.info("[CRAG] Rewritten query: '{}'", currentQuery);
        }
        return List.of(); // unreachable
    }

    @Override
    public String strategyName() { return "CRAG"; }
}
```

---

## Feature 4: Adaptive RAG Router

### Mô tả kỹ thuật
Router phân tích câu hỏi và tự chọn strategy phù hợp:
- **DIRECT**: Câu hỏi chung không cần RAG ("Xin chào", "Cảm ơn")
- **SIMPLE_RAG**: Câu hỏi đơn giản 1 ý ("Có sự kiện nhạc nào không?")
- **MULTI_HOP**: Câu hỏi phức tạp nhiều điều kiện
- **CRAG**: Câu hỏi mơ hồ cần tự sửa

### File: `rag/strategy/RagStrategy.java` (Interface)
```java
package com.flashticket.discovery.rag.strategy;

import dev.langchain4j.rag.content.Content;
import java.util.List;

public interface RagStrategy {
    List<Content> retrieve(String query);
    String strategyName();
}
```

### File: `rag/strategy/SimpleRagStrategy.java`
```java
package com.flashticket.discovery.rag.strategy;

import dev.langchain4j.rag.content.Content;
import com.flashticket.discovery.rag.retriever.EventContentRetriever;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;
import java.util.List;

@Component
@RequiredArgsConstructor
public class SimpleRagStrategy implements RagStrategy {
    private final EventContentRetriever retriever;

    @Override
    public List<Content> retrieve(String query) {
        return retriever.retrieve(dev.langchain4j.rag.query.Query.from(query));
    }

    @Override
    public String strategyName() { return "SIMPLE"; }
}
```

### File: `rag/strategy/AdaptiveRagRouter.java`
```java
package com.flashticket.discovery.rag.strategy;

import dev.langchain4j.model.chat.ChatLanguageModel;
import dev.langchain4j.model.input.PromptTemplate;
import dev.langchain4j.rag.content.Content;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Map;

/**
 * Adaptive Router — LLM phân loại câu hỏi rồi chọn strategy.
 *
 * Classification prompt trả về 1 trong 4: DIRECT, SIMPLE, MULTI_HOP, CRAG.
 * Fallback: SIMPLE nếu parse fail.
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class AdaptiveRagRouter {

    private final ChatLanguageModel chatModel;
    private final SimpleRagStrategy simpleStrategy;
    private final MultiHopRagStrategy multiHopStrategy;
    private final CorrectiveRagStrategy correctiveStrategy;

    private static final String CLASSIFY_PROMPT = """
        Phân loại câu hỏi về sự kiện/vé sau vào ĐÚNG 1 category:

        DIRECT — Chào hỏi, cảm ơn, không liên quan sự kiện. VD: "Xin chào", "Bạn là ai?"
        SIMPLE — Câu hỏi đơn giản 1 điều kiện. VD: "Có concert nào vào tuần này?"
        MULTI_HOP — Nhiều điều kiện, cần tra cứu đa chiều. VD: "Concert rock ở HN giá dưới 500k tháng 6?"
        CRAG — Câu hỏi mơ hồ, thiếu context. VD: "Có gì vui không?", "Muốn đi chơi"

        Câu hỏi: {{query}}
        Trả lời CHỈ 1 từ: DIRECT hoặc SIMPLE hoặc MULTI_HOP hoặc CRAG
        """;

    public RagResult route(String query) {
        String category = classify(query);
        log.info("[AdaptiveRAG] Query classified as: {} — '{}'", category, query);

        return switch (category) {
            case "DIRECT" -> new RagResult("DIRECT", List.of());
            case "MULTI_HOP" -> new RagResult("MULTI_HOP", multiHopStrategy.retrieve(query));
            case "CRAG" -> new RagResult("CRAG", correctiveStrategy.retrieve(query));
            default -> new RagResult("SIMPLE", simpleStrategy.retrieve(query));
        };
    }

    private String classify(String query) {
        try {
            String result = chatModel.generate(
                PromptTemplate.from(CLASSIFY_PROMPT)
                    .apply(Map.of("query", query))
                    .toUserMessage()
            ).content().text().trim().toUpperCase();
            if (List.of("DIRECT","SIMPLE","MULTI_HOP","CRAG").contains(result)) return result;
            return "SIMPLE";
        } catch (Exception e) {
            log.warn("[AdaptiveRAG] Classification failed, defaulting SIMPLE", e);
            return "SIMPLE";
        }
    }

    public record RagResult(String strategy, List<Content> contents) {}
}
```

---

## Checklist nhóm 1 — Trước khi chạy

- [ ] PGVector extension enabled: `CREATE EXTENSION IF NOT EXISTS vector;`
- [ ] `discovery_schema` created trong PostgreSQL
- [ ] RabbitMQ exchange `ex.discovery` + queue `q.discovery.event.sync` đã declared
- [ ] Gemini API key valid (test: `curl` trực tiếp Gemini API)
- [ ] Config server `discovery-service.yml` load thành công
- [ ] Embedding dimension = 768 (khớp với `text-embedding-004`)
- [ ] core-service publish test event qua RabbitMQ → consumer nhận được

## Test Cases — Nhóm 1

### Unit Tests
```java
// 1. EmbeddingIngestionService — verify document format
@Test void shouldBuildCorrectDocumentText() {
    var msg = new EventSyncMessage(UUID.randomUUID(), "Rock Concert", "Desc", ...);
    // Assert text contains title, categories, price
}

// 2. AdaptiveRagRouter — classification
@Test void shouldClassifyGreetingAsDirect() { /* "Xin chào" → DIRECT */ }
@Test void shouldClassifySimpleQuestion() { /* "Concert nào?" → SIMPLE */ }
@Test void shouldClassifyComplexAsMultiHop() { /* "Rock HN < 500k tháng 6" → MULTI_HOP */ }

// 3. RelevanceEvaluator — score parsing
@Test void shouldReturnDefaultOnParseError() { /* Non-numeric → 0.5 */ }
```

### Integration Tests
```java
// 1. Ingest → Retrieve roundtrip
@Test void shouldIngestAndRetrieveEvent() {
    ingestionService.upsertEvent(testEvent);
    var results = retriever.retrieve(Query.from("concert nhạc rock"));
    assertThat(results).isNotEmpty();
    assertThat(results.get(0).textSegment().metadata().getString("eventId"))
        .isEqualTo(testEvent.eventId().toString());
}

// 2. CRAG retry
@Test void shouldRewriteQueryOnLowRelevance() {
    // Mock evaluator trả score 0.3 lần 1, 0.8 lần 2
    // Verify retriever.retrieve() gọi 2 lần
}
```

## Potential Errors — Nhóm 1

| Error | Dấu hiệu | Fix |
|---|---|---|
| `PgVectorEmbeddingStore` NPE | `NullPointerException` at store initialization | Verify JDBC URL parse đúng host/port/db |
| Embedding dimension mismatch | `ERROR: expected 768 dimensions, got 1536` | Đảm bảo dùng `text-embedding-004` (768), không phải OpenAI ada (1536) |
| RabbitMQ deserialization fail | `MessageConversionException` | Đảm bảo `EventSyncMessage` record có constructor match JSON |
| LLM rate limit | `429 Too Many Requests` | Thêm exponential backoff + fallback sang OpenAI |
| Slow classification | Response > 3s cho classify prompt | Dùng `gemini-2.0-flash` (nhanh), giảm prompt length |

---

> **Next**: Part 3 — Nhóm 2: Agentic (Booking Tool, Payment Tool, Full ReAct Loop)
