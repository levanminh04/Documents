# 🔍 BÁO CÁO ĐÁNH GIÁ CHI TIẾT: `etl_olist_to_sql.py`

---

## TÓM TẮT NHANH

| Mức | Số lỗi | Ảnh hưởng |
|-----|--------|-----------|
| 🔴 CRITICAL | 3 | Sai kiến trúc đề bài / Sai kết quả OLAP |
| 🟡 MEDIUM | 4 | Dữ liệu vô lý khi demo |
| 🟢 LOW | 3 | Không ảnh hưởng kết quả, nhưng nên sửa |

---

## 🔴 CRITICAL #1: KIẾN TRÚC SAI – Cả 2 import script đều tạo TẤT CẢ 9 BẢNG

### Vấn đề

Cả `import_oracle.sql` và `import_sqlserver.sql` đều tạo và import **toàn bộ 9 bảng giống hệt nhau**. Đề bài yêu cầu **chia rõ** 2 DB:

```
Đề bài yêu cầu:                    Script hiện tại:
                                    
ORACLE (Văn phòng ĐD):             ORACLE: TẤT CẢ 9 bảng ❌
  - KhachHang                       SQL SERVER: TẤT CẢ 9 bảng ❌
  - KhachHangDuLich                 
  - KhachHangBuuDien               
                                    
SQL SERVER (Bán hàng):             
  - VanPhongDaiDien                
  - CuaHang                        
  - MatHang                        
  - MatHang_LuuTru                 
  - DonDatHang                     
  - MatHangDuocDat                 
```

### Tại sao nghiêm trọng?

- 💥 **Giảng viên sẽ hỏi ngay** tại sao 2 DB giống nhau – mất điểm kiến trúc
- 💥 ETL mất ý nghĩa – không có bước "trích xuất từ 2 nguồn khác nhau"
- 💥 Toàn bộ ý nghĩa của "2 DBMS khác nhau" trong đề bài bị bỏ qua

### Cách sửa

```
ORACLE chỉ tạo + import:           SQL SERVER chỉ tạo + import:
  - KhachHang                        - VanPhongDaiDien
  - KhachHangDuLich (KH_DuLich)      - CuaHang
  - KhachHangBuuDien (KH_BuuDien)    - MatHang
                                      - MatHang_LuuTru
                                      - DonDatHang
                                      - MatHangDuocDat
```

> [!IMPORTANT]
> Cần viết lại `import_oracle.sql` chỉ giữ 3 bảng KH, và `import_sqlserver.sql` chỉ giữ 6 bảng bán hàng.

---

## 🔴 CRITICAL #2: NGUY CƠ DUPLICATE PK – `MatHang_LuuTru`

### Vấn đề (dòng 481-491)

```python
for ma_ch in all_store_ids:
    n_products = random.randint(400, 600)
    store_products = random.sample(all_product_ids, min(n_products, n_total_products))
    for ma_mh in store_products:
        mh_lt_rows.append((ma_ch, ma_mh, so_luong_kho, thoi_gian))
```

`random.sample` **trong mỗi store là unique**, nhưng bước 2 (fallback dòng 496-501) **có thể thêm `(ma_ch, ma_mh)` đã tồn tại**:

```python
# Bước 2 fallback
for ma_mh in sorted(missing):
    ma_ch = random.choice(all_store_ids)  # Có thể trùng!
    mh_lt_rows.append((ma_ch, ma_mh, ...))
```

Nếu `ma_mh` đã có trong store `ma_ch` từ bước 1 → **DUPLICATE PK** → INSERT fail.

### Xác suất xảy ra

Với 100 stores × 400-600 products/store → xác suất trùng thấp nhưng **không phải 0**. Với ~20,000 products, mỗi store lấy ~500 nên coverage rất cao, fallback hiếm khi cần. Tuy nhiên **nếu xảy ra thì SQL INSERT sẽ fail**.

### Cách sửa

```python
# Thay random.choice bằng kiểm tra trùng
existing_pairs = set((row[0], row[1]) for row in mh_lt_rows)
for ma_mh in sorted(missing):
    available_stores = [s for s in all_store_ids if (s, ma_mh) not in existing_pairs]
    ma_ch = random.choice(available_stores)
    ...
```

---

## 🔴 CRITICAL #3: TỒN KHO vs BÁN HÀNG – CON SỐ VÔ LÝ KHI CHẠY OLAP

### Vấn đề cốt lõi

Đề bài nói: *"Sau mỗi lần bán, doanh nghiệp cần biết tổng tồn kho còn lại"*. Nhưng script hiện tại:

| Thông tin | Giá trị trong script | Vấn đề |
|-----------|---------------------|--------|
| `SoLuongKho` (tồn kho) | `random.randint(10, 200)` | Random, không liên quan đến bán |
| `SoLuongDat` (số lượng đặt) | `items_final['price'].count()` | Đếm số dòng Olist (thường = 1) |

**Kết quả khi chạy câu OLAP 7** (tồn kho mặt hàng X tại TP Y):

```
Cửa hàng CH01: Tồn kho Bánh cốm = 156    (random)
Cửa hàng CH02: Tồn kho Bánh cốm = 23     (random)
TỔNG Hà Nội: 179

Nhưng tổng đã bán Bánh cốm ở HN = 2      (vì SoLuongDat gần như luôn = 1)
```

→ Tồn kho cao ngất (10-200) nhưng bán chỉ 1-2 cái → **Trông cực kỳ vô lý** khi demo.

### Tại sao SoLuongDat gần như luôn = 1?

```python
# Dòng 556-559
items_grouped = items_final.groupby(['order_id', 'product_id']).agg(
    so_luong=('price', 'count'),   # ← Đếm SỐ DÒNG, không phải số lượng mua
    gia_dat=('price', 'mean')
)
```

Olist dataset: mỗi dòng `order_items` thường là 1 unit. Nếu KH mua 3 cái giống nhau, Olist có 3 dòng cùng `(order_id, product_id)`. Nhưng thực tế Olist rất hiếm khi có duplicate `(order_id, product_id)` → **SoLuongDat hầu như = 1**.

### Cách sửa

**Phương án A (Khuyến nghị):** Fake `SoLuongDat` hợp lý hơn:

```python
# Thay vì count, dùng random phù hợp
so_luong = random.randint(1, 5)  # Khách mua 1-5 cái/mặt hàng
```

**Phương án B:** Điều chỉnh tồn kho dựa trên tổng đã bán:

```python
# Tính tổng đã bán mỗi sản phẩm tại mỗi store
# Tồn kho = tổng ban đầu - tổng đã bán (đảm bảo > 0)
```

---

## 🟡 MEDIUM #1: TÊN MẶT HÀNG VÔ NGHĨA – `"health_beauty #0001"`

### Vấn đề (dòng 301)

```python
mo_ta = escape_sql(f"{cat} #{category_counter[cat]:04d}")
```

Kết quả: `'health_beauty #0001'`, `'computers_accessories #0003'`, `'unknown #0012'`...

### Tại sao có vấn đề?

- Khi demo câu OLAP 1, 5, 8 → Giảng viên thấy danh sách `"health_beauty #0247"` → **Trông không giống mặt hàng thật**
- Olist category name là tiếng Anh, gạch dưới → Không tự nhiên
- `unknown` xuất hiện khi product không có category → **Vô nghĩa**

### Cách sửa

```python
# Tạo tên sản phẩm tự nhiên hơn
cat_display = cat.replace('_', ' ').title()  # health_beauty → Health Beauty
mo_ta = escape_sql(f"{cat_display} - Model {category_counter[cat]:04d}")
```

Hoặc tạo mapping thủ công cho top categories:

```python
CAT_VN = {
    'health_beauty': 'Mỹ phẩm',
    'computers_accessories': 'Phụ kiện máy tính',
    'housewares': 'Đồ gia dụng', ...
}
```

---

## 🟡 MEDIUM #2: CỘT TÊN KHÔNG KHỚP – ETL output vs Import script

### Vấn đề

| Bảng | ETL output (column name) | Import script (column name) |
|------|--------------------------|----------------------------|
| KH_DuLich | `HuongDanVien` | `HuongDanVienDuLich` |
| MatHang_LuuTru | `SoLuongKho` | `SoLuongTrongKho` |
| DonDatHang | `MaKH` | `MaKhachHang` |

Xem cụ thể:

**ETL dòng 439:**
```python
['MaKH', 'HuongDanVien', 'ThoiGian']
```

**Import SQL Server dòng 55-60:**
```sql
CREATE TABLE KhachHangDuLich (
    MaKH INT PRIMARY KEY,
    HuongDanVienDuLich NVARCHAR(255),  -- ← Khác với 'HuongDanVien'
    ...
);
```

**ETL dòng 548:**
```python
['MaDon', 'NgayDatHang', 'MaKH']
```

**Import SQL Server dòng 79-84:**
```sql
CREATE TABLE DonDatHang (
    MaDon INT PRIMARY KEY,
    NgayDatHang DATE,
    MaKhachHang INT NOT NULL,  -- ← Khác với 'MaKH'
    ...
);
```

### Ảnh hưởng

💥 **INSERT sẽ FAIL** nếu column name trong INSERT statement không khớp CREATE TABLE. Tùy thuộc DBMS có auto-map theo thứ tự hay yêu cầu đúng column name.

### Cách sửa

Đồng bộ 1 trong 2 hướng: sửa ETL hoặc sửa import script.

---

## 🟡 MEDIUM #3: `TrongLuong` – ĐƠN VỊ GRAM, GIÁ TRỊ VÔ LÝ

### Vấn đề (dòng 311-312)

```python
tl = p.get('product_weight_g', 0)
trong_luong = int(tl) if pd.notna(tl) else 0
```

Olist `product_weight_g` có giá trị lên đến **40,000g (40kg)** cho thương mại điện tử. Khi insert: `TrongLuong = 40000`.

### Ảnh hưởng OLAP

Câu 1: *"trọng lượng của tất cả mặt hàng"* → Hiện `40000` không có đơn vị. Giảng viên hỏi: "40000 gì?" → Không rõ.

### Cách sửa

```python
# Lưu đơn vị rõ ràng
if tl >= 1000:
    trong_luong = escape_sql(f"{tl/1000:.1f} kg")
else:
    trong_luong = escape_sql(f"{int(tl)} g")
```

Hoặc nếu `TrongLuong` là FLOAT: giữ gram nhưng thêm comment trong schema.

---

## 🟡 MEDIUM #4: TIMESTAMP – `MatHang_LuuTru.ThoiGian` CỐ ĐỊNH 2015-2016

### Vấn đề (dòng 489)

```python
# ThoiGian cho tồn kho: 2015-01-01 → 2016-09-01
thoi_gian = escape_sql(fake.date_between(
    start_date=date(2015, 1, 1), 
    end_date=date(2016, 9, 1)
).strftime('%Y-%m-%d'))
```

Tồn kho **không bao giờ cập nhật** sau 09/2016, trong khi đơn hàng chạy từ 10/2016 → 08/2018. Theo logic kinh doanh: sau mỗi lần bán, tồn kho phải giảm và timestamp phải cập nhật.

### Ảnh hưởng

- Không phải lỗi nghiêm trọng (tồn kho snapshot ban đầu)
- Nhưng nếu giảng viên hỏi *"Tồn kho có cập nhật theo thời gian không?"* → Timestamps cũ lộ ra

### Cách giải thích khi demo

> "Bảng `MatHang_LuuTru` ghi nhận **snapshot tồn kho ban đầu** khi nhập hàng. Cập nhật tồn kho sau bán hàng là nghiệp vụ của OLTP, không nằm trong scope bài tập DW."

---

## 🟢 LOW #1: `pct` variable không dùng (dòng 482)

```python
pct = random.uniform(0.15, 0.25)  # Tính nhưng không dùng
n_products = random.randint(400, 600)  # Dùng hardcode thay vì pct
```

Code tính `pct` nhưng dùng `random.randint(400, 600)` → `pct` bị dead code.

---

## 🟢 LOW #2: Oracle `ALTER SESSION SET NLS_DATE_FORMAT` chỉ ở file 04

Chỉ file `04_khach_hang.sql` mới có `ALTER SESSION SET NLS_DATE_FORMAT = 'YYYY-MM-DD'` (vì `oracle_mode=True`). Nếu import file 05, 06 riêng lẻ → Date format có thể sai.

---

## 🟢 LOW #3: `Faker('pt_BR')` – Dữ liệu Brazil

```python
fake = Faker('pt_BR')
```

Tên khách hàng, địa chỉ, SĐT đều theo format Brazil. Hợp lý vì dùng Olist (Brazil dataset). Nhưng lưu ý khi demo: giải thích đây là doanh nghiệp tại Brazil.

---

## 📊 ĐÁNH GIÁ ẢNH HƯỞNG LÊN 9 CÂU OLAP

| Câu | Mô tả | Vấn đề dữ liệu? | Chi tiết |
|-----|-------|:---:|--------|
| 1 | Cửa hàng + mặt hàng | ⚠️ | Tên MH vô nghĩa (`health_beauty #0247`) |
| 2 | Đơn hàng + KH + ngày | ✅ | OK – ngày từ Olist gốc, tên KH fake hợp lý |
| 3 | Cửa hàng bán MH cho KH | ⚠️ | Cần JOIN tồn kho ↔ đơn hàng. Logic store assignment random nên **mọi KH đều mua từ store random, không theo TP KH sống** |
| 4 | VP tồn kho > N | ⚠️ | Tồn kho random 10-200: kết quả ra nhưng số liệu không phản ánh thực tế |
| 5 | Chi tiết đơn + cửa hàng | ⚠️ | Không biết đơn hàng fulfil từ cửa hàng nào (thiếu liên kết) |
| 6 | TP + Bang của KH | ✅ | OK |
| 7 | Tồn kho MH X tại TP Y | ⚠️ | Tồn kho random, SoLuongDat~1, tỷ lệ vô lý |
| 8 | Chi tiết đầy đủ đơn hàng | ⚠️ | Thiếu Store dimension cho đơn hàng |
| 9 | Phân loại KH | ✅ | OK – overlap 20% được tạo đúng |

### Vấn đề lớn nhất cho OLAP: THIẾU LIÊN KẾT ĐƠN HÀNG ↔ CỬA HÀNG

Câu 3, 5, 8 đều yêu cầu biết *"cửa hàng nào fulfil đơn hàng"*. Nhưng:
- `DonDatHang` chỉ có `(MaDon, NgayDatHang, MaKH)` – **không có MaCuaHang**
- `MatHangDuocDat` chỉ có `(MaDon, MaMH, SoLuongDat, GiaDat)` – **không có MaCuaHang**

→ Khi xây DW Star Schema, `Fact_Sales` cần `StoreKey` nhưng **không biết lấy từ đâu**.

**Giải pháp:** Khi thiết kế ETL cho DW, tạo logic gán store:

> Với mỗi `(MaDon, MaMH)`: Tìm store cùng TP với KH mà có `MatHang_LuuTru` chứa MH đó → gán store. Nếu không có → tìm store TP khác (đúng nghiệp vụ đề bài).

---

## ✅ ĐIỂM TỐT CỦA SCRIPT

| Aspect | Đánh giá |
|--------|----------|
| **FK Validation** (Bước 6) | ⭐⭐⭐ Rất tốt – kiểm tra toàn bộ FK integrity |
| **Business Constraint check** | ⭐⭐⭐ BC1 (mọi đơn có item), BC2 (mọi product có kho) |
| **Ghost order removal** | ⭐⭐⭐ Loại đơn hàng ma (362 đơn Olist) |
| **Customer dedup** | ⭐⭐⭐ Xử lý đúng customer_unique_id |
| **First order date** | ⭐⭐⭐ Tính MIN ngày đặt hàng đúng |
| **Seed cố định** | ⭐⭐ Reproducible output |
| **Batch INSERT** | ⭐⭐⭐ Multi-row cho SQL Server, single cho Oracle |
| **Output tự xóa khi fail** | ⭐⭐⭐ Tránh import dữ liệu lỗi |

---

## 🎯 ƯU TIÊN SỬA

| # | Vấn đề | Mức | Ưu tiên |
|---|--------|-----|---------|
| 1 | Tách import Oracle/SQL Server đúng đề bài | 🔴 | **P0 – Bắt buộc** |
| 2 | Sửa column name mismatch | 🟡 | **P0 – INSERT sẽ fail** |
| 3 | Tạo logic gán Store cho đơn hàng (cho DW ETL) | 🔴 | **P1 – Cần cho OLAP** |
| 4 | Fake SoLuongDat hợp lý hơn (1-5 thay vì luôn =1) | 🔴 | **P1 – Ảnh hưởng demo** |
| 5 | Sửa tên mặt hàng tự nhiên hơn | 🟡 | P2 |
| 6 | Kiểm tra duplicate PK MatHang_LuuTru | 🔴 | P2 |
| 7 | Đơn vị TrongLuong | 🟡 | P3 |
