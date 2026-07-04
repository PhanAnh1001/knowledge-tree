# Chunking trong RAG

> [Knowledge Tree](../../../../README.md) / [AI](../../../README.md) / [LLM](../../README.md) / [RAG](../README.md) / **Chunking**

**Chunking** chia tài liệu thành các đoạn có ngữ cảnh phù hợp để embedding, retrieval và citation chính xác.

## Điều hướng

- **Node cha:** [RAG](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** Chưa có.
- **Khái niệm sau trong cùng nhánh:** [Embedding](../embedding/README.md)

## A. 5 câu hỏi cốt lõi

### 1. Chunking là gì?
**Trả lời:** Chia tài liệu dài thành các chunk để index và truy xuất; áp dụng khi kho tri thức lớn hơn context của LLM.

### 2. Chia theo token hay ngữ nghĩa?
**Trả lời:** Ưu tiên heading, section, đoạn văn, danh sách và code block; chỉ dùng token splitter khi cấu trúc tài liệu kém.

### 3. Chunk size và overlap chọn thế nào?
**Trả lời:** Bắt đầu mức trung bình và overlap nhỏ, sau đó tune theo Recall@k, chất lượng answer, token context và chi phí.

### 4. Metadata chunk gồm gì?
**Trả lời:** Giữ `source`, `section`, `page`, `version`, `updatedAt`, `chunkIndex` và ACL để citation, filtering, audit chính xác.

### 5. Vì sao policy nên chunk theo section?
**Trả lời:** Quy tắc, điều kiện và ngoại lệ phụ thuộc section/subsection; cắt theo section tránh tách điều kiện khỏi quy tắc.

## B. 5 câu hỏi phổ biến mở rộng

### 6. Chunking theo lĩnh vực có khác nhau không?
**Trả lời:** Có: policy theo article–section–clause, kỹ thuật theo module/API/code, y tế theo guideline và chống chỉ định.

### 7. Xử lý bảng, code và PDF thế nào?
**Trả lời:** Không cắt giữa bảng/code; giữ header bảng, file/hàm code và caption hình để không mất cấu trúc.

### 8. Dấu hiệu chunking kém là gì?
**Trả lời:** Chunk đúng không nằm top-k, citation lệch section hoặc context thiếu ngoại lệ; cần kiểm tra preprocessing trước prompt.

### 9. Cập nhật tài liệu thế nào?
**Trả lời:** Dùng document ID, hash/version để re-chunk, upsert và xóa chunk cũ an toàn, tránh dữ liệu hết hiệu lực.

### 10. Khi nào dùng long context?
**Trả lời:** Phù hợp ít tài liệu cần đọc liền mạch; kho lớn vẫn cần chunking/retrieval để chọn lọc context.

## Liên kết liên quan

- [RAG](../README.md)
- [Embedding](../embedding/README.md)
- [Retrieval](../retrieval/README.md)
