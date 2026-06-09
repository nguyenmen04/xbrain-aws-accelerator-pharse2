# OpenTelemetry (OTel) - Concepts & Instrumentation

Tài liệu tham khảo: [OpenTelemetry Docs](https://opentelemetry.io/docs)

## OpenTelemetry là gì?
OpenTelemetry (OTel) là một framework mã nguồn mở (open-source) và tiêu chuẩn công nghiệp (industry standard) dùng để tạo ra, thu thập, và xuất các dữ liệu quan trắc (Telemetry data). 
Điểm mạnh lớn nhất của OTel là tính **vendor-neutral** (không phụ thuộc vào bất kỳ nhà cung cấp dịch vụ lưu trữ nào như Datadog, New Relic, v.v.). Bạn chỉ cần cấu hình ứng dụng một lần và có thể gửi dữ liệu đi bất cứ đâu.

## Ba Trụ Cột (Three Pillars) Của Observability

OTel tập trung vào 3 loại dữ liệu chính:

1.  **Traces (Dấu vết):** 
    *   Mô tả bức tranh toàn cảnh về cách một yêu cầu (request) đi qua nhiều dịch vụ (microservices) khác nhau.
    *   Một Trace được tạo thành từ nhiều **Spans** (đại diện cho một công việc hoặc một bước xử lý trong service).
    *   *Tác dụng:* Giúp tìm ra nút thắt cổ chai (bottleneck) về hiệu năng đang nằm ở microservice nào.
2.  **Metrics (Số liệu đo lường):**
    *   Là những dữ liệu định lượng được tổng hợp theo thời gian (ví dụ: CPU đang dùng 80%, số request lỗi là 5req/s, thời gian phản hồi trung bình là 200ms).
    *   *Tác dụng:* Rất nhẹ, lưu trữ ít tốn kém, cực kỳ phù hợp để tạo các Dashboard tổng quan và thiết lập luật Cảnh báo (Alerts).
3.  **Logs (Nhật ký):**
    *   Các đoạn văn bản ghi lại chi tiết một sự kiện cụ thể đã xảy ra vào một thời điểm (ví dụ: "User A vừa mua hàng thành công").
    *   *Tác dụng:* Cung cấp ngữ cảnh chi tiết nhất để debug lỗi (Root Cause Analysis).

## Khái niệm về Instrumentation (Cấy ghép code)

Để ứng dụng của bạn sinh ra được 3 loại dữ liệu trên, bạn cần "instrument" (cấy/gắn thêm code) vào nó. OTel hỗ trợ 2 cách chính:

*   **Auto-instrumentation (Cấy tự động):**
    *   Không cần sửa code của ứng dụng.
    *   OTel Agent sẽ chạy kèm với ứng dụng và tự động móc (hook) vào các framework/thư viện phổ biến (như Spring Boot trong Java, Express trong Node.js, v.v.) để lấy ra các thông số cơ bản về HTTP, Database calls.
    *   *Cách nhanh nhất để bắt đầu.*
*   **Manual-instrumentation (Cấy thủ công):**
    *   Bạn tự sử dụng bộ thư viện SDK của OTel trong code.
    *   Dùng để tạo ra các Spans, Metrics đặc thù mang ý nghĩa nghiệp vụ riêng biệt của ứng dụng (ví dụ: đếm số lượng "Đơn hàng đã hủy" thay vì chỉ đếm HTTP 404).

> [!TIP]
> Người mới bắt đầu nên cài đặt **Auto-instrumentation** trước để có cái nhìn tổng quan ngay lập tức. Sau đó kết hợp **Manual-instrumentation** tại các đoạn code phức tạp và quan trọng.
