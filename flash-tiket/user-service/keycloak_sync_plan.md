# Chiến Lược Đồng Bộ Dữ Liệu Keycloak & User Service

Mục tiêu: Đảm bảo khi một user mới đăng ký trên Keycloak (hoặc cập nhật thông tin email/password), User Service (MongoDB) sẽ ngay lập tức được cập nhật mà không phải chờ rà soát thủ công, đồng thời đảm bảo tính nhất quán dữ liệu (Data Consistency).

## 1. Vấn đề hiện tại / Rủi ro xung đột dữ liệu
Như trong lịch sử hệ thống từng có lỗi giao email xác nhận sai vì **Email gửi đi lấy từ MongoDB bị sai lệch so với Keycloak**. Điều này xảy ra do 2 bên không đồng bộ khi user thay đổi email.
**Nguyên tắc cốt lõi (Single Source of Truth - SSOT):**
- **Keycloak** là Nguồn chân lý cho dữ liệu xác thực (Authentication): `id` (subject), `email`, `password`, `is_email_verified`, `roles`.
- **User Service (MongoDB)** là Nguồn chân lý cho hồ sơ người dùng (Profile): `first_name`, `last_name`, `phone`, `avatar`, `banking_info`, v.v.

## 2. Lựa chọn chiến lược đồng bộ

Có 3 cách để User Service biết khi một User đăng ký mới trên Keycloak. Khuyến nghị sử dụng **Cách 1** hoặc **Cách 2** cho hệ thống Production.

### Cách 1: Keycloak Event Listener SPI + Message Broker (RabbitMQ/Kafka) -> Khuyên dùng 🌟
Đây là giải pháp chuyên nghiệp, mạnh mẽ và chịu lỗi (fault-tolerant) tốt nhất.
1. **Viết/Cài đặt Keycloak Custom Event Listener SPI** (một file .jar drop vào Keycloak).
2. Khi user thực hiện hành động (REGISTER, UPDATE_EMAIL, DELETE_ACCOUNT), SPI này sẽ bắt sự kiện và publish một message vào RabbitMQ (hoặc Kafka). 
   - *Message ví dụ:* `{"eventType": "REGISTER", "userId": "123-uuid", "email": "test@gmail.com"}`
3. **User Service** có một `@RabbitListener`ắng nghe queue này. Khi nhận được message, nó sẽ tạo một document mới trong MongoDB với `_id` khớp chính xác với `userId` của Keycloak.
**Ưu điểm:** Độc lập hoàn toàn (Decoupled), nếu User Service chết tạm thời, RabbitMQ vẫn giữ message, lúc sống lại sẽ xử lý bù -> Không bao giờ mất tính đồng bộ.

### Cách 2: Intercept tại API Gateway / Tự động đồng bộ khi Login (Lazy Sync)
Phù hợp nếu bạn không muốn can thiệp vào Keycloak SPI (không muốn viết code Java deploy vào Keycloak).
1. Người dùng bấm Đăng ký trên Keycloak, Keycloak tạo user.
2. Lúc này User Service chưa biết gì. 
3. Sau khi Đăng ký xong, Keycloak tự chuyển hướng (redirect) client cùng mã JWT để đăng nhập.
4. Client gọi bất kỳ API nào (vd: `GET /api/users/me`) với JWT (chứa `sub`, `email`, `name`).
5. **User Service** kiểm tra: Nếu `sub` này chưa có trong MongoDB -> Tự động Insert mới ngay lúc đó. Nếu có rồi nhưng email trong JWT lúc này khác email trong MongoDB -> Update MongoDB.
**Ưu điểm:** Cực kỳ dễ làm, nhàn rỗi. 
**Nhược điểm:** User đăng ký xong mà chưa bao giờ login (hoặc chưa bao giờ gọi API đi qua User Service) thì trong Database sẽ không tồn tại account của họ (chỉ có trên Keycloak).

### Cách 3: Keycloak Webhook
Sử dụng extension của Keycloak (như `keycloak-webhook`) để bắn HTTP POST request trực tiếp đến một API nội bộ của User Service (vd: `POST /api/internal/users/sync`).
**Nhược điểm:** Nếu HTTP POST thất bại do mạng chập chờn hoặc User Service bảo trì lúc đó, dữ liệu người dùng đó sẽ vĩnh viễn không được tạo bên User Service (trừ khi có cơ chế retry phức tạp).

## 3. Kiến trúc Đề Xuất cho Flash Ticket

> [!TIP]
> **Đề xuất sử dụng Cách 1 (RabbitMQ SPI) kết hợp với Cơ chế Tự sửa lỗi (Self-Healing) của Cách 2.**
> Hệ thống Flash Ticket đã có sẵn kiến trúc RabbitMQ (như thấy trong lịch sử hệ thống pg_backend_service/flash-ticket-system). Việc sử dụng Message Queue là tối ưu nhất.

**Luồng chạy chuẩn:**
1. User đăng ký tại `Keycloak`.
2. Keycloak gởi event `REGISTER` vào RabbitMQ Exchange `keycloak_events`.
3. User Service (Consumer) nhận event -> Insert vào MongoDB (`id`, `email`, `role`).

**Cơ chế Self-healing dự phòng:**
Mỗi khi một request đi qua API Gateway vào User Service (hoặc middleware của Core Service), bạn trích xuất JWT Token. Nếu phát hiện `email` trong JWT khác với CSDL hoặc User chưa tồn tại trong CSDL do thỏ (RabbitMQ) lỗi, thực hiện chèn/cập nhật ngay lập tức.

## User Review Required
Bạn có muốn tôi hướng dẫn chi tiết cách cấu hình **Keycloak Event Listener SPI** để bắn sự kiện ra RabbitMQ, hay bạn muốn dùng cấu trúc **Lazy Sync** để dễ triển khai hơn trong đồ án lúc này?
