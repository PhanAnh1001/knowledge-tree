# OCR và trích xuất ảnh con bằng mô hình mã nguồn mở

> [Knowledge Tree](../../../../../README.md) / [AI](../../../../README.md) / [LLM](../../../README.md) / [RAG](../../README.md) / [ETL tài liệu PDF kỹ thuật](../README.md) / **OCR và trích xuất ảnh con**

Giải pháp này dùng **mô hình mã nguồn mở (open-source model)** trên máy có một GPU khoảng **64 GB VRAM** để nhận dạng ký tự quang học (*Optical Character Recognition, OCR*), phân tích bố cục, tách hình/biểu đồ/sơ đồ con và tạo dữ liệu có tọa độ cho **Retrieval-Augmented Generation (RAG)**.

Phạm vi tài liệu gồm:

- hướng dẫn sử dụng, vận hành, bảo trì, sửa chữa và xử lý sự cố thiết bị;
- service manual, catalogue phụ tùng, sơ đồ điện/khí nén, exploded view;
- biên bản kiểm tra có bảng, checkbox, chữ ký hoặc ghi chú;
- đề nghị cấp/xuất vật tư có mã vật tư, số lượng, đơn vị và luồng phê duyệt;
- PDF số có text/vector, PDF scan và PDF lai chứa cả hai;
- quy mô từ khoảng 50.000 đến 1.000.000 PDF.

> **Khuyến nghị cốt lõi:** không chạy một VLM trên toàn bộ PDF. Dùng router theo trang gồm đường PDF số, đường OCR/layout và đường VLM fallback. Với ảnh con, ưu tiên lấy ảnh nhúng gốc; chỉ crop ảnh render khi hình nằm trong trang scan hoặc được tạo bởi vector drawing.

## Điều hướng

- **Node cha:** [ETL tài liệu PDF kỹ thuật cho RAG](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** Chưa có.
- **Khái niệm sau trong cùng nhánh:** Chưa có.

## Kiến trúc đề xuất

```text
Object Storage: PDF gốc bất biến
        │
        ▼
CPU preflight: checksum, repair, page count, text/image/vector coverage
        │
        ▼
Page Router
  ├─ A. PDF số tốt
  │    ├─ extract text + geometry
  │    ├─ extract embedded raster image lossless
  │    └─ detect vector/figure region khi cần
  │
  ├─ B. Trang scan hoặc glyph lỗi
  │    ├─ render 250–400 DPI
  │    ├─ rotate/deskew/unwarp/denoise
  │    ├─ layout detection
  │    ├─ OCR text theo block
  │    └─ crop figure/table/chart/caption
  │
  └─ C. Trang khó hoặc confidence thấp
       ├─ PaddleOCR-VL hoặc MinerU hybrid/VLM
       ├─ VLM mô tả hình kỹ thuật có kiểm soát
       └─ human review cho dữ liệu quan trọng
        │
        ▼
Canonical JSON: page/block/table/figure/field/bbox/provenance
        │
        ├─ text chunks → BM25 + vector index
        ├─ figure crops → object storage + multimodal index
        └─ form fields → structured database/API
```

### Stack mặc định khuyến nghị

| Tầng | Công nghệ/model | Vai trò |
| --- | --- | --- |
| PDF preflight và repair | `pikepdf`, `pypdf` | Mở/repair PDF, checksum, metadata, trang, lấy embedded image. |
| Render permissive-license | `pypdfium2` | Render trang hoặc vùng crop ở DPI yêu cầu. |
| PDF/vector nâng cao, tùy license | `PyMuPDF` | Text/bbox, xref image, image placement, vector paths và clip rendering. |
| OCR tiếng Việt/Anh | `PaddleOCR`, ưu tiên benchmark PP-OCRv5 `lang=vi` và PP-OCRv6 | Detect/recognize text, giữ polygon và confidence. |
| Layout/table/figure | `PP-StructureV3`, `PP-DocLayoutV2` hoặc model layout trong PaddleOCR | Phân loại heading, text, table, image, figure, caption, formula và reading order. |
| Document VLM fallback | `PaddleOCR-VL-1.6` 0.9B | Trang méo, layout khó, bảng/công thức/biểu đồ khó; không chạy mặc định toàn corpus. |
| All-in-one thay thế | `MinerU` pipeline/hybrid | Markdown/JSON, image/table/formula, reading order; dùng khi benchmark tốt hơn cho corpus. |
| Mô tả hình kỹ thuật tùy chọn | `Qwen3-VL-8B-Instruct` | Sinh mô tả, tag và quan hệ cho figure crop; không dùng thay OCR/source text. |
| Document model/integration tùy chọn | `Docling` | Unified document model, export figure/table, chunking và integration RAG. |
| Queue/state | RabbitMQ/Kafka/SQS/PubSub + PostgreSQL | Fan-out theo page, retry, checkpoint, version và DLQ. |
| Storage | S3/GCS/Azure Blob/MinIO | PDF gốc, page render, crop, raw output và canonical output. |

PaddleOCR cung cấp pipeline OCR, PP-StructureV3, document VLM, high-performance inference và parallel inference; PP-OCRv5 có model Latin hỗ trợ tiếng Việt. PP-StructureV3 giữ coordinate chi tiết hơn dòng PaddleOCR-VL, phù hợp làm đường chính cho RAG cần citation theo bbox. [PaddleOCR][paddleocr] [PP-OCRv5 multilingual][ppocr-v5-multi] [PP-StructureV3][pp-structure] [PaddleOCR parallel inference][paddle-parallel]

---

## A. 5 câu hỏi cốt lõi

### 1. Kiến trúc nào phù hợp nhất cho một GPU 64 GB và 50.000–1.000.000 PDF?

**Trả lời:** Dùng kiến trúc **page-level routed pipeline**, không dùng một model duy nhất cho mọi trang.

Ba đường xử lý:

1. **Native PDF fast path:** PDF có text số tốt đi qua parser CPU; ảnh raster nhúng được lấy trực tiếp từ PDF mà không OCR hoặc render lại.
2. **OCR/layout path:** trang scan, text layer rỗng hoặc glyph hỏng được render và đưa qua preprocessing, layout detection và OCR.
3. **Hard-page fallback:** trang mờ, méo, viết tay, biểu đồ/sơ đồ phức tạp hoặc output không vượt quality gate mới dùng PaddleOCR-VL, MinerU VLM/hybrid hoặc human review.

GPU 64 GB nên dành cho batch OCR/layout phần lớn thời gian. VLM fallback chạy ở queue ưu tiên thấp hoặc theo cửa sổ riêng để tránh tranh GPU với đường chính. Nếu cần ingest liên tục và đồng thời xử lý VLM, hai GPU tách vai trò ổn định hơn một GPU dùng chung.

**Áp dụng khi nào?**

- Corpus hỗn hợp giữa PDF số, scan và hybrid PDF.
- File có số trang khác nhau và chỉ một phần trang thực sự cần OCR.
- Backfill lớn rồi tiếp tục ingest tài liệu mới hằng ngày.
- Cần kiểm soát thời gian hoàn thành, GPU utilization và khả năng chạy lại.

**Bài toán giải quyết:**

- Tránh dùng GPU/VLM cho trang đã có text tốt.
- Retry một trang thay vì chạy lại PDF hàng trăm trang.
- Scale theo số trang và độ khó, không theo số file.
- Cô lập trang lỗi và thay model mà không phá downstream.

**So sánh:**

| Kiến trúc | Điểm mạnh | Điểm yếu | Chọn khi |
| --- | --- | --- | --- |
| Một OCR cho mọi trang | Dễ triển khai. | Lãng phí, mất text/vector tốt, yếu với layout. | POC nhỏ, gần như toàn bộ là scan đơn giản. |
| Một document VLM cho mọi trang | Linh hoạt với trang khó. | Chậm, output ít deterministic, khó kiểm soát chi phí và bbox. | Dataset nhỏ hoặc nghiên cứu. |
| Router theo file | Tốt hơn one-model. | Sai với PDF lai. | File tương đối đồng nhất. |
| Router theo trang | Tối ưu chất lượng/throughput. | Cần manifest, merge và observability. | Khuyến nghị từ hàng chục nghìn đến hàng triệu PDF. |

---

### 2. Nên chọn model open source nào cho OCR, layout và trang khó?

**Trả lời:** Stack mặc định nên là **PaddleOCR + PP-StructureV3**, thêm **PaddleOCR-VL** hoặc **MinerU** làm fallback.

#### OCR chính

- Với tiếng Việt và tiếng Anh trộn lẫn, bắt đầu benchmark bằng PP-OCRv5 multilingual/Latin với `lang=vi` vì danh sách chính thức có tiếng Việt.
- Benchmark thêm PP-OCRv6 unified model trên tập tài liệu thực tế, nhất là mã thiết bị, ký hiệu công nghiệp, màn hình số và chữ nhỏ.
- Giữ polygon/bbox và confidence từng dòng; không chỉ lấy text thuần.

#### Layout và crop region

- Dùng PP-StructureV3 hoặc PP-DocLayoutV2 để tìm `text`, `title`, `table`, `image`, `figure`, `chart`, `formula`, `figure_caption`, `table_caption`, header/footer và reading order.
- Chọn model lớn khi ưu tiên độ chính xác, model M/S khi throughput quan trọng; quyết định bằng benchmark trên trang thực tế.

#### Trang khó

- PaddleOCR-VL-1.6 là document VLM 0.9B, phù hợp fallback cục bộ vì nhỏ hơn general-purpose VLM và tập trung vào document parsing.
- MinerU 3.3 có pipeline, hybrid và VLM backend, có thể trích image, table, formula và JSON/Markdown; dùng khi kết quả benchmark tổng thể tốt hơn.
- Qwen3-VL-8B chỉ nên mô tả figure/diagram crop, sinh tag hoặc kiểm tra quan hệ; text OCR và mã kỹ thuật vẫn lấy từ engine deterministic.

PaddleOCR hiện có PP-OCRv6, PP-StructureV3 và PaddleOCR-VL-1.6; tài liệu chính thức cũng hỗ trợ high-performance inference và nhiều instance. MinerU 3.3 cung cấp pipeline/hybrid/VLM và image analysis ở chế độ phù hợp. [PaddleOCR repository][paddleocr] [PaddleOCR OCR pipeline][paddle-ocr-pipeline] [PP-StructureV3 usage][pp-structure-usage] [MinerU][mineru]

**Áp dụng khi nào?**

- Manual nhiều cột, bảng, caption và hình minh họa.
- Scan tiếng Việt có chữ nhỏ, xoay hoặc méo.
- Biên bản/phiếu có layout ổn định lẫn layout tự do.
- Cần chạy offline trong mạng nội bộ.

**Bài toán giải quyết:** cân bằng OCR accuracy, layout fidelity, bbox, throughput và khả năng vận hành production.

**So sánh model/tool:**

| Lựa chọn | Điểm mạnh | Giới hạn | Vai trò khuyến nghị |
| --- | --- | --- | --- |
| PaddleOCR + PP-StructureV3 | OCR đa ngôn ngữ, bbox chi tiết, layout/table, deployment tốt, Apache-2.0. | Cần ghép pipeline và tune theo corpus. | Đường chính. |
| PaddleOCR-VL | Document parsing mạnh, model nhỏ 0.9B, xử lý trang khó. | Ít coordinate chi tiết hơn PP-StructureV3; output sinh chuỗi cần validate. | Fallback 1–10% trang khó. |
| MinerU | All-in-one, image/table/formula, reading order, Markdown/JSON. | License có điều kiện bổ sung; cần benchmark và khóa version. | Thay thế/all-in-one hoặc fallback. |
| Docling | Document model, export figure/table, integration và GPU batch. | GPU OCR phụ thuộc backend; model license cần xem riêng. | Orchestrator/document model. |
| Surya | OCR/layout/table trong một model, nhiều ngôn ngữ. | Code/model license hạn chế hơn cho doanh nghiệp. | Chỉ khi pháp lý chấp nhận và benchmark thắng. |
| Tesseract/OCRmyPDF | Ổn định, tạo searchable PDF, CPU-friendly. | Yếu hơn với layout phức tạp và crop figure. | Baseline/archival, không phải đường GPU chính. |

---

### 3. Làm sao tách đúng “ảnh con” trong PDF vector và PDF scan?

**Trả lời:** Phải phân biệt **embedded raster image**, **vector drawing** và **scan page**.

#### Trường hợp A — Embedded raster image trong PDF số

- Lấy binary image gốc từ object/XObject bằng `pypdf`, `pikepdf` hoặc `PyMuPDF`.
- Không render cả trang rồi crop nếu có thể lấy ảnh gốc, vì render làm mất chất lượng và tăng dung lượng.
- Ghi lại mỗi lần ảnh xuất hiện trên trang: `page`, `xref/objectId`, `bbox`, transformation matrix, width/height và hash.
- Một image object có thể được dùng nhiều lần; deduplicate theo hash nhưng vẫn giữ nhiều placement.

`pypdf` hỗ trợ duyệt `page.images`; PyMuPDF có `Page.get_images()`, `Document.extract_image()` và thông tin vị trí ảnh. [pypdf image extraction][pypdf-images] [PyMuPDF image extraction][pymupdf-images]

#### Trường hợp B — Vector drawing/sơ đồ vector

- Vector line art không phải image object nên không thể “extract ảnh gốc” như JPEG/PNG.
- Dùng layout detector xác định vùng figure/diagram, kết hợp `get_drawings()` hoặc drawing clusters để kiểm tra vùng có vector.
- Render riêng rectangle đó bằng clip ở 300–400 DPI; lưu thêm bbox theo đơn vị PDF để mở lại đúng vị trí.
- Khi cần bảo toàn vector, lưu page SVG/path data như một artifact phụ, nhưng crop hiển thị cho RAG thường vẫn dùng PNG/WebP.

PyMuPDF mô tả vector drawing dưới dạng paths; tài liệu cũng nêu rằng line art không thể lấy trực tiếp như image và nên render vùng liên quan bằng clip. [PyMuPDF drawings][pymupdf-drawings] [PyMuPDF vector FAQ][pymupdf-vector-faq]

#### Trường hợp C — Trang scan

1. Render trang ở DPI phù hợp.
2. Layout model trả bbox của `image`, `figure`, `chart`, `table`, caption và text.
3. Merge box chồng lấn hoặc các mảnh thuộc cùng figure.
4. Mở rộng padding khoảng 1–3% kích thước vùng để tránh cắt mất viền/callout.
5. Crop từ page render gốc, không crop từ ảnh đã resize cho model.
6. OCR riêng text trong crop và vùng caption lân cận.
7. Lưu crop URI, bbox pixel, bbox PDF, page DPI, caption relation và content hash.

**Áp dụng khi nào?** Manual có exploded view, sơ đồ điện, ảnh thao tác, biểu đồ và hình kèm callout.

**Bài toán giải quyết:** tránh ảnh mờ, mất callout, crop nhầm table, lặp ảnh và mất liên kết giữa hình với chú thích.

**So sánh:**

| Cách | Chất lượng | Chi phí | Khi dùng |
| --- | --- | --- | --- |
| Extract embedded image | Tốt nhất, giữ bytes gốc. | Rất thấp. | PDF số có raster XObject. |
| Render clip vector region | Phụ thuộc DPI nhưng giữ hình đầy đủ. | Trung bình. | Sơ đồ/vector drawing. |
| Layout detect + crop scan | Cần model và tuning. | GPU/CPU. | Scan hoặc screenshot. |
| VLM tự tìm và mô tả hình | Linh hoạt nhưng khó đảm bảo bbox. | Cao. | Fallback, không thay detector chính. |

---

### 4. Output trung gian cần thiết kế thế nào để vừa dùng cho RAG vừa audit được?

**Trả lời:** Dùng một **Canonical Document Model** độc lập với model cụ thể. Raw output của PaddleOCR, MinerU hoặc Docling được giữ để debug nhưng downstream chỉ phụ thuộc canonical schema.

Ví dụ figure record:

```json
{
  "figureId": "doc123-p004-fig02",
  "documentId": "doc123",
  "documentVersion": "sha256:...",
  "pageNumber": 4,
  "kind": "technical_diagram",
  "sourceType": "vector_rendered_crop",
  "bboxPdf": [72.4, 154.0, 522.1, 611.8],
  "bboxPixel": [302, 642, 2175, 2548],
  "renderDpi": 300,
  "imageUri": "s3://rag-assets/doc123/v1/p004/fig02.webp",
  "originalImageUri": null,
  "captionText": "Hình 4-2. Cấu tạo cụm bơm",
  "captionBlockId": "p004-b019",
  "ocrTextRaw": "1 Motor 2 Khop noi 3 Bom",
  "ocrTextNormalized": "1 Motor; 2 Khớp nối; 3 Bơm",
  "nearbyTextBlockIds": ["p004-b017", "p004-b020"],
  "contentHash": "sha256:...",
  "engine": "pp-structure-v3",
  "modelVersion": "locked-version",
  "configHash": "sha256:...",
  "confidence": 0.93,
  "warnings": []
}
```

Các object chính:

- `Document`: source URI, checksum, business identity, version, ACL và hiệu lực.
- `Page`: dimensions, rotation, render artifacts, route và status.
- `Block`: type, text, polygon/bbox, reading order, section path.
- `Table`: grid/cell/span/header và text serialization.
- `Figure`: image/crop, caption, OCR text, nearby blocks và provenance.
- `Field`: key/value/normalized value/confidence cho biểu mẫu.
- `Relation`: figure–caption, callout–legend, reference–target và section hierarchy.

**Áp dụng khi nào?** Luôn áp dụng khi có nhiều parser/model, cần re-index, audit, highlight PDF hoặc multimodal RAG.

**Bài toán giải quyết:**

- Không khóa downstream vào schema của một model.
- Có thể so A/B model và rollback.
- Mở đúng trang/bbox khi trả citation.
- Truy ngược crop về PDF gốc.
- Tách dữ liệu RAG khỏi dữ liệu giao dịch.

**So sánh:** Markdown dễ dùng cho prototype nhưng không đủ geometry, relation, lineage và field-level quality. Docling cũng dùng unified document representation có text, tables, pictures, hierarchy, bounding boxes và provenance, là một tham khảo tốt khi thiết kế canonical model. [Docling Document][docling-document]

---

### 5. Một GPU 64 GB có đủ không và sizing throughput thế nào?

**Trả lời:** 64 GB VRAM đủ để chạy nhiều instance OCR/layout hoặc một document/general VLM cỡ nhỏ–trung bình, nhưng **không thể kết luận đủ cho 1 triệu PDF nếu chưa biết tổng số trang, tỷ lệ route vào GPU và deadline**.

Sizing theo công thức:

```text
total_gpu_pages = total_pages × gpu_route_ratio
backfill_seconds = total_gpu_pages / (measured_pages_per_second × target_utilization)
```

Ví dụ 1.000.000 PDF × 20 trang = 20.000.000 trang. Nếu chỉ 30% trang phải qua GPU thì còn 6.000.000 GPU-page. Với throughput end-to-end đo được 10 trang/giây và target utilization 80%, backfill lý thuyết khoảng 8,7 ngày; nếu chỉ đạt 2 trang/giây thì khoảng 43 ngày. Đây chỉ là capacity model, phải thay bằng benchmark của chính dữ liệu.

#### Cách dùng 64 GB VRAM

- Bắt đầu với 2 instance PP-Structure/OCR, tăng dần batch và instance đến khi GPU utilization cao nhưng p95 latency, OOM và queue stability vẫn đạt yêu cầu.
- Không tạo quá nhiều process chỉ vì còn VRAM; PDF render, image decode, table processing, temporary disk và Python scheduling có thể trở thành bottleneck.
- Dùng FP16/TensorRT/high-performance inference khi model và môi trường hỗ trợ.
- VLM fallback nên có concurrency riêng và giới hạn pixel/token; Qwen3-VL-8B BF16 có khoảng 16 GB trọng số trước overhead, trong khi model 32B BF16 gần 64 GB chỉ riêng trọng số nên không phù hợp co-host thoải mái trên một GPU 64 GB.
- Đối với backfill lớn, ưu tiên OCR/layout throughput; gom low-confidence pages để chạy VLM theo batch sau.

PaddleOCR hỗ trợ high-performance inference với Paddle Inference, ONNX Runtime, OpenVINO hoặc TensorRT tùy model, đồng thời có ví dụ chạy nhiều instance/device. Docling cũng cho phép tăng OCR/layout batch size và page concurrency trên GPU. [PaddleOCR HPI][paddle-hpi] [PaddleOCR parallel inference][paddle-parallel] [Docling GPU][docling-gpu]

**Máy cân đối gợi ý:** 32–64 CPU cores, 128–256 GB RAM, NVMe scratch đủ chứa page render tạm, object storage riêng và network ổn định. CPU/render/I/O thường phải được scale cùng GPU.

**Áp dụng khi nào?** Capacity planning, quyết định một hay nhiều GPU, lập SLA backfill và continuous ingest.

**Bài toán giải quyết:** tránh mua GPU theo số file, tránh OOM do VLM và phát hiện bottleneck ngoài GPU.

---

## B. 8 câu hỏi phổ biến mở rộng

### 6. Nên render trang ở DPI bao nhiêu và preprocessing thế nào?

**Trả lời:** Không dùng một DPI cố định cho mọi tài liệu.

- 200–250 DPI: scan rõ, chữ lớn, throughput ưu tiên.
- 300 DPI: mặc định tốt cho phần lớn manual và form in.
- 350–400 DPI: chữ nhỏ, part number, sơ đồ/callout dày.
- Trên 400 DPI: chỉ route cục bộ vì ảnh lớn làm tăng decode, VRAM, thời gian và storage.

Preprocessing theo thứ tự:

1. orientation detection;
2. crop margin/black border;
3. deskew;
4. perspective/unwarp khi chụp hoặc scan cong;
5. contrast/illumination normalization;
6. denoise nhẹ;
7. model inference;
8. fallback ở DPI cao hơn nếu confidence/rule không đạt.

Không làm threshold đen trắng quá mạnh cho sơ đồ màu, dấu mộc, chữ ký hoặc nền gradient. Giữ page render gốc và bản preprocess riêng để audit.

**Áp dụng khi nào?** Scan mờ, xoay, cong, photocopy nhiều lần hoặc chữ kỹ thuật nhỏ.

**Bài toán giải quyết:** OCR sai `0/O`, `1/I`, mất dấu tiếng Việt, mất đường mảnh và tăng chi phí do render quá lớn.

**So sánh:** tăng DPI thường giúp chữ nhỏ nhưng không sửa được blur/motion, compression artifact hoặc trang bị cắt; lúc đó cần preprocessing, model khác hoặc scan lại.

---

### 7. Làm sao OCR chính xác mã thiết bị, mã lỗi, part number và số liệu?

**Trả lời:** Không để LLM tự “sửa đẹp” text kỹ thuật. Dùng hai lớp `raw` và `normalized` cùng rule có thể audit.

- Giữ OCR raw, polygon, confidence và crop của token.
- Xây dictionary từ master data: equipment model, part number, error code, unit và tên vật tư.
- Dùng regex/finite-state rule cho pattern như `AB-1234-X`, `E07`, `220 V`, `35 N·m`.
- Candidate correction chỉ được chấp nhận khi edit distance nhỏ và khớp dictionary/context.
- Index cả raw và normalized; exact code ưu tiên BM25/keyword field.
- Low-confidence critical code phải chạy second OCR engine hoặc human review.

**Áp dụng khi nào?** Troubleshooting, phụ tùng, torque/specification, mã vật tư và số serial.

**Bài toán giải quyết:** semantic embedding không chịu được sai một ký tự; `E01` và `E07` có thể dẫn tới quy trình hoàn toàn khác.

**So sánh:** language-model correction tăng độ trôi chảy nhưng có thể hallucinate; constrained normalization an toàn và regression-test được.

---

### 8. Làm sao nối figure với caption, legend và đoạn văn liên quan?

**Trả lời:** Dùng relation extraction dựa trên geometry trước, semantic sau.

1. Lấy bbox figure và tất cả caption candidate gần trên/dưới.
2. Ưu tiên label `figure_caption` từ layout model.
3. Kiểm tra pattern `Hình/Figure/Fig.` và số thứ tự.
4. Chọn candidate theo khoảng cách, overlap trục ngang và reading order.
5. Tìm tham chiếu trong text như “xem Hình 4-2”.
6. OCR callout trong figure (`1`, `2`, `A`, `B`) và tìm legend có cùng token.
7. Lưu relation có confidence; không merge vĩnh viễn nếu mơ hồ.

Với figure nhiều mảnh, merge bbox dựa trên khoảng cách, common caption và vùng nền; tránh merge hai hình độc lập chỉ vì gần nhau.

**Áp dụng khi nào?** Manual kỹ thuật, exploded view, wiring diagram và ảnh từng bước thao tác.

**Bài toán giải quyết:** RAG retrieve được ảnh nhưng thiếu chú thích, hoặc gắn sai caption khiến mô tả hình sai.

**So sánh:** nearest-caption rule đơn giản đủ cho layout sạch; graph relation + reference resolution cần cho manual phức tạp.

---

### 9. Biên bản kiểm tra và phiếu cấp/xuất vật tư xử lý khác manual thế nào?

**Trả lời:** Manual chủ yếu tạo text/figure chunks cho RAG; biểu mẫu phải thêm nhánh **Key Information Extraction (KIE)** và structured database.

Pipeline form:

```text
form classification
→ template/version detection
→ layout + OCR
→ key/value/checkbox/table extraction
→ normalization + business validation
→ human review nếu critical/low confidence
→ database/API source of truth
→ text summary/chunks cho RAG
```

Field ví dụ:

```text
request_id, request_date, department, equipment_id,
item_code, item_name, quantity, unit, reason,
requester, approver, approval_status,
source_page, source_bbox, confidence
```

Checkbox, chữ ký và handwriting nên có detector/model hoặc rule riêng. Chữ ký chỉ nên phát hiện có/không và bbox nếu nghiệp vụ cho phép; không suy luận danh tính từ hình chữ ký.

**Áp dụng khi nào?** Phiếu vật tư, checklist bảo trì, biên bản kiểm tra, nghiệm thu và nhật ký vận hành.

**Bài toán giải quyết:** vector database không đảm bảo unique key, transaction, approval state, tổng hợp số lượng hoặc audit field.

**So sánh:**

| RAG index | Structured DB/API |
| --- | --- |
| Tra cứu gần nghĩa, hỏi đáp, lịch sử. | Dữ liệu nghiệp vụ chuẩn hóa và có constraint. |
| Chunk có thể lặp hoặc stale. | Có version/status/source of truth. |
| Không phù hợp làm tồn kho/phê duyệt. | Phù hợp workflow và báo cáo. |

---

### 10. Làm sao scale queue và worker từ 50.000 lên 1.000.000 PDF?

**Trả lời:** Mỗi stage phải idempotent và ghi checkpoint theo page/range.

Queue gợi ý:

```text
inventory
native-parse
page-render
ocr-layout-gpu
figure-crop
vlm-fallback
form-extract
normalize
quality-gate
chunk-index
human-review
DLQ
```

Work item tối thiểu:

```json
{
  "documentVersion": "sha256:...",
  "pageNumber": 18,
  "stage": "ocr-layout-gpu",
  "route": "scan_complex",
  "priority": 5,
  "attempt": 1,
  "configHash": "sha256:..."
}
```

Nguyên tắc:

- scheduler chỉ phát manifest/batch, không giữ hàng triệu task trong memory;
- queue partition theo route/priority/tenant khi cần;
- worker lấy micro-batch các trang có kích thước tương tự;
- backpressure theo queue age, GPU utilization và temporary disk;
- retry exponential backoff cho lỗi tạm thời;
- corrupt/encrypted/unsupported vào DLQ, không retry vô hạn;
- output bất biến theo extraction version;
- publish bằng index alias sau quality gate.

**Áp dụng khi nào?** Backfill từ hàng triệu đến hàng chục triệu trang và ingest tăng dần.

**Bài toán giải quyết:** retry cục bộ, autoscale, tránh duplicate và biết chính xác trang nào đang lỗi.

**So sánh:** Airflow/Argo phù hợp orchestration theo batch/job; queue worker phù hợp fan-out theo trang. Không nên tạo một DAG task cho mỗi trang ở quy mô hàng chục triệu trang.

---

### 11. Quality gate và bộ benchmark cần đo gì?

**Trả lời:** Đánh giá theo từng thành phần và downstream RAG, không chỉ nhìn confidence của model.

| Thành phần | Metric |
| --- | --- |
| OCR text | CER, WER, exact-match cho mã/giá trị. |
| Layout | element precision/recall, reading-order accuracy. |
| Figure crop | detection precision/recall, bbox IoU, crop completeness. |
| Caption relation | caption-link precision/recall. |
| Table | cell structure, header association, TEDS hoặc task metric tương đương. |
| Form | field precision/recall/F1, normalized-value exact match. |
| Provenance | page/bbox validity, mở đúng source. |
| RAG | Recall@k, citation correctness, grounded answer/abstention. |

Gold set phải stratified theo:

- PDF digital, scan sạch, scan mờ và hybrid;
- tiếng Việt, Anh và song ngữ;
- manual, troubleshooting, catalogue, sơ đồ, form;
- model/hãng thiết bị và template phiên bản;
- chữ nhỏ, nhiều cột, bảng dài, callout và handwriting.

Tạo ít nhất vài trăm đến vài nghìn trang đại diện trước khi chốt model. Critical field như part number, safety warning, quantity và unit có threshold/human review riêng.

**Áp dụng khi nào?** Trước pilot, trước mỗi model/config upgrade và trước publish index mới.

**Bài toán giải quyết:** phát hiện regression sớm và phân biệt lỗi OCR, layout, crop, chunking hay retrieval.

**So sánh:** benchmark công khai hữu ích để shortlist; benchmark nội bộ mới quyết định model production vì layout, ngôn ngữ và scan quality khác nhau.

---

### 12. License open source nào cần lưu ý khi triển khai doanh nghiệp?

**Trả lời:** “Có source code” không đồng nghĩa được dùng tự do trong mọi sản phẩm. Phải kiểm tra riêng code, model weights và dependency.

| Thành phần | License/ghi chú chính | Khuyến nghị |
| --- | --- | --- |
| PaddleOCR | Apache-2.0. | Thuận lợi cho commercial deployment. |
| Docling code | MIT; model riêng có license riêng. | Kiểm tra model catalog trước khi đóng gói. |
| `pypdfium2` | Apache-2.0 hoặc BSD-3-Clause; dependency PDFium có license kèm theo. | Tốt cho render permissive-license. |
| `pikepdf` | MPL-2.0. | Có thể dùng trong larger work; công bố sửa đổi pikepdf khi điều khoản yêu cầu. |
| PyMuPDF | AGPL hoặc commercial license tùy cách dùng/phân phối. | Review pháp lý trước khi dùng trong closed-source service. |
| DocLayout-YOLO | AGPL-3.0. | Không chọn mặc định nếu policy không chấp nhận copyleft mạnh. |
| Surya | Code GPL và model weights có điều kiện sử dụng. | Review kỹ trước commercial use. |
| MinerU | MinerU Open Source License dựa trên Apache 2.0 nhưng có điều kiện bổ sung. | Đọc đúng version license và legal review. |

Các repo chính thức công bố PaddleOCR là Apache-2.0, Docling là MIT, `pypdfium2` dùng Apache/BSD, DocLayout-YOLO là AGPL và Surya có điều kiện riêng cho code/weights. [PaddleOCR license][paddleocr] [Docling license][docling-license] [pypdfium2 license][pypdfium2] [DocLayout-YOLO][doclayout-yolo] [Surya][surya] [MinerU][mineru]

**Áp dụng khi nào?** Trước khi đóng Docker image, phân phối phần mềm, cung cấp SaaS hoặc dùng model với dữ liệu khách hàng.

**Bài toán giải quyết:** tránh thay stack muộn vì xung đột license và đảm bảo Software Bill of Materials (SBOM) đầy đủ.

**So sánh:** Nếu doanh nghiệp ưu tiên permissive license, dùng `pypdf`/`pypdfium2`/`pikepdf` + PaddleOCR/Docling; chỉ dùng component AGPL/custom-license khi legal chấp nhận.

---

### 13. Lộ trình triển khai thực tế nên chia giai đoạn thế nào?

**Trả lời:** Triển khai theo bốn giai đoạn và chỉ mở rộng sau khi có benchmark.

#### Giai đoạn 1 — Inventory và gold set

- Lấy 1.000–5.000 file đại diện.
- Thống kê trang/file, digital/scan/hybrid, image/vector ratio và ngôn ngữ.
- Chọn 20–50 document classes chính.
- Gắn nhãn 500–2.000 trang cho OCR/layout/figure/form.
- Chuẩn bị 100–300 câu hỏi retrieval/citation.

#### Giai đoạn 2 — Pilot một GPU

- CPU preflight + page router.
- `pypdf/pypdfium2` hoặc PyMuPDF theo policy license.
- PaddleOCR + PP-StructureV3 đường chính.
- Crop service và canonical JSON.
- Một fallback bằng PaddleOCR-VL hoặc MinerU.
- Dashboard throughput, error, GPU, queue age và quality.

#### Giai đoạn 3 — Production backfill

- Queue/worker theo stage.
- Batch theo page size/complexity.
- Checkpoint, idempotency và DLQ.
- Model/config versioning.
- Human-review queue.
- Index version + alias + rollback.

#### Giai đoạn 4 — Continuous ingestion

- Event từ DMS/object storage.
- Incremental extract/chunk/index.
- Regression suite khi đổi model.
- Theo dõi drift theo hãng/template/scan source.
- Retention, ACL và document-effective status.

**Definition of Done tối thiểu:**

- [ ] PDF gốc bất biến, checksum và version rõ ràng.
- [ ] Router phân biệt native, scan, vector và hard page.
- [ ] Embedded image được lấy lossless khi có thể.
- [ ] Vector/scan figure được crop có bbox PDF và pixel.
- [ ] OCR giữ raw + normalized + confidence.
- [ ] Figure liên kết caption/nearby text.
- [ ] Form critical field có validation/human review.
- [ ] Retry idempotent, có checkpoint và DLQ.
- [ ] Có gold set và regression benchmark.
- [ ] Có exact lexical index cho code/part number.
- [ ] Citation mở đúng PDF/page/bbox.
- [ ] License và SBOM đã được duyệt.

**Áp dụng khi nào?** Khi chuyển từ thử nghiệm sang hệ thống production có dữ liệu nội bộ và SLA.

**Bài toán giải quyết:** giảm rủi ro chọn model theo demo đẹp nhưng không scale hoặc không đáp ứng chất lượng nghiệp vụ.

**So sánh:** Big-bang ingest toàn corpus tạo nợ kỹ thuật và khó biết root cause; pilot có gold set giúp chốt route, model, capacity và schema trước khi tiêu tốn tài nguyên lớn.

---

## Cấu hình vận hành khuyến nghị cho một GPU 64 GB

```text
CPU pool
  inventory-worker          2–4 processes
  native-pdf-worker         4–16 processes
  render-worker             4–12 processes, giới hạn RAM/NVMe
  normalize/index-worker    tùy throughput downstream

GPU pool — chế độ backfill
  paddle-ocr-layout         bắt đầu 2 instances
  batch size                tune theo page dimension bucket
  precision                 FP16 khi validation không giảm
  VLM fallback              queue lại, chạy theo batch/window

GPU pool — chế độ continuous
  OCR/layout                quota GPU chính
  VLM fallback              concurrency thấp, pixel/token limit
  admission control         theo queue priority và GPU memory
```

Không cố định số process/batch từ tài liệu mẫu. Chạy load test với 3 bucket: A4 text, scan 300 DPI và trang sơ đồ/bảng phức tạp; tăng từng tham số một và đo end-to-end accepted pages/second.

## Quy tắc lưu ảnh crop

- Giữ bản gốc khi extract được embedded image.
- Crop render lưu PNG cho line art/text nhỏ hoặc WebP lossless/quality cao theo benchmark.
- Không dùng JPEG chất lượng thấp cho sơ đồ, chữ nhỏ và line art.
- Tên object theo `documentVersion/page/figureId`, không theo tên file người dùng.
- Lưu `contentHash` để deduplicate.
- Lưu thumbnail riêng; không dùng thumbnail làm nguồn OCR/RAG.
- Với crop có dữ liệu nhạy cảm, kế thừa ACL và retention của document.
- Không lưu mô tả VLM như ground truth; đánh dấu `generatedDescription` cùng model/prompt/version.

## Nguồn tham khảo

- [PaddleOCR repository, model families, release và license][paddleocr]
- [PP-OCRv5 multilingual, gồm tiếng Việt][ppocr-v5-multi]
- [PP-StructureV3 introduction][pp-structure]
- [PP-StructureV3 usage và model layout][pp-structure-usage]
- [PaddleOCR general OCR pipeline][paddle-ocr-pipeline]
- [PaddleOCR high-performance inference][paddle-hpi]
- [PaddleOCR parallel inference][paddle-parallel]
- [MinerU document parsing][mineru]
- [Docling GPU support][docling-gpu]
- [Docling export figures][docling-figures]
- [Docling unified document model][docling-document]
- [Docling license][docling-license]
- [pypdf extract images][pypdf-images]
- [pypdfium2 và license][pypdfium2]
- [pikepdf PDF repair/manipulation][pikepdf]
- [PyMuPDF image extraction][pymupdf-images]
- [PyMuPDF vector drawings][pymupdf-drawings]
- [PyMuPDF vector extraction FAQ][pymupdf-vector-faq]
- [DocLayout-YOLO][doclayout-yolo]
- [Surya OCR/layout][surya]
- [Qwen3-VL][qwen3-vl]

[paddleocr]: https://github.com/PaddlePaddle/PaddleOCR
[ppocr-v5-multi]: https://github.com/PaddlePaddle/PaddleOCR/blob/main/docs/version3.x/algorithm/PP-OCRv5/PP-OCRv5_multi_languages.en.md
[pp-structure]: https://www.paddleocr.ai/latest/en/version3.x/algorithm/PP-StructureV3/PP-StructureV3.html
[pp-structure-usage]: https://www.paddleocr.ai/latest/en/version3.x/pipeline_usage/PP-StructureV3.html
[paddle-ocr-pipeline]: https://www.paddleocr.ai/latest/en/version3.x/pipeline_usage/OCR.html
[paddle-hpi]: https://www.paddleocr.ai/main/en/version3.x/inference_deployment/local_inference/high_performance_inference.html
[paddle-parallel]: https://www.paddleocr.ai/latest/en/version3.x/inference_deployment/local_inference/parallel_inference.html
[mineru]: https://github.com/opendatalab/MinerU
[docling-gpu]: https://docling-project.github.io/docling/usage/gpu/
[docling-figures]: https://docling-project.github.io/docling/_generated/examples/export_figures/
[docling-document]: https://docling-project.github.io/docling/concepts/docling_document/
[docling-license]: https://github.com/docling-project/docling/blob/main/LICENSE
[pypdf-images]: https://pypdf.readthedocs.io/en/stable/user/extract-images.html
[pypdfium2]: https://github.com/pypdfium2-team/pypdfium2
[pikepdf]: https://github.com/pikepdf/pikepdf
[pymupdf-images]: https://pymupdf.readthedocs.io/en/latest/recipes-images.html
[pymupdf-drawings]: https://pymupdf.readthedocs.io/en/latest/recipes-drawing-and-graphics.html
[pymupdf-vector-faq]: https://pymupdf.readthedocs.io/en/latest/faq/index.html
[doclayout-yolo]: https://github.com/opendatalab/DocLayout-YOLO
[surya]: https://github.com/datalab-to/surya
[qwen3-vl]: https://github.com/QwenLM/Qwen3-VL
