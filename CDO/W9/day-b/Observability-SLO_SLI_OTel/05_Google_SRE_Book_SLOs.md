# Google SRE Book - Service Level Objectives (SLOs)

Tài liệu tham khảo: [SRE Book - SLO Chapter](https://sre.google/sre-book/service-level-objectives)

## Các Định Nghĩa Cốt Lõi (SLI, SLO, SLA)

Chương này giải thích cách Google đo lường độ tin cậy thực tế của dịch vụ để không làm quá tải hệ thống hay làm mất lòng khách hàng.

1.  **Chỉ số Cấp độ Dịch vụ - SLI (Service Level Indicator):**
    *   **Là gì?** Một con số đo lường **thực trạng** khách quan về hoạt động của hệ thống ở hiện tại.
    *   *Ví dụ:* Tỷ lệ request thành công (không phải 5xx), Độ trễ của request, Tính khả dụng (Uptime).
    *   *Cách tính chuẩn:* Luôn tính theo tỷ lệ (Tỷ lệ tốt / Tổng số lượng).

2.  **Mục tiêu Cấp độ Dịch vụ - SLO (Service Level Objective):**
    *   **Là gì?** Là **ngưỡng kỳ vọng** (mục tiêu) bạn đặt ra cho SLI. Nó là cam kết nội bộ giữa các team (Dev và Ops/SRE).
    *   *Ví dụ:* SLI (Tỷ lệ request thành công) > 99.9% tính trong chu kỳ 30 ngày qua.
    *   *Lưu ý:* 100% là một mục tiêu **sai lầm**. Việc cố gắng đạt 100% độ tin cậy là không thể (vì mạng mẽo, phần cứng luôn có thể hỏng) và làm trì trệ việc tung ra các tính năng mới do sợ rủi ro.

3.  **Thỏa thuận Cấp độ Dịch vụ - SLA (Service Level Agreement):**
    *   **Là gì?** Là một **bản hợp đồng kinh doanh** với người dùng cuối/khách hàng dựa trên các SLO. 
    *   Nếu hệ thống không đạt được SLO đã cam kết trong SLA, công ty thường phải bồi thường (thường là hoàn tiền hoặc tặng credit).
    *   *Mẹo:* Thông thường SLO nội bộ sẽ được đặt khắt khe hơn SLA (Ví dụ: SLO nội bộ là 99.9%, SLA cam kết với khách chỉ là 99.5% để tạo vùng đệm an toàn).

> [!IMPORTANT]
> Dành cho team kỹ thuật, bạn chỉ cần quan tâm đến **SLI** (để đo) và **SLO** (để nhắm mục tiêu). SLA thường là việc của bộ phận kinh doanh và pháp chế. Hệ thống có mạnh đến đâu, nếu không có SLO thì không ai biết nó đang tốt hay xấu.
