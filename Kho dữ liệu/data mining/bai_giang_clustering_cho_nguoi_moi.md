# 🎓 BÀI GIẢNG K-MEANS CLUSTERING (DÀNH CHO NGƯỜI LÀM QUEN TỪ SỐ 0)

Chào mừng bạn đến với lớp học Data Mining vỡ lòng! Hãy quên hết các con số toán học đi, chúng ta sẽ tưởng tượng bạn đang dọn dẹp tủ đồ chơi của một đứa trẻ.

---

## 1. PHÂN CỤM (CLUSTERING) LÀ CÁI GÌ?

Hãy tưởng tượng bạn có 10,000 món đồ chơi đổ đống trên sàn nhà. Mẹ bạn bảo: *"Xếp chúng thành các nhóm cho gọn"*.
Nhưng mẹ không bảo là nhóm "**Siêu nhân**", nhóm "**Ô tô**" hay nhóm "**Gấu bông**". Mẹ chỉ bảo **"Chia nhóm đi"**.

Bạn phải tự nhìn vào đống đồ chơi:
- A, mấy con này mềm mềm, nhiều lông $\rightarrow$ ném vào 1 góc.
- Mấy cái này cứng cứng, có bánh xe $\rightarrow$ ném vào 1 góc.

Hành động bạn tự động gom các đồ vật **có đặc điểm giống nhau lại gần nhau** mà không cần ai nói cho tên gọi của cái nhóm đó — đó chính là **Clustering (Phân cụm)**.

Trong bài tập của bạn, 10,000 món đồ chơi chính là **47,000 Đơn hàng**. Bạn không biết đơn hàng nào là "Khách sộp", đơn nào là "Khách bèo". Bạn yêu cầu máy tính: *"Ê máy tính, tự gom mấy đơn hàng giống giống nhau thành từng nhóm cho tôi"*.

---

## 2. K-MEANS LÀ GÌ?

K-Means là tên của một ông robot dọn đồ chơi ngu ngốc nhưng cực kỳ chăm chỉ.
- Chữ **"K"**: Nghĩa là Số lượng cái sọt rác (số cụm) mà bạn đưa cho robot.
- Chữ **"Means"**: Nghĩa là vị trí trung tâm của cái sọt rác đó (Điểm trung bình).

**K-Means hoạt động thế nào?**
1. **Bước 1:** Bạn quăng cho con robot 3 cái sọt (Vậy là K=3). Robot nhắm mắt ném đại 3 cái sọt vào 3 góc bất kỳ trong phòng.
2. **Bước 2:** Với mỗi món đồ chơi, robot xem cái sọt nào gần món đồ chơi nhất, nó sẽ gắn biển hiệu món đồ chơi đó thuộc về sọt đó.
3. **Bước 3:** Sau khi mọi đồ chơi đã có sọt, robot kéo cái sọt di chuyển vào **CHÍNH GIỮA (Means)** đám đồ chơi vừa được gán cho nó.
4. **Bước 4:** Lặp lại bước 2 và 3 liên tục. Sọt lại di chuyển, đồ chơi lại đổi sọt... cho đến khi các sọt không xê dịch được nữa. Tèn ten! Gọn gàng!

---

## 3. CHỌN "K" LÀ GÀ HAY TRỨNG? (ELBOW & SILHOUETTE)

Bạn thắc mắc rất cực kỳ chuẩn! K-Means có một điểm ngu: **Nó không biết nó cần bao nhiêu cái sọt.** 
Trừ khi bạn nói với nó *"K=3 cho tao"*, thì nó mới chịu làm. 

**Vấn đề:** Làm sao bạn biết phòng có bao nhiêu loại đồ chơi mà chọn số sọt (K)? K phải có trước K-Means, nhưng phải chạy K-Means mới biết chia mấy cụm là đẹp? Đây ĐÚNG CHUẨN LÀ BÀI TOÁN CON GÀ VÀ QUẢ TRỨNG!

**Cách giải quyết:** Chúng ta chơi đánh bùn sang ao! Ta xúi con robot dọn nhiều lần: "Làm thử nghiệm với 2 sọt xem sao? Thử 3 sọt xem? Thử 4 sọt xem?". Sau đó ta chấm điểm xem cách nào tốt nhất.

Có 2 ông giám khảo để chấm điểm:

### Giám khảo 1: ELBOW METHOD (Phương pháp cùi chỏ - Tìm sự mệt mỏi)
Ông giám khảo này đo khoảng cách. 
- Nếu bạn chỉ cho 1 sọt to ở giữa phòng, mọi đồ chơi đều ở rất xa sọt $\rightarrow$ Điểm phạt rất cao.
- Cho 2 sọt $\rightarrow$ Khoảng cách tới sọt gần hơn $\rightarrow$ Giảm phạt.
- Cho 3 sọt $\rightarrow$ Giảm phạt nhiều hơn nữa.
- Lợi dụng điều này, ta lách luật tăng sọt lên 100 sọt, mỗi đồ chơi 1 sọt thì độ phạt sẽ bằng 0. Nhưng chia 100 cụm thì chia làm cái gì nữa đúng không?

Khúc cua "Cùi chỏ" (Elbow) là chỗ mà việc bạn **thêm 1 cái sọt nữa hầu như KHÔNG giúp ích gì mấy cho việc dọn dẹp**. 
Ví dụ: Thêm sọt thứ 3 thì phòng gọn hẳn thấy rõ. Nhưng thêm sọt thứ 4, thứ 5 thì phòng chẳng gọn hơn bao nhiêu. Vậy chốt **K=3** ngay tại khúc cùi chỏ đó!

### Giám khảo 2: SILHOUETTE SCORE (Điểm hình chiếu - Đo độ hạnh phúc)
Ông này thì đi hỏi cảm nhận của từng món đồ chơi:
- *"Này chú gấu bông, chú nằm ở sọt 1 với cỗ xe tăng có vui không?"* $\rightarrow$ Gấu bông nói: *"Không, tôi khác hẳn xe tăng, tôi ghét sọt này"* (Điểm âm = -1).
- *"Này siêu nhân đỏ, chú nằm cung sọt với siêu nhân xanh thì sao?"* $\rightarrow$ Siêu nhân đỏ: *"Tôi hòa nhập quá tốt, đây chính là nhà"* (Điểm dương = 1).

Sau đó sẽ lấy Điểm Silhouette trung bình của cảm xúc tất cả đồ chơi. Ta thử từ K=2 đến K=10, lúc nào trung bình anh em "hạnh phúc" nhất (điểm gần 1 nhất) thì ta chốt K đó! Giám khảo này rất công bằng và khách quan!

---

## 4. TẠI SAO PHẢI CÓ TIỀN XỬ LÝ (LOG-TRANSFORM & STANDARD SCALER)?

Trước khi đưa cho con robot K-Means gom 47,000 đơn hàng, bạn phải qua bước tiền xử lý (như nhào bột dọn đường vậy). Lý do vì bộ não robot rất máy móc, chỉ biết so sánh vẹt.

### Log-transform là gì?
Tưởng tượng, trong 47,000 đơn hàng, đa số là các đơn 100k, 200k. Đột nhiên có ông khách sộp mua chiếc xe hơi 100 triệu! 
Với toán học Euclidean của robot, 100,000,000 là con số khổng lồ, nó lấn át hoàn toàn những số 100k kia. Robot sẽ tưởng cái đơn 100 triệu đó là cái rốn vũ trụ và chia cụm xung quanh nó cực kỳ lệch lạc.

**Hàm Logarit (Log)** giống như một cái kính lúp thần kỳ hoạt động ngược: 
Nó thu bé con voi khổng lồ lại thành "một động vật to", và phóng to con kiến thành "một động vật nhỏ". Cụ thể, $Log(100) = 2$, $Log(100,000,000) = 8$. Chênh lệch lúc này chỉ là 2 và 8 (rất dễ quản lý), thay vì 100 và trăm triệu. Giúp các con số "bớt cực đoan, không bị dính chùm", dàn đều ra để dễ nhìn.

### StandardScaler (Chuẩn hóa) là gì?
Bạn dựa vào 2 yếu tố để gom đơn: **Giá ($)** và **Trọng lượng (Gram)**.
- Giá tiền: từ 1$ đến 6000$ (tối đa tầm 4 chữ số).
- Trọng lượng: từ 1g đến 30,000g (tối đa tới 5 chữ số).

Do máy tính không hiểu "Đô la" hay "Gram", nó nhìn vào thì chỉ thấy con số **30000 to hơn 6000 rất nhiều**.
Hậu quả? Khoảng cách tạo ra bởi Trọng lượng lớn quá trình tạo ranh giới. Robot K-Means sẽ chỉ quan tâm đến trọng lượng, ai nặng thì gom vào, vứt bỏ toàn bộ yếu tố về giá! 

**StandardScaler** là vị pháp sư hô biến tất cả về 1 sân chơi công bằng:
Tính năng này sẽ "bóp" cả 2 giá trị này lại. Tiền sẽ bị bóp nghẹt chạy từ -3 đến +3. Trọng lượng cũng chạy từ -3 đến +3 (Trong đó 0 là mức trung bình). Lúc này Giá và Trọng lượng đều có quyền hạn bỏ phiếu ngang nhau, giúp robot phân cụm đều theo cả Giá và Trọng lượng.

---

## 🎯 TÓM LẠI THEO LUỒNG PLAN:

1. **Chuẩn bị đồ chơi:** Trích xuất 47,000 đơn hàng từ kho. Lấy 2 đặc điểm (features) là Giá và Trọng lượng.
2. **Kính thu nhỏ (Log-transform):** Thu nhỏ mấy cái đơn hàng tỷ giá dị biệt để khỏi phá team.
3. **Sân chơi công bằng (StandardScaler):** Đưa Giá và Trọng lượng về cùng 1 hệ đo đếm (-3 đến 3) để máy tính không bị mù quáng thiên vị con số nào lớn hơn.
4. **Vấn đề Gà-Trứng (Chọn K):** Con người ép robot dọn chạy thử từ 2 đến 10 cụm. Gọi giám khảo **Elbow** (nhìn chỗ đường cong gãy khúc như cùi chỏ) và giám khảo **Silhouette** (chọn chỗ điểm cao nhất) để chốt số lượng nhóm lý tưởng (Ví dụ: quyết định K = 3).
5. **Chạy thật (K-Means):** Bấm nút với K=3, K-Means hoạt động và dán nhãn cho cả 47,000 đơn hàng: mày là Cụm 0, mày là Cụm 1, thằng kia Cụm 2.
6. **Con người Đặt tên cụm:** Sau khi máy đã chia xong, con người (bạn) sẽ vào soi xem "Ủa cái Cụm 0 này máy nó gom mấy cái gì vào đây ta?". Phát hiện nó toàn gom đồ giá rẻ nhẹ hều, bạn tự tay đổi tên Cụm 0 thành: *"Cụm Hàng Tiêu Dùng Rẻ Khối Lượng Nhẹ"*.

Xong! Bạn đã hiểu toàn bộ bản chất Lớp 1 vỡ lòng của Data Mining! Bước tiếp theo là bắt tay vào đoạn code trong Notebook nhé!
