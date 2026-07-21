# Knowledge Tree

Kho kiến thức bằng tiếng Việt, tổ chức theo cây phân cấp (*mind-map tree*).

Mỗi khái niệm có:

- Tối đa 5 câu hỏi và trả lời cốt lõi.
- Tối đa 8 câu hỏi và trả lời phổ biến mở rộng.
- Thuật ngữ tiếng Anh khi phù hợp.
- Hướng dẫn áp dụng, bài toán giải quyết và các liên kết liên quan.

## Quy tắc biên soạn nội dung Q&A

Mỗi README khái niệm phải trình bày nội dung theo dạng hỏi–đáp có chiều sâu, ưu tiên câu hỏi thực tế thay vì chỉ nêu định nghĩa.

1. **Mỗi Q&A cần có câu trả lời rõ ràng và đủ ngữ cảnh.** Giải thích thuật ngữ tiếng Anh ở lần xuất hiện đầu tiên, ví dụ: *Retrieval-Augmented Generation (RAG)*.
2. **Phải nêu trường hợp áp dụng** bằng các tình huống, loại dữ liệu hoặc nhu cầu thực tế phù hợp với khái niệm.
3. **Phải nêu bài toán được giải quyết khi có thể**, như giới hạn kỹ thuật, rủi ro vận hành, chất lượng, chi phí hoặc trải nghiệm người dùng mà khái niệm giúp cải thiện.
4. **Phải so sánh với lựa chọn tương tự khi có ý nghĩa**, làm rõ điểm mạnh, giới hạn và điều kiện chọn giải pháp; ưu tiên bảng so sánh khi có từ hai lựa chọn trở lên.
5. **Nhóm cốt lõi** giải thích nền tảng để người mới hiểu và áp dụng được. **Nhóm mở rộng** trả lời các câu hỏi thường gặp về lựa chọn công nghệ, đánh đổi, vận hành, đánh giá chất lượng hoặc trường hợp không nên dùng.
6. Không thêm câu hỏi để đủ số lượng. Chỉ chọn các câu hỏi có giá trị học tập hoặc hỗ trợ ra quyết định; số lượng tối đa là giới hạn, không phải bắt buộc.
7. Nội dung cần nhất quán với các node con và ưu tiên liên kết tới khái niệm liên quan thay vì lặp lại phần giải thích chuyên sâu đã có ở node đó.
8. Với khái niệm là một **pipeline hoặc hệ thống**, cần phân biệt luồng chuẩn bị dữ liệu (*offline/ingestion*) với luồng phục vụ người dùng (*online/serving*), nêu ranh giới giữa các thành phần và các điểm cần kiểm thử hoặc quan sát.
9. Với nội dung có rủi ro bảo mật, dữ liệu thay đổi hoặc tác động nghiệp vụ, phải chỉ rõ *source of truth*, phạm vi quyền truy cập, điều kiện từ chối/trả lời và giới hạn của giải pháp.

## Cấu trúc

```text
knowledge-tree/
├── README.md
├── metadata.json
└── {domain}/{category}/.../{concept}/README.md
```

Tên thư mục dùng chữ thường, không dấu. Khi thuật ngữ có từ viết tắt tiếng Anh phổ biến trong ngành, URL dùng chính từ viết tắt đó; nếu không có, dùng tên đầy đủ dạng `kebab-case`.

## Khám phá cây kiến thức

- [Trí tuệ nhân tạo (Artificial Intelligence, AI)](ai/README.md)
  - [Mô hình ngôn ngữ lớn (Large Language Models, LLM)](ai/llm/README.md)
    - [RAG (Retrieval-Augmented Generation)](ai/llm/rag/README.md)
      - [Chunking](ai/llm/rag/chunking/README.md)
      - [Embedding](ai/llm/rag/embedding/README.md)
      - [Retrieval](ai/llm/rag/retrieval/README.md)
      - [Recall trong RAG](ai/llm/rag/recall/README.md)
      - [Chỉ mục pgvector (pgvector Index)](ai/llm/rag/pgvector-index/README.md)
      - [Các trường hợp ứng dụng RAG (RAG Use Cases)](ai/llm/rag/use-cases/README.md)
      - [RAGAS trong RAG](ai/llm/rag/ragas/README.md)
      - [ETL tài liệu PDF kỹ thuật cho RAG](ai/llm/rag/etl/README.md)
    - [Chain of Thought (CoT)](ai/llm/cot/README.md)
    - [Minimal Reproducible Example (MRE) cho AI và OCR tài liệu kỹ thuật](ai/llm/mre/README.md)

## Cập nhật gần đây

- **2026-07-21:** [ETL tài liệu PDF kỹ thuật cho RAG](ai/llm/rag/etl/README.md) được thêm mới với kiến trúc router theo file/trang, fast parser–OCR–layout–form–VLM fallback, canonical document model, queue-driven workers, idempotency, versioning, ACL, quality gate, chunking, hybrid search và lộ trình scale từ 50.000 đến 1.000.000 PDF.
- **2026-07-18:** [Minimal Reproducible Example (MRE) cho AI và OCR tài liệu kỹ thuật](ai/llm/mre/README.md) được thêm mới với cách rút PDF lớn thành case nhỏ, khóa prompt/model/OCR, định nghĩa expected–actual cho đề mục và hình ảnh, xử lý tài liệu máy móc đa ngành, bảo mật dữ liệu, đo lỗi không xác định và chuyển MRE thành regression suite.
- **2026-07-10:** [Chain of Thought (CoT)](ai/llm/cot/README.md) được thêm mới với nền tảng CoT, Few-shot/Zero-shot CoT, Self-Consistency, cách chọn giữa Prompt Chaining, Least-to-Most, Tree of Thoughts, ReAct và Program of Thoughts, kết hợp RAG/tool calling, giới hạn về faithfulness và checklist đánh giá production.
- **2026-07-08:** [RAGAS trong RAG](ai/llm/rag/ragas/README.md) được thêm mới với định nghĩa RAGAS/Ragas, dữ liệu evaluation sample, Context Precision, Context Recall, Faithfulness, Answer Relevancy, testset generation, threshold, đánh giá tiếng Việt/đa ngôn ngữ và cách tích hợp vào Java Spring/Next.js.
- **2026-07-08:** [Recall trong RAG](ai/llm/rag/recall/README.md) được thêm mới với định nghĩa Recall/Recall@k, cách đo bằng evaluation dataset, quan hệ với precision/MRR/nDCG, cách tăng recall bằng hybrid search, query rewrite, chunking, metadata filter, đánh giá tiếng Việt/đa ngôn ngữ và theo dõi production.
- **2026-07-07:** [Các trường hợp ứng dụng RAG](ai/llm/rag/use-cases/README.md) bổ sung bản đồ use case theo mục tiêu, ngành, loại dữ liệu, mức rủi ro và kiến trúc; bao gồm enterprise knowledge, CSKH, developer/SRE, sales/e-commerce, legal/compliance, finance/insurance, healthcare, education, operations/logistics, research và multimodal/graph/agentic RAG.
- **2026-07-06:** [Retrieval trong RAG](ai/llm/rag/retrieval/README.md) được mở rộng với pipeline offline/online, dense–sparse–hybrid retrieval, ACL và versioning, reranking/MMR/threshold, query rewrite, grounded abstention, đánh giá, quan sát production và chọn FAISS, pgvector, Qdrant hoặc Elasticsearch/OpenSearch.
- **2026-07-06:** [Embedding trong RAG](ai/llm/rag/embedding/README.md) được mở rộng với không gian vector tương thích, chọn model tiếng Việt/chuyên ngành, text và metadata để embedding, metric/normalization/index, hybrid search, đánh giá, re-embed migration, đa ngôn ngữ, cache và ACL.
- **2026-07-06:** [Chunking trong RAG](ai/llm/rag/chunking/README.md) được mở rộng với chiến lược chọn ranh giới chunk, `chunkSize`/overlap, metadata và ACL, tài liệu phức tạp (PDF/bảng/code), đánh giá retrieval, context assembly và vận hành re-index theo version.

## Quy ước thuật ngữ và URL

Dùng các từ viết tắt tiếng Anh phổ biến trong ngành khi chúng là tên khái niệm hoặc định danh ổn định, ví dụ: `ai`, `llm`, `rag`, `api`, `db`, `ui`, `ux`.

- Giải thích tên đầy đủ ở lần xuất hiện đầu tiên, ví dụ: *Retrieval-Augmented Generation (RAG)*.
- Không tự tạo từ viết tắt mơ hồ.
- Nếu một thuật ngữ có từ viết tắt chuyên ngành phổ biến, **phải dùng** dạng viết tắt chữ thường cho `id`, `slug`, tên thư mục và URL: `ai/llm/rag`.
- Nếu không có từ viết tắt phổ biến, dùng tên đầy đủ dạng `kebab-case`, ví dụ `clean-architecture`.
- Không rút gọn tên field của `metadata.json`; chỉ rút gọn định danh của khái niệm khi đó là thuật ngữ chuẩn.

## Metadata

`metadata.json` ở thư mục gốc là nguồn dữ liệu của cây kiến thức. Các field trong file này dùng tên đầy đủ, rõ nghĩa; không rút gọn chỉ để giảm số ký tự.

Các field cấp root:

| Field | Ý nghĩa |
| --- | --- |
| `version` | Phiên bản schema metadata. |
| `locale` | Ngôn ngữ hiển thị mặc định, ví dụ `vi-VN`. |
| `rootId` | ID của node gốc. |
| `description` | Mô tả ngắn cho metadata. |
| `nodeSchema` | Quy ước field và loại node được hỗ trợ. |
| `nodes` | Danh sách toàn bộ node trong cây. |
| `pathIndex` | Ánh xạ từ đường dẫn README tới `id` của node. |

Các field của một node:

| Field | Ý nghĩa |
| --- | --- |
| `id` | Định danh duy nhất, có thể dùng từ viết tắt phổ biến như `ai`, `llm`, `rag`. |
| `title` | Tên hiển thị bằng tiếng Việt, kèm thuật ngữ tiếng Anh khi cần. |
| `slug` | Segment URL và tên thư mục của node. |
| `type` | Loại node: `root`, `domain`, `category` hoặc `concept`. |
| `path` | Đường dẫn tới `README.md` của node. |
| `parentId` | ID của node cha; node gốc dùng `null`. |
| `order` | Thứ tự hiển thị giữa các node cùng cấp. |
| `children` | Danh sách ID các node con. |
| `updatedAt` | Ngày cập nhật nội dung quan trọng gần nhất của node, theo định dạng `YYYY-MM-DD`; là field tùy chọn. |

Metadata mô tả cấu trúc cây. Giao diện hoặc công cụ có thể suy ra node cha, node con và thứ tự cùng cấp từ `parentId`, `children` và `order`. `updatedAt` giúp hiển thị hoặc kiểm tra độ mới của nội dung mà không thay thế lịch sử commit của Git.

## Điều hướng (Navigation)

Điều hướng là một phần nội dung trong từng `README.md` của khái niệm, không chỉ là dữ liệu trong `metadata.json`.

Mỗi README khái niệm cần có mục `## Điều hướng` với:

1. **Breadcrumb**: liên kết từ Knowledge Tree đến node hiện tại.
2. **Node cha**: liên kết về node cha.
3. **Node con**: danh sách liên kết tới các node con, khi có.
4. **Khái niệm trước / sau**: liên kết tới node liền kề cùng cấp, theo `order`.

Ví dụ:

```md
> [Knowledge Tree](../../../README.md) / [AI](../../README.md) / [LLM](../README.md) / **RAG**

## Điều hướng

- **Node cha:** [LLM](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** Chưa có.
- **Khái niệm sau trong cùng nhánh:** Chưa có.
```

Các liên kết trong phần điều hướng phải khớp với cấu trúc được khai báo bằng `parentId`, `children` và `order` trong `metadata.json`.

## Đóng góp

Khi thêm, di chuyển hoặc cập nhật đáng kể một khái niệm, hãy cập nhật README của khái niệm, các liên kết điều hướng có ảnh hưởng và `metadata.json` trong cùng thay đổi hợp lý.
