# Model Context Protocol (MCP) trong tối ưu context và luồng DEV

> [Knowledge Tree](../../../README.md) / [AI](../../README.md) / [LLM](../README.md) / **Model Context Protocol (MCP)**

**Model Context Protocol (MCP)** là chuẩn mở để ứng dụng AI kết nối thống nhất với dữ liệu, công cụ và workflow bên ngoài. Trong luồng phát triển phần mềm (*development workflow — DEV workflow*), MCP có thể nối coding agent hoặc IDE với source code, GitHub/GitLab, issue tracker, tài liệu kiến trúc, CI/CD, log, APM, database và dịch vụ nội bộ.

Điểm quan trọng nhất:

> **MCP không tự động làm prompt ngắn hơn.** MCP chỉ cung cấp giao thức khám phá và gọi capability. Việc giảm token phụ thuộc vào cách MCP host chọn đúng tool/resource, chỉ nạp schema khi cần, lọc dữ liệu ngoài LLM và chỉ trả kết quả tối thiểu về context.

Luồng nên hướng tới:

```text
Yêu cầu DEV
  → tìm capability liên quan
  → nạp schema của vài tool cần thiết
  → đọc đúng file/log/issue cần dùng
  → xử lý, lọc hoặc tổng hợp ngoài context LLM
  → trả evidence ngắn gọn
  → sửa code / test / tạo PR với approval
```

## Điều hướng

- **Node cha:** [Mô hình ngôn ngữ lớn (LLM)](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** [Minimal Reproducible Example (MRE)](../mre/README.md)
- **Khái niệm sau trong cùng nhánh:** Chưa có.

## A. 5 câu hỏi cốt lõi

### 1. MCP là gì và đóng vai trò nào trong luồng DEV?

**Trả lời:** MCP chuẩn hóa cách một **MCP host** như IDE, coding agent hoặc ứng dụng AI làm việc với nhiều **MCP server**. Server có thể cung cấp ba primitive chính:

| Primitive | Vai trò | Ví dụ trong DEV | Bên thường quyết định sử dụng |
| --- | --- | --- | --- |
| **Tools** | Hàm có thể thực thi hành động hoặc truy vấn. | Tìm code, đọc CI log, chạy test, tạo issue, cập nhật PR. | Model/host, dưới chính sách approval. |
| **Resources** | Dữ liệu ngữ cảnh có URI và có thể đọc theo nhu cầu. | File nguồn, ADR, schema DB, runbook, log snapshot. | Ứng dụng/host. |
| **Prompts** | Template workflow tái sử dụng. | Review PR, điều tra incident, tạo kế hoạch migration. | Người dùng hoặc UI. |

MCP dùng kiến trúc host–client–server và JSON-RPC để capability có thể được khám phá, thương lượng và gọi theo cùng một giao diện, thay vì mỗi IDE phải tích hợp riêng từng API.

**Áp dụng khi nào?**

- Coding agent cần làm việc với nhiều hệ thống: repository, issue, CI, log và tài liệu.
- Nhiều IDE hoặc model cần dùng lại cùng một lớp tích hợp.
- Muốn tách logic nghiệp vụ và quyền truy cập khỏi prompt.
- Cần tool có schema rõ ràng, output có cấu trúc và có thể kiểm thử.

**Bài toán giải quyết:** giảm tích hợp point-to-point, hạn chế copy/paste thủ công, thống nhất cách agent lấy evidence và thực hiện hành động trong chu trình phát triển.

**So sánh:**

| Cách tích hợp | Điểm mạnh | Hạn chế | Khi chọn |
| --- | --- | --- | --- |
| Copy/paste vào prompt | Bắt đầu nhanh. | Tốn token, dễ cũ, khó audit. | Tác vụ nhỏ, một lần. |
| Gọi API trực tiếp trong agent | Kiểm soát sâu. | Mỗi hệ thống một adapter riêng, khó tái sử dụng. | Sản phẩm hẹp, ít integration. |
| Plugin độc quyền IDE | UX tốt trong một nền tảng. | Khó chuyển host/model. | Đội đã khóa vào một IDE. |
| **MCP** | Chuẩn hóa discovery, schema, resource và tool call. | Cần host/server tốt và quản trị bảo mật. | Hệ sinh thái nhiều tool, nhiều host. |

Nguồn: [MCP — What is MCP?](https://modelcontextprotocol.io/docs/getting-started/intro) · [MCP — Architecture overview](https://modelcontextprotocol.io/docs/2026-07-28/learn/architecture).

---

### 2. MCP giúp giảm token context prompt bằng những cơ chế nào?

**Trả lời:** Có bốn cơ chế chính, nhưng đều phải được host triển khai đúng.

#### 2.1. Progressive discovery — khám phá tăng dần

Host không đưa schema của mọi tool vào system prompt. Thay vào đó:

1. Cung cấp catalog hoặc meta-tool nhỏ như `search_tools`.
2. Tìm vài capability phù hợp với nhiệm vụ.
3. Chỉ nạp schema đầy đủ của tool được chọn.
4. Bỏ tool không còn cần khỏi context ở bước sau nếu host hỗ trợ.

Tài liệu MCP Client Best Practices mô tả trường hợp nạp toàn bộ tool definition có thể chiếm phần lớn context; mô hình **catalog → inspect → execute** giảm phần schema phải đưa cho model.

#### 2.2. Resource on demand — đọc dữ liệu theo nhu cầu

Resources hỗ trợ list, read, URI template, pagination và subscription. Host có thể tìm đúng file, section, log range hoặc DB schema cần thiết thay vì đính kèm toàn bộ repository hay tài liệu.

#### 2.3. Programmatic tool calling / code mode

Với chuỗi nhiều tool, model viết một script ngắn chạy trong sandbox. Dữ liệu trung gian được lọc, join, deduplicate hoặc aggregate trong sandbox; chỉ kết quả cuối quay về context. Cách này đặc biệt hiệu quả với hàng nghìn log line, test result hoặc record.

#### 2.4. Structured and bounded output

Tool nên trả JSON theo `outputSchema`, có filter, cursor, limit và summary. Khi cần dữ liệu lớn, trả resource URI hoặc handle để model đọc tiếp theo lát nhỏ, thay vì trả toàn bộ payload vào một tool result.

```text
Không tối ưu:
tools/list tất cả → đưa 500 schema vào prompt
→ get_logs trả 50.000 dòng → LLM tự lọc
→ đọc cả repository → sửa 1 file

Tối ưu:
search_tools("CI failure") → inspect 2 tool
→ get_failed_jobs(commit, limit=10)
→ get_log_excerpt(job, around="first root error")
→ read_file(path, startLine, endLine)
→ patch + targeted tests
```

**Áp dụng khi nào?**

- Có hàng chục MCP server hoặc hàng trăm tool.
- Log, codebase, tài liệu hoặc DB lớn hơn đáng kể context window.
- Workflow phải gọi nhiều tool liên tiếp.
- Chi phí input token và độ trễ tăng theo số vòng tool call.

**Bài toán giải quyết:** giảm token cho tool definition, dữ liệu thô và kết quả trung gian; giữ context tập trung vào mục tiêu, evidence và quyết định.

**Giới hạn:** MCP specification chỉ chuẩn hóa trao đổi context; nó không bắt buộc host phải tối ưu context. Host ngây thơ vẫn có thể nạp tất cả tool upfront và làm token tăng.

Nguồn: [MCP — Client Best Practices](https://modelcontextprotocol.io/docs/2026-07-28/develop/clients/client-best-practices) · [MCP — Resources](https://modelcontextprotocol.io/specification/2025-11-25/server/resources).

---

### 3. Kiến trúc MCP nào phù hợp cho một luồng DEV hoàn chỉnh?

**Trả lời:** Nên tách thành ba lớp.

```text
┌────────────────────────────────────────────────────────────┐
│ MCP Host: IDE / coding agent / developer portal            │
│ - quản lý context budget                                   │
│ - discovery, approval, policy, audit                       │
│ - lập kế hoạch và tổng hợp evidence                        │
└───────────────┬────────────────────────────────────────────┘
                │ MCP clients
     ┌──────────┼───────────┬───────────┬───────────┐
     ▼          ▼           ▼           ▼           ▼
  Repo MCP   Work MCP    CI MCP      Docs MCP   Observability MCP
  code/git   issue/PR    test/build   ADR/API    logs/traces/metrics
     │          │           │           │           │
     └──────────┴───────────┴───────────┴───────────┘
                       enterprise systems
```

#### Read path — đường đọc evidence

1. Đọc issue và acceptance criteria.
2. Tìm symbol, dependency và file liên quan.
3. Đọc ADR, API contract hoặc schema cần thiết.
4. Lấy test failure, log excerpt hoặc trace liên quan.
5. Tạo plan có dẫn chứng.

#### Write path — đường thay đổi

1. Sửa file trong phạm vi repository.
2. Chạy formatter, static analysis và targeted tests.
3. Hiển thị diff và kết quả kiểm tra.
4. Xin approval cho hành động có side effect đáng kể.
5. Commit, push hoặc tạo PR.
6. Theo dõi CI và liên kết evidence vào PR.

#### Nguyên tắc ranh giới

- Tool đọc và tool ghi nên tách tên, scope và credential.
- Mặc định chỉ đọc; nâng quyền theo tác vụ.
- Output lớn phải hỗ trợ filter/pagination.
- Host giữ token/credential, không đưa vào prompt hoặc model-generated code.
- Source of truth vẫn là Git, issue tracker, CI và observability system; hội thoại chỉ là lớp điều phối.

**Áp dụng khi nào?** Dự án có nhiều repository, microservice, pipeline CI/CD hoặc yêu cầu audit/compliance.

**Bài toán giải quyết:** biến agent từ chatbot copy/paste thành workflow có evidence, quyền hạn, trace và kết quả có thể kiểm tra.

**So sánh với một “mega server”:**

| Thiết kế | Điểm mạnh | Rủi ro |
| --- | --- | --- |
| Một MCP server cho mọi hệ thống | Dễ cài ban đầu. | Blast radius lớn, schema khổng lồ, credential khó tách. |
| Server theo hệ thống | Ranh giới rõ. | Nhiều connection hơn. |
| Server theo bounded context | Phù hợp nghiệp vụ, dễ policy. | Cần thiết kế ownership. |
| **Khuyến nghị** | Server theo trust boundary và ownership, host discovery theo nhu cầu. | Cần catalog tốt. |

Nguồn: [MCP — Architecture overview](https://modelcontextprotocol.io/docs/2026-07-28/learn/architecture) · [MCP — Tools](https://modelcontextprotocol.io/specification/2025-06-18/server/tools).

---

### 4. Ngoài giảm token, MCP còn đem lại tác dụng gì cho DEV?

**Trả lời:**

| Tác dụng | Cách MCP hỗ trợ | Giá trị cho DEV |
| --- | --- | --- |
| **Chuẩn hóa tích hợp** | Cùng primitive và lifecycle cho nhiều server. | Giảm adapter độc quyền giữa agent và hệ thống. |
| **Portability** | Server có thể được nhiều host dùng lại. | Dễ đổi IDE/model hơn so với plugin đóng. |
| **Capability discovery** | `tools/list`, `resources/list`, prompt discovery và `listChanged`. | Agent biết capability hiện có thay vì hard-code vào prompt. |
| **Grounding theo thời gian thực** | Đọc source of truth lúc chạy. | Tránh dựa vào snapshot copy trong prompt. |
| **Hành động có cấu trúc** | Input/output schema và tool call. | Ít lỗi định dạng hơn shell/text tự do. |
| **Quan sát và debug** | Logging, progress, notifications, Inspector. | Dễ test contract, xem message và tìm lỗi tích hợp. |
| **Quản trị quyền** | Host kiểm soát consent, credential và approval. | Giảm quyền dư thừa và hành động ngoài ý muốn. |
| **Workflow tái sử dụng** | Prompt template + tool/resource contract. | Chuẩn hóa review, incident, migration, release. |
| **Tách model khỏi integration** | Server không phụ thuộc một model cụ thể. | Thay model mà không viết lại toàn bộ connector. |
| **Hỗ trợ tác vụ dài** | Progress, cancellation và task-oriented pattern tùy phiên bản/capability. | UX tốt hơn cho test, build, scan và phân tích dài. |

**Áp dụng khi nào?** Khi muốn đưa coding agent vào quy trình nhóm, không chỉ dùng cho autocomplete cá nhân.

**Bài toán giải quyết:** portability, governance, testability, observability và phối hợp nhiều hệ thống.

Nguồn: [MCP — MCP Inspector](https://modelcontextprotocol.io/docs/2026-07-28/tools/inspector) · [MCP — Cancellation](https://modelcontextprotocol.io/specification/2025-11-25/basic/utilities/cancellation).

---

### 5. Khi nào nên dùng MCP, và khi nào direct API, CLI, RAG hay Agent Skills phù hợp hơn?

**Trả lời:** MCP là lớp giao tiếp, không phải lời giải thay thế mọi thành phần.

| Lựa chọn | Phù hợp nhất | Khác MCP |
| --- | --- | --- |
| **Direct API/SDK** | Một ứng dụng gọi một dịch vụ ổn định, cần tối ưu sâu. | Ít lớp hơn nhưng coupling cao và khó tái sử dụng giữa host. |
| **CLI** | Automation deterministic, script CI hoặc thao tác dev quen thuộc. | Model phải hiểu text CLI; MCP có schema và discovery tốt hơn. |
| **RAG** | Tìm và đưa tri thức liên quan vào model. | RAG tập trung retrieval; MCP có thể expose retriever như resource/tool và còn thực hiện action. |
| **Agent Skills / instruction package** | Đóng gói quy trình, quy ước và kiến thức thực hiện nhiệm vụ. | Skill mô tả “làm thế nào”; MCP cung cấp “kết nối và capability”. |
| **Function calling riêng provider** | Ít tool, chỉ dùng một model platform. | Nhanh nhưng interface gắn với provider; MCP chuẩn hóa lớp server. |
| **MCP** | Nhiều host, nhiều hệ thống, cần discovery, policy và tái sử dụng. | Có thêm độ phức tạp vận hành và bảo mật. |

**Nên dùng MCP khi:**

- Integration cần dùng lại giữa nhiều agent/IDE.
- Tool hoặc dữ liệu thay đổi động.
- Cần ranh giới quyền, approval và audit.
- Có nhiều nguồn context và hành động ngoài LLM.

**Không nên dùng chỉ vì “đang là xu hướng” khi:**

- Workflow chỉ có một API đơn giản.
- Script deterministic không cần LLM.
- Không có owner vận hành server và security patch.
- Host không hỗ trợ discovery hoặc output control đủ tốt.
- Chi phí thêm protocol lớn hơn lợi ích tích hợp.

**Bài toán giải quyết:** lựa chọn đúng abstraction, tránh biến MCP thành lớp proxy không tạo giá trị.

---

## B. 8 câu hỏi phổ biến

### 6. Thiết kế MCP server thế nào để thực sự tiết kiệm token?

**Trả lời:** Tối ưu đồng thời **tool catalog**, **input contract** và **output contract**.

#### Checklist tool

- Tên cụ thể theo domain, ví dụ `ci_get_failed_jobs`, không dùng `execute`.
- Mô tả một đến ba câu, nêu rõ khi dùng và khi không dùng.
- Input schema nhỏ, typed, có enum/default khi phù hợp.
- Tách read và write tool.
- Gộp các thao tác quá vụn thành một primitive nghiệp vụ hợp lý, nhưng không tạo “god tool”.
- Output có `outputSchema`, summary và stable IDs.
- Có `limit`, `cursor`, `fields`, `since`, `path`, `lineRange`, `severity`.
- Trả resource link/URI cho payload lớn.
- Không lặp lại prompt dài trong mọi tool description.
- Không trả secret, stack dump hoặc toàn bộ object khi không cần.

Ví dụ:

```json
{
  "name": "ci_get_failure_evidence",
  "description": "Lấy lỗi gốc đầu tiên và các đoạn log lân cận của job CI thất bại.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "jobId": {"type": "string"},
      "contextLines": {"type": "integer", "default": 30, "maximum": 200}
    },
    "required": ["jobId"]
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "rootError": {"type": "string"},
      "excerpt": {"type": "string"},
      "fullLogUri": {"type": "string"}
    },
    "required": ["rootError", "excerpt"]
  }
}
```

**Áp dụng khi nào?** Ngay từ lúc thiết kế server; sửa sau khi tool đã phổ biến thường khó vì compatibility.

**Bài toán giải quyết:** giảm schema token, tránh payload thừa và tăng xác suất model chọn/call tool đúng.

Nguồn: [MCP — Tools](https://modelcontextprotocol.io/specification/2025-06-18/server/tools).

---

### 7. MCP hỗ trợ những bước nào trong Software Development Life Cycle?

**Trả lời:**

| Giai đoạn | Resource/tool MCP có thể dùng | Kết quả mong đợi |
| --- | --- | --- |
| Backlog | Đọc issue, requirement, comment, design link. | Scope và acceptance criteria rõ. |
| Analysis | Tìm symbol, dependency, owner, ADR, schema. | Impact map có evidence. |
| Design | Đọc template, tạo draft ADR, kiểm tra constraint. | Thiết kế nhất quán kiến trúc. |
| Implementation | Đọc/sửa file, refactor, generate code có kiểm soát. | Diff nhỏ, truy vết được. |
| Test | Chạy targeted test, coverage, lint, security scan. | Bằng chứng pass/fail. |
| Review | Đọc PR diff/comment, phát hiện risk, đề xuất patch. | Review tập trung và actionable. |
| CI/CD | Đọc check, log, artifact; retry/cancel theo quyền. | Điều tra failure nhanh. |
| Release | Đọc release gate, changelog; tạo release note. | Quyết định release có căn cứ. |
| Operations | Query log/trace/metric/runbook, tạo incident. | Rút ngắn MTTR và liên kết lỗi về code. |

Một workflow tốt phải truyền **identifier** thay vì payload lớn: `issueId`, `commitSha`, `jobId`, `traceId`, `filePath`, `resourceUri`.

**Áp dụng khi nào?** Khi agent cần theo tác vụ từ yêu cầu đến PR hoặc từ incident về bản sửa.

**Bài toán giải quyết:** giảm đứt gãy giữa công cụ, tránh con người copy dữ liệu qua nhiều màn hình và duy trì lineage.

---

### 8. Phân biệt Tools, Resources và Prompts trong một MCP server DEV như thế nào?

**Trả lời:**

| Câu hỏi thiết kế | Chọn |
| --- | --- |
| Đây là dữ liệu để đọc và có identity/URI? | **Resource** |
| Đây là hành động, truy vấn có tham số hoặc side effect? | **Tool** |
| Đây là workflow/template người dùng chủ động chọn? | **Prompt** |

Ví dụ với code review:

- Resource: `git://repo/commit/{sha}/diff`.
- Tool: `github_submit_review`, `repo_find_references`.
- Prompt: `review-security-sensitive-change`.

**Sai lầm phổ biến:**

- Dùng tool `get_everything` trả toàn bộ repository.
- Dùng prompt để chứa credential hoặc dữ liệu động.
- Biến resource read-only thành action ẩn.
- Gộp hành động ghi vào tool đọc khiến approval không rõ.

**Áp dụng khi nào?** Khi thiết kế capability mới hoặc chia lại một server quá lớn.

**Bài toán giải quyết:** làm rõ control, side effect, caching, discovery và security policy.

Nguồn: [MCP — Understanding servers](https://modelcontextprotocol.io/docs/learn/server-concepts).

---

### 9. Đo hiệu quả giảm token của MCP trong DEV bằng chỉ số nào?

**Trả lời:** Không chỉ đo tổng token của một request; cần đo theo task end-to-end.

| Nhóm | Chỉ số |
| --- | --- |
| Context | Tool-definition tokens, resource tokens, tool-result tokens, peak context utilization. |
| Chi phí | Input/output token cost, tool infrastructure cost, cost per completed task. |
| Hiệu năng | Time to first useful action, tổng latency, số round trip, tool execution time. |
| Chất lượng | Task success rate, first-pass success, test pass, grounded evidence rate. |
| Tool use | Tool selection precision, invalid arguments, retry count, unnecessary calls. |
| An toàn | Write calls cần approval, denied calls, scope violations, secret exposure. |
| DEV outcome | Cycle time issue→PR, review rework, CI recovery time, incident MTTR. |

#### Thiết kế A/B test

1. Chọn tập task đại diện: bug fix, feature nhỏ, PR review, CI failure, incident.
2. Baseline: nạp tool upfront và direct tool calling.
3. Variant: progressive discovery + bounded output + code mode khi cần.
4. Giữ model, repository snapshot và quyền giống nhau.
5. Chấm cả success và token; không tối ưu token bằng cách làm giảm chất lượng.
6. Phân tích theo loại task và kích thước tool catalog.

**Áp dụng khi nào?** Trước và sau khi thêm nhiều MCP server hoặc thay host strategy.

**Bài toán giải quyết:** chứng minh lợi ích thật, phát hiện tối ưu token nhưng tăng latency, retry hoặc lỗi.

---

### 10. Bảo mật MCP trong luồng DEV cần những lớp nào?

**Trả lời:** MCP server có thể đọc source code, secret, production log và thực hiện side effect, nên phải xem nó như một integration có đặc quyền.

#### Lớp bắt buộc

- **Least privilege:** credential theo server, tool và môi trường.
- **Read-only by default:** write tool bật theo tác vụ.
- **Human in the loop:** xác nhận commit, push, merge, deploy, delete, DB write.
- **OS/container sandbox:** không coi workspace hint hoặc roots là security boundary.
- **Secret isolation:** token nằm ở host/broker; không đưa vào prompt hoặc sandbox code.
- **Input/output validation:** JSON Schema, path validation, size limit và allowlist.
- **Prompt-injection defense:** coi content từ issue, README, log và tool output là dữ liệu không tin cậy.
- **Audit:** user, tool, arguments đã chuẩn hóa, target, result, timestamp, approval.
- **Network policy:** local server không mặc nhiên được truy cập Internet.
- **Supply-chain control:** pin version, verify publisher, scan dependency và update định kỳ.

> Từ phiên bản MCP 2026-07-28, `roots` đã được đánh dấu deprecated; ngay cả trước đó, roots chỉ là thông tin định hướng, không phải cơ chế cưỡng chế quyền. Quyền thật phải do OS, sandbox và policy thực thi.

**Áp dụng khi nào?** Mọi MCP server truy cập repository riêng, cloud, issue nội bộ, CI/CD hoặc production.

**Bài toán giải quyết:** giảm data exfiltration, confused deputy, command injection, credential leakage và thay đổi trái phép.

Nguồn: [MCP — Security Best Practices](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices) · [MCP — Client concepts](https://modelcontextprotocol.io/docs/2026-07-28/learn/client-concepts).

---

### 11. Chọn STDIO hay Streamable HTTP cho MCP server DEV?

**Trả lời:**

| Tiêu chí | STDIO | Streamable HTTP |
| --- | --- | --- |
| Vị trí | Local process. | Remote hoặc shared service. |
| Latency | Thấp, không có network hop. | Phụ thuộc mạng. |
| Auth | Thường dựa vào user/process/OS. | OAuth, bearer token và policy mạng. |
| Scale | Một developer/client. | Nhiều client, quản lý tập trung. |
| Dữ liệu phù hợp | Local filesystem, local Git, compiler. | GitHub, CI, observability, enterprise services. |
| Vận hành | Cài binary/package trên máy. | Deploy, monitor, patch server tập trung. |
| Rủi ro | Binary local có quyền máy người dùng. | Session, auth, proxy và network attack. |

**Khuyến nghị hybrid:**

- Local code/file/compiler tool dùng STDIO và sandbox.
- Hệ thống shared như issue, CI, APM dùng Streamable HTTP.
- Không đưa production credential vào local server nếu không cần.
- Dùng server-side filtering để tránh tải dữ liệu lớn qua mạng rồi mới lọc.

**Áp dụng khi nào?** Khi thiết kế topology và trust boundary.

**Bài toán giải quyết:** cân bằng latency, deployability, credential, scale và quyền truy cập.

Nguồn: [MCP — Architecture overview](https://modelcontextprotocol.io/docs/2026-07-28/learn/architecture).

---

### 12. Kiểm thử và debug MCP server trong DEV như thế nào?

**Trả lời:** Kiểm thử theo ba tầng.

#### Contract test

- Capability negotiation.
- `tools/list`, `resources/list`, pagination và `listChanged`.
- Input/output schema.
- Error contract và timeout.
- Tool annotation, resource MIME type và URI.
- Authorization và denied scope.

#### Behavioral test

- Happy path và invalid input.
- Payload lớn, truncation, cursor.
- Concurrent call, retry và idempotency.
- Cancellation/progress.
- Partial side effect.
- Prompt injection trong issue, log hoặc file.
- Không rò secret trong output/log.

#### Agent evaluation

- Model chọn đúng tool không?
- Có đọc quá nhiều resource không?
- Tool call có thừa không?
- Bản sửa có pass test và bám acceptance criteria không?
- Khi thiếu evidence, model có dừng hoặc hỏi thay vì đoán không?

MCP Inspector cho phép xem resources, prompts, tools, schema, result, logs và notifications; phù hợp cho vòng lặp phát triển và edge-case testing.

**Áp dụng khi nào?** Trước khi publish server, khi nâng SDK/spec hoặc khi host/model thay đổi.

**Bài toán giải quyết:** tách lỗi protocol/server khỏi lỗi lựa chọn tool của model và lỗi workflow nghiệp vụ.

Nguồn: [MCP — Inspector](https://modelcontextprotocol.io/docs/2026-07-28/tools/inspector) · [MCP — Debugging](https://modelcontextprotocol.io/docs/tools/debugging).

---

### 13. Những anti-pattern nào làm MCP tốn token hơn và khiến DEV kém an toàn?

**Trả lời:**

| Anti-pattern | Hậu quả | Cách sửa |
| --- | --- | --- |
| Nạp mọi tool schema upfront | Context đầy trước khi đọc yêu cầu. | Progressive discovery và threshold. |
| Tool description dài như tài liệu | Token lặp ở mọi turn. | Mô tả ngắn; tài liệu chi tiết thành resource. |
| Tool trả toàn bộ log/repo/table | Context overflow, tăng latency. | Filter, aggregation, pagination, resource URI. |
| Một tool làm mọi thứ | Model khó chọn tham số, quyền quá rộng. | Chia theo intent và side effect. |
| Quá nhiều tool siêu nhỏ | Nhiều round trip, planning phức tạp. | Gộp thành primitive nghiệp vụ vừa đủ. |
| Model tự lọc hàng nghìn record | Token và chất lượng kém. | Lọc trong server hoặc sandbox code mode. |
| Credential trong env của code sinh bởi model | Rò secret và exfiltration. | Host broker giữ token. |
| Auto-approve write tool | Commit/deploy/delete ngoài ý muốn. | Policy và approval theo risk. |
| Tin tool annotation/output tuyệt đối | Server độc hại có thể đánh lừa host/model. | Trust registry, validation, output policy. |
| MCP thay cho deterministic automation | Tăng độ bất định không cần thiết. | Giữ script/CI cho bước deterministic. |

**Áp dụng khi nào?** Khi token tăng sau mỗi server mới, agent gọi sai tool hoặc đội bắt đầu bật auto-approval để “đỡ phiền”.

**Bài toán giải quyết:** ngăn MCP trở thành “context bloat protocol” hoặc một lớp remote execution thiếu kiểm soát.

---

## Kiến trúc khuyến nghị tối thiểu

```text
1. Catalog nhỏ:
   search_capabilities(query, readOnly?, system?, environment?)

2. Inspect theo nhu cầu:
   get_tool_schema(name)
   get_resource_metadata(uri)

3. Read có giới hạn:
   search_code(query, paths, limit)
   read_file(path, startLine, endLine)
   get_log_excerpt(traceId/jobId, around, maxLines)

4. Xử lý ngoài context:
   filter / join / deduplicate / aggregate trong server hoặc sandbox

5. Write có kiểm soát:
   apply_patch(diff)
   run_targeted_tests(testIds)
   create_draft_pr(...)
   deploy(...) → luôn approval

6. Evidence cuối:
   identifiers + diff summary + test results + links
```

## Checklist triển khai

- [ ] Có baseline token và task-success trước khi dùng MCP.
- [ ] Host có progressive discovery hoặc tool search.
- [ ] Chỉ kết nối server cần cho workspace/tác vụ.
- [ ] Tool schema ngắn, typed và có output schema.
- [ ] Query hỗ trợ filter, pagination và field selection.
- [ ] Payload lớn trả URI/handle, không đẩy thẳng vào context.
- [ ] Có code mode/sandbox cho chuỗi xử lý dữ liệu lớn khi host hỗ trợ.
- [ ] Read/write tool tách rời và least privilege.
- [ ] Credential ở host/broker; không vào model context.
- [ ] Có approval, audit, timeout, cancellation và rate limit.
- [ ] Contract test bằng Inspector và agent eval theo task.
- [ ] Theo dõi token, latency, retry, task success và DEV cycle time.

## Tài liệu tham khảo

1. [Model Context Protocol — What is MCP?](https://modelcontextprotocol.io/docs/getting-started/intro)
2. [Model Context Protocol — Architecture overview, phiên bản 2026-07-28](https://modelcontextprotocol.io/docs/2026-07-28/learn/architecture)
3. [Model Context Protocol — Client Best Practices](https://modelcontextprotocol.io/docs/2026-07-28/develop/clients/client-best-practices)
4. [Model Context Protocol — Tools](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)
5. [Model Context Protocol — Resources](https://modelcontextprotocol.io/specification/2025-11-25/server/resources)
6. [Model Context Protocol — Understanding MCP servers](https://modelcontextprotocol.io/docs/learn/server-concepts)
7. [Model Context Protocol — MCP Inspector](https://modelcontextprotocol.io/docs/2026-07-28/tools/inspector)
8. [Model Context Protocol — Debugging](https://modelcontextprotocol.io/docs/tools/debugging)
9. [Model Context Protocol — Security Best Practices](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices)
10. [Model Context Protocol — Client concepts](https://modelcontextprotocol.io/docs/2026-07-28/learn/client-concepts)
11. [Model Context Protocol — Cancellation](https://modelcontextprotocol.io/specification/2025-11-25/basic/utilities/cancellation)
