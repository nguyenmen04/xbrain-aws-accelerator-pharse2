# Prometheus

Tài liệu tham khảo: [Prometheus Docs](https://prometheus.io/docs)

## Prometheus là gì?
Prometheus là một hệ thống giám sát và lưu trữ dữ liệu chuỗi thời gian (Time-series database - TSDB) mã nguồn mở rất mạnh mẽ, đặc biệt phổ biến trong môi trường Kubernetes.

## Đặc điểm Cốt Lõi

1.  **Dữ liệu Chuỗi Thời Gian (Time-series data):**
    Prometheus chuyên lưu trữ các con số (Metrics) đi kèm với thời gian (Timestamp) và các nhãn (Labels/Key-Value pairs). 
    *Ví dụ:* `http_requests_total{method="GET", status="200"} 1056` (Vào lúc này, tổng số request GET thành công là 1056).

2.  **Cơ chế Pull (Kéo):**
    Đây là sự khác biệt lớn nhất của Prometheus. Thay vì các ứng dụng chủ động gửi (push) metrics tới server, Prometheus sẽ được cấu hình để theo chu kỳ (ví dụ: mỗi 15 giây) tự động đi "kéo" (scrape) dữ liệu từ một đường dẫn `/metrics` trên ứng dụng của bạn.
    *   *Lợi ích:* Nếu ứng dụng bị chết hoặc treo, nó không thể gửi dữ liệu. Với Pull model, Prometheus ngay lập tức biết rằng ứng dụng đang "không thể tiếp cận" (unreachable) và cảnh báo.

3.  **Ngôn ngữ truy vấn PromQL (Prometheus Query Language):**
    Một ngôn ngữ linh hoạt và vô cùng mạnh mẽ cho phép tính toán trực tiếp trên dữ liệu chuỗi thời gian.
    *Ví dụ:* Tính tốc độ request lỗi mỗi giây trong 5 phút qua: `rate(http_requests_total{status="500"}[5m])`

4.  **Hệ sinh thái:**
    *   **Exporters:** Vì Prometheus cần ứng dụng phải mở endpoint (cổng) `/metrics`, các "Exporters" ra đời như người phiên dịch. (Ví dụ: *Node Exporter* giúp thu thập thông số CPU/RAM của server Linux và phơi bày nó dưới dạng `/metrics` cho Prometheus kéo).
    *   **Alertmanager:** Một thành phần đi kèm Prometheus giúp quản lý các cảnh báo. Khi một truy vấn PromQL trả về vi phạm (ví dụ CPU > 90%), Alertmanager sẽ gộp các thông báo lại và gửi qua Slack/Email/PagerDuty.

> [!NOTE]
> Prometheus **không** được thiết kế để lưu Traces hay Logs. Nó chỉ chuyên dùng cho **Metrics** (số liệu đếm được). Việc vẽ biểu đồ từ dữ liệu của Prometheus thường được giao cho **Grafana**.
