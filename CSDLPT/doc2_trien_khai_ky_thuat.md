# 🔧 Tài liệu 2: Triển khai Kỹ thuật Chi tiết

> **Yêu cầu trước:** Đọc xong `doc1_ly_thuyet_nen_tang.md`  
> **Mục tiêu:** Bước theo từng bước để có hệ thống chạy được cuối cùng

---

## Mục lục

1. [Kiến trúc hệ thống tổng quan](#1-kiến-trúc-hệ-thống)
2. [Cấu trúc thư mục dự án](#2-cấu-trúc-thư-mục)
3. [Cài đặt môi trường](#3-cài-đặt-môi-trường)
4. [Module trích xuất đặc trưng âm thanh](#4-trích-xuất-đặc-trưng-âm-thanh)
5. [Thiết kế Database với pgvector](#5-database-pgvector)
6. [Backend API (FastAPI)](#6-backend-api)
7. [Frontend đơn giản](#7-frontend)
8. [Chạy và kiểm thử hệ thống](#8-chạy-và-kiểm-thử)

---

## 1. Kiến trúc hệ thống

```
┌──────────────────────────────────────────────────────────────┐
│                     CLIENT (Trình duyệt)                     │
│              Upload file │ Search │ Browse results           │
└──────────────────────────┬───────────────────────────────────┘
                           │ HTTP (REST API)
┌──────────────────────────▼───────────────────────────────────┐
│                    BACKEND — FastAPI (Python)                 │
│  ┌────────────────────────────────────────────────────────┐  │
│  │             Tầng Trích xuất Đặc trưng                  │  │
│  │  ┌──────────────────┐    ┌───────────────────────────┐ │  │
│  │  │ TỰ IMPLEMENT     │    │ THƯ VIỆN (librosa)        │ │  │
│  │  │ • Energy         │    │ • MFCC (13 hệ số)         │ │  │
│  │  │ • ZCR            │    │ • Spectral Centroid        │ │  │
│  │  │ • Silence Ratio  │    │ • Spectral Bandwidth       │ │  │
│  │  └────────┬─────────┘    │ • Spectral Rolloff         │ │  │
│  │           │              │ • Chroma (12 hệ số)        │ │  │
│  │           └──────────────┴──────────────┐            │ │  │
│  │                                         ▼            │ │  │
│  │              FEATURE VECTOR (41 chiều)                │ │  │
│  └─────────────────────────────────────────┬────────────┘  │
│                                            │                 │
│  ┌─────────────────────────────────────────▼────────────┐   │
│  │                 Tầng Lưu trữ & Truy vấn              │   │
│  │            PostgreSQL 16 + pgvector extension        │   │
│  │   ┌──────────────┐      ┌─────────────────────────┐  │   │
│  │   │ audio_files  │      │ audio_features           │  │   │
│  │   │ (metadata)   │─────▶│ vector(41) + HNSW index  │  │   │
│  │   └──────────────┘      └─────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

**Luồng dữ liệu:**
1. User upload file `.mp3/.wav`
2. Backend đọc file, trích xuất 41 đặc trưng → Feature Vector
3. Lưu metadata vào `audio_files`, lưu vector vào `audio_features`
4. Khi search: trích xuất vector của bài query → tìm K bài gần nhất bằng cosine distance

---

## 2. Cấu trúc thư mục dự án

```
CSDLDPT/
│
├── backend/
│   ├── main.py                  # FastAPI app, định nghĩa routes
│   ├── feature_extractor.py     # Module trích xuất đặc trưng
│   ├── database.py              # Kết nối PostgreSQL
│   ├── models.py                # SQLAlchemy models
│   ├── schemas.py               # Pydantic schemas (request/response)
│   └── requirements.txt         # Danh sách thư viện
│
├── frontend/
│   └── index.html               # Giao diện web đơn giản
│
├── uploads/                     # Thư mục lưu file âm thanh upload
│
├── sql/
│   └── init.sql                 # Script tạo bảng và index
│
└── README.md
```

---

## 3. Cài đặt môi trường

### 3.1 Yêu cầu hệ thống

| Phần mềm | Phiên bản | Cài đặt |
|---|---|---|
| Python | 3.9+ | python.org |
| PostgreSQL | 14+ | postgresql.org |
| pgvector | 0.5+ | Xem bước 3.3 |

### 3.2 Tạo Virtual Environment và cài thư viện

```bash
# Vào thư mục dự án
cd d:\PTIT\kì 2 năm 4\Cơ sở dữ liệu đa phương tiện\CSDLDPT

# Tạo và kích hoạt virtual environment
python -m venv venv
venv\Scripts\activate      # Windows

# Cài thư viện
pip install fastapi uvicorn python-multipart
pip install librosa numpy scipy
pip install psycopg2-binary sqlalchemy pgvector
pip install python-dotenv
```

**File `requirements.txt`:**
```
fastapi==0.110.0
uvicorn==0.29.0
python-multipart==0.0.9
librosa==0.10.1
numpy==1.26.4
scipy==1.13.0
psycopg2-binary==2.9.9
sqlalchemy==2.0.30
pgvector==0.2.5
python-dotenv==1.0.1
```

### 3.3 Cài pgvector cho PostgreSQL

```sql
-- Chạy trong psql với quyền superuser
CREATE EXTENSION IF NOT EXISTS vector;

-- Kiểm tra đã cài chưa
SELECT * FROM pg_extension WHERE extname = 'vector';
```

> ⚠️ Nếu lỗi "extension vector does not exist", cần cài pgvector package riêng.
> Tải tại: https://github.com/pgvector/pgvector/releases

### 3.4 Tạo file `.env`

```env
DATABASE_URL=postgresql://postgres:your_password@localhost:5432/multimedia_db
UPLOAD_DIR=uploads
MAX_FILE_SIZE_MB=50
```

---

## 4. Trích xuất đặc trưng âm thanh

### 4.1 File `feature_extractor.py`

```python
"""
Module trích xuất đặc trưng âm thanh.
- Phần 1: Tự implement (Time Domain)
- Phần 2: Dùng librosa (Frequency Domain)
"""
import numpy as np
import librosa
from typing import List


# ============================================================
# PHẦN 1: TỰ IMPLEMENT — Đặc trưng miền thời gian
# Tham chiếu: Slide 10 - Công thức Energy và ZCR
# ============================================================

def compute_energy(samples: np.ndarray) -> float:
    """
    Tính năng lượng trung bình (Short-time Energy).
    Công thức: E = (1/N) * Σ x(n)²
    → Đo âm thanh to hay nhỏ
    """
    return float(np.mean(samples ** 2))


def compute_zcr(samples: np.ndarray) -> float:
    """
    Tính tỷ lệ đổi dấu (Zero-Crossing Rate).
    Công thức: ZCR = (số lần đổi dấu) / N
    → Cao: âm cao (treble), Thấp: âm trầm (bass)
    """
    N = len(samples)
    if N < 2:
        return 0.0
    crossings = 0
    for n in range(1, N):
        if (samples[n] >= 0 and samples[n-1] < 0) or \
           (samples[n] < 0  and samples[n-1] >= 0):
            crossings += 1
    return crossings / N


def compute_silence_ratio(samples: np.ndarray,
                          threshold: float = 0.01) -> float:
    """
    Tính tỷ lệ khoảng lặng.
    → Giọng nói: cao (nhiều khoảng ngừng)
    → Nhạc sôi động: thấp (liên tục)
    """
    silence_count = np.sum(np.abs(samples) < threshold)
    return float(silence_count / len(samples))


# ============================================================
# PHẦN 2: DÙNG LIBROSA — Đặc trưng miền tần số
# Tham chiếu: Slide 10 - MFCC, Spectral Features
# ============================================================

def compute_mfcc_features(y: np.ndarray, sr: int,
                           n_mfcc: int = 13) -> np.ndarray:
    """
    Tính MFCC (Mel-Frequency Cepstral Coefficients).
    → n_mfcc=13 hệ số, lấy mean + std cho cả bài = 26 số
    """
    mfccs = librosa.feature.mfcc(y=y, sr=sr, n_mfcc=n_mfcc)
    mfcc_mean = np.mean(mfccs, axis=1)   # shape: (13,)
    mfcc_std  = np.std(mfccs,  axis=1)   # shape: (13,)
    return np.concatenate([mfcc_mean, mfcc_std])  # shape: (26,)


def compute_spectral_features(y: np.ndarray, sr: int) -> np.ndarray:
    """
    Tính các đặc trưng phổ tần số:
    - Spectral Centroid: trọng tâm phổ (Hz)
    - Spectral Bandwidth: độ rộng phổ (Hz)
    - Spectral Rolloff: tần số 85% năng lượng (Hz)
    """
    centroid   = librosa.feature.spectral_centroid(y=y, sr=sr)
    bandwidth  = librosa.feature.spectral_bandwidth(y=y, sr=sr)
    rolloff    = librosa.feature.spectral_rolloff(y=y, sr=sr)

    return np.array([
        float(np.mean(centroid)),
        float(np.mean(bandwidth)),
        float(np.mean(rolloff)),
    ])


def compute_chroma_features(y: np.ndarray, sr: int) -> np.ndarray:
    """
    Tính Chroma features: phân bố năng lượng trên 12 nốt nhạc
    (C, C#, D, D#, E, F, F#, G, G#, A, A#, B)
    → Phân biệt điệu trưởng / thứ, hòa âm
    """
    chroma = librosa.feature.chroma_stft(y=y, sr=sr)
    return np.mean(chroma, axis=1)  # shape: (12,)


# ============================================================
# HÀM TỔNG HỢP: Kết hợp tất cả thành 1 Feature Vector
# ============================================================

def extract_audio_features(file_path: str) -> dict:
    """
    Trích xuất toàn bộ đặc trưng của 1 file âm thanh.
    Trả về dict với 'vector' (41 chiều) và 'metadata'.
    
    Cấu trúc vector (41 chiều):
      [0]    energy          — tự code
      [1]    zcr             — tự code
      [2]    silence_ratio   — tự code
      [3-15] mfcc_mean (13)  — librosa
      [16-28]mfcc_std (13)   — librosa
      [29]   centroid        — librosa
      [30]   bandwidth       — librosa
      [31]   rolloff         — librosa
      [32-43]chroma (12)     — librosa  (tổng = 44, idx thực tế)
    Tổng: 3 + 13 + 13 + 3 + 12 = 44 chiều
    """
    # Load audio (librosa mặc định: 22050 Hz, mono)
    y, sr = librosa.load(file_path, sr=22050, mono=True)
    
    # Tính duration
    duration = librosa.get_duration(y=y, sr=sr)
    
    # === TỰ IMPLEMENT ===
    energy        = compute_energy(y)
    zcr           = compute_zcr(y)
    silence_ratio = compute_silence_ratio(y)
    
    # === DÙNG LIBROSA ===
    mfcc_features     = compute_mfcc_features(y, sr)      # (26,)
    spectral_features = compute_spectral_features(y, sr)  # (3,)
    chroma_features   = compute_chroma_features(y, sr)    # (12,)
    
    # Ghép thành 1 vector: [3] + [26] + [3] + [12] = 44 chiều
    feature_vector = np.concatenate([
        [energy, zcr, silence_ratio],  # (3,)
        mfcc_features,                  # (26,)
        spectral_features,              # (3,)
        chroma_features,                # (12,)
    ])
    
    # Chuẩn hoá vector (L2-norm) để khoảng cách cosine hoạt động đúng
    norm = np.linalg.norm(feature_vector)
    if norm > 0:
        feature_vector = feature_vector / norm
    
    return {
        "vector": feature_vector.tolist(),  # list 44 số float
        "duration": round(duration, 2),
        "sample_rate": sr,
        "features_breakdown": {
            "energy": energy,
            "zcr": zcr,
            "silence_ratio": silence_ratio,
            "mfcc_mean": mfcc_features[:13].tolist(),
            "spectral_centroid": float(spectral_features[0]),
        }
    }
```

---

## 5. Database pgvector

### 5.1 File `sql/init.sql` — Khởi tạo database

```sql
-- Bật extension pgvector
CREATE EXTENSION IF NOT EXISTS vector;

-- Bảng lưu thông tin file âm thanh
CREATE TABLE IF NOT EXISTS audio_files (
    id          SERIAL PRIMARY KEY,
    filename    VARCHAR(255) NOT NULL,
    filepath    TEXT         NOT NULL,
    duration    FLOAT,                     -- Thời lượng (giây)
    sample_rate INTEGER,                   -- Sample rate (Hz)
    file_size   BIGINT,                    -- Kích thước file (bytes)
    upload_at   TIMESTAMP DEFAULT NOW()
);

-- Bảng lưu vector đặc trưng
-- vector(44) = kiểu dữ liệu của pgvector, 44 chiều
CREATE TABLE IF NOT EXISTS audio_features (
    id           SERIAL PRIMARY KEY,
    audio_id     INTEGER REFERENCES audio_files(id) ON DELETE CASCADE,
    feature_vec  vector(44) NOT NULL,      -- Feature vector 44 chiều
    energy       FLOAT,                    -- Lưu riêng để filter nhanh
    zcr          FLOAT,
    silence_ratio FLOAT,
    created_at   TIMESTAMP DEFAULT NOW()
);

-- Tạo chỉ mục HNSW cho tìm kiếm vector nhanh
-- Tham chiếu: Slide 8 - Truy vấn không gian vector
--
-- HNSW = Hierarchical Navigable Small World
-- Thay vì quét tuần tự O(n), HNSW xây dựng đồ thị phân cấp
-- → Tìm kiếm gần đúng (ANN) với độ phức tạp O(log n)
--
-- So sánh:
--   Sequential scan: Phải so sánh với TẤT CẢ 1 triệu bài → chậm
--   HNSW index:      Chỉ so sánh với ~100 bài đại diện → nhanh 1000x
CREATE INDEX idx_audio_features_hnsw
    ON audio_features
    USING hnsw (feature_vec vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

-- Giải thích tham số HNSW:
--   m = 16:               Mỗi node kết nối với 16 node hàng xóm
--   ef_construction = 64: Độ chính xác khi xây dựng index (cao hơn = chậm hơn build nhưng tìm chính xác hơn)

-- View tiện lợi để xem kết quả query
CREATE OR REPLACE VIEW audio_search_view AS
SELECT
    af.id        AS feature_id,
    a.filename,
    a.filepath,
    a.duration,
    af.energy,
    af.zcr,
    af.silence_ratio,
    af.feature_vec
FROM audio_features af
JOIN audio_files   a ON a.id = af.audio_id;
```

### 5.2 Ví dụ truy vấn tìm kiếm tương tự

```sql
-- Tìm 5 bài nhạc giống bài có audio_id = 3 nhất
-- <=> là toán tử cosine distance của pgvector
SELECT
    a.filename,
    a.duration,
    1 - (af.feature_vec <=> target.feature_vec) AS similarity_score
FROM audio_features af
JOIN audio_files    a ON a.id = af.audio_id
CROSS JOIN (
    SELECT feature_vec FROM audio_features WHERE audio_id = 3
) AS target
WHERE af.audio_id != 3
ORDER BY af.feature_vec <=> target.feature_vec  -- nhỏ hơn = giống hơn
LIMIT 5;

-- Kết quả mẫu:
-- filename          | duration | similarity_score
-- ----------------------------|---------|------------------
-- bai_tuong_tu_1.mp3 | 214.5  | 0.9823            ← rất giống!
-- bai_tuong_tu_2.mp3 | 189.2  | 0.9541
-- bai_khac_mot_chut.mp3 | 201.0 | 0.8123
-- bai_khac_nhieu.mp3 | 175.3  | 0.6234
-- ...
```

**Giải thích toán tử `<=>`:**
- `<=>` = cosine distance = `1 - cosine_similarity`
- `<=>` nhỏ = giống nhau
- `<=>` = 0.0 : hoàn toàn giống
- `<=>` = 1.0 : khác hoàn toàn
- `similarity_score = 1 - <=>` giúp đổi sang % (0% → 100%)

### 5.3 Tại sao HNSW thắng Sequential Scan?

```
Kịch bản: Database có 1,000,000 bài nhạc, vector 44 chiều

Sequential Scan:
  → So sánh query vector với 1,000,000 vectors
  → Mỗi phép so sánh: 44 phép nhân + 44 phép cộng
  → Tổng: ~88,000,000 phép tính
  → Thời gian: ~5-10 giây ❌

HNSW Index:
  → Duyệt qua đồ thị phân cấp, chỉ kiểm tra ~500-2000 nodes
  → Thời gian: ~5-50 mili giây ✅ (nhanh hơn 100-1000x!)
```

---

## 6. Backend API

### 6.1 File `database.py`

```python
import os
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, DeclarativeBase
from dotenv import load_dotenv

load_dotenv()

DATABASE_URL = os.getenv("DATABASE_URL",
    "postgresql://postgres:password@localhost:5432/multimedia_db")

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(bind=engine)

class Base(DeclarativeBase):
    pass

def get_db():
    """Dependency injection cho FastAPI"""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### 6.2 File `models.py`

```python
from sqlalchemy import Column, Integer, String, Float, BigInteger, DateTime
from sqlalchemy.sql import func
from pgvector.sqlalchemy import Vector
from database import Base

class AudioFile(Base):
    __tablename__ = "audio_files"
    id          = Column(Integer, primary_key=True, index=True)
    filename    = Column(String(255), nullable=False)
    filepath    = Column(String, nullable=False)
    duration    = Column(Float)
    sample_rate = Column(Integer)
    file_size   = Column(BigInteger)
    upload_at   = Column(DateTime, server_default=func.now())

class AudioFeature(Base):
    __tablename__ = "audio_features"
    id            = Column(Integer, primary_key=True, index=True)
    audio_id      = Column(Integer, nullable=False)
    feature_vec   = Column(Vector(44))          # pgvector type
    energy        = Column(Float)
    zcr           = Column(Float)
    silence_ratio = Column(Float)
```

### 6.3 File `main.py` — FastAPI Routes

```python
import os
import shutil
from fastapi import FastAPI, File, UploadFile, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from fastapi.staticfiles import StaticFiles
from sqlalchemy.orm import Session
from sqlalchemy import text
from typing import List

from database import get_db, engine
from models import Base, AudioFile, AudioFeature
from feature_extractor import extract_audio_features

# Khởi tạo
Base.metadata.create_all(bind=engine)
app = FastAPI(title="Multimedia Database System", version="1.0")

app.add_middleware(CORSMiddleware,
    allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])

UPLOAD_DIR = "uploads"
os.makedirs(UPLOAD_DIR, exist_ok=True)


# ── Route 1: Upload và trích xuất đặc trưng ─────────────────

@app.post("/api/upload")
async def upload_audio(
    file: UploadFile = File(...),
    db:   Session    = Depends(get_db)
):
    """Upload file âm thanh và tự động trích xuất đặc trưng"""
    
    # Kiểm tra định dạng file
    allowed_extensions = {".mp3", ".wav", ".flac", ".ogg", ".m4a"}
    ext = os.path.splitext(file.filename)[1].lower()
    if ext not in allowed_extensions:
        raise HTTPException(400, f"Chỉ chấp nhận: {allowed_extensions}")
    
    # Lưu file
    filepath = os.path.join(UPLOAD_DIR, file.filename)
    with open(filepath, "wb") as buffer:
        shutil.copyfileobj(file.file, buffer)
    
    file_size = os.path.getsize(filepath)
    
    # Trích xuất đặc trưng
    try:
        features = extract_audio_features(filepath)
    except Exception as e:
        os.remove(filepath)
        raise HTTPException(500, f"Lỗi trích xuất đặc trưng: {str(e)}")
    
    # Lưu metadata vào DB
    audio_record = AudioFile(
        filename    = file.filename,
        filepath    = filepath,
        duration    = features["duration"],
        sample_rate = features["sample_rate"],
        file_size   = file_size,
    )
    db.add(audio_record)
    db.flush()  # Lấy ID trước khi commit
    
    # Lưu vector đặc trưng
    feature_record = AudioFeature(
        audio_id      = audio_record.id,
        feature_vec   = features["vector"],
        energy        = features["features_breakdown"]["energy"],
        zcr           = features["features_breakdown"]["zcr"],
        silence_ratio = features["features_breakdown"]["silence_ratio"],
    )
    db.add(feature_record)
    db.commit()
    
    return {
        "message": "Upload thành công",
        "audio_id": audio_record.id,
        "filename": file.filename,
        "duration": features["duration"],
        "features": features["features_breakdown"]
    }


# ── Route 2: Tìm kiếm âm thanh tương tự ────────────────────

@app.get("/api/search/{audio_id}")
def search_similar(
    audio_id: int,
    top_k:    int     = 5,
    db:       Session = Depends(get_db)
):
    """
    Tìm K bài âm thanh giống nhất với bài có audio_id.
    Sử dụng cosine distance trên HNSW index.
    """
    # Lấy vector của bài query
    query_feature = db.query(AudioFeature)\
        .filter(AudioFeature.audio_id == audio_id).first()
    
    if not query_feature:
        raise HTTPException(404, "Không tìm thấy bài nhạc")
    
    # Truy vấn tìm kiếm vector tương tự
    # <=> = cosine distance (pgvector operator)
    sql = text("""
        SELECT
            a.id          AS audio_id,
            a.filename,
            a.duration,
            af.energy,
            af.zcr,
            af.silence_ratio,
            1 - (af.feature_vec <=> :query_vec ::vector) AS similarity
        FROM audio_features af
        JOIN audio_files    a ON a.id = af.audio_id
        WHERE af.audio_id != :exclude_id
        ORDER BY af.feature_vec <=> :query_vec ::vector
        LIMIT :top_k
    """)
    
    results = db.execute(sql, {
        "query_vec":  str(query_feature.feature_vec),
        "exclude_id": audio_id,
        "top_k":      top_k
    }).fetchall()
    
    return {
        "query_audio_id": audio_id,
        "results": [
            {
                "audio_id":     r.audio_id,
                "filename":     r.filename,
                "duration":     r.duration,
                "similarity":   round(float(r.similarity) * 100, 2),  # %
                "energy":       r.energy,
                "zcr":          r.zcr,
                "silence_ratio":r.silence_ratio,
            }
            for r in results
        ]
    }


# ── Route 3: Danh sách tất cả file ──────────────────────────

@app.get("/api/files")
def list_files(db: Session = Depends(get_db)):
    """Lấy danh sách tất cả file âm thanh đã upload"""
    files = db.query(AudioFile).order_by(AudioFile.upload_at.desc()).all()
    return [
        {
            "id":        f.id,
            "filename":  f.filename,
            "duration":  f.duration,
            "file_size": f.file_size,
            "upload_at": f.upload_at.isoformat() if f.upload_at else None,
        }
        for f in files
    ]


# ── Route 4: Thống kê database ──────────────────────────────

@app.get("/api/stats")
def get_stats(db: Session = Depends(get_db)):
    """Thống kê tổng quan về dữ liệu trong hệ thống"""
    total_files = db.query(AudioFile).count()
    result = db.execute(text("""
        SELECT
            COUNT(*)                       AS total,
            ROUND(AVG(energy)::numeric, 4) AS avg_energy,
            ROUND(AVG(zcr)::numeric, 4)    AS avg_zcr,
            ROUND(AVG(silence_ratio)::numeric, 4) AS avg_silence
        FROM audio_features
    """)).fetchone()
    
    return {
        "total_files":   total_files,
        "avg_energy":    float(result.avg_energy or 0),
        "avg_zcr":       float(result.avg_zcr or 0),
        "avg_silence":   float(result.avg_silence or 0),
    }
```

---

## 7. Frontend

### File `frontend/index.html`

```html
<!DOCTYPE html>
<html lang="vi">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Hệ thống CSDL Đa phương tiện</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { font-family: Arial, sans-serif; background: #f0f2f5; padding: 20px; }
    .container { max-width: 800px; margin: 0 auto; }
    h1 { color: #333; text-align: center; margin-bottom: 24px; }
    .card { background: white; border-radius: 8px; padding: 20px; margin-bottom: 16px;
            box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
    input[type=file], input[type=number] { padding: 8px; margin: 8px 0; }
    button { background: #4f46e5; color: white; border: none; padding: 10px 20px;
             border-radius: 6px; cursor: pointer; font-size: 14px; }
    button:hover { background: #4338ca; }
    .result { background: #f8f9fa; border-radius: 6px; padding: 12px;
              margin-top: 12px; font-size: 13px; }
    .similarity-bar { background: #e0e0e0; border-radius: 4px; height: 8px; margin: 4px 0; }
    .similarity-fill { background: #4f46e5; height: 8px; border-radius: 4px; }
  </style>
</head>
<body>
  <div class="container">
    <h1>🎵 Hệ thống CSDL Đa phương tiện</h1>

    <!-- Upload -->
    <div class="card">
      <h2>📁 Upload File Âm thanh</h2>
      <input type="file" id="audioFile" accept=".mp3,.wav,.flac,.ogg">
      <br>
      <button onclick="uploadFile()">Upload & Trích xuất đặc trưng</button>
      <div id="uploadResult" class="result" style="display:none"></div>
    </div>

    <!-- Search -->
    <div class="card">
      <h2>🔍 Tìm kiếm tương tự</h2>
      <label>Audio ID: <input type="number" id="searchId" value="1" min="1"></label>
      <label>Top K: <input type="number" id="topK" value="5" min="1" max="20"></label>
      <br>
      <button onclick="searchSimilar()">Tìm bài tương tự</button>
      <div id="searchResult" class="result" style="display:none"></div>
    </div>

    <!-- File list -->
    <div class="card">
      <h2>📋 Danh sách Files</h2>
      <button onclick="loadFiles()">Tải danh sách</button>
      <div id="fileList" class="result" style="display:none"></div>
    </div>
  </div>

  <script>
    const API = "http://localhost:8000/api";

    async function uploadFile() {
      const file = document.getElementById("audioFile").files[0];
      if (!file) return alert("Chọn file trước!");
      const form = new FormData();
      form.append("file", file);
      const res  = await fetch(`${API}/upload`, { method: "POST", body: form });
      const data = await res.json();
      const el   = document.getElementById("uploadResult");
      el.style.display = "block";
      el.innerHTML = res.ok
        ? `✅ Upload thành công! Audio ID: <b>${data.audio_id}</b><br>
           Duration: ${data.duration}s | Energy: ${data.features.energy.toFixed(4)} | ZCR: ${data.features.zcr.toFixed(4)}`
        : `❌ Lỗi: ${data.detail}`;
    }

    async function searchSimilar() {
      const id   = document.getElementById("searchId").value;
      const topK = document.getElementById("topK").value;
      const res  = await fetch(`${API}/search/${id}?top_k=${topK}`);
      const data = await res.json();
      const el   = document.getElementById("searchResult");
      el.style.display = "block";
      if (!res.ok) { el.innerHTML = `❌ Lỗi: ${data.detail}`; return; }
      el.innerHTML = data.results.map(r =>
        `<div style="margin-bottom:10px">
           <b>${r.filename}</b> — ID: ${r.audio_id}<br>
           Độ tương tự: <b>${r.similarity}%</b>
           <div class="similarity-bar">
             <div class="similarity-fill" style="width:${r.similarity}%"></div>
           </div>
           Duration: ${r.duration}s | Energy: ${r.energy?.toFixed(4)}
         </div>`
      ).join("") || "Không tìm thấy kết quả";
    }

    async function loadFiles() {
      const res  = await fetch(`${API}/files`);
      const data = await res.json();
      const el   = document.getElementById("fileList");
      el.style.display = "block";
      el.innerHTML = data.map(f =>
        `<div>ID: <b>${f.id}</b> | ${f.filename} | ${f.duration?.toFixed(1)}s</div>`
      ).join("") || "Chưa có file nào";
    }
  </script>
</body>
</html>
```

---

## 8. Chạy và kiểm thử

### 8.1 Khởi tạo database

```bash
# Tạo database
psql -U postgres -c "CREATE DATABASE multimedia_db;"

# Chạy script khởi tạo
psql -U postgres -d multimedia_db -f sql/init.sql
```

### 8.2 Chạy backend

```bash
# Kích hoạt venv
venv\Scripts\activate

# Chạy server (hot-reload)
uvicorn backend.main:app --reload --port 8000
```

Mở trình duyệt: `http://localhost:8000/docs` → Swagger UI tự động của FastAPI

### 8.3 Kiểm thử bằng script

```python
# test_system.py
import requests
import os

API = "http://localhost:8000/api"

# Test 1: Upload
print("=== Test 1: Upload ===")
with open("test_audio.mp3", "rb") as f:
    res = requests.post(f"{API}/upload",
                        files={"file": ("test_audio.mp3", f, "audio/mpeg")})
print(res.json())

# Test 2: Tìm kiếm
print("\n=== Test 2: Tìm kiếm tương tự ===")
res = requests.get(f"{API}/search/1?top_k=3")
for r in res.json()["results"]:
    print(f"  {r['filename']}: {r['similarity']}% tương tự")

# Test 3: Thống kê
print("\n=== Test 3: Thống kê ===")
res = requests.get(f"{API}/stats")
print(res.json())
```

### 8.4 Checklist bảo vệ đồ án

| Câu hỏi giảng viên có thể hỏi | Câu trả lời nên chuẩn bị |
|---|---|
| "Energy tính thế nào?" | Trình bày công thức `E = Σx²/N`, chỉ code tự implement |
| "Tại sao dùng MFCC?" | Vì mô phỏng cách tai người nghe (thang Mel), phù hợp nhất cho âm thanh |
| "pgvector là gì?" | Extension của PostgreSQL cho phép lưu và tìm kiếm vector nhiều chiều |
| "HNSW hoạt động ra sao?" | Xây dựng đồ thị phân cấp, tìm kiếm ANN thay vì sequential scan O(n) |
| "Toán tử `<=>` làm gì?" | Tính cosine distance giữa 2 vector, nhỏ hơn = giống hơn |
| "Tại sao không tự implement FFT?" | Để đảm bảo chính xác toán học, tránh sai số; librosa là thư viện chuẩn công nghiệp |

---

*Phiên bản: 1.0 — Ngày tạo: 2026-02-25*
