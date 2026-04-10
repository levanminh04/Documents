# 🔧 PLAN TRIỂN KHAI LỚP 1: PHÂN CỤM ĐƠN HÀNG (Clustering)

## Tổng quan

| Hạng mục | Chi tiết |
|----------|---------|
| **Mục tiêu** | Phân cụm ~47,825 đơn hàng thành N nhóm dựa trên đặc tính sản phẩm |
| **Features** | `log(Gia)`, `log(TrongLuong)` |
| **Thuật toán** | K-Means |
| **Chọn K** | Elbow Method + Silhouette Score |
| **Output** | Mỗi đơn hàng được gán nhãn cụm + biểu đồ + bảng đặc trưng |

---

## BƯỚC 1: EXPORT DỮ LIỆU TỪ ORACLE

### Cách 1: SQL Developer (Export CSV thủ công)

Chạy query sau trong SQL Developer, rồi chuột phải → **Export** → chọn CSV:

```sql
SELECT fo.MaDon,
       fo.MaMH,
       p.Gia,
       p.TrongLuong,
       -- Các cột phụ (dùng cho Lớp 1.5 cross-tabulation sau)
       fo.TongTien,
       fo.SoLuongDat,
       fo.KenhBanHang,
       t.Quy,
       t.ThuTrongTuan,
       l.Bang,
       l.LoaiCuaHang,
       l.MaCuaHang
FROM FACT_ORDER fo
JOIN DIM_PRODUCT p  ON fo.MaMH = p.MaMH
JOIN DIM_TIME t     ON fo.MaThoiGian = t.MaThoiGian
JOIN DIM_LOCATION l ON fo.MaCuaHang = l.MaCuaHang
WHERE p.TrongLuong > 0;   -- Loại 3 sản phẩm TrongLuong = 0
```

> [!TIP]
> Export tất cả các cột (bao gồm Quy, ThuTrongTuan, Bang...) ngay từ đầu, dù chỉ dùng Gia + TrongLuong cho clustering. Các cột phụ sẽ dùng cho Lớp 1.5 (cross-tabulation) mà không cần export lại.

**Tên file**: `orders_for_mining.csv`
**Đặt tại**: `d:\PTIT\kì 2 năm 4\Kho dữ liệu và khai phá dữ liệu\BTL\data\`

### Cách 2: Python connect trực tiếp Oracle

```python
import oracledb
import pandas as pd

# Kết nối Oracle (điều chỉnh theo máy bạn)
conn = oracledb.connect(user="your_user", password="your_pass", dsn="localhost/orcl")

query = """
SELECT fo.MaDon, fo.MaMH,
       p.Gia, p.TrongLuong,
       fo.TongTien, fo.SoLuongDat, fo.KenhBanHang,
       t.Quy, t.ThuTrongTuan,
       l.Bang, l.LoaiCuaHang, l.MaCuaHang
FROM FACT_ORDER fo
JOIN DIM_PRODUCT p  ON fo.MaMH = p.MaMH
JOIN DIM_TIME t     ON fo.MaThoiGian = t.MaThoiGian
JOIN DIM_LOCATION l ON fo.MaCuaHang = l.MaCuaHang
WHERE p.TrongLuong > 0
"""

df = pd.read_sql(query, conn)
df.to_csv('data/orders_for_mining.csv', index=False)
conn.close()
print(f"Exported {len(df)} rows")
```

---

## BƯỚC 2: LOAD VÀ XỬ LÝ DỮ LIỆU

```python
import pandas as pd
import numpy as np

# 1. Load CSV
df = pd.read_csv('data/orders_for_mining.csv')
print(f"Tổng số đơn: {len(df)}")

# 2. Kiểm tra nhanh
print(df[['GIA', 'TRONGLUONG']].describe())

# 3. Log-transform (XỬ LÝ QUAN TRỌNG NHẤT)
df['log_Gia'] = np.log1p(df['GIA'])            # log(1 + Gia) để tránh log(0)
df['log_TrongLuong'] = np.log1p(df['TRONGLUONG'])

# 4. Kiểm tra phân phối sau transform
print("\nSau log-transform:")
print(df[['log_Gia', 'log_TrongLuong']].describe())
```

### Tại sao log-transform?

```
TRƯỚC log-transform:                SAU log-transform:
Gia: 1.2 ──────────── 6729          log_Gia: 0.79 ──── 8.81
     ████                                    ████████
     ██  (lệch phải nặng)                    ████████ (gần chuẩn hơn)
     █                                       ██████
                                             ████

K-Means dùng khoảng cách Euclidean → nếu không log, 
1 đơn $6000 sẽ "kéo" centroid mạnh hơn 1000 đơn $50.
Log bào mòn ảnh hưởng của outlier cực đoan.
```

---

## BƯỚC 3: CHUẨN HÓA (StandardScaler)

```python
from sklearn.preprocessing import StandardScaler

# 5. Chuẩn hóa — ĐƯA VỀ CÙNG THANG ĐO (mean=0, std=1)
features = df[['log_Gia', 'log_TrongLuong']].values
scaler = StandardScaler()
features_scaled = scaler.fit_transform(features)
```

### Tại sao chuẩn hóa?

| Trước | Sau StandardScaler |
|-------|-------------------|
| log_Gia: range ~0.8 → 8.8 | ~(-2.5 → 3.5), mean=0, std=1 |
| log_TrongLuong: range ~0 → 10.3 | ~(-2.5 → 3.5), mean=0, std=1 |

Nếu không chuẩn hóa, feature nào có range lớn hơn sẽ **chi phối** khoảng cách Euclidean → K-Means cluster theo feature đó chứ không cân bằng.

---

## BƯỚC 4: CHỌN K TỐI ƯU

### 4.1 Elbow Method

```python
from sklearn.cluster import KMeans
import matplotlib.pyplot as plt

# 6. Chạy K-Means với K từ 2 đến 10
inertias = []
K_range = range(2, 11)

for k in K_range:
    kmeans = KMeans(n_clusters=k, random_state=42, n_init=10)
    kmeans.fit(features_scaled)
    inertias.append(kmeans.inertia_)

# 7. Vẽ biểu đồ Elbow
plt.figure(figsize=(8, 5))
plt.plot(K_range, inertias, 'bo-', linewidth=2, markersize=8)
plt.xlabel('Số cụm (K)', fontsize=12)
plt.ylabel('Inertia (Tổng khoảng cách bình phương)', fontsize=12)
plt.title('Elbow Method — Tìm K tối ưu', fontsize=14)
plt.xticks(K_range)
plt.grid(True, alpha=0.3)
plt.savefig('output/elbow_chart.png', dpi=150, bbox_inches='tight')
plt.show()
```

**Cách đọc biểu đồ Elbow**:
- Trục X = số cụm K
- Trục Y = Inertia (tổng khoảng cách bình phương từ mỗi điểm đến centroid)
- Inertia luôn giảm khi K tăng (nhiều cụm → mỗi cụm chặt hơn)
- **Điểm "khuỷu tay"** = chỗ mà tốc độ giảm chậm lại đột ngột
- VD: Nếu từ K=2→3 giảm mạnh, K=3→4 giảm ít → **K=3 là khuỷu tay**

### 4.2 Silhouette Score (Bổ sung — cho chắc)

```python
from sklearn.metrics import silhouette_score

# 8. Tính Silhouette cho mỗi K
sil_scores = []

for k in K_range:
    kmeans = KMeans(n_clusters=k, random_state=42, n_init=10)
    labels = kmeans.fit_predict(features_scaled)
    sil = silhouette_score(features_scaled, labels)
    sil_scores.append(sil)
    print(f"K={k}: Silhouette = {sil:.4f}")

# 9. Vẽ biểu đồ Silhouette
plt.figure(figsize=(8, 5))
plt.plot(K_range, sil_scores, 'rs-', linewidth=2, markersize=8)
plt.xlabel('Số cụm (K)', fontsize=12)
plt.ylabel('Silhouette Score', fontsize=12)
plt.title('Silhouette Score — K tối ưu là điểm cao nhất', fontsize=14)
plt.xticks(K_range)
plt.grid(True, alpha=0.3)
plt.savefig('output/silhouette_chart.png', dpi=150, bbox_inches='tight')
plt.show()
```

**Cách đọc Silhouette Score**:
- Giá trị từ -1 đến 1
- **Càng cao càng tốt** (cụm chặt, cách biệt rõ)
- Chọn K có Silhouette Score **cao nhất**
- Thường Silhouette cao nhất trùng với Elbow point

### 4.3 Kết hợp 2 phương pháp để quyết định

**Kịch bản thuyết trình**:
> *"Nhóm em sử dụng 2 phương pháp để xác định số cụm:*
> - *Elbow Method cho thấy điểm gãy tại K = \_\_\_*
> - *Silhouette Score cao nhất tại K = \_\_\_*
> - *Kết hợp cả 2, nhóm em chọn K = \_\_\_, đồng thời phù hợp với ý nghĩa kinh doanh: [liệt kê tên cụm]"*

> [!IMPORTANT]
> **Nếu Elbow và Silhouette cho K khác nhau** (VD: Elbow → K=3, Silhouette → K=4): Chọn cái nào **dễ giải thích hơn** về mặt kinh doanh. Thầy không kiểm tra K "đúng" hay "sai" — thầy kiểm tra cách bạn **lập luận** để chọn K.

---

## BƯỚC 5: CHẠY K-MEANS VỚI K ĐÃ CHỌN

```python
# 10. Chạy K-Means chính thức (thay K_BEST bằng số đã chọn)
K_BEST = 3  # Thay bằng kết quả từ Elbow + Silhouette

kmeans_final = KMeans(n_clusters=K_BEST, random_state=42, n_init=10)
df['Cluster'] = kmeans_final.fit_predict(features_scaled)

# 11. Kiểm tra kích thước mỗi cụm
print("\nSố đơn hàng mỗi cụm:")
print(df['Cluster'].value_counts().sort_index())
print()
print("Tỷ lệ % mỗi cụm:")
print((df['Cluster'].value_counts(normalize=True).sort_index() * 100).round(1))
```

---

## BƯỚC 6: TRỰC QUAN HÓA

### 6.1 Scatter Plot 2D (Biểu đồ chính — BẮT BUỘC cho slide)

```python
import matplotlib.pyplot as plt
import matplotlib

# Thiết lập font hỗ trợ tiếng Việt (nếu cần)
plt.rcParams['font.size'] = 12

# 12. Scatter plot — MỖI MÀU LÀ 1 CỤM
colors = ['#e74c3c', '#3498db', '#2ecc71', '#f39c12', '#9b59b6']
cluster_names = {
    0: 'Cụm A',
    1: 'Cụm B', 
    2: 'Cụm C',
    # Thêm nếu K > 3
}

plt.figure(figsize=(10, 7))
for c in range(K_BEST):
    mask = df['Cluster'] == c
    plt.scatter(
        df.loc[mask, 'log_Gia'],
        df.loc[mask, 'log_TrongLuong'],
        c=colors[c],
        label=cluster_names.get(c, f'Cụm {c}'),
        alpha=0.4,
        s=10
    )

# Vẽ centroids
centroids_original = scaler.inverse_transform(kmeans_final.cluster_centers_)
for i, (cx, cy) in enumerate(centroids_original):
    plt.scatter(cx, cy, c=colors[i], marker='X', s=200, 
                edgecolors='black', linewidths=2, zorder=5)

plt.xlabel('log(Giá sản phẩm)', fontsize=13)
plt.ylabel('log(Trọng lượng)', fontsize=13)
plt.title(f'Phân cụm Đơn hàng (K-Means, K={K_BEST})', fontsize=15)
plt.legend(fontsize=11, markerscale=3)
plt.grid(True, alpha=0.2)
plt.savefig('output/scatter_clusters.png', dpi=150, bbox_inches='tight')
plt.show()
```

### 6.2 Bảng thống kê đặc trưng mỗi cụm

```python
# 13. Thống kê mỗi cụm (dùng GIÁ TRỊ GỐC, không phải log)
cluster_stats = df.groupby('Cluster').agg(
    So_don=('MaDon', 'count'),
    Gia_TB=('GIA', 'mean'),
    Gia_Median=('GIA', 'median'),
    Gia_Min=('GIA', 'min'),
    Gia_Max=('GIA', 'max'),
    TL_TB=('TRONGLUONG', 'mean'),
    TL_Median=('TRONGLUONG', 'median'),
).round(1)

print("\n=== ĐẶC TRƯNG TỪNG CỤM ===")
print(cluster_stats.to_string())
```

### 6.3 Bar chart so sánh cụm

```python
# 14. Bar chart — so sánh giá trung bình giữa các cụm
fig, axes = plt.subplots(1, 2, figsize=(12, 5))

# Giá trung bình
cluster_stats['Gia_TB'].plot(kind='bar', ax=axes[0], color=colors[:K_BEST])
axes[0].set_title('Giá trung bình theo cụm')
axes[0].set_xlabel('Cụm')
axes[0].set_ylabel('Giá (đơn vị tiền)')

# Trọng lượng trung bình
cluster_stats['TL_TB'].plot(kind='bar', ax=axes[1], color=colors[:K_BEST])
axes[1].set_title('Trọng lượng trung bình theo cụm')
axes[1].set_xlabel('Cụm')
axes[1].set_ylabel('Trọng lượng (gram)')

plt.tight_layout()
plt.savefig('output/cluster_comparison.png', dpi=150, bbox_inches='tight')
plt.show()
```

---

## BƯỚC 7: GÁN NHÃN KINH DOANH

Sau khi có bảng thống kê, dựa vào **Gia_TB** và **TL_TB** của mỗi cụm để gán nhãn:

```python
# 15. Gán nhãn (điều chỉnh sau khi xem thống kê thực tế)
label_map = {
    0: '🛒 Hàng nhẹ - Giá rẻ (Phụ kiện, tiêu dùng nhanh)',
    1: '📦 Hàng trung bình (Gia dụng, thời trang)',
    2: '🏠 Hàng nặng - Premium (Nội thất, điện máy)',
}

df['Cluster_Label'] = df['Cluster'].map(label_map)
print(df['Cluster_Label'].value_counts())
```

> [!IMPORTANT]
> **Không gán nhãn trước khi xem dữ liệu!** Chạy K-Means trước → xem thống kê → rồi mới đặt tên. Tên cụm phải **phản ánh đúng** đặc trưng số liệu. Nếu cụm 0 có Gia cao nhất thì nó là "Premium", không phải "Hàng rẻ".

---

## BƯỚC 8: EXPORT KẾT QUẢ (để dùng cho Lớp 1.5 và Lớp 2)

```python
# 16. Lưu kết quả ra CSV mới (có cột Cluster)
df.to_csv('data/orders_with_clusters.csv', index=False)
print(f"Đã lưu {len(df)} dòng với cột Cluster vào orders_with_clusters.csv")
```

File `orders_with_clusters.csv` sẽ có tất cả cột gốc + cột `Cluster` → dùng ngay cho:
- **Lớp 1.5**: Cross-tab `Cluster × Quy`, `Cluster × ThuTrongTuan`, `Cluster × Bang`
- **Lớp 2**: Anomaly detection tính Z-score `TongTien` theo từng Cluster

---

## CHECKLIST TRƯỚC KHI CHUYỂN SANG LỚP 1.5

- [ ] Export CSV từ Oracle (47,825 dòng, loại TrongLuong = 0)
- [ ] Load CSV vào Python, kiểm tra shape và null
- [ ] Log-transform Gia và TrongLuong
- [ ] StandardScaler
- [ ] Chạy Elbow Method (K=2→10) → lưu biểu đồ `elbow_chart.png`
- [ ] Chạy Silhouette Score (K=2→10) → lưu biểu đồ `silhouette_chart.png`
- [ ] Quyết định K dựa trên Elbow + Silhouette + ý nghĩa kinh doanh
- [ ] Chạy K-Means chính thức → gán nhãn cụm
- [ ] Trực quan hóa: scatter plot + bar chart → lưu `scatter_clusters.png`, `cluster_comparison.png`
- [ ] In bảng thống kê đặc trưng từng cụm
- [ ] Gán nhãn kinh doanh cho mỗi cụm
- [ ] Export `orders_with_clusters.csv`

---

## LƯU Ý KHI THUYẾT TRÌNH

### Slide Elbow Method
- Đưa biểu đồ Elbow lên
- Vẽ mũi tên/khoanh tròn vào điểm "khuỷu tay"
- Nói: *"Từ K=\_\_\_ trở đi, việc tăng số cụm không giảm đáng kể inertia nữa"*

### Slide Scatter Plot
- Đưa scatter plot 2D lên (log_Gia vs log_TrongLuong)
- Mỗi màu = 1 cụm
- Centroids đánh dấu X lớn
- Nói: *"Sản phẩm trong kho tự nhiên chia thành \_\_\_ nhóm rõ ràng"*

### Slide Đặc trưng cụm
- Bảng hoặc bar chart so sánh Gia_TB, TL_TB giữa các cụm
- Nhãn kinh doanh cho mỗi cụm
- Nói: *"Cụm A chiếm 60% đơn hàng — đây là hàng nhẹ giá rẻ, chiếm phần lớn giao dịch hằng ngày"*
