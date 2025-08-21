# Tài Liệu: Cơ Chế Ghi Log Trong Hệ Thống - Dành Cho Người Mới

## Mục Lục
1. [Tổng Quan Về Logging](#1-tổng-quan-về-logging)
2. [Mục Đích: Tại Sao Ghi Log Ra File?](#2-mục-đích-tại-sao-ghi-log-ra-file)
3. [Nội Dung Log Trong Hệ Thống](#3-nội-dung-log-trong-hệ-thống)
4. [Cơ Chế Xử Lý Log - AbstractJobProcessLog](#4-cơ-chế-xử-lý-log---abstractjobprocesslog)
5. [Cơ Chế Log Rotation (Cắt Log)](#5-cơ-chế-log-rotation-cắt-log)
6. [Cơ Chế Wait, Retry, Failed](#6-cơ-chế-wait-retry-failed)
7. [So Sánh: File Log vs Direct Database](#7-so-sánh-file-log-vs-direct-database)
8. [Ưu Nhược Điểm Từng Phương Pháp](#8-ưu-nhược-điểm-từng-phương-pháp)
9. [Kết Luận](#9-kết-luận)

---
<img width="828" height="810" alt="image" src="https://github.com/user-attachments/assets/d4152acc-1eb8-457a-9225-23525fc18b48" />


## 1. Tổng Quan Về Logging

### Logging là gì?
**Logging** là quá trình ghi lại các sự kiện, hoạt động và thông tin quan trọng của ứng dụng trong quá trình chạy. Giống như "nhật ký" của hệ thống.

### Ví dụ đơn giản:
Hãy tưởng tượng bạn vận hành một cửa hàng:
- **Log kỹ thuật**: "8:00 AM - Hệ thống POS khởi động thành công"
- **Log nghiệp vụ**: "9:15 AM - Khách hàng 0901234567 mua sản phẩm A với giá 100k"
- **Log lỗi**: "10:30 AM - Không thể kết nối máy in hóa đơn"

---

## 2. Mục Đích: Tại Sao Ghi Log Ra File?

### 🤔 Câu hỏi: Tại sao không ghi thẳng vào database?

### Lý do chính:

#### **A. Hiệu suất (Performance)**
```
Ví dụ minh họa:
- API xử lý 1000 request/giây
- Mỗi request cần ghi log → 1000 lần ghi database/giây
- Database sẽ bị quá tải → Chậm → Ảnh hưởng toàn bộ hệ thống
```

#### **B. Độ tin cậy (Reliability)**
```
Tình huống:
- Database bị lỗi/chậm → Log không ghi được
- Ghi file: Luôn ghi được (trừ khi ổ cứng đầy)
- File bị mất ít nguy hiểm hơn database bị crash
```

#### **C. Tách biệt trách nhiệm**
- **Ứng dụng chính**: Xử lý nghiệp vụ + ghi log file
- **Job xử lý log**: Đọc file + ghi database
- Nếu job log lỗi → Ứng dụng chính vẫn hoạt động bình thường

#### **D. Có thể xử lý sau (Asynchronous)**
```
Luồng xử lý:
1. Request đến → Xử lý nghiệp vụ → Ghi log file → Trả response (NHANH)
2. Job chạy nền → Đọc file log → Ghi database (CHẬM NHƯNG KHÔNG ẢNH HƯỞNG)
```

---

## 3. Nội Dung Log Trong Hệ Thống

### Phân loại log trong hệ thống này:

#### **A. Log Kỹ Thuật (Technical Logs)**
- **File**: `module.log`
- **Nội dung**: Khởi động ứng dụng, lỗi hệ thống, cảnh báo
- **Ví dụ**:
```
2024-08-21 10:30:15.123 | main | INFO | Application started successfully
2024-08-21 10:35:22.456 | pool-1 | ERROR | Database connection timeout
```

#### **B. Log Nghiệp Vụ (Business Logs)**
- **File**: `log-inbound.log`
- **Nội dung**: Thông tin API calls, transaction
- **Cấu trúc JSON**:
```json
{
  "account": "AGENT001",
  "channel": "7",
  "msisdn": "0901234567",
  "method": "POST",
  "clientIp": "192.168.1.100",
  "req": "{\"otp\":\"123456\"}",
  "response": "Success",
  "status": "200",
  "msg": "OTP verified successfully",
  "trandId": "TXN_20240821_001",
  "timeUpd": "2024/08/21 10:30:15",
  "timeMs": 150,
  "apiNode": "Node 1"
}
```

#### **C. Log Tổng Hợp (Summary Logs)**
- **File**: `logs-of-day.log`
- **Nội dung**: Tóm tắt các log đã được xử lý thành công

### Luồng tạo log nghiệp vụ:

```
1. Client gọi API → Controller nhận request
2. Controller xử lý → Tạo response
3. LoggerModule.writeLoggerApi() được gọi
4. Tạo đối tượng LogApi với thông tin đầy đủ
5. Convert sang JSON → Ghi vào file log-inbound.log
```

---

## 4. Cơ Chế Xử Lý Log - AbstractJobProcessLog

### 🔄 Tại sao cần quét định kỳ?

#### **Vấn đề**: 
Không thể ghi database ngay khi có log vì:
- Ảnh hưởng hiệu suất
- Database có thể tạm thời không khả dụng
- Cần xử lý batch để tối ưu

#### **Giải pháp**: Job chạy nền định kỳ

### Cách hoạt động:

#### **A. Cấu hình thời gian**
```yaml
logging:
  job:
    path:
      time-read: 10000  # Quét mỗi 10 giây
```

#### **B. Luồng xử lý chi tiết**:

```
1. [Mỗi 10 giây] Job thức dậy
   ↓
2. Quét thư mục "wait" → Tìm file log mới
   ↓
3. Quét thư mục "retry" → Tìm file cần thử lại
   ↓
4. Với mỗi file tìm được:
   a. Kết nối database
   b. Đọc từng dòng log
   c. Parse JSON → Đối tượng LogApi
   d. Thêm vào batch (mặc định 5000 records)
   e. Khi đủ batch → Commit vào database
   ↓
5. Nếu thành công → Xóa file
   Nếu thất bại → Chuyển file sang "retry" hoặc "failed"
```

#### **C. Ví dụ cụ thể**:

```
File: work-log-inbound.1.2024-08-21-103015.log
Nội dung:
{"account":"AGENT001","channel":"7",...}
{"account":"AGENT002","channel":"7",...}
{"account":"AGENT003","channel":"7",...}
... (3000 dòng)

Job xử lý:
- Đọc 3000 dòng
- Parse thành 3000 LogApi objects
- Tạo 3000 SQL INSERT statements
- Gộp thành 1 batch → Execute
- Commit → Xóa file
```

### Tại sao cần cơ chế này?

#### **1. Tối ưu database**
- Thay vì 3000 lần INSERT riêng lẻ
- Chỉ cần 1 lần batch INSERT → Nhanh hơn 10-100 lần

#### **2. Khôi phục được**
- File log = "backup" tạm thời
- Nếu database lỗi → File vẫn còn → Có thể xử lý lại

#### **3. Không block ứng dụng**
- Ứng dụng chỉ cần ghi file (nhanh)
- Việc ghi database (chậm) được làm riêng

---

## 5. Cơ Chế Log Rotation (Cắt Log)

### 🗂️ Vấn đề: File log ngày càng lớn

Nếu không cắt log:
```
Ngày 1: log-inbound.log (1MB)
Ngày 2: log-inbound.log (5MB)
Ngày 30: log-inbound.log (500MB)
Ngày 365: log-inbound.log (20GB) ← Khó quản lý!
```

### Giải pháp: Log Rotation

#### **Cấu hình trong log4j2.xml**:
```xml
<RollingFile name="log-inbound" 
             fileName="logs/work/log-inbound.log"
             filePattern="logs/work/wait/work-log-inbound.%i.%d{yyyy-MM-dd-HHmmss}.log">
    <Policies>
        <TimeBasedTriggeringPolicy interval="10" modulate="true"/>
        <CronTriggeringPolicy schedule="0 * * * * ? *"/>
    </Policies>
</RollingFile>
```

#### **Ý nghĩa**:
- **interval="10"**: Cắt file mỗi 10 phút
- **schedule="0 * * * * ? *"**: Hoặc cắt mỗi phút (theo cron)
- **%i**: Số thứ tự file
- **%d{yyyy-MM-dd-HHmmss}**: Timestamp

#### **Kết quả**:
```
logs/work/
├── log-inbound.log                           ← File hiện tại (đang ghi)
└── wait/
    ├── work-log-inbound.1.2024-08-21-103000.log  ← Đã cắt, chờ xử lý
    ├── work-log-inbound.2.2024-08-21-104000.log
    └── work-log-inbound.3.2024-08-21-105000.log
```

### Lợi ích của Log Rotation:

#### **1. Quản lý dễ dàng**
- File nhỏ → Đọc/xử lý nhanh
- Có thể xóa file cũ để tiết kiệm dung lượng

#### **2. Xử lý song song**
- Job có thể xử lý nhiều file cùng lúc
- File mới vẫn có thể ghi trong khi xử lý file cũ

#### **3. Khôi phục từng phần**
- Nếu 1 file bị lỗi → Chỉ mất data của file đó
- Các file khác vẫn an toàn

---

## 6. Cơ Chế Wait, Retry, Failed

### 📁 Hệ thống thư mục:

```
logs/work/
├── log-inbound.log           ← File đang ghi
├── wait/                     ← File chờ xử lý
│   ├── work-log-inbound.1.log
│   └── work-log-inbound.2.log
├── retry/                    ← File thử lại
│   └── work-log-inbound.3.log
└── failed/                   ← File thất bại
    └── work-log-inbound.4.log
```

### Luồng xử lý:

```
1. File mới được tạo → Đặt trong "wait/"
   ↓
2. Job đọc file từ "wait/"
   ↓
3a. Xử lý THÀNH CÔNG → Xóa file
   ↓
3b. Xử lý THẤT BẠI → Chuyển sang "retry/"
   ↓
4. Job đọc file từ "retry/" (với số lần retry đã ghi nhận)
   ↓
5a. Xử lý THÀNH CÔNG → Xóa file
   ↓
5b. Xử lý THẤT BẠI → Tăng số lần retry
   ↓
6. Nếu vượt quá max-retry (3 lần) → Chuyển sang "failed/"
```

### Cấu hình:
```yaml
logging:
  job:
    max-retry: 3          # Thử lại tối đa 3 lần
    path:
      wait: logs/work/wait
      retry: logs/work/retry
      failed: logs/work/failed
```

### Ví dụ cụ thể:

#### **Tình huống**: Database tạm thời bị lỗi

```
Time 10:00: File "work-log-inbound.1.log" được tạo trong "wait/"
Time 10:10: Job xử lý → Database lỗi → Chuyển file sang "retry/" (retry = 1)
Time 10:20: Job xử lý lại → Vẫn lỗi → Tăng retry = 2
Time 10:30: Job xử lý lại → Vẫn lỗi → Tăng retry = 3
Time 10:40: Job xử lý lại → Vẫn lỗi → Chuyển sang "failed/" (vượt quá max-retry)
```

### Lợi ích:

#### **1. Không mất dữ liệu**
- File luôn được bảo toàn đến khi xử lý thành công
- Có cơ chế thử lại tự động

#### **2. Tránh vòng lặp vô hạn**
- Giới hạn số lần retry → Không bị "treo" hệ thống
- File failed có thể được xử lý thủ công sau

#### **3. Giám sát và cảnh báo**
- Theo dõi thư mục "failed/" → Biết khi nào có vấn đề
- Có thể setup alert khi có file trong "failed/"

---

## 7. So Sánh: File Log vs Direct Database

### 🔄 Phương pháp 1: Ghi trực tiếp Database

```java
// Mỗi khi có request
public void handleRequest(Request req) {
    // Xử lý nghiệp vụ
    Response response = processBusinessLogic(req);
    
    // Ghi log trực tiếp
    database.insert("INSERT INTO log_api VALUES (...)");  // ← CHẬM!
    
    return response;
}
```

#### **Ưu điểm**:
- Đơn giản, trực tiếp
- Dữ liệu ngay lập tức có trong database
- Không cần xử lý file

#### **Nhược điểm**:
- **Chậm**: Mỗi request phải chờ database
- **Không tin cậy**: Database lỗi → Mất log
- **Coupling cao**: Log lỗi → Ảnh hưởng business logic

### 📄 Phương pháp 2: Ghi File → Xử lý sau (Hiện tại)

```java
// Mỗi khi có request
public void handleRequest(Request req) {
    // Xử lý nghiệp vụ
    Response response = processBusinessLogic(req);
    
    // Ghi log vào file
    fileLogger.info(logData);  // ← NHANH!
    
    return response;
}

// Job riêng biệt
@Scheduled(fixedDelay = 10000)
public void processLogFiles() {
    // Đọc file → Parse → Batch insert database
}
```

#### **Ưu điểm**:
- **Nhanh**: Request không phải chờ database
- **Tin cậy**: File luôn ghi được
- **Tách biệt**: Log lỗi không ảnh hưởng business
- **Batch processing**: Hiệu quả cao

#### **Nhược điểm**:
- Phức tạp hơn
- Dữ liệu không real-time trong database
- Cần quản lý file

---

## 8. Ưu Nhược Điểm Từng Phương Pháp

### 📊 Bảng so sánh chi tiết:

| Tiêu chí | Direct Database | File → Database |
|----------|----------------|-----------------|
| **Hiệu suất** | ❌ Chậm (mỗi request chờ DB) | ✅ Nhanh (ghi file instant) |
| **Độ tin cậy** | ❌ DB lỗi → Mất log | ✅ File backup data |
| **Độ phức tạp** | ✅ Đơn giản | ❌ Phức tạp hơn |
| **Real-time** | ✅ Ngay lập tức | ❌ Delay vài giây/phút |
| **Scalability** | ❌ DB bottleneck | ✅ Dễ scale |
| **Maintenance** | ✅ Ít component | ❌ Nhiều component |
| **Recovery** | ❌ Khó khôi phục | ✅ Dễ replay file |
| **Monitoring** | ✅ Query DB trực tiếp | ❌ Cần theo dõi file |

### 🎯 Khi nào dùng phương pháp nào?

#### **Dùng Direct Database khi**:
- Ứng dụng nhỏ, ít traffic
- Cần real-time data
- Team nhỏ, ít tài nguyên maintain

#### **Dùng File → Database khi**:
- Ứng dụng lớn, nhiều traffic
- Hiệu suất là ưu tiên
- Có tài nguyên develop và maintain
- Cần độ tin cậy cao

---

## 9. Kết Luận

### 🎯 Tại sao hệ thống này chọn File → Database?

#### **1. Đây là hệ thống API có traffic cao**
- Cần xử lý nhiều request đồng thời
- Không thể để logging làm chậm response time

#### **2. Dữ liệu log rất quan trọng**
- Log API calls để audit, billing
- Không thể mất dữ liệu → Cần backup (file)

#### **3. Database phải ổn định cho business**
- Tách biệt logging khỏi business logic
- Database lỗi → Business vẫn chạy được

### 📚 Tóm tắt luồng hoạt động:

```
1. API Request → Business Logic → Write File Log (FAST)
                              ↓
2. Background Job (10s interval) → Read Files → Batch Insert DB
                              ↓
3. Success → Delete File
   Failed → Retry (max 3 times) → Move to Failed folder
```

### 🔧 Điểm cần lưu ý khi vận hành:

#### **1. Monitoring**
- Theo dõi thư mục `failed/` → Alert khi có file
- Monitor disk space của thư mục log
- Theo dõi database connection pool

#### **2. Maintenance**
- Định kỳ dọn dẹp file log cũ
- Backup database log
- Check performance của log job

#### **3. Troubleshooting**
- File trong `retry/` → Tạm thời lỗi database
- File trong `failed/` → Lỗi nghiêm trọng, cần can thiệp
- Log job không chạy → Check cấu hình schedule

### 💡 Bài học rút ra:

**Logging không chỉ là "ghi lại thông tin"**, mà là **thiết kế cả một hệ thống** đảm bảo:
- **Performance**: Không làm chậm ứng dụng chính
- **Reliability**: Không mất dữ liệu quan trọng  
- **Maintainability**: Dễ vận hành và khắc phục sự cố
- **Scalability**: Có thể mở rộng khi traffic tăng

Đây là lý do tại sao các hệ thống enterprise thường chọn phương pháp **asynchronous logging** thay vì ghi trực tiếp database.
