# PHÂN TÍCH KỸ THUẬT OLAP — TỪ KHỐI RUBIK ĐẾN 9 CÂU TRUY VẤN

---

## PHẦN 1: HÌNH DUNG DATA CUBE TRONG KHÔNG GIAN

### 1.1 Từ bảng 2D lên khối 3D

Bảng SQL thông thường = **tờ giấy phẳng** (2 chiều: dòng × cột).

Data Cube = **khối Rubik nhiều chiều** — mỗi chiều (dimension) là 1 trục.

Với DW của bạn, hãy tưởng tượng **khối lập phương 3 chiều** đơn giản nhất:

```
                        ┌─────────────────────────┐
                       /│  Thang 1  Thang 2 ...   │
                      / │                          │
                     /  │                          │
            Truc Y: /   │   Mỗi Ô = 1 con số     │
           THOI GIAN    │   (SoLuongDat, TongTien) │
                 /      │                          │
                /       │                          │
               /        └─────────────────────────┘
              /        /                          /
             /        /                          /
            ┌────────/──────────────────────────┐
            │  SP1  SP2  SP3  SP4  ... SP20506  │──── Truc X: SAN PHAM
            │                                    │
            │  CH1  ░░░  ░░░  ░░░               │
            │  CH2  ░░░  ░░░  ░░░               │
            │  CH3  ░░░  ░░░  ░░░               │
            │  ...                               │
            │  CH130                             │
            └────────────────────────────────────┘
                          │
                    Truc Z: DIA DIEM (Cua hang)
```

**Mỗi ô nhỏ (░░░)** trong khối = một CON SỐ ĐO (measure):
- Ví dụ: ô tại (CH1, SP5, Thang3) = "Cửa hàng 1 bán sản phẩm 5 trong tháng 3 được bao nhiêu?"
- Giá trị ô = SoLuongDat, TongTien, SoLuongTon...

### 1.2 Vượt qua 3 chiều — tưởng tượng thế nào?

DW có **4 dimension**: Thời gian, Địa điểm, Sản phẩm, Khách hàng.
→ Đó là khối **4 chiều** (hypercube), không vẽ được trên giấy.

**Mẹo tưởng tượng**: Hãy nghĩ mỗi ô trong khối 3D ở trên **chứa thêm 1 khối nhỏ bên trong**:

```
  Khối 3D chính:  Địa điểm × Sản phẩm × Thời gian
                        │
                        ▼
  Mỗi ô bên trong lại tách ra theo: KHÁCH HÀNG
  
  Ô (CH1, SP5, Thang3) → bên trong chứa:
     ├── KH001: mua 2 cái, 500.000đ
     ├── KH002: mua 1 cái, 250.000đ  
     └── KH099: mua 5 cái, 1.250.000đ
```

### 1.3 Phân cấp (Hierarchy) — Mỗi trục có "zoom"

Mỗi dimension không chỉ có 1 mức mà có **nhiều mức zoom** (hierarchy):

```
  THỜI GIAN:    Năm  →  Quý  →  Tháng  →  Ngày  →  Thứ trong tuần
                (zoom out)                          (zoom in)
  
  ĐỊA ĐIỂM:    Bang  →  Thành phố  →  Cửa hàng
  
  KHÁCH HÀNG:   Loại KH  →  Bang  →  Thành phố  →  MaKH cụ thể
  
  SẢN PHẨM:    Khoảng giá  →  Kích cỡ  →  MaMH cụ thể
```

Drill-down = zoom in (phóng to), Roll-up = zoom out (thu nhỏ).

---

## PHẦN 2: 5 PHÉP TOÁN OLAP — THAO TÁC TRÊN KHỐI

### 2.1 SLICE (Cắt lát) — Cố định 1 chiều

**Hình dung**: Dùng dao cắt ngang khối Rubik tại 1 vị trí → được 1 LÁT PHẲNG 2D.

```
  Khối 3D:                         Sau khi SLICE (Thang = 3):
  
     Thoi gian                       Lát phẳng 2D
        ▲                           ┌──────────────┐
        │   ┌───────────┐           │ SP1 SP2 SP3  │
   T3 ──│───│░░░░░░░░░░░│ ← cắt    │              │
        │   │           │    đây    │ CH1 25  30  8 │
   T2 ──│   │           │    ====>  │ CH2 12  0   5 │
        │   │           │           │ CH3  0  18  3 │
   T1 ──│   └───────────┘           └──────────────┘
        └──────────▶ San pham       Địa điểm × Sản phẩm
        Dia diem ↗                  (tại Tháng 3)
```

**Ý nghĩa**: "Cho tôi xem toàn bộ dữ liệu **chỉ của Tháng 3**"
- Cố định Thời gian = Tháng 3
- Kết quả: bảng 2D gồm Địa điểm × Sản phẩm

### 2.2 DICE (Thái lựu / Cắt khối con) — Cố định NHIỀU chiều

**Hình dung**: Dùng dao cắt NHIỀU lát theo NHIỀU chiều → được 1 KHỐI NHỎ hơn.

```
  Khối gốc:                        Sau khi DICE:
                                    (Thang = 1..3 VÀ TP = 'TP01' VÀ Gia < 500)
  ┌─────────────────┐               ┌──────────┐
  │                 │               │          │ ← Khối con nhỏ hơn
  │     ████████    │    ====>      │  ████    │    chỉ chứa dữ liệu
  │     ████████    │               │  ████    │    thỏa 3 điều kiện
  │                 │               └──────────┘
  └─────────────────┘
```

**Ý nghĩa**: "Cho tôi xem dữ liệu **Quý 1, tại TP01, mặt hàng giá < 500**"
- Giới hạn cả 3 chiều cùng lúc → khối con

### 2.3 ROLL-UP (Cuộn lên / Thu gọn) — Zoom OUT

**Hình dung**: Đang xem chi tiết theo **Tháng**, giờ thu gọn lên **Quý**.
Các ô nhỏ (tháng) được GỘP LẠI thành ô lớn (quý) bằng SUM/AVG/COUNT.

```
  Trước (theo Tháng):               Sau Roll-up (theo Quý):
  
  T1: 100   T2: 150   T3: 200      Q1: 450 (= 100+150+200)
  T4: 120   T5: 180   T6: 160      Q2: 460 (= 120+180+160)
  T7:  90   T8: 200   T9: 170      Q3: 460 (=  90+200+170)
  T10: 80  T11: 110  T12: 250      Q4: 440 (=  80+110+250)
```

Hoặc thu gọn trục Địa điểm: **Cửa hàng → Thành phố → Bang**

```
  Trước (theo Cua hang):            Sau Roll-up (theo Thanh pho):
  
  CH1 (TP01): 300                   TP01: 750 (= 300+450)
  CH2 (TP01): 450                   TP02: 200
  CH3 (TP02): 200
```

**Ý nghĩa**: "Tôi không cần biết từng cửa hàng, **gộp theo thành phố** cho tôi"
→ Trong SQL: `GROUP BY TenThanhPho` + `SUM(TongTien)`

### 2.4 DRILL-DOWN (Khoan xuống) — Zoom IN

**Hình dung**: Ngược lại Roll-up. Đang xem theo Quý → tách ra xem theo Tháng.

```
  Đang xem:                         Drill-down:
  
  Q1: 450                           T1: 100
                      ====>         T2: 150
                                    T3: 200
```

**Ý nghĩa**: "Q1 doanh thu cao, **cho tôi xem chi tiết từng tháng** trong Q1"
→ Trong SQL: bỏ GROUP BY Quý, thay bằng GROUP BY Tháng + WHERE Quý = 1

### 2.5 PIVOT (Xoay) — Đổi hàng ↔ cột

**Hình dung**: Xoay khối Rubik 90 độ — chiều nào ở hàng, chiều nào ở cột.

```
  Trước Pivot:                       Sau Pivot:
  (Hàng = Cửa hàng, Cột = Tháng)   (Hàng = Tháng, Cột = Cửa hàng)
  
         T1    T2    T3                    CH1   CH2   CH3
  CH1   100   150   200             T1    100   120    80
  CH2   120   180   160             T2    150   180    90
  CH3    80    90   170             T3    200   160   170
```

**Ý nghĩa**: Cùng 1 bộ dữ liệu, chỉ **đổi góc nhìn** (hàng ↔ cột)
→ Trong SQL: thay đổi cột nào trong SELECT, cột nào trong GROUP BY

---

## PHẦN 3: ÁP DỤNG VÀO 9 CÂU TRUY VẤN

### Bảng tổng hợp nhanh

| Câu | Kỹ thuật OLAP chính | Dimension sử dụng | Hành động trên khối |
|-----|---------------------|-------------------|---------------------|
| 1 | **Projection** (Chiếu mặt) | Location × Product | Nhìn mặt trước khối |
| 2 | **Projection** | Customer × Time | Nhìn mặt bên khối |
| 3 | **Slice** | Location (cố định Customer=X) | Cắt 1 lát theo KH |
| 4 | **Dice** + Filter | Location (filter Product, SoLuong) | Cắt khối con + lọc |
| 5 | **Drill-down** | Customer × Product × Location | Zoom vào chi tiết |
| 6 | **Slice** (cực đoan) | Customer (1 điểm) | Chọn 1 ô duy nhất |
| 7 | **Dice** | Location × Product (cố định cả 2) | Cắt 2 chiều cùng lúc |
| 8 | **Slice** + Drill-down | Tất cả DIM (cố định MaDon) | Cắt lát + zoom chi tiết |
| 9 | **Roll-up** | Customer (gộp theo LoaiKH) | Thu gọn theo nhóm |

---

### CÂU 1: Cửa hàng + mặt hàng lưu kho
**Kỹ thuật: PROJECTION (Chiếu mặt phẳng)**

```
  Khối FACT_INVENTORY 3 chiều:
  
        Thời gian ▲
                  │    ┌─────────────────┐
                  │   /                 /│
                  │  /   MẶT PHẲNG    / │
                  │ /    BẠN NHÌN    /  │  ← Bạn đang nhìn MẶT TRƯỚC
                  │/    ▼▼▼▼▼▼▼    /   │     của khối Rubik
                  ┌─────────────────┐   │
                  │ SP1  SP2 ... SPn│   │
             CH1  │  50   30    12  │   │
             CH2  │  80    0    45  │  /
             CH3  │  25   60     8  │ /
                  └─────────────────┘
                  ──────────────────▶ Sản phẩm
                  │
                  ▼
              Địa điểm (Cửa hàng)
```

**Hành động**: Bạn bỏ qua trục Thời gian (dùng MAX để lấy mới nhất), 
chỉ nhìn vào MẶT PHẲNG 2D: **Địa điểm × Sản phẩm**.

Mỗi ô hiển thị: SoLuongTon + thông tin mô tả từ DIM_LOCATION và DIM_PRODUCT.

**Tại sao 512 dòng cho cuahang1**: Vì trên trục Sản phẩm, cuahang1 có 512 ô 
(512 mặt hàng), mỗi ô = 1 dòng kết quả.

---

### CÂU 2: Đơn hàng + tên KH + ngày đặt hàng
**Kỹ thuật: PROJECTION (Chiếu mặt bên)**

```
  Khối FACT_ORDER:
  
         Thời gian ▲
                   │     ┌──────────────┐
                   │    /              /
              T3 ──│── / ░░░░░░░░░░  / 
                   │  / ░░░░░░░░░░  /
              T2 ──│ / ░░░░░░░░░░  /
                   │/ ░░░░░░░░░░  /
              T1 ──┌──────────────┐
                   │              │
                   └──────────────┘──────▶ Sản phẩm
                  /
                 / ← MẶT BÊN bạn nhìn
                ▼
           Khách hàng
  
  Bạn nhìn MẶT BÊN: Khách hàng × Thời gian
  
         T1      T2      T3
  KH01  Don1    Don5    Don9
  KH02  Don2    Don6     -
  KH03  Don3    Don7    Don10
  KH04   -      Don8     -
```

**Hành động**: Bỏ qua trục Sản phẩm và Địa điểm, 
chỉ nhìn **Khách hàng × Thời gian** → mỗi ô = 1 đơn hàng.

---

### CÂU 3: Cửa hàng bán cho 1 KH cụ thể
**Kỹ thuật: SLICE (Cắt lát)**

```
  Khối 3D:   Khách hàng × Địa điểm × Sản phẩm
  
       KH ▲
          │    ┌────────────────┐
   KH03 ──│────│████████████████│ ← CẮT LÁT tại KH = 'KH03'
          │    │                │
   KH02 ──│    │                │
          │    │                │
   KH01 ──│    └────────────────┘
          └────────────────────▶ SP
         /
        ▼ Địa điểm
  
  Kết quả: 1 LÁT PHẲNG 2D
  ┌────────────────────────────┐
  │  Tất cả cửa hàng mà       │
  │  KH03 đã từng mua hàng    │
  │                            │
  │  CH1 - TP01 - 028xxxx     │
  │  CH5 - TP03 - 036xxxx     │
  └────────────────────────────┘
```

**Hành động**: Cố định chiều Khách hàng = 1 giá trị → dao cắt ngang → 
còn lại mặt phẳng Địa điểm (chỉ lấy thông tin cửa hàng).

---

### CÂU 4: Cửa hàng tồn kho > mức cụ thể
**Kỹ thuật: DICE (Thái lựu) + FILTER trên measure**

```
  Khối FACT_INVENTORY:
  
  DICE = cắt NHIỀU chiều cùng lúc:
  
  Bước 1: Slice theo Sản phẩm = 'MH cụ thể'     (cắt chiều 1)
  Bước 2: Filter SoLuongTon > 100                 (lọc trên measure)
  
       SP ▲
          │    ┌────────────────┐
  MH_X ───│────│▓▓▓▓▓▓▓▓▓▓▓▓▓▓│ ← chỉ lấy MH_X
          │    │                │
          │    └────────────────┘
          └──────────────────▶ Thời gian
         /
        ▼ Địa điểm
  
  Từ lát MH_X, lọc tiếp: chỉ giữ ô có SoLuongTon > 100
  
  Kết quả:
  ┌──────────────────────────────┐
  │  CH2 - TP01 - DiaChiVP...   │  (Ton kho = 150 > 100 ✓)
  │  CH7 - TP03 - DiaChiVP...   │  (Ton kho = 200 > 100 ✓)
  │  (CH1 bị loại vì chỉ = 30)  │
  └──────────────────────────────┘
```

---

### CÂU 5: Chi tiết từng đơn hàng (SP + cửa hàng + TP)
**Kỹ thuật: DRILL-DOWN (Khoan xuống chi tiết nhất)**

```
  Bình thường bạn nhìn ở mức tổng hợp (Roll-up):
  
  ┌─────────────────────────┐
  │ KH01:  5 đơn, 2.5 triệu│    ← MỨC CAO (tổng hợp)
  │ KH02:  3 đơn, 1.8 triệu│
  └─────────────────────────┘
  
  Drill-down ↓↓↓ vào KH01:
  
  ┌─────────────────────────────────────────────────┐
  │ Don1: SP_A (Mo ta X) - CH1 - TP01 - 500.000    │  ← MỨC THẤP
  │ Don1: SP_B (Mo ta Y) - CH1 - TP01 - 300.000    │    (chi tiết nhất)
  │ Don2: SP_C (Mo ta Z) - CH5 - TP03 - 700.000    │
  │ Don3: SP_A (Mo ta X) - CH2 - TP01 - 500.000    │
  │ Don4: SP_D (Mo ta W) - CH5 - TP03 - 250.000    │
  └─────────────────────────────────────────────────┘
```

**Hành động**: KHÔNG gộp (không GROUP BY) → hiển thị MỌI chiều ở mức chi tiết nhất:
Khách hàng cụ thể + Sản phẩm cụ thể + Cửa hàng cụ thể + Thời gian cụ thể.

---

### CÂU 6: TP và bang của 1 KH cụ thể
**Kỹ thuật: SLICE cực đoan (Chọn 1 điểm)**

```
  Chiều Khách hàng (1 trục duy nhất):
  
  ──●──────●──────●──────●──────●──────●──▶
   KH01   KH02   KH03   KH04   KH05  KH06
                    ▲
                    │ SLICE = chọn đúng 1 điểm
                    │
              Kết quả: 1 dòng duy nhất
              TP = 'TP05', Bang = 'California'
```

Đây là trường hợp đặc biệt: chỉ truy vấn 1 DIMENSION TABLE (DIM_CUSTOMER), 
không cần fact table → không có khối, chỉ có 1 trục.

---

### CÂU 7: Tồn kho 1 SP tại tất cả cửa hàng ở 1 TP
**Kỹ thuật: DICE (Cắt khối con 2 chiều)**

```
  Khối FACT_INVENTORY:
  
  Cắt chiều 1: Sản phẩm = 'MH_X'     ─┐
  Cắt chiều 2: Thành phố = 'TP01'     ─┤ DICE
  Giữ nguyên:  Cửa hàng (chi tiết)    ─┘
  
        SP ▲
           │
   MH_X ───│── ░░░░░  ← chỉ lấy hàng MH_X
           │
           └──────────▶ Thời gian
          /
         ▼ Địa điểm
  
  Trong lát MH_X, chỉ giữ cửa hàng ở TP01:
  
  ┌──────────────────────────┐
  │  CH1 (TP01): Ton kho = 50│
  │  CH2 (TP01): Ton kho = 80│
  │  CH4 (TP01): Ton kho = 25│
  │  (CH3 ở TP02 → bị loại) │
  └──────────────────────────┘
```

---

### CÂU 8: Toàn bộ thông tin 1 đơn hàng
**Kỹ thuật: SLICE (cố định MaDon) + DRILL-DOWN (hiện mọi chiều)**

```
  Khối 4 chiều đầy đủ:
  
  Customer × Product × Location × Time
  
  SLICE:   MaDon = '12345'
           → Cắt ra 1 ĐIỂM/VÙNG NHỎ trong không gian
  
  DRILL-DOWN: Hiện chi tiết TẤT CẢ các chiều cho điểm đó
  
  ┌────────────────────────────────────────────────────┐
  │  Don 12345:                                        │
  │    SP: MH_A - "Ao thun cotton" - 2 cai - 250k     │ ← Product dim
  │    KH: KH01 - "Nguyen Van A"                      │ ← Customer dim  
  │    CH: CH5  - "Cua hang 5" - TP03                  │ ← Location dim
  │    Ngay: 15/03/2017                                │ ← Time dim
  └────────────────────────────────────────────────────┘
```

---

### CÂU 9: Phân loại KH theo LoaiKH
**Kỹ thuật: ROLL-UP (Thu gọn theo phân cấp)**

```
  Trước Roll-up (mức KH cụ thể):
  
  KH001 - Du lich    - Nguyen A
  KH002 - Buu dien   - Tran B
  KH003 - Ca hai     - Le C
  KH004 - Du lich    - Pham D
  KH005 - Buu dien   - Hoang E
  ... (44,474 dòng)
  
  
  Roll-up theo LoaiKH:    ▲▲▲ THU GỌN ▲▲▲
  
  ┌─────────────────────────────────────┐
  │  Du lich:   26,684 KH              │  ← Gộp lại
  │  Buu dien:  26,684 KH              │
  │  Ca hai:     8,894 KH (ước tính)   │
  └─────────────────────────────────────┘
  
  Hình dung trên trục:
  
  ──●●●●●●●●──●●●●●●●●──●●●●●──▶  (chi tiết)
    Du lich    Buu dien   Ca hai
  
         Roll-up ▲▲▲
  
  ──────●──────────●────────●───▶  (thu gọn)
      Du lich   Buu dien  Ca hai
      26,684    26,684    8,894
```

---

## PHẦN 4: CÔNG THỨC NHANH — CÁCH NHẬN BIẾT KỸ THUẬT OLAP

Khi đọc 1 yêu cầu truy vấn, hãy tự hỏi 3 câu:

### Câu hỏi 1: "Có CỐ ĐỊNH chiều nào không?"

```
  "... của 1 khách hàng cụ thể"     → CỐ ĐỊNH Customer     → SLICE
  "... tại 1 thành phố cụ thể"      → CỐ ĐỊNH Location     → SLICE
  "... 1 mặt hàng cụ thể"           → CỐ ĐỊNH Product      → SLICE
  
  CỐ ĐỊNH 2+ chiều cùng lúc         → DICE (= nhiều SLICE)
```

### Câu hỏi 2: "Kết quả ở mức CHI TIẾT hay TỔNG HỢP?"

```
  "Liệt kê TỪNG đơn hàng..."        → Chi tiết           → DRILL-DOWN
  "TỔNG doanh thu theo thành phố..." → Tổng hợp           → ROLL-UP
  "ĐẾM số KH theo loại..."          → Tổng hợp           → ROLL-UP
```

### Câu hỏi 3: "Muốn đổi HÀNG ↔ CỘT không?"

```
  "Xem theo tháng (cột) và cửa hàng (hàng)"  → PIVOT
  "Đổi: cột = cửa hàng, hàng = tháng"        → PIVOT
```

### Flowchart tổng hợp:

```
  Yêu cầu truy vấn
       │
       ├── Cố định 1 giá trị trên 1 chiều? ──────→ SLICE
       │
       ├── Cố định giá trị trên 2+ chiều? ───────→ DICE
       │
       ├── Cần xem tổng hợp (SUM, COUNT, AVG)? ──→ ROLL-UP
       │
       ├── Cần xem chi tiết từ tổng hợp? ────────→ DRILL-DOWN  
       │
       ├── Đổi chiều hàng ↔ cột? ────────────────→ PIVOT
       │
       └── Chỉ xem 2 trong N chiều? ─────────────→ PROJECTION
```

---

## PHẦN 5: TỔNG KẾT BẰNG HÌNH ẢNH

```
              ROLL-UP ▲
          (thu gọn: CH → TP → Bang)
                     │
                     │
    SLICE ◄──────── CUBE ────────► DICE
  (cắt 1 chiều)   /  │  \      (cắt nhiều chiều)
                  /   │   \
                 /    │    \
                /     ▼     \
           PIVOT    DRILL-DOWN
        (xoay mặt)  (zoom chi tiết:
                      Bang → TP → CH)
```

**Tóm lại**: Data Cube = khối Rubik nhiều chiều. 
Mỗi phép OLAP = 1 cách THAO TÁC trên khối đó (cắt, xoay, zoom in/out).
SQL chỉ là CÁCH VIẾT để máy tính thực hiện các thao tác đó.
