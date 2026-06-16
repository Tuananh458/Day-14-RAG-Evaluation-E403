# Day 14 — Reflection
## Evaluation Report & Failure Analysis

---

## 1. Benchmark Results Summary

Paste results từ Exercise 3.2 và tóm tắt:

**Overall pass rate:** 15% (3/20 passed - kết quả phản ánh các lỗi cố tình thiết lập trong Simulated Agent để kiểm thử hệ thống đánh giá)

**Average scores:**

| Metric | Average | Min | Max | Std Dev |
|--------|---------|-----|-----|---------|
| Faithfulness | 0.39 | 0.00 | 0.62 | 0.19 |
| Relevance | 0.34 | 0.08 | 0.83 | 0.19 |
| Completeness | 0.64 | 0.11 | 1.00 | 0.29 |
| Overall Score | 0.45 | 0.15 | 0.78 | 0.17 |

**Score interpretation (theo bài giảng):**
- Bao nhiêu metrics ở Good (0.8–1.0)? 0 metrics
- Bao nhiêu metrics ở Needs Work (0.6–0.8)? 1 metric (Completeness ở 0.64)
- Bao nhiêu metrics ở Significant Issues (<0.6)? 3 metrics (Faithfulness ở 0.39, Relevance ở 0.34, Overall Score ở 0.45)

**Failure type distribution:**

| Failure Type | Count | Percentage |
|--------------|-------|------------|
| hallucination | 6 | 30% |
| irrelevant | 5 | 25% |
| incomplete | 1 | 5% |
| off_topic | 5 | 25% |
| refusal | 0 | 0% |

---

## 2. Top 3 Worst Failures — 5 Whys Analysis

Theo bài giảng: "Phân loại failure TRƯỚC KHI fix. Đừng fix từng failure riêng lẻ — CLUSTER rồi fix root cause."

### Failure 1

**Question:** What is the role of a Reranker in search?

**Agent Answer:** A Reranker sorts document chunks.

**Scores:** Faithfulness: 0.00 | Relevance: 0.25 | Completeness: 0.21 | Overall: 0.15

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Câu trả lời của agent quá ngắn gọn và thiếu hầu hết các từ khóa mô tả ý nghĩa cốt lõi của Reranker (như "reorder", "semantic relevance", "top of the list"). |
| Why 1 | Tại sao xảy ra? | Hệ thống heuristics so khớp từ vựng đánh giá điểm Faithfulness bằng 0.00 và Relevance bằng 0.25 do số lượng từ khớp quá ít. |
| Why 2 | Tại sao Why 1 xảy ra? | Agent trả lời quá ngắn gọn và sơ sài ("A Reranker sorts document chunks.") thay vì trả lời đầy đủ và chi tiết. |
| Why 3 | Tại sao Why 2 xảy ra? | Prompt của agent không có quy định rõ ràng về định dạng câu trả lời hoặc độ dài tối thiểu, cũng như không có các ví dụ mẫu (few-shot examples) để định hướng độ dài. |
| Why 4 | Root cause là gì? | Lỗi sinh câu trả lời bị thiếu thông tin cốt lõi ("Answer is missing key information — increase context window or improve generation"). |

**Root cause (from `find_root_cause()`):**
> "Answer is missing key information — increase context window or improve generation"

**Bạn có đồng ý với root cause suggestion không? Tại sao?**
> Đồng ý. Điểm Completeness của câu trả lời này rất thấp (0.21) do thiếu hụt thông tin trầm trọng so với đáp án mong đợi. Do đó, việc cải thiện thế hệ câu trả lời (improve generation) bằng cách bổ sung prompt hướng dẫn chi tiết và few-shot examples là giải pháp đúng đắn.

**Proposed fix (cụ thể, actionable):**
> 1. Thêm chỉ dẫn rõ ràng vào prompt của agent yêu cầu trả lời chi tiết và đầy đủ các khía cạnh kỹ thuật của câu hỏi.
> 2. Bổ sung 2-3 few-shot examples thể hiện các câu trả lời đầy đủ, chất lượng cao trong hệ thống RAG.

---

### Failure 2

**Question:** How do you calculate Context Precision and why does chunk order matter?

**Agent Answer:** Context Precision is a database optimization technique used to increase disk read speeds.

**Scores:** Faithfulness: 0.20 | Relevance: 0.18 | Completeness: 0.11 | Overall: 0.16

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Agent trả lời hoàn toàn sai lệch định nghĩa của Context Precision (nhầm lẫn với tối ưu hóa cơ sở dữ liệu đĩa). |
| Why 1 | Tại sao xảy ra? | Mô hình LLM bị hallucinate (bịa đặt kiến thức) hoặc không truy xuất đúng thông tin. |
| Why 2 | Tại sao Why 1 xảy ra? | Context được retrieve chứa thông tin nhiễu hoặc retriever không lấy được tài liệu phù hợp liên quan đến RAG Context Precision. |
| Why 3 | Tại sao Why 2 xảy ra? | Khoảng cách ngữ nghĩa giữa câu hỏi và tài liệu nguồn bị lệch, hoặc embedding model hoạt động kém hiệu quả trên các thuật ngữ viết tắt của RAG. |
| Why 4 | Root cause là gì? | Ngữ cảnh bị thiếu hoặc không liên quan ("Context is missing or irrelevant — improve retrieval"). |

**Root cause:**
> "Context is missing or irrelevant — improve retrieval"

**Proposed fix:**
> 1. Nâng cấp bộ retriever bằng cách tích hợp Hybrid Search và Reranker để kéo đúng tài liệu kỹ thuật về RAG Evaluation vào ngữ cảnh.
> 2. Tinh chỉnh chunk size và tăng độ phủ (overlap) của các chunk để đảm bảo ngữ cảnh nguyên vẹn được truyền tới LLM.

---

### Failure 3

**Question:** Design a regression testing strategy for a RAG agent where prompt templates are updated daily by non-engineers.

**Agent Answer:** Implement a daily automated test suite in CI/CD running the Golden Dataset, comparing metrics against baseline, and blocking deploy if drop > 0.05.

**Scores:** Faithfulness: 0.45 | Relevance: 0.08 | Completeness: 0.56 | Overall: 0.36

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Điểm Relevance cực kỳ thấp (0.08) mặc dù nội dung câu trả lời rất sát nghĩa với câu hỏi. |
| Why 1 | Tại sao xảy ra? | Số lượng từ trùng lặp trực tiếp giữa câu hỏi (chứa các từ khóa như "regression", "testing", "strategy", "prompt", "templates", "updated", "daily", "non-engineers") và câu trả lời rất ít. |
| Why 2 | Tại sao Why 1 xảy ra? | Heuristic so khớp từ vựng (word overlap) rất nhạy cảm với cách dùng từ khác biệt (paraphrasing). Agent trả lời trực diện vào giải pháp kỹ thuật ("automated test suite", "CI/CD", "baseline") thay vì lặp lại các từ khóa của câu hỏi. |
| Why 3 | Tại sao Why 2 xảy ra? | Hệ thống đánh giá hiện tại chỉ dùng lexical overlap đơn giản mà không sử dụng semantic similarity (như embeddings hoặc LLM-as-judge). |
| Why 4 | Root cause là gì? | Câu trả lời không trực tiếp giải quyết câu hỏi ("Answer does not address the question — improve prompt clarity" - theo chẩn đoán heuristics của code). |

**Root cause:**
> "Answer does not address the question — improve prompt clarity" (Lỗi chẩn đoán sai do hạn chế của lexical evaluation metric, thực chất là một false negative).

**Proposed fix:**
> 1. Thay đổi phương pháp đánh giá từ word overlap sang dùng LLM-as-Judge hoặc Semantic Similarity để đánh giá chính xác hơn các câu trả lời diễn đạt bằng từ đồng nghĩa.
> 2. Điều chỉnh prompt của agent sao cho lặp lại hoặc phản chiếu một số từ khóa chính của câu hỏi để tối ưu cho các bộ đánh giá lexical đơn giản.

---

## 3. Failure Clustering

Theo bài giảng: "Fix 1 root cause giải quyết nhiều failures cùng lúc."

**Cluster Analysis:**

| Cluster | Root Cause | Failures in cluster | Priority |
|---------|-----------|--------------------:|----------|
| 1 | Hallucination / Sai kiến thức do Retriever yếu (E03, M01, M02) | 6 | High |
| 2 | Lỗi so khớp từ vựng / Diễn đạt đồng nghĩa bị đánh trượt (E01, H02, H03, H05) | 5 | Medium |
| 3 | Thiếu sót thông tin / Trả lời quá ngắn do Prompt thiếu ràng buộc (M03, M07) | 6 | High |

**Nếu chỉ fix 1 cluster, bạn chọn cluster nào? Tại sao?**
> Chọn **Cluster 1 (Hallucination / Sai kiến thức)**. Vì trong hệ thống RAG thực tế, việc đưa ra thông tin bịa đặt hoặc sai lệch kiến thức là nghiêm trọng nhất, ảnh hưởng trực tiếp đến uy tín và độ an toàn của hệ thống. Khắc phục cluster này bằng cách cải tiến Retriever (hybrid + rerank) sẽ giải quyết triệt để nguồn cấp dữ liệu cho LLM.

---

## 4. Improvement Log (from `generate_improvement_log`)

Paste output của `generate_improvement_log()`:

```
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | off_topic | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| F002 | irrelevant | Answer does not address the question — improve prompt clarity | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F003 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness and relevance | Open |
| F004 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F005 | hallucination | Answer is missing key information — increase context window or improve generation | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F006 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness and relevance | Open |
| F007 | incomplete | Answer is missing key information — increase context window or improve generation | Implement hallucination checker to filter unsupported claims | Open |
| F008 | irrelevant | Answer does not address the question — improve prompt clarity | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F009 | off_topic | Answer does not address the question — improve prompt clarity | Add few-shot examples showing complete answers to improve completeness and relevance | Open |
| F010 | off_topic | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| F011 | irrelevant | Answer does not address the question — improve prompt clarity | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F012 | irrelevant | Answer does not address the question — improve prompt clarity | Add few-shot examples showing complete answers to improve completeness and relevance | Open |
| F013 | off_topic | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| F014 | irrelevant | Answer does not address the question — improve prompt clarity | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F015 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness and relevance | Open |
| F016 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F017 | hallucination | Multiple issues detected — review full pipeline | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
```

**Thêm 3 improvement suggestions từ `generate_improvement_suggestions()`:**
1. Implement hallucination checker to filter unsupported claims
2. Increase chunk size in RAG pipeline to reduce context fragmentation
3. Add few-shot examples showing complete answers to improve completeness and relevance

---

## 5. Regression Testing Strategy

### CI/CD Integration

**Câu 1: Khi nào chạy `run_regression()` trong production system?**
> *Mô tả CI/CD integration point (ví dụ: trước mỗi merge to main, sau mỗi prompt change, etc.):*
> Chạy tự động trong CI/CD pipeline trước mỗi lần merge code vào nhánh `main`, sau mỗi lần thay đổi prompt template của agent, nâng cấp phiên bản mô hình LLM, hoặc thay đổi cấu hình bộ retriever.

**Câu 2: Threshold regression 0.05 có phù hợp domain của bạn không?**
> *Strict hơn hay loose hơn? Tại sao?*
> Phù hợp. Đối với domain RAG Evaluation, sụt giảm điểm trung bình lớn hơn 0.05 (5%) là dấu hiệu rõ ràng của việc hồi quy chất lượng do thay đổi prompt hoặc code. Tuy nhiên, đối với hệ thống y tế hoặc pháp luật, threshold này cần strict hơn nữa (ví dụ: drop > 0.02 là phải chặn).

**Câu 3: Khi phát hiện regression — block deployment hay chỉ alert?**
> *Your answer + giải thích trade-off:*
> Block deployment nếu sự sụt giảm xảy ra ở metric Faithfulness (vì an toàn thông tin là tối cao).
> Đối với các metric khác như Completeness, có thể chỉ gửi Alert cho đội ngũ phát triển để rà soát thủ công, tránh làm chậm chu kỳ release phần mềm không cần thiết.

**Câu 4: Eval pipeline nên chạy ở đâu trong CI/CD flow?**

```
Code change → [Unit Tests] → [Run Eval on Golden Dataset (Regression Check)] → [Security & Compliance Scan] → Deploy
              (bước 1)       (bước 2)                                          (bước 3)
```
> *Điền 3 bước eval vào flow trên:*
> - Bước 1: Unit Tests
> - Bước 2: Run Eval on Golden Dataset (Regression Check)
> - Bước 3: Security & Compliance Scan

---

## 6. Continuous Improvement Loop

Theo bài giảng: Evaluate → Analyze → Improve → Augment (add to benchmark) → lặp lại

**Sau lab hôm nay, 3 actions tiếp theo bạn sẽ làm để improve agent:**

| Priority | Action | Metric sẽ improve | Expected impact |
|----------|--------|-------------------|-----------------|
| 1 | Tích hợp Hybrid Search & Reranking | Context Precision, Context Recall | Tăng điểm Precision từ 0.55 lên 0.95+ |
| 2 | Thiết lập Prompt Few-shot | Completeness | Tăng điểm Completeness trung bình thêm +0.20 |
| 3 | Thay đổi Evaluator sang LLM-as-Judge | Answer Relevancy | Giảm tỷ lệ báo lỗi giả (false negative) xuống dưới 5% |

**Bạn sẽ thêm failure cases nào vào benchmark cho sprint tiếp theo?**
> *List 2–3 cases mới cần thêm:*
> 1. Các câu hỏi chứa phủ định kép (double negation) để thử thách khả năng suy luận của LLM.
> 2. Các câu hỏi chứa thông tin lỗi thời (outdated context) để kiểm tra tính cập nhật thông tin của retriever.
> 3. Các truy vấn dài dòng từ người dùng thực tế trên production để kiểm tra tính bền bỉ của prompt routing.

---

## 7. Framework Reflection

**Framework bạn đã dùng trong lab:** Ragas-inspired word-overlap heuristic.

**Nếu dùng trong production, bạn sẽ chọn framework nào? Tại sao?**
> *Tham khảo trade-offs table trong bài giảng:*

| Tiêu chí | Lý do chọn |
|----------|------------|
| Focus phù hợp vì... | Chọn **DeepEval** vì nó tích hợp cực tốt với unit testing pytest, giúp phát triển ứng dụng dạng lập trình viên rất nhanh và trực quan. |
| CI/CD integration vì... | DeepEval hỗ trợ sẵn CLI và tự động tạo báo cáo HTML đẩy lên GitHub Actions mà không cần viết thêm script orchestration. |
| Team workflow vì... | DeepEval cho phép dễ dàng điều chỉnh prompt của judge và thêm bớt các TestCase trực tiếp từ giao diện code thân thuộc. |
