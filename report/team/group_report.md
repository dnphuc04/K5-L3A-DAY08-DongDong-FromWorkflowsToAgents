# Báo cáo nhóm — Lab 8 LogiX-VN

Số liệu benchmark và crash lấy từ các file do công cụ sinh trong `src/reports/`. Kết quả pytest ghi riêng như bằng chứng kiểm thử chức năng.

## 1. Thông tin nhóm

- Tên nhóm: DongDong
- Thành viên: xem `TEAM.md`
- Commit chốt (trùng `report/team/decision_adr.md`): `0f13675dd9cc24832dc466fe465a47c07ae97a18` (mốc code, test và report thực nghiệm). Kiểm tra tích hợp bổ sung tại `a8a9835`; các cập nhật tài liệu sau mốc thực nghiệm không thay thế hash dẫn chứng trong ADR.

## 2. Mục tiêu và cách tiếp cận

Nhóm xử lý claim hoàn tiền LogiX-VN theo cùng contract bằng workflow, single agent, orchestrator và swarm tham chiếu. Mục tiêu là chọn kiến trúc ít tự chủ nhất vẫn đúng nghiệp vụ, giữ approval, bảo mật trace và chống refund trùng. Database và policy hiện hành là nguồn sự thật; ticket là dữ liệu không đáng tin. Code tính tiền, host quyết quyền ghi, model chỉ hoạt động trong phạm vi được cấp.

Phúc phụ trách workflow, tích hợp và tổng hợp báo cáo; Tiến phụ trách single/ReAct; Hưng phụ trách orchestrator/handoff; Hoàng phụ trách kiểm thử an toàn, crash, benchmark và phân tích. Ba shape được triển khai song song trong các file riêng, sau đó tích hợp vào `main`. Mỗi người bổ sung test và báo cáo cá nhân; commit/test cụ thể nằm trong `TEAM.md`.

Nhóm kiểm tra theo thứ tự: smoke và host → student matrix từng shape → kiểm thử chéo approval, secret, retry/crash → benchmark bốn shape trên cùng 14 claim, năm seed và bảng giá → ADR/research card. Mỗi trial benchmark dùng SQLite mới. Quyết định chỉ dựa trên số liệu public và giá mô phỏng, không suy rộng thành kết quả hidden hoặc hiệu năng model thật.

## 3. Giải pháp

### 3.1. Workflow — Đồng Ngọc Phúc

Shape A trong `src/lab8/shapes/workflow.py` thực hiện các bước Python cố định, giữ nguyên chữ ký `run_claim`. Workflow đọc claim bằng `get_claim_view`; claim không tồn tại trả `failed`, thiếu `order_id` trả `needs_info`. Sau đó `lookup_order` kiểm tra phạm vi principal; đơn không tìm thấy hoặc không thuộc principal trả `escalated`. Workflow không suy ra mã đơn hoặc số ngày trễ từ ticket.

Với đơn hợp lệ, `compute_compensation` tính quote từ dữ kiện đơn và policy hiện hành. Quote bằng 0 trả `denied`, không gọi ghi refund. Quote dương được truyền đầy đủ vào `durability.issue_refund`: host kiểm approval, áp trần theo đơn và tự trừ tiền các claim trước đã nhận. Ví dụ entitlement 150.000 VND và đã hoàn 30.000 VND thì workflow truyền 150.000 VND, host ghi phần chênh 120.000 VND. `proposal_amount` vẫn giữ 150.000 VND. Host từ chối trả `denied`; xung đột amount trả `failed`; host chấp nhận trả `resolved`.

Workflow không gọi model (`model_calls = 0`); quyết định và số tiền đều do code xử lý. Mỗi lần gọi tool đọc được cộng vào `Budget`. Việc ghi tiền chỉ qua host để giữ approval, idempotency và ledger trong cùng cơ chế có sẵn. Workflow truyền `crash_at` xuống host, để `InjectedCrash` truyền ra ngoài và luôn đóng kết nối bằng `finally`. Khi chạy lại, host dùng khóa `refund:{claim_id}` để ngăn ghi trùng.

Mọi nhánh trả kết quả dùng `common.finish()` để lấy effect/audit từ ledger và ghi trace mới. Trace có các event `claim` (chỉ `found`), `order` (chỉ `visible`, `reason`), `quote` và `decision`. Không ghi ticket, principal, claim/order row thô, secret, header, SĐT, địa chỉ, tên hàng hay ghi chú nội bộ. Shape này không có handoff. Đường crash chủ động truyền exception, không trả `RunResult`; trace của lần resume được ghi khi hoàn tất.

Test riêng trong `src/tests/test_team_workflow.py` gồm 6 ca: ticket nói quá hoặc phủ nhận độ trễ không thay đổi quote ở seed 11 và 55; thiếu `order_id` không được lấy mã đơn từ ticket và không ghi thêm ledger; claim không tồn tại trả `failed` kèm trace hợp lệ. Test thiếu mã đơn cũng kiểm tra trace không chứa canary trong ticket. Ma trận student của workflow đã đạt 62/62, bao gồm retry và crash/resume. Đây là kiểm thử chức năng, chưa thay thế benchmark bốn shape hoặc kết quả hidden.

### 3.2. Single — Nguyễn Trần Tiến

Shape B trong `src/lab8/shapes/single.py` là một agent chạy vòng ReAct bằng `lab8.runtime.run_react` với `max_steps=8`, giữ nguyên chữ ký `run_claim`. Shape đọc claim bằng `get_claim_view`; claim không tồn tại trả `failed`, thiếu `order_id` trả `needs_info` mà không gọi thêm tool nào. Shape không lấy mã đơn hay số ngày trễ từ ticket.

**Ranh giới code và model.** Hàm `decide` đại diện cho model và chỉ được chọn tool đọc: lần đầu `lookup_order`, sau đó `compute_compensation` với order row vừa đọc, rồi trả `final`. `runtime.dispatch` từ chối `issue_refund` và `append_audit` bằng `ToolRejected`, nên model không có đường nào ghi tiền. Mỗi bước của vòng lặp là một model call MockLLM (`purpose="react"`), cộng một lần soạn phản hồi khách (`purpose="draft"`), đều được cộng vào `Budget`. Model không quyết định trạng thái hay số tiền. Quote do `compute_compensation` tính từ order row trong DB và policy hiện hành, nên ticket nói quá hoặc chèn lệnh không đổi được kết quả (C-002 bị `denied` ở cả 5 seed, kể cả seed 55 nơi swarm thua). Shape không gọi `lookup_refund_history`: host đã tự trừ phần đã hoàn theo đơn, gọi thêm chỉ tốn chi phí mà không đổi kết quả.

**Ghi tiền.** Đơn không tìm thấy hoặc thuộc principal khác trả `escalated`. Quote bằng 0 trả `denied` và không gọi host. Quote dương được truyền **đầy đủ** vào `durability.issue_refund` kèm `crash_at`; host kiểm approval, áp trần theo đơn, tự trừ phần đã hoàn và giữ idempotency. Nếu shape tự trừ trước thì host trừ thêm lần nữa và T14 sai. Host từ chối trả `denied` với `proposal_amount` là quote đầy đủ (có audit `policy_denied`); xung đột amount trả `failed`; host chấp nhận trả `resolved`. `InjectedCrash` được ghi vào trace rồi ném lại, không bị nuốt, nên T12 thấy crash thật; lần chạy lại cùng store nhận `idempotent_replay` và ledger chỉ có một dòng. Chạm trần 8 bước trả `failed` với `error_reason="step_cap"`; lỗi tool khác trong vòng lặp cũng trả `failed` thay vì ném exception ra khỏi `run_claim`.

**Trace.** Mọi nhánh return đi qua `common.finish()`, nên luôn có trace mới với `run_id` dạng `R-single-<claim>-s<seed>`. Các event gồm: `claim` (chỉ `has_order_id`), `observation` (order view đã qua allowlist của `lookup_order`), `quote`, `loop`, `decision` hoặc `host`, và `crash`/`error` khi có. Trace không chứa ticket, claim row thô, secret, header, SĐT, địa chỉ, ghi chú nội bộ hay tên hàng.

**Kiểm thử.** `src/tests/test_team_single.py` có 19 ca: một `decide` không bao giờ trả `final` bị cắt đúng ở 8 bước (`step_cap`); 14 claim public đều chạy xong mà không chạm trần; chọn `issue_refund` hoặc `append_audit` bị `ToolRejected` và ledger không có dòng; ghi lại mọi tool mà shape gọi trên C-001 thì toàn bộ thuộc `READONLY_TOOLS` và refund do host ghi; trace đúng contract và không chứa `CANARY_SECRET`, `phone`, `address`, `internal_note`, `item_name`, `parent_secret`. Ma trận student của single đạt 62/62, gồm T11 (retry) và T12 (crash/resume). Đây là kiểm thử chức năng, chưa thay thế benchmark bốn shape hoặc kết quả hidden; số liệu cost của single nằm ở mục 4.

**Giới hạn.** `decide` hiện là chính sách cố định chạy qua MockLLM, nên mọi seed cho cùng kết quả. Với model thật, single có thể chọn tool sai thứ tự hoặc dùng hết 8 bước; trường hợp đó shape trả `failed` chứ không ghi tiền, và host vẫn chặn mọi ghi sai.

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

| Vấn đề quan sát được | Cách xử lý và bằng chứng |
| --- | --- |
| Chạy `pip install -r requirements.txt` tại gốc repo báo `No such file or directory` | Chuyển vào `src/` trước khi cài và chạy lệnh lab. Giữ nguyên cấu trúc repo. |
| Pytest gặp `PermissionError: [WinError 5]` khi truy cập thư mục tạm `pytest-of-ThinkBook` trên Windows | Dùng `--basetemp` riêng trong `.pytest_cache`; không thay test hay logic host để né lỗi. Lệnh kiểm tra cuối được ghi bên dưới. |
| Merge `TEAM.md` xung đột giữa thông tin Tiến và Hưng | Gộp thủ công thông tin mới của cả hai, xóa conflict marker, giữ mỗi người một dòng và kiểm tra lại bảng. |
| PR ban đầu chọn repo gốc chương trình làm đích | Đóng PR nhầm; tạo PR vào `main` của repo nhóm và kiểm tra base repository trước khi merge. |

Tại commit tích hợp `a8a9835`, kiểm tra trên Windows với Python 3.14.6 bằng executable của `.venv` ở gốc repo. Các lệnh sau chạy từ `src/`:

```powershell
../.venv/Scripts/python.exe -m lab8.smoke
../.venv/Scripts/python.exe -m pytest -q --tb=short --basetemp=.pytest_cache/final-host-03
../.venv/Scripts/python.exe -m pytest -q -m student -o addopts= --tb=short --basetemp=.pytest_cache/final-student-03
../.venv/Scripts/python.exe -m lab8.check_submission
```

Kết quả: smoke pass; host và test nhóm **181 passed**; student **248 passed**; kiểm tra nộp bài **0 FAIL, 0 WARN**. Không còn shape stub. Đây là kết quả local, chưa chứng minh CI hoặc hidden pass. Python 3.14.6 nằm ngoài dải 3.11–3.13 trong hướng dẫn lab; cần chạy lại trên phiên bản được yêu cầu trước khi chốt môi trường tái lập.

## 6. Tổng hợp đóng góp

Phúc hoàn thành workflow, 6 ca test riêng, tích hợp và tổng hợp tài liệu; Tiến hoàn thành single và 19 ca test; Hưng hoàn thành orchestrator và 47 ca test; Hoàng hoàn thành 55 ca test an toàn xuyên shape, report crash/benchmark/ablation và phân tích. Chi tiết commit và tên test nằm trong bảng đóng góp `TEAM.md`; mỗi thành viên có báo cáo riêng trong `report/team/`. Số ca trên là test nhóm bổ sung, không cộng lẫn với 248 ca student.

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

- [Luật nghiệp vụ LogiX-VN](../../docs/business_rules.md): điều kiện hoàn tiền, policy hiệu lực, approval và trần theo đơn.
- [Output và trace contract](../../docs/contracts.md): trạng thái, proposal, ledger/audit và ranh giới dữ liệu trong trace.
- [Kiến trúc](../../docs/architecture.md) và [định nghĩa metric](../../docs/metrics.md): phạm vi code/model và cách so sánh các shape.
- [Quy định lab](../../RULES.md), [rubric](../../RUBRIC.md) và [hướng dẫn nộp](../../SUBMISSION.md): phạm vi được sửa, bằng chứng và điều kiện bắt buộc.
- [Benchmark public](../../src/reports/benchmark.json), [crash suite](../../src/reports/crash.json), [ablation R1](../../src/reports/ablation_r1.json): bằng chứng thực nghiệm của nhóm, commit `0f13675`.
- [ADR](decision_adr.md) và [research card](research_card.md): quyết định chọn workflow, giả thuyết R1 và giới hạn. Nguồn nghiên cứu được mô tả theo thư viện tham khảo của khóa trong research card; kết quả lab không được coi là tái lập toàn bộ nghiên cứu đó.
