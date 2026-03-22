# Bài Giảng 05: RMS Std — Sự Biến Động Âm Lượng

> **Vị trí trong vector:** Chiều [36] = RMS Std
> **Tổng: 1 chiều** — biến phân định "sự duy trì lực lượng" (nhạc cụ ngân dài vs nhạc cụ nổ chớp nhoáng).

---

## Bước 1 — Giải thích bằng ngôn ngữ đời thường

### RMS Std đo cái gì?

**RMS** (Root Mean Square) là một cách đo đạc chuyên ngành cho thứ mà chúng ta gọi nôm na là **Khối Âm Lượng** (Năng lượng tổng thể to hay nhỏ của âm thanh).

Bây giờ hãy tưởng tượng bạn đang theo dõi chân ga của 2 tài xế ô tô:
1. **Tài xế lái trên đường cao tốc (Violin kéo vĩ):** Họ nhấn chân ga ở mức 60km/h và **giữ nguyên không đổi** trong suốt 10 phút. Bảng vận tốc lúc nào cũng báo giá trị ngang bằng nhau.
2. **Tài xế lái xe đua địa hình (Guitar gảy rải nốt):** Họ rồ ga vọt lên 100km/h rồi lại phải nhả ga cho xe trôi chậm lại 20km/h, lặp đi lặp lại. Bảng vận tốc liên tục nhảy vọt lên cao rồi lại tụt xuống thấp.

Chữ **Std (Standard Deviation - Độ lệch chuẩn)** chính là **"Độ Nhấp Nhô của chân ga"**.

Âm thanh nào có **RMS Std cao** có nghĩa là nó chứa những tiếng nổ, tiếng gảy, tiếng đập... vang lên rất to (vọt ga) rồi vụt tắt ngay lập tức (nhả ga), nối tiếp nhau.
Âm thanh nào có **RMS Std thấp** có nghĩa là nó là một luồng năng lượng được bơm vào liên tục, ổn định không thay đổi (nhấn đều ga).

---

## Bước 2 — Giải thích kỹ thuật (đơn giản hóa)

RMS Std được tính trực tiếp trên miền thời gian (Time Domain) mà không cần biến đổi tần số phức tạp. Quá trình tính như sau:

1. **Chia khung (Framing):** Chia toàn bộ file âm thanh 5 giây thành khoảng hàng trăm khung hình nhỏ xíu (mỗi khung ~93 mili-giây).
2. **Đo Khối lượng (RMS):** Ở mỗi khung, hệ thống tính năng lượng tổng hợp của các mẫu sóng âm (bằng cách bình phương chúng lên, tính trung bình, rồi lấy căn bậc 2). Ta thu được 1 dãy số đại diện cho **"Biểu đồ đường Âm lượng"** chạy dọc theo bài nhạc. Ví dụ: `[0.1,  0.8,  0.4,  0.1,  0.7,  0.2]`.
3. **Đo Độ nhấp nhô (Std):** Máy tính không xuất ra mảng dữ liệu dài thòng này, mà áp dụng công thức **Độ lệch chuẩn (Std)** lên đúng dãy số đó. Nếu dãy số toàn là `[0.5, 0.49, 0.51, 0.5]` (rất ổn định) thì Std sẽ xấp xỉ `0`. Nếu nhảy vọt như `[0.1, 0.8...]` thì Std sẽ rất lớn.

Kết quả thu về chỉ là **1 con số duy nhất** (Độ lệch chuẩn của Âm lượng).

---

## Bước 3 — Lý do chọn tham số

- **Bỏ qua FFT, chọn `frame_size = 2048`:** Giống như ZCR, ta vẫn cắt âm thanh bằng cái khuôn `2048` mẫu. Nếu ta dùng khuôn quá dài (bằng cả bản nhạc), ta sẽ gom chung cả đoạn rồ ga lẫn đoạn nhả ga vào 1 giỏ đánh giá chung, và làm "phẳng" đi mức độ biến thiên âm lượng (những cú nẩy của tiếng gảy đàn sẽ bị san dập). Kích cỡ 2048 đủ nhỏ để bắt trọn từng cú nẩy bật của phím đàn, nhưng đủ lớn để không biến những rung động nhỏ lẻ thành một tiếng nổ.

---

## Bước 4 — Tại sao đặc trưng này có 1 chiều?

Bởi vì dù bản nhạc có chứa bao nhiêu tần số, bao nhiêu hài âm, hay có độ Sáng/Tối phức tạp đến đâu, thì "Tổng Năng lượng phát ra" ở một thời điểm chỉ có **Đúng 1 mốc duy nhất** (được thể hiện bằng một đường lượn sóng trên các phần mềm chỉnh sửa âm thanh). 

Vì chúng ta chỉ đo độ nhấp nhô của đúng 1 đường cong đó trên trục thời gian, kết quả thu được chỉ đơn giản là 1 chỉ số vô hướng mang tên "Mức độ biến thiên năng lượng" (Chiếm 1 chiều).

---

## Bước 5 — Tại sao lại bỏ Mean (Trung bình) mà chỉ lấy Std?

Thường các đặc trưng khác như ZCR, Centroid hay Contrast ta đều lấy **Mean** (tính trung bình các khung hình để làm đại diện). Tại sao riêng RMS ta lại bỏ Mean và ưu tiên **Std** vào Giai đoạn 1?

**Trả lời:** Ghi âm to hay nhỏ là do con người/mic, không phải do nhạc cụ!
- Nếu bạn thu âm một cây Cello đặt mic cách rốn đàn 10cm, vặn Volume 100% → RMS Mean của nó sẽ là `0.8`.
- Cùng cây Cello đó, đặt mic cách 2 mét, Volume 50% → RMS Mean tuột xuống còn `0.2` dù đánh cùng 1 bản nhạc y hệt.

Nếu bạn đưa RMS Mean vào Vector 37 chiều (Giai đoạn 1), hệ thống sẽ bị ảo tưởng rằng cây Cello số 1 và số 2 là hai nhạc cụ khác biệt hoàn toàn! Nhưng **RMS Std** (độ nhấp nhô bên trong dòng thời gian) thì mang tính **bản chất cốt lõi của loại nhạc cụ**, nó không bị đánh lừa bởi hành động vặn nút Volume của người thu âm. Dù nghe nhỏ, tiếng gảy (Pizzicato) vẫn cứ là một ngọn núi có đỉnh và có chân, std vẫn vọt lên.

*(Tuy nhiên sang Giai đoạn 2, thỉnh thoảng RMS Mean vẫn được cho thêm vào 1 chiều phụ trợ nhằm hỗ trợ bù trừ dữ liệu, phần này chưa cần bận tâm).*

---

## Bước 6 — Vai trò trong bài toán nhạc cụ bộ dây

Nếu ZCR là camera soi độ "sắc nhọn/tiếng ồn" của kỹ thuật chơi, thì RMS Std là **camera bắt quả tang "hành trình truyền lực" (Envelope)** của nghệ sĩ lên dây đàn:

| Cặp nhạc cụ | Tại sao RMS Std phân biệt được | Mức độ RMS Std |
|---|---|---|
| **Violin Arco (Kéo vĩ) vs Guitar (Gảy)** | Nghệ sĩ Violin dùng tay phải miết cây vĩ (cây cung) qua lại môt cách liên tục. Lực ma sát và năng lượng được duy trì y nguyên suốt đoạn nốt nhạc (bằng cách tì đều tay). Nghệ sĩ Guitar dùng phím gảy bật tưng cái dây lên một cú chí mạng, sau đó tay phải nghỉ ngơi, để mặc dây đàn tự tắt lịm dần đi. | Vilon Arco (Kéo): RMS Std cực thấp (đường đi ngang). <br> Guitar (Gảy): RMS Std rất cao (dốc lên đỉnh rồi lao dốc). |
| **Piano vs Cello** | (Nguyên lý mở rộng cho sinh viên) Piano là một loại gõ-dây (búa gõ vào dây thép rồi nảy ra, y như gảy), Cello cọ vĩ. | Piano luôn có RMS Std cao hơn mọi loại cọ vĩ. |

---

## Bước 7 — Câu hỏi giảng viên thường hỏi

### ❓ "Tại sao em đẩy phần Âm lượng (RMS Mean) ra hệ thống ngoài, mà lại nhét độ lệch chuẩn của Âm lượng (RMS Std) vào cốt lõi 37 chiều?"

> **Gợi ý trả lời:** "Dạ thưa thầy/cô, độ lớn trung bình của âm lượng (Mean) phụ thuộc hoàn toàn vào quá trình con người thu âm (vặn mic to/nhỏ). Nó là một 'cạm bẫy' giống như Tần số cơ bản F0 vậy, nếu dùng nó thì hệ thống đi đánh giá cách người thu âm phối khí chứ không đánh giá bản chất của nhạc cụ. Ngược lại, RMS Std phản ánh **cách thức truyền năng lượng vật lý** (âm hưởng Envelope). Dây đàn được gảy một phát xong bỏ lửng (Piano, Guitar, Pizzicato Violin) sẽ tạo ra mức độ biến thiên năng lượng rất cao. Dây đàn được kéo vĩ duy trì ma sát (Arco Cello/Violin) sẽ giữ vững năng lượng không đổi. Vì thế RMS Std là đặc trưng bắt buộc để máy tính đoán được kỹ thuật kích âm (Excitation Method)."

### ❓ "Có trường hợp nào Cello kéo vĩ (Arco) mà RMS Std vẫn vọt lên rất cao không?"

> **Gợi ý trả lời:** "Dạ có ạ, đó là khi nghệ sĩ kéo các nốt cực ngắn có ngắt nghỉ (kỹ thuật Staccato hoặc Spiccato - nảy vĩ). Lúc này tuy là kéo vĩ nhưng cường độ tắt bật liên tục, làm đường âm lượng nhấp nhô y hệt như đang gảy. Trong trường hợp đó RMS Std sẽ đánh đồng kỹ thuật này với kỹ thuật gảy thông thường, nhưng Vector 37 chiều của chúng em còn các vũ khí khác như ZCR, MFCC để bóc tách sự khác biệt phía sau sự nhấp nhô đó."

---

> **Tổng kết:** RMS Std đóng góp **1 chiều** mang ý nghĩa vật lý rành mạch nhất trong toàn bộ list đặc trưng. Bất cứ khi nào bạn nghe thấy một nốt nhạc vang lên mãnh liệt rồi chìm vào tự suy tàn không ai can thiệp (Nhạc cụ gảy/gõ), máy tính sẽ ghi lại một điểm RMS Std rất lớn. Đây là mấu chốt cắt đôi thế giới nhạc cụ ra làm 2 cực rõ rệt.

---

**Bạn đã sẵn sàng sang đặc trưng cuối cùng của Giai đoạn 1 (Attack Time — 1 chiều) chưa?**
