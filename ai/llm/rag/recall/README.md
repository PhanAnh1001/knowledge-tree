# Recall trong RAG

> [Knowledge Tree](../../../../README.md) / [AI](../../../README.md) / [LLM](../../README.md) / [RAG](../README.md) / **Recall**

**Recall** trong ngữ cảnh RAG (*Retrieval-Augmented Generation*) là thước đo cho biết hệ thống truy xuất có tìm được bao nhiêu phần bằng chứng đúng trong tổng số bằng chứng đúng cần có. Nếu retrieval không lấy được evidence cần thiết, LLM gần như không thể trả lời đúng một cách có căn cứ, dù prompt hay model phía sau rất mạnh.

```text
Recall = số evidence đúng đã retrieve / tổng số evidence đúng cần retrieve
Recall@k = tỷ lệ evidence đúng xuất hiện trong top-k kết quả đầu tiên
```

Recall thường được tối ưu ở giai đoạn **retrieve candidate** trước, sau đó mới dùng **precision**, **reranking**, **threshold** và **context selection** để giảm nhiễu trước khi đưa vào LLM.

## Điều hướng

- **Node cha:** [RAG](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** [Retrieval](../retrieval/README.md)
- **Khái niệm sau trong cùng nhánh:** [Chỉ mục pgvector](../pgvector-index/README.md)

---

## A. 5 câu hỏi cốt lõi

### 1. Recall là gì trong retrieval và RAG?

**Trả lời:** Recall là tỷ lệ thông tin liên quan mà hệ thống tìm được so với toàn bộ thông tin liên quan đáng lẽ cần tìm. Trong RAG, recall thường được đo trên các chunk, passage, section hoặc document chứa bằng chứng đúng cho một câu hỏi.

Ví dụ, câu hỏi “Khách VIP được hoàn tiền trong trường hợp nào?” có 3 section chính sách liên quan. Nếu top-10 kết quả retrieve được 2 section đúng, thì recall trên tập bằng chứng này là `2/3`.

**Áp dụng khi nào?** Dùng recall khi đánh giá search/retrieval cho chatbot chính sách, FAQ, tài liệu kỹ thuật, legal/compliance, CSKH, catalogue hoặc knowledge base nội bộ.

**Bài toán giải quyết:** Recall trả lời câu hỏi “hệ thống có bỏ sót bằng chứng quan trọng không?”. Đây là lỗi nền tảng trong RAG: nếu evidence không được retrieve, generation khó có thể đúng và dễ sinh câu trả lời suy đoán.

**So sánh với cách nhìn thông thường:**

| Cách nhìn | Ý nghĩa | Rủi ro |
| --- | --- | --- |
| Chỉ xem câu trả lời cuối | Biết người dùng có thấy câu trả lời đúng không. | Khó biết sai do retrieval, prompt hay LLM. |
| Đo recall retrieval | Biết evidence đúng có vào candidate set không. | Chưa đảm bảo câu trả lời cuối đúng. |
| Đo answer correctness | Biết output có đúng nghiệp vụ không. | Nếu không tách retrieval, khó debug nguyên nhân. |

---

### 2. Vì sao recall quan trọng hơn precision ở bước đầu của RAG?

**Trả lời:** Ở bước retrieve candidate, ưu tiên đầu tiên là **không bỏ sót bằng chứng đúng**. Precision thấp có thể được cải thiện bằng reranking, MMR, threshold, deduplication hoặc context selection; nhưng nếu recall thấp, bằng chứng đúng không có mặt để các bước sau chọn lại.

Một pipeline phổ biến:

```text
Retrieve nhiều candidate để tăng recall
→ Rerank / filter / select để tăng precision
→ Đưa context ngắn, đúng, có citation vào LLM
```

**Áp dụng khi nào?** Áp dụng khi xây RAG production có nhiều tài liệu, câu hỏi đa dạng, người dùng diễn đạt khác với văn bản gốc, hoặc cần tránh bỏ sót điều kiện nghiệp vụ quan trọng.

**Bài toán giải quyết:** Giảm lỗi “trả lời thiếu điều kiện” như chỉ lấy section “hoàn tiền 5%” nhưng bỏ sót section “không áp dụng cho hàng hoàn tiền toàn phần”.

**So sánh recall và precision trong pipeline:**

| Giai đoạn | Ưu tiên | Lý do |
| --- | --- | --- |
| Candidate retrieval | Recall cao | Cần đưa đủ evidence đúng vào tập ứng viên. |
| Reranking | Precision cao hơn | Sắp xếp lại để evidence quan trọng nằm trên. |
| Context assembly | Precision + coverage | Chỉ đưa đoạn cần thiết, tránh nhiễu và quá token. |
| Generation | Groundedness | Câu trả lời phải bám evidence đã chọn. |

---

### 3. Recall@k là gì và đo như thế nào?

**Trả lời:** **Recall@k** đo xem trong `k` kết quả đầu tiên có bao nhiêu evidence đúng được tìm thấy. `k` có thể là 5, 10, 20, 50 hoặc 100 tùy pipeline. Với RAG, thường đo nhiều mức như `Recall@5`, `Recall@10`, `Recall@20`, vì top-k nhỏ ảnh hưởng trực tiếp đến context đưa vào LLM, còn top-k lớn phản ánh khả năng candidate retrieval trước reranking.

Cách đo cơ bản:

1. Tạo bộ câu hỏi thật hoặc gần thật.
2. Gán nhãn `relevant_chunk_ids` hoặc `relevant_document_ids` cho từng câu hỏi.
3. Chạy retriever để lấy top-k kết quả.
4. So khớp kết quả retrieve với nhãn đúng.
5. Tính recall theo từng câu hỏi và trung bình toàn bộ dataset.

Ví dụ:

```json
{
  "question": "Điều kiện đổi điểm cho khách VIP là gì?",
  "relevantChunkIds": ["policy-v3:s4:c1", "policy-v3:s4:c2", "policy-v3:s5:c1"],
  "retrievedTop10": ["policy-v3:s4:c1", "faq-v2:c7", "policy-v3:s4:c2"]
}
```

Trong ví dụ này, recall trên top-10 là `2/3` vì tìm được 2 trong 3 chunk đúng.

**Áp dụng khi nào?** Dùng khi benchmark embedding model, chunking strategy, hybrid search, reranking, query rewrite hoặc thay đổi vector index.

**Bài toán giải quyết:** Có số đo định lượng trước/sau khi thay đổi hệ thống, tránh cảm tính “search có vẻ tốt hơn”.

**So sánh với Hit Rate@k:**

| Metric | Cách hiểu | Phù hợp khi |
| --- | --- | --- |
| Hit Rate@k | Trong top-k có ít nhất một evidence đúng không. | Câu hỏi chỉ cần một đoạn đúng. |
| Recall@k | Tìm được bao nhiêu evidence đúng trong toàn bộ evidence cần có. | Câu hỏi cần nhiều điều kiện, nhiều section hoặc nhiều nguồn. |

---

### 4. Làm sao cải thiện recall trong RAG?

**Trả lời:** Cải thiện recall không chỉ là tăng `top_k`. Cần kiểm tra toàn bộ chuỗi: dữ liệu nguồn, parse, chunking, metadata, embedding, query rewrite, retrieval strategy và filter.

Các cách phổ biến:

| Cách cải thiện | Khi dùng | Rủi ro cần kiểm soát |
| --- | --- | --- |
| Tăng `top_k` | Evidence đúng nằm thấp hơn top hiện tại. | Tăng latency, cost rerank và nhiễu context. |
| Hybrid search | Cần bắt cả từ khóa chính xác và nghĩa gần. | Cần tuning cách merge score. |
| Query rewrite / multi-query | Câu hỏi hội thoại, thiếu ngữ cảnh hoặc nhiều cách diễn đạt. | Có thể tạo query lệch ý nếu rewrite quá mạnh. |
| Chunking theo section | Tài liệu có heading, điều kiện, bảng, policy. | Chunk quá lớn làm giảm precision; quá nhỏ mất ngữ cảnh. |
| Parent-child retrieval | Chunk nhỏ dễ tìm, parent chứa đủ ngữ cảnh. | Cần thiết kế mapping parent/child rõ ràng. |
| Metadata filter đúng | Cần lọc version, tenant, role, product, region. | Filter sai có thể làm recall bằng 0. |
| Đổi embedding model | Ngôn ngữ/domain hiện tại không phù hợp. | Phải re-embed và benchmark lại toàn bộ. |

**Áp dụng khi nào?** Dùng khi log cho thấy câu trả lời sai vì evidence đúng không nằm trong candidate set, hoặc khi dataset đánh giá có `Recall@k` thấp ở nhóm câu hỏi quan trọng.

**Bài toán giải quyết:** Giảm bỏ sót tài liệu đúng, đặc biệt với câu hỏi tiếng Việt, câu hỏi có mã/số hiệu, câu hỏi cần nhiều điều kiện hoặc tài liệu nhiều phiên bản.

**So sánh với chỉ tăng context window:** Tăng context window giúp chứa nhiều đoạn hơn sau retrieval, nhưng không tự tìm được evidence bị bỏ sót. Recall phải được xử lý ở search layer trước.

---

### 5. Recall khác gì precision, MRR và nDCG?

**Trả lời:** Recall đo mức độ **không bỏ sót** evidence đúng. Precision đo mức độ **ít nhiễu** trong kết quả. MRR và nDCG quan tâm nhiều hơn đến **thứ hạng** của kết quả đúng.

| Metric | Câu hỏi trả lời | Dùng để quyết định |
| --- | --- | --- |
| Recall@k | Có tìm đủ evidence đúng trong top-k không? | Retriever có bỏ sót không. |
| Precision@k | Trong top-k có bao nhiêu kết quả thật sự đúng? | Context có bị nhiễu không. |
| MRR (*Mean Reciprocal Rank*) | Kết quả đúng đầu tiên xuất hiện sớm cỡ nào? | Câu hỏi cần một đáp án/evidence chính. |
| nDCG (*Normalized Discounted Cumulative Gain*) | Kết quả đúng và quan trọng có được xếp cao không? | Có nhiều mức độ liên quan khác nhau. |
| Answer correctness | Câu trả lời cuối có đúng không? | Chất lượng end-to-end. |
| Groundedness | Câu trả lời có bám evidence không? | Giảm hallucination và tăng audit. |

**Áp dụng khi nào?** Dùng recall cùng precision/MRR/nDCG khi cần benchmark retrieval một cách đầy đủ, đặc biệt trước khi thay embedding model, vector DB, chunking hoặc reranking.

**Bài toán giải quyết:** Tách lỗi “không tìm thấy evidence”, “tìm thấy nhưng xếp thấp”, “tìm thấy nhưng context nhiễu” và “LLM không dùng đúng evidence”.

---

## B. 8 câu hỏi phổ biến mở rộng

### 6. Nên chọn `top_k` bao nhiêu để recall đủ tốt?

**Trả lời:** Không có `top_k` cố định cho mọi RAG. `top_k` nhỏ như 5–10 phù hợp FAQ đơn giản; 20–50 thường dùng cho candidate retrieval trước reranking; 50–100 có thể dùng khi tài liệu lớn, nhiều phiên bản hoặc query khó, nhưng cần kiểm soát latency và chi phí.

**Áp dụng khi nào?** Tăng `top_k` khi `Recall@k` cho thấy evidence đúng thường nằm ngoài top hiện tại.

**Bài toán giải quyết:** Giảm bỏ sót evidence do ranking ban đầu chưa tốt.

**So sánh:**

| Top-k thấp | Top-k cao |
| --- | --- |
| Nhanh, ít tốn compute. | Recall tốt hơn nếu retriever còn yếu. |
| Dễ bỏ sót evidence đúng. | Cần rerank/select để tránh nhiễu context. |
| Phù hợp FAQ đơn giản. | Phù hợp tài liệu lớn, query phức tạp. |

Nguyên tắc thực tế: đo `Recall@5/10/20/50`, chọn mức tăng recall đáng kể nhưng latency/cost còn chấp nhận được.

---

### 7. Làm sao tạo bộ dữ liệu đánh giá recall?

**Trả lời:** Bộ đánh giá recall cần câu hỏi đại diện cho cách người dùng thật hỏi và nhãn evidence đúng. Không nên chỉ tạo câu hỏi từ heading, vì như vậy benchmark quá dễ và không phản ánh query tự nhiên.

Một record đánh giá nên có:

```json
{
  "questionId": "q-001",
  "question": "Khách VIP có được hoàn điểm khi hủy đơn không?",
  "expectedAnswer": "...",
  "relevantChunkIds": ["policy-v3:s4:c2", "policy-v3:s7:c1"],
  "requiredFilters": {
    "region": "VN",
    "status": "active",
    "customerTier": "VIP"
  },
  "notes": "Câu hỏi cần kết hợp điều kiện hoàn điểm và điều kiện hủy đơn."
}
```

**Áp dụng khi nào?** Khi hệ thống bắt đầu production, khi thay đổi chunking/embedding/index, hoặc khi phát hiện regression từ feedback người dùng.

**Bài toán giải quyết:** Có chuẩn kiểm thử lặp lại, giúp biết thay đổi nào làm recall tăng/giảm.

**So sánh nguồn câu hỏi:**

| Nguồn câu hỏi | Điểm mạnh | Giới hạn |
| --- | --- | --- |
| Câu hỏi thật từ log | Gần thực tế nhất. | Cần xử lý dữ liệu nhạy cảm. |
| Câu hỏi do chuyên gia tạo | Chính xác nghiệp vụ. | Có thể thiếu đa dạng cách diễn đạt. |
| Câu hỏi sinh tự động từ tài liệu | Tạo nhanh, phủ nhiều section. | Dễ quá giống văn bản gốc. |

---

### 8. Vì sao recall cao nhưng câu trả lời RAG vẫn sai?

**Trả lời:** Recall cao chỉ nói rằng evidence đúng đã xuất hiện trong candidate set, không đảm bảo evidence được chọn vào context cuối hoặc được LLM dùng đúng. Sai sót có thể nằm ở reranking, context assembly, prompt, citation policy hoặc generation.

Các nguyên nhân thường gặp:

1. Evidence đúng nằm trong top-50 nhưng bị reranker đẩy xuống thấp.
2. Context selection chọn đoạn ngắn thiếu điều kiện ngoại lệ.
3. Có evidence mâu thuẫn giữa nhiều version nhưng filter version sai.
4. Prompt không buộc model trả lời theo evidence.
5. LLM tổng quát hóa quá mức hoặc bỏ qua citation.

**Áp dụng khi nào?** Dùng khi retrieval dashboard tốt nhưng người dùng vẫn báo câu trả lời sai.

**Bài toán giải quyết:** Tránh kết luận nhầm rằng “retrieval tốt thì RAG tốt”.

**So sánh lỗi:**

| Dấu hiệu | Khả năng lỗi |
| --- | --- |
| Evidence đúng không có trong top-k | Recall/retrieval lỗi. |
| Evidence đúng có nhưng không vào prompt | Rerank/context selection lỗi. |
| Evidence đúng vào prompt nhưng answer sai | Prompt/generation/guardrail lỗi. |
| Citation sai nguồn | Citation mapping hoặc output validation lỗi. |

---

### 9. Filter, ACL và metadata ảnh hưởng recall như thế nào?

**Trả lời:** Filter và ACL (*Access Control List*) vừa giúp bảo mật vừa có thể làm recall giảm mạnh nếu metadata sai. Trong RAG doanh nghiệp, các filter như `tenantId`, `allowedRoles`, `documentStatus`, `effectiveFrom`, `region`, `product` là ràng buộc bắt buộc; nhưng nếu gắn sai metadata, evidence đúng sẽ bị loại trước khi search.

**Áp dụng khi nào?** Khi dữ liệu nhiều tenant, nhiều role, nhiều version, nhiều vùng áp dụng hoặc có tài liệu hết hiệu lực.

**Bài toán giải quyết:** Đảm bảo hệ thống không retrieve tài liệu người dùng không có quyền, nhưng vẫn không bỏ sót tài liệu đúng trong phạm vi được phép.

**So sánh:**

| Không filter | Filter đúng | Filter sai |
| --- | --- | --- |
| Recall có thể cao nhưng rủi ro lộ dữ liệu. | Recall đúng trong phạm vi quyền. | Recall thấp giả tạo, có thể bằng 0. |
| Dễ đưa policy hết hiệu lực vào answer. | Giữ đúng version/status. | Khó debug nếu không log filter. |

Nguyên tắc: log `query`, `filters`, `candidate count`, `top candidates`, `document version` và `ACL decision` để debug recall.

---

### 10. Multi-query, query rewrite và HyDE có giúp tăng recall không?

**Trả lời:** Có, nhưng cần kiểm soát. **Query rewrite** viết lại câu hỏi cho rõ ngữ cảnh; **multi-query** tạo nhiều biến thể truy vấn; **HyDE (Hypothetical Document Embeddings)** tạo một đoạn trả lời giả định rồi embed đoạn đó để tìm tài liệu gần nghĩa. Các kỹ thuật này giúp tăng recall khi người dùng hỏi mơ hồ hoặc dùng từ khác với tài liệu.

**Áp dụng khi nào?**

- Câu hỏi hội thoại: “thế còn gói cao cấp?”
- Câu hỏi dùng từ đời thường khác với thuật ngữ tài liệu.
- Tài liệu đa ngôn ngữ hoặc có nhiều cách gọi cùng một khái niệm.
- Query ngắn, thiếu ngữ cảnh nhưng lịch sử hội thoại có thông tin bổ sung.

**Bài toán giải quyết:** Tăng khả năng tìm đúng evidence khi query ban đầu quá nghèo thông tin.

**So sánh:**

| Kỹ thuật | Điểm mạnh | Rủi ro |
| --- | --- | --- |
| Query rewrite | Làm rõ câu hỏi theo hội thoại. | Rewrite sai ý người dùng. |
| Multi-query | Phủ nhiều cách diễn đạt. | Tăng số truy vấn và chi phí merge. |
| HyDE | Mạnh với semantic search khi query ngắn. | Đoạn giả định có thể kéo retrieval lệch hướng. |
| Hybrid search | Kết hợp từ khóa và ngữ nghĩa. | Cần tuning score/rank fusion. |

---

### 11. Chunking ảnh hưởng recall như thế nào?

**Trả lời:** Chunking ảnh hưởng trực tiếp đến recall vì retriever tìm trên đơn vị chunk. Chunk quá lớn có thể làm vector bị “loãng nghĩa”; chunk quá nhỏ có thể mất điều kiện quan trọng, khiến chunk đúng không đủ tín hiệu để được retrieve.

**Áp dụng khi nào?** Khi tài liệu là policy, legal, manual kỹ thuật, FAQ dài, bảng, transcript hoặc Markdown nhiều heading.

**Bài toán giải quyết:** Tăng khả năng evidence đúng được index và retrieve như một đơn vị độc lập.

**So sánh cách chunk:**

| Cách chunk | Ảnh hưởng đến recall | Khi dùng |
| --- | --- | --- |
| Fixed-size | Dễ triển khai nhưng có thể cắt ngang ý. | Prototype nhanh. |
| Section-aware | Giữ heading và điều kiện nghiệp vụ. | Policy, legal, tài liệu kỹ thuật. |
| Parent-child | Chunk nhỏ dễ retrieve, parent giữ ngữ cảnh. | Câu hỏi cần chi tiết nhưng answer cần đoạn đầy đủ. |
| Semantic chunking | Tách theo thay đổi chủ đề. | Văn bản dài ít cấu trúc. |

Nguyên tắc: benchmark cùng một bộ câu hỏi trước/sau khi thay đổi chunk size, overlap hoặc parser.

---

### 12. Recall trong tiếng Việt hoặc dữ liệu đa ngôn ngữ cần chú ý gì?

**Trả lời:** Với tiếng Việt, recall dễ giảm nếu embedding model không tốt cho tiếng Việt, tokenizer xử lý dấu/ký tự kém, hoặc dữ liệu trộn Việt–Anh nhưng query chỉ dùng một ngôn ngữ. Ngoài ra, tiếng Việt có nhiều cách diễn đạt đời thường khác với văn bản chính sách.

**Áp dụng khi nào?** RAG cho CSKH, chính sách nội bộ, pháp lý, tài liệu kỹ thuật hoặc catalogue có tiếng Việt, tiếng Anh và thuật ngữ viết tắt.

**Bài toán giải quyết:** Giảm bỏ sót do khác ngôn ngữ, khác thuật ngữ hoặc cách diễn đạt.

**Cách cải thiện:**

- Benchmark embedding model bằng câu hỏi tiếng Việt thật.
- Dùng hybrid search cho mã lỗi, SKU, tên sản phẩm, số điều khoản.
- Lưu synonym/domain glossary như “hoàn tiền”, “refund”, “trả lại tiền”.
- Cân nhắc query translation hoặc multilingual embedding nếu dữ liệu song ngữ.
- Không bỏ dấu tiếng Việt trong source nếu điều đó làm mất nghĩa hoặc citation khó đọc.

**So sánh:**

| Dữ liệu một ngôn ngữ | Dữ liệu đa ngôn ngữ |
| --- | --- |
| Dễ benchmark và debug hơn. | Cần kiểm tra cross-lingual retrieval. |
| Ít cần query translation. | Có thể cần multilingual embedding/hybrid. |
| Glossary đơn giản hơn. | Cần map thuật ngữ tương đương giữa ngôn ngữ. |

---

### 13. Theo dõi recall trên production như thế nào?

**Trả lời:** Không thể biết recall thật cho mọi câu hỏi production vì không phải lúc nào cũng có nhãn evidence đúng. Cần kết hợp offline benchmark, online trace, feedback người dùng và regression test.

**Áp dụng khi nào?** Khi hệ thống đã có người dùng thật, dữ liệu cập nhật thường xuyên hoặc team thường xuyên thay đổi parser, chunking, embedding, index, reranking.

**Bài toán giải quyết:** Phát hiện sớm việc cập nhật tài liệu hoặc thay đổi thuật toán làm retrieval kém đi.

Các tín hiệu nên theo dõi:

| Tín hiệu | Ý nghĩa |
| --- | --- |
| Offline Recall@k | Chất lượng retrieval trên bộ câu hỏi có nhãn. |
| No-result / low-score rate | Query không tìm được candidate đủ tốt. |
| Candidate count sau filter | Filter có đang loại quá nhiều không. |
| Citation click / feedback | Người dùng có thấy nguồn hữu ích không. |
| Answer flagged wrong | Cần truy vết lỗi retrieval hay generation. |
| Regression theo version | So sánh trước/sau khi re-index hoặc đổi model. |

**So sánh offline và online:**

| Offline evaluation | Online monitoring |
| --- | --- |
| Có nhãn, đo recall chính xác hơn. | Gần hành vi người dùng thật hơn. |
| Chạy trước khi release. | Phát hiện vấn đề sau release. |
| Có thể thiếu câu hỏi mới. | Khó biết ground truth đầy đủ. |

---

## Checklist áp dụng recall cho RAG production

1. Có bộ câu hỏi đánh giá gồm query thường gặp, query khó, query đa ngôn ngữ và query cần nhiều evidence.
2. Mỗi câu hỏi có `relevantChunkIds` hoặc `relevantDocumentIds` rõ ràng.
3. Đo `Recall@5`, `Recall@10`, `Recall@20`, `Recall@50` trước/sau mỗi thay đổi lớn.
4. Tách log giữa retrieval candidate, reranking, context assembly và generation.
5. Kiểm tra filter/ACL/version bằng test riêng, không gộp chung với semantic similarity.
6. So sánh keyword, vector và hybrid search trên cùng dataset.
7. Kiểm tra recall theo nhóm: tiếng Việt, tiếng Anh, mã lỗi/SKU, policy nhiều điều kiện, tài liệu cũ/mới.
8. Đặt ngưỡng regression: ví dụ `Recall@20` không được giảm quá một mức cho phép trên nhóm câu hỏi quan trọng.

## Gợi ý triển khai với Java Spring và Next.js

| Thành phần | Vai trò | Lưu ý triển khai |
| --- | --- | --- |
| Next.js | UI hỏi đáp, hiển thị citation, feedback đúng/sai. | Không quyết định ACL ở client; chỉ gửi user/session context. |
| Java Spring | API gateway, authN/authZ, orchestration RAG, logging. | Log `requestId`, filter, top-k, candidate IDs, latency. |
| Ingestion worker | Parse, chunk, embed, index, validate. | Idempotent theo content hash và document version. |
| Vector DB / search engine | Vector, keyword, hybrid retrieval. | Enforce filter/ACL ở query layer. |
| Evaluation job | Chạy benchmark recall định kỳ. | So sánh theo version model/index/chunking. |

Luồng kiểm thử tối thiểu:

```text
Evaluation dataset
  → Run retriever with filters
  → Compare retrieved IDs with relevant IDs
  → Compute Recall@k / Hit Rate@k / MRR
  → Save report by index version
  → Block release nếu regression vượt ngưỡng
```

## Tóm tắt để nhớ

> **Recall trả lời câu hỏi: “Retriever có tìm đủ bằng chứng đúng không?”**

Một hệ thống RAG tốt thường tối ưu theo thứ tự:

1. **Recall:** tìm đủ evidence đúng.
2. **Precision/reranking:** giảm nhiễu và xếp đúng evidence lên cao.
3. **Context assembly:** đưa đủ nhưng không quá nhiều context.
4. **Grounded generation:** LLM trả lời bám evidence và citation.
5. **Evaluation/monitoring:** phát hiện regression khi dữ liệu hoặc index thay đổi.

## Liên kết liên quan

- [Retrieval trong RAG](../retrieval/README.md)
- [Chunking trong RAG](../chunking/README.md)
- [Embedding trong RAG](../embedding/README.md)
- [RAG](../README.md)
- [LLM](../../README.md)
- [AI](../../../README.md)
- [Knowledge Tree](../../../../README.md)
