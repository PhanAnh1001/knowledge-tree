# RAG (Retrieval-Augmented Generation)

> [Knowledge Tree](../../../README.md) / [AI](../../README.md) / [LLM](../README.md) / **RAG**

**RAG (Retrieval-Augmented Generation)** là cách kết hợp **truy xuất tri thức** (*retrieval*) với **sinh câu trả lời** (*generation*) bằng Mô hình ngôn ngữ lớn (*Large Language Model, LLM*). Thay vì yêu cầu LLM trả lời hoàn toàn từ kiến thức đã học sẵn, hệ thống tìm bằng chứng phù hợp từ kho dữ liệu tại thời điểm người dùng hỏi, đưa bằng chứng đó vào *context*, rồi mới yêu cầu model trả lời.

> **Question → Retrieve → Select evidence → Generate → Cite / Abstain**

RAG không thay thế database, search engine, API nghiệp vụ hay hệ thống phân quyền. Nó là lớp giúp LLM **tìm đúng bằng chứng, dùng đúng bằng chứng và chỉ rõ nguồn bằng chứng** trước khi trả lời.

## Điều hướng

- **Node cha:** [LLM](../README.md)
- **Node con:** [Chunking](chunking/README.md), [Embedding](embedding/README.md), [Retrieval](retrieval/README.md), [Chỉ mục pgvector](pgvector-index/README.md)
- **Khái niệm trước / sau trong nhánh:** Chưa có.

## Nhánh con

1. [Chunking](chunking/README.md) — Chia tài liệu theo cấu trúc và ngữ cảnh.
2. [Embedding](embedding/README.md) — Biểu diễn query/chunk bằng vector ngữ nghĩa.
3. [Retrieval](retrieval/README.md) — Vector search, hybrid search, metadata filtering và chọn context.
4. [Chỉ mục pgvector](pgvector-index/README.md) — HNSW, IVFFlat, metric, filter/ACL và vận hành vector index trong PostgreSQL.

## Cách đọc trang này

Mỗi câu hỏi ưu tiên bốn góc nhìn: **khái niệm**, **khi áp dụng**, **bài toán giải quyết** và **đánh đổi với lựa chọn tương tự**. Với RAG, đừng chỉ hỏi “dùng vector database nào”; hãy lần lượt kiểm tra:

1. Dữ liệu nguồn có đúng, còn hiệu lực và đủ quyền truy cập không?
2. Hệ thống có retrieve được đúng bằng chứng cho câu hỏi thật không?
3. LLM có trả lời bám sát bằng chứng, biết từ chối khi thiếu dữ kiện và gắn citation đúng không?
4. Độ trễ, chi phí, bảo mật và khả năng vận hành có đáp ứng production không?

---

## A. 5 câu hỏi cốt lõi

### 1. RAG là gì, và nó giải quyết đúng loại bài toán nào?

**Trả lời:** RAG là kỹ thuật cho phép LLM trả lời dựa trên những đoạn dữ liệu được tìm thấy tại thời điểm hỏi. Một hệ thống RAG tối thiểu có sáu thành phần: **nguồn dữ liệu**, **pipeline nạp dữ liệu** (*ingestion*), **chunk**, **index/search**, **LLM**, và **lớp kiểm soát câu trả lời** như citation hoặc từ chối trả lời (*abstention*).

Ví dụ, người dùng hỏi: “Chính sách đổi trả của sản phẩm A cho khách VIP có hiệu lực đến khi nào?” Hệ thống không nên để model suy đoán. Nó cần lọc tài liệu theo `product=A`, `customerTier=VIP`, `status=active`, tìm section chính sách liên quan, rồi trả lời kèm tên file, mục và ngày hiệu lực.

**Dấu hiệu nên dùng RAG:** câu trả lời phụ thuộc vào tài liệu riêng, nội dung thay đổi theo thời gian, người dùng cần kiểm chứng nguồn, hoặc kho dữ liệu quá lớn để đưa nguyên văn vào prompt mỗi lần hỏi.

**Áp dụng khi nào?**

- Chatbot tra cứu chính sách, FAQ, tài liệu nội bộ, knowledge base hoặc wiki.
- Trợ lý kỹ thuật đọc API documentation, runbook, release note, tài liệu kiến trúc và hướng dẫn xử lý lỗi.
- CSKH cần trả lời từ catalogue, bảo hành, điều khoản, quy trình khiếu nại.
- Hệ thống tổng hợp thông tin từ nhiều tài liệu có phiên bản, ngôn ngữ, sản phẩm hoặc phạm vi quyền khác nhau.

**Bài toán giải quyết:**

- LLM không có dữ liệu nội bộ hoặc dữ liệu mới sau thời điểm huấn luyện.
- Cần cập nhật tri thức bằng re-index thay vì huấn luyện lại model.
- Cần giảm câu trả lời bịa đặt (*hallucination*) và tăng khả năng audit.
- Cần tách phần “tri thức thay đổi” khỏi phần “hành vi trả lời” của model.

**So sánh với các lựa chọn tương tự:**

| Cách làm | Điểm mạnh | Giới hạn | Chọn khi |
| --- | --- | --- | --- |
| Prompt tĩnh | Đơn giản, độ trễ thấp. | Chỉ chứa được ít thông tin; khó cập nhật và khó truy vết. | Quy tắc ngắn, ổn định, dùng cho mọi câu hỏi. |
| Search engine | Trả tài liệu/đường dẫn nhanh, dễ để người dùng tự đọc. | Không tự tổng hợp câu trả lời theo hội thoại. | Người dùng cần tìm và đọc tài liệu gốc. |
| RAG | Tìm evidence rồi tổng hợp câu trả lời có nguồn. | Chất lượng phụ thuộc mạnh vào data pipeline và retrieval. | Cần hỏi đáp từ kho tài liệu thay đổi hoặc riêng tư. |
| Fine-tuning | Dạy phong cách, định dạng hoặc hành vi ổn định. | Không phải cơ chế cập nhật tri thức thời gian thực. | Cần model cư xử nhất quán hơn là nhớ tài liệu mới. |
| API/database tool | Lấy số liệu giao dịch hoặc trạng thái realtime từ *source of truth*. | Không phù hợp để tìm ý nghĩa trong văn bản dài. | Trạng thái đơn hàng, tồn kho, số dư, tính giá, tạo giao dịch. |

**Nguyên tắc quyết định:** RAG phù hợp với **knowledge lookup**; API/database phù hợp với **operational truth**. Nhiều sản phẩm tốt kết hợp cả hai: gọi API lấy trạng thái đơn hàng, sau đó dùng RAG để giải thích chính sách giao hàng hoặc điều kiện đổi trả.

---

### 2. Pipeline RAG cơ bản hoạt động như thế nào?

**Trả lời:** Pipeline RAG có hai luồng cần tách rõ: **offline ingestion/indexing** để chuẩn bị tri thức, và **online query/serving** để xử lý một câu hỏi. Tách hai luồng giúp cập nhật tài liệu mà không làm chậm người dùng cuối, đồng thời cô lập lỗi parse/re-index khỏi API chat.

```text
Offline:  Source → Parse/Clean → Version → Chunk → Embed → Index
Online:   Question → Auth/Scope → Rewrite → Retrieve → Rerank → Prompt → Answer + Citation
```

**Luồng ingestion/indexing chi tiết:**

1. **Collect** — lấy tài liệu từ object storage, CMS, wiki, Git, database export hoặc API.
2. **Parse** — lấy text, heading, bảng, code block, trang, URL và các thuộc tính cần trích dẫn.
3. **Normalize** — bỏ boilerplate lặp, chuẩn hóa encoding, nhận diện ngôn ngữ, xử lý duplicate và dữ liệu hết hiệu lực.
4. **Version** — tạo định danh cho document/version; không ghi đè mù quáng một tài liệu đang được dùng.
5. **Chunk** — chia theo section/ngữ cảnh; gắn metadata như `source`, `page`, `section`, `effectiveFrom`, `tenantId`, `allowedRoles`.
6. **Embed** — dùng embedding model tương thích để biến chunk thành vector.
7. **Index** — lưu vector, text gốc, metadata, document version và trạng thái index.
8. **Validate** — kiểm tra số chunk, tỷ lệ parse lỗi, sample text, quyền truy cập và khả năng retrieve trước khi publish version mới.

**Luồng query/serving chi tiết:**

1. Xác thực người dùng và xác định phạm vi truy cập (*tenant*, role, product, region, document status).
2. Chuẩn hóa câu hỏi; có thể rewrite câu hỏi hội thoại thành câu độc lập, ví dụ “thế còn gói cao cấp?” thành “chính sách đổi trả của gói cao cấp là gì?”.
3. Chọn retrieval strategy: keyword, vector, hybrid hoặc gọi tool/API khi câu hỏi là dữ liệu nghiệp vụ.
4. Retrieve nhiều candidate để tối ưu *recall*; áp filter bắt buộc ngay trong truy vấn.
5. Rerank/select để tối ưu *precision* và giữ context trong token budget.
6. Tạo prompt gồm system policy, câu hỏi, evidence có định danh và quy tắc citation/refusal.
7. Kiểm tra output: citation có tồn tại, câu trả lời có vượt evidence, có thông tin nhạy cảm hay không.
8. Trả lời, lưu trace có kiểm soát và nhận feedback của người dùng.

**Áp dụng khi nào?** Dùng pipeline này khi kho dữ liệu lớn hơn prompt, nhiều người cùng dùng một nguồn tri thức, hoặc dữ liệu được cập nhật định kỳ.

**Bài toán giải quyết:**

- Cập nhật tài liệu theo kiểu incremental thay vì embed lại toàn bộ.
- Phân biệt lỗi do parse, chunking, retrieval, prompt hay model.
- Giảm token không liên quan trong mỗi lần hỏi.
- Có thể rollback từ document/index version mới về version cũ.

**So sánh với long context:**

| Long context | RAG |
| --- | --- |
| Đưa một lượng lớn tài liệu vào prompt ở mỗi câu hỏi. | Chỉ đưa evidence đã được chọn vào prompt. |
| Khởi động nhanh với ít tài liệu. | Cần ingestion và index, nhưng tốt hơn khi tài liệu tăng. |
| Có thể tốn token và nhiễu bởi phần không liên quan. | Tiết kiệm context nếu retrieval tốt. |
| Không cần xây search layer. | Có thể filter theo tenant, version, ACL và source. |

Trong production, hai cách có thể bổ sung cho nhau: RAG thu hẹp tập tài liệu xuống một section/chương; long context giữ nhiều đoạn liên quan để LLM tổng hợp sâu hơn.

---

### 3. Embedding trong RAG là gì, và chọn embedding model theo tiêu chí nào?

**Trả lời:** *Embedding* biến text thành vector số sao cho những đoạn gần nhau về nghĩa có vector gần nhau theo một metric như **cosine similarity**, **dot product** hoặc **Euclidean distance**. Vector không “hiểu” đúng/sai như con người; nó chỉ là biểu diễn giúp hệ thống tìm candidate ngữ nghĩa nhanh hơn keyword search thuần túy.

Ví dụ, “hoàn tiền đơn hàng”, “refund”, “đổi trả và nhận lại tiền” có thể không trùng từ nhưng thường gần nhau về nghĩa. Ngược lại, mã `PAY-401` hoặc SKU `A-123` không nên phụ thuộc hoàn toàn vào vector vì chúng cần exact match.

**Các tiêu chí chọn embedding model:**

| Tiêu chí | Câu hỏi cần trả lời | Tác động |
| --- | --- | --- |
| Ngôn ngữ | Dữ liệu và query là tiếng Việt, tiếng Anh hay đa ngôn ngữ? | Model không phù hợp ngôn ngữ làm giảm recall. |
| Loại văn bản | FAQ, tài liệu kỹ thuật, legal policy, code hay catalogue? | Domain-specific wording có thể cần benchmark riêng. |
| Độ dài input | Chunk có thể dài đến bao nhiêu token? | Text bị cắt trước khi embedding sẽ mất điều kiện quan trọng. |
| Kích thước vector | Hạ tầng và chi phí index có chịu được số chiều vector không? | Vector lớn thường tăng storage, RAM và latency. |
| Metric/index | Index đang dùng cosine, dot product hay L2? | Query và document phải dùng cùng quy ước. |
| Chi phí/độ trễ | Embed realtime hay batch; số chunk tăng bao nhanh? | Tác động trực tiếp đến re-index và vận hành. |

**Áp dụng khi nào?**

- Người dùng diễn đạt tự nhiên hoặc dùng từ đồng nghĩa khác với tài liệu.
- Cần semantic search, multilingual search hoặc truy xuất theo ý định.
- Kho tài liệu có nhiều đoạn văn hướng dẫn, chính sách, FAQ hoặc giải thích kỹ thuật.

**Bài toán giải quyết:** Embedding giảm bỏ sót thông tin do khác từ vựng, ví dụ người dùng hỏi “tôi muốn lấy lại tiền” trong khi chính sách chỉ viết “refund”.

**So sánh với keyword search:**

| Keyword / lexical search | Embedding / vector search |
| --- | --- |
| So khớp từ, cụm từ, BM25 hoặc inverted index. | So khớp mức độ gần nghĩa. |
| Mạnh với mã lỗi, SKU, tên riêng, số hiệu điều khoản, phiên bản. | Mạnh với câu hỏi tự nhiên và cách diễn đạt đa dạng. |
| Dễ giải thích, có tính xác định cao. | Có thể tìm đúng ý dù không trùng từ. |
| Dễ bỏ sót đồng nghĩa hoặc câu đa ngôn ngữ. | Có thể trả kết quả gần nghĩa nhưng sai điều kiện cụ thể. |

**Nguyên tắc vận hành:** Không thay embedding model mà giữ nguyên vector cũ trong cùng index. Khi đổi model, metric hoặc dimension, hãy tạo **index version mới**, re-embed toàn bộ dữ liệu liên quan, benchmark trước và có kế hoạch chuyển traffic/rollback.

---

### 4. Vector database trong RAG là gì, và khi nào cần dùng công cụ chuyên dụng?

**Trả lời:** *Vector database* hoặc vector-capable search engine lưu embedding, text gốc và metadata để tìm candidate gần với vector của query. Nó không chỉ là nơi chứa vector; trong RAG production, nó còn phải hỗ trợ filter, phân vùng dữ liệu, index strategy, quan sát latency và kiểm soát quyền truy cập.

Một record index thường cần ít nhất:

```json
{
  "chunkId": "policy-2026:v3:section-4:chunk-2",
  "text": "...",
  "embedding": "<vector>",
  "documentId": "policy-2026",
  "documentVersion": "v3",
  "metadata": {
    "page": 12,
    "section": "Quy đổi điểm",
    "language": "vi",
    "status": "active",
    "effectiveFrom": "2026-01-01",
    "tenantId": "tenant-a",
    "allowedRoles": ["sales", "support"]
  }
}
```

**Các lựa chọn phổ biến:**

| Công cụ / nhóm công cụ | Điểm phù hợp chính | Cân nhắc |
| --- | --- | --- |
| FAISS | Local, nhanh, đơn giản cho prototype hoặc batch retrieval. | Tự xây phần persistence, filter, HA và vận hành. |
| pgvector | Hợp lý khi PostgreSQL đã là nền tảng chính và dữ liệu/vector vừa phải. | Cần benchmark index/filter/concurrency theo tải thực. |
| Qdrant / Weaviate / Milvus | Vector-first, mạnh cho semantic search và metadata filtering. | Thêm một hệ thống dữ liệu mới cần vận hành. |
| Elasticsearch / OpenSearch | Hợp nhất full-text, hybrid search, filter và analytics. | Chi phí/tuning vận hành có thể cao hơn use case vector đơn giản. |
| Managed vector service | Giảm việc vận hành index và HA. | Phụ thuộc nhà cung cấp, chi phí và yêu cầu data residency. |

**Áp dụng khi nào?** Dùng khi có từ hàng nghìn đến hàng triệu chunk, query thường xuyên, cần semantic/hybrid search, hoặc phải filter theo sản phẩm, quyền, phiên bản, vùng, ngôn ngữ và tenant.

**Bài toán giải quyết:**

- Retrieval có độ trễ chấp nhận được khi dữ liệu tăng.
- Filter đúng phạm vi trước khi LLM nhìn thấy context.
- Tách lifecycle của document/index khỏi database giao dịch chính khi cần.

**Cách chọn thực tế:**

- Bắt đầu với **pgvector** khi đội ngũ đã vận hành PostgreSQL, workload chưa lớn và cần giảm hệ thống phụ.
- Cân nhắc **OpenSearch/Elasticsearch** khi full-text, BM25, faceting và hybrid search là trung tâm.
- Cân nhắc **vector DB chuyên dụng** khi vector workload, filtering, collection management hoặc scale độc lập là yêu cầu chính.

Đừng chọn theo benchmark công khai một cách tách rời. Hãy benchmark bằng **số chunk, filter cardinality, top-k, concurrency, latency p95/p99 và bộ câu hỏi thật** của hệ thống.

---

### 5. Chunking trong RAG là gì, và làm sao chia chunk không làm mất điều kiện quan trọng?

**Trả lời:** *Chunking* là chia tài liệu dài thành những đơn vị nhỏ để index và retrieve. Chunk quá nhỏ dễ mất ngữ cảnh; chunk quá lớn làm vector đại diện cho nhiều chủ đề, kéo theo retrieval kém chính xác và tốn token. Mục tiêu không phải “đúng 500 token”, mà là tạo **đơn vị evidence có thể đứng độc lập**.

Ví dụ một chính sách có câu: “Khách VIP được hoàn 5% điểm **cho đơn hàng hợp lệ, không áp dụng với hàng hoàn tiền toàn phần**.” Nếu cắt câu điều kiện sang chunk khác, LLM có thể chỉ lấy phần “hoàn 5% điểm” và trả lời sai.

**Áp dụng khi nào?** Dùng với hầu hết tài liệu dài: PDF, quy định, hợp đồng, manual kỹ thuật, wiki, Markdown nhiều heading, FAQ dài, code documentation và transcript.

**Bài toán giải quyết:**

- Tránh một vector đại diện cho cả tài liệu nhiều chủ đề.
- Tăng khả năng lấy đúng đoạn trả lời thay vì một chương dài.
- Giữ citation theo trang/section/source.
- Kiểm soát context window, latency và chi phí token.

**So sánh các chiến lược chunking:**

| Kiểu chunking | Cách làm | Phù hợp khi | Rủi ro chính |
| --- | --- | --- | --- |
| Fixed-size | Cắt theo token/ký tự, thường có overlap. | Prototype, văn bản đồng nhất. | Cắt giữa heading, bảng, điều kiện hoặc code block. |
| Recursive | Ưu tiên heading → paragraph → sentence → token. | Markdown, HTML, docs có cấu trúc. | Cần cấu hình separator đúng ngôn ngữ/tài liệu. |
| Semantic | Tách tại điểm chuyển chủ đề/ngữ nghĩa. | Văn bản dài, ít structure. | Khó dự đoán, tốn xử lý và cần đánh giá kỹ. |
| Section-aware | Giữ section, điều khoản, bảng, code block, số trang. | Policy, legal, technical doc. | Chunk có thể quá lớn nếu section dài. |
| Parent-child | Retrieve chunk con nhỏ, trả thêm parent/neighbor để đủ context. | Điều kiện, ngoại lệ, tham chiếu chéo. | Tăng phức tạp truy vấn và data model. |

**Quy tắc thực hành theo loại dữ liệu:**

- **Chính sách/pháp lý:** chunk theo điều khoản và heading; giữ số mục, ngày hiệu lực, phạm vi áp dụng, ngoại lệ.
- **Tài liệu kỹ thuật:** không cắt giữa code block, command và phần giải thích liên quan; giữ product/version/module.
- **FAQ:** một câu hỏi + câu trả lời thường là một chunk; không trộn nhiều câu hỏi khác chủ đề.
- **Bảng:** parse cấu trúc bảng thành text có header; gắn page/section và kiểm tra bằng câu hỏi có điều kiện.
- **Transcript:** gắn speaker/timestamp; gom theo chủ đề hoặc lượt hội thoại thay vì cắt thuần theo độ dài.

**Nguyên tắc đánh giá:** Đừng chọn chunk size bằng cảm tính. Lập bộ câu hỏi đại diện, sau đó đo *Recall@k*, citation correctness, answer correctness và cost trước/sau khi đổi parser, chunk size, overlap hoặc metadata.

---

## B. 8 câu hỏi phổ biến mở rộng

### 6. RAG khác Fine-tuning, prompt engineering và tool calling như thế nào?

**Trả lời:** Bốn kỹ thuật này thường bị gộp chung thành “làm chatbot thông minh hơn”, nhưng chúng giải quyết các vấn đề khác nhau.

| Kỹ thuật | Trọng tâm | Ví dụ phù hợp | Không nên kỳ vọng |
| --- | --- | --- | --- |
| Prompt engineering | Hướng dẫn model trong một lần hoặc một workflow. | Yêu cầu format JSON, tone, quy tắc trả lời. | Model tự có kiến thức nội bộ mới. |
| RAG | Lấy evidence từ tài liệu tại thời điểm hỏi. | Hỏi đáp chính sách, manual, wiki. | Trạng thái realtime hoặc phép tính chính xác. |
| Fine-tuning | Điều chỉnh hành vi/pattern từ nhiều ví dụ. | Phân loại, format cố định, tool-use convention. | Cập nhật tài liệu hàng ngày. |
| Tool calling | Gọi API/service/calculator để lấy hay thực thi dữ liệu nghiệp vụ. | Tracking đơn hàng, tạo ticket, tính giá. | Tự hiểu tài liệu dài nếu không có search/RAG. |

**Dùng RAG khi:** kiến thức thay đổi thường xuyên, là dữ liệu riêng, cần citation hoặc có tài liệu quá lớn.

**Dùng fine-tuning khi:** cần model học cấu trúc output, phong cách, taxonomy, cách gọi tool hoặc quy trình rất ổn định và có dataset ví dụ chất lượng.

**Dùng tool calling khi:** câu trả lời cần *source of truth* realtime, tính toán chính xác, hoặc có side effect như tạo ticket/đơn hàng. Hãy để RAG giải thích “chính sách”; để tool/API trả “trạng thái và hành động”.

**Mẫu kết hợp tốt:** Prompt đặt policy → RAG lấy tài liệu → Tool gọi dữ liệu realtime khi cần → LLM giải thích kết quả → citation/audit log.

---

### 7. Retrieval trong RAG là gì? Khi nào dùng hybrid search, filter và query rewrite?

**Trả lời:** *Retrieval* là bước tìm các chunk có khả năng chứa câu trả lời. Đây là phần quyết định chất lượng nhiều hơn việc chỉ đổi LLM. Nếu evidence sai hoặc thiếu, model mạnh vẫn có thể trả lời sai một cách thuyết phục.

**Các chiến lược retrieval:**

| Cách | Cách hoạt động | Mạnh khi |
| --- | --- | --- |
| Keyword / lexical | BM25 hoặc inverted index theo từ/cụm từ. | Error code, SKU, tên riêng, mã điều khoản, version. |
| Vector | Tìm ngữ nghĩa gần với query embedding. | Câu tự nhiên, từ đồng nghĩa, paraphrase. |
| Hybrid | Kết hợp lexical và vector score/ranking. | Tài liệu vừa có mã chính xác vừa có diễn đạt tự nhiên. |
| Metadata-filtered | Giới hạn tập tìm kiếm theo metadata. | Product, version, locale, effective date, tenant, role. |
| Multi-query / rewrite | Viết lại hoặc mở rộng query theo hội thoại/ý định. | Query mơ hồ, đại từ tham chiếu, câu hỏi nhiều điều kiện. |

**Ví dụ:**

- “Lỗi `PAY-401` xử lý thế nào?” → keyword search hoặc hybrid nên được ưu tiên.
- “Không đăng nhập được sau khi đổi mật khẩu” → vector/hybrid giúp nhận diện ngữ nghĩa.
- “Chính sách hoàn điểm cho Việt Nam năm nay?” → filter `region=VN`, `status=active`, `effectiveFrom <= now` trước; sau đó mới semantic search.

**Áp dụng khi nào?** Hybrid search là mặc định đáng cân nhắc khi tài liệu chứa mã, tên tính năng, điều kiện nghiệp vụ hoặc thuật ngữ nội bộ. Vector-only dễ bỏ exact match; keyword-only dễ bỏ cách diễn đạt tự nhiên.

**Bài toán giải quyết:** tăng *recall* của evidence đúng, giảm lấy nhầm tài liệu sai version/sản phẩm, và giảm context nhiễu trước khi generation.

**Nguyên tắc filter:** ACL, tenant, document status và version là **ràng buộc cứng** (*hard constraints*), không phải tín hiệu “xếp hạng thấp hơn”. Chúng phải được áp dụng trước hoặc ngay trong retrieval, không phải sau khi context đã vào prompt.

---

### 8. Reranking trong RAG là gì, và đánh đổi latency ra sao?

**Trả lời:** *Reranking* đánh giá lại các candidate sau retrieval để đưa đoạn phù hợp nhất lên đầu. Retriever thường tối ưu tốc độ trên hàng nghìn/hàng triệu chunk; reranker đọc kỹ hơn cặp `query + chunk`, nên thường chính xác hơn nhưng tốn chi phí hơn.

```text
Retrieve top 30–100 candidates → Rerank → Select top 3–10 evidence → Generate
```

Các con số trên chỉ là điểm bắt đầu để benchmark; không phải cấu hình cố định.

**Áp dụng khi nào?**

- Top-k retrieval thường đúng chủ đề nhưng không trả lời trực tiếp.
- Tài liệu có nhiều section gần nghĩa hoặc nhiều version tương tự.
- Câu hỏi có điều kiện/ngoại lệ và hệ thống yêu cầu độ chính xác cao.
- Khả năng chịu thêm latency có kiểm soát.

**Bài toán giải quyết:** tăng *precision* của context, giảm LLM phải đọc đoạn liên quan lỏng lẻo, và tăng xác suất lấy được đoạn chứa đáp án thật sự.

| Không reranking | Có reranking |
| --- | --- |
| Ít bước, latency thấp hơn. | Tốn thêm model call/compute. |
| Dễ phù hợp prototype hoặc FAQ rất đơn giản. | Phù hợp production khi retrieval ban đầu còn nhiều nhiễu. |
| Có thể đưa nhiều chunk “đúng chủ đề” nhưng thiếu đáp án. | Ưu tiên chunk trả lời trực tiếp hoặc chứa điều kiện cần thiết. |

**Lưu ý:** Reranker không cứu được evidence không được retrieve. Hãy xử lý parser, chunking, embedding, keyword/hybrid và metadata filter trước. Reranking chỉ chọn tốt hơn trong candidate set hiện có.

---

### 9. Metadata, ACL và multi-tenancy trong RAG để làm gì?

**Trả lời:** *Metadata* mô tả chunk; **ACL (Access Control List)** quy định ai được truy cập chunk đó. Trong doanh nghiệp, metadata không chỉ để hiển thị citation; nó là phần của **correctness** và **security**.

Ví dụ metadata:

```json
{
  "source": "loyalty-policy-2026.pdf",
  "documentVersion": "v3",
  "page": 12,
  "section": "Quy đổi điểm",
  "product": "membership",
  "region": "VN",
  "effectiveFrom": "2026-01-01",
  "status": "active",
  "tenantId": "tenant-a",
  "allowedRoles": ["sales", "support"]
}
```

**Áp dụng khi nào?**

- Có nhiều phiên bản, thị trường, sản phẩm hoặc công ty con.
- Hệ thống phục vụ multi-tenant hoặc phân quyền theo role/team/user.
- Cần trả lời theo thời điểm hiệu lực và giữ đường dẫn đến file/trang/section.
- Dữ liệu có mức độ nhạy cảm khác nhau.

**Bài toán giải quyết:**

- Không trộn chính sách cũ với chính sách còn hiệu lực.
- Không lộ dữ liệu của tenant/phòng ban khác.
- Có trace để debug chunk từ đâu, thuộc version nào và vì sao được retrieve.

| Chỉ vector search | Vector + metadata + ACL |
| --- | --- |
| Có thể lấy text gần nghĩa nhưng sai phạm vi. | Có thể giới hạn theo product, version, region, date và role. |
| Khó xử lý multi-tenant an toàn. | Tách tenant và enforce quyền từ đầu. |
| Citation nghèo thông tin. | Có file, section, page, URL, hiệu lực và version. |

**Nguyên tắc bảo mật:** `deny by default`. Tất cả chunk phải được gắn scope rõ ràng; request không có scope hợp lệ không được retrieve. Không chỉ “ẩn citation” hoặc kiểm quyền sau generation, vì dữ liệu nhạy cảm có thể đã đi vào prompt hoặc log.

---

### 10. Hallucination trong RAG là gì, và làm sao giảm theo từng tầng?

**Trả lời:** *Hallucination* là câu trả lời không được chứng minh bởi evidence, hoặc suy diễn vượt quá evidence. RAG làm giảm rủi ro nhưng không tự loại bỏ hallucination: model vẫn có thể hiểu sai context, trộn hai tài liệu, bỏ ngoại lệ, hoặc trả lời tự tin khi không có evidence.

**Ba dạng thường gặp:**

1. **Unsupported claim** — nêu một dữ kiện không có trong source.
2. **Over-generalization** — lấy quy tắc cho một sản phẩm/đối tượng rồi áp cho tất cả.
3. **Contradiction hiding** — source có mâu thuẫn hoặc nhiều version, nhưng model chọn một câu trả lời mà không nói rõ điều kiện.

**Giảm hallucination theo tầng:**

| Tầng | Biện pháp |
| --- | --- |
| Data | Loại tài liệu cũ/duplicate, version rõ ràng, parse bảng/heading đúng. |
| Retrieval | Hybrid search, hard filter, reranking, threshold và context diversity. |
| Prompt | Yêu cầu chỉ dùng evidence; nêu điều kiện/exceptions; không suy đoán. |
| Generation | Bắt buộc citation cho mệnh đề chính; từ chối khi evidence thiếu. |
| Validation | Kiểm tra citation tồn tại, answer-groundedness, policy violation và dữ liệu nhạy cảm. |
| Evaluation | Dùng bộ câu hỏi thật, câu bẫy, câu ngoài phạm vi và regression test. |

**Mẫu câu trả lời an toàn:** “Tôi chưa tìm thấy đủ thông tin trong các tài liệu đang được phép truy cập để kết luận. Bạn có thể cung cấp mã sản phẩm hoặc tài liệu liên quan không?”

**So sánh:**

| Chỉ dùng LLM | RAG có guardrail |
| --- | --- |
| Có thể trả lời từ kiến thức tổng quát hoặc suy đoán. | Có evidence cụ thể để bám theo. |
| Khó giải thích vì sao câu trả lời đúng. | Có citation để kiểm chứng và debug. |
| Không có giới hạn phạm vi dữ liệu. | Có thể áp ACL/version/scope trước retrieval. |

**Lưu ý:** Citation không tự chứng minh correctness. Citation có thể đúng file nhưng không hỗ trợ đúng mệnh đề. Vì vậy cần đo **citation correctness** và **groundedness**, không chỉ đo có/không có citation.

---

### 11. RAG có cần citation không? Thiết kế citation thế nào để người dùng kiểm chứng được?

**Trả lời:** Citation rất nên có trong hệ thống doanh nghiệp, đặc biệt khi câu trả lời ảnh hưởng tới khách hàng, quy trình, compliance hoặc quyết định kỹ thuật. Citation cần trỏ về **evidence cụ thể**, không chỉ tên một thư mục chung chung.

**Ví dụ hiển thị:**

> Khách hàng hạng VIP được hoàn 5% điểm cho đơn hàng hợp lệ; hàng hoàn tiền toàn phần không được áp dụng.  
> Nguồn: `loyalty-policy-2026.pdf`, v3, trang 12, mục “Quy đổi điểm”.

**Áp dụng khi nào?**

- Chính sách nội bộ, pháp lý, tài chính, bảo hiểm và CSKH.
- Trợ lý kỹ thuật cần chỉ chính xác release, module hoặc section tài liệu.
- Hệ thống cần audit, review hoặc sửa lỗi do retrieval.

**Bài toán giải quyết:** tăng niềm tin, giúp người dùng tự đọc bối cảnh đầy đủ, và giúp đội phát triển biết query đã retrieve chunk nào.

**Nguyên tắc thiết kế citation:**

1. Citation gắn gần mệnh đề mà nó chứng minh; tránh một citation chung ở cuối đoạn quá dài.
2. Lưu `documentId`, version, section, page/anchor và URL ổn định ngay từ ingestion.
3. Citation phải qua ACL tương tự text; không lộ tên file nhạy cảm cho người không có quyền.
4. Khi evidence mâu thuẫn, hiển thị điều kiện/version thay vì ép model kết luận một chiều.
5. Cho phép người dùng mở đúng section/source gốc nếu UI có quyền hiển thị.

| Không citation | Có citation |
| --- | --- |
| Người dùng khó kiểm tra độ đúng. | Có đường đi về nguồn gốc. |
| Khó debug lỗi do retrieval/chunking/prompt. | Trace được evidence và version. |
| Dễ trở thành “AI nói vậy”. | Trở thành bản tóm tắt có căn cứ. |

---

### 12. Có những kiến trúc RAG nào, và nên tăng độ phức tạp theo thứ tự nào?

**Trả lời:** Kiến trúc RAG nên tăng dần theo rủi ro, chất lượng dữ liệu và yêu cầu vận hành. Agent hay graph không tự động tốt hơn; chúng chỉ thêm khả năng, đồng thời tăng latency, chi phí và bề mặt lỗi.

| Kiến trúc | Thành phần chính | Phù hợp khi | Điểm cần cảnh giác |
| --- | --- | --- | --- |
| Naive RAG | Chunk → embed → vector retrieve → LLM. | Prototype, FAQ đơn giản, kho nhỏ. | Thiếu filter, citation, eval và kiểm soát version. |
| Advanced RAG | Parser tốt, hybrid, metadata/ACL, rerank, citation, evaluation. | Production chatbot nội bộ, docs kỹ thuật, CSKH. | Cần ownership cho data pipeline và đánh giá chất lượng. |
| Tool-augmented RAG | RAG + API/database/calculator. | Cần vừa giải thích tài liệu vừa lấy dữ liệu realtime. | Phải phân biệt knowledge lookup và transactional action. |
| Agentic RAG | Agent chọn search/tool, phân rã nhiều bước, tổng hợp. | Câu hỏi phức tạp, đường đi không xác định trước. | Khó kiểm thử, dễ loop, tăng cost/latency. |
| Graph RAG | Retrieval kết hợp knowledge graph/thực thể/quan hệ. | Query phụ thuộc quan hệ dày đặc hoặc multi-hop. | Xây và duy trì graph có chi phí cao; vector search đôi khi đủ. |

**Lộ trình an toàn:**

1. Naive RAG với citation tối thiểu và bộ câu hỏi đánh giá.
2. Thêm version, metadata filter, ACL, hybrid và observability.
3. Thêm reranking khi retrieval đủ candidate nhưng precision top-k chưa tốt.
4. Kết hợp tool/API cho dữ liệu realtime hoặc hành động nghiệp vụ.
5. Chỉ dùng agent/graph sau khi xác định rõ điểm nghẽn mà workflow xác định trước không giải quyết được.

**So sánh workflow và agent:** Nếu chuỗi bước đã biết rõ, ví dụ “xác thực → lọc tenant → search policy → gọi API đơn hàng → trả lời”, hãy dùng workflow xác định trước. Agent chỉ nên chọn đường đi khi query thực sự biến thiên, nhưng cần giới hạn số bước/tool, timeout, logging và approval cho hành động có tác động.

---

### 13. Khi nào không nên dùng RAG, hoặc nên dùng RAG cùng giải pháp khác?

**Trả lời:** RAG tốt cho tri thức dạng tài liệu. Nó không thay thế query database, API giao dịch, calculator, rule engine hoặc workflow có kiểm soát quyền.

**Không nên ưu tiên RAG khi:**

- Dữ liệu cần realtime tuyệt đối: trạng thái đơn hàng, tồn kho, số dư, giá hiện tại.
- Cần tính toán chính xác theo công thức, thuế, tỷ giá hoặc thuật toán định lượng.
- Cần tạo/sửa/duyệt giao dịch.
- Dữ liệu rất nhỏ, ổn định, có thể đặt trực tiếp vào prompt.
- Câu hỏi phổ thông không cần dữ liệu riêng và không cần citation.

| Bài toán | Giải pháp phù hợp hơn RAG | Vai trò RAG nếu có |
| --- | --- | --- |
| “Đơn hàng #123 đang ở đâu?” | Query database hoặc API tracking. | Giải thích trạng thái/điều kiện giao hàng. |
| “Tính tiền sau giảm giá và VAT.” | Calculator/service nghiệp vụ. | Giải thích công thức hoặc chính sách khuyến mãi. |
| “Đổi địa chỉ giao hàng.” | Workflow + API có xác thực. | Hướng dẫn điều kiện được phép đổi. |
| “Chính sách đổi trả sản phẩm A?” | RAG. | Nguồn tri thức chính. |
| “Lỗi X ở version Y xử lý thế nào?” | RAG + hybrid + version filter. | Nguồn tri thức chính. |
| “So sánh thay đổi giữa hai policy.” | RAG + document diff/workflow. | Retrieve evidence từ cả hai version. |

**Nguyên tắc thiết kế:** Với dữ liệu biến động hoặc hành động nghiệp vụ, luôn lấy sự thật từ *source of truth* là API/database/rule engine. Có thể để LLM/RAG diễn giải kết quả, nhưng không cho model suy đoán thay nguồn dữ liệu giao dịch.

---

## Checklist triển khai RAG production

Một RAG production là **search system + data pipeline + access-control system + generation layer**, không chỉ là một lời gọi LLM. Các điểm tối thiểu cần có:

- **Data quality:** parser đáng tin cậy; giữ heading, bảng, trang, URL, version và hiệu lực; loại bản trùng/hết hạn.
- **Document lifecycle:** content hash, trạng thái draft/published/retired, incremental re-index, rollback và audit version.
- **Access control:** tenant/ACL filter trước retrieval; không đưa dữ liệu trái quyền vào prompt, cache, trace hoặc citation.
- **Retrieval quality:** benchmark chunking, embedding, hybrid, filter và reranking bằng câu hỏi thật.
- **Answer policy:** grounded answer, citation, xử lý “không đủ thông tin”, prompt-injection defense và giới hạn side effect.
- **Evaluation:** đo retrieval, answer, citation, latency, cost và regression sau mọi thay đổi lớn.
- **Observability:** log request scope, query normalized, filter, top candidates, score/rank, prompt/model/index version, latency; giảm thiểu log dữ liệu nhạy cảm.
- **Operations:** retry có idempotency, queue/backpressure cho ingestion, backup, monitoring index health và rollout an toàn.

### Bộ chỉ số nên theo dõi

| Nhóm | Chỉ số | Ý nghĩa |
| --- | --- | --- |
| Retrieval | `Recall@k`, `MRR`, `nDCG`, filter pass rate | Evidence đúng có xuất hiện đủ sớm không? |
| Generation | answer correctness, relevance, groundedness | Câu trả lời có đúng, hữu ích, bám evidence không? |
| Citation | citation coverage, citation correctness | Có citation và citation có thực sự chứng minh mệnh đề không? |
| Operations | p50/p95/p99 latency, error rate, queue lag | Hệ thống có ổn định dưới tải không? |
| Cost | token/query, embedding/re-index cost, storage/index cost | Có thể mở rộng chi phí bền vững không? |
| Safety | ACL violation attempts, refusal rate, prompt-injection flags | Guardrail có hoạt động đúng không? |

### Bộ câu hỏi đánh giá tối thiểu

Tập đánh giá nên có các nhóm sau, thay vì chỉ dùng câu hỏi “dễ và đúng”:

1. Câu hỏi có đáp án nằm trong một chunk.
2. Câu hỏi cần kết hợp hai section cùng một tài liệu.
3. Câu hỏi có mã lỗi/SKU/tên phiên bản chính xác.
4. Câu hỏi dùng từ đồng nghĩa, tiếng Việt không dấu hoặc pha tiếng Anh.
5. Câu hỏi có điều kiện/ngoại lệ.
6. Câu hỏi về policy cũ để kiểm tra version filter.
7. Câu hỏi ngoài phạm vi để kiểm tra abstention.
8. Câu hỏi của tenant/role không đủ quyền để kiểm tra ACL.

---

## Gợi ý kiến trúc với Java Spring và Next.js

| Thành phần | Vai trò | Lưu ý triển khai |
| --- | --- | --- |
| Next.js | UI hỏi đáp, hiển thị citation, feedback, lịch sử chat. | Không tin cậy scope/ACL do client gửi; backend quyết định. |
| Java Spring | API, authN/authZ, orchestration RAG, audit log, rate limit, tool routing. | Tách request path khỏi ingestion worker; đặt timeout và fallback rõ ràng. |
| Object storage | Lưu PDF, DOCX, HTML export, Markdown gốc. | Lưu document version immutable hoặc có revision rõ ràng. |
| Ingestion worker | Parse, normalize, chunk, embed, index, validate, publish. | Idempotent theo content hash; queue để tránh block API. |
| Vector DB / search engine | Vector, lexical, hybrid, filter và retrieval. | Enforce tenant/ACL/document status ở query layer. |
| Database chính | User, tenant, quyền, document catalog, version, feedback, trace metadata. | Không dùng chat memory thay cho knowledge source. |
| LLM API / self-hosted model | Sinh câu trả lời từ evidence đã chọn. | Prompt chỉ nhận context đã qua kiểm soát; ghi rõ answer policy. |
| Evaluation / observability | Dataset, dashboard, trace, regression và alert. | Tránh lưu raw sensitive content lâu hơn cần thiết. |

### Luồng request gợi ý

```text
Browser (Next.js)
  → Spring API Gateway / RAG Service
    → Authenticate + resolve tenant/role
    → Query classifier: RAG hay Tool/API?
    → Retrieval with ACL + version + metadata filters
    → Optional reranking
    → LLM with evidence IDs
    → Output validation + citations
  ← Answer, sources, confidence/abstention state
```

### Contract dữ liệu nên thống nhất

| Entity | Trường nên có |
| --- | --- |
| Document | `documentId`, `title`, `sourceUrl`, `owner`, `status`, `language`, `createdAt` |
| DocumentVersion | `versionId`, `documentId`, `contentHash`, `effectiveFrom`, `effectiveTo`, `indexedAt` |
| Chunk | `chunkId`, `versionId`, `text`, `headingPath`, `page`, `chunkOrder`, `parentChunkId` |
| Scope | `tenantId`, `allowedRoles`, `product`, `region`, `classification` |
| RetrievalTrace | `requestId`, `query`, `filters`, `indexVersion`, `candidateIds`, `ranks`, `latencyMs` |
| Citation | `chunkId`, `documentId`, `versionId`, `section`, `page`, `sourceUrl` |

---

## Lộ trình triển khai thực tế

| Giai đoạn | Mục tiêu | Kết quả cần kiểm chứng |
| --- | --- | --- |
| 1. Baseline | Một nguồn dữ liệu, parse/chunk/index cơ bản, citation. | Retrieve đúng evidence cho bộ câu hỏi nền tảng. |
| 2. Trust | Versioning, metadata, ACL, refusal và trace. | Không lộ chéo tenant; policy cũ không xuất hiện. |
| 3. Quality | Hybrid, rerank, query rewrite, evaluation dashboard. | Recall/precision và answer quality cải thiện có đo lường. |
| 4. Integrate | Kết hợp API/tool, feedback loop, rollout/rollback. | Phân biệt rõ knowledge lookup với transactional action. |

Đừng bỏ qua giai đoạn baseline: một hệ thống nhỏ nhưng có dataset đánh giá, citation và trace thường đáng tin hơn một kiến trúc agent phức tạp nhưng không đo được retrieval.

## Tóm tắt để nhớ

> **RAG không chỉ là “có vector database”.** Chất lượng của nó phụ thuộc vào dữ liệu đầu vào, chunking, embedding, retrieval, metadata/ACL, reranking, answer policy, citation, evaluation và vận hành.

Một RAG tốt phải trả lời được ba câu hỏi:

1. **Có tìm đúng evidence trong đúng phạm vi quyền và phiên bản không?**
2. **Có dùng evidence đó để trả lời đúng, không suy đoán vượt dữ liệu không?**
3. **Có giúp người dùng kiểm chứng và giúp đội ngũ debug được không?**

## Liên kết liên quan

- [Chunking](chunking/README.md)
- [Embedding](embedding/README.md)
- [Retrieval](retrieval/README.md)
- [Chỉ mục pgvector](pgvector-index/README.md)
- [LLM](../README.md)
- [AI](../../README.md)
- [Knowledge Tree](../../../README.md)
