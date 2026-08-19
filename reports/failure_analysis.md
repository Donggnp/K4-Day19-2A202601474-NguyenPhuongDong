# Failure Analysis — Lab 19

## Case 1 — Flat RAG bỏ sót chuỗi multi-hop

**Tình huống:** câu hỏi yêu cầu nối founder từng làm việc ở Microsoft với startup sau đó nhận đầu tư từ Google.

**Quan sát:** một chunk có thể chứa funding announcement, còn chunk khác chứa employment/founding history. Vector top-k tối ưu tương đồng ngữ nghĩa cục bộ, không bảo đảm lấy đủ cả hai chunk.

**Root cause:** Flat RAG không có explicit relation path; mỗi chunk được xếp hạng độc lập.

**Mitigation:** GraphRAG resolve seed entities, BFS hai hop và linearize các cạnh có `source_chunk_id`/`published_date`. Nếu seed không match, diagnostics báo `NO_SEED` và hybrid vector context vẫn là fallback.

## Case 2 — GraphRAG mất cạnh do extraction hoặc super-node cap

**Tình huống:** một công ty lớn có hơn 100 cạnh, nhưng câu hỏi cần một quan hệ cũ hơn 50 cạnh mới nhất.

**Root cause:** NER/RE có thể bỏ qua quan hệ; hoặc policy temporal cap loại cạnh lịch sử để bảo vệ context. Hai lỗi đều làm đường BFS không đầy đủ dù thông tin có trong dữ liệu.

**Mitigation:** lưu evidence và confidence ở edge, kiểm tra provenance trước ingestion, xuất `supernode_events`, giới hạn global 250 cạnh. Với truy vấn lịch sử, thêm date-aware retrieval hoặc self-correction hop 3/vector fallback; không tăng cap vô hạn.

## Case 3 — False merge entity

**Tình huống:** embedding của `Sam Altman` và `Steve Altman` có thể cao vì cùng họ; `Apple` và `Apple Music` cũng có token chung.

**Root cause:** cosine similarity không đủ để xác định đồng nhất danh tính.

**Mitigation:** type guard, bỏ hậu tố pháp nhân, guard tên người khác first-name, chặn quan hệ substring/product marker, và lưu audit decision `REJECT_GUARD`. Khi uncertain, giữ hai node riêng để tránh false edge.

## Kết luận

GraphRAG không tự động đúng hơn: chất lượng phụ thuộc extraction, resolution và seed recall. Vì vậy pipeline đo cả quality, latency, token usage, provenance và diagnostics; một câu trả lời có điểm cao nhưng thiếu citation không được xem là production-ready.

