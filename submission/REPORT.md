# Báo cáo cá nhân — K4-L3B Day 13 Monitoring & LLMOps

## 1. Thông tin học viên

- **Họ và tên:** Đỗ Đình Long
- **MSSV:** 2A202602673
- **Lớp:** K4-L3B
- **Repository URL:** https://github.com/long2110d/K4-L3-DAY13-DoDinhLong-2A202602673-Monitoring-LLMOps
- **Commit HEAD khi đối chiếu:** `fcad74bd7b108fd18a2cb2147ea32c64a79f51e9`. Bản cập nhật báo cáo này chưa nằm trong commit trên; SHA nộp cuối cần lấy sau khi commit/push.
- **Challenge ID:** `day13-k4-l3b-monitoring-llmops-v1`.
- **Project Langfuse cá nhân:** `day13-k4-l3b-2A202602673`.

Báo cáo dựa trên ảnh evidence, source/config và dữ liệu chạy ngày 30/09/2026. Kết quả trong ảnh và log hiện tại thuộc các thời điểm khác nhau. Chưa có output baseline trước khi hoàn thiện source để so sánh before/after.

## 2. Evidence index

| Evidence | Đường dẫn | Nội dung thực tế và giới hạn |
|---|---|---|
| Pytest | [01-pytest.png](evidence/01-pytest.png) | 25 passed; lần cuối trong ảnh 1,62 giây |
| Log validator | [02-log-validator.png](evidence/02-log-validator.png) | 100/100 trên 155 records, 71 correlation IDs, 0 potential PII leaks |
| Dashboard validator | [03-dashboard-validator.png](evidence/03-dashboard-validator.png) | Contract hợp lệ 6/6 panel |
| Structured log | [04-structured-log.png](evidence/04-structured-log.png) | JSON log với các giá trị đã redacted |
| PII tests | [05-pii-redaction.png](evidence/05-pii-redaction.png) | Ảnh source tests; chưa phải input/output runtime đầy đủ |
| Trace list | [06-trace-list.png](evidence/06-trace-list.png) | Project cá nhân, 10 root và 20 child observations |
| Trace waterfall | [07-trace-waterfall.png](evidence/07-trace-waterfall.png) | Root, retrieval, generation, token và cost |
| Trace detail | [08-trace-metadata.png](evidence/08-trace-metadata.png) | Retrieval trả 1 document; chưa hiện đầy đủ metadata correlation/prompt |
| Prompt versions | [09-prompt-versions.png](evidence/09-prompt-versions.png) | v1 baseline/production, v2 candidate/latest |
| Prompt rollback | Chưa có file evidence | Chưa chứng minh chuyển production sang v2 rồi về v1 |
| Dashboard config | [11-dashboard-overview.png](evidence/11-dashboard-overview.png) | Ảnh YAML, chưa phải dashboard runtime |
| Incident timing | [12-incident-metric.png](evidence/12-incident-metric.png) | Trace timeline khoảng 2,65 giây; chưa phải biểu đồ metric |
| Ảnh mang tên incident log | [13-incident-log.png](evidence/13-incident-log.png) | Thực tế là observations, không có JSON log/correlation ID |
| Incident observations | [14-incident-trace.png](evidence/14-incident-trace.png) | Danh sách retrieval/generation; chưa hiện trace ID và correlation ID |
| Tổng hợp dữ liệu chạy | [15-log-summary.json](evidence/15-log-summary.json) | Tổng hợp 187 records và các event trong khoảng incident; không chứa hội thoại |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả có evidence | Nhận xét |
|---|---|---|---|
| `validate_logs.py` | Chưa lưu | 100/100 | Ảnh kiểm tra 155 records, không mặc nhiên áp dụng cho log mới hơn |
| `validate_dashboard.py` | Chưa lưu | 6/6 | Xác nhận cấu hình, chưa xác nhận dashboard runtime |
| `pytest` | Chưa lưu | 25 passed trong 1,62 s | Theo ảnh 01 |
| Số traces | Chưa lưu | 10 root observations, 20 children | Không tính 30 observations thành 30 traces; chưa kiểm tra metadata từng trace |
| PII leak | Chưa lưu | 0 potential leaks | Theo validator trên 155 records và phạm vi rule của validator |
| Latency P95 / TTFT P95 | Chưa lưu | 2.651,75 ms / 50 ms | Tính từ 86 response trong log hiện tại |
| Retrieval success rate | Chưa lưu | 100% (86/86) | Theo `tool_success`, không đồng nghĩa chất lượng retrieval đạt 100% |

Log hiện tại trải từ **03:58:37–13:20:27 UTC ngày 30/09/2026** (10:58:37–20:20:27 giờ Việt Nam), gồm 187 records: 86 `request_received`, 86 `response_sent`, 14 `app_started`, 1 `incident_enabled`, không có `request_failed`. Đây là dữ liệu tích lũy nhiều lần chạy và có practice incident, không phải baseline độc lập hay cửa sổ dashboard 60 phút.

P95 được tính bằng nội suy tuyến tính ở vị trí `(n - 1) × 0,95` trên dãy tăng dần. Tổng input/output là **2.948 / 10.933 tokens**, tổng cost ghi nhận **0,172839 USD**, quality proxy trung bình **0,8779**. Ứng dụng dùng `FakeLLM`; cost là ước tính từ công thức trong [agent.py](../app/agent.py), không phải hóa đơn nhà cung cấp. Quality là heuristic, chưa phải đánh giá độc lập độ đúng của câu trả lời.

## 4. Logging và PII

[middleware.py](../app/middleware.py) nhận `x-request-id`; nếu thiếu thì sinh `req-` kèm 8 ký tự hex từ UUID. Middleware xóa context ở đầu request, bind `correlation_id`, đặt ID vào `request.state` và trả lại qua response header. Agent nhận cùng ID để đưa vào metadata trace.

[main.py](../app/main.py) ghi `request_received`, `response_sent` hoặc `request_failed`, kèm `model`, `env`, `feature`, `session_id`, `user_id_hash`, `correlation_id`. Response log bổ sung latency, TTFT, input/output tokens, cost, quality, `tool_name` và `tool_success`. Timestamp UTC nằm trong trường `ts`.

[pii.py](../app/pii.py) có rule cho email, điện thoại Việt Nam, CCCD, thẻ thanh toán, hộ chiếu, địa chỉ và ngày sinh có nhãn. `summarize_text` scrub trước khi cắt preview; user ID được băm SHA-256 và lấy 12 ký tự đầu. Trong [logging_config.py](../app/logging_config.py), `scrub_event` đứng trước processor ghi file và JSON renderer, scrub `event` và các string ở tầng đầu của `payload`.

Evidence 04 thể hiện `[REDACTED_EMAIL]`, `[REDACTED_PHONE_VN]`, `[REDACTED_CREDIT_CARD]`; evidence 02 ghi nhận 0 potential leaks. Processor chưa scrub đệ quy mọi cấu trúc lồng nhau hoặc mọi metadata, nên kết quả validator không bảo đảm tuyệt đối cho mọi input.

## 5. Tracing và prompt versioning

Ảnh 06 hiển thị đúng project cá nhân, 10 `lab-agent-run`, 10 `retrieval` và 10 `generation`. Source tắt capture input/output ở root; retrieval và generation được tạo trong root. Generation ghi model, usage và cost; query preview được scrub và user ID được hash.

Ảnh 07 có trace ID **`a35555f94ee575f73fbe94b0a011f03a`**, root khoảng 0,16 s, retrieval khoảng 0,01 s, generation khoảng 0,15 s; tổng 206 tokens và 0,002574 USD. Ảnh xác nhận cấu trúc cha-con nhưng chưa đủ để gắn trace với prompt version cụ thể.

Để nối log với trace, lấy `correlation_id` trong JSON log rồi tìm metadata cùng giá trị trên Langfuse. Source đã truyền trường này, nhưng ảnh 08 đang mở retrieval output nên chưa chứng minh trực tiếp phép nối.

| Prompt | Version | Labels trong ảnh 09 | Trace ID theo version |
|---|---|---|---|
| `day13-chat` | 1 | `baseline`, `production` | Chưa có ID đọc được kèm version; ảnh 12 có generation gắn v1 |
| `day13-chat` | 2 | `candidate`, `latest` | Chưa có evidence trace chạy v2 |

Version 2 yêu cầu tối đa ba bullet ngắn. [prompt_management.py](../app/prompt_management.py) lấy prompt theo tên và label (mặc định `production`), cache 60 giây; nếu không lấy được thì dùng `local-v1` và ghi nguồn fallback.

Quy trình promote là chuyển label `production` sang v2, chờ cache hết hạn hoặc khởi động lại rồi kiểm tra trace mới. Rollback chuyển label về v1 và kiểm tra version trên request tiếp theo. **Ảnh hiện có chỉ chứng minh production đang ở v1, chưa chứng minh đã promote/rollback.**

## 6. Dashboard, SLO và alerts

[dashboard.yaml](../config/dashboard.yaml) đặt cửa sổ 60 phút, refresh 30 giây và sáu panel:

| Panel | Chỉ số / đơn vị | Threshold trong config |
|---|---|---|
| Latency / TTFT | P50/P95/P99 latency, P95 TTFT; ms | Latency P95 ≤ 3.000 ms |
| Traffic | Count, requests/phút | ≥ 1 request/phút |
| Errors / retrieval | Error rate, nhóm lỗi, tool success; % | Error rate ≤ 2% |
| Cost | Tổng và tổng theo phút; USD | Tổng ≤ 2,5 USD |
| Tokens | Tổng input/output; tokens | ≤ 50.000 |
| Quality | Điểm heuristic trung bình, 0–1 | ≥ 0,75 |

Validator đạt 6/6 nhưng ảnh 11 mới là YAML, chưa có evidence dashboard render đủ sáu panel với dữ liệu. Ngưỡng cost của panel theo cửa sổ hiển thị cần phân biệt với guardrail cost theo ngày trong SLO.

[slo.yaml](../config/slo.yaml) đặt mục tiêu **99,5% request trả kết quả trong ≤ 3.000 ms trong 28 ngày**, error budget **0,5%**. Với 10.000 requests, cho phép tối đa 50 requests lỗi hoặc quá chậm. Ngưỡng 3 giây có khoảng đệm so với trace thông thường khoảng 0,16 giây, nhưng chưa được hiệu chỉnh bằng workload production.

Mẫu log có 1/86 response vượt 3.000 ms, tức tỷ lệ đạt **98,84%**, không đạt **1,16%**. Đây chỉ là tập chạy thử, chưa đủ kết luận SLO 28 ngày. Đợt `rag_slow` khoảng 2,65 giây vẫn dưới ngưỡng 3 giây: cần theo dõi thay đổi so với baseline bên cạnh ngưỡng tuyệt đối.

**Alerts chưa hoàn thiện:** [alert_rules.yaml](../config/alert_rules.yaml) còn ba mục TODO; [alerts.md](../docs/alerts.md) còn runbook trống. Đề xuất ba alert: P95 > 3.000 ms trong 5 phút; error rate > 2% trong 5 phút; quality trung bình < 0,75 trong 10 phút với đủ mẫu. Owner dự kiến `student-2A202602673`, kênh Slack dự kiến `#k4-l3b-alerts`. Đây là đề xuất, chưa phải cấu hình đã triển khai.

## 7. Điều tra challenge và practice incident

Challenge ID ở mục 1. Dữ liệu kiểm chứng được bên dưới là **practice scenario `rag_slow`**; chưa có bằng chứng thực thi challenge chính thức nên không coi hai phần là cùng một sự cố.

- **Khoảng điều tra:** 13:20:14–13:20:27 UTC ngày 30/09/2026, tức 20:20:14–20:20:27 giờ Việt Nam.
- **Dấu mốc:** log ghi `incident_enabled` cho `rag_slow` lúc `13:20:14.187030Z`.
- **Triệu chứng:** năm response sau đó có latency 2.654, 2.651, 2.653, 2.652 và 2.652 ms; TTFT 50–52 ms. Ảnh 12 có root khoảng 2,65 s, generation khoảng 0,15 s.
- **Log đại diện:** `response_sent` lúc `2026-09-30T13:20:27.826142Z`, `correlation_id=req-d343d653`, `latency_ms=2652`, `ttft_ms=50`, `tool_success=true`. Các trường được lưu trong [tổng hợp log](evidence/15-log-summary.json).
- **Trace và span:** timeline phù hợp với retrieval chiếm phần lớn thời gian, nhưng ảnh 12/14 chưa hiện trace ID/correlation ID để nối chắc chắn với request đại diện. Ảnh 13 là observations lúc 20:07:02, khác khoảng incident trên và không phải JSON log.
- **Root cause có căn cứ:** practice `rag_slow` bật nhánh `time.sleep(2.5)` trong [mock_rag.py](../app/mock_rag.py), phù hợp với log bật scenario và độ trễ 2,65 s. Chưa đủ evidence kết luận root cause của challenge chính thức.
- **Fix action đề xuất:** gọi `POST /incidents/rag_slow/disable`, chạy lại cùng workload rồi so sánh latency và retrieval span. Log hiện có chưa ghi `incident_disabled` hay workload sau khôi phục.
- **Preventive measure:** theo dõi latency retrieval và biến động so với baseline, đặt timeout khi dùng retrieval service thật, lưu metric/log/trace có cùng correlation ID cho từng lần điều tra.

## 8. Giải thích và tự đánh giá

**Quyết định kỹ thuật:** dùng cùng correlation ID xuyên suốt request, log và trace để truy ngược request cụ thể. Scrub trước khi ghi file và chỉ lưu preview giúp giảm dữ liệu nhạy cảm mà vẫn giữ thông tin điều tra.

**Blocker khi đối chiếu:** `.venv` đang trỏ tới Python 3.11 không còn tồn tại nên lệnh chạy lại validators không khởi động được. Báo cáo giữ kết quả test/validator theo ảnh đã lưu; JSON log được tổng hợp bằng PowerShell. Cần tạo lại môi trường Python, cài requirements và chạy tests/validators trên commit nộp cuối; chưa xác nhận bước này hoàn tất.

**Metrics → Logs → Traces:** metrics xác định loại bất thường và khoảng thời gian; log xác định request bị ảnh hưởng bằng correlation ID; trace cùng ID phân tách thời gian giữa retrieval, prompt và generation. Chỉ khi nối đúng ID mới quy kết span cho request cụ thể; timestamp gần nhau chỉ là dấu hiệu để tìm kiếm.

**Vai trò trong vận hành LLM:** prompt version giúp tái hiện hành vi; label production hỗ trợ promote/rollback; token/cost giúp phát hiện chi phí tăng; SLO và error budget diễn đạt mục tiêu phục vụ người dùng cùng mức không đạt chấp nhận được. Với mock workload, cost/quality chủ yếu minh họa cơ chế đo.

**Điều học được:** vượt validator chưa đồng nghĩa hoàn thành observability end-to-end. Ảnh cấu hình không thay dashboard có dữ liệu; danh sách observations không thay trace nối được với log; production trên v1 không tự chứng minh rollback.

**Hạn chế:** chưa có baseline before/after; thiếu dashboard runtime, metadata trace đầy đủ, trace theo từng prompt version và evidence rollback; alerts/runbooks còn TODO; incident chưa nối đủ metric → log → trace; chưa chứng minh hoàn thành challenge chính thức hoặc phục hồi practice incident. Chưa kiểm chứng SLO đủ 28 ngày hay tái hiện trên môi trường mới.

## 9. Checklist trước khi nộp

- [x] Có ảnh kết quả pytest và hai validators.
- [x] Có ảnh project Langfuse cá nhân với 10 root observations và prompt v1/v2.
- [x] Các link evidence trỏ tới file hiện có; phần thiếu được ghi rõ.
- [ ] Chạy lại tests/validators trên môi trường hoạt động và commit nộp cuối.
- [ ] Bổ sung dashboard runtime, metadata correlation/prompt và evidence promote/rollback.
- [ ] Hoàn thiện ba alerts và runbooks.
- [ ] Bổ sung investigation challenge chính thức và nối đúng metric → log → trace.
- [ ] Kiểm tra toàn bộ artifact không lộ secret hoặc PII trước khi push.
- [ ] Commit/push báo cáo và evidence; lấy SHA cuối đã có trên remote.
- [ ] Nộp URL repository và SHA cuối trên LMS/Codelabs.
