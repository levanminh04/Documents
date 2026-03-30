# Kế Hoạch Các Tính Năng Mở Rộng Cho User Service

Hiện tại, User Service của bạn mới chỉ đóng vai trò cung cấp API CRUD (Tạo, Đọc, Cập nhật, Xóa) cho hồ sơ (Profile) của user. Khi thiết kế Microservices, mảng Quản trị Người dùng hoàn toàn có không gian để đóng gói hàng loạt các logic nghiệp vụ (Business Logic) liên quan mật thiết tới User. 

Với 3 roles `BUYER`, `ORGANIZER`, `ADMIN`, đây là những "vũ khí" bạn có thể phát triển thêm cho User Service để làm đồ án chuyên sâu và ấn tượng hơn:

## 1. Quy Trình Trở Thành Organizer (KYC & Onboarding Form)
Hệ thống bán vé nghiêm túc không thể cho phép ai cũng tự do tạo Event và bắt đầu thu tiền người dùng. Mọi hệ thống đều cần KYC (Know Your Customer).
- **Tính năng:** Buyer nộp hồ sơ xin trở thành Organizer (Upgrade Role Request).
- **Quy trình:** 
  1. Frontend gọi API: `POST /api/users/me/organizer-requests` gửi file Giấy phép đăng ký kinh doanh/CCCD và CMND, Mã số thuể.
  2. Bảng `Organizer_Requests` trong MongoDB lưu lại trạng thái `PENDING`.
  3. ADMIN sẽ có 1 màn hình Dashboard, gọi danh sách PENDING này, xem tài liệu, rồi bấm Duyệt hoặc Từ chối kèm theo lý do.
  4. Trả về thông báo Email và User Service gọi API Keycloak cấp phép thêm quyền (Role) `ORGANIZER` vào Token cho User này. Cập nhật `is_verified = true` (có dấu tích xanh).

## 2. Hệ Thống Theo Dõi (Follow & Fans System)
- **Tính năng:** Mối quan hệ giữa Buyer và Organizer.
- **Nghiệp vụ:** Buyer rất thích các sự kiện của công ty A (ví dụ như SpaceSpeaker). Họ có thể bấm nút "Follow Organizer".
- **Database (MongoDB):** Dùng một collection `user_follows` lưu `{ buyer_id, organizer_id, created_at }`. 
- **Tương tác liên Service:** Khi `core-service` xuất bản (Publish) một Event mới từ Organizer A, nó sẽ gõ cửa API User Service: "Đưa cho tao toàn bộ danh sách email các Buyer đang follow Organizer A này". Rồi gửi Email Notification hàng loạt: *"Ca sĩ bạn quan tâm vừa mở bán event mới nè!"*.

## 3. Quản Lý Sở Thích & Đề Xuất (User Preferences - Hỗ trợ cho AI)
Hệ thống Flash Ticket của bạn đã có `ai_schema` ở bên Core (`V2__complete_schema.sql`), chứng tỏ bạn đang tính đường cho recommendation (đề xuất).
- **Tính năng:** Hỏi người dùng về gu âm nhạc, thể loại Event yêu thích (Concert Rock, K-Pop, Thể thao, Hội thảo CNTT).
- **Nghiệp vụ:** 
  - Lưu Preferences vào MongoDB.  
  - API `GET /api/users/me/recommendation-tags` để Core-Service truy xuất và nạp vào pgvector để đề xuất các sự kiện phù hợp nhất ra màn hình Homepage theo dạng *Personalization (Cá nhân hóa)*.

## 4. Quản Lý Phiên Mạng, Hoạt Động & Bảo Mật (Audit Logs)
- **Tính năng:** Liệt kê các thiết bị mà Buyer/Organizer đang đăng nhập (như Facebook hay làm).
- **Nghiệp vụ:** Khi user quên mật khẩu hoặc bị rò rỉ tài khoản dẫn đến việc bị mua mất ghế, họ có thể vào check "Lịch sử đăng nhập" xem IP nào, trình duyệt nào đã truy cập.
- Tính năng này cũng giúp hệ thống Flash Sale detect xem 1 người dùng liệu có dùng chung 1 tool/bot bào vé trên nhiều IP đăng nhập bất thường hay không.

## 5. Danh Bạ Thanh Toán & Thông Tin Thu Hưởng (Payout/Billing Address)
- **Tính năng cho Buyer:** Người dùng không muốn nhập tay địa chỉ email, tên để nhận vé, hoá đơn quá nhiều lần. Tạo 1 cuốn sổ Billing Info (địa chỉ xuất hoá đơn đỏ, VAT). Lần sau mua vé chỉ cần click chọn.
- **Tính năng cho Organizer:** User Service sẽ là nơi duy nhất quản lý thông tin Account Ngân Hàng của Organizer một cách bảo mật. Core-Service (phần Payment/Payout) định kỳ (cuối sự kiện) sẽ call API sang User Service lấy STK Ngân Hàng của Organizer để kế toán chuyển quyền lợi tiền bán vé. (Các lệnh này nên yêu cầu mật khẩu cấp 2 hoặc OTP).

---

> [!TIP]
> **Lời Khuyên:** Để đồ án đạt điểm tối đa, KHUYẾN NGHỊ mạnh mẽ bạn nên làm **Tính năng 1 (Quy trình Onboarding Duyệt Organizer)** vì nó chặt chẽ về luồng hoạt động. Và **Tính năng 2 (Hệ thống Follow)** rất dễ demo, giao diện siêu đẹp, mang tính Social tạo Wow moment cho hệ thống bán vé.

Các tính năng này không tốn quá nhiều tài nguyên database nhưng làm hệ thống Flash Ticket sâu sắc và mang tính chất Doanh nghiệp (Enterprise) hơn rất nhiều.
