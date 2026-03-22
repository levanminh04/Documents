# Bài Giảng 08 (GĐ2): Decay Time — Thời Gian Suy Tàn

> **Vị trí trong vector (Giai đoạn 2):** Bổ sung thêm 1 chiều
> **Hạng mục:** "Đồng hồ bấm giờ phần hậu" — Mảnh ghép hoàn thiện cho Attack Time.

---

## Bước 1 — Giải thích bằng ngôn ngữ đời thường

### Decay Time đo cái gì?

Nếu Attack Time (GĐ1) đo thời gian "cất tiếng", thì Decay Time đo thời gian **"vụt tắt"**.

Hãy tưởng tượng bạn đang chạy đà một chiếc xe đạp rồi thả hai chân ra không đạp nữa.
- **Cách 1 (Đàn Harp/Guitar - Ngân vang):** Chiếc xe cứ thế trôi đi theo quán tính, bánh xe lăn chầm chậm, chầm chậm mãi rồi mới chịu dừng lại hoàn toàn. → **Decay Time kéo rất dài**.
- **Cách 2 (Violin Pizzicato - Cụt lủn):** Bạn vừa thả chân thì bóp phanh cái "két", xe khựng lại ngay lập tức. → **Decay Time cực ngắn**.

Trong âm thanh, **Decay Time** là chiếc đồng hồ bấm giờ đo khoảng thời gian từ lúc "âm lượng đạt đỉnh to nhất" cho đến khi "âm thanh chìm vào yên lặng". Âm thanh nào có độ cộng hưởng tốt, dây đàn rung tự do lâu thì tốn nhiều thời gian suy tàn. Âm thanh nào bị hãm lại hoặc hết lực tác động thì tàn dư kết thúc ngay chớp mắt.

---

## Bước 2 — Giải thích kỹ thuật (đơn giản hóa)

Tương tự mặt sấp ngửa của đồng xu, quy trình đo Decay Time dùng chung biểu đồ Âm lượng (RMS) y hệt như Attack Time nhưng đo chiều xuống dốc:

1. **Chuẩn bị đồ thị:** Máy tính nhìn vào lại biểu đồ âm lượng (RMS) của file.
2. **Xuất phát từ Đỉnh (Peak):** Tìm lại cái đỉnh núi lúc nãy vừa dừng bấm giờ ở bài Attack Time. Đặt điểm xuất phát tại đây.
3. **Tìm Điểm kết thúc (Offset):** Máy tính trượt dài theo sườn núi đi xuống, cho đến khi âm lượng chạm một vạch "tắt ngúm" (ví dụ: mốc 10% hoặc 1% so với đỉnh núi cao nhất). Bấm dừng đồng hồ tại đây.
4. **Phép trừ cơ bản:** `Decay Time (giây) = Thời gian tại Dưới chân núi - Thời gian tại Đỉnh núi`.

---

## Bước 3 — Lý do chọn tham số

Tham số mấu chốt ở đây là **Mức ngưỡng ngắt (Offset Threshold)**, ví dụ `10%`:
- **Tại sao không đo tới tận mức im lặng hoàn toàn `0%`?** Vì âm thanh trong thực tế không bao giờ có thể chạm đáy 0 tuyệt đối! Luôn có tiếng vọng của căn phòng (reverb), tiếng rè của amply, hoặc nhiễu môi trường dội lại. Nếu bắt máy tính đợi đến khi âm lượng về đúng số 0, nó có thể chờ đến... vô tận.
- Vạch `10%` được dân DSP (xử lý tín hiệu số) dùng làm mốc tiêu chuẩn: Đó là khoảnh khắc mà tai người cũng gần như hết cảm nhận được nốt nhạc, và bỏ qua đám nhiễu nền kéo rê phía sau.

---

## Bước 4 — Tại sao đặc trưng này có 1 chiều và không tính Mean/Std?

Cũng giống như ông bạn thân Attack Time, "thời gian trôi bánh xe từ A đến B" là một độ dài vật lý đo bằng đơn vị giây.
- Nó chỉ diễn ra có **một lần** xuyên suốt nốt nhạc (Từ lúc căng nhất đến lúc tàn).
- Nó không có tính chu kỳ, không lượn sóng nhấp nhô để sinh ra 400 mặt cắt thời gian.

Vì máy tính chỉ đo 1 khoảng cách, nó trả về đúng 1 giá trị vô hướng (Ví dụ `1.5 giây`). Ta ghép thẳng con số này vào thành **1 chiều rành mạch** trong vector 56 chiều của hệ thống, không cần gọt dũa Mean hay Std.

---

## Bước 5 — Vai trò trong bài toán nhạc cụ bộ dây

Attack Time phân biệt xuất sắc "Phe Gảy" (nhanh) và "Phe Kéo" (chậm). Thế nhưng nó bị "mù" khi phân biệt nội bộ trong phe Gảy. Đây là chỗ Decay Time bước lên xưng vương:

| Cặp nhạc cụ | Tại sao Decay Time giải được bài toán này | Thông số tham khảo |
|---|---|---|
| **Violin Pizzicato vs Classical Harp** | (Bài toán kinh điển). Một người móc ngón tay vào dây thép đàn Violin bé tí. Quả Attack Time vụt lên nhanh như chớp. / Một người gảy ngón tay vào dây ruột đàn Harp khổng lồ. Quả Attack Time cũng vụt lên y hệt! Nhưng... dây Harp dài nửa mét, được thả rông không hãm thanh. Dây đàn cứ thế dao động oang oang cả một khoảng chục giây. Dây đàn Violin gõ cái "tịt" ngúm luôn sau chớp mắt. | Harp: Decay rất dài `(> 2.0s)`. <br> Violin Pizz: Decay cực cụt `(< 0.5s)`. |
| **Acoustic Guitar vs Classical Harp** | Guitar thùng được thiết kế để tắt nhanh hơn Harp, dành chỗ cho các hợp âm nảy. Harp sinh ra để cộng hưởng ướt át. | Decay của Harp dài hơn Decay của Guitar. |

---

## Bước 6 — Câu hỏi giảng viên thường hỏi

### ❓ "Hệ thống của em đã có Attack Time làm trùm rồi, mắc mớ gì phải tốn công làm thêm thằng Decay Time này vào Giai đoạn 2?"

> **Gợi ý trả lời:** "Dạ thưa thầy/cô, Attack Time đúng là 'trùm' để chia tách Dây Kéo và Dây Gảy. Tuy nhiên, nó bị một Điểm Mù (Blind Spot) rất chí mạng: Nó gom Violin Pizzicato, Guitar gảy phím, Harp... tất tật vào chung một rọ vì chúng đều bật âm lượng lên cực đại trong thời gian sấp sỉ nhau (cực ngắn). Đầu vào giống nhau, nhưng cái 'Đuôi' phía sau khác hẳn nhau ạ. Dây thép đàn Violin bé xíu thì tắt tiếng cái rụp, dây ruột con hạc Harp thì cứ ngân nga mãi. Do đó, em bổ sung Decay Time để nó làm bộ lọc tầng 2, quét cho bằng sạch sự phân hóa chi tiết bên trong cái 'họ Gảy' đông đúc đó."

### ❓ "Nếu file âm thanh bị nghệ sĩ cố tình ấn tay chặn dây (Mute/Damping) ngay sau khi gảy thì Decay Time có còn đúng không?"

> **Gợi ý trả lời:** "Dạ thưa thầy/cô, CÓ ạ, kết quả đo vẫn hoàn toàn đúng với vật lý, nhưng ý nghĩa thay đổi một chút. Lúc đó Decay Time cực ngắn không còn phản ánh 'cấu tạo hộp vang' của nhạc cụ nữa, mà phản ánh rành mạch cái 'Kỹ thuật tay người chơi' (Damping Technique). Đây là giá trị cộng thêm rất lớn, vì nó giúp hệ thống không chỉ biết đó là cây Guitar, mà còn biết đó là tiếng Guitar đang bị bịt tiếng (Palm Mute). Rất có lợi cho AI sau này."

---

> **Tổng kết:** Cuộc đời của một nốt nhạc có Nở (Attack) và có Tàn (Decay). Bạn đã gom đủ bộ máy quét hành trình vòng đời hoàn chỉnh của âm nhạc. Không có cái gảy nhẹ nào qua nổi cặp mắt thần này.

---

**Bạn đã sẵn sàng sang đặc trưng tiếp theo (Spectral Flatness — 1 chiều) — chuyên gia bắt quả tang "hạt nhiễu/tiếng khò khè" chưa?**
