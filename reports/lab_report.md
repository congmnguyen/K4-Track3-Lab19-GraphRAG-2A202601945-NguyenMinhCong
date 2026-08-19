# Báo cáo Lab 19 — GraphRAG vs Flat RAG

**Học viên:** Nguyễn Minh Công · **Mã:** 2A202601945 · **Ngày:** 19/08/2026

> Benchmark dưới đây là lần chạy mẫu 5 câu lưu tại `outputs/`, không phải kết quả của golden set chính thức. Notebook hiện đọc `data/graphrag_golden_50_first5000.csv` gồm 25 câu G5000-26…G5000-50 từ upstream. Cần chạy lại với 5.000 dòng nguồn, Neo4j và API credentials trước khi nộp kết quả benchmark chính thức.

## Phần 1 — Thuyết minh kỹ thuật và phân tích lỗi

### 1. Coreference resolution

Ca khó điển hình: “Microsoft discussed OpenAI with regulators. The company later announced a new model.” *The company* có thể là Microsoft hoặc OpenAI. Nếu thay bằng Microsoft, extractor có thể tạo cạnh sai `Microsoft-DEVELOPED->model`. Pipeline chỉ thay khi antecedent duy nhất, rõ ràng và trong cùng chunk; trường hợp mơ hồ được giữ nguyên và ghi `unresolved_mentions`. False negative làm giảm recall nhưng an toàn hơn false edge lan sang mọi truy vấn hai hop.

### 2. Entity resolution và lexical guard

Ngưỡng vector là **0.90**, tối đa 5 láng giềng cùng type. Alias phổ biến được xử lý trước; ANN chỉ sinh candidate, `merge_guard()` mới quyết định gộp dựa trên tên đã bỏ suffix và lexical ratio ≥0.72. `Apple` và `Apple Music` có thể có cosine >0.85 do cùng token nhưng bị chặn vì Company và Technology khác nhau (type blocking). `Sam Altman` và `Steve Altman` cũng không được gộp dù cùng họ. Union-Find chỉ hợp nhất candidate qua cả type blocking và guard.

### 3. Super-node mitigation

Ba node dự kiến có degree cao nhất là Microsoft, Google và OpenAI; cell `graph_checks()` xuất degree thực từ Neo4j sau ingestion, nên báo cáo không giả lập con số chưa đo. Với degree >100, retrieval lấy tối đa 50 cạnh mới nhất; toàn query tối đa 250 cạnh và 14.000 ký tự. Chính sách giảm fan-out, latency và token, giữ tin hiện hành. Rủi ro là mất cạnh lịch sử. Cải thiện bằng nhận diện thời gian trong query, lọc date trước cap và chia quota theo relation/community.

### 4. Benchmark và ca lỗi

| Metric | Flat | Graph | Delta |
|---|---:|---:|---:|
| Comprehensiveness | 3.2 | 5.0 | +1.8 |
| Faithfulness | 4.8 | 5.0 | +0.2 |
| Multi-hop reasoning | 2.6 | 4.8 | +2.2 |
| Latency (s) | 0.464 | 0.770 | +0.306 |
| Tokens | 590.4 | 979.0 | +388.6 |

**Flat thất bại — G04.** Một chunk nói Microsoft đầu tư OpenAI, chunk khác nói OpenAI phát triển GPT-4. Top-k ưu tiên đoạn GPT-4 nên Flat bỏ quan hệ đầu tư. Graph đi theo `Microsoft-INVESTED_IN->OpenAI-DEVELOPED->GPT-4`, giữ evidence và chunk ID cho hai hop.

**Graph khó khăn — G05.** Nếu cạnh 2019 bị extractor bỏ sót hoặc newest-first cap loại, graph chỉ thấy 2023 và không mô tả được diễn tiến. Vector fallback vẫn có thể tìm bài cũ. Khắc phục: temporal filtering, extraction retry/checkpoint, sufficiency check với hop 3, rồi hợp nhất graph và vector.

### 5. Trade-off, kiểm soát agent và scale

Flat có index đơn giản, latency thấp, ít token; phù hợp factoid. Graph tốn extraction, resolution, graph storage và traversal nhưng tốt khi nối nhiều tài liệu. Hybrid được chọn vì graph có thể thiếu cạnh còn vector giữ nguyên văn bản.

Tôi từ chối cosine mọi cặp mention O(N²), không phù hợp 350 MB. Thiết kế dùng type blocking + FAISS + lexical guard. Ở khoảng 100.000 bài, bottleneck đầu tiên là LLM extraction (chi phí/rate limit/retry), sau đó entity resolution và database write. Giải pháp: async queue có idempotency key, batch/checkpoint, ANN/HNSW theo type, `UNWIND` 1.000 rows, partition thời gian/community và incremental ingestion. Secrets chỉ lấy từ environment/Colab Secrets.

## Phần 2 — Reflection và action plan

### 1. Mapping bài giảng

| Khái niệm | Module | Hàm | Quan sát |
|---|---|---|---|
| Conservative coreference | M1 | `resolve_coref_batch()`, `run_coref()` | Batch lỗi fallback nguyên văn và audit |
| Schema allowlist | M2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Record sai schema bị loại |
| Bulk Cypher | M2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | `UNWIND`, batch 1.000 |
| Entity resolution | M3 | `build_resolution_map()`, `merge_guard()`, `UF` | Alias → ANN → guard → union-find |
| Super-node cap | M4 | `retrieve_graph_context()` | 50/node, 250/query, 14.000 chars |
| LLM judge | M5 | `judge_answer()`, `run_evaluation()` | Ba score, rationale, checkpoint |

### 2. Debugging và bài học

Lỗi khó nhất là nhầm similarity cao với identity. Embedding đơn lẻ dễ gộp sản phẩm vào công ty hoặc hai người cùng họ. Tôi dùng blocking theo type, Unicode normalization, suffix stripping, lexical guard, manual alias nhỏ và audit `MERGE_MANUAL`/`MERGE_VECTOR`/`REJECT_GUARD`. Entity resolution cần bằng chứng và khả năng audit, không chỉ threshold.

Provenance rỗng là lỗi vận hành khác. `graph_checks()` bắt buộc `invalid_provenance_edges == 0`; production nên mở rộng kiểm tra để coi chuỗi ngày rỗng là invalid.

### 3. Action plan đồ án

**Đồ án:** trợ lý tra cứu quy định và phụ thuộc chính sách nội bộ. Chọn Hybrid GraphRAG vì câu hỏi nối quy định, phòng ban, vai trò, biểu mẫu và phiên bản; Flat vẫn giữ nội dung chưa thành cạnh.

- Nodes: `Policy`, `Clause`, `Department`, `Role`, `Form`, `System`, `Version`.
- Relations: `OWNS`, `APPLIES_TO`, `REQUIRES`, `SUPERSEDES`, `REFERENCES`, `USES_FORM`, `EFFECTIVE_FROM`.
- Resolution: ID tài liệu là khóa mạnh; dictionary alias; ANN chỉ sinh candidate; owner/type/date làm guard.
- Super-node: quota theo relation và thời gian hiệu lực, ưu tiên version active, vẫn có global cap.
- Evaluation: golden factoid/multi-hop/version comparison; faithfulness yêu cầu citation tới clause/version.

## Tự đánh giá

| Tiêu chí | Điểm | Ghi chú |
|---|---:|---|
| Hiểu GraphRAG | 4/5 | Nắm pipeline và failure mode |
| Kiểm soát agent | 4/5 | Giữ schema, scale guard, tránh O(N²) |
| Chất lượng graph | 4/5 | Có provenance/audit; cần chạy lại Aura thật |
| Phân tích/debug | 4/5 | Có root cause và mitigation |
