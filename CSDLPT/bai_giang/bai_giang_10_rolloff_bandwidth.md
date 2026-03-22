# Bài Giảng 10 (GĐ2): Bộ Đôi Rolloff & Bandwidth — Đo Lường Hình Dáng Phổ

> **Vị trí trong vector (Giai đoạn 2):** Bổ sung thêm 2 chiều (Mean Rolloff + Mean Bandwidth).
> **Hạng mục:** Bộ ba "Tam giác quang phổ" cùng với Spectral Centroid, tạc tượng lại kích thước thật của nhạc cụ.

---

## Bước 1 — Giải thích bằng ngôn ngữ đời thường

Hôm nay là bài giảng gộp 2 đặc trưng, bởi vì chúng giống hệt một cặp anh em sinh đôi chuyên xử lý một bức tranh duy nhất:

### Đo cái gì?

Hãy tưởng tượng lại biểu đồ tần số âm thanh (FFT) giống như một **dãy núi khổng lồ** nằm trải dài:
- **Ngọn núi:** Chứa năng lượng nốt nhạc.
- **Trục ngang:** Vị trí của đỉnh núi (trái là âm thanh Bass trầm, phải là âm thanh Treble chói).

Ta đã biết ở bài số 3: **Spectral Centroid** là cắm một cây cờ vào "Điểm trọng tâm" chia đều trọng lượng dãy núi làm hai nửa cân bằng. Vậy hai anh em còn lại làm gì?

1. **Spectral Rolloff (Điểm cắt trần):** Giống như bạn đang đi leo núi từ bên trái (foot của dãy núi tần số thấp) sang bên phải. Bạn cứ đi, nhặt từng hòn đá năng lượng bỏ vào bao tải. Cứ nhặt cho đến khi bạn lấy được **85% tổng khối lượng** của cả dãy núi thì bạn dừng lại. Tọa độ (Hz) ở chỗ bạn dừng chân chính là Rolloff! 
   - Nếu năng lượng tập trung toàn ở dưới chân núi mập mạp (Cello trầm ấm), bạn mới leo tí xíu đã gom đủ 85% → **Rolloff thấp.**
   - Nếu có quá nhiều chóp chói lóa nằm tít mây xanh (Violin chói lòa hoặc tiếng nhiễu ma sát vĩ), bạn phải lao lên tận đỉnh cao vút mới gom đủ % → **Rolloff vọt rất cao.**

2. **Spectral Bandwidth (Bề ngang quang phổ):** Giống như bạn kéo một cái thước dây đo từ sườn trái sang sườn phải của dãy núi để xem nó **Béo/Xòe rộng** (chứa cực kỳ nhiều tạp âm phức tạp và nhiễu) hay **Nhỏ gầy/Mỏng dính** (âm thanh cực kỳ sắc nhọn, gắt ở 1 điểm duy nhất).

---

## Bước 2 — Giải thích kỹ thuật (đơn giản hóa)

Toán học đằng sau đơn giản thôi:

- **Rolloff:** Bắt đầu cộng dồn năng lượng của các cột bar FFT từ `0Hz` trải lên trên. Dừng lại khi Tổng cộng dồn vượt qua một đường ranh giới cài sẵn (vd: `0.85` nhân với Tổng toàn hệ thống).
- **Bandwidth:** Mượn chính tọa độ của anh trai **Centroid** làm gốc tọa độ. Sau đó tính Phương Sai (Độ lệch chuẩn) của tất cả các cột tần số phân tán xung quanh cái cây cờ Centroid đó bao xa. Khoảng trôi dạt đó (tính bằng Hz) là Bandwidth.

---

## Bước 3 — Lý do chọn tham số

- Riêng **Rolloff** cần truyền giá trị cực kỳ nổi tiếng: `roll_percent = 0.85`.
- Tại sao phải là 85%? Tại sao không gom hết 100% hay chỉ gom 50%?
  - Dải phổ âm thanh không bao giờ về 0, luôn có các "vài cọng rác nhiễu (noise)" rải rác tuốt trên mốc 20,000Hz siêu âm. Nếu lấy 100%, bạn sẽ leo mút chỉ lên tận trần bầu trời.
  - Cắt bỏ đoạn 15% cuống phổ chói lóa phía rác đó đi là cách giới khoa học máy tính thống nhất để tìm ra "Đỉnh thực sự cao nhất mà tiếng đàn chạm đỉnh tới".

---

## Bước 4 — Tại sao đặc trưng này đóng góp 2 chiều (và dùng Mean)?

- Quá trình leo núi (Rolloff) trả về tọa độ `Hz` cuối cùng chạm vạch → 1 chiều.
- Quá trình lấy thước kéo ngang sườn núi (Bandwidth) trả về độ rộng `Hz` → 1 chiều.
- Giống những biểu đồ FFT khác, dãy núi này nhảy múa rung rinh theo từng mili-giây. Ta tính **Trung bình (Mean)** lại cho cả hành trình bài nhạc tóm lại gom về 2 chỉ số ổn định. Đóng góp cứng cáp 2 chiều cho Giai đoạn 2.

---

## Bước 5 — Vai trò trong bài toán nhạc cụ bộ dây

Vì Giai đoạn 1 đã có Centroid, thêm 2 anh em này vào sẽ khóa chặt mọi lỗ hổng của phổ nhạc, tạo ra "Bộ 3 kích thước 3D": Điểm giữa (Centroid) - Trần trên (Rolloff) - Bề ngang (Bandwidth). 

| Nhạc cụ | Ứng dụng Rolloff (Chặn trần) | Ứng dụng Bandwidth (Bề dày) |
|---|---|---|
| **Violin Arco (Vĩ cầm kéo)** | Tiếng rít của nhựa thông bôi lên vĩ rải rác ra những tín hiệu chói chang ở tần số cao. Quét 85% sẽ tóm dính cái trần cực kỳ chói lóa này, Rolloff vút bay lên khoảng `6000-8000Hz`. | Tiếng nhạc cọ xát vô cùng "nhiễu" do ma sát bề mặt rải rác mọi tần. Bandwidth banh chành béo múp. |
| **Guitar / Harp (Dây gảy)** | Nốt trầm ấm, âm thanh tròn trịa kết tụ chặt ở thân đàn dưới 1 vòm duy nhất. Gom 85% mất chưa tới nửa giây ở mốc `1500-2500Hz`. | Tiếng cực mảnh, sắc và lảnh lót. Bề ngang gọn nhẹ, Bandwidth gầy gò ốm nhom. |

---

## Bước 6 — Câu hỏi giảng viên thường hỏi

### ❓ "Tại sao trong Giai đoạn 1 (37 chiều) em chỉ dùng mỗi Spectral Centroid (Trọng tâm) để đo kích cỡ đàn, mà giờ sang Giai đoạn 2 (56 chiều) em lại bê thêm Rolloff và Bandwidth? Ba cái này đo cùng chung 1 biểu đồ cơ mà?"

> **Gợi ý trả lời:** "Dạ thưa thầy/cô, thầy/cô nói tuyệt đối chính xác, tụi nó làm việc trên cùng 1 biểu đồ FFT. 
> Ở GĐ1, việc chỉ dùng Centroid (1 Tọa độ ở giữa) được chọn làm mũi nhọn đại diện vì nó đo phản ứng mạnh mẽ nhất về độ sáng và chênh lệch to/nhỏ của hộp cộng hưởng (Ví dụ: Cello tịt dưới, Violin vút trên). Nó đủ xài làm code lõi 37 chiều (Tiết kiệm CPU tính toán).
> Tuy nhiên, chỉ biết 'điểm giữa' thì Máy tính dễ bị qua mặt. Ví dụ: Một ngọn núi tụ toàn năng lượng tại vạch 2000Hz, và Một cánh đồng bằng phẳng nằm dàn ngang từ 1000Hz - 3000Hz... cả 2 địa hình này đều có chung tọa độ Centroid ở giữa là 2000Hz! 
> Nâng cấp lên GĐ2, em mang **Bandwidth** vào để đo 'độ béo' của 2 bên cánh để khử ảo giác bằng phẳng đó, cộng thêm **Rolloff** chặn 'cái trần nhà' để xem cái cánh đồng đó có vọt tung nóc được không. Bộ ba này tạo thành Tam Giác Vàng Quang Phổ đảm bảo AI không bao giờ nhận nhầm kích thước hộp cộng hưởng mộc / điện tử nữa ạ."

---

> **Tổng kết:** Bộ não hệ thống Giai đoạn 2 vừa được tải lên "bản vẽ 3D địa hình dãy núi". Máy tính hiện có thể mường tượng được chiều cao, độ béo, và giới hạn chạm trần tột độ của từng âm thanh nhạc cụ bộ dây.

---

**Bạn đã sẵn sàng để đón nhận Cú Đấm Chốt Hạ cuối cùng của 56 chiều: RM Mean + Silence Ratio (2 chiều vĩ đại cuối cùng của dự án) chưa?**
