# Đánh giá LLM Judge Phase B

Nhóm: sieunhandochanh

Người phụ trách: Nguyễn Mạnh Cường - 2A202602650

Nguồn: `artifacts/judge_results.json`; run timestamp: 2026-10-12T07:15:37.425258+00:00.
Models generation/judge: Generation model: `gemini-3.8-flash-n`; Judge model: `ag/gemini-3.8-flash`.

## Năm cặp A/B và hai lượt

Nguồn cố định: A từ artifacts/answers_50q.json:answer; B từ
fixtures/human_labels_10q.json:model_answer; reference từ fixtures/test_set_50q.json:ground_truth.
Ids: 1, 5, 12, 21, 41. Không đổi nguồn sau khi xem kết quả judge.

| ID | Câu hỏi | Winner Lượt 1 | Winner Lượt 2 | Winner Chung cuộc | Nhất quán vị trí | Điểm Lượt 1 (A/B) | Điểm Lượt 2 (A/B) |
|---|---|---|---|---|---|---|---|
| 1 | Nhân viên được nghỉ bao nhiêu ngày khi kết hôn? | B | B | B | True | 0.80 / 0.95 | 0.75 / 0.90 |
| 5 | Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt? | A | A | A | True | 1.00 / 0.00 | 1.00 / 0.00 |
| 12 | Thưởng Tết tối thiểu cho nhân viên chính thức có từ 6 tháng trở lên là bao nhiêu? | A | A | A | True | 1.00 / 0.60 | 1.00 / 0.70 |
| 21 | Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào? | B | B | B | True | 0.50 / 1.00 | 0.50 / 1.00 |
| 41 | Nhân viên được nghỉ bao nhiêu ngày phép năm? | A | A | A | True | 0.95 / 0.10 | 0.95 / 0.20 |

Position bias count/rate: 0 (tỷ lệ 0.0%, 0/5 cases đổi quyết định khi tráo vị trí).
Verbosity win rate: 0.8 (80.0%); decisive cases: 5 (4/5 cases quyết định thắng khi câu trả lời dài hơn).

### Giải thích các case và phân tích bias:
- **Tính nhất quán vị trí (Position Consistency):** Đạt 100% (5/5 cặp). Cả 5 câu hỏi khi đảo thứ tự xuất hiện giữa Candidate 1 và Candidate 2 đều cho ra cùng một winner tương ứng (A hoặc B). Điều này chứng minh LLM Judge không bị ảnh hưởng bởi thứ tự trình bày prompt (không có primary-bias hay recency-bias).
- **Thiên lệch độ dài (Verbosity Bias):** 4/5 câu winner là câu dài hơn (câu 1, 5, 12, 41). Tuy nhiên, khi phân tích sâu nội dung ngữ nghĩa, câu dài hơn chiến thắng là do cung cấp đầy đủ thông tin hơn (ví dụ: nêu rõ điều kiện thâm niên, phân biệt phiên bản 2023 vs 2024 ở câu 41, hoặc nêu rõ người phê duyệt và hạn mức ở câu 5) chứ không phải do LLM chấm thiên vị hình thức độ dài đơn thuần.

## Calibration đúng model_answer

Helper đánh giá đúng question/model_answer đang có trong fixtures/human_labels_10q.json;
không lấy answer mới của pipeline. Prompt chỉ nhận question, answer, reference.
Không gửi human_label, human_note hoặc phân bố nhãn human vào prompt.

| Question ID | Model Answer Gốc | Nhãn Judge | Reasoning của Judge | Nhãn Human | Đồng thuận (Agree) |
|---|---|---|---|---|---|
| 1 | Nhân viên được nghỉ 3 ngày làm việc có lương khi kết hôn. | 1 | Câu trả lời hoàn toàn chính xác theo thông tin trong tài liệu tham khảo, nêu rõ số ngày nghỉ kết hôn là 3 ngày làm việc có lương. | 1 | True |
| 5 | Cần Giám đốc phòng ban phê duyệt. | 0 | Câu trả lời sai người phê duyệt; theo tài liệu tham khảo, đơn hàng trên 50 triệu (ở đây là 55 triệu) cần Tổng Giám đốc (CEO) phê duyệt. | 0 | True |
| 12 | Nhân viên được thưởng Tết tối thiểu 1 tháng lương. | 1 | Câu trả lời chính xác và đầy đủ thông tin theo reference, nêu rõ mức thưởng Tết tối thiểu là 1 tháng lương. | 1 | True |
| 21 | Được nghỉ 18 ngày phép (15 cơ bản + 3 thâm niên). Lương Senior từ 20 đến 35 triệu. | 1 | Câu trả lời chính xác và đầy đủ số ngày phép năm cùng khoảng lương theo thông tin trong tài liệu tham khảo. | 1 | True |
| 23 | Hoàn trả 25 triệu vì chưa đủ 1 năm cam kết. | 1 | Câu trả lời hoàn toàn chính xác theo thông tin trong tài liệu tham khảo: phải hoàn trả 100% chi phí (25 triệu) do nghỉ việc trước cam kết. | 1 | True |
| 29 | Trưởng phòng phê duyệt. Phạt 2% tháng trên 8 triệu. | 0 | Câu trả lời thiếu người phê duyệt là Kế toán trưởng (cần cả Trưởng phòng và Kế toán trưởng theo quy định) và không nêu rõ cách tính phí phạt cụ thể. | 0 | True |
| 33 | Phép: 19 ngày. Phụ cấp: 1.500.000 VNĐ/tháng. | 1 | Câu trả lời cung cấp chính xác và đầy đủ số ngày phép năm (19 ngày) và tổng phụ cấp hàng tháng (1.500.000 VNĐ/tháng) theo đúng thông tin từ reference. | 1 | True |
| 41 | Nhân viên được nghỉ 12 ngày phép năm. | 0 | Câu trả lời sử dụng số liệu từ chính sách cũ v2023 (12 ngày) đã bị thay thế, không chính xác theo chính sách hiện hành v2024 (15 ngày cơ bản). | 0 | True |
| 46 | Nhân viên thử việc không được nghỉ phép năm và phải xin nghỉ không lương nếu cần. | 1 | Câu trả lời chính xác theo reference, trả lời đúng trọng tâm câu hỏi là nhân viên thử việc không được nghỉ phép năm. | 1 | True |
| 50 | Được, miễn là đảm bảo kết nối an toàn. | 0 | Câu trả lời sai hoàn toàn so với reference; chính sách cấm sử dụng VPN cá nhân (NordVPN) và bắt buộc dùng VPN của công ty. | 0 | True |

Cohen’s κ: 1.0000.
Observed/chance agreement và diễn giải:
- Observed agreement $P_o = \frac{10}{10} = 1.0$ (100% câu trùng khớp giữa Judge và Human).
- Chance agreement $P_e = (0.6 \times 0.6) + (0.4 \times 0.4) = 0.36 + 0.16 = 0.52$.
- Hệ số Cohen's $\kappa = \frac{P_o - P_e}{1 - P_e} = \frac{1.0 - 0.52}{1 - 0.52} = 1.0$.
- Diễn giải: Mức độ đồng thuận tuyệt đối (Near-perfect agreement, $\kappa = 1.0 > 0.6$). LLM Judge phân biệt rất chính xác giữa câu trả lời đúng và các trường hợp sai người phê duyệt (câu 5), thiếu điều kiện (câu 29), dùng chính sách cũ đã hết hạn (câu 41) hoặc vi phạm chính sách bảo mật (câu 50).

## Giới hạn và cách dùng judge

- **Giới hạn số mẫu:** Bộ calibration gồm 10 câu nhãn người và 5 cặp pairwise. Mặc dù đạt $\kappa = 1.0$, mẫu thử còn tương đối nhỏ để kết luận khái quát cho toàn bộ không gian ngôn ngữ phức tạp.
- **Vai trò của LLM Judge:** LLM Judge là công cụ tự động hóa đắc lực giúp sàng lọc nhanh chất lượng trong CI/CD, nhưng không thể thay thế hoàn toàn người đánh giá (Human-in-the-loop). Trong production, các câu hỏi nhạy cảm hoặc điểm số biên (borderline scores gần ngưỡng quyết định) vẫn cần được đẩy vào hàng đợi kiểm duyệt thủ công của chuyên gia.
