# Phân tích lỗi Phase A

Nhóm: Cuong-Solo

Người phụ trách: Nguyễn Mạnh Cường - 2A202602650

Nguồn: `artifacts/ragas_50q.json`; run timestamp: 2026-10-12T07:13:15.544603+00:00.
Generation mode/fallback và điều kiện chạy: Toàn bộ 50/50 câu hỏi được sinh bằng LLM (`generation_mode: llm`), sử dụng pipeline RAG tích hợp đầy đủ m1–m5 (Hierarchical chunking, BM25 + Dense Qdrant, CrossEncoder reranking).
Models generation/evaluator: Generation model: `gemini-3.8-flash-n`; Evaluator models: `BAAI/bge-m3`, `ag/gemini-3.8-flash`.

## Điểm theo distribution

| Metric | factual (20) | multi_hop (20) | adversarial (10) |
|---|---|---|---|
| faithfulness | 0.9000 | 0.8400 | 0.8600 |
| answer_relevancy | 0.8500 | 0.7900 | 0.8100 |
| context_precision | 0.8800 | 0.8800 | 0.8800 |
| context_recall | 0.9332 | 0.8765 | 0.7299 |
| avg_score | 0.8908 | 0.8466 | 0.8200 |

## Bottom 10 và nguyên nhân

| Rank | Question ID | Distribution | Câu hỏi | Avg Score | Worst Metric | Diagnosis | Suggested Fix |
|---|---|---|---|---|---|---|---|
| 1 | 27 | multi_hop | Nhân viên có thâm niên 7 năm theo v2024 được nghỉ bao nhiêu ngày phép? Nếu bị trừ thêm 4 ngày ốm không có giấy y tế thì còn lại bao nhiêu? | 0.5825 | answer_relevancy | Answer doesn't match question | Improve prompt template |
| 2 | 33 | multi_hop | Nhân viên Manager có thâm niên 12 năm: tổng phụ cấp hàng tháng và số ngày phép năm theo v2024 là bao nhiêu? | 0.5875 | answer_relevancy | Answer doesn't match question | Improve prompt template |
| 3 | 50 | adversarial | Nhân viên Manager có thể dùng VPN cá nhân (như NordVPN) khi WFH để tăng bảo mật thêm không? | 0.6247 | answer_relevancy | Answer doesn't match question | Improve prompt template |
| 4 | 30 | multi_hop | So sánh quyền lợi bảo hiểm giữa nhân viên thử việc và nhân viên chính thức. | 0.6672 | answer_relevancy | Answer doesn't match question | Improve prompt template |
| 5 | 41 | adversarial | Nhân viên được nghỉ bao nhiêu ngày phép năm? | 0.7980 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |
| 6 | 44 | adversarial | Bao lâu phải đổi mật khẩu một lần? | 0.8122 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |
| 7 | 45 | adversarial | Có cần kích hoạt xác thực đa yếu tố (MFA) không? | 0.8200 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |
| 8 | 49 | adversarial | Theo chính sách nghỉ phép cũ (v2023), nhân viên được nghỉ bao nhiêu ngày? Còn chính sách nào đang có hiệu lực hiện tại? | 0.8200 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |
| 9 | 21 | multi_hop | Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào? | 0.8271 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |
| 10 | 42 | adversarial | Thâm niên bao nhiêu năm thì được cộng thêm ngày phép? | 0.8368 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |

### Đối chiếu đại diện các nhóm lỗi:
- **Lỗi Multi-hop (Câu 27, 33):** Các câu hỏi đòi hỏi tính toán 2 bước (thâm niên 7 năm -> 17 ngày phép, trừ 4 ngày ốm -> còn 13 ngày). LLM chỉ trả lời đúng số ngày phép theo thâm niên nhưng bỏ sót vế suy luận trừ ngày ốm không có giấy y tế, dẫn tới `answer_relevancy` sụt giảm mạnh.
- **Lỗi Adversarial / Version trap (Câu 41, 49, 50):** Câu 41 hỏi chung chung không nêu năm, câu 49 hỏi chính sách cũ 2023 kèm chính sách hiện hành. Retrieval lấy thiếu chunk phiên bản 2024 hoặc bị nhiễu do chunk cũ 2023 có độ tương đồng BM25 cao hơn, dẫn tới `context_recall` thấp (0.7299).

## Matrix và nhóm yếu nhất

| Metric | factual | multi_hop | adversarial | Tổng |
|---|---|---|---|---|
| faithfulness | 0 | 0 | 0 | 0 |
| context_recall | 2 | 2 | 6 | 10 |
| context_precision | 0 | 0 | 0 | 0 |
| answer_relevancy | 18 | 18 | 4 | 40 |

Nhóm yếu theo mean thấp nhất: adversarial; mean: 0.8200.
Worst metric xuất hiện nhiều nhất: answer_relevancy; số câu: 40.

### Giải thích nguyên nhân và đề xuất cải tiến:
- **Vì sao adversarial yếu nhất:** Nhóm câu hỏi adversarial có điểm trung bình thấp nhất (0.8200) do `context_recall` sụt xuống 0.7299. Các câu hỏi adversarial cố tình tạo bẫy phiên bản (version conflicts giữa chính sách 2023 và 2024) hoặc bẫy phủ định, khiến bộ tìm kiếm hybrid không lấy đủ context mới nhất hoặc trả về chunk phiên bản cũ.
- **Đề xuất khắc phục cụ thể:**
  1. Thêm bộ lọc metadata filter theo phiên bản tài liệu (`version >= 2024` hoặc `status == active`) trong M2 Hybrid Search.
  2. Bổ sung Chain-of-Thought (CoT) prompt trong khâu sinh trả lời để LLM giải quyết trọn vẹn các câu hỏi multi-hop có phép tính số học thay vì chỉ trích xuất trực tiếp.
  3. Để kiểm tra hiệu quả, chạy lại pipeline đánh giá với `python scripts/run_experiments.py --phase a --live` và so sánh avg_score của adversarial xem có tăng lên trên 0.85 không.

## Bẫy adversarial và giới hạn

So với nhóm factual (0.8908) và multi-hop (0.8466), nhóm adversarial chịu ảnh hưởng nặng nhất ở khâu retrieval context recall (0.7299 so với 0.9332 ở factual). Trong Bottom 10 có tới 6/10 câu thuộc nhóm adversarial (câu 41, 42, 44, 45, 49, 50). Điểm thấp của nhóm này phản ánh trung thực thách thức của RAG trước các câu hỏi mập mờ về phiên bản chính sách.
Giới hạn thực nghiệm: Bộ test gồm 50 câu (10 adversarial, 20 multi-hop, 20 factual). Việc đánh giá dùng kết hợp LLM thế hệ mới và hàm heuristic đối chiếu text overlap nên có thể chưa phản ánh hết ngữ nghĩa tinh vi trong một số câu phủ định phức tạp.
