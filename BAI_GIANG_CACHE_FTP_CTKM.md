# BÀI GIẢNG: CƠ CHẾ CACHE TRONG ỨNG DỤNG FTP-CTKM
## Dành cho Fresher Developer - Giải thích từ A đến Z

---

## **MỤC LỤC**
1. [Phần 1: Khởi động - Tại sao Cache tồn tại?](#phần-1-khởi-động---tại-sao-cache-tồn-tại)
2. [Phần 2: Giải phẫu khái niệm](#phần-2-giải-phẫu-khái-niệm)
3. [Phần 3: Cơ chế hoạt động](#phần-3-cơ-chế-hoạt-động)
4. [Phần 4: Ví dụ minh họa](#phần-4-ví-dụ-minh-họa)
5. [Phần 5: Những lầm tưởng thường gặp](#phần-5-những-lầm-tưởng-thường-gặp-của-người-mới)
6. [Phần 6: Kết luận & Checklist](#phần-6-kết-luận--checklist)

---

## **PHẦN 1: KHỞI ĐỘNG - TẠI SAO CACHE TỒN TẠI?**

### **1.1. Hình ảnh so sánh: Nhà hàng và Bếp**

Trước khi nói về code, hãy tưởng tượng em đang làm việc ở một **NHÀ HÀNG**.

#### **Tình huống 1: Không có bếp phụ (Không có Cache)**

```
KHÁCH HÀNG → GỌI MÓN → ĐẦU BẾP NẤU TỪ ĐẦU → PHỤC VỤ

Quy trình chi tiết:
1. Khách gọi: "Cho tôi món phở bò"
2. Bếp trưởng: Lấy thịt từ tủ đông, rã đông, thái, 
               nấu nước dùng từ xương, chuẩn bị rau...
3. Thời gian: 45 phút
4. Khách: 😤 Đợi lâu quá!

Khách tiếp theo cũng gọi phở bò:
→ Lặp lại toàn bộ quy trình!
→ 45 phút nữa!
```

#### **Tình huống 2: Có bếp phụ chuẩn bị sẵn (Có Cache)**

```
KHÁCH HÀNG → GỌI MÓN → LẤY TỪ BẾP PHỤ → PHỤC VỤ NGAY

Quy trình chi tiết:
1. Bếp phụ CHUẨN BỊ TRƯỚC: Nước dùng đã nấu sẵn, thịt đã thái, 
                          rau đã rửa, bánh phở sẵn sàng
2. Khách gọi: "Cho tôi món phở bò"
3. Bếp trưởng: Chỉ cần trụng bánh phở, xếp thịt, chan nước dùng
4. Thời gian: 3 phút
5. Khách: 😊 Nhanh quá!

Khách tiếp theo:
→ Cũng chỉ 3 phút!
```

**ĐÂY CHÍNH LÀ CACHE!**

> **Cache = Bếp phụ chuẩn bị sẵn nguyên liệu**
> 
> Thay vì mỗi lần nấu lại từ đầu (query database), ta chuẩn bị sẵn nguyên liệu (lưu data trong RAM) để phục vụ nhanh hơn.

---

### **1.2. Ứng dụng vào code của em**

Nhìn vào ứng dụng FTP-CTKM của em:

```
LUỒNG DỮ LIỆU HIỆN TẠI:

Database Oracle → PromotionFtpCache (RAM) → JobManageService → FTP Download
       ↑                    ↑
  "Kho nguyên liệu"    "Bếp phụ sẵn sàng"
```

**Cụ thể trong code:**

```java
// File: PromotionFtpCache.java
@Component
public class PromotionFtpCache extends CacheSwapService<PromotionFtp> {
    // Đây là "BẾP PHỤ" - lưu sẵn thông tin các FTP server cần kết nối
    
    protected ConcurrentHashMap<String, PromotionFtp> fetchDataFromDB() {
        return dbService.getPromotionFtpMap(); // Lấy "nguyên liệu" từ "kho" (Database)
    }
}

// File: JobManageService.java
for (Map.Entry<String, PromotionFtp> entry : promotionFtpCache.getCache().entrySet()) {
    jobFtp.downloadFilesCtkm(entry.getValue());
    // → Lấy từ "bếp phụ" (Cache), KHÔNG phải từ "kho" (Database)
}
```

---

### **1.3. Vấn đề thực tế mà Cache giải quyết**

#### **Vấn đề 1: Database là "điểm nghẽn" (Bottleneck)**

```
Hình dung Database như một CỔNG DUY NHẤT:

     ┌─ Thread 1: Query promotion
     ├─ Thread 2: Query promotion
     ├─ Thread 3: Query promotion    ───────►  [DATABASE]  
     ├─ Thread 4: Query promotion              (chỉ xử lý được
     └─ Thread 5: Query promotion               1 query/lần)

Kết quả:
- Thread 1: Đợi 0ms
- Thread 2: Đợi 5ms (chờ Thread 1 xong)
- Thread 3: Đợi 10ms (chờ Thread 2 xong)
- Thread 4: Đợi 15ms
- Thread 5: Đợi 20ms

→ Càng nhiều request, càng chậm!
```

#### **Vấn đề 2: Network latency (Độ trễ mạng)**

```
App Server ──────── 5ms ──────── Database Server
                     ↑
              "Đường đi xa"
              
Mỗi lần query = 5ms network + 2ms xử lý = 7ms
1000 queries = 7000ms = 7 GIÂY!
```

#### **Vấn đề 3: Database có thể "chết"**

```
Tình huống thực tế:
- 2:00 AM: DBA bảo trì database, restart
- 2:00 - 2:05 AM: Database không khả dụng
- App của em: 😱 CRASH! Không kết nối được DB!

Với Cache:
- 2:00 - 2:05 AM: Database không khả dụng
- App của em: 😊 Vẫn chạy bình thường với data trong Cache!
```

---

### **1.4. Tại sao ứng dụng FTP này cần Cache?**

Nhìn vào workflow của ứng dụng:

```
Mỗi 30 giây (@Scheduled):
1. JobManageService chạy
2. Cần biết: "FTP server nào cần download?"
3. Thông tin này NẰM TRONG DATABASE (bảng PROMOTION_FTP)
```

**KHÔNG CÓ CACHE:**
```
Mỗi 30 giây:
├─ Query DB: SELECT * FROM PROMOTION_FTP → 5ms
├─ Xử lý kết quả
└─ Download files

1 ngày = 24 * 60 * 2 = 2880 lần query
1 tháng = 86,400 lần query
→ Database bị "đánh" liên tục!
```

**CÓ CACHE:**
```
Khởi động app:
├─ Query DB 1 LẦN → Lưu vào Cache

Mỗi 30 giây:
├─ Đọc từ Cache (RAM) → 0.001ms ← NHANH GẤP 5000 LẦN!
├─ Xử lý kết quả
└─ Download files

Mỗi 30 giây (background):
└─ Refresh cache từ DB (để cập nhật thay đổi)
```

---

## **PHẦN 2: GIẢI PHẪU KHÁI NIỆM**

### **2.1. Cache là gì? (Định nghĩa chính thức)**

> **Cache** (phát âm: /kæʃ/ - "cát") là một **lớp lưu trữ tạm thời** nằm giữa ứng dụng và nguồn dữ liệu chính (database), giúp truy xuất dữ liệu nhanh hơn bằng cách lưu các dữ liệu thường xuyên được truy cập vào bộ nhớ nhanh (RAM).

**Phân tích từng phần:**

| Thuật ngữ | Giải thích đời thường |
|-----------|----------------------|
| **Lớp lưu trữ tạm thời** | Như "bếp phụ" - không phải kho chính, chỉ giữ những gì cần dùng ngay |
| **Nằm giữa app và DB** | Đứng ở giữa, chặn bớt request đến database |
| **Bộ nhớ nhanh (RAM)** | RAM nhanh gấp 1000 lần ổ cứng (disk) |

### **2.2. Các thuật ngữ quan trọng**

#### **a) Cache Hit vs Cache Miss**

```
CACHE HIT (Trúng cache):
┌─────────────────────────────────────┐
│  Request: "Lấy thông tin FTP 1454" │
│                                     │
│  Cache: "Có sẵn!" → Trả về ngay    │
│                                     │
│  Thời gian: 0.001ms ✅              │
└─────────────────────────────────────┘

CACHE MISS (Trượt cache):
┌─────────────────────────────────────┐
│  Request: "Lấy thông tin FTP 9999" │
│                                     │
│  Cache: "Không có!" → Query DB     │
│                                     │
│  Thời gian: 5ms ❌                  │
└─────────────────────────────────────┘
```

#### **b) Cache Invalidation (Làm mới cache)**

```
Vấn đề: Data trong cache có thể "cũ" (stale)

Ví dụ:
- 10:00 AM: Cache lưu: FTP server IP = 192.168.1.1
- 10:05 AM: DBA đổi IP thành 192.168.1.2 (trong DB)
- 10:10 AM: App đọc cache → Vẫn thấy IP cũ! → LỖI!

Giải pháp: Cache Invalidation
- Định kỳ refresh cache (như code của em: mỗi 30 giây)
- Hoặc: Event-driven (khi DB thay đổi thì thông báo cho cache)
```

#### **c) TTL (Time To Live)**

```
TTL = Thời gian "sống" của data trong cache

Ví dụ: TTL = 30 giây
- Data được cache lúc 10:00:00
- Đến 10:00:30 → Data "hết hạn" → Cần refresh
```

#### **d) Warm Cache vs Cold Cache**

```
COLD CACHE (Cache lạnh):
├─ App vừa khởi động
├─ Cache RỖNG
└─ Mọi request đều phải query DB → CHẬM!

WARM CACHE (Cache ấm):
├─ App đã chạy được 1 lúc
├─ Cache đã có data
└─ Hầu hết request đọc từ cache → NHANH!
```

---

### **2.3. Các loại Cache phổ biến**

```
┌─────────────────────────────────────────────────────────┐
│                    CÁC LOẠI CACHE                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. IN-MEMORY CACHE (Cache trong RAM) ← App của em     │
│     └─ Ví dụ: ConcurrentHashMap, Guava Cache, Caffeine │
│                                                         │
│  2. DISTRIBUTED CACHE (Cache phân tán)                 │
│     └─ Ví dụ: Redis, Memcached, Hazelcast              │
│                                                         │
│  3. HTTP CACHE (Cache trên browser/CDN)                │
│     └─ Ví dụ: Browser cache, Cloudflare, Varnish       │
│                                                         │
│  4. DATABASE CACHE (Cache trong DB)                    │
│     └─ Ví dụ: MySQL Query Cache, Oracle Result Cache   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Ứng dụng của em dùng loại 1: IN-MEMORY CACHE**

```java
// ConcurrentHashMap = Một loại HashMap an toàn với đa luồng
private final AtomicReference<ConcurrentHashMap<String, T>> activeCache
```

---

### **2.4. Giải thích các thành phần trong code**

#### **a) ConcurrentHashMap**

```java
// HashMap thường - KHÔNG an toàn khi nhiều thread cùng truy cập
HashMap<String, PromotionFtp> normalMap = new HashMap<>();

// ConcurrentHashMap - AN TOÀN khi nhiều thread cùng đọc/ghi
ConcurrentHashMap<String, PromotionFtp> safeMap = new ConcurrentHashMap<>();
```

**Tại sao cần "an toàn với đa luồng"?**

```
Tình huống nguy hiểm với HashMap thường:

Thread 1: Đang đọc phần tử thứ 5
Thread 2: Xóa phần tử thứ 3
→ Thread 1 có thể đọc sai data hoặc CRASH!

Với ConcurrentHashMap:
Thread 1: Đọc an toàn
Thread 2: Xóa an toàn
→ Không conflict!
```

#### **b) AtomicReference**

```java
private final AtomicReference<ConcurrentHashMap<String, T>> activeCache
```

**Giải thích đơn giản:**

```
AtomicReference = "Hộp đựng" có khả năng THAY THẾ NGUYÊN TỬ

Hình dung:
┌──────────────────────────────────────┐
│  AtomicReference (Cái hộp)          │
│  ┌────────────────────────────────┐  │
│  │  ConcurrentHashMap (Đồ bên trong) │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘

"Thay thế nguyên tử" nghĩa là:
- Khi đổi HashMap mới, việc đổi diễn ra NGAY LẬP TỨC
- Không có trạng thái "nửa cũ nửa mới"
- Thread khác hoặc thấy MAP CŨ, hoặc thấy MAP MỚI, không bao giờ thấy "nửa nạc nửa mỡ"
```

#### **c) Double Buffering (Kỹ thuật 2 buffer)**

```java
private final AtomicReference<ConcurrentHashMap<String, T>> activeCache;   // Buffer 1
private final AtomicReference<ConcurrentHashMap<String, T>> stagingCache;  // Buffer 2
```

**Giải thích bằng hình ảnh:**

```
SINGLE BUFFER (1 cache):
┌─────────────────┐
│  CACHE DUY NHẤT │ ← Vừa đọc vừa ghi cùng lúc → NGUY HIỂM!
└─────────────────┘

DOUBLE BUFFER (2 cache):
┌─────────────────┐     ┌─────────────────┐
│  ACTIVE CACHE   │     │  STAGING CACHE  │
│  (Đang phục vụ) │     │  (Đang cập nhật)│
└─────────────────┘     └─────────────────┘
         ↑                       ↑
    Thread đọc              Thread ghi
    
→ Đọc và ghi TÁCH BIỆT → AN TOÀN!
```

**Quy trình hoạt động:**

```
Bước 1: Active đang phục vụ, Staging đang được cập nhật
        ┌─────────┐          ┌─────────┐
        │ ACTIVE  │ ← Đọc    │ STAGING │ ← Ghi data mới
        │ [A,B,C] │          │ [X,Y,Z] │
        └─────────┘          └─────────┘

Bước 2: SWAP! (Đổi vai trò)
        ┌─────────┐          ┌─────────┐
        │ STAGING │ ← Đọc    │ ACTIVE  │ (sẽ được ghi tiếp)
        │ [X,Y,Z] │          │ [A,B,C] │
        └─────────┘          └─────────┘
        
→ Người đọc thấy data mới NGAY LẬP TỨC!
→ Không có thời gian "chết" (downtime)!
```

---

### **2.5. So sánh: Cách làm CRUD truyền thống vs Cache**

| Khía cạnh | CRUD Truyền thống | Với Cache |
|-----------|-------------------|-----------|
| **Mỗi lần cần data** | Query DB | Đọc từ RAM |
| **Thời gian** | 5-50ms | 0.001ms |
| **Khi DB chết** | App crash | App vẫn chạy (với data cũ) |
| **Tải lên DB** | Cao (mỗi request = 1 query) | Thấp (chỉ refresh định kỳ) |
| **Độ phức tạp code** | Đơn giản | Phức tạp hơn |
| **Đồng bộ data** | Luôn mới nhất | Có thể trễ vài giây |

---

## **PHẦN 3: CƠ CHẾ HOẠT ĐỘNG**

### **3.1. Kiến trúc tổng quan**

```
┌─────────────────────────────────────────────────────────────────┐
│                        ỨNG DỤNG FTP-CTKM                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────┐         ┌─────────────────┐              │
│   │ JobManageService│         │ JobImport       │              │
│   │ (Download FTP)  │         │ (Import data)   │              │
│   └────────┬────────┘         └────────┬────────┘              │
│            │                           │                        │
│            │ getCache()                │ containsKey()          │
│            ▼                           ▼                        │
│   ┌─────────────────────────────────────────────────────┐      │
│   │              CACHE LAYER (Lớp Cache)                │      │
│   │  ┌─────────────────┐    ┌─────────────────┐        │      │
│   │  │PromotionFtpCache│    │FunringActiveCache│        │      │
│   │  └────────┬────────┘    └────────┬────────┘        │      │
│   │           │                      │                  │      │
│   │           └──────────┬───────────┘                  │      │
│   │                      ▼                              │      │
│   │           ┌─────────────────┐                       │      │
│   │           │ CacheSwapService│ (Abstract class)      │      │
│   │           │  - activeCache  │                       │      │
│   │           │  - stagingCache │                       │      │
│   │           └─────────────────┘                       │      │
│   └─────────────────────────────────────────────────────┘      │
│                          │                                      │
│                          │ fetchDataFromDB() (mỗi 30s)         │
│                          ▼                                      │
│   ┌─────────────────────────────────────────────────────┐      │
│   │              DATABASE LAYER (Oracle)                │      │
│   │  - Bảng PROMOTION_FTP                               │      │
│   │  - Bảng FUNRING_ACTIVE                              │      │
│   └─────────────────────────────────────────────────────┘      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### **3.2. Luồng khởi động (Application Startup)**

```
TIMELINE KHI APP KHỞI ĐỘNG:

T = 0ms: Spring Boot bắt đầu
    │
    ▼
T = 100ms: Scan và tạo các Bean
    │
    ├─ Tạo DatabaseUtil bean
    ├─ Tạo PromotionFtpCache bean
    └─ Tạo FunringActiveCache bean
    │
    ▼
T = 200ms: @PostConstruct được gọi
    │
    ├─ PromotionFtpCache.forceRefresh()
    │   ├─ Query DB: SELECT * FROM PROMOTION_FTP
    │   ├─ Nhận được 5 records
    │   └─ activeCache = {5 PromotionFtp objects}
    │
    └─ FunringActiveCache.forceRefresh()
        ├─ Query DB: SELECT * FROM FUNRING_ACTIVE
        ├─ Nhận được 10,000 records
        └─ activeCache = {10,000 MSISDN strings}
    │
    ▼
T = 500ms: App sẵn sàng phục vụ ✅
    │
    └─ Cache đã "ấm" (warm), có data ngay từ đầu!
```

**Code tương ứng:**

```java
// CacheSwapService.java
@PostConstruct  // ← Annotation này làm method chạy ngay khi bean được tạo
public void forceRefresh() {
    log.info("#>> Force refresh cache for {}", this.getClass().getSimpleName());
    cacheDataSync();  // Load data từ DB vào cache
}
```

---

### **3.3. Luồng đọc Cache (Read Flow)**

```
LUỒNG KHI JOB CẦN ĐỌC DATA:

JobManageService                    PromotionFtpCache                    activeCache (RAM)
      │                                    │                                    │
      │ 1. getCache()                      │                                    │
      │───────────────────────────────────►│                                    │
      │                                    │                                    │
      │                                    │ 2. activeCache.get()               │
      │                                    │───────────────────────────────────►│
      │                                    │                                    │
      │                                    │ 3. return ConcurrentHashMap        │
      │                                    │◄───────────────────────────────────│
      │                                    │                                    │
      │ 4. return Map<String, PromotionFtp>│                                    │
      │◄───────────────────────────────────│                                    │
      │                                    │                                    │
      │ 5. for each entry: download()      │                                    │
      │                                    │                                    │

Thời gian: 0.001ms (CỰC NHANH vì đọc từ RAM!)
```

**Code tương ứng:**

```java
// JobManageService.java
for (Map.Entry<String, PromotionFtp> entry : promotionFtpCache.getCache().entrySet()) {
    jobFtp.downloadFilesCtkm(entry.getValue());
}

// CacheSwapService.java
@Override
public ConcurrentHashMap<String, T> getCache() {
    return activeCache.get();  // Chỉ cần lấy reference, không query DB!
}
```

---

### **3.4. Luồng cập nhật Cache (Refresh Flow) - QUAN TRỌNG NHẤT!**

Đây là phần phức tạp nhất, hãy đọc kỹ:

```
LUỒNG CẬP NHẬT CACHE MỖI 30 GIÂY:

Thời điểm T = 30,000ms (30 giây sau khi app start)

┌─────────────────────────────────────────────────────────────────────┐
│ BƯỚC 1: Scheduler trigger                                          │
│                                                                     │
│ @Scheduled(fixedDelayString = "${app.sql.sync-time}")              │
│ public void forceRefresh() {                                        │
│     super.forceRefresh();  // Gọi method cha                       │
│ }                                                                   │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ BƯỚC 2: Bắt đầu cacheDataSync()                                    │
│                                                                     │
│ ConcurrentHashMap<String, T> currentStaging = stagingCache.get();  │
│ // Lấy staging buffer hiện tại                                     │
│                                                                     │
│ Trạng thái lúc này:                                                │
│ ┌─────────────────┐    ┌─────────────────┐                         │
│ │  activeCache    │    │  stagingCache   │                         │
│ │  {A, B, C, D, E}│    │  {cũ hoặc rỗng} │                         │
│ │  (đang phục vụ) │    │  (sẽ được dùng) │                         │
│ └─────────────────┘    └─────────────────┘                         │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ BƯỚC 3: Fetch data mới từ Database                                 │
│                                                                     │
│ ConcurrentHashMap<String, T> newData = fetchDataFromDB();          │
│ // → SELECT * FROM PROMOTION_FTP                                   │
│ // → Trả về 6 records (có thêm 1 promotion mới!)                   │
│                                                                     │
│ newData = {A, B, C, D, E, F}  ← F là record mới                    │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ BƯỚC 4: Validate data                                              │
│                                                                     │
│ if (newData == null || newData.isEmpty()) {                        │
│     log.warn("Fetched data is empty, keeping current cache");      │
│     return;  // KHÔNG update nếu data rỗng!                        │
│ }                                                                   │
│                                                                     │
│ // Bảo vệ: Nếu DB trả về rỗng (có thể do lỗi), giữ cache cũ       │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ BƯỚC 5: Chuẩn bị staging buffer                                    │
│                                                                     │
│ currentStaging.clear();        // Xóa data cũ trong staging        │
│ currentStaging.putAll(newData); // Copy data mới vào staging       │
│                                                                     │
│ Trạng thái lúc này:                                                │
│ ┌─────────────────┐    ┌─────────────────────┐                     │
│ │  activeCache    │    │  stagingCache       │                     │
│ │  {A, B, C, D, E}│    │  {A, B, C, D, E, F} │ ← Data mới!         │
│ │  (vẫn phục vụ)  │    │  (sẵn sàng swap)    │                     │
│ └─────────────────┘    └─────────────────────┘                     │
│                                                                     │
│ LƯU Ý: Trong thời gian này, các thread khác VẪN đọc activeCache   │
│        bình thường, không bị ảnh hưởng!                            │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ BƯỚC 6: ATOMIC SWAP - Phép màu xảy ra ở đây! ⚡                    │
│                                                                     │
│ ConcurrentHashMap<String, T> previousActive =                      │
│     activeCache.getAndSet(currentStaging);                         │
│                                                                     │
│ // getAndSet() làm 2 việc CÙNG LÚC (atomic):                       │
│ // 1. Lấy giá trị cũ của activeCache                               │
│ // 2. Gán giá trị mới (currentStaging) vào activeCache             │
│                                                                     │
│ stagingCache.set(previousActive);                                  │
│ // Đưa cache cũ vào staging (để dùng lại buffer)                   │
│                                                                     │
│ Trạng thái SAU SWAP:                                               │
│ ┌─────────────────────┐    ┌─────────────────┐                     │
│ │  activeCache        │    │  stagingCache   │                     │
│ │  {A, B, C, D, E, F} │    │  {A, B, C, D, E}│                     │
│ │  (ĐANG phục vụ)     │    │  (chờ lần sau)  │                     │
│ └─────────────────────┘    └─────────────────┘                     │
│                                                                     │
│ → Từ giây này, mọi getCache() sẽ thấy data MỚI (có F)!            │
└─────────────────────────────────────────────────────────────────────┘
```

---

### **3.5. Tại sao gọi là "Double Buffering"?**

```
So sánh với GAME GRAPHICS (Đồ họa game):

Trong game, màn hình được vẽ 60 lần/giây (60 FPS).
Nếu vẽ trực tiếp lên màn hình → Bị "nhấp nháy" (flickering)!

Giải pháp: DOUBLE BUFFERING
┌─────────────────┐    ┌─────────────────┐
│  FRONT BUFFER   │    │  BACK BUFFER    │
│  (Hiển thị)     │    │  (Đang vẽ)      │
└─────────────────┘    └─────────────────┘

1. Hiển thị Front Buffer cho người xem
2. Vẽ frame mới vào Back Buffer (người xem không thấy)
3. SWAP! Back → Front, Front → Back
4. Lặp lại

→ Người xem luôn thấy hình hoàn chỉnh, không bị nhấp nháy!

TƯƠNG TỰ với Cache:
┌─────────────────┐    ┌─────────────────┐
│  ACTIVE CACHE   │    │  STAGING CACHE  │
│  (Đang phục vụ) │    │  (Đang cập nhật)│
└─────────────────┘    └─────────────────┘

1. Phục vụ request từ Active Cache
2. Cập nhật data mới vào Staging Cache
3. SWAP! Staging → Active, Active → Staging
4. Lặp lại

→ User luôn đọc được data, không bị "downtime"!
```

---

### **3.6. Timeline hoàn chỉnh của 1 phút hoạt động**

```
TIMELINE: PHÚT ĐẦU TIÊN SAU KHI APP KHỞI ĐỘNG

T = 0s:
├─ App start
├─ @PostConstruct chạy
├─ Cache được load với 5 promotions
└─ activeCache = {1454, 1455, 1456, 1457, 1458}

T = 5s:
├─ JobManageService chạy (do @Scheduled)
├─ Gọi promotionFtpCache.getCache()
├─ Nhận được 5 promotions
└─ Download files từ 5 FTP servers

T = 15s:
├─ DBA thêm promotion mới vào DB (1459)
└─ Cache CHƯA biết về promotion này!

T = 30s:
├─ Cache refresh chạy (do @Scheduled)
├─ Query DB → Nhận 6 promotions (có 1459)
├─ SWAP cache
└─ activeCache = {1454, 1455, 1456, 1457, 1458, 1459}

T = 35s:
├─ JobManageService chạy lại
├─ Gọi promotionFtpCache.getCache()
├─ Nhận được 6 promotions (CÓ 1459!)
└─ Download files từ 6 FTP servers ✅

→ Từ lúc DBA thêm (T=15s) đến lúc app thấy (T=30s) = 15 giây delay
→ Đây là trade-off của cache: NHANH nhưng có thể HƠI CŨ
```

---

### **3.7. Xử lý lỗi (Error Handling)**

```java
// CacheSwapService.java
public void cacheDataSync() {
    try {
        // ... fetch và update cache ...
    } catch (Exception e) {
        // Nếu có lỗi (DB down, network timeout, etc.)
        int currentSize = activeCache.get().size();
        log.error("#>> Cache update failed, keeping current cache (size: {})", currentSize, e);
        // KHÔNG throw exception, KHÔNG update cache
        // → App vẫn chạy với data cũ!
    }
}
```

**Kịch bản lỗi:**

```
T = 60s:
├─ Cache refresh chạy
├─ Query DB → SQLException (DB đang bảo trì!)
├─ Catch exception
├─ Log: "Cache update failed, keeping current cache (size: 6)"
└─ activeCache VẪN GIỮ NGUYÊN = {6 promotions}

T = 65s:
├─ JobManageService chạy
├─ Gọi getCache()
├─ Nhận được 6 promotions (data cũ nhưng VẪN HOẠT ĐỘNG!)
└─ Download files thành công ✅

→ App KHÔNG CHẾT dù DB bị lỗi!
→ Đây gọi là "Graceful Degradation" (Suy giảm nhẹ nhàng)
```

---

## **PHẦN 4: VÍ DỤ MINH HỌA**

### **4.1. So sánh Code: Không Cache vs Có Cache**

#### **CÁCH LÀM CŨ (Không Cache) - Như CRUD thông thường:**

```java
// ❌ Code KHÔNG dùng cache
@Service
public class JobManageServiceOld {
    private final DatabaseUtil dbService;
    
    @Scheduled(fixedDelay = 30000)
    public void initDownloadJobCtkm() {
        // Mỗi lần chạy đều query DB
        ConcurrentHashMap<String, PromotionFtp> promotions = dbService.getPromotionFtpMap();
        
        for (Map.Entry<String, PromotionFtp> entry : promotions.entrySet()) {
            jobFtp.downloadFilesCtkm(entry.getValue());
        }
    }
}
```

**Vấn đề:**
```
Mỗi 30 giây:
├─ Query DB: 5ms
├─ Network latency: 2ms
├─ DB connection overhead: 3ms
└─ Tổng: 10ms

1 ngày = 2880 queries
1 tháng = 86,400 queries
→ Database bị "đánh" liên tục!
→ Nếu DB chết 5 phút → App chết 5 phút!
```

#### **CÁCH LÀM MỚI (Có Cache) - Code hiện tại của em:**

```java
// ✅ Code DÙNG cache (code thực tế của em)

// Bước 1: Định nghĩa Cache
@Component
public class PromotionFtpCache extends CacheSwapService<PromotionFtp> {
    private final DatabaseUtil dbService;
    
    @Override
    protected ConcurrentHashMap<String, PromotionFtp> fetchDataFromDB() {
        return dbService.getPromotionFtpMap();  // Chỉ dùng khi refresh
    }
    
    @Scheduled(fixedDelayString = "${app.sql.sync-time}")
    public void forceRefresh() {
        super.forceRefresh();  // Refresh cache mỗi 30s
    }
}

// Bước 2: Sử dụng Cache
@Service
public class JobManageService {
    private final PromotionFtpCache promotionFtpCache;  // Inject cache
    
    @Scheduled(fixedDelay = 30000)
    public void initDownloadJobCtkm() {
        // Đọc từ RAM, KHÔNG query DB!
        for (Map.Entry<String, PromotionFtp> entry : promotionFtpCache.getCache().entrySet()) {
            jobFtp.downloadFilesCtkm(entry.getValue());
        }
    }
}
```

**Lợi ích:**
```
Mỗi 30 giây (Job download):
├─ Đọc cache: 0.001ms ← NHANH GẤP 10,000 LẦN!
├─ Không network
├─ Không DB connection
└─ Tổng: 0.001ms

Mỗi 30 giây (Cache refresh - chạy song song):
├─ Query DB: 5ms
└─ Swap cache: 0.001ms

→ Job download KHÔNG BỊ BLOCK bởi DB!
→ DB chết 5 phút → App VẪN CHẠY với data cũ!
```

---

### **4.2. Ví dụ FunringActiveCache - Cache với 10,000+ records**

```java
// FunringActiveCache.java
@Component
public class FunringActiveCache extends CacheSwapService<String> {
    private final DatabaseUtil dbService;
    
    @Override
    protected ConcurrentHashMap<String, String> fetchDataFromDB() {
        return dbService.getFunringActiveMap();  // 10,000+ MSISDN
    }
    
    @Scheduled(cron = "0 */30 * * * *")  // Mỗi 30 phút
    public void forceRefresh() {
        super.forceRefresh();
    }
}

// Sử dụng trong JobImport.java
public Map<String, String> readFile(File file) throws IOException {
    Map<String, String> records = new HashMap<>();
    try (BufferedReader br = new BufferedReader(new FileReader(file))) {
        String line;
        while ((line = br.readLine()) != null) {
            String[] parts = line.split(",");
            String status = "0";
            
            // ⚡ Kiểm tra MSISDN có trong cache không
            if (!activeCache.containsKey(parts[0])) {  // Đọc từ cache!
                status = "-1";
            }
            records.put(parts[0], status);
        }
    }
    return records;
}
```

**Phân tích hiệu suất:**

```
Giả sử file có 1,000 dòng, cần kiểm tra 1,000 MSISDN:

KHÔNG CÓ CACHE:
├─ 1,000 queries: SELECT * FROM FUNRING_ACTIVE WHERE MSISDN = ?
├─ Mỗi query: 5ms
└─ Tổng: 5,000ms = 5 GIÂY!

CÓ CACHE:
├─ 1,000 lần containsKey()
├─ Mỗi lần: 0.0001ms (lookup HashMap trong RAM)
└─ Tổng: 0.1ms ← NHANH GẤP 50,000 LẦN!
```

---

### **4.3. Ví dụ Debug: Theo dõi Cache hoạt động qua Log**

```
Khi app chạy, bạn sẽ thấy log như sau:

=== KHỞI ĐỘNG ===
2024-01-15 08:00:00.100 INFO  - #>> Force refresh cache for PromotionFtpCache
2024-01-15 08:00:00.150 INFO  - #>> Start cache update for PromotionFtpCache
2024-01-15 08:00:00.200 INFO  - #>> Cache swapped successfully for PromotionFtpCache: 0 → 5 items, time: 50ms
                                                                                       ↑
                                                                              Từ rỗng lên 5 items

=== SAU 30 GIÂY ===
2024-01-15 08:00:30.100 INFO  - #>> Force refresh cache for PromotionFtpCache
2024-01-15 08:00:30.110 INFO  - #>> Start cache update for PromotionFtpCache
2024-01-15 08:00:30.130 INFO  - #>> Cache swapped successfully for PromotionFtpCache: 5 → 5 items, time: 20ms
                                                                                       ↑
                                                                              5 items → 5 items (không đổi)

=== KHI DB THÊM PROMOTION MỚI ===
2024-01-15 08:01:00.100 INFO  - #>> Force refresh cache for PromotionFtpCache
2024-01-15 08:01:00.110 INFO  - #>> Start cache update for PromotionFtpCache
2024-01-15 08:01:00.130 INFO  - #>> Cache swapped successfully for PromotionFtpCache: 5 → 6 items, time: 20ms
                                                                                       ↑
                                                                              5 → 6 (có promotion mới!)

=== KHI DB BỊ LỖI ===
2024-01-15 08:01:30.100 INFO  - #>> Force refresh cache for PromotionFtpCache
2024-01-15 08:01:30.110 INFO  - #>> Start cache update for PromotionFtpCache
2024-01-15 08:01:30.150 ERROR - #>> Cache update failed for PromotionFtpCache, keeping current cache (size: 6)
                                                                                                         ↑
                                                                              Giữ nguyên 6 items, không crash!
```

---

### **4.4. Ví dụ xử lý lỗi (Resilience)**

#### **Kịch bản: Database bị lỗi ở giây thứ 60**

```
Timeline chi tiết:

Giây 0:
├─ App khởi động
├─ @PostConstruct chạy
├─ Load 5 PromotionFtp từ DB
└─ activeCache = {1454: PromotionFtp, 1455: PromotionFtp, ...} (5 items)

Giây 30:
├─ @Scheduled chạy lần 1
├─ Query DB thành công → 5 items
└─ Swap cache: activeCache vẫn = 5 items (mới)

Giây 60:
├─ @Scheduled chạy lần 2
├─ Query DB → SQLException (DB crash!)
│
├─ Trong CacheSwapService.cacheDataSync():
│   try {
│       newData = fetchDataFromDB();  // ⚠️ throw SQLException
│   } catch (Exception e) {
│       log.error("Cache update failed, keeping current cache");
│       return; // ⛔ DỪNG LẠI - KHÔNG swap
│   }
│
└─ activeCache KHÔNG BỊ THAY ĐỔI = vẫn giữ 5 items từ giây 30

Giây 90:
├─ @Scheduled chạy lần 3
├─ DB vẫn lỗi → catch Exception
└─ activeCache = vẫn 5 items cũ

Giây 120:
├─ DB được khôi phục
├─ Query thành công → 6 items (có thêm promotion mới)
└─ Swap thành công: activeCache = 6 items mới
```

#### **So sánh: Nếu KHÔNG có Cache (Query trực tiếp DB)**

```java
// ❌ Cách làm NGUY HIỂM (không có cache)
public void downloadFilesCtkm() {
    List<PromotionFtp> promotions = dbService.getPromotionFtpList(); // ⚠️ Query DB mỗi lần
    for (PromotionFtp ftp : promotions) {
        // Download files...
    }
}
```

**Vấn đề nếu DB crash:**
```
Giây 60:
├─ Job chạy → Query DB → SQLException
└─ ⚠️ ỨNG DỤNG NGỪNG HOẠT ĐỘNG! Không download file nào!

Giây 90:
└─ ⚠️ Tiếp tục fail, không download được

→ KẾT QUẢ: Mất 5 file khách hàng trong 2 phút DB lỗi
          Khách hàng phàn nàn dịch vụ không hoạt động!
```

**Với Cache:**
```
Giây 60:
├─ Job chạy
├─ Lấy từ cache (không query DB)
└─ ✅ Download file THÀNH CÔNG dù DB lỗi!

Giây 90:
└─ ✅ Tiếp tục download thành công

→ KẾT QUẢ: Chỉ mất khả năng CẬP NHẬT promotion mới
          Nhưng các promotion cũ vẫn hoạt động bình thường!
```

---

### **4.5. Ví dụ về Thread Safety (An toàn đa luồng)**

#### **Tình huống thực tế:**

```
ỨNG DỤNG CỦA BẠN:

Thread 1: Scheduler - Cập nhật cache mỗi 30s
Thread 2: JobManageService - Đọc cache để download files
Thread 3: API endpoint - Kiểm tra promotion còn hoạt động không
Thread 4: Monitoring - Đếm số lượng cache items

→ 4 thread cùng truy cập CÙNG MỘT OBJECT cache!
```

#### **Vấn đề nếu KHÔNG thread-safe:**

```java
// ❌ Code NGUY HIỂM (không thread-safe)
public class DangerousCache {
    private HashMap<String, PromotionFtp> cache = new HashMap<>(); // ⚠️ HashMap không thread-safe!
    
    public void updateCache() {
        HashMap<String, PromotionFtp> newData = fetchFromDB();
        cache.clear();              // ⚠️ Bước 1
        cache.putAll(newData);      // ⚠️ Bước 2
    }
    
    public PromotionFtp get(String key) {
        return cache.get(key);      // ⚠️ Bước 3
    }
}
```

**RACE CONDITION (Điều kiện tranh chấp):**

```
Timeline chi tiết (microsecond level):

Microsecond 1000:
├─ Thread 1 (Update): cache.clear() → cache = {} (RỖNG!)
│
Microsecond 1001:  ⚠️ VẤN ĐỀ XẢY RA Ở ĐÂY!
├─ Thread 2 (Read): cache.get("1454") 
│   └─ return null (vì cache đang rỗng!)
│   └─ ⛔ NullPointerException → App CRASH!
│
Microsecond 1002:
└─ Thread 1 (Update): cache.putAll(newData) → cache = {1454: ...}

→ KẾT QUẢ: Có 1 millisecond cache BỊ RỖNG!
          Mọi request trong khoảng này đều fail!
```

#### **Giải pháp của CacheSwapService:**

**1. ConcurrentHashMap (Thread-safe container):**

```java
// ✅ An toàn khi nhiều thread cùng đọc
private final AtomicReference<ConcurrentHashMap<String, T>> activeCache;

// Thread 1 đọc: activeCache.get().get("1454")
// Thread 2 đọc: activeCache.get().get("1455")
// Thread 3 đọc: activeCache.get().get("1456")
// → Tất cả đều an toàn, không block nhau
```

**2. AtomicReference.getAndSet() (Atomic Swap):**

```java
// ✅ Đây là PHÉP MÀU THUẬT!
ConcurrentHashMap<String, T> previousActive = activeCache.getAndSet(currentStaging);

// Giải thích "Atomic":
// - Toàn bộ thao tác này là 1 HÀNH ĐỘNG NGUYÊN TỬ
// - Không thể bị gián đoạn giữa chừng
// - CPU đảm bảo nó hoàn thành trong 1 "clock cycle"
```

**So sánh chi tiết:**

| Thời điểm | Cách NGUY HIỂM (HashMap + clear) | Cách AN TOÀN (AtomicReference + swap) |
|-----------|----------------------------------|---------------------------------------|
| **Microsecond 1000** | cache.clear() → cache = {} RỖNG! | currentStaging = {new data} ở bộ nhớ riêng |
| **Microsecond 1001** | ⚠️ Thread khác đọc → null | ✅ Thread khác đọc → vẫn thấy cache CŨ |
| **Microsecond 1002** | cache.putAll() → cache = {new} | activeCache.getAndSet() → SWAP NGAY LẬP TỨC |
| **Microsecond 1003** | ⚠️ Có khoảng thời gian cache rỗng! | ✅ Thread khác đọc → thấy cache MỚI ngay |

---

## **PHẦN 5: NHỮNG LẦM TƯỞNG THƯỜNG GẶP CỦA NGƯỜI MỚI**

### **5.1. "Cache là một cái HashMap lưu trong RAM thôi mà!"**

#### **❌ Lầm tưởng:**
"Em chỉ cần khai báo `HashMap cache = new HashMap()` là xong, đơn giản thôi mà!"

#### **✅ Sự thật:**
Cache trong môi trường Production phải giải quyết 5 vấn đề lớn:

**1. Thread Safety (An toàn đa luồng):**
```
Câu hỏi: Khi có 100 thread cùng đọc cache, có bị crash không?
→ HashMap thường: CÓ thể crash!
→ ConcurrentHashMap: KHÔNG crash
```

**2. Update Strategy (Chiến lược cập nhật):**
```
Câu hỏi: Cập nhật cache như thế nào để không làm gián đoạn service?
→ Clear rồi add: Có khoảng trống → Service bị gián đoạn
→ Double Buffering: Swap nguyên tử → Zero downtime
```

**3. Resilience (Khả năng phục hồi):**
```
Câu hỏi: DB lỗi thì cache xử lý thế nào?
→ Không có logic: Cache bị xóa → Service chết
→ Có try-catch: Giữ cache cũ → Service vẫn sống
```

**4. Memory Management (Quản lý bộ nhớ):**
```
Câu hỏi: Cache lớn quá thì RAM bị full?
→ Không kiểm soát: RAM full → App crash
→ Có giới hạn: Chỉ cache 10,000 items, cũ nhất bị xóa
```

**5. Monitoring (Giám sát):**
```
Câu hỏi: Làm sao biết cache đang hoạt động tốt?
→ Không có log: Không biết gì cả
→ Có metrics: Biết cache size, hit rate, update time
```

---

### **5.2. "Đọc file từ FTP thôi, sao phải cache?"**

#### **❌ Lầm tưởng:**
"Em chỉ download file mà, query DB 5 records thông tin FTP rồi download, nhanh lắm, 100ms thôi!"

#### **✅ Phân tích chi tiết:**

**Kịch bản thực tế:**

```
Cấu hình hiện tại: sync-time = 30000ms (30 giây)

KHÔNG CÓ CACHE:
─────────────────────────────────────────────
Giây 0:  Query DB → 5ms   + Download FTP → 2000ms = 2005ms
Giây 30: Query DB → 5ms   + Download FTP → 2000ms = 2005ms
Giây 60: Query DB → 5ms   + Download FTP → 2000ms = 2005ms
...
Giây 86400 (1 ngày): 
├─ Số lần chạy: 86400 / 30 = 2880 lần
├─ Tổng thời gian query DB: 2880 × 5ms = 14,400ms = 14.4 giây
└─ Số lượng connections tới DB: 2880 connections

CÓ CACHE:
─────────────────────────────────────────────
Giây 0:  Load cache → 5ms + Download FTP → 2000ms = 2005ms
Giây 30: Đọc cache → 0.001ms + Download FTP → 2000ms = 2000ms
Giây 60: Đọc cache → 0.001ms + Download FTP → 2000ms = 2000ms
...
Giây 86400 (1 ngày):
├─ Số lần query DB: 2880 lần (refresh cache)
├─ Số lần đọc cache: 2880 lần (job download)
├─ Tổng thời gian query DB: 2880 × 5ms = 14.4 giây (CHỈ refresh cache)
├─ Tổng thời gian đọc cache: 2880 × 0.001ms = 2.88ms (job download)
└─ Tiết kiệm: 2880 connections không cần thiết tới DB
```

**Nhưng lợi ích THỰC SỰ không phải là tốc độ!**

#### **Lợi ích số 1: GIẢM COUPLING (Giảm phụ thuộc)**

```
KHÔNG CÓ CACHE:
┌─────────────────┐
│  JobFtp         │
│  (Download)     │
└────────┬────────┘
         │ ⚠️ Phụ thuộc trực tiếp
         ▼
┌─────────────────┐
│   Database      │
└─────────────────┘

→ DB chết = Job chết
→ DB chậm = Job chậm
→ DB bận = Job phải đợi

CÓ CACHE:
┌─────────────────┐         ┌─────────────────┐
│  JobFtp         │         │  CacheRefresh   │
│  (Download)     │         │  (Background)   │
└────────┬────────┘         └────────┬────────┘
         │                           │
         ▼                           ▼
┌─────────────────┐         ┌─────────────────┐
│  Cache (RAM)    │ ◄───────┤   Database      │
└─────────────────┘         └─────────────────┘

→ DB chết = Cache vẫn hoạt động → Job vẫn chạy được!
→ DB chậm = Không ảnh hưởng Job (chỉ ảnh hưởng refresh)
→ DB bận = Job không cần đợi (đọc từ RAM)
```

#### **Lợi ích số 2: SINGLE SOURCE OF TRUTH (Nguồn dữ liệu duy nhất)**

**Tình huống thực tế:**

```java
// ❌ Code KHÔNG TốT (query trực tiếp nhiều nơi)

// File 1: JobFtp.java
public void downloadFiles() {
    List<PromotionFtp> promotions = dbService.getPromotionFtpList(); // Query 1
    // ...
}

// File 2: MonitoringService.java
public int countActivePromotions() {
    List<PromotionFtp> promotions = dbService.getPromotionFtpList(); // Query 2
    return promotions.size();
}

// File 3: ReportService.java
public void generateReport() {
    List<PromotionFtp> promotions = dbService.getPromotionFtpList(); // Query 3
    // ...
}
```

**Vấn đề: DATA INCONSISTENCY (Dữ liệu không nhất quán)**

```
Timeline:

Giây 0:
├─ JobFtp query → 5 promotions
│
Giây 1: 
├─ DBA thêm promotion mới vào DB → 6 promotions
│
Giây 2:
├─ MonitoringService query → 6 promotions
│
Giây 3:
├─ ReportService query → 6 promotions

→ TRONG CÙNG 1 LẦN CHẠY:
  JobFtp thấy: 5 promotions
  Monitoring thấy: 6 promotions
  Report thấy: 6 promotions
  
  ⚠️ DỮ LIỆU KHÔNG NHẤT QUÁN!
  ⚠️ Bug khó debug: "Sao job chỉ chạy 5 mà report lại báo 6?"
```

**Với Cache:**

```java
// ✅ Code TỐT (cache là nguồn duy nhất)

// File 1: JobFtp.java
public void downloadFiles() {
    Map<String, PromotionFtp> promotions = promotionFtpCache.getCache(); // Đọc cache
    // ...
}

// File 2: MonitoringService.java
public int countActivePromotions() {
    return promotionFtpCache.getCacheSize(); // Đọc cache
}

// File 3: ReportService.java
public void generateReport() {
    Map<String, PromotionFtp> promotions = promotionFtpCache.getCache(); // Đọc cache
    // ...
}
```

**Kết quả:**

```
Giây 0-29: Tất cả đều thấy snapshot GIỐNG NHAU (5 promotions)
Giây 30: Cache refresh → Tất cả cùng chuyển sang snapshot MỚI (6 promotions)
Giây 31-59: Tất cả đều thấy snapshot GIỐNG NHAU (6 promotions)

→ ✅ DỮ LIỆU NHẤT QUÁN trong cùng một chu kỳ!
→ ✅ Dễ debug: Biết chính xác cache nào đang được dùng
```

---

### **5.3. "AtomicReference là gì? Sao không dùng synchronized?"**

#### **❌ Lầm tưởng:**
"Em biết `synchronized` rồi, dùng nó lock lại là an toàn mà!"

```java
// ❌ Cách người mới nghĩ
public class SlowCache {
    private HashMap<String, PromotionFtp> cache = new HashMap<>();
    
    public synchronized void updateCache() {  // ⚠️ Lock toàn bộ method
        cache.clear();
        cache.putAll(fetchFromDB());
    }
    
    public synchronized PromotionFtp get(String key) {  // ⚠️ Lock toàn bộ method
        return cache.get(key);
    }
}
```

#### **✅ Vấn đề của synchronized:**

**Hiện tượng: BLOCKING (Chặn toàn bộ luồng khác)**

```
Timeline chi tiết:

Millisecond 0:
├─ Thread 1 (Update): Bắt đầu updateCache()
│   └─ ⚠️ LOCK toàn bộ object SlowCache
│
Millisecond 1:
├─ Thread 2 (Read): Gọi get("1454")
│   └─ ⛔ BỊ CHẶN! Phải đợi Thread 1 xong
│
Millisecond 2:
├─ Thread 3 (Read): Gọi get("1455")
│   └─ ⛔ BỊ CHẶN! Phải đợi Thread 1 xong
│
Millisecond 3:
├─ Thread 4 (Read): Gọi get("1456")
│   └─ ⛔ BỊ CHẶN! Phải đợi Thread 1 xong
│
...
Millisecond 100:
├─ Thread 1 (Update): fetchFromDB() xong (mất 100ms)
│   └─ 🔓 UNLOCK object
│
Millisecond 101:
├─ Thread 2, 3, 4: Tranh nhau chạy (còn bị chặn lẫn nhau!)

→ KẾT QUẢ: 
  - Trong 100ms update, TẤT CẢ request đều bị ĐỨNG IM!
  - Throughput giảm mạnh
  - Latency tăng cao
```

**Đo lường hiệu suất:**

```
Benchmark test: 1000 threads cùng đọc cache

synchronized HashMap:
├─ Throughput: 10,000 requests/second
├─ Latency P99: 50ms
└─ CPU usage: 80% (do context switching)

AtomicReference + ConcurrentHashMap:
├─ Throughput: 1,000,000 requests/second
├─ Latency P99: 0.1ms
└─ CPU usage: 20%

→ Nhanh hơn 100 LẦN!
```

#### **✅ Giải pháp của AtomicReference:**

**1. Non-blocking (Không chặn):**

```java
// ✅ Code tối ưu
public class FastCache {
    private final AtomicReference<ConcurrentHashMap<String, PromotionFtp>> activeCache;
    
    public void updateCache() {
        ConcurrentHashMap<String, PromotionFtp> newData = fetchFromDB(); // Không lock!
        activeCache.getAndSet(newData); // Chỉ lock 1 nanosecond!
    }
    
    public PromotionFtp get(String key) {
        return activeCache.get().get(key); // Hoàn toàn không lock!
    }
}
```

**Timeline:**

```
Millisecond 0:
├─ Thread 1 (Update): Bắt đầu updateCache()
│   └─ fetchFromDB() → Không lock gì cả!
│
Millisecond 1:
├─ Thread 2 (Read): get("1454")
│   └─ ✅ ĐỌC ĐƯỢC NGAY! (từ cache cũ)
│
Millisecond 2:
├─ Thread 3 (Read): get("1455")
│   └─ ✅ ĐỌC ĐƯỢC NGAY! (từ cache cũ)
│
...
Millisecond 100:
├─ Thread 1: fetchFromDB() xong
│   └─ activeCache.getAndSet(newData) → ⚡ Lock chỉ 0.000001ms!
│
Millisecond 100.000001:
├─ Thread 4 (Read): get("1456")
│   └─ ✅ ĐỌC ĐƯỢC NGAY! (từ cache mới)

→ KẾT QUẢ:
  - Chỉ có 1 nanosecond bị lock!
  - Tất cả request khác chạy song song
  - Zero downtime!
```

**2. Compare-And-Swap (CAS) - Cơ chế CPU:**

```
Giải thích bằng hình ảnh:

synchronized:
┌──────────────────────────────────────┐
│  🚪 CỬA PHÒNG (chỉ 1 người vào được)  │
│                                      │
│  Thread 1: Đang ở trong → LOCK       │
│  Thread 2: Đợi ngoài cửa             │
│  Thread 3: Đợi ngoài cửa             │
│  Thread 4: Đợi ngoài cửa             │
└──────────────────────────────────────┘

AtomicReference (CAS):
┌──────────────────────────────────────┐
│  📋 BẢNG THÔNG BÁO (ai cũng đọc được) │
│                                      │
│  Thread 1: Đọc → ✅                  │
│  Thread 2: Đọc → ✅                  │
│  Thread 3: Đọc → ✅                  │
│  Thread 4: Cập nhật → ⚡ Swap nhanh  │
└──────────────────────────────────────┘
```

**CAS hoạt động như thế nào:**

```java
// Pseudo-code của getAndSet()
public T getAndSet(T newValue) {
    T currentValue;
    do {
        currentValue = this.value;  // Đọc giá trị hiện tại
        // ⚡ CPU instruction: CMPXCHG (Compare and Exchange)
        // Nếu value vẫn = currentValue → swap thành newValue
        // Nếu value đã bị đổi bởi thread khác → retry
    } while (!compareAndSwap(currentValue, newValue));
    
    return currentValue;
}
```

**Tại sao nhanh hơn synchronized:**

```
synchronized:
1. Acquire lock → OS kernel call → Context switch
2. Thực hiện code
3. Release lock → OS kernel call → Context switch
→ Tổng: ~1000 CPU cycles

CAS (AtomicReference):
1. CPU instruction CMPXCHG (1 instruction duy nhất)
→ Tổng: ~10 CPU cycles

→ Nhanh hơn 100 LẦN!
```

---

### **5.4. "Tại sao cần 2 buffer? 1 cache không đủ sao?"**

#### **❌ Lầm tưởng:**
"Em chỉ cần 1 HashMap, update trực tiếp lên đó là xong mà!"

#### **✅ So sánh Single Buffer vs Double Buffer:**

**SINGLE BUFFER (1 cache):**

```java
// ❌ Cách làm ngây thơ
public class SingleBufferCache {
    private ConcurrentHashMap<String, PromotionFtp> cache = new ConcurrentHashMap<>();
    
    public void updateCache() {
        ConcurrentHashMap<String, PromotionFtp> newData = fetchFromDB();
        
        // ⚠️ VẤN ĐỀ: Phải xóa từng item cũ rồi add item mới
        cache.clear();  // ← Bước 1: Cache rỗng!
        cache.putAll(newData);  // ← Bước 2: Add dữ liệu mới
    }
}
```

**Vấn đề:**

```
Timeline:

Microsecond 0:
├─ Update thread: cache.clear()
│   └─ cache = {} (RỖNG!)
│
Microsecond 1-100: ⚠️ KHOẢNG THỜI GIAN NGUY HIỂM!
├─ Thread A: get("1454") → null → NullPointerException!
├─ Thread B: get("1455") → null → NullPointerException!
├─ Thread C: getCacheSize() → 0 → Cảnh báo sai "Cache rỗng!"
│
Microsecond 101:
└─ Update thread: cache.putAll(newData)
    └─ cache = {5 items}

→ KẾT QUẢ: Có 100 microsecond cache BỊ RỖNG!
          Mọi request trong khoảng này đều FAIL!
```

**DOUBLE BUFFER (2 cache):**

```java
// ✅ Cách làm chuyên nghiệp
public class DoubleBufferCache {
    private AtomicReference<ConcurrentHashMap<String, T>> activeCache;   // Buffer A
    private AtomicReference<ConcurrentHashMap<String, T>> stagingCache;  // Buffer B
    
    public void updateCache() {
        // Bước 1: Chuẩn bị dữ liệu mới ở stagingCache (không ảnh hưởng activeCache)
        ConcurrentHashMap<String, T> staging = stagingCache.get();
        staging.clear();
        staging.putAll(fetchFromDB());
        
        // Bước 2: SWAP nguyên tử
        activeCache.getAndSet(staging);  // ⚡ Chỉ mất 1 nanosecond!
    }
}
```

**Hoạt động:**

```
Timeline:

Microsecond 0:
├─ activeCache = {1454: A, 1455: B, 1456: C} (5 items) ← Đang phục vụ
├─ stagingCache = {rỗng hoặc data cũ}
│
Microsecond 1-100: (Chuẩn bị dữ liệu mới)
├─ Update thread: Làm việc với stagingCache
│   ├─ stagingCache.clear()
│   ├─ stagingCache.put("1454", A_new)
│   ├─ stagingCache.put("1455", B_new)
│   └─ stagingCache.put("1457", D_new)  (có thêm promotion mới)
│
├─ TRONG LÚC NÀY:
│   ├─ Thread A: activeCache.get("1454") → ✅ Vẫn trả về A (data cũ)
│   ├─ Thread B: activeCache.get("1455") → ✅ Vẫn trả về B (data cũ)
│   └─ Thread C: activeCache.getCacheSize() → ✅ Vẫn trả về 5
│
Microsecond 101: (SWAP NGUYÊN TỬ)
├─ previousActive = activeCache.getAndSet(stagingCache)
│   ├─ activeCache BÂY GIỜ MỚI trỏ đến staging (6 items mới)
│   └─ stagingCache trỏ đến previousActive (5 items cũ)
│
Microsecond 102:
├─ Thread A: activeCache.get("1454") → ✅ Trả về A_new (data mới)
├─ Thread B: activeCache.get("1457") → ✅ Trả về D_new (promotion mới)
└─ Thread C: activeCache.getCacheSize() → ✅ Trả về 6

→ KẾT QUẢ: ZERO DOWNTIME! Không có 1 microsecond nào cache bị rỗng!
```

**Minh họa bằng hình ảnh:**

```
SINGLE BUFFER = Sửa nhà khi đang ở:
┌─────────────────────────────────┐
│  🏠 NGÔI NHÀ DUY NHẤT            │
│                                 │
│  Bước 1: Phá tường → 🏚️ Không ở được!
│  Bước 2: Xây lại → 🏗️ Vẫn chưa ở được!
│  Bước 3: Hoàn thiện → 🏠 Mới ở được!
│                                 │
│  ⚠️ Phải đi thuê nhà tạm!       │
└─────────────────────────────────┘

DOUBLE BUFFER = Có 2 nhà, chuyển qua lại:
┌─────────────────────────────────┐
│  🏠 NHÀ A (đang ở)              │
│  🏗️ NHÀ B (đang sửa)            │
│                                 │
│  Bước 1: Ở nhà A, sửa nhà B     │
│  Bước 2: Chuyển sang nhà B ⚡    │
│  Bước 3: Ở nhà B, sửa nhà A     │
│                                 │
│  ✅ Luôn có nhà để ở!           │
└─────────────────────────────────┘
```

---

### **5.5. "@Scheduled thì chạy tự động rồi, tại sao cần @PostConstruct?"**

#### **❌ Lầm tưởng:**
"Có @Scheduled(fixedDelay=30000) rồi thì cứ 30s nó tự chạy, không cần @PostConstruct làm gì!"

#### **✅ Vấn đề: COLD START (Khởi động lạnh)**

**Code KHÔNG có @PostConstruct:**

```java
// ❌ Code thiếu sót
@Component
public class BadCache extends CacheSwapService<PromotionFtp> {
    
    @Scheduled(fixedDelay = 30000)  // Chạy sau 30s
    public void forceRefresh() {
        super.forceRefresh();
    }
    
    // ⚠️ THIẾU @PostConstruct!
}
```

**Timeline khi app khởi động:**

```
Giây 0:
├─ Spring Boot khởi động
├─ BadCache bean được tạo
└─ activeCache = {} (RỖNG!)  ⚠️

Giây 1:
├─ JobManageService chạy
├─ Gọi badCache.getCache()
└─ return {} (RỖNG!) → ⛔ KHÔNG download file nào!

Giây 10:
├─ API request: GET /promotions
└─ return [] (RỖNG!) → ⛔ Khách hàng thấy "Không có promotion"!

Giây 29:
└─ Cache vẫn rỗng... ⏳

Giây 30: ✅ LẦN ĐẦU @Scheduled chạy
├─ Query DB → 5 items
└─ activeCache = {5 items}

Giây 31:
├─ JobManageService chạy
└─ ✅ BÂY GIỜ MỚI download được!

→ KẾT QUẢ: 30 GIÂY ĐẦU TIÊN APP HOẠT ĐỘNG SAI!
          Khách hàng phàn nàn ngay khi deploy!
```

**Code CÓ @PostConstruct:**

```java
// ✅ Code đầy đủ
@Component
public class GoodCache extends CacheSwapService<PromotionFtp> {
    
    @PostConstruct  // ← Chạy NGAY khi bean được tạo
    public void init() {
        super.forceRefresh();  // Load cache ngay lập tức
    }
    
    @Scheduled(fixedDelay = 30000)
    public void forceRefresh() {
        super.forceRefresh();
    }
}
```

**Timeline:**

```
Giây 0:
├─ Spring Boot khởi động
├─ GoodCache bean được tạo
├─ @PostConstruct chạy → Load DB ngay!
└─ activeCache = {5 items} ✅ NGAY LẬP TỨC!

Giây 1:
├─ JobManageService chạy
└─ ✅ Download file THÀNH CÔNG ngay!

Giây 10:
├─ API request: GET /promotions
└─ ✅ Return 5 promotions!

Giây 30:
└─ @Scheduled chạy lần đầu (refresh cache)

→ KẾT QUẢ: App hoạt động NGAY TỪ GIÂY ĐẦU TIÊN!
```

**Minh họa:**

```
KHÔNG CÓ @PostConstruct = Nhà hàng mở cửa mà tủ lạnh rỗng:
┌────────────────────────────────────────┐
│  🍽️ NHÀ HÀNG                           │
│                                        │
│  6:00 - Mở cửa                         │
│  6:01 - Khách vào: "Cho tô phở"        │
│         → ⚠️ "Chưa có nguyên liệu!"    │
│  6:30 - Xe tải giao hàng đến           │
│  6:31 - Mới nấu được                   │
│                                        │
│  ⚠️ Mất 30 phút đầu!                   │
└────────────────────────────────────────┘

CÓ @PostConstruct = Chuẩn bị nguyên liệu trước khi mở cửa:
┌────────────────────────────────────────┐
│  🍽️ NHÀ HÀNG                           │
│                                        │
│  5:30 - Nhận nguyên liệu (@PostConstruct)
│  6:00 - Mở cửa                         │
│  6:01 - Khách vào: "Cho tô phở"        │
│         → ✅ "Dạ, ngay ạ!"             │
│                                        │
│  ✅ Phục vụ ngay từ phút đầu!          │
└────────────────────────────────────────┘
```

---

### **5.6. "fixedDelay vs fixedRate khác nhau gì?"**

#### **❌ Lầm tưởng:**
"Đều là 30 giây chạy 1 lần, giống nhau mà!"

#### **✅ Sự khác biệt lớn:**

**fixedDelay = Đợi xong mới đếm thời gian:**

```java
@Scheduled(fixedDelay = 30000)  // 30s SAU KHI task hoàn thành
public void refreshCache() {
    // Task này mất 5 giây
}
```

**Timeline:**

```
Giây 0: Bắt đầu task
Giây 5: Task hoàn thành ✅
Giây 35: Bắt đầu task tiếp theo (5 + 30 = 35)
Giây 40: Task hoàn thành ✅
Giây 70: Bắt đầu task tiếp theo (40 + 30 = 70)

→ Chu kỳ thực tế: 35 giây (không phải 30!)
→ INTERVAL = fixedDelay + execution_time
```

**fixedRate = Đếm thời gian từ lúc bắt đầu:**

```java
@Scheduled(fixedRate = 30000)  // 30s TỪ LÚC task bắt đầu
public void refreshCache() {
    // Task này mất 5 giây
}
```

**Timeline:**

```
Giây 0: Bắt đầu task
Giây 5: Task hoàn thành ✅
Giây 30: Bắt đầu task tiếp theo (bất kể task trước mất bao lâu)
Giây 35: Task hoàn thành ✅
Giây 60: Bắt đầu task tiếp theo

→ Chu kỳ thực tế: ĐÚNG 30 giây
→ INTERVAL = fixedRate (không phụ thuộc execution_time)
```

**Khi nào dùng cái nào:**

```
✅ Dùng fixedDelay khi:
- Task có thời gian thực thi KHÔNG ỔN ĐỊNH
- Muốn đảm bảo có KHOẢNG NGHỈ giữa các lần chạy
- Ví dụ: Query DB (có thể nhanh 100ms, có thể chậm 10s)

❌ KHÔNG dùng fixedRate khi:
- Task chạy lâu hơn interval
- Có thể gây OVERLAP (task mới chạy khi task cũ chưa xong)

Ví dụ NGUY HIỂM với fixedRate:
@Scheduled(fixedRate = 10000)  // 10s
public void slowTask() {
    // Task này mất 15 giây!
}

Timeline:
Giây 0: Task 1 bắt đầu
Giây 10: Task 2 bắt đầu (Task 1 vẫn chưa xong!) ⚠️
Giây 15: Task 1 xong
Giây 20: Task 3 bắt đầu (Task 2 vẫn chưa xong!) ⚠️

→ Có thể có 2-3 task chạy ĐỒNG THỜI!
→ CPU/Memory/DB bị QUÁNG! → App CRASH!
```

**Tại sao code của bạn dùng fixedDelay:**

```java
@Scheduled(fixedDelayString = "${app.sql.sync-time}")  // 30000ms
public void forceRefresh() {
    super.forceRefresh();  // Query DB - thời gian không ổn định
}
```

**Lý do:**
- Query DB có thể nhanh (100ms) hoặc chậm (5s) tùy tải hệ thống
- fixedDelay đảm bảo luôn có ít nhất 30s nghỉ giữa các lần query
- Tránh làm quá tải Database

---

## **PHẦN 6: KẾT LUẬN & CHECKLIST**

### **6.1. Tóm tắt: Tại sao cần Cache cho ứng dụng FTP này?**

#### **Không phải vì tốc độ (5ms → 0.001ms)**
#### **Mà vì:**

**1. RESILIENCE (Khả năng phục hồi):**
```
DB chết → App vẫn sống
DB chậm → App vẫn nhanh
DB bị tấn công → App không bị ảnh hưởng
```

**2. CONSISTENCY (Tính nhất quán):**
```
Tất cả components đọc cùng 1 snapshot
Không bị race condition giữa các thread
Dữ liệu đồng bộ trong cùng chu kỳ
```

**3. DECOUPLING (Giảm phụ thuộc):**
```
Job không phụ thuộc trực tiếp vào DB
Có thể restart DB mà không ảnh hưởng service
Dễ dàng thay đổi DB schema mà không ảnh hưởng code
```

**4. THREAD SAFETY (An toàn đa luồng):**
```
100 threads đồng thời đọc cache → OK
Update cache không block read operations → Zero downtime
```

---

### **6.2. Checklist: Đánh giá 1 hệ thống Cache có tốt không**

Khi bạn gặp bất kỳ code cache nào, hãy hỏi 10 câu hỏi này:

```
✅ 1. Thread-safe không?
   → Có dùng ConcurrentHashMap hoặc synchronized?

✅ 2. Zero-downtime khi update không?
   → Có dùng Double Buffering hay Atomic Swap?

✅ 3. Xử lý lỗi thế nào?
   → DB lỗi thì cache làm gì? (Giữ data cũ hay crash?)

✅ 4. Cold start như thế nào?
   → App vừa khởi động có ngay data không? (@PostConstruct?)

✅ 5. Refresh strategy ra sao?
   → fixedDelay hay fixedRate? Tại sao?

✅ 6. Memory có bị leak không?
   → Cache có giới hạn size? Có cơ chế eviction?

✅ 7. Monitoring thế nào?
   → Có log size, update time không?

✅ 8. Data consistency?
   → Tất cả nơi đọc cache có thấy cùng 1 snapshot không?

✅ 9. Có Single Point of Failure không?
   → Cache chết thì app chết luôn không?

✅ 10. Performance có đo được không?
    → Hit rate bao nhiêu? Latency ra sao?
```

---

### **6.3. Bài tập thực hành (để hiểu sâu hơn)**

#### **Bài 1: Phá vỡ cache xem điều gì xảy ra**

```java
// Thử sửa code:
@Component
public class BrokenCache extends CacheSwapService<PromotionFtp> {
    
    // ❌ XÓA @PostConstruct đi
    // public void forceRefresh() { ... }
    
    @Scheduled(fixedDelay = 300000)  // Đổi thành 5 PHÚT
    public void slowRefresh() {
        super.forceRefresh();
    }
}
```

**Quan sát:**
- App khởi động → 5 phút đầu không download file
- Thêm promotion mới vào DB → Phải đợi tối đa 5 phút mới thấy
- **KẾT LUẬN:** @PostConstruct và sync-time ngắn rất quan trọng!

#### **Bài 2: Test thread safety**

```java
// Viết test:
@Test
public void testConcurrentRead() throws Exception {
    PromotionFtpCache cache = new PromotionFtpCache(dbService);
    
    // Tạo 1000 threads cùng đọc cache
    ExecutorService executor = Executors.newFixedThreadPool(1000);
    
    for (int i = 0; i < 1000; i++) {
        executor.submit(() -> {
            for (int j = 0; j < 10000; j++) {
                cache.getCache().get("1454");  // Đọc 10,000 lần
            }
        });
    }
    
    // Trong khi đó, update cache mỗi 100ms
    ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
    scheduler.scheduleAtFixedRate(() -> cache.forceRefresh(), 0, 100, TimeUnit.MILLISECONDS);
    
    executor.shutdown();
    executor.awaitTermination(1, TimeUnit.MINUTES);
    
    // ✅ Nếu code tốt: Không có exception nào!
    // ❌ Nếu code xấu: ConcurrentModificationException!
}
```

#### **Bài 3: So sánh performance**

```java
// Đo thời gian:
public class PerformanceTest {
    
    public void testDirectDB() {
        long start = System.nanoTime();
        for (int i = 0; i < 10000; i++) {
            dbService.getPromotionFtpMap();  // Query trực tiếp
        }
        long end = System.nanoTime();
        System.out.println("Direct DB: " + (end - start) / 1_000_000 + "ms");
    }
    
    public void testWithCache() {
        cache.forceRefresh();  // Load 1 lần
        long start = System.nanoTime();
        for (int i = 0; i < 10000; i++) {
            cache.getCache();  // Đọc từ cache
        }
        long end = System.nanoTime();
        System.out.println("With Cache: " + (end - start) / 1_000_000 + "ms");
    }
}

// Kết quả mong đợi:
// Direct DB: ~50,000ms (5s query × 10,000 lần)
// With Cache: ~10ms (0.001ms × 10,000 lần)
// → Nhanh hơn 5000 LẦN!
```

---

### **6.4. Tài liệu tham khảo để học sâu hơn**

**1. Java Concurrency in Practice** (Brian Goetz)
- Chương 15: Atomic Variables and Non-blocking Synchronization
- Giải thích chi tiết về AtomicReference, CAS

**2. Spring Framework Documentation**
- @Scheduled annotation
- Task Execution and Scheduling

**3. Patterns of Enterprise Application Architecture** (Martin Fowler)
- Cache-Aside Pattern
- Double Buffering Pattern

---

## **LỜI KẾT:**

Em ơi, cache không chỉ là "HashMap để tăng tốc độ". Đó là một **kiến trúc phức tạp** để giải quyết nhiều vấn đề:
- Thread safety (an toàn đa luồng)
- Zero downtime (không gián đoạn dịch vụ)  
- Resilience (khả năng phục hồi khi lỗi)
- Consistency (tính nhất quán dữ liệu)

Code của dự án em là một **ví dụ điển hình** của cache production-ready. Hãy đọc kỹ từng dòng, hiểu tại sao nó được viết như vậy.

**Điều quan trọng nhất:** Đừng chỉ học cách code chạy được. Hãy học cách code **chạy tốt trong production** khi có hàng triệu request, khi DB lỗi, khi có 100 threads cùng truy cập!

---

**Chúc em học tốt! 🚀**

---
*Tài liệu được tạo: 26/12/2025*
*Dự án: FTP-CTKM*
*Chủ đề: Cache Mechanism với Double Buffering*
