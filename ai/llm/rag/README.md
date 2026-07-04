# RAG (Retrieval-Augmented Generation)

> [Knowledge Tree](../../../README.md) / [Trí tuệ nhân tạo (AI)](../../README.md) / [Mô hình ngôn ngữ lớn (LLM)](../README.md) / **RAG**

**RAG** (*Retrieval-Augmented Generation* — sinh câu trả lời có tăng cường truy xuất) cho phép LLM tìm và đọc các đoạn tài liệu liên quan trước khi trả lời. Công thức cần nhớ: **Question → Retrieve → Context → Generate → Cite**.

## Điều hướng

- **Node cha:** [Mô hình ngôn ngữ lớn (LLM)](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** Chưa có.
- **Khái niệm sau trong cùng nhánh:** Chưa có.

---

## A. 5 câu hỏi cốt lõi

### 1. RAG là gì và khác gì với việc hỏi LLM bình thường?

**Trả lời:** RAG kết hợp **truy xuất tài liệu** (*retrieval*) với **sinh câu trả lời** (*generation*). Thay vì dựa hoàn toàn vào kiến thức đã huấn luyện, LLM nhận thêm các đoạn tài liệu phù hợp làm *context* rồi trả lời dựa trên chúng.

- **Áp dụng:** Chatbot hỏi đáp chính sách, tra cứu tài liệu kỹ thuật, hỗ trợ khách hàng, hỏi đáp PDF hoặc wiki nội bộ.
- **Giải quyết:** Bổ sung tri thức riêng, cập nhật nhanh dữ liệu thay đổi, giảm trả lời bịa (*hallucination*).
- **So sánh:** Prompt thuần không có nguồn dữ liệu ngoài; *fine-tuning* thay đổi trọng số model và phù hợp hơn cho phong cách hoặc hành vi lặp lại.

### 2. Pipeline RAG cơ bản hoạt động như thế nào?

**Trả lời:** Tài liệu được nạp (*ingestion*), chia đoạn (*chunking*), chuyển thành vector (*embedding*) và lưu vào kho tìm kiếm. Khi có câu hỏi, hệ thống tìm các đoạn liên quan (*retrieval*), đưa chúng vào prompt (*context augmentation*) và yêu cầu LLM sinh câu trả lời.

- **Áp dụng:** Khi kho dữ liệu lớn hơn nhiều so với context window hoặc thường xuyên có tài liệu mới.
- **Giải quyết:** Chỉ đưa phần liên quan vào prompt, giảm token, giảm nhiễu và tăng khả năng truy vết nguồn.
- **So sánh:** *Long context* có thể đọc cả tài liệu dài, nhưng dễ tốn token và giảm tập trung khi kho tài liệu lớn; RAG chọn lọc context trước.

### 3. Chunking, embedding và vector database có vai trò gì?

**Trả lời:** *Chunking* chia tài liệu dài thành đoạn nhỏ; *embedding* biến mỗi đoạn và câu hỏi thành vector biểu diễn ngữ nghĩa; *vector database* lưu và tìm các vector gần nhau. Ba thành phần này quyết định hệ thống có lấy được đúng nội dung hay không.

- **Áp dụng:** Tài liệu dài, PDF, wiki, hướng dẫn kỹ thuật, điều khoản, chính sách hoặc knowledge base.
- **Giải quyết:** Tìm được nội dung cùng nghĩa dù người dùng dùng từ khác tài liệu.
- **So sánh:** *Keyword search* chính xác với mã đơn hàng, tên riêng hoặc cụm từ bắt buộc; *vector search* tốt hơn với cách hỏi tự nhiên và từ đồng nghĩa.

### 4. Retrieval tốt cần những kỹ thuật nào?

**Trả lời:** Một pipeline production thường dùng **hybrid search** (kết hợp keyword và vector), **metadata filtering** (lọc theo loại tài liệu, ngày, quyền) và **reranking** (xếp hạng lại các kết quả đầu). Thông thường retrieval lấy nhiều ứng viên, reranker chọn ít đoạn chất lượng cao để đưa vào LLM.

- **Áp dụng:** Kho tài liệu lớn, có nhiều thuật ngữ chính xác, nhiều phiên bản tài liệu hoặc yêu cầu phân quyền.
- **Giải quyết:** Tăng độ chính xác, giảm context nhiễu và hạn chế lấy nhầm tài liệu cũ.
- **So sánh:** Vector search đơn thuần dễ bỏ sót từ khóa định danh; keyword search đơn thuần dễ bỏ sót câu hỏi diễn đạt khác. Hybrid search thường cân bằng tốt hơn.

### 5. Làm sao RAG giảm hallucination và tạo câu trả lời đáng tin cậy?

**Trả lời:** Prompt cần yêu cầu LLM chỉ trả lời dựa trên context được cấp, nói rõ “không tìm thấy thông tin” khi thiếu bằng chứng và kèm **citation** (nguồn, trang, đường dẫn). Chất lượng retrieval vẫn là điều kiện tiên quyết: context sai thì LLM có thể trả lời sai một cách thuyết phục.

- **Áp dụng:** Pháp lý, y tế, tài chính, chính sách công ty và tài liệu có rủi ro nghiệp vụ.
- **Giải quyết:** Tăng khả năng kiểm chứng, audit và debug câu trả lời.
- **So sánh:** Không có citation khiến người dùng phải tin vào model; citation biến câu trả lời thành thông tin có thể đối chiếu với tài liệu gốc.

---

## B. 5 câu hỏi phổ biến mở rộng

### 6. Nên dùng RAG hay fine-tuning?

**Trả lời:** Dùng RAG khi trọng tâm là tri thức thay đổi thường xuyên hoặc dữ liệu riêng; dùng *fine-tuning* khi cần model tuân theo phong cách, định dạng hoặc hành vi ổn định. Nhiều hệ thống dùng cả hai: RAG cho kiến thức, fine-tuning cho cách trả lời.

- **Áp dụng:** RAG cho FAQ, chính sách, product docs; fine-tuning cho phân loại, format JSON ổn định hoặc giọng điệu thương hiệu.
- **Giải quyết:** Tránh phải huấn luyện lại model chỉ vì một tài liệu thay đổi.
- **So sánh:** RAG cập nhật bằng re-index tài liệu; fine-tuning cập nhật bằng chu trình huấn luyện và đánh giá mới.

### 7. Reranking là gì và khi nào cần dùng?

**Trả lời:** *Reranking* dùng một mô hình đánh giá lại mức liên quan giữa câu hỏi và các chunk đã tìm được để sắp xếp kết quả tốt hơn. Pipeline phổ biến là lấy 20–50 ứng viên rồi rerank để chọn 3–10 chunk tốt nhất.

- **Áp dụng:** Câu hỏi mơ hồ, kho dữ liệu lớn hoặc retrieval ban đầu có nhiều kết quả na ná nhau.
- **Giải quyết:** Loại context yếu trước khi gửi đến LLM, thường tăng chất lượng câu trả lời.
- **So sánh:** Không rerank nhanh hơn và rẻ hơn; rerank tăng độ chính xác nhưng thêm độ trễ và chi phí.

### 8. Metadata và phân quyền (ACL) trong RAG nên làm thế nào?

**Trả lời:** Mỗi chunk cần metadata như `source`, `page`, `department`, `documentVersion`, `updatedAt`, `product` và `accessRoles`. Hệ thống phải lọc theo quyền của người hỏi **trước** khi retrieval hoặc trước khi trả context cho LLM.

- **Áp dụng:** Knowledge base nội bộ có phòng ban, khách hàng, chi nhánh hoặc tài liệu mật.
- **Giải quyết:** Ngăn rò rỉ dữ liệu, ưu tiên tài liệu mới nhất và truy vết nguồn rõ ràng.
- **So sánh:** Chỉ dùng vector similarity không đảm bảo đúng phạm vi; vector kết hợp metadata/ACL tạo kết quả vừa liên quan vừa hợp lệ.

### 9. Có những kiến trúc RAG nào?

**Trả lời:** **Naive RAG** gồm chunk → embed → retrieve → generate; **Advanced RAG** thêm hybrid search, reranking, metadata và citation; **Agentic RAG** cho agent lập kế hoạch tìm kiếm hoặc gọi tool; **Graph RAG** bổ sung knowledge graph để xử lý quan hệ phức tạp.

- **Áp dụng:** Naive RAG cho prototype; Advanced RAG cho production; Agentic/Graph RAG cho truy vấn đa bước hoặc dữ liệu nhiều quan hệ.
- **Giải quyết:** Chọn mức độ phức tạp phù hợp thay vì dùng kiến trúc nặng ngay từ đầu.
- **So sánh:** Agentic RAG linh hoạt hơn nhưng khó kiểm soát và tốn chi phí; Graph RAG mạnh về quan hệ thực thể nhưng yêu cầu mô hình dữ liệu tốt.

### 10. Khi nào không nên dùng RAG và đánh giá chất lượng bằng gì?

**Trả lời:** Không nên dùng RAG cho phép tính chính xác, trạng thái đơn hàng thời gian thực hoặc thao tác nghiệp vụ—các trường hợp đó nên gọi API, database hoặc tool. Khi dùng RAG, cần đánh giá cả retrieval và generation qua các chỉ số như **Recall@k**, **Precision@k**, *groundedness*, độ đúng citation, độ trễ và chi phí mỗi câu hỏi.

- **Áp dụng:** Thiết kế kiến trúc chatbot và kiểm thử trước khi production.
- **Giải quyết:** Tránh dùng LLM thay cho hệ thống giao dịch; phát hiện lỗi retrieval thay vì chỉ nhìn câu trả lời cuối.
- **So sánh:** Tool/function calling trả kết quả xác định từ hệ thống nguồn; RAG phù hợp nhất khi cần đọc, tìm và tổng hợp tri thức phi cấu trúc.

---

## Checklist áp dụng nhanh

- Dữ liệu có thay đổi hoặc là dữ liệu riêng? → Ưu tiên RAG.
- Có mã, tên riêng hoặc thuật ngữ bắt buộc? → Thêm keyword/BM25 vào hybrid search.
- Có dữ liệu nhạy cảm? → Thiết kế metadata và ACL từ đầu.
- Có yêu cầu chính xác cao? → Thêm reranking, citation, cơ chế “không đủ thông tin” và bộ câu hỏi đánh giá.
- Cần lấy trạng thái hoặc thực thi tác vụ? → Kết hợp RAG với API/tool calling, không thay thế bằng RAG.

## Liên kết liên quan

- [Mô hình ngôn ngữ lớn (LLM)](../README.md)
- [Trí tuệ nhân tạo (AI)](../../README.md)
- [Knowledge Tree](../../../README.md)
