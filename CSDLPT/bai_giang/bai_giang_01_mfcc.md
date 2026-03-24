# Bài Giảng 01: MFCC — Dấu Vân Tay Âm Sắc

> **Vị trí trong vector:** Chiều [1–13] = MFCC mean × 13, Chiều [14–26] = MFCC std × 13
> **Tổng: 26 chiều** — chiếm 70% toàn bộ vector Giai Đoạn 1.

---

## Bước 1 — Giải thích bằng ngôn ngữ đời thường

### MFCC là gì?

Hãy tưởng tượng bạn nhắm mắt và nghe hai người nói cùng một câu "Xin chào". Dù cùng nội dung, bạn vẫn phân biệt được ai là nam, ai là nữ, ai là người quen. Bạn làm được điều đó nhờ **chất giọng** — một thứ không phải nốt nhạc, không phải âm lượng, mà là "màu sắc" riêng của giọng nói.

**MFCC chính là cách máy tính "nghe" chất giọng.**

Áp dụng sang nhạc cụ: khi cả Violin và Cello cùng chơi nốt G3 (cùng tần số 196Hz), tai bạn vẫn phân biệt được vì:
- Violin nghe **sáng, mỏng, rõ nét** — giống giọng nữ soprano
- Cello nghe **ấm, dày, trầm** — giống giọng nam baritone

Sự khác biệt "sáng/ấm/dày/mỏng" này chính là **âm sắc (timbre)**, và MFCC là bộ số mô tả nó.

### Tại sao gọi là "Dấu vân tay"?

Giống vân tay con người — mỗi nhạc cụ có một "vân" âm sắc độc nhất. Hai cây Violin khác nhau chơi hai nốt khác nhau, MFCC của chúng vẫn **giống nhau hơn** so với MFCC của một cây Guitar chơi cùng nốt. Đây là sức mạnh cốt lõi của MFCC: **nó bắt nhạc cụ, không bắt nốt nhạc**.

---

## Bước 2 — Giải thích kỹ thuật (đơn giản hóa)

MFCC được tính qua **5 bước tuần tự**. Mỗi bước dưới đây đều có lý do rõ ràng:

### Bước 2.1 — Chia nhỏ âm thanh thành "khung hình" (Framing)

**Vấn đề:** Một file nhạc 10 giây chứa ~220.000 con số (mẫu tín hiệu). Không ai phân tích cả 220.000 số cùng lúc được.

**Giải pháp:** Chia nhỏ thành các **frame** (khung), mỗi khung kéo dài ~93ms (~2048 mẫu), giống như chia một bộ phim thành từng khuôn hình.

**Phép so sánh:** Bạn đang quay video. Thay vì giữ camera mở liên tục rồi cố phân tích cả cuộn phim, bạn chụp nhiều ảnh nhanh liên tiếp. Mỗi "ảnh" là một frame.

**Tại sao lại overlap (chồng lên nhau)?** Các frame không cắt sát nhau mà trượt mỗi lần 512 mẫu (~23ms) — nghĩa là mỗi frame chồng ~75% lên frame trước. Lý do: nếu cắt sát, thông tin ở chỗ tiếp giáp sẽ bị mất.

❌ Hiểu nhầm phổ biến

“1 giây của tần số lấy mẫu = 1 frame”

👉 Điều này không đúng trong xử lý âm thanh.

⸻

✅ Phân biệt rõ 2 thứ

1️⃣ Tần số lấy mẫu (sampling rate)
	•	Là cách biểu diễn tín hiệu
	•	Ví dụ: 22050 Hz = 1 giây có 22050 mẫu

👉 Nó chỉ trả lời:

“Trong 1 giây có bao nhiêu điểm dữ liệu?”

⸻

2️⃣ Frame
	•	Là cách chia nhỏ dữ liệu để xử lý
	•	Ví dụ: 2048 mẫu / frame

👉 Nó trả lời:

“Mỗi lần tôi lấy bao nhiêu mẫu để phân tích?”

⸻

🔥 Điểm mấu chốt

👉 1 giây KHÔNG phải là 1 frame
Mà:

1 giây = rất nhiều frame

⸻

📌 Ví dụ cụ thể (giống của bạn)
	•	Sampling rate: 22050 Hz → 1 giây = 22050 mẫu
	•	Frame size: 2048 mẫu

👉 Số frame trong 1 giây:

\frac{22050}{2048} \approx 10.7 \text{ frame}

👉 Nhưng thực tế còn nhiều hơn vì có overlap!



### Bước 2.2 — Nhân cửa sổ Hann (Windowing)

**Vấn đề:** Khi cắt ngang một sóng âm thanh, hai đầu bị "đứt gãy" đột ngột. Sự đứt gãy này tạo ra các tần số giả mà thực tế không có trong âm thanh.

**Giải pháp:** Nhân mỗi frame với một đường cong hình chuông (cửa sổ Hann) — giữ nguyên phần giữa, làm mờ dần hai đầu về 0.

**Phép so sánh:** Khi chụp ảnh qua cửa sổ ô van trên tàu bay, viền ảnh bị tối dần — đó là hiệu ứng tương tự. Bạn giữ phần trung tâm rõ nét, phần rìa mờ dần để không tạo ra "biên giới cứng" giả tạo.

### Bước 2.3 — Chuyển sang miền tần số (FFT)

**Vấn đề:** Frame lúc này vẫn là một dãy số biên độ theo thời gian. Nó cho biết "âm thanh to/nhỏ thế nào ở mỗi thời điểm" nhưng **không cho biết có những tần số nào đang vang**.

**Giải pháp:** Phép biến đổi Fourier nhanh (FFT) chuyển frame từ "biên độ theo thời gian" sang "cường độ theo tần số".

**Phép so sánh:** Bạn đang đứng trong một dàn nhạc. Tai bạn nghe thấy một khối âm thanh hỗn tạp. FFT giống như bạn đeo tai nghe công nghệ cao có thể tách riêng ra: "À, đang có Violin ở 2000Hz, Cello ở 400Hz, và tiếng phòng ở 100Hz."

**Kết quả:** Mỗi frame bây giờ trở thành một **thanh bar** (giống equalizer trên loa) — mỗi cột cho biết cường độ của một tần số cụ thể.

### Bước 2.4 — Lọc qua bộ lọc Mel (Mel Filterbank)

**Vấn đề:** FFT đưa ra ~1025 cột tần số, chia đều từ 0Hz đến 11025Hz. Nhưng tai người **không nghe tần số đều**:
- Bạn phân biệt rõ ràng 200Hz vs 400Hz (khác nhau 200Hz)
- Bạn **không** phân biệt được 8000Hz vs 8200Hz (cũng khác 200Hz)

Tức là tai nhạy hơn ở tần số thấp, kém nhạy ở tần số cao. Nếu dùng FFT thô, phần tần số cao sẽ chiếm quá nhiều chiều mà thông tin rất ít.

**Giải pháp:** Nhóm 1025 cột FFT thành **40 nhóm** (gọi là 40 bộ lọc Mel), dày đặc ở tần số thấp và thưa dần ở tần số cao — mô phỏng cách tai người nghe.

**Phép so sánh:** Bàn phím piano. Các phím bass (trái) cách nhau ít Hz nhưng bạn nghe rõ từng nốt. Các phím treble (phải) cách nhau nhiều Hz nhưng tai bạn nghe chúng gần giống nhau. Mel filterbank "ép" trục tần số lại cho giống cách cảm nhận thật.

**Kết quả:** Từ 1025 cột → còn 40 nhóm đại diện.

### Bước 2.5 — Logarithm và DCT → ra MFCC

**Vấn đề 1:** Cảm nhận âm lượng cũng không tuyến tính. Tăng từ 1W lên 2W (gấp đôi) thì bạn thấy to hơn rõ rệt. Tăng từ 100W lên 101W (thêm 1%) thì bạn gần như không thấy khác biệt. Giải pháp: lấy **logarithm** của 40 giá trị năng lượng — nén khoảng giá trị cho giống cảm nhận thật.

**Vấn đề 2:** 40 nhóm Mel liền kề thường có giá trị tương tự nhau (nhóm 5 và nhóm 6 gần bằng nhau). Có thông tin dư thừa.

**Giải pháp:** Phép biến đổi DCT (Discrete Cosine Transform) — "ép" 40 nhóm xuống còn **13 hệ số MFCC** độc lập. DCT làm điều tương tự như khi bạn nén ảnh JPEG: giữ lại thông tin quan trọng nhất, bỏ phần chi tiết thừa.

**Kết quả cuối cùng:** Mỗi frame (~93ms) → **13 con số MFCC**. Toàn bộ file (hàng trăm frame) → ma trận 13 × N, trong đó N là số frame.

---

## Bước 3 — Lý do chọn tham số

### Tại sao `n_mfcc = 13`?

Con số 13 đến từ thực nghiệm trong lĩnh vực nhận dạng giọng nói từ thập niên 1980. Các nhà nghiên cứu phát hiện:
- **Hệ số 1** mô tả **hình dạng tổng thể** (sáng hay trầm) — ảnh hưởng lớn nhất
- **Hệ số 2–5** mô tả các **đặc điểm thô** của phổ tần — có ảnh hưởng rõ ràng
- **Hệ số 6–13** mô tả các **chi tiết nhỏ** — vẫn có ích nhưng yếu dần
- **Hệ số 14 trở đi** mô tả các **biến thiên quá nhanh** — gần như là nhiễu, không giúp nhận dạng

13 là điểm cân bằng: đủ thông tin mà không thêm nhiễu. Đây là **quy ước ngành** (convention) được dùng trong hầu hết hệ thống nhận dạng âm thanh, không chỉ riêng nhạc cụ.

### Tại sao `frame_size = 2048`?

Ở tần số lấy mẫu 22050Hz, frame 2048 mẫu kéo dài **~93ms**. Đây là khoảng thời gian mà âm thanh nhạc cụ **gần như đứng yên** (stationary) — tức đặc tính tần số không thay đổi đáng kể. Nếu frame quá ngắn (512 mẫu = 23ms), không đủ dữ liệu để FFT cho ra kết quả chính xác ở tần số thấp. Nếu frame quá dài (8192 mẫu = 370ms), âm thanh kịp thay đổi trong lúc phân tích.

### Tại sao `n_mels = 40`?

40 bộ lọc Mel là đủ để bao phủ dải tần 0–11025Hz với độ phân giải hợp lý cho con người. Giảm xuống 20 thì mất chi tiết; tăng lên 128 thì bộ lọc chồng lấp quá nhiều mà không thêm thông tin quan trọng. 40 là giá trị mặc định trong librosa và hầu hết hệ thống MIR.

---

## Bước 4 — Tại sao MFCC có 13 chiều? Mỗi chiều là gì?

### Tại sao không đặt tên riêng?

Khác với "Attack Time" hay "ZCR" có tên mô tả rõ ràng, 13 hệ số MFCC **không có tên** vì chúng là kết quả của phép biến đổi toán học (DCT) — mỗi hệ số không tương ứng với một khái niệm vật lý cụ thể nào.

Tuy nhiên, có thể hiểu sơ bộ:

| Hệ số | Mô tả gần đúng | Phép so sánh |
|---|---|---|
| **MFCC[0]** (C0) | Năng lượng tổng thể của phổ | "Tiếng to hay nhỏ" — nhưng đã lọc Mel |
| **MFCC[1]** | Phổ nghiêng về trầm hay sáng? | "Người nói giọng trầm hay giọng cao" |
| **MFCC[2]** | Phổ có 1 đỉnh hay 2 đỉnh năng lượng? | "Giọng ngực hay giọng mũi" |
| **MFCC[3-5]** | Chi tiết hình dạng phổ bậc thấp | "Ngoại hình khuôn mặt: to/nhỏ, dài/tròn" |
| **MFCC[6-12]** | Chi tiết phổ bậc cao — tinh tế hơn | "Chi tiết khuôn mặt: mắt, mũi, miệng" |

Hệ số C1 (hay thường gọi là Spectral Slope) đại diện cho độ dốc của phổ.

  - Giá trị C1 dương (> 0): Thường biểu thị năng lượng dịch chuyển nhiều hơn về phía tần số cao (âm thanh "sáng").

  - Giá trị C1 âm (< 0): Biểu thị năng lượng tập trung ở tần số thấp (âm thanh "trầm/tối").

Nếu C1 > 0: Năng lượng ở vùng tần số cao (bên phải) lớn hơn vùng tần số thấp (bên trái). Do đó, đường dốc tổng thể sẽ đi từ dưới bên trái lên trên bên phải. (phổ Mel)

Nếu C1  < 0: Năng lượng ở vùng tần số thấp (bên trái) lớn hơn. Đường dốc sẽ đi từ trên bên trái xuống dưới bên phải (giống hệt như cái biểu đồ màu tím trong ảnh của bạn). (phổ Mel)
hai nhạc cụ khác nhau có thể có C1 giống nhau nhưng hình dạng phổ khác hoàn toàn.

1. Tại sao C1 giống nhau nhưng phổ lại khác?

Hãy tưởng tượng C1 giống như độ dốc của một con đường từ điểm đầu đến điểm cuối:

Nhạc cụ A: Có phổ đi xuống đều tăm tắp từ trái sang phải (như một cái cầu trượt thẳng).

Nhạc cụ B: Có phổ uốn lượn, lúc đầu rất cao, giữa cực thấp, rồi cuối lại hơi cao một chút.

**Phép so sánh tổng thể:** Hãy tưởng tượng bạn mô tả khuôn mặt một người bằng 13 con số. Số đầu tiên là "mặt tròn hay dài?", số thứ hai là "trán rộng hay hẹp?", dần dần đến chi tiết hơn như "mũi cao bao nhiêu?". Không ai đặt tên cho "tham số khuôn mặt số 7" — nhưng tổ hợp 13 số này đủ để phân biệt bạn với mọi người khác.

---

## Bước 5 — Tại sao cần cả Mean lẫn Std?

### MFCC Mean (13 chiều): "Ảnh chân dung"

Một file âm thanh 10 giây tạo ra khoảng **430 frame**, mỗi frame có 13 hệ số MFCC. Ta lấy **trung bình** (mean) của 430 giá trị cho mỗi hệ số → được 13 con số đại diện cho **âm sắc trung bình** của cả file.

**Phép so sánh:** Nếu bạn chụp 430 ảnh khuôn mặt của một người ở nhiều góc độ, rồi "chồng" tất cả lại, bạn được một **ảnh trung bình** — nó không giống hoàn toàn bất kỳ ảnh nào nhưng đại diện tốt cho khuôn mặt người đó.

**MFCC Mean bắt được:** Loại nhạc cụ nào — Violin, Cello hay Guitar.

### MFCC Std (13 chiều): "Mức độ biến hóa"

**Phép so sánh:** Nếu 430 ảnh khuôn mặt đều giống nhau (người ngồi yên), std ≈ 0. Nếu 430 ảnh rất khác nhau (người đang nhắm/mở mắt, cười/khóc), std lớn.

Áp dụng cho nhạc cụ:
- **Violin kéo legato** (kéo dài liên tục): MFCC thay đổi rất ít giữa các frame → **std thấp**
- **Guitar chơi hợp âm rải** (arpeggio): mỗi nốt khác tần số, MFCC nhảy liên tục → **std cao**

**MFCC Std bắt được:** Kỹ thuật chơi và mức độ biến hóa âm sắc theo thời gian.

### Tóm tắt

| | Mean | Std |
|---|---|---|
| **Đo cái gì** | Âm sắc trung bình | Âm sắc thay đổi bao nhiêu |
| **Phân biệt** | Violin **vs** Guitar | Legato **vs** Arpeggio |
| **Nếu là ảnh** | Ảnh chân dung | Mức độ "nhấp nháy" |

---

## Bước 6 — Vai trò trong bài toán nhạc cụ bộ dây

### MFCC phân biệt được những cặp nào?

| Cặp nhạc cụ | Tại sao MFCC phân biệt được | Chiều nào đóng vai trò chính |
|---|---|---|
| **Violin vs Cello** | Thân đàn Violin nhỏ → phổ sáng, hài âm bậc cao mạnh. Cello lớn → phổ trầm hơn, hài âm bậc thấp chiếm ưu thế. | MFCC mean[1] (nghiêng trầm/sáng) |
| **Guitar vs Violin** | Guitar có 6 dây, thân gỗ lớn → cộng hưởng khác hoàn toàn. MFCC bắt chính xác hình dạng phổ này. | MFCC mean[2-5] (hình dạng phổ) |
| **Cello vs Contrabass** | Contrabass to gấp đôi, tần số cộng hưởng thấp hơn rõ rệt. MFCC phản ánh sự khác biệt kích thước hộp cộng hưởng. | MFCC mean[0-2] |
| **Harp vs Guitar** | Dây ruột (Harp) vs dây thép/nylon (Guitar) → khác chất liệu dây → khác hình dạng phổ. | MFCC mean[3-8] (chi tiết phổ) |

### MFCC KHÔNG phân biệt được gì?

- **Violin arco (kéo vĩ) vs Violin pizzicato (gảy)** trên cùng một cây đàn — vì MFCC mean gần giống nhau (cùng hộp cộng hưởng). Cần thêm đặc trưng khác (Attack Time, ZCR) để phân biệt.
- **Cùng nhạc cụ chơi nốt khác nhau** — MFCC bất biến với nốt nhạc khá tốt, nhưng ở hai đầu cực dải (nốt cực trầm vs cực cao trên cùng đàn) thì MFCC có thể hơi khác.

---

## Bước 7 — Câu hỏi giảng viên thường hỏi

### ❓ "MFCC là gì? Tại sao dùng MFCC thay vì FFT trực tiếp?"

> **Gợi ý trả lời:** "FFT trực tiếp cho ra hơn 1000 giá trị tần số, phần lớn là dư thừa vì tai người không nghe đều các tần số. MFCC nén thông tin quan trọng nhất vào 13 hệ số bằng cách mô phỏng cách tai người nghe (thang Mel) rồi phi tương quan hóa (DCT). Kết quả là 13 con số độc lập, nhỏ gọn, nhưng đủ để phân biệt nhạc cụ."

### ❓ "Tại sao lại lấy 13 hệ số mà không phải 20 hay 5?"

> **Gợi ý trả lời:** "13 là quy ước trong nghiên cứu nhận dạng giọng nói và MIR từ thập niên 1980. Các hệ số đầu mô tả hình dạng tổng thể của phổ, hệ số sau mô tả chi tiết dần. Hệ số thứ 14 trở đi bắt đầu bắt các biến thiên quá nhanh — gần giống nhiễu. 13 là điểm cân bằng giữa thông tin và nhiễu."

### ❓ "MFCC có nhạy cảm với nốt nhạc không? Nếu Violin chơi nốt C4 và D4 thì MFCC có khác không?"

> **Gợi ý trả lời:** "MFCC được thiết kế để bắt đặc tính âm sắc của nguồn phát, không phải nốt nhạc cụ thể. Violin chơi C4 và D4 sẽ có MFCC rất giống nhau vì hộp cộng hưởng và dây đàn tạo ra cùng hình dạng phổ. Tuy nhiên, ở hai đầu cực dải tần của một nhạc cụ, MFCC có thể hơi khác — vì vậy chúng em tính mean trên toàn bộ file để giảm thiểu ảnh hưởng này."

---

> **Tổng kết:** MFCC mean + std chiếm **26/37 chiều** (70%) trong vector Giai Đoạn 1. Đây là "cột sống" của hệ thống — nếu chỉ dùng MFCC thôi đã cho kết quả khá tốt. Các đặc trưng còn lại (Contrast, Centroid, ZCR...) bổ sung thêm góc nhìn để nâng cao độ chính xác.

---

**Bạn đã sẵn sàng sang đặc trưng tiếp theo (Spectral Contrast — 7 chiều) chưa?**
