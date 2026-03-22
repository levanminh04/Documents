# Bài Giảng 09 (GĐ2): Spectral Flatness — Độ Phẳng Của Phổ Nhạc

> **Vị trí trong vector (Giai đoạn 2):** Bổ sung thêm 1 chiều.
> **Hạng mục:** Chuyên gia lùng sục "Hạt nhiễu" và "Tiếng khò khè ma sát" đặc trưng của vĩ cầm.

---

## Bước 1 — Giải thích bằng ngôn ngữ đời thường

### Spectral Flatness đo cái gì?

Hãy tưởng tượng bạn đang bay ngang qua một vùng đất:
- **Trường hợp 1 (Khuông nhạc thuần túy):** Gồm những đỉnh núi chót vót xen kẽ những vực sâu thăm thẳm. Âm thanh này là tiếng nốt nhạc trong vắt lảnh lót (như tiếng Guitar, tiếng đàn Harp gảy). Các năng lượng âm thanh xếp gọn gàng vào từng ngọn núi (hài âm/nốt nhạc).
- **Trường hợp 2 (Tiếng nhiễu ồn):** Cảnh quan là một cánh đồng cỏ bằng phẳng, chỗ nào cũng cao ngang ngang nhau. Âm thanh này giống như tiếng tivi mất sóng (tiếng "xìiiii"), tiếng quạt máy, đi bộ đạp lá khô, hoặc tiếng chà xát đinh tai nhức óc.

Máy tính gọi cánh đồng chiều ở Trường hợp 2 là **"Flatness" (Sự phẳng lì)**.

- **Flatness càng HIGHT (gần 1):** Càng giống tiếng nhiễu, tiếng ma sát ràn rạt, rải rác đều ở mọi tần số.
- **Flatness càng LOW (gần 0):** Càng giống tiếng nhạc điệu có cao độ sắc nét, có nhạc tính thuần túy.

---

## Bước 2 — Giải thích kỹ thuật (đơn giản hóa)

Làm sao máy tính phân biệt được Đồi núi và Đồng cỏ? Nó dựa vào một mẹo toán học kinh điển so sánh 2 loại "Trung bình cộng":

1. **Vẽ Bản Đồ (FFT):** Đầu tiên, sử dụng FFT để tạo ra biểu đồ cột tần số (giống hệt các thuật toán trước).
2. **Loại Trung bình 1 — Trung bình cộng (Arithmetic Mean):** Cộng tất cả các cột vào rồi chia cho tổng số cột. Nếu có 1 ngọn núi siêu khủng khiếp, trung bình cộng này sẽ bị kéo vọt lên (tương tự như lương trung bình bị lệch vì 1 ông giám đốc tỷ phú).
3. **Loại Trung bình 2 — Trung bình nhân (Geometric Mean):** Nhân tất cả các cột vào rồi lấy căn. Toán học chỉ ra rằng, Trung bình nhân rất nhạy cảm với các Vực sâu (chỉ cần 1 cột bằng 0 là cả kết quả về 0).
4. **Việc của Spectral Flatness:** Lấy **Trung bình nhân chia cho Trung bình cộng**. 

Kết quả cực diệu kỳ:
- Nếu bản đồ toàn là những ngọn núi nhấp nhô (Nốt nhạc): Phép chia này sẽ ép kết quả về sát con số **0**.
- Nếu bản đồ bằng phẳng lì như đồng cỏ (Tiếng nhiễu xì): Hai cái Trung bình này bằng nhau, phép chia dắt tay nhau về con số **1**.

---

## Bước 3 — Lý do chọn tham số

- Đặc trưng này không cần phải nặn óc lựa tham số, nó bám theo cài đặt `frame_size=2048` truyền thống của phép biến đổi FFT như 1 hệ quả tất yếu.
- Điểm mấu chốt là **tính chất của công thức toán**: Kết quả luốn tự động bị "giam" trong lồng kính từ **0 đến 1**, bất kể bài nhạc thu âm to hay nhỏ. Điều này giúp tính toán chuẩn hóa rất dễ thở.

---

## Bước 4 — Tại sao đặc trưng này có 1 chiều và lại dùng Mean?

- Bản chất nó là một "phép chia" của khung hình (Toàn bộ mảng tần số chia ra được Đúng 1 tỷ số). Vậy nên nó chỉ sinh ra **1 con số vô hướng (1 chiều)**.
- Vì âm lượng và kỹ thuật thay đổi theo thời gian, ta sẽ tính hàng trăm cái Flatness cho hàng trăm cái frame. Sau đó ép dẹt lại thành **Mean Flatness (Trung bình độ phẳng)**. 
- Chỉ số Mean Flatness trả lời định kiến: *"Cả cuộc đời bài hát này, nó nghe giống âm thanh máy hút bụi (1) hay nghe giống tiếng chim lảnh lót (0)?"*.

---

## Bước 5 — Vai trò trong bài toán nhạc cụ bộ dây

Không có họ nhạc cụ nào tạo ra thứ hạt nhiễu ma sát quyến rũ ngang bằng họ Vĩ cầm. Spectral Flatness sinh ra để bắt lỗi cái này:

| Nhạc cụ | Giải phẫu âm thanh bằng Flatness | Điểm số Flatness |
|---|---|---|
| **Violin / Cello (Kéo Vĩ)** | Nghệ sĩ buộc phải bôi nhựa thông (rosin) lên sợi vĩ (lông ngựa). Sự cọ xát của nhựa thông lên dây thép tạo ra một thứ tiếng "rít" xè xè (Friction noise) hòa quyện chặt chẽ vào nốt nhạc phát ra. Dải ma sát này bị máy tính coi là "Nhiễu trắng", trải rợp lên dãy núi thành một đám mây bằng phẳng. | Flatness cao hơn bình thường (Thường > 0.05 hoặc 0.1) |
| **Guitar / Harp (Gảy dây)** | Vì gảy bằng ngón tay trơn tru hoặc phím gảy, nên ngoại trừ tiếng nổ bùm ở tích tắc đầu tiên, toàn bộ đuôi âm thanh sau đó là các hài âm tự nhiên căng mọng trong vắt tuyệt đối mà không có lực kéo nào ma sát thêm vào. | Flatness cực thấp (Siêu mượt, thường xuyên ép về < 0.01) |

---

## Bước 6 — Câu hỏi giảng viên thường hỏi

### ❓ "Ủa khoan, hệ thống Giai đoạn 1 của các em đã có Spectral Contrast (Độ tương phản) tính chênh lệch Đỉnh-Đáy rồi cơ mà? Tại sao nay lại còn tòi ra Spectral Flatness cũng tính Đỉnh-Đáy này? Bị thừa dữ liệu đúng không?"

> **Gợi ý trả lời:** "Dạ thưa thầy/cô, câu hỏi rất sắc bén ạ nhưng hai đặc trưng này không hề mâu thuẫn. **Spectral Contrast** được tính chi ly cho 7 dải tần tách biệt (từ trầm lên bổng), nó soi vào hình dạng gồ ghề của từng mảnh phổ một. Còn **Spectral Flatness** thì đúc kết lại bằng một hệ số duy nhất (tỷ lệ Trung bình nhân) càn quét tổng thể lên Toàn bộ phổ tần. Thực tế, tiếng ma sát của nhựa thông trên cây Vĩ (Rosin noise) là thứ Nhiễu siêu băng rộng rải đều khắp mọi dải tần — Spectral Contrast vì bị xé nhỏ ra 7 dải nên đôi lúc bị mù/bắt trượt cái tính chất trải thảm này, trong khi Flatness thì gom tất cả sự rối ren đó lại và nhảy số phát một. Chúng nâng đỡ nhau thay vì cản đường ạ."

---

> **Tổng kết:** Bạn vừa thu thập thêm 1 chiều kích phân hóa Cấu tạo vật lý. Âm thanh mộc mạc của Gảy và Cọ Xát không chỉ khác nhau ở khởi đầu (Attack Time) mà còn rành rọt ở sự "bụi bặm" trong tần số.

---

**Bạn đã sẵn sàng tới với bộ đôi đo lường hình dạng quang phổ: Spectral Rolloff và Spectral Bandwidth (Gồm 2 chiều) chưa?**
