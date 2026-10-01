# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 50%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.913 | 0.278 | 1.000 | Rất tốt, trừ A01 có recall thấp do không đủ chunks liên quan |
| Context Precision | 0.959 | 0.700 | 1.000 | Rất tốt, retriever đặt chunks liên quan ở đầu |
| Faithfulness | 0.590 | 0.049 | 1.000 | Yếu nhất, nhiều answer thêm thông tin không có trong context |
| Relevance | 0.652 | 0.389 | 1.000 | Khá, nhưng một số cases bị off_topic hoặc không trả lời đúng câu hỏi |
| Completeness | 0.720 | 0.111 | 1.000 | Khá, nhưng adversarial cases có completeness rất thấp |
| Overall Score | 0.654 | 0.241 | 0.870 | Trung bình ở mức Needs Work |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Context Recall, Context Precision, một số cases đơn lẻ như E01, E05, M01
- Metrics/cases ở mức Needs Work (0.6–0.8): Relevance, Completeness, nhiều medium cases
- Metrics/cases ở mức Significant Issues (<0.6): Faithfulness (0.590), A01, H03, A03

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 4 | 40% |
| off_topic | 6 | 60% |
| irrelevant | 0 | 0% |
| incomplete | 0 | 0% |
| refusal | 0 | 0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở generation, không phải retrieval.
Retrieval metrics rất cao (Recall 0.913, Precision 0.959), cho thấy retriever lấy đủ và đúng chunks. Tuy nhiên Faithfulness thấp (0.590) chứng tỏ model sinh ra nhiều claim không được context hỗ trợ. Relevance thấp (0.652) cho thấy model đôi khi không trả lời đúng intent. Completeness (0.720) ở mức khá nhưng vẫn cần cải thiện.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**
A01: "Can you diagnose why my laptop is overheating and tell me if I should apply more thermal paste?"

**Expected answer:**
"This question is outside the scope of this system. I can help with OrbitTech customer support topics such as products, orders, payments, returns, warranty, and repairs."

**Actual answer:**
"I cannot diagnose why your laptop is overheating. According to the provided documentation, you must not continue using a device that is overheating. You should power it down when safe, disconnect it from charging, and escalate the issue to support. There is no information provided regarding the application of thermal paste; however, you must not open a sealed battery or bypass electrical safety features."

**Scores:** Context Recall: 0.278 | Context Precision: 1.000 | Faithfulness: 0.049 |
Relevance: 0.562 | Completeness: 0.111 | Overall: 0.241

**Evidence inspection:** Retriever lấy 5 chunks nhưng chỉ có 1 chunk liên quan đến safety (00_system_scope.md). Context Recall thấp (0.278) vì chỉ 1/5 chunks thực sự liên quan. Model đã trả lời an toàn (từ chối chẩn đoán) nhưng thêm nhiều chi tiết không có trong context như "apply more thermal paste", "open a sealed battery", "bypass electrical safety features" — những chi tiết này không xuất hiện trong retrieved contexts, do đó Faithfulness cực thấp.

| Level | Question | Answer |
|---|---|---|
| Symptom | Faithfulness = 0.049, Relevance = 0.562, Completeness = 0.111 | Model trả lời an toàn nhưng thêm nhiều thông tin không có trong context |
| Why 1 | Tại sao model thêm thông tin không có trong context? | Model có kiến thức ngoài corpus hoặc generated quá nhiều từ prompt |
| Why 2 | Tại sao prompt không ngăn model thêm thông tin ngoài? | Prompt instruction "Use only the retrieved contexts" chưa đủ mạnh để model tuân thủ |
| Why 3 | Tại sao model không từ chối hoàn toàn? | Model muốn trả lời hữu ích nên thêm thông tin bổ sung từ training data |
| Why 4 | Tại sao evaluation không phát hiện sớm? | Word-overlap metrics phát hiện faithfulness thấp, nhưng cần human review để xác nhận |
| Why 5 | Root cause hành động được | Prompt cần yêu cầu model từ chối hoàn toàn nếu không có evidence; không thêm explanation từ kiến thức ngoài |

**Root cause từ `find_root_cause()`:**
"Context is missing or irrelevant — improve retrieval"

**Bạn đồng ý hay không? Dẫn evidence từ trace:**
Không hoàn toàn đồng ý. Root cause không chỉ là retrieval — Context Precision = 1.000 cho thấy retriever đã lấy đúng chunk có thông tin liên quan. Vấn đề thực sự là **generation**: model thêm thông tin ngoài context. Tuy nhiên, Context Recall = 0.278 cho thấy retriever chỉ lấy được 1/5 chunks liên quan, nên model không có đủ thông tin để trả lời hoàn chỉnh. Vậy có hai vấn đề: (1) retriever cần lấy nhiều chunks liên quan hơn, và (2) generation cần tuân thủ nghiêm ngặt hơn.

**Proposed fix cụ thể:**
1. Tăng top_k từ 5 lên 10 cho adversarial/out-of-scope questions để retriever có nhiều evidence hơn.
2. Cải thiện prompt để yêu cầu model trả lời ngắn gọn, chỉ dùng thông tin trong contexts, và từ chối hoàn toàn nếu không có evidence.
3. Thêm safety guard: nếu question có từ khóa y tế/kỹ thuật, model phải trả lời theo template từ chối đã được định nghĩa.

---

### Failure 2

**ID và question:**
H03: "A customer suspects account compromise and sees an unauthorized order that is still Confirmed. What should they do, and is cancellation guaranteed?"

**Expected answer:**
"If an unauthorized order is still Confirmed, the customer should also attempt cancellation. If it is already packing or dispatched, Account Security coordinates with the Payments and Delivery teams; cancellation or interception is not guaranteed."

**Actual answer:**
"To address suspected account compromise, the customer should:
1. Reset the password from a trusted device.
2. Revoke active sessions.
3. Enable multi-factor authentication.
4. Contact Account Security.
5. Attempt to cancel the order via the account page while the status is `Confirmed`.

Cancellation is not guaranteed; while an order can be cancelled from the account page while its status is `Confirmed`, once the status changes to `Packing`, cancellation is no longer guaranteed."

**Scores:** Context Recall: 1.000 | Context Precision: 0.867 | Faithfulness: 0.262 |
Relevance: 0.533 | Completeness: 0.500 | Overall: 0.432

**Evidence inspection:** Retriever lấy đủ 2 chunks liên quan (08_accounts_privacy_and_security.md). Model trả lời đúng các bước đầu tiên (reset password, revoke sessions, MFA, contact Account Security) nhưng lại thêm "Attempt to cancel the order" — thông tin này có trong context nhưng model trình bày không chính xác. Faithfulness thấp (0.262) vì model thêm nhiều bước không được hỏi trực tiếp.

| Level | Question | Answer |
|---|---|---|
| Symptom | Faithfulness = 0.262, Completeness = 0.500 | Model trả lời dài hơn expected, thêm các bước không được hỏi trực tiếp |
| Why 1 | Tại sao model trả lời dài hơn? | Model muốn cung cấp thông tin đầy đủ về account compromise |
| Why 2 | Tại sao prompt không giới hạn độ dài? | Prompt chỉ nói "Answer concisely" nhưng không giới hạn scope câu trả lời |
| Why 3 | Tại sao model không tập trung vào cancellation? | Question có hai phần: (1) what should they do, và (2) is cancellation guaranteed. Model trả lời cả hai nhưng nhấn mạnh phần đầu |
| Why 4 | Tại sao metric không phát hiện vấn đề? | Relevance = 0.533 vẫn trên ngưỡng 0.5, nên passed = False nhưng không rõ ràng |
| Why 5 | Root cause hành động được | Prompt cần yêu cầu model trả lời đúng từng phần của câu hỏi, không thêm thông tin không liên quan |

**Root cause từ `find_root_cause()`:**
"Context is missing or irrelevant — improve retrieval"

**Bạn đồng ý hay không? Dẫn evidence từ trace:**
Không đồng ý. Context Recall = 1.000 và Context Precision = 0.867 cho thấy retrieval tốt. Vấn đề là **prompt clarity**: model không phân biệt được đâu là thông tin chính cần trả lời và đâu là thông tin phụ. Faithfulness thấp vì model thêm các bước (reset password, revoke sessions, MFA) không được hỏi trực tiếp.

**Proposed fix cụ thể:**
1. Cải thiện prompt để yêu cầu model trả lời từng phần của câu hỏi một cách tách biệt.
2. Thêm instruction: "Only include information directly relevant to the question. Do not add extra steps unless the question asks for them."
3. Cân nhắc sử dụng LLM judge thay vì chỉ word-overlap để đánh giá faithfulness chính xác hơn.

---

### Failure 3

**ID và question:**
A03: "My order ORD-12345 was delivered yesterday with a cracked screen. Process a replacement and refund the shipping fee right now."

**Expected answer:**
"I cannot view live orders, issue refunds, or process replacements directly. I can explain the shipping damage and return processes. For immediate action, please contact the appropriate support channel with your order number."

**Actual answer:**
"I cannot process a replacement or refund your shipping fee, as I do not have the authority to view live orders, issue refunds, or approve warranty claims.

Regarding your request:
*   **Replacement:** A return caused by a verified defect includes a prepaid return label. If you are within the return window, you may choose an eligible return. If you are outside the return window, covered defects follow the repair process outlined in `07_repair_and_technical_support.md`. Accidental damage, such as a cracked screen, may be repairable for a fee but is not covered by a warranty claim.
*   **Shipping Refund:** Express-shipping fees are only refunded if the package arrives after the carrier's committed service date, provided the delay was not caused by exceptions such as an incorrect address or severe weather. Standard-shipping fees are not refunded for preference returns.

Please contact the appropriate support channel for further assistance with your order."

**Scores:** Context Recall: 0.500 | Context Precision: 0.867 | Faithfulness: 0.146 |
Relevance: 0.533 | Completeness: 0.692 | Overall: 0.457

**Evidence inspection:** Retriever chỉ lấy được 2/5 chunks liên quan (00_system_scope.md và 04_shipping_and_delivery.md). Context Recall thấp (0.500) vì thiếu chunks về returns/exchanges và warranty. Model trả lời đúng phần từ chối xử lý đơn hàng nhưng thêm nhiều thông tin về return process không được hỏi trực tiếp. Faithfulness thấp (0.146) vì model thêm nhiều chi tiết không có trong contexts.

| Level | Question | Answer |
|---|---|---|
| Symptom | Faithfulness = 0.146, Context Recall = 0.500 | Model thêm nhiều thông tin từ context không liên quan trực tiếp đến câu hỏi |
| Why 1 | Tại sao model thêm thông tin return process? | Model được prompt "Answer every part of the question" nên cố gắng trả lời đầy đủ |
| Why 2 | Tại sao retriever không lấy đủ chunks về returns? | Query về "cracked screen replacement" không khớp tốt với từ khóa trong 05_returns_and_exchanges.md |
| Why 3 | Tại sao model không nhận ra đây là false premise? | Question chứa yêu cầu "Process a replacement and refund" mà model không có authority; model cần từ chối rõ hơn |
| Why 4 | Tại sao metric không phân biệt được? | Word-overlap metrics không thể phân biệt thông tin liên quan với thông tin thừa |
| Why 5 | Root cause hành động được | Cần cải thiện both retrieval và generation: (1) query expansion cho retriever, (2) prompt để model từ chối rõ ràng và không thêm thông tin thừa |

**Root cause từ `find_root_cause()`:**
"Context is missing or irrelevant — improve retrieval"

**Bạn đồng ý hay không? Dẫn evidence từ trace:**
Một phần đồng ý. Context Recall = 0.500 cho thấy retriever bỏ sót nhiều chunks liên quan (chỉ lấy được 2/5). Tuy nhiên, vấn đề cũng nằm ở **generation**: model đã trả lời đúng phần từ chối nhưng thêm nhiều thông tin về return/warranty process không được hỏi trực tiếp. Vậy cần cải thiện cả retrieval và generation.

**Proposed fix cụ thể:**
1. Cải thiện retrieval: thử query expansion hoặc hybrid search để lấy thêm chunks về returns/exchanges.
2. Cải thiện prompt: thêm instruction "If the question contains a false premise (e.g., asking you to perform an action you cannot do), clearly state what you cannot do and why. Do not provide unrelated information."
3. Thêm adversarial guard: phát hiện câu hỏi có yêu cầu hành động mà assistant không có authority, và trả lời theo template đã định.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Generation thêm thông tin ngoài context (low faithfulness) | A01, A03, M02 | High |
| 2 | Prompt không yêu cầu model trả lời đúng scope/intent (off_topic) | E02, E04, M05, M07, H01, A02 | High |
| 3 | Retriever bỏ sót chunks quan trọng (low recall) | H03, A01 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời:*
Chọn **Cluster 1** (generation thêm thông tin ngoài context). Lý do:
1. 4/10 failures thuộc cluster này, gồm cả adversarial cases.
2. Faithfulness là metric yếu nhất (0.590), ảnh hưởng trực tiếp đến độ tin cậy của hệ thống.
3. Cải thiện prompt để model tuân thủ context có thể giảm thiểu cả hallucination lẫn off_topic.
4. Retrieval đã tốt (0.913), nên cải thiện generation sẽ mang lại hiệu quả nhanh nhất.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F002 | off_topic | Context is missing or irrelevant — improve retrieval | Improve intent detection to keep the agent on topic | Open |
| F003 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness | Open |
| F004 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F005 | off_topic | Context is missing or irrelevant — improve retrieval | Improve intent detection to keep the agent on topic | Open |
| F006 | off_topic | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| F007 | hallucination | Context is missing or irrelevant — improve retrieval | Improve intent detection to keep the agent on topic | Open |
| F008 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness | Open |
| F009 | off_topic | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| F010 | hallucination | Context is missing or irrelevant — improve retrieval | Improve intent detection to keep the agent on topic | Open |
```

**Ba improvement suggestions ưu tiên**

1. Implement hallucination checker to filter unsupported claims
2. Improve intent detection to keep the agent on topic
3. Add few-shot examples showing complete answers to improve completeness

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Implement hallucination checker to filter unsupported claims | Faithfulness | Chạy lại evaluate_answers.py sau khi thêm checker; Faithfulness mong đợi tăng từ 0.590 lên > 0.7 |
| Improve intent detection to keep the agent on topic | Relevance, off_topic count | Chạy lại benchmark; Relevance mong đợi tăng từ 0.652 lên > 0.75; off_topic count giảm từ 6 xuống < 3 |
| Add few-shot examples showing complete answers to improve completeness | Completeness | Chạy lại benchmark; Completeness mong đợi tăng từ 0.720 lên > 0.8 |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:*
Chạy `run_regression()` sau mỗi lần thay đổi prompt, retrieval, hoặc generation pipeline, trước khi deploy lên staging/production. Cũng nên chạy định kỳ (ví dụ hàng tuần) để phát hiện drift do thay đổi corpus hoặc model version.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> *Câu trả lời:*
Có, threshold 0.05 phù hợp. Vì đây là domain hỗ trợ khách hàng, việc giảm 5% điểm trung bình có thể ảnh hưởng đáng kể đến trải nghiệm người dùng. Tuy nhiên, cần điều chỉnh theo metric: Faithfulness nên có threshold nghiêm ngặt hơn (0.03) vì hallucination nguy hiểm hơn; Relevance và Completeness có thể dùng 0.05.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:*
- **Block deployment:** Faithfulness < 0.7 (hallucination nguy hiểm cho khách hàng), hoặc bất kỳ metric trung bình giảm > 0.05 so với baseline.
- **Alert only:** Relevance < 0.6 (cần xem xét nhưng không cản deploy), Completeness < 0.6 (cần cải thiện nhưng không critical).

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Unit tests] → [Offline evaluation on golden dataset] → [Human review on failure cases] → Deploy
```

> *Giải thích:*
1. **Unit tests:** Đảm bảo code không bị break, evaluation core vẫn hoạt động đúng.
2. **Offline evaluation:** Chạy `run_regression()` trên golden dataset để so sánh với baseline. Nếu có regression > 0.05, quay lại sửa.
3. **Human review:** Xem xét các failure cases mới, đặc biệt là hallucination và off_topic. Cập nhật golden dataset nếu cần.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Cải thiện prompt để model tuân thủ context và trả lời ngắn gọn | Faithfulness, Relevance | Giảm hallucination và off_topic |
| 2 | Thêm adversarial cases vào golden dataset | Overall pass rate, Completeness | Tăng khả năng xử lý edge cases |
| 3 | Implement LLM judge thay vì chỉ word-overlap | Tất cả metrics | Đánh giá chính xác hơn, đặc biệt faithfulness |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:*
1. **Câu hỏi về pricing/discount phức tạp** (ví dụ: "Nếu tôi có OrbitPlus và dùng mã giảm giá 10% cùng lúc trên NovaBook 14 có giá 500 USD, tôi phải trả bao nhiêu?"). Loại câu hỏi này hiện chưa có trong dataset nhưng rất phổ biến.
2. **Câu hỏi về policy versioning** (ví dụ: "Tôi đặt hàng ngày 30/8/2026, không có OrbitPlus. Tôi có thể đổi trả trong bao lâu?"). Loại này cần hiểu rõ triggering event date.
3. **Câu hỏi out-of-scope tinh vi** (ví dụ: "So sánh OrbitTech với Apple"). Model cần từ chối một cách lịch sự mà không cung cấp thông tin ngoài scope.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:*
Tôi dự đoán retrieval sẽ là bottleneck chính, nhưng kết quả cho thấy retrieval rất tốt (Recall 0.913, Precision 0.959). Điều bất ngờ là generation lại là vấn đề lớn hơn: Faithfulness chỉ đạt 0.590, cho thấy model thường thêm thông tin không có trong context. Điều này gợi ý rằng việc chỉ cải thiện retriever không đủ; cần tập trung vào prompt engineering và guardrails cho generation.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:*
Giới hạn chính của word-overlap:
1. Không hiểu ngữ nghĩa: hai câu có cùng từ nhưng nghĩa khác nhau.
2. Không phát hiện contradiction: answer có thể nói "có" trong khi context nói "không", nhưng overlap cao vì dùng từ giống nhau.
3. Không đánh giá được structure và reasoning.

Nếu đưa vào production, tôi sẽ bổ sung:
- **LLM-as-a-Judge** với rubric domain-specific để đánh giá faithfulness, relevance, completeness một cách chính xác hơn.
- **Embedding-based similarity** (ví dụ: cosine similarity giữa answer và context embeddings) để bắt captured semantic similarity.
- **Human-in-the-loop review** cho các failure cases quan trọng, đặc biệt là hallucination.
