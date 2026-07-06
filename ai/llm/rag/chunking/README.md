# Chunking trong RAG

> [Knowledge Tree](../../../../README.md) / [AI](../../../README.md) / [LLM](../../README.md) / [RAG](../README.md) / **Chunking**

**Chunking** là quá trình chia tài liệu thành các đơn vị nội dung (*chunks*) đủ nhỏ để truy xuất chính xác nhưng vẫn đủ ngữ cảnh để mô hình ngôn ngữ lớn (*Large Language Model, LLM*) hiểu, trích dẫn và trả lời đúng. Trong *Retrieval-Augmented Generation (RAG)*, đây là quyết định thiết kế ảnh hưởng trực tiếp tới chất lượng retrieval, độ tin cậy citation, độ trễ và chi phí.

## Điều hướng

- **Node cha:** [RAG](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** Chưa có.
- **Khái niệm sau trong cùng nhánh:** [Embedding](../embedding/README.md)

## Nguyên tắc nhanh

1. Chia theo **đơn vị có thể tự trả lời** (*answerable unit*), không chỉ theo số token.
2. Ưu tiên cấu trúc có sẵn: tiêu đề, điều/khoản, đoạn, bảng, hàm, class và khối code.
3. Giữ metadata đủ để lọc quyền, xác định phiên bản và trích dẫn đúng nguồn.
4. Chọn `chunkSize`, `overlap`, `topK` bằng tập câu hỏi kiểm thử thực tế; không có một cấu hình đúng cho mọi kho dữ liệu.
5. Xem chunking là pipeline **offline/ingestion** có version, kiểm thử và cơ chế re-index, không phải bước tiền xử lý làm một lần.

## A. 5 câu hỏi cốt lõi

### 1. Chunking là gì và vì sao RAG cần chunking?

**Trả lời:** Chunking chia một tài liệu thành các đoạn có ý nghĩa để hệ thống tạo embedding, lưu index và truy xuất đúng phần liên quan cho mỗi câu hỏi. Một tài liệu nguyên vẹn thường quá dài, chứa nhiều chủ đề, hoặc vượt ngân sách context của LLM. Embedding cả tài liệu làm vector trở nên “trung bình” cho nhiều ý khác nhau; truy xuất một câu/đoạn quá ngắn lại dễ thiếu điều kiện, ngoại lệ hoặc ngữ cảnh cần thiết.

Trong luồng **offline/ingestion**, tài liệu được parse, làm sạch, chia chunk, gắn metadata, tạo embedding rồi index. Trong luồng **online/serving**, câu hỏi được biến thành query, retrieval lấy các chunk phù hợp, hệ thống ghép context theo ngân sách token và LLM tạo câu trả lời kèm citation. Chunking nằm ở ranh giới giữa dữ liệu nguồn và retrieval: prompt tốt không thể bù hoàn toàn cho chunk thiếu hoặc sai ngữ cảnh.

**Áp dụng khi nào:** Hầu hết chatbot tra cứu tài liệu, chính sách, hướng dẫn kỹ thuật, FAQ, knowledge base nội bộ, tài liệu pháp lý hoặc hồ sơ sản phẩm. Có thể không cần chia nhỏ khi chỉ có vài văn bản rất ngắn và luôn phải đọc trọn vẹn.

**Bài toán giải quyết:** Giảm nhiễu khi retrieval, đưa đúng bằng chứng vào context, kiểm soát token/cost và giúp câu trả lời trích dẫn đúng section hoặc trang.

| Lựa chọn | Điểm mạnh | Giới hạn | Phù hợp khi |
| --- | --- | --- | --- |
| Embedding cả tài liệu | Đơn giản, ít bản ghi | Mất độ chính xác với tài liệu nhiều chủ đề; context lớn | Tài liệu rất ngắn, một chủ đề |
| Chunk theo câu | Recall chi tiết cao | Hay thiếu điều kiện, câu trước/sau và citation khó đọc | FAQ cực ngắn, dữ liệu có cấu trúc tốt |
| Chunk theo đoạn/section | Cân bằng ngữ nghĩa và ngữ cảnh | Cần parser và quy tắc theo cấu trúc | Lựa chọn mặc định cho RAG production |

### 2. Nên cắt chunk theo token, theo ngữ nghĩa hay theo cấu trúc tài liệu?

**Trả lời:** Ưu tiên **cấu trúc và ngữ nghĩa** trước, token sau. Một chunk tốt thường bắt đầu/kết thúc tại ranh giới tự nhiên như heading, điều khoản, đoạn văn, mục FAQ, bảng, hàm hoặc class. Token splitter là lớp bảo vệ cuối cùng khi section vẫn quá dài so với ngân sách retrieval và context.

Quy tắc thực hành: (1) parse và giữ hierarchy của tài liệu; (2) tách theo section/subsection; (3) gộp các đoạn liền nhau chỉ khi chúng cùng chủ đề; (4) chỉ cắt theo token ở section quá dài; (5) thêm heading/path của section vào mỗi chunk. Với dữ liệu không có cấu trúc tốt, có thể dùng semantic chunking dựa trên điểm thay đổi chủ đề, nhưng vẫn cần giới hạn kích thước và kiểm thử đầu ra.

**Áp dụng khi nào:** Cắt theo section cho policy, quy định và tài liệu có mục lục; theo paragraph cho bài viết; theo API/module/function cho tài liệu kỹ thuật; theo token cho OCR hoặc text thô không đáng tin cậy.

**Bài toán giải quyết:** Tránh tách quy tắc khỏi điều kiện, tách câu hỏi FAQ khỏi câu trả lời, hoặc cắt một hàm/code block thành nhiều phần vô nghĩa.

| Cách chia | Ưu điểm | Rủi ro | Cách giảm rủi ro |
| --- | --- | --- | --- |
| Fixed token/window | Dễ triển khai, kích thước ổn định | Cắt giữa ý, bảng hoặc code | Chỉ dùng làm fallback; thêm overlap nhỏ |
| Recursive splitter | Tôn trọng nhiều delimiter như heading/đoạn/câu | Không hiểu hoàn toàn chủ đề | Tùy chỉnh delimiter theo loại tài liệu |
| Semantic chunking | Có thể bám thay đổi chủ đề tốt | Chi phí/độ phức tạp cao; khó giải thích | Giới hạn min/max token, đánh giá bằng test set |
| Structure-aware chunking | Citation và ngữ cảnh tốt nhất | Cần parser riêng cho HTML, DOCX, PDF, Markdown | Chuẩn hóa parser và kiểm thử extraction |

### 3. Chọn `chunkSize` và `overlap` như thế nào?

**Trả lời:** Chọn kích thước theo **đơn vị thông tin cần thiết để trả lời**, rồi mới đo bằng token. `chunkSize` nhỏ tăng độ chính xác (*precision*) nhưng có thể làm mất ngữ cảnh và cần nhiều kết quả hơn; chunk lớn giữ bối cảnh tốt hơn nhưng dễ chứa nhiễu, làm giảm khả năng xếp hạng và tăng token khi gửi LLM. `overlap` chỉ nên bảo vệ các ý kéo dài qua ranh giới; overlap quá lớn tạo bản sao gần giống nhau, chiếm top-k và làm tốn chi phí.

Có thể dùng baseline để bắt đầu, sau đó tune bằng dữ liệu thật: FAQ thường một cặp hỏi–đáp; policy thường một section hoặc subsection với điều kiện/ngoại lệ liên quan; tài liệu kỹ thuật thường một hướng dẫn, API endpoint, class hoặc function hoàn chỉnh. Với prose dài, dùng window cỡ trung bình và overlap nhỏ theo ranh giới đoạn/câu. Những con số token chỉ là điểm khởi đầu vì tokenizer, ngôn ngữ, mô hình embedding, retriever và context budget đều khác nhau.

**Áp dụng khi nào:** Thiết lập lần đầu cho index mới, thay đổi model embedding, thêm loại tài liệu mới hoặc khi retrieval có dấu hiệu thiếu bối cảnh.

**Bài toán giải quyết:** Cân bằng recall, precision, latency, chi phí embedding/index và chi phí token ở bước generation.

| Cấu hình | Lợi ích | Rủi ro thường gặp | Cách kiểm tra |
| --- | --- | --- | --- |
| Chunk nhỏ, overlap thấp | Khớp câu hỏi hẹp tốt | Thiếu điều kiện/ngoại lệ | Đo context completeness và citation correctness |
| Chunk vừa, overlap nhỏ | Cân bằng tốt, thường là baseline | Cần tune theo domain | Đo Recall@k, answer quality, token cost |
| Chunk lớn | Đủ ngữ cảnh cho quy trình dài | Nhiễu, retrieval kém phân biệt | Kiểm tra rank của evidence đúng và p95 latency |
| Overlap lớn | Ít mất ý qua biên | Duplicate context, giảm diversity top-k | Đo tỷ lệ near-duplicate trong kết quả |

**Cách tune tối thiểu:** Chuẩn bị tập 30–100 câu hỏi thật có “evidence” kỳ vọng; thử một vài kích thước hợp lý và overlap nhỏ; so sánh Recall@k, MRR/NDCG khi có nhãn xếp hạng, chất lượng citation, tỷ lệ câu trả lời có đủ điều kiện và tổng token context. Chọn cấu hình nhỏ nhất vẫn đạt chất lượng mục tiêu.

### 4. Một chunk cần metadata và ngữ cảnh nào?

**Trả lời:** Mỗi chunk cần mang theo “địa chỉ” của nó trong tài liệu và dữ liệu vận hành để hệ thống lọc, trích dẫn, audit và cập nhật an toàn. Tối thiểu nên có `documentId`, `chunkId`, `sourceUri`, `title`, `sectionPath`, `chunkIndex`, `page` hoặc vị trí gốc nếu có, `version`, `updatedAt`, `language`, `textHash` và thông tin quyền truy cập (*Access Control List, ACL*) hoặc scope tương đương.

Ngoài metadata lọc, nên thêm **contextual header** vào phần text được embedding, ví dụ: `Sổ tay nhân sự > Nghỉ phép > Điều kiện hưởng phép năm`. Header giúp embedding hiểu ý nghĩa của đoạn ngắn mà không cần nhân bản toàn bộ tài liệu. Tuy nhiên, ACL không nên chỉ là text trong chunk: quyền phải được kiểm tra bằng filter bắt buộc ở tầng retrieval và bằng source-of-truth quản lý quyền.

**Áp dụng khi nào:** Luôn cần cho production; đặc biệt quan trọng với tài liệu có phiên bản, dữ liệu đa ngôn ngữ, multi-tenant, tài liệu nội bộ hoặc nội dung bị giới hạn quyền.

**Bài toán giải quyết:** Citation sai nguồn, truy xuất tài liệu hết hiệu lực, rò rỉ dữ liệu chéo tenant/role, không thể biết chunk được tạo bởi pipeline nào.

| Cách mang ngữ cảnh | Lợi ích | Giới hạn |
| --- | --- | --- |
| Chỉ lưu text | Đơn giản | Không lọc được quyền, version và nguồn tốt |
| Metadata filter | Tốt cho ACL, version, locale, document type | Nếu không thêm section title vào text thì vector có thể thiếu ngữ cảnh |
| Contextual header + metadata | Cân bằng retrieval, citation và kiểm soát | Cần kiểm soát token dư và chuẩn hóa schema |

Ví dụ metadata rút gọn:

```json
{
  "chunkId": "hr-handbook:v2026-07:leave:03",
  "documentId": "hr-handbook",
  "sourceUri": "kb://hr/handbook/leave",
  "sectionPath": ["Nghỉ phép", "Phép năm", "Điều kiện"],
  "chunkIndex": 3,
  "version": "2026-07",
  "effectiveFrom": "2026-07-01",
  "textHash": "...",
  "accessScope": ["employee"]
}
```

### 5. Làm sao đánh giá và vận hành chunking trong production?

**Trả lời:** Đánh giá chunking qua **khả năng tìm đúng evidence**, không chỉ qua câu trả lời nghe hay. Xây một tập test gồm câu hỏi đại diện, câu trả lời/section nguồn kỳ vọng, quyền truy cập và phiên bản tài liệu. Đo retrieval bằng Recall@k, MRR hoặc NDCG khi có nhãn; đo end-to-end bằng citation correctness, context completeness, groundedness/faithfulness, refusal đúng khi không có nguồn, latency và token cost.

Pipeline cần idempotent và có version. Mỗi lần parser, splitter, embedding model hoặc quy tắc metadata thay đổi đáng kể, cần ghi `pipelineVersion`/`chunkingConfigVersion`, tạo index mới hoặc re-index có kiểm soát. Khi tài liệu đổi, dùng `documentId` + `version`/hash để upsert chunk mới, xóa hoặc làm hết hiệu lực chunk cũ, rồi kiểm tra không còn citation tới nội dung đã hết hiệu lực.

**Áp dụng khi nào:** Khi RAG có người dùng thật, tài liệu cập nhật định kỳ, có yêu cầu audit hoặc thay đổi provider/model.

**Bài toán giải quyết:** Chất lượng giảm âm thầm sau cập nhật dữ liệu, kết quả không tái lập được, index chứa bản cũ, regression do thay embedding hoặc parser.

**Checklist vận hành:**

1. Chốt source of truth và quy tắc đồng bộ cho từng nguồn tài liệu.
2. Lưu raw document, nội dung đã parse và manifest chunk để có thể trace/debug.
3. Đặt ID ổn định; không dùng chỉ số mảng đơn thuần làm định danh duy nhất.
4. Test parser/chunker bằng snapshot cho heading, bảng, danh sách, code và văn bản song ngữ.
5. Chạy retrieval regression trước khi rollout index/chunking config mới.
6. Quan sát tỷ lệ không tìm thấy evidence, duplicate top-k, citation sai, p95 latency và chi phí token.

## B. 8 câu hỏi phổ biến mở rộng

### 6. Chunking có khác nhau theo lĩnh vực không?

**Trả lời:** Có. Đơn vị ngữ nghĩa, rủi ro khi mất bối cảnh và cách citation khác nhau theo domain. Không nên áp cùng một splitter cho mọi loại dữ liệu.

| Lĩnh vực | Đơn vị chunk ưu tiên | Cần giữ cùng nhau | Rủi ro chính |
| --- | --- | --- | --- |
| Chính sách/doanh nghiệp | Article → section → clause | Quy tắc, điều kiện, ngoại lệ, ngày hiệu lực | Trả lời thiếu ngoại lệ hoặc dùng bản cũ |
| Pháp lý | Điều, khoản, điểm; định nghĩa tham chiếu | Phạm vi áp dụng, định nghĩa, ngoại lệ, hiệu lực | Diễn giải sai hoặc bỏ giới hạn pháp lý |
| Y tế | Guideline section, recommendation, contraindication | Khuyến nghị, chống chỉ định, mức bằng chứng | Đưa lời khuyên thiếu an toàn |
| Tài chính | Sản phẩm/quy trình/điều khoản | Điều kiện, phí, thời hạn, disclaimer | Sai điều kiện nghiệp vụ hoặc compliance |
| Kỹ thuật | Module, endpoint, class, function, how-to task | Input/output, prerequisite, lỗi, code block | Code/hướng dẫn thiếu bước hoặc không chạy |
| CSKH/FAQ | Một intent và câu trả lời có hiệu lực | Điều kiện, SLA, kênh xử lý | Trả lời lan man hoặc không đúng chính sách |

**Áp dụng:** Tạo profile chunking theo `documentType` thay vì cố gom mọi tài liệu vào một cấu hình. Với domain rủi ro cao, luôn giữ citation và cơ chế từ chối khi evidence không đủ.

### 7. Xử lý PDF, bảng, hình và code block như thế nào?

**Trả lời:** Chất lượng extraction quyết định chất lượng chunking. Không nên xem PDF là text phẳng. Cần xác định nguồn là text-native hay scan/OCR, giữ heading/page/caption, xử lý header/footer lặp và kiểm tra thứ tự đọc nhiều cột.

- **Bảng:** Không cắt giữa header và row. Gắn header vào từng phần bảng khi bảng dài; nếu cần chia, mỗi phần phải lặp lại tên cột và đơn vị đo. Có thể lưu dạng Markdown/HTML hoặc text chuẩn hóa tùy model.
- **Code:** Giữ nguyên function/class/code block cùng signature, input/output, comments và lỗi liên quan. Không cắt giữa block code; với file lớn, ưu tiên module → class → function, kèm file path và symbol name.
- **Hình/diagram:** Lưu caption, alt text hoặc mô tả được kiểm duyệt; chỉ OCR chữ trong hình khi cần và đánh dấu độ tin cậy. Không để LLM suy đoán nội dung biểu đồ khi source không chứa mô tả đáng tin cậy.
- **PDF scan/OCR:** Lưu page number, confidence OCR và link về bản gốc; test các trang có bảng/biểu mẫu vì đây là nơi parser hay lỗi nhất.

**Bài toán giải quyết:** Tránh retrieval ra bảng thiếu tiêu đề, đoạn code không biên dịch được, hoặc text OCR bị đảo thứ tự làm citation sai trang.

### 8. Parent-child hoặc hierarchical chunking là gì, khi nào nên dùng?

**Trả lời:** **Hierarchical chunking** tạo nhiều cấp nội dung, chẳng hạn document → section → child chunk. **Parent-child retrieval** dùng child chunk nhỏ để tìm chính xác, nhưng khi đã match thì gửi parent section lớn hơn cho LLM để đủ ngữ cảnh. Đây là cách tách mục tiêu “retrieval chính xác” khỏi mục tiêu “generation đủ bối cảnh”.

**Áp dụng khi nào:** Tài liệu có cấu trúc rõ, yêu cầu trả lời phải giữ điều kiện/ngoại lệ, hoặc chunk nhỏ liên tục thiếu context nhưng chunk lớn làm retrieval bị nhiễu. Phù hợp cho policy, legal, technical runbook và tài liệu có section dài.

| Thiết kế | Ưu điểm | Chi phí/giới hạn |
| --- | --- | --- |
| Flat chunking | Đơn giản, dễ debug | Khó cân bằng precision và context |
| Parent-child | Match chính xác bằng child, trả lời bằng parent | Cần mapping ID, token budget và dedup |
| Multi-level summary + detail | Hỗ trợ cả câu hỏi tổng quan và chi tiết | Pipeline phức tạp, phải đánh giá tránh summary sai |

**Lưu ý:** Parent không được vượt context budget. Có thể chỉ lấy vùng parent liên quan, hoặc lấy parent + các child đa dạng thay vì nhét toàn bộ tài liệu.

### 9. Overlap có luôn cần không? Làm sao tránh duplicate context?

**Trả lời:** Không. Nếu chunking bám đúng section/đoạn và mỗi đơn vị đã tự đủ nghĩa, overlap có thể bằng 0 hoặc rất nhỏ. Overlap hữu ích nhất khi phải cắt một đoạn dài theo token hoặc khi một ý quan trọng kéo qua ranh giới. Nếu các chunk gần giống nhau cùng chiếm top-k, LLM nhận nhiều bản sao của một evidence và bỏ lỡ thông tin bổ sung.

**Cách giảm duplicate:** ưu tiên boundary ngữ nghĩa; chỉ overlap phần cuối/đầu cần thiết; deduplicate bằng `textHash`/similarity; dùng Maximal Marginal Relevance (MMR) hoặc diversity reranking; gộp các chunk kề nhau sau retrieval khi chúng thuộc cùng section và thực sự bổ sung nhau.

**So sánh:** Overlap giải quyết mất mát ở biên nhưng không giải quyết parser kém, metadata thiếu hoặc query không rõ. Đừng tăng overlap để chữa mọi lỗi retrieval.

### 10. Semantic chunking, LLM chunking và rule-based chunking nên chọn cách nào?

**Trả lời:** Bắt đầu bằng **rule-based, structure-aware chunking** vì dễ kiểm soát, rẻ và dễ audit. Dùng semantic chunking khi tài liệu ít cấu trúc nhưng có prose dài, nhiều chủ đề chuyển tiếp. Dùng LLM để hỗ trợ nhận diện cấu trúc, phân loại document type hoặc tạo metadata/caption có kiểm soát, không nên mặc định giao toàn bộ ranh giới chunk cho LLM nếu chi phí, tính tái lập và audit là quan trọng.

| Cách làm | Phù hợp | Ưu điểm | Hạn chế |
| --- | --- | --- | --- |
| Rule-based | Markdown, HTML, DOCX, policy có cấu trúc | Ổn định, nhanh, dễ test | Cần rule riêng cho từng nguồn |
| Semantic | Văn bản thô/prose dài | Bám thay đổi chủ đề tốt hơn | Khó predict, cần evaluate kỹ |
| LLM-assisted | Tài liệu phức tạp, extraction/normalization khó | Linh hoạt, hiểu cấu trúc hơn | Cost, latency, nondeterminism, rủi ro hallucination |

**Nguyên tắc:** Lưu kết quả parse và boundary để replay; luôn validate min/max length, hierarchy, metadata và bản gốc sau bước dùng LLM.

### 11. Có nên tạo chunk khác nhau cho từng loại câu hỏi hoặc từng ngôn ngữ không?

**Trả lời:** Có thể, nhưng chỉ khi test chứng minh lợi ích. Với câu hỏi “tổng quan”, summary chunk hoặc parent section có thể hữu ích; với câu hỏi tác vụ, chunk procedure nhỏ có bước tuần tự tốt hơn. Với kho song ngữ, giữ `language`, tên riêng, thuật ngữ gốc và bản dịch đã kiểm duyệt trong metadata/text. Không tự động dịch mọi chunk nếu bản dịch có thể làm mất nghĩa pháp lý hoặc kỹ thuật.

**Áp dụng:** Kho kiến thức vừa FAQ vừa tài liệu dài; sản phẩm đa ngôn ngữ; search hỗn hợp tiếng Việt–Anh. Có thể route query theo intent/locale rồi chọn retriever/chunk type phù hợp.

**Bài toán giải quyết:** Một index duy nhất trả lời tốt cho intent ngắn nhưng kém với câu hỏi tổng quan, hoặc thất bại khi query dùng thuật ngữ khác ngôn ngữ nguồn.

### 12. Làm sao ghép nhiều chunk vào context của LLM mà không vượt token budget?

**Trả lời:** Chunking chỉ là nửa đầu; cần **context assembly** ở serving. Xác định ngân sách token cho system prompt, lịch sử hội thoại, câu hỏi, context, output và safety margin. Sau đó retrieval lấy nhiều ứng viên, filter theo quyền/version, rerank, deduplicate, rồi chọn các chunk đa dạng nhưng có liên kết nguồn rõ ràng. Có thể ưu tiên những chunk cùng document/section khi câu trả lời cần điều kiện liên tiếp.

**Áp dụng:** Chat có lịch sử dài, mô hình context giới hạn, tài liệu chứa nhiều section tương tự hoặc nhu cầu yêu cầu citation nhiều nguồn.

| Cách ghép context | Khi phù hợp | Rủi ro |
| --- | --- | --- |
| Top-k theo score | Đơn giản, dữ liệu nhỏ | Nhiều chunk trùng hoặc thiếu điều kiện liên quan |
| Rerank + dedup | Production phổ biến | Thêm latency/cost |
| Neighbor expansion | Nội dung cần câu trước/sau | Có thể nở token nhanh |
| Parent retrieval | Section có ngữ cảnh mạnh | Parent quá lớn nếu không budget tốt |

**Quy tắc an toàn:** Khi evidence không đủ sau filter/budget, yêu cầu LLM nói không tìm thấy thông tin thay vì ghép chunk xa chủ đề để đoán.

### 13. Dấu hiệu chunking kém là gì và debug theo thứ tự nào?

**Trả lời:** Dấu hiệu phổ biến là: chunk chứa đáp án đúng không xuất hiện trong top-k; xuất hiện nhưng thiếu điều kiện/ngoại lệ; nhiều chunk gần giống nhau; citation trỏ sai section; kết quả đúng với bản tài liệu cũ nhưng không đúng bản mới; hoặc câu trả lời chỉ đúng khi tăng top-k/token rất lớn.

**Quy trình debug:**

1. Kiểm tra source of truth: tài liệu đúng, version còn hiệu lực, quyền đúng chưa.
2. Kiểm tra extraction: thứ tự đọc PDF, heading, bảng, code, ký tự OCR và nội dung bị bỏ sót.
3. Xem manifest chunk: boundary, header, metadata, độ dài, overlap, hash và chunk hàng xóm.
4. Chạy retrieval không có LLM generation để xác định lỗi nằm ở data/chunking/embedding hay prompt.
5. Kiểm tra filter ACL/version/locale và query rewrite.
6. So sánh rank của evidence đúng trước/sau rerank; kiểm tra duplicate top-k.
7. Chỉ sau đó mới tune `chunkSize`, overlap, model embedding, index hoặc prompt.

**So sánh:** Prompt engineering giúp LLM sử dụng context tốt hơn; chunking/retrieval engineering quyết định context có đúng hay không. Không nên sửa prompt trước khi chứng minh evidence đúng đã được retrieve.

### 14. Khi nào dùng long context thay vì chunking và retrieval?

**Trả lời:** Long context hữu ích khi cần đọc một số ít tài liệu theo mạch liên tục, như review hợp đồng cụ thể, phân tích một incident report hoặc tóm tắt một hồ sơ được người dùng tải lên. Tuy nhiên, kho tri thức lớn vẫn nên chunk và retrieve để tìm đúng tài liệu, áp dụng filter quyền/version, giảm chi phí và cung cấp citation rõ ràng.

| Phương án | Tốt khi | Không phù hợp khi |
| --- | --- | --- |
| Long context trực tiếp | Ít tài liệu, cần hiểu toàn cảnh, nội dung phiên làm việc ngắn hạn | Kho lớn, dữ liệu thay đổi, cần search/citation/ACL chặt |
| Chunking + retrieval | Nhiều tài liệu, nhiều phiên bản, câu hỏi đa dạng | Một tài liệu nhỏ luôn cần đọc toàn bộ |
| Kết hợp | Retrieve tài liệu/section trước rồi dùng long context cho phần liên quan | Cần kiểm soát kỹ token và nguồn dẫn |

**Kết luận:** Long context không thay thế hoàn toàn chunking; nó thay đổi giới hạn kích thước context ở bước generation. Retrieval vẫn cần đơn vị index, metadata, quyền truy cập và chiến lược chọn evidence.

## Quy trình áp dụng đề xuất

1. **Phân loại nguồn dữ liệu:** policy, FAQ, technical docs, PDF scan, bảng, code; xác định owner và source of truth.
2. **Thiết kế schema metadata:** document ID, version, effective date, ACL, section path, page/source URL và chunking config version.
3. **Tạo test set:** câu hỏi thật, expected evidence, trường hợp không có dữ liệu và trường hợp bị hạn chế quyền.
4. **Parse có cấu trúc:** chuẩn hóa heading, list, table, code, footnote, header/footer và chất lượng OCR.
5. **Dùng structure-aware splitter làm baseline:** áp dụng token limit chỉ khi cần; thêm contextual header.
6. **Index và retrieval test:** đo Recall@k, citation correctness, duplicate rate, latency và token cost.
7. **Tune có kiểm soát:** thay đổi một biến mỗi lần (`chunkSize`, overlap, splitter, metadata, retriever hoặc reranker), lưu lại config và kết quả.
8. **Vận hành theo version:** re-index có kiểm soát khi tài liệu hoặc pipeline đổi; monitor stale content, lỗi ACL và regression.

## Liên kết liên quan

- [RAG](../README.md)
- [Embedding](../embedding/README.md)
- [Retrieval](../retrieval/README.md)
- [Chỉ mục pgvector (pgvector Index)](../pgvector-index/README.md)
