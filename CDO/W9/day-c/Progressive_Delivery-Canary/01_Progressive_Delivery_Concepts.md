# Progressive Delivery & Canary Deployment - Khái niệm cốt lõi

Tài liệu tham khảo: [Progressive Delivery patterns (CNCF)](https://www.cncf.io/blog/2024/01/26/progressive-delivery/)

---

## 1. Progressive Delivery là gì?

Trước đây với CI/CD truyền thống (Continuous Delivery), khi code được hợp nhất, hệ thống thường đẩy thẳng 100% bản cập nhật mới lên Production. Rủi ro của phương pháp này là nếu có lỗi, toàn bộ người dùng sẽ bị ảnh hưởng.

**Progressive Delivery (Phân phối lũy tiến)** nâng cấp khái niệm này bằng cách:

*   **Kiểm soát "Blast Radius" (Bán kính ảnh hưởng):** Lỗi (nếu có) chỉ ảnh hưởng tới một lượng siêu nhỏ người dùng, tránh gây sập toàn bộ hệ thống.
*   **Tách biệt Deploy và Release:**
    *   **Deploy:** Đưa code mới lên server và khởi chạy (có thể chưa ai nhìn thấy).
    *   **Release:** Mở luồng giao thông (traffic) và quyết định ai được phép sử dụng tính năng mới đó.
*   **Ra quyết định dựa trên dữ liệu (Data-driven):** Quá trình phát hành (Release) hoàn toàn phụ thuộc vào các chỉ số giám sát (Metrics/SLO).

---

## 2. Canary Deployment (Chiến lược cốt lõi)

Thuật ngữ *Canary* xuất phát từ việc thợ mỏ xưa mang theo chim hoàng yến xuống hầm để phát hiện khí độc. Trong lĩnh vực phần mềm, chiến lược này hoạt động như sau:

1.  **Triển khai song song:** Đưa phiên bản mới (v2) lên Production song song với phiên bản cũ (v1).
2.  **Mở traffic nhỏ:** Chỉ trích xuất một lượng rất nhỏ traffic (ví dụ 5%) để chuyển vào v2.
3.  **Quan sát & Đánh giá:** Đo lường các chỉ số sức khỏe của v2 (như Error rate, Latency).
4.  **Tăng dần hoặc Quay xe (Rollback):**
    *   Nếu v2 chạy tốt: Tiếp tục tăng dần traffic (10% -> 25% -> 50% -> 100%).
    *   Ngay khi phát hiện "khí độc" (tỷ lệ lỗi tăng đột biến): Lập tức chuyển 100% traffic về lại v1 tự động.

> [!TIP]
> **Tóm lại:** Progressive Delivery là tư duy triển khai chậm mà chắc. Canary Deployment là cách làm cụ thể (nhỏ giọt traffic) để thực hiện tư duy đó.
