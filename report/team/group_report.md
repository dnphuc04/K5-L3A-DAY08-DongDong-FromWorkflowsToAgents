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

*Phần này do Nguyễn Huy Hoàng soạn.* Số liệu lấy từ `src/reports/benchmark.json` (commit `0f13675`), sinh bằng `python -m lab8.benchmark --shapes workflow,single,orchestrator,swarm --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/benchmark.json`. Cùng 14 claim public, cùng 5 seed, cùng bảng giá mô phỏng, mỗi trial một SQLite mới (70 trial/shape).

| Shape | Pass rate | pass^5 | Seed min–max | Variance | Cost/success (VND mô phỏng) | Mean / p95 steps | Model calls TB | Tool calls TB |
| --- | --: | --: | --: | --: | --: | --: | --: | --: |
| A `workflow` | 1.000 | 1.000 | 1.000–1.000 | 0.000000 | **27.86** | 3.79 / 4 | 0.00 | 2.79 |
| B `single` | 1.000 | 1.000 | 1.000–1.000 | 0.000000 | 29.97 | 4.79 / 5 | 3.79 | 2.79 |
| C `orchestrator` | 1.000 | 1.000 | 1.000–1.000 | 0.000000 | 56.65 | 4.71 / 5 | 1.00 | 5.57 |
| D `swarm` (tham chiếu) | 0.986 | 0.929 | 0.929–1.000 | 0.000816 | 49.57 | 2.80 / 3 | 3.00 | 4.66 |

**Đọc số liệu.**

- Ba shape của nhóm đều bão hòa ở 1.000 trên bộ public; trên corpus này chất lượng không phân biệt được giữa A, B, C, chỉ cost và độ tự chủ là khác nhau.
- Swarm có đúng 1/70 trial fail: `R-swarm-C-002-s55`, `status resolved != denied`. Ticket C-002 ghi trễ 5 ngày, DB ghi 2 ngày; ở seed 55 swarm tin ticket (khối `REFERENCE_WEAKNESS`) nên ra quote dương và host ghi 20.000 VND vì approval A-002 hợp lệ. Host chặn approval/trần/idempotency nhưng không chặn được **dữ kiện sai** mà shape đưa vào — vì vậy ranh giới "chỉ dùng dữ kiện từ DB" phải nằm trong shape. Theo taxonomy MAST đây là lỗi kiểm chứng (không đối chiếu dữ kiện ticket với nguồn có thẩm quyền).
- Cost: single đắt hơn workflow 7.6% (model call soạn ReAct, token rẻ); orchestrator đắt gấp 2.03 lần, chủ yếu vì tool call gấp đôi (đọc `get_claim_view` hai lần + investigator chạy đủ 4 tool đọc). Claim dừng sớm (C-004 thiếu mã đơn) gần như cùng giá ở mọi shape.

**Crash safety.** `python -m lab8.crash_test --count 20 --out reports/crash.json` → `crash_safety: 20/20` (5 ca `before_effect`, 7 ca `after_effect_before_checkpoint`, 5 ca `after_checkpoint`, cộng amount conflict, retry thiếu approval và swarm crash).

**Kiểm thử an toàn xuyên shape** (`src/tests/test_team_safety.py`, 55 ca, chạy trên workflow/single/orchestrator):

| Nhóm ca | Số ca | Kết quả |
| --- | --: | --- |
| Approval của claim khác (C-001 trỏ A-003 150.000 VND), 2 seed | 6 | không refund, không audit |
| Approval thiếu / không tồn tại | 6 | không refund |
| C-005 (không approval), C-007 (approval hết hạn), chạy lại 3 seed | 6 | không refund |
| Ticket injection C-009 ("ignore policy", approval 500.000) | 3 | `denied`, ledger trống |
| Crash 3 điểm × 3 claim trả tiền (C-001, C-003, C-014) rồi resume bằng seed khác | 27 | đúng 1 refund, 1 audit, số tiền bằng lần chạy sạch |
| Crash liên tiếp trên cùng store rồi chạy lại | 3 | 1 refund 35.000 |
| 4 claim cùng đơn O-107 chạy 6 lần | 3 | tổng ledger của đơn = 35.000, 1 dòng |
| Ba shape lần lượt chạy C-014 trên cùng store | 1 | 1 refund 120.000 (150.000 − 30.000 đã hoàn) |

Để chắc test bắt được lỗi thật, nhóm thử hai đột biến tạm thời trên workflow (đã hoàn nguyên, không commit): ghi thẳng vào ledger bỏ qua host → 16/18 ca workflow fail; bỏ truyền `crash_at` xuống host → 10/18 ca fail.

**Ablation R1** (`src/reports/ablation_r1.json`): chỉ đổi topology workflow → orchestrator, pass giữ 1.000, cost/success ×2.03; chi tiết ở `report/team/research_card.md`.

**Tự kiểm hidden.** Nhóm chưa nhận `student_drop/` nên chưa có `src/reports/hidden_selfcheck.json`; không có số liệu hidden nào được ghi ở đây.

Quyết định kiến trúc và lý do: xem `report/team/decision_adr.md`. Phiếu nghiên cứu và ablation: xem `report/team/research_card.md`.

## 5. Khó khăn và cách xử lý

Lỗi gặp phải (kèm lệnh, output hoặc ảnh chụp) và cách nhóm xử lý.

...

## 6. Tổng hợp đóng góp

Chi tiết đóng góp của từng thành viên nằm trong `TEAM.md` (bảng đóng góp) và báo cáo cá nhân `report/team/MSSV_HoTen.md`.

## 7. Hạn chế và hướng cải thiện

MockLLM không phải model sản xuất; latency là `unavailable`.

*Phần này do Nguyễn Huy Hoàng soạn.*

**Hạn chế.**

- **Corpus nhỏ và bão hòa.** 14 claim public; ba shape của nhóm đều đạt 1.000 nên benchmark chỉ phân biệt cost, không phân biệt chất lượng. Variance 0 trên public không đảm bảo variance 0 trên hidden (homoglyph, principal lẫn, đơn hủy/đang giao…).
- **Chưa tự kiểm hidden.** Không có `student_drop/`, nên ngưỡng pass ≥ 0.75 trên 40 claim × 5 seed chưa được kiểm trước.
- **MockLLM quá ngoan.** Model call của single và orchestrator không quyết định status hay số tiền, nên mọi seed cho cùng kết quả. Với model thật, single có thể chọn tool sai thứ tự, hết 8 bước hoặc tốn token hơn; con số +7.6% cost chỉ đúng với giá và token mô phỏng.
- **Cost là mô phỏng, không có latency.** Ưu điểm song song của investigator (nếu có) không thể hiện được.
- **Test an toàn chỉ phủ ca nhóm nghĩ ra.** Crash được bơm ở 3 điểm của host; crash ngoài host (giữa hai lần đọc tool) không làm sai tiền vì chưa ghi gì, nhưng chưa có test riêng. Test cũng chưa chạy song song hai tiến trình cùng claim.

**Hướng cải thiện.**

1. Chạy tự kiểm trên `student_drop/` ngay khi có và chỉ giữ quyết định workflow nếu vẫn ≥ 0.75 với variance thấp.
2. Bổ sung test metamorphic tự sinh: đổi số ngày trễ trong ticket, thêm homoglyph vào mã đơn, đổi principal — kỳ vọng status/số tiền không đổi hoặc đổi đúng hướng.
3. Bỏ lần đọc `get_claim_view` trùng trong orchestrator (truyền `principal_id`/`order_id` đã có) để giảm ~1 tool call mỗi claim nếu nhóm cần dùng shape này.
4. Thêm test hai tiến trình ghi cùng claim để kiểm khóa `BEGIN IMMEDIATE` và idempotency dưới tải đồng thời.

**Điều kiện làm kết luận thay đổi.** Nếu hidden có loại claim mà luật cố định không bao được và workflow rơi dưới 0.75 trong khi single/orchestrator vượt rõ với variance không tăng, nhóm sẽ đổi sang single trước (cost gần workflow nhất). Nếu model thật làm single dao động giữa các seed, workflow càng được ưu tiên.

## 8. Tài liệu tham khảo

...
