# PHÂN TÍCH 6 PHƯƠNG PHÁP DATA MINING TRÊN KHO DỮ LIỆU

## Tổng quan dữ liệu hiện có

| Bảng | Số dòng (ước tính) | Thông tin chính |
|------|---------------------|----------------|
| `FACT_ORDER` | ~47,825 | MaDon, MaKH, MaMH, MaCuaHang, MaThoiGian, KenhBanHang, SoLuongDat, GiaDat, TongTien |
| `FACT_INVENTORY` | ~55,000+ | MaCuaHang, MaMH, MaThoiGian, SoLuongTon |
| `DIM_CUSTOMER` | ~44,474 | TenKH, LoaiKH (Du lich / Buu dien / Ca hai), MaThanhPho, Bang |
| `DIM_PRODUCT` | ~20,506 | MoTa, KichCo, TrongLuong, Gia |
| `DIM_LOCATION` | 130 | TenCuaHang, LoaiCuaHang (Vat ly / Truc tuyen), TenThanhPho, Bang |
| `DIM_TIME` | 1,461 | Ngay, Thang, Quy, Nam, ThuTrongTuan (2015–2018) |

### Đặc điểm quan trọng của dữ liệu:
- Dữ liệu **synthetic** (xào nấu từ dataset Olist thương mại điện tử Brazil)
- Đa số đơn hàng chỉ có **1 sản phẩm** (ít đơn có > 1 item)
- LoaiKH được gán **ngẫu nhiên** bởi script ETL, không phản ánh hành vi thực
- KenhBanHang được **suy ra từ LoaiKH** (không phải dữ liệu gốc)
- Giá sản phẩm, trọng lượng: lấy từ Olist gốc → **tương đối hợp lý**

---

## PHƯƠNG PHÁP 1: PHÂN LOẠI (Classification)

### Ý tưởng áp dụng
> **Bài toán**: Dự đoán **kênh bán hàng** (KenhBanHang) mà khách hàng sẽ sử dụng
> dựa trên đặc điểm khách hàng và sản phẩm.

- **Nhãn (Label)**: `KenhBanHang` = 'Tai cua hang' hoặc 'Truc tuyen'
- **Đặc trưng (Features)**: Bang, TenThanhPho, Gia sản phẩm, TrongLuong, KichCo, Quy, ThuTrongTuan

### Dữ liệu đầu vào (SQL trích xuất)
```sql
SELECT c.Bang, c.TenThanhPho, c.LoaiKH,
       p.Gia, p.TrongLuong,
       t.Quy, t.ThuTrongTuan,
       fo.KenhBanHang   -- nhan can du doan
FROM   FACT_ORDER fo
JOIN   DIM_CUSTOMER c ON fo.MaKH = c.MaKH
JOIN   DIM_PRODUCT p  ON fo.MaMH = p.MaMH
JOIN   DIM_TIME t     ON fo.MaThoiGian = t.MaThoiGian;
```

### Cách thực hiện
1. Export dữ liệu ra CSV
2. Python: dùng `sklearn.tree.DecisionTreeClassifier` hoặc `sklearn.naive_bayes.GaussianNB`
3. Chia 70% train / 30% test
4. Train model → đánh giá accuracy, confusion matrix

### Thuật toán phù hợp
- **Decision Tree** (dễ trực quan hóa, giải thích được)
- **Naive Bayes** (nhanh, hoạt động tốt với categorical data)

### Đánh giá mức độ phù hợp: ⭐⭐⭐ (3/5)

| Ưu điểm | Hạn chế |
|----------|---------|
| Dữ liệu đủ lớn (~47K rows) | KenhBanHang được **suy ra trực tiếp** từ LoaiKH trong ETL |
| Có nhãn rõ ràng (2 class) | → Model sẽ có accuracy rất cao (~99%) nhưng **không có ý nghĩa thực tế** |
| Dễ demo | Vì nhãn = f(LoaiKH), model chỉ cần học 1 rule: LoaiKH → KenhBanHang |

### Giải pháp cải thiện
- **Bỏ LoaiKH** ra khỏi features → buộc model học từ các đặc trưng khác
- Kết quả sẽ accuracy thấp hơn nhưng **hợp lý hơn** cho demo
- Hoặc đổi bài toán: Dự đoán **LoaiKH** dựa trên hành vi mua hàng (tổng chi tiêu, số đơn, sản phẩm...)

### Kết luận
Làm được, demo được, nhưng cần cẩn thận **loại bỏ biến bị leak** (LoaiKH) để kết quả không quá hoàn hảo phi thực tế.

---

## PHƯƠNG PHÁP 2: PHÂN CỤM (Clustering)

### Ý tưởng áp dụng
> **Bài toán**: Phân nhóm khách hàng thành các cụm dựa trên **hành vi mua hàng**
> (RFM Analysis: Recency, Frequency, Monetary)

### Dữ liệu đầu vào (SQL trích xuất)
```sql
-- Tinh RFM cho moi khach hang
SELECT c.MaKH,
       c.LoaiKH,
       c.Bang,
       MAX(fo.MaThoiGian) AS LastPurchase,       -- Recency (ngay mua gan nhat)
       COUNT(DISTINCT fo.MaDon) AS Frequency,     -- Frequency (so don)
       SUM(fo.TongTien) AS Monetary,              -- Monetary (tong chi tieu)
       AVG(fo.GiaDat) AS AvgPrice,                -- Gia trung binh
       COUNT(DISTINCT fo.MaMH) AS UniqueProducts  -- So SP khac nhau
FROM   FACT_ORDER fo
JOIN   DIM_CUSTOMER c ON fo.MaKH = c.MaKH
GROUP BY c.MaKH, c.LoaiKH, c.Bang;
```

### Cách thực hiện
1. Export CSV
2. Chuẩn hóa dữ liệu (StandardScaler)
3. Dùng Elbow Method chọn số cụm K tối ưu
4. Chạy K-Means
5. Trực quan hóa bằng scatter plot (PCA giảm chiều nếu cần)

### Thuật toán phù hợp
- **K-Means** (phổ biến nhất, dễ hiểu)
- K = 3~5 cụm (VD: Khách VIP, Khách trung bình, Khách mua ít)

### Đánh giá mức độ phù hợp: ⭐⭐⭐⭐ (4/5)

| Ưu điểm | Hạn chế |
|----------|---------|
| **Không cần nhãn** → không bị leak như Classification | Dữ liệu synthetic → cụm có thể không phản ánh thực tế |
| ~44K khách hàng → đủ lớn | Frequency thấp (đa số KH chỉ mua 1-2 đơn) |
| RFM là phương pháp chuẩn trong marketing | Cần giải thích ý nghĩa từng cụm |
| Kết quả luôn ra được (unsupervised) | |

### Kết luận
**Rất phù hợp** cho bài tập. Clustering không yêu cầu dữ liệu phải "đúng" — nó luôn cho kết quả, và bạn chỉ cần giải thích ý nghĩa các cụm. Đây là phương pháp an toàn nhất.

---

## PHƯƠNG PHÁP 3: LUẬT KẾT HỢP (Association Rule Discovery)

### Ý tưởng áp dụng
> **Bài toán**: Tìm các mặt hàng thường được **mua cùng nhau** bởi cùng 1 khách hàng
> (Market Basket Analysis)

### Vấn đề chính: Đa số đơn chỉ có 1 sản phẩm
Nếu basket = 1 đơn hàng → Apriori hầu như không tìm được rule nào.

### Giải pháp: Gom theo khách hàng
Basket = **tất cả sản phẩm 1 khách hàng từng mua qua mọi đơn**.

### Dữ liệu đầu vào (SQL trích xuất)
```sql
-- Gio hang = tat ca SP 1 KH tung mua
SELECT MaKH, MaMH
FROM   FACT_ORDER
GROUP BY MaKH, MaMH;

-- Kiem tra: trung binh moi KH mua bao nhieu SP khac nhau?
SELECT AVG(cnt) AS avg_products_per_customer FROM (
    SELECT MaKH, COUNT(DISTINCT MaMH) AS cnt
    FROM FACT_ORDER GROUP BY MaKH
);
```

### Cách thực hiện
1. Export CSV (MaKH, MaMH)
2. Python: `mlxtend.frequent_patterns.apriori`
3. Chọn min_support (VD: 0.005 = 0.5%)
4. Sinh rules với min_confidence (VD: 0.3 = 30%)
5. Sắp xếp theo Lift giảm dần

### Thuật toán phù hợp
- **Apriori** (chuẩn, dễ hiểu)
- **FP-Growth** (nhanh hơn với dataset lớn)

### Đánh giá mức độ phù hợp: ⭐⭐⭐ (3/5)

| Ưu điểm | Hạn chế |
|----------|---------|
| Phương pháp kinh điển, dễ demo | ~20K sản phẩm → support rất thấp cho từng item |
| Gom theo KH giải quyết vấn đề 1 SP/đơn | Sản phẩm được gán **ngẫu nhiên** → rules không có ý nghĩa thực |
| Kết quả dễ giải thích: "Mua A → Mua B" | Có thể ra rất ít rules hoặc rules vô nghĩa |

### Mẹo cho bài tập
- Nếu quá nhiều sản phẩm → **gom theo nhóm sản phẩm** (dùng KichCo hoặc khoảng giá) thay vì MaMH riêng lẻ
- VD: thay vì "MH001 → MH002", ra "Sản phẩm nhỏ giá rẻ → Sản phẩm lớn giá cao"

### Kết luận
Làm được nhưng cần **tinh chỉnh** (gom theo KH, gom nhóm SP). Kết quả có thể hơi "nhạt" vì dữ liệu synthetic.

---

## PHƯƠNG PHÁP 4: HỒI QUY (Regression)

### Ý tưởng áp dụng
> **Bài toán**: Dự đoán **tổng tiền đơn hàng** (TongTien) dựa trên đặc trưng
> sản phẩm, khách hàng, thời gian, địa điểm.

### Dữ liệu đầu vào (SQL trích xuất)
```sql
SELECT p.Gia, p.TrongLuong,
       t.Thang, t.Quy, t.Nam,
       CASE WHEN l.LoaiCuaHang = 'Truc tuyen' THEN 1 ELSE 0 END AS IsOnline,
       fo.SoLuongDat,
       fo.TongTien  -- bien muc tieu (target)
FROM   FACT_ORDER fo
JOIN   DIM_PRODUCT p  ON fo.MaMH = p.MaMH
JOIN   DIM_TIME t     ON fo.MaThoiGian = t.MaThoiGian
JOIN   DIM_LOCATION l ON fo.MaCuaHang = l.MaCuaHang;
```

### Cách thực hiện
1. Export CSV
2. Python: `sklearn.linear_model.LinearRegression` hoặc `RandomForestRegressor`
3. Chia train/test (70/30)
4. Đánh giá: R², MAE, RMSE
5. Trực quan: scatter plot predicted vs actual

### Thuật toán phù hợp
- **Linear Regression** (đơn giản, dễ giải thích)
- **Random Forest Regressor** (chính xác hơn)

### Đánh giá mức độ phù hợp: ⭐⭐ (2/5)

| Ưu điểm | Hạn chế |
|----------|---------|
| Dữ liệu đủ lớn | TongTien = SoLuongDat × GiaDat → **quan hệ tuyến tính hoàn hảo** |
| Dễ implement | Model chỉ cần học phép nhân → **R² ≈ 1.0**, quá hoàn hảo |
| | Không có ý nghĩa khám phá gì mới |

### Giải pháp cải thiện
- **Đổi target**: Dự đoán **tổng chi tiêu của KH trong quý tiếp theo** (aggregate level)
- Hoặc dự đoán **số lượng đơn hàng theo tháng** cho mỗi thành phố

```sql
-- Du lieu cho du doan doanh thu theo thang/thanh pho
SELECT l.MaThanhPho, t.Nam, t.Thang,
       COUNT(DISTINCT fo.MaDon) AS SoDon,
       SUM(fo.TongTien) AS DoanhThu
FROM   FACT_ORDER fo
JOIN   DIM_LOCATION l ON fo.MaCuaHang = l.MaCuaHang
JOIN   DIM_TIME t ON fo.MaThoiGian = t.MaThoiGian
GROUP BY l.MaThanhPho, t.Nam, t.Thang;
```

### Kết luận
Khả thi nhưng **cần thay đổi bài toán** để tránh kết quả quá hoàn hảo. Nếu dùng aggregated data thì kết quả sẽ hợp lý hơn.

---

## PHƯƠNG PHÁP 5: PHÁT HIỆN BẤT THƯỜNG (Anomaly Detection)

### Ý tưởng áp dụng
> **Bài toán**: Phát hiện **đơn hàng bất thường** — đơn hàng có giá trị, số lượng,
> hoặc mẫu mua sắm lệch xa so với bình thường.

### Dữ liệu đầu vào (SQL trích xuất)
```sql
-- Thong ke mua hang cua moi KH
SELECT c.MaKH, c.LoaiKH, c.Bang,
       COUNT(DISTINCT fo.MaDon) AS SoDon,
       SUM(fo.TongTien) AS TongChiTieu,
       AVG(fo.TongTien) AS TBDon,
       MAX(fo.TongTien) AS MaxDon,
       COUNT(DISTINCT fo.MaMH) AS SoSP
FROM   FACT_ORDER fo
JOIN   DIM_CUSTOMER c ON fo.MaKH = c.MaKH
GROUP BY c.MaKH, c.LoaiKH, c.Bang;
```

### Cách thực hiện
1. Export CSV
2. Chuẩn hóa features
3. Dùng Isolation Forest hoặc Z-score
4. Đánh dấu các điểm anomaly (outlier)
5. Phân tích: tại sao những KH/đơn này bất thường?

### Thuật toán phù hợp
- **Isolation Forest** (`sklearn.ensemble.IsolationForest`) — unsupervised, dễ dùng
- **Z-Score** đơn giản cho từng biến
- **LOF** (Local Outlier Factor)

### Đánh giá mức độ phù hợp: ⭐⭐⭐ (3/5)

| Ưu điểm | Hạn chế |
|----------|---------|
| Không cần nhãn (unsupervised) | Dữ liệu synthetic → anomaly có thể do **lỗi sinh dữ liệu**, không phải insight |
| Luôn tìm được vài outlier | Khó đánh giá kết quả đúng/sai (không có ground truth) |
| Dễ giải thích: "KH này chi tiêu gấp 10x trung bình" | |

### Kết luận
Làm được, kết quả dễ trình bày. Nhưng khi thầy hỏi "tại sao bất thường?" thì cần chuẩn bị câu trả lời hợp lý (VD: "KH mua số lượng lớn bất thường, có thể là đơn sỉ").

---

## PHƯƠNG PHÁP 6: MẪU TUẦN TỰ (Sequential Pattern Discovery)

### Ý tưởng áp dụng
> **Bài toán**: Tìm **trình tự mua hàng theo thời gian** — khách hàng mua sản phẩm A
> trước, sau đó thường mua sản phẩm B.

### Dữ liệu đầu vào (SQL trích xuất)
```sql
-- Lich su mua hang theo thu tu thoi gian
SELECT fo.MaKH,
       fo.MaThoiGian,
       fo.MaMH,
       p.KichCo,
       CASE WHEN p.Gia < 100 THEN 'Gia re'
            WHEN p.Gia < 500 THEN 'Gia TB'
            ELSE 'Gia cao' END AS NhomGia,
       ROW_NUMBER() OVER (PARTITION BY fo.MaKH ORDER BY fo.MaThoiGian) AS ThuTuMua
FROM   FACT_ORDER fo
JOIN   DIM_PRODUCT p ON fo.MaMH = p.MaMH
ORDER BY fo.MaKH, fo.MaThoiGian;
```

### Cách thực hiện
1. Export CSV
2. Python: dùng `PrefixSpan` (thư viện `prefixspan`)
3. Mỗi sequence = danh sách sản phẩm 1 KH mua **theo thứ tự thời gian**
4. Tìm frequent subsequences

### Thuật toán phù hợp
- **PrefixSpan** (dễ implement trong Python)
- **GSP** (Generalized Sequential Patterns)

### Đánh giá mức độ phù hợp: ⭐⭐ (2/5)

| Ưu điểm | Hạn chế |
|----------|---------|
| Phương pháp hay, ấn tượng khi demo | Đa số KH chỉ mua **1 đơn** → sequence chỉ có 1 phần tử |
| Kết quả dễ giải thích | Rất ít KH có sequence dài ≥ 2 → kết quả nghèo nàn |
| | Thứ tự mua là **ngẫu nhiên** → patterns không có ý nghĩa |

### Giải pháp cải thiện
- Gom sản phẩm thành **nhóm giá** (Giá rẻ → Giá TB → Giá cao) thay vì MaMH cụ thể
- Hoặc dùng **kênh bán hàng** làm sequence: "Lần 1: Tại cửa hàng → Lần 2: Trực tuyến"
- Lọc chỉ KH có ≥ 2 đơn:
```sql
SELECT MaKH FROM FACT_ORDER GROUP BY MaKH HAVING COUNT(DISTINCT MaDon) >= 2;
```

### Kết luận
Khó nhất trong 6 phương pháp với dữ liệu này. Nếu bắt buộc phải làm thì cần gom nhóm sản phẩm và lọc KH.

---

## BẢNG TỔNG HỢP SO SÁNH

| # | Phương pháp | Phù hợp | Dễ làm | Dễ demo | Rủi ro phi lý | Khuyên dùng |
|---|------------|---------|--------|---------|---------------|-------------|
| 1 | Classification | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⚠ Cao (leak biến) | ✅ Nếu loại bỏ LoaiKH |
| 2 | **Clustering** | **⭐⭐⭐⭐** | **⭐⭐⭐⭐** | **⭐⭐⭐⭐⭐** | **✅ Thấp** | **✅ KHUYÊN DÙNG #1** |
| 3 | Association Rules | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⚠ TB (rules vô nghĩa) | ✅ Nếu gom theo KH |
| 4 | Regression | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⚠ Cao (R²≈1.0) | ⚠ Cần đổi target |
| 5 | Anomaly Detection | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ✅ Thấp | ✅ Dễ demo |
| 6 | Sequential Pattern | ⭐⭐ | ⭐⭐ | ⭐⭐ | ⚠ Cao (ít sequence) | ⚠ Cần tinh chỉnh nhiều |

---

## KHUYẾN NGHỊ THỨ TỰ ƯU TIÊN

### Nếu chỉ chọn 1: → **Clustering (K-Means RFM)**
- An toàn nhất, luôn cho kết quả
- Không yêu cầu nhãn
- Kết quả dễ giải thích: "Nhóm 1 = VIP, Nhóm 2 = Trung bình, Nhóm 3 = Mua ít"

### Nếu chọn 2: → **Clustering + Classification**
- Clustering trước → tạo nhãn cụm
- Classification sau → dùng nhãn cụm làm target, dự đoán KH mới thuộc nhóm nào

### Nếu cần đủ 6: Đều làm được, nhưng cần lưu ý:
- Regression: đổi target sang aggregated level
- Sequential: lọc KH có ≥ 2 đơn, gom nhóm SP
- Classification: loại bỏ LoaiKH khỏi features

---

## VỀ LO LẮNG "DỮ LIỆU KHÔNG CHÍNH XÁC"

Dữ liệu synthetic **không phải vấn đề** cho bài tập trên lớp, vì:

1. **Mục đích**: Thể hiện bạn **biết quy trình** data mining, không phải tạo insight kinh doanh thực
2. **Thầy đánh giá**: Cách tiếp cận, code, giải thích → không đánh giá kết quả đúng/sai
3. **Thực tế**: Ngay cả dữ liệu thật cũng có noise, data mining vẫn chạy được

### Điều cần tránh:
- ❌ Kết quả accuracy = 100% → **quá hoàn hảo**, rõ ràng có vấn đề
- ❌ Nói "Mua tã → Mua bia" nhưng dữ liệu không có tã và bia → **phi lý**
- ✅ Accuracy 70-85% → **hợp lý** cho dữ liệu thực tế
- ✅ Nói "Cụm 1 gồm KH chi tiêu cao, cụm 2 chi tiêu thấp" → **luôn đúng**
