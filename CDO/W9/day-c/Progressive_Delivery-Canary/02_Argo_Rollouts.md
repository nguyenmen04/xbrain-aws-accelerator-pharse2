# Argo Rollouts & Khái niệm Analysis

Tài liệu tham khảo: [Argo Rollouts Docs](https://argoproj.github.io/argo-rollouts/)

---

## 1. Argo Rollouts là gì?

Argo Rollouts là một Controller mạnh mẽ trong Kubernetes, cung cấp một resource mới mang tên `Rollout` nhằm mục đích thay thế cho resource `Deployment` mặc định của K8s.

Trong khi K8s Deployment chỉ hỗ trợ cập nhật kiểu `RollingUpdate` (thay thế dần các Pod nhưng không có khả năng kiểm soát traffic nhạy bén), Argo Rollouts sinh ra để mang đến các chiến lược nâng cao như **Canary** và **Blue/Green**.

---

## 2. Khái niệm "Analysis" (Điểm làm nên sự khác biệt)

Để Progressive Delivery thực sự hiệu quả, hệ thống phải biết tự động đánh giá: *"Lúc nào nên tăng traffic, lúc nào nên Rollback?"*. Trong Argo Rollouts, cơ chế này gọi là **Analysis** (Phân tích).

Thay vì để con người phải ngồi nhìn Dashboard quan sát thủ công, Analysis cho phép Argo tự động "hỏi" các hệ thống giám sát (Prometheus, Datadog...) để đưa ra quyết định.

### Các thành phần chính của Analysis:

*   **AnalysisTemplate:**
    *   Là bản "hợp đồng đo lường" định nghĩa sẵn cách lấy chỉ số.
    *   Ví dụ: *Query Prometheus để xem tỷ lệ HTTP 500 có < 1% không, hay đo xem độ trễ có vượt quá 200ms hay không.*
*   **AnalysisRun:**
    *   Là thực thể chạy thực tế của template trong quá trình Canary diễn ra.
    *   Trạng thái **Successful (Thành công)**: Các chỉ số đều tốt -> Argo cho phép xả bước traffic tiếp theo.
    *   Trạng thái **Failed (Thất bại)**: Lỗi vượt mức cho phép -> Argo lập tức huỷ bỏ bản cập nhật và Rollback về bản cũ.
    *   Trạng thái **Inconclusive (Chưa rõ ràng)**: Thiếu dữ liệu, có thể tạm dừng để con người vào kiểm tra trực tiếp.

> [!IMPORTANT]
> Việc tích hợp **Analysis** biến Argo Rollouts thành một cỗ máy tự động hóa hoàn toàn. Bạn có thể tự tin đi ngủ trong khi hệ thống đang tự chạy Canary, tự đo đếm và tự lùi lại nếu có bất cứ lỗi nào xảy ra.
