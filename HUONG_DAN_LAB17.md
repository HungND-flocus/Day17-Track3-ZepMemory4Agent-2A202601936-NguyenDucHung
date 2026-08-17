# HƯỚNG DẪN THỰC HIỆN & BÀI HỌC DỰ ÁN LAB 17: MULTI-MEMORY AGENT VỚI ZEP CLOUD V3

> **Dự án**: Day 17 - Multi-Memory Agent với Zep Cloud V3  
> **Tác giả**: VinUni Codelab / VinAI Lab  
> **Mục tiêu chính**: Hoàn thiện 4 tầng bộ nhớ cho AI Agent, vượt qua bộ benchmark 11 test cases (E01-E11) đạt hit rate $\ge 80\%$ ($\ge 9/11$ PASS), đáp ứng tiêu chuẩn Privacy-by-Design và hiểu rõ cơ chế Context Engineering.

---

## I. TỔNG QUAN DỰ ÁN & KIẾN TRÚC THIẾT KẾ

### 1. Bối cảnh & Bài toán
Trong các hệ thống AI Agent thực tế, nếu chỉ phụ thuộc vào Prompt Context mặc định hoặc Short-term message history, Agent sẽ gặp các vấn đề lớn:
- **Trôi bối cảnh (Context Drift / Loss)** khi hội thoại kéo dài qua nhiều turn.
- **Không nhớ được thông tin cross-session** (sở thích, quyết định cũ của User qua nhiều phiên làm việc).
- **Không rút ra được bài học kinh nghiệm (Episodic Memory)** từ các thử nghiệm thất bại/thành công trước đó.
- **Không truy xuất được tri thức domain chung (Semantic Memory)** độc lập với từng user.
- **Vượt quá giới hạn Token Budget & Chi phí API cao**.

Lab 17 giải quyết triệt để vấn đề này bằng cách kết hợp **Zep Cloud V3 SDK** (Managed Memory Backend) cùng hệ thống Short-term Compaction local và các baseline Redis/Qdrant.

### 2. Kiến trúc 4 Tầng Bộ Nhớ (4 Memory Layers)

```text
                     +---------------------------------------+
                     |  control_plane/*.md                   |
                     | persona / rules / schema / open-loops |
                     +-------------------+-------------------+
                                         |
JSON Sessions ---> LangGraph Router ---> retrieve(query)
                                         |
        +--------------------------------+--------------------------------+
        |                                |                                |
 1. Short-Term                   2. Zep User Graph               3. Zep Standalone Graph
 (Buffer / Summary / Sliding)    (Long-Term Facts & Episodes)    (Semantic / Domain KB)
        |                                |                                |
        +--------------------------------+--------------------------------+
                                         |
                             Priority & Token Budget Trimming
                             (Short-term 10% | Long-term 4% | Episodic 3% | Semantic 3%)
                                         |
                                   Merged Context
                                         |
                               Ground-Truth Evaluator
```

1. **Short-Term / Working Memory (Local Sliding Window + Compaction)**:
   - Lưu trữ các turn thoại gần nhất ($K=4 \div 6$ turns).
   - Tự động rút gọn các turn cũ thành **State/Summary + Durable Notes** để giữ lại các mốc quan trọng (deadline, constraint) mà không làm nổ token.
2. **Long-Term / Declarative Memory (Zep User Graph - Context Block & Facts)**:
   - Lưu các thông tin bền vững: User Preferences, Decisions, Open Tasks/Loops giữa các phiên làm việc riêng biệt.
   - Xử lý xung đột thông tin theo nguyên tắc **Recency Wins** (Fact mới cập nhật ghi đè preference cũ nhưng vẫn duy trì lịch sử vết).
3. **Episodic Memory (Zep User Graph - Episodes Search)**:
   - Truy xuất lịch sử trải nghiệm/trajectory của User: *"Lần trước đã thử làm gì, thất bại ra sao, cách fix thành công là gì"*.
   - Lưu trữ nguyên vẹn Provenance (nguồn gốc) và bài học rút ra.
4. **Semantic Memory (Zep Standalone Graph - Domain Knowledge)**:
   - Tri thức domain/tài liệu chuẩn hóa dùng chung cho toàn bộ hệ thống (không thuộc về riêng một user nào).
   - Tìm kiếm theo ngữ nghĩa (Semantic Graph Search) để trích xuất các quy tắc hệ thống, playbook sự cố, retry rule.

---

## II. THẮNG ĐIỂM & TIÊU CHÍ ĐÁNH GIÁ (GRADING MATRIX)

| Khối | Điểm tối đa | Nguồn chấm / Điểm mốc |
| --- | ---: | --- |
| **11 Test Cases Auto (E01-E11)** | **56 điểm** | Đánh giá qua `reports/benchmark.json` (`--impl student`) |
| **Privacy Drill** | **6 điểm** | Chạy `forget.py` + `--verify-only` (Screenshot) |
| **Phân tích Benchmark (`comparison.md`)** | **6 điểm** | 4 câu trả lời so sánh trong `README_submission.md` |
| **Báo cáo `README_submission.md`** | **6 điểm** | Trả lời 3 câu hỏi chính (dưới 400 từ) |
| **Artefacts & Quy trình** | **6 điểm** | Repo đủ file, code không lỗi, 4 ảnh chụp minh chứng |
| **TRẦN NỀN CƠ BẢN** | **80 điểm** | **Cần $\ge 56/80$ đ & Hit Rate $\ge 80\%$ (9/11 PASS)** |
| **Golden Set (20/20 cases)** | **+10 điểm** | `reports/golden_benchmark.json` (60 phút cuối) |
| **Streamlit UI Demo / Report đẹp** | **+10 điểm** | `src/demo_ui.py` (Mini product UI) |
| **TỔNG ĐIỂM TỐI ĐA** | **100 điểm** | |

---

## III. CẤU TRÚC THƯ MỤC DỰ ÁN

```text
.
├── LAB.md                       # Hướng dẫn chi tiết & Thang điểm chính thức của Lab
├── README.md                    # Hướng dẫn nhanh dự án
├── HUONG_DAN_LAB17.md           # [File này] Hướng dẫn thực hành & tổng hợp bài học
├── docker-compose.yml           # Khởi tạo App container, Redis local, Qdrant local
├── Dockerfile & Makefile        # Cấu hình môi trường & lệnh tắt
├── requirements.txt             # Các thư viện Python (zep-cloud, redis, qdrant-client, v.v.)
├── control_plane/               # Các file quy tắc hệ thống (AGENTS, CONTEXT_LAYERS, SOUL, MEMORY)
├── data/
│   ├── sessions.json            # Tập dữ liệu chat giả lập + Ground-truth kỳ vọng (E01-E11)
│   ├── consent.json             # Danh sách consent GDPR của synthetic users
│   ├── knowledge.jsonl          # Tri thức domain dùng chung cho Semantic Graph
│   └── compiled_kb.jsonl        # Compiled knowledge base đã qua trích xuất/curate
├── src/
│   ├── memory_student.py        # ★ FILE CHÍNH HỌC VIÊN CẦN ĐIỀN CODE (4 TODOs)
│   ├── memory_reference.py      # Implementation tham khảo của Giảng viên
│   ├── short_term.py            # Local short-term memory (Compaction & Durable Notes)
│   ├── context_budget.py        # ContextBudgetManager phân bổ token theo tỉ lệ 10/4/3/3
│   ├── zep_common.py            # Zep Client wrapper, helper search & render
│   ├── seed.py                  # Ingest dữ liệu vào Zep Cloud
│   ├── evaluate.py              # Ground-truth evaluator chấm điểm retrieval
│   ├── compare_reports.py       # So sánh Memory-enabled vs No-memory baseline
│   ├── forget.py                # Privacy drill: xóa dữ liệu user (Right to be Forgotten)
│   └── demo_ui.py               # Streamlit Mini-product UI
└── reports/                     # Thư mục chứa báo cáo benchmark (.json & .md)
```

---

## IV. HƯỚNG DẪN THỰC HÀNH CHI TIẾT TỪNG BƯỚC (STEP-BY-STEP)

### Bước 1: Khởi động môi trường & Ingest dữ liệu (Phút 0 - 15)

1. **Cấu hình file môi trường `.env`**:
   Copy file `.env.example` thành `.env` và điền khóa `ZEP_API_KEY`:
   ```bash
   cp .env.example .env
   # Sửa .env -> ZEP_API_KEY=z_...
   ```

2. **Build và khởi động Docker services**:
   ```bash
   docker compose build
   docker compose up -d redis qdrant
   ```

3. **Chạy Smoke Test**:
   ```bash
   docker compose run --rm app python -m src.smoke
   ```
   *Yêu cầu*: Đầu ra báo `[OK]` cho Redis, Qdrant, `sessions.json` và `ZEP_API_KEY`.

4. **Seed dữ liệu vào Zep Cloud (Chỉ chạy 1 lần)**:
   ```bash
   docker compose run --rm app python -m src.seed
   ```
   *Lưu ý*: Lệnh này sẽ tạo Synthetic Users (`minh-lab17`, `lan-lab17`), khởi tạo các thread, ingest lịch sử chat và nạp Standalone Semantic Graph.

---

### Bước 2: Pha A - Khám phá Short-Term Memory & Compaction (Phút 15 - 30)

Chạy script demo short-term memory:
```bash
docker compose run --rm app python -m src.demo_short_term
```

**Thử nghiệm quan trọng**:
- Mở `src/demo_short_term.py`, thử giảm `max_recent_messages` từ `6` xuống `4`.
- Quan sát cơ chế Compaction trong `src/short_term.py`: Các tin nhắn cũ bị dồn lại thành **Summary**, nhưng các thông tin cốt lõi (như mốc deadline `REVIEW-DEADLINE-1600`) vẫn được trích xuất lưu giữ trong **Durable Notes**.
- *Đánh giá*: Đảm bảo hiểu lý do tại sao Buffer thuần túy sẽ làm nổ context, trong khi Sliding Window + Compaction giữ vững thông tin quan trọng.

---

### Bước 3: Pha B - Cài đặt TODO 1/4: Long-Term Memory Retrieval (`retrieve_long_term`)

Mở file `src/memory_student.py` và hoàn thiện hàm `retrieve_long_term`:

```python
def retrieve_long_term(self, user_id: str, thread_id: str, query: str) -> str:
    # 1. Gọi helper scaffolding có sẵn để chuẩn bị thread context
    prime_eval_thread(self.client, user_id, thread_id, query)
    
    # 2. Truy xuất Context Block từ Zep Cloud
    user_context = self.client.thread.get_user_context(thread_id=thread_id)
    context_block = getattr(user_context, "context", "") or ""
    
    # 3. (Tùy chọn/Khuyên dùng) Tìm kiếm thêm các Facts bền vững (edges) để hỗ trợ audit và recency
    try:
        facts = self.client.graph.search(
            user_id=user_id,
            query=cap_query(query),
            scope="edges",
            limit=20,
        )
        fact_text = render_graph_search(facts)
    except Exception:
        fact_text = ""

    return join_nonempty([context_block, fact_text], sep="\n\n")
```

**Kiểm tra riêng tầng Long-term**:
```bash
docker compose run --rm app python -m src.evaluate --impl student --reuse-seeded --only-layer long_term
```
*Các test cases đạt được*: E02 (Preference), E03 (Open loops), E08 (Recency / Conflict), E09 (User Isolation).

---

### Bước 4: Pha C - Cài đặt TODO 2/4: Episodic Memory Retrieval (`retrieve_episodic`)

Cài đặt hàm `retrieve_episodic` trong `src/memory_student.py` để tìm kiếm các trải nghiệm/trajectory lịch sử:

```python
def retrieve_episodic(self, user_id: str, query: str) -> str:
    # Bắt buộc search theo user_id với scope="episodes"
    results = self.client.graph.search(
        user_id=user_id,
        query=cap_query(query),
        scope="episodes",
        limit=15,
    )
    # Giới hạn ký tự từng episode (episode_char_cap=180) để không nuốt chửng token budget
    return render_graph_search(results, episode_char_cap=180)
```

**Kiểm tra riêng tầng Episodic**:
```bash
docker compose run --rm app python -m src.evaluate --impl student --reuse-seeded --only-layer episodic
```
*Các test cases đạt được*: E04 (Kinh nghiệm fix async timeout), E05 (Nguyên nhân thất bại & bài học).

---

### Bước 5: Pha D - Cài đặt TODO 3/4: Semantic Memory Retrieval (`retrieve_semantic`)

Cài đặt hàm `retrieve_semantic` trong `src/memory_student.py` để tìm kiếm tri thức domain trong Standalone Graph:

```python
def retrieve_semantic(self, graph_id: str, query: str) -> str:
    q = cap_query(query)
    try:
        # Khuyên dùng scope="episodes" để giữ nguyên các mã marker nguyên bản (ví dụ: PAYMENT-RULE-3)
        results = self.client.graph.search(
            graph_id=graph_id,
            query=q,
            scope="episodes",
            limit=8,
        )
    except Exception:
        # Fallback về scope="nodes" nếu cần
        results = self.client.graph.search(
            graph_id=graph_id,
            query=q,
            scope="nodes",
            limit=8,
        )
    return render_graph_search(results)
```

**Kiểm tra riêng tầng Semantic**:
```bash
docker compose run --rm app python -m src.evaluate --impl student --reuse-seeded --only-layer semantic
```
*Các test cases đạt được*: E06 (Payment retry rule), E11 (Connection pooling playbook).

---

### Bước 6: Pha E - Cài đặt TODO 4/4: Context Assembly (`assemble_context`) & Đánh giá toàn bộ (Phút 85 - 110)

Cài đặt hàm `assemble_context` trong `src/memory_student.py` để quản lý Token Budget (10/4/3/3):

```python
def assemble_context(self, layers: dict[str, str]) -> tuple[str, dict[str, dict[str, int]]]:
    # Sử dụng ContextBudgetManager phân bổ ngân sách token theo thứ tự ưu tiên:
    # 1. Short-term (10%) -> 2. Long-term (4%) -> 3. Episodic (3%) -> 4. Semantic (3%)
    return self.budget.assemble(layers)
```

**Chạy Đánh Giá Toàn Bộ & So Sánh Baseline**:
```bash
# 1. Chạy đánh giá No-Memory Baseline
docker compose run --rm app python -m src.evaluate --impl no_memory

# 2. Chạy đánh giá Student Implementation
docker compose run --rm app python -m src.evaluate --impl student --reuse-seeded

# 3. Tổng hợp báo cáo so sánh
docker compose run --rm app python -m src.compare_reports
```

*Kết quả*: Báo cáo sẽ được xuất ra `reports/benchmark.md`, `reports/benchmark_no_memory.md` và `reports/comparison.md`.

---

### Bước 7: Thực hiện Privacy Drill & Hoàn thiện Báo cáo

1. **Thực hiện quyền được quên (Right to be Forgotten)**:
   *(Lưu ý: Chỉ chạy sau khi đã hoàn thành và lưu báo cáo benchmark)*:
   ```bash
   docker compose run --rm app python -m src.forget --user-id minh-lab17
   ```

2. **Xác minh dữ liệu đã bị xóa hoàn toàn**:
   ```bash
   docker compose run --rm app python -m src.forget --user-id minh-lab17 --verify-only
   ```
   *Yêu cầu đầu ra*: `Zep user absent: True` và `Redis user keys remaining: 0`. Chụp lại màn hình terminal làm minh chứng (`privacy.png`).

3. **Viết file `README_submission.md` (Dưới 400 từ)**:
   Trả lời đầy đủ 3 câu hỏi chính:
   - Tầng bộ nhớ nào quan trọng nhất trong bài lab này? Chỉ rõ case minh họa.
   - Trực quan hóa Trade-off giữa Managed Zep Context Block vs Tự xây Redis + Qdrant.
   - Giải pháp Guardrail chống Memory Poisoning / Prompt Injection qua bộ nhớ.
   Cùng 4 câu hỏi phân tích từ `reports/comparison.md`.

---

### Bước 8: Điểm cộng Golden Set & Demo UI (60 phút cuối)

1. **Golden Set (20 cases ẩn từ Giảng viên)**:
   Khi giảng viên cung cấp file `data/golden_eval.json`, chạy lệnh:
   ```bash
   docker compose run --rm app python -m src.evaluate --impl student --reuse-seeded --golden
   ```
   *Điều kiện cộng +10đ*: Đạt **20/20 PASS** (`reports/golden_benchmark.json`).

2. **Streamlit UI Demo (+10đ)**:
   Chạy ứng dụng UI tương tác:
   ```bash
   make ui   # Truy cập http://localhost:8501
   ```
   *Checklist UI*: Load case từ `sessions.json` -> Hiển thị query/layer/user -> Chạy student retrieval & hiển thị merged context -> Cho phép chat tiếp tục giữ nguyên context.

---

## V. NHỮNG BÀI HỌC CỐT LÕI ĐẠT ĐƯỢC SAU KHI HOÀN THÀNH LAB 17

### 1. Sự Khác Biệt & Cách Phối Hợp 4 Tầng Bộ Nhớ AI Agent
- **Short-term Memory**: Là bộ nhớ làm việc tạm thời. Không thể chỉ lưu Buffer thô (raw list messages) vì token sẽ tăng tuyến tính $O(N)$. Cần kết hợp **Sliding Window + Summary Compaction** để cô đọng hội thoại nhưng giữ nguyên **Durable Notes** (các ràng buộc/deadline không được quên).
- **Long-term Memory (Declarative)**: Lưu giữ sự thật (Facts) và Preferences giữa các phiên làm việc riêng biệt. Managed Memory Graph giúp giải quyết bài toán xung đột thông tin theo nguyên tắc **Recency Wins** (Ví dụ: User đổi từ Python sang TypeScript cho project mới).
- **Episodic Memory**: Truy xuất kinh nghiệm lịch sử dựa trên quỹ đạo hành động (Trajectory). Giúp Agent học từ sai lầm quá khứ (Ví dụ: Lần trước tăng Timeout bị lỗi, thay bằng ClientSession pool mới fix được).
- **Semantic Memory**: Tri thức domain chuẩn hóa độc lập với User. Cần truy xuất bằng Standalone Graph Search để bổ sung playbook/quy tắc vào bối cảnh xử lý.

### 2. Context Engineering & Token Budget Allocation
- **Không phải cứ lấy nhiều Memory là tốt**: Truy xuất thừa thông tin sẽ gây ra hiện tượng **Context Contamination (Nhiễu bối cảnh)** và lãng phí chi phí API.
- **Chiến lược Phân bổ Token Ngân sách (10/4/3/3)**:
  Giới hạn tổng context (ví dụ 8000 tokens) theo tỉ lệ ưu tiên khắt khe: Short-term (10%), Long-term (4%), Episodic (3%), Semantic (3%).
- **Thứ tự ưu tiên ghép Context (Priority Order)**:
  `Short-Term` $\rightarrow$ `Long-Term` $\rightarrow$ `Episodic` $\rightarrow$ `Semantic`.

### 3. Managed Graph Memory (Zep V3) vs Raw KV/Vector DB (Redis / Qdrant)
- **Tự xây với Redis + Qdrant**:
  - *Ưu điểm*: Toàn quyền kiểm soát, chi phí tự host thấp, truy xuất KV siêu nhanh.
  - *Nhược điểm*: Phải tự làm thủ công toàn bộ khâu Chunking, Embedding, Vector Search, Deduplication, Graph Linking, Entity Resolution, Recency Decay và Temporal Reasoning.
- **Managed Zep Cloud V3**:
  - *Ưu điểm*: Đã tích hợp sẵn User Knowledge Graph, Tự động trích xuất Facts/Episodes, Tự động tổng hợp **Context Block** sẵn sàng bơm vào Prompt, Hỗ trợ sẵn Right-to-be-Forgotten.
  - *Nhược điểm*: Phụ thuộc vào SaaS Vendor, hạn chế tùy biến sâu thuật toán ranking graph.

### 4. Đánh Giá Agent Memory bằng Ground-Truth Evidence (Không dùng LLM Judging)
- Tránh bẫy **LLM Hallucination**: Khi đánh giá bộ nhớ Agent, nếu dùng LLM để chấm điểm câu trả lời, LLM có thể tự bịa ra đáp án nghe có vẻ hợp lý dù hệ thống retrieval lấy sai dữ liệu.
- Phương pháp **Ground-Truth Marker Matching**: Kiểm tra trực tiếp xem văn bản được retrieve có chứa chính xác các từ khóa bằng chứng (`must_contain_all`) hay chứa các từ cấm (`must_not_contain`) hay không. Đây là cách duy nhất đánh giá chính xác hiệu năng của hệ thống Memory Retrieval.

### 5. Tuân Thao Privacy-by-Design & Quyền Được Quên (GDPR Right to be Forgotten)
- Agent Memory lưu trữ thông tin cá nhân (PII) của người dùng qua nhiều phiên. Do đó hệ thống phải tích hợp:
  - **Consent Management**: Chỉ lưu trữ khi có Opt-in.
  - **PII Minimization**: Tự động lọc bớt Email, Số điện thoại trước khi lưu vào Graph.
  - **User Isolation**: Ngăn chặn rò rỉ dữ liệu chéo giữa `user_A` và `user_B`.
  - **Right to be Forgotten**: Khả năng xóa toàn bộ vết bộ nhớ của 1 User trên cả Graph Store và KV Store chỉ bằng 1 lệnh duy nhất.

---

## VI. TỔNG KẾT CHECKLIST TRƯỚC KHI NỘP BÀI

- [x] Đã hoàn thành 4 hàm trong `src/memory_student.py` (Không còn `NotImplementedError`).
- [x] `python -m src.evaluate --impl student --reuse-seeded` chạy thành công và tạo `reports/benchmark.md`.
- [x] Hit Rate tập Practice đạt $\ge 80\%$ ($\ge 9/11$ PASS).
- [x] Đã thực hiện lệnh `forget.py` và có ảnh minh chứng `privacy.png` (Xác nhận `Zep user absent: True`).
- [x] Đã tạo file `README_submission.md` (đáp ứng đúng giới hạn 400 từ).
- [x] Không commit secret (`.env`, `ZEP_API_KEY`) hoặc file `data/golden_eval.json` lên Git.
