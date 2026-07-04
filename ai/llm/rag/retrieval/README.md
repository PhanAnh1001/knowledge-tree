# Retrieval trong RAG

> [Knowledge Tree](../../../../README.md) / [AI](../../../README.md) / [LLM](../../README.md) / [RAG](../README.md) / **Retrieval**

## Điều hướng
- **Node cha:** [RAG](../README.md)
- **Khái niệm trước:** [Embedding](../embedding/README.md)
- **Khái niệm sau:** [Reranking](../reranking/README.md)

## A. 5 câu hỏi cốt lõi

### 1. Retrieval là gì?
Nhận query, lọc phạm vi và trả về chunk làm bằng chứng trước khi LLM trả lời.

### 2. Vector search hoạt động thế nào?
Embed query rồi tìm vector gần nhất theo similarity metric.

### 3. Hybrid search là gì?
Kết hợp vector search với keyword/BM25 để vừa hiểu ngữ nghĩa vừa giữ exact match.

### 4. Metadata filter nên đặt ở đâu?
Áp trước hoặc trong retrieval theo phiên bản, phạm vi dữ liệu và thời gian hiệu lực.

### 5. Top-k chọn thế nào?
Lấy đủ rộng cho recall, sau đó threshold/MMR/dedup để giữ context đa dạng.

## B. 5 câu hỏi phổ biến mở rộng

### 6. FAISS khác Qdrant thế nào?
FAISS là thư viện vector; Qdrant là vector database có persistence, filter và service API.

### 7. Khi nào query rewrite hữu ích?
Khi câu hỏi mơ hồ; chỉ bật khi evaluation chứng minh tăng recall.

### 8. Không có evidence tốt thì sao?
Dùng threshold và trả lời không đủ thông tin.

### 9. Xử lý tài liệu mới/cũ thế nào?
Filter hoặc boost documentVersion, effectiveDate, updatedAt; loại bản thu hồi.

### 10. Khi nào không dùng retrieval?
Dữ liệu realtime, phép tính chính xác và tác vụ nghiệp vụ cần hệ thống nguồn.

## Liên kết liên quan
- [RAG](../README.md)
- [Embedding](../embedding/README.md)
- [Reranking](../reranking/README.md)
