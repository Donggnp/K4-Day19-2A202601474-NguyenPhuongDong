# Báo cáo Lab 19 — Production GraphRAG vs Flat RAG

**Học viên:** Nguyen Phuong Dong
**Ngày thực hiện:** 19/08/2026

## 1. Tóm tắt triển khai

Notebook triển khai pipeline gồm: stream dataset, exact dedup bằng SHA-1, rolling chunk 220/40, conservative coreference, NER/RE theo allowlist, entity resolution bằng FAISS ANN + lexical guard + Union-Find, Neo4j bulk insert bằng `UNWIND`, Flat RAG `IndexFlatIP`, BFS GraphRAG, super-node cap và Golden evaluation.

Các ngưỡng chính:

- `LAB_MAX_ARTICLES=1500`, `LAB_MAX_CHUNKS=3000`, `EXTRACTION_MAX_CHUNKS=400`.
- Entity ANN cosine `0.90`; fuzzy seed fallback `0.66`; lexical guard `0.72`.
- Super-node degree `>100` → tối đa 50 cạnh mới nhất; global cap 250 cạnh; graph context 14.000 ký tự.

## 2. Kết quả và giới hạn

Notebook đã được kiểm tra JSON/AST và có đủ 5 Golden questions thuộc `factoid`, `multi-hop`, `cross-doc`. Hai file CSV được tạo ở `outputs/`. Do workspace không có credentials, artifact hiện tại là offline smoke fallback; các điểm không đại diện cho LLM-as-a-Judge production. Khi chạy lại với secrets, cell evaluation sẽ ghi đè CSV bằng kết quả thật và checkpoint giúp resume.

GraphRAG có lợi thế ở multi-hop nhờ nối quan hệ và giữ provenance. Flat RAG thường nhanh/rẻ hơn vì không có graph extraction/traversal. GraphRAG vẫn có failure mode: seed miss, extraction miss, false merge hoặc temporal cap bỏ cạnh lịch sử.

## 3. Phân tích lỗi

1. Flat RAG có thể lấy chunk funding nhưng bỏ chunk founder/employment, vì vector search không biểu diễn relation path.
2. GraphRAG có thể mất cạnh do NER/RE hoặc super-node cap; diagnostics và audit giúp phân biệt `NO_SEED`, thiếu edge và context bị cắt.
3. Cosine cao không đủ cho entity identity; lexical guard chặn `Sam Altman`/`Steve Altman` và `Apple`/`Apple Music`.

Chi tiết 10 câu bảo vệ nằm trong [technical_defense.md](technical_defense.md), failure analysis nằm trong [failure_analysis.md](failure_analysis.md), và reflection cá nhân nằm trong [reflection_NguyenPhuongDong.md](reflection_NguyenPhuongDong.md).

## 4. Reflection

Điểm quan trọng nhất là không biến fallback thành kết quả giả: pipeline phải phân biệt rõ remote LLM run và offline smoke run. Trong đồ án thực tế, mình sẽ dùng hybrid retrieval: Flat RAG cho factoid, GraphRAG cho câu hỏi nối nhiều issue/release/component, kèm entity audit và super-node policy ngay từ đầu.
