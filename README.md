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

## Quy ước thuật ngữ

Dùng các từ viết tắt tiếng Anh phổ biến trong ngành khi chúng là tên khái niệm hoặc định danh ổn định, ví dụ: `ai`, `llm`, `rag`, `api`, `db`, `ui`, `ux`.

- Giải thích tên đầy đủ ở lần xuất hiện đầu tiên, ví dụ: *Retrieval-Augmented Generation (RAG)*.
- Không tự tạo từ viết tắt mơ hồ.
- `id` có thể dùng viết tắt chuyên ngành phổ biến như `ai`, `llm`, `rag`.
- Tên thư mục và đường dẫn README vẫn dùng `kebab-case` đầy đủ, ổn định và dễ đọc.

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
| `slug` | Tên định danh dạng `kebab-case`. |
| `type` | Loại node: `root`, `domain`, `category` hoặc `concept`. |
| `path` | Đường dẫn tới `README.md` của node. |
| `parentId` | ID của node cha; node gốc dùng `null`. |
| `order` | Thứ tự hiển thị giữa các node cùng cấp. |
| `children` | Danh sách ID các node con. |

Metadata mô tả cấu trúc cây; không cần lưu các field viết tắt riêng cho navigation. Giao diện hoặc công cụ có thể suy ra node cha, node con và thứ tự cùng cấp từ `parentId`, `children` và `order`.

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

Ví dụ đường dẫn:

```text
software-engineering/architecture/clean-architecture/README.md
```

## Đóng góp

Khi thêm hoặc di chuyển một khái niệm, hãy cập nhật README của khái niệm và `metadata.json` trong cùng commit.
