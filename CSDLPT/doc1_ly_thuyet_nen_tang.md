# 📚 Tài liệu 1: Lý thuyết Nền tảng — Nhập môn Đặc trưng Đa phương tiện

> **Dành cho ai?** Người chưa có nền tảng xử lý tín hiệu. Tài liệu này giải thích từ số không, có ví dụ số cụ thể, tránh toán học nặng nề.
> 
> **Mục tiêu:** Sau khi đọc xong, bạn hiểu *tại sao* lại trích xuất các đặc trưng này, *chúng là gì* dưới dạng con số, và *dùng ở đâu* trong dự án CSDL đa phương tiện.

---

## Mục lục

1. [Âm thanh là gì dưới mắt máy tính?](#1-âm-thanh-là-gì-dưới-mắt-máy-tính)
2. [Miền thời gian (Time Domain) — "Nhìn theo thời gian"](#2-miền-thời-gian)
3. [Miền tần số (Frequency Domain) — "Nhìn theo thành phần"](#3-miền-tần-số)
4. [Đặc trưng miền thời gian — Tự code được!](#4-đặc-trưng-miền-thời-gian)
5. [Đặc trưng miền tần số — Dùng thư viện](#5-đặc-trưng-miền-tần-số)
6. [MFCC — Vua của đặc trưng âm thanh](#6-mfcc)
7. [Đặc trưng ảnh (Bonus)](#7-đặc-trưng-ảnh)
8. [Tổng kết: Feature Vector là gì?](#8-tổng-kết)
9. [Tham chiếu slide bài giảng](#9-tham-chiếu-slide)

---

## 1. Âm thanh là gì dưới mắt máy tính?

### 1.1 Âm thanh = Mảng số

Âm thanh trong thực tế là **sóng áp suất không khí** dao động. Micro thu lại sự dao động đó và **chuyển thành dãy số**. Mỗi số đại diện cho "độ cao" của sóng âm tại một thời điểm.

**Ví dụ cực kỳ đơn giản:**

Hãy tưởng tượng bạn thu âm tiếng "A" trong 0.001 giây. Kết quả là một mảng số kiểu như:

```
[0.0, 0.12, 0.35, 0.58, 0.72, 0.81, 0.72, 0.58, 0.35, 0.12, 0.0, -0.12, -0.35, ...]
```

Mỗi số nằm trong khoảng **[-1.0, +1.0]**, thể hiện biên độ (độ lớn) của sóng âm.

### 1.2 Sample Rate (Tần số lấy mẫu)

**Sample rate** = Máy tính lấy bao nhiêu mẫu (số) trong 1 giây.

| Sample Rate | Ý nghĩa | Ứng dụng |
|---|---|---|
| **8,000 Hz** | 8,000 mẫu/giây | Điện thoại cũ |
| **22,050 Hz** | 22,050 mẫu/giây | Music (chất lượng th thường) |
| **44,100 Hz** | 44,100 mẫu/giây | CD âm nhạc chuẩn |
| **48,000 Hz** | 48,000 mẫu/giây | Studio recording |

**Ví dụ:**
- File nhạc MP3 3 phút, sample rate = 44,100 Hz
- Số mẫu = 3 × 60 × 44,100 = **7,938,000 mẫu**
- Đây là một mảng Python có 7.9 triệu phần tử số thực!

```python
import librosa
y, sr = librosa.load("bai_nhac.mp3")
# y là mảng numpy, sr là sample rate
print(y.shape)   # (7938000,) - 7.9 triệu số
print(sr)        # 22050 (librosa mặc định resamples về 22050)
print(y[:5])     # [-0.002, 0.015, 0.031, 0.028, -0.003]
```

---

## 2. Miền thời gian

### 2.1 Giải thích bằng hình ảnh

**Miền thời gian** = Bạn vẽ đồ thị biên độ âm thanh theo trục thời gian.
- Trục X = Thời gian (giây)
- Trục Y = Biên độ (từ -1 đến +1)

```
Biên độ
  1.0 |      /\    /\          /\  /\
  0.5 |    /    \/    \      /    \/  \
  0.0 |---/------------------------\---→ Thời gian
 -0.5 |                    /    \
 -1.0 |                  /      \/
      0s    0.5s    1.0s    1.5s    2.0s
```

Đây là cái nhìn **thô và trực tiếp nhất** về âm thanh. Bạn có thể thấy:
- Chỗ nào **to** (biên độ cao) → tiếng ồn, giọng hát mạnh
- Chỗ nào **nhỏ/im lặng** (biên độ ≈ 0) → khoảng lặng giữa câu

### 2.2 So sánh trực quan

| | Miền thời gian | Miền tần số |
|---|---|---|
| **Câu hỏi trả lời** | Âm thanh TO hay NHỎ ở thời điểm nào? | Âm thanh gồm những ÂM VỰC nào? |
| **Thấy được** | Biên độ theo thời gian | Tần số nào có mặt và mạnh đến đâu |
| **Ví dụ** | "Giây thứ 2 rất to" | "Bài nhạc có nhiều tiếng bass (tần số thấp)" |
| **Tự code được?** | ✅ Dễ | ❌ Cần FFT |

---

## 3. Miền tần số

### 3.1 Tần số là gì?

**Tần số (Frequency)** = Số lần dao động trong 1 giây, đơn vị **Hz (Hertz)**.

**Liên hệ với âm nhạc:**

| Tần số | Nghe thấy gì | Ví dụ trong cuộc sống |
|---|---|---|
| **20 - 250 Hz** | Âm trầm (Bass) | Tiếng trống bass, tiếng sấm |
| **250 - 2,000 Hz** | Âm trung | Giọng người nói, đàn guitar |
| **2,000 - 8,000 Hz** | Âm cao | Tiếng sáo, tiếng chim |
| **8,000 - 20,000 Hz** | Âm rất cao (Treble) | Tiếng kim loại chạm nhau |
| **> 20,000 Hz** | Siêu âm | Tai người không nghe được |

**Nốt nhạc và tần số:**

| Nốt nhạc | Tần số |
|---|---|
| Đô (C4) | 261.6 Hz |
| Rê (D4) | 293.7 Hz |
| Mi (E4) | 329.6 Hz |
| Sol (G4) | 392.0 Hz |
| La (A4) | **440 Hz** (tiêu chuẩn) |

### 3.2 FFT — "Máy soi" tần số

**FFT (Fast Fourier Transform)** = Thuật toán "phân tích" âm thanh ra thành từng thành phần tần số.

**Phép so sánh dễ hiểu:**
> Giả sử bạn pha một ly nước chanh muối đường. FFT giống như một máy phân tích hoá học giúp bạn biết: có bao nhiêu % chanh, bao nhiêu % muối, bao nhiêu % đường. Với âm thanh, FFT cho biết: có bao nhiêu tần số 100Hz, bao nhiêu tần số 1000Hz, v.v.

**Ví dụ với con số:**

Một đoạn âm thanh 0.1 giây (2205 mẫu ở 22050Hz), sau khi FFT cho ra:

```
Tần số (Hz)  | Cường độ (dB) | Giải thích
-------------|---------------|---------------------------
50 Hz        | -60 dB        | Gần như không có
100 Hz       | -20 dB        | Có ít tiếng bass
250 Hz       | -5 dB         | Có nhiều tiếng trung-trầm
440 Hz       | 0 dB          | ĐỈNH → Nốt La đang vang lên!
880 Hz       | -8 dB         | Harmonics (âm bội) của nốt La
1760 Hz      | -15 dB        | Harmonics bậc 2
```

*dB (decibel): 0 dB = mạnh nhất, -60 dB = gần như im lặng. Giá trị âm = nhỏ hơn ngưỡng tham chiếu.*

### 3.3 Spectrogram — Hình ảnh hoá âm thanh

**Spectrogram** = Chạy FFT liên tục theo từng khung thời gian ngắn (window), ghép lại thành hình ảnh.

```
Tần số
(Hz)
8000 |  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  ← im lặng vùng cao
4000 |  ░░░▓▓░░▓▓▓░░░░▓▓░░░░▓▓▓░░░░░░  ← có tiếng ở vùng này đôi khi
1000 |  ▓▓▓▓▓▓▓▓▓▓▓▓▓░░░▓▓▓▓▓▓▓▓▓▓▓▓  ← giọng nói chủ yếu ở đây
 250 |  ████████████░░░████████████████  ← bass mạnh liên tục
  0  |──────────────────────────────────→ Thời gian (giây)
     0s               1s              2s
      ▓ = mạnh   ░ = yếu   khoảng trắng = im lặng
```

---

## 4. Đặc trưng miền thời gian

> 📌 **Tham chiếu slide:** *Slide 10 - Chỉ số hóa và truy vấn dữ liệu âm thanh*

Đây là các đặc trưng **bạn có thể tự code** bằng Python thuần, không cần thư viện tín hiệu số.

### 4.1 Short-time Energy (Năng lượng ngắn hạn)

**Ý nghĩa:** Đo xem âm thanh **to hay nhỏ** trong từng khoảng thời gian ngắn.

**Công thức:**
$$E = \frac{1}{N} \sum_{n=0}^{N-1} x(n)^2$$

*Trong đó: x(n) là mẫu thứ n, N là tổng số mẫu trong đoạn*

**Ví dụ tính tay:**

Giả sử đoạn âm thanh 8 mẫu: `[0.1, 0.5, 0.8, 0.9, 0.8, 0.5, 0.1, 0.0]`

```
Bước 1: Bình phương từng mẫu
x²:  [0.01, 0.25, 0.64, 0.81, 0.64, 0.25, 0.01, 0.00]

Bước 2: Tổng
Σx² = 0.01 + 0.25 + 0.64 + 0.81 + 0.64 + 0.25 + 0.01 + 0.00 = 2.61

Bước 3: Chia N
E = 2.61 / 8 = 0.326
```

**Giải thích kết quả:**
- E ≈ 0.0 : đoạn im lặng hoàn toàn
- E ≈ 0.33 : đoạn âm thanh vừa (như ví dụ trên)
- E ≈ 1.0 : đoạn âm thanh rất to (biên độ bão hòa)

**Tự code Python:**

```python
def compute_energy(samples):
    """Tính năng lượng trung bình của đoạn âm thanh"""
    N = len(samples)
    energy = sum(x**2 for x in samples) / N
    return energy

# Ví dụ
samples = [0.1, 0.5, 0.8, 0.9, 0.8, 0.5, 0.1, 0.0]
print(compute_energy(samples))  # → 0.326
```

**Ứng dụng trong dự án:** Phân biệt nhạc sôi động (energy cao) với nhạc nhẹ nhàng (energy thấp).

---

### 4.2 Zero-Crossing Rate (ZCR) — Tốc độ đổi dấu

**Ý nghĩa:** Đếm xem trong 1 giây, sóng âm **cắt qua trục 0 bao nhiêu lần**. ZCR cao → âm thanh có nhiều tần số cao (sắc, cao giọng). ZCR thấp → âm bass, giọng trầm.

**Công thức:**
$$ZCR = \frac{1}{2N} \sum_{n=1}^{N} |sgn[x(n)] - sgn[x(n-1)]|$$

*sgn(x) = +1 nếu x > 0, -1 nếu x < 0, 0 nếu x = 0*

**Ví dụ tính tay:**

Mảng 8 mẫu: `[0.5, 0.3, -0.1, -0.4, 0.2, 0.6, -0.3, -0.5]`

```
Kí hiệu dấu:  [+,   +,   -,   -,   +,   +,   -,   -]

Đếm chỗ đổi dấu:
  n=1: + → + : KHÔNG đổi
  n=2: + → - : ĐỔI ✓ (lần 1)
  n=3: - → - : KHÔNG đổi
  n=4: - → + : ĐỔI ✓ (lần 2)
  n=5: + → + : KHÔNG đổi
  n=6: + → - : ĐỔI ✓ (lần 3)
  n=7: - → - : KHÔNG đổi

Số lần đổi dấu = 3
ZCR = 3 / (2 × 8) = 3/16 = 0.1875
```

**So sánh ZCR của các loại âm thanh:**

| Loại âm thanh | ZCR điển hình | Giải thích |
|---|---|---|
| Tiếng ồn trắng (white noise) | 0.45 - 0.50 | Dao động kí chanh loạn |
| Giọng nói (phụ âm S, F, SH) | 0.30 - 0.45 | Nhiều tần số cao |
| Giọng nói (nguyên âm A, O) | 0.05 - 0.15 | Tần số cơ bản thấp |
| Nhạc Bass/Drum | 0.02 - 0.08 | Dao động chậm, ổn định |
| Im lặng | ≈ 0.0 | Không có tín hiệu |

**Tự code Python:**

```python
def compute_zcr(samples):
    """Tính tỷ lệ đổi dấu (Zero-Crossing Rate)"""
    N = len(samples)
    crossings = 0
    for n in range(1, N):
        if (samples[n] >= 0 and samples[n-1] < 0) or \
           (samples[n] < 0 and samples[n-1] >= 0):
            crossings += 1
    return crossings / N   # chuẩn hoá theo N (không nhân 2 cho đơn giản)

samples = [0.5, 0.3, -0.1, -0.4, 0.2, 0.6, -0.3, -0.5]
print(compute_zcr(samples))  # → 0.375 (3 lần / 8 mẫu)
```

---

### 4.3 Silence Ratio (Tỷ lệ khoảng lặng)

**Ý nghĩa:** Phần trăm thời gian âm thanh **gần như im lặng** (biên độ dưới ngưỡng). Giọng nói có nhiều khoảng lặng (ngắt câu), nhạc thường liên tục.

**Công thức:**
$$Silence\_ Ratio = \frac{\text{số mẫu có } |x(n)| < \text{threshold}}{N}$$

**Ví dụ với ngưỡng threshold = 0.02:**

```
Mảng: [0.5, 0.3, 0.01, 0.005, 0.008, 0.4, 0.6, 0.015]
Ngưỡng: 0.02

Kiểm tra từng mẫu:
  0.5   > 0.02 → KHÔNG phải silence
  0.3   > 0.02 → KHÔNG phải silence
  0.01  < 0.02 → LÀ silence ✓
  0.005 < 0.02 → LÀ silence ✓
  0.008 < 0.02 → LÀ silence ✓
  0.4   > 0.02 → KHÔNG phải silence
  0.6   > 0.02 → KHÔNG phải silence
  0.015 < 0.02 → LÀ silence ✓

Silence count = 4
Silence Ratio = 4 / 8 = 0.5 → 50% là im lặng
```

**Tự code Python:**

```python
def compute_silence_ratio(samples, threshold=0.02):
    """Tính tỷ lệ khoảng lặng"""
    N = len(samples)
    silence_count = sum(1 for x in samples if abs(x) < threshold)
    return silence_count / N

samples = [0.5, 0.3, 0.01, 0.005, 0.008, 0.4, 0.6, 0.015]
print(compute_silence_ratio(samples))  # → 0.5 (50% silence)
```

---

## 5. Đặc trưng miền tần số

> 📌 **Tham chiếu slide:** *Slide 10 - Chỉ số hóa và truy vấn dữ liệu âm thanh; Slide 3 - Nén dữ liệu DPT*

Các đặc trưng này yêu cầu FFT. **Dùng librosa** để đảm bảo chính xác.

### 5.1 Spectral Centroid (Trọng tâm phổ)

**Ý nghĩa:** "Trung tâm" của năng lượng âm thanh nằm ở tần số nào. Âm sáng, cao → centroid cao. Âm trầm, tối → centroid thấp.

**Ví dụ số:**

Giả sử sau FFT, năng lượng phân bố như sau:

```
Tần số (Hz): [100,  500, 1000, 2000, 4000]
Cường độ:    [0.1,  0.4,  0.8,  0.3,  0.1]
```

```
Spectral Centroid = Σ(tần_số × cường_độ) / Σ(cường_độ)
= (100×0.1 + 500×0.4 + 1000×0.8 + 2000×0.3 + 4000×0.1) / (0.1+0.4+0.8+0.3+0.1)
= (10 + 200 + 800 + 600 + 400) / 1.7
= 2010 / 1.7
= 1182 Hz
```

→ Âm thanh này có "trọng tâm" ở ~1182 Hz (vùng trung cao).

```python
import librosa
y, sr = librosa.load("audio.mp3")
centroid = librosa.feature.spectral_centroid(y=y, sr=sr)
# centroid.mean() → ví dụ: 1500.0 Hz
```

### 5.2 Spectral Bandwidth (Độ rộng phổ)

**Ý nghĩa:** Đo xem năng lượng **trải rộng** hay **tập trung** quanh centroid.

- **Bandwidth nhỏ** (ví dụ: 200 Hz) → Âm thuần, đơn giản (tiếng kèn đơn điệu)
- **Bandwidth lớn** (ví dụ: 2000 Hz) → Âm phức tạp, nhiều thành phần (dàn nhạc, noise)

### 5.3 Spectral Rolloff

**Ý nghĩa:** Tần số mà **85% tổng năng lượng** nằm bên dưới. Là ngưỡng phân biệt âm "cốt lõi" và "đuôi" của tín hiệu.

**Ví dụ:** Rolloff = 3500 Hz → 85% năng lượng bài nhạc nằm trong dải 0–3500 Hz (đây là bài có nhiều bass và mid).

---

## 6. MFCC — Vua của đặc trưng âm thanh

> 📌 **Tham chiếu slide:** *Slide 10 - Chỉ số hóa và truy vấn dữ liệu âm thanh*

### 6.1 MFCC là gì?

**MFCC = Mel-Frequency Cepstral Coefficients** (Hệ số Cepstral theo thang Mel)

Nghe phức tạp, nhưng ý tưởng cốt lõi rất đơn giản:

> **MFCC là cách mô tả âm thanh theo cách tai người nghe, không phải theo cách máy tính đo.**

### 6.2 Tại sao cần MFCC?

Tai người **không nghe tuyến tính**. Nghĩa là:
- Bạn dễ phân biệt 100Hz vs 200Hz (chênh 100Hz)
- Nhưng khó phân biệt 10,000Hz vs 10,100Hz (cũng chênh 100Hz!)

→ Tai người nhạy hơn ở tần số thấp, kém nhạy hơn ở tần số cao.

**Thang Mel** mô phỏng đặc điểm này, "co giãn" trục tần số theo cách tai người cảm nhận.

### 6.3 MFCC được tính như thế nào? (5 bước)

```
Âm thanh thô
     ↓
[Bước 1] Cắt thành từng khung nhỏ (frame)
         Mỗi frame ~ 23ms (512 mẫu ở 22050Hz)
     ↓
[Bước 2] Tính FFT cho từng frame
         → Biết được phổ tần số của frame đó
     ↓
[Bước 3] Áp Mel Filterbank (~26 bộ lọc tam giác)
         → Nhóm các tần số lại theo cách tai người nghe
     ↓
[Bước 4] Log của Mel Spectrum
         → Giống cách tai người cảm nhận độ to (phi tuyến)
     ↓
[Bước 5] DCT (Biến đổi Cosine rời rạc)
         → Lấy 13 hệ số đầu tiên = 13 MFCC
     ↓
Vector 13 số (hoặc 20, 40 tùy cấu hình)
```

### 6.4 MFCC nom như thế nào?

Sau khi tính, mỗi frame cho ra một vector MFCC. Ví dụ với 13 hệ số:

```
MFCC của một frame âm thanh:
[-24.5, 12.3, -3.1, 4.7, -2.9, 1.2, -0.8, 3.3, -1.1, 0.9, -0.5, 1.8, -0.3]
  ^        ^
  |        |
Hệ số 1   Hệ số 2
(quan trọng nhất, | (chi tiết tần số
liên quan âm lượng) | ở mức trung)
```

**Ý nghĩa từng hệ số (đơn giản hóa):**
- **MFCC[0]** (C0): Liên quan đến năng lượng tổng thể, âm lượng
- **MFCC[1-4]**: Hình dạng tổng quát của phổ, phân biệt giọng nam/nữ, nhạc cụ
- **MFCC[5-12]**: Chi tiết nhỏ hơn về cấu trúc âm thanh

### 6.5 Ví dụ so sánh MFCC của 2 bài nhạc

```
Bài nhạc Pop (upbeat):
MFCC mean: [-15.2, 8.1, -2.3, 5.2, -1.8, 2.1, -0.5, 2.8, ...]

Bài nhạc Classical (chậm, nhẹ):
MFCC mean: [-22.1, 4.3, -0.9, 1.8, -0.6, 0.7, -0.2, 0.9, ...]
```

Hai vector này **khác nhau** → CSDL có thể đo được sự khác biệt bằng khoảng cách cosine!

### 6.6 Cách dùng librosa để tính MFCC

```python
import librosa
import numpy as np

y, sr = librosa.load("bai_nhac.mp3")  # load file

# Tính MFCC
mfccs = librosa.feature.mfcc(y=y, sr=sr, n_mfcc=13)
# mfccs.shape = (13, số_frames)
# Ví dụ: (13, 1320) cho bài 30 giây

# Lấy trung bình theo thời gian → vector đại diện cho cả bài
mfcc_mean = np.mean(mfccs, axis=1)
# mfcc_mean.shape = (13,) → 13 con số đại diện cho bài nhạc!

print(mfcc_mean)
# [-18.3, 7.2, -1.8, 3.4, -1.2, 1.5, -0.4, 2.1, -0.8, 0.6, -0.3, 1.1, -0.2]
```

**Tại sao lấy mean?** Mỗi bài nhạc có hàng nghìn frame, mỗi frame cho 1 vector MFCC. Lấy mean (hoặc mean + std) theo thời gian → rút gọn thành 1 vector duy nhất đại diện cho cả bài.

---

## 7. Đặc trưng ảnh (Bonus)

> 📌 **Tham chiếu slide:** *Slide 11 - Chỉ số hóa và truy vấn dữ liệu ảnh*

### 7.1 Color Histogram (Histogram màu sắc)

**Ý nghĩa:** Đếm xem ảnh có bao nhiêu pixel của từng màu.

**Ví dụ với ảnh 4x4 pixel (rất đơn giản):**

```
Ảnh gốc (mỗi ô là giá trị đỏ R):
[200, 210, 50,  60 ]
[190, 205, 45,  55 ]
[220, 215, 40,  65 ]
[195, 200, 55,  50 ]

Sau khi chia thành 4 bins (nhóm):
  Bin 0 (0-63):    8 pixels   → màu tối/xanh lá
  Bin 1 (64-127):  0 pixels
  Bin 2 (128-191): 0 pixels
  Bin 3 (192-255): 8 pixels   → màu đỏ sáng

Color histogram = [8, 0, 0, 8]
→ Vector đặc trưng: [0.5, 0.0, 0.0, 0.5] (chuẩn hóa)
```

```python
import cv2
import numpy as np

img = cv2.imread("anh.jpg")
hist = cv2.calcHist([img], [0, 1, 2], None, [8, 8, 8], [0, 256]*3)
hist_normalized = cv2.normalize(hist, hist).flatten()
# hist_normalized.shape = (512,) → vector 512 chiều
```

### 7.2 Tóm tắt: Ảnh vs Âm thanh

| | Âm thanh | Ảnh |
|---|---|---|
| **Đặc trưng miền "thô"** | Energy, ZCR | Color Histogram |
| **Đặc trưng nâng cao** | MFCC, Spectral | HOG, SIFT, CNN features |
| **Vector điển hình** | 13-40 chiều | 128-512 chiều |

---

## 8. Tổng kết: Feature Vector là gì?

### 8.1 Kết hợp tất cả thành 1 "dấu vân tay"

Mỗi file âm thanh được biến thành **1 vector số** — gọi là **Feature Vector** hay **Embedding**. Đây là "dấu vân tay" của bài nhạc đó trong không gian toán học.

**Ví dụ Feature Vector của 1 bài nhạc (26 chiều):**

```
[
  # Miền thời gian (tự code) — 3 chiều
  energy       = 0.042,
  zcr          = 0.087,
  silence_ratio= 0.150,

  # MFCC (librosa) — 13 chiều
  mfcc_1  = -18.3,
  mfcc_2  =   7.2,
  mfcc_3  =  -1.8,
  ...
  mfcc_13 =  -0.2,

  # Spectral features (librosa) — 3 chiều
  spectral_centroid   = 1823.0,
  spectral_bandwidth  = 1205.0,
  spectral_rolloff    = 3841.0,

  # Chroma (librosa) — 12 chiều (C,D,E,F,G,A,B + flats/sharps)
  chroma_C  = 0.42,
  chroma_D  = 0.18,
  ...
  chroma_B  = 0.09,
]
```

### 8.2 Từ vector đến truy vấn tương tự

**Bài toán:** "Cho tôi các bài nhạc nghe giống bài A"

```
Bài A → [0.04, 0.08, 0.15, -18.3, 7.2, ..., 1823, 1205, 3841]
                    ↓
        Tính khoảng cách cosine với tất cả bài nhạc trong DB
                    ↓
Bài B → khoảng cách = 0.05  ← rất giống!
Bài C → khoảng cách = 0.41  ← hơi giống
Bài D → khoảng cách = 0.89  ← rất khác
                    ↓
        Trả về: [Bài B, Bài C, ...] theo thứ tự giống nhất
```

**Khoảng cách cosine:** Đo góc giữa 2 vector trong không gian nhiều chiều.
- = 0 : hoàn toàn giống nhau (cùng hướng)
- = 1 : hoàn toàn khác nhau (vuông góc)

---

## 9. Tham chiếu slide bài giảng

| Slide | Nội dung | Liên quan đến phần |
|---|---|---|
| **Slide 2**: Các loại dữ liệu DPT | Đặc điểm của âm thanh, ảnh, video là dữ liệu DPT | Mục 1, 7 |
| **Slide 3**: Nén dữ liệu DPT | Lý do cần trích xuất đặc trưng thay vì lưu raw | Mục 5 |
| **Slide 4**: Kiến trúc hệ CSDL DPT | Tổng thế pipeline: thu thập → trích xuất → lưu trữ → truy vấn | Tổng quan |
| **Slide 7**: Cấu trúc dữ liệu đa chiều | Feature vector nhiều chiều, cây B+ cho vector | Mục 8 |
| **Slide 8**: Truy vấn không gian vector | Khoảng cách cosine, phân cụm K-means | Mục 8.2 |
| **Slide 10**: Chỉ số hóa âm thanh | Energy, ZCR, MFCC, Spectrogram | Mục 4, 5, 6 |
| **Slide 11**: Chỉ số hóa ảnh | Color histogram, texture, shape | Mục 7 |
| **Slide 12**: Chỉ số hóa video | Keyframe extraction, motion features | Mở rộng |

---

## 📝 Tóm tắt 1 trang

```
ÂM THANH = mảng số (sample rate 22050Hz → 22050 số/giây)
                        ↓
          ┌─────────────┴──────────────┐
    MIỀN THỜI GIAN              MIỀN TẦN SỐ
    (nhìn theo t/gian)          (nhìn theo tần số)
          │                           │
    ┌─────┼───────┐            ┌──────┼────────┐
  Energy ZCR Silence       Spectral MFCC  Chroma
  (to/nhỏ)(cao/thấp)(lặng)  (centroid) (13 số) (nốt nhạc)
          │                           │
          └─────────────┬─────────────┘
                  FEATURE VECTOR
               [v1, v2, v3, ..., vN]
                        ↓
              LƯU vào pgvector (PostgreSQL)
                        ↓
              TRUY VẤN bằng khoảng cách cosine
                        ↓
              KẾT QUẢ: "Top 5 bài nhạc tương tự"
```

---

*📌 Tài liệu tiếp theo:* **doc2_trien_khai_ky_thuat.md** — Chi tiết code, schema database, API, và hướng dẫn chạy hệ thống.
