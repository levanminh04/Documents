# 🔬 Tài liệu 3: Chi tiết Thuật toán Manual DSP (Bóc tách Librosa)

> **Mục tiêu:** Giải thích chi tiết "bên trong chiếc hộp đen" của các thư viện xử lý tín hiệu. Tài liệu này giúp bạn hiểu từng dòng code NumPy tương ứng với bước nào trong lý thuyết xử lý tín hiệu số (DSP).

---

## Tổng quan Pipeline Manual
Để đi từ một file âm thanh thô (.wav/.mp3) đến một Vector đặc trưng, chúng ta đi qua 5 trạm dừng chính:

1. **Framing & Windowing:** Chia nhỏ và làm mượt.
2. **FFT (Fast Fourier Transform):** Soi tần số.
3. **Spectral Features:** Trích xuất đặc tính phổ (Centroid, Rolloff).
4. **Mel Filterbank:** "Bóp méo" tần số theo tai người.
5. **Log + DCT:** Nén dữ liệu thành MFCC.

---

## 1. Framing & Windowing (Chia khung & Tạo cửa sổ)

### Tại sao phải làm bước này?
Âm thanh là tín hiệu biến đổi liên tục. Nếu tính FFT trên cả bài nhạc 3 phút, ta sẽ mất hết thông tin về thời gian. Ta cần chia nhỏ bài nhạc thành các **Frames** (~20-40ms) để coi tín hiệu trong đó là đứng yên (stationary).

### Code & Giải thích:
```python
def get_frames(samples, frame_size=2048, hop_size=512):
    frames = []
    for start in range(0, len(samples) - frame_size, hop_size):
        frames.append(samples[start : start + frame_size])
    return np.array(frames)
```
- **frame_size (2048):** Độ dài mỗi khung. Ở 22050Hz, nó tương đương ~93ms.
- **hop_size (512):** Bước nhảy. Ta cho các khung chồng lên nhau (overlap) để không làm mất thông tin ở biên.

### Windowing (Hann Window):
Khi cắt một đoạn tín hiệu, các điểm đầu và cuối thường bị "đứt gãy" đột ngột. Điều này tạo ra các tần số giả (Spectral Leakage). Ta nhân với **Window function** để làm mượt hai đầu về 0.

```python
# Công thức Hann: w(n) = 0.5 * (1 - cos(2πn / (N-1)))
hann_window = 0.5 * (1 - np.cos(2 * np.pi * np.arange(N) / (N - 1)))
windowed_frames = frames * hann_window
```

---

## 2. FFT & Power Spectrum (Biến đổi Fourier)

### Lý thuyết:
FFT chuyển từ **Miền thời gian** (Biên độ) sang **Miền tần số** (Cường độ của từng tần số).

### Code NumPy:
```python
# rfft: FFT cho tín hiệu thực, trả về N/2 + 1 bins (từ 0Hz đến sr/2)
stft = np.fft.rfft(windowed_frames, axis=1)
magnitude = np.abs(stft)  # Phổ biên độ
power = magnitude ** 2     # Phổ năng lượng (Power Spectrum)
```

**Tham chiếu Slide 10:** Đây chính là bước đầu tiên để nhìn thấy "hình hài" của âm thanh trong không gian tần số.

---

## 3. Trích xuất Đặc trưng Phổ (Manual)

### 3.1 Spectral Centroid (Trọng tâm phổ)
Nó giống như "điểm cân bằng" của phổ. Nếu Centroid cao → âm thanh xì xào, sắc bén. Thấp → âm trầm, ấm.

```python
# Công thức: Centroid = Σ(f * M) / ΣM
freqs = np.fft.rfftfreq(frame_size, d=1.0/sr)
centroid = np.sum(freqs * magnitude) / np.sum(magnitude)
```

### 3.2 Spectral Rolloff
Tần số mà dưới đó chứa 85% năng lượng. Dùng để phân biệt tiếng ồn (noise) với âm thanh có cấu trúc nhạc tính.

---

## 4. Mel Filterbank (Bộ lọc Mel)

Đây là bước quan trọng nhất để tạo ra MFCC. Tai người không nghe tần số một cách tuyến tính (ta nhạy với âm trầm hơn âm cao).

### Cách xây dựng:
1. Chuyển tần số Hz sang Mel: `m = 2595 * log10(1 + f/700)`
2. Chia đều các điểm trên thang Mel.
3. Chuyển ngược lại Hz để biết các điểm đó nằm đâu trên phổ FFT.
4. Tạo các bộ lọc hình tam giác chung đỉnh.

```python
# Tạo n_mels bộ lọc (thường là 40)
for m in range(1, n_mels + 1):
    # Tạo tam giác từ hz_pts[m-1] lên hz_pts[m] rồi xuống hz_pts[m+1]
    # ... code vòng lặp tạo dốc tam giác ...
```

---

## 5. Log & DCT (Hệ số MFCC)

### Logarithm:
Cảm nhận về độ to của tai người cũng theo thang Log (Decibel). Ta lấy `log` của năng lượng sau khi qua bộ lọc Mel.

### DCT (Discrete Cosine Transform):
Tại sao không dùng FFT nữa mà dùng DCT?
Các bộ lọc Mel nằm cạnh nhau thường có năng lượng tương quan (giống nhau). DCT giúp "phi tương quan" (decorrelate) chúng, biến chúng thành các hệ số độc lập.

```python
from scipy.fftpack import dct
mfccs = dct(log_mel_energy, type=2, axis=0, norm='ortho')[:13]
```
- Ta thường chỉ lấy **13 hệ số đầu tiên**. Các hệ số sau chứa thông tin về các biến đổi quá nhanh của phổ, thường không mang lại giá trị nhận dạng cao.

---

## 💡 Bí kíp Bảo vệ đồ án (Defense Strategy)

Khi thầy cô hỏi: **"Tại sao em lại tự viết code này mà không dùng hàm của Librosa?"**

**Bạn trả lời:**
1. **Sâu sát bản chất:** "Librosa là một thư viện mạnh nhưng nó là 'hộp đen'. Nhóm em muốn tự implement bằng NumPy dựa trên công thức trong Slide 10 để hiểu rõ cách tín hiệu được framing, nhân cửa sổ Hann, và cách thang Mel 'bóp méo' tần số như thế nào."
2. **Kiểm soát thông số:** "Việc tự code giúp nhóm em kiểm soát chính xác số lượng Mel filters và cách tính Spectral Centroid, giúp tối ưu hóa Vector đặc trưng cho riêng bài toán tìm kiếm âm thanh của nhóm."
3. **Hiệu năng:** "Dùng NumPy thuần giúp giảm phụ thuộc vào các thư viện quá nặng, giữ cho backend FastAPI của nhóm gọn nhẹ hơn."

---

## 📊 So sánh Tóm tắt

| Bước | Thành phần xử lý | Ý nghĩa lý thuyết |
|---|---|---|
| **Framing** | `for start in range...` | Chia nhỏ tín hiệu thực tế |
| **Windowing** | `np.cos` (Hann) | Chống nhiễu tại biên khung |
| **FFT** | `np.fft.rfft` | Chuyển sang miền tần số |
| **Mel Filter** | `np.linspace + log` | Mô phỏng tai người |
| **DCT** | `scipy.fftpack.dct` | Rút gọn thông tin quan trọng nhất |

---

*Tài liệu này đi kèm với code thực tế trong `backend/feature_extractor.py` tại `doc2_trien_khai_ky_thuat.md`.*
