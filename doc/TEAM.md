# Nhóm thực hiện K4 Track 3A

Tên nhóm: sieunhandochanh

Repo chung: https://github.com/cuong-cpu21/K4-L3-DAY24-sieunhandochanh-Eval-Guard

Trưởng nhóm/người nộp: Nguyễn Mạnh Cường - 2A202602650

Ngày lab/deadline áp dụng: 12/10/2026 — Asia/Ho_Chi_Minh (UTC+07:00)

Commit dùng để chấm: 595424bd4ee1daf8e246741d355f26da20a3aaf0

Mỗi nhóm nộp một repo. Điền đủ thành viên và một mục INDIVIDUAL cho từng người.

## Thành viên và phân công

| STT | Họ và tên | MSSV | Vai trò / công việc thật | Bằng chứng |
|---|---|---|---|---|
| 1 | Nguyễn Mạnh Cường | 2A202602650 | Toàn bộ các module RAG m1-m5, Phase A RAGAS, Phase B LLM Judge, Phase C Guardrails và báo cáo | Commit history, artifacts, reports |

Người tích hợp/chạy batch/nộp link: Nguyễn Mạnh Cường, thực hiện toàn bộ thiết lập môi trường, chạy batch và tạo báo cáo.

## Môi trường và chuẩn bị trước lớp

OS/Python/compiler: Windows 11, Python 3.11, MSVC Build Tools.
Virtualenv/dependencies/imports: Đã cài đặt đầy đủ requirements, chạy offline pytest đạt 86/86 passed.
Models/Qdrant/50 answers đã chuẩn bị: Sử dụng Qdrant in-memory/localhost, BAAI/bge-m3 cho embedding, LLM ag/gemini-3.8-flash qua Azure OpenAI endpoint.

## INDIVIDUAL

### NguyenManhCuong-2A202602650

- Phần được giao và phần thực tế đã làm: Thực hiện toàn bộ bài lab bao gồm việc tích hợp m1-m5 từ Day 18, hoàn thiện Phase A (RAGAS evaluation & failure clustering), Phase B (Pairwise LLM Judge & calibration với Cohen's Kappa), Phase C (Presidio PII masking, NeMo Guardrails input/output check & đo độ trễ latency).
- Bằng chứng: Các commit trong git history, các file mã nguồn hoàn thiện tại `src/`, và 4 artifacts trong `artifacts/`.
- Cách kiểm tra: Đã chạy toàn bộ unit tests offline với kết quả 86/86 passed, chạy live experiment để trích xuất metrics cho các báo cáo.
- Điều đã học: Hiểu rõ quy trình đánh giá hệ thống RAG với 4 chỉ số RAGAS (faithfulness, answer_relevancy, context_precision, context_recall), phương pháp LLM-as-a-Judge kèm kiểm soát position bias và tính hệ số Cohen's Kappa so với human label, kiến trúc guardrails đa tầng kết hợp regex/Presidio và semantic rails.
- Giới hạn/cách xử lý: Xử lý rate limit và format JSON từ LLM khi làm Judge, tối ưu luồng chạy tuần tự để đo độ trễ chính xác theo ngân sách 500ms.
- Nguồn/AI hỗ trợ nếu có: Tham khảo tài liệu RAGAS, NeMo Guardrails documentation và trợ lý lập trình để debug cú pháp.

## Cập nhật sau khi nộp

Không có cập nhật.
