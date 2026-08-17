# Những gì tôi sẽ học được từ Lab 17 — Multi-Memory Agent với Zep Cloud V3

> File này tổng hợp các kiến thức, kỹ năng và tư duy mà bạn sẽ tích lũy được khi hoàn thành dự án. Được viết theo lộ trình từ nền tảng lý thuyết → thực hành → tư duy hệ thống → ứng dụng production.

---

## 1. Nền tảng lý thuyết về Memory for AI Agents

### 1.1. Phân biệt 4 loại bộ nhớ — bản chất và mục đích
- **Short-term / Working Memory**: vùng nhớ nóng của cuộc hội thoại hiện tại. Bạn sẽ hiểu vì sao chỉ dùng buffer thuần sẽ làm nổ token (tăng tuyến tính theo số turn), và vì sao cần **sliding window + compaction + durable notes**.
- **Long-term / Declarative Memory**: facts, preferences, decisions bền vững qua nhiều thread. Bạn sẽ nắm được khái niệm **Context Block** của Zep — đoạn context Zep tự lắp ráp từ user graph dựa trên relevance.
- **Episodic Memory**: trajectory + outcome + provenance. Khác với long-term ở chỗ nó lưu **trải nghiệm**, không phải fact. Bạn sẽ học cách retrieval trả lời câu "lần trước đã thử cách nào, vì sao fail, fix ra sao".
- **Semantic Memory**: tri thức domain dùng chung, độc lập user. Bạn sẽ thấy sự khác biệt giữa **user-scoped graph** (long-term/episodic) và **standalone graph** (semantic KB).

### 1.2. Các thuật ngữ chuyên ngành
`Context Block`, `Episode`, `Compaction`, `Token Budget`, `Recency Wins`, `User-scoped namespace`, `Provenance`. Tất cả đều được dùng xuyên suốt code, report và README submission.

---

## 2. Zep Cloud V3 SDK — cách dùng thực tế

### 2.1. Luồng V3 chuẩn
Bạn sẽ thuộc luồng API chính:

```
user.add → thread.create → thread.add_messages → thread.get_user_context
                                                  ↓
                              graph.search(user_id=..., scope="episodes")
                              graph.search(graph_id=..., scope="episodes")
```

### 2.2. Hai kiểu graph trong Zep
- **User graph**: gắn với `user_id` — long-term facts + episodic memory.
- **Standalone graph**: gắn với `graph_id` — semantic KB dùng chung nhiều user.

### 2.3. Scope — chọn đúng scope để không mất marker
- `scope="episodes"` → giữ nguyên văn, phù hợp marker nguyên bản (`PAYMENT-RULE-3`, `CONN-POOL-FIRST`).
- `scope="nodes"` → chỉ entity/fact, dễ mất literal.
- `scope="auto"` → không ổn định, hay mất marker — lab cấm dùng cho semantic.

### 2.4. Hai chiến lược ingestion
- `prime_eval_thread` — chuẩn bị thread context cho mỗi query.
- Polling đến khi graph có dữ liệu searchable (Zep ingestion bất đồng bộ — đây là pitfall thường gặp).

---

## 3. Short-Term Memory Engineering

### 3.1. So sánh 3 chiến lược
| Chiến lược | Đặc điểm | Hạn chế |
|---|---|---|
| Buffer | Giữ tất cả | Token tăng tuyến tính, vượt budget |
| Summary | Nén old turns | Mất chi tiết, có thể "quên" constraint |
| **Sliding Window** (default lab) | summary + last K turns + durable notes | Cần tune K hợp lý |

### 3.2. Compaction có chủ đích
Bạn sẽ học triết lý: compaction **không phải "tóm tắt văn hóa"**. Phải ưu tiên 4 loại thông tin:
- **State** — trạng thái hiện tại
- **Decision** — quyết định đã chốt
- **TODO** — task còn open
- **Constraint** — ràng buộc không được phá (ví dụ deadline `REVIEW-DEADLINE-1600`)

### 3.3. Durable Notes
Một kỹ thuật quan trọng: thông tin `deadline / constraint` được tách riêng khỏi raw turns. Dù raw turn bị evict vì sliding window, durable notes vẫn còn. Đây là pattern "what to keep vs what to drop".

---

## 4. Context Engineering — kỹ năng sản xuất quan trọng nhất

### 4.1. Token Budget Allocation
Bạn sẽ học cách phân bổ budget tỉ lệ `10/4/3/3`:

```
short-term  10%   ← mức ưu tiên cao nhất vì luôn liên quan trực tiếp đến câu hiện tại
long-term    4%
episodic     3%
semantic     3%
```

Tổng cộng chỉ ~20% context window cho memory layers; phần còn lại dành cho system prompt, instructions và output.

### 4.2. Priority Order khi merge
```
STM → LT → EP → SEM
```
Khi budget không đủ, layer sau bị cắt trước. Bạn sẽ hiểu vì sao short-term **luôn** chiếm slot — nó là "ground truth" của câu đang hỏi.

### 4.3. Tư duy "lấy nhiều ≠ tốt"
Có một anti-pattern phổ biến: tham retrieve để "chắc ăn". Lab dạy bạn:
- Retrieval quá nhiều → **Context Contamination** (nhiễu bối cảnh).
- Token tăng → chi phí API tăng, latency tăng.
- No-memory baseline có thể có **token reduction cao** nhưng **hit rate thấp** — đây là bài học quan trọng về trade-off.

---

## 5. Recency Wins & Conflict Resolution

### 5.1. Nguyên tắc khi fact mâu thuẫn
Khi user thay đổi preference qua nhiều phiên (ví dụ Python → TypeScript cho project mới):
- **Fact mới được ưu tiên** trong context hiện tại.
- **Fact cũ vẫn được giữ** trong user graph để phục vụ history / provenance / audit.

### 5.2. Áp dụng thực tế trong lab
- Case E08: `BLUEBIRD-42` + `TypeScript` + `NestJS` (recency).
- Case E10: `REVIEW-DEADLINE-1600` + `Friday` + `16:00` (compaction giữ constraint).

---

## 6. User Isolation — kỹ năng bảo mật cốt lõi

### 6.1. Namespace scoping
Bạn sẽ thấy hậu quả nếu sai `user_id`: dữ liệu user A leak sang user B. Zep SDK thiết kế user-scoped API để chặn lỗi này ở mức API, nhưng bạn vẫn phải:
- Truyền đúng `user_id` vào mọi call.
- Không nhầm `user_id` với `graph_id`.

### 6.2. Kỹ thuật test isolation
Case E09 ép bạn kiểm chứng: query của `lan-lab17` **không** được trả về fact của `minh-lab17` (`ORCHID-27`) và ngược lại. Đây là pattern test bạn sẽ dùng cho mọi hệ thống multi-tenant sau này.

---

## 7. Privacy by Design & GDPR

### 7.1. 4 nguyên tắc bạn sẽ thực hành
- **Consent**: chỉ ingest khi `memory_opt_in = true` (xem `data/consent.json`).
- **PII Minimization**: tự động redact email/phone trong message content.
- **Right to be Forgotten**: xóa user-scoped memory + Redis keys chỉ bằng 1 lệnh (`python -m src.forget --user-id minh-lab17`).
- **Verify after delete**: chạy `--verify-only` để chứng minh memory không còn được retrieve.

### 7.2. Phân biệt xóa user-scoped vs shared KB
Bạn sẽ hiểu: khi forget một user, **chỉ xóa user-scoped data**. Semantic graph dùng chung nhiều user thì giữ nguyên — vì đó là tri thức domain, không phải PII của user demo.

---

## 8. Ground-Truth Evaluation — phương pháp luận đánh giá

### 8.1. Vì sao KHÔNG dùng LLM judging cho memory test
- LLM có thể **tự bịa** câu trả lời nghe hợp lý dù retrieval sai.
- Nó **che lấp** lỗi memory system bên dưới.
- Bạn không đo được thật sự **retrieval hit rate**.

### 8.2. Marker Matching Pattern
```
PASS = (∀marker ∈ must_contain_all: marker ∈ retrieved_text)
      ∧ (∀marker ∈ must_not_contain: marker ∉ retrieved_text)
```

Đây là pattern cực kỳ giá trị — áp dụng được cho mọi eval hệ thống RAG / search / retrieval sau này.

### 8.3. So sánh A/B hai implementation
Lab dạy cách chạy:
- `--impl no_memory` → baseline không có memory.
- `--impl student` → agent có memory.
- `src/compare_reports.py` → so sánh hit rate, token, latency.

Từ đó đưa ra kết luận dựa trên dữ liệu, không phải cảm tính.

---

## 9. Trade-off: Managed Memory (Zep) vs Self-Hosted (Redis + Qdrant)

### 9.1. Managed Zep
- **+** Có sẵn user knowledge graph, auto extract facts/episodes, Context Block sẵn sàng bơm vào prompt, hỗ trợ Right-to-be-Forgotten.
- **–** Phụ thuộc SaaS vendor, ít tùy biến ranking algorithm, tốn chi phí subscription.

### 9.2. Self-Hosted (Redis + Qdrant)
- **+** Toàn quyền kiểm soát, self-host rẻ, KV cực nhanh.
- **–** Phải tự làm: chunking, embedding, vector search, dedup, graph linking, entity resolution, recency decay, temporal reasoning.

### 9.3. Bài học chiến lược
- Với team nhỏ / MVP → managed Zep thắng.
- Với hệ thốống enterprise / cần customization sâu + data residency → self-host thắng.
- Phương án hybrid (managed cho user memory + self-host cho domain KB) thường là ngon nhất.

---

## 10. Orchestration với LangGraph

### 10.1. Pattern skeleton
```
route(query) → retrieve(query) → assemble(layers) → answer
```

### 10.2. Ý tưởng cốt lõi
- **Router**: phân loại query → layer nào cần retrieve.
- **Retrieve**: gọi từng layer, trả về text.
- **Assemble**: áp budget + priority, merge thành 1 context string.
- **Answer**: đưa context + query cho LLM.

Bạn sẽ học cách **tách retrieval logic ra khỏi generation logic** — vì retrieval test được bằng marker matching, còn generation thì không.

---

## 11. Memory Poisoning Guardrail — tư duy phòng thủ

### 11.1. Mối nguy
User có thể cố tình (hoặc vô tình) inject thông tin sai vào memory qua nhiều phiên. Ví dụ: "Tôi ghét Python" → nếu agent tin và ghi vào long-term, mọi gợi ý sau sẽ lệch.

### 11.2. Guardrail patterns bạn sẽ học
- **Provenance**: mỗi fact phải truy ngược được nguồn (episode nào, khi nào).
- **Confidence scoring**: fact có bao nhiêu lần được xác nhận, mâu thuẫn bao nhiêu lần.
- **Cross-check với semantic KB**: nếu user fact mâu thuẫn với domain rule đã compiled → cảnh báo.
- **Heartbeat dry-run** (xem `src/heartbeat.py`): chỉ de-dup / đánh stale / recap, **không tự ý thêm instruction mới** vào durable memory.

---

## 12. Compiled Knowledge Base — kỹ thuật nâng cao

Bạn sẽ làm quen với `data/compiled_kb.jsonl`: tri thức đã được extract/curate sẵn (entity/decision pages, source IDs, contradictions, freshness).

Tư duy: **không phải mọi query đều nên bắt đầu từ raw transcript**. Một knowledge base đã compiled sẽ:
- Nhanh hơn (skip ingestion cost).
- Ổn định hơn (không bị nhiễu bởi turn mới).
- Traceable hơn (mỗi fact có source ID rõ ràng).

---

## 13. Kỹ năng DevOps / quy trình

- **Docker Compose**: build image, quản lý services, multi-container network.
- **Env management**: `.env.example` vs `.env`, secret hygiene, `.gitignore` đúng file nhạy cảm.
- **Reproducible eval**: `--reuse-seeded`, `--only-layer`, `--golden` flags — cách chạy eval incremental.
- **CI mindset**: `pytest -q` trước khi commit, lock hành vi bằng unit test.

---

## 14. Tư duy hệ thống & kỹ năng mềm

### 14.1. Trade-off thinking
Mọi quyết định đều có trade-off:
- Latency vs recall?
- Cost vs accuracy?
- Privacy vs personalization?
- Managed vs self-hosted?

Lab ép bạn **đưa ra lý do**, không chỉ chọn đáp án.

### 14.2. Viết tài liệu kỹ thuật
- `README_submission.md` ≤ 400 từ phải truyền tải được insight lớn nhất.
- `comparison.md` phải có số liệu, không có cảm tính.
- Screenshots phải chứa output thật (`[OK]`, marker, verification).

### 14.3. Phân tích lỗi & debug
- Khi recall sai → check `user_id`? `scope`? polling? marker literal?
- Khi leak user → check scope long-term/episodic.
- Khi semantic ra preference → check scope đang dùng.

Đây là kỹ năng **diagnostic reasoning** — tìm lỗi logic ở integration point thay vì fix triệu chứng.

---

## 15. Áp dụng ngoài bài lab

Sau khi hoàn thành, bạn sẽ có khung tư duy để:
- **Thiết kế chatbot doanh nghiệp**: hỏi đáp nội bộ có nhớ lịch sử từng nhân viên.
- **Xây dựng AI tutor**: nhớ được học viên nào hay sai ở đâu (episodic).
- **Tạo trợ lý cá nhân (assistant)**: nhớ preference, thói quen, project đang làm.
- **Audit tự động**: mọi decision của agent đều có provenance truy ngược.
- **Hệ thốống RAG production**: biết cách budget, scope, evaluate bằng marker thay vì LLM judging.

---

## Tóm tắt 1 dòng

> Hoàn thành Lab 17, bạn sẽ có đủ tư duy + kỹ năng để **thiết kế, xây dựng, đánh giá và vận hành** một AI Agent có memory thật sự — không chỉ "nhớ trong context" mà là nhớ xuyên phiên, có provenance, tôn trọng privacy, và được verify bằng ground-truth.
