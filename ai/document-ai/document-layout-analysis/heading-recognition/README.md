# Nhận diện đề mục trong PDF scan bằng OCR và Gemma 4 (Heading Recognition)

> [Knowledge Tree](../../../../README.md) / [AI](../../../README.md) / [Document AI](../../README.md) / [Document Layout Analysis](../README.md) / **Heading Recognition**

**Nhận diện đề mục (heading recognition)** trong PDF scan không chỉ là đọc đúng ký tự bằng *Optical Character Recognition (OCR)*. Hệ thống còn phải xác định dòng nào là tiêu đề, phân biệt tiêu đề tài liệu với đề mục phần, suy ra cấp `H1/H2/H3...`, liên kết mỗi đề mục với nội dung bên dưới và loại các vùng gây nhiễu như đầu trang, chân trang, caption hoặc ô bảng in đậm.

> **Lưu ý về tên model:** Google phát hành biến thể chính thức **Gemma 4 31B**, không phải 32B. Nội dung dưới đây hiểu “Gemma 4 32B” là cách gọi gần đúng của `gemma-4-31b-it`.

Khuyến nghị cốt lõi là dùng **pipeline lai (hybrid pipeline)**: model layout/OCR chuyên dụng xử lý phần hình học và ký tự; Gemma 4 xử lý các trường hợp mơ hồ về vai trò ngữ nghĩa, cấp đề mục và quan hệ phân cấp. Không nên để Gemma nhận toàn bộ trang scan rồi tự vừa OCR, vừa tìm bbox, vừa dựng hierarchy trong một lần duy nhất.

```text
PDF scan
   │
   ├─ Render ảnh gốc chất lượng cao
   ├─ Orientation → deskew → dewarp → denoise/contrast
   │
   ├─ Layout detector
   │     └─ document_title, section_header, text, table, caption,
   │        page_header, page_footer, list, warning...
   │
   ├─ OCR theo block/line + bbox + confidence
   │
   ├─ Heading candidate builder
   │     └─ text, bbox, line height, whitespace, numbering,
   │        alignment, repetition, neighboring blocks
   │
   ├─ Gemma 4 semantic resolver
   │     └─ role, heading level, normalized text, parent candidate,
   │        confidence, needs_review
   │
   ├─ Deterministic hierarchy builder + validation
   │
   └─ Canonical document JSON → Markdown/HTML/RAG/index
```

## Điều hướng

- **Node cha:** [Phân tích bố cục tài liệu (Document Layout Analysis)](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** Chưa có.
- **Khái niệm sau trong cùng nhánh:** Chưa có.

---

## A. 5 câu hỏi cốt lõi

### 1. Vì sao OCR đọc đúng chữ nhưng vẫn nhận sai heading?

**Trả lời:** OCR chủ yếu trả lời “dòng này ghi gì?”, còn nhận diện heading phải trả lời thêm “dòng này đóng vai trò gì trong cấu trúc tài liệu?”. Một dòng chữ lớn chưa chắc là heading; nó có thể là tiêu đề bảng, nhãn cảnh báo, tên hình, đầu trang lặp lại hoặc nội dung được nhấn mạnh.

Heading recognition gồm ít nhất bốn tác vụ khác nhau:

1. **Text recognition:** đọc chính xác ký tự.
2. **Layout role classification:** phân loại `document_title`, `section_heading`, `body`, `caption`, `header`, `footer`...
3. **Heading level classification:** suy ra `H1/H2/H3...` hoặc `level=null` khi không đủ bằng chứng.
4. **Hierarchy recovery:** xác định đề mục cha, section bắt đầu và phạm vi nội dung thuộc section.

Các model layout như DocLayNet/PP-DocLayout thường có nhãn `Title` hoặc `Section-header`, nhưng việc suy ra cấp heading và cây phân cấp vẫn cần thêm quy tắc, sequence model hoặc VLM. DocParser cũng xem khôi phục cấu trúc phân cấp là bài toán riêng, không chỉ là detection bounding box.

**Áp dụng khi nào?**

- Manual thiết bị, quy trình vận hành, bảo trì và sửa chữa.
- PDF scan có nhiều cấp chương/mục nhưng font không hoàn toàn nhất quán.
- Tài liệu cần chuyển sang Markdown, HTML hoặc chunk theo section cho RAG.
- Cần tạo mục lục tự động hoặc citation theo section.

**Bài toán giải quyết:**

- Tránh gắn đoạn văn in đậm thành heading.
- Tránh bỏ sót heading nhỏ nhưng có đánh số rõ.
- Tránh chunk sai ranh giới section.
- Giữ được `sectionPath`, ví dụ `5. Bảo trì > 5.2 Thay vòng bi > Cảnh báo`.

**So sánh:**

| Đầu ra | Đọc được chữ | Biết vùng | Biết vai trò | Biết cấp | Biết cha–con |
| --- | --- | --- | --- | --- | --- |
| OCR plain text | Có | Không | Không | Không | Không |
| OCR có bbox | Có | Có | Không | Không | Không |
| Layout detector | Có thể cần OCR | Có | Có | Thường chưa đủ | Không |
| Hybrid OCR + layout + Gemma + rules | Có | Có | Có | Có | Có |

---

### 2. Kiến trúc nào phù hợp nhất khi đang dùng Gemma 4 31B?

**Trả lời:** Dùng Gemma 4 31B như **semantic resolver/fallback**, không dùng làm detector và OCR duy nhất cho mọi trang.

Pipeline khuyến nghị:

1. **Render master image:** tạo ảnh trang chất lượng cao và giữ nguyên để crop lại.
2. **Preprocess:** phát hiện chiều, deskew, dewarp, cân bằng nền và giảm nhiễu.
3. **Layout detection:** dùng PP-StructureV3/PP-DocLayout, Surya hoặc DocLayout-YOLO để lấy vùng và nhãn sơ bộ.
4. **OCR theo vùng:** đọc từng block/line, giữ polygon/bbox và confidence.
5. **Candidate routing:** chỉ đưa các block có khả năng là heading hoặc bị model bất đồng sang Gemma.
6. **Gemma classification:** phân loại role, cấp heading, text chuẩn hóa và quan hệ với heading trước đó.
7. **Hierarchy rules:** dùng thuật toán quyết định để sửa các cấu trúc bất hợp lý và dựng stack heading.
8. **Quality gate:** đánh dấu `needs_review` khi confidence thấp, model bất đồng hoặc hierarchy vi phạm rule.

PP-StructureV3 có các lớp như document title và paragraph title; Surya có nhãn `SectionHeader`, `PageHeader`, reading order và bbox. Gemma 4 có khả năng OCR, document/PDF parsing và xử lý ảnh với token budget thay đổi, phù hợp cho phần suy luận ngữ nghĩa hoặc trang khó.

**Áp dụng khi nào?**

- Corpus lớn, cần kiểm soát chi phí và khả năng debug.
- Tài liệu nhiều loại: manual, catalogue, biểu mẫu, báo cáo, scan cũ.
- Cần giữ bbox và provenance để kiểm toán hoặc hiển thị citation.

**Bài toán giải quyết:** tách lỗi detector, OCR và hierarchy thành các stage có thể đo độc lập; giảm việc model lớn phải đọc toàn bộ mọi trang.

**So sánh:**

| Kiến trúc | Điểm mạnh | Giới hạn | Khuyến nghị |
| --- | --- | --- | --- |
| Gemma end-to-end toàn trang | Prototype nhanh, hiểu ngữ nghĩa tốt | Nặng, khó debug, dễ mất chữ nhỏ/bbox | Chỉ dùng thử nghiệm hoặc fallback |
| Layout + OCR thuần | Nhanh, ổn định, giữ tọa độ | Khó suy ra H1/H2/H3 và heading mơ hồ | Baseline production tốt |
| Layout + OCR + rules | Rẻ, dễ giải thích | Rule tăng nhanh theo domain | Hợp template ổn định |
| Layout + OCR + Gemma + rules | Cân bằng chất lượng, chi phí, debug | Pipeline nhiều thành phần | Lựa chọn mặc định khuyến nghị |

---

### 3. Nên tiền xử lý ảnh và crop heading như thế nào?

**Trả lời:** Tạo một **ảnh master** ở độ phân giải đủ cao, nhưng cung cấp cho từng model một ảnh dẫn xuất phù hợp thay vì dùng duy nhất một ảnh cho toàn pipeline.

Quy trình khởi đầu thực tế:

1. Render trang scan khoảng **300 DPI**; tăng lên 350–400 DPI cho chữ rất nhỏ hoặc scan mờ.
2. Phân loại hướng `0/90/180/270°` trước OCR.
3. Deskew cho lệch nhỏ; dewarp nếu trang cong hoặc chụp từ camera.
4. Giữ hai phiên bản:
   - ảnh grayscale gần bản gốc để bảo toàn bold, màu, đường kẻ và hierarchy thị giác;
   - ảnh enhanced/binarized để OCR chữ mờ.
5. Chạy layout trên ảnh được resize theo model, nhưng crop từ ảnh master.
6. Mở rộng crop một khoảng nhỏ quanh bbox để giữ dấu, số thứ tự và khoảng trắng trước/sau.
7. Với chữ nhỏ, upscale crop trước khi OCR/Gemma; không phóng toàn trang nếu chỉ một vùng khó.
8. Cung cấp thêm thumbnail toàn trang hoặc block trước/sau để Gemma không mất ngữ cảnh.

Gemma 4 hỗ trợ visual token budget `70, 140, 280, 560, 1120`. Với trang dày chữ hoặc crop có chữ nhỏ, nên benchmark `560` và `1120`; ngân sách cao giữ nhiều chi tiết hơn nhưng tăng compute. Không nên coi token budget cao là thay thế cho layout crop vì một trang A4 dày chữ vẫn có quá nhiều chi tiết nhỏ.

**Áp dụng khi nào?** scan photocopy, tài liệu nghiêng, nền xám, chữ nhỏ, font condensed hoặc heading sát bảng/hình.

**Bài toán giải quyết:** giảm mất dấu tiếng Việt, tách dòng sai, gộp heading với đoạn sau và bỏ sót heading nhỏ.

**So sánh:**

| Cách xử lý ảnh | Ưu điểm | Rủi ro |
| --- | --- | --- |
| Chỉ ảnh gốc | Giữ phong cách thị giác | OCR kém nếu nền/nhiễu xấu |
| Chỉ binarized | Chữ tương phản cao | Có thể mất bold, màu và nét mảnh |
| Dual-view | Kết hợp OCR và style tốt hơn | Tăng storage/compute |
| Whole-page high resolution | Giữ context | Tốn token, chữ nhỏ vẫn có thể bị nén |
| ROI crop + page thumbnail | Chi tiết tốt và vẫn có context | Cần layout detector đáng tin cậy |

---

### 4. Nên cung cấp tín hiệu và prompt nào cho Gemma để nhận heading ổn định?

**Trả lời:** Không chỉ gửi ảnh. Hãy gửi **ảnh + OCR text + geometry + style proxy + context theo thứ tự đọc** và giới hạn output bằng schema rõ ràng.

Mỗi candidate nên có:

```json
{
  "id": "p012-b007",
  "page": 12,
  "bbox_norm": [0.083, 0.191, 0.712, 0.238],
  "ocr_text": "5.2 THAY VÒNG BI TRỤC CHÍNH",
  "ocr_confidence": 0.93,
  "layout_label": "SectionHeader",
  "layout_confidence": 0.88,
  "line_height_ratio": 1.42,
  "gap_before_ratio": 1.8,
  "gap_after_ratio": 0.7,
  "is_centered": false,
  "numbering_pattern": "5.2",
  "repetition_rate": 0.0,
  "previous_block_text": "5.1 Kiểm tra độ rơ",
  "next_block_text": "Tháo nắp che và khóa nguồn..."
}
```

Output nên tách text nhìn thấy và text chuẩn hóa:

```json
{
  "id": "p012-b007",
  "role": "section_heading",
  "level": 2,
  "text_raw": "5.2 THAY VÒNG BI TRỤC CHÍNH",
  "text_normalized": "5.2 Thay vòng bi trục chính",
  "parent_heading_id": "p010-b003",
  "confidence": 0.96,
  "needs_review": false,
  "evidence_tags": ["numbering_depth_2", "large_text", "space_before"]
}
```

Prompt rút gọn:

```text
Bạn là bộ phân tích cấu trúc tài liệu kỹ thuật.
Ảnh được đặt trước phần mô tả text.

Chỉ phân loại các candidate đã cung cấp; không tự tạo bbox mới.
Giữ nguyên text_raw theo OCR/ảnh. text_normalized chỉ sửa lỗi rõ ràng.
role chỉ nhận một trong:
document_title, section_heading, body, list_item, warning_title,
figure_caption, table_caption, page_header, page_footer, other.

level chỉ dùng 1..6 khi role=section_heading; nếu không đủ bằng chứng trả null.
Dùng numbering, kích thước tương đối trong cùng tài liệu, khoảng trắng,
vị trí, block lân cận và heading stack. Không coi chữ in đậm là heading
nếu không bắt đầu một section mới.

Trả JSON đúng schema; trường hợp mơ hồ đặt needs_review=true.
```

Sau Gemma, luôn chạy validator:

- `role != section_heading` thì `level = null`;
- không nhận `page_header/footer` lặp lại làm heading;
- không để heading con đứng trước heading cha nếu numbering mâu thuẫn;
- không cho model đổi bbox;
- parse JSON lỗi thì retry một lần với output lỗi, sau đó fallback.

**Áp dụng khi nào?** heading mơ hồ, font không ổn định, tài liệu đa ngôn ngữ hoặc cần hierarchy thay vì chỉ label.

**Bài toán giải quyết:** giảm hallucination, output tự do, model tự sửa quá mức và cấp heading không nhất quán.

**So sánh:**

| Input cho Gemma | Chất lượng dự kiến | Khả năng debug |
| --- | --- | --- |
| Chỉ ảnh toàn trang | Không ổn định với chữ nhỏ | Thấp |
| Ảnh crop | Đọc chữ tốt hơn nhưng thiếu context | Trung bình |
| Crop + thumbnail | Cân bằng chi tiết/context | Khá |
| Crop + OCR + geometry + neighbors | Tốt nhất cho role/hierarchy | Cao |

---

### 5. Đánh giá heading recognition bằng metric nào?

**Trả lời:** Tách metric theo stage và thêm metric end-to-end. Chỉ đo OCR CER hoặc layout mAP là chưa đủ.

| Thành phần | Metric gợi ý |
| --- | --- |
| Phát hiện vùng heading | Precision, Recall, F1, AP/mAP theo IoU |
| OCR text heading | Character Error Rate (CER), Word Error Rate (WER), normalized exact match |
| Phân loại role | Macro-F1 và confusion matrix |
| Phân loại cấp | Accuracy/macro-F1 trên `H1..H6` |
| Gán cha–con | Parent-edge precision/recall/F1 |
| Ranh giới section | Section-boundary precision/recall/F1 |
| Cây tài liệu | Tree edit distance hoặc relation F1 |
| Markdown cuối | Heading sequence match và structural similarity |
| RAG | Retrieval recall theo section, citation accuracy, answer grounding |

Golden set nên chia theo:

- loại tài liệu và nhà cung cấp;
- tiếng Việt/Anh/song ngữ;
- scan rõ, mờ, nghiêng, photocopy nhiều lần;
- heading đánh số và không đánh số;
- một cột, nhiều cột, bảng/hình chen giữa;
- hard negatives như caption, warning, header/footer và TOC.

Chia train/validation/test theo **document family**, không random theo trang, để tránh các trang cùng template xuất hiện ở cả train và test.

**Áp dụng khi nào?** chọn model, chỉnh prompt, thay preprocessing, fine-tune, rollout phiên bản mới hoặc đánh giá tác động tới RAG.

**Bài toán giải quyết:** tránh trường hợp OCR tốt nhưng hierarchy sai; detector mAP cao nhưng bỏ heading nhỏ; Markdown đẹp nhưng sectionPath không đúng.

**So sánh:**

- Public benchmark giúp sàng lọc model.
- Golden set nội bộ phản ánh đúng domain.
- End-to-end RAG test đo giá trị nghiệp vụ.

Cần dùng cả ba, trong đó golden set nội bộ là source of truth cho quyết định production.

---

## B. 8 câu hỏi phổ biến mở rộng

### 6. Nên gửi toàn trang hay crop từng heading cho Gemma?

**Trả lời:** Dùng mô hình **hai góc nhìn (two-view input)**:

1. thumbnail hoặc ảnh toàn trang để hiểu cột, vùng, phong cách và vị trí;
2. crop candidate ở độ phân giải cao để đọc chữ và nhận style chi tiết.

Khi trang dày chữ, chỉ gửi toàn trang sẽ khiến heading nhỏ bị nén trong representation. Chỉ gửi crop lại dễ nhầm caption hoặc label vì thiếu vị trí và ngữ cảnh. Với nhiều candidate, có thể gửi một page thumbnail cùng các crop được đánh `candidate_id` theo reading order.

**Áp dụng khi nào?** trang manual có heading nhỏ, nhiều block, nhiều cột hoặc caption gần heading.

**Bài toán giải quyết:** cân bằng local detail và global context.

**So sánh:**

| Chiến lược | Chọn khi |
| --- | --- |
| Whole page only | Trang rất đơn giản, chữ lớn |
| Crop only | Candidate đã được detector xác định chắc chắn |
| Page + crop | Mặc định cho trang phức tạp |
| Page + nhiều crop + metadata | Dựng hierarchy và xử lý hàng loạt candidate |

---

### 7. Làm thế nào phân biệt heading với chữ in đậm, caption, đầu trang và nội dung bảng?

**Trả lời:** Kết hợp tín hiệu trong trang và xuyên trang; không dựa riêng vào font lớn/in đậm.

Các hard negative quan trọng:

- **Page header/footer:** vị trí gần biên và lặp lại trên nhiều trang.
- **Figure/table caption:** gần figure/table, thường có tiền tố `Hình`, `Figure`, `Bảng`, `Table`.
- **Warning title:** có icon, khung, màu hoặc từ `CẢNH BÁO/WARNING`; có thể là subtype riêng thay vì H-level.
- **Table cell header:** nằm trong bbox bảng, có hàng/cột lân cận.
- **TOC entry:** nhiều dòng có số trang/dotted leader; không bắt đầu section thật.
- **Bold lead-in:** chữ in đậm đầu đoạn nhưng phần sau tiếp tục cùng dòng hoặc không có khoảng trắng section.

Rule hữu ích:

```text
Nếu block lặp cùng vị trí/text trên nhiều trang → page_header/footer.
Nếu block nằm trong table bbox → table content, không phải document heading.
Nếu block sát figure/table và có pattern caption → caption.
Nếu TOC page hoặc có dotted leader + page number → toc_entry.
Nếu sau block là body paragraphs và có khoảng trắng trước rõ → tăng heading score.
```

**Áp dụng khi nào?** báo cáo, manual, catalogue, form, tài liệu có nhiều khung cảnh báo.

**Bài toán giải quyết:** giảm false positive làm vỡ section tree.

**So sánh:** rule xuyên trang tốt cho header/footer; layout relation tốt cho caption/table; Gemma tốt cho trường hợp semantically ambiguous.

---

### 8. Làm thế nào suy ra H1/H2/H3 khi font không nhất quán?

**Trả lời:** Suy ra cấp heading theo **document-local hierarchy**, không dùng một ngưỡng font size toàn corpus.

Thứ tự tín hiệu nên ưu tiên:

1. numbering depth: `1` → level 1, `1.2` → level 2, `1.2.3` → level 3;
2. quan hệ với heading trước/sau và heading stack;
3. style cluster trong cùng tài liệu: line height, boldness proxy, alignment, indentation;
4. whitespace trước/sau;
5. vị trí đầu trang/chương;
6. semantic cues như `Chương`, `Phần`, `Mục`, `Chapter`, `Section`.

Thuật toán stack đơn giản:

```text
for heading in reading_order:
    predicted_level = numbering_level or style_level or gemma_level
    predicted_level = validate_against_previous_stack(predicted_level)
    pop stack until stack.top.level < predicted_level
    parent = stack.top or document_root
    push heading
```

Không ép `document_title` thành `H1`. Nếu tài liệu nhảy từ H1 sang style giống H3 nhưng không có H2, nên đánh dấu bất thường hoặc cho phép theo rule domain, không tự động renumber âm thầm.

**Áp dụng khi nào?** tài liệu ghép từ nhiều nguồn, scan qua nhiều phiên bản, heading không đánh số hoặc font bị biến dạng do scan.

**Bài toán giải quyết:** tạo sectionPath ổn định cho Markdown, tìm kiếm và RAG.

**So sánh:** numbering rule rất chính xác khi có số; style clustering phù hợp tài liệu nhất quán; Gemma giải quyết semantic edge case; hierarchy validator giữ tính hợp lệ toàn tài liệu.

---

### 9. Khi nào chỉ cần prompt/rule, khi nào cần fine-tune Gemma?

**Trả lời:** Tối ưu theo thứ tự từ rẻ đến đắt:

1. sửa render, orientation, crop và OCR;
2. map taxonomy và thêm deterministic rules;
3. cải thiện prompt/schema và candidate context;
4. calibrate threshold/routing;
5. fine-tune khi lỗi còn lại lặp lại có tính domain.

Nên fine-tune khi:

- model liên tục nhầm cùng một loại heading nội bộ;
- tiếng Việt/chữ viết tắt ngành khiến semantic classification kém;
- style/template doanh nghiệp ổn định nhưng khác dữ liệu pretraining;
- prompt dài dần nhưng lỗi không giảm;
- có golden set và dữ liệu gán nhãn đủ đại diện.

Google cung cấp hướng dẫn fine-tune Gemma vision bằng **Quantized Low-Rank Adaptation (QLoRA)**. QLoRA giữ model nền ở 4-bit và chỉ train adapter, phù hợp hơn full fine-tuning. Tuy nhiên, nên benchmark Gemma 4 12B hoặc 26B A4B cho tác vụ classifier trước khi chọn 31B vì heading classification sau layout/OCR thường không cần model lớn nhất.

**Áp dụng khi nào?** domain ổn định, volume lớn và lỗi hệ thống có tác động rõ tới downstream.

**Bài toán giải quyết:** giảm domain shift mà không phải full fine-tune 31B.

**So sánh:**

| Cách | Dữ liệu cần | Chi phí | Độ ổn định |
| --- | --- | --- | --- |
| Prompt | Ít | Thấp | Trung bình |
| Rules | Ít | Thấp | Cao với pattern rõ |
| Classifier nhỏ | Vừa | Thấp–trung bình | Cao cho taxonomy cố định |
| QLoRA Gemma | Nhiều hơn | Trung bình–cao | Tốt cho case đa dạng |
| Full fine-tune | Rất nhiều | Rất cao | Chỉ hợp bài toán đặc thù lớn |

---

### 10. Dataset fine-tune heading nên có những gì?

**Trả lời:** Dataset phải chứa cả positive và **hard negative**, đồng thời giữ context theo trang/tài liệu.

Taxonomy gợi ý:

```text
document_title, h1, h2, h3, h4, h5, h6,
body, list_item, warning_title,
figure_caption, table_caption,
page_header, page_footer, toc_entry, other
```

Mỗi sample nên lưu:

- page image và crop;
- bbox/polygon chuẩn hóa;
- OCR text raw;
- role và level;
- parent heading id;
- block trước/sau;
- loại tài liệu, ngôn ngữ, nguồn scan;
- flags về blur, skew, low contrast, multi-column;
- annotation provenance và adjudication status.

Dữ liệu augmentation nên mô phỏng lỗi thật:

- blur, JPEG artifacts, photocopy noise;
- skew/perspective/warp;
- nền không đều, vết bẩn, mất nét;
- scale nhỏ, font condensed;
- crop sát hoặc crop thừa;
- lỗi OCR tiếng Việt như mất dấu hoặc nhầm `I/l/1`, `O/0`.

Quy trình thực tế:

1. lấy mẫu theo document family và độ khó;
2. gán nhãn một baseline set;
3. chạy model;
4. dùng active learning lấy disagreement/confidence thấp;
5. double annotation cho subset khó;
6. khóa test set trước fine-tune.

**Áp dụng khi nào?** chuẩn bị classifier, fine-tune detector hoặc QLoRA VLM.

**Bài toán giải quyết:** tránh model chỉ học heading dễ và thất bại đúng các edge case production.

**So sánh:** dữ liệu ngẫu nhiên nhiều trang không tốt bằng dữ liệu ít hơn nhưng cân bằng domain, class và hard negative.

---

### 11. Nên kết hợp Gemma với model mã nguồn mở nào?

**Trả lời:** Chọn theo tầng trách nhiệm, không coi các model là hoàn toàn thay thế nhau.

| Lựa chọn | Vai trò tốt nhất | Điểm mạnh | Giới hạn |
| --- | --- | --- | --- |
| PP-StructureV3 / PP-DocLayout | Layout + title/paragraph title + OCR pipeline | Nhiều module, bbox rõ, có thể fine-tune | Cần map taxonomy và benchmark tiếng Việt/domain |
| Surya | OCR + layout + reading order | Có `SectionHeader`, `PageHeader`, bbox và reading order | Cần kiểm tra license model và chất lượng corpus thật |
| DocLayout-YOLO | Detector layout nhanh | Phù hợp large-scale detection | Cần OCR và hierarchy stage riêng |
| Gemma 4 31B | Semantic resolver, hierarchy, fallback | Hiểu context và multi-image tốt | Nặng, output sinh cần validation |
| Gemma 4 12B/26B A4B | Resolver production tiết kiệm hơn | Chi phí thấp hơn 31B | Cần benchmark accuracy |
| Rules/style classifier | Post-processing và hard constraints | Rẻ, deterministic | Dễ tăng ngoại lệ nếu dùng đơn độc |

Thiết kế khuyến nghị:

```text
PP-DocLayout/Surya → OCR → rule candidate filter
                           ├─ high confidence: accept
                           ├─ medium/disagreement: Gemma
                           └─ critical/low confidence: human review
```

**Áp dụng khi nào?** chọn stack cho POC hoặc thay thế pipeline OCR hiện tại.

**Bài toán giải quyết:** dùng model chuyên dụng cho phần hình học và model lớn cho phần reasoning có ROI cao.

**So sánh:** PP-StructureV3 và Surya là pipeline document chuyên dụng; DocLayout-YOLO là detector; Gemma là VLM tổng quát. So sánh phải dùng cùng golden set và cùng canonical output.

---

### 12. GPU 64 GB nên chạy Gemma 4 31B thế nào để không lãng phí?

**Trả lời:** 31 tỷ tham số ở BF16 cần khoảng **62 GB chỉ cho trọng số** theo phép tính `31B × 2 byte`, chưa tính KV cache, activation, runtime và ảnh. Vì vậy một GPU 64 GB không phải cấu hình thoải mái cho full-precision inference, càng không phù hợp full fine-tuning.

Khuyến nghị:

- dùng 4-bit/8-bit inference hoặc model 26B A4B/12B;
- batch layout/OCR riêng, không giữ Gemma cho mọi stage;
- chỉ route candidate mơ hồ sang Gemma;
- gom các crop có kích thước/token budget tương tự thành batch;
- cache theo `page_hash + crop_bbox + model_version + prompt_version`;
- dùng queue riêng cho VLM và giới hạn concurrency theo VRAM;
- lưu token budget, latency, output length và OOM/retry;
- tránh gửi toàn bộ PDF nhiều trang vào một request;
- dùng context document-level dạng heading stack/metadata thay vì ảnh tất cả trang trước.

Với QLoRA, trọng số nền được quantize 4-bit và adapter được train; vẫn cần benchmark memory theo framework, image token budget, batch size và sequence length. Bắt đầu với model nhỏ hơn để xác nhận dataset/prompt trước khi tiêu tốn tài nguyên cho 31B.

**Áp dụng khi nào?** chạy local/self-hosted trên một GPU 64 GB hoặc xây worker GPU chia sẻ cho nhiều tài liệu.

**Bài toán giải quyết:** tránh OOM, throughput thấp và chi phí inference không cần thiết.

**So sánh:**

- 31B phù hợp fallback chất lượng cao.
- 26B A4B phù hợp cân bằng compute/chất lượng.
- 12B phù hợp resolver production nếu golden set đạt yêu cầu.
- detector/classifier nhỏ phù hợp fast path.

---

### 13. Nên xử lý confidence, fallback và tích hợp RAG thế nào?

**Trả lời:** Mỗi heading phải có provenance và confidence theo stage; không gộp thành một điểm mơ hồ duy nhất.

Canonical output gợi ý:

```json
{
  "heading_id": "docA-p012-b007",
  "page": 12,
  "bbox": [205, 670, 1765, 835],
  "text_raw": "5.2 THAY VÒNG BI TRỤC CHÍNH",
  "text_normalized": "5.2 Thay vòng bi trục chính",
  "role": "section_heading",
  "level": 2,
  "parent_heading_id": "docA-p010-b003",
  "section_path": ["5. Bảo trì", "5.2 Thay vòng bi trục chính"],
  "confidence": {
    "layout": 0.88,
    "ocr": 0.93,
    "semantic": 0.96,
    "hierarchy": 0.91
  },
  "provenance": {
    "layout_model": "...",
    "ocr_model": "...",
    "vlm_model": "google/gemma-4-31b-it",
    "prompt_version": "heading-v3",
    "pipeline_version": "2026-07-29"
  },
  "needs_review": false
}
```

Routing ví dụ cần được calibrate bằng golden set:

```text
layout/OCR/rule đồng thuận cao → accept fast path
model bất đồng hoặc confidence trung bình → Gemma resolver
Gemma + rules vẫn bất đồng → needs_review
trang quan trọng và confidence thấp → human review
```

Khi tạo dữ liệu RAG:

- chunk bắt đầu bằng heading, không để heading ở cuối chunk trước;
- lưu `sectionPath`, `page`, `bbox`, `headingId`, `documentVersion`;
- loại page header/footer lặp lại;
- giữ warning/table/figure như object riêng nhưng gắn section cha;
- render citation highlight từ bbox gốc;
- đo retrieval recall và citation accuracy theo section.

**Áp dụng khi nào?** pipeline production, dữ liệu kỹ thuật có yêu cầu truy vết hoặc RAG cần trả đúng section nguồn.

**Bài toán giải quyết:** fallback có kiểm soát, rollback được, reprocess theo version và tránh lỗi heading lan sang chunking/retrieval.

**So sánh:** confidence một số duy nhất đơn giản nhưng khó debug; confidence theo stage cho phép route đúng model và biết lỗi nằm ở render, layout, OCR hay hierarchy.

---

## Checklist triển khai ưu tiên

1. Xác nhận đang dùng đúng model `gemma-4-31b-it`, không phải tên 32B tự đặt.
2. Tạo golden set theo document family, ngôn ngữ và độ khó.
3. Đo riêng layout recall, OCR CER, role F1, level F1 và parent-edge F1.
4. Render master 300 DPI; thêm 350–400 DPI chỉ cho nhóm chữ nhỏ/mờ.
5. Dùng layout detector lấy candidate và bbox trước Gemma.
6. OCR theo block, giữ raw text, polygon và confidence.
7. Loại header/footer, table cell, caption và TOC bằng relation/rule.
8. Gửi Gemma page thumbnail + crop + OCR + geometry + neighbors.
9. Dùng JSON schema, enum, validator và `needs_review`.
10. Dựng heading stack bằng code quyết định, không giao toàn bộ cho model sinh.
11. Route chỉ candidate mơ hồ sang 31B; benchmark 12B/26B A4B cho fast path.
12. Chỉ QLoRA sau khi error taxonomy cho thấy lỗi domain lặp lại.
13. Version model, prompt, preprocessing, taxonomy và canonical schema.

## Tài liệu liên quan trong Knowledge Tree

- [Phân tích bố cục tài liệu (Document Layout Analysis)](../README.md)
- [OCR và trích xuất ảnh con bằng mô hình mã nguồn mở](../../../llm/rag/etl/ocr-image-extraction/README.md)
- [Minimal Reproducible Example (MRE) cho AI và OCR tài liệu kỹ thuật](../../../llm/mre/README.md)
- [Kiểm thử hồi quy đa phương thức (Multimodal Regression Engineering)](../../../multimodal-ai/multimodal-regression-engineering/README.md)

## Tài liệu tham khảo

- [Gemma releases — Google AI for Developers][gemma-releases]
- [Gemma 4 model card — Google AI for Developers][gemma-model-card]
- [Gemma vision understanding and variable resolution][gemma-vision]
- [Fine-tune Gemma vision tasks with Transformers and QLoRA][gemma-qlora]
- [PP-StructureV3 pipeline — PaddleOCR documentation][pp-structure]
- [Surya OCR, layout and reading order][surya]
- [DocLayNet: A Large Human-Annotated Dataset for Document-Layout Analysis][doclaynet]
- [PP-DocLayout: A Unified Document Layout Detection Model][pp-doclayout]
- [DocLayout-YOLO: Document Layout Analysis][doclayout-yolo]
- [DocParser: Hierarchical Structure Parsing of Document Renderings][docparser]

[gemma-releases]: https://ai.google.dev/gemma/docs/releases
[gemma-model-card]: https://ai.google.dev/gemma/docs/core/model_card_4
[gemma-vision]: https://ai.google.dev/gemma/docs/capabilities/vision
[gemma-qlora]: https://ai.google.dev/gemma/docs/core/huggingface_vision_finetune_qlora
[pp-structure]: https://www.paddleocr.ai/main/en/version3.x/pipeline_usage/PP-StructureV3.html
[surya]: https://github.com/datalab-to/surya
[doclaynet]: https://arxiv.org/abs/2206.01062
[pp-doclayout]: https://arxiv.org/abs/2503.17213
[doclayout-yolo]: https://arxiv.org/abs/2410.12628
[docparser]: https://arxiv.org/abs/1911.01702
