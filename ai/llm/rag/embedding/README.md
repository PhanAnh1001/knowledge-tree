# Embedding trong RAG

> [Knowledge Tree](../../../../README.md) / [AI](../../../README.md) / [LLM](../../README.md) / [RAG](../README.md) / **Embedding**

## Điều hướng
- **Node cha:** [RAG](../README.md)
- **Khái niệm trước:** [Chunking](../chunking/README.md)
- **Khái niệm sau:** [Retrieval](../retrieval/README.md)

## A. 5 câu hỏi cốt lõi

### 1. Embedding là gì?
Vector biểu diễn ngữ nghĩa của query và chunk, dùng để tìm nội dung gần nghĩa.

### 2. Vì sao query và document cần tương thích?
Phải dùng cùng model hoặc không gian vector tương thích để score có ý nghĩa.

### 3. Chọn model tiếng Việt thế nào?
Kiểm tra trực tiếp query, thuật ngữ nội bộ và dữ liệu Việt/Anh thực tế.

### 4. Similarity metric là gì?
Cosine, dot product hoặc Euclidean phải khớp model và index.

### 5. Embed nội dung nào?
Thường gồm heading/section cùng body chunk; metadata dùng cho filter.

## B. 5 câu hỏi phổ biến mở rộng

### 6. Có cần normalize text?
Có, nhưng không xóa mã, tên riêng hay số phiên bản.

### 7. Migrate model thế nào?
Re-embed sang index mới, evaluate song song và rollback được.

### 8. Embedding có thay hybrid search không?
Không; mã và tên riêng vẫn cần keyword/BM25.

### 9. Khi nào cache embedding?
Cache theo text hash và model version cho dữ liệu lặp lại.

### 10. Đánh giá bằng gì?
Dùng Recall@k, MRR/nDCG và quality end-to-end.

## Liên kết liên quan
- [RAG](../README.md)
- [Chunking](../chunking/README.md)
- [Retrieval](../retrieval/README.md)
