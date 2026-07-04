# RAG (Retrieval-Augmented Generation)

> [Knowledge Tree](../../../README.md) / [AI](../../README.md) / [LLM](../README.md) / **RAG**

**RAG (Retrieval-Augmented Generation)** là cách kết hợp truy xuất tri thức (*retrieval*) với sinh câu trả lời bằng mô hình ngôn ngữ lớn (*Large Language Model, LLM*). Công thức cần nhớ:

> **Question → Retrieve → Context → Generate → Cite**

RAG không thay thế database, search engine hay API nghiệp vụ. Nó là lớp giúp LLM tìm đúng bằng chứng từ kho tri thức trước khi trả lời, từ đó tạo câu trả lời có căn cứ và dễ kiểm chứng hơn.

## Điều hướng

- **Node cha:** [LLM](../README.md)
- **Node con:** [Chunking](chunking/README.md), [Embedding](embedding/README.md), [Retrieval](retrieval/README.md)
- **Khái niệm trước / sau trong nhánh:** Chưa có.

## Nhánh con

1. [Chunking](chunking/README.md) — Chia tài liệu theo cấu trúc và ngữ cảnh.
2. [Embedding](embedding/README.md) — Biểu diễn query/chunk bằng vector ngữ nghĩa.
3. [Retrieval](retrieval/README.md) — Vector search, hybrid search, metadata filtering và chọn context.

## A. 5 câu hỏi cốt lõi

### 1. RAG là gì?

**Trả lời:** RAG là kỹ thuật để LLM trả lời dựa trên các đoạn tài liệu được tìm thấy tại thời điểm người dùng hỏi, thay vì chỉ dựa vào kiến thức đã có trong trọng số của model. Hệ thống thường tìm các đoạn có liên quan trong kho tri thức, đưa chúng vào *context*, rồi yêu cầu LLM tổng hợp câu trả lời dựa trên những bằng chứng đó.

**Ví dụ:** Người dùng hỏi: “Chính sách đổi trả sản phẩm A thế nào?” Thay vì để LLM tự suy đoán, hệ thống tìm đúng đoạn trong tài liệu chính sách đổi trả của sản phẩm A, rồi trả lời và dẫn nguồn.

**Áp dụng khi nào?**

- Chatbot tra cứu tài liệu nội bộ, chính sách doanh nghiệp, FAQ hoặc knowledge base.
- Trợ lý kỹ thuật đọc tài liệu API, runbook, tài liệu vận hành và source documentation.
- Hệ thống CSKH cần trả lời từ catalogue, điều khoản bảo hành, quy trình xử lý khiếu nại.
- Dữ liệu thay đổi thường xuyên hoặc là dữ liệu riêng không có trong kiến thức công khai của model.

**Bài toán giải quyết:**

- LLM không biết dữ liệu riêng của tổ chức.
- Cần cập nhật kiến thức mà không phải huấn luyện lại model.
- Cần giảm câu trả lời bịa đặt (*hallucination*) và tăng khả năng kiểm chứng.
- Người dùng cần biết câu trả lời lấy từ tài liệu nào.

**So sánh với các cách tương tự:**

| Cách làm | Điểm khác với RAG | Phù hợp hơn khi |
| --- | --- | --- |
| Prompt thông thường | Chỉ dựa vào hướng dẫn và kiến thức sẵn có của LLM; không truy xuất tài liệu riêng. | Nội dung rất ngắn, ổn định và có thể đặt trực tiếp trong prompt. |
| Fine-tuning | Thay đổi hành vi hoặc phong cách của model bằng huấn luyện; không phù hợp để cập nhật tài liệu thường xuyên. | Cần model tuân thủ định dạng, giọng văn hoặc quy tắc hành vi ổn định. |
| Search engine | Trả về tài liệu/đường dẫn nhưng không tổng hợp câu trả lời tự nhiên. | Người dùng muốn tự đọc kết quả tìm kiếm. |
| RAG | Tìm bằng chứng rồi dùng LLM tổng hợp câu trả lời theo ngữ cảnh. | Cần trả lời hội thoại từ kho tri thức có nguồn. |

---

### 2. Pipeline RAG cơ bản hoạt động như thế nào?

**Trả lời:** Pipeline RAG thường gồm hai luồng: **ingestion/indexing** để chuẩn bị dữ liệu trước, và **query/serving** để trả lời câu hỏi lúc chạy.

**Luồng ingestion/indexing:**

1. **Load documents** — đọc dữ liệu từ PDF, DOCX, HTML, Markdown, wiki, database hoặc API.
2. **Parse và clean** — tách nội dung chính, bỏ phần nhiễu, giữ cấu trúc như tiêu đề, bảng, trang và section.
3. **Chunking** — chia tài liệu thành các đoạn nhỏ có nghĩa.
4. **Embedding** — chuyển từng chunk thành vector ngữ nghĩa.
5. **Indexing** — lưu vector, text gốc và metadata vào vector database hoặc search engine.

**Luồng query/serving:**

1. Nhận câu hỏi của người dùng.
2. Có thể viết lại câu hỏi (*query rewriting*) hoặc nhận diện ý định.
3. Tìm các chunk liên quan (*retrieval*), có thể kèm filter metadata và hybrid search.
4. Sắp xếp lại kết quả (*reranking*) và chọn context tốt nhất.
5. Đưa context, câu hỏi và quy tắc trả lời vào LLM.
6. Trả lời, gắn citation và từ chối có kiểm soát khi evidence không đủ.

**Áp dụng khi nào?** Dùng khi tài liệu nhiều hơn khả năng đưa trực tiếp vào prompt, hoặc khi cần tái sử dụng cùng một kho tri thức cho nhiều câu hỏi.

**Bài toán giải quyết:** Pipeline tách rõ phần chuẩn bị dữ liệu và phần trả lời, giúp cập nhật tài liệu, tối ưu chất lượng retrieval và kiểm soát chi phí token tốt hơn.

**So sánh với long context:**

| Long context | RAG |
| --- | --- |
| Đưa một lượng lớn tài liệu vào prompt mỗi lần hỏi. | Chỉ lấy một số đoạn liên quan nhất. |
| Nhanh để làm thử với ít tài liệu. | Hiệu quả hơn cho kho tài liệu lớn và thường xuyên thay đổi. |
| Tốn token, dễ nhiễu khi có nhiều nội dung không liên quan. | Tiết kiệm token hơn nếu retrieval tốt. |
| Không cần index ban đầu. | Cần ingestion, embedding và index. |

Trong production, RAG và long context có thể kết hợp: dùng RAG để thu hẹp dữ liệu, sau đó dùng context dài hơn cho phần tài liệu thật sự liên quan.

---

### 3. Embedding trong RAG là gì?

**Trả lời:** *Embedding* là cách chuyển văn bản thành một dãy số gọi là **vector**. Các câu hoặc đoạn có nghĩa gần nhau sẽ có vector gần nhau theo một thước đo như cosine similarity hoặc dot product.

**Ví dụ:** “đổi trả hàng”, “hoàn trả sản phẩm” và “return policy” có thể khác chữ, nhưng embedding có thể nhận diện chúng gần về ngữ nghĩa hơn keyword search thuần túy.

**Áp dụng khi nào?**

- Người dùng diễn đạt tự nhiên, không dùng đúng từ như trong tài liệu.
- Có nhiều cách gọi cho cùng một khái niệm.
- Cần semantic search hoặc tìm kiếm đa ngôn ngữ.
- Kho tài liệu thiên về đoạn văn, hướng dẫn, quy định hoặc FAQ.

**Bài toán giải quyết:** Embedding giảm tình trạng bỏ sót tài liệu chỉ vì câu hỏi và tài liệu dùng từ khác nhau, ví dụ người dùng hỏi “hoàn tiền” trong khi tài liệu dùng “refund”.

**So sánh với keyword search:**

| Keyword search | Embedding / vector search |
| --- | --- |
| Tìm theo từ hoặc cụm từ xuất hiện trực tiếp. | Tìm theo mức độ gần về nghĩa. |
| Mạnh với mã đơn hàng, SKU, tên sản phẩm, số hiệu điều khoản. | Mạnh với câu hỏi tự nhiên và cách diễn đạt đa dạng. |
| Dễ giải thích, nhanh và có tính xác định cao. | Có thể tìm đúng ý dù không trùng từ. |
| Có thể bỏ sót khi người dùng dùng từ đồng nghĩa. | Có thể trả về kết quả gần nghĩa nhưng không đúng điều kiện cụ thể. |

Vì mỗi cách có điểm mạnh riêng, production RAG thường dùng **hybrid search**: kết hợp lexical/keyword search và vector search.

---

### 4. Vector Database trong RAG là gì?

**Trả lời:** *Vector database* là nơi lưu vector embedding của các chunk, text gốc và metadata để hệ thống có thể tìm các vector gần với câu hỏi nhanh chóng. Một số hệ thống dùng vector database chuyên dụng; một số khác dùng search engine hoặc PostgreSQL có hỗ trợ vector.

**Các lựa chọn phổ biến:**

| Công cụ | Điểm phù hợp chính |
| --- | --- |
| FAISS | Local, nhanh, phù hợp prototype hoặc tự vận hành index trong ứng dụng. |
| Qdrant | Mạnh về filtering metadata, dễ triển khai và vận hành cho nhiều use case RAG. |
| Milvus | Hệ thống vector chuyên dụng, phù hợp quy mô dữ liệu lớn và workload vector nặng. |
| Pinecone | Managed service, giảm gánh nặng vận hành hạ tầng. |
| Weaviate | Có hệ sinh thái và nhiều tính năng tích hợp cho ứng dụng AI. |
| pgvector | Phù hợp khi đã dùng PostgreSQL và quy mô/nhu cầu vector chưa cần tách một database chuyên dụng. |
| Elasticsearch / OpenSearch | Phù hợp khi cần kết hợp full-text search, filter, analytics và vector search trong cùng nền tảng. |

**Áp dụng khi nào?** Dùng khi có từ hàng nghìn đến hàng triệu chunk, cần tìm semantic search nhanh và thường xuyên lọc theo metadata như phòng ban, phiên bản, sản phẩm, ngôn ngữ hoặc quyền truy cập.

**Bài toán giải quyết:** Vector database giúp retrieval có độ trễ chấp nhận được khi dữ liệu lớn, đồng thời hỗ trợ phân vùng dữ liệu, filter và mở rộng hạ tầng.

**Lưu ý:** Không phải hệ thống RAG nào cũng cần vector database độc lập. Nếu dữ liệu ít và PostgreSQL hiện có đáp ứng được query, `pgvector` có thể giảm độ phức tạp. Ngược lại, nếu cần hybrid search, faceting và full-text search mạnh, Elasticsearch/OpenSearch thường thuận tiện hơn.

---

### 5. Chunking trong RAG là gì?

**Trả lời:** *Chunking* là việc chia tài liệu dài thành những đoạn nhỏ để index và truy xuất. Chunk tốt phải đủ nhỏ để retrieval chính xác, nhưng đủ lớn để giữ bối cảnh và câu trả lời không bị thiếu điều kiện quan trọng.

**Ví dụ:** Một PDF chính sách 100 trang không nên được lưu là một vector duy nhất. Có thể chia theo heading/section, sau đó tiếp tục chia thành chunk khoảng 300–800 token tùy loại tài liệu; mỗi chunk giữ metadata như tài liệu nguồn, trang, tiêu đề section và ngày hiệu lực.

**Áp dụng khi nào?** Dùng với hầu hết tài liệu dài như PDF, quy định, manual kỹ thuật, hợp đồng, FAQ dài, knowledge base hoặc tài liệu Markdown nhiều section.

**Bài toán giải quyết:**

- Tránh việc một vector đại diện cho quá nhiều chủ đề.
- Tăng khả năng tìm đúng đoạn trả lời thay vì trả về cả tài liệu.
- Kiểm soát kích thước context và chi phí token.
- Giữ được citation theo section hoặc số trang.

**So sánh các kiểu chunking:**

| Kiểu chunking | Đặc điểm | Phù hợp khi |
| --- | --- | --- |
| Fixed-size chunking | Chia theo số ký tự/token cố định, thường có overlap. | Prototype hoặc dữ liệu khá đồng nhất. |
| Recursive chunking | Ưu tiên ranh giới heading, paragraph, sentence rồi mới cắt nhỏ hơn. | Markdown, HTML, tài liệu hướng dẫn có cấu trúc. |
| Semantic chunking | Chia tại điểm chuyển chủ đề/ngữ nghĩa. | Văn bản dài, ít cấu trúc nhưng cần giữ mạch ý. |
| Section-aware chunking | Giữ nguyên section, heading, điều khoản, bảng và số trang. | Chính sách công ty, pháp lý, tài liệu kỹ thuật. |
| Parent-child chunking | Tìm chunk nhỏ để chính xác, nhưng trả về chunk cha lớn hơn để đủ context. | Tài liệu có điều kiện, ngoại lệ và tham chiếu chéo. |

**Nguyên tắc thực tế:** Đừng chọn kích thước chunk bằng cảm tính. Hãy tạo bộ câu hỏi đại diện, đo retrieval trước/sau khi thay đổi chunk size, overlap, parser và metadata.

## B. 8 câu hỏi phổ biến mở rộng

### 6. RAG khác Fine-tuning như thế nào?

**Trả lời:** RAG bổ sung kiến thức tại thời điểm trả lời bằng cách tìm tài liệu; fine-tuning thay đổi trọng số hoặc hành vi của model bằng dữ liệu huấn luyện. Hai kỹ thuật giải quyết hai loại vấn đề khác nhau.

**Dùng RAG khi:**

- Kiến thức thay đổi thường xuyên: chính sách, giá, catalogue, tài liệu sản phẩm, hướng dẫn vận hành.
- Cần trả lời từ dữ liệu riêng của tổ chức.
- Cần citation và khả năng kiểm chứng nguồn.
- Muốn cập nhật tri thức bằng cách nạp lại tài liệu thay vì train lại.

**Dùng fine-tuning khi:**

- Cần model giữ một giọng văn, phong cách hoặc cấu trúc output nhất quán.
- Cần tuân thủ mẫu gọi tool, JSON schema hoặc quy trình xử lý ổn định.
- Có nhiều ví dụ chất lượng cao mô tả hành vi mong muốn.

| Tiêu chí | RAG | Fine-tuning |
| --- | --- | --- |
| Cập nhật kiến thức | Nhanh: thêm/sửa tài liệu và re-index. | Chậm hơn: cần chuẩn bị dữ liệu và huấn luyện lại. |
| Dữ liệu mới hoặc thay đổi liên tục | Rất phù hợp. | Không phải lựa chọn đầu tiên. |
| Học phong cách trả lời | Hạn chế nếu chỉ dùng context. | Phù hợp hơn. |
| Citation / kiểm chứng | Tự nhiên hơn vì có nguồn retrieved. | Không bảo đảm model nhớ đúng nguồn. |
| Chi phí khởi tạo | Cần pipeline dữ liệu và search. | Cần dữ liệu huấn luyện và chi phí train/evaluate. |

**Kết luận:** Nhiều hệ thống tốt kết hợp cả hai: fine-tuning hoặc prompt để chuẩn hóa hành vi, RAG để cung cấp tri thức chính xác và cập nhật.

---

### 7. Retrieval trong RAG là gì? Khi nào dùng hybrid search?

**Trả lời:** *Retrieval* là bước chọn các chunk có khả năng trả lời câu hỏi nhất. Retrieval tốt là điều kiện cần để LLM trả lời tốt; nếu lấy sai evidence, LLM dù mạnh vẫn có thể trả lời sai hoặc lan man.

**Các cách retrieval phổ biến:**

| Cách | Cách hoạt động | Mạnh khi |
| --- | --- | --- |
| Keyword / lexical retrieval | So khớp từ khóa, BM25 hoặc inverted index. | Mã lỗi, SKU, số hiệu điều khoản, tên riêng, cụm từ chính xác. |
| Vector retrieval | Tìm vector gần nghĩa với query. | Câu hỏi tự nhiên, từ đồng nghĩa, diễn đạt đa dạng. |
| Hybrid search | Kết hợp điểm keyword và vector. | Kho vừa có mã/tên riêng vừa có câu hỏi ngữ nghĩa. |
| Metadata-filtered retrieval | Lọc trước hoặc trong lúc search theo metadata. | Có nhiều sản phẩm, phòng ban, phiên bản, vùng/quốc gia hoặc quyền. |

**Ví dụ:** Câu hỏi “Lỗi `PAY-401` xử lý thế nào?” nên ưu tiên keyword search vì mã lỗi là exact-match. Câu hỏi “không đăng nhập được sau khi đổi mật khẩu” cần vector search để nhận diện ngữ nghĩa. Một hệ thống hỗ trợ kỹ thuật thường cần hybrid search để xử lý cả hai.

**Áp dụng khi nào?** Hybrid search nên là mặc định đáng cân nhắc khi tài liệu chứa nhiều mã, tên tính năng, điều kiện nghiệp vụ hoặc thuật ngữ đặc thù; vector-only thường dễ bỏ qua exact match, còn keyword-only thường bỏ qua cách diễn đạt tự nhiên.

**Bài toán giải quyết:** Retrieval và hybrid search tăng *recall* của các đoạn đúng, giảm tình trạng lấy nhầm tài liệu gần nghĩa nhưng sai phiên bản/sản phẩm.

---

### 8. Reranking trong RAG là gì?

**Trả lời:** *Reranking* là bước đánh giá lại các kết quả retrieval để sắp xếp những chunk phù hợp nhất lên đầu. Thay vì đưa thẳng top 3–5 từ vector search vào LLM, hệ thống có thể lấy top 20–50 ứng viên rồi để reranker chọn lại top 3–10.

**Quy trình điển hình:**

1. Retriever lấy nhanh nhiều candidate có khả năng liên quan.
2. Reranker đọc từng cặp `query + chunk` để đánh giá mức độ phù hợp sâu hơn.
3. Hệ thống chọn ít chunk chất lượng cao nhất làm context cho LLM.

**Áp dụng khi nào?**

- Kho tài liệu lớn và nhiều đoạn có nghĩa gần nhau.
- Câu hỏi có nhiều điều kiện hoặc cần phân biệt chi tiết nhỏ.
- Kết quả top-k ban đầu thường đúng chủ đề nhưng chưa đúng câu trả lời.
- Hệ thống yêu cầu độ chính xác cao như hỗ trợ kỹ thuật, chính sách hoặc pháp lý nội bộ.

**Bài toán giải quyết:** Reranking giảm nhiễu context, tăng *precision* ở các kết quả đầu, nhờ đó LLM dễ trả lời đúng trọng tâm hơn.

| Không reranking | Có reranking |
| --- | --- |
| Độ trễ thấp hơn. | Tăng độ trễ và chi phí một bước đánh giá. |
| Có thể đưa chunk đúng chủ đề nhưng không trả lời trực tiếp vào prompt. | Ưu tiên chunk trả lời trực tiếp và đủ điều kiện. |
| Phù hợp prototype hoặc workload đơn giản. | Phù hợp production cần chất lượng retrieval cao. |

**Lưu ý:** Reranking không sửa được dữ liệu không được retrieve. Cần tối ưu retrieval, chunking và metadata trước; reranker chỉ chọn tốt hơn từ tập candidate đã có.

---

### 9. Metadata và ACL trong RAG để làm gì?

**Trả lời:** *Metadata* là dữ liệu mô tả gắn với mỗi chunk; **ACL (Access Control List)** là quy tắc xác định người dùng nào được phép truy cập chunk đó. Metadata biến vector search từ “tìm đoạn gần nghĩa” thành “tìm đúng đoạn gần nghĩa trong đúng phạm vi”.

**Ví dụ metadata:**

```json
{
  "source": "loyalty-policy-2026.pdf",
  "page": 12,
  "section": "Quy đổi điểm",
  "product": "membership",
  "department": "sales",
  "effectiveFrom": "2026-01-01",
  "status": "active",
  "allowedRoles": ["sales", "support"]
}
```

**Áp dụng khi nào?**

- Có nhiều phiên bản tài liệu và chỉ tài liệu mới nhất được dùng.
- Một chatbot phục vụ nhiều sản phẩm, thị trường, công ty con hoặc phòng ban.
- Có dữ liệu nhạy cảm cần phân quyền theo user, role, team hoặc tenant.
- Cần citation có số trang, section, file gốc và ngày hiệu lực.

**Bài toán giải quyết:**

- Tránh lấy chính sách cũ hoặc không còn hiệu lực.
- Tránh trộn tài liệu của các sản phẩm/khách hàng khác nhau.
- Hạn chế lộ dữ liệu trái quyền truy cập.
- Giúp debug vì biết chunk đến từ nguồn, phiên bản và section nào.

| Chỉ vector search | Vector + metadata + ACL |
| --- | --- |
| Có thể lấy nội dung gần nghĩa nhưng sai sản phẩm hoặc sai thời điểm. | Có thể filter chính xác theo product, version, region và trạng thái. |
| Khó kiểm soát dữ liệu đa tenant. | Có thể enforce quyền trước retrieval. |
| Citation nghèo thông tin. | Dễ hiển thị file, trang, section và hiệu lực tài liệu. |

**Nguyên tắc bảo mật:** Không chỉ kiểm tra quyền sau khi LLM đã nhận context. Filter ACL phải được áp dụng trước hoặc ngay trong retrieval để context nhạy cảm không đi vào prompt.

---

### 10. Hallucination trong RAG là gì và giảm như thế nào?

**Trả lời:** *Hallucination* là khi LLM đưa ra thông tin không được chứng minh bởi dữ liệu đầu vào hoặc không đúng sự thật nhưng diễn đạt rất tự tin. RAG giúp giảm hallucination, nhưng không tự động loại bỏ hoàn toàn vì model vẫn có thể suy diễn sai từ context hoặc answer khi context thiếu.

**Áp dụng khi nào cần đặc biệt quan tâm?**

- Pháp lý, y tế, tài chính, bảo hiểm và chính sách công ty.
- Tài liệu kỹ thuật cần câu trả lời chính xác theo phiên bản.
- CSKH có điều kiện, điều khoản hoặc cam kết với khách hàng.

**Các biện pháp giảm hallucination:**

1. **Grounded prompting** — yêu cầu LLM chỉ trả lời từ evidence được cấp.
2. **Abstention / refusal có kiểm soát** — trả lời “Không tìm thấy đủ thông tin trong tài liệu hiện có” khi context không đủ.
3. **Citation bắt buộc** — mỗi ý chính cần liên kết đến chunk/file nguồn.
4. **Threshold** — không trả lời chắc chắn khi score retrieval thấp hoặc evidence mâu thuẫn.
5. **Hybrid search và reranking** — tăng chất lượng context trước khi generation.
6. **Version filtering** — chỉ dùng tài liệu còn hiệu lực.
7. **Evaluation** — kiểm tra định kỳ bằng bộ câu hỏi thật và các trường hợp dễ sai.

**So sánh:**

| Chỉ dùng LLM | RAG có guardrail |
| --- | --- |
| Có thể trả lời từ kiến thức tổng quát hoặc suy đoán. | Có evidence cụ thể để bám theo. |
| Khó giải thích vì sao câu trả lời đúng. | Có thể hiển thị nguồn và kiểm tra lại. |
| Dễ nguy hiểm khi câu hỏi chứa dữ liệu nội bộ hoặc thông tin thay đổi. | Phù hợp hơn khi cần traceability. |

**Lưu ý quan trọng:** Nếu retrieval trả về chunk sai, citation vẫn có thể xuất hiện nhưng câu trả lời vẫn sai. Vì vậy, kiểm tra chất lượng retrieval phải được ưu tiên trước khi chỉ tối ưu prompt.

---

### 11. RAG có cần citation không?

**Trả lời:** Có, đặc biệt với hệ thống doanh nghiệp hoặc lĩnh vực cần kiểm chứng. *Citation* là thông tin chỉ ra câu trả lời dựa trên file nào, section nào, trang nào, URL nào hoặc đoạn dữ liệu nào.

**Ví dụ hiển thị:**

> Khách hàng hạng VIP được hoàn 5% điểm cho đơn hàng hợp lệ.  
> Nguồn: `loyalty-policy-2026.pdf`, trang 12, mục “Quy đổi điểm”.

**Áp dụng khi nào?**

- Chatbot nội bộ có tài liệu quy định hoặc quy trình.
- Tư vấn kỹ thuật, pháp lý, tài chính, bảo hiểm và CSKH.
- Người dùng cần mở file gốc để đọc bối cảnh đầy đủ.
- Đội vận hành cần audit hoặc debug các câu trả lời có vấn đề.

**Bài toán giải quyết:** Citation tăng niềm tin, giúp người dùng tự kiểm chứng và giúp đội phát triển biết retrieval đang lấy chunk nào.

| Không citation | Có citation |
| --- | --- |
| Người dùng khó kiểm tra độ đúng. | Người dùng có đường đi về nguồn gốc. |
| Khó debug lỗi do retrieval, chunking hay prompt. | Dễ xác định evidence sai hoặc thiếu. |
| Câu trả lời dễ bị xem là “AI nói vậy”. | Câu trả lời trở thành tóm tắt có căn cứ. |

**Nguyên tắc:** Citation phải khớp với nội dung thực sự hỗ trợ câu trả lời. Không nên chỉ gắn một nguồn chung chung vào cuối đoạn nếu source không chứng minh được mệnh đề cụ thể.

---

### 12. RAG có những kiến trúc phổ biến nào?

**Trả lời:** Kiến trúc RAG nên tăng dần theo độ phức tạp của dữ liệu, rủi ro và yêu cầu chất lượng; không cần dùng agent hoặc graph ngay từ đầu.

| Kiến trúc | Mô tả | Áp dụng khi |
| --- | --- | --- |
| Naive RAG | Chunk → embedding → vector retrieval → LLM. | Prototype, FAQ đơn giản, ít tài liệu. |
| Advanced RAG | Thêm parser tốt, hybrid search, metadata filter, reranking, citation, evaluation và observability. | Production chatbot nội bộ, tài liệu kỹ thuật, CSKH. |
| Agentic RAG | Agent quyết định khi nào search, hỏi lại dữ liệu, gọi API/tool hoặc thực hiện nhiều bước. | Câu hỏi phức tạp cần tổng hợp từ nhiều nguồn và hành động có kiểm soát. |
| Graph RAG | Kết hợp retrieval với knowledge graph/quan hệ thực thể. | Dữ liệu có quan hệ dày đặc như tổ chức, sản phẩm, dependency, quyền hạn hoặc hồ sơ liên kết. |

**Áp dụng thực tế:**

- FAQ chính sách: bắt đầu bằng Naive RAG, sau đó thêm citation và metadata.
- Tài liệu kỹ thuật: Advanced RAG với hybrid search, reranking, version filter và citation theo section.
- Phân tích hồ sơ, điều tra hoặc tổng hợp quy trình nhiều bước: cân nhắc Agentic RAG, nhưng phải giới hạn tool, có log và approval rõ ràng.
- Kho tri thức có quan hệ phức tạp giữa người, hệ thống, hợp đồng và sự kiện: cân nhắc Graph RAG nếu vector search không đủ để trả lời truy vấn theo quan hệ.

**So sánh:** Agentic RAG không tự động tốt hơn Advanced RAG. Agent tăng khả năng xử lý nhiều bước nhưng tăng độ trễ, chi phí, bề mặt lỗi và độ khó kiểm thử. Hãy dùng workflow xác định trước khi bài toán đã biết rõ các bước; chỉ dùng agent khi việc chọn tool/nguồn thật sự biến thiên theo câu hỏi.

---

### 13. Khi nào không nên dùng RAG?

**Trả lời:** Không phải mọi câu hỏi về hệ thống AI đều cần RAG. RAG tốt cho tri thức dạng tài liệu; nó không phải giải pháp thay thế cho query database, API nghiệp vụ, công cụ tính toán hoặc workflow giao dịch.

**Không nên ưu tiên RAG khi:**

- Dữ liệu cần realtime tuyệt đối như trạng thái đơn hàng, tồn kho, số dư hoặc giá hiện tại.
- Cần tính toán chính xác theo công thức, quy tắc thuế hoặc thuật toán định lượng.
- Cần tạo, sửa hoặc duyệt giao dịch trong hệ thống.
- Dữ liệu rất nhỏ, ổn định và có thể đặt trực tiếp vào prompt.
- Câu hỏi là kiến thức phổ thông không cần dữ liệu riêng.

| Bài toán | Giải pháp phù hợp hơn RAG |
| --- | --- |
| “Đơn hàng #123 đang ở đâu?” | Query database hoặc gọi API tracking. |
| “Tính tổng tiền sau giảm giá và VAT.” | Function calling / calculator / service nghiệp vụ. |
| “Đổi địa chỉ giao hàng cho đơn này.” | Workflow + API có xác thực và kiểm soát quyền. |
| “Chính sách đổi trả cho sản phẩm A?” | RAG từ tài liệu chính sách. |
| “Hướng dẫn cấu hình lỗi X của phiên bản Y?” | RAG + hybrid search + filter theo version. |
| “Tổng hợp thay đổi từ nhiều tài liệu.” | RAG, có thể kết hợp workflow hoặc agent có kiểm soát. |

**Nguyên tắc thiết kế:** Với dữ liệu biến động hoặc có tác động nghiệp vụ, hãy lấy *source of truth* từ API/database. Có thể để LLM/RAG giải thích kết quả hoặc hướng dẫn người dùng, nhưng không được suy đoán thay cho nguồn dữ liệu giao dịch.

## Checklist triển khai RAG production

Một hệ thống RAG production cần được xem như một hệ thống search + data pipeline + AI generation, không chỉ là một lời gọi LLM. Các điểm tối thiểu cần có:

- **Data quality:** parser đáng tin cậy, loại bỏ bản trùng, giữ heading/section/trang và version tài liệu.
- **Access control:** ACL/tenant filter trước retrieval; không đưa dữ liệu trái quyền vào prompt.
- **Retrieval quality:** test chunking, embedding, hybrid search, filters và reranking bằng câu hỏi thật.
- **Answer policy:** grounded answer, citation, xử lý “không đủ thông tin” và prompt-injection defense.
- **Evaluation:** theo dõi `Recall@k`, `Precision@k`, groundedness, citation correctness, answer relevance, latency và cost.
- **Observability:** log query đã chuẩn hóa, filter áp dụng, top-k chunks, score, prompt version, model version, latency và lỗi; tránh log dữ liệu nhạy cảm không cần thiết.
- **Operations:** versioning index/document, incremental re-index, retry, backup, rollback và rollout an toàn.

## Gợi ý kiến trúc với Java Spring và Next.js

| Thành phần | Vai trò |
| --- | --- |
| Next.js | Giao diện hỏi đáp, hiển thị citation, feedback và lịch sử chat. |
| Java Spring | API, xác thực/phân quyền, orchestration RAG, audit log, rate limit và tích hợp hệ thống nghiệp vụ. |
| Object storage | Lưu file gốc như PDF, DOCX, HTML export hoặc Markdown. |
| Ingestion worker | Parse, chunk, embed, index và re-index tài liệu. |
| Vector DB / Search engine | Lưu vector, text, metadata; thực hiện vector, keyword hoặc hybrid search. |
| Database chính | Lưu user, tenant, quyền, catalogue tài liệu, version, lịch sử chat và feedback. |
| LLM API hoặc self-hosted model | Sinh câu trả lời có kiểm soát từ context được truy xuất. |
| Evaluation / observability | Đo chất lượng, theo dõi lỗi, chi phí, latency và regression sau mỗi thay đổi. |

## Tóm tắt để nhớ

> **RAG không chỉ là “có vector database”.** Chất lượng của nó phụ thuộc vào dữ liệu đầu vào, chunking, embedding, retrieval, metadata/ACL, reranking, prompt policy, citation và evaluation.

Một RAG tốt phải trả lời được ba câu hỏi: **Có tìm đúng evidence không? Có dùng evidence đó để trả lời đúng không? Có cho người dùng kiểm chứng được không?**

## Liên kết liên quan

- [Chunking](chunking/README.md)
- [Embedding](embedding/README.md)
- [Retrieval](retrieval/README.md)
- [LLM](../README.md)
- [AI](../../README.md)
- [Knowledge Tree](../../../README.md)
