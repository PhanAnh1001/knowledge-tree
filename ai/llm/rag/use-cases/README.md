# Các trường hợp ứng dụng RAG (RAG Use Cases)

> [Knowledge Tree](../../../../README.md) / [AI](../../../README.md) / [LLM](../../README.md) / [RAG](../README.md) / **Các trường hợp ứng dụng RAG**

**Retrieval-Augmented Generation (RAG)** không có một danh sách use case đóng, vì cùng một năng lực “tìm bằng chứng rồi tổng hợp câu trả lời” có thể được áp dụng cho nhiều ngành. Trang này cung cấp **bản đồ bao quát** để chọn đúng bài toán, nguồn dữ liệu, kiến trúc và mức kiểm soát; không nên hiểu RAG là giải pháp thay thế cho API, database, rule engine hay quy trình phê duyệt.

> **Mẫu quyết định:** Câu hỏi cần tri thức riêng hoặc thay đổi → retrieve evidence đúng quyền/phạm vi → LLM giải thích có citation → con người hoặc workflow xử lý quyết định/hành động rủi ro cao.

## Điều hướng

- **Node cha:** [RAG](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** [Chỉ mục pgvector](../pgvector-index/README.md)
- **Khái niệm sau trong cùng nhánh:** Chưa có.

## Bản đồ nhanh: RAG giải quyết những loại việc gì?

| Nhóm công việc (*job to be done*) | Ví dụ | Dạng RAG thường dùng | Không nên chỉ dùng RAG khi |
| --- | --- | --- | --- |
| Hỏi–đáp có nguồn (*grounded Q&A*) | “Chính sách hoàn tiền cho gói Pro là gì?” | Document RAG + citation | Cần trạng thái realtime hoặc tính tiền chính xác. |
| Tìm và dẫn đường tri thức (*knowledge discovery*) | “Runbook nào xử lý lỗi thanh toán này?” | Hybrid search + lọc metadata | Người dùng chỉ cần search keyword đơn giản. |
| Tóm tắt theo ngữ cảnh (*contextual summarization*) | “Tóm tắt thay đổi policy ảnh hưởng đội CSKH.” | Retrieve section liên quan → summarize | Phải tóm tắt toàn bộ một tài liệu ngắn đã nằm vừa context. |
| So sánh và truy vết bằng chứng | “Bản v3 khác v2 ở điều kiện đổi trả nào?” | Multi-document RAG + version filter | Cần diff có tính pháp lý; khi đó cần document-diff có kiểm chứng. |
| Hỗ trợ ra quyết định (*decision support*) | “Các điều kiện nào cần kiểm tra trước khi duyệt yêu cầu?” | RAG + checklist/rules + human review | Hệ thống phải tự quyết hoặc tự phê duyệt quyết định rủi ro cao. |
| Trợ lý tác vụ (*task assistance*) | “Hướng dẫn tạo ticket sự cố đúng quy trình.” | RAG + workflow/tool guidance | Có side effect: tạo, sửa, duyệt giao dịch phải qua API/workflow có authz. |

## A. 5 câu hỏi cốt lõi

### 1. RAG phù hợp với những đặc điểm bài toán nào?

**Trả lời:** RAG phù hợp nhất khi câu trả lời phụ thuộc vào **tri thức ngoài model**: tài liệu nội bộ, quy định, hướng dẫn, catalogue, release note, knowledge base hoặc nhiều phiên bản dữ liệu cần được tìm đúng tại thời điểm hỏi. Giá trị chính không phải chỉ là “trả lời tự nhiên”, mà là **lấy được evidence phù hợp, trong đúng phạm vi quyền, rồi giúp người dùng kiểm chứng**.

**Dấu hiệu nên dùng RAG:**

- Người hỏi cần một câu trả lời từ nguồn riêng của tổ chức hoặc từ dữ liệu thay đổi theo thời gian.
- Nội dung nằm trong văn bản dài, nhiều file, nhiều hệ thống hoặc nhiều ngôn ngữ.
- Cần link đến trang, section, phiên bản, ngày hiệu lực hoặc tài liệu gốc.
- Câu hỏi có cách diễn đạt đa dạng, nhưng không thể chỉ dựa vào exact keyword.
- Câu trả lời cần tuân thủ tenant, role, product, region, trạng thái tài liệu hoặc ngày hiệu lực.

**Dấu hiệu không nên lấy RAG làm lõi:** dữ liệu là trạng thái giao dịch realtime; yêu cầu là phép tính chính xác; tác vụ có tác động như tạo đơn, đổi số dư, phê duyệt hồ sơ; hoặc knowledge base quá nhỏ và ổn định để đặt trực tiếp trong prompt.

**Áp dụng khi nào?** Dùng như một lớp *knowledge retrieval* cho chatbot, copilot, search portal, trợ lý nghiệp vụ, trợ lý kỹ thuật và phân tích tài liệu.

**Bài toán giải quyết:** giảm thời gian tìm tài liệu, tăng khả năng truy vết, cập nhật tri thức không cần huấn luyện lại LLM, và giảm suy đoán khi model không có hoặc không được phép có dữ liệu cần thiết.

**So sánh với lựa chọn tương tự:**

| Lựa chọn | Mạnh khi | Giới hạn so với RAG |
| --- | --- | --- |
| Search engine truyền thống | Người dùng muốn tự đọc danh sách tài liệu. | Không tổng hợp theo hội thoại hoặc trả lời trực tiếp có cấu trúc. |
| Long context | Ít tài liệu, khởi động nhanh, cần đọc trọn một tập nhỏ. | Tốn token, khó filter/version/ACL khi corpus lớn. |
| Fine-tuning | Cần học phong cách, format, taxonomy hoặc cách dùng tool ổn định. | Không phải cơ chế cập nhật knowledge mới hàng ngày. |
| API/database/rule engine | Cần sự thật realtime, tính toán đúng hoặc side effect. | Không giỏi tìm ý nghĩa trong corpus văn bản dài. |
| RAG | Cần find–select–explain evidence từ knowledge corpus. | Phụ thuộc chất lượng dữ liệu, retrieval và answer policy. |

---

### 2. Có những nhóm ứng dụng RAG nào theo nghiệp vụ và ngành?

**Trả lời:** Có thể phân loại theo “người dùng cần biết hoặc làm gì”, thay vì chỉ theo tên ngành. Bảng dưới bao phủ các nhóm use case phổ biến nhất; một sản phẩm thực tế thường kết hợp nhiều nhóm.

| Nhóm ứng dụng | Câu hỏi/tác vụ tiêu biểu | Nguồn tri thức | Pattern nên bắt đầu | Điểm kiểm soát quan trọng |
| --- | --- | --- | --- | --- |
| Tri thức nội bộ (*enterprise knowledge*) | Quy trình, SOP, onboarding, HR, IT, policy nội bộ. | Wiki, Drive, SharePoint, Git, handbook. | Hybrid RAG + ACL. | Role, phòng ban, ngày hiệu lực. |
| CSKH và self-service | Đổi trả, bảo hành, hướng dẫn sử dụng, khiếu nại. | FAQ, policy, manual, CRM knowledge. | RAG + citation + human handoff. | Không bịa chính sách; phân biệt thông tin cá nhân và policy công khai. |
| Hỗ trợ kỹ thuật và developer | Error code, API, runbook, release note, kiến trúc. | Docs, code docs, incident postmortem, ticket đã xử lý. | Hybrid + version/product filter + rerank. | Exact match cho code/error; version chính xác. |
| Bán hàng và thương mại điện tử | So sánh sản phẩm, tư vấn phù hợp, điều kiện khuyến mãi. | Catalogue, spec, FAQ, campaign rules. | RAG + product filter + API giá/tồn kho. | Giá/tồn kho là API source of truth. |
| Pháp lý, tuân thủ và audit | Điều khoản, quy định, nghĩa vụ, checklist, policy mapping. | Contract, law/policy, control library, audit evidence. | Section-aware RAG + version/citation. | Human review; không kết luận pháp lý tự động. |
| Tài chính, bảo hiểm và rủi ro | Quy trình hồ sơ, coverage, claim guidance, KYC policy. | Product policy, procedure, regulatory docs. | RAG + rule/API + approval workflow. | Không tự cấp tín dụng, chi trả hoặc quyết định hồ sơ. |
| Y tế và khoa học sự sống | Hướng dẫn lâm sàng, protocol, dược phẩm, SOP lab. | Guideline được phê duyệt, literature, SOP. | Curated RAG + provenance + review. | Không thay thế chẩn đoán/điều trị; kiểm soát nguồn và thời hạn. |
| Giáo dục và đào tạo | Học liệu, tutor, hỏi đáp môn học, hỗ trợ giảng viên. | Giáo trình, bài giảng, rubric, knowledge base. | Course-scoped RAG + citation. | Academic integrity, phạm vi môn học, chống đáp án không có nguồn. |
| Vận hành, sản xuất và logistics | Hướng dẫn bảo trì, troubleshooting, tiêu chuẩn an toàn, vận chuyển. | Manual, BOM, SOP, lịch sử sự cố, checklist. | Multimodal/technical RAG + asset filter. | Phiên bản thiết bị, an toàn lao động, approval khi thao tác rủi ro. |
| An ninh và IT operations | Runbook SOC/NOC, xử lý sự cố, playbook, policy. | Runbook, KB, alert taxonomy, postmortem. | RAG + tool guidance + audit trace. | Không cho agent tự thực thi destructive action mặc định. |
| Nghiên cứu và phân tích | Tổng hợp paper, market research, due diligence. | Paper, report, note, database export. | Multi-document RAG + citation map. | Phân biệt evidence với suy luận; kiểm tra nguồn đối lập. |
| Nội dung, media và editorial | Tìm tư liệu, fact-check nội bộ, tái sử dụng content. | CMS, archive, style guide, asset catalog. | Metadata-rich RAG. | Bản quyền, nguồn, thời gian xuất bản. |
| Dịch vụ công | Hướng dẫn thủ tục, eligibility, hồ sơ, biểu mẫu. | Quy định, thủ tục, FAQ cơ quan. | Policy RAG + form/workflow routing. | Điều kiện pháp lý và phiên bản văn bản. |
| Du lịch, bất động sản, hospitality | Tư vấn gói dịch vụ, amenity, quy định lưu trú, hợp đồng. | Catalogue, policy, listing, local guide. | RAG + availability/pricing API. | Giá và tình trạng còn chỗ realtime. |
| Thu mua và quản trị nhà cung cấp | Quy chế mua sắm, đánh giá vendor, điều khoản, RFP. | Procurement policy, contract, vendor docs. | RAG + structured extraction + workflow. | Bảo mật hồ sơ thầu, phân quyền nhóm đánh giá. |
| Đa ngôn ngữ và localization | Hỏi bằng tiếng Việt về tài liệu tiếng Anh/nhật; chuẩn hóa thuật ngữ. | Glossary, manual đa ngôn ngữ, style guide. | Multilingual RAG + locale filter. | Kiểm tra chất lượng dịch thuật, tên sản phẩm và điều kiện pháp lý. |

**Áp dụng khi nào?** Chọn một hàng làm điểm khởi đầu, sau đó xác định thêm rủi ro, loại dữ liệu và action cần thực hiện.

**Bài toán giải quyết:** chuyển knowledge nằm rải rác trong file, wiki và hệ thống chuyên môn thành trải nghiệm hỏi–đáp có kiểm chứng, thay vì để người dùng mở thủ công nhiều nguồn.

**So sánh theo mức rủi ro:** nhóm kiến thức thông tin có thể tự phục vụ; nhóm tư vấn chuyên môn cần citation và cảnh báo; nhóm có quyết định pháp lý, y tế, tài chính hoặc hành động vận hành phải thêm rule, approval và người chịu trách nhiệm.

---

### 3. Chọn kiến trúc RAG nào cho từng loại use case?

**Trả lời:** Hãy bắt đầu từ pattern đơn giản nhất thỏa được rủi ro và chất lượng, không bắt đầu bằng agent. Kiến trúc tăng dần theo độ phức tạp của nguồn, nhu cầu realtime, quan hệ dữ liệu và mức tự động hóa.

| Pattern | Khi dùng | Ví dụ | Thành phần chính | Đánh đổi |
| --- | --- | --- | --- | --- |
| FAQ/document RAG | Một corpus vừa phải, câu hỏi độc lập. | Handbook nhân sự. | Chunk, vector/hybrid search, LLM, citation. | Nhanh làm nhưng dễ thiếu version/ACL nếu làm tối giản. |
| Hybrid RAG | Có cả câu tự nhiên và mã/chính xác. | Support lỗi `PAY-401`, API docs. | BM25 + vector + metadata filter. | Cần tuning score/rank. |
| Federated RAG | Tri thức ở nhiều repository/hệ thống. | Wiki + Git + ticket + Drive. | Connectors, canonical metadata, dedup/version. | Khó đồng nhất quyền và lifecycle. |
| Multi-tenant/ACL RAG | Có tenant, department hoặc dữ liệu nhạy cảm. | SaaS B2B, intranet. | Scope resolver, hard filter, audit trace. | ACL là phần correctness, không chỉ là feature UI. |
| Tool-augmented RAG | Cần vừa giải thích vừa lấy dữ liệu live. | Tracking đơn hàng + chính sách giao. | RAG router + API/tool + output policy. | Phải tách knowledge lookup khỏi operational truth. |
| Multimodal RAG | Đáp án nằm trong bảng, sơ đồ, ảnh, scan hoặc PDF layout. | Manual thiết bị, báo cáo scan. | OCR/layout parser, image/table retrieval, citation trang. | Parser sai làm retrieval sai; cần kiểm thử trên hình/bảng thật. |
| Graph/relationship-aware RAG | Query cần đi qua quan hệ nhiều bước. | “Sản phẩm nào phụ thuộc service nào?” | Entity/relationship store + vector retrieval. | Tốn chi phí mô hình hóa và duy trì graph. |
| Agentic RAG | Đường đi không xác định, phải chọn nhiều search/tool. | Điều tra nhiều nguồn có giới hạn. | Planner, tool policy, loop limits, approval. | Latency, cost và bề mặt lỗi cao hơn workflow xác định trước. |

**Nguyên tắc:** Nếu luồng đã biết, dùng workflow xác định trước. Ví dụ: `auth → xác định tenant → retrieve policy → gọi API đơn hàng → generate answer`. Agent chỉ nên được dùng khi chính câu hỏi quyết định đường đi và mọi tool đã có scope, timeout, budget, log và policy rõ ràng.

**Áp dụng khi nào?** Chọn pattern dựa trên evidence cần lấy, không dựa trên mức độ “AI trông thông minh”.

**Bài toán giải quyết:** tránh đầu tư quá mức vào agent/graph cho FAQ đơn giản, đồng thời tránh dùng naive RAG cho data có ACL, version hoặc dependency phức tạp.

---

### 4. Làm thế nào phân biệt RAG với search, long context, fine-tuning, SQL và tool calling trong một sản phẩm?

**Trả lời:** Các kỹ thuật này bổ sung nhau. Câu hỏi quan trọng là “phần nào là tri thức văn bản, phần nào là dữ liệu có cấu trúc/realtime, phần nào là hành động, phần nào là hành vi model?”.

| Nhu cầu | Cơ chế chính | Vai trò RAG |
| --- | --- | --- |
| Tìm đúng tài liệu để người dùng tự đọc | Search portal. | Có thể thêm answer summary và semantic search. |
| Trả lời từ quy định/manual/wiki | RAG. | Là cơ chế chính để lấy evidence. |
| Tóm tắt một tài liệu nhỏ người dùng vừa tải lên | Long context. | Có thể không cần index dài hạn. |
| Trả lời từ hàng nghìn tài liệu thay đổi | RAG + index. | Giảm context và tăng khả năng filter/citation. |
| Lấy đơn hàng, số dư, giá, tồn kho hiện tại | API/database/SQL. | Giải thích policy, thuật ngữ và hướng dẫn bước tiếp theo. |
| Tính thuế/chiết khấu/điểm | Calculator/rule engine. | Retrieve điều kiện và diễn giải kết quả. |
| Tạo ticket, thay đổi hồ sơ, phê duyệt | Workflow/tool calling + authz. | Hướng dẫn, xác định quy trình, hỗ trợ người dùng điền đúng. |
| Cố định format, phong cách hoặc cách gọi tool | Prompt/fine-tuning. | Cung cấp knowledge biến động cho model đã được điều chỉnh hành vi. |

**Áp dụng khi nào?** Đây là câu hỏi thiết kế bắt buộc trước khi thêm RAG vào bất kỳ chatbot nào.

**Bài toán giải quyết:** tránh “RAG hóa” mọi thứ, đặc biệt là làm model suy đoán trạng thái giao dịch, số liệu hoặc kết quả tính toán mà hệ thống có nguồn xác định.

---

### 5. Đánh giá use case RAG trước khi triển khai production như thế nào?

**Trả lời:** Một use case tốt phải chứng minh được cả **giá trị nghiệp vụ** lẫn **khả năng làm đúng**. Thử nghiệm bằng câu hỏi thật, không chỉ demo vài câu dễ.

**Checklist chọn use case:**

1. Người dùng hiện mất thời gian tìm gì? Tần suất, chi phí và rủi ro của lỗi là bao nhiêu?
2. Có source of truth rõ ràng không? Ai sở hữu và cập nhật tài liệu?
3. Câu trả lời có thể được chứng minh bằng section/page/version cụ thể không?
4. Quyền truy cập được suy ra từ backend hay chỉ do client tự gửi?
5. Nếu không tìm đủ evidence, hệ thống từ chối hoặc chuyển người xử lý thế nào?
6. Có dữ liệu đánh giá gồm câu hỏi đúng, câu mơ hồ, câu ngoại lệ, câu ngoài phạm vi và câu không có quyền không?
7. Chỉ số thành công là gì: deflection, resolution time, search success, error reduction, compliance, CSAT hay giảm thời gian onboarding?

| Mức rủi ro | Ví dụ | Hành vi chấp nhận được |
| --- | --- | --- |
| Thấp | FAQ công khai, tài liệu hướng dẫn. | Trả lời kèm link nguồn; có fallback search. |
| Trung bình | SOP nội bộ, technical troubleshooting, hỗ trợ nhân viên. | Citation, ACL, feedback, human escalation. |
| Cao | Pháp lý, tài chính, y tế, an toàn vận hành. | Curated sources, evidence rõ, cảnh báo, review/phê duyệt của chuyên gia. |
| Có side effect | Tạo giao dịch, thay đổi dữ liệu, thực thi lệnh. | Tool/API có xác thực, idempotency, approval; RAG không tự ra lệnh. |

**Bài toán giải quyết:** tránh rollout một chatbot “nói hay” nhưng không truy vết được, không đo được và không chịu trách nhiệm được khi trả lời sai.

---

## B. 8 câu hỏi phổ biến mở rộng

### 6. Dùng RAG cho trợ lý tri thức nội bộ, HR, IT và onboarding như thế nào?

**Trả lời:** Đây thường là use case có ROI sớm nhất: nhân viên hỏi nhiều về quy trình, quyền lợi, setup máy, công cụ, quy định nghỉ phép, SOP, biểu mẫu và cách liên hệ đúng bộ phận. RAG làm giảm thời gian tìm wiki/Drive và giảm câu hỏi lặp lại cho đội hỗ trợ.

**Áp dụng khi nào?** Knowledge nằm trong handbook, wiki, SOP, tài liệu đào tạo, ticket đã giải quyết và knowledge base có chủ sở hữu rõ ràng.

**Thiết kế nên có:** source catalog; `department`, `role`, `country`, `effectiveFrom`, `status`; ACL từ identity backend; citation đến section; luồng “tạo ticket/chuyển người phụ trách” khi câu hỏi cần hành động.

**Bài toán giải quyết:** onboarding chậm, tài liệu không được tìm thấy, câu trả lời phụ thuộc vào vài nhân sự kỳ cựu và SOP bị dùng nhầm phiên bản.

**So sánh:** intranet search chỉ tìm link; RAG có thể trả lời và hướng dẫn bước tiếp theo. Tuy nhiên, nếu chỉ cần vài form cố định, portal hoặc workflow rõ ràng sẽ đơn giản và kiểm soát hơn chatbot.

---

### 7. RAG hỗ trợ CSKH, contact center và self-service có gì khác chatbot FAQ cũ?

**Trả lời:** RAG cho phép nhân viên hoặc khách hàng hỏi tự nhiên về policy, hướng dẫn, catalogue và ngoại lệ mà không phải duy trì hàng nghìn intent thủ công. Nó phù hợp để **trả lời có căn cứ** và hỗ trợ agent làm việc nhanh hơn; không nên dùng để suy đoán trạng thái đơn hàng hoặc tự hứa quyền lợi ngoài policy.

**Áp dụng khi nào?** Bán lẻ, ngân hàng, bảo hiểm, viễn thông, SaaS, du lịch, dịch vụ có volume câu hỏi cao và knowledge base cập nhật thường xuyên.

**Pattern khuyến nghị:** retrieve policy đúng `product/region/effective date/customer tier`; gọi CRM/order API chỉ khi cần data khách hàng; hiển thị citation nội bộ cho agent; chuyển người phụ trách khi confidence/evidence thấp.

**Bài toán giải quyết:** giảm AHT (*Average Handle Time*), tăng độ nhất quán, giảm agent tra manual thủ công, hỗ trợ khách hàng tự phục vụ nhưng vẫn truy vết được policy.

**So sánh:** intent-based bot dự đoán một intent có sẵn và phù hợp flow hẹp; RAG linh hoạt hơn với câu hỏi dài/đa dạng nhưng cần retrieval, guardrail và đánh giá evidence kỹ hơn.

---

### 8. RAG cho developer, SRE, IT support và security operations nên thiết kế ra sao?

**Trả lời:** Đây là nhóm use case mạnh vì tri thức kỹ thuật thường nhiều version, có error code, config, command, runbook và lịch sử incident. Retrieval nên ưu tiên **hybrid search**: keyword/exact match cho mã lỗi và tên service, vector search cho mô tả tự nhiên.

**Áp dụng khi nào?** API documentation, code docs, architecture decision record, runbook, release note, postmortem, ticket triage, cloud operations, SOC/NOC knowledge.

**Thiết kế nên có:** filter theo `service`, `environment`, `version`, `language`, `region`; giữ code block và command không bị cắt; citation theo repository/file/commit hoặc runbook section; cảnh báo rõ command có tính phá hủy; không tự thực thi production command nếu chưa qua policy/approval.

**Bài toán giải quyết:** MTTR cao do tìm sai runbook, đội hỗ trợ trả lời theo version cũ, knowledge nằm trong ticket/chat, và dev mới khó hiểu hệ thống.

**So sánh:** code search tốt cho truy vấn chính xác trong source; RAG tốt để kết hợp runbook, doc, postmortem và diễn giải. Một copilot tốt thường dùng cả code search, RAG và tool/API đọc trạng thái quan sát được.

---

### 9. RAG cho sales, e-commerce, marketing, travel và hospitality áp dụng vào đâu?

**Trả lời:** Nhóm này dùng RAG để tư vấn sản phẩm/dịch vụ từ catalogue lớn, so sánh thông số, giải thích điều kiện khuyến mãi, chuẩn bị sales enablement và trả lời câu hỏi trước mua. Khi câu trả lời có giá, tồn kho, lịch hoặc availability, phải gọi API live.

**Áp dụng khi nào?** Catalogue nhiều thuộc tính, tài liệu product marketing, FAQ, terms, use case theo ngành, hướng dẫn dịch vụ và nội dung đa ngôn ngữ.

**Pattern khuyến nghị:** RAG lấy evidence từ product spec/policy; metadata filter theo market, product line, campaign, language; API lấy price, stock, booking availability; rule engine kiểm promotion eligibility; LLM tổng hợp thành câu trả lời dễ hiểu.

**Bài toán giải quyết:** nhân viên sales khó tìm đúng collateral, khách hàng không hiểu khác biệt sản phẩm, mô tả thiếu nhất quán giữa kênh, và đội CS phải xử lý nhiều câu hỏi trước mua.

**So sánh:** recommender system tối ưu xếp hạng sản phẩm theo hành vi; RAG tối ưu lời giải thích có bằng chứng. Có thể dùng recommender để chọn candidate product và RAG để giải thích “vì sao phù hợp”.

---

### 10. RAG dùng cho legal, compliance, finance, insurance, procurement và dịch vụ công có an toàn không?

**Trả lời:** Có thể hữu ích mạnh cho **tra cứu, tóm tắt, checklist, so sánh điều khoản, chuẩn bị hồ sơ và tìm evidence**, nhưng không nên tự đưa ra kết luận ràng buộc, quyết định quyền lợi, tư vấn pháp lý/tài chính cá nhân hay phê duyệt hồ sơ mà không có review phù hợp.

**Áp dụng khi nào?** Policy dày, luật/quy định/contract có cấu trúc, nhiều phiên bản và yêu cầu audit; người dùng cần tìm clause, điều kiện, nghĩa vụ hoặc evidentiary trail.

**Thiết kế bắt buộc:** document version immutable, effective dates, jurisdiction/product/role filter, section-level citation, cảnh báo giới hạn, nguồn được phê duyệt, audit trace và human-in-the-loop cho quyết định.

**Bài toán giải quyết:** giảm thời gian đọc lượng lớn văn bản, phát hiện section liên quan, chuẩn hóa checklist, hỗ trợ chuẩn bị thông tin cho chuyên gia.

**So sánh:** document management/search giúp lưu và tìm file; RAG giúp tổng hợp và hỏi–đáp. Rule engine mới là nơi mã hóa điều kiện quyết định xác định; con người/chuyên gia vẫn chịu trách nhiệm cho kết luận thuộc phạm vi chuyên môn.

---

### 11. RAG có thể dùng trong y tế, khoa học sự sống và giáo dục như thế nào?

**Trả lời:** Trong y tế và life sciences, RAG phù hợp hơn với hỗ trợ tìm guideline, protocol, SOP, tài liệu thuốc đã phê duyệt, tóm tắt evidence và chuẩn bị thông tin cho chuyên gia — không phải thay thế chẩn đoán hoặc quyết định điều trị. Trong giáo dục, RAG có thể tạo tutor bám giáo trình, giải thích khái niệm, dẫn chứng học liệu và hỗ trợ giáo viên chuẩn bị nội dung.

**Áp dụng khi nào?** Corpus được curate, có chủ sở hữu, version rõ; yêu cầu citation mạnh; users hiểu giới hạn của trợ lý.

**Bài toán giải quyết:** tìm guideline chậm, knowledge phân tán, khó truy ngược nguồn, người học cần lời giải thích theo tài liệu chính thức thay vì câu trả lời chung chung.

**Kiểm soát quan trọng:** cập nhật theo phiên bản, giao diện nêu nguồn và hạn chế, escalation đến chuyên gia, đánh giá câu hỏi bẫy; bảo vệ dữ liệu cá nhân và tách phiếu/record bệnh nhân khỏi corpus tri thức nếu không có quyền xử lý phù hợp.

**So sánh:** knowledge tutor có thể dùng long context cho một chương ngắn; course RAG hữu ích khi giáo trình, bài giảng, rubric và FAQ trải rộng. Clinical decision support cần thêm dữ liệu có cấu trúc, rule và quy trình chuyên môn — RAG chỉ cung cấp lớp evidence/giải thích.

---

### 12. RAG cho sản xuất, logistics, field service và vận hành vật lý khác gì tài liệu kỹ thuật thông thường?

**Trả lời:** Trong vận hành vật lý, đáp án thường phụ thuộc vào model thiết bị, serial/range, firmware, vị trí, hướng dẫn an toàn, biểu đồ hoặc bảng. Vì vậy cần **technical/multimodal RAG** với metadata tài sản và kiểm soát nghiêm ngặt hơn cho hướng dẫn có rủi ro.

**Áp dụng khi nào?** Manual thiết bị, maintenance record, troubleshooting guide, SOP an toàn, inventory procedure, shipping policy, inspection checklist, ảnh/sơ đồ/scan.

**Pattern khuyến nghị:** parse heading, bảng, ảnh chú thích; index theo `assetModel`, `serialRange`, `firmware`, `site`, `safetyClass`; retrieve parent section để không mất cảnh báo; mở link đến trang/sơ đồ gốc; yêu cầu xác nhận hoặc chuyển kỹ sư khi thao tác vượt thẩm quyền.

**Bài toán giải quyết:** technician dùng nhầm manual, thiếu bước an toàn, mất thời gian tìm sơ đồ, tri thức từ sự cố trước không được tái sử dụng.

**So sánh:** CMMS/EAM quản lý tài sản, lịch bảo trì và work order; RAG hỗ trợ tìm đúng hướng dẫn và giải thích. Không để RAG thay CMMS hoặc bypass safety workflow.

---

### 13. RAG cho nghiên cứu, phân tích, media, nội dung và enterprise search mang lại gì?

**Trả lời:** RAG có thể biến kho tài liệu thành trợ lý nghiên cứu: tìm các đoạn liên quan, tổng hợp nhiều nguồn, so sánh quan điểm, trích dẫn evidence và tạo brief. Nó cũng phù hợp với newsroom, market intelligence, consulting, due diligence và knowledge discovery trong doanh nghiệp.

**Áp dụng khi nào?** Có nhiều báo cáo, paper, transcript, CMS archive, note, deck, tài liệu thị trường hoặc corpus đa ngôn ngữ; người dùng cần câu trả lời có thể mở lại nguồn.

**Thiết kế nên có:** multi-document retrieval, dedup/near-duplicate handling, source diversity, citation per claim, date filter, separation giữa “trích từ nguồn” và “suy luận/tổng hợp”, cùng UI cho phép mở evidence đầy đủ.

**Bài toán giải quyết:** researcher mất thời gian đọc và lọc corpus; insight không truy được nguồn; việc tái sử dụng nội dung cũ và fact-check nội bộ chậm.

**So sánh:** BI/warehouse trả lời số liệu có cấu trúc; RAG trả lời lý do, bối cảnh và evidence văn bản. Khi cần cả hai, truy vấn dữ liệu có cấu trúc qua SQL/API rồi dùng RAG để bổ sung diễn giải/tài liệu liên quan.

---

### 14. Khi nào cần multimodal RAG, Graph RAG hoặc agentic RAG?

**Trả lời:** Chỉ nâng cấp khi loại evidence hoặc đường đi truy vấn buộc phải nâng cấp.

- **Multimodal RAG** cần khi bảng, biểu đồ, sơ đồ, ảnh, scan hoặc layout PDF chứa phần đáp án; không chỉ embed text OCR mù quáng.
- **Graph RAG** cần khi quan hệ giữa entity là phần cốt lõi của câu hỏi, như dependency hệ thống, cấu trúc tổ chức, supply chain hoặc quan hệ giữa quy định và control.
- **Agentic RAG** cần khi hệ thống phải chọn giữa nhiều nguồn/tool và phân rã câu hỏi nhiều bước; chỉ dùng với loop limit, tool allow-list, time/cost budget và approval.

**Áp dụng khi nào?** Multimodal cho technical manual/contract scan/report; Graph RAG cho query multi-hop; agentic RAG cho nghiên cứu hoặc trợ lý workflow biến thiên.

**Bài toán giải quyết:** tránh mất evidence trong ảnh/bảng, giảm failure khi cần reasoning qua quan hệ, và tự động hóa chọn tool ở phạm vi kiểm soát.

**So sánh:** RAG văn bản chuẩn thường đủ cho FAQ, policy và docs có cấu trúc. Graph/agent không tự tăng correctness nếu corpus, metadata và retrieval cơ bản vẫn sai; chúng chỉ thêm khả năng và cả chi phí vận hành.

---

## Ma trận chọn use case và lộ trình khởi động

| Bước | Câu hỏi quyết định | Kết quả cần có |
| --- | --- | --- |
| 1. Chọn pain point | Ai đang mất thời gian tìm gì, và lỗi hiện tại gây thiệt hại gì? | Use case hẹp có owner và KPI. |
| 2. Xác định source of truth | Câu trả lời đến từ document, API, database hay rule? | Phân ranh RAG với tool/API rõ ràng. |
| 3. Kiểm kê dữ liệu | Source có cập nhật, version, metadata, ACL và quyền sử dụng không? | Data contract + ingestion plan. |
| 4. Tạo eval set | Câu hỏi thật, câu điều kiện, câu ngoài phạm vi, câu không quyền. | Dataset đánh giá có đáp án/evidence chuẩn. |
| 5. Xây baseline | Parse → chunk → hybrid/metadata retrieve → answer + citation. | Demo có trace, abstention và feedback. |
| 6. Đo và cải thiện | Recall, groundedness, citation correctness, latency, cost, business KPI. | Quyết định thêm rerank/tool/agent dựa trên số liệu. |
| 7. Rollout an toàn | Scope nhỏ, human fallback, monitoring, rollback. | Production readiness theo mức rủi ro. |

## Ví dụ phân luồng câu hỏi trong một sản phẩm

```text
Question
  ├─ Cần trạng thái/số liệu realtime? → Authenticated API / database / rule engine
  │                                      └─ RAG giải thích policy và thuật ngữ nếu cần
  ├─ Cần knowledge từ tài liệu?       → ACL + version filter → hybrid retrieval → citation
  ├─ Cần hành động?                   → Workflow/tool có approval và audit
  └─ Không đủ evidence?               → Abstain, hỏi làm rõ hoặc chuyển người phụ trách
```

## Liên kết liên quan

- [RAG](../README.md)
- [Chunking](../chunking/README.md)
- [Embedding](../embedding/README.md)
- [Retrieval](../retrieval/README.md)
- [Chỉ mục pgvector](../pgvector-index/README.md)
- [LLM](../../README.md)
- [AI](../../../README.md)
- [Knowledge Tree](../../../../README.md)
