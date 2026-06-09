# Grafana

Tài liệu tham khảo: [Grafana Docs](https://grafana.com/docs/grafana/latest)

## Grafana là gì?
Grafana là một nền tảng mã nguồn mở hàng đầu dùng để phân tích và trực quan hóa dữ liệu (Visualization). Nó đóng vai trò là "bảng điều khiển" (Dashboard) trung tâm cho toàn bộ hệ thống giám sát của bạn.

## Các Vai Trò Chính

1.  **Giao diện Trực quan hóa (Visualization Front-end):**
    Grafana bản thân nó **không** lưu trữ bất kỳ dữ liệu (Metrics hay Logs) nào. Nó chỉ lấy dữ liệu từ các kho lưu trữ (Data Sources) để chuyển đổi những dòng dữ liệu thô kệch, những con số phức tạp thành các biểu đồ (Graphs, Charts, Heatmaps) đẹp mắt, dễ đọc và dễ hiểu.

2.  **Kết nối đa dạng (Multiple Data Sources):**
    Grafana cực kỳ linh hoạt. Trên một Dashboard duy nhất, bạn có thể thiết lập:
    *   Biểu đồ CPU lấy từ **Prometheus**.
    *   Màn hình theo dõi Logs lấy từ **Loki**.
    *   Bản đồ theo dõi lỗi (Traces) lấy từ **Jaeger** / **Tempo**.
    *   Thậm chí hiển thị bảng danh sách khách hàng từ cơ sở dữ liệu **MySQL/PostgreSQL**.

3.  **Quản lý Cảnh báo (Alerting):**
    Ngoài việc dùng Alertmanager của Prometheus, bản thân Grafana cũng tích hợp sẵn tính năng Alerting. Bạn có thể cài đặt trực tiếp trên biểu đồ: "Nếu đường line màu đỏ này vượt quá ngưỡng này trong 5 phút, hãy gửi tin nhắn cho tôi qua Slack".

## Cách Dùng Grafana Hiệu Quả Cho Người Mới
*   **Không cần xây lại từ đầu:** Cộng đồng Grafana cung cấp hàng ngàn Dashboard có sẵn tại trang chủ. Nếu bạn muốn giám sát Kubernetes hay Nginx, chỉ cần tải Dashboard JSON mẫu về (import) và sử dụng ngay.
*   **Biến số (Variables):** Sử dụng variables ở thanh trên cùng của Dashboard để tạo ra biểu đồ động. Thay vì làm 10 dashboard cho 10 server, hãy làm 1 dashboard có menu thả xuống (dropdown) để chọn xem từng server.

> [!TIP]
> Trong bộ ba phổ biến PLG (Prometheus, Loki, Grafana), Grafana đóng vai trò như cửa sổ giao tiếp cuối cùng giữa hệ thống và kỹ sư. Nó cho phép toàn team (từ Dev đến Ops) nhìn chung một bức tranh thời gian thực về sức khỏe ứng dụng.
