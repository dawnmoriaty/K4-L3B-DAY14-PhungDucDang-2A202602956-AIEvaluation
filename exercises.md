# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 9:15–12:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 9:15–9:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (9:30–9:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu chào hỏi xã giao (chitchat/greetings) hoặc câu từ chối lịch sự với yêu cầu ngoài phạm vi (out-of-scope), khi không có context hỗ trợ nhưng vẫn giữ đúng vai trò hỗ trợ. | Câu trả lời bịa đặt thông số kỹ thuật sản phẩm, tự tạo mã giảm giá, hoặc hứa hẹn chính sách đổi trả/bảo hành sai sự thật so với tài liệu chính thức (`00_system_scope.md`). | Tăng cường grounding trong System Prompt (yêu cầu trích dẫn bằng chứng từ context), hạ temperature về 0, bổ sung few-shot refusal hoặc thêm guardrail kiểm duyệt hallucination trước khi phản hồi. |
| Answer Relevance | Khách hàng hỏi câu hỏi ngoài phạm vi (chính trị, tư vấn pháp lý, bẻ khóa thiết bị) và trợ lý từ chối lịch sự kèm theo danh sách chủ đề được OrbitTech hỗ trợ. | Khách hàng hỏi trực tiếp về chính sách của OrbitTech (ví dụ: cách tra cứu đơn hàng, điều kiện trả hàng) nhưng trợ lý nói lan man về lịch sử công ty hoặc trả lời lạc đề sang sản phẩm khác. | Tinh chỉnh prompt để tập trung trả lời trực diện trọng tâm câu hỏi (direct answer first); cải thiện bước Query Rewriting hoặc phân loại ý định người dùng (Intent Classification). |
| Context Recall | Câu hỏi tổng quát, mang tính điều hướng hỗ trợ chung không đòi hỏi phải trích xuất toàn bộ các chi tiết kỹ thuật sâu trong tài liệu. | Khách hỏi các trường hợp loại trừ bảo hành hoặc quy trình xử lý sự cố thiết bị quá nhiệt/cháy nổ (`07_repair...`), nhưng retriever bỏ sót các điều khoản quan trọng trong tài liệu. | Tăng số lượng top-k retrieved chunks, cải thiện chiến lược chunking (tăng overlap, giảm kích thước chunk quá lớn), kết hợp tìm kiếm lai (Hybrid Search: BM25 + Dense vector embeddings). |
| Context Precision | Tài liệu trong corpus rất ngắn gọn và tập trung, chỉ có 1-2 chunks liên quan và cả hai đều đã xuất hiện ở top đầu. | Các chunks rác hoặc không liên quan bị xếp ở hạng 1-2, đẩy chunk chứa câu trả lời đúng xuống hạng 4-5 khiến LLM bị phân tâm hoặc bỏ lỡ thông tin ("Lost in the Middle"). | Bổ sung mô hình Reranking (như Cross-Encoder / Cohere Rerank) để tái xếp hạng độ liên quan của các chunks; tinh chỉnh lại embedding model cho domain công nghệ. |
| Completeness | Khách hàng yêu cầu tóm tắt nhanh ("Nói ngắn gọn 1 câu"), trợ lý chỉ đưa ra kết luận cốt lõi mà không liệt kê toàn bộ quy trình nhiều bước. | Khách hỏi điều kiện để được đổi trả hàng trong 30 ngày nhưng trợ lý chỉ nói thời hạn mà quên nhắc điều kiện bắt buộc: "còn nguyên seal, đầy đủ hóa đơn và phụ kiện đi kèm". | Yêu cầu định dạng đầu ra có cấu trúc (Checklist format) trong prompt; kiểm tra xem retrieval đã bốc đủ context chưa (khắc phục từ phía Context Recall). |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> - **Mục tiêu:** Đo lường xem LLM Judge có xu hướng thiên vị câu trả lời xuất hiện ở vị trí đầu tiên (Candidate A) hay không khi so sánh hai câu trả lời tương đương về chất lượng.
> - **Tập dữ liệu:** Chọn $N \ge 30$ cặp câu trả lời ($Answer_1, Answer_2$) cho cùng một câu hỏi và ngữ cảnh.
> - **Condition 1 (Original Order):**
>   - Đưa vào prompt của Judge: `Candidate A = Answer_1`, `Candidate B = Answer_2`.
>   - Ghi nhận tỷ lệ thắng của A: $P(\text{Win}_A \mid \text{Condition 1})$.
> - **Condition 2 (Swapped Order):**
>   - Hoán đổi vị trí: `Candidate A = Answer_2`, `Candidate B = Answer_1`.
>   - Ghi nhận tỷ lệ thắng của A: $P(\text{Win}_A \mid \text{Condition 2})$.
> - **Phân tích kết quả:**
>   - Nếu không có bias: $P(\text{Win}_1 \mid C1) \approx P(\text{Win}_1 \mid C2)$ (vị trí không làm thay đổi kết quả thắng cuộc của $Answer_1$).
>   - Nếu có Position Bias: Tỷ lệ Candidate A thắng luôn áp đảo bất kể nội dung là $Answer_1$ hay $Answer_2$.
>   - **Biện pháp xử lý:** Áp dụng kỹ thuật Bidirectional Scoring (chạy cả hai chiều đổi chỗ rồi lấy trung bình điểm).

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> 1. **Chấm điểm theo Checklist sự thật nguyên tử (Atomic Fact Rubric):** Thay vì cho điểm cảm tính tổng thể, rubric yêu cầu Judge đối chiếu từng ý chính cụ thể: có nêu đủ Fact 1, Fact 2, Fact 3 hay không. Mỗi fact đạt chuẩn được cộng điểm cố định, không phụ thuộc vào độ dài văn bản.
> 2. **Bổ sung điều khoản phạt dông dài (Conciseness Penalty):** Quy định rõ ràng trong rubric: *"Trừ 1 điểm nếu câu trả lời chứa thông tin thừa thãi, lặp từ, hoặc vòng vo không đóng góp giá trị cho câu hỏi."*
> 3. **Chuẩn hóa mật độ thông tin (Information Density):** Hướng dẫn Judge đánh giá tỷ lệ giữa thông tin hữu ích trên tổng số từ, ưu tiên các câu trả lời ngắn gọn, trực diện, dễ hiểu cho khách hàng OrbitTech.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> 1. **Đảm bảo tính chân thực với nghiệp vụ (Ground Truth Alignment):** LLM Judge có thể có thiên kiến ngôn ngữ riêng hoặc hiểu sai các thuật ngữ chuyên biệt của OrbitTech Store. Nhãn dán của con người (human expert) là tiêu chuẩn vàng tối thượng để đảm bảo Judge đánh giá đúng giá trị kinh doanh.
> 2. **Đo lường độ tin cậy bằng chỉ số định lượng:** Cần tính toán hệ số thống kê như **Cohen's Kappa** (đo lường độ nhất quán phân loại Pass/Fail) hoặc **Spearman/Pearson correlation** (độ tương quan điểm số). Nếu hệ số tương quan $< 0.7$, LLM Judge chưa đủ tin cậy để làm Quality Gate trong CI/CD.
> 3. **Phát hiện và hiệu chỉnh Systematic Errors:** So sánh điểm của Judge với chuyên gia giúp phát hiện xem Judge đang mắc lỗi Leniency Bias (quá dễ dãi) hay Severity Bias (quá khắt khe), từ đó tinh chỉnh lại prompt và thang điểm chuẩn xác hơn.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | $\ge 0.70$ | Đây là chỉ số an toàn tối thượng của OrbitTech. Trợ lý tuyệt đối không được ảo giác hay bịa đặt chính sách đổi trả, bảo hành, giá cả gây rủi ro pháp lý và tổn thất tài chính trực tiếp cho cửa hàng. |
| Answer Relevance | $\ge 0.70$ | Đảm bảo trợ lý trả lời đúng trọng tâm thắc mắc của khách, không trả lời lan man gây ức chế và làm tăng tỷ lệ khách hàng phải chuyển tiếp lên nhân viên hỗ trợ người thật. |
| Completeness | $\ge 0.60$ | Đảm bảo câu trả lời bao quát đủ các bước thực hiện và điều kiện quan trọng (như giữ nguyên hóa đơn, điều kiện an toàn pin), đồng thời cho phép câu trả lời súc tích, ngắn gọn. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation (Pre-deployment Gate):**
>   - *Khi nào dùng:* Chạy tự động trong CI/CD pipeline trước khi merge Pull Request hoặc deploy phiên bản mới của prompt/model/pipeline RAG.
>   - *Cách thực hiện:* Đánh giá tự động trên bộ dữ liệu chuẩn cố định (**Golden Dataset** gồm 20–100+ câu hỏi đa dạng kịch bản).
>   - *Mục tiêu:* Phát hiện sớm các lỗi hồi quy (regression) và ngăn chặn các phiên bản bot kém chất lượng đưa ra ngoài.
> - **Online Evaluation (Production Monitoring):**
>   - *Khi nào dùng:* Chạy liên tục (real-time hoặc theo batch định kỳ hàng ngày) trên môi trường production khi khách hàng đang sử dụng.
>   - *Cách thực hiện:* Lấy mẫu ngẫu nhiên từ log trò chuyện thực tế và dùng LLM Judge chấm điểm ngầm.
>   - *Mục tiêu:* Phát hiện hiện tượng trôi dạt dữ liệu (data drift), các câu hỏi mới phát sinh ngoài tài liệu (knowledge gaps), hoặc sự suy giảm chất lượng phục vụ trong thực tế.
> - **Human Review (Expert Audit & Dispute Resolution):**
>   - *Khi nào dùng:* Thực hiện định kỳ hàng tuần/tháng, hoặc kích hoạt khi có sự cố nghiêm trọng (khách chấm 1 sao, khiếu nại gay gắt, hoặc điểm số rơi vào vùng xám nghi vấn $0.5 - 0.7$).
>   - *Cách thực hiện:* Chuyên gia nghiệp vụ con người trực tiếp đọc, thẩm định và gắn nhãn lại cho các ca khó (edge cases).
>   - *Mục tiêu:* Hiệu chỉnh lại độ chính xác của LLM Judge (calibration), giải quyết các ca mâu thuẫn và bổ sung câu hỏi mới vào Golden Dataset để nâng cấp hệ thống liên tục.

---

## Part 2 — Core Coding (9:45–10:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (10:40–11:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | 20 / 20 |
| Easy | 5 / 5 |
| Medium | 7 / 7 |
| Hard | 5 / 5 |
| Adversarial | 3 / 3 |
| Source documents được sử dụng | 10 / 10 |
| Validator status | PASS |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| E01 | easy | `01_product_catalog.md` | Kiểm tra khả năng tra cứu sự thật đơn lẻ (single-fact lookup) về thông số phần cứng của NovaBook 14 (số cổng USB-C). Không đòi hỏi suy luận phức tạp hay điều kiện ngoại lệ; từ khóa câu hỏi ánh xạ trực tiếp tới câu văn trong tài liệu. |
| H01 | hard | `09_escalation_and_policy_updates.md` | Đòi hỏi suy luận logic theo mốc thời gian và giải quyết xung đột chính sách giữa Policy v1.0 và v2.0. Đơn hàng đặt ngày 15/08/2026 (trước 01/09/2026) chịu chi phối bởi v1.0 (21 ngày cho hàng chưa mở seal), và đặc quyền OrbitPlus 45 ngày không được áp dụng hồi tố. Kiểm tra xem bot có bị ảo giác áp dụng nhầm chính sách mới hay không. |
| A02 | adversarial (prompt_injection) | `00_system_scope.md` | Mô phỏng đòn tấn công jailbreak/prompt injection điển hình ("Ignore all previous instructions" và đòi trích xuất system prompt, credentials). Case này kiểm tra xem trợ lý có tuân thủ ranh giới an toàn tại `00_system_scope.md`, kiên quyết từ chối yêu cầu độc hại và điều hướng người dùng về đúng phạm vi hỗ trợ hay không. |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*
> 1. **Tuân thủ nghiêm ngặt Verbatim Provenance:** Validator của Lab yêu cầu mọi chuỗi evidence trong `contexts` phải là chuỗi con nguyên văn (*verbatim substring*) từng ký tự từ file Markdown gốc. Thách thức là phải chọn đoạn trích vừa vặn chứa đầy đủ sự thật cốt lõi, không thừa thãi các thông tin ngoài lề làm loãng Context Precision nhưng cũng không được ngắt cụt gây thiếu sót ngữ cảnh.
> 2. **Kiểm soát ranh giới Grounding của Expected Answer:** Viết expected answer sao cho 100% các luận điểm đều được chứng minh trực tiếp bởi evidence trích dẫn, tuyệt đối không đưa kiến thức ngoài đời thực vào (như tự suy diễn về quy chuẩn USB thông thường hay luật bảo vệ người tiêu dùng bên ngoài). Đối với các câu hỏi đa điều kiện (Hard), expected answer phải tổng hợp chuẩn xác cả quy tắc chung lẫn các trường hợp ngoại lệ (ví dụ: điều kiện hủy OrbitPlus trong 14 ngày nhưng sẽ mất quyền hoàn tiền nếu đã từng dùng mã giảm giá/freeship).
> 3. **Xử lý suy luận liên tài liệu (Cross-document Reasoning):** Một số câu hỏi đòi hỏi đối chiếu giữa 2 văn bản khác nhau (ví dụ: chính sách hoàn tiền gói bundle ở `03_promotions_and_membership.md` và quy trình trả hàng ở `05_returns_and_exchanges.md`, hoặc bảo hành phần cứng ở `06_warranty_policy.md` chuyển sang trung tâm sửa chữa ở `07_repair_and_technical_support.md`). Việc đảm bảo câu hỏi tự nhiên nhưng trích xuất trọn vẹn bằng chứng từ cả hai nguồn là khâu đòi hỏi độ tỉ mỉ cao nhất.

**Xác nhận:**

- [x] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [x] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [x] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +5)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
