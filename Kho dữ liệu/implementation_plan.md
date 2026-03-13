# 🏗️ KẾ HOẠCH TRIỂN KHAI BÀI TẬP LỚN – KHO DỮ LIỆU CHUỖI CỬA HÀNG

---

## 📋 PHÂN TÍCH ĐỀ BÀI (Không bỏ sót yêu cầu nào)

### Checklist yêu cầu từ đề bài

| # | Yêu cầu đề bài | Mục trong báo cáo | Trạng thái |
|---|----------------|-------------------|------------|
| 1 | Giới thiệu – Mục tiêu và phạm vi | Mục 1 – Viết báo cáo | [ ] |
| 2 | Yêu cầu nghiệp vụ – Đặc tả ứng dụng cho người dùng | Mục 2 – Viết báo cáo | [ ] |
| 3 | Đặc tả chức năng – Đầu vào/đầu ra của kho dữ liệu | Mục 3 – Viết báo cáo | [ ] |
| 4 | Thiết kế kho dữ liệu – Phương pháp luận + Star Schema | Mục 4 – **Thiết kế** | [ ] |
| 5 | Cài đặt khối dữ liệu – Tạo DW + Tải dữ liệu (ETL) | Mục 5 – **Code** | [ ] |
| 6 | Báo cáo OLAP – Nhúng lệnh sinh báo cáo OLAP | Mục 6 – **Code + Demo** | [ ] |
| 7 | Kiểm tra tính đúng đắn – So sánh OLAP vs nguồn gốc | Mục 7 – **Kiểm thử** | [ ] |
| 8 | Kết luận – Tổng kết | Mục 8 – Viết báo cáo | [ ] |

### Yêu cầu kỹ thuật ẩn trong đề bài

| Yêu cầu ẩn | Giải thích |
|------------|-----------|
| **2 DBMS khác nhau** | Oracle + SQL Server bắt buộc |
| **2 CSDL nguồn riêng biệt** | DB "Văn phòng ĐD" (Oracle) + DB "Bán hàng" (SQL Server) |
| **ETL xuyên nền tảng** | Trích xuất từ Oracle + SQL Server → Nạp vào DW |
| **Star Schema** | Đề yêu cầu rõ "lược đồ hình sao" |
| **Chiều thời gian** | Đề yêu cầu rõ "Thiết lập một chiều thời gian" |
| **9 câu truy vấn OLAP** | Mỗi câu cần có lệnh SQL/MDX cụ thể |
| **Roll-up, Drill-down, Slice, Dice** | Đề yêu cầu rõ các thao tác OLAP |
| **Bảng theo chiều** | Dimension tables cho Star Schema |

---

## 🗄️ PHÂN BỔ DATABASE: ORACLE vs SQL SERVER

> ✅ **Đúng rồi!** Đề bài đã chia rõ:

### Oracle Database – "Cơ sở dữ liệu Văn phòng đại diện"

```
┌─────────────────────────────────────────────────────┐
│  ORACLE – DB Văn phòng đại diện                     │
│                                                      │
│  1. KhachHang (MaKH, TenKH, MaThanhPho,            │
│                NgayDatHangDauTien)                    │
│                                                      │
│  2. KhachHangDuLich (*MaKH, HuongDanVienDuLich,    │
│                       ThoiGian)                      │
│                                                      │
│  3. KhachHangBuuDien (*MaKH, DiaChiBuuDien,        │
│                        ThoiGian)                     │
└─────────────────────────────────────────────────────┘
```

### SQL Server Database – "Cơ sở dữ liệu Bán hàng"

```
┌─────────────────────────────────────────────────────┐
│  SQL SERVER – DB Bán hàng                            │
│                                                      │
│  1. VanPhongDaiDien (MaThanhPho, TenThanhPho,      │
│                      DiaChiVP, Bang, ThoiGian)       │
│                                                      │
│  2. CuaHang (MaCuaHang, *MaThanhPho,               │
│              SoDienThoai, ThoiGian)                   │
│                                                      │
│  3. MatHang (MaMH, MoTa, KichCo, TrongLuong,       │
│              Gia, ThoiGian)                           │
│                                                      │
│  4. MatHang_LuuTru (*MaCuaHang, *MaMH,             │
│                     SoLuongTrongKho, ThoiGian)       │
│                                                      │
│  5. DonDatHang (MaDon, NgayDatHang, *MaKH)          │
│                                                      │
│  6. MatHangDuocDat (*MaDon, *MaMH, SoLuongDat,     │
│                     GiaDat, ThoiGian)                │
└─────────────────────────────────────────────────────┘
```

### Đề bài cho thiếu gì không?

> **Đề bài cho ĐỦ các bảng cần tạo.** Dùng nguyên gốc hoàn toàn được.

Lưu ý:

| Vấn đề | Giải pháp |
|--------|-----------|
| `DonDatHang.MaKH` nhưng `KhachHang` nằm ở Oracle | Đây là **điểm kết nối liên DB** – MaKH phải khớp giữa 2 hệ thống |
| Cột `ThoiGian` ở hầu hết các bảng | Dùng để **ETL gia tăng** – biết dữ liệu nào mới cần đồng bộ |
| Không có bảng Dim riêng cho DW | Bạn tự thiết kế Star Schema trong DB Data Warehouse |

---

## 🔗 LIÊN KẾT GIỮA 2 DATABASE

### "Khi có đơn hàng mới, bảng khách hàng có cập nhật không?"

**Trong bài tập:** 2 DB hoạt động **độc lập**, không tự động đồng bộ. **ETL** là cầu nối duy nhất.

```
1. Nhân viên VP nhập khách hàng mới vào Oracle
2. Nhân viên bán hàng tạo đơn hàng trong SQL Server (nhập MaKH có sẵn)
3. ETL chạy → Kéo dữ liệu từ cả 2 → Nạp vào DW
4. Báo cáo OLAP truy vấn DW
```

### SQL Server & Oracle có hỗ trợ Scheduled Jobs không?

| DBMS | Tính năng | Mô tả |
|------|-----------|-------|
| **SQL Server** | **SQL Server Agent Jobs** | ✅ Job chạy định kỳ (giờ/ngày/tuần) |
| **Oracle** | **DBMS_SCHEDULER** | ✅ Job chạy định kỳ |

**Ứng dụng cho bài tập:**
- **Đủ demo:** Viết stored procedure ETL, **chạy thủ công** khi demo
- **Ghi điểm:** Cấu hình SQL Server Agent Job gọi SP ETL theo lịch

---

## 📊 CHIẾN LƯỢC TẠO DỮ LIỆU DEMO

### Nguyên tắc vàng: Dữ liệu phải nhất quán xuyên suốt 2 DB

```
┌─ FILE MASTER (Excel) ──────────────────────────────┐
│                                                      │
│  Thiết kế TOÀN BỘ dữ liệu trong 1 file Excel       │
│  trước, rồi tách ra INSERT vào từng DB               │
│                                                      │
│  Sheet 1: ThanhPho       → SQL Server                │
│  Sheet 2: CuaHang        → SQL Server                │
│  Sheet 3: MatHang        → SQL Server                │
│  Sheet 4: KhachHang      → Oracle                    │
│  Sheet 5: KH_DuLich      → Oracle                    │
│  Sheet 6: KH_BuuDien     → Oracle                    │
│  Sheet 7: TonKho         → SQL Server                │
│  Sheet 8: DonHang        → SQL Server                │
│  Sheet 9: ChiTietDon     → SQL Server                │
│                                                      │
│  → Đảm bảo MaKH, MaThanhPho, MaMH KHỚP nhau!       │
└──────────────────────────────────────────────────────┘
```

### Thứ tự INSERT (quan trọng – do FK)

| Thứ tự | Bảng | DB | Số bản ghi | Ghi chú |
|--------|------|----|------------|---------|
| 1 | VanPhongDaiDien | SQL Server | 5 | Tạo trước (CuaHang phụ thuộc) |
| 2 | MatHang | SQL Server | 15-20 | Đa dạng kích cỡ, giá |
| 3 | CuaHang | SQL Server | 10-12 | Mỗi TP 2-3 cửa hàng |
| 4 | KhachHang | Oracle | 15-20 | MaThanhPho khớp VanPhongDaiDien |
| 5 | KhachHangDuLich | Oracle | ~8 | FK → KhachHang |
| 6 | KhachHangBuuDien | Oracle | ~8 | FK → KhachHang |
| 7 | MatHang_LuuTru | SQL Server | ~50 | Mỗi cửa hàng 5-10 mặt hàng |
| 8 | DonDatHang | SQL Server | 25-30 | MaKH khớp Oracle, trải đều 2024 |
| 9 | MatHangDuocDat | SQL Server | ~80 | Mỗi đơn 1-5 mặt hàng |

> ⚠️ **Câu 9 cần:** Một số KH thuộc **cả 2 loại** (vừa du lịch vừa bưu điện). Ví dụ KH05 có mặt ở cả bảng `KhachHangDuLich` và `KhachHangBuuDien`.

---

## 🏗️ CÁC PHASE TRIỂN KHAI

### PHASE 1: TẠO 2 DATABASE NGUỒN

#### Oracle – Create Tables

```sql
CREATE TABLE KhachHang (
    MaKH        VARCHAR2(10) PRIMARY KEY,
    TenKH       NVARCHAR2(100) NOT NULL,
    MaThanhPho  VARCHAR2(10) NOT NULL,
    NgayDatHangDauTien DATE,
    ThoiGian    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE KhachHangDuLich (
    MaKH               VARCHAR2(10) PRIMARY KEY,
    HuongDanVienDuLich  NVARCHAR2(100),
    ThoiGian            TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (MaKH) REFERENCES KhachHang(MaKH)
);

CREATE TABLE KhachHangBuuDien (
    MaKH            VARCHAR2(10) PRIMARY KEY,
    DiaChiBuuDien   NVARCHAR2(200),
    ThoiGian        TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (MaKH) REFERENCES KhachHang(MaKH)
);
```

#### SQL Server – Create Tables

```sql
CREATE TABLE VanPhongDaiDien (
    MaThanhPho  VARCHAR(10) PRIMARY KEY,
    TenThanhPho NVARCHAR(100) NOT NULL,
    DiaChiVP    NVARCHAR(200),
    Bang        NVARCHAR(50),
    ThoiGian    DATETIME2 DEFAULT GETDATE()
);

CREATE TABLE CuaHang (
    MaCuaHang   VARCHAR(10) PRIMARY KEY,
    MaThanhPho  VARCHAR(10) NOT NULL,
    SoDienThoai VARCHAR(20),
    ThoiGian    DATETIME2 DEFAULT GETDATE(),
    FOREIGN KEY (MaThanhPho) REFERENCES VanPhongDaiDien(MaThanhPho)
);

CREATE TABLE MatHang (
    MaMH        VARCHAR(10) PRIMARY KEY,
    MoTa        NVARCHAR(200),
    KichCo      NVARCHAR(50),
    TrongLuong  NVARCHAR(50),
    Gia         DECIMAL(12,2),
    ThoiGian    DATETIME2 DEFAULT GETDATE()
);

CREATE TABLE MatHang_LuuTru (
    MaCuaHang       VARCHAR(10),
    MaMH            VARCHAR(10),
    SoLuongTrongKho INT,
    ThoiGian        DATETIME2 DEFAULT GETDATE(),
    PRIMARY KEY (MaCuaHang, MaMH),
    FOREIGN KEY (MaCuaHang) REFERENCES CuaHang(MaCuaHang),
    FOREIGN KEY (MaMH) REFERENCES MatHang(MaMH)
);

CREATE TABLE DonDatHang (
    MaDon       VARCHAR(10) PRIMARY KEY,
    NgayDatHang DATE,
    MaKH        VARCHAR(10) NOT NULL
    -- MaKH tham chiếu Oracle, không tạo FK
);

CREATE TABLE MatHangDuocDat (
    MaDon       VARCHAR(10),
    MaMH        VARCHAR(10),
    SoLuongDat  INT,
    GiaDat      DECIMAL(12,2),
    ThoiGian    DATETIME2 DEFAULT GETDATE(),
    PRIMARY KEY (MaDon, MaMH),
    FOREIGN KEY (MaDon) REFERENCES DonDatHang(MaDon),
    FOREIGN KEY (MaMH) REFERENCES MatHang(MaMH)
);
```

---

### PHASE 2: NẠP DỮ LIỆU DEMO

Viết file INSERT cho từng DB riêng. Cấu trúc:

```
📁 scripts/
├── 01_oracle_insert_khachhang.sql
├── 02_oracle_insert_kh_dulich.sql
├── 03_oracle_insert_kh_buudien.sql
├── 04_sqlserver_insert_thanhpho.sql
├── 05_sqlserver_insert_cuahang.sql
├── 06_sqlserver_insert_mathang.sql
├── 07_sqlserver_insert_tonkho.sql
├── 08_sqlserver_insert_donhang.sql
└── 09_sqlserver_insert_chitiet_donhang.sql
```

---

### PHASE 3: THIẾT KẾ & TẠO DATA WAREHOUSE

**DW đặt trên SQL Server** (khác database):

```sql
CREATE DATABASE DataWarehouse_CuaHang;
```

#### Star Schema

```
                         ┌──────────────┐
                         │ Dim_Customer │
                         └──────┬───────┘
                                │
  ┌──────────────┐      ┌──────┴───────┐      ┌──────────────┐
  │  Dim_Store   │──────│  Fact_Sales  │──────│ Dim_Product  │
  └──────────────┘      └──┬────────┬──┘      └──────────────┘
                           │        │
                  ┌────────┘        └────────┐
           ┌──────┴───────┐          ┌───────┴──────┐
           │   Dim_City   │          │   Dim_Time   │
           └──────────────┘          └──────────────┘
```

#### DW Tables (tóm tắt)

| Bảng | Loại | Cột chính |
|------|------|-----------|
| **Dim_Time** | Dimension | TimeKey, FullDate, Day, Month, Quarter, Year |
| **Dim_City** | Dimension | CityKey, MaThanhPho, TenThanhPho, DiaChiVP, Bang |
| **Dim_Store** | Dimension | StoreKey, MaCuaHang, CityKey, SoDienThoai |
| **Dim_Product** | Dimension | ProductKey, MaMH, MoTa, KichCo, TrongLuong, Gia |
| **Dim_Customer** | Dimension | CustomerKey, MaKH, TenKH, LoaiKH, HDV, DiaChi... |
| **Fact_Sales** | Fact | SalesKey, TimeKey, CityKey, StoreKey, ProductKey, CustomerKey, MaDon, SoLuongDat, GiaDat, TongTien, SoLuongTonKho |

> **Dim_Customer.LoaiKH** = `'Du lịch'` / `'Bưu điện'` / `'Cả hai'` – Kết quả Transform ghép 3 bảng Oracle → 1 Dimension.

---

### PHASE 4: ETL

#### Kết nối Oracle từ SQL Server

**Cách 1 (chuyên nghiệp):** Linked Server

```sql
EXEC sp_addlinkedserver @server='ORACLE_VPDD',
    @srvproduct='Oracle', @provider='OraOLEDB.Oracle',
    @datasrc='ORACLE_SID';

-- Truy vấn Oracle qua Linked Server
SELECT * FROM OPENQUERY(ORACLE_VPDD,
    'SELECT * FROM KhachHang');
```

**Cách 2 (đơn giản cho demo):** Export Oracle → CSV → Import SQL Server staging → ETL vào DW.

#### ETL Stored Procedure

```sql
CREATE PROCEDURE sp_ETL_LoadDataWarehouse AS
BEGIN
    -- 1. Load Dim_Time (lịch 2024)
    -- 2. Load Dim_City (từ VanPhongDaiDien)
    -- 3. Load Dim_Store (từ CuaHang)
    -- 4. Load Dim_Product (từ MatHang)
    -- 5. Load Dim_Customer (từ Oracle: KhachHang
    --    + LEFT JOIN KhachHangDuLich + LEFT JOIN KhachHangBuuDien
    --    → Transform: tính LoaiKH)
    -- 6. Load Fact_Sales (JOIN DonDatHang + MatHangDuocDat + Dim keys)
    PRINT 'ETL completed at ' + CONVERT(VARCHAR, GETDATE(), 120);
END;
```

#### Cần quét định kỳ không?

| Mức | Cách | Phù hợp |
|-----|------|---------|
| **Đủ demo** | Chạy `EXEC sp_ETL_LoadDataWarehouse` thủ công | ✅ Bài tập |
| **Ghi điểm** | SQL Server Agent Job chạy SP hàng ngày | ⭐ Ấn tượng |

---

### PHASE 5: 9 BÁO CÁO OLAP

> Đề yêu cầu: *"nhúng các lệnh hoặc thanh lệnh để sinh báo cáo OLAP"*
> → Viết SQL query chạy trực tiếp trên DW, sinh kết quả.

| Câu | Tóm tắt | Thao tác OLAP | Dimensions |
|-----|---------|---------------|------------|
| 1 | Cửa hàng + mặt hàng bán tại đó | Dice | Store, Product, City |
| 2 | Đơn hàng + khách + ngày | Slice | Customer, Time |
| 3 | Cửa hàng bán MH mà KH đặt | Dice | Store, Product, Customer |
| 4 | VP đại diện tồn kho > N | Slice + Filter | City, Product, Store |
| 5 | Chi tiết đơn + cửa hàng bán MH đó | Drill-down | Order, Product, Store, City |
| 6 | TP và bang của KH | Slice | Customer, City |
| 7 | Tồn kho MH X tại TP Y | Slice + Roll-up | Product, Store, City |
| 8 | Toàn bộ chi tiết 1 đơn hàng | Dice | All dimensions |
| 9 | Phân loại KH: du lịch / bưu điện / cả hai | Slice | Customer |

Mỗi câu sẽ có:
- SQL query cụ thể chạy trên DW (Star Schema)
- SQL query tương đương trên DB nguồn (để kiểm tra Phase 6)
- Ghi chú thao tác OLAP nào được sử dụng

---

### PHASE 6: KIỂM TRA TÍNH ĐÚNG ĐẮN

**Phương pháp:** Chạy cùng logic trên **DB nguồn** và **DW**, so sánh kết quả khớp nhau.

```
[DB Nguồn]                        [Data Warehouse]
SELECT d.MaDon, d.MaKH            SELECT f.MaDon, cu.TenKH
FROM DonDatHang d                  FROM Fact_Sales f
                                   JOIN Dim_Customer cu ...

       → So sánh: Số dòng, giá trị phải KHỚP!
```

---

## ⭐ PHƯƠNG ÁN NÂNG CAO (Ghi điểm – Không overkill)

| # | Phương án | Khó | Ấn tượng |
|---|----------|-----|----------|
| 1 | **SQL Server Agent Job** chạy ETL tự động | ⭐⭐ | Hiểu ETL thực tế |
| 2 | **Incremental ETL** – chỉ load dữ liệu mới (dựa cột ThoiGian) | ⭐⭐ | Hiểu ETL gia tăng |
| 3 | **Excel PivotTable** kết nối DW làm Dashboard | ⭐ | Trực quan, dễ demo |
| 4 | **Stored Procedure có tham số** cho 9 câu OLAP | ⭐⭐ | Chuyên nghiệp |
| 5 | **Fact_Inventory** riêng cho tồn kho | ⭐⭐ | Tư duy thiết kế sâu |

> **Đề xuất:** Chọn **1 + 3 + 4** = Agent Job + Excel Dashboard + SP có tham số

---

## 📅 TIMELINE

| Tuần | Công việc | Output |
|------|----------|--------|
| 1 | Thiết kế dữ liệu demo + tạo 2 DB + INSERT | 2 DB hoạt động |
| 2 | Star Schema + DW + ETL | DW có dữ liệu |
| 3 | 9 câu OLAP + kiểm tra đúng đắn | Demo chạy được |
| 4 | Nâng cao + viết báo cáo | Hoàn chỉnh |

---

## 📁 CẤU TRÚC THƯ MỤC

```
📁 BTL/
├── 📁 01_oracle_scripts/
│   ├── create_tables.sql
│   └── insert_data.sql
├── 📁 02_sqlserver_scripts/
│   ├── create_tables.sql
│   └── insert_data.sql
├── 📁 03_datawarehouse/
│   ├── create_star_schema.sql
│   └── etl_procedure.sql
├── 📁 04_olap_queries/
│   ├── query_01.sql ... query_09.sql
│   └── verify_results.sql
├── 📁 05_advanced/
│   ├── agent_job_setup.sql
│   └── incremental_etl.sql
├── 📄 BaoCao_BTL.docx
└── 📄 DuLieuMau.xlsx
```
