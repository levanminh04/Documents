# Plan: Environment & Database Preparation for Testing

Để chuẩn bị môi trường sẵn sàng cho việc chạy và kiểm thử (run & test) sự tích hợp giữa `core-service` và `discovery-service`, chúng ta cần thực hiện cấu hình Database, biến môi trường, và trình tự khởi động.

## 1. Chuẩn bị PostgreSQL Database

Vì `discovery-service` sử dụng `PGVector` để lưu trữ Embeddings và có schema riêng, bạn cần chạy script SQL sau vào PostgreSQL (có thể chạy qua pgAdmin hoặc DBeaver, kết nối vào database hiện tại của hệ thống):

```sql
-- 1. Cài đặt extension pgvector (BẮT BUỘC để dùng Semantic Search)
CREATE EXTENSION IF NOT EXISTS vector;

-- 2. Tạo schema riêng cho discovery-service
CREATE SCHEMA IF NOT EXISTS discovery_schema;

-- (Bảng discovery_schema.event_embeddings sẽ được Hibernate tự động tạo 
-- do ddl-auto thường set là update, hoặc Langchain4j tự tạo qua builder).
```

> [!NOTE]
> Trong `core-service` (file `core-service.yml`), Hibernate đang được set là `ddl-auto: validate`. Nếu database của bạn chưa có các cột `min_price` và `banner_url` trong bảng `event_schema.events`, khi chạy `core-service` có thể báo lỗi. Nếu báo lỗi, bạn cần chạy thêm SQL:
> ```sql
> ALTER TABLE event_schema.events ADD COLUMN min_price numeric(15,2);
> ALTER TABLE event_schema.events ADD COLUMN banner_url varchar(500);
> ```

## 2. Cấu hình biến môi trường (.env)

Trong thư mục `discovery-service`, đảm bảo bạn đã tạo file `.env` chứa các giá trị thực tế. 

> [!IMPORTANT]
> **BẮT BUỘC** phải có `GEMINI_API_KEY` hợp lệ. Bạn có thể lấy từ Google AI Studio.

File `.env` mẫu (`discovery-service/.env`):
```env
# Server
SERVER_PORT=8085

# PostgreSQL
POSTGRES_URL=jdbc:postgresql://localhost:5432/flashticket_db
POSTGRES_USER=postgres
POSTGRES_PASSWORD=your_password

# RabbitMQ
RABBITMQ_HOST=localhost
RABBITMQ_PORT=5672
RABBITMQ_USERNAME=guest
RABBITMQ_PASSWORD=guest

# Keycloak (Dùng chung với core-service)
KEYCLOAK_ISSUER_URI=http://localhost:9090/realms/flash-ticket

# Gemini AI (TẠO KEY TỪ GOOGLE AI STUDIO)
GEMINI_API_KEY=AIzaSy...
```

## 3. Trình tự khởi động (Start-up Sequence)

Vì kiến trúc là Microservices, bạn cần khởi động các service theo đúng thứ tự để tránh lỗi mất kết nối:

1.  **Infrastructure:** Chạy Docker Compose cho PostgreSQL, RabbitMQ, Keycloak, Redis.
    *   `docker-compose up -d`
2.  **Eureka Server:** Chạy service Registry trước tiên.
3.  **Config Server:** Chạy Config Server để cung cấp cấu hình cho các service khác.
4.  **Core Service:** Chạy Core Service. Lúc này RabbitMQ queues và exchanges (`ex.discovery`, `q.discovery.event.sync`) sẽ được tự động tạo.
5.  **Discovery Service:** Chạy service mới này. Nó sẽ tự động kết nối PGVector và RabbitMQ.
6.  **API Gateway:** Chạy Gateway cuối cùng để định tuyến request.

## 4. Kiểm thử sau khi Run (Verification Plan)

Sau khi các service báo trạng thái `UP` trên Eureka:

### A. Kiểm thử Ingestion (Đồng bộ dữ liệu)
- **Tạo Event mới** (hoặc Publish một Event có sẵn) bằng API của `core-service`.
- Mở RabbitMQ Management (http://localhost:15672), kiểm tra xem có message nào đi vào `q.discovery.event.sync` không.
- Kiểm tra console log của `discovery-service`, bạn sẽ thấy log: `[Ingest] Upserted ... segments for event...`.
- Vào database, kiểm tra bảng `discovery_schema.event_embeddings` xem đã có data chưa.

### B. Kiểm thử Search & Chat API (qua API Gateway)
- Dùng Postman gọi: `GET http://localhost:8080/api/discovery/search?q=tìm sự kiện âm nhạc`
  -> Kì vọng: Trả về kết quả search từ PGVector.
- Dùng Postman gọi (kèm JWT Token lấy từ Keycloak):
  ```json
  POST http://localhost:8080/api/chat
  {
      "sessionId": "session-123",
      "message": "Có sự kiện nào về rock không? Đặt cho tôi 2 vé."
  }
  ```
  -> Kì vọng: AI sẽ tìm kiếm event và trả lời yêu cầu xác nhận.

---

## User Review Required
Bạn có gặp khó khăn gì trong việc thiết lập `GEMINI_API_KEY` hay cài đặt `pgvector` không? 
Bạn có muốn tôi viết sẵn một bash script (`start-all.sh`) để tự động khởi động toàn bộ hệ thống theo đúng trình tự không? Vui lòng cho biết ý kiến của bạn để chúng ta tiến hành!
