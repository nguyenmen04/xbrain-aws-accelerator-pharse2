# Grafana Loki (Logs)

Tài liệu tham khảo: [Loki Docs](https://grafana.com/docs/loki/latest)

## Loki là gì?
Loki là một hệ thống gom cụm và lưu trữ log (Log aggregation system) được thiết kế đặc biệt lấy cảm hứng từ Prometheus. Nó được mệnh danh là "Prometheus dành cho Logs".

## Tại sao lại cần Loki? (Vấn đề của các hệ thống cũ)
Các hệ thống quản lý log truyền thống (như Elasticsearch) rất mạnh mẽ vì chúng tạo chỉ mục (index) cho **toàn bộ văn bản** (full-text search). Nghĩa là mọi chữ trong mỗi dòng log đều được lập danh sách để tìm kiếm nhanh. 
Nhưng nhược điểm lớn là điều này **cực kỳ tốn kém** cả về CPU lẫn dung lượng ổ cứng (RAM) khi khối lượng log lên tới hàng Terabytes mỗi ngày.

## Phương pháp độc đáo của Loki

Loki tiếp cận một cách rất thông minh để giải quyết bài toán chi phí: **Chỉ Index các Nhãn (Labels)**.

1.  **Lưu trữ dựa trên Label:**
    Thay vì mổ xẻ nội dung của dòng log `"Error connecting to database XYZ"`, Loki chỉ quan tâm đến các thẻ (nhãn) gắn kèm với dòng log đó, ví dụ: `{app="my-backend", env="production", level="error"}`.
    *   Loki chỉ lập index cho các chuỗi nhãn này.
    *   Phần nội dung văn bản thô của log sẽ được gộp lại (chunk), nén chặt và đẩy thẳng vào kho lưu trữ giá siêu rẻ như Amazon S3 hoặc Google Cloud Storage.

2.  **Sự tương đồng với Prometheus:**
    Bởi vì cách đặt nhãn (Labels) của Loki giống hệt Prometheus, bạn có thể dễ dàng liên kết dữ liệu giữa hai bên trong Grafana. 
    *Ví dụ:* Đang xem biểu đồ (Prometheus) thấy CPU của pod `{app="api"}` tăng cao đột ngột, bạn chỉ cần giữ nguyên nhãn `{app="api"}` và bấm nút chuyển sang tab Logs (Loki) là sẽ xem được ngay các log gây ra lỗi tại thời điểm đó, mà không cần gõ lại từ khóa.

3.  **Công cụ thu thập Promtail:**
    Thường đi kèm với Loki là agent tên **Promtail**. Promtail chạy trên các máy chủ, đọc các file `.log`, gán nhãn cho chúng (dựa vào cấu hình Kubernetes hoặc thư mục) và gửi về cho hệ thống Loki trung tâm lưu trữ.

> [!WARNING]
> Vì không dùng full-text index, nếu bạn gõ từ khóa tìm kiếm một chuỗi text (regex) bất kỳ trên giao diện Grafana, Loki sẽ phải dùng cách quét "cơ bắp" (brute-force) qua các file đã nén. Nếu quét một khoảng thời gian quá dài, nó có thể chậm hơn Elasticsearch. Bí quyết dùng Loki là luôn **lọc theo Label trước** để thu hẹp phạm vi thời gian/ứng dụng, sau đó mới tìm kiếm văn bản bên trong.
