# DESIGN.md — Thiết kế bộ điều phối Closed-Loop Auto-Remediation (Thư mục tdd)

Tài liệu này giải thích các quyết định kiến trúc, tham số an toàn và chiến lược thiết kế được sử dụng cho bộ điều phối closed-loop tự động khắc phục sự cố tại thư mục `tdd`.

---

## 1. Decision engine: Rule-based hay LLM-based?

**Lựa chọn: Rule-based.**

### Lý do lựa chọn:
1. **Thời gian phản hồi cực thấp (Latency < 1ms):** Bộ điều phối e-commerce cần phản ứng tức thời. Rule-based xử lý mapping cảnh báo chỉ trong vài phần triệu giây, trong khi LLM cần từ 200ms đến hơn 1s cho một lượt gọi API qua mạng.
2. **Độ tin cậy và Determinism tuyệt đối (100%):** Trong hệ thống vận hành AIOps, cùng một sự cố kỹ thuật (như `InstanceDown`) phải được giải quyết bằng một runbook cố định đã qua kiểm duyệt (`restart_service.sh`). LLM có rủi ro tạo ra các phản hồi ngẫu nhiên (hallucination) có thể gây gián đoạn hệ thống.
3. **Chi phí vận hành:** Rule-based miễn phí hoàn toàn, không phụ thuộc vào kết nối API bên ngoài hay token usage.

### Trade-offs:
* **Rule-based:** Không linh hoạt khi xuất hiện các cảnh báo có định dạng mới hoặc mô tả bằng ngôn ngữ tự nhiên phức tạp. Cần cập nhật file cấu hình yaml thủ công khi scale-up hệ thống.
* **LLM-based:** Tự suy luận tốt trong các kịch bản lỗi chưa từng gặp, nhưng đòi hỏi cơ chế phòng vệ hallucination phức tạp (như registry whitelist mà chúng ta đã xây dựng).

---

## 2. Blast-Radius Configuration (Giới hạn vùng ảnh hưởng)

Cấu hình an toàn trong [config.yaml](config.yaml):
```yaml
blast_radius:
  max_actions_per_minute: 3
  max_restarts_per_service_per_hour: 5
```

### Lý do chọn các giá trị này:
* `max_actions_per_minute: 3`: Stack hệ thống Ronki có 5 dịch vụ. Nếu xảy ra hiện tượng lỗi dây chuyền (cascade failure), giới hạn 3 hành động mỗi phút ngăn chặn bộ điều phối restart đồng loạt tất cả các container cùng lúc, giảm thiểu hiện tượng thundering herd và quá tải cơ sở dữ liệu.
* `max_restarts_per_service_per_hour: 5`: Nếu một dịch vụ bị lỗi và phải khởi động lại quá 5 lần trong một giờ mà vẫn không tự phục hồi (ví dụ: do cấu hình sai, lỗi OOM lặp lại, hoặc lỗi logic code), việc tiếp tục khởi động lại là vô ích và có hại. Hệ thống cần dừng lại để kỹ sư can thiệp trực tiếp.

---

## 3. Verify Step (Bước xác minh sau hành động)

### Metric và Ngưỡng xác minh (Threshold):
* **Trạng thái kết nối (`up_required: 1`):** Dịch vụ phải có trạng thái hoạt động bình thường (`up == 1` trên Prometheus) trước khi đo đạc latency.
* **Độ trễ p99 (`latency_p99_max_ms: 500`):** Dựa trên [baseline.json](../data-pack/data/baseline.json), độ trễ p99 của dịch vụ chậm nhất (`checkout-svc`) ở điều kiện bình thường là 230ms. Đặt ngưỡng 500ms giúp lọc bỏ các nhiễu độ trễ nhỏ nhưng vẫn phát hiện được sự cố nghẽn mạng thực sự.

### Timeout và Chu kỳ Polling:
* **Timeout (`verify_timeout_seconds: 60`):** Khởi động lại container và cập nhật metrics trong Prometheus thường mất từ 15-30 giây. 60 giây là khoảng thời gian đủ rộng.
* **Polling Interval (`verify_poll_interval_seconds: 10`):** Đồng bộ với chu kỳ scrape của Prometheus để nhận dữ liệu mới nhất.
* **Consecutive Passes (`verify_min_samples: 3`):** Đòi hỏi tối thiểu 3 mẫu liên tiếp vượt qua kiểm tra. Điều này ngăn chặn trường hợp dịch vụ vừa khởi động lại có 1 mẫu phản hồi nhanh nhất thời nhưng sau đó lại nghẽn.

---

## 4. Circuit Breaker Reset (Cơ chế cầu chì)

**Reset mode: Manual (Thủ công).**

### Lý do:
* Cầu chì tự động mở khi phát hiện 3 lỗi xác minh/hành động liên tiếp. Đây là tín hiệu hệ thống gặp lỗi nghiêm trọng vượt quá khả năng tự chữa lành (Self-healing).
* Nếu để cầu chì tự động đóng lại (auto-reset) sau N phút, bộ điều phối có thể rơi vào vòng lặp restart vô hạn khi sự cố gốc chưa được khắc phục, dẫn đến cạn kiệt tài nguyên hệ thống.
* Yêu cầu reset thủ công bằng cách khởi động lại quy trình Python đảm bảo kỹ sư đã vào kiểm tra log, phân tích nguyên nhân gốc rễ và xác nhận hệ thống an toàn.

---

## 5. Chiến lược xử lý Concurrency (Stress #5)

* **Thiết kế Mutex:** Sử dụng một dictionary `_service_locks` chứa các đối tượng `threading.Lock` được phân khóa theo tên dịch vụ.
* **Non-blocking lock:** Sử dụng phương thức `acquire(blocking=False)`. Nếu một cảnh báo trùng lặp gửi tới khi dịch vụ đang được xử lý, thread sẽ ngay lập tức ghi nhận cảnh báo `SERVICE_LOCK_BUSY` và bỏ qua thay vì đứng xếp hàng chờ đợi. Các dịch vụ khác nhau xử lý song song độc lập hoàn toàn mà không block nhau.

---

## 6. Thứ tự hoàn tác Multi-Step (Stress #4)

* **Quy tắc LIFO (Last-In, First-Out):** Khi một chuỗi deploy (A → B → C) bị lỗi ở bước C, bộ điều phối sẽ duyệt qua danh sách các bước đã hoàn thành theo thứ tự ngược lại (B → A) để thực hiện rollback (Rollback-B rồi mới đến Rollback-A).
* Thiết kế này đảm bảo trạng thái cấu hình của hệ thống luôn nhất quán, tránh tình trạng rollback bước khởi đầu trước làm lộ cấu hình lỗi ra môi trường mạng.

---

## 7. LLM Hallucination Defense (Stress #6)

* **Whitelist registry:** Kiểm tra tên runbook quyết định trực tiếp với danh sách đăng ký an toàn (`runbook_registry`) trong file `config.yaml` trước khi thực thi dry-run.
* Nếu tên runbook trả về từ quyết định không tồn tại trong whitelist, bộ điều phối ngay lập tức log cảnh báo `DECISION_VALIDATION_FAILED` và dừng xử lý, bảo vệ hệ thống khỏi việc thực thi các tiến trình không mong muốn.
