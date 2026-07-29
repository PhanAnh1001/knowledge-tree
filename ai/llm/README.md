# Mô hình ngôn ngữ lớn (Large Language Models, LLM)

> [Knowledge Tree](../../README.md) / [Trí tuệ nhân tạo (AI)](../README.md) / **Mô hình ngôn ngữ lớn (LLM)**

## Vai trò trong cây kiến thức

*Mô hình ngôn ngữ lớn* (*Large Language Model — LLM*) là mô hình AI được huấn luyện trên lượng lớn dữ liệu ngôn ngữ để hiểu, tóm tắt, sinh và biến đổi văn bản. Các kỹ thuật như **RAG**, **Chain of Thought (CoT)**, *prompt engineering*, *tool calling*, **Model Context Protocol (MCP)**, *fine-tuning* và **Minimal Reproducible Example (MRE)** giúp đưa LLM vào ứng dụng thực tế một cách đáng tin cậy, có thể kiểm thử, kết nối được với hệ thống bên ngoài và dễ điều tra lỗi hơn.

## Điều hướng

- **Node cha:** [Trí tuệ nhân tạo (AI)](../README.md)
- **Node con:** [RAG (Retrieval-Augmented Generation)](rag/README.md), [Chain of Thought (CoT)](cot/README.md), [Minimal Reproducible Example (MRE)](mre/README.md), [Model Context Protocol (MCP)](mcp/README.md)
- **Khái niệm trước / sau trong nhánh:** Chưa có node cùng cấp để điều hướng.

## Nhánh con

1. [RAG (Retrieval-Augmented Generation)](rag/README.md) — Kết hợp truy xuất tri thức với sinh câu trả lời từ LLM.
2. [Chain of Thought (CoT)](cot/README.md) — Tổ chức suy luận nhiều bước, phân rã bài toán và kiểm chứng kết quả.
3. [Minimal Reproducible Example (MRE)](mre/README.md) — Đóng gói ca lỗi nhỏ nhất có thể tái hiện cho prompt AI, OCR PDF, trích xuất đề mục và hình ảnh trong tài liệu kỹ thuật.
4. [Model Context Protocol (MCP)](mcp/README.md) — Chuẩn hóa kết nối giữa ứng dụng AI với dữ liệu và công cụ; tối ưu context bằng progressive discovery, đọc theo nhu cầu và xử lý kết quả ngoài LLM.
