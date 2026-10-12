# Blueprint CI/CD cho RAG và guardrails

Nhóm: sieunhandochanh

Người phụ trách: Nguyễn Mạnh Cường - 2A202602650

Đây là **đề xuất CI/CD**, không phải workflow đã được triển khai/chạy.
Nguồn kết quả: artifacts/ragas_50q.json, judge_results.json, guard_results.json.
Run timestamp: 2026-10-12T07:26:24.108830+00:00; guard models: `ag/gemini-3.8-flash`.
Models generation/evaluator/judge từ A/B: Generation model: `gemini-3.8-flash-n`; Evaluator models: `BAAI/bge-m3`, `ag/gemini-3.8-flash`; Judge model: `ag/gemini-3.8-flash`.

## Kiến trúc và failure action

Input → Presidio → NeMo input → RAG → NeMo output → response.
Request thực dừng sớm khi bị chặn; khác phép đo input layers chạy đủ hai tầng.

### Mô tả data flow và cơ chế xử lý:
1. **Layer 1 (Presidio PII Scan):** Quét phát hiện CCCD, CMND, Số điện thoại Việt Nam, Email. Nếu phát hiện PII nhạy cảm, short-circuit trả về thông báo từ chối lập tức; nếu là query cần xử lý, thực hiện redact/masking sang dạng `<VN_CCCD>`, `<EMAIL_ADDRESS>` trước khi chuyển tiếp.
2. **Layer 2 (NeMo Input Rail):** Sử dụng self-check input rail để chặn prompt injection, DAN jailbreak và yêu cầu truy cập trái phép mật khẩu admin hoặc tài liệu mật. Nếu vi phạm, dừng ngay lập tức và trả thông điệp từ chối Colang.
3. **Layer 3 (RAG Pipeline):** Thực hiện Hybrid Search (BM25 + Qdrant Dense) và Reranking với CrossEncoder, sinh câu trả lời bằng LLM.
4. **Layer 4 (NeMo Output Rail):** Kiểm duyệt câu trả lời trước khi gửi tới người dùng. Nếu LLM vô tình sinh lọt mật khẩu hoặc CCCD nhân viên, output rail sẽ chặn và thay thế bằng câu thông báo từ chối an toàn.
5. **Secrets & Fallback:** API keys được quản lý trong Vault / CI Secrets / file `.env` được bảo mật nghiêm ngặt. Nếu LLM service gặp sự cố hoặc timeout, hệ thống tự động fallback sang cơ chế trích xuất trực tiếp (extractive fallback) hoặc thông báo tạm ngưng dịch vụ.

## Phạm vi và số đo latency

Bắt buộc Task 12: **input_layers_sequential**, Presidio + NeMo input, không
short-circuit. Engines tạo trước đo; có hai warm-up thật không tính vào mẫu.
Percentile nội suy tuyến tính từ raw samples; tổng từ tổng từng sample.

Số mẫu timed: 20; warm-up: 2;
phương pháp: linear.

| Layer / scope | P50 ms | P95 ms | P99 ms | Tình trạng |
|---|---|---|---|---|
| Presidio | 0.55 ms | 0.99 ms | 1.06 ms | Cực nhanh, chiếm tỷ trọng không đáng kể |
| NeMo input | 4530.47 ms | 6132.12 ms | 10285.48 ms | Bottleneck do network latency và LLM API call |
| Input guard sequential | 4531.10 ms | 6132.71 ms | 10286.02 ms | Vượt ngưỡng ngân sách đề xuất 500ms |
| RAG | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc |
| NeMo output | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc |
| Full guard / short-circuit request | chưa đo | chưa đo | chưa đo | Không suy từ input_guard_ms |

Input-layer budget target: 500ms; kết quả thực tế/budget_ok: 6132.71 ms và False.

### Phân tích bottleneck và định hướng tối ưu:
- **Nguyên nhân vượt ngưỡng 500ms:** Presidio chạy local chỉ tốn trung bình < 1ms, hoàn toàn đáp ứng tốt. Tuy nhiên NeMo Input Rail gọi API LLM từ xa qua mạng internet (Azure proxy), khiến độ trễ P95 lên tới 6132.71ms.
- **Giải pháp tối ưu:** Trong production, cần thay thế LLM API từ xa bằng một mô hình phân loại nhỏ gọn chạy local (on-premise Small Language Model như Llama-3-8B-Quantized hoặc SetFit classification model chạy qua vLLM / TensorRT-LLM) để ép thời gian guardrail xuống dưới 150ms.
- Các tầng ngoài phạm vi (RAG, NeMo output) được giữ nguyên trạng thái "chưa đo", sẽ tiến hành lấy mẫu với batch 50 truy vấn khi triển khai staging benchmark.

## CI gates đề xuất

- RAGAS faithfulness ≥0.75, avg_score ≥0.65 trên 50 câu.
- Attacks ≥18/20 cho CI đề xuất; clean 8/8 allowed; output cases đúng.
- Full guard P95 <500ms chỉ áp dụng sau khi thực sự đo full guard.

Mốc bắt buộc lab vẫn attacks ≥15/20; bonus ≥18/20 kèm clean controls đạt.

### Đề xuất kịch bản thực thi CI Job:
```yaml
# GitHub Actions / GitLab CI Job snippet đề xuất (chưa triển khai trong repo)
eval-and-guard-gate:
  stage: test
  script:
    - python -m pytest tests/ -q
    - python scripts/run_experiments.py --phase a --live
    - python scripts/run_experiments.py --phase b --live
    - python scripts/run_experiments.py --phase c --live --latency-samples 20
    - python scripts/check_lab.py --with-tests
```
Nếu bất kỳ chỉ số nào thấp hơn ngưỡng (ví dụ attacks bị lọt > 2 câu, hoặc false positive > 0 trên clean inputs), script trả về exit code khác 0 để tự động chặn quá trình deploy merge request.

## Monitoring và cải tiến

- **Chỉ số giám sát:** Theo dõi tỷ lệ chặn (block rate) theo thời gian thực, false positive rate từ feedback người dùng (nút "báo cáo câu trả lời sai"), và độ trễ P95/P99 theo từng giờ.
- **Xử lý drift & privacy:** Lấy mẫu ngẫu nhiên 5% query thực tế hàng ngày (đã anonymize PII bằng Presidio) để chuyên gia gắn nhãn đối chiếu và phát hiện distribution drift.
- **Kết quả thực nghiệm của nhóm:**
  - RAGAS mean/worst distribution: mean tổng thể 0.8525; worst distribution là adversarial với mean 0.8200 (context recall 0.7299).
  - Cohen’s κ: 1.0000 (độ đồng thuận hoàn hảo với nhãn người trên 10 câu calibration).
  - Attacks passed: 20/20 (100% các cuộc tấn công jailbreak và PII injection đều bị chặn); clean false_positive_rate: 0.0 (0/8 câu hợp lệ bị chặn nhầm).

### 3–5 đề xuất cải tiến có thể kiểm chứng:
1. **Bổ sung metadata filtering cho tài liệu cũ:** Loại bỏ các chunk chính sách năm 2023 khi truy vấn không yêu cầu so sánh lịch sử để tăng `context_recall` cho adversarial set từ 0.7299 lên >0.85.
2. **Local Guardrail Classifier:** Thay thế NeMo LLM-based check bằng mô hình phân loại intent chạy local (ONNX Runtime) để giảm độ trễ input guard từ 6132ms xuống dưới 200ms.
3. **Cải tiến Chain-of-Thought trong RAG prompt:** Hướng dẫn LLM tính toán chi tiết ngày phép và phụ cấp cho các câu hỏi multi-hop số học nhằm nâng cao `answer_relevancy` trên 0.85.
4. **Active Learning Feedback Loop:** Thu thập các câu query bị chặn biên để huấn luyện bổ sung quy tắc cho NeMo Colang rules.
