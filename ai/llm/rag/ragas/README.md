# RAGAS trong RAG

> [Knowledge Tree](../../../../README.md) / [AI](../../../README.md) / [LLM](../../README.md) / [RAG](../README.md) / **RAGAS**

**RAGAS** thường được hiểu là **Retrieval Augmented Generation Assessment** — bộ phương pháp và thư viện giúp đánh giá hệ thống **Retrieval-Augmented Generation (RAG)** bằng các chỉ số cho cả phần **retrieval** và **generation**. Thay vì chỉ demo vài câu “trông có vẻ đúng”, RAGAS giúp tạo vòng lặp đo lường: chạy bộ câu hỏi đánh giá, lấy `retrieved_contexts`, `response`, `reference` nếu có, rồi chấm các metric như **Context Precision**, **Context Recall**, **Faithfulness** và **Answer Relevancy**.

> **RAG app → Evaluation dataset → RAGAS metrics → Compare experiments → Improve retrieval/prompt/data**

RAGAS không tự làm RAG tốt hơn. Nó giúp đội phát triển nhìn thấy lỗi nằm ở đâu: tài liệu không được retrieve, context bị nhiễu, câu trả lời không bám bằng chứng, hoặc câu trả lời đúng nguồn nhưng không đúng ý người hỏi.

## Điều hướng

- **Node cha:** [RAG](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** [Các trường hợp ứng dụng RAG](../use-cases/README.md)
- **Khái niệm sau trong cùng nhánh:** Chưa có.

## A. 5 câu hỏi cốt lõi

### 1. RAGAS là gì, và nó giải quyết bài toán nào trong RAG?

**Trả lời:** RAGAS là framework/thư viện đánh giá chất lượng ứng dụng LLM, đặc biệt hữu ích cho RAG. Nó đo các khía cạnh riêng của pipeline: **retriever có lấy đúng bằng chứng không**, **context có đủ không**, **LLM có bám context không**, và **câu trả lời có đúng ý câu hỏi không**.

Một hệ thống RAG có thể thất bại theo nhiều kiểu:

1. Retrieve sai tài liệu.
2. Retrieve đúng chủ đề nhưng thiếu đoạn chứa đáp án.
3. Retrieve đủ nhưng đưa quá nhiều context nhiễu.
4. LLM trả lời vượt quá bằng chứng.
5. Câu trả lời đúng sự thật nhưng không trả lời đúng câu hỏi.

RAGAS giúp tách các lỗi này thành chỉ số để so sánh trước/sau khi thay đổi chunking, embedding model, hybrid search, reranker, prompt hoặc LLM.

**Áp dụng khi nào?**

- Khi xây chatbot hỏi đáp từ tài liệu nội bộ, FAQ, policy, manual, runbook hoặc knowledge base.
- Khi cần benchmark nhiều cấu hình RAG: `chunkSize`, overlap, vector-only vs hybrid, rerank hay không rerank.
- Khi cần regression test trước khi deploy phiên bản RAG mới.
- Khi cần giải thích vì sao chất lượng giảm: do retrieval, generation hay dữ liệu.

**Bài toán giải quyết:** giảm đánh giá cảm tính (*vibe check*), phát hiện lỗi có hệ thống, chọn cấu hình dựa trên số liệu và tạo baseline chất lượng cho production.

**So sánh với đánh giá thủ công:**

| Cách đánh giá | Điểm mạnh | Giới hạn | Chọn khi |
| --- | --- | --- | --- |
| Đọc thủ công vài câu trả lời | Nhanh, hiểu ngữ cảnh sâu. | Không scale, dễ thiên kiến, khó regression. | Giai đoạn đầu hoặc case rủi ro cao cần review. |
| Metric truyền thống như BLEU/ROUGE | Rẻ, deterministic, hợp với so khớp text. | Không đo tốt groundedness hoặc retrieval. | Tóm tắt/dịch máy có reference rõ. |
| RAGAS | Đo riêng retrieval và generation, có thể dùng LLM-as-judge. | Tốn chi phí evaluator, cần dataset và kiểm tra metric. | RAG cần cải tiến có đo lường. |
| User feedback production | Phản ánh người dùng thật. | Nhiễu, chậm, thiếu ground truth. | Theo dõi sau deploy và tìm case mới cho eval set. |

---

### 2. Dữ liệu đầu vào của RAGAS gồm những gì?

**Trả lời:** Một mẫu đánh giá (*evaluation sample*) thường mô tả một lượt hỏi–đáp, gồm các trường quan trọng:

| Trường | Ý nghĩa | Bắt buộc khi nào |
| --- | --- | --- |
| `user_input` | Câu hỏi của người dùng. | Hầu hết metric. |
| `retrieved_contexts` | Các đoạn evidence mà retriever đưa vào prompt. | Metric về retrieval và groundedness. |
| `response` | Câu trả lời do hệ thống tạo ra. | Metric về answer/generation. |
| `reference` | Câu trả lời chuẩn hoặc kỳ vọng. | Metric cần ground truth như context recall/correctness. |
| `reference_contexts` | Các đoạn context chuẩn nên được retrieve. | Khi muốn đo retrieval bằng document/chunk chuẩn. |
| `retrieved_context_ids` / `reference_context_ids` | ID chunk/doc để so sánh không cần LLM. | Khi corpus có nhãn evidence chuẩn. |

Ví dụ tối thiểu cho RAG:

```json
{
  "user_input": "Khách VIP có được hoàn điểm cho đơn hàng bị hoàn tiền toàn phần không?",
  "retrieved_contexts": [
    "Khách VIP được hoàn 5% điểm cho đơn hàng hợp lệ. Không áp dụng với hàng hoàn tiền toàn phần."
  ],
  "response": "Không. Đơn hàng hoàn tiền toàn phần không được áp dụng hoàn điểm VIP.",
  "reference": "Không, hàng hoàn tiền toàn phần không được hoàn điểm."
}
```

**Áp dụng khi nào?** Dùng cấu trúc này khi muốn lưu lại trace của mỗi câu hỏi để có thể chạy lại đánh giá sau khi đổi parser, chunking, retrieval hoặc prompt.

**Bài toán giải quyết:** biến một lượt chat khó debug thành dữ liệu có thể đo, so sánh và tái chạy.

**So sánh với log thông thường:**

| Log thông thường | Evaluation sample |
| --- | --- |
| Thường chỉ có request/response, thiếu context. | Có cả câu hỏi, context, answer, reference/evidence nếu có. |
| Hữu ích cho debug kỹ thuật. | Hữu ích cho đo chất lượng RAG. |
| Dễ lẫn dữ liệu nhạy cảm nếu không kiểm soát. | Cần schema hóa, ẩn PII và lưu đúng phạm vi. |

---

### 3. Context Precision và Context Recall trong RAGAS khác nhau thế nào?

**Trả lời:** **Context Precision** đo các context được retrieve có được xếp hạng tốt và ít nhiễu không. **Context Recall** đo hệ thống có lấy đủ thông tin cần thiết để trả lời không. Hai metric này bổ sung nhau: precision cao nhưng recall thấp nghĩa là lấy ít đoạn sạch nhưng thiếu đáp án; recall cao nhưng precision thấp nghĩa là lấy đủ nhưng nhiều nhiễu.

| Metric | Câu hỏi nó trả lời | Tín hiệu lỗi | Cách cải thiện thường gặp |
| --- | --- | --- | --- |
| Context Precision | Đoạn liên quan có nằm ở top không? | Top-k có nhiều đoạn nhiễu, reranker yếu. | Reranking, threshold, metadata filter, chunking tốt hơn. |
| Context Recall | Các ý quan trọng trong reference có được context hỗ trợ không? | Không lấy được đoạn chứa đáp án, chunk bị cắt sai. | Tăng top-k, hybrid search, query rewrite, embedding/chunking tốt hơn. |

Ví dụ: câu hỏi về “điều kiện hoàn điểm VIP”. Nếu hệ thống retrieve đúng đoạn hoàn điểm nhưng bỏ mất câu “không áp dụng với hàng hoàn tiền toàn phần”, **recall thấp**. Nếu hệ thống retrieve đoạn đúng nhưng đặt sau nhiều đoạn chung chung về loyalty, **precision thấp**.

**Áp dụng khi nào?**

- Dùng Context Recall khi ưu tiên không bỏ sót evidence, ví dụ pháp lý, policy, troubleshooting nhiều điều kiện.
- Dùng Context Precision khi context window hẹp, latency cao hoặc LLM dễ nhiễu bởi đoạn không liên quan.

**Bài toán giải quyết:** biết nên tối ưu “lấy đủ hơn” hay “lọc sạch hơn”.

**So sánh với Recall@k / Precision@k truyền thống:** RAGAS có thể dùng LLM hoặc reference để đánh giá ngữ nghĩa của context; metric truyền thống thường cần nhãn chunk đúng/sai rõ ràng. Nếu đã có `reference_context_ids`, ID-based metric rẻ và ổn định hơn; nếu chưa có nhãn evidence, LLM-based metric linh hoạt hơn nhưng cần kiểm soát evaluator.

---

### 4. Faithfulness và Answer Relevancy đo điều gì?

**Trả lời:** **Faithfulness** đo câu trả lời có nhất quán với `retrieved_contexts` không; một claim trong answer phải được context hỗ trợ. **Answer Relevancy** đo câu trả lời có trả lời đúng ý `user_input` không, không nhất thiết đo factual accuracy.

| Metric | Đo phần nào | Ví dụ lỗi | Ý nghĩa |
| --- | --- | --- | --- |
| Faithfulness | Answer có bám evidence không? | Context nói “không áp dụng hoàn tiền toàn phần”, answer lại nói “mọi đơn VIP đều được hoàn điểm”. | Phát hiện hallucination/unsupported claim. |
| Answer Relevancy | Answer có đúng trọng tâm câu hỏi không? | Người dùng hỏi điều kiện đổi trả, answer giải thích lịch sử chương trình loyalty. | Phát hiện trả lời lan man hoặc lệch intent. |

Một câu trả lời có thể **faithful nhưng không relevant**: context nói đúng nhiều thông tin về chính sách, answer trích đúng nhưng không trả lời câu hỏi cụ thể. Ngược lại, câu trả lời có thể **relevant nhưng không faithful**: trả lời đúng trọng tâm nhưng bịa điều kiện không có trong context.

**Áp dụng khi nào?**

- Faithfulness: bắt buộc với RAG có citation, legal/compliance, CSKH, technical support, tri thức nội bộ.
- Answer Relevancy: cần cho chatbot, search answer, trợ lý nội bộ và bất kỳ hệ thống nào dễ trả lời lan man.

**Bài toán giải quyết:** tách lỗi “không bám nguồn” khỏi lỗi “không đúng ý hỏi”.

**So sánh với Answer Correctness:** Answer Correctness thường cần reference/ground truth để so sánh đúng sai của câu trả lời; Faithfulness chỉ kiểm tra answer có được retrieved context hỗ trợ không. Nếu retrieved context sai nhưng answer bám context đó, faithfulness vẫn có thể cao; vì vậy phải đo cả retrieval metric.

---

### 5. Dùng RAGAS trong vòng lặp cải tiến RAG như thế nào?

**Trả lời:** Cách dùng thực tế là tạo một **evaluation loop** lặp lại mỗi khi thay đổi dữ liệu, retriever, reranker, prompt hoặc LLM.

```text
1. Chọn câu hỏi thật + edge cases
2. Lưu expected answer/evidence khi có thể
3. Chạy pipeline RAG để lấy retrieved_contexts + response
4. Chấm RAGAS metrics
5. So sánh với baseline
6. Phân tích lỗi theo retrieval/generation/data
7. Cải thiện và chạy lại regression
```

**Áp dụng khi nào?** Mỗi lần thay đổi `chunkSize`, parser PDF, embedding model, vector index, hybrid weighting, reranker, prompt template, citation policy hoặc LLM.

**Bài toán giải quyết:** tránh deploy thay đổi làm giảm chất lượng mà không biết; biến việc tối ưu RAG thành thí nghiệm có so sánh.

**So sánh với A/B test production:**

| Offline RAGAS eval | Online A/B test |
| --- | --- |
| Chạy trước deploy trên dataset có kiểm soát. | Chạy với người dùng thật sau deploy. |
| Dễ kiểm tra regression và edge cases. | Đo hành vi thực tế, CSAT, conversion, deflection. |
| Có thể thiếu phân bố câu hỏi thật. | Có rủi ro ảnh hưởng người dùng nếu bản mới kém. |
| Nên là cổng chất lượng trước production. | Nên là vòng phản hồi sau production. |

---

## B. 8 câu hỏi phổ biến mở rộng

### 6. Khi không có ground truth thì RAGAS còn dùng được không?

**Trả lời:** Có, nhưng phải chọn metric phù hợp. Một số metric cần `reference` hoặc `reference_contexts`; một số metric có thể dùng `user_input`, `retrieved_contexts` và `response` để đánh giá theo kiểu không cần reference (*reference-free* hoặc *referenceless*).

**Áp dụng khi nào?** Khi mới triển khai chưa có bộ đáp án chuẩn, hoặc khi muốn đánh giá production traces không có nhãn.

**Bài toán giải quyết:** vẫn có tín hiệu chất lượng ban đầu để lọc case xấu, phát hiện hallucination và tạo backlog cải thiện.

**Lưu ý:** Không nên dựa hoàn toàn vào referenceless eval cho quyết định rủi ro cao. Hãy dần xây bộ `reference` và `reference_contexts` từ câu hỏi thật, review chuyên gia và lỗi production.

**So sánh:**

| Có ground truth | Không có ground truth |
| --- | --- |
| Đo correctness/recall rõ hơn. | Khởi động nhanh hơn. |
| Tốn công gán nhãn. | Phụ thuộc evaluator nhiều hơn. |
| Phù hợp regression quan trọng. | Phù hợp monitoring, triage và khám phá lỗi. |

---

### 7. RAGAS có thay thế human evaluation không?

**Trả lời:** Không. RAGAS giúp tự động hóa phần đo lặp lại, còn human evaluation vẫn quan trọng để kiểm tra tiêu chí nghiệp vụ, ngữ cảnh chuyên môn, tone, rủi ro pháp lý và lỗi tinh vi mà metric có thể bỏ qua.

**Áp dụng khi nào?**

- Dùng RAGAS để chạy hàng trăm/hàng nghìn case thường xuyên.
- Dùng human review cho mẫu đại diện, case điểm thấp, case rủi ro cao và case metric bất đồng.

**Bài toán giải quyết:** giảm tải review thủ công nhưng vẫn giữ kiểm soát chất lượng.

**So sánh:** RAGAS tốt cho regression và so sánh cấu hình; human review tốt cho chuẩn hóa rubric, kiểm tra lỗi khó và tạo ground truth.

---

### 8. RAGAS khác LangSmith, DeepEval, TruLens hoặc framework eval khác như thế nào?

**Trả lời:** RAGAS tập trung mạnh vào metric và quy trình đánh giá RAG/LLM app. LangSmith thiên về tracing, dataset, experiment và observability trong hệ sinh thái LangChain. DeepEval mạnh ở unit testing/evaluation framework với nhiều metric, threshold và CI/CD. TruLens thường được biết tới với cách nhìn theo RAG triad như context relevance, groundedness và answer relevance.

| Công cụ | Mạnh ở | Khi chọn |
| --- | --- | --- |
| RAGAS | Metric RAG, testset, đánh giá retrieval/generation. | Cần chấm RAG độc lập theo metric phổ biến. |
| LangSmith | Trace, dataset, experiment, online/offline eval, quan sát production. | Dùng LangChain/LangGraph hoặc cần nền tảng trace/experiment. |
| DeepEval | Test case, threshold, CI/CD, nhiều metric LLM/RAG/agent. | Muốn viết eval như unit test trong pipeline dev. |
| TruLens | Feedback functions, RAG triad, observability cho LLM app. | Muốn quan sát và chấm từng thành phần app. |
| Human rubric nội bộ | Tiêu chí nghiệp vụ riêng. | Domain rủi ro cao hoặc tiêu chí không có metric sẵn. |

**Áp dụng khi nào?** Có thể kết hợp: dùng LangSmith/observability để lưu traces, dùng RAGAS hoặc DeepEval để chấm metric, dùng human review để hiệu chỉnh rubric.

**Bài toán giải quyết:** chọn đúng lớp công cụ: metric library, test framework, observability platform hay quy trình review.

---

### 9. RAGAS có tạo được test dataset không?

**Trả lời:** Có thể dùng RAGAS cho **testset generation**: sinh câu hỏi đánh giá từ corpus để có bộ kiểm thử ban đầu, đặc biệt khi chưa có nhiều log người dùng. Tuy nhiên, testset sinh tự động chỉ nên là điểm khởi đầu.

**Áp dụng khi nào?**

- Corpus mới, chưa có traffic thật.
- Cần nhanh chóng tạo câu hỏi single-hop, multi-hop hoặc câu hỏi theo persona/use case.
- Cần bao phủ nhiều section tài liệu để kiểm tra retrieval.

**Bài toán giải quyết:** giảm thời gian khởi tạo evaluation dataset.

**Rủi ro:** câu hỏi sinh tự động có thể quá “sạch”, quá gần wording của tài liệu hoặc không đại diện cho cách người dùng thật hỏi. Nên bổ sung câu hỏi thật, câu mơ hồ, câu ngoài phạm vi, câu sai quyền và câu có điều kiện/ngoại lệ.

**So sánh:**

| Synthetic testset | Human-curated testset |
| --- | --- |
| Nhanh, bao phủ rộng tài liệu. | Sát nghiệp vụ và rủi ro thật hơn. |
| Có thể thiếu độ tự nhiên. | Tốn thời gian và chuyên gia. |
| Tốt cho baseline ban đầu. | Tốt cho regression quan trọng. |

---

### 10. Nên đặt ngưỡng pass/fail cho RAGAS metric như thế nào?

**Trả lời:** Không nên dùng một ngưỡng chung cho mọi dự án. Hãy chạy baseline, xem phân phối điểm, review thủ công một mẫu, rồi đặt ngưỡng theo mức rủi ro và mục tiêu.

**Gợi ý thực tế:**

1. Bắt đầu với 50–100 câu hỏi đại diện.
2. Chạy pipeline hiện tại để lấy điểm.
3. Review các case điểm cao/thấp để xem metric có hợp lý không.
4. Đặt ngưỡng riêng cho từng nhóm: FAQ, policy, technical docs, legal, out-of-scope.
5. Dùng ngưỡng như guardrail regression, không phải chân lý tuyệt đối.

**Áp dụng khi nào?** Khi đưa RAGAS vào CI/CD hoặc release gate.

**Bài toán giải quyết:** tránh deploy bản có recall/faithfulness giảm mạnh.

**So sánh:** threshold cứng phù hợp với metric ổn định và rủi ro cao; so sánh tương đối phù hợp khi đang chọn giữa nhiều cấu hình và metric còn nhiễu.

---

### 11. RAGAS có phù hợp với tiếng Việt và dữ liệu đa ngôn ngữ không?

**Trả lời:** Có thể dùng, nhưng cần kiểm tra evaluator LLM, embedding evaluator và prompt đánh giá có hiểu tiếng Việt/domain không. Metric LLM-as-judge không tự đảm bảo đúng với tiếng Việt, thuật ngữ nội bộ hoặc văn bản pháp lý/kỹ thuật.

**Áp dụng khi nào?** Corpus tiếng Việt, song ngữ Việt–Anh, tài liệu nội bộ nhiều thuật ngữ, hoặc người dùng hỏi tiếng Việt về tài liệu tiếng Anh.

**Bài toán giải quyết:** đo được chất lượng RAG trong đúng ngôn ngữ người dùng thật.

**Khuyến nghị:**

- Tạo eval set có tiếng Việt có dấu, không dấu, pha tiếng Anh, từ viết tắt nội bộ.
- Dùng evaluator LLM đủ mạnh với tiếng Việt.
- Review thủ công một mẫu để hiệu chỉnh prompt/rubric.
- Tách metric theo ngôn ngữ và loại tài liệu.

**So sánh:** Nếu dùng evaluator tiếng Anh kém hiểu tiếng Việt, điểm có thể không phản ánh chất lượng thật. Với câu hỏi tiếng Việt nhưng source tiếng Anh, cần đánh giá cả retrieval cross-lingual lẫn answer faithfulness.

---

### 12. Những lỗi thường gặp khi dùng RAGAS là gì?

**Trả lời:** Lỗi lớn nhất là xem điểm RAGAS như chân lý tuyệt đối. Metric chỉ là tín hiệu; nó phụ thuộc vào evaluator model, prompt đánh giá, dữ liệu đầu vào và chất lượng reference.

**Các lỗi phổ biến:**

1. Chỉ đo answer mà không lưu `retrieved_contexts`, nên không biết retrieval sai hay generation sai.
2. Dùng dataset quá dễ, chỉ có câu hỏi nằm nguyên văn trong tài liệu.
3. Không có câu ngoài phạm vi để kiểm tra refusal.
4. Không tách điểm theo loại câu hỏi, ngôn ngữ, product, version hoặc tenant.
5. Đổi evaluator model nhưng so sánh điểm như cùng một thước đo.
6. Bỏ qua chi phí và độ trễ của evaluation.
7. Không review sample điểm thấp/cao để kiểm tra metric có hợp lý không.

**Áp dụng khi nào?** Luôn kiểm tra các lỗi này trước khi dùng RAGAS làm release gate.

**Bài toán giải quyết:** tránh tối ưu nhầm metric, làm hệ thống trông tốt trên dashboard nhưng kém trong thực tế.

**So sánh:** Metric tự động phù hợp để phát hiện xu hướng; review thủ công phù hợp để xác nhận nguyên nhân và chỉnh tiêu chí.

---

### 13. Tích hợp RAGAS vào Java Spring và Next.js như thế nào?

**Trả lời:** Nên tách phần **RAG serving** khỏi phần **evaluation job**. Java Spring phục vụ API production và ghi trace; job Python hoặc service riêng chạy RAGAS định kỳ hoặc trong CI/CD trên dataset đã chọn. Next.js hiển thị câu trả lời, citation, feedback và màn hình so sánh nếu cần.

```text
Next.js UI
  → Spring RAG API
    → Auth + retrieval + LLM answer
    → Log trace: user_input, retrieved_contexts IDs, response, metadata

Evaluation job
  → Load eval dataset
  → Replay questions against RAG API or saved traces
  → Run RAGAS metrics
  → Store results by app version / index version / prompt version
```

**Áp dụng khi nào?** Khi đội backend dùng Java Spring nhưng muốn tận dụng hệ sinh thái Python cho RAGAS và LLM evaluation.

**Bài toán giải quyết:** không ép production service chạy evaluator nặng; vẫn có regression test độc lập.

**Lưu ý vận hành:** ẩn PII trước khi lưu eval dataset, version hóa prompt/index/model, và không đưa dữ liệu trái quyền vào evaluation job.

**So sánh:** Chạy RAGAS trực tiếp trong request path làm tăng latency và chi phí; chạy offline/asynchronous phù hợp hơn cho regression và monitoring.

---

## Checklist dùng RAGAS cho RAG production

| Nhóm | Câu hỏi kiểm tra |
| --- | --- |
| Dataset | Có câu hỏi thật, edge case, out-of-scope, sai quyền, nhiều version không? |
| Schema | Có lưu `user_input`, `retrieved_contexts`, `response`, `reference` hoặc evidence chuẩn không? |
| Retrieval | Có đo Context Precision, Context Recall hoặc ID-based retrieval metric không? |
| Generation | Có đo Faithfulness, Answer Relevancy và answer correctness khi có reference không? |
| Segmentation | Có tách điểm theo language, tenant, product, document type và risk level không? |
| Regression | Có so sánh theo app version, prompt version, index version, embedding model không? |
| Human review | Có review mẫu để kiểm tra evaluator không? |
| Security | Có loại/ẩn PII và enforce ACL trước khi lưu trace/eval không? |

## Tóm tắt để nhớ

> **RAGAS không phải là “điểm số cho chatbot”, mà là bộ thước đo để bóc tách lỗi RAG.**  
> Retrieval sai thì sửa data/chunking/search; answer không faithful thì sửa prompt/context policy/model; answer không relevant thì sửa query understanding và response policy.

Một vòng lặp RAGAS tốt cần ba thứ:

1. **Dataset đủ đại diện.**
2. **Metric đúng câu hỏi cần đo.**
3. **Quy trình review và regression rõ ràng.**

## Nguồn tham khảo

- [Ragas documentation](https://docs.ragas.io/en/stable/)
- [Ragas available metrics](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/)
- [RAGAS: Automated Evaluation of Retrieval Augmented Generation](https://arxiv.org/abs/2309.15217)
- [LangSmith evaluation concepts](https://docs.smith.langchain.com/evaluation)
- [DeepEval metrics introduction](https://deepeval.com/docs/metrics-introduction)

## Liên kết liên quan

- [RAG](../README.md)
- [Retrieval](../retrieval/README.md)
- [Recall trong RAG](../recall/README.md)
- [Chunking](../chunking/README.md)
- [Embedding](../embedding/README.md)
- [Các trường hợp ứng dụng RAG](../use-cases/README.md)
- [LLM](../../README.md)
- [AI](../../../README.md)
- [Knowledge Tree](../../../../README.md)
