# Technical Defense — Lab 19

**Học viên:** Nguyen Phuong Dong  
**Ngày:** 19/08/2026  

> Ghi chú trung thực: workspace hiện không chứa thông tin xác thực Neo4j/Groq/Hugging Face, vì vậy các CSV đi kèm là offline smoke artifact. Khi chạy với credentials, notebook sẽ thay thế bằng dữ liệu HackerNoon và LLM Judge thật.

## 1. Coreference Resolution

Pipeline dùng `resolve_coref_batch()` với prompt bảo thủ: chỉ thay đại từ khi tiền ngữ xuất hiện rõ trong cùng chunk. Nếu model trả JSON lỗi, thiếu item hoặc tình huống mơ hồ, code giữ `resolved_text` nguyên bản và ghi `COREF_BATCH_FAILED`/`unresolved_mentions`. Cách này ưu tiên tránh false edge hơn là cố tăng recall. Ví dụ rủi ro là câu “The company acquired it” khi chunk vừa nhắc nhiều công ty; câu này phải giữ nguyên và không tạo cạnh ACQUIRED.

## 2. Entity Resolution Threshold và Lexical Guard

Ngưỡng ANN là cosine similarity `0.90`; fuzzy seed matching dùng `0.66`. Lexical guard dùng SequenceMatcher tối thiểu `0.72`, sau khi bỏ hậu tố pháp nhân. Guard chặn các cặp nguy hiểm như `Sam Altman`/`Steve Altman` (cùng họ nhưng khác tên) và `Apple`/`Apple Music` (một bên là công ty, một bên là sản phẩm). Chỉ entity cùng type mới được merge; manual alias như `MSFT` → `Microsoft` được ghi `MERGE_MANUAL`, các cặp vector được ghi `MERGE_VECTOR` hoặc `REJECT_GUARD`.

## 3. Đồ thị và Super-node

Top degree thật được lấy sau khi chạy Neo4j qua `top_degree_df`; preflight offline không có graph nên không tự bịa top-3. Chính sách retrieval là: node có degree `>100` chỉ lấy 50 cạnh mới nhất, toàn traversal tối đa 250 cạnh và textual context tối đa 14.000 ký tự. Temporal cap giảm token explosion và ưu tiên thông tin mới, nhưng có thể bỏ sót một quan hệ lịch sử. Với truy vấn lịch sử, cần bổ sung filter theo khoảng ngày hoặc route self-correction.

## 4. Flat RAG và GraphRAG

Flat RAG dùng `faiss.IndexFlatIP`, embedding chuẩn hóa và top-k 6. GraphRAG lấy seed, exact/ANN match, BFS tối đa 2 hop, cắt super-node rồi ghép `=== GRAPH ===` với `=== VECTOR ===`. Flat thường nhanh và rẻ hơn; GraphRAG có indexing/extraction overhead nhưng giữ được chuỗi quan hệ và provenance `source_chunk_id`, `published_date`, `evidence`.

Các số trong CSV hiện tại chỉ là smoke fallback, không dùng để kết luận chất lượng production. Kết quả graded phải được tạo lại sau khi có `GROQ_API_KEY`, `NEO4J_URI` và judge key.

## 5. Bulk ingestion và provenance

Nodes và edges đều dùng `UNWIND $rows AS row`, batch mặc định 1.000. Mỗi edge bắt buộc có `source_chunk_id`, `published_date`, `evidence`, `confidence`; edge thiếu ngày hoặc evidence bị loại khỏi extraction. `strict_provenance_check()` kiểm tra cả `NULL` lẫn chuỗi rỗng, tránh lỗi “đếm thiếu provenance bằng 0” khi property tồn tại nhưng rỗng.

## 6. Failure mode của Flat RAG

Với câu multi-hop, hai bằng chứng có thể nằm ở hai chunk khác nhau và không cùng vocabulary với câu hỏi. Top-k vector có thể lấy chunk nói về đầu tư nhưng bỏ chunk nói về founder/work history. GraphRAG nối hai mẩu đó qua các cạnh `WORKED_AT`, `FOUNDED`, `INVESTED_IN`, đồng thời cho generator thấy provenance của từng cạnh.

## 7. Failure mode của GraphRAG

GraphRAG vẫn phụ thuộc NER/RE và seed extraction. Nếu LLM không trích được seed, entity resolution merge sai, hoặc edge bị super-node cap loại khỏi 50 cạnh mới nhất, traversal sẽ thiếu context. Code trả diagnostics (`NO_SEED`, `supernode_events`, `collected_edges`) để truy vết thay vì coi câu trả lời rỗng là thành công.

## 8. Vì sao cần Hybrid Context

Graph tốt cho cấu trúc và multi-hop nhưng không giữ toàn bộ diễn đạt tự nhiên. Vector chunks giữ evidence nguyên văn và các chi tiết chưa được trích xuất thành relation. Ghép hai nguồn giúp generator vừa suy luận theo đường đi vừa kiểm tra lại bằng văn bản gốc.

## 9. Trade-off khi scale 350MB

Bottleneck đầu tiên là LLM extraction và rate limit, sau đó là RAM/latency của embedding và ANN. Cách scale là stream + checkpoint, batch async có retry/backoff, cache theo `chunk_id`, dùng HNSW/IVF thay FlatIP khi dữ liệu lớn, blocking theo type/name prefix trước entity resolution, và partition graph theo community. Không gửi cả dataset vào một prompt.

## 10. Kiểm soát AI Coding Agent

Quyết định không dùng pairwise cosine `O(N²)` cho toàn bộ mentions; thay bằng FAISS ANN + lexical guard + Union-Find. Cũng không chấp nhận fallback chấm mọi câu 4/5, vì đó là benchmark giả. Fallback hiện tại chạy đủ 5 câu nhưng dùng deterministic token-overlap và ghi rõ `offline smoke`; chỉ LLM Judge thật mới được dùng cho kết luận chất lượng.

