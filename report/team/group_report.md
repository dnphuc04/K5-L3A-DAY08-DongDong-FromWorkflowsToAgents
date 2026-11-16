# Báo cáo nhóm — Lab 8 LogiX-VN

Tối đa 5 trang ngoài phụ lục. Số liệu lấy từ file trong `src/reports/`, không chép tay từ terminal.

## 1. Thông tin nhóm

- Tên nhóm: DongDong
- Thành viên: xem `TEAM.md`
- Commit chốt (trùng `report/team/decision_adr.md`):

## 2. Mục tiêu và cách tiếp cận

Nhóm hiểu bài toán thế nào, chia việc ra sao, thứ tự làm bốn shape.

...

## 3. Giải pháp

### 3.1. Workflow — Đồng Ngọc Phúc

Shape A trong `src/lab8/shapes/workflow.py` thực hiện các bước Python cố định, giữ nguyên chữ ký `run_claim`. Workflow đọc claim bằng `get_claim_view`; claim không tồn tại trả `failed`, thiếu `order_id` trả `needs_info`. Sau đó `lookup_order` kiểm tra phạm vi principal; đơn không tìm thấy hoặc không thuộc principal trả `escalated`. Workflow không suy ra mã đơn hoặc số ngày trễ từ ticket.

Với đơn hợp lệ, `compute_compensation` tính quote từ dữ kiện đơn và policy hiện hành. Quote bằng 0 trả `denied`, không gọi ghi refund. Quote dương được truyền đầy đủ vào `durability.issue_refund`: host kiểm approval, áp trần theo đơn và tự trừ tiền các claim trước đã nhận. Ví dụ entitlement 150.000 VND và đã hoàn 30.000 VND thì workflow truyền 150.000 VND, host ghi phần chênh 120.000 VND. `proposal_amount` vẫn giữ 150.000 VND. Host từ chối trả `denied`; xung đột amount trả `failed`; host chấp nhận trả `resolved`.

Workflow không gọi model (`model_calls = 0`); quyết định và số tiền đều do code xử lý. Mỗi lần gọi tool đọc được cộng vào `Budget`. Việc ghi tiền chỉ qua host để giữ approval, idempotency và ledger trong cùng cơ chế có sẵn. Workflow truyền `crash_at` xuống host, để `InjectedCrash` truyền ra ngoài và luôn đóng kết nối bằng `finally`. Khi chạy lại, host dùng khóa `refund:{claim_id}` để ngăn ghi trùng.

Mọi nhánh trả kết quả dùng `common.finish()` để lấy effect/audit từ ledger và ghi trace mới. Trace có các event `claim` (chỉ `found`), `order` (chỉ `visible`, `reason`), `quote` và `decision`. Không ghi ticket, principal, claim/order row thô, secret, header, SĐT, địa chỉ, tên hàng hay ghi chú nội bộ. Shape này không có handoff. Đường crash chủ động truyền exception, không trả `RunResult`; trace của lần resume được ghi khi hoàn tất.

Test riêng trong `src/tests/test_team_workflow.py` gồm 6 ca: ticket nói quá hoặc phủ nhận độ trễ không thay đổi quote ở seed 11 và 55; thiếu `order_id` không được lấy mã đơn từ ticket và không ghi thêm ledger; claim không tồn tại trả `failed` kèm trace hợp lệ. Test thiếu mã đơn cũng kiểm tra trace không chứa canary trong ticket. Ma trận student của workflow đã đạt 62/62, bao gồm retry và crash/resume. Đây là kiểm thử chức năng, chưa thay thế benchmark bốn shape hoặc kết quả hidden.

### 3.2. Single

Chờ bổ sung mô tả triển khai của thành viên phụ trách.

### 3.3. Orchestrator

Chờ bổ sung mô tả triển khai của thành viên phụ trách.

## 4. Kết quả

Bảng so sánh 4 shape trên bộ public (từ `src/reports/benchmark.json`): pass rate, pass^5, seed min–max, variance, cost/success, mean/p95 steps. Kèm crash safety (`src/reports/crash.json`) và, nếu có, kết quả tự kiểm trên `student_drop/`.

...

Quyết định kiến trúc và lý do: xem `report/team/decision_adr.md`. Phiếu nghiên cứu và ablation: xem `report/team/research_card.md`.

## 5. Khó khăn và cách xử lý

Lỗi gặp phải (kèm lệnh, output hoặc ảnh chụp) và cách nhóm xử lý.

...

## 6. Tổng hợp đóng góp

Chi tiết đóng góp của từng thành viên nằm trong `TEAM.md` (bảng đóng góp) và báo cáo cá nhân `report/team/MSSV_HoTen.md`.

## 7. Hạn chế và hướng cải thiện

MockLLM không phải model sản xuất; latency là `unavailable`. Nêu thêm giới hạn của nhóm và điều kiện làm kết luận thay đổi.

...

## 8. Tài liệu tham khảo

...
