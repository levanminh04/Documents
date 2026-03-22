# Bài Giảng 07 (GĐ2): Delta MFCC Std — Mức Độ Biến Hóa Âm Sắc

> **Vị trí trong vector (Giai đoạn 2):** Bổ sung thêm 13 chiều
> **Hạng mục:** Chuyên trị các kỹ thuật "Rung" (Vibrato) và "Reo" (Tremolo) kinh điển của nhạc cụ bộ dây.

---

## Bước 1 — Giải thích bằng ngôn ngữ đời thường

### Delta đo cái gì?

Hãy bước vào lại buồng lái ô tô. Ở Giai đoạn 1, MFCC gốc giống như cái **Công tơ mét** báo vận tốc cho bạn biết bạn đang chạy 60km/h ở một thời điểm. 
Nhưng nếu chỉ biết là "tốc độ 60km/h", bạn không biết được chiếc xe đang đi đều đều (Cruise Control), hay đang trong quá trình **đạp lút cán ga tăng tốc**, hay đang **đạp phanh cháy lốp**.

- Sự thay đổi vận tốc đó người ta gọi là Gia tốc (tăng tốc/giảm tốc). 
- Trong xử lý tín hiệu, chữ **Delta (Δ)** chính là toán học đo Sự biến đổi (Vận tốc).

Nếu **MFCC** là "Bức ảnh chụp chân dung âm sắc" tại 1 mili-giây, thì **Delta MFCC** đo đạc xem "Bức ảnh đó đang bị biến dạng nhanh cỡ nào sang khung hình tiếp theo". Một âm thanh trong trẻo, thẳng tuột (như tiếng sáo thổi thẳng cẳng) có Delta rất thấp. Một âm thanh chứa đầy tiếng luyến láy, rung ngón tay trái xảo diệu (Vibrato trên vĩ cầm) khiến âm sắc chớp nháy liên tục → Delta sẽ vọt lên nhảy nhót.

---

## Bước 2 — Giải thích kỹ thuật (đơn giản hóa)

Việc tính Delta MFCC thực chất là phép lấy **đạo hàm** của MFCC theo thời gian.

1. **Chuẩn bị nền móng:** Hệ thống đã phải tính xong toàn bộ mảng 13 hệ số MFCC thô cho mọi khung hình trước đó.
2. **Trừ khung hình:** Đơn giản nhất, hệ thống lấy khung hình đằng sau trừ đi khung hình đằng trước. (Tức là `MFCC[giây_2] - MFCC[giây_1]`).
3. **Lưu trữ vận tốc:** Con số kết quả đại diện cho "Chênh lệch âm sắc trong 1 tíc tắc". Năng lượng âm trầm tăng lên? Âm chói giảm đi? Tất cả được ghi lại dưới dạng Vector vận tốc (Delta).

---

## Bước 3 — Lý do chọn tham số

Trong thực tế code librosa, người ta không lấy n+1 trừ n-1 một cách thô thiển, vì nó sẽ chịu ảnh hưởng cực mạnh bởi một hạt nhiễu nhỏ. Tham số quan trọng ở đây là `width = 9` (Độ rộng cửa sổ toán học):
- Hệ thống sẽ nhìn qua 9 khung hình liên tiếp rồi vẽ một đường xu hướng (quy hồi tuyến tính - Linear Regression) để xem "quán tính âm sắc đang đi lên hay đi xuống".
- Kích thước `9` giúp kết quả vận tốc mượt mà hơn, "tha thứ" cho một hai cọng rác âm thanh nhiễu, nhưng đủ nhạy để bắt kịp nhịp độ tay rền vĩ (Tremolo) của nghệ sĩ (vài nhịp một giây).

---

## Bước 4 — Tại sao Delta lại chép trọn 13 chiều?

Trở lại ví dụ "mô tả khuôn mặt bằng 13 con số" của MFCC bài 1.
- Nếu MFCC[1] là "độ sáng tổng thể", thì Delta MFCC[1] đo "tốc độ tăng độ chói".
- Nếu MFCC[2] là "hình dạng phổ thấp", thì Delta MFCC[2] đo "tốc độ phình to của phổ trầm".
- ...

Vì bản gốc có 13 đặc điểm, thì Delta bắt buộc phải có 13 "vận tốc" đi kèm để giám sát tốc độ thay đổi của từng đặc điểm đó. Do đó, nó thừa kế nguyên vẹn **13 chiều** (curse of dimensionality - đây là lúc hệ thống bắt đầu phình to).

---

## Bước 5 — Tại sao ta lại lấy Std (Độ lệch chuẩn) mà không lấy Mean?

Quy luật vật lý: Âm thanh thay đổi đi lên rồi vòng xuống (chữ V), âm lượng phình to rồi nhỏ lại, tần số vút lên rồi đáp xuống. Sự tăng giảm này bù trừ cho nhau, khiến cho **Trung bình của Delta (Mean) trên toàn bài luôn xấp xỉ bằng 0**. Một con số 0 vô thưởng vô phạt chẳng nói lên điều gì.

Nhưng **Độ lệch chuẩn (Std) của Delta** lại là kho báu!
- Nếu âm thanh thẳng tuột (Guitar phím gảy dứt khoát): Delta chỉ bằng 0, dao động lác đác quanh 0 → **Std cực thấp**.
- Nếu âm thanh rung dạt dào (Violin Vibrato tay trái): Mảng Delta nhảy nhót liên hồi dao động cực lớn từ dương sang âm ngùn ngụt → **Std cực lớn**.

Sử dụng Std của Delta MFCC giúp bắt trọn tính năng "Rung" (Oscillation/Vibrato) bao trùm cả bản nhạc.

---

## Bước 6 — Vai trò trong bài toán nhạc cụ bộ dây

Vì sao ở Giai đoạn 2 ta phải gọi chi viện 13 chiều hùng hậu này? Bởi vì nó cắt đứt một nút thắt cổ chai vô cùng đáng sợ của bộ dây: Nhạc cụ mộc vs Nhạc cụ điện tử.

| Sức mạnh | Lý giải ứng dụng vào đàn dây |
|---|---|
| **Bảo chứng Kỹ thuật Vibrato (Rung dây rung phím)** | Cello Arco và Violin Arco luôn được các nghệ sĩ dùng thuật Vibrato (lắc cổ tay trái trên cần đàn cực nhanh) để nốt nhạc nghe quyến rũ sướt mướt. Thao tác rền này khiến toàn bộ chuỗi hài âm biến hóa chóng mặt (Delta Std vọt lên đỉnh). Ngược lại Harp và Guitar Thùng hầu như không rung tay trái (hoặc rất ít) nên Delta Std tịt ngòi êm ái. |
| **Bắt lỗi Attack / Pizzicato lẫn lộn** | Khi ngón tay móc vào dây kéo căng nảy đứt ra (Pizzicato), âm sắc có một quá trình "chết" biến dạng dốc trượt xuống không phanh (decay âm sắc). Delta MFCC Std sẽ quét được độ dốc khốc liệt đó, cứu trợ nếu thằng Attack Time hay ZCR bị lỗi bắt nhầm. |

---

## Bước 7 — Câu hỏi giảng viên thường hỏi

### ❓ "Nghe rất thần thánh, thế tại sao ngay từ đầu ở Giai đoạn 1 nhóm em không nhét luôn Delta MFCC Std vào mà phải giấu sang Giai đoạn 2?"

> **Gợi ý trả lời:** "Dạ thưa thầy/cô, việc bổ sung Delta MFCC Std nhét ngay một lúc 13 tọa độ mới vào vector. Từ 37 chiều phình lên 50 chiều sẽ khiến mô hình gặp rủi ro Lời nguyền số chiều (Curse of Dimensionality), khiến thời gian tìm kiếm trên PostgreSQL `pgvector` tăng vọt và chi phí tính mảng đạo hàm cũng cao hơn. Bọn em coi Giai đoạn 1 (37 chiều gốc) là "bộ xương" phân biệt trực tiếp vỏ vật lý nhạc cụ. Chỉ khi hệ thống Giai đoạn 1 nhầm lẫn giữa cây đàn A (chơi Vibrato sướt mướt) và cây đàn B (chơi cứng lặp đi lặp lại), bọn em mới cần 'upgrade' mở khóa 'ống kính soi kỹ thuật tay trái' này để dứt điểm sự sai lệch."

---

> **Tổng kết:** Delta MFCC Std là hệ thống định vị quán tính (như la bàn hồi chuyển trên điện thoại), nó không hỏi bạn là ai, nó chỉ hỏi Tốc Độ Thay Đổi của bạn mãnh liệt đến mức nào. Với sự gia nhập của 13 hiệp sĩ này, vector của bạn chính thức chạm ngưỡng 50 chiều. 

---

**Sẵn sàng cho 1 thành viên nhỏ nhưng có võ: Decay Time — 1 chiều chưa?**
