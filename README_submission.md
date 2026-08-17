# BÁO CÁO BÀI NỘP LAB 17 - MULTI-MEMORY AGENT VỚI ZEP

**Họ và tên**: Nguyễn Đức Hưng  
**Mã học viên**: 2A202601936  
**Bài lab**: Day 17 - MultiMemory Agent với Zep Cloud V3  

---

## 1. Trả Lời 3 Câu Hỏi Bắt Buộc (Dưới 400 từ)

### Câu 1: Tầng bộ nhớ quan trọng nhất trong bộ test này?
**Long-term Memory (Declarative)** và **Episodic Memory** là 2 tầng quan trọng nhất:
- **Long-term**: Quyết định khả năng nhớ thông tin cross-session, open-loops và cập nhật preference mới (Recency). *Minh họa*: Case **E08** (đổi preference sang TypeScript cho project BLUEBIRD-42) và **E09** (phân lập dữ liệu User Isolation giữa Minh và Lan).
- **Episodic**: Truy xuất quỹ đạo kinh nghiệm quá khứ. *Minh họa*: Case **E04** (nhớ giải pháp fix async HTTP timeout bằng `ClientSession` và `concurrency=20`).

### Câu 2: Trade-off giữa Managed Zep Context Block vs Tự xây Redis + Qdrant
- **Managed Zep (Context Block / Graph Search)**:
  - *Ưu điểm*: Tự động trích xuất Facts, liên kết Knowledge Graph, hỗ trợ sẵn Recency Decay, Context Block sẵn sàng bơm thẳng vào Agent mà không cần tự code pipeline trích xuất.
  - *Nhược điểm*: Đổi lấy chi phí SaaS, bị giới hạn tùy biến thuật toán ranking graph nội bộ.
- **Tự xây với Redis (KV) + Qdrant (Vector DB)**:
  - *Ưu điểm*: Kiểm soát 100% dữ liệu, latency cực thấp, chi phí tự host cố định.
  - *Nhược điểm*: Phải tự làm toàn bộ khâu Temporal Reasoning (Recency), Deduplication, Entity Resolution và Graph Traversal thủ công.

### Câu 3: Guardrail chống Memory Poisoning (Prompt Injection qua Bộ nhớ)
- **Input Sanitization & Opt-in Gate**: Kiểm tra Consent Registry (`consent.json`) và tự động làm sạch PII (`minimize_pii`) trước khi ingest.
- **Scope & Isolation Enforcer**: Bắt buộc gắn namespace `user_id` khắt khe cho Long-term/Episodic để tránh Poisoning chéo giữa các user.
- **Human-in-the-loop Review**: Các thông tin có tác động lớn (High-impact preference update) phải qua bước duyệt trước khi ghi bền vững vào Durable Memory Graph.

---

## 2. Phân Tích Báo Cáo Benchmark (`reports/comparison.md`)

1. **Layer hit rate thấp nhất**: Nới lỏng budget có thể làm nhiễu Semantic nếu query quá chung; việc áp dụng `scope="episodes"` giúp bảo toàn mã literal code (`PAYMENT-RULE-3`, `CONN-POOL-FIRST`).
2. **Case retrieve nhiều token nhất**: Các case hỗn hợp (Mixed E07) và Episodic chứa nhiều văn bản transcript.
3. **Case mixed E07**: Cần phối hợp **Long-term** (`Python`) và **Semantic** (`Idempotency-Key`).
4. **Token Reduction vs Hit Rate**: No-memory baseline có token reduction rất cao nhưng Hit Rate bằng 0 cho các câu hỏi cần nhớ quá khứ, chứng tỏ reduction cao không có ý nghĩa nếu thiếu khả năng recall dữ liệu.

---

## 3. Ảnh Minh Chứng (Screenshots)

*Lưu các file ảnh vào thư mục `submission/` theo đúng tên bên dưới để tự động hiển thị:*

### 3.1. Long-term Memory Benchmark (E02, E03, E08 PASS)
![Long-term Benchmark](submission/long_term.png)

### 3.2. Episodic Memory Benchmark (E04, E05 PASS)
![Episodic Benchmark](submission/episodic.png)

### 3.3. Semantic Memory Benchmark (E06, E11 PASS)
![Semantic Benchmark](submission/semantic.png)

### 3.4. Privacy Drill - Right-to-be-forgotten (`src.forget`)
![Privacy Drill](submission/privacy.png)

### 3.5. Streamlit UI Demo (Điểm cộng UI +10đ)
![Streamlit UI Demo](submission/ui_demo.png)

