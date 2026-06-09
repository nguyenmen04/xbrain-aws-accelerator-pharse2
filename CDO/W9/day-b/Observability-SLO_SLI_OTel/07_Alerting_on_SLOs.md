# Alerting on SLOs: Cảnh Báo Đa Khung Thời Gian

Tài liệu tham khảo: [Alerting on SLOs (Multi-window burn rate alert)](https://sre.google/workbook/alerting-on-slos)

Chương này giải quyết một nỗi đau lớn của SRE: **Alert Fatigue (Sự mệt mỏi vì bị cảnh báo rác làm phiền)**. Việc cảnh báo sai hoặc cảnh báo quá mức sẽ khiến kỹ sư lờ đi các cảnh báo thực sự quan trọng.

## Tốc độ tiêu hao (Burn Rate)
Thay vì cảnh báo "Hệ thống đang có 500 lỗi!", hãy cảnh báo dựa trên câu hỏi: **"Với tốc độ lỗi như hiện tại, bao lâu nữa thì ngân sách lỗi (Error Budget) sẽ cạn sạch?"**

Khái niệm này gọi là **Burn Rate**:
*   `Burn Rate = 1`: Tốc độ tiêu hao chuẩn. Nếu cứ tiếp diễn trong 30 ngày, ngân sách lỗi sẽ cạn đúng vào ngày 30.
*   `Burn Rate = 10`: Ngân sách lỗi đang cạn nhanh gấp 10 lần bình thường. Nó sẽ sạch bách trong vòng 3 ngày (30/10).
*   `Burn Rate = 36`: Ngân sách sẽ cạn trong chưa tới 1 ngày. (Trường hợp khẩn cấp).

## Phương Pháp Tốt Nhất: Multi-window Burn Rate Alerts

Để không báo động giả, hệ thống cần đủ thông minh để nhận ra lỗi là sự cố chớp nhoáng rồi tự hết, hay là lỗi đang kéo dài nghiêm trọng. Phương pháp tốt nhất của Google kết hợp **2 khung thời gian** (dài và ngắn) để đưa ra quyết định.

### 1. Cảnh báo Nhanh (Fast Burn / Paging Alert)
Dành cho các sự cố diện rộng, ảnh hưởng nghiêm trọng tới người dùng tức thì.
*   **Điều kiện:** Nếu Burn Rate rất cao (ví dụ: > 14) diễn ra trong một khung thời gian ngắn (ví dụ: mất 2% Error Budget chỉ trong vòng 1 giờ).
*   **Hành động:** **Page**. Kích hoạt chuông điện thoại gọi kỹ sư trực ca (On-call) dậy ngay lập tức, bất kể là nửa đêm.

### 2. Cảnh báo Chậm (Slow Burn / Ticketing Alert)
Dành cho các sự cố rò rỉ ngầm, lỗi xuất hiện rải rác nhưng kéo dài nhiều ngày. Mặc dù không ai phàn nàn tức thì, nhưng cuối tháng bạn sẽ trượt SLO.
*   **Điều kiện:** Nếu Burn Rate tương đối cao (ví dụ: > 1.5) duy trì liên tục trong thời gian dài (ví dụ: mất 5% Error Budget trong vòng 3 ngày qua).
*   **Hành động:** **Ticket**. Không cần đánh thức ai. Hệ thống tự động tạo một Ticket (Jira/Email) để team vào công ty vào sáng hôm sau và xử lý trong giờ hành chính.

> [!TIP]
> Việc áp dụng Burn Rate Alert giúp giảm tới 80% lượng cảnh báo rác, cho phép kỹ sư có những giấc ngủ ngon hơn, và chỉ phải bật dậy khi hệ thống thực sự đang "cháy" ngân sách.
