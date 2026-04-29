# 🎯 TƯ VẤN KHAI PHÁ DỮ LIỆU — TheLook Ecommerce Dataset

## SƠ LƯỢC VỀ THELOOK

**TheLook** là dataset giả lập của Google/Looker, mô phỏng một **cửa hàng thời trang trực tuyến toàn cầu**.

### Các bảng sẽ dùng (KHÔNG dùng bảng `events`)

| Bảng | ~Số dòng | Cột quan trọng | Vai trò |
|---|---|---|---|
| `users` | ~100K | `id`, `age`, `gender`, `country`, `city`, `traffic_source`, `created_at` | Hồ sơ khách hàng |
| `orders` | ~100K | `order_id`, `user_id`, `status`, `created_at`, `num_of_item` | Đơn hàng (1 dòng = 1 đơn) |
| `order_items` | ~180K | `order_id`, `user_id`, `product_id`, `sale_price`, `status`, `created_at`, `shipped_at`, `delivered_at`, `returned_at` | Chi tiết từng sản phẩm trong đơn |
| `products` | ~30K | `id`, `cost`, `category`, `name`, `brand`, `retail_price`, `department` | Danh mục sản phẩm |
| `distribution_centers` | ~10 | `id`, `name`, `latitude`, `longitude` | Trung tâm phân phối |
| ~~`events`~~ | ~~2.4M~~ | — | ❌ **KHÔNG DÙNG** |
| `inventory_items` | ~nhiều | có thể dùng hoặc không | Tuỳ hướng đi |

### Đặc thù quan trọng của TheLook (khác Olist)

1. **Thời trang** — sản phẩm có `category` (Jeans, Dresses, Outerwear...), `department` (Men/Women), `brand`
2. **Toàn cầu** — khách hàng từ nhiều quốc gia (China, USA, Brazil, UK, Germany...)
3. **Có trạng thái đơn hàng** — `status`: Complete, Cancelled, Returned, Shipped, Processing
4. **Có nguồn traffic** — `traffic_source`: Search, Organic, Facebook, Email, Display
5. **Có thông tin giao hàng** — `shipped_at`, `delivered_at`, `returned_at` → tính được thời gian giao hàng
6. **Multi-item orders** — 1 đơn hàng có thể chứa nhiều sản phẩm (khác Olist nơi 96% chỉ mua 1 món)

---

## CÁI NHÌN TỔNG QUÁT: "KỂ CHUYỆN" TRONG DATA MINING

> [!IMPORTANT]
> **Trả lời thắc mắc của bạn:** Hoàn toàn được. Trong thực tế, đa số dự án data mining trong doanh nghiệp đều bắt đầu từ một **giả thuyết kinh doanh** (business hypothesis) rồi dùng dữ liệu để **xác nhận hoặc bác bỏ** giả thuyết đó. Bạn không cần "phát hiện quy luật mới chưa ai biết" — điều đó là nghiên cứu tiến sĩ, không phải bài tập sinh viên.

**Quy trình chuẩn trong thực tế:**
1. Quan sát hiện tượng kinh doanh (ví dụ: "tỷ lệ hoàn trả đơn hàng cao")
2. Đặt giả thuyết (ví dụ: "phải chăng sản phẩm đắt tiền bị trả lại nhiều hơn?")
3. Dùng phương pháp data mining để kiểm chứng
4. Rút ra hành động (action) cho doanh nghiệp

Cái bạn gọi là "tự vẽ ra câu chuyện" chính xác là cách data mining hoạt động trong thực tế. Giảng viên sẽ đánh giá cao nếu bạn có một câu chuyện **mạch lạc**, các lớp phân tích **liên kết với nhau**, và kết luận **có thể hành động được** (actionable).

---

## CÁC HƯỚNG ĐI CỤ THỂ

Dưới đây là **3 hướng đi chính**, mỗi hướng đều sát thực tế TheLook, đều dùng đủ 3 phương pháp bắt buộc (Phân cụm, Phân loại, Luật kết hợp), và có gợi ý phương pháp bổ sung.

---

### 🅰️ HƯỚNG A: "GIẢM TỶ LỆ HOÀN TRẢ ĐƠN HÀNG" (Return Rate Reduction)

**Bối cảnh thực tế:** Trong ngành thời trang online, tỷ lệ hoàn trả (return rate) là vấn đề nghiêm trọng — trung bình 20-30% đơn hàng bị trả lại. TheLook có cột `status = 'Returned'` và `returned_at`, đây là dữ liệu vàng.

**Câu chuyện xuyên suốt:** *"Cửa hàng thời trang TheLook đang chịu tổn thất lớn từ đơn hàng bị hoàn trả. Ban giám đốc yêu cầu đội phân tích dữ liệu tìm hiểu: Ai hoàn trả? Trả sản phẩm gì? Và có thể dự đoán trước đơn nào sẽ bị trả không?"*

#### Lớp 1 — Phân cụm khách hàng (Clustering)
- **Input:** Từ `order_items` + `users`, tính cho mỗi user: tổng đơn, tổng chi tiêu, tỷ lệ hoàn trả, tuổi, số category đã mua
- **Phương pháp:** K-Means
- **Output:** Phân khúc khách hàng (ví dụ: "Khách trung thành - ít trả", "Khách mua nhiều - trả nhiều", "Khách mới - 1 lần")
- **Liên kết sang Lớp 2:** Gán nhãn CLUSTER cho mỗi user

#### Lớp 2 — Dự đoán đơn hàng bị hoàn trả (Classification)
- **Input:** Mỗi dòng `order_items` với features: `sale_price`, `category`, `department`, `brand`, `traffic_source` của user, `age`, `gender`, `country`, CLUSTER từ Lớp 1
- **Target (Y):** `status == 'Returned'` hay không (binary: 0/1)
- **Phương pháp:** Decision Tree → Feature Importance → Biết được yếu tố nào ảnh hưởng nhất đến hoàn trả
- **Giá trị:** Doanh nghiệp có thể cảnh báo sớm khi phát hiện đơn hàng có xác suất hoàn trả cao

#### Lớp 3 — Luật kết hợp sản phẩm bị trả (Association Rules)
- **Input:** Chỉ lấy các đơn hàng bị trả (`status = 'Returned'`), group `category` (hoặc `brand`) theo `order_id`
- **Phương pháp:** Apriori
- **Output:** "Khi khách mua Jeans cùng Outerwear, khả năng trả cả 2 rất cao" → Gợi ý điều chỉnh combo sản phẩm
- **Lưu ý:** Dùng `category` thay vì `product_id` để tránh ma trận thưa (TheLook có ~30K product nhưng chỉ ~20 category)

#### ➕ Phương pháp bổ sung gợi ý: Phân tích thời gian hoàn trả
- Tính `returned_at - delivered_at` = số ngày từ nhận hàng đến trả
- Vẽ phân phối → Phát hiện: Đa số trả trong 7 ngày đầu? → Gợi ý kéo dài thời gian thử đồ

**Điểm mạnh:** ⭐⭐⭐⭐⭐ Rất sát thực tế ngành thời trang, dữ liệu TheLook hỗ trợ đầy đủ, câu chuyện rõ ràng.  
**Rủi ro:** Cần kiểm tra tỷ lệ Returned trong TheLook có đủ lớn không (nếu quá ít thì model classification bị imbalanced).

---

### 🅱️ HƯỚNG B: "TỐI ƯU CHIẾN LƯỢC MARKETING THEO NGUỒN KHÁCH" (Traffic Source Optimization)

**Bối cảnh thực tế:** Cửa hàng chi ngân sách marketing cho nhiều kênh (Search, Facebook, Email, Organic, Display). Kênh nào đem lại khách hàng giá trị nhất? Kênh nào nhiều đơn hủy?

**Câu chuyện xuyên suốt:** *"TheLook cần tối ưu ngân sách marketing. Đội data cần phân tích: Khách hàng từ mỗi nguồn traffic có hành vi mua sắm khác nhau không? Có thể dự đoán nguồn traffic của khách dựa trên hành vi mua không? Và khách từ nguồn nào thường mua combo sản phẩm?"*

#### Lớp 1 — Phân cụm hành vi mua sắm (Clustering)
- **Input:** Cho mỗi user: tổng chi tiêu (`sale_price`), số đơn, số category khác nhau, average order value (AOV), tỷ lệ Complete vs Cancel
- **Phương pháp:** K-Means
- **Output:** Phân khúc hành vi: "Khách VIP", "Khách tiết kiệm", "Khách dạo chơi"

#### Lớp 2 — Dự đoán nguồn Traffic (Classification)
- **Input:** Features hành vi: AOV, số lượng mua, category yêu thích, `age`, `gender`, `country`, CLUSTER
- **Target (Y):** `traffic_source` (5 lớp: Search, Organic, Facebook, Email, Display)
- **Phương pháp:** Decision Tree (multi-class)
- **Giá trị:** Nếu model dự đoán tốt → các nguồn traffic thực sự thu hút các loại khách khác nhau → cơ sở phân bổ ngân sách

#### Lớp 3 — Luật kết hợp category theo nguồn traffic (Association Rules)
- **Input:** Mỗi đơn hàng = danh sách `category` đã mua. Chạy riêng cho từng `traffic_source` hoặc gộp chung
- **Phương pháp:** Apriori
- **Output:** "Khách từ Facebook thường mua Dresses + Accessories cùng nhau" vs "Khách từ Search mua Jeans + Tops"

#### ➕ Phương pháp bổ sung gợi ý: Cohort Analysis
- Nhóm user theo tháng `created_at` → tính retention rate (% quay lại mua lần 2)
- So sánh retention giữa các traffic_source → "Khách từ Email có retention cao nhất"

**Điểm mạnh:** ⭐⭐⭐⭐ Câu chuyện marketing rất thực tế, TheLook có `traffic_source` sẵn.  
**Rủi ro:** TheLook là dữ liệu giả lập, `traffic_source` có thể phân bố đều (ít insight đặc biệt). Cần xem thực tế.

---

### 🅲 HƯỚNG C: "PHÂN TÍCH THỊ TRƯỜNG THỜI TRANG XUYÊN QUỐC GIA" (Cross-Country Fashion Analytics)

**Bối cảnh thực tế:** TheLook bán hàng toàn cầu. Xu hướng thời trang khác nhau giữa các quốc gia. Người Mỹ thích gì? Người Trung Quốc mua gì? Mùa nào bán gì?

**Câu chuyện xuyên suốt:** *"TheLook muốn mở rộng thị trường quốc tế. Đội data cần trả lời: Các thị trường quốc gia có gu thời trang khác nhau không? Có thể dự đoán quốc gia của khách dựa trên giỏ hàng? Và ở mỗi thị trường, sản phẩm nào thường được mua cùng nhau?"*

#### Lớp 1 — Phân cụm sản phẩm (Clustering)
- **Input:** Cho mỗi `product_id`: `retail_price`, `cost`, `department` (encode), số lần được mua, tỷ lệ bị trả
- **Phương pháp:** K-Means
- **Output:** Phân khúc sản phẩm: "Sản phẩm best-seller rẻ", "Sản phẩm premium ít mua", "Sản phẩm vấn đề (hay bị trả)"

#### Lớp 2 — Dự đoán quốc gia/khu vực khách hàng (Classification)
- **Input:** Dựa trên giỏ hàng: category yêu thích, price range, department, CLUSTER sản phẩm, `gender`, `age`
- **Target (Y):** `country` (gộp thành 4-5 khu vực lớn: North America, South America, Europe, Asia, Other)
- **Phương pháp:** Decision Tree
- **Giá trị:** "Khách châu Á mua Dresses + Accessories giá rẻ, khách Âu mua Outerwear + Jeans giá cao"

#### Lớp 3 — Luật kết hợp theo khu vực (Association Rules)
- **Input:** Chạy Apriori riêng cho từng khu vực địa lý
- **Output:** So sánh luật kết hợp giữa các thị trường → Gợi ý chiến lược sản phẩm khác nhau cho từng khu vực

#### ➕ Phương pháp bổ sung gợi ý: Phân tích mùa vụ
- Nhóm đơn hàng theo tháng + category → tìm pattern mùa vụ (áo khoác bán nhiều tháng 10-12? Swim bán nhiều tháng 5-7?)
- Heatmap: Tháng × Category → mật độ doanh thu

**Điểm mạnh:** ⭐⭐⭐⭐ Tận dụng tốt tính toàn cầu của TheLook, câu chuyện rộng.  
**Rủi ro:** Vì TheLook là dữ liệu giả lập, sự khác biệt giữa các quốc gia có thể không rõ ràng. Nhưng nếu phát hiện "không có sự khác biệt" → đó cũng là một kết quả data mining hợp lệ.

---

## SO SÁNH 3 HƯỚNG

| Tiêu chí | 🅰️ Hoàn trả | 🅱️ Marketing | 🅲 Xuyên quốc gia |
|---|---|---|---|
| **Câu chuyện rõ ràng** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Sát thực tế** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Dữ liệu đủ chất lượng** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Association Rules khả thi** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Dễ trình bày / bảo vệ** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Dùng bảng nào** | orders, order_items, users, products | orders, order_items, users, products | orders, order_items, users, products |

> [!TIP]
> **Khuyến nghị cá nhân:** Hướng A (Hoàn trả) là lựa chọn an toàn và mạnh nhất. Câu chuyện rõ, actionable, và rất dễ giải thích cho giảng viên: "Tôi dùng clustering để phân khúc khách, classification để dự đoán đơn nào sẽ bị trả, và association rules để tìm combo sản phẩm hay bị trả cùng nhau."

---

## ĐIỂM KHÁC BIỆT LỚN SO VỚI DỰ ÁN OLIST CŨ

| Khía cạnh | Olist cũ (demo) | TheLook mới |
|---|---|---|
| **Phân cụm cái gì** | Cụm đơn hàng (theo GIA, TRONGLUONG) | Cụm **khách hàng** (theo hành vi mua) hoặc cụm **sản phẩm** |
| **Phân loại cái gì** | Dự đoán kênh bán hàng (Online/Offline) | Dự đoán **trả hàng** hoặc **nguồn traffic** hoặc **quốc gia** |
| **Luật kết hợp trên gì** | Trên MÃ sản phẩm → ma trận thưa, 96% đơn 1 món | Trên **CATEGORY** (~20 loại) → ma trận đặc hơn, đơn nhiều món hơn |
| **Vấn đề ma trận thưa** | Rất nặng (20K product, 96% đơn 1 item) | Nhẹ hơn nhiều nếu dùng category/brand thay vì product_id |

---

## BƯỚC TIẾP THEO

Bạn hãy cho tôi biết:
1. **Chọn hướng nào?** (A, B, C, hoặc mix)
2. **Có muốn kết hợp hướng nào lại không?** (ví dụ: Lớp 1+2 theo hướng A, Lớp 3 theo hướng C)
3. **Đã có file CSV của TheLook chưa?** Nếu chưa, bạn tải từ đâu (BigQuery / Kaggle)?

Sau khi bạn chọn hướng, tôi sẽ tạo implementation plan chi tiết và bắt đầu code.
