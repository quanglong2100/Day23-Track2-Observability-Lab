# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Trần Quang Long
**Submission date:** _YYYY-MM-DD_
**Lab repo URL:** _public GitHub URL_

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
```text
Docker:        OK  (29.3.0-1)
Compose v2:    OK  (2.40.3)
RAM available: 7.76 GB (OK)
Ports free:    OK
Report written: /workspaces/Day23-Track2-Observability-Lab/00-setup/setup-report.json
```
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app`         | screenshot `alertmanager-firing.png` |
| _T0+90s_ | `ServiceDown` fired   | screenshot `slack-firing.png` |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | screenshot `slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

Tôi rất bất ngờ với khái niệm "Dashboards-as-code" (provisioning). Thay vì phải vào UI của Grafana, click tạo từng biểu đồ rồi export/import file JSON thủ công, toàn bộ cấu hình, Data source và Dashboard được nạp tự động thông qua các file YAML ngay lúc container khởi động. Điều này giúp việc quản lý hạ tầng trở nên nhất quán và dễ dàng đưa vào Git/CI-CD.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```json
{"level": "info", "event": "prediction served", "model": "llama3-mock", "input_tokens": 5, "output_tokens": 42, "duration_seconds": 0.168, "trace_id": "e6248aaae0b8ab89bc0beaef980969d1", "timestamp": "2026-05-13T13:35:00Z"}
```
*(Log line chứa `trace_id` giúp liên kết trực tiếp một sự kiện log cụ thể với toàn bộ vòng đời (span) của request trên Jaeger).*

### Tail-sampling math

If your service produced N traces/sec, what fraction did the policy keep? Show the calculation.

Giả sử service sinh ra `N` traces/sec. Tỷ lệ phân bổ: 1% lỗi, 1% siêu chậm (>2s), và 98% bình thường.
Theo policy trong `otel-config.yaml`:
- Giữ 100% lỗi: `N * 0.01 * 1.0`
- Giữ 100% chậm: `N * 0.01 * 1.0`
- Giữ 1% bình thường: `N * 0.98 * 0.01`
=> Tỉ lệ Sampled = `0.01 + 0.01 + 0.0098 = 0.0298` (Khoảng **3%**).

**Kết luận:** Hệ thống giữ lại 100% các trace có giá trị debug cao (lỗi, chậm) nhưng tiết kiệm được tới **97%** dung lượng lưu trữ. Trong quá trình làm Lab, tôi thử query 1 trace id bình thường và nhận lỗi `404 trace not found`, đây chính là minh chứng hoàn hảo cho việc policy Tail-Sampling đã hoạt động và drop trace đó.


---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 0.4352,
    "kl": 0.4812,
    "ks_stat": 0.382,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.041,
    "kl": 0.035,
    "ks_stat": 0.021,
    "ks_pvalue": 0.95,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.032,
    "kl": 0.028,
    "ks_stat": 0.025,
    "ks_pvalue": 0.88,
    "drift": "no"
  },
  "response_quality": {
    "psi": 0.8215,
    "kl": 1.152,
    "ks_stat": 0.58,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```


### Which test fits which feature?

For each of `prompt_length`, `embedding_norm`, `response_length`, `response_quality`, name the test (PSI / KL / KS / MMD) you'd choose in production and why.
- **PSI (Population Stability Index)**: Phù hợp nhất cho Categorical features hoặc các feature đã được chia bin. Rất dễ diễn giải cho team Business vì nó đo tỷ lệ phần trăm phân bổ thay đổi.
- **KS (Kolmogorov-Smirnov)**: Phù hợp cho 1D Continuous features (như `prompt_length`, `response_length`). Nó đo khoảng cách xa nhất giữa 2 hàm phân phối tích lũy (CDF) để phát hiện sự thay đổi hình dáng phân phối.
- **KL Divergence**: Phù hợp khi cần so sánh 2 phân phối xác suất (ví dụ: topic distribution).
- **MMD (Maximum Mean Discrepancy)**: Phù hợp nhất cho high-dimensional data (như raw `embedding_norm` hoặc vector embeddings) vì nó dùng "kernel trick" để đo khoảng cách giữa 2 tập dữ liệu đa chiều.
---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

Việc expose metrics của Day 20 (`llama.cpp`) là khó nhất. Lý do là `llama.cpp` HTTP server không hỗ trợ xuất định dạng `/metrics` chuẩn của Prometheus ra hộp (out-of-the-box). Nó đòi hỏi phải tạo thêm một stub/sidecar bằng Python để scrape các thông số HTTP của nó và dịch sang dạng Prom `Counter` và `Gauge` (như tokens/sec hay queue depth).

---

## 6. The single change that mattered most

> **Grader reads this closest.** What one thing about your stack design — a metric you added, a label you dropped, a panel you reorganized, an alert threshold you tuned — made the biggest difference between "works" and "useful"? Write 1-2 paragraphs. Connect it to a concept from the deck.
Sự thay đổi tạo ra tác động lớn nhất trong trải nghiệm Lab này không nằm ở các biểu đồ, mà nằm ở việc cấu hình **Routing & Alerting Pipeline (Alertmanager + Slack Webhook)** và tư duy **"Vibe Coding" vs "Robust Engineering"**. Ban đầu, hệ thống bị crash liên tục (`unsupported scheme ""`) vì Docker Compose không tự động expand biến môi trường vào file `alertmanager.yml` trong môi trường Codespaces. Thay vì chỉ vẽ một biểu đồ lỗi đẹp đẽ, tôi nhận ra Observability không có ý nghĩa gì nếu "đường ống báo động" (Alert Pipeline) bị vỡ một cách im lặng (silent failure). 

Việc phải hardcode URL hoặc sửa lại luồng truyền biến môi trường đã dạy tôi một bài học sâu sắc về bài giảng §9 (Postmortems & Runbooks): **Một hệ thống Observability chuẩn Production phải đảm bảo tính tin cậy ngay từ tầng cấu hình hạ tầng**. Hơn nữa, việc gặp lỗi với chuỗi string matching tĩnh (`grep -q '"database":"ok"'`) khi check health của Grafana càng củng cố nguyên tắc: Hệ thống giám sát phải đo lường dựa trên các tín hiệu chuẩn (HTTP Status Code 200) thay vì dựa vào format chuỗi dễ gãy (brittle). Khi nhận được tin nhắn báo cháy 🔥 và khắc phục ✅ trên Slack, đó là lúc hệ thống AI không còn là một hộp đen, mà thực sự trở thành một sản phẩm có thể vận hành và bảo trì được vào lúc 2 giờ sáng.
