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

## Quy ước viết tắt cho metadata và navigation

`metadata.json` dùng các từ viết tắt tiếng Anh đã phổ biến trong ngành để dữ liệu ngắn gọn và ổn định. Ưu tiên viết thường, ví dụ: `ai`, `llm`, `rag`, `api`, `db`, `ui`, `ux`.

- Chỉ dùng từ viết tắt đã thông dụng hoặc được định nghĩa rõ tại node đầu tiên; không tự tạo viết tắt mơ hồ.
- `id` của khái niệm ưu tiên viết tắt chuyên ngành: `ai`, `llm`, `rag`.
- Đường dẫn file vẫn dùng `kebab-case` đầy đủ trong `p` và `pi`; không đổi đường dẫn chỉ để rút ngắn metadata.
- `nav` dùng các key: `bc` (*breadcrumb*), `up` (*parent*), `pv` (*previous*), `nx` (*next*), `ch` (*children*).

Các key metadata rút gọn:

| Key | Ý nghĩa |
| --- | --- |
| `v`, `loc`, `r`, `d` | version, locale, root ID, description |
| `ns`, `n`, `pi` | node schema, nodes, path index |
| `t`, `k`, `p`, `pid`, `o`, `ch` | title, kind, README path, parent ID, order, children |

## Metadata và điều hướng

`metadata.json` ở thư mục gốc là nguồn dữ liệu cho cây kiến thức. Mỗi node dùng `id`, `t`, `k`, `p`, `pid`, `o`, `nav` và `ch`.

Mỗi README của khái niệm cần có breadcrumb, liên kết về node cha, liên kết các node con và điều hướng khái niệm trước/sau trong cùng nhánh. Giá trị của các liên kết này phải khớp với `nav` trong `metadata.json`.

Ví dụ đường dẫn:

```text
software-engineering/architecture/clean-architecture/README.md
```

## Đóng góp

Khi thêm hoặc di chuyển một khái niệm, hãy cập nhật README của khái niệm và `metadata.json` trong cùng commit.
