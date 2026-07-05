# Chỉ mục pgvector (pgvector Index)

> [Knowledge Tree](../../../../README.md) / [AI](../../../README.md) / [LLM](../../README.md) / [RAG](../README.md) / **Chỉ mục pgvector**

**pgvector** là extension PostgreSQL để lưu và tìm kiếm vector embedding. **Chỉ mục pgvector (pgvector index)** là cấu trúc tăng tốc truy vấn *nearest-neighbor search* — tìm các vector gần nhất với vector câu hỏi — mà vẫn giữ dữ liệu, giao dịch, `JOIN`, phân quyền và backup trong PostgreSQL.

> **Embedding → distance operator → `ORDER BY ... LIMIT k` → top-k chunks**

Mặc định, pgvector có thể tìm kiếm **chính xác** (*exact nearest-neighbor search*), cho *recall* hoàn hảo nhưng thường chậm dần khi số vector tăng. Hai chỉ mục gần đúng (*approximate nearest-neighbor*, ANN) là **HNSW** và **IVFFlat** đổi một phần recall lấy độ trễ thấp hơn.

## Điều hướng

- **Node cha:** [RAG](../README.md)
- **Node con:** Chưa có.
- **Khái niệm trước trong cùng nhánh:** [Retrieval](../retrieval/README.md)
- **Khái niệm sau trong cùng nhánh:** Chưa có.

## A. 5 câu hỏi cốt lõi

### 1. Chỉ mục pgvector là gì, và có bắt buộc cho RAG không?

**Trả lời:** Chỉ mục pgvector là index PostgreSQL dành cho truy vấn tìm vector gần nhất. Nó không tạo embedding và cũng không quyết định chunk nào đúng về nghiệp vụ; nó chỉ giảm thời gian tìm ứng viên gần với query embedding. Hai loại ANN chính là `hnsw` và `ivfflat`.

Không phải RAG nào cũng cần ANN index. Với dữ liệu nhỏ, exact search không index thường đơn giản hơn, có kết quả đầy đủ và là baseline tốt để đánh giá chất lượng. Khi số chunk, tần suất truy vấn hoặc yêu cầu latency tăng, ANN index giúp giữ thời gian retrieval ổn định hơn.

| Cách tìm | Độ chính xác retrieval | Tốc độ khi dữ liệu lớn | Khi nên dùng |
| --- | --- | --- | --- |
| Exact search, không ANN index | Recall hoàn hảo trong tập dữ liệu đã filter | Có thể chậm do phải so sánh nhiều vector | Prototype, dữ liệu nhỏ, benchmark/evaluation, truy vấn hiếm |
| HNSW | Gần đúng, thường có trade-off speed–recall tốt | Nhanh, nhưng build/index lớn hơn | RAG production cần latency thấp và có đủ RAM/storage |
| IVFFlat | Gần đúng, phụ thuộc `lists` và `probes` | Build nhanh hơn, tốn ít bộ nhớ hơn HNSW | Dữ liệu đã có sẵn, cần giảm tài nguyên build/index |

**Áp dụng khi nào?** Dùng pgvector index khi ứng dụng đã có PostgreSQL chứa document, tenant, quyền, version và metadata; muốn thêm semantic retrieval mà không vận hành ngay một vector database riêng.

**Bài toán giải quyết:** Giảm độ trễ truy xuất top-k chunk, giữ retrieval gần với dữ liệu quan hệ để có thể `JOIN`, transaction, backup, replica và filter theo tenant/ACL.

**So sánh với cách tương tự:** pgvector không thay thế parser, chunking, reranking hay full-text search. Nó là một lớp ANN trong pipeline RAG; keyword/BM25 vẫn cần thiết khi câu hỏi có SKU, error code, tên riêng hoặc điều khoản cần exact match.

---

### 2. HNSW và IVFFlat khác nhau thế nào? Chọn loại nào trước?

**Trả lời:** **HNSW (Hierarchical Navigable Small World)** tạo đồ thị nhiều tầng để đi nhanh đến các vector lân cận. Nó thường cho trade-off tốc độ–recall tốt hơn IVFFlat, nhưng build chậm hơn và dùng nhiều memory/storage hơn. **IVFFlat (Inverted File with Flat compression)** chia vector vào các list/cluster, rồi chỉ tìm trong một phần list gần query; build nhanh hơn và nhẹ hơn nhưng nhạy với cách chọn `lists`/`probes`.

| Tiêu chí | HNSW | IVFFlat |
| --- | --- | --- |
| Cấu trúc | Đồ thị nhiều tầng | Cụm/list vector |
| Query speed–recall | Thường tốt hơn | Thường kém hơn HNSW ở cùng mức tài nguyên |
| Build | Chậm hơn, dùng nhiều memory hơn | Nhanh hơn, nhẹ hơn |
| Dữ liệu trước khi tạo index | Có thể tạo trên table rỗng | Nên tạo sau khi có dữ liệu đại diện để train list |
| Tham số query chính | `hnsw.ef_search` | `ivfflat.probes` |
| Điểm khởi đầu thực tế | Lựa chọn mặc định cho retrieval chất lượng cao | Lựa chọn khi build/resource là ràng buộc chính |

**Áp dụng khi nào?** Bắt đầu với HNSW nếu retrieval là đường nóng (*hot path*) của chatbot và bạn có thể chấp nhận chi phí build/index. Cân nhắc IVFFlat khi bulk-load lớn, memory hạn chế hoặc cần build nhanh hơn.

**Bài toán giải quyết:** HNSW ưu tiên chất lượng top-k và latency; IVFFlat ưu tiên chi phí tạo/chứa index. Không có ngưỡng số lượng vector cố định áp dụng cho mọi hệ thống — hãy benchmark trên đúng dimension, filter, top-k và phần cứng production.

**So sánh với B-tree:** B-tree tốt cho equality/range như `tenant_id`, `document_version`, `status`; nó không thay HNSW/IVFFlat cho khoảng cách vector. Trong RAG đa tenant thường cần cả B-tree cho filter và pgvector index cho ANN.

---

### 3. Chọn distance metric và operator class như thế nào?

**Trả lời:** Metric phải nhất quán giữa cách embedding model được thiết kế, phép đo bạn dùng để đánh giá và operator class của index. Nếu query dùng cosine distance nhưng index tạo bằng L2 operator class, planner không dùng index đó cho truy vấn cosine.

| Ý nghĩa cần đo | SQL operator | Operator class ví dụ | Ghi chú |
| --- | --- | --- | --- |
| Euclidean / L2 distance | `<->` | `vector_l2_ops` | Phù hợp khi model/quy trình đánh giá dùng khoảng cách Euclidean |
| Negative inner product | `<#>` | `vector_ip_ops` | `<#>` trả về **negative** inner product để PostgreSQL scan theo thứ tự tăng dần |
| Cosine distance | `<=>` | `vector_cosine_ops` | Cosine similarity = `1 - cosine distance` |

Ví dụ tạo HNSW index cho cosine distance:

```sql
CREATE INDEX CONCURRENTLY rag_chunks_embedding_hnsw_cosine_idx
ON rag_chunks USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

**Áp dụng khi nào?** Chọn metric theo tài liệu của embedding model và bộ câu hỏi đánh giá thật. Với các model được normalize, inner product và cosine thường cho thứ hạng tương đương về mặt toán học; nhưng SQL operator, index và benchmark vẫn phải thống nhất.

**Bài toán giải quyết:** Tránh lỗi “có index nhưng query không dùng”, hoặc tệ hơn là top-k thay đổi vì metric không khớp với cách model được đánh giá.

**So sánh với similarity score:** pgvector ưu tiên trả về **distance** trong operator KNN; distance càng nhỏ càng gần. Không nên đổi biểu thức trong `ORDER BY` thành `1 - distance DESC` khi cần index, vì planner cần dạng distance operator tăng dần trực tiếp.

---

### 4. Schema và truy vấn pgvector tối thiểu cho RAG nên viết thế nào?

**Trả lời:** Lưu text chunk, vector, dữ liệu định danh và metadata/ACL trong cùng row hoặc trong các bảng quan hệ liên kết. Kích thước `vector(n)` phải bằng dimension của embedding model đang dùng.

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE rag_chunks (
  id           bigserial PRIMARY KEY,
  tenant_id    uuid NOT NULL,
  document_id  uuid NOT NULL,
  content      text NOT NULL,
  metadata     jsonb NOT NULL DEFAULT '{}'::jsonb,
  embedding    vector(1536) NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX CONCURRENTLY rag_chunks_tenant_id_idx
  ON rag_chunks (tenant_id);

CREATE INDEX CONCURRENTLY rag_chunks_embedding_hnsw_cosine_idx
  ON rag_chunks USING hnsw (embedding vector_cosine_ops);
```

Truy vấn top-k cần có `ORDER BY` trực tiếp trên distance operator và `LIMIT`:

```sql
SELECT id, document_id, content,
       embedding <=> $2::vector AS cosine_distance
FROM rag_chunks
WHERE tenant_id = $1
  AND metadata @> '{"status":"active"}'::jsonb
ORDER BY embedding <=> $2::vector
LIMIT 8;
```

**Áp dụng khi nào?** Mẫu này phù hợp với chatbot tài liệu có tenant, phiên bản, trạng thái hiệu lực hoặc quyền truy cập. Có thể thay `metadata` bằng các cột quan hệ rõ ràng hơn cho filter nóng như `product_id`, `locale`, `effective_from`, `effective_to`.

**Bài toán giải quyết:** Đưa source text và metadata về cùng chuỗi truy xuất để có thể trả citation, filter đúng phạm vi và tránh phải đồng bộ hai kho dữ liệu ngay từ đầu.

**So sánh với chỉ lưu vector:** Chỉ lưu `id + embedding` khiến retrieval khó filter, audit và hiển thị citation. Lưu text/metadata cùng PostgreSQL giúp đơn giản pipeline; với document rất lớn, file gốc vẫn nên ở object storage và DB giữ reference/page/section.

---

### 5. Filter metadata và ACL hoạt động thế nào với HNSW/IVFFlat?

**Trả lời:** Điều kiện tenant, quyền và hiệu lực tài liệu phải ở SQL `WHERE` ngay khi retrieval; không được chờ đến sau khi context đã vào prompt. Với ANN index, pgvector có thể scan ứng viên vector trước rồi mới áp filter, nên query có filter chọn lọc cao có thể trả ít hơn `k` kết quả nếu tham số search quá thấp.

Ví dụ HNSW cho một query, dùng `SET LOCAL` để không làm thay đổi session dùng chung:

```sql
BEGIN;
SET LOCAL hnsw.ef_search = 100;
SET LOCAL hnsw.iterative_scan = strict_order;

SELECT id, content
FROM rag_chunks
WHERE tenant_id = $1
  AND metadata @> '{"status":"active"}'::jsonb
ORDER BY embedding <=> $2::vector
LIMIT 8;
COMMIT;
```

**Áp dụng khi nào?** Luôn áp dụng cho RAG multi-tenant, dữ liệu nội bộ, tài liệu có phiên bản, quy định theo quốc gia hoặc phân quyền theo role/team.

**Bài toán giải quyết:** Ngăn cross-tenant data leak, tránh trả tài liệu hết hiệu lực và tăng xác suất lấy đủ top-k sau filter.

**So sánh các chiến lược filter:**

| Tình huống filter | Hướng ưu tiên |
| --- | --- |
| Filter rất chọn lọc, ít row khớp | B-tree/composite index trên filter có thể đủ cho exact vector search trong tập nhỏ |
| Ít giá trị filter cố định | Cân nhắc partial HNSW/IVFFlat index |
| Rất nhiều tenant/phân vùng dữ liệu | Cân nhắc partitioning theo ranh giới dữ liệu phù hợp |
| ANN + filter vẫn trả ít kết quả | Tăng `ef_search`/`probes` có kiểm soát hoặc dùng iterative scan nếu phiên bản pgvector hỗ trợ |

**Lưu ý bảo mật:** ACL là quyết định truy cập, không phải metadata trang trí. Có thể kết hợp filter bắt buộc từ service layer với Row Level Security (RLS) của PostgreSQL; tuyệt đối không tin `tenant_id` hay `allowedRoles` do client gửi trực tiếp.

## B. 8 câu hỏi phổ biến mở rộng

### 6. Nên tạo index trước hay sau khi nạp dữ liệu ban đầu?

**Trả lời:** Với bulk ingestion, thường nạp dữ liệu ban đầu trước rồi tạo index, vì PostgreSQL/pgvector có thể build hiệu quả hơn so với duy trì index sau từng insert. IVFFlat đặc biệt nên được tạo sau khi table có dữ liệu đại diện, vì nó cần chia list/cluster. HNSW có thể tạo trên table rỗng, nhưng bulk-load rồi build vẫn thường thuận tiện hơn cho initial load.

```sql
-- 1. Nạp row + embedding bằng batch/COPY.
-- 2. Sau đó tạo index ngoài transaction để giảm chặn ghi.
CREATE INDEX CONCURRENTLY rag_chunks_embedding_hnsw_cosine_idx
ON rag_chunks USING hnsw (embedding vector_cosine_ops);
```

**Áp dụng khi nào?** Migration từ knowledge base cũ, import PDF lớn, re-embed toàn bộ khi đổi model, hoặc backfill nhiều tháng dữ liệu.

**Bài toán giải quyết:** Giảm thời gian ingest ban đầu, hạn chế lock ghi khi rollout production và tránh IVFFlat có list chất lượng kém vì tạo index khi table gần rỗng.

**So sánh với online writes:** Sau khi index tồn tại, `INSERT`, `UPDATE`, `DELETE` bình thường vẫn được PostgreSQL duy trì. Nhưng bulk backfill cực lớn nên có kế hoạch riêng về window, disk, WAL, replica lag và rollback.

---

### 7. Tuning HNSW nên bắt đầu từ tham số nào?

**Trả lời:** Ba tham số quan trọng là `m`, `ef_construction` và `hnsw.ef_search`.

| Tham số | Ảnh hưởng chính | Trade-off |
| --- | --- | --- |
| `m` | Số kết nối tối đa mỗi layer | Tăng recall/khả năng tìm nhưng index lớn hơn, build tốn hơn |
| `ef_construction` | Số candidate khi build graph | Tăng chất lượng graph nhưng build chậm hơn và insert nặng hơn |
| `hnsw.ef_search` | Số candidate khi query | Tăng recall nhưng latency query cao hơn |

```sql
CREATE INDEX CONCURRENTLY rag_chunks_embedding_hnsw_cosine_idx
ON rag_chunks USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

BEGIN;
SET LOCAL hnsw.ef_search = 100;
SELECT id, content
FROM rag_chunks
ORDER BY embedding <=> $1::vector
LIMIT 8;
COMMIT;
```

**Áp dụng khi nào?** Bắt đầu bằng default hợp lý, chỉ tăng tham số sau khi đo `Recall@k`, p95/p99 latency, index size và thời gian build trên dữ liệu thật.

**Bài toán giải quyết:** Tách hai loại tuning: chất lượng graph khi build và mức nỗ lực khi query. Điều này cho phép một index phục vụ nhiều tier: query thường dùng `ef_search` thấp hơn, truy vấn quan trọng dùng cao hơn.

**So sánh với IVFFlat:** HNSW tuning thiên về graph quality và search breadth; IVFFlat thiên về số list và probes. Không sao chép giá trị tham số từ hệ thống khác vì dimension, filter selectivity và query distribution thay đổi kết quả đáng kể.

---

### 8. Tuning IVFFlat với `lists` và `probes` như thế nào?

**Trả lời:** `lists` được chọn khi tạo index; `probes` là số list cần search lúc query. Tăng `lists` có thể tăng tốc vì mỗi list nhỏ hơn, nhưng nếu giữ `probes` thấp thì recall giảm. Tăng `probes` cải thiện recall nhưng chậm hơn.

```sql
CREATE INDEX CONCURRENTLY rag_chunks_embedding_ivfflat_cosine_idx
ON rag_chunks USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

BEGIN;
SET LOCAL ivfflat.probes = 10;
SELECT id, content
FROM rag_chunks
ORDER BY embedding <=> $1::vector
LIMIT 8;
COMMIT;
```

Điểm xuất phát phổ biến từ tài liệu pgvector là `rows / 1000` list khi table đến khoảng một triệu row, `sqrt(rows)` khi lớn hơn; sau đó thử `sqrt(lists)` cho `probes`. Đây chỉ là điểm bắt đầu để benchmark, không phải công thức production cố định.

**Áp dụng khi nào?** Dùng khi chọn IVFFlat vì tốc độ build/memory, đã có corpus đủ đại diện và có thể kiểm tra recall sau mỗi thay đổi.

**Bài toán giải quyết:** Kiểm soát được điểm cân bằng giữa latency, index cost và kết quả top-k.

**So sánh với `probes = lists`:** Khi `probes` bằng số `lists`, IVFFlat tiến gần exact search và planner có thể không dùng ANN index. Điều đó phù hợp làm baseline nhất thời, nhưng không phải tối ưu latency.

---

### 9. Vì sao query không dùng pgvector index, và debug thế nào?

**Trả lời:** Dạng truy vấn KNN phải là `ORDER BY <distance operator> ASC LIMIT k`. Các biểu thức bọc distance, sắp xếp giảm dần hoặc thiếu `LIMIT` thường làm planner không thể dùng ANN index.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, content
FROM rag_chunks
WHERE tenant_id = $1
ORDER BY embedding <=> $2::vector
LIMIT 8;
```

Checklist debug:

1. Operator phải khớp operator class: `<=>` với `vector_cosine_ops`, `<->` với `vector_l2_ops`, `<#>` với `vector_ip_ops`.
2. `ORDER BY` phải dùng distance operator trực tiếp và theo thứ tự tăng dần.
3. Có `LIMIT` hợp lý cho KNN.
4. Query vector có đúng dimension, không phải `NULL`; với cosine, zero vector không được index.
5. Chạy `ANALYZE` sau bulk load và xem execution plan thay vì đoán.
6. Table nhỏ có thể đúng khi planner chọn sequential scan vì nó nhanh hơn index scan.

**Áp dụng khi nào?** Luôn dùng `EXPLAIN (ANALYZE, BUFFERS)` trước khi ép planner hoặc thay tham số server.

**Bài toán giải quyết:** Phân biệt lỗi SQL/index design với việc planner đang đưa ra lựa chọn tốt hơn cho corpus nhỏ; tránh “sửa” bằng `enable_seqscan = off` trong code production.

**So sánh với ORM abstraction:** ORM có thể tiện cho CRUD, nhưng truy vấn vector hot path nên giữ SQL/plan test rõ ràng. Repository method hoặc native query có integration test tốt thường đáng tin cậy hơn việc che toàn bộ metric/operator trong abstraction.

---

### 10. Đo recall, latency và chất lượng retrieval ra sao?

**Trả lời:** ANN index không được đánh giá chỉ bằng QPS hay p50 latency. Cần có tập query đại diện, kết quả expected hoặc baseline exact search, và các chỉ số `Recall@k`, `MRR`, `nDCG`, p50/p95/p99 latency, số chunk trả về sau filter, index size và cost build.

Một cách kiểm tra recall là so kết quả ANN với exact search trên cùng query/filter. Trong transaction benchmark, có thể tạm tắt index scan để lấy baseline exact theo hướng dẫn pgvector:

```sql
BEGIN;
SET LOCAL enable_indexscan = off;

SELECT id
FROM rag_chunks
WHERE tenant_id = $1
ORDER BY embedding <=> $2::vector
LIMIT 8;
COMMIT;
```

**Áp dụng khi nào?** Trước rollout index, sau đổi embedding model, chunking, metric, HNSW/IVFFlat parameter hoặc cách filter.

**Bài toán giải quyết:** Tránh tối ưu một con số latency nhưng làm mất chunk đúng; trong RAG, retrieval sai thường không được LLM hoặc reranker sửa hoàn toàn.

**So sánh với đánh giá answer-only:** Answer quality tốt không chứng minh retrieval tốt vì LLM có thể trả lời bằng kiến thức sẵn có hoặc suy đoán. Đo evidence-level (`Recall@k`, citation correctness) giúp tìm đúng nguyên nhân hơn.

---

### 11. INSERT/UPDATE/DELETE nhiều có ảnh hưởng thế nào? Khi nào cần REINDEX/VACUUM?

**Trả lời:** PostgreSQL duy trì index khi ghi dữ liệu, nên hệ thống có thể ingest incremental. Tuy nhiên, workload cập nhật/xóa lớn tạo dead tuples và bloat như các index PostgreSQL khác. Tài liệu pgvector lưu ý `VACUUM` với HNSW có thể mất thời gian; một chiến lược là reindex concurrently rồi vacuum khi dữ liệu/index đã churn nhiều.

```sql
REINDEX INDEX CONCURRENTLY rag_chunks_embedding_hnsw_cosine_idx;
VACUUM (ANALYZE) rag_chunks;
```

**Áp dụng khi nào?** Re-embedding hàng loạt, thay thế nhiều phiên bản tài liệu, retention job xóa chunk cũ hoặc khi index size/latency/recall suy giảm theo thời gian.

**Bài toán giải quyết:** Giữ index có thể vận hành lâu dài thay vì chỉ nhanh ở ngày đầu launch.

**So sánh với append-only corpus:** Nếu tài liệu chủ yếu thêm mới và ít xóa, bảo trì đơn giản hơn. Nếu dữ liệu biến động mạnh, cần quan sát autovacuum, disk/WAL, replica lag và lên lịch reindex như một hoạt động vận hành chính thức.

---

### 12. Đổi embedding model hoặc dimension thì migration index làm thế nào?

**Trả lời:** Không nên trộn embedding từ hai model/metric/dimension trong cùng một index mà vẫn kỳ vọng khoảng cách có ý nghĩa. Cách an toàn là version hóa embedding, backfill vector mới, tạo index mới, so sánh retrieval, rồi chuyển read traffic theo rollout có thể rollback.

Hai hướng dữ liệu phổ biến:

| Hướng | Khi phù hợp | Đánh đổi |
| --- | --- | --- |
| Cột/bảng mới theo model | Chuyển model theo đợt, cần rollback rõ | Schema nhiều hơn nhưng đơn giản cho query/index |
| Một cột `vector` không cố định dimension + `model_id` | Cần lưu nhiều model | Mỗi dimension/model cần expression + partial index riêng |

Ví dụ chỉ index một model có cùng dimension:

```sql
CREATE TABLE embeddings (
  model_id  bigint NOT NULL,
  item_id   bigint NOT NULL,
  embedding vector NOT NULL,
  PRIMARY KEY (model_id, item_id)
);

CREATE INDEX CONCURRENTLY embeddings_model_42_hnsw_idx
ON embeddings USING hnsw ((embedding::vector(1536)) vector_cosine_ops)
WHERE model_id = 42;
```

**Áp dụng khi nào?** Nâng model embedding, đổi dimension, thử nghiệm multilingual model hoặc cần A/B retrieval.

**Bài toán giải quyết:** Tránh kết quả gần về hình thức nhưng vô nghĩa vì các vector nằm trong không gian biểu diễn khác nhau; đồng thời cho phép rollout không downtime.

**So sánh với overwrite ngay:** Overwrite toàn bộ embedding khiến rollback khó và chất lượng giảm khó truy vết. Dual-write/backfill + shadow evaluation tốn storage tạm thời nhưng kiểm soát rủi ro tốt hơn.

---

### 13. Khi nào chọn pgvector thay vì Qdrant, OpenSearch/Elasticsearch, FAISS hoặc managed vector database?

**Trả lời:** Không có lựa chọn luôn tốt nhất; chọn theo nơi dữ liệu nguồn sống, mô hình filter/hybrid search, yêu cầu scale độc lập và năng lực vận hành của đội.

| Lựa chọn | Điểm mạnh chính | Hạn chế / khi không ưu tiên |
| --- | --- | --- |
| PostgreSQL + pgvector | SQL, transaction, `JOIN`, backup/replica, ACL và dữ liệu nghiệp vụ cùng một hệ | Vector workload rất lớn có thể cạnh tranh tài nguyên với OLTP |
| Vector database chuyên dụng, ví dụ Qdrant | Tách vector workload, payload filter và scale vector độc lập | Thêm hệ thống, đồng bộ dữ liệu, vận hành và consistency boundary |
| OpenSearch / Elasticsearch | Full-text/BM25, filter, analytics và hybrid search mạnh | Vận hành search cluster, không phải lựa chọn đơn giản nhất nếu đã có Postgres và use case thuần vector |
| FAISS | Thư viện ANN nhanh, linh hoạt cho local/offline | Không tự cung cấp persistence, transaction, ACL, multi-tenant API như database |
| Managed vector database | Giảm vận hành hạ tầng vector | Vendor cost, giới hạn tuỳ biến và dữ liệu/metadata phải đồng bộ sang service khác |

**Áp dụng khi nào?** Chọn pgvector trước khi nhu cầu chính là semantic search gần dữ liệu quan hệ, quy mô có thể benchmark được trên PostgreSQL hiện hữu và đội muốn tối thiểu thành phần hạ tầng. Tách vector store khi benchmark chỉ ra PostgreSQL không còn đáp ứng latency/throughput, hoặc vector workload cần scale độc lập khỏi OLTP.

**Bài toán giải quyết:** Ra quyết định theo trade-off thực tế thay vì theo trào lưu công cụ: đơn giản kiến trúc ban đầu nhưng vẫn có đường mở rộng khi workload đổi.

**So sánh với “hybrid search trong Postgres”:** PostgreSQL có thể kết hợp pgvector với full-text search ở mức SQL để làm hybrid theo nhu cầu cơ bản. Khi ranking lexical, faceting, analytics và search operations trở thành năng lực trung tâm, một search engine chuyên dụng thường thuận tiện hơn.

## Checklist production

- [ ] Xác nhận extension, PostgreSQL version và pgvector version được managed provider hỗ trợ.
- [ ] Chọn một embedding model, dimension, metric và operator class nhất quán.
- [ ] Có exact-search baseline và bộ query đánh giá theo use case thật.
- [ ] Index HNSW hoặc IVFFlat được benchmark với `Recall@k`, p95/p99 latency, index size và build time.
- [ ] Tất cả tenant/ACL/version/effective-date filter được áp trước hoặc trong SQL retrieval.
- [ ] Query hot path dùng `ORDER BY distance ASC LIMIT k`, được kiểm tra bằng `EXPLAIN (ANALYZE, BUFFERS)`.
- [ ] Chạy `ANALYZE` sau bulk load; có theo dõi `pg_stat_statements`, index size, autovacuum và replica lag.
- [ ] Có chiến lược backfill, dual-index/dual-read, rollback và reindex khi đổi embedding model hoặc dữ liệu churn cao.
- [ ] RAG vẫn có citation, threshold/abstention, hybrid search hoặc reranking khi bộ evaluation chứng minh cần thiết.

## Liên kết liên quan

- [RAG](../README.md)
- [Retrieval](../retrieval/README.md)
- [Embedding](../embedding/README.md)
- [Chunking](../chunking/README.md)
- [LLM](../../README.md)
- [AI](../../../README.md)
- [Knowledge Tree](../../../../README.md)

## Nguồn tham khảo

- [pgvector — README chính thức](https://github.com/pgvector/pgvector)
- [PostgreSQL — Index Types](https://www.postgresql.org/docs/current/indexes-types.html)
