# Bài Giảng 03: Spectral Centroid — Trọng Tâm Phổ (Độ Sáng Của Âm Thanh)

> **Vị trí trong vector:** Chiều [34] = Spectral Centroid (Mean)
> **Tổng: 1 chiều** — biến phân định "sáng" (treble) và "tối/ấm" (bass).

---

## Bước 1 — Giải thích bằng ngôn ngữ đời thường

### Spectral Centroid đo cái gì?

Hãy tưởng tượng một cái **bập bênh (seesaw)** ở công viên. Trục bập bênh được đặt lên một thanh đo dài từ 0 đến 100.
Thay vì trẻ em ngồi lên, bây giờ các **tần số âm thanh** sẽ nhảy lên bập bênh:
- Tần số trầm (bass) nặng nề sẽ ngồi bên trái.
- Tần số cao (treble) nhẹ nhàng nhưng sắc nhọn sẽ ngồi bên phải.

Nếu âm thanh có rất nhiều bass gầm gừ, bập bênh sẽ nghiêng hẳn sang trái. Để giữ thăng bằng, bạn phải dời trục bập bênh về sát bên trái (về phía âm trầm). Trái lại, nếu âm thanh chói tai đầy tiếng xì xèo, bạn phải kéo trục bập bênh sang phải.

**Tọa độ của trục bập bênh đó chính là Spectral Centroid (Trọng tâm phổ).** 

- Trọng tâm **cao** = bập bênh nghiêng phải = âm thanh **chói, sáng** (Violin).
- Trọng tâm **thấp** = bập bênh nghiêng trái = âm thanh **ấm, đục, trầm** (Cello, Contrabass).

---

## Bước 2 — Giải thích kỹ thuật (đơn giản hóa)

Spectral Centroid trả về một con số duy nhất có đơn vị là **Hertz (Hz)**. Nó được tính theo các bước sau:

1. **Từ âm thanh ra biểu đồ (FFT):** Đầu tiên, ta trích xuất khung hình (frame) và dùng biến đổi FFT để ra biểu đồ tần số: trục ngang là tần số Hz (từ 0 – 11000Hz), trục dọc là chiều cao năng lượng (âm lượng).
2. **Xác định vị trí các "đứa trẻ":** Mỗi thanh bar tần số đóng vai trò là một "đứa trẻ". Tọa độ của nó là chỉ số Hz, còn cân nặng của nó là chiều cao năng lượng.
3. **Tính trung bình cộng có trọng số (Weighted Average):** Hệ thống nhân (Tọa độ Hz) × (Năng lượng) của tất cả cột tần số lại, rồi chia cho tổng Năng lượng.
4. **Kết quả:** Ta được duy nhất 1 con số Hz đại diện cho "điểm thăng bằng". Mặc dù kết quả ví dụ là `2500Hz`, nhưng **không có nghĩa là nốt nhạc đó đang kêu ở 2500Hz**, nó chỉ là tọa độ cân bằng của toàn bộ biểu đồ mà thôi.

---

## Bước 3 — Lý do chọn tham số

Chúng ta thực hiện trích xuất Centroid dựa trên biến đổi FFT, vậy nên các tham số liên quan là `frame_size` và `hop_size`:

- **`frame_size = 2048`:** Tham số "quốc dân" ở tần số lấy mẫu 22050Hz. Nó gom đủ ~93ms âm thanh. Nếu frame quá nhỏ, độ phân giải tần số sẽ bị kém (biểu đồ quá thô, bập bênh bị chia làm quá ít bậc đo) dẫn đến việc định vị tọa độ trọng tâm không chính xác.
- **Tần số lấy mẫu `sr = 22050`:** Giới hạn phổ đo được sẽ là một nửa (Nyquist Frequency) tức là 11025 Hz. Con số này là vừa vặn vì các hài âm quan trọng nhất của nhạc cụ dây (dù là Violin chói nhất) đều kết thúc trước vạch 11000 Hz. Nếu dùng 44100 Hz, dải vạch sẽ kéo dài đến 22000Hz (chủ yếu là nhiễu siêu âm) và có xu hướng "kéo" sai lệch cái bập bênh sang bên phải.

---

## Bước 4 — Tại sao Spectral Centroid chỉ có 1 chiều?

Quay lại hình ảnh cái bập bênh: Một cái bập bênh vật lý chỉ có **một điểm duy nhất** để cắm trục thăng bằng. 

Bất kể trong bản nhạc có phức tạp đến đâu — có tiếng Cello đang lót nền cộng thêm tiếng Violin đang réo rắt — thì biểu đồ tổng hợp vẫn chỉ có **một trung bình cộng tổng thể**. Quá trình toán học "trung bình có trọng số" luôn triệt tiêu mọi chiều dư thừa và kết tụ về đúng **một con số** (Hz) cho mỗi khung hình (frame). Do đó, nó chỉ chiếm 1 chiều trong không gian vector.

---

## Bước 5 — Tại sao lại lấy Mean trên toàn file?

Giống như các đặc trưng khác, khi tính Centroid cho một file âm thanh kéo dài 5 giây, ta thu được hàng chục tọa độ dao động qua lại theo từng mili-giây.
- Ví dụ giây thứ 1 (kéo nhẹ): Trọng tâm 1500Hz.
- Giây thứ 2 (kéo dồn dập, gắt): Trọng tâm vọt lên 3500Hz.

Ta thực hiện thao tác **ép dẹt (tính Mean)** để thu về đúng 1 giá trị duy nhất (Ví dụ: `Mean = 2100Hz`). 
Nó đại diện cho câu hỏi: *"Nhìn chung bản ghi âm bài hát này, âm thanh nghiêng về phe Sáng hay phe Tối?"*. Điều này là quá đủ để phân định "kích cỡ" của nhạc cụ. Giai đoạn 1 chưa cần chiều tĩnh/động phức tạp, 1 con số đánh giá tổng quang là đủ tốt.

---

## Bước 6 — Vai trò trong bài toán nhạc cụ bộ dây

Vì Spectral Centroid đánh giá độ "Sáng/Tối", nó là **máy quét kích thước hộp cộng hưởng** tuyệt vời nhất. Kích thước thùng đàn quyết định đặc điểm này rõ nhất trong bộ dây:

| Cặp nhạc cụ | Tại sao Centroid phân biệt được | Điểm thăng bằng nằm ở đâu? |
|---|---|---|
| **Violin vs Cello** | Violin có thùng nhỏ, dây căng, tạo ra các hài âm bậc cao cực kỳ chói tai. Cello thùng to, dây dày, cộng hưởng tốt âm trầm. | Violin: Thường dao động 2000 – 3000 Hz (Nghiêng phải). Cello: Thường dao động 400 – 1000 Hz (Nghiêng trái). |
| **Viola vs Violin** | Hai nhạc cụ này rất hay bị nhầm lẫn vì ngoại hình sinh đôi. Nhưng Viola to hơn Violin 1 chút, tạo ra âm xỉn và u tối hơn. | Trọng tâm Viola sẽ có xu hướng thấp hơn Violin ~500Hz cho cùng một nốt nhạc dải trung. |

*Lưu ý:* Centroid kém hiệu quả nếu lấy 1 cây Violin chơi ở nốt siêu trầm so sánh với 1 cây Cello chơi ở nốt siêu bổng. (Dù vậy, bộ MFCC 26 chiều sẽ cứu cánh ở trường hợp này).

---

## Bước 7 — Câu hỏi giảng viên thường hỏi

### ❓ "Tại sao không dùng luôn Tần số cơ bản (F0 / nốt nhạc đang chơi) mà lại vác Spectral Centroid vào vector làm gì?"

> **Gợi ý trả lời:** "Dạ thưa thầy/cô, F0 đi copy nốt nhạc chứ không nhận dạng nhạc cụ. Nếu một cây Violin và một cây Guitar cùng chơi nốt 'La' (440Hz), thì F0 của chúng giống hệt nhau (440Hz). Nhưng Centroid của Violin sẽ nằm ở ~2500Hz (do Violin có nhiều âm bồi phụ ở tầng cao làm tiếng sắc gọn), trong khi Guitar trầm ấm hơn nên Centroid chỉ ~1200Hz. Nên Centroid bắt được bản chất âm sắc vật lý (độ sáng), còn F0 chỉ bắt được nốt nhạc."

### ❓ "Nếu âm thanh có chứa tiếng ồn nền (như tiếng xì của quạt, tiếng gió) thì Centroid có bị sai lệch không?"

> **Gợi ý trả lời:** "Dạ có ạ. Tiếng ồn thường chứa rất nhiều tần số siêu cao vô trật tự (nhiễu trắng). Đám nhiễu này giống như một đám trẻ con nhẹ cân nhưng ngồi ở tận cùng bên phải của cái bập bênh, sẽ tạo mô-men lực kéo lệch trọng tâm của cả bản nhạc lên mức cao ảo. Đó là một trong những lý do tập dữ liệu gốc cần tiền xử lý làm sạch (trim) và ta cần MFCC để bù đắp, vì MFCC ít bị ảnh hưởng bởi nhiễu hơn."

---

> **Tổng kết:** Bạn đã nạp thêm 1 chiều kích cho vector. Spectral Centroid hoạt động như một hệ thống "cân đo kích thước nhạc cụ" ẩn, định vị nhanh chóng xem hệ thống đang phải đối mặt với một nhạc cụ bé hạt tiêu (sáng bừng) hay một gã khổng lồ (trầm đục).

---

**Bạn đã sẵn sàng sang đặc trưng tiếp theo (ZCR — 1 chiều) chưa?**
