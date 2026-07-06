# Retrieval trong RAG

> [Knowledge Tree](../../../../README.md) / [AI](../../../README.md) / [LLM](../../README.md) / [RAG](../README.md) / **Retrieval**

## Điều hướng

- **Node cha:** [RAG](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** [Embedding](../embedding/README.md)
- **Khái niệm sau trong cùng nhánh:** [Chỉ mục pgvector](../pgvector-index/README.md)

## Mục tiêu của Retrieval

*Retrieval* là bước tìm và xếp hạng các đoạn tài liệu (*chunks*) có khả năng làm bằng chứng cho câu hỏi trước khi mô hình ngôn ngữ lớn (*Large Language Model, LLM*) sinh câu trả lời. Trong *Retrieval-Augmented Generation (RAG)*, retrieval không chỉ là “tìm văn bản giống câu hỏi”; nó phải chọn được bằng chứng **đúng chủ đề, đúng quyền truy cập, đúng phiên bản và còn hiệu lực**.

Một luồng phục vụ điển hình:

```text
Câu hỏi + danh tính người dùng
  → chuẩn hoá / phát hiện ngôn ngữ / gắn phạm vi quyền
  → query rewrite hoặc tách câu hỏi (khi cần)
  → lọc metadata và truy hồi ứng viên (dense, sparse hoặc hybrid)
  → fusion + reranking + đa dạng hoá kết quả
  → kiểm tra ngưỡng bằng chứng / từ chối có căn cứ
  → ghép context, giữ nguồn trích dẫn
  → LLM trả lời dựa trên context
```

| Luồng | Việc chính | Đầu ra cần kiểm soát |
| --- | --- | --- |
| **Chuẩn bị dữ liệu (*offline / ingestion*)** | Chunking, gắn metadata, tạo embedding, tạo chỉ mục từ kho dữ liệu nguồn | `chunkId`, văn bản, nguồn, phiên bản, thời hạn hiệu lực, ACL, embedding/index version |
| **Phục vụ truy vấn (*online / serving*)** | Nhận query, áp quyền và filter, lấy ứng viên, xếp hạng, quyết định trả lời | Context nhỏ nhưng đủ bằng chứng, thứ hạng, điểm số, nguồn trích dẫn, lý do từ chối khi thiếu evidence |

> **Source of truth:** tài liệu gốc, CMS, database nghiệp vụ hoặc API chính chủ là nguồn sự thật. Vector index chỉ là bản dẫn xuất để tìm nhanh; không được dùng nó làm nguồn quyết định cho dữ liệu thay đổi tức thời, giao dịch, số dư hoặc hành động nghiệp vụ.

## A. 5 câu hỏi cốt lõi

### 1. Retrieval là gì và khác gì với Generation hoặc Search truyền thống?

**Trả lời:** Retrieval nhận một truy vấn (*query*) rồi trả về tập bằng chứng phù hợp nhất để LLM dùng khi trả lời. *Generation* là bước LLM diễn đạt câu trả lời từ context đó; retrieval quyết định LLM được “nhìn thấy” dữ liệu nào. Search truyền thống thường tối ưu để người dùng tự đọc danh sách kết quả, còn retrieval cho RAG tối ưu để đưa một số ít đoạn ngắn, có provenance rõ ràng, vào context window của LLM.

**Áp dụng khi nào:** Dùng khi chatbot phải trả lời từ tài liệu nội bộ, chính sách, hướng dẫn kỹ thuật, knowledge base CSKH hoặc tài liệu có thay đổi theo thời gian.

**Bài toán giải quyết:** Giảm *hallucination* (bịa thông tin), không phải fine-tune lại model mỗi khi tài liệu đổi, và giúp câu trả lời truy vết được về nguồn.

**So sánh:**

| Lựa chọn | Phù hợp khi | Giới hạn so với Retrieval trong RAG |
| --- | --- | --- |
| Prompt chỉ có kiến thức sẵn của LLM | Kiến thức chung, không cần bằng chứng riêng | Dễ lỗi thời và không biết dữ liệu nội bộ |
| Fine-tuning | Muốn thay đổi phong cách, định dạng hoặc hành vi lặp lại ổn định | Không phù hợp làm kho tra cứu thường xuyên thay đổi; khó trích nguồn chính xác |
| Full-text search cho người dùng | Người dùng tự mở và đọc tài liệu | Không tự tạo context cô đọng cho LLM; thường kém với diễn đạt đồng nghĩa |
| Retrieval cho RAG | Cần câu trả lời dựa trên tài liệu có kiểm soát | Phụ thuộc chất lượng chunk, metadata, index và ranking |

### 2. Một pipeline retrieval tốt gồm những lớp nào?

**Trả lời:** Tách pipeline thành các lớp để mỗi lớp có trách nhiệm rõ ràng: (1) chuẩn hoá query và nhận diện người dùng; (2) áp *access control* (ACL) và filter bắt buộc; (3) lấy nhiều ứng viên (*candidate retrieval*); (4) hợp nhất/xếp hạng lại; (5) loại kết quả trùng hoặc quá giống nhau; (6) chọn context cuối cùng cùng trích dẫn. Không nên để LLM tự quyết quyền truy cập hay tự chọn tài liệu từ một danh sách quá rộng.

**Áp dụng khi nào:** Hệ thống có nhiều phòng ban, tenant, loại tài liệu, phiên bản chính sách hoặc dữ liệu có ngày hiệu lực.

**Bài toán giải quyết:** Ngăn lộ dữ liệu chéo tenant, giảm việc đưa context thừa vào LLM, và cho phép xác định lỗi nằm ở query, filter, candidate retrieval hay ranking.

**So sánh:** Một bước vector search đơn lẻ dễ xây nhưng khó kiểm soát quyền, chất lượng và chi phí. Pipeline nhiều tầng có thêm độ trễ, đổi lại cho chất lượng tốt hơn, quan sát được và an toàn hơn trong production.

### 3. Dense retrieval (vector search) hoạt động như thế nào và khi nào chưa đủ?

**Trả lời:** *Dense retrieval* dùng cùng embedding model để biến query và chunk thành vector trong cùng không gian. Hệ thống tìm các vector gần query theo *similarity metric* như cosine similarity hoặc inner product. Nó mạnh ở quan hệ ngữ nghĩa: “điều kiện hoàn tiền” có thể tìm được đoạn nói “quy trình refund” dù không trùng chữ.

Embedding của query phải tương thích với embedding đã dùng khi index tài liệu. Nếu thay model, thay cách chuẩn hoá hoặc thay metric mà không re-embed/re-index đồng bộ, điểm tương đồng sẽ không còn đáng tin.

**Áp dụng khi nào:** Câu hỏi tự nhiên, nhiều cách diễn đạt, dữ liệu tiếng Việt có từ đồng nghĩa hoặc tài liệu dài cần tìm theo ý nghĩa.

**Bài toán giải quyết:** Tăng *semantic recall* — vẫn tìm được bằng chứng khi từ khoá giữa query và tài liệu không giống hệt nhau.

**So sánh:** Dense retrieval thường yếu hơn keyword search khi cần mã lỗi, mã sản phẩm, số điều khoản, SKU, tên riêng hiếm hoặc cụm phải khớp chính xác. Vì vậy nó thường là một nhánh trong hybrid retrieval, không phải mặc định thay thế hoàn toàn lexical search.

### 4. Hybrid retrieval là gì, và vì sao thường phù hợp cho production?

**Trả lời:** *Hybrid retrieval* kết hợp ít nhất hai tín hiệu: **dense/vector search** cho ngữ nghĩa và **sparse/lexical search** như BM25 cho từ khoá chính xác. Mỗi nhánh trả về danh sách ứng viên; sau đó hệ thống hợp nhất bằng chiến lược như *Reciprocal Rank Fusion (RRF)* hoặc mô hình học để xếp hạng.

Ví dụ, câu “lỗi `AUTH-401` khi refresh token” cần lexical search để giữ đúng mã lỗi, nhưng dense search lại hữu ích để lấy hướng dẫn “token hết hạn” hoặc “xác thực lại”.

**Áp dụng khi nào:** Kho tri thức vừa có văn bản tự nhiên vừa có thuật ngữ chính xác: tài liệu kỹ thuật, chính sách có số điều, product catalog, nhật ký lỗi đã chuẩn hoá hoặc hỗ trợ khách hàng.

**Bài toán giải quyết:** Giảm bỏ sót evidence do chỉ dùng semantic search, đồng thời tránh kết quả chỉ khớp từ bề mặt nhưng sai ý nghĩa.

**So sánh:**

| Cách truy hồi | Điểm mạnh | Điểm yếu | Nên chọn khi |
| --- | --- | --- | --- |
| Sparse / BM25 | Exact match, dễ giải thích, nhẹ | Kém với đồng nghĩa và diễn đạt khác | Mã, tên riêng, thuật ngữ ổn định |
| Dense / vector | Hiểu ngữ nghĩa và paraphrase | Có thể bỏ qua token quan trọng | Hỏi đáp tự nhiên, knowledge base dài |
| Hybrid | Cân bằng recall của cả hai | Cần thêm index, fusion và đánh giá | Production có dữ liệu hỗn hợp |

### 5. Chọn top-k, reranking, MMR và threshold như thế nào?

**Trả lời:** Không nên dùng một `topK` duy nhất từ index đến LLM. Hãy truy hồi tập ứng viên đủ rộng để ưu tiên *recall*, sau đó dùng các tầng để ưu tiên *precision*:

1. Lấy ứng viên từ dense và/hoặc sparse retrieval.
2. Fusion để có một danh sách chung.
3. Dùng *reranker* để đánh giá trực tiếp cặp `query–chunk`.
4. Dùng *Maximum Marginal Relevance (MMR)* hoặc deduplication để không lấy nhiều chunk gần như giống nhau.
5. Áp threshold theo điểm hoặc theo quy tắc business; chỉ gửi context có evidence đủ mạnh.

Điểm bắt đầu có thể là lấy vài chục ứng viên, rerank một tập nhỏ hơn và chỉ gửi vài chunk đa dạng vào LLM; các con số phải được tune bằng tập đánh giá và ngân sách độ trễ, không sao chép máy móc giữa dự án.

**Áp dụng khi nào:** Khi LLM nhận context bị loãng, trả lời lặp, trích nhiều đoạn cùng một tài liệu nhưng thiếu góc nhìn khác, hoặc latency tăng do đưa quá nhiều text.

**Bài toán giải quyết:** Cân bằng *recall*, *precision*, chi phí token và latency; tăng khả năng có một evidence đúng nằm trong context cuối.

**So sánh:** Tăng `topK` đơn thuần thường tăng recall nhưng làm LLM phân tâm và tốn token. Reranking tăng precision nhưng tốn thêm latency/chi phí. MMR bảo đảm đa dạng nhưng cần cẩn thận để không loại mất nhiều evidence bổ sung từ cùng một quy trình phức tạp.

## B. 8 câu hỏi phổ biến mở rộng

### 6. Metadata filter, ACL, tenant và phiên bản tài liệu phải được áp dụng ở đâu?

**Trả lời:** Áp filter bảo mật và phạm vi dữ liệu **trước hoặc ngay trong truy vấn retrieval**, không áp sau khi đã lấy chunk. Metadata tối thiểu nên có `tenantId`/`organizationId`, `accessPolicy` hoặc nhóm quyền, `documentStatus`, `documentVersion`, `effectiveFrom`, `effectiveTo`, `sourceId`, `updatedAt` và `indexVersion`.

Một retrieval request nên mang *retrieval contract* rõ ràng, ví dụ: `userId`, tenant, roles, locale, thời điểm tham chiếu và loại tài liệu được phép. Với chính sách đã bị thu hồi, cần loại ngay từ filter hoặc gắn trạng thái để không được chọn cho context.

**Áp dụng khi nào:** RAG cho doanh nghiệp, đa tenant, HR, legal, tài chính, tài liệu trả phí hoặc môi trường có dữ liệu nội bộ.

**Bài toán giải quyết:** Ngăn *data leakage*, tránh trả lời từ chính sách cũ, và tạo audit trail để điều tra “vì sao hệ thống đã dùng tài liệu này”.

**So sánh:** Post-filter sau vector search có thể làm rỗng kết quả và làm lộ dấu vết dữ liệu bị cấm trong log/cache. Pre-filter/in-filter an toàn hơn; đánh đổi là cần vector store và index hỗ trợ filter hiệu quả.

### 7. Khi nào nên dùng query rewrite, query expansion hoặc query decomposition?

**Trả lời:**

- *Query rewrite* viết lại câu hỏi để rõ ý hơn, ví dụ thay đại từ hoặc chuẩn hoá thuật ngữ.
- *Query expansion* thêm từ đồng nghĩa, alias, mã nội bộ hoặc biến thể ngôn ngữ.
- *Query decomposition* tách câu hỏi nhiều phần thành các truy vấn nhỏ, sau đó tổng hợp evidence.

Chỉ bật các kỹ thuật này khi evaluation cho thấy tăng recall hoặc chất lượng đáp án. Rewrite sai có thể đổi ý định người dùng; query expansion quá rộng có thể đưa nhiễu; decomposition tăng số lần truy vấn và cần kiểm soát cách tổng hợp.

**Áp dụng khi nào:** Câu hỏi mơ hồ, đa ngôn ngữ, có thuật ngữ viết tắt, nhiều thực thể, hoặc câu hỏi cần đối chiếu nhiều tài liệu.

**Bài toán giải quyết:** Làm query gần hơn với cách tài liệu được viết, tăng khả năng tìm đủ evidence cho truy vấn nhiều bước.

**So sánh:** Dùng LLM rewrite linh hoạt nhưng chi phí và khó tái lập hơn rule-based normalization. Với mã lỗi, SKU và tên riêng, nên bảo toàn token gốc thay vì rewrite tự do.

### 8. Reranker là gì và khác candidate retriever ra sao?

**Trả lời:** Candidate retriever phải nhanh nên thường dùng vector similarity, BM25 hoặc ANN index. *Reranker* nhận một số ứng viên đã lấy và chấm trực tiếp mức liên quan của cặp `query–chunk`; nhiều reranker dùng cross-encoder nên chính xác hơn nhưng không thể chạy trên toàn bộ corpus.

Một pattern phổ biến là: retrieval ưu tiên recall → reranking ưu tiên precision → MMR/dedup ưu tiên diversity. Reranker cần nhìn được query và chunk đầy đủ, nên đặc biệt hữu ích khi các chunk có embedding gần nhau nhưng chỉ một đoạn thật sự trả lời đúng câu hỏi.

**Áp dụng khi nào:** Corpus lớn, câu hỏi chi tiết, hybrid retrieval còn nhiều kết quả gần đúng, hoặc cần giảm context trước khi gọi LLM đắt tiền.

**Bài toán giải quyết:** Đưa evidence đúng lên top context, giảm “lost in the middle” khi LLM phải đọc quá nhiều đoạn.

**So sánh:** Bi-encoder/vector retrieval rẻ và nhanh cho first-stage retrieval; cross-encoder reranker chậm hơn nhưng chính xác hơn. Có thể bỏ reranker ở corpus nhỏ, latency rất chặt hoặc evaluation cho thấy lợi ích không đáng chi phí.

### 9. Vì sao retrieval trả về kết quả không liên quan, và debug theo thứ tự nào?

**Trả lời:** Đừng chỉ nhìn câu trả lời của LLM. Hãy log và kiểm tra từng tầng của retrieval:

| Dấu hiệu | Nguyên nhân thường gặp | Cách kiểm tra / khắc phục |
| --- | --- | --- |
| Không có kết quả | Filter quá chặt, index chưa cập nhật, query sai tenant | Log filter cuối, kiểm tra index freshness và document status |
| Có kết quả nhưng sai chủ đề | Chunk quá lớn/nhỏ, embedding kém phù hợp, query mơ hồ | So sánh raw query với rewritten query; kiểm tra chunk boundary và embedding model |
| Có tài liệu đúng nhưng không vào top | Metric/index/fusion chưa phù hợp, thiếu reranker | Đo Recall@k trước rerank; thử hybrid/RRF và kiểm tra score distribution |
| Context lặp nhiều đoạn giống nhau | Top-k quá nhiều, thiếu dedup/MMR | Nhóm theo `documentId`/section, áp diversity policy |
| LLM vẫn trả lời sai dù evidence đúng | Prompt/context assembly yếu hoặc evidence mâu thuẫn | Đánh giá retrieval và generation tách riêng; giữ citation theo từng claim |

**Áp dụng khi nào:** Mọi hệ thống RAG đã bắt đầu có người dùng thật hoặc có yêu cầu chất lượng rõ ràng.

**Bài toán giải quyết:** Tránh sửa prompt một cách cảm tính khi lỗi thực ra nằm ở dữ liệu, index, filter hoặc ranking.

**So sánh:** Debug bằng “một vài demo thành công” dễ tạo cảm giác sai về chất lượng. Tập truy vấn đại diện, log có cấu trúc và metric theo tầng giúp phát hiện regressions đáng tin hơn.

### 10. Khi không có evidence đủ tốt, hệ thống nên phản hồi thế nào?

**Trả lời:** Retrieval phải hỗ trợ *grounded abstention* — từ chối trả lời hoặc yêu cầu làm rõ khi không có bằng chứng đạt tiêu chuẩn. Tiêu chuẩn có thể kết hợp: score threshold, khoảng cách điểm giữa kết quả đầu và phần còn lại, số nguồn độc lập, trạng thái tài liệu, độ mới và policy theo domain.

Câu trả lời nên minh bạch: nêu rằng chưa tìm thấy thông tin trong phạm vi được cấp quyền, gợi ý từ khoá hoặc nguồn chính chủ phù hợp, nhưng không bịa phần còn thiếu. Với tác vụ rủi ro cao, chuyển người dùng tới quy trình/hệ thống nguồn thay vì suy đoán.

**Áp dụng khi nào:** Legal, HR, y tế, tài chính, chính sách doanh nghiệp, hỗ trợ kỹ thuật có thể gây thao tác sai hoặc bất kỳ chatbot nào có cam kết “trả lời theo tài liệu”.

**Bài toán giải quyết:** Giảm trả lời tự tin nhưng sai; tạo UX an toàn hơn khi corpus chưa bao phủ câu hỏi.

**So sánh:** Luôn trả lời giúp trải nghiệm có vẻ mượt nhưng rủi ro cao. Threshold quá chặt lại làm tăng từ chối sai (*false abstention*); cần tune bằng dữ liệu thật và tách ngưỡng theo loại câu hỏi.

### 11. Đánh giá retrieval bằng các chỉ số nào?

**Trả lời:** Xây một bộ test gồm query đại diện, expected document/chunk hoặc *relevance judgments*, tenant/quyền truy cập, thời điểm hiệu lực và loại câu hỏi khó. Đo retrieval riêng trước khi đánh giá LLM:

| Chỉ số | Ý nghĩa | Dùng để trả lời |
| --- | --- | --- |
| `Recall@k` | Evidence đúng có xuất hiện trong top-k không | Retriever có bỏ sót tài liệu quan trọng không? |
| `MRR` | Kết quả đúng đầu tiên đứng cao đến đâu | Người dùng/LLM có gặp evidence đúng sớm không? |
| `nDCG@k` | Chất lượng thứ hạng với nhiều mức relevance | Ranking có ưu tiên đoạn tốt hơn không? |
| Filter/ACL pass rate | Kết quả có tôn trọng phạm vi quyền không | Có rò rỉ hoặc loại nhầm dữ liệu không? |
| Freshness / version accuracy | Evidence có đúng bản còn hiệu lực không | Có đang dùng tài liệu cũ không? |
| Latency và cost | Thời gian/chi phí theo từng tầng | Có đạt SLO và ngân sách không? |

Sau đó đánh giá end-to-end: câu trả lời có bám evidence (*faithfulness/groundedness*), đủ ý, đúng nghiệp vụ và có trích dẫn đúng không. Một LLM trả lời tốt trên câu hỏi dễ không chứng minh retrieval tốt; ngược lại evidence đúng nhưng prompt tệ cũng có thể cho đáp án kém.

**Áp dụng khi nào:** Trước khi production, khi đổi embedding/index/reranker, sau re-index lớn hoặc khi có phản hồi chất lượng từ người dùng.

**Bài toán giải quyết:** Ra quyết định dựa trên số liệu thay vì cảm nhận; phát hiện regression theo từng tầng.

**So sánh:** Chỉ đo câu trả lời cuối khó biết lỗi ở đâu. Chỉ đo retrieval lại chưa phản ánh UX. Cần cả hai lớp, nhưng retrieval metric là nền tảng để tối ưu đúng chỗ.

### 12. Vận hành retrieval trong production cần quan sát và tối ưu gì?

**Trả lời:** Trace mỗi request bằng một `requestId` nhưng không log thô dữ liệu nhạy cảm. Các trường hữu ích gồm: query đã được làm sạch/redact, query version, filter/ACL policy version, nguồn ứng viên, rank/score, document/chunk/version được chọn, thời gian từng tầng, token context, cache hit và outcome (answered, abstained, escalated).

Theo dõi drift: tỷ lệ zero-result, score distribution, tỷ lệ từ chối, latency p50/p95/p99, cache hit, tỷ lệ tài liệu cũ, lỗi index và khác biệt chất lượng trước/sau deploy. Re-index nên có `indexVersion`, chạy song song/canary, đánh giá rồi mới chuyển traffic; phải có rollback về index cũ.

**Áp dụng khi nào:** Mọi production RAG có dữ liệu thay đổi, nhiều người dùng hoặc SLA/chi phí rõ ràng.

**Bài toán giải quyết:** Phát hiện corpus bị stale, filter hỏng, reranker gây latency, regression do đổi model hoặc sự cố index mà không chờ đến khi người dùng báo lỗi.

**So sánh:** Cache giúp nhanh và rẻ nhưng phải gắn key với tenant, roles, filter, model/index version và freshness policy; cache không đúng phạm vi quyền có thể gây lộ dữ liệu.

### 13. Chọn FAISS, pgvector, Qdrant hay Elasticsearch/OpenSearch cho retrieval thế nào? Khi nào không nên dùng retrieval?

**Trả lời:** Chọn theo nơi dữ liệu đang ở, nhu cầu filter/hybrid, khả năng vận hành và quy mô, không chỉ theo benchmark ANN.

| Lựa chọn | Phù hợp | Điểm mạnh | Đánh đổi |
| --- | --- | --- | --- |
| **FAISS** | Prototype/offline retrieval, đội tự vận hành index | Nhanh, linh hoạt, nhiều thuật toán ANN | Là thư viện; persistence, filtering, API, HA và multi-tenant do hệ thống khác đảm nhiệm |
| **pgvector** | Dữ liệu đã ở PostgreSQL, quy mô vừa đến lớn có kiểm soát | Gần transaction/metadata/SQL, đơn giản hoá kiến trúc | Cần thiết kế index, vacuum và tuning Postgres; không phải lúc nào cũng tối ưu cho search chuyên dụng |
| **Qdrant** | Vector-first RAG cần filter và service chuyên biệt | API vector rõ ràng, metadata filter, vận hành như vector database | Thêm hệ thống/chi phí vận hành và cần đồng bộ source data |
| **Elasticsearch / OpenSearch** | Cần full-text/BM25, hybrid search, search analytics đã có sẵn | Mạnh về lexical search, filter và hệ sinh thái search | Vector capability, mapping và relevance tuning có thể phức tạp hơn |

**Không nên dùng retrieval làm nguồn quyết định duy nhất** cho dữ liệu realtime, phép tính chính xác, số dư/tồn kho, giao dịch hoặc hành động có side effect. Khi đó hãy gọi API/database nguồn và dùng retrieval chỉ để giải thích hướng dẫn, chính sách hoặc tài liệu liên quan.

**Áp dụng khi nào:** Bước thiết kế kiến trúc hoặc khi migration từ prototype sang production.

**Bài toán giải quyết:** Tránh chọn vector database chỉ vì xu hướng trong khi hệ thống cần lexical/hybrid, SQL transaction hoặc vận hành tối giản.

**So sánh:** Một stack chuyên biệt có thể đạt khả năng search tốt hơn; một stack gần dữ liệu nguồn giảm độ phức tạp đồng bộ. Quyết định cuối nên dựa trên workload thật: query mix, filter cardinality, ACL, SLO, chi phí và năng lực vận hành.

## Checklist áp dụng nhanh

- [ ] Xác định *source of truth* và dữ liệu nào retrieval chỉ được dùng để tham khảo.
- [ ] Thiết kế chunk và metadata có `source`, version, thời hạn hiệu lực, tenant/ACL.
- [ ] Dùng sparse, dense hoặc hybrid theo query mix thực tế; không chọn chỉ vì benchmark.
- [ ] Áp ACL và filter trước/trong truy vấn, không sau khi lấy kết quả.
- [ ] Tách candidate retrieval, reranking, diversity và context assembly để đo/debug từng tầng.
- [ ] Có threshold/abstention và đường chuyển tiếp cho câu hỏi thiếu bằng chứng.
- [ ] Đánh giá bằng tập query có relevance, permission và version; theo dõi Recall@k, ranking, latency và grounding.
- [ ] Version hoá embedding, index và pipeline; có chiến lược re-index/rollback.

## Liên kết liên quan

- [RAG](../README.md)
- [Chunking trong RAG](../chunking/README.md)
- [Embedding trong RAG](../embedding/README.md)
- [Chỉ mục pgvector](../pgvector-index/README.md)
