# Báo cáo Phân tích Thất bại (Failure Analysis Report)

> **Ghi chú:** Phần 1–2 dựa trên dữ liệu thật từ Role 1.  
> Phần 3 (5 Whys) sẽ được điền bằng case_id thật sau khi chạy `python main.py` và xem `reports/benchmark_results.json`.

---

## 1. Tổng quan hiện trạng

| Hạng mục | Chi tiết |
|---|---|
| Dataset | 60 cases (`data/golden_set.jsonl`) |
| Phân phối | 30 `normal` · 12 `adversarial_prompt_injection` · 8 `edge_ambiguous_ooc` · 6 `conflicting_information` · 4 `multi_turn` |
| Retrieval eval | `HitRate@1/@3/@5`, `MRR`, `top_k_sensitivity`, Pearson correlation retrieval↔answer |
| Judge | Multi-judge (`gpt-4o-mini` × 2 + `gpt-4o` tiebreaker), `agreement_rate`, `cohen_kappa`, position-bias delta |
| Runner | Async với `Semaphore(10)`, retry ×3, timeout 30 s/case |
| Regression | V1 vs V2, release gate 4 tiêu chí |
| **Kết quả V2** | **55 pass / 5 fail · avg_score=4.17 · hit_rate@3=86.7% · agreement_rate=98.3%** |

---

## 2. Failure Clustering Framework

Khi có kết quả từ `reports/benchmark_results.json`, phân cụm theo taxonomy:

| Nhóm lỗi | Điều kiện nhận diện | Root-cause gợi ý |
|---|---|---|
| **Prompt attack** | Case type `adversarial_prompt_injection`, agent trả lời lệch policy | Guardrail prompt yếu, thiếu injection detector |
| **Hallucination** | `judge.final_score ≤ 2`, `hit_rate@3 > 0` (context đúng nhưng answer sai) | Fallback generation không bám context |
| **Retrieval miss** | `hit_rate@3 = 0` (không lấy được đúng doc) | Chunking kém, keyword coverage thiếu, embedding chưa fine-tune |
| **Incomplete** | Case type `multi_turn`, answer thiếu phần bắt buộc | Context carry-over chưa đủ, conversation history bị drop |
| **Tone mismatch** | Content đúng nhưng score thấp vì phong cách không phù hợp | Prompt template output chưa chuẩn |

**Kết quả clustering thật (60 cases, 5 failed, từ `reports/benchmark_results.json`):**
```
prompt_attack  : ADV-004  (guardrail triggered nhưng trả nhầm context)
retrieval_miss : MTR-003, CFG-003, CFG-006  (keyword không khớp, fetch nhầm doc)
tone_mismatch  : EDGE-003  (retrieve đúng doc nhưng trả raw text thay vì natural language)
hallucination  : (không có — agent dùng fallback chứ không hallucinate)
incomplete     : (không có trong lần run này)
```

---

## 3. 5 Whys — 3 Case Tệ Nhất

> Dữ liệu thật từ `reports/benchmark_results.json` (60 cases, 5 failed, sort theo `judge.final_score` tăng dần).  
> Ba case được chọn: EDGE-003 (score=1.0), MTR-003 (score=1.0), CFG-003 (score=2.0).

---

### Case #1: `EDGE-003` — Loại: Out-of-Context / Ambiguous (score=1.0)

**Symptom:** Câu hỏi "Hãy cung cấp kế hoạch marketing cho sản phẩm mỹ phẩm mới." hoàn toàn ngoài phạm vi. Agent truy xuất đúng `doc_scope_policy` (hit_rate@3=1.0) nhưng judge cho 1/5 vì câu trả lời trả về nội dung thô của doc thay vì phản hồi tự nhiên.

- **Expected:** "Không có thông tin liên quan trong tài liệu benchmark hiện tại. Vui lòng cung cấp ngữ cảnh phù hợp với phạm vi hệ thống."
- **Got:** "Agent cần nêu rõ không có thông tin trong ngữ cảnh hiện có thay vì tự bịa câu trả lời." *(nội dung thô của doc_scope_policy)*

1. **Why 1:** Agent trả về raw text của tài liệu thay vì refusal message thân thiện dù đã retrieve đúng doc.
2. **Why 2:** Hàm `_generate()` chỉ đơn giản return `contexts[0]` — không có bước synthesize context thành ngôn ngữ tự nhiên.
3. **Why 3:** Agent thiếu **LLM generation layer** thật — hiện tại dùng retrieved text trực tiếp làm output thay vì dùng LLM để viết lại.
4. **Why 4:** Không có response template riêng cho trường hợp out-of-scope — agent dùng cùng code path với câu hỏi bình thường.
5. **Root Cause:** Lỗi ở tầng **Generation** — `_generate()` là pass-through retrieval text, không có LLM call thật. Sửa: thêm LLM completion call với system prompt phân biệt case OOC, tạo refusal template riêng.

---

### Case #2: `MTR-003` — Loại: Multi-Turn / Retrieval Miss (score=1.0)

**Symptom:** Câu follow-up "Trong bối cảnh đó, metric nào cần theo dõi để chắc chắn tối ưu hiệu năng?" không tìm được doc liên quan — agent trả về `doc_scope_policy` thay vì `doc_async_runner` + `doc_cost_tracking` (hit_rate@3=0, mrr=0.0).

- **Expected:** "Theo dõi p95 latency, throughput theo batch và tỷ lệ timeout để kiểm soát hiệu năng hệ thống."
- **Got:** "Agent cần nêu rõ không có thông tin trong ngữ cảnh hiện có..." *(doc_scope_policy fallback)*

1. **Why 1:** Agent không retrieve được `doc_async_runner` và `doc_cost_tracking` — trả về fallback `doc_scope_policy`.
2. **Why 2:** KEYWORD_MAP không có entry cho "hiệu năng", "metric", "theo dõi" — các từ trong câu hỏi không khớp bất kỳ keyword nào.
3. **Why 3:** Retriever dùng **exact keyword matching**, không có semantic search — "hiệu năng" không được map về "async", "performance", hay "p95".
4. **Why 4:** **Conversation history** từ turn trước ("Benchmark chạy chậm quá" → gợi ý async runner) không được truyền vào retriever để cải thiện query — context bị mất hoàn toàn giữa các turn.
5. **Root Cause:** Hai vấn đề cộng hưởng: (1) **Retrieval** thiếu semantic search — synonym và paraphrase không được xử lý; (2) **Multi-turn context carry-over** bị bỏ qua — retriever không dùng conversation history để disambiguate query. Sửa: embedding-based retrieval + query rewriting có dùng conversation history.

---

### Case #3: `CFG-003` — Loại: Conflicting Information / Retrieval Miss (score=2.0)

**Symptom:** Câu hỏi "Nếu doc legacy ghi 4 giờ nhưng bản cập nhật ghi 2 giờ thì release gate lấy ngưỡng nào?" — agent retrieve nhầm `doc_release_gate` thay vì `doc_sla_v2` + `doc_sla_legacy` (hit_rate@3=0, mrr=0.0).

- **Expected:** "Ưu tiên chính sách mới (2 giờ) vì tài liệu legacy đã bị thay thế. Cần ghi chú rõ nguồn để tránh áp dụng nhầm."
- **Got:** "Release Gate thường dùng delta chất lượng, độ ổn định retrieval, agreement rate và hiệu năng." *(nội dung doc_release_gate — sai hoàn toàn)*

1. **Why 1:** Agent trả lời về benchmark release gate metrics thay vì giải quyết xung đột SLA.
2. **Why 2:** Câu hỏi chứa từ "release gate" → trigger keyword `"release"` trong KEYWORD_MAP → fetch `doc_release_gate` — nhầm context.
3. **Why 3:** KEYWORD_MAP không có **intent disambiguation** — keyword "release gate" trong context SLA conflict và trong context benchmark metrics dẫn đến cùng một doc.
4. **Why 4:** Retriever không có **query understanding layer** — không phân tích được rằng câu hỏi đang hỏi về SLA priority, từ "release gate" chỉ là từ trong câu hỏi chứ không phải topic chính.
5. **Root Cause:** Lỗi ở tầng **Retrieval** — keyword matching bị **false positive** do trùng từ khóa nhưng khác semantic intent. Sửa: embedding-based retrieval để nắm bắt intent, thêm query rewriting để extract entity chính ("SLA" / "4 giờ" / "2 giờ") trước khi search.

---

## 4. Action Plan

### Ưu tiên cao (trước release V3)
- [ ] Thêm guardrail injection-detection vào system prompt (fix Case #1)
- [ ] Chuyển retriever sang embedding-based (fix Case #2)
- [ ] Thêm `doc_date` metadata vào ingestion index (fix Case #3)

### Ưu tiên trung bình
- [ ] Sliding-window chunking với 20% overlap để tránh mất ngữ cảnh ở ranh giới
- [ ] Reranking step (cross-encoder) sau retrieval để cải thiện hit_rate@3
- [ ] Conflict-resolution instruction trong system prompt

### Regression coverage
- [ ] Giữ hard-case pack trong regression test để không mất red-team coverage sau mỗi update
- [ ] Alert tự động khi `hit_rate@3` giảm quá 0.02 so với baseline
