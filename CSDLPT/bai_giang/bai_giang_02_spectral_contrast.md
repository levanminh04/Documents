# Bài Giảng 02: Spectral Contrast — Độ Nét Của Tín Hiệu

> **Vị trí trong vector:** Chiều [27–33] = Spectral Contrast × 7
> **Tổng: 7 chiều** — bổ sung khía cạnh "thuần khiết" của âm sắc.

---

## Bước 1 — Giải thích bằng ngôn ngữ đời thường

### Spectral Contrast đo cái gì?

Hãy tưởng tượng bạn đang nhìn vào một dãy núi in bóng lên nền trời.
- Nếu các ngọn núi mọc lên **rất cao và nhọn hoắt**, xen kẽ là các thung lũng **rất sâu**, bạn gọi đó là bức tranh có **độ tương phản cao (High Contrast)**.
- Nếu đó chỉ là một đồi cát **nhấp nhô thoải đều**, bức tranh có **độ tương phản thấp (Low Contrast)**.

Áp dụng sang âm thanh:
- Ngọn núi = Các nốt nhạc, các âm bồi (hài âm) thực sự vang lên.
- Thung lũng = Đầu mút sợi dây cọ xát, tiếng rít của vĩ, tiếng ồn xung quanh.
- **Spectral Contrast (Tương phản phổ)** đo sự chênh lệch giữa đỉnh núi và đáy thung lũng. 

Âm thanh nào **rất trong trẻo, vang vọng rõ nốt** (như tiếng đàn Harp gảy) thì ngọn núi cao, thung lũng sâu → **Contrast cao**. 
Âm thanh nào có **nhiều tiếng xì xào, ma sát** (như lúc mới bắt đầu kéo vĩ trên Violin) thì ngọn núi và thung lũng bị lẫn vào nhau → **Contrast thấp**.

---

## Bước 2 — Giải thích kỹ thuật (đơn giản hóa)

Spectral Contrast được tính theo quy trình sau:

1. **Phân tích phổ (FFT):** Đầu tiên, ta chuyển âm thanh từ sóng thời gian sang khung cảnh "dãy núi" tần số bằng phép biến đổi FFT (như đã học ở bài trước).
2. **Chia lô (Frequency Bands):** Thay vì nhìn cả dãy núi dài, ta lấy ranh giới chia dãy núi thành **7 dải tần số** (ví dụ: Bass, Low-Mid, Mid, High-Mid...).
3. **Tìm đỉnh và đáy:** Trong mỗi dải đó, hệ thống tìm **đỉnh núi cao nhất** (năng lượng lớn nhất) và **đáy thung lũng sâu nhất** (năng lượng nhỏ nhất).
4. **Trừ cho nhau:** `Contrast = Đỉnh - Đáy`. 

Vì tai người nghe âm lượng theo hàm log, phép tính này trên thực tế lấy log của Đỉnh trừ đi log của Đáy. Ta làm việc này ở cả 7 dải tần số.

---

## Bước 3 — Lý do chọn tham số

Tại sao Librosa lại ngầm định chia thành 7 dải tần số? Các dải này phụ thuộc vào 2 tham số: `fmin` (tần số thấp nhất) và `n_bands` (số lượng dải).

- `n_bands = 6` (mặc định của Librosa): Thuật toán sẽ chia dải tần dựa trên ranh giới là các quãng tám (octave).
  - Vì dải tần giới hạn của file âm thanh lấy mẫu ở 22050 Hz là ~11025 Hz, thuật toán cắt ra 6 dải quãng tám nằm ở giữa, cộng với **1 dải chóp** (dải cao nhất bao phủ toàn bộ phần còn lại).
  - Vậy `6 + 1 = 7` dải tần số.
- Các dải này hẹp ở tần số thấp và rộng ở tần số cao — tiếp tục mô phỏng **cách tai người phân giải âm thanh** (giống như bộ lọc Mel trong MFCC, tai người nhạy phân biệt các nốt trầm hơn).

---

## Bước 4 — Tại sao Spectral Contrast lại có 7 chiều?

Mỗi chiều tương ứng với **một dải tần số cụ thể**. Không phải bộ phận nào của dãy núi cũng nhấp nhô giống nhau!

- **Chiều 1 (Dải trầm nhất, ví dụ < 200Hz):** Đo độ rõ nét của âm nền, của thùng đàn to. Ở Cello, dải này có ngọn núi rất cao. Ở Violin, dải này gần như chẳng có núi đồi gì.
- **Chiều 2–5 (Dải trung):** Nơi chứa những âm bồi (hài âm) quan trọng nhất giúp ta định hình giọng nhạc cụ.
- **Chiều 6–7 (Dải siêu cao, > 4000Hz):** Nơi chủ yếu chứa tiếng ồn — ví dụ tiếng ma sát của lông vĩ lên dây đàn trần, tiếng móng tay chạm vào dây nilon.

7 con số này cho ta một cái nhìn toàn cảnh: "Dải trầm thì âm thanh rất sạch (High), nhưng dải cao thì rất xước/mờ (Low)".

---

## Bước 5 — Tại sao tính Mean (Trung bình) của 7 dải?

Thực tế, `librosa.feature.spectral_contrast` trả về một ma trận 7 hàng (ứng với 7 dải) nhưng có **hàng trăm cột** (ứng với hàng trăm frame thời gian của bài nhạc).

Bởi vì một file âm thanh kéo dài nhiều giây, giá trị Contrast thay đổi liên tục. Ta thực hiện hành động **"ép dẹt" (tính mean theo trục thời gian)** để tóm tắt: *"Nhìn chung trong cả bản ghi âm này, dải trầm sạch đến mức nào? Dải cao xước đến mức nào?"*. 

Về mặt kỹ thuật, bạn sẽ gom hàng trăm giá trị của dãy số 1 thành **một con số Mean duy nhất**. Làm tương tự cho 7 dãy, bạn thu được đúng **7 giá trị (7 chiều)**. Việc lấy trung bình giúp đặc trưng ổn định, không bị tác động bởi một tiếng gảy lỗi ngắn ngủi.

---

## Bước 6 — Vai trò trong bài toán nhạc cụ bộ dây

Spectral Contrast là một "trợ thủ đắc lực" cho MFCC. Trong khi MFCC bắt "hình dáng chung" thì Contrast bắt "độ rít" và "độ sạch".

| Cặp nhạc cụ | Vai trò của Spectral Contrast | Chiều nào đóng vai trò chính |
|---|---|---|
| **Violin Arco (Kéo) vs Violin Pizz (Gảy)** | Kéo vĩ sinh ra tiếng ma sát liên tục (âm nhiễu) ở tầng cao. Gảy dây thì nốt trong trẻo nhưng tắt sớm. Contrast ở tần số cực cao của Kéo sẽ bị thấp hơn (thung lũng bị tiếng rít lấp đầy). | Contrast [6-7] (Dải cực cao) |
| **Guitar Nilon vs Guitar Thép** | Dây thép tạo ra các hài âm chói và sắc cạnh hơn. Ngọn núi của dây thép nhọn hoắt, Contrast cao hơn hẳn ở dải trung-cao. | Contrast [4-5] (Dải trung cao) |
| **Harp vs Cello (pizzicato)** | Harp được thiết kế chuyên dụng để gảy cộng hưởng cực tốt, âm cực trong. Contrast của Harp ở dải trung sẽ vượt trội so với tiếng gảy vội vã của Cello. | Contrast [2-4] (Dải trầm tới trung) |

---

## Bước 7 — Câu hỏi giảng viên thường hỏi

### ❓ "Ủa, sao em đã dùng MFCC rồi mà còn phải dùng thêm Spectral Contrast? Chúng không bị trùng lặp thông tin à?"

> **Gợi ý trả lời:** "Dạ thưa thầy/cô, chúng không trùng lặp mà bổ trợ cho nhau. MFCC mô tả **'đường bao'** tổng thể của phổ (Envelope) — tức là phổ nghiêng về trầm hay bổng, dáng thoải ra sao. Còn Spectral Contrast lại đo **'bề mặt mịn hay gai góc'** của phổ đó thông qua việc so chênh lệch năng lượng Đỉnh và Đáy. MFCC dễ phân biệt Violin và Cello, còn Contrast lại rất nhạy trong việc phân biệt tiếng ma sát kéo vĩ dạt dào với tiếng gảy tròn vành rõ nốt."

### ❓ "Tại sao lại là 7 chiều mà không phải con số khác?"

> **Gợi ý trả lời:** "Dạ, thuật toán chuẩn (và hàm librosa) chia phổ tần số theo các dải octave (quãng tám) mô phỏng thính giác người. File âm thanh của em lấy mẫu ở 22050Hz, dùng 6 dải octave bao trùm phổ chính cộng thêm 1 dải gom toàn bộ tần số cực cao phía trên, nên kết quả tự nhiên cho ra 7 dải (7 chiều). Em đã thử nghiệm và thấy 7 chiều này giữ lại đủ bức tranh phân hóa mà không làm vector bị phình to (Curse of Dimensionality)."

---

> **Tổng kết:** Spectral Contrast đóng góp **7 chiều** vững chắc. Nó như một chuyên gia soi kính lúp vào từng dải tần số để xem âm nhạc đang được chơi sạch và thuần túy đến mức nào, hay đang bị lẫn lộn tiếng ồn ma sát. Kết hợp với 26 chiều của MFCC, hệ thống bạn đã có một "bộ vuốt" quét qua mọi ngóc ngách của không gian Spectral.

---

**Bạn đã sẵn sàng sang đặc trưng tiếp theo (Spectral Centroid — 1 chiều) chưa?**
