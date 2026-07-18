# Minimal Reproducible Example (MRE) cho AI và OCR tài liệu kỹ thuật

> [Knowledge Tree](../../../README.md) / [AI](../../README.md) / [LLM](../README.md) / **Minimal Reproducible Example (MRE)**

**Minimal Reproducible Example (MRE)** là một bộ dữ liệu và hướng dẫn thử nghiệm **nhỏ nhất nhưng vẫn đủ để người khác tái hiện đúng lỗi hoặc hành vi cần đánh giá**. Trong pipeline xử lý tài liệu bằng AI, MRE không chỉ là một đoạn prompt hoặc một ảnh chụp lỗi. Nó thường phải khóa đồng thời file đầu vào, trang hoặc vùng gây lỗi, loại PDF, công cụ trích xuất, cấu hình OCR, prompt, model, tham số sinh, schema đầu ra, kết quả mong đợi và kết quả thực tế.

MRE đặc biệt hữu ích với tài liệu hướng dẫn kỹ thuật cho máy móc vì một trang có thể chứa đồng thời chữ in, mã linh kiện, cảnh báo an toàn, bảng thông số, ảnh chụp, bản vẽ vector, chú thích hình, nhiều cột và nhiều ngôn ngữ. Nếu chỉ mô tả “AI đọc sai PDF”, nhóm phát triển khó xác định lỗi nằm ở PDF parser, OCR, layout analysis, prompt, model thị giác, hậu xử lý hay dữ liệu chuẩn.

> **Tài liệu đầy đủ gây lỗi → chọn trang/vùng nhỏ nhất → khóa môi trường và cấu hình → định nghĩa expected/actual → cung cấp một lệnh chạy → tái hiện và phân loại lỗi**

## Điều hướng

- **Node cha:** [Mô hình ngôn ngữ lớn (LLM)](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** [Chain of Thought (CoT)](../cot/README.md)
- **Khái niệm sau trong cùng nhánh:** Chưa có.

## A. 5 câu hỏi cốt lõi

### 1. Minimal Reproducible Example là gì và giải quyết bài toán nào trong AI/OCR?

**Trả lời:** Một MRE tốt phải thỏa ba thuộc tính:

1. **Minimal — tối thiểu:** chỉ giữ lại dữ liệu, bước xử lý và cấu hình cần thiết để lỗi còn xuất hiện.
2. **Complete — đầy đủ:** người nhận có đủ file, dependency, tham số, prompt và hướng dẫn để chạy mà không phải đoán phần còn thiếu.
3. **Reproducible — tái hiện được:** người tạo đã chạy lại từ đầu và xác nhận bộ ví dụ thực sự tạo ra lỗi hoặc hành vi cần phân tích.

Trong AI/OCR, MRE giúp biến mô tả mơ hồ như:

> “Model đọc sai tài liệu máy nén khí 300 trang, mất hình và đề mục.”

thành một ca có thể kiểm chứng:

> “Trang 47–48 của `manual-mre.pdf`, SHA-256 cụ thể, là PDF scan 300 dpi. Pipeline A nhận sai `4.2 Lubrication` thành nội dung thường, bỏ hình `Fig. 4-3`, trong khi `expected.json` yêu cầu heading level 2 và figure liên kết với section `4.2`. Chạy bằng một lệnh trong Docker tạo `actual.json`.”

**Áp dụng khi nào?**

- Lỗi chỉ xảy ra với một số trang, model hoặc phiên bản thư viện.
- OCR lúc đúng lúc sai hoặc khác nhau giữa máy local và server.
- Trích xuất mất hình, sai thứ tự đọc, sai cấp đề mục hoặc gắn nhầm chú thích.
- Cần gửi bug cho nhóm AI, OCR vendor, cloud provider hoặc thư viện mã nguồn mở.
- Cần biến một lỗi production thành test chống tái diễn.

**Bài toán giải quyết:** rút ngắn thời gian khoanh vùng nguyên nhân, tránh gửi cả kho tài liệu mật, tạo bằng chứng so sánh giữa các giải pháp và cung cấp test case ổn định cho regression testing.

**So sánh:**

| Cách báo lỗi | Điểm mạnh | Hạn chế | Khi dùng |
| --- | --- | --- | --- |
| Mô tả bằng lời | Nhanh. | Thiếu dữ liệu, khó tái hiện, dễ hiểu sai. | Chỉ để thông báo ban đầu. |
| Ảnh chụp màn hình | Cho thấy biểu hiện. | Không có input gốc, cấu hình và output máy đọc được. | Bổ sung cho MRE, không thay thế MRE. |
| Gửi toàn bộ tài liệu và hệ thống | Có nhiều ngữ cảnh. | Nặng, có thể lộ dữ liệu, khó xác định biến gây lỗi. | Điều tra nội bộ sau khi MRE chưa đủ. |
| **MRE** | Nhỏ, chạy được, so sánh được, dễ chia sẻ. | Cần công sức rút gọn và xác minh. | Mặc định cho debugging và regression. |

Nguồn nền tảng: [Stack Overflow — How to create a Minimal, Reproducible Example](https://stackoverflow.com/help/minimal-reproducible-example).

---

### 2. Một MRE tối thiểu cho prompt AI và OCR PDF cần những thành phần nào?

**Trả lời:** Bộ MRE nên chứa đủ **input, execution context, expected result và observed result**. Cấu trúc thực tế có thể như sau:

```text
mre-heading-figure-extraction/
├── README.md
├── input/
│   ├── manual-pages-047-048.pdf
│   ├── page-047-reference.png
│   └── sha256.txt
├── config/
│   ├── prompt.txt
│   ├── request.json
│   ├── schema.json
│   └── environment.md
├── expected/
│   └── expected.json
├── actual/
│   ├── actual.json
│   └── raw-response.json
├── src/
│   └── run.py
├── requirements.lock
├── Dockerfile
└── run.sh
```

#### Thành phần bắt buộc

| Nhóm | Cần ghi gì? | Vì sao cần? |
| --- | --- | --- |
| Input | File hoặc 1–3 trang nhỏ nhất còn gây lỗi; số trang; hash. | Chứng minh mọi người chạy cùng dữ liệu. |
| Loại PDF | Digital PDF, scanned PDF hay mixed/hybrid PDF. | Quyết định có thể trích text trực tiếp hay phải OCR. |
| Tiền xử lý | Render DPI, deskew, crop, denoise, rotation, color mode. | Đây thường là biến làm OCR thay đổi mạnh. |
| OCR/parser | Tên tool, phiên bản, ngôn ngữ, page segmentation, layout feature. | Tái hiện đúng text và bounding box đầu vào cho LLM. |
| Prompt | Toàn bộ system/developer/user instruction có liên quan. | Một phần prompt bị thiếu có thể đổi hành vi. |
| Model | Provider, model ID hoặc snapshot, API/SDK version. | Hành vi có thể khác giữa model hoặc snapshot. |
| Generation config | Temperature, top-p, max tokens, seed nếu có, timeout, retry. | Phân biệt lỗi cấu hình với lỗi prompt/model. |
| Output contract | JSON Schema, quy ước tọa độ, heading level, figure linkage. | Tránh tranh luận bằng cảm giác về “đúng”. |
| Expected | Kết quả chuẩn do người hiểu tài liệu xác nhận. | Là oracle/gold result để đánh giá. |
| Actual | Raw response và output sau hậu xử lý. | Cho biết lỗi xuất hiện ở model hay parser sau model. |
| Run instructions | Một lệnh chạy từ môi trường sạch. | Kiểm chứng tính reproducible. |

**Áp dụng khi nào?** Mọi lỗi cần người khác điều tra, đặc biệt khi pipeline gồm nhiều lớp: PDF parser → OCR → layout → prompt/model → JSON parser → business rules.

**Bài toán giải quyết:** tránh tình trạng chỉ gửi prompt nhưng thiếu OCR text, chỉ gửi JSON sau xử lý nhưng thiếu raw response, hoặc chỉ gửi PDF nhưng thiếu model và tham số.

**So sánh MRE prompt-only và end-to-end:**

| Loại MRE | Thành phần | Dùng để kiểm tra |
| --- | --- | --- |
| Prompt-only | Prompt + text/image input chuẩn hóa + model config. | Lỗi instruction, schema, suy luận hoặc model. |
| OCR-only | PDF/image + preprocessing + OCR config + expected text/layout. | Lỗi nhận dạng ký tự, tọa độ, dòng, block. |
| End-to-end | PDF + OCR/parser + prompt/model + post-processing. | Lỗi tích hợp và truyền sai dữ liệu giữa các bước. |

---

### 3. Làm thế nào rút một PDF hàng trăm trang thành MRE mà không làm mất lỗi?

**Trả lời:** Không nên bắt đầu bằng cách “convert toàn bộ PDF sang ảnh”. Trước tiên phải xác định bản chất của từng trang.

#### Bước 1 — Phân loại trang

- **Digital/native PDF:** có text layer, font, vector và ảnh nhúng; ưu tiên trích xuất trực tiếp.
- **Scanned PDF:** mỗi trang chủ yếu là ảnh; cần render/OCR.
- **Mixed PDF:** một số phần là text thật, một số phần là ảnh scan hoặc text nằm trong hình.
- **Born-digital nhưng text giả:** chữ được vẽ thành vector nhỏ hoặc encoding font bất thường; nhìn thấy chữ nhưng parser không lấy được text đúng.

PyMuPDF khuyến nghị chỉ OCR khi cần, ví dụ trang bị phủ hoàn toàn bởi ảnh, không có text hoặc có rất nhiều vector nhỏ mô phỏng chữ. OCR nên là fallback vì chậm hơn và làm mất thông tin font gốc như bold/italic. [PyMuPDF — OCR](https://pymupdf.readthedocs.io/en/latest/recipes-ocr.html).

#### Bước 2 — Tìm đơn vị nhỏ nhất còn gây lỗi

1. Xác định trang đầu tiên có lỗi.
2. Thử chỉ giữ trang đó.
3. Nếu lỗi phụ thuộc tiêu đề cha, bảng chú giải hoặc caption ở trang trước, thêm đúng trang phụ thuộc.
4. Nếu lỗi chỉ nằm trong một vùng, vẫn nên giữ **PDF trang gốc** và cung cấp thêm ảnh crop để chỉ vị trí; không chỉ gửi ảnh crop.
5. Kiểm tra lại từ môi trường sạch để chắc lỗi vẫn xuất hiện.

#### Bước 3 — Bảo toàn hai biểu diễn

Nên giữ đồng thời:

- `manual-pages-047-048.pdf`: snippet PDF gần với cấu trúc gốc.
- `page-047-reference.png`: ảnh render để con người nhìn cùng một bố cục.

Lý do là “hình nhìn thấy trên trang” không nhất thiết là một ảnh nhúng duy nhất. Một sơ đồ máy có thể được ghép từ vector, text và nhiều raster image. PyMuPDF hỗ trợ cả trích ảnh nhúng theo `xref` và lấy image block kèm vị trí qua `Page.get_text("dict")`; hai cách phục vụ hai mục tiêu khác nhau. [PyMuPDF — Images](https://pymupdf.readthedocs.io/en/latest/recipes-images.html).

#### Bước 4 — Khóa tính toàn vẹn

```text
file: manual-pages-047-048.pdf
sha256: <hash>
source_page_numbers: 47-48
page_numbering_rule: physical pages, 1-based
crop: none
redaction: part serial number replaced; geometry preserved
```

**Áp dụng khi nào?** Khi tài liệu lớn, bí mật, có nhiều kiểu trang hoặc lỗi chỉ xảy ra ở một vùng nhỏ.

**Bài toán giải quyết:** giảm kích thước và dữ liệu nhạy cảm nhưng vẫn bảo toàn encoding, layout, vector, font, image mask và các yếu tố gây lỗi.

**So sánh các cách rút gọn:**

| Cách | Có giữ cấu trúc PDF? | Phù hợp |
| --- | --- | --- |
| Tách đúng trang từ PDF | Có, phần lớn. | MRE mặc định. |
| Chụp màn hình trang | Không. | Chỉ làm ảnh minh họa. |
| Print-to-PDF | Có thể thay font, flatten, rasterize. | Chỉ dùng nếu chính print pipeline là đối tượng test. |
| Render thành PNG | Mất text/vector gốc nhưng cố định ảnh đầu vào OCR. | MRE cho OCR image-level. |
| Tạo PDF synthetic | Kiểm soát tốt, không lộ dữ liệu. | Khi tái tạo được cùng lỗi và ghi rõ khác biệt. |

---

### 4. MRE cho AI prompt phải khóa những gì để kết quả có thể so sánh?

**Trả lời:** Prompt AI không chỉ là đoạn user message. Một MRE phải ghi lại toàn bộ **effective request** mà model thực sự nhận.

#### Các biến cần khóa

```yaml
provider: example-provider
model: exact-model-or-snapshot
api_version: exact-version
sdk: package==version
messages:
  - role: system
    content_file: prompt-system.txt
  - role: user
    content_file: prompt-user.txt
input_assets:
  - input/manual-pages-047-048.pdf
generation:
  temperature: 0
  top_p: 1
  max_output_tokens: 4000
response_format: config/schema.json
retry_policy: disabled-for-mre
postprocess_version: git-commit-sha
```

OpenAI lưu ý hành vi prompting có thể thay đổi giữa các snapshot và output model vốn có tính biến thiên; cách tốt để giữ hành vi nhất quán hơn là dùng phiên bản model được pin và triển khai eval. Nguyên tắc này cũng hữu ích khi làm việc với provider khác: ghi model ID chính xác, ngày chạy và bộ đánh giá. [OpenAI API — Backward compatibility](https://platform.openai.com/docs/api-reference/backward-compatibility).

#### Cách tách biến để tìm nguyên nhân

1. **OCR baseline:** dùng chính OCR/layout output đã lưu, bỏ qua bước đọc PDF.
2. **Prompt baseline:** gửi input cố định vào model, không qua agent/tool/retry.
3. **Post-processing baseline:** chạy parser trên raw response đã lưu.
4. **End-to-end:** chỉ chạy sau khi từng lớp riêng lẻ đã ổn định.

#### Không nên làm

- Chỉ copy nội dung prompt nhìn thấy trên UI nhưng bỏ system instruction.
- Dùng alias model tự động trỏ sang phiên bản mới mà không ghi ngày chạy.
- Bật retry rồi chỉ lưu response cuối; lần lỗi có thể đã bị che.
- Thay PDF bằng OCR text nhưng vẫn kết luận model đọc PDF sai.
- Chỉ ghi “temperature thấp” thay vì giá trị cụ thể.

**Áp dụng khi nào?** Khi cùng prompt cho kết quả khác nhau, khi nâng model/SDK, khi chuyển provider hoặc khi output JSON đôi lúc không hợp lệ.

**Bài toán giải quyết:** phân biệt bốn nhóm lỗi: input conversion, OCR/layout, model/prompt và parser/business logic.

**So sánh:**

| Cách kiểm tra | Ưu điểm | Hạn chế |
| --- | --- | --- |
| Chạy một lần | Nhanh. | Không đo được độ dao động. |
| Chạy lặp cùng cấu hình | Phát hiện lỗi ngẫu nhiên và tỷ lệ lỗi. | Tốn chi phí hơn. |
| Pin model + regression set | So sánh release có kiểm soát. | Cần quản lý phiên bản và gold data. |
| Chỉ xem demo thủ công | Dễ bắt đầu. | Dễ cherry-pick case đẹp, không có bằng chứng định lượng. |

---

### 5. Định nghĩa expected output thế nào cho bài toán trích xuất đề mục và hình ảnh?

**Trả lời:** Expected output phải là hợp đồng có thể chấm được, không phải câu “trích đúng nội dung”. Với tài liệu kỹ thuật, nên tách **nội dung, cấu trúc, không gian và liên kết ngữ nghĩa**.

Ví dụ schema khái niệm:

```json
{
  "document_id": "manual-mre-001",
  "pages": [
    {
      "page": 47,
      "width": 2480,
      "height": 3508,
      "coordinate_system": "pixel-top-left",
      "sections": [
        {
          "id": "sec-4-2",
          "title": "4.2 Lubrication",
          "level": 2,
          "bbox": [214, 302, 1398, 388],
          "source": "native_text",
          "confidence": 1.0
        }
      ],
      "figures": [
        {
          "id": "fig-4-3",
          "bbox": [320, 720, 2110, 2320],
          "caption": "Fig. 4-3 Lubrication points",
          "caption_bbox": [540, 2350, 1940, 2425],
          "linked_section_id": "sec-4-2",
          "asset_path": "assets/fig-4-3.png"
        }
      ]
    }
  ]
}
```

#### Phải thống nhất trước khi chấm

- Trang dùng số vật lý hay số in trong tài liệu; bắt đầu từ 0 hay 1.
- Tọa độ theo pixel, point hay normalized 0–1; gốc tọa độ ở đâu.
- `bbox` dùng `[x0,y0,x1,y1]` hay polygon.
- Đề mục nhiều dòng được nối thế nào.
- Running header/footer có được coi là heading không.
- Caption có thuộc figure hay là paragraph độc lập.
- Một hình ghép từ nhiều object được xuất thành một asset render hay nhiều ảnh nhúng.
- Figure liên kết với section gần nhất theo hình học hay theo nội dung.

#### Metric nên dùng

| Thành phần | Metric gợi ý |
| --- | --- |
| Text OCR | Character Error Rate (CER), Word Error Rate (WER), exact match cho part number. |
| Heading detection | Precision, Recall, F1. |
| Heading hierarchy | Level accuracy, parent-child accuracy, tree edit distance. |
| Reading order | Pairwise order accuracy hoặc sequence accuracy. |
| Bounding box | IoU, center-distance, containment. |
| Figure detection | Figure precision/recall, asset coverage. |
| Caption linking | Link accuracy, section association accuracy. |
| JSON contract | Schema-valid rate, required-field completeness. |

Google Document AI Layout Parser có thể phân tích heading, header, footer, table structure và figure; Azure Document Intelligence biểu diễn cả section/subsection cùng các phần tử paragraph, table và figure. Đây là ví dụ cho thấy output tài liệu nên giữ cấu trúc thay vì chỉ trả về chuỗi text phẳng. [Google Cloud Document AI — Layout Parser](https://docs.cloud.google.com/document-ai/docs/layout-parse-chunk) · [Microsoft Document Intelligence — Analyze response](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/analyze-document-response?view=doc-intel-4.0.0).

**Áp dụng khi nào?** Khi xây parser tài liệu, hệ thống tìm kiếm/RAG, kiểm tra bảo trì, trích hướng dẫn vận hành hoặc chuyển manual thành dữ liệu có cấu trúc.

**Bài toán giải quyết:** tạo ground truth rõ ràng, chấm đúng từng lỗi và tránh tình trạng text đúng nhưng sai thứ tự, mất hình hoặc gắn sai section.

---

## B. 8 câu hỏi phổ biến

### 6. Nên dùng text extraction, OCR, layout model hay Vision-Language Model trong MRE?

**Trả lời:** Các phương án không thay thế hoàn toàn nhau. MRE nên tách chúng thành baseline để biết lớp nào tạo lỗi.

| Phương án | Đầu vào phù hợp | Lấy được | Điểm mạnh | Giới hạn |
| --- | --- | --- | --- | --- |
| Native PDF extraction | PDF sinh từ phần mềm, có text layer. | Text span, font, bbox, ảnh nhúng, object. | Nhanh, giữ ký tự gốc tốt. | Reading order có thể sai; không đọc text trong ảnh. |
| OCR | Scan/image hoặc vùng không có text. | Ký tự và vị trí ước lượng. | Dùng được với tài liệu giấy. | Sai ký tự, part number; mất style; phụ thuộc chất lượng ảnh. |
| Layout/document model | Tài liệu có heading, bảng, figure, form. | Block, role, table, section, figure. | Hiểu cấu trúc tốt hơn OCR thuần. | Chi phí cao hơn; output/provider khác nhau. |
| Vision-Language Model (VLM) | Trang hoặc crop cần hiểu cả hình và chữ. | Mô tả, quan hệ ngữ nghĩa, structured answer. | Linh hoạt với hình phức tạp. | Có thể hallucinate; khó bảo toàn toàn bộ tọa độ và ký tự nhỏ. |
| Hybrid pipeline | Manual phức tạp. | Kết hợp object, OCR, layout và semantic extraction. | Chất lượng và khả năng kiểm chứng tốt hơn. | Nhiều thành phần nên MRE phải chi tiết hơn. |

Amazon Textract, chẳng hạn, biểu diễn layout bằng nhiều block type như title, section header, figure, table, key-value và list. Điều này khác OCR thuần chỉ trả line/word. [Amazon Textract — Block](https://docs.aws.amazon.com/textract/latest/APIReference/API_Block.html).

**Khuyến nghị MRE:** lưu output của từng tầng:

```text
01-native-extraction.json
02-ocr.json
03-layout.json
04-model-raw.json
05-normalized-result.json
```

**Áp dụng khi nào?** Khi chưa rõ lỗi là “không đọc được ký tự”, “không hiểu đây là heading”, “không gom đúng figure” hay “model suy diễn sai”.

**Bài toán giải quyết:** tránh thay model LLM liên tục trong khi nguyên nhân thật nằm ở ảnh render hoặc OCR.

---

### 7. Tạo MRE cho tài liệu máy móc thuộc nhiều ngành nghề thế nào?

**Trả lời:** Không nên dùng một trang “đại diện cho mọi tài liệu kỹ thuật”. Mỗi ngành có các failure mode khác nhau. Hãy tạo **một MRE cho một kiểu lỗi**, sau đó gom thành test suite nhiều lát cắt.

| Lát cắt tài liệu | Ví dụ lỗi cần MRE |
| --- | --- |
| Cơ khí | Bản vẽ exploded view bị tách thành nhiều ảnh; số callout không gắn đúng part list. |
| Điện/điện tử | Sơ đồ mạch là vector; OCR bỏ ký hiệu đầu cực, điện áp hoặc dấu cực tính. |
| Tự động hóa/PLC | Code block, ladder diagram và bảng I/O bị đảo thứ tự. |
| Ô tô/xe máy | Torque value, unit và model applicability bị gắn sai bước thao tác. |
| Thiết bị y tế | Cảnh báo, chống chỉ định và quy trình vệ sinh bị hạ thành text thường. |
| Xây dựng/nông nghiệp | Ảnh hiện trường nền nhiễu; nhãn trên hình nhỏ; tài liệu song ngữ. |
| Dầu khí/hóa chất | P&ID, hazard symbol, tag thiết bị và bảng điều kiện vận hành. |
| Dệt may/thực phẩm | Quy trình nhiều bước, thông số nhiệt/áp/thời gian và bảng troubleshooting. |

IEC/IEEE 82079-1:2019 đưa ra nguyên tắc chung cho việc chuẩn bị thông tin hướng dẫn sử dụng của nhiều loại sản phẩm, từ sản phẩm đơn giản đến máy công nghiệp và hệ thống phức tạp. Khi xây bộ MRE cho manual, có thể dùng cấu trúc hướng dẫn, cảnh báo, bước thao tác và thông tin đánh giá làm các lát cắt kiểm thử; tuy nhiên cần đối chiếu thêm tiêu chuẩn chuyên ngành áp dụng cho sản phẩm cụ thể. [IEC — IEC/IEEE 82079-1:2019](https://webstore.iec.ch/en/publication/29075).

**Áp dụng khi nào?** Khi sản phẩm nhận nhiều manual từ vendor, nhiều dòng máy hoặc nhiều ngôn ngữ.

**Bài toán giải quyết:** tránh tối ưu cho manual chữ đơn giản nhưng thất bại với sơ đồ, cảnh báo và bảng thông số có giá trị vận hành cao.

**So sánh:**

- **Một MRE tổng hợp lớn:** gần production nhưng khó biết feature nào gây lỗi.
- **Nhiều MRE đơn mục tiêu:** dễ debug và regression; cần thêm một bộ end-to-end để kiểm tra tương tác giữa các feature.

---

### 8. Làm sao ẩn dữ liệu mật mà vẫn giữ lỗi tái hiện được?

**Trả lời:** Sanitization phải giữ lại thuộc tính tạo ra lỗi. Xóa nội dung quá mạnh có thể làm MRE không còn hợp lệ.

#### Thứ tự ưu tiên

1. **Tách trang nhỏ nhất** trước khi chỉnh sửa.
2. Thay tên khách hàng, serial number hoặc địa chỉ bằng chuỗi có độ dài và kiểu ký tự tương tự.
3. Giữ font, cỡ chữ, rotation, màu, bbox, số cột và quan hệ giữa caption–figure.
4. Với part number, tạo mã giả vẫn giữ pattern như `AB-1234-X9`.
5. Với hình máy độc quyền, crop đúng vùng lỗi hoặc tạo hình synthetic có cùng loại line-art/text overlay.
6. Ghi toàn bộ phép biến đổi trong `README.md`.
7. Chạy lại để xác nhận lỗi vẫn xuất hiện sau sanitization.

```text
Sanitization log
- Company name replaced with 12-character placeholder.
- Serial number pattern preserved: AA-9999-A9.
- Figure geometry unchanged.
- Page 48 removed because unrelated.
- Reproduction result after redaction: 5/5 runs fail in the same way.
```

**Không nên:** dùng blur cho vùng chữ đang gây OCR lỗi; rasterize lại toàn trang nếu lỗi liên quan font/encoding; thay bảng thật bằng đoạn text đơn giản rồi cho rằng vẫn là cùng case.

**Áp dụng khi nào?** Khi gửi issue ra ngoài tổ chức, dùng tài liệu của khách hàng hoặc manual có sở hữu trí tuệ.

**Bài toán giải quyết:** cân bằng giữa bảo mật và khả năng điều tra kỹ thuật.

**So sánh:**

| Cách | Bảo mật | Giữ lỗi |
| --- | --- | --- |
| Chỉ mô tả bằng lời | Cao | Thấp. |
| Che đen/blur tùy ý | Trung bình | Có thể phá OCR/layout. |
| Thay nội dung nhưng giữ hình học/pattern | Cao | Tốt nếu kiểm tra lại. |
| Synthetic document | Rất cao | Tốt khi tái tạo đúng failure mode. |

---

### 9. Kiểm thử “trích xuất hình ảnh” thế nào khi một figure không phải ảnh nhúng?

**Trả lời:** Cần phân biệt ít nhất ba mục tiêu:

1. **Embedded image extraction:** lấy binary image object nguyên bản từ PDF.
2. **Rendered figure extraction:** crop vùng figure như người dùng nhìn thấy, gồm raster, vector, text label và annotation.
3. **Semantic figure detection:** xác định vùng nào là figure, caption nào thuộc figure và figure thuộc section nào.

Một bản vẽ kỹ thuật có thể không xuất hiện trong danh sách ảnh nhúng vì nó được vẽ bằng vector. Ngược lại, logo nền hoặc mask cũng có thể xuất hiện như image object nhưng không phải figure nội dung. Vì vậy test “số ảnh extract được” không đủ.

#### Expected output nên chứa

- `figure_id`, page và bbox/polygon.
- Loại figure: photo, line drawing, schematic, chart, exploded view, icon group.
- Asset mong đợi: original embedded image hay rendered crop.
- Caption text và caption bbox.
- Section liên kết.
- Danh sách callout/label quan trọng nếu nghiệp vụ cần.
- Quy tắc loại logo, watermark, icon lặp và background.

#### Metric

- Figure region precision/recall.
- IoU bbox.
- Caption-link accuracy.
- Perceptual similarity của rendered asset.
- Coverage của callout/label quan trọng.
- Duplicate rate giữa các trang.

PyMuPDF chỉ ra rằng ảnh PDF được nhận diện bằng `xref`, có thể xuất hiện nhiều lần, có soft mask và có thể cần logic loại pseudo-image; `Page.get_text("dict")` lại cung cấp image block kèm bbox và bytes. MRE nên ghi rõ API và khái niệm “image” đang kiểm thử. [PyMuPDF — Images](https://pymupdf.readthedocs.io/en/latest/recipes-images.html).

**Áp dụng khi nào?** Khi xây kho ảnh linh kiện, liên kết hình với bước bảo trì, tạo RAG đa phương thức hoặc hiển thị figure độc lập trên giao diện.

**Bài toán giải quyết:** tránh báo “mất hình” khi pipeline thực ra chỉ lấy ảnh nhúng nhưng figure là vector hoặc composite.

---

### 10. Kiểm thử đề mục, phân cấp section và thứ tự đọc ra sao?

**Trả lời:** Heading extraction gồm ba bài toán riêng:

1. **Detection:** dòng nào là title/heading.
2. **Level classification:** heading cấp 1, 2, 3 hay caption/list item.
3. **Tree construction:** section nào là cha của section nào và nội dung nào thuộc section đó.

#### Các case MRE quan trọng

- Heading đánh số: `4`, `4.2`, `4.2.1`.
- Heading không đánh số nhưng khác font/weight.
- Heading nhiều dòng.
- Running header giống tiêu đề nhưng lặp trên mọi trang.
- Mục lục chứa cùng tên heading nhưng không phải section body.
- Hai cột, text box bên lề và note cảnh báo.
- Heading ở cuối trang, nội dung bắt đầu ở trang sau.
- Tiêu đề nằm trong ảnh scan.
- Manual song ngữ có hai tiêu đề tương ứng.

#### Gold data nên biểu diễn thành cây

```json
{
  "id": "4",
  "title": "Maintenance",
  "level": 1,
  "children": [
    {
      "id": "4.2",
      "title": "Lubrication",
      "level": 2,
      "content_element_ids": ["p-47-3", "fig-4-3"]
    }
  ]
}
```

#### So sánh output

| Mức | Điều kiện pass |
| --- | --- |
| Text | Heading text đúng sau normalization đã quy định. |
| Geometry | Bbox overlap đạt ngưỡng. |
| Role | Không nhầm header/footer/caption thành heading. |
| Level | Level đúng hoặc sai lệch trong rule cho phép. |
| Tree | Parent-child và thứ tự section đúng. |
| Membership | Paragraph/table/figure được gắn đúng section. |

Azure Document Intelligence phân biệt geometric role như text/table/figure với logical role như title/heading/footer; đây là cách phân tách hữu ích khi thiết kế expected output. [Microsoft Document Intelligence — Layout model](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/prebuilt/layout?view=doc-intel-4.0.0).

**Áp dụng khi nào?** Khi chunk tài liệu theo section, tạo mục lục, RAG theo chương, sinh SOP hoặc liên kết hình/bảng với nội dung.

**Bài toán giải quyết:** ngăn chunk bị trộn chương, mất ngữ cảnh tiêu đề hoặc truy xuất đúng text nhưng sai phạm vi quy trình.

---

### 11. Làm sao tạo MRE khi output AI có tính không xác định?

**Trả lời:** “Reproducible” với model sinh không nhất thiết nghĩa là mọi ký tự phải giống nhau ở mọi lần chạy. Nó có thể nghĩa là **failure mode xuất hiện với tỷ lệ đo được trong một cấu hình đã khóa**.

#### Quy trình

1. Đóng băng input, prompt, model/snapshot, tool output và post-processing.
2. Tắt retry/fallback để không che lần lỗi đầu.
3. Chạy lặp, ví dụ 10 hoặc 20 lần tùy chi phí và mức độ hiếm của lỗi.
4. Lưu raw response, request ID, timestamp, latency và kết quả parser của từng lần.
5. Chấm theo tiêu chí semantic/schema thay vì chỉ diff chuỗi.
6. Báo tỷ lệ lỗi theo nhóm.

```text
Runs: 20
Schema invalid: 0/20
Missed heading 4.2: 7/20
Missed figure-caption link: 3/20
Hallucinated figure: 1/20
```

#### Phân loại

- **Deterministic preprocessing bug:** cùng input luôn tạo OCR/layout sai giống nhau.
- **Stochastic model failure:** cùng effective input nhưng model đôi lúc bỏ field hoặc gắn sai.
- **Version drift:** hành vi đổi sau khi model/SDK/service được nâng cấp.
- **Race/retry bug:** output phụ thuộc timeout, concurrency hoặc lần gọi fallback.

**Áp dụng khi nào?** Khi lỗi “thỉnh thoảng mới xảy ra”, A/B model hoặc thay prompt.

**Bài toán giải quyết:** biến nhận xét “model không ổn định” thành số liệu có thể so sánh giữa phiên bản.

**So sánh:** exact output equality phù hợp parser deterministic; pass-rate và metric distribution phù hợp model sinh.

---

### 12. Đóng gói MRE thế nào để nhóm nội bộ hoặc vendor chạy được ngay?

**Trả lời:** `README.md` của case nên trả lời được bảy câu hỏi trong vài phút:

1. Vấn đề là gì?
2. Expected và actual khác ở đâu?
3. File nào là input chuẩn?
4. Cấu hình/phần mềm chính xác là gì?
5. Chạy bằng lệnh nào?
6. Kết quả pass/fail được xác định thế nào?
7. Dữ liệu đã được ẩn và được phép chia sẻ chưa?

#### Mẫu README cho một case

```md
# MRE: Missing figure and heading on page 47

## Symptom
`4.2 Lubrication` is returned as body text and `Fig. 4-3` is absent.

## Expected
See `expected/expected.json`.

## Actual
See `actual/actual.json` and `actual/raw-response.json`.

## Environment
- OS / container digest
- OCR engine and language pack versions
- PDF parser version
- Model snapshot and API version
- Prompt revision / Git commit

## Run
`./run.sh`

## Pass criteria
- Heading `4.2 Lubrication` detected at level 2.
- Figure region IoU >= 0.8.
- Figure linked to section `4.2`.

## Reproduction rate
7/10 runs with retries disabled.

## Sanitization
Company and serial numbers replaced; layout preserved.
```

#### Đóng gói môi trường

- Dùng lockfile hoặc container image digest, không chỉ ghi `latest`.
- Một lệnh chạy tạo lại `actual/` từ input.
- Không yêu cầu credential thật; dùng biến môi trường và `.env.example`.
- Lưu raw output nhưng loại token/API key/PII.
- Nếu API trả request ID, lưu ID để vendor tra log.
- Với file lớn, dùng artifact storage có checksum và quyền truy cập rõ ràng.

**Áp dụng khi nào?** Khi handoff giữa data/AI/backend/QA, mở issue hoặc làm việc với cloud support.

**Bài toán giải quyết:** giảm trao đổi qua lại kiểu “thiếu file”, “không biết lệnh chạy”, “không giống phiên bản của tôi”.

**So sánh:** notebook thuận tiện khám phá nhưng dễ có hidden state; script/container dễ tái hiện hơn. Có thể kèm notebook để phân tích, nhưng `run.sh` hoặc test tự động nên là nguồn chuẩn.

---

### 13. Làm sao phát triển MRE thành regression suite cho production?

**Trả lời:** Sau khi sửa lỗi, MRE không nên bị bỏ quên. Hãy chuyển nó thành một fixture có metadata, gold output và test tự động.

#### Manifest gợi ý

```yaml
id: mre-manual-heading-figure-001
status: active
domain: industrial-machinery
document_type: maintenance-manual
pdf_type: scanned
languages: [en]
features: [ocr, heading, figure, caption-link]
failure_modes:
  - heading-demoted
  - figure-missed
severity: high
source: sanitized-production-case
pages: [47]
gold_version: 2
reviewed_by: mechanical-sme
last_reviewed: 2026-07-18
```

#### Các tầng kiểm thử

| Tầng | Nội dung | Tần suất |
| --- | --- | --- |
| Unit | Normalizer, bbox conversion, JSON parser. | Mỗi commit. |
| Component | OCR-only, layout-only, prompt-only trên fixture nhỏ. | Mỗi PR hoặc hằng ngày. |
| End-to-end | PDF → output cuối. | Hằng ngày/trước release. |
| Shadow/production sample | Dữ liệu mới đã kiểm soát quyền và ẩn danh. | Theo lịch và giám sát. |

#### Quy tắc cập nhật gold

- Không tự động chấp nhận output mới chỉ vì model mới tạo ra nó.
- Mọi thay đổi expected phải có lý do và reviewer hiểu tài liệu.
- Lưu phiên bản gold, prompt, model và metric trước/sau.
- Tách **acceptable variation** khỏi regression thật.
- Đặt threshold theo mức rủi ro; part number, cảnh báo và giá trị an toàn có thể cần exactness cao hơn text mô tả.
- Khi case không còn đại diện, deprecate có lý do thay vì xóa lịch sử.

#### Dashboard nên theo dõi

- OCR CER/WER theo ngôn ngữ và loại scan.
- Heading/figure/table F1 theo loại manual.
- Schema-valid rate.
- Critical field exact match: model, part number, torque, pressure, temperature, warning code.
- Latency p50/p95, cost/page, retry rate.
- Regression count theo model/prompt/parser version.

**Áp dụng khi nào?** Ngay sau khi MRE giúp tìm và sửa được một lỗi có khả năng tái diễn.

**Bài toán giải quyết:** ngăn nâng model, prompt, OCR engine hoặc parser làm sống lại lỗi cũ; tạo dữ liệu quyết định có nên release.

**So sánh:** MRE là một ca chẩn đoán nhỏ; regression suite là tập hợp MRE có tổ chức, metric, ownership và quy trình cập nhật.

---

## Checklist MRE thực hành

### Input

- [ ] Chỉ còn trang/vùng nhỏ nhất vẫn gây lỗi.
- [ ] Có file PDF gốc rút gọn và ảnh render tham chiếu khi cần.
- [ ] Ghi số trang, loại PDF và SHA-256.
- [ ] Sanitization đã được mô tả và kiểm tra lại.

### Pipeline

- [ ] Phiên bản PDF parser/OCR/layout model được pin.
- [ ] Ghi DPI, rotation, crop, deskew, denoise và language pack.
- [ ] Lưu output trung gian của từng tầng.
- [ ] Tắt retry/fallback khi điều tra lỗi đầu tiên.

### Prompt/model

- [ ] Có toàn bộ effective prompt và input asset.
- [ ] Model/snapshot, API, SDK và generation parameters rõ ràng.
- [ ] Không chứa key, token hoặc dữ liệu không được phép chia sẻ.
- [ ] Có raw response và request ID nếu hệ thống cung cấp.

### Expected/actual

- [ ] Expected output được reviewer có chuyên môn xác nhận.
- [ ] Có schema, coordinate convention và normalization rule.
- [ ] Tiêu chí pass/fail định lượng được.
- [ ] Với lỗi ngẫu nhiên, có số lần chạy và tỷ lệ lỗi.

### Execution

- [ ] Chạy được bằng một lệnh từ môi trường sạch.
- [ ] Dependency được khóa bằng lockfile/container digest.
- [ ] MRE đã được chạy lại trước khi chia sẻ.
- [ ] Sau khi sửa, case được chuyển vào regression suite.

## Mẫu quyết định nhanh

| Hiện tượng | MRE nên bắt đầu từ đâu? |
| --- | --- |
| PDF có text nhưng output rỗng | Native extraction + kiểm tra encoding/font trước OCR. |
| Scan đọc sai part number | Ảnh crop + DPI/preprocessing + OCR config + exact expected text. |
| Mất heading | Layout blocks + font/bbox + heading gold tree. |
| Mất hình | Phân biệt embedded image và rendered figure; lưu bbox/caption. |
| Sai thứ tự hai cột | Text/layout blocks có bbox + expected reading sequence. |
| JSON đôi lúc thiếu field | Prompt-only MRE + pinned model + lặp nhiều run. |
| Local đúng, server sai | Container/lockfile + OS/font/OCR language pack + checksum input. |
| Model mới làm giảm chất lượng | Regression suite chạy cùng fixture và metric trước/sau. |
| Manual chứa dữ liệu mật | Tách trang + thay nội dung giữ layout/pattern + reproduction check. |
| Tài liệu có cảnh báo an toàn | Gold data do chuyên gia xác nhận; không dùng AI output làm source of truth. |

## Nguồn tham khảo

1. Stack Overflow, [How to create a Minimal, Reproducible Example](https://stackoverflow.com/help/minimal-reproducible-example).
2. OpenAI, [API backward compatibility and model snapshot guidance](https://platform.openai.com/docs/api-reference/backward-compatibility).
3. Google Cloud, [Process documents with Gemini layout parser](https://docs.cloud.google.com/document-ai/docs/layout-parse-chunk).
4. Google Cloud, [Document AI supported files and OCR image-quality guidance](https://docs.cloud.google.com/document-ai/docs/file-types).
5. Microsoft, [Document Intelligence layout model](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/prebuilt/layout?view=doc-intel-4.0.0).
6. Microsoft, [Document Intelligence analyze response: sections and figures](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/analyze-document-response?view=doc-intel-4.0.0).
7. Amazon Web Services, [Amazon Textract Block types](https://docs.aws.amazon.com/textract/latest/APIReference/API_Block.html).
8. PyMuPDF, [OCR recipes](https://pymupdf.readthedocs.io/en/latest/recipes-ocr.html).
9. PyMuPDF, [Image extraction recipes](https://pymupdf.readthedocs.io/en/latest/recipes-images.html).
10. Tesseract OCR, [Improving OCR output quality](https://tesseract-ocr.github.io/tessdoc/ImproveQuality.html).
11. Tesseract OCR, [Input formats and PDF limitation](https://tesseract-ocr.github.io/tessdoc/InputFormats.html).
12. IEC, [IEC/IEEE 82079-1:2019 — Preparation of information for use of products](https://webstore.iec.ch/en/publication/29075).
