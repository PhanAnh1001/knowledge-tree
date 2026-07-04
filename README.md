# Knowledge Tree

Kho kiến thức bằng tiếng Việt, tổ chức theo cây phân cấp (*mind-map tree*).

Mỗi khái niệm có:

- 5 câu hỏi và trả lời cốt lõi.
- 5 câu hỏi và trả lời phổ biến mở rộng.
- Thuật ngữ tiếng Anh khi phù hợp.
- Hướng dẫn áp dụng, bài toán giải quyết và các liên kết liên quan.

## Cấu trúc

```text
knowledge-tree/
├── README.md
├── metadata.json
└── {linh-vuc}/{linh-vuc-con}/.../{khai-niem}/README.md
```

Tên thư mục dùng `kebab-case`, không dấu. Nội dung hiển thị bằng tiếng Việt được định nghĩa trong từng README và `metadata.json`.

## Khám phá cây kiến thức

- [Trí tuệ nhân tạo (Artificial Intelligence, AI)](artificial-intelligence/README.md)
  - [Mô hình ngôn ngữ lớn (Large Language Models, LLM)](artificial-intelligence/large-language-models/README.md)
    - [RAG (Retrieval-Augmented Generation)](artificial-intelligence/large-language-models/retrieval-augmented-generation/README.md)

## Metadata và điều hướng

`metadata.json` ở thư mục gốc là nguồn dữ liệu cho cây kiến thức. Mỗi node khai báo `id`, `title`, `slug`, `type`, `path`, `parentId`, `order` và `children`.

Mỗi README của khái niệm cần có breadcrumb, liên kết về node cha, liên kết các node con và điều hướng khái niệm trước/sau trong cùng nhánh.

Ví dụ đường dẫn:

```text
software-engineering/architecture/clean-architecture/README.md
```

## Đóng góp

Khi thêm hoặc di chuyển một khái niệm, hãy cập nhật README của khái niệm và `metadata.json` trong cùng commit.
