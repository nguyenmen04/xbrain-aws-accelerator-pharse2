# Flagger - Giải pháp tự động hóa với Service Mesh

Tài liệu tham khảo: [Flagger Docs](https://flagger.app/)

---

## 1. Flagger là gì?

Tương tự như Argo Rollouts, **Flagger** là một công cụ giúp tự động hóa quá trình Progressive Delivery (Canary, A/B Testing, Blue/Green). Tuy nhiên, cách tiếp cận và đối tượng nhắm đến của Flagger có sự khác biệt.

Flagger tỏa sáng nhất khi bạn **đã có sẵn một hệ thống Service Mesh** (như Istio, Linkerd) hoặc các Ingress Controller nâng cao (NGINX, Contour, Gloo).

---

## 2. Cách Flagger hoạt động

Flagger quản lý quá trình Canary bằng cách kết hợp chặt chẽ với các công cụ điều phối traffic:

*   **Không cần đổi Resource:** Thay vì tạo resource mới để thay thế Deployment như Argo Rollouts, Flagger nhận một K8s Deployment tiêu chuẩn bình thường.
*   **Tự động sinh cấu hình:** Flagger tự động tạo ra các resource phụ trợ (như Canary Deployment, Primary Deployment, Virtual Services cho Istio).
*   **Điều hướng traffic đỉnh cao (Routing):** Nhờ sức mạnh của Service Mesh, Flagger có thể điều hướng chính xác từng % nhỏ nhất của request. Thậm chí có thể phân loại người dùng (A/B testing) dựa trên HTTP Header hoặc Cookie thay vì chỉ chia tỷ lệ ngẫu nhiên.

---

## 3. Metrics-driven Promotion (Thăng cấp dựa trên chỉ số)

Đây là trái tim của Flagger:

1.  **Đo lường tự động:** Khi mở traffic cho bản Canary, Flagger liên tục gọi đến Prometheus (hoặc Datadog, New Relic) để kiểm tra các chỉ số cốt lõi: `success-rate` (tỷ lệ thành công) và `latency` (độ trễ).
2.  **Webhooks (Kiểm tra nghiệp vụ):** Flagger còn cho phép gọi các webhook bên ngoài để chạy Load Test hoặc Integration Test trước khi quyết định mở traffic.
3.  **Tự động thăng cấp / Loại bỏ:**
    *   **Promote:** Nếu mọi chỉ số đạt chuẩn trong thời gian quy định, Flagger tự động tăng traffic và cuối cùng thăng cấp bản Canary thành bản chính thức (Primary).
    *   **Reject:** Nếu tỷ lệ lỗi chạm ngưỡng cho phép, Flagger ngay lập tức ngắt traffic tới bản Canary và báo cáo lỗi (vào Slack/Teams).

> [!TIP]
> **Nên chọn Argo Rollouts hay Flagger?**
> *   Chọn **Argo Rollouts** nếu bạn muốn mọi thứ đơn giản, quản lý vòng đời ứng dụng ngay trong Kubernetes Controller.
> *   Chọn **Flagger** nếu hệ thống của bạn yêu cầu điều phối traffic phức tạp, tinh vi và đã ứng dụng Service Mesh.
