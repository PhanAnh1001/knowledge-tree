# Kiểm thử hồi quy đa phương thức (Multimodal Regression Engineering)

> [Knowledge Tree](../../../README.md) / [AI](../../README.md) / [AI đa phương thức](../README.md) / **Multimodal Regression Engineering**

**Multimodal Regression Engineering** trong tài liệu này là tên gọi thực hành cho việc xây dựng một hệ thống kiểm thử lặp lại nhằm phát hiện chất lượng suy giảm khi thay đổi mô hình, prompt, bộ tiền xử lý, OCR, parser, dữ liệu, hạ tầng hoặc logic ứng dụng có đầu vào/đầu ra gồm nhiều phương thức như văn bản, ảnh, âm thanh, video, PDF và dữ liệu có cấu trúc.

Đây chưa phải là một thuật ngữ chuẩn hóa độc lập được dùng thống nhất như *software regression testing* hay *Testing, Evaluation, Verification and Validation (TEVV)*. Vì vậy nên hiểu nó là giao điểm của:

- kiểm thử hồi quy phần mềm (*software regression testing*);
- đánh giá mô hình và hệ thống AI (*AI evaluation*);
- kiểm thử độ bền trước biến đổi dữ liệu (*robustness testing*);
- kiểm thử quan hệ giữa nhiều phương thức (*cross-modal alignment testing*);
- quan sát và kiểm soát phát hành (*observability and release gating*).

NIST nhấn mạnh rằng đánh giá AI cần có phép đo đáng tin cậy, dataset/testbed phù hợp ngữ cảnh, quy trình TEVV có thể lặp lại và tài liệu hóa. Các benchmark như MMMU, MM-Vet và OCRBench cho thấy không thể suy chất lượng đa phương thức chỉ từ một điểm tổng hợp hoặc chỉ từ khả năng xử lý văn bản. [NIST TEVV][nist-tevv] [NIST AI RMF][nist-ai-rmf] [MMMU][mmmu] [MM-Vet][mm-vet] [OCRBench][ocrbench]

```text
Thay đổi code/model/prompt/data
             │
             ▼
      Versioned test cases
 text + image + audio + video + PDF + metadata
             │
             ▼
   Deterministic checks ── task metrics ── model judge
             │                    │
             └──── human review ──┘
                       │
                       ▼
       Baseline comparison by slice
                       │
             pass / warn / block
                       │
              deploy + monitor
```

## Điều hướng

- **Node cha:** [AI đa phương thức (Multimodal AI)](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** Chưa có.
- **Khái niệm sau trong cùng nhánh:** Chưa có.

---

## A. 5 câu hỏi cốt lõi

### 1. Kiểm thử hồi quy đa phương thức là gì và khác kiểm thử hồi quy phần mềm thông thường như thế nào?

**Trả lời:** Kiểm thử hồi quy truyền thống xác nhận rằng một thay đổi không làm hỏng hành vi phần mềm đã biết: API vẫn trả đúng schema, thuật toán vẫn tính đúng kết quả, giao diện không vỡ và luồng nghiệp vụ vẫn chạy. Kiểm thử hồi quy đa phương thức mở rộng mục tiêu đó sang hệ thống AI có đầu vào hoặc đầu ra khó xác định bằng một giá trị chính xác duy nhất.

Ví dụ, khi đổi model vision-language trong pipeline PDF kỹ thuật, hệ thống có thể:

- OCR đúng chữ nhưng nối sai caption với hình;
- nhận đúng bảng nhưng đảo thứ tự hàng;
- trả lời đúng ý nhưng dẫn chứng sai vùng ảnh;
- đọc được trang rõ nhưng suy giảm mạnh với ảnh mờ hoặc xoay;
- cải thiện điểm trung bình nhưng làm hỏng nhóm tài liệu tiếng Việt hoặc sơ đồ điện;
- giữ chất lượng nhưng tăng gấp đôi chi phí và độ trễ.

Do đó, một regression suite đa phương thức phải kiểm tra đồng thời:

1. **Hợp đồng phần mềm:** schema, kiểu dữ liệu, lỗi, timeout, retry, idempotency.
2. **Chất lượng từng phương thức:** OCR, detection, transcription, captioning, video understanding.
3. **Quan hệ chéo phương thức:** câu trả lời có bám đúng ảnh, âm thanh hoặc vùng tài liệu hay không.
4. **Chất lượng end-to-end:** người dùng có nhận được kết quả đúng, có bằng chứng và dùng được hay không.
5. **Thuộc tính vận hành:** latency, chi phí, throughput, độ ổn định và tỷ lệ fallback.
6. **Rủi ro:** privacy, safety, bias, prompt injection hoặc nội dung bị bỏ sót.

OpenAI Evals mô hình hóa một evaluation thành dataset, tiêu chí kiểm tra và các lần chạy trên nhiều cấu hình; OpenAI graders còn hỗ trợ nội dung text, image và audio trong model-based grader. Đây là ví dụ về cách chuyển đánh giá AI từ kiểm tra thủ công sang quy trình có thể lặp lại. [OpenAI Evals][openai-evals] [OpenAI Graders][openai-graders]

**Áp dụng khi nào?**

- Thay model OCR, VLM, speech-to-text, image generation hoặc video model.
- Sửa prompt, system instruction, tool schema, parser hay hậu xử lý.
- Nâng version thư viện render PDF, thay DPI, crop, deskew hoặc compression.
- Thay chunking/retrieval trong multimodal RAG.
- Thay GPU, quantization, batching hoặc serving runtime.
- Chuẩn bị phát hành một hệ thống AI có chất lượng khó xác định bằng unit test.

**Bài toán giải quyết:**

- Phát hiện lỗi chất lượng ẩn sau khi refactor hoặc đổi model.
- Ngăn “điểm tổng tăng nhưng nhóm quan trọng giảm”.
- Tách lỗi code khỏi lỗi model, lỗi dữ liệu và lỗi grader.
- Biến đánh giá thủ công rời rạc thành release gate có bằng chứng.

**So sánh:**

| Cách kiểm thử | Đầu ra xác định | Hiểu nội dung | Kiểm tra nhiều phương thức | Phù hợp nhất |
| --- | --- | --- | --- | --- |
| Unit test | Rất tốt | Thấp | Chỉ khi tự viết logic | Hàm, parser, schema, quy tắc |
| Snapshot test | Tốt với output ổn định | Thấp | Có thể lưu artifact | Phát hiện thay đổi byte/DOM/JSON |
| Visual regression | So sánh pixel hoặc ảnh | Thường thấp | Chủ yếu ảnh/UI | Giao diện, render, layout ổn định |
| Model benchmark | Theo dataset chuẩn | Có | Có thể đa phương thức | So sánh năng lực model tổng quát |
| Multimodal regression suite | Kết hợp exact + metric + rubric | Cao | Có | Bảo vệ use case và pipeline production |

---

### 2. Một test case đa phương thức cần lưu những gì để tái lập và so sánh được?

**Trả lời:** Test case không nên chỉ gồm `prompt` và `expected_text`. Nó phải là một **gói bằng chứng có phiên bản** (*versioned evidence package*) đủ để chạy lại cùng input, biết cấu hình nào đã dùng và xác định phần nào thay đổi.

Ví dụ tối thiểu:

```json
{
  "caseId": "manual-pump-vi-0042",
  "task": "figure_grounded_qa",
  "risk": "high",
  "slice": [
    "vi",
    "technical-manual",
    "scan",
    "diagram",
    "low-contrast"
  ],
  "inputs": {
    "pdf": "sha256:...",
    "pages": [17],
    "question": "Van nào cần đóng trước khi tháo bơm?",
    "attachments": [
      {
        "type": "image",
        "uri": "artifact://page-17.webp",
        "sha256": "..."
      }
    ]
  },
  "expectations": {
    "requiredFacts": ["đóng van hút", "đóng van xả"],
    "forbiddenClaims": ["đóng van an toàn trước"],
    "evidenceRegions": ["p17-r08", "p17-r11"],
    "outputSchema": "schemas/grounded-answer-v2.json",
    "abstainIfEvidenceMissing": true
  },
  "oracle": {
    "deterministic": ["json_schema", "citation_exists"],
    "metrics": ["fact_recall", "evidence_iou"],
    "rubric": "rubrics/technical-safety-answer-v3.md",
    "humanReview": "required_on_disagreement"
  },
  "environment": {
    "pipelineVersion": "2026.07.27",
    "model": "provider/model-version",
    "promptVersion": "sha256:...",
    "ocrVersion": "sha256:...",
    "renderDpi": 200,
    "seed": 42
  }
}
```

Các nhóm dữ liệu quan trọng:

- **Input gốc bất biến:** file, trang, ảnh, audio, video và hash.
- **Task contract:** nhiệm vụ, schema, ngôn ngữ, domain, mức rủi ro.
- **Expected evidence:** text, vùng ảnh, timestamp audio/video, cell bảng hoặc section.
- **Oracle:** exact rule, metric, rubric, model judge và yêu cầu human review.
- **Slice:** loại tài liệu, độ phân giải, ngôn ngữ, thiết bị, mức nhiễu, nhóm người dùng.
- **Lineage:** version model, prompt, preprocessing, dependencies, hardware và seed.
- **Artifact:** output thô, output chuẩn hóa, crop, trace, token usage, latency và log lỗi.

Một dataset tốt thường gồm bốn nguồn:

1. **Golden set:** case quan trọng đã được chuyên gia xác nhận.
2. **Bug museum:** mọi lỗi production đáng kể được chuyển thành test.
3. **Coverage set:** case đại diện cho các slice dữ liệu.
4. **Challenge set:** edge case, nhiễu, tấn công và biến đổi có kiểm soát.

MLflow mô tả evaluation dataset như tập ví dụ có hoặc không có ground truth, dùng để ngăn regression, so sánh phiên bản và kiểm tra các vấn đề chuyên biệt. [MLflow datasets][mlflow-datasets]

**Áp dụng khi nào?** Ngay từ khi tạo prototype có nhiều hơn vài chục case hoặc khi nhiều nhóm cùng thay đổi model/prompt/pipeline.

**Bài toán giải quyết:** tái lập lỗi, audit kết quả, biết candidate khác baseline ở đâu và tránh phụ thuộc vào file hoặc prompt “đang nằm trên máy một người”.

**So sánh:**

| Dataset | Mục tiêu | Có thay đổi thường xuyên? | Có chặn release? |
| --- | --- | --- | --- |
| Public benchmark | So sánh năng lực chung | Ít | Không nên dùng một mình |
| Golden set nội bộ | Bảo vệ nghiệp vụ quan trọng | Có kiểm soát | Có |
| Production sample | Phản ánh phân phối hiện tại | Có | Thường cảnh báo/canary |
| Adversarial/challenge set | Tìm điểm yếu | Có | Chặn theo mức rủi ro |
| Training/validation set | Huấn luyện và chọn model | Có | Không được trộn với test kín |

---

### 3. Nên tổ chức “kim tự tháp kiểm thử” đa phương thức theo những tầng nào?

**Trả lời:** Không nên chỉ chạy end-to-end bằng một VLM judge vì chậm, đắt và khó debug. Một regression architecture tốt dùng nhiều tầng từ rẻ, xác định và cục bộ đến đắt, ngữ nghĩa và end-to-end.

```text
                 Human acceptance / field pilot
              ───────────────────────────────────
             End-to-end task + risk + business KPI
           ─────────────────────────────────────────
          Cross-modal grounding and consistency tests
        ───────────────────────────────────────────────
       Task metrics: OCR, layout, table, VQA, ASR, video
     ───────────────────────────────────────────────────
    Contract, schema, parser, transform and deterministic tests
```

**Tầng 1 — Contract và deterministic tests**

- file đọc được, MIME đúng, hash đúng;
- JSON hợp schema;
- bbox nằm trong kích thước trang;
- timestamp tăng dần;
- citation trỏ tới artifact tồn tại;
- không rò PII trong log;
- timeout/retry/fallback đúng.

**Tầng 2 — Metric theo tác vụ và phương thức**

- OCR: Character Error Rate (CER), Word Error Rate (WER), exact field match;
- layout/detection: Intersection over Union (IoU), precision, recall, mAP;
- bảng: cell precision/recall/F1, tree-edit hoặc structure similarity;
- ASR: WER, timestamp error, speaker attribution;
- classification: accuracy, macro-F1, AUROC;
- retrieval: Recall@k, MRR, nDCG;
- generation: fact coverage, constraint pass rate, style/safety rubric.

**Tầng 3 — Cross-modal tests**

- text trả lời có được hỗ trợ bởi đúng vùng ảnh không;
- caption có gắn đúng figure không;
- transcript có khớp đoạn audio không;
- mô tả video có đúng sự kiện và thứ tự thời gian không;
- câu trả lời có dùng đúng chart/table thay vì suy đoán từ prompt không.

**Tầng 4 — End-to-end task tests**

Ví dụ: “Từ PDF scan, tìm đúng quy trình thay vòng bi, giữ cảnh báo an toàn, dẫn đúng trang và hình, không bịa bước thao tác.”

**Tầng 5 — Human acceptance và field evaluation**

Dùng khi rubric khó tự động hóa, rủi ro cao hoặc cần đánh giá tính hữu dụng trong ngữ cảnh thật. NIST ARIA và AI RMF nhấn mạnh đánh giá hệ thống trong bối cảnh sử dụng, không chỉ trong phòng thí nghiệm. [NIST TEVV][nist-tevv] [NIST AI RMF][nist-ai-rmf]

**Áp dụng khi nào?** Mọi hệ thống production; đặc biệt khi pipeline có nhiều stage như render → layout → OCR → extraction → retrieval → VLM answer.

**Bài toán giải quyết:** giảm chi phí eval, phát hiện lỗi sớm và khoanh vùng stage hỏng.

**So sánh:**

| Kiến trúc | Ưu điểm | Giới hạn |
| --- | --- | --- |
| Chỉ end-to-end | Gần trải nghiệm người dùng | Chậm, khó biết lỗi nằm đâu |
| Chỉ component metrics | Debug tốt | Có thể bỏ sót lỗi tích hợp |
| Chỉ model judge | Linh hoạt với output mở | Có bias, drift và chi phí |
| Kim tự tháp kết hợp | Cân bằng tốc độ, độ tin cậy và chẩn đoán | Cần thiết kế dataset và lineage tốt |

---

### 4. Chọn oracle, grader và metric như thế nào khi không có một đáp án duy nhất?

**Trả lời:** **Test oracle** là cơ chế quyết định output có đạt yêu cầu hay không. Với multimodal AI, không có một oracle phù hợp cho mọi case. Nên dùng **oracle ladder** theo thứ tự ưu tiên:

1. **Deterministic oracle:** exact match, regex, JSON schema, geometry, checksum, rule nghiệp vụ.
2. **Reference-based metric:** so với ground truth text, bbox, label, timestamp hoặc cấu trúc.
3. **Constraint-based oracle:** phải có/không được có, citation bắt buộc, không vượt ngân sách.
4. **Pairwise comparison:** candidate tốt hơn, bằng hay kém baseline.
5. **Rubric-based model judge:** model chấm theo tiêu chí rõ ràng.
6. **Human expert:** nguồn chuẩn cho case mơ hồ, rủi ro cao và calibration.

OpenAI graders hỗ trợ string check, text similarity, Python grader và model-based grader, trong đó model grader có thể nhận text, image và audio. Vertex AI cũng hỗ trợ model-based metrics và khuyến nghị so điểm judge với human ratings khi tùy chỉnh judge. [OpenAI Graders][openai-graders] [Vertex judge][vertex-judge]

**Quy tắc lựa chọn:**

- Có thể viết rule xác định thì không dùng LLM judge.
- Output có cấu trúc thì chấm từng field trước, sau đó mới chấm ngữ nghĩa.
- Với answer mở, ưu tiên pairwise candidate–baseline để giảm biến thiên do thang điểm.
- Rubric phải tách tiêu chí: correctness, completeness, grounding, safety, format.
- Metric tổng hợp không được che lấp hard failure.
- Mọi judge quan trọng cần calibration với mẫu do người chấm và theo dõi agreement.

**Ví dụ metric theo đầu ra:**

| Đầu ra | Metric/oracle nên dùng | Không nên dùng một mình |
| --- | --- | --- |
| OCR text | CER, WER, normalized exact match | Cosine similarity |
| Layout region | IoU, class F1, reading-order score | Pixel diff |
| Bảng | Cell F1, structure match, field rules | BLEU trên Markdown |
| VQA/QA | Exact/F1 nếu đóng; rubric + grounding nếu mở | Chỉ ROUGE |
| Audio transcript | WER, timestamp error, entity recall | Model judge chung chung |
| Video event | Event precision/recall, temporal IoU, ordering | Frame similarity |
| Image generation | Constraint checks, human/rubric, safety | Chỉ CLIP similarity |
| Multimodal RAG | retrieval recall, faithfulness, evidence localization | Chỉ answer relevancy |

MM-Vet dùng evaluator dựa trên LLM cho câu trả lời mở và phân tích các tổ hợp năng lực; OCRBench tách nhiều nhóm tác vụ OCR thay vì dùng một điểm nhận dạng chữ duy nhất. Đây là lý do cần nhiều oracle theo cấu trúc nhiệm vụ. [MM-Vet][mm-vet] [OCRBench][ocrbench]

**Áp dụng khi nào?** Khi output có nhiều cách diễn đạt đúng, hoặc cần chấm cả nội dung lẫn evidence.

**Bài toán giải quyết:** tránh false fail do diễn đạt khác và false pass do câu trả lời nghe hợp lý nhưng không bám nguồn.

**So sánh:**

| Grader | Chi phí | Độ ổn định | Hiểu ngữ nghĩa | Dùng cho release gate |
| --- | ---: | ---: | ---: | --- |
| Rule/code | Thấp | Cao | Thấp–trung bình | Rất tốt |
| Metric tham chiếu | Thấp | Cao | Theo task | Rất tốt |
| Model judge | Trung bình–cao | Trung bình | Cao | Có, nếu calibrated |
| Human expert | Cao | Phụ thuộc guideline | Rất cao | Cho case trọng yếu/mẫu kiểm tra |

---

### 5. Thiết kế baseline, ngưỡng và release gate ra sao để không chặn nhầm hoặc lọt regression?

**Trả lời:** Release gate không nên là “điểm trung bình candidate lớn hơn baseline”. Cần so sánh theo **nhiều lớp ngưỡng**:

1. **Hard invariants:** không vi phạm schema, safety rule, citation, ACL, PII, crash.
2. **Critical-case gate:** tất cả case an toàn hoặc nghiệp vụ cấp cao phải pass.
3. **Slice gate:** không nhóm quan trọng nào giảm quá tolerance.
4. **Aggregate gate:** metric tổng thể không thấp hơn baseline ngoài biên cho phép.
5. **Operational budget:** p95 latency, cost/case, GPU memory, timeout và fallback trong ngân sách.
6. **Uncertainty gate:** nếu khác biệt nhỏ hơn nhiễu đo, kết quả là inconclusive thay vì pass/fail cứng.
7. **Manual review gate:** bất đồng giữa grader hoặc thay đổi lớn phải được duyệt.

Ví dụ chính sách:

```yaml
release_gate:
  hard_fail:
    schema_pass_rate: 1.0
    critical_safety_pass_rate: 1.0
    missing_evidence_rate_max: 0.0

  slices:
    vi_scan_low_contrast:
      cer_delta_max: 0.01
      grounded_answer_pass_rate_delta_min: -0.02
    technical_diagram:
      evidence_iou_delta_min: -0.03

  aggregate:
    weighted_quality_delta_min: 0.00
    bootstrap_confidence: 0.95

  operations:
    p95_latency_delta_max: 0.15
    cost_per_case_delta_max: 0.10

  action:
    pass: deploy_canary
    inconclusive: human_review
    fail: block
```

Nên lưu cả **absolute threshold** và **relative threshold**:

- absolute: `critical_pass_rate == 100%`;
- relative: `candidate - baseline >= -tolerance`;
- guardrail: không case nào thuộc lớp “catastrophic” được fail;
- statistical: dùng bootstrap confidence interval hoặc paired test khi dataset đủ lớn;
- practical significance: chênh lệch có ý nghĩa vận hành, không chỉ có ý nghĩa thống kê.

MLflow cung cấp regression testing và CI/CD bằng scorer + assertion; OpenAI Evals hỗ trợ chạy cùng evaluation trên nhiều model/parameter. Đây là mẫu chung cho baseline comparison có thể tự động hóa. [MLflow regression][mlflow-regression] [OpenAI Evals][openai-evals]

**Áp dụng khi nào?** Trước merge, trước deploy, khi đổi model/provider và trong canary.

**Bài toán giải quyết:** giảm release dựa trên cảm giác; tránh candidate cải thiện phần dễ nhưng làm hỏng phần quan trọng.

**So sánh:**

| Gate | Ưu điểm | Rủi ro |
| --- | --- | --- |
| Một điểm tổng | Đơn giản | Che lỗi theo slice |
| Pass rate từng case | Dễ hiểu | Quá cứng với output ngẫu nhiên |
| Statistical delta | Xử lý nhiễu tốt | Cần đủ mẫu và hiểu thống kê |
| Risk-based composite gate | Phù hợp production | Cần taxonomy rủi ro rõ |
| Human-only approval | Linh hoạt | Chậm, không lặp lại tốt |

---

## B. 8 câu hỏi phổ biến

### 6. Làm sao kiểm thử mô hình không xác định, cùng input nhưng output thay đổi?

**Trả lời:** Không nên cố biến mọi output thành deterministic bằng cách chỉ đặt temperature bằng 0. Một số API, model hoặc hạ tầng vẫn có biến thiên. Hãy đo **phân phối hành vi**:

- chạy mỗi case nhiều lần với cùng cấu hình;
- lưu seed khi hệ thống hỗ trợ nhưng không xem seed là bảo đảm tuyệt đối;
- dùng pass rate, mean/median, worst-case và variance;
- tách lỗi “thỉnh thoảng” khỏi lỗi “luôn xảy ra”;
- với pairwise judge, đảo thứ tự A/B để phát hiện position bias;
- đặt ngưỡng như `pass_at_5 >= 0.8` hoặc `critical_failures == 0/10`;
- khóa version model, prompt, tool, dependency và preprocess;
- cache input artifact, không cache output khi đang đo stochasticity.

**Áp dụng khi nào?** LLM/VLM generation, agent, speech generation, image/video generation hoặc pipeline có sampling.

**Bài toán giải quyết:** tránh kết luận sai từ một lần chạy may mắn hoặc xui rủi.

**So sánh:**

| Cách | Khi phù hợp | Giới hạn |
| --- | --- | --- |
| Một lần chạy | Unit/contract deterministic | Không đại diện biến thiên |
| N lần, lấy trung bình | Metric liên tục | Có thể che worst-case |
| N lần, pass rate | Case dạng đạt/không đạt | Tốn chi phí |
| Worst-case/quantile | Use case rủi ro cao | Có thể quá bảo thủ |
| Sequential testing | Muốn dừng sớm khi đủ bằng chứng | Triển khai phức tạp hơn |

---

### 7. Metamorphic testing và perturbation testing áp dụng thế nào cho ảnh, text, audio, video và PDF?

**Trả lời:** Khi không biết output chính xác, có thể biết **quan hệ kỳ vọng** giữa input gốc và input biến đổi. Đó là *metamorphic relation*.

Ví dụ:

- xoay ảnh 90° rồi sửa orientation đúng thì nội dung OCR không đổi đáng kể;
- nén JPEG nhẹ không được làm mất cảnh báo an toàn;
- đổi kích thước ảnh trong giới hạn không được đổi số lượng object;
- paraphrase câu hỏi nhưng giữ ý nghĩa thì answer facts phải nhất quán;
- thêm khoảng im lặng đầu audio chỉ được dịch timestamp, không đổi transcript;
- lấy một clip con chứa cùng sự kiện thì event label vẫn phải đúng;
- render cùng PDF ở 200 và 300 DPI phải giữ reading order và section hierarchy;
- che vùng evidence thì model phải giảm confidence hoặc abstain, không được trả lời như cũ.

Nghiên cứu robustness của image-text model cho thấy mô hình có thể suy giảm đáng kể dưới image/text perturbation và distribution shift. MetaRA dùng metamorphic relations để tạo biến thể có kiểm soát cho VQA; OCR-Robust kiểm tra OCR reasoning dưới các mức nhiễu thị giác và cho thấy clean accuracy cao không đồng nghĩa robustness cao. [Multimodal robustness][mm-robustness] [MetaRA][metara] [OCR-Robust][ocr-robust]

**Áp dụng khi nào?** Dữ liệu production có scan mờ, ảnh điện thoại, tiếng ồn, codec khác nhau, nhiều layout hoặc người dùng diễn đạt đa dạng.

**Bài toán giải quyết:** tìm lỗi mà golden set “sạch” không phát hiện.

**So sánh:**

| Kiểu test | Có ground truth đầy đủ? | Mục tiêu |
| --- | --- | --- |
| Golden test | Có | Đúng trên case biết trước |
| Perturbation test | Có hoặc suy ra từ case gốc | Bền trước nhiễu |
| Metamorphic test | Không nhất thiết | Bảo toàn/biến đổi quan hệ mong đợi |
| Adversarial test | Không nhất thiết | Tìm cách làm hệ thống thất bại |
| Distribution-shift test | Có mẫu từ miền mới | Khả năng tổng quát hóa |

---

### 8. Làm sao khoanh vùng regression trong pipeline PDF/OCR/Document AI thay vì chỉ biết kết quả cuối sai?

**Trả lời:** Mỗi stage phải xuất **artifact chuẩn hóa và metric cục bộ**, sau đó liên kết bằng trace ID:

```text
PDF bytes
  ├─ preflight.json
  ├─ rendered-page.webp
  ├─ layout.json
  ├─ ocr-tokens.json
  ├─ table.json / figures/
  ├─ document-model.json
  ├─ chunks.json
  ├─ retrieval.json
  └─ answer.json + evidence.json
```

Cho một case lỗi, so baseline và candidate theo thứ tự:

1. **Input/preflight:** file hash, page count, rotation, digital/scan/hybrid.
2. **Render:** kích thước, DPI, crop box, màu, alpha.
3. **Layout:** region count, class, bbox, reading order, figure-caption links.
4. **OCR:** token/line text, confidence, CER, language.
5. **Structure:** heading hierarchy, table cells, form keys.
6. **Index/retrieval:** chunk IDs, Recall@k, filters, ACL.
7. **Generation:** required facts, evidence, abstention.
8. **Serving:** timeout, truncation, token budget, fallback.

Một error taxonomy hữu ích:

- `INGESTION_CHANGED`
- `RENDER_CHANGED`
- `LAYOUT_MISSED_REGION`
- `OCR_TEXT_REGRESSION`
- `READING_ORDER_REGRESSION`
- `TABLE_STRUCTURE_REGRESSION`
- `RETRIEVAL_MISS`
- `GROUNDING_FAILURE`
- `GENERATION_HALLUCINATION`
- `GRADER_DISAGREEMENT`
- `INFRA_NONDETERMINISM`

**Áp dụng khi nào?** Pipeline nhiều model/engine hoặc số lượng PDF lớn.

**Bài toán giải quyết:** giảm thời gian root-cause analysis và tránh đổi model cuối trong khi lỗi nằm ở render hoặc OCR.

**So sánh:**

| Quan sát | Biết end-to-end sai | Biết stage sai | Tái lập được |
| --- | ---: | ---: | ---: |
| Chỉ log text | Có | Thấp | Thấp |
| Lưu output cuối | Có | Thấp | Trung bình |
| Trace + artifact từng stage | Có | Cao | Cao |
| Trace + artifact + version/hash | Có | Cao | Rất cao |

---

### 9. Model-as-a-judge có đáng tin không và cần hiệu chuẩn với con người ra sao?

**Trả lời:** Model judge hữu ích cho output mở, nhưng không phải source of truth mặc định. Nó có thể có:

- position bias;
- verbosity bias;
- self-preference hoặc provider bias;
- sensitivity với prompt/rubric;
- lỗi khi input ảnh nhỏ, audio dài hoặc evidence bị cắt;
- drift khi model judge được cập nhật;
- inconsistency giữa các lần chạy.

Quy trình calibration:

1. Chọn 100–500 case đại diện theo slice và rủi ro.
2. Có ít nhất hai người chấm độc lập với rubric rõ.
3. Giải quyết bất đồng và tạo adjudicated label.
4. Chạy judge trên cùng case, đo agreement, precision/recall theo nhãn hoặc correlation theo điểm.
5. Phân tích confusion theo slice, không chỉ điểm chung.
6. Sửa rubric/prompt; khóa judge version.
7. Đặt vùng “uncertain” để chuyển human review.
8. Định kỳ re-calibrate khi domain, model hoặc rubric đổi.

Vertex AI khuyến nghị dùng human ratings làm ground truth để kiểm tra judge model. OpenAI GDPval dùng chuyên gia và rubric, đồng thời lưu ý automated grader chưa thay thế expert graders. [Vertex judge][vertex-judge] [OpenAI GDPval][openai-gdpval]

**Áp dụng khi nào?** Chấm groundedness, completeness, style, visual quality hoặc task success khó viết bằng code.

**Bài toán giải quyết:** tự động hóa phần lớn eval mà vẫn biết giới hạn của grader.

**So sánh:**

| Cách chấm | Quy mô | Chi phí | Độ tin cậy |
| --- | ---: | ---: | --- |
| Một judge, không calibration | Cao | Trung bình | Thấp–không biết |
| Judge + rubric + calibration | Cao | Trung bình | Tốt hơn, đo được |
| Ensemble judges | Trung bình | Cao | Có thể giảm bias đơn lẻ |
| Human expert | Thấp | Rất cao | Cao nếu guideline tốt |
| Hybrid judge + human escalation | Cao | Tối ưu hơn | Phù hợp production |

---

### 10. Cần version hóa những gì để kết quả eval có thể audit và tái lập?

**Trả lời:** Tối thiểu phải version hóa:

- test dataset và từng binary artifact bằng hash;
- expected output, rubric và annotation;
- prompt/system instruction/tool schema;
- model ID, provider, snapshot/version và sampling params;
- OCR/layout/parser model;
- preprocessing: DPI, resize, crop, normalization, codec;
- code commit, container image, dependencies và driver;
- feature flag, fallback policy và routing rule;
- grader model, grader prompt và metric implementation;
- hardware/runtime region khi có thể ảnh hưởng;
- eval runner version và cấu hình concurrency/cache.

Nên tạo một **evaluation manifest**:

```json
{
  "runId": "eval-20260727-184500",
  "baseline": "run-20260725-090000",
  "gitCommit": "abc123",
  "containerDigest": "sha256:...",
  "datasetVersion": "mmre-golden-7",
  "datasetDigest": "sha256:...",
  "pipeline": {
    "render": "v4",
    "layout": "model-x@sha256:...",
    "ocr": "model-y@sha256:...",
    "prompt": "sha256:..."
  },
  "grader": {
    "model": "judge-snapshot",
    "rubric": "sha256:..."
  }
}
```

**Áp dụng khi nào?** Luôn cần ở production và khi nhiều model/provider thay đổi độc lập.

**Bài toán giải quyết:** trả lời được “vì sao tuần trước pass nhưng hôm nay fail?” và hỗ trợ rollback/audit.

**So sánh:**

| Cách ghi nhận | Audit | Reproduce | Chi phí vận hành |
| --- | ---: | ---: | ---: |
| Ghi tên model chung | Thấp | Thấp | Thấp |
| Ghi config trong log | Trung bình | Trung bình | Thấp |
| Manifest + hash + immutable artifact | Cao | Cao | Trung bình |
| Full environment capture | Rất cao | Rất cao | Cao |

---

### 11. Làm sao cân bằng chất lượng, latency, throughput và chi phí trong regression suite?

**Trả lời:** Chất lượng AI là bài toán đa mục tiêu. Một candidate tốt hơn 1% nhưng chậm gấp ba hoặc tăng chi phí 80% có thể không phù hợp production.

Nên tách suite thành:

- **PR smoke suite:** 20–100 case, deterministic + critical, chạy nhanh.
- **Nightly suite:** vài trăm đến vài nghìn case, nhiều slice và lặp lại.
- **Release suite:** golden + challenge + security + performance.
- **Shadow/canary:** mẫu production thật, không hoặc có tác động giới hạn.
- **Continuous monitoring:** sampling trace, drift và feedback.

Các metric vận hành:

- p50/p95/p99 latency theo loại input;
- time-to-first-token/frame;
- throughput trang/phút, audio phút/phút, video fps;
- GPU memory, CPU/RAM, queue time;
- token/image/audio/video unit usage;
- cost per successful task, không chỉ cost per request;
- retry/fallback/escalation rate;
- cache hit rate và error rate.

**Áp dụng khi nào?** Model lớn, tài liệu nhiều trang, audio/video dài hoặc hệ thống scale từ hàng chục nghìn tới hàng triệu file.

**Bài toán giải quyết:** ngăn tối ưu chất lượng cục bộ làm hệ thống không kinh tế hoặc không đạt SLA.

**So sánh:**

| Chiến lược | Tốc độ feedback | Độ phủ | Chi phí |
| --- | ---: | ---: | ---: |
| Chạy full suite mỗi commit | Chậm | Cao | Cao |
| Tiered suite | Nhanh ở PR, sâu ở release | Cao | Tối ưu |
| Chỉ benchmark offline | Trung bình | Không phản ánh serving | Trung bình |
| Canary + offline | Tốt | Cao nhất | Cần observability tốt |

---

### 12. Regression suite cần kiểm tra safety, security, privacy và bias như thế nào?

**Trả lời:** Với multimodal system, rủi ro không chỉ nằm trong text output. Input có thể chứa prompt injection trong ảnh/PDF, PII trong tiếng nói, khuôn mặt, biển số, metadata ảnh, watermark hoặc nội dung nhạy cảm.

Các nhóm test:

- **Access control:** cùng câu hỏi nhưng user khác quyền phải thấy evidence khác nhau.
- **Prompt injection:** instruction trong ảnh/PDF không được ghi đè system policy.
- **Data exfiltration:** không trả content ngoài tài liệu hoặc tenant được phép.
- **PII:** OCR/transcript/log/crop không rò dữ liệu bị che.
- **Safety:** không bỏ qua cảnh báo vận hành hoặc biến đổi nội dung nguy hiểm.
- **Bias/fairness:** đo theo nhóm ngôn ngữ, giọng nói, màu da, thiết bị, chất lượng ảnh khi phù hợp và hợp pháp.
- **Abstention:** thiếu evidence phải từ chối hoặc yêu cầu ảnh rõ hơn.
- **Content integrity:** không gán sai caption, sửa số đo hoặc bỏ đơn vị.
- **Artifact retention:** binary test có chính sách lưu, mã hóa và xóa.

NIST AI RMF và GenAI Profile đặt evaluation trong toàn bộ lifecycle, gồm tính hợp lệ, an toàn, bảo mật, riêng tư, minh bạch và bias. [NIST AI RMF][nist-ai-rmf] [NIST GenAI Profile][nist-genai-profile]

**Áp dụng khi nào?** Tài liệu nội bộ, y tế, tài chính, sản xuất, camera, giọng nói hoặc dữ liệu người dùng.

**Bài toán giải quyết:** ngăn release đạt accuracy nhưng vi phạm quyền truy cập hoặc tạo rủi ro vận hành.

**So sánh:**

| Test | Mục tiêu | Có thể tự động hoàn toàn? |
| --- | --- | --- |
| Schema/ACL/PII regex | Hợp đồng và rò rỉ rõ ràng | Phần lớn |
| Adversarial prompt/image | Khả năng chống tấn công | Phần lớn, cần cập nhật |
| Bias slice metrics | Chênh lệch theo nhóm | Định lượng được |
| Safety rubric | Ngữ cảnh và mức độ nguy hiểm | Cần judge/human |
| Red team/field test | Failure mode chưa biết | Không |

---

### 13. Nên dùng công cụ nào: pytest tự xây, OpenAI Evals, Vertex AI Evaluation, MLflow, LangSmith hay Ragas?

**Trả lời:** Không có công cụ duy nhất bao phủ toàn bộ regression engineering. Chọn theo ranh giới hệ thống và khả năng lock-in chấp nhận được.

| Công cụ/cách tiếp cận | Điểm mạnh | Giới hạn | Phù hợp |
| --- | --- | --- | --- |
| `pytest` + code tùy chỉnh | Deterministic, CI dễ, kiểm soát artifact/schema | Tự xây UI, dataset, judge orchestration | Component, contract, pipeline nội bộ |
| OpenAI Evals/Graders | Dataset + run + nhiều grader; model grader nhận text/image/audio | Gắn với API/platform OpenAI cho phần managed | Ứng dụng dùng OpenAI, multimodal grading |
| Vertex AI Evaluation | Managed evaluation, judge customization, gen-media evaluation | Gắn Google Cloud; tính năng có thể khác theo release stage | Hệ sinh thái Vertex/Gemini/Imagen/Veo |
| MLflow GenAI Evaluation | Open-source tracking, dataset, scorer, regression CI/CD, offline/online | Multimodal binary workflow có thể cần code tích hợp thêm | Multi-provider, self-hosted MLOps |
| LangSmith | Dataset/trace/evaluator; attachment image/audio/document | SaaS và thiên về LangChain/agent workflow | Multimodal app, trace-centric debugging |
| Ragas | Metric cho RAG, gồm multimodal faithfulness ở một số phiên bản | Không thay thế test framework tổng quát | RAG và multimodal RAG |
| Harness tự xây | Tùy biến tối đa, tối ưu dữ liệu nhạy cảm | Chi phí kỹ thuật và bảo trì cao | Quy mô lớn, domain đặc thù, on-prem |

LangSmith hỗ trợ attachment như image, audio và document trong evaluation; MLflow phân biệt offline regression testing với online evaluation; Ragas cung cấp metric cho RAG và multimodal faithfulness. [LangSmith multimodal][langsmith-multimodal] [MLflow evaluation][mlflow-evaluation] [Ragas metrics][ragas-metrics]

**Khuyến nghị thực dụng cho pipeline PDF kỹ thuật:**

```text
pytest
  ├─ schema, bbox, hash, ACL, idempotency
  ├─ CER/WER, IoU, table/field metrics
  └─ golden critical cases

MLflow hoặc LangSmith
  ├─ dataset version
  ├─ run comparison
  ├─ traces/artifacts
  └─ model judge + human feedback

Ragas hoặc custom metrics
  └─ multimodal RAG faithfulness, retrieval, evidence

CI/CD
  ├─ PR smoke
  ├─ nightly full
  ├─ release gate
  └─ canary monitoring
```

**Áp dụng khi nào?** Khi chuyển từ script thử nghiệm sang quy trình nhiều thành viên, nhiều model và nhiều lần phát hành.

**Bài toán giải quyết:** tránh xây lại mọi thứ, nhưng vẫn giữ deterministic test và artifact lineage ở lớp nội bộ.

**So sánh kết luận:** dùng framework managed để tăng tốc orchestration và quan sát; giữ source of truth của test case, expected evidence, rubric, hash và hard gate trong repository hoặc kho dữ liệu do đội kiểm soát.

---

## Checklist triển khai tối thiểu

- [ ] Xác định task, risk tier và critical slices.
- [ ] Tạo 30–100 golden cases đầu tiên từ production/bug thật.
- [ ] Lưu binary artifact bằng hash; không chỉ lưu URL tạm.
- [ ] Viết deterministic contract trước model judge.
- [ ] Tách metric từng stage và end-to-end.
- [ ] Định nghĩa evidence grounding cho answer đa phương thức.
- [ ] Thêm perturbation/metamorphic cases.
- [ ] Khóa version model, prompt, preprocess, grader và dependency.
- [ ] So candidate với baseline theo từng slice.
- [ ] Có hard gate cho safety, ACL, PII và critical cases.
- [ ] Đo latency, throughput và cost per successful task.
- [ ] Calibrate judge với human labels.
- [ ] Chạy smoke ở PR, full suite định kỳ, canary sau deploy.
- [ ] Đưa mọi bug production quan trọng trở lại regression suite.

## Nguồn tham khảo

- [NIST — AI Test, Evaluation, Validation and Verification][nist-tevv]
- [NIST — AI Risk Management Framework 1.0][nist-ai-rmf]
- [NIST — Generative AI Profile][nist-genai-profile]
- [OpenAI API — Evals][openai-evals]
- [OpenAI API — Graders][openai-graders]
- [OpenAI — GDPval grading methodology][openai-gdpval]
- [Google Cloud — Evaluate a judge model][vertex-judge]
- [Google Cloud — Multimodal evaluation for generative media][vertex-gen-media]
- [MMMU benchmark][mmmu]
- [MM-Vet benchmark][mm-vet]
- [OCRBench][ocrbench]
- [Robustness of multimodal image-text models under distribution shift][mm-robustness]
- [MetaRA metamorphic robustness assessment][metara]
- [OCR-Robust visual perturbation benchmark][ocr-robust]
- [MLflow — Evaluation datasets][mlflow-datasets]
- [MLflow — Regression testing and CI/CD][mlflow-regression]
- [MLflow — GenAI evaluation][mlflow-evaluation]
- [LangSmith — Evaluate multimodal content with attachments][langsmith-multimodal]
- [Ragas — Available evaluation metrics][ragas-metrics]

[nist-tevv]: https://www.nist.gov/ai-test-evaluation-validation-and-verification-tevv
[nist-ai-rmf]: https://www.nist.gov/publications/artificial-intelligence-risk-management-framework-ai-rmf-10
[nist-genai-profile]: https://www.nist.gov/publications/artificial-intelligence-risk-management-framework-generative-artificial-intelligence
[openai-evals]: https://platform.openai.com/docs/api-reference/evals
[openai-graders]: https://platform.openai.com/docs/api-reference/graders
[openai-gdpval]: https://openai.com/index/gdpval/
[vertex-judge]: https://docs.cloud.google.com/vertex-ai/generative-ai/docs/models/evaluate-judge-model
[vertex-gen-media]: https://cloud.google.com/blog/products/ai-machine-learning/evaluate-your-gen-media-models-on-vertex-ai/
[mmmu]: https://arxiv.org/abs/2311.16502
[mm-vet]: https://arxiv.org/abs/2308.02490
[ocrbench]: https://arxiv.org/abs/2305.07895
[mm-robustness]: https://arxiv.org/abs/2212.08044
[metara]: https://arxiv.org/abs/2605.19307
[ocr-robust]: https://arxiv.org/abs/2606.26041
[mlflow-datasets]: https://mlflow.org/docs/latest/genai/datasets/
[mlflow-regression]: https://mlflow.org/docs/latest/genai/eval-monitor/regression-testing/
[mlflow-evaluation]: https://mlflow.org/docs/latest/genai/eval-monitor/
[langsmith-multimodal]: https://docs.langchain.com/langsmith/evaluate-with-attachments
[ragas-metrics]: https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/
