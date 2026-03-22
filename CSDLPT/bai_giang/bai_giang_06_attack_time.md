# Bài Giảng 06: Attack Time — Thời Gian Khởi Phát

> **Vị trí trong vector:** Chiều [0] = Attack Time
> **Tổng: 1 chiều** — biến phân định rạch ròi nhất giữa họ "Dây kéo" (Arco) và họ "Dây gảy" (Plucked). Đây cũng là chiều cuối cùng khép lại 37 chiều của Giai đoạn 1.

---

## Bước 1 — Giải thích bằng ngôn ngữ đời thường

### Attack Time đo cái gì?

Hãy tưởng tượng bạn đang chuẩn bị tưới cây và cần lấy nước từ vòi:
- **Cách 1 (Gảy đàn - Pizzicato/Guitar):** Bạn vặn mạnh cần gạt vòi nước lên mức to nhất ngay lập tức. Nước phụt ra **đạt sức ép tối đa trong nháy mắt**.
- **Cách 2 (Kéo vĩ - Arco Violin/Cello):** Bạn nắm lấy núm vặn và xoay từ từ để dòng nước mạnh dần lên, mạnh dần lên, và **phải mất một lúc sau nó mới đạt sức ép tối đa**.

Trong âm thanh, **Attack Time** chính là chiếc đồng hồ bấm giờ đo khoảng thời gian từ lúc "bắt đầu vặn vòi" cho đến lúc "nước phụt ra đạt đỉnh cao nhất". 

Âm thanh nào có **Attack Time ngắn** đồng nghĩa với việc năng lượng được cung cấp trọn vẹn trong một tích tắc (tiếng súng nổ, tiếng gõ trống, tiếng gảy đàn). 
Âm thanh nào có **Attack Time dài** đồng nghĩa năng lượng cần thời gian để tích tụ dần dần lên tới đỉnh (tiếng cọ xát của vĩ kéo, tiếng hát ngân nga từ từ bật hơi).

---

## Bước 2 — Giải thích kỹ thuật (đơn giản hóa)

Khác với các đặc trưng trước đó phải xử lý phức tạp trên từng khung hình (frame), Attack Time làm việc trực tiếp trên "Biểu đồ đường Âm lượng" (chính là đường cong RMS mà ta đã học ở bài trước).

1. **Chuẩn bị đồ thị:** Máy tính nhìn vào toàn bộ biểu đồ âm lượng (RMS) của file âm thanh.
2. **Tìm Điểm bắt đầu (Start):** Máy tính không xuất phát từ giây số 0, vì có thể file ghi âm bị dính tiếng ồn hoặc trống không lúc đầu. Nó tìm khoảnh khắc đầu tiên mà âm lượng bắt đầu tăng vượt qua một mốc "tỉnh giấc" (ví dụ: mốc 20% âm lượng tối đa) và bấm giờ báo chạy.
3. **Tìm Điểm đỉnh điểm (Peak):** Máy tính tiếp tục chạy trên biểu đồ để tìm thấy cái đỉnh núi cao nhất (100% âm lượng lớn nhất). Bấm dừng đồng hồ.
4. **Phép trừ cơ bản:** `Attack Time (giây) = Thời gian tại Peak - Thời gian tại Start`.

---

## Bước 3 — Lý do chọn tham số

Tham số quan trọng nhất của thuật toán đo Attack Time chính là **Mức ngưỡng tỉnh giấc (Onset Threshold)**, ví dụ `20%`:
- Tại sao không để máy bắt đầu đo từ 0% hay 1%? Bởi vì trong bất kỳ phòng thu nào cũng có tiếng ồn nền (Noise Floor) hay tiếng ghế cọt kẹt trước khi nghệ sĩ thực sự chơi đàn. Máy có thể nhầm sự thay đổi của tiếng ghế là "điểm xuất phát", trong khi nốt đàn thật mãi vài giây sau mới vang lên.
- Đặt mốc `20%` (so với đỉnh núi cao nhất) giúp máy phớt lờ hết mọi âm thanh nhiễu nhỏ bé, và chỉ thực sự bấm giờ khi nốt đàn bắt đầu bùng nổ vọt lên.

---

## Bước 4 — Tại sao đặc trưng này có 1 chiều và không cần tính Mean/Std?

Tương tự như chiều dài của 1 đoạn thẳng, "thời gian chạy từ A đến B" là định lượng có một thứ nguyên duy nhất (giây). 
Hơn nữa, đây là một **Giá trị Toàn cục (Global Feature)** chứ không phải một giá trị chạy theo từng frame như MFCC hay ZCR:
- MFCC trích xuất được 400 giá trị cho 400 frame thời gian, nên ta mới cần thao tác "ép dẹt" (tính Mean).
- Attack Time thì mặc định thuật toán chỉ tìm kiếm cái đỉnh núi cao nhất **đầu tiên** của file âm thanh để làm thước đo duy nhất. Máy tính chỉ nhả ra đúng số đo `0.04` (giây), và ta nhét nó với đúng hình dạng vô hướng nguyên bản của nó thẳng vào vector (chiếm 1 chiều chốt sổ).

---

## Bước 5 — Vai trò trong bài toán nhạc cụ bộ dây

Vì sao hệ thống có RMS Std, ZCR rồi (cũng phát hiện gảy/kéo mạnh mẽ) mà vẫn thèm khát Attack Time? Vì Attack Time không hề nhầm lẫn.

| Cặp nhạc cụ | Tại sao Attack Time quyết định tất cả | Thông số tham khảo |
|---|---|---|
| **Dây gảy (Plucked): Guitar, Harp, Violin Pizzicato** | Phương thức tạo âm cơ học (kéo căng dây rồi buông) khiến năng lượng đạt đỉnh trong tức thời. Không có bất kỳ khoảng trễ nào (zero delay). Dây thép hay dây nilon đạt biên độ cực đại ngay sau khoảnh khắc buông tay. | Attack Time cực ngắn: Tầm `0.01` đến `0.05` giây. |
| **Dây miết (Bowed): Cello Arco, Violin Arco** | Sợi lông đuôi ngựa của cây vĩ cọ xát vào dây đàn. Nó không đẩy dây ra xa nhất rồi buông mà nó "chà" liên tục để ép dây rung động cộng hưởng. Lực rung này cần thời gian để tích lũy dần lên hộp cộng hưởng mới đạt âm lượng tối đa. | Attack Time dài hơn hẳn: Rơi vào khoảng `0.1` đến hơn `0.3` giây. |

*(Đây là lý do nó được vinh dự đặt ở vị trí [0] của Vector Giai Đoạn 1).*

---

## Bước 6 — Câu hỏi giảng viên thường hỏi

### ❓ "Tại sao Attack Time lại bị sai số nếu file âm thanh chưa được cắt gọn (trim) tĩnh lặng đầu file? Các em khắc phục thế nào?"

> **Gợi ý trả lời:** "Dạ thưa thầy/cô, nếu một file âm thanh có 2 giây im lặng hoàn toàn ở đầu (chứa nhiễu nền) và có một cái đỉnh nhiễu nhỏ lỡ vượt qua mức ngưỡng, thuật toán có thể nhận nhầm đó là lúc bắt đầu và cái nốt đàn thực thụ mãi sau mới là đỉnh, dẫn đến kết quả Attack Time bị thành 2.5 giây — một con số sai thực tế hoàn toàn. Để ngăn cản lỗi logic này, quy trình hệ thống bắt buộc phải đi vòng qua bước Tiền xử lý (Pre-processing): Em dùng hàm `librosa.effects.trim` để cắt bỏ sạch sành sanh phần tĩnh lặng (silence) ở 2 đầu file. Tín hiệu đem vào máy tính đo lúc này sẽ là ngay khoảnh khắc nốt đàn vừa cất tiếng."

### ❓ "Nếu bản thu âm là 1 câu nhạc rải hợp âm (Ví dụ: C - E - G) chứ không phải 1 nốt đơn thì Attack Time đo nốt nào?"

> **Gợi ý trả lời:** "Dạ, nếu theo thiết kế đơn giản, hàm thư viện Python sẽ ngầm định quét từ trái sang phải, tìm khoảnh khắc bắt đầu đầu tiên của câu nhạc (ấn nốt C) và kéo dài đồng hồ nhảy số tới tận khi gặp cái đỉnh cao tuyệt đối nhất của toàn bộ câu nhạc ghép lại (có thể nằm tuốt ở nốt G). Điều này không phản ánh đúng Attack Time vật lý của từng nốt gảy. Vì thế ở Giai đoạn chuẩn bị dữ liệu (Dataset), quy định ngầm định là file âm thanh (nhất là nhạc cụ gảy) lý tưởng nhất là các nốt đơn tấu (Single Notes), hoặc nếu là hợp âm thì là một cú bùng nổ đồng loạt (Strumming) để bảo toàn tính chân thực của thông số này."

---

> **TỔNG KẾT GIAI ĐOẠN 1 (37 CHIỀU)**
> Bạn đã hoàn thành việc khám phá toàn bộ **37 chiều** của vector nòng cốt:
> - **[0] Attack Time (1 chiều):** Cắt ngang Gảy và Kéo bằng thời gian đạt đỉnh.
> - **[1-26] MFCC Mean+Std (26 chiều):** Mô phỏng tai người nhận diện Âm sắc, Hộp cộng hưởng, và biến hóa kỹ thuật legato/arpeggio.
> - **[27-33] Spectral Contrast (7 chiều):** Kính lúp soi độ "xước" và độ thuần khiết của sóng hài ở 7 dải tần.
> - **[34] Spectral Centroid (1 chiều):** Cân đo kích thước nhạc cụ (Trầm/Sáng).
> - **[35] ZCR (1 chiều):** Tính số vạch dao động 0 bị chém qua để lọc ra tiếng nhiễu ma sát / gảy đàn.
> - **[36] RMS Std (1 chiều):** Đo dốc âm lượng để quét hành trình truyền lực của nghệ sĩ.

---

**Chúc mừng bạn đã làm chủ Giai đoạn 1! Hệ thống với 37 chiều này hoàn toàn có thể đem đi thi tốt!**
**Bạn có muốn tiếp tục sang 19 chiều mở rộng của Giai đoạn 2 (56 chiều) – Nơi nâng cấp sức mạnh tuyệt đối để phân đôi Viola và Violin không?**
