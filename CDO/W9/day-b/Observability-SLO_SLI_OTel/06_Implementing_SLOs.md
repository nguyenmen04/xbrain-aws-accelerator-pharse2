# The Site Reliability Workbook - Implementing SLOs

Tài liệu tham khảo: [Implementing SLOs](https://sre.google/workbook/implementing-slos)

Chương này hướng dẫn thực hành chi tiết cách thiết lập SLO cho một dịch vụ.

## Triết Lý Đo Lường Hướng Người Dùng (User-centric)

Cách tiếp cận đúng khi tạo SLO là phải **đứng trên góc nhìn của người dùng**. Người dùng không quan tâm máy chủ của bạn chạy bao nhiêu phần trăm CPU hay có bao nhiêu RAM trống. Họ chỉ quan tâm:
*   "App này có đang chạy không?" (Availability)
*   "Sao tôi bấm nút Mua mà nó quay vòng vòng lâu thế?" (Latency)

Do đó, SLI/SLO phải phản ánh trực tiếp trải nghiệm của người dùng.

## Ngân Sách Lỗi (Error Budget) - Trái tim của SLO

Thay vì chỉ nhìn vào mức độ thành công (ví dụ 99.9%), SRE tập trung quản lý **Ngân sách lỗi (Error Budget)**.

*   **Công thức:** `Error Budget = 100% - SLO`
*   *Ví dụ:* Nếu SLO của bạn là 99.9% cho chu kỳ 30 ngày (tương đương khoảng 4.3 triệu request). Thì Error Budget của bạn là **0.1%**. Tức là bạn được phép để xảy ra tối đa **4,300 request lỗi** trong tháng đó.

### Ý Nghĩa Của Error Budget
Nó là công cụ giao tiếp và đàm phán giữa team phát triển (Dev - muốn tung tính năng nhanh) và team vận hành (SRE/Ops - muốn hệ thống ổn định):
*   **Khi ngân sách còn dồi dào:** Team Dev cứ thoải mái đẩy (deploy) tính năng mới lên, dù thỉnh thoảng có vài lỗi nhỏ làm tốn chút budget cũng không sao.
*   **Khi ngân sách cạn kiệt (hoặc âm):** Hệ thống đang không đạt cam kết. Toàn bộ team **phải dừng việc ra mắt tính năng mới**. Ưu tiên số 1 lúc này là bảo trì, viết test, và sửa lỗi cho đến khi bước sang chu kỳ tháng mới có lại budget.

## Các Khung Thời Gian (Windows)
Đừng đánh giá SLO dựa trên "hôm qua". Google khuyên nên dùng khung thời gian **Cuộn (Rolling Window)**, phổ biến nhất là **28 hoặc 30 ngày**.
Hôm nay bạn đánh giá dữ liệu từ 30 ngày trước, ngày mai bạn đánh giá dữ liệu của 30 ngày trước đó tịnh tiến lên. Nó phản ánh chân thực hơn về ấn tượng tổng thể của người dùng.
