# ETL tài liệu PDF kỹ thuật cho RAG

> [Knowledge Tree](../../../../README.md) / [AI](../../../README.md) / [LLM](../../README.md) / [RAG](../README.md) / **ETL tài liệu PDF kỹ thuật**

**Extract, Transform, Load (ETL)** cho tài liệu PDF kỹ thuật là pipeline biến file gốc không đồng nhất — PDF có text/vector, ảnh scan, biểu mẫu, bảng, sơ đồ — thành dữ liệu có cấu trúc, có nguồn gốc và đủ chất lượng để dùng cho **Retrieval-Augmented Generation (RAG)**.

Mục tiêu không phải chỉ “OCR ra text”, mà phải giữ được:

- tài liệu, phiên bản, trang và section;
- thứ tự đọc (*reading order*), heading, đoạn văn, bảng, hình và chú thích;
- key–value, checkbox, chữ ký hoặc trường nghiệp vụ trong biểu mẫu;
- tọa độ (*bounding box*) để mở đúng vị trí trên PDF;
- quyền truy cập (*Access Control List, ACL*), nguồn gốc (*provenance*) và trạng thái hiệu lực;
- dữ liệu lỗi, độ tin cậy và khả năng chạy lại (*reprocessing*).

> **PDF source → Inventory → Classify → Route → Parse/OCR/Layout/Form → Normalize → Validate → Chunk → Index → Publish**

Với 50.000 đến 1.000.000 PDF, đơn vị sizing phải là **số trang và độ khó của trang**, không chỉ là số file. Một triệu file, trung bình 20 trang, tương đương 20 triệu page-work-item cần lập lịch, retry, lưu trạng thái và kiểm soát chi phí.

## Điều hướng

- **Node cha:** [RAG](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** [RAGAS trong RAG](../ragas/README.md)
- **Khái niệm sau trong cùng nhánh:** Chưa có.

## Kiến trúc khuyến nghị

```text
                    ┌──────────────────────────────────────────┐
                    │ Object Storage: original PDF (immutable) │
                    └──────────────────┬───────────────────────┘
                                       │ manifest + checksum
                              ┌────────▼─────────┐
                              │ Inventory/Router │
                              └───┬────┬────┬────┘
                                  │    │    │
                   digital text ──┘    │    └── complex/form/diagram
                                       │
                                     scan
        ┌──────────────────┐  ┌────────▼────────┐  ┌─────────────────────┐
        │ Fast PDF parser  │  │ OCR preprocess │  │ Layout/Form/VLM     │
        │ text + geometry  │  │ rotate/deskew  │  │ tables + fields      │
        └────────┬─────────┘  └────────┬────────┘  └──────────┬──────────┘
                 └─────────────────────┴───────────────────────┘
                                       │
                              ┌────────▼─────────┐
                              │ Canonical JSON   │
                              │ page/block/table │
                              │ field/figure/ACL │
                              └────────┬─────────┘
                                       │ quality gate
             ┌─────────────────────────┼──────────────────────────┐
             │                         │                          │
     ┌───────▼────────┐       ┌────────▼────────┐        ┌────────▼────────┐
     │ Search index   │       │ Vector index    │        │ Structured DB   │
     │ BM25/exact code│       │ semantic chunks │        │ forms/workflows │
     └────────────────┘       └─────────────────┘        └─────────────────┘
```

**Nguyên tắc cốt lõi:** dùng **router nhiều tầng**. Không chạy OCR/layout/VLM đắt tiền cho toàn bộ corpus. Trang có text tốt đi qua parser nhanh; trang scan mới OCR; bảng, biểu mẫu, sơ đồ hoặc kết quả confidence thấp mới chuyển sang pipeline mạnh hơn hay human review.

---

## A. 5 câu hỏi cốt lõi

### 1. ETL cho PDF kỹ thuật khác gì với việc chỉ OCR toàn bộ file?

**Trả lời:** OCR chỉ là một bước nhận dạng ký tự từ ảnh. ETL tài liệu phải giải quyết đầy đủ vòng đời dữ liệu: tiếp nhận file, phát hiện loại tài liệu, chọn parser, chuẩn hóa cấu trúc, kiểm tra chất lượng, tạo chunk, index, versioning, ACL và publish có kiểm soát.

Một pipeline tối thiểu nên có các tầng:

1. **Landing/Bronze:** lưu PDF gốc bất biến, checksum, nguồn, thời gian nhận và ACL ban đầu.
2. **Inventory:** đọc metadata kỹ thuật như kích thước, số trang, encryption, text coverage, image ratio, ngôn ngữ dự kiến.
3. **Classification/Route:** phân tuyến từng tài liệu hoặc từng trang.
4. **Extraction:** parser text/vector, OCR, layout/table, form/key–value hoặc VLM fallback.
5. **Canonical/Silver:** đưa mọi engine về cùng schema.
6. **Quality gate:** kiểm tra cấu trúc, confidence, sampling và rule nghiệp vụ.
7. **RAG/Gold:** chunk, embedding, full-text index, citation metadata và document status.
8. **Publish:** chuyển alias/index version sau khi pass kiểm thử; rollback được.

Apache Tika có thể phát hiện và trích xuất text/metadata từ hơn một nghìn định dạng, còn OCRmyPDF thêm text layer có thể tìm kiếm cho PDF scan. Các công cụ layout-aware như Docling hoặc Unstructured tập trung thêm vào reading order, bảng, hình và cấu trúc tài liệu. [Apache Tika][tika] [OCRmyPDF][ocrmypdf] [Docling][docling] [Unstructured][unstructured]

**Áp dụng khi nào?**

- Manual thiết bị, hướng dẫn vận hành, sửa chữa, xử lý sự cố.
- Biên bản kiểm tra có bảng, checkbox, chữ ký hoặc ghi chú viết tay.
- Phiếu đề nghị cấp/xuất vật tư có mã vật tư, số lượng, đơn vị, người đề nghị.
- Kho tài liệu có nhiều phiên bản, nhiều hãng, nhiều model và nhiều quyền truy cập.

**Bài toán giải quyết:**

- Tránh mất heading, bảng, số trang và nguồn citation.
- Không dùng chi phí OCR/VLM cho PDF đã có text tốt.
- Có thể tìm lỗi theo từng bước thay vì chỉ biết “RAG trả lời sai”.
- Tách dữ liệu hỏi đáp khỏi dữ liệu giao dịch cần lưu vào hệ thống nghiệp vụ.

**So sánh:**

| Cách làm | Điểm mạnh | Giới hạn | Chọn khi |
| --- | --- | --- | --- |
| OCR toàn bộ | Dễ hiểu, một luồng duy nhất. | Chậm, tốn chi phí, có thể làm hỏng text/vector tốt và mất cấu trúc. | Corpus nhỏ, gần như toàn bộ là scan đơn giản. |
| Parser PDF thuần | Nhanh, rẻ, giữ text số. | Không đọc được scan và thường yếu với reading order/bảng phức tạp. | PDF sinh từ phần mềm, layout đơn giản. |
| Layout-aware parser | Giữ cấu trúc, bảng, hình tốt hơn. | Tốn CPU/GPU hơn; cần benchmark. | Manual, catalogue, báo cáo kỹ thuật nhiều layout. |
| ETL có router | Tối ưu chất lượng, chi phí và khả năng scale. | Kiến trúc, observability và test phức tạp hơn. | Từ hàng chục nghìn đến hàng triệu PDF hỗn hợp. |

---

### 2. Nên phân loại và route PDF/vector/scan ở mức file hay mức trang?

**Trả lời:** Nên phân loại hai cấp: **file-level preflight** để quyết định luồng sơ bộ và **page-level routing** cho PDF lai (*hybrid PDF*). Một file có thể có 30 trang text số, 5 trang scan và 2 trang bảng phức tạp; ép cả file vào một pipeline sẽ lãng phí hoặc giảm chất lượng.

**Preflight file-level:**

- file có mở được, bị hỏng, mã hóa hoặc có chữ ký số hay không;
- số trang, kích thước, DPI ước lượng và tổng số ảnh;
- text character count, tỷ lệ trang có text, font/glyph bất thường;
- duplicate theo checksum và near-duplicate theo fingerprint;
- ngôn ngữ, loại tài liệu và phiên bản thiết bị dự kiến.

**Routing page-level:**

| Tín hiệu | Route mặc định |
| --- | --- |
| Text layer đủ, reading order hợp lý | Fast parser. |
| Không có text hoặc text coverage rất thấp | OCR. |
| Text có nhưng glyph lỗi/không tìm kiếm được | Re-OCR có kiểm soát. |
| Nhiều cột, bảng, figure, caption | Layout-aware parser. |
| Form cố định, key–value, checkbox | Form extractor/custom model. |
| Sơ đồ điện, exploded view, ảnh có callout | Image/diagram extraction + multimodal enrichment. |
| Confidence thấp hoặc rule bất thường | Fallback engine hoặc human review. |

OCRmyPDF hỗ trợ rotate, deskew, clean và tạo sidecar text; Docling có pipeline OCR, layout và table; Unstructured cung cấp các strategy như `fast`, `hi_res` và `ocr_only`. Các khả năng này phù hợp để xây router thay vì chọn một engine duy nhất. [OCRmyPDF cookbook][ocrmypdf-cookbook] [Docling pipeline][docling-pipeline] [Unstructured strategies][unstructured]

**Áp dụng khi nào?** Luôn nên dùng page-level route khi PDF được ghép từ nhiều nguồn, scan bổ sung vào tài liệu số, hoặc hồ sơ có cả form và tài liệu hướng dẫn.

**Bài toán giải quyết:** giảm số page phải OCR/layout, tránh OCR chồng lên text tốt, và cô lập những trang khó để xử lý riêng.

**So sánh:**

- **File-level only:** triển khai nhanh nhưng quá thô.
- **Page-level:** tối ưu hơn, nhưng cần manifest và merge kết quả đúng thứ tự.
- **Region-level:** tốt nhất với form/sơ đồ rất phức tạp, nhưng chỉ nên dùng cho loại tài liệu có ROI cao.

---

### 3. Schema dữ liệu trung gian nên thiết kế thế nào để dùng được cho cả RAG và biểu mẫu?

**Trả lời:** Không lưu output riêng theo từng engine làm hợp đồng dữ liệu chính. Hãy chuẩn hóa về một **Canonical Document Model** có thể biểu diễn text, geometry, bảng, hình, field và quan hệ.

Ví dụ rút gọn:

```json
{
  "documentId": "manual-pump-x100",
  "documentVersion": "sha256:...",
  "sourceUri": "s3://docs/manual-x100-v3.pdf",
  "documentType": "equipment_manual",
  "equipment": {"manufacturer": "...", "model": "X100"},
  "language": "vi",
  "status": "active",
  "acl": {"tenantId": "factory-a", "roles": ["maintenance"]},
  "pages": [
    {
      "pageNumber": 12,
      "width": 2480,
      "height": 3508,
      "blocks": [
        {
          "blockId": "p12-b07",
          "type": "procedure_step",
          "text": "Ngắt nguồn điện trước khi tháo nắp...",
          "bbox": [120, 450, 2200, 640],
          "confidence": 0.97,
          "sectionPath": ["5. Bảo trì", "5.2 Tháo nắp"],
          "engine": "docling-x.y"
        }
      ],
      "tables": [],
      "figures": []
    }
  ]
}
```

**Field quan trọng:**

- `documentId`, `documentVersion`, `sourceUri`, checksum;
- `pageNumber`, page dimensions, rotation;
- `blockId`, `type`, `text`, `bbox`, reading order;
- `sectionPath`, heading level, list/step number;
- table cells với row/column span và header relation;
- figure/image URI, caption, page region;
- form fields: key, value, confidence, normalized value;
- `engine`, model/version, prompt/config hash;
- ACL, retention, effective date và status;
- lỗi/warning và lineage đến file gốc.

Docling Document giữ layout/bounding box và mô hình bảng; Azure Document Intelligence trả cấu trúc text, table, selection mark và section; Amazon Textract biểu diễn forms/tables theo các block và quan hệ. [Docling Document][docling-document] [Azure layout][azure-layout] [Amazon Textract][textract-analysis]

**Áp dụng khi nào?** Khi có nhiều engine, muốn re-index mà không extract lại, hoặc cần vừa hỏi đáp vừa đẩy dữ liệu form sang workflow nghiệp vụ.

**Bài toán giải quyết:** tránh vendor lock-in ở schema, hỗ trợ audit, A/B parser và thay model mà không đổi toàn bộ downstream.

**So sánh:**

| Lưu chỉ Markdown/text | Canonical JSON |
| --- | --- |
| Đơn giản, phù hợp prototype. | Phức tạp hơn nhưng giữ geometry, bảng, field và lineage. |
| Khó mở đúng vùng PDF. | Có thể highlight đúng page/bbox. |
| Khó dùng cho form/table. | Dùng chung cho RAG, search, QA và automation. |

---

### 4. Làm sao scale từ 50.000 lên 1.000.000 PDF mà không phải thiết kế lại toàn bộ?

**Trả lời:** Tách pipeline thành các **stage độc lập, idempotent và queue-driven**. Mỗi stage nhận work item có khóa ổn định, ghi output bất biến, cập nhật state store và phát sự kiện cho stage tiếp theo.

```text
Object Storage
  → manifest/inventory queue
  → classify queue
  → parser queue | OCR queue | layout queue | form queue
  → normalize queue
  → validate queue
  → chunk/embed/index queue
  → publish index alias
```

**Các nguyên tắc scale:**

1. **Size theo page:** lưu `pageCount`, `estimatedComplexity`, `route` và `expectedComputeClass`.
2. **Tách worker pool:** parser CPU nhẹ, OCR CPU/GPU, layout GPU, cloud API connector và embedding worker không tranh tài nguyên.
3. **Backpressure:** queue depth, rate limit và quota quyết định autoscale; không đọc toàn bộ corpus vào scheduler.
4. **Idempotency:** khóa như `documentVersion + page + stage + configHash`; retry không tạo duplicate.
5. **Checkpoint:** output theo page/range để một trang lỗi không làm chạy lại file 500 trang.
6. **Dead-letter queue (DLQ):** lỗi hỏng file, timeout, quota, unsupported encryption hoặc confidence thấp được phân loại riêng.
7. **Immutable output:** mỗi lần đổi parser/model tạo extraction version mới; publish bằng alias sau validation.
8. **Batch manifest:** chia corpus theo tenant, loại tài liệu, kích thước và ưu tiên nghiệp vụ.

Apache Beam cung cấp mô hình thống nhất cho batch và streaming, phù hợp cho transform dữ liệu quy mô lớn; các API như Amazon Textract và Google Document AI hỗ trợ xử lý bất đồng bộ/batch cho tài liệu nhiều trang. [Apache Beam][beam] [Textract async][textract-async] [Document AI batch][document-ai-batch]

**Áp dụng khi nào?** Từ khoảng hàng chục nghìn file trở lên, đặc biệt khi có backfill lớn rồi tiếp tục ingest tăng dần hằng ngày.

**Bài toán giải quyết:** autoscale từng stage, retry cục bộ, tránh scheduler quá tải và cho phép chuyển engine mà không ảnh hưởng toàn pipeline.

**So sánh orchestration:**

| Cách | Phù hợp | Không nên dùng |
| --- | --- | --- |
| Script tuần tự | POC vài trăm file. | Backfill lớn, cần retry/quan sát. |
| Airflow/Argo orchestration theo batch | Lập lịch, dependency và backfill theo dataset/job. | Tạo một task orchestration cho từng trang trong hàng chục triệu trang. |
| Queue + worker | Fan-out lớn, retry và autoscale theo stage. | Workflow rất ít task, cần giao diện DAG là chính. |
| Beam/Spark/Dataflow | Normalize, enrich, dedup và transform quy mô lớn. | Gọi OCR nặng mà không có connector/timeout/idempotency rõ. |

---

### 5. Đánh giá chất lượng extract thế nào trước khi cho dữ liệu vào RAG?

**Trả lời:** Không dùng một metric OCR duy nhất. Quality gate phải đo theo loại thành phần và đo cả tác động downstream lên retrieval/câu trả lời.

**Bộ đánh giá nên phân tầng:**

| Tầng | Metric gợi ý | Ví dụ lỗi |
| --- | --- | --- |
| File/page | parse success, missing page, rotation, empty text | Mất trang phụ lục, trang xoay 90°. |
| OCR text | Character Error Rate, Word Error Rate, exact code accuracy | `O` thành `0`, `I` thành `1`, sai mã lỗi. |
| Layout | reading-order accuracy, heading/paragraph classification | Đọc cột phải trước cột trái. |
| Table | cell/row/column accuracy, header association | Lệch cột số lượng và đơn vị. |
| Form | field precision/recall/F1, normalized-value accuracy | Lấy nhầm ngày kiểm tra hoặc mã vật tư. |
| Figure | caption-link accuracy, page/bbox validity | Chú thích gắn nhầm sơ đồ. |
| RAG | Recall@k, citation correctness, answer faithfulness | Không retrieve được bước xử lý lỗi. |

**Sampling bắt buộc:** tạo gold set theo từng nhóm: digital, scan sạch, scan mờ, nhiều cột, bảng, form, chữ viết tay, sơ đồ, tiếng Việt/Anh và từng hãng thiết bị. Không chỉ random sample vì case khó thường chiếm tỷ lệ nhỏ nhưng rủi ro cao.

**Quality gate ví dụ:**

- không publish nếu thiếu trang hoặc checksum không khớp;
- mã thiết bị/mã lỗi phải đạt exact-match threshold cao hơn text mô tả;
- field nghiệp vụ quan trọng có confidence thấp phải human review;
- bảng vật tư phải pass rule tổng, đơn vị và số lượng;
- một index version mới chỉ publish khi eval retrieval không giảm quá ngưỡng đã định.

**Áp dụng khi nào?** Bắt buộc với tài liệu an toàn, vận hành, sửa chữa, kiểm tra và cấp phát vật tư vì sai một ký tự có thể làm sai quy trình hoặc mã thiết bị.

**Bài toán giải quyết:** biến chất lượng từ cảm nhận thành release gate có thể regression test.

**So sánh:**

- **Confidence của engine:** hữu ích để route, nhưng không phải ground truth.
- **Review thủ công:** sâu nhưng không scale; dùng cho gold set và low-confidence queue.
- **Downstream RAG eval:** đo tác động thực, nhưng cần kết hợp kiểm tra extract để xác định root cause.

---

## B. 8 câu hỏi phổ biến mở rộng

### 6. Nên chọn open-source, cloud Document AI hay mô hình hybrid?

**Trả lời:** Với 50.000–1.000.000 PDF, phương án thực tế thường là **hybrid routing**: open-source cho đường nhanh và dữ liệu nhạy cảm; managed Document AI cho form/layout khó, burst capacity hoặc khi đội ngũ không muốn vận hành model.

| Nhóm | Ví dụ | Điểm mạnh | Giới hạn | Chọn khi |
| --- | --- | --- | --- | --- |
| Metadata/text parser | Apache Tika, PDF parser | Nhanh, rẻ, self-host. | Yếu với scan/layout phức tạp. | PDF digital, preflight. |
| OCR/searchable PDF | OCRmyPDF + Tesseract | Self-host, rotate/deskew, text layer. | Bảng/reading order không phải trọng tâm. | Scan text thông thường. |
| Layout-aware OSS | Docling, Unstructured | Bảng, hình, reading order, JSON/Markdown. | Cần benchmark CPU/GPU và version model. | Manual/catalogue/report. |
| Managed Document AI | Google Document AI, Azure Document Intelligence, Amazon Textract | Async/batch, forms/tables, managed scale. | Chi phí theo trang, quota, data residency, lock-in output. | Form, burst lớn, time-to-market. |
| VLM | Vision-language model | Hiểu bố cục/sơ đồ linh hoạt. | Đắt, ít deterministic, cần guardrail. | Fallback cho trang khó, không làm mặc định toàn corpus. |

Google Document AI Layout Parser tạo cấu trúc và chunk theo layout; Azure Layout trích text, table và structure; Amazon Textract hỗ trợ text, forms, tables, queries và signatures. [Google Layout Parser][google-layout] [Azure layout][azure-layout] [Amazon Textract][textract-analysis]

**Quy tắc quyết định:** benchmark trên 500–2.000 trang đại diện, tính **cost per accepted page**, không chỉ cost per processed page. Một engine rẻ nhưng tạo nhiều lỗi/human review có thể đắt hơn.

---

### 7. Xử lý tài liệu hướng dẫn kỹ thuật có bảng, sơ đồ và tham chiếu chéo thế nào?

**Trả lời:** Tài liệu kỹ thuật không nên được flatten thành một chuỗi text duy nhất.

- Giữ `sectionPath`, số bước, cảnh báo, điều kiện và thứ tự procedure.
- Không cắt rời “CẢNH BÁO/CAUTION” khỏi bước thao tác liên quan.
- Bảng phải lưu cả HTML/JSON grid và phiên bản text tuyến tính cho search.
- Figure phải được tách ảnh, giữ caption, page và bbox.
- Callout trong hình như `1`, `2`, `A`, `B` phải liên kết với legend nếu có.
- Tham chiếu “xem Hình 5-2”, “xem mục 7.3” cần resolve thành relation giữa block.
- Mã lỗi, part number, model và torque value phải được đưa vào lexical index; không dựa hoàn toàn vào embedding.

Docling hỗ trợ layout, bảng, hình và reading order; khả năng export table cho phép giữ dữ liệu dạng Markdown/CSV/HTML. [Docling][docling] [Docling tables][docling-tables]

**Áp dụng khi nào?** Manual vận hành, service manual, catalogue phụ tùng, sơ đồ điện/khí nén và quy trình xử lý sự cố.

**Bài toán giải quyết:** giúp RAG trả đúng bước, điều kiện, cảnh báo và hình liên quan thay vì chỉ tìm đoạn văn gần nghĩa.

**So sánh:**

- **Text-only RAG:** đủ cho FAQ đơn giản.
- **Layout-aware RAG:** cần cho manual nhiều cột/bảng.
- **Multimodal RAG:** thêm giá trị khi câu trả lời phụ thuộc trực tiếp vào sơ đồ/hình, nhưng phải giữ citation đến image region.

---

### 8. Biên bản kiểm tra và đề nghị cấp/xuất vật tư nên đưa hết vào vector database không?

**Trả lời:** Không. Cần tách hai mục tiêu:

1. **RAG/search:** dùng text và chunk để tra cứu nội dung, hướng dẫn, ghi chú, lý do và lịch sử.
2. **Structured extraction/workflow:** lưu các field nghiệp vụ vào database hoặc API nguồn sự thật.

Ví dụ field của phiếu vật tư:

```text
request_id, request_date, department, equipment_id,
item_code, item_name, quantity, unit, reason,
requester, approver, approval_status, source_page, confidence
```

Amazon Textract có form key–value và table extraction; Azure Document Intelligence hỗ trợ key-value/table/selection marks; với form ổn định có thể dùng custom model hoặc template rule. [Textract forms][textract-forms] [Azure layout][azure-layout]

**Áp dụng khi nào?** Khi dữ liệu được dùng để phê duyệt, thống kê tồn kho, tạo phiếu hoặc đối soát.

**Bài toán giải quyết:** vector database không đảm bảo transaction, uniqueness, trạng thái phê duyệt hay tính toán tồn kho; structured DB/API mới là source of truth.

**So sánh:**

| Vector index | Structured database |
| --- | --- |
| Tìm gần nghĩa và hỏi đáp. | Truy vấn chính xác, aggregate, constraint, workflow. |
| Dữ liệu chunk không phù hợp làm bản ghi giao dịch. | Phù hợp lưu field đã chuẩn hóa và trạng thái. |
| Có thể chứa bản text tham chiếu. | Giữ liên kết về PDF/page để audit. |

---

### 9. Chunking tài liệu kỹ thuật nên thực hiện trước hay sau extract layout?

**Trả lời:** Chunking phải thực hiện **sau khi** đã có layout/section/reading order. Cắt theo token trước sẽ phá heading, procedure, bảng và cảnh báo.

**Chiến lược:**

- chunk theo section/subsection;
- procedure: giữ tiêu đề, điều kiện, cảnh báo và các bước liên tiếp;
- troubleshooting: một symptom + possible cause + remedy là một unit;
- bảng: lưu table như một unit; bảng dài có thể chia theo nhóm hàng nhưng lặp header;
- form: mỗi bản ghi hoặc nhóm field liên quan là một unit;
- parent-child: retrieve chunk nhỏ, bổ sung parent section/neighbor khi assemble context;
- figure: tạo chunk caption + mô tả + link image; không tự suy diễn nội dung quan trọng nếu chưa có validation.

**Metadata chunk tối thiểu:** `documentId`, `documentVersion`, `pageStart`, `pageEnd`, `sectionPath`, `equipmentModel`, `documentType`, `language`, `acl`, `sourceUri`, `bboxRefs`, `effectiveDate`.

**Áp dụng khi nào?** Với mọi corpus PDF dùng cho RAG, đặc biệt manual và quy trình.

**Bài toán giải quyết:** tăng Recall@k, citation đúng và tránh trả lời thiếu điều kiện an toàn.

**So sánh:** fixed-size chỉ phù hợp baseline; section-aware và parent-child thường phù hợp hơn cho tài liệu kỹ thuật.

---

### 10. Quản lý duplicate, phiên bản và cập nhật incremental thế nào?

**Trả lời:** Mỗi file cần hai lớp định danh:

- **content identity:** checksum của bytes hoặc nội dung chuẩn hóa;
- **business identity:** mã tài liệu, thiết bị, phiên bản, ngày hiệu lực, đơn vị sở hữu.

**Quy trình:**

1. checksum trùng → không extract lại, chỉ cập nhật liên kết nguồn nếu cần;
2. business ID giống nhưng checksum khác → tạo version mới;
3. near-duplicate → so fingerprint theo trang/text để phát hiện scan lại hoặc thêm watermark;
4. extract version phụ thuộc `parserVersion + modelVersion + configHash`;
5. chunk/index version phụ thuộc extraction version và chunk config;
6. publish version mới bằng alias; version cũ giữ để rollback theo retention policy.

**Áp dụng khi nào?** Kho tài liệu từ nhiều thư mục, email, DMS và scan lại; tài liệu hãng cập nhật định kỳ.

**Bài toán giải quyết:** tránh embed hàng triệu chunk lặp, không trộn hướng dẫn cũ/mới và audit được câu trả lời dùng phiên bản nào.

**So sánh:** ghi đè file đơn giản nhưng phá lineage; append-only + status/alias phức tạp hơn nhưng an toàn cho production.

---

### 11. Bảo mật và ACL nên được gắn ở bước nào?

**Trả lời:** ACL phải đi cùng tài liệu từ landing zone và được propagate đến page, block, chunk và index. Không đợi đến lúc LLM đã nhận context mới lọc.

**Kiểm soát cần có:**

- object storage encryption và private network khi phù hợp;
- tenant/site/department/role/document classification;
- PII hoặc thông tin nhạy cảm trong biên bản;
- secret/credential redaction trong tài liệu kỹ thuật;
- retention và quyền xóa;
- audit log cho ingest, review, publish và query;
- hard filter ACL trong retrieval;
- index riêng khi yêu cầu cách ly vật lý hoặc data residency.

**Áp dụng khi nào?** Hầu hết dữ liệu nội bộ; đặc biệt hồ sơ thiết bị, vật tư, nhà máy và biên bản có tên/chữ ký.

**Bài toán giải quyết:** ngăn người dùng retrieve được chunk ngoài quyền dù semantic score cao.

**So sánh:** post-filter sau retrieval có nguy cơ thiếu kết quả hoặc rò rỉ context; pre-filter/hard filter an toàn hơn.

---

### 12. Cần theo dõi metric vận hành và chiến lược retry nào?

**Trả lời:** Observability phải theo từng stage, engine, loại tài liệu và tenant.

**Metric chính:**

- input files/pages, throughput pages/minute;
- queue depth và oldest-message age;
- stage latency p50/p95/p99;
- success, retry, timeout, DLQ rate;
- empty-page, low-confidence, table/form detection rate;
- CPU/GPU utilization, memory, temporary disk;
- cloud API request, throttling và cost estimate;
- output bytes, chunk count, embedding count;
- validation failure theo rule;
- publish lag và index freshness.

**Retry classification:**

| Lỗi | Xử lý |
| --- | --- |
| Network/5xx/quota | Exponential backoff + jitter. |
| Timeout file lớn | Split page range, tăng worker class có giới hạn. |
| Corrupt/encrypted | DLQ + reason code, không retry vô hạn. |
| Low confidence | Fallback engine hoặc human review. |
| Schema/rule failure | Quarantine output, giữ raw result để debug. |
| Bug/config mới | Reprocess theo extraction version mới. |

**Áp dụng khi nào?** Ngay từ pilot; không chờ đến một triệu file mới thêm monitoring.

**Bài toán giải quyết:** biết bottleneck nằm ở OCR, layout, API quota, storage hay indexing; dự báo thời gian backfill và chi phí.

---

### 13. Lộ trình triển khai và stack tham khảo theo quy mô là gì?

**Trả lời:** Bắt đầu bằng sampling và benchmark, không bắt đầu bằng mua GPU hoặc chọn vector database.

#### Giai đoạn 1 — Khảo sát 1–2 tuần dữ liệu

- inventory 1.000–5.000 file đại diện;
- thống kê trang/file, digital/scan/hybrid, ngôn ngữ, bảng/form/sơ đồ;
- xác định 20–50 loại tài liệu chính và mức rủi ro;
- tạo gold set 500–2.000 trang cùng 100–300 câu hỏi RAG.

#### Giai đoạn 2 — Pilot

- object storage + manifest DB;
- queue + container workers;
- fast parser + OCR + một layout engine;
- canonical JSON, quality dashboard;
- hybrid search BM25 + vector;
- citation mở đúng page/bbox.

#### Giai đoạn 3 — Production backfill

- worker pool tách theo route;
- autoscale, quota manager, DLQ;
- page/range checkpoint;
- index version + alias + rollback;
- human-review queue cho field quan trọng;
- cost and quality gate trước publish.

#### Giai đoạn 4 — Continuous ingestion

- event/CDC từ DMS/object storage;
- incremental extract/chunk/index;
- regression suite khi đổi parser/model;
- review tài liệu hết hiệu lực và ACL định kỳ.

**Stack vendor-neutral tham khảo:**

| Thành phần | 50k–200k PDF | 200k–1M PDF |
| --- | --- | --- |
| Storage | S3/GCS/Blob/MinIO | Object storage + lifecycle + replicated metadata. |
| State/manifest | PostgreSQL | PostgreSQL partitioning hoặc scalable metadata store. |
| Queue | RabbitMQ/SQS/PubSub/Service Bus | Managed queue/event bus, nhiều queue theo workload. |
| Orchestrator | Airflow/Argo/Temporal theo batch | Orchestrator theo batch + queue workers; tránh task-per-page trong DAG. |
| Parser/OCR | Tika/PDF parser, OCRmyPDF/Tesseract | Nhiều worker pool; GPU/layout và managed fallback. |
| Layout | Docling/Unstructured | OSS + managed Document AI theo route/priority. |
| Transform | Python/Java workers | Beam/Spark/Dataflow cho normalize/enrich quy mô lớn. |
| Search | PostgreSQL + pgvector hoặc OpenSearch | OpenSearch/Elasticsearch/vector DB được benchmark theo chunk/filter. |
| Analytics | SQL + dashboard | Data lake/lakehouse cho lineage, cost và quality history. |

**Khuyến nghị mặc định cho bài toán này:**

1. Lưu PDF gốc bất biến trong object storage.
2. PostgreSQL giữ manifest, version, route, trạng thái và lỗi.
3. Queue tách `fast-parse`, `ocr`, `layout`, `form`, `fallback`, `normalize`, `index`.
4. Fast path dùng PDF parser/Tika; scan dùng OCRmyPDF/Tesseract; layout dùng Docling hoặc Unstructured.
5. Managed Document AI chỉ dùng cho form/layout khó, burst hoặc low-confidence fallback sau benchmark.
6. Canonical JSON là hợp đồng dữ liệu trung gian.
7. Search dùng BM25 + vector; structured form đi vào database/API riêng.
8. Publish theo index version và alias sau quality gate.

---

## Checklist nghiệm thu

- [ ] PDF gốc bất biến, có checksum và source URI.
- [ ] Có manifest theo file/page/stage/config version.
- [ ] Route digital/scan/layout/form ở mức trang khi cần.
- [ ] Output canonical giữ page, bbox, reading order, table và field.
- [ ] Có ACL từ source đến chunk/index.
- [ ] Retry idempotent, checkpoint và DLQ.
- [ ] Có gold set theo từng loại tài liệu và case khó.
- [ ] Đo extract quality và Recall@k/citation downstream.
- [ ] Structured form không dùng vector DB làm source of truth.
- [ ] Index version mới có validation, alias và rollback.
- [ ] Dashboard có throughput, lỗi, queue age, quality và cost.
- [ ] Thay parser/model không ghi đè output cũ.

## Nguồn tham khảo

- [Apache Tika — phát hiện và trích xuất text/metadata][tika]
- [OCRmyPDF — tạo searchable text layer cho PDF scan][ocrmypdf]
- [OCRmyPDF Cookbook — rotate, deskew, clean và sidecar text][ocrmypdf-cookbook]
- [Docling — PDF understanding, OCR, layout, bảng và hình][docling]
- [Docling pipeline options][docling-pipeline]
- [Docling Document model][docling-document]
- [Docling table export][docling-tables]
- [Unstructured PDF partitioning strategies][unstructured]
- [Google Document AI Layout Parser][google-layout]
- [Google Document AI batch processing][document-ai-batch]
- [Azure Document Intelligence Layout][azure-layout]
- [Amazon Textract document analysis][textract-analysis]
- [Amazon Textract asynchronous multipage processing][textract-async]
- [Amazon Textract forms/key–value][textract-forms]
- [Apache Beam overview][beam]

[tika]: https://tika.apache.org/
[ocrmypdf]: https://ocrmypdf.readthedocs.io/en/stable/
[ocrmypdf-cookbook]: https://ocrmypdf.readthedocs.io/en/latest/cookbook.html
[docling]: https://docling-project.github.io/docling/
[docling-pipeline]: https://docling-project.github.io/docling/reference/pipeline_options/
[docling-document]: https://docling-project.github.io/docling/concepts/docling_document/
[docling-tables]: https://docling-project.github.io/docling/_generated/examples/export_tables/
[unstructured]: https://docs.unstructured.io/open-source/core-functionality/partitioning
[google-layout]: https://docs.cloud.google.com/document-ai/docs/layout-parse-chunk
[document-ai-batch]: https://docs.cloud.google.com/document-ai/docs/reference/rest/v1/projects.locations.processors/batchProcess
[azure-layout]: https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/prebuilt/layout?view=doc-intel-4.0.0
[textract-analysis]: https://docs.aws.amazon.com/textract/latest/dg/how-it-works-analyzing.html
[textract-async]: https://docs.aws.amazon.com/textract/latest/dg/async.html
[textract-forms]: https://docs.aws.amazon.com/textract/latest/dg/how-it-works-kvp.html
[beam]: https://beam.apache.org/get-started/beam-overview/
