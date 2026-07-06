# Embedding trong RAG

> [Knowledge Tree](../../../../README.md) / [AI](../../../README.md) / [LLM](../../README.md) / [RAG](../README.md) / **Embedding**

**Embedding** là phép biến đổi một đoạn văn bản thành vector số thực (*dense vector*) sao cho các nội dung gần nghĩa có xu hướng ở gần nhau trong không gian vector (*vector space*). Trong *Retrieval-Augmented Generation (RAG)*, embedding liên kết câu hỏi của người dùng với các chunk đã được chuẩn bị từ tài liệu, để hệ thống tìm được bằng chứng phù hợp trước khi *Large Language Model (LLM)* tạo câu trả lời.

Embedding không thay thế tri thức nguồn, metadata, bộ lọc quyền hay bước xếp hạng. Nó là tín hiệu ngữ nghĩa cho retrieval: chất lượng thực tế phụ thuộc đồng thời vào [Chunking](../chunking/README.md), model, text được đưa vào model, similarity metric, index, filter và cách ghép context.

## Điều hướng

- **Node cha:** [RAG](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** [Chunking](../chunking/README.md)
- **Khái niệm sau trong cùng nhánh:** [Retrieval](../retrieval/README.md)

## Nguyên tắc nhanh

1. Embed **query** và **document chunk** bằng cùng không gian vector, hoặc bằng cặp model/query-document mà nhà cung cấp xác nhận tương thích.
2. Chọn model bằng tập câu hỏi và tài liệu thật của doanh nghiệp; benchmark công khai chỉ là tín hiệu ban đầu.
3. Đưa vào vector phần text có ý nghĩa truy xuất, gồm ngữ cảnh section cần thiết; giữ quyền, phiên bản và nguồn trong metadata để filter/citation.
4. Similarity metric, cách normalize vector và cấu hình index phải khớp với khuyến nghị của model; không tự đổi metric chỉ vì thư viện hỗ trợ.
5. Xem embedding là artefact của pipeline **offline/ingestion** có version, hash, đo lường, re-embed và rollback; không phải dữ liệu tạo một lần rồi quên.

## A. 5 câu hỏi cốt lõi

### 1. Embedding là gì và nó hoạt động ở đâu trong RAG?

**Trả lời:** Embedding model nhận text và trả về một dãy số có số chiều cố định, gọi là vector. Model được huấn luyện để vector của các đoạn cùng ý nghĩa gần nhau hơn vector của các đoạn không liên quan. Khi người dùng hỏi, hệ thống tạo vector cho query, tìm các vector chunk gần nhất trong vector database hoặc index, rồi chuyển các chunk đó cho LLM làm context.

Trong luồng **offline/ingestion**, hệ thống parse tài liệu, [chunking](../chunking/README.md), gắn metadata, tạo vector và ghi `vector + metadata + text/source reference` vào index. Trong luồng **online/serving**, hệ thống kiểm tra quyền, chuẩn hóa query, tạo query embedding, tìm kiếm có filter, có thể rerank, ghép context và yêu cầu LLM trả lời dựa trên evidence. Embedding không “ghi nhớ” tài liệu theo nghĩa cập nhật trọng số của LLM; nó là lớp tìm kiếm trên dữ liệu đã index.

**Áp dụng khi nào:** Chatbot tra cứu chính sách, tài liệu kỹ thuật, knowledge base, FAQ, tài liệu hỗ trợ khách hàng, kho tài liệu nội bộ hoặc tìm kiếm đa ngôn ngữ cần hiểu câu hỏi diễn đạt khác với văn bản nguồn.

**Bài toán giải quyết:** Giảm phụ thuộc vào việc query phải trùng từ khoá với tài liệu; tìm được nội dung như “điều kiện nghỉ phép năm” khi tài liệu viết “tiêu chuẩn hưởng phép thường niên”.

| Cách tìm kiếm | Điểm mạnh | Giới hạn | Phù hợp khi |
| --- | --- | --- | --- |
| Keyword/BM25 | Rất tốt với mã lỗi, tên riêng, ký hiệu, từ hiếm; dễ giải thích | Yếu khi query và tài liệu dùng cách diễn đạt khác nhau | Log, mã sản phẩm, API name, thuật ngữ chính xác |
| Dense embedding | Bắt được ngữ nghĩa và paraphrase | Có thể bỏ sót token hiếm hoặc nhầm nội dung gần nghĩa | Hỏi–đáp tài liệu tự nhiên, dữ liệu Việt/Anh |
| Full-text LLM context | Không cần retrieval phức tạp khi toàn bộ dữ liệu rất nhỏ | Bị giới hạn context, chi phí và khó cập nhật | Một tài liệu ngắn, tác vụ đọc toàn văn |
| Hybrid search | Kết hợp semantic và keyword để tăng recall | Nhiều thành phần cần tune/đánh giá hơn | Lựa chọn mặc định cho kho dữ liệu đa dạng ở production |

### 2. Vì sao query và document phải dùng cùng không gian vector?

**Trả lời:** Similarity chỉ có ý nghĩa khi vector query và vector document được tạo trong không gian tương thích. Cách an toàn nhất là dùng cùng model, cùng model version, cùng cấu hình tiền xử lý và cùng cách encode cho cả hai. Một số model là **bi-encoder bất đối xứng** (*asymmetric bi-encoder*): model cung cấp template hoặc API riêng cho query và document, chẳng hạn `encodeQuery` và `encodeDocument`. Hai đầu vẫn tương thích vì được thiết kế thành một cặp; không được thay thế ngẫu nhiên một đầu bằng model khác.

Không nên tạo document vector bằng model A rồi query vector bằng model B, dù cả hai có cùng số chiều. Cùng dimension không đồng nghĩa cùng hệ tọa độ. Kết quả có thể không báo lỗi nhưng ranking trở nên vô nghĩa, khó phát hiện bằng kiểm thử đơn vị.

**Áp dụng khi nào:** Luôn áp dụng khi tạo index mới, đổi nhà cung cấp embedding, thay model version, dùng endpoint query/document riêng hoặc di chuyển từ thử nghiệm sang production.

**Bài toán giải quyết:** Tránh lỗi retrieval “trông có vẻ chạy” nhưng top-k không liên quan do vector không thể so sánh đúng.

| Thiết kế encode | Ưu điểm | Rủi ro | Khi dùng |
| --- | --- | --- | --- |
| Cùng một encoder cho query và chunk | Đơn giản, ít sai cấu hình | Có thể chưa tối ưu cho ý định query ngắn | Baseline tốt, hệ thống nhỏ/vừa |
| Query/document encoder cùng họ model | Có thể tối ưu retrieval bất đối xứng | Phải dùng đúng instruction/API từng phía | Model có hướng dẫn chính thức cho retrieval |
| Hai model độc lập | Chỉ hợp lệ nếu có mapping/huấn luyện tương thích rõ ràng | Similarity thường vô nghĩa | Chỉ dùng trong hệ thống nghiên cứu có kiểm chứng |

**Checklist tối thiểu:** Lưu `embeddingModel`, `embeddingModelVersion`, `dimension`, `metric`, `preprocessingVersion` và loại encoder vào record/index. Khi serving, từ chối hoặc route query sang index phù hợp nếu các giá trị này không khớp.

### 3. Chọn embedding model cho tiếng Việt và dữ liệu chuyên ngành như thế nào?

**Trả lời:** Không nên chọn chỉ vì model đứng đầu một leaderboard. Model phù hợp phải được kiểm tra trên câu hỏi thật, cách viết tắt, tên sản phẩm, thuật ngữ nội bộ, typo, câu Việt không dấu, câu Việt–Anh trộn lẫn và tài liệu có cấu trúc của doanh nghiệp. Các tiêu chí thực tế gồm chất lượng Recall@k/MRR/nDCG, hỗ trợ tiếng Việt và đa ngôn ngữ, giới hạn input, số chiều vector, latency, chi phí, giấy phép sử dụng, khả năng triển khai trong vùng dữ liệu yêu cầu và chính sách lưu dữ liệu của nhà cung cấp.

Hãy bắt đầu bằng model multilingual có chất lượng tốt, thiết lập một baseline retrieval và đánh giá. Chỉ cân nhắc fine-tune hoặc model chuyên ngành khi lỗi còn lại có mẫu lặp lại, đủ dữ liệu relevance có nhãn và lợi ích cao hơn chi phí huấn luyện, vận hành và re-index. Fine-tune không sửa được chunking sai, parser mất bảng hoặc ACL filter thiếu.

**Áp dụng khi nào:** Khi xây index đầu tiên, thêm tiếng Việt vào hệ thống đa ngôn ngữ, dữ liệu nhiều từ viết tắt nội bộ, thay model vì chi phí/latency hoặc chất lượng retrieval giảm.

**Bài toán giải quyết:** Tránh chọn model mạnh trên dữ liệu tổng quát nhưng kém với tên chính sách, mã lỗi, văn phong nghiệp vụ hoặc câu hỏi tiếng Việt thực tế.

| Lựa chọn model | Điểm mạnh | Hạn chế | Phù hợp khi |
| --- | --- | --- | --- |
| Multilingual general-purpose | Triển khai nhanh, hỗ trợ Việt/Anh và cross-lingual | Có thể chưa hiểu sâu jargon nội bộ | Giai đoạn đầu, kho tài liệu đa ngôn ngữ |
| Model ưu tiên tiếng Việt/domain | Có thể khớp văn phong hoặc thuật ngữ tốt hơn | Cần tự benchmark; có thể yếu khi mở rộng ngôn ngữ | Dữ liệu chủ yếu một ngôn ngữ/lĩnh vực |
| Fine-tuned model | Tối ưu cho distribution đã biết | Cần dữ liệu nhãn, đánh giá và vòng đời model | Lỗi retrieval lặp lại, quy mô sử dụng đủ lớn |
| API managed | Ít hạ tầng model, cập nhật thuận tiện | Phụ thuộc vendor, chi phí/dữ liệu cần kiểm soát | Đội ngũ muốn ra sản phẩm nhanh |
| Self-hosted model | Kiểm soát dữ liệu và latency nội vùng | Cần GPU/CPU capacity, monitoring, nâng cấp | Dữ liệu nhạy cảm hoặc quy mô ổn định lớn |

**Cách đánh giá thực dụng:** Tạo tập 50–200 query đại diện trước; với mỗi query, ghi chunk/section được coi là evidence đúng. So sánh model trong cùng pipeline, cùng chunking, top-k và filter. Không so kết quả của model mới với index cũ có thay đổi đồng thời quá nhiều biến.

### 4. Nên embed phần nào của một tài liệu và phần nào chỉ nên để metadata?

**Trả lời:** Embedding nên nhận phần văn bản dùng để trả lời và phần ngữ cảnh giúp đoạn văn tự mang nghĩa. Một chunk thường phù hợp với cấu trúc: `title > sectionPath > body`. Ví dụ, thay vì chỉ embed “Nhân viên được hưởng 12 ngày”, nên embed “Sổ tay nhân sự > Nghỉ phép năm > Số ngày phép: Nhân viên được hưởng 12 ngày...”. Header ngắn làm giảm tình trạng một đoạn chung chung bị nhầm giữa nhiều chính sách khác nhau.

Metadata dùng cho lọc và quản trị, không nên bị coi là nội dung câu trả lời. Các field thường có: `documentId`, `chunkId`, `sourceUri`, `title`, `sectionPath`, `page`, `version`, `updatedAt`, `language`, `tenantId`, `documentType`, `accessScope`, `textHash`, `embeddingModel` và `preprocessingVersion`. Quyền truy cập (*Access Control List, ACL*) phải được thực thi bằng filter bắt buộc tại retrieval, dựa trên *source of truth* của hệ thống quyền; không chỉ ghi câu “chỉ HR” vào text.

**Áp dụng khi nào:** Luôn cần ở production, đặc biệt với policy có version, dữ liệu nhiều tenant/role, tài liệu cần citation, hoặc kho có nhiều đoạn ngắn giống nhau.

**Bài toán giải quyết:** Tăng ngữ cảnh ngữ nghĩa cho chunk ngắn; giảm citation sai; lọc đúng tài liệu còn hiệu lực; tránh rò rỉ dữ liệu chéo tenant hoặc role.

| Nội dung đưa vào vector | Lợi ích | Rủi ro | Khuyến nghị |
| --- | --- | --- | --- |
| Chỉ body chunk | Ít token, đơn giản | Mất ngữ cảnh của đoạn ngắn | Chỉ dùng khi chunk tự đủ nghĩa |
| Body + title/section path | Cân bằng chất lượng và chi phí | Header quá dài có thể lấn nội dung chính | Lựa chọn mặc định |
| Toàn bộ tài liệu lặp trong mỗi chunk | Ngữ cảnh rất nhiều | Tốn chi phí, nhiễu, duplicate tín hiệu | Tránh dùng |
| Metadata chỉ để filter/citation | An toàn, dễ vận hành | Vector không tự hiểu metadata nếu không có header | Kết hợp header ngắn khi cần |

Ví dụ record rút gọn:

```json
{
  "chunkId": "hr-handbook:v2026-07:annual-leave:03",
  "text": "Sổ tay nhân sự > Nghỉ phép năm > Điều kiện: ...",
  "documentId": "hr-handbook",
  "sectionPath": ["Nghỉ phép năm", "Điều kiện"],
  "sourceUri": "kb://hr/handbook/annual-leave",
  "version": "2026-07",
  "accessScope": ["employee"],
  "textHash": "sha256:...",
  "embeddingModel": "approved-model-id",
  "preprocessingVersion": "2026-07-06"
}
```

### 5. Similarity metric, normalization và vector index liên quan với nhau thế nào?

**Trả lời:** Retrieval xếp hạng document vector theo độ gần với query vector. Ba metric phổ biến là **cosine similarity**, **dot product** và **Euclidean distance/L2**. Cosine đo góc giữa hai vector; dot product chịu ảnh hưởng cả hướng lẫn độ lớn; L2 đo khoảng cách hình học. Model hoặc nhà cung cấp thường chỉ rõ metric và việc có cần normalize vector hay không. Hãy tuân theo hướng dẫn đó và cấu hình vector database/index khớp hoàn toàn.

Với cosine similarity, nhiều hệ thống normalize vector về độ dài 1 rồi dùng dot product vì hai cách trở nên tương đương về thứ tự xếp hạng. Nhưng không được normalize hoặc đổi metric một cách mù quáng: một số model sử dụng độ lớn vector như một tín hiệu. Cần kiểm tra cả hướng dẫn model lẫn kết quả trên evaluation set.

Index gần đúng (*Approximate Nearest Neighbor, ANN*) như HNSW, IVF hoặc disk-based ANN đổi một phần độ chính xác lấy tốc độ và chi phí. Đây là tầng phục vụ tìm kiếm, không sửa được embedding không tốt. Chi tiết về cách lấy, filter và rerank nằm ở [Retrieval](../retrieval/README.md); cấu hình index PostgreSQL được trình bày tại [pgvector Index](../pgvector-index/README.md).

**Áp dụng khi nào:** Khi tạo collection/index, chuyển vector database, thay metric, tối ưu latency, hoặc thấy top-k thay đổi bất thường sau migrate.

**Bài toán giải quyết:** Tránh score không nhất quán, recall giảm do index/metric sai, hoặc tối ưu ANN làm mất evidence quan trọng mà không đo lường.

| Metric | Ý nghĩa | Lưu ý | Phù hợp khi |
| --- | --- | --- | --- |
| Cosine similarity | So hướng vector | Thường đi cùng L2 normalization | Model/documentation khuyến nghị cosine |
| Dot product | Hướng và độ lớn | Chỉ dùng đúng khi model hỗ trợ | Model tối ưu inner product |
| Euclidean/L2 | Khoảng cách hình học | Scale vector ảnh hưởng kết quả | Model/index khuyến nghị L2 |
| ANN index | Tìm gần đúng nhanh | Có trade-off recall–latency | Kho dữ liệu đủ lớn cần response nhanh |

## B. 8 câu hỏi phổ biến mở rộng

### 6. Có cần làm sạch và chuẩn hóa text trước khi embedding không?

**Trả lời:** Có, nhưng mục tiêu là sửa nhiễu do extraction chứ không làm mất tín hiệu nghiệp vụ. Nên chuẩn hóa Unicode, khoảng trắng, ký tự xuống dòng lặp, header/footer trang bị lặp do PDF, hyphenation do xuống dòng, HTML boilerplate và OCR noise rõ ràng. Giữ nguyên dấu tiếng Việt, số, tên riêng, mã lỗi, mã sản phẩm, tên API, semantic version và nội dung pháp lý. Không lower-case hoặc bỏ dấu toàn cục nếu chưa đo ảnh hưởng; điều đó có thể làm mất phân biệt mã/tên hoặc làm citation khó đọc.

Tách riêng `displayText` để hiển thị/citation và `embeddingText` sau khi làm sạch. Cả hai cần liên kết qua `textHash` hoặc source offsets để audit được chunk nào đã tạo vector nào.

**Áp dụng khi nào:** PDF/OCR, HTML crawl, tài liệu copy từ Word, log/document kỹ thuật hoặc dữ liệu có header/footer lặp.

**Bài toán giải quyết:** Giảm vector bị chi phối bởi boilerplate; tăng chất lượng match; giữ khả năng trích dẫn nguyên văn đáng tin cậy.

| Cách xử lý | Lợi ích | Rủi ro |
| --- | --- | --- |
| Làm sạch có rule và version | Reproducible, dễ rollback | Phải bảo trì rule theo nguồn dữ liệu |
| Dùng raw text nguyên bản | Bảo toàn nội dung | Nhiễu extraction có thể làm retrieval kém |
| Chuẩn hóa quá mạnh | Có thể giảm biến thể câu chữ | Mất mã, dấu, điều kiện pháp lý hoặc tên riêng |

### 7. Kích thước chunk, giới hạn token và số chiều vector ảnh hưởng thế nào đến embedding?

**Trả lời:** Chunking quyết định model nhìn thấy đơn vị thông tin nào; embedding không thể khôi phục ý bị cắt mất ở đầu vào. Chunk quá ngắn dễ thiếu điều kiện/ngoại lệ, chunk quá dài chứa nhiều chủ đề và có thể vượt giới hạn input dẫn đến truncation. Truncation âm thầm đặc biệt nguy hiểm vì vector chỉ đại diện phần đầu, trong khi hệ thống vẫn tưởng đã index toàn bộ nội dung.

Số chiều (*dimension*) ảnh hưởng dung lượng index, chi phí RAM/disk và tốc độ search, nhưng dimension lớn không tự động tốt hơn. Chất lượng representation của model và evaluation thực tế quan trọng hơn. Không thể trộn vector khác dimension trong cùng collection; khi đổi model có dimension khác, cần collection/index mới.

**Áp dụng khi nào:** Khi chọn model, điều chỉnh [chunking](../chunking/README.md), gặp lỗi input quá dài, chi phí vector store tăng hoặc migrate model.

**Bài toán giải quyết:** Tránh bỏ sót nội dung cuối chunk, index phình quá mức và so sánh vector không hợp lệ.

| Quyết định | Tác động chính | Cách kiểm soát |
| --- | --- | --- |
| Chunk nhỏ | Precision cao hơn nhưng dễ thiếu context | Đo context completeness và Recall@k |
| Chunk lớn | Nhiều ngữ cảnh nhưng dễ nhiễu/truncation | Kiểm tra token trước encode; chia theo structure |
| Dimension lớn | Có thể tăng capacity biểu diễn | So latency, storage và quality trên test set |
| Đổi dimension | Không tương thích collection cũ | Tạo index song song, re-embed toàn bộ |

### 8. Embedding có thay thế keyword search/BM25 không?

**Trả lời:** Không. Embedding thường tốt hơn cho đồng nghĩa, paraphrase và câu hỏi tự nhiên; keyword/BM25 thường tốt hơn với mã lỗi, part number, tên hàm, phiên bản, acronyms, địa chỉ chính xác và token hiếm. Với kho dữ liệu doanh nghiệp, hybrid retrieval thường cho recall tốt hơn vì hai tín hiệu bù cho nhau.

Một chiến lược thông dụng là lấy ứng viên từ semantic search và keyword search, gộp kết quả bằng score fusion như *Reciprocal Rank Fusion (RRF)*, áp dụng metadata/ACL filter, rồi rerank top-N nếu cần. Không nên dùng hybrid chỉ vì “nghe mạnh hơn”: hãy chứng minh bằng tập query gồm cả câu hỏi ngữ nghĩa lẫn query chứa mã/tên chính xác.

**Áp dụng khi nào:** Knowledge base kỹ thuật, sản phẩm, log, ticket hỗ trợ, tài liệu song ngữ và kho chứa cả prose lẫn định danh chính xác.

**Bài toán giải quyết:** Giảm bỏ sót evidence do từ khoá hiếm, đồng thời vẫn tìm được nội dung diễn đạt khác câu hỏi của người dùng.

| Tình huống query | Semantic embedding | Keyword/BM25 | Lựa chọn ưu tiên |
| --- | --- | --- | --- |
| “Tôi được nghỉ phép bao nhiêu ngày?” | Mạnh | Có thể đủ | Semantic hoặc hybrid |
| `ERR_CONNECTION_RESET` | Có thể không ổn định | Rất mạnh | Keyword hoặc hybrid |
| Tên API + mô tả mục đích | Bổ sung ngữ nghĩa tốt | Khớp tên API tốt | Hybrid |
| Viết tắt nội bộ mới xuất hiện | Cần dữ liệu/model hiểu đúng | Khớp literal tốt | Hybrid + synonym dictionary khi cần |

### 9. Đánh giá chất lượng embedding và retrieval bằng cách nào?

**Trả lời:** Đánh giá đúng không chỉ hỏi LLM có trả lời “hay” hay không. Trước hết cần tập query có evidence mong đợi: một hoặc nhiều chunk/section/document đúng, kèm nhãn quyền và phiên bản nếu liên quan. Đo **Recall@k** để biết evidence đúng có xuất hiện trong top-k không; dùng **MRR** hoặc **nDCG** khi thứ hạng và nhiều mức relevance quan trọng; đo thêm p50/p95 latency, tỷ lệ filter loại nhầm, chi phí embedding/index và tỷ lệ citation đúng ở end-to-end.

Chia tập kiểm thử theo loại query: tiếng Việt có dấu/không dấu, Việt–Anh, paraphrase, mã lỗi, tên riêng, câu ngắn/mơ hồ, tài liệu cũ/mới, query không có đáp án và query bị từ chối theo ACL. Dùng lỗi thực từ production đã được review để bổ sung liên tục vào evaluation set.

**Áp dụng khi nào:** Trước launch, sau thay đổi model/chunking/index, khi chất lượng bị phản ánh, hoặc trong CI/CD của ingestion pipeline.

**Bài toán giải quyết:** Phân biệt lỗi embedding với lỗi chunking, filter, ANN index, reranker hoặc LLM generation; tránh quyết định theo vài demo đẹp.

| Chỉ số | Trả lời câu hỏi | Không tự đo được |
| --- | --- | --- |
| Recall@k | Evidence đúng có được lấy vào top-k không? | Câu trả lời có dùng evidence đúng không |
| MRR / nDCG | Evidence quan trọng có đứng đủ cao không? | Citation và policy compliance |
| p95 latency | Trải nghiệm ở trường hợp chậm có chấp nhận được không? | Chất lượng ngữ nghĩa |
| End-to-end groundedness | LLM có bám vào evidence không? | Nguyên nhân gốc ở retrieval nếu thiếu trace |

### 10. Khi nào phải re-embed và migration model an toàn ra sao?

**Trả lời:** Cần re-embed khi đổi model/model version, thay cách tiền xử lý có ảnh hưởng text, thay cấu trúc chunk, thay dimension/metric, hoặc phát hiện index cũ thiếu nội dung/metadata quan trọng. Chỉ thay query model mà giữ document vector cũ thường không an toàn nếu không gian vector thay đổi.

Migration nên theo đường song song: tạo index version mới, backfill document vector bằng pipeline có idempotency, dual-write cho dữ liệu mới trong thời gian chuyển đổi, chạy evaluation và shadow traffic, so sánh top-k/latency/cost, rollout theo tỷ lệ, rồi giữ index cũ đủ lâu để rollback. Không xoá index cũ trước khi xác nhận index mới đáp ứng chất lượng và quyền truy cập.

**Áp dụng khi nào:** Nâng cấp model, thay vendor, tối ưu chi phí, đổi parser/chunking, hoặc khắc phục lỗi retrieval diện rộng.

**Bài toán giải quyết:** Tránh downtime, index dữ liệu nửa cũ nửa mới, vector không tương thích và không thể điều tra khi ranking thay đổi.

| Cách migrate | Ưu điểm | Rủi ro |
| --- | --- | --- |
| Ghi đè index tại chỗ | Nhanh, ít hạ tầng | Khó rollback; trạng thái hỗn hợp |
| Index version song song | Có rollback và A/B/shadow evaluation | Tốn storage/compute tạm thời |
| Re-embed theo lazy read | Giảm đỉnh chi phí | Chất lượng không đồng đều lâu; khó kiểm soát |

**Metadata vận hành cần có:** `indexVersion`, `embeddingModel`, `modelVersion`, `dimension`, `metric`, `preprocessingVersion`, `chunkingVersion`, `createdAt`, `sourceVersion` và trạng thái ingest. Đây là dữ liệu để truy vết, không chỉ để hiển thị.

### 11. Xử lý tiếng Việt, tiếng Anh và câu hỏi trộn ngôn ngữ như thế nào?

**Trả lời:** Dùng model multilingual đã được benchmark với cặp query–document thực tế là lựa chọn đầu tiên cho kho Việt/Anh. Nếu hệ thống có tài liệu theo ngôn ngữ rõ ràng, lưu `language` metadata để quan sát, filter hoặc ưu tiên kết quả đúng ngôn ngữ. Tuy nhiên, filter cứng theo ngôn ngữ có thể làm mất tài liệu tiếng Anh cần thiết cho query tiếng Việt, như tài liệu API hoặc tên sản phẩm.

Có thể dùng query expansion/translation như một nhánh bổ sung khi benchmark cho thấy cross-lingual recall chưa đủ, nhưng phải lưu query gốc, query mở rộng và nguồn kết quả để debug. Không dịch tự động nội dung pháp lý hoặc text dùng để citation rồi hiển thị như nguồn gốc; citation luôn phải trỏ về tài liệu gốc.

**Áp dụng khi nào:** Tài liệu nội bộ Việt–Anh, sản phẩm quốc tế, hỗ trợ kỹ thuật dùng thuật ngữ tiếng Anh, người dùng gõ không dấu hoặc code-switching.

**Bài toán giải quyết:** Tìm được hướng dẫn tiếng Anh cho câu hỏi tiếng Việt mà vẫn ưu tiên nguồn cùng ngôn ngữ khi phù hợp.

| Phương án | Điểm mạnh | Lưu ý |
| --- | --- | --- |
| Multilingual embedding | Đơn giản, hỗ trợ cross-lingual | Phải benchmark theo ngôn ngữ/domain |
| Tách index theo ngôn ngữ | Cấu hình rõ, dễ tối ưu riêng | Có thể mất cross-lingual evidence |
| Dịch/expand query | Có thể tăng recall | Thêm latency, chi phí và bề mặt lỗi |

### 12. Có nên cache embedding và deduplicate nội dung không?

**Trả lời:** Có, nhất là khi ingestion lặp lại hoặc nhiều tài liệu có chung section/template. Cache key nên dựa trên `normalizedTextHash + embeddingModelVersion + preprocessingVersion`, không chỉ raw text. Khi text, model hoặc preprocessing thay đổi thì cache phải miss để vector mới được tạo. Deduplicate cần thận trọng: hai đoạn text giống nhau nhưng khác tenant, version, source hoặc quyền vẫn phải giữ metadata/provenance riêng để filter và citation đúng.

Cache query embedding hữu ích với câu hỏi lặp nhiều, nhưng cần TTL, quota và chính sách dữ liệu phù hợp. Tránh log hoặc cache raw query vô thời hạn nếu query có thể chứa dữ liệu cá nhân/bí mật. Vector cũng là dữ liệu cần được bảo vệ: không mặc định coi vector đã vô hại chỉ vì không còn là câu văn gốc.

**Áp dụng khi nào:** Batch ingestion, kho tài liệu versioned, chatbot có query lặp, hệ thống có chi phí API embedding đáng kể.

**Bài toán giải quyết:** Giảm chi phí/latency, rút ngắn thời gian re-index và vẫn bảo toàn isolation theo tenant/role.

| Cách tối ưu | Lợi ích | Điều kiện an toàn |
| --- | --- | --- |
| Cache theo content hash + model version | Tránh encode lặp | Invalidate khi text/pipeline/model đổi |
| Deduplicate vector vật lý | Giảm storage | Tách ownership, ACL và citation ở record logic |
| Cache query embedding | Giảm latency query lặp | TTL, không lưu dữ liệu nhạy cảm quá lâu |

### 13. Embedding có rủi ro bảo mật hoặc kiểm soát quyền nào cần chú ý?

**Trả lời:** Có. Embedding không phải cơ chế phân quyền và không được dùng làm ranh giới bảo mật. Trước hoặc ngay trong retrieval, hệ thống phải tính scope quyền từ *source of truth* (identity/tenant/role/document permission) và áp dụng metadata filter bắt buộc. Không gửi chunk chưa được phép vào reranker hoặc LLM chỉ vì chúng đứng cao trong vector search. Với query không có quyền hoặc không có evidence hợp lệ, hệ thống cần từ chối/giới hạn câu trả lời thay vì suy đoán.

Bảo vệ vector store như bảo vệ kho dữ liệu: phân quyền truy cập, mã hóa khi truyền/lưu theo yêu cầu tổ chức, secret management, audit log, retention/deletion theo chính sách, cô lập tenant và kiểm tra filter qua automated tests. Khi vendor xử lý embedding, cần đánh giá data residency, retention, training policy và thoả thuận dữ liệu phù hợp trước khi gửi nội dung nhạy cảm.

**Áp dụng khi nào:** Multi-tenant SaaS, HR/finance/legal/health data, tài liệu nội bộ phân quyền, hệ thống có yêu cầu audit/compliance.

**Bài toán giải quyết:** Ngăn leakage chéo tenant/role, citation từ tài liệu hết quyền, và các câu trả lời dựa trên evidence không được phép truy cập.

| Biện pháp | Bảo vệ gì | Không thay thế |
| --- | --- | --- |
| ACL/tenant filter bắt buộc khi retrieval | Quyền xem kết quả | Kiểm tra identity và chính sách nguồn |
| Metadata version/status filter | Không dùng tài liệu hết hiệu lực | Quy trình quản lý tài liệu nguồn |
| Audit log query → retrieved chunks → answer | Điều tra và compliance | Phòng ngừa rò rỉ từ đầu |
| Encryption/access control cho vector store | Giảm rủi ro truy cập trái phép | Isolation logic trong ứng dụng |

## Checklist triển khai embedding cho RAG

1. Xác định chunk là đơn vị có thể trả lời; giữ `title` và `sectionPath` cần thiết trong `embeddingText`.
2. Chọn một model baseline và metric đúng theo tài liệu model; lưu version/dimension trong index metadata.
3. Chuẩn hóa extraction có version; giữ raw/display text để citation và audit.
4. Xây tập query–evidence đại diện, gồm query semantic, keyword, Việt/Anh, không có đáp án và query bị giới hạn quyền.
5. Đo Recall@k, MRR/nDCG, latency, cost, filter correctness và grounded/citation quality trước khi quyết định model.
6. Dùng hybrid search khi kho có mã, tên riêng hoặc thuật ngữ chính xác; đánh giá trước khi thêm reranker.
7. Re-embed qua index version song song; shadow test và rollback được.
8. Áp dụng ACL/version/tenant filter từ source of truth trước khi bất kỳ chunk nào đi vào prompt.

## Liên kết liên quan

- [RAG](../README.md)
- [Chunking](../chunking/README.md)
- [Retrieval](../retrieval/README.md)
- [Chỉ mục pgvector (pgvector Index)](../pgvector-index/README.md)
