# Discovery Service — Full Implementation Walkthrough

## Build Status
| Service | Files | Status |
|---------|-------|--------|
| `discovery-service` | 31 source files | ✅ BUILD SUCCESS |
| `core-service` | (patched, 5 changes) | ✅ BUILD SUCCESS |

---

## Architecture Overview

```mermaid
graph TB
    subgraph "API Gateway (port 8080)"
        GW["/api/chat/** → DISCOVERY-SERVICE"]
        GW2["/api/discovery/** → DISCOVERY-SERVICE"]
    end
    
    subgraph "discovery-service (port 8085)"
        CC[ChatController<br/>POST /api/chat]
        DC[DiscoveryController<br/>GET /api/discovery/search]
        
        CS[ChatService]
        MD[MoodDetector]
        DA[DiscoveryAssistant<br/>AiServices + ReAct]
        
        AR[AdaptiveRagRouter<br/>LLM Classification]
        SR[SimpleRagStrategy]
        MH[MultiHopRagStrategy]
        CR[CorrectiveRagStrategy]
        RE[RelevanceEvaluator]
        ECR[EventContentRetriever<br/>PGVector Search]
        
        EST[EventSearchTool]
        BT[BookingTool]
        PT[PaymentTool]
        
        EDL[EventDataListener<br/>RabbitMQ Consumer]
        EIS[EmbeddingIngestionService]
    end
    
    subgraph "core-service"
        ESP[EventSyncPublisher<br/>@TransactionalEventListener]
    end
    
    subgraph "Infrastructure"
        PG[(PostgreSQL<br/>PGVector)]
        RMQ[RabbitMQ<br/>ex.discovery]
        GEM[Gemini 2.0 Flash]
    end
    
    GW --> CC
    GW2 --> DC
    CC --> CS --> MD --> GEM
    CS --> DA --> ECR --> PG
    DA --> EST & BT & PT
    EST & BT & PT -.->|JWT forwarded| CS
    DC --> AR --> SR & MH & CR
    SR & MH & CR --> ECR
    CR --> RE --> GEM
    
    ESP -->|AFTER_COMMIT| RMQ
    RMQ --> EDL --> EIS --> PG
```

---

## File Inventory (31 files)

### Phase 1: Scaffold & Rename
| File | Description |
|------|-------------|
| [DiscoveryServiceApplication.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/DiscoveryServiceApplication.java) | Main class |
| [pom.xml](file:///d:/Project/flash-ticket-system/discovery-service/pom.xml) | LangChain4j 1.0.0-beta5, RabbitMQ, Security, JPA |
| [application.yaml](file:///d:/Project/flash-ticket-system/discovery-service/src/main/resources/application.yaml) | Spring config import |
| [.env](file:///d:/Project/flash-ticket-system/discovery-service/.env) | Environment variables |
| [SecurityConfig.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/config/SecurityConfig.java) | JWT + Keycloak role extraction |
| [RabbitMQConfig.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/config/RabbitMQConfig.java) | Exchange/Queue/Binding topology |
| [LangChainConfig.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/config/LangChainConfig.java) | EmbeddingModel + EmbeddingStore + AiServices |
| [RestClientConfig.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/config/RestClientConfig.java) | Load-balanced RestClient |
| [AsyncConfig.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/config/AsyncConfig.java) | Thread pool for async ops |
| [DiscoveryMQConstants.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/shared/messaging/DiscoveryMQConstants.java) | RabbitMQ constants |
| [GlobalExceptionHandler.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/shared/exception/GlobalExceptionHandler.java) | REST error handling |

### Phase 2: Data Ingestion
| File | Description |
|------|-------------|
| [EmbeddingIngestionService.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/ingest/service/EmbeddingIngestionService.java) | Event → Document → Embedding → PGVector |
| [EventDataListener.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/ingest/listener/EventDataListener.java) | RabbitMQ consumer (Manual ACK) |

### Phase 3: Chat API + Simple RAG
| File | Description |
|------|-------------|
| [ChatRequest.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/chat/dto/ChatRequest.java) | Request DTO |
| [ChatResponse.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/chat/dto/ChatResponse.java) | Response DTO (includes mood + strategy) |
| [DiscoveryAssistant.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/chat/service/DiscoveryAssistant.java) | AiServices interface + System Prompt (Vietnamese) |
| [ChatService.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/chat/service/ChatService.java) | Orchestrator: Mood → JWT → Assistant → Response |
| [JwtContextHolder.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/chat/service/JwtContextHolder.java) | ThreadLocal JWT propagation |
| [ChatController.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/chat/controller/ChatController.java) | POST /api/chat (authenticated) |
| [EventContentRetriever.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/rag/retriever/EventContentRetriever.java) | PGVector semantic search (top-5, min 0.7) |
| [DiscoveryController.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/rag/controller/DiscoveryController.java) | GET /api/discovery/search (public) |

### Phase 4: Advanced RAG
| File | Description |
|------|-------------|
| [RagStrategy.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/rag/strategy/RagStrategy.java) | Strategy interface |
| [SimpleRagStrategy.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/rag/strategy/SimpleRagStrategy.java) | Direct vector search |
| [MultiHopRagStrategy.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/rag/strategy/MultiHopRagStrategy.java) | Decompose → Parallel retrieve → Merge |
| [CorrectiveRagStrategy.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/rag/strategy/CorrectiveRagStrategy.java) | Retrieve → Evaluate → Rewrite → Retry |
| [AdaptiveRagRouter.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/rag/strategy/AdaptiveRagRouter.java) | LLM classification → route to strategy |
| [RelevanceEvaluator.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/rag/evaluator/RelevanceEvaluator.java) | LLM-based relevance scoring (0.0–1.0) |

### Phase 5: Mood-Aware + Agentic Tools
| File | Description |
|------|-------------|
| [UserMood.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/mood/model/UserMood.java) | 5 mood states with tone instructions |
| [MoodDetector.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/mood/MoodDetector.java) | LLM classification (EXCITED/STRESSED/SAD/RELAXED/NEUTRAL) |
| [MoodAwarePromptEnhancer.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/mood/MoodAwarePromptEnhancer.java) | Tone + retrieval bias adjustment |
| [CoreServiceClient.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/shared/client/CoreServiceClient.java) | REST client → core-service (JWT forwarding) |
| [EventSearchTool.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/agent/tool/EventSearchTool.java) | @Tool: search + detail via REST |
| [BookingTool.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/agent/tool/BookingTool.java) | @Tool: create booking (JWT from ThreadLocal) |
| [PaymentTool.java](file:///d:/Project/flash-ticket-system/discovery-service/src/main/java/com/flashticket/discovery/agent/tool/PaymentTool.java) | @Tool: create VNPay link (JWT from ThreadLocal) |

### Core Service Changes
| File | Changes |
|------|---------|
| [RabbitMQConstants.java](file:///d:/Project/flash-ticket-system/core-service/src/main/java/com/flashticket/core/shared/messaging/RabbitMQConstants.java) | +4 discovery sync constants |
| [EventSyncSpringEvent.java](file:///d:/Project/flash-ticket-system/core-service/src/main/java/com/flashticket/core/shared/event/EventSyncSpringEvent.java) | NEW: Spring ApplicationEvent record |
| [EventSyncPublisher.java](file:///d:/Project/flash-ticket-system/core-service/src/main/java/com/flashticket/core/shared/event/EventSyncPublisher.java) | NEW: @TransactionalEventListener + @Async |
| [OrganizerEventService.java](file:///d:/Project/flash-ticket-system/core-service/src/main/java/com/flashticket/core/event/service/OrganizerEventService.java) | +1 field, +4 publish calls |
| [RabbitMQConfig.java](file:///d:/Project/flash-ticket-system/core-service/src/main/java/com/flashticket/core/config/messaging/RabbitMQConfig.java) | +5 discovery topology beans |

### Gateway + Config Server
| File | Changes |
|------|---------|
| [GatewayConfig.java](file:///d:/Project/flash-ticket-system/apigateway/src/main/java/com/flashticket/gateway/GatewayConfig.java) | Route → `lb://DISCOVERY-SERVICE`, paths `/api/discovery/**`, `/api/chat/**` |
| [FallBackController.java](file:///d:/Project/flash-ticket-system/apigateway/src/main/java/com/flashticket/gateway/FallBackController.java) | `/fallback/discovery` |
| [discovery-service.yml](file:///d:/Project/flash-ticket-system/configserver/src/main/resources/config/discovery-service.yml) | NEW: Full config |
| [gateway-service.yml](file:///d:/Project/flash-ticket-system/configserver/src/main/resources/config/gateway-service.yml) | `discoveryServiceBreaker` + 30s timeout |

---

## Before Running — Checklist

1. **PostgreSQL**: Run these SQL commands:
   ```sql
   CREATE EXTENSION IF NOT EXISTS vector;
   CREATE SCHEMA IF NOT EXISTS discovery_schema;
   ```
2. **Gemini API Key**: Set `GEMINI_API_KEY` in `discovery-service/.env`
3. **RabbitMQ**: Must be running (docker-compose up rabbitmq)
4. **Keycloak**: Must be running for JWT validation

## API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/api/discovery/search?q=...` | Public | Semantic event search (Adaptive RAG) |
| `GET` | `/api/discovery/health` | Public | Service health check |
| `POST` | `/api/chat` | JWT | AI chat with mood detection + tools |
