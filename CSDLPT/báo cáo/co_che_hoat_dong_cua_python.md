# 📖 Giải Thích Chi Tiết: config.py & database.py

---

## Phần 1: Cơ Chế `import` Trong Python — Cách File Này Dùng Được Biến Của File Kia

### 1.1. Nguyên Lý Cốt Lõi

Trong Python, mỗi file `.py` được gọi là một **module**. Khi bạn viết:

```python
from backend.config import DATABASE_URL
```

Python sẽ thực hiện 3 bước:

```
Bước 1: Tìm file   → backend/config.py
Bước 2: Chạy file   → Thực thi TẤT CẢ code trong config.py từ trên xuống dưới
Bước 3: Lấy biến    → "Rút" biến DATABASE_URL ra, mang vào file hiện tại để dùng
```

> [!IMPORTANT]
> Điểm then chốt: Khi Python `import` một file, nó **chạy toàn bộ code** trong file đó. Mọi biến được gán giá trị (ví dụ `SAMPLE_RATE = 22050`) sẽ trở thành "tài sản" mà các file khác có thể lấy ra dùng.

### 1.2. Ví Dụ Cụ Thể Trong Dự Án

```
config.py định nghĩa:        SAMPLE_RATE = 22050
                                    ↓
preprocessor.py viết:         from backend.config import SAMPLE_RATE
                                    ↓
preprocessor.py dùng:         y = librosa.load(file, sr=SAMPLE_RATE)
                              # → y = librosa.load(file, sr=22050)
```

Hiệu quả giống hệt việc bạn viết trực tiếp `22050` trong preprocessor.py, nhưng **gom tất cả hằng số về 1 chỗ** để dễ quản lý.

### 1.3. Tại Sao Tách Ra File Riêng?

Nếu không tách, bạn phải viết:

```python
# preprocessor.py
y = librosa.load(file, sr=22050)    # Viết cứng số 22050

# extractor.py  
mfcc = librosa.feature.mfcc(y, sr=22050)   # Lại viết cứng 22050

# batch_extract.py
print(f"Sample rate: 22050")   # Lại viết cứng 22050
```

**Vấn đề**: Nếu muốn đổi sang 16000 Hz, phải sửa ở **tất cả** các file → dễ sót, dễ sai.

Tách ra `config.py`:

```python
# config.py — SỬA 1 LẦN DUY NHẤT
SAMPLE_RATE = 22050   # Đổi thành 16000 ở đây → tất cả file tự cập nhật

# preprocessor.py
from backend.config import SAMPLE_RATE
y = librosa.load(file, sr=SAMPLE_RATE)    # Tự động dùng giá trị mới
```

---

## Phần 2: config.py — "Bảng Điều Khiển Trung Tâm"

File [config.py](file:///d:/PTIT/kì%202%20năm%204/Cơ%20sở%20dữ%20liệu%20đa%20phương%20tiện/CSDLDPT/backend/config.py) chứa **tất cả hằng số** cấu hình của hệ thống. Nó được chia thành 4 khối:

### Khối 1: Kết nối Database (dòng 8–15)

```python
DB_HOST = os.getenv("DB_HOST", "15.134.248.39")
DB_PORT = os.getenv("DB_PORT", "5432")
DB_NAME = os.getenv("DB_NAME", "audiostring_db")
DB_USER = os.getenv("DB_USER", "user1")
DB_PASSWORD = os.getenv("DB_PASSWORD", "123456")

DATABASE_URL = f"postgresql://{DB_USER}:{DB_PASSWORD}@{DB_HOST}:{DB_PORT}/{DB_NAME}"
```

**Giải thích `os.getenv("DB_HOST", "15.134.248.39")`:**

```
os.getenv("DB_HOST", "15.134.248.39")
         ↑ tên biến       ↑ giá trị mặc định
         môi trường

→ Nghĩa là: "Hỏi hệ điều hành có biến DB_HOST không?
   - Nếu CÓ → dùng giá trị đó
   - Nếu KHÔNG → dùng mặc định '15.134.248.39'"
```

Kết quả cuối cùng `DATABASE_URL` sẽ là chuỗi kết nối hoàn chỉnh:
```
postgresql://user1:123456@15.134.248.39:5432/audiostring_db
```

**Ai dùng?** → `database.py` import biến này để kết nối PostgreSQL.

### Khối 2: Đường dẫn thư mục (dòng 19–26)

```python
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
DATASET_DIR = os.path.join(BASE_DIR, "dataset")
METADATA_CSV = os.path.join(DATASET_DIR, "metadata.csv")
UPLOAD_DIR = os.path.join(BASE_DIR, "uploads")

os.makedirs(UPLOAD_DIR, exist_ok=True)
```

**Phân tích `BASE_DIR`** — đây là cách Python tìm thư mục gốc của dự án:

```
__file__                = "D:/PTIT/.../CSDLDPT/backend/config.py"
os.path.abspath(↑)      = "D:/PTIT/.../CSDLDPT/backend/config.py"   (đường dẫn tuyệt đối)
os.path.dirname(↑)      = "D:/PTIT/.../CSDLDPT/backend"             (bỏ tên file → lấy thư mục cha)
os.path.dirname(↑)      = "D:/PTIT/.../CSDLDPT"                     (lên thêm 1 cấp → thư mục gốc)
```

Kết quả:
```
BASE_DIR     = "D:/PTIT/.../CSDLDPT"
DATASET_DIR  = "D:/PTIT/.../CSDLDPT/dataset"
METADATA_CSV = "D:/PTIT/.../CSDLDPT/dataset/metadata.csv"
UPLOAD_DIR   = "D:/PTIT/.../CSDLDPT/uploads"
```

`os.makedirs(UPLOAD_DIR, exist_ok=True)` → Tự động tạo thư mục `uploads/` nếu chưa tồn tại. Dòng này **chạy ngay khi** bất kỳ file nào `import config`.

**Ai dùng?**
- `batch_extract.py` dùng `DATASET_DIR` để quét file .wav
- `main.py` dùng `UPLOAD_DIR` để lưu file user upload

### Khối 3: Tham số DSP (dòng 30–38)

```python
SAMPLE_RATE = 22050       # Tần số lấy mẫu (Hz)
FRAME_SIZE = 2048         # Kích thước cửa sổ FFT (samples)
HOP_SIZE = 512            # Bước nhảy giữa 2 cửa sổ
N_MELS = 40               # Số dải Mel filter bank
N_MFCC = 13               # Số hệ số MFCC trích xuất
TRIM_TOP_DB = 60           # Ngưỡng trim silence (dB)
SKIP_SECONDS = 0.0        # Bỏ qua mấy giây đầu
EXTRACT_SECONDS = 3.0     # Cắt lấy bao nhiêu giây
```

**Ai dùng?** → `preprocessor.py`, `cepstral.py`, `spectral.py`, `temporal.py` — tất cả đều import các hằng số này thay vì viết cứng số.

### Khối 4: Cấu trúc Vector (dòng 42–57)

```python
PITCH_DIM = 1              # Pitch vector có 1 chiều
TIMBRE_DIM = 18            # Timbre vector có 18 chiều

FEATURE_NAMES_TIMBRE = [
    *[f"mfcc_mean_{i}" for i in range(1, 11)],   # 10 tên: mfcc_mean_1 → mfcc_mean_10
    *[f"contrast_{i}" for i in range(1, 5)],      # 4 tên:  contrast_1 → contrast_4
    *[f"mfcc_std_{i}" for i in range(1, 5)],      # 4 tên:  mfcc_std_1 → mfcc_std_4
]

TOP_K = 5                  # Số kết quả trả về cho user
```

**Ai dùng?** → `fit_scaler.py` dùng `PITCH_DIM`, `main.py` dùng `TOP_K`.

---

## Phần 3: database.py — "Cổng Giao Tiếp Với PostgreSQL"

File [database.py](file:///d:/PTIT/kì%202%20năm%204/Cơ%20sở%20dữ%20liệu%20đa%20phương%20tiện/CSDLDPT/backend/database.py) cung cấp **3 hàm** để toàn bộ hệ thống giao tiếp với database. Không file nào trong dự án tự kết nối DB — tất cả đều gọi qua file này.

### Hàm 1: `get_connection()` (dòng 9–12)

```python
from backend.config import DATABASE_URL   # ← Lấy chuỗi kết nối từ config.py

def get_connection():
    conn = psycopg2.connect(DATABASE_URL)
    return conn
```

**Hoạt động:**
```
1. Import DATABASE_URL từ config.py
   → "postgresql://user1:123456@15.134.248.39:5432/audiostring_db"

2. psycopg2.connect(DATABASE_URL)
   → Mở kết nối TCP tới máy chủ PostgreSQL tại IP 15.134.248.39
   → Đăng nhập bằng user1/123456
   → Chọn database audiostring_db

3. Trả về đối tượng "conn" (connection)
   → Các file khác dùng conn để gửi lệnh SQL
```

**Ai gọi hàm này?**
- `fit_scaler.py` gọi `conn = get_connection()` để đọc/ghi vector
- `similarity.py` gọi `conn = get_connection()` để chạy câu SQL tìm kiếm
- `execute_query()` (hàm thứ 2 ngay bên dưới) cũng gọi hàm này

### Hàm 2: `execute_query()` (dòng 15–28)

```python
def execute_query(query: str, params=None, fetch=False):
    conn = get_connection()                          # Bước 1: Mở kết nối
    try:
        with conn.cursor(cursor_factory=RealDictCursor) as cur:
            cur.execute(query, params)               # Bước 2: Chạy SQL
            if fetch:
                result = cur.fetchall()              # Bước 3a: Lấy kết quả (SELECT)
            else:
                result = None                        # Bước 3b: Không lấy (INSERT/UPDATE)
            conn.commit()                            # Bước 4: Xác nhận thay đổi
        return result
    finally:
        conn.close()                                 # Bước 5: LUÔN đóng kết nối
```

**Ví dụ cách file khác gọi hàm này:**

```python
# normalizer.py — Đọc scaler từ DB
from backend.database import execute_query

rows = execute_query(
    "SELECT mean_vec, std_vec FROM scaler_params WHERE version = %s",
    (10,),          # params: version = 10
    fetch=True      # CẦN lấy kết quả trả về
)
# → rows = [{"mean_vec": [58.3], "std_vec": [14.7]}]
```

**Giải thích `RealDictCursor`:**
```
Cursor thường:     row = (58.3, 14.7)          → Truy cập bằng số: row[0], row[1]
RealDictCursor:    row = {"mean_vec": 58.3, "std_vec": 14.7}  → Truy cập bằng tên: row["mean_vec"]
```

**Giải thích `try/finally`:**
```
try:
    ... làm việc với DB ...
finally:
    conn.close()    ← DÙ THÀNH CÔNG HAY LỖI, luôn đóng kết nối
                       Nếu không đóng → rò rỉ kết nối → DB hết slot → sập hệ thống
```

### Hàm 3: `run_migrations()` (dòng 31–47)

```python
def run_migrations(migrations_dir: str):
    sql_files = sorted(glob.glob(os.path.join(migrations_dir, "*.sql")))
    conn = get_connection()
    try:
        with conn.cursor() as cur:
            for sql_file in sql_files:
                with open(sql_file, "r", encoding="utf-8") as f:
                    cur.execute(f.read())           # Đọc file .sql và chạy
            conn.commit()
    finally:
        conn.close()
```

**Hoạt động:**
```
1. Quét thư mục migrations/ → Tìm tất cả file .sql
2. Sắp xếp theo tên (sorted) → 001_create_tables.sql chạy trước 002_create_views.sql
3. Với mỗi file .sql:
   - Đọc nội dung SQL (CREATE TABLE, CREATE INDEX, CREATE VIEW...)
   - Gửi cho PostgreSQL thực thi
4. commit() → Lưu tất cả thay đổi
```

**Ai gọi?** → Lệnh khởi tạo hệ thống:
```bash
python -c "from backend.database import run_migrations; run_migrations('migrations')"
```

---

## Phần 4: Bản Đồ Ai Import Ai

```
config.py ──────────────────────────────────────────────────┐
  │ DATABASE_URL                                            │ SAMPLE_RATE, N_MFCC,
  ↓                                                         │ TRIM_TOP_DB, ...
database.py                                                 ↓
  │ get_connection()                              preprocessor.py
  │ execute_query()                               cepstral.py
  ↓                                               spectral.py
┌─────────────┐    ┌──────────────┐               temporal.py
│fit_scaler.py│    │normalizer.py │                    │
│  gọi        │    │  gọi         │                    ↓
│get_connection│   │execute_query │              extractor.py
└─────────────┘    └──────────────┘                    │
                         │                             ↓
                         ↓                        main.py ← Gọi cả 2 nhánh
                   similarity.py
                   gọi get_connection()
```

> [!TIP]
> **Quy tắc chung trong dự án này:**
> - Cần hằng số → `from backend.config import TÊN_BIẾN`
> - Cần giao tiếp DB → `from backend.database import get_connection` hoặc `execute_query`
> - Không file nào tự kết nối DB trực tiếp bằng `psycopg2.connect(...)` ngoại trừ `database.py`
