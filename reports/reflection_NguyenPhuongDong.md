# Reflection và Action Plan

## Mapping bài giảng vào code

| Khái niệm | Module | Hàm/khối code | Bài học |
|---|---|---|---|
| Conservative coreference | M1 | `resolve_coref_batch()` | Mơ hồ thì giữ nguyên và log, không suy diễn. |
| Schema allowlist | M2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Reject sớm giảm edge bẩn. |
| Bulk Cypher | M2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | `UNWIND` giảm round-trip so với insert từng row. |
| Entity resolution | M3 | `build_resolution_map()`, `UF`, `merge_guard()` | ANN chỉ là candidate search; lexical guard mới là safety gate. |
| Super-node mitigation | M4 | `retrieve_graph_context()` | Cap theo degree, date và global context. |
| LLM Judge | M5 | `judge_answer()` | Cần checkpoint và rationale; không dùng điểm mock để kết luận. |

## Debugging và bài học

Lỗi đáng chú ý nhất là nhánh không có API key ban đầu chỉ tạo một dòng G01 và fallback judge cho điểm cố định. Điều đó làm benchmark thiếu 4 câu và gây bias. Mình sửa để chạy đủ 5 câu, kiểm tra ba nhóm câu hỏi, dùng local deterministic scorer có nhãn offline, và để remote judge thay thế khi key được cấu hình.

Bài học thứ hai là provenance không chỉ kiểm tra `NULL`; chuỗi rỗng cũng là dữ liệu không hợp lệ. Notebook có `strict_provenance_check()` và loại edge thiếu ngày/evidence trước bulk insert.

## Action Plan đồ án

**Dự án:** trợ lý hỏi đáp tài liệu kỹ thuật và lịch sử issue của phần mềm.

**Khi nào dùng GraphRAG:** dùng hybrid GraphRAG cho câu hỏi phụ thuộc nhiều issue/release/component; dùng Flat RAG cho tra cứu một đoạn tài liệu đơn lẻ để giảm latency.

**Nodes:** `Project`, `Repository`, `Issue`, `Person`, `Release`, `Component`, `Technology`.  
**Relations:** `OWNS`, `AFFECTS`, `FIXED_BY`, `AUTHORED`, `RELEASED_IN`, `DEPENDS_ON`, `USES`.

**Entity resolution:** exact ID trước, alias map theo repository/user, ANN candidate theo type, lexical guard và audit.  
**Super-node:** không traverse toàn bộ repository lớn; cap cạnh theo thời gian, lọc theo component/query, global edge cap và partition theo project.

## Tự đánh giá

| Tiêu chí | Điểm (1–5) | Ghi chú |
|---|---:|---|
| Hiểu GraphRAG | 4 | Nắm được retrieval path và giới hạn extraction. |
| Kiểm soát AI agent | 4 | Reject fallback chấm cố định và O(N²). |
| Chất lượng graph | 3 | Cần chạy lại với Neo4j + HackerNoon thật. |
| Debug/analysis | 4 | Có cache, checkpoint, audit và diagnostics. |

