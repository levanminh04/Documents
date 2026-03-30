# Rà Soát Cơ Sở Dữ Liệu & Thiết Kế Schema Cho User Service

Sau khi rà soát file `database/mongodb/user_service_schema.js` mà bạn đang có, tôi xác nhận rằng Schema hiện tại của bạn được thiết kế rất **chuyên nghiệp và bài bản**. Nó bao phủ các tính năng nâng cao (Multi-factor Auth, Preferences, Activity Tracking) cực kỳ tốt.

## 1. Đánh giá Schema hiện tại (user_service_schema.js) so với Plan

- **Thay vì gộp chung (Embedding)** thông tin Organizer vào trong `users` như đề xuất ban đầu của tôi, bạn đã tách thành 2 collection độc lập: `users` và `organizer_profiles`, sau đó liên kết bằng `organizerProfileId`. 
👉 **Đánh giá:** Đây là một cách tiếp cận **RẤT TỐT** (Referencing pattern). Điều này giúp document `users` nhẹ hơn khi query cho các tác vụ thông thường (login, check role), đồng thời profile của Organizer có thể phình to thoải mái (thống kê, danh sách giấy tờ, v.v.) mà không làm nặng BSON size của User Document.

- Schema của bạn đã có sẵn `preferences`, `user_activity_logs`, `businessInfo` (cho KYC). Nghĩa là CSDL của bạn **đã sẵn sàng 100% để phát triển các tính năng mở rộng** mà tôi vừa đề xuất ở Artifact khác!

## 2. Nguy cơ Xung Đột Dữ Liệu với Core Service (PostgreSQL)

Như đã đề cập trước đó, bảng `events` bên PostgreSQL (Core Service) đang lưu cứng `organizer_name` và `organizer_logo_url`.
**Giải Quyết:** Khi Tổ chức cập nhật `organizerName` hoặc `logoUrl` trong collection `organizer_profiles` trên MongoDB, User Service **bắt buộc phải bắn 1 Event (RabbitMQ)** để Core Service cập nhật lại tên/logo bên PostgreSQL. Nếu không, giao diện Event sẽ bị lỗi thời.

## 3. Bổ sung collection `user_follows`

Để hỗ trợ tính năng **Hệ Thống Theo Dõi (Follow & Fans System)** như bạn đã hỏi, chúng ta **CẦN BỔ SUNG** collection `user_follows` vào file `user_service_schema.js`. Bạn hãy thêm đoạn mã sau vào cuối file schema của bạn:

```javascript
// ============================================================================
// COLLECTION: user_follows
// ============================================================================
// Mô tả: Lưu trữ mối quan hệ Buyer theo dõi (Follow) Organizer
// Dùng để gửi Notification khi Organizer có sự kiện mới
// ============================================================================

db.createCollection("user_follows", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["followerId", "organizerProfileId", "createdAt"],
      properties: {
        followerId: {
          bsonType: "string",
          description: "ID của User (Buyer) đi follow"
        },
        organizerProfileId: {
          bsonType: "string",
          description: "ID của OrganizerProfile được follow"
        },
        createdAt: {
          bsonType: "date",
          description: "Thời điểm follow"
        }
      }
    }
  }
});

// Indexes cực kỳ quan trọng cho bảng này để truy vấn nhanh
// 1. Tìm tất cả organizer mà user đang follow
db.user_follows.createIndex({ "followerId": 1 });

// 2. Lấy ra danh sách TẤT CẢ follower của 1 organizer (để spam email khi có event mới)
db.user_follows.createIndex({ "organizerProfileId": 1 });

// 3. Đảm bảo 1 user chỉ follow 1 organizer 1 lần duy nhất (Unique Compound Index)
db.user_follows.createIndex(
  { "followerId": 1, "organizerProfileId": 1 }, 
  { unique: true, name: "idx_unique_user_follow" }
);
```

> [!TIP]
> **Tóm lại:** Cấu trúc hiện tại của bạn rất tốt (sẽ không bị xung đột). Bạn chỉ cần copy đoạn script tạo `user_follows` này dán vào dưới cùng file `user_service_schema.js` là CSDL đã hoàn thiện và đáp ứng được toàn bộ các tính năng xịn xò nhất!

## User Review Required
Bạn có muốn tôi giúp bạn cập nhật thẳng đoạn code này vào file `user_service_schema.js` luôn không? Hoặc nếu bạn muốn, chúng ta có thể chuyển sang bước Code Java (tạo Entity / Controller) cho User Service.
