# Chain of Thought (CoT)

> [Knowledge Tree](../../../README.md) / [AI](../../README.md) / [LLM](../README.md) / **Chain of Thought (CoT)**

**Chain of Thought (CoT)**, có thể dịch là **chuỗi suy luận**, là kỹ thuật khuyến khích Mô hình ngôn ngữ lớn (*Large Language Model — LLM*) xử lý một bài toán qua các bước trung gian trước khi đưa ra kết luận. CoT đặc biệt hữu ích khi đáp án phụ thuộc vào nhiều phép biến đổi, điều kiện hoặc quyết định nối tiếp nhau, thay vì chỉ cần nhớ một thông tin đơn lẻ.

> **Problem → Decompose → Reason through intermediate steps → Verify → Final answer**

CoT không tự bảo đảm câu trả lời đúng. Một chuỗi giải thích có vẻ hợp lý vẫn có thể chứa lỗi, hợp thức hóa một đáp án sai hoặc không phản ánh đầy đủ cơ chế thực sự đã dẫn đến kết quả. Trong production, nên kết hợp CoT với dữ liệu nguồn, công cụ tính toán, bộ kiểm thử và bước xác minh thay vì coi phần giải thích của model là bằng chứng cuối cùng.

## Điều hướng

- **Node cha:** [Mô hình ngôn ngữ lớn (LLM)](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** [RAG (Retrieval-Augmented Generation)](../rag/README.md)
- **Khái niệm sau trong cùng nhánh:** Chưa có.

## A. 5 câu hỏi cốt lõi

### 1. Chain of Thought là gì, và nó giải quyết bài toán nào?

**Trả lời:** CoT là cách để LLM tạo hoặc sử dụng các bước suy luận trung gian trước khi đưa ra đáp án cuối. Trong nghiên cứu CoT ban đầu, model được cung cấp một số ví dụ gồm cả cách giải từng bước; model sau đó bắt chước cấu trúc này để giải bài toán mới. Kỹ thuật này đã cải thiện kết quả trên nhiều bài toán số học, suy luận thông thường và suy luận ký hiệu, đặc biệt với các model đủ mạnh.

CoT phù hợp khi câu trả lời cần một **quá trình**, chẳng hạn:

- Bài toán nhiều phép tính hoặc nhiều điều kiện.
- Phân tích yêu cầu phần mềm thành quy tắc nghiệp vụ.
- Chẩn đoán lỗi từ triệu chứng, log và các giả thuyết.
- So sánh phương án theo nhiều tiêu chí.
- Lập kế hoạch có phụ thuộc giữa các bước.
- Kiểm tra một kết luận dựa trên bằng chứng đã cho.

Ví dụ, với yêu cầu “khách VIP được hoàn 5% điểm nhưng đơn hoàn tiền toàn phần không được tích điểm”, model phải nhận diện trạng thái khách hàng, trạng thái hoàn tiền, thứ tự áp dụng quy tắc và ngoại lệ. Trả lời trực tiếp dễ bỏ qua ngoại lệ; phân rã theo bước giúp kiểm tra từng điều kiện.

**Áp dụng khi nào?** Khi bài toán có từ hai bước phụ thuộc trở lên, cần giải thích có cấu trúc, hoặc cần kiểm tra model đã xét đủ điều kiện quan trọng.

**Bài toán giải quyết:** giảm việc đoán đáp án ngay lập tức, giúp model dành thêm token cho phân rã vấn đề, theo dõi dữ kiện và kiểm tra mâu thuẫn.

**Không nên kỳ vọng:** CoT không bổ sung kiến thức mới, không biến dữ liệu sai thành đúng và không thay thế calculator, database, search engine hay API nghiệp vụ.

**So sánh với trả lời trực tiếp:**

| Cách làm | Điểm mạnh | Giới hạn | Chọn khi |
| --- | --- | --- | --- |
| Trả lời trực tiếp (*direct answer*) | Nhanh, ít token, phù hợp câu hỏi đơn giản. | Dễ bỏ bước hoặc điều kiện trong bài toán phức tạp. | Phân loại đơn giản, tra cứu rõ ràng, format ngắn. |
| Chain of Thought | Hỗ trợ suy luận nhiều bước và giải thích cấu trúc. | Tăng token, latency; reasoning vẫn có thể sai. | Toán, logic, phân tích điều kiện, lập kế hoạch. |
| Tool/API | Lấy dữ liệu hoặc tính toán từ nguồn xác định. | Cần tích hợp và kiểm soát lỗi công cụ. | Giá, tồn kho, số dư, phép tính chính xác, hành động có side effect. |

Nguồn nền tảng: [Chain-of-Thought Prompting Elicits Reasoning in Large Language Models](https://arxiv.org/abs/2201.11903).

---

### 2. CoT prompting hoạt động như thế nào?

**Trả lời:** CoT prompting định hình phần ngữ cảnh để model không chỉ nhìn thấy `input → answer`, mà còn nhìn thấy hoặc nhận được yêu cầu về **các bước trung gian**. Có hai dạng cơ bản:

1. **Few-shot CoT:** đưa một số ví dụ có câu hỏi, cách phân tích và đáp án.
2. **Zero-shot CoT:** không đưa ví dụ; chỉ thêm chỉ dẫn như “hãy giải theo từng bước” hoặc một cấu trúc phân tích rõ ràng.

Mẫu few-shot tối giản:

```text
Ví dụ:
Câu hỏi: Một đơn hàng 500.000đ được giảm 10%, sau đó tính VAT 8%. Tổng tiền là bao nhiêu?
Phân tích:
1. Tiền sau giảm = 500.000 × 90% = 450.000đ.
2. VAT = 450.000 × 8% = 36.000đ.
3. Tổng = 486.000đ.
Đáp án: 486.000đ.

Bài toán mới:
...
```

Mẫu production nên ưu tiên kết quả có thể kiểm chứng thay vì yêu cầu một đoạn độc thoại dài:

```text
Hãy:
1. Liệt kê dữ kiện và ràng buộc quan trọng.
2. Chia bài toán thành các bước ngắn.
3. Dùng công cụ cho phép tính hoặc dữ liệu cần độ chính xác.
4. Kiểm tra ngoại lệ và mâu thuẫn.
5. Trả về kết luận, phép tính/bằng chứng chính và mức độ chắc chắn.
```

**Áp dụng khi nào?** Few-shot phù hợp khi cần model tuân thủ một kiểu suy luận hoặc định dạng ổn định. Zero-shot phù hợp khi muốn thử nhanh hoặc bài toán có cấu trúc quen thuộc.

**Bài toán giải quyết:** hướng model dành quá trình sinh token cho suy luận trung gian thay vì tạo ngay đáp án đầu tiên có vẻ hợp lý.

**So sánh hai dạng:**

| Dạng | Ưu điểm | Nhược điểm | Chọn khi |
| --- | --- | --- | --- |
| Zero-shot CoT | Prompt ngắn, triển khai nhanh, không cần viết ví dụ. | Ít kiểm soát cách phân tích và format. | Prototype, bài toán phổ biến, model mạnh. |
| Few-shot CoT | Dạy được cấu trúc, quy ước và mức chi tiết. | Tốn context; ví dụ sai có thể truyền lỗi. | Nghiệp vụ đặc thù, cần output ổn định. |
| Fine-tuning với rationale | Có thể học pattern reasoning từ nhiều mẫu. | Cần dataset, training và đánh giá kỹ. | Khối lượng lớn, tác vụ ổn định, cần giảm prompt dài. |

Nguồn: [Large Language Models are Zero-Shot Reasoners](https://arxiv.org/abs/2205.11916).

---

### 3. Khi nào nên và không nên dùng CoT?

**Trả lời:** CoT có giá trị khi **chi phí suy luận thêm** đổi lại khả năng giải bài toán tốt hơn. Không nên bật CoT như một quy tắc mặc định cho mọi request.

**Nên dùng khi:**

- Có nhiều điều kiện, ngoại lệ hoặc phép biến đổi nối tiếp.
- Cần phân rã một yêu cầu lớn thành các phần nhỏ.
- Cần kiểm tra giả thuyết, so sánh phương án hoặc lập kế hoạch.
- Kết quả có thể được xác minh bằng test, calculator, code, tài liệu hoặc rule engine.
- Sai sót do bỏ bước thường xuyên hơn sai sót do thiếu kiến thức.

**Không cần hoặc không nên dùng khi:**

- Câu hỏi chỉ là tra cứu một sự kiện rõ ràng.
- Nhiệm vụ là phân loại đơn giản với nhãn cố định.
- Độ trễ và chi phí token quan trọng hơn phần giải thích.
- Dữ liệu đầu vào không đủ; reasoning dài chỉ làm câu trả lời sai thuyết phục hơn.
- Nội dung chứa dữ liệu nhạy cảm không nên xuất hiện trong trace hoặc log.
- Hệ thống cần kết quả xác định tuyệt đối từ code, SQL, calculator hoặc rule engine.

**Quy tắc thực hành:**

```text
Simple lookup/classification → Direct answer
Multi-step reasoning → CoT or decomposition
Exact computation → CoT + calculator/code
Current/private knowledge → RAG/search + concise reasoning
Business action → Tool/API + validation + audit
Planning/search space lớn → Least-to-Most hoặc Tree of Thoughts
```

**Bài toán giải quyết:** tránh hai cực đoan: model trả lời quá nhanh với bài khó, hoặc tiêu tốn reasoning dài cho bài rất đơn giản.

**So sánh với prompt dài:** Prompt dài cung cấp thêm hướng dẫn hoặc dữ liệu; CoT tập trung vào cách xử lý nhiều bước. Một prompt có thể dài nhưng vẫn yêu cầu trả lời trực tiếp, và một prompt CoT có thể rất ngắn. Hai khái niệm không đồng nhất.

---

### 4. CoT khác Prompt Chaining, Least-to-Most, Tree of Thoughts, ReAct và Program of Thoughts thế nào?

**Trả lời:** Các kỹ thuật này cùng hỗ trợ bài toán phức tạp nhưng khác ở cách tổ chức suy luận và mức độ tương tác với môi trường.

| Kỹ thuật | Cách hoạt động | Mạnh khi | Hạn chế chính |
| --- | --- | --- | --- |
| **CoT** | Một luồng bước trung gian dẫn đến đáp án. | Bài toán nhiều bước nhưng đường giải tương đối tuyến tính. | Dễ đi sai từ sớm và tiếp tục hợp thức hóa. |
| **Prompt Chaining** | Ứng dụng chia workflow thành nhiều lần gọi model; output bước trước là input bước sau. | Cần kiểm soát, quan sát và retry từng giai đoạn. | Tăng số request, latency và state management. |
| **Least-to-Most** | Tách bài khó thành các bài con từ dễ đến khó rồi giải tuần tự. | Bài toán có cấu trúc phân cấp và cần tổng quát từ dễ sang khó. | Phụ thuộc chất lượng bước decomposition. |
| **Tree of Thoughts (ToT)** | Mở nhiều nhánh suy luận, đánh giá, quay lui hoặc chọn nhánh tốt. | Search, planning, puzzle, quyết định có nhiều đường đi. | Tốn nhiều inference và phức tạp hơn CoT. |
| **ReAct** | Xen kẽ reasoning với action/tool và observation. | Agent cần tìm kiếm, gọi API hoặc tương tác môi trường. | Cần quản lý tool, quyền, vòng lặp và lỗi action. |
| **Program of Thoughts (PoT)** | Model biểu diễn phép tính thành chương trình; máy tính thực thi. | Toán, tài chính, thao tác bảng và tính toán chính xác. | Cần sandbox, kiểm tra code và giới hạn an toàn. |

**Cách chọn:**

- Chọn **CoT** khi bài toán chủ yếu là reasoning trong context hiện có.
- Chọn **Prompt Chaining** khi cần tách trách nhiệm giữa phân tích, truy xuất, kiểm tra và tạo output.
- Chọn **Least-to-Most** khi bài toán lớn có thể chia thành subproblem rõ ràng.
- Chọn **ToT** khi cần thử nhiều phương án và có tiêu chí chấm nhánh.
- Chọn **ReAct** khi model phải lấy thêm thông tin hoặc thực hiện hành động.
- Chọn **PoT** khi phần khó nằm ở tính toán và có thể giao cho code.

**Bài toán giải quyết:** tránh dùng một chuỗi reasoning tuyến tính cho mọi loại tác vụ. Kiến trúc suy luận nên khớp với không gian tìm kiếm và công cụ có sẵn.

Nguồn: [Least-to-Most Prompting](https://arxiv.org/abs/2205.10625), [Tree of Thoughts](https://arxiv.org/abs/2305.10601), [ReAct](https://arxiv.org/abs/2210.03629), [Program of Thoughts](https://arxiv.org/abs/2211.12588).

---

### 5. Thiết kế CoT cho production như thế nào để có thể kiểm chứng và vận hành?

**Trả lời:** Production không nên chỉ dùng prompt “think step by step” rồi tin toàn bộ đoạn reasoning. Cần biến CoT thành một phần của pipeline có **input contract, output contract, verifier, metrics và quy tắc bảo mật**.

Một pipeline tham khảo:

```text
Request
  → classify complexity/risk
  → collect trusted context or call tools
  → generate plan/structured rationale
  → execute calculations/actions
  → verify constraints and evidence
  → produce concise answer
  → log safe trace + metrics
```

**Thiết kế input:**

- Xác định mục tiêu, dữ kiện, ràng buộc và điều model không được giả định.
- Tách dữ liệu nguồn khỏi chỉ dẫn để giảm prompt injection.
- Với RAG, gắn `sourceId`, version, quyền truy cập và citation vào evidence.
- Với tool, khai báo schema, timeout, retry và trường hợp không được gọi.

**Thiết kế output:**

- Ưu tiên các trường có thể kiểm tra: `assumptions`, `evidenceIds`, `calculation`, `decision`, `confidence`, `unresolvedQuestions`.
- Không bắt buộc hiển thị toàn bộ reasoning dài cho người dùng cuối.
- Với nghiệp vụ rủi ro cao, trả kết luận kèm bằng chứng, quy tắc áp dụng và các điểm cần con người phê duyệt.

Ví dụ output có cấu trúc:

```json
{
  "decision": "Không hoàn điểm",
  "appliedRules": ["VIP_RATE_5_PERCENT", "FULL_REFUND_EXCLUSION"],
  "evidenceIds": ["loyalty-policy-v3#section-4.2"],
  "calculation": null,
  "needsHumanReview": false
}
```

**Verifier nên kiểm tra:**

- Phép tính bằng calculator hoặc code.
- Rule bằng rule engine hoặc test case.
- Claim bằng citation/evidence.
- Output bằng JSON Schema.
- Action bằng authorization, idempotency và business validation.

**Bài toán giải quyết:** hạn chế reasoning “nghe hợp lý” nhưng không kiểm chứng được; giảm lỗi format; giữ trace đủ để debug nhưng không làm lộ dữ liệu nhạy cảm.

**So sánh với việc hiển thị CoT đầy đủ:** bản giải thích dài có thể hữu ích trong nghiên cứu hoặc debug, nhưng trong sản phẩm thường nên hiển thị **lý do ngắn, bằng chứng và kết quả kiểm tra**. Điều này dễ audit hơn và giảm token, latency cũng như rủi ro ghi log thông tin không cần thiết.

---

## B. 8 câu hỏi phổ biến mở rộng

### 6. Few-shot CoT và Zero-shot CoT nên chọn loại nào?

**Trả lời:** Hãy bắt đầu bằng zero-shot để tạo baseline nhanh, sau đó chuyển sang few-shot khi cần tăng tính ổn định hoặc dạy model một kiểu phân tích đặc thù.

**Zero-shot CoT phù hợp khi:**

- Chưa có dataset ví dụ tốt.
- Bài toán quen thuộc như toán, logic, phân tích điều kiện chung.
- Muốn kiểm tra model có đủ khả năng trước khi đầu tư prompt engineering.

**Few-shot CoT phù hợp khi:**

- Thuật ngữ và quy tắc thuộc domain riêng.
- Cần model xét điều kiện theo thứ tự cụ thể.
- Cần output có schema, nhãn hoặc quy ước rõ ràng.
- Zero-shot cho kết quả dao động giữa các lần chạy.

**Cách thử nghiệm:**

1. Đo direct answer làm baseline.
2. Thử zero-shot CoT với cùng bộ câu hỏi.
3. Thêm 3–8 ví dụ đa dạng, đúng và ngắn gọn.
4. So sánh accuracy, token, latency và lỗi theo nhóm.
5. Chỉ giữ few-shot nếu lợi ích vượt chi phí context.

**Bài toán giải quyết:** tránh viết hàng loạt ví dụ trước khi biết CoT có thực sự giúp use case hay không.

Nguồn: [Zero-Shot Reasoners](https://arxiv.org/abs/2205.11916) và [CoT Prompting](https://arxiv.org/abs/2201.11903).

---

### 7. Self-Consistency là gì, và khi nào nên kết hợp với CoT?

**Trả lời:** **Self-Consistency** tạo nhiều chuỗi reasoning khác nhau cho cùng một câu hỏi, sau đó chọn đáp án xuất hiện nhất quán nhất thay vì chỉ dùng một đường sinh greedy duy nhất.

```text
Question
  → sample reasoning path A → answer X
  → sample reasoning path B → answer X
  → sample reasoning path C → answer Y
  → aggregate/vote → choose X
```

Kỹ thuật này hữu ích vì một bài toán có thể có nhiều cách giải đúng. Nếu nhiều đường độc lập hội tụ về cùng kết quả, độ tin cậy thường tốt hơn một lần sinh đơn lẻ.

**Áp dụng khi nào?**

- Bài toán toán học, logic hoặc multiple-choice có đáp án rõ.
- Sai số của một lần suy luận còn cao nhưng có thể chạy nhiều sample.
- Có ngân sách inference và độ trễ cho phép.

**Không phù hợp khi:**

- Output là sáng tạo hoặc không có cách tổng hợp rõ ràng.
- Các sample cùng bị dẫn dắt bởi dữ liệu sai hoặc prompt bias.
- Tác vụ realtime có ngân sách latency thấp.
- Một lần gọi tool chính xác đã đủ giải quyết bài toán.

**Bài toán giải quyết:** giảm phụ thuộc vào một reasoning path tình cờ, nhưng đổi lại chi phí token và latency tăng gần theo số sample.

Nguồn: [Self-Consistency Improves Chain of Thought Reasoning](https://arxiv.org/abs/2203.11171).

---

### 8. Viết ví dụ Few-shot CoT tốt như thế nào?

**Trả lời:** Ví dụ tốt phải dạy model **quy trình ra quyết định**, không chỉ dạy văn phong dài dòng.

Checklist cho exemplar:

- Đáp án và từng bước phải chính xác.
- Đại diện cho nhiều nhóm lỗi: ngoại lệ, dữ kiện thừa, thiếu dữ kiện, mâu thuẫn.
- Ngắn nhất có thể nhưng không bỏ bước quyết định.
- Dùng cùng thuật ngữ và schema với production.
- Có ví dụ từ chối hoặc yêu cầu thêm dữ liệu khi đầu vào không đủ.
- Không để vị trí đáp án, độ dài hoặc câu chữ tạo shortcut ngoài ý muốn.
- Không dùng dữ liệu nhạy cảm thật trong prompt mẫu.

Ví dụ kém:

```text
Hãy suy nghĩ thật kỹ, phân tích sâu và trả lời chính xác nhất.
```

Ví dụ tốt hơn:

```text
1. Xác định tất cả điều kiện bắt buộc.
2. Kiểm tra ngoại lệ trước quy tắc mặc định.
3. Chỉ dùng dữ kiện trong POLICY_CONTEXT.
4. Nếu thiếu ngày hiệu lực hoặc loại khách hàng, trả NEED_MORE_INFORMATION.
5. Trả JSON theo schema đã cho.
```

**Áp dụng khi nào?** Khi model hay bỏ cùng một loại điều kiện, dùng sai thứ tự ưu tiên, hoặc output không ổn định.

**Bài toán giải quyết:** giảm ambiguity của prompt và biến “reasoning tốt” thành hành vi có thể quan sát, kiểm thử.

**So sánh với thêm nhiều ví dụ:** chất lượng và độ phủ quan trọng hơn số lượng. Nhiều ví dụ giống nhau chỉ làm tăng context mà không tăng khả năng tổng quát.

---

### 9. Kết hợp CoT với RAG, search và tool calling như thế nào?

**Trả lời:** CoT xử lý **cách suy luận**; RAG/search cung cấp **bằng chứng**; tool/API cung cấp **dữ liệu hoặc hành động chính xác**. Ba lớp này bổ sung nhau.

Mẫu workflow:

```text
User question
  → identify missing knowledge/data
  → retrieve documents or call API
  → reason over returned evidence
  → calculate/validate with tools
  → answer with citation and result
```

Ví dụ CSKH:

1. RAG lấy chính sách đổi trả đang có hiệu lực.
2. API lấy trạng thái đơn hàng và loại sản phẩm.
3. CoT kiểm tra từng điều kiện và ngoại lệ.
4. Rule engine hoặc code xác nhận eligibility.
5. LLM giải thích ngắn gọn kèm citation.

**Áp dụng khi nào?** Khi bài toán vừa cần tri thức riêng, vừa cần dữ liệu realtime hoặc phép tính chính xác.

**Bài toán giải quyết:** CoT thuần túy dễ bịa dữ kiện; RAG thuần túy có thể lấy đúng tài liệu nhưng model vẫn áp dụng sai điều kiện; tool thuần túy trả dữ liệu nhưng không giải thích được chính sách. Kết hợp ba lớp giúp tách rõ nguồn sự thật và vai trò reasoning.

**Nguyên tắc an toàn:**

- Không để reasoning tự tạo tham số nhạy cảm mà không validate.
- ACL phải áp dụng trước khi evidence vào prompt.
- Action có side effect cần authorization, idempotency và confirmation phù hợp.
- Citation phải trỏ tới evidence thực sự được dùng.

Nguồn liên quan: [ReAct](https://arxiv.org/abs/2210.03629).

---

### 10. Vì sao CoT vẫn trả lời sai dù các bước nhìn có vẻ hợp lý?

**Trả lời:** CoT là chuỗi text do model sinh, nên mỗi bước vẫn có thể mắc lỗi. Các lỗi thường gặp gồm:

- **Sai từ bước đầu:** hiểu nhầm yêu cầu hoặc chọn sai công thức.
- **Error propagation:** bước sau dựa trên kết quả sai của bước trước.
- **Missing constraint:** bỏ qua ngoại lệ, quyền, phiên bản hoặc thời gian hiệu lực.
- **Hallucinated premise:** tự thêm dữ kiện không có trong input.
- **Arithmetic slip:** logic đúng nhưng phép tính sai.
- **Rationalization:** chọn đáp án trước rồi tạo lời giải thích nghe hợp lý.
- **Prompt bias:** bị ảnh hưởng bởi vị trí đáp án, ví dụ mẫu hoặc gợi ý không liên quan.

**Cách giảm lỗi:**

- Tách bước lập kế hoạch và bước thực thi.
- Dùng calculator/code cho tính toán.
- Yêu cầu trích `evidenceId` cho từng claim quan trọng.
- Thêm kiểm tra ngược: thay dữ kiện, kiểm tra boundary và counterexample.
- Chạy self-consistency hoặc verifier độc lập cho bài toán phù hợp.
- Dùng unit test/golden set thay vì đánh giá qua vài demo.

**Bài toán giải quyết:** nhận thức đúng rằng “có giải thích” không đồng nghĩa “đã được chứng minh”.

**So sánh với không dùng CoT:** CoT giúp nhìn thấy nhiều lỗi trung gian hơn, nhưng cũng có thể làm người đọc tin quá mức vào một lời giải thích trôi chảy.

---

### 11. CoT có phải là lời giải thích trung thực về suy nghĩ bên trong model không?

**Trả lời:** Không thể mặc định như vậy. CoT nhìn thấy là một chuỗi ngôn ngữ được tạo ra; nó có thể hữu ích để kiểm tra và debug, nhưng không bảo đảm phản ánh đầy đủ nguyên nhân nội bộ dẫn đến đáp án. Nghiên cứu đã cho thấy model có thể bị ảnh hưởng bởi các tín hiệu thiên lệch trong prompt nhưng không nhắc đến tín hiệu đó trong phần giải thích, hoặc tạo rationale hợp lý hóa đáp án sai.

Vì vậy, cần phân biệt:

- **Plausibility:** lời giải thích nghe hợp lý với con người.
- **Correctness:** kết quả và các bước có đúng không.
- **Faithfulness:** lời giải thích có thực sự phản ánh yếu tố đã chi phối dự đoán không.
- **Verifiability:** các claim có thể kiểm tra bằng dữ liệu, code hoặc quy tắc không.

**Áp dụng thực tế:** dùng CoT như một tín hiệu hỗ trợ, không phải bằng chứng duy nhất cho audit, compliance hay safety. Với người dùng cuối, ưu tiên cung cấp **bằng chứng, phép tính, điều kiện áp dụng và giới hạn** thay vì tuyên bố đây là toàn bộ “suy nghĩ thật” của model.

**Bài toán giải quyết:** tránh nhầm lẫn giữa khả năng tạo giải thích và tính minh bạch cơ chế.

Nguồn: [Language Models Don't Always Say What They Think](https://arxiv.org/abs/2305.04388) và [Reasoning Models Don't Always Say What They Think](https://arxiv.org/abs/2505.05410).

---

### 12. Tối ưu token, latency, chi phí và bảo mật khi dùng CoT thế nào?

**Trả lời:** Không phải request nào cũng cần cùng mức reasoning. Nên có **routing theo độ phức tạp và rủi ro**.

Chiến lược:

1. **Complexity classifier:** câu đơn giản trả lời trực tiếp; câu nhiều bước mới dùng reasoning sâu.
2. **Reasoning budget:** giới hạn số bước, độ dài hoặc số nhánh theo loại tác vụ.
3. **Structured output:** yêu cầu các trường ngắn thay vì một bài giải dài.
4. **Tool-first:** phép tính, query và validation giao cho công cụ thay vì dùng nhiều token để mô phỏng.
5. **Cache:** cache retrieval, tool result hoặc kết quả cho câu hỏi ổn định; không cache mù dữ liệu cá nhân.
6. **Early exit:** dừng khi thiếu dữ kiện bắt buộc hoặc verifier đã xác nhận đáp án.
7. **Selective retry:** chỉ chạy self-consistency cho case không chắc chắn hoặc giá trị cao.
8. **Safe logging:** log `decision`, `evidenceIds`, error code và metric; redact PII, secret và dữ liệu không cần thiết.

**Áp dụng khi nào?** Mọi hệ thống production có traffic đáng kể, SLA hoặc dữ liệu nhạy cảm.

**Bài toán giải quyết:** tránh chi phí và latency tăng tuyến tính theo độ dài reasoning, đồng thời giảm rủi ro lưu trace chứa thông tin nhạy cảm.

**So sánh:** một CoT dài cho mọi request đơn giản để phát triển nhưng khó scale; routing nhiều mức phức tạp cần thêm logic nhưng cho hiệu quả vận hành tốt hơn.

---

### 13. Đánh giá CoT bằng những metric và bộ kiểm thử nào?

**Trả lời:** Không nên chỉ chấm phần văn bản reasoning “có hay không”. Cần đo cả chất lượng cuối, độ đúng của bước trung gian và chi phí vận hành.

| Nhóm metric | Ví dụ | Ý nghĩa |
| --- | --- | --- |
| Kết quả cuối | Exact Match, accuracy, F1, pass rate | Đáp án cuối có đúng không. |
| Bước trung gian | Step correctness, rule coverage, calculation validity | Có bước sai hoặc bỏ điều kiện không. |
| Grounding | Citation correctness, evidence coverage, unsupported claim rate | Reasoning có bám nguồn không. |
| Tính ổn định | Consistency across runs, variance theo seed/model | Kết quả có dao động không. |
| Tool use | Tool selection accuracy, argument validity, execution success | Model có gọi đúng công cụ và tham số không. |
| An toàn | PII leakage, unauthorized evidence, unsafe action rate | Có vi phạm quyền hoặc làm lộ dữ liệu không. |
| Vận hành | Tokens, latency p50/p95, cost, retry rate | Chất lượng có tương xứng chi phí không. |

Một evaluation set tốt nên có:

- Case bình thường và boundary.
- Ngoại lệ và quy tắc xung đột.
- Câu hỏi thiếu dữ kiện cần từ chối hoặc hỏi lại.
- Dữ kiện thừa để kiểm tra model không bị nhiễu.
- Prompt injection hoặc evidence không đáng tin.
- Case tiếng Việt, thuật ngữ nội bộ và lỗi chính tả thực tế.
- Ground truth về đáp án, rule/evidence và action mong đợi.

Quy trình regression:

```text
Dataset version
  → run direct baseline
  → run CoT configuration
  → verify answer + steps + evidence/tools
  → compare quality, latency and cost
  → review failures by category
  → deploy only when critical slices do not regress
```

**Áp dụng khi nào?** Trước khi thay model, prompt, exemplar, tool, temperature, retrieval hoặc reasoning budget.

**Bài toán giải quyết:** ngăn việc tối ưu một vài demo nhưng làm giảm chất lượng ở nhóm case khác; phát hiện trade-off giữa accuracy và chi phí.

---

## Mẫu quyết định nhanh

| Tình huống | Cách tiếp cận đề xuất |
| --- | --- |
| FAQ đơn giản, câu trả lời nằm rõ trong một đoạn | RAG/search + trả lời trực tiếp + citation. |
| Tính giá qua nhiều bước nhưng công thức rõ | CoT ngắn + calculator/code + kiểm tra schema. |
| Chính sách có nhiều ngoại lệ | RAG + structured reasoning + rule verifier. |
| Lập kế hoạch với nhiều phương án | Least-to-Most hoặc Tree of Thoughts có tiêu chí chấm. |
| Agent cần tra cứu và thao tác hệ thống | ReAct/tool calling + authorization + audit. |
| Bài toán logic có đáp án duy nhất, giá trị cao | CoT + self-consistency + verifier. |
| Câu hỏi đơn giản, latency thấp | Direct answer; không bật CoT mặc định. |

## Nguồn tham khảo

1. Jason Wei và cộng sự, [Chain-of-Thought Prompting Elicits Reasoning in Large Language Models](https://arxiv.org/abs/2201.11903), 2022.
2. Takeshi Kojima và cộng sự, [Large Language Models are Zero-Shot Reasoners](https://arxiv.org/abs/2205.11916), 2022.
3. Xuezhi Wang và cộng sự, [Self-Consistency Improves Chain of Thought Reasoning in Language Models](https://arxiv.org/abs/2203.11171), 2022.
4. Denny Zhou và cộng sự, [Least-to-Most Prompting Enables Complex Reasoning in Large Language Models](https://arxiv.org/abs/2205.10625), 2022.
5. Shunyu Yao và cộng sự, [Tree of Thoughts: Deliberate Problem Solving with Large Language Models](https://arxiv.org/abs/2305.10601), 2023.
6. Shunyu Yao và cộng sự, [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629), 2022.
7. Wenhu Chen và cộng sự, [Program of Thoughts Prompting](https://arxiv.org/abs/2211.12588), 2022.
8. Miles Turpin và cộng sự, [Language Models Don't Always Say What They Think](https://arxiv.org/abs/2305.04388), 2023.
9. Yanda Chen và cộng sự, [Reasoning Models Don't Always Say What They Think](https://arxiv.org/abs/2505.05410), 2025.
