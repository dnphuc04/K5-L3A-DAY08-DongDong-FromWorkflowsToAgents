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

### 3.3. Orchestrator — Nguyễn Việt Hưng

Shape C trong `src/lab8/shapes/orchestrator.py` tách vai: orchestrator giữ quyền ghi, investigator chỉ đọc. Orchestrator đọc claim bằng `get_claim_view`, không đọc claim row thô vì row này chứa `parent_secret` và `raw_headers`. Claim không tồn tại trả `failed`, thiếu `order_id` trả `needs_info`.

**Handoff.** Orchestrator tạo handoff `handoff.v1` bằng `make_handoff`, với `facts={}` và `allowed_tools` gồm 4 tool đọc: `lookup_order`, `lookup_policy`, `lookup_refund_history`, `compute_compensation`. Allowlist nằm trong code: `make_handoff` chỉ giữ trường thuộc `ALLOWED_FIELDS` và key thuộc `FACT_KEYS`, từ chối tool ghi, và từ chối payload còn canary hay header. Ticket không được đưa vào handoff vì đây là dữ liệu khách tự viết, có thể sai hoặc chứa lệnh chèn; tóm tắt ngữ cảnh bằng LLM cũng không được coi là sanitize.

**Investigator.** `run_readonly_investigators` chạy các tool trong handoff qua `dispatch(worker=True)` với `principal_id` của claim, nên investigator không có đường nào gọi `issue_refund`. Đơn không tìm thấy hoặc thuộc principal khác trả `escalated`. Quote do `compute_compensation` tính từ order row trong DB và policy hiện hành. Shape không dùng số ngày trễ trong ticket, nên C-002 (ticket ghi 5 ngày, đơn ghi 2 ngày) bị `denied` ở cả 5 seed. Đây là ca swarm tham chiếu thua ở seed 55.

**Ghi tiền.** Quote bằng 0 trả `denied` và không gọi host. Quote dương được truyền đầy đủ vào `durability.issue_refund` kèm `crash_at`; host kiểm approval, áp trần theo đơn, trừ phần đã hoàn và giữ idempotency. Ví dụ C-014: shape đề xuất 150.000 VND, host ghi 120.000 VND, `proposal_amount` vẫn là 150.000. `InjectedCrash` được ghi vào trace, flush rồi ném lại; lần chạy lại nhận `idempotent_replay` nên ledger chỉ có một dòng. Mỗi lần chạy có đúng một model call (`purpose="draft"`, soạn phản hồi khách); model không quyết định trạng thái hay số tiền.

**Trace.** Mọi nhánh return đi qua `common.finish()`, nên luôn có trace mới với `run_id` dạng `R-orchestrator-<claim>-s<seed>`. Các event gồm: `claim` (chỉ `has_order_id`), `handoff` (bản đã sanitize), `investigator` (`order_visible`, `order_reason`, lịch sử hoàn tiền dạng số), `quote`, `decision`, và `crash` khi có. Trace không chứa ticket, claim/order row thô, secret, header, SĐT, địa chỉ, ghi chú nội bộ hay tên hàng.

**Kiểm thử.** `src/tests/test_team_orchestrator.py` có 47 ca. Test bắt handoff thật mà orchestrator gửi cho investigator trên 14 claim và kiểm tra allowlist, scope, tool chỉ đọc, không lộ dữ liệu nhạy cảm. Test cũng kiểm tra việc lỡ cấp `issue_refund` cho investigator bị `HandoffError` chặn và không ghi tiền; trace của 14 claim × 2 seed đúng contract và sạch; crash/resume chỉ ghi một refund; ticket không ghi đè dữ kiện DB. Phép so sánh dùng giá trị nhạy cảm thật lấy từ dữ liệu, không chỉ tìm từ khóa. Khi cố tình cho orchestrator ghi claim row vào trace, 29/47 ca fail. Ma trận student của orchestrator đạt 62/62, gồm T11 (retry) và T12 (crash/resume). Đây là kiểm thử chức năng, chưa thay thế benchmark bốn shape hoặc kết quả hidden.

**Giới hạn.** Với claim có đơn hợp lệ, `tool_calls` là 6 vì `get_claim_view` được đọc hai lần: một lần ở orchestrator để lấy `principal_id`/`order_id` trước khi tạo handoff, một lần bên trong `run_readonly_investigators`. Chi phí này sẽ thể hiện trong benchmark ở mục 4.

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
