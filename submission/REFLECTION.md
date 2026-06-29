# Day 23 Lab Reflection

**Sinh viên:** Vũ Quang Vinh
**Ngày nộp:** 2026-06-29
**Lab repo URL:** https://github.com/Chipzgk/Day23-Track2-2A202600935-VuQuangVinh

---

## 1. Hardware + setup

Môi trường: Windows 11, Docker Desktop, Python 3.12 (virtual env ai_env).
Stack 7 services khởi động thành công sau khi fix Alertmanager — lỗi ban đầu do
alertmanager.yml dùng api_url thay vì api_url_file để đọc webhook URL từ file,
dẫn đến lỗi "unsupported scheme for URL". Fix bằng cách tách webhook ra file riêng
slack_webhook_url.txt và thêm vào .gitignore để không lộ secret.

Lỗi thứ hai: Grafana datasource UID trong dashboard JSON là "prometheus" nhưng
Grafana assign UID thực là "PBFA97CFB590B2093" — phải replace hàng loạt trong
3 file dashboard JSON để panels hiện data.

---

## 2. Track 02 — Dashboards & Alerts

### 6 panels thiết yếu

Xem submission/screenshots/dashboard-overview.png.

### Burn-rate panel

Xem submission/screenshots/slo-burn-rate.png.

### Alert fire + resolve

| Thời điểm | Sự kiện | Bằng chứng |
|---|---|---|
| T0 | Kill day23-app bằng docker compose stop app | — |
| T0+90s | ServiceDown chuyển PENDING sang FIRING | alertmanager-firing.png |
| T0+2m | Slack nhận fire message | slack-firing.png |
| T1 | Restore app bằng docker compose start app | — |
| T1+2m | Alert resolve, Slack nhận resolve message | slack-resolved.png |

### Điều bất ngờ về Prometheus/Grafana

Datasource UID trong Grafana dashboard JSON phải khớp chính xác với UID được
Grafana assign khi provision — không phải tên "prometheus" mà là chuỗi hash
"PBFA97CFB590B2093". Nếu không khớp, toàn bộ panels hiện "No data" mà không
có error message rõ ràng, rất khó debug nếu không biết kiểm tra API datasources.

---

## 3. Track 03 — Tracing & Logs

### Trace screenshot từ Jaeger

Xem submission/screenshots/jaeger-trace.png — trace 6fa92b1 với 4 spans:
predict → embed-text → vector-search → generate-tokens, Total Spans 4, Depth 2.

### Log line correlate với trace

Log line có trace_id liên kết với Jaeger:

    {"model": "llama3-mock", "input_tokens": 4, "output_tokens": 16, "quality": 0.867,
    "duration_seconds": 0.0853, "trace_id": "974cab05cf8f05a317476adb34e24582",
    "event": "prediction served", "level": "info", "timestamp": "2026-06-29T09:29:35.293308Z"}

trace_id 974cab05cf8f05a317476adb34e24582 liên kết log này với span tương ứng
trong Jaeger, cho phép jump từ log sang trace trong một click.

### Tail-sampling math

Service chạy khoảng 19 req/s (đo từ locust). Policy giữ:
- 100% error traces (keep-errors policy)
- 100% traces trên 2000ms (keep-slow policy, khoảng 1% tổng dựa trên latency distribution)
- 1% remaining healthy traces (probabilistic-1pct policy)

Ước tính: 19 req/s x 0.01 = 0.19 healthy traces/s được giữ lại.
Cộng với error traces, tổng khoảng 0.2 traces/s — Jaeger storage giảm 99% so với
giữ tất cả, nhưng vẫn đảm bảo mọi lỗi đều được capture đầy đủ.

---

## 4. Track 04 — Drift Detection

### PSI scores

Kết quả từ 04-drift-detection/reports/drift-summary.json:

    prompt_length:    PSI=3.461  KL=1.798  KS=0.702  drift=yes
    embedding_norm:   PSI=0.019  KL=0.032  KS=0.052  drift=no
    response_length:  PSI=0.016  KL=0.018  KS=0.056  drift=no
    response_quality: PSI=8.849  KL=13.501 KS=0.941  drift=yes

### Test nào phù hợp feature nào?

- prompt_length (continuous, shift lớn về mean): KS test phù hợp nhất vì nhạy
  với shift về location và shape, không cần binning. PSI tốt để monitor production
  vì dễ interpret — PSI > 0.2 là significant drift.

- embedding_norm (continuous, phân phối hẹp, ổn định): KS test đủ dùng.
  PSI = 0.019 xác nhận không có drift — cả 3 tests đồng thuận.

- response_length (continuous, variance lớn): KL divergence phù hợp khi có prior
  về distribution dự kiến. KS cho kết quả tương tự trong trường hợp này.

- response_quality (beta distribution, shift mạnh từ high sang low quality):
  PSI và KL đều bắt được rõ ràng (PSI=8.849, KL=13.501). KS stat=0.941 gần 1.0
  cho thấy hai distributions gần như không overlap — đây là failure mode nguy hiểm
  nhất cần alert ngay lập tức trong production.

---

## 5. Track 05 — Cross-Day Integration

### Prior-day metric nào khó expose nhất?

Day 19 (Qdrant vector store) sẽ khó nhất vì Qdrant expose metrics qua REST API
không phải Prometheus format chuẩn — cần viết custom exporter scrape endpoint
và convert sang Prometheus exposition format. Khác với Day 20 (llama.cpp) đã có
sẵn stub script emit metrics trực tiếp qua prometheus_client, Day 19 cần thêm
logic parse JSON response từ Qdrant và map sang gauge/counter metrics phù hợp,
đồng thời xử lý authentication nếu Qdrant được bảo vệ.

---

## 6. Thay đổi quan trọng nhất

Thay đổi có tác động lớn nhất là sửa tracer.start_span("predict") thành
tracer.start_as_current_span("predict") và bọc toàn bộ logic xử lý request
bên trong with block. Đây không chỉ là syntax fix — đây là sự khác biệt cơ bản
giữa "tạo span" và "set span làm current context trong thread". Khi dùng start_span,
các child spans (embed-text, vector-search, generate-tokens) không biết parent là
ai, nên chúng xuất hiện trong Jaeger như các traces độc lập thay vì một cây spans
liên kết. Kết quả: không thể trace được luồng request từ đầu đến cuối — Jaeger
hiển thị Total Spans 1 thay vì 4.

Điều này liên kết trực tiếp với khái niệm context propagation trong deck section 7
về OTel GenAI. OTel sử dụng thread-local context để biết span nào đang active.
start_as_current_span tự động attach span mới vào context hiện tại và cleanup khi
with block kết thúc. Trong production AI system, đây là nền tảng của observability:
không có distributed trace đúng, bạn không thể biết 197ms trong generate-tokens là
do model chậm hay do I/O bottleneck — và không thể tối ưu đúng chỗ. Đây là ví dụ
điển hình của "works but not useful" — service vẫn chạy, metrics vẫn emit, nhưng
engineer on-call không có đủ thông tin để debug khi incident xảy ra lúc 2 giờ sáng.