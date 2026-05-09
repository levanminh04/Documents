# Discovery Service — Implementation Plan

> **Scope**: 7 tính năng RAG + Agentic + Mood-Aware cho hệ thống FlashTicket
> **Chiến lược**: Chia thành 5 phần plan, triển khai tuần tự

## Mục lục các phần Plan

| Part | Nội dung | Status |
|------|----------|--------|
| **Part 1** | Foundation & Architecture (file này) | ✅ Current |
| **Part 2** | Nhóm 1 — RAG Nâng Cao (Multi-hop, CRAG, Adaptive) | ⬜ Next |
| **Part 3** | Nhóm 2 — Agentic (Booking, Payment, Full ReAct) | ⬜ |
| **Part 4** | Lớp xuyên suốt — Mood-Aware | ⬜ |
| **Part 5** | Thứ tự triển khai, Rủi ro, Verification | ⬜ |

---

## Part 1: Foundation & Architecture

### 1.1 Quyết định kiến trúc đã xác nhận

| Quyết định | Giá trị | Lý do |
|---|---|---|
| Service name | `discovery-service` (refactor từ `ai-service`) | User confirmed |
| LangChain4j version | **1.0.0-beta5** | Stable API, compatible Spring Boot 3.5.x, có `@Tool`, `AiServices`, `RetrievalAugmentor` |
| LLM Provider | Google Gemini (primary) + OpenAI (fallback) | Chi phí thấp, Gemini 2.0 Flash phù hợp |
| Vector Store | PGVector (PostgreSQL — đã có `pgvector/pgvector:pg14` trong docker-compose) | Không thêm infra mới |
| Messaging | **RabbitMQ** cho data sync (Event→PGVector) | Đồng nhất với hệ thống hiện tại, volume thấp |
| Auth | Forward JWT qua REST call tới core-service | Giữ IDOR protection |
| Port | `8085` (giữ nguyên từ ai-service config) | Không conflict |

### 1.2 Tại sao RabbitMQ thay vì Kafka cho Data Sync?

> [!IMPORTANT]
> **Volume analysis**: Organizer tạo/update event ~ vài chục lần/ngày. Đây là **low-throughput, high-reliability** use case — RabbitMQ phù hợp hơn Kafka.

| Tiêu chí | RabbitMQ | Kafka |
|---|---|---|
| Đã có trong hệ thống | ✅ Có | ⚠️ Có nhưng chưa dùng |
| Phù hợp volume | ✅ Low-medium | Overkill |
| Delivery guarantee | ✅ Manual ACK + DLQ | ✅ |
| Thêm infra complexity | ✅ Không | ❌ Zookeeper + Broker |
| Team familiarity | ✅ Đã dùng thành thạo | ❌ Chưa |

**Quyết định**: Dùng RabbitMQ. Xóa Kafka/Zookeeper dependencies khỏi discovery-service.

### 1.3 Refactor ai-service → discovery-service

#### Files cần thay đổi (existing codebase):

**1. Rename thư mục & Java package:**
```
ai-service/ → discovery-service/
com.flashticket.ai → com.flashticket.discovery
```

**2. [MODIFY] `discovery-service/pom.xml`:**
- `artifactId`: `ai-service` → `discovery-service`
- `name`: `discovery-service`
- Xóa Kafka dependencies (`spring-cloud-stream`, `spring-cloud-stream-binder-kafka`)
- Nâng `langchain4j.version` từ `0.35.0` → `1.0.0-beta5`
- Thêm dependencies mới (xem bên dưới)

**3. [MODIFY] `discovery-service/src/main/resources/application.yml`:**
```yaml
spring:
  application:
    name: discovery-service  # Thay đổi từ ai-service
  config:
    import: optional:configserver:http://localhost:8888
```

**4. [NEW] `configserver/src/main/resources/config/discovery-service.yml`:**
- Copy từ `ai-service.yml`, đổi tên, thêm config RabbitMQ + LangChain4j

**5. [MODIFY] `apigateway/.../GatewayConfig.java`:**
```java
// Thay đổi route ai-service → discovery-service
.route("discovery-service", r -> r
    .path("/api/discovery/**", "/api/chat/**")
    .filters(f -> f.circuitBreaker(config -> config
        .setName("discoveryServiceBreaker")
        .setFallbackUri("forward:/fallback/discovery")))
    .uri("lb://DISCOVERY-SERVICE"))
```

**6. [MODIFY] `configserver/.../gateway-service.yml`:**
- Rename `aiServiceBreaker` → `discoveryServiceBreaker` trong resilience4j config

**7. [MODIFY] `apigateway/.../FallBackController.java`:**
- Rename `/fallback/ai` → `/fallback/discovery`

**8. [DELETE] `configserver/src/main/resources/config/ai-service.yml`** (thay bằng discovery-service.yml)

### 1.4 Dependencies mới cho discovery-service (pom.xml)

```xml
<properties>
    <java.version>24</java.version>
    <spring-cloud.version>2025.0.0</spring-cloud.version>
    <langchain4j.version>1.0.0-beta5</langchain4j.version>
</properties>

<dependencies>
    <!-- Spring Boot Core -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- Security — JWT validation -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>

    <!-- Spring Cloud -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>

    <!-- RabbitMQ — Event data sync -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>

    <!-- PostgreSQL + PGVector -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- ═══ LangChain4j ═══ -->
    <dependency>
        <groupId>dev.langchain4j</groupId>
        <artifactId>langchain4j-spring-boot-starter</artifactId>
        <version>${langchain4j.version}</version>
    </dependency>
    <dependency>
        <groupId>dev.langchain4j</groupId>
        <artifactId>langchain4j-google-ai-gemini-spring-boot-starter</artifactId>
        <version>${langchain4j.version}</version>
    </dependency>
    <dependency>
        <groupId>dev.langchain4j</groupId>
        <artifactId>langchain4j-open-ai-spring-boot-starter</artifactId>
        <version>${langchain4j.version}</version>
    </dependency>
    <dependency>
        <groupId>dev.langchain4j</groupId>
        <artifactId>langchain4j-pgvector</artifactId>
        <version>${langchain4j.version}</version>
    </dependency>

    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

### 1.5 Config Server — discovery-service.yml

```yaml
server:
  port: ${SERVER_PORT:8085}

spring:
  datasource:
    url: ${POSTGRES_URL}
    username: ${POSTGRES_USER}
    password: ${POSTGRES_PASSWORD}
    driver-class-name: org.postgresql.Driver
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        default_schema: discovery_schema
        format_sql: true

  # RabbitMQ — Event data sync
  rabbitmq:
    host: ${RABBITMQ_HOST}
    port: ${RABBITMQ_PORT:5672}
    username: ${RABBITMQ_USERNAME}
    password: ${RABBITMQ_PASSWORD}
    listener:
      simple:
        default-requeue-rejected: false

  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${KEYCLOAK_ISSUER_URI}

# Eureka
eureka:
  client:
    serviceUrl:
      defaultZone: ${EUREKA_SERVER_URL:http://localhost:8761/eureka/}
    register-with-eureka: true
    fetch-registry: true
  instance:
    prefer-ip-address: true
    instance-id: ${spring.cloud.client.ip-address}:${spring.application.name}:${server.port}

# LangChain4j
langchain4j:
  google-ai-gemini:
    chat-model:
      api-key: ${GEMINI_API_KEY}
      model-name: ${GEMINI_MODEL:gemini-2.0-flash}
      temperature: 0.3
  open-ai:
    chat-model:
      api-key: ${OPENAI_API_KEY:}
      model-name: ${OPENAI_MODEL:gpt-4o-mini}

# Core Service base URL (for REST calls via Eureka)
app:
  core-service:
    base-url: http://CORE-SERVICE
  user-service:
    base-url: http://USER-SERVICE

management:
  endpoints:
    web:
      exposure:
        include: "*"

logging:
  level:
    com.flashticket.discovery: ${LOG_LEVEL:DEBUG}
    dev.langchain4j: ${LOG_LEVEL_AI:DEBUG}
```

### 1.6 Package Structure — discovery-service

```
discovery-service/src/main/java/com/flashticket/discovery/
├── DiscoveryServiceApplication.java
│
├── config/                          ← Spring beans configuration
│   ├── SecurityConfig.java          ← JWT validation (copy pattern từ core)
│   ├── RabbitMQConfig.java          ← Exchange, Queue, Binding cho data sync
│   ├── LangChainConfig.java         ← EmbeddingModel, EmbeddingStore, ChatMemory beans
│   ├── RestClientConfig.java        ← WebClient/RestClient cho S2S calls
│   └── AsyncConfig.java             ← ThreadPool cho async operations
│
├── rag/                             ← RAG engine (Nhóm 1)
│   ├── retriever/
│   │   ├── EventContentRetriever.java
│   │   └── MultiSourceRetriever.java
│   ├── strategy/
│   │   ├── RagStrategy.java         ← Interface
│   │   ├── SimpleRagStrategy.java
│   │   ├── MultiHopRagStrategy.java
│   │   ├── CorrectiveRagStrategy.java
│   │   └── AdaptiveRagRouter.java   ← Router chọn strategy
│   ├── evaluator/
│   │   └── RelevanceEvaluator.java  ← CRAG: đánh giá chất lượng
│   └── augmentor/
│       └── DiscoveryRetrievalAugmentor.java
│
├── agent/                           ← Agentic tools (Nhóm 2)
│   ├── tool/
│   │   ├── EventSearchTool.java     ← @Tool: search events
│   │   ├── BookingTool.java         ← @Tool: tạo booking qua REST
│   │   ├── PaymentTool.java         ← @Tool: tạo VNPay link
│   │   ├── VoucherTool.java         ← @Tool: validate/apply voucher
│   │   └── UserProfileTool.java     ← @Tool: get user preferences
│   └── service/
│       └── AgentOrchestrator.java   ← ReAct loop coordinator
│
├── mood/                            ← Mood-Aware layer (Lớp xuyên suốt)
│   ├── MoodDetector.java            ← Phát hiện mood từ message
│   ├── MoodAwarePromptEnhancer.java ← Điều chỉnh prompt theo mood
│   └── model/
│       └── UserMood.java            ← Enum: EXCITED/STRESSED/SAD/RELAXED/NEUTRAL
│
├── chat/                            ← Chat API layer
│   ├── controller/
│   │   └── ChatController.java      ← REST endpoints
│   ├── dto/
│   │   ├── ChatRequest.java
│   │   ├── ChatResponse.java
│   │   └── ConversationContext.java
│   ├── service/
│   │   ├── ChatService.java         ← Main entry point
│   │   └── DiscoveryAssistant.java  ← LangChain4j AiServices interface
│   └── memory/
│       └── PersistentChatMemory.java
│
├── ingest/                          ← Data ingestion pipeline
│   ├── listener/
│   │   └── EventDataListener.java   ← RabbitMQ consumer
│   ├── service/
│   │   └── EmbeddingIngestionService.java
│   └── transformer/
│       └── EventDocumentTransformer.java
│
├── shared/                          ← Shared utilities
│   ├── client/
│   │   ├── CoreServiceClient.java   ← REST client → core-service
│   │   └── UserServiceClient.java   ← REST client → user-service
│   ├── messaging/
│   │   └── DiscoveryMQConstants.java
│   └── exception/
│       ├── AiServiceException.java
│       └── GlobalExceptionHandler.java
│
└── model/                           ← JPA entities (discovery_schema)
    ├── ChatSession.java
    └── ChatMessage.java
```

### 1.7 Database Schema — discovery_schema

```sql
-- Flyway: V1__init_discovery_schema.sql
CREATE SCHEMA IF NOT EXISTS discovery_schema;

-- Chat session tracking
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

-- Chat message history (for persistent memory)
CREATE TABLE discovery_schema.chat_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES discovery_schema.chat_sessions(id),
    role VARCHAR(20) NOT NULL, -- USER, AI, SYSTEM, TOOL
    content TEXT NOT NULL,
    tool_name VARCHAR(100),
    tool_result TEXT,
    mood VARCHAR(20),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_chat_messages_session ON discovery_schema.chat_messages(session_id);

-- PGVector embedding table (managed by LangChain4j EmbeddingStore)
-- LangChain4j tự tạo table này khi khởi tạo PgVectorEmbeddingStore
-- Schema: discovery_schema.event_embeddings
```

### 1.8 RabbitMQ Topology cho Data Sync

```
core-service (Producer)                    discovery-service (Consumer)
┌─────────────────┐                       ┌──────────────────────┐
│ OrganizerEvent   │                       │ EventDataListener    │
│ Service          │                       │                      │
│                  │    ex.discovery       │ @RabbitListener      │
│ publishEvent(    │───►(Direct Exchange)──►│ (q.discovery.event)  │
│   EventSyncEvent │    RK: event.sync     │                      │
│ )                │                       │ → EmbeddingIngestion │
└─────────────────┘                       │   Service            │
                                          └──────────────────────┘

Queues:
  q.discovery.event.sync     ← Event CRUD sync (upsert embeddings)
  q.discovery.event.sync.dlq ← Dead letter queue
```

> [!WARNING]
> **Ràng buộc**: core-service chỉ thêm 1 dòng `eventPublisher.publishEvent(...)` sau khi Event được PUBLISHED/UPDATED. **KHÔNG thay đổi business logic**.

### 1.9 REST API Endpoints

```
POST /api/chat                    ← Gửi message, nhận AI response
GET  /api/chat/sessions           ← List chat sessions của user
GET  /api/chat/sessions/{id}      ← Lịch sử chat 1 session
POST /api/chat/sessions           ← Tạo session mới
DELETE /api/chat/sessions/{id}    ← Xóa session

POST /api/discovery/search        ← Semantic search events (non-chat)
GET  /api/discovery/health        ← AI health check (model connectivity)
```

### 1.10 Security — Forward JWT Pattern

```
User → Gateway (validate JWT) → discovery-service (extract userId)
                                        │
                                        ├── REST call → core-service
                                        │   Header: Authorization: Bearer {original-jwt}
                                        │   → core-service validates JWT again (Defense in Depth)
                                        │
                                        └── REST call → user-service
                                            Header: Authorization: Bearer {original-jwt}
```

> [!IMPORTANT]
> discovery-service **KHÔNG** dùng service account. Forward JWT gốc của user để core-service validate IDOR.

### 1.11 Checklist trước khi triển khai Part 1

- [ ] Rename `ai-service/` → `discovery-service/`
- [ ] Update `pom.xml` (dependencies, artifactId)
- [ ] Update `application.yml` (spring.application.name)
- [ ] Tạo `configserver/.../discovery-service.yml`
- [ ] Xóa `configserver/.../ai-service.yml`
- [ ] Update `GatewayConfig.java` routes
- [ ] Update `gateway-service.yml` circuit breaker
- [ ] Update `FallBackController.java`
- [ ] Tạo `DiscoveryServiceApplication.java`
- [ ] Tạo `SecurityConfig.java` (copy pattern từ core)
- [ ] Tạo `V1__init_discovery_schema.sql`
- [ ] Verify: `mvn clean compile` thành công
- [ ] Verify: service start + register vào Eureka
- [ ] Verify: `/actuator/health` trả OK

### 1.12 Potential Errors & Fix — Part 1

| Error | Stack trace dấu hiệu | Fix |
|---|---|---|
| Eureka registration fail | `Connection refused: localhost:8761` | Đảm bảo Eureka server đang chạy |
| Config server fail | `Could not resolve placeholder` | Kiểm tra `discovery-service.yml` đã tạo đúng path |
| PGVector extension missing | `type "vector" does not exist` | `CREATE EXTENSION IF NOT EXISTS vector;` trong Flyway migration |
| JWT validation fail | `401 Unauthorized` trên mọi request | Verify `issuer-uri` trong config trỏ đúng Keycloak |
| Schema not found | `relation "discovery_schema.chat_sessions" does not exist` | Chạy Flyway migration hoặc kiểm tra `default_schema` config |
| LangChain4j bean conflict | `Multiple beans of type ChatLanguageModel` | Dùng `@Qualifier` để phân biệt Gemini vs OpenAI model |

---

> **Next**: Part 2 sẽ chi tiết Nhóm 1 — RAG Nâng Cao (Multi-hop, CRAG, Adaptive RAG) với đầy đủ code skeleton, integration points, và test cases.
