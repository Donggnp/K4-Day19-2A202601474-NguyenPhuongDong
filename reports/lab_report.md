# Báo cáo Lab 19 — Production GraphRAG vs Flat RAG

**Học viên:** Nguyen Phuong Dong
**Notebook:** `Day19_GraphRAG_vs_FlatRAG_Production_Lab_Guide.ipynb`
**Ngày chạy:** 20/08/2026

## 1. Mục tiêu

Lab triển khai và so sánh hai kiến trúc truy hồi thông tin:

- **Flat RAG:** truy hồi các chunks tương đồng bằng FAISS.
- **Hybrid GraphRAG:** trích xuất entity/relation, nạp vào Neo4j, duyệt graph và kết hợp graph context với vector context.

Pipeline còn kiểm tra provenance của cạnh, entity resolution, super-node policy và benchmark trên Golden Dataset gồm các nhóm `factoid`, `multi-hop` và `cross-doc`.

## 2. Cấu hình và dữ liệu

Các secret cần thiết đã được nhận diện trong môi trường Kaggle:

- `HF_TOKEN`: đã cấu hình.
- `GROQ_API_KEY`: đã cấu hình.
- `NEO4J_URI`, `NEO4J_PASSWORD`: đã cấu hình.
- `OPENAI_API_KEY`: không cấu hình; pipeline sử dụng Groq.

Dataset HackerNoon đã tồn tại cục bộ với dung lượng **58.35 MB** và được sử dụng lại, không tải lại.

Trong lần chạy thực tế mới nhất, cấu hình được đặt ở mức trung gian để giảm thời gian:

```python
SMOKE_TEST = False
LAB_MAX_ARTICLES = 500
LAB_MAX_CHUNKS = 800
EXTRACTION_MAX_CHUNKS = 50
CHUNK_WORDS = 220
```

Cấu hình full theo yêu cầu của lab là:

```python
SMOKE_TEST = False
LAB_MAX_ARTICLES = 1500
LAB_MAX_CHUNKS = 3000
EXTRACTION_MAX_CHUNKS = 400
CHUNK_WORDS = 220
```

## 3. Kết quả thực tế của lần chạy mới nhất

### 3.1. Tiền xử lý và index

| Hạng mục | Kết quả thực tế |
|---|---:|
| Dataset đầu vào | 100,000 dòng, 58.35 MB |
| Sau exact dedup | 42,887 bài viết |
| Bài viết được xử lý | 500 |
| Số chunks | 503 |
| Coreference batches | 5 |
| FAISS vectors | 503 |
| Neo4j connection | Thành công |

### 3.2. NER/RE và Knowledge Graph

Lần chạy mới đã khắc phục lỗi model Groq và sinh được dữ liệu thật:

```text
✅ Extracted triples: 23
✅ Canonicalized triples: 23
✅ Nodes bulk inserted via UNWIND.
✅ Edges bulk inserted via UNWIND.
```

Kết quả graph:

```text
Graph Statistics:
{'nodes': 39, 'edges': 23, 'invalid_provenance_edges': 0}
Invalid provenance edges: 0
```

Như vậy, graph đã có dữ liệu và toàn bộ 23 edges hiện có đều chứa provenance bắt buộc. Node có degree cao nhất là `VIQ Solutions` với degree bằng 3; super-node cap chưa được kích hoạt vì degree chưa vượt ngưỡng 100.

Entity-resolution audit hiện vẫn không có dòng (`No audit rows`). Đây là điểm còn thiếu so với rubric yêu cầu audit tối thiểu 10 dòng.

### 3.3. Benchmark thực tế

Golden Dataset hợp lệ với 5 câu hỏi. Kết quả benchmark mới nhất:

| Nhóm | Metric | Flat RAG | GraphRAG |
|---|---|---:|---:|
| cross-doc | Comprehensiveness | 1.000 | 1.000 |
| cross-doc | Faithfulness | 5.000 | 3.000 |
| cross-doc | Multi-hop reasoning | 3.000 | 1.000 |
| cross-doc | Latency (s) | 3.835 | 3.783 |
| cross-doc | Token usage | 923.5 | 840.0 |
| factoid | Comprehensiveness | 1.000 | 1.000 |
| factoid | Faithfulness | 1.000 | 1.000 |
| factoid | Multi-hop reasoning | 1.000 | 1.000 |
| factoid | Latency (s) | 0.506 | 0.457 |
| factoid | Token usage | 828.0 | 609.0 |
| multi-hop | Comprehensiveness | 3.000 | 1.000 |
| multi-hop | Faithfulness | 4.500 | 1.000 |
| multi-hop | Multi-hop reasoning | 3.500 | 1.000 |
| multi-hop | Latency (s) | 2.890 | 6.314 |
| multi-hop | Token usage | 948.5 | 750.0 |

GraphRAG hiện chưa vượt Flat RAG ở chất lượng. Nguyên nhân hợp lý là graph mới chỉ có 23 cạnh, chưa đủ dày để trả lời tốt câu hỏi multi-hop. GraphRAG cũng chậm hơn Flat RAG ở nhóm multi-hop do phải thực hiện seed matching và traversal.

Hai file kết quả đã được export:

- `outputs/graphrag_eval_results.csv`
- `outputs/graphrag_vs_flatrag_summary.csv`

## 4. Dự phóng theo full configuration

Phần này là **ước tính theo mật độ quan sát được**, không phải số liệu đã chạy full và không được trình bày thay cho benchmark thực tế.

Từ lần chạy hiện tại:

```text
23 triples / 50 extraction chunks = 0.46 triples/chunk
39 nodes / 23 triples ≈ 1.70 nodes/triple
```

Nếu chạy với `EXTRACTION_MAX_CHUNKS = 400`, phép ngoại suy tuyến tính cho quy mô dữ liệu là:

| Chỉ số | Thực tế hiện tại | Dự phóng full | Ghi chú |
|---|---:|---:|---|
| Articles | 500 | 1,500 | Nhân theo scale 3x |
| Chunks | 503 | khoảng 1,510–3,000 | Phụ thuộc số chunks thực tế và cap |
| Extraction chunks | 50 | 400 | Cấu hình full |
| Triples | 23 | khoảng 184 | Ước tính `23 × 400 / 50` |
| Nodes | 39 | khoảng 312 | Ước tính theo node/triple hiện tại |
| Edges | 23 | khoảng 184 | Gần bằng số triples hợp lệ |
| Coreference batches | 5 | khoảng 40 | Nếu giữ batch size 10 |
| Invalid provenance edges | 0 | mục tiêu 0 | Phải kiểm tra lại bằng Neo4j |

Các điểm chất lượng, latency và token usage **không được nhân tuyến tính**. Chúng phải được chạy lại từ output full vì phụ thuộc vào nội dung chunks, cache, rate limit, số seed match và mật độ graph.

## 5. Phân tích kỹ thuật

### 5.1. Flat RAG

Flat RAG đã xây dựng FAISS index với 503 vectors. Đây là baseline đơn giản và có latency thấp ở các câu hỏi factoid. Tuy nhiên, vector search không đảm bảo nối được các quan hệ nằm ở nhiều tài liệu khác nhau.

### 5.2. GraphRAG

Notebook đã triển khai:

- allowlist node type `Company`, `Person`, `Technology`;
- allowlist relation type;
- entity resolution bằng alias, FAISS ANN, lexical guard và Union-Find;
- Neo4j bulk insert bằng `UNWIND`;
- BFS traversal tối đa hai hops;
- super-node cap ở degree lớn hơn 100;
- global edge cap 250 và graph context cap 14,000 ký tự;
- provenance gồm `source_chunk_id`, `published_date`, `evidence`, `confidence`.

Lần chạy hiện tại chứng minh được ingestion và provenance hoạt động, nhưng graph còn nhỏ nên chưa thể đánh giá ưu thế GraphRAG một cách thuyết phục.

## 6. Failure modes và biện pháp kiểm soát

### Failure mode 1 — Model Groq bị deprecate

Model cũ `llama-3.3-70b-versatile` trả về lỗi 404. Notebook đã đổi sang model mới và có fallback model. Sau khi sửa, cell NER/RE đã tạo được 23 triples.

### Failure mode 2 — Graph thưa

23 edges chưa đủ để bao phủ các câu hỏi multi-hop. Vì vậy GraphRAG đạt điểm thấp hơn Flat RAG ở nhóm multi-hop. Biện pháp là tăng extraction chunks lên 400 và kiểm tra entity-resolution audit.

### Failure mode 3 — Audit entity resolution rỗng

`No audit rows` cho thấy lần chạy này chưa tạo đủ cặp entity để đánh giá merge/reject. Cần tăng số triples hoặc bổ sung test cases có alias gần nhau như `Microsoft Corporation`/`Microsoft` và các cặp dễ false merge.

### Failure mode 4 — Super-node

Node cao nhất chỉ có degree 3 nên chưa kích hoạt cap. Khi chạy full, policy degree > 100 và giới hạn 50 edges mới nhất sẽ giúp tránh context explosion, nhưng có thể bỏ sót quan hệ lịch sử.

## 7. Reflection cá nhân

Bài lab cho thấy cần phân biệt runtime success, data success và evaluation success. Lần chạy mới đã đạt runtime success và data success cơ bản: graph có node, edge và provenance hợp lệ. Tuy nhiên, evaluation success ở mức production chưa hoàn tất vì graph còn nhỏ và audit entity resolution rỗng.

Trong đồ án thực tế, tôi sẽ dùng Flat RAG cho factoid, GraphRAG cho câu hỏi cần nối quan hệ, và hybrid retrieval cho câu hỏi cross-document. Tôi cũng sẽ kiểm tra dữ liệu trung gian sau từng module thay vì chỉ dựa vào việc notebook chạy hết cell.

## 8. Kết luận

Lần chạy mới đã sửa được lỗi Groq và tạo thành công Knowledge Graph với **39 nodes, 23 edges và 0 edge thiếu provenance**. Pipeline, Neo4j ingestion, FAISS và benchmark đều chạy được.

Tuy nhiên, số liệu full configuration trong mục 4 chỉ là dự phóng, chưa phải kết quả thực nghiệm. Để có báo cáo full chính thức, cần chạy lại với:

```python
SMOKE_TEST = False
LAB_MAX_ARTICLES = 1500
LAB_MAX_CHUNKS = 3000
EXTRACTION_MAX_CHUNKS = 400
CHUNK_WORDS = 220
```

Sau lần chạy đó, cần cập nhật lại bảng benchmark, số nodes/edges và entity-resolution audit bằng output thực tế.
