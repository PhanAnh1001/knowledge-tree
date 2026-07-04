# RAG (Retrieval-Augmented Generation)

> [Knowledge Tree](../../../README.md) / [AI](../../README.md) / [LLM](../README.md) / **RAG**

RAG kết hợp truy xuất tài liệu với LLM: **Question → Retrieve → Context → Generate → Cite**.

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
Kết hợp retrieval với generation để LLM trả lời dựa trên evidence từ kho tri thức.

### 2. Pipeline cơ bản?
Ingestion → chunking → embedding → index → retrieval → context → LLM → citation.

### 3. Chunking, embedding, retrieval liên hệ thế nào?
Chunking tạo đơn vị tri thức, embedding tạo vector, retrieval chọn evidence; lỗi ở một tầng sẽ ảnh hưởng answer.

### 4. Khi nào dùng hybrid search?
Khi kho có mã, tên riêng hoặc điều kiện exact-match bên cạnh truy vấn ngữ nghĩa.

### 5. Làm sao giảm hallucination?
Chỉ trả lời theo evidence, có citation, threshold và hành vi “không đủ thông tin”.

## B. 5 câu hỏi phổ biến mở rộng

### 6. RAG hay fine-tuning?
RAG cho knowledge thay đổi/dữ liệu riêng; fine-tuning cho phong cách hoặc hành vi ổn định.

### 7. Metadata và ACL để làm gì?
Filter nguồn, phiên bản và quyền trước retrieval, tránh lộ dữ liệu hoặc lấy tài liệu hết hiệu lực.

### 8. Khi nào không dùng RAG?
Dữ liệu realtime, tính toán chính xác và thao tác nghiệp vụ cần API/database/tool.

### 9. Đánh giá RAG bằng gì?
Đo Recall@k, Precision@k, groundedness, citation correctness, latency và cost.

### 10. Production RAG cần chú ý gì?
Versioning, evaluation, observability, retry, backup, ACL và rollout an toàn.

## Liên kết liên quan
- [LLM](../README.md)
- [AI](../../README.md)
- [Knowledge Tree](../../../README.md)
