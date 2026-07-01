# Bài viết ngắn (Write-up) - Lab 25: GPU FinOps Optimization

## 1. Baseline vs. Optimized
- **Chi phí (Spend):** Chi phí Baseline hàng tháng là **$27,133**. Sau khi áp dụng tối ưu hóa (Optimized), chi phí giảm xuống còn **$15,660**.
- **Tiết kiệm tổng cộng:** Số tiền tiết kiệm dự kiến là **$11,473**, tương đương **42%**.
- **Inference Unit Economics:** Đo lường chi phí mỗi 1 triệu token (`$/1M-token`), baseline là **$6.488/1M-token** và sau tối ưu chỉ còn **$1.126/1M-token** (tiết kiệm lên tới 82.6% cho phần suy luận).

## 2. Phân tích từng đòn bẩy (Levers)
Dựa vào biểu đồ tiết kiệm (Savings by lever):
- **Purchasing (spot/reserved)** đóng góp nhiều nhất vào việc giảm chi phí (khoảng **$9,006**). Lý do là vì việc chuyển đổi từ trả tiền theo giờ (On-Demand) với giá đắt đỏ sang Spot (cho các tác vụ có thể bị gián đoạn) và Reserved (cho tác vụ chạy liên tục) mang lại mức chiết khấu lớn nhất (lên đến 45% hoặc thậm chí cao hơn) trên toàn bộ máy chủ, chứ không chỉ trên lượng nhỏ request.
- **Inference (cascade/cache/batch)** tiết kiệm được **$1,212**. Mặc dù tiết kiệm `$/1M-token` lớn (lên tới >80%), nhưng tổng chi phí của inference so với chi phí mua (purchasing) nguyên máy chủ GPU có thể thấp hơn. 
- **Right-size util-lies** tiết kiệm **$655**.
- **Kill idle GPUs** (tắt máy để không) tiết kiệm **$600**.

## 3. GPU-Util Lie
- Các GPU gặp hiện tượng "GPU-Util Lie" là: **`gpu-h100-4`** và **`gpu-a10g-1`**. Mức sử dụng `util%` hiển thị trên hệ thống rất cao (ví dụ H100 là 98.2%, A10G là 96.9%) nhưng giá trị MFU thực sự rất thấp (dưới 30%, ví dụ H100 chỉ đạt 0.194).
- **Tác động tài chính:** Bạn phải trả toàn bộ tiền thuê cho một GPU mạnh mẽ, nhưng chỉ nhận được lượng sức mạnh tính toán chưa tới 1/5 khả năng tối đa của nó. Chuyển đổi (Right-sizing) hoặc thay thế GPU bị "lie" tiết kiệm cho công ty $655 mỗi tháng. Hiện tượng này thường xảy ra do "memory stall" (tắc nghẽn bộ nhớ, ví dụ LLM decoding là memory-bound) chứ không phải do thiếu I/O.

## 4. Phần mở rộng (Extensions) đã làm

**Extension 1: Cải thiện `recommend_tier()`**
- **Mô tả:** Đã thêm logic kiểm tra tỷ lệ gián đoạn dựa vào loại GPU (`gpu_type`) và kiểm tra thời gian thực tế của công việc (`job_days`). Cụ thể, các tác vụ ngắn hạn (< 1 năm) không nên bị khóa bằng cam kết Reserved 3 năm, thay vào đó so sánh với chiết khấu Reserved 1 năm (khoảng 20%). Ngoài ra, các GPU như A10G hay L4 thường có tỷ lệ bị "đòi lại" cao hơn nên không phù hợp với Spot.
- **Kết quả:** Sau khi áp dụng thay đổi, một số job như `job-infer-search` trên hệ `L4` đã được điều chỉnh quay về dùng `on_demand` thay vì dùng spot hoặc reserved sai chiến lược. Việc này đảm bảo tỷ lệ downtime thấp cho công việc quan trọng.

**Extension 3: Đánh giá lợi ích kinh tế của bộ nhớ đệm (Cache)**
- **Mô tả:** Đã tạo thêm hàm `cache_is_worth_it` để đánh giá chi phí ghi vào cache (đắt gấp 1.25 lần base) và chi phí đọc từ cache (chiết khấu 90%). Hàm sẽ quyết định việc dùng cache chỉ khi số lần đọc lại đủ bù đắp chi phí ghi. Trong môi trường cấu hình, giả sử cần số lần đọc (`avg_cache_reads`) lớn hơn mức bù lỗ (ví dụ 1.25x so với chiết khấu) thì mới bật. Trong M2, chúng ta gán đọc trung bình 15 lần để kiểm tra độ trễ.
- **Kết quả:** Số lượt dùng request có giá trị tối ưu về cache vẫn được duy trì, tiết kiệm cho M2 giữ vững ở mức 82.6%. Điều này mang lại **insight quan trọng**: Prompt Caching là vũ khí cực mạnh cho các prompt ngữ cảnh dài, nhưng nếu các prompt là một lần (1-shot read) thì hoàn toàn lãng phí tiền lưu trữ cache.

## 5. Khuyến nghị cho NimbusAI (Tư cách FinOps Lead)
1. **Tự động áp dụng Lifecycle (Spot/Reserved):** Lập tức phân tách các workload có thể bị gián đoạn (Training) qua Spot và cấu hình Checkpoint tự động. Mua ngay Reserved capacity cho Inference 24/7.
2. **Khuyến khích API Router thông minh (Cascade):** Các truy vấn dễ nên được tự động chuyển hướng qua mô hình "small" thay vì luôn đánh vào "large". 
3. **Triển khai Chargeback & FOCUS:** Với độ phủ tag đạt 92%, NimbusAI đã sẵn sàng thực hiện Chargeback. Cần gửi hóa đơn showback hoặc chargeback định kỳ tới các team (Assistant, Search, Eval, RAG) theo chuẩn hóa đơn FOCUS để mỗi team có trách nhiệm tự tối ưu `$/1M-token` nội bộ.
