# Bản Tư Vấn: Triển Khai Multi-Vector Architecture

## 1. Hiện Trạng Hệ Thống (Vấn đề cần giải quyết)

### Pipeline hiện tại
```
Audio File → extract 22D → Z-Score → L2-Norm → 1 vector duy nhất → Cosine Search
```

### Cấu trúc 22 chiều hiện tại (1 vector duy nhất)
| Vị trí   | Đặc trưng              | Vai trò            |
|----------|------------------------|-------------------|
| [0-9]    | MFCC Mean C1-C10       | Âm sắc (Timbre)   |
| [10-12]  | F0 MIDI × 3            | Cao độ (Pitch)     |
| [13]     | RMS Mean               | Cường độ (Energy)  |
| [14-17]  | Spectral Contrast B1-B4| Kết cấu hài âm    |
| [18-21]  | MFCC Std C1-C4         | Độ ổn định âm sắc  |

### Lỗi kiến trúc đã chứng minh
Khi gộp tất cả vào 1 vector rồi áp L2-Norm, **sự thay đổi của nhóm A (MFCC) sẽ kéo méo nhóm B (Pitch)**, mặc dù chúng đại diện cho các thuộc tính vật lý hoàn toàn độc lập.

> **Ví dụ thực tế:** `Va+S-ord-A#3-ff` (Viola Sordina) và `viola_ord_A#3_ff` (Viola thường) đều có Raw Pitch = 58.00 MIDI, nhưng sau L2-Norm, giá trị Pitch bị biến dạng thành -13 và -10, dẫn đến kết quả tìm kiếm sai hoàn toàn.

---

## 2. Đề Xuất: Chia Thành 2 Vector

> [!IMPORTANT]
> **Nguyên tắc chia:** Mỗi vector phải trả lời **MỘT câu hỏi duy nhất** về âm thanh. Không pha trộn.

### Vector 1: Pitch Vector (3 chiều) — "Nốt nhạc nào?"
| Chiều | Đặc trưng | Ý nghĩa vật lý |
|-------|-----------|-----------------|
| [0]   | F0 MIDI   | Tần số cơ bản → Nốt nhạc (A4=69, C4=60...) |
| [1]   | F0 MIDI   | Bản sao lần 1 (tăng trọng số trong không gian Euclidean) |
| [2]   | F0 MIDI   | Bản sao lần 2 |

**Tại sao lặp 3 lần?** Trong không gian Euclidean, khoảng cách giữa 2 vector 3 chiều `[58,58,58]` và `[60,60,60]` là `√(4+4+4) = 3.46`. Nếu chỉ dùng 1 chiều `[58]` vs `[60]`, khoảng cách chỉ là `2`. Việc lặp 3 lần giúp **khuếch đại sự khác biệt giữa các nốt**, đảm bảo rằng hệ thống không bao giờ nhầm lẫn A#3 với C4 (chênh 2 bán cung).

**Chuẩn hóa:** Chỉ dùng Z-Score. **KHÔNG dùng L2-Norm** cho vector này vì:
- Pitch là giá trị tuyệt đối (nốt A4 luôn là MIDI 69), không cần bất biến tỷ lệ.
- L2-Norm sẽ chiếu mọi pitch lên mặt cầu đơn vị, khiến mọi nốt đều có khoảng cách = 0 (vì `[58,58,58]/||...|| = [0.577,0.577,0.577]` cho MỌI nốt!).

**Thuật toán tìm kiếm:** **Euclidean Distance** (`<->` trong pgvector). Lọc ra Top N file có nốt gần nhất.

### Vector 2: Timbre Vector (19 chiều) — "Nghe như nhạc cụ nào?"
| Vị trí  | Đặc trưng              | Ý nghĩa vật lý | Đóng góp |
|---------|------------------------|-----------------|----------|
| [0-9]   | MFCC Mean C1-C10       | "Dấu vân tay" phổ tần của nhạc cụ | **Cốt lõi** — Phân biệt Violin vs Viola vs Cello |
| [10]    | RMS Mean               | Cường độ trung bình | Phân biệt pp/mf/ff |
| [11-14] | Spectral Contrast B1-B4| Tỷ lệ đỉnh/thung trong 4 dải tần | Phân biệt kỹ thuật (ordinario vs sul_ponticello) |
| [15-18] | MFCC Std C1-C4         | Độ biến thiên của âm sắc theo thời gian | Phân biệt tremolo vs sustain |

**Chuẩn hóa:** Z-Score → L2-Norm. L2-Norm ở đây là **đúng chỗ** vì:
- Tất cả 19 chiều đều mô tả "chất lượng âm thanh" → cùng bản chất → chia tỷ lệ là hợp lý.
- Cosine Similarity sẽ so sánh "hình dáng" phổ tần, không bị ảnh hưởng bởi âm lượng tổng.

**Thuật toán tìm kiếm:** **Cosine Similarity** (`<=>` trong pgvector). Xếp hạng theo độ giống nhau về âm sắc.

---

## 3. Có Nên Bổ Sung Thêm Chiều Không?

> [!WARNING]
> **Câu trả lời: KHÔNG nên bổ sung thêm vào lúc này.**

### Lý do:
1. **Mỗi chiều phải giải thích được.** Giảng viên hỏi "Spectral Contrast Band 3 là gì?" — bạn phải trả lời ngay được. Thêm những chiều mới mà bạn chưa hiểu sâu (ví dụ: Spectral Centroid, Zero Crossing Rate) sẽ tạo ra "lỗ hổng phòng thủ" khi vấn đáp.

2. **19 chiều Timbre đã đủ mạnh.** Bộ dataset Strings mới có 14 kỹ thuật chơi khác nhau, nhưng sự khác biệt giữa chúng (ordinario vs tremolo vs pizzicato) đã được mã hóa tốt trong MFCC Mean (hình dáng phổ tần) và MFCC Std (độ biến thiên). Thêm chiều chỉ tăng "noise dimension" mà không tăng khả năng phân biệt.

3. **Nguyên tắc Occam's Razor trong học thuật:** Một hệ thống 22 chiều mà giải thích được TẤT CẢ từng chiều sẽ được đánh giá cao hơn một hệ thống 50 chiều mà giải thích nửa vời.

### Ngoại lệ (chỉ bổ sung nếu giảng viên yêu cầu):
- **Spectral Centroid** (1 chiều): "Độ sáng" của âm thanh. Dễ giải thích: Violin sáng → centroid cao, Contrabass tối → centroid thấp. Nếu thêm, đặt vào **Timbre Vector**.

---

## 4. Thuật Toán Tìm Kiếm: Filter-and-Rank

### Luồng xử lý mới (2 giai đoạn)

```
┌─────────────────────────────────────────────────────────┐
│                    STAGE 1: PITCH FILTER                │
│                                                         │
│  Query Pitch [58, 58, 58]                               │
│       ↓                                                 │
│  Euclidean Distance trên pitch_vector                   │
│       ↓                                                 │
│  Lọc ra Top 50 file có nốt gần nhất                    │
│  (Đảm bảo 100% các file trả về đều cùng nốt hoặc      │
│   lân cận ±1 bán cung)                                  │
└───────────────────────┬─────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│                   STAGE 2: TIMBRE RANK                  │
│                                                         │
│  Trong 50 file từ Stage 1:                              │
│       ↓                                                 │
│  Cosine Similarity trên timbre_vector                   │
│       ↓                                                 │
│  Sắp xếp theo độ giống nhau về âm sắc                  │
│       ↓                                                 │
│  Trả về Top 6 kết quả cuối cùng                        │
└─────────────────────────────────────────────────────────┘
```

### Tại sao ưu việt hơn hệ thống cũ?

| Tiêu chí | Hệ thống cũ (Single Vector) | Hệ thống mới (Multi-Vector) |
|----------|----------------------------|----------------------------|
| Pitch bị méo? | ✗ Bị L2-Norm bóp méo | ✓ Pitch có vector riêng, không bị ảnh hưởng |
| Sordina tìm đúng? | ✗ MFCC đổi → Pitch bị kéo theo | ✓ MFCC chỉ ảnh hưởng Stage 2, Stage 1 vẫn đúng nốt |
| Giải thích được? | Khó — phải giải thích tại sao cùng nốt mà similarity thấp | Dễ — "Stage 1 lọc đúng nốt, Stage 2 xếp hạng theo âm sắc" |

---

## 5. Thay Đổi Database

### Schema mới (Migration 015)
```sql
-- 1. Thêm 2 cột vector mới
ALTER TABLE audio_files ADD COLUMN pitch_vector vector(3);
ALTER TABLE audio_files ADD COLUMN timbre_vector vector(19);

-- 2. Tạo Index riêng cho từng vector
-- Pitch: Euclidean Distance (L2)
CREATE INDEX idx_pitch_vector ON audio_files
  USING hnsw (pitch_vector vector_l2_ops)
  WITH (m = 16, ef_construction = 64);

-- Timbre: Cosine Similarity
CREATE INDEX idx_timbre_vector ON audio_files
  USING hnsw (timbre_vector vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);
```

### Bảng scaler_params mới
Cần 2 bộ scaler riêng:
- `scaler_params` version=10 → mean/std cho **pitch_vector** (3 chiều)
- `scaler_params` version=11 → mean/std cho **timbre_vector** (19 chiều)

> [!NOTE]
> Cột `feature_vector` (22D cũ) **KHÔNG cần xóa**. Giữ lại để so sánh A/B giữa hệ thống cũ và mới khi demo cho giảng viên.

---

## 6. Tóm Tắt Các File Cần Sửa

| File | Thay đổi |
|------|---------|
| `config.py` | Thêm hằng số `PITCH_DIM = 3`, `TIMBRE_DIM = 19` |
| `extractor.py` | Trả về 2 vector riêng biệt thay vì 1 |
| `normalizer.py` | Tách thành `normalize_pitch()` (chỉ Z-Score) và `normalize_timbre()` (Z-Score + L2) |
| `similarity.py` | Viết lại thành 2-stage: Filter pitch → Rank timbre |
| `main.py` | Cập nhật API response trả về cả pitch_vector và timbre_vector |
| Migration SQL | Thêm 2 cột + 2 index |
| `fit_scaler.py` | Tính scaler riêng cho pitch và timbre |
| `extract_all.py` | Lưu 2 vector vào 2 cột |
| `frontend/app_v3.js` | Cập nhật Raw Inspector hiển thị 2 nhóm riêng biệt |

---

## 7. Bảng Giải Thích Ý Nghĩa Từng Chiều (Chuẩn Bị Cho Vấn Đáp)

### Pitch Vector (3 chiều)
| Chiều | Tên | Ý nghĩa | Ví dụ |
|-------|-----|---------|-------|
| 0-2 | F0 MIDI ×3 | Tần số cơ bản của nốt nhạc, chuyển sang thang MIDI (logarithmic). Lặp 3 lần để tăng trọng số khi tính khoảng cách Euclidean | A4 = MIDI 69 = 440Hz. Chênh 1 bán cung = chênh 1 MIDI |

**Câu hỏi giảng viên có thể hỏi:**
- *"Tại sao dùng MIDI mà không dùng Hz?"* → Hz là thang tuyến tính, khoảng cách giữa C4 (262Hz) và C5 (523Hz) gấp đôi khoảng cách C3→C4, dù tai người nghe chúng "xa bằng nhau" (đều 1 octave). MIDI là thang logarithmic phản ánh đúng cảm nhận thính giác.
- *"Tại sao lặp 3 lần?"* → Khuếch đại trọng số Pitch trong không gian Euclidean 3D.

### Timbre Vector (19 chiều)
| Chiều | Tên | Ý nghĩa | Đóng góp |
|-------|-----|---------|----------|
| 0 | MFCC Mean C1 | Năng lượng tổng thể của phổ tần trên thang Mel | Phân biệt nhạc cụ to/nhỏ |
| 1 | MFCC Mean C2 | Tỷ lệ năng lượng tần thấp vs tần cao | Violin (sáng, C2 dương) vs Contrabass (tối, C2 âm) |
| 2 | MFCC Mean C3 | Độ phẳng/nhọn của phổ tần | Phân biệt ordinario (phẳng) vs sul_ponticello (nhọn) |
| 3-9 | MFCC Mean C4-C10 | Chi tiết ngày càng mịn của "dấu vân tay" phổ tần | Formant, cộng hưởng thân đàn, chất liệu dây |
| 10 | RMS Mean | Biên độ trung bình bình phương (cường độ) | pp ≈ 0.005, ff ≈ 0.05 |
| 11 | Spectral Contrast B1 | Chênh lệch đỉnh/thung trong dải 0-200Hz | Nền tảng bass của nhạc cụ |
| 12 | Spectral Contrast B2 | Chênh lệch đỉnh/thung trong dải 200-800Hz | Vùng cộng hưởng chính |
| 13 | Spectral Contrast B3 | Chênh lệch đỉnh/thung trong dải 800-3200Hz | Vùng "sáng" của âm thanh |
| 14 | Spectral Contrast B4 | Chênh lệch đỉnh/thung trong dải 3200-8000Hz | Harmonic bậc cao, tiếng miết vĩ |
| 15 | MFCC Std C1 | Độ biến thiên năng lượng tổng theo thời gian | Tremolo (std cao) vs sustain (std thấp) |
| 16 | MFCC Std C2 | Độ biến thiên tỷ lệ sáng/tối | Phân biệt col_legno (bất ổn) vs ordinario (ổn định) |
| 17 | MFCC Std C3 | Độ biến thiên hình dáng phổ | Kỹ thuật chơi có giao động hay không |
| 18 | MFCC Std C4 | Độ biến thiên chi tiết phổ | Vibrato (std cao) vs non-vibrato |
