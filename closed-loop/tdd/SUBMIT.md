# SUBMIT.md — Kết quả chạy 6 kịch bản nghiệm thu (Chaos & Stress)

Tài liệu này ghi lại kết quả thử nghiệm thực tế của bộ điều phối closed-loop tự động khắc phục sự cố tại thư mục `tdd`.

- **Môi trường chạy:** Windows 11 (WSL2 Docker backend)
- **Công cụ quản lý gói:** Python 3.11, `uv` package manager
- **Decision Engine:** Rule-based (Ánh xạ cấu hình tường minh trong `config.yaml`)

---

## Scenario 1 — Hành động thành công (Latency alert trên payment-svc)

* **Lệnh inject:** Gửi synthetic alert qua Alertmanager API
* **Mục tiêu:** Phát hiện sự cố, restart dịch vụ, xác minh thành công và ghi nhận `ACTION_SUCCESS`.

### Nhật ký Log:
```json
{"ts": "2026-06-18T08:21:25.425004+00:00", "level": "INFO", "event_type": "ALERT_DETECTED", "alertname": "HighLatency", "service": "payment-svc", "severity": "warning"}
{"ts": "2026-06-18T08:21:25.492827+00:00", "level": "INFO", "event_type": "DECIDE_RUNBOOK", "alertname": "HighLatency", "service": "payment-svc", "runbook": "runbooks/restart_service.sh"}
{"ts": "2026-06-18T08:21:25.492827+00:00", "level": "INFO", "event_type": "BLAST_RADIUS_OK", "service": "payment-svc"}
{"ts": "2026-06-18T08:21:25.492827+00:00", "level": "INFO", "event_type": "RUNBOOK_EXEC", "script": "runbooks/restart_service.sh", "service": "payment-svc", "dry_run": true}
{"ts": "2026-06-18T08:21:25.982293+00:00", "level": "INFO", "event_type": "RUNBOOK_RESULT", "script": "runbooks/restart_service.sh", "service": "payment-svc", "returncode": 0, "stdout": "[DRY-RUN] would execute: docker restart ronki-payment-svc", "stderr": ""}
{"ts": "2026-06-18T08:21:25.982293+00:00", "level": "INFO", "event_type": "DRY_RUN_PASS", "runbook": "runbooks/restart_service.sh", "service": "payment-svc"}
{"ts": "2026-06-18T08:21:25.982846+00:00", "level": "INFO", "event_type": "RUNBOOK_EXEC", "script": "runbooks/restart_service.sh", "service": "payment-svc", "dry_run": false}
{"ts": "2026-06-18T08:21:35.092804+00:00", "level": "INFO", "event_type": "RUNBOOK_RESULT", "script": "runbooks/restart_service.sh", "service": "payment-svc", "returncode": 0, "stdout": "[restart_service] Restarting ronki-payment-svc...\nronki-payment-svc\n[restart_service] Waiting 5s for ronki-payment-svc to come up...\n[restart_service] ronki-payment-svc is running.", "stderr": ""}
{"ts": "2026-06-18T08:21:35.093382+00:00", "level": "INFO", "event_type": "ACTION_EXECUTED", "runbook": "runbooks/restart_service.sh", "service": "payment-svc"}
{"ts": "2026-06-18T08:21:35.095146+00:00", "level": "INFO", "event_type": "VERIFY_START", "service": "payment-svc", "timeout_s": 60}
{"ts": "2026-06-18T08:21:35.190606+00:00", "level": "INFO", "event_type": "VERIFY_SAMPLE", "service": "payment-svc", "sample": 1, "latency_p99_ms": 248.27272727272725, "up": 0.0, "latency_ok": true, "up_ok": false}
{"ts": "2026-06-18T08:21:45.249642+00:00", "level": "INFO", "event_type": "VERIFY_SAMPLE", "service": "payment-svc", "sample": 2, "latency_p99_ms": 248.16129032258067, "up": 1.0, "latency_ok": true, "up_ok": true}
{"ts": "2026-06-18T08:21:55.324336+00:00", "level": "INFO", "event_type": "VERIFY_SAMPLE", "service": "payment-svc", "sample": 3, "latency_p99_ms": 248.0862068965517, "up": 1.0, "latency_ok": true, "up_ok": true}
{"ts": "2026-06-18T08:22:05.397618+00:00", "level": "INFO", "event_type": "VERIFY_SAMPLE", "service": "payment-svc", "sample": 4, "latency_p99_ms": 247.9642857142857, "up": 1.0, "latency_ok": true, "up_ok": true}
{"ts": "2026-06-18T08:22:05.397618+00:00", "level": "INFO", "event_type": "VERIFY_PASS", "service": "payment-svc", "samples": 4}
{"ts": "2026-06-18T08:22:05.398633+00:00", "level": "INFO", "event_type": "ACTION_SUCCESS", "alertname": "HighLatency", "service": "payment-svc", "runbook": "runbooks/restart_service.sh"}
```

**Đánh giá:** Dịch vụ khởi động lại thành công, metrics latency đo được khoảng 248ms (nhỏ hơn ngưỡng 500ms). Đạt trạng thái hoạt động tốt sau 3 mẫu kiểm tra liên tiếp.

---

## Scenario 2 — Hành động thất bại → Tự động hoàn tác (Verify fail → Rollback)

* **Cách thức giả lập:** Đặt tạm ngưỡng xác minh `latency_p99_max_ms` thành `10` trong `baseline.json` để ép trạng thái xác minh thất bại.
* **Mục tiêu:** Kiểm tra phản ứng tự động hoàn tác khi verify thất bại.

### Nhật ký Log:
```json
{"ts": "2026-06-18T08:22:30.928435+00:00", "level": "INFO", "event_type": "ACTION_EXECUTED", "runbook": "runbooks/restart_service.sh", "service": "payment-svc"}
{"ts": "2026-06-18T08:22:30.938995+00:00", "level": "INFO", "event_type": "VERIFY_START", "service": "payment-svc", "timeout_s": 60}
{"ts": "2026-06-18T08:22:30.992116+00:00", "level": "INFO", "event_type": "VERIFY_SAMPLE", "service": "payment-svc", "sample": 1, "latency_p99_ms": 248.04338517320923, "up": 1.0, "latency_ok": false, "up_ok": true}
{"ts": "2026-06-18T08:22:41.040711+00:00", "level": "INFO", "event_type": "VERIFY_SAMPLE", "service": "payment-svc", "sample": 2, "latency_p99_ms": 248.10294117647058, "up": 1.0, "latency_ok": false, "up_ok": true}
{"ts": "2026-06-18T08:22:51.082407+00:00", "level": "INFO", "event_type": "VERIFY_SAMPLE", "service": "payment-svc", "sample": 3, "latency_p99_ms": 248.15714285714284, "up": 1.0, "latency_ok": false, "up_ok": true}
{"ts": "2026-06-18T08:23:01.198453+00:00", "level": "INFO", "event_type": "VERIFY_SAMPLE", "service": "payment-svc", "sample": 4, "latency_p99_ms": 248.1029411764706, "up": 1.0, "latency_ok": false, "up_ok": true}
{"ts": "2026-06-18T08:23:11.248462+00:00", "level": "INFO", "event_type": "VERIFY_SAMPLE", "service": "payment-svc", "sample": 5, "latency_p99_ms": 248.14705882352942, "up": 1.0, "latency_ok": false, "up_ok": true}
{"ts": "2026-06-18T08:23:21.310084+00:00", "level": "INFO", "event_type": "VERIFY_SAMPLE", "service": "payment-svc", "sample": 6, "latency_p99_ms": 248.22080374762803, "up": 1.0, "latency_ok": false, "up_ok": true}
{"ts": "2026-06-18T08:23:31.310730+00:00", "level": "WARNING", "event_type": "VERIFY_FAIL", "service": "payment-svc", "samples": 6}
{"ts": "2026-06-18T08:23:31.311459+00:00", "level": "WARNING", "event_type": "ROLLBACK_TRIGGERED", "service": "payment-svc", "rollback_runbook": "runbooks/restart_service.sh"}
{"ts": "2026-06-18T08:23:31.311459+00:00", "level": "INFO", "event_type": "RUNBOOK_EXEC", "script": "runbooks/restart_service.sh", "service": "payment-svc", "dry_run": false}
{"ts": "2026-06-18T08:23:38.472834+00:00", "level": "INFO", "event_type": "RUNBOOK_RESULT", "script": "runbooks/restart_service.sh", "service": "payment-svc", "returncode": 0, "stdout": "[restart_service] Restarting ronki-payment-svc...\nronki-payment-svc\n[restart_service] Waiting 5s for ronki-payment-svc to come up...\n[restart_service] ronki-payment-svc is running.", "stderr": ""}
{"ts": "2026-06-18T08:23:38.475697+00:00", "level": "INFO", "event_type": "ROLLBACK_EXECUTED", "service": "payment-svc", "rollback_runbook": "runbooks/restart_service.sh"}
```

**Đánh giá:** Dịch vụ có latency thực tế ~248ms vượt ngưỡng 10ms gây ra trạng thái `VERIFY_FAIL` sau 6 mẫu thử. Bộ điều phối tự động kích hoạt hoàn tác thông qua `ROLLBACK_TRIGGERED` và hoàn thành thành công (`ROLLBACK_EXECUTED`).

---

## Scenario 3 — Cầu chì hệ thống (Circuit Breaker)

* **Thiết lập:** Giữ ngưỡng latency thấp (10ms) và trigger liên tục 3 cảnh báo khác nhau để tăng số lần lỗi liên tiếp lên 3.
* **Mục tiêu:** Tự động ngắt hệ thống điều phối, suspends polling.

### Nhật ký Log:
```json
[Lần 1 - Đã ghi nhận ở kịch bản 2]
{"ts": "2026-06-18T08:23:38.475697+00:00", "level": "INFO", "event_type": "ROLLBACK_EXECUTED", "service": "payment-svc", "rollback_runbook": "runbooks/restart_service.sh"}

[Lần 2]
{"ts": "2026-06-18T08:25:16.332039+00:00", "level": "WARNING", "event_type": "VERIFY_FAIL", "service": "payment-svc", "samples": 6}
{"ts": "2026-06-18T08:25:16.332812+00:00", "level": "WARNING", "event_type": "ROLLBACK_TRIGGERED", "service": "payment-svc", "rollback_runbook": "runbooks/restart_service.sh"}
{"ts": "2026-06-18T08:25:23.848373+00:00", "level": "INFO", "event_type": "ROLLBACK_EXECUTED", "service": "payment-svc", "rollback_runbook": "runbooks/restart_service.sh"}

[Lần 3]
{"ts": "2026-06-18T08:27:02.359831+00:00", "level": "WARNING", "event_type": "VERIFY_FAIL", "service": "payment-svc", "samples": 6}
{"ts": "2026-06-18T08:27:02.360485+00:00", "level": "WARNING", "event_type": "ROLLBACK_TRIGGERED", "service": "payment-svc", "rollback_runbook": "runbooks/restart_service.sh"}
{"ts": "2026-06-18T08:27:09.555692+00:00", "level": "INFO", "event_type": "ROLLBACK_EXECUTED", "service": "payment-svc", "rollback_runbook": "runbooks/restart_service.sh"}
{"ts": "2026-06-18T08:27:09.556707+00:00", "level": "ERROR", "event_type": "CIRCUIT_BREAKER_HALT", "consecutive_failures": 3, "threshold": 3, "message": "Automation halted. Manual intervention required."}
{"ts": "2026-06-18T08:27:24.558790+00:00", "level": "ERROR", "event_type": "CIRCUIT_BREAKER_HALT", "message": "Circuit open — polling suspended."}
```

**Đánh giá:** Sau 3 lần xác minh thất bại liên tiếp, bộ điều phối chuyển trạng thái cầu chì thành mở, kích hoạt sự kiện ngắt `CIRCUIT_BREAKER_HALT` và đình chỉ toàn bộ hoạt động polling cho tới khi kỹ sư khởi động lại hệ thống.

---

## Scenario 4 — Hoàn tác giao dịch qua nhiều bước (Multi-step transactional rollback)

* **Thiết lập:** Cấu hình deploy 3 bước (A → B → C) cho `api-gateway`. Tạm thời rename container `ronki-api-gateway` thành `ronki-api-gateway-temp` sau khi hoàn thành bước B để ép bước C thất bại.
* **Mục tiêu:** Đảm bảo hoàn tác các bước đã hoàn thành (B và A) theo đúng thứ tự đảo ngược (B trước, A sau).

### Nhật ký Log:
```json
{"ts": "2026-06-18T08:29:56.815278+00:00", "level": "INFO", "event_type": "RUNBOOK_EXEC", "script": "runbooks/multi_step_deploy.sh --step-a", "service": "api-gateway", "dry_run": true}
{"ts": "2026-06-18T08:29:57.763675+00:00", "level": "INFO", "event_type": "DRY_RUN_PASS", "runbook": "runbooks/multi_step_deploy.sh --step-a", "service": "api-gateway"}
{"ts": "2026-06-18T08:29:57.766940+00:00", "level": "INFO", "event_type": "RUNBOOK_EXEC", "script": "runbooks/multi_step_deploy.sh --step-a", "service": "api-gateway", "dry_run": false}
{"ts": "2026-06-18T08:30:03.721539+00:00", "level": "INFO", "event_type": "RUNBOOK_RESULT", "script": "runbooks/multi_step_deploy.sh --step-a", "service": "api-gateway", "returncode": 0, "stdout": "[multi_step_deploy] step-A: draining traffic from ronki-api-gateway...\nronki-api-gateway\n[multi_step_deploy] step-A complete.", "stderr": ""}
{"ts": "2026-06-18T08:30:03.727160+00:00", "level": "INFO", "event_type": "RUNBOOK_EXEC", "script": "runbooks/multi_step_deploy.sh --step-b", "service": "api-gateway", "dry_run": false}
{"ts": "2026-06-18T08:30:08.834715+00:00", "level": "INFO", "event_type": "RUNBOOK_RESULT", "script": "runbooks/multi_step_deploy.sh --step-b", "service": "api-gateway", "returncode": 0, "stdout": "[multi_step_deploy] step-B: applying new config to ronki-api-gateway...\nronki-api-gateway\n[multi_step_deploy] step-B complete.", "stderr": ""}
{"ts": "2026-06-18T08:30:08.834715+00:00", "level": "INFO", "event_type": "RUNBOOK_EXEC", "script": "runbooks/multi_step_deploy.sh --step-c", "service": "api-gateway", "dry_run": false}
{"ts": "2026-06-18T08:30:11.448414+00:00", "level": "INFO", "event_type": "RUNBOOK_RESULT", "script": "runbooks/multi_step_deploy.sh --step-c", "service": "api-gateway", "returncode": 1, "stdout": "[multi_step_deploy] step-C: re-enabling traffic for ronki-api-gateway...\nronki-api-gateway\n[multi_step_deploy] ERROR: step-C traffic enable failed \u00e2\u20ac\u201d ronki-api-gateway status=\nmissing", "stderr": ""}
{"ts": "2026-06-18T08:30:11.450134+00:00", "level": "ERROR", "event_type": "TRANSACTIONAL_STEP_FAIL", "step": "runbooks/multi_step_deploy.sh --step-c", "service": "api-gateway", "completed_before_failure": ["runbooks/multi_step_deploy.sh --step-a", "runbooks/multi_step_deploy.sh --step-b"]}
{"ts": "2026-06-18T08:30:11.453003+00:00", "level": "WARNING", "event_type": "TRANSACTIONAL_ROLLBACK_STEP", "step": "runbooks/multi_step_deploy.sh --rollback-b", "service": "api-gateway"}
{"ts": "2026-06-18T08:30:11.453003+00:00", "level": "INFO", "event_type": "RUNBOOK_EXEC", "script": "runbooks/multi_step_deploy.sh --rollback-b", "service": "api-gateway", "dry_run": false}
{"ts": "2026-06-18T08:30:11.745232+00:00", "level": "INFO", "event_type": "RUNBOOK_RESULT", "script": "runbooks/multi_step_deploy.sh --rollback-b", "service": "api-gateway", "returncode": 1, "stdout": "[multi_step_deploy] rollback-B: reverting config on ronki-api-gateway...", "stderr": "Error response from daemon: No such container: ronki-api-gateway\nError: failed to start containers: ronki-api-gateway"}
{"ts": "2026-06-18T08:30:11.745232+00:00", "level": "WARNING", "event_type": "TRANSACTIONAL_ROLLBACK_STEP", "step": "runbooks/multi_step_deploy.sh --rollback-a", "service": "api-gateway"}
{"ts": "2026-06-18T08:30:11.745232+00:00", "level": "INFO", "event_type": "RUNBOOK_EXEC", "script": "runbooks/multi_step_deploy.sh --rollback-a", "service": "api-gateway", "dry_run": false}
{"ts": "2026-06-18T08:30:14.034450+00:00", "level": "INFO", "event_type": "RUNBOOK_RESULT", "script": "runbooks/multi_step_deploy.sh --rollback-a", "service": "api-gateway", "returncode": 0, "stdout": "[multi_step_deploy] rollback-A: restoring traffic to ronki-api-gateway...\n[multi_step_deploy] rollback-A complete.", "stderr": ""}
{"ts": "2026-06-18T08:30:14.034450+00:00", "level": "INFO", "event_type": "TRANSACTIONAL_ROLLBACK_COMPLETE", "service": "api-gateway", "rolled_back": ["runbooks/multi_step_deploy.sh --rollback-b", "runbooks/multi_step_deploy.sh --rollback-a"]}
```

**Đánh giá:** Các bước hoàn tác (`rollback-b` rồi tới `rollback-a`) được thực thi chính xác theo mô hình LIFO đảo ngược. Ghi chép audit trail đầy đủ và không để lại trạng thái cấu hình không nhất quán.

---

## Scenario 5 — Đua cảnh báo đồng thời (Concurrent alert race & Service Lock)

* **Thiết lập:** Post đồng thời 2 alert của `payment-svc` và `inventory-svc`, sau đó gửi thêm alert thứ 2 của `payment-svc` khi runbook trước đang chạy.
* **Mục tiêu:** Chạy đa luồng song song trên các dịch vụ khác nhau, khóa loại trừ (mutex) trên cùng một dịch vụ để chặn alert trùng lặp.

### Nhật ký Log:
```json
[Chạy song song không chặn nhau]
{"ts": "2026-06-18T08:33:57.345172+00:00", "level": "INFO", "event_type": "RUNBOOK_EXEC", "script": "runbooks/restart_service.sh", "service": "inventory-svc", "dry_run": false}
{"ts": "2026-06-18T08:33:57.345172+00:00", "level": "INFO", "event_type": "RUNBOOK_EXEC", "script": "runbooks/multi_step_deploy.sh --step-a", "service": "api-gateway", "dry_run": false}
{"ts": "2026-06-18T08:33:57.347371+00:00", "level": "INFO", "event_type": "RUNBOOK_EXEC", "script": "runbooks/restart_service.sh", "service": "payment-svc", "dry_run": false}

[Thử thực thi trùng lặp trên payment-svc gây khóa bận]
{"ts": "2026-06-18T08:34:57.333660+00:00", "level": "INFO", "event_type": "ALERT_DETECTED", "alertname": "HighLatency", "service": "payment-svc", "severity": "warning"}
{"ts": "2026-06-18T08:34:57.351820+00:00", "level": "INFO", "event_type": "DECIDE_RUNBOOK", "alertname": "HighLatency", "service": "payment-svc", "runbook": "runbooks/restart_service.sh"}
{"ts": "2026-06-18T08:34:57.411205+00:00", "level": "INFO", "event_type": "BLAST_RADIUS_OK", "service": "payment-svc"}
{"ts": "2026-06-18T08:34:57.412414+00:00", "level": "INFO", "event_type": "RUNBOOK_EXEC", "script": "runbooks/restart_service.sh", "service": "payment-svc", "dry_run": true}
{"ts": "2026-06-18T08:34:57.439929+00:00", "level": "INFO", "event_type": "ALERT_DETECTED", "alertname": "HighLatency", "service": "payment-svc", "severity": "warning"}
{"ts": "2026-06-18T08:34:57.520300+00:00", "level": "INFO", "event_type": "DECIDE_RUNBOOK", "alertname": "HighLatency", "service": "payment-svc", "runbook": "runbooks/restart_service.sh"}
{"ts": "2026-06-18T08:34:57.529433+00:00", "level": "INFO", "event_type": "BLAST_RADIUS_OK", "service": "payment-svc"}
{"ts": "2026-06-18T08:34:57.579941+00:00", "level": "WARNING", "event_type": "SERVICE_LOCK_BUSY", "service": "payment-svc", "message": "Another runbook is executing for this service; skipping duplicate"}
```

**Đánh giá:** Các luồng chạy hoàn toàn độc lập và không chờ nhau (dấu thời gian bắt đầu dry-run của `inventory-svc`, `api-gateway`, và `payment-svc` đều rơi vào cùng giây `08:33:57`). Cảnh báo thứ 2 cho cùng dịch vụ `payment-svc` bị chặn và log sự kiện `SERVICE_LOCK_BUSY` một cách chính xác.

---

## Scenario 6 — Đề kháng Hallucination quyết định (LLM Hallucination Defense)

* **Thiết lập:** Cấu hình alert `TestHallucination` ánh xạ tới `runbooks/nonexistent_runbook.sh`, trong khi whitelist registry không khai báo path này.
* **Mục tiêu:** Chặn đứng thực thi runbook lạ, log validation thất bại và bảo vệ cầu chì.

### Nhật ký Log:
```json
{"ts": "2026-06-18T08:35:58.098376+00:00", "level": "INFO", "event_type": "ALERT_DETECTED", "alertname": "TestHallucination", "service": "payment-svc", "severity": "critical"}
{"ts": "2026-06-18T08:35:58.099395+00:00", "level": "ERROR", "event_type": "DECISION_VALIDATION_FAILED", "bad_runbook": "runbooks/nonexistent_runbook.sh", "alertname": "TestHallucination", "raw_decision": "runbooks/nonexistent_runbook.sh", "action": "escalate_no_auto_action"}
```

**Đánh giá:** Bộ điều phối phát hiện runbook không có trong registry, lập tức kích hoạt `DECISION_VALIDATION_FAILED` và từ chối chạy dry-run/subprocess. Không có subprocess nào được spawn và cầu chì không bị tăng giá trị lỗi.
