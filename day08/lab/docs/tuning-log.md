# Tuning Log — RAG Pipeline (Day 08 Lab)

> Template: Ghi lại mỗi thay đổi và kết quả quan sát được.
> A/B Rule: Chỉ đổi MỘT biến mỗi lần.

---

## Baseline (Sprint 2)

**Ngày:** 2026-04-13  
**Config:**
```
retrieval_mode = "dense"
chunk_size = 400 tokens
overlap = 80 tokens
top_k_search = 10
top_k_select = 3
use_rerank = False
llm_model = gpt-4o-mini (fallback: gemini-1.5-flash)
```

**Scorecard Baseline:**
| Metric | Average Score |
|--------|--------------|
| Faithfulness | 5.00 /5 |
| Answer Relevance | 2.00 /5 |
| Context Recall | 0.00 /5 |
| Completeness | 1.00 /5 |

**Câu hỏi yếu nhất (điểm thấp):**
q01, q02, q03 (và hầu như toàn bộ tập test) có completeness thấp vì chưa build index `rag_lab`, nên retriever không có dữ liệu để lấy context.
Kết quả baseline hiện tại chủ yếu phản ánh trạng thái "abstain an toàn" chứ chưa phản ánh chất lượng retrieval thật.

**Giả thuyết nguyên nhân (Error Tree):**
- [ ] Indexing: Chunking cắt giữa điều khoản
- [ ] Indexing: Metadata thiếu effective_date
- [x] Retrieval: Dense bỏ lỡ exact keyword / alias
- [x] Retrieval: Top-k quá ít → thiếu evidence
- [ ] Generation: Prompt không đủ grounding
- [ ] Generation: Context quá dài → lost in the middle

---

## Variant 1 (Sprint 3)

**Ngày:** 2026-04-13  
**Biến thay đổi:** Retrieval strategy (Dense -> Hybrid)  
**Lý do chọn biến này:**
Corpus chứa nhiều từ khóa đặc thù (P1, Level 3, Approval Matrix, ERR-403) bên cạnh mô tả ngôn ngữ tự nhiên.
Hybrid (Dense + BM25) giúp tăng khả năng bắt exact match và alias, nhất là cho các câu hỏi dạng mã lỗi/tên cũ tài liệu.

**Config thay đổi:**
```
retrieval_mode = "hybrid"
# Các tham số còn lại giữ nguyên như baseline
```

**Scorecard Variant 1:**
| Metric | Baseline | Variant 1 | Delta |
|--------|----------|-----------|-------|
| Faithfulness | 5.00/5 | TODO | TODO |
| Answer Relevance | 2.00/5 | TODO | TODO |
| Context Recall | 0.00/5 | TODO | TODO |
| Completeness | 1.00/5 | TODO | TODO |

**Nhận xét:**
Pipeline variant đã code xong (sparse BM25 + RRF hybrid + rerank fallback).
Cần chạy lại sau khi build index để có so sánh thực nghiệm chính xác theo từng câu.

**Kết luận:**
Chưa kết luận định lượng vì baseline hiện tại chạy khi chưa có index.
Bước kế tiếp là build index và chạy đủ A/B để xác nhận delta trên context recall và completeness.

---

## Variant 2 (nếu có thời gian)

**Biến thay đổi:** ___________  
**Config:**
```
# TODO
```

**Scorecard Variant 2:**
| Metric | Baseline | Variant 1 | Variant 2 | Best |
|--------|----------|-----------|-----------|------|
| Faithfulness | ? | ? | ? | ? |
| Answer Relevance | ? | ? | ? | ? |
| Context Recall | ? | ? | ? | ? |
| Completeness | ? | ? | ? | ? |

---

## Tóm tắt học được

> TODO (Sprint 4): Điền sau khi hoàn thành evaluation.

1. **Lỗi phổ biến nhất trong pipeline này là gì?**
   > Build index chưa được chạy trước retrieval, khiến toàn bộ pipeline rơi vào chế độ abstain.

2. **Biến nào có tác động lớn nhất tới chất lượng?**
   > Retrieval strategy (đặc biệt hybrid cho câu hỏi chứa alias/keyword).

3. **Nếu có thêm 1 giờ, nhóm sẽ thử gì tiếp theo?**
   > Bật cross-encoder rerank để lọc top-k cuối và giảm chunk nhiễu trong prompt.
