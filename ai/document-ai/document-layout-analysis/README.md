# Phân tích bố cục tài liệu (Document Layout Analysis)

> [Knowledge Tree](../../../README.md) / [AI](../../README.md) / [Document AI](../README.md) / **Document Layout Analysis**

**Document Layout Analysis (DLA)** là quá trình phát hiện, phân loại và liên kết các vùng trên trang tài liệu để biến ảnh trang hoặc PDF thành cấu trúc máy có thể xử lý. DLA thường xác định các thành phần như tiêu đề, đoạn văn, danh sách, bảng, hình, chú thích, công thức, đầu trang và chân trang; sau đó khôi phục thứ tự đọc và quan hệ giữa chúng.

DLA nằm giữa bước đọc file/tiền xử lý và các bước OCR, nhận dạng bảng, trích xuất trường dữ liệu, chuyển đổi Markdown/HTML hoặc tạo dữ liệu cho RAG. Một hệ thống tốt không chỉ trả về các hình chữ nhật mà còn phải bảo toàn ý nghĩa và trật tự của tài liệu.

```text
PDF/Image
   │
   ├─ Preflight: digital, scan, hybrid, rotation, DPI
   │
   ├─ Page rendering / native PDF geometry
   │
   ├─ Layout detection or segmentation
   │      └─ title, text, list, table, figure, caption, header...
   │
   ├─ OCR or native-text alignment per region
   │
   ├─ Reading order + hierarchy + region relationships
   │
   ├─ Specialized parsing: table, formula, form, diagram
   │
   └─ Canonical document model → Markdown/JSON/RAG/index
```

## Điều hướng

- **Node cha:** [Trí tuệ tài liệu (Document AI)](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** Chưa có.
- **Khái niệm sau trong cùng nhánh:** Chưa có.

---

## A. 5 câu hỏi cốt lõi

### 1. Document Layout Analysis là gì và đầu ra cần có những gì?

**Trả lời:** DLA là bước phân tích **cấu trúc vật lý của trang** (*physical layout*) và, ở mức cao hơn, suy ra một phần **cấu trúc logic** (*logical structure*). Thay vì xem trang như một ảnh phẳng hoặc một chuỗi ký tự, DLA chia trang thành các vùng đồng nhất, gán loại cho từng vùng và mô tả quan hệ giữa chúng. Survey về DLA xem đây là bước tiền xử lý quan trọng của hệ thống hiểu tài liệu, gồm tiền xử lý, phân tích bố cục, hậu xử lý và đánh giá. [DLA survey][dla-survey]

Đầu ra tối thiểu nên có:

```json
{
  "page": 12,
  "width": 2480,
  "height": 3508,
  "regions": [
    {
      "id": "p12-r07",
      "type": "figure",
      "bbox": [180, 720, 2110, 2050],
      "polygon": null,
      "confidence": 0.96,
      "readingOrder": 8,
      "parentId": "section-5-2",
      "relatedTo": ["p12-r08"]
    }
  ]
}
```

Các field quan trọng gồm:

- `type`: loại vùng như `title`, `text`, `list`, `table`, `figure`, `caption`, `formula`, `header`, `footer`;
- `bbox` hoặc `polygon`: tọa độ vùng;
- `confidence`: độ tin cậy để route fallback hoặc human review;
- `readingOrder`: thứ tự đọc;
- `parentId` hoặc `sectionPath`: quan hệ phân cấp;
- `relatedTo`: liên kết figure–caption, table–caption, footnote–reference;
- `source`: engine, model, version và tham số xử lý.

**Áp dụng khi nào?**

- PDF nhiều cột, manual thiết bị, báo cáo kỹ thuật, catalogue, biểu mẫu hoặc tài liệu có hình và bảng.
- Cần chuyển tài liệu sang Markdown/HTML nhưng vẫn giữ đúng thứ tự và cấu trúc.
- Cần crop hình, bảng hoặc sơ đồ theo vùng.
- Cần chunk tài liệu theo section cho RAG thay vì cắt chuỗi ký tự tùy ý.

**Bài toán giải quyết:**

- Tránh trộn tiêu đề, chân trang, hai cột hoặc chú thích vào cùng một đoạn.
- Giữ liên kết giữa hình và caption, bảng và tiêu đề bảng.
- Cung cấp tọa độ để mở đúng vùng nguồn khi citation.
- Tạo hợp đồng dữ liệu chung giữa nhiều OCR/layout engine.

**So sánh:**

| Cách biểu diễn | Có text | Có tọa độ | Có loại vùng | Có reading order | Phù hợp nhất |
| --- | --- | --- | --- | --- | --- |
| Plain text | Có | Không | Không | Thường chỉ ngầm định | Tài liệu đơn giản, một cột |
| OCR word boxes | Có | Có | Không | Thường chưa đủ | Nhận dạng chữ và tìm vị trí từ |
| Layout detection | Có thể chưa | Có | Có | Có thể cần bước riêng | Phân vùng trang |
| Structured document model | Có | Có | Có | Có, kèm hierarchy | ETL, RAG, chuyển đổi và kiểm toán |

---

### 2. DLA khác OCR như thế nào và nên chạy theo thứ tự nào?

**Trả lời:** **Optical Character Recognition (OCR)** trả lời câu hỏi “ký tự hoặc dòng chữ này là gì?”, còn DLA trả lời “vùng này là thành phần gì, nằm ở đâu, quan hệ với vùng nào và nên đọc theo thứ tự nào?”. Hai bước bổ trợ nhau nhưng không thay thế nhau.

Không có một thứ tự cố định cho mọi tài liệu. Ba kiểu pipeline phổ biến:

1. **Layout-first:** phát hiện vùng trước, sau đó OCR từng vùng. Phù hợp trang scan phức tạp vì tránh OCR lẫn giữa cột, bảng và hình.
2. **OCR-first:** OCR toàn trang trước, sau đó gom word/line box thành block và phân loại. Phù hợp biểu mẫu hoặc trang có text rõ, khi token và geometry hỗ trợ phân tích.
3. **Joint/end-to-end:** mô hình xử lý đồng thời ảnh, text và vị trí hoặc sinh trực tiếp cấu trúc. LayoutLMv3 là ví dụ mô hình Document AI kết hợp text, image patch và vị trí cho nhiều tác vụ, gồm DLA. [LayoutLMv3][layoutlmv3]

Với PDF số, nên ưu tiên lấy text và geometry gốc từ PDF trước. Chỉ OCR những trang hoặc vùng không có text tốt. Với PDF scan, thường render trang, sửa hướng/deskew, chạy layout và OCR theo vùng. Với PDF lai, nên route theo từng trang.

**Áp dụng khi nào?**

- Layout-first: báo, sách nhiều cột, manual nhiều figure, scan chất lượng không đồng đều.
- OCR-first: form cố định, hóa đơn đơn giản, tài liệu mà word box đã đáng tin cậy.
- Joint model: cần một backbone cho classification, extraction và layout; có GPU và dữ liệu phù hợp.

**Bài toán giải quyết:** chọn đúng ranh giới trách nhiệm giữa OCR và layout, tránh OCR toàn bộ tài liệu một cách tốn kém hoặc làm mất text số vốn đã chính xác.

**So sánh:**

| Pipeline | Điểm mạnh | Giới hạn |
| --- | --- | --- |
| OCR-only | Dễ triển khai, trả text nhanh | Mất loại vùng, hierarchy và thường sai reading order |
| Layout-first + OCR vùng | Tốt cho trang phức tạp, dễ crop | Nhiều bước, lỗi detector có thể làm mất text |
| OCR-first + grouping | Tận dụng word geometry | Phụ thuộc OCR; khó khi text dính hoặc hình phức tạp |
| Joint multimodal | Có thể tối ưu end-to-end | Nặng, khó debug và cần benchmark theo domain |

---

### 3. Cấu trúc vật lý, cấu trúc logic và thứ tự đọc khác nhau thế nào?

**Trả lời:**

- **Physical layout** mô tả vị trí trực quan: vùng chữ, đường kẻ, bảng, ảnh, khoảng trắng và cột.
- **Logical structure** mô tả vai trò ngữ nghĩa: tiêu đề cấp 1, section, đoạn văn, bước thao tác, cảnh báo, chú thích hay tài liệu tham khảo.
- **Reading order** là thứ tự mà nội dung nên được đọc hoặc tuần tự hóa.

Một bounding box lớn ở đầu trang có thể là `title`, nhưng việc biết nó thuộc section nào hoặc ảnh hưởng đến các đoạn phía sau là logical structure. Hai vùng nằm cạnh nhau không có nghĩa vùng bên trái luôn đọc trước: với sidebar, callout, chú thích hoặc layout hai cột, cần xét cột, hierarchy và quan hệ.

Nên biểu diễn reading order dưới dạng danh sách hoặc đồ thị có hướng:

```text
Title → Introduction → Column-1 blocks → Column-2 blocks
                         └──────────────→ Figure caption
```

Với tài liệu kỹ thuật, hierarchy còn cần giữ `sectionPath`, ví dụ `5. Bảo trì > 5.2 Thay vòng bi > Cảnh báo an toàn`.

**Áp dụng khi nào?** mọi trường hợp cần xuất Markdown/HTML, TTS/accessibility, tìm kiếm theo section, chunking hoặc QA dựa trên nội dung nhiều cột.

**Bài toán giải quyết:** tránh câu bị đảo, trộn hai cột, đặt caption xa hình, đưa header/footer vào nội dung chính hoặc làm mất mối quan hệ heading–paragraph.

**So sánh:**

| Mức phân tích | Câu hỏi trả lời | Ví dụ output |
| --- | --- | --- |
| Geometry | Ở đâu? | `bbox`, `polygon` |
| Physical class | Đây là vùng gì? | `table`, `figure`, `text` |
| Logical role | Vai trò là gì? | `section_heading`, `warning` |
| Reading order | Đọc theo thứ tự nào? | `order=1..n` hoặc graph |
| Document hierarchy | Thuộc phần nào? | `parentId`, `sectionPath` |

---

### 4. Có những nhóm phương pháp DLA nào và nên chọn nhóm nào?

**Trả lời:** Có bốn nhóm chính:

1. **Rule-based/heuristic:** dùng projection profile, connected components, whitespace, XY-cut, đường kẻ và quy tắc hình học.
2. **Object detection:** coi từng thành phần là object và dự đoán bounding box + class, như Faster R-CNN, YOLO, DETR.
3. **Semantic/instance segmentation:** gán nhãn pixel hoặc mask, hữu ích khi vùng không phải hình chữ nhật hoặc chồng lấn.
4. **Multimodal/VLM/end-to-end:** kết hợp image, text và spatial token hoặc sinh cấu trúc trực tiếp.

LayoutParser cung cấp API và model zoo để ghép layout detection với OCR và pipeline tài liệu. DocLayNet cung cấp dữ liệu đa dạng hơn PubLayNet và cho thấy model học từ corpus đa miền có khả năng tổng quát tốt hơn trên tài liệu doanh nghiệp. [LayoutParser][layoutparser] [DocLayNet][doclaynet]

**Áp dụng khi nào?**

- Rule-based: biểu mẫu cố định, template ổn định, cần chạy CPU nhanh.
- Object detection: taxonomy rõ, vùng chủ yếu hình chữ nhật, cần cân bằng tốc độ và độ chính xác.
- Segmentation: báo cũ, handwriting, vùng cong/chồng lấn hoặc cần đường biên chính xác.
- Multimodal/VLM: tài liệu biến thiên mạnh, cần hiểu đồng thời text–image–layout hoặc fallback cho trang khó.

**Bài toán giải quyết:** chọn mức phức tạp phù hợp thay vì mặc định dùng model lớn cho mọi trang.

**So sánh:**

| Nhóm | Dữ liệu huấn luyện | Tốc độ | Khả năng giải thích | Tổng quát hóa |
| --- | --- | --- | --- | --- |
| Rule-based | Ít hoặc không cần | Cao | Cao | Thấp khi layout thay đổi |
| Detector | Box + class | Cao đến trung bình | Khá | Tốt nếu dữ liệu đủ đa dạng |
| Segmentation | Mask/polygon | Trung bình đến thấp | Khá | Tốt với vùng phức tạp |
| Multimodal/VLM | Ảnh + text + nhãn/instruction | Thấp hơn | Khó hơn | Có tiềm năng cao nhưng cần kiểm chứng |

**Khuyến nghị production:** dùng router nhiều tầng. Parser/rule nhanh xử lý trang dễ; detector xử lý phần lớn trang; model mạnh hoặc human review chỉ nhận trang confidence thấp hoặc loại tài liệu giá trị cao.

---

### 5. Dữ liệu huấn luyện và chỉ số đánh giá DLA nên thiết kế thế nào?

**Trả lời:** Dataset cần phản ánh đúng domain production, không chỉ có nhiều trang. PubLayNet có quy mô lớn và chủ yếu lấy từ bài báo khoa học; DocLayNet có 80.863 trang được gán nhãn thủ công từ nguồn đa dạng hơn, với 11 lớp bố cục. Điều này minh họa rủi ro **domain shift**: model tốt trên paper khoa học có thể giảm chất lượng khi gặp manual, patent, invoice hoặc report doanh nghiệp. [PubLayNet][publaynet] [DocLayNet][doclaynet]

Schema annotation nên quy định rõ:

- taxonomy và định nghĩa từng class;
- box, polygon hay mask;
- quy tắc với vùng lồng nhau và chồng lấn;
- cách gán caption, footnote, header/footer;
- reading order và hierarchy;
- trang bị loại khỏi dataset;
- mức đồng thuận giữa annotator và quy trình adjudication.

Không nên chỉ đánh giá một metric:

| Mục tiêu | Chỉ số gợi ý |
| --- | --- |
| Phát hiện vùng | Precision, Recall, F1, AP/mAP theo IoU |
| Đúng class | Confusion matrix, macro-F1 |
| Biên vùng | IoU hoặc mask IoU |
| Reading order | Pairwise accuracy, Kendall/Spearman hoặc edit distance |
| Quan hệ | Precision/Recall cho figure–caption, parent–child |
| Chất lượng cuối | Markdown similarity, table accuracy, retrieval/QA quality |

**Áp dụng khi nào?** trước khi chọn model, fine-tune, đổi engine hoặc rollout phiên bản mới.

**Bài toán giải quyết:** tránh trường hợp mAP cao nhưng output vẫn không dùng được vì sai reading order, mất heading hoặc gắn caption nhầm hình.

**So sánh:**

- **Benchmark công khai:** tốt để sàng lọc model và tái lập nghiên cứu.
- **Golden set nội bộ:** phản ánh đúng tài liệu, ngôn ngữ và lỗi nghiệp vụ.
- **End-to-end test:** đo tác động thật tới Markdown, extraction hoặc RAG.

Nên dùng cả ba; benchmark công khai không thay thế golden set production.

---

## B. 8 câu hỏi phổ biến mở rộng

### 6. Nên xử lý PDF số, PDF scan và PDF lai khác nhau thế nào?

**Trả lời:** DLA nên bắt đầu bằng **preflight** và **page-level routing**.

| Loại trang | Tín hiệu | Luồng ưu tiên |
| --- | --- | --- |
| Digital PDF | Có text object, font và tọa độ tốt | Native parser → layout từ geometry/ảnh → align text |
| Scan | Gần như chỉ có ảnh | Render → orientation/deskew → DLA → OCR theo vùng |
| Hybrid | Có cả text và ảnh scan | Route từng trang/vùng, merge theo canonical schema |
| Born-digital nhưng glyph lỗi | Text tồn tại nhưng copy/search sai | So sánh native text với OCR, re-OCR có kiểm soát |

Không nên rasterize toàn bộ PDF số rồi OCR lại mặc định, vì sẽ mất ký tự chính xác, font, link, vector và tăng chi phí. Ngược lại, parser thuần thường không đủ cho scan hoặc reading order phức tạp.

**Áp dụng khi nào?** gần như mọi kho PDF doanh nghiệp, đặc biệt file ghép từ nhiều nguồn hoặc tài liệu có phụ lục scan.

**Bài toán giải quyết:** giảm GPU/CPU, tránh OCR chồng lên text tốt và cô lập trang khó để retry.

**So sánh:**

- File-level routing dễ triển khai nhưng quá thô.
- Page-level routing cân bằng tốt giữa chất lượng và độ phức tạp.
- Region-level routing tối ưu nhất cho form/sơ đồ nhưng cần orchestration phức tạp hơn.

---

### 7. Nên thiết kế taxonomy nhãn bố cục như thế nào?

**Trả lời:** Taxonomy phải xuất phát từ **downstream task**, không phải chỉ sao chép dataset công khai. Một taxonomy nền tảng có thể gồm:

```text
text, title, section_heading, list, table, figure, caption,
formula, header, footer, footnote, page_number
```

Tài liệu kỹ thuật có thể cần thêm:

```text
warning, caution, procedure_step, parts_list, diagram,
callout, specification, revision_block, signature, checkbox
```

Nguyên tắc thiết kế:

1. Hai class chỉ nên tách khi downstream xử lý khác nhau.
2. Định nghĩa phải đủ rõ để annotator đồng thuận.
3. Tránh class quá hiếm; có thể dùng hierarchy, ví dụ `text > warning`.
4. Giữ mapping giữa taxonomy nội bộ và taxonomy của model/dataset.
5. Version taxonomy vì đổi class sẽ ảnh hưởng model, schema và index.

PP-StructureV3 hỗ trợ pipeline bố cục với nhiều loại vùng, gồm title, text, table, figure, formula, header, footer, footnote và các thành phần khác; tuy nhiên taxonomy mặc định vẫn cần map về schema nghiệp vụ. [PP-StructureV3][ppstructure]

**Áp dụng khi nào?** xây dataset, fine-tune model, chuẩn hóa output từ nhiều engine hoặc thiết kế canonical document model.

**Bài toán giải quyết:** tránh model trả label không đủ chi tiết hoặc quá chi tiết nhưng không tạo giá trị.

**So sánh:**

- Taxonomy nhỏ: dễ gán nhãn, model ổn định, nhưng downstream phải suy luận thêm.
- Taxonomy lớn: giàu thông tin, nhưng class imbalance và nhầm lẫn tăng.
- Taxonomy phân cấp: linh hoạt nhất, nhưng cần schema và evaluator hỗ trợ hierarchy.

---

### 8. Làm thế nào xác định reading order cho tài liệu nhiều cột và sidebar?

**Trả lời:** Không nên chỉ sort theo `y` rồi `x`. Quy trình tốt hơn:

1. loại header/footer/page number khỏi luồng chính;
2. phát hiện column hoặc vùng lớn;
3. nhóm block theo column/section;
4. tạo cạnh ưu tiên dựa trên quan hệ trên–dưới, trái–phải và containment;
5. xử lý ngoại lệ như caption, sidebar, footnote và callout;
6. topological sort graph;
7. kiểm tra cycle và fallback bằng rule ổn định.

Ví dụ, caption nên nằm ngay sau figure liên quan dù tọa độ có thể khiến nó bị xếp sau block khác. Một sidebar có thể được gắn vào section gần nhất nhưng không chen giữa câu đang đọc.

Surya cung cấp layout analysis và reading-order detection như các tác vụ riêng, cho thấy reading order không nên được coi là hệ quả tự động của bounding box. [Surya][surya]

**Áp dụng khi nào?** báo nhiều cột, patent, manual có callout, tài liệu có footnote hoặc cần text tuần tự cho LLM/TTS.

**Bài toán giải quyết:** giảm câu đảo thứ tự, trộn cột và chunk sai ngữ cảnh.

**So sánh:**

| Cách | Ưu điểm | Nhược điểm |
| --- | --- | --- |
| Sort `y,x` | Rất nhanh | Sai nhiều cột/sidebar |
| Column heuristic | Tốt với layout quen thuộc | Khó với vùng lồng nhau |
| Graph rules | Kiểm soát và debug được | Nhiều edge case |
| Learned reading order | Tổng quát hơn | Cần dữ liệu thứ tự và evaluator riêng |

---

### 9. DLA có đủ để trích xuất bảng, hình, công thức và biểu mẫu không?

**Trả lời:** Không. DLA chủ yếu **định vị và phân loại vùng**. Sau đó cần parser chuyên biệt:

- `table` → nhận dạng hàng, cột, merged cell và cell text;
- `figure`/`diagram` → crop, caption linking, image classification hoặc VLM enrichment;
- `formula` → formula recognition/LaTeX OCR;
- `form` → key–value, checkbox, signature và field relation;
- `chart` → OCR trục/chú giải và chart-to-data nếu cần.

DLA giúp route đúng module và giới hạn vùng xử lý. PP-StructureV3 kết hợp layout detection, table recognition và formula recognition trong một pipeline; Docling cũng có các stage layout, OCR, table structure và document model. [PP-StructureV3][ppstructure] [Docling models][docling-models]

**Áp dụng khi nào?** tài liệu kỹ thuật, phiếu kiểm tra, catalogue, báo cáo tài chính hoặc tài liệu khoa học.

**Bài toán giải quyết:** tránh kỳ vọng detector có thể tự hiểu toàn bộ nội dung bên trong vùng.

**So sánh:**

- DLA-only: đủ để crop và phân tuyến.
- DLA + specialist models: dễ kiểm thử từng loại vùng, phù hợp production.
- End-to-end VLM: triển khai nhanh cho prototype, nhưng output khó ổn định và chi phí cao hơn.

---

### 10. Nên chọn LayoutParser, PaddleOCR, Docling, Unstructured hay Surya?

**Trả lời:** Đây không phải các sản phẩm hoàn toàn tương đương; chúng nằm ở các tầng khác nhau.

| Công cụ | Trọng tâm | Chọn khi | Lưu ý |
| --- | --- | --- | --- |
| LayoutParser | Toolkit/model wrapper cho document image analysis | Muốn ghép detector, OCR và custom pipeline nghiên cứu | Cần tự thiết kế nhiều bước production |
| PaddleOCR / PP-StructureV3 | OCR + layout + table + formula pipeline | Cần pipeline mã nguồn mở giàu module, nhiều cấu hình | Cần benchmark model/ngôn ngữ và vận hành Paddle stack |
| Docling | Chuyển đổi nhiều định dạng sang document model có cấu trúc | Cần PDF understanding, reading order, table và output chuẩn hóa | Dùng abstraction cao hơn detector thuần |
| Unstructured | Partition tài liệu thành elements cho ingestion | Cần nhanh chóng đưa nhiều định dạng vào ETL/RAG | Strategy `fast`, `hi_res`, `ocr_only` có trade-off khác nhau |
| Surya | OCR, layout, reading order và table recognition | Cần pipeline GPU gọn và hỗ trợ đa ngôn ngữ | Kiểm tra license, model size và benchmark trên corpus thật |

Unstructured mô tả `hi_res` là strategy dựa trên model để nhận dạng bố cục; Docling biểu diễn các item cùng layout/bounding box trong `DoclingDocument`; Surya tách layout và reading order thành khả năng rõ ràng. [Unstructured strategies][unstructured] [Docling document][docling-document] [Surya][surya]

**Áp dụng khi nào?** giai đoạn chọn stack, xây proof of concept hoặc thay engine hiện tại.

**Bài toán giải quyết:** tránh so sánh sai tầng, ví dụ so một detector library với một document conversion pipeline hoàn chỉnh.

**Khuyến nghị:** tạo cùng một golden set, chạy tất cả engine về cùng canonical schema, rồi so chất lượng, latency, VRAM/RAM, license, khả năng batch, retry và mức dễ debug.

---

### 11. Khi nào cần fine-tune hoặc domain adaptation?

**Trả lời:** Cần fine-tune khi model có lỗi lặp lại mang tính domain, ví dụ:

- nhầm `warning` với `text`;
- bỏ sót revision block trong bản vẽ kỹ thuật;
- nhận sai bảng không đường kẻ;
- gộp figure và caption;
- không nhận dạng layout tiếng Việt hoặc biểu mẫu nội bộ;
- model học từ scientific papers nhưng production là manual/patent/report.

Quy trình thực tế:

1. tạo golden set theo loại tài liệu, nguồn và độ khó;
2. chạy baseline, lập error taxonomy;
3. chọn các class/lỗi có ROI cao;
4. gán nhãn active-learning từ trang confidence thấp hoặc disagreement;
5. fine-tune nhỏ trước, giữ test set bất biến;
6. đánh giá detection, reading order và downstream;
7. canary rollout, lưu model/version để rollback.

DocLayNet được tạo nhằm tăng tính đa dạng bố cục so với các dataset lấy chủ yếu từ bài báo khoa học, minh họa rõ nhu cầu dữ liệu đúng domain. [DocLayNet][doclaynet]

**Áp dụng khi nào?** sau khi đã thử threshold, preprocessing, mapping taxonomy và rule post-processing nhưng lỗi cốt lõi vẫn còn.

**Bài toán giải quyết:** giảm domain shift và lỗi có tính hệ thống.

**So sánh:**

- Prompt/VLM adjustment: nhanh nhưng khó đảm bảo ổn định.
- Rule post-processing: rẻ cho lỗi hình học rõ ràng, nhưng dễ tích lũy ngoại lệ.
- Fine-tune detector: ổn định cho taxonomy cụ thể, cần dữ liệu.
- Train from scratch: chỉ hợp lý khi domain rất khác và có dataset lớn.

---

### 12. Thiết kế DLA thế nào khi xử lý từ 50.000 đến 1.000.000 PDF?

**Trả lời:** Đơn vị lập lịch nên là **page work item**, không chỉ là file. Kiến trúc nên tách:

```text
Inventory → Page Router → Render/Parse Workers → Layout Workers
          → OCR/Specialist Workers → Normalize → Quality Gate → Publish
```

Các nguyên tắc vận hành:

- lưu PDF gốc bất biến và checksum;
- idempotency theo `documentVersion + page + pipelineVersion`;
- queue riêng cho fast parser, OCR, layout, table và VLM;
- batch các trang có kích thước tương tự để tối ưu GPU;
- cache page image và native text geometry;
- giới hạn DPI/kích thước ảnh, chống decompression bomb;
- timeout, retry có phân loại và dead-letter queue;
- autoscale theo backlog, page complexity và VRAM;
- lưu confidence, model version, thời gian và lỗi theo từng page;
- sampling quality và reprocess có version.

Docling cho phép cấu hình giới hạn kích thước file và số trang; Unstructured khuyến nghị thử strategy nhanh trước và có cơ chế chia PDF theo trang/batch, phù hợp tư duy router theo độ khó. [Docling advanced][docling-advanced] [Unstructured scale][unstructured-scale]

**Áp dụng khi nào?** kho tài liệu trung bình đến rất lớn, ingestion định kỳ hoặc cần re-index khi model thay đổi.

**Bài toán giải quyết:** kiểm soát chi phí, retry chính xác, tránh một file lỗi làm hỏng job lớn và hỗ trợ rollout model an toàn.

**So sánh:**

- Monolithic file job: đơn giản nhưng retry tốn kém và khó scale.
- Page queue: linh hoạt, dễ autoscale, cần merge kết quả.
- Region queue: tối ưu cho specialist model nhưng tăng số work item và metadata.

---

### 13. DLA ảnh hưởng thế nào đến RAG và lỗi nào thường gặp khi tích hợp?

**Trả lời:** DLA quyết định chất lượng **document structure** trước khi chunking và retrieval. Một pipeline tích hợp tốt nên:

1. loại header/footer lặp lại;
2. dựng hierarchy heading–section;
3. nối paragraph theo reading order;
4. giữ table/figure/caption thành object riêng;
5. chunk theo section và giới hạn token;
6. lưu `page`, `bbox`, `regionId`, `sectionPath`, `sourceUri`;
7. index text và metadata, đồng thời giữ liên kết tới hình/bảng;
8. render citation highlight đúng vùng khi trả lời.

Các lỗi phổ biến:

- chunk trộn hai cột;
- heading đứng cuối chunk trước thay vì đầu chunk sau;
- caption tách khỏi figure;
- header/footer lặp lại làm nhiễu retrieval;
- bảng bị flatten thành chuỗi khó hiểu;
- OCR word box không cùng hệ tọa độ với ảnh/PDF;
- resize ảnh nhưng không scale bbox;
- mất model/pipeline version nên không tái lập được kết quả.

**Áp dụng khi nào?** xây RAG cho manual thiết bị, quy trình vận hành, sửa chữa, biên bản kiểm tra hoặc cấp/xuất vật tư.

**Bài toán giải quyết:** tăng khả năng truy hồi đúng section, giữ citation và giảm câu trả lời sai do dữ liệu đầu vào bị đảo hoặc mất cấu trúc.

**So sánh:**

| Cách chunk | Ưu điểm | Rủi ro |
| --- | --- | --- |
| Fixed characters/tokens | Nhanh, dễ | Cắt giữa bảng, heading và câu |
| Paragraph-based | Tốt hơn plain text | Phụ thuộc reading order |
| Layout/section-aware | Giữ ngữ nghĩa và citation | Cần DLA + hierarchy đáng tin cậy |
| Multimodal chunk | Giữ cả text, image, table | Storage/index và retrieval phức tạp hơn |

**Nguyên tắc:** DLA không trực tiếp bảo đảm RAG tốt; cần đo end-to-end bằng retrieval recall, citation accuracy và answer grounding trên golden questions.

---

## Checklist áp dụng nhanh

1. Phân loại corpus theo digital/scan/hybrid và loại tài liệu.
2. Định nghĩa taxonomy dựa trên downstream task.
3. Chọn canonical schema có bbox, class, reading order, hierarchy và provenance.
4. Tạo golden set chứa trang dễ, khó và edge case.
5. Benchmark ít nhất một fast path và một layout-aware path.
6. Đánh giá detection, reading order và downstream output.
7. Dùng page-level router, confidence threshold và fallback.
8. Version model, taxonomy, preprocessing và schema.
9. Theo dõi lỗi theo class, nguồn tài liệu và engine.
10. Chỉ fine-tune sau khi xác định lỗi domain có ROI rõ ràng.

## Tài liệu tham khảo

- [Document layout analysis: A comprehensive survey — ACM Computing Surveys][dla-survey]
- [DocLayNet: A Large Human-Annotated Dataset for Document-Layout Segmentation — IBM Research][doclaynet]
- [PubLayNet: largest dataset ever for document layout analysis][publaynet]
- [LayoutParser: A Unified Toolkit for Deep Learning Based Document Image Analysis][layoutparser]
- [LayoutLMv3: Pre-training for Document AI with Unified Text and Image Masking][layoutlmv3]
- [PP-StructureV3 — PaddleOCR documentation][ppstructure]
- [Docling model catalog][docling-models]
- [Docling document model][docling-document]
- [Unstructured partitioning strategies][unstructured]
- [Surya OCR and layout analysis][surya]

[dla-survey]: https://doi.org/10.1145/3355610
[doclaynet]: https://research.ibm.com/publications/doclaynet-a-large-human-annotated-dataset-for-document-layout-segmentation
[publaynet]: https://arxiv.org/abs/1908.07836
[layoutparser]: https://arxiv.org/abs/2103.15348
[layoutlmv3]: https://www.microsoft.com/en-us/research/publication/layoutlmv3-pre-training-for-document-ai-with-unified-text-and-image-masking/
[ppstructure]: https://www.paddleocr.ai/main/en/version3.x/pipeline_usage/PP-StructureV3.html
[docling-models]: https://docling-project.github.io/docling/usage/model_catalog/
[docling-document]: https://docling-project.github.io/docling/concepts/docling_document/
[docling-advanced]: https://docling-project.github.io/docling/usage/advanced_options/
[unstructured]: https://docs.unstructured.io/open-source/concepts/partitioning-strategies
[unstructured-scale]: https://docs.unstructured.io/open-source/how-to/speed-up-large-files-batches
[surya]: https://github.com/datalab-to/surya
