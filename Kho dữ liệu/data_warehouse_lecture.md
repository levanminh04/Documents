# 📚 BÀI GIẢNG NHẬP MÔN DATA WAREHOUSE
## Dành cho người mới bắt đầu tuyệt đối – Áp dụng vào Bài toán Chuỗi cửa hàng

> **Cách đọc tài liệu này:** Đọc từ đầu đến cuối, theo thứ tự. Mỗi phần xây dựng trên phần trước. Đừng nhảy cóc!
>
> **Quy ước:** `[📖 Slide X]` = Kiến thức có trong slide bài giảng, bạn nên đối chiếu.

---

# PHẦN 1: CÂU CHUYỆN TRƯỚC KHI HỌC KỸ THUẬT

## 1.1 Hãy tưởng tượng bạn là ông chủ...

Bạn là chủ của **"Đặc Sản Việt"** – một chuỗi cửa hàng bán đặc sản & quà lưu niệm toàn quốc.

```
🏪 Hà Nội:    3 cửa hàng  (bánh cốm, ô mai, trà sen...)
🏪 Đà Nẵng:   2 cửa hàng  (mỳ Quảng, bánh tráng...)
🏪 TP.HCM:    4 cửa hàng  (bánh tráng trộn, khô bò...)
🏪 Đà Lạt:    2 cửa hàng  (mứt dâu, atiso...)
```

Khách hàng của bạn có 2 loại:
- 🧳 **Khách du lịch**: Được hướng dẫn viên dẫn tới cửa hàng mua quà
- 📮 **Khách bưu điện**: Ở xa, gửi đơn đặt hàng qua bưu điện

## 1.2 Vấn đề kinh doanh thực tế

Cuối tháng, bạn muốn biết:
- "Tháng này bán được bao nhiêu tiền?"
- "Mặt hàng nào bán chạy nhất?"
- "Thành phố nào mang lại doanh thu cao nhất?"
- "Khách du lịch hay khách bưu điện mua nhiều hơn?"

**Nhưng vấn đề là:**

| Bộ phận | Phần mềm | Lưu gì |
|---------|----------|--------|
| Văn phòng đại diện | Oracle Database | Thông tin khách hàng |
| Phòng bán hàng | SQL Server | Đơn hàng, cửa hàng, mặt hàng |

→ Dữ liệu nằm **rải rác ở 2 nơi khác nhau**, dùng **2 phần mềm khác nhau**.

Muốn trả lời câu hỏi "Khách hàng A đã mua gì, ở cửa hàng nào?" → Phải mở **cả 2 hệ thống**, tìm kiếm thủ công, rồi ghép lại. **Cực kỳ chậm và dễ sai!**

> 💡 **Đây chính là lý do Data Warehouse ra đời**: Gom tất cả dữ liệu về MỘT NƠI để phân tích dễ dàng.

---

# PHẦN 2: DATABASE vs DATA WAREHOUSE

`[📖 Slide Lecture 1&2]`

## 2.1 Database (Cơ sở dữ liệu) – Cuốn sổ ghi chép hàng ngày

**Database** giống như **cuốn sổ ghi chép hàng ngày** của cửa hàng:
- Khách A vừa mua 3 hộp bánh → **Ghi vào sổ**
- Nhập thêm 100 hộp bánh → **Ghi vào sổ**
- Khách B đổi hàng → **Sửa trong sổ**

**Đặc điểm:**
- Ghi nhanh, sửa nhanh, xóa nhanh
- Phục vụ **công việc hàng ngày** (bán hàng, nhập hàng)
- Dữ liệu **luôn thay đổi** (cập nhật liên tục)

## 2.2 Data Warehouse (Kho dữ liệu) – Thư viện báo cáo

**Data Warehouse** giống như **thư viện lưu trữ tất cả sổ sách** từ mọi cửa hàng:
- Không dùng để ghi chép hàng ngày
- Dùng để **đọc, phân tích, so sánh**
- Thu thập dữ liệu từ **nhiều cuốn sổ** (nhiều database) về một nơi

**So sánh trực quan:**

| Tiêu chí | Database (Sổ ghi chép) | Data Warehouse (Thư viện) |
|----------|----------------------|--------------------------|
| **Mục đích** | Ghi chép hàng ngày | Phân tích, báo cáo |
| **Thao tác chính** | Thêm/Sửa/Xóa | Chỉ Đọc |
| **Dữ liệu** | Hiện tại (hôm nay) | Lịch sử (nhiều tháng/năm) |
| **Người dùng** | Nhân viên bán hàng | Giám đốc, nhà phân tích |
| **Tốc độ** | Nhanh cho từng giao dịch | Nhanh cho phân tích lớn |
| **Câu hỏi** | "Khách A mua gì hôm nay?" | "Doanh thu Q1 so với Q2?" |

```
📝 Database                    📊 Data Warehouse
┌─────────────┐               ┌──────────────────┐
│ Ghi chép    │               │ Phân tích        │
│ từng giao   │──── ETL ────▶│ tổng hợp từ      │
│ dịch nhỏ    │               │ nhiều nguồn      │
└─────────────┘               └──────────────────┘
  "Bán 3 hộp                   "Tổng doanh thu
   bánh cốm"                    tháng 3 là 500tr"
```

## 2.3 Áp dụng vào bài tập chuỗi cửa hàng

Trong bài tập, doanh nghiệp có **2 Database nguồn**:

```
┌───────────────────────┐    ┌───────────────────────┐
│   ORACLE DATABASE     │    │   SQL SERVER DATABASE  │
│   "Văn phòng ĐD"     │    │   "Bán hàng"          │
│                       │    │                       │
│ • Khách hàng          │    │ • Văn phòng đại diện  │
│ • KH du lịch          │    │ • Cửa hàng            │
│ • KH bưu điện         │    │ • Mặt hàng            │
│                       │    │ • Tồn kho             │
│                       │    │ • Đơn đặt hàng        │
│                       │    │ • Chi tiết đơn hàng   │
└───────────┬───────────┘    └───────────┬───────────┘
            │                            │
            └──────────┬─────────────────┘
                       │ ETL
                       ▼
            ┌──────────────────┐
            │  DATA WAREHOUSE  │
            │  (Kho dữ liệu)  │
            │                  │
            │  Phân tích OLAP  │
            └──────────────────┘
```

> 💡 **Tại sao 2 database khác nhau?** Vì trong thực tế, bộ phận quản lý khách hàng và bộ phận bán hàng thường dùng phần mềm khác nhau. Bài tập mô phỏng đúng thực tế này.

---

# PHẦN 3: OLTP vs OLAP

`[📖 Slide Lecture 1&2]`

## 3.1 OLTP – Online Transaction Processing

**OLTP** = Xử lý giao dịch trực tuyến = **Công việc hàng ngày**

Ví dụ OLTP:
- Nhân viên bán hàng nhập đơn hàng mới
- Cập nhật số lượng tồn kho
- Thêm khách hàng mới

→ OLTP phục vụ **Database thông thường**.

## 3.2 OLAP – Online Analytical Processing

**OLAP** = Xử lý phân tích trực tuyến = **Phân tích dữ liệu**

Ví dụ OLAP:
- Giám đốc hỏi: "So sánh doanh thu 4 quý năm nay"
- "Thành phố nào bán chạy nhất loại mặt hàng X?"
- "Xu hướng bán hàng 3 năm gần đây?"

→ OLAP phục vụ **Data Warehouse**.

**So sánh:**

| | OLTP (Giao dịch) | OLAP (Phân tích) |
|---|---|---|
| **Ai dùng?** | Nhân viên | Giám đốc, nhà phân tích |
| **Làm gì?** | Thêm/Sửa/Xóa dữ liệu | Đọc & phân tích dữ liệu |
| **Câu hỏi** | "Bán cho ai? Bao nhiêu?" | "Tổng doanh thu Q1?" |
| **Dữ liệu** | Mới nhất (real-time) | Lịch sử (theo thời gian) |
| **Tốc độ** | Nhanh cho 1 giao dịch | Nhanh cho triệu bản ghi |

## 3.3 Áp dụng vào bài tập

Bài tập yêu cầu bạn tạo **báo cáo OLAP** – tức là trả lời các câu hỏi phân tích. Ví dụ câu hỏi 7 trong đề:

> "Tìm mức độ tồn kho của một mặt hàng cụ thể tại tất cả các cửa hàng ở một thành phố"

Đây là câu hỏi **OLAP** vì nó cần **tổng hợp dữ liệu** từ nhiều cửa hàng, không chỉ tra cứu 1 bản ghi.

---

# PHẦN 4: FACT TABLE & DIMENSION

`[📖 Slide Lecture 3, 4]`

## 4.1 Fact Table – Bảng sự kiện (Trái tim của Data Warehouse)

**Fact Table** = Bảng ghi lại **các sự kiện đã xảy ra** mà bạn muốn đo lường.

> 🎯 **Cách nhớ**: "Fact" = "Sự thật" = "Chuyện gì đã xảy ra?"

Trong chuỗi cửa hàng, sự kiện quan trọng nhất là: **"Khách hàng đặt mua hàng"**.

Mỗi dòng trong Fact Table trả lời câu hỏi:

> "Khách hàng nào, đã mua mặt hàng gì, bao nhiêu cái, giá bao nhiêu, vào ngày nào?"

**Ví dụ Fact Table đơn giản:**

| MãĐơn | MãKH | MãMH | MãCửaHàng | NgàyĐặt | SốLượng | GiáĐặt |
|--------|------|------|-----------|----------|---------|--------|
| DH001 | KH01 | MH05 | CH03 | 2024-01-15 | 3 | 50000 |
| DH001 | KH01 | MH12 | CH03 | 2024-01-15 | 1 | 120000 |
| DH002 | KH03 | MH05 | CH07 | 2024-01-16 | 10 | 50000 |

**Fact Table chứa 2 loại thông tin:**

```
┌─────────────────────────────────────────────┐
│              FACT TABLE                      │
│                                              │
│  🔑 Khóa ngoại (FK)     📏 Measure (Đo)    │
│  ─────────────────       ───────────────     │
│  MãKH        ──┐        Số lượng đặt        │
│  MãMH        ──┤        Giá đặt             │
│  MãCửaHàng   ──┤        Tổng tiền           │
│  MãThờiGian  ──┘                             │
│       │                                      │
│       ▼                                      │
│  (trỏ đến các                                │
│   Dimension Table)                           │
└─────────────────────────────────────────────┘
```

- **Khóa ngoại (FK)**: Liên kết đến các bảng chiều → trả lời "Ai? Cái gì? Ở đâu? Khi nào?"
- **Measure (Số đo)**: Các con số bạn muốn tính toán → "Bao nhiêu? Tổng bao nhiêu tiền?"

## 4.2 Dimension – Chiều phân tích (Góc nhìn)

**Dimension** = **Góc nhìn** để phân tích dữ liệu.

> 🎯 **Cách nhớ**: "Dimension" = "Chiều" = "Nhìn dữ liệu từ góc nào?"

Khi giám đốc hỏi: **"Doanh thu tháng 1 của bánh cốm ở Hà Nội là bao nhiêu?"**

Câu hỏi này nhìn dữ liệu từ **3 góc (3 chiều)**:
- ⏰ **Chiều Thời gian**: Tháng 1
- 📦 **Chiều Sản phẩm**: Bánh cốm
- 📍 **Chiều Địa điểm**: Hà Nội

**Mỗi Dimension là một bảng chứa thông tin mô tả chi tiết:**

**Dim_Product (Chiều Sản phẩm):**

| MãMH | Mô tả | Kích cỡ | Trọng lượng | Giá |
|------|--------|---------|-------------|-----|
| MH05 | Bánh cốm | Hộp 500g | 500g | 50000 |
| MH12 | Trà sen | Hộp 200g | 200g | 120000 |

**Dim_Customer (Chiều Khách hàng):**

| MãKH | TênKH | ThànhPhố | LoạiKH | NgàyĐặtĐầu |
|------|-------|----------|--------|-------------|
| KH01 | Nguyễn Văn A | Hà Nội | Du lịch | 2024-01-10 |
| KH03 | Trần Thị B | TP.HCM | Bưu điện | 2023-06-15 |

**Dim_Time (Chiều Thời gian):**

| MãTG | Ngày | Tháng | Quý | Năm |
|------|------|-------|-----|-----|
| T001 | 15 | 1 | Q1 | 2024 |
| T002 | 16 | 1 | Q1 | 2024 |

## 4.3 Fact vs Dimension – Phân biệt thế nào?

| | Fact (Sự kiện) | Dimension (Chiều) |
|---|---|---|
| **Chứa gì?** | Số liệu đo lường | Thông tin mô tả |
| **Ví dụ** | Số lượng: 3, Giá: 50000 | Tên: "Bánh cốm", TP: "Hà Nội" |
| **Thay đổi?** | Thêm liên tục (mỗi đơn hàng) | Ít thay đổi |
| **Câu hỏi** | "Bao nhiêu?" | "Cái gì? Ai? Ở đâu? Khi nào?" |

> 💡 **Mẹo nhớ**: Nếu bạn có thể **cộng/trừ/trung bình** một giá trị → đó là **Measure** trong **Fact Table**. Nếu bạn dùng nó để **lọc/nhóm** → đó là **Dimension**.

---

# PHẦN 5: STAR SCHEMA – LƯỢC ĐỒ HÌNH SAO

`[📖 Slide Lecture 3, 4]`

## 5.1 Star Schema là gì?

**Star Schema** = Cách sắp xếp các bảng trong Data Warehouse **trông giống hình ngôi sao**.

- **Trung tâm** (tâm sao): Fact Table
- **Các cánh sao**: Các Dimension Table

```
                    ┌──────────────┐
                    │ Dim_Customer │
                    │   (Khách)    │
                    └──────┬───────┘
                           │
   ┌──────────────┐  ┌─────┴──────┐  ┌──────────────┐
   │  Dim_Store   │──│            │──│ Dim_Product  │
   │  (Cửa hàng) │  │ FACT_ORDER │  │  (Mặt hàng)  │
   └──────────────┘  │  (Đơn đặt  │  └──────────────┘
                     │   hàng)    │
   ┌──────────────┐  │            │  ┌──────────────┐
   │  Dim_City    │──│            │──│  Dim_Time    │
   │ (Thành phố)  │  └────────────┘  │ (Thời gian)  │
   └──────────────┘                  └──────────────┘
```

> 💡 **Tại sao gọi là "hình sao"?** Vì khi vẽ ra, bảng Fact ở giữa và các bảng Dimension tỏa ra xung quanh, trông giống ngôi sao!

## 5.2 Star Schema cho bài toán chuỗi cửa hàng

### Fact_Order (Bảng sự kiện – Đơn hàng)

| Cột | Loại | Mô tả |
|-----|------|-------|
| `MaDon` | FK | Mã đơn đặt hàng |
| `MaKH` | FK | → Dim_Customer |
| `MaMH` | FK | → Dim_Product |
| `MaCuaHang` | FK | → Dim_Store |
| `MaThanhPho` | FK | → Dim_City |
| `MaThoiGian` | FK | → Dim_Time |
| `SoLuongDat` | Measure | Số lượng đặt |
| `GiaDat` | Measure | Đơn giá |
| `TongTien` | Measure | = SốLượng × Giá |

### Dim_Product (Chiều Sản phẩm)

| Cột | Mô tả |
|-----|-------|
| `MaMH` (PK) | Mã mặt hàng |
| `MoTa` | Mô tả mặt hàng |
| `KichCo` | Kích cỡ |
| `TrongLuong` | Trọng lượng |
| `Gia` | Giá niêm yết |

### Dim_Customer (Chiều Khách hàng)

| Cột | Mô tả |
|-----|-------|
| `MaKH` (PK) | Mã khách hàng |
| `TenKH` | Tên khách hàng |
| `MaThanhPho` | Thành phố sinh sống |
| `NgayDatDau` | Ngày đặt hàng đầu tiên |
| `LoaiKH` | Du lịch / Bưu điện / Cả hai |
| `HDVDuLich` | Hướng dẫn viên (nếu KH du lịch) |
| `DiaChiBuuDien` | Địa chỉ (nếu KH bưu điện) |

### Dim_Store (Chiều Cửa hàng)

| Cột | Mô tả |
|-----|-------|
| `MaCuaHang` (PK) | Mã cửa hàng |
| `MaThanhPho` | Thành phố của cửa hàng |
| `SoDienThoai` | Số điện thoại |

### Dim_City (Chiều Thành phố)

| Cột | Mô tả |
|-----|-------|
| `MaThanhPho` (PK) | Mã thành phố |
| `TenThanhPho` | Tên thành phố |
| `DiaChiVP` | Địa chỉ văn phòng đại diện |
| `Bang` | Bang |

### Dim_Time (Chiều Thời gian)

| Cột | Mô tả |
|-----|-------|
| `MaThoiGian` (PK) | Mã thời gian |
| `Ngay` | Ngày |
| `Thang` | Tháng |
| `Quy` | Quý |
| `Nam` | Năm |
| `NgayTrongTuan` | Thứ mấy |

## 5.3 Snowflake Schema (Bông tuyết) – Biến thể

`[📖 Slide Lecture 3, 4]`

**Snowflake** = Star Schema nhưng các Dimension được **tách nhỏ hơn nữa** (chuẩn hóa).

```
Star Schema:                    Snowflake Schema:
                               
  Dim_Store ── FACT             Dim_Store ── FACT
  (có cả tên                       │
   thành phố)                  Dim_City
                                   │
                               Dim_Bang
```

Trong Star Schema, bảng `Dim_Store` chứa luôn tên thành phố, bang. Trong Snowflake, tách ra thành `Dim_City` riêng, `Dim_Bang` riêng.

| | Star Schema | Snowflake Schema |
|---|---|---|
| **Đơn giản** | ✅ Dễ hiểu, dễ query | ❌ Phức tạp hơn |
| **Tốc độ** | ✅ Nhanh (ít JOIN) | ❌ Chậm hơn (nhiều JOIN) |
| **Trùng lặp** | ❌ Có trùng lặp dữ liệu | ✅ Ít trùng lặp |
| **Phổ biến** | ✅ Dùng nhiều hơn | Ít dùng hơn |

> 💡 **Trong bài tập, dùng Star Schema** là đủ và phù hợp nhất.

---

# PHẦN 6: DATA CUBE – KHỐI DỮ LIỆU

`[📖 Slide Lecture 5, 6]`

## 6.1 Data Cube là gì?

Hãy tưởng tượng bạn có một **khối Rubik**, mỗi ô chứa một con số (doanh thu).

- **Trục ngang**: Sản phẩm (bánh cốm, ô mai, trà sen)
- **Trục dọc**: Thành phố (Hà Nội, Đà Nẵng, TP.HCM)
- **Trục sâu**: Thời gian (Q1, Q2, Q3, Q4)

```
                    Thời gian
                   ╱
                  ╱  Q1    Q2    Q3    Q4
                 ╱ ┌─────┬─────┬─────┬─────┐
   Thành phố    ╱  │     │     │     │     │ Bánh cốm
               ╱   ├─────┼─────┼─────┼─────┤
  Hà Nội ────╱──▶  │ 50  │ 30  │ 45  │ 80  │ Ô mai
  Đà Nẵng ──╱──▶   ├─────┼─────┼─────┼─────┤
  TP.HCM ──╱──▶    │     │     │     │     │ Trà sen
                    └─────┴─────┴─────┴─────┘
                         Sản phẩm ──────────▶
```

Mỗi ô giao nhau cho bạn biết: **"Doanh thu của [sản phẩm X] tại [thành phố Y] trong [quý Z] là bao nhiêu?"**

## 6.2 Tại sao cần Data Cube?

Vì nó cho phép bạn **nhìn dữ liệu từ nhiều góc** cùng lúc:
- Cắt theo thời gian → Doanh thu Q1 của tất cả sản phẩm, tất cả thành phố
- Cắt theo sản phẩm → Doanh thu bánh cốm qua các quý, các thành phố
- Cắt theo thành phố → Doanh thu Hà Nội qua các quý, các sản phẩm

---

# PHẦN 7: OLAP OPERATIONS – CÁC THAO TÁC PHÂN TÍCH

`[📖 Slide Lecture 5, 6]`

## 7.1 Roll-up (Cuộn lên) – Nhìn tổng quát hơn

**Roll-up** = Gom nhỏ thành lớn, chi tiết thành tổng quát.

```
TRƯỚC (chi tiết theo Tháng):          SAU roll-up (tổng quát theo Quý):
                                       
Tháng 1: 100 triệu                    Q1: 330 triệu (= 100+110+120)
Tháng 2: 110 triệu        ────▶       Q2: 360 triệu
Tháng 3: 120 triệu                    Q3: 310 triệu
Tháng 4: 115 triệu                    Q4: 400 triệu
...                                    
```

**Ví dụ bài tập:** Từ doanh thu từng cửa hàng → Roll-up thành doanh thu từng thành phố → Roll-up thành doanh thu từng bang.

## 7.2 Drill-down (Khoan xuống) – Nhìn chi tiết hơn

**Drill-down** = Ngược lại với Roll-up. Từ tổng quát → chi tiết.

```
TRƯỚC (tổng theo Quý):               SAU drill-down (chi tiết theo Tháng):
                                       
Q1: 330 triệu             ────▶       Tháng 1: 100 triệu
                                       Tháng 2: 110 triệu
                                       Tháng 3: 120 triệu
```

**Ví dụ bài tập:** Thấy Q1 doanh thu giảm → Drill-down xem tháng nào giảm → Drill-down xem ngày nào giảm.

## 7.3 Slice (Cắt lát) – Chọn 1 giá trị của 1 chiều

**Slice** = Cắt 1 lát khối Rubik → giảm từ 3 chiều xuống 2 chiều.

```
Cube 3 chiều                    Slice: Chọn "Q1"
(TP × SP × TG)                  → Bảng 2 chiều (TP × SP)
                                 
┌─────────────┐                  ┌──────────────────┐
│  ╱╱╱╱╱╱╱╱╱  │                  │ Chỉ Q1           │
│ ╱╱╱ Q1 ╱╱╱  │   ────▶         │       BC   OM    │
│╱╱╱╱╱╱╱╱╱╱╱  │                  │ HN    50   30   │
│  Q2  Q3  Q4  │                  │ ĐN    20   40   │
└─────────────┘                  │ HCM   35   25   │
                                 └──────────────────┘
```

**Ví dụ bài tập câu 7:** "Tồn kho mặt hàng X tại thành phố Y" = Slice theo mặt hàng VÀ thành phố.

## 7.4 Dice (Cắt khối con) – Chọn nhiều giá trị của nhiều chiều

**Dice** = Cắt 1 khối nhỏ từ khối lớn (chọn nhiều điều kiện).

```
Dice: Chọn (HN, ĐN) × (Bánh cốm, Ô mai) × (Q1, Q2)
→ Khối con nhỏ hơn
```

**Ví dụ:** "Doanh thu bánh cốm và ô mai tại Hà Nội và Đà Nẵng trong Q1 và Q2"

---

# PHẦN 8: ETL – TRÍCH XUẤT, CHUYỂN ĐỔI, NẠP

`[📖 Slide Lecture 3]`

## 8.1 ETL là gì?

**ETL** = **E**xtract (Trích xuất) + **T**ransform (Chuyển đổi) + **L**oad (Nạp)

Hãy tưởng tượng bạn muốn nấu một nồi lẩu từ nguyên liệu mua ở 2 chợ khác nhau:

| Bước | ETL | Ví dụ nấu lẩu |
|------|-----|----------------|
| **E** – Extract | Lấy dữ liệu từ nguồn | Đi chợ A mua rau, chợ B mua thịt |
| **T** – Transform | Làm sạch, đồng nhất | Rửa rau, thái thịt, ướp gia vị |
| **L** – Load | Nạp vào kho dữ liệu | Cho tất cả vào nồi lẩu |

## 8.2 ETL trong bài tập chuỗi cửa hàng

```
 ┌─ EXTRACT ──────────────────────────────────────┐
 │                                                 │
 │  Oracle DB          SQL Server DB               │
 │  ┌──────────┐      ┌──────────────┐            │
 │  │ KháchHàng│      │ VănPhòngĐD   │            │
 │  │ KH_DuLịch│      │ CửaHàng      │            │
 │  │ KH_BưuĐiện│    │ MặtHàng      │            │
 │  └────┬─────┘      │ ĐơnĐặtHàng  │            │
 │       │             │ TồnKho       │            │
 │       │             └──────┬───────┘            │
 └───────┼────────────────────┼────────────────────┘
         │                    │
 ┌─ TRANSFORM ───────────────┼────────────────────┐
 │       │                    │                    │
 │       ▼                    ▼                    │
 │  • Ghép KH + KH_DuLịch + KH_BưuĐiện           │
 │    → Dim_Customer (1 bảng duy nhất)             │
 │  • Tạo cột LoạiKH = "Du lịch"/"Bưu điện"      │
 │  • Chuẩn hóa định dạng ngày tháng              │
 │  • Tạo Dim_Time từ các cột ngày                │
 │  • Tính TổngTiền = SốLượng × Giá               │
 └─────────────────────┬──────────────────────────┘
                       │
 ┌─ LOAD ──────────────┼──────────────────────────┐
 │                     ▼                           │
 │           DATA WAREHOUSE                        │
 │  ┌────────────────────────────┐                │
 │  │ Fact_Order                 │                │
 │  │ Dim_Product                │                │
 │  │ Dim_Customer               │                │
 │  │ Dim_Store                  │                │
 │  │ Dim_City                   │                │
 │  │ Dim_Time                   │                │
 │  └────────────────────────────┘                │
 └─────────────────────────────────────────────────┘
```

---

# PHẦN 9: ÁP DỤNG VÀO 9 CÂU HỎI TRONG BÀI TẬP

Dưới đây là phân tích từng câu hỏi theo kiến thức đã học:

| Câu | Yêu cầu (tóm tắt) | Loại OLAP | Dimension tham gia |
|-----|--------------------|-----------|--------------------|
| 1 | Cửa hàng + mặt hàng bán ở đó | Slice/Dice | Store, Product, City |
| 2 | Đơn hàng + khách hàng | Slice | Customer, Time |
| 3 | Cửa hàng bán mặt hàng KH đặt | Dice | Store, Product, Customer |
| 4 | VP đại diện có tồn kho > N | Slice + điều kiện | City, Product, Store |
| 5 | Chi tiết đơn hàng + cửa hàng | Drill-down | Order, Product, Store, City |
| 6 | TP và bang của khách hàng | Slice | Customer, City |
| 7 | Tồn kho mặt hàng theo TP | Slice | Product, Store, City |
| 8 | Chi tiết đơn hàng đầy đủ | Dice | Order, Product, Customer, Store, City |
| 9 | Phân loại khách hàng | Slice | Customer |

**Ví dụ chi tiết câu 7:**

> "Tìm mức tồn kho của mặt hàng X tại tất cả cửa hàng ở thành phố Y"

```
Thao tác OLAP: SLICE
  → Cắt theo Dim_Product: MaMH = 'X'
  → Cắt theo Dim_City: TenTP = 'Y'

Kết quả:
┌────────────┬──────────┬───────────┐
│ Cửa hàng   │ Mặt hàng │ Tồn kho  │
├────────────┼──────────┼───────────┤
│ CH01-HN    │ Bánh cốm │    50     │
│ CH02-HN    │ Bánh cốm │    30     │
│ CH03-HN    │ Bánh cốm │    25     │
├────────────┼──────────┼───────────┤
│ TỔNG HN    │ Bánh cốm │   105     │  ← Roll-up
└────────────┴──────────┴───────────┘
```

---

# PHẦN 10: DATA WAREHOUSE ARCHITECTURE

`[📖 Slide Lecture 1&2, 3]`

```
┌─────────────────────────────────────────────────────────┐
│                  KIẾN TRÚC TỔNG QUAN                    │
│                                                          │
│  ┌──────────┐  ┌──────────┐                             │
│  │ Oracle   │  │SQL Server│     DATA SOURCES            │
│  │ (KH)     │  │(Bán hàng)│     (Nguồn dữ liệu)       │
│  └────┬─────┘  └────┬─────┘                             │
│       │              │                                   │
│       ▼              ▼                                   │
│  ┌──────────────────────┐                               │
│  │      ETL LAYER       │   STAGING AREA                │
│  │  Extract-Transform   │   (Khu vực trung gian)        │
│  │       -Load          │                               │
│  └──────────┬───────────┘                               │
│             ▼                                            │
│  ┌──────────────────────┐                               │
│  │   DATA WAREHOUSE     │   STORAGE                     │
│  │   (Star Schema)      │   (Lưu trữ)                  │
│  │                      │                               │
│  │  Fact + Dimensions   │                               │
│  └──────────┬───────────┘                               │
│             ▼                                            │
│  ┌──────────────────────┐                               │
│  │   OLAP SERVER        │   ANALYSIS                    │
│  │   (Data Cube)        │   (Phân tích)                 │
│  └──────────┬───────────┘                               │
│             ▼                                            │
│  ┌──────────────────────┐                               │
│  │   BÁO CÁO / REPORT  │   PRESENTATION                │
│  │   (9 câu hỏi OLAP)  │   (Trình bày)                 │
│  └──────────────────────┘                               │
└─────────────────────────────────────────────────────────┘
```

---

# PHẦN 11: NHỮNG HIỂU LẦM PHỔ BIẾN

| ❌ Hiểu lầm | ✅ Sự thật |
|-------------|-----------|
| DW thay thế Database | DW **bổ sung** cho DB, không thay thế |
| DW lưu dữ liệu real-time | DW lưu dữ liệu **theo lịch trình** (hàng ngày/tuần) |
| Star Schema = ER Diagram | Star Schema **khác hoàn toàn** ER Diagram |
| Càng nhiều Dimension càng tốt | Chỉ cần Dimension **phục vụ phân tích** |
| Fact Table chỉ có 1 | Có thể có **nhiều Fact Table** cho các sự kiện khác nhau |
| OLAP = viết SQL | OLAP là **khái niệm**, SQL chỉ là 1 cách thực hiện |

---

# PHẦN 12: CHECKLIST THIẾT KẾ STAR SCHEMA

Khi thiết kế Star Schema cho bất kỳ bài toán nào, hãy trả lời lần lượt:

- [ ] **Bước 1:** Xác định **sự kiện kinh doanh** (Business Process) → Đó là Fact
  - *Bài tập: "Khách hàng đặt mua hàng" → Fact_Order*
- [ ] **Bước 2:** Xác định **mức chi tiết** (Grain) → Mỗi dòng Fact là gì?
  - *Bài tập: Mỗi dòng = "1 mặt hàng trong 1 đơn hàng"*
- [ ] **Bước 3:** Xác định **các chiều** (Dimensions) → Ai? Cái gì? Ở đâu? Khi nào?
  - *Bài tập: Customer, Product, Store, City, Time*
- [ ] **Bước 4:** Xác định **số đo** (Measures) → Đo cái gì?
  - *Bài tập: SốLượngĐặt, GiáĐặt, TổngTiền*
- [ ] **Bước 5:** Vẽ **Star Schema** → Fact ở giữa, Dim xung quanh
- [ ] **Bước 6:** Kiểm tra: Mỗi câu hỏi phân tích có thể trả lời bằng schema này không?

---

# PHẦN 13: DATA WAREHOUSE & DATA MINING

`[📖 Slide Lecture 7&8]`

```
DATA WAREHOUSE                    DATA MINING
(Kho dữ liệu)                    (Khai phá dữ liệu)
                                   
"TỔ CHỨC dữ liệu                 "TÌM KIẾM quy luật
 để dễ phân tích"                  ẩn trong dữ liệu"
                                   
Ví dụ:                            Ví dụ:
"Doanh thu Q1 là                  "Khách mua bánh cốm
 bao nhiêu?"                      thường mua thêm trà sen"
 (Bạn hỏi → DW trả lời)          (DM tự phát hiện)
```

| | Data Warehouse | Data Mining |
|---|---|---|
| **Mục đích** | Lưu trữ & báo cáo | Phát hiện mẫu ẩn |
| **Câu hỏi** | Bạn tự đặt câu hỏi | Máy tự tìm câu trả lời |
| **Kết quả** | Báo cáo, biểu đồ | Quy luật, dự đoán |
| **Ví dụ** | "Bán bao nhiêu?" | "Ai sẽ mua tiếp?" |

> 💡 **Mối quan hệ**: Data Warehouse cung cấp **dữ liệu sạch, có tổ chức** cho Data Mining khai thác. DW là **nền tảng**, DM là **ứng dụng nâng cao**.

---

# PHẦN 14: LỘ TRÌNH HỌC DATA WAREHOUSE

```
📍 Giai đoạn 1: HIỂU CONCEPT (Bạn đang ở đây!)
   ├── DB vs DW
   ├── OLTP vs OLAP
   ├── Fact, Dimension, Star Schema
   ├── Data Cube, OLAP Operations
   └── ETL
       │
       ▼
📍 Giai đoạn 2: THIẾT KẾ SCHEMA
   ├── Phân tích yêu cầu bài tập
   ├── Xác định Fact & Dimensions
   ├── Vẽ Star Schema
   └── Viết CREATE TABLE
       │
       ▼
📍 Giai đoạn 3: CÀI ĐẶT ETL
   ├── Tạo 2 DB nguồn (Oracle + SQL Server)
   ├── Nhập dữ liệu mẫu
   ├── Viết script ETL
   └── Nạp vào Data Warehouse
       │
       ▼
📍 Giai đoạn 4: PHÂN TÍCH OLAP
   ├── Tạo Data Cube
   ├── Thực hiện Roll-up, Drill-down, Slice, Dice
   ├── Trả lời 9 câu hỏi trong đề bài
   └── Tạo báo cáo
```

---

# PHẦN 15: CÂU HỎI THƯỜNG GẶP (FAQ)

**Q: Fact Table có thể có nhiều hơn 1 không?**
> Có! Ví dụ: Fact_Order (đặt hàng) và Fact_Inventory (tồn kho) là 2 sự kiện khác nhau.

**Q: Dimension có bắt buộc phải có Dim_Time không?**
> Gần như bắt buộc. Hầu hết phân tích đều cần chiều thời gian để so sánh xu hướng.

**Q: Star Schema có giống ER Diagram không?**
> **Không!** ER Diagram dùng cho Database (OLTP, chuẩn hóa). Star Schema dùng cho DW (OLAP, phi chuẩn hóa). Đừng nhầm lẫn!

**Q: Tại sao Dim_Customer gộp cả du lịch và bưu điện?**
> Vì trong DW, ta muốn **1 bảng duy nhất** để dễ phân tích. Dùng cột `LoạiKH` để phân biệt thay vì tách thành 2 bảng.

**Q: "Thời gian" trong đề bài (trường Thời gian ở mỗi bảng) là gì?**
> Đó là timestamp ghi nhận khi dữ liệu được tạo/cập nhật, dùng cho ETL để biết dữ liệu nào mới cần đồng bộ.

---

> 🎯 **Bước tiếp theo:** Khi đã hiểu các khái niệm trong tài liệu này, hãy chuyển sang **Giai đoạn 2 – Thiết kế Star Schema** cụ thể cho bài tập, bao gồm viết câu lệnh CREATE TABLE và chuẩn bị dữ liệu mẫu.
