# 📋 PLAN TỔNG QUÁT — DATA MINING: "Giải mã Cỗ máy Bán hàng"

## 1. BỐI CẢNH DỮ LIỆU

### 1.1 Data Warehouse hiện có

| Bảng | Số dòng | Vai trò |
|------|---------|---------|
| `FACT_ORDER` | ~47,825 | Bảng fact chính — mỗi dòng là 1 line item (MaDon, MaMH) |
| `FACT_INVENTORY` | ~55,000+ | Tồn kho theo cửa hàng × sản phẩm × thời gian |
| `DIM_CUSTOMER` | ~44,474 | Khách hàng (LoaiKH gán ngẫu nhiên) |
| `DIM_PRODUCT` | ~20,506 | Sản phẩm (Gia, TrongLuong, KichCo từ Olist gốc) |
| `DIM_LOCATION` | 130 | Cửa hàng vật lý + kho trực tuyến |
| `DIM_TIME` | 1,461 | Lịch 2015–2018 |

### 1.2 Nguồn gốc dữ liệu

- Dataset gốc: **Olist (thương mại điện tử Brazil)**
- ETL xào nấu thành mô hình chuỗi bán lẻ đa kênh
- **Dữ liệu thật từ Olist**: Giá, Trọng lượng, Ngày đặt hàng, Thành phố/Bang
- **Dữ liệu fake/random**: LoaiKH, KenhBanHang, gán KH→Cửa hàng

---

## 2. KHẢO SÁT DỮ LIỆU ĐÃ THỰC HIỆN

### 2.1 Phân phối khách hàng theo số đơn

```sql
SELECT COUNT(*) FROM (
  SELECT f.MaKH, COUNT(f.MaDon)
  FROM FACT_ORDER f
  GROUP BY f.MaKH
  HAVING COUNT(f.MaDon) > 2
);
-- Kết quả: ~400 khách hàng
```

```sql
SELECT COUNT(*) FROM (
  SELECT f.MaKH, COUNT(f.MaDon)
  FROM FACT_ORDER f
  GROUP BY f.MaKH
  HAVING COUNT(f.MaDon) > 1
);
-- Kết quả: ~2,113 khách hàng
```

| Nhóm KH | Số lượng | Tỷ lệ |
|----------|----------|--------|
| Mua đúng 1 đơn | ~42,361 | **95.2%** |
| Mua 2 đơn | ~1,713 | 3.9% |
| Mua >2 đơn | ~400 | 0.9% |

> [!IMPORTANT]
> **Kết luận**: 95% KH chỉ mua 1 lần → RFM Clustering trên khách hàng không khả thi (Frequency ≈ 1 cho gần như tất cả). Association Rules cũng không khả thi (giỏ hàng quá mỏng).

---

### 2.2 Trọng lượng sản phẩm

```sql
SELECT COUNT(*) AS tong_don,
       COUNT(p.TrongLuong) AS co_trong_luong,
       COUNT(*) - COUNT(p.TrongLuong) AS null_trong_luong,
       ROUND((COUNT(*) - COUNT(p.TrongLuong)) / COUNT(*) * 100, 1) AS pct_null,
       MIN(p.TrongLuong) AS min_tl,
       MAX(p.TrongLuong) AS max_tl,
       ROUND(AVG(p.TrongLuong), 2) AS avg_tl
FROM FACT_ORDER fo
JOIN DIM_PRODUCT p ON fo.MaMH = p.MaMH;
-- Kết quả: 47825 | 47825 | 0 | 0% | 0 | 30000 | 2016.18
```

```sql
SELECT COUNT(*) FROM DIM_PRODUCT WHERE TrongLuong = 0;  -- 3 bản ghi
SELECT COUNT(*) FROM DIM_PRODUCT WHERE TrongLuong < 50; -- 5 bản ghi
```

**Nhận xét**: Không có NULL. 3 sản phẩm TrongLuong = 0 (xử lý bằng cách thay thành 1 hoặc loại bỏ). Range 0–30,000g, avg ~2kg — phân phối hợp lý, lệch phải.

---

### 2.3 Phân phối giá sản phẩm

```sql
SELECT MIN(Gia) AS min_gia, MAX(Gia) AS max_gia,
       ROUND(AVG(Gia), 2) AS avg_gia,
       PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY Gia) AS q1,
       PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY Gia) AS median_gia,
       PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY Gia) AS q3,
       PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY Gia) AS p99
FROM DIM_PRODUCT p
JOIN FACT_ORDER fo ON p.MaMH = fo.MaMH;
-- Kết quả: 1.2 | 6729 | 119.16 | 39.9 | 74.9 | 133.49 | 858.976
```

**Nhận xét**: Lệch phải nặng. 50% đơn nằm trong 39.9–133.49. Outlier cực đoan (max 6,729 vs P99 859). Cần **log-transform**.

---

### 2.4 Phân phối SoLuongDat

```sql
SELECT SoLuongDat, COUNT(*) AS so_don,
       ROUND(COUNT(*) / 47825.0 * 100, 2) AS pct
FROM FACT_ORDER
GROUP BY SoLuongDat
ORDER BY SoLuongDat;
```

| SoLuongDat | Số đơn | % |
|------------|--------|---|
| 1 | 44,336 | **92.70%** |
| 2 | 2,648 | 5.54% |
| 3+ | 841 | 1.76% |

> [!WARNING]
> **Kết luận**: SoLuongDat ≈ 1 cho 92.7% dữ liệu → **loại bỏ khỏi features** (gần như hằng số).

---

### 2.5 Phân phối theo Bang

```sql
SELECT l.Bang, COUNT(*) AS so_don,
       ROUND(COUNT(*) / 47825.0 * 100, 1) AS pct
FROM FACT_ORDER fo
JOIN DIM_LOCATION l ON fo.MaCuaHang = l.MaCuaHang
GROUP BY l.Bang
ORDER BY so_don DESC;
```

| Bang | Số đơn | % |
|------|--------|---|
| SP | 25,856 | **54.1%** |
| RJ | 6,287 | 13.1% |
| MG | 3,802 | 7.9% |
| *(10 bang khác)* | *(còn lại)* | 24.9% |

**Nhận xét**: SP chiếm hơn phân nửa. Top 3 = 75.1%. Không dùng cho clustering (one-hot 13 giá trị lấn át). Dùng cho phân tích mô tả.

---

### 2.6 Phân phối theo Quý

```sql
SELECT t.Quy, COUNT(*) AS so_don,
       ROUND(COUNT(*) / 47825.0 * 100, 1) AS pct
FROM FACT_ORDER fo
JOIN DIM_TIME t ON fo.MaThoiGian = t.MaThoiGian
GROUP BY t.Quy
ORDER BY t.Quy;
```

| Quý | Số đơn | % |
|-----|--------|---|
| Q1 | 12,405 | 25.9% |
| **Q2** | **14,244** | **29.8%** |
| Q3 | 12,479 | 26.1% |
| **Q4** | **8,697** | **18.2%** |

**Nhận xét**: CÓ tính mùa vụ tự nhiên (Q2 >> Q4). Không dùng cho clustering nhưng dùng cho cross-tabulation sau clustering.

---

### 2.7 Phân phối theo Thứ trong tuần

```sql
SELECT t.ThuTrongTuan, COUNT(*) AS so_don,
       ROUND(COUNT(*) / 47825.0 * 100, 1) AS pct
FROM FACT_ORDER fo
JOIN DIM_TIME t ON fo.MaThoiGian = t.MaThoiGian
GROUP BY t.ThuTrongTuan
ORDER BY so_don DESC;
```

| Thứ | Số đơn | % |
|-----|--------|---|
| Monday | 7,820 | 16.4% |
| Tuesday | 7,794 | 16.3% |
| Wednesday | 7,343 | 15.4% |
| Thursday | 7,126 | 14.9% |
| Friday | 6,763 | 14.1% |
| Sunday | 5,823 | 12.2% |
| Saturday | 5,156 | **10.8%** |

**Nhận xét**: CÓ pattern (ngày thường > cuối tuần ~50%). Không dùng cho clustering nhưng dùng cho cross-tabulation.

---

### 2.8 Hoạt động cửa hàng

```sql
SELECT l.MaCuaHang, l.LoaiCuaHang,
       COUNT(DISTINCT t.Nam * 100 + t.Thang) AS so_thang_co_don,
       48 - COUNT(DISTINCT t.Nam * 100 + t.Thang) AS so_thang_trong
FROM FACT_ORDER fo
JOIN DIM_LOCATION l ON fo.MaCuaHang = l.MaCuaHang
JOIN DIM_TIME t ON fo.MaThoiGian = t.MaThoiGian
GROUP BY l.MaCuaHang, l.LoaiCuaHang
ORDER BY so_thang_trong DESC;
```

**Nhận xét**: 130 cửa hàng, mỗi cửa hàng chỉ có đơn hàng trong **17–21 tháng / 48 tháng** (~40% thời gian). Dữ liệu cửa hàng theo tháng rất sparse.

---

### 2.9 KichCo

```sql
SELECT KichCo, COUNT(*) AS so_sp
FROM DIM_PRODUCT
GROUP BY KichCo
ORDER BY so_sp DESC;
-- 7,578 giá trị riêng biệt, max so_sp = 372, min = 1
```

**Nhận xét**: Quá phân tán → **không dùng được**.

---

## 3. BẢNG TỔNG KẾT FEATURES

| Feature | Nguồn gốc | Phương sai | Quyết định | Lý do |
|---------|-----------|-----------|-----------|-------|
| **Gia** | ✅ Olist gốc | ✅ IQR=93.6, range 1.2–6729 | ✅ **GIỮ** (log-transform) | Feature mạnh nhất |
| **TrongLuong** | ✅ Olist gốc | ✅ Range 0–30000g | ✅ **GIỮ** (log-transform) | Feature mạnh thứ 2 |
| TongTien | — | Có | ❌ BỎ | ≈ Gia × 1 (multicollinear) |
| SoLuongDat | ✅ Olist gốc | ❌ 92.7% = 1 | ❌ BỎ | Gần như constant |
| KenhBanHang | ❌ Random | — | ❌ BỎ | Derive từ LoaiKH ngẫu nhiên |
| Bang | ⚠️ ETL assign | SP=54% | ❌ BỎ (cho clustering) | One-hot lấn át; dùng cho mô tả |
| Quy | ✅ Olist gốc | CÓ pattern | ❌ BỎ (cho clustering) | Dùng cho cross-tab sau |
| ThuTrongTuan | ✅ Olist gốc | CÓ pattern | ❌ BỎ (cho clustering) | Dùng cho cross-tab sau |
| KichCo | ✅ Olist gốc | 7578 giá trị | ❌ BỎ | Quá phân tán |

---

## 4. KIẾN TRÚC GIẢI PHÁP — 3 LỚP

```
┌─────────────────────────────────────────────────────────────────┐
│                    LỚP 1: PHÂN CỤM (Clustering)                │
│                                                                 │
│  Input: 47,825 đơn hàng × 2 features (log_Gia, log_TrongLuong) │
│  Thuật toán: K-Means                                            │
│  Output: Mỗi đơn hàng được gán nhãn cụm (A, B, C, ...)        │
│  Tìm K tối ưu: Elbow Method + Silhouette Score                 │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│               LỚP 1.5: PHÂN TÍCH CHÉO (Cross-tabulation)       │
│                                                                 │
│  Mỗi cụm  ×  Quý (Q1-Q4)        → Có mùa vụ khác nhau?        │
│  Mỗi cụm  ×  ThuTrongTuan       → Mua cuối tuần khác nhau?     │
│  Mỗi cụm  ×  Bang               → Phân bố địa lý khác nhau?    │
│  Mỗi cụm  ×  KenhBanHang        → Tỷ lệ online khác nhau?      │
│                                                                 │
│  Không cần thuật toán mới — chỉ groupby + biểu đồ               │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              LỚP 2: PHÁT HIỆN BẤT THƯỜNG (Anomaly Detection)   │
│                                                                 │
│  Góc 1: Đơn hàng bất thường (Z-score trên TongTien THEO CỤM)  │
│  Góc 2: Cửa hàng bất thường (Isolation Forest trên aggregate)  │
│  Liên kết: Ngưỡng bất thường được tính RIÊNG cho mỗi cụm      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. CÂU CHUYỆN KINH DOANH

### Tên đề tài
**"Giải mã Cỗ máy Bán hàng: Phân cụm Sản phẩm và Phát hiện Bất thường trong Chuỗi Bán lẻ Đa kênh"**

### Mạch truyện

1. **Mở**: Công ty bán lẻ đa kênh tại Brazil, doanh thu tăng trưởng nhưng 95% KH chỉ mua 1 lần → không thể phân tích hành vi KH lặp lại → chuyển sang phân tích **cỗ máy bán hàng** (sản phẩm, đơn hàng, cửa hàng)
2. **Thân — Lớp 1**: Phân cụm đơn hàng → phát hiện N phân khúc sản phẩm tự nhiên
3. **Thân — Lớp 1.5**: Phân tích chéo → mỗi phân khúc có hành vi mua sắm theo thời gian/địa lý khác nhau
4. **Thân — Lớp 2**: Phát hiện đơn hàng/cửa hàng bất thường theo từng phân khúc
5. **Kết**: Đề xuất hệ thống giám sát phân tầng — ngưỡng cảnh báo riêng cho mỗi phân khúc sản phẩm

---

## 6. DELIVERABLES

| # | Deliverable | Mô tả | Công cụ |
|---|-------------|-------|---------|
| 1 | File CSV export từ Oracle | Dữ liệu đơn hàng với 2 features | SQL*Plus / SQL Developer |
| 2 | `step1_clustering.py` | Lớp 1: K-Means + Elbow + Silhouette + Scatter plot | Python (sklearn, matplotlib) |
| 3 | `step1b_crosstab.py` | Lớp 1.5: Cross-tabulation theo Quý, Thứ, Bang | Python (pandas, seaborn) |
| 4 | `step2_anomaly.py` | Lớp 2: Z-score theo cụm + Isolation Forest cửa hàng | Python (sklearn) |
| 5 | Slide thuyết trình | 8–10 slides với biểu đồ | PowerPoint / Google Slides |
| 6 | Báo cáo (nếu cần) | Tóm tắt quy trình + insight | Word / PDF |

---

## 7. CÂU HỎI CẦN XÁC NHẬN

> [!IMPORTANT]
> 1. **Bạn export dữ liệu bằng cách nào?** SQL Developer (export ra CSV) hay dùng Python connect trực tiếp Oracle (cx_Oracle/oracledb)?
> 2. **Python environment**: Bạn đã có Python + pip cài sẵn chưa? Có dùng Jupyter Notebook không?
> 3. **Thứ tự triển khai**: Bạn muốn làm Lớp 1 trước (clustering) rồi tôi hướng dẫn tiếp Lớp 1.5 và Lớp 2, hay muốn plan hết cả 3 lớp cùng lúc?
