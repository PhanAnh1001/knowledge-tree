# Recall trong RAG

> [Knowledge Tree](../../../../README.md) / [AI](../../../README.md) / [LLM](../../README.md) / [RAG](../README.md) / **Recall**

**Recall** trong RAG (*Retrieval-Augmented Generation*) là thước đo cho biết hệ thống truy xuất tìm được bao nhiêu bằng chứng đúng trong tổng số bằng chứng đúng cần có. Nếu retrieval không lấy được evidence cần thiết, LLM khó trả lời đúng dù prompt/model phía sau tốt.

```text
Recall = evidence đúng đã retrieve / tổng evidence đúng cần retrieve
Recall@k = tỷ lệ evidence đúng xuất hiện trong top-k kết quả đầu tiên
```

## Điều hướng

- **Node cha:** [RAG](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** [Retrieval](../retrieval/README.md)
- **Khái niệm sau trong cùng nhánh:** [Chỉ mục pgvector](../pgvector-index/README.md)

---

## A. 5 câu hỏi cốt lõi

### 1. Recall là gì trong retrieval và RAG?

**Trả lời:** Recall là tỷ lệ thông tin liên quan mà hệ thống tìm được so với toàn bộ thông tin liên quan đáng lẽ cần tìm. Trong RAG, recall thường đo trên chunk, passage, section hoặc document chứa bằng chứng đúng cho một câu hỏi.

**Áp dụng khi nào?** Khi đánh giá search/retrieval cho chatbot chính sách, FAQ, tài liệu kỹ thuật, CSKH, catalogue hoặc knowledge base nội bộ.

**Bài toán giải quyết:** Biết hệ thống có bỏ sót bằng chứng quan trọng không. Nếu evidence không được retrieve, generation dễ suy đoán hoặc trả lời thiếu điều kiện.

**So sánh:** Chỉ xem câu trả lời cuối giúp biết output đúng/sai nhưng khó debug; đo recall giúp tách lỗi retrieval khỏi lỗi prompt/LLM.

### 2. Vì sao recall quan trọng ở bước đầu của RAG?

**Trả lời:** Ở bước lấy candidate, ưu tiên đầu tiên là không bỏ sót evidence đúng. Precision thấp còn có thể cải thiện bằng reranking, threshold, MMR hoặc context selection; recall thấp thì evidence đúng không có mặt để các bước sau chọn lại.

```text
Retrieve nhiều candidate để tăng recall → rerank để tăng precision → chọn context → generate
```

**Áp dụng khi nào?** Khi tài liệu lớn, nhiều phiên bản, nhiều cách diễn đạt, hoặc câu hỏi cần nhiều điều kiện nghiệp vụ.

**Bài toán giải quyết:** Giảm lỗi “trả lời thiếu điều kiện”, ví dụ chỉ lấy điều khoản “hoàn 5%” nhưng bỏ sót ngoại lệ “không áp dụng với đơn hoàn tiền toàn phần”.

**So sánh:** Recall phù hợp cho candidate set; precision phù hợp cho context cuối; groundedness phù hợp cho câu trả lời cuối.

### 3. Recall@k là gì và đo như thế nào?

**Trả lời:** **Recall@k** đo xem trong `k` kết quả đầu tiên có bao nhiêu evidence đúng. Thường đo `Recall@5`, `Recall@10`, `Recall@20`, `Recall@50` để biết top-k nào đủ tốt trước khi rerank hoặc đưa vào context.

**Cách đo:** tạo bộ câu hỏi thật hoặc gần thật, gán `relevantChunkIds`, chạy retriever lấy top-k, rồi so khớp IDs đã retrieve với IDs đúng. Ví dụ có 3 chunk đúng nhưng top-10 chỉ lấy được 2 chunk thì recall là `2/3`.

**Áp dụng khi nào?** Khi benchmark embedding model, chunking strategy, hybrid search, query rewrite, reranking hoặc vector index.

**Bài toán giải quyết:** Có số đo định lượng trước/sau thay đổi, tránh đánh giá cảm tính “search tốt hơn”.

**So sánh:** `Hit Rate@k` chỉ hỏi “có ít nhất một evidence đúng không”; `Recall@k` hỏi “đã lấy được bao nhiêu evidence đúng”.

### 4. Làm sao cải thiện recall trong RAG?

**Trả lời:** Không chỉ tăng `top_k`; cần kiểm tra cả parse, chunking, metadata, embedding, query rewrite, hybrid search và filter. Tăng `top_k` giúp ít bỏ sót hơn nhưng tăng latency/cost và cần reranking để giảm nhiễu.

**Áp dụng khi nào?** Khi log hoặc evaluation cho thấy evidence đúng không nằm trong candidate set.

**Bài toán giải quyết:** Tăng khả năng tìm đúng tài liệu với câu hỏi tiếng Việt, mã/SKU, policy nhiều điều kiện, tài liệu nhiều version hoặc dữ liệu đa ngôn ngữ.

**So sánh nhanh:** Hybrid search tốt hơn vector-only khi có mã lỗi/tên riêng; query rewrite tốt cho câu hỏi hội thoại; parent-child retrieval tốt khi chunk nhỏ dễ tìm nhưng cần parent để đủ ngữ cảnh; metadata filter đúng giúp recall đúng phạm vi quyền.

### 5. Recall khác gì precision, MRR và nDCG?

**Trả lời:** Recall đo mức độ không bỏ sót evidence đúng. Precision đo mức độ ít nhiễu. MRR (*Mean Reciprocal Rank*) đo kết quả đúng đầu tiên xuất hiện sớm cỡ nào. nDCG (*Normalized Discounted Cumulative Gain*) đo thứ hạng có ưu tiên kết quả quan trọng hơn không.

| Metric | Câu hỏi trả lời | Dùng khi |
| --- | --- | --- |
| Recall@k | Có tìm đủ evidence đúng không? | Debug bỏ sót retrieval. |
| Precision@k | Top-k có nhiều kết quả đúng không? | Giảm nhiễu context. |
| MRR | Evidence đúng đầu tiên đứng thứ mấy? | Câu hỏi cần một nguồn chính. |
| nDCG | Kết quả quan trọng có xếp cao không? | Có nhiều mức độ liên quan. |

**Bài toán giải quyết:** Tách lỗi “không tìm thấy”, “tìm thấy nhưng xếp thấp”, “context nhiễu” và “LLM không dùng đúng evidence”.

---

## B. 8 câu hỏi phổ biến mở rộng

### 6. Nên chọn `top_k` bao nhiêu?

**Trả lời:** Không có số cố định. FAQ đơn giản có thể bắt đầu `top_k=5–10`; production thường retrieve `20–50` candidate rồi rerank; dữ liệu lớn hoặc query khó có thể thử `50–100` nhưng phải đo p95 latency và cost.

**Áp dụng:** Tăng `top_k` khi `Recall@k` cho thấy evidence đúng thường nằm ngoài top hiện tại. **So sánh:** top-k thấp nhanh nhưng dễ bỏ sót; top-k cao recall tốt hơn nhưng nhiễu và tốn hơn.

### 7. Làm sao tạo bộ dữ liệu đánh giá recall?

**Trả lời:** Mỗi record nên có `question`, `relevantChunkIds` hoặc `relevantDocumentIds`, filter bắt buộc như tenant/version/region/status và ghi chú nghiệp vụ. Nguồn câu hỏi tốt nhất là log thật đã ẩn dữ liệu nhạy cảm, câu hỏi do chuyên gia tạo và câu hỏi sinh từ tài liệu để phủ nhiều section.

**Áp dụng:** Trước production, trước khi đổi embedding/chunking/index, hoặc khi có feedback sai. **Bài toán:** tạo regression test lặp lại thay vì kiểm thử thủ công.

### 8. Vì sao recall cao nhưng câu trả lời vẫn sai?

**Trả lời:** Recall cao chỉ chứng minh evidence đúng có trong candidate set, chưa chắc evidence đó vào prompt hoặc được LLM dùng đúng. Lỗi có thể nằm ở reranking, context assembly, prompt, citation mapping hoặc generation.

**Áp dụng:** Khi dashboard retrieval tốt nhưng người dùng vẫn báo sai. **So sánh:** evidence không có trong top-k là lỗi recall; evidence có nhưng không vào prompt là lỗi rerank/context; evidence vào prompt nhưng answer sai là lỗi prompt/generation.

### 9. Filter, ACL và metadata ảnh hưởng recall thế nào?

**Trả lời:** Filter và ACL (*Access Control List*) giúp bảo mật nhưng có thể làm recall giảm nếu metadata sai. Các filter như `tenantId`, `allowedRoles`, `documentStatus`, `effectiveFrom`, `region`, `product` nên là ràng buộc cứng trước retrieval.

**Áp dụng:** Hệ thống nhiều tenant, role, version hoặc vùng áp dụng. **Bài toán:** không lộ dữ liệu nhưng vẫn tìm đủ evidence trong phạm vi được phép. **So sánh:** không filter thì nguy cơ rò rỉ; filter sai thì recall có thể bằng 0.

### 10. Query rewrite, multi-query và HyDE có giúp tăng recall không?

**Trả lời:** Có. Query rewrite làm rõ câu hỏi theo hội thoại; multi-query tạo nhiều biến thể truy vấn; HyDE (*Hypothetical Document Embeddings*) tạo đoạn giả định để tìm tài liệu gần nghĩa. Chúng hữu ích khi query ngắn, mơ hồ hoặc khác từ vựng tài liệu.

**Áp dụng:** Câu hỏi như “thế còn gói cao cấp?”, dữ liệu song ngữ hoặc người dùng dùng từ đời thường. **So sánh:** rewrite rẻ hơn nhưng có thể sai ý; multi-query phủ rộng hơn nhưng tốn hơn; HyDE mạnh về semantic nhưng có thể kéo lệch hướng.

### 11. Chunking ảnh hưởng recall ra sao?

**Trả lời:** Retriever tìm trên đơn vị chunk, nên chunk quá lớn làm vector loãng nghĩa, chunk quá nhỏ mất điều kiện quan trọng. Chunk tốt là evidence có thể đứng độc lập, có heading/section/page/source rõ.

**Áp dụng:** Policy, legal, manual kỹ thuật, FAQ dài, bảng, transcript hoặc Markdown nhiều heading. **So sánh:** fixed-size dễ làm nhưng hay cắt ý; section-aware giữ ngữ cảnh tốt; parent-child cân bằng giữa tìm chi tiết và trả lời đủ ngữ cảnh.

### 12. Recall với tiếng Việt hoặc đa ngôn ngữ cần chú ý gì?

**Trả lời:** Recall dễ giảm nếu embedding model yếu với tiếng Việt, dữ liệu trộn Việt–Anh, có nhiều thuật ngữ viết tắt hoặc người dùng dùng cách nói đời thường. Hybrid search giúp bắt mã lỗi, SKU, tên riêng; multilingual embedding hoặc query translation giúp dữ liệu song ngữ.

**Áp dụng:** CSKH, chính sách nội bộ, tài liệu kỹ thuật, legal hoặc catalogue tiếng Việt. **Bài toán:** giảm bỏ sót do khác ngôn ngữ, khác thuật ngữ hoặc cách diễn đạt.

### 13. Theo dõi recall trên production thế nào?

**Trả lời:** Không thể biết recall thật cho mọi query production vì thiếu ground truth, nên cần kết hợp offline evaluation, online trace, feedback và regression test. Nên log query đã normalize, filters, candidate IDs, scores, index version, reranker version, latency và citation được dùng.

**Áp dụng:** Khi dữ liệu cập nhật thường xuyên hoặc team thay đổi parser/chunking/embedding/index. **Bài toán:** phát hiện regression sau re-index hoặc đổi model. **So sánh:** offline đo chính xác hơn vì có nhãn; online gần hành vi người dùng thật hơn nhưng khó biết ground truth đầy đủ.

---

## Checklist áp dụng recall cho RAG production

1. Có dataset gồm query thường gặp, query khó, query đa ngôn ngữ và query cần nhiều evidence.
2. Mỗi câu hỏi có `relevantChunkIds`/`relevantDocumentIds` và filter bắt buộc.
3. Đo `Recall@5/10/20/50` trước và sau mỗi thay đổi lớn.
4. Tách log retrieval, reranking, context assembly và generation.
5. Kiểm tra ACL/version/filter bằng test riêng, không gộp vào semantic similarity.
6. So sánh keyword, vector và hybrid search trên cùng dataset.
7. Theo dõi recall theo nhóm: tiếng Việt, mã lỗi/SKU, policy nhiều điều kiện, tài liệu cũ/mới.
8. Chặn release nếu recall giảm vượt ngưỡng ở nhóm câu hỏi quan trọng.

## Gợi ý triển khai với Java Spring và Next.js

| Thành phần | Vai trò | Lưu ý |
| --- | --- | --- |
| Next.js | UI hỏi đáp, citation, feedback. | Không quyết định ACL ở client. |
| Java Spring | Auth, orchestration RAG, logging. | Log requestId, filter, top-k, candidate IDs, latency. |
| Ingestion worker | Parse, chunk, embed, index. | Idempotent theo content hash và document version. |
| Vector/search engine | Keyword, vector, hybrid retrieval. | Enforce filter/ACL ở query layer. |
| Evaluation job | Chạy benchmark recall định kỳ. | Lưu report theo index/model/chunking version. |

## Tóm tắt để nhớ

> **Recall trả lời câu hỏi: “Retriever có tìm đủ bằng chứng đúng không?”**

Thứ tự tối ưu RAG nên là: **recall** để tìm đủ evidence, **precision/reranking** để giảm nhiễu, **context assembly** để giữ đủ ngữ cảnh, **grounded generation** để trả lời bám nguồn, và **evaluation/monitoring** để phát hiện regression.

## Liên kết liên quan

- [Retrieval trong RAG](../retrieval/README.md)
- [Chunking trong RAG](../chunking/README.md)
- [Embedding trong RAG](../embedding/README.md)
- [RAG](../README.md)
- [LLM](../../README.md)
- [AI](../../../README.md)
- [Knowledge Tree](../../../../README.md)
