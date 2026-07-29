# Ứng dụng MCP với Codex cho dự án RAG tài liệu kỹ thuật

> [Knowledge Tree](../../../../README.md) / [AI](../../../README.md) / [LLM](../../README.md) / [RAG](../README.md) / **MCP–Codex Workflow**

**Model Context Protocol (MCP)** là giao thức chuẩn để một ứng dụng AI như **Codex** khám phá và sử dụng dữ liệu, công cụ và quy trình từ các hệ thống bên ngoài. Trong dự án chatbot **Retrieval-Augmented Generation (RAG)** cho tài liệu kỹ thuật thiết bị, MCP phù hợp nhất khi đóng vai trò **lớp tích hợp cho quy trình phát triển, kiểm thử và vận hành kỹ thuật**: giúp Codex đọc ticket, mã nguồn, kiến trúc, trace, phiên bản index, bộ eval và gọi các hành động được kiểm soát.

MCP không tự thay thế parser/OCR, vector database, search engine, API nghiệp vụ, CI/CD hay lớp phân quyền của chatbot. Nó chuẩn hóa cách Codex tiếp cận những hệ thống đó.

> **Issue → Gather context through MCP → Reproduce → Plan → Change code/config → Test → Run RAG eval → Review evidence → Pull request**

## Điều hướng

- **Node cha:** [RAG](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** [ETL tài liệu PDF kỹ thuật cho RAG](../etl/README.md)
- **Khái niệm sau trong cùng nhánh:** Chưa có.

## Kiến trúc khuyến nghị

```text
                          ┌────────────────────────────┐
                          │ Developer / Reviewer       │
                          └──────────────┬─────────────┘
                                         │ task + approval
                              ┌──────────▼──────────┐
                              │ Codex CLI / IDE     │
                              │ Codex host          │
                              └───────┬─────────────┘
                                      │ MCP clients
          ┌───────────────────────────┼────────────────────────────┐
          │                           │                            │
┌─────────▼──────────┐     ┌──────────▼──────────┐      ┌─────────▼──────────┐
│ Code/Work MCP      │     │ RAG Observability   │      │ Eval/Release MCP   │
│ issue, PR, docs    │     │ trace, chunk, index │      │ dataset, run, gate │
└─────────┬──────────┘     └──────────┬──────────┘      └─────────┬──────────┘
          │                           │                            │
          ▼                           ▼                            ▼
 Git + issue tracker       logs/traces/vector registry       CI/eval/deploy

Runtime chatbot data plane — tách biệt khỏi Codex development control plane:

User → Auth/ACL → Query routing → Retrieval/Rerank → LLM → Citation/Abstain
                         │
          PDF/Object Storage + Document Registry + Search/Vector Index
```

**Ranh giới quan trọng:** Codex là coding agent phục vụ đội phát triển. Chatbot runtime là sản phẩm phục vụ người vận hành/bảo trì. Có thể dùng MCP ở cả hai phía, nhưng phải dùng **server, credential, tool allowlist, audit log và approval policy khác nhau**.

---

## A. 5 câu hỏi cốt lõi

### 1. MCP và Codex đóng vai trò gì trong dự án RAG tài liệu kỹ thuật?

**Trả lời:** Codex là coding agent đọc mã nguồn, chỉnh sửa workspace, chạy lệnh và phối hợp công cụ. MCP là lớp kết nối giúp Codex truy cập các hệ thống không nằm trực tiếp trong repository theo giao diện thống nhất.

Trong dự án tài liệu hướng dẫn vận hành, bảo dưỡng, sửa chữa và biên bản kiểm tra thiết bị, Codex có thể dùng MCP để:

1. Đọc issue, yêu cầu nghiệp vụ và tiêu chí nghiệm thu.
2. Tra cứu tài liệu kiến trúc, schema canonical document và quy tắc metadata.
3. Tìm trace của một câu hỏi trả lời sai.
4. Xem chunk, filter, candidate retrieval, rerank score và citation đã dùng.
5. Chạy một bộ eval/regression có phiên bản.
6. Tạo hoặc cập nhật pull request sau khi mã nguồn và bằng chứng kiểm thử đã sẵn sàng.

MCP nên được xem là **integration contract** cho development control plane, không phải “bộ não RAG”. Retrieval quality vẫn phụ thuộc vào ETL, chunking, embedding, hybrid search, reranking, prompt, ACL và evaluation.

**Áp dụng khi nào?**

- Dự án có nhiều hệ thống: GitHub/GitLab, issue tracker, object storage, vector database, tracing, eval service và CI/CD.
- Lỗi RAG cần đối chiếu nhiều nguồn thay vì chỉ sửa code trong một repository.
- Muốn chuẩn hóa workflow giữa Codex CLI, IDE và máy của nhiều thành viên.

**Bài toán giải quyết:**

- Giảm thao tác copy/paste log, trace, ticket và tài liệu giữa nhiều giao diện.
- Giúp Codex có context đúng thời điểm thay vì nhồi toàn bộ tài liệu vào prompt.
- Tạo luồng có thể audit: công cụ nào được gọi, input nào được dùng, kết quả nào chứng minh thay đổi là đúng.

**So sánh:**

| Cách tích hợp | Điểm mạnh | Giới hạn | Chọn khi |
| --- | --- | --- | --- |
| Copy/paste thủ công | Nhanh cho thử nghiệm nhỏ. | Dễ thiếu context, lỗi thời, khó audit. | POC hoặc một lỗi đơn giản. |
| Script/CLI riêng | Kiểm soát tốt, dễ dùng trong CI. | Mỗi agent phải học giao diện riêng; schema không thống nhất. | Tác vụ ổn định, tự động hóa không cần hội thoại. |
| MCP | Tool/resource có schema, có discovery, dùng lại giữa client hỗ trợ MCP. | Phải vận hành server, auth và policy. | Nhiều nguồn context/công cụ, workflow agent lặp lại. |
| Agent framework riêng | Orchestration linh hoạt, có thể xây multi-agent sâu. | Chi phí xây và vận hành lớn hơn. | Sản phẩm agent runtime cần logic điều phối riêng. |

---

### 2. Nên tách luồng MCP–Codex và luồng chatbot runtime như thế nào?

**Trả lời:** Nên tách thành hai mặt phẳng:

1. **Development control plane:** Codex, repository, issue, traces, eval, preview deployment và PR. Mục tiêu là thay đổi hệ thống an toàn.
2. **Runtime data plane:** chatbot nhận câu hỏi thật, xác thực người dùng, retrieve tài liệu, gọi API nghiệp vụ, sinh câu trả lời và citation. Mục tiêu là phục vụ người dùng với latency và SLA ổn định.

```text
Development:
Task → Codex → MCP context/tools → Code change → Tests/Evals → PR → Deploy

Runtime:
Question → Auth/ACL → Retrieve/Tool → Answer + Citation/Refusal → Trace
```

Không cho Codex development dùng credential runtime toàn quyền. MCP quan sát production nên mặc định chỉ đọc dữ liệu đã che thông tin nhạy cảm. Các tool như `publish_index`, `retry_ingestion`, `rollback_release` hoặc ghi vào CMMS/EAM phải yêu cầu approval rõ ràng.

**Source of truth nên được tuyên bố rõ:**

| Loại dữ liệu | Source of truth | Dữ liệu dẫn xuất |
| --- | --- | --- |
| PDF/manual gốc | Object storage bất biến + document registry | OCR text, canonical JSON, chunk, embedding. |
| Trạng thái thiết bị/lệnh bảo trì | CMMS/EAM/ERP/API nghiệp vụ | Cache hoặc bản tóm tắt. |
| Mã nguồn/cấu hình | Git repository | Build artifact, container image. |
| Kỳ vọng chất lượng | Eval dataset có version | Report, metric, release gate. |
| Trạng thái index | Index registry/manifest | Alias đang phục vụ, dashboard. |

**Áp dụng khi nào?** Luôn áp dụng cho production, đặc biệt khi tài liệu có ACL, dữ liệu nhà máy, quy trình an toàn hoặc thao tác có thể ảnh hưởng vận hành thiết bị.

**Bài toán giải quyết:** ngăn việc coding agent vô tình trở thành đường vòng truy cập production; tránh lẫn dữ liệu dẫn xuất với dữ liệu nghiệp vụ chính thức; dễ rollback và điều tra sự cố.

**So sánh:**

- **Một agent/toàn quyền:** nhanh lúc đầu nhưng blast radius lớn và audit kém.
- **Hai mặt phẳng tách biệt:** thêm cấu hình và service account, nhưng rõ ranh giới trách nhiệm, dễ kiểm soát và phù hợp production.

---

### 3. Luồng làm việc end-to-end của Codex cho một task RAG nên diễn ra thế nào?

**Trả lời:** Dùng workflow theo bằng chứng, không bắt đầu bằng sửa prompt ngay lập tức.

#### Bước 1 — Nhận task và tiêu chí nghiệm thu

Codex đọc issue qua MCP, trích ra:

- hiện tượng và phạm vi thiết bị/model;
- câu hỏi mẫu, expected answer và tài liệu nguồn;
- ràng buộc ACL, ngôn ngữ, latency, chi phí;
- metric hoặc release gate cần đạt.

#### Bước 2 — Thu thập context tối thiểu

Codex đọc `AGENTS.md`, README kiến trúc và các resource liên quan; sau đó gọi tool quan sát như:

- `search_traces(question, timeRange, equipmentModel)`;
- `get_trace(traceId)`;
- `inspect_chunk(chunkId)`;
- `retrieve_preview(query, filters, indexVersion)`;
- `get_document_version(documentId)`.

#### Bước 3 — Tạo MRE và phân lớp nguyên nhân

Ví dụ lỗi: “Hỏi quy trình thay dầu bơm X100 nhưng chatbot trích manual X200”. Codex tạo Minimal Reproducible Example gồm query, user scope, expected document/model, index version và actual candidates.

Phân loại lỗi theo pipeline:

```text
Source/version → Parse/OCR → Metadata → Chunk → Index → Filter → Retrieve
→ Rerank → Context assembly → Prompt/model → Citation/output validation
```

#### Bước 4 — Lập kế hoạch thay đổi nhỏ nhất

Ví dụ nguyên nhân là filter `equipmentModel` chỉ áp sau retrieval. Kế hoạch đúng là đưa filter bắt buộc vào truy vấn hybrid search, thêm test và regression case; không “sửa prompt để model chú ý model thiết bị”.

#### Bước 5 — Thực hiện trong workspace cô lập

Codex chỉnh sửa code/config trong branch hoặc worktree; chạy formatter, type check, unit test, integration test và static checks theo `AGENTS.md`.

#### Bước 6 — Chạy eval RAG

Gọi eval MCP hoặc CLI để chạy các slice:

- đúng model/serial family;
- manual nhiều phiên bản;
- câu hỏi an toàn;
- OCR scan;
- biên bản kiểm tra;
- câu hỏi thiếu bằng chứng phải từ chối.

#### Bước 7 — So sánh và review evidence

So sánh baseline với candidate về retrieval recall/precision, groundedness, citation accuracy, latency, token và cost. Kiểm tra không làm giảm slice khác.

#### Bước 8 — Tạo PR có thể review

PR cần nêu root cause, thay đổi, test, eval report, trace trước/sau, rủi ro và rollback.

**Áp dụng khi nào?** Mọi feature, bug hoặc tối ưu có thể ảnh hưởng retrieval, nguồn citation, quyền truy cập hoặc nội dung hướng dẫn kỹ thuật.

**Bài toán giải quyết:** tránh sửa mò; giảm lỗi “pass một câu hỏi nhưng hỏng toàn bộ corpus”; biến tri thức điều tra thành regression suite.

**So sánh:**

| Workflow | Đặc điểm | Rủi ro |
| --- | --- | --- |
| Prompt-first | Sửa prompt ngay khi thấy answer sai. | Che nguyên nhân ở metadata/retrieval; khó ổn định. |
| Code-first | Đọc code rồi đoán lỗi. | Thiếu evidence production và data lineage. |
| Evidence-first MCP–Codex | Trace → MRE → layer → patch → eval. | Cần observability và tool contract tốt. |

---

### 4. Nên thiết kế Resources, Tools và Prompts MCP nào cho dự án này?

**Trả lời:** MCP có ba primitive chính: **Resources** cung cấp context, **Tools** thực thi hành động và **Prompts** đóng gói workflow do người dùng chủ động gọi. Thiết kế tốt cần tool nhỏ, typed, ít quyền và trả output có định danh.

#### Resources nên có

| Resource URI ví dụ | Nội dung | Ghi chú |
| --- | --- | --- |
| `rag-arch://service/retrieval` | Kiến trúc, owner, dependency, SLO. | Read-only, versioned. |
| `rag-doc://schema/canonical-document/v3` | Schema page/block/table/figure/ACL. | Source cho ETL và index. |
| `rag-eval://suite/equipment-safety/v12` | Dataset manifest và rubric. | Không trả dữ liệu nhạy cảm ngoài scope. |
| `rag-index://version/prod-2026-07-29` | Manifest model/chunk/index/config. | Không coi index là source of truth của PDF. |
| `rag-runbook://incident/wrong-citation` | Runbook điều tra citation sai. | Có owner và updatedAt. |

#### Tools quan sát mặc định read-only

```text
search_traces
get_trace
get_retrieval_candidates
inspect_chunk
get_document_version
compare_index_versions
get_ingestion_job
get_eval_run
compare_eval_runs
```

#### Tools ghi cần approval

```text
create_regression_case
retry_ingestion_job
create_preview_index
publish_index_alias
rollback_index_alias
create_pull_request
trigger_deployment
```

Mỗi tool nên trả structured output như `traceId`, `documentId`, `documentVersion`, `chunkId`, `indexVersion`, `requestId`, `status`, `warnings` và URL nội bộ để người review mở bằng chứng.

#### Prompts/workflow nên có

- `/diagnose-rag-answer` — điều tra một answer sai theo pipeline.
- `/add-rag-feature` — triển khai feature kèm test/eval/PR checklist.
- `/review-index-release` — so sánh index candidate với production.
- `/triage-ingestion-failure` — khoanh vùng parser/OCR/layout/schema.

**Áp dụng khi nào?** Khi cùng một thao tác điều tra hoặc release lặp lại qua nhiều thành viên và nhiều repository.

**Bài toán giải quyết:** giảm tool “đa năng” khó kiểm soát; giúp Codex chọn đúng capability; chuẩn hóa output để test và audit.

**So sánh:**

- **Một tool `run_sql` hoặc `call_api`:** linh hoạt nhưng quá rộng, khó chặn exfiltration và destructive action.
- **Tool theo use case:** nhiều schema hơn nhưng quyền hẹp, dễ allowlist, test và ghi audit.
- **Resource thay cho tool read:** tốt với context ổn định hoặc định danh được; tool phù hợp truy vấn động và tính toán.

---

### 5. Bảo mật, quyền và approval cho MCP–Codex nên thiết kế ra sao?

**Trả lời:** Dùng nguyên tắc **least privilege**, **read-only by default**, **explicit approval for writes** và **end-to-end identity propagation**.

#### Lớp 1 — Codex sandbox

Thiết lập mặc định phù hợp cho development:

```toml
sandbox_mode = "workspace-write"
approval_policy = "on-request"
approvals_reviewer = "user"
```

Codex được đọc/chỉnh trong workspace và chạy lệnh cục bộ thông thường; khi cần network, ra ngoài workspace hoặc hành động vượt policy thì phải xin approval.

#### Lớp 2 — MCP server allowlist

- Chỉ bật server cần cho task.
- Chỉ expose `enabled_tools` cần thiết.
- Tool read có thể tự động; tool write yêu cầu prompt/approval.
- Tách token quan sát production khỏi token release.
- Không để secret trong repository hay `AGENTS.md`.

#### Lớp 3 — Authorization nghiệp vụ

- MCP server phải kiểm tra user/service identity, tenant, site, role và purpose.
- Filter ACL phải áp trước retrieval, không chỉ lọc sau khi đã lấy chunk.
- Tool lấy tài liệu phải trả đúng phiên bản và trạng thái hiệu lực.
- Dữ liệu từ manual/PDF phải được coi là **untrusted content**; không thực thi câu lệnh nhúng trong tài liệu.

#### Lớp 4 — Safety cho thiết bị

- Với hướng dẫn có nguy cơ điện, áp suất, nhiệt, hóa chất hoặc lockout/tagout, câu trả lời phải bám tài liệu được phê duyệt, hiện hành và đúng model.
- Khi thiếu bằng chứng hoặc xung đột phiên bản, hệ thống phải từ chối/đề nghị kiểm tra manual chính thức.
- MCP tool không được tự động phát lệnh điều khiển thiết bị chỉ từ nội dung LLM tạo ra.

#### Lớp 5 — Audit và quan sát

Ghi lại ai gọi tool nào, thời gian, scope, input đã redaction, result ID, approval, latency và lỗi. Không ghi token, Authorization header hoặc toàn bộ tài liệu nhạy cảm vào log.

**Áp dụng khi nào?** Bắt buộc khi có dữ liệu nội bộ, nhiều nhà máy/tenant, quy trình an toàn, production access hoặc tool có khả năng ghi.

**Bài toán giải quyết:** prompt injection, tool misuse, data exfiltration, thao tác nhầm môi trường, dùng sai phiên bản manual và không truy được trách nhiệm.

**So sánh:**

| Mô hình quyền | Ưu điểm | Rủi ro |
| --- | --- | --- |
| Full access + no approval | Tốc độ cao. | Blast radius rất lớn; không phù hợp mặc định. |
| Read-only | An toàn cho khám phá/điều tra. | Không tự hoàn tất release hoặc cập nhật. |
| Workspace write + on-request | Cân bằng cho coding hằng ngày. | Cần thiết kế approval rõ để không gây mệt mỏi. |
| Tool-level approval | Chính xác theo hành động. | Phải phân loại tool và schema tốt. |

---

## B. 8 câu hỏi phổ biến

### 6. Nên viết `AGENTS.md` và `.codex/config.toml` như thế nào cho repository RAG?

**Trả lời:** `AGENTS.md` chứa quy tắc làm việc và kiến thức bền vững của dự án; `.codex/config.toml` chứa cấu hình Codex/MCP theo project. Không đưa secret vào hai file này.

Ví dụ `AGENTS.md` ở root:

```md
# Repository instructions

## Source of truth
- Original PDFs and document versions live in the document registry.
- Search/vector indexes are derived artifacts.
- Equipment state comes from CMMS/EAM APIs, never from RAG chunks.

## Architecture boundaries
- `services/ingestion`: parse/OCR/layout/canonical document.
- `services/retrieval`: ACL filter, hybrid retrieval, rerank.
- `services/chat`: context assembly, answer, citation, abstention.
- Do not bypass ACL or document-status filters.

## Validation
- Run unit and integration tests for the changed service.
- Run the affected RAG eval slices before opening a PR.
- Every fixed production defect must add an MRE/regression case.

## Safety
- Never publish an index, deploy production, or retry bulk ingestion without approval.
- Never place secrets or raw sensitive document content in logs.
```

Ví dụ `.codex/config.toml`:

```toml
sandbox_mode = "workspace-write"
approval_policy = "on-request"
approvals_reviewer = "user"

[mcp_servers.rag_observability]
url = "https://rag-observability.example.internal/mcp"
bearer_token_env_var = "RAG_OBS_MCP_TOKEN"
enabled_tools = [
  "search_traces",
  "get_trace",
  "inspect_chunk",
  "compare_eval_runs"
]
required = true
tool_timeout_sec = 60

[mcp_servers.rag_release]
url = "https://rag-release.example.internal/mcp"
bearer_token_env_var = "RAG_RELEASE_MCP_TOKEN"
enabled_tools = ["create_preview_index", "publish_index_alias", "rollback_index_alias"]
default_tools_approval_mode = "writes"
```

Codex đọc hướng dẫn theo chuỗi từ global tới project và thư mục hiện tại; file gần thư mục làm việc hơn có thể ghi đè hướng dẫn phía trên. Vì vậy có thể đặt `AGENTS.override.md` trong `services/ingestion/` hoặc `services/retrieval/` cho lệnh test và quy tắc riêng.

**Áp dụng khi nào?** Ngay từ đầu dự án hoặc khi Codex thường xuyên hỏi lại lệnh setup, ranh giới service và definition of done.

**Bài toán giải quyết:** giảm prompt lặp; tránh agent chạy sai test, sửa nhầm layer hoặc bỏ qua eval/ACL.

**So sánh:** `README.md` phục vụ người đọc và kiến trúc tổng quát; `AGENTS.md` tối ưu chỉ dẫn hành động cho coding agent; config định nghĩa capability và quyền chứ không thay thế tài liệu dự án.

---

### 7. Khi nào dùng MCP local STDIO và khi nào dùng remote Streamable HTTP?

**Trả lời:** Dùng **STDIO** cho tool chạy cùng máy và gắn với workspace; dùng **Streamable HTTP** cho dịch vụ dùng chung, có auth, audit và dữ liệu tập trung.

| Tiêu chí | STDIO local | Streamable HTTP remote |
| --- | --- | --- |
| Vị trí | Process do Codex khởi chạy. | Service qua URL. |
| Phù hợp | Filesystem, local parser, test helper, dev database cô lập. | Tracing, eval service, document registry, CI/CD, shared index registry. |
| Auth | Environment/process boundary. | Bearer token hoặc OAuth; cần HTTPS. |
| Latency | Thấp, không có network hop. | Phụ thuộc mạng và service. |
| Quản trị | Mỗi máy phải cài dependency. | Quản lý tập trung, dễ audit/version. |
| Rủi ro | Local package/process có thể đọc máy nếu cấp quá rộng. | Data egress, auth, availability và server trust. |

**Khuyến nghị:**

- Local STDIO: `mre-builder`, parser fixture inspector, test report reader.
- Remote HTTP: Git/issue, observability, eval, document registry, index release.
- Không expose trực tiếp production database bằng một MCP SQL server toàn quyền; đặt domain API/tool hẹp ở trước.

**Áp dụng khi nào?** Chọn theo vị trí source of truth và trust boundary, không chọn chỉ vì cài đặt tiện.

**Bài toán giải quyết:** cân bằng developer experience với quản trị tập trung và bảo mật.

**So sánh:** REST API là giao diện service; MCP bọc capability để AI client discover/call theo schema. Có thể xây MCP server phía trên REST API hiện có thay vì thay API.

---

### 8. Codex dùng MCP để phát triển pipeline ETL/OCR/index như thế nào?

**Trả lời:** MCP giúp Codex quan sát và điều khiển pipeline theo work-item có định danh, nhưng ETL vẫn chạy bằng queue/workers và workflow engine chuyên dụng.

Luồng xử lý một lỗi ingestion:

```text
Issue → get_ingestion_job → get_page_artifacts → inspect_parser_output
→ compare_engine_outputs → reproduce fixture → patch router/schema
→ run ingestion tests → process preview pages → quality gate → PR
```

Tool đề xuất:

- `get_ingestion_job(jobId)` — trạng thái, retry count, engine, error code.
- `get_page_artifacts(documentVersion, page)` — text, layout blocks, figure metadata đã redaction.
- `compare_extraction_engines(fixtureId, engines)` — benchmark trên fixture.
- `run_ingestion_fixture(fixtureId, pipelineVersion)` — chạy test cô lập.
- `create_preview_document_version(...)` — ghi candidate, không publish.
- `publish_document_version(...)` — tool write yêu cầu approval.

Codex nên tạo fixture nhỏ từ trang lỗi thay vì chạy lại hàng triệu PDF. Sau khi sửa, chạy regression theo loại trang: digital text, scan, bảng, form, sơ đồ, chữ xoay, multi-column và PDF lai.

**Áp dụng khi nào?** OCR sai, reading order sai, mất bảng/hình, metadata thiết bị sai, job treo hoặc index thiếu tài liệu.

**Bài toán giải quyết:** truy ngược lỗi theo page/engine/version; giảm chi phí tái xử lý; biến case production thành fixture.

**So sánh:** MCP không thay Airflow/Temporal/Kafka/Celery. Workflow engine chịu trách nhiệm scheduling/retry/state; MCP cung cấp giao diện quan sát và hành động có kiểm soát cho Codex.

---

### 9. Codex dùng MCP để debug một câu trả lời RAG sai như thế nào?

**Trả lời:** Điều tra từ output ngược về source, giữ nguyên `traceId`, `indexVersion`, `documentVersion` và user scope.

Checklist:

1. **Output:** câu nào sai, citation nào không hỗ trợ?
2. **Context:** model nhận những chunk nào, thứ tự và token budget?
3. **Rerank:** candidate đúng có bị hạ điểm hay loại bỏ?
4. **Retrieval:** candidate đúng có xuất hiện trong top-k?
5. **Filter/ACL:** model thiết bị, site, language, status, effective date có đúng?
6. **Index:** chunk đúng có tồn tại trong version đang phục vụ?
7. **ETL:** text, heading, bảng, page và metadata gốc có đúng?
8. **Source:** manual có phải bản hiện hành và được phê duyệt?

Ví dụ tool chain:

```text
get_trace(traceId)
→ get_retrieval_candidates(traceId)
→ inspect_chunk(candidateChunkId)
→ get_document_version(documentId, version)
→ retrieve_preview(query, correctedFilters, sameIndexVersion)
→ create_regression_case(traceId)
```

**Áp dụng khi nào?** Answer hallucination, sai model, citation sai trang, bỏ sót bước an toàn, dùng manual cũ hoặc trả lời từ biên bản không đúng thiết bị.

**Bài toán giải quyết:** xác định lỗi nằm ở data, search hay generation; tránh đổi LLM khi nguyên nhân là metadata/filter.

**So sánh:** log text đơn lẻ chỉ cho biết lỗi cuối; distributed trace có span cho auth, rewrite, retrieval, rerank, LLM và citation giúp tái hiện đầy đủ hơn.

---

### 10. Nên tích hợp eval và regression gate với MCP–Codex ra sao?

**Trả lời:** Eval dataset phải nằm ngoài prompt tạm thời, có version và chia slice theo rủi ro. Codex dùng MCP để tìm suite liên quan, chạy candidate, so sánh baseline và đính report vào PR.

Một eval record nên có:

```json
{
  "caseId": "pump-x100-oil-change-001",
  "question": "Quy trình thay dầu bơm X100 là gì?",
  "userScope": {"site": "A", "role": "maintenance"},
  "expectedDocuments": ["manual-x100-v3"],
  "requiredFacts": ["isolate power", "release pressure"],
  "forbiddenFacts": ["instructions for X200"],
  "expectedBehavior": "answer_with_citation",
  "risk": "safety_high"
}
```

Metric/gate nên gồm nhiều tầng:

- **Ingestion:** parse success, structure/table/figure accuracy, metadata validity.
- **Retrieval:** Recall@k, MRR/nDCG, filter correctness, correct-version rate.
- **Answer:** groundedness/faithfulness, required-fact coverage, citation accuracy, abstention.
- **System:** p95 latency, token, cost, error rate.
- **Safety:** critical instruction omissions, wrong-equipment contamination, ACL leakage.

Không dùng một điểm trung bình duy nhất để release. Slice `safety_high` hoặc `ACL` phải có hard gate riêng.

**Áp dụng khi nào?** Trước merge/release các thay đổi parser, chunking, embedding, index, filter, reranker, prompt hoặc model.

**Bài toán giải quyết:** phát hiện regression ngoài case đang sửa; làm thay đổi model/index có thể so sánh và rollback.

**So sánh:** unit test kiểm tra logic xác định; RAG eval kiểm tra hành vi xác suất và chất lượng end-to-end. Cần cả hai.

---

### 11. Có nên cho Codex tự động publish index, deploy hoặc sửa dữ liệu production không?

**Trả lời:** Mặc định không. Hãy chia autonomy thành ba mức:

| Mức | Hành động | Approval |
| --- | --- | --- |
| A — Observe | Đọc code, issue, trace, eval, manifest. | Có thể pre-approve nếu dữ liệu đúng scope. |
| B — Prepare | Sửa workspace, tạo fixture, chạy test, tạo preview index/PR. | Workspace policy; preview write có thể yêu cầu approval. |
| C — Apply | Publish alias, deploy production, retry bulk, rollback, cập nhật CMMS. | Approval rõ ràng, policy và audit bắt buộc. |

Để tự động hóa production an toàn hơn, tool phải hỗ trợ:

- dry-run/preview;
- idempotency key;
- expected current version để tránh race;
- bounded scope như một index alias hoặc một batch;
- rollback target;
- post-condition verification;
- audit event và link report.

**Áp dụng khi nào?** Mức C chỉ dùng khi workflow đã ổn định, tool domain-specific, có release gate và owner trực.

**Bài toán giải quyết:** tránh publish nhầm index, chạy lại toàn corpus, deploy khi eval chưa đạt hoặc ghi dữ liệu thiết bị ngoài phạm vi.

**So sánh:** “human in every read” gây chậm; “no approval anywhere” nguy hiểm. Approval nên tập trung ở hành động có side effect hoặc mở rộng trust boundary.

---

### 12. MCP khác gì direct API/function calling, CLI, CI/CD và agent framework?

**Trả lời:** Các công nghệ bổ sung nhau, không loại trừ nhau.

| Thành phần | Trách nhiệm chính | MCP liên hệ thế nào? |
| --- | --- | --- |
| REST/gRPC API | Contract của service nghiệp vụ. | MCP server có thể gọi API và expose tool phù hợp agent. |
| Function/tool calling | Model gọi function trong một ứng dụng. | MCP chuẩn hóa discovery, transport và server capability giữa nhiều client. |
| CLI/script | Tác vụ xác định, dễ chạy local/CI. | MCP tool có thể bọc CLI; Codex vẫn trực tiếp dùng CLI trong workspace khi phù hợp. |
| CI/CD | Build, test, artifact, deploy theo pipeline. | Codex/MCP trigger hoặc đọc trạng thái; không thay runner. |
| Workflow engine | Scheduling, retry, state dài hạn. | MCP cung cấp control/inspection surface. |
| Agent framework | Orchestration agent runtime, memory, guardrail. | Có thể dùng MCP làm tool layer. |
| RAG pipeline | Retrieve evidence và sinh answer. | MCP không thay retrieval/index; chỉ kết nối agent với chúng. |

**Áp dụng khi nào?** Dùng MCP khi muốn cùng một capability có thể được Codex hoặc AI client khác khám phá và gọi; giữ API/CLI/CI hiện có làm implementation phía dưới.

**Bài toán giải quyết:** tránh viết tích hợp bespoke cho từng coding agent; vẫn bảo toàn hệ thống vận hành đã ổn định.

**So sánh:** nếu chỉ có một lệnh ổn định chạy trong CI, CLI đủ tốt. Nếu cần reasoning tương tác qua nhiều nguồn và tool, MCP mang lại giá trị lớn hơn.

---

### 13. Lộ trình triển khai MCP–Codex thực tế và các anti-pattern phổ biến là gì?

**Trả lời:** Triển khai theo mức rủi ro, bắt đầu read-only và một workflow có ROI cao.

#### Giai đoạn 1 — Repository discipline

- Viết `AGENTS.md`.
- Chuẩn hóa lệnh setup/test/eval.
- Version hóa MRE và eval dataset.
- Bảo đảm trace chứa document/chunk/index version.

#### Giai đoạn 2 — Read-only MCP

- Issue/PR context.
- Architecture/runbook resources.
- Trace, chunk, document version và eval report.
- Test server bằng MCP Inspector.

#### Giai đoạn 3 — Safe preparation tools

- Tạo regression case.
- Chạy eval suite.
- Tạo preview index.
- Tạo PR/report.

#### Giai đoạn 4 — Controlled writes

- Retry một job/batch giới hạn.
- Publish alias có expected version và rollback.
- Trigger preview/staging deployment.
- Production deploy vẫn qua release policy.

#### Giai đoạn 5 — Đo hiệu quả

Theo dõi thời gian từ issue tới MRE, thời gian root-cause, tỷ lệ task có regression case, số approval, tool failure, eval regression lọt production và lead time PR.

#### Anti-pattern cần tránh

1. Expose tool `run_sql`/`run_shell` production toàn quyền.
2. Một MCP server chứa mọi hệ thống và credential.
3. Dùng vector index làm source of truth.
4. Cho Codex đọc toàn bộ corpus khi chỉ cần trace/chunk nhỏ.
5. Không version hóa prompt, model, index và eval dataset.
6. Tool description mơ hồ, output chỉ là text không có ID.
7. Tự động publish dựa trên một metric trung bình.
8. Tin nội dung tài liệu/MCP output như instruction thay vì dữ liệu không tin cậy.
9. Sửa answer bằng prompt mà không kiểm tra ETL/retrieval.
10. Không có timeout, rate limit, redaction và audit.

**Áp dụng khi nào?** Khi đưa MCP từ thử nghiệm cá nhân sang workflow của nhóm hoặc production-adjacent environment.

**Bài toán giải quyết:** đạt giá trị sớm mà không mở quyền quá rộng; tạo nền tảng đo được trước khi tăng autonomy.

**So sánh:** big-bang integration tạo nhiều tool nhưng ít tin cậy. Incremental rollout tạo một “golden workflow” trước, sau đó nhân rộng theo evidence.

---

## Mẫu workflow cụ thể: sửa lỗi trả lời sai model thiết bị

```text
1. Developer giao task:
   "Trace T-9382 trả quy trình X200 cho câu hỏi X100. Điều tra và sửa."

2. Codex đọc:
   - AGENTS.md
   - issue và acceptance criteria qua Work MCP
   - trace T-9382 qua Observability MCP

3. Codex phát hiện:
   - query filter có equipmentFamily=PUMP
   - thiếu equipmentModel=X100 ở retrieval
   - chunk X100 đứng hạng 14, X200 đứng hạng 2

4. Codex tạo MRE:
   - query, user scope, index version
   - expected doc manual-x100-v3
   - actual candidates và scores

5. Codex sửa:
   - đưa equipmentModel vào mandatory pre-filter
   - thêm fallback khi model metadata thiếu
   - thêm unit/integration test

6. Codex gọi Eval MCP:
   - run slice wrong-equipment-contamination
   - run safety_high và ACL smoke tests
   - compare baseline/candidate

7. Codex tạo PR:
   - root cause
   - diff
   - trace trước/sau
   - metric và case regression
   - rollback: revert + restore previous service version
```

## Checklist triển khai

- [ ] Mỗi source of truth có owner, ID và version.
- [ ] `AGENTS.md` nêu architecture boundary, test/eval và safety rules.
- [ ] MCP server tách read và write hoặc ít nhất tách credential/scope.
- [ ] Tool có input/output schema, timeout, rate limit và audit.
- [ ] Production data được redaction và kiểm tra ACL trước khi trả.
- [ ] Codex dùng `workspace-write` + `on-request` làm mặc định.
- [ ] Mọi bug production tạo MRE/regression case.
- [ ] Release gate theo slice rủi ro, không chỉ dùng điểm trung bình.
- [ ] Publish/deploy/rollback có preview, expected version và approval.
- [ ] MCP server được test bằng Inspector và integration test trong Codex client.

## Tài liệu tham khảo

- [OpenAI Codex — Model Context Protocol][openai-codex-mcp]
- [OpenAI Codex — Custom instructions with AGENTS.md][openai-agents-md]
- [OpenAI Codex — Sandbox][openai-sandbox]
- [OpenAI Codex — Agent approvals & security][openai-approvals]
- [OpenAI Codex — Configuration reference][openai-config]
- [MCP — Architecture overview][mcp-architecture]
- [MCP — Understanding MCP servers][mcp-server-concepts]
- [MCP — Specification][mcp-spec]
- [MCP — Tools][mcp-tools]
- [MCP — Inspector][mcp-inspector]
- [MCP — Security best practices][mcp-security]

[openai-codex-mcp]: https://developers.openai.com/codex/extend/mcp
[openai-agents-md]: https://developers.openai.com/codex/agent-configuration/agents-md
[openai-sandbox]: https://developers.openai.com/codex/sandboxing
[openai-approvals]: https://developers.openai.com/codex/agent-approvals-security
[openai-config]: https://developers.openai.com/codex/config-file/config-reference
[mcp-architecture]: https://modelcontextprotocol.io/docs/learn/architecture
[mcp-server-concepts]: https://modelcontextprotocol.io/docs/learn/server-concepts
[mcp-spec]: https://modelcontextprotocol.io/specification/2025-11-25
[mcp-tools]: https://modelcontextprotocol.io/specification/2025-11-25/server/tools
[mcp-inspector]: https://modelcontextprotocol.io/docs/tools/inspector
[mcp-security]: https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices
