# Model Context Protocol (MCP): giao thức, luồng giao tiếp và phân biệt đọc/ghi

> [Knowledge Tree](../../../../README.md) / [AI](../../../README.md) / [LLM](../../README.md) / [MCP](../README.md) / **Giao thức và đọc/ghi**

## Tóm tắt trọng yếu

- **MCP (Model Context Protocol)** là giao thức chuẩn hóa để ứng dụng AI kết nối với hệ thống bên ngoài thông qua server có capability được khám phá và gọi theo schema. MCP không phải mô hình AI, cơ sở dữ liệu hay hệ thống phân quyền thay thế.
- Luồng điển hình: **MCP host → MCP client → MCP server → adapter/API/DB của hệ thống nghiệp vụ**. Host thường là ứng dụng AI/IDE/agent; client quản lý phiên giao tiếp; server công bố tools/resources/prompts và thực thi phần tích hợp.
- MCP dùng **JSON-RPC 2.0** làm định dạng message. Trong khởi tạo, client/server thương lượng phiên bản và capability; sau đó discovery, đọc resource hoặc gọi tool diễn ra bằng request/response và có thể có notification.
- **MCP read** chủ yếu truy vấn/đọc dữ liệu; **MCP write/update** thực hiện hành động có side effect. Đây không phải hai giao thức MCP riêng mà là cách phân loại capability và quyền trong ứng dụng.
- Hệ thống truyền thống không nhất thiết phải “được huấn luyện hiểu MCP”. Thường cần một **MCP server/adapter** ánh xạ capability MCP sang API/service/DB hiện có, cộng với schema, auth, policy, validation, audit, timeout, rate limit và xử lý lỗi.
- MCP chuẩn hóa giao tiếp, không tự tạo quyền truy cập, không bảo đảm dữ liệu đúng, không thay thế authorization nghiệp vụ, và không bắt buộc host phải dùng AI cho mọi bước.

## Điều hướng

- **Node cha:** [MCP trong tối ưu context và luồng DEV](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** Chưa có.
- **Khái niệm sau trong cùng nhánh:** Chưa có.

## A. 5 câu hỏi cốt lõi

### 1. MCP là gì, và giải quyết vấn đề gì so với tích hợp truyền thống?

**Trả lời:** MCP là **Model Context Protocol**, một giao thức mở để ứng dụng AI khám phá và sử dụng context/capability từ hệ thống bên ngoài theo một hợp đồng thống nhất. Thay vì mỗi AI host tự viết tích hợp riêng cho từng dịch vụ, host có thể kết nối tới MCP server dùng chung.

Ví dụ, hệ thống quản lý đơn hàng đã có REST API. MCP server có thể công bố tool `get_order_status`; AI host khám phá tool, cung cấp tham số `orderId`, server gọi API hiện hữu rồi trả kết quả có cấu trúc. Backend không cần biến thành mô hình AI và thường không cần thay API lõi.

**Áp dụng khi nào?**
- Một hoặc nhiều AI host cần truy cập cùng hệ thống nội bộ.
- Muốn dùng lại tích hợp giữa IDE, agent, desktop app hoặc ứng dụng AI khác nhau.
- Cần khám phá tool/resource theo schema, quản lý phiên và chuẩn hóa lỗi.
- Muốn tách logic kết nối khỏi prompt và code điều phối riêng của từng model.

**Bài toán giải quyết:** giảm tích hợp point-to-point, chuẩn hóa hợp đồng giữa AI và dịch vụ, tạo ranh giới rõ cho kiểm thử, bảo mật và audit.

**So sánh:**

| Cách | Ưu điểm | Hạn chế | Khi phù hợp |
|---|---|---|---|
| Gọi REST/GraphQL API trực tiếp | Ít lớp, kiểm soát sâu, hợp với ứng dụng xác định | Mỗi host/agent có thể phải tự xây adapter và policy | Một ứng dụng, một dịch vụ, luồng ổn định |
| SDK/plugin riêng | Trải nghiệm tích hợp tốt | Dễ gắn với nền tảng cụ thể | Chỉ phục vụ một host |
| Gửi dữ liệu vào prompt/copy-paste | Dễ thử nghiệm | Dữ liệu nhanh lỗi thời, tốn context, khó audit | Demo hoặc việc thủ công nhỏ |
| **MCP** | Capability discovery, schema và giao thức chung | Cần vận hành server, auth/policy và tương thích phiên bản | Nhiều host, nhiều tích hợp hoặc agent workflow |

Nguồn: [MCP — Introduction](https://modelcontextprotocol.io/docs/getting-started/intro), [MCP — Architecture](https://modelcontextprotocol.io/docs/learn/architecture).

---

### 2. Các thành phần tham gia và ranh giới trách nhiệm là gì?

**Trả lời:** Kiến trúc điển hình gồm ba thành phần MCP và hệ thống nghiệp vụ phía sau.

```text
Người dùng
   │ yêu cầu bằng ngôn ngữ tự nhiên
   ▼
MCP Host (AI app / IDE / agent)
  - quản lý model, context, consent/approval, policy
   │
   ├── Model / planner quyết định có cần tool hay không
   │
   ▼
MCP Client (trong host)
  - quản lý kết nối, phiên, capability negotiation
  - gửi JSON-RPC request/nhận response
   │  MCP transport
   ▼
MCP Server (adapter/cổng tích hợp)
  - công bố tools/resources/prompts
  - validate input, kiểm tra quyền, gọi nghiệp vụ
   │
   ▼
Adapter / service / API / database hiện có
  - source of truth và quy tắc nghiệp vụ
```

| Thành phần | Trách nhiệm chính | Không nên mặc định giao cho nó |
|---|---|---|
| **Host** | UX, chọn model, context, chính sách ứng dụng và approval | Tin mọi tool output là an toàn |
| **Client** | Kết nối server, khởi tạo phiên, gửi request/nhận response | Tự cấp quyền nghiệp vụ |
| **MCP server** | Công bố capability, xác thực đầu vào, ánh xạ lời gọi sang hệ thống | Bỏ qua authorization của backend |
| **Backend nghiệp vụ** | Source of truth, validation nghiệp vụ, transaction và quyền dữ liệu | Phải hiểu prompt hoặc chạy model |
| **Identity/policy/audit layer** | Danh tính, scope, quyết định quyền và nhật ký | Chỉ dựa vào tên tool hoặc mô tả từ model |

**Áp dụng khi nào?** Khi chia ownership giữa team AI platform, team tích hợp và team backend; đặc biệt hữu ích để không nhầm MCP server với AI server.

**Bài toán giải quyết:** làm rõ nơi đặt credential, authorization, business validation, audit và trách nhiệm khi xảy ra lỗi.

Nguồn: [MCP — Architecture](https://modelcontextprotocol.io/docs/learn/architecture), [MCP — Server concepts](https://modelcontextprotocol.io/docs/learn/server-concepts).

---

### 3. Format giao tiếp MCP cụ thể trông như thế nào?

**Trả lời:** MCP dùng **JSON-RPC 2.0** cho message protocol. Các message thường là JSON object. Request có `jsonrpc`, `id`, `method`, có thể có `params`; response dùng cùng `id` và có `result` hoặc `error`. Notification không có `id` và không mong response.

Ví dụ minh họa dưới đây mô tả hình dạng message; tham số thực tế phụ thuộc method và phiên bản specification mà client/server đã thương lượng.

**A. Khởi tạo (initialize request):**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "VERSION_SUPPORTED_BY_CLIENT",
    "capabilities": {},
    "clientInfo": {
      "name": "example-ai-host",
      "version": "1.0.0"
    }
  }
}
```

Server trả về protocol version được chọn, server info và capabilities của nó. Client thường gửi notification `notifications/initialized` sau khi khởi tạo thành công.

**B. Khám phá tools (tools/list request):**

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/list",
  "params": {}
}
```

**C. Một tool definition đơn giản (ví dụ rút gọn):**

```json
{
  "name": "get_order_status",
  "description": "Đọc trạng thái đơn hàng mà người gọi được phép xem.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "orderId": { "type": "string" }
    },
    "required": ["orderId"]
  }
}
```

**D. Gọi tool (tools/call request):**

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "get_order_status",
    "arguments": {
      "orderId": "ORD-123"
    }
  }
}
```

Kết quả có thể chứa `content` (ví dụ text hoặc image content) và tùy khả năng/version có thể có `structuredContent` cùng `isError`. Tool result ở tầng MCP khác với JSON-RPC protocol error: một lỗi nghiệp vụ của tool có thể được biểu diễn trong tool result, còn lỗi JSON-RPC biểu diễn lỗi xử lý request ở tầng protocol.

**E. Response JSON-RPC thành công (minh họa):**

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"orderId\":\"ORD-123\",\"status\":\"shipped\"}"
      }
    ],
    "isError": false
  }
}
```

**Lưu ý:** đây là các ví dụ rút gọn để hiểu cấu trúc, không phải bộ message hoàn chỉnh cho mọi phiên bản. Không hard-code protocol version tùy tiện; hãy dùng version được client/server hỗ trợ và xác nhận theo specification chính thức.

**Áp dụng khi nào?** Khi debug raw message, viết MCP client/server, tạo contract test hoặc điều tra lỗi schema/transport.

**Bài toán giải quyết:** tạo hợp đồng trao đổi có thể parse và kiểm thử được thay vì dựa vào câu chữ tự do.

Nguồn: [JSON-RPC 2.0 specification](https://www.jsonrpc.org/specification), [MCP — Lifecycle](https://modelcontextprotocol.io/specification/latest/basic/lifecycle), [MCP — Tools](https://modelcontextprotocol.io/specification/latest/server/tools).

---

### 4. MCP có những capability nào, và khác nhau ra sao?

**Trả lời:** Các capability thường gặp ở MCP server là **Tools**, **Resources** và **Prompts**. Chúng không đồng nghĩa với nhau.

| Capability | Ý nghĩa | Ví dụ | Ai thường quyết định dùng |
|---|---|---|---|
| **Tools** | Hành động/truy vấn có tham số; có thể tạo side effect | `get_order_status`, `update_shipping_address` | Host/model đề xuất gọi; host/policy kiểm soát |
| **Resources** | Dữ liệu được định danh bằng URI, có thể đọc theo nhu cầu | `docs://policies/refund`, schema hoặc tài liệu | Ứng dụng/host thường chọn và đọc |
| **Prompts** | Template hướng dẫn/workflow tái sử dụng | Template phân tích incident hoặc review hợp đồng | Người dùng/host chọn |
| **Sampling** | Một số server có thể yêu cầu host thực hiện lượt sinh từ model, nếu được hỗ trợ và cho phép | Server cần tổng hợp hoặc xử lý có hỗ trợ model | Host vẫn kiểm soát việc sử dụng model |
| **Notifications** | Báo thay đổi/trạng thái trong phiên | Capability/resource list thay đổi, progress | Client/host xử lý theo capability |

Không phải server nào cũng cần cả ba primitive; chỉ công bố capability thực sự hỗ trợ. Resource không phải lúc nào cũng là một file vật lý và tool không phải lúc nào cũng ghi dữ liệu.

**Áp dụng khi nào?** Khi quyết định một capability nên là resource, tool hay prompt; khi muốn tránh tool `get_everything` quá rộng.

**Bài toán giải quyết:** tách dữ liệu, hành động và quy trình; từ đó rõ hơn về caching, approval, quyền và trải nghiệm người dùng.

Nguồn: [MCP — Server concepts](https://modelcontextprotocol.io/docs/learn/server-concepts), [MCP — Resources](https://modelcontextprotocol.io/specification/latest/server/resources), [MCP — Tools](https://modelcontextprotocol.io/specification/latest/server/tools).

---

### 5. Phân biệt MCP read với MCP update/write như thế nào?

**Trả lời:** Không có một giao thức riêng mang tên “MCP read” và một giao thức khác tên “MCP update”. Đây là cách phân loại **hành vi capability và quyền hạn**.

| Khía cạnh | Read/query | Update/write/action |
|---|---|---|
| Mục đích | Lấy thông tin từ source of truth | Thay đổi trạng thái hoặc kích hoạt hành động |
| Ví dụ | `get_customer`, `search_orders`, `read_file` | `update_customer`, `cancel_order`, `create_ticket` |
| Side effect | Thường không; vẫn có thể có chi phí hoặc ghi audit | Có thể thay đổi dữ liệu, gửi thông báo, tạo giao dịch |
| Quyền | Scope đọc theo tenant/record/field | Scope ghi cụ thể, kiểm tra trạng thái và quyền mạnh hơn |
| Approval | Thường tự động nếu policy cho phép | Tùy rủi ro; thường cần xác nhận cho hành động trọng yếu |
| Idempotency | Cần kiểm soát truy vấn, pagination/cache | Cần idempotency key hoặc chống thực hiện lặp khi phù hợp |
| Audit | Ghi ai đọc gì theo yêu cầu bảo mật | Ghi actor, mục tiêu, thay đổi trước/sau, approval và kết quả |

**Thiết kế khuyến nghị:**
1. Đặt tên read/write rõ ràng, không giấu side effect trong tool có tên “get”.
2. Tách credential và scope khi có thể; mặc định read-only.
3. Backend vẫn thực thi authorization và business validation cho từng bản ghi.
4. Với write, kiểm tra precondition/version, validate dữ liệu, transaction và idempotency.
5. Trả kết quả xác nhận có định danh và trạng thái; không tuyên bố thành công chỉ vì request đã gửi.
6. Dùng approval cho xóa, chuyển tiền, publish, deploy hoặc hành động khó hoàn tác theo mức rủi ro.

**Áp dụng khi nào?** Bất kỳ AI agent nào có thể cập nhật CRM/ERP, tạo ticket, sửa repository, thay đổi database hoặc gọi workflow vận hành.

**Bài toán giải quyết:** giảm rủi ro prompt injection, thao tác lặp, confused deputy và thay đổi ngoài ý muốn; đồng thời vẫn cho phép AI thực hiện công việc có kiểm soát.

Nguồn: [MCP — Tools](https://modelcontextprotocol.io/specification/latest/server/tools), [MCP — Security Best Practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices).

---

## B. 8 câu hỏi phổ biến

### 6. Hệ thống bên ngoài đang có REST API/database thì cần thêm gì để hỗ trợ MCP?

**Trả lời:** Thường không cần viết lại backend. Thêm một **MCP server/adapter** làm lớp chuyển đổi giữa giao thức MCP và API/service hiện hữu là đủ, nếu API hiện tại có thể thực hiện nghiệp vụ cần thiết.

```text
AI Host / Agent
   │ MCP client + JSON-RPC
   ▼
MCP server / adapter
   ├── tools/list, resources/list/read, prompts/list/get
   ├── JSON Schema validation
   ├── identity / authorization / policy
   ├── mapping MCP arguments ↔ API request
   ├── mapping API response/error ↔ MCP result/error
   ├── timeout / retry / rate limit / audit
   ▼
REST API / GraphQL / application service
   ▼
Database và hệ thống nghiệp vụ hiện hữu
```

**Những phần cần bổ sung hoặc xác minh:**

| Thành phần | Cần làm gì | Bắt buộc trong mọi dự án? |
|---|---|---|
| MCP server/adapter | Công bố tool/resource/prompt và ánh xạ sang nghiệp vụ | Có nếu hệ thống chưa có MCP server phù hợp |
| Capability contract | Tên, mô tả, input schema, output và error contract | Có |
| Authentication | Xác định danh tính host/user/service và credential flow | Có; cách triển khai phụ thuộc transport |
| Authorization | Kiểm tra tenant, record, field và action trên backend | Có |
| Validation | Kiểm tra kiểu, range, enum, trạng thái nghiệp vụ | Có |
| Read/write separation | Scope và approval theo side effect | Cần cho mọi hệ thống có write; mức chi tiết tùy rủi ro |
| Reliability | Timeout, giới hạn payload, retry có kiểm soát, idempotency | Theo tính chất tác vụ; write cần đặc biệt cẩn trọng |
| Audit/observability | Request ID, actor, tool, target, kết quả, latency và lỗi | Nên có; thường bắt buộc với dữ liệu nhạy cảm |
| MCP client/host | Ứng dụng AI kết nối được tới server và quản lý phiên | Có ở phía AI; có thể dùng host có sẵn |
| Test và vận hành | Contract test, kiểm thử quyền, monitoring, versioning | Có cho production |

**Ví dụ:** API hiện có `GET /orders/{id}` và `PATCH /orders/{id}`. MCP adapter có thể cung cấp `get_order` và `update_order`. Adapter chuyển `orderId` thành URL/path parameter, kiểm tra schema và quyền, gọi API hiện hữu rồi chuyển kết quả thành MCP tool result. Quy tắc “đơn đã giao không được hủy” vẫn phải do service nghiệp vụ thực thi.

**Áp dụng khi nào?** Tích hợp hệ thống legacy hoặc SaaS không hỗ trợ MCP trực tiếp.

**Bài toán giải quyết:** giúp AI dùng hệ thống cũ qua một contract chuẩn mà không buộc hệ thống cũ phụ thuộc vào model.

Nguồn: [MCP — Build a server](https://modelcontextprotocol.io/docs/develop/build-server), [MCP — Authorization](https://modelcontextprotocol.io/specification/latest/basic/authorization).

---

### 7. Có bắt buộc phải có “AI server” riêng không?

**Trả lời:** Không. Cần phân biệt **AI model service**, **MCP host/client** và **MCP server**:

- **AI model service:** chạy hoặc cung cấp LLM; có thể là API cloud hoặc model tự host.
- **MCP host/client:** ứng dụng dùng model và quản lý kết nối MCP, ví dụ IDE/desktop app/agent backend.
- **MCP server:** expose capability và kết nối tới hệ thống nghiệp vụ; thường không cần gọi LLM.
- **Backend nghiệp vụ:** tiếp tục là nơi thực thi quy tắc nghiệp vụ và giữ source of truth.

Có thể dùng ứng dụng AI hiện có làm host, chạy MCP server như process local qua STDIO, hoặc triển khai MCP server từ xa qua Streamable HTTP. Nếu tự xây sản phẩm AI, backend ứng dụng có thể đóng vai host/client; không nhất thiết phải tách thành một máy chủ AI độc lập.

**Áp dụng khi nào?** Khi lập kiến trúc và ngân sách triển khai; tránh nhầm “cần MCP” với “phải dựng thêm LLM server”.

**Bài toán giải quyết:** giảm thành phần dư thừa và tách đúng trách nhiệm giữa inference, điều phối, tích hợp và nghiệp vụ.

Nguồn: [MCP — Architecture](https://modelcontextprotocol.io/docs/learn/architecture).

---

### 8. MCP chạy qua transport nào? STDIO và Streamable HTTP khác nhau ra sao?

**Trả lời:** Transport là cách chuyển message MCP giữa client và server; nó không thay đổi ý nghĩa của capability.

| Tiêu chí | STDIO | Streamable HTTP |
|---|---|---|
| Kiểu triển khai | Client khởi chạy process local và trao đổi qua stdin/stdout | Client gửi request HTTP tới MCP endpoint; có thể hỗ trợ streaming |
| Trường hợp điển hình | Tool filesystem/local repo, compiler, môi trường developer | Server dùng chung, cloud API, hệ thống doanh nghiệp |
| Authentication | Thường dựa vào process/OS và môi trường chạy; vẫn phải bảo vệ quyền local | Có thể dùng cơ chế auth phù hợp, OAuth khi cần |
| Vận hành | Phân phối package/binary và quản lý process | Deploy, TLS, network policy, scale và monitor service |
| Rủi ro | Process có thể thừa hưởng quyền máy người dùng | Lộ endpoint, token/session, cấu hình proxy và network |

Không ghi log tùy tiện vào stdout của server STDIO nếu stdout được dùng làm kênh protocol; log nên đi qua stderr hoặc logging phù hợp. Với remote transport, cần TLS, auth, kiểm tra origin/host theo hướng dẫn bảo mật và giới hạn mạng.

**Áp dụng khi nào?** STDIO cho tích hợp local gắn với workspace; Streamable HTTP cho server cần chia sẻ, quản lý tập trung hoặc chạy từ xa.

**Bài toán giải quyết:** chọn đúng mô hình kết nối theo latency, scale, credential và trust boundary.

Nguồn: [MCP — Transports](https://modelcontextprotocol.io/specification/latest/basic/transports).

---

### 9. MCP server có thể gọi database trực tiếp không, hay phải đi qua API?

**Trả lời:** Cả hai đều có thể, nhưng lựa chọn phụ thuộc kiến trúc và mức rủi ro. Với nghiệp vụ quan trọng, nên gọi **application service/API** để tái sử dụng transaction, validation và authorization. Truy cập DB trực tiếp phù hợp hơn cho truy vấn read-only có giới hạn hoặc hệ thống mà service layer không tồn tại, với quyền DB rất hẹp.

| Cách | Ưu điểm | Rủi ro/giới hạn |
|---|---|---|
| Qua API/service | Tái sử dụng business rule, auth, transaction | Thêm hop và phụ thuộc API |
| Đọc DB trực tiếp | Truy vấn linh hoạt, có thể hiệu quả | Dễ bypass business rule; lộ dữ liệu rộng; schema coupling |
| Ghi DB trực tiếp | Có thể làm được về kỹ thuật | Nguy cơ cao: invariant, audit, transaction và side effect bị bỏ qua |

Không cho model tự sinh SQL tự do với quyền production rộng. Ưu tiên tool có tham số typed, query allowlist, read-only role, row/tenant filters, query timeout và giới hạn số dòng. Với ghi dữ liệu, gọi service nghiệp vụ có kiểm soát thay vì cấp quyền SQL tổng quát.

**Áp dụng khi nào?** Khi quyết định adapter nên gọi API nào và cấp credential gì.

**Bài toán giải quyết:** giữ nguyên tính toàn vẹn nghiệp vụ và giảm nguy cơ data exfiltration hoặc sửa dữ liệu ngoài phạm vi.

Nguồn: [MCP — Security Best Practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices).

---

### 10. Làm sao bảo vệ hệ thống khỏi prompt injection và tool call nguy hiểm?

**Trả lời:** Không được coi văn bản do model tạo hoặc dữ liệu tool trả về là ranh giới bảo mật. Issue, email, trang web, file và log đều có thể chứa chỉ dẫn độc hại; server/host phải thực thi policy độc lập với nội dung đó.

Checklist production:

- **Least privilege:** credential riêng, scope nhỏ, giới hạn tenant và môi trường.
- **Backend authorization:** xác minh user/action/resource ở mọi lần gọi; không chỉ kiểm tra trong prompt.
- **Read-only by default:** bật quyền ghi khi cần và theo tác vụ.
- **Approval theo rủi ro:** yêu cầu xác nhận trước hành động khó hoàn tác.
- **Schema/validation:** allowlist tham số, giới hạn kích thước, path và enum; không nối input vào shell/SQL không an toàn.
- **Output minimization:** chỉ trả field cần thiết; redact secret/PII.
- **Audit:** actor, tool, target, approval, timestamp, outcome và correlation ID.
- **Isolation:** sandbox process, giới hạn network và filesystem.
- **Reliability:** timeout, rate limit, giới hạn concurrency; retry chỉ khi an toàn.
- **Supply chain:** pin dependency, kiểm tra nguồn server và cập nhật bảo mật.

**Áp dụng khi nào?** Mọi MCP server chạm dữ liệu nội bộ, code, cloud, CRM/ERP, CI/CD hoặc production.

**Bài toán giải quyết:** giảm rò dữ liệu, quyền vượt mức, confused deputy, command injection và hành động trái phép.

Nguồn: [MCP — Security Best Practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices), [MCP — Authorization](https://modelcontextprotocol.io/specification/latest/basic/authorization).

---

### 11. Thiết kế output và lỗi thế nào để AI xử lý đáng tin cậy?

**Trả lời:** Dùng output nhỏ, có cấu trúc và giới hạn; phân biệt rõ lỗi protocol, lỗi tool và lỗi nghiệp vụ. Không trả hàng chục nghìn dòng log hoặc toàn bộ bản ghi nếu chỉ cần trạng thái và vài trường.

Ví dụ output nghiệp vụ nên ưu tiên một cấu trúc ổn định như:

```json
{
  "orderId": "ORD-123",
  "status": "shipped",
  "updatedAt": "2026-10-14T10:00:00Z",
  "source": "order-service"
}
```

Khuyến nghị:
- Khai báo output schema khi khả năng/spec hỗ trợ.
- Dùng pagination/cursor, filter, field selection và giới hạn kích thước.
- Với payload lớn, trả summary + stable ID/URI để đọc tiếp.
- Trả lỗi có thể xử lý: mã lỗi, thông điệp an toàn, có retry được không và correlation ID.
- Không để exception/stack trace hoặc secret lọt vào model context.
- Không tự retry write nếu không biết operation có idempotent hay không.
- Không coi HTTP 200 hoặc JSON-RPC response hợp lệ là bằng chứng nghiệp vụ thành công; xác minh trạng thái kết quả.

**Áp dụng khi nào?** Tool trả log, báo cáo, bảng dữ liệu lớn hoặc khi agent thường hiểu sai lỗi.

**Bài toán giải quyết:** giảm token, tránh context overflow, tăng khả năng recovery và giúp điều tra lỗi.

Nguồn: [MCP — Tools](https://modelcontextprotocol.io/specification/latest/server/tools), [JSON-RPC 2.0](https://www.jsonrpc.org/specification).

---

### 12. Cần kiểm thử và theo dõi gì trước khi đưa MCP vào production?

**Trả lời:** Kiểm thử cả protocol, quyền, mapping nghiệp vụ và chất lượng tác vụ của agent.

1. **Contract test:** initialize/version negotiation, list capability, schema, result/error, pagination và transport.
2. **Authorization test:** user A không đọc/ghi dữ liệu user B; kiểm tra tenant, field, environment và scope.
3. **Side-effect test:** write bị từ chối khi thiếu quyền/approval; kiểm tra idempotency, transaction và trạng thái sau lỗi.
4. **Reliability test:** timeout, server unavailable, rate limit, concurrent call, payload lớn và cancellation nếu hỗ trợ.
5. **Injection test:** dữ liệu độc hại trong file/log/issue không làm agent vượt policy.
6. **Agent evaluation:** chọn đúng tool, truyền đúng tham số, dùng evidence đúng và biết dừng khi thiếu quyền/dữ liệu.
7. **Observability:** đo latency, error rate, retry, token/context, số lần gọi tool, task success và chi phí.
8. **Versioning:** kiểm thử khi nâng MCP SDK/spec, schema hoặc backend API.

Dùng MCP Inspector hoặc bộ contract test tự động để xem request/result và thử edge case; không chỉ dựa vào một demo thành công.

**Áp dụng khi nào?** Trước release, khi thay host/model/SDK, hoặc khi cấp thêm write capability.

**Bài toán giải quyết:** phân biệt lỗi transport, lỗi adapter, lỗi backend và lỗi quyết định của model; ngăn regression về bảo mật và chất lượng.

Nguồn: [MCP — Inspector](https://modelcontextprotocol.io/docs/tools/inspector), [MCP — Debugging](https://modelcontextprotocol.io/docs/tools/debugging).

---

### 13. Khi nào không nên thêm MCP?

**Trả lời:** Không nên thêm MCP chỉ vì đang dùng AI. Nếu một ứng dụng duy nhất gọi một API ổn định, direct SDK/API có thể đơn giản hơn. Nếu bước xử lý hoàn toàn deterministic, script hoặc workflow engine thường dễ dự đoán hơn agent gọi tool nhiều vòng.

| Nhu cầu | Lựa chọn nên cân nhắc |
|---|---|
| Một ứng dụng, một API, contract ổn định | Direct API/SDK |
| Job định kỳ và logic xác định | Cron/workflow engine/queue |
| Tìm kiếm tri thức trong tài liệu | Search/RAG; MCP có thể expose nó như tool/resource |
| Nhiều AI host cần dùng lại cùng integration | MCP |
| Quy trình hướng dẫn và quy ước thao tác | Prompt/Agent Skills; MCP bổ sung kết nối nếu cần |
| Hành động ghi có rủi ro cao | MCP hoặc API đều được, nhưng cần policy, authorization và approval tương xứng |

MCP chuẩn hóa lớp kết nối; nó không tự thay thế RAG, workflow engine, API nghiệp vụ, IAM hay hệ thống audit.

**Áp dụng khi nào?** Trong thiết kế kiến trúc và đánh giá chi phí vận hành.

**Bài toán giải quyết:** tránh thêm một tầng phức tạp mà không tạo ra khả năng tái sử dụng, quản trị hoặc trải nghiệm tốt hơn.

---

## Checklist triển khai nhanh

- [ ] Xác định use case và source of truth; liệt kê dữ liệu/action mà AI thật sự cần.
- [ ] Chọn host/client hiện có hay tự xây; xác định server local (STDIO) hay remote (Streamable HTTP).
- [ ] Thiết kế tool/resource/prompt với tên rõ, schema typed và output giới hạn.
- [ ] Dùng MCP adapter gọi API/service hiện hữu trước khi cân nhắc thay đổi backend.
- [ ] Tách read/write, dùng least privilege và backend authorization.
- [ ] Có validation nghiệp vụ, timeout, rate limit, audit và correlation ID.
- [ ] Write có precondition, idempotency phù hợp và approval theo mức rủi ro.
- [ ] Kiểm thử quyền, lỗi, payload lớn, prompt injection và agent task success.
- [ ] Theo dõi latency, error rate, token, tool calls, cost và business outcome.
- [ ] Pin/version dependency và kiểm thử tương thích khi nâng phiên bản.

## Tài liệu tham khảo

1. [Model Context Protocol — Introduction](https://modelcontextprotocol.io/docs/getting-started/intro)
2. [Model Context Protocol — Architecture](https://modelcontextprotocol.io/docs/learn/architecture)
3. [Model Context Protocol — Lifecycle](https://modelcontextprotocol.io/specification/latest/basic/lifecycle)
4. [Model Context Protocol — Transports](https://modelcontextprotocol.io/specification/latest/basic/transports)
5. [Model Context Protocol — Tools](https://modelcontextprotocol.io/specification/latest/server/tools)
6. [Model Context Protocol — Resources](https://modelcontextprotocol.io/specification/latest/server/resources)
7. [Model Context Protocol — Authorization](https://modelcontextprotocol.io/specification/latest/basic/authorization)
8. [Model Context Protocol — Security Best Practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices)
9. [Model Context Protocol — Build a server](https://modelcontextprotocol.io/docs/develop/build-server)
10. [Model Context Protocol — Inspector](https://modelcontextprotocol.io/docs/tools/inspector)
11. [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification)
