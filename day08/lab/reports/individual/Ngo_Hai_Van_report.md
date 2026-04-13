# Báo Cáo Cá Nhân — Lab Day 08: RAG Pipeline

**Họ và tên:** Ngô Hải Văn  
**Vai trò trong nhóm:** Retrieval Owner — Hybrid & Rerank  
**Ngày nộp:** 2026-04-13  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi đã làm gì trong lab này? (100-150 từ)

Tôi phụ trách Sprint 3 — phần nâng cao retrieval trong file `rag_answer.py`. Cụ thể, tôi implement ba hàm chính: `retrieve_sparse()` sử dụng BM25Okapi để tìm kiếm theo keyword, `retrieve_hybrid()` kết hợp dense và sparse bằng Reciprocal Rank Fusion (RRF) với trọng số dense_weight=0.6 và sparse_weight=0.4, và `rerank()` hỗ trợ cả cross-encoder lẫn fallback lexical rerank (kết hợp 70% dense score + 30% keyword overlap). Ngoài ra, tôi viết các hàm tiện ích `_normalize_tokens()` và `_doc_key()` để chuẩn hóa token và tránh trùng lặp khi merge kết quả RRF. Công việc của tôi nối trực tiếp với phần baseline dense retrieval của Dũng — tôi nhận đầu ra từ `retrieve_dense()` của Dũng làm một nhánh input cho hybrid, đồng thời bổ sung nhánh sparse để tăng recall cho các query chứa keyword đặc thù. Tôi cũng ghi lại toàn bộ quá trình thử nghiệm vào `docs/tuning-log.md`.

---

## 2. Điều tôi hiểu rõ hơn sau lab này (100-150 từ)

Trước lab, tôi chỉ biết dense retrieval dùng embedding là đủ tốt. Sau khi thực hành, tôi hiểu rõ hơn rằng dense search mạnh ở ngữ nghĩa (paraphrase, đồng nghĩa) nhưng yếu ở exact match — ví dụ query "ERR-403-AUTH" hay "P1" cần đúng từ khóa đó trong chunk, và BM25 xử lý tốt hơn hẳn ở trường hợp này. Reciprocal Rank Fusion là cách kết hợp hai danh sách kết quả mà không cần normalize score về cùng thang — chỉ dựa trên thứ hạng (rank) với hằng số k=60, đơn giản nhưng hiệu quả. Tôi cũng hiểu rõ hơn về funnel logic: search rộng (top-10) rồi rerank chọn lọc (top-3) giúp giảm noise trong prompt mà vẫn giữ được chunk relevant nhất. Điều này quan trọng vì context quá dài sẽ gây hiện tượng "lost in the middle" khiến LLM bỏ sót thông tin ở giữa.

---

## 3. Điều tôi ngạc nhiên hoặc gặp khó khăn (100-150 từ)

Điều ngạc nhiên lớn nhất là baseline ban đầu cho Context Recall = 0.00/5 và Completeness = 1.00/5. Giả thuyết ban đầu của tôi là do dense retrieval bỏ lỡ keyword, nhưng thực tế nguyên nhân gốc đơn giản hơn: chưa build index `rag_lab`, nên retriever không có dữ liệu để lấy context, toàn bộ pipeline rơi vào chế độ abstain. Sau khi build index xong, scorecard baseline nhảy lên Faithfulness 4.60, Context Recall 5.00 — cho thấy lỗi nằm ở indexing chứ không phải retrieval. Khó khăn kỹ thuật lớn nhất là xử lý dedup khi merge kết quả RRF — cùng một chunk có thể xuất hiện trong cả dense lẫn sparse với metadata hơi khác nhau. Tôi giải quyết bằng hàm `_doc_key()` tạo composite key từ source + section + 160 ký tự đầu của text để nhận diện chunk trùng.

---

## 4. Phân tích một câu hỏi trong scorecard (150-200 từ)

**Câu hỏi:** q07 — "Approval Matrix để cấp quyền hệ thống là tài liệu nào?"

**Phân tích:**

Đây là câu hỏi difficulty=hard vì dùng tên cũ "Approval Matrix" trong khi tài liệu thực tế đã đổi tên thành "Access Control SOP". Trong scorecard baseline (dense), q07 đạt Faithfulness=5 nhưng Completeness chỉ 1/5 — nghĩa là model trả lời đúng với context nhận được, nhưng context thiếu thông tin cần thiết để trả lời đầy đủ.

Lỗi nằm ở tầng **retrieval**: dense embedding encode "Approval Matrix" thành vector ngữ nghĩa chung chung về "phê duyệt quyền truy cập", không match chính xác với cụm từ "Access Control SOP" trong tài liệu. Đây chính là use case lý tưởng cho hybrid retrieval — BM25 sparse search có thể bắt được từ "Approval" và "Matrix" xuất hiện trong metadata hoặc nội dung chunk, bổ sung cho dense search.

Variant hybrid được kỳ vọng cải thiện recall cho q07 vì RRF kết hợp cả rank từ dense (bắt ngữ nghĩa "cấp quyền") lẫn sparse (bắt keyword "Approval"). Đây là lý do chính tôi chọn hybrid làm variant thay vì chỉ dùng rerank — rerank chỉ sắp xếp lại kết quả đã có, còn hybrid mở rộng tập kết quả ban đầu.

---

## 5. Nếu có thêm thời gian, tôi sẽ làm gì? (50-100 từ)

Tôi sẽ bật cross-encoder rerank (đặt `RERANK_WITH_CROSS_ENCODER=1` với model `ms-marco-MiniLM-L-6-v2`) kết hợp sau hybrid retrieval. Scorecard cho thấy q09 và q10 bị abstain dù có context (Relevance=2), gợi ý rằng top-3 chunk đưa vào prompt chưa đủ relevant. Cross-encoder chấm lại từng cặp (query, chunk) sẽ lọc chính xác hơn fallback lexical rerank hiện tại, giúp giảm noise và tăng Completeness cho các câu hỏi khó.

---

*File: `reports/individual/Ngo_Hai_Van_report.md`*
