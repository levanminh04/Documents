# Bài Giảng 04: ZCR (Zero-Crossing Rate) — Tốc Độ Đổi Dấu

> **Vị trí trong vector:** Chiều [35] = ZCR (Mean)
> **Tổng: 1 chiều** — biến phân định độ "sắc nhọn" và "nhiễu" của âm thanh dựa trên số lần dao động.

---

## Bước 1 — Giải thích bằng ngôn ngữ đời thường

### ZCR đo cái gì?

Hãy tưởng tượng bạn đang lái xe trên một con đường thẳng có vạch kẻ liền ở chính giữa (đường phân cách 0). 
- Nếu bạn lái xe **rất điềm tĩnh, mượt mà**, chiếc xe của bạn cứ đi thẳng tắp một bên lề, rất hiếm khi cán qua vạch giữa đường.
- Nếu bạn là một tay đua **liên tục lạng lách, đánh võng** từ trái sang phải rồi lại từ phải sang trái, chiếc xe của bạn sẽ cán qua vạch giữa đường liên tục.

Trong âm thanh, sóng âm dao động lên xuống quanh một trục `0` (lúc thì màng loa đẩy ra ngoài tạo áp suất dương, lúc thì màng loa thụt vào trong tạo áp suất âm). 
- **ZCR (Zero-Crossing Rate)** chính là việc túc trực ở vạch giữa đường và **đếm xem chiếc xe lạng qua lạng lại bao nhiêu lần** trong một giây. 

Âm thanh nào mượt mà, êm ái, ngân dài ổn định (như tiếng ca sĩ hát ngân nga, tiếng kéo vĩ từ tốn) thì sóng âm lên xuống rất đều đặn, chậm rãi → **ZCR thấp (trên đường ít vạch mốc bị cán).**
Âm thanh nào gắt, sắc nhọn, hoặc chứa nhiều tiếng ồn/tiếng gõ (như tiếng chũm chọe xèo xèo, tiếng móng tay gảy bật vào dây thép) thì sóng âm giật cục, đánh võng liên tục không theo quy luật → **ZCR vọt lên rất cao.**

---

## Bước 2 — Giải thích kỹ thuật (đơn giản hóa)

Khác biệt lớn nhất của ZCR so với MFCC hay Spectral Centroid là: **Nó không cần dùng đến biến đổi FFT (không cần chuyển sang miền tần số).** Nó làm việc trực tiếp trên sóng âm thô nguyên bản (Miền thời gian - Time Domain).

Quy trình cực kỳ đơn giản:
1. **Chia khung (Framing):** Lấy một đoạn âm thanh ngắn (frame). Bản chất đoạn âm thanh này là một dãy các con số biên độ, ví dụ: `[0.1,  0.5,  -0.2,  -0.4,  0.3]`.
2. **Đếm số lần đổi dấu:** Lập trình viên cho máy tính duyệt qua dãy số này. Cứ mỗi lần con số chuyển từ khoảng dương (`+`) sang khoảng âm (`-`) hoặc ngược lại, máy đếm lên 1.
   *(Ví dụ trên: 0.5 xuống -0.2 là 1 lần cắt; -0.4 lên 0.3 là lần cắt thứ 2. Tổng có 2 lần cắt vạch không).*
3. **Tính tỷ lệ:** Lấy tổng số lần đã đếm chia cho tổng số con số trong khung hình đó. Ta được một tỷ lệ `Rate` (ví dụ: 0.05).

---

## Bước 3 — Lý do chọn tham số

Biến số duy nhất ta cần quan tâm ở đây là độ dài của khung hình `frame_size` mà ta dùng để "đếm" (thường tương thích với hệ thống gốc, vẫn là `2048` mẫu):

- **Tại sao không đếm trên độ dài tùy ý?** Nếu đếm trên một đoạn quá ngắn (vd 10 mẫu), số lần đổi dấu có thể bằng 0 hoàn toàn, hoặc vọt lên bằng 100%, tỷ lệ này nhảy múa liên tục không mang tính thống kê. Khung `2048` mẫu (khoảng 93 mili-giây) cung cấp một kích cỡ mẫu "đủ lớn" để số lượng lạng lách của âm thanh mang ý nghĩa một tỷ lệ phần trăm ổn định, mà không quá dài khiến ta lỡ mất những đoạn gảy chớp nhoáng của dây đàn.

---

## Bước 4 — Tại sao ZCR lại có 1 chiều?

Tỷ lệ (Rate) tự bản thân nó chỉ là một phép toán chia cơ bản: `Số lần cán vạch / Tổng số điểm`. Nó trả về trực tiếp một con số vô hướng. 
Không có chia cắt dải tần số (như Spectral Contrast - 7 dải), không có khái niệm đa chiều (như MFCC - 13 hệ số), ZCR nhìn nhận đoạn âm thanh như một thể thống nhất và đưa ra một điểm số duy nhất cho "độ lạng lách" của nó. Do đó, ZCR chiếm đúng 1 chiều rành mạch.

---

## Bước 5 — Tại sao tính Mean (Trung bình) của ZCR?

Trong hệ thống Giai đoạn 1 của đồ án (37 chiều), chúng ta đang lưu lại **Mean ZCR** cho toàn bộ file âm thanh.

Khi bạn đánh một nốt đàn Guitar:
- Giai đoạn **Attack (Lúc ngón tay/phím gảy bật vào dây đàn):** Âm thanh va chạm sinh ra rất nhiều nhiễu hạt sắc bén, ZCR ở khung hình này sẽ vọt lên khoảng `0.3` (30%).
- Giai đoạn **Decay & Sustain (Lúc dây đàn bắt đầu ngân tự do):** Âm thanh mượt dần lại thành các hình sin lặp lại, ZCR hạ xuống và ổn định ở khoảng `0.05` (5%).

Khi ta tính Mean (cộng tất cả các khung hình lại chia đều), con số Mean này sẽ tóm tắt lại "sự kiện" của toàn bộ bài. Một file Guitar với nhiều nốt rải (arpeggio) sẽ có các đỉnh ZCR xuất hiện liên tục, đẩy Mean ZCR lên cao. Trong khi đó, một file Violin kéo vĩ dài 5 giây chỉ có một lúc bắt gắt ở đầu, sau đó êm ái hoàn toàn, kéo Mean ZCR xuống thấp.

---

## Bước 6 — Vai trò trong bài toán nhạc cụ bộ dây

ZCR là "camera" chuyên dùng để bắt quả tang **kỹ thuật chơi (Technique)** và **cách kích âm (Excitation Method)**:

| Cặp nhạc cụ | Tại sao ZCR phân biệt được | Mức độ ZCR |
|---|---|---|
| **Violin Arco (Kéo vĩ) vs Violin Pizzicato (Gảy)** | Cùng trên một cây đàn, độ Sáng (Centroid) hoặc Hộp cộng hưởng (MFCC) rất giống nhau. Làm sao máy biết nốt nhạc vừa được phát ra bằng cách nào? ZCR sẽ là cứu tinh! Tiếng gõ/gảy bật vào dây thép chứa tín hiệu nhiễu băng rộng (rất lạng lách). Tiếng vĩ cọ xát tuy cũng có nhiễu nhưng nhanh chóng bị sóng hài cưỡng bức lấn át. | Pizzicato (Gảy): ZCR cao hơn rõ rệt. <br> Arco (Kéo): ZCR thấp hơn. |
| **Acoustic Guitar vs Classical Harp** | Guitar gảy bằng phím (Pick) hoặc móng tay dài tạo ra va chạm rất cứng dứt khoát. Harp gảy bằng phần thịt đầu ngón tay mềm mại (Flesh pluck) tạo âm êm hơn nảy hơn. | Nhờ độ "sắc" của móng/phím, ZCR của Guitar sẽ nhỉnh hơn Harp. |

---

## Bước 7 — Câu hỏi giảng viên thường hỏi

### ❓ "Tại sao âm thanh 'chói' hoặc 'nhiễu' lại có ZCR cao hơn? Giải thích theo cách em hiểu?"

> **Gợi ý trả lời:** "Dạ thưa thầy/cô, âm thanh 'chói' hoặc 'nhiễu' bản chất là do sự kết hợp của rất nhiều các tần số cao. Tần số cao nghĩa là sóng âm dao động xoắn tít, lên xuống cực kỳ nhanh trong một khoảng thời gian ngắn (như lò xo bị nén chặt). Vì nó đi lên đi xuống quá nhanh đan xen nhau, nó bắt buộc phải 'cán dải phân cách 0' nhiều lần hơn. Ngược lại, một tần số trầm giống như sóng biển thoai thoải, rề rà mất một lúc lâu mới chìm xuống dưới nấc 0. Nên ZCR vô tình tỷ lệ thuận trực tiếp với lượng tần số cao dầy đặc hoặc nhiễu có trong âm thanh."

### ❓ "MFCC tính bằng miền tần số (biến đổi FFT), còn ZCR tính bằng miền nào? Ưu điểm của việc này?"

> **Gợi ý trả lời:** "Dạ, trong khi MFCC, Centroid, Contrast đều nằm ở miền tần số (Frequency Domain) phụ thuộc vào FFT, thì ZCR lại hoàn toàn nằm ở **Miền thời gian (Time Domain)**. Nó làm việc trên mảng biên độ thô nguyên bản của âm thanh. Ưu điểm siêu việt của việc này là **tính toán cực kì nhẹ và chi phí gần bằng 0**. Kẹp nó vào vector là một món hời lớn: tốn rất ít CPU để trích xuất nhưng lại mang về giá trị phân định gảy/kéo (technique) cực rành mạch."

---

> **Tổng kết:** Bạn đã bổ sung thêm 1 chiều kích vô cùng thực dụng. Nếu Spectral Centroid đo "Kích thước" của ca sĩ, thì ZCR đo xem "Ca sĩ hát mượt hay đang đọc Rap gắt gỏng". ZCR không tốn nhiều công sức để tính, nhưng là liều thuốc giải hoàn hảo khi MFCC bó tay trước sự phân biệt giữa gảy và kéo trên cùng một loại nhạc cụ. 

---

**Bạn đã sẵn sàng sang đặc trưng tiếp theo (RMS Std — 1 chiều) chưa?**
