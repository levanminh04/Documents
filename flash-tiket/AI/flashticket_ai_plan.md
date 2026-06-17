# 🧠 Flash Ticket System — AI Integration Plan (Đồ Án Tốt Nghiệp)

> **Chiến lược:** Không build hệ thống mới. Tách core-service → microservices + tích hợp AI sâu vào nghiệp vụ ticketing hiện có. AI phải giải quyết vấn đề THẬT, không phải demo toy.

---

## Tổng quan: 8 AI Modules

```
┌────────────────────────────────────────────────────────────┐
│                    FLASH TICKET + AI                        │
│                                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              AI SERVICE (Python/FastAPI)              │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │   │
│  │  │Recommend │ │Smart     │ │Dynamic   │ │Fraud   │ │   │
│  │  │Engine    │ │Search    │ │Pricing   │ │Detect  │ │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └────────┘ │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │   │
│  │  │Content   │ │Forecast  │ │Sentiment │ │Image   │ │   │
│  │  │Generate  │ │& Insight │ │Analysis  │ │Analysis│ │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └────────┘ │   │
│  └─────────────────────────────────────────────────────┘   │
│                         ▲                                   │
│            RabbitMQ + REST API                              │
│                         ▼                                   │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌─────────┐  │
│  │Event   │ │Booking │ │Payment │ │User    │ │Discovery│  │
│  │Service │ │Service │ │Service │ │Service │ │Service  │  │
│  └────────┘ └────────┘ └────────┘ └────────┘ └─────────┘  │
└────────────────────────────────────────────────────────────┘
```

---

## Module 1: AI Recommendation Engine ⭐⭐⭐⭐⭐
> **Tác động:** Buyer mở app → thấy events "dành cho mình" thay vì danh sách generic

### Vấn đề hiện tại
Homepage hiển thị `Featured events` (6 events mới nhất) → giống nhau cho TẤT CẢ users. Không personalization.

### Giải pháp: Hybrid Recommendation

**A. Collaborative Filtering (dựa trên hành vi tập thể)**
```
Ma trận User × Event:
         Event1  Event2  Event3  Event4
UserA    [  5      0      3      0  ]    ← 5=mua, 3=xem, 0=chưa
UserB    [  5      4      0      0  ]
UserC    [  0      0      4      5  ]

→ ALS (Alternating Least Squares): Phân rã ma trận
→ "UserA chưa thấy Event2, nhưng UserB giống UserA và thích Event2"
→ Recommend Event2 cho UserA
```
- **Data sources:** Purchase history, event views, wishlist, follows
- **Library:** Surprise (Python) hoặc implicit (Python)
- **Cold start:** User mới → dùng Content-Based

**B. Content-Based Filtering (dựa trên nội dung event)**
```
Mỗi event → text embedding (title + description + tags + category)
  → Google text-embedding-004 (768D) — đã có trong discovery-service
  → Lưu PGVector

User profile vector = trung bình embedding các events đã mua/xem

Recommend = cosine similarity (user_vector, all_event_vectors)
```

**C. Hybrid Ranking**
```python
final_score = (
    α * collaborative_score +    # 0.4 — "người giống bạn thích"
    β * content_score +           # 0.3 — "nội dung giống bạn thích" 
    γ * popularity_score +        # 0.1 — trending, view count
    δ * recency_score +           # 0.1 — events mới hơn ưu tiên
    ε * geo_score                 # 0.1 — events gần bạn hơn ưu tiên
)
```

### Tích hợp vào hệ thống
- **API:** `GET /api/recommendations?userId={id}&limit=10`
- **Homepage:** Thay "Featured Events" bằng "Gợi ý cho bạn"
- **Event Detail:** Section "Bạn có thể thích" (similar events)
- **Post-purchase:** "Người mua vé này cũng mua..."
- **Evaluation:** NDCG@10, Precision@K, A/B test

---

## Module 2: AI Smart Search ⭐⭐⭐⭐⭐
> **Tác động:** Buyer gõ "concert rock cuối tuần ở Sài Gòn" → AI hiểu ý nghĩa, không cần keyword chính xác

### Vấn đề hiện tại
Search hiện tại dùng SQL `LIKE '%keyword%'` → không hiểu semantic. Gõ "show nhạc sôi động" → không ra kết quả vì DB không có từ "sôi động."

### Giải pháp: Semantic Search + Hybrid

```
User query: "concert rock cuối tuần ở Sài Gòn"
    │
    ├─ (1) Text Embedding (768D) → PGVector cosine search
    │      → Tìm events có description gần nghĩa
    │
    ├─ (2) NLP Intent Parsing (Gemini API)
    │      → Extract: {genre: "rock", type: "concert", 
    │                   time: "weekend", city: "HCMC"}
    │      → SQL filter: WHERE category='concert' 
    │                     AND city='HCMC' 
    │                     AND day_of_week IN (6,7)
    │
    └─ (3) Hybrid Merge: Combine semantic + structured results
           → Re-rank by relevance score
```

### Tích hợp
- **Search bar** trên homepage + header
- **Auto-suggest:** Gõ vài ký tự → AI suggest hoàn chỉnh
- **"Không tìm thấy?"** → Chuyển sang chatbot AI Discovery (đã có)

---

## Module 3: AI Dynamic Pricing ⭐⭐⭐⭐
> **Tác động:** Organizer set min/max price → AI tự tối ưu giá theo demand realtime

### Logic

```
Input Features:
  - sales_velocity: Bao nhiêu vé bán trong 1h gần nhất
  - time_to_event: Còn bao lâu đến ngày diễn
  - inventory_ratio: % vé còn lại
  - view_count_recent: Lượt xem event gần đây
  - wishlist_count: Bao nhiêu người thêm yêu thích
  - day_of_week: Ngày trong tuần
  - competitor_events: Có event cạnh tranh cùng ngày không

Model: XGBoost Regressor
  → Output: optimal_price_multiplier (0.8 - 1.5)
  → final_price = base_price × multiplier
  → Bounded by: organizer_min_price ≤ final_price ≤ organizer_max_price

Transparency cho buyer:
  "Giá hiện tại: 800.000đ (Nhu cầu cao — chỉ còn 15% vé)"
```

### Rules
- Organizer **bật/tắt** dynamic pricing per ticket type
- Organizer set **floor price** và **ceiling price**
- Price chỉ thay đổi **mỗi 1h** (không real-time quá → gây hoang mang)
- Log mọi price change → audit trail

---

## Module 4: AI Fraud Detection ⭐⭐⭐⭐
> **Tác động:** Phát hiện bot mua vé, scalper, giao dịch bất thường

### Features cho ML Model

```python
transaction_features = {
    # Velocity features
    "orders_per_hour_by_ip": 15,        # 1 IP đặt 15 đơn/giờ → suspicious
    "orders_per_hour_by_user": 3,       # 1 user đặt 3 đơn/giờ
    
    # Pattern features  
    "is_new_account": True,             # Account tạo < 24h
    "minutes_since_registration": 5,    # Đăng ký 5 phút trước rồi mua ngay
    "payment_failure_rate": 0.8,        # 80% payment fail → thử thẻ ăn cắp
    
    # Behavioral features
    "time_on_page_seconds": 2,          # Chỉ ở trang 2 giây → bot
    "unique_events_in_cart": 1,         # Chỉ mua 1 event → scalper target
    "max_quantity_requested": 10,       # Luôn mua max → scalper
    
    # Device features
    "is_known_vpn": True,
    "device_fingerprint_seen_count": 50 # 1 device tạo 50 accounts
}

Model: Isolation Forest (unsupervised — không cần labeled data!)
  → anomaly_score > 0.7 → BLOCK booking
  → anomaly_score > 0.5 → Yêu cầu CAPTCHA / xác minh SĐT
  → anomaly_score < 0.5 → Allow
```

### Tích hợp
- Chạy TRƯỚC Redis lock (Guard 0.5 — trước cả spam lock)
- Không block hoàn toàn: score cao → thêm CAPTCHA, không reject ngay
- Admin dashboard: danh sách flagged transactions, override thủ công

---

## Module 5: AI Content Generation ⭐⭐⭐⭐
> **Tác động:** Organizer tạo event nhanh 10x — AI viết mô tả, suggest tags, SEO

### Features

**A. Auto-Generate Event Description**
```
Organizer input: {
    title: "Rock Storm 2026",
    category: "Concert", 
    venue: "Sân vận động Mỹ Đình",
    date: "2026-12-31",
    artists: "Sơn Tùng, Đen Vâu, Binz"
}

→ Gemini API generate:
"🎸 Rock Storm 2026 - Đại nhạc hội đón năm mới hoành tráng nhất!
Hòa mình vào đêm nhạc đỉnh cao với sự góp mặt của Sơn Tùng M-TP, 
Đen Vâu và Binz tại Sân vận động Mỹ Đình. Countdown 2027 cùng 
hàng vạn fan trong không khí bùng nổ..."
```

**B. Auto-Suggest Tags & Categories**
```
Từ title + description → AI suggest:
  tags: ["rock", "countdown", "new-year", "concert", "outdoor"]
  categories: ["Concert", "Festival"]
  mood: ["energetic", "festive"]
```

**C. Auto-Generate SEO**
```
meta_title: "Rock Storm 2026 | Đại Nhạc Hội Countdown Mỹ Đình"
meta_description: "Mua vé Rock Storm 2026 - Sơn Tùng, Đen Vâu, Binz..."
slug: "rock-storm-2026-my-dinh" (đã có auto-slug)
```

**D. Marketing Copy Generation**
```
Organizer chọn: "Tạo post Facebook" / "Tạo email marketing"
→ AI generate bài post/email phù hợp format + tone
```

---

## Module 6: AI Chatbot Nâng Cấp ⭐⭐⭐⭐
> **Đã có discovery-service. Nâng cấp thêm:**

### Nâng cấp 1: Personalized Recommendations trong chat
```
User: "Cuối tuần này có gì hay không?"
Bot: (Query recommendation engine + lọc weekend)
  → "Dựa trên lịch sử mua vé Rock của bạn, có 2 sự kiện hay:
     1. Rock Storm - Sân Mỹ Đình - Thứ 7
     2. Underground Night - Hanoi Rock City - Chủ nhật
     Bạn muốn xem chi tiết event nào?"
```

### Nâng cấp 2: Booking Assistant
```
User: "Đặt 2 vé VIP Rock Storm cho tôi"
Bot: → Call BookingTool (đã có @Tool)
  → "Đã tạo đơn hàng #TB-20261215-123456:
     2x VIP Rock Storm = 2.400.000đ
     Đơn hàng hết hạn sau 15 phút.
     [Thanh toán ngay →]"
```

### Nâng cấp 3: Post-Purchase Support
```
User: "Vé của tôi đâu rồi?"
Bot: → Query TicketTool → Show QR + event info
User: "Tôi muốn chuyển vé cho bạn tôi"  
Bot: → Guide transfer flow + call TransferTool
```

---

## Module 7: AI Analytics & Forecasting ⭐⭐⭐
> **Tác động:** Organizer dashboard hiện AI insights thay vì chỉ raw numbers

### A. Demand Forecasting
```
Input: Historical sales data + event features
Model: Prophet (Facebook) hoặc XGBoost
Output: "Event này dự kiến bán được 3,200/5,000 vé (64%)"
  → Organizer quyết định: tăng marketing hay giảm giá?
```

### B. AI Insight Cards (trên dashboard)
```
- "🔥 Tốc độ bán vé hôm nay nhanh hơn 45% so với cùng kỳ tuần trước"
- "⚠️ Loại vé Regular đang bán chậm — có thể cân nhắc giảm giá hoặc đẩy marketing"
- "📈 Peak time mua vé: 20:00-22:00 — nên chạy ads vào khung giờ này"
- "🎯 85% buyer đến từ Facebook — nên tập trung marketing trên Facebook"
```

### C. User Segmentation (RFM)
```
Phân nhóm buyer tự động:
  Champions (mua nhiều, gần đây, giá trị cao) → VIP treatment
  Loyal (mua đều, tần suất cao) → Early access
  At Risk (từng mua nhiều, lâu không quay lại) → Win-back email
  New (mới mua lần đầu) → Onboarding + recommend
```

---

## Module 8: AI Image Analysis ⭐⭐⭐
> **Tác động:** Tự động kiểm tra chất lượng ảnh event + moderate content

### Features
```
Organizer upload banner → AI check:
  ✅ Resolution đủ lớn (>1200px width)
  ✅ Không chứa nội dung nhạy cảm
  ✅ Có text rõ ràng (event name visible)
  ⚠️ "Ảnh hơi tối, có thể không nổi bật trên feed"
  💡 "Gợi ý: Thêm logo organizer vào góc"
```

- Dùng Gemini Vision API (đã có Gemini key từ discovery-service)
- Auto-generate alt text cho SEO

---

## Tech Stack cho AI Service

| Component | Công nghệ | Lý do |
|---|---|---|
| **Framework** | Python FastAPI | ML ecosystem (scikit-learn, surprise, xgboost) |
| **LLM** | Gemini API | Đã có trong discovery-service, nhất quán |
| **Embeddings** | Google text-embedding-004 (768D) | Đã có, nhất quán |
| **Vector DB** | PGVector | Đã có, không thêm dependency |
| **ML Models** | scikit-learn, XGBoost, Surprise | Lightweight, không cần GPU |
| **Communication** | RabbitMQ + REST | Đã có infrastructure |
| **Model Serving** | FastAPI endpoints | Đơn giản, đủ cho đồ án |

---

## Mức độ ưu tiên cho đồ án

| Ưu tiên | Module | Lý do |
|---|---|---|
| 🔴 **PHẢI CÓ** | Recommendation Engine | Điểm AI chính, hội đồng sẽ hỏi kỹ |
| 🔴 **PHẢI CÓ** | Smart Search | Thay đổi trải nghiệm user rõ rệt nhất |
| 🔴 **PHẢI CÓ** | AI Chatbot (nâng cấp) | Đã có base, chỉ cần polish + tích hợp RecSys |
| 🟡 **NÊN CÓ** | Content Generation | Wow factor cao, implement nhanh (Gemini API call) |
| 🟡 **NÊN CÓ** | Fraud Detection | Ấn tượng kỹ thuật (ML unsupervised) |
| 🟢 **BONUS** | Dynamic Pricing | Nâng cao, nếu đủ thời gian |
| 🟢 **BONUS** | Analytics/Forecasting | Nếu kết hợp DW star schema |
| 🟢 **BONUS** | Image Analysis | Quick win (1 Gemini Vision API call) |
